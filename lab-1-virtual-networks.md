# Lab 1: Virtual Networks & Connectivity

## Overview
This lab walks you through creating and configuring Azure Virtual Networks (VNets), subnets, Network Security Groups (NSGs), and VNet peering. You'll build a realistic two-VNet environment with connectivity rules.

**Estimated time**: 45–60 minutes  
**Cost**: ~$0.50–$1 (mostly free resources; tear down at the end)

---

## Objectives
- Create and configure VNets and subnets
- Implement Network Security Groups with inbound/outbound rules
- Test connectivity between VNets using peering
- Understand subnet routing and service endpoints

---

## Part 1: Create First VNet with Subnets

### Step 1: Create VNet "Prod-VNet"
1. Go to **Azure Portal** → search **Virtual networks**
2. Click **+ Create**
3. **Basics tab:**
   - **Subscription**: Your subscription
   - **Resource Group**: Create new → name it `az104-lab1-rg`
   - **Name**: `Prod-VNet`
   - **Region**: `East US` (or closest to you)
4. **IP Addresses tab:**
   - Delete the default `10.0.0.0/16` and set it to `10.0.0.0/16` (it's the same, but you're practicing)
   - **Subnets**: You'll see one default subnet. Delete it and create two:
     - **Subnet 1**: Name: `WebSubnet`, Address range: `10.0.1.0/24`
     - **Subnet 2**: Name: `DbSubnet`, Address range: `10.0.2.0/24`
5. **Security tab**: Leave defaults (will configure NSGs separately)
6. **Review + create** → **Create**

**Validation**: After ~30 seconds, you should see "Deployment successful"

---

## Part 2: Create Second VNet

### Step 2: Create VNet "Dev-VNet"
1. Repeat the VNet creation process:
   - **Name**: `Dev-VNet`
   - **Resource Group**: Same `az104-lab1-rg`
   - **Region**: `East US`
   - **IP address**: `10.1.0.0/16`
   - **Subnets**:
     - `DevSubnet`: `10.1.1.0/24`

**Why separate VNets?** Simulates two environments (Prod vs Dev). Later you'll connect them.

---

## Part 3: Create and Configure NSG for Prod-VNet

### Step 3: Create NSG for Web Tier
1. Search **Network security groups** → **+ Create**
2. **Basics:**
   - **Name**: `WebSubnet-NSG`
   - **Resource Group**: `az104-lab1-rg`
   - **Region**: `East US`
3. **Create**

### Step 4: Add Inbound Rules to WebSubnet-NSG
1. Open the newly created `WebSubnet-NSG`
2. Left sidebar → **Inbound security rules** → **+ Add**
3. **Add inbound security rule #1** (Allow HTTP from Internet):
   - **Source**: `Any`
   - **Source port ranges**: `*`
   - **Destination**: `Any`
   - **Destination port ranges**: `80`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `100`
   - **Name**: `AllowHTTP`
   - Click **Add**

4. **Add inbound security rule #2** (Allow HTTPS from Internet):
   - **Source**: `Any`
   - **Destination port ranges**: `443`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `110`
   - **Name**: `AllowHTTPS`
   - Click **Add**

5. **Add inbound security rule #3** (Allow SSH from your IP):
   - **Source**: `IP Addresses`
   - **Source IP addresses/CIDR ranges**: Enter your public IP (find it via "what's my IP" search)
   - **Destination port ranges**: `22`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `120`
   - **Name**: `AllowSSHFromMyIP`
   - Click **Add**

**Why these rules?** Web tier needs HTTP/HTTPS public access but SSH only from you.

### Step 5: Associate NSG with WebSubnet
1. Still in `WebSubnet-NSG` → Left sidebar → **Subnets** → **+ Associate**
2. **Virtual network**: `Prod-VNet`
3. **Subnet**: `WebSubnet`
4. Click **OK**

### Step 6: Create and Configure NSG for Database Tier
1. Create a second NSG called `DbSubnet-NSG` (repeat Step 3)
2. Add **one inbound rule**:
   - **Source**: `IP Addresses`
   - **Source IP addresses**: `10.0.1.0/24` (traffic from WebSubnet only)
   - **Destination port ranges**: `3306` (MySQL) or `5432` (PostgreSQL)
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `100`
   - **Name**: `AllowDBFromWeb`
3. Associate this NSG with `DbSubnet` in `Prod-VNet`

**Key insight**: Database tier only accepts traffic from web tier—demonstrates least privilege.

---

## Part 4: Set Up VNet Peering

### Step 7: Create Peering Between Prod-VNet and Dev-VNet
1. Open `Prod-VNet` → Left sidebar → **Peerings** → **+ Add**
2. **Peering settings:**
   - **Peering link name** (Prod → Dev): `ProdToDevPeering`
   - **Virtual network**: `Dev-VNet`
   - **Peering link name** (Dev → Prod): `DevToProdPeering`
   - Leave traffic settings as default (allow forwarded traffic, allow gateway transit)
3. Click **Add**

**Validation**: After ~1 minute, both peerings should show **Connected** status.

---

## Part 5: Validation & Testing

### Step 8: Verify Connectivity
1. Go to `Prod-VNet` → **Subnets** → Click `WebSubnet`
   - You should see `WebSubnet-NSG` listed under "Network security group"

2. Go to `Dev-VNet` → **Peerings**
   - You should see `DevToProdPeering` with status **Connected**

3. Check routing:
   - Open `Prod-VNet` → Left sidebar → **Subnets** → Click `WebSubnet`
   - Scroll down → **Route table**: Note it says "None" (using default system routes)

### Step 9: Understand What You've Built
- **Prod-VNet** (10.0.0.0/16): Production environment with web and database tiers
- **Dev-VNet** (10.1.0.0/16): Development environment
- **WebSubnet-NSG**: Allows public HTTP/HTTPS, SSH only from your IP
- **DbSubnet-NSG**: Allows database traffic only from WebSubnet
- **Peering**: Dev-VNet can now reach Prod-VNet and vice versa (traffic flows through Microsoft backbone, not internet)

---

## Part 6: Cleanup (IMPORTANT - Save Your Credits)

1. Go to **Resource groups** → Click `az104-lab1-rg`
2. Click **Delete resource group**
3. Type the resource group name to confirm
4. Click **Delete**

**Wait 2–3 minutes** for all resources to tear down. Check **Notifications** (bell icon) to confirm.

---

## Key Concepts to Understand

| Concept | What It Does |
|---------|-------------|
| **VNet** | Isolated network in Azure; defines IP space and subnets |
| **Subnet** | Divides VNet into smaller ranges; controls routing and NSG scope |
| **NSG** | Stateful firewall; allows/denies traffic at network interface or subnet level |
| **Peering** | Direct, low-latency connection between two VNets without VPN overhead |
| **Inbound vs Outbound** | Inbound = traffic entering; Outbound = traffic leaving (Azure defaults allow all outbound) |

---

## Exam Tips
- **NSGs are stateful**: If you allow inbound on port 443, responses are automatically allowed outbound
- **Peering is unidirectional link names but bidirectional traffic**: Both VNets need peerings (which Azure creates simultaneously)
- **Default deny**: NSGs implicitly deny all traffic unless explicitly allowed
- **Service endpoints** (not covered here) let you secure Azure services (like Storage) to specific subnets without NAT

---

## Next Steps
- Deploy a VM in WebSubnet and test actual SSH/HTTP connectivity
- Add User Defined Routes (UDRs) to force traffic through a Network Virtual Appliance
- Implement Azure Firewall for centralized filtering across VNets
