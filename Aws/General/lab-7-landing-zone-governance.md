# Lab 7: Landing Zone & Governance at Scale

Check box if done: [ ]

## Overview
A single well-configured AWS account doesn't scale to a real organization — at 5 accounts, let alone 50, "is every account configured correctly" stops being answerable by checking each one by hand. This lab covers the governance layer that makes an account count irrelevant to that question: an Organizational Unit hierarchy that groups accounts by blast-radius and purpose, Service Control Policies that make certain actions structurally impossible regardless of local IAM, Tag Policies that standardize (and report on) tagging compliance org-wide, and a policy-as-code approach so none of this lives only as something someone once clicked into place in the console.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (AWS Organizations, SCPs, and Tag Policies are free — this lab is entirely governance configuration, no billable resources)

---

## Scenario
Your company is past the "one AWS account for everything" stage — new teams keep requesting accounts, and there's no consistent guardrail preventing someone from disabling CloudTrail in a new account, no enforced tagging standard so Lab 6's cost-attribution tags actually exist everywhere, and no repeatable way to stand up a new account's governance baseline other than someone remembering to click the same handful of console settings again. You're fixing the structural piece: an OU hierarchy, SCP guardrails, a Tag Policy backing Lab 6's tagging scheme, and all of it defined as code instead of console clicks nobody can review in a pull request.

**Note up front**: like [SCS-C02 Lab 7](../SCS-C02/aws-sec-lab-7-multi-account-governance.md), this lab assumes a real AWS Organization with multiple accounts to fully exercise SCP and Tag Policy enforcement — there's no way to meaningfully test "does this block a different account" against a single account. **If you're working in a single personal AWS account with no Organization**: follow this lab conceptually — read each policy and understand exactly what it would enforce and why, and configure the Organization itself as a management-account-only exercise, understanding that the real payoff (guardrails that hold across every account, without per-account setup) only shows up with genuine multiple accounts.

---

## Objectives
- Design an OU hierarchy that groups accounts by governance need, not org chart
- Write and attach Service Control Policies establishing a landing-zone security baseline
- Understand the difference between an SCP (preventive, blocks the API call) and a Tag Policy (reporting, flags non-compliance) — and how to combine them for actual enforcement
- Deploy the OU structure and policies as code, not console clicks
- Explain what an SCP can never do (it is not a substitute for IAM)

---

## Part 1: AWS Organizations — Structure Before Policy

### Step 1: Confirm or Create the Organization
```bash
aws organizations describe-organization
# If this errors with AWSOrganizationsNotInUseException:
aws organizations create-organization --feature-set ALL
```
`--feature-set ALL` is required — the default `CONSOLIDATED_BILLING` feature set supports neither SCPs nor Tag Policies.

### Step 2: The OU Shape This Lab Assumes
```
Management Account (root of the Organization)
  ├── Organizational Unit: Security
  │     └── security-tooling-account
  ├── Organizational Unit: Workloads
  │     ├── Organizational Unit: Production
  │     │     └── prod-account
  │     └── Organizational Unit: NonProduction
  │           ├── dev-account
  │           └── staging-account
  └── Organizational Unit: Sandbox
        └── sandbox-account (individual/experimental use, loosest guardrails)
```
The shape follows blast radius, not the org chart: `Production` gets the strictest SCPs, `Sandbox` gets the most permissive (still guardrailed, just not to the same degree), and `Security` is isolated so the tooling account watching everything else isn't itself subject to the same policies it's auditing.

```bash
ROOT_ID=$(aws organizations list-roots --query "Roots[0].Id" --output text)

aws organizations create-organizational-unit --parent-id $ROOT_ID --name Workloads
WORKLOADS_ID=$(aws organizations list-organizational-units-for-parent --parent-id $ROOT_ID --query "OrganizationalUnits[?Name=='Workloads'].Id" --output text)

aws organizations create-organizational-unit --parent-id $WORKLOADS_ID --name Production
aws organizations create-organizational-unit --parent-id $WORKLOADS_ID --name NonProduction
aws organizations create-organizational-unit --parent-id $ROOT_ID --name Sandbox
```

---

## Part 2: Service Control Policies — a Landing-Zone Baseline

### Step 3: SCPs Are Not IAM
An SCP never *grants* permission — it only ever narrows what's possible, by defining the maximum available permissions for every principal in the account(s) it's attached to, IAM policies included. A user with `AdministratorAccess` in an account whose SCP denies `cloudtrail:StopLogging` genuinely cannot stop CloudTrail, full stop — no IAM policy in that account can override an SCP deny.

### Step 4: A Baseline Guardrail Set
The kind of guardrails a landing zone applies to every workload account regardless of what that account is for:

```bash
aws organizations create-policy \
  --name LandingZoneBaseline \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyLeavingOrganization",
        "Effect": "Deny",
        "Action": "organizations:LeaveOrganization",
        "Resource": "*"
      },
      {
        "Sid": "DenyDisablingCloudTrail",
        "Effect": "Deny",
        "Action": ["cloudtrail:StopLogging", "cloudtrail:DeleteTrail", "cloudtrail:UpdateTrail"],
        "Resource": "*"
      },
      {
        "Sid": "DenyRootUserActions",
        "Effect": "Deny",
        "Action": "*",
        "Resource": "*",
        "Condition": {"StringLike": {"aws:PrincipalArn": "arn:aws:iam::*:root"}}
      },
      {
        "Sid": "RestrictToApprovedRegions",
        "Effect": "Deny",
        "NotAction": ["iam:*", "organizations:*", "sts:*", "support:*", "cloudfront:*", "route53:*"],
        "Resource": "*",
        "Condition": {"StringNotEquals": {"aws:RequestedRegion": ["us-east-1", "us-west-2"]}}
      }
    ]
  }'

POLICY_ID=$(aws organizations list-policies --filter SERVICE_CONTROL_POLICY --query "Policies[?Name=='LandingZoneBaseline'].Id" --output text)
aws organizations attach-policy --policy-id $POLICY_ID --target-id $WORKLOADS_ID
```
`DenyRootUserActions` is the guardrail most landing zones skip and shouldn't — the root user bypasses IAM policy entirely by design, so an SCP-level deny is the only thing that can constrain it short of not having root credentials at all (which is the actual best practice, this is defense in depth on top of it).

### Step 5: Validate the Guardrail Actually Blocks the Action
From a principal in an account under `Workloads`:
```bash
aws cloudtrail stop-logging --name <any-trail-name> --profile workload-account-test
```
**Expected result**: `AccessDeniedException` citing an explicit deny from a service control policy — even with `AdministratorAccess` in that account's own IAM.

---

## Part 3: Tag Policies — Standardizing, Not Blocking

### Step 6: The Distinction That Matters
An SCP is **preventive** — it blocks the API call outright. A **Tag Policy is not preventive by itself** — it defines what "compliant" tagging looks like and reports non-compliant resources in the Organization's Tag Policy compliance report, but a resource created with a missing or wrong-value tag still gets created. This is a real and commonly-missed distinction: Tag Policies alone give you visibility into inconsistent tagging, not enforcement.

```bash
aws organizations create-policy \
  --name RequireProjectAndEnvironmentTags \
  --type TAG_POLICY \
  --content '{
    "tags": {
      "Project": {
        "tag_key": {"@@assign": "Project"},
        "enforced_for": {"@@assign": ["ec2:instance", "s3:bucket", "rds:db"]}
      },
      "Environment": {
        "tag_key": {"@@assign": "Environment"},
        "tag_value": {"@@assign": ["dev", "staging", "prod"]},
        "enforced_for": {"@@assign": ["ec2:instance", "s3:bucket", "rds:db"]}
      }
    }
  }'

TAG_POLICY_ID=$(aws organizations list-policies --filter TAG_POLICY --query "Policies[?Name=='RequireProjectAndEnvironmentTags'].Id" --output text)
aws organizations attach-policy --policy-id $TAG_POLICY_ID --target-id $ROOT_ID
```
This backs [Lab 6's](lab-6-cost-management-finops.md) `Project`/`Environment` tagging scheme with an org-wide standard — `Environment` is constrained to an explicit allowed-value list (`dev`/`staging`/`prod`), which is how a Tag Policy catches `Environment=Dev` or `Environment=production` as non-compliant typos/drift, not just "tag missing."

### Step 7: Get Actual Enforcement — Pair It With an SCP
To make missing required tags a hard block instead of just a compliance report finding, add an SCP denying resource creation without the required tag present at request time:

```bash
aws organizations create-policy \
  --name RequireProjectTagOnCreate \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "DenyEC2LaunchWithoutProjectTag",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {"Null": {"aws:RequestTag/Project": "true"}}
    }]
  }'
```
This is the mechanism most teams miss: a Tag Policy alone tells you tagging is inconsistent after the fact; an SCP with an `aws:RequestTag` condition is what actually stops the untagged resource from being created in the first place. Use both together — the Tag Policy for standardization and reporting breadth across many resource types, the SCP for hard enforcement on the specific resource types that matter most (here, EC2 instances).

---

## Part 4: Policy-as-Code — Terraform, Not Console Clicks

### Step 8: Why Terraform Over CDK Here
[Lab 2](lab-2-iac-cdk.md) made the case for CDK where infrastructure has real conditional complexity worth expressing in a general-purpose language. Organizations governance is close to the opposite case: a handful of static, mostly-declarative policy documents attached to fixed OU targets — there's little to no benefit from loops or type-checked abstraction here, and Terraform's simpler HCL model, plus its long-established `aws_organizations_policy`/`aws_organizations_policy_attachment` resources, is the more natural fit. This is a genuine trade-off, not "always pick Terraform" — see Lab 2's Part 5 for the reverse case.

```hcl
# main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_organizations_policy" "landing_zone_baseline" {
  name    = "LandingZoneBaseline"
  type    = "SERVICE_CONTROL_POLICY"
  content = file("${path.module}/policies/landing-zone-baseline.json")
}

resource "aws_organizations_policy_attachment" "baseline_to_workloads" {
  policy_id = aws_organizations_policy.landing_zone_baseline.id
  target_id = var.workloads_ou_id
}

resource "aws_organizations_policy" "tag_policy" {
  name    = "RequireProjectAndEnvironmentTags"
  type    = "TAG_POLICY"
  content = file("${path.module}/policies/require-tags.json")
}

resource "aws_organizations_policy_attachment" "tags_to_root" {
  policy_id = aws_organizations_policy.tag_policy.id
  target_id = var.org_root_id
}
```
The policy JSON documents live as versioned files in `policies/`, reviewed in a pull request before anything is applied — the same "a security guardrail that only exists because someone clicked it into place once isn't a guardrail your organization can rely on" argument from [Lab 3's](lab-3-cicd-oidc.md) CI/CD lab, applied to governance instead of application infrastructure. Wire this into the same OIDC-federated pipeline from Lab 3 (plan-on-PR, apply-on-merge) rather than running `terraform apply` by hand from a laptop with standing Organizations-management-account credentials.

```bash
terraform init
terraform plan
terraform apply
```

---

## Cleanup

```bash
# Detach and delete SCPs
aws organizations detach-policy --policy-id $POLICY_ID --target-id $WORKLOADS_ID
aws organizations delete-policy --policy-id $POLICY_ID

# Detach and delete the Tag Policy
aws organizations detach-policy --policy-id $TAG_POLICY_ID --target-id $ROOT_ID
aws organizations delete-policy --policy-id $TAG_POLICY_ID

# Delete OUs created purely for this lab (leaf before root, and only if empty of accounts)
aws organizations delete-organizational-unit --organizational-unit-id <production-ou-id>
aws organizations delete-organizational-unit --organizational-unit-id <nonproduction-ou-id>
aws organizations delete-organizational-unit --organizational-unit-id $WORKLOADS_ID
aws organizations delete-organizational-unit --organizational-unit-id <sandbox-ou-id>
```
If this was configured against a real Organization with existing member accounts, confirm every SCP is detached before deleting any OU — an OU with an active member account can't be deleted regardless, but leftover policy attachments on OUs you do remove can otherwise orphan silently.

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Organizational Unit (OU)** | A container grouping accounts for policy attachment purposes — SCPs and Tag Policies attach to OUs (or accounts, or the root) and apply to everything beneath the attachment point |
| **SCP as a permission ceiling** | An SCP never grants access — it defines the maximum possible permissions for every principal in scope, IAM policies included; explicit deny at this layer beats any IAM allow |
| **Tag Policy (reporting, not preventive)** | Defines and reports on tagging standards org-wide, but does not by itself block a non-compliant resource from being created — enforcement requires pairing it with an SCP |
| **`aws:RequestTag` condition key** | The IAM/SCP condition key that inspects tags being applied *at the moment of the API call* — the mechanism that turns "we have a tagging standard" into "untagged resources can't be created" |
| **`DenyRootUserActions`** | An SCP statement denying all actions from the account root user via a `PrincipalArn` condition — defense in depth on top of (not instead of) simply not using root credentials |
| **Policy-as-code** | Governance policies defined as versioned files, reviewed via pull request, and applied through a pipeline — not console clicks with no audit trail of who changed what and why |

---

## Common Mistakes
- **Attaching an SCP directly to the Organization root without testing on a single OU first**: a misconfigured deny-all-by-mistake SCP at the root can lock out every account in the Organization simultaneously
- **Assuming a Tag Policy blocks non-compliant resources**: it doesn't — it's a reporting/standardization mechanism; actual enforcement needs an SCP with an `aws:RequestTag` condition alongside it
- **Forgetting SCPs never grant permissions**: a common early mistake is expecting an SCP allow statement to grant access — it only ever narrows what IAM can separately grant
- **Not carving out a break-glass exception in a restrictive SCP**: an SCP that blocks even legitimate emergency administrative action is a guardrail that creates its own incident
- **Building this once in the console instead of as Terraform**: the entire value of a landing-zone baseline is that it's reproducible and reviewable across every account — a console-only version isn't a guardrail your organization can actually rely on

---

## Next Steps
This lab covers governance breadth — OU design, Tag Policies, policy-as-code; [SCS-C02 Lab 7: Multi-Account Security & Governance](../SCS-C02/aws-sec-lab-7-multi-account-governance.md) covers the security-guardrail depth this one only touches on — SCPs as an incident-response control, delegated administration for centralized GuardDuty/Security Hub, and cross-account Athena forensics queries. The tagging enforcement built here directly backs [Lab 6's](lab-6-cost-management-finops.md) cost-allocation tagging strategy — read that lab first if you haven't, since this one assumes its tagging scheme. For the IAM fundamentals underpinning every SCP condition in this lab, see [SAA-C03 Lab 1](../SAA-C03/aws-lab-1-vpc-iam.md).
