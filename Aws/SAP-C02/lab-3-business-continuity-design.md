# Lab 3: Business Continuity Design

Check box if done: [ ]

## Overview
SAP-C02 doesn't test whether you can enable AWS Backup — it tests whether you can take a stated RTO/RPO and pick the *correct* disaster recovery strategy among four legitimate options, because the cheapest one usually fails the requirement and the most bulletproof one is usually unjustifiable spend. This lab builds that decision first, then proves it: an AWS Backup plan with cross-region copy, and an actual forced failover into the secondary region with the recovery time stopwatched, not estimated from documentation.

**Estimated time**: 75–100 minutes
**Cost**: ~$2–$5 (a small EC2 instance, a db.t4g.micro RDS instance, backup storage in two regions for the lab's duration, and a brief secondary-region restore — all deleted at the end)

---

## Scenario
You're the architect for a business-critical order-processing application: one EC2 instance running the application tier, one RDS PostgreSQL instance holding order data. The business has handed you a hard requirement: **RTO (Recovery Time Objective) of 1 hour** and **RPO (Recovery Point Objective) of 15 minutes** — if the primary region goes down, the application must be running again within an hour, and at most 15 minutes of order data can be lost. Your manager suggests "just turn on AWS Backup" and call the requirement met. That's wrong on both counts: default AWS Backup restores land you back in the *same* region, so it does nothing for a regional outage, and a restore-from-scratch of an EC2 instance and a database is not a 1-hour operation once you account for AMI/snapshot restore time, network re-attachment, and DNS propagation. You need a design that actually satisfies both numbers, and a design review answer for why the cheaper options don't.

---

## Objectives
- Build an RTO/RPO-driven comparison across the four standard DR strategies and justify the recommended one before touching the CLI
- Configure an AWS Backup plan with a tight backup window and cross-region copy to a secondary region
- Trigger an on-demand backup and confirm the cross-region copy completes
- Execute an actual forced failover into the secondary region and measure the wall-clock recovery time against the 1-hour RTO
- Understand why RPO and RTO are satisfied by different mechanisms, not the same one

---

## Part 1: Design Decision — Meeting RTO 1h / RPO 15min

### Step 1: Evaluate the Four DR Strategies Against the Requirement

| Strategy | RPO achieved | RTO achieved | Standing secondary-region cost | Why / why not |
|---|---|---|---|---|
| **Backup & Restore** | Depends entirely on backup frequency — hourly AWS Backup jobs put a floor near 60 minutes of possible loss, which already fails a 15-minute RPO on its own | Hours — nothing is pre-provisioned in the secondary region; every resource (compute, database, networking) is created from scratch during the incident | Lowest — only backup storage cost, no standing compute | Fails both numbers as a standalone strategy. It's the cheapest option and the correct *floor*, not the correct *ceiling*. |
| **Pilot Light** | Minutes, if database replication is continuous (not backup-based) rather than snapshot-based | Tens of minutes to ~1 hour — core data infrastructure (a minimal, always-on database replica) exists, but the application tier must be scaled up from zero during the incident | Low-moderate — a small standing database replica, no standing application compute | Close to satisfying both numbers, but scaling the application tier from zero adds real, variable time that eats directly into a 1-hour budget — risky to rely on as the sole margin |
| **Warm Standby** | Minutes — continuous or near-continuous data replication to an already-running, scaled-down secondary environment | Minutes to low tens of minutes — the application tier is already running at reduced capacity; failover means scaling *up* an existing fleet, not standing one up from nothing | Moderate — a running (small) application tier plus a running database replica, both billing continuously | Satisfies both numbers with real margin, at a cost premium over pilot light that's justified by materially lower failover-time risk |
| **Multi-Site Active-Active** | Near-zero — both regions serve live traffic against continuously synchronized data | Near-zero — no failover event at all, traffic simply routes away from the failed region | Highest — full production capacity running in both regions simultaneously, all the time | Massive overkill for a 1-hour RTO / 15-minute RPO requirement — this strategy earns its cost only when the RTO requirement approaches zero |

### Step 2: The Recommendation — Warm Standby

**Backup & Restore is ruled out outright** — it fails the stated RPO on its own terms, before RTO is even considered. **Multi-Site Active-Active is ruled out on cost proportionality** — nothing about a 1-hour RTO justifies paying for full duplicate production capacity running continuously in two regions. That leaves Pilot Light and Warm Standby as the real candidates.

**Warm Standby wins over Pilot Light for this scenario specifically because of the RTO number.** Pilot Light's failure mode is scaling the application tier from zero under incident pressure — that scale-up time is exactly the kind of variable, hard-to-guarantee step that erodes a 1-hour budget when things don't go perfectly (AMI launch delays, capacity constraints, configuration drift discovered only during the incident). Warm Standby removes that variable by keeping a minimal application tier already running and warm; failover becomes "scale up an existing, already-validated fleet" rather than "stand up a fleet from scratch during an outage." The cost premium over Pilot Light — one small EC2 instance running continuously in the secondary region — is proportionate to the risk it removes from a hard 1-hour commitment.

This lab builds the RPO and data-durability half of Warm Standby (AWS Backup with a tight cross-region copy schedule) and proves the RTO half with an actual timed restore into the secondary region — the closest a single-account lab can get to a real warm-standby failover without provisioning duplicate always-on compute for the exercise itself.

---

## Part 2: Deploy the Primary-Region Application

### Step 3: Create the Primary-Region EC2 Instance and RDS Database
```bash
PRIMARY_REGION="us-east-1"
SECONDARY_REGION="us-west-2"

aws ec2 run-instances --region $PRIMARY_REGION \
  --image-id $(aws ec2 describe-images --region $PRIMARY_REGION --owners amazon \
    --filters "Name=name,Values=al2023-ami-*-x86_64" "Name=state,Values=available" \
    --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text) \
  --instance-type t3.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=sap-c02-lab3-app}]' \
  --count 1

aws rds create-db-instance --region $PRIMARY_REGION \
  --db-instance-identifier sap-c02-lab3-db \
  --db-instance-class db.t4g.micro \
  --engine postgres \
  --master-username labadmin \
  --master-user-password "<your-own-password>" \
  --allocated-storage 20 \
  --no-publicly-accessible
```
Wait for both to report `running` / `available` before continuing — a backup job against a still-provisioning resource will fail or capture an incomplete state.

---

## Part 3: Configure AWS Backup with Cross-Region Copy

### Step 4: Create Backup Vaults in Both Regions
```bash
aws backup create-backup-vault --region $PRIMARY_REGION --backup-vault-name sap-c02-lab3-primary-vault
aws backup create-backup-vault --region $SECONDARY_REGION --backup-vault-name sap-c02-lab3-secondary-vault
```

### Step 5: Create a Backup Plan With Cross-Region Copy Action
This is the mechanism that implements Warm Standby's RPO half — every backup taken in the primary region is automatically copied to the secondary region without a separate manual step.

```bash
SECONDARY_VAULT_ARN=$(aws backup describe-backup-vault --region $SECONDARY_REGION \
  --backup-vault-name sap-c02-lab3-secondary-vault --query "BackupVaultArn" --output text)

aws backup create-backup-plan --region $PRIMARY_REGION --backup-plan '{
  "BackupPlanName": "sap-c02-lab3-plan",
  "Rules": [{
    "RuleName": "hourly-with-cross-region-copy",
    "TargetBackupVaultName": "sap-c02-lab3-primary-vault",
    "ScheduleExpression": "cron(0 * ? * * *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 120,
    "Lifecycle": {"DeleteAfterDays": 7},
    "CopyActions": [{
      "DestinationBackupVaultArn": "'"$SECONDARY_VAULT_ARN"'",
      "Lifecycle": {"DeleteAfterDays": 7}
    }]
  }]
}'
```
Hourly is the tightest realistic *scheduled* interval for a snapshot-based backup — note explicitly, the way the RTO/RPO decision in Part 1 already accounted for, that scheduled AWS Backup jobs alone put an RPO floor near an hour. Warm Standby's actual sub-hour RPO comes from the standing database replica's continuous replication in a full deployment; this lab's on-demand backup in Step 6 is what proves the cross-region copy mechanism works, not the sole RPO mechanism a production Warm Standby would rely on.

### Step 6: Assign Resources and Trigger an On-Demand Backup
```bash
PLAN_ID=$(aws backup list-backup-plans --region $PRIMARY_REGION \
  --query "BackupPlansList[?BackupPlanName=='sap-c02-lab3-plan'].BackupPlanId" --output text)

DB_ARN=$(aws rds describe-db-instances --region $PRIMARY_REGION \
  --db-instance-identifier sap-c02-lab3-db --query "DBInstances[0].DBInstanceArn" --output text)

aws backup start-backup-job --region $PRIMARY_REGION \
  --backup-vault-name sap-c02-lab3-primary-vault \
  --resource-arn "$DB_ARN" \
  --iam-role-arn "arn:aws:iam::<your-account-id>:role/service-role/AWSBackupDefaultServiceRole" \
  --lifecycle DeleteAfterDays=7 \
  --idempotency-token lab3-ondemand-1
```
On-demand rather than waiting for the hourly schedule — there needs to be a completed recovery point to actually restore from before the failover test in Part 4.

**Validation checkpoint**:
```bash
aws backup list-backup-jobs --region $PRIMARY_REGION --by-resource-arn "$DB_ARN" \
  --query "BackupJobs[0].{State:State,PercentDone:PercentDone}"
```
Poll until `State: COMPLETED`. Then confirm the cross-region copy landed:
```bash
aws backup list-copy-jobs --region $PRIMARY_REGION --by-resource-arn "$DB_ARN" \
  --query "CopyJobs[0].{State:State,DestinationRegion:DestinationBackupVaultArn}"
```
`State: COMPLETED` here is the proof the RPO mechanism is real — a recovery point now genuinely exists in the secondary region, independent of the primary region's availability.

---

## Part 4: Forced Failover Test — Measure the Actual Recovery Time

This is the step a real design review will ask you to prove happened, not just describe. Start a timer before Step 7 and don't stop it until the restored database in the secondary region reports `available`.

### Step 7: Start the Clock, Then Restore Into the Secondary Region
```bash
START_TIME=$(date +%s)

RECOVERY_POINT_ARN=$(aws backup list-recovery-points-by-backup-vault --region $SECONDARY_REGION \
  --backup-vault-name sap-c02-lab3-secondary-vault \
  --query "RecoveryPoints[0].RecoveryPointArn" --output text)

aws backup start-restore-job --region $SECONDARY_REGION \
  --recovery-point-arn "$RECOVERY_POINT_ARN" \
  --iam-role-arn "arn:aws:iam::<your-account-id>:role/service-role/AWSBackupDefaultServiceRole" \
  --metadata '{"DBInstanceClass":"db.t4g.micro","DBInstanceIdentifier":"sap-c02-lab3-db-dr"}'
```

### Step 8: Poll Until the Restore Completes
```bash
RESTORE_JOB_ID=$(aws backup list-restore-jobs --region $SECONDARY_REGION \
  --query "RestoreJobs[0].RestoreJobId" --output text)

# Poll every 30-60s
aws backup describe-restore-job --region $SECONDARY_REGION --restore-job-id $RESTORE_JOB_ID \
  --query "{Status:Status,PercentDone:PercentDone}"
```

### Step 9: Confirm the Database is Live, Stop the Clock
```bash
aws rds describe-db-instances --region $SECONDARY_REGION \
  --db-instance-identifier sap-c02-lab3-db-dr --query "DBInstances[0].DBInstanceStatus"
```
Once this shows `available`:
```bash
END_TIME=$(date +%s)
echo "Recovery time: $(( (END_TIME - START_TIME) / 60 )) minutes"
```

**Validation checkpoint**: the elapsed time printed above is your measured RTO for the database tier — compare it directly against the 1-hour requirement from the scenario. In a real Warm Standby deployment, the application tier's "recovery time" is near-zero (an autoscaling policy adding capacity to an already-running fleet, seconds to low minutes), so the database restore measured here represents the dominant portion of total system RTO. If this restore alone approaches the 1-hour budget, that's a legitimate design finding — it means the database tier needs a continuously-replicating read replica in the secondary region promoted on failover (true Warm Standby data tier) rather than restore-from-backup, which is a slower mechanism than replication regardless of region.

---

## Cleanup

**Order matters** — restore jobs and the DR database must be removed before backup vaults will delete cleanly, and vaults refuse deletion while recovery points remain.

```bash
# 1. Delete the DR database created by the restore test
aws rds delete-db-instance --region $SECONDARY_REGION \
  --db-instance-identifier sap-c02-lab3-db-dr --skip-final-snapshot

# 2. Delete recovery points from both vaults
aws backup list-recovery-points-by-backup-vault --region $PRIMARY_REGION \
  --backup-vault-name sap-c02-lab3-primary-vault --query "RecoveryPoints[].RecoveryPointArn" --output text | \
  xargs -n1 -I{} aws backup delete-recovery-point --region $PRIMARY_REGION \
  --backup-vault-name sap-c02-lab3-primary-vault --recovery-point-arn {}

aws backup list-recovery-points-by-backup-vault --region $SECONDARY_REGION \
  --backup-vault-name sap-c02-lab3-secondary-vault --query "RecoveryPoints[].RecoveryPointArn" --output text | \
  xargs -n1 -I{} aws backup delete-recovery-point --region $SECONDARY_REGION \
  --backup-vault-name sap-c02-lab3-secondary-vault --recovery-point-arn {}

# 3. Delete the backup plan
aws backup delete-backup-plan --region $PRIMARY_REGION --backup-plan-id $PLAN_ID

# 4. Delete both vaults
aws backup delete-backup-vault --region $PRIMARY_REGION --backup-vault-name sap-c02-lab3-primary-vault
aws backup delete-backup-vault --region $SECONDARY_REGION --backup-vault-name sap-c02-lab3-secondary-vault

# 5. Delete the primary-region application resources
aws rds delete-db-instance --region $PRIMARY_REGION \
  --db-instance-identifier sap-c02-lab3-db --skip-final-snapshot
aws ec2 terminate-instances --region $PRIMARY_REGION \
  --instance-ids $(aws ec2 describe-instances --region $PRIMARY_REGION \
    --filters "Name=tag:Name,Values=sap-c02-lab3-app" --query "Reservations[0].Instances[0].InstanceId" --output text)
```
Cross-region backup storage is the ongoing-cost item to watch here — confirm both vaults report empty (`list-recovery-points-by-backup-vault` returning `[]`) before considering cleanup complete, since a vault with lingering recovery points continues billing storage even after the source database is gone.

Confirm with:
```bash
aws backup list-backup-vaults --query "BackupVaultList[?contains(BackupVaultName, 'sap-c02-lab3')]"
aws rds describe-db-instances --region $PRIMARY_REGION --query "DBInstances[?DBInstanceIdentifier=='sap-c02-lab3-db']"
```
Both should return empty.

---

## Key Concepts

| Term | Definition |
|---|---|
| **RTO (Recovery Time Objective)** | Maximum acceptable time to restore service after an outage — drives the failover automation and pre-provisioning design |
| **RPO (Recovery Point Objective)** | Maximum acceptable data loss, measured in time — drives replication/backup frequency design |
| **Backup & Restore** | Lowest-cost DR strategy: periodic backups, everything rebuilt from scratch on failure — fails tight RTO/RPO requirements by design |
| **Pilot Light** | Core data infrastructure kept running (minimally) in the DR region; compute is scaled up from zero during failover |
| **Warm Standby** | A scaled-down but fully running copy of the application stack in the DR region; failover scales existing capacity up rather than provisioning from nothing |
| **Multi-Site Active-Active** | Full production capacity running simultaneously in multiple regions with continuous data synchronization — near-zero RTO/RPO at the highest standing cost |
| **AWS Backup cross-region copy** | A `CopyAction` on a backup plan rule that automatically replicates each recovery point to a vault in a second region, independent of the source region's availability |
| **Recovery point** | A specific, restorable snapshot of a resource's state at a point in time, tracked by AWS Backup regardless of which underlying service (RDS, EBS, EFS) created it |

---

## Common Mistakes
- **Assuming AWS Backup alone satisfies a regional RTO/RPO requirement**: without an explicit cross-region `CopyAction`, backups stay in the source region and are useless the moment that region is the thing that's down
- **Picking Pilot Light for a tight RTO without accounting for application-tier scale-up time**: the database being ready fast doesn't matter if the application tier still needs to be provisioned from zero before it can serve traffic
- **Treating a snapshot-based backup schedule as meeting a sub-hour RPO**: scheduled backup jobs have a real floor on frequency; a genuinely tight RPO needs continuous replication (a read replica, streaming replication), not just a tighter cron schedule
- **Never timing an actual restore**: an RTO number in a design document that's never been measured against a real restore is a guess, not a validated design — this lab's Part 4 stopwatch is the difference
- **Forgetting backup vaults bill for storage independent of the source resource**: deleting the RDS instance doesn't stop cross-region backup storage costs — recovery points must be deleted explicitly

---

## Next Steps
This lab builds directly on the storage and database service choices from [Lab 2: Data & Storage Database Design](lab-2-data-storage-database-design.md). It assumes comfort with the RDS Multi-AZ failover mechanics covered in [SAA-C03 Lab 5: Resilient Multi-AZ Architecture](../SAA-C03/aws-lab-5-resilient-architecture.md) — that lab's forced failover measures AZ-level (not regional) recovery, a distinction this lab's Decision table deliberately calls out (Availability Zones and Warm Standby answer different failure scopes, not the same one). Continue to [Lab 4: Compute & Application Architecture Design](lab-4-compute-application-architecture-design.md) for the hosting-model decisions that determine how fast the application tier itself can scale during a real failover.
