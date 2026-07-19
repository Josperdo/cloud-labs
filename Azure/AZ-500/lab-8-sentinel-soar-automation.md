# Lab 8: Sentinel SOAR Playbooks & Incident Response Automation

Check box if done: [ ]

## Overview
This is the capstone of the AZ-500 track, and it's deliberately built to tie every earlier lab together instead of introducing an isolated new topic. Lab 4 got you to a working detection (an NSG rule deletion firing an incident) and a manual triage loop. This lab automates the *response* half: Logic Apps playbooks that take real remediation action, Sentinel Automation Rules that decide when and how those playbooks fire, an enrichment step so you don't auto-remediate on a hunch, and an end-to-end test proving the whole chain actually works. By the end, a compromised account detected by Lab 1's Identity Protection signals gets disabled automatically, and a suspicious network event tied to Lab 2/6's controls triggers automatic VM isolation — without you clicking through an incident by hand.

**Estimated time**: 90–120 minutes
**Cost**: ~$1–$3 (Logic Apps consumption pricing is pay-per-execution and trivially cheap for a lab's worth of test triggers; reuses the Log Analytics workspace and Sentinel instance from Lab 4 — no new standing infrastructure beyond it)

---

## Scenario
Your SOC has one analyst covering identity, network, and data alerts across every lab environment in this track, and manual triage doesn't scale. Leadership wants two things automated with guardrails: a compromised user account should be disabled the moment a high-confidence risk signal fires, and a VM showing signs of compromise should be network-isolated automatically — but neither should fire on a low-confidence alert, because an auto-remediation system that disables real users on false positives will get turned off within a week by an angry help desk. You're building the automation and the enrichment gate that keeps it trustworthy.

---

## Objectives
- Add a Sentinel analytics rule for a compromised-account signal, extending Lab 4's rule set
- Build a Logic App playbook that disables a compromised user account via Microsoft Graph
- Build a second Logic App playbook that isolates a VM via NSG changes
- Wire both playbooks to Sentinel Automation Rules rather than firing them directly from analytics rules
- Add an enrichment step so remediation only fires when justified, not on every alert
- Run an end-to-end test and confirm the incident lifecycle closes correctly
- Tear down every automation component explicitly

---

## Part 1: Extend the Detection Layer

### Step 1: Recap What Lab 4 Already Built
Lab 4 connected Azure Activity to Sentinel and wrote a scheduled analytics rule detecting NSG rule deletions. That rule and connector are the foundation this lab builds on — if you're starting fresh, revisit Lab 4's Steps 6–8 first (Log Analytics workspace, Sentinel enablement, Azure Activity connector).

### Step 2: Add a Compromised-Account Analytics Rule
This rule ties directly to Lab 1's Identity Protection risk signals — a risky sign-in or risky user event is exactly the kind of high-confidence trigger worth automating a response to.

1. **Microsoft Sentinel** → **Data connectors** → connect **Microsoft Entra ID Protection** (surfaces risky user/risky sign-in events into the workspace)
2. **Analytics** → **+ Create** → **Scheduled query rule**
3. **General**: Name: `High-Risk User Detected`, Severity: `High`
4. **Set rule logic**:
```kusto
AADRiskyUsers
| where RiskLevel == "high"
| where TimeGenerated > ago(5m)
```
5. **Run query every**: `5 minutes`, **Alert threshold**: greater than 0
6. **Incident settings**: Create incidents — **Enabled**
7. **Review + create**

**Validation checkpoint**: `az monitor log-analytics workspace table show --resource-group az500-lab4-rg --workspace-name az500-sentinel-law --name AADRiskyUsers` (or the equivalent portal Logs query `AADRiskyUsers | take 10`) confirms the table exists and is receiving data before you rely on it.

---

## Part 2: Playbook 1 — Auto-Disable a Compromised Account

### Step 3: Why This Playbook Needs Careful Scoping
Disabling a user account is a real, disruptive action — get the trigger condition wrong and you've built an automated denial-of-service against your own workforce. This playbook should only ever be wired to fire from the automation-rule layer in Part 4, never called directly from an analytics rule with no enrichment gate in front of it.

### Step 4: Create the Logic App

```bash
az group create --name az500-lab8-rg --location eastus2

az logic-app create \
  --resource-group az500-lab8-rg \
  --name az500-disable-user-playbook \
  --location eastus2 \
  --definition '{
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "triggers": {
      "Microsoft_Sentinel_incident": {
        "type": "ApiConnectionWebhook",
        "inputs": {}
      }
    },
    "actions": {}
  }'
```

In practice, this scaffold is filled in through the Sentinel-integrated Logic App designer rather than raw CLI JSON — the next step walks that path, which is how you'd actually build it.

### Step 5: Build the Playbook in the Designer
1. **Microsoft Sentinel** → **Automation** → **Playbooks** → **az500-disable-user-playbook** → **Edit in Logic Apps designer** (or create fresh from the Sentinel-specific playbook template)
2. **Trigger**: **Microsoft Sentinel** → **When Sentinel incident creation rule is triggered**
3. **Action 1 — get the incident's associated user entity**: **Microsoft Sentinel** → **Entities - Get Accounts**, feeding `Incident ARM ID` from the trigger
4. **Action 2 — disable the account**: **Microsoft Graph** connector → **Update user** → set `Account Enabled` = `false`, using the account object ID from Action 1's output
5. **Action 3 — add an incident comment**: **Microsoft Sentinel** → **Add comment to incident**, e.g. `"Auto-disabled account <account> due to High-Risk User Detected — automated response, review required."`
6. **Save**

**Why "add a comment" belongs in every automated playbook**: an analyst reviewing this incident later needs to know a machine took the action, not a human — the audit trail matters as much as the remediation itself.

---

## Part 3: Playbook 2 — Auto-Isolate a Suspicious VM

### Step 6: Build the Isolation Playbook
This ties back to the network controls from Lab 2/6 — instead of a human manually attaching a deny-all NSG, the playbook does it the moment a qualifying incident fires.

1. **Microsoft Sentinel** → **Automation** → **Playbooks** → **+ Add** → create `az500-isolate-vm-playbook`, same Sentinel incident trigger as Step 5
2. **Action 1 — get the incident's associated host/IP entity**: **Microsoft Sentinel** → **Entities - Get Hosts** (or **Get IPs**, depending on which entity type the triggering analytics rule surfaces)
3. **Action 2 — isolate the VM's NIC**: **Azure Resource Manager** connector → invoke an action that associates the VM's NIC with an isolation NSG containing only a `DenyAllInBound`/`DenyAllOutBound` rule set (conceptually: swap the NIC's NSG association from `data-nsg` to a dedicated `isolation-nsg` with no allow rules) — in production this is often a small Azure Function or Automation runbook called from the Logic App, since NIC-NSG reassociation isn't a single first-class Logic Apps connector action
4. **Action 3 — add an incident comment** documenting the automated isolation, same pattern as Step 5

```bash
# The isolation NSG this playbook attaches, created ahead of time so the playbook only needs to swap association
az network nsg create --resource-group az500-lab8-rg --name isolation-nsg
# No allow rules added intentionally — implicit DenyAllInBound/DenyAllOutBound is the entire point
```

**Validation checkpoint**: `az network nsg rule list --resource-group az500-lab8-rg --nsg-name isolation-nsg` should return an empty list — no explicit allow rules, meaning the implicit deny-all rules are the only ones in effect once a NIC is associated with it.

---

## Part 4: Automation Rules — Why Not Just Wire Playbooks Directly

### Step 7: The Case for Automation Rules Over Direct Analytics-Rule-to-Playbook Wiring
Sentinel lets you attach a playbook directly to an analytics rule, but that approach doesn't scale: every analytics rule needs its own wiring, there's no central place to see what automation exists, and you can't easily apply one condition (like a business-hours check or an enrichment gate) across multiple rules without duplicating it everywhere. **Automation Rules** sit above both analytics rules and playbooks — one rule engine that evaluates incident conditions and centrally decides which playbook(s) run, in what order, with what incident status changes.

### Step 8: Create an Automation Rule for the High-Risk User Playbook
1. **Microsoft Sentinel** → **Automation** → **+ Create** → **Automation rule**
2. **Name**: `Auto-Disable High-Risk User`
3. **Trigger**: **When incident is created**
4. **Conditions**: **Analytics rule name** → **Equals** → `High-Risk User Detected`
5. **Actions**: **Run playbook** → `az500-disable-user-playbook`
6. **Incident status change**: leave as **Active** (not auto-closed — a disabled account still needs human review, not a silent close)
7. **Order**: `1` (Automation Rules run in order when multiple could match the same incident — set this deliberately, don't leave it to chance)
8. **Create**

Repeat the same pattern for `az500-isolate-vm-playbook`, conditioned on the network-related analytics rule from Lab 4/6 instead.

---

## Part 5: Enrichment Before Remediation

### Step 9: Why "Alert Fires → Immediately Remediate" Is the Wrong Design
Not every alert deserves the same response. A single failed sign-in flagged as medium risk shouldn't trigger the same automated account disable as a confirmed leaked-credential high-risk signal. Auto-remediating on every alert without an enrichment/confidence check is how automation earns a reputation for "breaking things" and gets disabled by frustrated stakeholders — exactly the failure mode described in the scenario.

### Step 10: Add an Enrichment Condition to the Automation Rule
Extend Step 8's automation rule (or the playbook itself) with an additional check before the disable action fires:

1. In the `az500-disable-user-playbook` designer, insert a **Condition** action after Action 1 (get account) and before Action 2 (disable):
   - **If** the risk level from the triggering `AADRiskyUsers` entity is `high` **AND** the account has no active PIM-eligible role justification logged in the last hour (cross-referencing Lab 1's PIM audit history — a legitimate just-in-time elevation shouldn't be misread as compromise)
   - **Then** proceed to disable
   - **Else** add a comment recommending manual review instead of auto-disabling

This mirrors the same "prove it before you act" discipline used throughout the track — Lab 1's report-only Conditional Access, Lab 6's Alert-mode IDPS and Detection-mode WAF, all before enforcement. Automated response deserves the same discipline, not less, because the blast radius of a bad automated action is larger than a bad manual one.

**Validation checkpoint**: Review the playbook run history (**Logic App** → **Overview** → **Runs history**) after a test firing — confirm the enrichment condition branch is visible in the run and that it's actually evaluating, not just present as unused scaffolding.

---

## Part 6: End-to-End Test

### Step 11: Trigger the Chain
```bash
# Simulate conditions that feed AADRiskyUsers in a real tenant (Identity Protection risk detection is
# Microsoft-evaluated and can't be forced synthetically in a lab) — for this exercise, manually insert a
# test incident via the Sentinel API/portal to validate the automation chain without waiting on a real
# risk detection:
```
1. **Microsoft Sentinel** → **Incidents** → **+ Create incident (Preview)** (or use the analytics rule's **Test with current data** capability if available) with a title matching `High-Risk User Detected` and a test user entity attached
2. Confirm the Automation Rule from Step 8 fires within a few minutes
3. **Logic App** → **az500-disable-user-playbook** → **Runs history** → confirm a successful run, and that the enrichment condition evaluated as expected for your test data
4. **Sentinel** → **Incidents** → open the test incident → confirm the automated comment from Step 5 Action 3 is present

### Step 12: Confirm Incident Lifecycle Behavior
- The incident should remain **Active** (not silently auto-closed) with the automated comment attached — a human still needs to review and formally close it with a classification, exactly like Lab 4's manual triage loop
- Confirm the test account shows `accountEnabled: false` via `az ad user show --id <placeholder-test-user-id> --query accountEnabled` (only if you used a genuinely disposable test account — never point this at a real user)

---

## Part 7: Cleanup

```bash
# Disable playbooks before deleting so any in-flight runs complete cleanly
az logic-app deployment stop --resource-group az500-lab8-rg --name az500-disable-user-playbook 2>/dev/null || true
az logic-app deployment stop --resource-group az500-lab8-rg --name az500-isolate-vm-playbook 2>/dev/null || true

az group delete --name az500-lab8-rg --yes --no-wait
```

1. **Microsoft Sentinel** → **Automation** → delete both automation rules (`Auto-Disable High-Risk User` and the VM isolation equivalent) — these live at the Sentinel workspace level, not inside the deleted resource group, and won't be removed by the `az group delete` above
2. If you enabled Defender for SQL, Defender for Storage, or any other paid Defender plan across this track solely for testing, confirm every one is reset to `Free` (`az security pricing list --query "[].{name:name, tier:pricingTier}" -o table`) — these are subscription-scoped and outlive any single resource group's deletion, a lesson worth re-confirming one last time at the end of the track

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Extending Sentinel analytics rules across identity + network signals** | A mature detection layer correlates across domains instead of one rule per resource type |
| **Logic Apps playbooks for real remediation (disable user, isolate VM)** | Moves from alerting to acting — the actual definition of SOAR |
| **Automation Rules as the central orchestration layer** | Centralized, ordered, reusable automation instead of one-off analytics-rule-to-playbook wiring |
| **Enrichment/confidence gating before auto-remediation** | Prevents automation from becoming a false-positive-driven denial-of-service against your own users |
| **End-to-end test with an explicit incident-lifecycle check** | Proves the chain works and closes correctly, not just that a playbook exists |

---

## Common Mistakes to Avoid
- **Wiring a playbook directly to an analytics rule instead of through Automation Rules**: loses the central visibility, ordering, and reusable-condition benefits Automation Rules provide
- **Auto-remediating on every alert regardless of confidence**: this is how automation gets a reputation for breaking things and gets turned off by frustrated stakeholders
- **Auto-closing incidents that involved a real remediation action**: a human should always review and formally classify an incident where the system took disruptive action, not just let it silently close
- **Skipping the audit-trail comment step in a playbook**: a later reviewer needs to know a machine took the action and why, not just that the account is disabled
- **Testing the automation chain against a real production user or VM**: always use a disposable test identity/resource, the same discipline as Lab 1's break-glass account isolation from production-critical paths

---

## Next Steps
- Extend the enrichment step in Part 5 with IP reputation lookups (a threat intelligence connector) before the VM isolation playbook fires, so a single suspicious-but-unconfirmed connection doesn't isolate a production VM
- Add a third playbook that revokes a user's active sessions (`Revoke-MgUserSignInSession` via the Graph connector) alongside the account disable, since a disabled account can still ride out an already-issued token until it expires
- Build a weekly Automation Rule that reviews and reports on every automated action taken that week, so the SOC has a standing audit summary instead of hunting through individual playbook run histories
- This closes out the AZ-500 track's core lab sequence — for the SOC-analyst-focused deep dive on everything this lab touched (KQL, analytics rule tuning, incident investigation, and this same SOAR automation pattern in much greater depth), continue to [SC-200](../SC-200/README.md), particularly [SC-200 Lab 6: SOAR Automation & Logic Apps](../SC-200/lab-6-soar-automation-logic-apps.md). For deeper identity governance beyond what Lab 1/5 covered here, see [SC-300](../SC-300/README.md). To build the equivalent security-operations muscle in AWS, see [AWS SCS-C02](../../Aws/SCS-C02/README.md)
