# Lab 5: Hybrid Identity & Azure AD Connect Security

Check box if done: [ ]

## Overview
Most enterprises aren't cloud-only — they have an on-premises Active Directory that still owns the source of truth for identity, and Azure AD Connect (now Microsoft Entra Connect) is the bridge that syncs it to Entra ID. That bridge is one of the highest-value targets in the entire environment: compromise the Connect server and you can potentially manipulate identities across both the on-prem forest and the cloud tenant. This lab covers scoping what actually syncs, choosing between password hash sync and pass-through authentication, adding seamless SSO, staging a cutover safely instead of flipping a switch on the whole company, and treating the Connect server itself as the Tier-0 asset it is.

**Estimated time**: 75–90 minutes
**Cost**: ~$0 (Entra ID Connect and staged rollout are licensing-included features; this lab uses a simulated/conceptual on-prem AD via a lab VM rather than a real corporate forest)

---

## Scenario
Your company is migrating to Entra ID but keeping on-premises AD as the authoritative directory for the next two years — a common "hybrid forever" reality, not a temporary state. The previous admin installed Azure AD Connect with defaults: full-forest sync (including a stale "Contractors-OLD" OU that should never have left on-prem), password hash sync with no fallback plan, and the Connect server has local admin rights held by half the helpdesk team. You're fixing the scoping, evaluating the auth model, and locking down the server before this becomes an incident.

---

## Objectives
- Scope Azure AD Connect sync with organizational unit (OU) filtering so only intended identities reach the cloud
- Understand the security trade-offs between Password Hash Sync (PHS) and Pass-Through Authentication (PTA)
- Enable Seamless SSO and explain what it does and doesn't protect against
- Use staged rollout to test a new authentication method against a pilot group before a tenant-wide cutover
- Harden the Azure AD Connect server itself as a Tier-0 asset

---

## Part 1: Scoping What Actually Syncs

### Step 1: Why Full-Forest Sync Is a Problem
Azure AD Connect's default "sync everything" configuration means every user, group, and contact in on-prem AD — including disabled accounts, service accounts, stale contractor OUs, and anything else nobody's cleaned up in years — gets pushed into Entra ID. Every synced object is an object your cloud identity governance now has to account for, and every stale/forgotten account is potential attack surface (password spray target, dormant admin group member, etc.). Scoping sync to only what's actually needed shrinks that surface deliberately.

### Step 2: Review the Current Sync Scope
On the Azure AD Connect server:

1. Launch **Azure AD Connect** → **Configure** → **Customize synchronization options**
2. Sign in with your Entra ID Global Administrator (or Hybrid Identity Administrator) credentials
3. On the **Domain/OU Filtering** page, review what's currently selected — in the scenario above, this would show the entire domain checked, including `OU=Contractors-OLD`

**Validation checkpoint**: Note every OU currently in scope. Cross-reference against a list of OUs that should actually have cloud presence (typically: active employee accounts, active service accounts that need cloud resources, active security groups used in cloud RBAC/CA policies). Anything else is a candidate for exclusion.

### Step 3: Apply OU Filtering
1. On the **Domain/OU Filtering** page, uncheck OUs that shouldn't sync — in this scenario, `OU=Contractors-OLD` and any other stale/disabled-account containers
2. Continue through the wizard → **Configure** to apply

```powershell
# Alternative: verify current scoping filters via PowerShell on the Connect server
Import-Module ADSync
Get-ADSyncConnectorRunProfileResult -RunHistoryId (Get-ADSyncRunProfileResult | Select-Object -Last 1).RunHistoryId
```

### Step 4: Add Attribute- and Group-Based Filtering for Finer Control
OU filtering is coarse. For finer control — e.g., "sync users, but only if they're a member of `SG-CloudUsers`" — use **group-based filtering** instead, configured the same way but selecting **Sync selected security group** on the **Domain/OU Filtering** page rather than domain-wide OU checkboxes.

**Why this matters more than it looks like it does**: scoping isn't just tidiness. A disabled on-prem account that still syncs to the cloud with a stale password hash is exactly the kind of forgotten identity that shows up in a post-incident review as "how did they get in."

### Step 5: Force a Full Sync and Confirm the Scope Change Took Effect
```powershell
Start-ADSyncSyncCycle -PolicyType Initial
```

**Expected result**: Entra ID (**Entra ID** → **Users**) should no longer show any accounts from the excluded OU after the sync completes. Existing synced objects from excluded OUs are soft-deleted in Entra ID on the next cycle, not left behind.

---

## Part 2: Password Hash Sync vs. Pass-Through Authentication

### Step 6: Understand the Trade-Off
Both methods let users sign in to Entra ID with their on-prem AD password — the difference is where the actual authentication decision happens.

| | Password Hash Sync (PHS) | Pass-Through Authentication (PTA) |
|---|---|---|
| **Where auth happens** | Entra ID validates against a synced hash of the hash | On-prem PTA agent validates against live AD in real time |
| **On-prem dependency** | None — cloud sign-in works even if on-prem AD is down | Requires at least one PTA agent online; AD outage blocks cloud sign-in too |
| **On-prem password policies (lockout, logon hours) enforced in real time** | No — enforced only at the next sync interval | Yes — every sign-in checks live AD |
| **Leaked-credential detection (Entra ID)** | Full support — Entra ID has the hash to check against breach corpora | Full support |
| **Attack surface** | No standing on-prem auth dependency to attack | PTA agents are an additional on-prem component that can be targeted |
| **Disabling a user takes effect** | Up to 30 minutes (next sync cycle), or immediately via manual sync | Immediately — live AD check on every sign-in |

**The real trade-off**: PHS gives you cloud resilience (sign-in still works if on-prem goes down) at the cost of a slight lag in policy/lockout enforcement. PTA gives you real-time on-prem policy enforcement at the cost of a hard dependency on on-prem infrastructure and additional agent surface. Most organizations choose PHS specifically *because* it decouples cloud availability from on-prem availability — it's also required as a fallback even if you use PTA, which the next steps configure.

### Step 7: Confirm PHS Is Enabled as a Fallback (Even If Using PTA)
Even organizations that choose PTA as primary should enable PHS as backup — if every PTA agent goes offline, PHS lets cloud sign-in keep working instead of taking down Entra ID auth tenant-wide.

1. On the Connect server: **Azure AD Connect** → **Configure** → **Customize synchronization options**
2. **Optional features** → confirm **Password hash synchronization** is checked, even if PTA is the primary sign-in method
3. **Configure**

```powershell
# Verify PHS is active from the cloud side
Get-MgUserAuthenticationMethod -UserId <placeholder-user-id> -ErrorAction SilentlyContinue
```

**Validation checkpoint**: **Entra ID** → **Entra Connect** → **Connect Sync** — the sync status page should list **Password Hash Sync** as an enabled feature regardless of which method is primary.

---

## Part 3: Seamless SSO

### Step 8: What Seamless SSO Does and Doesn't Do
Seamless SSO automatically signs users in when they're on a domain-joined corporate device already authenticated to on-prem AD — no password re-prompt for Entra ID-connected apps. It uses a Kerberos ticket against a computer account (`AZUREADSSOACC`) created in on-prem AD, not a certificate or token that leaves the corporate network.

**What it does not do**: it is not a security boundary and it does not replace MFA or Conditional Access. It removes a *prompt*, not a *control* — a Conditional Access policy requiring MFA still fires on top of a seamless SSO sign-in exactly as it would on a manually-typed password.

### Step 9: Enable Seamless SSO
1. **Azure AD Connect** → **Configure** → **Customize synchronization options** → **Optional features** → check **Seamless single sign-on**
2. **Configure** — this creates the `AZUREADSSOACC` computer account in the on-prem domain and configures the Kerberos decryption key exchange with Entra ID

### Step 10: Roll Out via Group Policy (Conceptual — Document the Approach)
Seamless SSO requires the Entra ID URLs to be in the browser's Intranet zone for automatic Kerberos negotiation to trigger. In a real environment this is pushed via GPO:

```
Computer Configuration → Policies → Administrative Templates →
  Windows Components → Internet Explorer → Internet Control Panel →
  Security Page → Site to Zone Assignment List

Add: https://autologon.microsoftazuread-sso.com → Zone 1 (Intranet)
```

**Why this matters for the exam and for real deployments**: Seamless SSO fails silently and falls back to a normal password prompt if the zone assignment isn't pushed — a very common "why isn't SSO working" support ticket that traces back to a missing GPO, not a Connect misconfiguration.

**Validation checkpoint**: From a domain-joined test machine with the GPO applied, browse to a Entra ID-connected app. Expect direct sign-in with no password prompt. From a non-domain-joined machine, expect the normal password prompt — SSO only applies to devices already authenticated to the corporate domain.

---

## Part 4: Staged Rollout — Testing Before Cutover

### Step 11: Why You Never Flip Auth Methods Tenant-Wide
Changing the primary sign-in method (e.g., migrating from PHS to PTA, or from federation to cloud auth) for the entire tenant in one step means any misconfiguration is discovered by every user simultaneously. Staged rollout lets you assign a pilot security group to the new method while everyone else keeps using the current one — same tenant, two auth paths running in parallel, until you're confident.

### Step 12: Create a Pilot Group and Enable Staged Rollout
1. **Entra ID** → **Groups** → create `SG-StagedRollout-Pilot`, add a small number of test users (never yourself alone — you need to confirm it works for someone other than the admin doing the testing)
2. **Entra ID** → **Entra Connect** → **Staged rollout**
3. Enable the feature you're testing — e.g., **Pass-through authentication** — toggle to **On**
4. **Manage groups** → add `SG-StagedRollout-Pilot`

### Step 13: Validate the Pilot Before Wider Rollout
1. Sign in as a pilot group member → confirm authentication behaves as expected under the new method (PTA agent logs on the Connect server should show the authentication request if testing PTA)
2. Sign in as a non-pilot user → confirm they're still authenticating via the original method (unaffected)

**Validation checkpoint**: On the Connect server (if testing PTA), `Event Viewer` → **Applications and Services Logs** → **Microsoft** → **AAD Connect Authentication Agent** should show successful authentication events correlated to the pilot user's sign-in attempt.

### Step 14: Expand or Roll Back
- **If validation passes**: add more users/groups to the staged rollout group incrementally, monitoring at each step, before finally changing the tenant-wide default in **Entra Connect** → **User sign-in**
- **If validation fails**: remove the group from staged rollout — affected users immediately revert to the original method with no tenant-wide impact, because staged rollout never touched the default

---

## Part 5: Hardening the Azure AD Connect Server (Tier-0 Asset)

### Step 15: Why This Server Is Tier-0
The Connect server holds credentials capable of writing to on-prem AD (via the AD DS Connector account) and reading/writing to Entra ID (via the Entra ID Connector account). An attacker with administrative access to this server can, in the worst case, use it as a pivot to compromise the entire on-prem AD forest *and* manipulate cloud identity — this is functionally equivalent to compromising a domain controller. Microsoft's Enterprise Access Model classifies it as Tier-0 for exactly this reason.

### Step 16: Restrict Local Administrator Access
1. On the Connect server, open **Local Users and Groups** → **Administrators**
2. Remove any account that isn't a dedicated, documented Tier-0 admin — helpdesk, general IT, and application support accounts have no legitimate reason to hold local admin here
3. Document the remaining members and the justification for each

```powershell
# Enumerate current local admins for review
Get-LocalGroupMember -Group "Administrators"
```

### Step 17: Apply Tier-0 Isolation Principles
- The Connect server should only be administered from a Privileged Access Workstation (PAW) or equivalent hardened jump host — never from a general-purpose helpdesk machine
- No general-purpose software (browsers used for general browsing, email clients, etc.) should be installed on this server — it should do one job
- Enable Windows Defender Application Control or, at minimum, keep the server on the same patching SLA as domain controllers, not the standard server patch cycle

### Step 18: Confirm the Connector Accounts Have Least Necessary AD Permissions
The on-prem **AD DS Connector account** created during Connect installation should have exactly the delegated permissions Connect needs (replicate directory changes, write to synced attributes on scoped OUs) — not Domain Admin. If a previous install granted it broader rights than needed, review and reduce.

```powershell
# On a domain controller: inspect the ACL delegated to the connector account on the synced OU
Get-Acl "AD:\OU=Employees,DC=lab,DC=local" | Select-Object -ExpandProperty Access |
  Where-Object { $_.IdentityReference -like "*MSOL_*" -or $_.IdentityReference -like "*ADSync*" }
```

**Validation checkpoint**: Confirm the connector account is **not** a member of Domain Admins, Enterprise Admins, or any other highly privileged on-prem group. It should only appear in the delegated ACLs on the specific OUs it needs to sync.

### Step 19: Enable Auditing on the Connect Server
Ensure the Connect server's security event log is forwarded to your SIEM (the same Log Analytics workspace / Sentinel instance built in Lab 4) — sign-in and process-creation events on this box are high-value detections. A logon to the Connect server by an account outside the documented Tier-0 admin list is worth an immediate alert.

---

## Part 6: Cleanup

1. If this was a disposable lab environment, remove the pilot group from staged rollout and disable the staged rollout feature: **Entra Connect** → **Staged rollout** → toggle features **Off**
2. Delete the pilot group: **Entra ID** → **Groups** → `SG-StagedRollout-Pilot` → **Delete**
3. If you stood up a lab AD Connect server solely for this exercise, deprovision the VM (`az vm delete` or equivalent) — do not leave a Tier-0-capable server running unmonitored
4. Revert any OU filtering changes if you were working against a shared lab directory others depend on

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **OU/group-based sync scoping** | Prevents stale or unintended on-prem accounts from expanding cloud attack surface |
| **PHS vs. PTA trade-off analysis** | Demonstrates judgment about availability vs. real-time policy enforcement, a core AZ-500 theme |
| **PHS enabled as fallback under PTA** | Prevents a tenant-wide cloud sign-in outage if all PTA agents go down |
| **Seamless SSO with correct scope understanding** | Improves UX without mistaking it for a security control |
| **Staged rollout for auth method changes** | Avoids tenant-wide blast radius from an untested authentication change |
| **Tier-0 hardening of the Connect server** | Recognizes and protects the single highest-value pivot point in a hybrid identity environment |

---

## Common Mistakes to Avoid
- **Syncing the entire forest by default**: every stale OU and disabled account becomes cloud attack surface for no operational benefit
- **Treating Seamless SSO as a security control**: it removes a password prompt, not a policy check — Conditional Access and MFA still need to be layered on top
- **Choosing PTA without also enabling PHS as fallback**: an all-PTA-agents-down scenario should degrade, not fully block, cloud sign-in
- **Changing the tenant-wide auth method without staged rollout**: any misconfiguration is discovered by the entire user base simultaneously instead of a small pilot group
- **Granting the AD DS Connector account Domain Admin "to make sync easier"**: it only needs delegated rights on the OUs it syncs — broad rights turn a sync account into a domain-compromise vector
- **Leaving the Connect server administrable by general IT/helpdesk accounts**: this server is functionally equivalent to a domain controller in terms of blast radius and should be treated with the same Tier-0 discipline

---

## Next Steps
- Layer Conditional Access device-compliance requirements (from [Lab 1](lab-1-identity-access-protection.md)) on top of Seamless SSO so a fast sign-in still requires a managed, compliant device
- Extend the Sentinel Azure Activity connector (from [Lab 4](lab-4-security-ops-automation.md)) with a Windows Security Events connector on the Connect server itself, and write an analytics rule alerting on any interactive logon by an account outside the documented Tier-0 admin list
- Review Microsoft's Enterprise Access Model in full and map every other Tier-0 asset in your environment (domain controllers, PKI servers, federation servers) the same way this lab treated the Connect server
- Continue to [Lab 6: Advanced Network Security](lab-6-advanced-network-security.md) to extend perimeter controls beyond the NSG/Bastion baseline from Lab 2
