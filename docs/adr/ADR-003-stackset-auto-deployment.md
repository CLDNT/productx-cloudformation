# ADR-003: StackSet Auto-Deployment — Add StackInstancesGroup to Target Org Root

Status: Proposed  
Date: 2026-08-27  
Author: Haider Zaidi  
Reviewers: CEO, COO, Solutions Architects, Product Manager

## Context

The `CloudelligentOnboardingMemberStackSet` resource in `Cloudelligent_WAFR_Onboarding.yaml` currently creates a **StackSet shell only** — it defines the StackSet with `PermissionModel: SERVICE_MANAGED` and `AutoDeployment: Enabled: true`, but does **not** include a `StackInstancesGroup` property.

This means:
1. Deploying the main stack creates the StackSet definition but deploys **zero stack instances** to member accounts
2. An SA must manually navigate to the CloudFormation console (or CLI) and add stack instances, targeting specific OUs or the org root
3. Only *after* that manual step do member accounts receive the ProductX + Cloudviz roles
4. The `AutoDeployment: Enabled` setting only kicks in for **new** accounts joining already-targeted OUs — it does not retroactively deploy to existing accounts

This manual step adds friction to the onboarding process, is error-prone (wrong OU, wrong region), and contradicts the original design intent of a single-stack deployment.

## Decision

**Add a `StackInstancesGroup` property to the StackSet resource, targeting the AWS Organizations root.**

This will cause the StackSet to automatically deploy stack instances to **all member accounts** in the organization at stack creation time, without manual intervention.

## Proposed Change

```yaml
CloudelligentOnboardingMemberStackSet:
  Type: AWS::CloudFormation::StackSet
  Condition: IsOrg
  Properties:
    StackSetName: CloudelligentOnboardingMemberStackSet
    # ... existing properties ...
    StackInstancesGroup:
      - DeploymentTargets:
          OrganizationalUnitIds:
            - !GetAtt RootOULookup.Value   # or passed as parameter
        Regions:
          - !Ref 'AWS::Region'
```

### Open question: How to obtain the Org Root OU ID

The Org Root OU ID (e.g. `r-xxxx`) is required for `OrganizationalUnitIds`. Options:

**Option A — New parameter:**  
Add an `OrgRootId` parameter that the SA fills in during deployment. The SA gets this from AWS Organizations console (it's the top-level ID starting with `r-`).

**Option B — Custom resource (Lambda):**  
A Lambda-backed custom resource calls `organizations:ListRoots` to retrieve the root OU ID at deploy time. Adds complexity (Lambda role, code, timeout handling) but eliminates the manual parameter.

**Option C — Hardcode targeting all accounts (AccountFilterType: NONE):**  
Use `AccountFilterType: NONE` with the org root OU. This is the simplest path but requires the root OU ID.

**Recommendation:** Option A (new parameter) — simple, no Lambda overhead, SA already has console access during the onboarding call.

## Consequences

### Positive

1. **Zero manual StackSet steps** — deploying the main stack with `EnableOrganizations=yes` immediately deploys roles to all member accounts
2. **Reduced onboarding time** — no separate "add stack instances" step for the SA
3. **Fewer errors** — no risk of targeting wrong OUs or forgetting regions
4. **Matches original design intent** — the template was intended to be a single deployment covering the entire org
5. **AutoDeployment continues to work** — new accounts added after initial deployment still get roles automatically

### Negative

1. **New parameter** — SA must provide the Org Root OU ID (one extra field during deployment)
2. **All-or-nothing** — targets the entire org root; if a customer only wants a subset of accounts, this won't work (fallback: leave `StackInstancesGroup` out and add manually, as today)
3. **Stack deletion** — deleting the main stack will attempt to delete all stack instances across the org. If any instance is in a failed state, the main stack delete will fail. SA must clean up failed instances first.
4. **Region** — must specify which region(s) to deploy to; the `StackInstancesGroup` requires an explicit `Regions` list

### Mitigations

- Make `StackInstancesGroup` conditional on a new parameter (e.g. `AutoDeployToOrg` = yes/no) so SAs can opt out for partial-org deployments
- Document the "delete failed instances before deleting main stack" requirement in the SA Runbook
- Default region to the stack's own region (`AWS::Region`) since IAM roles are global

## Alternatives Considered

### Alternative: Keep as-is (manual stack instances)

- SA continues to manually add instances post-deployment
- **Rejected because:** Adds friction, error-prone, contradicts the goal of a streamlined onboarding process. The original template design intended auto-deployment.

### Alternative: Lambda custom resource to discover root OU

- Eliminates the parameter but adds a Lambda, IAM role for Lambda, and custom resource
- **Rejected because:** Over-engineered for a value the SA can paste in 5 seconds. Adds failure modes (Lambda timeout, permissions issues) for minimal benefit.

## Implementation

1. Add `OrgRootId` parameter (optional, only needed when `EnableOrganizations=yes`)
2. Add `StackInstancesGroup` to the StackSet resource, targeting `OrgRootId` in region `AWS::Region`
3. Consider making this conditional on a new `AutoDeployToOrg` parameter for flexibility
4. Update README deployment modes section
5. Update SA Runbook — remove manual "add stack instances" step
6. Test: deploy in a sandbox org, verify all member accounts receive roles

## Questions for Review

1. Do we have any customers who need partial-org deployment (subset of OUs only)?
2. Is the Org Root OU ID (`r-xxxx`) readily available during the onboarding call, or would the SA need to look it up separately?
3. Should we keep the manual path as a fallback (conditional parameter) or go all-in on auto-deploy?
