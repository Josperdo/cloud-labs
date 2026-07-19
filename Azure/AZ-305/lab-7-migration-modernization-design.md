# Lab 7: Migration & Modernization Design

Check box if done: [ ]

## Overview
AZ-305 doesn't test whether you can lift-and-shift a VM — it tests whether you can look at a portfolio of legacy workloads and assign each one the *right* migration strategy, because "rehost everything" and "refactor everything" are both wrong answers most of the time, just in opposite directions. This lab builds the 5 R's decision framework against a realistic legacy portfolio, walks the Azure Migrate assessment workflow that feeds that decision with real data, deploys an actual offline database migration to make the online/offline trade-off concrete, and implements the strangler-fig pattern for incrementally modernizing a monolith without a risky big-bang cutover.

**Estimated time**: 75–90 minutes
**Cost**: ~$1–$4 (a Basic-tier Azure SQL Database for the migration exercise, two small App Service instances for the strangler-fig demo, Front Door Standard's small hourly base charge — nothing here is a large always-on meter; delete everything the same session)

---

## Scenario
You're the architect running a discovery-and-planning engagement for a company migrating out of an aging on-prem datacenter before a lease expires in six months. Product and IT hand you a mixed bag: a file server nobody wants to touch, a decade-old order-management monolith the business depends on daily, a custom-built CRM two staff maintain part-time, an internal reporting dashboard with a handful of monthly viewers, and a SQL Server database backing all of it. Leadership's instruction is blunt: "just move it all to Azure as-is, we don't have time for a rewrite." You know that's the wrong default for at least some of these workloads, and the design review is where you have to show why — workload by workload, not as a blanket policy.

---

## Objectives
- Build a 5 R's (Rehost/Replatform/Refactor/Repurchase/Retire) decision matrix and apply it to five distinct legacy workloads
- Understand the Azure Migrate discovery → assessment → right-sizing workflow and what each phase actually produces
- Decide between online and offline database migration based on downtime tolerance
- Perform an actual offline database migration using export/import, and understand what Azure Database Migration Service adds for the online case
- Implement the strangler-fig pattern: route new functionality to a modernized service while a legacy app keeps serving everything else

---

## Part 1: Design Decision — The 5 R's Against a Real Portfolio

### Step 1: The Framework
| Strategy | What Changes | Effort | Risk | Best Fit |
|---|---|---|---|---|
| **Rehost** ("lift and shift") | Infrastructure only — VM moves to Azure, application binary untouched | Lowest | Lowest — no code path changes to validate | Time-constrained migrations, workloads with no near-term modernization budget, apps too fragile or undocumented to safely touch |
| **Replatform** ("lift, tinker, and shift") | Minor changes to take advantage of a managed service (e.g., self-hosted SQL Server → Azure SQL Database) without changing app architecture | Low-medium | Low-medium — the app logic is unchanged, only its dependencies | Workloads with sound core logic but avoidable operational overhead (patching, backups, HA) in their current form |
| **Refactor** | Application architecture changes — decomposing a monolith, adopting managed compute (Functions, Container Apps) | High | Medium-high — real code changes need real testing | High business-value workloads where the current architecture is actively limiting feature velocity or scalability |
| **Repurchase** | Replace the custom app with a SaaS/COTS product | Medium (migration/data effort), low (build effort) | Medium — data migration and process-fit risk, but no code to maintain afterward | Custom-built systems duplicating functionality a mature SaaS product already provides well |
| **Retire** | Decommission — the workload isn't migrated at all | Lowest | Lowest, if usage is confirmed genuinely gone | Workloads with no current business use, confirmed via actual usage data, not assumption |

### Step 2: Apply It to This Scenario's Portfolio
| Workload | Current State | Constraint | Recommended Strategy | Why |
|---|---|---|---|---|
| **File server** | Windows file share, no app logic, just storage | Needs to be available quickly, no budget to redesign file access patterns right now | **Rehost** → Azure Files or a rehosted VM | Zero application logic to refactor; this is a storage-migration problem, not an architecture problem — solve it with the cheapest, fastest option |
| **Order-management monolith** | Decade-old, business-critical, daily use, increasingly hard to add features to | High business value, but also high risk if changed carelessly under a 6-month deadline | **Replatform now, Refactor later (strangler-fig)** | The deadline rules out a full refactor before the lease expires — replatform onto Azure SQL Database and App Service now for the operational win, then modernize incrementally post-migration using Part 4's pattern rather than a risky big-bang rewrite under time pressure |
| **Custom CRM** | Built in-house years ago, duplicates functionality mature CRM SaaS products already offer | Two part-time maintainers, ongoing maintenance burden the business doesn't want to keep funding | **Repurchase** | Building and maintaining bespoke CRM logic isn't a differentiator for this business — a SaaS CRM with a data-migration effort costs less over 3 years than continuing to carry this app's maintenance burden |
| **Internal reporting dashboard** | Handful of monthly viewers, unclear owner | Low usage, no one has been able to name an active stakeholder | **Retire** (pending usage confirmation) | Migrating an app with confirmed near-zero usage spends real migration effort protecting something nobody needs — confirm via access logs before retiring, but don't default to migrating it "just in case" |
| **SQL Server database (backs the monolith)** | On-prem SQL Server, backing the order-management system | Tied directly to the monolith's Replatform decision above | **Replatform** → Azure SQL Database | Moves with the monolith's Replatform decision — Part 3 executes this specific migration |

**The exam-relevant discipline here**: the instruction to "just move it all as-is" is a Rehost-everything default, and it's wrong for at least three of these five workloads. AZ-305 scenario questions test exactly this — resisting a one-size-fits-all migration strategy in favor of a workload-by-workload assessment.

---

## Part 2: Azure Migrate — Discovery, Assessment, and Right-Sizing

Standing up a full on-prem source environment (physical or virtualized servers, a network path to them, and the Azure Migrate discovery appliance) is out of scope for a single-session lab — the same constraint the rest of this repo handles by walking the workflow conceptually against the real portal surface, rather than skipping it. If you have a lab VM environment (on-prem or another cloud) available, the steps below are directly executable against it.

### Step 3: Understand the Three-Phase Workflow
1. **Discovery**: the Azure Migrate appliance (a VM deployed inside the source environment) agentlessly inventories running servers — OS, installed software, dependencies between servers (which servers talk to which, over which ports) — without installing anything on the source machines themselves
2. **Assessment**: Azure Migrate compares discovered server specs (CPU, memory, disk IOPS, network throughput) against Azure VM SKUs and recommends a right-sized target, along with an estimated monthly cost — this is where "rehost this VM" turns into "rehost this VM as a Standard_D4s_v5"
3. **Readiness reporting**: flags workloads that aren't migration-ready as-is — an OS version past Azure support, a dependency on local hardware (a license dongle, a specific NIC), or a discovered dependency on a server not included in the current migration wave

### Step 4: Run It (If You Have a Source Environment) or Review the Surface (If Not)
**If you have a lab VM environment**:
1. Portal → **Azure Migrate** → **Create project** → select subscription and resource group
2. **Discover** → **Servers, databases and web apps** → download and deploy the Azure Migrate appliance OVA/VHD into the source environment, register it with the project during setup
3. Let discovery run for at least 24–48 hours in a real environment (dependency mapping needs observed traffic over time) before assessing
4. **Assess** → select discovered servers → choose assessment type (**Azure VM** for rehost, **Azure SQL** for a database Replatform assessment) → review the generated readiness and right-sizing report

**If you don't have a source environment**: review **Azure Migrate** → **Discover** and **Assess** in the portal to see the configuration surface and the shape of a completed assessment report (Microsoft Learn's Azure Migrate documentation includes sample assessment output) — the workflow and its outputs are the exam-relevant knowledge here, not running it against live infrastructure.

**Expected result of a completed assessment**: a per-server readiness verdict (Ready / Ready with conditions / Not ready), a recommended Azure VM SKU with a confidence rating, and a monthly cost estimate — this is the data that should drive the 5 R's decision in Part 1, not intuition about what a server "probably" needs.

---

## Part 3: Database Migration — Online vs. Offline

### Step 5: The Decision
| Factor | Offline Migration | Online Migration (Azure Database Migration Service, continuous sync) |
|---|---|---|
| **Downtime** | Application is down for the duration of the export/import | Near-zero — DMS performs an initial full sync, then applies ongoing changes continuously until a short, planned cutover window |
| **Complexity** | Low — export, transfer, import, repoint the connection string | Higher — requires a stable network path to the source for the sync duration, and monitoring the replication lag before cutover |
| **Tooling** | `az sql db export`/`import`, SSMS, or `bacpac` file transfer | Azure Database Migration Service (VM-based, billed hourly for the duration the service instance runs) |
| **Best fit** | Workloads that can tolerate a maintenance window (nights/weekends, low-traffic apps) | Business-critical workloads with no acceptable downtime window (the "6-month lease deadline" pressure in this scenario doesn't itself justify online migration — that's about *downtime tolerance*, a separate axis) |
| **Risk during migration** | Lower — no live sync to monitor, but a longer irreversible window once cutover starts | Requires validating replication lag stays low and stable before triggering cutover; a network blip during sync just delays convergence rather than failing the whole migration |

**Recommendation for the order-management database**: the scenario doesn't state a zero-downtime requirement — a scheduled maintenance window is acceptable for this internal system. **Offline migration** is the right call here, executed in Part 3's hands-on steps below. A customer-facing, always-on transactional system would flip this to Azure Database Migration Service's online path instead.

### Step 6: Execute an Offline Migration (Export → Import)
```bash
RG="rg-az305-lab7"
LOCATION="eastus"
SQL_SERVER="sql-az305-lab7-<your-unique-suffix>"
ADMIN_USER="sqladmin"
ADMIN_PASSWORD="<your-strong-password>"

az group create --name $RG --location $LOCATION

az sql server create --name $SQL_SERVER --resource-group $RG --location $LOCATION \
  --admin-user $ADMIN_USER --admin-password "$ADMIN_PASSWORD"

az sql server firewall-rule create --resource-group $RG --server $SQL_SERVER \
  --name AllowMyIp --start-ip-address <your-public-ip> --end-ip-address <your-public-ip>

# "Source" database standing in for the on-prem order-management DB
az sql db create --resource-group $RG --server $SQL_SERVER --name db-lab7-legacy \
  --edition Basic
```

Create a storage account to hold the exported `.bacpac` file:
```bash
STORAGE_ACCOUNT="stlab7migrate<your-unique-suffix>"
az storage account create --name $STORAGE_ACCOUNT --resource-group $RG \
  --location $LOCATION --sku Standard_LRS

az storage container create --account-name $STORAGE_ACCOUNT --name migration-exports
```

Export the source database:
```bash
az sql db export --resource-group $RG --server $SQL_SERVER --name db-lab7-legacy \
  --admin-user $ADMIN_USER --admin-password "$ADMIN_PASSWORD" \
  --storage-key-type StorageAccessKey \
  --storage-key "$(az storage account keys list --resource-group $RG --account-name $STORAGE_ACCOUNT --query [0].value -o tsv)" \
  --storage-uri "https://$STORAGE_ACCOUNT.blob.core.windows.net/migration-exports/db-lab7-legacy.bacpac"
```
This is the offline migration's actual downtime window in miniature — during export, the source database is still technically online, but any writes that happen after the export starts won't be in the `.bacpac` unless the migration plan accounts for a final short freeze before this step.

Import into the target database (standing in for the modernized Azure SQL target):
```bash
az sql db import --resource-group $RG --server $SQL_SERVER --name db-lab7-modernized \
  --admin-user $ADMIN_USER --admin-password "$ADMIN_PASSWORD" \
  --storage-key-type StorageAccessKey \
  --storage-key "$(az storage account keys list --resource-group $RG --account-name $STORAGE_ACCOUNT --query [0].value -o tsv)" \
  --storage-uri "https://$STORAGE_ACCOUNT.blob.core.windows.net/migration-exports/db-lab7-legacy.bacpac" \
  --edition Basic
```

**Validation checkpoint**:
```bash
az sql db list --resource-group $RG --server $SQL_SERVER --query "[].name" -o tsv
```
Confirm both `db-lab7-legacy` and `db-lab7-modernized` exist, and that the import operation reports `Succeeded` — poll with `az sql db show --name db-lab7-modernized --resource-group $RG --server $SQL_SERVER --query status` if the import is still running.

### Step 7: What Azure Database Migration Service Adds for the Online Case
DMS wasn't needed above because offline was the right call for this workload — but understand what changes if it weren't: `az dms create` provisions a migration service instance (billed hourly while it runs), then a migration project connects a source (the on-prem SQL Server, reached over a VPN/ExpressRoute path or a self-hosted integration runtime) to the target Azure SQL Database. DMS performs an initial full data copy, then keeps applying transactional changes continuously — the source stays live and serving production traffic the entire time. Cutover is a short, planned window (typically minutes) where writes are briefly paused, the last few seconds of replication lag catch up, and the application's connection string repoints to the target — dramatically shorter downtime than the export/import approach above, at the cost of needing sustained connectivity to the source for the sync duration and a running (billed) DMS instance.

---

## Part 4: Strangler-Fig Pattern for Incremental Modernization

### Step 8: The Pattern
The order-management monolith from Part 1 was assigned "Replatform now, Refactor later" — this part builds the "later." Instead of rewriting the whole monolith at once (high risk, long timeline, no incremental value delivered), new functionality gets built as a separate, modern service and routed to selectively, while the legacy app keeps serving everything else. Over time, more routes move to the new service until the legacy app has nothing left to serve and can be retired — the legacy app is "strangled" one route at a time rather than replaced in one risky cutover.

### Step 9: Deploy the Legacy App and the New Modernized Service
```bash
SUFFIX="<your-unique-suffix>"

az appservice plan create --name asp-lab7-legacy --resource-group $RG \
  --location $LOCATION --sku B1 --is-linux

az webapp create --name app-lab7-legacy-$SUFFIX --resource-group $RG \
  --plan asp-lab7-legacy --runtime "NODE:20-lts"

az appservice plan create --name asp-lab7-modern --resource-group $RG \
  --location $LOCATION --sku B1 --is-linux

az webapp create --name app-lab7-modern-$SUFFIX --resource-group $RG \
  --plan asp-lab7-modern --runtime "NODE:20-lts"
```
In a real strangler-fig migration, the "legacy" app here would be the actual existing monolith deployed as-is (matching Part 1's Replatform decision), and only the "modern" app is genuinely new code — this lab deploys placeholder apps for both to demonstrate the routing pattern without requiring the real monolith's source.

### Step 10: Front Both with Path-Based Routing
```bash
az afd profile create --profile-name afd-lab7-strangler --resource-group $RG \
  --sku Standard_AzureFrontDoor

az afd endpoint create --resource-group $RG --profile-name afd-lab7-strangler \
  --endpoint-name lab7-app-$SUFFIX --enabled-state Enabled

az afd origin-group create --resource-group $RG --profile-name afd-lab7-strangler \
  --origin-group-name legacy-origins --probe-request-type GET --probe-protocol Https --probe-path /
az afd origin create --resource-group $RG --profile-name afd-lab7-strangler \
  --origin-group-name legacy-origins --origin-name legacy \
  --host-name app-lab7-legacy-$SUFFIX.azurewebsites.net \
  --origin-host-header app-lab7-legacy-$SUFFIX.azurewebsites.net \
  --priority 1 --weight 1000 --http-port 80 --https-port 443 --enabled-state Enabled

az afd origin-group create --resource-group $RG --profile-name afd-lab7-strangler \
  --origin-group-name modern-origins --probe-request-type GET --probe-protocol Https --probe-path /
az afd origin create --resource-group $RG --profile-name afd-lab7-strangler \
  --origin-group-name modern-origins --origin-name modern \
  --host-name app-lab7-modern-$SUFFIX.azurewebsites.net \
  --origin-host-header app-lab7-modern-$SUFFIX.azurewebsites.net \
  --priority 1 --weight 1000 --http-port 80 --https-port 443 --enabled-state Enabled

# New functionality route — only /api/v2/* goes to the modernized service
az afd route create --resource-group $RG --profile-name afd-lab7-strangler \
  --endpoint-name lab7-app-$SUFFIX --route-name modern-route \
  --origin-group modern-origins --patterns-to-match "/api/v2/*" \
  --supported-protocols Https Http --link-to-default-domain Enabled --forwarding-protocol MatchRequest

# Everything else still goes to the legacy monolith
az afd route create --resource-group $RG --profile-name afd-lab7-strangler \
  --endpoint-name lab7-app-$SUFFIX --route-name legacy-route \
  --origin-group legacy-origins --patterns-to-match "/*" \
  --supported-protocols Https Http --link-to-default-domain Enabled --forwarding-protocol MatchRequest
```
This is the strangler-fig mechanism made concrete: `/api/v2/*` is where new feature work lands going forward, and every other path keeps hitting the untouched legacy app — no big-bang cutover, no risk to the paths that already work.

**Validation checkpoint**: `curl https://lab7-app-$SUFFIX.z01.azurefd.net/` should route to the legacy origin group, and `curl https://lab7-app-$SUFFIX.z01.azurefd.net/api/v2/anything` should route to the modern origin group — confirm both by checking each App Service's own request logs (**App Service → Log stream**) to see which one actually received each request.

### Step 11: The Incremental Path Forward
Each subsequent sprint, another route pattern (`/api/v2/orders/*`, `/api/v2/customers/*`, and so on) gets added to `modern-route` and removed from what `legacy-route` still needs to serve, until the legacy app's remaining surface area shrinks to nothing and it can finally be decommissioned — completing the Refactor that a 6-month deadline made too risky to attempt as a single cutover in Part 1.

---

## Cleanup
```bash
az afd profile delete --resource-group $RG --profile-name afd-lab7-strangler --yes

az webapp delete --name app-lab7-legacy-$SUFFIX --resource-group $RG
az webapp delete --name app-lab7-modern-$SUFFIX --resource-group $RG
az appservice plan delete --name asp-lab7-legacy --resource-group $RG --yes
az appservice plan delete --name asp-lab7-modern --resource-group $RG --yes

az sql server delete --name $SQL_SERVER --resource-group $RG --yes

az group delete --name $RG --yes --no-wait
```

---

## Key Concepts

| Term | Definition |
|---|---|
| **The 5 R's** | Rehost, Replatform, Refactor, Repurchase, Retire — a framework for assigning the right-sized migration effort per workload instead of one blanket strategy |
| **Azure Migrate discovery appliance** | An agentless VM deployed in the source environment that inventories servers and maps dependencies between them, without installing software on the source machines |
| **Right-sizing** | Matching a discovered on-prem server's actual observed utilization (not its allocated spec) to the smallest Azure VM SKU that comfortably fits it — avoids paying for oversized VMs built to match an over-provisioned on-prem server |
| **Offline vs. online database migration** | Offline accepts a downtime window in exchange for simplicity (export/import); online (DMS) keeps the source live via continuous sync, trading added complexity for near-zero downtime |
| **Strangler-fig pattern** | Incrementally routing specific functionality to a new service while a legacy system keeps serving everything else, until the legacy system's remaining surface shrinks to nothing |
| **Big-bang cutover** | Migrating or replacing an entire system in one event — the strangler-fig pattern exists specifically as the lower-risk alternative to this |

---

## Common Mistakes
- **Defaulting to Rehost for everything under deadline pressure**: it's the fastest strategy per-workload, but applying it uniformly (as this lab's opening instruction pushed for) ignores workloads like the custom CRM where Repurchase is a better multi-year outcome
- **Retiring a workload based on assumption instead of confirmed usage data**: "nobody uses this anymore" needs an access-log check, not a guess, before it becomes a decommission decision
- **Choosing online migration (DMS) by default "to be safe"**: it costs more in tooling complexity and a running billed instance — offline is the right, simpler answer whenever a maintenance window is actually acceptable
- **Attempting a full monolith refactor under a hard external deadline**: this is exactly the scenario strangler-fig solves — modernize incrementally after the deadline-driven migration lands, not as part of it
- **Skipping the Azure Migrate assessment phase and sizing target VMs by guesswork**: right-sizing recommendations exist specifically to avoid both over-provisioning (wasted cost) and under-provisioning (a performance regression discovered in production)

---

## Next Steps
This lab's strangler-fig deployment reuses the Front Door path-based routing mechanics from [Lab 6](lab-6-multi-region-global-distribution-design.md), applied to per-path service routing instead of per-region failover. Continue to [Lab 8: Capstone — Landing Zone Solution Design](lab-8-capstone-landing-zone-solution-design.md), which pulls this lab's migration-strategy thinking together with identity, data, network, and compute design into one synthesis. For the App Service hosting decision this lab's legacy/modern apps assume, see [Lab 4: Compute & Application Infrastructure Design](lab-4-compute-app-infrastructure-design.md).
