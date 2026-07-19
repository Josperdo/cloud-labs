# Lab 1: Source Control Strategy

Check box if done: [ ]

## Overview
Source control strategy is the part of DevOps that's easy to skip past — "we use Git, what else is there?" — and the part AZ-400 spends real weight on, because a repo with no branch policy is one accidental `git push --force origin main` away from losing history. This lab builds a trunk-based branching strategy on Azure Repos with branch policies that actually enforce it: minimum reviewer counts, linked work items, comment resolution, and a build-validation hook that blocks merge until CI is green. You'll also stand up the Azure DevOps project and repo that every later lab in this track builds on.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (Azure DevOps free tier — 5 free users, unlimited private repos, no billed Azure resources in this lab)

---

## Scenario
You've joined a team that's been pushing directly to `main` for a year. There's no PR review, no linked work items, and the last incident was a broken `main` that blocked three other developers for half a day. Your first task as the DevOps engineer is to fix the process, not just the code: pick a branching strategy the team can actually follow, and make the tooling enforce it instead of relying on people remembering the rules.

---

## Objectives
- Stand up an Azure DevOps organization, project, and repo via the CLI
- Compare trunk-based development and GitFlow, and justify picking trunk-based for this team
- Configure branch policies: minimum reviewers, comment resolution, linked work items, and block-direct-push
- Wire a build-validation policy that gates merges on a passing pipeline run
- Add a PR template and required-reviewer groups so review expectations are explicit, not tribal knowledge
- Prove the policy set actually blocks a non-compliant PR before allowing a compliant one through

---

## Part 1: Stand Up the Project and Repo

### Step 1: Install the Azure DevOps CLI Extension
```bash
az extension add --name azure-devops
az login
```

### Step 2: Create the Organization (Portal) and a Project (CLI)
Azure DevOps organizations can't be created via CLI — create one once at `https://dev.azure.com` (free, no Azure subscription required), then do everything else from the CLI.

```bash
az devops configure --defaults organization=https://dev.azure.com/<ORG>

az devops project create \
  --name az400-devops-lab \
  --description "AZ-400 track — source control through SRE labs" \
  --visibility private \
  --source-control git

az devops configure --defaults organization=https://dev.azure.com/<ORG> project=az400-devops-lab
```

### Step 3: Create the Application Repo
```bash
az repos create --name az400-app --project az400-devops-lab
```

**Expected result**: `az repos list --project az400-devops-lab -o table` shows `az400-app` alongside the project's default repo.

### Step 4: Trunk-Based Development vs. GitFlow
This is a design decision AZ-400 expects you to be able to defend, not just execute.

| | Trunk-Based Development | GitFlow |
|---|---|---|
| **Branch lifetime** | Feature branches live hours to a few days | `develop`, `release/*`, `hotfix/*` branches can live for weeks |
| **Merge frequency** | Multiple times a day, small diffs | Large, infrequent merges between long-lived branches |
| **CI/CD fit** | Matches continuous delivery well — every merge to `main` is potentially releasable | Naturally batches releases; needs extra branch-to-branch sync |
| **Conflict risk** | Low — branches diverge for hours, not weeks | Higher — long-lived branches drift apart |
| **Best for** | Teams practicing continuous delivery, SaaS products, this lab track | Teams with scheduled/versioned releases (e.g., shipped desktop software), regulated release trains |

**Decision for this team**: trunk-based, with short-lived `feature/*` branches merged to `main` via PR. It matches the CI/CD pipelines you'll build in Labs 2–3, and the team's actual problem (long-lived, drifting branches) is exactly what trunk-based avoids.

---

## Part 2: Branch Policies on `main`

Azure Repos branch policies are configured per-branch and enforced server-side — they can't be bypassed by a local Git client the way a `.gitignore`-style convention can.

### Step 5: Require a Minimum Number of Reviewers
```bash
az repos policy required-reviewer create \
  --project az400-devops-lab \
  --repository-id az400-app \
  --branch main \
  --blocking true \
  --enabled true \
  --required-reviewer-ids "<reviewer-group-or-user-id>" \
  --message "At least one senior engineer must review infra-affecting changes"
```

For a plain minimum-reviewer-count policy (any reviewer, not a specific person/group):
```bash
az repos policy min-reviewer-count create \
  --project az400-devops-lab \
  --repository-id az400-app \
  --branch main \
  --blocking true \
  --enabled true \
  --minimum-approver-count 1 \
  --creator-vote-counts false \
  --allow-downvotes false \
  --reset-on-source-push true
```

`--reset-on-source-push true` matters: without it, a reviewer can approve a PR and then the author can silently push unreviewed commits before merging.

### Step 6: Require Comment Resolution Before Merge
```bash
az repos policy comment-required create \
  --project az400-devops-lab \
  --repository-id az400-app \
  --branch main \
  --blocking true \
  --enabled true
```

### Step 7: Require Linked Work Items
Traceability from code change to work item is one of the things AZ-400 explicitly tests under "collaboration."
```bash
az repos policy work-item-linking create \
  --project az400-devops-lab \
  --repository-id az400-app \
  --branch main \
  --blocking true \
  --enabled true
```

### Step 8: Block Direct Pushes to `main`
1. Portal → **Repos** → **Branches** → `main` → **Branch policies**
2. Under **Require a minimum number of reviewers**, confirm it's already blocking direct pushes as a side effect of the reviewer policy (any push that isn't through a completed PR is rejected)
3. Confirm **Allow force pushes**: `Off` and **Allow deletes**: `Off` under **Branch security**

**Validation checkpoint**: `az repos policy list --project az400-devops-lab --repository-id az400-app --branch refs/heads/main -o table` shows all four policies as `isEnabled: true, isBlocking: true`.

---

## Part 3: Build Validation Policy

This is the hook that ties source control to CI — the pipeline you'll build in [Lab 2](lab-2-continuous-integration-pipelines.md) plugs in here.

### Step 9: Create a Placeholder Pipeline to Reference
If you haven't built the real CI pipeline yet, create a minimal one now so the policy has something to point at (Lab 2 replaces its contents):
```bash
az pipelines create \
  --name az400-app-ci \
  --project az400-devops-lab \
  --repository az400-app \
  --repository-type tfsgit \
  --branch main \
  --yml-path azure-pipelines.yml \
  --skip-first-run true
```

### Step 10: Add the Build Validation Policy
```bash
PIPELINE_ID=$(az pipelines show --name az400-app-ci --project az400-devops-lab --query id -o tsv)

az repos policy build create \
  --project az400-devops-lab \
  --repository-id az400-app \
  --branch main \
  --blocking true \
  --enabled true \
  --build-definition-id "$PIPELINE_ID" \
  --display-name "CI build validation" \
  --manual-queue-only false \
  --queue-on-source-update-only true \
  --valid-duration 720
```

`--valid-duration 720` expires a passing build result after 12 hours — this prevents a PR from merging on a stale green build if `main` has moved on since.

---

## Part 4: PR Template and Required Reviewer Groups

### Step 11: Add a PR Template
Azure Repos picks up a template automatically at this path. Create it in the repo root:

`.azuredevops/pull_request_template.md`
```markdown
## What does this PR do?
<!-- One or two sentences -->

## Linked work item
AB#<work-item-id>

## Type of change
- [ ] Feature
- [ ] Bug fix
- [ ] Infra / pipeline change
- [ ] Documentation only

## Checklist
- [ ] Tests added or updated
- [ ] No secrets or credentials in the diff
- [ ] Pipeline run is green
- [ ] Linked work item is accurate
```

### Step 12: Configure Automatic Required Reviewers by Path
Use required-reviewer policies scoped to a path filter so infra changes always pull in someone who owns that area — the closest Azure Repos equivalent to GitHub's `CODEOWNERS` file (which is what you'd use instead if this repo lived on GitHub):
```bash
az repos policy required-reviewer create \
  --project az400-devops-lab \
  --repository-id az400-app \
  --branch main \
  --blocking true \
  --enabled true \
  --required-reviewer-ids "<infra-reviewers-group-id>" \
  --path-filter "/infra/*;/azure-pipelines.yml" \
  --message "Infra and pipeline changes require infra-team review"
```

### Step 13: Test the Full Policy Set End-to-End
```bash
git clone https://dev.azure.com/<ORG>/az400-devops-lab/_git/az400-app
cd az400-app
git checkout -b feature/readme-update
echo "# az400-app" > README.md
git add README.md && git commit -m "Add README"
git push -u origin feature/readme-update

az repos pr create \
  --project az400-devops-lab \
  --repository az400-app \
  --source-branch feature/readme-update \
  --target-branch main \
  --title "Add README" \
  --work-items <work-item-id>
```

**Validation checkpoint**: Open the PR in the portal. Confirm the **Complete** button is disabled and shows unmet policies (reviewer approval pending, build validation running/pending, or unresolved comments). Approve as a second identity (or temporarily lower the reviewer count to test), wait for the build validation run to go green, then confirm **Complete** becomes available only once every policy shows a checkmark.

---

## Part 5: Cleanup

Keep the project and repo — Labs 2–8 build directly on top of `az400-devops-lab` / `az400-app`. If you're tearing the whole track down instead:

```bash
az repos policy list --project az400-devops-lab --repository-id az400-app --branch refs/heads/main --query "[].id" -o tsv | \
  xargs -I{} az repos policy delete --project az400-devops-lab --id {}

az pipelines delete --id "$PIPELINE_ID" --project az400-devops-lab --yes

az devops project delete --id $(az devops project show --project az400-devops-lab --query id -o tsv) --yes
```

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **Trunk-based development vs. GitFlow** | Branching strategy is a deliberate architectural choice, not a default — it has to match how the team actually releases |
| **Server-enforced branch policies** | Policy that lives in tooling survives a new hire not knowing the "unwritten rules" |
| **Build validation as a merge gate** | Connects source control directly to CI — nothing merges to `main` without a green pipeline run |
| **Work item linking** | Gives every code change a traceable "why," which matters for audits and incident postmortems |
| **`--reset-on-source-push`** | Closes the gap where an approved PR gets silently modified after approval |
| **PR templates** | Makes review expectations explicit instead of relying on the reviewer to know what to check |

---

## Common Mistakes to Avoid
- **Forgetting `--reset-on-source-push true`**: without it, an approved PR can be pushed to again after approval and merged without a fresh review
- **Setting minimum reviewers to 1 with `--creator-vote-counts true`**: lets the PR author approve their own change, defeating the policy entirely
- **Skipping the build-validation expiry (`--valid-duration`)**: a build result from days ago can be used to merge against a `main` that has since changed underneath it
- **Confusing "branch policy" with "branch permission"**: policies (this lab) gate PR completion; permissions (repo security settings) control who can push/force-push/delete at all — you generally need both
- **Not testing the policy set before trusting it**: a policy that looks configured in the portal but has the wrong path filter or a typo'd group ID silently does nothing

---

## Next Steps
- [Lab 2: Continuous Integration Pipelines](lab-2-continuous-integration-pipelines.md) replaces the placeholder pipeline from Part 3 with a real multi-stage build
- [Lab 3: Continuous Delivery & Release Management](lab-3-continuous-delivery-release-management.md) picks up the artifact this pipeline will publish
- For the equivalent GitHub-native pattern (branch protection rules, `CODEOWNERS`, required status checks), see [Azure/General Lab 3](../General/lab-3-cicd-oidc.md), which uses GitHub Actions throughout
