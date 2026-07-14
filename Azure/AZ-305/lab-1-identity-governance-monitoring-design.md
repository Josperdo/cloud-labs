# Lab 1: Identity, Governance & Monitoring Design

Check box if done: [ ]

## Overview
AZ-305 tests design decisions, not button-clicking. Anyone can create a management group or assign a policy in the portal — the exam (and the job) cares about *why* you chose a 3-level hierarchy over a flat list, and whether you centralized monitoring or split it per subscription, and what that decision costs in RBAC complexity, compliance boundaries, and query performance later. This lab forces that decision on paper before any resource is deployed, then implements the chosen design.

**Estimated time**: 60-75 minutes
**Cost**: ~$0 (management groups, Policy, and Log Analytics workspace creation are free; only ingested log volume could bill, and this lab stays under free-tier daily ingestion)

---

## Scenario
You're the architect for a company that's been running everything in a single flat Azure subscription since day one — dev, test, and production mixed together, no consistent tagging, no region restrictions, logs scattered across whatever diagnostic settings someone remembered to configure. Leadership just approved splitting into separate subscriptions for production, non-production, and shared services (networking, identity), with two more subscriptions planned for next quarter. Before anyone provisions a fourth subscription ad hoc, you need a governance and monitoring design that scales without a redesign every time the company adds one.

---

## Objectives
- Design a management group hierarchy that supports policy inheritance and RBAC delegation without becoming unmanageable
- Decide between a centralized and per-subscription Log Analytics workspace topology and justify the tradeoff
- Build an Azure Policy initiative bundling region restriction and mandatory tagging, assigned at a management group scope
- Deploy centralized logging with diagnostic settings routing multiple resources into one workspace
- Validate policy enforcement and confirm log data actually flows before calling the design "done"

---

## Part 1: Design Decision — Management Group Hierarchy & Monitoring Topology

This is the part that actually maps to the exam. Don't touch the CLI until the decision is made and justified.

### Decision 1: Flat Subscription List vs. Management Group Hierarchy

| Factor | Flat (all subscriptions under Tenant Root Group) | 2-3 Level Hierarchy (Root → Landing Zones → Prod/Non-Prod) |
|---|---|---|
| Policy inheritance | Assigned per-subscription; no shared enforcement point, drift guaranteed as subscriptions are added | Assign once at the right level; new subscriptions inherit automatically when placed in the group |
| RBAC delegation | No way to delegate "manage all non-prod" without touching every subscription | One role assignment at the group level applies to every subscription underneath, present and future |
| Blast radius of a bad assignment | Contained to one subscription, but a fix has to be repeated everywhere | Higher leverage at the top of the hierarchy — more risk if done carelessly, more efficient if done right |
| Scales with new subscriptions | No — every addition repeats every policy/RBAC assignment manually | Yes — drop the subscription into the correct group and inheritance handles the rest |
| Overhead at this company's size (3-5 subs) | Low now, grows linearly and painfully | Slightly higher upfront design cost, flat afterward |

### Decision 2: Centralized Log Analytics Workspace vs. Workspace-per-Subscription

| Factor | Centralized (single workspace under Shared Services) | Workspace-per-Subscription |
|---|---|---|
| Data sovereignty / compliance | Fine absent a regulatory requirement to physically isolate logs by environment; can still be region-pinned | Required if compliance mandates isolating production audit logs from non-prod |
| RBAC on query access | Needs table-level RBAC (custom roles scoped to specific tables) to stop cross-environment queries | Resource-context RBAC is automatic — Reader on a subscription's resources only surfaces that subscription's data |
| Cross-workspace queries | None needed — one workspace, one KQL query | Cross-environment investigation needs federated `workspace()` queries — slower, easier to get wrong |
| Cost | No per-workspace minimum overhead; cheaper at low-to-moderate volume since ingestion pools together | Billed per workspace regardless; splitting low volume across several workspaces loses any pooled commitment-tier discount |
| Operational simplicity | One place for alerts, dashboards, and a future SIEM connection | Alerting and dashboards duplicated or federated per environment |

### Recommendation for This Scenario
**Management group hierarchy: 3 levels** — `Tenant Root Group` → `Contoso` (intermediate root) → `Landing Zones` (split into `Production` and `Non-Production`), with `Shared Services` as a sibling branch. This keeps policy assignment points few and predictable: region + tagging policy at the `Contoso` level applies to everything, with room for stricter policies (e.g., deny public IPs) scoped only to `Production` later.

**Monitoring: single centralized Log Analytics workspace** under Shared Services. At this scale, with no stated data-isolation requirement, the operational simplicity of one query and alerting surface outweighs the RBAC design cost of table-level access control. If a compliance requirement to isolate production logs appears later, splitting workspaces is a config change — diagnostic settings get repointed, not a re-architecture.

---

## Part 2: Build the Management Group Hierarchy

Replace `contoso` with your own placeholder — never a real company or tenant name.

```bash
az account show --query "{subscriptionId:id, tenantId:tenantId}" -o table

# Level 1: intermediate root under Tenant Root Group
az account management-group create --name "contoso-root" --display-name "Contoso"

# Level 2: Landing Zones, parented under contoso-root
az account management-group create --name "contoso-landingzones" \
  --display-name "Landing Zones" --parent "contoso-root"

# Level 3: Production and Non-Production, parented under Landing Zones
az account management-group create --name "contoso-production" \
  --display-name "Production" --parent "contoso-landingzones"
az account management-group create --name "contoso-nonproduction" \
  --display-name "Non-Production" --parent "contoso-landingzones"

# Shared Services sits alongside Landing Zones — not a workload environment
az account management-group create --name "contoso-sharedservices" \
  --display-name "Shared Services" --parent "contoso-root"

# Move your subscription into the appropriate group
az account management-group subscription add \
  --name "contoso-nonproduction" --subscription "<your-subscription-id>"
```

**Validation checkpoint**:
```bash
az account management-group list -o table
az account management-group show --name "contoso-nonproduction" --expand -o jsonc
```
Confirm five management groups exist and your subscription is nested under `contoso-nonproduction` → `contoso-landingzones` → `contoso-root`. In the portal, the **Management groups** tree view should match this nesting.

---

## Part 3: Deploy a Policy Initiative (Region Restriction + Mandatory Tags)

An initiative (policy set) bundles multiple definitions into one assignment — one compliance evaluation and one assignment to manage, instead of tracking several that could drift out of sync.

### Step 1: Confirm the Built-In Definitions
```bash
az policy definition list --query "[?displayName=='Allowed locations'].{name:name, id:id}" -o table
az policy definition list --query "[?displayName=='Require a tag on resources'].{name:name, id:id}" -o table
```
The "require a tag" definition enforces one tag per reference, so it's referenced twice in the initiative below — once per required tag.

### Step 2: Build the Initiative Definition
Save as `governance-initiative.json`:

```json
{
  "properties": {
    "displayName": "Contoso Baseline Governance",
    "policyDefinitions": [
      { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c",
        "policyDefinitionReferenceId": "allowedLocations",
        "parameters": { "listOfAllowedLocations": { "value": "[parameters('allowedRegions')]" } } },
      { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/871b6d14-10aa-478d-b590-94f262ecfa99",
        "policyDefinitionReferenceId": "requireEnvironmentTag",
        "parameters": { "tagName": { "value": "environment" } } },
      { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/871b6d14-10aa-478d-b590-94f262ecfa99",
        "policyDefinitionReferenceId": "requireCostCenterTag",
        "parameters": { "tagName": { "value": "costCenter" } } }
    ],
    "parameters": {
      "allowedRegions": { "type": "Array", "metadata": { "displayName": "Allowed Regions" } }
    }
  }
}
```

```bash
az policy set-definition create --name "contoso-baseline-governance" \
  --display-name "Contoso Baseline Governance" \
  --management-group "contoso-root" --definitions governance-initiative.json
```

### Step 3: Assign the Initiative at the Management Group Level
Assigning at `contoso-root` inherits down to every subscription in the hierarchy, including subscriptions added next quarter, with zero extra work.

```bash
az policy assignment create --name "contoso-baseline-governance-assignment" \
  --display-name "Contoso Baseline Governance" \
  --policy-set-definition "contoso-baseline-governance" \
  --management-group "contoso-root" \
  --params '{ "allowedRegions": { "value": ["eastus", "eastus2"] } }' \
  --enforcement-mode Default
```

`--enforcement-mode Default` means Deny-effect policies actively block non-compliant requests — the built-in tag policy uses `Deny` by default, enforced immediately, not audit-only. If a company already has non-compliant resources, evaluate in `DoNotEnforce` mode first and review the compliance report before flipping to `Default`.

### Validation Checkpoint: Confirm Non-Compliant Deployments Are Denied
```bash
# Wrong region, no tags — expect RequestDisallowedByPolicy
az group create --name test-denied-rg --location westus2
```
Then confirm a compliant deployment succeeds:
```bash
az group create --name test-allowed-rg --location eastus \
  --tags environment=nonprod costCenter=IT-1000
az policy state list --resource-group test-allowed-rg -o table
```

---

## Part 4: Centralized Log Analytics Workspace + Diagnostic Settings

### Step 1: Deploy the Workspace
```bash
az group create --name rg-contoso-monitoring --location eastus

az monitor log-analytics workspace create \
  --resource-group rg-contoso-monitoring --workspace-name law-contoso-central \
  --location eastus --sku PerGB2018 --retention-time 30
```
`PerGB2018` has no per-workspace commitment minimum — consistent with the centralization decision in Part 1, since cost scales with ingested volume, not workspace count.

### Step 2: Route a Resource's Diagnostics into the Workspace
```bash
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-contoso-monitoring --workspace-name law-contoso-central --query id -o tsv)

az storage account create --name stcontosologtest01 \
  --resource-group rg-contoso-monitoring --location eastus --sku Standard_LRS

STORAGE_ID=$(az storage account show --name stcontosologtest01 \
  --resource-group rg-contoso-monitoring --query id -o tsv)

az monitor diagnostic-settings create --name "route-to-central-workspace" \
  --resource "$STORAGE_ID" --workspace "$WORKSPACE_ID" \
  --logs '[{"category":"StorageRead","enabled":true},{"category":"StorageWrite","enabled":true}]' \
  --metrics '[{"category":"Transaction","enabled":true}]'
```

### Step 3: Also Route the Management Group's Activity Log
Activity log data at the management group scope captures control-plane operations (like the policy denial from Part 3) across every subscription underneath it — this is what makes the centralized workspace a single audit surface.

```bash
az monitor diagnostic-settings management-group create \
  --name "route-activity-log-to-central-workspace" \
  --management-group "contoso-root" --workspace "$WORKSPACE_ID" \
  --logs '[{"category":"Administrative","enabled":true},{"category":"Policy","enabled":true}]'
```

**Validation checkpoint**: data takes a few minutes to appear. Generate activity first (the `test-denied-rg` denial from Part 3 already produced one), then query:
```bash
az monitor log-analytics query --workspace "$WORKSPACE_ID" \
  --analytics-query "AzureActivity | where OperationNameValue == 'MICROSOFT.RESOURCES/SUBSCRIPTIONS/RESOURCEGROUPS/WRITE' | project TimeGenerated, OperationNameValue, ActivityStatusValue | order by TimeGenerated desc | take 10" \
  -o table
```
If rows return, the pipeline from resource → diagnostic setting → workspace is confirmed end to end.

---

## Cleanup
Order matters — policy assignments before the initiative, subscriptions moved out before groups are deleted, groups deleted leaf-to-root.

```bash
# 1. Assignment before initiative
az policy assignment delete --name "contoso-baseline-governance-assignment" \
  --management-group "contoso-root"
az policy set-definition delete --name "contoso-baseline-governance" \
  --management-group "contoso-root"

# 2. Management-group-scoped diagnostic setting
az monitor diagnostic-settings management-group delete \
  --name "route-activity-log-to-central-workspace" --management-group "contoso-root"

# 3. Resource groups (removes storage account and workspace with them)
az group delete --name test-denied-rg --yes --no-wait
az group delete --name test-allowed-rg --yes --no-wait
az group delete --name rg-contoso-monitoring --yes --no-wait

# 4. Move subscription back to Tenant Root Group before groups can be deleted
az account management-group subscription remove \
  --name "contoso-nonproduction" --subscription "<your-subscription-id>"

# 5. Delete management groups leaf-to-root — a group with children or subscriptions
#    still attached cannot be deleted
az account management-group delete --name "contoso-production"
az account management-group delete --name "contoso-nonproduction"
az account management-group delete --name "contoso-landingzones"
az account management-group delete --name "contoso-sharedservices"
az account management-group delete --name "contoso-root"
```

Confirm with `az account management-group list -o table` (only Tenant Root Group should remain) and `az group list -o table` (none of the lab's resource groups should remain).

---

## Key Concepts

| Term | Definition |
|---|---|
| **Management group** | A container above the subscription level used to apply policy and RBAC to multiple subscriptions at once via inheritance |
| **Policy definition** | A single rule (e.g., "allowed locations") describing a condition and an effect |
| **Policy initiative (policy set)** | A bundle of multiple policy definitions assigned and evaluated together as one unit |
| **Policy effect: Deny** | Blocks the non-compliant request at submission time — the resource is never created |
| **Policy effect: Audit** | Allows the request but flags the resource non-compliant in the report — no blocking |
| **Policy effect: Append** | Adds fields (e.g., a tag) to the request before creation, rather than blocking or flagging |
| **Log Analytics workspace** | The data store and KQL query engine behind Azure Monitor logs; the unit of RBAC and retention configuration |
| **Diagnostic settings** | Per-resource (or management-group-scoped, for activity logs) config that routes logs/metrics to a destination like a workspace |

---

## Common Mistakes
- **Assuming policy inheritance flows both directions**: it only flows downward — a policy assigned at `contoso-production` has no effect on `contoso-nonproduction` or on `contoso-root`
- **Assigning Deny-effect policies without testing in Audit/DoNotEnforce mode first**: fine in a greenfield lab, but blocks legitimate deployments the moment it's assigned against an environment with existing resources
- **Letting workspace-per-subscription sprawl happen by default**: every subscription that spins up its own workspace "for convenience" multiplies RBAC surface, duplicates alerting, and loses pooled-ingestion cost benefit — this should be a deliberate decision
- **Forgetting management group deletion order**: Azure refuses to delete a group with subscriptions or child groups still attached
- **Treating table-level RBAC as optional in a centralized workspace**: without it, anyone with Reader on the workspace can query every table, including subscriptions they shouldn't see

---

## Next Steps
This design builds on the diagnostic settings and workspace basics from [AZ-104 Lab 5](../AZ-104/lab-5-monitoring.md) and the governance concepts from [SC-300 Lab 1](../SC-300/lab-1-identity-governance-foundations.md) — this lab adds the multi-subscription topology decision those labs don't need at single-subscription scale. Continue to [Lab 2: Data & Storage Design](lab-2-data-storage-design.md) for the next AZ-305 design domain.
