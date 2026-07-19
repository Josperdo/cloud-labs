# AZ-305: Azure Solutions Architect Expert — Hands-On Labs

Eight labs covering the four AZ-305 domains: identity/governance/monitoring design, data storage design, business continuity design, and infrastructure solutions design — the heaviest-weighted domain on the exam, so it gets four labs here (compute/application, network, global distribution, and migration) instead of one. AZ-305 is a *design* exam — the questions aren't "how do you click this," they're "given these requirements, which option is correct and why." Every lab here pairs a requirements-driven decision (a comparison table you build before you touch the portal) with a hands-on deployment of the option that decision leads to, closing with a capstone (Lab 8) that pulls every prior domain into one governed landing zone design.

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
| **Lab 6** | Multi-Region & Global Distribution Design | 75–90 min | ~$1–$4 | Front Door vs. Traffic Manager vs. App Gateway, Cosmos DB multi-region write models and consistency levels, active-active vs. active-passive, failover testing |
| **Lab 7** | Migration & Modernization Design | 75–90 min | ~$1–$4 | The 5 R's decision matrix, Azure Migrate assessment workflow, online vs. offline database migration, strangler-fig pattern |
| **Lab 8** | Capstone — End-to-End Landing Zone Solution Design | 90–110 min | ~$1–$4 | Subscription vending, policy-driven guardrails, cost/governance overlay, synthesis of Labs 1–7 into one architecture |

**Total time**: ~9–11.5 hours
**Total cost**: ~$5–$24 if done sequentially and cleaned up after each lab (Lab 5's Azure Firewall is still the single biggest line item if included — deploy, do the lab, tear down the same session; nothing else in the track approaches its hourly cost)

---

## Prerequisites

- Azure subscription with $50+ credit (or free tier — Lab 3 and Lab 5 are the most cost-sensitive)
- Completion of AZ-104, or equivalent comfort with VNets, storage accounts, RBAC, and Log Analytics
- Familiarity with AZ-500 Lab 1 (identity/PIM) and SC-300 Lab 1 (governance/administrative units) — this track builds design judgment on top of that hands-on knowledge rather than re-teaching it
- **Azure CLI** authenticated (`az login`)
- A second Azure region available to your subscription (Labs 3 and 5 already require a secondary region for DR/hub-spoke design; Lab 6 needs one too, for its multi-region App Service and Cosmos DB failover build)

---

## How to Use These Labs

Work them in domain order — each lab's decision matrix references concepts (redundancy tiers, hosting models, network topology) that later labs assume you've already reasoned through once.

- **Coming straight from AZ-104/AZ-500/SC-300?** Start with Lab 1 — governance and monitoring design is the connective tissue between "I can configure this" (earlier tracks) and "I know which one to configure and why" (AZ-305).
- **Weak on the exam's design-judgment style?** Every lab's decision matrix step is the one to slow down on — the goal isn't the deployment, it's being able to defend the choice out loud.
- **Want the strongest interview story?** Lab 5's hub-spoke + Azure Firewall design is the one senior interviewers probe hardest for network depth — but Lab 8's capstone is the one to walk through if asked "design a landing zone" as an open-ended prompt, since it's the only lab that shows all seven prior decisions reasoned through for a single company at once.
- **Short on time?** Labs 1, 2, and 4 run near-$0 and cover two of the exam's four domains; Labs 6–8 are the ones to prioritize if the goal is rounding out infrastructure-solutions depth specifically, since that domain is 30–35% of the exam by itself.

---

## Lab Format

Same format as the other tracks — Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps — with one addition specific to this track: each lab opens its hands-on section with a **Design Decision** step, a comparison table of the viable options and why one wins for the stated requirements, before any portal/CLI work begins.

---

## Important Notes

### Cost Management
- Azure Firewall (Lab 5) and AKS-adjacent compute bill **hourly** — deploy, complete the lab, and `terraform destroy` / delete the resource group the same session
- Azure Site Recovery (Lab 3) can incur replication storage costs if left running — disable replication and delete the vault at the end of the lab, don't just stop the failover test
- Azure Front Door Standard (Labs 6 and 7) has a small hourly base charge on top of usage — nowhere near Azure Firewall's cost, but not literally free either; delete the profile at the end of each lab rather than leaving it "for later"
- Delete the resource group at the end of every lab

### Exam Alignment
These eight labs map to the four AZ-305 exam domains, with Infrastructure Solutions — 30–35% of the exam by itself — covered across four labs instead of one: Compute/App (Lab 4), Network (Lab 5), Global Distribution (Lab 6), and Migration (Lab 7). Lab 6 reinforces the network and application-architecture objectives inside that same domain with a global-routing and multi-region-data lens Lab 5 doesn't cover; Lab 7 reinforces the domain's workload-assessment and migration-planning objective specifically, which Labs 1–6 don't touch at all. Lab 8's capstone doesn't add exam-objective breadth on its own — it's where Labs 1–7's separate domain coverage gets exercised together, which is the format AZ-305's longer scenario-based questions actually use. These labs are built for design-judgment depth on the highest-yield scenarios, not exhaustive coverage of every exam objective — AZ-305 rewards being able to explain trade-offs, and that's what each lab's decision-matrix step is for.

### Relationship to Other Tracks
AZ-305 is deliberately the capstone of the Azure certification path in this repo: Lab 1 assumes AZ-104/SC-300 governance concepts, Lab 3 assumes AZ-104 storage redundancy basics, and Lab 5 assumes AZ-500's network security concepts. Lab 8 is the track's own internal capstone — it assumes Labs 1 through 7 are already done and doesn't introduce a new domain, only synthesizes the seven that came before it into one landing zone design. If a step feels unfamiliar, the earlier track (or earlier lab) it draws from is called out inline.

---

## Additional Resources

- [AZ-305 Study Guide](https://learn.microsoft.com/en-us/certifications/azure-solutions-architect/)
- [Microsoft Learn AZ-305 Path](https://learn.microsoft.com/en-us/training/courses/az-305t00)
- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)
- [Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/)
