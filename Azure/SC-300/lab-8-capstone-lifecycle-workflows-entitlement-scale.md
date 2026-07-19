# Lab 8: Lifecycle Workflows & Entitlement at Scale (Capstone)

Check box if done: [ ]

## Overview
Every prior lab in this track solved one piece of identity administration in isolation: least-privilege roles (Lab 1), app onboarding (Lab 2), access certification (Lab 3), risk-aware Conditional Access and automation (Lab 4), hybrid sync (Lab 5), guest access (Lab 6), and application/workload governance (Lab 7). A real new-hire's first day touches all of it at once — account provisioning, group and access-package assignment, a Conditional Access policy that has to apply correctly from minute one, and eventually an offboarding event that has to actually remove everything granted along the way. This capstone builds Entra ID Lifecycle Workflows to automate that joiner→mover→leaver sequence, extends it with a custom Logic App task for the one step no built-in template covers, and closes by tracing a single new-hire scenario through every governance mechanism this track built.

**Estimated time**: 75–90 minutes
**Cost**: ~$0 (Lifecycle Workflows requires an Entra ID Governance license — the P2 trial from Labs 3–4 includes a time-limited Governance trial; the optional Logic App in Part 2 runs on the Consumption plan, a fraction of a cent for the handful of test executions this lab triggers — delete it during cleanup so it doesn't linger)

---

## Scenario
HR just handed the identity team a new requirement: when a new employee is added to the HR system, their Entra ID account, group memberships, and starter access should be ready *before* their first day, not assembled ad hoc by whoever's on help-desk duty that morning. Equally important: when someone leaves, their access needs to disappear on a defined schedule, not "whenever someone remembers to disable the account." You're building both directions as automated workflows, adding one custom step neither built-in template covers (notifying a facilities system to revoke a physical badge), and then writing the synthesis leadership asked for: how does this tie into everything else the identity team has already built.

---

## Objectives
- Understand what Entra ID Lifecycle Workflows automate and how they differ from Conditional Access or entitlement management
- Build a joiner (pre-hire onboarding) workflow using a built-in template
- Build a leaver (offboarding) workflow with a defined, auditable execution schedule
- Extend a workflow with a custom task extension that calls a Logic App at a specific trigger point
- Synthesize Labs 1, 3, 4, and 6 into one end-to-end onboarding → access-provisioning → periodic-review → offboarding pipeline

---

## Part 1: What Lifecycle Workflows Actually Automate

### Step 1: Distinguish Lifecycle Workflows from What You've Already Built
It's easy to conflate this with entitlement management (Lab 3) or Conditional Access (Lab 4) — they're related but solve different problems:
- **Entitlement management** (Lab 3) answers "how does a user *request* access" — it's pull-based, initiated by the person who wants access
- **Conditional Access** (Lab 4) answers "under what conditions can a session proceed" — it's a gate evaluated at sign-in time, not a provisioning action
- **Lifecycle Workflows** answers "what should happen automatically *because* someone's employment status changed" — it's push-based, triggered by an HR attribute (hire date, department, termination date) rather than a user's own request

A new hire doesn't request their starter group membership through My Access — a lifecycle workflow provisions it for them, tied to their employment start date, so it's already in place before they need to ask.

### Step 2: Confirm Licensing
1. **Entra ID** → **Identity Governance** → **Lifecycle workflows** → if you see a licensing prompt instead of the workflow gallery, this feature requires the **Entra ID Governance** license tier specifically — the P2 trial activated for Labs 3–4 typically includes a time-limited Governance trial, but confirm under **Entra ID** → **Licenses** → **All products** before continuing
2. If Governance licensing isn't available in your tenant, read through Parts 2–4 as a configuration walkthrough — the workflow design and trigger logic are the exam-relevant content even without executing every step live

---

## Part 2: Build a Joiner Workflow

### Step 3: Set Up the Trigger Population
Lifecycle Workflows trigger off Entra ID user attributes — most commonly `employeeHireDate`. Set one on a test user to simulate an upcoming start date.

1. **Entra ID** → **Users** → create `new.hire1@yourdomain.onmicrosoft.com` (or reuse a throwaway test user from an earlier lab)
2. Open the user → **Properties** → **Job info** → set **Employee hire date** to two days from today (Lifecycle Workflows can trigger a defined number of days *before* the hire date, which is the entire point — provisioning ahead of day one, not on it)

### Step 4: Create the Joiner Workflow from a Template
1. **Identity Governance** → **Lifecycle workflows** → **Workflows** → **+ Create workflow**
2. Select the **Onboard pre-hire employee** template
3. **Basics**: **Name**: `New Hire Onboarding — Pre-Day-One`, **Trigger type**: `Employee hire date`, **Days before/after**: `2 days before`
4. **Scope**: `Users assigned to this workflow` or a dynamic rule targeting a specific department attribute (dynamic scoping is what makes this run unattended for every future hire, not just the one test user)
5. **Review tasks**: the template pre-populates common tasks — **Generate Temporary Access Pass**, **Send welcome email**, **Add user to groups** — keep the defaults for this walkthrough
6. **Add a task**: **Add user to groups** → select a starter group (e.g., a general-access group from an earlier lab, or create `new-hire-starter-access`)
7. **Create**

**Expected result**: the workflow is created in a disabled state by default — review the task list once before enabling, the same discipline as testing a Conditional Access policy in report-only mode before turning it on.

### Step 5: Run and Verify
1. Open the workflow → **Enable workflow**
2. **Run workflow** → **Run for a specific user** → select `new.hire1`
3. **Workflow history** → select the run → confirm each task shows a **Succeeded** status

**Validation checkpoint**: **Entra ID** → **Groups** → your starter group → **Members** — confirm `new.hire1` was added automatically, with no admin manually assigning group membership. This is the mechanism that answers the scenario's ask directly: access exists before day one because a workflow put it there, not because someone remembered to.

---

## Part 3: Build a Leaver Workflow

### Step 6: Create the Offboarding Workflow
1. **Lifecycle workflows** → **Workflows** → **+ Create workflow** → **Offboard an employee** template
2. **Basics**: **Name**: `Employee Offboarding — Termination Date`, **Trigger type**: `Employee last day of work`, **Days before/after**: `On last day`
3. **Review tasks**: keep **Remove user from all groups**, **Remove user from all Teams**, **Disable user account** — add **Revoke all sessions** explicitly if it isn't already included, since a disabled account with an unexpired session token can still complete in-flight requests until the token naturally expires
4. **Create** → **Enable workflow**

### Step 7: Understand Why the Schedule Matters as Much as the Tasks
1. Set a test user's **Employee leave date** to today
2. **Run workflow** → **Run for a specific user** → select that user
3. **Workflow history** → confirm **Disable user account** and **Remove user from all groups** both show **Succeeded**

**Expected result**: every access grant the joiner workflow and every subsequent manual/entitlement-management assignment gave this user is removed in one automated run, on a schedule tied to an HR-driven date — not "whenever the manager remembers to email IT."

**This is the actual fix for the guest-access gap called out in [Lab 6](lab-6-external-identities.md)**: Lifecycle Workflows apply to employee (member) accounts with an HR-tracked termination date; guests still need Lab 6's access-review mechanism specifically because they have no HR offboarding trigger to key off of. The two mechanisms cover two different populations — knowing which one applies to which user type is itself an exam-relevant distinction.

---

## Part 4: Custom Extensibility — a Logic App Task

### Step 8: Why a Built-In Task Isn't Enough Here
The offboarding template covers Entra ID and Microsoft 365 access well, but the scenario also needs the facilities team's badge-access system notified — a system with no native Lifecycle Workflows task, because it isn't a Microsoft identity surface at all. **Custom task extensions** solve exactly this: a workflow task that calls a Logic App at a specific point in the run, and the Logic App does whatever custom action (an HTTP call to a third-party API, a database update, an email to a specific mailbox) the built-in tasks don't cover.

### Step 9: Build the Logic App
1. Portal → **Logic Apps** → **+ Add** → **Consumption** plan → **Name**: `badge-access-revocation`, same resource group as any other lab throwaway resources
2. Once deployed, open **Logic App Designer** → trigger: **When a HTTP request is received**
3. Save to generate the callback URL — Lifecycle Workflows will call this URL when the custom task fires, passing the affected user's details in the request body
4. Add a placeholder action after the trigger — for this lab, a simple **Compose** action outputting `"Badge access revocation requested for: " + triggerBody()?['data']?['subject']?['userPrincipalName']` stands in for the real facilities-system API call
5. **Save**

### Step 10: Register the Logic App as a Custom Task Extension
1. **Identity Governance** → **Lifecycle workflows** → **Custom task extensions** → **+ Create custom task extension**
2. **Basics**: **Name**: `Revoke Badge Access`, **Logic App**: select `badge-access-revocation`
3. **Task extension details**: configure the callback so the workflow waits for the Logic App's response before marking the task complete (or fire-and-forget, depending on whether downstream tasks in the workflow depend on the outcome — for badge revocation, fire-and-forget is acceptable since nothing else in the offboarding sequence depends on facilities confirming)
4. **Create**

### Step 11: Add the Custom Task to the Offboarding Workflow
1. **Employee Offboarding — Termination Date** workflow → **Tasks** → **+ Add task** → select the custom task category → `Revoke Badge Access`
2. Position it after **Disable user account** in the task order — sequencing matters here, since a badge system that also checks Entra ID account status would otherwise fire before the account is actually disabled
3. **Save**

**Validation checkpoint**: re-run the offboarding workflow (Step 7) against a test user and confirm **Workflow history** now shows four tasks, including `Revoke Badge Access`, with a **Succeeded** status — and check the Logic App's **Run history** to confirm it actually received the callback with the correct user's UPN in the payload.

**This is the pattern to remember for the exam and for real design conversations**: Lifecycle Workflows' built-in templates cover the Microsoft-ecosystem 80%; custom task extensions are how the remaining 20% — anything outside Entra ID, Exchange, and Teams — gets folded into the same automated, auditable sequence instead of living as a manual step someone has to remember.

---

## Part 5: Synthesis — One Employee, Every Lab

### Step 12: Trace a Single New-Hire Scenario End to End
This is the deliverable that ties the track together. Walk a hypothetical new hire, `sales.rep2`, through every mechanism built across Labs 1–7:

| Stage | Mechanism | Built In |
|---|---|---|
| **Pre-day-one provisioning** | Lifecycle Workflow triggers 2 days before `employeeHireDate`, creates starter group membership automatically | This lab, Part 2 |
| **Scoped administrative delegation** | Help-desk support for this employee is scoped to their department via an administrative unit, not tenant-wide | [Lab 1](lab-1-identity-governance-foundations.md) |
| **App access request** | Beyond the starter group, the employee requests additional app-specific access (e.g., a reporting tool) via an entitlement management access package, approved by their manager | [Lab 3](lab-3-access-governance.md) |
| **Session-time enforcement** | Every sign-in is evaluated against layered Conditional Access — MFA, and if the employee is later granted an admin role, location and device compliance too | [Lab 4](lab-4-identity-protection-graph-automation.md) |
| **Periodic re-certification** | The access package and any group membership are subject to quarterly access reviews — access doesn't just persist because it was once approved | [Lab 3](lab-3-access-governance.md) |
| **External collaboration boundary** | If this employee later needs to loop in an external contractor on a project, that's a B2B guest invite with its own guest-scoped access review — a structurally different lifecycle than the employee's own | [Lab 6](lab-6-external-identities.md) |
| **Application/workload governance** | Any app registration or pipeline this employee's team owns is subject to ownership assignment, credential hygiene, and consent review | [Lab 7](lab-7-workload-identity-app-lifecycle-governance.md) |
| **Offboarding** | On their last day, a Lifecycle Workflow disables the account, strips every group/Teams membership, revokes sessions, and (via the custom task extension) notifies facilities — all on the HR-tracked termination date | This lab, Part 3–4 |

### Step 13: The Thread Connecting All Eight Labs
No single lab in this track is "the identity governance solution" — each one closes one specific gap where access could otherwise be granted without review, persist without justification, or linger without an owner. Lifecycle Workflows is the capstone specifically because it's the only mechanism that's genuinely **event-driven end to end**: it doesn't wait for an admin to notice a hire or a termination, the way every other lab's controls (except Conditional Access's real-time evaluation) still ultimately depend on someone configuring and periodically checking them. The identity administrator's actual job, demonstrated across this track, is designing that full sequence — least-privilege delegation, request/approval, session enforcement, periodic re-certification, and automated provisioning/deprovisioning — as one coherent system instead of eight disconnected settings.

---

## Part 6: Cleanup

1. **Identity Governance** → **Lifecycle workflows** → disable and delete `New Hire Onboarding — Pre-Day-One` and `Employee Offboarding — Termination Date`
2. **Custom task extensions** → delete `Revoke Badge Access`
3. Portal → **Logic Apps** → delete `badge-access-revocation` (and its resource group, if created solely for this lab): `az group delete --name <logic-app-rg> --yes --no-wait`
4. **Entra ID** → **Groups** → delete `new-hire-starter-access` if created for this lab
5. **Entra ID** → **Users** → delete `new.hire1` and any other throwaway test users
6. Review every earlier lab's cleanup section (Labs 1, 3, 4, 5, 6, 7) and confirm no test groups, access packages, Conditional Access policies, or app registrations were left behind across the full track

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Lifecycle Workflows joiner template, triggered on `employeeHireDate`** | Provisions starter access before day one instead of ad hoc on a new hire's first morning |
| **Lifecycle Workflows leaver template, triggered on termination date** | Removes access on a defined, auditable HR-driven schedule instead of "whenever IT remembers" |
| **Custom task extensions via Logic Apps** | Extends automated offboarding to non-Microsoft systems the built-in templates can't reach |
| **Distinguishing Lifecycle Workflows from entitlement management and Conditional Access** | Three different mechanisms (push-provisioning, pull-request, session-gate) that solve three different problems and are frequently confused in exam scenarios |
| **Tracing one identity through every governance control in the track** | The actual synthesis skill this track builds toward — designing a coherent lifecycle, not isolated settings |

---

## Common Mistakes to Avoid
- **Enabling a workflow before reviewing its task list**: workflows are created disabled specifically so you can verify the task sequence first — enabling immediately risks running an unreviewed action against real users
- **Confusing Lifecycle Workflows with entitlement management**: one pushes access automatically based on an HR attribute, the other lets a user pull/request access — a joiner workflow that tries to handle every possible access request turns into an unmaintainable task list; let entitlement management handle the request-driven cases
- **Treating a custom task extension's Logic App as fire-and-forget without checking its run history**: a badge-revocation call that silently fails looks identical to one that succeeded, from the workflow's perspective, unless the callback is configured to wait for and validate the response
- **Applying Lifecycle Workflows logic to guest users**: guests have no `employeeHireDate`/termination date to trigger off — that population is governed by Lab 6's guest-scoped access reviews, not this lab's mechanism
- **Presenting each lab's control as standalone instead of part of one pipeline**: the exam and real design conversations both expect you to explain how a gap in one control (e.g., no owner on an app registration) undermines another (e.g., a stale credential nobody rotates) — Part 5's synthesis is the actual skill being assessed here, not optional wrap-up color

---

## Next Steps
This closes out the SC-300 track. Every lab built one piece of the identity administration discipline — least-privilege role design ([Lab 1](lab-1-identity-governance-foundations.md)), app onboarding ([Lab 2](lab-2-app-integration-sso.md)), access governance and entitlement ([Lab 3](lab-3-access-governance.md)), risk-aware Conditional Access and Graph automation ([Lab 4](lab-4-identity-protection-graph-automation.md)), hybrid identity ([Lab 5](lab-5-hybrid-identity-adconnect.md)), external identities ([Lab 6](lab-6-external-identities.md)), and application/workload governance ([Lab 7](lab-7-workload-identity-app-lifecycle-governance.md)) — with this lab's Lifecycle Workflows automating the connective tissue between all of them.

- For the *security-control* counterpart to this track's *administrative* depth — how PIM, Conditional Access, and identity protection get used defensively rather than operationally — see [AZ-500](../AZ-500/README.md), particularly [AZ-500 Lab 1: Identity & Access Protection](../AZ-500/lab-1-identity-access-protection.md)
- For the *architecture-level* decisions this track assumed were already made (hybrid vs. cloud-only identity model, default cross-tenant trust posture, Zero Trust identity strategy) see [SC-100: Cybersecurity Architect Expert](../SC-100/README.md), particularly [SC-100 Lab 2: Identity & Access Architecture Design](../SC-100/lab-2-identity-access-architecture-design.md)
- Extend Part 4's custom task extension pattern to a second real-world non-Microsoft system (a ticketing system, a VPN access list) — the badge-revocation example here is deliberately the simplest possible illustration of the mechanism
