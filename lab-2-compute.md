# Lab 2: Compute – Virtual Machines & Availability

## Overview
This lab covers creating VMs, configuring availability options (Availability Sets, Availability Zones), managed disks, and basic VM customization. You'll build a scalable, fault-tolerant VM environment.

**Estimated time**: 50–70 minutes  
**Cost**: ~$2–$5 (B-series VMs are cheap; tear down at the end)

---

## Objectives
- Create and configure Windows and Linux VMs
- Understand Availability Sets vs Availability Zones
- Configure managed disks and disk encryption
- Use custom script extensions for VM initialization
- Manage VM sizing and pricing

---

## Part 1: Create Availability Set

### Step 1: Create an Availability Set
1. Go to **Azure Portal** → search **Availability sets** → **+ Create**
2. **Basics:**
   - **Resource Group**: Create new → `az104-lab2-rg`
   - **Name**: `WebServers-AvSet`
   - **Region**: `East US`
   - **Fault domains**: `2` (default; how many independent power/hardware failures the set can survive)
   - **Update domains**: `5` (default; how Azure staggers maintenance updates)
3. **Review + create** → **Create**

**Why this matters**: Availability Sets ensure your VMs are distributed across different fault domains. If one rack fails, your app still runs.

---

## Part 2: Create First VM

### Step 2: Create Linux VM #1
1. Search **Virtual machines** → **+ Create** → **Azure virtual machine**
2. **Basics tab:**
   - **Resource Group**: `az104-lab2-rg`
   - **Virtual machine name**: `web-vm-01`
   - **Region**: `East US`
   - **Image**: `Ubuntu Server 22.04 LTS - x64 Gen2`
   - **VM architecture**: `x64`
   - **Size**: Click **See all sizes** → search `B1s` → select it
     - *Why B1s?* Cheapest compute option, perfect for learning
   - **Authentication type**: `SSH public key`
   - **Username**: `azureuser`
   - **SSH public key source**: `Generate new key pair`
   - **Key pair name**: `web-vm-01-key`

3. **Disk tab:**
   - **OS disk type**: `Premium SSD` (leave as is; the default)
   - Leave encryption at default (unencrypted for this lab)

4. **Networking tab:**
   - **Virtual network**: `Create new` → name it `Lab2-VNet`, address space `10.0.0.0/16`
   - **Subnet**: `Create new` → name it `VmSubnet`, address space `10.0.1.0/24`
   - **Public IP**: Leave as default (auto-creates one for SSH access)
   - **Network security group**: `Create new` → name it `VM-NSG`
     - You'll configure this in a moment

5. **Management tab:**
   - **Monitoring**: Leave defaults (can enable later if you want)

6. **Advanced tab:**
   - Leave defaults

7. **Review + create** → **Create**

8. **Key pair generation popup**: Click **Download private key and create resource**
   - This downloads `web-vm-01-key.pem` — **keep this safe; you'll need it to SSH**

**Wait ~3–5 minutes** for VM deployment.

---

## Part 3: Configure VM NSG and SSH Access

### Step 3: Configure VM-NSG to Allow SSH
1. Search **Network security groups** → Open `VM-NSG`
2. **Inbound security rules** → **+ Add**
3. **Add inbound rule:**
   - **Source**: `IP Addresses`
   - **Source IP addresses**: Your public IP (find via "what's my IP")
   - **Destination port ranges**: `22`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `100`
   - **Name**: `AllowSSHFromMyIP`
4. Click **Add**

### Step 4: Test SSH Connectivity (Optional But Recommended)
1. Go back to `web-vm-01` → Copy its **Public IP address**
2. Open terminal/PowerShell on your machine
3. Run:
   ```bash
   ssh -i path/to/web-vm-01-key.pem azureuser@<PUBLIC_IP>
   ```
4. Type `yes` when prompted about RSA key fingerprint
5. If successful, you're in the VM. Type `exit` to leave.

**This validates**: Public IP, NSG rule, and SSH key are all working.

---

## Part 4: Create Second VM with Custom Script

### Step 5: Create Linux VM #2 (In Same Availability Set)
1. **Virtual machines** → **+ Create** → **Azure virtual machine**
2. **Basics tab:**
   - **Resource Group**: `az104-lab2-rg`
   - **Virtual machine name**: `web-vm-02`
   - **Region**: `East US`
   - **Image**: `Ubuntu Server 22.04 LTS - x64 Gen2`
   - **Size**: `B1s`
   - **Authentication type**: `SSH public key`
   - **Username**: `azureuser`
   - **SSH public key source**: `Generate new key pair`
   - **Key pair name**: `web-vm-02-key`

3. **Disk tab**: Leave defaults

4. **Networking tab:**
   - **Virtual network**: `Lab2-VNet` (the one you created earlier)
   - **Subnet**: `VmSubnet`
   - **Public IP**: Auto-create
   - **Network security group**: `Existing` → select `VM-NSG`

5. **Management tab:** Leave defaults

6. **Advanced tab:**
   - **Custom data**: Paste this (it runs on first boot):
     ```bash
     #!/bin/bash
     apt-get update
     apt-get install -y nginx
     systemctl start nginx
     ```
     *This installs and starts Nginx, so you can test web access later*

7. **Review + create** → **Create**

8. Download the private key again

**Wait ~3–5 minutes**

---

## Part 5: Move VM to Availability Set (Portal Workaround)

**Note**: Azure doesn't let you add an existing VM to an Availability Set in the portal. This step shows you *why* you need to plan ahead.

1. Open `web-vm-02` → **Overview** tab
2. Scroll down → Look for **Availability options**: Shows "No infrastructure redundancy required"

**Key learning**: You would have needed to specify the Availability Set *during creation*. For this lab, you've learned the workflow; in production, you'd plan this upfront.

---

## Part 6: Create Windows VM (Different Approach)

### Step 6: Create Windows VM
1. **Virtual machines** → **+ Create** → **Azure virtual machine**
2. **Basics tab:**
   - **Resource Group**: `az104-lab2-rg`
   - **Virtual machine name**: `admin-vm-01`
   - **Region**: `East US`
   - **Image**: `Windows Server 2022 Datacenter - x64 Gen2`
   - **Size**: `B2s` (Windows needs more resources)
   - **Authentication type**: `Password`
   - **Username**: `azureadmin`
   - **Password**: Create a strong one (min 12 chars, uppercase, lowercase, number, special char)
     - Example: `P@ssw0rd!Lab2024`

3. **Disk tab**: Leave defaults (Windows needs Premium SSD)

4. **Networking tab:**
   - **Virtual network**: `Lab2-VNet`
   - **Subnet**: `VmSubnet`
   - **Public IP**: Auto-create
   - **Network security group**: `Existing` → select `VM-NSG`

5. **Management tab**: Leave defaults

6. **Review + create** → **Create**

**Wait ~5–8 minutes** (Windows takes longer)

---

## Part 7: Configure RDP Access

### Step 7: Add RDP Rule to VM-NSG
1. Open `VM-NSG` → **Inbound security rules** → **+ Add**
2. **Add inbound rule:**
   - **Source**: `IP Addresses`
   - **Source IP addresses**: Your public IP
   - **Destination port ranges**: `3389`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `110`
   - **Name**: `AllowRDPFromMyIP`
3. Click **Add**

### Step 8: Connect via RDP (Optional)
1. Open `admin-vm-01` → Copy **Public IP**
2. On your machine:
   - **Windows**: Press `Win + R` → type `mstsc` → enter the IP
   - **Mac/Linux**: Use Remote Desktop Connection app or `rdesktop`
3. Username: `azureadmin`, Password: the one you created

---

## Part 8: Understand Disk Management

### Step 9: Inspect VM Disks
1. Open `web-vm-01` → **Disks** (left sidebar)
2. You'll see:
   - **OS disk**: The boot disk (e.g., `web-vm-01_OsDisk_1...`)
   - **Data disk**: None (you didn't add any)

### Step 10: Create and Attach a Data Disk
1. Search **Disks** → **+ Create**
2. **Create managed disk:**
   - **Resource Group**: `az104-lab2-rg`
   - **Disk name**: `web-vm-01-data-disk`
   - **Region**: `East US`
   - **Availability zone**: Leave as default
   - **Disk SKU**: `Premium SSD` (or `Standard SSD` to save money)
   - **Source type**: `None` (blank disk)
   - **Size**: `32 GiB`
3. **Create**

### Step 11: Attach Disk to VM
1. After creation, open the new disk → **Overview**
2. Click **Attach to VM** at the top
3. **Attach managed disk:**
   - **Virtual machine**: `web-vm-01`
   - **LUN** (Logical Unit Number): `0`
4. Click **Attach**

**What you've learned**: Managed disks are independent resources you can attach/detach from VMs. In production, you'd use these for databases, application data, etc.

---

## Part 9: Validation & Testing

### Step 12: Verify Everything
1. Go to **Virtual machines** dashboard
   - You should see three VMs: `web-vm-01`, `web-vm-02`, `admin-vm-01`
   - All should have a public IP

2. Check `web-vm-01` → **Disks**
   - Should list both OS disk and the data disk you attached

3. Check `web-vm-02` → **Extensions**
   - Should show the Nginx installation ran (Custom Script Extension)

4. (Optional) SSH into `web-vm-02` and run:
   ```bash
   curl http://localhost
   ```
   You should see the Nginx welcome page HTML

---

## Part 10: Cleanup

1. Go to **Resource groups** → Click `az104-lab2-rg`
2. Click **Delete resource group**
3. Type the resource group name
4. Click **Delete**

**Wait 3–5 minutes** for all VMs and disks to tear down.

---

## Key Concepts to Understand

| Concept | What It Does |
|---------|-------------|
| **Availability Set** | Spreads VMs across fault domains to survive hardware failures |
| **Availability Zone** | Spreads VMs across physically separate datacenters within a region |
| **Managed Disk** | Azure handles the underlying storage account; you just reference it by name |
| **Custom Data** | Script that runs once at first boot (via User Data/Cloud-Init) |
| **Public IP** | Allows inbound internet traffic to the VM |
| **SSH vs RDP** | SSH for Linux, RDP for Windows remote access |

---

## Exam Tips
- **Availability Set planning**: You set this at VM creation; you can't add an existing VM to an Availability Set
- **Zone-redundancy**: If you use Availability Zones (Premium disk + Zone-redundant storage), you get 99.99% SLA
- **VM sizing**: B-series is burstable (good for dev/test), D-series for sustained workloads
- **Custom data is one-time**: Changes after VM creation won't re-run; use VM extensions or automation for updates
- **Public IPs are billable**: If you're not using them, delete them to save ~$3/month per IP

---

## Next Steps
- Deploy a Load Balancer in front of multiple VMs
- Use Azure Automation to start/stop VMs on schedule
- Create a VM image from `web-vm-01` (generalize it, then capture)
- Implement disk encryption (Azure Disk Encryption)
