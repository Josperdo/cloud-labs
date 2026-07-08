# Lab 3: Access Governance

Check box if done: [ ]

## Overview
Granting access is the easy half of identity administration. The hard half is proving, months later, that the access still makes sense — that the contractor who left in March had their access removed, that the "temporary" project access from last quarter didn't quietly become permanent. This lab builds the governance layer: access reviews that force periodic re-certification, entitlement management that packages related access into something a user can request and a manager can approve, and PIM extended to group membership instead of just admin roles.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (requires Entra ID P2 trial)

---

## Scenario
An internal audit asked a question your team couldn't answer cleanly: "who currently has access to the Finance reporting tool, and when was that access last reviewed?" Nobody knew — access had been granted ad hoc over a year, nobody had left when they should have, and there was no process to catch it. You're building the fix: a recurring access review, a self-service access package for onboarding, and time-bound group membership via PIM.

---

## Objectives
- Configure a recurring access review for a group's membership
- Complete a review as both the reviewer and understand what happens on non-response
- Build an access package in entitlement management with an approval workflow
- Extend PIM to make group membership itself just-in-time instead of standing
- Understand how these three features fit together as a governance lifecycle

---

## Part 1: Access Reviews

### Step 1: Set Up the Target Group
1. **Entra ID** → **Groups** → **+ New group**
2. **Group type**: `Security`, **Name**: `finance-reporting-access`, **Membership type**: `Assigned`
3. Add 2–3 test users as members (create throwaway users if needed: `finance.user1@yourdomain.onmicrosoft.com`, etc.)

### Step 2: Create a Recurring Access Review
1. **Entra ID** → **Identity Governance** → **Access reviews** → **+ New access review**
2. **Select what to review**: `Teams + Groups`
3. **Group**: `finance-reporting-access`
4. **Scope**: `Guest and member users` (or `Members only`, depending on your population)
5. **Review name**: `Finance Reporting Access — Quarterly Review`
6. **Frequency**: `Quarterly`, **Duration**: `14 days`
7. **Reviewers**: `Selected users` → assign yourself, or `Group owners` if the group has a designated owner
8. **Upon completion settings**:
   - **Auto apply results**: `Enable` — results take effect automatically rather than requiring a second manual step
   - **If reviewers don't respond**: `Remove access` — the safe default; access that nobody can justify shouldn't persist by default
9. **Create**

### Step 3: Understand What Just Happened
This review will now fire automatically every quarter, generating a task for the assigned reviewer(s), and — critically — has a defined, safe outcome if nobody responds. This is the mechanism that answers the audit question from the scenario: "when was this last reviewed" now has a real, queryable answer instead of "never."

### Step 4: Complete a Review Cycle (Simulated)
Since a real quarterly cycle won't fire within this lab session, walk through the reviewer experience directly:

1. **Identity Governance** → **Access reviews** → your review → **Overview**
2. If you're a reviewer, you'll see each member listed with **Approve** / **Deny** actions
3. Approve one member, deny another (simulating "this person still needs it" vs. "this person shouldn't have this anymore")
4. **Stop review** (to end the cycle manually for lab purposes) → confirm the denied member's group membership was removed

**Validation checkpoint**: **Groups** → `finance-reporting-access` → **Members** — confirm the denied user is no longer listed.

---

## Part 2: Entitlement Management — Access Packages

### Step 5: Why Access Packages
Access reviews clean up access that already exists. Entitlement management controls how access is granted in the first place — instead of a Slack message to an admin, a user requests a defined **access package** (which might bundle a group, an app role assignment, and a SharePoint site into one request), and an approval workflow decides whether they get it.

### Step 6: Create a Catalog
1. **Identity Governance** → **Entitlement management** → **Catalogs** → **+ New catalog**
2. **Name**: `Finance Resources`, **Description**: `Access packages for Finance department resources`
3. **Create**

### Step 7: Add a Resource to the Catalog
1. Open the `Finance Resources` catalog → **Resources** → **+ Add resources**
2. **Resource type**: `Groups and Teams` → select `finance-reporting-access`
3. **Add**

### Step 8: Create an Access Package
1. **Entitlement management** → **Access packages** → **+ New access package**
2. **Basics**: **Catalog**: `Finance Resources`, **Name**: `Finance Reporting Tool Access`
3. **Resource roles**: select `finance-reporting-access` → **Role**: `Member`
4. **Requests**:
   - **Users who can request access**: `For users in your directory`
   - **Require approval**: `Yes`
   - **Approver**: yourself (or a designated manager)
   - **Require requestor justification**: `Yes`
5. **Lifecycle**:
   - **Access package assignments expire**: `On date` or `Number of days` — set `90 days`
   - **Require access reviews**: `Yes`, tied to the review cadence from Part 1
6. **Review + Create**

### Step 9: Test the Request Flow
1. Copy the **My Access** portal link (`myaccess.microsoft.com`) — a requesting user would go here, not the admin portal
2. As a test user (or by reasoning through the flow if you can't sign in as a second identity), the request would show: package name, description, and a justification field
3. As the approver, **Entitlement management** → **Approvals** — approve or deny the pending request

**Validation checkpoint**: An approved request results in the user gaining `finance-reporting-access` group membership automatically — no admin manually adding them to a group.

---

## Part 3: PIM for Groups — Just-in-Time Membership

### Step 10: Extend Just-in-Time Access Beyond Admin Roles
[AZ-500 Lab 1](../AZ-500/lab-1-identity-access-protection.md) covered PIM for Entra ID admin *roles*. PIM also supports making **group membership** itself eligible rather than standing — useful for access that grants meaningful privilege (like this Finance group) without needing a full admin role.

### Step 11: Enable a Group for PIM
1. **Identity Governance** → **Privileged Identity Management** → **Groups** → **Discover groups**
2. Select `finance-reporting-access` → **Manage groups**
3. Back in **PIM | Groups**, select the group → **Settings** → **Member** role → **Edit**
4. **Activation** tab: **Require justification**: On, **Maximum activation duration**: `8 hours`
5. **Assignment** tab: eligible assignments should have a **maximum duration** so they don't become permanent by default
6. **Update**

### Step 12: Assign a User as Eligible (Not Active)
1. **PIM | Groups** → `finance-reporting-access` → **Members** → **Add assignments**
2. **Membership type**: `Member`, **Assignment type**: `Eligible`
3. Select a test user → **Assign**

**What this achieves**: the user can now activate membership in `finance-reporting-access` for up to 8 hours when they actually need it — for the other 16 hours of the day (and every day they don't need it at all), they carry zero standing access to the Finance reporting tool.

---

## Part 4: How These Three Fit Together

### Step 13: The Governance Lifecycle, End to End
1. **Request**: user requests the `Finance Reporting Tool Access` package via My Access (Part 2)
2. **Approve**: designated approver reviews the justification and approves
3. **Grant**: access is granted, time-bound to 90 days
4. **Activate (if PIM-eligible)**: if the underlying group is PIM-enabled, the user still activates membership just-in-time rather than holding it standing, even after approval (Part 3)
5. **Re-certify**: the quarterly access review re-evaluates whether the access still makes sense, with a safe default (remove) if nobody responds (Part 1)

This is the actual answer to the audit question from the scenario — every step in this lifecycle is logged, time-bound, and requires an affirmative justification, so "who has access and why" is always answerable.

---

## Part 5: Cleanup

1. **Entitlement management** → **Access packages** → delete `Finance Reporting Tool Access`
2. **Entitlement management** → **Catalogs** → delete `Finance Resources`
3. **Access reviews** → delete `Finance Reporting Access — Quarterly Review`
4. **PIM | Groups** → remove `finance-reporting-access` from PIM management if no longer needed
5. **Entra ID** → **Groups** → delete `finance-reporting-access`
6. Delete any throwaway test users created for this lab

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Recurring access reviews with a safe non-response default** | Turns "who still needs this" from a never-asked question into an automated, auditable process |
| **Entitlement management access packages** | Replaces ad hoc access grants with a requestable, approvable, time-bound package |
| **Time-bound access package assignments** | Prevents "temporary" access from silently becoming permanent |
| **PIM for group membership** | Extends just-in-time access beyond admin roles to any group that grants meaningful privilege |
| **Understanding the full request → approve → activate → re-certify lifecycle** | The actual governance model an identity admin is responsible for maintaining |

---

## Common Mistakes to Avoid
- **Setting "if reviewers don't respond" to "no change" instead of "remove access"**: silently defeats the purpose of the review — stale access simply persists forever if a reviewer is on vacation
- **Access packages with no expiration**: without an expiry, entitlement management becomes just another way to grant permanent access, minus the audit trail benefit
- **Confusing PIM-eligible group membership with entitlement management**: they solve different problems — entitlement management controls the *request/approval* step, PIM controls whether granted access is *standing or activated on demand*
- **Building an access review with no designated reviewer who'll actually respond**: an access review nobody looks at just becomes an automatic mass-removal event on schedule

---

## Next Steps
- Add a second-stage approver to the access package for higher-sensitivity resources (multi-stage approval)
- Configure access package **connected organizations** to allow external partner users to request access through the same governed flow
- Continue to [Lab 4: Identity Protection & Graph Automation](lab-4-identity-protection-graph-automation.md) to automate reporting on access review outcomes via Microsoft Graph
- Review Entra ID's **access reviews for PIM-eligible role assignments** (distinct from group-membership reviews) — apply the same quarterly cadence to the admin roles from AZ-500 Lab 1
