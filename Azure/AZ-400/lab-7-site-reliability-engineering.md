# Lab 7: Site Reliability Engineering Practices

Check box if done: [ ]

## Overview
SRE turns "is the app healthy" from a gut feeling into a number you can gate a deploy on. This lab defines a Service Level Objective (SLO) and error budget for `az400-app`, builds an Azure Monitor alert that fires when the app is burning that budget too fast, and — the part that actually connects SRE to DevOps rather than leaving it as a dashboard nobody looks at — wires that alert into the Prod deployment gate from [Lab 3](lab-3-continuous-delivery-release-management.md) so a pipeline can automatically refuse to deploy while the service is already unhealthy.

**Estimated time**: 90 minutes
**Cost**: ~$0–$1

---

## Scenario
The team ships to Prod multiple times a day now, which is good — except twice last month a deploy went out while the service was already degraded from an unrelated issue, and the new deploy made triage harder by adding a second variable to an active incident. You're asked to define what "healthy" actually means in measurable terms, alert on it, and make the pipeline itself refuse to add change on top of an already-burning incident.

---

## Objectives
- Define an SLI, SLO, and error budget for `az400-app`
- Create an Azure Monitor alert rule and Action Group that fires on SLO-relevant signal (HTTP 5xx rate)
- Calculate error budget burn rate and understand fast-burn vs. slow-burn alerting
- Wire the alert as an automated release gate check in the `Prod` environment
- Write a minimal incident response runbook and confirm the Action Group notifies on-call correctly

---

## Part 1: Define the SLI, SLO, and Error Budget

### Step 1: Pick the SLI
The Service Level Indicator is the actual measured signal. For `az400-app`, use **HTTP success rate**: the percentage of requests returning a non-5xx status, measured over Application Insights request telemetry from [Lab 6](lab-6-instrumentation-feedback-strategy.md).

### Step 2: Set the SLO
| Term | Value | Meaning |
|------|-------|---------|
| **SLI** | HTTP success rate (non-5xx / total requests) | The thing you actually measure |
| **SLO** | 99.5% success rate, measured over a rolling 30 days | The target you're committing to |
| **Error budget** | 0.5% of requests allowed to fail per 30-day window | The "spending money" for risk — deploys, experiments, planned maintenance all draw from it |

At roughly 30 days (43,200 minutes), a 99.5% SLO allows about **216 minutes (~3h 36m) of full-downtime-equivalent budget per month** — or a proportionally larger amount of partial degradation. This number is what makes SRE different from "just monitor everything": it's an explicit, spendable budget, not an unstated expectation of zero failures.

### Step 3: Why This Matters for Release Decisions
An error budget isn't just a dashboard number — it's a policy lever. Common real-world rule: **when the budget for the current window is exhausted, non-emergency deploys pause** until the budget resets or the team explicitly accepts the risk. Part 3 automates a simplified version of that rule.

---

## Part 2: Azure Monitor Alert Rule and Action Group

### Step 4: Create an Action Group
```bash
az monitor action-group create \
  --name az400-oncall-ag \
  --resource-group az400-lab6-rg \
  --short-name az400oncall \
  --email-receivers name=primary-oncall email=<your-email>@example.com
```

### Step 5: Create a Metric Alert on 5xx Rate
```bash
APPINSIGHTS_ID=$(az monitor app-insights component show \
  --app az400-lab6-appinsights --resource-group az400-lab6-rg \
  --query id -o tsv)

az monitor metrics alert create \
  --name "az400-high-5xx-rate" \
  --resource-group az400-lab6-rg \
  --scopes "$APPINSIGHTS_ID" \
  --condition "avg requests/failed > 5" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 1 \
  --action az400-oncall-ag \
  --description "Fires when failed request count exceeds threshold — indicates active error-budget burn"
```

`--window-size 5m` with `--evaluation-frequency 1m` is a **fast-burn** alert configuration — it reacts quickly to a sharp spike, appropriate for something actively burning a meaningful fraction of the monthly budget right now. A complementary **slow-burn** alert (longer window, lower threshold) would catch a smaller, sustained degradation that a fast-burn window is too short to notice — worth adding a second rule for in a production setup; this lab implements the fast-burn case to keep scope manageable.

**Validation checkpoint**: `az monitor metrics alert show --name az400-high-5xx-rate --resource-group az400-lab6-rg --query "{enabled:enabled, severity:severity}"` confirms the rule is active.

---

## Part 3: Wire the Alert as an Automated Release Gate

### Step 6: Add an Invoke-REST-API Check to the `Prod` Environment
Azure DevOps Environments support **checks** beyond manual approval — an **Invoke REST API** check can query Azure Monitor's alert state and block the deployment job from starting if a Sev1 alert is currently active.

1. Portal → **Pipelines** → **Environments** → **Prod** → **Approvals and checks** → **+** → **Invoke REST API**
2. **Connection type**: `Azure Resource Manager` (reuse `az400-lab3-sc-federated`)
3. **Method**: `GET`
4. **URL**:
   ```
   https://management.azure.com/subscriptions/<your-subscription-id>/resourceGroups/az400-lab6-rg/providers/Microsoft.Insights/metricAlerts/az400-high-5xx-rate?api-version=2018-03-01
   ```
5. **Completion event**: `Callback` is unnecessary here — use `apiResponse` and add a **Success criteria** expression:
   ```
   eq(root['properties']['isEnabled'], true) && ne(root['properties']['status'], 'Fired')
   ```
   (exact field name may be `status` or require checking `properties.description` depending on API version — validate the actual response shape with the `az monitor metrics alert show` call from Step 5 and adjust the JSONPath accordingly)
6. **Time between evaluations**: `1 minute`, **Timeout**: `15 minutes`

This means: when the pipeline reaches the `DeployProd` stage, before the manual approval from Lab 3 even prompts anyone, Azure DevOps polls the alert's current state — if it's actively firing, the check fails and the stage never proceeds to request approval at all.

### Step 7: Simulate a Burn and Confirm the Gate Blocks
Generate synthetic 5xx traffic against the App Service from Lab 3 to trip the alert (adjust to however the app actually returns errors, or use a throwaway endpoint):
```bash
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code}\n" https://az400-app-<unique-suffix>.azurewebsites.net/force-error
done
```

**Validation checkpoint**: trigger the release pipeline while the alert is active. Confirm the `DeployProd` stage's Invoke REST API check fails or times out, and the stage never reaches the manual approval step — the pipeline is refusing to add a deploy on top of an active incident, exactly the scenario from the lab's opening.

---

## Part 4: Incident Response Basics

### Step 8: Write a Minimal Runbook
Keep this in the repo (`docs/runbooks/high-5xx-rate.md`) so it's versioned alongside the code it applies to:
```markdown
# Runbook: az400-high-5xx-rate

## Trigger
Azure Monitor alert `az400-high-5xx-rate` fires when failed request count exceeds 5 in a 5-minute window.

## First response (5 minutes)
1. Acknowledge in the Action Group notification / on-call channel.
2. Check the Application Insights release annotation timeline (Lab 6) — did a deploy happen in the last 30 minutes?
3. If yes: consider the slot swap rollback from Lab 3, Step 8.
4. If no recent deploy: check Azure Monitor for upstream dependency failures (database, external API).

## Escalation
If unresolved after 15 minutes, page secondary on-call and open a Sev2 incident.

## Resolution
Confirm 5xx rate back under threshold for 10 consecutive minutes before closing.

## Postmortem
Required for any incident that consumed more than 10% of the monthly error budget in a single event.
```

### Step 9: Confirm the Action Group Actually Notifies
```bash
az monitor action-group test-notifications create \
  --action-group-name az400-oncall-ag \
  --resource-group az400-lab6-rg \
  --alert-type servicehealth \
  --receivers primary-oncall
```
**Validation checkpoint**: confirm the test notification arrives at the configured email — an Action Group that's never been test-fired is a common reason real incidents go unnoticed until someone happens to check a dashboard.

---

## Part 5: Cleanup

```bash
az monitor metrics alert delete --name az400-high-5xx-rate --resource-group az400-lab6-rg
az monitor action-group delete --name az400-oncall-ag --resource-group az400-lab6-rg
```
Remove the Invoke REST API check from the `Prod` environment via **Approvals and checks** if you're not continuing to the next lab with the same environment.

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **SLI/SLO/error budget definitions** | Converts "is it healthy" from a subjective feeling into a number with an agreed threshold |
| **Fast-burn vs. slow-burn alert windows** | Matches alert sensitivity to how quickly a given failure mode actually consumes the budget |
| **Automated release gate on live alert state** | Stops a pipeline from making an active incident worse, without needing a human to remember to check first |
| **Versioned runbooks** | Keeps incident response knowledge next to the code, reviewed and updated the same way as everything else in the repo |
| **Testing the Action Group before you need it** | An alert that's configured but never verified to actually notify anyone is a false sense of security |

---

## Common Mistakes to Avoid
- **Setting an SLO without an error budget policy behind it**: 99.5% is just a number until the team agrees what happens when it's exhausted (this lab: pipeline gate)
- **Only configuring a fast-burn alert**: catches spikes but misses a slow, sustained degradation that never crosses the fast-burn threshold in any single 5-minute window
- **Wiring the release gate to block on *any* alert history instead of *currently firing* state**: you want "is it broken right now," not "was it ever broken," or the gate never clears
- **Skipping the Action Group test notification**: the alert rule can be perfectly configured and still never reach anyone if the receiver email/webhook itself is wrong
- **Treating the runbook as a one-time document**: update it after every real incident it's used for — a runbook that doesn't reflect the last incident's lessons is stale by definition

---

## Next Steps
- [Lab 8: Package & Dependency Management](lab-8-package-dependency-management.md) closes out the track by tying source control, CI, CD, security, IaC, instrumentation, and this SRE gate into one end-to-end pipeline picture
- Add a slow-burn companion alert (longer window, lower threshold) alongside the fast-burn rule built here for more complete budget-burn coverage
- Cross-reference [AZ-500 Lab 4](../AZ-500/lab-4-security-ops-automation.md) for extending the Action Group into a Sentinel-driven incident workflow instead of email-only notification
