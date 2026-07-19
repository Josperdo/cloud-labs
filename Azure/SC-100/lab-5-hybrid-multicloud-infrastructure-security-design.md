# Lab 5: Hybrid & Multicloud Infrastructure Security Design

Check box if done: [ ]

## Overview
Lab 4's compliance program assumed everything worth governing lives in one Azure subscription. That's rarely true past a certain company size — the org from this track's scenario still runs an on-prem file server (the very one the Lab 1 attacker reached) and, post-acquisition, inherited a subsidiary running production workloads in AWS. A governance and detection strategy that only covers native Azure resources has a blind spot exactly where the last incident happened. This lab designs the hybrid identity model and the multicloud security posture strategy, then onboards a representative resource from outside Azure into the same control plane the earlier labs built.

**Estimated time**: 60–75 minutes
**Cost**: ~$0 (Azure Arc onboarding and Defender for Cloud's free posture tier for connected resources don't bill; enabling a paid Defender plan on the connector is described but not required to complete the lab)

---

## Scenario
The on-prem file server from Lab 1's incident still authenticates against a local Active Directory that syncs to Entra ID, but the exact sync method has never been documented or revisited since it was set up years ago. Separately, the newly acquired subsidiary runs its production workloads in AWS, monitored today only through native AWS tooling that the central security team has no visibility into — the first the SOC would hear about an AWS incident is when the subsidiary's team escalates it manually. You're designing both the hybrid identity authentication model going forward and how multicloud resources get pulled into the same Defender for Cloud posture and Sentinel detection surface the rest of this track has been building.

---

## Objectives
- Decide between Password Hash Sync, Pass-through Authentication, and Federation for hybrid identity, and justify the choice against the org's incident history
- Decide between siloed native-cloud tooling and a unified multicloud connector model for cross-cloud security posture
- Onboard a hybrid/on-prem server to Azure Arc so it becomes governable by the same Policy and Defender for Cloud surface as native Azure resources
- Enable a Defender for Cloud multicloud connector for AWS
- Validate that the Arc-connected resource and the AWS connector both surface in the same posture inventory
- Query Azure Resource Graph to prove unified visibility technically, and identify any coverage gaps between environments

---

## Part 1: Design Decision — Hybrid Identity Model and Multicloud Security Posture Strategy

### Decision 1: Password Hash Sync vs. Pass-through Authentication vs. Federation (AD FS)

| Factor | Password Hash Sync (PHS) | Pass-through Authentication (PTA) | Federation (AD FS) |
|---|---|---|---|
| **On-prem dependency for auth** | None after sync — Entra ID validates the hash directly, so cloud sign-in works even if every on-prem domain controller is down | High — every sign-in requires a live PTA agent round-trip to validate against on-prem AD; an on-prem outage blocks cloud sign-in too | Highest — every sign-in redirects to on-prem AD FS; an AD FS or network outage blocks all federated sign-in |
| **Leaked credential detection** | Strong — Microsoft can compare synced hashes against known-breached credential databases (Identity Protection's leaked credentials detection) | Weaker — Microsoft never sees the password hash, so this specific detection doesn't apply the same way | Weaker for the same reason as PTA, plus AD FS has its own historical CVE exposure as an internet-facing component |
| **Implementation complexity / attack surface** | Lowest — no additional on-prem agent exposed to the internet | Moderate — PTA agents are outbound-only (no inbound firewall rule needed), lower exposure than AD FS | Highest — AD FS proxy/WAP servers are historically a common target (this is close to the class of infrastructure the Lab 1 VPN breach resembles) |
| **Fit given the org's incident history** | Strong — reduces on-prem-facing attack surface exactly where the org already had a breach, and enables the leaked-credential signal Identity Protection needs for Lab 1's risk-based CA layer | Weaker fit — keeps a live dependency on on-prem infrastructure trust | Poor fit — the highest-complexity, highest-historical-CVE option, hardest to justify right after an on-prem-adjacent breach |

### Decision 2: Siloed Native-Cloud Tooling vs. Unified Multicloud Connector Model

| Factor | Siloed (AWS Security Hub for AWS workloads, native Azure tooling for Azure, no cross-pollination) | Defender for Cloud Multicloud Connectors (Arc-enabled, single Defender for Cloud pane across Azure + AWS + GCP) |
|---|---|---|
| **Unified visibility for the central SOC** | None — the SOC built in Lab 3 has zero visibility into the AWS subsidiary until someone manually escalates | Full — AWS resources appear in the same Defender for Cloud inventory, Secure Score, and (via the same connector) Sentinel, alongside native Azure resources |
| **Agent/extension footprint** | Zero incremental footprint, but zero incremental coverage too | Requires the Azure Arc agent (for CSPM-only connectors, agentless — API-based scanning) or Arc + extensions for workload-level protection |
| **Cost** | Pay for AWS-native tooling and Azure-native tooling separately, with no shared licensing benefit | Defender for Cloud's foundational CSPM tier is free per connected account; paid Defender plans (Defender for Servers, etc.) extend coverage for a per-resource cost, same pricing model as native Azure resources |
| **Coverage parity across clouds** | Whatever each native tool happens to cover — inconsistent by definition, since AWS Security Hub and Azure-native tools were never designed to produce comparable findings | Consistent — the same regulatory compliance initiatives and recommendations from Lab 4 extend to AWS resources through the connector, in the same taxonomy |
| **Fit for this scenario** | Poor — this is exactly the blind spot in the scenario: the SOC has no idea what's happening in AWS today | Strong — directly closes the visibility gap without asking the subsidiary to rip out their existing AWS investment |

### Recommendation for This Scenario
**Password Hash Sync** for hybrid identity — lowest on-prem-facing attack surface, and it directly enables Identity Protection's leaked-credential detection, which strengthens the risk-based layer of Lab 2's Conditional Access design. If a regulatory requirement later mandates on-prem-only credential validation, PTA is the fallback — Federation is not recommended given the org's incident profile. **Defender for Cloud multicloud connectors with Azure Arc** for the AWS subsidiary — extends the same posture management, compliance initiatives, and (via Arc) Sentinel detection surface across cloud boundaries instead of maintaining two disconnected security programs. Part 2–3 implement the Arc onboarding and AWS connector halves of this design.

---

## Part 2: Onboard a Hybrid Server to Azure Arc

Arc-enabling a server makes it a first-class citizen for Azure Policy (Lab 4's compliance initiatives), Defender for Cloud, and Sentinel data collection — the same governance surface as a native Azure VM, without migrating the server itself.

### Step 1: Register the Required Resource Providers
```bash
az provider register --namespace Microsoft.HybridCompute
az provider register --namespace Microsoft.GuestConfiguration
az provider register --namespace Microsoft.HybridConnectivity

az provider show --namespace Microsoft.HybridCompute --query registrationState -o tsv
```

### Step 2: Create a Service Principal for Arc Onboarding
```bash
az ad sp create-for-rbac --name "sc100-lab5-arc-onboarding" \
  --role "Azure Connected Machine Onboarding" \
  --scopes "/subscriptions/<subscription-id>/resourceGroups/sc100-lab5-rg"
```
Scope the onboarding service principal to a resource group, not the subscription — least privilege applies to onboarding automation the same way it applies to human roles.

### Step 3: Install and Run the Connected Machine Agent (on the target server)
```bash
# Run on the on-prem/hybrid server itself, not the CLI workstation
# Download: https://aka.ms/AzureConnectedMachineAgent

azcmagent connect \
  --resource-group "sc100-lab5-rg" \
  --tenant-id "<tenant-id>" \
  --location "eastus" \
  --subscription-id "<subscription-id>" \
  --service-principal-id "<onboarding-sp-app-id>" \
  --service-principal-secret "<onboarding-sp-secret>"
```
`azcmagent` runs directly on the target machine — this is the one step in this lab that can't be run from the CLI workstation, since Arc onboarding registers *that specific server's* identity.

**Validation checkpoint**:
```bash
az connectedmachine list --resource-group sc100-lab5-rg -o table
```
The server should appear with `status: Connected`. It's now visible to Azure Policy (Lab 4's regulatory compliance initiatives can evaluate it), Defender for Cloud, and can host the Azure Monitor Agent for Sentinel log collection (Lab 3).

---

## Part 3: Enable the Defender for Cloud Multicloud Connector for AWS

### Step 1: Confirm the Security Connector Resource Provider
```bash
az provider register --namespace Microsoft.Security
```

### Step 2: Create the AWS Connector (CSPM Tier — Agentless, Free)
```bash
cat > aws-connector.json << 'EOF'
{
  "location": "global",
  "properties": {
    "hierarchyIdentifier": "<aws-account-id>",
    "environmentName": "AWS",
    "environmentData": {
      "environmentType": "AwsAccount",
      "organizationalData": { "organizationMembershipType": "Member" }
    },
    "offerings": [
      { "offeringType": "CspmMonitorAws" }
    ]
  }
}
EOF

az rest --method PUT \
  --url "https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/sc100-lab5-rg/providers/Microsoft.Security/securityConnectors/sc100-lab5-aws-connector?api-version=2023-10-01-preview" \
  --body @aws-connector.json --headers "Content-Type=application/json"
```
`CspmMonitorAws` is the agentless posture-management offering — it reads AWS resource configuration via API (using a cross-account IAM role Defender for Cloud provisions) without installing anything inside the AWS account, matching the "close the gap without ripping out the subsidiary's existing setup" framing from Part 1.

### Step 3: Confirm the Connector and Its AWS-Side IAM Role
```bash
az rest --method GET \
  --url "https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/sc100-lab5-rg/providers/Microsoft.Security/securityConnectors/sc100-lab5-aws-connector?api-version=2023-10-01-preview" \
  --query "{name:name, offerings:properties.offerings[].offeringType}"
```
The portal (**Defender for Cloud → Environment settings**) surfaces a CloudFormation template to deploy in the AWS account — that template creates the read-only cross-account IAM role Defender for Cloud assumes to scan AWS resource configuration. That deployment step happens on the AWS side and is outside this lab's Azure-CLI scope, but the connector resource created here is what triggers it.

**Validation checkpoint**:
```bash
az rest --method GET \
  --url "https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/sc100-lab5-rg/providers/Microsoft.Security/securityConnectors?api-version=2023-10-01-preview" \
  --query "value[].{name:name, hierarchy:properties.hierarchyIdentifier}" -o table
```
The connector should be listed. Once the AWS-side CloudFormation stack is deployed (in a live environment), AWS resources begin appearing in **Defender for Cloud → Inventory** alongside native Azure and Arc-connected resources — the same Secure Score and regulatory compliance surface from Lab 4, now spanning three environments.

---

## Part 4: Query the Unified Inventory Across Hybrid and Multicloud

The design's actual payoff is a single inventory query that answers "what do we have and is it covered" across all three environments — proving Decision 2's unified-visibility claim technically, not just architecturally.

### Step 1: Query Azure Resource Graph for Arc-Connected and Native Resources Together
```bash
az graph query -q "
Resources
| where type =~ 'microsoft.hybridcompute/machines' or type =~ 'microsoft.compute/virtualmachines'
| extend resourceOrigin = iff(type =~ 'microsoft.hybridcompute/machines', 'Arc-connected (hybrid)', 'Native Azure')
| project name, resourceOrigin, location, resourceGroup"
```
Azure Resource Graph treats Arc-connected servers and native Azure VMs as queryable resources in the same graph — this is the concrete mechanism behind "Arc makes hybrid resources first-class citizens," not just a marketing claim.

### Step 2: Extend the Query to Include Defender for Cloud Coverage Status
```bash
az graph query -q "
securityresources
| where type =~ 'microsoft.security/assessments'
| where properties.displayName contains 'Arc' or properties.displayName contains 'multi-cloud'
| project resourceId = id, assessmentName = properties.displayName, status = properties.status.code"
```
**Expected result**: assessment results covering both the Arc-connected server from Part 2 and, once the AWS CloudFormation stack is live, AWS resources under the connector from Part 3 — the same recommendation taxonomy applied regardless of where the resource actually runs.

**Validation checkpoint**: cross-reference Step 1's resource list against Step 2's assessment coverage. Any resource in Step 1 with zero matching assessments in Step 2 is a coverage gap — either the Arc agent's Defender for Cloud extension hasn't been enabled yet, or (for the AWS side) the connector's CloudFormation stack hasn't finished deploying. Finding and closing that gap is the operational task this design enables; this lab proves the query exists to find it.

---

## Cleanup
```bash
az rest --method DELETE \
  --url "https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/sc100-lab5-rg/providers/Microsoft.Security/securityConnectors/sc100-lab5-aws-connector?api-version=2023-10-01-preview"

# Run on the on-prem/hybrid server itself
azcmagent disconnect

az ad sp delete --id "<onboarding-sp-app-id>"
az group delete --name sc100-lab5-rg --yes --no-wait
```
Confirm with `az connectedmachine list --resource-group sc100-lab5-rg -o table` (empty) and the security connector list query from Part 3 (empty).

---

## What You Practiced

| Concept | Why It Matters |
|---|---|
| PHS vs. PTA vs. Federation trade-offs | SC-100 tests choosing a hybrid identity model against a stated risk/availability/attack-surface profile, not just knowing the three options exist |
| Azure Arc as a governance on-ramp | The mechanism that extends Policy, Defender for Cloud, and Sentinel to resources outside native Azure — a recurring SC-100 theme |
| Multicloud CSPM via Defender for Cloud connectors | Closes the exact "SOC has no visibility into acquired subsidiary's cloud" scenario pattern the exam favors |
| Agentless (CSPM-only) vs. agent-based multicloud coverage | Knowing which offering type to reach for depending on whether posture visibility or full workload protection is the requirement |
| Cross-account IAM role provisioning for CSPM | Understanding that agentless scanning still requires a scoped, read-only trust relationship — not literally zero configuration on the target cloud |
| Azure Resource Graph across hybrid/multicloud | The concrete query surface that turns "unified visibility" from an architecture diagram claim into something an engineer can actually run and get an answer from |

---

## Common Mistakes to Avoid
- **Recommending Federation by default because it "feels more secure"**: it's the highest-complexity, highest-historical-CVE option of the three — the right choice depends on the org's actual constraints, and often isn't Federation
- **Choosing PTA or Federation without weighing the on-prem outage dependency**: both make cloud sign-in dependent on on-prem infrastructure staying up, which PHS avoids entirely
- **Treating Arc onboarding as identical to deploying a native Azure VM**: the agent, service principal scoping, and connectivity model are all different, even though the governance surface afterward looks the same
- **Granting the Arc onboarding service principal subscription-level scope**: scope it to the specific resource group being onboarded into
- **Assuming a multicloud connector alone provides workload-level protection**: the CSPM tier used here is posture/configuration visibility only — protecting running workloads (malware detection, etc.) requires enabling the corresponding paid Defender plan on top of the connector
- **Declaring "unified visibility" achieved without ever running a cross-environment query**: an architecture diagram showing Arc and the AWS connector pointing at the same Defender for Cloud icon isn't proof — Part 4's Resource Graph query is what actually demonstrates it, and what surfaces coverage gaps before an auditor or an incident does

---

## Next Steps
- Continue to [Lab 6: Network Security Architecture Design](lab-6-network-security-architecture-design.md) to design how this now-multicloud, now-hybrid estate gets network-level access control
- For hands-on hybrid AD and Entra Connect mechanics this lab's identity decision builds on, see [SC-300 Lab 1: Identity Governance Foundations](../SC-300/lab-1-identity-governance-foundations.md)
- For native Azure VM security controls this lab extends to hybrid/multicloud scope, see [AZ-500 Lab 2: Platform & Network Security](../AZ-500/lab-2-platform-network-security.md)
