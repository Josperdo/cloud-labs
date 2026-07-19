# Lab 6: Cost Management & FinOps

Check box if done: [ ]

## Overview
"How do you keep cloud spend under control?" is a real interview question at any company past a certain size, and it doesn't have a hand-wavy answer — the concrete response is budgets with alerts, Advisor's cost recommendations, tagging that lets finance actually allocate spend to a team, and a defensible reservation-vs-pay-as-you-go decision. This lab builds a small but complete version of that practice against a real (or reused) subscription.

**Estimated time**: 45–60 minutes
**Cost**: ~$0 (this lab only configures cost governance tooling — no billable compute beyond whatever existing resources you point it at)

---

## Scenario
You're the closest thing to a FinOps function this org has — there's no dedicated cost team, but leadership wants monthly Azure spend visibility and guardrails before the org scales further. You're setting up a budget with threshold alerts so nobody finds out about an overspend from the invoice, pulling Advisor's cost recommendations to find waste that's already sitting in the subscription, applying a cost-allocation tagging scheme so spend can be attributed to a team or project, and working through the reserved-capacity-vs-pay-as-you-go decision for a steady-state workload.

---

## Objectives
- Create a monthly budget with 50%/80%/100% threshold alerts wired to a notification target
- Pull and interpret Azure Advisor's cost recommendations
- Apply a consistent cost-allocation tagging scheme and query spend by tag
- Work through the Reserved Instance / Savings Plan vs. pay-as-you-go decision with real math
- Identify and rightsize an oversized VM based on observed utilization, not guesswork

---

## Part 1: Create a Budget with Threshold Alerts

### Step 1: Create the Budget
A budget with no alert wired to it is just a number nobody looks at. This one fires at 50% (early warning), 80% (getting serious), and 100% (over) of the monthly amount.

```bash
az consumption budget create \
  --budget-name "lab6-monthly-budget" \
  --category Cost \
  --amount 100 \
  --time-grain Monthly \
  --start-date 2026-07-01 \
  --end-date 2027-06-30 \
  --notifications '{
    "Actual_GreaterThan_50_Percent": {
      "enabled": true, "operator": "GreaterThan", "threshold": 50,
      "contactEmails": ["<your-email>@example.com"], "contactRoles": ["Owner"]
    },
    "Actual_GreaterThan_80_Percent": {
      "enabled": true, "operator": "GreaterThan", "threshold": 80,
      "contactEmails": ["<your-email>@example.com"], "contactRoles": ["Owner"]
    },
    "Actual_GreaterThan_100_Percent": {
      "enabled": true, "operator": "GreaterThan", "threshold": 100,
      "contactEmails": ["<your-email>@example.com"], "contactRoles": ["Owner"]
    }
  }'
```

Replace `<your-email>@example.com` with your own address, and adjust `--amount` to whatever's realistic for your subscription's actual spend. Add `--resource-group <your-resource-group>` to scope the budget to a single resource group instead of the whole subscription — useful once you're tracking spend per team or environment rather than one subscription-wide number.

**Validation checkpoint**: `az consumption budget list --query "[].{name:name, amount:amount, timeGrain:timeGrain}" -o table` should list the budget(s) you created. In the portal, **Cost Management + Billing** → **Budgets** should show the same, with the threshold percentages visible under the budget's alert configuration.

---

## Part 2: Review Azure Advisor Cost Recommendations

### Step 3: Pull Cost Recommendations
Advisor evaluates actual usage patterns across the subscription and flags waste it can see directly — it's the fastest way to find low-effort savings before doing anything manual.

```bash
az advisor recommendation list \
  --category Cost \
  --query "[].{impact:impact, resource:impactedValue, description:shortDescription.problem}" \
  -o table
```

### Step 4: Read What It's Actually Telling You
Advisor's cost category typically surfaces a handful of recurring patterns:

- **Underutilized VMs** — running well below provisioned CPU/memory over a sustained window; suggested action is resize or shut down
- **Orphaned/unattached disks** — a managed disk not attached to any VM, still billing hourly; suggested action is delete (after confirming it's not a deliberate backup/snapshot source)
- **Reservation opportunities** — steady-state VM or SQL usage running long enough that Advisor calculates an RI would be cheaper than continued pay-as-you-go
- **Unused App Service plans / idle resources** — provisioned but effectively zero-traffic, same shutdown-or-downsize logic as underutilized VMs

**Validation checkpoint**: Your `az advisor recommendation list` output should show at least one recommendation type with a populated `resource` and `description` field (if the subscription is brand-new with minimal usage history, this may be empty for a few days — Advisor needs usage data to generate cost recommendations, it doesn't evaluate instantaneously).

---

## Part 3: Apply a Cost-Allocation Tagging Scheme

### Step 5: Define the Scheme
Three tags cover most cost-allocation needs: who pays for it, what environment it's in, and who's accountable for it.

| Tag | Example Value | Purpose |
|-----|---------------|---------|
| `costCenter` | `IT-1000` | Which budget line this resource bills against |
| `environment` | `prod`, `nonprod` | Separates production spend from dev/test noise in reports |
| `owner` | `<team-alias>` | Who to contact before deleting or resizing this resource |

### Step 6: Tag Existing Resources
```bash
az resource tag \
  --tags costCenter=IT-1000 environment=nonprod owner=<your-username> \
  --ids $(az resource list --resource-group <your-resource-group> --query "[].id" -o tsv)
```

For a single resource:
```bash
az tag update \
  --resource-id <resource-id> \
  --operation Merge \
  --tags costCenter=IT-1000 environment=nonprod owner=<your-username>
```

`--operation Merge` adds/updates these tags without clobbering any existing ones — `Replace` would wipe out tags you didn't specify.

### Step 7: Query Spend by Tag
```bash
az costmanagement query \
  --type Usage \
  --timeframe MonthToDate \
  --dataset-aggregation '{"totalCost":{"name":"PreTaxCost","function":"Sum"}}' \
  --dataset-grouping name=costCenter type=TagKey \
  --scope "/subscriptions/<your-subscription-id>"
```

Same query in the portal: **Cost Management + Billing** → **Cost analysis** → **Group by** → select `costCenter` (or `environment`, `owner`) from the tag list.

**This pairs directly with the tag-enforcement Policy initiative from [AZ-305 Lab 1](../AZ-305/lab-1-identity-governance-monitoring-design.md).** Tagging only works as a cost-allocation tool if it's actually enforced — a Policy initiative with `Deny` effect on missing `costCenter`/`environment` tags is what turns this from "a scheme people can ignore" into "a scheme that's structurally true of every resource in the subscription." Manually tagging resources after the fact (this lab) demonstrates the querying side; the Policy lab demonstrates the enforcement side.

**Validation checkpoint**: `az resource show --ids <resource-id> --query tags` returns the three tags you set. The `az costmanagement query` (or portal Cost analysis grouped view) returns a per-`costCenter` cost breakdown — it may show `$0.00` for a low-usage lab subscription, but the grouping itself should work.

---

## Part 4: Reserved Instances / Savings Plans vs. Pay-As-You-Go

This part is decision math, not a hands-on purchase — buying a real reservation commits real money for the term, which doesn't belong in a $0 lab.

### Step 8: Know the Three Options
- **Pay-as-you-go (PAYG)**: billed per second/hour of actual usage, no commitment. Right for bursty, short-lived, or unpredictable workloads — exactly what most of these labs are.
- **Reserved Instance (RI)**: a 1- or 3-year commitment to a specific VM size/region in exchange for a discount. Right for workloads you know will run continuously for the term.
- **Azure Savings Plan**: a 1- or 3-year commitment to a *dollar amount per hour* of compute spend (not a specific VM size), with more flexibility to shift between VM families/regions than an RI, at a slightly smaller discount.

### Step 9: Work the Decision for a 24/7 Steady-State VM
Illustrative, approximate numbers — vendor discount tiers change over time, so treat this as reasoning shape, not a quote. Always confirm current numbers with the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/) before making a real commitment.

| Commitment | Relative Cost (illustrative) | Notes |
|------------|-------------------------------|-------|
| Pay-as-you-go | 100% (baseline) | No commitment, full flexibility, most expensive per-hour rate |
| 1-year Reserved Instance | ~60–70% of PAYG | Vendor-typical 30–40% discount for locking in 1 year |
| 3-year Reserved Instance | ~40–50% of PAYG | Vendor-typical 50–60% discount, largest discount, least flexibility |
| Savings Plan (1-year) | Slightly above equivalent RI | Trades a bit of discount for the ability to move spend across VM families/regions |

### Step 10: Apply the Rule of Thumb
**RI/Savings Plan** make sense when a workload is predictable and effectively always-on — a production database, a steady-traffic web tier, anything you'd be uncomfortable turning off. **PAYG** is correct for anything bursty or short-lived — every lab VM in this repo, dev/test environments shut down nightly, anything you're not confident will still exist in a year. A 1-year RI on a VM deleted in month two is a sunk cost with no exit. Break-even question before buying either: **"Am I more than ~60–70% confident this exact VM size, in this exact region, will still be running a year from now?"** If not, PAYG is the safer default.

---

## Part 5: Rightsizing Exercise

### Step 11: Identify an Oversized VM
Use Advisor's underutilization signal (Part 2) or pull raw utilization from Azure Monitor directly:

```bash
az monitor metrics list \
  --resource <vm-resource-id> \
  --metric "Percentage CPU" \
  --interval PT1H \
  --start-time 2026-07-07T00:00:00Z \
  --end-time 2026-07-14T00:00:00Z \
  --aggregation Average Maximum \
  -o table
```

Look at both **Average** and **Maximum**, not average alone — a VM that averages 8% CPU but spikes to 90% during a nightly batch job is not a rightsizing candidate; one that averages 8% and peaks at 15% is.

### Step 12: Pick a Smaller SKU That Still Covers Observed Peak
```bash
az vm list-sizes --resource-group <your-resource-group> --query "[].name" -o table
```
Choose a SKU with headroom over the observed **maximum**, not the average — resizing to fit the average will throttle the workload the next time it spikes.

### Step 13: Resize the VM
```bash
az vm deallocate --resource-group <your-resource-group> --name <vm-name>
az vm resize --resource-group <your-resource-group> --name <vm-name> --size Standard_B2s
az vm start --resource-group <your-resource-group> --name <vm-name>
```

Most SKU changes require the VM to be deallocated first — resizing a running VM in-place only works between sizes on the same hardware cluster, which isn't guaranteed. Deallocate/resize/start is the reliable path.

**Validation checkpoint**: `az vm show --resource-group <your-resource-group> --name <vm-name> --query hardwareProfile.vmSize` returns the new, smaller SKU. Re-run the Azure Monitor query from Step 11 after a day of normal usage to confirm CPU headroom is still reasonable (not pegged at 90%+ average).

---

## Cleanup
1. Delete any budget created purely for this exercise: `az consumption budget delete --budget-name lab6-monthly-budget` (and `lab6-rg-budget` if created) — or leave it running if it's scoped to a subscription you're continuing to use, since budgets cost nothing and this is exactly the guardrail you'd want long-term
2. Remove lab-only tags if they were applied to shared/pre-existing resources you don't want permanently tagged: `az tag update --resource-id <resource-id> --operation Delete --tags costCenter environment owner`
3. If you resized a VM purely to demonstrate `az vm resize`, resize it back to its original SKU (or leave the smaller size if it now correctly matches observed load — that's the actual point of the exercise)
4. No reservations were purchased in this lab, so there's no reservation cancellation/refund process to worry about

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Budget** | A spend threshold tracked over a time grain (monthly/quarterly/annually), scoped to a subscription or resource group |
| **Cost alert / notification** | The action tied to a budget threshold (email, action group) — a budget without one is invisible until someone checks the portal |
| **Azure Advisor (Cost category)** | Recommendation engine that evaluates actual usage telemetry to flag underutilized resources, orphaned disks, and reservation opportunities |
| **Cost-allocation tagging** | A consistent tag scheme (`costCenter`, `environment`, `owner`) that lets Cost Management attribute spend to a team, project, or environment |
| **Reserved Instance (RI)** | A 1- or 3-year commitment to a specific VM size/region for a discounted rate; loses flexibility, gains savings |
| **Azure Savings Plan** | A commitment to a dollar-per-hour compute spend rather than a specific SKU — more flexible than an RI, slightly smaller discount |
| **Rightsizing** | Changing a resource's SKU to match observed utilization instead of provisioned/guessed capacity |

---

## Common Mistakes
- **Creating a budget with no notification wired up**: the alert exists on paper but nobody gets told when it fires — check `contactEmails`/`contactRoles` are actually populated
- **Tagging inconsistently**: `CostCenter` vs `costCenter` vs `cost-center` are three different keys to Cost Management, not one — case and naming drift silently breaks cost-by-tag reporting
- **Buying a Reserved Instance for a workload that isn't actually steady-state**: a 1- or 3-year commitment on a VM that gets deleted or resized in month two is a sunk cost with no exit
- **Rightsizing based on peak load instead of the full picture**: resizing to fit the average alone throttles workloads with legitimate periodic spikes (nightly batch jobs, month-end processing)
- **Treating tagging as optional guidance**: without Policy enforcement, tagging schemes decay within weeks as resources get created without them — see Part 3's note on pairing this with Policy

---

## Next Steps
This lab's tagging scheme is only as strong as its enforcement — see [AZ-305 Lab 1](../AZ-305/lab-1-identity-governance-monitoring-design.md) for the Policy initiative that makes `costCenter`/`environment` tags mandatory rather than optional, and the root [README](../../README.md) for the cost-and-safety philosophy this whole repo follows. Continue to [Lab 7: Landing Zone & Governance at Scale](lab-7-landing-zone-governance.md) for how these guardrails scale from a single subscription to a full management group hierarchy.
