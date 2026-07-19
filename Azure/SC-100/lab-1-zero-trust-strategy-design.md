# Lab 1: Zero Trust Strategy & Design Decision Frameworks

Check box if done: [ ]

## Overview
Every vendor calls their product "Zero Trust." SC-100 doesn't test whether you can recite the marketing definition — it tests whether you can take a real organization's current state, measure it against the three Zero Trust principles (verify explicitly, use least privilege access, assume breach), and produce a roadmap that a CISO could actually fund and a platform team could actually execute. This lab builds that roadmap from a maturity assessment grounded in real, queryable signals — Microsoft Secure Score and Defender for Cloud's regulatory/posture data — not a slide of aspirational statements.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$1 (Secure Score, Defender for Cloud free posture data, and Conditional Access in report-only mode are free; Part 5's Log Analytics workspace stays under free-tier daily ingestion if cleaned up the same session)

---

## Scenario
You've joined a 2,500-employee organization as its first Cybersecurity Architect. Six weeks ago, an attacker used a phished VPN credential to pivot from a contractor's laptop into the on-prem file server hosting finance records — no MFA on the VPN, no device compliance check, and once inside the corporate network the attacker could reach almost everything because the network was flat and trust was implicit past the perimeter. The board approved a "Zero Trust initiative" without defining what that means operationally. Leadership wants two things from you in the next two weeks: a maturity assessment showing where the organization actually stands today, and a phased roadmap they can put budget against — not a five-year transformation plan that never ships anything.

---

## Objectives
- Apply the three Zero Trust principles (verify explicitly, least privilege access, assume breach) as evaluation criteria rather than slogans
- Decide between a phased/incremental rollout and a big-bang rollout, and justify the choice for a real incident-driven scenario
- Baseline current security posture using Microsoft Secure Score and Defender for Cloud, not self-reported opinion
- Build a Zero Trust maturity matrix across the six defense pillars (Identity, Endpoints, Apps, Data, Infrastructure, Network)
- Translate the highest-leverage gap into one deployable, validated quick-win control
- Design a recurring measurement cadence so the roadmap's progress is provable at each phase boundary, not just asserted

---

## Part 1: Design Decision — Adoption Model: Phased Rollout vs. Big-Bang Rollout

### Decision: How the Zero Trust Initiative Gets Rolled Out

| Factor | Big-Bang (all six pillars simultaneously) | Phased / Crawl-Walk-Run (identity first, then endpoints, then network/data) |
|---|---|---|
| **Risk of business disruption** | High — MFA, device compliance, and network segmentation landing on every user at once produces a flood of help-desk tickets and blocked legitimate work | Lower — each phase is scoped and tested (report-only first) before enforcement, so failures are isolated and diagnosable |
| **Time-to-first-value** | Slow — nothing ships until every pillar's controls are ready, which for a 2,500-person org with legacy AD realistically takes 12+ months before anything is enforced | Fast — the highest-yield control (MFA for privileged accounts, closing the exact gap that caused the breach) can be enforced within days |
| **Resource requirements** | Requires the full security team plus every application/infra owner engaged concurrently — rarely realistic outside a small startup | Concentrates effort on one pillar at a time; matches how most security teams are actually staffed |
| **Executive buy-in over time** | Fragile — a single big-bang failure (a broken CA policy locking out finance during quarter-close) can kill funding for the whole initiative | Durable — each phase's measurable win (Secure Score delta, closed incident-class) builds the case for the next phase's budget |
| **Alignment with Microsoft's Rapid Modernization Plan (RaMP) guidance** | Not aligned — RaMP is explicitly a phased, priority-ordered playbook | Directly aligned — RaMP's own ordering (protect identities and devices first) is what this phased model follows |
| **Fit for this scenario** | Poor — the org just had an incident and needs a credible near-term win, not a multi-quarter blackout before anything changes | Strong — starts exactly where the breach happened (identity, unmanaged devices) and expands outward |

### Recommendation for This Scenario
**Phased rollout following Microsoft's RaMP ordering**: Identity and privileged access first (closes the exact gap the breach exploited), then endpoints/device compliance, then network segmentation and data protection. Each phase runs in **report-only or audit mode before enforcement**, so the rollout produces evidence before it produces disruption. Part 4 of this lab implements the first phase's highest-leverage control end-to-end so the roadmap isn't just a document — it ships something real in week one.

---

## Part 2: Baseline the Current State

You can't build a credible maturity assessment from opinion. Pull the two signals that are actually queryable today: Microsoft Secure Score (identity + endpoint + app posture, from Microsoft 365 Defender) and Defender for Cloud's Secure Score (Azure resource posture).

### Step 1: Confirm CLI Access and Tenant Context
```bash
az login
az account show --query "{subscriptionId:id, tenantId:tenantId}" -o table
```

### Step 2: Pull the Defender for Cloud Secure Score
```bash
az extension add --name security --upgrade

az security secure-scores list -o table
az security secure-score-controls list \
  --query "[].{control:displayName, current:score.current, max:score.max, percentage:score.percentage}" \
  -o table
```
**Expected result**: a list of controls (e.g., "Enable MFA," "Secure management ports," "Remediate vulnerabilities") each with a current/max score. The controls with the largest gap between current and max are the ones with the most leverage — that's roadmap material, not the ones already near 100%.

### Step 3: Pull Microsoft Secure Score via Graph
Microsoft Secure Score covers Entra ID, endpoints, and Microsoft 365 apps — the identity and endpoint pillars Defender for Cloud doesn't fully see.
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/security/secureScores?\$top=1" \
  --query "value[0].{currentScore:currentScore, maxScore:maxScore, percentage: (currentScore/maxScore*100)}"

az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/security/secureScoreControlProfiles?\$top=25" \
  --query "value[].{control:title, category:controlCategory, implementationStatus:implementationStatus}" \
  -o table
```
**Validation checkpoint**: both commands should return real data, not an empty array — if `secureScores` is empty, Secure Score hasn't finished its first calculation cycle yet (can take up to 24 hours in a brand-new tenant).

---

## Part 3: Build the Zero Trust Maturity Matrix

Score each pillar against the three-stage maturity model (Traditional → Advanced → Optimal) using the Part 2 data as evidence, not guesswork.

| Pillar | Current State (Traditional/Advanced/Optimal) | Evidence | Target State (6-Month) |
|---|---|---|---|
| **Identity** | Traditional — no MFA enforcement on VPN/admin accounts, standing admin roles, no Conditional Access | Secure Score control "Ensure all users can complete MFA registration" at ~20% of max; no CA policies exist | Advanced — CA enforcing MFA + device compliance for privileged and remote access, PIM for admin roles |
| **Endpoints** | Traditional — contractor devices unmanaged, no compliance policy gate | Secure Score endpoint category flags unmanaged/non-compliant devices | Advanced — device compliance required before access to sensitive apps |
| **Applications** | Traditional — flat network access, no per-app access boundary | No CA app-scoped policies; VPN grants network-wide reach | Advanced — access scoped per-application, not per-network |
| **Data** | Traditional — file server access controlled by network location, not sensitivity | No sensitivity labeling deployed | Advanced — sensitivity labels on finance data, DLP policy enforced (see Lab 8) |
| **Infrastructure** | Advanced — Azure resources exist with some Defender for Cloud coverage already | Defender for Cloud Secure Score already above 50% on resource hygiene controls | Optimal — regulatory compliance initiative assigned (see Lab 4) |
| **Network** | Traditional — flat network, implicit trust once past VPN | Incident report: lateral movement from contractor device to finance server | Advanced — segmented access, identity-aware per-app connectivity (see Lab 6) |

### Phased Roadmap (RaMP-Aligned)

| Phase | Timeframe | Focus | Primary Control | Success Metric | Maps to Lab |
|---|---|---|---|---|---|
| **Phase 1** | Days 1–30 | Identity quick win | Require MFA + block legacy auth for all users; PIM for privileged roles | Zero standing privileged role assignments; MFA registration >95% | This lab (Part 4) + Lab 2 |
| **Phase 2** | Days 30–90 | Device trust | Require compliant/hybrid-joined device for privileged and remote access | 100% of privileged sign-ins from compliant devices | Lab 2 |
| **Phase 3** | Days 90–150 | Security operations | Centralize detection/response in Sentinel | Mean time to triage under 4 hours for credential-access alerts | Lab 3 |
| **Phase 4** | Days 150–210 | Network segmentation | Replace flat VPN access with identity-aware per-app access | Zero users with network-wide VPN reach | Lab 6 |
| **Phase 5** | Days 210–270 | Data protection | Classify and label finance data, enforce DLP | 100% of finance data discovered and labeled | Lab 8 |

Tying each phase to a measurable success metric (not just a completed activity) is what turns this from a checklist into something a CISO can report against — "Phase 1 is done" is weaker evidence than "standing privileged assignments dropped from 14 to 0."

---

## Part 4: Implement the Phase 1 Quick Win — Require MFA in Report-Only Mode

This is the control that would have stopped the breach: MFA on the credential the attacker phished. Deploying it in **report-only mode first** proves the roadmap produces evidence before it produces enforcement-driven disruption — directly following the Part 1 decision.

### Step 1: Create a Report-Only Conditional Access Policy Requiring MFA
```bash
cat > ca-policy-mfa-reportonly.json << 'EOF'
{
  "displayName": "SC100-Lab1-Require-MFA-AllUsers-ReportOnly",
  "state": "enabledForReportingButNotEnforced",
  "conditions": {
    "users": { "includeUsers": ["All"], "excludeUsers": ["<break-glass-account-object-id>"] },
    "applications": { "includeApplications": ["All"] }
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["mfa"]
  }
}
EOF

az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --body @ca-policy-mfa-reportonly.json \
  --headers "Content-Type=application/json"
```
Always exclude a break-glass account from any Conditional Access policy — a misconfigured policy in enforced mode with no exclusion can lock every admin out of the tenant simultaneously.

### Step 2: Review the Report-Only Impact Before Enforcing
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/auditLogs/signIns?\$filter=conditionalAccessStatus eq 'reportOnlyFailure'&\$top=25" \
  --query "value[].{user:userPrincipalName, app:appDisplayName, status:conditionalAccessStatus}" \
  -o table
```
**Expected result**: this list shows every sign-in that *would have been blocked or challenged* had the policy been enforced. Review it for legitimate service accounts or break-glass gaps before flipping `state` to `enabled` — this is the report-only-first payoff from Part 1's decision.

### Step 3: Confirm the Policy Exists and Is Still Report-Only
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies?\$filter=startswith(displayName,'SC100-Lab1')" \
  --query "value[].{name:displayName, state:state}" -o table
```
**Validation checkpoint**: `state` should read `enabledForReportingButNotEnforced`. Flipping it to `enabled` is a deliberate Phase 1 decision made *after* reviewing Step 2's data, not part of this lab — that decision belongs to the organization, informed by evidence this lab produced.

---

## Part 5: Establish a Recurring Secure Score Review Cadence

A maturity assessment taken once and never revisited decays the moment the org adds a new app or a new admin — Part 1's phased model only works if leadership can see the trendline, not just the day-one snapshot. This closes the loop between "we assessed maturity" and "we can prove maturity is improving."

### Step 1: Create a Log Analytics Workspace to Capture Secure Score Over Time
```bash
az group create --name sc100-lab1-rg --location eastus

az monitor log-analytics workspace create \
  --resource-group sc100-lab1-rg --workspace-name law-sc100-lab1 \
  --location eastus --sku PerGB2018 --retention-time 90
```

### Step 2: Schedule a Recurring Export of Secure Score Data
```bash
cat > secure-score-export-logic.json << 'EOF'
{
  "properties": {
    "triggers": {
      "Recurrence": { "type": "Recurrence", "recurrence": { "frequency": "Week", "interval": 1 } }
    },
    "actions": {
      "Call_Graph_Secure_Score": {
        "type": "Http",
        "inputs": { "method": "GET", "uri": "https://graph.microsoft.com/v1.0/security/secureScores?$top=1" }
      }
    }
  }
}
EOF
```
A weekly Logic App (or equivalently, a scheduled Azure Automation runbook) pulling `secureScores` and writing the result to the Part 5 Step 1 workspace turns Lab 1's one-time baseline into a trendline. Wiring the Logic App's HTTP trigger to Microsoft Graph requires an app registration with `SecurityEvents.Read.All` consented — a one-time setup step outside this lab's scope, referenced here as the design choice rather than built end-to-end.

### Step 3: Build a Workbook Query Against the Trend Data
```bash
az monitor log-analytics query --workspace "$(az monitor log-analytics workspace show \
  --resource-group sc100-lab1-rg --workspace-name law-sc100-lab1 --query customerId -o tsv)" \
  --analytics-query "SecureScoreTrend_CL | summarize avg(CurrentScore_d) by bin(TimeGenerated, 7d) | order by TimeGenerated asc" \
  -o table
```
**Expected result**: once weekly exports have run for a few cycles, this returns a week-over-week trendline — the artifact that answers "is the Zero Trust initiative actually working" with a chart instead of an assertion, and is what turns Part 3's roadmap into something reviewable at each phase boundary in Part 3's roadmap table.

**Validation checkpoint**: confirm the workspace exists and is queryable (`az monitor log-analytics workspace show` returns a `provisioningState` of `Succeeded`); the custom `SecureScoreTrend_CL` table populates only after at least one export cycle runs, so an empty result on day one is expected, not a failure.

---

## Cleanup
```bash
POLICY_ID=$(az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies?\$filter=startswith(displayName,'SC100-Lab1')" \
  --query "value[0].id" -o tsv)

az rest --method DELETE \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$POLICY_ID"

az group delete --name sc100-lab1-rg --yes --no-wait
```
Confirm with `az rest --method GET --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies?\$filter=startswith(displayName,'SC100-Lab1')" --query "value" -o table` — expect an empty array. Confirm the workspace is gone with `az group show --name sc100-lab1-rg`, expecting a "ResourceGroupNotFound" error. No Defender plans were enabled in this lab, so there's nothing else to disable.

---

## What You Practiced

| Concept | Why It Matters |
|---|---|
| Zero Trust as measurable state, not a slogan | SC-100 questions score you on evidence-driven design decisions, not restating "verify explicitly" |
| RaMP-aligned phased rollout | Matches Microsoft's own prescriptive guidance and is directly examinable |
| Report-only mode before enforcement | The single most common way real Conditional Access rollouts fail is skipping this step |
| Secure Score as a design input | Turns an assessment from opinion into something defensible with data |
| Maturity matrix across six pillars | The structure SC-100 scenario questions are built around — expect to be handed a "current state" and asked to identify the gap |
| Recurring measurement cadence | Distinguishes a living architecture practice from a one-time assessment document that goes stale the day it's delivered |

---

## Common Mistakes to Avoid
- **Treating Zero Trust as a product to buy rather than an architecture to design**: no single SKU delivers it; SC-100 tests the cross-pillar design, not product knowledge
- **Enforcing a CA policy without a break-glass exclusion**: this is the single most common way to lock an entire tenant out of Entra ID, including every admin
- **Skipping report-only mode**: enforcing before reviewing sign-in impact turns a security improvement into a self-inflicted outage
- **Building a roadmap with no phase producing a validated result**: a five-phase plan where nothing is provably done until phase 5 won't survive a budget review after an incident
- **Ignoring the pillars that already score well**: Infrastructure in this scenario is already Advanced — spending Phase 1 budget there instead of Identity misallocates effort against the actual attack path

---

## Next Steps
- Continue to [Lab 2: Identity & Access Architecture Design](lab-2-identity-access-architecture-design.md) to turn the Phase 1 quick win into a full Conditional Access + PIM layering strategy
- For the hands-on Conditional Access and PIM mechanics this lab assumes, see [AZ-500 Lab 1: Identity & Access Protection](../AZ-500/lab-1-identity-access-protection.md) and [SC-300 Lab 1: Identity Governance Foundations](../SC-300/lab-1-identity-governance-foundations.md)
- For the general architecture-decision method this track mirrors, see [AZ-305 Lab 1: Identity, Governance & Monitoring Design](../AZ-305/lab-1-identity-governance-monitoring-design.md)
- The operational, day-to-day execution of the roadmap built here is covered by [SC-200](../SC-200/README.md) (security operations), [SC-300](../SC-300/README.md) (identity governance), and [SC-500](../SC-500/README.md) (information protection) — SC-100 designs the strategy those tracks implement
