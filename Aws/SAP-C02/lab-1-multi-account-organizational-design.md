# Lab 1: Multi-Account Organizational Design

Check box if done: [ ]

## Overview
SAP-C02 doesn't test whether you can create an organizational unit — that's a console form. It tests whether you can look at a company that's outgrown its single AWS account and design the account structure, guardrails, and centralized logging that keep it governable as it adds its tenth account, or its fiftieth. This lab forces that decision on paper first — single account vs. multi-account, centralized vs. per-account logging — then implements the structure the decision leads to: an OU hierarchy, Service Control Policies, AWS RAM resource sharing, and an organization-wide CloudTrail trail.

**Estimated time**: 75–90 minutes
**Cost**: ~$0 (AWS Organizations, OUs, SCPs, and RAM sharing carry no direct charge; the only spend risk is CloudTrail's S3 storage, which stays negligible for a lab-length trail)

---

## Scenario
You're the architect for a company that started with one AWS account because that's what the first engineer created three years ago. Today that account holds production, staging, and every developer's scratch workloads side by side, with IAM users hand-managed per team and no consistent tagging. Leadership just approved a formal AWS Organization: a dedicated Security OU for log aggregation and audit tooling, a Workloads OU split into Production and Non-Production, and a stated plan to onboard two acquired companies' AWS footprints within the year. Before any acquisition's account lands in this structure ungoverned, you need an account design, a guardrail model, and a centralized-logging pattern that scale without a redesign every time a new account is added.

---

## Objectives
- Design a multi-account OU hierarchy and justify it over continuing with a single account plus IAM boundaries
- Decide between centralized (organization trail, one log-archive account) and per-account CloudTrail logging, and justify the tradeoff
- Build the OU hierarchy in AWS Organizations and move an account into it
- Write and attach Service Control Policies enforcing a region restriction and a logging-tamper guardrail
- Share a resource across accounts with AWS RAM instead of duplicating it per account
- Deploy an organization-wide CloudTrail trail and validate centralized log delivery

---

## Part 1: Design Decision — Account Structure and Logging Topology

This is the part that actually maps to the exam. Don't touch the CLI until the decision is made and justified.

### Decision 1: Single Account with IAM Boundaries vs. Multi-Account OU Structure

| Factor | Single Account (IAM permission boundaries, tags, resource-level policies) | Multi-Account (AWS Organizations, OUs, SCPs) |
|---|---|---|
| **Blast radius** | One mistaken IAM policy or compromised credential can reach every workload — prod and non-prod share the same account boundary | An SCP-enforced account boundary is a hard stop that IAM policy alone can't create; a compromised dev credential can't touch the production account at all |
| **Billing isolation** | Cost allocation depends entirely on tagging discipline, which erodes as headcount grows — this company already has none | Per-account billing is exact by construction; consolidated billing still rolls it up for the CFO |
| **Guardrail enforcement** | IAM policies must be attached correctly on every principal, every time — drift is a certainty at scale, not a risk | SCPs apply at the OU level regardless of what IAM looks like inside the account — one assignment point, inherited automatically |
| **Acquired-company onboarding** | A new company's AWS footprint has to be manually merged into the shared account's IAM/tagging model, or left as a parallel unmanaged account | A new account joins the Organization, lands in the right OU, and inherits every guardrail immediately — no manual merge |
| **Overhead at this company's scale** | Low today, compounds badly with two planned acquisitions | Higher upfront design cost, flat afterward — the point of doing it now instead of after acquisition #1 |

### Decision 2: Centralized (Organization Trail + Log-Archive Account) vs. Per-Account CloudTrail

| Factor | Centralized — one organization trail delivering to a dedicated log-archive account | Per-Account CloudTrail Trails |
|---|---|---|
| **Audit completeness** | `--is-organization-trail` applies automatically to every current and future member account — no account can opt out or be forgotten | Depends on someone creating a trail in every new account; exactly the kind of manual step that gets skipped under deadline pressure |
| **Tamper resistance** | Log-archive account has no workloads and minimal IAM principals — a compromised workload account still can't reach or delete the centralized logs | Logs sit in the same account as the workload — a compromised account's attacker can plausibly disable or delete its own trail |
| **Cross-account investigation** | One S3 bucket, one Athena table, query across every account's activity at once | Federated queries across N buckets, slower and easier to miss an account |
| **Operational simplicity** | One bucket policy, one lifecycle policy, one place to manage retention | Retention and access policy duplicated (or drifting) per account |
| **Cost** | Slightly higher S3 storage in one place, but no duplicated per-account trail overhead | Same total log volume, just fragmented — no cost advantage, only operational disadvantage |

### Recommendation for This Scenario
**Multi-account OU hierarchy**: `Root` → `Security` OU (holding a dedicated `log-archive` account and, eventually, a security-tooling account) and `Workloads` OU split into `Production` and `Non-Production`. This gives the two planned acquisitions a governed landing spot from day one instead of a third and fourth ungoverned account.

**Centralized logging via a single organization trail** delivering to the log-archive account. At this company's stated trajectory — multiple acquisitions, a stated audit requirement implied by "formal AWS Organization" — the tamper-resistance argument alone (a compromised workload account cannot delete evidence of its own compromise) outweighs the marginal simplicity of per-account trails. Part 5 builds exactly this.

---

## Part 2: Build the OU Hierarchy

Replace every placeholder account ID and organization ID below with your own — never hardcode a real account ID into committed code or documentation.

```bash
aws organizations describe-organization
# If this errors with AWSOrganizationsNotInUseException, create one:
aws organizations create-organization --feature-set ALL
```
`--feature-set ALL` is required — the default `CONSOLIDATED_BILLING` feature set doesn't support SCPs at all, which makes Part 3 impossible later if skipped here.

```bash
ROOT_ID=$(aws organizations list-roots --query "Roots[0].Id" --output text)

# Security OU — holds log-archive and (eventually) security-tooling accounts
aws organizations create-organizational-unit --parent-id $ROOT_ID --name "Security"

# Workloads OU — parent for environment-split sub-OUs
aws organizations create-organizational-unit --parent-id $ROOT_ID --name "Workloads"

WORKLOADS_OU_ID=$(aws organizations list-organizational-units-for-parent \
  --parent-id $ROOT_ID --query "OrganizationalUnits[?Name=='Workloads'].Id" --output text)

aws organizations create-organizational-unit --parent-id $WORKLOADS_OU_ID --name "Production"
aws organizations create-organizational-unit --parent-id $WORKLOADS_OU_ID --name "NonProduction"
```

Real new-member accounts are created with `aws organizations create-account --email <unique-email> --account-name "<name>"` — each requires a unique, verifiable email address and takes several minutes to provision. This lab does not create throwaway member accounts (unlike a management group, an AWS account is not free to abandon — it requires deliberate closure and an email you control). Instead, move your existing account into the structure to prove inheritance end to end, exactly as a real account would experience it:

```bash
aws organizations move-account \
  --account-id <your-account-id> \
  --source-parent-id $ROOT_ID \
  --destination-parent-id <nonproduction-ou-id>
```

**Validation checkpoint**:
```bash
aws organizations list-organizational-units-for-parent --parent-id $ROOT_ID --query "OrganizationalUnits[].{Name:Name,Id:Id}" --output table
aws organizations list-accounts-for-parent --parent-id <nonproduction-ou-id> --query "Accounts[].{Name:Name,Id:Id}" --output table
```
Confirm `Security` and `Workloads` exist directly under root, `Production`/`NonProduction` exist under `Workloads`, and your account is listed under `NonProduction`.

---

## Part 3: Service Control Policies

An SCP never grants a permission — it only ever narrows what's possible for every principal in the account(s) it's attached to, IAM policies included. This is the mechanism that makes "a compromised dev credential can't touch production" from Part 1's decision actually true.

### Step 1: Region Restriction (Allow-List)
This company has no stated multi-region requirement yet — a region-restriction SCP prevents both accidental cost sprawl (someone spins up resources in an unreviewed region) and the compliance drift that comes from workloads landing somewhere nobody's monitoring.

```bash
aws organizations create-policy \
  --name RestrictToApprovedRegions \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "DenyOutsideApprovedRegions",
      "Effect": "Deny",
      "NotAction": ["iam:*", "organizations:*", "sts:*", "support:*", "cloudfront:*", "route53:*", "waf:*", "wafv2:*"],
      "Resource": "*",
      "Condition": {"StringNotEquals": {"aws:RequestedRegion": ["us-east-1", "us-west-2"]}}
    }]
  }'
```
`NotAction` combined with `Deny` makes this an allow-list: everything is denied outside the approved regions except the global services named, which aren't region-scoped to begin with.

### Step 2: Prevent Disabling the Audit Trail
This SCP is what actually backs Decision 2's tamper-resistance argument for accounts *outside* the log-archive account — even an account with local `AdministratorAccess` can't touch the organization trail's local delivery.

```bash
aws organizations create-policy \
  --name DenyCloudTrailTampering \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "DenyDisablingOrgTrail",
      "Effect": "Deny",
      "Action": ["cloudtrail:StopLogging", "cloudtrail:DeleteTrail", "cloudtrail:UpdateTrail"],
      "Resource": "*"
    }, {
      "Sid": "DenyLeavingOrganization",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }]
  }'
```

### Step 3: Attach Both to the Workloads OU
Attaching at `Workloads` (not per-sub-OU) means Production and NonProduction inherit identically, and so does any future sub-OU added under Workloads.

```bash
REGION_POLICY_ID=$(aws organizations list-policies --filter SERVICE_CONTROL_POLICY \
  --query "Policies[?Name=='RestrictToApprovedRegions'].Id" --output text)
TRAIL_POLICY_ID=$(aws organizations list-policies --filter SERVICE_CONTROL_POLICY \
  --query "Policies[?Name=='DenyCloudTrailTampering'].Id" --output text)

aws organizations attach-policy --policy-id $REGION_POLICY_ID --target-id $WORKLOADS_OU_ID
aws organizations attach-policy --policy-id $TRAIL_POLICY_ID --target-id $WORKLOADS_OU_ID
```

**Validation checkpoint**: from a principal inside the account now under `NonProduction`, attempt a denied action:
```bash
aws ec2 run-instances --region ap-southeast-2 --image-id ami-0000000000000 --instance-type t3.micro
```
Expect `AccessDenied` citing an explicit deny from a service control policy — even with `AdministratorAccess` locally. The region named in the command was deliberately chosen outside the approved list.

---

## Part 4: Cross-Account Sharing with AWS RAM

AWS RAM lets a central account share a resource — a subnet, a Transit Gateway attachment, a License Manager configuration, a Resolver rule — with other accounts in the Organization without duplicating it per account. Console-first note: enabling organization-wide sharing is a one-time setting most teams configure once via the console during initial Organization setup, but every operational step below (creating shares, associating principals, checking share status) is fully CLI-drivable and is what you'll actually do repeatedly.

### Step 4: Enable Sharing with Your Organization
```bash
aws ram enable-sharing-with-aws-organization
```
This lets a resource share target an entire OU as a principal instead of enumerating individual account IDs — critical for the "acquisitions land in Workloads and inherit shared resources automatically" property this design needs.

### Step 5: Create a Resource Share
This lab shares a subnet from the account's default VPC as a stand-in for the pattern a real deployment would use to share centralized networking (a Transit Gateway attachment, in Lab 5) or a centralized artifact (an AMI, a License Manager configuration) from a network or shared-services account into every workload account.

```bash
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=default-for-az,Values=true" \
  --query "Subnets[0].SubnetId" --output text)

aws ram create-resource-share \
  --name shared-workload-subnet \
  --resource-arns "arn:aws:ec2:<region>:<your-account-id>:subnet/$SUBNET_ID" \
  --principals "$WORKLOADS_OU_ID" \
  --no-allow-external-principals
```
`--no-allow-external-principals` scopes the share to accounts inside your Organization only — an explicit, deliberate choice, not the default you want to discover was wrong later.

**Validation checkpoint**:
```bash
aws ram get-resource-shares --resource-owner SELF --name shared-workload-subnet \
  --query "resourceShares[].{Name:name,Status:status}" --output table
```
Confirm `Status: ACTIVE`. In a real multi-account Organization, any account that lands in the `Workloads` OU (present or future — this is the acquisition-onboarding property from Decision 1) would see this subnet appear automatically in its own EC2 console under **Subnets**, with no manual invitation step required because organization sharing is enabled.

---

## Part 5: Organization-Wide CloudTrail Trail

### Step 6: Create the Log-Delivery Bucket
A real deployment puts this bucket in a dedicated log-archive account (Decision 2) — this lab creates it in the current account and documents the bucket policy that makes centralization work, since the policy shape is identical regardless of which account holds the bucket.

```bash
LOG_BUCKET="sap-c02-lab1-orgtrail-<your-unique-suffix>"
aws s3api create-bucket --bucket $LOG_BUCKET --region us-east-1

ORG_ID=$(aws organizations describe-organization --query "Organization.Id" --output text)

cat > trail-bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::$LOG_BUCKET"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::$LOG_BUCKET/AWSLogs/$ORG_ID/*",
      "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}}
    }
  ]
}
EOF

aws s3api put-bucket-policy --bucket $LOG_BUCKET --policy file://trail-bucket-policy.json
```
The `AWSLogs/$ORG_ID/*` path is not optional — an organization trail delivers every member account's logs under this exact prefix structure, one subfolder per account ID within it.

### Step 7: Create the Organization Trail
Must be run from the Organization's management account.

```bash
aws cloudtrail create-trail \
  --name org-wide-trail \
  --s3-bucket-name $LOG_BUCKET \
  --is-organization-trail \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name org-wide-trail
```

**Validation checkpoint**:
```bash
aws cloudtrail get-trail-status --name org-wide-trail --query "IsLogging"
aws cloudtrail list-trails --query "Trails[?Name=='org-wide-trail']"
```
Confirm `IsLogging: true`. Generate some activity (the denied `run-instances` call from Part 3 already produced one), wait a few minutes, then confirm delivery:
```bash
aws s3 ls s3://$LOG_BUCKET/AWSLogs/$ORG_ID/ --recursive | head
```
Objects appearing under your account's ID within that prefix confirm the pipeline works end to end — this is the single trail every current and future member account's activity lands in, satisfying Decision 2 without per-account setup.

---

## Cleanup
Order matters — undo the trail and RAM share first, detach SCPs before deleting them, move the account out of the OU hierarchy before deleting OUs, delete OUs leaf-to-root.

```bash
# 1. Stop and delete the organization trail
aws cloudtrail stop-logging --name org-wide-trail
aws cloudtrail delete-trail --name org-wide-trail
aws s3 rm s3://$LOG_BUCKET --recursive
aws s3api delete-bucket --bucket $LOG_BUCKET

# 2. Delete the RAM resource share
SHARE_ARN=$(aws ram get-resource-shares --resource-owner SELF --name shared-workload-subnet --query "resourceShares[0].resourceShareArn" --output text)
aws ram delete-resource-share --resource-share-arn $SHARE_ARN

# 3. Detach and delete SCPs
aws organizations detach-policy --policy-id $REGION_POLICY_ID --target-id $WORKLOADS_OU_ID
aws organizations detach-policy --policy-id $TRAIL_POLICY_ID --target-id $WORKLOADS_OU_ID
aws organizations delete-policy --policy-id $REGION_POLICY_ID
aws organizations delete-policy --policy-id $TRAIL_POLICY_ID

# 4. Move the account back to root before OUs can be deleted
aws organizations move-account \
  --account-id <your-account-id> \
  --source-parent-id <nonproduction-ou-id> \
  --destination-parent-id $ROOT_ID

# 5. Delete OUs leaf-to-root — an OU with children or accounts still attached cannot be deleted
NONPROD_OU_ID=<nonproduction-ou-id>
PROD_OU_ID=$(aws organizations list-organizational-units-for-parent --parent-id $WORKLOADS_OU_ID --query "OrganizationalUnits[?Name=='Production'].Id" --output text)
SECURITY_OU_ID=$(aws organizations list-organizational-units-for-parent --parent-id $ROOT_ID --query "OrganizationalUnits[?Name=='Security'].Id" --output text)

aws organizations delete-organizational-unit --organizational-unit-id $NONPROD_OU_ID
aws organizations delete-organizational-unit --organizational-unit-id $PROD_OU_ID
aws organizations delete-organizational-unit --organizational-unit-id $WORKLOADS_OU_ID
aws organizations delete-organizational-unit --organizational-unit-id $SECURITY_OU_ID
```
Confirm with `aws organizations list-organizational-units-for-parent --parent-id $ROOT_ID` (should return empty) and `aws s3 ls | grep sap-c02-lab1` (should return nothing).

If this Organization was created solely for this lab and you don't intend to keep it, leaving it in place costs nothing extra — but AWS Organizations deletion itself requires every member account to first leave or be closed, which is a deliberate, largely irreversible step outside this lab's scope. Don't create a throwaway Organization casually.

---

## Key Concepts

| Term | Definition |
|---|---|
| **AWS Organizations** | The account-management service providing consolidated billing, OU hierarchy, and SCP enforcement across multiple AWS accounts |
| **Organizational Unit (OU)** | A container beneath the Organization root used to group accounts and apply SCPs by inheritance |
| **Service Control Policy (SCP)** | A permission *ceiling* attached to an account or OU — narrows the maximum available permissions for every principal underneath it, IAM policies included; never grants access on its own |
| **AWS RAM (Resource Access Manager)** | Shares a supported resource type from one account into other accounts or OUs without duplicating the resource |
| **Organization trail** | A CloudTrail trail created with `--is-organization-trail`, automatically capturing every current and future member account's activity into one S3 destination |
| **Log-archive account** | A dedicated, workload-free account whose sole purpose is holding centralized audit logs, isolating them from the accounts they audit |
| **Consolidated billing** | The Organizations feature aggregating all member accounts' costs onto the management account's bill, without requiring shared tagging discipline to attribute spend by account |

---

## Common Mistakes
- **Creating an Organization with the default `CONSOLIDATED_BILLING` feature set**: SCPs are unusable until you explicitly create with `--feature-set ALL`, and upgrading later requires member accounts to approve the change
- **Attaching a region-restriction SCP without allow-listing IAM, STS, and other global services**: this locks every account under the OU out of IAM itself, including the ability to fix the mistake
- **Putting the log-archive bucket in the same account as the workloads it audits**: defeats the entire tamper-resistance argument from Decision 2 — a compromised workload account can then delete its own evidence
- **Forgetting `enable-sharing-with-aws-organization` before targeting an OU as a RAM principal**: RAM shares can only target individual account IDs until organization sharing is explicitly enabled
- **Deleting an OU before moving its accounts out**: Organizations refuses to delete any OU that still has accounts or child OUs attached

---

## Next Steps
This design overlaps directly with [SCS-C02 Lab 7: Multi-Account Security & Governance](../SCS-C02/aws-sec-lab-7-multi-account-governance.md) — that lab goes deeper on SCP deny-list vs. allow-list strategy and delegated administration for GuardDuty/Security Hub/Config, which this lab deliberately doesn't re-teach. This lab assumes comfort with the VPC and IAM basics from [SAA-C03 Labs 1–4](../SAA-C03/README.md). Continue to [Lab 2: Data & Storage Database Design](lab-2-data-storage-database-design.md) for the next SAP-C02 design domain.
