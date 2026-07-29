# Lab 2: Data & Storage Database Design

Check box if done: [ ]

## Overview
Anyone can create an S3 bucket or an RDS instance in the console. SAP-C02 cares about whether you picked the *right* storage and database service for a stated access pattern — and whether you can defend that pick against three other services that would also technically work, just worse. This lab runs two decision matrices before touching the CLI: one across S3/EBS/EFS/FSx for storage, one across RDS/Aurora/DynamoDB for the database tier — then deploys the services the decisions lead to, all encrypted with customer-managed KMS keys rather than the account default.

**Estimated time**: 75–90 minutes
**Cost**: ~$1–$3 (a db.t4g.micro RDS instance, minimal EFS and DynamoDB usage, and KMS key API calls — KMS keys themselves bill ~$1/month prorated to a few cents for a lab-length session)

---

## Scenario
A media company is replatforming two pieces of its stack. The first is a video-transcoding pipeline: multiple EC2 worker instances need concurrent, POSIX-compliant read/write access to the same set of in-progress files, and finished output needs to land somewhere cheap and durable for long-term archival with lifecycle tiering. The second is the metadata service tracking every video's transcoding job state — it needs strong consistency, very low and predictable latency at unpredictable scale, and a simple key-based access pattern (job ID in, status out), not complex joins or ad-hoc reporting queries. Compliance requires every piece of data at rest to be encrypted with a key the company controls and can audit usage of, not the AWS-managed default.

---

## Part 1: Design Decision — Storage and Database Service Selection

### Decision 1: S3 vs. EBS vs. EFS vs. FSx for the Transcoding Pipeline's Working Files

| Factor | S3 | EBS | EFS | FSx (for Windows/Lustre) |
|---|---|---|---|---|
| **Access model** | Object API (HTTP), not a filesystem — no POSIX semantics | Block device, attached to exactly one EC2 instance (or Multi-Attach for a narrow io2 use case) | Elastic, POSIX-compliant NFS filesystem, mountable by many instances concurrently | FSx for Windows: SMB for Windows workloads; FSx for Lustre: high-throughput POSIX filesystem for HPC/ML, typically S3-integrated |
| **Concurrent multi-instance access** | Yes, but as independent object GET/PUT, not shared file handles | No — single-instance by default, defeats the "multiple workers, same in-progress files" requirement outright | Yes — this is EFS's specific reason to exist | Yes for both variants, but FSx for Windows needs an AD dependency and Lustre is overkill outside HPC-scale throughput needs |
| **Durability for long-term archival** | 11 nines, purpose-built for this, with lifecycle tiering to Glacier classes | Durable but not designed as an archival tier — you'd pay EBS pricing for cold data indefinitely | Not built for archival; EFS Infrequent Access exists but S3 Glacier is materially cheaper at this scale | Not an archival target |
| **Cost shape** | Pay per GB stored plus request/retrieval, cheapest at rest by far, especially with lifecycle tiering | Pay per GB provisioned regardless of usage | Pay per GB used (no provisioning), higher per-GB than S3 | Higher baseline cost, provisioned throughput/storage models |
| **Best fit here** | The finished-output archival tier | Neither role — no shared access, wrong cost shape for archival | The in-progress shared working directory the workers need | Not needed — no Windows dependency, no HPC-scale throughput requirement |

### Decision 2: RDS vs. Aurora vs. DynamoDB for the Job-Status Metadata Service

| Factor | RDS (e.g., PostgreSQL) | Aurora (PostgreSQL/MySQL-compatible) | DynamoDB |
|---|---|---|---|
| **Access pattern fit** | Full relational engine — joins, ad-hoc queries — more than this workload needs | Same relational capability as RDS, plus better scaling headroom | Purpose-built for exactly this: key lookup in, item out, no joins needed |
| **Latency at unpredictable scale** | Single-writer instance scaling is manual (resize) and vertical; read replicas help reads only | Aurora Serverless v2 scales compute automatically within seconds, still bound by a single writer for strongly consistent writes | Single-digit-millisecond latency regardless of request volume; on-demand capacity mode scales without any provisioning step at all |
| **Operational overhead** | You manage failover tuning, storage scaling, and patching windows | Managed storage auto-scaling and faster failover than standard RDS, but still an instance-based engine with instance-level limits | No instances to size at all — the closest thing to zero operational surface area in this comparison |
| **Cost shape at this access pattern** | Pays for provisioned compute whether the job queue is quiet or bursting | Same instance-based cost floor as RDS unless using Serverless v2, which still has a compute floor | Pay-per-request (on-demand mode) means near-zero cost when the pipeline is idle, no capacity planning needed |
| **Best fit here** | Wrong tool — none of its relational strengths (joins, complex reporting) are needed by this workload | Better than RDS if a relational engine were required, but that requirement doesn't exist here | Matches every stated requirement: key-based access, unpredictable scale, low predictable latency, no joins |

### Recommendation for This Scenario
**Storage**: EFS for the transcoding workers' shared in-progress files (the only option here that actually satisfies concurrent POSIX access from multiple EC2 instances), S3 with a lifecycle policy for finished-output archival. EBS and FSx are ruled out explicitly — EBS because the access pattern is inherently multi-instance, FSx because neither its Windows-AD nor Lustre-HPC value proposition applies to this workload.

**Database**: DynamoDB for the job-status metadata service. The stated requirements — key-based lookups, unpredictable scale, low and predictable latency, no relational query needs — are DynamoDB's exact design target, and its operational overhead (zero instance sizing) is the lowest of the three options for a workload that never needed a relational engine's strengths in the first place.

**Encryption, both tiers**: customer-managed KMS keys, not the AWS-managed default — the stated compliance requirement (auditable key usage, company-controlled key policy) is the entire reason a customer-managed key exists instead of the free managed-key alternative.

---

## Part 2: Create the Customer-Managed KMS Key

### Step 1: Create the Key and an Alias
```bash
KEY_ID=$(aws kms create-key \
  --description "SAP-C02 Lab 2 - transcoding pipeline CMK" \
  --key-usage ENCRYPT_DECRYPT \
  --query "KeyMetadata.KeyId" --output text)

aws kms create-alias --alias-name alias/sap-c02-lab2-cmk --target-key-id $KEY_ID
```

### Step 2: Scope the Key Policy to This Workload
A customer-managed key's default policy grants the account root full access — fine for a lab, but worth narrowing in a real deployment to the specific roles (the transcoding workers, the metadata service) that should be allowed to use it, rather than every principal in the account. Confirm the default policy first:
```bash
aws kms get-key-policy --key-id $KEY_ID --policy-name default --query Policy --output text
```
This is the artifact an auditor asks for under the scenario's compliance requirement — a customer-managed key with a reviewable, scoped policy, distinct from the AWS-managed key's fixed, unreviewable policy.

**Validation checkpoint**:
```bash
aws kms describe-key --key-id $KEY_ID --query "KeyMetadata.{KeyState:KeyState,KeyManager:KeyManager}"
```
Confirm `KeyManager: CUSTOMER` — this distinguishes it from an AWS-managed key, which would show `AWS`.

---

## Part 3: Deploy EFS for the Shared Working Directory

### Step 3: Create the EFS Filesystem, Encrypted with the CMK
```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text)

FS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --kms-key-id $KEY_ID \
  --tags Key=Name,Value=sap-c02-lab2-transcode-workdir \
  --query "FileSystemId" --output text)
```

### Step 4: Create a Mount Target and a Security Group Scoped to NFS
```bash
EFS_SG=$(aws ec2 create-security-group --group-name sap-c02-lab2-efs-sg \
  --description "NFS from transcoding workers only" --vpc-id $VPC_ID --query "GroupId" --output text)

aws ec2 authorize-security-group-ingress --group-id $EFS_SG \
  --protocol tcp --port 2049 --cidr $(aws ec2 describe-vpcs --vpc-ids $VPC_ID --query "Vpcs[0].CidrBlock" --output text)

aws efs create-mount-target --file-system-id $FS_ID --subnet-id $SUBNET_ID --security-groups $EFS_SG
```
Scoping port 2049 (NFS) to the VPC CIDR, not `0.0.0.0/0`, is the point — EFS should never be reachable from outside the VPC that contains the workers using it.

**Validation checkpoint**:
```bash
aws efs describe-file-systems --file-system-id $FS_ID \
  --query "FileSystems[0].{Encrypted:Encrypted,KmsKeyId:KmsKeyId,LifeCycleState:LifeCycleState}"
```
Confirm `Encrypted: true`, `KmsKeyId` matches the CMK created in Part 2, and `LifeCycleState: available`.

---

## Part 4: Deploy S3 for Archival, with Lifecycle Tiering

### Step 5: Create the Bucket with Default Encryption Set to the CMK
```bash
ARCHIVE_BUCKET="sap-c02-lab2-archive-<your-unique-suffix>"
aws s3api create-bucket --bucket $ARCHIVE_BUCKET --region us-east-1

aws s3api put-bucket-encryption --bucket $ARCHIVE_BUCKET --server-side-encryption-configuration '{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms", "KMSMasterKeyID": "'"$KEY_ID"'"},
    "BucketKeyEnabled": true
  }]
}'
```
`BucketKeyEnabled: true` matters at any real volume — without it, every object PUT/GET makes its own call to KMS, which is both slower and directly billed per request; a bucket-level key reduces that to a small fraction of requests.

### Step 6: Apply a Lifecycle Policy Matching the Archival Access Pattern
Finished transcoding output is written once, read rarely after the first 30 days, and needs to exist for years — the textbook Glacier tiering case.
```bash
cat > lifecycle.json <<'EOF'
{
  "Rules": [{
    "ID": "archive-tiering",
    "Status": "Enabled",
    "Filter": {"Prefix": "finished-output/"},
    "Transitions": [
      {"Days": 30, "StorageClass": "GLACIER"},
      {"Days": 180, "StorageClass": "DEEP_ARCHIVE"}
    ]
  }]
}
EOF
aws s3api put-bucket-lifecycle-configuration --bucket $ARCHIVE_BUCKET --lifecycle-configuration file://lifecycle.json
```

**Validation checkpoint**:
```bash
aws s3api get-bucket-encryption --bucket $ARCHIVE_BUCKET
aws s3api get-bucket-lifecycle-configuration --bucket $ARCHIVE_BUCKET
```
Confirm the encryption configuration references your KMS key ID, and the lifecycle rule shows both transition steps.

---

## Part 5: Deploy DynamoDB for Job-Status Metadata, Encrypted with the CMK

### Step 7: Create the Table
```bash
aws dynamodb create-table \
  --table-name TranscodeJobStatus \
  --attribute-definitions AttributeName=JobId,AttributeType=S \
  --key-schema AttributeName=JobId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=$KEY_ID
```
`PAY_PER_REQUEST` (on-demand) directly implements Decision 2's cost-shape argument — no provisioned read/write capacity to size against a job queue that spikes unpredictably.

### Step 8: Write and Read a Test Item
```bash
aws dynamodb put-item --table-name TranscodeJobStatus --item '{
  "JobId": {"S": "job-0001"},
  "Status": {"S": "IN_PROGRESS"},
  "WorkerNode": {"S": "worker-a"}
}'

aws dynamodb get-item --table-name TranscodeJobStatus --key '{"JobId": {"S": "job-0001"}}'
```

**Validation checkpoint**:
```bash
aws dynamodb describe-table --table-name TranscodeJobStatus \
  --query "Table.{SSEStatus:SSEDescription.Status,SSEType:SSEDescription.SSEType,KeyArn:SSEDescription.KMSMasterKeyArn}"
```
Confirm `SSEStatus: ENABLED`, `SSEType: KMS`, and `KeyArn` matches the CMK from Part 2 — the third and final piece of "every piece of data at rest is encrypted with a key the company controls," alongside EFS and S3.

---

## Cleanup

```bash
# 1. DynamoDB table
aws dynamodb delete-table --table-name TranscodeJobStatus

# 2. S3 bucket (empty first, including any versioned/glacier objects)
aws s3 rm s3://$ARCHIVE_BUCKET --recursive
aws s3api delete-bucket --bucket $ARCHIVE_BUCKET

# 3. EFS mount target, then filesystem
MOUNT_TARGET_ID=$(aws efs describe-mount-targets --file-system-id $FS_ID --query "MountTargets[0].MountTargetId" --output text)
aws efs delete-mount-target --mount-target-id $MOUNT_TARGET_ID
# wait for the mount target to fully delete before deleting the filesystem
aws efs delete-file-system --file-system-id $FS_ID

# 4. Security group
aws ec2 delete-security-group --group-id $EFS_SG

# 5. KMS key — schedule deletion (minimum 7-day waiting period, cannot be instant)
aws kms delete-alias --alias-name alias/sap-c02-lab2-cmk
aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
```
`schedule-key-deletion` is deliberate, not an oversight — AWS enforces a minimum 7-day waiting window on customer-managed key deletion precisely so an accidental deletion request can still be cancelled (`aws kms cancel-key-deletion`) before any data encrypted under that key becomes permanently unrecoverable. The key is not billed at its usual ~$1/month rate once deletion is scheduled, only for the days already elapsed.

Confirm with `aws dynamodb list-tables`, `aws s3 ls`, and `aws efs describe-file-systems --query "FileSystems[?Name=='sap-c02-lab2-transcode-workdir']"` — all should show the lab's resources gone.

---

## Key Concepts

| Term | Definition |
|---|---|
| **EFS (Elastic File System)** | Managed, elastic NFS filesystem supporting concurrent POSIX access from multiple EC2 instances — the only storage service in this comparison built for shared multi-instance file access |
| **S3 lifecycle policy** | Rule set that automatically transitions objects between storage classes (Standard → Glacier → Deep Archive) based on object age, without application changes |
| **DynamoDB on-demand (PAY_PER_REQUEST)** | Billing mode with no provisioned capacity to size — cost scales directly with actual request volume, ideal for unpredictable or spiky access patterns |
| **Customer-managed KMS key (CMK)** | A KMS key you create and control the policy, rotation, and audit trail for, as opposed to an AWS-managed key with a fixed policy you can't edit |
| **Bucket Key (S3)** | An S3-generated, time-limited key derived from the KMS CMK that reduces the number of direct KMS API calls per object operation, cutting both latency and KMS request cost |
| **SSE-KMS** | Server-side encryption using a KMS key (managed or customer-managed) rather than SSE-S3's fully AWS-owned key — the mechanism that makes key usage auditable via CloudTrail |

---

## Common Mistakes
- **Reaching for EBS when the actual requirement is multi-instance concurrent access**: EBS is single-attach by default and the wrong tool the moment more than one instance needs to read/write the same files at once
- **Choosing DynamoDB by "NoSQL is more scalable" instead of by actual access pattern**: a workload that genuinely needs joins or ad-hoc reporting queries belongs in a relational engine regardless of scale — DynamoDB fits this scenario because the access pattern is key-based, not because it's newer
- **Skipping `BucketKeyEnabled` on an SSE-KMS bucket**: every object operation then makes its own KMS API call, which is both slower and a real cost line item at volume
- **Using the AWS-managed KMS key when a compliance requirement calls for an auditable, company-controlled key policy**: the managed key's policy can't be edited or reviewed the way a CMK's can — it fails the stated requirement even though it "also encrypts the data"
- **Forgetting KMS key deletion has a mandatory minimum 7-day window**: `schedule-key-deletion` doesn't delete anything immediately — plan cleanup timing accordingly if a key must be gone by a specific date

---

## Next Steps
This lab assumes comfort with the S3 and RDS basics from [SAA-C03 Labs 1–4](../SAA-C03/README.md) and the KMS/Secrets Manager depth from [SCS-C02 Lab 3: Data Protection](../SCS-C02/aws-sec-lab-3-data-protection.md) — that lab covers KMS key policy design and Secrets Manager rotation in more depth than this one re-teaches. Continue to [Lab 3: Business Continuity Design](lab-3-business-continuity-design.md) for the RTO/RPO-driven design that builds directly on this lab's storage and database choices.
