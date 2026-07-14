# Lab 2: Infrastructure as Code with Bicep

Check box if done: [ ]

## Overview
This repo already covers Terraform in depth (AZ-104 Labs 6–7) — provider config, state, modules. But "Terraform-only" is a gap on a resume for Azure-native-tooling-heavy roles: plenty of Azure teams standardize on Bicep because it's a first-party ARM abstraction with no third-party state to manage. This lab builds the same category of infrastructure in Bicep so the portfolio demonstrates real IaC breadth, not just one tool.

**Estimated time**: 60-75 minutes
**Cost**: ~$0-$1

---

## Scenario
You've been asked to stand up a small piece of networking and storage infrastructure — a VNet with one subnet and a storage account — but this team deploys everything through Bicep, not Terraform. You need a parameters file so the same template deploys differently per environment without editing the template itself, and a `what-if` preview before every apply so nobody finds out what a deployment does by watching it happen in production. You also need to be ready to explain in an interview why you'd reach for Bicep instead of Terraform on a given project — "because that's what the team uses" is true but not a complete answer.

---

## Objectives
- Install and validate the Bicep CLI tooling
- Write Bicep syntax fundamentals: `param`, `resource`, `output`, and resource references
- Author a parameters file for environment-specific values
- Run and interpret a `what-if` deployment preview before every apply
- Extract a resource into a reusable Bicep module and reference it with the `module` keyword
- Reason through Bicep vs. Terraform trade-offs well enough to defend a tool choice in an interview

---

## Part 1: Install Bicep and Author main.bicep

### Step 1: Install the Bicep CLI
Bicep ships as an extension to the Azure CLI — no separate download required.

```bash
az bicep install
az bicep version
```

**What this does**: `az bicep install` pulls the Bicep compiler binary and wires it into the `az` CLI. `az bicep version` confirms it's on PATH and reports the compiler version — useful to pin against in a README when your VS Code Bicep extension and CLI compiler versions drift apart.

### Step 2: Create a Project Directory
```bash
mkdir bicep-azure-network
cd bicep-azure-network
```

### Step 3: Write main.bicep
Create `main.bicep`:

```bicep
@description('Base name used to derive resource names')
param namePrefix string

@description('Azure region for all resources')
param location string = resourceGroup().location

@description('Environment label — dev, staging, or prod')
@allowed([
  'dev'
  'staging'
  'prod'
])
param environment string = 'dev'

@description('Address space for the virtual network')
param vnetAddressPrefix string = '10.0.0.0/16'

@description('Address prefix for the app subnet')
param subnetAddressPrefix string = '10.0.1.0/24'

var vnetName = '${namePrefix}-vnet-${environment}'
var subnetName = 'app-subnet'
// Storage account names must be globally unique, lowercase, no hyphens, <=24 chars
var storageAccountName = toLower('${namePrefix}st${environment}${uniqueString(resourceGroup().id)}')

resource vnet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name: vnetName
  location: location
  tags: {
    environment: environment
    managed_by: 'bicep'
  }
  properties: {
    addressSpace: {
      addressPrefixes: [
        vnetAddressPrefix
      ]
    }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: subnetAddressPrefix
        }
      }
    ]
  }
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  tags: {
    environment: environment
    managed_by: 'bicep'
  }
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    supportsHttpsTrafficOnly: true
  }
}

output vnetId string = vnet.id
output subnetId string = vnet.properties.subnets[0].id
output storageAccountName string = storageAccount.name
```

**Reading this**: `param` declares an input, `resource` declares a resource with a symbolic name (`vnet`, `storageAccount`) you use to reference it elsewhere — same idea as a Terraform local name, but Bicep resolves dependencies from these references automatically, same as Terraform's implicit graph. `resourceGroup().location` and `resourceGroup().id` are ARM template functions Bicep exposes directly — there's no separate provider block to configure because Bicep only ever targets Azure.

**Validation checkpoint**: Compile the file to catch syntax errors before you ever call Azure.
```bash
az bicep build --file main.bicep
```
This produces `main.json` (the compiled ARM template) with no errors. **Bicep always compiles down to ARM JSON before deployment** — Bicep is a syntax layer on top of ARM, not a separate deployment engine, which is why every Bicep feature has a 1:1 ARM equivalent under the hood.

---

## Part 2: Parameters File and What-If Deployment

### Step 4: Author main.parameters.json
Create `main.parameters.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "namePrefix": {
      "value": "lab2"
    },
    "environment": {
      "value": "dev"
    },
    "vnetAddressPrefix": {
      "value": "10.0.0.0/16"
    },
    "subnetAddressPrefix": {
      "value": "10.0.1.0/24"
    }
  }
}
```

**Why a parameters file instead of inline `--parameters` flags**: it's version-controlled, diffable in a PR, and you keep one file per environment (`main.parameters.dev.json`, `main.parameters.prod.json`) without touching `main.bicep` at all — the direct equivalent of Terraform's `terraform.tfvars` per environment.

### Step 5: Create the Resource Group and Preview with What-If
```bash
az group create --name rg-bicep-lab2 --location eastus

az deployment group what-if \
  --resource-group rg-bicep-lab2 \
  --template-file main.bicep \
  --parameters main.parameters.json
```

**What this does**: `what-if` compiles the template, evaluates it against what's already deployed in the resource group, and prints the diff — without changing anything. This is the Bicep equivalent of `terraform plan`, and skipping it before `apply`/`create` is the same mistake in either tool.

**Reading the output**: each resource line is prefixed with a marker:
- `+` **Create** — resource doesn't exist yet, will be created
- `~` **Modify** — resource exists, specific properties will change (the diff shows old → new per property)
- `-` **Delete** — resource exists in Azure but isn't in your template (only shown if you deploy in `Complete` mode, not the default `Incremental` mode)
- `=` **NoChange** — resource matches the template exactly, nothing happens

On a fresh resource group you should see two `+ Create` entries: the VNet and the storage account.

**Validation checkpoint**: Confirm the what-if output shows exactly 2 resources to create and zero to modify or delete before proceeding.

---

## Part 3: Deploy and Verify

### Step 6: Deploy
```bash
az deployment group create \
  --resource-group rg-bicep-lab2 \
  --template-file main.bicep \
  --parameters main.parameters.json \
  --name lab2-initial-deploy
```

**Wait ~1-2 minutes.** The command returns the outputs block (`vnetId`, `subnetId`, `storageAccountName`) once deployment succeeds.

**Validation checkpoint**: Confirm both resources exist.
```bash
az resource list --resource-group rg-bicep-lab2 --output table
```
You should see one `Microsoft.Network/virtualNetworks` and one `Microsoft.Storage/storageAccounts` resource. Also check the deployment itself:
```bash
az deployment group show --resource-group rg-bicep-lab2 --name lab2-initial-deploy --query properties.provisioningState
```
Should return `"Succeeded"`.

---

## Part 4: A Reusable Bicep Module

### Step 7: Extract the Storage Account into storage.bicep
Create `storage.bicep`:

```bicep
@description('Name of the storage account')
param storageAccountName string

@description('Azure region')
param location string

@description('Environment label for tagging')
param environment string

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  tags: {
    environment: environment
    managed_by: 'bicep'
  }
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    supportsHttpsTrafficOnly: true
  }
}

output storageAccountName string = storageAccount.name
output storageAccountId string = storageAccount.id
```

### Step 8: Reference the Module from main.bicep
Replace the `storageAccount` resource block in `main.bicep` with a `module` call:

```bicep
module storage 'storage.bicep' = {
  name: 'storageDeploy'
  params: {
    storageAccountName: storageAccountName
    location: location
    environment: environment
  }
}
```

Update the output at the bottom of `main.bicep` to pull from the module instead of the inline resource:
```bicep
output storageAccountName string = storage.outputs.storageAccountName
```

**Why this matters**: a module is Bicep's unit of reuse and composition — the same storage module can be called from a dozen different `main.bicep` files across projects with different parameters, the same way a Terraform module in a `modules/` directory gets called with different `source` blocks. It also scopes the storage account's implementation details behind a clean parameter interface.

**Validation checkpoint**: Redeploy and confirm nothing actually changes — the module produces the exact same ARM resource as the inline block did, just declared differently.
```bash
az deployment group what-if \
  --resource-group rg-bicep-lab2 \
  --template-file main.bicep \
  --parameters main.parameters.json
```
Expect `NoChange` on both the VNet and the storage account. **This proves idempotency**: re-running a deployment against infrastructure that already matches the template is a no-op, regardless of whether the resource was declared inline or through a module.

---

## Part 5: Design Decision — Bicep vs. Terraform

| Dimension | Bicep | Terraform |
|---|---|---|
| **State management** | Stateless — ARM tracks deployment history server-side; no local/remote state file to lock, corrupt, or secure | Explicit state file (local or remote backend); powerful but it's an operational artifact you must protect and share correctly |
| **Multi-cloud** | Azure-only | Multi-cloud (AWS, GCP, Azure, etc.) with one workflow |
| **Day-0 Azure feature support** | Usually first — Bicep is maintained by the ARM team, so brand-new API versions land immediately | Lags — the `azurerm` provider needs a release cycle to catch up to new ARM API versions |
| **Module ecosystem** | Azure Verified Modules (Microsoft-maintained, smaller but curated) | Terraform Registry (much larger, community + partner modules across every provider) |
| **Team familiarity** | Natural fit for teams already deep in ARM templates or pure-Azure shops | Natural fit for teams spanning multiple clouds or already invested in HCL tooling |

**No state file cuts both ways**: it removes an entire operational failure mode (state drift, lock contention, a corrupted state file blocking every future deploy), but it also means Bicep can't do a local `plan` against a state snapshot the way Terraform can — `what-if` always makes a live call to Azure to diff against reality.

**When I'd pick each**: Bicep, for an Azure-only environment where I want zero state-file operational overhead and same-day support for new Azure resource types. Terraform, for anything multi-cloud, or where the team already has HCL expertise and wants one workflow across providers instead of a different IaC tool per cloud.

---

## Cleanup

```bash
az group delete --name rg-bicep-lab2 --yes --no-wait
```

**No local state file to clean up.** Unlike the `terraform.tfstate` file from AZ-104 Lab 6, Bicep/ARM deployments keep their history in Azure itself — deleting the resource group removes the resources, but the deployment records persist at the subscription/resource-group scope until explicitly removed. If you want a clean slate (or you're rotating through this lab repeatedly), delete the deployment history too:

```bash
az deployment group delete --resource-group rg-bicep-lab2 --name lab2-initial-deploy
```

Note this only works while the resource group still exists — once `az group delete` finishes, the deployment history goes with it.

---

## Key Concepts

| Term | Definition |
|---|---|
| **Bicep vs. ARM template** | Bicep is a domain-specific language that compiles to ARM JSON (`az bicep build`); every Bicep deployment is an ARM deployment under the hood — Bicep just removes the JSON boilerplate |
| **Idempotency** | Re-running the same deployment against infrastructure that already matches the template produces no changes (`NoChange` in what-if) — the defining property that makes IaC safe to re-apply |
| **What-if** | A dry-run that diffs your template against live Azure state and prints Create/Modify/Delete/NoChange without deploying anything — Bicep's equivalent of `terraform plan` |
| **Module** | A separate `.bicep` file referenced via the `module` keyword with its own `param`/`output` interface — Bicep's unit of reuse and composition |
| **Parameters vs. variables** | `param` values come from outside the template (a parameters file, CLI flag, or default) at deploy time; `var` values are computed inside the template and can't be overridden externally |
| **Incremental vs. Complete deployment mode** | Incremental (the default) only adds/updates resources in the template and leaves unrelated resources in the resource group alone; Complete deletes anything in the resource group not defined in the template — a destructive mode used rarely and deliberately |
| **Deployment history** | ARM records every deployment (template, parameters, outcome) at the scope it targeted; this is what what-if diffs against instead of a local state file |

---

## Common Mistakes
- **Skipping `what-if` before `create`**: the same mistake as running `terraform apply` without reading `plan` first — you find out what a deployment does by watching it happen instead of before
- **Assuming Bicep has drift detection like a state file gives Terraform**: Bicep has no local record of "what it manages" — `what-if` diffs the template against live Azure state at that moment, so it doesn't warn you about resources someone else changed outside of any deployment unless you re-run it
- **Hardcoding values that should be parameters**: region, environment label, and address prefixes belong in `main.parameters.json`, not baked into `main.bicep` — otherwise every environment needs a template edit
- **Forgetting storage account naming constraints**: lowercase, no hyphens, 3-24 characters, globally unique — `uniqueString(resourceGroup().id)` is the standard pattern to avoid collisions without hardcoding a suffix
- **Not pinning API versions on resource types**: `Microsoft.Storage/storageAccounts@2023-01-01` — omitting or mismatching this can silently change which properties are available or valid

---

## Next Steps
Compare this against the Terraform approach to the same category of infrastructure in [AZ-104 Lab 6](../AZ-104/lab-6-terraform-fundamentals.md) and [AZ-104 Lab 7](../AZ-104/lab-7-terraform-modules.md) — same VNet/storage concepts, different tooling philosophy. [Lab 3](lab-3-cicd-oidc.md) picks up from here and deploys this same Bicep template through a GitHub Actions pipeline using OIDC federated auth instead of a stored credential.
