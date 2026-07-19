# Lab 4: Security Operations & Automation

Check box if done: [ ]

## Overview
Security operations work is judged on two things: do you catch real problems, and can you prove your controls are actually deployed everywhere they should be — not just in the one subscription you remembered to click through. This lab covers both. You'll enable Microsoft Defender for Cloud and work through its recommendations like a real triage session, stand up a Sentinel analytics rule that fires on a real (simulated) suspicious event, and then throw away the manual clicking entirely — codifying the whole security baseline in Terraform so it's repeatable across every subscription you'll ever manage.

**Estimated time**: 75–90 minutes
**Cost**: ~$1–$3 (Defender for Cloud plans and the Log Analytics workspace bill briefly; disable/destroy at the end — see Cleanup)

---

## Scenario
You've been asked to stand up baseline security monitoring for a new subscription: cloud security posture visibility, a way to detect suspicious control-plane activity (like someone deleting an NSG rule that shouldn't be touched), and — because you'll be asked to do this again for the next five subscriptions — a Terraform module that deploys the same baseline without you clicking through the portal five more times.

---

## Objectives
- Enable Microsoft Defender for Cloud and work through the secure score / recommendations workflow
- Understand what each Defender plan actually covers (and costs)
- Create a Log Analytics workspace and enable Microsoft Sentinel
- Build a scheduled analytics rule that detects a real control-plane event and generates an incident
- Triage a simulated incident end-to-end
- Deploy the same security baseline (Defender plans + diagnostic settings) via Terraform

---

## Part 1: Microsoft Defender for Cloud — Posture and Recommendations

### Step 1: Set Up a Target Environment
You need something with a few intentional gaps for Defender to flag. Reuse the storage account pattern from earlier labs, but skip a couple of hardening steps this time on purpose.

```bash
az group create --name az500-lab4-rg --location eastus2

az storage account create \
  --name az500lab4$RANDOM \
  --resource-group az500-lab4-rg \
  --location eastus2 \
  --sku Standard_LRS
  # Note: intentionally not setting --min-tls-version or --allow-blob-public-access false —
  # Defender for Cloud should flag both as recommendations
```

### Step 2: Enable Defender for Cloud's Free Foundational Plan
1. Portal → **Microsoft Defender for Cloud** → **Environment settings** → select your subscription
2. Confirm **Foundational CSPM** is **On** (it's free and on by default) — this gives you the secure score and recommendations engine
3. Leave the paid plans (Defender for Servers, Defender for Storage, etc.) **Off** for now — you'll enable one specific plan in Step 4 to see what it adds

### Step 3: Review Your Secure Score
1. **Microsoft Defender for Cloud** → **Overview** — note your current secure score percentage
2. **Recommendations** → sort by **Risk level** — find the recommendations related to the storage account you just created (e.g., "Secure transfer to storage accounts should be enabled," "Storage accounts should restrict network access")

### Step 4: Enable Defender for Storage and Compare
```bash
az security pricing create --name StorageAccounts --tier Standard
```

1. Return to **Recommendations** after a few minutes — Defender for Storage adds threat-detection-specific recommendations on top of the free CSPM findings (e.g., malware scanning, anomalous access alerts) that the free tier doesn't provide
2. This is the actual answer to "what does the paid plan get me" — CSPM tells you what's misconfigured; the paid Defender plans add active threat detection on top

### Step 5: Remediate One Recommendation
1. Open the "Secure transfer to storage accounts should be enabled" recommendation (or equivalent)
2. Click **Fix** if a quick-fix is offered, or remediate manually:
```bash
az storage account update \
  --name <your-storage-account> \
  --resource-group az500-lab4-rg \
  --min-tls-version TLS1_2 \
  --https-only true
```
3. **Validation checkpoint**: recommendations can take up to 24 hours to re-evaluate in a live environment — for this lab, confirm the setting changed directly: `az storage account show -n <name> -g az500-lab4-rg --query "{tls:minimumTlsVersion, https:enableHttpsTrafficOnly}"`

---

## Part 2: Microsoft Sentinel — Detection and Incident Triage

### Step 6: Create a Log Analytics Workspace and Enable Sentinel

```bash
az monitor log-analytics workspace create \
  --resource-group az500-lab4-rg \
  --workspace-name az500-sentinel-law \
  --location eastus2
```

Sentinel isn't a standalone resource — it's a solution layered onto a Log Analytics workspace.

1. Portal → **Microsoft Sentinel** → **+ Create** → select the `az500-sentinel-law` workspace → **Add**

### Step 7: Connect the Azure Activity Data Connector
This feeds Azure control-plane events (resource creates, deletes, role assignments — everything that shows up in the Activity Log) into Sentinel so you can write detections against them.

1. **Microsoft Sentinel** → **Content hub** (or **Data connectors**) → find **Azure Activity**
2. **Open connector page** → **Connect** at the subscription level
3. Confirm the connector status shows **Connected**

### Step 8: Build a Scheduled Analytics Rule
Detect something that should almost never happen and is worth immediate attention: an NSG security rule being deleted.

1. **Microsoft Sentinel** → **Analytics** → **+ Create** → **Scheduled query rule**
2. **General**: Name: `NSG Rule Deleted`, Severity: `Medium`
3. **Set rule logic** — query:
```kusto
AzureActivity
| where OperationNameValue == "MICROSOFT.NETWORK/NETWORKSECURITYGROUPS/SECURITYRULES/DELETE"
| where ActivityStatusValue == "Success"
```
4. **Run query every**: `5 minutes`, **Lookup data from the last**: `5 minutes`
5. **Alert threshold**: Generate an alert when number of results is **greater than 0**
6. **Incident settings**: Create incidents from alerts triggered by this rule — **Enabled**
7. **Review + create**

### Step 9: Trigger the Detection
Give the rule something real to catch:

```bash
az network nsg create --resource-group az500-lab4-rg --name test-nsg

az network nsg rule create \
  --resource-group az500-lab4-rg --nsg-name test-nsg \
  --name TempRule --priority 200 \
  --direction Inbound --access Allow --protocol Tcp --destination-port-ranges 22

az network nsg rule delete \
  --resource-group az500-lab4-rg --nsg-name test-nsg --name TempRule
```

### Step 10: Triage the Incident
Wait 5–10 minutes for the scheduled rule to run and the Activity Log data to flow.

1. **Microsoft Sentinel** → **Incidents** — you should see a new incident: `NSG Rule Deleted`
2. Open it → **Investigate** → review the entity (who deleted it, when, from what IP)
3. Set **Status**: `Active` → **Owner**: assign to yourself → add a comment documenting your triage ("Confirmed this was the Lab 4 exercise, no action needed") → **Status**: `Closed` → **Classification**: `True positive - expected/authorized`

**This full loop — detect, investigate, classify, close, with a documented reason — is exactly what a SOC analyst or security engineer does dozens of times a day.** The specific rule matters less than being fluent in the workflow.

---

## Part 3: Codify the Baseline with Terraform

### Step 11: Why This Matters More Than the Portal Clicks
Everything in Parts 1–2 was one subscription, done once, by hand. The real job is making sure this baseline exists in *every* subscription your org owns, consistently, without relying on someone remembering all the steps. That's what this Terraform config does: enable a Defender plan and wire up diagnostic settings to the Log Analytics workspace, as code.

### Step 12: Write the Configuration
Create a new working directory and `main.tf`:

```bash
mkdir az500-security-baseline && cd az500-security-baseline
```

```hcl
# main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

data "azurerm_subscription" "current" {}

resource "azurerm_resource_group" "baseline" {
  name     = "az500-baseline-rg"
  location = "East US 2"
}

resource "azurerm_log_analytics_workspace" "security" {
  name                = "az500-baseline-law"
  resource_group_name = azurerm_resource_group.baseline.name
  location            = azurerm_resource_group.baseline.location
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# Enable Defender for Cloud's Storage Accounts plan at the subscription level
resource "azurerm_security_center_subscription_pricing" "storage" {
  tier          = "Standard"
  resource_type = "StorageAccounts"
}

# Route subscription-level Activity Log events to the workspace
resource "azurerm_monitor_diagnostic_setting" "activity_log" {
  name                       = "activity-log-to-sentinel"
  target_resource_id         = data.azurerm_subscription.current.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.security.id

  enabled_log {
    category = "Administrative"
  }

  enabled_log {
    category = "Security"
  }

  enabled_log {
    category = "Alert"
  }
}
```

### Step 13: Deploy It

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

### Step 14: Verify

```bash
terraform output 2>/dev/null || true
az security pricing show --name StorageAccounts --query "{tier: pricingTier}"
az monitor diagnostic-settings subscription list --query "value[].name" -o tsv
```

Confirm the Defender plan and diagnostic setting both show as active — deployed identically to how you did it by hand in Part 1, but now reproducible with `terraform apply` in any subscription.

### Step 15: Tear It Down

```bash
terraform destroy -auto-approve
```

---

## Part 4: Cleanup

```bash
# Disable the paid Defender plan you enabled manually in Part 1 (destroying the RG doesn't disable subscription-level pricing)
az security pricing create --name StorageAccounts --tier Free

az group delete --name az500-lab4-rg --yes --no-wait

# Confirm the Terraform-managed resource group is gone too
az group show --name az500-baseline-rg 2>&1 | grep -i "could not be found" && echo "Cleaned up"
```

> **Important**: Defender for Cloud pricing tiers are set at the **subscription** level, not tied to any resource group. Deleting resource groups does not disable a paid plan — you must explicitly set it back to `Free` or it keeps billing.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Defender for Cloud CSPM + recommendations** | Continuous posture visibility instead of point-in-time manual audits |
| **Understanding free CSPM vs. paid Defender plans** | Lets you make an informed cost/coverage tradeoff instead of enabling everything blindly |
| **Sentinel scheduled analytics rule** | Turns raw Activity Log data into an actionable, triaged incident |
| **Full incident triage loop** | The actual day-to-day workflow of security operations, not just the detection engineering half |
| **Terraform-deployed security baseline** | Consistent, auditable, repeatable across every subscription — not dependent on a checklist and a human remembering it |

---

## Common Mistakes to Avoid
- **Assuming Defender for Cloud's free tier includes threat detection**: it's posture/configuration visibility (CSPM) only — active threat detection requires the paid plans
- **Forgetting Defender pricing tiers are subscription-scoped**: deleting the resource group where you tested something doesn't stop the subscription-level plan from billing
- **Writing analytics rules with no clear "why"**: every detection should map to a specific risk (this lab's NSG-delete rule maps directly to the network hardening from [Lab 2](lab-2-platform-network-security.md))
- **Closing incidents without a documented classification**: "True positive," "False positive," or "Benign positive" with a reason is what makes an incident log useful for later audits and pattern analysis
- **Treating the Terraform config as a one-off**: the entire point is reusing it — check it into a shared module repo, not a throwaway lab folder, once you're doing this for real

---

## Next Steps
- Extend the Sentinel rule from Step 8 to also alert on break-glass account sign-ins (from [Lab 1](lab-1-identity-access-protection.md)) and Key Vault secret access from unexpected identities (from [Lab 3](lab-3-data-app-security.md))
- Add `azurerm_security_center_subscription_pricing` resources for the other Defender plans (VirtualMachines, KeyVaults, AppServices) to the Terraform baseline
- Look into Sentinel automation rules and playbooks (Logic Apps) to auto-remediate low-severity incidents instead of just alerting on them
- Continue to [Lab 5: Hybrid Identity & Azure AD Connect Security](lab-5-hybrid-identity-adconnect.md) to extend this operations baseline to the on-prem/cloud identity bridge and its Tier-0 hardening requirements
