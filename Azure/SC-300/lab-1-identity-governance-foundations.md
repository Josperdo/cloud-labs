# Lab 1: Identity Governance Foundations

Check box if done: [ ]

## Overview
The most common identity administration mistake is reaching for Global Administrator any time someone needs to manage users or reset passwords. This lab builds the alternative: least-privilege built-in roles, a custom role scoped to exactly what a support-desk function needs, and administrative units that let you delegate management of one department without granting tenant-wide reach.

**Estimated time**: 45–60 minutes
**Cost**: ~$0

---

## Scenario
Your company is growing. The help desk team needs to reset passwords and manage licenses for the Sales department only — not Engineering, not Finance, and definitely not the ability to touch admin roles. Right now, the one person who does this work has Global Administrator, "because it was easier to just give them everything." You're fixing that today: least-privilege roles, scoped to exactly the population they should manage.

---

## Objectives
- Understand the built-in Entra ID role hierarchy and choose the least-privileged role for a task
- Create a custom role scoped to a narrow, specific set of permissions
- Create an administrative unit and delegate a role scoped to it only
- Verify the delegated admin can manage users inside the unit but not outside it

---

## Part 1: The Built-In Role Model

### Step 1: Survey the Built-In Roles
1. **Entra ID** → **Roles and administrators**
2. Search for these three roles and read their descriptions:
   - **Global Administrator** — full tenant control, including the ability to grant itself any other role
   - **User Administrator** — can create/manage users and groups, reset non-admin passwords, but cannot touch role assignments or tenant-wide settings
   - **Helpdesk Administrator** — narrower still: can reset passwords for non-admin users only, cannot create or delete users

**The pattern to internalize**: every built-in role has a specific, documented set of permissions. "Which role can do X" is a lookup, not a guess — click into **Helpdesk Administrator** → **Description** and note the exact permission list before assigning it to anyone.

### Step 2: Assign Helpdesk Administrator to a Test User
1. **Entra ID** → **Users** → **+ New user** → create `sales.support@yourdomain.onmicrosoft.com`
2. **Roles and administrators** → **Helpdesk Administrator** → **+ Add assignments** → select the new user

**Validation checkpoint**: **Roles and administrators** → **Helpdesk Administrator** → **Assignments** — confirm `sales.support` is listed.

---

## Part 2: Custom Roles

### Step 3: Why a Custom Role Might Beat a Built-In One
Suppose the actual requirement is narrower than any built-in role: this person should be able to reset passwords *and* view (but not change) group memberships, but nothing else. No built-in role matches that exactly. Rather than over-granting with `User Administrator`, build a custom role with exactly those permissions.

> **Licensing note**: Custom Entra ID roles require Entra ID P1 or P2. If you don't have a premium license, read through this section — the concept (least-privilege via explicit permission lists) is what matters for the exam even if you can't execute the create step.

### Step 4: Create the Custom Role
1. **Entra ID** → **Roles and administrators** → **+ New custom role**
2. **Basics**: Name: `Sales Password Resetter`, Description: `Reset passwords and view group membership for the Sales department only`
3. **Permissions** → search and add:
   - `microsoft.directory/users/password/update`
   - `microsoft.directory/groups/members/read`
4. Leave every other permission unchecked — that's the entire point of a custom role
5. **Review + create**

### Step 5: Compare Against What You'd Have Over-Granted
1. Open **User Administrator**'s permission list and count how many permissions it grants beyond the two above (dozens — including creating/deleting users, managing all groups, managing device settings)
2. This comparison is the answer to "why not just use an existing role" in an interview or exam scenario: a custom role's blast radius if compromised is exactly its permission list, not "whatever User Administrator happens to include."

---

## Part 3: Administrative Units — Scoping Delegation

### Step 6: Why Roles Alone Aren't Enough
Even a narrowly-permissioned role like Helpdesk Administrator, assigned normally, applies **tenant-wide** — that person could reset the password of anyone in the company, not just Sales. Administrative units add a scope dimension: "this role, but only for users in this unit."

### Step 7: Create an Administrative Unit for Sales
1. **Entra ID** → **Roles and administrators** → **Administrative units** → **+ Add**
2. **Name**: `Sales Department`, **Description**: `Scoped administration for Sales team members`
3. **Create**

### Step 8: Add Members to the Unit
1. Create two more test users: `sales.rep1@yourdomain.onmicrosoft.com` and `eng.dev1@yourdomain.onmicrosoft.com`
2. Open **Sales Department** administrative unit → **Users** → **+ Add members** → add only `sales.rep1` (leave `eng.dev1` out — it represents a user outside the delegated scope)

### Step 9: Assign a Scoped Role
1. Inside the **Sales Department** administrative unit → **Roles and administrators** → **Helpdesk Administrator** → **+ Add assignments**
2. Assign `sales.support@yourdomain.onmicrosoft.com`

This is a **scoped assignment** — `sales.support` now has Helpdesk Administrator permissions, but only for users inside the Sales Department administrative unit.

---

## Part 4: Verify the Scope Actually Holds

### Step 10: Confirm In-Scope Access Works
1. Sign in as `sales.support` (or use **Entra ID** → **Users** → **sales.rep1** → **Reset password**, checking the audit log for who performed it, if you can't sign in interactively as a test user in your tenant)
2. Confirm `sales.support` can reset `sales.rep1`'s password

### Step 11: Confirm Out-of-Scope Access Is Denied
1. As `sales.support`, attempt to reset `eng.dev1`'s password (a user **not** in the Sales Department administrative unit)

**Expected result**: Access denied. `sales.support`'s Helpdesk Administrator role only applies within the administrative unit boundary — `eng.dev1` is outside it, so the role grants nothing there.

**This is the step people skip and then wonder why "scoped" delegation didn't actually scope anything** — the scope only holds if the target user is genuinely a member of the administrative unit the role was assigned against.

---

## Part 5: Cleanup

1. **Entra ID** → **Roles and administrators** → **Administrative units** → **Sales Department** → delete
2. **Roles and administrators** → **Helpdesk Administrator** → remove the `sales.support` assignment (tenant-wide, if any remains)
3. Delete the custom role `Sales Password Resetter` if no longer needed
4. **Entra ID** → **Users** → delete `sales.support`, `sales.rep1`, `eng.dev1`

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Choosing the least-privileged built-in role** | Reduces blast radius; most admin tasks don't need Global Administrator or even User Administrator |
| **Custom roles with an explicit permission list** | Grants exactly what's needed when no built-in role fits precisely |
| **Administrative units** | Delegates management of a subset of users/groups without tenant-wide reach |
| **Scoped role assignment (role + administrative unit together)** | The actual mechanism that turns "department IT support" into a real, bounded permission set |
| **Verifying both the allow and the deny case** | Proves the scope boundary is real, not assumed |

---

## Common Mistakes to Avoid
- **Defaulting to Global Administrator "to be safe"**: the opposite is true — it's the least safe option and the one with the largest blast radius
- **Assigning a role tenant-wide when only a scoped assignment was intended**: always assign inside the administrative unit's own **Roles and administrators** blade, not the tenant-wide one, when scoping is the goal
- **Forgetting a user needs to be an explicit member of the administrative unit**: a role scoped to an AU grants nothing for a user who isn't in that AU, even if they're an obvious "should be" member
- **Building a custom role that duplicates most of a built-in role's permissions**: at that point, just use the built-in role — custom roles earn their complexity when the requirement is genuinely narrower

---

## Next Steps
- Explore **dynamic** administrative units (membership rules based on department attribute, instead of manual add) for scaling this pattern past a handful of users
- Continue to [Lab 2: Application Integration & SSO](lab-2-app-integration-sso.md) to see how app-level permissions layer on top of this same least-privilege model
- Review the full built-in role list and identify which role you'd assign for five other common help-desk-style requests (license management, group membership changes, device management)
