# ADR-005: Data-Plane Read Restriction Under a HIPAA Customer Constraint — Keep ReadOnlyAccess, Deny Customer-Data Reads

Status: Proposed  
Date: 2026-09-02  
Author: Haider Zaidi  
Reviewers: CEO, COO, Product Manager, Solutions Architects

## Context

`ProductXWAFRRole` (and its member-account counterpart `ProductXWAFRMemberRole`) attach the AWS-managed `ReadOnlyAccess` policy plus `SecurityAudit` plus inline policies. `ReadOnlyAccess` is very broad — it grants both metadata reads (Describe/List) **and data-plane reads** (`s3:GetObject`, `dynamodb:GetItem`, `cloudwatch:GetMetricData`, etc.) across ~500 services.

A HIPAA-covered customer has raised a formal, gating concern: `ReadOnlyAccess` grants `s3:GetObject`, which permits reading **S3 object contents** — potential PHI. Their compliance posture does not allow a third-party cross-account role that can read object data, even read-only. We need a permission model that satisfies this constraint without degrading ProductX functionality.

Two candidate directions were evaluated:
1. Replace `ReadOnlyAccess` with the narrower AWS-managed `ViewOnlyAccess` (enumeration/metadata only, no data-plane reads).
2. Keep `ReadOnlyAccess` but surgically remove `s3:GetObject` where it isn't needed.

To evaluate direction (1), every `*Command()` call the platform makes into a customer account was extracted from the `scanner`, `optimizer-sync`, `cur-ingester`, `poller-aws-verify`, `api-aws`, and `packages/shared` codebases, mapped to its real IAM action via the import package, filtered to exclude platform-side calls, and evaluated against **`ViewOnly` + `SecurityAudit` + inline** vs. **`ReadOnly` + `SecurityAudit` + inline**.

## Analysis

### Why ProductX does not need `s3:GetObject` on customer data buckets

Per the engine design (`docs/knowledge/design.md`) and the scanner code, **every legitimate `s3:GetObject` call ProductX makes is scoped to the CUR (Cost & Usage Report) bucket** — the Billed Cost source:
- Design doc Data Sources table: "`s3:GetObject` on the Cost and Usage Report (CUR) bucket = the **Billed Cost** source."
- The CUR bucket read is already granted by the dedicated `ManageCURPolicy` (`arn:aws:s3:::productx-cur-*/*`) and `ExistingCURBucketPolicy` (the specified bucket) — **not** by `ReadOnlyAccess`.

The WAFR scanner's S3 work is entirely **configuration/metadata** reads, never object contents:
- `s3:GetPublicAccessBlock`, `s3:GetObjectLockConfiguration`, `s3:GetEncryptionConfiguration`, `s3:GetBucketPolicy`, `s3:GetBucketTagging`, `s3:GetBucketLocation`, `s3:ListBucket`, etc.

**Therefore the HIPAA concern is resolvable with zero loss of capability:** ProductX has no functional need to read object contents on general customer buckets. The only object read it requires is the CUR bucket, which is covered by dedicated prefix-scoped policies independent of `ReadOnlyAccess`.

### Why not solve it by switching to ViewOnlyAccess

`ViewOnlyAccess` would remove `s3:GetObject` (satisfying the HIPAA concern) but it also removes ~31 other data-plane reads the scanner depends on. **Switching to ViewOnlyAccess breaks 32 of 191 customer-account actions.** 158 still work. The 32 failures fall into three tiers:

**Tier 1 — silently wrong data (disqualifying).**

| Action | Consequence |
|---|---|
| `s3:GetPublicAccessBlock` | Fabricated finding on every bucket |
| `s3:GetObjectLockConfiguration` | Object Lock always reads as absent |

These fail **wrong, not absent**. `storage.ts:77-91` wraps them in `bestEffort`, which converts a 403 into `null` — indistinguishable from "not configured." So `publicAccessBlocked` becomes `false` for **every** bucket, flows into the WAFR security signal at `signals/storage.ts:16` as a **FAILED** signal, and into `resource-catalog.ts:533`, which labels the bucket **"Public"** in the UI — with nothing in the logs explaining why (`bestEffort` stays silent on 403s). Every customer would see every bucket falsely reported as publicly accessible.

Note the irony: the customer objects to `s3:GetObject`, but the S3 permissions the scanner genuinely needs are the **configuration** reads in this tier — which ViewOnlyAccess also strips.

**Tier 2 — right-sizing engine loses inputs.**

| Action | Impact |
|---|---|
| `logs:StartQuery`, `logs:GetQueryResults` | Kills the Lambda memory recipe (`lambda-memory.ts:542`) |
| `pi:GetResourceMetrics` | Kills RDS swap evidence (`pi-metrics.ts:172`) |

With Compute Optimizer already removed (ADR-004) and the recipe engine running solo, losing CloudWatch Logs Insights means **Lambda memory sizing stops existing** rather than falling back.

**Tier 3 — ~26 WAFR signals silently vanish.**

`budgets:*` (3), `ce:GetCostAndUsage`, `ce:GetSavingsPlansCoverage`, `ce:ListCostAllocationTags`, `ce:ListCostCategoryDefinitions`, `xray:GetServiceGraph`, `macie2:GetMacieSession`, `resiliencehub:*` (2), `drs:DescribeSourceServers`, `fis:*` (2), `route53-recovery-readiness:ListReadinessChecks`, `backup:GetBackupPlan`, `backup:ListRestoreTestingPlans`, `batch:DescribeJobQueues`, `appconfig:ListApplications`, `application-signals:ListServices`, `codeconnections:ListConnections`, `codeguru-reviewer:*`, `codeguru-profiler:*`, `imagebuilder:ListImagePipelines`, `lakeformation:ListLFTags`, `ssm-incidents:ListResponsePlans`, `account:GetContactInformation`.

These degrade cleanly (signal absent, not falsified) via `tryRow`/try-catch. **One amplifier:** `governance.ts:75-77` returns rows when `GetCostAndUsage` throws, so a single denial takes the whole cost-governance block (categories, allocation tags, Savings Plans) with it.

**Verdict on ViewOnlyAccess: rejected.** It would satisfy the HIPAA concern as a side effect but at the cost of falsified security findings (Tier 1), lost right-sizing capability (Tier 2), and ~26 missing signals (Tier 3). It over-solves the narrow `s3:GetObject` problem by breaking 31 unrelated reads.

## Decision

**Keep `ReadOnlyAccess`. Add an explicit `Deny` on data-plane content reads across all services, with a carve-out for the CUR bucket (the one object read ProductX legitimately needs).**

An explicit `Deny` overrides the `Allow` in `ReadOnlyAccess`. ProductX reads resource **configuration and metadata** (Describe/Get/List) plus **CloudWatch metrics**, **CUR objects**, **Cost Explorer**, and (for two right-sizing recipes) **CloudWatch Logs Insights** and **Performance Insights**. It never reads customer **application data** — DynamoDB items, SQS message bodies, Kinesis records, S3 object contents (outside CUR), etc. Denying those content reads satisfies the HIPAA customer and hardens the baseline for everyone, with zero functional loss.

### Confirmed-safe Deny set

The following are data-plane content reads that ProductX provably does not use (per the engine design doc and the 191-action scanner analysis). Denying them is safe:

- **S3 object contents:** `s3:GetObject`, `s3:GetObjectVersion` (carve-out: CUR bucket) and `s3-object-lambda:GetObject`
- **DynamoDB item data:** `dynamodb:GetItem`, `dynamodb:BatchGetItem`, `dynamodb:Query`, `dynamodb:Scan`, `dynamodb:PartiQLSelect`
- **DAX cache data:** `dax:GetItem`, `dax:BatchGetItem`, `dax:Query`, `dax:Scan`
- **SQS message bodies:** `sqs:ReceiveMessage`
- **Kinesis stream data:** `kinesis:GetRecords`, `kinesis:GetShardIterator`
- **OpenSearch document data:** `es:ESHttpGet`, `es:ESHttpHead`
- **SimpleDB data:** `sdb:Select`, `sdb:GetAttributes`
- **Keyspaces (Cassandra) row data:** `cassandra:Select`
- **Timestream timeseries data:** `timestream:Select`
- **QLDB ledger data:** `qldb:GetBlock`, `qldb:GetRevision`, `qldb:GetDigest`

### Explicitly NOT denied (ProductX depends on these — denying them breaks functionality)

- `cloudwatch:GetMetricData`, `cloudwatch:GetMetricStatistics` — core engine metric source
- `logs:StartQuery`, `logs:GetQueryResults` — Lambda memory recipe (`lambda-memory.ts:542`) (Tier 2)
- `pi:GetResourceMetrics` — RDS swap evidence (`pi-metrics.ts:172`) (Tier 2)
- `ce:GetCostAndUsage` and the rest of `ce:*` — cost governance / savings (Tier 3)
- `s3:GetObject` on the CUR bucket — Billed Cost source

### Deferred pending scanner-code verification (NOT in this Deny)

Two actions could not be confirmed from the design doc alone and are **intentionally left out** of the Deny until verified against the scanner code, to avoid a Tier-1-style silent break:
- `logs:GetLogEvents`, `logs:FilterLogEvents` — confirm the scanner reads log data only via Logs Insights (`StartQuery`), not raw event reads
- `ssm:GetParameter`, `ssm:GetParameters` — confirm the scanner reads only SSM `Describe*`/`List*` metadata, not parameter values

If verification shows ProductX does not use them, they can be added to the Deny in a follow-up.

### Proposed policy

```yaml
- Sid: DenyCustomerDataPlaneReads
  Effect: Deny
  Action:
    - s3:GetObjectVersion
    - s3-object-lambda:GetObject
    - dynamodb:GetItem
    - dynamodb:BatchGetItem
    - dynamodb:Query
    - dynamodb:Scan
    - dynamodb:PartiQLSelect
    - dax:GetItem
    - dax:BatchGetItem
    - dax:Query
    - dax:Scan
    - sqs:ReceiveMessage
    - kinesis:GetRecords
    - kinesis:GetShardIterator
    - es:ESHttpGet
    - es:ESHttpHead
    - sdb:Select
    - sdb:GetAttributes
    - cassandra:Select
    - timestream:Select
    - qldb:GetBlock
    - qldb:GetRevision
    - qldb:GetDigest
  Resource: '*'
- Sid: DenyS3ObjectReadOutsideCUR
  Effect: Deny
  Action:
    - s3:GetObject
    - s3:GetObjectVersion
  NotResource:
    - 'arn:aws:s3:::productx-cur-*/*'
    - !If [HasExistingCURBucket, !Sub 'arn:aws:s3:::${ExistingCURBucket}/*', !Ref 'AWS::NoValue']
```

Two statements are used because `s3:GetObject`/`s3:GetObjectVersion` need the CUR `NotResource` carve-out, while the rest are denied on all resources. (`s3:GetObjectVersion` appears in both — the second statement's `NotResource` carve-out is the effective one for S3; listing it in the first statement is harmless but could be dropped for tidiness.)

**Critical implementation guardrail:** the Deny targets **content-read actions only**. It must NOT include any `Describe*`, `List*`, or configuration `Get*` action — in particular `s3:GetPublicAccessBlock`, `s3:GetObjectLockConfiguration`, `s3:GetEncryptionConfiguration`, `s3:GetBucketPolicy`, `dynamodb:DescribeTable`, `sqs:GetQueueAttributes`, `kinesis:DescribeStream`, `es:Describe*` — or it would strip signals the scanner needs (re-introducing the Tier 1 fabricated-"Public"-bucket class of failure).

### Scope of application

Recommend applying the Deny as the **default** for all customers, not just the HIPAA one. ProductX has no functional need to read customer application data, so denying it tightens the baseline posture at effectively zero cost and pre-empts the same objection from future customers. (See open question 2.)

## Consequences

### Positive
- Satisfies the HIPAA customer: ProductX cannot read S3 object contents (PHI) or any other customer application data (DynamoDB items, SQS messages, Kinesis records, etc.).
- Zero functional loss — WAFR security signals, right-sizing, and cost analysis rely on config/metadata reads, CloudWatch metrics, Logs Insights, PI, CE, and CUR access, none of which are affected.
- Avoids the ViewOnlyAccess breakage (Tiers 1–3 above).
- If applied as the default, closes the data-plane-read objection for all current and future customers — a strong, provable HIPAA/least-privilege talking point.
- Broader and stronger than an S3-only Deny, while remaining low-maintenance (two Deny statements vs. a 191-action custom policy).

### Negative
- ProductX still uses the broad `ReadOnlyAccess` policy, so a customer objecting to `ReadOnlyAccess` *in its entirety* (not just data reads) is not fully satisfied by this change — see Alternative B.
- The `NotResource` scoping for the bring-your-own-CUR bucket only works cleanly if that bucket name is known at deploy time; CUR can be connected asynchronously (see open question 4).
- Two data-plane reads (`logs:GetLogEvents`/`FilterLogEvents`, `ssm:GetParameter*`) remain allowed pending scanner-code verification; the Deny is not yet exhaustive.

### Neutral
- The Deny covers the customer application-data reads a HIPAA reviewer cares about (S3 objects, DynamoDB items, SQS messages, Kinesis records, etc.). It deliberately preserves the data reads ProductX depends on (CloudWatch metrics, Logs Insights, PI, Cost Explorer, CUR objects), which are operational/cost telemetry rather than customer application data.

## Alternatives Considered

### Alternative A: Switch to `ViewOnlyAccess` (+ compensating inline for the 32 broken actions)
- Rejected. Breaks 32 of 191 actions (Tiers 1–3 above). Even with compensating inline policies, it reconstructs most of the data-plane surface it was meant to remove, and every new scanner capability risks a Tier-1-style silent failure. The targeted data-plane Deny achieves the HIPAA goal far more surgically.

### Alternative B: Full custom scoped policy from the 191-action inventory
- Replace `ReadOnlyAccess` entirely with an exact-191-action custom policy. This is the correct answer **if the HIPAA customer (or any customer) rejects `ReadOnlyAccess` wholesale**, not just data-plane reads. It is now feasible because the 191-action inventory exists, but it takes on full maintenance ownership — every new AWS API the scanner calls must be added or it silently breaks. Reserve for customers whose policy forbids AWS broad managed read policies outright.

## Implementation

1. Add the `DenyCustomerDataPlaneReads` and `DenyS3ObjectReadOutsideCUR` statements (above) as an inline policy on `ProductXWAFRRole` and `ProductXWAFRMemberRole` (StackSet member template).
2. Verify the Deny lists **content-read actions only** — cross-check against configuration/metadata actions (`Describe*`, `List*`, config `Get*`) to ensure none are caught, especially the Tier 1 S3 config actions.
3. Complete scanner-code verification for the two deferred actions (`logs:GetLogEvents`/`FilterLogEvents`, `ssm:GetParameter*`); add to the Deny if unused.
4. Validate: with the Deny in place, confirm (a) CUR ingestion still works, (b) S3 public-access-block/encryption signals still populate (no fabricated "Public" findings), (c) Lambda memory recipe (Logs Insights) and RDS swap evidence (PI) still work, (d) Cost Explorer / cost-governance signals still work, (e) a DynamoDB `GetItem` / SQS `ReceiveMessage` / non-CUR `s3:GetObject` is denied.
5. Decide scope (open question 2): HIPAA-customer-only vs. default-for-all.
6. Update `README.md` (Write Permission Tracker + Security Controls — note the data-plane-read Deny), `CHANGELOG.md`, and `CUSTOMER_SECURITY_REFERENCE.md` (state explicitly that ProductX cannot read customer application data — S3 object contents, DynamoDB items, SQS messages, Kinesis records, etc. — a strong HIPAA talking point).

## Verification

- Analysis derived from actual `*Command()` usage across `scanner`, `optimizer-sync`, `cur-ingester`, `poller-aws-verify`, `api-aws`, `packages/shared`.
- 191 customer-account actions evaluated; 158 pass under ViewOnly, 32 break (Tiers 1–3).
- Tier 1 (2 actions) confirmed to produce false security findings via `storage.ts:77-91` → `signals/storage.ts:16` → `resource-catalog.ts:533`.
- Engine design doc confirms `s3:GetObject` is used only for the CUR bucket; no scanner path reads general object contents.

## Questions for Review

1. Is the HIPAA customer's objection specifically customer-data reads (→ this Deny), or `ReadOnlyAccess` in its entirety (→ Alternative B)? Confirm scope with their security team.
2. Should the data-plane Deny be the **default** for all customers (recommended) or applied only to customers who request it?
3. Should the two deferred actions (`logs:GetLogEvents`/`FilterLogEvents`, `ssm:GetParameter*`) be added to the Deny once scanner-code verification confirms they are unused? (Requires the same `*Command()` extraction, filtered to these actions.)
4. Should the 191-action inventory be checked into the repo as the basis for the Alternative B custom policy, for customers who reject `ReadOnlyAccess` outright?
5. `ExistingCURBucket` can be connected asynchronously (empty at deploy time). If it is empty at deploy, the `NotResource` only protects `productx-cur-*`. Is that acceptable, or do we require the CUR bucket name at deploy time when the Deny is active?
