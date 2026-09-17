# ADR-006: Naming, Descriptions, De-Branding, and a Read-Only Guardrail (Management Feedback)

Status: Accepted  
Date: 2026-09-17  
Accepted: 2026-09-17  
Author: Haider Zaidi  
Reviewers: CEO, COO, Product Manager, Solutions Architects

## Context

Management reviewed `Cloudelligent_WAFR_Onboarding.yaml` (v2.3) — which is shared with external customers for review — and provided five directives:

1. Change to accept a Root OU ID (in addition to a child OU).
2. Proper naming and descriptions throughout the template.
3. A single consistent role-naming approach; do not give prominence to third-party tooling in role names.
4. Hardcoded, generic descriptions referencing Cloudelligent / Product X only — no mention of specific tools (WAFR, CloudOps, Cloudviz, Archera, etc.).
5. The template stays strictly read-only from a **forward-looking** standpoint — no *new* write/elevated permissions are to be requested from the customer.

This ADR plans those changes. Two of the five are already satisfied or clarified; three require template edits; one carries an **accepted breaking change** for existing customers.

### Current state (v2.3)

- **Template `Description`** names "ProductX WAFR cross-account role," "Cloudelligent assessment role (for Cloudviz)," "Billing read-only role," and "ReservedAI role (for Archera)."
- **Role names:** `ProductXWAFRRole` / `ProductXWAFRMemberRole`, `cloudelligent-assessment-role` (Cloudviz, both mgmt + StackSet member), `BillingReadOnlyRole`, `ReservedAI-Write` (Archera).
- **Descriptions/comments** reference tool-specific terms (WAFR, diagrams, spot data feeds, etc.).
- **Existing write permissions** (in scope to KEEP per point 5, Option A): `WAFRWorkloadManagement` (11 `wellarchitected:*` writes) and `ManageCURPolicy` (`s3:CreateBucket`, `s3:PutBucketPolicy`, `cur:Put/Modify/DeleteReportDefinition`). Archera's `ReservedAI-Write` also holds scoped writes. These predate this ADR and are retained.

## Decisions

### Point 1 — Root OU ID: ALREADY DONE (ADR-003)
The `OrgUnitId` parameter added in ADR-003 already accepts **either** a root ID (`r-xxxx`) **or** a child OU ID (`ou-xxxx-xxxxxxxx`). No further change required. Marked complete.

### Point 2 — Proper naming and descriptions
Rewrite the template-level `Description`, all parameter descriptions, resource comments, and `Outputs` descriptions to be clear, consistent, and professional. Remove the "combines X + Y + Z for vendor" framing in favor of a single generic statement of purpose.

### Point 3 — Role naming / third-party de-emphasis
**Scope decision (confirmed with management):** the only **role-name change** is the Cloudviz role.
- `cloudelligent-assessment-role` → **`CloudelligentDiscoveryRole`** (applies to BOTH the management-account role `CloudvizArchitectureDiagramRole` and the StackSet member role `CloudvizMemberRole`, which currently share that `RoleName`).
- **All other role names are unchanged** — `ProductXWAFRRole`, `ProductXWAFRMemberRole`, `BillingReadOnlyRole`, and `ReservedAI-Write` keep their current `RoleName` values (renaming them carries integration risk and was explicitly scoped out).

> **Rationale for not renaming the others:** third-party and internal backends may hardcode the role name/ARN. Management limited the rename to the Cloudviz role only.

### Point 4 — Generic, de-branded descriptions
De-brand **all** descriptions and comments (this is safe — descriptions have no functional effect). Remove tool/vendor names (WAFR, CloudOps, Cloudviz, Archera, ReservedAI, "architecture diagrams," "spot data feeds," etc.) from customer-visible text. Replace with generic Cloudelligent / Product X read-only "discovery / assessment / cost-visibility" language. Where a third party must be referenced for the customer to understand a trust relationship, keep it to the **bare minimum** (e.g. the trust principal account ID stands on its own; no need to name the vendor or its product in prose).

### Point 5 — Read-only guardrail (forward-looking, Option A)
Adopt as a **governance principle**, not a removal:
- The existing write permissions (`WAFRWorkloadManagement`, `ManageCURPolicy`, Archera's scoped writes) are **retained** — they are core/agreed functionality.
- **No new write or elevated permissions** may be added to the template without explicit management + security review and a corresponding ADR.
- The accepted data-plane-read `Deny` (ADR-005) stands and is the model for "least privilege by default."
- This principle will be recorded as a comment/section in the template header and in the README so future editors don't silently add writes.

## Accepted Breaking Change — Cloudviz Role Rename

Renaming the Cloudviz role **will break diagram generation for existing customers** until Cloudviz's stored Role ARN is updated.

**Why it breaks:** Cloudviz's platform stores the full **Role ARN** (`arn:aws:iam::<customer-acct>:role/cloudelligent-assessment-role`) and calls `sts:AssumeRole` against it. The role **name is part of the ARN**. Renaming the role changes the ARN; Cloudviz keeps calling the old ARN, which no longer resolves → `AssumeRole` fails. Leaving the **trust policy** (Cloudviz account `282762468439` + static `ExternalId`) unchanged is correct but insufficient — it authorizes the right principal, but there is no role at the old name for it to apply to.

**Management decision:** this breakage for existing customers is **accepted**. Existing customers must re-onboard / update the stored Role ARN on the Cloudviz side (or their diagrams break until they do). New customers receive `CloudelligentDiscoveryRole` from the start and are unaffected.

**Not changing:** trust principal account, static `ExternalId`, and the role's permissions (`CloudvizReadOnlyAccess`, 52 actions) all stay identical — only `RoleName` changes.

## Consequences

### Positive
- Customer-facing template no longer advertises third-party tooling; reads as a single Cloudelligent Product X read-only onboarding artifact.
- Clearer, more professional naming/descriptions for external security review.
- Read-only guardrail prevents scope creep in future edits.
- Root-OR-child OU targeting already supported.

### Negative / Risk
- **Existing-customer breakage** for Cloudviz diagram generation until each customer's stored Role ARN is updated (accepted by management).
- Coordination required: template rename + Cloudviz-side ARN update should be sequenced for each existing customer to minimize downtime.
- Role name and its references must change **together** in the template: the `RoleName` on both `CloudvizArchitectureDiagramRole` and `CloudvizMemberRole`. (The Cloudviz role is not referenced by `!Ref`/`!GetAtt` elsewhere except the `CloudvizRoleARN` output, which resolves dynamically and needs no name edit.)

### Neutral
- De-branding descriptions has zero functional impact.
- Keeping the other role names avoids touching ProductX/finance/Archera integrations.

## Implementation Plan (pending approval — NOT yet applied)

1. **CFT — descriptions/naming (points 2 + 4):**
   - Rewrite template `Description` to a generic Cloudelligent Product X read-only onboarding statement.
   - De-brand parameter descriptions (`ExistingCURBucket`, `EnableOrganizations`, `OrgUnitId`, etc.) and all resource comments.
   - De-brand `Outputs` descriptions.
2. **CFT — Cloudviz rename (point 3, gated):**
   - Change `RoleName` from `cloudelligent-assessment-role` to `CloudelligentDiscoveryRole` on both the management `CloudvizArchitectureDiagramRole` and the StackSet member `CloudvizMemberRole`.
   - Leave trust policy, `ExternalId`, and permissions untouched.
3. **CFT — read-only guardrail (point 5):**
   - Add a header comment / section stating no new write or elevated permissions may be introduced without review, referencing this ADR.
4. **Docs:**
   - README: update role table names/descriptions and fully de-brand tool/vendor tool names throughout (per resolved Q1), retaining bare-minimum third-party references (account IDs) only where needed for accuracy.
   - CUSTOMER_SECURITY_REFERENCE: update the Cloudviz role name and de-brand descriptions; this is the external-facing doc and must match the template's generic tone.
   - CHANGELOG: add a `[2.4]` entry noting the de-branding, the Cloudviz rename + accepted existing-customer breakage, and the read-only guardrail.
5. **Rollout / escalation:**
   - Escalate to Cloudviz (and account management for affected customers) the coordinated cutover: rename in template + update stored Role ARN on Cloudviz per existing customer.
   - Sequence existing-customer updates; communicate expected diagram-generation downtime.
6. **Validation:**
   - Template parses; both Cloudviz roles carry the new name; no dangling `cloudelligent-assessment-role` references remain.
   - External-safety re-check: customer-facing template + CUSTOMER_SECURITY_REFERENCE contain no tool/vendor branding beyond the bare-minimum trust references.

## Open Questions for Review

1. **Resolved — full de-branding.** All docs, including the internal README, are de-branded of tool/vendor tool names. Bare-minimum third-party references (e.g. trust principal account IDs) are retained where needed for accuracy.
2. **Base name confirmed** as `CloudelligentDiscoveryRole`. The management and member Cloudviz roles keep the shared name (as today) — intentional.
3. **Cutover sequencing:** who owns notifying existing Cloudviz-using customers, and is there a maintenance window, or is best-effort acceptable given the breakage is accepted? (Still open — rollout/account-management item.)
4. **Resolved — comment/note only.** The read-only guardrail is enforced via a template header comment + README note referencing this ADR. No CI check will be added at this time.

## Alternatives Considered

- **Rename all four roles to a single convention:** rejected — third-party (Archera) and internal (finance) backends may hardcode names; management scoped the rename to Cloudviz only.
- **Do not rename Cloudviz (de-brand description only):** would avoid the breakage, but management explicitly wants the Cloudviz role renamed and accepts the existing-customer impact.
- **Strip all write permissions to make the template literally read-only:** rejected per Option A — existing WAFR and CUR writes are core/agreed functionality; the read-only directive is forward-looking (no new writes), not retroactive.
