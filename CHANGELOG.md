# Cloudelligent WAFR Onboarding Template — Changelog

**Template:** `Cloudelligent_WAFR_Onboarding.yaml`  
**Maintained by:** Haider Zaidi  
**Audience:** CEO, COO, Sales, Product Manager, Solutions Architects

---

## How to Read This Log

Each entry documents a change to the CloudFormation template deployed into customer AWS accounts. Entries include:

- **What changed** — technical description
- **Why** — business/security rationale
- **Customer impact** — what existing or new customers need to know
- **Sales talking point** — how to position this externally
- **Action required** — who needs to do what

---

## [2.3] — 2026-09-17

### Added: Automatic StackSet Deployment to a Chosen OU (ADR-003)

**What changed:**  
The organization StackSet now **auto-deploys** member-account roles. Added an `OrgUnitId` parameter and a `StackInstancesGroup` on `CloudelligentOnboardingMemberStackSet`. When `EnableOrganizations=yes`, the SA supplies one OU ID (a child OU `ou-xxxx-xxxxxxxx` or the org root `r-xxxx`) and the roles deploy to every account under it automatically, in the stack's region. Previously the StackSet was a shell that required an SA to add stack instances manually.

**Why:**  
The manual "add stack instances" step added onboarding friction and was error-prone (wrong OU, wrong region). Auto-deployment makes org onboarding a single stack deployment. Accounts that later join the target OU are still covered automatically via the existing `AutoDeployment: Enabled` setting.

**Customer impact:**
- New org customers: roles deploy to the chosen OU in one step; no manual StackSet work.
- Existing org customers: on the next stack update they must supply `OrgUnitId`. If they previously added instances manually to the same OU, CloudFormation reconciles to the declared target.
- Scope is controlled by which OU ID is supplied — a narrow child OU limits coverage; the root covers the whole org.

**Sales talking point:**  
> "For AWS Organizations, onboarding is a single deployment — you tell us which OU to cover and the roles roll out to every account in it automatically, including accounts you add later."

**Action required:**

| Who | Action |
|-----|--------|
| SAs | During org onboarding, obtain the target OU (or root) ID from the customer's AWS Organizations console and pass it as `OrgUnitId`. To remove, delete the main stack (clean up any failed instances first). |
| Product Manager | Note: one OU per deployment. Multi-OU customers need a stack update or a separately managed instance. |
| Sales | Position org onboarding as single-step, auto-scaling to new accounts. |

**Reference:** [ADR-003](docs/adr/ADR-003-stackset-auto-deployment.md).

---

## [2.2] — 2026-09-17

### Removed: ComputeOptimizerPolicy — Compute Optimizer Is Not Used (ADR-004)

**What changed:**  
Removed the `ComputeOptimizerPolicy` from `ProductXWAFRRole` (management account) and the `ComputeOptimizerAccess` inline policy from `ProductXWAFRMemberRole` (member accounts via StackSet). These granted 17 read-only `compute-optimizer:Get*`/`Describe*` actions. Template resource count drops from 9 to 8.

**Why:**  
An earlier draft added these permissions on the assumption that AWS Compute Optimizer would act as a fallback/supplement to the ProductX Cost Optimization Engine. The authoritative engine design (`docs/knowledge/design.md`, locked decision #2) confirms this is **not** the case: "Product X always runs its own methodology... there is no external recommendation source and no fallback routing." Compute Optimizer is not a component, dependency, or data source — the permissions were never exercised. Removing them tightens least privilege and eliminates a documentation contradiction that a security reviewer would catch.

**Customer impact:**
- New customers: smaller permission footprint to review; no Compute Optimizer permissions requested.
- Existing customers: the policy is removed on the next stack update. **No functional impact** — nothing ever called these APIs.

**Sales talking point:**  
> "ProductX runs its own proprietary optimization engine and requests only the permissions it actually uses. It does not rely on — or request access to — AWS Compute Optimizer."

**Action required:**

| Who | Action |
|-----|--------|
| SAs | Existing clients get the policy removed on their next stack update. No pre-work needed. |
| Product Manager | Confirmed: the ProductX UI does not surface Compute Optimizer data; no roadmap dependency in the next 6–12 months. |
| Sales | Drop any "we also surface Compute Optimizer recommendations" positioning — it was never accurate. |

**Reference:** [ADR-004](docs/adr/ADR-004-remove-compute-optimizer.md).

> **Note:** A prior changelog entry described `ComputeOptimizerPolicy` as a "fallback/supplement." That framing was based on an assumption later corrected by the engine design document and is superseded by this removal. Compute Optimizer was never consumed by ProductX.

---

### Clarified: Cost Optimization Engine vs. Compute Optimizer (documentation)

**What changed:**  
Updated README source-of-truth table and business use case mapping to reference the ProductX Cost Optimization Engine design document (`docs/knowledge/design.md`) as the authoritative source for FinOps permission requirements. Clarified the relationship between the proprietary engine and AWS Compute Optimizer.

**Why:**  
The engine collects CloudWatch metrics directly and runs its own proprietary methodology. It does not call AWS Compute Optimizer APIs, read its recommendations, or check enrollment. Compute Optimizer is **not a dependency, fallback, or supplement** — it is not consumed anywhere in ProductX. This distinction matters for security reviews and accurate positioning. (The dedicated `ComputeOptimizerPolicy` was subsequently removed — see the removal entry above and [ADR-004](docs/adr/ADR-004-remove-compute-optimizer.md).)

The engine's actual dependencies (all read-only, all already covered by `ReadOnlyAccess` + `ManageCURPolicy`/`ExistingCURBucketPolicy`):
- CloudWatch: `GetMetricData`, `GetMetricStatistics`, `ListMetrics`
- Resource inventory: `Describe*`/`Get*`/`List*` across EC2, ASG, ECS, Lambda, RDS, ElastiCache, DynamoDB, DocumentDB, MemoryDB, WorkSpaces, SageMaker
- Cost Explorer: `ce:GetReservationPurchaseRecommendation`, `ce:GetReservationUtilization`, `ce:DescribeNotificationSubscription`
- Cost Optimization Hub: `cost-optimization-hub:Get*`, `cost-optimization-hub:List*` (AfterDiscounts pricing mode)
- CUR via S3: `s3:GetObject` (Billed Cost source)
- Budgets: `budgets:ViewBudget`, `budgets:Describe*`
- Support: `support:DescribeTrustedAdvisorCheckResult`, `support:DescribeServices`, `support:DescribeSupportLevel`

**Customer impact:** None — documentation only.

**Sales talking point:**  
> "ProductX runs its own proprietary cost optimization engine — it doesn't rely on AWS Compute Optimizer. It collects metrics directly from CloudWatch and applies a deeper methodology. It does not request or use Compute Optimizer permissions."

**Action required:**

| Who | Action |
|-----|--------|
| Sales | Position the engine as proprietary and self-contained; do not claim Compute Optimizer integration |
| Product Manager | Reference `docs/knowledge/design.md` as the canonical source for engine capabilities and coverage |
| SAs | When customers ask "do we need to enable Compute Optimizer?" — no, ProductX works independently and does not use it |

---

## [2.1] — 2026-09-03

### Added: Explicit Deny on Customer Application-Data Reads (`DataPlaneReadDeny`, ADR-005)

**What changed:**  
`ProductXWAFRRole` (management account) and `ProductXWAFRMemberRole` (member accounts via StackSet) now carry a new inline policy, `DataPlaneReadDeny`, with two `Deny` statements:
- `DenyCustomerDataPlaneReads` — denies content reads across DynamoDB/DAX (items), SQS (message bodies), Kinesis (records), OpenSearch, SimpleDB, Keyspaces, Timestream, QLDB, and S3 Object Lambda.
- `DenyS3ObjectReadOutsideCUR` — denies `s3:GetObject`/`s3:GetObjectVersion` everywhere **except** the CUR buckets ProductX legitimately reads (`productx-cur-*`, `reserved-ai-cur-*`, `reserved-ai-spot-*`, and a supplied `ExistingCURBucket`).

An explicit IAM `Deny` overrides the `Allow` in the `ReadOnlyAccess` managed policy, so this is a hard guarantee, not a convention. The Deny targets **content-read actions only** — it does not touch configuration/metadata reads (`Describe*`/`List*`/config `Get*`), CloudWatch metrics, CloudWatch Logs Insights, Performance Insights, or Cost Explorer, all of which ProductX depends on.

**Why:**  
A HIPAA-covered prospect objected that `ReadOnlyAccess` grants `s3:GetObject` (potential PHI exposure). Analysis confirmed ProductX has **no functional need** to read customer application data — the WAFR scanner reads resource *configuration*, and the only object read it needs is the CUR (cost) data, which is granted by dedicated prefix-scoped policies, not `ReadOnlyAccess`. We therefore deny customer-data reads outright and apply it as the **default for all customers**, not just the HIPAA one.

**Customer impact:**
- New customers: ProductX is provably unable to read application data (S3 object contents outside CUR, database items, queue messages, stream records) — enforced by IAM, verifiable in the template.
- Existing customers: Need a stack update to apply the new Deny policy. **No loss of ProductX functionality** — WAFR findings, cost analytics, and right-sizing all rely on config/metadata, metrics, and CUR access, none of which are affected.
- HIPAA/regulated customers: This directly answers the "can your tool read our PHI?" question with "no — it is denied at the IAM level."

**Sales talking point:**  
> "ProductX cannot read your application data — not your S3 object contents, database records, queue messages, or stream data. This is enforced by an explicit IAM Deny that overrides the AWS read policy, so it's a hard technical guarantee your security team can verify in the template. ProductX reads only resource configuration, performance metrics, and your cost data."

**Action required:**

| Who | Action |
|-----|--------|
| SAs | Existing clients need a stack update to apply `DataPlaneReadDeny`. After deploy, validate via IAM Policy Simulator that config reads / metrics / CUR still work and object/item reads are denied. |
| Product Manager | Confirm no scanner code path reads `logs:GetLogEvents`/`FilterLogEvents` or `ssm:GetParameter*` values (raw log events / SecureString params). If unused, they will be added to the Deny in a follow-up (ADR-005 open question 2). |
| Sales | Use the data-plane Deny as a security differentiator, especially for HIPAA/regulated prospects. |

**Reference:** [ADR-005](docs/adr/ADR-005-deny-customer-data-plane-reads.md) — full 191-action analysis, the rejected ViewOnlyAccess alternative, and the CUR carve-out rationale.

---

## [2.0] — 2026-08-27

### Changed: Cloudviz Role Now Uses a Custom Scoped Policy (was ReadOnlyAccess)

**What changed:**  
Both Cloudviz roles — `CloudvizArchitectureDiagramRole` (management account) and `CloudvizMemberRole` (member accounts via StackSet) — now use a custom inline policy (`CloudvizReadOnlyAccess`, 52 targeted Describe/List/Get actions) instead of the broad AWS `ReadOnlyAccess` managed policy. Source: `docs/knowledge/cloudviz_custom_readonly_policy.json`.

**Why:**  
Cloudviz only needs to enumerate resource types and their configuration to draw architecture diagrams (EC2, RDS, Lambda, ECS, EKS, ELB, S3 metadata, etc.). It does not need the full read surface of `ReadOnlyAccess`, and in particular does not need data-plane reads. Scoping to the minimum required actions tightens least privilege and reduces the permission footprint customers must review.

**Customer impact:**
- New customers: Cloudviz role is now provably limited to diagram-relevant read actions
- Existing customers: Need a stack update to move from `ReadOnlyAccess` to the scoped policy

**Sales talking point:**  
> "Our diagram tool uses a purpose-built policy limited to ~50 read actions needed to map your architecture — not a broad read-all policy. It cannot read your application data."

**Action required:**

| Who | Action |
|-----|--------|
| SAs | Existing clients need a stack update to apply the scoped Cloudviz policy |
| Product Manager | Confirm Cloudviz diagram generation works with the scoped action list; add actions if any resource type fails to render |
| Sales | Update security responses — Cloudviz uses a custom minimal policy, not `ReadOnlyAccess` |

---

### Added: WAFRWorkloadManagement Inline Policy

**What changed:**  
Added the `WAFRWorkloadManagement` inline policy to `ProductXWAFRRole` (management account) and `ProductXWAFRMemberRole` (member accounts via StackSet). This policy grants write access to the AWS Well-Architected Tool to create workloads, record assessment answers, manage lenses, create milestones, and tag workloads.

**Why:**  
This was present in the standalone `ProductX_WAFR_Workload_ComputeOptimizer_Role.yaml` template but was never carried over to the combined `Cloudelligent_WAFR_Onboarding.yaml`. Without it, ProductX cannot create or manage WAFR workloads in the customer's AWS account — which is the core product function.

Gap identified via business use case analysis against CoreStack capabilities:
- CoreStack's "Compass" module used these same `wellarchitected:*` write permissions for WAF Sync
- ProductX replicates this with scoped permissions (management account scoped to account-level ARNs)

**Customer impact:**
- New customers: WAFR assessments will work end-to-end from day one
- Existing customers (if any deployed the combined template): Need stack update to gain WAFR write capability

**Sales talking point:**  
> "The only write actions ProductX takes in your account are within the AWS Well-Architected Tool — creating assessment workloads and recording findings. It cannot touch your infrastructure."

**Action required:**

| Who | Action |
|-----|--------|
| SAs | Any clients deployed with the combined template before this date need a stack update |
| Product Manager | Confirm ProductX WAFR assessment flow works end-to-end with the updated template |
| Sales | Update security responses — write is limited to Well-Architected Tool workloads only |

---

### Pending: CUR Ownership Decision (ADR-001)

**What:** [ADR-001](docs/adr/ADR-001-cur-ownership.md) proposes removing `ManageCURPolicy` from ProductX and letting Archera own CUR creation exclusively. Decision is pending stakeholder review.

**Current state:** `ManageCURPolicy` remains in the template. ProductX can create its own CUR (`productx-cur-*`) OR read from Archera's bucket via `ExistingCURBucket`.

**Action required:**

| Who | Action |
|-----|--------|
| CEO/COO/PM/Sales/SAs | Review ADR-001 and decide: should ProductX keep CUR write, or delegate to Archera? |

---

### Updated: `ExistingCURBucket` Parameter Description

**What changed:**  
Parameter description clarified to explain it enables ProductX to read an existing CUR bucket (e.g. Archera's `reserved-ai-cur-<uuid>.<first-10-digits-of-mgmt-account-id>`). This is optional — ProductX can also create its own CUR via `ManageCURPolicy`, or the CUR can be connected later via the ProductX Cost Explorer UI.

**Customer impact:** None — parameter behavior unchanged.

**Action required:**

| Who | Action |
|-----|--------|
| SAs | `ExistingCURBucket` is optional at deploy time. Fill it if Archera is already onboarded; otherwise connect CUR later in ProductX |

---

### Added: Template README with Write Permission Tracker

**What changed:**  
Created `README.md` documenting every component, permission source, and — critically — a line-by-line tracker of every write permission with justification, scope, and risk level.

**Why:**  
Multiple audiences (customer security teams, AWS partner reviewers, internal sales) were asking "what can this role do?" without a single reference document. This eliminates back-and-forth and speeds up security review approvals.

**Customer impact:** Can be shared directly with customer security/engineering teams during pre-sales or onboarding.

**Sales talking point:**  
> "Here's our full permission breakdown — every write action is documented with its scope and justification. We're transparent by design."

**Action required:**

| Who | Action |
|-----|--------|
| Sales | Bookmark README for security questionnaire responses and customer-facing calls |
| SAs | Reference README when customers ask "what does this stack do?" |
| Product Manager | Review and flag if any permissions are missing or outdated |

---

## [1.0] — 2026-08-19

### Initial: Combined Onboarding Template

**What changed:**  
Created the unified `Cloudelligent_WAFR_Onboarding.yaml` combining:
- ProductX WAFR role (replacing old CoreStack `cloudelligent-assessment-role` for WAFR)
- Cloudviz architecture diagram role
- Billing read-only role
- Archera ReservedAI role
- Service-managed StackSet for org member accounts

**Why:**  
Single stack replaces the old CoreStack template (`Corestack_FinOps_read_access.yml`). Eliminates dependency on CoreStack platform (discontinuation planned). Consolidates 4 separate role deployments into one customer-facing stack.

**Customer impact:**
- New customers deploy one stack instead of multiple
- Per-client External ID (confused-deputy protection) replaces the old shared hardcoded ID
- Org rollout uses a service-managed StackSet shell (SA manually adds stack instances to target OUs/accounts)

> **Superseded by [2.3]:** As of v2.3 the StackSet auto-deploys to a chosen OU via the `OrgUnitId` parameter — the manual "add stack instances" step described here no longer applies. See the [2.3] entry.

**Sales talking point:**  
> "One CloudFormation stack covers your entire AWS Organization. Deploy once in the management account, add member accounts to the StackSet, and every account is connected — no per-account role setup." _(Note: as of v2.3, member accounts are covered automatically via the `OrgUnitId` you provide — no manual StackSet step.)_

**Action required:**

| Who | Action |
|-----|--------|
| SAs | Stop using `Corestack_FinOps_read_access.yml` for new clients |
| Sales | Update pitch deck — single stack deployment, per-client security isolation |
| CEO/COO | CoreStack contract can be terminated after existing client migration |
| Product Manager | Ensure ProductX UI generates correct template URL for this new stack |

---

## Upcoming (Planned)

| Target Date | Change | Status |
|---|---|---|
| Post-cutoff | Migration plan for existing CoreStack clients → new template | Not started |
| TBD | Evaluate removing `ComplianceSignalAccess` inline policy (redundant with ReadOnlyAccess) | ADR-002 Proposal 1 — pending review |
| TBD | Evaluate consolidating `BillingReadOnlyRole` into ProductX role | ADR-002 Proposal 4 — pending finance input |
| TBD | Evaluate scoping `wellarchitected:CreateWorkload` to specific resource tags | Research |
