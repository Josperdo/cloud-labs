# Lab 3: Storage — Setting Up Persistent Storage for an Application

## Overview
Most applications need somewhere to store files — user uploads, application logs, backups, exports. Azure Blob Storage handles all of that, but getting it right from day one means thinking about cost management (old files should automatically get cheaper to store), data protection (accidental deletions should be recoverable), and access control (not everyone who needs to read a file should get the master key). This lab sets up a storage account the way you'd do it for a real application.

**Estimated time**: 45–60 minutes
**Cost**: ~$0.50 (minimal; tear down when done)

---

## Scenario
Your application needs storage for three things: user-uploaded files (accessed frequently), application logs (written constantly, but rarely read after 30 days), and daily database backups (kept for 90 days, then discarded). You need to set this up with the right cost profile, make sure mistakes are recoverable, and give a contractor temporary read access to a specific file without handing over the storage account keys.

---

## Objectives
- Create a storage account with the right settings from the start
- Organize data into containers by purpose
- Set up lifecycle policies to automatically manage storage costs
- Enable soft delete and versioning so files can be recovered
- Generate a SAS token to grant temporary access without sharing credentials
- Lock down the storage account to block public internet access

---

## Part 1: Create the Storage Account

### Step 1: Create the Storage Account
The settings you choose here affect cost, durability, and security for the lifetime of the account — it's easier to get them right upfront than to change them later.

1. Search **Storage accounts** → **+ Create**
2. **Basics tab:**
   - **Resource Group**: Create new → `app-storage-rg`
   - **Storage account name**: `appfiles<randomnumber>` *(globally unique, lowercase, 3–24 chars)*
   - **Region**: `East US`
   - **Performance**: `Standard` (Premium is for VM disks and high-throughput scenarios; not needed for blob storage)
   - **Redundancy**: `Geo-redundant storage (GRS)`
     - *Why GRS?* Your data is replicated to a secondary region. If the primary region has an outage, your data still exists. For most production apps, GRS is the minimum acceptable option.

3. **Advanced tab:**
   - **Require secure transfer**: `Enabled` (forces HTTPS; never allow HTTP access to storage)
   - **Minimum TLS version**: `TLS 1.2`
   - **Allow Blob public access**: `Disabled` (prevents anyone from accidentally making a container publicly readable)

4. **Data protection tab:**
   - **Blob soft delete**: `Enabled`, retention: `14 days`
   - **Container soft delete**: `Enabled`, retention: `14 days`
   - **Versioning**: `Enabled`
   - *Why 14 days?* Mistakes are usually caught within a day or two, but 14 days gives you a buffer without dramatically increasing storage costs from deleted-but-retained blobs.

5. **Review + create** → **Create**

---

## Part 2: Create Containers for Each Purpose

### Step 2: Create the Application Containers
Containers are like top-level folders. Organizing by purpose lets you apply different lifecycle policies to each one.

1. Open the storage account → Left sidebar → **Containers** → **+ Container**
2. **Container 1 — user uploads:**
   - **Name**: `user-uploads`
   - **Public access level**: `Private`
   - Click **Create**

3. **Container 2 — application logs:**
   - **Name**: `app-logs`
   - **Public access level**: `Private`
   - Click **Create**

4. **Container 3 — database backups:**
   - **Name**: `db-backups`
   - **Public access level**: `Private`
   - Click **Create**

**All three are private**: No public access, ever. Files are only readable via authenticated requests or SAS tokens you explicitly generate.

### Step 3: Upload Test Files
You need some data to work with.

1. Create a small text file on your machine called `test-upload.txt` with any content
2. Open `user-uploads` container → Click **Upload** → Select the file → **Upload**
3. Open `app-logs` container → **Upload** the same file (simulating a log file)
4. Open `db-backups` container → **Upload** the same file (simulating a backup)

---

## Part 3: Set Up Lifecycle Policies

### Step 4: Create Lifecycle Rules to Manage Costs
Without lifecycle policies, blobs stay in the hot tier indefinitely and you pay full price even for files that haven't been touched in months. You're going to set up rules that automatically move data to cheaper tiers as it ages.

**The storage tiers:**
- **Hot**: Files accessed frequently; highest storage cost, lowest access cost
- **Cool**: Files accessed occasionally; lower storage cost, higher access cost (minimum 30-day commitment)
- **Cold**: Files rarely accessed; even lower storage cost (minimum 90-day commitment)
- **Archive**: Files almost never accessed; cheapest storage, but takes hours to retrieve (minimum 180-day commitment)

1. Go to storage account → Left sidebar → **Lifecycle management** → **+ Add a rule**

**Rule 1 — App logs tiering:**
- **Rule name**: `logs-tiering`
- **Rule type**: `Base blobs`
2. **Filter set:**
   - **Blob prefix**: `app-logs/` *(applies only to blobs in this container)*
3. **Base blobs tab:**
   - Move to cool storage: `30` days since last modification
   - Move to archive storage: `90` days since last modification
   - Delete the blob: `180` days since last modification
4. Click **Add**

**Rule 2 — Database backup retention:**
1. **+ Add a rule**
   - **Rule name**: `backup-retention`
2. **Filter set:**
   - **Blob prefix**: `db-backups/`
3. **Base blobs tab:**
   - Move to cool storage: `7` days since last modification
   - Move to archive storage: `30` days since last modification
   - Delete the blob: `90` days since last modification
4. Click **Add**

**What you've set up**: Logs automatically get cheaper as they age and are deleted after 6 months. Backups move to archive after 30 days and are deleted after 90. The `user-uploads` container has no lifecycle rule — those files stay in hot tier until the application deletes them.

---

## Part 4: Verify Data Protection Settings

### Step 5: Test Soft Delete (Simulating an Accidental Deletion)
Soft delete is your safety net. It means deleted blobs aren't actually gone — they're retained for the number of days you configured.

1. Open `app-logs` container → Click the three-dot menu on your test file → **Delete** → **OK**
2. The blob disappears from the normal view
3. Click the **Show deleted blobs** toggle at the top of the container view
4. Your deleted blob reappears (grayed out)
5. Click the three-dot menu → **Undelete**
6. Toggle off **Show deleted blobs** — the file is back

**This is one of the first things you'd check** if a developer says "I accidentally deleted a file from storage." Without soft delete enabled, the answer would be "it's gone."

### Step 6: Check Blob Versioning
Versioning keeps a history of every change to a blob. If someone overwrites a file with corrupt data, you can restore the previous version.

1. Upload the same `test-upload.txt` file to `user-uploads` again (overwriting the existing one)
2. Click on the blob → Look for **Versions** in the blade or the version history section
3. You should see two versions: the original upload and the overwrite

**Real scenario**: A data processing pipeline overwrites a file with empty data due to a bug. With versioning, you roll back to the last good version. Without it, the data is gone.

---

## Part 5: Grant Temporary Access with a SAS Token

### Step 7: Generate a Shared Access Signature
A contractor needs to read a specific file from your storage account. You could give them the storage account key — but that grants full access to everything, forever, and you'd have to rotate the key when they're done. A SAS token is scoped to exactly what they need and expires automatically.

1. Open `db-backups` container → Click the test blob
2. In the blob overview, click **Generate SAS** at the top
3. **Configure the SAS:**
   - **Signing method**: `Storage account key`
   - **Permissions**: Check `Read` only (uncheck everything else)
   - **Start time**: Leave as now
   - **Expiry time**: Set to 24 hours from now
   - **Allowed protocols**: `HTTPS only`
4. Click **Generate SAS token and URL**
5. Copy the **Blob SAS URL** (the long URL at the bottom)

### Step 8: Test the SAS URL
1. Open a private/incognito browser window
2. Paste the SAS URL — you should see or download the file
3. Now try modifying the URL by removing some characters from the token — access is denied
4. Close the incognito window

**Why this matters**: The contractor gets exactly what they need: read access to that one file, for 24 hours. When it expires, the URL stops working. You never shared your account key, and there's nothing to rotate when they're done.

---

## Part 6: Lock Down Network Access

### Step 9: Restrict the Storage Account to Your IP
By default, storage accounts accept traffic from anywhere on the internet. For a production app, you'd restrict this to specific VNets or IPs.

1. Go to storage account → Left sidebar → **Networking**
2. **Public network access**: Select **Enabled from selected virtual networks and IP addresses**
3. Under **Firewall** → **Address range**: Click **Add your client IP address**
   *(Azure auto-fills your current public IP)*
4. Under **Exceptions**: Check **Allow Azure services on the trusted services list to access this storage account**
   *(This lets Azure Monitor, Azure Backup, and similar services still reach the account)*
5. Click **Save**

**What "trusted services exception" means**: Without this, enabling the firewall would break Azure's own monitoring and backup features. The trusted services list is controlled by Microsoft and only includes Azure's own internal services.

**In a real deployment**: You'd add the VNet and subnet where your application runs, rather than your public IP. That way only your app servers can reach storage — not anyone on the internet, and not even you unless you're on the office VPN.

---

## Part 7: Understand Access Keys and When Not to Use Them

### Step 10: Inspect Access Keys
1. Go to storage account → Left sidebar → **Access keys**
2. You'll see **key1** and **key2**

**Key1 and key2 exist for rotation**: You update your apps to use key2, then regenerate key1. That way there's no downtime while rotating credentials.

**When to use each access method:**

| Method | When to use |
|--------|------------|
| **Managed identity** | Your app runs in Azure (VM, App Service, etc.) — no credentials needed |
| **SAS token** | Temporary, scoped access for external users or short-lived needs |
| **Access key** | Last resort; avoid; full access, doesn't expire, hard to audit |
| **Entra ID (RBAC)** | Internal users or services that need ongoing access |

The cardinal rule: **never commit an access key to git or put it in an unencrypted config file.**

---

## Part 8: Cleanup

1. **Resource groups** → `app-storage-rg` → **Delete resource group**
2. Confirm with the resource group name → **Delete**

**Wait 1–2 minutes** for the storage account and all blobs to be deleted.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|--------------------------|
| **GRS replication** | Data survives a regional outage; minimum bar for most production apps |
| **Disabling public blob access** | Prevents accidental public exposure of private files |
| **Lifecycle policies** | Storage costs grow fast without automation; logs and backups shouldn't stay in hot tier forever |
| **Soft delete** | Recovers from accidental deletion without a support ticket |
| **Versioning** | Recovers from overwrites, which soft delete doesn't protect against |
| **SAS tokens** | Grants time-limited access without sharing the account key |
| **Network restrictions** | Blocks public internet access to storage; only your app can reach it |

---

## Common Mistakes to Avoid
- **Leaving "Allow Blob public access" enabled**: A developer can accidentally create a public container and expose files to the internet; disable this at the account level
- **Storing access keys in code or config files**: Keys get committed to git; use managed identities or Key Vault references instead
- **No lifecycle policy on logs**: Log containers can grow to hundreds of GB/month without automated tiering and deletion
- **Skipping soft delete**: The first time a developer accidentally deletes something in production, you'll wish you had it; it costs almost nothing to enable
- **Generating SAS tokens that never expire**: A SAS token is a credential; if it leaks, it works until it expires — set short expiry times for sensitive resources
- **LRS for production data**: Locally redundant storage only keeps three copies within a single datacenter; a datacenter failure loses your data

---

## Next Steps
- Configure a storage account private endpoint so it's only reachable from within your VNet (no public IP at all)
- Use Azure Key Vault to store the storage account connection string instead of hardcoding it
- Set up Azure Backup to take snapshots of your storage account on a schedule
- Write a script using the Azure CLI or Python SDK to upload files programmatically using a managed identity
- Enable immutable storage on the backups container so that backups can't be modified or deleted before their retention period expires
