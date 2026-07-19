# AWS Security Lab 5: Network & Perimeter Security

Check box if done: []

## Overview
IAM controls who can call an API. Network controls determine what can reach a resource at all — and the SCS-C02 exam, like real infrastructure security work, treats these as two separate layers that both have to hold. This lab builds that layer from the inside out: security groups and NACLs at the instance/subnet boundary, VPC endpoints so traffic to AWS services never crosses the public internet, Network Firewall for stateful inline inspection of traffic that does need to leave the VPC, WAF and Shield at the edge in front of a public application, and Flow Logs analysis to actually investigate what's crossing these boundaries instead of just trusting the rules are working.

**Estimated time**: 90–110 minutes
**Cost**: ~$2–$6 (Network Firewall bills ~$0.395/hour plus per-GB processing — the single most expensive resource in this lab; tear it down the same session. WAF is ~$5/month per web ACL prorated to fractions of a cent for a lab session; VPC endpoints and Flow Logs are near-free at this scale)

---

## Scenario
A security review of your VPC turned up three findings: security group rules are so broad nobody can explain why half of them exist, EC2 instances reach S3 and DynamoDB over the public internet via a NAT Gateway (an unnecessary cost and exposure path), and there's no inline inspection of egress traffic — if an instance were compromised, nothing would notice it beaconing out to a command-and-control server. Separately, a public-facing application has no protection against common web exploits or a volumetric DDoS attempt. You're closing all four gaps and proving each control actually does something with real traffic, not just reading as "enabled" in the console.

---

## Objectives
- Explain and demonstrate the stateful-vs-stateless distinction between security groups and NACLs, and use each for what it's actually good at
- Deploy Gateway and Interface VPC endpoints to remove public internet dependency for AWS service traffic
- Deploy AWS Network Firewall for stateful, rule-group-based inline inspection of VPC traffic
- Attach AWS WAF to a public endpoint and understand Shield Standard vs. Shield Advanced
- Query VPC Flow Logs with Athena to investigate a specific traffic pattern

---

## Part 1: Security Groups vs. NACLs — Stateful vs. Stateless

### Step 1: Why This Distinction Is Tested (and Misunderstood)
A **security group** is stateful and attaches to an ENI (instance-level): if you allow inbound traffic, the response traffic is automatically allowed back out, regardless of outbound rules. A **network ACL (NACL)** is stateless and attaches to a subnet: inbound and outbound rules are evaluated independently, so allowing a request in does *not* automatically allow the response out — you have to explicitly permit both directions, including the ephemeral port range the client's OS chose for the return leg.

| | Security Group | Network ACL |
|---|---|---|
| **Scope** | Instance/ENI level | Subnet level |
| **State** | Stateful — return traffic auto-allowed | Stateless — inbound/outbound evaluated independently |
| **Rule evaluation** | All rules evaluated; explicit allow only (no deny rules) | Rules evaluated in numbered order, first match wins; supports explicit deny |
| **Default behavior** | Deny all inbound, allow all outbound | Default NACL allows all; custom NACLs deny all until rules added |
| **Best fit** | Primary access control — "what can talk to this instance" | Coarse subnet-wide blocklist — "block this CIDR from an entire subnet regardless of instance-level rules" |
| **Ephemeral ports** | Not your concern — statefulness handles it | Must explicitly allow (typically 1024–65535) for return traffic |

**The practical takeaway**: security groups are your primary, day-to-day access control. NACLs are best used sparingly, for something a security group can't do — like a hard, subnet-wide block against a specific CIDR (e.g., a known-malicious range) that you want enforced regardless of what any instance's security group says, because NACLs support explicit deny and security groups don't.

### Step 2: Build a Deliberately Broad Security Group, Then Tighten It
```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)

aws ec2 create-security-group \
  --group-name netlab-broad-sg \
  --description "Intentionally over-permissive for comparison" \
  --vpc-id $VPC_ID

SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=netlab-broad-sg" --query "SecurityGroups[0].GroupId" --output text)

# The anti-pattern: SSH open to the entire internet
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0

# Tighten: revoke the broad rule, replace with a scoped CIDR representing a corporate/admin range
aws ec2 revoke-security-group-ingress \
  --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID --protocol tcp --port 22 --cidr 10.0.0.0/16
```

**Validation checkpoint**: `aws ec2 describe-security-groups --group-ids $SG_ID --query "SecurityGroups[0].IpPermissions"` should show only the scoped `10.0.0.0/16` rule — confirm `0.0.0.0/0` no longer appears anywhere in the output. Port 22 open to the world is one of the single most common findings in real cloud security audits, and it's exactly what Lab 6's Session Manager pattern eliminates the *need* for entirely.

### Step 3: Add a Subnet-Wide NACL Deny for a Specific CIDR
```bash
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text)
NACL_ID=$(aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=$SUBNET_ID" --query "NetworkAcls[0].NetworkAclId" --output text)

# Deny a specific CIDR at the subnet level, evaluated before any security group is even consulted
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID --rule-number 90 \
  --protocol -1 --rule-action deny --egress false \
  --cidr-block 198.51.100.0/24
```

**Why rule number 90, not 100 or higher?** NACL rules evaluate in ascending numeric order and stop at the first match — a lower number needs to be checked before any broader "allow" rule with a higher number, or the deny never gets a chance to apply.

---

## Part 2: VPC Endpoints — Keeping AWS Service Traffic Off the Public Internet

### Step 4: Why This Matters Beyond Cost
Without a VPC endpoint, an EC2 instance reaching S3 or DynamoDB routes that traffic through a NAT Gateway and out to the public AWS service endpoint over the internet (still TLS-encrypted, but still leaving your VPC and incurring NAT data-processing charges). A **VPC endpoint** creates a private path to the AWS service that never touches the public internet — smaller attack surface, no NAT dependency for that traffic, and often cheaper at volume.

### Step 5: Deploy a Gateway Endpoint for S3
Gateway endpoints (S3 and DynamoDB only) work via route table entries, not ENIs, and cost nothing beyond normal data transfer:
```bash
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query "RouteTables[0].RouteTableId" --output text)

aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids $ROUTE_TABLE_ID \
  --vpc-endpoint-type Gateway
```

### Step 6: Deploy an Interface Endpoint (PrivateLink) for Secrets Manager
Most other AWS services use **Interface endpoints** — an ENI with a private IP in your subnet, backed by AWS PrivateLink:
```bash
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --vpc-endpoint-type Interface \
  --subnet-ids $SUBNET_ID \
  --security-group-ids $SG_ID
```

| | Gateway Endpoint | Interface Endpoint (PrivateLink) |
|---|---|---|
| **Supported services** | S3, DynamoDB only | Most other AWS services (Secrets Manager, KMS, SNS, SQS, STS, etc.) |
| **Mechanism** | Route table entry | ENI with private IP, DNS-resolved |
| **Cost** | Free | Hourly charge per AZ + per-GB data processed |
| **Security control point** | Endpoint policy + existing IAM/resource policies | Endpoint policy + security group on the ENI itself |

### Step 7: Scope the Endpoint with a Policy — Don't Leave It Wide Open
A default endpoint allows access to the entire service for anyone in the VPC. Scope it to what's actually needed:
```bash
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id <endpoint-id-from-step-5> \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::netlab-*/*"
    }]
  }'
```

**Validation checkpoint**: From an instance in this VPC, `aws s3 ls` against a bucket outside the `netlab-*` prefix should still work via IAM if permitted, but write/read through *this specific endpoint path* to non-matching buckets is denied by the endpoint policy — confirming the endpoint itself, not just IAM, is enforcing scope. Check **VPC** → **Endpoints** → your endpoint → **Policy** in the console to review the effective document.

---

## Part 3: AWS Network Firewall — Stateful Inline Inspection

### Step 8: What Network Firewall Adds That Security Groups Can't Do
Security groups and NACLs make allow/deny decisions based on IP, port, and protocol. They cannot inspect *inside* permitted traffic — they can't tell you that outbound HTTPS traffic to port 443 is actually a C2 beacon disguised as normal web traffic, or block by domain name, or apply Suricata-compatible intrusion detection/prevention rules. **AWS Network Firewall** sits inline in a dedicated subnet and does exactly that.

### Step 9: Create a Firewall Policy with a Domain-Block Rule Group
```bash
aws network-firewall create-rule-group \
  --rule-group-name netlab-domain-block \
  --type STATEFUL \
  --capacity 100 \
  --rule-group '{
    "RulesSource": {
      "RulesSourceList": {
        "Targets": ["malicious-example.test"],
        "TargetTypes": ["TLS_SNI", "HTTP_HOST"],
        "GeneratedRulesType": "DENYLIST"
      }
    }
  }'

RULE_GROUP_ARN=$(aws network-firewall describe-rule-group --rule-group-name netlab-domain-block --type STATEFUL --query "RuleGroupResponse.RuleGroupArn" --output text)

aws network-firewall create-firewall-policy \
  --firewall-policy-name netlab-fw-policy \
  --firewall-policy '{
    "StatelessDefaultActions": ["aws:forward_to_sfe"],
    "StatelessFragmentDefaultActions": ["aws:forward_to_sfe"],
    "StatefulRuleGroupReferences": [{"ResourceArn": "'"$RULE_GROUP_ARN"'"}]
  }'
```

### Step 10: Deploy the Firewall Into a Dedicated Firewall Subnet
Network Firewall requires its own subnet, separate from the workload subnet, with route tables directing traffic through it:
```bash
FW_SUBNET_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.250.0/28 --query "Subnet.SubnetId" --output text)

aws network-firewall create-firewall \
  --firewall-name netlab-firewall \
  --firewall-policy-arn $(aws network-firewall describe-firewall-policy --firewall-policy-name netlab-fw-policy --query "FirewallPolicyResponse.FirewallPolicyArn" --output text) \
  --vpc-id $VPC_ID \
  --subnet-mappings SubnetId=$FW_SUBNET_ID
```

**Why route tables matter here**: creating the firewall resource alone inspects nothing. Traffic only flows through it if the workload subnet's route table sends outbound traffic to the firewall endpoint first, and the firewall subnet's route table sends inspected traffic on to an internet gateway or NAT. Skipping this wiring is the most common reason a freshly deployed Network Firewall "isn't blocking anything" — it was never in the traffic path.

### Step 11: Validate the Block
From an instance routed through the firewall, attempt to resolve/reach the blocked domain and confirm it fails, while traffic to an unrelated, unblocked domain succeeds — proving the firewall is inspecting selectively, not blocking everything or nothing.

---

## Part 4: WAF and Shield — Edge Protection

### Step 12: Attach AWS WAF to a Public Endpoint
WAF evaluates HTTP(S) requests against rules before they reach an ALB, API Gateway, or CloudFront distribution — the layer that stops SQL injection, XSS, and known bad request patterns before your application code ever sees them.
```bash
aws wafv2 create-web-acl \
  --name netlab-web-acl \
  --scope REGIONAL \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=netlabWebAcl \
  --rules '[{
    "Name": "AWSManagedCommonRuleSet",
    "Priority": 0,
    "OverrideAction": {"None": {}},
    "Statement": {"ManagedRuleGroupStatement": {"VendorName": "AWS", "Name": "AWSManagedRulesCommonRuleSet"}},
    "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "commonRuleSet"}
  }]'
```

Associate it with an existing ALB:
```bash
aws wafv2 associate-web-acl \
  --web-acl-arn <web-acl-arn-from-create-output> \
  --resource-arn <your-alb-arn>
```

### Step 13: Understand Shield Standard vs. Shield Advanced

| | Shield Standard | Shield Advanced |
|---|---|---|
| **Cost** | Free, automatic for every AWS account | $3,000/month per organization, 1-year commitment |
| **Coverage** | Network/transport layer (L3/L4) volumetric and common attacks | Adds application-layer (L7) protection, larger/more sophisticated attack coverage |
| **Visibility** | None — protection happens silently | Attack diagnostics, near-real-time notifications, historical reports |
| **Cost protection** | None | DDoS cost protection — reimburses scaling charges incurred during an attack |
| **Support** | None | 24/7 access to the AWS DDoS Response Team (DRT) |
| **When it's worth it** | Sufficient for most workloads | Justified for internet-facing, revenue-critical applications where an extended outage has a quantifiable cost that exceeds $36k/year |

**For this lab**: Shield Standard is already active on your account with no action required — there's nothing to configure. The exam and real architecture decisions hinge on knowing *when* Shield Advanced's cost is justified, not on deploying it in a lab (its cost commitment makes that impractical for a self-funded exercise).

### Step 14: Validate WAF Is Actually Blocking Something
Send a request containing an obvious SQLi pattern at the protected endpoint and confirm it's blocked, then send a clean request and confirm it succeeds:
```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://<your-alb-dns>/?id=1' OR '1'='1"
curl -s -o /dev/null -w "%{http_code}\n" "http://<your-alb-dns>/"
```
**Expected result**: the first request returns `403` (blocked by the managed rule set), the second returns your application's normal response code — proving WAF discriminates rather than blocking indiscriminately.

---

## Part 5: VPC Flow Logs — Investigating Traffic with Athena

### Step 15: Enable Flow Logs to S3
```bash
aws s3api create-bucket --bucket netlab-flowlogs-$RANDOM --region us-east-1
FLOW_BUCKET=$(aws s3api list-buckets --query "Buckets[?starts_with(Name, 'netlab-flowlogs')].Name" --output text)

aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination "arn:aws:s3:::$FLOW_BUCKET" \
  --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${log-status}'
```

### Step 16: Query Flow Logs with Athena
```sql
CREATE EXTERNAL TABLE netlab_flow_logs (
  version int, account_id string, interface_id string, srcaddr string, dstaddr string,
  srcport int, dstport int, protocol int, packets bigint, bytes bigint,
  start bigint, end bigint, action string, log_status string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ' '
LOCATION 's3://netlab-flowlogs-<suffix>/AWSLogs/<your-account-id>/vpcflowlogs/';
```

Then investigate rejected traffic — often the first signal of a misconfigured security group or an actual scan against your environment:
```sql
SELECT srcaddr, dstaddr, dstport, COUNT(*) AS attempts
FROM netlab_flow_logs
WHERE action = 'REJECT'
GROUP BY srcaddr, dstaddr, dstport
ORDER BY attempts DESC
LIMIT 20;
```

**Validation checkpoint**: this query should surface the SSH attempt against `netlab-broad-sg` from Part 1 if it's still within the log window, along with any port-scanning traffic hitting rejected ports — this is the concrete "investigate an anomaly" skill, not just "logging is enabled." A high `attempts` count from a single `srcaddr` against many different `dstport` values in a short window is the classic signature of a port scan.

---

## Part 6: Cleanup

```bash
# Network Firewall — delete in dependency order (most expensive resource first)
aws network-firewall delete-firewall --firewall-name netlab-firewall
aws network-firewall delete-firewall-policy --firewall-policy-name netlab-fw-policy
aws network-firewall delete-rule-group --rule-group-name netlab-domain-block --type STATEFUL
aws ec2 delete-subnet --subnet-id $FW_SUBNET_ID

# WAF
aws wafv2 disassociate-web-acl --resource-arn <your-alb-arn>
aws wafv2 delete-web-acl --name netlab-web-acl --scope REGIONAL --id <web-acl-id> --lock-token <lock-token>

# VPC Endpoints
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <s3-endpoint-id> <secretsmanager-endpoint-id>

# Flow Logs
aws ec2 delete-flow-logs --flow-log-ids <flow-log-id>
aws s3 rm s3://$FLOW_BUCKET --recursive
aws s3 rb s3://$FLOW_BUCKET

# Security group and NACL entry
aws ec2 delete-network-acl-entry --network-acl-id $NACL_ID --rule-number 90 --egress false
aws ec2 delete-security-group --group-id $SG_ID
```

> **Cost warning**: Network Firewall bills hourly per endpoint (~$0.395/hr) plus data processing regardless of traffic volume — this is the resource most likely to generate an unexpected bill if left running. Confirm `aws network-firewall list-firewalls` returns empty before ending the session.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Security group vs. NACL stateful/stateless distinction** | Misusing these is one of the most common real-world network misconfigurations and a frequent exam trap |
| **Gateway and Interface VPC endpoints** | Removes public internet dependency for AWS service traffic, shrinking attack surface and often reducing NAT cost |
| **AWS Network Firewall deployment and routing** | Stateful, signature-based inline inspection that security groups and NACLs structurally cannot provide |
| **WAF managed rule sets on a public endpoint** | Stops common web exploits before they reach application code |
| **Shield Standard vs. Advanced trade-off** | Informed cost/coverage decision-making rather than defaulting to the expensive option "to be safe" |
| **Flow Logs + Athena investigation** | The actual mechanics of investigating a traffic anomaly, not just confirming logging is turned on |

---

## Common Mistakes to Avoid
- **Deploying Network Firewall without updating route tables**: the firewall inspects nothing if traffic was never routed through it — this is the single most common deployment mistake
- **Treating NACLs as the primary access control**: they're stateless and easy to misconfigure (forgetting ephemeral ports); use security groups as primary and NACLs only for coarse subnet-wide blocks
- **Leaving a VPC endpoint's policy at the wide-open default**: an endpoint without a scoped policy grants access to the entire service to anything in the VPC, not just what you intended
- **Committing to Shield Advanced's $3,000/month without a quantified justification**: Shield Standard covers most workloads; Advanced is for internet-facing, revenue-critical applications where the cost is provably justified
- **Enabling Flow Logs and never querying them**: logs nobody looks at provide the same real-world security value as no logs at all

---

## Next Steps
- Continue to [Lab 6: Compute & Container Security](aws-sec-lab-6-compute-container-security.md) to apply the same "eliminate the exposed attack surface" logic from this lab's security-group tightening to EC2 access patterns directly
- Add a CloudWatch Logs Insights query as an alternative to the Athena approach in Part 5, and compare query ergonomics for near-real-time vs. historical investigation
- Extend the Network Firewall rule group with a Suricata-format IDS/IPS signature set instead of a simple domain denylist
- Compare this security-group/NACL model directly against Azure NSGs (single stateful layer, no separate stateless equivalent) from [AZ-500 Lab 2](../../Azure/AZ-500/lab-2-platform-network-security.md) — a common interview comparison question
