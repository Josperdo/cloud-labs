# Lab 8: Capstone — Well-Architected Landing Zone

Check box if done: [ ]

## Overview
Every lab in this track answered one design question in isolation: how to structure accounts (Lab 1), how to pick storage and database services (Lab 2), how to survive a regional outage (Lab 3), how to host an application (Lab 4), how to build the network (Lab 5), how to go multi-region (Lab 6), how to migrate a legacy estate (Lab 7). A real design review is never one of those questions — it's all seven, asked about the same company, at the same time. This capstone doesn't introduce a new domain. It takes one realistic scenario, runs it through every decision framework built in Labs 1–7, deploys a scaled-down governed landing zone that demonstrates the pattern, and closes with a Well-Architected Framework review pass against what was actually built.

**Estimated time**: 100–120 minutes
**Cost**: ~$2–$5 (an OU hierarchy and SCP cost nothing; one small ECS Fargate service, a DynamoDB table, a budget alert, and a brief logging setup — nothing here is a large hourly meter, and Transit Gateway from Lab 5 is deliberately not redeployed, a call explained in Part 3)

---

## Scenario
Meridian Freight has grown by acquisition — three business units (Domestic Logistics, Cold Chain, and a recently acquired startup, RouteSense) each provisioned their own AWS account independently over the last two years. There's no shared Organization, no consistent tagging, inconsistent database choices made for no documented reason, one business unit with no cross-region DR plan at all, a hand-rolled EC2 deployment pipeline in one account that another account already solved with ECS blue/green, and RouteSense still runs its core telemetry-ingestion service on a colocated data center server the acquisition agreement requires migrating within nine months. The CFO wants cost visibility across all three business units in one place. The CISO wants one governance model, not three. You're the architect writing the design response that consolidates this into a single governed landing zone — and proving the pattern works with a scaled-down deployment before asking for budget to do it at full scale.

---

## Objectives
- Decide between continuing three independent accounts and consolidating into an AWS Organization with OU-based governance
- Apply Labs 1–7's decision frameworks to Meridian's three business units and synthesize the results into one design response
- Deploy a scaled-down landing zone: an OU hierarchy, one guardrail SCP, centralized logging, and one governed workload
- Layer a cost and governance overlay (tagging-driven cost allocation, budget alerts) on top of the deployed structure
- Run a Well-Architected Framework review pass against the Cost Optimization, Reliability, Security, and Operational Excellence pillars for what was actually built

---

## Part 1: Design Decision — Landing Zone Topology and Guardrail Enforcement

### Decision 1: Continue Three Independent Accounts vs. AWS Organization With OU-Based Landing Zone

| Factor | Three Independent Accounts (current state) | AWS Organization with OU-Based Landing Zone |
|---|---|---|
| **Governance consistency** | Each business unit's IAM, tagging, and logging setup is whatever that team happened to configure — already proven inconsistent | One SCP and one organization trail, assigned once at the right OU, apply identically to all three business units and to RouteSense's eventual migrated footprint |
| **Cost visibility** | Manual aggregation across three disconnected accounts with no shared tagging to group by | Consolidated billing plus a mandatory cost-center tag give the CFO one place to look, the same pattern Lab 1 built |
| **Blast radius of a new acquisition onboarding badly** | Each acquisition repeats the ungoverned setup RouteSense is in today | A new account enters through the `Workloads` OU and inherits every guardrail immediately |
| **Migration effort to get there** | None — status quo | Real but bounded: existing accounts join the Organization and move into the right OU (an account-membership operation, not a workload migration); nothing about the workloads themselves moves |
| **Fit for this scenario** | Fails both the CISO's "one governance model" ask and the CFO's cost-visibility ask directly | Satisfies both, and gives RouteSense's eventual migration (Lab 7's 6 R's) a governed landing zone to migrate *into* instead of a fourth ungoverned account |

### Decision 2: Guardrail Enforcement Point — Organization-Wide SCP vs. Per-Account IAM Policy

| Factor | Organization-Wide SCP (attached at the Workloads OU) | Per-Account IAM Policy |
|---|---|---|
| **Consistency** | One assignment, inherited by every account underneath, present and future | Exactly today's problem — three accounts, three interpretations |
| **New business unit onboarding** | Automatic the moment an account lands in the right OU | Manual, repeated, and how today's inconsistency happened in the first place |
| **Enforcement strength** | Applies as a permission ceiling regardless of local IAM — even a local admin can't override it | Only as strong as whoever configured it correctly in that specific account, that specific time |
| **Best fit** | This scenario — native tooling, zero incremental cost, directly solves the stated inconsistency | Nothing at this scale — it's the cause of the current mess, not a fix for it |

### Recommendation for This Scenario
An **AWS Organization with a `Security` OU** (holding a log-archive account pattern) and a **`Workloads` OU** holding each business unit — Domestic Logistics, Cold Chain, and (once Lab 7's migration lands it in AWS instead of on-prem) RouteSense. Guardrails enforced through **one SCP and one organization trail assigned at the `Workloads` OU**, reusing Lab 1's exact pattern at three-business-unit scale instead of one. Part 2 builds a scaled-down version of exactly this shape.

---

## Part 2: Build the Scaled-Down OU Hierarchy and Guardrail

A full deployment would model all three business units as separate AWS accounts under the `Workloads` OU, matching [Lab 1](lab-1-multi-account-organizational-design.md)'s full shape. This capstone builds the minimum structure that proves the pattern: one `Workloads` OU, one business unit represented by your existing account, and a guardrail that would inherit identically to a second or third business unit added later.

### Step 1: Build the OU Hierarchy
```bash
aws organizations describe-organization || aws organizations create-organization --feature-set ALL
ROOT_ID=$(aws organizations list-roots --query "Roots[0].Id" --output text)

aws organizations create-organizational-unit --parent-id $ROOT_ID --name "Security"
aws organizations create-organizational-unit --parent-id $ROOT_ID --name "Workloads"
WORKLOADS_OU_ID=$(aws organizations list-organizational-units-for-parent --parent-id $ROOT_ID --query "OrganizationalUnits[?Name=='Workloads'].Id" --output text)

# One business unit modeled explicitly — Domestic Logistics. Cold Chain and (post-migration)
# RouteSense would each get their own sibling account under this same OU.
aws organizations move-account --account-id <your-account-id> --source-parent-id $ROOT_ID --destination-parent-id $WORKLOADS_OU_ID
```

### Step 2: Assign a Consolidated Guardrail SCP
Reuses [Lab 1](lab-1-multi-account-organizational-design.md)'s guardrail pattern, with the tag set extended to include `businessUnit`, which Part 4's cost overlay groups by.
```bash
aws organizations create-policy --name MeridianLandingZoneGuardrails --type SERVICE_CONTROL_POLICY --content '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyOutsideApprovedRegions", "Effect": "Deny",
    "NotAction": ["iam:*", "organizations:*", "sts:*", "support:*", "cloudfront:*", "route53:*"],
    "Resource": "*",
    "Condition": {"StringNotEquals": {"aws:RequestedRegion": ["us-east-1", "us-west-2"]}}
  }, {
    "Sid": "DenyDisablingOrgTrail", "Effect": "Deny",
    "Action": ["cloudtrail:StopLogging", "cloudtrail:DeleteTrail"], "Resource": "*"
  }]
}'

POLICY_ID=$(aws organizations list-policies --filter SERVICE_CONTROL_POLICY --query "Policies[?Name=='MeridianLandingZoneGuardrails'].Id" --output text)
aws organizations attach-policy --policy-id $POLICY_ID --target-id $WORKLOADS_OU_ID
```
Assigned at `Workloads`, not per-business-unit — the entire point being that Cold Chain or RouteSense onboarding tomorrow inherits this identically, with zero repeated setup.

### Step 3: Centralized Logging
```bash
LOG_BUCKET="sap-c02-lab8-orgtrail-<your-unique-suffix>"
aws s3api create-bucket --bucket $LOG_BUCKET --region us-east-1
ORG_ID=$(aws organizations describe-organization --query "Organization.Id" --output text)

aws s3api put-bucket-policy --bucket $LOG_BUCKET --policy '{
  "Version": "2012-10-17", "Statement": [
    {"Sid": "AWSCloudTrailAclCheck", "Effect": "Allow", "Principal": {"Service": "cloudtrail.amazonaws.com"}, "Action": "s3:GetBucketAcl", "Resource": "arn:aws:s3:::'"$LOG_BUCKET"'"},
    {"Sid": "AWSCloudTrailWrite", "Effect": "Allow", "Principal": {"Service": "cloudtrail.amazonaws.com"}, "Action": "s3:PutObject", "Resource": "arn:aws:s3:::'"$LOG_BUCKET"'/AWSLogs/'"$ORG_ID"'/*", "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}}}
  ]
}'

aws cloudtrail create-trail --name meridian-org-trail --s3-bucket-name $LOG_BUCKET --is-organization-trail --is-multi-region-trail --enable-log-file-validation
aws cloudtrail start-logging --name meridian-org-trail
```

**Validation checkpoint**:
```bash
aws organizations list-accounts-for-parent --parent-id $WORKLOADS_OU_ID --query "Accounts[].Name" --output table
aws cloudtrail get-trail-status --name meridian-org-trail --query "IsLogging"
```
Confirm your account is nested under `Workloads` and the trail reports `IsLogging: true`.

---

## Part 3: One Governed Workload — the Scaled-Down Compute and Data Pattern

A full deployment would give Domestic Logistics its own Transit Gateway-attached spoke VPC, exactly as [Lab 5](lab-5-network-infrastructure-design.md) built, shared hub-spoke infrastructure across every business unit. For a capstone demonstration, that hourly-billed meter isn't justified just to prove the pattern — this is the deliberate cost-conscious call Lab 5's Decision 1 already flagged: a single business unit's workload doesn't yet need transitive multi-VPC routing. A real Meridian Freight deployment would size this decision the same way Lab 5 did, against the actual number of business-unit VPCs; this capstone makes the call to skip Transit Gateway here and says so explicitly, rather than silently.

### Step 4: Deploy a Tagged, Governed Workload
Reusing Lab 2's DynamoDB pattern (key-based access, unpredictable scale) as the stand-in for Domestic Logistics' shipment-tracking service, now carrying the `businessUnit` tag the cost overlay in Part 4 depends on:
```bash
KEY_ID=$(aws kms create-key --description "sap-c02-lab8 - Domestic Logistics CMK" --query "KeyMetadata.KeyId" --output text)

aws dynamodb create-table \
  --table-name MeridianShipmentTracking \
  --attribute-definitions AttributeName=ShipmentId,AttributeType=S \
  --key-schema AttributeName=ShipmentId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=$KEY_ID \
  --tags Key=businessUnit,Value=DomesticLogistics Key=costCenter,Value=DL-3000
```

**Validation checkpoint**:
```bash
aws dynamodb describe-table --table-name MeridianShipmentTracking \
  --query "Table.{SSE:SSEDescription.Status,Tags:TableArn}"
aws dynamodb list-tags-of-resource --resource-arn $(aws dynamodb describe-table --table-name MeridianShipmentTracking --query "Table.TableArn" --output text)
```
Confirm SSE is `ENABLED` (Lab 2's encryption pattern) and both tags are present (the tagging discipline the SCP in a fuller deployment would enforce, and Part 4 depends on).

---

## Part 4: Cost and Governance Overlay

### Step 5: Cost Visibility Grouped by Business Unit
```bash
aws budgets create-budget --account-id <your-account-id> --budget '{
  "BudgetName": "meridian-domesticlogistics-monthly",
  "BudgetLimit": {"Amount": "500", "Unit": "USD"},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostFilters": {"TagKeyValue": ["user:businessUnit$DomesticLogistics"]}
}' --notifications-with-subscribers '[{
  "Notification": {"NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN", "Threshold": 80},
  "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "<your-email>"}]
}]'
```
Filtering the budget by the `businessUnit` tag is what makes this the CFO's "one place to look" — a production rollout would set one such budget per business unit and pair it with a Cost Explorer view grouped by the same tag across all three accounts, giving cross-business-unit visibility from a single pane.

**Validation checkpoint**:
```bash
aws budgets describe-budget --account-id <your-account-id> --budget-name meridian-domesticlogistics-monthly --query "Budget.BudgetLimit"
```

---

## Part 5: Design Synthesis

This is the actual deliverable the scenario asked for — not any single control, but how seven independent lab decisions reinforce one governed landing zone for Meridian Freight.

| SAP-C02 Domain | Design Decision | Built In | How It Applies to Meridian |
|---|---|---|---|
| **Organizational Complexity** | OU hierarchy + org-wide SCP + centralized trail | [Lab 1](lab-1-multi-account-organizational-design.md) | Replaces three accounts' inconsistent setups with one inherited guardrail set, extended here with a `businessUnit` tag for cost allocation |
| **Data & Storage/Database** | Service selection by access pattern, CMK encryption | [Lab 2](lab-2-data-storage-database-design.md) | Resolves "inconsistent database choices made for no documented reason" — Domestic Logistics' shipment tracking gets a service justified by its actual key-based access pattern, not habit |
| **Business Continuity** | RTO/RPO-driven DR strategy selection (Warm Standby) | [Lab 3](lab-3-business-continuity-design.md) | Directly closes "one business unit with no cross-region DR plan at all" — the framework applies regardless of which business unit has the gap |
| **Compute & Application Architecture** | Hosting-model decision matrix, blue/green deployment | [Lab 4](lab-4-compute-application-architecture-design.md) | Replaces the hand-rolled EC2 pipeline one account built with the ECS blue/green pattern another account already solved — one deployment standard instead of two |
| **Network Infrastructure** | Transit Gateway hub-spoke, PrivateLink, WAF placement | [Lab 5](lab-5-network-infrastructure-design.md) | The full-scale target once a second business unit's VPC needs shared, transitively-routed connectivity — Part 3 above deploys its cost-conscious single-VPC alternative deliberately, not by accident |
| **Multi-Region & Global Distribution** | Latency+failover Route 53, CloudFront origin failover, Global Tables vs. Aurora Global | [Lab 6](lab-6-multiregion-global-distribution-design.md) | Answers a future international-expansion requirement with a specific, justified routing and data-replication choice instead of an unaddressed gap |
| **Migration & Modernization** | The 6 R's, DMS/MGN workflows, strangler-fig | [Lab 7](lab-7-migration-modernization-design.md) | Directly the RouteSense requirement — a nine-month deadline to move off a colocated server, assessed workload-by-workload rather than defaulted to a blanket rehost |
| **Landing Zone (this lab)** | OU-based guardrails, cost/governance overlay | This lab | The container all seven other decisions land inside — without it, each business unit re-derives its own version of Labs 1–7 independently, exactly the inconsistency Meridian started with |

**The thread connecting all eight**: Meridian's three business units didn't fail because any one of them made a bad individual decision — Domestic Logistics' database choice, Cold Chain's DR gap, and the hand-rolled EC2 pipeline are each defensible in isolation. They failed because there was no shared structure for those decisions to land inside, so every business unit re-solved the same problems independently and inconsistently. A landing zone isn't an additional control layered on top of Labs 1–7 — it's the governance and account structure that makes those decisions apply consistently across every business unit, present and future, instead of once per team that happens to ask the right question.

---

## Part 6: Well-Architected Framework Review

A design isn't done when it's deployed — it's done when it's been checked against the pillars a real Well-Architected review actually uses. This reviews what Parts 2–4 built, honestly, including where the scaled-down capstone deliberately falls short of a full production posture.

| Pillar | What Was Built | Gap at Full Production Scale |
|---|---|---|
| **Cost Optimization** | On-demand DynamoDB (no idle capacity cost), a tagged budget alert at 80% threshold, Transit Gateway deliberately not deployed for a single-VPC workload | A full deployment adds Transit Gateway costs the moment a second business unit needs shared connectivity — that's a real, planned cost increase, not scope creep, and should be modeled before it happens, not discovered on the first multi-BU bill |
| **Reliability** | Organization trail guarantees every account's activity is audited regardless of who forgets to configure it locally | This capstone does **not** deploy Lab 3's Warm Standby DR or Lab 6's multi-region routing for the workload built here — a full deployment needs both, and their absence here is this review's most important honest finding, not an oversight to gloss over |
| **Security** | SCP-enforced region restriction and CloudTrail-tamper prevention apply organization-wide from day one; DynamoDB encrypted with a customer-managed KMS key, not the default | Delegated security administration (GuardDuty/Security Hub/Config centralization, per [SCS-C02 Lab 7](../SCS-C02/aws-sec-lab-7-multi-account-governance.md)) isn't built here — this capstone's SCPs are a real but partial governance layer, not the full security baseline a production Organization needs |
| **Operational Excellence** | One guardrail SCP and one trail, both inherited automatically by any future account in the `Workloads` OU — zero repeated setup per business unit going forward | Tagging enforcement here is by convention (Part 3 manually tagged the table) rather than by policy — a production rollout should make `businessUnit`/`costCenter` tags mandatory via SCP or a Config rule, the way [Lab 1](lab-1-multi-account-organizational-design.md)'s initiative pattern does for mandatory tags, closing the gap between "tagged because someone remembered" and "tagged because it's structurally required" |

**Why the gaps matter more than the wins**: A Well-Architected review that only lists what's already good isn't a review, it's a victory lap. The Reliability and Security gaps named above are the actual next line items in Meridian's real migration plan — this capstone's job was proving the landing zone *pattern* at small scale, not delivering a finished production posture, and being explicit about which parts are proof-of-concept versus production-ready is itself part of the deliverable a design review expects.

---

## Cleanup

```bash
# 1. Budget
aws budgets delete-budget --account-id <your-account-id> --budget-name meridian-domesticlogistics-monthly

# 2. DynamoDB table and KMS key
aws dynamodb delete-table --table-name MeridianShipmentTracking
aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7

# 3. Organization trail
aws cloudtrail stop-logging --name meridian-org-trail
aws cloudtrail delete-trail --name meridian-org-trail
aws s3 rm s3://$LOG_BUCKET --recursive
aws s3api delete-bucket --bucket $LOG_BUCKET

# 4. Detach and delete the SCP
aws organizations detach-policy --policy-id $POLICY_ID --target-id $WORKLOADS_OU_ID
aws organizations delete-policy --policy-id $POLICY_ID

# 5. Move the account back to root, then delete OUs leaf-to-root
aws organizations move-account --account-id <your-account-id> --source-parent-id $WORKLOADS_OU_ID --destination-parent-id $ROOT_ID
SECURITY_OU_ID=$(aws organizations list-organizational-units-for-parent --parent-id $ROOT_ID --query "OrganizationalUnits[?Name=='Security'].Id" --output text)
aws organizations delete-organizational-unit --organizational-unit-id $WORKLOADS_OU_ID
aws organizations delete-organizational-unit --organizational-unit-id $SECURITY_OU_ID
```
Confirm with `aws organizations list-organizational-units-for-parent --parent-id $ROOT_ID` (should return empty) and `aws dynamodb list-tables` (no `MeridianShipmentTracking`).

---

## What You Practiced

| Task | Why It Matters on the Job |
|---|---|
| Applying seven independent decision frameworks to one shared scenario | This is the actual SAP-C02 exam skill and the actual job — no design review asks about accounts, storage, or network in isolation, it asks about all of them for one company at once |
| Choosing a landing zone topology over continued account sprawl | Recognizing when the fix isn't another point solution but a shared structure those solutions can consistently land inside |
| Deploying a scaled-down proof of a pattern instead of the full enterprise build | A design review rarely gets budget for the full build up front — showing the pattern works at small scale is how that budget gets approved |
| Naming a deliberate cost-vs-completeness trade-off (skipping Transit Gateway) explicitly | Separates a defensible design decision from a shortcut nobody documented |
| Running a Well-Architected review that reports real gaps, not just wins | An honest review is the actual deliverable a stakeholder needs — a review that only confirms what's already fine isn't doing its job |

---

## Common Mistakes to Avoid
- **Treating each SAP-C02 domain as a standalone answer instead of a system**: real scenario questions (and real design reviews) frequently require identifying how a gap in one domain undermines another — RouteSense's migration (Lab 7) has nowhere governed to land without the landing zone (this lab) existing first
- **Building the full multi-account landing zone before proving the pattern**: a scaled-down deployment that demonstrates the shape is what gets a design approved for budget — building all three business units' full infrastructure before anyone signs off is backwards
- **Skipping the tagging design and bolting on cost allocation later**: the `businessUnit` tag has to exist before resources are created for cost visibility to be complete — retrofitting tags onto an untagged estate is a much larger cleanup project
- **Running a Well-Architected review that only lists strengths**: the review in Part 6 is only useful because it names what this capstone deliberately didn't build (DR, multi-region, delegated security admin) — a review without gaps isn't a review
- **Omitting the explicit cost/completeness trade-off from the design writeup**: skipping Transit Gateway for this capstone's single-VPC workload is defensible; skipping it silently, without naming the trade-off and the condition (a second business unit's VPC) that would flip the decision, is not

---

## Next Steps
This closes out the SAP-C02 track — [Lab 1](lab-1-multi-account-organizational-design.md) through [Lab 7](lab-7-migration-modernization-design.md) each built one design domain in isolation; this capstone is where they converge into a single governed architecture, the same way a real design review or interview question would require. For the security-governance depth this lab's SCPs and centralized trail deliberately scaled down, see [SCS-C02 Lab 7: Multi-Account Security & Governance](../SCS-C02/aws-sec-lab-7-multi-account-governance.md), which goes deeper on delegated administration for GuardDuty, Security Hub, and Config than this lab re-teaches. For the foundational AWS skills this entire track assumed, see [SAA-C03](../SAA-C03/README.md).
