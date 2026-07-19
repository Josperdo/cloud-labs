# SC-500: Cloud and AI Security Engineer Associate — Hands-On Labs

**SC-500 is a beta certification as of mid-2026.** Microsoft has published the exam ("Implementing End-to-End Security Controls for Cloud and AI Workloads") and a beta skills-measured outline, but the exam itself, its exact objective weighting, and its GA date are all still subject to change. Treat these labs as built from that published beta study guide, not as a guarantee of final exam content — Microsoft has positioned SC-500 as the direct successor to **[AZ-500](../AZ-500/README.md)**, which retires August 31, 2026.

Eight labs covering the four SC-500 beta skill areas: identity/access/governance for workloads, storage and database security, network security, compute security (including containers), and — the material that genuinely differentiates this cert from AZ-500 — securing AI workloads and using Microsoft Purview and Security Copilot to manage posture across all of it. Where AZ-500 treats identity as the primary security boundary for a traditional Azure estate, SC-500 assumes that baseline and adds the workload-centric and AI-specific controls a cloud/AI security engineer is now expected to own.

These labs assume AZ-104-level comfort with core Azure services and, ideally, that you've worked through AZ-500 first — SC-500 leans on concepts (managed identity, Private Endpoints, Defender for Cloud plans) that AZ-500 builds from scratch and this track builds on top of.

---

## Labs Overview

| Lab | Focus Area | Duration | Cost | Key Skills |
|-----|-----------|----------|------|-------------|
| **Lab 1** | Identity, Access & Governance Foundations | 60–75 min | ~$0 | Workload-scoped RBAC, Azure Policy guardrails, managed identity fundamentals |
| **Lab 2** | Securing Storage & Databases | 60–90 min | ~$1–$4 | Defender for Storage, Defender for SQL, TDE with customer-managed key, storage network rules |
| **Lab 3** | Securing Networking for Cloud Workloads | 75–90 min | ~$2–$5 | NSG flow logs, Private Endpoint for Azure SQL, Azure Firewall baseline |
| **Lab 4** | Securing Compute Workloads | 75–90 min | ~$2–$6 | Defender for Servers, JIT VM access, Defender for Containers, vulnerability assessment |
| **Lab 5** | AI Workload Security Fundamentals | 60–90 min | ~$1–$5 | Azure OpenAI network isolation, content filtering, prompt-injection mitigation, Defender for AI Services |
| **Lab 6** | Microsoft Purview DSPM for AI | 60–75 min | ~$0–$2 | Sensitive info types, data classification, DSPM for AI, data security policies |
| **Lab 7** | Defender for Cloud — CSPM & Regulatory Compliance | 60–75 min | ~$0–$2 | Defender CSPM, attack path analysis, regulatory compliance dashboard, governance rules |
| **Lab 8 (Capstone)** | Security Copilot & Posture Monitoring | 60–90 min | ~$4–$10 | Security Copilot promptbooks, cross-domain investigation, unified posture workbook |

**Total time**: ~8.5–10.5 hours
**Total cost**: ~$10–$34 if done sequentially and cleaned up after each lab (Security Copilot's per-hour Security Compute Units in Lab 8 are the single biggest line item — provision it last and deprovision immediately after)

---

## Prerequisites

- An Azure subscription with Owner or Contributor + User Access Administrator rights (needed for RBAC and Policy work in Lab 1)
- **Azure OpenAI access** for Lab 5 — this is not automatically available on every subscription; request access/quota through the [Azure OpenAI access request process](https://aka.ms/oai/access) before you reach Lab 5, since approval can take time
- A **Microsoft Purview** account (or the ability to create one) for Lab 6
- **Microsoft Defender for Cloud** enabled at the subscription level (free Foundational CSPM tier is enough to start; Labs 4, 5, and 7 turn on paid plans)
- AZ-104-level comfort with VNets, NSGs, storage accounts, and RBAC — see [AZ-104](../AZ-104/README.md) if you need it
- Recommended: completion of [AZ-500](../AZ-500/README.md) — this track assumes you already know *why* managed identity, Private Endpoints, and Defender for Cloud plans exist and spends its time on the workload- and AI-specific layer on top of them
- **Azure CLI** authenticated (`az login`)

---

## How to Use These Labs

Work them in order — Lab 1's RBAC/Policy baseline and Lab 2's storage/database resources are reused as the "workload" that Labs 3, 4, and 7 layer network, compute, and posture controls onto. Lab 8 is a capstone that intentionally pulls signals from every prior lab, so it works best done last.

- **Coming straight from AZ-500?** Labs 1–4 will feel familiar in shape but narrower in scope — AZ-500 secures the *identity and platform*; SC-500 secures the *workload* running on top of it. Pay attention to what's different: Policy-as-guardrail instead of Conditional Access, Defender for SQL/Containers instead of just Defender for Storage.
- **New to AI security entirely?** Lab 5 and Lab 6 are the two labs with no AZ-500 equivalent — this is the actual new material in the cert. Don't skip the Azure OpenAI quota request; it's the most common blocker.
- **Want the interview story?** Lab 8's Security Copilot investigation is the strongest talking point — "I can tie identity, storage, network, compute, AI, and DSPM signals into one investigation instead of pivoting across six blades by hand."

---

## Lab Format

Same format as the other tracks: Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps.

---

## Important Notes

### Beta Status
SC-500 is in beta. Microsoft periodically revises beta exams' objective domains as it collects data from early test-takers, and the exam code, weighting, or even the name could change before general availability. These labs are built to the skills-measured outline published for the beta and are a defensible way to build the hands-on muscle the exam is testing — but if you're actively studying for a scheduled beta exam, cross-check the current outline on Microsoft Learn before you sit it.

### Cost Management
- Defender for Cloud paid plans (Labs 2, 4, 5, 7) bill **hourly per resource** — enable them, do the lab, disable them the same session
- Azure Firewall (Lab 3) and AKS (Lab 4) are the most expensive per-hour resources in this track — don't leave them running overnight
- Security Copilot (Lab 8) is billed by **Security Compute Unit (SCU)** per hour regardless of how much you use it — provision the minimum capacity, do the lab in one sitting, and deprovision immediately
- Delete the resource group at the end of every lab, and separately confirm subscription-level Defender plans are set back to `Free` — deleting resources does not disable subscription-scoped billing

### Exam Alignment
These eight labs map to the four SC-500 beta skill areas: manage identity, access, and governance (20–25%); secure storage, databases, and networking (25–30%); secure compute including AI workloads (20–25%); and manage and monitor security posture (20–25%). They're not exhaustive of every beta objective — they're built to give you defensible, explainable hands-on experience with the highest-yield scenarios, with particular weight on the AI-workload material that's genuinely new versus AZ-500.

---

## Additional Resources

- [SC-500 Certification Overview (Microsoft Learn)](https://learn.microsoft.com/en-us/credentials/certifications/cloud-ai-security-engineer-associate/)
- [Microsoft Learn SC-500 Training Path](https://learn.microsoft.com/en-us/training/courses/sc-500t00)
- [Azure OpenAI Service Access Request](https://aka.ms/oai/access)
- [Microsoft Purview DSPM for AI documentation](https://learn.microsoft.com/en-us/purview/ai-microsoft-purview)
- [Microsoft Defender for Cloud documentation](https://learn.microsoft.com/en-us/azure/defender-for-cloud/)
- [Microsoft Security Copilot documentation](https://learn.microsoft.com/en-us/copilot/security/microsoft-security-copilot)
