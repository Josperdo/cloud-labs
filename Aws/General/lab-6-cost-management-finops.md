# Lab 6: Cost Management & FinOps

Check box if done: [ ]

## Overview
"Our AWS bill is too high" is not an actionable finding — "us-east-1 EC2 spend under the `Project=checkout` tag grew 40% month-over-month because three `m5.2xlarge` instances are running at 8% average CPU" is. This lab builds the tooling that turns the first sentence into the second: AWS Budgets with real alert thresholds, Cost Explorer queried and grouped by tag instead of eyeballed in the console, a cost-allocation tagging strategy that actually propagates into billing data, and Compute Optimizer rightsizing recommendations backed by real CloudWatch utilization history.

**Estimated time**: 45–60 minutes
**Cost**: ~$0 — this lab only configures cost-governance tooling; **the Cost Explorer API itself bills $0.01 per request**, a real (if tiny) difference from Azure's free Cost Management API worth knowing for an interview, and one this lab's queries will incur a handful of times

---

## Scenario
Finance flagged a cost overrun with no supporting detail beyond a total dollar figure, and nobody on the team can currently answer "which project, which service, which resource" without manually clicking through the Billing console. You're building the answer: a budget with a real alert threshold instead of finding out about an overrun after the invoice arrives, Cost Explorer queries grouped by the `Project` tag so cost is attributable, an explicit tagging strategy applied to existing resources so future spend is attributable by default, and a rightsizing pass using Compute Optimizer's actual utilization data instead of guessing at instance sizes.

---

## Objectives
- Create an AWS Budget with a threshold alert delivered via SNS
- Query Cost Explorer via CLI, grouped and filtered by cost-allocation tag
- Activate cost-allocation tags and understand the propagation delay into billing data
- Work through the Savings Plans vs. Reserved Instances vs. On-Demand decision for a steady-state workload
- Pull real rightsizing recommendations from Compute Optimizer, with the Trusted Advisor fallback explained where a Business/Enterprise support plan isn't available

---

## Part 1: AWS Budgets With a Threshold Alert

### Step 1: Create an SNS Topic for Notifications
```bash
aws sns create-topic --name lab6-budget-alerts
TOPIC_ARN=$(aws sns list-topics --query "Topics[?ends_with(TopicArn, ':lab6-budget-alerts')].TopicArn" --output text)
aws sns subscribe --topic-arn $TOPIC_ARN --protocol email --notification-endpoint <your-email>
```
Confirm the subscription via the email AWS sends before continuing — an unconfirmed subscription silently receives nothing, the same trap as [Lab 5's](lab-5-observability-apm.md) alarm notifications.

### Step 2: Define the Budget
```bash
cat > budget.json << 'EOF'
{
  "BudgetName": "lab6-monthly-budget",
  "BudgetLimit": {"Amount": "50.0", "Unit": "USD"},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostFilters": {}
}
EOF

cat > notifications.json << 'EOF'
[{
  "Notification": {
    "NotificationType": "ACTUAL",
    "ComparisonOperator": "GREATER_THAN",
    "Threshold": 80,
    "ThresholdType": "PERCENTAGE"
  },
  "Subscribers": [{"SubscriptionType": "SNS", "Address": "TOPIC_ARN_PLACEHOLDER"}]
}]
EOF
sed -i "s|TOPIC_ARN_PLACEHOLDER|$TOPIC_ARN|" notifications.json

aws budgets create-budget \
  --account-id <AWS_ACCOUNT_ID> \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```
`ACTUAL` at 80% fires once real spend crosses the threshold; a `FORECASTED` notification type is worth adding alongside it in a real environment — it warns based on Cost Explorer's spend trend projection *before* the threshold is actually crossed, giving lead time an `ACTUAL`-only budget can't.

**Validation checkpoint**:
```bash
aws budgets describe-budget --account-id <AWS_ACCOUNT_ID> --budget-name lab6-monthly-budget
```
Confirm `BudgetLimit` and the notification threshold match what was defined.

---

## Part 2: Cost Explorer — Query and Group by Tag

### Step 3: Run a Cost-by-Tag Query
```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '-30 days' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=TAG,Key=Project \
  --query "ResultsByTime[0].Groups[*].{Tag:Keys[0],Cost:Metrics.UnblendedCost.Amount}"
```
**Every call to this API costs $0.01** — not a real constraint at lab scale, but worth building the habit of batching and caching Cost Explorer queries in any real cost-reporting automation, rather than calling it per-resource in a loop.

### Step 4: Filter to a Specific Service and Tag Value
```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '-30 days' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --filter '{
    "And": [
      {"Dimensions": {"Key": "SERVICE", "Values": ["Amazon Elastic Compute Cloud - Compute"]}},
      {"Tags": {"Key": "Project", "Values": ["checkout"]}}
    ]
  }'
```
This is the query that answers the scenario's actual question — EC2 spend, scoped to one project tag, over a specific window — instead of a single undifferentiated total.

---

## Part 3: Activate and Apply Cost-Allocation Tags

### Step 5: Why Tags Don't Automatically Appear in Cost Explorer
Tagging a resource in EC2/S3/RDS does **not** automatically make that tag queryable in Cost Explorer or show up on the bill — cost-allocation tags have to be explicitly activated in Billing, and even after activation, **it can take up to 24 hours for tagged usage to start appearing in cost reports.** This is the single most common reason a "we tagged everything" cost-attribution effort looks broken the same day it's rolled out.

```bash
aws ce list-cost-allocation-tags --status Inactive --query "CostAllocationTags[*].TagKey"

aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Project,Status=Active TagKey=Environment,Status=Active
```

### Step 6: Tag Existing Resources
```bash
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list "arn:aws:ec2:us-east-1:<AWS_ACCOUNT_ID>:instance/<instance-id>" \
  --tags Project=checkout,Environment=dev
```
`resourcegroupstaggingapi` can tag many resource types across services in one call — useful for a bulk tagging pass across an account instead of tagging each resource through its own service-specific API.

### Step 7: Define the Scheme, Not Just the Mechanics
A tagging scheme is only useful if it's consistent and enforced, not just possible. A minimal viable scheme for cost allocation:
- `Project` — the cost center / team this resource belongs to (required on everything billable)
- `Environment` — `dev`/`staging`/`prod` (required)
- `ManagedBy` — `terraform`/`cdk`/`manual` (useful for cleanup audits, not just cost)

[Lab 7](lab-7-landing-zone-governance.md) covers **enforcing** this scheme with Tag Policies and SCPs at the Organizations level — this lab only covers applying and querying it.

---

## Part 4: Savings Plans vs. Reserved Instances vs. On-Demand

### Step 8: The Three Options
| | On-Demand | Reserved Instances | Savings Plans |
|---|---|---|---|
| **Discount** | 0% (baseline) | Up to ~72% for 1–3yr commitment | Up to ~66% for 1–3yr commitment |
| **Flexibility** | None needed — pay per second, no commitment | Locked to instance family/region (Standard) or with limited flexibility (Convertible) | Commits to a $/hour spend, applies automatically across instance families, sizes, and (for Compute Savings Plans) even EC2/Fargate/Lambda |
| **Best fit** | Unpredictable or short-lived workloads, bursty dev/test | Steady-state workloads where the exact instance type/family is already known and stable | Steady-state spend where the workload's compute mix might shift over the commitment period |

### Step 9: Work the Decision for a Steady-State Workload
A `m5.xlarge` running 24/7 for a stable production service is the textbook case for leaving On-Demand pricing behind — the question is RI vs. Savings Plan, not whether to commit at all. If the team is confident this exact instance family stays fixed for the next year, a Standard RI captures the largest discount. If there's any chance of migrating instance families, moving workload to Fargate, or adding Lambda into the mix, a Compute Savings Plan trades a few percentage points of discount for that flexibility — and in practice, "we're confident nothing changes for 12 months" is a claim worth being skeptical of.

### Step 10: The Rule of Thumb
Reserve RIs for genuinely fixed, well-understood, long-running instance types (a legacy monolith that isn't migrating anywhere). Default to Savings Plans for anything with a realistic chance of architectural change during the commitment window — the discount gap is small enough that the flexibility is almost always worth it.

---

## Part 5: Rightsizing With Compute Optimizer

### Step 11: Enable Compute Optimizer
```bash
aws compute-optimizer update-enrollment-status --status Active
```
Compute Optimizer needs **14 days of CloudWatch utilization history** before it produces meaningful recommendations — a brand-new instance or a freshly-enrolled account won't have real findings yet. **If you don't have 14 days of history**, this section is still worth working through conceptually: the recommendation output structure below is what you'd see against a real, established workload, and understanding what it flags is the actual skill being tested, not just running the command.

### Step 12: Pull EC2 Rightsizing Recommendations
```bash
aws compute-optimizer get-ec2-instance-recommendations \
  --query "instanceRecommendations[*].{Instance:instanceArn,Current:currentInstanceType,Finding:finding,Recommended:recommendationOptions[0].instanceType}"
```
`finding` values (`Underprovisioned`, `Overprovisioned`, `Optimized`, `NotOptimized`) are the headline signal — `Overprovisioned` is the direct rightsizing opportunity, backed by actual observed CPU/memory/network utilization rather than a guess.

### Step 13: Trusted Advisor as an Alternative — With a Caveat
Trusted Advisor also surfaces rightsizing recommendations, but **the full set of cost-optimization checks (including "Low Utilization Amazon EC2 Instances") requires a Business or Enterprise Support plan** — the AWS Free Tier and Basic/Developer support plans only expose 7 core checks, none of which include rightsizing. If you're on Basic support:
```bash
aws support describe-trusted-advisor-checks --language en \
  --query "checks[?category=='cost_optimizing'].name"
```
This will return an empty or minimal list without upgraded support — Compute Optimizer (Step 11–12) is the free-tier-accessible equivalent and is the tool to actually rely on unless the account genuinely carries Business/Enterprise support.

---

## Cleanup

```bash
# Budget
aws budgets delete-budget --account-id <AWS_ACCOUNT_ID> --budget-name lab6-monthly-budget

# SNS topic
aws sns delete-topic --topic-arn $TOPIC_ARN

# Deactivate the cost-allocation tags if you don't want them tracked going forward
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Project,Status=Inactive TagKey=Environment,Status=Inactive

# Disable Compute Optimizer enrollment if this account shouldn't stay enrolled
aws compute-optimizer update-enrollment-status --status Inactive
```
No compute resources were created by this lab — cleanup here is entirely configuration teardown, not resource deletion.

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Cost-allocation tag activation** | A separate, explicit Billing-console/API step required before a resource tag becomes queryable in Cost Explorer or visible on the Cost and Usage Report — tagging alone isn't enough |
| **Cost Explorer API cost** | $0.01 per `GetCostAndUsage`-family API call — a real, if small, AWS-specific line item that Azure's equivalent free API doesn't have |
| **AWS Budgets `ACTUAL` vs. `FORECASTED`** | `ACTUAL` fires once real spend crosses a threshold; `FORECASTED` fires based on Cost Explorer's trend projection, giving lead time before the threshold is actually hit |
| **Savings Plan vs. Reserved Instance** | Both trade a 1–3yr commitment for a discount; RIs lock to an instance family/region, Savings Plans commit to a $/hour spend that applies flexibly across families and (for Compute Savings Plans) services |
| **Compute Optimizer 14-day minimum** | Compute Optimizer needs two weeks of CloudWatch history before recommendations are meaningful — a fresh account or instance won't show real findings yet |
| **Trusted Advisor support-plan gating** | Full cost-optimization checks require Business/Enterprise Support; Basic support exposes only 7 core checks, none of them cost-related — Compute Optimizer is the free-tier-accessible alternative |

---

## Common Mistakes
- **Tagging resources and expecting same-day visibility in Cost Explorer**: cost-allocation tags need explicit activation, and even after activation there's up to a 24-hour propagation delay before tagged usage appears in reports
- **Calling the Cost Explorer API in a tight loop**: each call costs $0.01 — fine occasionally, a real line item if a cost-reporting script calls it per-resource instead of batching queries
- **Assuming Trusted Advisor rightsizing is available on any support plan**: it's gated behind Business/Enterprise Support specifically — Compute Optimizer is the correct free-tier tool, not a lesser substitute
- **Defaulting to Reserved Instances for every steady-state workload**: fine for genuinely fixed infrastructure, a bad trade if there's any realistic chance the instance family or compute platform changes during the commitment window — Savings Plans exist specifically for that uncertainty
- **Setting a budget with only an `ACTUAL` notification and no `FORECASTED` one**: means finding out about an overrun only after it's already happened, rather than getting a trend-based warning ahead of it

---

## Next Steps
This lab covers FinOps tooling depth [SAA-C03 Lab 7](../SAA-C03/aws-lab-7-well-architected-cost.md) doesn't — Budgets API automation, tag-activation propagation, and Compute Optimizer specifically, versus that lab's broader Well-Architected cost review. The tagging scheme defined in Part 3 is enforced at scale (not just applied) in [Lab 7: Landing Zone & Governance](lab-7-landing-zone-governance.md) via Tag Policies and SCPs. For the IAM permissions a real cost-reporting automation would need scoped correctly, see [SAA-C03 Lab 1](../SAA-C03/aws-lab-1-vpc-iam.md).
