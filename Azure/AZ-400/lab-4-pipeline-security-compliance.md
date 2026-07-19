# Lab 4: Pipeline Security & Compliance (DevSecOps)

Check box if done: [ ]

## Overview
[Azure/General Lab 3](../General/lab-3-cicd-oidc.md) already made the case for keyless pipeline authentication using GitHub Actions OIDC. This lab is that same argument applied to Azure Pipelines — **workload identity federation** replaces the stored client secret behind `az400-lab3-sc` — plus the rest of the DevSecOps gate a real pipeline needs before it's trustworthy: secret scanning so a leaked key never reaches a commit history, dependency (SCA) scanning so a known-vulnerable package doesn't ship silently, and an SBOM so "what's actually in this build" is answerable in an audit instead of guessed at.

**Estimated time**: 75–90 minutes
**Cost**: ~$0

---

## Scenario
A security review of `az400-devops-lab` flagged three gaps: the Azure service connection created in Lab 3 was set up before you knew better and may still be backed by a client secret, there's no automated check for hardcoded credentials in commits, and nobody can currently answer "which npm packages, and which versions, actually shipped in build #482." You're closing all three before this pipeline is allowed to touch anything beyond the lab resource group.

---

## Objectives
- Explain why workload identity federation is the Azure Pipelines equivalent of GitHub Actions OIDC, and configure it
- Replace a secret-backed service connection with a federated-credential-backed one
- Add secret scanning to the CI pipeline
- Add SCA (software composition analysis) dependency scanning with a severity-based fail threshold
- Generate and publish a Software Bill of Materials (SBOM) as a build artifact
- Gate the pipeline so a scan failure blocks the build, the same way a failing test does

---

## Part 1: Why Workload Identity Federation, Not a Stored Secret

The mechanism is the same trust model as [Azure/General Lab 3](../General/lab-3-cicd-oidc.md): a short-lived, cryptographically signed token replaces a long-lived secret. The difference is the token issuer. Where GitHub Actions uses `token.actions.githubusercontent.com`, Azure Pipelines has its own OIDC issuer per organization: `https://vstoken.dev.azure.com/<organization-id>`. When a pipeline run needs to authenticate to Azure, Azure DevOps mints a token from that issuer, and Entra ID validates it against a **federated credential** you configure ahead of time on the app registration — exactly the same pattern, different issuer and subject format.

| | GitHub Actions OIDC | Azure Pipelines Workload Identity Federation |
|---|---|---|
| **Token issuer** | `token.actions.githubusercontent.com` | `https://vstoken.dev.azure.com/<org-id>` |
| **Subject claim format** | `repo:<org>/<repo>:ref:refs/heads/main` | `sc://<ORG>/<PROJECT>/<service-connection-name>` |
| **Where the trust is configured** | `az ad app federated-credential create` | Same command, or portal: **New service connection** → **Workload identity federation (automatic)**, which does this for you |
| **What it replaces** | A GitHub Actions secret holding `AZURE_CLIENT_SECRET` | A service connection's stored client secret/certificate |

---

## Part 2: Migrate the Service Connection to Workload Identity Federation

### Step 1: Check Whether the Existing Connection Uses a Secret
```bash
az devops service-endpoint list --project az400-devops-lab --query "[?name=='az400-lab3-sc'].{name:name, authScheme:authorization.scheme}" -o table
```
`ServicePrincipal` with a `serviceprincipalkey` present means it's secret-backed. `WorkloadIdentityFederation` means it's already federated (if you used the portal's automatic option in Lab 3, it likely already is — this section shows the manual path for the general case).

### Step 2: Create the App Registration (Manual Path)
```bash
az ad app create --display-name "az400-pipeline-deploy"
APP_ID=$(az ad app list --display-name "az400-pipeline-deploy" --query "[0].appId" -o tsv)
az ad sp create --id "$APP_ID"
```

### Step 3: Create the Federated Credential
The `subject` must match the service connection's identity exactly — get the organization ID first:
```bash
ORG_ID=$(az devops project show --project az400-devops-lab --query id -o tsv)

az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "az400-devops-lab-service-connection",
    "issuer": "https://vstoken.dev.azure.com/'"$ORG_ID"'",
    "subject": "sc://<ORG>/az400-devops-lab/az400-lab3-sc-federated",
    "description": "Federated credential for az400 pipeline service connection",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

### Step 4: Create the Federated Service Connection
Easiest via portal (it wires the subject claim for you correctly): **Project settings** → **Service connections** → **New service connection** → **Azure Resource Manager** → **Workload identity federation (manual)** → paste the app registration's client ID and tenant ID → name it `az400-lab3-sc-federated`.

Then scope RBAC the same way as [Azure/General Lab 3](../General/lab-3-cicd-oidc.md) — resource-group scope, not subscription-wide:
```bash
SP_OBJECT_ID=$(az ad sp show --id "$APP_ID" --query id -o tsv)

az role assignment create \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Contributor" \
  --scope "/subscriptions/<your-subscription-id>/resourceGroups/az400-lab3-rg"
```

### Step 5: Repoint the Pipeline and Delete the Old Connection
Update every `azureSubscription:` reference in `azure-pipelines.yml` from `az400-lab3-sc` to `az400-lab3-sc-federated`, then:
```bash
az devops service-endpoint delete --id $(az devops service-endpoint list --project az400-devops-lab --query "[?name=='az400-lab3-sc'].id" -o tsv) --project az400-devops-lab --yes
```

**Validation checkpoint**: re-run the Lab 3 pipeline. The `AzureWebApp@1` login step succeeds with no client secret referenced anywhere in the service connection's configuration — `az devops service-endpoint show --id <id> --query "authorization.scheme"` returns `WorkloadIdentityFederation`.

---

## Part 3: Secret Scanning

### Step 6: Add a Secret-Scanning Step to the Build Stage
The Microsoft Security DevOps extension bundles credential scanning (formerly the standalone CredScan task) alongside other scanners. Install it once for the organization: **Organization settings** → **Extensions** → **Browse marketplace** → **Microsoft Security DevOps** → **Get it free**.

```yaml
          - task: MicrosoftSecurityDevOps@1
            displayName: 'Microsoft Security DevOps (secrets + IaC scan)'
            inputs:
              categories: 'secrets'
```

An open-source alternative that needs no extension install (useful if the org can't install marketplace extensions):
```yaml
          - script: |
              curl -sSfL https://raw.githubusercontent.com/gitleaks/gitleaks/master/scripts/install.sh | sh -s -- -b /usr/local/bin
              gitleaks detect --source . --report-format json --report-path $(Build.ArtifactStagingDirectory)/gitleaks-report.json --exit-code 1
            displayName: 'gitleaks secret scan'
```
`--exit-code 1` is what makes this a real gate — a non-zero exit fails the step, which (with default `continueOnError: false`) fails the stage.

---

## Part 4: SCA — Dependency Vulnerability Scanning

### Step 7: Add a Dependency Scan with a Severity Gate
```yaml
          - task: MicrosoftSecurityDevOps@1
            displayName: 'Microsoft Security DevOps (dependency scan)'
            inputs:
              categories: 'dependencies'

          - script: |
              npm audit --audit-level=high
            displayName: 'npm audit (fail on High/Critical)'
```
`npm audit --audit-level=high` exits non-zero only for High/Critical findings — Low/Moderate findings are logged but don't block the build, which keeps the gate from becoming so noisy that people start ignoring it. Tune the threshold per team risk appetite; the important part for the exam is knowing this is a policy decision, not a fixed default.

---

## Part 5: SBOM Generation

### Step 8: Generate an SBOM with Microsoft's `sbom-tool`
```yaml
          - script: |
              curl -Lo sbom-tool https://github.com/microsoft/sbom-tool/releases/latest/download/sbom-tool-linux-x64
              chmod +x sbom-tool
              ./sbom-tool generate \
                -b $(Build.ArtifactStagingDirectory)/app \
                -bc . \
                -pn az400-app \
                -pv $(Build.BuildNumber) \
                -ps "az400-devops-lab" \
                -nsb https://<ORG>.dev.azure.com
            displayName: 'Generate SBOM'

          - publish: $(Build.ArtifactStagingDirectory)/app/_manifest
            artifact: sbom
            displayName: 'Publish SBOM artifact'
```
The SBOM (in SPDX format) lists every dependency and version that went into the published artifact — this is what turns "which packages shipped in build #482" from an archaeology exercise into a one-artifact-download answer.

---

## Part 6: Gate the Pipeline on Scan Results

### Step 9: Confirm Failures Actually Block
Every scan step above uses its own non-zero exit code as the gate — no extra configuration needed, since Azure Pipelines fails a stage by default when any step in it fails. Add this as an explicit backstop:
```yaml
  - stage: SecurityGate
    displayName: 'Security Gate Summary'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - job: GateCheck
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: echo "All security scans passed — proceeding to Publish"
            displayName: 'Gate passed'
```
Make `Publish` depend on `SecurityGate` instead of `Build` directly, so there's one unambiguous stage a reviewer can point to and say "this is what proves the build is clean."

**Validation checkpoint**: intentionally commit a fake credential (`AWS_SECRET_ACCESS_KEY=AKIAFAKEEXAMPLE1234`) in a throwaway branch, open a PR, and confirm the secret-scan step fails the build and the PR's build-validation policy (from [Lab 1](lab-1-source-control-strategy.md)) blocks merge.

---

## Part 7: Cleanup

```bash
az ad app federated-credential delete --id "$APP_ID" --federated-credential-id "az400-devops-lab-service-connection"

az role assignment delete \
  --assignee-object-id "$SP_OBJECT_ID" \
  --role "Contributor" \
  --scope "/subscriptions/<your-subscription-id>/resourceGroups/az400-lab3-rg"

az ad sp delete --id "$APP_ID"
az ad app delete --id "$APP_ID"

az devops service-endpoint delete --id $(az devops service-endpoint list --project az400-devops-lab --query "[?name=='az400-lab3-sc-federated'].id" -o tsv) --project az400-devops-lab --yes
```

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **Workload identity federation for Azure Pipelines** | Removes the standing secret from the service connection — the exact same risk reduction as GitHub Actions OIDC, applied to Azure DevOps's own issuer |
| **Secret scanning as a build gate** | Catches a leaked credential before it's merged, not after it's already in Git history forever |
| **SCA scanning with a severity threshold** | Blocks known-exploitable dependencies without making every Low-severity finding a build-breaker |
| **SBOM generation** | Answers "what's actually in this build" for audits, incident response, and vulnerability-disclosure triage |
| **A single named gate stage** | Gives reviewers and auditors one place to point at instead of scattering "trust me, it's scanned" across steps |

---

## Common Mistakes to Avoid
- **Leaving the old secret-backed service connection active "just in case"**: delete it once the federated one is verified working — an unused credential is still a live attack surface
- **Scoping the federated credential's subject too loosely**: it must match the exact service connection identity, not a wildcard
- **Treating SCA findings as informational only**: without an explicit `--audit-level` or equivalent threshold, `npm audit` and similar tools report but don't fail — silently doing nothing
- **Generating an SBOM but never publishing it as an artifact**: an SBOM that isn't retrievable after the build completes doesn't help an auditor six months later
- **Running secret scans only on PR, not on `main`**: a secret that was merged before this lab existed won't be caught unless you also scan `main`'s existing history at least once

---

## Next Steps
- [Lab 5: Infrastructure as Code Pipelines](lab-5-infrastructure-as-code-pipelines.md) applies this same federated-identity pattern to a Terraform plan/apply pipeline
- [Lab 8: Package & Dependency Management](lab-8-package-dependency-management.md) extends SCA scanning to packages consumed from an internal Azure Artifacts feed
- Cross-reference [AZ-500 Lab 4: Security Operations & Automation](../AZ-500/lab-4-security-ops-automation.md) for wiring pipeline scan failures into a Sentinel analytics rule for org-wide visibility
