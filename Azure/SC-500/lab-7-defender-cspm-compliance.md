# Lab 7: Defender for Cloud — CSPM & Regulatory Compliance

Check box if done: [ ]

## Overview
Labs 1–5 built individual controls — RBAC, Policy, encryption, network isolation, compute hardening, AI-specific safeguards. This lab steps back and looks at the whole workload the way an auditor or a CISO would: what's the aggregate secure score, where are the highest-risk attack paths across everything you built, and how does this environment map against a named regulatory framework. It's the lab that turns a pile of individually-good controls into a defensible, reportable security posture.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$2 (Defender CSPM's core capabilities are included once Defender plans from earlier labs are active; this lab reuses those rather than enabling new paid plans)

---

## Scenario
Leadership wants two things before this workload goes to production: a secure score they can track over time, and proof it satisfies a specific regulatory framework the company is contractually obligated to follow. You're enabling Defender CSPM's advanced features, walking the attack path analysis to find the highest-risk chain across the resources built in Labs 1–5, reviewing the regulatory compliance dashboard, and setting up governance rules so findings don't just sit in a list forever with no owner.

---

## Objectives
- Enable **Defender CSPM** and understand what it adds beyond Foundational CSPM
- Interpret the **secure score** and prioritize recommendations by actual risk, not just count
- Use **attack path analysis** to find the highest-risk chain across resources from prior labs
- Review the **regulatory compliance dashboard** against a named framework
- Build a **governance rule** that assigns ownership and a due date to findings

---

## Part 1: Enable Defender CSPM

### Step 1: Foundational CSPM vs. Defender CSPM
Every subscription gets **Foundational CSPM** for free — secure score, recommendations, and basic asset inventory. **Defender CSPM** is a paid add-on that adds **attack path analysis**, **cloud security graph** (the relationship model connecting resources, identities, and data across the environment), **agentless vulnerability scanning**, and **security governance** (the ownership/due-date workflow in Part 4). This is the same free-vs-paid distinction from AZ-500 Lab 4's Defender for Storage comparison, applied at the whole-subscription posture level instead of a single resource type.

```bash
az security pricing create --name CloudPosture --tier Standard
```

### Step 2: Confirm Coverage of the Workload Built Across Labs 1–5
```bash
az security pricing list --query "[].{name:name, tier:pricingTier}" -o table
```

Confirm `CloudPosture` shows `Standard`, alongside `StorageAccounts`, `SqlServers`, `VirtualMachines`, `Containers`, and `AI` from earlier labs if those resource groups still exist. If you've cleaned up prior labs' resource groups already, that's fine — this lab's findings will simply reflect whatever resources are currently live; the workflow is what matters, not a specific finding count.

---

## Part 2: Secure Score and Recommendation Prioritization

### Step 3: Understand How Secure Score Is Weighted
Secure score is not a simple percentage of recommendations resolved — each recommendation is weighted by its potential security impact, and score is calculated per resource type and rolled up. A single high-impact recommendation (e.g., "MFA should be enabled on accounts with owner permissions") can be worth more to your score than ten low-impact ones combined.

1. Portal → **Microsoft Defender for Cloud** → **Overview** → note the current secure score percentage and the point breakdown by security control category
2. **Recommendations** → sort by **Risk level** (Defender CSPM's risk-based prioritization, not just secure score points) — this factors in exploitability and business impact (e.g., internet exposure, sensitive data presence) on top of the raw score weight

### Step 4: Compare Risk Level to Raw Score Weight on One Finding
Pick a recommendation flagged as **High risk** and check its score impact — it's common to find a high-risk item (because it's internet-exposed or touches sensitive data) that carries only moderate score points, and vice versa. This is exactly why Defender CSPM's risk-based view exists: **triage by risk level, not by score points alone.**

---

## Part 3: Attack Path Analysis

### Step 5: Why Attack Paths Matter More Than Isolated Findings
A single misconfiguration in isolation might look low-severity. The same misconfiguration, chained with two others across identity, network, and data, can form a complete path from "internet-exposed resource" to "sensitive data exfiltration." Attack path analysis is Defender CSPM's cloud security graph doing exactly that chaining automatically, instead of you manually reasoning across a dozen unrelated-looking recommendations.

### Step 6: Review Attack Paths Across the Lab Workload
1. **Microsoft Defender for Cloud** → **Attack path analysis**
2. Filter or search for paths touching resources from this track (resource group names starting `sc500-`)
3. Open the highest-risk path shown — it will typically render as a chain, e.g.: *Internet-exposed VM → has a vulnerability → has a high-privilege managed identity → can access a storage account containing sensitive data*

**If no multi-hop path currently exists in your environment** (likely if you've been diligent about cleanup between labs), this is itself the correct outcome — it means the isolation and least-privilege patterns from Labs 1–5 successfully broke the chain. Note in your own words *which* control from an earlier lab would have broken which link, e.g., "the JIT VM access from Lab 4 removes the 'internet-exposed VM' starting point that a real attack path analysis would otherwise flag."

### Step 7: Use the Cloud Security Graph Explorer
1. **Microsoft Defender for Cloud** → **Cloud security explorer**
2. Run a query for a specific risky pattern, e.g.: resources with a **managed identity** that has a **high-privilege role assignment** AND is **exposed to the internet**
3. This is the proactive version of Step 6 — instead of waiting for Defender to surface a pre-built attack path, you're querying the graph directly for a pattern you specifically care about

### Step 8: Exempt a Finding That's a Deliberate, Documented Exception
Not every recommendation should be remediated — sometimes a finding is a known, accepted risk (e.g., a lab VM intentionally left without a specific agent because the workload doesn't support it). Marking it "won't fix" silently is worse than not tracking it at all; a formal exemption keeps it visible with a documented reason instead of disappearing from view.

1. Open any recommendation from Part 2 → **Exempt**
2. **Exemption scope**: the specific resource, not the whole subscription
3. **Exemption category**: **Risk accepted** (or **False positive**, if that's the actual reason)
4. **Justification**: a specific, reviewable reason — e.g., "Lab VM decommissioned within 24 hours; risk window accepted for the duration of the exercise"
5. **Expiration date**: set one — an exemption without an expiration tends to become permanent by default, which defeats the point of documenting it as a deliberate, time-bound exception

**Validation checkpoint**: the exempted recommendation should now show status **Exempted** rather than disappearing from the recommendations list entirely — it stays visible with its justification attached, which is the audit trail a real compliance review would expect to see.

---

## Part 4: Regulatory Compliance Dashboard

### Step 8: Add a Compliance Standard
```bash
az security regulatory-compliance-standards list --query "[].name" -o tsv
```

1. Portal → **Microsoft Defender for Cloud** → **Regulatory compliance** → **Manage compliance policies**
2. Select your subscription → **Add more standards** → choose a framework relevant to a real engagement, e.g., **NIST SP 800-53 Rev. 5** or **ISO 27001:2013**
3. **Add**

### Step 9: Review Control Mapping Against the Workload
1. **Regulatory compliance** → open the newly added standard
2. Expand a control family (e.g., **Access Control** or **System and Communications Protection**) and note which underlying Defender for Cloud recommendations are mapped to it — this is the same recommendation data from Part 2, re-organized around a framework's control structure instead of a raw priority list
3. Identify at least one control that's satisfied specifically because of a Lab 1–5 decision — e.g., a "network segmentation" control satisfied by the Private Endpoints from Lab 3, or an "encryption key management" control satisfied by the customer-managed key from Lab 2

**This mapping is the actual deliverable in a real compliance engagement**: leadership and auditors don't want a list of Azure recommendations, they want "here is our current state against the framework we're contractually bound to."

### Step 10: Export the Compliance Report
1. **Regulatory compliance** → **Download report** (PDF)
2. This is the artifact you'd actually hand to an auditor or include in a compliance package — note what it contains (overall pass rate per standard, control-by-control status) versus what it doesn't (it reports Azure resource configuration compliance; it does not by itself constitute a full compliance attestation, which typically requires additional non-technical evidence)

---

## Part 5: Governance Rules — Assigning Ownership

### Step 11: Why Findings Need Owners and Due Dates, Not Just a List
A recommendation sitting in a dashboard with no assigned owner and no deadline is a recommendation that, in practice, never gets fixed. Governance rules turn Defender for Cloud from a read-only dashboard into an actual accountability workflow.

1. **Microsoft Defender for Cloud** → **Environment settings** → **Governance rules** → **+ Create governance rule**
2. **Scope**: your subscription
3. **Conditions**: apply to recommendations with **Risk level: High**
4. **Assign owner**: by resource tag — use the `workload` tag applied throughout this track (`workload=sc500-demo-app`) mapped to a specific owner email/group
5. **Set remediation timeframe**: `7 days` for high-risk findings
6. **Enable notifications**: weekly digest email to the assigned owner
7. **Create**

### Step 12: Verify the Rule Applies
```bash
az security recommendation-metadata list --query "[?contains(recommendationDisplayName, 'workload')]" -o table 2>/dev/null || \
  echo "Verify via Portal: Governance rules > select the rule > Recommendations affected"
```

**Validation checkpoint**: Portal → **Governance rules** → select the rule you created → **Recommendations affected** should list findings on resources carrying the `workload=sc500-demo-app` tag, each with an assigned owner and countdown to the remediation deadline.

### Step 13: Add a Second Rule for Lower-Risk Findings with a Longer Window
Not every finding deserves the same urgency. Mirror Step 11 with a second rule for medium-risk findings, but with a longer remediation window — this is how a real governance program avoids the two failure modes of "everything is a 7-day fire drill" (which trains owners to ignore deadlines) and "nothing has a deadline at all" (which means nothing gets fixed).

1. **Governance rules** → **+ Create governance rule**
2. **Conditions**: **Risk level: Medium**
3. **Remediation timeframe**: `30 days`
4. **Create**

**Why two tiers instead of one blanket rule**: a single 7-day deadline applied to every finding regardless of severity either gets ignored (too aggressive for low-risk items) or masks genuinely urgent findings in the same queue as routine ones. Tiered timeframes keep the deadline meaningful.

### Step 14: Track Governance Rule Compliance Over Time
1. **Governance rules** → **Governance report** (or **Overview**, depending on portal version)
2. Review the **overdue** count — recommendations past their governance rule deadline with no remediation — this is the single most useful number to bring to a leadership review, more useful than the raw secure score alone, because it measures whether the accountability process is actually working, not just whether the environment is currently well-configured

---

## Part 6: Cleanup

```bash
# Governance rules and compliance standard selections are subscription-scoped settings, not billed resources —
# remove them if this is a disposable lab subscription
az security pricing create --name CloudPosture --tier Free
```

1. Portal → **Governance rules** → delete the rule created in Part 5 if not continuing to use this subscription
2. **Regulatory compliance** → **Manage compliance policies** → remove the added standard if not continuing
3. If any earlier labs' resource groups (`sc500-lab1-rg` through `sc500-lab5-rg`) are still present, confirm they're deleted — this lab's findings depend on what's live, and leftover resources from earlier labs continue to bill and to count against the secure score

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **Defender CSPM vs. Foundational CSPM** | Understanding the paid tier's specific additions (attack paths, graph, governance) is what lets you justify the cost to a budget owner |
| **Risk-based recommendation prioritization** | Score-weight and actual exploitability/impact risk aren't the same axis — triage by risk level, not just point value |
| **Attack path analysis** | Chains isolated, individually low-severity findings into the full exploit path an attacker would actually use |
| **Regulatory compliance dashboard mapping** | Translates raw Azure recommendations into the control-family language an auditor or compliance framework actually speaks |
| **Governance rules (owner + due date)** | Converts a static findings list into an accountable, trackable remediation workflow |

---

## Common Mistakes to Avoid
- **Chasing secure score points without checking risk level**: a low-point, high-risk finding (e.g., something internet-exposed) deserves priority over several high-point, low-risk ones
- **Treating a clean attack path view as "nothing to check"**: it's worth confirming *why* it's clean (which specific controls broke which links) rather than assuming Defender simply hasn't found anything yet
- **Downloading a compliance report and treating it as a complete attestation**: it reports technical configuration state against mapped controls — real compliance typically also requires policy documentation, training records, and other non-technical evidence Defender for Cloud doesn't track
- **Creating recommendations with no governance rule behind them**: a finding without an owner and a deadline has no accountability mechanism and tends to go stale
- **Forgetting that Defender CSPM findings depend on what resources currently exist**: leftover resource groups from earlier labs skew both the secure score and the attack path results

---

## Next Steps
- Continue to [Lab 8: Security Copilot & Posture Monitoring (Capstone)](lab-8-security-copilot-posture-monitoring.md) to investigate one of this lab's findings using Security Copilot and tie every prior lab's signals into one monitoring view
- Revisit [AZ-500 Lab 4](../AZ-500/lab-4-security-ops-automation.md) for the Sentinel-side detection and incident triage workflow that complements this lab's posture/compliance workflow — CSPM tells you what's misconfigured; Sentinel tells you what's actively happening
- Automate the Terraform-based Defender plan enablement pattern from AZ-500 Lab 4 to also enable `CloudPosture` at deployment time, so Defender CSPM is on from day one of any new subscription
- Add a second regulatory standard relevant to your own industry and compare control overlap — many frameworks share a large percentage of underlying technical controls
