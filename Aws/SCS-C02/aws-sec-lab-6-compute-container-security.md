# AWS Security Lab 6: Compute & Container Security

Check box if done: []

## Overview
Every EC2 instance and container is a piece of attack surface with its own OS, its own packages, and its own IAM role — and most real-world compromises start with exactly one of those three being wrong: an open SSH port, an unpatched vulnerability nobody noticed, or an instance role so broad that compromising one small service compromises the whole account. This lab closes all three: replacing SSH entirely with an auditable, keyless access pattern, scanning EC2 and container images continuously for known vulnerabilities, scoping an instance role down from "works" to "least privilege," and enabling runtime threat detection for container workloads.

**Estimated time**: 75–90 minutes
**Cost**: ~$0–$2 (Inspector's EC2/ECR scanning has a free tier and then bills per instance/image scanned — small for a lab; a t3.micro test instance is Free Tier eligible or a few cents/hour otherwise; ECR storage is fractions of a cent for one test image)

---

## Scenario
A new engineer just asked "how do I SSH into the prod web server" — and the fact that the answer involves a bastion host, a shared PEM key, and a security group rule someone will forget to remove is itself the finding. Separately, a recent audit discovered an EC2 instance role with `AdministratorAccess` attached "temporarily" eight months ago, and nobody's checked whether any of the account's container images have known CVEs since they were built. You're fixing access, visibility, and privilege scope for compute workloads in one pass.

---

## Objectives
- Replace SSH/bastion access with AWS Systems Manager Session Manager and explain the security advantages
- Enable Amazon Inspector for continuous EC2 and ECR vulnerability scanning
- Contrast an over-broad EC2 instance role with a properly scoped one
- Enable ECR image scanning on push and act on a real finding
- Understand what GuardDuty EKS Protection detects and how to enable it

---

## Part 1: Session Manager — Eliminating the Open-22 Attack Surface

### Step 1: Why Bastion Hosts and Open SSH Are the Problem, Not a Mitigation
A bastion host reduces *where* SSH is exposed but doesn't eliminate the underlying risks: a shared or leaked private key never expires on its own, SSH sessions aren't natively logged anywhere centralized, and the bastion itself is a permanently-running, internet-facing (or semi-facing) target. **AWS Systems Manager Session Manager** removes the open port entirely — no inbound rule for port 22 anywhere, no key to distribute or rotate, and every session is a fully logged, IAM-governed API call instead of a raw TCP connection.

### Step 2: Launch a Test Instance With No Inbound Rules At All
```bash
aws iam create-role --role-name netlab-ssm-role --assume-role-policy-document '{
  "Version": "2012-10-17",
  "Statement": [{"Effect": "Allow", "Principal": {"Service": "ec2.amazonaws.com"}, "Action": "sts:AssumeRole"}]
}'

aws iam attach-role-policy --role-name netlab-ssm-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile --instance-profile-name netlab-ssm-profile
aws iam add-role-to-instance-profile --instance-profile-name netlab-ssm-profile --role-name netlab-ssm-role

SG_ID=$(aws ec2 create-security-group --group-name netlab-ssm-sg --description "No inbound rules at all" --vpc-id <your-vpc-id> --query "GroupId" --output text)
# Deliberately no ingress rules added — SSM doesn't need any inbound port open

aws ec2 run-instances \
  --image-id <latest-amazon-linux-2023-ami> \
  --instance-type t3.micro \
  --iam-instance-profile Name=netlab-ssm-profile \
  --security-group-ids $SG_ID \
  --subnet-id <your-subnet-id>
```

**What's missing on purpose**: no key pair specified, no port 22 ingress rule. There is structurally no network path to SSH into this instance — the only access path is through SSM, which the instance reaches outbound over HTTPS, the same direction any normal application traffic flows.

### Step 3: Connect via Session Manager
```bash
aws ssm start-session --target <instance-id>
```

**Validation checkpoint**: confirm `aws ec2 describe-security-groups --group-ids $SG_ID --query "SecurityGroups[0].IpPermissions"` returns an empty list — there is no inbound rule making this connection possible, proving the access path is entirely through the IAM-governed SSM API, not the network layer.

### Step 4: Review the Audit Trail Session Manager Gives You for Free
```bash
aws ssm describe-sessions --state History --query "Sessions[*].{Target:Target,Owner:Owner,StartDate:StartDate}"
```
Every session — who started it, when, against which instance — is logged automatically. Optionally, enable session logging to S3 or CloudWatch Logs for full command-level transcripts:
```bash
aws ssm update-document --name "SSM-SessionManagerRunShell" --content '{
  "schemaVersion": "1.0",
  "inputs": {"cloudWatchLogGroupName": "/ssm/session-logs", "cloudWatchEncryptionEnabled": true}
}' --document-version "$LATEST"
```

**This is the concrete answer to the scenario's SSH question**: no shared key, no bastion to maintain and patch, and — unlike raw SSH — a complete, centralized, IAM-attributable log of exactly who accessed exactly which instance and when.

---

## Part 2: Amazon Inspector — Continuous Vulnerability Scanning

### Step 5: Enable Inspector
```bash
aws inspector2 enable --resource-types EC2 ECR
```
Unlike a point-in-time vulnerability scan, Inspector runs continuously — it re-evaluates EC2 instances and ECR images automatically whenever new CVEs are published, not just when someone remembers to trigger a scan.

### Step 6: Review Findings for the Test Instance
```bash
aws inspector2 list-findings \
  --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_EC2_INSTANCE"}]}' \
  --query "findings[*].{Title:title,Severity:severity,Resource:resources[0].id}"
```

**Validation checkpoint**: for any real instance running for more than a few hours, Inspector should have at least some findings — the point isn't that the instance is broken, it's that vulnerability visibility now exists continuously instead of relying on someone running an occasional manual scan. Sort by `severity` and confirm you can explain the remediation path for at least one `HIGH` or `CRITICAL` finding (typically: patch the named package via the OS package manager).

### Step 7: Understand What Inspector Does and Doesn't Cover
Inspector scans for known CVEs in OS packages and language-level dependencies (for Lambda and container images) and evaluates network reachability for EC2 findings — it does not do behavioral/runtime threat detection. That's GuardDuty's job (Lab 2), not Inspector's — the two are complementary, the same way Config and GuardDuty are complementary from Lab 2.

---

## Part 3: Least-Privilege EC2 Instance Roles

### Step 8: The Over-Broad Role — What "Temporary" Actually Looks Like
```bash
aws iam create-policy --policy-name netlab-overbroad-policy --policy-document '{
  "Version": "2012-10-17",
  "Statement": [{"Effect": "Allow", "Action": "*", "Resource": "*"}]
}'
```
This is the audit finding from the scenario, reproduced deliberately: an instance role that can do literally anything in the account. If the application running on this instance has a remote code execution vulnerability, an attacker who gains code execution inherits this exact policy via the instance metadata service — the instance role's scope *is* the blast radius of that compromise.

### Step 9: Determine What the Instance Actually Needs
Use IAM Access Analyzer's policy generation, based on actual CloudTrail activity, to find the real permission set instead of guessing:
```bash
aws accessanalyzer start-policy-generation \
  --policy-generation-details '{"principalArn": "arn:aws:iam::<your-account-id>:role/netlab-ssm-role"}'

aws accessanalyzer get-generated-policy --job-id <job-id-from-previous-command>
```
This inspects the role's actual CloudTrail event history and generates a policy scoped to only the actions it genuinely used — a data-driven starting point instead of a guess.

### Step 10: Replace the Broad Policy With a Scoped One
Suppose the instance only ever reads from one specific S3 bucket and writes CloudWatch metrics:
```bash
aws iam create-policy --policy-name netlab-scoped-policy --policy-document '{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": ["s3:GetObject"], "Resource": "arn:aws:s3:::netlab-app-data/*"},
    {"Effect": "Allow", "Action": ["cloudwatch:PutMetricData"], "Resource": "*"}
  ]
}'

aws iam detach-role-policy --role-name netlab-ssm-role --policy-arn arn:aws:iam::<your-account-id>:policy/netlab-overbroad-policy
aws iam attach-role-policy --role-name netlab-ssm-role --policy-arn arn:aws:iam::<your-account-id>:policy/netlab-scoped-policy
```

**Validation checkpoint**: from a session on the instance, attempt an action outside the scoped policy (e.g., `aws ec2 describe-instances`) and confirm `AccessDenied`, then confirm the two permitted actions still succeed. Proving the *restriction* works is just as important as proving the *legitimate* use case still works — an instance role change that breaks the application gets rolled back by the next engineer who touches it, silently reintroducing the over-broad policy.

---

## Part 4: ECR Image Scanning on Push

### Step 11: Enable Scan on Push
```bash
aws ecr create-repository \
  --repository-name netlab-app \
  --image-scanning-configuration scanOnPush=true
```
With `scanOnPush` enabled, every image pushed to this repository is automatically scanned by Inspector the moment it lands — vulnerabilities surface before the image is ever pulled into a running environment, not after.

### Step 12: Push an Image and Review the Scan
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-east-1.amazonaws.com

docker tag netlab-app:latest <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/netlab-app:latest
docker push <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/netlab-app:latest

aws ecr describe-image-scan-findings \
  --repository-name netlab-app --image-id imageTag=latest \
  --query "imageScanFindings.findingSeverityCounts"
```

**Expected result**: a severity breakdown (`CRITICAL`, `HIGH`, `MEDIUM`, etc.) for the pushed image. A base image built from an outdated public tag (e.g., an old `python:3.9` instead of a current slim/distroless variant) will almost always show findings — this is the concrete demonstration of why base image freshness matters, not just application code.

### Step 13: Gate Deployment on Scan Results (Conceptual)
In a real CI/CD pipeline, this is where you'd add a gate: fail the build if `CRITICAL` findings are present, rather than relying on someone manually checking the ECR console before deploying:
```bash
CRITICAL_COUNT=$(aws ecr describe-image-scan-findings --repository-name netlab-app --image-id imageTag=latest \
  --query "imageScanFindings.findingSeverityCounts.CRITICAL" --output text)

if [ "$CRITICAL_COUNT" != "None" ] && [ "$CRITICAL_COUNT" -gt 0 ]; then
  echo "Blocking deployment: $CRITICAL_COUNT CRITICAL findings"
  exit 1
fi
```
This is exactly the "shift left" idea from a security perspective: catching the vulnerable image before it deploys is categorically cheaper than detecting it running in production.

---

## Part 5: GuardDuty EKS Protection — Runtime Detection for Kubernetes

### Step 14: What EKS Protection Covers (and Why This Section Is Conceptual)
Standing up a full EKS cluster solely to demonstrate this feature is expensive and out of scope for a focused lab — the control plane alone bills ~$0.10/hour regardless of node usage, and a lab-only cluster with worker nodes attached quickly becomes the most expensive resource in the entire track. Instead, this section covers exactly what the feature does and how to enable it, so the concept and configuration are demonstrable without keeping a cluster running for the sake of a single lab section.

**GuardDuty EKS Protection has two components**:
- **EKS Audit Log Monitoring**: analyzes Kubernetes audit logs (control-plane API activity) for suspicious activity — anonymous API access attempts, unusual `configmap`/`secret` access patterns, privilege escalation via RBAC changes
- **EKS Runtime Monitoring**: deploys a GuardDuty security agent to worker nodes for real runtime visibility — process execution, file access, and network connections at the container/pod level, catching things audit logs alone can't (e.g., a container process spawning a reverse shell)

### Step 15: Enable EKS Protection (If You Have an Existing Cluster)
```bash
aws guardduty update-detector \
  --detector-id <your-detector-id-from-lab-2> \
  --features '[{"Name":"EKS_AUDIT_LOGS","Status":"ENABLED"},{"Name":"EKS_RUNTIME_MONITORING","Status":"ENABLED","AdditionalConfiguration":[{"Name":"EKS_ADDON_MANAGEMENT","Status":"ENABLED"}]}]'
```
`EKS_ADDON_MANAGEMENT` lets GuardDuty auto-manage the runtime agent's deployment as an EKS add-on, instead of you manually installing and upgrading a DaemonSet.

### Step 16: What a Finding Looks Like
A representative EKS Protection finding: `Impact:Kubernetes/SuccessfulAnonymousAccess` — an anonymous (unauthenticated) request succeeded against the Kubernetes API server, which should never happen in a correctly configured cluster and is a strong signal either of a misconfigured RBAC binding or active exploitation attempt. Runtime findings follow the same `ThreatPurpose:ResourceType/ThreatFamily` naming convention introduced in [Lab 2](aws-sec-lab-2-detection-monitoring.md) — e.g., `Execution:Runtime/NewBinaryExecuted`.

**Validation checkpoint (conceptual)**: if you do have a non-production EKS cluster available, enable the feature and generate GuardDuty's EKS sample findings the same way Lab 2 did for standard findings, then confirm they appear in Security Hub alongside your EC2 and IAM findings — the aggregation model from Lab 2 extends to container workloads without a separate pane of glass.

---

## Part 6: Cleanup

```bash
aws inspector2 disable --resource-types EC2 ECR

aws ecr batch-delete-image --repository-name netlab-app --image-ids imageTag=latest
aws ecr delete-repository --repository-name netlab-app --force

aws ec2 terminate-instances --instance-ids <instance-id>

aws iam detach-role-policy --role-name netlab-ssm-role --policy-arn arn:aws:iam::<your-account-id>:policy/netlab-scoped-policy
aws iam detach-role-policy --role-name netlab-ssm-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam remove-role-from-instance-profile --instance-profile-name netlab-ssm-profile --role-name netlab-ssm-role
aws iam delete-instance-profile --instance-profile-name netlab-ssm-profile
aws iam delete-role --role-name netlab-ssm-role

aws iam delete-policy --policy-arn arn:aws:iam::<your-account-id>:policy/netlab-overbroad-policy
aws iam delete-policy --policy-arn arn:aws:iam::<your-account-id>:policy/netlab-scoped-policy

aws ec2 delete-security-group --group-id $SG_ID

# If EKS Protection was enabled against a real cluster, disable it explicitly
aws guardduty update-detector --detector-id <your-detector-id> \
  --features '[{"Name":"EKS_AUDIT_LOGS","Status":"DISABLED"},{"Name":"EKS_RUNTIME_MONITORING","Status":"DISABLED"}]'
```

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Session Manager instead of SSH/bastion** | Eliminates an entire class of network attack surface and produces a centralized, IAM-attributable access log for free |
| **Amazon Inspector for EC2 and ECR** | Continuous vulnerability visibility instead of point-in-time or manual scanning |
| **Access Analyzer-driven least-privilege instance roles** | Scopes the blast radius of a compute compromise to only what the workload actually uses |
| **ECR scan-on-push with a deployment gate** | Catches vulnerable images before they reach production, not after |
| **GuardDuty EKS Protection (audit + runtime)** | Extends behavioral threat detection to container workloads, covering what Inspector's static scanning can't |

---

## Common Mistakes to Avoid
- **Keeping a bastion host "just in case" after adopting Session Manager**: it's the exact attack surface Session Manager was meant to eliminate — if it's not removed, it's still a live target
- **Treating Inspector and GuardDuty as redundant**: Inspector finds known vulnerabilities in static code/packages; GuardDuty (including EKS Protection) detects behavioral/runtime threats — the same detection-layering lesson from Lab 2, applied to compute
- **Granting broad instance roles "to avoid follow-up IAM tickets"**: this is precisely how an eight-month-old `AdministratorAccess` instance role happens — start scoped, expand only with evidence
- **Scanning container images but never gating deployment on the results**: a scan nobody acts on has the same practical value as no scan
- **Standing up a full EKS cluster just to test GuardDuty EKS Protection in a personal lab**: understand the feature and its enablement path conceptually first — the cluster and node cost isn't justified for a single feature demo

---

## Next Steps
- Continue to [Lab 7: Multi-Account Security & Governance](aws-sec-lab-7-multi-account-governance.md) to apply least-privilege and centralized visibility patterns across an entire AWS Organization, not just one account
- Add an EventBridge rule (extending [Lab 4](aws-sec-lab-4-incident-response-automation.md)'s pattern) that automatically quarantines an EC2 instance when Inspector reports a new `CRITICAL` finding
- If you have access to a non-production EKS cluster, complete Part 5 end to end rather than conceptually, and compare the audit-log findings against a Kubernetes-native tool like Falco
- Compare Session Manager's model directly against Azure Bastion from [AZ-500 Lab 2](../../Azure/AZ-500/lab-2-platform-network-security.md) — both eliminate open management ports, but the underlying mechanisms (IAM API vs. managed jump host) are architecturally different and worth being able to explain
