# Lab 2: Identity & Access Architecture Design

Check box if done: [ ]

## Overview
Identity is the control plane Zero Trust actually runs on — Lab 1's roadmap put it in Phase 1 for exactly that reason. But "turn on Conditional Access" isn't a design; a tenant with fifty overlapping CA policies is often *less* secure than one with five well-layered ones, because nobody can predict which policy wins when three apply to the same sign-in. This lab designs a layered Conditional Access model and a Privileged Identity Management (PIM) strategy at enterprise scale, then implements a representative slice of both.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (Conditional Access and PIM require Entra ID P2 — use a P2 trial; no billable Azure resources are created)

---

## Scenario
The 2,500-employee org from Lab 1 has three distinct populations hitting the same tenant: full-time employees on managed devices, contractors on unmanaged BYOD, and a growing set of workload identities (service principals, managed identities) calling internal APIs. A single "require MFA for everyone" policy — Lab 1's Phase 1 quick win — was a reasonable first step, but it's too blunt for what comes next: contractors need tighter device and session controls than employees, admin roles need standing-access elimination, and workload identities need their own Conditional Access treatment since they can't complete an MFA prompt. You need a policy layering model that scales to more personas and more apps without becoming an unmanageable pile of exceptions.

---

## Objectives
- Design a Conditional Access layering model that avoids policy conflicts and unpredictable evaluation order
- Decide between standing privileged access, just-in-time (JIT) PIM-eligible roles, and Privileged Access Groups
- Build persona-scoped Conditional Access policies (employee, contractor, workload identity) in report-only mode
- Configure a PIM-eligible role assignment requiring justification and time-bound activation
- Validate policy layering behavior using the "what if" tool and sign-in log evidence
- Bundle multiple privileged roles into one JIT-eligible Privileged Access Group for a rotating platform team

---

## Part 1: Design Decision — Conditional Access Layering Model

### Decision 1: Policy Organization — Persona-Based vs. Risk-Based vs. Flat App-Based

| Factor | Persona-Based (one policy set per user population: employee/contractor/workload) | Risk-Based (driven by Identity Protection sign-in/user risk score) | Flat App-Based (one policy per application, duplicated per app) |
|---|---|---|---|
| **Coverage completeness** | High — every population gets an explicit, intentional policy; nothing falls through by default | Adaptive — strong for anomaly response, but risk score alone doesn't express "contractors always need a compliant device" | Low — easy to forget a persona-specific requirement when adding the 30th app |
| **Policy count / manageability** | Moderate — scales with number of personas, not number of apps | Low — a handful of risk-tier policies can cover the whole tenant | High — grows linearly with every new app, becomes unmanageable past ~15–20 apps |
| **Conflict / evaluation predictability** | Predictable if layered deliberately (baseline → persona → app-specific → session) | Can interact unpredictably with persona policies unless risk is treated as one layer, not the whole model | High conflict risk — near-duplicate policies per app drift out of sync over time |
| **Fit for this scenario** | Strong — directly expresses "contractors are different from employees are different from workload identities" | Complementary, not sufficient alone — should be one layer, not the whole strategy | Poor — this org already has 3 distinct personas and dozens of apps; flat model won't hold |

### Decision 2: Standing Access vs. Just-in-Time PIM vs. Privileged Access Groups

| Factor | Standing Admin Assignment (permanent role membership) | PIM-Eligible Roles (JIT activation, time-bound, requires justification/approval) | Privileged Access Groups (PIM-eligible membership in a group that owns the role) |
|---|---|---|---|
| **Attack surface if credential is compromised** | Full role privilege available immediately and indefinitely — this is what let Lab 1's attacker escalate | Privilege only exists during an active, time-boxed, logged activation window | Same JIT benefit as PIM roles, plus the group can hold multiple role assignments across resources, so one eligible membership governs many privileges consistently |
| **Auditability** | Weak — "why does this account have Global Admin" has no answer beyond "it always has" | Strong — every activation is logged with justification, approver, and duration | Strong, and centralizes the audit trail per group instead of per individual role assignment |
| **Operational overhead** | Lowest day-to-day, highest risk | Moderate — users must actively activate, which adds friction admins sometimes resent | Slightly higher setup cost (group + role mapping), lower ongoing overhead than managing PIM per role per user |
| **Best fit** | Never, for privileged roles, in a Zero Trust design | Individual privileged roles held by a small number of admins | Cross-resource privilege bundles (e.g., "Subscription Owner + Key Vault Administrator" for a platform team) held by multiple rotating people |

### Recommendation for This Scenario
**Layered Conditional Access**, evaluated in this order: (1) tenant-wide baseline (block legacy auth, require MFA — Lab 1's Phase 1 policy), (2) persona-specific layers (contractor: compliant device + blocked persistent browser session; employee: hybrid-joined or compliant device; workload identity: FAPI/token-binding-appropriate policy scoped to service principals, since interactive controls like MFA don't apply), (3) app-specific session controls (e.g., stricter session timeout for the finance app from Lab 1's incident). **PIM-eligible roles for every individual privileged assignment**, with **Privileged Access Groups** for the platform team's multi-role bundle. No standing admin assignments survive this design. Part 2–3 implement the persona layer and the PIM-eligible role.

---

## Part 2: Build the Persona-Scoped Conditional Access Layer

### Step 1: Confirm the Baseline Policy from Lab 1 Still Exists
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies?\$filter=startswith(displayName,'SC100')" \
  --query "value[].{name:displayName, state:state}" -o table
```
If Lab 1 was cleaned up, recreate it first — the persona layer in this lab is designed to sit *on top of* that baseline, not replace it.

### Step 2: Create the Contractor Persona Policy (Report-Only)
Contractors get a tighter device requirement than employees, since BYOD can't be attested as compliant the way a corporate-managed device can.
```bash
cat > ca-policy-contractor.json << 'EOF'
{
  "displayName": "SC100-Lab2-Contractors-RequireCompliantDevice-ReportOnly",
  "state": "enabledForReportingButNotEnforced",
  "conditions": {
    "users": {
      "includeGroups": ["<contractors-group-object-id>"],
      "excludeUsers": ["<break-glass-account-object-id>"]
    },
    "applications": { "includeApplications": ["All"] }
  },
  "grantControls": {
    "operator": "AND",
    "builtInControls": ["mfa", "compliantDevice"]
  },
  "sessionControls": {
    "persistentBrowser": { "isEnabled": true, "mode": "never" }
  }
}
EOF

az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --body @ca-policy-contractor.json --headers "Content-Type=application/json"
```
`persistentBrowser: never` prevents the "stay signed in" option — on unmanaged contractor hardware, a persistent session is a persistent credential sitting on a device you don't control.

### Step 3: Create the Workload Identity Policy
Service principals and managed identities cannot complete an interactive MFA challenge — they need their own policy scoped to the workload identity sign-in type, restricting them to trusted locations rather than requiring an interactive control that doesn't apply.
```bash
cat > ca-policy-workload.json << 'EOF'
{
  "displayName": "SC100-Lab2-WorkloadIdentities-TrustedLocationOnly-ReportOnly",
  "state": "enabledForReportingButNotEnforced",
  "conditions": {
    "clientApplications": { "includeServicePrincipals": ["ServicePrincipalsInMyTenant"] },
    "applications": { "includeApplications": ["All"] },
    "locations": { "includeLocations": ["All"], "excludeLocations": ["AllTrusted"] }
  },
  "grantControls": { "operator": "OR", "builtInControls": ["block"] }
}
EOF

az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --body @ca-policy-workload.json --headers "Content-Type=application/json"
```
This is a Workload Identities Conditional Access policy (a distinct policy type from user CA) — it blocks service principal sign-ins from outside trusted network locations, closing the gap where a leaked client secret could be used from anywhere on the internet.

**Validation checkpoint**:
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies?\$filter=startswith(displayName,'SC100-Lab2')" \
  --query "value[].{name:displayName, state:state}" -o table
```
Both policies should show `enabledForReportingButNotEnforced` — following the same report-first discipline from Lab 1 before either is enforced.

---

## Part 3: Configure a PIM-Eligible Privileged Role Assignment

### Step 1: Confirm No Standing Assignment Exists for the Target Role
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleAssignments?\$filter=roleDefinitionId eq '<security-administrator-role-template-id>'" \
  --query "value[].principalId" -o table
```
Any principal listed here holds the role permanently — the target for this design is zero permanent holders of privileged roles.

### Step 2: Create a PIM-Eligible Assignment (Time-Bound, Requires Justification)
```bash
cat > pim-eligible-assignment.json << 'EOF'
{
  "action": "adminAssign",
  "justification": "SC-100 Lab 2 - PIM eligible assignment for platform admin, JIT activation only",
  "roleDefinitionId": "<security-administrator-role-template-id>",
  "directoryScopeId": "/",
  "principalId": "<target-user-object-id>",
  "scheduleInfo": {
    "startDateTime": "<current-utc-timestamp>",
    "expiration": { "type": "afterDuration", "duration": "P180D" }
  }
}
EOF

az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleEligibilityScheduleRequests" \
  --body @pim-eligible-assignment.json --headers "Content-Type=application/json"
```
`afterDuration: P180D` forces re-justification every 180 days rather than granting permanent eligibility — eligibility itself expires, not just each activation.

### Step 3: Require Approval and MFA on Activation
```bash
cat > pim-role-settings.json << 'EOF'
{
  "roleDefinitionId": "<security-administrator-role-template-id>",
  "rules": [
    { "id": "Enablement_EndUser_Assignment", "ruleType": "RoleManagementPolicyEnablementRule",
      "enabledRules": ["MultiFactorAuthentication", "Justification"] },
    { "id": "Approval_EndUser_Assignment", "ruleType": "RoleManagementPolicyApprovalRule",
      "setting": { "isApprovalRequired": true } }
  ]
}
EOF
```
Applying this policy object to the role's PIM management policy (via the `roleManagementPolicyAssignments`/`roleManagementPolicies` Graph endpoints) means activation isn't just time-boxed — it requires a fresh MFA challenge, a written justification, and a named approver, giving every privilege escalation a reviewable trail.

**Validation checkpoint**:
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleEligibilityScheduleRequests?\$filter=principalId eq '<target-user-object-id>'" \
  --query "value[].{role:roleDefinitionId, status:status}" -o table
```
`status` should show `Provisioned` — the user is now PIM-eligible, not standing-assigned. They must actively activate the role (with MFA + justification) to gain privilege, and it expires automatically when the activation window ends.

---

## Part 4: Build a Privileged Access Group for the Platform Team

The Decision 2 recommendation reserved Privileged Access Groups for multi-role bundles held by rotating team members, rather than managing PIM settings per role per individual — the platform team from this scenario needs both Subscription Owner and Key Vault Administrator, and re-doing Part 3's per-role PIM setup for both roles for every platform engineer doesn't scale.

### Step 1: Create the Security Group That Will Own the Roles
```bash
az ad group create \
  --display-name "SC100-Lab2-Platform-Team-PIM" \
  --mail-nickname "sc100lab2platformteampim" \
  --description "PIM-eligible membership grants Subscription Owner + Key Vault Administrator via role-assignable group"
```

### Step 2: Enable the Group as Role-Assignable
A group must be created with `isAssignableToRole: true` to be eligible for PIM role-assignable group behavior — this can't be toggled after creation, it has to be set at group creation time.
```bash
GROUP_ID=$(az ad group show --group "SC100-Lab2-Platform-Team-PIM" --query id -o tsv)

az rest --method PATCH \
  --url "https://graph.microsoft.com/v1.0/groups/$GROUP_ID" \
  --body '{"isAssignableToRole": true}' --headers "Content-Type=application/json"
```

### Step 3: Assign Roles to the Group, Then Make Group Membership PIM-Eligible
```bash
az role assignment create \
  --assignee-object-id "$GROUP_ID" --assignee-principal-type Group \
  --role "Key Vault Administrator" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/<key-vault-resource-group>"

cat > pim-group-eligible-membership.json << 'EOF'
{
  "action": "adminAssign",
  "justification": "SC-100 Lab 2 - platform team JIT membership, bundles Key Vault Administrator role",
  "accessId": "member",
  "groupId": "<group-id-from-step-1>",
  "principalId": "<platform-engineer-object-id>",
  "scheduleInfo": {
    "startDateTime": "<current-utc-timestamp>",
    "expiration": { "type": "afterDuration", "duration": "P180D" }
  }
}
EOF

az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identityGovernance/privilegedAccess/group/eligibilityScheduleRequests" \
  --body @pim-group-eligible-membership.json --headers "Content-Type=application/json"
```
One PIM-eligible group membership now governs every role assigned to the group — adding a second bundled role later (e.g., Network Contributor, if the platform team's scope grows) requires one more `az role assignment create` against the group, not a second parallel PIM setup per engineer.

**Validation checkpoint**:
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identityGovernance/privilegedAccess/group/eligibilityScheduleRequests?\$filter=groupId eq '$GROUP_ID'" \
  --query "value[].{principal:principalId, status:status}" -o table
```
Confirm the platform engineer shows `status: Provisioned` for group membership eligibility — activating membership (with the same MFA + justification + approval discipline as Part 3) grants both bundled roles for the activation window, then both expire together.

---

## Cleanup
```bash
for name in "SC100-Lab2-Contractors-RequireCompliantDevice-ReportOnly" "SC100-Lab2-WorkloadIdentities-TrustedLocationOnly-ReportOnly"; do
  POLICY_ID=$(az rest --method GET \
    --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies?\$filter=displayName eq '$name'" \
    --query "value[0].id" -o tsv)
  az rest --method DELETE --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$POLICY_ID"
done

ELIGIBILITY_ID=$(az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleEligibilityScheduleRequests?\$filter=principalId eq '<target-user-object-id>'" \
  --query "value[0].id" -o tsv)
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleEligibilityScheduleRequests" \
  --body "{\"action\":\"adminRemove\",\"roleDefinitionId\":\"<security-administrator-role-template-id>\",\"directoryScopeId\":\"/\",\"principalId\":\"<target-user-object-id>\"}" \
  --headers "Content-Type=application/json"

az role assignment delete --assignee-object-id "$GROUP_ID" --role "Key Vault Administrator" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/<key-vault-resource-group>"
az ad group delete --group "SC100-Lab2-Platform-Team-PIM"
```
Confirm with the same list query from Part 2 Step 3 and Part 3's validation checkpoint — both should return empty.

---

## What You Practiced

| Concept | Why It Matters |
|---|---|
| Conditional Access layering order | SC-100 scenario questions routinely describe conflicting policies and ask which one wins — layering discipline is the answer |
| Persona-based policy design | Real tenants have multiple populations with different risk profiles; one policy for everyone under- or over-protects someone |
| Workload identity Conditional Access | A frequently-missed pillar — service principals and managed identities need their own policy type, not user MFA controls |
| PIM-eligible vs. standing access | The single highest-leverage change against credential-based lateral movement, and directly tied to Lab 1's incident |
| Privileged Access Groups | Scales JIT access across multiple role assignments without managing PIM settings per role per user |

---

## Common Mistakes to Avoid
- **Building app-specific policies before persona-specific ones**: without the persona layer, every new app re-solves "what does a contractor need" from scratch and drifts out of sync
- **Forgetting workload identities need a separate Conditional Access policy type**: user-targeted CA policies silently don't apply to service principal sign-ins — this is a common tenant-wide blind spot
- **Granting PIM eligibility with no expiration**: eligibility that never expires just moves the standing-access problem one step back instead of solving it
- **Skipping approval requirements on high-privilege role activation**: JIT alone (self-service activation, no approver) still lets a compromised credential activate privilege on demand
- **Not excluding the break-glass account from every layer**: a persona or workload policy without the same exclusion as the baseline policy reintroduces lockout risk

---

## Next Steps
- Continue to [Lab 3: Security Operations Architecture Design](lab-3-security-operations-architecture-design.md) to design where the sign-in and PIM activation logs generated here actually get detected and investigated
- For hands-on Conditional Access and PIM configuration depth, see [AZ-500 Lab 1: Identity & Access Protection](../AZ-500/lab-1-identity-access-protection.md) and [SC-300 Lab 3: Access Governance](../SC-300/lab-3-access-governance.md)
- The day-to-day identity governance operations this design enables are covered in [SC-300](../SC-300/README.md) — SC-100 designs the layering strategy, SC-300 runs it
