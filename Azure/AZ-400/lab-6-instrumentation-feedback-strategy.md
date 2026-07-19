# Lab 6: Instrumentation & Feedback Strategy

Check box if done: [ ]

## Overview
A deploy that nobody can correlate with a performance change is a deploy you can't learn from. This lab wires Application Insights into the release pipeline from [Lab 3](lab-3-continuous-delivery-release-management.md) two ways: a **release annotation** stamped onto the App Insights timeline every time a deploy completes, so a latency spike at 2:14pm is instantly checkable against "did we just deploy," and **feature flags** via Azure App Configuration, which decouple *deploying* code from *releasing* a feature — the code can ship dark and get turned on for 10% of users without a second deployment.

**Estimated time**: 75 minutes
**Cost**: ~$0–$1 (Application Insights and App Configuration both have generous free tiers; costs stay near zero at lab scale)

---

## Scenario
The team's current process is "deploy, then wait and see if anyone complains." You're asked to close that feedback loop: every deploy should leave a visible marker on the monitoring timeline, and the next risky feature should ship behind a flag so it can be enabled for a small percentage of traffic first — decoupling the deploy (code reaches production) from the release (users actually see the new behavior).

---

## Objectives
- Provision Application Insights and Azure App Configuration
- Add a release annotation step to the pipeline that fires on every successful Prod deploy
- Create a feature flag and target it at a percentage of traffic
- Wire the app (conceptually) to check the flag at runtime via the App Configuration feature management SDK pattern
- Verify a deploy annotation lines up with a metric change on the Application Insights timeline

---

## Part 1: Provision Application Insights and App Configuration

### Step 1: Create Both Resources
```bash
az group create --name az400-lab6-rg --location eastus2

az monitor app-insights component create \
  --app az400-lab6-appinsights \
  --location eastus2 \
  --resource-group az400-lab6-rg \
  --application-type web

az appconfig create \
  --name az400-lab6-appconfig-<unique-suffix> \
  --resource-group az400-lab6-rg \
  --location eastus2 \
  --sku free
```

### Step 2: Capture the Connection Strings
```bash
APPINSIGHTS_KEY=$(az monitor app-insights component show \
  --app az400-lab6-appinsights --resource-group az400-lab6-rg \
  --query instrumentationKey -o tsv)

APPCONFIG_CONN=$(az appconfig credential list \
  --name az400-lab6-appconfig-<unique-suffix> --resource-group az400-lab6-rg \
  --query "[?name=='Primary'].connectionString" -o tsv)
```
Store both as **pipeline secret variables** (Library → Variable group, marked secret) rather than plain YAML variables — neither should appear in a build log.

---

## Part 2: Release Annotations

### Step 3: Add an Annotation Step After the Prod Deploy
Application Insights exposes an annotations REST API; the cleanest pipeline integration is a script step using the Azure CLI's REST passthrough, run as the last step of the `DeployProd` stage from Lab 3:
```yaml
                - task: AzureCLI@2
                  displayName: 'Create Application Insights release annotation'
                  inputs:
                    azureSubscription: 'az400-lab3-sc-federated'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      APPINSIGHTS_ID=$(az monitor app-insights component show \
                        --app az400-lab6-appinsights --resource-group az400-lab6-rg \
                        --query id -o tsv)

                      az rest --method put \
                        --uri "https://management.azure.com${APPINSIGHTS_ID}/Annotations?api-version=2015-05-01" \
                        --body "{
                          \"AnnotationName\": \"Release $(Build.BuildNumber)\",
                          \"EventTime\": \"$(date -u +%Y-%m-%dT%H:%M:%S.000Z)\",
                          \"Category\": \"Deployment\",
                          \"Properties\": \"{\\\"Label\\\":\\\"Deployment\\\",\\\"ReleaseName\\\":\\\"$(Build.BuildNumber)\\\"}\"
                        }"
```
Every field here is deliberate: `Category: Deployment` is what makes App Insights render it as a deploy marker (a vertical line) rather than a generic note, and `$(Build.BuildNumber)` ties the marker back to the exact pipeline run that produced it — click the annotation, and you know which build to `git log` against.

### Step 4: Verify the Annotation Landed
```bash
az rest --method get \
  --uri "https://management.azure.com$(az monitor app-insights component show --app az400-lab6-appinsights --resource-group az400-lab6-rg --query id -o tsv)/Annotations?api-version=2015-05-01" \
  --query "[].{name:AnnotationName, time:EventTime}" -o table
```

**Validation checkpoint**: portal → **Application Insights** → **Metrics** → any chart with a time axis → the deploy should appear as a labeled vertical marker. This is the single fastest way to answer "did we just cause this" during an incident review.

---

## Part 3: Feature Flags with Azure App Configuration

### Step 5: Create a Feature Flag
```bash
az appconfig feature set \
  --name az400-lab6-appconfig-<unique-suffix> \
  --feature new-checkout-flow \
  --label production \
  --yes
```
By default a newly created flag is **off** — nothing changes for any user until you explicitly enable it.

### Step 6: Target a Percentage of Traffic Instead of a Hard On/Off
```bash
az appconfig feature filter add \
  --name az400-lab6-appconfig-<unique-suffix> \
  --feature new-checkout-flow \
  --label production \
  --filter-name Microsoft.Percentage \
  --filter-parameters Value=10

az appconfig feature enable \
  --name az400-lab6-appconfig-<unique-suffix> \
  --feature new-checkout-flow \
  --label production
```
This is the decoupling in practice: the code for `new-checkout-flow` is already deployed to 100% of Prod instances from the last pipeline run, but only ~10% of requests actually see the new behavior, chosen by the `Microsoft.Percentage` filter's consistent hashing — the same user gets the same evaluation result across requests, not a coin-flip per request.

### Step 7: How the App Checks the Flag at Runtime
Conceptually (Node.js example using `@azure/app-configuration` + `@microsoft/feature-management`):
```javascript
const { AppConfigurationClient } = require("@azure/app-configuration");
const { FeatureManager, ConfigurationMapFeatureFlagProvider } = require("@microsoft/feature-management");

const client = new AppConfigurationClient(process.env.APPCONFIG_CONNECTION_STRING);
const featureFlags = new Map();
for await (const flag of client.listConfigurationSettings({ keyFilter: ".appconfig.featureflag/*" })) {
  featureFlags.set(flag.key, JSON.parse(flag.value));
}
const featureManager = new FeatureManager(new ConfigurationMapFeatureFlagProvider(featureFlags));

if (await featureManager.isEnabled("new-checkout-flow", { userId: request.userId })) {
  // new behavior
} else {
  // existing behavior
}
```
The important point for AZ-400 isn't the SDK syntax — it's that the flag check happens **at runtime, per request**, with no redeploy required to change who sees what. Widening the rollout from 10% to 50% to 100% is an `az appconfig feature filter update` call, not a pipeline run.

### Step 8: Widen the Rollout
```bash
az appconfig feature filter update \
  --name az400-lab6-appconfig-<unique-suffix> \
  --feature new-checkout-flow \
  --label production \
  --index 0 \
  --filter-parameters Value=50
```

**Validation checkpoint**: `az appconfig feature show --name az400-lab6-appconfig-<unique-suffix> --feature new-checkout-flow --label production --query "conditions.client_filters"` shows the updated `Value: 50` — confirm this change required no pipeline run and no new deployment.

---

## Part 4: Cleanup

```bash
az group delete --name az400-lab6-rg --yes --no-wait
```

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **Release annotations** | Turns "did we just deploy" from a Slack-history search into a one-click check on the metrics timeline |
| **Deploy vs. release decoupling** | Feature flags let code ship to 100% of instances while only a controlled fraction of users see the new behavior |
| **Percentage-based feature filters** | Gradual, reversible exposure without a redeploy — the instrumentation-layer analog of the canary deployment strategy from Lab 3 |
| **Runtime flag evaluation** | Rollback for a bad feature becomes a config change (seconds), not a redeploy (minutes) |
| **Tagging annotations with the build number** | Makes every timeline marker traceable back to an exact commit/pipeline run, not just "a deploy happened sometime" |

---

## Common Mistakes to Avoid
- **Skipping the annotation step because "we can just check the pipeline history"**: that requires already suspecting a deploy caused the issue — annotations put the deploy directly on the same chart as the symptom, so you don't have to suspect it first
- **Treating feature flags as permanent if/else branches**: a flag left in the codebase after full rollout is technical debt — remove the flag and the old code path once it's fully released
- **Using a hard on/off flag when a gradual rollout was the actual goal**: `Microsoft.Percentage` (or a custom targeting filter) exists specifically so "10% of users" doesn't require hand-rolled bucketing logic in the app
- **Storing the App Configuration connection string in plain pipeline variables**: mark it secret in a variable group, same as the Application Insights key
- **Forgetting `Category: Deployment` on the annotation**: without it, App Insights treats the annotation as a generic note rather than rendering the deploy marker style that makes it visually distinct on a chart

---

## Next Steps
- [Lab 7: Site Reliability Engineering Practices](lab-7-site-reliability-engineering.md) uses the same Application Insights resource to define SLOs and wire an automated release gate on error-budget burn
- Revisit [Lab 3](lab-3-continuous-delivery-release-management.md) to add the annotation step to the DeployDev and DeployQA stages too, so pre-Prod deploys are equally traceable
- [Azure/General Lab 5: Observability & APM](../General/lab-5-observability-apm.md) goes deeper on Application Insights distributed tracing and workbooks if this lab's annotation-only integration left you wanting more
