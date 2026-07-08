# Lab 4: Identity Protection & Graph Automation

Check box if done: [ ]

## Overview
Everything in the first three labs was one click, one policy, one review at a time — fine for a handful of objects, unworkable at real organizational scale. This lab closes the track with two things: a more advanced Conditional Access scenario layering device and location signals on top of the basic MFA policy from AZ-500, and a Microsoft Graph PowerShell script that reports on identity data across the whole tenant in seconds — the kind of automation that separates "I clicked through the portal" from "I can operate at scale" in an interview.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (requires Entra ID P2 trial for the risk-based Conditional Access scenario)

---

## Scenario
Two things landed on your desk this week. First: leadership wants admin actions blocked from outside the country unless the device is compliant — a more nuanced policy than the blanket MFA requirement you built in AZ-500. Second: the CISO wants a weekly report of every user with an admin role, their last sign-in date, and whether they have MFA registered — and doing that by hand in the portal takes an afternoon. You're building both.

---

## Objectives
- Build a layered Conditional Access policy combining location, device compliance, and risk signals
- Understand named locations and how they refine Conditional Access conditions
- Connect to Microsoft Graph via PowerShell and authenticate with least-privilege scopes
- Write a script that reports on privileged users, their MFA registration status, and last sign-in
- Understand where Graph automation fits relative to portal work and PIM-gated access

---

## Part 1: Layered Conditional Access

### Step 1: Define a Named Location
Named locations let a Conditional Access policy reason about "inside vs. outside a trusted network/country" instead of just "any location."

1. **Entra ID** → **Security** → **Conditional Access** → **Named locations** → **+ IP ranges location** (or **+ Countries location**)
2. For a countries-based location: **Name**: `Approved Countries`, select the country/countries your organization actually operates from
3. **Create**

### Step 2: Build a Layered Policy for Admin Roles
This policy combines three conditions the AZ-500 Lab 1 baseline policy didn't: it targets admin roles specifically (not all users), requires the request to come from an approved location, and requires a compliant device — not just MFA.

1. **Conditional Access** → **+ New policy**
2. **Name**: `Admin Access — Location + Compliant Device`
3. **Assignments → Users**: **Include** → **Directory roles** → select a few admin roles (`Global Administrator`, `User Administrator`, `Helpdesk Administrator`) — **exclude** your break-glass accounts
4. **Conditions → Locations**: **Include**: `Any location` → **Exclude**: `Approved Countries`
5. **Grant**: **Require device to be marked as compliant** AND **Require multi-factor authentication** (set to "Require all the selected controls," not "require one of")
6. **Enable policy**: `Report-only` first

### Step 3: Test and Promote
1. **Conditional Access** → **Insights and reporting** → review report-only results for a few sign-ins
2. Once confirmed, edit the policy → **Enable policy**: `On`

**What this demonstrates over the AZ-500 baseline**: Conditional Access conditions compose. A single policy can require *all* of location + device compliance + MFA simultaneously, and different policies can target different populations (all users vs. admin roles only) with different bars — this is the layered defense-in-depth model the exam expects you to design, not just enable one setting.

---

## Part 2: Connect to Microsoft Graph via PowerShell

### Step 4: Install and Connect
```powershell
Install-Module Microsoft.Graph -Scope CurrentUser

# Connect with the minimum scopes needed for this script — not admin consent for everything Graph offers
Connect-MgGraph -Scopes "User.Read.All", "Directory.Read.All", "UserAuthenticationMethod.Read.All", "AuditLog.Read.All"
```

**Why scope the connection explicitly?** `Connect-MgGraph` with no `-Scopes` argument still asks for consent, but being explicit about exactly the scopes a script needs is the same least-privilege discipline as the app permission review in [Lab 2](lab-2-app-integration-sso.md) — a script requesting `Directory.ReadWrite.All` for a read-only report is over-privileged.

### Step 5: Verify the Connection
```powershell
Get-MgContext | Select-Object Account, Scopes
```
Confirm the account and scope list match what you requested.

---

## Part 3: Build the Privileged User Report

### Step 6: Pull Every User with an Admin Directory Role
```powershell
# Get all directory role assignments (built-in + custom admin roles)
$adminRoles = Get-MgDirectoryRole | Where-Object { $_.DisplayName -match "Administrator" }

$privilegedUsers = foreach ($role in $adminRoles) {
    $members = Get-MgDirectoryRoleMember -DirectoryRoleId $role.Id
    foreach ($member in $members) {
        $user = Get-MgUser -UserId $member.Id -Property DisplayName, UserPrincipalName, SignInActivity -ErrorAction SilentlyContinue
        if ($user) {
            [PSCustomObject]@{
                DisplayName       = $user.DisplayName
                UPN               = $user.UserPrincipalName
                Role              = $role.DisplayName
                LastSignIn        = $user.SignInActivity.LastSignInDateTime
            }
        }
    }
}

$privilegedUsers | Format-Table -AutoSize
```

> **Note**: `SignInActivity` requires an Entra ID P1/P2 license on the querying app/user context. If it returns null, your tenant's license tier doesn't expose sign-in activity via Graph — the rest of the script still works.

### Step 7: Add MFA Registration Status
```powershell
$report = foreach ($user in $privilegedUsers) {
    $methods = Get-MgUserAuthenticationMethod -UserId $user.UPN -ErrorAction SilentlyContinue
    $hasMfa = ($methods | Where-Object { $_.AdditionalProperties.'@odata.type' -match "phoneAuthenticationMethod|microsoftAuthenticatorAuthenticationMethod|fido2AuthenticationMethod" }).Count -gt 0

    [PSCustomObject]@{
        DisplayName = $user.DisplayName
        UPN         = $user.UPN
        Role        = $user.Role
        LastSignIn  = $user.LastSignIn
        MfaRegistered = $hasMfa
    }
}

$report | Sort-Object Role | Format-Table -AutoSize
```

### Step 8: Flag the Actual Risk — Privileged Users Without MFA
This is the line that turns raw data into an actionable finding — the CISO doesn't want the whole table, they want the exceptions:

```powershell
$noMfa = $report | Where-Object { -not $_.MfaRegistered }

if ($noMfa) {
    Write-Warning "$($noMfa.Count) privileged user(s) without MFA registered:"
    $noMfa | Format-Table DisplayName, UPN, Role -AutoSize
} else {
    Write-Output "All privileged users have MFA registered."
}
```

### Step 9: Export for the Weekly Report
```powershell
$report | Export-Csv -Path "./privileged-user-report-$(Get-Date -Format 'yyyy-MM-dd').csv" -NoTypeInformation
```

**Validation checkpoint**: Open the exported CSV and confirm it contains one row per admin-role assignment, with realistic-looking `DisplayName`, `UPN`, `Role`, and `MfaRegistered` values matching what you'd see clicking through **Roles and administrators** manually — the script output should match the portal's ground truth.

---

## Part 4: Turn It Into a Reusable, Read-Only Tool

### Step 10: Wrap It as a Function
```powershell
function Get-PrivilegedUserMfaReport {
    [CmdletBinding()]
    param(
        [string]$OutputPath = "./privileged-user-report-$(Get-Date -Format 'yyyy-MM-dd').csv"
    )

    $adminRoles = Get-MgDirectoryRole | Where-Object { $_.DisplayName -match "Administrator" }

    $report = foreach ($role in $adminRoles) {
        $members = Get-MgDirectoryRoleMember -DirectoryRoleId $role.Id
        foreach ($member in $members) {
            $user = Get-MgUser -UserId $member.Id -Property DisplayName, UserPrincipalName -ErrorAction SilentlyContinue
            if (-not $user) { continue }

            $methods = Get-MgUserAuthenticationMethod -UserId $user.Id -ErrorAction SilentlyContinue
            $hasMfa = ($methods | Where-Object { $_.AdditionalProperties.'@odata.type' -match "phoneAuthenticationMethod|microsoftAuthenticatorAuthenticationMethod|fido2AuthenticationMethod" }).Count -gt 0

            [PSCustomObject]@{
                DisplayName   = $user.DisplayName
                UPN           = $user.UserPrincipalName
                Role          = $role.DisplayName
                MfaRegistered = $hasMfa
            }
        }
    }

    $report | Export-Csv -Path $OutputPath -NoTypeInformation
    Write-Output "Report written to $OutputPath ($($report.Count) privileged role assignments, $(($report | Where-Object {-not $_.MfaRegistered}).Count) without MFA)"
    return $report
}
```

This is a **read-only** function by design — it queries and reports, it doesn't modify anything. A script that both detects and auto-remediates (e.g., force-registering MFA) is a much bigger blast radius if it has a bug, and isn't something to hand to a scheduled task without a lot more testing.

### Step 11: Disconnect
```powershell
Disconnect-MgGraph
```

---

## Part 5: Cleanup

1. **Conditional Access** → disable or delete `Admin Access — Location + Compliant Device` and the `Approved Countries` named location if this is a disposable lab tenant
2. Delete the exported CSV report if it contains real tenant data you don't want lingering locally
3. No Azure resources were created in this lab — nothing to `az group delete`

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Layered Conditional Access (location + device + MFA, targeted at admin roles)** | Defense-in-depth — different populations get different bars, and conditions compose |
| **Named locations** | Turns "trusted network" from a vague concept into an explicit, referenceable Conditional Access condition |
| **`Connect-MgGraph` with explicit least-privilege scopes** | Same least-privilege discipline applied to automation, not just human role assignments |
| **Querying directory roles, users, and auth methods via Graph** | The actual mechanism behind every "who has access to X" report an identity admin gets asked for |
| **Read-only reporting script, not auto-remediation** | Appropriately scoped blast radius for a first automation pass — detect and report before you automate a fix |

---

## Common Mistakes to Avoid
- **Combining Conditional Access grant controls with "require one of" instead of "require all"**: silently weakens a policy meant to be a hard combination of conditions — always confirm which mode you selected
- **Connecting to Graph with broad or default scopes**: request exactly the scopes the script needs, the same way you'd scope an app registration's permissions
- **Treating `SignInActivity` as available in every tenant**: it requires a P1/P2 license context — a null result usually means a licensing gap, not that the user never signed in
- **Writing an automation script that both detects and fixes issues on its first version**: start read-only, prove the detection logic is accurate against the portal's ground truth, and only add remediation once you trust the report

---

## Next Steps
- Schedule the `Get-PrivilegedUserMfaReport` function as a weekly task (Azure Automation runbook or a scheduled GitHub Action) and pipe results to email or Teams
- Extend the script to cross-reference PIM-eligible vs. permanently-active role assignments (flag any admin role assignment that isn't PIM-eligible, tying back to [AZ-500 Lab 1](../AZ-500/lab-1-identity-access-protection.md))
- This closes out the SC-300 track — for the equivalent identity-and-access-at-scale muscle in AWS, see [AWS SCS-C02 Lab 1: IAM Deep Dive](../../Aws/SCS-C02/aws-sec-lab-1-iam-deep-dive.md)
