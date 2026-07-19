# Lab 2: KQL Fundamentals for Security Analysts

Check box if done: [ ]

## Overview
Every other lab in this track assumes you can read and write KQL fluently — it's the actual daily-use language of a SOC analyst, the same way SQL is for a data analyst. This lab builds that fluency deliberately: filtering and projecting, aggregating with `summarize`, correlating tables with `join`, parsing unstructured fields analysts run into constantly (user agents, JSON blobs in `AdditionalDetails`), and saving reusable queries as functions instead of re-typing the same fifteen lines every shift. Nothing here is exotic — it's the small set of patterns that covers the overwhelming majority of real analyst queries.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (query execution against an existing workspace has no meaningful incremental cost within Sentinel's ingestion-based billing)

---

## Scenario
Your team lead asks you to build a small library of saved queries new analysts can reuse for daily triage instead of relearning KQL from scratch every time: a sign-in failure summary, a privileged-role-change report, and a reusable function for "show me everything this user did today." You'll build all three, and along the way pick up the query patterns that show up in literally every later lab.

---

## Objectives
- Filter, project, and sort results with `where`, `project`, `sort`, `distinct`
- Aggregate data with `summarize`, `count()`, `bin()`, and visualize with `render`
- Correlate two tables with `join`
- Parse unstructured string and JSON fields with `parse`, `extract`, and `parse_json`
- Package a query as a reusable Sentinel function

---

## Setup (if starting fresh)
This lab queries the workspace and connectors from [Lab 1](lab-1-sentinel-workspace-data-connectors.md). If you're picking this lab up independently:
```bash
az group create --name sc200-lab2-rg --location eastus2
az monitor log-analytics workspace create \
  --resource-group sc200-lab2-rg --workspace-name sc200-lab2-law --location eastus2
```
Then follow Lab 1, Steps 4–7 to connect Azure Activity and Entra ID sign-in logs before continuing — these queries need real data to be useful.

---

## Part 1: Filtering and Projection

### Step 1: The Building Blocks
```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"
| project TimeGenerated, UserPrincipalName, AppDisplayName, IPAddress, ResultType, ResultDescription
| sort by TimeGenerated desc
```
`where` filters rows top to bottom — put the most selective filter first for performance (`TimeGenerated` first is a habit worth building; Sentinel's engine is time-partitioned, so time-filtering early narrows the scan dramatically). `project` picks and reorders columns — always prefer it over returning every column, both for readability and because wide tables like `SigninLogs` have 50+ columns you don't need.

### Step 2: Distinct Values
```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"
| distinct ResultType, ResultDescription
```
Before building a detection around failure codes, always check what values actually appear in your tenant — `ResultType == "50126"` (invalid username or password) behaves very differently from `ResultType == "50053"` (account locked), and you don't want to guess.

### Step 3: Filtering with `in` and String Operators
```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType in ("50126", "50053", "50074")
| where AppDisplayName has_any ("Office 365", "Azure Portal")
| project TimeGenerated, UserPrincipalName, AppDisplayName, ResultType, IPAddress
```
`has_any` is preferred over `contains` for whole-word matching — it's indexed and faster, and avoids partial-word false positives (`contains "365"` would also match an app named "365DaysApp").

---

## Part 2: Aggregation with `summarize`

### Step 4: Count Failures by User
```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"
| summarize FailureCount = count() by UserPrincipalName
| where FailureCount > 5
| sort by FailureCount desc
```
This is the skeleton of nearly every "who's getting hammered with failed logins" triage query — you'll extend it directly in Lab 3 when this becomes a scheduled analytics rule.

### Step 5: Time-Bucketed Aggregation
```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"
| summarize FailureCount = count() by bin(TimeGenerated, 1h), UserPrincipalName
| sort by TimeGenerated asc
```
`bin()` groups timestamps into fixed-size buckets — essential for spotting bursts (ten failures inside one hour is a very different signal than ten failures spread across a day).

### Step 6: Visualize It
```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| summarize SignInCount = count() by bin(TimeGenerated, 1h), ResultType
| render timechart
```
`render` only affects the portal/notebook display, not the underlying data — useful when eyeballing a pattern before committing to a threshold in a detection rule.

### Step 7: Multiple Aggregations at Once
```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| summarize
    TotalAttempts = count(),
    FailedAttempts = countif(ResultType != "0"),
    DistinctIPs = dcount(IPAddress),
    FirstSeen = min(TimeGenerated),
    LastSeen = max(TimeGenerated)
    by UserPrincipalName
| extend FailureRate = round(100.0 * FailedAttempts / TotalAttempts, 1)
| where FailureRate > 50 and TotalAttempts > 5
| sort by FailureRate desc
```
`countif()` and `dcount()` inside a single `summarize` avoid needing multiple passes over the data — `dcount(IPAddress)` here is a classic signal: a single user authenticating from many distinct IPs in a short window is exactly the shape of a password-spray or impossible-travel scenario.

---

## Part 3: Correlating Tables with `join`

### Step 8: Join Sign-Ins to Audit Log Role Changes
A common investigative question: did this user do anything administratively sensitive around the same time they had suspicious sign-in activity?
```kusto
let SuspiciousUsers = SigninLogs
    | where TimeGenerated > ago(24h)
    | summarize FailedAttempts = countif(ResultType != "0") by UserPrincipalName
    | where FailedAttempts > 5
    | project UserPrincipalName;
AuditLogs
| where TimeGenerated > ago(24h)
| where OperationName has "role"
| extend InitiatedBy = tostring(InitiatedBy.user.userPrincipalName)
| join kind=inner (SuspiciousUsers) on $left.InitiatedBy == $right.UserPrincipalName
| project TimeGenerated, OperationName, InitiatedBy, Result
```
`let` defines a reusable subquery — building the "suspicious users" list once and referencing it keeps the join readable. `kind=inner` returns only rows with a match in both tables; `kind=leftouter` would keep every `AuditLogs` row and fill unmatched fields with null, useful when you want to see everything vs. only the intersection.

### Step 9: Understand the Join Cost Tradeoff
```kusto
AzureActivity
| where TimeGenerated > ago(24h)
| where OperationNameValue has "DELETE"
| join kind=inner (
    SigninLogs
    | where TimeGenerated > ago(24h)
    | project UserPrincipalName, IPAddress
) on $left.Caller == $right.UserPrincipalName
| project TimeGenerated, OperationNameValue, Caller, IPAddress
```
**Why this matters for performance**: always filter and project *before* the join, not after — reducing each side to the minimum rows/columns needed before joining is the difference between a query that runs in seconds and one that times out on a busy workspace.

---

## Part 4: Parsing Unstructured Data

### Step 10: Parsing JSON Fields
Several Sentinel tables store nested detail in a dynamic (JSON) column. `AuditLogs.TargetResources` is a common one:
```kusto
AuditLogs
| where TimeGenerated > ago(24h)
| where OperationName == "Add member to role"
| extend TargetUser = tostring(TargetResources[0].userPrincipalName)
| extend RoleName = tostring(parse_json(tostring(AdditionalDetails))[0].value)
| project TimeGenerated, InitiatedBy = tostring(InitiatedBy.user.userPrincipalName), TargetUser, RoleName
```
`dynamic` columns need `tostring()`/`toint()` etc. to cast the extracted value into something you can filter or display cleanly — a raw dynamic value renders as JSON and won't compare correctly against a string literal.

### Step 11: Parsing Freeform Strings
```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| project TimeGenerated, UserPrincipalName, UserAgent
| parse UserAgent with * "(" OS ";" * ")" *
| where isnotempty(OS)
| summarize count() by OS
| sort by count_ desc
```
`parse` is a lightweight pattern-matcher — faster to write than a full regex for well-structured strings like user-agent headers. For anything less predictable, `extract()` with a regex is the fallback:
```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| extend BrowserVersion = extract(@"Chrome/([\d.]+)", 1, UserAgent)
| where isnotempty(BrowserVersion)
| project TimeGenerated, UserPrincipalName, BrowserVersion
```

**Validation checkpoint**: run Step 10 and Step 11 and confirm the parsed columns (`RoleName`, `OS`, `BrowserVersion`) contain readable values, not `null` or raw JSON — a parse pattern that doesn't match the actual data returns empty silently, which is easy to miss if you don't check.

---

## Part 5: Save It as a Reusable Function

### Step 12: Build the "What Did This User Do Today" Function
```kusto
let GetUserActivity = (upn: string, lookback: timespan) {
    union
        (SigninLogs
            | where TimeGenerated > ago(lookback)
            | where UserPrincipalName == upn
            | project TimeGenerated, Source = "SignIn", Detail = strcat(AppDisplayName, " from ", IPAddress), ResultType),
        (AuditLogs
            | where TimeGenerated > ago(lookback)
            | where tostring(InitiatedBy.user.userPrincipalName) == upn
            | project TimeGenerated, Source = "AuditLog", Detail = OperationName, ResultType = Result)
    | sort by TimeGenerated desc
};
GetUserActivity("<user-upn>", 24h)
```
This is exactly the query shape an investigation graph pivot (Lab 5) runs behind the scenes — a single timeline across multiple tables for one entity.

### Step 13: Save as a Sentinel Function
1. **Microsoft Sentinel** → **Logs** → paste the query body (without the final call line) → **Save** → **Save as function**
2. **Function name**: `GetUserActivity`
3. **Legacy category**: `Custom Queries`
4. Once saved, any analyst on the team can call `GetUserActivity("<user-upn>", 24h)` from any query in the workspace — no copy-pasting the underlying logic

---

## Part 6: Cleanup

```bash
# Only needed if you created the standalone Setup resources at the top of this lab
az group delete --name sc200-lab2-rg --yes --no-wait
```

If you're continuing to Lab 3, no cleanup is needed — the `GetUserActivity` function and the workspace from Lab 1 carry forward.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **`where`/`project`/`sort` fundamentals** | The 80% of daily triage queries that never need anything fancier |
| **`summarize` with `count()`, `countif()`, `dcount()`, `bin()`** | Turning raw event streams into the aggregated signals detections and dashboards are built from |
| **`join` across tables with pre-filtering** | Correlating identity and activity data — the core move behind almost every investigation |
| **Parsing `dynamic`/JSON and freeform string fields** | Real telemetry is rarely flat — extracting structured signal from nested or unstructured fields is a constant analyst task |
| **Saving a query as a reusable function** | Building institutional knowledge into the workspace instead of re-deriving the same query every shift |

---

## Common Mistakes to Avoid
- **Filtering by time last instead of first**: `TimeGenerated` should almost always be the first `where` clause — Sentinel's storage is time-partitioned, and filtering late scans far more data than necessary
- **Joining before filtering/projecting**: reduce both sides of a `join` to only the rows and columns you need first, or the query gets slow and expensive on any real-sized workspace
- **Forgetting `dynamic` columns need casting**: comparing a raw `dynamic` value to a string literal silently fails to match — always `tostring()` or `toint()` first
- **Assuming a `parse` pattern matched**: `parse` fails silently and leaves the target column empty rather than erroring — always spot-check parsed columns against a few raw rows
- **Rebuilding the same query from scratch every time**: if you've written the same `summarize` or `join` logic twice, it belongs in a saved function

---

## Next Steps
- Continue to [Lab 3: Sentinel Analytics Rules & Detection Engineering](lab-3-analytics-rules-detection-engineering.md) — the failure-count query from Step 4 becomes the basis for a real scheduled detection
- Practice against your own tenant's `SecurityEvent` table if you have a Windows VM sending Log Analytics agent data — the same `summarize`/`join` patterns apply to process and logon event correlation
- Bookmark the [KQL Quick Reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference) — no analyst memorizes every operator, they memorize where to look it up fast
