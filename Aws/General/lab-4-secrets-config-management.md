# Lab 4: Secrets & Config Management at Scale

Check box if done: [ ]

## Overview
"We store secrets in Secrets Manager" is table stakes; the actual skill is knowing when a value belongs in Secrets Manager versus SSM Parameter Store, proving rotation genuinely happens on a schedule instead of assuming a secret set once six months ago is still safe, and scoping IAM so only the one principal that needs a given secret can read it. This lab builds all three, plus the automated rotation Lambda most engineers have heard of but never actually deployed.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$1 — **unlike SSM Parameter Store's standard tier, AWS Secrets Manager has no free tier at all: each secret bills $0.40/month, prorated to about a cent for this lab's duration, and it keeps billing during the standard 30-day pending-deletion window unless you force-delete it in Cleanup.**

---

## Scenario
Your app currently has a database password sitting in a `.env` file that made it into git two commits ago, and non-secret config (feature flags, API endpoints) hardcoded alongside it with no per-environment separation. You're fixing both: real secrets go into Secrets Manager with automated rotation so a compromised credential has a short shelf life even if nobody notices the compromise, non-secret config goes into SSM Parameter Store with a hierarchical naming scheme so `/app/prod/*` and `/app/dev/*` can't collide, and IAM policy on both ensures only the specific role that needs a given value can read it.

---

## Objectives
- Create and retrieve a secret in AWS Secrets Manager
- Deploy an automated rotation Lambda implementing the create/set/test/finish rotation pattern
- Store hierarchical, non-secret configuration in SSM Parameter Store, including `SecureString` parameters
- Scope IAM policy so only an intended principal can read a specific secret or parameter — and prove an unscoped principal is denied
- Reason through when to use Secrets Manager versus Parameter Store

---

## Part 1: Create and Retrieve a Secret

### Step 1: Store the Secret
```bash
aws secretsmanager create-secret \
  --name lab4/app/db-credentials \
  --description "Lab 4 demo DB credentials" \
  --secret-string '{"username":"appuser","password":"<initial-placeholder-value>"}'
```
Storing a structured JSON blob (not just a bare string) is the standard pattern — an application typically needs both the username and password together, and the rotation Lambda in Part 2 updates both fields as a unit.

### Step 2: Retrieve It
```bash
aws secretsmanager get-secret-value \
  --secret-id lab4/app/db-credentials \
  --query SecretString \
  --output text
```

**Validation checkpoint**: the JSON blob from Step 1 comes back exactly. Note the `VersionId` in the full (non-`--query`-filtered) response — every write to a secret creates a new version rather than overwriting in place, which is what makes Part 2's rotation safe: the old version stays retrievable under the `AWSPREVIOUS` stage label until the next rotation cycle ages it out.

---

## Part 2: Automated Rotation

### Step 3: Write the Rotation Lambda
Secrets Manager's rotation contract is four steps, called in order on separate invocations: `createSecret` (generate a new candidate value), `setSecret` (apply it to the actual backing system — a real RDS instance, an API), `testSecret` (verify the new value actually works), `finishSecret` (promote the candidate to the current version). This lab has no real backing database, so `setSecret`/`testSecret` are simplified — the structure is what matters, and it's identical to what you'd write against a real system.

```python
# rotate_db_credentials.py
import boto3
import json
import secrets
import string

sm = boto3.client('secretsmanager')

def lambda_handler(event, context):
    secret_arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    if step == 'createSecret':
        create_secret(secret_arn, token)
    elif step == 'setSecret':
        set_secret(secret_arn, token)
    elif step == 'testSecret':
        test_secret(secret_arn, token)
    elif step == 'finishSecret':
        finish_secret(secret_arn, token)

def create_secret(secret_arn, token):
    current = json.loads(sm.get_secret_value(SecretId=secret_arn, VersionStage='AWSCURRENT')['SecretString'])
    alphabet = string.ascii_letters + string.digits
    new_password = ''.join(secrets.choice(alphabet) for _ in range(24))
    new_secret = {'username': current['username'], 'password': new_password}
    sm.put_secret_value(
        SecretId=secret_arn,
        ClientRequestToken=token,
        SecretString=json.dumps(new_secret),
        VersionStages=['AWSPENDING'],
    )

def set_secret(secret_arn, token):
    # In a real rotation, this calls the backing system (e.g. RDS ModifyDBInstance
    # or an ALTER USER statement) to actually apply the AWSPENDING password.
    print(f"set_secret: would apply AWSPENDING credentials for {secret_arn}")

def test_secret(secret_arn, token):
    # In a real rotation, this connects to the backing system with the AWSPENDING
    # credentials and confirms authentication succeeds before finishSecret promotes it.
    print(f"test_secret: would verify AWSPENDING credentials work for {secret_arn}")

def finish_secret(secret_arn, token):
    sm.update_secret_version_stage(
        SecretId=secret_arn,
        VersionStage='AWSCURRENT',
        MoveToVersionId=token,
    )
```

**The critical safety property this preserves**: `finishSecret` is the only step that repoints `AWSCURRENT`. If `testSecret` had failed, the pending version would simply never get promoted, and the application keeps using the last known-good credential — rotation fails safe, not silently forward onto an untested value.

### Step 4: Deploy the Rotation Lambda
```bash
zip rotation.zip rotate_db_credentials.py

aws iam create-role \
  --role-name lab4-rotation-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{"Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]
  }'

aws iam put-role-policy \
  --role-name lab4-rotation-role \
  --policy-name rotation-permissions \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue", "secretsmanager:PutSecretValue", "secretsmanager:UpdateSecretVersionStage", "secretsmanager:DescribeSecret"],
      "Resource": "arn:aws:secretsmanager:us-east-1:<AWS_ACCOUNT_ID>:secret:lab4/app/db-credentials-*"
    }]
  }'

aws lambda create-function \
  --function-name lab4-rotate-db-credentials \
  --runtime python3.12 \
  --handler rotate_db_credentials.lambda_handler \
  --role arn:aws:iam::<AWS_ACCOUNT_ID>:role/lab4-rotation-role \
  --zip-file fileb://rotation.zip \
  --timeout 30
```
Note the resource ARN is scoped to `lab4/app/db-credentials-*` specifically, not `secret:*` — the rotation function should only ever be able to touch the one secret it's responsible for.

### Step 5: Grant Secrets Manager Permission to Invoke the Lambda, and Wire Up Rotation
```bash
aws lambda add-permission \
  --function-name lab4-rotate-db-credentials \
  --statement-id AllowSecretsManagerInvoke \
  --action lambda:InvokeFunction \
  --principal secretsmanager.amazonaws.com

aws secretsmanager rotate-secret \
  --secret-id lab4/app/db-credentials \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:<AWS_ACCOUNT_ID>:function:lab4-rotate-db-credentials \
  --rotation-rules AutomaticallyAfterDays=30 \
  --rotate-immediately
```
`--rotate-immediately` triggers the first rotation cycle now instead of waiting 30 days, so this lab is verifiable in real time.

### Step 6: Verify Rotation Happened
```bash
aws secretsmanager describe-secret --secret-id lab4/app/db-credentials \
  --query "{LastRotated:LastRotatedDate,NextRotation:NextRotationDate,Rules:RotationRules}"

aws secretsmanager get-secret-value --secret-id lab4/app/db-credentials --query SecretString --output text
```

**Validation checkpoint**: `LastRotatedDate` is populated (not `null`), and the retrieved password differs from Step 2's original value — proof the full four-step cycle ran and actually promoted a new version, not just that a Lambda executed without erroring.

---

## Part 3: SSM Parameter Store — Hierarchical Config

### Step 7: Store Plain Config
Parameter Store is the right home for values that aren't secrets — endpoints, feature flags, non-sensitive tuning knobs — where Secrets Manager's rotation machinery and per-secret cost add nothing.

```bash
aws ssm put-parameter \
  --name "/lab4/app/dev/api-endpoint" \
  --value "https://api-dev.internal.example.com" \
  --type String

aws ssm put-parameter \
  --name "/lab4/app/dev/feature-flags/new-checkout" \
  --value "true" \
  --type String
```

### Step 8: Store a SecureString Parameter
For a value that's sensitive but doesn't need Secrets Manager's rotation lifecycle (a static API key from a third party, for instance):

```bash
aws ssm put-parameter \
  --name "/lab4/app/dev/third-party-api-key" \
  --value "<placeholder-third-party-key>" \
  --type SecureString \
  --key-id alias/aws/ssm
```
`SecureString` encrypts the value at rest with KMS and requires `ssm:GetParameter` **plus** `kms:Decrypt` to read the plaintext back — two permission checks, not one, which is exactly the extra friction you want on a sensitive value.

### Step 9: Retrieve the Whole Hierarchy at Once
```bash
aws ssm get-parameters-by-path \
  --path "/lab4/app/dev" \
  --recursive \
  --with-decryption \
  --query "Parameters[*].{Name:Name,Value:Value}"
```
**Why the hierarchical naming matters**: `/lab4/app/dev/*` and a future `/lab4/app/prod/*` never collide, and an IAM policy can scope access to an entire environment's config with a single path-prefixed resource ARN instead of enumerating every parameter name individually — this is the mechanism Part 4 uses next.

---

## Part 4: IAM Scoping — Prove Only the Intended Principal Can Read

### Step 10: Create a Read-Only Role Scoped to Exactly This App's Secret and Parameters
```bash
aws iam create-role \
  --role-name lab4-app-read-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{"Effect": "Allow", "Principal": {"AWS": "arn:aws:iam::<AWS_ACCOUNT_ID>:root"}, "Action": "sts:AssumeRole"}]
  }'

aws iam put-role-policy \
  --role-name lab4-app-read-role \
  --policy-name scoped-read-access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "secretsmanager:GetSecretValue",
        "Resource": "arn:aws:secretsmanager:us-east-1:<AWS_ACCOUNT_ID>:secret:lab4/app/db-credentials-*"
      },
      {
        "Effect": "Allow",
        "Action": ["ssm:GetParameter", "ssm:GetParametersByPath"],
        "Resource": "arn:aws:ssm:us-east-1:<AWS_ACCOUNT_ID>:parameter/lab4/app/dev/*"
      },
      {
        "Effect": "Allow",
        "Action": "kms:Decrypt",
        "Resource": "arn:aws:kms:us-east-1:<AWS_ACCOUNT_ID>:alias/aws/ssm"
      }
    ]
  }'
```

### Step 11: Confirm an Unscoped Role Is Denied
Assume a different, unrelated role (any role in the account without an explicit allow on this secret) and attempt the same read:
```bash
aws secretsmanager get-secret-value --secret-id lab4/app/db-credentials --profile unrelated-role-test
```
**Expected result**: `AccessDeniedException` — no explicit allow on this secret's ARN exists for that principal, and IAM's default-deny means "not explicitly allowed" is functionally identical to "explicitly denied" for this purpose.

### Step 12: Confirm the Scoped Role Succeeds
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<AWS_ACCOUNT_ID>:role/lab4-app-read-role \
  --role-session-name lab4-test

# export the temporary credentials from the output, then:
aws secretsmanager get-secret-value --secret-id lab4/app/db-credentials --query SecretString --output text
```
**Validation checkpoint**: Step 11 fails with `AccessDeniedException`, Step 12 succeeds — the same deny-then-allow proof pattern used for IRSA in [Lab 1](lab-1-eks-fundamentals.md), applied here to secrets access instead of an S3 bucket.

---

## Part 5: Secrets Manager vs. Parameter Store — When to Use Which

| Dimension | Secrets Manager | SSM Parameter Store |
|---|---|---|
| **Cost** | $0.40/secret/month, no free tier | Standard tier free (up to 10,000 parameters); Advanced tier ($0.05/parameter/month) needed past that or for >4KB values |
| **Rotation** | Built-in four-step Lambda rotation, with native templates for RDS/Redshift/DocumentDB | No built-in rotation — you'd build your own EventBridge-scheduled rotation Lambda from scratch |
| **Versioning** | Full version history with stage labels (`AWSCURRENT`/`AWSPENDING`/`AWSPREVIOUS`) | Version history exists but no stage-label concept — just numbered versions |
| **Cross-account/cross-service sharing** | Resource policies support cross-account access directly | Cross-account access requires more manual IAM setup |
| **Best fit** | Database credentials, API keys needing scheduled rotation, anything where "prove this was rotated on schedule" matters for compliance | Non-secret config, feature flags, hierarchical app settings, and static sensitive values that don't need automated rotation |

**The practical rule**: if it needs to rotate on a schedule or you need to prove it does for an audit, it belongs in Secrets Manager despite the per-secret cost. If it's config — secret or not — that just needs to be read at startup and doesn't rotate, Parameter Store's free standard tier and hierarchical naming make it the better and cheaper fit.

---

## Cleanup

```bash
# Force-delete the secret immediately instead of the default 30-day recovery window —
# Secrets Manager keeps billing during that window, so this avoids a lingering charge
aws secretsmanager delete-secret \
  --secret-id lab4/app/db-credentials \
  --force-delete-without-recovery

# Remove the rotation Lambda and its role
aws lambda delete-function --function-name lab4-rotate-db-credentials
aws iam delete-role-policy --role-name lab4-rotation-role --policy-name rotation-permissions
aws iam delete-role --role-name lab4-rotation-role

# Remove the SSM parameters
aws ssm delete-parameter --name "/lab4/app/dev/api-endpoint"
aws ssm delete-parameter --name "/lab4/app/dev/feature-flags/new-checkout"
aws ssm delete-parameter --name "/lab4/app/dev/third-party-api-key"

# Remove the test IAM role
aws iam delete-role-policy --role-name lab4-app-read-role --policy-name scoped-read-access
aws iam delete-role --role-name lab4-app-read-role
```

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Four-step rotation contract** | `createSecret` → `setSecret` → `testSecret` → `finishSecret`, invoked separately by Secrets Manager; only `finishSecret` promotes the new version, so a failed `testSecret` leaves the app on the last known-good credential |
| **`AWSCURRENT`/`AWSPENDING`/`AWSPREVIOUS`** | Version stage labels Secrets Manager moves between versions during rotation — the mechanism that lets rotation be atomic and reversible rather than an in-place overwrite |
| **`SecureString` parameter** | An SSM parameter encrypted at rest with KMS, requiring `ssm:GetParameter` and `kms:Decrypt` both, unlike a plain `String` parameter |
| **Hierarchical parameter naming** | `/app/env/component/key`-style naming that lets a single path-prefixed IAM resource ARN scope access to an entire environment's config at once |
| **Least-privilege resource scoping** | Every policy in this lab names the exact secret/parameter ARN (or path prefix) rather than `*` — the difference between "this role can read one thing" and "this role can read everything in the account" |
| **Secrets Manager vs. Parameter Store cost model** | Secrets Manager has no free tier and bills per secret regardless of use; Parameter Store's standard tier is free — a real cost driver in choosing between them at scale, not just a feature checklist |

---

## Common Mistakes
- **Leaving a secret in the default 30-day pending-deletion window during cleanup**: Secrets Manager keeps billing until the window elapses — use `--force-delete-without-recovery` for lab/test secrets you're certain you don't need to recover
- **Putting genuinely non-secret config in Secrets Manager out of habit**: pays the $0.40/month fee and the rotation-machinery overhead for a value that never needed either — Parameter Store's free tier is the better fit
- **Scoping an IAM policy's resource to `secret:*` "to make it work" and never narrowing it**: defeats the entire purpose of Part 4 — always scope to the specific secret ARN or parameter path prefix actually needed
- **Assuming `testSecret` is optional or a formality**: skipping real verification here means a broken rotation gets promoted to `AWSCURRENT` anyway, and the application breaks in production the next time it fetches the secret
- **Forgetting `kms:Decrypt` when granting `SecureString` parameter access**: `ssm:GetParameter` alone isn't enough — the caller needs both permissions, or `--with-decryption` silently returns an encrypted blob instead of erroring clearly

---

## Next Steps
For a deeper, security-specific treatment of KMS key policies and encryption-at-rest patterns underpinning this lab's `SecureString` parameters, see [SCS-C02 Lab 3: Data Protection](../SCS-C02/aws-sec-lab-3-data-protection.md). The IAM scoping pattern in Part 4 is the same deny-then-allow verification used for IRSA in [Lab 1](lab-1-eks-fundamentals.md) and OIDC federation in [Lab 3](lab-3-cicd-oidc.md) — worth noticing as one recurring pattern across this whole track, not three unrelated techniques. [Lab 5](lab-5-observability-apm.md) continues with the same instrumented-app mindset, this time watching what that app does at runtime instead of what it can access.
