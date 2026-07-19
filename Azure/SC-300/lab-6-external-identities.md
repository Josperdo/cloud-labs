# Lab 6: External Identities

Check box if done: [ ]

## Overview
Not everyone who needs access to your tenant's resources is your employee. Contractors, partner-org collaborators, and vendors all need a way in — and the lazy answer, creating a local account with a password in your directory for someone who already has an identity somewhere else, is both a support burden and a security liability nobody wants to own. This lab builds the real pattern: B2B collaboration guest invites, cross-tenant access settings that decide how much you trust another organization's own MFA and Conditional Access, and the governance controls that stop guest access from becoming the thing nobody remembers to clean up.

**Estimated time**: 60–75 minutes
**Cost**: ~$0

---

## Scenario
Your company is starting a joint project with an external design agency. Three of their staff need access to a specific SharePoint site and one internal app — nothing else. Separately, security has flagged that guest invites have been wide open ("anyone can invite anyone") since the tenant was created, and nobody has ever looked at how much your tenant currently trusts sign-ins coming from *other* organizations' Conditional Access and MFA decisions. You're fixing all three: locking down who can invite, inviting the agency's staff properly, and reviewing cross-tenant trust.

---

## Objectives
- Configure external collaboration settings to control who can invite guests and who can be invited
- Invite a B2B guest user and understand how their identity is represented in your tenant
- Configure cross-tenant access settings to control inbound and outbound trust with a specific partner organization
- Understand B2B direct connect and when it's the better fit over standard B2B invites
- Apply access review governance to guest accounts specifically

---

## Part 1: External Collaboration Settings — Who Can Invite Whom

### Step 1: Review the Current (Wide-Open) Default
1. **Entra ID** → **External Identities** → **External collaboration settings**
2. Note the default values: **Guest user access** is often left at the least-restrictive tier, and **Guest invite settings** frequently allows `Anyone in the organization can invite guest users` — the setting flagged in the scenario

### Step 2: Restrict Who Can Send Invites
1. **Guest invite settings** → change to `Only users assigned to specific admin roles can invite guest users`
2. This scopes invitations to holders of the **Guest Inviter** role (or higher) instead of every employee — pair this with assigning the **Guest Inviter** role to the specific people (project leads, not all managers) who legitimately need to invite external collaborators

### Step 3: Restrict What a Guest Can See Once Inside
1. **Guest user access** → set to `Guest user access is restricted to properties and memberships of their own directory objects`
2. This is the least-privileged tier — a guest can see their own profile and the groups/apps they're a member of, but cannot browse the full user or group directory the way a member can

**Validation checkpoint**: **External collaboration settings** should now show a restricted invite population and restricted guest directory visibility — the two changes together close the "wide open" gap from the scenario without blocking the legitimate collaboration the project needs.

---

## Part 2: Invite a B2B Guest User

### Step 4: Send the Invitation
1. **Entra ID** → **Users** → **+ New user** → **Invite external user**
2. **Email address**: `contractor@partneragency.example` (placeholder — use an address you actually control for testing, e.g., a secondary personal account)
3. **Display name**: `Partner Agency Contractor`
4. **Send invite message** → optionally customize the message the invitee receives
5. **Review + invite**

### Step 5: Understand the Object That Gets Created
1. **Entra ID** → **Users** → find the new guest — note the **User type** column shows `Guest`, and the **Identities** field on their profile shows the external email as a federated identity source, not a password Entra ID manages
2. Until the invitee accepts (clicks the redemption link in the invite email), their account shows **Invitation accepted**: `Pending`

**This is the core distinction from a normal member account**: a B2B guest authenticates with credentials from their *own* organization's identity provider (or a personal Microsoft/Google account) — your tenant never stores or manages their password, and if their home organization disables them, they lose access here too without you doing anything.

### Step 6: Scope the Guest to Exactly What They Need
1. Assign the guest to the app role or group they actually need — using the same mechanisms from earlier labs: **Enterprise applications** → app roles (Lab 2), or a security group feeding an access package (Lab 3)
2. Do **not** assign the guest any administrative role, and confirm they were not accidentally added to a broad "All Company" distribution group during onboarding

---

## Part 3: Cross-Tenant Access Settings — Deciding How Much You Trust Another Tenant

### Step 7: Understand What Cross-Tenant Access Settings Control
B2B invites happen per-user. Cross-tenant access settings operate at the **organization** level — they decide, for a specific partner tenant, whether you trust the MFA and device-compliance claims that tenant's own Conditional Access already enforced, instead of re-challenging the guest with your own MFA every time.

### Step 8: Configure Inbound Trust for the Partner Organization
1. **Entra ID** → **External Identities** → **Cross-tenant access settings** → **Organizational settings** → **Add organization**
2. Enter the partner agency's tenant ID (placeholder: `<partner-tenant-id>`) or domain
3. Open the new organization's settings → **Inbound access** → **Trust settings**
4. Enable **Trust multi-factor authentication from Microsoft Entra tenants** — if the partner's own Conditional Access already enforced MFA at their sign-in, your tenant accepts that claim instead of prompting again
5. Leave **Trust compliant devices** and **Trust hybrid Azure AD joined devices** off unless you've specifically validated the partner's device compliance posture — trusting MFA is a much smaller trust extension than trusting device state you've never inspected

### Step 9: Configure Outbound Access (Your Users Collaborating with Their Tenant)
1. Same organization's settings → **Outbound access** → define which of *your* users/groups are allowed to be invited into the *partner's* tenant, if that direction of collaboration is expected
2. If it's not expected (this scenario is inbound-only — their staff into your tenant), leave outbound access at the default-deny and don't configure it — least privilege applies to tenant trust relationships the same way it applies to role assignments

**Validation checkpoint**: **Cross-tenant access settings** → **Organizational settings** lists the partner tenant with the specific inbound trust settings configured — not the tenant-wide default, which typically doesn't trust external MFA claims until explicitly configured per organization.

### Step 10: Why This Matters Beyond the One Partner
Cross-tenant access settings also have a **default** tab that applies to every organization *not* individually configured — review it (**Cross-tenant access settings** → **Default settings**) and confirm it's set restrictively. An organization-specific override like the one just built should be the exception you deliberately grant, not the default posture for every tenant in the world that happens to send you a B2B invite.

---

## Part 4: B2B Direct Connect — When Invites Aren't the Right Model

### Step 11: Understand the Different Use Case
Standard B2B collaboration creates a guest object in your directory — right for "external individual needs ongoing access to specific resources." **B2B direct connect** is different: it establishes a mutual, two-way trust between two tenants so users from either side can join a **Microsoft Teams shared channel** without ever appearing as a guest object in the other tenant's directory at all. It's the right fit for cross-company team collaboration (a shared channel both companies work in daily), not for "give this one contractor access to an app."

### Step 12: Configure the Mutual Trust
1. **Cross-tenant access settings** → the partner organization's settings → **Inbound access** and **Outbound access** → **B2B direct connect** tab (separate from the B2B collaboration tab used in Part 3)
2. Both tenants must configure this setting pointing at each other — it's inherently mutual, unlike a one-directional B2B invite
3. Once configured, a Teams owner on either side can add the other organization's users to a **shared channel**, and those users authenticate with their home-tenant credentials with no invite/redemption step

**Validation checkpoint (conceptual)**: this requires a second real tenant to fully demonstrate — the concept to retain for the exam and for design conversations is *B2B collaboration = guest object + invite redemption; B2B direct connect = mutual tenant trust + no guest object, scoped to shared-channel collaboration*.

---

## Part 5: Guest Access Governance

### Step 13: Apply Access Reviews to Guests Specifically
1. **Identity Governance** → **Access reviews** → **+ New access review**
2. **Select what to review**: `Teams + Groups` or `Applications` (whichever the guest was assigned through) → **Scope**: `Guest users only`
3. **Reviewers**: the project lead who requested the collaboration, not a generic IT reviewer — the person who knows whether the agency's engagement is still active is the right reviewer, the same principle as Lab 3
4. **If reviewers don't respond**: `Remove access` — a guest with no active reviewer is exactly the account that should lose access by default, not keep it

**Why guest-scoped reviews matter more than member reviews**: a guest account has no HR offboarding process to trigger cleanup when the engagement ends — nobody automatically disables them the way a member gets disabled when they leave payroll. An access review is often the *only* mechanism that will ever catch a guest whose project ended six months ago.

---

## Part 6: Cleanup

1. **Entra ID** → **Users** → delete the guest user (`Partner Agency Contractor`)
2. **Identity Governance** → **Access reviews** → delete the guest-scoped review created in Part 5
3. **External Identities** → **Cross-tenant access settings** → **Organizational settings** → remove the partner organization entry if it was added purely for this lab
4. **External collaboration settings** → revert **Guest invite settings** and **Guest user access** if this is a shared/production tenant and the restricted values weren't already your intended posture

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Restricting who can invite guests** | Closes the most common "wide open by default" gap in external collaboration settings |
| **B2B guest invitation and redemption** | The standard mechanism for giving an external individual scoped access without your tenant managing their password |
| **Cross-tenant access settings (inbound trust)** | Decides how much of another organization's own MFA/device posture you accept instead of re-challenging |
| **B2B direct connect vs. standard B2B collaboration** | Two different external-identity models for two different collaboration shapes — ongoing individual access vs. mutual team collaboration |
| **Access reviews scoped to guests only** | Guests have no HR-driven offboarding trigger — a review is often the only cleanup mechanism that exists |

---

## Common Mistakes to Avoid
- **Leaving "anyone can invite guests" enabled**: turns every employee into an unmonitored external-access-granting admin — scope invite rights to a specific role
- **Trusting a partner tenant's device compliance without ever validating their compliance policy**: MFA trust and device-compliance trust are separate settings for a reason — don't enable both just because one made sense
- **Using standard B2B invites for a use case that's really B2B direct connect (or vice versa)**: an invited guest object for someone who just needs a shared Teams channel creates directory clutter; direct connect for someone who needs access to a specific app doesn't work at all — the app assignment model requires an actual guest object
- **Never reviewing guest access because "no one leaves the company"**: guests were never employed by your company in the first place — the review has to be tied to the *engagement*, not to HR-driven termination

---

## Next Steps
- Configure **entitlement management connected organizations** (touched on in [Lab 3](lab-3-access-governance.md)'s Next Steps) so external users request access through the same governed access-package flow as internal users, instead of a manual invite
- Review your tenant's **cross-tenant access default settings**, not just the per-organization override built in this lab, and confirm the default posture is restrictive
- Continue to [Lab 7: Workload Identity & Application Lifecycle Governance](lab-7-workload-identity-app-lifecycle-governance.md) — external *human* identities are one category of "not your employee" access; the next lab covers non-human identities calling your APIs
- For the architecture-level decision of default cross-tenant trust posture across an entire multi-tenant estate, see [SC-100 Lab 2: Identity & Access Architecture Design](../SC-100/lab-2-identity-access-architecture-design.md)
