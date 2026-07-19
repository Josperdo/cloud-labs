# Lab 8: Capstone — End-to-End Landing Zone Solution Design

Check box if done: [ ]

## Overview
Every lab in this track answered one design question in isolation: how to govern (Lab 1), how to store data (Lab 2), how to survive an outage (Lab 3), how to host an app (Lab 4), how to build the network (Lab 5), how to go global (Lab 6), how to migrate a legacy estate (Lab 7). A real design review is never one of those questions — it's all seven, asked about the same company, in the same room, at the same time. This capstone doesn't introduce a new domain. It takes a single realistic scenario, runs it through every decision framework built in Labs 1–7, and deploys a scaled-down governed landing zone that demonstrates the pattern without requiring a full enterprise environment to stand it up.

**Estimated time**: 90–110 minutes
**Cost**: ~$1–$4 (a couple of free management groups and a policy initiative cost nothing; one spoke VNet with NSGs only — no Azure Firewall this time, a deliberate design call explained in Part 2 — plus a small storage account and budget alert; nothing here is a large hourly meter)

---

## Scenario
Contoso Retail Group has grown by acquisition — four business units (Retail Storefront, Warehouse Ops, Wholesale Partners, and a recently-acquired startup, Fintrack) each provisioned their own Azure subscription independently over the last three years. There's no shared management group hierarchy, no consistent tagging, three different storage redundancy tiers chosen for no documented reason, one business unit with no DR plan at all, inconsistent hosting choices (one team hand-rolled a VM-based deployment pipeline that another team already solved with App Service slots), no global routing strategy despite two business units now serving international customers, and Fintrack still runs its core ledger system on an on-prem server the acquisition agreement requires migrating within 12 months. The CFO wants cost visibility across all four business units in one place. The CISO wants one governance model, not four. You're the architect writing the design response that consolidates all of this into a single governed landing zone — and proving the pattern works with a scaled-down deployment before asking for budget to do it at full scale.

---

## Objectives
- Decide between continuing four independent subscriptions and consolidating into a CAF-aligned, multi-subscription landing zone with subscription vending
- Apply Labs 1–7's decision frameworks to Contoso's four business units and synthesize the results into one design response
- Deploy a scaled-down landing zone: two management groups, one policy initiative, and one spoke VNet
- Layer a cost and governance overlay (tagging-driven cost allocation, budget alerts) on top of the deployed structure
- Produce a synthesis table mapping every prior lab's decision into this one consolidated architecture

---

## Part 1: Design Decision — Landing Zone Topology and Guardrail Enforcement Model

### Decision 1: Continue Independent Subscriptions vs. CAF-Aligned Multi-Subscription Landing Zone

| Factor | Four Independent Subscriptions (current state) | CAF-Aligned Landing Zone with Subscription Vending |
|---|---|---|
| **Governance consistency** | Each business unit's policy, RBAC, and tagging is whatever that team happened to set up — already proven inconsistent (three redundancy tiers, no shared tagging) | One policy initiative assigned at the landing zone management group applies to all four business units identically, and to any future acquisition's subscription automatically |
| **Cost visibility** | Requires manually aggregating cost data across four disconnected subscriptions with no shared tagging to group by | Management-group-scoped cost views plus a mandatory cost-center tag (Lab 1's pattern, reused here) give the CFO one place to look |
| **Blast radius of a new business unit onboarding badly** | Each acquisition repeats the same ungoverned setup Fintrack is in today | A new subscription enters through subscription vending, inherits the guardrails immediately — no repeat of today's problem |
| **Migration effort to get there** | None — it's the status quo | Real, but bounded: existing subscriptions move under the new management group hierarchy (a metadata operation, not a resource migration) and pick up policy going forward; nothing about the workloads themselves has to move |
| **Fit for this scenario** | Fails both the CISO's "one governance model" ask and the CFO's cost-visibility ask directly | Satisfies both, and gives Fintrack's eventual migration (Lab 7) a governed landing zone to migrate *into* instead of a fifth ungoverned subscription |

### Decision 2: Policy Enforcement Point — Management Group Initiative vs. Per-Subscription Configuration vs. Third-Party CSPM

| Factor | Management-Group-Scoped Policy Initiative | Per-Subscription Manual Configuration | Third-Party CSPM Tooling |
|---|---|---|---|
| **Consistency** | One assignment, inherited by every subscription underneath, present and future | Exactly the problem this scenario already has — four teams, four interpretations | Consistent, but adds a second control plane alongside Azure Policy that has to stay in sync |
| **Cost** | Free (Azure Policy itself has no charge) | Free, but the *inconsistency* has a real cost — untagged resources the CFO can't allocate, redundancy tiers nobody can justify | Licensed, ongoing cost |
| **New business unit onboarding** | Automatic via inheritance the moment a subscription lands in the right management group | Manual, repeated, and exactly how today's inconsistency happened | Requires the new subscription to be onboarded into the third-party tool separately |
| **Best fit** | This scenario — native tooling, zero incremental cost, and directly solves the stated inconsistency problem | Nothing at this scale — it's the cause of the current mess, not a fix for it | Organizations needing multicloud policy consistency Azure Policy alone can't reach (Contoso is Azure-only here) |

### Recommendation for This Scenario
A **CAF-aligned landing zone**: an intermediate root management group, a **Platform** branch for shared services, and a **Landing Zones** branch holding each business unit as its own subscription (Retail Storefront, Warehouse Ops, Wholesale Partners today; Fintrack once its migration — Lab 7's 5 R's framework — lands it in Azure instead of on-prem). Guardrails enforced through **one Azure Policy initiative assigned at the Landing Zones management group**, reusing Lab 1's pattern at four-business-unit scale instead of one. Part 2 builds a scaled-down version of exactly this shape.

---

## Part 2: Build the Scaled-Down Management Group Hierarchy and Guardrail Initiative

A full deployment would model all four business units as separate subscriptions under Platform/Landing Zones branches matching [General Lab 7](../General/lab-7-landing-zone-governance.md)'s full CAF shape. This capstone builds the minimum structure that proves the pattern: one Landing Zones group, one business unit represented underneath it, and a guardrail initiative that would inherit identically to a second, third, or fourth business unit added later.

### Step 1: Build the Management Group Hierarchy
```bash
az account management-group create --name "contosoretail-root" --display-name "Contoso Retail Group"

az account management-group create --name "contosoretail-landingzones" \
  --display-name "Landing Zones" --parent "contosoretail-root"

# One business unit modeled explicitly — Retail Storefront. Warehouse Ops, Wholesale
# Partners, and (post-migration) Fintrack would each get their own sibling group here.
az account management-group create --name "contosoretail-retailstorefront" \
  --display-name "Retail Storefront" --parent "contosoretail-landingzones"

# Move your subscription into the business-unit group to prove inheritance end to end
az account management-group subscription add \
  --name "contosoretail-retailstorefront" --subscription "<your-subscription-id>"
```

**Validation checkpoint**:
```bash
az account management-group show --name "contosoretail-root" --expand -o jsonc
```
Confirm the nesting: `contosoretail-root` → `contosoretail-landingzones` → `contosoretail-retailstorefront`, with your subscription attached at the bottom.

### Step 2: Assign a Consolidated Guardrail Initiative
This reuses Lab 1's initiative pattern — region restriction plus mandatory tagging — but the tag set now includes a `businessUnit` tag, which is the field the cost overlay in Part 3 groups by.
```bash
cat > landingzone-guardrails.json <<'EOF'
{
  "properties": {
    "displayName": "Contoso Retail Group Landing Zone Guardrails",
    "policyDefinitions": [
      { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c",
        "policyDefinitionReferenceId": "allowedLocations",
        "parameters": { "listOfAllowedLocations": { "value": "[parameters('allowedRegions')]" } } },
      { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/871b6d14-10aa-478d-b590-94f262ecfa99",
        "policyDefinitionReferenceId": "requireBusinessUnitTag",
        "parameters": { "tagName": { "value": "businessUnit" } } },
      { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/871b6d14-10aa-478d-b590-94f262ecfa99",
        "policyDefinitionReferenceId": "requireCostCenterTag",
        "parameters": { "tagName": { "value": "costCenter" } } }
    ],
    "parameters": {
      "allowedRegions": { "type": "Array", "metadata": { "displayName": "Allowed Regions" } }
    }
  }
}
EOF

az policy set-definition create --name "contosoretail-landingzone-guardrails" \
  --display-name "Contoso Retail Group Landing Zone Guardrails" \
  --management-group "contosoretail-landingzones" --definitions landingzone-guardrails.json

az policy assignment create --name "contosoretail-guardrails-assignment" \
  --display-name "Contoso Retail Group Landing Zone Guardrails" \
  --policy-set-definition "contosoretail-landingzone-guardrails" \
  --management-group "contosoretail-landingzones" \
  --params '{ "allowedRegions": { "value": ["eastus", "eastus2"] } }' \
  --enforcement-mode Default
```
Assigned at `contosoretail-landingzones`, not per-business-unit — the entire point being that Warehouse Ops or Wholesale Partners onboarding tomorrow inherits this identically, with zero repeated setup.

**Validation checkpoint**:
```bash
az group create --name test-no-tags-rg --location eastus
```
Expect `RequestDisallowedByPolicy` — no `businessUnit`/`costCenter` tags. Then:
```bash
az group create --name rg-az305-lab8-retailstorefront --location eastus \
  --tags businessUnit=RetailStorefront costCenter=RS-2000
```
Should succeed — this resource group is where Part 3's spoke VNet gets built.

---

## Part 3: One Spoke VNet — the Scaled-Down Network Pattern

A full deployment would put Azure Firewall in a hub, exactly as Lab 5 built, shared across every business unit's spoke. For a capstone demonstration, that hourly-billed meter isn't justified just to prove the pattern works — this is the **NSG+UDR-only cost-conscious alternative Lab 5's Decision 2 explicitly named** as the trade-off when Azure Firewall's cost isn't warranted. A real Contoso Retail Group deployment would size this decision the same way Lab 5 did, against real traffic and compliance requirements; this capstone makes the deliberate call to skip it here and say so.

### Step 3: Deploy the Spoke VNet with NSG-Based Segmentation
```bash
RG="rg-az305-lab8-retailstorefront"
LOCATION="eastus"

az network vnet create --resource-group $RG --name spoke-retailstorefront-vnet \
  --address-prefix 10.10.0.0/16 --subnet-name workload-subnet --subnet-prefix 10.10.1.0/24

az network nsg create --resource-group $RG --name nsg-retailstorefront-workload

az network nsg rule create --resource-group $RG --nsg-name nsg-retailstorefront-workload \
  --name DenyAllInboundInternet --priority 4096 --direction Inbound --access Deny \
  --protocol "*" --source-address-prefixes Internet --destination-port-ranges "*"

az network nsg rule create --resource-group $RG --nsg-name nsg-retailstorefront-workload \
  --name AllowVnetInbound --priority 100 --direction Inbound --access Allow \
  --protocol "*" --source-address-prefixes VirtualNetwork --destination-port-ranges "*"

az network vnet subnet update --resource-group $RG --vnet-name spoke-retailstorefront-vnet \
  --name workload-subnet --network-security-group nsg-retailstorefront-workload
```
This mirrors Lab 5's hub-spoke *intent* — a workload subnet that isn't flatly open to the internet — without the hub, the firewall, or the peering that a real multi-business-unit hub-spoke would add once a second business unit's spoke needs shared egress inspection.

**Validation checkpoint**:
```bash
az network nsg rule list --resource-group $RG --nsg-name nsg-retailstorefront-workload -o table
```
Confirm both rules exist, with `DenyAllInboundInternet` and `AllowVnetInbound` both present.

---

## Part 4: Cost and Governance Overlay

### Step 4: Cost Visibility Grouped by Business Unit
The `businessUnit` tag enforced in Part 2 is what makes this possible — without it, the CFO's "one place to look" ask has nothing to group by.
```bash
az storage account create --name stlab8retail<your-unique-suffix> --resource-group $RG \
  --location $LOCATION --sku Standard_LRS --tags businessUnit=RetailStorefront costCenter=RS-2000

az consumption budget create --budget-name budget-retailstorefront-monthly \
  --amount 500 --time-grain Monthly \
  --start-date $(date +%Y-%m-01) --end-date 2027-01-01 \
  --resource-group $RG \
  --notifications '{"Actual_GreaterThan_80_Percent":{"enabled":true,"operator":"GreaterThan","threshold":80,"contactEmails":["<your-email>"],"thresholds":[80]}}'
```
This is deliberately scoped to one resource group for the lab — a production rollout would set a budget per business-unit management group or subscription instead, and pair it with a Cost Management view filtered by the `businessUnit` tag across all four subscriptions, giving the CFO the single cross-business-unit view the scenario asked for.

**Validation checkpoint**:
```bash
az consumption budget show --budget-name budget-retailstorefront-monthly --resource-group $RG -o table
```
Confirm the budget exists with the 80% alert threshold configured.

---

## Part 5: Design Synthesis

This is the actual deliverable the scenario asked for — not any single control, but showing how seven independent lab decisions reinforce one governed landing zone for Contoso Retail Group.

| AZ-305 Domain | Design Decision | Built In | How It Applies to Contoso |
|---|---|---|---|
| **Identity, Governance & Monitoring** | Management group hierarchy + centralized policy initiative | [Lab 1](lab-1-identity-governance-monitoring-design.md) | Replaces four teams' inconsistent setups with one inherited guardrail set, extended here with a `businessUnit` tag for cost allocation |
| **Data Storage** | Service selection by access pattern, redundancy tier by stated RPO | [Lab 2](lab-2-data-storage-design.md) | Resolves the "three different redundancy tiers chosen for no documented reason" problem — each business unit's storage gets a tier justified by an actual RPO, not habit |
| **Business Continuity** | Backup + ASR for the RTO/RPO gap neither covers alone | [Lab 3](lab-3-business-continuity-design.md) | Directly closes "one business unit with no DR plan at all" — the framework applies regardless of which business unit has the gap |
| **Compute & Application Infrastructure** | Hosting-model decision matrix (App Service vs. AKS vs. Container Apps vs. VMSS vs. Functions) | [Lab 4](lab-4-compute-app-infrastructure-design.md) | Replaces the hand-rolled VM pipeline one team built with the same App Service + deployment slots pattern another team already solved — one hosting standard instead of two |
| **Network Infrastructure** | Hub-spoke topology, perimeter security tier selection | [Lab 5](lab-5-network-infrastructure-design.md) | The full-scale target once a second business unit's spoke needs shared egress inspection — Part 3 above deploys its cost-conscious NSG-only alternative deliberately, not by accident |
| **Global Distribution** | Front Door vs. Traffic Manager vs. App Gateway, Cosmos DB write topology | [Lab 6](lab-6-multi-region-global-distribution-design.md) | Answers "no global routing strategy despite two business units now serving international customers" with a specific, justified service choice instead of an unaddressed gap |
| **Migration & Modernization** | The 5 R's, online vs. offline database migration, strangler-fig | [Lab 7](lab-7-migration-modernization-design.md) | Directly the Fintrack requirement — a 12-month deadline to move off an on-prem ledger system, assessed workload-by-workload rather than defaulted to a blanket rehost |
| **Landing Zone (this lab)** | Subscription vending, policy-driven guardrails, cost/governance overlay | This lab | The container all seven other decisions land inside — without it, each business unit re-derives its own version of Labs 1–7 independently, which is exactly the inconsistency Contoso started with |

**The thread connecting all eight**: Contoso's four business units didn't fail because any one of them made a bad individual decision — Retail Storefront's redundancy tier, Warehouse Ops' DR gap, and the hand-rolled VM pipeline are each defensible in isolation. They failed because there was no shared structure for those decisions to land inside, so every business unit re-solved the same problems independently and inconsistently. A landing zone isn't an additional control on top of Labs 1–7 — it's the governance and subscription structure that makes Labs 1–7's decisions apply consistently across every business unit, present and future, instead of once per team that happens to ask the right question.

---

## Cleanup

```bash
# 1. Assignment before initiative
az policy assignment delete --name "contosoretail-guardrails-assignment" \
  --management-group "contosoretail-landingzones"
az policy set-definition delete --name "contosoretail-landingzone-guardrails" \
  --management-group "contosoretail-landingzones"

# 2. Budget
az consumption budget delete --budget-name budget-retailstorefront-monthly --resource-group $RG

# 3. Resource groups
az group delete --name test-no-tags-rg --yes --no-wait
az group delete --name $RG --yes --no-wait

# 4. Move subscription back to Tenant Root Group before groups can be deleted
az account management-group subscription remove \
  --name "contosoretail-retailstorefront" --subscription "<your-subscription-id>"

# 5. Delete management groups leaf-to-root
az account management-group delete --name "contosoretail-retailstorefront"
az account management-group delete --name "contosoretail-landingzones"
az account management-group delete --name "contosoretail-root"
```
Confirm with `az account management-group list -o table` (only the Tenant Root Group should remain) and `az group list -o table` (no `rg-az305-lab8-*` or `test-no-tags-rg` groups should remain).

---

## What You Practiced

| Task | Why It Matters on the Job |
|---|---|
| Applying seven independent decision frameworks to one shared scenario | This is the actual AZ-305 exam skill and the actual job — no design review asks about identity, storage, or network in isolation, it asks about all of them for one company at once |
| Choosing a landing zone topology over continued subscription sprawl | Recognizing when the fix isn't another point solution but a shared structure those solutions can consistently land inside |
| Deploying a scaled-down proof of a pattern instead of the full enterprise build | A design review rarely gets budget for the full build up front — showing the pattern works at small scale is how that budget gets approved |
| Making (and stating) a deliberate cost-vs-completeness trade-off | Skipping Azure Firewall here wasn't an oversight — naming the trade-off explicitly is what separates a defensible design decision from a shortcut |
| Synthesizing multiple designs into one narrative | The Part 5 table is the deliverable a stakeholder actually reads — not eight disconnected labs, one coherent architecture |

---

## Common Mistakes to Avoid
- **Treating each AZ-305 domain as a standalone answer instead of a system**: real scenario questions (and real design reviews) frequently require identifying how a gap in one domain undermines another — Fintrack's migration (Lab 7) has nowhere governed to land without the landing zone (this lab) existing first
- **Building the full enterprise landing zone before proving the pattern**: a scaled-down deployment that demonstrates the shape is what gets a design approved for budget — building all four business units' full infrastructure before anyone signs off is backwards
- **Skipping the guardrail tag design and bolting on cost allocation later**: the `businessUnit` tag has to be enforced *before* resources exist for cost visibility to be complete — retrofitting tags onto an existing untagged estate is a much larger cleanup project
- **Assuming a landing zone is a one-time deployment**: like [General Lab 7](../General/lab-7-landing-zone-governance.md) emphasizes, it's the ongoing structure every future business unit (or acquisition) lands into, not a project that finishes when this lab's resource group is deleted
- **Omitting the explicit cost/completeness trade-off from the design writeup**: skipping Azure Firewall for the capstone deployment is defensible; skipping it silently, without naming the trade-off and the conditions that would flip the decision, is not

---

## Next Steps
This closes out the AZ-305 track — [Lab 1](lab-1-identity-governance-monitoring-design.md) through [Lab 7](lab-7-migration-modernization-design.md) each built one design domain in isolation; this capstone is where they converge into a single governed architecture, the same way a real design review or interview question would require. For the full-depth, multi-business-unit CAF management group shape and policy-as-code deployment pattern this lab's Part 2 deliberately scaled down, see [Azure General Lab 7: Landing Zone & Governance at Scale](../General/lab-7-landing-zone-governance.md). For the security-architecture counterpart to this entire track — the same design-decision-first discipline applied to Zero Trust instead of general infrastructure — see [SC-100: Microsoft Cybersecurity Architect Expert](../SC-100/README.md), whose own Lab 8 capstone follows this identical synthesis pattern.
