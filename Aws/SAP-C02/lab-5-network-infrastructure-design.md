# Lab 5: Network Infrastructure Design

Check box if done: [ ]

## Overview
SAA-C03 covers a single VPC's subnets, route tables, and security groups. SAP-C02 asks the question that comes after: once there are five VPCs, or fifty, how do they talk to each other, to on-premises, and to the internet without every connection being a bespoke peering relationship and every perimeter control reinvented per VPC. This lab decides the topology and the perimeter security model first, then builds a Transit Gateway hub-spoke network, private service connectivity via VPC endpoints, and a WAF-protected edge — with an explicit, upfront warning that Transit Gateway is the single most expensive resource in this entire track.

**Estimated time**: 90–120 minutes
**Cost**: ~$3–$8 — **Transit Gateway bills hourly per attachment (~$0.05/hr per attachment) plus ~$0.02/GB processed, regardless of traffic volume. With two spoke attachments running for the lab's duration, this is a real, ongoing meter the moment it's created. Deploy it, complete Parts 2–4, and delete it the same session — see Cleanup.**

---

## Scenario
You're the architect standing up the network foundation for a landing zone that will eventually hold a dozen or more workload VPCs. The design can't be a flat set of ad-hoc VPC peering connections — that model breaks down mathematically well before a dozen VPCs. Security requires every workload VPC's traffic to a shared service (in this lab, a central "shared services" VPC) to route through a single, auditable hub, and every internet-facing service must sit behind a WAF, not directly exposed on an ALB with no layer-7 filtering at all. A future phase will connect an on-premises data center, and you need to have already reasoned through Direct Connect vs. Site-to-Site VPN before that requirement becomes urgent.

---

## Objectives
- Decide between VPC peering and Transit Gateway hub-spoke topology and justify the choice for a growing multi-VPC estate
- Decide between AWS Direct Connect and Site-to-Site VPN for future on-premises connectivity
- Deploy a Transit Gateway with two spoke VPCs attached and route tables enforcing hub-routed traffic
- Deploy a VPC interface endpoint (PrivateLink) so a spoke VPC reaches an AWS service without traversing the internet
- Deploy an AWS WAF Web ACL in front of an Application Load Balancer and understand where Shield and CloudFront fit relative to it
- Prove Transit Gateway routing is real by testing connectivity between spokes through the hub

---

## Part 1: Design Decision — Topology and Connectivity Model

### Decision 1: VPC Peering vs. Transit Gateway Hub-Spoke

| Factor | VPC Peering (mesh) | Transit Gateway Hub-Spoke |
|---|---|---|
| **Scaling math** | Each new VPC needs a direct peering connection to every VPC it must reach — n×(n-1)/2 connections at full mesh, unmanageable past a handful of VPCs | Each new VPC attaches once to the Transit Gateway — connections scale linearly, one attachment per VPC regardless of how many other VPCs it needs to reach |
| **Transitive routing** | Not supported — VPC A peered to B and B peered to C does **not** let A reach C; that requires a direct A-C peering too | Native — any attached VPC can reach any other attached VPC through the hub's route tables, no direct relationship required |
| **Centralized inspection/routing control** | None — traffic between two peered VPCs goes directly between them, bypassing any shared inspection point | Route tables at the Transit Gateway can force spoke-to-spoke or spoke-to-internet traffic through a shared inspection VPC before continuing |
| **Cost** | No hourly charge, billed only for data transfer | Hourly per attachment plus per-GB data processing — a real, continuous cost the moment it's deployed |
| **Best fit** | A handful of VPCs with simple, mostly bilateral connectivity needs and no requirement for centralized routing control | This scenario — a dozen-plus planned VPCs needing transitive reachability through one auditable hub |

### Decision 2: AWS Direct Connect vs. Site-to-Site VPN for Future On-Premises Connectivity

| Factor | Direct Connect | Site-to-Site VPN |
|---|---|---|
| **Connection type** | Dedicated physical fiber connection to an AWS Direct Connect location — private, not routed over the public internet at all | IPsec tunnel over the public internet — encrypted, but still subject to internet path variability |
| **Bandwidth and latency consistency** | Consistent, predictable — dedicated capacity (50 Mbps to 100 Gbps) with no contention from other internet traffic | Variable — subject to internet congestion and typically capped lower per tunnel (up to ~1.25 Gbps per tunnel) |
| **Setup time** | Weeks to months — requires physical cross-connect provisioning at a Direct Connect location, often through a partner | Minutes to hours — fully API/console-driven, no physical provisioning |
| **Cost shape** | Higher fixed cost (port-hours) but lower per-GB data transfer cost at high sustained volume | Lower fixed cost, but per-GB data transfer costs add up faster at high sustained volume, plus consuming internet-facing bandwidth |
| **Best fit** | Sustained high-volume, latency-sensitive workloads once the on-premises connection is a permanent, planned part of the architecture | An interim connection while Direct Connect provisions, a low-to-moderate volume connection, or disaster-recovery/backup path even after Direct Connect exists |

### Recommendation for This Scenario
**Transit Gateway hub-spoke.** VPC peering's non-transitive routing and quadratic connection growth make it structurally wrong for a dozen-plus planned VPCs — this isn't a cost-sensitivity call, peering genuinely cannot deliver the "any workload VPC reaches shared services through one auditable hub" requirement without an unmanageable web of direct peerings. Transit Gateway's hourly cost is the trade-off made deliberately, not accidentally, for that capability.

**Site-to-Site VPN now, Direct Connect later.** Nothing in the scenario states on-premises connectivity is needed immediately, and Direct Connect's multi-week provisioning lead time means starting that process only once it's actually urgent is too late. The standard pattern — and the one to note for the exam — is deploying Site-to-Site VPN first for immediate connectivity, then provisioning Direct Connect in parallel once volume/latency requirements justify it, and keeping the VPN as a backup path once Direct Connect is live rather than decommissioning it.

Parts 2–4 build the Transit Gateway hub-spoke topology; VPN/Direct Connect provisioning itself is out of scope for this lab (it requires either a physical cross-connect or a customer-gateway device this lab can't assume exists) but the decision above is the artifact a design review actually asks for.

---

## Part 2: Deploy the Transit Gateway Hub-Spoke Topology

### Step 1: Create the Hub (Shared Services) and Two Spoke VPCs
```bash
REGION="us-east-1"

HUB_VPC=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query "Vpc.VpcId" --output text)
SPOKE1_VPC=$(aws ec2 create-vpc --cidr-block 10.1.0.0/16 --query "Vpc.VpcId" --output text)
SPOKE2_VPC=$(aws ec2 create-vpc --cidr-block 10.2.0.0/16 --query "Vpc.VpcId" --output text)

aws ec2 create-tags --resources $HUB_VPC --tags Key=Name,Value=sap-c02-lab5-hub
aws ec2 create-tags --resources $SPOKE1_VPC --tags Key=Name,Value=sap-c02-lab5-spoke1
aws ec2 create-tags --resources $SPOKE2_VPC --tags Key=Name,Value=sap-c02-lab5-spoke2

HUB_SUBNET=$(aws ec2 create-subnet --vpc-id $HUB_VPC --cidr-block 10.0.1.0/24 --query "Subnet.SubnetId" --output text)
SPOKE1_SUBNET=$(aws ec2 create-subnet --vpc-id $SPOKE1_VPC --cidr-block 10.1.1.0/24 --query "Subnet.SubnetId" --output text)
SPOKE2_SUBNET=$(aws ec2 create-subnet --vpc-id $SPOKE2_VPC --cidr-block 10.2.1.0/24 --query "Subnet.SubnetId" --output text)
```
Non-overlapping CIDRs across all three VPCs are mandatory — Transit Gateway routing (like VPC peering) cannot route between overlapping address spaces.

### Step 2: Create the Transit Gateway and Attach All Three VPCs
```bash
TGW_ID=$(aws ec2 create-transit-gateway --description "sap-c02-lab5-hub-spoke" \
  --options AmazonSideAsn=64512,DefaultRouteTableAssociation=disable,DefaultRouteTableAssociation=disable \
  --query "TransitGateway.TransitGatewayId" --output text)

# Wait for state: available
aws ec2 wait transit-gateway-available --transit-gateway-ids $TGW_ID

HUB_ATTACH=$(aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TGW_ID \
  --vpc-id $HUB_VPC --subnet-ids $HUB_SUBNET --query "TransitGatewayVpcAttachment.TransitGatewayAttachmentId" --output text)
SPOKE1_ATTACH=$(aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TGW_ID \
  --vpc-id $SPOKE1_VPC --subnet-ids $SPOKE1_SUBNET --query "TransitGatewayVpcAttachment.TransitGatewayAttachmentId" --output text)
SPOKE2_ATTACH=$(aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TGW_ID \
  --vpc-id $SPOKE2_VPC --subnet-ids $SPOKE2_SUBNET --query "TransitGatewayVpcAttachment.TransitGatewayAttachmentId" --output text)
```
> Each `create-transit-gateway-vpc-attachment` call is now an hourly-billed attachment. The Transit Gateway itself has no separate charge beyond its attachments and data processing.

### Step 3: Add Routes in Each VPC's Route Table Pointing to the Transit Gateway
```bash
HUB_RT=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$HUB_VPC" --query "RouteTables[0].RouteTableId" --output text)
SPOKE1_RT=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$SPOKE1_VPC" --query "RouteTables[0].RouteTableId" --output text)
SPOKE2_RT=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$SPOKE2_VPC" --query "RouteTables[0].RouteTableId" --output text)

# Each spoke routes to the OTHER spoke's CIDR (and the hub's) via the Transit Gateway
aws ec2 create-route --route-table-id $SPOKE1_RT --destination-cidr-block 10.2.0.0/16 --transit-gateway-id $TGW_ID
aws ec2 create-route --route-table-id $SPOKE1_RT --destination-cidr-block 10.0.0.0/16 --transit-gateway-id $TGW_ID
aws ec2 create-route --route-table-id $SPOKE2_RT --destination-cidr-block 10.1.0.0/16 --transit-gateway-id $TGW_ID
aws ec2 create-route --route-table-id $SPOKE2_RT --destination-cidr-block 10.0.0.0/16 --transit-gateway-id $TGW_ID
aws ec2 create-route --route-table-id $HUB_RT --destination-cidr-block 10.1.0.0/16 --transit-gateway-id $TGW_ID
aws ec2 create-route --route-table-id $HUB_RT --destination-cidr-block 10.2.0.0/16 --transit-gateway-id $TGW_ID
```
This is the routing design that directly implements Decision 1: spoke-to-spoke traffic (10.1.0.0/16 to 10.2.0.0/16) is reachable purely because both attach to the same Transit Gateway — no direct relationship between the two spokes exists or is needed, unlike the peering model this replaced.

**Validation checkpoint**:
```bash
aws ec2 describe-transit-gateway-attachments --filters "Name=transit-gateway-id,Values=$TGW_ID" \
  --query "TransitGatewayAttachments[].{VPC:ResourceId,State:State}" --output table
```
Confirm all three attachments show `State: available`. Launch a test instance in each spoke's subnet (small, no public IP) and confirm one can reach the other's private IP via ICMP or a simple TCP listener — that cross-spoke reachability with zero direct peering between the spokes is the proof this topology works as designed.

---

## Part 3: PrivateLink — VPC Interface Endpoint

### Step 4: Deploy an Interface Endpoint for Systems Manager in Spoke 1
Interface endpoints (PrivateLink) let a VPC reach a supported AWS service over a private ENI instead of the public internet — this matters most for services a Transit Gateway hub design shouldn't be routing to the internet just to reach an AWS API.
```bash
SPOKE1_SG=$(aws ec2 create-security-group --group-name sap-c02-lab5-endpoint-sg \
  --description "HTTPS from spoke1 only" --vpc-id $SPOKE1_VPC --query "GroupId" --output text)
aws ec2 authorize-security-group-ingress --group-id $SPOKE1_SG --protocol tcp --port 443 --cidr 10.1.0.0/16

aws ec2 create-vpc-endpoint \
  --vpc-id $SPOKE1_VPC \
  --service-name com.amazonaws.$REGION.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids $SPOKE1_SUBNET \
  --security-group-ids $SPOKE1_SG \
  --private-dns-enabled
```
`--private-dns-enabled` is what makes this transparent to applications — the standard `ssm.<region>.amazonaws.com` hostname resolves to the endpoint's private IP inside the VPC instead of a public one, with no code or config change required on anything calling it.

**Validation checkpoint**:
```bash
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$SPOKE1_VPC" \
  --query "VpcEndpoints[0].{State:State,PrivateDns:PrivateDnsEnabled}"
```
Confirm `State: available`. From an instance in Spoke 1's subnet: `nslookup ssm.$REGION.amazonaws.com` should resolve to a `10.1.1.x` address, not a public one.

---

## Part 4: WAF, Shield, and CloudFront — Perimeter Placement

### Step 5: Deploy a WAF Web ACL and Associate It With an ALB
```bash
ALB_SG=$(aws ec2 create-security-group --group-name sap-c02-lab5-alb-sg --description "ALB ingress" --vpc-id $SPOKE1_VPC --query "GroupId" --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
SPOKE1_SUBNET2=$(aws ec2 create-subnet --vpc-id $SPOKE1_VPC --cidr-block 10.1.2.0/24 --query "Subnet.SubnetId" --output text)

ALB_ARN=$(aws elbv2 create-load-balancer --name sap-c02-lab5-alb \
  --subnets $SPOKE1_SUBNET $SPOKE1_SUBNET2 --security-groups $ALB_SG --query "LoadBalancers[0].LoadBalancerArn" --output text)

WEB_ACL_ID=$(aws wafv2 create-web-acl \
  --name sap-c02-lab5-waf \
  --scope REGIONAL \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=sap-c02-lab5-waf \
  --rules '[{"Name":"AWS-AWSManagedRulesCommonRuleSet","Priority":0,"OverrideAction":{"None":{}},
    "Statement":{"ManagedRuleGroupStatement":{"VendorName":"AWS","Name":"AWSManagedRulesCommonRuleSet"}},
    "VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"CommonRuleSet"}}]' \
  --query "Summary.Id" --output text)

WEB_ACL_ARN=$(aws wafv2 get-web-acl --name sap-c02-lab5-waf --scope REGIONAL --id $WEB_ACL_ID --query "WebACL.ARN" --output text)
aws wafv2 associate-web-acl --web-acl-arn $WEB_ACL_ARN --resource-arn $ALB_ARN
```
The AWS-managed Common Rule Set covers OWASP Top 10-style patterns (SQLi, XSS payloads) without hand-writing rules — a reasonable default before layering custom rules for application-specific abuse patterns.

### Step 6: Understand Where Shield and CloudFront Fit Relative to This WAF
This lab deploys a **regional** WAF Web ACL directly on the ALB — the right placement when there's no CDN in front of it. In a design with CloudFront in front of the ALB (the pattern Lab 6 builds), the WAF association point moves to CloudFront instead (`--scope CLOUDFRONT`), inspecting and blocking malicious requests at the edge, before they ever reach the region at all — strictly better for both latency (attack traffic dropped closer to the source) and origin load (the ALB and everything behind it never sees the blocked requests).

**Shield Standard is automatic and free** on every CloudFront distribution and ALB — it covers common network/transport-layer (L3/L4) DDoS patterns with no configuration. **Shield Advanced** is a paid, opt-in service adding L7 DDoS protection, cost protection against scaling charges incurred during an attack, and direct access to the AWS DDoS Response Team — justified for internet-facing applications where a DDoS event has a real, quantifiable business cost, not a default for every workload.

**Validation checkpoint**:
```bash
aws wafv2 get-web-acl-for-resource --resource-arn $ALB_ARN --query "WebACL.Name"
```
Confirm it returns `sap-c02-lab5-waf`, proving the association is active.

---

## Cleanup

**Transit Gateway attachments are the meter that's running right now — remove them first.**

```bash
# 1. Transit Gateway attachments FIRST
aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id $HUB_ATTACH
aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id $SPOKE1_ATTACH
aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id $SPOKE2_ATTACH

# 2. Transit Gateway itself (only after all attachments report deleted)
aws ec2 delete-transit-gateway --transit-gateway-id $TGW_ID

# 3. WAF association and Web ACL
aws wafv2 disassociate-web-acl --resource-arn $ALB_ARN
aws wafv2 delete-web-acl --name sap-c02-lab5-waf --scope REGIONAL --id $WEB_ACL_ID --lock-token $(aws wafv2 get-web-acl --name sap-c02-lab5-waf --scope REGIONAL --id $WEB_ACL_ID --query "LockToken" --output text)

# 4. ALB
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN

# 5. VPC interface endpoint
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $(aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$SPOKE1_VPC" --query "VpcEndpoints[0].VpcEndpointId" --output text)

# 6. Security groups
for sg in $ALB_SG $SPOKE1_SG; do aws ec2 delete-security-group --group-id $sg; done

# 7. Subnets and VPCs
for vpc in $HUB_VPC $SPOKE1_VPC $SPOKE2_VPC; do
  for subnet in $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$vpc" --query "Subnets[].SubnetId" --output text); do
    aws ec2 delete-subnet --subnet-id $subnet
  done
  aws ec2 delete-vpc --vpc-id $vpc
done
```

Confirm the Transit Gateway specifically is gone before moving on to anything else:
```bash
aws ec2 describe-transit-gateways --transit-gateway-ids $TGW_ID --query "TransitGateways[0].State"
```
Expect either an error (already deleted) or `State: deleted`. Attachments billed by the hour even at zero traffic — don't leave this "for later."

---

## Key Concepts

| Term | Definition |
|---|---|
| **Transit Gateway** | A regional hub that VPCs (and VPNs, Direct Connect gateways) attach to once, gaining transitive routing to every other attachment through centrally managed route tables |
| **VPC Peering** | A direct, non-transitive connection between exactly two VPCs — connection count grows quadratically as more VPCs need mutual reachability |
| **Direct Connect** | A dedicated physical connection from on-premises to AWS, bypassing the public internet — high setup lead time, consistent bandwidth/latency, lower per-GB cost at high volume |
| **Site-to-Site VPN** | An IPsec tunnel over the public internet connecting on-premises to a VPC or Transit Gateway — fast to provision, variable performance, common as an interim or backup path to Direct Connect |
| **PrivateLink / Interface Endpoint** | A private ENI in your VPC providing access to a supported AWS (or third-party) service without traversing the public internet, with optional private DNS override |
| **AWS WAF** | A layer-7 web application firewall that inspects HTTP(S) requests against rule groups (managed or custom) before they reach an ALB, API Gateway, or CloudFront origin |
| **Shield Standard vs. Advanced** | Standard is automatic, free, L3/L4 DDoS protection on every ALB/CloudFront distribution; Advanced is paid, opt-in, adds L7 protection, cost protection, and DRT access |

---

## Common Mistakes
- **Building a VPC peering mesh and discovering the transitive-routing limitation only after the third or fourth VPC**: peering never becomes transitive no matter how many connections exist — this has to be designed around from the start, not patched later
- **Forgetting Transit Gateway bills per attachment, not just per gateway**: a design with ten spoke VPCs has ten hourly-billed attachments, not one flat gateway fee — size the cost estimate accordingly
- **Skipping `--private-dns-enabled` on an interface endpoint**: without it, existing application code still resolves the service's public hostname and never actually uses the private path, even though the endpoint exists
- **Placing WAF at the ALB when CloudFront is also in the design**: this inspects requests only after they've already reached the region — placing the Web ACL at CloudFront (`--scope CLOUDFRONT`) blocks malicious traffic at the edge instead
- **Assuming Shield Standard is sufficient for every internet-facing workload without evaluating Shield Advanced's cost-protection and L7 coverage**: Standard has no L7 DDoS protection at all — a workload with real DDoS exposure needs that evaluation explicitly, not a default assumption

---

## Next Steps
This lab assumes comfort with the VPC, subnet, and security group fundamentals from [SAA-C03 Lab 1: VPC, Security Groups & IAM](../SAA-C03/aws-lab-1-vpc-iam.md), and overlaps directly with the network/perimeter depth in [SCS-C02 Lab 5: Network & Perimeter Security](../SCS-C02/aws-sec-lab-5-network-perimeter-security.md) — that lab goes deeper on Network Firewall, Flow Logs, and stateful/stateless inspection than this one re-teaches. Continue to [Lab 6: Multi-Region & Global Distribution Design](lab-6-multiregion-global-distribution-design.md) to extend this network foundation across regions, or jump to [Lab 4: Compute & Application Architecture Design](lab-4-compute-application-architecture-design.md) to review the compute tier this network would carry traffic for.
