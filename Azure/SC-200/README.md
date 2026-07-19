# SC-200: Microsoft Security Operations Analyst — Hands-On Labs

Eight labs covering the core SC-200 exam domains: managing the security operations environment, responding to incidents, and threat hunting — built around Microsoft Sentinel and Microsoft Defender XDR. This track maps directly to SC-200's three exam domains (Manage a security operations environment 40–45%, Respond to incidents 35–40%, Perform threat hunting 20–25%) and is the SOC-analyst counterpart to [AZ-500](../AZ-500/README.md) (security engineering — building and hardening controls) and feeds into [SC-100](../SC-100/README.md) (security architecture — designing the SIEM/XDR strategy these labs operate day to day).

AZ-500 Lab 4 already stood up a Log Analytics workspace, enabled Sentinel, and built one analytics rule at a security-engineer level — "deploy the platform and prove it works." This track starts where that leaves off and goes deep on the analyst's actual job: writing KQL fluently, engineering detections with entity mapping and MITRE ATT&CK coverage, working incidents end to end, automating response with SOAR playbooks, and proactively hunting for what scheduled rules miss.

These labs assume you're comfortable with basic Azure navigation and CLI (see [AZ-104](../AZ-104/README.md) if not) and benefit from — but don't strictly require — having done [AZ-500 Lab 4](../AZ-500/lab-4-security-ops-automation.md) first.

---

## Labs Overview

| Lab | Focus Area | Duration | Cost | Key Skills |
|-----|-----------|----------|------|-------------|
| **Lab 1** | Sentinel Workspace & Data Connectors | 60–75 min | ~$0–$1 | Log Analytics workspace design, Sentinel onboarding, Azure Activity + Entra ID sign-in/audit connectors |
| **Lab 2** | KQL Fundamentals for Security Analysts | 60–75 min | ~$0 | Filtering, `summarize`, `join`, `parse`, time-window pivoting, saved functions |
| **Lab 3** | Sentinel Analytics Rules & Detection Engineering | 75–90 min | ~$1–$3 | Scheduled analytics rules, entity mapping, alert grouping, MITRE ATT&CK mapping, tuning |
| **Lab 4** | Microsoft Defender XDR — Endpoint & Identity | 60–90 min | ~$0 | Defender for Endpoint onboarding, Defender for Identity signals, unified XDR incident view |
| **Lab 5** | Incident Response & Investigation | 60–75 min | ~$1–$3 | Triage, entity investigation graph, evidence collection, incident classification |
| **Lab 6** | SOAR Automation with Logic Apps Playbooks | 75–90 min | ~$1–$5 | Automation rules, Logic Apps playbooks, auto-remediation with guardrails |
| **Lab 7** | Threat Hunting with Sentinel & Advanced Hunting | 60–75 min | ~$0–$1 | Hunting queries, bookmarks, livestream, hunting-to-detection pipeline |
| **Lab 8** | Defender for Cloud Apps & Cross-Workload Correlation (Capstone) | 90–120 min | ~$1–$5 | Cloud App policies, cross-product KQL correlation, capstone investigation |

**Total time**: ~9.5–12.5 hours
**Total cost**: ~$3–$18 if done sequentially and cleaned up after each lab

---

## Prerequisites

- Azure subscription with Owner or Contributor access (Log Analytics workspace and Sentinel deployment)
- **Entra ID P1/P2 trial** enabled — P1 is required for Entra ID diagnostic settings (sign-in/audit log export) used from Lab 1 onward; P2 sharpens the identity-risk signals referenced in Labs 4 and 8
- **Microsoft Defender XDR / Microsoft 365 E5 trial** (or standalone Defender for Endpoint P2 + Defender for Identity trials) for Labs 4, 5, and 8 — these labs use `<placeholder>` values throughout since onboarding requires your own tenant and device
- **Azure CLI** authenticated (`az login`) with the `sentinel` extension (`az extension add --name sentinel`)
- Comfort with basic KQL is helpful but not required — Lab 2 teaches it from the ground up
- Completion of [AZ-500 Lab 4](../AZ-500/lab-4-security-ops-automation.md) is recommended but not required; this track re-teaches the workspace/Sentinel basics at analyst depth in Lab 1

---

## How to Use These Labs

Work them in order — each lab builds on the workspace, connectors, and detection created in the ones before it, and Labs 5–8 assume the Sentinel foundation from Labs 1–3 exists. If you're starting fresh partway through, each lab notes the minimum setup needed to catch up.

- **Coming straight from AZ-500?** Lab 1 will feel familiar (you've deployed a workspace and enabled Sentinel before) — the difference here is analyst depth: connector health, ingestion tuning, and query validation, not just "enable it and move on."
- **Weak on KQL?** Don't skip Lab 2 — nearly every later lab assumes you can read and modify a `summarize`/`join` query without hand-holding.
- **Want the SOC-analyst-to-security-engineer story for interviews?** Point to Lab 6 (SOAR automation) alongside AZ-500 Lab 4's Terraform baseline — "I can both detect and codify the response."

---

## Lab Format

Same format as AZ-500 and SC-300: Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps.

---

## Important Notes

### Licensing
Entra ID diagnostic settings (sign-in and audit log export to Log Analytics, used starting in Lab 1) require **Entra ID P1** at minimum. Defender for Endpoint, Defender for Identity, and Defender for Cloud Apps (Labs 4 and 8) require their own trials or an **M365 E5 trial**, which bundles all three. Activate whichever trials you need before starting the lab that requires them — each lab's Prerequisites-equivalent note calls out exactly what's needed.

### Cost Management
- Log Analytics ingestion and retention (Lab 1 onward) bill **per GB per day** — Sentinel's free 31-day trial covers up to 10 GB/day, which is far more than these labs generate, but confirm your trial is still active
- Logic Apps consumption plan (Lab 6) bills **per action execution** — a handful of test runs costs pennies, but don't leave a playbook wired to a noisy production rule
- Delete the resource group and disable connectors at the end of every lab — Defender/Cloud Apps trial licenses don't bill per-resource, but Log Analytics ingestion and Logic Apps executions do

### Exam Alignment
These eight labs map to SC-200's three exam domains: managing the security operations environment (Labs 1–4), responding to incidents (Labs 5–6), and threat hunting (Labs 7–8). They're not exhaustive of every exam objective — they're built to give you defensible, explainable hands-on experience with the highest-yield scenarios a working SOC analyst actually does.

---

## Additional Resources

- [SC-200 Study Guide](https://learn.microsoft.com/en-us/certifications/exams/sc-200/)
- [Microsoft Learn SC-200 Path](https://learn.microsoft.com/en-us/training/courses/sc-200t00)
- [Microsoft Sentinel Documentation](https://learn.microsoft.com/en-us/azure/sentinel/)
- [KQL Quick Reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
