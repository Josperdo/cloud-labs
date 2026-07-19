# Lab 1: Identity, Access & Governance Foundations for Cloud Workloads

Check box if done: [ ]

## Overview
AZ-500's identity lab is about protecting the *tenant* — break-glass accounts, PIM, Conditional Access. This lab is narrower and more workload-focused: given a resource group that hosts an actual application, how do you scope access to exactly what each identity needs, enforce that scope automatically instead of trusting people to follow a wiki page, and let workload resources authenticate to each other without a single stored credential? That's the RBAC + Azure Policy + managed identity triangle this lab builds.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (RBAC, Azure Policy, and managed identities carry no direct charge)

---

## Scenario
A new workload — a web app that reads from a storage account — is landing in its own resource group. You've been asked to set up the access model before anyone deploys anything into it: a custom role for the app team that can manage the workload's resources but can't touch RBAC or delete the resource group itself, Azure Policy guardrails so nobody can accidentally create a publicly-exposed storage account or skip tagging, and a managed identity so the app itself never holds a credential.

---

## Objectives
- Create and assign a **custom RBAC role** scoped to a workload resource group
- Author and assign **Azure Policy** definitions that enforce governance guardrails (deny, audit, and append effects)
- Use a **policy initiative** to group related policies into one assignment
- Understand **system-assigned vs. user-assigned managed identity** and when each fits
- Validate that both the RBAC boundary and the Policy guardrails actually hold

---

## Part 1: Scope RBAC to the Workload

### Step 1: Create the Resource Group
```bash
az group create --name sc500-lab1-rg --location eastus2
```

### Step 2: Why a Custom Role Instead of a Built-In One
`Contributor` on the resource group is too broad — it can create role assignments indirectly through some resource types and gives no guardrail against deleting the resource group itself. `Reader` is too narrow. The app team needs to manage the workload's resources (storage, app config, etc.) without touching access control or deleting the group wholesale. That gap is exactly what a custom role is for.

### Step 3: Author the Custom Role Definition
```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
RG_ID="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/sc500-lab1-rg"

cat > workload-operator-role.json <<EOF
{
  "Name": "SC500 Workload Operator",
  "Description": "Manage workload resources; cannot modify RBAC or delete the resource group.",
  "Actions": [
    "Microsoft.Storage/*",
    "Microsoft.Web/*",
    "Microsoft.Insights/*",
    "Microsoft.Resources/subscriptions/resourceGroups/read"
  ],
  "NotActions": [
    "Microsoft.Authorization/*/Delete",
    "Microsoft.Authorization/*/Write",
    "Microsoft.Resources/subscriptions/resourceGroups/delete"
  ],
  "AssignableScopes": [
    "$RG_ID"
  ]
}
EOF

az role definition create --role-definition workload-operator-role.json
```

**Why `NotActions` matters here**: `Microsoft.Storage/*` and `Microsoft.Web/*` are broad wildcards. `NotActions` carves the dangerous pieces back out — role and resource-group deletion — without hand-listing every individual permission the app team actually needs. This is the same pattern Microsoft's own built-in roles use.

### Step 4: Assign the Custom Role
```bash
# Replace with a real user/group object ID from your tenant
APP_TEAM_ID="<app-team-group-object-id>"

az role assignment create \
  --role "SC500 Workload Operator" \
  --assignee $APP_TEAM_ID \
  --scope $RG_ID
```

**Validation checkpoint**: `az role assignment list --scope $RG_ID --query "[].roleDefinitionName" -o tsv` should list `SC500 Workload Operator`. Confirm the role's `NotActions` by checking `az role definition list --name "SC500 Workload Operator" --query "[].permissions[0].notActions"` — the delete/write authorization actions should be present.

---

## Part 2: Azure Policy Guardrails

### Step 5: Why Policy, Not Just RBAC
RBAC controls *who* can act; Policy controls *what a valid action looks like*, regardless of who's doing it. Even a Contributor with full rights to create a storage account shouldn't be able to create one with public blob access enabled — that's a guardrail everyone is subject to, not a permission some people have and others don't.

### Step 6: Assign a Built-In Deny Policy — Block Public Blob Access
Built-in policy definition IDs are GUIDs that can vary by Azure Policy library update — look the current one up rather than hardcoding a value that may drift or no longer resolve:

```bash
PUBLIC_ACCESS_POLICY_ID=$(az policy definition list \
  --query "[?contains(displayName,'Storage accounts should prevent shared key access')].id | [0]" \
  -o tsv)

az policy assignment create \
  --name deny-public-blob-access \
  --display-name "Deny storage accounts with public blob access" \
  --policy "$PUBLIC_ACCESS_POLICY_ID" \
  --scope $RG_ID \
  --params '{}'
```

> If the query above returns nothing, browse **Policy** → **Definitions** in the portal and search for a public-access-related built-in policy — naming and exact coverage shift over time as Microsoft updates the built-in policy library.

### Step 7: Author a Custom Policy — Require a `workload` Tag
Built-in policies won't know your org's tagging taxonomy. Write a custom one.

```bash
cat > require-workload-tag.json <<EOF
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "field": "tags['workload']",
      "exists": "false"
    },
    "then": {
      "effect": "deny"
    }
  }
}
EOF

az policy definition create \
  --name require-workload-tag \
  --display-name "Require a 'workload' tag on all resources" \
  --mode Indexed \
  --rules require-workload-tag.json
```

### Step 8: Assign an Audit Policy — Flag Storage Accounts Without TLS 1.2 Minimum
Not every guardrail should hard-deny — some are better as visibility first, enforcement later, so you don't break something you didn't anticipate.

```bash
az policy assignment create \
  --name audit-min-tls \
  --display-name "Audit storage accounts below TLS 1.2 minimum" \
  --policy "storage-accounts-should-have-the-specified-minimum-tls-version" \
  --scope $RG_ID \
  --params '{"minimumTlsVersion": {"value": "TLS1_2"}}' 2>/dev/null || \
  echo "Look up the exact built-in policy name with: az policy definition list --query \"[?contains(displayName,'minimum TLS')]\""
```

### Step 9: Group Policies into an Initiative
An initiative bundles related policy assignments so they're managed and reported on as one unit — useful once you have more than two or three guardrails for a workload.

```bash
cat > workload-guardrails-initiative.json <<EOF
{
  "properties": {
    "displayName": "SC500 Workload Guardrails",
    "policyDefinitions": [
      { "policyDefinitionId": "/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Authorization/policyDefinitions/require-workload-tag" }
    ]
  }
}
EOF

az policy set-definition create \
  --name sc500-workload-guardrails \
  --display-name "SC500 Workload Guardrails" \
  --definitions workload-guardrails-initiative.json

az policy assignment create \
  --name workload-guardrails-assignment \
  --policy-set-definition sc500-workload-guardrails \
  --scope $RG_ID
```

### Step 10: Prove the Deny Guardrail Actually Blocks Something
```bash
az storage account create \
  --name sc500lab1$RANDOM \
  --resource-group sc500-lab1-rg \
  --location eastus2 \
  --sku Standard_LRS
```

**Expected result**: the create fails with a policy-denied error, because you didn't include the `workload` tag. Retry with the tag and confirm it succeeds:

```bash
az storage account create \
  --name sc500lab1$RANDOM \
  --resource-group sc500-lab1-rg \
  --location eastus2 \
  --sku Standard_LRS \
  --tags workload=sc500-demo-app
```

**Validation checkpoint**: the second command should succeed. If both fail, check `az policy assignment list --scope $RG_ID` for a stray assignment; if both succeed, confirm the policy assignment finished propagating (`az policy state list` can take several minutes to reflect a new assignment).

---

## Part 3: Managed Identity Fundamentals

### Step 11: System-Assigned vs. User-Assigned — When Each Fits
A **system-assigned** managed identity is tied to the lifecycle of a single resource — it's created and destroyed with that resource, and it can't be shared. A **user-assigned** managed identity is a standalone resource that can be attached to multiple resources at once (e.g., several App Service instances that all need the same downstream access). Use system-assigned when the identity's lifecycle should exactly match one resource; use user-assigned when multiple resources need the same identity, or when you want the identity provisioned before the resource that will use it.

### Step 12: Create a User-Assigned Managed Identity
```bash
az identity create \
  --resource-group sc500-lab1-rg \
  --name sc500-app-identity

APP_IDENTITY_ID=$(az identity show -g sc500-lab1-rg -n sc500-app-identity --query id -o tsv)
APP_IDENTITY_PRINCIPAL_ID=$(az identity show -g sc500-lab1-rg -n sc500-app-identity --query principalId -o tsv)
```

### Step 13: Create a Web App with a System-Assigned Identity, and Attach the User-Assigned One Too
A resource can have both kinds simultaneously — useful when the resource needs its own unique identity for one purpose and a shared identity for another.

```bash
az appservice plan create \
  --name sc500-lab1-plan \
  --resource-group sc500-lab1-rg \
  --sku B1 \
  --is-linux

az webapp create \
  --resource-group sc500-lab1-rg \
  --plan sc500-lab1-plan \
  --name sc500lab1app$RANDOM \
  --runtime "PYTHON:3.11"

WEBAPP_NAME=$(az webapp list -g sc500-lab1-rg --query "[0].name" -o tsv)

az webapp identity assign \
  --resource-group sc500-lab1-rg \
  --name $WEBAPP_NAME \
  --identities [system] $APP_IDENTITY_ID
```

### Step 14: Grant Least-Privilege Access to the Storage Account
```bash
STORAGE_ACCOUNT=$(az storage account list -g sc500-lab1-rg --query "[0].name" -o tsv)
STORAGE_ID=$(az storage account show -g sc500-lab1-rg -n $STORAGE_ACCOUNT --query id -o tsv)

az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee $APP_IDENTITY_PRINCIPAL_ID \
  --scope $STORAGE_ID
```

**Why `Storage Blob Data Reader` and not `Storage Blob Data Contributor`**: the app in this scenario only reads data. Granting write/delete on top of what it actually does is the same least-privilege mistake as handing out `Contributor` when `Reader` would do.

**Validation checkpoint**: `az role assignment list --assignee $APP_IDENTITY_PRINCIPAL_ID --scope $STORAGE_ID --query "[].roleDefinitionName" -o tsv` should return exactly `Storage Blob Data Reader` — nothing broader.

---

## Part 4: Cleanup

```bash
az group delete --name sc500-lab1-rg --yes --no-wait

# Policy assignments and definitions at the subscription scope aren't removed by deleting the resource group
az policy assignment delete --name deny-public-blob-access --scope /subscriptions/$SUBSCRIPTION_ID
az policy assignment delete --name audit-min-tls --scope /subscriptions/$SUBSCRIPTION_ID
az policy assignment delete --name workload-guardrails-assignment --scope /subscriptions/$SUBSCRIPTION_ID
az policy set-definition delete --name sc500-workload-guardrails
az policy definition delete --name require-workload-tag
az role definition delete --name "SC500 Workload Operator"
```

> **Note**: Policy assignments scoped to a resource group are deleted along with it, but assignments and *definitions* created at the subscription scope are not — clean them up explicitly or they'll silently apply to the next resource group you create.

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **Custom RBAC role with `NotActions`** | Grants a broad, workable permission set while explicitly carving out the dangerous slice, mirroring how Microsoft's own built-in roles are built |
| **Azure Policy deny effect** | Prevents a misconfiguration from ever being created, instead of detecting it after the fact |
| **Azure Policy audit effect** | Gives visibility into a guardrail before you commit to enforcing it and potentially breaking something |
| **Policy initiatives** | Bundles related guardrails into one manageable, reportable assignment as the guardrail count grows |
| **System-assigned vs. user-assigned managed identity** | Choosing correctly avoids either identity sprawl (one-off identities everywhere) or over-sharing (one identity with too many attached resources) |
| **Scoping identity access to a single role on a single resource** | The same least-privilege principle from RBAC, applied to workload-to-workload authentication |

---

## Common Mistakes to Avoid
- **Using `Contributor` because writing a custom role feels like overhead**: it's the single most common over-privileging mistake in real Azure environments
- **Jumping straight to `deny` policies without an `audit` trial period**: a badly-scoped deny policy can block legitimate deployments across an entire subscription with no warning
- **Forgetting that subscription-scoped Policy assignments and definitions survive resource group deletion**: they'll silently apply to the next thing you create in that subscription
- **Granting a managed identity `Contributor` on a resource when it only needs a data-plane role like `Storage Blob Data Reader`**: control-plane and data-plane roles are different — grant the narrower one
- **Not verifying the policy actually blocks non-compliant resources (Step 10)**: an unverified guardrail is a guardrail you're assuming works

---

## Next Steps
- Continue to [Lab 2: Securing Storage & Databases](lab-2-storage-database-security.md), which builds directly on this resource group's RBAC and Policy baseline
- Compare this workload-scoped RBAC approach against AZ-500's tenant-wide identity controls in [AZ-500 Lab 1](../AZ-500/lab-1-identity-access-protection.md) — both matter, at different layers
- For deeper custom-role and administrative-unit patterns, see [SC-300 Lab 1](../SC-300/lab-1-identity-governance-foundations.md)
- Extend the policy initiative with `deployIfNotExists` effects that auto-remediate non-compliant resources (e.g., auto-enable diagnostic settings) instead of only denying or auditing
