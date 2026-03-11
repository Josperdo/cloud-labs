# Lab 4: Identity & Access Control (IAM/Entra ID)

## Overview
This lab covers Azure role-based access control (RBAC), Entra ID (formerly Azure AD), managed identities, and conditional access policies. You'll build a realistic IAM scenario with custom roles and scoped access.

**Estimated time**: 60–75 minutes  
**Cost**: ~$0 (IAM/Entra features are free or included in licensing)

---

## Objectives
- Understand RBAC: roles, resource scope, and principle of least privilege
- Create custom RBAC roles
- Assign roles at different scopes (subscription, resource group, resource)
- Configure Entra ID users and groups
- Implement conditional access policies
- Understand managed identities for service-to-service auth

---

## Part 1: Understand Built-In Roles

### Step 1: Inspect Azure Built-In Roles
1. Go to **Azure Portal** → search **Roles and administrators** (or **Access Control (IAM)**)
2. Click **Roles** (left sidebar)
3. Filter by **Type**: `Built-in`
4. Search for these and understand their scope:
   - **Owner**: Full access to everything (subscription level)
   - **Contributor**: Can create/modify/delete but can't change permissions
   - **Reader**: Read-only access
   - **Virtual Machine Contributor**: Can manage VMs but not access them or the VNet
   - **Storage Account Contributor**: Can manage storage accounts but not data within them

**Key insight**: Each built-in role has a set of allowed actions (e.g., `Microsoft.Compute/virtualMachines/write`).

---

## Part 2: Create Resource Groups and Users

### Step 2: Create Resource Groups for IAM Testing
1. Search **Resource groups** → **+ Create**
2. **Create RG #1:**
   - **Name**: `az104-lab4-prod`
   - **Region**: `East US`
   - Click **Create**

3. **Create RG #2:**
   - **Name**: `az104-lab4-dev`
   - **Region**: `East US`
   - Click **Create**

**Why two RGs?** You'll demonstrate scoped access (different permissions in prod vs dev).

### Step 3: Create Test Users in Entra ID
1. Go to **Azure Portal** → Search **Entra ID** (or **Azure Active Directory**)
2. Left sidebar → **Users** → **+ New user** → **Create new user**
3. **Create User #1 (Prod Manager):**
   - **User principal name (UPN)**: `prodmanager@yourdomain.onmicrosoft.com`
     - *Note: Use your actual Entra ID domain; it's usually `yourcompany.onmicrosoft.com` or a custom domain*
   - **Mail nickname**: `prodmanager`
   - **Display name**: `Prod Manager`
   - **Password**: Check **Let me create the password** → Create a strong one (e.g., `ProdMgr!2024`)
   - **Usage Location**: `United States`
4. Click **Create**

4. **Create User #2 (Dev Developer):**
   - **UPN**: `devdev@yourdomain.onmicrosoft.com`
   - **Mail nickname**: `devdev`
   - **Display name**: `Dev Developer`
   - **Password**: Create a strong one
   - **Usage Location**: `United States`
5. Click **Create**

6. **Create User #3 (Auditor):**
   - **UPN**: `auditor@yourdomain.onmicrosoft.com`
   - **Mail nickname**: `auditor`
   - **Display name**: `Auditor`
   - **Password**: Create a strong one
   - **Usage Location**: `United States`
7. Click **Create**

**Note**: You're creating test accounts. In production, these would come from Azure AD Sync or Microsoft 365.

---

## Part 3: Create Groups

### Step 4: Create Security Groups
1. Still in **Entra ID** → Left sidebar → **Groups** → **+ New group**
2. **Create Group #1:**
   - **Group type**: `Security`
   - **Group name**: `Prod-Admins`
   - **Group description**: `Admins for production resources`
   - **Membership type**: `Assigned` (you manually add members)
3. Click **Create**

4. **Create Group #2:**
   - **Group type**: `Security`
   - **Group name**: `Dev-Team`
   - **Group description**: `Developers with dev environment access`
   - **Membership type**: `Assigned`
5. Click **Create**

### Step 5: Add Users to Groups
1. Open **Prod-Admins** group → Left sidebar → **Members** → **+ Add members**
2. Search for and select:
   - `Prod Manager`
   - `Auditor`
3. Click **Select**

4. Open **Dev-Team** group → Left sidebar → **Members** → **+ Add members**
5. Select:
   - `Dev Developer`
   - `Auditor` (auditor is in both groups)
6. Click **Select**

**Why groups?** Instead of assigning roles to individual users, you assign to groups. When users join/leave, you just modify group membership.

---

## Part 4: Assign RBAC Roles at Different Scopes

### Step 6: Assign Roles at Subscription Level
1. Go to **Subscriptions** (search in Portal)
2. Click your subscription name
3. Left sidebar → **Access Control (IAM)** → **+ Add** → **Add role assignment**
4. **Basics tab:**
   - **Role**: Search `Contributor` → select it
   - **Assign access to**: `User, group, or service principal`
   - **Members**: Click **+ Select members** → Search `Prod-Admins` → select it
5. Click **Review + assign** → **Assign**

**What this means**: All users in Prod-Admins group can create, modify, and delete resources anywhere in your subscription (but not change IAM permissions).

### Step 7: Assign Roles at Resource Group Level (Scoped Access)
1. Go to **Resource groups** → Click `az104-lab4-prod`
2. Left sidebar → **Access Control (IAM)** → **+ Add** → **Add role assignment**
3. **Basics tab:**
   - **Role**: `Virtual Machine Contributor`
   - **Assign access to**: `User, group, or service principal`
   - **Members**: Click **+ Select members** → Search `Prod Manager` → select it
4. Click **Review + assign** → **Assign**

**What this means**: Prod Manager can manage VMs in the `az104-lab4-prod` resource group only (not subscription-wide).

### Step 8: Assign Reader Role to Auditor (RG Level)
1. Still in `az104-lab4-prod` RG → **Access Control (IAM)** → **+ Add** → **Add role assignment**
2. **Basics tab:**
   - **Role**: `Reader`
   - **Members**: `Auditor`
3. Click **Review + assign** → **Assign**

4. Repeat for `az104-lab4-dev` RG:
   - Role: `Reader`
   - Members: `Auditor`

**What this means**: Auditor can read (view) resources in both RGs but can't modify them.

### Step 9: Assign Contributor to Dev Team
1. Go to `az104-lab4-dev` RG → **Access Control (IAM)** → **+ Add** → **Add role assignment**
2. **Basics tab:**
   - **Role**: `Contributor`
   - **Members**: `Dev-Team`
3. Click **Review + assign** → **Assign**

**Summary of what you've set up**:
- **Prod-Admins**: Contributor at subscription level (can modify anything)
- **Prod Manager**: VM Contributor in prod RG only (can manage VMs, nothing else)
- **Dev-Team**: Contributor in dev RG only (can do anything in dev)
- **Auditor**: Reader in both prod and dev RGs (read-only everywhere)

---

## Part 5: Create and Assign a Custom RBAC Role

### Step 10: Create Custom Role
1. Search **Roles and administrators** → **Roles** → **+ Create custom role**
2. **Basics tab:**
   - **Custom role name**: `Storage Blob Reader`
   - **Description**: `Can read blob data in storage accounts`
   - **Baseline permissions**: Start from scratch → **Custom created role**
3. Click **Next: Permissions**

4. **Permissions tab:**
   - Click **+ Add permissions**
   - Search: `Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read`
   - Check the checkbox for this permission
   - Click **Add**
   - Search: `Microsoft.Storage/storageAccounts/listKeys/action`
   - Check and add it too
   - Click **Next: Assignable scopes**

5. **Assignable scopes tab:**
   - Leave default (/ = subscription level)
   - Click **Review + create**

6. Click **Create**

**What you've created**: A role that can only read blobs in storage accounts (no delete, no write permissions).

### Step 11: Assign Custom Role
1. Go to `az104-lab4-prod` RG → **Access Control (IAM)** → **+ Add** → **Add role assignment**
2. **Basics tab:**
   - **Role**: Search `Storage Blob Reader` → select it
   - **Members**: `Dev Developer`
3. Click **Review + assign** → **Assign**

**What this means**: Dev Developer can read blobs in storage accounts within the prod RG, but can't delete or modify them.

---

## Part 6: Understanding Managed Identities

### Step 12: Create a User-Assigned Managed Identity
1. Search **Managed Identities** → **+ Create**
2. **Basics:**
   - **Resource Group**: `az104-lab4-prod`
   - **Region**: `East US`
   - **Name**: `lab4-app-identity`
3. Click **Create**

4. After creation, open the managed identity → **Overview**
   - Note the **Object ID** (this is the identity's unique identifier)

**Why managed identities?** Applications running in Azure can use this identity to access Azure resources (like storage or databases) without storing secrets in code.

### Step 13: Assign Role to Managed Identity
1. Still in the managed identity → Left sidebar → **Azure role assignments** → **+ Add role assignment**
2. **Scope**: `Resource group`
3. **Resource group**: `az104-lab4-prod`
4. **Role**: `Storage Blob Contributor`
5. Click **Save**

**What this does**: Any app using this managed identity can read/write blobs in storage accounts within the prod RG.

---

## Part 7: Conditional Access (Premium Feature)

### Step 14: Create a Conditional Access Policy
**Note**: Conditional Access requires Entra ID Premium P1 or P2. If you don't have it, you can view but not create policies. This is informational.

1. Go to **Entra ID** → Left sidebar → **Security** → **Conditional Access**
2. Click **+ New policy**
3. **Name**: `Require MFA for Dev Group`
4. **Assignments → Users or workload identities:**
   - **Include**: Select **Specific users and groups** → Search `Dev-Team` → select it
5. **Target resources:**
   - **Include**: `Cloud apps` → Check **Select apps** → Search `Microsoft Azure Management` → select it
6. **Conditions:**
   - **Sign-in risk**: Leave unchecked
   - **Device platforms**: Leave unchecked
7. **Grant:**
   - Check **Require multi-factor authentication**
   - Check **Require compliant device**
8. **Session:** Leave defaults
9. **Enable policy**: `On`
10. Click **Create**

**What this does**: When users in Dev-Team access Azure resources, they're required to use MFA (multi-factor auth).

**In exam context**: Conditional Access is about zero-trust security. You'll see questions about risk-based access, device compliance, and MFA policies.

---

## Part 8: Review Permissions and Validate

### Step 15: Check Effective Permissions
1. Go to `az104-lab4-prod` RG → **Access Control (IAM)** → **Check access**
2. Search for `Prod Manager`
3. View what roles this user has and what actions they can perform

4. Repeat for `Auditor`:
   - Should show **Reader** permission in prod and dev RGs

5. Repeat for `Dev Developer`:
   - Should show **Contributor** in dev RG and **Storage Blob Reader** in prod RG

**Key insight**: The effective permissions reflect all roles assigned at this scope and inherited from parent scopes.

### Step 16: Understand Scope Hierarchy
```
Subscription
  ├─ Resource Group (prod)
  │   ├─ Virtual Machine
  │   └─ Storage Account
  └─ Resource Group (dev)
      └─ Virtual Machine
```

- Roles assigned at subscription level apply to all RGs and resources
- Roles assigned at RG level apply only to that RG and its resources
- Roles assigned at resource level apply only to that resource

---

## Part 9: Cleanup

1. Go to **Entra ID** → **Users** → Delete `Prod Manager`, `Dev Developer`, `Auditor`
   - Select user → **Delete user** → **Delete**

2. Go to **Entra ID** → **Groups** → Delete `Prod-Admins`, `Dev-Team`
   - Select group → **Delete group** → **Delete**

3. Go to **Resource groups** → Delete `az104-lab4-prod`, `az104-lab4-dev`
   - Select RG → **Delete resource group** → Confirm

**Don't delete**:
- The managed identity (it's harmless) or delete it if you want a clean slate
- The custom role (it's stored at subscription level; you might reference it later)

---

## Key Concepts to Understand

| Concept | What It Does |
|---------|-------------|
| **RBAC** | Azure's permission model: who can do what to which resources |
| **Role** | A collection of permissions (actions + data actions) |
| **Scope** | Where a role applies: subscription, RG, or individual resource |
| **Principle of Least Privilege** | Give users the minimum permissions they need |
| **Custom Role** | Role you create with specific permissions |
| **Managed Identity** | Identity for apps/services to authenticate without secrets |
| **Entra ID** | Azure's directory and identity service (formerly Azure AD) |
| **Conditional Access** | Policies that enforce MFA, device compliance, etc. based on risk |

---

## Exam Tips
- **Scope hierarchy**: Child scopes inherit from parent. A subscription-level Contributor can do everything; RG-level Contributor can only modify that RG.
- **Deny assignments**: Override allows (if you Deny a role, even Owner can't do it)
- **Service principals**: Apps use these to authenticate. Managed identities are service principals managed by Azure.
- **Role assignments are immediate**: Changes take effect within a few seconds
- **Entra ID Premium**: Conditional Access requires Premium P1+; free tier doesn't have it
- **Custom roles**: Only assignable at scopes you defined during creation

---

## Next Steps
- Integrate Entra ID with on-premises AD (hybrid identity)
- Set up multi-factor authentication (MFA) for all users
- Create a custom role for "Database Administrator" (specific SQL Server actions)
- Implement role-based access control for a multi-tenant SaaS application
- Use managed identities in a VM to securely access Key Vault
