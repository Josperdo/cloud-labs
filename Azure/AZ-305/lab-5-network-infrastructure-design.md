# Lab 5: Network Infrastructure Design

Check box if done: [ ]

## Overview
AZ-305 doesn't test whether you can create a VNet — AZ-104 already covers that. It tests whether you can design a *landing zone* network: a topology where perimeter security, egress inspection, and PaaS connectivity are centralized and consistent across every workload that gets added later, instead of re-invented per app. This is the capstone of the track's infrastructure-solutions domain — it pulls together topology, perimeter security, and private connectivity decisions into one build, before Labs 6–8 extend that same landing zone outward to global distribution, migration, and a full multi-domain synthesis.

**Estimated time**: 60–90 minutes
**Cost**: ~$1–$5 — **Azure Firewall bills hourly (~$0.90+/hr, Standard SKU) the moment it's deployed, regardless of traffic volume. Deploy it, complete Part 3 and Part 4, and delete it the same session — see Cleanup.**

---

## Scenario
You're the architect standing up the network foundation for a new landing zone. The design has to support workloads that don't exist yet, so it can't be a flat VNet with ad-hoc NSG rules per app — it needs a hub carrying shared services (centralized firewall, and eventually shared DNS) and one or more spoke VNets that hold the actual workloads. Two non-negotiables from the security team: every byte of inbound and outbound internet traffic must flow through centralized inspection, and every PaaS service a workload talks to (storage, Key Vault, databases) must be reachable only over a private IP inside the VNet — never the public endpoint.

---

## Objectives
- Design a hub-spoke topology and justify it over a flat/mesh VNet design
- Decide between Azure Firewall, NSG+UDR only, and a third-party NVA for centralized egress/ingress inspection
- Distinguish Application Gateway/WAF from Azure Front Door and know when to use each (or both)
- Deploy a hub VNet and spoke VNet with bidirectional peering
- Deploy Azure Firewall and force spoke egress through it with a UDR
- Deploy a Private Endpoint for a PaaS service and prove public access is actually blocked

---

## Part 1: Design Decision — Topology and Perimeter Security Selection

### Decision 1: Hub-Spoke vs. Mesh/Flat VNet Design

| Factor | Hub-Spoke | Mesh / Flat VNet |
|---|---|---|
| **Centralized control** | Shared services (firewall, DNS, VPN/ExpressRoute gateway) live once in the hub; every spoke inherits the same perimeter | Every VNet manages its own security and connectivity independently — no single enforcement point |
| **Peering cost/complexity** | Linear — each spoke peers once with the hub | Grows quadratically (n×(n-1)/2 peerings) as VNets are added; becomes unmanageable past a handful of VNets |
| **Blast radius** | Contained — a compromised spoke's egress is still filtered by the hub firewall; spokes typically can't reach each other directly unless explicitly routed | Larger — direct VNet-to-VNet or flat address space means lateral movement is easier once one VNet is compromised |
| **Operational complexity at scale** | Low — new workload = new spoke, peer it, done. Governance (Azure Policy, RBAC) applies at the hub/landing-zone level | High — every new VNet reintroduces the same security decisions from scratch, and drift between VNets is common |
| **When flat is acceptable** | N/A | Single small app, single team, no plan to add more workloads — the overhead of a hub isn't justified |

### Decision 2: Azure Firewall vs. NSG+UDR Only vs. Third-Party NVA

| Factor | Azure Firewall | NSG + UDR Only | Third-Party NVA |
|---|---|---|---|
| **Cost** | Highest — bills hourly (~$0.90+/hr Standard) plus data processing, regardless of traffic | Free — NSGs and route tables have no direct cost | Variable — VM compute cost plus often a per-hour license fee (e.g., Palo Alto, Fortinet) |
| **Inspection depth** | L3/L4 + L7 FQDN filtering, threat intelligence feed, TLS inspection (Premium SKU) | L3/L4 only — 5-tuple rules (source/dest IP, port, protocol); **no FQDN or application-layer awareness** | Full L3–L7, vendor-dependent — often the deepest inspection available (IPS/IDS, advanced threat feeds) |
| **Operational overhead** | Fully managed — no patching, no HA configuration, Microsoft handles scaling | Lowest — just rule maintenance, but rules must be meticulously scoped since there's no app-layer visibility | Highest — you own patching, licensing, HA pairing, and scaling the appliance yourself |
| **Best fit** | Teams that want centralized L7 control without managing infrastructure | Cost-sensitive labs/small landing zones where L3/L4 filtering is genuinely sufficient | Enterprises with existing vendor investment or requirements Azure Firewall doesn't cover (custom IPS signatures, specific compliance certs) |

**Cost-conscious alternative**: if the hourly Firewall cost isn't justified for a given environment (or for this lab, if you want to skip Part 3's spend entirely), NSGs on every subnet plus a UDR forcing traffic through a lightweight route (even just to a "black hole" next hop for anything unexpected) covers basic segmentation. You lose FQDN-based egress filtering and centralized logging, which is the trade-off to call out explicitly in a design justification.

### Note: Application Gateway/WAF vs. Azure Front Door
Both are L7 reverse proxies with WAF capability — the difference is scope. **Application Gateway** is regional: it sits in front of app(s) within a single region/VNet for path-based routing, SSL offload, and WAF inspection after traffic has already arrived. **Front Door** is global: it sits at Microsoft's edge (anycast) for cross-region load balancing/failover, CDN caching, and WAF inspection *before* traffic enters any Azure region. Use App Gateway alone for a single-region app, Front Door alone for a globally distributed static/API front end, and both together for a multi-region app — Front Door for global routing/edge WAF, App Gateway per region behind it.

**Recommendation for this scenario**: hub-spoke topology, Azure Firewall in the hub for centralized egress/ingress inspection, Private Endpoints for every PaaS service the spokes consume. Part 2–4 build exactly this. The NSG+UDR-only alternative is noted above for anyone replicating this without the Firewall's hourly cost.

---

## Part 2: Deploy Hub VNet and Spoke VNet with Peering

```bash
az group create --name az305-lab5-rg --location eastus2

# Hub VNet — carries shared services. AzureFirewallSubnet is a mandatory, fixed name (/26 minimum) — Azure Firewall won't deploy into a subnet named anything else.
az network vnet create \
  --resource-group az305-lab5-rg \
  --name hub-vnet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name AzureFirewallSubnet \
  --subnet-prefix 10.0.1.0/26

# Spoke VNet — holds the workload
az network vnet create \
  --resource-group az305-lab5-rg \
  --name spoke-vnet \
  --address-prefix 10.1.0.0/16 \
  --subnet-name workload-subnet \
  --subnet-prefix 10.1.1.0/24
```

### Peer the VNets (both directions — peering is not transitive or automatically bidirectional)

```bash
az network vnet peering create \
  --resource-group az305-lab5-rg \
  --name hub-to-spoke \
  --vnet-name hub-vnet \
  --remote-vnet spoke-vnet \
  --allow-vnet-access true \
  --allow-forwarded-traffic true

az network vnet peering create \
  --resource-group az305-lab5-rg \
  --name spoke-to-hub \
  --vnet-name spoke-vnet \
  --remote-vnet hub-vnet \
  --allow-vnet-access true \
  --allow-forwarded-traffic true
```

`--allow-forwarded-traffic true` on both sides matters here — without it, traffic that arrives at the firewall's NIC already routed (rather than originating in that VNet) gets dropped, which breaks Part 3 entirely.

**Validation checkpoint**:
```bash
az network vnet peering list --resource-group az305-lab5-rg --vnet-name hub-vnet -o table
az network vnet peering list --resource-group az305-lab5-rg --vnet-name spoke-vnet -o table
```
Both should show `PeeringState: Connected`. If either shows `Disconnected`, the peering wasn't created on both sides.

---

## Part 3: Deploy Azure Firewall with a Basic Rule

```bash
az network public-ip create \
  --resource-group az305-lab5-rg \
  --name fw-pip \
  --sku Standard \
  --allocation-method Static

az network firewall create \
  --resource-group az305-lab5-rg \
  --name hub-fw \
  --location eastus2

az network firewall ip-config create \
  --resource-group az305-lab5-rg \
  --firewall-name hub-fw \
  --name fw-ipconfig \
  --public-ip-address fw-pip \
  --vnet-name hub-vnet
```

> Firewall deployment takes 5–10 minutes. It is now billing hourly.

Force spoke egress through the firewall with a UDR:

```bash
FW_PRIVATE_IP=$(az network firewall ip-config list \
  --resource-group az305-lab5-rg \
  --firewall-name hub-fw \
  --query "[0].privateIPAddress" -o tsv)

az network route-table create \
  --resource-group az305-lab5-rg \
  --name spoke-udr

az network route-table route create \
  --resource-group az305-lab5-rg \
  --route-table-name spoke-udr \
  --name to-firewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_PRIVATE_IP

az network vnet subnet update \
  --resource-group az305-lab5-rg \
  --vnet-name spoke-vnet \
  --name workload-subnet \
  --route-table spoke-udr
```

Add an application rule and a workload VM to prove inspection is happening:

```bash
az network firewall application-rule create \
  --resource-group az305-lab5-rg \
  --firewall-name hub-fw \
  --collection-name allow-web \
  --name allow-microsoft \
  --protocols Https=443 \
  --source-addresses 10.1.0.0/16 \
  --target-fqdns "*.microsoft.com" \
  --priority 100 \
  --action Allow

az vm create \
  --resource-group az305-lab5-rg \
  --name spoke-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --vnet-name spoke-vnet \
  --subnet workload-subnet \
  --nsg "" \
  --public-ip-address "" \
  --admin-username labadmin \
  --generate-ssh-keys
```

`spoke-vm` has no public IP and no Bastion — `az vm run-command invoke` executes commands through the VM agent over the Azure control plane, which is enough to validate egress without adding another billable resource.

**Validation checkpoint**:
```bash
az vm run-command invoke \
  --resource-group az305-lab5-rg \
  --name spoke-vm \
  --command-id RunShellScript \
  --scripts "curl -s -o /dev/null -w 'microsoft.com: %{http_code}\n' --max-time 6 https://www.microsoft.com; curl -s -o /dev/null -w 'example.com: %{http_code}\n' --max-time 6 https://www.example.com"
```
Expect `microsoft.com` to return an HTTP status code (traffic matched the allow rule and was forwarded) and `example.com` to time out or return no code (it doesn't match any allow rule, and Azure Firewall's application rules default-deny anything unmatched). That gap between the two results is the proof the firewall is inspecting FQDNs, not just passing traffic through.

---

## Part 4: Private Endpoint for a PaaS Service

```bash
az storage account create \
  --name az305lab5$RANDOM \
  --resource-group az305-lab5-rg \
  --location eastus2 \
  --sku Standard_LRS \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --public-network-access Disabled
```

`--public-network-access Disabled` removes the public listener entirely — the Private Endpoint becomes the only path in, not just the preferred one.

```bash
STORAGE_ACCOUNT=$(az storage account list -g az305-lab5-rg --query "[0].name" -o tsv)
STORAGE_ID=$(az storage account show -g az305-lab5-rg -n $STORAGE_ACCOUNT --query id -o tsv)

az network private-endpoint create \
  --resource-group az305-lab5-rg \
  --name storage-pe \
  --vnet-name spoke-vnet \
  --subnet workload-subnet \
  --private-connection-resource-id $STORAGE_ID \
  --group-id blob \
  --connection-name storage-pe-connection

az network private-dns zone create \
  --resource-group az305-lab5-rg \
  --name "privatelink.blob.core.windows.net"

az network private-dns link vnet create \
  --resource-group az305-lab5-rg \
  --zone-name "privatelink.blob.core.windows.net" \
  --name spoke-dns-link \
  --virtual-network spoke-vnet \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group az305-lab5-rg \
  --endpoint-name storage-pe \
  --name storage-dns-zone-group \
  --private-dns-zone "privatelink.blob.core.windows.net" \
  --zone-name blob
```

Note: this Private Endpoint traffic does **not** get routed through the firewall by the `0.0.0.0/0` UDR from Part 3 — Azure always prefers the more specific system route for the VNet's own address space over a broader UDR, so a destination inside `10.1.0.0/16` stays on the VNet-local path. That's correct behavior: PaaS traffic over a Private Endpoint never needs internet-egress inspection.

**Validation checkpoint**:
```bash
az vm run-command invoke \
  --resource-group az305-lab5-rg \
  --name spoke-vm \
  --command-id RunShellScript \
  --scripts "nslookup $STORAGE_ACCOUNT.blob.core.windows.net"
```
Expect the hostname to resolve to a `10.1.1.x` address — a private IP inside `workload-subnet`, not a public one. If it resolves publicly instead, the private DNS zone link is missing or wasn't scoped to `spoke-vnet`.

---

## Cleanup

Azure Firewall is the single most expensive resource in this entire repo. Delete it **first**, immediately after finishing validation — don't wait until you're done reviewing, and don't leave it "for later."

```bash
# 1. Firewall FIRST — this is the meter that's running right now
az network firewall delete --resource-group az305-lab5-rg --name hub-fw

# 2. Release the firewall's public IP (can't delete it while attached)
az network public-ip delete --resource-group az305-lab5-rg --name fw-pip

# 3. Private endpoint
az network private-endpoint delete --resource-group az305-lab5-rg --name storage-pe

# 4. Route table (detach implied by deletion, but do this before the VNet if you want to confirm cleanly)
az network route-table delete --resource-group az305-lab5-rg --name spoke-udr

# 5. VNet peerings (both directions)
az network vnet peering delete --resource-group az305-lab5-rg --vnet-name hub-vnet --name hub-to-spoke
az network vnet peering delete --resource-group az305-lab5-rg --vnet-name spoke-vnet --name spoke-to-hub

# 6. VNets
az network vnet delete --resource-group az305-lab5-rg --name hub-vnet
az network vnet delete --resource-group az305-lab5-rg --name spoke-vnet

# 7. Resource group — catches the VM, storage account, disks, NICs, and private DNS zone
az group delete --name az305-lab5-rg --yes --no-wait
```

Confirm the firewall specifically is gone before moving on to anything else:
```bash
az network firewall list --resource-group az305-lab5-rg -o table
```
Empty output means billing has stopped. Steps 2–7 can run at whatever pace you like after that — step 1 is the one with a clock on it.

---

## Key Concepts

| Term | Definition |
|---|---|
| **Hub-spoke topology** | A hub VNet holding shared services (firewall, DNS, gateways) peered to one or more spoke VNets holding workloads — centralizes control and keeps peering complexity linear as workloads are added |
| **Azure Firewall** | Managed, stateful L3–L7 firewall with FQDN filtering and threat intelligence; the centralized egress/ingress inspection point in a hub |
| **UDR (User Defined Route) / forced tunneling** | A custom route table overriding Azure's default system routes — used here to force `0.0.0.0/0` traffic from the spoke through the firewall's private IP instead of straight to the internet |
| **Application Gateway vs. Front Door** | App Gateway is a regional L7 reverse proxy/WAF for a single region's app; Front Door is a global edge L7 + CDN + WAF for multi-region routing and failover — often used together |
| **Private Endpoint vs. Service Endpoint** | A Private Endpoint gives a PaaS resource an actual private IP inside your VNet (traffic never leaves the Microsoft backbone, works cross-VNet/cross-region); a Service Endpoint just optimizes routing to the resource's public IP over the Azure backbone — the resource still has a public endpoint |
| **Private DNS zone** | An Azure-managed DNS zone (e.g., `privatelink.blob.core.windows.net`) linked to a VNet that overrides public resolution so a PaaS resource's standard hostname resolves to its Private Endpoint's private IP instead of its public one |
| **AzureFirewallSubnet** | The mandatory, fixed subnet name Azure Firewall requires to deploy into a VNet — must be exactly this name, minimum /26 |

---

## Common Mistakes
- **Not knowing `AzureFirewallSubnet` is a mandatory, fixed name**: Azure Firewall will refuse to deploy into a subnet named anything else — this trips people up in the portal and in Bicep/Terraform alike
- **Leaving Azure Firewall running after the lab session ends**: it bills hourly whether or not any traffic passes through it — do Cleanup step 1 the moment validation is done, not "later today"
- **Creating a Private Endpoint but not disabling public access on the resource**: the resource is now reachable both privately and publicly, which defeats the entire point — always pair a Private Endpoint with `--public-network-access Disabled`
- **Forgetting to link the private DNS zone to the VNet, or peering only one direction**: the Private Endpoint's hostname keeps resolving publicly if the DNS zone isn't linked; peering must be created explicitly on both VNets, and `--allow-forwarded-traffic` must be set on both or firewall-routed traffic gets dropped

---

## Next Steps
This lab is the network foundation the rest of the track builds outward from — everything from VNet basics through identity, data, business continuity, and compute design converges here in a single landing-zone network build. For the foundational VNet/NSG concepts this lab assumes, see [AZ-104 Lab 1: Virtual Networks](../AZ-104/lab-1-virtual-networks.md). For the Bastion and Private Endpoint depth this lab builds on and extends to a multi-VNet topology, see [AZ-500 Lab 2: Platform & Network Security](../AZ-500/lab-2-platform-network-security.md). To review the compute/application side of this same landing zone, see [Lab 4: Compute & Application Infrastructure Design](lab-4-compute-app-infrastructure-design.md) in this folder. Continue to [Lab 6: Multi-Region & Global Distribution Design](lab-6-multi-region-global-distribution-design.md) to extend this hub-spoke pattern globally, or jump ahead to [Lab 8: Capstone — Landing Zone Solution Design](lab-8-capstone-landing-zone-solution-design.md) to see how this network design fits into the track's full synthesis.
