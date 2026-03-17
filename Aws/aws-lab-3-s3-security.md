# AWS Lab 3: S3, Security & Lifecycle Management

Check box if done: []

## Overview
This lab mirrors Azure Lab 3 (storage) but on AWS. You'll create S3 buckets, configure access controls, implement lifecycle policies, and practice security hardening. S3 is the most important AWS service—it's *everywhere*.

**Estimated time**: 3–4 hours  
**Cost**: ~$0.25 (minimal storage; tear down at the end)  
**Key difference from Azure**: Azure has "containers" (blob storage). AWS has "buckets" (S3). Same concept, different terminology.

---

## Objectives
- Create S3 buckets with proper naming and configuration
- Understand S3 access control (bucket policies, ACLs, IAM)
- Configure lifecycle policies for cost optimization
- Implement versioning and MFA delete
- Practice "least privilege" access patterns
- Use S3 for multiple purposes (app data, backups, logs)

---

## Part 1: Create S3 Buckets with Security

### Step 1: Create First S3 Bucket
1. Go to **AWS Console** → Search **S3** → Click **Buckets** → **Create bucket**
2. **General configuration:**
   - **Bucket name**: `lab3-app-data-<your-random-suffix>`
     - *Important: S3 names are globally unique. Add random suffix to avoid conflicts.*
     - Example: `lab3-app-data-20240311xyz`
   - **AWS Region**: `us-east-1`
   - **Copy settings from existing bucket**: Leave unchecked

3. **Object Ownership**:
   - Select **Bucket owner preferred** (you own all objects)

4. **Block Public Access settings**:
   - Check **Block all public access** (secure by default)
   - This prevents accidental public exposure

5. **Bucket versioning**:
   - **Enable** versioning (track object history)

6. **Server-side encryption**:
   - Select **Enable** with **Amazon S3-managed keys (SSE-S3)**
   - *Encryption at rest; data is encrypted when stored*

7. Click **Create bucket**

**What you've created**: A secure S3 bucket with versioning and encryption. All traffic is private by default.

### Step 2: Create Second Bucket (for backups)
1. Repeat Step 1:
   - **Bucket name**: `lab3-backups-<your-random-suffix>`
   - Same security settings (Block Public Access, versioning, encryption)

---

## Part 2: Upload and Manage Objects

### Step 3: Upload Objects
1. Open `lab3-app-data-*` bucket
2. Click **Upload**
3. **Drag and drop** or select a file (use a text file, image, or anything ~1KB)
4. Click **Upload**

5. Repeat: Upload the same file to `lab3-backups-*` bucket

**Validation**: Both buckets have at least 1 object.

### Step 4: Understand Object Versioning
1. Open `lab3-app-data-*` → Click on the object you uploaded
2. **Versions** tab → You should see at least 1 version (the one you just uploaded)
3. Go back to bucket → **Upload** → Select the same file again (no changes needed)
4. Click **Upload** → Confirm

5. Click the object again → **Versions** tab
   - Now you should see 2 versions (the old one and the new one)

**Why versioning?** If you accidentally overwrite or delete an object, you can restore an older version. Costs extra storage (you pay for all versions), but worth it for critical data.

---

## Part 3: Configure Access Control (Security Deep Dive)

### Step 5: Create Bucket Policy (Allow Specific User Read Access)
1. Open `lab3-app-data-*` bucket → **Permissions** tab → **Bucket policy**
2. Click **Edit**
3. Paste this policy (replace `BUCKET_NAME` with your actual bucket name):
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowReadFromIAMUser",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:amakemiam::ACCOUNT_ID:user/prod-engineer"
         },
         "Action": [
           "s3:GetObject",
           "s3:ListBucket"
         ],
         "Resource": [
           "arn:aws:s3:::lab3-app-data-*",
           "arn:aws:s3:::lab3-app-data-*/*"
         ]
       }
     ]
   }
   ```
   - Replace `ACCOUNT_ID` with your AWS Account ID (12-digit number; find it in top-right menu → **Account ID**)
   - Replace `lab3-app-data-*` with your actual bucket name

4. Click **Save changes**

**What this does**: Allows the IAM user `prod-engineer` (from Lab 1) to list and read objects in this bucket. No one else can.

### Step 6: Deny Public Access (Even More Secure)
1. Still in **Permissions** tab
2. Check that **Block all public access** is enabled (it should be from creation)

**Why "Block all public access"?** Even if someone accidentally adds a public policy, AWS blocks it. Defense in depth.

---

## Part 4: Lifecycle Policies for Cost Optimization

### Step 7: Create Lifecycle Policy
1. **Backups bucket** → **Management** tab (not Permissions) → **Create lifecycle rule**
2. **Lifecycle rule configuration:**
   - **Lifecycle rule name**: `BackupToGlacier`
   - **Scope**: Apply to all objects in the bucket
   - **Transitions** section:
     - Check **Transition current versions**
     - **Days after object creation**: `30` → **Storage class**: `Standard-IA` (Infrequent Access)
     - Click **Add transition**
     - **Days after object creation**: `90` → **Storage class**: `Glacier Instant Retrieval`
   - **Expiration** section:
     - Check **Expire current versions**
     - **Days after object creation**: `365` (delete after 1 year)
   - **Manage noncurrent versions** (optional for this lab):
     - Can transition/delete old versions too
     - Skip for simplicity

3. Click **Create rule**

**What this does**:
- Day 0–30: Objects stored in Standard (expensive, fast access)
- Day 30–90: Moved to Standard-IA (cheaper, slower access)
- Day 90–365: Moved to Glacier (very cheap, very slow, for archives)
- Day 365+: Deleted

**Cost impact**: Standard ≈ $0.023/GB/month. Glacier ≈ $0.004/GB/month. You save ~80% after 90 days.

**Interview question this preps you for**: "How would you optimize storage costs for log data that's accessed frequently for 30 days, then rarely?"

---

## Part 5: Compare AWS S3 to Azure Storage

### Step 8: Mental Model (Interview Prep)

**If someone asks: "What's the difference between AWS S3 and Azure Blob Storage?"**

Your answer:
- **Both** are object storage (unstructured data: files, images, videos, logs)
- **S3**: Organizes data in buckets. Tiers: Standard, Standard-IA, Glacier, Deep Archive. Lifecycle policies automatically move objects.
- **Azure Blob**: Organizes data in containers. Tiers: Hot, Cool, Archive. Lifecycle policies work similarly.
- **Key difference**: AWS has *4 tiers* (more granular pricing). Azure has *3 tiers*. Both optimize for cost.

**Comparison table**:

| Aspect | AWS S3 | Azure Blob |
|--------|--------|-----------|
| **Container** | Bucket | Container |
| **Access tiers** | Standard, Standard-IA, Glacier, Deep Archive | Hot, Cool, Archive |
| **Encryption** | SSE-S3, SSE-KMS, SSE-C | AES-256, customer-managed keys |
| **Versioning** | Built-in, per-bucket | Snapshots + versioning (separate) |
| **Lifecycle rules** | Transitions + expiration | Transitions + deletion |
| **Access control** | Bucket policies + IAM + ACLs | Azure RBAC + SAS tokens |
| **Pricing** | Pay-per-use, Standard > IA > Glacier | Similar tiering, slightly different rates |

---

## Part 6: Create S3 Presigned URL (Like Azure SAS)

### Step 9: Generate Presigned URL
1. Open `lab3-app-data-*` bucket → Click on an object
2. **Object actions** → **Share with a presigned URL**
3. **Duration**: `1 hour`
4. Click **Create presigned URL**
5. Copy the URL

**What this is**: A time-limited, read-only URL that anyone can use to access the object (without AWS credentials).

### Step 10: Test Presigned URL
1. Open a new **incognito/private browser window**
2. Paste the presigned URL
3. The object should download or display (depending on file type)
4. After 1 hour, the URL expires and anyone who tries will get "Access Denied"

**Why this matters**: You can share objects with external users without giving them AWS credentials. Like Azure's SAS tokens.

---

## Part 7: S3 Bucket Policy for Public Read (Learn Why NOT To Do This)

### Step 11: Create Public Read Policy (Educational Only)
1. Open `lab3-app-data-*` bucket → **Permissions** → **Bucket policy**
2. Click **Edit** and paste:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "PublicRead",
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:GetObject",
         "Resource": "arn:aws:s3:::lab3-app-data-*/*"
       }
     ]
   }
   ```
3. Click **Save changes**

**Important warning**: This makes the bucket publicly readable. In production, you'd only do this for static websites or public assets (never sensitive data).

### Step 12: Test Public Access
1. Open a new incognito window
2. Go to: `https://lab3-app-data-YOUR-BUCKET.s3.amazonaws.com/your-file-name`
3. The file should display or download

**Key insight**: Anyone on the internet can now access this file. This is why "Block all public access" is the safe default.

### Step 13: Revert to Private
1. Delete the public bucket policy:
   - Go back to **Bucket policy** → Clear the JSON → Click **Save changes**

**Always remember**: Public access is a liability. Use presigned URLs instead.

---

## Part 8: S3 Access Logging (Audit Trail)

### Step 14: Enable S3 Access Logs
1. Open `lab3-app-data-*` bucket → **Properties** tab (not Permissions or Management)
2. Scroll down → **Server access logging**
3. Click **Edit**
4. Check **Enable**
5. **Target bucket**: `lab3-backups-*` (logs go to a separate bucket)
6. **Target prefix**: `logs/` (organize logs with a prefix)
7. Click **Save changes**

**What this does**: Every access to `lab3-app-data-*` (GET, PUT, DELETE) is logged to the backups bucket. You can audit who accessed what and when.

**Interview question**: "How would you audit who accessed sensitive data in S3?"

---

## Part 9: S3 Server Access Logs vs CloudTrail (Know the Difference)

### Step 15: Understand Logging Options

| Log Type | What It Logs | Storage Location |
|----------|-------------|------------------|
| **S3 Access Logs** | Object-level operations (GET, PUT, DELETE, HEAD) | Another S3 bucket |
| **CloudTrail** | AWS API calls (bucket creation, policy changes, etc.) | CloudTrail or S3 bucket |
| **S3 Object Lambda** | Custom transformations on object access | N/A for this lab |

**Example**: 
- User uploads a file → S3 Access Logs record it
- User changes bucket policy → CloudTrail records it
- User deletes a file → S3 Access Logs record it

---

## Part 10: Cleanup

1. **Empty buckets** (you must do this before deleting):
   - Open `lab3-app-data-*` → **Select all** → **Delete**
   - Open `lab3-backups-*` → **Select all** → **Delete**

2. **Delete buckets**:
   - **Buckets** → Select both buckets → **Delete**
   - Type the bucket name to confirm
   - Click **Delete bucket**

**Wait 1–2 minutes** for all resources to delete.

---

## Key Concepts to Understand

| Concept | What It Does |
|---------|-------------|
| **S3 Bucket** | Container for objects (files); named globally unique |
| **Object** | A file (key) with data and metadata |
| **Storage Class** | Pricing tier: Standard (fast, expensive) to Deep Archive (slow, cheap) |
| **Lifecycle Policy** | Automatically transition objects between classes or delete them |
| **Bucket Policy** | JSON policy controlling bucket-level access |
| **IAM Policy** | Controls which IAM users can access S3 resources |
| **Presigned URL** | Time-limited, credential-less URL for sharing objects |
| **Versioning** | Tracks object history; allows rollback to older versions |
| **Access Logging** | Records all object operations to a separate bucket |
| **Block Public Access** | Security setting that prevents accidental public exposure |

---

## Interview Prep: Common Questions

**Q: "How would you architect S3 for a data lake?"**
A: Create separate buckets for raw data, processed data, and archives. Use lifecycle policies to move old data to Glacier. Enable versioning to prevent accidental deletes. Use bucket policies to restrict access by role.

**Q: "What's the difference between a bucket policy and an IAM policy?"**
A: Bucket policies control access *to the bucket* (anyone trying to access it). IAM policies control what *specific users* can do. Use IAM policies for employees, bucket policies for cross-account access or public scenarios.

**Q: "How would you secure sensitive data in S3?"**
A: Enable encryption (SSE-KMS with customer-managed keys). Enable versioning. Enable MFA Delete (requires MFA to delete objects). Use IAM policies for access control. Enable access logging. Block all public access.

**Q: "What are the S3 storage classes and when would you use each?"**
A: Standard (instant access), Standard-IA (infrequent access, retrieval fee), Glacier (archives, very slow), Deep Archive (long-term retention, slowest). Use lifecycle policies to automatically transition based on age.

---

## Next Steps
- Enable S3 Cross-Region Replication (automatic backup to another region)
- Set up S3 bucket versioning with MFA Delete
- Create an S3 event notification (triggers Lambda when objects are uploaded)
- Use S3 Select to query objects without downloading them
- Implement S3 server-side encryption with customer-managed keys (KMS)
