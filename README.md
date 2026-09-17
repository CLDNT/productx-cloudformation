# Cloudelligent Discovery Role — Template Reference

**Template:** `Cloudelligent_WAFR_Onboarding.yaml`  
**Version:** 2.4  
**Last Updated:** 2026-09-17  
**Owner:** Cloudelligent Product X Team

---

## Purpose

This CloudFormation template is deployed into each customer's AWS Management Account during onboarding. It creates read-only cross-account IAM roles that allow Cloudelligent's tooling to:

1. Run AWS Well-Architected Framework (WAFR) reviews (ProductX)
2. Read cost data for FinOps visibility (ProductX + Archera)
3. Provision CUR exports for cost analytics (ProductX + Archera)
4. Perform read-only resource discovery for diagram generation (Cloudviz)
5. Provide billing analytics (internal finance)

> **Note:** RI/SP *purchasing* permissions are deployed via Archera's own separate template (`docs/knowledge/archera-cf-template.yaml`), not this stack.

> **De-branding (ADR-006):** The customer-facing template is de-branded — its `Description`, comments, and parameter/output descriptions do not name third-party tools; logical IDs use a neutral `Cloudelligent*` scheme. This README is the internal reference and retains vendor names (Cloudviz, Archera) for clarity. **Logical ID ≠ deployed RoleName**: only the Cloudviz role's `RoleName` was changed (to `CloudelligentDiscoveryRole`); all other deployed `RoleName` values are unchanged to avoid breaking integrations.

For AWS Organizations customers, a service-managed StackSet automatically deploys member-account roles to a single OU (or the org root) supplied by the SA via the `OrgUnitId` parameter. Accounts joining that OU later are covered automatically.

---

## Template Components

| # | Resource (Logical ID) | IAM Role Name | Trusted Principal | Business Use Case | Access Level |
|---|---|---|---|---|---|
| 1 | `CloudelligentWAFRRole` | `ProductXWAFRRole` | Cloudelligent ProductX account (`820440428496`) | WAFR assessments, cost optimization engine, compliance scanning, cost data read, workload management | **Read-only** (config/metadata + CUR) + WAFR workload management. Explicit `Deny` blocks all customer application-data reads (see `DataPlaneReadDeny`, ADR-005). |
| 2 | `AllowAssumeMemberRoles` | _(policy on #1, name `ProductXWAFRAssumeMemberRoles`)_ | — | Hub-spoke: lets the ProductX role in the mgmt account assume member-account roles | Read (sts:AssumeRole) |
| 3 | `ManageCURPolicy` | _(policy on #1, name `ProductXManageCUR`)_ | — | Create and manage CUR export + S3 delivery bucket for cost visibility | **Write** (scoped to `productx-cur-*`) |
| 4 | `ExistingCURBucketPolicy` | _(policy on #1, name `ProductXReadExistingCURBucket`)_ | — | Read CUR data from an existing S3 bucket (e.g. Archera's) for cost visibility | **Read-only** |
| 5 | `CloudelligentDiscoveryRole` | `CloudelligentDiscoveryRole` | Cloudviz account (`282762468439`) | Read-only resource discovery for architecture diagram generation | **Read-only** (`CloudelligentDiscoveryReadOnlyAccess`, 52 actions) |
| 6 | `CloudelligentBillingRole` | `BillingReadOnlyRole` | Cloudelligent finance (`898668804275`) | Internal billing analytics, invoice access, Trusted Advisor checks | **Read-only** |
| 7 | `CloudelligentFinOpsRole` | `ReservedAI-Write` | Archera (`967800896805`) | Cost visibility, CUR creation, spot data feeds | **Read + Write** (scoped) |
| 8 | `CloudelligentOnboardingMemberStackSet` | _(StackSet)_ | — | Auto-deploys member-account roles to the OU supplied in `OrgUnitId` (current + future accounts) | N/A (deployment vehicle) |

> **Logical ID vs. RoleName:** the left column is the CloudFormation **logical ID** (de-branded per ADR-006). The **IAM Role Name** column is what is actually created in the customer account. Only the Cloudviz role's deployed name changed (`cloudelligent-assessment-role` → `CloudelligentDiscoveryRole`); the ProductX, Billing, and Archera roles keep their existing deployed names to avoid breaking third-party/internal integrations that reference them.

---

## Source of Truth

| Component | Permission Source | Business Use Case Source |
|---|---|---|
| ProductX WAFR Role (read) | [AWS ReadOnlyAccess Policy](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/ReadOnlyAccess.html) + [SecurityAudit Policy](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/SecurityAudit.html) | All 4 use cases: WAFR, FinOps, Cost Analytics, SecOps |
| WAFRWorkloadManagement (write) | [AWS Well-Architected Tool API](https://docs.aws.amazon.com/wellarchitected/latest/APIReference/) | WAFR Assessments — create workloads, record answers, sync to AWS WAT |
| ComplianceSignalAccess | [AWS Config API](https://docs.aws.amazon.com/config/latest/APIReference/) + [Security Hub API](https://docs.aws.amazon.com/securityhub/1.0/APIReference/) + [IAM Access Analyzer API](https://docs.aws.amazon.com/access-analyzer/latest/APIReference/) | Compliance Standards (SecOps) — CIS Benchmark evidence; WAFR Security Pillar |
| CUR Read (ExistingCURBucket) | [ADR-001](docs/adr/ADR-001-cur-ownership.md) | Cost Analytics — read existing CUR bucket (optional, bring-your-own) |
| CUR Write (ManageCURPolicy) | [AWS CUR API](https://docs.aws.amazon.com/cur/latest/userguide/what-is-cur.html) + [S3 API](https://docs.aws.amazon.com/AmazonS3/latest/API/) | Cost Analytics — create CUR export + delivery bucket (`productx-cur-*`) |
| Cost Optimization Engine (via ReadOnlyAccess) | [ProductX Cost Optimization Engine Design](docs/knowledge/design.md) — Part VI Dependencies | Optimize Usage (FinOps) — rightsizing, idle, orphaned, scheduling, optimize configuration, optimize rate |
| Cost Explorer API (via ReadOnlyAccess) | [Cost Explorer API](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/) | Cost Analytics — spend trends, RI utilization, anomaly detection; Engine savings/pricing |
| Cost Optimization Hub (via ReadOnlyAccess) | [Cost Optimization Hub API](https://docs.aws.amazon.com/cost-optimization-hub/latest/APIReference/) | Engine AfterDiscounts pricing mode (discount-aware RI/SP rates) |
| Discovery Role (Cloudviz) | Custom scoped read-only policy `CloudelligentDiscoveryReadOnlyAccess` (`docs/knowledge/cloudviz_custom_readonly_policy.json`) — vendor-specified minimum for diagram generation | Architecture diagram generation for client deliverables |
| BillingReadOnlyRole | Cloudelligent internal finance requirements | Monthly cost review per client (WAF Cost Optimization pillar) |
| FinOps Role (Archera) | [Archera CF Template](docs/knowledge/archera-cf-template.yaml) — vendor-specified | Cost visibility for RI/SP analysis + CUR provisioning. Note: RI/SP *purchasing* permissions are in Archera's full template, not this stack. |

### Business Use Case to Permission Mapping

| Business Use Case | AWS Services Read | Write Permissions | Covered By |
|---|---|---|---|
| **WAFR Assessments** — identify HRIs/MRIs, redeem AWS funding | EC2, RDS, S3, IAM, Config, Security Hub, GuardDuty, KMS, Lambda, ELB, CloudTrail, 50+ services | `wellarchitected:Create/Update/Delete` (workloads only) | `ReadOnlyAccess` + `SecurityAudit` + `ComplianceSignalAccess` + `WAFRWorkloadManagement` |
| **Optimize Usage (FinOps)** — rightsizing, idle, orphaned, scheduling, optimize config/rate | CloudWatch (`GetMetricData`, `GetMetricStatistics`, `ListMetrics`), EC2, ASG, ECS, Lambda, RDS, ElastiCache, DynamoDB, DocumentDB, MemoryDB, WorkSpaces, SageMaker, Cost Explorer, Cost Optimization Hub, Budgets, Support/Trusted Advisor | None — engine is strictly read-only; remediation is out of scope | `ReadOnlyAccess` + `ManageCURPolicy` or `ExistingCURBucketPolicy` (CUR for Billed Cost) |
| **Cost Analytics** — tag-based cost breakdown, daily reports | S3 (CUR bucket), Cost Explorer (`ce:*`), Budgets, Billing | CUR/S3 provisioning (`productx-cur-*`) via `ManageCURPolicy` | `ReadOnlyAccess` + `ManageCURPolicy` + `ExistingCURBucketPolicy` (optional) |
| **Compliance Standards (SecOps)** — CIS Benchmark, FTR | GuardDuty, Inspector, Config, Security Hub, IAM, KMS, CloudTrail, VPC, S3, EC2 | None | `ReadOnlyAccess` + `SecurityAudit` + `ComplianceSignalAccess` |

> **Note on AWS Compute Optimizer:** The ProductX Cost Optimization Engine does **not** use AWS Compute Optimizer. It collects CloudWatch metrics directly and applies its own proprietary methodology — there is no external recommendation source and no fallback routing. The template therefore grants **no** dedicated `compute-optimizer:*` permissions. (A `ComputeOptimizerPolicy` existed in earlier drafts on the assumption of a fallback that never materialized; it was removed per [ADR-004](docs/adr/ADR-004-remove-compute-optimizer.md). Note: `ReadOnlyAccess` still nominally includes Compute Optimizer read actions, but nothing in ProductX calls them.)

---

## Write Permission Tracker

Every write permission in this template is documented below. **ProductX has zero infrastructure write permissions** — it cannot create, modify, or delete any customer resource (EC2, RDS, VPC, etc.). Its write access is limited to the AWS Well-Architected Tool for workload management and CUR/S3 bucket provisioning for cost data. All other S3/CUR write permissions belong to the Archera role.

### Summary

| Role | Total Permissions | Read-Only | Write | Write Justification |
|---|---|---|---|---|
| ProductXWAFRRole | 2 managed policies (`ReadOnlyAccess`, `SecurityAudit`) + 36 inline actions (ComplianceSignalAccess + WAFRWorkloadManagement) + ManageCURPolicy + `DataPlaneReadDeny` (24 explicit Deny) | All read except 11 WAFR workload management actions + 5 CUR/S3 provisioning actions; customer application-data reads explicitly denied (see `DataPlaneReadDeny`) | 11 Well-Architected Tool actions + 5 CUR/S3 write actions | WAFR workload management + CUR bucket creation for cost visibility |
| CloudelligentDiscoveryRole (Cloudviz) | Custom inline policy (`CloudelligentDiscoveryReadOnlyAccess`, 52 scoped read actions) | All | None | — |
| BillingReadOnlyRole | 47 inline actions | All | None | — |
| ReservedAI-Write | 84 inline actions | 81 read | 3 write (`s3:CreateBucket`, `s3:PutBucketPolicy`, `cur:PutReportDefinition`) | CUR provisioning — create S3 bucket for cost data delivery and define the CUR export |

### Detailed Write Permissions

#### ProductXWAFRRole — Well-Architected Tool (write)

These are required for ProductX to create and manage WAFR workloads in the customer's own AWS Well-Architected Tool console.

| # | Action | Resource Scope | Business Justification | Risk Level | Reversible |
|---|---|---|---|---|---|
| 1 | `wellarchitected:CreateWorkload` | `*` | Create WAFR workloads for assessment | Low | Yes (delete workload) |
| 2 | `wellarchitected:UpdateWorkload` | `*` | Update workload metadata during review | Low | Yes |
| 3 | `wellarchitected:DeleteWorkload` | `*` | Clean up completed/abandoned workloads | Low | No (data loss) |
| 4 | `wellarchitected:AssociateLenses` | `arn:aws:wellarchitected:*:<account>:workload/*` | Attach WAFR lens to workload | Low | Yes (disassociate) |
| 5 | `wellarchitected:DisassociateLenses` | `arn:aws:wellarchitected:*:<account>:workload/*` | Remove lens from workload | Low | Yes (re-associate) |
| 6 | `wellarchitected:UpgradeLensReview` | `arn:aws:wellarchitected:*:<account>:workload/*` | Upgrade to latest lens version | Low | No |
| 7 | `wellarchitected:UpdateLensReview` | `arn:aws:wellarchitected:*:<account>:workload/*` | Save review progress | Low | Yes |
| 8 | `wellarchitected:UpdateAnswer` | `arn:aws:wellarchitected:*:<account>:workload/*` | Record answers to WAFR pillar questions | Low | Yes (update again) |
| 9 | `wellarchitected:CreateMilestone` | `arn:aws:wellarchitected:*:<account>:workload/*` | Snapshot workload state at a point in time | Low | No (immutable) |
| 10 | `wellarchitected:TagResource` | `arn:aws:wellarchitected:*:<account>:workload/*` | Tag workloads with client/project metadata | Low | Yes (untag) |
| 11 | `wellarchitected:UntagResource` | `arn:aws:wellarchitected:*:<account>:workload/*` | Remove incorrect tags | Low | Yes (re-tag) |

**Note:** `wellarchitected:ListTagsForResource` is also granted on the same resource scope but is a read-only action (included alongside tagging for completeness in the IAM statement).

**Note:** These actions ONLY affect the Well-Architected Tool service. They cannot modify any customer infrastructure (EC2, RDS, S3, etc.).

#### ProductXWAFRRole — CUR/S3 Provisioning (write)

These allow ProductX to create a Cost & Usage Report and its delivery bucket for cost visibility in the ProductX Cost Explorer.

| # | Action | Resource Scope | Business Justification | Risk Level | Reversible |
|---|---|---|---|---|---|
| 12 | `cur:PutReportDefinition` | `arn:aws:cur:*:*:definition/productx-*` | Create CUR export for cost data | Low | Yes (delete report) |
| 13 | `cur:ModifyReportDefinition` | `arn:aws:cur:*:*:definition/productx-*` | Update CUR configuration | Low | Yes |
| 14 | `cur:DeleteReportDefinition` | `arn:aws:cur:*:*:definition/productx-*` | Clean up CUR if no longer needed | Low | No (data stops) |
| 15 | `s3:CreateBucket` | `arn:aws:s3:::productx-cur-*` | Create delivery bucket for CUR data | Low | Yes (delete bucket) |
| 16 | `s3:PutBucketPolicy` | `arn:aws:s3:::productx-cur-*` | Allow AWS billing service to deliver CUR to bucket | Low | Yes (update policy) |

**Scope constraint:** CUR definitions are locked to `productx-*` prefix. S3 bucket creation is locked to `productx-cur-*` prefix. ProductX CANNOT create or modify any other CUR or S3 bucket.

**Note:** [ADR-001](docs/adr/ADR-001-cur-ownership.md) proposes transferring CUR ownership to Archera and removing this policy. Decision pending stakeholder review.

#### ReservedAI-Write (Archera) — CUR Provisioning & Cost Data Access

| # | Action | Resource Scope | Business Justification | Risk Level | Reversible |
|---|---|---|---|---|---|
| 1 | `s3:CreateBucket` | `arn:aws:s3:::reserved-ai-cur-*`, `arn:aws:s3:::reserved-ai-spot-*` | Create CUR delivery bucket for cost data export | Low | Yes (delete bucket) |
| 2 | `s3:PutBucketPolicy` | `arn:aws:s3:::reserved-ai-cur-*`, `arn:aws:s3:::reserved-ai-spot-*` | Allow AWS billing service to deliver CUR to bucket | Low | Yes (update policy) |
| 3 | `cur:PutReportDefinition` | `arn:aws:cur:*:*:definition/reserved-ai-*` | Create the Cost & Usage Report export definition | Low | Yes (delete report) |

**Scope constraint:** Bucket names are locked to `reserved-ai-cur-*` and `reserved-ai-spot-*` prefixes. The role CANNOT create or modify any other S3 bucket.

#### ReservedAI-Write (Archera) — IAM Self-Inspection

| # | Action | Resource Scope | Business Justification | Risk Level | Reversible |
|---|---|---|---|---|---|
| 4 | `iam:GetRolePolicy` | `arn:aws:iam::*:role/ReservedAI-Write` | Archera validates its own permissions are intact | None | Read-only operation |
| 5 | `iam:ListRolePolicies` | `arn:aws:iam::*:role/ReservedAI-Write` | Same | None | Read-only operation |
| 6 | `iam:ListAttachedRolePolicies` | `arn:aws:iam::*:role/ReservedAI-Write` | Same | None | Read-only operation |
| 7 | `iam:GetPolicy` | `arn:aws:iam::*:role/ReservedAI-Write` | Same | None | Read-only operation |
| 8 | `iam:GetPolicyVersion` | `arn:aws:iam::*:role/ReservedAI-Write` | Same | None | Read-only operation |
| 9 | `iam:SimulatePrincipalPolicy` | `arn:aws:iam::*:role/ReservedAI-Write` | Archera tests whether it can perform required actions | None | Read-only operation |

**Note:** These IAM actions are scoped exclusively to the `ReservedAI-Write` role itself. Archera cannot read or modify any other IAM role, user, or policy.

---

## Security Controls

| Control | Implementation |
|---|---|
| Confused-deputy protection | ProductX: per-client `ExternalId` (unique, issued by platform). Archera: static ExternalId. Cloudviz: static ExternalId. |
| Least privilege | ProductX CUR/S3 write is prefix-scoped (`productx-cur-*`). Archera write is prefix-scoped (`reserved-ai-*`). All roles deny by default. |
| Data-plane read Deny (ADR-005) | `ProductXWAFRRole` (and its member-account role) carry an explicit `DataPlaneReadDeny` policy that denies all customer application-data reads — S3 object contents (except CUR buckets), DynamoDB/DAX items, SQS messages, Kinesis records, OpenSearch/SimpleDB/Keyspaces/Timestream/QLDB data. Explicit Deny overrides `ReadOnlyAccess`. ProductX reads only resource **configuration/metadata**, CloudWatch metrics, Cost Explorer, and CUR objects. |
| No long-term credentials | Cross-account assumed roles only. No access keys. |
| Blast radius isolation | Each role trusts exactly one principal. Compromise of one platform cannot escalate to another. |
| Org deployment | Service-managed StackSet auto-deploys member-account roles to the OU (or root) supplied in `OrgUnitId`, in the stack's own region. `AutoDeployment: Enabled` covers accounts that later join the OU. To remove member roles, delete the main stack (which removes the stack instances); failed instances must be cleaned up first. |
| Auditability | All role assumptions logged in CloudTrail. Role names are deterministic for easy log filtering. |

---

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `TrustedAccountId` | No | `820440428496` | ProductX platform AWS account. Do not change unless directed. |
| `ExternalId` | Yes | _(none)_ | Unique per-customer identifier issued by ProductX. Prevents confused deputy. |
| `EnableOrganizations` | No | `no` | Set `yes` for multi-account orgs. Deploys the StackSet and auto-deploys member-account roles to `OrgUnitId`. |
| `OrgUnitId` | Only if `EnableOrganizations=yes` | _(empty)_ | The single AWS Organizations root (`r-xxxx`) or organizational unit (`ou-xxxx-xxxxxxxx`) to deploy member-account roles into. Roles auto-deploy to all current accounts in this OU and to any that later join it. |
| `ExistingCURBucket` | No | _(empty)_ | Optional. Name of an existing CUR bucket to read (e.g. Archera's `reserved-ai-cur-<uuid>.<10-digit-acct-id>`). Leave empty if ProductX will create its own CUR via `ManageCURPolicy`, or connect later via the ProductX Cost Explorer UI. |

---

## Deployment Modes

| Mode | Stack | Roles Created | Use Case |
|---|---|---|---|
| Single Account | Main stack only | `ProductXWAFRRole`, `CloudelligentDiscoveryRole` (Cloudviz), `BillingReadOnlyRole`, `ReservedAI-Write` (Archera) | Client with one AWS account |
| Organization | Main stack + StackSet (auto-deploys instances to `OrgUnitId`) | All above + `ProductXWAFRMemberRole` + a member `CloudelligentDiscoveryRole` in every account under the target OU | Client with AWS Organizations |

**Organization mode workflow:**
1. SA obtains the target OU ID (`ou-xxxx-xxxxxxxx`) or org root ID (`r-xxxx`) from the AWS Organizations console.
2. Deploy main stack with `EnableOrganizations=yes` and `OrgUnitId=<that ID>`.
3. The StackSet automatically deploys member-account roles to every account currently under that OU, in the stack's region.
4. New accounts that later join the OU receive roles automatically (`AutoDeployment: Enabled`).
5. To remove: delete the main stack (this removes the stack instances). If any instance is in a failed state, clean it up first or the delete will fail.

---

## What This Template Does NOT Do

- ❌ Create, modify, or delete any customer infrastructure (EC2, RDS, VPC, etc.)
- ❌ Create S3 buckets outside the `productx-cur-*` namespace (CUR delivery only)
- ❌ Access customer application data — S3 object contents (beyond CUR), DynamoDB/DAX items, SQS messages, Kinesis records, OpenSearch/SimpleDB/Keyspaces/Timestream/QLDB data — **enforced by an explicit IAM `Deny` (`DataPlaneReadDeny`, ADR-005), not just convention**
- ❌ Modify IAM policies, users, or roles (beyond Archera's self-inspection of its own role)
- ❌ Purchase Reserved Instances or Savings Plans (that requires Archera's full template, deployed separately)
- ❌ Send data outside of AWS (all operations are AWS API calls within the customer's account)

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for detailed version history with stakeholder context and action items.
