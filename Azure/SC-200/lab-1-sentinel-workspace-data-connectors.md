# Lab 1: Microsoft Sentinel Workspace & Data Connectors

Check box if done: [ ]

## Overview
Every SOC analyst task in this track — writing a query, tuning a detection, working an incident, hunting proactively — depends on telemetry actually landing in a workspace first. This lab builds that foundation the way an analyst joining a new tenant actually would: stand up a Log Analytics workspace with a deliberate retention/cost posture, layer Microsoft Sentinel on top of it, connect the two data sources nearly every SC-200 scenario touches (Azure Activity Log and Entra ID sign-in/audit logs), and — critically — verify data is actually flowing before declaring the job done. AZ-500 Lab 4 covered this same ground at a "deploy it and build one rule" level; here the focus is analyst-depth: connector health, ingestion validation, and understanding what each table actually contains before you're asked to query it.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$1 (Log Analytics ingestion during Sentinel's free 31-day/10GB-per-day trial; a few hours of testing in this lab is negligible — see Cleanup)

---

## Scenario
You've just joined the SOC for a mid-size org and been handed a subscription with no monitoring in place. Before you can write a single detection or work a single incident, you need telemetry flowing somewhere you can query it. Your first-day task: stand up the workspace, enable Sentinel, connect the two foundational data sources, and prove — with a query, not a screenshot — that data is actually arriving.

---

## Objectives
- Design and deploy a Log Analytics workspace with an appropriate SKU and retention policy
- Enable Microsoft Sentinel on the workspace
- Connect the Azure Activity Log data connector (subscription control-plane events)
- Connect Entra ID sign-in and audit log diagnostic settings (identity telemetry)
- Validate data ingestion with KQL rather than assuming the portal's "Connected" status is the whole story
- Understand connector health monitoring and what to check when a connector goes quiet

---

## Part 1: Deploy the Log Analytics Workspace

### Step 1: Create the Resource Group and Workspace
```bash
az group create --name sc200-lab1-rg --location eastus2

az monitor log-analytics workspace create \
  --resource-group sc200-lab1-rg \
  --workspace-name sc200-sentinel-law \
  --location eastus2 \
  --sku PerGB2018 \
  --retention-time 90
```

**Why these choices matter for the exam and the job**: `PerGB2018` is the standard pay-as-you-go SKU — Sentinel requires it (or a commitment tier at scale) and it's what you'll see in nearly every real deployment. 90-day retention is a common compliance baseline; Sentinel itself extends free retention to 90 days on top of whatever the workspace is set to, but the workspace setting still governs non-Sentinel tables and matters once you're paying for it beyond the free tier.

### Step 2: Enable Microsoft Sentinel on the Workspace
```bash
az extension add --name sentinel

az sentinel onboarding-state create \
  --resource-group sc200-lab1-rg \
  --workspace-name sc200-sentinel-law \
  --name default
```

Sentinel is a solution layered onto a Log Analytics workspace, not a standalone resource — this is why the workspace has to exist first. You can confirm the same thing in the portal: **Microsoft Sentinel** → **+ Create** → select `sc200-sentinel-law` → **Add**, if you'd rather click through it once to see where this lives.

**Expected result**: `az sentinel onboarding-state show --resource-group sc200-lab1-rg --workspace-name sc200-sentinel-law --name default` returns without error, confirming Sentinel is onboarded to the workspace.

### Step 3: Understand What You Just Committed To
```bash
az monitor log-analytics workspace show \
  --resource-group sc200-lab1-rg \
  --workspace-name sc200-sentinel-law \
  --query "{sku: sku.name, retention: retentionInDays}"
```
Confirm the SKU and retention match what you set. This is the number a manager or an exam question will ask you to reason about when estimating ingestion cost — know how to check it, not just how to set it.

---

## Part 2: Connect the Azure Activity Log Data Connector

This feeds subscription control-plane events (resource creates/deletes, role assignments, everything visible in the Activity Log) into Sentinel. It's the connector nearly every "who changed what" investigation starts with.

### Step 4: Enable the Connector
```bash
az monitor diagnostic-settings subscription create \
  --name "activity-log-to-sentinel" \
  --location eastus2 \
  --workspace "/subscriptions/<subscription-id>/resourceGroups/sc200-lab1-rg/providers/Microsoft.OperationalInsights/workspaces/sc200-sentinel-law" \
  --logs '[{"category": "Administrative", "enabled": true}, {"category": "Security", "enabled": true}, {"category": "Alert", "enabled": true}, {"category": "Policy", "enabled": true}]'
```

Or via the portal: **Microsoft Sentinel** → **Content hub** → search **Azure Activity** → **Install** → **Open connector page** → **Connect** at the subscription level.

### Step 5: Confirm Connector Status
1. **Microsoft Sentinel** → **Data connectors** → **Azure Activity** — status should show **Connected**
2. Note this is a *configuration* status, not proof of data flow — that comes in Part 4

---

## Part 3: Connect Entra ID Sign-In and Audit Logs

Identity telemetry — who signed in, from where, and what administrative actions they took — is the backbone of nearly every identity-related detection in Labs 3, 5, and 7.

### Step 6: Configure Entra ID Diagnostic Settings
This requires **Entra ID P1** at minimum; without it, `SigninLogs` won't populate.

1. Portal → **Entra ID** → **Monitoring** → **Diagnostic settings** → **+ Add diagnostic setting**
2. **Name**: `entra-logs-to-sentinel`
3. **Log categories**: check `SignInLogs`, `AuditLogs`, `NonInteractiveUserSignInLogs`, `ServicePrincipalSignInLogs`
4. **Destination**: **Send to Log Analytics workspace** → select `sc200-sentinel-law`
5. **Save**

### Step 7: Install the Corresponding Sentinel Connector
1. **Microsoft Sentinel** → **Content hub** → search **Microsoft Entra ID** → **Install**
2. **Open connector page** — this connector reflects the diagnostic setting you just configured rather than requiring separate authentication; confirm it shows the data types as **Connected**

**Validation checkpoint**: The Entra ID connector page in Sentinel and the diagnostic setting you created in Step 6 should reference the same workspace. If the connector page shows "Not connected" after a few minutes, re-check the diagnostic setting was actually saved — a common miss is selecting the destination but not clicking Save on the diagnostic setting itself.

---

## Part 4: Validate Data Is Actually Flowing

Don't trust "Connected" status alone — an analyst's first move on a quiet dashboard is always to query the raw table.

### Step 8: Query for Activity Log Data
```kusto
AzureActivity
| where TimeGenerated > ago(1h)
| project TimeGenerated, OperationNameValue, ActivityStatusValue, Caller, ResourceGroup
| sort by TimeGenerated desc
| take 20
```
If this returns nothing, generate an event to prime it:
```bash
az group create --name sc200-lab1-test-rg --location eastus2
```
Then re-run the query a few minutes later — Activity Log ingestion into Sentinel typically lands within 5–15 minutes.

### Step 9: Query for Entra ID Sign-In Data
```kusto
SigninLogs
| where TimeGenerated > ago(1h)
| project TimeGenerated, UserPrincipalName, AppDisplayName, IPAddress, ResultType
| sort by TimeGenerated desc
| take 20
```
Sign in to the Azure portal yourself if the table is empty — that generates a sign-in event you can use to confirm ingestion.

**Expected result**: Both queries return rows within 15 minutes of the triggering activity. `ResultType == "0"` means a successful sign-in; anything else is a failure code you'll use heavily starting in Lab 2.

### Step 10: Check Connector Health Programmatically
```kusto
SentinelHealth
| where TimeGenerated > ago(1d)
| where SentinelResourceKind == "AzureActivityLog" or SentinelResourceKind == "AADConnector"
| project TimeGenerated, SentinelResourceKind, Status, Description
| sort by TimeGenerated desc
```
The `SentinelHealth` table is how you'd actually monitor connector reliability at scale instead of clicking through the Data Connectors blade — this is the table a real health-monitoring analytics rule would query.

---

## Part 5: Cleanup

```bash
# Remove the test resource group created to prime Activity Log data
az group delete --name sc200-lab1-test-rg --yes --no-wait

# If you're not continuing to Lab 2 immediately, remove the diagnostic setting to stop Entra log ingestion
az monitor diagnostic-settings subscription delete --name "activity-log-to-sentinel"

# Delete the lab resource group (this also removes the workspace and Sentinel onboarding)
az group delete --name sc200-lab1-rg --yes --no-wait
```

> **If you're continuing to Lab 2 or Lab 3 in the same session, keep `sc200-sentinel-law`** — subsequent labs in this track build directly on this workspace and its connectors. Only tear it down if you're stopping here for good.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Deploying a Log Analytics workspace with deliberate SKU/retention** | The cost and compliance decisions behind "just enable Sentinel" that a real SOC has to own |
| **Enabling Sentinel via CLI, not just the portal** | Repeatable, scriptable onboarding — the same instinct AZ-500's Terraform lab pushes at the infrastructure layer |
| **Connecting Azure Activity and Entra ID data sources** | The two connectors nearly every other SC-200 lab and real investigation depends on |
| **Validating ingestion with KQL instead of trusting connector status** | "Connected" in the UI is a configuration state, not proof of data — analysts verify with a query |
| **Querying `SentinelHealth` for connector monitoring** | How you'd actually catch a silently-broken connector before it costs you a missed detection |

---

## Common Mistakes to Avoid
- **Assuming "Connected" status means data is flowing**: it means the configuration is valid — always confirm with a direct query against the table, especially before trusting a detection built on top of it
- **Forgetting Entra ID diagnostic settings require P1**: without it, `SigninLogs` stays empty no matter how correctly the connector is configured
- **Not saving the diagnostic setting itself**: selecting log categories and a destination but skipping the final Save is the single most common reason a connector shows "Not connected"
- **Over-provisioning retention without understanding the cost**: Sentinel gives 90 days free on Sentinel-specific tables regardless of workspace retention — setting workspace retention far beyond that only adds cost for non-Sentinel tables
- **Treating workspace deployment as a one-time task**: connector health should be checked periodically (Step 10's `SentinelHealth` query), not assumed to stay working forever

---

## Next Steps
- Continue to [Lab 2: KQL Fundamentals for Security Analysts](lab-2-kql-fundamentals.md) to start querying the `SigninLogs` and `AzureActivity` data you just connected
- Compare this workspace-first approach to [AZ-500 Lab 4](../AZ-500/lab-4-security-ops-automation.md), which deploys the same foundation via Terraform — worth doing once you're comfortable with the manual/CLI version here
- For the architecture-level question of *how many workspaces* a real org should run (single vs. multi-workspace, Azure Lighthouse for MSSPs), see [SC-100](../SC-100/README.md)
