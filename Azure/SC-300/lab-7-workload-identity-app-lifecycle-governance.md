# Lab 7: Workload Identity & Application Lifecycle Governance

Check box if done: [ ]

## Overview
Every app registration created in Lab 2 was one deliberate action with a clear owner — you. In a real tenant, app registrations accumulate for years: a developer registers something for a proof of concept, the project ships or dies, and the registration just... stays, usually with a client secret quietly approaching (or long past) its expiry, and usually with no owner anyone can name. This lab treats applications and their non-human identities as things that need the same lifecycle discipline as user accounts — registration through retirement — plus the specific governance problem of workload identities: pipelines and services that authenticate as an app registration, and the credential hygiene and consent controls that keep that population from becoming unmanaged debt.

**Estimated time**: 60–75 minutes
**Cost**: ~$0

---

## Scenario
A security review of your tenant surfaced three findings. First: 40+ app registrations with no assigned owner, several with client secrets that expired over a year ago and were apparently never noticed because nothing broke (the app in question was already dead). Second: a GitHub Actions deployment pipeline is still authenticating with a stored client secret sitting in a repo secret, despite workload identity federation being available and secret-free. Third: user consent is still open tenant-wide, meaning any employee can grant *any* app they click "Accept" on access to their mailbox or files, with no admin ever reviewing what got consented to. You're fixing the process gap behind all three, not just cleaning up the current mess.

---

## Objectives
- Understand the app registration lifecycle and why ownership assignment is a governance control, not paperwork
- Configure workload identity federation so a CI/CD pipeline authenticates with no stored secret
- Use Microsoft Graph PowerShell to audit and govern service principal credentials at scale
- Configure admin consent workflow and restrict user consent for high-risk permissions
- Run an access review scoped to a specific enterprise application, distinct from the role/group reviews in Lab 3

---

## Part 1: The App Registration Lifecycle

### Step 1: Why Registrations Become Unmanaged Debt
An app registration has no built-in expiry and no HR-style offboarding trigger the way a user account does. Nothing forces cleanup when a project ends — the registration just sits in **App registrations**, indefinitely, until someone notices. The lifecycle an identity admin actually owns has four stages: **registration** (Lab 2's Part 1), **credential issuance and rotation** (Part 3 of this lab), **publishing/consent** (Part 4), and **retirement** — and the single biggest gap in most tenants is that nobody owns the retirement stage, or even knows which registrations are still in active use.

### Step 2: Assign Owners to Every App Registration
Ownership is the field that makes the rest of this lab possible — an access review or a stale-credential report is only actionable if there's a name attached to fix it.

1. **Entra ID** → **App registrations** → open (or re-create) `Internal LOB App` from Lab 2, or use any existing registration in your tenant
2. **Owners** → **+ Add owners** → assign yourself or a test user
3. Repeat for any other registrations you control — in a real tenant, this is the first remediation task from a "we found 40 unowned apps" finding, usually run as a bulk Graph script against the whole registration list rather than one at a time in the portal

### Step 3: Inventory Ownerless Registrations via Graph
```powershell
Connect-MgGraph -Scopes "Application.Read.All"

Get-MgApplication -All -Property Id, DisplayName, CreatedDateTime |
    ForEach-Object {
        $owners = Get-MgApplicationOwner -ApplicationId $_.Id
        if ($owners.Count -eq 0) {
            [PSCustomObject]@{
                DisplayName = $_.DisplayName
                AppId       = $_.Id
                CreatedDate = $_.CreatedDateTime
            }
        }
    } | Sort-Object CreatedDate | Format-Table -AutoSize
```

**Expected result**: any registration with zero owners appears in the output. Sorted oldest-first, this is exactly the "who registered this and do they still need it" triage list a real cleanup project starts from.

**Validation checkpoint**: your test registration from Step 2 does **not** appear in this list — confirming the owner assignment actually took.

---

## Part 2: Workload Identity Federation for CI/CD

### Step 4: The Governance Version of a Problem You May Have Already Solved
If you've worked through [Azure General Lab 3: CI/CD with OIDC Federated Auth](../General/lab-3-cicd-oidc.md), you've already built this from the pipeline-engineer side using the Azure CLI. This section covers the same federated-credential mechanism from the identity-administrator side — reviewing and approving the trust relationship as something you're accountable for governing, not just something a pipeline needs to work. The subject claim scoping is the actual security control either way; here the emphasis is on *administering* that trust deliberately rather than rubber-stamping whatever a developer requests.

### Step 5: Register the Application (No Secret)
1. **Entra ID** → **App registrations** → **+ New registration**
2. **Name**: `gh-actions-deploy-pipeline`, **Supported account types**: `Single tenant`
3. **Register** — do **not** proceed to **Certificates & secrets** to create a client secret; that step is deliberately skipped for the rest of this lab

### Step 6: Add the Federated Credential
1. Open the new registration → **Certificates & secrets** → **Federated credentials** tab → **+ Add credential**
2. **Federated credential scenario**: `GitHub Actions deploying Azure resources`
3. **Organization**: `<your-github-org>`, **Repository**: `<your-repo>`, **Entity type**: `Branch`, **Branch name**: `main`
4. **Name**: `gh-main-branch-deploy`
5. **Add**

**Expected result**: the **Federated credentials** tab now lists one trust entry, and **Certificates & secrets** → **Client secrets** remains empty — there is no rotating credential to leak, expire, or forget, because none was ever created.

### Step 7: Review the Trust Boundary Like an Approval, Not a Formality
Open the federated credential back up and read the generated **Subject identifier** (`repo:<org>/<repo>:ref:refs/heads/main`). This string is the entire security boundary: a token from a fork, a feature branch, or any other repository will not match it and authentication fails before any Azure call happens. As the identity admin approving this configuration, the review question is the same one you'd ask of any access grant — is the subject scoped to exactly the branch/repo that should have this trust, or was it left broad "to be safe"?

**Validation checkpoint**: **Certificates & secrets** → **Federated credentials** shows exactly one entry with a subject identifier scoped to a specific repo and branch — not a wildcard, and not a broader `pull_request`-and-`main` combined trust when the pipeline only needs one of them.

---

## Part 3: Service Principal Credential Governance at Scale

### Step 8: Why Proactive Beats Reactive Here
The security review's second finding — expired secrets on dead apps nobody noticed — happened because credential hygiene was reactive: nobody looked until something broke, and a dead app breaking is silent. The fix is a script that finds every credential approaching or past expiry *before* it becomes an incident, run on a schedule, not after an audit flags it.

### Step 9: Query All App Registration Credentials
```powershell
Connect-MgGraph -Scopes "Application.Read.All"

$credentialReport = foreach ($app in Get-MgApplication -All -Property Id, DisplayName, PasswordCredentials, KeyCredentials) {
    foreach ($cred in @($app.PasswordCredentials) + @($app.KeyCredentials)) {
        [PSCustomObject]@{
            AppDisplayName  = $app.DisplayName
            CredentialType  = if ($cred.CustomKeyIdentifier) { "Certificate" } else { "ClientSecret" }
            KeyId           = $cred.KeyId
            EndDate         = $cred.EndDateTime
            DaysUntilExpiry = [math]::Round((New-TimeSpan -Start (Get-Date) -End $cred.EndDateTime).TotalDays)
        }
    }
}

$credentialReport | Sort-Object DaysUntilExpiry | Format-Table -AutoSize
```

### Step 10: Flag What Actually Needs Action
```powershell
$expired       = $credentialReport | Where-Object { $_.DaysUntilExpiry -lt 0 }
$expiringSoon  = $credentialReport | Where-Object { $_.DaysUntilExpiry -ge 0 -and $_.DaysUntilExpiry -le 30 }

Write-Output "Expired: $($expired.Count) | Expiring within 30 days: $($expiringSoon.Count)"
$expired | Format-Table AppDisplayName, CredentialType, EndDate -AutoSize
```

An **expired** credential is not itself a live risk — it can no longer authenticate — but its presence signals nobody has cleaned this app up, exactly the "dead app nobody noticed" pattern from the scenario. A credential **expiring within 30 days** on an app that's still live is the one that turns into an outage if it isn't rotated in time.

### Step 11: Remove a Confirmed-Dead Credential
Once ownership (Part 1) confirms an app is genuinely retired, remove its stale credentials rather than leaving them as directory clutter:

```powershell
# Requires Application.ReadWrite.All — a materially higher-privilege scope than the read-only audit above
Connect-MgGraph -Scopes "Application.ReadWrite.All"

Remove-MgApplicationPassword -ApplicationId "<app-object-id>" -KeyId "<key-id-from-report>"
```

**Real-world practice**: never run the removal step against the full expired-credential list unattended — confirm with the app's owner (from Part 1) that it's actually retired first. A credential that looks expired-and-abandoned but belongs to a low-traffic quarterly batch job is a very different remediation than a genuinely dead app.

**Validation checkpoint**: re-run the Step 9 query and confirm the removed credential no longer appears in `$credentialReport` for that application.

---

## Part 4: App Consent Governance

### Step 12: Close the Open Consent Gap and Give It a Release Valve
The third audit finding — any employee can consent to any app's permission request — is a default, not a misconfiguration unique to your tenant. Left open, a convincing phishing-style "sign in with Microsoft" consent screen from a malicious app is a real path to mailbox or file access with no admin ever in the loop. Blocking it only works operationally if a legitimate request still has a path to approval, so both settings get configured together.

1. **Entra ID** → **Enterprise applications** → **Consent and permissions** → **User consent settings** → change from `Allow user consent for apps` to `Allow user consent for apps from verified publishers, for selected permissions` — known-low-risk permissions from verified publishers still work without friction; anything higher-risk routes to admin review instead of a silent user click
2. **Admin consent settings** → **Users can request admin consent to apps they are unable to consent to**: `Yes` → **Who can review admin consent requests**: assign yourself or a designated reviewer group → **Selected users will receive email notifications for requests**: `Yes`
3. **Save** both blades

**Expected result**: a user who hits the consent block now sees a **Request approval** option instead of a dead end, and the designated reviewer gets a request to approve or deny — the same request → approve pattern as Lab 3's access packages, applied to app consent instead of group/resource access.

### Step 13: Review What's Already Been Consented To
Closing the front door doesn't retroactively review what already got through it. Audit existing grants before assuming the tenant is clean:

```powershell
Connect-MgGraph -Scopes "Directory.Read.All"

Get-MgOauth2PermissionGrant -All |
    Where-Object { $_.ConsentType -eq "AllPrincipals" } |
    ForEach-Object {
        $sp = Get-MgServicePrincipal -ServicePrincipalId $_.ClientId -Property DisplayName
        [PSCustomObject]@{
            AppName     = $sp.DisplayName
            Scopes      = $_.Scope
            ConsentType = $_.ConsentType
        }
    } | Format-Table -AutoSize
```

`ConsentType -eq "AllPrincipals"` isolates **tenant-wide** admin consent grants — the ones where a single approval (or, before Step 12, a single accidental user click on a since-tightened setting) opened access for every user in the tenant. Cross-reference the `Scopes` column against what each app plausibly needs; a scheduling app requesting `Mail.ReadWrite` for the whole org is worth a follow-up conversation with whoever approved it.

**Validation checkpoint**: identify at least one existing grant in the output and confirm whether its scope is proportionate to what the app is actually for — the same over-broad-permission scrutiny from [Lab 2](lab-2-app-integration-sso.md)'s consent section, now applied tenant-wide instead of to one app you registered yourself.

---

## Part 5: Application Access Reviews

### Step 14: Why This Is a Different Review Than Lab 3's
Lab 3 built access reviews scoped to **group membership** and PIM-eligible roles. This section reviews access to a specific **application** directly — who currently has an app role assignment on an enterprise app, independent of which group got them there. The distinction matters because an app can accumulate direct user assignments, group-based assignments, and legacy grants from before entitlement management existed, and a group-scoped review alone won't catch someone assigned directly to the app.

### Step 15: Create an Access Review Scoped to an Application
1. **Identity Governance** → **Access reviews** → **+ New access review**
2. **Select what to review**: `Applications`
3. **Application**: `Internal LOB App` (from Lab 2) or `gh-actions-deploy-pipeline`
4. **Scope**: `Guest and member users`
5. **Reviewers**: `Selected users` → the application owner assigned in Part 1
6. **If reviewers don't respond**: `Remove access`
7. **Create**

**Expected result**: this review evaluates every current user/group assignment on the app's **Users and groups** blade directly — the same surface configured in [Lab 2 Step 8](lab-2-app-integration-sso.md), now under periodic re-certification instead of a one-time assignment nobody revisits.

### Step 16: Confirm the Distinction Holds
Compare this review's scope against Lab 3's `finance-reporting-access` group review: the group review only ever sees group membership, so a user assigned directly to an app's `Approver` role — bypassing the group entirely — would never surface there. The application-scoped review here is the only mechanism that catches that direct-assignment case.

**Validation checkpoint**: **Identity Governance** → **Access reviews** now lists two reviews with different `Select what to review` types (`Teams + Groups` from Lab 3, `Applications` from this lab) — confirming they cover genuinely different access surfaces, not the same population twice.

---

## Part 6: Cleanup

1. **Entra ID** → **App registrations** → delete `gh-actions-deploy-pipeline` (federated credential is removed automatically with the registration)
2. **Identity Governance** → **Access reviews** → delete the application-scoped review from Part 5
3. **Enterprise applications** → **Consent and permissions** → revert **User consent settings** and **Admin consent settings** if this is a shared/production tenant and the restricted values weren't already your intended posture
4. Remove any test client secrets or credentials created solely for the Step 11 removal walkthrough

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **App registration ownership assignment and auditing** | Turns "who owns this app" from an unanswerable question into a queryable Graph report |
| **Workload identity federation (no stored secret)** | Removes an entire class of leakable, rotation-dependent credential from CI/CD authentication |
| **Graph-based credential expiry reporting and removal** | Proactive credential hygiene — finding stale secrets before they cause an outage or become forgotten attack surface |
| **Restricting user consent, with an admin consent request workflow as the release valve** | Closes the "any employee can grant any app mailbox access" gap without just breaking legitimate app requests |
| **Auditing existing tenant-wide consent grants** | Retroactively catches over-broad permission grants that predate the tightened consent policy |
| **Application-scoped access reviews** | Catches direct app-role assignments that a group-membership-only review would miss entirely |

---

## Common Mistakes to Avoid
- **Treating "no owner listed" as harmless**: an ownerless app registration is an app nobody is accountable for cleaning up, rotating credentials on, or explaining to an auditor — assign owners at registration time, not during a cleanup project years later
- **Creating a client secret for a CI/CD pipeline "just to get it working," meaning to switch to federation later**: that migration rarely happens once the secret-based version works — build the federated credential first
- **Bulk-deleting every expired credential without confirming the app is actually retired**: an expired-looking credential on a legitimately active app (rotated but not cleaned up, or a low-frequency batch job) needs owner confirmation, not an automated sweep
- **Setting user consent to fully blocked instead of the verified-publisher/selected-permissions middle tier**: fully blocking consent with no admin consent request workflow just converts every legitimate app request into an unresolvable support ticket
- **Reviewing group-based access and assuming application access is covered**: direct app-role assignments bypass group membership entirely — an application-scoped review is the only thing that catches them

---

## Next Steps
- Schedule the Part 3 credential-expiry report as a weekly automation (the same pattern as [Lab 4](lab-4-identity-protection-graph-automation.md)'s `Get-PrivilegedUserMfaReport`), and alert on anything crossing the 30-day threshold instead of discovering it manually
- Extend the Part 4 consent audit to cross-reference `AllPrincipals` grants against the app ownership data from Part 1 — flag any tenant-wide grant on an app with no assigned owner as a priority review item
- For the pipeline-engineer side of workload identity federation — building the actual GitHub Actions workflow this lab's federated credential trusts — see [Azure General Lab 3: CI/CD with OIDC Federated Auth](../General/lab-3-cicd-oidc.md)
- Continue to [Lab 8: Lifecycle Workflows & Entitlement at Scale](lab-8-capstone-lifecycle-workflows-entitlement-scale.md), the capstone lab, which automates the human side of the lifecycle problem this lab solved for applications
