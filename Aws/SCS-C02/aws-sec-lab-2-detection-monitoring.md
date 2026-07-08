# AWS Security Lab 2: Detection & Monitoring

Check box if done: []

## Overview
AWS's detection stack has three services that people often assume are redundant — they're not, they answer different questions. CloudTrail answers "what API call happened and who made it." GuardDuty answers "does this activity look malicious." AWS Config answers "is this resource configured the way it should be, right now, and was it always." This lab wires up all three, plus a Config rule with automatic evaluation, and generates real findings you can walk through rather than just reading about.

**Estimated time**: 60–90 minutes
**Cost**: ~$0–$1 (GuardDuty has a 30-day free trial per account; Config bills per configuration item recorded — small for a single lab, but not free)

---

## Scenario
Your org just failed a security review with one finding repeated three times: "no centralized audit log, no threat detection, no automated configuration compliance checking." You're standing up all three in one pass, and then proving each one actually works by generating a real finding for it to catch.

---

## Objectives
- Enable a CloudTrail trail with log file validation and understand what it does and doesn't capture
- Enable GuardDuty and understand its finding categories
- Enable AWS Config with a managed rule and trigger a real non-compliant finding
- Aggregate GuardDuty and Config findings into Security Hub for a single pane of glass
- Understand which of the three services to reach for, for a given question

---

## Part 1: CloudTrail — The Audit Log Foundation

### Step 1: Create a Trail
```bash
aws s3api create-bucket --bucket cloudtrail-logs-$RANDOM --region us-east-1
BUCKET_NAME=$(aws s3api list-buckets --query "Buckets[?starts_with(Name, 'cloudtrail-logs')].Name" --output text)

aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::'"$BUCKET_NAME"'"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::'"$BUCKET_NAME"'/AWSLogs/<your-account-id>/*",
      "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}}
    }
  ]
}'

aws cloudtrail create-trail \
  --name org-audit-trail \
  --s3-bucket-name $BUCKET_NAME \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name org-audit-trail
```

**Why multi-region?** An attacker who compromises credentials often pivots to a region your team doesn't normally operate in, specifically because activity there is less likely to be watched. A single-region trail creates exactly that blind spot.

**Why log file validation?** It generates a cryptographic digest for each batch of log files, so tampering with a log after the fact (an attacker covering their tracks) is detectable — without it, CloudTrail logs are just files in a bucket with no integrity guarantee.

### Step 2: Generate and Find a Real Event
```bash
aws iam create-user --user-name cloudtrail-test-user
aws iam delete-user --user-name cloudtrail-test-user
```

Wait 5–15 minutes for delivery, then:
```bash
aws s3 ls s3://$BUCKET_NAME/AWSLogs/<your-account-id>/CloudTrail/ --recursive | tail -5
```

**Validation checkpoint**: Download the most recent log file and confirm it contains both the `CreateUser` and `DeleteUser` events, with the calling principal's identity, source IP, and timestamp — this is the raw material every other detection tool in this lab is built on top of.

---

## Part 2: GuardDuty — Threat Detection

### Step 3: Enable GuardDuty
```bash
aws guardduty create-detector --enable
DETECTOR_ID=$(aws guardduty list-detectors --query "DetectorIds[0]" --output text)
```

GuardDuty analyzes CloudTrail management events, VPC Flow Logs, and DNS logs continuously — you don't feed it anything manually; it's already consuming the same event stream CloudTrail (Part 1) produces, plus network-layer signals CloudTrail never sees.

### Step 4: Generate a Sample Finding
Real GuardDuty findings take time to develop from genuine suspicious activity — for a lab, use AWS's built-in sample finding generator instead of waiting:
```bash
aws guardduty create-sample-findings \
  --detector-id $DETECTOR_ID \
  --finding-types "Recon:IAMUser/MaliciousIPCaller" "UnauthorizedAccess:EC2/SSHBruteForce"
```

### Step 5: Review the Findings
```bash
aws guardduty list-findings --detector-id $DETECTOR_ID --query "FindingIds" --output text
```

1. **GuardDuty** → **Findings** in the console
2. Open one finding — note the **Severity** (Low/Medium/High), the **resource affected**, and the **finding type** naming convention: `ThreatPurpose:ResourceType/ThreatFamily` (e.g., `Recon:IAMUser/MaliciousIPCaller` reads as "reconnaissance activity, against an IAM user, from a known-malicious IP")

**Validation checkpoint**: Confirm you can explain, in your own words, what real-world activity each sample finding type represents — this naming convention itself is worth internalizing for the exam.

---

## Part 3: AWS Config — Continuous Compliance

### Step 6: Enable Config
```bash
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::<your-account-id>:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

aws s3api create-bucket --bucket config-logs-$RANDOM --region us-east-1
CONFIG_BUCKET=$(aws s3api list-buckets --query "Buckets[?starts_with(Name, 'config-logs')].Name" --output text)

aws configservice put-delivery-channel \
  --delivery-channel name=default,s3BucketName=$CONFIG_BUCKET

aws configservice start-configuration-recorder --configuration-recorder-name default
```

> If the service-linked role doesn't exist yet, create it first: `aws iam create-service-linked-role --aws-service-name config.amazonaws.com`

### Step 7: Add a Managed Rule
```bash
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "s3-bucket-public-read-prohibited",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }
}'
```

### Step 8: Generate a Real Non-Compliant Finding
```bash
aws s3api create-bucket --bucket config-test-public-$RANDOM --region us-east-1
TEST_BUCKET=$(aws s3api list-buckets --query "Buckets[?starts_with(Name, 'config-test-public')].Name" --output text)

# Intentionally misconfigure it
aws s3api put-public-access-block --bucket $TEST_BUCKET \
  --public-access-block-configuration BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false

aws s3api put-bucket-acl --bucket $TEST_BUCKET --acl public-read
```

### Step 9: Watch Config Catch It
Wait a few minutes for Config to evaluate, then:
```bash
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-public-read-prohibited \
  --compliance-types NON_COMPLIANT
```

**Expected result**: `$TEST_BUCKET` appears as `NON_COMPLIANT`. Unlike CloudTrail (which tells you *what happened*) or GuardDuty (which tells you *what looks malicious*), Config is telling you **the current state doesn't match policy** — and it will keep re-evaluating and re-flagging this bucket every time its configuration changes, not just once.

### Step 10: Remediate and Confirm Config Catches the Fix Too
```bash
aws s3api put-public-access-block --bucket $TEST_BUCKET \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-acl --bucket $TEST_BUCKET --acl private
```
Re-run Step 9's query after a few minutes — the bucket should now show `COMPLIANT`.

---

## Part 4: Security Hub — One Pane of Glass

### Step 11: Enable Security Hub and the Integrations
```bash
aws securityhub enable-security-hub --enable-default-standards
```
GuardDuty and Config findings flow into Security Hub automatically once enabled — no manual connector configuration needed for AWS-native services.

### Step 12: Review the Aggregated View
1. **Security Hub** → **Findings**
2. Filter by **Product name**: `GuardDuty` — confirm your sample findings from Part 2 appear here too
3. Filter by **Product name**: `Config` — confirm the S3 public-access finding from Part 3 appears here too
4. **Security Hub** → **Security standards** → review your score against **AWS Foundational Security Best Practices** — this is a broader, continuously-evaluated checklist than the single Config rule you added manually

**This is the answer to "which tool do I check every morning"**: Security Hub, not each service individually — it's the aggregation layer specifically built so you don't have to check CloudTrail, GuardDuty, and Config separately.

---

## Part 5: Cleanup

```bash
aws securityhub disable-security-hub

aws configservice stop-configuration-recorder --configuration-recorder-name default
aws configservice delete-config-rule --config-rule-name s3-bucket-public-read-prohibited
aws configservice delete-delivery-channel --delivery-channel-name default
aws configservice delete-configuration-recorder --configuration-recorder-name default

aws guardduty delete-detector --detector-id $DETECTOR_ID

aws cloudtrail stop-logging --name org-audit-trail
aws cloudtrail delete-trail --name org-audit-trail

aws s3 rb s3://$BUCKET_NAME --force
aws s3 rb s3://$CONFIG_BUCKET --force
aws s3 rb s3://$TEST_BUCKET --force
```

> **Important**: GuardDuty, Config, and Security Hub are account/region-level services — confirm each is explicitly disabled, not just that the S3 buckets are gone. They continue evaluating and billing independently of any specific resource.

---

## Key Concepts to Understand

| Service | Answers | Data Source |
|---------|---------|--------------|
| **CloudTrail** | "What API call happened, and who made it?" | Every management (and optionally data) event in the account |
| **GuardDuty** | "Does this activity look malicious?" | CloudTrail events + VPC Flow Logs + DNS logs, correlated against threat intelligence |
| **AWS Config** | "Is this resource configured correctly, right now and historically?" | Continuous resource configuration snapshots evaluated against rules |
| **Security Hub** | "What's my aggregated security posture across all of the above?" | Findings ingested from GuardDuty, Config, Inspector, and other integrated services |

---

## Common Mistakes to Avoid
- **Treating GuardDuty and Config as redundant**: GuardDuty detects behavioral/threat signals; Config detects configuration drift from policy — a resource can be perfectly "configured" and still generate a GuardDuty finding for suspicious runtime behavior, and vice versa
- **Single-region CloudTrail trail**: creates a blind spot in exactly the region an attacker is most likely to pivot to
- **Skipping log file validation**: without it, there's no way to prove CloudTrail logs weren't tampered with after an incident
- **Forgetting these are account-level services**: deleting the S3 bucket a trail writes to doesn't stop CloudTrail, GuardDuty, Config, or Security Hub from running (and billing) — each needs its own explicit disable step

---

## Next Steps
- Continue to [Lab 3: Data Protection](aws-sec-lab-3-data-protection.md) to add KMS and Secrets Manager visibility to what Config and Security Hub are watching
- Then [Lab 4: Incident Response Automation](aws-sec-lab-4-incident-response-automation.md) closes the loop — instead of a human reviewing the Config finding from Part 3 and remediating it manually, EventBridge + Lambda does it automatically
- Add a custom Config rule (via a Lambda-backed rule) for something specific to your environment that no AWS managed rule covers
