# AZ-500: Azure Security Engineer — Hands-On Labs

Eight labs covering the core AZ-500 domains in full depth: identity & access security, network security, data/application security, and security operations — each domain covered twice, once at the platform baseline and once at the advanced/hybrid layer — with Terraform woven into the operations labs so the security baseline is repeatable, not just clicked together once.

These labs assume you're comfortable with basic Azure navigation and CLI (see [AZ-104](../AZ-104/README.md) if not). They go deeper on the *why* behind each control — AZ-500 tests judgment about tradeoffs, not just where the buttons are.

---

## Labs Overview

| Lab | Domain | Duration | Cost | Key Skills |
|-----|--------|----------|------|-------------|
| **Lab 1** | Identity & Access Protection | 60–75 min | ~$0 (P2 trial) | Break-glass accounts, PIM eligible roles, Conditional Access, Identity Protection risk policies |
| **Lab 2** | Platform & Network Security | 60–90 min | ~$1–$3 | NSGs, Azure Bastion, Private Endpoints, deny-by-default network design |
| **Lab 3** | Data & Application Security | 60–75 min | ~$0–$1 | Key Vault RBAC, customer-managed keys, managed identity for app-to-vault auth |
| **Lab 4** | Security Operations & Automation | 75–90 min | ~$1–$3 | Defender for Cloud, Sentinel analytics rules, Terraform-deployed security baseline |
| **Lab 5** | Hybrid Identity & AD Connect Security | 75–90 min | ~$0 | OU/group sync scoping, PHS vs. PTA, Seamless SSO, staged rollout, Tier-0 server hardening |
| **Lab 6** | Advanced Network Security | 90–120 min | ~$8–$18 | Azure Firewall Premium IDPS/TLS inspection, DDoS Protection tier decision, WAF policy on App Gateway |
| **Lab 7** | SQL & Storage Data Security | 75–90 min | ~$0–$2 | Defender for SQL, Always Encrypted, Dynamic Data Masking, storage encryption scopes |
| **Lab 8** | Sentinel SOAR Playbooks & IR Automation | 90–120 min | ~$1–$3 | Logic Apps playbooks, Sentinel Automation Rules, enrichment-gated auto-remediation |

**Total time**: ~9.75–12.5 hours
**Total cost**: ~$11–$30 if done sequentially and cleaned up after each lab (Lab 6 is the single most expensive lab in the track — see its cost warnings before deploying)

---

## Prerequisites

- Azure subscription with an **Entra ID P2 trial** enabled (required for PIM and Identity Protection in Lab 1 — free to activate in a trial/dev tenant under **Entra ID** → **Licenses** → **All products** → **Try/Buy**)
- Global Administrator or Privileged Role Administrator on the tenant (needed to configure PIM and Conditional Access)
- **Azure CLI** authenticated (`az login`)
- **Terraform CLI** (Lab 4)
- Completion of AZ-104 labs, or equivalent comfort with VNets, NSGs, storage accounts, and RBAC
- **Lab 5 note**: no real corporate AD forest is required — the lab treats Azure AD Connect and an on-prem AD conceptually/via a simulated lab VM, so it's completable without standing up a full hybrid environment
- **Lab 6 note**: Azure DDoS **Network Protection** tier (~$2,944/month) is cost-prohibitive for a lab and is deliberately scoped as a portal walkthrough only — the hands-on deployment steps use the far cheaper **IP Protection** tier instead; do not enable Network Protection unless you intend to keep it running in a real environment

---

## How to Use These Labs

Work them in order — Lab 2's VM and Lab 3's storage account are good candidates to reuse across labs if you want to save setup time, but each lab's instructions assume you're starting fresh so nothing breaks if you don't.

- **Weak on identity?** Start with Lab 1 — it's the domain most AZ-500 questions test.
- **Coming from a networking background?** Lab 2 will feel familiar; focus on the *why* callouts (Bastion vs public IP, private endpoints vs service endpoints).
- **Want the automation story for interviews?** Lab 4's Terraform section is the one to point to — "I don't just click Enable in the portal, I codify the baseline."
- **Short on time or budget?** Labs 1, 3, 5, and 7 are the cheapest and fastest of the eight — Lab 6 is the one to schedule deliberately given its cost profile.
- **Lab 8 is the capstone** — it doesn't introduce a new domain so much as it wires Labs 1, 2/6, and 4 together into one automated detect-enrich-remediate chain. Do it last; it assumes the Sentinel workspace from Lab 4 already exists.

---

## Lab Format

Same format as AZ-104: Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps.

---

## Important Notes

### Licensing
PIM and Identity Protection require **Entra ID P2**. A trial tenant gives you a free 30-day P2 trial — activate it before Lab 1. Without P2, you can still read the lab and understand the concepts, but you won't be able to complete the hands-on steps.

### Cost Management
- Azure Bastion (Lab 2), Defender for Cloud paid plans (Lab 4, 7), Azure Firewall Premium and Application Gateway WAF_v2 (Lab 6) all bill **hourly** — deploy them, do the lab, and tear down the same session
- Delete the resource group at the end of every lab
- Disable any Defender for Cloud paid plans you enabled (Lab 4's Defender for Storage, Lab 7's Defender for SQL) once you're done — they don't stop billing just because the resources are gone, since pricing tiers are set at the subscription level
- Never enable Azure DDoS **Network Protection** (Lab 6) for lab purposes — its flat ~$2,944/month cost is wildly disproportionate to a short lab window; the lab defaults to IP Protection for exactly this reason

### Exam Alignment
These eight labs map to the same four AZ-500 exam domains as before, now covered twice each — a platform-baseline lab and an advanced/hybrid-layer lab per domain: manage identity and access (**Lab 1 + Lab 5**), secure networking (**Lab 2 + Lab 6**), secure compute/storage/data (**Lab 3 + Lab 7**), and manage security operations (**Lab 4 + Lab 8**). They're not exhaustive of every exam objective — they're built to give you defensible, explainable hands-on experience with the highest-yield scenarios, including the hybrid identity, advanced network, and SOAR automation topics that separate a working knowledge of Azure security from an exam-ready one.

---

## Additional Resources

- [AZ-500 Study Guide](https://learn.microsoft.com/en-us/certifications/azure-security-engineer/)
- [Microsoft Learn AZ-500 Path](https://learn.microsoft.com/en-us/training/courses/az-500t00)
- [Azure Security Benchmark](https://learn.microsoft.com/en-us/security/benchmark/azure/)
