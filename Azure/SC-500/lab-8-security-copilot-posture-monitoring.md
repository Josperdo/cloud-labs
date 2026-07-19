# Lab 8: Security Copilot & Posture Monitoring (Capstone)

Check box if done: [ ]

## Overview
This is the capstone — it doesn't introduce a new resource type to secure, it ties together every signal produced by Labs 1–7 into one investigation and one monitoring view. You'll provision Microsoft Security Copilot, use it to investigate a real finding from Lab 7's regulatory compliance dashboard, build a custom prompt that pulls context across identity (Lab 1), storage/database (Lab 2), network (Lab 3), compute (Lab 4), AI (Lab 5), and data security posture (Lab 6), and finish with a unified workbook that's the actual artifact you'd hand to a security leadership team as "here's our posture, in one place."

**Estimated time**: 60–90 minutes
**Cost**: ~$4–$10 (Security Copilot bills hourly per Security Compute Unit regardless of usage volume — this is the single most expensive line item in the entire track; provision the minimum capacity, complete the lab in one sitting, and deprovision immediately)

---

## Scenario
Leadership has asked for a single investigation write-up: take the highest-risk finding from Lab 7, explain what it means, what's actually at risk, and what to do about it — in language a non-technical stakeholder can follow. You're also asked to sketch what "one dashboard, everything visible" would look like for this workload going forward, instead of six different people checking six different blades.

---

## Objectives
- Provision **Security Copilot** with minimal **Security Compute Units (SCUs)**
- Use a built-in **promptbook** to investigate a real Defender for Cloud finding
- Build a **custom prompt** that synthesizes context across identity, data, network, compute, AI, and DSPM signals
- Understand SCU billing well enough to avoid an unexpectedly large bill
- Assemble a **unified posture workbook** referencing every prior lab's monitoring surface

---

## Part 1: Provision Security Copilot

### Step 1: Understand SCU Billing Before You Provision Anything
Security Copilot bills by the **Security Compute Unit-hour**, provisioned in a capacity pool, **regardless of how much you actually use it during that time** — it's closer to reserving compute capacity than to a pay-per-query model. Provisioning 1 SCU for one hour to do this entire lab in a single focused session is dramatically cheaper than provisioning it, walking away, and coming back later. Read this whole lab once before provisioning so you're not paying for idle capacity while you read.

### Step 2: Provision the Minimum Capacity
1. Portal → **Microsoft Security Copilot** → **Copilot capacity** (may also be reached via `securitycopilot.microsoft.com`)
2. **+ Create capacity**
3. Select your subscription and resource group (reuse `sc500-lab7-rg` or create `sc500-lab8-rg`)
4. **Number of SCUs**: `1` (the minimum) — sufficient for this lab's promptbook and prompt volume
5. **Create**

**Validation checkpoint**: capacity provisioning typically completes within a few minutes. Confirm status shows **Active** before continuing — don't start the clock on paid capacity and then wait around for it to finish provisioning.

### Step 3: Connect Data Sources (Plugins)
Security Copilot's value comes from the plugins it has access to — without them, it's a general-purpose chat model with no view into your actual environment.

1. **Microsoft Security Copilot** → **Sources** (or **Plugins**)
2. Confirm/enable: **Microsoft Defender for Cloud**, **Microsoft Entra**, **Microsoft Purview**, and **Azure Resource Graph** (naming and exact plugin set vary by rollout) — these are the plugins that let Copilot actually query the resources built across Labs 1–7

---

## Part 2: Investigate a Real Finding with a Promptbook

### Step 4: Pick the Finding to Investigate
Go back to [Lab 7](lab-7-defender-cspm-compliance.md)'s output: the highest-risk item from the attack path analysis (Step 6) or the highest-risk recommendation from the regulatory compliance dashboard (Step 9). If those resources have since been cleaned up, use any current high-risk recommendation from **Defender for Cloud** → **Recommendations** instead — the workflow is what this lab is teaching, not a specific finding.

### Step 5: Run a Built-In Promptbook
1. **Microsoft Security Copilot** → **Promptbooks**
2. Select **"Vulnerability impact assessment"** or **"Summarize an incident"** (exact promptbook names vary by rollout) — pick whichever most closely matches the finding type you selected in Step 4
3. Provide the resource or recommendation ID as input when prompted
4. Run the promptbook and review the generated output — it should include an impact summary, affected resources, and suggested remediation steps

### Step 6: Evaluate the Output Like a Security Engineer, Not Like a User
Promptbook output is a draft, not a verdict. Cross-check at least one factual claim it makes against the actual Defender for Cloud finding (e.g., does the severity it states match what the portal shows; does the affected resource list match). This is the same discipline as reviewing any AI-generated output before it goes to a stakeholder — Copilot accelerates the investigation, it doesn't replace verifying the investigation's conclusions.

**Validation checkpoint**: you should be able to point to one specific claim in the promptbook's output and confirm it against the source data in Defender for Cloud directly.

---

## Part 3: A Custom Cross-Domain Prompt

### Step 7: Why a Custom Prompt Beats Checking Six Blades Manually
Each prior lab left signals in a different place: RBAC/Policy compliance state (Lab 1), Defender for Storage/SQL findings (Lab 2), NSG flow log and firewall data (Lab 3), Defender for Servers/Containers findings (Lab 4), Defender for AI Services alerts (Lab 5), DSPM for AI reports (Lab 6), and the aggregated secure score/compliance/attack-path view (Lab 7). A human pulling all of that together manually is checking six or seven different blades and mentally correlating them. A well-scoped Copilot prompt can pull from several plugin-connected sources in one pass.

### Step 8: Write the Prompt
In the Security Copilot prompt bar:

```
Summarize the current security posture of resources tagged workload=sc500-demo-app
across this subscription. Include:
1. Any RBAC role assignments or Azure Policy non-compliance on these resources
2. Any active Defender for Cloud recommendations, grouped by resource type
   (storage, SQL, network, compute, AI)
3. The current secure score contribution from this resource group
4. Any Defender for AI Services alerts on AI resources in scope
Present the result as a table grouped by security domain, with risk level and
a one-line recommended action per row.
```

### Step 9: Review and Refine
1. Run the prompt and review the output table
2. If a domain comes back empty (e.g., no AI resources currently exist because Lab 5's resource group was cleaned up), that's expected — note it rather than treating it as a failure
3. Refine the prompt once, narrowing or widening scope based on what the first pass returned — prompt iteration is a normal part of getting a useful result, not a sign something's broken

**This output is your actual capstone deliverable**: a single, cross-domain security posture summary generated from one prompt, covering ground that would otherwise require separately opening Entra ID, Defender for Cloud (three different resource-type views), and Purview.

---

## Part 4: Unified Posture Monitoring View

### Step 10: Why a Standing Workbook Beats a One-Time Prompt
The Copilot investigation in Parts 2–3 is point-in-time. A workbook is the thing that exists continuously, so the next person doesn't need to reconstruct this same cross-domain view from scratch.

### Step 11: Build a Workbook Referencing Every Prior Lab's Signal
1. Portal → **Azure Monitor** → **Workbooks** → **+ New**
2. Add a query tile pulling from the Log Analytics workspace built in [Lab 3](lab-3-network-security.md) (`sc500-lab3-law`) for NSG flow log / firewall summary data
3. Add an **Azure Resource Graph** tile querying resources tagged `workload=sc500-demo-app` and their current Defender for Cloud recommendation counts:

```kusto
resources
| where tags['workload'] == 'sc500-demo-app'
| project name, type, resourceGroup, location
```

4. Add a tile summarizing the current secure score (Defender for Cloud has a native workbook template — **Microsoft Defender for Cloud** → **Workbooks** → **Secure Score Over Time** — clone relevant panels into this workbook rather than rebuilding from scratch)
5. Add a text tile linking out to the Purview DSPM for AI report from [Lab 6](lab-6-purview-dspm.md) — DSPM findings aren't natively queryable in an Azure Monitor workbook, so a direct link is the practical integration point today
6. **Save** as `SC500 Unified Posture View`

### Step 12: Pin It Somewhere It'll Actually Get Looked At
1. **Workbooks** → `SC500 Unified Posture View` → **Pin to dashboard**
2. This is the difference between "we have a report" and "we have a report someone opens every week" — pin it to whatever dashboard your team actually looks at day to day

**Validation checkpoint**: the workbook should render without query errors and show at least the resource inventory and secure score tiles populated, even if some domains are sparse due to earlier cleanup.

---

## Part 5: Cleanup

### Step 13: Deprovision Security Copilot Capacity Immediately
This is the most important cleanup step in the entire SC-500 track — SCU billing continues as long as capacity is provisioned, whether or not you're actively using it.

1. Portal → **Microsoft Security Copilot** → **Copilot capacity**
2. Select the capacity pool created in Step 2 → **Delete**

```bash
# If capacity was provisioned via a resource group, confirm it's gone
az resource list --resource-group sc500-lab8-rg --resource-type "Microsoft.SecurityCopilot/capacities" -o table
```

**Validation checkpoint**: the query above should return no results after deletion. Don't consider this lab closed until you've confirmed this.

### Step 14: Clean Up Remaining Resources
```bash
az group delete --name sc500-lab8-rg --yes --no-wait

# Sweep for any earlier labs' resource groups still lingering
az group list --query "[?starts_with(name, 'sc500-')].name" -o tsv
```

Delete any remaining `sc500-*` resource groups from Labs 1–7 you haven't already cleaned up, and confirm every subscription-level Defender pricing tier set to `Standard` across the track is reset:

```bash
for plan in StorageAccounts SqlServers VirtualMachines Containers AI CloudPosture; do
  az security pricing create --name $plan --tier Free
done
```

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **SCU-based capacity billing** | Understanding hourly-regardless-of-usage billing is what prevents an accidental large bill — the single costliest mistake possible in this entire track |
| **Promptbook-driven investigation** | Accelerates a standard investigation pattern (impact assessment, incident summary) without replacing the analyst's judgment |
| **Verifying AI-generated findings against source data** | The same "don't trust, verify" discipline applied throughout this track, now applied to the AI tool itself |
| **Custom cross-domain prompts** | Synthesizes identity, data, network, compute, AI, and DSPM signals that otherwise live in five or six separate portal blades |
| **A standing unified posture workbook** | Converts a one-time investigation into a continuously available view the next person doesn't have to rebuild |

---

## Common Mistakes to Avoid
- **Provisioning Security Copilot capacity before reading through the full lab**: SCU billing starts the moment capacity is active — read first, provision second, and work through the lab in one focused session
- **Leaving Security Copilot capacity active "in case I need it again"**: this is the most expensive idle resource in the whole track by far; deprovision the same session
- **Treating promptbook or custom prompt output as ground truth without spot-checking it**: Copilot synthesizes from connected sources but can still misstate severity or miss context — verify at least one claim against the source
- **Writing an overly broad first prompt and giving up when it returns too much or too little**: prompt refinement is expected, not a sign of failure
- **Building the workbook but never pinning or sharing it**: an unpinned workbook that nobody revisits provides the same value as no workbook at all

---

## Next Steps
- This closes out the SC-500 track. For deeper identity governance underneath everything built here, see [SC-300](../SC-300/README.md); for the security-engineering fundamentals SC-500 assumes, see [AZ-500](../AZ-500/README.md), the certification SC-500 succeeds
- For architecture-level security strategy — how these workload-level controls fit into an organization-wide Zero Trust and security posture strategy — continue to **SC-100: Cybersecurity Architect Expert** (not yet built in this repo; a natural next track to add)
- Extend the unified workbook with a scheduled export/email so the posture summary reaches stakeholders automatically instead of requiring someone to open it
- Since SC-500 is still in beta as of mid-2026, periodically re-check the [current skills outline](https://learn.microsoft.com/en-us/credentials/certifications/cloud-ai-security-engineer-associate/) — beta exams are the most likely to have their objective weighting revised before GA
