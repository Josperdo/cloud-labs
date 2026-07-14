# Lab 7: Landing Zone & Governance at Scale

Check box if done: [ ]

## Overview
"Cloud Adoption Framework," "landing zone," and "policy-as-code" show up constantly in enterprise and platform-engineering job descriptions, but they're easy to nod along to without ever having built the thing. This lab builds the terminology hands-on: a real CAF landing zone management group shape, a multi-policy initiative bundling several guardrails into one assignment, and a policy definition deployed through Bicep instead of the portal — extending the management-group and single-policy work from AZ-305 Lab 1 into a broader governance-at-scale, Infrastructure-as-Code lens.

**Estimated time**: 60-75 minutes
**Cost**: ~$0 (management groups, Policy, and Bicep deployments at management group scope have no cost)

---

## Scenario
Your organization is adopting Microsoft's Cloud Adoption Framework to structure a multi-team, multi-subscription Azure estate — new teams get a subscription through a defined "subscription vending" process instead of filing an ad hoc request that lands wherever. You're documenting the landing zone management group structure so platform and workload teams share a common mental model, building a Policy initiative that bundles several landing zone guardrails into one assignment instead of tracking three separate ones, and proving that a custom policy definition can be deployed the same way application code is — through a template, reviewed and version-controlled, not clicked together once in the portal and forgotten.

---

## Objectives
- Map CAF landing zone terminology (Platform, Landing Zones, Sandbox, Decommissioned) to a real management group hierarchy
- Bundle three landing zone guardrails into a single Policy initiative and assign it at management group scope
- Understand what Azure Verified Modules are and where to find them for Bicep and Terraform
- Deploy a custom Policy definition via Bicep (`az deployment mg create`) instead of the portal — policy-as-code
- Validate policy compliance state and confirm the custom definition actually deployed before calling it done

---

## Part 1: CAF Landing Zone Structure (Document, Don't Just Deploy)

### Step 1: The Standard CAF Shape
The Cloud Adoption Framework's reference management group hierarchy sits under the Tenant Root Group as an intermediate root, then splits into two branches with distinct purposes:

- **Intermediate Root** (e.g., `contoso-root`) — the top-level container for the whole estate. Policy and RBAC assigned here inherit to everything underneath, present and future.
- **Platform** — subscriptions that provide shared services *to* workloads, not workloads themselves, further split into:
  - **Management** — centralized logging, automation accounts, Log Analytics
  - **Connectivity** — hub networking, VPN/ExpressRoute gateways, DNS, firewall
  - **Identity** — domain controllers or identity infrastructure kept separate from workload subscriptions so an identity outage doesn't cascade from an app team's mistake
- **Landing Zones** — where workloads actually run, split into:
  - **Corp** — internal-facing apps, typically peered to the hub for on-prem connectivity
  - **Online** — internet-facing apps, often with lighter or no hub dependency
- **Sandbox** — isolated experimentation with no connectivity back to production; developers can get elevated rights here safely because nothing here can reach anything that matters
- **Decommissioned** — a holding area for subscriptions pending deletion, keeping them out of the active hierarchy while preserving an audit trail until cleanup actually happens

**Subscription vending** is the process this hierarchy enables: a new team requests a subscription, it gets created from a standard template (baseline policy, baseline RBAC, correct network peering) and dropped straight into the right landing zone group — no manual policy/RBAC setup repeated per subscription.

### Step 2: Extend or Build the Hierarchy
If you completed **AZ-305 Lab 1**, you already have `contoso-root` → `contoso-landingzones` (with `contoso-production` / `contoso-nonproduction` underneath) and `contoso-sharedservices` as a sibling. That maps to CAF's Landing Zones branch and a simplified Platform branch. Add the remaining CAF groups to complete the shape:

```bash
az account management-group create --name "contoso-platform" \
  --display-name "Platform" --parent "contoso-root"
az account management-group create --name "contoso-sandbox" \
  --display-name "Sandbox" --parent "contoso-root"
az account management-group create --name "contoso-decommissioned" \
  --display-name "Decommissioned" --parent "contoso-root"
```

If you're starting fresh for this lab, build a minimal illustrative version instead — same idea, fewer groups:

```bash
az account management-group create --name "contoso-root" --display-name "Contoso"
az account management-group create --name "contoso-landingzones" \
  --display-name "Landing Zones" --parent "contoso-root"
az account management-group create --name "contoso-platform" \
  --display-name "Platform" --parent "contoso-root"
az account management-group create --name "contoso-sandbox" \
  --display-name "Sandbox" --parent "contoso-root"
```

**Validation checkpoint**:
```bash
az account management-group list -o table
az account management-group show --name "contoso-root" --expand -o jsonc
```
Confirm the tree shows `contoso-platform`, `contoso-sandbox`, and (if built) `contoso-decommissioned` as children of `contoso-root`, alongside the Landing Zones branch.

---

## Part 2: A Multi-Policy Initiative for a Landing Zone

An initiative (policy set) bundles multiple definitions into one assignment — one compliance evaluation, one assignment to manage, instead of three that can drift out of sync with each other. This one bundles three real landing zone guardrails: deny public IP creation, require diagnostic settings, and restrict allowed VM SKUs.

### Step 1: Look Up the Built-In Definition IDs
Look these up dynamically rather than hardcoding GUIDs — built-in policy IDs are stable but typo-prone to copy by hand, and this is the same workflow you'd use against a real tenant.

```bash
DENY_PUBLIC_IP_ID=$(az policy definition list \
  --query "[?displayName=='Not allowed resource types'].id | [0]" -o tsv)
REQUIRE_DIAG_ID=$(az policy definition list \
  --query "[?displayName=='Audit diagnostic setting'].id | [0]" -o tsv)
ALLOWED_VM_SKU_ID=$(az policy definition list \
  --query "[?displayName=='Allowed virtual machine size SKUs'].id | [0]" -o tsv)
```

### Step 2: Build the Initiative Definition
```bash
cat > landingzone-guardrails.json <<EOF
{
  "properties": {
    "displayName": "Landing Zone Guardrails",
    "policyDefinitions": [
      { "policyDefinitionId": "$DENY_PUBLIC_IP_ID",
        "policyDefinitionReferenceId": "denyPublicIp",
        "parameters": {
          "listOfResourceTypesNotAllowed": {
            "value": ["Microsoft.Network/publicIPAddresses"]
          }
        } },
      { "policyDefinitionId": "$REQUIRE_DIAG_ID",
        "policyDefinitionReferenceId": "requireDiagnosticSettings" },
      { "policyDefinitionId": "$ALLOWED_VM_SKU_ID",
        "policyDefinitionReferenceId": "restrictVmSkus",
        "parameters": {
          "listOfAllowedSKUs": {
            "value": ["Standard_B1s", "Standard_B2s", "Standard_D2s_v3"]
          }
        } }
    ]
  }
}
EOF

az policy set-definition create --name "landingzone-guardrails" \
  --display-name "Landing Zone Guardrails" \
  --management-group "contoso-landingzones" \
  --definitions landingzone-guardrails.json
```

**Why deny public IPs at this scope**: landing zone workloads are expected to reach the internet (if at all) through the hub's firewall/NAT gateway in the Connectivity subscription, not by attaching a public IP directly to a NIC — the guardrail enforces that architectural decision instead of relying on every team remembering it.

### Step 3: Assign at Management Group Scope
```bash
az policy assignment create --name "landingzone-guardrails-assignment" \
  --display-name "Landing Zone Guardrails" \
  --policy-set-definition "landingzone-guardrails" \
  --management-group "contoso-landingzones" \
  --enforcement-mode Default
```

Assigning at `contoso-landingzones` — not per-subscription — means a new subscription vended into `contoso-production` or `contoso-nonproduction` next quarter inherits all three guardrails with zero extra work.

**Validation checkpoint**:
```bash
az policy assignment list \
  --scope "/providers/Microsoft.Management/managementGroups/contoso-landingzones" -o table

az policy state trigger-scan --management-group "contoso-landingzones"
az policy state summarize --management-group "contoso-landingzones"
```
The summary should list all three definitions inside the initiative with a resource compliance count (0 resources evaluated is expected if the landing zone subscriptions are still empty — the point is confirming the assignment and its definitions are correctly registered).

---

## Part 3: Intro to Azure Verified Modules (AVM)

Azure Verified Modules are Microsoft-published, tested infrastructure modules for both Bicep and Terraform — VNets, storage accounts, AKS clusters, and full landing zone patterns — maintained centrally so individual teams stop hand-rolling the same VNet module for the tenth time with subtly different defaults. Each module is versioned with semver, has a documented parameter interface, and passes a shared set of conformance tests (naming, tagging, security defaults) before publication.

**Where to find them**:
- **Bicep** — the [Bicep Public Registry](https://azure.github.io/Azure-Verified-Modules/indexes/bicep/bicep-resource-modules/), referenced with the `br/public:avm/...` alias
- **Terraform** — the [Terraform Registry](https://registry.terraform.io/namespaces/Azure), under modules published by the `Azure` org (`avm-res-*` for single-resource modules, `avm-ptn-*` for multi-resource patterns like a hub-spoke network or a full landing zone)

`avm/res/...` modules wrap one resource type (analogous to the hand-written `storage.bicep` module from Lab 2, but tested and versioned by Microsoft). `avm/ptn/...` modules wrap a whole pattern — a landing zone subscription vending pipeline, for example — composed from several `avm/res` modules underneath.

Referencing an AVM storage account module in Bicep, pinned to a specific version:

```bicep
module storageAccount 'br/public:avm/res/storage/storage-account:0.14.3' = {
  name: 'avmStorageDeploy'
  params: {
    name: storageAccountName
    location: location
    skuName: 'Standard_LRS'
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}
```

Same idea Lab 2 covered with a hand-written module and the `module` keyword — the only difference is the source is `br/public:avm/...` instead of a local `.bicep` file, and someone else maintains, tests, and version-bumps it.

---

## Part 4: Deploy a Policy Definition via Policy-as-Code (Bicep)

Governance itself should be version-controlled and code-reviewed, the same as application infrastructure — a policy definition clicked together once in the portal has no diff history and no PR review trail. This part deploys a custom Policy definition through Bicep at management group scope.

### Step 1: Write the Bicep Template
Create `policy-definition.bicep`:

```bicep
targetScope = 'managementGroup'

@description('Name of the custom policy definition')
param policyName string = 'require-environment-tag-deny'

@description('Display name shown in the Azure Policy portal')
param policyDisplayName string = 'Require environment tag (deny)'

resource requireEnvironmentTag 'Microsoft.Authorization/policyDefinitions@2023-04-01' = {
  name: policyName
  properties: {
    displayName: policyDisplayName
    description: 'Denies resource creation if the environment tag is missing.'
    policyType: 'Custom'
    mode: 'Indexed'
    parameters: {}
    policyRule: {
      if: {
        field: 'tags[\'environment\']'
        exists: 'false'
      }
      then: {
        effect: 'deny'
      }
    }
  }
}

output policyDefinitionId string = requireEnvironmentTag.id
```

`targetScope = 'managementGroup'` is what makes this deployable directly at management group level instead of a resource group — the same Bicep language, a different scope keyword.

### Step 2: Deploy at Management Group Scope
```bash
az deployment mg create \
  --management-group-id "contoso-landingzones" \
  --location eastus \
  --template-file policy-definition.bicep \
  --name landingzone-custom-policy-deploy
```

**Validation checkpoint**:
```bash
az policy definition show --name "require-environment-tag-deny" \
  --management-group "contoso-landingzones"

az deployment mg show --management-group-id "contoso-landingzones" \
  --name landingzone-custom-policy-deploy --query properties.provisioningState
```
Should return `"Succeeded"`, and the policy definition should show up alongside the built-in ones from Part 2 — proving the same `az deployment` workflow used for VNets and storage accounts in Lab 2 works identically for governance objects.

---

## Cleanup

Order matters — assignment before initiative, initiative and custom definition before the management groups that host them, and management groups leaf-to-root.

```bash
# 1. Assignment before the initiative that backs it
az policy assignment delete --name "landingzone-guardrails-assignment" \
  --management-group "contoso-landingzones"
az policy set-definition delete --name "landingzone-guardrails" \
  --management-group "contoso-landingzones"

# 2. Custom policy definition from Part 4
az policy definition delete --name "require-environment-tag-deny" \
  --management-group "contoso-landingzones"

# 3. Management groups created purely for this lab (leaf before root)
az account management-group delete --name "contoso-sandbox"
az account management-group delete --name "contoso-decommissioned"
az account management-group delete --name "contoso-platform"
```

If `contoso-landingzones`, `contoso-production`, `contoso-nonproduction`, `contoso-sharedservices`, and `contoso-root` came from AZ-305 Lab 1 and you're continuing to use that hierarchy, leave them in place. Otherwise remove your subscription first and delete the rest leaf-to-root the same way AZ-305 Lab 1's cleanup does.

Confirm with:
```bash
az account management-group list -o table
```
Only the groups you intended to keep (or just the Tenant Root Group) should remain.

---

## Key Concepts

| Term | Definition |
|---|---|
| **Landing zone** | A pre-provisioned, pre-governed subscription (or set of subscriptions) a workload team deploys into — an ongoing structure with baked-in policy, RBAC, and networking, not a one-time deployment |
| **CAF management group structure** | The reference hierarchy — Platform (Management/Connectivity/Identity) and Landing Zones (Corp/Online), plus Sandbox and Decommissioned — that gives policy and RBAC assignment consistent, predictable inheritance points |
| **Subscription vending** | The process of provisioning a new subscription from a standard template and dropping it into the correct landing zone group, instead of an ad hoc request that lands with no baseline governance |
| **Policy initiative vs. definition** | A definition is one rule with one effect; an initiative bundles several definitions into a single assignment so they're evaluated and managed together instead of drifting independently |
| **Azure Verified Modules (AVM)** | Microsoft-published, tested Bicep and Terraform modules for common resources (`avm/res/...`) and multi-resource patterns (`avm/ptn/...`), versioned centrally instead of every team writing its own |
| **Policy-as-code** | Deploying policy definitions and assignments through a reviewed, version-controlled template (Bicep/Terraform/ARM) instead of the portal, so governance changes go through the same PR review as application code |
| **Management group scope inheritance** | Policy and RBAC assigned at a management group apply to every subscription nested beneath it, including subscriptions added after the assignment was made |

---

## Common Mistakes
- **Treating "landing zone" as a single deployment** instead of an ongoing structure — a landing zone is the standing baseline every new subscription lands into, not a project that finishes
- **Writing a custom Bicep/Terraform module before checking Azure Verified Modules** — reinventing a VNet or storage module Microsoft already publishes, tests, and version-bumps wastes effort and produces something less battle-tested
- **Assigning policy at too low a scope** (a single subscription instead of the landing zone management group) — it works for that one subscription but misses the entire point of centralized governance: new subscriptions won't inherit it
- **Forgetting management group deletion requires children removed first** — Azure refuses to delete a group with subscriptions or child groups still attached, so cleanup has to go leaf-to-root
- **Pinning nothing when referencing an AVM module** — always pin an explicit version (`:0.14.3`, not floating), the same discipline as pinning a Terraform provider or module version

---

## Next Steps
This closes out the General track. It extends the management group and single-policy work from [AZ-305 Lab 1](../AZ-305/lab-1-identity-governance-monitoring-design.md) into a multi-policy initiative and adds the policy-as-code deployment pattern on top of the Bicep fundamentals from [Lab 2: Infrastructure as Code with Bicep](lab-2-iac-bicep.md) in this same folder.
