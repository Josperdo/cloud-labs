# Lab 5: Infrastructure as Code Pipelines

Check box if done: [ ]

## Overview
[Azure/General Lab 2](../General/lab-2-iac-bicep.md) covered Bicep fundamentals and [Lab 3](../General/lab-3-cicd-oidc.md) covered the plan-on-PR/apply-on-merge pattern using GitHub Actions. This lab builds the Azure DevOps-native version of that same pattern with Terraform: remote state in Azure Storage with automatic locking, a PR pipeline that runs `terraform plan` and posts the diff for review, a `main`-branch pipeline that applies behind the Prod approval gate from [Lab 3](lab-3-continuous-delivery-release-management.md), and a scheduled pipeline that catches drift — infrastructure someone changed by hand outside the pipeline — before it causes a confusing `apply` later.

**Estimated time**: 90 minutes
**Cost**: ~$0–$1 (a storage account for state plus a small resource group of lab infrastructure — delete both in Cleanup)

---

## Scenario
The team wants infrastructure changes to go through the same PR review and CI gates as application code, not `terraform apply` run ad hoc from someone's laptop. You're building the pipeline that makes that the *only* practical way to change infrastructure: state lives in a shared, locked backend; every PR shows the plan diff as a required review artifact; every merge to `main` applies automatically behind the same Prod approval used for app deploys; and a nightly job flags any manual change that's drifted the real infrastructure away from what's in Git.

---

## Objectives
- Configure a Terraform `azurerm` remote backend with state locking
- Author a small Terraform config and wire it into an Azure Pipelines PR-plan / main-apply pattern
- Reuse the workload-identity-federated service connection from Lab 4 for Terraform's Azure auth
- Build a scheduled drift-detection pipeline using `terraform plan -detailed-exitcode`
- Explain how Azure Storage blob leases provide state locking without a separate locking service

---

## Part 1: Create the Remote State Backend

### Step 1: Storage Account and Container for State
```bash
az group create --name az400-tfstate-rg --location eastus2

az storage account create \
  --name az400tfstate<unique-suffix> \
  --resource-group az400-tfstate-rg \
  --sku Standard_LRS \
  --encryption-services blob \
  --min-tls-version TLS1_2 \
  --https-only true \
  --allow-blob-public-access false

az storage container create \
  --name tfstate \
  --account-name az400tfstate<unique-suffix> \
  --auth-mode login
```

**Why this matters for locking**: the `azurerm` backend acquires a **blob lease** on the state file for the duration of any `plan` or `apply`. A second `terraform apply` attempted concurrently fails immediately with a lock error instead of both processes racing to write the same state file — no separate DynamoDB-style lock table needed, unlike Terraform's AWS S3 backend.

---

## Part 2: Author the Terraform Configuration

### Step 2: Project Structure
```bash
mkdir -p infra && cd infra
```

`backend.tf`
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  backend "azurerm" {
    resource_group_name = "az400-tfstate-rg"
    storage_account_name = "az400tfstate<unique-suffix>"
    container_name       = "tfstate"
    key                  = "az400-lab5.tfstate"
  }
}

provider "azurerm" {
  features {}
}
```

`main.tf`
```hcl
resource "azurerm_resource_group" "lab5" {
  name     = "az400-lab5-rg"
  location = "East US 2"
}

resource "azurerm_storage_account" "app_storage" {
  name                     = "az400lab5sa<unique-suffix>"
  resource_group_name      = azurerm_resource_group.lab5.name
  location                 = azurerm_resource_group.lab5.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  min_tls_version          = "TLS1_2"
  https_traffic_only_enabled = true
}

output "storage_account_name" {
  value = azurerm_storage_account.app_storage.name
}
```

### Step 3: Initialize and Verify Locally
```bash
terraform init
terraform plan -out=tfplan
```
Confirm the plan shows two resources to add before wiring it into a pipeline — debugging Terraform syntax errors inside a CI log is slower than doing it locally first.

---

## Part 3: PR Pipeline — Plan Only

### Step 4: `azure-pipelines-iac.yml`
A separate pipeline file from the app's CI pipeline keeps infra changes and app changes independently triggerable.
```yaml
# azure-pipelines-iac.yml
trigger: none

pr:
  branches:
    include:
      - main
  paths:
    include:
      - infra/*

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Plan
    displayName: 'Terraform Plan (PR)'
    jobs:
      - job: TerraformPlan
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: '1.7.5'

          - task: AzureCLI@2
            displayName: 'Terraform Init + Plan'
            inputs:
              azureSubscription: 'az400-lab3-sc-federated'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                cd infra
                export ARM_USE_OIDC=true
                export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
                terraform init
                terraform plan -no-color -out=tfplan | tee plan_output.txt
                terraform show -no-color tfplan >> plan_output.txt

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: 'infra/plan_output.txt'
              ArtifactName: 'terraform-plan'
```
`trigger: none` deliberately disables push-triggered runs on this pipeline — infra changes should only ever plan on PR and apply through the explicit Apply pipeline below, never as a side effect of an unrelated push. `ARM_USE_OIDC=true` tells the `azurerm` provider to use the federated token from the `AzureCLI@2` task's login context instead of expecting a client secret.

**Validation checkpoint**: open a PR touching `infra/main.tf`, confirm the pipeline runs and the `terraform-plan` artifact contains a readable plan diff — this is what a reviewer reads before approving, the infra equivalent of a code diff.

---

## Part 4: Main-Branch Pipeline — Apply Behind Approval

### Step 5: Apply Stage, Gated by the Prod Environment
```yaml
# azure-pipelines-iac.yml (continued — separate trigger-based pipeline, or add as a second stage gated on branch)
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infra/*

pr: none

stages:
  - stage: Apply
    displayName: 'Terraform Apply (main)'
    jobs:
      - deployment: TerraformApply
        environment: 'Prod'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: TerraformInstaller@1
                  inputs:
                    terraformVersion: '1.7.5'

                - task: AzureCLI@2
                  displayName: 'Terraform Init + Apply'
                  inputs:
                    azureSubscription: 'az400-lab3-sc-federated'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      cd infra
                      export ARM_USE_OIDC=true
                      export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
                      terraform init
                      terraform apply -auto-approve
```
Reusing the `Prod` environment from [Lab 3](lab-3-continuous-delivery-release-management.md) means the same manual-approval check gates both application deploys and infrastructure applies to production — one governance model, not two.

**Validation checkpoint**: merge the PR from Step 4, confirm the Apply pipeline starts, pauses for approval, and after approving, `terraform output storage_account_name` (run locally against the same state) reflects the newly created resource.

---

## Part 5: Scheduled Drift Detection

### Step 6: A Nightly Plan-Only Pipeline
Someone changing a resource by hand in the portal — "just this once, to unblock a demo" — is how infrastructure quietly stops matching Git. Catch it automatically.

```yaml
# azure-pipelines-drift.yml
schedules:
  - cron: '0 6 * * *'
    displayName: 'Nightly drift check'
    branches:
      include:
        - main
    always: true

trigger: none
pr: none

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: TerraformInstaller@1
    inputs:
      terraformVersion: '1.7.5'

  - task: AzureCLI@2
    displayName: 'Detect drift'
    inputs:
      azureSubscription: 'az400-lab3-sc-federated'
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        cd infra
        export ARM_USE_OIDC=true
        export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
        terraform init
        terraform plan -detailed-exitcode -no-color -out=tfplan
        exit_code=$?
        if [ $exit_code -eq 2 ]; then
          echo "##vso[task.logissue type=warning]Drift detected — infrastructure has diverged from Terraform state"
          exit 1
        elif [ $exit_code -eq 1 ]; then
          echo "Terraform plan failed"
          exit 1
        else
          echo "No drift detected"
        fi
```
`terraform plan -detailed-exitcode` returns `0` for no changes, `1` for an error, and `2` for "changes are needed" — exactly the three-way signal drift detection needs, versus the plain `plan` command which doesn't distinguish "error" from "changes pending" in its exit code.

**Validation checkpoint**: manually change a tag on `az400-lab5-rg` via `az group update` (or the portal), then manually queue the drift pipeline — confirm it fails with the drift warning logged, without applying anything.

---

## Part 6: Cleanup

```bash
cd infra
terraform destroy -auto-approve
cd ..

az storage account delete --name az400tfstate<unique-suffix> --resource-group az400-tfstate-rg --yes
az group delete --name az400-tfstate-rg --yes --no-wait

az pipelines delete --name azure-pipelines-iac --project az400-devops-lab --yes
az pipelines delete --name azure-pipelines-drift --project az400-devops-lab --yes
```

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **`azurerm` remote backend with blob-lease locking** | Prevents two concurrent `apply` runs from corrupting shared state, with no extra locking infrastructure needed |
| **Separate plan-on-PR / apply-on-merge pipelines** | Mirrors the app CI/CD split from Labs 2–3 — infra changes get the same review rigor as code |
| **`ARM_USE_OIDC` with a federated service connection** | Terraform authenticates to Azure with no stored secret, reusing the pattern hardened in Lab 4 |
| **Reusing the `Prod` environment approval** | One governance surface for both app deploys and infra applies, instead of two inconsistent ones |
| **`terraform plan -detailed-exitcode` for drift detection** | Distinguishes "no changes," "changes pending," and "error" — the signal a scheduled drift job actually needs |

---

## Common Mistakes to Avoid
- **Running `terraform apply` from a pipeline with `pr:` trigger enabled**: a malicious or careless PR should never be able to apply infrastructure changes — only `plan`
- **Sharing one state file across every environment**: use a distinct `key` per environment (or a workspace) so a Dev apply can't collide with Prod's state
- **Skipping `-detailed-exitcode` in the drift pipeline**: without it, a scheduled `plan` can't tell "everything's fine" from "the plan itself failed"
- **Forgetting `trigger: none` / `pr: none` on the pipeline that shouldn't fire from the other trigger type**: the same YAML trigger/PR-block mistake from Lab 2, with higher stakes when it's infrastructure instead of app code
- **Storing the state storage account key instead of using OIDC/managed identity for backend auth too**: `az storage container create --auth-mode login` above uses your own Entra identity for setup; the pipeline's backend access should also go through the federated service connection, not a storage account key embedded in `backend.tf`

---

## Next Steps
- [Lab 8: Package & Dependency Management](lab-8-package-dependency-management.md) ties this IaC pipeline together with the app CI/CD pipeline into one end-to-end picture
- Revisit [Azure/General Lab 2](../General/lab-2-iac-bicep.md) if you'd rather implement this same pattern in Bicep — swap the `TerraformInstaller@1`/`terraform` steps for `az deployment group what-if` (PR) and `az deployment group create` (main)
- [Lab 7: Site Reliability Engineering Practices](lab-7-site-reliability-engineering.md) covers what to do when drift detection or a bad apply actually causes an incident
