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
The template-level `Description` was rewritten to a single generic statement ("Cloudelligent Discovery Role. Creates read-only cross-account IAM roles that let Cloudelligent perform automated cloud assessment, cost visibility, and reporting…"). **All comments were removed** from the template (it is shared with customers), and **parameter/output `Description` fields were removed** — explanatory content lives in the docs, not the customer-facing CF. (`ConstraintDescription` validation messages were retained as they are functional.)

### Point 3 — Role naming / third-party de-emphasis
Two levels of naming were addressed:

**Deployed `RoleName` values — only the Cloudviz role changed:**
- `cloudelligent-assessment-role` → **`CloudelligentDiscoveryRole`** (both the management-account role and the StackSet member role, which share the name).
- Its inline policy `CloudvizReadOnlyAccess` → **`CloudelligentDiscoveryReadOnlyAccess`**.
- **All other deployed `RoleName` values are unchanged** — `ProductXWAFRRole`, `ProductXWAFRMemberRole`, `BillingReadOnlyRole`, `ReservedAI-Write` — because third-party/internal backends may hardcode them (see Accepted Breaking Change).

**Logical IDs (CloudFormation-internal identifiers) — fully de-branded** (existing stacks are out of scope, so this is safe/cosmetic):
- `ProductXWAFRRole` → `CloudelligentWAFRRole`
- `ProductXWAFRMemberRole` → `CloudelligentWAFRMemberRole`
- `CloudvizArchitectureDiagramRole` → `CloudelligentDiscoveryRole`
- `CloudvizMemberRole` → `CloudelligentDiscoveryMemberRole`
- `BillingReadOnlyRole` → `CloudelligentBillingRole`
- `ReservedAIRole` → `CloudelligentFinOpsRole`
- Outputs: `CloudvizRoleARN` → `DiscoveryRoleArn`, `CloudvizMemberRoleArn` → `DiscoveryMemberRoleArn`

> **Logical ID ≠ deployed RoleName.** Logical IDs are internal to the template and were de-branded freely. Deployed `RoleName` values are what appears in the customer's IAM and what third parties assume — only Cloudviz's changed.

### Point 4 — Generic, de-branded descriptions (WAFR term retained)
De-branded all customer-visible text of tool/vendor names (Cloudviz, Archera, ReservedAI, "architecture diagrams," "spot data feeds," ProductX prose). **The term "WAFR" is retained** — management confirmed WAFR is an acceptable, accurate descriptor (preferred over vague "assessment" wording). This supersedes the original point-4 wording that would have stripped WAFR; the intent is to de-emphasize *third-party* tooling, not the AWS Well-Architected Framework itself.

### Point 5 — Read-only guardrail (forward-looking, Option A)
Adopt as a **governance principle**, not a removal:
- The existing write permissions (`WAFRWorkloadManagement`, `ManageCURPolicy`, Archera's scoped writes) are **retained** — they are core/agreed functionality.
- **No new write or elevated permissions** may be added to the template without explicit management + security review and a corresponding ADR.
- The accepted data-plane-read `Deny` (ADR-005) stands and is the model for "least privilege by default."
- This principle is recorded in the **README** (not as a template comment — comments were removed from the customer-facing template per point 2).

## Accepted Breaking Change — Cloudviz Role Rename

Renaming the Cloudviz role **will break diagram generation for existing customers** until Cloudviz's stored Role ARN is updated.

**Why it breaks:** Cloudviz's platform stores the full **Role ARN** (`arn:aws:iam::<customer-acct>:role/cloudelligent-assessment-role`) and calls `sts:AssumeRole` against it. The role **name is part of the ARN**. Renaming the role changes the ARN; Cloudviz keeps calling the old ARN, which no longer resolves → `AssumeRole` fails. Leaving the **trust policy** (Cloudviz account `282762468439` + static `ExternalId`) unchanged is correct but insufficient — it authorizes the right principal, but there is no role at the old name for it to apply to.

**Management decision:** this breakage for existing customers is **accepted**. Existing customers must re-onboard / update the stored Role ARN on the Cloudviz side (or their diagrams break until they do). New customers receive `CloudelligentDiscoveryRole` from the start and are unaffected.

**Not changing:** trust principal account, static `ExternalId`, and the role's permission *set* (the 52 read actions) all stay identical — only the `RoleName` and the inline policy's *name* (`CloudvizReadOnlyAccess` → `CloudelligentDiscoveryReadOnlyAccess`) change.

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

## Implementation (applied 2026-09-17)

1. **CFT — descriptions/naming (points 2 + 4):** ✅ Done
   - Rewrote template `Description` to a generic de-branded statement.
   - Removed **all comments** from the template and removed parameter/output `Description` fields (kept `ConstraintDescription`).
   - De-branded logical IDs and Output names to the `Cloudelligent*` scheme.
2. **CFT — Cloudviz rename (point 3):** ✅ Done
   - Changed `RoleName` `cloudelligent-assessment-role` → `CloudelligentDiscoveryRole` on both the management and member Cloudviz roles; renamed inline policy `CloudvizReadOnlyAccess` → `CloudelligentDiscoveryReadOnlyAccess`.
   - Trust policy, `ExternalId`, and the 52-action permission set unchanged.
3. **CFT — read-only guardrail (point 5):** ✅ Done — recorded in the README (not the template, since comments were removed).
4. **Docs:** ✅ Done — README (role table, source-of-truth, write tracker, deployment modes), CUSTOMER_SECURITY_REFERENCE (role name, action count, de-branded prose), CHANGELOG (`[2.4]`), versions bumped to 2.4.
5. **Rollout / escalation:** ⏳ Pending — coordinate with Cloudviz + account management the per-customer stored-ARN update for existing customers (breakage accepted). Owner TBD (open question 3).
6. **Validation:** ✅ Done — template parses; both Cloudviz roles carry the new name; zero `cloudelligent-assessment-role` / `Cloudviz` references remain in the template; external-safety re-checked.

## Open Questions for Review

1. **Resolved — full de-branding.** All docs, including the internal README, are de-branded of tool/vendor tool names. Bare-minimum third-party references (e.g. trust principal account IDs) are retained where needed for accuracy.
2. **Base name confirmed** as `CloudelligentDiscoveryRole`. The management and member Cloudviz roles keep the shared name (as today) — intentional.
3. **Cutover sequencing:** who owns notifying existing Cloudviz-using customers, and is there a maintenance window, or is best-effort acceptable given the breakage is accepted? (Still open — rollout/account-management item.)
4. **Resolved — comment/note only.** The read-only guardrail is enforced via a template header comment + README note referencing this ADR. No CI check will be added at this time.

## Alternatives Considered

- **Rename all four roles to a single convention:** rejected — third-party (Archera) and internal (finance) backends may hardcode names; management scoped the rename to Cloudviz only.
- **Do not rename Cloudviz (de-brand description only):** would avoid the breakage, but management explicitly wants the Cloudviz role renamed and accepts the existing-customer impact.
- **Strip all write permissions to make the template literally read-only:** rejected per Option A — existing WAFR and CUR writes are core/agreed functionality; the read-only directive is forward-looking (no new writes), not retroactive.
