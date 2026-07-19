# Lab 2: Continuous Integration Pipelines

Check box if done: [ ]

## Overview
The placeholder pipeline from [Lab 1](lab-1-source-control-strategy.md) satisfies the build-validation policy but does nothing useful. This lab writes the real thing: a multi-stage Azure Pipelines YAML build that installs dependencies, lints, runs tests, and publishes a versioned artifact — triggered automatically on every PR (as build validation) and on every push to `main` (as CI proper). You'll also learn the trigger/pr split that trips up almost everyone the first time, dependency caching to keep runs fast, and YAML templates so pipeline logic isn't copy-pasted across every repo the team owns.

**Estimated time**: 75–90 minutes
**Cost**: ~$0 (Microsoft-hosted parallel jobs are free for the first 1,800 minutes/month on one free parallel job; private projects may need to request the free tier via Microsoft's parallelism request form if it's not already granted)

---

## Scenario
`az400-app` (created in Lab 1) needs a CI pipeline before anyone can trust the build-validation policy that's gating merges. The team wants: fast feedback on every PR, a single pipeline definition (not one YAML per branch), an artifact that Lab 3's release pipeline can actually deploy, and pipeline logic that doesn't need to be hand-copied into the next repo the team spins up.

---

## Objectives
- Author a multi-stage `azure-pipelines.yml`: build/test stage, then artifact-publish stage
- Correctly separate `trigger` (CI, push-based) from `pr` (build validation) configuration
- Publish a versioned build artifact consumable by a release pipeline
- Add dependency caching to cut pipeline run time
- Extract shared steps into a reusable YAML template
- Rewire Lab 1's build-validation policy to point at the real pipeline and prove it blocks a failing PR

---

## Part 1: A Minimal Single-Stage Pipeline

### Step 1: Repo Layout
Working in the `az400-app` repo cloned in Lab 1, add a minimal Node app (any stack works — swap the tasks below for your language):
```bash
mkdir -p src tests
cat > package.json <<'EOF'
{
  "name": "az400-app",
  "version": "1.0.0",
  "scripts": {
    "lint": "echo 'lint ok'",
    "test": "echo 'tests ok'"
  }
}
EOF
```

### Step 2: `azure-pipelines.yml` — Single Stage First
```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '20.x'
    displayName: 'Install Node 20'

  - script: npm ci
    displayName: 'Install dependencies'

  - script: npm run lint
    displayName: 'Lint'

  - script: npm test
    displayName: 'Run tests'
```

`trigger` fires on pushes to `main` (post-merge CI). `pr` fires on pull requests targeting `main` (pre-merge build validation). They are independent — a pipeline with only `trigger` configured will never run against a PR, which is the single most common reason a Lab 1-style build-validation policy silently never triggers.

**Expected result**: commit and push this file, then check **Pipelines** in the portal — a run should start automatically from the push trigger.

---

## Part 2: Multi-Stage YAML — Build, Test, Publish

Stages are the top-level unit of a pipeline; each stage contains jobs, each job contains steps. Splitting into stages makes dependencies and parallelism explicit and gives Lab 3's release pipeline a clean artifact to consume.

### Step 3: Rewrite as Two Stages
```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

variables:
  buildConfiguration: 'release'
  vmImageName: 'ubuntu-latest'

stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: BuildAndTest
        pool:
          vmImage: $(vmImageName)
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'
            displayName: 'Install Node 20'

          - script: npm ci
            displayName: 'Install dependencies'

          - script: npm run lint
            displayName: 'Lint'

          - script: npm test -- --ci
            displayName: 'Run tests'

          - task: PublishTestResults@2
            condition: succeededOrFailed()
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: '**/test-results.xml'
            displayName: 'Publish test results'

  - stage: Publish
    displayName: 'Publish Build Artifact'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - job: PublishArtifact
        pool:
          vmImage: $(vmImageName)
        steps:
          - script: |
              mkdir -p $(Build.ArtifactStagingDirectory)/app
              cp -r src package.json $(Build.ArtifactStagingDirectory)/app
              echo "$(Build.BuildNumber)" > $(Build.ArtifactStagingDirectory)/app/VERSION
            displayName: 'Stage artifact contents'

          - publish: $(Build.ArtifactStagingDirectory)/app
            artifact: az400-app-drop
            displayName: 'Publish artifact'
```

`dependsOn: Build` plus `condition: succeeded()` means a failing test run never produces a publishable artifact — the Publish stage simply doesn't execute.

**Validation checkpoint**: open the completed run in the portal — you should see two distinct stages, and the `az400-app-drop` artifact listed under **Artifacts**.

---

## Part 3: Dependency Caching

### Step 4: Cache `node_modules` Between Runs
```yaml
          - task: Cache@2
            inputs:
              key: 'npm | "$(Agent.OS)" | package-lock.json'
              restoreKeys: |
                npm | "$(Agent.OS)"
              path: $(System.DefaultWorkingDirectory)/node_modules
            displayName: 'Cache node_modules'

          - script: npm ci
            displayName: 'Install dependencies'
```
Place the `Cache@2` task immediately before the install step, in both stages that need it. The cache key includes a hash of `package-lock.json`, so a dependency change automatically invalidates it — you never deploy against a stale cache silently.

---

## Part 4: Reusable YAML Templates

### Step 5: Extract the Build Steps into a Template
Copy-pasting the same lint/test/cache steps into every repo's pipeline is how teams end up with 40 pipelines that all drift independently. Extract them once:

`templates/build-steps.yml`
```yaml
parameters:
  - name: nodeVersion
    type: string
    default: '20.x'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: ${{ parameters.nodeVersion }}
    displayName: 'Install Node ${{ parameters.nodeVersion }}'

  - task: Cache@2
    inputs:
      key: 'npm | "$(Agent.OS)" | package-lock.json'
      restoreKeys: |
        npm | "$(Agent.OS)"
      path: $(System.DefaultWorkingDirectory)/node_modules
    displayName: 'Cache node_modules'

  - script: npm ci
    displayName: 'Install dependencies'

  - script: npm run lint
    displayName: 'Lint'

  - script: npm test -- --ci
    displayName: 'Run tests'
```

### Step 6: Reference It from `azure-pipelines.yml`
```yaml
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: BuildAndTest
        pool:
          vmImage: $(vmImageName)
        steps:
          - template: templates/build-steps.yml
            parameters:
              nodeVersion: '20.x'
```

For a shared template used across multiple repos, this file would live in a separate "pipeline templates" repo referenced via a `resources: repositories:` block instead of a relative path — worth knowing for the exam, but a relative path is enough for a single-repo lab.

---

## Part 5: Rewire the Build Validation Policy

### Step 7: Point Lab 1's Policy at the Real Pipeline
The pipeline created in Lab 1 already points at this same `azure-pipelines.yml` path, so no policy change is needed — confirm it, don't recreate it:
```bash
az repos policy list \
  --project az400-devops-lab \
  --repository-id az400-app \
  --branch refs/heads/main \
  --query "[?type.displayName=='Build'].settings.buildDefinitionId"
```

### Step 8: Prove It Blocks a Failing PR
```bash
git checkout -b feature/break-the-build
sed -i 's/tests ok/exit 1/' package.json
git add package.json && git commit -m "Intentionally break tests"
git push -u origin feature/break-the-build

az repos pr create \
  --project az400-devops-lab --repository az400-app \
  --source-branch feature/break-the-build --target-branch main \
  --title "Break the build (test)" --work-items <work-item-id>
```

**Validation checkpoint**: the PR's build-validation check should turn red and the **Complete** button stays disabled. Push a fix, confirm the pipeline re-runs automatically (`--reset-on-source-push` behavior from Lab 1) and turns green before merge is allowed.

---

## Part 6: Cleanup

Keep the pipeline — Lab 3 consumes the `az400-app-drop` artifact directly. To remove it entirely instead:
```bash
az pipelines delete --name az400-app-ci --project az400-devops-lab --yes
```

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **`trigger` vs. `pr` YAML blocks** | The most common reason a "working" pipeline never actually gates PRs — they're independent, not aliases of each other |
| **Multi-stage YAML with `dependsOn`/`condition`** | Prevents publishing an artifact built from a failing test run |
| **`Cache@2`** | Cuts pipeline run time without weakening the build — cache keys tied to the lockfile hash |
| **YAML templates** | Keeps pipeline logic DRY across repos instead of copy-pasted and drifting |
| **Artifact publishing** | Produces the exact deployable unit Lab 3's release pipeline consumes — build once, deploy the same bits everywhere |

---

## Common Mistakes to Avoid
- **Configuring only `trigger` and assuming PRs are covered**: PR builds require an explicit `pr:` block
- **Publishing an artifact before tests run**: always gate the publish stage on the build stage succeeding
- **Caching without a lockfile-based key**: a cache keyed only on OS silently serves stale dependencies after a `package-lock.json` change
- **One pipeline YAML per environment**: environment-specific behavior belongs in the release pipeline (Lab 3), not forked CI YAML files
- **Ignoring `PublishTestResults@2`**: without it, a failing test only shows as a red step in a log, not structured results in the **Tests** tab — much slower to triage

---

## Next Steps
- [Lab 3: Continuous Delivery & Release Management](lab-3-continuous-delivery-release-management.md) takes the `az400-app-drop` artifact through Dev → QA → Prod
- [Lab 4: Pipeline Security & Compliance](lab-4-pipeline-security-compliance.md) adds secret scanning and SCA scanning into this same Build stage
- For the GitHub Actions equivalent of trigger/PR separation (`on: push` / `on: pull_request`), see [Azure/General Lab 3](../General/lab-3-cicd-oidc.md)
