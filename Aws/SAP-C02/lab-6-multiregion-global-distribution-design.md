# Lab 6: Multi-Region & Global Distribution Design

Check box if done: [ ]

## Overview
Lab 3 built regional disaster recovery — a secondary region that comes online when the primary fails. This lab asks the next question: once a second region genuinely exists, how does traffic actually find its way there, and how does a multi-region *data* layer stay consistent while it does. Route 53 offers four meaningfully different routing policies for this, and DynamoDB Global Tables and Aurora Global Database solve multi-region data very differently — this lab decides both, then builds a CloudFront + Route 53 multi-region distribution and forces an actual failover to measure how fast traffic actually moves.

**Estimated time**: 90–110 minutes
**Cost**: ~$1–$4 (two small S3-hosted origins, a CloudFront distribution, and Route 53 health checks — nothing here is a large hourly meter, but CloudFront and health checks do bill in small amounts for the lab's duration)

---

## Scenario
Your company's application currently serves 100% of its traffic from a single region and has just had its first real regional outage — forty minutes of downtime that made the incident report. Leadership wants two things: users should be served from whichever region gives them the best latency under normal conditions, and if a region goes down, traffic should redirect automatically without a human paging in at 2 a.m. to change a DNS record by hand. Separately, the product team wants a new feature — per-user preferences — writable from any region with reads that are fast everywhere, while the existing order-history reporting workload stays relational and only needs to be read (not written) from the secondary region.

---

## Objectives
- Compare Route 53 routing policies (latency, weighted, failover, geolocation) and justify the one(s) that fit this scenario
- Compare DynamoDB Global Tables and Aurora Global Database for two different multi-region data requirements in the same company
- Deploy origins in two regions and front them with a Route 53 failover routing policy backed by health checks
- Deploy a CloudFront distribution with an origin group for edge-level failover, distinct from Route 53's DNS-level failover
- Force an actual failover and measure how long it takes traffic to move, at both the DNS layer and the CDN layer

---

## Part 1: Design Decision — Routing Policy and Multi-Region Data Model

### Decision 1: Route 53 Routing Policy Comparison

| Policy | What it optimizes for | Failure handling | Best fit |
|---|---|---|---|
| **Simple** | Nothing — one record, one (or a static set of) answer(s) | None — no health check awareness | Never appropriate once more than one region exists |
| **Weighted** | Traffic split by an assigned proportion (e.g., 90/10 for a canary release) | Only if combined with health checks per record | Gradual traffic shifting (canary deploys, migration cutover) — not primarily a resilience mechanism |
| **Latency-based** | Routes each requester to the region with the lowest measured latency for them specifically | Removes a region from consideration if its health check fails, falling back to the next-lowest-latency healthy region | This scenario's "best latency under normal conditions" requirement, directly |
| **Failover** | Routes 100% of traffic to a primary record; only serves the secondary if the primary's health check fails | This **is** the failure-handling mechanism — its entire purpose | This scenario's "redirect automatically on regional outage" requirement, directly |
| **Geolocation** | Routes based on the requester's geographic location (country/continent), regardless of latency | Requires an explicit default record; no automatic health-based fallback unless paired with health checks per record | Compliance/data-residency requirements (EU users must hit an EU region) — not what this scenario asks for |

### Step 1: Why This Scenario Needs Two Policies, Not One
Latency-based and Failover routing solve different halves of the stated requirement, and neither alone satisfies both. **Latency-based routing** handles "best latency under normal conditions" — but a naive latency policy with no health check still happily routes users to a dead region if nothing is monitoring health. **Failover routing** handles "redirect automatically on outage" — but a pure primary/secondary failover policy ignores latency entirely once both regions are healthy, sending 100% of normal traffic to one region even when a closer one is available. The standard resolution AWS itself documents: use **latency-based routing with health checks attached to every record**. This gets the latency-optimization behavior under normal conditions *and* the failover behavior during an outage, because a latency-based record with a failed health check is automatically excluded from consideration — functionally converging on failover behavior exactly when it's needed, without giving up latency-optimization the rest of the time.

### Decision 2: DynamoDB Global Tables vs. Aurora Global Database

| Factor | DynamoDB Global Tables | Aurora Global Database |
|---|---|---|
| **Write model** | Multi-active — every region can accept writes to the same table concurrently | Single-writer — one primary region accepts writes; secondary regions are read-only replicas |
| **Consistency model** | Eventually consistent across regions (typically sub-second propagation), with last-writer-wins conflict resolution | Strongly consistent within the primary region; secondary regions lag by typically under a second via dedicated replication infrastructure, but are not writable |
| **Fit for "writable from any region, fast reads everywhere"** | Exact match — this is the workload Global Tables was purpose-built for | Fails outright — the product requirement needs multi-region writes, which Aurora Global Database's architecture does not support |
| **Fit for "relational reporting, reads only from secondary"** | Wrong tool even if it could satisfy the write pattern — the workload needs joins and relational reporting queries, not key-value access | Exact match — single-writer is fine because the secondary region only ever needs to read, and the workload is inherently relational |
| **Failover to promote a secondary as a new writer** | Not applicable — every region already accepts writes, so there's no "promotion" step at all | Supported — a secondary region's cluster can be promoted to become the new primary/writer if the original primary region is lost |

### Recommendation for This Scenario
**DynamoDB Global Tables for the per-user preferences feature** — the stated requirement (writable from any region, fast reads everywhere) is exactly what multi-active replication is for, and Aurora Global Database's single-writer model cannot satisfy it at all, regardless of instance sizing. **Aurora Global Database for the order-history reporting workload** — it needs relational query capability DynamoDB doesn't provide, and its access pattern (read-only from the secondary region) is exactly Aurora Global Database's supported shape. This is a genuine "both services are correct, for different parts of the same company" answer — a common SAP-C02 pattern, and a reason to be suspicious of any single blanket "always use DynamoDB for multi-region" or "always use Aurora for multi-region" rule of thumb.

Parts 2–4 build the routing half of this design (latency+failover-backed Route 53, CloudFront origin failover); deploying both DynamoDB Global Tables and a multi-region Aurora cluster is out of scope for this lab's budget — the decision above, backed by the comparison table, is the artifact a design review actually asks for.

---

## Part 2: Deploy Origins in Two Regions

### Step 2: Create an S3 Static Website in Each Region
```bash
PRIMARY_REGION="us-east-1"
SECONDARY_REGION="us-west-2"
PRIMARY_BUCKET="sap-c02-lab6-primary-<your-unique-suffix>"
SECONDARY_BUCKET="sap-c02-lab6-secondary-<your-unique-suffix>"

for pair in "$PRIMARY_BUCKET:$PRIMARY_REGION:Primary (us-east-1)" "$SECONDARY_BUCKET:$SECONDARY_REGION:Secondary (us-west-2)"; do
  BUCKET=$(echo $pair | cut -d: -f1); REGION=$(echo $pair | cut -d: -f2); LABEL=$(echo $pair | cut -d: -f3)
  aws s3api create-bucket --bucket $BUCKET --region $REGION $( [ "$REGION" != "us-east-1" ] && echo "--create-bucket-configuration LocationConstraint=$REGION" )
  aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration \
    "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
  echo "<h1>$LABEL</h1>" > index.html
  aws s3 cp index.html s3://$BUCKET/index.html --region $REGION
  aws s3 website s3://$BUCKET --index-document index.html --region $REGION
  aws s3api put-bucket-policy --bucket $BUCKET --region $REGION --policy '{
    "Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":"*","Action":"s3:GetObject","Resource":"arn:aws:s3:::'$BUCKET'/*"}]
  }'
done
```

**Validation checkpoint**: `curl` each bucket's website endpoint (`http://$PRIMARY_BUCKET.s3-website-$PRIMARY_REGION.amazonaws.com`) and confirm both return their distinct label text.

---

## Part 3: Route 53 — Latency Routing With Health-Check-Driven Failover

### Step 3: Create Health Checks Against Both Origins
```bash
PRIMARY_HC=$(aws route53 create-health-check --caller-reference lab6-primary-$(date +%s) --health-check-config \
  "IPAddress=,Type=HTTP,ResourcePath=/index.html,FullyQualifiedDomainName=$PRIMARY_BUCKET.s3-website-$PRIMARY_REGION.amazonaws.com,Port=80,RequestInterval=30,FailureThreshold=2" \
  --query "HealthCheck.Id" --output text)

SECONDARY_HC=$(aws route53 create-health-check --caller-reference lab6-secondary-$(date +%s) --health-check-config \
  "IPAddress=,Type=HTTP,ResourcePath=/index.html,FullyQualifiedDomainName=$SECONDARY_BUCKET.s3-website-$SECONDARY_REGION.amazonaws.com,Port=80,RequestInterval=30,FailureThreshold=2" \
  --query "HealthCheck.Id" --output text)
```
`FailureThreshold=2` at a 30-second interval means Route 53 needs roughly one minute of consecutive failures before marking an endpoint unhealthy — this is the floor on how fast DNS-level failover can react, and it's a number worth stating explicitly in a design review rather than assuming failover is instantaneous.

### Step 4: Create Latency-Based Records, Each Tied to Its Region's Health Check
```bash
HOSTED_ZONE_ID="<your-hosted-zone-id>"

aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.<your-domain>", "Type": "CNAME", "SetIdentifier": "primary-latency",
      "Region": "'"$PRIMARY_REGION"'", "TTL": 30,
      "ResourceRecords": [{"Value": "'"$PRIMARY_BUCKET"'.s3-website-'"$PRIMARY_REGION"'.amazonaws.com"}],
      "HealthCheckId": "'"$PRIMARY_HC"'"
    }
  }]
}'

aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.<your-domain>", "Type": "CNAME", "SetIdentifier": "secondary-latency",
      "Region": "'"$SECONDARY_REGION"'", "TTL": 30,
      "ResourceRecords": [{"Value": "'"$SECONDARY_BUCKET"'.s3-website-'"$SECONDARY_REGION"'.amazonaws.com"}],
      "HealthCheckId": "'"$SECONDARY_HC"'"
    }
  }]
}'
```
A short `TTL` (30 seconds) is deliberate for this lab — it puts a floor on how quickly resolvers pick up the failover, which is exactly what gets measured in Part 5. A production record can use a longer TTL once the failover-speed/caching-efficiency tradeoff has been consciously made.

**Validation checkpoint**:
```bash
aws route53 get-health-check-status --health-check-id $PRIMARY_HC --query "HealthCheckObservations[0].StatusReport.Status"
```
Confirm both health checks report success (`Success: HTTP Status Code 200`) before moving on — Part 5's timed test needs a known-healthy starting state.

---

## Part 4: CloudFront — Edge-Level Origin Failover

Route 53's failover is DNS-based, bounded by resolver caching and TTL. CloudFront's origin group failover happens at the CDN edge, per-request, with no DNS propagation delay at all — a meaningfully different (and typically faster) failover mechanism worth layering in front of the DNS-level one.

### Step 5: Create an Origin Group With Primary and Secondary S3 Origins
```bash
DISTRIBUTION_CONFIG=$(cat <<EOF
{
  "CallerReference": "sap-c02-lab6-$(date +%s)",
  "Comment": "sap-c02-lab6-multiregion",
  "Enabled": true,
  "Origins": {
    "Quantity": 2,
    "Items": [
      {"Id": "primary-origin", "DomainName": "$PRIMARY_BUCKET.s3-website-$PRIMARY_REGION.amazonaws.com", "CustomOriginConfig": {"HTTPPort": 80, "HTTPSPort": 443, "OriginProtocolPolicy": "http-only"}},
      {"Id": "secondary-origin", "DomainName": "$SECONDARY_BUCKET.s3-website-$SECONDARY_REGION.amazonaws.com", "CustomOriginConfig": {"HTTPPort": 80, "HTTPSPort": 443, "OriginProtocolPolicy": "http-only"}}
    ]
  },
  "OriginGroups": {
    "Quantity": 1,
    "Items": [{
      "Id": "primary-secondary-group",
      "FailoverCriteria": {"StatusCodes": {"Quantity": 3, "Items": [403, 404, 500]}},
      "Members": {"Quantity": 2, "Items": [{"OriginId": "primary-origin"}, {"OriginId": "secondary-origin"}]}
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "primary-secondary-group",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
  }
}
EOF
)
echo "$DISTRIBUTION_CONFIG" > dist-config.json
DIST_ID=$(aws cloudfront create-distribution --distribution-config file://dist-config.json --query "Distribution.Id" --output text)
```
`FailoverCriteria` listing `403, 404, 500` means CloudFront treats any of those response codes from the primary origin as a trigger to retry the *same request* against the secondary origin — immediately, within that request's lifecycle, not on the next DNS lookup.

**Validation checkpoint**:
```bash
aws cloudfront get-distribution --id $DIST_ID --query "Distribution.Status"
```
Wait for `Deployed` (can take several minutes), then `curl` the distribution's domain name and confirm it returns the primary region's content.

---

## Part 5: Force a Failover and Measure Both Mechanisms

### Step 6: Break the Primary Origin
```bash
START_TIME=$(date +%s)
aws s3api put-bucket-policy --bucket $PRIMARY_BUCKET --region $PRIMARY_REGION --policy '{
  "Version":"2012-10-17","Statement":[{"Effect":"Deny","Principal":"*","Action":"s3:GetObject","Resource":"arn:aws:s3:::'$PRIMARY_BUCKET'/*"}]
}'
```
Denying public read access simulates a regional failure from the client's perspective — the primary origin now returns `403` for every request, which is both what Route 53's health check will fail on and what CloudFront's `FailoverCriteria` is configured to catch.

### Step 7: Measure CloudFront's Failover (Edge-Level, Per-Request)
```bash
while true; do
  RESULT=$(curl -s -o /dev/null -w "%{http_code}" https://$(aws cloudfront get-distribution --id $DIST_ID --query "Distribution.DomainName" --output text)/index.html)
  if [ "$RESULT" == "200" ]; then
    echo "CloudFront serving secondary content after $(( $(date +%s) - START_TIME )) seconds"
    break
  fi
  sleep 2
done
```
Expect this to resolve within seconds to low tens of seconds — no DNS involved, just CloudFront's per-request retry against the origin group's second member.

### Step 8: Measure Route 53's Failover (DNS-Level, Bounded by Health Check Interval + TTL)
```bash
while true; do
  STATUS=$(aws route53 get-health-check-status --health-check-id $PRIMARY_HC --query "HealthCheckObservations[0].StatusReport.Status" --output text)
  if [[ "$STATUS" == *"Failure"* ]]; then
    echo "Route 53 health check marked primary unhealthy at $(( $(date +%s) - START_TIME )) seconds"
    break
  fi
  sleep 5
done
# Then confirm resolvers actually pick up the change:
dig app.<your-domain> +short
```

**Validation checkpoint**: compare the two measured times directly. CloudFront's edge failover should measurably beat Route 53's DNS failover — the concrete proof for the design decision that a CDN with origin-group failover is a meaningfully faster failure-recovery layer than DNS failover alone, and why production architectures commonly deploy both rather than treating them as redundant.

---

## Cleanup

```bash
# 1. Restore/remove the primary bucket policy, then delete both buckets
aws s3 rm s3://$PRIMARY_BUCKET --recursive --region $PRIMARY_REGION
aws s3api delete-bucket --bucket $PRIMARY_BUCKET --region $PRIMARY_REGION
aws s3 rm s3://$SECONDARY_BUCKET --recursive --region $SECONDARY_REGION
aws s3api delete-bucket --bucket $SECONDARY_BUCKET --region $SECONDARY_REGION

# 2. CloudFront distribution — must be disabled before it can be deleted
aws cloudfront get-distribution-config --id $DIST_ID > current-config.json
ETAG=$(jq -r .ETag current-config.json)
jq '.DistributionConfig.Enabled = false' current-config.json | jq .DistributionConfig > disabled-config.json
aws cloudfront update-distribution --id $DIST_ID --distribution-config file://disabled-config.json --if-match $ETAG
# Wait for Status: Deployed after disabling before deleting
aws cloudfront delete-distribution --id $DIST_ID --if-match $(aws cloudfront get-distribution --id $DIST_ID --query "ETag" --output text)

# 3. Route 53 records and health checks
aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch '{"Changes":[
  {"Action":"DELETE","ResourceRecordSet":{"Name":"app.<your-domain>","Type":"CNAME","SetIdentifier":"primary-latency","Region":"'"$PRIMARY_REGION"'","TTL":30,"ResourceRecords":[{"Value":"'"$PRIMARY_BUCKET"'.s3-website-'"$PRIMARY_REGION"'.amazonaws.com"}],"HealthCheckId":"'"$PRIMARY_HC"'"}},
  {"Action":"DELETE","ResourceRecordSet":{"Name":"app.<your-domain>","Type":"CNAME","SetIdentifier":"secondary-latency","Region":"'"$SECONDARY_REGION"'","TTL":30,"ResourceRecords":[{"Value":"'"$SECONDARY_BUCKET"'.s3-website-'"$SECONDARY_REGION"'.amazonaws.com"}],"HealthCheckId":"'"$SECONDARY_HC"'"}}
]}'
aws route53 delete-health-check --health-check-id $PRIMARY_HC
aws route53 delete-health-check --health-check-id $SECONDARY_HC
```
CloudFront distribution deletion is asynchronous and only proceeds once `Enabled: false` has fully propagated (`Status: Deployed` after the disable update) — don't skip the wait, the delete call fails otherwise.

Confirm with `aws cloudfront list-distributions --query "DistributionList.Items[?Comment=='sap-c02-lab6-multiregion']"` and `aws route53 list-health-checks --query "HealthChecks[?CallerReference | contains(@, 'lab6')]"` — both should return empty.

---

## Key Concepts

| Term | Definition |
|---|---|
| **Latency-based routing** | Routes each DNS query to the record with the lowest measured latency for that specific resolver, automatically excluding any record whose health check has failed |
| **Failover routing** | Routes all traffic to a primary record; serves the secondary only when the primary's associated health check reports unhealthy |
| **Route 53 health check** | An active monitor polling an endpoint at a configured interval/threshold; its status directly gates which records latency and failover routing policies consider eligible |
| **CloudFront origin group** | Two origins (primary + secondary) with defined failover status codes; CloudFront retries a failed request against the secondary origin at the edge, within the same request, no DNS involved |
| **DynamoDB Global Tables** | Multi-active, multi-region DynamoDB replication — every replica table accepts writes, propagated to other regions with eventual consistency |
| **Aurora Global Database** | Single-writer, multi-region Aurora replication — one primary region accepts writes, secondary regions serve low-lag reads and can be promoted to primary on failure |
| **TTL (Time to Live)** | How long a DNS resolver caches a record before re-querying — directly bounds how fast a DNS-level routing/failover change is observed by clients |

---

## Common Mistakes
- **Using Failover routing alone and ignoring latency under normal conditions**: this sends 100% of healthy-state traffic to one region regardless of where the requester actually is, defeating half the stated requirement
- **Using Latency routing alone without health checks attached**: a "best latency" record with no health awareness happily routes users straight into a dead region — the health check is what turns latency routing into a resilience mechanism, not an optional add-on
- **Assuming DNS failover is instant**: it's bounded by health check interval × failure threshold, plus however long resolvers hold the previous answer per TTL — a real, measurable number, not zero
- **Treating CloudFront origin failover and Route 53 failover as redundant, so only building one**: they fail over at different layers on different timelines — CloudFront's edge-level failover is typically faster, but only covers content actually served through that distribution
- **Assuming DynamoDB Global Tables' eventual consistency is safe for every use case**: last-writer-wins conflict resolution across concurrently-written regions can silently drop a write — fine for user preferences, not fine for anything requiring strict ordering guarantees

---

## Next Steps
This lab builds on the RTO/RPO-driven DR reasoning from [Lab 3: Business Continuity Design](lab-3-business-continuity-design.md) — that lab's Warm Standby recommendation assumed a routing mechanism to actually redirect traffic during failover, which is exactly what this lab built. It assumes comfort with basic Route 53 and CloudFront concepts from [SAA-C03 Labs 1–4](../SAA-C03/README.md). Continue to [Lab 7: Migration & Modernization Design](lab-7-migration-modernization-design.md) for the workload-assessment domain this track hasn't covered yet.
