# Lab 3: CI/CD with OIDC Federated Auth

Check box if done: [ ]

## Overview
A long-lived IAM user access key pasted into a GitHub Actions secret is a standing credential with no expiry, valid from anywhere, that has to be manually rotated and manually revoked the moment it leaks. This lab replaces that with OIDC federation: GitHub issues a short-lived, cryptographically signed token per workflow run, and IAM trusts that token directly — no access key exists anywhere, ever. This closes out the CDK pipeline from [Lab 2](lab-2-iac-cdk.md) with the deployment mechanism a real team would actually use.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 — IAM OIDC providers, roles, and GitHub Actions minutes on a public repo are free; this lab only redeploys the near-zero-cost CDK stacks from Lab 2

---

## Scenario
Your team's current pipeline stores an IAM user's access key and secret as GitHub repo secrets, and nobody's rotated them in over a year because rotation means updating the secret in every repo that uses them and hoping nothing breaks mid-deploy. You're replacing that with federated auth: a GitHub Actions workflow that authenticates to AWS using a token GitHub itself issues, trusted by an IAM role scoped to exactly one repository and branch, with permissions scoped to exactly what deploying the Lab 2 CDK stacks requires. You also need a plan-on-PR/apply-on-merge pattern so nobody finds out what a deployment does by watching it hit production.

---

## Objectives
- Understand why OIDC federation eliminates an entire class of credential-leak risk that key rotation can only ever partially mitigate
- Create an IAM OIDC identity provider trusting GitHub Actions' token issuer
- Create an IAM role with a trust policy scoped to a specific repository and branch
- Scope the role's permissions policy to least privilege for a CDK deploy
- Build a GitHub Actions workflow using `aws-actions/configure-aws-credentials` with no stored long-lived credentials, running `cdk diff` on pull requests and `cdk deploy` on merge to `main`

---

## Part 1: Design Decision — Why OIDC Instead of a Stored Access Key

### The mechanism
GitHub Actions can request a short-lived, signed OIDC token from `token.actions.githubusercontent.com` for any workflow run, containing verifiable claims about that run — which repository, which branch or tag, which workflow, sometimes which environment. AWS IAM, configured to trust that issuer, exchanges the token for temporary credentials via `sts:AssumeRoleWithWebIdentity` — no access key ever needs to exist, because the "credential" is a token minted fresh for that specific run and dead within the hour.

### Why this beats a stored secret
| | Long-lived IAM user access key | OIDC federation |
|---|---|---|
| **Lifespan** | Exists until manually rotated or revoked — commonly months to years in practice | Minted per workflow run, expires automatically within the hour |
| **Blast radius if leaked** | Valid from anywhere, for as long as it exists undetected | A leaked token is already expired or nearly expired by the time it could be misused elsewhere |
| **Scoping** | Scoped by IAM policy alone — nothing stops the key from being used outside CI | Scoped by IAM policy **and** a trust condition on repo/branch — a token from a different repo or branch is rejected before IAM policy is even evaluated |
| **Rotation burden** | Manual, recurring, and easy to defer indefinitely | Nothing to rotate — there's no standing secret to rotate |
| **Audit trail** | CloudTrail shows the IAM user, indistinguishable from any other use of that same key | CloudTrail shows the assumed role plus the federated identity's claims, directly traceable to the specific repo/workflow run |

---

## Part 2: Create the IAM OIDC Identity Provider

### Step 1: Register GitHub's OIDC Issuer With IAM
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea
```
AWS validates the connection to GitHub's issuer directly as of 2023, so the thumbprint is effectively a legacy required parameter rather than something IAM actually depends on for security — still required by the API, so include GitHub's current published root CA thumbprint (verify the current value against [GitHub's OIDC documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect) before relying on this in a real environment, since root CAs do eventually rotate).

**This provider is a one-time, account-wide resource** — every repository's workflow that needs to assume an AWS role reuses this same provider; don't recreate it per project.

---

## Part 3: Create the IAM Role and Scope Its Trust Policy

### Step 2: Write the Trust Policy
The critical security control in this entire lab lives in this one `Condition` block — scope it to your exact repository and branch, not a wildcard.

```bash
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:<github-org>/<repo-name>:ref:refs/heads/main"
      }
    }
  }]
}
EOF
```
The `sub` claim encodes repo, and (depending on trigger) branch/tag/environment — scoping it this tightly means a workflow running in a fork, a different branch, or a different repository entirely **cannot** assume this role, regardless of what permissions the role itself grants. Part 5's workflow also needs to assume this same role from pull-request-triggered runs, which use a different `sub` format (`repo:<org>/<repo>:pull_request`) — add a second `StringLike` entry or a list of allowed patterns rather than broadening this to a wildcard.

### Step 3: Create the Role
```bash
aws iam create-role \
  --role-name github-actions-cdk-deploy \
  --assume-role-policy-document file://trust-policy.json
```

---

## Part 4: Scope Least-Privilege Permissions

### Step 4: The Permissions This Role Actually Needs
A CDK deploy needs to call CloudFormation, read/write the bootstrap asset bucket, and pass the CDK-generated execution roles — not blanket `AdministratorAccess`. CDK's own bootstrap process (Lab 2, Step 2) already created scoped deployment roles (`cdk-hnb659fds-deploy-role-...`, `cdk-hnb659fds-file-publishing-role-...`); the cleanest pattern is letting this GitHub role assume *those* roles rather than duplicating their permissions:

```bash
cat > permissions-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/cdk-hnb659fds-*"
    },
    {
      "Effect": "Allow",
      "Action": ["cloudformation:DescribeStacks", "cloudformation:DescribeStackEvents"],
      "Resource": "arn:aws:cloudformation:us-east-1:<AWS_ACCOUNT_ID>:stack/Lab2*/*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name github-actions-cdk-deploy \
  --policy-name cdk-deploy-scoped \
  --policy-document file://permissions-policy.json
```
**The anti-pattern to avoid**: attaching `AdministratorAccess` "to get the pipeline working" and never coming back to scope it down. A compromised or misconfigured workflow with admin access to the account is a far bigger incident than a scoped role that can only assume CDK's own already-scoped deploy roles for one project's stacks.

---

## Part 5: The GitHub Actions Workflow

### Step 5: Write the Workflow
`.github/workflows/deploy.yml`:

```yaml
name: cdk-plan-apply

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write   # required — this is what lets the job request an OIDC token at all
  contents: read

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/github-actions-cdk-deploy
          aws-region: us-east-1

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci
        working-directory: cdk-aws-lab

      - run: npx cdk diff --all
        working-directory: cdk-aws-lab

  apply:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/github-actions-cdk-deploy
          aws-region: us-east-1

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci
        working-directory: cdk-aws-lab

      - run: npx cdk deploy --all --require-approval never
        working-directory: cdk-aws-lab
```

**Reading this**: `permissions: id-token: write` is not optional boilerplate — without it, the job has no ability to request a token from GitHub's OIDC issuer, and `configure-aws-credentials` fails immediately. `aws-actions/configure-aws-credentials` handles the entire `AssumeRoleWithWebIdentity` exchange and exports the resulting temporary credentials as environment variables for every subsequent step in the job — no explicit `aws sts assume-role-with-web-identity` call needed.

### Step 6: Push the Workflow and Open a Test PR
```bash
git add .github/workflows/deploy.yml
git commit -m "Add OIDC-federated CDK deploy pipeline"
git checkout -b test-oidc-pipeline
git push -u origin test-oidc-pipeline
gh pr create --title "Test OIDC pipeline" --body "Validates the plan job"
```

**Validation checkpoint**: The `plan` job runs, successfully assumes the role, and posts a `cdk diff` output with no errors. Check the **Actions** tab — a successful `configure-aws-credentials` step logs the assumed role ARN, confirming federation worked without any stored key.

### Step 7: Merge and Verify the Apply Job
```bash
gh pr merge --squash
```
**Validation checkpoint**: The `apply` job runs on the merge to `main` and completes `cdk deploy` successfully. Cross-reference in CloudTrail:
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
  --max-results 5
```
Expect an event showing the `github-actions-cdk-deploy` role assumed, with the federated identity's claims in the event detail — this is the audit trail advantage from Part 1's comparison table, made concrete.

---

## Cleanup

```bash
# Remove the role's inline policy and the role itself
aws iam delete-role-policy --role-name github-actions-cdk-deploy --policy-name cdk-deploy-scoped
aws iam delete-role --role-name github-actions-cdk-deploy

# Remove the OIDC provider — only if nothing else in the account still uses it
aws iam list-open-id-connect-providers
aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com

# Tear down the CDK stacks this pipeline deployed
cd cdk-aws-lab && cdk destroy --all --force
```
Check for other repositories or roles trusting this same OIDC provider before deleting it — it's an account-wide resource, and deleting it breaks every role that trusts it, not just this lab's.

---

## Key Concepts

| Term | Definition |
|------|------------|
| **OIDC federation** | Trusting short-lived, signed tokens from an external identity provider (here, GitHub Actions) instead of a long-lived stored credential — the identity provider vouches for the caller per-request |
| **IAM OIDC identity provider** | The IAM resource registering trust in an external token issuer's URL — a one-time, account-wide setup, not per-repository |
| **`sub` claim scoping** | The specific field in GitHub's OIDC token (`repo:org/repo:ref:refs/heads/branch`) that a trust policy's `Condition` matches against — this is what prevents a workflow from a different repo or branch from assuming the role |
| **`AssumeRoleWithWebIdentity`** | The STS API call that exchanges a valid OIDC token for temporary AWS credentials, scoped to the assumed role's permissions and expiring automatically |
| **`aws-actions/configure-aws-credentials`** | The GitHub Action that requests the OIDC token, performs the STS exchange, and exports temporary credentials as environment variables for the rest of the job |
| **Plan-on-PR / apply-on-merge** | Running a read-only preview (`cdk diff`) on every pull request and the actual deploy only after merge to the protected branch — the CI/CD equivalent of never running `terraform apply`/`cdk deploy` without having seen the diff first |

---

## Common Mistakes
- **Scoping the trust policy's `sub` claim to a wildcard (`repo:org/*:*`)**: grants every repository and branch in the org the ability to assume this role — scope it to the exact repo and branch/ref pattern actually needed
- **Storing `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` as repo secrets "just in case" alongside the OIDC role**: defeats the entire point — if a long-lived key still exists as a fallback, it's still the weakest link an attacker will find first
- **Forgetting `permissions: id-token: write` in the workflow**: the job fails at the `configure-aws-credentials` step with an unclear permissions error — this is almost always the first thing to check when OIDC federation "isn't working"
- **Attaching `AdministratorAccess` to get the pipeline working and never revisiting it**: the standing risk of a compromised CI job is proportional to what its role can do — scope it to exactly what a deploy needs, as Part 4 does
- **Not testing the PR-triggered `sub` format separately from the push-triggered format**: `pull_request` events produce a different `sub` claim than `push` events on `main` — a trust policy tested only against one trigger type can silently fail for the other

---

## Next Steps
This is the same OIDC federation pattern from [Lab 1's IRSA](lab-1-eks-fundamentals.md) applied to a different token issuer — GitHub instead of the EKS cluster's own OIDC provider, but the same "external identity provider vouches for a scoped caller" mechanism underneath. For the IAM fundamentals this lab's trust and permissions policies build on, see [SAA-C03 Lab 1](../SAA-C03/aws-lab-1-vpc-iam.md). [Lab 4](lab-4-secrets-config-management.md) continues the "no long-lived credentials anywhere" theme at the application layer — Secrets Manager and SSM Parameter Store instead of hardcoded config.
