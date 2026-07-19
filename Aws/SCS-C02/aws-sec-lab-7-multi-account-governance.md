# AWS Security Lab 7: Multi-Account Security & Governance

Check box if done: []

## Overview
Every lab so far has operated inside one account. Real organizations don't — they run dozens to thousands of accounts, and the security question stops being "is this one account configured correctly" and becomes "can I prove every account is configured correctly, and can I stop an account from ever drifting out of policy in the first place." This lab covers the two tools that answer that at scale: Service Control Policies, which make certain actions structurally impossible account-wide regardless of IAM permissions, and delegated administration, which centralizes CloudTrail, Config, GuardDuty, and Security Hub visibility into one security-tooling account instead of forcing you to check every account individually.

**Estimated time**: 75–90 minutes
**Cost**: ~$0–$1 (SCPs are free; centralized logging/detection services bill the same per-account rates as Labs 2–3, aggregated rather than duplicated in cost)

---

## Scenario
Your company just failed a security audit with a finding that should be embarrassing at this scale: 40 AWS accounts, no organization-wide guardrail preventing someone from disabling CloudTrail in any of them, and a security team that manually logs into each account to check GuardDuty and Security Hub because there's no centralized view. You're building both fixes. **Note up front**: SCPs and delegated administration require AWS Organizations with a real multi-account structure to fully exercise — this lab explains and configures both against that model, and calls out explicitly where a single-account AWS Free Tier setup can validate the mechanics and where it genuinely can't.

---

## Objectives
- Understand AWS Organizations' role as the container for account-wide governance
- Write and attach Service Control Policies using both deny-list and allow-list strategies
- Explain what an SCP can and cannot do (it is not a substitute for IAM)
- Configure a delegated administrator account for centralized CloudTrail, Config, GuardDuty, and Security Hub
- Query a centralized CloudTrail/Config log bucket with Athena to answer a cross-account security question

---

## Part 1: AWS Organizations — The Prerequisite

### Step 1: What You Need for the Full Exercise (and the Fallback If You Don't Have It)
Everything in this lab assumes an AWS Organization with at least a management account and one or more member accounts — this is genuinely required for a real multi-account test; there is no way to meaningfully exercise cross-account SCP enforcement or delegated administration against a single account. **If you're working in a single personal AWS account with no Organization**: follow this lab conceptually — read each SCP and understand exactly what it would block and why, and configure delegated administration's settings against your own account as the "member" to see the configuration surface, understanding that the real value (a security team not needing standing access to every account) only manifests with genuine multiple accounts. This mirrors how [AZ-500](../../Azure/AZ-500/README.md) scopes down sections that need infrastructure most solo learners don't have.

### Step 2: Confirm or Create an Organization
```bash
aws organizations describe-organization
# If this errors with "AWSOrganizationsNotInUseException", create one:
aws organizations create-organization --feature-set ALL
```
`--feature-set ALL` is required for SCPs to be usable at all — the default `CONSOLIDATED_BILLING` feature set doesn't support them.

### Step 3: Understand the Account Structure This Lab Assumes
```
Management Account (root of the Organization)
  ├── Organizational Unit: Security
  │     └── security-tooling-account (delegated admin target — Part 3)
  ├── Organizational Unit: Workloads
  │     ├── prod-account
  │     └── dev-account
```
SCPs attach to OUs (or individual accounts, or the Organization root) and apply to every account beneath that attachment point — this is what makes them scale where per-account IAM policies don't.

---

## Part 2: Service Control Policies — Deny-List vs. Allow-List

### Step 4: The Critical Thing to Understand First — SCPs Are Not IAM
An SCP never *grants* permission — it only ever narrows what's possible, by defining the maximum available permissions for every principal in the account(s) it's attached to, IAM policies included. A user with `AdministratorAccess` in an account whose SCP denies `cloudtrail:StopLogging` genuinely cannot stop CloudTrail, full stop — no IAM policy in that account can override an SCP deny, the same "explicit deny always wins" principle from [Lab 1](aws-sec-lab-1-iam-deep-dive.md), just evaluated one layer higher, before IAM is ever consulted.

### Step 5: Deny-List Strategy — Block Specific Dangerous Actions
The most common real-world starting point: leave everything else alone, explicitly block a short list of high-risk actions.
```bash
aws organizations create-policy \
  --name DenyDisableSecurityServices \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyDisablingCloudTrail",
        "Effect": "Deny",
        "Action": ["cloudtrail:StopLogging", "cloudtrail:DeleteTrail", "cloudtrail:UpdateTrail"],
        "Resource": "*"
      },
      {
        "Sid": "DenyDisablingGuardDuty",
        "Effect": "Deny",
        "Action": ["guardduty:DeleteDetector", "guardduty:DisassociateFromMasterAccount", "guardduty:UpdateDetector"],
        "Resource": "*",
        "Condition": {"StringNotLike": {"aws:PrincipalArn": "arn:aws:iam::*:role/OrganizationAccountAccessRole"}}
      },
      {
        "Sid": "DenyLeavingOrganization",
        "Effect": "Deny",
        "Action": "organizations:LeaveOrganization",
        "Resource": "*"
      }
    ]
  }'
```
The `Condition` on the GuardDuty statement is worth noting: it's an example of carving out a narrow, documented exception (here, the Organizations management role) rather than blocking the action so absolutely that legitimate administrative work becomes impossible — a common SCP design mistake is being so restrictive that break-glass administration itself gets blocked.

### Step 6: Attach the Policy to an OU
```bash
POLICY_ID=$(aws organizations list-policies --filter SERVICE_CONTROL_POLICY --query "Policies[?Name=='DenyDisableSecurityServices'].Id" --output text)
OU_ID=$(aws organizations list-organizational-units-for-parent --parent-id <root-id> --query "OrganizationalUnits[?Name=='Workloads'].Id" --output text)

aws organizations attach-policy --policy-id $POLICY_ID --target-id $OU_ID
```

### Step 7: Allow-List Strategy — Restrict to an Approved Region Set
A stricter model for regulated workloads: deny everything *except* an explicitly approved list, rather than trying to enumerate every dangerous action individually.
```bash
aws organizations create-policy \
  --name RestrictToApprovedRegions \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "DenyOutsideApprovedRegions",
      "Effect": "Deny",
      "NotAction": ["iam:*", "organizations:*", "sts:*", "support:*", "cloudfront:*", "route53:*"],
      "Resource": "*",
      "Condition": {"StringNotEquals": {"aws:RequestedRegion": ["us-east-1", "us-west-2"]}}
    }]
  }'
```
`NotAction` combined with `Deny` is the mechanic that makes this an allow-list in practice: it denies every action *except* the ones named (which are global/non-regional services that shouldn't be region-restricted in the first place), and only when the request targets a region outside the approved list.

| | Deny-list SCP | Allow-list SCP |
|---|---|---|
| **Default posture** | Everything allowed except explicitly denied actions | Everything denied except explicitly permitted scope |
| **Maintenance burden** | Lower — add new denies as new risks are identified | Higher — every legitimate new service/region needs an explicit addition or it's blocked |
| **Best fit** | Most organizations, most accounts — guardrails against known-bad actions | Regulated/high-security environments (data residency, restricted service catalogs) |
| **Failure mode if incomplete** | A dangerous action nobody thought to deny is still possible | A legitimate action nobody thought to allow breaks unexpectedly |

### Step 8: Validate the Guardrail Actually Blocks the Action
From a principal in an account under the `Workloads` OU, attempt the denied action:
```bash
aws cloudtrail stop-logging --name <any-trail-name> --profile workload-account-test
```
**Expected result**: `AccessDeniedException`, citing an explicit deny from a service control policy — even if the calling principal has `AdministratorAccess` in that account's own IAM. This is the proof that matters: an SCP-backed guardrail holds regardless of what IAM permissions exist locally in the account.

---

## Part 3: Delegated Administration — Centralizing Security Visibility

### Step 9: Why Delegated Administration Beats "Log Into Every Account"
Without delegation, a security analyst either needs standing IAM access in every single account (a huge blast radius if that analyst's credentials are ever compromised) or has to assume a role into each account one at a time to check GuardDuty/Security Hub/Config — neither scales past a handful of accounts. **Delegated administration** designates one account (typically a dedicated security-tooling account, never the Organizations management account itself) as the aggregation point — findings and configuration data flow to it automatically from every member account.

### Step 10: Designate the Delegated Administrator for GuardDuty
From the Organizations management account:
```bash
aws guardduty enable-organization-admin-account --admin-account-id <security-tooling-account-id>
```
From that point forward, GuardDuty findings across every member account in the Organization appear in the delegated admin account's GuardDuty console — no per-account login required.

### Step 11: Repeat for Security Hub, Config, and CloudTrail
```bash
aws securityhub enable-organization-admin-account --admin-account-id <security-tooling-account-id>

aws configservice put-organization-conformance-pack \
  --organization-conformance-pack-name org-baseline \
  --template-s3-uri s3://<config-templates-bucket>/baseline-conformance-pack.yaml

# CloudTrail: create an organization trail from the management account instead of delegating per-account
aws cloudtrail create-trail \
  --name org-wide-trail \
  --s3-bucket-name <centralized-cloudtrail-bucket> \
  --is-organization-trail \
  --is-multi-region-trail \
  --enable-log-file-validation
```
**Note the asymmetry**: GuardDuty, Security Hub, and Config support true delegated administration (a member account promoted to see everyone's findings). CloudTrail instead uses `--is-organization-trail`, created once from the management account, which automatically captures every member account's activity into one centralized bucket — a different mechanism reaching a similar centralization goal.

### Step 12: Auto-Enable for New Accounts
The point of centralization breaks the moment someone creates account #41 and forgets to enroll it. Configure auto-enrollment so new accounts are covered automatically:
```bash
aws guardduty update-organization-configuration \
  --detector-id <detector-id-in-delegated-admin-account> \
  --auto-enable-organization-members NEW
```
`NEW` enrolls only future accounts automatically (existing accounts must be added explicitly once) — review the alternative `ALL` setting if you want every current and future account enrolled without exception, understanding it will also affect existing accounts you may not have reviewed yet.

### Step 13: Validate Centralization From the Delegated Admin Account
From the `security-tooling-account`:
```bash
aws guardduty list-members --detector-id <detector-id> --query "Members[*].{Account:AccountId,Status:RelationshipStatus}"
```
**Expected result**: every enrolled member account listed with `RelationshipStatus: Enabled` — this list, checked from one account, is the entire point of this section. Compare this against the audit finding from the scenario: instead of 40 logins, this is one query.

---

## Part 4: Athena Over Centralized Logs — Answering a Cross-Account Question

### Step 14: The Kind of Question This Solves
"Which accounts had an IAM policy change in the last 7 days that added `*` to a `Resource` field?" is unanswerable by checking accounts one at a time in any reasonable timeframe — but it's a single Athena query against a centralized CloudTrail bucket populated by the organization trail from Step 11.

### Step 15: Set Up the Athena Table Against the Organization Trail's Bucket
```sql
CREATE EXTERNAL TABLE org_cloudtrail_logs (
  eventVersion STRING, eventTime STRING, eventSource STRING, eventName STRING,
  awsRegion STRING, sourceIPAddress STRING, userIdentity STRUCT<
    type: STRING, arn: STRING, accountId: STRING, userName: STRING>,
  requestParameters STRING, responseElements STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION 's3://<centralized-cloudtrail-bucket>/AWSLogs/<org-id>/';
```

### Step 16: Run the Cross-Account Query
```sql
SELECT userIdentity.accountId, eventTime, userIdentity.arn, eventName
FROM org_cloudtrail_logs
WHERE eventName IN ('PutUserPolicy', 'PutRolePolicy', 'AttachUserPolicy', 'AttachRolePolicy')
  AND eventTime > date_format(current_date - interval '7' day, '%Y-%m-%dT%H:%i:%sZ')
ORDER BY eventTime DESC;
```

**Validation checkpoint**: this query should return results spanning multiple `accountId` values if more than one account in the Organization has had recent IAM policy activity — the cross-account column is the proof this is genuinely centralized, not just one account's logs. Compare this against Lab 2's single-account CloudTrail query: same underlying data source, but this one answers a question no single-account trail could.

---

## Part 5: Cleanup

```bash
# Detach and delete SCPs
aws organizations detach-policy --policy-id $POLICY_ID --target-id $OU_ID
aws organizations delete-policy --policy-id $POLICY_ID

# Disable delegated administration (from the management account)
aws guardduty disable-organization-admin-account --admin-account-id <security-tooling-account-id>
aws securityhub disable-organization-admin-account --admin-account-id <security-tooling-account-id>

# Disable the organization trail
aws cloudtrail stop-logging --name org-wide-trail
aws cloudtrail delete-trail --name org-wide-trail

aws s3 rm s3://<centralized-cloudtrail-bucket> --recursive
aws s3 rb s3://<centralized-cloudtrail-bucket>
```

> **Important**: if this was a disposable test Organization, leaving member accounts costs nothing extra beyond their own resource usage — but confirm SCPs are detached before deleting an OU, and confirm delegated administration is explicitly disabled rather than assuming account deletion cleans it up.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **SCPs as a permission ceiling, not a grant** | Enforces guardrails that hold regardless of what IAM permissions exist in any individual account |
| **Deny-list vs. allow-list SCP strategy** | Matches the guardrail's strictness to the actual risk/compliance posture instead of defaulting to one pattern everywhere |
| **Delegated administration for GuardDuty/Security Hub/Config** | Centralizes security visibility without granting a security team standing access to every account |
| **Organization CloudTrail trail** | Guarantees every member account is audited by default, closing the "account #41 got missed" gap |
| **Athena over centralized logs** | Answers cross-account questions that are structurally impossible to answer by checking accounts individually |

---

## Common Mistakes to Avoid
- **Attaching an SCP directly to the Organization root without testing on a single OU first**: a misconfigured deny-all-by-mistake SCP at the root can lock out every account in the Organization simultaneously, including the management account's own workloads in some configurations
- **Forgetting SCPs never grant permissions**: a common early mistake is expecting an SCP allow statement to grant access — it only ever narrows what IAM can separately grant
- **Not carving out a break-glass exception in a restrictive SCP**: an SCP that blocks even legitimate emergency administrative action is a guardrail that creates its own incident
- **Delegating administration to the Organizations management account itself**: best practice is a dedicated security-tooling account — the management account is uniquely privileged and should have minimal standing tooling
- **Enabling `auto-enable-organization-members: ALL` without reviewing existing accounts first**: this immediately changes detection/aggregation behavior for every current account, not just new ones — understand the blast radius before flipping it

---

## Next Steps
- Continue to [Lab 8: Incident Response Capstone](aws-sec-lab-8-incident-response-capstone.md), which assumes the centralized detection and governance this lab built, and walks a full detection-to-remediation incident across it
- If you have a real Organization, add a Config conformance pack enforcing this lab's SCPs' intent as a detective control too — an SCP prevents the action, a Config rule proves the preventive control itself hasn't drifted
- Build a Security Hub central configuration policy (organization-wide standard enablement) so every member account gets the same Security Hub standards without per-account setup
- Compare this SCP model directly against Azure Policy and Management Group-scoped initiatives from [AZ-500](../../Azure/AZ-500/README.md) — both are permission-ceiling/guardrail mechanisms above IAM/RBAC, applied at a container level rather than per-resource
