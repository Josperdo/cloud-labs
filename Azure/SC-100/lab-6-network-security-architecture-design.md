# Lab 6: Network Security Architecture Design

Check box if done: [ ]

## Overview
Every incident and design decision so far traces back to one root cause: a VPN that granted flat, network-wide trust the moment a credential authenticated. Lab 2 fixed the identity side of that with layered Conditional Access. This lab fixes the network side — designing between a modernized traditional hub-spoke firewall perimeter and a Security Service Edge (SSE) model that makes access decisions per-application and identity-aware instead of per-network. AZ-305 Lab 5 already builds the traditional hub-spoke pattern in full; this lab's job is the *decision* between that pattern and its Zero Trust-native alternative, then a representative implementation of the recommended one.

**Estimated time**: 60–90 minutes
**Cost**: ~$0–$2 (Entra Internet Access/Private Access — branded together as Global Secure Access — bills per-user for the Microsoft 365/Internet Access traffic profiles beyond a trial allotment; this lab's steps stay within trial/free evaluation scope and avoid deploying Azure Firewall, which is the cost driver in the hub-spoke alternative)

---

## Scenario
The org's remaining VPN infrastructure — the exact class of asset involved in the Lab 1 incident — still grants any authenticated user network-layer access to everything behind it once connected. The platform team's default instinct is "replace it with a proper hub-spoke and Azure Firewall," which AZ-305 Lab 5 already shows how to build well. But the org now also has a large remote/hybrid workforce (post-acquisition) and contractors (Lab 2) who need access to *specific applications*, not the network segment those applications happen to live in. You're deciding whether the fix is a better-built version of the same network-perimeter model, or a fundamentally different, identity-aware access model — before committing budget to either.

---

## Objectives
- Decide between Security Service Edge (Entra Internet Access/Private Access) and a modernized hub-spoke firewall perimeter for remote and internet access
- Understand where the two models are complementary rather than mutually exclusive
- Enable Global Secure Access and configure a Private Access application to replace one class of VPN-granted access
- Configure a traffic forwarding profile and validate identity-aware per-app access
- Prove per-connection enforcement with traffic log evidence, contrasting an allowed and a denied access attempt
- Compare the resulting access model against the flat-VPN state directly

---

## Part 1: Design Decision — Security Service Edge vs. Traditional Hub-Spoke Firewall Perimeter

### Decision: Remote/Internet Access Model

| Factor | Traditional Hub-Spoke + Firewall/VPN (see [AZ-305 Lab 5](../AZ-305/lab-5-network-infrastructure-design.md) for the full build) | Security Service Edge — Entra Internet Access + Private Access (Global Secure Access) |
|---|---|---|
| **Access granularity** | Network-layer — once a VPN tunnel is established, the user has IP reachability to whatever the firewall/NSG rules allow, typically an entire subnet or VNet | Application-layer and identity-aware — access is granted per application via Conditional Access, evaluated on every connection, not just at tunnel establishment |
| **Zero Trust alignment ("assume breach," "least privilege")** | Weaker by default — a compromised VPN credential (Lab 1's exact scenario) grants network reachability, and lateral movement depends entirely on internal segmentation being airtight | Stronger by design — there is no network tunnel to compromise into; a compromised credential is still gated by Conditional Access (device compliance, MFA, risk) on every app access, and lateral network movement isn't possible because there's no shared network segment to move across |
| **Scalability for remote/hybrid workforce** | Requires VPN gateway capacity planning, often backhauls traffic to a central hub even for destinations near the user, adding latency | Cloud-delivered edge (Microsoft's global network) — traffic is inspected close to the user, no backhaul required, scales with Entra ID's existing global footprint |
| **Latency for geographically distributed users** | Backhaul to hub region adds round-trip latency, worse the farther the user is from the hub | Generally lower — traffic exits near the user through Microsoft's edge network rather than tunneling to one region |
| **Cost model** | Firewall bills hourly regardless of traffic (~$0.90+/hr Standard SKU) plus VPN gateway cost | Per-user licensing for Internet Access/Private Access traffic profiles — no always-on infrastructure meter running whether or not anyone is connected |
| **Maturity / ecosystem fit** | Mature, well-understood, needed regardless for east-west traffic between Azure workloads (spoke-to-spoke, spoke-to-PaaS) | Newer capability, purpose-built for north-south user-to-app access; does not replace the need for a hub-spoke perimeter around Azure-to-Azure and Azure-to-internet workload traffic |
| **Where each wins** | East-west workload traffic, Azure Firewall FQDN filtering for outbound resource traffic, any traffic that isn't "a user reaching an app" | North-south user access — exactly the VPN replacement use case this scenario describes |

### Recommendation for This Scenario
These are **complementary, not competing** — this is the trap the exam scenario is designed to test. **Replace the user-facing VPN with Entra Private Access** (per-app, identity-aware access for remote/hybrid users and contractors reaching internal apps) and **Entra Internet Access** (identity-aware, policy-enforced internet egress, replacing "backhaul everything through the VPN to filter it"). **Keep hub-spoke + Azure Firewall** for what it's actually needed for: east-west traffic between Azure workloads and centralized outbound filtering for workload (not user) traffic — the build in AZ-305 Lab 5 remains correct for that purpose. The mistake would be replacing one with the other outright; the correct design uses SSE for user-to-app access and hub-spoke for workload-to-workload and workload-to-internet traffic. Part 2–3 implement the SSE half of this design.

---

## Part 2: Enable Global Secure Access and Configure Private Access

Global Secure Access (Microsoft's SSE offering, comprising Entra Internet Access and Entra Private Access) is managed primarily through Microsoft Graph and the Entra admin center — there is no dedicated `az` CLI module for it yet, so this lab uses `az rest` against the Graph API's `networkAccess` surface, consistent with how Labs 1–2 handled Conditional Access.

### Step 1: Confirm Licensing and Enable the Private Access Traffic Forwarding Profile
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/beta/networkAccess/forwardingProfiles" \
  --query "value[].{name:trafficForwardingType, state:state}" -o table

az rest --method PATCH \
  --url "https://graph.microsoft.com/beta/networkAccess/forwardingProfiles/privateAccess" \
  --body '{"state": "enabled"}' \
  --headers "Content-Type=application/json"
```
**Expected result**: the `privateAccess` forwarding profile shows `state: enabled` — this is the on/off switch for routing traffic to internal apps through the Global Secure Access client instead of a traditional VPN tunnel.

### Step 2: Register the Internal Application (Replacing One VPN-Granted Destination)
Register the on-prem file server segment from Lab 1's incident as a discrete, named application instead of a network range the VPN granted blanket access to.
```bash
cat > private-access-app.json << 'EOF'
{
  "displayName": "SC100-Lab6-Finance-FileServer",
  "segments": [
    {
      "destinationHost": "<internal-fileserver-hostname-or-ip>",
      "protocol": "tcp",
      "ports": ["445"]
    }
  ]
}
EOF

az rest --method POST \
  --url "https://graph.microsoft.com/beta/networkAccess/connectivity/applications" \
  --body @private-access-app.json --headers "Content-Type=application/json"
```
This is the structural difference from the VPN model: access is defined per destination-and-port (SMB on the finance file server specifically), not "everything reachable once the tunnel is up."

### Step 3: Gate the Application Behind a Conditional Access Policy
This is where Lab 2's layering model and this lab's network design connect — Private Access applications are Conditional Access-protected resources, just like any other app.
```bash
cat > ca-policy-private-access.json << 'EOF'
{
  "displayName": "SC100-Lab6-PrivateAccess-FileServer-RequireCompliantDevice",
  "state": "enabledForReportingButNotEnforced",
  "conditions": {
    "users": { "includeUsers": ["All"], "excludeUsers": ["<break-glass-account-object-id>"] },
    "applications": { "includeApplications": ["<private-access-app-id-from-step-2>"] }
  },
  "grantControls": { "operator": "AND", "builtInControls": ["mfa", "compliantDevice"] }
}
EOF

az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --body @ca-policy-private-access.json --headers "Content-Type=application/json"
```

**Validation checkpoint**:
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/beta/networkAccess/connectivity/applications" \
  --query "value[].{name:displayName, segments:segments}" -o table
```
The finance file server should appear as a discrete, port-scoped application — access to it now requires passing the Conditional Access policy from Step 3, on every connection, not just once at VPN tunnel establishment.

---

## Part 3: Configure Internet Access Traffic Profile

### Step 1: Enable the Microsoft 365 and Internet Access Forwarding Profiles
```bash
az rest --method PATCH \
  --url "https://graph.microsoft.com/beta/networkAccess/forwardingProfiles/internetAccess" \
  --body '{"state": "enabled"}' --headers "Content-Type=application/json"
```

### Step 2: Confirm the Traffic Forwarding Profiles Are Active
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/beta/networkAccess/forwardingProfiles" \
  --query "value[].{profile:trafficForwardingType, state:state}" -o table
```
**Expected result**: both `privateAccess` and `internetAccess` show `enabled`. Once users install the Global Secure Access client (or use it via the Entra-integrated Windows/mobile client), their internet-bound traffic is policy-inspected at Microsoft's edge instead of backhauling through the hub firewall — closing the latency gap identified in Part 1 while keeping the same identity-aware enforcement model.

---

## Part 4: Validate Access Logs Prove Per-Application Enforcement

The design claim from Part 1 — "access is evaluated per application, not once at tunnel establishment" — needs the same evidence discipline as every other lab in this track, not just a portal screenshot.

### Step 1: Query Global Secure Access Traffic Logs
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/beta/networkAccess/connectivity/logs?\$filter=applicationId eq '<private-access-app-id>'&\$top=25" \
  --query "value[].{user:userPrincipalName, app:applicationName, action:accessAction, timestamp:createdDateTime}" \
  -o table
```
**Expected result**: each row is a discrete, per-connection access decision tied to a specific user and the specific finance-file-server application — not an entry showing "VPN tunnel established" once per session with no further per-resource record.

### Step 2: Compare Against a Denied Access Attempt
Attempt access to the same application from a device that doesn't satisfy the Part 2 Step 3 Conditional Access policy (e.g., a non-compliant device), then re-run Step 1's query filtered to that user.
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/beta/networkAccess/connectivity/logs?\$filter=applicationId eq '<private-access-app-id>' and userPrincipalName eq '<test-user-upn>'&\$top=5" \
  --query "value[].{action:accessAction, reason:conditionalAccessStatus}" -o table
```
**Validation checkpoint**: the denied attempt should show `accessAction: blocked` with `conditionalAccessStatus` reflecting the failed device compliance check — direct evidence that each connection attempt is independently evaluated against the Conditional Access policy, the exact behavior a flat VPN tunnel structurally cannot provide once a session is already established.

---

## Cleanup
```bash
POLICY_ID=$(az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies?\$filter=startswith(displayName,'SC100-Lab6')" \
  --query "value[0].id" -o tsv)
az rest --method DELETE --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$POLICY_ID"

APP_ID=$(az rest --method GET \
  --url "https://graph.microsoft.com/beta/networkAccess/connectivity/applications" \
  --query "value[?displayName=='SC100-Lab6-Finance-FileServer'].id | [0]" -o tsv)
az rest --method DELETE --url "https://graph.microsoft.com/beta/networkAccess/connectivity/applications/$APP_ID"

az rest --method PATCH \
  --url "https://graph.microsoft.com/beta/networkAccess/forwardingProfiles/privateAccess" \
  --body '{"state": "disabled"}' --headers "Content-Type=application/json"
az rest --method PATCH \
  --url "https://graph.microsoft.com/beta/networkAccess/forwardingProfiles/internetAccess" \
  --body '{"state": "disabled"}' --headers "Content-Type=application/json"
```
Confirm with the Part 2 Step 3 validation query — expect an empty applications list, and both forwarding profiles showing `disabled`.

---

## What You Practiced

| Concept | Why It Matters |
|---|---|
| SSE vs. hub-spoke as complementary, not competing | The single highest-value insight in this lab — SC-100 scenario questions often bait "replace X with Y" when the correct answer is "both, for different traffic" |
| Per-application, per-port access definition | The concrete technical difference between Zero Trust network access and traditional VPN's implicit network-wide trust |
| Private Access apps as Conditional Access-protected resources | Connects the identity layering work from Lab 2 directly to network access decisions |
| Traffic forwarding profiles (Internet Access / Private Access / Microsoft 365 Access) | The building blocks Global Secure Access is actually configured through |
| Recognizing where hub-spoke remains necessary | East-west and workload-to-internet traffic still needs the AZ-305 Lab 5 pattern — SSE doesn't eliminate that requirement |
| Per-connection log evidence vs. per-session tunnel logs | The concrete, auditable difference between "access is re-evaluated continuously" and "access was checked once at login" |

---

## Common Mistakes to Avoid
- **Recommending SSE as a full replacement for hub-spoke networking**: it replaces user-to-app VPN access, not workload-to-workload or workload-to-internet traffic — a subscription with zero network perimeter for its Azure resources is still exposed
- **Recommending hub-spoke+VPN as a full replacement for SSE**: it's a valid, necessary pattern for workload traffic, but reintroduces the exact flat-network-trust problem from Lab 1's incident for user access if it's the only control in place
- **Registering a Private Access application with an overly broad destination range**: defining the segment as an entire subnet instead of the specific host/port needed reproduces the VPN's flat-trust problem inside the "modern" tool
- **Forgetting Private Access apps need their own Conditional Access policy**: registering the app alone doesn't enforce device compliance or MFA — that's a separate, deliberate step, same discipline as Lab 2
- **Assuming Global Secure Access has a mature dedicated `az` CLI module**: as of this writing it's primarily Graph API and portal-driven — reach for `az rest` against the `networkAccess` Graph surface, not a nonexistent native command group
- **Presenting the design without per-connection log evidence**: an architecture claim that access is "continuously evaluated" is unverifiable without pulling the traffic logs, as Part 4 does — this is the same evidence-over-assertion discipline running through every lab in this track

---

## Next Steps
- Continue to [Lab 7: Application & API Security Architecture Design](lab-7-application-api-security-architecture-design.md) to design the access model for the applications this lab's Private Access policy now protects
- For the full hub-spoke + Azure Firewall build this lab deliberately keeps for workload traffic, see [AZ-305 Lab 5: Network Infrastructure Design](../AZ-305/lab-5-network-infrastructure-design.md)
- For the Conditional Access layering model this lab's Private Access policy builds on, see [Lab 2: Identity & Access Architecture Design](lab-2-identity-access-architecture-design.md) in this track
