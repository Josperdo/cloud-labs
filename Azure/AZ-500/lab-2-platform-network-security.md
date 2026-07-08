# Lab 2: Platform & Network Security

Check box if done: [ ]

## Overview
Most Azure network security questions boil down to one principle: nothing should be reachable that doesn't need to be, and nothing should need a public IP if there's another way in. This lab builds a two-tier network with a deny-by-default posture, replaces direct public-IP admin access with Azure Bastion, and locks a storage account down to a Private Endpoint so its data plane traffic never touches the public internet.

**Estimated time**: 60–90 minutes
**Cost**: ~$1–$3 (Azure Bastion Basic SKU bills ~$0.19/hr — deploy it, do the lab, delete it the same session)

---

## Scenario
You're securing a two-tier app: a web tier and a data tier that should never be reachable from the internet directly. Right now, the previous engineer's habit was to slap a public IP on every VM "to make it easier to RDP/SSH in." You're removing every unnecessary public IP, tightening the NSGs to deny-by-default, and proving the data tier's storage account is unreachable from anywhere except inside the VNet.

---

## Objectives
- Design a VNet with tiered subnets and NSGs that deny by default
- Deploy Azure Bastion for admin access without exposing any VM to the public internet
- Lock a storage account to a Private Endpoint and disable public network access
- Verify the negative: prove that direct/public access actually fails

---

## Part 1: Build the Network Foundation

### Step 1: Create the Resource Group and VNet

```bash
az group create --name az500-lab2-rg --location eastus2

az network vnet create \
  --resource-group az500-lab2-rg \
  --name az500-vnet \
  --address-prefix 10.20.0.0/16 \
  --subnet-name web-subnet \
  --subnet-prefix 10.20.1.0/24

az network vnet subnet create \
  --resource-group az500-lab2-rg \
  --vnet-name az500-vnet \
  --name data-subnet \
  --address-prefixes 10.20.2.0/24

# Bastion requires its own dedicated subnet named exactly AzureBastionSubnet, /26 or larger
az network vnet subnet create \
  --resource-group az500-lab2-rg \
  --vnet-name az500-vnet \
  --name AzureBastionSubnet \
  --address-prefixes 10.20.255.0/26
```

### Step 2: Create Deny-by-Default NSGs

```bash
az network nsg create --resource-group az500-lab2-rg --name web-nsg
az network nsg create --resource-group az500-lab2-rg --name data-nsg
```

Azure NSGs already deny all inbound internet traffic by default via the implicit `DenyAllInBound` rule (priority 65500) — but explicit rules make intent visible and are what an AZ-500 exam question (and a security review) expects to see.

**Web NSG — allow only HTTPS from the internet:**
```bash
az network nsg rule create \
  --resource-group az500-lab2-rg --nsg-name web-nsg \
  --name AllowHTTPS --priority 100 \
  --direction Inbound --access Allow --protocol Tcp \
  --destination-port-ranges 443 --source-address-prefixes Internet
```

**Data NSG — allow only from the web subnet, on the app port only:**
```bash
az network nsg rule create \
  --resource-group az500-lab2-rg --nsg-name data-nsg \
  --name AllowFromWebTier --priority 100 \
  --direction Inbound --access Allow --protocol Tcp \
  --destination-port-ranges 8080 \
  --source-address-prefixes 10.20.1.0/24
```

Notice there's no SSH/RDP rule on either NSG. That's intentional — admin access comes through Bastion in Part 2, not through a port opened on the NSG.

### Step 3: Attach NSGs to Subnets

```bash
az network vnet subnet update \
  --resource-group az500-lab2-rg --vnet-name az500-vnet \
  --name web-subnet --network-security-group web-nsg

az network vnet subnet update \
  --resource-group az500-lab2-rg --vnet-name az500-vnet \
  --name data-subnet --network-security-group data-nsg
```

---

## Part 2: Azure Bastion — Admin Access Without Public IPs

### Step 4: Why Bastion Instead of a Public IP + NSG Rule
A public IP with an NSG rule allowing SSH/RDP "only from my IP" still means: the VM has a public IP, that IP is scanned by the entire internet constantly, and the NSG rule breaks the moment your IP changes (home ISP, coffee shop, VPN). Bastion puts the VM's management interface entirely inside the VNet — you connect through the Azure portal over TLS, and the VM never has a public IP at all.

### Step 5: Deploy Bastion

```bash
az network public-ip create \
  --resource-group az500-lab2-rg \
  --name bastion-pip \
  --sku Standard

az network bastion create \
  --resource-group az500-lab2-rg \
  --name az500-bastion \
  --public-ip-address bastion-pip \
  --vnet-name az500-vnet \
  --location eastus2 \
  --sku Basic
```

> Bastion takes 5–10 minutes to provision. Move on to Step 6 while it deploys.

### Step 6: Deploy a VM in the Data Subnet — No Public IP

```bash
az vm create \
  --resource-group az500-lab2-rg \
  --name data-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --vnet-name az500-vnet \
  --subnet data-subnet \
  --nsg "" \
  --public-ip-address "" \
  --admin-username labadmin \
  --generate-ssh-keys
```

**Validation checkpoint**: `az vm show -d -g az500-lab2-rg -n data-vm --query publicIps -o tsv` should return **nothing**. There is no public IP on this VM — try to `ssh` to it directly from your machine and confirm it's unreachable (there's no address to even attempt).

### Step 7: Connect via Bastion
1. Portal → **Virtual machines** → **data-vm** → **Connect** → **Bastion**
2. Choose **SSH**, authentication type **SSH Private Key from Local File**, upload the private key generated in Step 6 (`~/.ssh/id_rsa` by default)
3. Click **Connect** — a browser-based SSH session opens

**What just happened**: your traffic went browser → Azure portal (TLS) → Bastion host (inside the VNet) → target VM's private IP. At no point was the VM's management port exposed to the internet.

---

## Part 3: Private Endpoint — Locking Down a Storage Account

### Step 8: Create a Storage Account with Public Access Disabled

```bash
az storage account create \
  --name az500lab2$RANDOM \
  --resource-group az500-lab2-rg \
  --location eastus2 \
  --sku Standard_LRS \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --public-network-access Disabled
```

> Save the generated storage account name — you'll need it in the next steps. Retrieve it anytime with:
> `az storage account list -g az500-lab2-rg --query "[].name" -o tsv`

With `--public-network-access Disabled`, this storage account is unreachable from the public internet **at all** — not even with the account key. The only way in is a Private Endpoint.

### Step 9: Create a Private Endpoint in the Data Subnet

```bash
STORAGE_ACCOUNT=$(az storage account list -g az500-lab2-rg --query "[0].name" -o tsv)
STORAGE_ID=$(az storage account show -g az500-lab2-rg -n $STORAGE_ACCOUNT --query id -o tsv)

az network private-endpoint create \
  --resource-group az500-lab2-rg \
  --name storage-pe \
  --vnet-name az500-vnet \
  --subnet data-subnet \
  --private-connection-resource-id $STORAGE_ID \
  --group-id blob \
  --connection-name storage-pe-connection
```

### Step 10: Create a Private DNS Zone So Names Resolve Correctly
Without this, the storage account's normal hostname (`<account>.blob.core.windows.net`) still resolves to its public IP — which now rejects connections. The private DNS zone overrides that resolution for resources inside the VNet.

```bash
az network private-dns zone create \
  --resource-group az500-lab2-rg \
  --name "privatelink.blob.core.windows.net"

az network private-dns link vnet create \
  --resource-group az500-lab2-rg \
  --zone-name "privatelink.blob.core.windows.net" \
  --name storage-dns-link \
  --virtual-network az500-vnet \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group az500-lab2-rg \
  --endpoint-name storage-pe \
  --name storage-dns-zone-group \
  --private-dns-zone "privatelink.blob.core.windows.net" \
  --zone-name blob
```

### Step 11: Verify from Inside the VNet (via Bastion) — Should Succeed

From the Bastion session on `data-vm`:

```bash
nslookup $STORAGE_ACCOUNT.blob.core.windows.net
# Expect: resolves to a 10.20.2.x private address, not a public IP

curl -sI https://$STORAGE_ACCOUNT.blob.core.windows.net/
# Expect: an HTTP response (403 is fine — it confirms you reached the service; a connection timeout means the private endpoint or DNS isn't wired up correctly)
```

### Step 12: Verify from Your Local Machine — Should Fail
From your own terminal (outside the VNet):

```bash
curl -sI --max-time 5 https://$STORAGE_ACCOUNT.blob.core.windows.net/
```

**Expected result**: timeout or connection refused. This is the proof that matters — the storage account is genuinely unreachable from the public internet, not just "unreachable unless you have the key."

---

## Part 4: Cleanup

```bash
az group delete --name az500-lab2-rg --yes --no-wait
```

Confirm Bastion in particular is gone — it's the most expensive resource in this lab and bills hourly until deleted:
```bash
az network bastion list --resource-group az500-lab2-rg
```

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|---------------------------|
| **Deny-by-default NSGs with explicit narrow allows** | Minimizes blast radius; nothing is reachable "by accident" |
| **Azure Bastion instead of public IP + NSG rule** | Removes the VM's management surface from the internet entirely |
| **Private Endpoint + private DNS zone** | Storage/PaaS data plane traffic never leaves the VNet, even with valid credentials |
| **`--public-network-access Disabled`** | A stronger control than firewall rules — the resource has no public listener at all |
| **Verifying the negative (access fails from outside)** | Proves the control works instead of assuming it does |

---

## Common Mistakes to Avoid
- **Creating the Private Endpoint but skipping the private DNS zone**: the hostname still resolves publicly, and connections fail — this is the most common private endpoint misconfiguration
- **Leaving `--public-network-access Enabled` and relying only on firewall rules**: firewall rules can be misconfigured; disabling public access entirely removes that failure mode
- **Opening SSH/RDP "temporarily" for troubleshooting and forgetting to close it**: use Bastion or Just-In-Time VM access instead so there's never a standing open port
- **Testing only the "should work" case**: always verify the "should fail" case too (Step 12) — a control that isn't proven to block anything might not be blocking anything

---

## Next Steps
- Add Azure Firewall as a centralized egress point and force all outbound traffic through it via User Defined Routes (cost note: Azure Firewall bills continuously, ~$1.25/hr — deploy for a short test window only)
- Enable Just-In-Time VM access in Defender for Cloud as an alternative/complement to Bastion for environments where Bastion isn't available
- Extend this pattern to Key Vault and SQL Database — both support Private Endpoints the same way
- Continue to [Lab 3: Data & Application Security](lab-3-data-app-security.md) to secure secrets and encryption keys for this same environment
