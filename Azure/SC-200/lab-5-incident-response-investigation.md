# Lab 5: Incident Response & Investigation

Check box if done: [ ]

## Overview
Everything up to this lab has been building the machinery — workspace, connectors, KQL fluency, a tuned detection, and unified XDR incidents. This lab is where you actually do the job: work an incident from the moment it lands in the queue to the moment it's closed with a documented, defensible classification. That loop — triage, investigate, collect evidence, contain, classify, close — is what a SOC analyst does dozens of times a shift, and it's the single most heavily-weighted skill area on the SC-200 exam (Respond to incidents, 35–40%).

**Estimated time**: 60–75 minutes
**Cost**: ~$1–$3 (reuses existing workspace/rule infrastructure; no new billable resources beyond incidental query execution)

---

## Scenario
The brute-force detection rule you engineered in Lab 3 just fired. It's now sitting in your incident queue alongside whatever else landed today. Your job: work it the way a real SOC analyst would — figure out what actually happened, pivot across every entity involved, document what you found, take a containment action if warranted, and close it out with a classification that would hold up if someone audited your work six months later.

---

## Objectives
- Triage an incident queue by severity and status
- Use the investigation graph to pivot across related entities
- Collect and document evidence with comments, tasks, and bookmarks
- Take a manual containment action appropriate to the finding
- Close an incident with a defensible classification

---

## Setup (if starting fresh)
This lab assumes the tuned analytics rule from [Lab 3](lab-3-analytics-rules-detection-engineering.md) is enabled and has produced at least one incident. If you disabled it during Lab 3 cleanup, re-enable it and trigger a test firing (Lab 3, Step 6) before starting here — you need a real incident in the queue to work through this lab meaningfully.

---

## Part 1: Triage the Incident Queue

### Step 1: Review the Queue Like a Shift Start
1. **Microsoft Sentinel** → **Incidents**
2. Sort by **Severity** (High → Low), then within severity by **Last update time**
3. Note the **Status** column: `New`, `Active`, `Closed` — an incident sitting in `New` for hours is a queue-management failure worth flagging in a real SOC

```kusto
// The programmatic version of the same triage — useful for a shift-handoff report
SecurityIncident
| where TimeGenerated > ago(24h)
| where Status != "Closed"
| project TimeGenerated, IncidentNumber, Title, Severity, Status, Owner
| sort by Severity asc, TimeGenerated asc
```

### Step 2: Open the Target Incident and Read the Summary
1. Open the **Repeated Sign-In Failures Followed by Success** incident from Lab 3
2. **Overview** tab: read the alert count, entity count, and MITRE ATT&CK tags — this is your 30-second orientation before diving in, and it's exactly what you'd relay in a two-sentence handoff to another analyst

### Step 3: Check for Related Incidents
```kusto
SecurityIncident
| where TimeGenerated > ago(7d)
| where Title has "Sign-In"
| project TimeGenerated, IncidentNumber, Title, Severity, Status
| sort by TimeGenerated desc
```
**Why this matters**: an incident rarely exists in isolation. If the same account triggered a similar alert last week that was closed as a false positive, that context changes how you triage this one — a repeat offender account is a very different risk posture than a first-time anomaly.

---

## Part 2: Investigate with the Entity Graph

### Step 4: Open the Investigation Graph
1. From the incident, click **Investigate**
2. This renders the entity graph built directly from the entity mapping you configured in Lab 3, Step 4 — the Account and IP entities appear as connected nodes

### Step 5: Pivot from the Account Entity
1. Click the **Account** node → **Entities** panel opens showing related activity: other alerts involving this account, its sign-in history, its group memberships
2. Click **Run playbook** or **Insights** (if available) to see Sentinel's built-in behavioral insights for this account — e.g., "this account has not signed in from this IP in the last 90 days," a strong corroborating signal

### Step 6: Pivot from the IP Entity
1. Click the **IP** node → review the **Insights** tab: does this IP resolve to a known cloud/hosting provider ASN, a residential ISP, or a Tor exit node? This context matters enormously for classification later
2. Cross-reference with your own KQL, since the graph's built-in insights are a starting point, not the full picture:
```kusto
SigninLogs
| where TimeGenerated > ago(7d)
| where IPAddress == "<flagged-ip-address>"
| summarize DistinctUsers = dcount(UserPrincipalName), Attempts = count() by IPAddress
```
An IP hitting many distinct user accounts is a strong password-spray signal distinct from a single-account brute force — worth noting in your final classification.

### Step 7: Expand the Timeline with the Function from Lab 2
```kusto
GetUserActivity("<flagged-user-upn>", 24h)
```
Reuse the reusable function you built in [Lab 2](lab-2-kql-fundamentals.md) — this is exactly the scenario it was built for: a single-call timeline across sign-in and audit data for the entity you're investigating, instead of re-writing the union query under incident pressure.

**Validation checkpoint**: by this point you should be able to state, in one sentence, what happened — e.g., "Account `<upn>` had 6 failed sign-ins from `<ip>` (a residential ISP range, not a known VPN/hosting provider) followed by one success; no unusual audit log activity followed; this looks like the account owner mistyping their password, not a compromise." If you can't state a conclusion yet, you need more evidence, not less investigation time.

---

## Part 3: Collect and Document Evidence

### Step 8: Add Investigation Comments
1. On the incident, **Comments** tab → document your findings from Part 2 in plain language: what you checked, what you found, what it means
2. This isn't busywork — a documented comment trail is what turns "I remember investigating something like this" into an auditable record when the same account triggers again in three months

### Step 9: Create Tasks for Follow-Up Work
1. **Tasks** tab → **+ New task**
2. Example: `Confirm with account owner whether the failed sign-ins were self-initiated` — assign a due date/owner if this needs to leave your hands
3. Tasks keep multi-step investigations from silently stalling — an incident with an open task is visibly not finished, unlike an incident sitting quietly in `Active` with no next step recorded

### Step 10: Bookmark Anything Worth Reuse
If your investigation surfaced a query worth reusing beyond this one incident (e.g., the IP-reputation cross-reference from Step 6):
1. Run the query in **Logs** → select the result rows → **Bookmark selected rows**
2. **Bookmark name**: `Password-spray-pattern-check` — this becomes searchable and reusable in future investigations and directly feeds Lab 7's hunting workflow

---

## Part 4: Containment (When Warranted)

### Step 11: Decide Whether Containment Is Justified
Not every incident needs a response action — this one, based on the Step 7 conclusion, likely doesn't (benign self-mistype). But know the manual containment options for when a true positive does warrant it:

**Disable the compromised account (Entra ID):**
```powershell
Update-MgUser -UserId "<flagged-user-upn>" -AccountEnabled:$false
```

**Revoke active sessions to force re-authentication everywhere:**
```powershell
Revoke-MgUserSignInSession -UserId "<flagged-user-upn>"
```

**Isolate a compromised device (if Defender for Endpoint flagged the same account, from Lab 4):**
1. **security.microsoft.com** → **Devices** → select the device → **Isolate device** → **Full isolation** (keeps the Defender sensor connection alive while cutting all other network activity)

**Why this lab only documents these rather than running them**: containment is a high-blast-radius action — disabling the wrong account or isolating the wrong device has real business impact. This lab's incident, based on the investigation in Part 2, is a documented false positive, so containment isn't exercised live here. [Lab 6](lab-6-soar-automation-logic-apps.md) builds the automated version of the disable-account action with guardrails, once you've seen the manual decision process that should precede automating it.

---

## Part 5: Classify and Close

### Step 12: Set the Classification
1. On the incident, **Status**: `Closed`
2. **Classification**: choose the option that matches your Part 2 conclusion:
   - `True positive - suspicious activity` — if containment was warranted
   - `Benign positive - suspicious but expected` — if the pattern is real but explainable (e.g., a confirmed self-mistype)
   - `False positive - incorrect alert logic` — if the rule itself needs tuning (feeds back into Lab 3)
3. **Comment**: a final one-line summary — this is the line a compliance auditor or your manager reads first: `"Confirmed with account owner: self-initiated failed logins due to a password manager sync issue. No compromise indicators. Closing as benign positive."`

### Step 13: Confirm the Closure Is Queryable
```kusto
SecurityIncident
| where TimeGenerated > ago(1d)
| where Status == "Closed"
| project IncidentNumber, Title, ClassificationReason, ClassificationComment
```
**Validation checkpoint**: your classification and comment should appear here — this table is what a real SOC's monthly metrics report (true-positive rate, mean-time-to-close, classification distribution) is built from, so an incident closed without a proper classification silently corrupts that reporting.

---

## Part 6: Cleanup

No new billable resources were created in this lab — it worked entirely against existing infrastructure from Labs 1–3.

1. Confirm no test accounts were left disabled if you exercised the containment commands in Step 11 against a throwaway account — re-enable if needed:
```powershell
Update-MgUser -UserId "<flagged-user-upn>" -AccountEnabled:$true
```
2. Leave the Lab 3 analytics rule and Lab 1 connectors enabled if continuing to Lab 6, which triggers off the same rule

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Triaging a queue by severity and staleness** | The first five minutes of every SOC shift — knowing what to work first |
| **Using the investigation graph to pivot across entities** | Turns entity mapping (built in Lab 3) into actual investigative leverage instead of a cosmetic feature |
| **Documenting findings via comments and tasks** | The difference between "I think I remember this" and an auditable investigation record |
| **Weighing containment against investigation conclusions** | Real incident response — not every alert warrants disabling an account, and knowing the line matters |
| **Closing with a specific, defensible classification** | Feeds SOC metrics (true-positive rate, time-to-close) and protects you if the closure is ever questioned later |

---

## Common Mistakes to Avoid
- **Jumping straight to containment before investigating**: disabling an account or isolating a device on an unconfirmed alert has real business impact — investigate first, act second, unless the evidence is already overwhelming
- **Closing an incident with no comment or a generic one**: "Closed - reviewed" tells a future auditor nothing; write what you actually found
- **Skipping the check for related/prior incidents**: repeat-offender context (Step 3) frequently changes the correct classification
- **Treating "benign positive" and "false positive" as interchangeable**: benign positive means the *pattern is real* but explainable; false positive means the *detection logic itself* is wrong and needs tuning — misclassifying blurs your Lab 3 tuning signal
- **Leaving tasks open with no owner or due date**: an unowned task is functionally the same as no task — it won't get done

---

## Next Steps
- Continue to [Lab 6: SOAR Automation with Logic Apps Playbooks](lab-6-soar-automation-logic-apps.md) to automate the containment decision process you exercised manually here — with guardrails so automation doesn't skip the judgment calls this lab practiced
- Bookmark queries you write in future real investigations — they become the seed material for [Lab 7's threat hunting workflow](lab-7-threat-hunting.md)
- For the architecture-level question of how incident response fits into a broader security operations strategy (staffing, SLAs, escalation tiers), see [SC-100](../SC-100/README.md)
