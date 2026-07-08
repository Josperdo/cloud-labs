# Lab 2: Application Integration & SSO

Check box if done: [ ]

## Overview
"Add this app to SSO" is one of the most common identity administration tickets — and one of the most misunderstood, because Entra ID splits every application into two separate objects (an app registration and an enterprise app) that people conflate constantly. This lab builds a real app registration, walks through what each of those two objects actually controls, configures app roles for authorization, and works through the consent model that decides whether a user can sign in at all.

**Estimated time**: 60–75 minutes
**Cost**: ~$0

---

## Scenario
A team built an internal line-of-business web app and needs it integrated with Entra ID: users sign in with their existing company account (no separate password), and the app needs to know whether a given user is a "Viewer" or an "Approver" so it can show the right UI. You're registering the app, configuring authentication, defining app roles, and deciding how consent should work — the way an identity admin actually onboards an app.

---

## Objectives
- Understand the difference between an app registration and an enterprise application (they're two views of the same underlying identity)
- Register an application and configure a redirect URI
- Define app roles and understand how they map to authorization inside the app
- Assign users/groups to app roles
- Understand admin consent vs. user consent and when each is appropriate

---

## Part 1: App Registration vs. Enterprise Application

### Step 1: The Concept Before the Clicks
Every application integrated with Entra ID has **two objects**:
- **App registration** (`Applications` → `App registrations`): the **definition** — what the app is, its authentication settings, its declared permissions (API scopes it needs, app roles it defines). This exists once, in the tenant where the app was registered (the "home" tenant).
- **Enterprise application** (`Applications` → `Enterprise applications`): the **service principal** — the local instantiation of that app in *your* tenant, which is what actually gets user/group assignments, Conditional Access policies, and provisioning configuration applied to it.

If you register an app in your own tenant, you'll see both objects appear simultaneously (registering creates the service principal automatically). If someone else's multi-tenant app is added to your tenant (like installing "Zoom" or "Slack" from the gallery), you only see the enterprise application — the app registration lives in the vendor's tenant, not yours.

### Step 2: Register the Application
1. **Entra ID** → **App registrations** → **+ New registration**
2. **Name**: `Internal LOB App`
3. **Supported account types**: `Accounts in this organizational directory only (Single tenant)`
4. **Redirect URI**: Platform: `Web`, URI: `https://localhost:5000/signin-oidc` (placeholder for local dev testing)
5. **Register**

### Step 3: Confirm Both Objects Exist
1. Note the **Application (client) ID** from the registration's **Overview** page
2. **Entra ID** → **Enterprise applications** → search `Internal LOB App` — confirm it appears with the same client ID

**This is the object you'll use for role assignment, Conditional Access, and provisioning in the rest of this lab** — not the app registration.

---

## Part 2: Authentication Configuration

### Step 4: Configure the Auth Flow
1. Open the app registration → **Authentication**
2. Under **Implicit grant and hybrid flows**, leave both **Access tokens** and **ID tokens** unchecked unless you have a specific legacy requirement — modern apps use the **Authorization Code flow with PKCE**, which needs neither
3. Under **Advanced settings**, confirm **Allow public client flows** is `No` (this is a confidential web app with a client secret, not a public/native client)

### Step 5: Create a Client Secret
1. **Certificates & secrets** → **+ New client secret**
2. **Description**: `local-dev`, **Expires**: `6 months`
3. **Add** — copy the secret value immediately (it's shown once)

**Real-world practice**: client secrets are exactly the kind of credential that shouldn't live in a config file — the app should retrieve this from Key Vault via managed identity (see [AZ-500 Lab 3](../AZ-500/lab-3-data-app-security.md)) rather than storing it directly, and it should be rotated well before its 6-month expiry.

---

## Part 3: App Roles — Authorization Inside the App

### Step 6: Why App Roles
Authentication answers "who is this user." App roles answer "what should this user be allowed to do inside the app" — and unlike a group membership check the app would have to query separately, app roles are baked directly into the token the app receives at sign-in.

### Step 7: Define Two App Roles
1. App registration → **App roles** → **+ Create app role**
2. **Role 1**:
   - **Display name**: `Approver`
   - **Allowed member types**: `Users/Groups`
   - **Value**: `Approver` (this exact string appears in the token's `roles` claim)
   - **Description**: `Can approve submitted requests`
3. **Role 2**:
   - **Display name**: `Viewer`
   - **Value**: `Viewer`
   - **Description**: `Read-only access`
4. Save both

### Step 8: Assign Users to App Roles
App role assignment happens on the **enterprise application**, not the app registration.

1. **Enterprise applications** → **Internal LOB App** → **Users and groups** → **+ Add user/group**
2. Select a test user → **Select a role** → `Viewer` → **Assign**
3. Repeat, assigning a second test user (or yourself) to `Approver`

**Validation checkpoint**: **Users and groups** should now list two assignments with different roles. When each user signs in, the app receives a token containing a `roles` claim (`["Viewer"]` or `["Approver"]`) — the application code checks this claim to decide what to render or allow, without a separate database lookup.

### Step 9: Require Assignment (Close the Open Door)
By default, many app registrations allow **any** user in the tenant to sign in, assigned or not — "assignment required" defaults to off.

1. **Enterprise applications** → **Internal LOB App** → **Properties**
2. **Assignment required?**: `Yes` → **Save**

**Validation checkpoint**: A user who has *not* been explicitly assigned a role should now be blocked at sign-in with an "unauthorized" error, instead of getting in with no role at all (which typically breaks the app in confusing ways rather than cleanly denying access).

---

## Part 4: Consent — Who Approves What the App Can Access

### Step 10: Understand the Two Consent Types
- **User consent**: an individual user approves the app's requested permissions for themselves only, the first time they sign in
- **Admin consent**: an administrator approves the app's requested permissions for the entire organization (or a specific group) at once — required for any "application" (non-delegated) permission, and for high-privilege delegated permissions your tenant's consent policy restricts

### Step 11: Add an API Permission and Grant Admin Consent
1. App registration → **API permissions** → **+ Add a permission** → **Microsoft Graph** → **Delegated permissions** → search and add `User.Read` (likely already present by default)
2. Add one more: **Application permissions** → `User.Read.All` (a higher-privilege, admin-only permission — represents something like the app needing to read the full user directory for a sync job)
3. Note the warning icon and "Not granted" status next to the application permission
4. **Grant admin consent for [tenant]** → confirm

**Validation checkpoint**: After granting, the status changes to **Granted**. Without this step, the app's own calls using `User.Read.All` would fail with an insufficient-privileges error even though the permission is *requested* — requesting a permission and being *granted* it are different things.

### Step 12: Why This Matters for Security Review
Any application permission requiring admin consent is worth scrutinizing before approving — `User.Read.All` is far broader than most line-of-business apps genuinely need. Part of an identity admin's job is pushing back on over-broad permission requests the same way you'd push back on an over-broad RBAC role assignment.

---

## Part 5: Cleanup

1. **Entra ID** → **Enterprise applications** → **Internal LOB App** → **Properties** → **Delete**
2. **App registrations** → **Internal LOB App** → **Delete** (deleting the enterprise app doesn't automatically remove the registration in single-tenant apps you own)
3. Confirm both are gone: search `Internal LOB App` in both blades — expect no results

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **App registration vs. enterprise application** | The #1 source of confusion in app onboarding tickets — knowing which object controls what saves real troubleshooting time |
| **App roles** | Puts authorization data directly in the token, avoiding a separate app-side permission lookup |
| **"Assignment required" set to Yes** | Prevents any tenant user from signing into an app that should be restricted to specific people |
| **User consent vs. admin consent** | Understanding which permission requests need administrator review before they're granted |
| **Scrutinizing application permissions before granting admin consent** | Same least-privilege discipline as RBAC role assignment, applied to app permissions |

---

## Common Mistakes to Avoid
- **Managing user/group assignments from the app registration instead of the enterprise application**: assignments live on the enterprise app (service principal) — the app registration has no such option
- **Leaving "Assignment required?" at its default (often `No`)**: silently allows any tenant user to sign in, regardless of whether they were ever explicitly granted access
- **Granting admin consent to application permissions without reading what they actually allow**: `User.Read.All` and similar broad application permissions deserve the same scrutiny as an over-broad Azure RBAC role
- **Confusing "permission requested" with "permission granted"**: a listed API permission with "Not granted" status does nothing for the app until consent is completed

---

## Next Steps
- Configure SAML-based SSO instead of OIDC for an app that requires it (common with older enterprise SaaS products)
- Set up automatic user provisioning (SCIM) for an enterprise app that supports it, instead of manual role assignment
- Continue to [Lab 3: Access Governance](lab-3-access-governance.md) to periodically re-certify who still needs Viewer/Approver access to this app
- Review your tenant's user consent settings (**Enterprise applications** → **Consent and permissions**) and consider restricting user consent to verified publishers only
