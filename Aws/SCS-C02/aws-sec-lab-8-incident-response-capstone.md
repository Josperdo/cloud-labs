# AWS Security Lab 8: Incident Response Capstone — Forensics & Containment

Check box if done: []

## Overview
Seven labs built the pieces: IAM evaluation logic, detection tooling, data protection, automated remediation, network controls, compute/container hardening, and organization-wide governance. This capstone runs them together against the scenario every security engineer eventually faces for real — a compromised EC2 instance, live, right now. You'll isolate it without touching or destroying evidence, preserve forensic state via an EBS snapshot, extend Lab 4's EventBridge + Lambda pattern to auto-revoke a leaked IAM credential the moment GuardDuty flags it, and walk the entire chain end to end: detection, containment, evidence preservation, credential rotation.

**Estimated time**: 90–120 minutes
**Cost**: ~$1–$3 (a forensics EC2 instance and EBS snapshot run for a short window; Lambda/EventBridge are effectively free at this scale)

---

## Scenario
GuardDuty just fired a `UnauthorizedAccess:EC2/MaliciousIPCaller.Custom` finding — an EC2 instance is making calls to a known-malicious IP, and a correlated finding shows an IAM access key being used from a geography nobody on the team has ever logged in from. This is no longer a configuration review; it's an active incident. The instinct to SSH in and start poking around is exactly wrong — it risks destroying volatile evidence and tips off an attacker who may still have access. You're running the actual playbook: isolate first, preserve evidence before touching anything, revoke the credential automatically, and only then investigate.

---

## Objectives
- Isolate a compromised EC2 instance via security group quarantine without terminating or rebooting it
- Snapshot the instance's EBS volume for offline forensic analysis without disturbing the live instance
- Attach the snapshot to a dedicated forensics instance for investigation
- Build automated IAM credential revocation triggered directly by a GuardDuty finding
- Walk the full chain end to end: detect → contain → preserve → rotate

---

## Part 1: Detection — The Finding That Starts Everything

### Step 1: Generate a Realistic Compromise Signal
Using the same sample-finding mechanism from [Lab 2](aws-sec-lab-2-detection-monitoring.md), generate the two correlated findings this scenario describes:
```bash
DETECTOR_ID=$(aws guardduty list-detectors --query "DetectorIds[0]" --output text)

aws guardduty create-sample-findings \
  --detector-id $DETECTOR_ID \
  --finding-types "UnauthorizedAccess:EC2/MaliciousIPCaller.Custom" "UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS"
```

### Step 2: Identify the Affected Resources
```bash
aws guardduty list-findings --detector-id $DETECTOR_ID --query "FindingIds" --output text

aws guardduty get-findings --detector-id $DETECTOR_ID --finding-ids <finding-id> \
  --query "Findings[0].Resource"
```
**Validation checkpoint**: from the finding detail, extract the specific `instanceId` and the IAM `accessKeyId` involved — every subsequent step in this lab acts on these two specific identifiers, not "the environment in general." A remediation with too broad a scope (recall [Lab 4](aws-sec-lab-4-incident-response-automation.md)'s scope-discipline lesson) is itself a risk during an active incident.

---

## Part 2: Containment — Isolate Without Destroying Evidence

### Step 3: Why Terminating or Rebooting the Instance Is Wrong Here
Terminating the instance destroys the in-memory state (running processes, network connections, anything not written to disk) that a forensic investigation needs most. Rebooting can trigger malware persistence mechanisms designed to detect and react to a restart. The correct first move is **network isolation while the instance keeps running exactly as it is**.

### Step 4: Create an Isolation Security Group
```bash
VPC_ID=$(aws ec2 describe-instances --instance-ids <compromised-instance-id> --query "Reservations[0].Instances[0].VpcId" --output text)

aws ec2 create-security-group \
  --group-name ir-quarantine-sg \
  --description "Zero inbound, zero outbound except to the forensics endpoint" \
  --vpc-id $VPC_ID

QUARANTINE_SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=ir-quarantine-sg" --query "SecurityGroups[0].GroupId" --output text)

# Revoke the default allow-all-outbound rule that exists on every new security group
aws ec2 revoke-security-group-egress \
  --group-id $QUARANTINE_SG_ID --protocol -1 --cidr 0.0.0.0/0

# Allow outbound only to a forensics collection endpoint (e.g., a VPC endpoint used to pull memory/disk artifacts)
aws ec2 authorize-security-group-egress \
  --group-id $QUARANTINE_SG_ID --protocol tcp --port 443 --cidr 10.0.200.0/28
```
**Why explicitly revoke the default egress rule**: every new security group is created with an implicit allow-all-outbound rule — leaving it in place would let a compromised instance continue reaching its command-and-control infrastructure even after "isolation." This is the single most commonly missed step in a real quarantine action.

### Step 5: Swap the Instance's Security Groups
```bash
aws ec2 modify-instance-attribute \
  --instance-id <compromised-instance-id> \
  --groups $QUARANTINE_SG_ID
```
This replaces every security group previously attached to the instance's network interface with only the quarantine group — the instance keeps running, keeps its IP, keeps its disk and memory state, but can no longer reach or be reached by anything except the narrow forensics path just carved out.

### Step 6: Validate the Isolation Actually Holds
```bash
aws ec2 describe-instances --instance-ids <compromised-instance-id> \
  --query "Reservations[0].Instances[0].SecurityGroups"
```
**Expected result**: only `ir-quarantine-sg` listed — confirm the previous (compromised-context) security groups are gone. Then, from a source that was previously permitted, attempt a connection to the instance and confirm it fails — proving the negative, the same discipline every prior lab in this track has required at each validation checkpoint.

### Step 7: Preserve the IAM Trail Before It Rotates Out of Reach
```bash
aws iam list-access-keys --user-name <affected-user-name> \
  --query "AccessKeyMetadata[*].{KeyId:AccessKeyId,Status:Status,Created:CreateDate}"

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<compromised-access-key-id> \
  --query "Events[*].{Time:EventTime,Name:EventName,Source:SourceIPAddress}"
```
Capture this output *before* Part 4 revokes the key — it's still recoverable from CloudTrail afterward, but documenting it now removes any ambiguity about what the key was doing right up to containment.

---

## Part 3: Evidence Preservation — EBS Snapshot Forensics

### Step 8: Snapshot the Volume Without Touching the Live Instance
```bash
VOLUME_ID=$(aws ec2 describe-instances --instance-ids <compromised-instance-id> \
  --query "Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId" --output text)

aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "IR capstone - forensic snapshot of compromised instance $(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Purpose,Value=forensics},{Key=Case,Value=ir-capstone-lab8}]'
```
A snapshot is a point-in-time, read-only copy of the volume — creating it doesn't modify, unmount, or interrupt the live instance in any way. This is the core discipline of disk forensics: work from a copy, never from the original.

### Step 9: Copy the Snapshot to a Locked-Down Forensics Account or Region (Best Practice)
```bash
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id <snapshot-id-from-step-8> \
  --destination-region us-west-2 \
  --description "IR capstone - forensic snapshot copy, isolated region"
```
Copying to a separate region (or a dedicated forensics account, per [Lab 7](aws-sec-lab-7-multi-account-governance.md)'s delegated-account model) protects the evidence from an attacker with broader account access who could otherwise delete the original snapshot to cover their tracks.

### Step 10: Attach the Snapshot to a Dedicated Forensics Instance
```bash
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --snapshot-id <snapshot-id> \
  --volume-type gp3

FORENSICS_VOLUME_ID=$(aws ec2 describe-volumes --filters "Name=snapshot-id,Values=<snapshot-id>" --query "Volumes[0].VolumeId" --output text)

aws ec2 attach-volume \
  --volume-id $FORENSICS_VOLUME_ID \
  --instance-id <forensics-instance-id> \
  --device /dev/sdf
```
**Why a separate forensics instance, not the compromised one**: mounting the copied volume read-only on a clean, dedicated instance (built with its own least-privilege role, per [Lab 6](aws-sec-lab-6-compute-container-security.md)) means any analysis tooling you run has zero risk of being tampered with by whatever compromised the original — you're never trusting a binary or a shell on a box you already know is compromised.

### Step 11: Mount Read-Only and Begin Analysis
```bash
sudo mkdir -p /mnt/forensics
sudo mount -o ro /dev/sdf1 /mnt/forensics

# Example: pull a timeline of recently modified files as a starting point
find /mnt/forensics -type f -newermt "$(date -u -d '48 hours ago' +%Y-%m-%dT%H:%M:%S)" -exec ls -la {} \;
```
**Validation checkpoint**: confirm the mount is genuinely read-only (`mount | grep forensics` should show `ro` in the options) — an accidental read-write mount during analysis can alter file access timestamps and other metadata that matters to the investigation.

---

## Part 4: Automated Credential Revocation — Extending Lab 4's Pattern

### Step 12: The Architecture
```
GuardDuty finding (credential exfiltration) → EventBridge rule (type + severity match) →
Lambda (deactivates key, attaches deny-all policy) → SNS notification to the security team
```
This is [Lab 4](aws-sec-lab-4-incident-response-automation.md)'s EventBridge + Lambda pattern, unchanged in structure, applied to a credential-compromise finding instead of a public-bucket Config finding — the same auto-remediation judgment call applies: revoking a credential GuardDuty has flagged as actively exfiltrated is unambiguous and low-risk-to-revert, which is exactly the bar Lab 4 set for when automation is appropriate.

### Step 13: The Lambda Function
```python
# revoke_compromised_credential.py
import boto3
import json

iam = boto3.client('iam')
sns = boto3.client('sns')

SNS_TOPIC_ARN = "arn:aws:sns:us-east-1:<your-account-id>:ir-capstone-alerts"

def lambda_handler(event, context):
    detail = event.get('detail', {})
    finding_type = detail.get('type', '')
    resource = detail.get('resource', {})
    access_key_details = resource.get('accessKeyDetails', {})

    access_key_id = access_key_details.get('accessKeyId')
    user_name = access_key_details.get('userName')

    if not access_key_id or not user_name:
        print(f"Missing key/user details in finding: {json.dumps(detail)}")
        return {'statusCode': 400, 'body': 'Incomplete finding detail'}

    print(f"Deactivating access key {access_key_id} for user {user_name} (finding: {finding_type})")

    # Deactivate rather than delete — preserves the key for forensic review, per Step 15
    iam.update_access_key(
        UserName=user_name,
        AccessKeyId=access_key_id,
        Status='Inactive'
    )

    # Attach an explicit deny-all policy as a second layer, in case other credentials exist for this identity
    iam.put_user_policy(
        UserName=user_name,
        PolicyName='ir-capstone-emergency-deny',
        PolicyDocument=json.dumps({
            "Version": "2012-10-17",
            "Statement": [{"Effect": "Deny", "Action": "*", "Resource": "*"}]
        })
    )

    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject=f"Auto-revoked credential: {user_name}",
        Message=f"GuardDuty finding {finding_type} triggered automatic deactivation of access key "
                f"{access_key_id} for user {user_name}. Manual investigation required before reactivation."
    )

    return {'statusCode': 200, 'body': f'Revoked {access_key_id} for {user_name}'}
```

### Step 14: Wire Up the EventBridge Rule
```bash
aws events put-rule \
  --name auto-revoke-compromised-credential \
  --event-pattern '{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
      "type": ["UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS", "UnauthorizedAccess:IAMUser/MaliciousIPCaller.Custom"],
      "severity": [{"numeric": [">=", 7]}]
    }
  }'

aws events put-targets \
  --rule auto-revoke-compromised-credential \
  --targets "Id"="1","Arn"="<lambda-function-arn>"

aws lambda add-permission \
  --function-name revoke-compromised-credential \
  --statement-id AllowEventBridgeInvoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn "<eventbridge-rule-arn>"
```
**Why gate on `severity >= 7`**: not every GuardDuty finding warrants automatic credential revocation — a `Low` severity reconnaissance finding is a very different judgment call than a `High` severity confirmed exfiltration finding. Scoping the pattern to only high-severity, high-confidence finding types keeps this automation inside the "unambiguous" bar Lab 4 set, rather than firing on noisy or speculative findings.

**Why deactivate, not delete**: deleting an access key removes it from `list-access-keys` entirely; deactivating (`Status: Inactive`) achieves the same practical outcome — the key can no longer authenticate — while preserving it for the forensic record and a deliberate, documented deletion once the investigation concludes.

### Step 15: Test End to End
```bash
aws guardduty create-sample-findings \
  --detector-id $DETECTOR_ID \
  --finding-types "UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS"

# Wait a minute or two, then confirm the key was deactivated
aws iam list-access-keys --user-name <affected-user-name> \
  --query "AccessKeyMetadata[*].{KeyId:AccessKeyId,Status:Status}"
```
**Expected result**: `Status: Inactive` for the affected key, with no manual action taken — the same "prove the automation actually fired" discipline from Lab 4's Step 10, now applied to a credential instead of a bucket policy.

---

## Part 5: The Full Chain — What Just Happened, End to End

| Stage | Lab Capability Used | Action Taken |
|-------|---------------------|---------------|
| **Detect** | GuardDuty (Lab 2) | Correlated findings flagged a compromised instance and an exfiltrated credential |
| **Contain** | Security groups (Lab 5) | Instance isolated via quarantine SG; default egress explicitly revoked |
| **Preserve** | EBS snapshots, IAM read access (Lab 3's key-policy discipline applied to evidence handling) | Volume snapshotted and copied cross-region before any live analysis |
| **Investigate** | Least-privilege forensics instance (Lab 6) | Snapshot mounted read-only on a clean, scoped-role instance |
| **Remediate** | EventBridge + Lambda (Lab 4), extended | Compromised credential auto-deactivated within minutes of the finding |
| **Govern** | Organization-wide detection (Lab 7) | If this were a multi-account org, the same GuardDuty finding and Lambda pattern would fire identically regardless of which member account the instance lived in, via delegated administration |

This is the answer to "walk me through how you'd handle a compromised instance" in an interview — not a single tool, but the seven-lab chain, applied in the order that avoids destroying evidence while still moving at automation speed on the parts that don't require human judgment.

---

## Part 6: Cleanup

```bash
# Lambda / EventBridge
aws events remove-targets --rule auto-revoke-compromised-credential --ids "1"
aws events delete-rule --name auto-revoke-compromised-credential
aws lambda delete-function --function-name revoke-compromised-credential
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:<your-account-id>:ir-capstone-alerts

# Forensics resources
sudo umount /mnt/forensics
aws ec2 detach-volume --volume-id $FORENSICS_VOLUME_ID
aws ec2 delete-volume --volume-id $FORENSICS_VOLUME_ID
aws ec2 deregister-image --image-id <forensics-ami-id> 2>/dev/null || true
aws ec2 terminate-instances --instance-ids <forensics-instance-id>

# Snapshots — retain per your organization's evidence-retention policy; delete only once the case is closed
aws ec2 delete-snapshot --snapshot-id <snapshot-id>
aws ec2 delete-snapshot --snapshot-id <copied-snapshot-id> --region us-west-2

# Quarantined instance and security group — restore or terminate per your actual incident conclusion
aws ec2 delete-security-group --group-id $QUARANTINE_SG_ID
aws ec2 terminate-instances --instance-ids <compromised-instance-id>

# IAM cleanup
aws iam delete-user-policy --user-name <affected-user-name> --policy-name ir-capstone-emergency-deny
aws iam delete-access-key --user-name <affected-user-name> --access-key-id <compromised-access-key-id>
```

> **Real-world note**: in an actual incident, snapshots and CloudTrail evidence are typically retained per a documented retention policy (often tied to legal/compliance requirements), not deleted at the end of a triage session the way this lab's cleanup does for cost purposes.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Security-group quarantine instead of terminate/reboot** | Preserves live evidence while still cutting off attacker access — the correct first move in almost every compute-compromise incident |
| **EBS snapshot forensics on a copy, never the original** | Core forensic discipline — analysis tooling never runs against evidence that could be altered or destroyed |
| **Cross-region snapshot copy** | Protects evidence from an attacker with account access broad enough to delete the original |
| **Automated credential deactivation extending Lab 4's pattern** | Demonstrates the same EventBridge + Lambda architecture generalizes across finding types, not just one |
| **Deactivate-not-delete for compromised credentials** | Balances immediate containment against preserving the forensic record |
| **Narrating the full 7-lab chain as one incident** | The actual shape of a real investigation — no single control in isolation tells the whole story |

---

## Common Mistakes to Avoid
- **Terminating or rebooting a compromised instance as the first response**: destroys volatile evidence and can trigger malware persistence/anti-forensic behavior
- **Forgetting to revoke the default allow-all-outbound rule when building a quarantine security group**: leaves the "isolated" instance still able to reach command-and-control infrastructure
- **Running forensic analysis tools directly on the compromised instance**: any output can't be trusted if the instance's own binaries or kernel may be compromised — always work from a copy on a clean instance
- **Deleting a compromised access key instead of deactivating it**: destroys forensic value that a deactivated-but-intact key preserves for the investigation
- **Auto-remediating on every GuardDuty finding regardless of severity**: apply the same judgment-call discipline from Lab 4 — automation belongs on unambiguous, high-confidence findings, not everything GuardDuty reports

---

## Next Steps
This closes out the SCS-C02 track. Revisit [Lab 2](aws-sec-lab-2-detection-monitoring.md) and [Lab 5](aws-sec-lab-5-network-perimeter-security.md) periodically — detection tuning and network baseline review are ongoing work, never a one-time setup, the same way this capstone's containment steps are only as good as the detection that triggers them.

- For the identity-governance layer that reduces how often a credential-exfiltration incident like this one happens in the first place — least-privilege from the start, PIM-equivalent just-in-time access — see [Lab 1](aws-sec-lab-1-iam-deep-dive.md) and, for the cross-cloud comparison, [AZ-500 Lab 1](../../Azure/AZ-500/lab-1-identity-access-protection.md)
- For the equivalent SIEM/SOAR-driven investigation workflow in Microsoft's stack — Sentinel analytics rules, incident triage, automation playbooks — see [SC-200](../../Azure/SC-200/README.md), which builds the same detect-triage-remediate muscle this capstone built, against a different toolset
- For the compliance/posture-monitoring layer that would have flagged the over-broad instance role or missing MFA that often precedes an incident like this one — see [SC-500](../../Azure/SC-500/README.md)
- Build a runbook document (outside this repo) codifying this lab's five-stage chain as your organization's actual incident response procedure — the difference between a lab exercise and a real capability is whether the team can execute this from memory under pressure, not whether the steps exist somewhere
