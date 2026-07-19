# Lab 6: Multi-Region & Global Distribution Design

Check box if done: [ ]

## Overview
AZ-305 doesn't test whether you can create a Front Door profile — it tests whether you can look at a global-traffic requirement and pick the right routing layer, and separately, whether you can look at a data requirement and pick the right Cosmos DB write model and consistency level without defaulting to "multi-region write, strong consistency" because it sounds the safest. This lab builds both decisions, then deploys a Front Door-fronted, two-region App Service backend and a multi-region Cosmos DB account to prove the design actually fails over the way the decision predicted it would.

**Estimated time**: 75–90 minutes
**Cost**: ~$1–$4 (two Basic B1 App Service Plans across two regions, Azure Front Door Standard's small hourly base charge plus negligible request/data-transfer volume, and a serverless Cosmos DB account billed per-request — nothing here is a large hourly meter like Lab 5's Azure Firewall, but nothing is fully free either; delete everything the same session)

---

## Scenario
You're the architect for a customer-facing web application that just landed its first large customers outside North America. Leadership has two requirements: users in Europe and Asia should not be routed through a North American data center for every request, and a full regional outage in the primary Azure region must not take the whole application down. The security team has a third requirement layered on top: whatever routes global traffic also needs to be able to enforce a WAF policy at the edge, before traffic ever reaches a backend. You need to pick the right global routing service, the right Cosmos DB write topology for the app's session/profile data, and the right consistency level — then prove the design survives a simulated regional failure.

---

## Objectives
- Build a global traffic routing decision matrix (Azure Front Door vs. Traffic Manager vs. Application Gateway) and justify Front Door for this scenario
- Decide between Cosmos DB single-region write and multi-region write models, and select a consistency level against stated latency/availability needs
- Weigh active-active against active-passive architecture trade-offs in terms of idle-capacity cost versus conflict-resolution complexity
- Deploy App Service backends in two regions behind a Front Door origin group with health probes
- Deploy a multi-region Cosmos DB account with the chosen consistency level
- Prove failover actually works by taking the primary region's backend offline and observing traffic shift

---

## Part 1: Design Decision — Global Routing, Write Topology, and Consistency

### Decision 1: Azure Front Door vs. Traffic Manager vs. Application Gateway

| Factor | Azure Front Door | Traffic Manager | Application Gateway |
|---|---|---|---|
| **Routing layer** | L7 (HTTP/HTTPS), terminated at Microsoft's global edge (anycast) | DNS-based (L4) — resolves a hostname to an IP, doesn't proxy traffic itself | L7, but **regional** — sits in front of backends within one region/VNet |
| **Latency-based routing** | Real edge-based routing to the lowest-latency healthy origin, evaluated per request | Latency-based DNS responses exist, but the client caches the resolved IP for the record's TTL — routing only re-evaluates on the next DNS lookup | Not applicable — it isn't a multi-region routing tool |
| **Failover speed** | Seconds — health probes run at the edge and an unhealthy origin is pulled from rotation on the next probe cycle | Slower and less deterministic — bound by DNS TTL and how aggressively clients and resolvers actually honor it, often minutes | N/A — no cross-region failover concept |
| **WAF integration** | Native at the edge (Premium SKU) — malicious traffic is blocked before it reaches any Azure region | None — Traffic Manager never sees the traffic itself, only resolves a name, so there's no L7 payload to inspect | Native, but only protects the single region that Application Gateway instance sits in front of |
| **Protocol support** | HTTP/HTTPS only | Any TCP-based protocol — not limited to HTTP | HTTP/HTTPS only |
| **CDN/caching** | Built in | None | None |
| **Best fit** | Global HTTP(S) apps needing latency-aware routing, edge WAF, and fast failover in one service | Non-HTTP global services (game servers, custom TCP protocols), or a fallback for workloads Front Door can't front | A single region's app needing L7 load balancing, path-based routing, or WAF within that region |

### Decision 2: Cosmos DB Single-Region Write vs. Multi-Region Write

| Factor | Single-Region Write (with multi-region reads) | Multi-Region Write (active-active) |
|---|---|---|
| **Write latency** | All writes go to one region — clients far from it pay a round trip to get there | Writes land in whichever region the client is closest to — low local write latency everywhere |
| **Conflict resolution** | Not needed — one writer, no concurrent conflicting writes possible | Required — last-writer-wins (default, by `_ts`) or a custom conflict-resolution stored procedure must be designed and tested |
| **Availability for writes during a regional outage** | Requires a failover (automatic or manual) promoting a read region to the write region — a brief write-unavailability window during that promotion | No failover needed for writes — the surviving regions keep accepting writes without any promotion event |
| **Application complexity** | Lower — standard single-writer mental model | Higher — the app must tolerate eventual convergence and understand the conflict-resolution policy in play |
| **Cost** | Lower — write-capacity RU/s provisioned once | Higher — write-capacity RU/s billed in *every* write-enabled region |
| **Best fit** | The large majority of apps — global reads are common, a centralized writer with a fast automatic failover plan is acceptable | Genuinely global, write-heavy, latency-sensitive workloads (collaborative editing, multi-region IoT ingestion) where write latency and write availability in every region outweigh the added complexity |

### Decision 3: Consistency Level Selection

| Level | Guarantee | Latency/Availability Cost | Works With Multi-Region Write? |
|---|---|---|---|
| **Strong** | Reads always return the most recent committed write, globally | Highest write latency — every write must sync to a quorum before acknowledging | **No** — Strong consistency is only available with single-region write; Cosmos DB doesn't offer it once multiple write regions are enabled |
| **Bounded Staleness** | Reads lag writes by at most K versions or T time — a bounded, configurable window | Lower latency than Strong, still gives a predictable staleness bound | Yes |
| **Session** (default) | Within one client's session, reads always see that session's own writes | Low latency, and the most common real-world default — "you always see your own writes" covers most UX needs | Yes |
| **Consistent Prefix** | Reads never see writes out of order, but may be stale | Lower latency than Session in high-throughput scenarios | Yes |
| **Eventual** | No ordering guarantee at all — reads eventually converge | Lowest latency, highest throughput | Yes |

The exam-relevant trap here: reaching for "Strong consistency" as the default-safe answer silently rules out multi-region write, since the two are mutually exclusive in Cosmos DB. If the workload needs multi-region write, the consistency ceiling is Bounded Staleness — not Strong.

### Decision 4: Active-Active vs. Active-Passive Architecture

| Factor | Active-Passive (primary serves, secondary is standby) | Active-Active (all regions serve live traffic) |
|---|---|---|
| **Idle capacity cost** | The standby region's compute (App Service Plan instances, etc.) either sits idle and billed, or is scaled to zero and takes time to warm up when needed | No idle capacity — every region does real work, so spend maps directly to load served |
| **Conflict resolution burden** | None at the app tier — one region writes at a time | Present at the data tier if writes are also multi-region (Decision 2) — the app has to be conflict-aware |
| **Operational simplicity** | Simpler mental model — "the primary is down, promote the secondary" | More moving parts — routing, data replication, and conflict handling all have to agree on the current state simultaneously |
| **Best fit** | DR-driven requirements where a short failover window is acceptable and cost predictability matters more than zero idle spend | Latency-driven requirements (users physically distributed) where every region needs to be genuinely useful all the time, not just insurance |

### Recommendation for This Scenario
**Azure Front Door Standard** for global HTTP routing — it's the only option of the three that satisfies latency-aware routing, edge WAF, and fast automated failover in one service; Traffic Manager's DNS-TTL-bound failover is too slow for this scenario's "a regional outage must not take the app down" requirement, and Application Gateway alone has no cross-region concept at all.

For Cosmos DB: **single-region write with multi-region reads**, **Session consistency**, and **automatic failover** enabled. The scenario's stated requirements are read-latency (serve EU/Asia users locally) and regional resilience — neither requires multi-region write's added conflict-resolution complexity and doubled write-capacity cost. If a future requirement demands sub-100ms writes from every region (not stated here), that's the trigger to revisit multi-region write and drop to Bounded Staleness.

For the overall architecture: **active-passive at the compute tier, with Front Door's priority-based origin routing driving the failover**, since the requirement is resilience, not the added cost of two fully-live compute regions. Front Door's origin groups support both models with the same mechanism — origins at the *same* priority get latency-based active-active routing between them; a *lower*-priority origin only receives traffic if every higher-priority origin fails its health probe. Part 2 deploys the active-passive shape; the note in Part 4 shows the one-line change to make it active-active.

---

## Part 2: Deploy App Service Backends in Two Regions

### Step 1: Set Variables and Create the Resource Group
```bash
RG="rg-az305-lab6"
PRIMARY_LOCATION="eastus"
SECONDARY_LOCATION="westus"
SUFFIX="<your-unique-suffix>"

az group create --name $RG --location $PRIMARY_LOCATION
```
`eastus` / `westus` is the same Azure-paired region pair used for Lab 3's ASR target — reusing a familiar pairing here isn't required, but it's the same reasoning: paired regions get staggered platform maintenance, so both regions are never patched at the same time.

### Step 2: Create a Small Node App That Reports Its Own Region
```bash
mkdir lab6-app && cd lab6-app

cat > server.js <<'EOF'
const http = require('http');
const region = process.env.REGION_NAME || 'unknown';
http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end(`Serving from: ${region}\n`);
}).listen(process.env.PORT || 8080);
EOF

cat > package.json <<'EOF'
{ "name": "lab6-app", "version": "1.0.0", "main": "server.js", "scripts": { "start": "node server.js" } }
EOF

zip -r app.zip server.js package.json
cd ..
```
A response that names its own region is what makes the failover test in Part 5 provable, instead of just "it still returned 200."

### Step 3: Deploy the Primary (East US) App
```bash
az appservice plan create --name asp-lab6-primary --resource-group $RG \
  --location $PRIMARY_LOCATION --sku B1 --is-linux

az webapp create --name app-lab6-primary-$SUFFIX --resource-group $RG \
  --plan asp-lab6-primary --runtime "NODE:20-lts"

az webapp config appsettings set --name app-lab6-primary-$SUFFIX --resource-group $RG \
  --settings REGION_NAME="East US (primary)"

az webapp deploy --name app-lab6-primary-$SUFFIX --resource-group $RG \
  --src-path lab6-app/app.zip --type zip
```

### Step 4: Deploy the Secondary (West US) App
```bash
az appservice plan create --name asp-lab6-secondary --resource-group $RG \
  --location $SECONDARY_LOCATION --sku B1 --is-linux

az webapp create --name app-lab6-secondary-$SUFFIX --resource-group $RG \
  --plan asp-lab6-secondary --runtime "NODE:20-lts"

az webapp config appsettings set --name app-lab6-secondary-$SUFFIX --resource-group $RG \
  --settings REGION_NAME="West US (secondary)"

az webapp deploy --name app-lab6-secondary-$SUFFIX --resource-group $RG \
  --src-path lab6-app/app.zip --type zip
```

**Validation checkpoint**: `curl https://app-lab6-primary-$SUFFIX.azurewebsites.net` and `curl https://app-lab6-secondary-$SUFFIX.azurewebsites.net` should each return their own region's text directly — confirm both before moving on, since Front Door will only ever be as correct as the origins behind it.

---

## Part 3: Deploy Front Door with a Priority-Based Origin Group

### Step 5: Create the Front Door Profile and Endpoint
```bash
az afd profile create --profile-name afd-lab6-global --resource-group $RG \
  --sku Standard_AzureFrontDoor

az afd endpoint create --resource-group $RG --profile-name afd-lab6-global \
  --endpoint-name lab6-global-$SUFFIX --enabled-state Enabled
```

### Step 6: Create the Origin Group with Health Probes
```bash
az afd origin-group create --resource-group $RG --profile-name afd-lab6-global \
  --origin-group-name webapp-origins \
  --probe-request-type GET --probe-protocol Https --probe-path / \
  --probe-interval-in-seconds 30 \
  --sample-size 4 --successful-samples-required 3 --additional-latency-in-milliseconds 50
```
`--sample-size 4 --successful-samples-required 3` means Front Door needs 3 of the last 4 probes to succeed before treating an origin as healthy again — this is what makes failover deliberate rather than flapping on a single dropped probe.

### Step 7: Add Both Origins at Different Priorities (Active-Passive)
```bash
az afd origin create --resource-group $RG --profile-name afd-lab6-global \
  --origin-group-name webapp-origins --origin-name primary-eastus \
  --host-name app-lab6-primary-$SUFFIX.azurewebsites.net \
  --origin-host-header app-lab6-primary-$SUFFIX.azurewebsites.net \
  --priority 1 --weight 1000 --http-port 80 --https-port 443 --enabled-state Enabled

az afd origin create --resource-group $RG --profile-name afd-lab6-global \
  --origin-group-name webapp-origins --origin-name secondary-westus \
  --host-name app-lab6-secondary-$SUFFIX.azurewebsites.net \
  --origin-host-header app-lab6-secondary-$SUFFIX.azurewebsites.net \
  --priority 2 --weight 1000 --http-port 80 --https-port 443 --enabled-state Enabled
```
Different priorities (1 vs. 2) is what makes this active-passive per Decision 4 — Front Door sends 100% of traffic to priority 1 and only touches priority 2 if every priority-1 origin fails its probes. Setting both to `--priority 1` instead would make this active-active, splitting traffic between regions by latency.

### Step 8: Create the Route
```bash
az afd route create --resource-group $RG --profile-name afd-lab6-global \
  --endpoint-name lab6-global-$SUFFIX --route-name default-route \
  --origin-group webapp-origins --supported-protocols Https Http \
  --link-to-default-domain Enabled --forwarding-protocol MatchRequest
```

### Step 9: Restrict the Origins to Front Door Traffic Only
Without this, both App Services are still reachable directly at their own `azurewebsites.net` hostnames — bypassing Front Door, and any WAF policy attached to it, entirely.
```bash
AFD_ID=$(az afd profile show --resource-group $RG --profile-name afd-lab6-global \
  --query "frontDoorId" -o tsv)

az webapp config access-restriction add --resource-group $RG --name app-lab6-primary-$SUFFIX \
  --rule-name AllowFrontDoorOnly --priority 100 \
  --service-tag AzureFrontDoor.Backend --http-header x-azure-fdid=$AFD_ID

az webapp config access-restriction add --resource-group $RG --name app-lab6-secondary-$SUFFIX \
  --rule-name AllowFrontDoorOnly --priority 100 \
  --service-tag AzureFrontDoor.Backend --http-header x-azure-fdid=$AFD_ID
```
The `x-azure-fdid` header check matters as much as the service tag — the service tag alone only proves traffic came from *some* Front Door profile, not specifically yours; anyone else's Front Door instance could otherwise reach your origin.

**Validation checkpoint**:
```bash
curl https://lab6-global-$SUFFIX.z01.azurefd.net
```
Expect `Serving from: East US (primary)`. Front Door takes a few minutes to propagate after route creation — retry if the first request 404s.

---

## Part 4: Deploy Multi-Region Cosmos DB

### Step 10: Create the Account with Single-Region Write and a Read Region
```bash
az cosmosdb create --name cosmos-lab6-$SUFFIX --resource-group $RG \
  --default-consistency-level Session \
  --locations regionName=$PRIMARY_LOCATION failoverPriority=0 isZoneRedundant=false \
  --locations regionName=$SECONDARY_LOCATION failoverPriority=1 isZoneRedundant=false \
  --enable-automatic-failover true \
  --capacity-mode Serverless
```
This is Decision 2's recommendation deployed literally — no `--enable-multiple-write-locations` flag, so `$PRIMARY_LOCATION` is the only write region and `$SECONDARY_LOCATION` is read-only until a failover promotes it. Serverless capacity mode keeps this near-free for a lab with negligible request volume; a production account matching this design would size provisioned RU/s instead.

**Validation checkpoint**:
```bash
az cosmosdb show --name cosmos-lab6-$SUFFIX --resource-group $RG \
  --query "{consistency:consistencyPolicy.defaultConsistencyLevel, writeLocations:writeLocations[].locationName, readLocations:readLocations[].locationName}" -o jsonc
```
Confirm one write location (`East US`) and two read locations.

### Step 11: Trigger a Manual Regional Failover Test
```bash
az cosmosdb failover-priority-change --name cosmos-lab6-$SUFFIX --resource-group $RG \
  --failover-policies "$SECONDARY_LOCATION=0" "$PRIMARY_LOCATION=1"
```
This promotes West US to the write region and demotes East US to read-only — the same kind of controlled failover test Lab 3's ASR test-failover performs for compute, but here it completes in well under a minute rather than tens of minutes, because there's no VM boot or disk attach involved, only a metadata-level write-region reassignment.

**Validation checkpoint**: re-run the Step 10 query — `writeLocations` should now show `West US` first. Fail back before cleanup:
```bash
az cosmosdb failover-priority-change --name cosmos-lab6-$SUFFIX --resource-group $RG \
  --failover-policies "$PRIMARY_LOCATION=0" "$SECONDARY_LOCATION=1"
```

---

## Part 5: Test Front Door Failover

### Step 12: Stop the Primary App and Observe the Failover
```bash
az webapp stop --name app-lab6-primary-$SUFFIX --resource-group $RG
```
Wait roughly 60–90 seconds for the origin-group health probe (30-second interval, 3-of-4 samples required from Step 6) to mark `primary-eastus` unhealthy, then retry:
```bash
curl https://lab6-global-$SUFFIX.z01.azurefd.net
```
**Expected result**: the response changes to `Serving from: West US (secondary)` with no client-side reconfiguration — the same URL, the same DNS record, just a different origin behind it. This is the concrete difference Decision 1 predicted between Front Door's edge-probed failover and Traffic Manager's DNS-TTL-bound failover.

### Step 13: Restart the Primary
```bash
az webapp start --name app-lab6-primary-$SUFFIX --resource-group $RG
```
Wait for the next successful probe cycle, then confirm traffic returns to `East US (primary)` — priority 1 is preferred again the moment it's healthy.

---

## Cleanup
```bash
az afd route delete --resource-group $RG --profile-name afd-lab6-global \
  --endpoint-name lab6-global-$SUFFIX --route-name default-route --yes
az afd origin-group delete --resource-group $RG --profile-name afd-lab6-global \
  --origin-group-name webapp-origins --yes
az afd endpoint delete --resource-group $RG --profile-name afd-lab6-global \
  --endpoint-name lab6-global-$SUFFIX --yes
az afd profile delete --resource-group $RG --profile-name afd-lab6-global --yes

az cosmosdb delete --name cosmos-lab6-$SUFFIX --resource-group $RG --yes

az group delete --name $RG --yes --no-wait
```
Confirm with `az group show --name $RG` returning a `ResourceGroupNotFound` error once deletion completes.

---

## Key Concepts

| Term | Definition |
|---|---|
| **Origin group priority vs. weight** | Priority determines failover order (lower number preferred, only used if all higher priorities are unhealthy); weight distributes traffic between origins that share the *same* priority |
| **Anycast edge routing** | Front Door's global points of presence advertise the same IP everywhere — a client's request naturally routes to the nearest healthy edge node without any DNS-level decision |
| **Multi-region write (multi-master)** | A Cosmos DB mode where more than one region accepts writes concurrently, trading conflict-resolution complexity for local write latency and write availability everywhere |
| **Bounded Staleness vs. Session consistency** | Bounded Staleness gives a globally consistent, provable staleness window; Session gives per-client read-your-writes guarantees with lower latency — Session is the more common real-world default |
| **Automatic failover (Cosmos DB)** | A policy that promotes the next-priority region to writable automatically if the current write region becomes unavailable, without a manual `failover-priority-change` call |
| **Active-active vs. active-passive** | Active-active serves live traffic from every region simultaneously (no idle spend, but requires conflict-aware design if data also writes multi-region); active-passive keeps a standby ready to take over (simpler, but pays for idle or slower-to-warm capacity) |

---

## Common Mistakes
- **Choosing Strong consistency and multi-region write together**: Cosmos DB doesn't allow it — Strong consistency requires single-region write; picking Strong first and multi-region write second (or vice versa) as independent decisions is how this gets missed
- **Leaving origins publicly reachable at their own hostname**: without the access-restriction rule from Step 9, anyone can bypass Front Door (and any WAF policy on it) by hitting `app-lab6-primary-<suffix>.azurewebsites.net` directly
- **Assuming Traffic Manager gives the same failover speed as Front Door**: it's DNS-based — failover speed is bound by TTL and resolver/client caching behavior, not a probe-driven edge decision
- **Setting both origins to the same priority when active-passive was the actual requirement**: that silently turns the design into active-active, which changes both the cost profile and (if writes are also multi-region) the conflict-resolution requirement
- **Forgetting to fail Cosmos DB back to the primary region after testing**: leaves the write region pinned to the secondary indefinitely, which is easy to miss since the application keeps working either way

---

## Next Steps
This lab extends [Lab 5](lab-5-network-infrastructure-design.md)'s Application Gateway/Front Door distinction into an actual global deployment, and reuses [Lab 4](lab-4-compute-app-infrastructure-design.md)'s App Service hosting decision as the backend this lab fronts globally. Continue to [Lab 7: Migration & Modernization Design](lab-7-migration-modernization-design.md) for the next AZ-305 design domain, or to [Lab 8: Capstone — Landing Zone Solution Design](lab-8-capstone-landing-zone-solution-design.md) to see how global distribution fits into a full landing zone design alongside every other AZ-305 lab.
