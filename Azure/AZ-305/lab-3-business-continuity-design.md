# Lab 3: Business Continuity Design

Check box if done: [ ]

## Overview
AZ-305 doesn't test whether you can click "Enable backup" — it tests whether you can take a business requirement like "RTO 4 hours, RPO 15 minutes" and translate it into the correct combination of Azure services, because no single feature satisfies both numbers on its own. This lab builds the decision-making muscle first, then implements the design: Azure Backup for operational recovery, Azure Site Recovery for regional disaster recovery, and a documented (not necessarily executed) failover test procedure.

**Estimated time**: 60-90 minutes
**Cost**: ~$1-$3 (a small B-series VM plus Recovery Services vault backup storage for the duration of the lab — deleted at the end)

---

## Scenario
You're the architect for a business-critical application running on a single Azure VM. The business has handed you a hard requirement: **RTO (Recovery Time Objective) of 4 hours** and **RPO (Recovery Point Objective) of 15 minutes**. In plain terms — if the VM or its region goes down, the app must be back up within 4 hours, and you can lose at most 15 minutes of data. Your manager asks you to "just turn on backup" and call it done. You know that's wrong: Azure Backup alone protects against accidental deletion and corruption, not a full regional outage, and it won't get you anywhere near a 4-hour RTO if the entire region is gone. You need to design a stack that actually meets both numbers — and be ready to explain in the design review why the cheaper, simpler options don't.

---

## Objectives
- Build an RTO/RPO-driven comparison table and justify the recommended design before touching the portal
- Configure Azure Backup with a tight-interval policy for operational data-loss recovery
- Perform an on-demand backup and a file-level restore to prove backup actually works
- Configure Azure Site Recovery (ASR) replication to a secondary region for regional DR
- Understand the distinction between Availability Zones and Paired Regions, and when each applies
- Document a safe test-failover procedure without leaving costly resources running

---

## Part 1: Design Decision — Meeting RTO 4h / RPO 15min

### Step 1: Evaluate the Options Against the Requirement

| Option | RPO achieved | RTO achieved | Why / why not |
|---|---|---|---|
| **Manual snapshots only** | Hours to days (whenever someone remembers) | Hours+ (manual disk swap, no orchestration) | Fails RPO outright — a manual, infrequent process can't hit 15 minutes. No automation, no SLA. |
| **Azure Backup alone** | Meets RPO for *data* if scheduled tightly (hourly log backups on top of daily image backups) | Fails RTO for a **regional** outage — restore target is the same region the backup runs against by default; a full VM restore is not a 4-hour operation at scale | Protects against corruption, accidental deletion, and ransomware — not against the region disappearing. It's a data-loss control, not a DR control. |
| **GRS/GZRS storage alone** | Meets data durability (storage-level replication, RPO measured in minutes) | Fails RTO — GRS replication is passive; there's no compute failover, no running VM in the secondary region, and Microsoft-initiated failover for GRS is not customer-triggered on demand | Protects the *bytes*, not the *running application*. You'd still need to stand up compute from scratch after a storage failover. |
| **Availability Zones (AZs)** | Sub-second RPO (synchronous replication across zones) | Minutes (automatic failover within the region) | Excellent for a **datacenter-level** failure inside a region. Does **not** protect against a **regional** outage — all zones in a region can be affected by a region-wide event. |
| **Azure Site Recovery (ASR)** | Near-continuous replication, typically low single-digit minutes of data loss (well inside 15 min) | Orchestrated, automated failover — minutes to low hours depending on VM count and recovery plan complexity, well inside 4 hours for a single VM | Purpose-built for regional DR: replicates VM disks continuously to a secondary region and lets you trigger an orchestrated failover on demand. |

### Step 2: The Recommendation — Azure Backup + ASR, Together

Neither service alone meets both numbers, and they aren't redundant with each other — they protect against different failure modes:

- **Azure Backup** answers: *"Someone deleted a VM / ransomware encrypted the disk / a bad deployment corrupted data — can I get last Tuesday's state back?"* This is operational recovery, and it needs retention (days to years), not just a recent replica.
- **Azure Site Recovery** answers: *"The primary region is down — can I get the application running somewhere else, fast?"* This is regional disaster recovery, and it needs a warm, continuously-updated replica, not a point-in-time backup.

A design that only has ASR has no protection against logical corruption (ASR will happily replicate corrupted data to the secondary region within seconds). A design that only has Azure Backup has no protection against a regional outage (nothing to fail over *to* — you'd be restoring VMs from scratch in a new region, blowing well past a 4-hour RTO). **Together**, they cover both axes of business continuity: Backup for "restore an earlier good state," ASR for "run somewhere else right now."

**Availability Zones are a complementary third layer**, not a replacement for either: enable AZs for the primary VM's resiliency against datacenter failure (near-zero RTO/RPO, essentially free architecturally), and layer ASR on top for the regional case AZs don't cover. This lab focuses on Backup + ASR since that's the pair that directly satisfies the stated RTO/RPO; note AZ placement as a design callout in Part 4.

---

## Part 2: Configure Azure Backup with a Tight-Interval Policy

### Step 3: Create the Resource Group and Test VM
```bash
az group create --name rg-az305-lab3 --location eastus

az vm create \
  --resource-group rg-az305-lab3 \
  --name vm-bc-lab \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --zone 1
```
The `--zone 1` flag pins the VM to an Availability Zone — this is the AZ layer referenced in Part 1's design decision, at no extra cost.

### Step 4: Create a Recovery Services Vault
```bash
az backup vault create \
  --resource-group rg-az305-lab3 \
  --name rsv-az305-lab3 \
  --location eastus
```

### Step 5: Create a Backup Policy With Frequent Scheduling
The default Azure Backup policy only supports daily backups (24-hour RPO at best) — that alone fails a 15-minute RPO requirement. Enhanced policies support hourly backups down to 4-hour intervals for VM snapshots, plus more frequent log backups for workload-aware backup (SQL/SAP HANA). For a standalone VM, note this explicitly as a design caveat: **VM-level Azure Backup cannot natively hit a 15-minute RPO** — its snapshot-based policy floor is hourly. Configure the tightest available interval and document that the 15-minute RPO for this workload is actually satisfied by ASR's near-continuous replication (Part 4), not by Azure Backup — this is exactly the kind of distinction AZ-305 expects you to articulate.

```bash
az backup policy create \
  --resource-group rg-az305-lab3 \
  --vault-name rsv-az305-lab3 \
  --name policy-hourly-vm \
  --backup-management-type AzureIaasVM \
  --policy '{
      "eTag": null,
      "properties": {
        "backupManagementType": "AzureIaasVM",
        "instantRpDetails": {},
        "schedulePolicy": {
          "schedulePolicyType": "SimpleSchedulePolicyV2",
          "scheduleRunFrequency": "Hourly",
          "hourlySchedule": {
            "interval": 4,
            "scheduleWindowStartTime": "2026-07-14T08:00:00Z",
            "scheduleWindowDuration": 12
          }
        },
        "retentionPolicy": {
          "retentionPolicyType": "LongTermRetentionPolicy",
          "dailySchedule": {
            "retentionTimes": ["2026-07-14T08:00:00Z"],
            "retentionDuration": { "count": 30, "durationType": "Days" }
          }
        },
        "instantRPDetails": {},
        "timeZone": "UTC",
        "instantRpRetentionRangeInDays": 2
      }
    }'
```
This creates hourly backups (the tightest snapshot-based interval Azure Backup supports for standard VM backup) with 30 days of daily retention.

### Step 6: Enable Backup on the VM
```bash
az backup protection enable-for-vm \
  --resource-group rg-az305-lab3 \
  --vault-name rsv-az305-lab3 \
  --vm vm-bc-lab \
  --policy-name policy-hourly-vm
```

**Validation checkpoint**: 
```bash
az backup item list \
  --resource-group rg-az305-lab3 \
  --vault-name rsv-az305-lab3 \
  --output table
```
Confirm `vm-bc-lab` is listed with `ProtectionState: Protected` and the policy name matches `policy-hourly-vm`.

---

## Part 3: Perform a File-Level Restore

### Step 7: Trigger an On-Demand Backup
Don't wait for the first scheduled run — force one so there's a recovery point to restore from.

```bash
az backup protection backup-now \
  --resource-group rg-az305-lab3 \
  --vault-name rsv-az305-lab3 \
  --container-name vm-bc-lab \
  --item-name vm-bc-lab \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  --retain-until 21-08-2026
```

Poll the job until it completes:
```bash
az backup job list \
  --resource-group rg-az305-lab3 \
  --vault-name rsv-az305-lab3 \
  --output table
```

### Step 8: Perform a File-Level (Not Full-VM) Restore
A full-VM restore proves the backup exists; a **file-level** restore proves you can actually recover data without the operational overhead of standing up a whole new VM — this is the far more common real-world recovery scenario (someone deleted a config file, not the entire server).

1. Portal → **Recovery Services vaults** → `rsv-az305-lab3` → **Backup items** → **Azure Virtual Machine** → `vm-bc-lab`
2. Select the completed recovery point → **File Recovery**
3. Choose the OS type of the VM (Linux) → **Download Script**
4. Azure generates a time-limited script and credentials that mount the recovery point as an iSCSI disk on a machine you control (this can be the source VM itself or any machine with network access to Azure)
5. Run the downloaded script on the target machine, browse the mounted volume, copy out the specific file(s) needed
6. When done, run the script's **unmount/disconnect** step — the mount is billed and time-limited, don't leave it attached

**Validation checkpoint**: Confirm you can browse the mounted recovery point's filesystem and copy at least one file back out. This is the proof point that backup is *restorable*, not just *running* — a backup you've never test-restored is a backup you don't actually have.

---

## Part 4: Configure Azure Site Recovery Replication to a Secondary Region

ASR configuration is largely portal-driven (the CLI/PowerShell `Az.RecoveryServices` cmdlets exist — e.g. `New-AzRecoveryServicesAsrPolicy`, `New-AzRecoveryServicesAsrReplicationProtectedItem` — but the multi-step wizard nature of ASR setup makes the portal the practical path for a one-off design lab).

### Step 9: Enable Replication for the VM
1. Portal → **Recovery Services vaults** → `rsv-az305-lab3` → **Site Recovery** → **Configure disaster recovery** → **+ Virtual machine**
2. **Source**: region `East US`, resource group `rg-az305-lab3`
3. **Target region**: choose a paired region — for `East US`, the Azure-paired region is `West US` (paired regions matter here: Microsoft prioritizes recovery of one region in a pair before the other during a broad outage, and platform updates are staggered across pairs to avoid both going down for the same maintenance event)
4. Select `vm-bc-lab` as the VM to replicate
5. Leave target resource group, network, and storage as the wizard's suggested defaults (auto-created in the target region) unless you have an existing landing zone to target
6. **Replication policy**: default policy replicates continuously with app-consistent recovery points typically every few minutes to an hour depending on configuration — well inside the 15-minute RPO target
7. **Enable Replication**

### Step 10: Monitor Initial Replication
Initial replication copies the full disk to the target region and can take anywhere from tens of minutes to a few hours depending on disk size and network throughput — this is a one-time cost; subsequent replication is delta-only.

1. **Site Recovery** → **Replicated items** → `vm-bc-lab`
2. Confirm status progresses from `Initial replication in progress` → `Protected`
3. Note the **RPO** value shown on the item — this is your live, measured RPO, not a theoretical one; confirm it's within the 15-minute target once steady-state replication is reached

**Validation checkpoint**: `Replicated items` shows `vm-bc-lab` with health `Protected` and a reported RPO under 15 minutes.

### Step 11: Document (Don't Execute) the Failover Test Procedure
A full/planned failover actually moves production traffic and has real cost and downtime implications — not something to run casually in a lab. A **test failover** is the safe equivalent: it spins up a copy of the VM in an isolated network in the target region without touching the live replication or the production VM, so document the procedure and understand it rather than necessarily running it end-to-end if cost/time is a constraint:

1. **Site Recovery** → **Replicated items** → `vm-bc-lab` → **Test Failover**
2. **Recovery point**: choose "Latest processed" for the freshest data, or a specific earlier point to test a specific recovery scenario
3. **Azure virtual network**: select (or create) an **isolated test network** — critically, not the production VNet, so the test VM can't collide with or disrupt live traffic
4. Run the test failover — Azure spins up a VM from the replicated disks in the target region, attached to the isolated network
5. **Validate**: connect to the test VM (via Bastion or a jumpbox in the isolated network), confirm the application starts, data is present and current as of the chosen recovery point
6. **Critical step — clean up the test failover**: **Site Recovery** → the replicated item → **Cleanup test failover**. This deletes the temporary test VM and its resources. Skipping this step leaves a duplicate VM (and its compute/storage cost) running indefinitely in the target region — the test failover does not automatically expire.

**Why this matters for the exam and for real DR**: an untested failover plan is not a DR plan, it's a hope. Test failovers should be run on a recurring schedule (quarterly is a common baseline) specifically because they don't touch production, so there's no excuse not to.

---

## Cleanup

**Deletion order matters here — the Recovery Services vault will refuse to delete while it still has protected items (ASR replication or Backup items) attached to it.** Work through both protection types before touching the vault.

1. **Disable ASR replication first**:
   - Portal → **Site Recovery** → **Replicated items** → `vm-bc-lab` → **Disable Replication** → confirm
   - Wait for the item to disappear from **Replicated items** (this deletes the replicated disks and configuration in the target region)
   - If a test failover was left running, run **Cleanup test failover** before disabling replication

2. **Stop and disable Azure Backup, then delete the backup data**:
   ```bash
   az backup protection disable \
     --resource-group rg-az305-lab3 \
     --vault-name rsv-az305-lab3 \
     --container-name vm-bc-lab \
     --item-name vm-bc-lab \
     --backup-management-type AzureIaasVM \
     --delete-backup-data true \
     --yes
   ```
   `--delete-backup-data true` is required — without it, backup is stopped but recovery points are retained (and continue billing) per the retention policy.

3. **Confirm the vault has no remaining protected items**:
   ```bash
   az backup item list \
     --resource-group rg-az305-lab3 \
     --vault-name rsv-az305-lab3 \
     --output table
   ```
   Should return empty. Also check the **Site Recovery** → **Replicated items** blade in the portal is empty.

4. **Delete the vault**:
   ```bash
   az backup vault delete \
     --resource-group rg-az305-lab3 \
     --name rsv-az305-lab3 \
     --yes
   ```
   If this fails with a "vault has associated resources" error, something from steps 1-2 wasn't fully removed — check for leftover backup fabric/container registrations in the vault's **Manage** → **Site Recovery Infrastructure** blade.

5. **Delete the VM and resource group** (this also removes the ASR target-region resources if they were placed in the same resource group, or delete the target resource group separately if the wizard created one there):
   ```bash
   az group delete --name rg-az305-lab3 --yes --no-wait
   ```

6. If ASR auto-created a target resource group in the secondary region, delete it explicitly:
   ```bash
   az group list --query "[?starts_with(name, 'rg-az305-lab3-asr')].name" --output table
   az group delete --name <target-asr-resource-group> --yes --no-wait
   ```

---

## Key Concepts

| Term | Definition |
|---|---|
| **RTO (Recovery Time Objective)** | Maximum acceptable time to restore service after an outage — drives your failover *automation and orchestration* design |
| **RPO (Recovery Point Objective)** | Maximum acceptable data loss, measured in time since the last good recovery point — drives your replication/backup *frequency* design |
| **Azure Backup** | Point-in-time, retained copies of data for operational recovery (deletion, corruption, ransomware) — not a regional failover mechanism |
| **Azure Site Recovery (ASR)** | Continuous replication of VM disks to a secondary region with orchestrated, on-demand failover — the regional DR mechanism |
| **Availability Zone (AZ)** | Physically separate datacenter within the same Azure region, connected by low-latency networking — protects against datacenter-level failure, not regional failure |
| **Paired Region** | A predefined secondary Azure region tied to a primary (e.g., East US ↔ West US) with staggered platform updates and prioritized recovery — the target for cross-region DR |
| **Test failover** | A non-disruptive ASR failover into an isolated network that validates DR readiness without affecting production or live replication |
| **Planned/unplanned failover** | A real failover that moves production traffic to the secondary region — planned allows a graceful shutdown and zero data loss sync first, unplanned is triggered during an actual outage and accepts the replication engine's last-known state |

---

## Common Mistakes
- **Treating Azure Backup as a DR solution**: it protects data, not availability — it has no answer for "the region is gone"
- **Assuming GRS/GZRS storage redundancy alone satisfies a compute RTO**: replicated bytes in a second region are not a running application; you still need compute failover
- **Trying to delete the Recovery Services vault before removing protected items**: both ASR replicated items and Backup items with retained recovery points must be removed first, or vault deletion fails
- **Never running a test failover until the real outage happens**: an unvalidated DR plan routinely fails on details (missing NSG rules, wrong DNS, forgotten dependent resources) that only surface when you actually test it
- **Leaving a test failover running**: the cleanup step is not optional — the test VM bills like any other running VM until you clean it up

---

## Next Steps
This lab assumed comfort with VM basics from [AZ-104 Lab 2: Compute](../AZ-104/lab-2-compute.md); pair it with [AZ-104 Lab 8: Break Things](../AZ-104/lab-8-break-things.md) for hands-on chaos/failure-mode practice that reinforces why untested DR plans fail in the specific ways this lab's test-failover step is designed to catch. Continue the AZ-305 track backward to [Lab 2: Data Storage Design](lab-2-data-storage-design.md) for the redundancy-tier reasoning this lab builds on, or forward to [Lab 4: Compute & Application Infrastructure Design](lab-4-compute-app-infrastructure-design.md) for hosting-model decisions.
