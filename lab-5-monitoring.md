# Lab 5: Monitoring & Logging (Observability)

## Overview
This lab covers Azure Monitor, Log Analytics, Application Insights, diagnostic settings, alerts, and dashboards. You'll build a complete observability stack for monitoring resources and troubleshooting issues.

**Estimated time**: 60–75 minutes  
**Cost**: ~$2–$5 (Log Analytics ingestion; tear down at the end)

---

## Objectives
- Set up Log Analytics workspaces
- Enable diagnostic logging on Azure resources
- Query logs with KQL (Kusto Query Language)
- Create alerts based on metrics and logs
- Build custom dashboards
- Understand Application Insights for app monitoring

---

## Part 1: Create Log Analytics Workspace

### Step 1: Create Log Analytics Workspace
1. Go to **Azure Portal** → Search **Log Analytics workspaces** → **+ Create**
2. **Basics tab:**
   - **Resource Group**: Create new → `az104-lab5-rg`
   - **Name**: `lab5-workspace`
   - **Region**: `East US`
   - **Pricing tier**: `Pay-as-you-go`
     - *Why not Reservation?* For learning, PAYG is cheaper.
3. Click **Review + create** → **Create**

**Wait ~2 minutes** for workspace to deploy.

---

## Part 2: Create Resources to Monitor

### Step 2: Create a Storage Account (for diagnostics)
1. Search **Storage accounts** → **+ Create**
2. **Basics:**
   - **Resource Group**: `az104-lab5-rg`
   - **Storage account name**: `labmon<randomnumber>`
   - **Region**: `East US`
   - **Redundancy**: `Locally redundant storage (LRS)` (cheaper for lab)
3. Click **Review + create** → **Create**

### Step 3: Create a Virtual Network (for diagnostics)
1. Search **Virtual networks** → **+ Create**
2. **Basics:**
   - **Resource Group**: `az104-lab5-rg`
   - **Name**: `lab5-vnet`
   - **Region**: `East US`
3. **IP Addresses:**
   - Leave defaults (10.0.0.0/16, one subnet)
4. Click **Review + create** → **Create**

### Step 4: Create a Network Security Group (for diagnostics)
1. Search **Network security groups** → **+ Create**
2. **Basics:**
   - **Resource Group**: `az104-lab5-rg`
   - **Name**: `lab5-nsg`
   - **Region**: `East US`
3. Click **Review + create** → **Create**

**Why these resources?** You'll enable diagnostics on them and monitor the logs.

---

## Part 3: Enable Diagnostic Logging

### Step 5: Enable Diagnostics on Storage Account
1. Open the storage account → Left sidebar → **Diagnostic settings** (under Monitoring)
2. Click **+ Add diagnostic setting**
3. **Diagnostic setting configuration:**
   - **Diagnostic setting name**: `storage-diagnostics`
   - **Logs:**
     - Check **StorageRead**
     - Check **StorageWrite**
     - Check **StorageDelete**
   - **Metrics:**
     - Check **Transaction**
   - **Destination details:**
     - Check **Send to Log Analytics workspace**
     - **Subscription**: Your subscription
     - **Log Analytics workspace**: `lab5-workspace`
4. Click **Save**

**What this does**: All read/write/delete operations on this storage account are sent to Log Analytics.

### Step 6: Enable Diagnostics on Virtual Network
1. Open the VNet → Left sidebar → **Diagnostic settings**
2. Click **+ Add diagnostic setting**
3. **Diagnostic setting configuration:**
   - **Diagnostic setting name**: `vnet-diagnostics`
   - **Logs:**
     - Check **VMProtectionAlerts**
   - **Metrics:**
     - Check **AllMetrics**
   - **Destination details:**
     - Check **Send to Log Analytics workspace**
     - Select `lab5-workspace`
4. Click **Save**

### Step 7: Enable Diagnostics on NSG
1. Open the NSG → Left sidebar → **Diagnostic settings**
2. Click **+ Add diagnostic setting**
3. **Diagnostic setting configuration:**
   - **Diagnostic setting name**: `nsg-diagnostics`
   - **Logs:**
     - Check **NetworkSecurityGroupEvent**
     - Check **NetworkSecurityGroupRuleCounter**
   - **Metrics:** (NSG doesn't have metrics; skip)
   - **Destination details:**
     - Check **Send to Log Analytics workspace**
     - Select `lab5-workspace`
4. Click **Save**

**What you've done**: Enabled diagnostics on three resources. Data will start flowing to Log Analytics (though it may take 5–10 minutes for data to appear).

---

## Part 4: Query Logs with KQL

### Step 8: Write Your First KQL Query
1. Go to **Log Analytics workspace** (`lab5-workspace`) → **Logs** (left sidebar)
2. Close the **Queries** panel on the right (or minimize it)
3. You'll see a blank query editor

4. **Write a simple query**:
   ```kusto
   AzureActivity
   | where TimeGenerated > ago(24h)
   | where ResourceGroup == "az104-lab5-rg"
   | summarize Count = count() by OperationName
   | sort by Count desc
   ```

5. Click **Run** (or Shift+Enter)

**What this does**: Shows all Azure activities in your resource group in the last 24 hours, grouped by operation type.

### Step 9: Query Storage Diagnostics
1. Clear the previous query and write:
   ```kusto
   StorageBlobLogs
   | where TimeGenerated > ago(1h)
   | where OperationName == "PutBlob"
   | project TimeGenerated, CallerIpAddress, OperationName, BlobPath
   ```

2. Click **Run**

**What this does**: Shows all blob uploads in the last hour (if any data exists yet).

**Note**: If no results, that's expected—you haven't done any blob operations. The query is correct; there's just no data yet.

### Step 10: Query NSG Logs
1. Clear and write:
   ```kusto
   AzureNetworkSecurityGroupEvent
   | where TimeGenerated > ago(24h)
   | where ResourceGroup == "az104-lab5-rg"
   | summarize DenyCount = countif(Action == "Deny") by SourceIP
   ```

2. Click **Run**

**What this does**: Shows which IPs were denied access by your NSG.

**Key KQL concepts** (for the exam):
- `|` = pipe (send results to next operation)
- `where` = filter rows
- `summarize` = aggregate (count, sum, avg, etc.)
- `project` = select columns
- `sort` = order results

---

## Part 5: Create Alerts

### Step 11: Create a Metric Alert
1. Search **Monitor** → Left sidebar → **Alerts** → **+ Create** → **Alert rule**
2. **Scope:**
   - **Resource**: Click **Select resource** → Search storage account → select it
3. **Condition:**
   - Click **+ Add condition**
   - **Signal name**: Search `transactions` → Select `Transactions`
   - **Operator**: `Greater than`
   - **Aggregation type**: `Total`
   - **Threshold value**: `100` (arbitrary; alert if >100 transactions in 5 min)
4. Click **Done**

5. **Actions:**
   - Click **+ Create action group**
   - **Action group name**: `lab5-alerts`
   - **Short name**: `lab5a`
   - **Region**: `East US`
   - Click **Next: Notifications**
   - **Notification type**: `Email/SMS message/Push/Voice`
   - **Name**: `Send email`
   - **Email address**: Enter your email
   - Click **OK** → **Review + create** → **Create**

6. Back on alert rule:
   - **Alert rule name**: `Storage High Transaction Alert`
   - **Severity**: `3 – Informational`
   - Click **Create alert rule**

**What this does**: When your storage account exceeds 100 transactions in 5 minutes, you'll get an email.

### Step 12: Create a Log-Based Alert
1. Go to **Log Analytics workspace** → **Alerts** (left sidebar) → **+ New alert rule**
2. **Condition:**
   - Copy this query into the editor:
     ```kusto
     AzureActivity
     | where ResourceGroup == "az104-lab5-rg"
     | where OperationName == "Delete"
     | summarize Count = count() by bin(TimeGenerated, 5m)
     ```
   - Click **Continue**

3. **Alert logic:**
   - **Measurement**: `Table rows`
   - **Aggregation**: `Count`
   - **Operator**: `Greater than`
   - **Threshold**: `5` (alert if more than 5 delete operations in 5 min)
   - Click **Next**

4. **Actions:**
   - **Action groups**: Select `lab5-alerts` (the one you created earlier)

5. **Details:**
   - **Alert rule name**: `Unusual Delete Activity Alert`
   - **Description**: `Alert if unusual number of delete operations`
   - **Severity**: `2 – Warning`
   - Click **Create alert rule**

**What this does**: Monitors for unusual delete activity (potential data loss or attack).

---

## Part 6: Build a Custom Dashboard

### Step 13: Create a Dashboard
1. Go to **Monitor** → Left sidebar → **Workbooks** → **+ New**
2. Click **Edit**
3. Click **+ Add** → **Add metric**
4. **Metric configuration:**
   - **Resource type**: `Storage accounts`
   - **Resource**: Select your storage account
   - **Namespace**: `Blob`
   - **Metric**: `Transactions`
   - **Aggregation**: `Sum`
5. Click **Apply**

6. Repeat to add another metric:
   - Click **+ Add** → **Add metric**
   - **Metric**: `Ingress`
   - **Aggregation**: `Sum`
   - Click **Apply**

7. Save the workbook:
   - Click **Done Editing** (top)
   - **Save as**: `lab5-dashboard`
   - **Name**: `Lab5 Monitoring Dashboard`
   - Click **Save**

**What you've created**: A custom dashboard showing storage transactions and ingress traffic.

---

## Part 7: Application Insights (Simulated)

### Step 14: Create Application Insights Resource
1. Search **Application Insights** → **+ Create**
2. **Basics:**
   - **Resource Group**: `az104-lab5-rg`
   - **Name**: `lab5-appinsights`
   - **Region**: `East US`
   - **Resource Mode**: `Workspace-based`
   - **Log Analytics Workspace**: `lab5-workspace`
3. Click **Review + create** → **Create**

**Wait ~1 minute**

### Step 15: Understand Application Insights
1. Open **Application Insights** resource → **Overview**
2. You'll see:
   - **Live Metrics**: Real-time app performance
   - **Failures**: Errors and exceptions
   - **Performance**: Response times and dependencies
   - **Availability**: Results of web tests

3. Go to **Logs** (left sidebar)
   - Write a query:
     ```kusto
     customMetrics
     | summarize Count = count() by name
     ```
   - Click **Run** (won't return data because no app is instrumented)

**In real scenarios**: You'd install an SDK in your app (Node.js, Python, .NET, etc.) that sends telemetry to Application Insights.

---

## Part 8: Review Monitoring Best Practices

### Step 16: Understand What You've Built
1. Go to **Monitor** → **Overview**
   - You should see:
     - **Alerts**: 2 rules you created
     - **Workbooks**: 1 dashboard
     - **Log Analytics**: 1 workspace with data flowing in

2. Go to `lab5-workspace` → **Agents** (left sidebar)
   - Shows connected data sources (your storage account, VNet, NSG diagnostics)

3. Go to **Alerts** → **Alert rules**
   - View your 2 alerts

**What's important for the exam**:
- **Metrics**: CPU, memory, network (collected automatically)
- **Logs**: Diagnostic data (must enable)
- **Alerts**: Trigger actions when thresholds are exceeded
- **Workbooks**: Custom visualizations and dashboards
- **Application Insights**: For app-level monitoring (separate from infrastructure)

---

## Part 9: Cleanup

1. Go to **Resource groups** → `az104-lab5-rg` → **Delete resource group**
2. Type resource group name to confirm
3. Click **Delete**

**Wait 3–5 minutes** for all resources to tear down.

---

## Key Concepts to Understand

| Concept | What It Does |
|---------|-------------|
| **Azure Monitor** | Umbrella service for monitoring; collects metrics and logs |
| **Log Analytics** | Stores and queries logs using KQL |
| **Metric** | Numeric data point (CPU %, transactions/sec, etc.) |
| **Log** | Structured event data (who, what, when) |
| **Diagnostic settings** | Enable log/metric collection for resources |
| **Alert rule** | Condition that triggers an action (email, webhook, automation) |
| **Action group** | Recipients and methods for alerts (email, SMS, runbook) |
| **KQL** | Kusto Query Language; SQL-like syntax for querying logs |
| **Workbook** | Custom dashboard combining metrics, logs, and visualizations |
| **Application Insights** | APM tool for app-level monitoring (performance, availability, errors) |

---

## Exam Tips
- **Diagnostic vs Monitoring**: Diagnostics = logs; monitoring = ongoing observation
- **Azure Monitor is free**: You only pay for data ingestion/retention in Log Analytics
- **KQL is testable**: Know basic operators (`where`, `summarize`, `project`)
- **Alert dependencies**: Alerts need action groups; without them, nothing happens
- **Log retention**: Default is 30 days; you can extend (costs more)
- **Custom metrics**: Apps can send custom metrics to Application Insights (requires SDK)

---

## Next Steps
- Set up a web test in Application Insights to monitor endpoint availability
- Create a metric alert for CPU > 80% on a VM (from Lab 2)
- Export logs to storage account for long-term archival
- Create an automation runbook triggered by an alert (e.g., auto-scale VM)
- Query Application Insights for application exceptions and performance trends
- Set up a Log Analytics alert for security events (e.g., failed logins)
