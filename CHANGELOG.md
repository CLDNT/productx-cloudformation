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

## [2.0] — 2026-08-27

### Added: ComputeOptimizerPolicy (Compute Optimizer fallback)

**What changed:**  
Added a dedicated `ComputeOptimizerPolicy` to `ProductXWAFRRole` (management account) and `ComputeOptimizerAccess` to `ProductXWAFRMemberRole` (member accounts via StackSet). Grants read-only `compute-optimizer:Get*`/`Describe*` actions.

**Why:**  
AWS Compute Optimizer serves as a **fallback/supplement** to the ProductX Cost Optimization Engine. The engine does not depend on Compute Optimizer — it computes its own recommendations from CloudWatch metrics — but native AWS recommendations are surfaced alongside the engine's findings when available. This gives customers a second data point and covers any resource types the engine hasn't yet modeled.

**Customer impact:**
- New customers: Compute Optimizer recommendations appear alongside ProductX findings (read-only, no enrollment required by ProductX)
- Existing customers: Need stack update to gain the fallback recommendations

**Sales talking point:**  
> "ProductX runs its own proprietary optimization engine AND surfaces AWS Compute Optimizer's native recommendations as a cross-check — you get both, all read-only."

**Action required:**

| Who | Action |
|-----|--------|
| SAs | Existing clients need a stack update to receive Compute Optimizer fallback recommendations |
| Product Manager | Confirm the ProductX UI surfaces Compute Optimizer data as a supplement, not a primary source |

---

### Clarified: Cost Optimization Engine vs. Compute Optimizer (documentation)

**What changed:**  
Updated README source-of-truth table and business use case mapping to reference the ProductX Cost Optimization Engine design document (`docs/knowledge/design.md`) as the authoritative source for FinOps permission requirements. Clarified the relationship between the proprietary engine and AWS Compute Optimizer.

**Why:**  
The engine collects CloudWatch metrics directly and runs its own proprietary methodology. It does not call AWS Compute Optimizer APIs, read its recommendations, or check enrollment as part of its core analysis. Compute Optimizer is a **fallback/supplement** (see the `ComputeOptimizerPolicy` entry above), not a dependency — this distinction matters for security reviews and accurate positioning.

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
> "ProductX runs its own proprietary cost optimization engine — it doesn't rely on AWS Compute Optimizer for its analysis. It collects metrics directly from CloudWatch and applies a deeper methodology, then cross-checks against Compute Optimizer's native recommendations as a supplement."

**Action required:**

| Who | Action |
|-----|--------|
| Sales | Position the engine as proprietary + Compute Optimizer as a supplementary cross-check |
| Product Manager | Reference `docs/knowledge/design.md` as the canonical source for engine capabilities and coverage |
| SAs | When customers ask "do we need to enable Compute Optimizer?" — no, ProductX works independently but will surface its recommendations if present |

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

**Sales talking point:**  
> "One CloudFormation stack covers your entire AWS Organization. Deploy once in the management account, add member accounts to the StackSet, and every account is connected — no per-account role setup."

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
