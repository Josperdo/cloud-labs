# Lab 7: Migration & Modernization Design

Check box if done: [ ]

## Overview
"Migrate it to the cloud" is not a design decision — it's five or six different possible decisions wearing one sentence. SAP-C02's migration domain tests whether you can look at a specific workload's constraints (timeline, licensing, technical debt, business value) and pick the right one of the 6 R's, not default to "rehost everything, sort it out later." This lab builds that decision matrix across a small portfolio of workloads, then works the two migration tools the exam expects you to know operationally — Database Migration Service and Application Migration Service — plus the strangler-fig pattern for modernizing a legacy app incrementally instead of in one risky cutover.

**Estimated time**: 75–90 minutes
**Cost**: ~$1–$4 (a small DMS replication instance and two minimal RDS/EC2 endpoints for the duration of the migration demo, an API Gateway with negligible request volume — all deleted at the end)

---

## Scenario
Your company is closing its last data center in nine months and migrating four applications to AWS. The team's instinct is to lift-and-shift everything on the same timeline, but a five-minute look at each app shows that's wrong for at least half of them: a legacy inventory system on an unsupported OS with no vendor support left, a commercial CRM the company pays an ISV to license and could instead consume as SaaS, a monolithic order-processing app the team actually wants to invest in modernizing over time (not just move as-is), and an internal reporting tool nobody has opened in fourteen months. Four apps, four different correct answers — and the exam (and the actual migration plan) expects you to justify each one individually.

---

## Objectives
- Apply the 6 R's migration framework to a small application portfolio and justify each pick individually
- Understand what AWS Database Migration Service (DMS) actually automates versus what still requires manual assessment
- Understand the AWS Application Migration Service (MGN) workflow, and why it's console/agent-driven rather than CLI-first for this lab
- Run a real DMS replication task migrating data between two database endpoints
- Deploy a strangler-fig pattern routing incrementally between a legacy backend and a modernized one, without a single big-bang cutover

---

## Part 1: Design Decision — The 6 R's Applied to a Four-App Portfolio

### Step 1: The Framework
| R | What it means | Effort | When it's correct |
|---|---|---|---|
| **Rehost** | "Lift and shift" — move the workload as-is, no code changes, typically VM-to-VM | Lowest | Time-constrained migrations, workloads with no near-term modernization budget, or a first step before a later replatform |
| **Replatform** | Move with minor, targeted changes (e.g., swap self-managed MySQL for RDS) without touching application architecture | Low-moderate | The workload benefits from a managed service swap but a full rearchitecture isn't justified |
| **Repurchase** | Replace with a different product, usually SaaS, dropping the self-hosted version entirely | Low (migration effort), but real switching/retraining cost | A commercial or commodity capability exists as SaaS and self-hosting it provides no differentiation |
| **Refactor / Re-architect** | Rebuild using cloud-native patterns — often the only path to real elasticity, resilience, or feature-velocity gains | Highest | Business-critical workloads planned for ongoing, active investment, where the current architecture is the limiting factor |
| **Retire** | Decommission — don't migrate it at all | Lowest (execution), but requires confirming nobody actually depends on it | Workloads with negligible or zero verified usage |
| **Retain** | Leave it where it is, for now | None (this migration cycle) | Workloads with a hard dependency (licensing, compliance, imminent replacement) that makes migrating now not worth the effort |

### Step 2: Apply the Framework to Each of the Four Apps

| App | Constraint | Chosen R | Why |
|---|---|---|---|
| **Legacy inventory system** | Unsupported OS, no vendor support left | **Rehost**, immediately, onto a supported AMI | The data center closure deadline doesn't allow time for a rearchitecture, and an unsupported OS is itself the most urgent risk — get it onto AWS and a patchable OS first, revisit modernization later as a separate project |
| **Commercial CRM (ISV-licensed)** | Company pays for a license to run software it doesn't differentiate on | **Repurchase** — move to the vendor's SaaS offering if one exists | Self-hosting a commodity CRM provides no competitive advantage; migrating the self-hosted version is real effort spent preserving a status quo worth abandoning instead |
| **Monolithic order-processing app** | Business wants ongoing investment, not just relocation | **Refactor** — but not on this migration's nine-month clock | This is the workload where "just get it to AWS" would be the wrong call even though it's technically the most business-critical: a Refactor scoped to this deadline risks both the migration and the modernization. Correct answer: **Rehost now** to hit the deadline, **Refactor afterward** as a separate, deliberately-scoped project — this is the "Rehost as staging step" pattern the framework explicitly allows for |
| **Internal reporting tool, unused for 14 months** | No verified active usage | **Retire** — but only after confirming with the actual business owner, not assuming from log data alone | Migrating a workload nobody uses is pure wasted migration effort; the discipline point is confirming retirement with a human owner, since "no logs" sometimes means "used by a process, not a person" |

**The pattern to internalize**: one company, one deadline, four different correct answers — and the order-processing app shows that even "Refactor" as the *eventual* right answer doesn't mean Refactor is right *for this migration wave*. Sequencing (Rehost now, Refactor later) is itself a legitimate design decision the 6 R's framework supports.

---

## Part 2: Database Migration Service — Assessment and Migration Workflow

DMS automates the data-movement mechanics of a database migration (initial full load plus ongoing change data capture) — it does not automate the *decision* of which R applies, or schema conversion for heterogeneous engine changes (that's AWS Schema Conversion Tool's job, run before DMS for a cross-engine migration).

### Step 3: Create a Replication Instance
```bash
aws dms create-replication-instance \
  --replication-instance-identifier sap-c02-lab7-dms \
  --replication-instance-class dms.t3.micro \
  --allocated-storage 20 \
  --no-publicly-accessible \
  --engine-version 3.5.2
```

### Step 4: Define Source and Target Endpoints
This lab migrates between two RDS PostgreSQL instances (standing in for "legacy self-managed database" → "managed RDS target") — the same mechanics apply for a true heterogeneous migration (e.g., Oracle → Aurora PostgreSQL), just with AWS SCT run first to convert the schema.

```bash
aws dms create-endpoint \
  --endpoint-identifier sap-c02-lab7-source \
  --endpoint-type source \
  --engine-name postgres \
  --server-name <source-db-endpoint> \
  --port 5432 \
  --database-name inventory \
  --username labadmin \
  --password "<your-password>"

aws dms create-endpoint \
  --endpoint-identifier sap-c02-lab7-target \
  --endpoint-type target \
  --engine-name postgres \
  --server-name <target-db-endpoint> \
  --port 5432 \
  --database-name inventory \
  --username labadmin \
  --password "<your-password>"
```

### Step 5: Test Both Endpoint Connections Before Creating the Task
Skipping this step is the most common reason a migration task fails hours into a full load instead of immediately.
```bash
REP_INSTANCE_ARN=$(aws dms describe-replication-instances --query "ReplicationInstances[0].ReplicationInstanceArn" --output text)
SOURCE_ARN=$(aws dms describe-endpoints --filters Name=endpoint-id,Values=sap-c02-lab7-source --query "Endpoints[0].EndpointArn" --output text)
TARGET_ARN=$(aws dms describe-endpoints --filters Name=endpoint-id,Values=sap-c02-lab7-target --query "Endpoints[0].EndpointArn" --output text)

aws dms test-connection --replication-instance-arn $REP_INSTANCE_ARN --endpoint-arn $SOURCE_ARN
aws dms test-connection --replication-instance-arn $REP_INSTANCE_ARN --endpoint-arn $TARGET_ARN
```

### Step 6: Create and Start a Migration Task — Full Load + CDC
`full-load-and-cdc` migrates existing data first, then keeps applying ongoing changes — this is what enables a near-zero-downtime cutover, since the source stays writable and in sync right up until the actual cutover moment, instead of requiring a write-freeze for the entire migration duration.

```bash
aws dms create-replication-task \
  --replication-task-identifier sap-c02-lab7-task \
  --source-endpoint-arn $SOURCE_ARN \
  --target-endpoint-arn $TARGET_ARN \
  --replication-instance-arn $REP_INSTANCE_ARN \
  --migration-type full-load-and-cdc \
  --table-mappings '{"rules":[{"rule-type":"selection","rule-id":"1","rule-name":"1","object-locator":{"schema-name":"public","table-name":"%"},"rule-action":"include"}]}'

TASK_ARN=$(aws dms describe-replication-tasks --filters Name=replication-task-id,Values=sap-c02-lab7-task --query "ReplicationTasks[0].ReplicationTaskArn" --output text)
aws dms start-replication-task --replication-task-arn $TASK_ARN --start-replication-task-type start-replication
```

**Validation checkpoint**:
```bash
aws dms describe-replication-tasks --filters Name=replication-task-id,Values=sap-c02-lab7-task \
  --query "ReplicationTasks[0].{Status:Status,Percent:ReplicationTaskStats.FullLoadProgressPercent}"
```
Confirm `Status` progresses from `starting` → `running`, and `FullLoadProgressPercent` reaches 100. Once full load completes and `Status` shows `running` with CDC active, any new write to the source table should appear in the target within seconds — that ongoing sync is the mechanism that eventually lets cutover happen with a write-freeze measured in minutes, not the hours a one-shot full load alone would require.

---

## Part 3: AWS Application Migration Service — the Workflow (Console/Agent-First)

Console-first note, same convention this repo uses elsewhere for setup steps that are inherently agent- or wizard-driven: MGN requires installing the **AWS Replication Agent** directly on each source server (physical, VM, or another cloud) — there is no CLI equivalent to "install an agent on a machine that isn't provisioned by AWS," so this section documents the workflow rather than executing it against a real source server this lab doesn't have.

### Step 7: Understand the MGN Workflow
1. **Install the Replication Agent** on each source server — it begins continuous, block-level replication to a staging area in your target AWS account/region, without interrupting the source server's normal operation
2. **Group source servers into a wave** — a batch of servers migrating together, typically grouped by application dependency (a database server and its application server belong in the same wave)
3. **Launch a test instance** from the replicated data — this validates the migrated server boots and functions correctly in AWS *without* touching the production cutover, directly analogous to Lab 3's ASR-style test failover discipline: validate before you commit
4. **Perform the cutover** — once testing passes, launch the actual cutover instance from the most current replicated data, and redirect production traffic to it
5. **Decommission the source server** once the cutover instance is confirmed stable

### Step 8: Why This Matters for the Legacy Inventory System's Rehost
The Part 1 decision picked Rehost for the legacy inventory system specifically because of its deadline pressure — MGN's continuous block-level replication (not a one-time image copy) is what makes that Rehost low-risk: the source server keeps running and replicating right up until test and cutover are both confirmed good, rather than requiring a single all-or-nothing image capture with no chance to validate first.

---

## Part 4: Strangler-Fig Modernization Pattern

The order-processing app's decision in Part 1 was "Rehost now, Refactor later, as a separate project" — the strangler-fig pattern is *how* that later refactor actually happens without a risky big-bang rewrite: new functionality is built as separate, modern services, and a routing layer incrementally shifts traffic path-by-path from the legacy monolith to the new services, until nothing routes to the legacy system anymore and it can be retired.

### Step 9: Deploy a Legacy Backend Stand-In and a Modernized Lambda Function
```bash
aws lambda create-function \
  --function-name sap-c02-lab7-modern-orders \
  --runtime python3.12 \
  --role arn:aws:iam::<your-account-id>:role/lambda-basic-execution \
  --handler index.handler \
  --zip-file fileb://modern-orders.zip
```
`modern-orders.zip` contains a minimal handler returning a JSON response marked `"source": "modernized-service"` — a stand-in for a genuinely rebuilt endpoint. The legacy backend is any existing HTTP endpoint (an EC2/ALB target, or simply a second Lambda tagged `"source": "legacy-monolith"` for this lab's purposes) representing the monolith's untouched code.

### Step 10: Create an API Gateway Routing Some Paths to Legacy, Some to Modern
```bash
API_ID=$(aws apigatewayv2 create-api --name sap-c02-lab7-strangler --protocol-type HTTP --query "ApiId" --output text)

# New order-creation flow: routed to the modernized service
MODERN_INTEGRATION=$(aws apigatewayv2 create-integration --api-id $API_ID \
  --integration-type AWS_PROXY --integration-uri arn:aws:lambda:<region>:<your-account-id>:function:sap-c02-lab7-modern-orders \
  --payload-format-version 2.0 --query "IntegrationId" --output text)
aws apigatewayv2 create-route --api-id $API_ID --route-key "ANY /orders/create" --target integrations/$MODERN_INTEGRATION

# Everything else: still routed to the legacy backend, untouched
LEGACY_INTEGRATION=$(aws apigatewayv2 create-integration --api-id $API_ID \
  --integration-type AWS_PROXY --integration-uri arn:aws:lambda:<region>:<your-account-id>:function:sap-c02-lab7-legacy-monolith \
  --payload-format-version 2.0 --query "IntegrationId" --output text)
aws apigatewayv2 create-route --api-id $API_ID --route-key "ANY /{proxy+}" --target integrations/$LEGACY_INTEGRATION

aws apigatewayv2 create-stage --api-id $API_ID --stage-name '$default' --auto-deploy
```
The catch-all `/{proxy+}` route to the legacy backend, alongside one explicitly modernized path (`/orders/create`), is the strangler-fig pattern's actual mechanism: every future modernized capability gets its own explicit route added the same way, incrementally shrinking what the catch-all rule still needs to cover — the legacy monolith is "strangled" one route at a time, never replaced in a single cutover.

**Validation checkpoint**:
```bash
API_ENDPOINT=$(aws apigatewayv2 get-api --api-id $API_ID --query "ApiEndpoint" --output text)
curl $API_ENDPOINT/orders/create   # expect {"source": "modernized-service", ...}
curl $API_ENDPOINT/orders/12345    # expect {"source": "legacy-monolith", ...}
```
Confirm the two paths return different `source` values from the same API — proof that traffic is genuinely split by route, with the legacy system still fully functional for everything not yet migrated.

---

## Cleanup

```bash
# 1. API Gateway
aws apigatewayv2 delete-api --api-id $API_ID

# 2. Lambda functions
aws lambda delete-function --function-name sap-c02-lab7-modern-orders
aws lambda delete-function --function-name sap-c02-lab7-legacy-monolith

# 3. DMS task, endpoints, replication instance (in this order — the instance can't delete while attached to a task)
aws dms stop-replication-task --replication-task-arn $TASK_ARN
aws dms delete-replication-task --replication-task-arn $TASK_ARN
aws dms delete-endpoint --endpoint-arn $SOURCE_ARN
aws dms delete-endpoint --endpoint-arn $TARGET_ARN
aws dms delete-replication-instance --replication-instance-arn $REP_INSTANCE_ARN

# 4. Underlying RDS endpoints used for the DMS demo
aws rds delete-db-instance --db-instance-identifier <source-db-identifier> --skip-final-snapshot
aws rds delete-db-instance --db-instance-identifier <target-db-identifier> --skip-final-snapshot
```
The DMS replication instance is the item most likely to be forgotten — it bills hourly like any other provisioned instance regardless of whether a task is actively running against it.

Confirm with `aws dms describe-replication-instances --query "ReplicationInstances[?ReplicationInstanceIdentifier=='sap-c02-lab7-dms']"` — should return empty.

---

## Key Concepts

| Term | Definition |
|---|---|
| **The 6 R's** | Rehost, Replatform, Repurchase, Refactor/Re-architect, Retire, Retain — the framework for classifying the correct migration strategy per workload, not a single answer applied to an entire portfolio |
| **AWS Database Migration Service (DMS)** | Automates the data-movement mechanics of a database migration (full load + change data capture); does not automate schema conversion for heterogeneous engines (that's AWS SCT) or the R decision itself |
| **Full load + CDC (Change Data Capture)** | A DMS migration type that copies existing data once, then continuously replicates ongoing changes — enables cutover with a short write-freeze instead of a long one |
| **AWS Application Migration Service (MGN)** | Agent-based, continuous block-level replication of source servers to AWS, enabling test launches before an actual production cutover — the standard tool for Rehost migrations |
| **Wave (migration planning)** | A group of source servers migrated together, typically grouped by application dependency so related systems cut over in the same event |
| **Strangler-fig pattern** | Incrementally routing traffic path-by-path from a legacy system to new services until the legacy system serves nothing and can be retired, avoiding a single high-risk big-bang rewrite |

---

## Common Mistakes
- **Defaulting every workload in a portfolio to Rehost because it's the fastest path to "done"**: it's the right call for time-constrained or dependency-fragile workloads, but applying it universally skips real Repurchase/Retire savings and defers Refactor work that should be explicitly planned, not silently dropped
- **Treating "Refactor eventually" and "Rehost now" as contradictory**: they're sequential, not competing — a workload can correctly get both, in that order, across two separate projects
- **Skipping `test-connection` before starting a DMS task**: a bad connection string or security group rule surfaces immediately with `test-connection` and hours later, mid-full-load, without it
- **Retiring a workload based on log silence alone**: "no logs" can mean "genuinely unused" or "used by a batch process that ran successfully and generated no error logs" — confirm with the business owner before decommissioning
- **Building a strangler-fig catch-all route that never shrinks**: the pattern only works if new capability is deliberately carved out of the catch-all over time — an API Gateway config that adds modernized routes but never revisits or measures what's still hitting the legacy catch-all isn't actually strangling anything

---

## Next Steps
This lab assumes comfort with the Lambda and API Gateway basics from [SAA-C03 Lab 6: Serverless Architecture](../SAA-C03/aws-lab-6-serverless.md). It's the migration counterpart to [Lab 4: Compute & Application Architecture Design](lab-4-compute-application-architecture-design.md)'s hosting-model decisions — a Refactor target chosen here is exactly the kind of decision that lab's EC2/ECS/EKS/Lambda comparison matrix would be applied to. Continue to [Lab 8: Capstone — Well-Architected Landing Zone](lab-8-capstone-well-architected-landing-zone.md), which closes the track by synthesizing every prior lab's decisions, including this one's migration wave planning, into a single governed landing zone.
