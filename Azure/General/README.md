# Azure General — Certification-Agnostic Applied Labs

Seven labs covering high-demand Azure skills that show up constantly in job postings and interviews but aren't fully owned by any single certification: Kubernetes (AKS), infrastructure-as-code breadth beyond Terraform, CI/CD with keyless cloud authentication, secrets/config management at scale, application observability, cost governance (FinOps), and enterprise landing-zone patterns.

This track exists for a specific reason: certifications prove you know the exam's slice of Azure, but hiring managers and ATS keyword scans look for AKS, Bicep, CI/CD, observability, and FinOps by name. These labs are the answer to "what have you actually built with this?" for the keywords that AZ-104/AZ-500/SC-300/[AZ-305](../AZ-305/README.md) don't cover in depth — built to be well-rounded, resume-searchable, and defensible in an interview, not to prep for a specific test.

---

## Labs Overview

| Lab | Focus Area | Duration | Cost | Key Skills |
|-----|-----------|----------|------|-------------|
| **Lab 1** | AKS Fundamentals | 75–90 min | ~$1–$3 | AKS cluster (spot/B-series node pool), kubectl, Helm, ingress, Entra ID-integrated cluster RBAC |
| **Lab 2** | Infrastructure as Code with Bicep | 60–75 min | ~$0–$1 | Bicep modules, parameter files, what-if deploys, Bicep vs. Terraform trade-offs |
| **Lab 3** | CI/CD with OIDC Federated Auth | 60–75 min | ~$0 | GitHub Actions, workload identity federation (no stored secrets), plan-on-PR/apply-on-merge |
| **Lab 4** | Secrets & Config Management at Scale | 60–75 min | ~$0 | Key Vault, App Configuration, managed identity, secret rotation |
| **Lab 5** | Observability & Application Performance Monitoring | 60–75 min | ~$0–$1 | Application Insights, distributed tracing, Azure Monitor workbooks, SLO-style alerting |
| **Lab 6** | Cost Management & FinOps | 45–60 min | ~$0 | Budgets/cost alerts, Azure Advisor, tagging for cost allocation, rightsizing |
| **Lab 7** | Landing Zone & Governance at Scale | 60–75 min | ~$0 | Cloud Adoption Framework landing zones, Policy initiatives at scale, policy-as-code |

**Total time**: ~7–8 hours
**Total cost**: ~$1–$5 if done sequentially and cleaned up after each lab (Lab 1's AKS node pool is the only hourly-billed resource — delete the cluster the same session)

---

## Prerequisites

- Azure subscription with $25+ credit (or free tier)
- Completion of AZ-104 or equivalent comfort with resource groups, VNets, and RBAC
- **Azure CLI** authenticated (`az login`) and **Bicep CLI** (`az bicep install`)
- **kubectl** and **Helm** installed (Lab 1)
- A **GitHub account** and the **GitHub CLI** (`gh`) for Lab 3's OIDC pipeline
- Familiarity with the Terraform basics from [AZ-104 Labs 6–7](../AZ-104/README.md) helps for Lab 2's comparison, but isn't required

---

## How to Use These Labs

Labs are independent of each other in concept but loosely build infrastructure you can reuse — the resource group and VNet from Lab 1 are convenient (not required) to reuse in Labs 4 and 5.

- **Targeting DevOps/platform or security engineer roles?** Labs 1, 3, and 4 are the strongest resume/interview trio — containers, keyless CI/CD, and secrets management are three of the most-screened-for keywords in current job postings.
- **Want a FinOps or cost-optimization talking point?** Lab 6 is short and self-contained — good for rounding out an interview answer about "how do you keep cloud spend under control."
- **Prepping for a platform/architecture-adjacent interview?** Lab 7's landing-zone lab pairs well with [AZ-305 Lab 1](../AZ-305/lab-1-identity-governance-monitoring-design.md) — one is design theory, the other is the CAF terminology and policy-as-code pattern hiring managers expect you to know by name.

---

## Lab Format

Same format as the other tracks: Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps.

---

## Important Notes

### Cost Management
- AKS (Lab 1) is the only lab with an hourly-billed resource (the node pool VMs — the control plane itself is free on the Free tier). Use a single B-series or spot node and delete the cluster (`az aks delete`) the same session.
- Everything else in this track is either free-tier Azure services (Key Vault, App Configuration, Policy, Advisor) or scoped to stay under $1.
- Delete the resource group at the end of every lab.

### Why "Certification-Agnostic"
No single Azure certification tests Kubernetes operations, Bicep, GitHub Actions OIDC, or FinOps in real depth — they're each a small footnote on AZ-104/AZ-500/AZ-305 at best. This track exists to close that gap deliberately, so the portfolio demonstrates the parts of the job that show up in day-to-day work and job descriptions, not just what's on an exam blueprint.

---

## Additional Resources

- [AKS Documentation](https://learn.microsoft.com/en-us/azure/aks/)
- [Bicep Documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [GitHub Actions OIDC with Azure](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure)
- [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/)
- [Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/)
