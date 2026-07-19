# Lab 6: SOAR Automation with Logic Apps Playbooks

Check box if done: [ ]

## Overview
Lab 5 walked the manual decision process behind incident containment — investigate first, act second, document the reasoning. This lab automates the mechanical parts of that response without automating away the judgment: you'll build a Sentinel automation rule wired to a Logic Apps playbook that triggers automatically when the Lab 3 detection fires, enriches the incident with context, and — behind an explicit human-approval gate — can disable a risky account. This is SOAR (Security Orchestration, Automation, and Response) in practice: reducing the mechanical drag of incident response (open portal, look up account, click disable, document it) without removing the analyst from the actual decision.

**Estimated time**: 75–90 minutes
**Cost**: ~$1–$5 (Logic Apps consumption plan bills per action execution — a handful of test runs is a few cents; see Cleanup for how to avoid ongoing charges)

---

## Scenario
Your team lead reviewed how long the manual Lab 5 investigation took and pointed out that the enrichment steps — pulling the account's recent sign-in history, checking the source IP's reputation, posting a summary somewhere the on-call analyst will see it — are the same every time and don't need a human doing them by hand. She wants a playbook that runs automatically on the brute-force detection, does the enrichment, and pauses for approval before taking any action that actually changes something (disabling the account) — full automation on the safe half, a guardrail on the consequential half.

---

## Objectives
- Build a Logic Apps playbook triggered by a Sentinel incident
- Configure least-privilege permissions for the playbook's managed identity
- Wire a Sentinel automation rule to run the playbook when the Lab 3 detection fires
- Add an enrichment step and a conditional, approval-gated remediation action
- Test the full loop end to end and confirm the audit trail

---

## Setup (if starting fresh)
This lab assumes the tuned analytics rule from [Lab 3](lab-3-analytics-rules-detection-engineering.md) is enabled. If you're picking this lab up independently, re-enable it (or any scheduled analytics rule producing incidents) before continuing — the automation rule in Part 3 needs a real trigger source.

---

## Part 1: Create the Logic App

### Step 1: Deploy an Empty Logic App
```bash
az group create --name sc200-lab6-rg --location eastus2

az logic workflow create \
  --resource-group sc200-lab6-rg \
  --location eastus2 \
  --name sc200-disable-risky-user-playbook \
  --definition '{"$schema":"https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#","contentVersion":"1.0.0.0","triggers":{},"actions":{},"outputs":{}}'
```

### Step 2: Enable the Sentinel Trigger
Sentinel playbooks require a specific trigger type registered by the Sentinel connector. This is easiest to configure in the Logic App Designer:
1. Portal → your Logic App → **Logic app designer**
2. Choose trigger: **Microsoft Sentinel** → **When Microsoft Sentinel incident creation rule was triggered**
3. **Save** — this generates the trigger's callback and registers the app as eligible to attach to a Sentinel automation rule

### Step 3: Grant the Playbook a Managed Identity
```bash
az logic workflow identity assign \
  --resource-group sc200-lab6-rg \
  --name sc200-disable-risky-user-playbook
```
Using a system-assigned managed identity means the playbook authenticates to Sentinel and Microsoft Graph without a stored credential or secret — the same least-privilege pattern SC-300 Lab 4 applied to `Connect-MgGraph` scopes.

### Step 4: Assign the Minimum Required Roles
```bash
# Allows the playbook to read/update Sentinel incidents it's triggered against
az role assignment create \
  --assignee-object-id "<playbook-managed-identity-object-id>" \
  --assignee-principal-type ServicePrincipal \
  --role "Microsoft Sentinel Responder" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/sc200-lab1-rg/providers/Microsoft.OperationalInsights/workspaces/sc200-sentinel-law"
```
**Microsoft Sentinel Responder** grants incident read/update and comment permissions — enough to enrich and document, without the broader **Contributor** access that would let the playbook modify analytics rules or connectors it has no business touching.

For the Graph-based user-disable action in Part 3, grant the managed identity the Graph application permission `User.ReadWrite.All` scoped as narrowly as your tenant allows — ideally via a custom role restricted to specific administrative units rather than tenant-wide, if your environment supports it.

---

## Part 2: Build the Enrichment Steps

### Step 5: Pull Incident Entities
Back in the Logic App designer, after the trigger:
1. **+ New step** → **Microsoft Sentinel** → **Entities - Get Accounts**
2. **Incident ARM ID**: dynamic content from the trigger (`Microsoft Sentinel incident tenant Id` / `Incident ARM ID`)

This retrieves the Account entity mapped back in Lab 3, Step 4 — the automation only works because that entity mapping exists; a rule with no mapped entities gives the playbook nothing to act on.

### Step 6: Query Recent Sign-In History for Context
1. **+ New step** → **Azure Monitor Logs** → **Run query and list results**
2. **Subscription/Resource group/Workspace**: `sc200-sentinel-law`
3. **Query**:
```kusto
SigninLogs
| where UserPrincipalName == '@{body('Entities_-_Get_Accounts')?['UserPrincipalName']}'
| where TimeGenerated > ago(7d)
| summarize FailedAttempts = countif(ResultType != "0"), DistinctIPs = dcount(IPAddress) by UserPrincipalName
```
This is the enrichment step: instead of an analyst manually running this query the way you did by hand in Lab 5, the playbook runs it automatically and attaches the result to the incident.

### Step 7: Post the Enrichment as an Incident Comment
1. **+ New step** → **Microsoft Sentinel** → **Add comment to incident (V3)**
2. **Incident ARM ID**: from the trigger
3. **Comment**:
```
Automated enrichment: Account @{body('Entities_-_Get_Accounts')?['UserPrincipalName']} had @{body('Run_query_and_list_results')?['value'][0]['FailedAttempts']} failed sign-ins from @{body('Run_query_and_list_results')?['value'][0]['DistinctIPs']} distinct IPs in the last 7 days.
```
This is exactly the kind of comment you wrote by hand in Lab 5, Step 8 — now generated automatically the moment the incident is created, so the analyst who picks it up starts already informed instead of starting from zero.

---

## Part 3: Add the Approval-Gated Remediation Action

### Step 8: Add a Condition on Severity
Not every incident this playbook could theoretically run against warrants even asking about remediation:
1. **+ New step** → **Control** → **Condition**
2. **If**: `Severity` (from trigger) **is equal to** `High`
3. Everything from here goes in the **If true** branch — a Medium-severity incident (like the Lab 3 rule's default) gets enrichment only, not a remediation prompt

### Step 9: Add a Human Approval Gate
Inside the **If true** branch:
1. **+ New step** → **Office 365 Outlook** (or **Microsoft Teams**) → **Send approval email** (or **Post adaptive card and wait for a response**)
2. **To**: `<soc-lead-email>`
3. **Subject**: `Approval needed: disable account @{body('Entities_-_Get_Accounts')?['UserPrincipalName']}?`
4. **Options**: `Approve`, `Reject`

**This is the deliberate guardrail**: full auto-remediation on a first-version playbook is a large blast radius if the detection has a false-positive bug — Lab 5 spent an entire lab establishing that not every alert warrants containment. Wiring an unconditional auto-disable straight off a Medium-severity brute-force rule would automate past the exact judgment call that lab practiced. An approval gate keeps a human in the loop for the consequential action while still automating everything mechanical around it.

### Step 10: Add the Conditional Remediation Action
Inside a second **Condition**, checking the approval response equals `Approve`:
1. **+ New step** → **Microsoft Graph** (via HTTP action with managed identity auth, or the Entra ID connector if available) → update the user object:
```json
{
  "method": "PATCH",
  "uri": "https://graph.microsoft.com/v1.0/users/@{body('Entities_-_Get_Accounts')?['UserPrincipalName']}",
  "authentication": {
    "type": "ManagedServiceIdentity"
  },
  "body": {
    "accountEnabled": false
  }
}
```
2. Follow with another **Add comment to incident (V3)** step documenting the action: `"Account disabled following SOC lead approval — see approval response for authorization record."`

### Step 11: Handle the Rejected Path
In the **If false** branch of the approval condition, add a comment step instead: `"Disable action rejected by approver — leaving account active, escalate to manual investigation."` An automation that only handles its happy path silently drops the rejection case, which looks identical to "nothing happened" in the incident log — always give the negative branch its own explicit action.

---

## Part 4: Wire It to Sentinel with an Automation Rule

### Step 12: Create the Automation Rule
1. **Microsoft Sentinel** → **Automation** → **+ Create** → **Automation rule**
2. **Name**: `Run enrichment and approval-gated response on brute-force detection`
3. **Trigger**: `When incident is created`
4. **Conditions**: `Analytics rule name` **Equals** `Repeated Sign-In Failures Followed by Success` (the Lab 3 rule)
5. **Actions**: `Run playbook` → select `sc200-disable-risky-user-playbook`
6. **Order**: `1` (automation rule execution order matters if multiple rules could match the same incident)
7. **Enabled**: `Yes`

### Step 13: Confirm the Wiring
```bash
az sentinel automation-rule show \
  --resource-group sc200-lab1-rg \
  --workspace-name sc200-sentinel-law \
  --automation-rule-id "<automation-rule-id>" \
  --query "{name: displayName, enabled: triggeringLogic.isEnabled}"
```

---

## Part 5: Test End to End

### Step 14: Re-Trigger the Lab 3 Detection
Follow [Lab 3, Step 6](lab-3-analytics-rules-detection-engineering.md) to regenerate the failed-sign-in-then-success pattern, or manually raise a test incident against the same analytics rule if you have one queued from Lab 5.

### Step 15: Watch the Playbook Run
1. **Microsoft Sentinel** → **Incidents** → open the new incident → **Comments** tab — confirm the enrichment comment from Step 7 appears automatically within a minute or two of incident creation
2. Check `<soc-lead-email>` (or Teams) for the approval prompt if the test incident's severity matches Step 8's condition
3. **Logic App** → **Overview** → **Runs history** — open the run, confirm every step succeeded and inspect the actual enrichment values passed between steps

**Validation checkpoint**: the incident should show the enrichment comment automatically, with no manual action taken — this is the proof the automation rule → playbook wiring actually works, independent of whether you approve or reject the remediation prompt.

### Step 16: Test Both Approval Branches
Trigger the flow twice if practical — once approving, once rejecting — and confirm:
- **Approve** path: account `accountEnabled` flips to `false`, comment documents the approved action
- **Reject** path: account remains enabled, comment documents the rejection explicitly (Step 11)

---

## Part 6: Cleanup

```bash
# Delete the automation rule
az sentinel automation-rule delete \
  --resource-group sc200-lab1-rg \
  --workspace-name sc200-sentinel-law \
  --automation-rule-id "<automation-rule-id>" \
  --yes

# Delete the Logic App and its resource group
az group delete --name sc200-lab6-rg --yes --no-wait
```

```powershell
# Re-enable any test account you disabled during Step 16's approve-path test
Update-MgUser -UserId "<test-user-upn>" -AccountEnabled:$true
```

> **Important**: confirm no test runs are still queued in the Logic App's run history with a pending approval — an unresolved "waiting for response" run can sit indefinitely and is worth explicitly cancelling if you're tearing the lab down.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Logic Apps playbook triggered by a Sentinel incident** | The actual mechanism behind SOAR — automated response wired directly to detection, not a separate manual step |
| **Managed identity with scoped roles (Sentinel Responder, narrow Graph permission)** | Least-privilege automation — the playbook can do exactly what it needs and nothing more |
| **Automated enrichment (KQL query run inside the playbook)** | Removes the mechanical, repeatable half of investigation from the analyst's plate |
| **Approval-gated remediation instead of full auto-remediation** | The guardrail that separates responsible SOAR from a script that can cause its own incident |
| **Explicit handling of the rejected/negative approval branch** | Prevents a silent no-op from looking identical to a successful automated response in the audit trail |

---

## Common Mistakes to Avoid
- **Granting the playbook's managed identity broad permissions "to make it work"**: scope to exactly what the playbook does — Sentinel Responder for incident actions, a narrow Graph permission for the specific remediation, nothing tenant-wide by default
- **Auto-remediating without an approval gate on a first-version playbook**: Lab 5 exists specifically to show that not every alert warrants containment — skipping the human gate automates past that judgment call before the detection has proven itself reliable
- **Only building the happy path**: a rejected approval or a failed API call needs its own explicit, documented outcome — not silence
- **Forgetting automation rule conditions can match more broadly than intended**: scoping to a specific analytics rule name (Step 12) prevents this playbook from firing on unrelated incidents it wasn't designed to handle
- **Leaving a test playbook wired to a noisy production analytics rule**: a playbook that pings a SOC lead for approval on every firing of an untuned rule trains people to ignore the prompts — tune the detection (Lab 3) before automating a response to it

---

## Next Steps
- Continue to [Lab 7: Threat Hunting with Sentinel & Defender Advanced Hunting](lab-7-threat-hunting.md) — proactive hunting is what catches what this playbook's trigger rule doesn't
- Extend the playbook to also isolate a device via Defender for Endpoint (from [Lab 4](lab-4-defender-xdr-endpoint-identity.md)) when the incident includes a Host entity, not just an Account entity
- For the infrastructure-as-code counterpart to this lab's manual portal-built playbook, look into deploying Logic Apps definitions via Terraform's `azurerm_logic_app_workflow` resource, extending the pattern from [AZ-500 Lab 4](../AZ-500/lab-4-security-ops-automation.md)
