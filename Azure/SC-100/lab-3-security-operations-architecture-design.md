# Lab 3: Security Operations Architecture Design

Check box if done: [ ]

## Overview
Lab 2 built the identity layer that generates sign-in and PIM activation logs. None of that matters if there's nowhere coherent to detect, correlate, and respond to what those logs show. This lab designs the SIEM/SOAR topology — how many Sentinel workspaces, what retention tier per data type, how alerts become incidents and incidents trigger automated response — before deploying a representative slice of that design.

**Estimated time**: 60–90 minutes
**Cost**: ~$1–$3 (Sentinel bills per GB ingested on top of the underlying Log Analytics workspace cost; this lab stays under 1 GB by using low-volume test data and deletes the workspace at the end)

---

## Scenario
The org now has three subsidiaries after a recent acquisition, each in a different Azure region for data residency reasons (EU subsidiary's logs legally cannot leave the EU), plus a central security operations team that needs to investigate incidents across all three without three separate consoles. The CISO also wants a cost model that doesn't treat a low-value informational log (e.g., routine sign-in success) the same as a high-value signal (e.g., impossible-travel alert) — ingesting everything into the most expensive tier is not sustainable at the org's log volume. You're designing the Sentinel workspace topology and the retention/tiering strategy before the SOC team commits to it operationally.

---

## Objectives
- Decide between a single centralized Sentinel workspace and a multi-workspace topology, accounting for data residency
- Decide how to tier log retention by data value instead of ingesting everything at the same cost
- Deploy Microsoft Sentinel onto a Log Analytics workspace and connect a data source
- Build a scheduled analytics rule that turns a raw log pattern into an incident
- Configure table-level log plans to demonstrate the retention tiering decision technically, not just on paper
- Add a SOAR automation rule so high-severity incidents get an owner and a bounded first response automatically

---

## Part 1: Design Decision — Sentinel Workspace Topology and Retention Tiering

### Decision 1: Single Centralized Workspace vs. Multi-Workspace with Cross-Workspace Queries

| Factor | Single Centralized Sentinel Workspace | Multi-Workspace (per subsidiary/region) + Azure Lighthouse for cross-tenant/cross-workspace queries |
|---|---|---|
| **Data residency / compliance** | Fails outright if any subsidiary has a legal requirement to keep data in-region — the EU subsidiary's logs cannot sit in a US-region workspace | Satisfies residency by design — each workspace stays in its required region, satisfying local regulatory requirements |
| **SOC investigation experience** | Single console, single KQL query, no federation needed — fastest incident triage | Cross-workspace `workspace()` queries or a Lighthouse-delegated view work, but add query complexity and a small latency/cost overhead per cross-workspace call |
| **Cost** | Ingestion pools together; can hit commitment-tier discounts sooner | Each workspace bills separately; splitting the same total volume across three workspaces can lose a pooled commitment-tier discount unless each individually clears the tier threshold |
| **Blast radius / tenant isolation** | A single compromised workspace exposes every subsidiary's data | Isolated — a compromise or misconfiguration in one workspace doesn't expose the others |
| **RBAC granularity** | Requires table-level RBAC to stop the EU team from querying the US subsidiary's data in the same workspace | Workspace-scoped RBAC is automatic — Reader on a workspace only ever sees that workspace |
| **Fit for this scenario** | Disqualified by the EU data residency requirement alone | Required — the residency constraint forces multi-workspace regardless of the operational cost |

### Decision 2: Retention Tiering — Uniform Analytics Tier vs. Tiered by Log Value

| Factor | Uniform Analytics (Full) Tier for Everything | Tiered: Analytics for high-value / Basic Logs for high-volume-low-value / Archive for compliance-only |
|---|---|---|
| **Cost per GB ingested** | Highest flat rate applied to every log type, including routine informational events | Analytics tier only for logs that need real-time KQL + alerting; Basic Logs tier costs a fraction of Analytics for high-volume tables queried infrequently; Archive tier is cheapest, no live query, restore-on-demand |
| **Query capability** | Full KQL, real-time alerting, all tables identical | Analytics: full KQL + alerting. Basic Logs: limited KQL (single-table search, no join), 8-day free query window then pay-per-query. Archive: no live query — must restore a time range first |
| **Best fit for high-volume/low-signal data** | Wasteful — e.g., verbose network flow logs or routine sign-in success events cost the same per GB as a critical security alert table | Correct fit — route verbose/low-signal tables (e.g., `NetworkFlowLogs`, high-volume `SigninLogs` success events) to Basic Logs; keep alert-driving tables (`SecurityAlert`, `AuditLogs`, `SigninLogs` failures) on Analytics |
| **Compliance-only data (e.g., 7-year retention mandate)** | Expensive to retain at Analytics-tier pricing for years | Archive tier is purpose-built for this — cheap long-term storage, restore only when an audit or investigation needs it |

### Recommendation for This Scenario
**Multi-workspace topology** — one Sentinel-enabled workspace per subsidiary region (satisfying the EU residency constraint), with **Azure Lighthouse delegation** giving the central SOC team a single pane across all three without moving data. **Tiered retention within each workspace**: Analytics tier for `SecurityAlert`, `SigninLogs`, and `AuditLogs` (the tables analytics rules and incidents depend on), Basic Logs tier for high-volume diagnostic tables that are rarely queried live, and Archive tier for anything held only to satisfy a multi-year compliance retention mandate. Part 2–4 build one representative workspace with this tiering applied.

---

## Part 2: Deploy Microsoft Sentinel on a Log Analytics Workspace

### Step 1: Create the Workspace
```bash
az group create --name sc100-lab3-rg --location eastus

az monitor log-analytics workspace create \
  --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 \
  --location eastus --sku PerGB2018 --retention-time 90
```
90-day retention on the workspace default is the Analytics-tier baseline — Part 4 overrides specific tables to Basic Logs to demonstrate the tiering decision.

### Step 2: Onboard Sentinel to the Workspace
```bash
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 --query customerId -o tsv)
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/sc100-lab3-rg/providers/Microsoft.OperationalInsights/workspaces/law-sc100-lab3/providers/Microsoft.SecurityInsights/onboardingStates/default?api-version=2023-11-01" \
  --body "{\"properties\":{}}"
```
**Expected result**: HTTP 200/201. Sentinel is now enabled on this workspace — the portal's Microsoft Sentinel blade should list `law-sc100-lab3` under "Manage active workspaces."

### Step 3: Connect a Data Source
```bash
az extension add --name sentinel --upgrade

az sentinel data-connector create \
  --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 \
  --data-connector-id "$(uuidgen)" \
  --kind "AzureActiveDirectory" \
  --tenant-id "<tenant-id>"
```
Connecting Entra ID sign-in and audit logs is the data source that feeds the analytics rule in Part 3 — it's also exactly the log stream Lab 2's Conditional Access and PIM activity lands in.

**Validation checkpoint**:
```bash
az sentinel data-connector list --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 -o table
```
Confirm the connector shows a `Connected` or provisioning state.

---

## Part 3: Build a Scheduled Analytics Rule

Turn a raw log pattern (repeated failed sign-ins followed by a success — a classic password-spray-then-success pattern) into an actual Sentinel incident, not just a queryable log.

```bash
cat > analytics-rule.json << 'EOF'
{
  "displayName": "SC100-Lab3-Repeated-Failed-SignIn-Then-Success",
  "description": "Detects 5+ failed sign-ins followed by a success for the same user within 30 minutes",
  "severity": "Medium",
  "enabled": true,
  "query": "SigninLogs | where ResultType != \"0\" | summarize FailCount = count() by UserPrincipalName, bin(TimeGenerated, 30m) | where FailCount >= 5 | join kind=inner (SigninLogs | where ResultType == \"0\") on UserPrincipalName",
  "queryFrequency": "PT30M",
  "queryPeriod": "PT30M",
  "triggerOperator": "GreaterThan",
  "triggerThreshold": 0,
  "tactics": ["CredentialAccess", "InitialAccess"]
}
EOF

az sentinel alert-rule create \
  --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 \
  --rule-id "$(uuidgen)" --kind Scheduled \
  --scheduled-rule-body @analytics-rule.json
```

**Validation checkpoint**:
```bash
az sentinel alert-rule list --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 \
  --query "[].{name:displayName, enabled:enabled}" -o table
```
The rule should show `enabled: true`. Once real sign-in data matching the pattern lands (or in a live tenant, once it naturally occurs), Sentinel converts the alert into an incident visible in **Microsoft Sentinel → Incidents** — this is the operational handoff to SC-200's investigation workflow.

---

## Part 4: Apply Tiered Retention to a High-Volume Table

Demonstrate the Decision 2 tiering choice technically: move a high-volume, low-signal table to the Basic Logs plan while leaving alert-driving tables on Analytics.

```bash
az monitor log-analytics workspace table update \
  --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 \
  --name "AzureDiagnostics" --plan Basic

az monitor log-analytics workspace table update \
  --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 \
  --name "SigninLogs" --retention-time 90
```
`AzureDiagnostics` moves to Basic Logs — high-volume, rarely queried live outside an active investigation. `SigninLogs` stays on the Analytics plan at full retention, since it's the table the analytics rule in Part 3 depends on for real-time alerting.

**Validation checkpoint**:
```bash
az monitor log-analytics workspace table list \
  --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 \
  --query "[?name=='AzureDiagnostics' || name=='SigninLogs'].{table:name, plan:plan}" -o table
```
Confirm `AzureDiagnostics` shows `plan: Basic` and `SigninLogs` shows `plan: Analytics` — the tiering decision from Part 1 is now technically enforced, not just documented.

---

## Part 5: Add SOAR Automation — Auto-Escalate High-Severity Incidents

A SIEM without an automation layer still depends entirely on an analyst noticing the incident queue — the "SOAR" half of the design this lab set out to build. An automation rule closes that gap for the highest-severity case: incidents matching Part 3's credential-attack pattern get an owner and a severity-appropriate response the moment they're created, not whenever someone next checks the queue.

### Step 1: Create an Automation Rule Triggered on Incident Creation
```bash
cat > automation-rule.json << 'EOF'
{
  "displayName": "SC100-Lab3-AutoAssign-CredentialAccess-Incidents",
  "order": 1,
  "triggeringLogic": {
    "isEnabled": true,
    "triggersOn": "Incidents",
    "triggersWhen": "Created",
    "conditions": [
      { "conditionType": "Property", "conditionProperties": { "propertyName": "IncidentSeverity", "operator": "Equals", "propertyValues": ["Medium", "High"] } }
    ]
  },
  "actions": [
    { "order": 1, "actionType": "ModifyProperties",
      "actionConfiguration": { "severity": "High", "status": "Active", "owner": { "assignedTo": "<soc-tier2-oncall-group-object-id>" } } },
    { "order": 2, "actionType": "RunPlaybook",
      "actionConfiguration": { "logicAppResourceId": "<disable-user-playbook-resource-id>" } }
  ]
}
EOF

az sentinel automation-rule create \
  --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 \
  --automation-rule-id "$(uuidgen)" --automation-rule @automation-rule.json
```
Chaining a `RunPlaybook` action (invoking a Logic App that, for example, disables the affected user account pending investigation) onto the Part 3 analytics rule is what makes this SOAR rather than just SIEM — detection and a bounded, reviewable first response happen without waiting on human triage latency.

**Validation checkpoint**:
```bash
az sentinel automation-rule list --resource-group sc100-lab3-rg --workspace-name law-sc100-lab3 \
  --query "[].{name:displayName, order:order}" -o table
```
Confirm the rule is listed and `order: 1` — automation rule ordering matters when multiple rules could match the same incident; this rule should fire before any lower-priority rule that only tags or comments.

---

## Cleanup
```bash
az group delete --name sc100-lab3-rg --yes --no-wait
```
This removes the workspace, Sentinel onboarding, the data connector, the analytics rule, and all table-plan configuration together. Confirm with `az group show --name sc100-lab3-rg` — expect a "ResourceGroupNotFound" error once deletion completes.

---

## What You Practiced

| Concept | Why It Matters |
|---|---|
| Multi-workspace Sentinel topology | SC-100 scenarios routinely include a data residency constraint that forces this decision — recognizing the trigger is the exam skill |
| Azure Lighthouse for cross-workspace SOC visibility | Solves the "single pane of glass" requirement without violating residency by centralizing data |
| Retention tiering (Analytics/Basic Logs/Archive) | Directly answers the "SIEM cost is unsustainable at scale" scenario pattern common on this exam |
| Scheduled analytics rules | The mechanism that turns raw ingested logs into an actionable Sentinel incident |
| Table-level plan configuration | Proves the cost/retention design decision is implementable per table, not an all-or-nothing workspace setting |
| SOAR automation rules + playbooks | Completes the "SIEM/SOAR" strategy — detection without automated triage still bottlenecks on human response latency |

---

## Common Mistakes to Avoid
- **Defaulting to a single Sentinel workspace without checking data residency requirements first**: this is the most common trap in SC-100 SOC-design scenarios — always check for a stated residency or sovereignty constraint before recommending centralization
- **Putting every table on the Analytics plan by default**: it's the easiest setting to leave alone and the most expensive one to leave alone at scale
- **Building analytics rules with no MITRE ATT&CK tactic tagged**: untagged rules break the ability to map coverage against a framework, which SC-200's detection engineering (and audits) depend on
- **Forgetting Basic Logs has a shorter free query window and no join support**: moving an alert-driving table to Basic Logs silently breaks any analytics rule that depends on it
- **Treating Lighthouse delegation as a data movement problem**: it delegates query access across tenants/workspaces — the underlying data never leaves its residency-compliant workspace
- **Building automation rules that only tag or comment on incidents**: this is SIEM with extra labels, not SOAR — the automation should take a bounded, reviewable containment action (like the playbook trigger in Part 5), not just organize the queue
- **Skipping automation rule ordering**: with multiple rules capable of matching the same incident, an unordered or misordered rule set can let a low-priority rule "claim" an incident before the high-severity rule gets to act on it

---

## Next Steps
- Continue to [Lab 4: Regulatory Compliance & Governance Architecture](lab-4-regulatory-compliance-governance-architecture.md) to design how this SOC's findings map into a formal compliance and audit program
- For the day-to-day SOC analyst workflow this architecture enables — triaging the incidents this lab's analytics rule generates — see [SC-200](../SC-200/README.md)
- For the general Log Analytics workspace topology decision this lab specializes for security operations, see [AZ-305 Lab 1: Identity, Governance & Monitoring Design](../AZ-305/lab-1-identity-governance-monitoring-design.md)
