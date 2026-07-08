# AWS Lab 5: Resilient Multi-AZ Architecture

Check box if done: []

## Overview
Lab 2 built a load-balanced, auto-scaling web tier — but if you look closely, it likely ran in a single Availability Zone. This lab fixes that gap and adds the piece Lab 2 didn't touch: a database. You'll spread compute across two AZs, add an RDS Multi-AZ database, and then do the thing that actually proves resilience — force a failover and watch the architecture survive it.

**Estimated time**: 3–4 hours
**Cost**: ~$1–$3 (RDS db.t3.micro is free-tier eligible for 750 hrs/month; tear down at the end regardless)

---

## Objectives
- Deploy an Auto Scaling Group across two Availability Zones behind an ALB
- Deploy an RDS Multi-AZ database and understand what "Multi-AZ" actually replicates
- Force an AZ failure and observe how the ALB and ASG respond
- Force an RDS failover and measure the actual downtime
- Explain the resilience story end-to-end, the way an interviewer will ask you to

---

## Part 1: Multi-AZ Compute (Building on Lab 2)

### Step 1: Create the VPC with Subnets in Two AZs
1. **VPC** → **Create VPC** → **VPC and more** (this wizard creates subnets across AZs for you)
2. **Name tag**: `resilient-vpc`
3. **IPv4 CIDR**: `10.0.0.0/16`
4. **Number of Availability Zones**: `2`
5. **Number of public subnets**: `2`, **Number of private subnets**: `2`
6. **NAT gateways**: `1 per AZ` (this is what costs a small amount — note it, and remember to tear it down)
7. **Create VPC**

### Step 2: Create the Launch Template
1. **EC2** → **Launch Templates** → **Create launch template**
2. **Name**: `resilient-web-lt`
3. **AMI**: Amazon Linux 2023
4. **Instance type**: `t3.micro`
5. **Security group**: create/select one allowing inbound 80 from the ALB's security group only (not `0.0.0.0/0` directly to instances)
6. **User data**:
```bash
#!/bin/bash
dnf install -y nginx
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>Served from $AZ</h1>" > /usr/share/nginx/html/index.html
systemctl enable nginx
systemctl start nginx
```
7. **Create launch template**

### Step 3: Create the ALB Spanning Both AZs
1. **EC2** → **Load Balancers** → **Create load balancer** → **Application Load Balancer**
2. **Name**: `resilient-alb`
3. **Scheme**: `Internet-facing`
4. **VPC**: `resilient-vpc`, **Mappings**: select **both** public subnets (one per AZ) — this is the step that actually makes the ALB multi-AZ; picking only one subnet defeats the purpose
5. **Security group**: allow inbound 80 from `0.0.0.0/0`
6. **Listeners and routing**: create a new target group `resilient-tg`, protocol HTTP, port 80, health check path `/`
7. **Create load balancer**

### Step 4: Create the Auto Scaling Group Across Both AZs
1. **EC2** → **Auto Scaling Groups** → **Create Auto Scaling group**
2. **Launch template**: `resilient-web-lt`
3. **VPC**: `resilient-vpc`, **Subnets**: select **both private subnets** (one per AZ)
4. **Attach to an existing load balancer** → target group `resilient-tg`
5. **Health checks**: enable **ELB health checks** in addition to EC2 status checks — this is important: an instance can pass its own EC2 health check while failing to actually serve traffic (nginx crashed, disk full), and only the ELB check catches that
6. **Group size**: Desired `2`, Min `2`, Max `4`
7. **Create Auto Scaling group**

**Validation checkpoint**: Wait for instances to register as healthy, then hit the ALB's DNS name repeatedly in a browser or with `curl` in a loop — you should see the AZ name in the response alternate between both AZs, confirming traffic is actually distributed.

---

## Part 2: RDS Multi-AZ Database

### Step 5: Understand What Multi-AZ Actually Does
A common misconception: Multi-AZ is **not** a read-replica or a scaling feature — it's purely for availability. AWS provisions a synchronously-replicated **standby** in a second AZ that you never query directly; on a failure of the primary, AWS automatically promotes the standby and repoints the same DNS endpoint at it. Your application never needs to know a failover happened beyond a brief connection interruption.

### Step 6: Create an RDS Multi-AZ Instance
1. **RDS** → **Create database** → **Standard create**
2. **Engine**: `MySQL` (or PostgreSQL — either demonstrates the same concept)
3. **Templates**: `Free tier` (note: free tier normally doesn't support Multi-AZ — for this lab, switch to **Dev/Test** template instead and manually enable Multi-AZ, understanding this incurs a small cost beyond free tier)
4. **DB instance identifier**: `resilient-db`
5. **Master username/password**: set your own — don't leave defaults
6. **Instance configuration**: `db.t3.micro`
7. **Availability & durability**: **Multi-AZ DB instance** — select this option explicitly
8. **Connectivity**: **VPC**: `resilient-vpc`, create a new **DB subnet group** using the two **private** subnets, **Public access**: `No`
9. **VPC security group**: create one allowing inbound 3306 (MySQL) only from the web tier's security group — the database should never be reachable from the internet or from anything except the app tier
10. **Create database**

> Provisioning a Multi-AZ instance takes 10–15 minutes — move to Part 3 concepts while it deploys, then come back for Step 7.

### Step 7: Confirm Multi-AZ Is Active
1. **RDS** → **Databases** → `resilient-db` → **Configuration** tab
2. Confirm **Multi-AZ** shows `Yes` and note the primary's Availability Zone

---

## Part 3: Force Failures and Observe

### Step 8: Simulate an AZ Failure for the Compute Tier
1. **EC2** → **Instances** → identify one instance and its AZ
2. Select it → **Instance state** → **Stop instance** (simulating that AZ becoming unavailable to this instance)
3. Watch **EC2** → **Auto Scaling Groups** → `resilient-web-lt`'s ASG → **Activity** tab

**Expected result**: the ASG's health check marks the stopped instance unhealthy within a couple of minutes, terminates it, and launches a replacement — in whichever AZ the ASG determines is appropriate for balance. Meanwhile, the ALB continues routing traffic to the remaining healthy instance in the other AZ the entire time. Confirm the ALB's DNS name still returns `200 OK` throughout.

### Step 9: Force an RDS Failover
```bash
aws rds reboot-db-instance \
  --db-instance-identifier resilient-db \
  --force-failover
```

### Step 10: Measure the Actual Downtime
While the failover is in progress, run a small loop from a machine that can reach the DB (or via a bastion/instance inside the VPC):

```bash
while true; do
  mysql -h resilient-db.<your-endpoint>.rds.amazonaws.com -u admin -p -e "SELECT 1" 2>&1 | tail -1
  sleep 1
done
```

**Expected result**: a handful of failed connection attempts (typically 60–120 seconds) as AWS promotes the standby, updates DNS, and the new primary comes online — then connections succeed again automatically, against the **same endpoint hostname**. Nothing in your application config needs to change.

**Validation checkpoint**: **RDS** → `resilient-db` → **Configuration** — note the **Availability Zone** value changed from what you recorded in Step 7. The standby is now the primary.

---

## Part 4: The Resilience Story (Interview Prep)

### Step 11: Articulate What Actually Happened
Write out (or say out loud) the answer to: *"Walk me through what happens if an entire Availability Zone goes down in your architecture."*

A strong answer covers:
1. The ALB stops routing to targets in the failed AZ (health checks fail) and continues serving 100% of traffic from the healthy AZ
2. The ASG detects unhealthy/unreachable instances and launches replacements, preferring AZs that still have capacity
3. RDS Multi-AZ promotes its standby in the surviving AZ automatically, with the same connection endpoint
4. The application experiences a brief RDS connection interruption (seconds to low minutes) but no ALB-tier outage and no manual intervention

---

## Part 5: Cleanup

```bash
aws rds delete-db-instance \
  --db-instance-identifier resilient-db \
  --skip-final-snapshot
```

Then in the console:
1. **EC2** → **Auto Scaling Groups** → delete `resilient-web-lt`'s ASG (this terminates its instances automatically)
2. **EC2** → **Load Balancers** → delete `resilient-alb`
3. **EC2** → **Target Groups** → delete `resilient-tg`
4. **VPC** → delete `resilient-vpc` (this also removes the NAT gateways — confirm they're gone, they're the ongoing-cost item in this lab)

```bash
# Confirm nothing billable is left running
aws ec2 describe-nat-gateways --filter "Name=tag:Name,Values=resilient*" --query "NatGateways[].State"
aws rds describe-db-instances --query "DBInstances[?DBInstanceIdentifier=='resilient-db']"
```

---

## Key Concepts to Understand

| Concept | Definition |
|---------|-----------|
| **Multi-AZ (RDS)** | Synchronous standby replica in a second AZ for availability, promoted automatically on failure — not a scaling or read feature |
| **Read replica** | A separate concept from Multi-AZ — asynchronous, used for read scaling, and not automatically promoted on primary failure unless manually configured |
| **ELB health checks vs. EC2 status checks** | EC2 checks confirm the instance is running; ELB checks confirm the application is actually responding — always enable both |
| **DNS-based failover** | RDS failover works by repointing the same endpoint hostname, so application config never needs to change |
| **NAT Gateway per AZ** | Required for true AZ independence — a single shared NAT gateway makes your "multi-AZ" private subnets dependent on one AZ for outbound internet |

---

## Interview Prep: Common Questions

1. **"What's the difference between Multi-AZ and a read replica?"** — Multi-AZ is a synchronous standby for availability, invisible to the app except during failover; a read replica is asynchronous, has its own endpoint, and is for offloading read traffic, not automatic failover.
2. **"How long does an RDS Multi-AZ failover actually take?"** — Typically 60–120 seconds; you measured this directly in Step 10 rather than quoting a number from documentation.
3. **"Why put the ASG in private subnets instead of public?"** — The web tier doesn't need inbound internet access directly; the ALB is the only thing that should be internet-facing, and NAT Gateways handle any outbound needs (package updates, etc.) from the private subnets.
4. **"What would you add next for even higher availability?"** — Multi-Region failover (Route 53 health checks + a standby region), read replicas for read-heavy scaling, and RDS Proxy to pool connections and reduce failover impact further.

---

## Next Steps
- Add an RDS read replica and measure how much read load it can absorb
- Introduce RDS Proxy in front of the database and re-measure failover downtime (RDS Proxy significantly reduces failover impact for the application)
- Continue to [Lab 6: Serverless Architecture](aws-lab-6-serverless.md) to compare this always-on resilience model against a serverless alternative that sidesteps AZ-failure planning almost entirely
