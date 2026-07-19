# SC-100: Microsoft Cybersecurity Architect Expert — Hands-On Labs

Eight labs covering the four SC-100 exam domains: designing solutions aligned to security best practices, designing security operations/identity/compliance capabilities, designing security solutions for infrastructure, and designing security solutions for applications and data. SC-100 is a *design* exam, and it's a *cross-domain* one — it doesn't test whether you can configure Conditional Access or deploy a firewall in isolation, it tests whether you can weigh options against a stated set of requirements and show how identity, network, application, and data decisions reinforce each other into one coherent Zero Trust architecture. Every lab here opens with a comparison-matrix decision before any hands-on configuration, the same structural pattern as this repo's AZ-305 track — SC-100 is the security-architecture counterpart to AZ-305, and it sits above [AZ-500](../AZ-500/README.md), [SC-200](../SC-200/README.md), [SC-300](../SC-300/README.md), and [SC-500](../SC-500/README.md) in the learning path: this track designs the strategy, those tracks implement and operate it.

All eight labs follow one continuous scenario — a mid-size organization recovering from a credential-based breach — so the identity, network, application, and data decisions in later labs build on and reference the earlier ones, culminating in a capstone synthesis in Lab 8.

---

## Labs Overview

| Lab | Focus Area | Duration | Cost | Key Skills |
|-----|-----------|----------|------|-------------|
| **Lab 1** | Zero Trust Strategy & Design Decision Frameworks | 60–75 min | ~$0 | Zero Trust maturity assessment, RaMP-aligned phased roadmap, Secure Score baselining, report-only Conditional Access rollout |
| **Lab 2** | Identity & Access Architecture Design | 60–75 min | ~$0 | Conditional Access layering (persona/risk/app), workload identity CA policies, PIM-eligible roles vs. standing access, Privileged Access Groups |
| **Lab 3** | Security Operations Architecture Design | 60–90 min | ~$1–$3 | Single vs. multi-workspace Sentinel, data residency-driven topology, retention tiering (Analytics/Basic Logs/Archive), scheduled analytics rules |
| **Lab 4** | Regulatory Compliance & Governance Architecture | 60–75 min | ~$0 | Unified control framework mapping, built-in regulatory compliance Policy initiatives, Purview Compliance Manager evidence mapping |
| **Lab 5** | Hybrid & Multicloud Infrastructure Security Design | 60–75 min | ~$0 | Password Hash Sync vs. PTA vs. Federation, Azure Arc onboarding, Defender for Cloud multicloud connectors (AWS CSPM) |
| **Lab 6** | Network Security Architecture Design | 60–90 min | ~$0–$2 | Security Service Edge (Entra Internet Access/Private Access) vs. hub-spoke firewall, per-app identity-aware network access |
| **Lab 7** | Application & API Security Architecture Design | 60–75 min | ~$1–$3 | WAF vs. APIM policies vs. layered enforcement, JWT validation, per-subscription rate limiting, shift-left SAST/DAST in CI/CD |
| **Lab 8** | Data Security Architecture Design (Capstone) | 75–90 min | ~$1–$3 | CMK vs. Microsoft-managed vs. HYOK encryption, Purview DSPM automated classification, full Zero Trust architecture synthesis |

**Total time**: ~8–9.5 hours
**Total cost**: ~$3–$11 if done sequentially and cleaned up after each lab (Lab 3's Sentinel ingestion and Lab 7/8's Key Vault Premium SKU are the main cost drivers; none require an always-on hourly-billed appliance like Azure Firewall)

---

## Prerequisites

This track assumes **AZ-500, SC-200, and SC-300-level operational knowledge already in hand** — SC-100 is a capstone architecture certification, not a starting point. If Conditional Access, PIM, Sentinel analytics rules, or Azure Policy initiatives are unfamiliar mechanics rather than known tools you're now learning to choose *between*, complete those tracks first:

- Completion of [AZ-500](../AZ-500/README.md), or equivalent comfort with identity protection, network security, Key Vault, and Defender for Cloud/Sentinel basics
- Completion of [SC-300](../SC-300/README.md), or equivalent comfort with Conditional Access, PIM, and identity governance mechanics
- Familiarity with [SC-200](../SC-200/README.md)-level Sentinel/Defender operational concepts (this track designs the SIEM topology; SC-200 runs the day-to-day detection and response inside it)
- An Azure subscription with **Entra ID P2** enabled (Conditional Access, PIM, and Identity Protection features used throughout)
- **Azure CLI** authenticated (`az login`), with the `security`, `sentinel`, and `purview` extensions installable (`az extension add --name <name>`)
- A Microsoft Graph-capable identity with permission to create Conditional Access policies and PIM role assignments (Global Administrator or a scoped custom role) — used via `az rest` throughout, since several Entra ID and Global Secure Access features don't yet have dedicated `az` CLI command groups
- (Optional, Lab 5) An AWS account for the multicloud connector exercise — the lab uses `<aws-account-id>` as a placeholder and doesn't require deploying AWS workloads, only registering the account with Defender for Cloud

---

## How to Use These Labs

Work them in order — this track tells one continuous story (a breach-driven Zero Trust initiative), and later labs' scenarios explicitly reference earlier labs' decisions and controls. Lab 8's capstone synthesis assumes all seven prior labs were completed.

- **Coming straight from AZ-500/SC-300?** Start with Lab 1 — it's the bridge between "I know how to configure these controls" and "I know which architecture to recommend and why," which is the entire SC-100 exam.
- **Preparing specifically for the exam's design-judgment questions?** Every lab's Part 1 decision matrix is the section to slow down on — the goal is being able to defend the recommendation out loud against a stated set of requirements, not memorizing the hands-on CLI steps.
- **Want the strongest interview story?** Lab 8's synthesis table, showing how seven independent design decisions reinforce one Zero Trust architecture, is the one to walk through if asked "design a security architecture for an organization recovering from a breach."
- **Short on time or budget?** Labs 1, 2, 4, and 5 run at ~$0 and are the highest-yield for exam-domain coverage (identity, governance, and compliance are 45–55% of the exam combined); Labs 3, 6, 7, and 8 carry small costs from Sentinel ingestion, APIM, and Key Vault Premium.

---

## Lab Format

Same format as the other tracks — Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps — with the same addition AZ-305 uses: each lab opens with a **Design Decision** step (Part 1), a comparison matrix of the viable options weighed against criteria like cost, complexity, compliance fit, and Zero Trust alignment, with a justified recommendation, before any hands-on configuration begins. SC-100's decisions are security-architecture-specific — Conditional Access layering, SIEM topology, encryption key custody — rather than AZ-305's general infrastructure scope, but the design-first discipline is identical.

---

## Important Notes

### Cost Management
- Azure Key Vault Premium (Labs 2, 8) and API Management Consumption tier (Lab 7) have no idle hourly charge — they bill per-operation/per-call, but delete the resource group at the end of each lab regardless
- Microsoft Sentinel (Lab 3) bills per GB ingested — this track's labs stay under 1 GB of test data, but a live deployment's ingestion cost scales with real log volume, which is exactly the retention-tiering decision Lab 3 designs for
- No lab in this track deploys Azure Firewall or another always-on hourly-billed network appliance — Lab 6 deliberately designs around Security Service Edge instead, and cross-references [AZ-305 Lab 5](../AZ-305/lab-5-network-infrastructure-design.md) for the hub-spoke pattern where that pattern still applies
- Delete the resource group at the end of every lab; Labs 2 and 8 also involve Entra ID objects (Conditional Access policies, PIM assignments, Key Vault soft-deleted keys) that need their own explicit cleanup, called out in each lab's Cleanup section

### Exam Alignment
These eight labs map to the four SC-100 exam domains: design solutions aligned to security best practices (20–25%), design security operations/identity/compliance capabilities (25–30%), design security solutions for infrastructure (25–30%), and design security solutions for applications and data (20–25%). They're built for design-judgment depth on the highest-yield scenario patterns — credential compromise, data residency constraints, multicloud visibility gaps, API exposure — not exhaustive coverage of every exam objective. SC-100 rewards being able to justify a trade-off against stated requirements, and that's what every lab's Part 1 decision matrix exists to practice.

### Relationship to Other Tracks
SC-100 is deliberately positioned as the security-architecture capstone in this repo, parallel to how AZ-305 caps the general Azure architecture path. It assumes the hands-on mechanics from AZ-500 (security engineering), SC-300 (identity governance), and SC-200 (security operations) are already known — this track is about choosing *between* the things those tracks already taught how to build, the same relationship AZ-305 has to AZ-104/AZ-500/SC-300. If a hands-on step feels unfamiliar rather than a refresher, the earlier track it draws from is called out inline in that lab.

---

## Additional Resources

- [SC-100 Study Guide](https://learn.microsoft.com/en-us/certifications/cybersecurity-architect-expert/)
- [Microsoft Learn SC-100 Path](https://learn.microsoft.com/en-us/training/courses/sc-100t00)
- [Microsoft Cybersecurity Reference Architectures (MCRA)](https://learn.microsoft.com/en-us/security/adoption/mcra)
- [Zero Trust Rapid Modernization Plan (RaMP)](https://learn.microsoft.com/en-us/security/zero-trust/adopt/rapid-modernization-plan)
- [Microsoft Cloud Adoption Framework — Security](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/secure/)
