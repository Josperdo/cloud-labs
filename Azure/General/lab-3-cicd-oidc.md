# Lab 3: CI/CD with OIDC Federated Auth

Check box if done: [ ]

## Overview
"How does your pipeline authenticate to the cloud?" is a near-guaranteed interview question, and "we store a long-lived client secret in a GitHub secret" is the wrong answer — it's a standing credential that can leak, doesn't expire on its own, and isn't scoped to a specific repo or branch by anything Azure enforces. This lab builds the keyless answer: Entra ID workload identity federation (OIDC), where GitHub Actions exchanges a short-lived signed token for Azure access, and no secret exists anywhere to steal.

**Estimated time**: 60-75 minutes
**Cost**: ~$0

---

## Scenario
You need a GitHub Actions pipeline that deploys infrastructure to Azure — a plan/preview on every pull request, an apply on merge to `main`. The requirement from your team lead is explicit: no `AZURE_CLIENT_SECRET` in GitHub Secrets, ever. You're setting up an Entra ID app registration with a federated credential trust scoped to your exact repo and branch, wiring `azure/login` to use OIDC instead of a secret, and scoping the service principal's Azure permissions to a single resource group instead of the whole subscription.

---

## Objectives
- Create an Entra ID app registration and service principal with no client secret or certificate
- Configure federated credentials scoped to a specific GitHub repo, branch, and PR trust relationship
- Scope RBAC to a single resource group instead of subscription-wide `Contributor`
- Build a GitHub Actions workflow that plans on pull requests and applies on merge to `main`
- Verify the OIDC token exchange in an actual Actions run log

---

## Part 1: Design Decision — Why OIDC Instead of a Stored Secret

### The mechanism
When a GitHub Actions job runs with `permissions: id-token: write`, GitHub's own OIDC token issuer (`token.actions.githubusercontent.com`) can mint a short-lived JSON Web Token for that job. The token is cryptographically signed by GitHub and its claims identify exactly which repo, which branch or PR, and which environment the job is running under.

Entra ID doesn't need a secret to trust that token — it needs a **federated credential**: a trust relationship you configure ahead of time that says "if a token arrives from `token.actions.githubusercontent.com`, signed correctly, with a subject claim matching `repo:<org>/<repo>:ref:refs/heads/main`, treat it as this app registration." `azure/login` presents the GitHub token to Entra ID at runtime, Entra validates it against the federated credential, and issues a normal short-lived Azure access token back to the job. That access token is what actually calls the Azure Resource Manager API.

### Why this beats a stored secret
A stored `AZURE_CLIENT_SECRET` GitHub secret works, but:
- It's a long-lived credential (default expiry up to 2 years) sitting in GitHub's secret store — a GitHub compromise or an overly-permissive workflow leaks it directly
- Nothing on the Azure side scopes it to a specific repo or branch — anyone who has the secret can authenticate as that service principal from anywhere, not just your pipeline
- It needs manual or scripted rotation before expiry, and a missed rotation is an outage

With OIDC, there's no secret to leak because none exists. The trust is enforced by Entra ID checking the token's subject claim — a token from a fork, a different branch, or a different repo simply won't match and authentication fails before any Azure call happens.

---

## Part 2: Create an App Registration and Service Principal

### Step 1: Create the App Registration
```bash
az ad app create --display-name "gh-oidc-deploy-pipeline"
```

Note the `appId` from the output — this is your `client-id` for later steps.

### Step 2: Create the Service Principal Backing the App
```bash
APP_ID=$(az ad app list --display-name "gh-oidc-deploy-pipeline" --query "[0].appId" -o tsv)

az ad sp create --id "$APP_ID"
```

There is no `--create-cert` and no secret generated anywhere in this flow — the whole point of OIDC federation is that the app registration authenticates via the federated credential trust configured in Part 3, not via a client secret or certificate. If you've used `az ad sp create-for-rbac` before, note that its default behavior generates a client secret; skip it here, or pass no credential-generating flags and configure federation manually as below.

**Validation checkpoint**: `az ad sp show --id "$APP_ID" --query "{displayName:displayName, appId:appId, id:id}"` returns the service principal with no credentials listed under `az ad app credential list --id "$APP_ID"`.

---

## Part 3: Configure the Federated Credential

Two separate federated credentials are needed: one trusting pushes to `main` (for apply), one trusting pull requests (for plan/preview).

### Step 3: Federated Credential for the `main` Branch
```bash
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "gh-main-branch-deploy",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<org>/<repo>:ref:refs/heads/main",
    "description": "Allows main branch pushes to authenticate for apply",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

### Step 4: Federated Credential for Pull Requests
```bash
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "gh-pull-request-plan",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<org>/<repo>:pull_request",
    "description": "Allows PR-triggered runs to authenticate for plan/preview only",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

The `subject` field is the entire security boundary here. `repo:<org>/<repo>:ref:refs/heads/main` matches only tokens minted for jobs running on `main` in that exact repo. `repo:<org>/<repo>:pull_request` matches only PR-triggered runs. A token from any other branch, a fork, or a different repo won't match either subject and Entra rejects the authentication attempt outright.

**Validation checkpoint**: `az ad app federated-credential list --id "$APP_ID" -o table` shows both credentials with the correct `subject` values.

---

## Part 4: Scope Least-Privilege RBAC

### Step 5: Assign Contributor at Resource Group Scope Only
```bash
SP_OBJECT_ID=$(az ad sp show --id "$APP_ID" --query id -o tsv)

az role assignment create \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Contributor" \
  --scope "/subscriptions/<your-subscription-id>/resourceGroups/<target-resource-group>"
```

Do not scope this to the subscription. A pipeline identity only ever needs to touch the resource group(s) it deploys into — subscription-wide `Contributor` means a compromised or misconfigured workflow (or a malicious PR that gets a maintainer to approve a run) can touch every resource in the subscription, not just the ones this pipeline owns.

**Validation checkpoint**: `az role assignment list --assignee "$APP_ID" --all -o table` shows exactly one `Contributor` assignment, scoped to the resource group path — not the subscription root.

---

## Part 5: The GitHub Actions Workflow

### Step 6: Write the Workflow

`.github/workflows/deploy.yml`:

```yaml
name: Deploy Infrastructure

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write   # required — without this, GitHub never issues an OIDC token and azure/login fails silently into "no credential" errors
  contents: read

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: <your-app-registration-client-id>
          tenant-id: <your-tenant-id>
          subscription-id: <your-subscription-id>

      - name: Terraform Init
        run: terraform init
        working-directory: ./infra

      - name: Terraform Plan
        run: terraform plan -no-color -out=tfplan
        working-directory: ./infra

  apply:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: <your-app-registration-client-id>
          tenant-id: <your-tenant-id>
          subscription-id: <your-subscription-id>

      - name: Terraform Init
        run: terraform init
        working-directory: ./infra

      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: ./infra
```

If you're deploying Bicep instead of Terraform, swap the `plan`/`apply` steps for:
```yaml
      - name: What-If (PR)
        run: az deployment group what-if --resource-group <target-resource-group> --template-file main.bicep

      - name: Deploy (main)
        run: az deployment group create --resource-group <target-resource-group> --template-file main.bicep
```

Note `client-secret` never appears anywhere in this file — `azure/login@v2` detects that `id-token: write` is set and no `client-secret` input was provided, and automatically requests the GitHub OIDC token and exchanges it with Entra ID.

### Step 7: Verify a Successful Run
Open a pull request against `main` to trigger the `plan` job, then check the Actions run log:

1. The **Azure Login** step should show `Federated token details` or `OIDC token retrieved` in its log (exact wording varies by `azure/login` version), followed by a successful login with no client secret referenced anywhere
2. There is no step anywhere requesting or masking a secret value — compare this to a `client-secret`-based login, which shows a masked `***` value in the log
3. The `terraform plan` (or `what-if`) step runs and shows the intended resource changes

**Validation checkpoint**: Merge the PR and confirm the `apply` job runs on the push to `main`, using the second federated credential (the `ref:refs/heads/main` one) — the PR-triggered `plan` job earlier authenticated using the `pull_request` federated credential instead. If you temporarily rename or delete a federated credential and re-run the workflow, the login step should fail with an `AADSTS70021` (no matching federated identity record found) error — a good way to prove the trust boundary is actually being enforced rather than just assumed.

---

## Cleanup
1. Remove both federated credentials:
   ```bash
   az ad app federated-credential delete --id "$APP_ID" --federated-credential-id "gh-main-branch-deploy"
   az ad app federated-credential delete --id "$APP_ID" --federated-credential-id "gh-pull-request-plan"
   ```
2. Remove the role assignment:
   ```bash
   az role assignment delete \
     --assignee-object-id "$SP_OBJECT_ID" \
     --role "Contributor" \
     --scope "/subscriptions/<your-subscription-id>/resourceGroups/<target-resource-group>"
   ```
3. Delete the service principal and app registration:
   ```bash
   az ad sp delete --id "$APP_ID"
   az ad app delete --id "$APP_ID"
   ```
4. Delete any resource group the pipeline actually deployed into during testing:
   ```bash
   az group delete --name <target-resource-group> --yes --no-wait
   ```
5. Remove or archive the `.github/workflows/deploy.yml` file from the test repo if this was a throwaway repo, and delete the `production` environment protection rule in the GitHub repo settings if you configured one

---

## Key Concepts

| Term | Definition |
|------|------------|
| **OIDC (OpenID Connect)** | An identity layer on top of OAuth 2.0 where a trusted issuer mints a signed token asserting who the caller is — here, GitHub asserts "this job is running for repo X, branch Y" |
| **Workload identity federation** | Letting a workload (a CI job, not a human) authenticate using a token from an external trusted issuer, instead of a stored secret native to the target cloud |
| **Federated credential** | The Entra ID object defining which external issuer and which subject claim are trusted to act as a given app registration |
| **Subject claim** | The specific string in the incoming token (e.g., `repo:<org>/<repo>:ref:refs/heads/main`) that Entra ID matches against the federated credential — this is the actual scoping mechanism |
| **Short-lived token** | The Azure access token issued after a successful federated exchange; it expires in about an hour and is never persisted anywhere, unlike a stored client secret |
| **Least-privilege RBAC scoping** | Granting the minimum role at the smallest resource scope the workload actually needs — a resource group here, not the subscription |
| **`id-token: write` permission** | The GitHub Actions workflow permission that allows the job to request an OIDC token from GitHub's issuer in the first place |

---

## Common Mistakes
- **Scoping the federated credential's subject too broadly** — using a wildcard or trusting `ref:refs/heads/*` instead of `ref:refs/heads/main` lets any branch (including attacker-controlled feature branches) authenticate as your deploy identity
- **Assigning `Contributor` at subscription scope instead of resource-group scope** — turns a scoped deploy pipeline into a subscription-wide blast radius if the workflow is ever compromised
- **Forgetting `permissions: id-token: write` in the workflow** — this fails silently into a generic "unable to get authority" or "no matching federated identity" error in `azure/login`, which looks like a misconfigured federated credential when the real problem is the missing permission block
- **Reusing one federated credential subject for both PR and main-branch triggers** — collapses the plan/apply trust boundary; a PR run should never be able to authenticate with the same trust used for production applies
- **Leaving a `client-secret` input in the workflow "just in case"** — if `azure/login` finds a `client-secret`, it uses secret-based auth instead of OIDC, silently defeating the entire point of this setup

---

## Next Steps
This pipeline is only useful once it has infrastructure to deploy — see [`lab-2-iac-bicep.md`](lab-2-iac-bicep.md) for the Bicep template this workflow's `what-if`/`deploy` steps target, or [AZ-104 Lab 7](../AZ-104/lab-7-terraform-modules.md) if you're using the Terraform variant instead. Once the pipeline itself is trusted end-to-end, [`lab-4-secrets-config-management.md`](lab-4-secrets-config-management.md) covers how to handle the application-level secrets (database passwords, API keys) that the deployed infrastructure still needs, now that the pipeline's own cloud authentication no longer requires one.
