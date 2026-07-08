# AWS Security Lab 4: Incident Response Automation

Check box if done: []

## Overview
Lab 2 showed AWS Config catching a misconfigured public S3 bucket — and then a human had to notice the finding and fix it manually. This lab removes the human from that loop entirely: EventBridge watches for the same class of event in real time, and triggers a Lambda function that automatically remediates it — usually within seconds of the misconfiguration occurring, not whenever someone next checks Security Hub. The whole thing is deployed via Terraform, because a security guardrail that only exists in one account because someone clicked through the console once isn't a guardrail your organization can actually rely on.

**Estimated time**: 75–90 minutes
**Cost**: ~$0–$1 (EventBridge and Lambda are effectively free at this scale; the S3 buckets used for testing cost fractions of a cent)

---

## Scenario
The public-bucket incident from Lab 2 wasn't actually a lab exercise last time — it was real, and it sat public for six hours before anyone caught it in a routine Security Hub review. Leadership's ask is blunt: "make sure that can't happen again, and make sure it doesn't depend on someone checking a dashboard." You're building automated, real-time remediation, and packaging it as Terraform so it deploys the same way in every account your org runs.

---

## Objectives
- Understand the EventBridge + Lambda auto-remediation pattern and when it's appropriate (and when it isn't)
- Write a Lambda function that automatically re-secures a public S3 bucket the moment it's detected
- Wire up an EventBridge rule that triggers on the relevant Config compliance change
- Deploy the entire guardrail — Lambda, IAM role, EventBridge rule — via Terraform
- Test end to end: create a public bucket, and watch it get auto-remediated without touching it yourself

---

## Part 1: The Auto-Remediation Pattern

### Step 1: When This Pattern Is (and Isn't) Appropriate
Auto-remediation is a strong fit for **unambiguous, low-risk-to-revert** misconfigurations — a public S3 bucket that should never be public is a clean case: flipping Block Public Access back on has essentially no chance of breaking a legitimate workflow, because nothing should have been relying on public access in the first place. It's a **bad** fit for anything where the "correct" state is ambiguous or where automatic action could disrupt legitimate activity (e.g., auto-terminating an EC2 instance because of a single anomalous API call) — those cases belong in a human-reviewed workflow instead, which is why Lab 2 built the detection/visibility layer before this lab adds automation on top of it.

### Step 2: The Architecture
```
S3 bucket configuration changes
        │
        ▼
  AWS Config evaluates against a managed rule
        │
        ▼
  Config emits a compliance change event
        │
        ▼
  EventBridge rule matches the event
        │
        ▼
  Lambda function re-applies Block Public Access
        │
        ▼
  (Optional) SNS notification — "auto-remediated bucket X at time Y"
```

---

## Part 2: Write the Remediation Function

### Step 3: The Lambda Code
This function receives the EventBridge event, extracts the non-compliant bucket name, and forces Block Public Access back on.

```python
# remediate_public_bucket.py
import boto3
import json

s3 = boto3.client('s3')
sns = boto3.client('sns')

def lambda_handler(event, context):
    detail = event.get('detail', {})
    config_item = detail.get('newEvaluationResult', {}) or detail.get('configurationItem', {})

    bucket_name = detail.get('resourceId') or config_item.get('resourceId')
    if not bucket_name:
        print(f"No bucket name found in event: {json.dumps(event)}")
        return {'statusCode': 400, 'body': 'No resourceId in event'}

    print(f"Remediating public access on bucket: {bucket_name}")

    s3.put_public_access_block(
        Bucket=bucket_name,
        PublicAccessBlockConfiguration={
            'BlockPublicAcls': True,
            'IgnorePublicAcls': True,
            'BlockPublicPolicy': True,
            'RestrictPublicBuckets': True
        }
    )

    try:
        s3.put_bucket_acl(Bucket=bucket_name, ACL='private')
    except Exception as e:
        print(f"ACL reset skipped or failed (may already be private): {e}")

    print(f"Remediation complete for {bucket_name}")
    return {'statusCode': 200, 'body': f'Remediated {bucket_name}'}
```

**Note the scope discipline**: this function does exactly one thing — re-secure the specific bucket named in the event. It doesn't scan the whole account, doesn't delete anything, and doesn't touch any resource type other than the one it was built for. A remediation function with a broad, vague scope is itself a security risk if its logic ever has a bug.

---

## Part 3: Terraform — Deploy the Whole Guardrail

### Step 4: Project Structure
```bash
mkdir aws-auto-remediation && cd aws-auto-remediation
mkdir lambda
# Save the Python code from Step 3 as lambda/remediate_public_bucket.py
```

### Step 5: Package and Deploy the Lambda Function

`main.tf`:
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_caller_identity" "current" {}

data "archive_file" "lambda_zip" {
  type        = "zip"
  source_file = "${path.module}/lambda/remediate_public_bucket.py"
  output_path = "${path.module}/lambda/remediate_public_bucket.zip"
}

resource "aws_iam_role" "remediation_role" {
  name = "s3-auto-remediation-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

# Least-privilege: only the two S3 actions this function actually calls, plus log writes
resource "aws_iam_role_policy" "remediation_policy" {
  name = "s3-auto-remediation-policy"
  role = aws_iam_role.remediation_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:PutBucketPublicAccessBlock", "s3:PutBucketAcl"]
        Resource = "arn:aws:s3:::*"
      },
      {
        Effect   = "Allow"
        Action   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"]
        Resource = "arn:aws:logs:*:*:*"
      }
    ]
  })
}

resource "aws_lambda_function" "remediate_public_bucket" {
  function_name    = "remediate-public-s3-bucket"
  role             = aws_iam_role.remediation_role.arn
  handler          = "remediate_public_bucket.lambda_handler"
  runtime          = "python3.12"
  timeout          = 30
  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
}
```

### Step 6: Wire Up the EventBridge Rule

Append to `main.tf`:
```hcl
resource "aws_cloudwatch_event_rule" "config_noncompliant_s3" {
  name        = "s3-public-access-noncompliant"
  description = "Fires when AWS Config marks an S3 bucket non-compliant for public access"

  event_pattern = jsonencode({
    source      = ["aws.config"]
    detail-type = ["Config Rules Compliance Change"]
    detail = {
      configRuleName    = ["s3-bucket-public-read-prohibited"]
      newEvaluationResult = {
        complianceType = ["NON_COMPLIANT"]
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "trigger_remediation" {
  rule      = aws_cloudwatch_event_rule.config_noncompliant_s3.name
  target_id = "remediate-lambda"
  arn       = aws_lambda_function.remediate_public_bucket.arn
}

resource "aws_lambda_permission" "allow_eventbridge" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.remediate_public_bucket.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.config_noncompliant_s3.arn
}

output "lambda_function_name" {
  value = aws_lambda_function.remediate_public_bucket.function_name
}

output "eventbridge_rule_name" {
  value = aws_cloudwatch_event_rule.config_noncompliant_s3.name
}
```

> This assumes the `s3-bucket-public-read-prohibited` Config rule from [Lab 2](aws-sec-lab-2-detection-monitoring.md) already exists in the account. If you tore it down during that lab's cleanup, recreate it first (Lab 2, Step 7) before applying this configuration.

### Step 7: Deploy

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

---

## Part 4: Test End to End

### Step 8: Create a Public Bucket and Do Nothing
This is the test that proves the point — you should **not** need to manually fix anything from here forward.

```bash
aws s3api create-bucket --bucket auto-remediate-test-$RANDOM --region us-east-1
TEST_BUCKET=$(aws s3api list-buckets --query "Buckets[?starts_with(Name, 'auto-remediate-test')].Name" --output text)

aws s3api put-public-access-block --bucket $TEST_BUCKET \
  --public-access-block-configuration BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false

aws s3api put-bucket-acl --bucket $TEST_BUCKET --acl public-read
```

### Step 9: Watch the Pipeline Fire
Allow a few minutes for Config to evaluate the change and emit the compliance event, then check the Lambda's logs:

```bash
aws logs tail /aws/lambda/remediate-public-s3-bucket --since 10m
```

**Expected result**: a log entry showing `Remediating public access on bucket: <TEST_BUCKET>` followed by `Remediation complete`.

### Step 10: Verify the Bucket Was Actually Fixed
```bash
aws s3api get-public-access-block --bucket $TEST_BUCKET
```

**Expected result**: all four block settings show `true` — restored automatically, without you running a single remediation command by hand. Compare this timeline (minutes) against the six-hour exposure window from the lab's opening scenario.

---

## Part 5: Cleanup

```bash
terraform destroy -auto-approve

aws s3 rb s3://$TEST_BUCKET --force
```

---

## Key Concepts to Understand

| Concept | Definition |
|---------|-----------|
| **EventBridge + Lambda auto-remediation** | A serverless pattern that reacts to a compliance/security event in real time and takes corrective action without human involvement |
| **When auto-remediation is appropriate** | Unambiguous, low-risk-to-revert misconfigurations — not decisions requiring judgment or with a real chance of disrupting legitimate activity |
| **Least-privilege remediation role** | The remediation Lambda's IAM role should grant only the exact actions the function calls — a compromised remediation function with `s3:*` is a bigger risk than the misconfiguration it was meant to fix |
| **Terraform-deployed guardrail** | The same auto-remediation logic reproducibly deployed to every account, rather than existing only where someone happened to click it into place once |
| **Detection (Lab 2) vs. remediation (this lab)** | Detection tells you something is wrong; remediation fixes it — building both, in that order, is the mature version of a security program |

---

## Common Mistakes to Avoid
- **Skipping the least-privilege IAM role for the remediation function itself**: an over-permissioned auto-remediation Lambda is a bigger single point of failure than the misconfiguration it exists to fix
- **Auto-remediating anything with real judgment involved**: reserve automation for clean, unambiguous cases; route anything else to human review (Security Hub, as built in Lab 2)
- **Building this once in the console instead of as Terraform**: the entire value of this lab is that the guardrail is reproducible across every account — a console-only version isn't a guardrail your organization can rely on
- **Not testing the negative case**: always verify the remediation actually re-applies (Step 10), not just that the Lambda executed without erroring

---

## Next Steps
- Add an SNS notification to the Lambda so security team members are notified when an auto-remediation fires (visibility even when action is automatic)
- Extend the pattern to a second finding type (e.g., an unencrypted EBS volume, a security group with `0.0.0.0/0` on port 22)
- Add a DynamoDB table logging every remediation event with a timestamp, for the audit trail a real incident response program requires
- This closes out the SCS-C02 track — for the equivalent automated-remediation pattern in Azure, see [AZ-500 Lab 4](../../Azure/AZ-500/lab-4-security-ops-automation.md), and compare Sentinel's playbook/automation-rule model against this EventBridge + Lambda pattern directly
