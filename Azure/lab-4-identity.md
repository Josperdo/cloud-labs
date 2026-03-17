# Lab 4: Access Management — Onboarding & Service Identities

Check box if done: [x]

## Overview
As a cloud engineer, you'll regularly be asked to onboard new team members, grant an application access to Azure resources, and audit who has access to what. This lab simulates three common real-world tasks: onboarding a new developer, setting up a managed identity so an app can authenticate without stored credentials, and verifying permissions before handing things off.

**Estimated time**: 50–65 minutes
**Cost**: ~$0 (IAM and Entra ID features are free)

---

## Scenario
Your team just hired a junior developer (**Alex**) who needs access to the development environment to do their job. Separately, you're deploying a background job that needs to read files from a storage account — and you need to set that up without hardcoding a password anywhere in the code. Finally, before you wrap up, you want to confirm that no one accidentally has more access than they should.

---

## Objectives
- Onboard a new user and give them scoped access to a dev environment
- Understand why you assign roles to groups instead of individuals
- Create a managed identity so an application can authenticate to Azure services
- Verify effective permissions before handing access off
- Practice revoking access (offboarding)

---

## Part 1: Set Up the Environment

### Step 1: Create Resource Groups
You'll use two resource groups to simulate a real environment — a dev workspace and a prod workspace. Most engineers on a team shouldn't have write access to prod.

1. Search **Resource groups** → **+ Create**
2. **Dev RG:**
   - **Name**: `team-dev`
   - **Region**: `East US`
   - Click **Create**

3. **Prod RG:**
   - **Name**: `team-prod`
   - **Region**: `East US`
   - Click **Create**

---

## Part 2: Onboard a New Developer

### Step 2: Create the User Account
In most companies, user accounts are created by IT or synced from Active Directory. In smaller orgs or startups, this falls on the cloud team.

1. Go to **Entra ID** → **Users** → **+ New user** → **Create new user**
2. **Fill in the details:**
   - **User principal name**: `alex.dev@yourdomain.onmicrosoft.com`
     *(Replace `yourdomain` with your actual Entra tenant domain)*
   - **Display name**: `Alex Dev`
   - **Password**: Check **Let me create the password** → set a temporary password
   - **Usage Location**: `United States`
3. Click **Create**

**What happens next in real life**: You'd send Alex the temporary password through a secure channel (not Slack or email in plaintext). Alex would be prompted to change it on first login.

### Step 3: Create a Group for the Dev Team
Instead of assigning roles directly to Alex, you assign them to a group. This way, when Alex leaves or another developer joins, you just update group membership — the role assignments don't change.

1. Go to **Entra ID** → **Groups** → **+ New group**
2. **Group details:**
   - **Group type**: `Security`
   - **Group name**: `dev-team`
   - **Description**: `Developers with access to the dev environment`
   - **Membership type**: `Assigned`
3. Click **Create**

### Step 4: Add Alex to the Group
1. Open the **dev-team** group → Left sidebar → **Members** → **+ Add members**
2. Search for `Alex Dev` → select → **Select**

### Step 5: Assign Alex's Team the Right Role
Alex needs to be able to create and manage resources in the dev environment, but not touch production.

1. Go to **Resource groups** → **team-dev**
2. Left sidebar → **Access Control (IAM)** → **+ Add** → **Add role assignment**
3. **Basics tab:**
   - **Role**: `Contributor`
   - **Assign access to**: `User, group, or service principal`
   - **Members**: Click **+ Select members** → search `dev-team` → select
4. Click **Review + assign** → **Assign**

**Why Contributor and not Owner?** Contributor can create and modify resources but can't change who has access. That's a meaningful security boundary — developers shouldn't be able to grant themselves or others additional permissions.

---

## Part 3: Set Up a Managed Identity for an Application

### Step 6: Why Managed Identities Exist
Imagine your app needs to read files from a storage account. The naive approach is to generate a storage account key and put it in your app's config file. The problem: keys get committed to git, shared in Slack, or left in old config files. They don't expire automatically.

A managed identity is an identity that Azure creates and manages for you. Your app uses it to authenticate, and Azure handles the credentials entirely — there's nothing to rotate, leak, or store.

### Step 7: Create a User-Assigned Managed Identity
1. Search **Managed Identities** → **+ Create**
2. **Basics:**
   - **Resource Group**: `team-dev`
   - **Region**: `East US`
   - **Name**: `background-job-identity`
3. Click **Review + create** → **Create**

**Why user-assigned instead of system-assigned?** A system-assigned identity is tied to a specific resource (like a VM) and deleted when that resource is deleted. User-assigned identities can be attached to multiple resources and have an independent lifecycle. For a shared service identity, user-assigned is the safer choice.

### Step 8: Create a Storage Account to Grant Access To
1. Search **Storage accounts** → **+ Create**
2. **Basics:**
   - **Resource Group**: `team-dev`
   - **Storage account name**: `devfiles<randomnumber>` *(must be globally unique, lowercase)*
   - **Region**: `East US`
   - **Redundancy**: `LRS`
3. Click **Review + create** → **Create**

### Step 9: Grant the Identity Access to Storage
Now you'll give the managed identity permission to read blobs from this storage account. This replaces what used to be a connection string or access key in the app's config.

1. Open the storage account → Left sidebar → **Access Control (IAM)** → **+ Add** → **Add role assignment**
2. **Role**: Search `Storage Blob Data Reader` → select it
3. **Members**: Click **+ Select members** → switch to **Managed identity** tab → select `background-job-identity`
4. Click **Review + assign** → **Assign**

**What happens in code**: The application would use the Azure SDK with `DefaultAzureCredential`, which automatically picks up the managed identity when running in Azure. No credentials in the code or config.

---

## Part 4: Verify Permissions Before Handoff

### Step 10: Check Alex's Effective Access
Before you hand off to Alex, confirm they actually have the access you intended — and nothing more.

1. Go to **Resource groups** → **team-dev**
2. Left sidebar → **Access Control (IAM)** → **Check access** tab
3. Search for `Alex Dev`
4. Confirm:
   - Alex has **Contributor** access to `team-dev` (via the dev-team group)
   - Alex does **not** have access to `team-prod`

5. Now check `team-prod` → **Access Control (IAM)** → **Check access**
6. Search `Alex Dev` — should show no role assignments

**This is a real step in real orgs.** Before onboarding someone, verify the permissions are scoped correctly. Mistakes here can give someone unintended access to production data.

### Step 11: Check the Managed Identity's Access
1. Open the storage account → **Access Control (IAM)** → **Role assignments** tab
2. You should see `background-job-identity` listed with the `Storage Blob Data Reader` role

### Step 12: Understand the Scope Hierarchy
Role assignments flow down from parent to child:

```
Subscription
  ├─ Resource Group: team-dev      ← dev-team has Contributor here
  │   ├─ Storage Account           ← background-job-identity has Blob Reader here
  │   └─ Other resources
  └─ Resource Group: team-prod     ← dev-team has NO access here
```

If you had assigned dev-team at the subscription level, they'd have access to everything — including prod. Always assign roles at the lowest scope that works.

---

## Part 5: Offboarding (Revoking Access)

### Step 13: Simulate Removing Access
When someone leaves the team, you remove them from their groups. Because roles are assigned to groups (not individuals), removing them from the group immediately revokes all the access that group granted.

1. Go to **Entra ID** → **Groups** → **dev-team** → **Members**
2. Select `Alex Dev` → **Remove**

Alex no longer has access to `team-dev`. You don't have to hunt down every role assignment across the subscription — they were all tied to the group.

**In practice**: If Alex's account is being deactivated entirely (they're leaving the company), you'd also disable or delete the user in Entra ID.

---

## Part 6: Optional — Enforce MFA for the Dev Team

### Step 14: Create a Conditional Access Policy
**Note**: This requires Entra ID Premium P1 or P2. If you don't have a premium license, skip to the cleanup section — this step is informational.

Conditional Access lets you say: "Before anyone in the dev team can access Azure, they must complete MFA." This is a standard security baseline in most companies.

1. Go to **Entra ID** → **Security** → **Conditional Access** → **+ New policy**
2. **Name**: `Require MFA — Dev Team`
3. **Assignments → Users or workload identities:**
   - **Include**: `dev-team` group
4. **Target resources:**
   - **Include**: Cloud apps → `Microsoft Azure Management`
5. **Grant:**
   - Check **Require multi-factor authentication**
6. **Enable policy**: `On`
7. Click **Create**

**What this does in practice**: If a dev team member tries to access the Azure portal or CLI without completing MFA, they'll be blocked. This is now standard operating procedure at most companies.

---

## Part 7: Cleanup

1. **Entra ID** → **Users** → Delete `Alex Dev`
2. **Entra ID** → **Groups** → Delete `dev-team`
3. **Resource groups** → Delete `team-dev` and `team-prod`
   - Deleting the RGs also deletes the storage account, managed identity, and all role assignments within them

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|--------------------------|
| **Creating a user and group** | Standard onboarding; groups make permission management scalable |
| **Scoped role assignments** | Prevents developers from accidentally breaking prod |
| **Managed identity** | Eliminates credential sprawl; no secrets in config files or git |
| **Verifying permissions** | Catches misconfiguration before access is handed off |
| **Removing group membership** | Clean, fast offboarding without hunting down role assignments |
| **Conditional Access** | Enforces MFA and security baselines at the identity layer |

---

## Common Mistakes to Avoid
- **Assigning roles to individuals instead of groups**: When someone leaves, you have to find and remove every role assignment manually
- **Using Owner when Contributor is enough**: Owner allows changing IAM permissions — that's a powerful and often unnecessary privilege
- **Using storage account keys instead of managed identities**: Keys don't expire automatically, get committed to code, and are hard to audit
- **Assigning roles at subscription scope when RG scope is sufficient**: Subscription-level roles apply everywhere, including resources you haven't created yet

---

## Next Steps
- Attach the managed identity to a VM or App Service and test it with the Azure SDK
- Set up a Privileged Identity Management (PIM) policy so prod access requires approval
- Review the audit log in Entra ID to see a history of sign-ins and permission changes
- Create an Azure Policy that prevents anyone from assigning the Owner role without a justification
