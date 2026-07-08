# Lab 1: Identity & Access Protection

Check box if done: [ ]

## Overview
Identity is the primary security boundary in Azure — there's no network perimeter that matters more than who can authenticate and what they can do once they're in. This lab builds the identity control stack a security engineer is actually responsible for: emergency access accounts that survive a lockout, just-in-time privileged roles instead of standing admin access, Conditional Access enforcing MFA and blocking legacy protocols, and risk-based policies that respond to Identity Protection signals automatically.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (all Entra ID features; PIM and Identity Protection require an Entra ID P2 trial license)

---

## Scenario
You're the security engineer for a small company's Azure tenant. An audit just flagged three problems: there's no emergency access account (if Conditional Access misfires, everyone including admins could be locked out), the two Global Administrators have standing admin access all day every day, and there's no MFA enforcement or risk-based response anywhere in the tenant. You're fixing all three today.

---

## Objectives
- Create and correctly exclude break-glass emergency access accounts
- Convert a standing admin role assignment to PIM-eligible, just-in-time activation
- Build a Conditional Access policy requiring MFA and blocking legacy authentication
- Configure Identity Protection risk-based sign-in and user risk policies
- Verify the whole stack doesn't lock you out before you walk away

---

## Part 1: Break-Glass Emergency Access Accounts

### Step 1: Why This Comes First
Every subsequent step in this lab tightens access. If a Conditional Access policy is misconfigured, or PIM approval breaks, or your admin account gets risk-flagged by Identity Protection, you need a way in that bypasses all of it. Set this up *before* you touch anything else.

### Step 2: Create Two Break-Glass Accounts
Use two, not one — if one account is compromised or its credential is lost, you still have a path in.

1. **Entra ID** → **Users** → **+ New user** → **Create new user**
2. **User 1**:
   - **User principal name**: `breakglass1@yourdomain.onmicrosoft.com`
   - **Display name**: `BREAKGLASS-1 Emergency Access — DO NOT DELETE`
   - **Password**: Let me create the password → generate a long, random one (24+ characters) and store it in a physical safe or an offline password manager — not in this repo, not in a note, not in a chat message
3. Repeat for `breakglass2@yourdomain.onmicrosoft.com`

### Step 3: Assign Global Administrator Directly (Not via a Group, Not via PIM)
Break-glass accounts are the one legitimate exception to "assign roles to groups" and "use PIM for admin roles." They need to work *when everything else is broken*.

1. **Entra ID** → **Roles and administrators** → **Global Administrator** → **+ Add assignments**
2. Select both break-glass accounts → **Add**

### Step 4: Set Them to Never Expire and Enable Cloud-Only Auth
1. Confirm both accounts are **cloud-only** (not synced from on-prem AD) — if your on-prem identity provider goes down, a synced account goes down with it
2. Do **not** enable MFA registration for these accounts — you'll explicitly exclude them from MFA-requiring Conditional Access policies in Part 3 instead. (An account that requires MFA to break the glass in an MFA outage defeats the purpose.)

**Real-world practice**: Set up an alert (Log Analytics / Sentinel, covered in Lab 4) that fires on any sign-in from a break-glass account — they should almost never be used, so any sign-in is worth investigating immediately.

---

## Part 2: Privileged Identity Management (PIM) — Just-in-Time Access

### Step 5: Why Standing Admin Access Is a Risk
A Global Administrator account that's always "on" is a permanent high-value target — if it's ever phished, the attacker has full tenant control indefinitely. PIM makes admin roles **eligible** rather than **active**: a user has to explicitly activate the role, provide a justification, and (optionally) get approval or pass an MFA challenge, and the elevation expires automatically.

### Step 6: Convert Your Own Admin Role to Eligible
1. **Entra ID** → **Roles and administrators** → search **Privileged Identity Management** (or go directly to **PIM** in the top search bar)
2. **Microsoft Entra roles** → **Roles** → find your own admin role (e.g., **User Administrator** — use a lower-privilege role than Global Admin for this exercise so you don't accidentally lock yourself out of PIM itself)
3. Click the role → **Add assignments** → select yourself
4. **Assignment type**: `Eligible`
5. **Duration**: Permanently eligible (in production, set a review-driven expiration instead)
6. Click **Assign**

### Step 7: Remove the Standing (Active) Assignment
If you had a permanent/active assignment for that same role before, remove it now — otherwise you still have standing access and PIM is providing zero benefit.

1. Same role → **Assigned** tab → find the active assignment → **Remove**

### Step 8: Configure Role Settings — Require Justification and MFA on Activation
1. **PIM** → **Microsoft Entra roles** → **Roles** → select the role → **Settings** → **Edit**
2. **Activation** tab:
   - **Require Azure MFA on activation**: On
   - **Require justification on activation**: On
   - **Maximum activation duration**: `4 hours`
3. **Assignment** tab:
   - **Require Azure MFA on active assignment**: On
4. **Update**

### Step 9: Activate the Role and Verify
1. Go to **PIM** → **My roles** → **Microsoft Entra roles**
2. Find your eligible role → **Activate**
3. Provide a justification (e.g., "Lab 1 — configuring Conditional Access policies")
4. Complete the MFA prompt if configured
5. Confirm the role is now listed under **Active** with a countdown to expiration

**Validation checkpoint**: Check **PIM** → **Microsoft Entra roles** → **Audit history**. You should see the activation event with your justification text logged.

---

## Part 3: Conditional Access — Enforce MFA and Block Legacy Auth

### Step 10: Build the MFA-for-Everyone Policy
1. **Entra ID** → **Security** → **Conditional Access** → **+ New policy**
2. **Name**: `Require MFA — All Users`
3. **Assignments → Users**:
   - **Include**: `All users`
   - **Exclude**: both break-glass accounts (this is the step people skip and then lock themselves out)
4. **Target resources**:
   - **Include**: `All cloud apps`
5. **Grant**:
   - **Grant access** → check **Require multi-factor authentication**
6. **Enable policy**: Start in **Report-only** mode first
7. **Create**

### Step 11: Test in Report-Only Mode Before Enforcing
1. Sign in as a test (non-break-glass) user
2. **Conditional Access** → **Insights and reporting** → confirm the policy shows as "Success" (would have required MFA) for that sign-in, without actually blocking anything yet
3. Once you've confirmed it behaves as expected, edit the policy → **Enable policy: On**

### Step 12: Block Legacy Authentication
Legacy protocols (IMAP, POP3, older Office clients using basic auth) can't perform MFA — they're a common bypass path for password-spray attacks.

1. **+ New policy** → **Name**: `Block Legacy Authentication`
2. **Assignments → Users**: Include `All users`, exclude break-glass accounts
3. **Target resources**: `All cloud apps`
4. **Conditions** → **Client apps** → check only **Exchange ActiveSync clients** and **Other clients** (these represent legacy/basic auth)
5. **Grant** → **Block access**
6. **Enable policy**: `On` directly — there's no legitimate reason to allow legacy auth in a modern tenant, so report-only isn't necessary here (skip only if you know something in your tenant still depends on basic auth)
7. **Create**

**Validation checkpoint**: **Conditional Access** → **Insights and reporting** → filter by the legacy auth policy over the next 24 hours. Any blocked sign-in attempts here are either misconfigured legacy clients or actual attack traffic — both worth investigating.

---

## Part 4: Identity Protection — Risk-Based Policies

### Step 13: Understand the Two Risk Types
- **Sign-in risk**: risk associated with a specific authentication attempt (impossible travel, anonymous IP, unfamiliar sign-in properties)
- **User risk**: risk associated with the likelihood an account is compromised (leaked credentials found in a breach dump, confirmed compromise from Microsoft threat intelligence)

They're different signals and get different policies.

### Step 14: Configure a Sign-in Risk Policy
1. **Entra ID** → **Security** → **Identity Protection** → **Sign-in risk policy**
2. **Assignments → Users**: `All users`, exclude break-glass accounts
3. **Conditions → Sign-in risk**: `Medium and above`
4. **Access** → **Grant access** → **Require multi-factor authentication**
5. **Enforce Policy**: `On`
6. **Save**

### Step 15: Configure a User Risk Policy
1. **Identity Protection** → **User risk policy**
2. **Assignments → Users**: `All users`, exclude break-glass accounts
3. **Conditions → User risk**: `High`
4. **Access** → **Grant access** → **Require password change**
5. **Enforce Policy**: `On`
6. **Save**

**Why different remediations?** A risky *sign-in* (this specific attempt looks suspicious) is resolved by proving it's really you (MFA). A risky *user* (the account's credentials are likely compromised) needs the credential itself to change — MFA doesn't fix a leaked password.

### Step 16: Review the Risk Reports
1. **Identity Protection** → **Risky users** and **Risky sign-ins**
2. These populate over time as Microsoft's detection engine evaluates real sign-in traffic — in a brand-new trial tenant they may be empty. Note what each report shows and how you'd triage an entry (dismiss as false positive, confirm compromise, force password reset)

---

## Part 5: Verify You Didn't Lock Yourself Out

### Step 17: Confirm Break-Glass Access Still Works
1. Open a private/incognito browser window
2. Sign in as `breakglass1@yourdomain.onmicrosoft.com`
3. Confirm: no MFA prompt, no Conditional Access block, full Global Administrator access
4. Sign out immediately — this account should not be used for routine work

**This is the single most important validation step in the lab.** A Conditional Access misconfiguration that excludes the wrong group, or a break-glass account accidentally caught by an "All users" policy, is one of the most common real-world Entra ID incidents — and it's entirely self-inflicted.

---

## Part 6: Cleanup

1. **Identity Protection** → disable the sign-in risk and user risk policies (or leave enabled if you're continuing to use this tenant — they cost nothing)
2. **Conditional Access** → disable or delete both policies if this is a throwaway lab tenant
3. **PIM** → deactivate your active role assignment (or let it expire)
4. **Entra ID** → **Users** → decide whether to keep the break-glass accounts (recommended to keep in any tenant you continue using) or delete them if this was a fully disposable lab tenant
5. If you activated an Entra ID P2 trial solely for this lab, no action needed — it expires automatically after 30 days

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Break-glass accounts, correctly excluded** | Prevents a Conditional Access misconfiguration from locking out every admin simultaneously |
| **PIM eligible roles** | Eliminates standing privileged access; every elevation is logged, justified, and time-boxed |
| **MFA via Conditional Access** | Baseline control against credential-only compromise |
| **Blocking legacy authentication** | Closes off a common MFA-bypass path for password spray attacks |
| **Risk-based Conditional Access** | Automates response to Identity Protection signals instead of relying on manual review |
| **Testing in report-only mode first** | Catches policy misconfigurations before they lock out real users |

---

## Common Mistakes to Avoid
- **Forgetting to exclude break-glass accounts from a policy**: The single most common cause of full-tenant lockouts
- **Skipping report-only mode**: Enforcing a new Conditional Access policy directly risks blocking legitimate access before you've verified the conditions are correct
- **Assigning admin roles permanently instead of PIM-eligible**: Standing access is a bigger blast radius if any single credential is compromised
- **Using the same remediation for sign-in risk and user risk**: MFA doesn't fix a leaked password; a password reset is overkill for "this sign-in looked unusual"
- **MFA-enrolling break-glass accounts**: If MFA infrastructure itself fails, an MFA-required break-glass account is useless exactly when you need it

---

## Next Steps
- Add an access review that periodically re-certifies who's eligible for each PIM role (covered in [SC-300 Lab 3](../SC-300/lab-3-access-governance.md))
- Build a Sentinel analytics rule that alerts on any break-glass account sign-in (covered in [AZ-500 Lab 4](lab-4-security-ops-automation.md))
- Extend Conditional Access with device compliance requirements (require Intune-compliant or hybrid-joined devices for admin roles)
- Review Identity Protection's risk detection details for the specific signals available at your license tier
