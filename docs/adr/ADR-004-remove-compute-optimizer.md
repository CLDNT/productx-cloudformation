# ADR-004: Remove ComputeOptimizerPolicy — Compute Optimizer Is Not Used

Status: Accepted  
Date: 2026-09-01  
Accepted: 2026-09-17  
Author: Haider Zaidi  
Reviewers: CEO, COO, Product Manager, Solutions Architects

## Context

The `Cloudelligent_WAFR_Onboarding.yaml` template currently includes a `ComputeOptimizerPolicy` — a dedicated IAM policy granting 17 read-only `compute-optimizer:Get*`/`Describe*` actions — attached to both `ProductXWAFRRole` (management account) and `ProductXWAFRMemberRole` (member accounts via StackSet).

This policy was added on the assumption that AWS Compute Optimizer would serve as a **fallback/supplement** to the ProductX Cost Optimization Engine — surfacing native AWS recommendations alongside the engine's own findings.

The authoritative ProductX Cost Optimization Engine design document (`docs/knowledge/design.md`) has since been clarified and now states unambiguously:

- **Locked decision #2:** "Proprietary Product X engine — Product X always runs its own methodology to generate findings and recommendations from metrics it collects itself. There is **no external recommendation source and no fallback routing**."
- **Data Sources & IAM Mapping (Part VI):** "There are **no** `compute-optimizer:*` permissions — Product X does not call AWS Compute Optimizer."
- **Overview:** "AWS Compute Optimizer is **not a component, dependency, or data source** of this system."

The engine collects CloudWatch metrics directly and applies its own proprietary methodology. It does not call Compute Optimizer, read its recommendations, or check enrollment — and there is no fallback path that would consume Compute Optimizer data.

**Conclusion:** The `compute-optimizer:*` permissions are not used by the engine, and there is no platform-level fallback consuming them either. They are dead weight in the template.

## Decision

**Remove the `ComputeOptimizerPolicy` resource entirely** from `Cloudelligent_WAFR_Onboarding.yaml`, including:
1. The `ComputeOptimizerPolicy` IAM policy on `ProductXWAFRRole` (management account)
2. The `ComputeOptimizerAccess` inline policy on `ProductXWAFRMemberRole` (StackSet member template)

No compensating permissions are required — nothing consumes Compute Optimizer data.

## Consequences

### Positive

1. **Tighter least privilege** — removes 17 permissions that are never exercised. Smaller permission surface for customer security review.
2. **Documentation consistency** — the CFT, README, and engine design doc will all agree that Compute Optimizer is not used. Eliminates a contradiction that a sharp customer security reviewer or AWS partner reviewer would likely catch.
3. **Accurate sales positioning** — reinforces the "proprietary engine, no reliance on native AWS tooling" differentiator without the caveat of "but we also request Compute Optimizer permissions."
4. **Reduced resource count** — template drops from 9 resources to 8.

### Negative

1. **No native cross-check** — if ProductX ever wants to surface AWS Compute Optimizer's native recommendations as a second opinion in the UI, the permissions would need to be re-added. Per the current design (locked decision #2), this is explicitly not planned.
2. **Existing customers need a stack update** — any client already deployed with `ComputeOptimizerPolicy` will have it removed on the next stack update (no functional impact since nothing uses it).

### Neutral

- No functional change to any ProductX feature — the engine never used these permissions.
- `compute-optimizer:*` read actions are also present in the AWS `ReadOnlyAccess` managed policy (still attached to `ProductXWAFRRole`). Removing the dedicated `ComputeOptimizerPolicy` does **not** revoke Compute Optimizer read access that comes via `ReadOnlyAccess` — but since nothing calls those APIs, this is immaterial. If a future decision replaces `ReadOnlyAccess` with a scoped policy, Compute Optimizer access disappears entirely, which is consistent with this decision.

## Alternatives Considered

### Alternative A: Keep `ComputeOptimizerPolicy` as a future-proofing measure
- Retain the policy in case ProductX later wants native recommendations
- **Rejected because:** Violates least privilege by requesting permissions with no current consumer. Contradicts the engine design doc's locked decision. Customer security teams reasonably object to unused permissions. If needed later, it can be re-added via a documented change.

### Alternative B: Keep the policy but document it as "platform-level, not engine"
- Frame Compute Optimizer as consumed by the ProductX platform UI rather than the engine
- **Rejected because:** This is not true per Interpretation B — Compute Optimizer is not consumed anywhere, by the engine or the platform. Documenting a non-existent consumer would be inaccurate.

## Implementation

1. ✅ **Done** — Removed the `ComputeOptimizerPolicy` resource from `Cloudelligent_WAFR_Onboarding.yaml` (management account).
2. ✅ **Done** — Removed the `ComputeOptimizerAccess` inline policy from `ProductXWAFRMemberRole` in the StackSet `TemplateBody`.
3. ✅ **Done** — Updated `README.md`: removed the `ComputeOptimizerPolicy` component row (now 8 resources), the Source-of-Truth entry, the "Optimize Usage — Compute Optimizer fallback" mapping row, the "Note on AWS Compute Optimizer", and the Write Permission Tracker reference.
4. ✅ **Done** — Updated `CHANGELOG.md`: superseded the "Added: ComputeOptimizerPolicy" entry with a removal entry referencing this ADR.
5. ✅ **Done** — Updated `CUSTOMER_SECURITY_REFERENCE.md`: removed the `ComputeOptimizerPolicy` line from conditional resources.
6. ✅ **Done** — Validated the template: 8 resources, zero `compute-optimizer` references.

## Verification

- Template validates as well-formed CloudFormation (8 top-level resources).
- `grep` confirms zero `compute-optimizer` references remain in the CFT.
- All docs cross-checked so no reference to Compute Optimizer as a dependency/fallback survives.
- Engine design doc (`docs/knowledge/design.md`) already reflects the correct state — no change needed there.

## Questions for Review

1. **Resolved on acceptance** — No roadmap item in the next 6–12 months requires Compute Optimizer recommendations; the proprietary engine remains the sole recommendation source (design.md locked decision #2).
2. **Resolved on acceptance** — Confirmed the ProductX platform UI does not surface Compute Optimizer data today. The removal premise holds.
