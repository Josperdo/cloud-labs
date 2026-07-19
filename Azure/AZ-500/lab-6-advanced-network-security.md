# Lab 6: Advanced Network Security

Check box if done: [ ]

## Overview
Lab 2 built the baseline: deny-by-default NSGs, no public IPs on VMs, Private Endpoints for data-plane traffic. This lab goes past the baseline into the controls that catch what a stateless NSG can't — a centralized firewall that inspects traffic *content*, not just ports and IPs; DDoS protection that defends against volumetric attacks NSGs were never designed to stop; and a web application firewall that understands HTTP well enough to block SQL injection and cross-site scripting at layer 7. These are also the most expensive controls in the entire track — treat the cost warnings in this lab as seriously as the security guidance.

**Estimated time**: 90–120 minutes
**Cost**: ~$8–$18 if deployed for a short window and torn down the same session (Azure Firewall Premium and Application Gateway WAF_v2 both bill hourly — see cost callouts throughout and the Cleanup section)

---

## Scenario
The network team signed off on Lab 2's NSG/Bastion baseline, but a recent tabletop exercise raised two gaps: NSGs can't inspect *inside* allowed traffic (a compromised web server could exfiltrate data over port 443 all day and an NSG would never notice), and the public-facing web tier has no layer-7 protection — a basic SQL injection attempt would sail straight through an NSG's port-443 allow rule. You're closing both gaps: a centralized Azure Firewall Premium for east-west and egress inspection, and a WAF in front of the public web tier. You're also documenting (not necessarily deploying) the DDoS protection tier decision, because the "right" tier here is genuinely a cost question, not just a security one.

---

## Objectives
- Deploy Azure Firewall Premium and enable IDPS (Intrusion Detection and Prevention System) in Alert mode, then Alert+Deny
- Configure TLS inspection and understand the certificate chain requirement and its trade-offs
- Compare Azure DDoS Protection's IP Protection and Network Protection tiers and make a documented tier decision
- Deploy a Web Application Firewall policy on Application Gateway with managed rules and a custom rule, tested in Detection mode before Prevention
- Tear everything down deliberately — this lab's resources are the most expensive in the track

---

## Part 1: Azure Firewall Premium and IDPS

### Step 1: Why Firewall Premium Instead of Standard
Azure Firewall Standard filters on IP, port, and FQDN. Firewall **Premium** adds IDPS (signature-based intrusion detection/prevention against known attack patterns), TLS inspection (decrypt, inspect, re-encrypt outbound HTTPS), URL filtering (not just FQDN filtering), and web category filtering. The scenario's "compromised web server exfiltrating over 443" gap is specifically what IDPS and TLS inspection are built to catch — a Standard-tier firewall would let that traffic through unexamined as long as the destination FQDN looked legitimate.

> **Cost warning**: Azure Firewall Premium bills continuously — base Firewall pricing (~$1.25/hr) plus a premium add-on. Do not leave it running longer than this lab session. Deploy, complete the lab, tear down.

### Step 2: Build the Hub VNet and Deploy Firewall Premium

```bash
az group create --name az500-lab6-rg --location eastus2

az network vnet create \
  --resource-group az500-lab6-rg \
  --name az500-hub-vnet \
  --address-prefix 10.30.0.0/16 \
  --subnet-name AzureFirewallSubnet \
  --subnet-prefix 10.30.0.0/26

az network public-ip create \
  --resource-group az500-lab6-rg \
  --name fw-pip \
  --sku Standard \
  --allocation-method Static

az network firewall policy create \
  --resource-group az500-lab6-rg \
  --name az500-fw-policy \
  --sku Premium

az network firewall create \
  --resource-group az500-lab6-rg \
  --name az500-firewall \
  --location eastus2 \
  --tier Premium \
  --policy az500-fw-policy

az network firewall ip-config create \
  --resource-group az500-lab6-rg \
  --firewall-name az500-firewall \
  --name fw-ipconfig \
  --public-ip-address fw-pip \
  --vnet-name az500-hub-vnet
```

> Firewall deployment takes 10–15 minutes. Continue reading Step 3 while it provisions.

### Step 3: Enable IDPS in Alert Mode First
Just like Lab 1's Conditional Access policies started in report-only mode, IDPS starts in **Alert** mode — log what it would have blocked, without blocking anything, so you can confirm it's not about to break legitimate traffic before enforcement.

```bash
az network firewall policy update \
  --resource-group az500-lab6-rg \
  --name az500-fw-policy \
  --idps-mode Alert
```

**Validation checkpoint**: `az network firewall policy show --resource-group az500-lab6-rg --name az500-fw-policy --query "intrusionDetection.mode"` should return `Alert`.

### Step 4: Review Alerts, Then Move to Alert+Deny
1. Portal → **az500-firewall** → **Logs** (or the associated Log Analytics workspace if diagnostics are wired up) → review any IDPS signature hits over a representative traffic window
2. Once you've confirmed no legitimate traffic is triggering signatures unexpectedly, switch to enforcement:

```bash
az network firewall policy update \
  --resource-group az500-lab6-rg \
  --name az500-fw-policy \
  --idps-mode Deny
```

**Why this order matters**: flipping straight to Deny risks blocking legitimate traffic that happens to resemble a known attack signature (a surprisingly common false-positive source with generic signatures) — Alert-first gives you evidence before you enforce, the same discipline as Lab 1's report-only Conditional Access and Lab 6's own WAF section later in this lab.

---

## Part 2: TLS Inspection

### Step 5: Why TLS Inspection Requires a Certificate Chain
Firewall Premium's IDPS and application-layer filtering can't inspect encrypted traffic without decrypting it first. TLS inspection makes the firewall a man-in-the-middle by design: it terminates the outbound TLS session, inspects the plaintext, then re-encrypts and forwards it using a certificate it controls. For clients to trust that re-encrypted connection without throwing certificate errors, the firewall needs an **intermediate CA certificate** that's already trusted by every client behind it (typically distributed via Group Policy to domain-joined machines, the same distribution mechanism as Lab 5's Seamless SSO zone assignment).

### Step 6: Store the Intermediate CA Certificate in Key Vault
Firewall Premium reads the TLS inspection certificate from Key Vault, not from a local upload — consistent with Lab 3's pattern of centralizing key material instead of scattering it across resources.

```bash
az keyvault create \
  --resource-group az500-lab6-rg \
  --name az500kv6$RANDOM \
  --location eastus2 \
  --enable-rbac-authorization true

# In a real deployment: import your organization's intermediate CA cert (pfx) here.
# This lab references it conceptually rather than generating a real trusted CA chain.
az keyvault certificate create \
  --vault-name <your-keyvault-name> \
  --name fw-tls-inspection-ca \
  --policy "$(az keyvault certificate get-default-policy)"
```

Grant Azure Firewall's managed identity `Key Vault Certificate User` and `Key Vault Crypto User` on the vault, then configure TLS inspection in the firewall policy:

1. **az500-fw-policy** → **TLS inspection** → **Enable TLS inspection**
2. Select the Key Vault and the `fw-tls-inspection-ca` certificate
3. **Save**

### Step 7: Know What Breaks Before You Turn This On
TLS inspection isn't free of side effects — document these trade-offs before enabling it broadly:
- **Certificate pinning**: apps that pin a specific certificate/public key (many mobile apps, some API clients) hard-fail against the firewall's re-issued certificate — there's no way to inspect pinned-cert traffic without breaking it
- **Compliance-sensitive traffic**: some regulated traffic (payment or health endpoints) may have contractual restrictions on being decrypted by a third-party device, even your own firewall — exclude these destinations explicitly
- **Performance**: decrypting/re-encrypting every session adds latency, small but nonzero at scale

**Validation checkpoint**: **TLS inspection** should list the applied certificate scoped to specific rule collections, not blindly applied to all traffic — exclude known-pinned destinations from the inspected rule set.

---

## Part 3: Azure DDoS Protection — Tier Decision

### Step 8: Compare the Tiers
DDoS Protection in Azure comes in two tiers with a large cost gap, and choosing wrong in either direction is a real mistake — over-provisioning wastes budget, under-provisioning leaves a genuine gap.

| | IP Protection | Network Protection |
|---|---|---|
| **Scope** | Per public IP address | Entire virtual network |
| **Approximate cost** | Per-IP, pay-as-you-go — a few dollars per protected IP/month | ~$2,944/month flat, regardless of how many resources it covers |
| **Adaptive tuning + Rapid Response support + attack cost protection** | No | Yes, all three |
| **Best fit** | Small number of public-facing IPs, cost-sensitive | Large environments with many public IPs, compliance requirements demanding the highest tier |

### Step 9: Make the Call for This Lab
Network Protection's flat ~$2,944/month makes it cost-prohibitive to actually deploy for a short lab session — do **not** enable it here. Treat it as a portal walkthrough only: **DDoS protection plans** → review the **Create a DDoS protection plan** blade to see the Network Protection configuration surface (VNet association, alerting) without completing the create. For the IP Protection tier, which is realistic to test:

```bash
az network public-ip update \
  --resource-group az500-lab6-rg \
  --name fw-pip \
  --ddos-protection-mode Enabled \
  --ddos-protection-plan-mode "IpProtection"
```

**Validation checkpoint**: `az network public-ip show --resource-group az500-lab6-rg --name fw-pip --query "ddosSettings"` should reflect the IP Protection mode. Document, in your own notes, which tier your organization would choose in production and why — this decision should reference the number of public IPs in scope and any compliance requirement for Rapid Response support, not just sticker price.

---

## Part 4: WAF Policy on Application Gateway

### Step 10: Deploy Application Gateway with WAF_v2

```bash
az network vnet subnet create \
  --resource-group az500-lab6-rg \
  --vnet-name az500-hub-vnet \
  --name appgw-subnet \
  --address-prefixes 10.30.1.0/24

az network public-ip create \
  --resource-group az500-lab6-rg \
  --name appgw-pip \
  --sku Standard \
  --allocation-method Static

az network application-gateway waf-policy create \
  --resource-group az500-lab6-rg \
  --name az500-waf-policy

az network application-gateway create \
  --resource-group az500-lab6-rg \
  --name az500-appgw \
  --location eastus2 \
  --sku WAF_v2 \
  --vnet-name az500-hub-vnet \
  --subnet appgw-subnet \
  --public-ip-address appgw-pip \
  --waf-policy az500-waf-policy \
  --priority 100
```

### Step 11: Attach the OWASP Managed Rule Set
```bash
az network application-gateway waf-policy managed-rule rule-set add \
  --resource-group az500-lab6-rg \
  --policy-name az500-waf-policy \
  --type OWASP \
  --version 3.2
```

The OWASP Core Rule Set (CRS) covers the common web attack categories — SQL injection, XSS, remote file inclusion, protocol anomalies — without you having to write signatures yourself. It's the baseline every WAF policy should start from.

### Step 12: Add a Custom Rule
Managed rules cover generic attack patterns; custom rules cover *your* application's specific known-bad traffic. This example rate-limits requests carrying a suspicious header pattern:

```bash
az network application-gateway waf-policy custom-rule create \
  --resource-group az500-lab6-rg \
  --policy-name az500-waf-policy \
  --name BlockSuspiciousHeader \
  --priority 10 \
  --rule-type MatchRule \
  --action Block

az network application-gateway waf-policy custom-rule match-condition add \
  --resource-group az500-lab6-rg \
  --policy-name az500-waf-policy \
  --name BlockSuspiciousHeader \
  --match-variables RequestHeaders.Value \
  --operator Contains \
  --values "<placeholder-known-bad-header-value>"
```

### Step 13: Test in Detection Mode Before Switching to Prevention
Same pattern as Lab 1's report-only Conditional Access and this lab's own Alert-mode IDPS — a WAF policy should prove itself before it starts blocking real traffic.

```bash
az network application-gateway waf-policy policy-setting update \
  --resource-group az500-lab6-rg \
  --policy-name az500-waf-policy \
  --state Enabled \
  --mode Detection
```

1. Send representative traffic to the Application Gateway's public IP (including at least one deliberately malformed request, e.g. a query string containing `' OR '1'='1`) to generate WAF log entries
2. Review **az500-waf-policy** → **Diagnostics** (or the linked Log Analytics workspace) for `ApplicationGatewayFirewallLog` entries — confirm the malicious-looking request was logged as a match, and confirm no legitimate request patterns triggered a false match
3. Once confident, switch to enforcement:

```bash
az network application-gateway waf-policy policy-setting update \
  --resource-group az500-lab6-rg \
  --policy-name az500-waf-policy \
  --state Enabled \
  --mode Prevention
```

**Validation checkpoint**: Repeat the malformed request from Step 13.1 — it should now receive a `403 Forbidden` directly from the Application Gateway, never reaching the backend.

---

## Part 5: Cleanup

```bash
az group delete --name az500-lab6-rg --yes --no-wait

# Confirm the two most expensive resources in the track are actually gone -- don't rely solely on
# the resource-group delete finishing silently
az network firewall list --resource-group az500-lab6-rg
az network application-gateway list --resource-group az500-lab6-rg

# If you enabled a DDoS Network Protection plan against the advice in Step 9, delete it explicitly --
# it is not scoped to this resource group and bills ~$2,944/month regardless of resource-group state
az network ddos-protection list --query "[].name" -o tsv
```

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Azure Firewall Premium with IDPS** | Inspects the content of allowed traffic, not just source/port — catches what NSGs structurally cannot |
| **Alert-mode before Deny-mode IDPS** | Same report-only discipline as Lab 1's Conditional Access — proves the control before it can break legitimate traffic |
| **TLS inspection trade-off analysis** | Recognizes that decrypting traffic for inspection has real costs (pinned certs, compliance, latency), not just benefits |
| **DDoS Protection tier decision (IP vs Network)** | Demonstrates cost-aware judgment instead of defaulting to the most expensive tier "to be safe" |
| **WAF managed rules + custom rules, Detection before Prevention** | Layer-7 protection tuned to both generic (OWASP) and application-specific attack patterns, deployed without blind enforcement |

---

## Common Mistakes to Avoid
- **Enabling IDPS Deny mode without an Alert-mode observation period first**: risks blocking legitimate traffic that happens to match a generic signature
- **Enabling TLS inspection tenant-wide without checking for certificate-pinned applications**: guaranteed outage for any pinned client
- **Defaulting to DDoS Network Protection "because it's more secure"**: for most small/lab environments the ~$2,944/month cost is entirely disproportionate to the risk — IP Protection covers the realistic threat model for a handful of public IPs
- **Switching a WAF policy straight to Prevention mode**: skips the evidence-gathering step that catches false positives before they block real customers
- **Leaving Firewall Premium or Application Gateway WAF_v2 running after the lab session**: these are the two most expensive resources in the entire AZ-500 track — verify deletion explicitly, don't assume it

---

## Next Steps
- Route the data-tier traffic from [Lab 2](lab-2-platform-network-security.md) through this lab's Azure Firewall using User Defined Routes, so east-west traffic between the web and data subnets is inspected, not just perimeter traffic
- Extend the WAF policy's custom rules with geo-filtering if the application has a known-legitimate user base concentrated in specific regions
- Wire Firewall and WAF diagnostic logs into the Sentinel workspace from [Lab 4](lab-4-security-ops-automation.md) and build an analytics rule on repeated IDPS or WAF blocks from the same source IP
- Continue to [Lab 7: SQL & Storage Data Security](lab-7-sql-storage-data-security.md) to apply the same defense-in-depth discipline to data at rest and in query
