# Lab 3: Securing Networking for Cloud Workloads

Check box if done: [ ]

## Overview
AZ-500's networking lab established the fundamentals: deny-by-default NSGs, Bastion instead of public IPs, a Private Endpoint for storage. This lab builds on top of that baseline for a workload-centric scenario: NSG flow logs so you can actually see what your deny-by-default rules are blocking, a Private Endpoint for a database this time (not storage), and a centralized Azure Firewall so every workload subnet's outbound traffic goes through one inspected, logged choke point instead of each subnet managing its own egress path.

**Estimated time**: 75–90 minutes
**Cost**: ~$2–$5 (Azure Firewall is the dominant cost here at ~$1.25/hr — deploy it, do the lab, delete it the same session)

---

## Scenario
The workload from Labs 1–2 is getting a network redesign: its NSGs need flow logs so you can prove what they're actually blocking (not just assume), its SQL database needs to be reachable only from inside the VNet, and all outbound internet traffic from the workload subnet needs to funnel through a centralized firewall instead of each resource managing its own egress rules.

---

## Objectives
- Enable **NSG flow logs** and **Traffic Analytics** to get visibility into allowed/denied traffic
- Create a **Private Endpoint** for an Azure SQL Database and verify public access is blocked
- Deploy an **Azure Firewall** as a centralized, policy-driven egress point
- Route workload subnet traffic through the firewall with a **User Defined Route (UDR)**
- Verify both the negative (blocked traffic) and positive (allowed traffic) cases

---

## Part 1: NSG Flow Logs and Traffic Analytics

### Step 1: Build the Network Foundation
```bash
az group create --name sc500-lab3-rg --location eastus2

az network vnet create \
  --resource-group sc500-lab3-rg \
  --name sc500-vnet \
  --address-prefix 10.30.0.0/16 \
  --subnet-name workload-subnet \
  --subnet-prefix 10.30.1.0/24

az network vnet subnet create \
  --resource-group sc500-lab3-rg \
  --vnet-name sc500-vnet \
  --name data-subnet \
  --address-prefixes 10.30.2.0/24

az network vnet subnet create \
  --resource-group sc500-lab3-rg \
  --vnet-name sc500-vnet \
  --name AzureFirewallSubnet \
  --address-prefixes 10.30.255.0/26

az network nsg create --resource-group sc500-lab3-rg --name workload-nsg

az network nsg rule create \
  --resource-group sc500-lab3-rg --nsg-name workload-nsg \
  --name AllowHTTPS --priority 100 \
  --direction Inbound --access Allow --protocol Tcp \
  --destination-port-ranges 443 --source-address-prefixes Internet

az network vnet subnet update \
  --resource-group sc500-lab3-rg --vnet-name sc500-vnet \
  --name workload-subnet --network-security-group workload-nsg
```

### Step 2: Why Flow Logs Matter Beyond the NSG Rules Themselves
An NSG's rules tell you what's *allowed to happen*; flow logs tell you what *actually happened* — every flow evaluated, allowed or denied, with source/destination IP and port. Without flow logs, "prove the NSG is working" is an assumption. With them, it's a query.

### Step 3: Create a Log Analytics Workspace and Storage Account for Flow Logs
```bash
az monitor log-analytics workspace create \
  --resource-group sc500-lab3-rg \
  --workspace-name sc500-lab3-law \
  --location eastus2

az storage account create \
  --name sc500lab3fl$RANDOM \
  --resource-group sc500-lab3-rg \
  --location eastus2 \
  --sku Standard_LRS

FLOW_STORAGE=$(az storage account list -g sc500-lab3-rg --query "[?starts_with(name,'sc500lab3fl')].name" -o tsv)
NSG_ID=$(az network nsg show -g sc500-lab3-rg -n workload-nsg --query id -o tsv)
LAW_ID=$(az monitor log-analytics workspace show -g sc500-lab3-rg -n sc500-lab3-law --query id -o tsv)
```

### Step 4: Enable NSG Flow Logs with Traffic Analytics
Network Watcher must be present in the region (Azure enables it automatically in most regions on first use).

```bash
az network watcher flow-log create \
  --resource-group sc500-lab3-rg \
  --name workload-nsg-flowlog \
  --nsg $NSG_ID \
  --storage-account $FLOW_STORAGE \
  --enabled true \
  --retention 7 \
  --workspace $LAW_ID \
  --interval 10 \
  --location eastus2
```

**Why Traffic Analytics on top of raw flow logs**: raw flow logs are JSON blobs sitting in storage — technically complete but impractical to query at scale. Traffic Analytics processes them into Log Analytics with a 10-minute processing interval, giving you a queryable, visual view of top talkers, malicious IP matches, and traffic patterns.

**Validation checkpoint**: `az network watcher flow-log show --resource-group sc500-lab3-rg --name workload-nsg-flowlog --query "{enabled: enabled, retention: retentionPolicy.days}"` should show `enabled: true`.

---

## Part 2: Private Endpoint for Azure SQL Database

### Step 5: Deploy a SQL Server and Database
```bash
az sql server create \
  --name sc500-lab3-sql-$RANDOM \
  --resource-group sc500-lab3-rg \
  --location eastus2 \
  --admin-user sqladmin \
  --admin-password '<StrongP@ssw0rd-Placeholder>' \
  --enable-public-network-access false

SQL_SERVER=$(az sql server list -g sc500-lab3-rg --query "[0].name" -o tsv)
SQL_SERVER_ID=$(az sql server show -g sc500-lab3-rg -n $SQL_SERVER --query id -o tsv)

az sql db create \
  --resource-group sc500-lab3-rg \
  --server $SQL_SERVER \
  --name workload-db \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 1 \
  --compute-model Serverless \
  --auto-pause-delay 60
```

`--enable-public-network-access false` means, exactly like the storage account pattern from AZ-500 Lab 2, this database is unreachable from the public internet under any credential. The only way in is the Private Endpoint you're about to create.

### Step 6: Create the Private Endpoint
```bash
az network private-endpoint create \
  --resource-group sc500-lab3-rg \
  --name sql-pe \
  --vnet-name sc500-vnet \
  --subnet data-subnet \
  --private-connection-resource-id $SQL_SERVER_ID \
  --group-id sqlServer \
  --connection-name sql-pe-connection
```

### Step 7: Wire Up Private DNS
```bash
az network private-dns zone create \
  --resource-group sc500-lab3-rg \
  --name "privatelink.database.windows.net"

az network private-dns link vnet create \
  --resource-group sc500-lab3-rg \
  --zone-name "privatelink.database.windows.net" \
  --name sql-dns-link \
  --virtual-network sc500-vnet \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group sc500-lab3-rg \
  --endpoint-name sql-pe \
  --name sql-dns-zone-group \
  --private-dns-zone "privatelink.database.windows.net" \
  --zone-name sqlServer
```

### Step 8: Verify Public Access Fails
```bash
curl -sv --max-time 5 telnet://$SQL_SERVER.database.windows.net:1433 2>&1 | tail -5
```

**Expected result**: connection timeout — the SQL server has no public listener. This is the same "verify the negative" discipline as AZ-500 Lab 2's storage Private Endpoint test, applied to a database instead.

**Validation checkpoint**: from a VM or Bastion session inside `sc500-vnet` (deploy a throwaway VM in `data-subnet` if you don't have one handy), `nslookup $SQL_SERVER.database.windows.net` should resolve to a `10.30.2.x` address, not a public IP.

---

## Part 3: Azure Firewall — Centralized Egress

### Step 9: Why Centralize Egress Instead of Per-Subnet NSG Egress Rules
NSGs are good at *segmenting* traffic between subnets but weak at *inspecting* it — an NSG allow rule doesn't know if the destination FQDN is a legitimate SaaS endpoint or a malicious C2 domain, and doesn't log the *reason* a flow was permitted the way a firewall rule collection does. Centralizing egress through Azure Firewall gives you FQDN-based filtering, threat intelligence-based filtering, and one place to review every outbound decision your workload made.

### Step 10: Deploy Azure Firewall
```bash
az network public-ip create \
  --resource-group sc500-lab3-rg \
  --name firewall-pip \
  --sku Standard

az network firewall create \
  --resource-group sc500-lab3-rg \
  --name sc500-firewall \
  --location eastus2

az network firewall ip-config create \
  --resource-group sc500-lab3-rg \
  --firewall-name sc500-firewall \
  --name firewall-ipconfig \
  --public-ip-address firewall-pip \
  --vnet-name sc500-vnet

az network firewall update --resource-group sc500-lab3-rg --name sc500-firewall

FIREWALL_PRIVATE_IP=$(az network firewall show -g sc500-lab3-rg -n sc500-firewall --query "ipConfigurations[0].privateIPAddress" -o tsv)
```

> Firewall deployment takes 5–10 minutes. Move on to Step 11 while it provisions.

### Step 11: Enable Threat Intelligence-Based Filtering
```bash
az network firewall update \
  --resource-group sc500-lab3-rg \
  --name sc500-firewall \
  --threat-intel-mode Deny
```

`Deny` mode blocks traffic to/from IPs and domains on Microsoft's threat intelligence feed outright; `Alert` mode only logs it. Start with `Alert` in a production rollout if you're worried about false positives; `Deny` is fine here since this is a lab environment.

### Step 12: Add an Application Rule — Allow Only Specific FQDNs Outbound
```bash
az network firewall application-rule create \
  --resource-group sc500-lab3-rg \
  --firewall-name sc500-firewall \
  --collection-name workload-app-rules \
  --name allow-windows-update \
  --protocols Https=443 \
  --source-addresses 10.30.1.0/24 \
  --target-fqdns "*.update.microsoft.com" \
  --priority 100 \
  --action Allow
```

Everything else outbound from `workload-subnet` is denied by the firewall's implicit default-deny — the same deny-by-default principle from NSGs, now applied at the FQDN-inspection layer.

### Step 13: Route Workload Subnet Traffic Through the Firewall
```bash
az network route-table create \
  --resource-group sc500-lab3-rg \
  --name workload-route-table

az network route-table route create \
  --resource-group sc500-lab3-rg \
  --route-table-name workload-route-table \
  --name to-firewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FIREWALL_PRIVATE_IP

az network vnet subnet update \
  --resource-group sc500-lab3-rg \
  --vnet-name sc500-vnet \
  --name workload-subnet \
  --route-table workload-route-table
```

**Validation checkpoint**: `az network vnet subnet show -g sc500-lab3-rg --vnet-name sc500-vnet -n workload-subnet --query routeTable.id` should return the route table's ID. Any resource in `workload-subnet` now has all `0.0.0.0/0` traffic forced through `sc500-firewall` — test from a VM in that subnet (if deployed) that an FQDN not in the allow list (e.g., a generic public site) fails to resolve/connect, while `*.update.microsoft.com` succeeds.

### Step 14: Review Firewall Logs
1. Portal → **sc500-firewall** → **Diagnostic settings** → send **AzureFirewallApplicationRule** and **AzureFirewallNetworkRule** logs to `sc500-lab3-law`
2. **Logs** → query:
```kusto
AzureDiagnostics
| where ResourceType == "AZUREFIREWALLS"
| where Category == "AzureFirewallApplicationRule"
| project TimeGenerated, msg_s
| take 20
```

---

## Part 4: Cleanup

```bash
az group delete --name sc500-lab3-rg --yes --no-wait
```

Confirm the firewall in particular is gone — it's the most expensive resource in this lab and bills continuously until deleted:
```bash
az network firewall list --resource-group sc500-lab3-rg
```

---

## What You Practiced

| Concept | Why It Matters |
|---------|-----------------|
| **NSG flow logs + Traffic Analytics** | Turns "the NSG should be blocking this" into a provable, queryable fact instead of an assumption |
| **Private Endpoint for a database** | Same public-access-elimination pattern as AZ-500's storage Private Endpoint, applied to a different, equally common resource type |
| **Azure Firewall FQDN-based application rules** | Inspects and filters *by destination identity*, something NSGs (IP/port only) can't do |
| **Threat intelligence-based filtering** | Blocks traffic to/from known-malicious infrastructure automatically, without hand-maintaining an IP blocklist |
| **User Defined Routes forcing traffic through a firewall** | Centralizes egress decisions and logging into one inspectable choke point instead of per-resource, per-subnet sprawl |

---

## Common Mistakes to Avoid
- **Enabling flow logs but never looking at Traffic Analytics**: raw flow log JSON in storage is technically "enabled logging" but practically useless without the processed, queryable layer
- **Creating the Private Endpoint but skipping the private DNS zone**: same failure mode as AZ-500 Lab 2 — the hostname keeps resolving publicly and connections fail
- **Forgetting the UDR step**: creating an Azure Firewall doesn't route anything through it automatically — without the route table attached to the subnet, traffic still egresses directly
- **Using `Deny` threat intelligence mode in production without first testing in `Alert` mode**: a false positive under `Deny` silently drops legitimate traffic with no easy way to tell why
- **Leaving Azure Firewall running after the lab**: at ~$1.25/hr continuous billing, this is the single most expensive resource across the whole SC-500 track if forgotten

---

## Next Steps
- Continue to [Lab 4: Securing Compute Workloads](lab-4-compute-workload-security.md) to put a VM and a container workload behind this same network foundation
- Compare this Private Endpoint pattern against AZ-500's storage-focused version in [AZ-500 Lab 2](../AZ-500/lab-2-platform-network-security.md) — same mechanism, different resource type
- Add DNS-based network rules and IDPS (Intrusion Detection and Prevention) alerting on the firewall's **Premium** SKU if your workload requires TLS inspection
- Extend the flow log Traffic Analytics workbook to alert on any traffic hitting a `Deny` decision from the workload subnet's NSG — an unexpected deny is worth investigating just as much as an unexpected allow
