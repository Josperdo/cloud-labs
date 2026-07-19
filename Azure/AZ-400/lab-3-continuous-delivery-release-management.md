# Lab 3: Continuous Delivery & Release Management

Check box if done: [ ]

## Overview
A green CI build sitting in Azure Artifacts isn't a release — it's potential energy. This lab turns the artifact published in [Lab 2](lab-2-continuous-integration-pipelines.md) into an actual deployment pipeline: three environments (Dev → QA → Prod), each defined as an Azure DevOps **Environment** so you get deployment history and approvals for free, a manual approval gate in front of Prod, and a blue-green deployment to Azure App Service using deployment slots so a bad release can be rolled back by swapping a slot pointer instead of redeploying.

**Estimated time**: 90–105 minutes
**Cost**: ~$1–$3 (App Service **Standard S1** tier, required for deployment slots — delete it same-session; see Cleanup)

---

## Scenario
`az400-app` builds cleanly and merges are gated by CI, but there's still no path from "merged to `main`" to "running somewhere a user can hit it." You're asked to build that path with three properties the team explicitly wants: an automatic promotion through Dev and QA so bugs are caught early, a human approval before anything touches Prod, and zero-downtime Prod deploys — a bad release should be a slot swap away from rollback, not a multi-minute redeploy.

---

## Part 1: Provision the Target App Service

### Step 1: Create the App Service Plan and Web App
Standard tier is required for slots — Free/Basic tiers don't support them.
```bash
az group create --name az400-lab3-rg --location eastus2

az appservice plan create \
  --name az400-lab3-plan \
  --resource-group az400-lab3-rg \
  --sku S1 \
  --is-linux

az webapp create \
  --name az400-app-<unique-suffix> \
  --resource-group az400-lab3-rg \
  --plan az400-lab3-plan \
  --runtime "NODE:20-lts"
```

### Step 2: Create a Staging Slot for Blue-Green Deploys
```bash
az webapp deployment slot create \
  --name az400-app-<unique-suffix> \
  --resource-group az400-lab3-rg \
  --slot staging
```

**Expected result**: `az webapp deployment slot list --name az400-app-<unique-suffix> --resource-group az400-lab3-rg -o table` shows both `production` and `staging`.

### Step 3: Create a Service Connection
Portal → **Project settings** → **Service connections** → **New service connection** → **Azure Resource Manager** → **Workload identity federation (automatic)** → scope to `az400-lab3-rg` → name it `az400-lab3-sc`. Workload identity federation avoids a stored client secret in the service connection — the same OIDC-style trust pattern covered in depth in [Lab 4](lab-4-pipeline-security-compliance.md) and [Azure/General Lab 3](../General/lab-3-cicd-oidc.md).

---

## Part 2: Define Environments and the Prod Approval Gate

### Step 4: Create Three Environments
```bash
az pipelines environment create --name Dev --project az400-devops-lab
az pipelines environment create --name QA --project az400-devops-lab
az pipelines environment create --name Prod --project az400-devops-lab
```

### Step 5: Add a Manual Approval Check on Prod
Environment approvals aren't exposed via `az pipelines environment` yet — configure via the portal:
1. **Pipelines** → **Environments** → **Prod** → **⋮** → **Approvals and checks** → **+** → **Approvals**
2. **Approvers**: add yourself or a release-manager group
3. **Timeout**: `12 hours`
4. **Instructions**: `Confirm QA sign-off before approving a Prod deploy`
5. **Create**

**Why an environment-level gate instead of a YAML `if` condition**: the approval is enforced by Azure DevOps itself, shows up in deployment history with who approved and when, and can't be bypassed by editing the YAML in the same PR that needs the approval.

---

## Part 3: Multi-Stage Deploy Pipeline

### Step 6: Extend `azure-pipelines.yml` with Deploy Stages
Append to the file from Lab 2, after the `Publish` stage:
```yaml
  - stage: DeployDev
    displayName: 'Deploy to Dev'
    dependsOn: Publish
    condition: succeeded()
    jobs:
      - deployment: DeployDev
        environment: 'Dev'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: az400-app-drop
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'az400-lab3-sc'
                    appType: 'webAppLinux'
                    appName: 'az400-app-<unique-suffix>'
                    slotName: 'production'
                    package: '$(Pipeline.Workspace)/az400-app-drop'

  - stage: DeployQA
    displayName: 'Deploy to QA'
    dependsOn: DeployDev
    condition: succeeded()
    jobs:
      - deployment: DeployQA
        environment: 'QA'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: az400-app-drop
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'az400-lab3-sc'
                    appType: 'webAppLinux'
                    appName: 'az400-app-<unique-suffix>'
                    slotName: 'production'
                    package: '$(Pipeline.Workspace)/az400-app-drop'

  - stage: DeployProd
    displayName: 'Deploy to Prod (blue-green)'
    dependsOn: DeployQA
    condition: succeeded()
    jobs:
      - deployment: DeployProdSlot
        environment: 'Prod'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: az400-app-drop
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'az400-lab3-sc'
                    appType: 'webAppLinux'
                    appName: 'az400-app-<unique-suffix>'
                    slotName: 'staging'
                    package: '$(Pipeline.Workspace)/az400-app-drop'
                - task: AzureAppServiceManage@0
                  inputs:
                    azureSubscription: 'az400-lab3-sc'
                    action: 'Swap Slots'
                    webAppName: 'az400-app-<unique-suffix>'
                    resourceGroupName: 'az400-lab3-rg'
                    sourceSlot: 'staging'
                    targetSlot: 'production'
```

Note the difference in the Prod stage: Dev and QA deploy straight to `production` (they're disposable, lower-risk environments where speed matters more than zero-downtime). Prod deploys to the **`staging` slot first**, and only swaps into `production` after the deploy succeeds — the swap itself is the blue-green cutover.

**Expected result**: pushing to `main` runs Build → Publish → DeployDev → DeployQA automatically, then **pauses** at DeployProd awaiting the approval configured in Step 5.

---

## Part 4: Deployment Strategies — `runOnce` vs. Rolling vs. Canary

AZ-400 expects you to know the strategy options even if this lab only implements one:

| Strategy | Behavior | Where it fits |
|----------|----------|----------------|
| **`runOnce`** | Deploy, run, done — no built-in gradual rollout (used above) | Simple services, or where blue-green via slot swap already provides safety |
| **Rolling** | Deploy to a subset of instances at a time (`maxParallel`) | VM scale sets / multi-instance deployments where instance-by-instance rollout matters |
| **Canary** | Deploy to a small traffic percentage first, expand gradually | AKS with a service mesh or Azure Front Door weighted routing; highest safety, highest complexity |

Blue-green via App Service slot swap (this lab) gets you most of canary's safety — instant rollback by swapping back — without needing a mesh or weighted routing layer. True canary (partial traffic split) is covered conceptually in [Lab 7](lab-7-site-reliability-engineering.md) alongside error-budget-based rollout decisions.

---

## Part 5: Validate the Approval Gate

### Step 7: Trigger a Run and Approve Prod
```bash
git checkout main && git pull
git commit --allow-empty -m "Trigger release pipeline"
git push
```

1. **Pipelines** → open the running pipeline → confirm DeployDev and DeployQA complete automatically
2. DeployProd shows **Waiting for approval** — open it, review the diff, click **Approve**
3. Confirm the stage runs and the slot swap task completes

**Validation checkpoint**:
```bash
az webapp show --name az400-app-<unique-suffix> --resource-group az400-lab3-rg --query "state"
curl -I https://az400-app-<unique-suffix>.azurewebsites.net
```
A `200`/`302` response confirms the swap succeeded and `production` is serving the newly deployed code.

### Step 8: Prove Rollback Is a Swap, Not a Redeploy
```bash
az webapp deployment slot swap \
  --name az400-app-<unique-suffix> \
  --resource-group az400-lab3-rg \
  --slot staging \
  --target-slot production \
  --action swap
```
Running the swap a second time in the same direction is a no-op for demonstration; in a real rollback you'd swap `production` back to the previous `staging` contents — the point is this is a control-plane operation measured in seconds, not a redeploy.

---

## Part 6: Cleanup

```bash
az group delete --name az400-lab3-rg --yes --no-wait

az pipelines environment delete --environment-id $(az pipelines environment list --project az400-devops-lab --query "[?name=='Dev'].id" -o tsv) --project az400-devops-lab --yes
az pipelines environment delete --environment-id $(az pipelines environment list --project az400-devops-lab --query "[?name=='QA'].id" -o tsv) --project az400-devops-lab --yes
az pipelines environment delete --environment-id $(az pipelines environment list --project az400-devops-lab --query "[?name=='Prod'].id" -o tsv) --project az400-devops-lab --yes
```

> **Important**: App Service Standard tier bills **hourly** regardless of traffic. Don't leave `az400-lab3-plan` running between lab sessions.

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **Azure DevOps Environments** | Gives deployment history, approvals, and checks without hand-rolling gate logic in YAML |
| **Environment-scoped manual approval** | Enforced by the platform, auditable, and can't be bypassed by editing the pipeline that needs it |
| **`dependsOn` stage chaining** | Encodes the Dev → QA → Prod promotion order directly in the pipeline definition |
| **Blue-green via slot swap** | Zero-downtime deploy with an instant, control-plane-only rollback path |
| **Deployment strategy selection (`runOnce` vs. rolling vs. canary)** | Matches deployment risk tolerance to the right level of complexity instead of defaulting to the fanciest option |

---

## Common Mistakes to Avoid
- **Deploying straight to `production` in every environment**: skips the entire point of slot-based blue-green for the environment that actually needs it (Prod)
- **Putting the approval check in YAML (`condition`) instead of the Environment**: a YAML-only gate can be edited away in the same PR that needs approving
- **Forgetting Standard tier is required for slots**: a Free/Basic App Service plan silently has no **Deployment slots** option in the portal
- **Swapping slots without validating the staging slot first**: hit the staging URL directly (`https://<app>-staging.azurewebsites.net`) before swapping, not after
- **Leaving the S1 plan running after the lab**: it bills hourly whether or not anything is deployed to it

---

## Next Steps
- [Lab 4: Pipeline Security & Compliance](lab-4-pipeline-security-compliance.md) hardens this same pipeline's service connection and adds scanning gates before deploy
- [Lab 6: Instrumentation & Feedback Strategy](lab-6-instrumentation-feedback-strategy.md) adds release annotations to this pipeline so deploys are visible on the Application Insights timeline
- [Lab 7: Site Reliability Engineering Practices](lab-7-site-reliability-engineering.md) extends the Prod gate to block automatically on error-budget burn, not just require manual approval
