# ADR-002: CloudFormation Template Optimization — Permission Reduction & Resource Consolidation

Status: Proposed  
Date: 2026-08-27  
Author: Haider Zaidi  
Reviewers: CEO, COO, Product Manager, Sales Lead, Solutions Architects

## Context

The `Cloudelligent_WAFR_Onboarding.yaml` template currently deploys **8 resources** (4 IAM roles + 3 policies + 1 StackSet) into each customer's AWS account. Following the addition of `WAFRWorkloadManagement`, a review was conducted to identify further optimization opportunities.

Goals:
- Minimize the number of IAM roles deployed in customer accounts (each role is a security review item)
- Remove redundant permissions that are already covered by managed policies
- Reduce the write-permission surface presented to customer security teams
- Simplify the template for faster customer approval cycles

Current state:

| Resource | Role Name | Trust Principal | Purpose |
|---|---|---|---|
| ProductXWAFRRole | `ProductXWAFRRole` | Cloudelligent ProductX (`820440428496`) | WAFR + FinOps + Cost + Compliance |
| CloudvizArchitectureDiagramRole | `cloudelligent-assessment-role` | Cloudviz (`282762468439`) | Architecture diagrams |
| BillingReadOnlyRole | `BillingReadOnlyRole` | Internal finance (`898668804275`) | Invoice/billing analytics |
| ReservedAIRole | `ReservedAI-Write` | Archera (`967800896805`) | Cost visibility + CUR creation (RI/SP purchasing is in Archera's separate full template) |

Plus conditional policies (`AllowAssumeMemberRoles`, `ExistingCURBucketPolicy`) and the StackSet.

---

## Proposals

### Proposal 1: Remove `ComplianceSignalAccess` Inline Policy (redundant)

**Recommendation: REMOVE**

Every action in this policy is already granted by the `ReadOnlyAccess` managed policy attached to `ProductXWAFRRole`:

| Action | Already Covered By |
|---|---|
| `config:DescribeComplianceByConfigRule` | `ReadOnlyAccess` → `config:Describe*` |
| `config:GetComplianceDetailsByConfigRule` | `ReadOnlyAccess` → `config:Get*` |
| `config:DescribeConfigRules` | `ReadOnlyAccess` → `config:Describe*` |
| `config:ListDiscoveredResources` | `ReadOnlyAccess` → `config:List*` |
| `securityhub:GetFindings` | `ReadOnlyAccess` → `securityhub:Get*` |
| `securityhub:DescribeHub` | `ReadOnlyAccess` → `securityhub:Describe*` |
| `access-analyzer:ListAnalyzers` | `ReadOnlyAccess` → `access-analyzer:ListAnalyzers` (explicit) |
| `access-analyzer:ListFindings` | `ReadOnlyAccess` → `access-analyzer:ListFindings` (explicit) |

**Impact:** Zero functional change. Template becomes cleaner.

**Risk:** None. If `ReadOnlyAccess` is ever replaced with a custom policy in the future, these actions would need to be re-added explicitly.

**Counter-argument for keeping:** Serves as inline documentation of what compliance signals ProductX reads. Makes the template self-documenting for security reviewers who don't want to cross-reference the managed policy.

**Decision needed:** Remove for cleanliness, or keep for documentation value?

---

### Proposal 2: Remove `CloudvizArchitectureDiagramRole`

**Recommendation: KEEP (not actionable)**

This role creates `cloudelligent-assessment-role` with `ReadOnlyAccess`, trusted by Cloudviz's AWS account (`282762468439`). It exists solely to let Cloudviz generate architecture diagrams.

**Decision:** Cloudviz remains a required tool. ProductX does not currently support architecture diagram generation, so this role cannot be removed. The role stays in the template.

**Future consideration:** If/when ProductX adds native architecture visualization, this role becomes a candidate for removal. Until then, it remains a hard dependency for client deliverables.

---

### Proposal 3: Keep `ReservedAIRole` (Archera) in This Stack

**Recommendation: KEEP (not actionable)**

The Archera `ReservedAI-Write` role is functionally different from the rest of the template (it has write permissions, trusts a vendor account, uses a static External ID). A separation was considered but **rejected** due to the operational requirement to consolidate all onboarding roles into a single CloudFormation stack.

**Rationale for keeping consolidated:**

1. **Single deployment** — SAs deploy one stack per customer. Splitting into two stacks doubles the deployment steps, increases onboarding time, and introduces sequencing dependencies.
2. **Single rollback point** — If onboarding needs to be reversed, one stack deletion cleans up everything.
3. **Customer expectation** — Customers approve one template, not two. Fewer approval cycles = faster onboarding.
4. **Atomic state** — All roles are versioned together. No risk of drift between a "ProductX stack" and an "Archera stack" deployed at different times.

**Accepted tradeoff:** The primary template includes write permissions (S3 bucket creation, CUR report definition) that belong to Archera. This is documented in the write-permission tracker and is explainable to security teams as narrowly scoped to `reserved-ai-*` prefixes only.

---

### Proposal 4: Consolidate `BillingReadOnlyRole` into ProductX Role

**Recommendation: CONSOLIDATE (conditional on tooling migration)**

`BillingReadOnlyRole` grants billing/invoice read permissions to an internal finance user (`898668804275:user/billingviewer`). Every permission in this role is **already covered** by `ReadOnlyAccess` on `ProductXWAFRRole`.

The only reason it exists as a separate role is that it trusts a **different AWS principal** (internal finance account vs. ProductX platform account).

**Option A — Migrate billing tooling to ProductX account:**
If the `billingviewer` user/service can be moved to (or federated through) the ProductX account (`820440428496`), it could assume `ProductXWAFRRole` directly. This eliminates the separate role entirely.

**Option B — Add finance principal to ProductX role trust policy:**
Add `898668804275` as an additional trusted principal on `ProductXWAFRRole`. The finance tool would then assume ProductX's role. 

⚠️ **Risk:** This gives the finance tool access to ALL of ProductX's permissions (resource inventory, compliance signals, Well-Architected write). That violates least privilege unless you scope it down with session policies or a separate trust condition.

**Option C — Keep as-is:**
No change. The role stays separate with its own narrow permissions. Costs nothing operationally; it's just one extra role name to explain to customers.

**Questions for stakeholders:**

1. Is the billing tooling on `898668804275` still actively used?
2. Can it be migrated to the ProductX platform account?
3. Is the overhead of a separate role actually a problem (customer complaints, security review friction)?

**Impact if consolidated:** 1 fewer role per customer account.

**Risk:** Medium if Option B is chosen (over-permissioned finance access). Low if Option A (clean migration). None if Option C (status quo).

---

## Decision Matrix

| # | Proposal | Status | Impact | Effort | Risk | Decision Needed By |
|---|---|---|---|---|---|---|
| 1 | Remove `ComplianceSignalAccess` | Open | Low (template hygiene) | 5 min | None | SAs + PM |
| 2 | Remove Cloudviz role | **Not actionable** | — | — | — | — (ProductX lacks architecture tooling) |
| 3 | Separate Archera to own stack | **Not actionable** | — | — | — | — (single-stack consolidation required) |
| 4 | Consolidate BillingReadOnly | Open | Low (1 role per account) | Depends on migration | Medium | Finance + Engineering |

## Outcome After Actionable Proposals (if 1 + 4 accepted)

**Before (current):** 4 roles + 3 policies + 1 StackSet = 8 resources.

**After:**
- Remove redundant `ComplianceSignalAccess` inline policy (template simplification, no functional change)
- Potentially consolidate `BillingReadOnlyRole` (1 fewer role per account, pending finance tooling migration)

**Net result:** Template is cleaner. Core architecture (4 roles in 1 stack) remains unchanged.

---

## Next Steps

1. Circulate this ADR to reviewers
2. Collect decisions on Proposal 1 (remove ComplianceSignalAccess) and Proposal 4 (consolidate BillingReadOnly)
3. Implement accepted proposals
4. Update README.md write-permission tracker and CHANGELOG.md
