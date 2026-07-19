# Lab 7: Threat Hunting with Sentinel & Defender Advanced Hunting

Check box if done: [ ]

## Overview
Every lab so far has been reactive by design — a rule fires, an incident lands, you respond. Threat hunting inverts that: you go looking for attacker behavior *before* any rule catches it, working from a hypothesis instead of an alert. This lab covers Sentinel's built-in hunting workflow (hunting queries, bookmarks, livestream), Defender Advanced Hunting for cross-workload queries against Defender XDR's raw telemetry tables, and — the step that closes the loop — turning a successful hunt into a new scheduled analytics rule so the next occurrence doesn't require a human going looking for it again.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$1 (query execution against existing workspace data; negligible incremental cost)

---

## Scenario
Your detection rule from Lab 3 catches brute-force patterns, but your team lead is worried about what it *doesn't* catch — a patient attacker making a handful of low-and-slow authentication attempts per day, staying under any reasonable threshold. She wants you to proactively hunt for that pattern instead of waiting for a rule to catch it, and if you find something real, turn it into a permanent detection so this doesn't stay a manual exercise.

---

## Objectives
- Run and adapt Sentinel's built-in hunting queries
- Write a custom hunting query from a stated hypothesis
- Bookmark findings and attach them to an investigation
- Use livestream to watch a query fire against real-time data
- Query Defender Advanced Hunting tables for endpoint/identity telemetry a Sentinel-only hunt misses
- Convert a successful hunting query into a scheduled analytics rule

---

## Part 1: Run Built-In Hunting Queries

### Step 1: Browse the Hunting Gallery
1. **Microsoft Sentinel** → **Hunting** → **Queries** tab
2. Filter by **Data source**: `SigninLogs` — note the range of pre-built hypotheses Microsoft ships: impossible travel, sign-ins from anonymized IPs, unfamiliar sign-in properties
3. Each query lists its own **MITRE ATT&CK tactic** — the same tagging discipline from Lab 3, now applied to hunts instead of rules

### Step 2: Run One and Read the Results Bar
1. Select a built-in query (e.g., "Failed logon attempts to Azure Portal from multiple IPs")
2. **Run Query** → note the **Results** count displayed inline in the gallery, letting you scan dozens of hypotheses for hits without opening each one individually
3. Open the full results for any query returning >0 rows

**Why start here instead of writing from scratch**: the built-in gallery encodes hypotheses from Microsoft's own threat research — reviewing it first tells you what's already been thought of, so your own hunting time goes toward gaps specific to your environment, not reinventing a known pattern.

---

## Part 2: Write a Custom Hunt from a Hypothesis

### Step 3: State the Hypothesis
Following the scenario: *"An attacker patient enough to stay under the Lab 3 rule's 5-failure/20-minute threshold might still show a detectable pattern across a longer window — repeated low-volume failures against the same account over several days, from a small number of IPs, is unusual for normal user behavior (people who forget passwords usually fix it same-day)."*

### Step 4: Build the Query
```kusto
let LookbackDays = 14d;
let LowVolumeThreshold = 2;   // stays under Lab 3's 5-per-20min threshold
let MinDaysActive = 3;         // pattern persists across multiple days, not one bad afternoon
SigninLogs
| where TimeGenerated > ago(LookbackDays)
| where ResultType != "0"
| summarize
    FailuresPerDay = count(),
    ActiveDays = dcount(bin(TimeGenerated, 1d)),
    DistinctIPs = dcount(IPAddress),
    IPList = make_set(IPAddress, 5)
    by UserPrincipalName, bin(TimeGenerated, 1d)
| summarize
    TotalFailures = sum(FailuresPerDay),
    DaysWithFailures = dcount(bin_TimeGenerated),
    AvgFailuresPerActiveDay = avg(FailuresPerDay)
    by UserPrincipalName
| where DaysWithFailures >= MinDaysActive and AvgFailuresPerActiveDay <= LowVolumeThreshold
| sort by DaysWithFailures desc
```
This is deliberately shaped to find what Lab 3's rule structurally cannot — the threshold and time window are inverted (low volume, long window) rather than high volume, short window.

### Step 5: Save as a Hunting Query
1. **Hunting** → **+ New Query**
2. Paste the query, set **Name**: `Low-and-slow credential attempts`, **Tactics**: `CredentialAccess`, **Technique**: `T1110.003` (Password Spraying variant)
3. **Save** — this makes it reappear in the gallery for the next analyst who runs a hunting pass, the same institutional-knowledge benefit as Lab 2's saved function

**Validation checkpoint**: run the saved query against your workspace and manually review any accounts returned — cross-reference with `GetUserActivity()` from Lab 2 to see if the pattern correlates with anything else notable for that account before concluding it's worth escalating.

---

## Part 3: Bookmark and Investigate Findings

### Step 6: Bookmark a Result Worth Tracking
1. Run the Step 4 query, select the row(s) for any account you want to track further
2. **Bookmark selected rows** → **Bookmark name**: `low-and-slow-candidate-<upn>` → add a note explaining why: `"14-day pattern of 1-2 failures/day across 5 active days — below rule threshold, escalating for review."`

### Step 7: Attach the Bookmark to an Incident or Create One
1. **Hunting** → **Bookmarks** tab → select your bookmark → **Incident actions** → **Create new incident**
2. This routes your hunting finding into the same triage/investigation workflow from [Lab 5](lab-5-incident-response-investigation.md) — a hunt that finds something real doesn't stay a side project, it becomes a worked incident with the same rigor

---

## Part 4: Livestream a Query

### Step 8: Set Up a Livestream Session
Livestream runs a query on a short interval against real-time data and surfaces new matches as they arrive — useful during an active hunt session or while monitoring a specific hypothesis for a limited window rather than committing to a permanent rule yet.
1. **Hunting** → **Livestream** → **+ New livestream query**
2. Paste a narrower, real-time-appropriate version:
```kusto
SigninLogs
| where TimeGenerated > ago(5m)
| where ResultType != "0"
| where AppDisplayName == "Azure Portal"
```
3. **Name**: `Live Azure Portal sign-in failures` → **Create**
4. Leave it running while you continue other work — it notifies as new matching events arrive, without the overhead of a formal scheduled rule with incident creation

**Why livestream instead of just committing straight to a rule**: it lets you validate a hypothesis is producing signal worth acting on in real time, before you invest in the full rule-engineering process from Lab 3 (entity mapping, MITRE tagging, tuning) for something that might turn out to be noise.

---

## Part 5: Defender Advanced Hunting — Beyond Sentinel's Own Tables

### Step 9: Understand What Advanced Hunting Adds
Sentinel's hunting queries run against whatever tables you've connected (Part 1–4, all against `SigninLogs`). Defender Advanced Hunting, in `security.microsoft.com`, queries Defender XDR's raw telemetry directly — process execution, device logon events, email — much of which is far higher-volume and higher-fidelity than what typically gets piped into a Log Analytics workspace by default.

### Step 10: Run a Cross-Table Hunt in Advanced Hunting
1. **security.microsoft.com** → **Hunting** → **Advanced hunting**
2. Example query correlating identity and endpoint telemetry from [Lab 4](lab-4-defender-xdr-endpoint-identity.md):
```kusto
DeviceLogonEvents
| where Timestamp > ago(7d)
| where LogonType == "Network"
| join kind=inner (
    IdentityLogonEvents
    | where Timestamp > ago(7d)
    | where ActionType == "LogonFailed"
    | summarize FailedLogons = count() by AccountUpn
) on $left.AccountName == tostring(split($right.AccountUpn, "@")[0])
| where FailedLogons > 3
| project Timestamp, DeviceName, AccountName, FailedLogons
```
This correlates network logon events on endpoints with failed identity logons observed by Defender for Identity's domain-controller sensor — a cross-workload join that's only possible because both signal types exist in Advanced Hunting's shared schema, something Sentinel-only hunting (querying just `SigninLogs`) structurally can't see without ingesting the raw Defender tables too.

### Step 11: Compare the Two Hunting Surfaces
| | Sentinel Hunting | Defender Advanced Hunting |
|---|---|---|
| **Query language** | KQL | KQL (same syntax, different schema) |
| **Data scope** | Whatever's connected to the workspace | Defender XDR's own raw telemetry (endpoint, identity, email, cloud apps) |
| **Best for** | Cross-source correlation once everything's centralized in one workspace | Deep, high-volume endpoint/identity telemetry that isn't always piped into Sentinel by default |
| **Cross-pollination** | Can ingest Defender XDR raw tables (Lab 4, Step 11) to query both in one place | Can be connected back into Sentinel the same way |

Neither replaces the other — a mature SOC hunts in both, and increasingly centralizes into Sentinel once Defender XDR's raw tables are ingested (Lab 4's optional raw-table connector setting).

---

## Part 6: Convert a Hunt into a Detection

### Step 12: Promote the Low-and-Slow Query to a Scheduled Rule
Once a hunting query has proven itself against real data (not just a hypothesis), it belongs in the same detection-engineering pipeline as Lab 3:
```bash
az sentinel alert-rule create \
  --resource-group sc200-lab1-rg \
  --workspace-name sc200-sentinel-law \
  --rule-id "low-and-slow-credential-detection" \
  --kind Scheduled \
  --display-name "Low-and-Slow Credential Attempts (Multi-Day)" \
  --description "Detects accounts with a sustained multi-day pattern of low-volume sign-in failures that stays under single-window brute-force thresholds — promoted from a Lab 7 hunting query." \
  --severity Low \
  --enabled true \
  --query "let LookbackDays = 14d; let LowVolumeThreshold = 2; let MinDaysActive = 3; SigninLogs | where TimeGenerated > ago(LookbackDays) | where ResultType != \"0\" | summarize FailuresPerDay = count() by UserPrincipalName, bin(TimeGenerated, 1d) | summarize TotalFailures = sum(FailuresPerDay), DaysWithFailures = dcount(bin_TimeGenerated), AvgFailuresPerActiveDay = avg(FailuresPerDay) by UserPrincipalName | where DaysWithFailures >= MinDaysActive and AvgFailuresPerActiveDay <= LowVolumeThreshold" \
  --query-frequency P1D \
  --query-period P14D \
  --trigger-operator GreaterThan \
  --trigger-threshold 0 \
  --tactics CredentialAccess \
  --techniques T1110
```
Note `--query-frequency P1D` (daily) and `--query-period P14D` (14-day lookback) — a long-window hunting pattern like this doesn't need to run every 10 minutes the way Lab 3's rule does; running it daily against a rolling two-week window is both sufficient and cheaper.

**This closes the loop the scenario asked for**: what started as a manual hunt because the Lab 3 rule structurally couldn't catch this pattern is now itself a standing detection, following the exact entity-mapping and severity discipline from Lab 3 (add entity mapping for `UserPrincipalName` the same way before considering this production-ready).

---

## Part 7: Cleanup

```bash
# Remove the promoted rule if this was exploratory rather than something you want running long-term
az sentinel alert-rule delete \
  --resource-group sc200-lab1-rg \
  --workspace-name sc200-sentinel-law \
  --rule-id "low-and-slow-credential-detection" \
  --yes
```
1. **Hunting** → **Livestream** → stop and delete the `Live Azure Portal sign-in failures` session from Step 8 — livestream sessions keep running (and keep querying, incurring small ongoing cost) until explicitly stopped
2. Delete or keep the `low-and-slow-candidate-<upn>` bookmark and its associated test incident from Step 7, closing the incident with an appropriate classification per [Lab 5](lab-5-incident-response-investigation.md)'s workflow if you haven't already

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Running and adapting built-in hunting queries** | Starting from known hypotheses instead of reinventing well-understood attack patterns |
| **Writing a custom hunt from a stated hypothesis** | The actual skill being tested — hunting starts with "what might we be missing," not with a query |
| **Bookmarking and routing findings into incidents** | A hunt that finds something real gets the same investigative rigor as an alert-triggered incident |
| **Livestream for real-time hypothesis validation** | Cheap way to confirm a pattern is worth formalizing before investing in full rule engineering |
| **Defender Advanced Hunting cross-table correlation** | Reaching telemetry (raw endpoint/identity events) that isn't always centralized in Sentinel by default |
| **Promoting a proven hunt into a scheduled rule** | Closes the loop — a one-time manual find becomes permanent, automatic coverage |

---

## Common Mistakes to Avoid
- **Hunting with no stated hypothesis**: "let me poke around the logs" produces noise; "attackers might stay under our known threshold by doing X" produces a testable, falsifiable query
- **Treating a hunting finding as done once bookmarked**: a bookmark with no follow-up incident or documented conclusion is a lead that goes cold — route real findings into the Lab 5 investigation workflow
- **Leaving livestream sessions running indefinitely**: they're meant for active hunting windows, not permanent detection — stop them or promote them to a real scheduled rule (Step 12)
- **Promoting a hunting query to a rule without entity mapping or tuning**: a promoted rule still needs the full Lab 3 treatment (entity mapping, alert grouping, threshold validation against historical data) before it's production-ready, not just a copy-paste of the hunting query
- **Only hunting in one surface**: Sentinel-only hunting misses raw endpoint/identity telemetry that Defender Advanced Hunting sees natively, and vice versa for cross-source correlation — a mature practice uses both

---

## Next Steps
- Continue to [Lab 8: Defender for Cloud Apps & Cross-Workload Correlation (Capstone)](lab-8-defender-cloud-apps-capstone.md) to extend hunting and correlation across a third data source — SaaS application activity
- Build a recurring hunting cadence: schedule a weekly pass through the built-in hunting gallery (Part 1) even without a specific hypothesis, since Microsoft adds new queries as new attack techniques emerge
- For how a security architect decides which detections belong as standing rules vs. periodic hunts vs. one-off investigations at an org-wide level, see [SC-100](../SC-100/README.md)
