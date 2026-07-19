# Lab 8: Defender for Cloud Apps & Cross-Workload Correlation (Capstone)

Check box if done: [ ]

## Overview
This capstone adds the last major signal source this track hasn't touched — SaaS/cloud application activity via Microsoft Defender for Cloud Apps (MDCA) — and then ties every prior lab together into one correlated investigation. You'll configure MDCA anomaly detection and OAuth app governance policies, connect its logs into Sentinel, write a KQL query correlating identity (Sentinel), endpoint/identity (Defender XDR), and cloud app activity (MDCA) for a single account, and walk a full capstone scenario that chains a Lab 3 detection through Lab 4's XDR correlation, Lab 6's automated response, and Lab 7's hunting discipline into one narrative — the way a real multi-stage investigation actually unfolds.

**Estimated time**: 90–120 minutes
**Cost**: ~$1–$5 (reuses existing Sentinel workspace; MDCA licensing is trial-based with no separate Azure resource cost beyond incidental query/ingestion volume)

---

## Scenario
Six weeks into the job, you get an incident that doesn't fit neatly into any single lab you've done so far: a user's account shows the low-and-slow credential pattern your Lab 7 hunt promoted to a rule, followed days later by an OAuth app requesting broad mailbox permissions, followed by unusual file-sharing activity in a SaaS app outside business hours. No single signal source tells the whole story — Sentinel sees the sign-ins, Defender XDR sees the device, and only Defender for Cloud Apps sees the OAuth grant and the file activity. This lab builds the last piece of that visibility and then walks the whole correlated story.

---

## Objectives
- Connect Defender for Cloud Apps to your Microsoft 365 environment and review the app catalog
- Create an anomaly detection policy and an OAuth app governance policy
- Connect Defender for Cloud Apps logs into Sentinel
- Write a cross-product KQL query correlating Sentinel, Defender XDR, and MDCA signal for one identity
- Walk a full capstone incident narrative spanning every prior lab in this track

---

## Prerequisites for This Lab
- **Microsoft Defender for Cloud Apps** license (included in M365 E5 trial, or standalone MDCA trial)
- Completion of, or familiarity with, Labs 1–7 — this lab explicitly builds on infrastructure and concepts from each of them
- Security Administrator or Cloud App Security Administrator role in `security.microsoft.com` / `security.microsoft.com/cloudapps`

---

## Part 1: Connect Defender for Cloud Apps

### Step 1: Connect Microsoft 365
1. **security.microsoft.com** → **Cloud Apps** → **Connected apps** → **App Connectors**
2. **+ Connect an app** → **Microsoft 365**
3. Grant the requested API-level consent — MDCA uses each SaaS provider's native API to pull activity logs rather than a network-level proxy for first-party Microsoft 365 connections
4. **Connect** — initial data population can take several hours in a live environment; for this lab, review the connector's expected data types while it populates

### Step 2: Review the Cloud App Catalog
1. **Cloud Apps** → **Cloud app catalog**
2. Search for a few common third-party apps your org might use — note each app's **Risk score** (calculated from dozens of factors: data encryption, compliance certifications, vulnerability history)
3. This catalog is what powers **Cloud Discovery** — analyzing firewall/proxy logs to surface *unsanctioned* SaaS usage (shadow IT) an org didn't explicitly approve, distinct from the sanctioned Microsoft 365 connection in Step 1

---

## Part 2: Build Cloud App Security Policies

### Step 3: Create an Anomaly Detection Policy
1. **Cloud Apps** → **Policies** → **Policy management** → **+ Create policy** → **Anomaly detection policy**
2. Review the built-in **Impossible travel** policy template — it's enabled by default, but confirm its scope:
   - **Users**: all users, or exclude known-traveling executives/service accounts prone to false positives
   - **Sensitivity**: `Medium` is a reasonable default — `Low` sensitivity widens the definition of "impossible," increasing false positives
3. This is the same impossible-travel concept from Lab 2's `dcount(IPAddress)` query and Lab 1's `SigninLogs` — MDCA's version additionally factors in travel velocity between the two locations, which raw sign-in log analysis alone doesn't calculate

### Step 4: Create an OAuth App Governance Policy
This is the policy specific to the scenario's OAuth-permission-grant thread:
1. **Policies** → **Policy management** → **+ Create policy** → **OAuth app policy**
2. **Name**: `Flag high-permission mailbox OAuth grants`
3. **App permissions**: filter to apps requesting `Mail.ReadWrite` or `Mail.Send` at the `Mail` category, **High** permission level
4. **Filters**: **Community use** → apps used by few other organizations (a low community-trust signal worth flagging even if the permission itself is common)
5. **Alerts**: **Create an alert for each matching event**
6. **Create**

**Why this matters for the scenario**: an OAuth app requesting broad mailbox access is one of the more dangerous, and more easily overlooked, consent-phishing vectors — a user clicking "Accept" on a malicious app's permission request grants persistent API access without ever typing a password, which is why it doesn't show up in any sign-in log the way a credential-based attack would.

### Step 5: Create a Session Policy for File Activity (Optional Depth)
If your MDCA license includes Conditional Access App Control:
1. **Policies** → **+ Create policy** → **Session policy**
2. **Session control type**: `Control file download (with inspection)`
3. **Activity source**: `File download` where sensitivity label matches a category you'd want to block or monitor from unmanaged devices

---

## Part 3: Connect MDCA to Sentinel

### Step 6: Install the Connector
1. **Microsoft Sentinel** → **Content hub** → search **Microsoft Defender for Cloud Apps** → **Install**
2. **Data connectors** → **Microsoft Defender for Cloud Apps** → **Open connector page** → **Connect** (alerts) and optionally **Connect** (discovery logs, if using Cloud Discovery)

### Step 7: Validate Ingestion
```kusto
// Confirms MDCA alerts are landing in the same workspace as Sentinel's own incidents
SecurityAlert
| where TimeGenerated > ago(1d)
| where ProductName == "Microsoft Cloud App Security"
| project TimeGenerated, AlertName, AlertSeverity, Entities
```

---

## Part 4: Cross-Product Correlation Query

### Step 8: Build the Three-Way Correlation
This is the query the scenario is actually asking for: one identity's activity across Sentinel (sign-ins), Defender XDR (device logons), and MDCA (cloud app activity) in a single timeline.
```kusto
let TargetUser = "<flagged-user-upn>";
let LookbackWindow = 14d;
let SignInActivity = SigninLogs
    | where TimeGenerated > ago(LookbackWindow)
    | where UserPrincipalName == TargetUser
    | project TimeGenerated, Source = "Sentinel-SignIn", Detail = strcat(AppDisplayName, " from ", IPAddress), Severity = iff(ResultType != "0", "Medium", "Info");
let CloudAppActivity = SecurityAlert
    | where TimeGenerated > ago(LookbackWindow)
    | where ProductName == "Microsoft Cloud App Security"
    | where Entities has TargetUser
    | project TimeGenerated, Source = "MDCA-Alert", Detail = AlertName, Severity = AlertSeverity;
SignInActivity
| union CloudAppActivity
| sort by TimeGenerated asc
```
If you completed [Lab 4's](lab-4-defender-xdr-endpoint-identity.md) raw table connector, extend this with a third `union` branch against `IdentityLogonEvents` or `DeviceLogonEvents` for full three-workload coverage. This single-query timeline is the practical answer to "no single signal source tells the whole story" — it's the same `union` pattern from Lab 2's `GetUserActivity` function, generalized across product boundaries instead of just table boundaries.

### Step 9: Save This as a Function Too
Following the discipline from Lab 2:
1. **Logs** → paste the query body → **Save** → **Save as function**
2. **Function name**: `GetCrossWorkloadTimeline`
3. This becomes the single most useful query in the entire workspace for any multi-stage investigation going forward — reusable exactly like `GetUserActivity`, at a broader scope

---

## Part 5: Walk the Capstone Incident

### Step 10: Reconstruct the Full Narrative
Using `GetCrossWorkloadTimeline("<flagged-user-upn>", 14d)`, walk through what a real correlated investigation looks like, mapping each stage back to the lab that built the capability:

1. **Day 1–5**: Low-volume failed sign-ins, several per day, never exceeding the Lab 3 rule's single-window threshold — invisible to that rule alone, but caught by the **Lab 7** hunting query once promoted to a standing detection
2. **Day 6**: A successful sign-in from a new location, triggering MDCA's **impossible travel** anomaly policy (**Part 2** of this lab)
3. **Day 7**: The same account grants a third-party OAuth app broad mailbox permissions, caught by the **OAuth app governance policy** (**Step 4**)
4. **Day 7, later**: Defender for Endpoint (**Lab 4**) shows unusual process activity on the account owner's device — if you connected raw Defender XDR tables, this appears in the same `GetCrossWorkloadTimeline` output
5. **Response**: this pattern — credential compromise, followed by persistent OAuth access, followed by endpoint anomaly — is exactly the kind of high-severity chain that should route through the approval-gated playbook from **Lab 6**, this time with the approval justified rather than rejected
6. **Closure**: the incident gets the full **Lab 5** treatment — investigation graph, documented comments, containment (disable the account **and** revoke the OAuth app's consent), and a `True positive` classification

### Step 11: Revoke a Malicious OAuth Grant (the Missing Containment Action from Lab 5)
Lab 5 covered account disable and session revocation. This scenario needs one more containment action Lab 5 didn't: revoking app consent.
```powershell
# Requires the Microsoft.Graph module connected with Application.ReadWrite.All / DelegatedPermissionGrant.ReadWrite.All
Get-MgUserOauth2PermissionGrant -UserId "<flagged-user-upn>" |
    Where-Object { $_.ClientId -eq "<malicious-app-object-id>" } |
    ForEach-Object { Remove-MgOauth2PermissionGrant -OAuth2PermissionGrantId $_.Id }
```
Or via MDCA directly: **Cloud Apps** → **OAuth apps** → select the flagged app → **Ban app** — this revokes the app's access tokens tenant-wide, not just for the one affected user, which matters if the same malicious app targeted multiple accounts via the same consent-phishing campaign.

**Validation checkpoint**: confirm via `Get-MgUserOauth2PermissionGrant -UserId "<flagged-user-upn>"` that the malicious app no longer appears in the user's granted permissions.

---

## Part 6: Cleanup

```bash
# Remove the MDCA connector if not continuing to use it beyond this track
```
1. **Microsoft Sentinel** → **Data connectors** → **Microsoft Defender for Cloud Apps** → **Disconnect** if this was trial-only
2. **Cloud Apps** → **Policies** → delete or disable the test policies created in Part 2 if they were lab-only and not intended for ongoing use
3. If any test resource groups remain from earlier labs in this track, confirm final teardown:
```bash
az group delete --name sc200-lab1-rg --yes --no-wait
az group list --query "[?starts_with(name, 'sc200-lab')].name" -o tsv
```
4. Re-enable any test account and re-consent any test OAuth app you deliberately disabled/revoked for this lab's exercise, if they were needed for anything beyond the lab itself

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Connecting and configuring Defender for Cloud Apps** | The SaaS/cloud-app visibility layer most SOC analysts underuse relative to endpoint and identity telemetry |
| **OAuth app governance policy** | Catches consent-phishing, a credential-less compromise vector invisible to sign-in-log-based detection |
| **MDCA → Sentinel connector and cross-product KQL correlation** | The practical mechanics behind "single pane of glass" — one query, three data sources, one identity's story |
| **Reconstructing a multi-stage incident across every prior lab's capability** | The actual shape of a real, non-trivial investigation — no single tool or lab in isolation tells the whole story |
| **Revoking OAuth consent as a distinct containment action** | Closes the gap Lab 5's account-disable/session-revoke containment options didn't cover |

---

## Common Mistakes to Avoid
- **Treating MDCA as redundant with Sentinel/Defender XDR**: it sees SaaS-layer signal (OAuth grants, file activity, app risk) that neither of the other two products observes — skipping it leaves a real blind spot, not a duplicated one
- **Disabling an account without also revoking OAuth consent**: a disabled account's previously-granted OAuth tokens can, depending on the grant type, continue functioning until explicitly revoked — account disable alone doesn't always fully contain an OAuth-based compromise
- **Banning an OAuth app without checking how many other users granted it consent**: a consent-phishing campaign rarely targets just one account — check `Cloud Apps` → `OAuth apps` for other users who granted the same app before considering the incident closed
- **Building the cross-workload query once and never reusing it**: save it as a function (Step 9) the same way Lab 2 taught — a query this valuable shouldn't be re-typed
- **Walking through this capstone as a checklist instead of a narrative**: the point of Part 5 is connecting *why* each lab's capability mattered at each stage — that's the story worth being able to tell in an interview, not just "I did 8 labs"

---

## Next Steps
- This closes out the SC-200 track. Revisit [Lab 3](lab-3-analytics-rules-detection-engineering.md) and [Lab 7](lab-7-threat-hunting.md) periodically — detection engineering and hunting are never "done," they're an ongoing tuning loop against a changing environment
- For the security-engineering counterpart — hardening the controls (Conditional Access, Key Vault, network segmentation) that reduce how often incidents like this capstone's scenario occur in the first place — see [AZ-500](../AZ-500/README.md)
- For the identity-governance-at-scale counterpart — the PIM, access reviews, and Graph automation that reduce standing privileged access an attacker could exploit after a compromise like this one — see [SC-300](../SC-300/README.md)
- For the architecture-level question of how to design the SIEM/XDR strategy this entire track operationalizes — workspace topology, connector prioritization, build-vs-buy on detection content — see [SC-100](../SC-100/README.md)
