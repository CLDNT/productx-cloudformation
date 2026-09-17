# ADR-003: StackSet Auto-Deployment — Add StackInstancesGroup to Target an SA-Provided OU

Status: Accepted  
Date: 2026-08-27  
Accepted: 2026-09-17  
Author: Haider Zaidi  
Reviewers: CEO, COO, Solutions Architects, Product Manager

## Context

Prior to this decision, the `CloudelligentOnboardingMemberStackSet` resource in `Cloudelligent_WAFR_Onboarding.yaml` created a **StackSet shell only** — it defined the StackSet with `PermissionModel: SERVICE_MANAGED` and `AutoDeployment: Enabled: true`, but did **not** include a `StackInstancesGroup` property.

That meant:
1. Deploying the main stack created the StackSet definition but deployed **zero stack instances** to member accounts
2. An SA had to manually navigate to the CloudFormation console (or CLI) and add stack instances, targeting specific OUs or the org root
3. Only *after* that manual step did member accounts receive the ProductX + Cloudviz roles
4. The `AutoDeployment: Enabled` setting only kicked in for **new** accounts joining already-targeted OUs — it did not retroactively deploy to existing accounts

This manual step added friction to onboarding, was error-prone (wrong OU, wrong region), and contradicted the goal of a streamlined single-stack deployment. This ADR resolves that by adding a `StackInstancesGroup` driven by an SA-provided OU ID.

## Decision

**Add a `StackInstancesGroup` to the StackSet that targets a single AWS Organizations OU ID supplied by the SA as a parameter (`OrgUnitId`), deploying member-account roles automatically.**

Decisions locked on acceptance:
- **Auto-deploy is mandatory** in org mode — when `EnableOrganizations=yes`, the SA provides `OrgUnitId` and member roles deploy automatically at stack creation. There is no longer a "shell only, add instances later" path.
- **A single OU ID, provided once at onboarding.** It may be either the org root (`r-xxxx`) or a child OU (`ou-xxxx-xxxxxxxx`) — the SA chooses the scope. Not a list; covering a second OU later is a stack update or a separately managed instance.
- **Org Root/OU ID obtained via parameter (Option A)** — no Lambda custom resource.
- **Single region** (`AWS::Region`) — IAM roles are global, so one instance per account is sufficient.
- **Future accounts** joining the targeted OU are covered automatically via the already-enabled `AutoDeployment: Enabled: true`.

## Implemented Change

```yaml
Parameters:
  OrgUnitId:
    Type: String
    Default: ''
    AllowedPattern: '^$|^(r-[0-9a-z]{4,32}|ou-[0-9a-z]{4,32}-[0-9a-z]{8,32})$'
    Description: >-
      Required when EnableOrganizations=yes. The single AWS Organizations root
      (r-xxxx) or organizational unit (ou-xxxx-xxxxxxxx) to deploy member-account
      roles into.

CloudelligentOnboardingMemberStackSet:
  Type: AWS::CloudFormation::StackSet
  Condition: IsOrg
  Properties:
    # ... existing properties (SERVICE_MANAGED, AutoDeployment: Enabled) ...
    StackInstancesGroup:
      - DeploymentTargets:
          OrganizationalUnitIds:
            - !Ref OrgUnitId
        Regions:
          - !Ref 'AWS::Region'
```

### Resolved: how the OU ID is obtained

Resolved in favor of **Option A — a parameter (`OrgUnitId`)** the SA fills in during deployment (from the AWS Organizations console). The Lambda-custom-resource approach (Option B) was rejected: it adds a Lambda, an execution role, `organizations:ListRoots` permissions, and custom-resource rollback failure modes for a value the SA can paste in seconds. Targeting the entire org via a hardcoded root (Option C) was not chosen because the SA-provided ID already supports the root as one valid value while also allowing a narrower child OU.

## Consequences

### Positive

1. **Zero manual StackSet steps** — deploying the main stack with `EnableOrganizations=yes` immediately deploys roles to all member accounts
2. **Reduced onboarding time** — no separate "add stack instances" step for the SA
3. **Fewer errors** — no risk of targeting wrong OUs or forgetting regions
4. **Matches streamlined-onboarding intent** — a single stack deployment covers the SA-chosen OU (or the whole org, if the root is supplied)
5. **AutoDeployment continues to work** — new accounts added after initial deployment still get roles automatically

### Negative

1. **New required parameter** — in org mode the SA must provide `OrgUnitId` (one extra field during deployment).
2. **Single OU per deployment** — the StackSet targets exactly one OU (or the root). A customer wanting a subset of accounts points `OrgUnitId` at a narrower child OU; a customer wanting multiple non-nested OUs needs a stack update to change the value or a separately managed instance. This is acceptable under the agreed "one OU, once at onboarding" model.
3. **Stack deletion** — deleting the main stack will attempt to delete all stack instances across the targeted OU. If any instance is in a failed state, the main stack delete will fail. SA must clean up failed instances first.
4. **Region** — the `StackInstancesGroup` requires an explicit `Regions` list; we deploy to `AWS::Region` only (IAM roles are global, so a single region suffices).
5. **Broad posture change** — every account in the targeted OU (current and future) receives the ProductX + Cloudviz member roles, including the ADR-005 `DataPlaneReadDeny`. Consistent and intended, but the ADR-005 CUR-bucket edge case now applies per-account across the OU.

### Mitigations

- Deployment scope is controlled by **which OU ID the SA supplies** — a narrow child OU limits the blast radius; the root ID covers the whole org. This replaces the need for an opt-out flag.
- Document the "delete stack instances before deleting the main stack" requirement in the SA Runbook.
- Region is fixed to the stack's own region (`AWS::Region`) since IAM roles are global — no per-region fan-out.

## Alternatives Considered

### Alternative: Keep as-is (manual stack instances)

- SA continues to manually add instances post-deployment
- **Rejected because:** Adds friction, error-prone, contradicts the goal of a streamlined onboarding process. The original template design intended auto-deployment.

### Alternative: Lambda custom resource to discover root OU

- Eliminates the parameter but adds a Lambda, IAM role for Lambda, and custom resource
- **Rejected because:** Over-engineered for a value the SA can paste in 5 seconds. Adds failure modes (Lambda timeout, permissions issues) for minimal benefit.

## Implementation

1. ✅ **Done** — Added `OrgUnitId` parameter (accepts `r-xxxx` or `ou-xxxx-xxxxxxxx`; empty for single-account).
2. ✅ **Done** — Added `StackInstancesGroup` to the StackSet targeting `OrgUnitId` in `AWS::Region`.
3. ✅ **Decided** — Auto-deploy is mandatory in org mode (no opt-out parameter); the manual "add instances" path is removed.
4. ✅ **Done** — Updated README (parameters, deployment modes, org workflow).
5. ⏳ **Pending** — SA Runbook: document the "delete stack instances before deleting the main stack" requirement and the OU-ID onboarding step. (Runbook not yet started — tracked separately.)
6. ⏳ **Pending validation** — Deploy in a sandbox org and verify accounts in the target OU receive both roles, and that a new account joining the OU is auto-covered.

## Questions for Review

1. **Resolved** — Partial-org deployment is supported by pointing `OrgUnitId` at a specific child OU rather than the root. Full-org coverage uses the root ID. One OU per onboarding.
2. **Resolved** — The OU/root ID is provided by the SA during the onboarding call (Option A parameter). It is readily available from the AWS Organizations console.
3. **Resolved** — No manual fallback path is retained; auto-deploy is mandatory in org mode per the accepted decision. If a customer needs multiple OUs, that is handled by a stack update or a separately managed instance.
