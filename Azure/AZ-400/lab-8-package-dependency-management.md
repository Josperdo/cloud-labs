# Lab 8: Package & Dependency Management (Capstone)

Check box if done: [ ]

## Overview
This capstone closes the track by extracting a shared library out of `az400-app`, publishing it as a versioned package to a private **Azure Artifacts** feed, consuming it back in the app pipeline behind a vulnerability gate, and — the actual point of a capstone — showing how every prior lab in this track (source control, CI, CD, pipeline security, IaC, instrumentation, SRE) fits together as one continuous chain rather than eight disconnected exercises.

**Estimated time**: 90–120 minutes
**Cost**: ~$0–$1

---

## Scenario
Two features in `az400-app` need the same date-formatting and validation logic. Copy-pasting it a third time isn't acceptable — the team wants a shared internal package, versioned properly, published through a pipeline (not `npm publish` from someone's laptop), and consumed with the same dependency-vulnerability gate built in [Lab 4](lab-4-pipeline-security-compliance.md). You're also asked to produce a one-page map of how this pipeline chain works end to end, for the next engineer who joins the team.

---

## Objectives
- Create an Azure Artifacts feed with an upstream source to the public npm registry
- Extract a shared library and apply a semantic versioning strategy
- Build a publish pipeline that versions and pushes the package on every merge to `main`
- Consume the package in `az400-app` with a vulnerability scan gate before it's allowed in
- Document the full Lab 1–8 pipeline chain as a single reference diagram/table

---

## Part 1: Create an Azure Artifacts Feed

### Step 1: Create the Feed with an Upstream Source
```bash
az artifacts feed create \
  --name az400-shared-packages \
  --organization https://dev.azure.com/<ORG> \
  --project az400-devops-lab
```

An **upstream source** lets the feed transparently proxy the public npm registry — consumers `npm install` against one feed URL and get both internal and public packages, with every public package cached the first time it's pulled (protects against a left-pad-style registry outage or a yanked version breaking your build).
1. Portal → **Artifacts** → `az400-shared-packages` → **Feed settings** → **Upstream sources** → **+ Add** → **npmjs.org**

---

## Part 2: Extract and Version the Shared Library

### Step 2: Extract the Library
```bash
mkdir az400-shared-utils && cd az400-shared-utils
git init
cat > package.json <<'EOF'
{
  "name": "@az400/shared-utils",
  "version": "0.1.0",
  "main": "index.js",
  "publishConfig": {
    "registry": "https://pkgs.dev.azure.com/<ORG>/az400-devops-lab/_packaging/az400-shared-packages/npm/registry/"
  }
}
EOF

cat > index.js <<'EOF'
function formatDate(date) {
  return new Date(date).toISOString().split("T")[0];
}
module.exports = { formatDate };
EOF
```

### Step 3: Choose a Versioning Strategy
Semantic Versioning (`MAJOR.MINOR.PATCH`) is the standard AZ-400 expects you to know how to apply, not just define:

| Change type | Version bump | Example |
|-------------|--------------|---------|
| Breaking API change (function signature, removed export) | MAJOR | `1.4.2` → `2.0.0` |
| New backward-compatible feature | MINOR | `1.4.2` → `1.5.0` |
| Bug fix, no API change | PATCH | `1.4.2` → `1.4.3` |

For automated versioning tied to commit history instead of manually editing `package.json`, **GitVersion** derives the version from Git tags and commit message conventions (e.g., `+semver: breaking` in a commit message forces a major bump). This lab uses the simpler manual-bump pattern below to keep the pipeline readable; swap in a `GitVersion.yml` config and the `GitVersion@6` pipeline task if the team wants it fully automatic.

---

## Part 3: Publish Pipeline

### Step 4: `azure-pipelines-publish.yml`
```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - az400-shared-utils/*

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '20.x'

  - script: |
      cd az400-shared-utils
      npm ci
      npm test
    displayName: 'Install and test shared library'

  - task: MicrosoftSecurityDevOps@1
    displayName: 'SCA scan on shared library'
    inputs:
      categories: 'dependencies'

  - script: |
      cd az400-shared-utils
      npm audit --audit-level=high
    displayName: 'npm audit gate'

  - script: |
      cd az400-shared-utils
      # PATCH bump on every merge; use `npm version minor`/`major` manually for larger changes
      npm version patch --no-git-tag-version
      echo "##vso[task.setvariable variable=packageVersion]$(node -p "require('./package.json').version")"
    displayName: 'Bump version'

  - task: npmAuthenticate@0
    inputs:
      workingFile: 'az400-shared-utils/.npmrc'

  - script: |
      cd az400-shared-utils
      echo "registry=https://pkgs.dev.azure.com/<ORG>/az400-devops-lab/_packaging/az400-shared-packages/npm/registry/" > .npmrc
      npm publish
    displayName: 'Publish to Azure Artifacts feed'
```
`npmAuthenticate@0` injects a short-lived token scoped to the pipeline's build identity into `.npmrc` — no npm token stored as a pipeline secret, following the same "don't store a credential you don't have to" principle from Lab 4's service connection work.

**Validation checkpoint**: `az artifacts universal publish --organization https://dev.azure.com/<ORG> --feed az400-shared-packages --name shared-utils-check --version $(node -p "require('./az400-shared-utils/package.json').version") --path .` isn't needed for npm packages specifically — instead confirm via portal: **Artifacts** → `az400-shared-packages` → `@az400/shared-utils` shows the newly published version.

---

## Part 4: Consume the Package with a Vulnerability Gate

### Step 5: Reference the Feed from `az400-app`
`az400-app/.npmrc`:
```
registry=https://pkgs.dev.azure.com/<ORG>/az400-devops-lab/_packaging/az400-shared-packages/npm/registry/
always-auth=true
```
```bash
cd az400-app
npm install @az400/shared-utils
```

### Step 6: Add the Feed-Aware SCA Gate to the App's CI Pipeline
Extend the `templates/build-steps.yml` from [Lab 2](lab-2-continuous-integration-pipelines.md) — the same `npm audit --audit-level=high` gate from [Lab 4](lab-4-pipeline-security-compliance.md) now also covers the internally published package, since it's just another entry in `package-lock.json` from the scanner's perspective:
```yaml
  - task: npmAuthenticate@0
    inputs:
      workingFile: '.npmrc'
    displayName: 'Authenticate to internal feed'

  - script: npm ci
    displayName: 'Install dependencies (internal + public)'

  - script: npm audit --audit-level=high
    displayName: 'SCA gate — internal and public dependencies'
```
This is the payoff of building the SCA gate generically in Lab 4 rather than special-casing it to public npm packages — an internal package with a known-vulnerable transitive dependency gets caught by the exact same check, with zero additional pipeline logic.

**Validation checkpoint**: bump `@az400/shared-utils` to intentionally depend on a package with a known high-severity advisory, publish it, and confirm `npm audit --audit-level=high` fails the app's CI build the next time it installs.

---

## Part 5: Upstream Sources — Why They Matter

### Step 7: Confirm the Upstream Proxy Behavior
```bash
az artifacts universal list --organization https://dev.azure.com/<ORG> --feed az400-shared-packages --scope project 2>/dev/null || \
  echo "Check portal: Feed settings -> Views -> confirm public packages appear cached after first install"
```
The first time any pipeline or developer `npm install`s a public package (e.g., `lodash`) through this feed, Azure Artifacts fetches it from npmjs.org **once** and caches it permanently in the feed. Every subsequent install — even if npmjs.org is down or that exact version is later unpublished upstream — is served from the cache. This is a real production incident category (a maintainer unpublishing a package mid-day) that upstream sources close off entirely.

---

## Part 6: End-to-End Pipeline Recap

### Step 8: The Full Chain, Lab by Lab
| Lab | Contributes | Artifact it hands to the next stage |
|-----|-------------|--------------------------------------|
| **1 — Source Control Strategy** | Branch policies, PR review, work item linking | A `main` branch that only ever changes through a reviewed, policy-gated PR |
| **2 — Continuous Integration** | Multi-stage build, test, lint, artifact publish | `az400-app-drop` build artifact |
| **3 — Continuous Delivery** | Dev → QA → Prod promotion, blue-green slot swap | A running Prod deployment behind an approval gate |
| **4 — Pipeline Security** | Workload identity federation, secret scan, SCA scan, SBOM | A build with no stored secrets and a documented dependency manifest |
| **5 — Infrastructure as Code** | Plan-on-PR/apply-on-merge Terraform, drift detection | Infrastructure that matches Git, with locking and audit history |
| **6 — Instrumentation** | Release annotations, feature flags | A deploy that's traceable on the metrics timeline and reversible without a redeploy |
| **7 — SRE Practices** | SLO/error budget, automated release gate on live alert state | A pipeline that refuses to deploy on top of an active incident |
| **8 — Package Management (this lab)** | Versioned internal packages, feed-based SCA gate, upstream caching | Reusable, scanned, cached dependencies for every repo the team owns |

Read top to bottom, this is the actual answer to "walk me through your CI/CD pipeline" in an interview — not a single tool, but a chain where each stage's output is the next stage's guarantee.

---

## Part 7: Cleanup

```bash
az artifacts feed delete --name az400-shared-packages --organization https://dev.azure.com/<ORG> --project az400-devops-lab --yes

az pipelines delete --name azure-pipelines-publish --project az400-devops-lab --yes

# Full track teardown (only if you're done with the entire AZ-400 track):
az group delete --name az400-lab3-rg --yes --no-wait
az group delete --name az400-lab6-rg --yes --no-wait
az group delete --name az400-lab5-rg --yes --no-wait
az devops project delete --id $(az devops project show --project az400-devops-lab --query id -o tsv) --yes
```

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **Azure Artifacts feed with upstream sources** | Internal packages and cached public packages served from one URL, immune to upstream registry outages/unpublishes |
| **Semantic versioning discipline** | Consumers can safely auto-update PATCH/MINOR bumps and know to review MAJOR bumps |
| **Token-based feed authentication (`npmAuthenticate@0`)** | No long-lived npm token stored as a pipeline secret |
| **Feed-aware SCA scanning** | The same vulnerability gate covers internal and public dependencies without special-casing either |
| **End-to-end pipeline literacy** | Being able to explain the whole chain, not just the stage you personally touched, is what separates "ran a pipeline" from "designed a DevOps process" |

---

## Common Mistakes to Avoid
- **Publishing packages by hand (`npm publish` from a laptop)**: defeats the versioning discipline and audit trail a pipeline gives you for free — always publish from CI
- **Skipping the upstream source and installing straight from npmjs.org in CI**: loses the caching protection and makes every build dependent on npmjs.org's uptime
- **Auto-bumping MAJOR versions the same way as PATCH**: a breaking change needs a human decision, not an automatic bump — gate MAJOR bumps behind an explicit `npm version major` commit, not the CI script
- **Not running the SCA gate against the feed's resolved dependency tree**: scanning only the shared library's own `package.json` misses vulnerabilities introduced transitively when it's consumed
- **Treating this capstone as "just another lab" instead of the integration point it is**: the value here is the recap table in Part 6 — being able to narrate the full chain is the actual interview-ready skill

---

## Next Steps
- This closes the AZ-400 track. Return to the [AZ-400 README](README.md) for the full lab list and suggested pacing.
- For deeper package/dependency hygiene beyond this lab's scope, cross-reference [Azure/General Lab 6: Cost Management & FinOps](../General/lab-6-cost-management-finops.md) for tagging strategies that extend naturally to package/feed cost attribution
- Pair this track's pipeline security posture with [AZ-500](../AZ-500/README.md) for the identity and security-operations depth behind the controls used in Labs 4 and 7
