# Lab 3: Storage & Data Management

## Overview
This lab covers Azure Storage accounts (blobs, tables, queues), lifecycle policies, replication, access tiers, and disaster recovery concepts. You'll configure a realistic storage scenario with tiering and backups.

**Estimated time**: 45–60 minutes  
**Cost**: ~$0.50 (minimal storage if you clean up promptly)

---

## Objectives
- Create and configure storage accounts
- Understand blob tiers (hot/cool/archive) and lifecycle policies
- Configure replication (LRS, GRS, RA-GRS)
- Implement soft delete and versioning
- Test access policies and shared access signatures (SAS)

---

## Part 1: Create Storage Account

### Step 1: Create Storage Account
1. Go to **Azure Portal** → search **Storage accounts** → **+ Create**
2. **Basics tab:**
   - **Resource Group**: Create new → `az104-lab3-rg`
   - **Storage account name**: `labstg<randomnumber>` (must be globally unique, lowercase, 3-24 chars)
     - Example: `labstg20240311`
   - **Region**: `East US`
   - **Performance**: `Standard` (Premium is for VMs with high IOPS)
   - **Redundancy**: `Geo-redundant storage (GRS)`
     - *Why GRS?* Replicates data to a secondary region. Good for HA scenarios.

3. **Advanced tab:**
   - **Require secure transfer**: `Enabled` (forces HTTPS)

4. **Data protection tab**
   - **Blob soft delete**: `Enabled` (lets you recover deleted blobs for 7 days)
   - **Container soft delete**: `Enabled`
   - **Versioning**: `Enabled` (tracks blob history)
   - Leave other defaults

5. **Review + create** → **Create**

**Wait ~1 minute**

---

## Part 2: Create Blob Containers and Upload Data

### Step 2: Create Containers
1. Open the storage account → Left sidebar → **Containers** (under Data storage)
2. Click **+ Container**
3. **Create container #1:**
   - **Name**: `app-logs`
   - **Public access level**: `Private`
4. Click **Create**

5. Repeat for **Container #2:**
   - **Name**: `backups`
   - **Public access level**: `Private`

6. Repeat for **Container #3:**
   - **Name**: `archive`
   - **Public access level**: `Private`

**Why three containers?** Separate concerns: logs are temporary, backups are recovery-critical, archive is long-term cold storage.

---

## Part 3: Upload Sample Data and Test Tiers

### Step 3: Upload Sample Files
1. Open `app-logs` container
2. Click **Upload**
3. **Create a local test file**:
   - On your machine, create a text file called `test-log.txt`
   - Add some content: `2024-03-11 10:00:00 - Application started`
4. Select this file and upload it

5. Repeat: Upload the same file to `backups` container (simulating a backup)

**Validate**: Both containers should show 1 blob each.

---

## Part 4: Configure Lifecycle Management

### Step 4: Set Up Lifecycle Policies
1. Go back to storage account → Left sidebar → **Lifecycle management** (under Data management)
2. Click **+ Add a rule**
3. **Add rule:**
   - **Rule name**: `LogsToArchive`
   - **Rule type**: `Base blobs` (this applies to current blob versions)

4. **If** section:
   - **Blob type**: Check `Block blobs`
   - **Blob subtype**: Leave unchecked
   - **Blob container**: `Specify containers` → Select `app-logs`
   - **Days since blob was last modified**: `30`

5. **Then** section:
   - Check **Move to cool storage**: Days after last modification: `30`
   - Check **Move to archive storage**: Days after last modification: `90`
   - Check **Delete the blob**: Days after last modification: `180`

6. Click **Add**

**What this does**:
- Blobs in `app-logs` after 30 days → Cool tier (cheaper, slower access)
- After 90 days → Archive tier (cheapest, slowest, long-term retention)
- After 180 days → Auto-delete

**In exam terms**: This is how you optimize costs for time-series data (logs, metrics, backups).

---

## Part 5: Configure Access Tiers Manually

### Step 5: Change Blob Tier
1. Open `backups` container → Click the blob you uploaded
2. Click **Change tier** (top menu)
3. **Set tier to**: `Cool`
4. Click **Save**

**Why cool instead of hot?** Backups are accessed infrequently but need quick retrieval (unlike archive). Cool saves ~50% on storage costs.

---

## Part 6: Test Shared Access Signature (SAS)

### Step 6: Generate SAS for Blob
1. Open the blob in `backups` container
2. Click **Generate SAS** (top menu)
3. **SAS token parameters:**
   - **Signing method**: `Storage account key`
   - **Permissions**: Check `Read`
   - **Start time**: Leave default (now)
   - **Expiry time**: `1 hour from now`
   - **IP addresses**: Leave blank (any IP can use it)
   - **Protocol**: `HTTPS only`
4. Click **Generate SAS token and URL**

**Copy the Blob SAS URL** (the long URL at the bottom)

### Step 7: Test SAS Access
1. Open a new **incognito/private browser window**
2. Paste the SAS URL
3. You should see the file content (it's publicly accessible via the SAS token, not the storage account key)
4. Close the incognito window

**Why SAS?** You can grant temporary, limited access to blobs without sharing storage account keys.

---

## Part 7: Configure Replication and Failover

### Step 8: Check Replication Status
1. Go back to storage account → **Redundancy** (left sidebar, under Settings)
2. You should see:
   - **Primary region**: `East US`
   - **Replication type**: `Geo-redundant storage (GRS)`
   - **Secondary region**: `West US` (auto-paired)

3. Look for **Last sync time** (shows when data was last replicated to secondary)

**Note**: GRS replicates data asynchronously. You can't failover manually; Microsoft handles it if primary fails.

### Step 9: Upgrade to RA-GRS (Optional, Not Recommended for Lab)
1. In **Redundancy** settings, click **Change replication type**
2. Select `Read-access geo-redundant storage (RA-GRS)`
3. Click **Save**

**Why not do this?** RA-GRS costs more (~2x). For the exam, understand that RA-GRS lets you *read* from the secondary region during an outage.

---

## Part 8: Test Soft Delete and Versioning

### Step 10: Delete and Recover a Blob
1. Open `app-logs` container
2. Right-click the blob (or click three-dot menu) → **Delete**
3. Confirm deletion

4. Now, in the container, click **Show deleted blobs** (toggle at top)
5. You'll see the deleted blob in a grayed-out state
6. Right-click it → **Undelete**

**Validate**: The blob reappears.

**Why soft delete?** Protects against accidental deletion or malicious attacks. Without it, deleted data is gone.

### Step 11: View Blob Versions
1. Open `backups` container → Click the blob
2. In the blob overview, look for **Versions** section
3. Click **Version history** or look for multiple versions listed

**What you'll see**: Each time the blob was modified, a new version was created (if you uploaded multiple times, you'd see multiple versions).

---

## Part 9: Restrict Public Network Access

### Step 12: Limit Access to Your IP Only
1. Go to storage account → **Networking** (left sidebar, under Security + networking)
2. On the **Firewalls and virtual networks** tab, find **Public network access:**
   - Select **Enabled from selected virtual networks and IP addresses**
3. Scroll down to **Firewall** → **Address range:**
   - Click **Add your client IP address** (it auto-detects and fills in your public IP)
4. Scroll down to **Exceptions:**
   - Check **Allow Azure services on the trusted services list to access this storage account**
5. Click **Save** at the top

**What happens**: Only traffic from your IP can access this storage account. All other public internet traffic is blocked. Azure-internal services (like Azure Backup, Monitor) are still allowed via the trusted services exception.

**Validate**: Copy the SAS URL from Part 6 and paste it in your browser — it should still work because your IP is allowed. If you tried from a different network (e.g., mobile hotspot), it would be denied.

> **Note**: If you don't see the **Enabled from selected virtual networks and IP addresses** option, your subscription or account type may restrict network rule changes. You can read-only review the setting and move on — the exam concept is what matters.

---

## Part 10: Understanding Access Keys and Connection Strings

### Step 13: Inspect Access Keys
1. Go to storage account → **Access keys** (left sidebar, under Security + networking)
2. You'll see:
   - **Storage account name**
   - **Key1** and **Key2** (two keys for rotation)
   - **Connection string**

**Important**: These are secrets. Never commit them to GitHub or share them.

### Step 14: Understand the Difference
- **Storage account key**: Allows full access to all data in the account
- **SAS token**: Allows scoped, time-limited access (specific blob, specific permissions)
- **Connection string**: Combines account name + key for SDKs to authenticate

---

## Part 11: Validation & Testing

### Step 15: Verify Your Setup
1. Go to **Storage accounts dashboard** → Click your account
2. Check:
   - **Containers**: 3 containers (`app-logs`, `backups`, `archive`)
   - **Blobs**: At least 1 blob in `backups` and `app-logs`
   - **Replication**: Shows GRS
   - **Soft delete**: Enabled
   - **Versioning**: Enabled
   - **Lifecycle policies**: 1 rule configured

3. Optional: Download a blob via Azure Portal to confirm read access works

---

## Part 12: Cleanup

1. Go to **Resource groups** → Click `az104-lab3-rg`
2. Click **Delete resource group**
3. Type the resource group name
4. Click **Delete**

**Wait 1–2 minutes** for all storage resources to delete.

---

## Key Concepts to Understand

| Concept | What It Does |
|---------|-------------|
| **Hot tier** | Frequently accessed data; highest cost to store, lowest cost to access |
| **Cool tier** | Infrequently accessed; lower storage cost, higher access cost (min 30-day retention) |
| **Archive tier** | Rarely accessed; lowest storage cost, very high access latency (min 180-day retention) |
| **Lifecycle policy** | Automatically moves blobs between tiers or deletes them based on age |
| **GRS** | Geo-redundant storage; async replication to secondary region (can't read secondary) |
| **RA-GRS** | Read-access GRS; you can read from secondary during outage (pricier) |
| **Soft delete** | Deleted blobs recoverable for N days (7–365) |
| **Versioning** | Tracks all blob modifications; can restore previous versions |
| **SAS** | Shared Access Signature; scoped, time-limited access token |

---

## Exam Tips
- **Tier retention minimums**: Cool (30 days), Archive (180 days) — you'll see exam questions about these
- **GRS sync timing**: Not instant; RTO/RPO numbers matter for your RTO requirements
- **Lifecycle policies are powerful**: They're how you optimize CapEx/OpEx for large datasets
- **Access tiers and lifecycle**: You can't move blobs *out* of archive to hot via lifecycle (must do manually)
- **Blob snapshots vs versioning**: Snapshots are point-in-time; versioning tracks all changes

---

## Next Steps
- Set up Azure Blob Storage as a backup target for VMs (from Lab 2)
- Use Azure Table Storage for time-series data or NoSQL workloads
- Implement Azure Queue Storage for async messaging
- Configure Storage Account encryption with customer-managed keys
