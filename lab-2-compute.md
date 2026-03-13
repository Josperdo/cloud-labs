# Lab 2: Compute — Deploying Web Servers with Safe Remote Access

## Overview
Spinning up a VM is straightforward. Doing it correctly — with proper network segmentation, locked-down remote access, an app installed on first boot, and persistent storage for application data — takes a bit more thought. This lab simulates the kind of compute deployment an entry-level cloud engineer would be asked to set up for a new application.

**Estimated time**: 50–65 minutes
**Cost**: ~$2–$5 (tear down at the end)

> **Free Trial Note**: This lab uses `Standard_DS1_v2` (1 vCPU) for Linux VMs, which is typically available without a quota increase. If that size is unavailable in your region, open **See all sizes**, filter by 1–2 vCPUs, and pick the cheapest available option. Free trial accounts are capped at 4 vCPUs per region.

---

## Scenario
Your team needs two web servers deployed for a new application — one is the primary, and the second ensures the app stays up if the first needs maintenance or has a hardware issue. Both need nginx installed and ready to serve traffic. SSH access should be locked down to your IP only. The app also needs a separate disk for storing uploaded files so that data isn't mixed with the OS disk.

---

## Objectives
- Deploy Linux VMs into an existing VNet with proper networking
- Lock down SSH access to a specific IP address
- Use custom data to install and start an application on first boot
- Attach a managed data disk to separate application data from the OS
- Understand availability options so VMs survive hardware failures
- Know how to stop VMs to avoid unnecessary charges

---

## Part 1: Set Up the Network

### Step 1: Create a VNet for the Application
If you completed Lab 1, this pattern is familiar. VMs go inside a VNet — you can't put them on the internet directly.

1. Search **Virtual networks** → **+ Create**
2. **Basics tab:**
   - **Resource Group**: Create new → `webservers-rg`
   - **Name**: `app-vnet`
   - **Region**: `East US`
3. **IP Addresses tab:**
   - Address space: `10.0.0.0/16`
   - Add a subnet: Name: `web-subnet`, Range: `10.0.1.0/24`
4. **Review + create** → **Create**

---

## Part 2: Deploy the First Web Server

### Step 2: Create the First Linux VM
1. Search **Virtual machines** → **+ Create** → **Azure virtual machine**
2. **Basics tab:**
   - **Resource Group**: `webservers-rg`
   - **Virtual machine name**: `web-vm-01`
   - **Region**: `East US`
   - **Availability options**: `Availability zone`
   - **Zone**: `1`
     *(You'll deploy the second VM in Zone 2 — this way, a datacenter-level failure doesn't take both servers down at once)*
   - **Image**: `Ubuntu Server 22.04 LTS - x64 Gen2`
   - **Size**: `Standard_DS1_v2` (1 vCPU / 3.5 GB RAM)
   - **Authentication type**: `SSH public key`
   - **Username**: `azureuser`
   - **SSH public key source**: `Generate new key pair`
   - **Key pair name**: `web-vm-01-key`

3. **Disks tab:**
   - **OS disk type**: `Standard SSD` (sufficient for a web server; save Premium SSD for databases)

4. **Networking tab:**
   - **Virtual network**: `app-vnet`
   - **Subnet**: `web-subnet`
   - **Public IP**: Leave as default (auto-creates one; needed for SSH and web access in this lab)
   - **NIC network security group**: `Advanced` → **Create new**
     - Name it `web-vm-nsg`
     - You'll configure rules after creation — remove the default RDP/SSH rules for now if any appear, since you'll set them up explicitly

5. **Advanced tab:**
   - **Custom data**: Paste the following script
     ```bash
     #!/bin/bash
     apt-get update -y
     apt-get install -y nginx
     systemctl enable nginx
     systemctl start nginx
     echo "web-vm-01 is running" > /var/www/html/index.html
     ```
     *This runs once on first boot. By the time the VM is ready, nginx will already be installed and serving traffic.*

6. **Review + create** → **Create**
7. When prompted: **Download private key and create resource**
   - Save `web-vm-01-key.pem` somewhere secure on your machine

**Wait ~3–5 minutes** for deployment.

---

## Part 3: Lock Down SSH Access

### Step 3: Configure the NSG to Allow SSH Only From Your IP
The VM was created with a network security group. Right now it may have no rules (or overly permissive defaults). You need to explicitly allow SSH from your IP and HTTP/HTTPS from the internet.

1. Search **Network security groups** → Open `web-vm-nsg`
2. **Inbound security rules** → **+ Add**

3. **Rule 1 — Allow SSH from your IP only:**
   - **Source**: `IP Addresses`
   - **Source IP**: Your public IP (search "what's my IP")
   - **Destination port**: `22`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `100`
   - **Name**: `AllowSSH`
   - Click **Add**

4. **Rule 2 — Allow HTTP:**
   - **Source**: `Any`
   - **Destination port**: `80`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `110`
   - **Name**: `AllowHTTP`
   - Click **Add**

5. **Rule 3 — Allow HTTPS:**
   - **Source**: `Any`
   - **Destination port**: `443`
   - **Protocol**: `TCP`
   - **Action**: `Allow`
   - **Priority**: `120`
   - **Name**: `AllowHTTPS`
   - Click **Add**

**Why these priorities?** Rules are evaluated from lowest number to highest. Lower number = higher priority. Leave gaps between priorities (100, 110, 120) so you can insert rules later without renumbering.

### Step 4: Test SSH Access
1. Open `web-vm-01` → Copy the **Public IP address**
2. Open a terminal on your machine and run:
   ```bash
   chmod 400 /path/to/web-vm-01-key.pem
   ssh -i /path/to/web-vm-01-key.pem azureuser@<PUBLIC_IP>
   ```
3. Type `yes` when prompted about the fingerprint
4. Once connected, verify nginx is running:
   ```bash
   curl http://localhost
   ```
   You should see `web-vm-01 is running`
5. Type `exit` to disconnect

**If SSH fails**: Double-check the NSG rule source IP matches your current public IP, and confirm the NSG is associated with the VM's network interface or subnet.

---

## Part 4: Deploy the Second Web Server

### Step 5: Create the Second VM in a Different Availability Zone
The second VM is almost identical to the first, but placed in Zone 2. If the hardware hosting Zone 1 fails, Zone 2 is unaffected.

1. **Virtual machines** → **+ Create** → **Azure virtual machine**
2. **Basics tab:**
   - **Resource Group**: `webservers-rg`
   - **Virtual machine name**: `web-vm-02`
   - **Region**: `East US`
   - **Availability options**: `Availability zone`
   - **Zone**: `2`
   - **Image**: `Ubuntu Server 22.04 LTS - x64 Gen2`
   - **Size**: `Standard_DS1_v2`
   - **Authentication type**: `SSH public key`
   - **Username**: `azureuser`
   - **SSH public key source**: `Generate new key pair`
   - **Key pair name**: `web-vm-02-key`

3. **Networking tab:**
   - **Virtual network**: `app-vnet`
   - **Subnet**: `web-subnet`
   - **Public IP**: Auto-create
   - **NIC network security group**: `Advanced` → Select existing → `web-vm-nsg`
     *(Both VMs share the same NSG so you manage rules in one place)*

4. **Advanced tab:**
   - **Custom data**:
     ```bash
     #!/bin/bash
     apt-get update -y
     apt-get install -y nginx
     systemctl enable nginx
     systemctl start nginx
     echo "web-vm-02 is running" > /var/www/html/index.html
     ```

5. **Review + create** → **Create**
6. Download the `web-vm-02-key.pem` file

**Wait ~3–5 minutes**

**Why two availability zones instead of two VMs in the same zone?** Availability zones are physically separate datacenters within the same Azure region. If there's a power failure or cooling issue in one zone, the other is unaffected. Two VMs in the same zone don't give you that protection.

---

## Part 5: Add a Data Disk for Application Storage

### Step 6: Create a Managed Disk
Application data — uploaded files, logs, local caches — should not live on the OS disk. If you ever need to replace or resize the OS, you'd lose that data. A separate data disk can be detached from one VM and reattached to another.

1. Search **Disks** → **+ Create**
2. **Basics:**
   - **Resource Group**: `webservers-rg`
   - **Disk name**: `web-vm-01-data`
   - **Region**: `East US`
   - **Availability zone**: `1` (must match the VM it'll attach to)
   - **Disk SKU**: `Standard SSD`
   - **Source type**: `None` (blank disk)
   - **Size**: `32 GiB`
3. Click **Review + create** → **Create**

### Step 7: Attach the Disk to the VM
1. Open `web-vm-01` → Left sidebar → **Disks**
2. Click **+ Attach existing disks** or **Add data disk**
3. Select `web-vm-01-data`
4. Click **Apply**

### Step 8: Mount the Disk Inside the VM (Optional but Complete)
The disk is attached at the Azure level but needs to be formatted and mounted inside the OS before the app can use it.

1. SSH into `web-vm-01`:
   ```bash
   ssh -i web-vm-01-key.pem azureuser@<PUBLIC_IP>
   ```
2. Find the new disk (it'll show up as `sdc` or similar):
   ```bash
   lsblk
   ```
3. Format and mount it:
   ```bash
   sudo mkfs.ext4 /dev/sdc
   sudo mkdir /data
   sudo mount /dev/sdc /data
   ```
4. Make the mount persist across reboots:
   ```bash
   echo '/dev/sdc /data ext4 defaults 0 2' | sudo tee -a /etc/fstab
   ```
5. Verify:
   ```bash
   df -h /data
   ```

---

## Part 6: Understand Availability and Cost

### Step 9: Review What Protects Your VMs
1. Go to **Virtual machines** dashboard
2. Open `web-vm-01` → **Overview**
   - Look for **Availability zone**: Should show `1`
3. Open `web-vm-02` → **Overview**
   - Should show zone `2`

If Azure has an issue in Zone 1, web-vm-02 in Zone 2 is still running. A load balancer (not in this lab, but a natural next step) would route traffic to whichever VMs are healthy.

### Step 10: Stop VMs When Not in Use
VMs cost money while running, even if they're idle. In dev and test environments, stopping them when you're done saves significant money.

1. Open `web-vm-01` → Click **Stop** at the top
2. Confirm the stop
3. Repeat for `web-vm-02`

When a VM is stopped (deallocated), you're not charged for compute. You still pay a small amount for the managed disks.

**In a real environment**: You'd automate this with Azure Automation or start/stop schedules so dev VMs aren't running overnight or on weekends.

---

## Part 7: Cleanup

1. **Resource groups** → `webservers-rg` → **Delete resource group**
2. Confirm with the resource group name → **Delete**

This deletes both VMs, both disks, the VNet, NSG, and public IPs.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|--------------------------|
| **Availability zones** | Deploying across zones means a datacenter issue doesn't take your whole app down |
| **SSH key authentication** | Password auth on a public VM is a security risk; SSH keys are the standard |
| **NSG rules scoped to your IP** | Open SSH ports get brute-forced within minutes of deployment |
| **Custom data (cloud-init)** | Automated app installation on first boot instead of manual SSH setup |
| **Managed data disk** | Keeps application data independent from the OS so you can replace the VM without data loss |
| **Stopping VMs when idle** | Dev/test VMs running 24/7 waste budget; stop them when you're done |

---

## Common Mistakes to Avoid
- **Opening SSH to `Any` in the NSG**: Port 22 is constantly scanned; always restrict to known IPs
- **Putting app data on the OS disk**: If you resize or recreate the VM, you'll lose everything on the OS disk
- **Both VMs in the same availability zone**: They look redundant but aren't — a single zone failure takes both down
- **Using Premium SSD everywhere**: Premium SSD costs 3–4x more than Standard SSD; use it only where IOPS performance matters (databases, not web servers)
- **Forgetting to deallocate VMs**: "Stopped" in the OS doesn't stop billing; you must deallocate from the Azure portal to stop compute charges
- **Hardcoding the VM's public IP**: Public IPs can change; use DNS names or a load balancer in front of VMs for stable addressing

---

## Next Steps
- Deploy a Load Balancer in front of both web VMs to distribute traffic and enable health checks
- Set up Azure Bastion so you can SSH without exposing port 22 publicly at all
- Create a VM image from `web-vm-01` so future deployments start with nginx pre-installed
- Use VM Scale Sets to automatically add or remove VMs based on CPU load
- Set up auto-shutdown schedules on dev VMs to avoid overnight charges
