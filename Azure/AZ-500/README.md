# AZ-500: Azure Security Engineer — Hands-On Labs

Four labs covering the core AZ-500 domains: identity & access security, network security, data/application security, and security operations — with Terraform woven in for the operations lab so the security baseline is repeatable, not just clicked together once.

These labs assume you're comfortable with basic Azure navigation and CLI (see [AZ-104](../AZ-104/README.md) if not). They go deeper on the *why* behind each control — AZ-500 tests judgment about tradeoffs, not just where the buttons are.

---

## Labs Overview

| Lab | Domain | Duration | Cost | Key Skills |
|-----|--------|----------|------|-------------|
| **Lab 1** | Identity & Access Protection | 60–75 min | ~$0 (P2 trial) | Break-glass accounts, PIM eligible roles, Conditional Access, Identity Protection risk policies |
| **Lab 2** | Platform & Network Security | 60–90 min | ~$1–$3 | NSGs, Azure Bastion, Private Endpoints, deny-by-default network design |
| **Lab 3** | Data & Application Security | 60–75 min | ~$0–$1 | Key Vault RBAC, customer-managed keys, managed identity for app-to-vault auth |
| **Lab 4** | Security Operations & Automation | 75–90 min | ~$1–$3 | Defender for Cloud, Sentinel analytics rules, Terraform-deployed security baseline |

**Total time**: ~4.5–5.5 hours
**Total cost**: ~$3–$7 if done sequentially and cleaned up after each lab

---

## Prerequisites

- Azure subscription with an **Entra ID P2 trial** enabled (required for PIM and Identity Protection in Lab 1 — free to activate in a trial/dev tenant under **Entra ID** → **Licenses** → **All products** → **Try/Buy**)
- Global Administrator or Privileged Role Administrator on the tenant (needed to configure PIM and Conditional Access)
- **Azure CLI** authenticated (`az login`)
- **Terraform CLI** (Lab 4)
- Completion of AZ-104 labs, or equivalent comfort with VNets, NSGs, storage accounts, and RBAC

---

## How to Use These Labs

Work them in order — Lab 2's VM and Lab 3's storage account are good candidates to reuse across labs if you want to save setup time, but each lab's instructions assume you're starting fresh so nothing breaks if you don't.

- **Weak on identity?** Start with Lab 1 — it's the domain most AZ-500 questions test.
- **Coming from a networking background?** Lab 2 will feel familiar; focus on the *why* callouts (Bastion vs public IP, private endpoints vs service endpoints).
- **Want the automation story for interviews?** Lab 4's Terraform section is the one to point to — "I don't just click Enable in the portal, I codify the baseline."

---

## Lab Format

Same format as AZ-104: Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps.

---

## Important Notes

### Licensing
PIM and Identity Protection require **Entra ID P2**. A trial tenant gives you a free 30-day P2 trial — activate it before Lab 1. Without P2, you can still read the lab and understand the concepts, but you won't be able to complete the hands-on steps.

### Cost Management
- Azure Bastion (Lab 2) and Defender for Cloud paid plans (Lab 4) bill **hourly** — deploy them, do the lab, and tear down the same session
- Delete the resource group at the end of every lab
- Disable any Defender for Cloud paid plans you enabled for Lab 4 once you're done — they don't stop billing just because the resources are gone

### Exam Alignment
These four labs map to the four AZ-500 exam domains: manage identity and access, secure networking, secure compute/storage/data, and manage security operations. They're not exhaustive of every exam objective — they're built to give you defensible, explainable hands-on experience with the highest-yield scenarios.

---

## Additional Resources

- [AZ-500 Study Guide](https://learn.microsoft.com/en-us/certifications/azure-security-engineer/)
- [Microsoft Learn AZ-500 Path](https://learn.microsoft.com/en-us/training/courses/az-500t00)
- [Azure Security Benchmark](https://learn.microsoft.com/en-us/security/benchmark/azure/)
