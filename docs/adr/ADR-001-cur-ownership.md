# ADR-001: CUR Ownership — Archera Owns, ProductX Reads

Status: Proposed  
Date: 2026-08-27  
Author: Haider Zaidi  
Reviewers: CEO, COO, Product Manager, Sales Lead, Solutions Architects

## Context

Cloudelligent is discontinuing CoreStack and consolidating client onboarding onto its own ProductX platform. The existing onboarding stack (`Corestack_FinOps_read_access.yml`) deployed into customer AWS accounts creates:

1. `cloudelligent-assessment-role` — a broad read-only role trusted by CoreStack's AWS accounts
2. `BillingReadOnlyRole` — billing access for an internal viewer
3. `ReservedAI-Write` — Archera's role for RI/SP purchasing and CUR creation

The new ProductX stack (`ProductX_WAFR_Workload_ComputeOptimizer_Role.yaml`) currently includes a `ManageCURPolicy` that grants ProductX the ability to:
- Create CUR report definitions (`cur:PutReportDefinition`)
- Create and configure S3 buckets (`s3:CreateBucket`, `s3:PutBucketPolicy` on `productx-cur-*`)
- Read CUR data from the bucket it provisions

Simultaneously, Archera (which remains as a side tool for RI/SP purchasing) already has its own `ReservedAI-Write` role with CUR creation permissions (`cur:PutReportDefinition` on `reserved-ai-*`) and S3 bucket creation (`s3:CreateBucket` on `reserved-ai-cur-*`).

The question: **Who should own CUR creation in the customer's account — ProductX or Archera?**

Both tools need to read CUR data. Only one should create it.

## Decision

**Archera owns CUR creation. ProductX reads from the Archera-provisioned CUR bucket.**

The ProductX CloudFormation template will:
- Remove the `ManageCURPolicy` resource entirely
- Rely on the `ExistingCURBucket` parameter to grant read access to the Archera-created bucket (naming format: `reserved-ai-cur-<uuid>.<first-10-digits-of-mgmt-account-id>`)
- Retain no S3 write or CUR write permissions

The Archera onboarding step (already part of the process) creates the CUR. ProductX connects to it via Cost Explorer → Select Bucket.

## Consequences

### Positive

1. **Least privilege for ProductX** — The ProductX role becomes purely read-only in the customer's AWS account. Zero S3 write, zero CUR write. This is easier to justify during customer security reviews.
2. **Single CUR, two readers** — No duplicate CUR exports consuming S3 storage or generating redundant billing data.
3. **Clear separation of concerns** — ProductX = WAFR assessments + cost visibility (read). Archera = RI/SP purchasing + CUR provisioning (write). Each tool has exactly the permissions its function requires.
4. **Simpler CFn template** — Fewer IAM policies to explain to customers during onboarding calls.
5. **Reduced blast radius** — If the ProductX role is ever compromised, the attacker cannot create buckets or modify billing exports in the customer account.

### Negative

1. **Archera dependency for cost data** — ProductX Cost Explorer will not show data until Archera onboarding is complete and the CUR has been provisioned (typically 24 hours after Archera generates it).
2. **Sequencing constraint** — Archera must be onboarded before (or concurrently with) the ProductX CUR connection step. The SA Runbook must enforce this ordering.
3. **Bucket naming dependency** — ProductX must know the Archera bucket name. Archera uses the format `reserved-ai-cur-<uuid>.<first-10-digits-of-management-account-id>` (e.g. `reserved-ai-cur-804eb7fc-5603-4eca-b93b-dc64e329c473.8779124326`). The UUID is not deterministic — the SA must look up the actual bucket name from Archera or the customer's S3 console during onboarding.

### Neutral

- The `ExistingCURBucket` parameter already exists in the ProductX template for bring-your-own-CUR scenarios. This decision simply makes it the default path rather than an optional one.
- Archera's `ReservedAI-Write` role and permissions remain unchanged from the old CoreStack template.

## Alternatives Considered

### Alternative A: ProductX owns CUR, Archera reads it

- ProductX creates `productx-cur-<account-id>` bucket and CUR export
- Archera pointed to read from ProductX's bucket
- **Rejected because:** Grants ProductX unnecessary S3 write + CUR write permissions. Violates the goal of making ProductX purely read-only. Would also require changes to Archera's onboarding flow (currently expects to create its own CUR).

### Alternative B: Both tools create their own CUR

- ProductX creates `productx-cur-*`, Archera creates `reserved-ai-cur-*`
- **Rejected because:** Duplicate CUR exports waste storage, confuse billing teams during audits ("why are there two identical reports?"), and require broader write permissions on the ProductX role.

### Alternative C: Client creates CUR manually, both tools read it

- Customer creates a CUR during the on-call session (old process)
- Both ProductX and Archera use `ExistingCURBucket` to read it
- **Rejected because:** Adds friction to the onboarding call (6+ manual steps), requires a backfill support ticket, and was a major pain point of the old CoreStack process that ProductX was designed to eliminate.

## Implementation

1. Remove `ManageCURPolicy` from `Cloudelligent_WAFR_Onboarding.yaml`
2. Make `ExistingCURBucket` parameter guidance reference Archera's bucket naming convention
3. Update SA Runbook to enforce sequencing: Archera onboarding → CUR generation → ProductX CUR connection
4. Update customer-facing guide to remove any CUR manual creation steps
