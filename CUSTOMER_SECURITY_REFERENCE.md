# Cloudelligent Onboarding Stack — Customer Security Reference

**Template:** `Cloudelligent_WAFR_Onboarding.yaml`  
**Version:** 2.2  
**Date:** September 2026

---

## Overview

This CloudFormation template deploys IAM roles into your AWS Management Account that allow Cloudelligent to deliver the following services:

1. **AWS Well-Architected Framework Reviews** — automated scanning, workload creation, and assessment reporting
2. **Cost Optimization** — rightsizing recommendations, idle/orphaned resource identification, scheduling recommendations
3. **Cost Analytics** — cost breakdown by tags, daily spend reports, RI/SP utilization tracking
4. **Compliance Scanning** — CIS Benchmark assessments, security posture evaluation
5. **Architecture Diagrams** — automated infrastructure visualization

For AWS Organizations, a StackSet extends coverage to member accounts.

---

## What Gets Deployed

| # | IAM Role | Purpose | Access Level |
|---|---|---|---|
| 1 | `ProductXWAFRRole` | WAFR assessments, cost optimization, compliance scanning, cost visibility | Read-only (config/metadata + cost data; application-data reads explicitly denied) + Well-Architected Tool workload management + CUR provisioning |
| 2 | `cloudelligent-assessment-role` | Architecture diagram generation | Read-only |
| 3 | `BillingReadOnlyRole` | Billing analytics and invoice visibility | Read-only |
| 4 | `ReservedAI-Write` | Cost data export provisioning (for RI/SP analysis tooling) | Read + limited write (scoped) |

Additional conditional resources:
- **AllowAssumeMemberRoles** — policy enabling the ProductX role to assume matching roles in member accounts (org mode only)
- **ManageCURPolicy** — policy enabling ProductX to create a Cost & Usage Report and delivery bucket
- **ExistingCURBucketPolicy** — policy enabling ProductX to read an existing CUR bucket (if provided)
- **StackSet** — deploys `ProductXWAFRRole` and `cloudelligent-assessment-role` to member accounts (org mode only)

---

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `TrustedAccountId` | No | Pre-filled | Cloudelligent's platform account. Do not change. |
| `ExternalId` | Yes | _(provided by Cloudelligent)_ | Unique per-customer identifier for confused-deputy protection. |
| `EnableOrganizations` | No | `no` | Set `yes` if deploying across an AWS Organization. |
| `ExistingCURBucket` | No | _(empty)_ | Optional. Name of an existing CUR S3 bucket to read. Leave empty to let ProductX create one automatically, or connect later. |

---

## Write Permissions — Complete List

This template has **zero infrastructure write permissions**. It cannot create, modify, or delete any customer resource (EC2, RDS, VPC, Lambda, etc.). All write permissions are limited to two narrowly-scoped services:

### 1. AWS Well-Architected Tool (ProductX)

| Action | Resource Scope | Purpose |
|---|---|---|
| `wellarchitected:CreateWorkload` | All workloads | Create assessment workloads |
| `wellarchitected:UpdateWorkload` | All workloads | Update workload metadata |
| `wellarchitected:DeleteWorkload` | All workloads | Remove completed assessments |
| `wellarchitected:AssociateLenses` | Your account's workloads only | Attach WAFR lens |
| `wellarchitected:DisassociateLenses` | Your account's workloads only | Remove lens |
| `wellarchitected:UpgradeLensReview` | Your account's workloads only | Upgrade to latest lens version |
| `wellarchitected:UpdateLensReview` | Your account's workloads only | Save review progress |
| `wellarchitected:UpdateAnswer` | Your account's workloads only | Record assessment answers |
| `wellarchitected:CreateMilestone` | Your account's workloads only | Snapshot workload state |
| `wellarchitected:TagResource` | Your account's workloads only | Tag workloads |
| `wellarchitected:UntagResource` | Your account's workloads only | Remove tags |

These actions **only** affect the AWS Well-Architected Tool. They cannot touch any other AWS service or resource.

### 2. Cost & Usage Report Provisioning (ProductX)

| Action | Resource Scope | Purpose |
|---|---|---|
| `cur:PutReportDefinition` | `productx-*` prefixed reports only | Create cost data export |
| `cur:ModifyReportDefinition` | `productx-*` prefixed reports only | Update export configuration |
| `cur:DeleteReportDefinition` | `productx-*` prefixed reports only | Remove export |
| `s3:CreateBucket` | `productx-cur-*` prefixed buckets only | Create delivery bucket |
| `s3:PutBucketPolicy` | `productx-cur-*` prefixed buckets only | Allow AWS billing to deliver data |

ProductX **cannot** create or modify any S3 bucket or CUR report outside the `productx-*` / `productx-cur-*` namespace.

### 3. Cost Data Export Provisioning (Archera)

| Action | Resource Scope | Purpose |
|---|---|---|
| `s3:CreateBucket` | `reserved-ai-cur-*` and `reserved-ai-spot-*` only | Create cost data delivery bucket |
| `s3:PutBucketPolicy` | `reserved-ai-cur-*` and `reserved-ai-spot-*` only | Allow AWS billing to deliver data |
| `cur:PutReportDefinition` | `reserved-ai-*` prefixed reports only | Create cost data export |

Archera **cannot** create or modify any S3 bucket outside the `reserved-ai-*` namespace.

### 4. IAM Self-Inspection (Archera only)

The Archera role can read its **own** policy configuration only (`iam:GetRolePolicy`, `iam:ListRolePolicies`, etc. scoped exclusively to `arn:aws:iam::*:role/ReservedAI-Write`). It cannot read or modify any other IAM resource.

---

## Read Permissions

**ProductX assessment role** uses two AWS managed policies for read access, **narrowed by an explicit Deny**:

- **[ReadOnlyAccess](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/ReadOnlyAccess.html)** — AWS-maintained policy providing read access across services (Describe, Get, List actions only)
- **[SecurityAudit](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/SecurityAudit.html)** — AWS-maintained policy for security assessment (audit-level access)

These are standard AWS managed policies used by hundreds of third-party tools for read-only cloud assessment.

**Explicit data-access Deny.** Because `ReadOnlyAccess` is broad, the ProductX role also carries an explicit IAM `Deny` policy (`DataPlaneReadDeny`) that **blocks all reads of your application data**, overriding the managed policy:

- **S3 object contents** — denied everywhere except the Cost & Usage Report (cost data) buckets ProductX is explicitly configured to read
- **Database records** — DynamoDB and DAX item reads (`GetItem`, `Query`, `Scan`, etc.), SimpleDB, Keyspaces, Timestream, QLDB
- **Messaging and streaming data** — SQS message bodies, Kinesis stream records
- **Search data** — OpenSearch document reads

ProductX reads only **resource configuration and metadata** (how resources are set up), **performance metrics** (CloudWatch), and your **cost data** (Cost Explorer + CUR). It **cannot** read the contents of your objects, databases, queues, or streams. Because this is an explicit IAM `Deny`, it is a hard technical guarantee your security team can verify directly in the template — not a matter of trust or convention. This is particularly relevant for regulated data (e.g. PHI under HIPAA): ProductX has no ability to read it.

**Architecture diagram role** (`cloudelligent-assessment-role`) uses a **custom, tightly-scoped read-only policy** — approximately 50 specific `Describe`/`List`/`Get` actions limited to the resource types needed for diagram generation (EC2, RDS, Lambda, ECS, EKS, ELB, S3 metadata, etc.). It does **not** use the broad `ReadOnlyAccess` policy and cannot read application data.

---

## Security Controls

| Control | How It's Implemented |
|---|---|
| **No application-data access** | An explicit IAM `Deny` blocks reads of S3 object contents (outside cost-report buckets), database records, queue messages, and stream data. Overrides the AWS read policy — verifiable in the template. |
| **Confused-deputy protection** | Per-customer `ExternalId` required for role assumption. Prevents unauthorized cross-account access. |
| **No long-term credentials** | All access via cross-account IAM role assumption. No access keys, no stored secrets. |
| **Prefix-scoped writes** | S3 and CUR writes are locked to specific naming prefixes. Cannot affect any other bucket or report. |
| **Blast radius isolation** | Each role trusts exactly one principal. Compromise of one cannot escalate to another. |
| **CloudTrail auditability** | Every role assumption is logged in your CloudTrail. Role names are deterministic for easy filtering. |
| **Cleanup on removal** | Deleting the CloudFormation stack removes all deployed roles immediately. |

---

## What This Template Does NOT Do

- ❌ Create, modify, or delete any customer infrastructure (EC2, RDS, VPC, Lambda, etc.)
- ❌ Create S3 buckets outside the `productx-cur-*` or `reserved-ai-*` namespaces
- ❌ Access customer application data (database records, object contents, queue/stream data, secrets) — **blocked by an explicit IAM `Deny`, not just convention**
- ❌ Modify any IAM policies, users, or roles in your account
- ❌ Purchase Reserved Instances or Savings Plans
- ❌ Send data outside of AWS (all operations are AWS API calls within your account)
- ❌ Make any changes that could affect running workloads

---

## Deployment Modes

| Mode | What Happens | Use Case |
|---|---|---|
| **Single Account** | Deploys roles in your management account only | One AWS account |
| **Organization** | Deploys roles in management account + creates a StackSet for member accounts | Multiple AWS accounts under AWS Organizations |

For Organization mode, CloudFormation StackSets trusted access must be enabled in your AWS Organizations settings (one-time setup).

---

## Removal

To fully remove Cloudelligent access:

1. **Single account:** Delete the CloudFormation stack → all roles are removed immediately
2. **Organization:** Delete all StackSet instances first, then delete the main stack

After deletion, Cloudelligent has zero access to your AWS environment.

---

## Questions?

If your security team requires additional detail about any permission, needs a walkthrough of the template, or wants to discuss scope reduction, contact your Cloudelligent Solutions Architect.
