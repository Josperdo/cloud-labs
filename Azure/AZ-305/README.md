# AZ-305: Azure Solutions Architect Expert — Hands-On Labs

Five labs covering the four AZ-305 domains: identity/governance/monitoring design, data storage design, business continuity design, and infrastructure solutions design (split across compute/application and network, since it's the heaviest-weighted domain on the exam). AZ-305 is a *design* exam — the questions aren't "how do you click this," they're "given these requirements, which option is correct and why." Every lab here pairs a requirements-driven decision (a comparison table you build before you touch the portal) with a hands-on deployment of the option that decision leads to.

These labs assume the AZ-104 foundation (VNets, RBAC, storage, monitoring — see [AZ-104](../AZ-104/README.md)) plus the security and identity depth from [AZ-500](../AZ-500/README.md) and [SC-300](../SC-300/README.md). AZ-305 doesn't reteach those — it's about choosing *between* the things those tracks already taught you how to build.

---

## Labs Overview

| Lab | Domain | Duration | Cost | Key Skills |
|-----|--------|----------|------|-------------|
| **Lab 1** | Identity, Governance & Monitoring Design | 60–75 min | ~$0 | Management group hierarchy, Azure Policy initiatives, centralized vs. per-subscription Log Analytics design |
| **Lab 2** | Data Storage Solution Design | 60–75 min | ~$0–$1 | Storage service decision matrix, redundancy tier vs. RPO, customer-managed key encryption |
| **Lab 3** | Business Continuity Design | 60–90 min | ~$1–$3 | RTO/RPO-driven design, Azure Backup + restore test, Azure Site Recovery, availability zones vs. paired regions |
| **Lab 4** | Compute & Application Infrastructure Design | 60–75 min | ~$0–$3 | Hosting model decision matrix (App Service/AKS/Container Apps/VMSS/Functions), autoscale, deployment slots |
| **Lab 5** | Network Infrastructure Design | 60–90 min | ~$1–$5 | Hub-spoke topology, Azure Firewall/App Gateway/Front Door/WAF placement, private endpoints at scale |

**Total time**: ~5–6.5 hours
**Total cost**: ~$2–$12 if done sequentially and cleaned up after each lab (Lab 5's Azure Firewall bills hourly — deploy, do the lab, tear down the same session)

---

## Prerequisites

- Azure subscription with $50+ credit (or free tier — Lab 3 and Lab 5 are the most cost-sensitive)
- Completion of AZ-104, or equivalent comfort with VNets, storage accounts, RBAC, and Log Analytics
- Familiarity with AZ-500 Lab 1 (identity/PIM) and SC-300 Lab 1 (governance/administrative units) — this track builds design judgment on top of that hands-on knowledge rather than re-teaching it
- **Azure CLI** authenticated (`az login`)
- A second Azure region available to your subscription (Labs 3 and 5 use a secondary region for DR/hub-spoke design)

---

## How to Use These Labs

Work them in domain order — each lab's decision matrix references concepts (redundancy tiers, hosting models, network topology) that later labs assume you've already reasoned through once.

- **Coming straight from AZ-104/AZ-500/SC-300?** Start with Lab 1 — governance and monitoring design is the connective tissue between "I can configure this" (earlier tracks) and "I know which one to configure and why" (AZ-305).
- **Weak on the exam's design-judgment style?** Every lab's decision matrix step is the one to slow down on — the goal isn't the deployment, it's being able to defend the choice out loud.
- **Want the strongest interview story?** Lab 5's hub-spoke + Azure Firewall design is the one senior interviewers probe hardest — it's the lab to walk through in detail if asked "design a secure landing zone network."

---

## Lab Format

Same format as the other tracks — Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps — with one addition specific to this track: each lab opens its hands-on section with a **Design Decision** step, a comparison table of the viable options and why one wins for the stated requirements, before any portal/CLI work begins.

---

## Important Notes

### Cost Management
- Azure Firewall (Lab 5) and AKS-adjacent compute bill **hourly** — deploy, complete the lab, and `terraform destroy` / delete the resource group the same session
- Azure Site Recovery (Lab 3) can incur replication storage costs if left running — disable replication and delete the vault at the end of the lab, don't just stop the failover test
- Delete the resource group at the end of every lab

### Exam Alignment
These five labs map to the four AZ-305 exam domains (Infrastructure Solutions split into Compute/App and Network since it's 30–35% of the exam by itself). They're built for design-judgment depth on the highest-yield scenarios, not exhaustive coverage of every exam objective — AZ-305 rewards being able to explain trade-offs, and that's what each lab's decision-matrix step is for.

### Relationship to Other Tracks
AZ-305 is deliberately the capstone of the Azure certification path in this repo: Lab 1 assumes AZ-104/SC-300 governance concepts, Lab 3 assumes AZ-104 storage redundancy basics, and Lab 5 assumes AZ-500's network security concepts. If a step feels unfamiliar, the earlier track it draws from is called out inline.

---

## Additional Resources

- [AZ-305 Study Guide](https://learn.microsoft.com/en-us/certifications/azure-solutions-architect/)
- [Microsoft Learn AZ-305 Path](https://learn.microsoft.com/en-us/training/courses/az-305t00)
- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)
- [Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/)
