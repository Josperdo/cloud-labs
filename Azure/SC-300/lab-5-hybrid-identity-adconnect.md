# Lab 5: Hybrid Identity with Microsoft Entra Connect

Check box if done: [ ]

## Overview
Most organizations don't start cloud-only — they have an existing on-premises Active Directory full of years of user and group history, and the identity team's job is to project that directory into Entra ID without dragging every stale service account and disabled test user along with it. This lab is the identity-*administration* side of hybrid identity: designing sync scope so only the right objects flow to the cloud, choosing between password hash sync and pass-through authentication, using staged rollout to cut over authentication methods without a big-bang risk, and monitoring sync health so a broken sync cycle gets caught in hours, not weeks. It deliberately does not cover Tier-0 server hardening for the Connect server itself — that's a security-architecture concern, not a provisioning one.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (Entra ID P1 required for staged rollout — the P2 trial from Labs 3–4 already covers this)

---

## Scenario
Your company is being acquired and its on-premises Active Directory — 15 years of accumulated OUs, disabled accounts, and a few service accounts nobody can explain — needs to sync to the parent company's Entra ID tenant. The parent's identity team has one hard requirement: don't sync everything. Only actual employee accounts in the `Corp Users` and `Corp Groups` OUs should ever reach the cloud, authentication should move from password hash sync to a validated state incrementally (not flipped for all 4,000 users at once), and someone needs to be able to answer "is sync healthy right now" without SSH-ing into the Connect server and reading event logs.

> **Environment note**: A full hands-on walkthrough of the Microsoft Entra Connect installer requires a Windows Server joined to an on-premises (or simulated) Active Directory domain — outside the $0, Entra-ID-only scope this track has held to in Labs 1–4. Parts 1–2 are written as an exact configuration runbook you would execute on that Connect server; Parts 3–4 (staged rollout, sync health monitoring) are fully executable against any Entra ID tenant, hybrid or cloud-only, using the admin center and Microsoft Graph.

---

## Objectives
- Design a sync scope using OU-based and attribute-based filtering to avoid syncing the entire on-premises directory
- Understand the trade-offs between password hash sync (PHS) and pass-through authentication (PTA)
- Use staged rollout to cut a subset of users onto a new authentication method before a tenant-wide change
- Monitor sync health and freshness using the admin center and Microsoft Graph PowerShell
- Recognize the most common sync-scoping mistakes that cause "why is this disabled user still in Entra ID" tickets

---

## Part 1: Scoping the Sync — Filtering Design

### Step 1: Why "Sync Everything" Is the Default Mistake
Microsoft Entra Connect's **Express Settings** install path syncs every user, group, and contact in the forest by default. On a small greenfield domain that's harmless. On a 15-year-old directory like the one in the scenario, it means every disabled service account, every test user a former admin never cleaned up, and every distribution list from a decommissioned department shows up in Entra ID — inflating license counts, cluttering directory search, and giving stale accounts a live path into cloud apps if they're ever re-enabled by mistake.

### Step 2: Design the OU-Based Filter
OU-based filtering is the coarse-grained tool: sync entire organizational units, exclude everything else.

1. On the Connect server: **Microsoft Entra Connect** → **Configure** → **Customize synchronization options** → **Domain/OU Filtering**
2. Select **Sync selected domains and OUs**
3. Check only:
   - `contoso.local/Corp Users`
   - `contoso.local/Corp Groups`
4. Leave every other OU unchecked — including `Service Accounts`, `Disabled Users`, and `Test`

**Expected result**: only objects inside the two selected OUs are eligible for sync. An object moved out of scope on the next sync cycle is treated as deleted in Entra ID (soft-deleted, recoverable for 30 days) — not left behind as an orphan.

### Step 3: Design an Attribute-Based Filter for the Edge Case
OU filtering doesn't catch everything — suppose `Corp Users` also contains a handful of shared mailboxes that shouldn't get Entra ID licenses or sign-in ability. Attribute-based filtering (configured via the **Synchronization Rules Editor** on the Connect server) adds a second, finer condition on top of OU scope.

1. On the Connect server: **Synchronization Rules Editor** → **Add new rule**
2. **Name**: `Exclude Shared Mailboxes from Sync`, **Direction**: `Inbound`, **Precedence**: a value lower than the default in-from-AD-User rule (lower number = evaluated first)
3. **Scoping filter**: `msExchRecipientTypeDetails` **Equals** `34359738368` (the shared-mailbox recipient type value)
4. **Transformations**: set `cloudFiltered` to `True` when the scoping filter matches — this is the flag that excludes the object from provisioning to Entra ID without touching the OU-level scope

**Validation checkpoint**: after the next delta sync, `Get-ADSyncCSObject` or a synchronization service manager export shows the shared mailbox accounts flagged `cloud-filtered`, not provisioned.

### Step 4: Why Filtering Belongs to Identity Administration, Not Just IT
Getting sync scope wrong is an access-governance failure, not just a plumbing bug: a disabled on-prem account that syncs into Entra ID as enabled, or a service account nobody remembers, becomes exactly the kind of stale identity that Lab 3's access reviews exist to catch — filtering it out at the source is cheaper than catching it downstream.

---

## Part 2: Password Hash Sync vs. Pass-Through Authentication

### Step 5: Compare the Two Authentication Options
| | Password Hash Sync (PHS) | Pass-Through Authentication (PTA) |
|---|---|---|
| **Where auth happens** | Entra ID, using a synced hash of the on-prem password hash | On-premises domain controller, via a lightweight agent |
| **On-prem dependency for sign-in** | None — cloud sign-in works even if on-prem AD is down | Requires at least one PTA agent reachable at sign-in time |
| **Leaked-credential detection** | Entra ID can compare against known-breached password lists directly | Same detection, but the comparison happens against the synced hash, not a live on-prem check |
| **Typical choice** | Most organizations — simpler, resilient to on-prem outages | Organizations with a hard compliance requirement that the password itself never leaves on-premises, even in hashed form |

### Step 6: Configure the Choice on the Connect Server
1. **Microsoft Entra Connect** → **Configure** → **Change user sign-in**
2. Select **Password Hash Synchronization** (the default and the right choice unless a specific compliance requirement says otherwise)
3. Leave **Enable single sign-on** checked (configures Seamless SSO via a computer object in on-prem AD, so managed devices skip the sign-in prompt entirely on the corporate network)
4. **Next** → **Configure**

**Real-world practice**: document *why* PHS was chosen over PTA in the runbook you hand to whoever inherits this tenant — "we didn't evaluate the alternative" is a worse answer in an audit than either choice defensibly made.

---

## Part 3: Staged Rollout — Cutting Over Without a Big Bang

### Step 7: Why Staged Rollout Exists
Changing every user's authentication method tenant-wide in one step is exactly the kind of change that goes wrong at 4,000 users and fine at 40. Staged rollout lets you move a cloud-side security group's members onto a new authentication method (PHS, PTA, or Seamless SSO) while everyone else keeps using the old one — entirely from the Entra admin center, no changes needed on the Connect server itself.

### Step 8: Build the Pilot Group and Enable Staged Rollout
1. **Entra ID** → **Groups** → **+ New group** → **Security** group named `staged-rollout-pilot-phs`, add 2–3 test users
2. **Entra ID** → **Microsoft Entra Connect** → **Manage** → **Staged rollout of cloud authentication**
3. **Enable staged rollout for managed user sign-in** → toggle **Password Hash Sync** to `On`
4. Under that feature → **Manage groups** → add `staged-rollout-pilot-phs`

**Expected result**: only members of `staged-rollout-pilot-phs` authenticate using the new method; every other synced user is unaffected until you add their group or expand membership.

### Step 9: Validate the Pilot Before Expanding
1. Have a pilot-group member sign in and confirm success
2. **Entra ID** → **Sign-in logs** → filter by that user → confirm the **Authentication method** field matches the staged-rollout configuration, not the tenant-wide default
3. Only after a validated pilot window would you complete the equivalent change tenant-wide on the Connect server itself and remove the staged-rollout group (staged rollout is a bridge, not a permanent state)

**This is the step most tenants skip** — staged rollout gets enabled for the pilot group and then just... never gets promoted or cleaned up, leaving two authentication methods live indefinitely with no one able to explain why.

---

## Part 4: Sync Health Monitoring

### Step 10: Check Sync Status from the Admin Center
1. **Entra ID** → **Microsoft Entra Connect** → **Overview**
2. Confirm **Last sync** shows a recent timestamp (default sync cycle is every 30 minutes) — a timestamp hours old with no explanation is the first sign of a broken sync

### Step 11: Query Sync Freshness via Microsoft Graph
```powershell
Connect-MgGraph -Scopes "Organization.Read.All", "User.Read.All"

# Confirm the tenant is hybrid and check overall directory sync configuration
Get-MgOrganization -Property OnPremisesSyncEnabled, OnPremisesLastSyncDateTime |
    Select-Object OnPremisesSyncEnabled, OnPremisesLastSyncDateTime

# Spot-check individual synced users for staleness
Get-MgUser -Filter "onPremisesSyncEnabled eq true" -Property DisplayName, OnPremisesSyncEnabled, OnPremisesLastSyncDateTime -All |
    Select-Object DisplayName, OnPremisesLastSyncDateTime |
    Sort-Object OnPremisesLastSyncDateTime |
    Select-Object -First 10
```

**Validation checkpoint**: the ten oldest `OnPremisesLastSyncDateTime` values should all still be within one sync-cycle window of "now." A cluster of users with a `LastSyncDateTime` far older than the rest usually points to an object-level sync error (a duplicate attribute value, an export failure) rather than a global outage — the global sync overview would already show that.

### Step 12: Know Where Real Errors Surface
On the Connect server itself, **Synchronization Service Manager** → **Operations** tab lists every sync run with per-object error counts, and **Microsoft Entra Connect Health** (a separate installed agent, cloud-reporting) surfaces those same errors in the admin center without needing to RDP to the server — worth calling out by name even though installing the Health agent is outside this lab's scope, since "how would you monitor this without logging into the server" is a fair interview question here.

---

## Part 5: Cleanup

1. **Entra ID** → **Microsoft Entra Connect** → **Staged rollout of cloud authentication** → remove `staged-rollout-pilot-phs` from the Password Hash Sync feature, then disable the feature if no longer needed
2. **Entra ID** → **Groups** → delete `staged-rollout-pilot-phs`
3. Delete any throwaway test users created for the pilot group
4. If a Synchronization Rules Editor rule was created on a real Connect server for Step 3, disable or delete it in a test environment only — never leave an experimental sync rule active in a production hybrid tenant

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **OU-based and attribute-based sync filtering** | Prevents stale/irrelevant on-prem objects from ever reaching Entra ID, instead of cleaning them up after the fact |
| **Choosing PHS vs. PTA with documented reasoning** | A defensible, auditable authentication design decision instead of "whatever the wizard defaulted to" |
| **Staged rollout for authentication method changes** | De-risks a tenant-wide auth change by validating against a small pilot group first |
| **Monitoring sync health via the admin center and Graph** | Catches a broken sync cycle in hours instead of discovering it when a new hire can't sign in weeks later |
| **Distinguishing sync-scoping problems from sync-health problems** | Two different failure modes with two different fixes — knowing which one you're looking at saves troubleshooting time |

---

## Common Mistakes to Avoid
- **Installing with Express Settings on a directory that has stale/irrelevant OUs**: syncs the entire forest by default — always evaluate OU/attribute filtering before the first sync, not after cleanup becomes necessary
- **Leaving a staged-rollout pilot group in place indefinitely**: it's meant to validate a cutover, not be a permanent parallel authentication path that nobody remembers exists
- **Assuming "last sync: 30 minutes ago" means every object synced cleanly**: a healthy overall sync timestamp can still hide per-object export errors — check `OnPremisesLastSyncDateTime` on individual users, not just the tenant-wide summary
- **Choosing PTA "for security" without evaluating the availability trade-off**: PTA introduces a dependency on the PTA agent being reachable at sign-in time — a decision that deserves the same trade-off analysis as any other architecture choice, not a reflexive "on-prem check is more secure"

---

## Next Steps
- Pair sync scoping with the administrative unit model from [Lab 1](lab-1-identity-governance-foundations.md) — dynamic AUs can key off the same on-prem attributes (department, OU-derived extension attributes) that drive sync filtering
- Extend [Lab 4](lab-4-identity-protection-graph-automation.md)'s Graph reporting script to flag any synced user with a `OnPremisesLastSyncDateTime` older than one sync cycle, as a scheduled health check
- For the Conditional Access angle on hybrid-joined devices (a different but related hybrid identity concern), see [AZ-500 Lab 1](../AZ-500/lab-1-identity-access-protection.md)'s Next Steps
- Continue to [Lab 6: External Identities](lab-6-external-identities.md) — hybrid sync brings your own workforce into Entra ID; external identities brings in everyone else who needs access
- For the architecture-level decision of *whether and how* to run hybrid identity at all (vs. a cloud-only or fully federated model), see [SC-100 Lab 2: Identity & Access Architecture Design](../SC-100/lab-2-identity-access-architecture-design.md)
