# Lab 4: Regulatory Compliance & Governance Architecture

Check box if done: [ ]

## Overview
The org from Labs 1–3 now has layered identity controls and a tiered SOC — but none of that satisfies an auditor by itself. Regulators and customers want evidence: a mapped control framework, a measurable compliance score, and proof that technical controls are actually enforced, not just documented in a policy binder. This lab designs a compliance program structure — how controls get mapped once and reused across multiple regulations instead of re-assessed from scratch each time — then implements the automated enforcement layer with Azure Policy.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (Azure Policy assignment and evaluation are free; this lab doesn't create billable resources beyond what's needed to generate a compliance signal)

---

## Scenario
The org is now pursuing two customer contracts that each require a different compliance attestation — one demands ISO 27001, the other NIST SP 800-53 (a US federal-adjacent client). The compliance team's first instinct is to run two entirely separate assessment projects, each interviewing the same platform team about the same MFA policy, the same encryption-at-rest setting, and the same network segmentation, because the two frameworks happen to score those controls differently. You've been asked to design a compliance program that scales to a third and fourth framework next year without doubling the audit burden each time, and to make sure the technical half of "compliant" is actually verifiable, not just attested to on a form.

---

## Objectives
- Decide between a framework-per-regulation compliance program and a unified control framework with framework-specific mappings
- Decide how administrative/procedural attestation (Purview Compliance Manager) and automated technical enforcement (Azure Policy) fit together rather than substitute for each other
- Assign a built-in regulatory compliance Policy initiative and read its live compliance state
- Map a Policy control to the Purview Compliance Manager improvement action it evidences
- Validate that the compliance score reflects real resource state, not a static checklist
- Escalate one reviewed, fully-compliant control from audit to enforced without flipping the entire initiative at once

---

## Part 1: Design Decision — Compliance Program Structure and Enforcement Mechanism

### Decision 1: Framework-Per-Regulation vs. Unified Control Framework

| Factor | Framework-Per-Regulation (separate assessment per regulation, controls re-evaluated from scratch each time) | Unified Control Framework (map organizational controls once, reuse evidence across every regulation's own scoring) |
|---|---|---|
| **Audit overhead as frameworks are added** | Grows linearly (or worse) — a third and fourth framework each repeat interviews and evidence-gathering for controls already assessed under the first two | Grows sub-linearly — most technical controls (MFA, encryption, logging) map to multiple frameworks; only the framework-specific gaps need new evidence |
| **Control duplication** | High — "require MFA" gets documented, evidenced, and tracked three separate times for three frameworks | Low — one control record, multiple framework mappings pointing to the same evidence |
| **Evidence reuse** | None by design — each framework's assessment starts cold | Direct — Microsoft Purview Compliance Manager's control mapping is built exactly for this, reusing improvement actions across template assessments |
| **Consistency risk** | High — nothing stops the ISO assessment finding "MFA compliant" while the NIST assessment (assessed independently, maybe by a different team) finds a gap in the same control | Low — one source of truth per control reduces the chance of contradictory findings on the same technical state |
| **Fit for this scenario** | Poor — the org is about to run this exercise for two frameworks now and is explicitly planning to add more | Strong — designed for exactly the multi-framework growth this org is facing |

### Decision 2: Manual Attestation Only vs. Azure Policy Automated Enforcement vs. Combined

| Factor | Manual Attestation Only (compliance team fills out a questionnaire, no technical verification) | Azure Policy Regulatory Compliance Initiative (automated, continuous technical control evaluation) | Combined — Purview Compliance Manager for administrative/procedural controls + Azure Policy for technical controls |
|---|---|---|---|
| **Evidence quality** | Weak — "we require encryption" can be true in policy documents and false in three storage accounts nobody checked | Strong for technical controls — Policy evaluates actual resource configuration continuously, not a point-in-time claim | Strongest — technical controls get continuous automated proof, procedural/administrative controls (background checks, training, incident response plans) get structured tracking Policy can't evaluate |
| **Coverage** | Covers everything, shallowly | Only covers controls Azure Policy can technically evaluate — access reviews, training records, and vendor contracts are outside its scope | Full coverage — each control type is verified by the mechanism actually capable of verifying it |
| **Audit defensibility** | Weakest — "we attest that we do this" is the first thing an auditor probes and the easiest to fail | Strong for the technical half — a compliance report generated from live resource state is hard evidence | Strongest — auditors get automated evidence for technical controls and structured (even if manual) evidence for everything else |
| **Fit for this scenario** | Insufficient alone for either ISO 27001 or NIST 800-53, both of which expect technical evidence | Necessary but not sufficient — doesn't cover procedural controls at all | Correct fit |

### Note: Assignment Scope — Management Group vs. Subscription
Both regulatory initiatives in this lab are assigned at subscription scope for simplicity, but a multi-subscription landing zone (the [AZ-305 Lab 1](../AZ-305/lab-1-identity-governance-monitoring-design.md) hierarchy this track's org would eventually adopt) should assign them at the management group level instead — one assignment inherited by every subscription underneath, rather than a separate assignment repeated per subscription that can drift out of sync as subscriptions are added. The compliance program's scalability depends on this the same way Lab 1's identity roadmap depended on management-group-level policy assignment: assign once at the highest correct scope, not per resource as the estate grows.

### Recommendation for This Scenario
**Unified control framework** in Microsoft Purview Compliance Manager, mapping the org's actual controls once and scoring them against both the ISO 27001 and NIST SP 800-53 templates from that single mapping. **Combined enforcement**: Azure Policy regulatory compliance initiatives handle the technical controls continuously and automatically (encryption, MFA enforcement, network restrictions), while Compliance Manager tracks the procedural controls (policies, training, incident response) that Policy structurally cannot evaluate. Part 2–4 implement the Azure Policy half of this design.

---

## Part 2: Assign a Built-In Regulatory Compliance Initiative

### Step 1: Find the Built-In Initiative Definitions
```bash
az policy set-definition list --query "[?contains(displayName,'NIST SP 800-53')].{name:name, id:id}" -o table
az policy set-definition list --query "[?contains(displayName,'ISO 27001')].{name:name, id:id}" -o table
```
Both are built-in, Microsoft-maintained initiatives — no custom policy set to author, unlike the custom governance initiative in AZ-305 Lab 1. This is deliberate: regulatory frameworks are standardized enough that Microsoft maintains the control mapping directly.

Azure Policy ships built-in initiatives for most frameworks an organization is likely to face — worth knowing what's available before assuming one needs to be built from scratch:

| Framework | Typical Trigger | Built-In Initiative Exists? |
|---|---|---|
| NIST SP 800-53 Rev. 5 | US federal or federal-adjacent contracts | Yes |
| ISO 27001:2013 | International customers, general security certification | Yes |
| PCI DSS v4 | Payment card processing | Yes |
| HIPAA/HITRUST | US healthcare data | Yes |
| SOC 2 Type II | SaaS vendor due diligence from enterprise customers | Yes (via the Microsoft cloud security benchmark mapping) |
| A bespoke internal control standard with no public framework | Company-specific requirements not covered by any regulator | No — this is the one case that still needs a custom initiative, as built in AZ-305 Lab 1 |

### Step 2: Assign the NIST Initiative at Subscription Scope
```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
NIST_ID=$(az policy set-definition list --query "[?contains(displayName,'NIST SP 800-53')].id | [0]" -o tsv)

az policy assignment create \
  --name "sc100-lab4-nist-800-53" \
  --display-name "NIST SP 800-53 Rev. 5 Regulatory Compliance" \
  --policy-set-definition "$NIST_ID" \
  --scope "/subscriptions/$SUBSCRIPTION_ID" \
  --enforcement-mode DoNotEnforce
```
`DoNotEnforce` is deliberate here — this initiative is almost entirely `Audit`-effect policies for reporting compliance state, not `Deny`-effect policies meant to block deployments. Evaluate first; the org decides separately whether any individual control should escalate to enforced/Deny.

### Step 3: Assign the ISO 27001 Initiative Alongside It
```bash
ISO_ID=$(az policy set-definition list --query "[?contains(displayName,'ISO 27001')].id | [0]" -o tsv)

az policy assignment create \
  --name "sc100-lab4-iso-27001" \
  --display-name "ISO 27001:2013 Regulatory Compliance" \
  --policy-set-definition "$ISO_ID" \
  --scope "/subscriptions/$SUBSCRIPTION_ID" \
  --enforcement-mode DoNotEnforce
```
Assigning both at the same scope, evaluated against the same live resource state, is the technical proof of Decision 1's unified-framework model — one set of Azure resources, two frameworks' worth of automated evidence, no duplicated deployment.

**Validation checkpoint**:
```bash
az policy assignment list --query "[?starts_with(name,'sc100-lab4')].{name:displayName, mode:enforcementMode}" -o table
```
Both assignments should be listed with `mode: DoNotEnforce`.

---

## Part 3: Review the Compliance State

### Step 1: Trigger a Compliance Scan
```bash
az policy state trigger-scan --resource-group sc100-lab4-rg 2>/dev/null || \
az policy state trigger-scan --subscription "$SUBSCRIPTION_ID"
```
Policy evaluates on a schedule (roughly every 24 hours) or on resource change — triggering a manual scan gets a fresh read without waiting.

### Step 2: Read the Regulatory Compliance Report
```bash
az policy state list \
  --filter "PolicyAssignmentName eq 'sc100-lab4-nist-800-53'" \
  --query "[].{control:policyDefinitionName, resource:resourceId, compliant:complianceState}" \
  -o table | head -30
```

### Step 3: Pull the Aggregate Compliance Percentage
```bash
az rest --method GET \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.PolicyInsights/policyStates/latest/summarize?api-version=2019-10-01" \
  --body '{"policyAssignments":[{"policyAssignmentId":"/subscriptions/'"$SUBSCRIPTION_ID"'/providers/Microsoft.Authorization/policyAssignments/sc100-lab4-nist-800-53"}]}' \
  --query "value[0].results.resourceDetails" -o table
```
**Expected result**: a compliant/non-compliant resource count per control. This is the automated technical-control evidence half of Decision 2 — a live number, not a checkbox someone filled in from memory.

---

## Part 4: Map a Policy Control to a Compliance Manager Improvement Action

Azure Policy proves the technical state; Microsoft Purview Compliance Manager tracks the full control lifecycle (owner, evidence attachment, procedural steps Policy can't see) and rolls it into an overall compliance score.

| Azure Policy Control (Technical Evidence) | Compliance Manager Improvement Action (Full Control Lifecycle) | Framework(s) Covered |
|---|---|---|
| "Storage accounts should use customer-managed key for encryption" (Audit result from Part 3) | "Implement and manage encryption keys" — tracks key rotation procedure, ownership, and the Policy audit result as attached evidence | NIST 800-53 (SC-12, SC-28), ISO 27001 (A.10.1) |
| "MFA should be enabled for accounts with owner permissions" | "Require multifactor authentication for privileged accounts" — links to Lab 2's Conditional Access policy as procedural evidence | NIST 800-53 (IA-2), ISO 27001 (A.9.4) |
| "Diagnostic logs in Key Vault should be enabled" | "Enable audit/log records for security incidents" — links to Lab 3's Sentinel data connector configuration | NIST 800-53 (AU-2), ISO 27001 (A.12.4) |
| "Guest accounts with owner permissions on Azure resources should be removed" | "Review and remove excessive privileged access" — links to Lab 2's PIM-eligible role design as procedural evidence that standing access was eliminated | NIST 800-53 (AC-2, AC-6), ISO 27001 (A.9.2) |
| "Azure Cosmos DB/Storage should have firewall rules" | "Restrict access to systems processing sensitive data" — links to Lab 6's network access design as procedural evidence | NIST 800-53 (SC-7), ISO 27001 (A.13.1) |

In the Compliance Manager portal, each improvement action is scored (0–100% implemented) and contributes to the overall Compliance Score. Attaching the Part 3 Policy compliance report as evidence on the matching improvement action is what turns "we believe this control is met" into an auditable record — this mapping table is the deliverable that makes the unified control framework from Decision 1 real rather than aspirational.

**Validation checkpoint**: In **Microsoft Purview compliance portal → Compliance Manager → Improvement actions**, confirm the three actions above exist (searchable by the NIST/ISO template names) and that each can accept an attached evidence file or link — this is where the Part 3 CLI output gets attached in a live implementation.

---

## Part 5: Escalate One Reviewed Control from Audit to Enforced

`DoNotEnforce` in Part 2 was deliberate — evaluate before blocking. Once Part 3's compliance report has been reviewed and a specific control's non-compliant resources have either been remediated or explicitly exempted, that control (not the whole initiative) can move to active enforcement. This mirrors the report-only-to-enforced pattern from Labs 1 and 2, applied to governance instead of access control.

### Step 1: Identify a Low-Risk, High-Confidence Control to Escalate First
```bash
az policy state list \
  --filter "PolicyAssignmentName eq 'sc100-lab4-nist-800-53' and ComplianceState eq 'Compliant'" \
  --query "[].policyDefinitionName" -o tsv | sort -u | head -5
```
A control already showing 100% compliant across the environment is the safest one to escalate first — enforcement can't break anything that's already conforming, and it locks in the current good state against future drift.

### Step 2: Create a Scoped Assignment for Just That Control in Enforced Mode
Rather than flipping the entire initiative to `Default` (which would enforce every Deny-effect control at once, including ones that haven't been reviewed), assign a narrower initiative or a single policy definition in enforced mode for the specific reviewed control.
```bash
az policy assignment create \
  --name "sc100-lab4-mfa-owner-enforced" \
  --display-name "Enforce MFA for Owner-Permission Accounts (Reviewed, Compliant)" \
  --policy "<mfa-owner-accounts-policy-definition-id>" \
  --scope "/subscriptions/$SUBSCRIPTION_ID" \
  --enforcement-mode Default
```

**Validation checkpoint**:
```bash
az policy assignment list --query "[?name=='sc100-lab4-mfa-owner-enforced'].enforcementMode" -o tsv
```
Confirm `Default` — this control is now actively enforced while the remaining, unreviewed controls in the two broad initiatives stay in `DoNotEnforce`, giving the organization a controlled, control-by-control path from audit to enforcement instead of an all-or-nothing switch.

---

## Cleanup
```bash
az policy assignment delete --name "sc100-lab4-nist-800-53" --scope "/subscriptions/$SUBSCRIPTION_ID"
az policy assignment delete --name "sc100-lab4-iso-27001" --scope "/subscriptions/$SUBSCRIPTION_ID"
az policy assignment delete --name "sc100-lab4-mfa-owner-enforced" --scope "/subscriptions/$SUBSCRIPTION_ID"
```
Confirm with `az policy assignment list --query "[?starts_with(name,'sc100-lab4')]"` — expect an empty array. No resource group or billable resource was created in this lab, so this is the only cleanup step.

---

## What You Practiced

| Concept | Why It Matters |
|---|---|
| Unified control framework mapping | SC-100 tests recognizing when multiple regulatory requirements should be mapped once, not re-assessed per framework |
| Automated (Policy) vs. procedural (Compliance Manager) evidence | The exam distinguishes what Azure Policy can and cannot technically verify — a recurring scenario pattern |
| Built-in regulatory compliance initiatives | Knowing these exist (vs. authoring a custom initiative, as in AZ-305 Lab 1) is itself a design decision — don't rebuild what Microsoft already maintains and updates |
| DoNotEnforce for audit-first rollout | Same report-before-enforce discipline as Labs 1–2, applied to compliance instead of access control |
| Control-to-evidence mapping | The artifact an auditor actually wants — traceability from technical state to the control it satisfies |
| Control-by-control enforcement escalation | Avoids the false choice between "audit everything forever" and "enforce everything at once" — the actual design skill is a graduated path between them |
| Management-group-scoped assignment | Prevents the compliance program's audit overhead from scaling with subscription count, the same inheritance principle Lab 1's roadmap and AZ-305 Lab 1 both rely on |

---

## Common Mistakes to Avoid
- **Running a separate assessment project per regulation**: the audit burden compounds with every new framework instead of scaling sub-linearly
- **Treating Azure Policy compliance percentage as the whole compliance program**: it only ever covers what's technically evaluable — procedural and administrative controls need Compliance Manager or equivalent tracking
- **Assigning regulatory initiatives in `Default` enforcement mode on day one**: most controls in these initiatives are Audit-effect already, but any Deny-effect control in an unreviewed initiative can block legitimate deployments in an environment with existing non-compliant resources
- **Assuming compliance percentage reflects real-time state without triggering or waiting for a scan**: Policy evaluation isn't instantaneous — a change made minutes ago may not show up in the compliance report yet
- **Losing the mapping between technical evidence and the control it satisfies**: without Part 4's explicit table, the Policy compliance report and the Compliance Manager score drift apart and stop reinforcing each other
- **Escalating an entire initiative to enforced instead of one reviewed control at a time**: flipping the whole initiative from `DoNotEnforce` to `Default` re-introduces the same blast-radius risk Part 5 was designed to avoid — every Deny-effect control in the initiative activates simultaneously, including ones nobody has reviewed yet
- **Assigning the same initiative redundantly per subscription in a multi-subscription estate**: re-creates the exact governance drift risk AZ-305 Lab 1's management-group hierarchy exists to prevent — assign once at the correct inherited scope

---

## Next Steps
- Continue to [Lab 5: Hybrid & Multicloud Infrastructure Security Design](lab-5-hybrid-multicloud-infrastructure-security-design.md) to extend this governance model to resources outside a single Azure subscription
- For the underlying management group and Azure Policy initiative mechanics this lab builds on, see [AZ-305 Lab 1: Identity, Governance & Monitoring Design](../AZ-305/lab-1-identity-governance-monitoring-design.md)
- For the operational side of running a compliance program day-to-day (access reviews, entitlement management), see [SC-300](../SC-300/README.md); for information protection and DLP controls that feed the data-related improvement actions, see [SC-500](../SC-500/README.md)
