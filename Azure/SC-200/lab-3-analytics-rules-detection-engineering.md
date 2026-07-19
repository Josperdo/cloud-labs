# Lab 3: Sentinel Analytics Rules & Detection Engineering

Check box if done: [ ]

## Overview
AZ-500 Lab 4 built one scheduled analytics rule to prove the detect-to-incident pipeline works end to end. This lab is about becoming a detection engineer, not just enabling one rule: taking the failure-count query from Lab 2 and turning it into a properly engineered detection — with entity mapping so Sentinel understands *who* and *what* is involved, MITRE ATT&CK tagging so the detection is explainable to a framework everyone in the industry shares, alert grouping tuned so one attack doesn't spawn fifty incidents, and a tuning pass to bring the false-positive rate down before it ships. This is the actual skill SC-200 tests under "manage a security operations environment" — not clicking Create, but engineering a rule someone can trust.

**Estimated time**: 75–90 minutes
**Cost**: ~$1–$3 (scheduled query executions and the small amount of test data generated; negligible against Sentinel's free ingestion tier — see Cleanup)

---

## Scenario
Your team lead reviewed the ad-hoc failure-count query you wrote in Lab 2 and told you to productionize it: turn it into a real scheduled rule that fires an alert, creates an incident with the right entities attached so the on-call analyst doesn't have to go dig through raw logs, tags it to MITRE ATT&CK so it shows up correctly in coverage reporting, and is tuned tightly enough that it doesn't create an incident every time someone fat-fingers their password twice.

---

## Objectives
- Design a detection around a specific, explainable risk (credential brute-force / password spray)
- Build a scheduled analytics rule with entity mapping (Account, IP)
- Map the rule to a MITRE ATT&CK tactic and technique
- Configure alert grouping and incident creation settings to control incident volume
- Trigger and validate the detection produces a well-formed incident
- Tune the rule to reduce false positives without losing detection coverage

---

## Setup (if starting fresh)
This lab assumes the Sentinel workspace, Azure Activity connector, and Entra ID sign-in connector from [Lab 1](lab-1-sentinel-workspace-data-connectors.md). If picking this up independently, complete Lab 1 first — analytics rules need real `SigninLogs` data to mean anything.

---

## Part 1: Design the Detection

### Step 1: Define the Risk in One Sentence
Before writing a single line of KQL, an engineered detection starts with a specific, statable risk: *"A single account experiencing repeated authentication failures followed by a success is consistent with a successful brute-force or password-spray attempt (MITRE ATT&CK T1110 — Brute Force)."* Write this down — it's what goes in the rule description, and it's what you'll explain in an interview when asked "why does this rule exist."

### Step 2: Build and Validate the Detection Query
Start in **Microsoft Sentinel** → **Logs** so you can iterate before committing to a rule:
```kusto
let threshold = 5;
let timeframe = 20m;
SigninLogs
| where TimeGenerated > ago(timeframe)
| summarize
    FailedAttempts = countif(ResultType != "0"),
    SuccessAttempts = countif(ResultType == "0"),
    IPAddresses = make_set(IPAddress, 5),
    LastFailureTime = maxif(TimeGenerated, ResultType != "0")
    by UserPrincipalName
| where FailedAttempts >= threshold and SuccessAttempts > 0
| project UserPrincipalName, FailedAttempts, SuccessAttempts, IPAddresses, LastFailureTime
```
This is deliberately more specific than Lab 2's version: it only fires when failures are *followed by* a success — several failed attempts with no eventual success is more likely someone forgetting their password than a successful compromise, while failures-then-success is the higher-confidence brute-force signal worth an analyst's attention.

**Validation checkpoint**: run this in Logs against real or test data before proceeding — confirm it returns zero rows on quiet data (no false-firing on normal activity) so you have a baseline before adding test noise in Step 6.

---

## Part 2: Build the Scheduled Analytics Rule

### Step 3: Create the Rule via CLI
```bash
az sentinel alert-rule create \
  --resource-group sc200-lab1-rg \
  --workspace-name sc200-sentinel-law \
  --rule-id "brute-force-signin-detection" \
  --kind Scheduled \
  --display-name "Repeated Sign-In Failures Followed by Success" \
  --description "Detects a single account with 5+ failed sign-ins followed by a success within 20 minutes — consistent with brute-force or password-spray (MITRE T1110)." \
  --severity Medium \
  --enabled true \
  --query "let threshold = 5; let timeframe = 20m; SigninLogs | where TimeGenerated > ago(timeframe) | summarize FailedAttempts = countif(ResultType != \"0\"), SuccessAttempts = countif(ResultType == \"0\"), IPAddresses = make_set(IPAddress, 5), LastFailureTime = maxif(TimeGenerated, ResultType != \"0\") by UserPrincipalName | where FailedAttempts >= threshold and SuccessAttempts > 0" \
  --query-frequency PT10M \
  --query-period PT20M \
  --trigger-operator GreaterThan \
  --trigger-threshold 0 \
  --tactics CredentialAccess \
  --techniques T1110
```

Or via the portal for a friendlier first pass: **Microsoft Sentinel** → **Analytics** → **+ Create** → **Scheduled query rule**, and set each field below manually.

### Step 4: Configure Entity Mapping
Entity mapping is what turns a plain alert into something an investigation graph can pivot on — without it, Sentinel has no structured idea that `UserPrincipalName` in your results *is* an Account entity.

In the portal rule wizard, **Set rule logic** → **Entity mapping**:
| Entity type | Identifier | Column |
|---|---|---|
| Account | Full Name | `UserPrincipalName` |
| IP | Address | `IPAddresses` |

**Why this matters**: entity mapping is the mechanism behind Lab 5's investigation graph — pivot from this incident to "what else did this account do" only works because Sentinel knows `UserPrincipalName` maps to a real Account entity, not just an arbitrary string column.

### Step 5: Configure Alert Grouping and Incident Settings
**Incident settings** tab:
1. **Create incidents from alerts triggered by this analytics rule**: **Enabled**
2. **Alert grouping**: **Enabled** — group alerts into a single incident if:
   - **Grouping method**: **Group alerts into a single incident if all the entities match** (Account + IP)
   - **Time window**: `24 hours`
3. **Reopen matching incidents**: **Enabled** — if the same account triggers again after the incident was closed, reopen it rather than creating a duplicate

**Why this matters**: without grouping, a single ongoing brute-force attempt against one account could spawn a new incident every 10-minute rule run — a queue-flooding problem real SOCs actively tune against. Grouping by matching entities collapses repeated firings against the same account into one incident the analyst works once.

---

## Part 3: Trigger and Validate the Detection

### Step 6: Generate Test Signal
You can't realistically fake failed sign-ins against a real tenant safely, so simulate the pattern by reasoning through it: if you have a non-production test account, deliberately mistype its password 5+ times in the Azure portal sign-in page, then sign in successfully. If you don't have a disposable test account, review the query against historical data instead — most tenants have *some* real mistyped-password activity in `SigninLogs` you can validate the logic against.

```kusto
// Confirm the pattern exists somewhere in recent history before relying on a live test
SigninLogs
| where TimeGenerated > ago(7d)
| summarize FailedAttempts = countif(ResultType != "0"), SuccessAttempts = countif(ResultType == "0") by UserPrincipalName, bin(TimeGenerated, 20m)
| where FailedAttempts >= 5 and SuccessAttempts > 0
```

### Step 7: Confirm Incident Creation
1. **Microsoft Sentinel** → **Incidents**
2. Wait for the next scheduled run (up to 10 minutes per the `--query-frequency` set above)
3. Confirm a new incident titled **Repeated Sign-In Failures Followed by Success** appears, with **Entities** showing the mapped Account and IP
4. Open it → confirm **MITRE ATT&CK** tab shows **Credential Access → Brute Force (T1110)**

**Expected result**: the incident's entity panel shows a clickable Account entity (not just raw text), and the MITRE tab reflects the tactic/technique you set in Step 3 — both are proof the rule was engineered correctly, not just "a query that returned rows."

---

## Part 4: Tune the Rule

### Step 8: Reduce Noise Without Losing Coverage
A first-pass rule almost always needs tuning. Two realistic tuning moves for this detection:

**Exclude known-noisy service accounts or break-glass accounts** that legitimately retry often:
```kusto
let threshold = 5;
let timeframe = 20m;
let ExcludedAccounts = dynamic(["<break-glass-account-upn>", "<service-account-upn>"]);
SigninLogs
| where TimeGenerated > ago(timeframe)
| where UserPrincipalName !in (ExcludedAccounts)
| summarize
    FailedAttempts = countif(ResultType != "0"),
    SuccessAttempts = countif(ResultType == "0"),
    IPAddresses = make_set(IPAddress, 5)
    by UserPrincipalName
| where FailedAttempts >= threshold and SuccessAttempts > 0
```

**Raise the threshold if the rule fires too often for normal typo behavior** — check first with a tuning query before guessing:
```kusto
SigninLogs
| where TimeGenerated > ago(30d)
| where ResultType != "0"
| summarize FailedAttempts = count() by UserPrincipalName, bin(TimeGenerated, 20m)
| summarize DistinctIncidentWindows = dcount(bin_TimeGenerated) by FailedAttempts
| sort by FailedAttempts asc
```
This shows the distribution of failure counts across your actual tenant history — if most legitimate typo-driven failures top out at 2–3 attempts and only real attacks produce 5+, your threshold of 5 is well-calibrated; if legitimate users regularly hit 5+ (e.g., an app with a broken retry loop), raise the threshold.

### Step 9: Apply the Tuned Query
Update the rule with the exclusion list added, either via the portal (**Analytics** → select the rule → **Edit** → update the query) or by re-running the `az sentinel alert-rule create` command from Step 3 with the same `--rule-id` and the updated `--query`.

**Validation checkpoint**: after tuning, re-run the Step 6 historical validation query and confirm known-noisy accounts no longer appear in results while genuine brute-force-shaped activity still does.

---

## Part 5: Cleanup

```bash
# Disable or delete the analytics rule
az sentinel alert-rule delete \
  --resource-group sc200-lab1-rg \
  --workspace-name sc200-sentinel-law \
  --rule-id "brute-force-signin-detection" \
  --yes

# Close out any test incidents generated during this lab
```
1. **Microsoft Sentinel** → **Incidents** → select the test incident → **Status**: `Closed` → **Classification**: `True positive - expected/authorized` → comment: "Lab 3 detection engineering exercise"

> If you're continuing to Lab 5 (Incident Response) or Lab 6 (SOAR Automation), leave the rule enabled — both labs use it as the trigger for later work.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Stating the risk before writing the query** | Every detection needs a defensible "why" — this is what you explain in an incident review or an interview |
| **Entity mapping (Account, IP)** | The structured link between an alert and an investigable entity — powers the investigation graph in Lab 5 |
| **MITRE ATT&CK tactic/technique tagging** | Standardizes detection coverage reporting against a framework every SOC and every interviewer recognizes |
| **Alert grouping and incident creation tuning** | Prevents one ongoing attack from flooding the incident queue with duplicates |
| **Tuning against historical data before guessing a threshold** | Data-driven threshold-setting instead of picking an arbitrary number and hoping |

---

## Common Mistakes to Avoid
- **Skipping entity mapping**: an alert with no mapped entities still creates an incident, but the investigation graph in Lab 5 has nothing to pivot on — the incident becomes much harder to work
- **No alert grouping on a rule that can fire repeatedly**: one ongoing attack against a single account floods the incident queue with near-duplicate incidents instead of one incident an analyst can own
- **Picking a threshold by guessing instead of querying historical data**: Step 8's distribution query takes two minutes and prevents both an under-tuned (noisy) and over-tuned (blind) rule
- **Writing a rule with no MITRE mapping**: coverage reporting and gap analysis (a real SOC exercise, and an SC-200 topic) both depend on rules being tagged consistently
- **Never revisiting a rule after it ships**: detection engineering is iterative — a rule tuned once against day-one data will drift as the environment changes

---

## Next Steps
- Continue to [Lab 4: Microsoft Defender XDR — Endpoint & Identity Protection](lab-4-defender-xdr-endpoint-identity.md) to add endpoint and identity telemetry beyond what Sentinel's own connectors provide
- The tuned rule from this lab becomes the trigger source for [Lab 6: SOAR Automation with Logic Apps Playbooks](lab-6-soar-automation-logic-apps.md) — keep it enabled if you're continuing the track in order
- For the security-engineer counterpart to this lab (same analytics-rule mechanics, framed around infrastructure change detection), see [AZ-500 Lab 4](../AZ-500/lab-4-security-ops-automation.md)
