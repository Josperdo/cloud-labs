# Lab 5: Monitoring — Getting Visibility Before You Go Live

## Overview
Deploying resources is only half the job. Before anything goes to production, you need to make sure you'll know when something breaks — before your users or manager tells you. This lab walks through setting up logging, writing queries to answer real operational questions, creating a meaningful alert, and building a simple health dashboard your team can actually use.

**Estimated time**: 55–70 minutes
**Cost**: ~$2–$5 (Log Analytics ingestion; tear down at the end)

---

## Scenario
Your team just deployed a new set of resources and your manager asks: "Are we set up to know if something goes wrong?" Right now, the answer is no. By the end of this lab, you'll have centralized logging in place, a KQL query you can run during an incident to understand what happened, an alert that fires before issues escalate, and a dashboard the team can check at a glance.

---

## Objectives
- Set up a Log Analytics workspace as a central log store
- Enable diagnostic logging on deployed resources
- Write KQL queries to answer real operational questions
- Create an alert that fires on conditions that actually matter
- Build a simple team-facing health dashboard

---

## Part 1: Create the Log Analytics Workspace

### Step 1: Set Up Centralized Log Storage
Log Analytics is where logs from all your resources end up. Think of it as a searchable database of everything that's happened across your Azure environment. You want one workspace per environment (dev, staging, prod) so logs don't mix.

1. Search **Log Analytics workspaces** → **+ Create**
2. **Basics tab:**
   - **Resource Group**: Create new → `monitoring-lab-rg`
   - **Name**: `prod-logs`
   - **Region**: `East US`
   - **Pricing tier**: `Pay-as-you-go`
3. Click **Review + create** → **Create**

**Wait ~2 minutes** for deployment.

**In real orgs**: The workspace name would typically reflect the environment and team, like `payments-prod-logs` or `platform-eastus-logs`. One workspace per team or per environment keeps log queries fast and costs predictable.

---

## Part 2: Create Resources and Enable Logging

### Step 2: Deploy a Storage Account
This simulates a resource your application depends on — maybe it's storing uploads, exports, or audit files.

1. Search **Storage accounts** → **+ Create**
2. **Basics:**
   - **Resource Group**: `monitoring-lab-rg`
   - **Storage account name**: `appfiles<randomnumber>`
   - **Region**: `East US`
   - **Redundancy**: `LRS`
3. Click **Review + create** → **Create**

### Step 3: Enable Diagnostic Logging on the Storage Account
By default, Azure doesn't collect detailed logs — you have to opt in. This is a step that's often skipped and regretted later when something goes wrong and there's no audit trail.

1. Open the storage account → Left sidebar → **Diagnostic settings** → **+ Add diagnostic setting**
2. **Configure:**
   - **Name**: `storage-to-logs`
   - **Logs**: Check **StorageRead**, **StorageWrite**, **StorageDelete**
   - **Metrics**: Check **Transaction**
   - **Destination**: Check **Send to Log Analytics workspace** → select `prod-logs`
3. Click **Save**

**What you just enabled**: Every read, write, and delete operation on this storage account is now being recorded. During an incident, you can answer "what was accessed and when?" and "who deleted that file?"

### Step 4: Create a Network Security Group (for security log practice)
1. Search **Network security groups** → **+ Create**
2. **Basics:**
   - **Resource Group**: `monitoring-lab-rg`
   - **Name**: `app-nsg`
   - **Region**: `East US`
3. Click **Review + create** → **Create**

### Step 5: Enable Diagnostic Logging on the NSG
NSG flow logs tell you which IPs are hitting your network and whether traffic is being allowed or denied. This is critical for security auditing.

1. Open the NSG → **Diagnostic settings** → **+ Add diagnostic setting**
2. **Configure:**
   - **Name**: `nsg-to-logs`
   - **Logs**: Check **NetworkSecurityGroupEvent**, **NetworkSecurityGroupRuleCounter**
   - **Destination**: **Send to Log Analytics workspace** → `prod-logs`
3. Click **Save**

**Wait 5–10 minutes** before moving on — it takes a few minutes for logs to start flowing to the workspace.

---

## Part 3: Query Logs to Answer Real Questions

### Step 6: Open Log Analytics and Write Your First Query
These are the kinds of queries you'd run during an incident or a routine check.

1. Go to **Log Analytics workspaces** → `prod-logs` → Left sidebar → **Logs**
2. Close any sample query panels that appear

**Query 1: What has happened in this resource group in the last hour?**
Run this after a deployment or when something seems off — it gives you a timeline of activity.
```kusto
AzureActivity
| where TimeGenerated > ago(1h)
| where ResourceGroup == "monitoring-lab-rg"
| project TimeGenerated, Caller, OperationName, ActivityStatus
| sort by TimeGenerated desc
```

**Query 2: Did anyone delete anything recently?**
This is often the first query you run when someone says "something's missing."
```kusto
AzureActivity
| where TimeGenerated > ago(24h)
| where OperationName has "delete" or OperationName has "Delete"
| project TimeGenerated, Caller, OperationName, ResourceGroup, Resource
| sort by TimeGenerated desc
```

**Query 3: What storage operations happened, and from where?**
Useful if you suspect unauthorized access or need to trace a data access issue.
```kusto
StorageBlobLogs
| where TimeGenerated > ago(1h)
| project TimeGenerated, CallerIpAddress, OperationName, BlobPath, StatusCode
| sort by TimeGenerated desc
```
*Note: This may return no results if no blob operations have happened yet — that's fine. The query is correct.*

**Query 4: Summarize activity by operation to see what's most common**
```kusto
AzureActivity
| where TimeGenerated > ago(24h)
| where ResourceGroup == "monitoring-lab-rg"
| summarize Count = count() by OperationName
| sort by Count desc
| take 10
```

### Step 7: Quick KQL Reference
You'll use these operators constantly:

| Operator | What it does | Example |
|----------|-------------|---------|
| `where` | Filter rows | `where StatusCode == 403` |
| `project` | Choose columns to display | `project TimeGenerated, Caller` |
| `summarize` | Aggregate (count, sum, avg) | `summarize count() by OperationName` |
| `sort by` | Order results | `sort by TimeGenerated desc` |
| `take` | Limit to N rows | `take 20` |
| `has` | Substring match | `where OperationName has "delete"` |
| `ago()` | Time range | `where TimeGenerated > ago(6h)` |

---

## Part 4: Create Meaningful Alerts

### Step 8: Create an Alert for Unauthorized Access Attempts
A storage account returning 403s (permission denied) is a signal worth knowing about. It could mean a misconfigured app, a credential that got revoked, or an external scan.

1. Search **Monitor** → Left sidebar → **Alerts** → **+ Create** → **Alert rule**
2. **Scope**: Click **Select resource** → find your storage account → select it
3. **Condition:**
   - Click **+ Add condition**
   - Signal: search `Availability` — or scroll to find a relevant metric
   - Actually, let's use a log-based alert. Close this and instead:

**Create a log-based alert:**
1. Go to `prod-logs` → Left sidebar → **Alerts** → **+ New alert rule**
2. **Condition** — paste this query:
   ```kusto
   StorageBlobLogs
   | where StatusCode == 403
   | summarize Count = count() by bin(TimeGenerated, 5m)
   ```
3. Click **Continue**
4. **Alert logic:**
   - **Measurement**: `Table rows`
   - **Operator**: `Greater than`
   - **Threshold**: `5`
   - **Evaluation frequency**: `5 minutes`
5. Click **Next: Actions**

### Step 9: Create an Action Group (Who Gets Notified)
1. Click **+ Create action group**
2. **Basics:**
   - **Action group name**: `on-call-team`
   - **Short name**: `oncall`
3. Click **Next: Notifications**
4. **Add a notification:**
   - **Type**: `Email/SMS message/Push/Voice`
   - **Name**: `Email alert`
   - Enter your email address → **OK**
5. Click **Review + create** → **Create**

6. Back on the alert rule:
   - **Alert rule name**: `Unauthorized Storage Access`
   - **Description**: `More than 5 permission-denied errors on storage in 5 minutes`
   - **Severity**: `2 – Warning`
7. Click **Create alert rule**

**Why this threshold?** One or two 403s might be a misconfigured retry. More than five in five minutes is likely a systematic problem — either an app is broken or something unusual is happening.

### Step 10: Create a "Someone Deleted a Resource" Alert
Accidental deletions happen. This alert fires if any resource in your RG is deleted, so you know immediately.

1. **Monitor** → **Alerts** → **+ Create** → **Alert rule**
2. **Scope**: Select your subscription (so it catches any deletion)
3. **Condition:**
   - Click **+ Add condition**
   - Search for `All Administrative operations` (under Activity Log signals)
   - Select it
4. **Alert logic:**
   - **Operation name**: Filter for `Delete`
   - **Event initiated by**: Leave blank (catches any user)
5. **Actions**: Select `on-call-team`
6. **Details:**
   - **Name**: `Resource Deletion Alert`
   - **Severity**: `1 – Error`
7. Click **Create alert rule**

---

## Part 5: Build a Team Health Dashboard

### Step 11: Create a Monitoring Workbook
A workbook is a shareable, interactive dashboard. This is what your team bookmarks to check the state of the environment at a glance.

1. Search **Monitor** → Left sidebar → **Workbooks** → **+ New**
2. Click **Edit**

**Add a title:**
- Click **+ Add** → **Add text**
- Type: `## Environment Health — Production`
- Click **Done Editing** (for that element)

**Add a metric chart:**
- Click **+ Add** → **Add metric**
- **Resource type**: `Storage accounts`
- **Resource**: Select your storage account
- **Namespace**: `Blob`
- **Metric**: `Transactions`
- **Aggregation**: `Sum`
- **Time range**: `Last 4 hours`
- Click **Apply**

**Add a log query:**
- Click **+ Add** → **Add query**
- Paste:
  ```kusto
  AzureActivity
  | where TimeGenerated > ago(4h)
  | where ResourceGroup == "monitoring-lab-rg"
  | summarize Operations = count() by OperationName
  | sort by Operations desc
  | take 10
  ```
- **Visualization**: `Grid`
- Click **Run Query** → **Done Editing**

### Step 12: Save and Share
1. Click **Done Editing** (top of workbook)
2. Click **Save**:
   - **Name**: `Production Health Dashboard`
   - **Resource group**: `monitoring-lab-rg`
3. Click **Apply**

**Sharing it**: Workbooks can be pinned to an Azure Dashboard (the portal homepage) or shared as a link. In practice, you'd pin it to a team dashboard that's bookmarked in the browser.

---

## Part 6: Review What You've Set Up

### Step 13: Verify the Monitoring Stack
1. Go to **Monitor** → **Overview**
   - **Alerts**: Should show 2 alert rules
   - **Workbooks**: 1 dashboard

2. Go to `prod-logs` → **Overview**
   - Should show data sources flowing in

3. Go to **Monitor** → **Alert rules**
   - Both alerts should be listed as `Enabled`

**A quick mental model of what you've built:**

```
Resources (storage, NSG)
    │
    ▼ diagnostic settings
Log Analytics Workspace (prod-logs)
    │
    ├─ Alerts (fire on conditions → email on-call-team)
    │
    └─ Workbooks (visual dashboard for team)
```

---

## Part 7: Cleanup

1. Go to **Resource groups** → `monitoring-lab-rg` → **Delete resource group**
2. Confirm with the resource group name → **Delete**

This deletes the storage account, NSG, workspace, and workbook. Note that alert rules scoped to the subscription level (the resource deletion alert) may need to be deleted separately under **Monitor** → **Alert rules**.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|--------------------------|
| **Enabling diagnostic settings** | Default logging is minimal; you have to opt into useful log categories |
| **KQL queries** | Your first tool during an incident — "what happened and who did it?" |
| **Log-based alert** | Detects patterns that metric thresholds miss (e.g., error codes) |
| **Action groups** | Determines who gets paged and how — critical to get right before go-live |
| **Activity log alert** | Catches destructive operations (deletions) in near-real-time |
| **Workbook dashboard** | Gives the team a shared view without having to write queries every time |

---

## Common Mistakes to Avoid
- **Not enabling diagnostics before go-live**: When something breaks, you'll have no logs to query
- **Setting alert thresholds too low**: An alert that fires every few minutes gets ignored — tune thresholds to be meaningful
- **No action group on an alert**: An alert rule without an action group notifies no one
- **Using one Log Analytics workspace for everything**: Mixing dev and prod logs makes queries noisy and cost tracking harder
- **Forgetting log retention settings**: Default is 30 days; if compliance requires 90 days, you have to configure this explicitly

---

## Next Steps
- Create an availability test in Application Insights to ping your app's endpoint every 5 minutes
- Set up a CPU > 80% alert on a VM (connect to Lab 2)
- Write a KQL query that detects failed login attempts across your environment
- Export logs to a storage account for long-term retention or compliance archiving
- Create an Azure Monitor alert that triggers an automation runbook to auto-remediate an issue
