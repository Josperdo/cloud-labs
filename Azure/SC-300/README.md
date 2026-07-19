# SC-300: Identity and Access Administrator — Hands-On Labs

Eight labs covering the core SC-300 domains: identity governance, application integration/SSO, access governance, identity protection with automation, hybrid identity, external identities, workload identity/application lifecycle governance, and lifecycle workflows at scale. Where AZ-500 focuses on identity as a *security* control, SC-300 goes deeper on identity as an *administrative* discipline — the day-to-day work of running an identity platform for an organization: who gets what role, how apps get onboarded, how access gets periodically re-certified, how non-employee and non-human identities get governed, and how to automate the parts that don't scale by hand.

These labs assume you're comfortable with Entra ID basics (users, groups, RBAC) — see [AZ-104 Lab 4](../AZ-104/lab-4-identity.md) if not.

---

## Labs Overview

| Lab | Domain | Duration | Cost | Key Skills |
|-----|--------|----------|------|-------------|
| **Lab 1** | Identity Governance Foundations | 45–60 min | ~$0 | Entra ID role model, custom roles, administrative units |
| **Lab 2** | Application Integration & SSO | 60–75 min | ~$0 | App registrations vs enterprise apps, SAML/OIDC, app roles, consent |
| **Lab 3** | Access Governance | 60–75 min | ~$0 (P2 trial) | Access reviews, entitlement management, PIM for groups |
| **Lab 4** | Identity Protection & Graph Automation | 60–75 min | ~$0 (P2 trial) | Advanced Conditional Access, risk policies, Microsoft Graph PowerShell automation |
| **Lab 5** | Hybrid Identity with Microsoft Entra Connect | 60–75 min | ~$0 (P1 required) | Sync scoping/filtering, PHS vs. PTA, staged rollout, sync health monitoring |
| **Lab 6** | External Identities | 60–75 min | ~$0 | B2B collaboration, cross-tenant access settings, B2B direct connect, guest access reviews |
| **Lab 7** | Workload Identity & Application Lifecycle Governance | 60–75 min | ~$0 | App registration ownership, federated credentials, Graph credential auditing, consent governance |
| **Lab 8** | Lifecycle Workflows & Entitlement at Scale (Capstone) | 75–90 min | ~$0 (Governance license; optional Logic App is fractions of a cent) | Joiner/mover/leaver automation, custom task extensions, cross-lab governance synthesis |

**Total time**: ~8–10 hours
**Total cost**: ~$0 — every SC-300 lab uses Entra ID features only, no billed Azure compute/storage beyond a few cents of optional Logic App execution in Lab 8

---

## Prerequisites

- Azure subscription with an Entra ID tenant, **Entra ID P2 trial** enabled (Labs 3–4, and recommended tenant-wide before Lab 5 — see [AZ-500 Lab 1](../AZ-500/lab-1-identity-access-protection.md) prerequisites for how to activate it)
- **Entra ID P1** license, at minimum, for Lab 5's staged rollout feature (included in the P2 trial above)
- **Entra ID Governance** license for Lab 8's Lifecycle Workflows — the P2 trial typically includes a time-limited Governance trial; verify under **Entra ID** → **Licenses** → **All products** before starting Lab 8, and read Parts 2–4 as a configuration walkthrough if it isn't available in your tenant
- Global Administrator or a delegated administrative role on the tenant
- **Microsoft Graph PowerShell SDK** installed for Labs 4, 5, and 7: `Install-Module Microsoft.Graph -Scope CurrentUser`
- A Windows Server + on-premises (or simulated) Active Directory domain for the fully hands-on portions of Lab 5's Microsoft Entra Connect installer — Lab 5's Parts 3–4 (staged rollout, sync monitoring) run against any Entra ID tenant without one
- A GitHub repository (or willingness to work through the steps conceptually) for Lab 7's workload identity federation section
- Completion of, or comfort with, basic Entra ID user/group/role administration

---

## How to Use These Labs

- **New to identity administration?** Work Labs 1–8 in order — each one builds on identity objects (users, groups, apps) and concepts (least privilege, access reviews, consent) established in earlier labs. Lab 8 explicitly pulls together Labs 1, 3, 4, and 6.
- **Coming from AZ-500?** You've already touched PIM and Conditional Access at a security-control level in [AZ-500 Lab 1](../AZ-500/lab-1-identity-access-protection.md) — these labs go deeper on the *administration* side: custom roles, app lifecycle, access certification, hybrid sync, and external/workload identity governance, which AZ-500 doesn't cover.
- **Prepping for interviews?** Lab 4's Graph automation script and Lab 7's credential-expiry reporting are the strongest talking points — "I don't just click through the Entra portal, I can query and remediate identity data at scale via API." Lab 8's cross-lab synthesis is the best answer to "walk me through how you'd design identity governance for an organization."
- **Short on a P2/Governance trial?** Labs 1, 2, and 6 need no premium licensing at all — start there, and read the license-gated portions of Labs 3–5, 7, and 8 as configuration walkthroughs if a trial isn't available.

---

## Lab Format

Same format as the other tracks: Overview → Scenario → Objectives → step-by-step walkthrough → validation checkpoints → cleanup → key concepts table → common mistakes → next steps.

---

## Important Notes

### Licensing
Labs 3 and 4 require **Entra ID P2** for access reviews on PIM-eligible roles, entitlement management, and Identity Protection risk policies. Lab 5's staged rollout requires **Entra ID P1** (included in P2). Lab 8's Lifecycle Workflows require the **Entra ID Governance** license tier specifically — activate a 30-day P2 trial before starting Lab 3 if you haven't already (see AZ-500 Lab 1 prerequisites), and confirm Governance licensing separately before Lab 8.

### Cost
Every lab in this track is Entra ID / Microsoft Graph only — there's no billed Azure compute or storage anywhere in SC-300, with one narrow exception: Lab 8's optional custom task extension runs a Logic App on the Consumption plan, which costs a fraction of a cent for the handful of test executions the lab triggers and should be deleted during cleanup. Cost management here is otherwise about license tiers, not resource cleanup.

### Exam Alignment
These eight labs map to the four SC-300 exam domains, each reinforced by at least two labs: **implement identity management** (Labs 1, 5), **implement authentication and access management** (Labs 4, 6), **implement access management for apps** (Labs 2, 7), and **plan and implement identity governance** (Labs 3, 8). As with the other tracks, they're built for defensible hands-on depth on the highest-yield scenarios, not exhaustive coverage of every exam objective.

---

## Additional Resources

- [SC-300 Study Guide](https://learn.microsoft.com/en-us/certifications/identity-and-access-administrator/)
- [Microsoft Learn SC-300 Path](https://learn.microsoft.com/en-us/training/courses/sc-300t00)
- [Microsoft Graph PowerShell SDK docs](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)
