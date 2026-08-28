# Cloudelligent WAFR Onboarding Stack — Template Reference

**Template:** `Cloudelligent_WAFR_Onboarding.yaml`  
**Version:** 2.0  
**Last Updated:** 2026-08-27  
**Owner:** Cloudelligent Product X Team

---

## Purpose

This CloudFormation template is deployed into each customer's AWS Management Account during onboarding. It creates the IAM roles that allow Cloudelligent's tooling to:

1. Run AWS Well-Architected Framework reviews (ProductX)
2. Read cost data for FinOps visibility (ProductX + Archera)
3. Provision CUR exports for cost analytics (ProductX + Archera)
4. Generate architecture diagrams (Cloudviz)
5. Provide billing analytics (internal finance)

> **Note:** RI/SP *purchasing* permissions are deployed via Archera's own separate template (`docs/knowledge/archera-cf-template.yaml`), not this stack.

For AWS Organizations customers, a service-managed StackSet is created as a shell — an SA must manually add stack instances (targeting OUs or accounts) to deploy member-account roles.

---

## Template Components

| # | Resource (Logical ID) | IAM Role Name | Trusted Principal | Business Use Case | Access Level |
|---|---|---|---|---|---|
| 1 | `ProductXWAFRRole` | `ProductXWAFRRole` | Cloudelligent ProductX account (`820440428496`) | WAFR assessments, cost optimization engine, compliance scanning, cost data read, workload management | **Read-only** + WAFR workload management |
| 2 | `AllowAssumeMemberRoles` | _(policy on #1)_ | — | Hub-spoke: lets ProductX role in mgmt account assume member account roles | Read (sts:AssumeRole) |
| 3 | `ManageCURPolicy` | _(policy on #1)_ | — | Create and manage CUR export + S3 delivery bucket for cost visibility | **Write** (scoped to `productx-cur-*`) |
| 4 | `ExistingCURBucketPolicy` | _(policy on #1)_ | — | Read CUR data from an existing S3 bucket (e.g. Archera's) for cost visibility | **Read-only** |
| 5 | `CloudvizArchitectureDiagramRole` | `cloudelligent-assessment-role` | Cloudviz account (`282762468439`) | Auto-generate architecture diagrams from live AWS resources | **Read-only** |
| 6 | `BillingReadOnlyRole` | `BillingReadOnlyRole` | Cloudelligent finance (`898668804275`) | Internal billing analytics, invoice access, Trusted Advisor checks | **Read-only** |
| 7 | `ReservedAIRole` | `ReservedAI-Write` | Archera (`967800896805`) | Cost visibility, CUR creation, spot data feeds | **Read + Write** (scoped) |
| 8 | `CloudelligentOnboardingMemberStackSet` | _(StackSet)_ | — | Creates StackSet shell for member account role deployment (SA manually adds stack instances) | N/A (deployment vehicle) |

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
| Cloudviz Role | Cloudviz platform requirements (vendor-specified) | Architecture diagram generation for client deliverables |
| BillingReadOnlyRole | Cloudelligent internal finance requirements | Monthly cost review per client (WAF Cost Optimization pillar) |
| ReservedAI Role (Archera) | [Archera CF Template](docs/knowledge/archera-cf-template.yaml) — vendor-specified | Cost visibility for RI/SP analysis + CUR provisioning. Note: RI/SP *purchasing* permissions are in Archera's full template, not this stack. |

### Business Use Case to Permission Mapping

| Business Use Case | AWS Services Read | Write Permissions | Covered By |
|---|---|---|---|
| **WAFR Assessments** — identify HRIs/MRIs, redeem AWS funding | EC2, RDS, S3, IAM, Config, Security Hub, GuardDuty, KMS, Lambda, ELB, CloudTrail, 50+ services | `wellarchitected:Create/Update/Delete` (workloads only) | `ReadOnlyAccess` + `SecurityAudit` + `ComplianceSignalAccess` + `WAFRWorkloadManagement` |
| **Optimize Usage (FinOps)** — rightsizing, idle, orphaned, scheduling, optimize config/rate | CloudWatch (`GetMetricData`, `GetMetricStatistics`, `ListMetrics`), EC2, ASG, ECS, Lambda, RDS, ElastiCache, DynamoDB, DocumentDB, MemoryDB, WorkSpaces, SageMaker, Cost Explorer, Cost Optimization Hub, Budgets, Support/Trusted Advisor | None — engine is strictly read-only; remediation is out of scope | `ReadOnlyAccess` + `ManageCURPolicy` or `ExistingCURBucketPolicy` (CUR for Billed Cost) |
| **Cost Analytics** — tag-based cost breakdown, daily reports | S3 (CUR bucket), Cost Explorer (`ce:*`), Budgets, Billing | CUR/S3 provisioning (`productx-cur-*`) via `ManageCURPolicy` | `ReadOnlyAccess` + `ManageCURPolicy` + `ExistingCURBucketPolicy` (optional) |
| **Compliance Standards (SecOps)** — CIS Benchmark, FTR | GuardDuty, Inspector, Config, Security Hub, IAM, KMS, CloudTrail, VPC, S3, EC2 | None | `ReadOnlyAccess` + `SecurityAudit` + `ComplianceSignalAccess` |

> **Note on AWS Compute Optimizer:** The `ReadOnlyAccess` managed policy includes `compute-optimizer:Get*` permissions. However, the ProductX Cost Optimization Engine does **not** depend on AWS Compute Optimizer — it collects CloudWatch metrics directly and applies its own proprietary methodology. The Compute Optimizer permissions remain available for any future native-recommendation surfacing but are not required by the engine. See [ProductX Cost Optimization Engine Design — Dependencies](docs/knowledge/design.md).

---

## Write Permission Tracker

Every write permission in this template is documented below. **ProductX has zero infrastructure write permissions** — it cannot create, modify, or delete any customer resource (EC2, RDS, VPC, etc.). Its write access is limited to the AWS Well-Architected Tool for workload management and CUR/S3 bucket provisioning for cost data. All other S3/CUR write permissions belong to the Archera role.

### Summary

| Role | Total Permissions | Read-Only | Write | Write Justification |
|---|---|---|---|---|
| ProductXWAFRRole | 2 managed policies (`ReadOnlyAccess`, `SecurityAudit`) + 19 inline actions + CUR management policy | All read except 11 WAFR workload management actions + CUR/S3 provisioning | 11 Well-Architected Tool actions + 5 CUR/S3 write actions | WAFR workload management + CUR bucket creation for cost visibility |
| CloudvizArchitectureDiagramRole | 1 managed policy (`ReadOnlyAccess`) | All | None | — |
| BillingReadOnlyRole | 38 inline actions | All | None | — |
| ReservedAI-Write | ~65 inline actions | ~62 read | 3 write (`s3:CreateBucket`, `s3:PutBucketPolicy`, `cur:PutReportDefinition`) | CUR provisioning — create S3 bucket for cost data delivery and define the CUR export |

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
| No long-term credentials | Cross-account assumed roles only. No access keys. |
| Blast radius isolation | Each role trusts exactly one principal. Compromise of one platform cannot escalate to another. |
| Org deployment | Service-managed StackSet created as a shell. SA manually adds stack instances to target OUs/accounts. Stack instances must also be manually deleted to remove member-account roles. `AutoDeployment: Enabled` means new accounts joining targeted OUs will receive roles automatically after initial setup. |
| Auditability | All role assumptions logged in CloudTrail. Role names are deterministic for easy log filtering. |

---

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `TrustedAccountId` | No | `820440428496` | ProductX platform AWS account. Do not change unless directed. |
| `ExternalId` | Yes | _(none)_ | Unique per-customer identifier issued by ProductX. Prevents confused deputy. |
| `EnableOrganizations` | No | `no` | Set `yes` for multi-account orgs. Deploys StackSet to member accounts. |
| `ExistingCURBucket` | No | _(empty)_ | Optional. Name of an existing CUR bucket to read (e.g. Archera's `reserved-ai-cur-<uuid>.<10-digit-acct-id>`). Leave empty if ProductX will create its own CUR via `ManageCURPolicy`, or connect later via the ProductX Cost Explorer UI. |

---

## Deployment Modes

| Mode | Stack | Roles Created | Use Case |
|---|---|---|---|
| Single Account | Main stack only | ProductXWAFRRole, Cloudviz, Billing, ReservedAI | Client with one AWS account |
| Organization | Main stack + StackSet (SA manually adds instances) | All above + ProductXWAFRMemberRole + CloudvizMemberRole in targeted member accounts | Client with AWS Organizations |

**Organization mode workflow:**
1. Deploy main stack with `EnableOrganizations=yes` → creates StackSet shell
2. SA manually adds stack instances to the StackSet, targeting specific OUs or accounts
3. After initial setup, new accounts joining targeted OUs receive roles automatically (`AutoDeployment: Enabled`)
4. To remove: SA must manually delete stack instances before deleting the main stack

---

## What This Template Does NOT Do

- ❌ Create, modify, or delete any customer infrastructure (EC2, RDS, VPC, etc.)
- ❌ Create S3 buckets outside the `productx-cur-*` namespace (CUR delivery only)
- ❌ Access customer application data (databases, object storage contents beyond CUR)
- ❌ Modify IAM policies, users, or roles (beyond Archera's self-inspection of its own role)
- ❌ Purchase Reserved Instances or Savings Plans (that requires Archera's full template, deployed separately)
- ❌ Send data outside of AWS (all operations are AWS API calls within the customer's account)

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for detailed version history with stakeholder context and action items.
