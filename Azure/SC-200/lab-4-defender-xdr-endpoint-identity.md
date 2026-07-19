# Lab 4: Microsoft Defender XDR — Endpoint & Identity Protection

Check box if done: [ ]

## Overview
Sentinel is a SIEM built on top of whatever telemetry you connect to it — but a large share of the highest-fidelity signal in a real SOC comes from Microsoft Defender XDR, not raw logs. Defender for Endpoint sees process trees, file behavior, and network connections directly on the device; Defender for Identity watches domain-controller traffic for lateral-movement and credential-theft techniques no sign-in log alone reveals. This lab covers onboarding and reading both, then connects Defender XDR to Sentinel so its incidents flow into the same queue you built detections for in Lab 3 — the unified view that lets an analyst work one incident instead of stitching together three portals.

**Estimated time**: 60–90 minutes
**Cost**: ~$0 (requires a Defender for Endpoint / Defender for Identity trial or M365 E5 trial license — no billable Azure resources beyond the existing Sentinel workspace)

---

## Scenario
Sentinel alone caught the brute-force pattern from Lab 3 by watching sign-in logs — but it has no visibility into what happens *after* a successful compromise: did the attacker move laterally to another machine, or query Active Directory for other privileged accounts? Your team lead wants Defender for Endpoint and Defender for Identity onboarded and their signal routed into the same Sentinel incident queue, so an analyst working a case sees the full picture — identity, endpoint, and network — in one place.

---

## Objectives
- Understand Defender for Endpoint's architecture and onboarding methods
- Onboard a device and review its inventory, timeline, and alerts
- Understand Defender for Identity's sensor model and the lateral-movement signals it surfaces
- Connect the Microsoft Defender XDR data connector to Sentinel
- Review a correlated incident in the unified Microsoft Defender portal

---

## Prerequisites for This Lab
- **Microsoft Defender for Endpoint Plan 2** (or M365 E5 trial, which includes it)
- **Microsoft Defender for Identity** trial, with the sensor requiring a domain controller to install on — if you don't have a lab AD environment, complete Parts 1 and 3 conceptually using the architecture walkthroughs and skip live sensor installation; Part 4 (Sentinel connector) works regardless
- Access to `security.microsoft.com` with Security Administrator or Global Administrator role

---

## Part 1: Defender for Endpoint — Onboarding

### Step 1: Understand the Onboarding Methods
Before touching a device, know the options — this is exam-relevant on its own:
| Method | Best For |
|---|---|
| **Local script (PowerShell)** | One-off test machines, small pilots |
| **Group Policy** | On-prem AD-joined fleets |
| **Microsoft Intune / Configuration Manager** | Entra-joined or hybrid-joined managed fleets — the standard production method |
| **VDI onboarding script** | Non-persistent virtual desktop pools |

Production environments almost always use Intune or Configuration Manager for consistency and scale; the local script exists for exactly this kind of lab or a single quick pilot.

### Step 2: Generate the Onboarding Package
1. **security.microsoft.com** → **Settings** → **Endpoints** → **Onboarding**
2. **Operating system**: `Windows 10/11`
3. **Deployment method**: `Local Script (for up to 10 devices)`
4. **Download package** — this produces `WindowsDefenderATPOnboardingScript.cmd`

### Step 3: Run the Onboarding Script
On the target `<device-name>`:
```powershell
# Run as Administrator on the target device
& "C:\path\to\WindowsDefenderATPOnboardingScript.cmd"
```
This registers the device's Defender sensor with your tenant and begins streaming telemetry — process creation, network connections, file/registry activity — to the Defender for Endpoint backend.

### Step 4: Verify Onboarding
```powershell
# On the onboarded device — confirms the sensor service is running and reporting
Get-Service -Name Sense | Select-Object Status, StartType
```

**Expected result**: `Sense` service status is `Running`. In the portal, **Assets** → **Devices** should show `<device-name>` within 5–20 minutes with **Onboarding status**: `Onboarded`.

### Step 5: Run a Detection Test
Microsoft publishes a safe, signed test script that simulates malicious behavior without any real payload — this is the standard way to validate a fresh onboarding without waiting for a real event:
```powershell
# Downloads and runs Microsoft's official EICAR-style detection test — safe, no real malicious code
powershell.exe -NoExit -ExecutionPolicy Bypass -WindowStyle Hidden $ErrorActionPreference= 'silentlycontinue';(New-Object System.Net.WebClient).DownloadFile('http://127.0.0.1/1.exe', 'C:\test-WDATP-test\invoice.exe');Start-Process 'C:\test-WDATP-test\invoice.exe'
```
This is Microsoft's documented sample command (it deliberately targets a non-routable loopback address and fails safely — it exists purely to trigger detection logic, not to execute anything). Run it from an elevated PowerShell prompt on the onboarded test device only.

**Validation checkpoint**: within a few minutes, **security.microsoft.com** → **Incidents & alerts** → **Alerts** should show a new alert referencing `<device-name>` — confirming the sensor is not just onboarded but actively detecting.

---

## Part 2: Review the Device Inventory and Timeline

### Step 6: Explore Device Inventory
1. **Assets** → **Devices** → select `<device-name>`
2. **Overview** tab: risk level, exposure level, onboarding status, tags
3. **Timeline** tab: a chronological feed of every process, network connection, and file event the sensor observed — this is the endpoint equivalent of the `SigninLogs` timeline you built in Lab 2's `GetUserActivity` function, except at the process level

### Step 7: Pivot from Alert to Timeline
1. Open the alert generated in Step 5 → **Story** view — this graphs the process tree (parent → child processes) that triggered detection
2. Click the flagged process → jumps to that exact point in the device timeline
3. This process-tree pivot is Defender for Endpoint's version of the entity investigation graph you'll use for identity/account entities in Lab 5

---

## Part 3: Defender for Identity — Sensor Model and Signals

### Step 8: Understand the Sensor Architecture
Defender for Identity is fundamentally different from Defender for Endpoint: instead of an agent watching one device's activity, its sensor installs directly on **domain controllers** and AD FS servers, and monitors network traffic and Windows Events for authentication protocol abuse — Kerberos, NTLM, LDAP — that reveals attacker behavior regardless of which endpoint they're using.

| Signal Type | Example Detection |
|---|---|
| **Reconnaissance** | Account enumeration via LDAP, SMB session enumeration |
| **Compromised credentials** | Brute force via Kerberos/NTLM, honeytoken account access |
| **Lateral movement** | Pass-the-hash, pass-the-ticket, remote code execution attempts |
| **Domain dominance** | DCSync (illegitimate replication requests), Golden Ticket usage |

### Step 9: Sensor Installation (Conceptual Walkthrough)
If you have a lab domain controller available:
1. **security.microsoft.com** → **Settings** → **Identities** → **Sensors** → **Add sensor**
2. Download the sensor package, run the installer on `<domain-controller-name>` with a gMSA or service account with directory read permissions
3. Sensor begins parsing DC network traffic and forwarding parsed security events — no agent-side log forwarding needed, it reads traffic directly

If you don't have a lab AD environment, review **Settings** → **Identities** → **Sensors** in the portal to see the configuration surface, and continue to Step 10 — the Sentinel connector and unified incident view work independent of whether you have live sensor data.

### Step 10: Understand a Lateral-Movement Alert
Even without live data, know the shape of the alert you'd be triaging: a **"Suspected DCSync attack"** alert fires when a non-domain-controller account issues a directory replication request — a technique tools like Mimikatz use to dump password hashes for every account in the domain. The alert's entity graph shows the source account, the domain controller targeted, and the specific replication operation attempted — exactly the kind of high-confidence, high-severity signal that has no equivalent in sign-in logs alone.

---

## Part 4: Connect Microsoft Defender XDR to Sentinel

### Step 11: Install the Connector
1. **Microsoft Sentinel** → **Content hub** → search **Microsoft Defender XDR** → **Install**
2. **Data connectors** → **Microsoft Defender XDR** → **Open connector page**
3. Under **Incidents & alerts**, toggle **Connect incidents & alerts** — this ingests Defender for Endpoint, Defender for Identity, Defender for Office 365, and Defender for Cloud Apps incidents directly into Sentinel's incident queue as unified incidents, not separate per-product alerts
4. Optionally enable the raw event tables (`DeviceProcessEvents`, `DeviceNetworkEvents`, `IdentityLogonEvents`) if you want to query Defender XDR telemetry directly with KQL rather than only seeing summarized incidents

### Step 12: Confirm the Connection
```kusto
// Confirms Defender XDR incidents are landing in the Sentinel workspace
SecurityIncident
| where TimeGenerated > ago(1d)
| where ProviderName == "Microsoft Defender XDR"
| project TimeGenerated, Title, Severity, Status
```

**Expected result**: the Step 5 test alert (or any Defender for Identity alert, if you completed Step 9) should appear here as a `SecurityIncident` row within a few minutes of it firing in `security.microsoft.com`.

---

## Part 5: Review the Unified Incident

### Step 13: Work the Incident from the Unified Portal
1. **security.microsoft.com** → **Incidents & alerts** → **Incidents**
2. Open the incident generated from Step 5's test alert
3. Note the **Alerts** tab can contain alerts from *multiple products* under one incident — this is Defender XDR's automatic correlation engine grouping related signal (e.g., a suspicious process alert from Defender for Endpoint plus a related sign-in anomaly) into a single incident rather than making the analyst manually connect them
4. Compare this to the same incident visible from **Microsoft Sentinel** → **Incidents** (post Step 11) — same underlying incident, unified queue, either portal

**This is the core value proposition of XDR over siloed products**: correlation happens automatically at the platform level, before an analyst even opens the incident, instead of being something each analyst has to manually reconstruct by checking three separate tools.

---

## Part 6: Cleanup

```powershell
# Offboard the test device (portal-generated offboarding script, same delivery method as onboarding)
& "C:\path\to\WindowsDefenderATPOffboardingScript_valid_until_<date>.cmd"

# Remove the test artifact created by the detection test in Step 5
Remove-Item -Path "C:\test-WDATP-test" -Recurse -Force -ErrorAction SilentlyContinue
```
1. **security.microsoft.com** → **Incidents & alerts** → close the test incident with a documented classification
2. If this was a trial-only environment, no further cleanup is billable — Defender XDR licensing isn't tied to per-resource Azure spend
3. Leave the **Microsoft Defender XDR** Sentinel connector enabled if continuing to Lab 5 or Lab 8, which both build on unified incidents

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Defender for Endpoint onboarding methods and architecture** | Knowing when local-script onboarding is appropriate vs. when it signals an unmanaged, ad-hoc deployment |
| **Reading a device timeline and process-tree story view** | The endpoint-level equivalent of log correlation — most malware/lateral-movement investigations live here |
| **Defender for Identity's network-sensor model** | Understanding why identity-based attacks (DCSync, pass-the-hash) are invisible to sign-in logs but visible here |
| **Connecting Defender XDR incidents into Sentinel** | Building the single-pane-of-glass incident queue a real SOC operates from, instead of working three portals |
| **Recognizing automatic cross-product correlation** | The actual differentiator of XDR platforms over point products — correlation as a platform feature, not manual analyst work |

---

## Common Mistakes to Avoid
- **Assuming Defender for Endpoint and Defender for Identity have the same deployment model**: one is a device agent, the other is a network sensor on domain controllers — conflating them leads to onboarding the wrong thing in the wrong place
- **Treating the detection test script as optional**: without it, you can't confirm the sensor is actively detecting, only that it's installed — onboarded and working are different claims
- **Not enabling raw event tables when you need KQL-level investigation**: the default connector setting only brings in incidents/alerts, not `DeviceProcessEvents` — enable the raw tables explicitly if a later hunting exercise (Lab 7) needs them
- **Working the same incident twice in two different portals without realizing it's one incident**: Sentinel and the unified Defender portal show the same underlying incident — always confirm which portal your team's actual workflow uses before splitting effort across both
- **Forgetting Defender for Identity requires domain-controller-level access**: this isn't something you casually spin up in an isolated lab without an AD environment already in place

---

## Next Steps
- Continue to [Lab 5: Incident Response & Investigation](lab-5-incident-response-investigation.md) to work a full incident lifecycle using the unified queue you just connected
- If you skipped live Defender for Identity sensor installation, revisit it once you have a lab AD environment — the DCSync/pass-the-hash detections are some of the highest-signal alerts in the entire Defender XDR suite
- For the security-engineer-side counterpart — hardening the endpoints and identities these sensors protect, rather than monitoring them — see [AZ-500 Lab 1](../AZ-500/lab-1-identity-access-protection.md) and [AZ-500 Lab 2](../AZ-500/lab-2-platform-network-security.md)
