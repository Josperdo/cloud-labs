# AZ-400: Designing and Implementing Microsoft DevOps Solutions — Hands-On Labs

Eight labs covering the AZ-400 exam domains end to end: source control strategy, continuous integration, continuous delivery/release management, pipeline security and compliance (DevSecOps), infrastructure-as-code pipelines, instrumentation and feedback strategy, site reliability engineering practices, and package/dependency management. They build on each other — the same Azure DevOps project, repo, and app work through the whole track, so by Lab 8 you have one continuous pipeline chain, not eight disconnected exercises.

This track pairs with [Azure/General](../General/README.md) for the underlying CI/CD and IaC mechanics (Bicep, GitHub Actions OIDC) that Labs 4 and 5 reuse and extend into Azure DevOps-native patterns, and with [Azure/AZ-500](../AZ-500/README.md) for the identity and security-operations depth behind the pipeline security controls built in Lab 4 and the release gates in Lab 7.

---

## Labs Overview

| Lab | Focus Area | Duration | Cost | Key Skills |
|-----|-----------|----------|------|-------------|
| **Lab 1** | Source Control Strategy | 60–75 min | ~$0 | Trunk-based vs. GitFlow, Azure Repos branch policies, required reviewers, build validation, PR templates |
| **Lab 2** | Continuous Integration Pipelines | 75–90 min | ~$0 | Multi-stage Azure Pipelines YAML, trigger vs. PR triggers, artifact publishing, caching, reusable templates |
| **Lab 3** | Continuous Delivery & Release Management | 90–105 min | ~$1–$3 | Environments, manual approval gates, Dev/QA/Prod promotion, blue-green deploys via App Service slots |
| **Lab 4** | Pipeline Security & Compliance (DevSecOps) | 75–90 min | ~$0 | Workload identity federation, secret scanning, SCA scanning, SBOM generation |
| **Lab 5** | Infrastructure as Code Pipelines | 90 min | ~$0–$1 | Terraform `azurerm` remote state + locking, plan-on-PR/apply-on-merge, scheduled drift detection |
| **Lab 6** | Instrumentation & Feedback Strategy | 75 min | ~$0–$1 | Application Insights release annotations, Azure App Configuration feature flags, deploy/release decoupling |
| **Lab 7** | Site Reliability Engineering Practices | 90 min | ~$0–$1 | SLOs/error budgets, Azure Monitor alerting, automated release gates, incident runbooks |
| **Lab 8** | Package & Dependency Management (Capstone) | 90–120 min | ~$0–$1 | Azure Artifacts feeds, semantic versioning, feed-aware SCA gating, upstream source caching |

**Total time**: ~9.5–11.5 hours
**Total cost**: ~$2–$8 if done sequentially and cleaned up after each lab (Lab 3's App Service Standard plan is the only sustained hourly-billed resource — delete it the same session)

---

## Prerequisites

- An **Azure DevOps organization** (free at `dev.azure.com`, 5 free users, unlimited private Git repos) — created once before Lab 1
- A **GitHub account** is not required for this track (Azure Repos/Pipelines throughout) but is useful for comparing against [Azure/General Lab 3](../General/lab-3-cicd-oidc.md)'s GitHub Actions OIDC pattern, referenced in Labs 4 and 5
- **Azure CLI** authenticated (`az login`) with the `azure-devops` extension installed (`az extension add --name azure-devops`)
- **Terraform CLI** (Lab 5)
- Owner or Contributor on an Azure subscription, plus enough Entra ID permission to create app registrations and federated credentials (Labs 4, 5)
- Completion of [AZ-104](../AZ-104/README.md), or equivalent comfort with resource groups, App Service, and RBAC — this track assumes you can navigate core Azure resources without step-by-step hand-holding

---

## How to Use These Labs

Work them in order — each lab extends the same `az400-devops-lab` Azure DevOps project and `az400-app` repo created in Lab 1, and later labs reference resources, pipelines, and service connections built in earlier ones (the Lab 3 App Service, the Lab 4 federated service connection, and the Lab 6 Application Insights resource all get reused through Lab 8).

- **Coming from a security background?** Lab 4's workload identity federation section is the fastest way to see the AZ-500-adjacent identity material applied specifically to pipelines.
- **Coming from an infrastructure/IaC background?** Labs 2 and 5 will feel the most natural — go deeper on Labs 3, 6, and 7, which is where AZ-400 diverges hardest from a pure IaC skill set.
- **Want the strongest interview story?** Lab 8's end-to-end recap table is built specifically to be the answer to "walk me through your CI/CD pipeline" — point to it directly.

---

## Lab Format

Same format as the other tracks: Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps.

---

## Important Notes

### Cost Management
- Lab 3's App Service **Standard S1** plan bills hourly and is required for deployment slots — deploy it, do the lab, delete it the same session
- Everything else in this track is either free-tier Azure DevOps/Azure services (Azure Repos, Pipelines, Artifacts free tier; Application Insights and App Configuration free tiers) or scoped to stay under $1
- Delete resource groups and Azure DevOps pipelines/service connections at the end of every lab that creates them — see each lab's Cleanup section

### Exam Alignment
These eight labs map to the AZ-400 exam's domain weighting: source control (10–15%), continuous integration (20–25%), continuous delivery/release management (10–15%), security and compliance (10–15%), dependency/package management, collaboration (10–15%), instrumentation (5–10%), and site reliability engineering (5–10%). They're built for defensible, hands-on fluency with the highest-yield scenarios — not exhaustive coverage of every exam objective.

### Why Azure DevOps, Not GitHub Actions
AZ-400 tests Azure Repos, Azure Pipelines, Azure Boards, and Azure Artifacts by name — this track uses them throughout so the hands-on experience matches what the exam (and most AZ-400-hiring teams) actually expects. Where a pattern is functionally identical to something already built with GitHub Actions elsewhere in this repo (OIDC/workload identity federation, IaC plan-on-PR/apply-on-merge), the relevant lab cross-links to [Azure/General](../General/README.md) instead of re-explaining the underlying mechanism from scratch.

---

## Additional Resources

- [AZ-400 Study Guide](https://learn.microsoft.com/en-us/certifications/exams/az-400/)
- [Microsoft Learn AZ-400 Path](https://learn.microsoft.com/en-us/training/courses/az-400t00)
- [Azure Pipelines YAML Schema Reference](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/)
- [Azure DevOps CLI Reference](https://learn.microsoft.com/en-us/cli/azure/devops)
- [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
