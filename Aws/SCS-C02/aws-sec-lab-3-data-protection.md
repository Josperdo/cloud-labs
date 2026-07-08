# AWS Security Lab 3: Data Protection

Check box if done: []

## Overview
"Is it encrypted" is the wrong first question — everything in AWS *can* be encrypted with a checkbox. The real questions are: who controls the key, can you prove access is actually restricted, and where do application secrets live so they're never in a config file. This lab builds a customer-managed KMS key with a real key policy, locks an S3 bucket down completely, and moves a secret out of application config into Secrets Manager with automatic rotation.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$1 (a KMS customer-managed key costs $1/month, prorated for the lab's duration; Secrets Manager costs $0.40/month per secret, also prorated — both are fractions of a cent for a same-day lab)

---

## Scenario
A compliance requirement landed on your desk: prove that a specific S3 bucket's data is encrypted with a key your organization controls (not AWS's default key), prove that bucket is unreachable from the public internet under any circumstance, and eliminate a hardcoded database password sitting in an application's environment variables. Three concrete, provable controls — not just "encryption is enabled."

---

## Objectives
- Create a customer-managed KMS key with a key policy that restricts usage to specific principals
- Encrypt an S3 bucket with that key and prove AWS-managed defaults aren't in play
- Configure S3 Block Public Access at the account and bucket level, and understand why both layers matter
- Move a secret into Secrets Manager and enable automatic rotation
- Verify a principal without key access genuinely cannot decrypt the data

---

## Part 1: Customer-Managed KMS Keys

### Step 1: Why Customer-Managed Over AWS-Managed
AWS-managed keys (`aws/s3`, etc.) are free and encrypt data by default, but you don't control their key policy, can't restrict which principals can use them beyond IAM permissions alone, and can't enable detailed key usage logging as precisely. A **customer-managed key (CMK)** gives you a key policy — a resource-based policy on the key itself, evaluated the same "explicit deny always wins" way as the S3 bucket policy in [Lab 1](aws-sec-lab-1-iam-deep-dive.md) — plus full control over rotation and deletion.

### Step 2: Create the Key with a Scoped Key Policy
```bash
MY_ARN=$(aws sts get-caller-identity --query Arn --output text)

aws kms create-key \
  --description "data-protection-lab-key" \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowRootAccountAdmin",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::<your-account-id>:root"},
        "Action": "kms:*",
        "Resource": "*"
      },
      {
        "Sid": "AllowSpecificPrincipalToUseKey",
        "Effect": "Allow",
        "Principal": {"AWS": "'"$MY_ARN"'"},
        "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey"],
        "Resource": "*"
      }
    ]
  }'

KEY_ID=$(aws kms list-keys --query "Keys[-1].KeyId" --output text)
aws kms create-alias --alias-name alias/data-protection-lab --target-key-id $KEY_ID
```

**Note what's absent**: unlike the AWS-managed key, this policy names exactly one principal (you) as allowed to encrypt/decrypt with it. Anyone else in the account — even an administrator with broad IAM permissions — is denied by the key policy unless explicitly added here, the same explicit-allow-required logic from Lab 1's resource policy discussion.

### Step 3: Enable Automatic Key Rotation
```bash
aws kms enable-key-rotation --key-id $KEY_ID
```
AWS rotates the underlying key material annually while keeping the same key ID/ARN — nothing referencing this key needs to change when rotation happens.

---

## Part 2: Encrypt S3 with the Customer-Managed Key

### Step 4: Create and Encrypt the Bucket
```bash
aws s3api create-bucket --bucket data-protection-lab-$RANDOM --region us-east-1
BUCKET_NAME=$(aws s3api list-buckets --query "Buckets[?starts_with(Name, 'data-protection-lab')].Name" --output text)

aws s3api put-bucket-encryption --bucket $BUCKET_NAME --server-side-encryption-configuration '{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "'"$KEY_ID"'"
    },
    "BucketKeyEnabled": true
  }]
}'
```

`BucketKeyEnabled: true` reduces KMS API call volume (and cost) for high-throughput buckets by caching a bucket-level data key briefly instead of calling KMS on every single object operation — worth knowing as a cost/performance detail, not just a security one.

### Step 5: Upload and Verify
```bash
echo "sensitive compliance data" > /tmp/testfile.txt
aws s3 cp /tmp/testfile.txt s3://$BUCKET_NAME/

aws s3api head-object --bucket $BUCKET_NAME --key testfile.txt --query "{Algorithm: ServerSideEncryption, KeyId: SSEKMSKeyId}"
```

**Validation checkpoint**: confirm `SSEKMSKeyId` matches your customer-managed key's ARN, not an `aws/s3`-prefixed AWS-managed key. This is the concrete proof the compliance requirement from the scenario asked for.

---

## Part 3: S3 Block Public Access — Two Layers

### Step 6: Understand Why Both Account and Bucket Level Matter
S3 Block Public Access can be set at the **account level** (applies to every bucket, current and future) and the **bucket level** (applies to just that bucket). Setting it only at the bucket level leaves every *other* bucket in the account without the protection — and leaves the door open for a future bucket to be created without it by default.

### Step 7: Enable Both Layers
```bash
# Account level — blocks public access for every bucket in this account
aws s3control put-public-access-block \
  --account-id <your-account-id> \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Bucket level — belt and suspenders, and what a bucket-specific compliance check will look for
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### Step 8: Prove It — Attempt to Make the Bucket Public and Watch It Fail
```bash
aws s3api put-bucket-acl --bucket $BUCKET_NAME --acl public-read
```
**Expected result**: `AccessDenied` — Block Public Access rejects the request outright, rather than relying on nobody ever making this mistake. This is a stronger control than "we have a policy against public buckets" — it's structurally enforced.

---

## Part 4: Secrets Manager — Getting the Password Out of Config

### Step 9: Store a Secret
```bash
aws secretsmanager create-secret \
  --name app/database-credentials \
  --description "Database credentials for the lab app" \
  --secret-string '{"username":"appuser","password":"CHANGE_ME_INITIAL_VALUE"}'
```

### Step 10: Retrieve It the Way an Application Would
```bash
aws secretsmanager get-secret-value \
  --secret-id app/database-credentials \
  --query SecretString --output text
```
In real application code, this would be an SDK call (`boto3.client('secretsmanager').get_secret_value(...)`) made at runtime using the compute resource's IAM role — never a value baked into a Docker image, environment variable file, or committed config.

### Step 11: Enable Automatic Rotation
Automatic rotation requires a Lambda function that knows how to actually change the credential on the target system (e.g., calling `ALTER USER` on an RDS database) — AWS provides templates for common database engines. For this lab, demonstrate the configuration concept:

```bash
# In practice, you'd deploy AWS's provided rotation Lambda template first, then reference its ARN here:
aws secretsmanager rotate-secret \
  --secret-id app/database-credentials \
  --rotation-lambda-arn <rotation-lambda-arn> \
  --rotation-rules AutomaticallyAfterDays=30
```

**Why this matters even if you don't wire up a real rotation Lambda in this lab**: a secret that's never rotated is functionally similar to a hardcoded credential — if it ever leaks, it remains valid indefinitely. Automatic rotation on a schedule bounds how long a leaked credential stays useful, without requiring anyone to remember to do it manually.

### Step 12: Restrict Who Can Read the Secret
```bash
aws secretsmanager put-resource-policy \
  --secret-id app/database-credentials \
  --resource-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "'"$MY_ARN"'"},
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*"
    }]
  }'
```

Same pattern as the KMS key policy in Part 1 — a resource policy naming exactly who can read this specific secret, independent of broader IAM permissions any principal might otherwise have.

---

## Part 5: Cleanup

```bash
aws secretsmanager delete-secret --secret-id app/database-credentials --force-delete-without-recovery

aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME

# KMS keys can't be deleted immediately — schedule deletion (minimum 7-day waiting period)
aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
```

> **Note**: The KMS key enters a "pending deletion" state for 7 days rather than being removed immediately — this is a deliberate AWS safeguard against accidental key deletion, since a deleted key makes any data encrypted with it permanently unrecoverable. It stops incurring the $1/month charge once deletion completes.

---

## Key Concepts to Understand

| Concept | Definition |
|---------|-----------|
| **AWS-managed key vs. customer-managed key (CMK)** | AWS-managed keys are free with no custom key policy; CMKs cost $1/month but give you full key policy control, rotation control, and audit granularity |
| **Key policy** | A resource-based policy on the KMS key itself — evaluated with the same explicit-deny-wins logic as any other resource policy |
| **Account-level vs. bucket-level Block Public Access** | Account-level protects every bucket including future ones; bucket-level is what a per-bucket compliance check inspects — enable both |
| **Secrets Manager resource policy** | Restricts which principals can retrieve a specific secret's value, independent of their broader IAM permissions |
| **Automatic secret rotation** | Bounds the lifetime of a leaked credential's usefulness by changing it on a schedule, without manual intervention |

---

## Common Mistakes to Avoid
- **Relying on AWS-managed keys when a compliance requirement specifies customer-controlled encryption**: `aws/s3` and similar AWS-managed keys don't satisfy "customer-managed key" requirements even though the data is genuinely encrypted
- **Enabling Block Public Access at only one level (account or bucket)**: leaves a gap — enable both, always
- **Storing secrets in environment variables or `.env` files instead of Secrets Manager**: the exact anti-pattern this lab replaces; env vars are visible to anything with process-list or container-inspect access
- **Never rotating a secret after creating it**: a "protected" secret that's five years old and never rotated has the same practical risk profile as a hardcoded one, if it was ever exposed at any point in those five years

---

## Next Steps
- Deploy one of AWS's provided Lambda rotation templates for RDS MySQL/PostgreSQL and complete a real end-to-end rotation
- Add a Config rule (from [Lab 2](aws-sec-lab-2-detection-monitoring.md)) that flags any S3 bucket *not* using a customer-managed KMS key
- Continue to [Lab 4: Incident Response Automation](aws-sec-lab-4-incident-response-automation.md) to auto-remediate a bucket that drifts away from these controls, instead of relying on manual review to catch it
