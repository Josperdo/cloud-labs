# AWS Security Lab 1: IAM Deep Dive

Check box if done: []

## Overview
Most "access denied" tickets, and most SCS-C02 exam questions, come down to one thing: not understanding how AWS actually evaluates a permission request when multiple policies apply at once. This lab builds every layer that participates in that evaluation — identity policies, resource policies, permission boundaries, and cross-account roles — and then deliberately creates a conflict between them so you can trace exactly which one wins and why.

**Estimated time**: 60–90 minutes
**Cost**: ~$0 (IAM has no charge; the S3 bucket used for resource-policy testing costs fractions of a cent)

---

## Scenario
A developer on your team is confused: their IAM user has a policy that should allow full S3 access, but they're still getting `AccessDenied` on a specific bucket. Separately, you've been asked to let a contractor create IAM users for their own onboarding — but never anything more privileged than that, no matter what policy they try to attach to themselves. You're solving both, and in the process building the mental model for how AWS actually decides allow vs. deny.

---

## Objectives
- Understand and demonstrate the evaluation order: explicit deny → organization SCPs → resource policy → identity policy → permission boundary
- Reproduce and explain a real "identity policy allows, resource policy doesn't" conflict
- Build a permission boundary that caps what a delegated administrator can grant themselves
- Use IAM Policy Simulator to test a policy before attaching it
- Configure a cross-account role with a least-privilege trust policy

---

## Part 1: Identity Policy vs. Resource Policy — The Conflict That Confuses Everyone

### Step 1: Create a Test User and an S3 Bucket

```bash
aws iam create-user --user-name iam-lab-dev

aws iam create-policy \
  --policy-name FullS3Access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    }]
  }'

aws iam attach-user-policy \
  --user-name iam-lab-dev \
  --policy-arn arn:aws:iam::<your-account-id>:policy/FullS3Access

aws s3api create-bucket --bucket iam-lab-bucket-$RANDOM --region us-east-1
BUCKET_NAME=$(aws s3api list-buckets --query "Buckets[?starts_with(Name, 'iam-lab-bucket')].Name" --output text)
```

`iam-lab-dev`'s **identity policy** now grants `s3:*` on every S3 resource in the account, unconditionally.

### Step 2: Attach a Resource Policy That Explicitly Denies This User
```bash
aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": {"AWS": "arn:aws:iam::<your-account-id>:user/iam-lab-dev"},
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::'"$BUCKET_NAME"'",
      "arn:aws:s3:::'"$BUCKET_NAME"'/*"
    ]
  }]
}'
```

### Step 3: Test as the User and Observe
```bash
# Configure a profile using iam-lab-dev's access keys (create them first if needed)
aws iam create-access-key --user-name iam-lab-dev
# Configure a named profile "iam-lab-dev" with the returned keys

aws s3 ls s3://$BUCKET_NAME --profile iam-lab-dev
```

**Expected result**: `AccessDenied`, despite the identity policy granting `s3:*` on `*`.

**Why**: AWS evaluates **every applicable policy** for a request — identity policies attached to the principal, and resource policies attached to the resource being accessed — and combines them. **An explicit `Deny` anywhere in that set always wins**, regardless of how permissive any `Allow` elsewhere is. This is the single most important rule in IAM policy evaluation, and it's exactly the scenario the developer in this lab's opening scenario hit.

**Validation checkpoint**: Remove the bucket policy (`aws s3api delete-bucket-policy --bucket $BUCKET_NAME`) and retry the same `aws s3 ls` — it should now succeed, proving the resource policy's explicit deny was the only thing blocking access.

---

## Part 2: Permission Boundaries — Capping What a Delegated Admin Can Grant

### Step 4: The Problem Permission Boundaries Solve
Suppose you delegate "create and manage IAM users" to a contractor. Even with a narrowly-worded identity policy, nothing stops that contractor from creating a new IAM user and attaching `AdministratorAccess` to it — privilege escalation through the exact permission you gave them. A **permission boundary** caps the *maximum* permissions any user or role the contractor creates can ever have, regardless of what identity policy gets attached later.

### Step 5: Create the Boundary Policy
```bash
aws iam create-policy \
  --policy-name DeveloperBoundary \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:*", "dynamodb:*", "logs:*"],
        "Resource": "*"
      },
      {
        "Effect": "Deny",
        "Action": ["iam:*", "organizations:*"],
        "Resource": "*"
      }
    ]
  }'
```

This boundary caps the *ceiling*: even if someone later attaches an identity policy granting `iam:*`, the boundary's explicit deny on `iam:*` still applies — a boundary is evaluated as one more input into the same "explicit deny always wins" logic from Part 1.

### Step 6: Create a Contractor Role That Can Only Create Boundary-Constrained Users
```bash
aws iam create-policy \
  --policy-name ContractorCanCreateUsers \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["iam:CreateUser", "iam:AttachUserPolicy"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::<your-account-id>:policy/DeveloperBoundary"
        }
      }
    }]
  }'
```

The `Condition` block is what makes this work: the contractor can only create IAM users **if** that user's permission boundary is set to exactly `DeveloperBoundary` — attempting to create a user without a boundary, or with a different (weaker) one, is denied by AWS's policy evaluation before the request even completes.

### Step 7: Validate the Escalation Path Is Actually Closed
Attach `ContractorCanCreateUsers` to a test principal, then attempt (as that principal) to create a new IAM user **without** the boundary condition satisfied:
```bash
aws iam create-user --user-name should-fail --profile contractor-test
```
**Expected result**: `AccessDenied` — the condition on `iam:PermissionsBoundary` isn't met. Now retry, correctly specifying the boundary:
```bash
aws iam create-user --user-name should-succeed \
  --permissions-boundary arn:aws:iam::<your-account-id>:policy/DeveloperBoundary \
  --profile contractor-test
```
This should succeed — and the resulting user, no matter what identity policy is later attached to it, can never exceed `DeveloperBoundary`'s permissions.

---

## Part 3: IAM Policy Simulator — Test Before You Attach

### Step 8: Simulate a Policy Change Before Applying It
Before attaching a new or modified policy in production, validate it against specific actions and resources:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<your-account-id>:user/iam-lab-dev \
  --action-names s3:DeleteBucket s3:GetObject \
  --resource-arns arn:aws:s3:::$BUCKET_NAME
```

**Validation checkpoint**: review the `EvalDecision` field for each action — this tells you definitively whether the combination of policies would allow or deny each specific action, without ever making the real API call. This is the tool to reach for before attaching any policy you're not 100% sure about, rather than trial-and-error against production.

---

## Part 4: Cross-Account Roles — Least-Privilege Trust Policies

### Step 9: Why Cross-Account Roles Beat Shared Credentials
If a second AWS account (a partner, a separate environment) needs access to resources in this account, the anti-pattern is creating an IAM user with access keys and handing them over — keys that can leak, don't expire on their own, and aren't easy to audit. The correct pattern is a role the other account's principals can **assume**, with a trust policy that scopes exactly who can assume it.

### Step 10: Create a Role with a Scoped Trust Policy
```bash
aws iam create-role \
  --role-name CrossAccountReadOnly \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::<TRUSTED-ACCOUNT-ID>:root"},
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {"sts:ExternalId": "shared-secret-value"}
      }
    }]
  }'

aws iam attach-role-policy \
  --role-name CrossAccountReadOnly \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

**Why the `ExternalId` condition?** Without it, this trust policy would let *any* principal in the trusted account assume the role — including ones that account's own admins never intended to grant this access to (the "confused deputy" problem, common when the trusted account itself delegates broadly). The external ID is a shared secret both sides agree on, adding a second factor to the trust relationship.

### Step 11: Validate the Least-Privilege Boundary
Confirm the attached policy is `ReadOnlyAccess`, not something broader — a common mistake is provisioning a working cross-account role quickly with `AdministratorAccess` "to get it working" and never tightening it afterward:
```bash
aws iam list-attached-role-policies --role-name CrossAccountReadOnly
```

---

## Part 5: Cleanup

```bash
aws iam detach-user-policy --user-name iam-lab-dev --policy-arn arn:aws:iam::<your-account-id>:policy/FullS3Access
aws iam delete-access-key --user-name iam-lab-dev --access-key-id <key-id-from-step-3>
aws iam delete-user --user-name iam-lab-dev

aws s3 rb s3://$BUCKET_NAME --force

aws iam delete-role --role-name CrossAccountReadOnly 2>/dev/null || \
  (aws iam detach-role-policy --role-name CrossAccountReadOnly --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess && \
   aws iam delete-role --role-name CrossAccountReadOnly)

aws iam delete-policy --policy-arn arn:aws:iam::<your-account-id>:policy/FullS3Access
aws iam delete-policy --policy-arn arn:aws:iam::<your-account-id>:policy/DeveloperBoundary
aws iam delete-policy --policy-arn arn:aws:iam::<your-account-id>:policy/ContractorCanCreateUsers
```

---

## Key Concepts to Understand

| Concept | Definition |
|---------|-----------|
| **Policy evaluation order** | Explicit deny (anywhere) always wins; otherwise, an explicit allow in *any* applicable policy (identity, resource, boundary, SCP) is required, and absence of any allow is an implicit deny |
| **Identity policy vs. resource policy** | Identity policies attach to a principal (user/role) and say what *it* can do; resource policies attach to a resource and say who can access *it* — both must permit the action |
| **Permission boundary** | Caps the maximum permissions a principal can ever have, regardless of what identity policies are attached later — used to prevent privilege escalation by delegated administrators |
| **Trust policy vs. permissions policy on a role** | The trust policy controls *who can assume* the role; the permissions policy controls *what the role can do once assumed* — two different documents with two different jobs |
| **`sts:ExternalId`** | A shared-secret condition on a trust policy that mitigates the confused-deputy problem in cross-account role assumption |

---

## Common Mistakes to Avoid
- **Assuming a broad identity policy is enough**: a resource policy's explicit deny overrides it every time — always check both when troubleshooting unexpected `AccessDenied` errors
- **Attaching a permission boundary without also restricting who can create/modify boundaries**: if a contractor can edit or remove the boundary itself, it provides no real ceiling
- **Skipping `sts:ExternalId` on a cross-account trust policy**: opens the door to the confused-deputy problem where an unintended principal in the trusted account assumes your role
- **Attaching `AdministratorAccess` to get a cross-account integration "working" and never revisiting it**: the single most common real-world IAM security gap found in cloud security audits

---

## Next Steps
- Enable IAM Access Analyzer and review its findings on unused permissions across the resources touched in this lab
- Continue to [Lab 2: Detection & Monitoring](aws-sec-lab-2-detection-monitoring.md) to detect exactly this kind of over-privileged access at scale, not just one policy at a time
- Compare this IAM evaluation model directly against Azure RBAC's additive-only model (no explicit deny by default) from [AZ-500 Lab 1](../../Azure/AZ-500/lab-1-identity-access-protection.md) — a common, high-value interview comparison question
