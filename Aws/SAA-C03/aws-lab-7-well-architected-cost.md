# AWS Lab 7: Well-Architected Cost & Performance Review

Check box if done: []

## Overview
The SAA-C03 exam leans hard on cost optimization and performance efficiency questions — "which of these four options is cheapest while still meeting the requirement" is one of the most common question shapes. This lab is different from the others: instead of building new infrastructure, you're auditing what you've already built across Labs 1–6, running an actual Well-Architected Framework review, and making (and justifying) real cost/performance tradeoff decisions.

**Estimated time**: 2–3 hours
**Cost**: ~$0 (this lab reviews and analyzes existing resources; it doesn't require new infrastructure beyond what prior labs created, and Trusted Advisor / Cost Explorer are included with your account)

---

## Objectives
- Run a Well-Architected Framework review against a real workload
- Use Cost Explorer to identify what actually drove spend in this project so far
- Review S3 storage classes and lifecycle policies with real cost numbers, not just theory
- Use Trusted Advisor to find concrete optimization opportunities
- Build a tagging strategy and explain why untagged resources are a governance problem, not just a cosmetic one

---

## Part 1: Cost Explorer — What Actually Cost Money

### Step 1: Enable and Open Cost Explorer
1. **Billing and Cost Management** → **Cost Explorer** → **Launch Cost Explorer** (enable if this is your first time; data can take 24 hours to populate for a brand-new account)
2. Set the date range to cover the period you've been working through this repo's labs

### Step 2: Break Down Cost by Service
1. **Group by**: `Service`
2. Identify your top 3 cost drivers. For most people working through Labs 1–6, expect **NAT Gateway** (billed hourly regardless of traffic) and **EC2** (if instances were left running between sessions) near the top — not the services people assume, like S3 or Lambda

### Step 3: Break Down Cost by Day
1. **Group by**: none, view the daily granularity line chart
2. Look for **spikes that don't correspond to active lab work** — these are the "I forgot to tear it down" moments. If you find one, that's a real, personal example of the #1 cost-optimization lesson: idle resources cost the same as active ones unless they're serverless or explicitly stopped

**Reflection**: Note which lab (if any) left something running longest. This is worth remembering for the interview answer to "tell me about a time you found a cost issue" — a real example beats a hypothetical one.

---

## Part 2: S3 Storage Classes and Lifecycle — Revisited with Real Numbers

### Step 4: Recall Lab 3's Storage Setup
If you still have the bucket from [Lab 3](aws-lab-3-s3-security.md), open it. If not, create a small test bucket and upload a few files to work through this section conceptually.

### Step 5: Compare Storage Class Pricing Directly
1. Open the [S3 pricing page](https://aws.amazon.com/s3/pricing/) for your region
2. Build a table for your own reference:

| Storage Class | Retrieval | Min Storage Duration | Relative Cost (approx., us-east-1) |
|---------------|-----------|----------------------|-------------------------------------|
| S3 Standard | Instant | None | Baseline (1x) |
| S3 Standard-IA | Instant | 30 days | ~0.5x storage, but retrieval fee per GB |
| S3 One Zone-IA | Instant | 30 days | ~0.4x storage, single-AZ risk |
| S3 Glacier Instant Retrieval | Instant | 90 days | ~0.25x storage |
| S3 Glacier Flexible Retrieval | Minutes–hours | 90 days | ~0.18x storage |
| S3 Glacier Deep Archive | Hours | 180 days | ~0.05x storage |

3. **The exam-relevant insight**: cheaper storage classes trade retrieval speed and minimum-duration commitments for lower per-GB storage cost — and every class except Standard charges an early-deletion penalty if you delete before the minimum duration. A lifecycle policy that moves data to Glacier after 30 days, only to delete it on day 35, costs *more* than leaving it in Standard the whole time.

### Step 6: Build a Realistic Lifecycle Policy
1. Bucket → **Management** → **Lifecycle rules** → **Create lifecycle rule**
2. **Rule name**: `tiered-archive`
3. **Transition current versions**: Standard-IA after `30 days`, Glacier Flexible Retrieval after `90 days`
4. **Expiration**: after `365 days`
5. **Create rule**

**Validation checkpoint**: verify each transition day count is comfortably past that class's minimum storage duration (30 days for IA, 90 for Glacier) — a transition scheduled before the minimum duration is a design mistake, not just a missed optimization.

---

## Part 3: Right-Sizing with Trusted Advisor

### Step 7: Review Trusted Advisor's Cost Optimization Checks
1. **Trusted Advisor** → **Cost Optimization** category
2. Review whatever checks your support tier exposes (Basic support shows a limited set; Business/Enterprise shows the full list including low-utilization EC2 instance detection)
3. Even on Basic support, check the **Service Limits** and **Security** categories — they're available at every tier and worth reviewing regardless of this lab's cost focus

### Step 8: Manually Right-Size Using CloudWatch (Works at Any Support Tier)
Trusted Advisor's deeper cost checks require paid support, but you can do the same analysis manually:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<your-instance-id> \
  --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 3600 \
  --statistics Average Maximum
```

**Interpretation**: if average CPU utilization sits well under 10% and max never exceeds 20-30%, that instance is oversized for its actual workload — a smaller instance type (or Lambda, if the workload fits — see [Lab 6](aws-lab-6-serverless.md)) would cost less for the same effective capacity.

---

## Part 4: Tagging Strategy

### Step 9: Why Untagged Resources Are a Governance Problem
Cost Explorer's "group by service" view in Part 1 tells you *what* cost money, not *why* or *whose* it was. Without tags, a bill showing "$4 of EC2" for the month can't be attributed to a specific lab, project, or owner — which is exactly the gap that makes untracked spend possible in a real organization with dozens of engineers provisioning resources.

### Step 10: Define and Apply a Tagging Standard
Define a minimal, consistent tag set:

| Tag Key | Example Value | Purpose |
|---------|---------------|---------|
| `Project` | `cloud-linux-labs` | Groups cost across services for one initiative |
| `Environment` | `lab` | Distinguishes throwaway lab resources from anything meant to persist |
| `Owner` | `your-name` | Who to contact / who's accountable |
| `AutoDelete` | `true` | Signals this resource is safe for automated cleanup tooling to remove |

Apply it to any resources still running from earlier labs:
```bash
aws ec2 create-tags \
  --resources <resource-id> \
  --tags Key=Project,Value=cloud-linux-labs Key=Environment,Value=lab Key=Owner,Value=<your-name> Key=AutoDelete,Value=true
```

### Step 11: Use Cost Allocation Tags in Cost Explorer
1. **Billing and Cost Management** → **Cost Allocation Tags** → activate the tag keys you just used (can take up to 24 hours to reflect in Cost Explorer)
2. Once active, **Cost Explorer** → **Group by**: `Tag` → select `Project` — this is what lets a team answer "how much did this specific initiative cost" instead of only "how much did EC2 cost across everything"

---

## Part 5: Run a Lightweight Well-Architected Review

### Step 12: Walk Through the Five Pillars Against Lab 5's Architecture
Using the [AWS Well-Architected Tool](https://aws.amazon.com/well-architected-tool/) (free) or just working through it manually, answer for the architecture built in [Lab 5](aws-lab-5-resilient-architecture.md):

| Pillar | Question | Your Answer |
|--------|----------|--------------|
| **Operational Excellence** | Can you deploy a change without manual console clicking? | Not yet — this is exactly what [Lab 8](aws-lab-8-terraform-aws.md) fixes |
| **Security** | Is anything more broadly accessible than it needs to be? | Check every security group for `0.0.0.0/0` rules that shouldn't be there |
| **Reliability** | What's your actual RTO/RPO if an AZ fails? | You measured this directly in Lab 5, Part 3 |
| **Performance Efficiency** | Is any resource oversized for its real load? | Answered in Part 3 of this lab |
| **Cost Optimization** | Is anything running that doesn't need to be always-on? | Answered in Part 1 of this lab |

### Step 13: Document Findings
Write a short findings summary (3-5 bullets) — this is the artifact worth keeping for a portfolio or interview: "I ran a Well-Architected review against my own project and found X, Y, Z, and fixed/plan to fix them."

---

## Cleanup

This lab doesn't create billable resources of its own — if Part 3 or the tagging exercise surfaced anything still running from earlier labs that should have been torn down, clean it up now:

```bash
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --query "Reservations[].Instances[].[InstanceId,Tags]"
```

---

## Key Concepts to Understand

| Concept | Definition |
|---------|-----------|
| **Cost Explorer vs. Trusted Advisor** | Cost Explorer shows historical spend and trends; Trusted Advisor gives forward-looking recommendations (some gated behind paid support tiers) |
| **S3 minimum storage duration** | Deleting or transitioning an object before a storage class's minimum duration triggers an early-deletion charge — cheaper classes aren't free to exit early |
| **Right-sizing** | Matching instance type/size to actual observed utilization, not to a guess made at provisioning time |
| **Cost allocation tags** | Tags must be explicitly activated in Billing settings before Cost Explorer can group by them — tagging alone isn't enough |
| **Well-Architected Framework's five pillars** | Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization — a structured lens for reviewing any architecture, not just a checklist for new builds |

---

## Interview Prep: Common Questions

1. **"Tell me about a time you identified a cost optimization."** — Use your own Cost Explorer findings from Part 1 as a real, specific example instead of a hypothetical.
2. **"How would you decide between S3 Standard-IA and Glacier for a given dataset?"** — Access frequency and retrieval latency tolerance, weighed against the minimum storage duration and early-deletion penalties.
3. **"Why do tags matter for cost management?"** — Without them, spend can be seen in aggregate but not attributed to a project, team, or owner — which blocks accountability and makes cost anomalies hard to root-cause.
4. **"Walk me through the Well-Architected Framework."** — Five pillars, and the ability to name a concrete question your own project raised in each one (not just recite the pillar names) is what separates a memorized answer from a demonstrated one.

---

## Next Steps
- Set up an AWS Budget with an alert threshold so a runaway cost shows up as a notification instead of a surprise bill
- If you have access to a paid support tier, revisit the full Trusted Advisor Cost Optimization category
- Continue to [Lab 8: Terraform on AWS](aws-lab-8-terraform-aws.md) — the direct fix for the Operational Excellence gap identified in Step 12
