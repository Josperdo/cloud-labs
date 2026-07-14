# Lab 4: Compute & Application Infrastructure Design

Check box if done: [ ]

## Overview
AZ-305 doesn't test whether you can deploy a web app — it tests whether you picked the right hosting model for the workload's shape before you deployed anything. This lab builds the decision matrix an architect actually defends in a design review (App Service vs. AKS vs. Container Apps vs. VM Scale Sets vs. Functions), then deploys the option that wins for a bursty, low-complexity internal web app: App Service with autoscale and deployment slots.

**Estimated time**: 60-75 minutes
**Cost**: ~$0-$3 (App Service Plan on a low tier, deleted at the end — S1 during the lab, or B1 if you skip the autoscale steps; nothing left running afterward)

---

## Scenario
You're the architect for a small internal tools team. Leadership wants a new internal web app — traffic is bursty: heavy during business hours, near-idle overnight and on weekends. The team has three developers, no Kubernetes experience, and no appetite to build container operations expertise for a single low-complexity app. Leadership's other hard requirement is zero-downtime releases — the last app they shipped took the whole team offline for ten minutes during a deploy, and that can't happen again. You need to choose a hosting model and design around both constraints before anyone writes a line of infrastructure code.

---

## Objectives
- Build a hosting model decision matrix (App Service, AKS, Container Apps, VM Scale Sets, Functions) scored against team skill, statefulness, traffic pattern, operational overhead, and cost
- Recommend and justify App Service for this workload, and name the trigger conditions that would flip the recommendation
- Deploy an App Service Plan on a tier that supports autoscale and configure a CPU-based autoscale rule
- Use deployment slots to ship a zero-downtime release, including swap-with-preview
- Understand where Container Apps, AKS, and Functions fit as the workload's shape changes

---

## Part 1: Design Decision — Hosting Model Selection

### Step 1: Build the Comparison Table
This is the step that matters most in this lab — the deployment in Part 2 only makes sense in light of this table. Score each option against the constraints that actually apply here.

| Criterion | App Service | AKS | Container Apps | VM Scale Sets | Azure Functions |
|---|---|---|---|---|---|
| **Team skill fit** | High — PaaS, no cluster ops | Low — needs Kubernetes expertise the team doesn't have | Medium — containers without cluster management | Medium — IaaS, patching/OS ownership | High — no infra to manage |
| **Statefulness** | Stateless web apps (state externalized to DB/cache) | Either, but adds complexity for stateful sets | Stateless, revision-based | Either — full OS control | Stateless, short-lived executions |
| **Traffic pattern fit** | Good — built-in autoscale rules on schedule or metric | Good but overkill — HPA + cluster autoscaler for one app | Good — scale-to-zero and per-revision scaling | Good, but slower scale-out (VM boot time) | Best for spiky/event-driven, not for a continuously-running UI |
| **Operational overhead** | Low — platform patches OS/runtime | High — you own the control plane, upgrades, networking, RBAC | Low-medium — still container build/push pipeline | High — you own OS patching, image management | Lowest — no servers, no scaling config |
| **Cost at this scale** | Low — single S1 instance handles business-hours load, scales down overnight | Higher — cluster overhead (node pool minimums) even for one small app | Low — pay only for active revisions/consumption | Medium — VMs bill even when scaled to a "floor" instance count | Lowest for true event-driven bursts; not a fit for a persistent web UI |

### Step 2: Recommendation and Rationale
**App Service** wins for this scenario:
- The team has zero Kubernetes experience — AKS would trade a hosting decision for a six-month operations learning curve the team doesn't need for one web app
- The workload is a straightforward stateful-free web app, not a microservices architecture — Container Apps' per-service scaling and revision model solve a problem this app doesn't have
- App Service's built-in **deployment slots** solve the zero-downtime requirement directly, with no extra tooling (blue/green via a slot swap, not a hand-rolled load balancer cutover)
- Autoscale rules handle the bursty business-hours traffic pattern natively — no cluster autoscaler, no VM boot-time lag

### Step 3: When You'd Pick Something Else
- **AKS**: the team already runs Kubernetes elsewhere, or this app is one of many microservices that need to share a cluster, common scheduling, and service mesh — see [General Lab 1 (AKS Fundamentals)](../General/lab-1-aks-fundamentals.md) in this repo for the alternative path and what that operational commitment looks like hands-on
- **Container Apps**: multiple small, independently-scaled services, each needing per-service scale-to-zero, without wanting to own a Kubernetes control plane
- **VM Scale Sets**: the workload needs OS-level customization, specific kernel modules, or licensing that PaaS doesn't support
- **Functions (Consumption plan)**: the workload is genuinely event-driven and sporadic (a nightly batch job, a webhook handler that fires a few times an hour) rather than a continuously-available web UI

---

## Part 2: Deploy App Service with Autoscale

### Step 4: Set Variables and Create the Resource Group
```bash
RG="rg-az305-lab4"
LOCATION="eastus"
PLAN="asp-lab4-webapp"
APP_NAME="app-lab4-webapp-<your-unique-suffix>"

az group create --name $RG --location $LOCATION
```

### Step 5: Create an App Service Plan on an Autoscale-Capable Tier
Autoscale rules based on metrics require **Standard (S-series)** or higher — the **Free (F1)**, **Shared (D1)**, and **Basic (B-series)** tiers don't support autoscale at all, only manual scale. Use **S1** for this lab.

```bash
az appservice plan create \
  --name $PLAN \
  --resource-group $RG \
  --location $LOCATION \
  --sku S1 \
  --is-linux
```

### Step 6: Create the Web App
```bash
az webapp create \
  --name $APP_NAME \
  --resource-group $RG \
  --plan $PLAN \
  --runtime "NODE:20-lts"
```

### Step 7: Create the Autoscale Setting
The autoscale setting targets the App Service Plan (not the app itself) — scaling adds or removes plan instances, and every app on the plan scales together.

```bash
PLAN_ID=$(az appservice plan show --name $PLAN --resource-group $RG --query id -o tsv)

az monitor autoscale create \
  --resource-group $RG \
  --resource $PLAN_ID \
  --name autoscale-lab4 \
  --min-count 1 \
  --max-count 4 \
  --count 1
```

### Step 8: Add Scale-Out and Scale-In Rules Based on CPU Percentage
Scale out when average CPU exceeds 70% over a 10-minute window; scale in when it drops below 25%. Different thresholds and directions prevent the same metric crossing from immediately triggering the opposite action.

```bash
# Scale out: +1 instance when CPU > 70%
az monitor autoscale rule create \
  --resource-group $RG \
  --autoscale-name autoscale-lab4 \
  --condition "Percentage CPU > 70 avg 10m" \
  --scale out 1

# Scale in: -1 instance when CPU < 25%
az monitor autoscale rule create \
  --resource-group $RG \
  --autoscale-name autoscale-lab4 \
  --condition "Percentage CPU < 25 avg 10m" \
  --scale in 1
```

**Validation checkpoint**: `az monitor autoscale show --name autoscale-lab4 --resource-group $RG --query "profiles[0].rules"` should list both rules. In the portal, **App Service Plan** → **Scale out (App Service plan)** → **Custom autoscale** should show the same two rules with min 1 / max 4 / default 1 instance counts.

---

## Part 3: Deployment Slots for Zero-Downtime Releases

### Step 9: Create a Staging Slot
Slots are full instances of the app running on the same plan (they share the plan's compute, so no extra plan cost — S1 supports up to 5 slots per app).

```bash
az webapp deployment slot create \
  --name $APP_NAME \
  --resource-group $RG \
  --slot staging
```

### Step 10: Deploy a Change to Staging Only
Deploy directly to the staging slot's own URL — production traffic never sees it until the swap.

```bash
az webapp deploy \
  --name $APP_NAME \
  --resource-group $RG \
  --slot staging \
  --src-path ./app.zip \
  --type zip
```

Browse to `https://$APP_NAME-staging.azurewebsites.net` and confirm the new version is live there, with production (`https://$APP_NAME.azurewebsites.net`) still serving the old version.

### Step 11: Swap Staging into Production
```bash
az webapp deployment slot swap \
  --name $APP_NAME \
  --resource-group $RG \
  --slot staging \
  --target-slot production
```

A swap exchanges the two slots' routing — the app that was in staging is now live in production, warmed up already (no cold start), with zero downtime. The old production version now sits in the staging slot, which doubles as your instant rollback path (swap again to revert).

### Step 12: Swap with Preview for App Setting Validation
A plain swap can break the app if a slot-specific app setting (e.g., a connection string marked "slot setting") doesn't carry over the way you expect. **Swap with preview** applies the target slot's configuration to the source slot *before* the swap completes, so you can validate the app actually starts correctly with production settings before traffic moves.

```bash
az webapp deployment slot swap \
  --name $APP_NAME \
  --resource-group $RG \
  --slot staging \
  --target-slot production \
  --action preview
```

Verify the staging slot is healthy with the previewed configuration, then complete the swap:

```bash
az webapp deployment slot swap \
  --name $APP_NAME \
  --resource-group $RG \
  --slot staging \
  --target-slot production \
  --action swap
```

**Validation checkpoint**: `az webapp deployment slot list --name $APP_NAME --resource-group $RG -o table` should show both slots. Confirm production is now serving the version that was in staging, and there was no gap where either the app returned errors or was unreachable.

---

## Part 4: Alternative Path Comparison Note

If this workload were instead a **microservices architecture** — several independently deployable services, each with its own scaling needs and release cadence — the App Service Plan model breaks down, because every app on a plan scales together as a unit. **Container Apps** becomes the better fit: each service is a separately scaled revision, with scale-to-zero for services that see no traffic overnight, and no App Service Plan sizing compromise across services with different load profiles. If the number of services grows large enough to need shared networking policy, service mesh, or the team already operates Kubernetes elsewhere, **AKS** is the next step up — at the cost of the operational overhead this lab's Part 1 table flagged as the reason to avoid it here.

If the workload were instead **sporadic and event-driven** — a webhook handler, a nightly batch job, a queue-triggered process that runs for seconds at a time — neither App Service nor Container Apps is the efficient choice, because both keep at least one instance provisioned to serve a continuously-available UI. **Azure Functions on a Consumption plan** bills per execution with true scale-to-zero between invocations, which is the right cost model for something that runs a few times a day rather than something people are actively browsing during business hours. The decision axis in both cases is the same one from Part 1's table: does the workload need a continuously-warm, stateless web UI (App Service), independently-scaled services without cluster ownership (Container Apps), full cluster control (AKS), or execution-triggered compute with no idle cost (Functions)?

---

## Cleanup

```bash
# Delete the autoscale setting
az monitor autoscale delete --name autoscale-lab4 --resource-group $RG

# Delete the staging deployment slot
az webapp deployment slot delete --name $APP_NAME --resource-group $RG --slot staging

# Delete the web app
az webapp delete --name $APP_NAME --resource-group $RG

# Delete the App Service Plan
az appservice plan delete --name $PLAN --resource-group $RG --yes

# Delete the resource group (catches anything left behind)
az group delete --name $RG --yes --no-wait
```

---

## Key Concepts

| Term | Definition |
|---|---|
| **App Service Plan tier** | Determines available features and capacity — Free/Shared/Basic tiers cap out at manual scale only; Standard and above unlock metric-based autoscale, slots, and custom domains with SSL |
| **Autoscale rules vs. scale-out** | "Scale-out" is the general concept (adding instances); autoscale *rules* are the specific metric thresholds and actions (e.g., CPU > 70% → +1 instance) that drive scale-out/scale-in automatically |
| **Deployment slot** | A separate, fully functional instance of an app sharing the same App Service Plan, used to stage a release before it's live in production |
| **Slot swap** | Exchanges routing between two slots instantly, with the target slot pre-warmed — avoids cold-start downtime a redeploy-in-place would cause |
| **Swap with preview** | A two-phase swap that applies target-slot config to the source slot first, letting you validate the app under production settings before traffic actually moves |
| **Slot-specific (sticky) app setting** | An app setting or connection string marked to stay with a *slot* rather than follow the swap — used for values that must differ between staging and production (e.g., a staging database connection string) |
| **PaaS vs. container vs. serverless compute** | PaaS (App Service) manages the runtime but you still think in "app instances"; containers (AKS/Container Apps) package the app with its dependencies for portability and per-service scaling; serverless (Functions) removes the instance concept entirely and bills per execution |

---

## Common Mistakes
- **Choosing a tier that doesn't support autoscale**: Free, Shared, and Basic tiers only offer manual scale — you won't discover this until the autoscale blade is greyed out or the CLI command errors
- **Forgetting slot-specific settings**: A connection string that isn't marked "slot setting" follows the swap and can point staging's database at production, or vice versa, after a swap
- **Autoscale rules with no cooldown/opposing thresholds**: Using the same threshold for scale-out and scale-in (e.g., both at 50%) causes flapping — the plan scales out and back in repeatedly as the metric hovers near the boundary
- **Treating deployment slots as free extra compute**: Slots share the plan's resources — a busy staging slot can compete with production for CPU/memory on the same instances
- **Skipping swap-with-preview for config-dependent releases**: A plain swap that surfaces a missing or misconfigured app setting only shows the problem *after* it's already live in production

---

## Next Steps
This lab builds on the VM and App Service basics from [AZ-104 Lab 2 (Compute)](../AZ-104/lab-2-compute.md). If the microservices/container path from Part 4 is the one your real workload needs, [General Lab 1 (AKS Fundamentals)](../General/lab-1-aks-fundamentals.md) is the hands-on alternative to this lab's App Service path. Continue the AZ-305 track with [Lab 3: Business Continuity Design](lab-3-business-continuity-design.md) or [Lab 5: Network Infrastructure Design](lab-5-network-infrastructure-design.md).
