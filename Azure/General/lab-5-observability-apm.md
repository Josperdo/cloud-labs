# Lab 5: Observability & Application Performance Monitoring

Check box if done: [ ]

## Overview
AZ-104 Lab 5 covered infrastructure monitoring — Log Analytics, diagnostic settings, and KQL against activity/resource logs. That tells you the box is healthy; it says nothing about whether the application running on it is actually serving requests correctly. This lab covers application-level observability: distributed tracing, dependency mapping, and SLO-style alerting. This is what "APM" means in a job posting, and it's what production on-call actually leans on day to day — infra metrics tell you the server is up, APM tells you whether users are having a bad time.

**Estimated time**: 60-75 minutes
**Cost**: ~$0-$1 (Application Insights free tier data allowance covers this lab; a minimal App Service to host the instrumented app)

---

## Scenario
You inherited a production app that's "monitored" in the sense that someone set up CPU and memory alerts on the App Service Plan two years ago. Nobody has touched observability since. Last week a customer complained about slow checkout, and the on-call engineer had nothing to look at — CPU was fine, memory was fine, and there was zero visibility into request latency, which dependency call was slow, or where time was actually being spent inside a request. You're fixing that today: instrumenting the app with Application Insights, building a workbook that visualizes the request path, and creating an alert that fires on a real SLO breach instead of a proxy metric that doesn't correlate with user experience.

---

## Objectives
- Create a workspace-based Application Insights resource
- Instrument an App Service for auto-collection of requests, dependencies, and exceptions
- Read a distributed trace and identify where time is spent across a request's dependency chain
- Write APM-specific KQL against the `requests` and `dependencies` tables
- Build an Azure Monitor workbook visualizing request rate, failure rate, and latency
- Create an SLO-style alert (failed request rate / p95 latency) wired to an action group

---

## Part 1: Create an Application Insights Resource

### Step 1: Create It Workspace-Based
Classic (non-workspace) Application Insights is on a deprecation path — workspace-based is the current default Microsoft recommends. It stores telemetry inside a Log Analytics workspace instead of a proprietary store, so you query it with the same KQL engine and can join app telemetry against the infra logs from AZ-104 Lab 5 if both live in the same workspace.

```bash
# Resource group + Log Analytics workspace (skip if reusing one from a prior lab)
az group create --name observability-lab-rg --location eastus

az monitor log-analytics workspace create \
  --resource-group observability-lab-rg \
  --workspace-name observability-logs \
  --location eastus

WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group observability-lab-rg \
  --workspace-name observability-logs \
  --query id -o tsv)

# Workspace-based Application Insights, linked to that workspace
az monitor app-insights component create \
  --app app-insights-lab \
  --location eastus \
  --resource-group observability-lab-rg \
  --workspace "$WORKSPACE_ID" \
  --application-type web
```

**Validation checkpoint**:
```bash
az monitor app-insights component show \
  --app app-insights-lab \
  --resource-group observability-lab-rg \
  --query "{name:name, workspaceId:workspaceResourceId, connectionString:connectionString}" -o table
```
Confirm `workspaceId` is populated (not null) — that's what distinguishes workspace-based from classic. Note the `connectionString` value; you'll need it in Part 2.

---

## Part 2: Instrument an App (Auto-Instrumentation via App Service)

### Step 2: Deploy a Minimal App Service
Any App Service running a supported runtime (.NET, Java, Node.js, Python) works — the point is the instrumentation step, not the app itself.

```bash
az appservice plan create \
  --name observability-lab-plan \
  --resource-group observability-lab-rg \
  --sku B1 \
  --is-linux

az webapp create \
  --name app-lab-<yourinitials><randomnumber> \
  --resource-group observability-lab-rg \
  --plan observability-lab-plan \
  --runtime "NODE:20-lts"
```

### Step 3: Connect the App Service to Application Insights
For supported runtimes, App Service can auto-instrument the app — no code changes, no SDK, no redeploy — by injecting an agent at the platform level that hooks the runtime and collects requests, dependencies, and exceptions automatically. (The portal does the same thing via the **Application Insights** blade under **Settings** → toggle it on.)

```bash
CONNECTION_STRING=$(az monitor app-insights component show \
  --app app-insights-lab \
  --resource-group observability-lab-rg \
  --query connectionString -o tsv)

az webapp config appsettings set \
  --name app-lab-<yourinitials><randomnumber> \
  --resource-group observability-lab-rg \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="$CONNECTION_STRING"
```

The connection string is what auto-instrumentation looks for; setting it as an app setting is the entire integration for supported runtimes.

**When auto-instrumentation isn't enough**: it covers the common cases (incoming HTTP requests, outbound HTTP/SQL/Redis calls, unhandled exceptions) but won't capture custom business events or spans around code you specifically want traced (e.g., "how long did the pricing calculation take"). For that, add the SDK directly (`applicationinsights` npm package, `Microsoft.ApplicationInsights` NuGet package, etc.) and call `trackEvent`/`trackMetric`, or wrap a block as a custom dependency. Auto-instrumentation and SDK instrumentation aren't mutually exclusive — most production apps use both.

### Step 4: Generate Traffic and Validate
1. Restart the App Service so the setting takes effect: `az webapp restart --name app-lab-<yourinitials><randomnumber> --resource-group observability-lab-rg`
2. Hit the app's URL several times (`curl` in a loop, or just refresh the browser 5-10 times) to generate request telemetry
3. Wait 2-3 minutes for telemetry to land — Application Insights ingestion isn't instant

**Validation checkpoint**: In the portal, open the Application Insights resource → **Investigate** → **Performance**. You should see entries under the `requests` operation list with real durations. If nothing shows up after 5 minutes, confirm the app setting saved correctly and that the app actually received traffic (a cold-started App Service can take a minute to respond to the first request).

---

## Part 3: Read a Distributed Trace / Dependency View

### Step 5: Open a Transaction End-to-End
1. Application Insights → **Investigate** → **Transaction search**
2. Click any request in the list to open its **end-to-end transaction detail** view
3. This shows a waterfall: the incoming request at the top, and every dependency call it made underneath — outbound HTTP calls, database queries, cache lookups — each with its own duration, nested by when it started relative to the parent request

This is the actual value of distributed tracing: a raw request duration of "480ms" tells you nothing about *why* it was 480ms. The trace tells you it was 40ms of app code, a 380ms call to an external payment API, and 60ms writing to a database — so you know where to look instead of guessing. "Checkout is slow" becomes "the payment gateway dependency is slow," pointed at a specific external system.

### Step 6: Query Dependencies Directly
APM-specific KQL against telemetry tables that don't exist in an infra-only workspace (AZ-104 Lab 5 queried `AzureActivity` and `StorageBlobLogs`; here it's `requests` and `dependencies`, which only populate once an app is instrumented).

**p95 request duration by operation, last hour:**
```kusto
requests
| where timestamp > ago(1h)
| summarize p95Duration = percentile(duration, 95) by name
| sort by p95Duration desc
```

**Slowest dependency types — is it the database, an external API, or something else?**
```kusto
dependencies
| where timestamp > ago(1h)
| summarize avgDuration = avg(duration), count() by type, target
| sort by avgDuration desc
```

Both queries key off `operation_Id` under the hood — the correlation ID that ties every dependency call back to the request that triggered it, which is the mechanism distributed tracing actually runs on. To pull it directly: `requests | where success == false | project operation_Id, name, resultCode | join kind=inner (dependencies | project operation_Id, depName=name, depTarget=target) on operation_Id`.

---

## Part 4: Build an Azure Monitor Workbook

### Step 7: Create a Workbook with Request-Health Tiles
1. Application Insights resource → **Workbooks** → **+ New**
2. Add a text tile with a title: `## App Health — <app name>`

**Tile 1 — Request rate:**
```kusto
requests
| where timestamp > ago(4h)
| summarize RequestsPerMin = count() by bin(timestamp, 1m)
| render timechart
```

**Tile 2 — Failed request rate (%):**
```kusto
requests
| where timestamp > ago(4h)
| summarize Total = count(), Failed = countif(success == false) by bin(timestamp, 5m)
| extend FailureRatePct = round(100.0 * Failed / Total, 2)
| project timestamp, FailureRatePct
| render timechart
```

**Tile 3 — p95 duration:** reuse the p95-by-operation query from Part 3, Step 6, dropping the `by name` and adding `by bin(timestamp, 5m)` instead so it renders as a trend over the selected window rather than a single current value.

3. For each tile: **+ Add** → **Add query**, paste the KQL, set visualization to `Line chart`, **Run Query** → **Done Editing**
4. **Done Editing** (top of workbook) → **Save** → name it `App Health Dashboard`, save into `observability-lab-rg`

**Validation checkpoint**: All three tiles render data (not empty charts) — if a tile is empty, confirm the time range covers the traffic you generated in Part 2, Step 4.

---

## Part 5: SLO-Style Alerting

### Step 8: Why Not Just Alert on CPU
CPU > 80% doesn't tell you whether users are getting errors or slow responses — an app can peg CPU while serving everything fine, or sit at 20% CPU while every third request times out on a stuck downstream dependency. An SLO-style alert measures what users actually experience: failure rate and latency. That's the signal that correlates with "is the app broken."

### Step 9: Create the Action Group
```bash
az monitor action-group create \
  --resource-group observability-lab-rg \
  --name app-oncall-ag \
  --short-name apponcall \
  --action email oncall-email <your-email>
```

### Step 10: Create a Log-Based Alert on Failed Request Rate
```bash
APP_INSIGHTS_ID=$(az monitor app-insights component show \
  --app app-insights-lab \
  --resource-group observability-lab-rg \
  --query id -o tsv)

az monitor scheduled-query create \
  --name "high-failed-request-rate" \
  --resource-group observability-lab-rg \
  --scopes "$APP_INSIGHTS_ID" \
  --condition "count 'FailureRatePct' > 5" \
  --condition-query FailureRatePct='requests | summarize Total = count(), Failed = countif(success == false) by bin(timestamp, 5m) | extend FailureRatePct = 100.0 * Failed / Total | where FailureRatePct > 5' \
  --window-size 5m \
  --evaluation-frequency 5m \
  --severity 2 \
  --action-groups app-oncall-ag \
  --description "Failed request rate exceeded 5% over a 5-minute window"
```

### Step 11: A Latency-Based Variant
The same pattern covers p95 latency instead of failure rate — swap the condition query and threshold:
```bash
--condition "count 'P95' > 2000" \
--condition-query P95='requests | summarize P95 = percentile(duration, 95) by bin(timestamp, 5m) | where P95 > 2000' \
--description "p95 request latency exceeded 2s over a 5-minute window"
```
Adjust thresholds (`5%`, `2000ms`) to whatever your actual SLO commits to — the point of an SLO-style alert is that the threshold traces back to a stated user-facing target, not an arbitrary round number. In practice you'd run both: one on failure rate, one on latency, since a service can breach either independently.

**Validation checkpoint**: `az monitor scheduled-query list --resource-group observability-lab-rg -o table` should list the rule as `enabled: true`. In the portal, **Application Insights** → **Alerts** → **Alert rules** should show it with `app-oncall-ag` attached under Actions — an alert rule with no action group notifies nobody, same failure mode called out in AZ-104 Lab 5.

---

## Cleanup

```bash
# Alert rule(s) — repeat for each scheduled-query rule you created
az monitor scheduled-query delete --name "high-failed-request-rate" --resource-group observability-lab-rg --yes

# Action group
az monitor action-group delete --name app-oncall-ag --resource-group observability-lab-rg

# Workbook — delete via portal (Application Insights > Workbooks > select > Delete),
# or via Resource Graph / az resource delete if you captured its resource ID

# App Service + plan
az webapp delete --name app-lab-<yourinitials><randomnumber> --resource-group observability-lab-rg
az appservice plan delete --name observability-lab-plan --resource-group observability-lab-rg --yes

# Application Insights + Log Analytics workspace
az monitor app-insights component delete --app app-insights-lab --resource-group observability-lab-rg
az monitor log-analytics workspace delete --resource-group observability-lab-rg --workspace-name observability-logs --yes
```
Or skip the individual deletes and remove everything at once: `az group delete --name observability-lab-rg --yes --no-wait`.

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Application Insights** | The APM layer of Azure Monitor — collects request, dependency, exception, and trace telemetry from a running application, as opposed to Log Analytics/infra logs which cover the platform underneath it |
| **Workspace-based mode** | Stores Application Insights telemetry inside a Log Analytics workspace instead of a proprietary store; current recommended mode, enables joining app and infra telemetry in one query |
| **Distributed tracing** | Correlating every dependency call triggered by a single incoming request (via `operation_Id`) so you can see the full call chain and where time was spent, not just the front-door duration |
| **Dependency tracking** | Auto-captured telemetry for outbound calls the app makes — HTTP calls, SQL queries, cache lookups — each with its own duration and success/failure status |
| **Auto-instrumentation** | Platform-level telemetry collection enabled by setting `APPLICATIONINSIGHTS_CONNECTION_STRING`, with no SDK or code change required, for supported App Service runtimes |
| **Workbook** | A shareable, interactive Azure Monitor dashboard combining KQL queries, metrics, and text into one view |
| **SLO-style alert** | An alert on a derived, user-facing condition (failure rate, p95 latency) tied to a stated service-level target, rather than a raw infra metric that doesn't reliably correlate with user experience |
| **Action group** | The notification target (email, SMS, webhook, function, etc.) an alert rule fires into — without one, an alert rule is silent |

---

## Common Mistakes
- **Alerting only on infra metrics** (CPU, memory) and missing real user-facing degradation — a server can be "healthy" by every infra metric while every third request fails or times out
- **Not correlating traces across service boundaries** — reading a single service's logs in isolation instead of following `operation_Id` through the full dependency chain, missing where the actual bottleneck is
- **Workbooks left personal/un-shared** when they should be saved to a shared resource group and pinned somewhere the team actually looks, not left as a private workbook only the creator can find
- **Sampling silently dropping telemetry at high volume** — Application Insights adaptive sampling reduces ingestion cost by dropping a percentage of telemetry once volume crosses a threshold; a rare error can get sampled out entirely and never show up in queries or alerts if you don't know sampling is active
- **Classic Application Insights on a deprecation path** — provisioning new classic (non-workspace-based) resources instead of workspace-based, then having to migrate later

---

## Next Steps
This lab extends the infra-monitoring foundation from [AZ-104 Lab 5](../AZ-104/lab-5-monitoring.md) into application-level observability. For security-relevant signal — threat detection, incident response, SIEM correlation — that's a different tool for a different question, covered in [AZ-500 Lab 4](../AZ-500/lab-4-security-ops-automation.md) (Sentinel); APM tells you the app is slow, Sentinel tells you the app is under attack. Next up: [Lab 6: Cost Management & FinOps](lab-6-cost-management-finops.md).
