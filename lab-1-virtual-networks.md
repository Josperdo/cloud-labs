# Lab 1: Virtual Networks — Laying Out a New Application Environment

## Overview
Before any servers or services get deployed, the network needs to exist. As a cloud engineer, setting up a VNet is often the first thing you do when starting a new project. Done right, the network enforces security boundaries so your database tier can't be reached directly from the internet even if something else is misconfigured. This lab walks through creating a properly segmented network, locking down inter-tier traffic, and connecting it to a dev environment.

**Estimated time**: 45–60 minutes
**Cost**: ~$0.50–$1 (mostly free; tear down at the end)

---

## Scenario
Your team is deploying a new web application with a web tier and a database tier. Security policy says the database must not be reachable from the public internet — only the web servers should be able to talk to it. You also need to connect this production network to the dev team's environment so developers can reach production-equivalent infrastructure during testing.

---

## Objectives
- Create a VNet segmented into logical tiers (web and database)
- Write NSG rules that enforce the rule: web tier is public-facing, database tier is private
- Peer two VNets so separate environments can communicate without going through the internet
- Understand how to verify the network is set up the way you think it is

---

## Part 1: Create the Application VNet

### Step 1: Create the VNet with Web and Database Subnets
The IP address range you choose matters — pick something that doesn't overlap with any other network you'll need to connect to later. `10.0.0.0/16` gives you 65,000 addresses, which is plenty for most apps.

1. Search **Virtual networks** → **+ Create**
2. **Basics tab:**
   - **Resource Group**: Create new → `app-network-rg`
   - **Name**: `app-vnet`
   - **Region**: `East US`
3. **IP Addresses tab:**
   - Set the address space to `10.0.0.0/16`
   - Delete any existing default subnet, then add two:
     - **Subnet 1**: Name: `web-subnet`, Address range: `10.0.1.0/24`
     - **Subnet 2**: Name: `db-subnet`, Address range: `10.0.2.0/24`
4. **Security tab**: Leave defaults (you'll configure NSGs separately for more control)
5. **Review + create** → **Create**

**Why separate subnets?** Subnets let you apply different security rules to different parts of the same network. Web servers need to be reachable from the internet; the database should never be.

---

## Part 2: Create the Dev VNet

### Step 2: Create a Dev VNet
The dev environment uses a different IP range so the two networks don't overlap when you peer them. Overlapping address spaces are the most common reason VNet peering fails.

1. **Virtual networks** → **+ Create**
2. **Basics tab:**
   - **Resource Group**: `app-network-rg`
   - **Name**: `dev-vnet`
   - **Region**: `East US`
3. **IP Addresses tab:**
   - Address space: `10.1.0.0/16`
   - Add one subnet:
     - **Name**: `dev-subnet`, Address range: `10.1.1.0/24`
4. **Review + create** → **Create**

---

## Part 3: Secure the Web Tier

### Step 3: Create an NSG for the Web Subnet
An NSG is a stateful firewall applied at the subnet or network interface level. You'll create one for the web tier that allows public HTTP/HTTPS traffic and controlled SSH access.

1. Search **Network security groups** → **+ Create**
2. **Basics:**
   - **Name**: `web-nsg`
   - **Resource Group**: `app-network-rg`
   - **Region**: `East US`
3. Click **Create**

### Step 4: Add Inbound Rules to the Web NSG
1. Open `web-nsg` → Left sidebar → **Inbound security rules** → **+ Add**
2. **Rule 1 — Allow HTTP from the internet:**
   - **Source**: `Any`
   - **Destination port ranges**: `80`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `100`
   - **Name**: `AllowHTTP`
   - Click **Add**

3. **Rule 2 — Allow HTTPS from the internet:**
   - **Destination port ranges**: `443`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `110`
   - **Name**: `AllowHTTPS`
   - Click **Add**

4. **Rule 3 — Allow SSH only from your IP:**
   - **Source**: `IP Addresses`
   - **Source IP addresses**: Your public IP (search "what's my IP" to find it)
   - **Destination port ranges**: `22`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `120`
   - **Name**: `AllowSSH`
   - Click **Add**

**Why lock down SSH to your IP?** Port 22 is constantly scanned by automated bots. A VM with SSH open to `Any` will start receiving brute-force login attempts within minutes of deployment. Restricting to your IP eliminates that entire attack surface.

### Step 5: Associate the Web NSG with the Web Subnet
1. Still in `web-nsg` → Left sidebar → **Subnets** → **+ Associate**
2. **Virtual network**: `app-vnet`
3. **Subnet**: `web-subnet`
4. Click **OK**

---

## Part 4: Secure the Database Tier

### Step 6: Create an NSG for the Database Subnet
The database NSG enforces the most important security rule in this architecture: the database should only accept traffic from the web tier. Nothing else.

1. Create a second NSG: **+ Create** → Name: `db-nsg`, same RG and region
2. Open `db-nsg` → **Inbound security rules** → **+ Add**
3. **Rule — Allow database traffic from web subnet only:**
   - **Source**: `IP Addresses`
   - **Source IP addresses**: `10.0.1.0/24` (the web subnet's range)
   - **Destination port ranges**: `5432` (PostgreSQL) or `3306` (MySQL) — pick whichever matches your app
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `100`
   - **Name**: `AllowDBFromWebTier`
   - Click **Add**

4. Associate `db-nsg` with `db-subnet` in `app-vnet` (same as Step 5)

**What this means operationally**: Even if someone compromises a server in the web tier, they can only reach the database on the database port. They can't RDP/SSH to the database server or access it in any other way, because the NSG blocks everything else.

---

## Part 5: Connect the Dev Environment via VNet Peering

### Step 7: Create VNet Peering
Peering connects two VNets so that traffic between them travels over the Azure backbone, not the public internet. It's low-latency and doesn't require a VPN gateway.

1. Open `app-vnet` → Left sidebar → **Peerings** → **+ Add**
2. **Fill in the peering details:**
   - **Peering link name (this VNet → remote)**: `app-to-dev`
   - **Remote virtual network**: `dev-vnet`
   - **Peering link name (remote → this VNet)**: `dev-to-app`
   - Leave traffic settings at default (allow forwarded traffic)
3. Click **Add**

**Wait ~1 minute**, then verify both peerings show **Connected** status.

**Why peering instead of VPN?** VPN gateways cost ~$25/month and add latency. Peering is free for intra-region traffic and is the right tool when both VNets are in the same region and owned by your organization.

---

## Part 6: Verify What You've Built

### Step 8: Confirm the NSGs Are Associated Correctly
1. Open `app-vnet` → Left sidebar → **Subnets**
2. Click `web-subnet`:
   - Should show `web-nsg` under "Network security group"
3. Click `db-subnet`:
   - Should show `db-nsg` under "Network security group"

### Step 9: Confirm Peering Is Active
1. Open `dev-vnet` → **Peerings**
   - Should show `dev-to-app` with status **Connected**
2. Open `app-vnet` → **Peerings**
   - Should show `app-to-dev` with status **Connected**

### Step 10: Understand What the Network Looks Like
```
app-vnet (10.0.0.0/16)              dev-vnet (10.1.0.0/16)
├─ web-subnet (10.0.1.0/24)  ◄──── Peering ────► dev-subnet (10.1.1.0/24)
│   └─ NSG: HTTP/HTTPS open, SSH from your IP only
└─ db-subnet (10.0.2.0/24)
    └─ NSG: DB port from web-subnet only, everything else denied
```

Traffic from the internet can reach the web subnet on ports 80 and 443.
Traffic from the web subnet can reach the database subnet on the DB port only.
Traffic from the dev VNet can reach both subnets via the peering connection.
No other traffic is allowed in by default (NSGs have an implicit deny-all rule).

---

## Part 7: Cleanup

1. **Resource groups** → `app-network-rg` → **Delete resource group**
2. Confirm with the resource group name → **Delete**

This deletes both VNets, both NSGs, and the peering.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|--------------------------|
| **Subnet segmentation** | Separating tiers lets you apply different firewall rules to each part of the app |
| **NSG rules for web tier** | Public-facing services need HTTP/HTTPS open; SSH should never be open to the world |
| **NSG rules for database tier** | Databases should only accept traffic from the app layer, not directly from the internet |
| **VNet peering** | The right way to connect two Azure environments without a VPN gateway |
| **Verifying associations** | Confirming NSGs are attached to the right subnets before deploying anything into them |

---

## Common Mistakes to Avoid
- **Overlapping IP address spaces**: If `app-vnet` and `dev-vnet` both used `10.0.0.0/16`, peering would fail — plan your address ranges upfront
- **Opening SSH to `Any`**: Bots scan port 22 constantly; always restrict it to known IPs
- **Forgetting to associate the NSG with the subnet**: Creating an NSG without associating it does nothing — the subnet is still wide open
- **Using one subnet for everything**: Without segmentation, you can't apply different security rules to web vs. database tiers
- **Not checking the implicit deny**: NSGs deny all traffic not explicitly allowed — if you forget to add a rule, traffic is silently blocked

---

## Next Steps
- Deploy VMs into `web-subnet` and `db-subnet` and test that the NSG rules work as expected
- Add a Bastion host so you can SSH to VMs without exposing port 22 publicly at all
- Create User Defined Routes (UDRs) to force all outbound traffic through a firewall or proxy
- Set up a VPN gateway if you need to connect this VNet to an on-premises network
