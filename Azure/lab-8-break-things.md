# Lab 8: Break Things — Chaos Engineering on Your Own Infrastructure

Check box if done: []

## Overview
The best way to understand how Azure protects you — and where it doesn't — is to intentionally break things in a controlled environment. This lab deploys a small, cheap environment and then systematically attacks its own controls: misconfigured NSGs, deleted dependencies, bypassed locks, revoked access, and forced Terraform state conflicts. Each break teaches you something a real incident would have taught you the hard way.

**Estimated time**: 60–90 minutes
**Cost**: ~$1–3 (one B1s VM for ~1 hour + storage account; everything gets torn down at the end)

> **Note**: This lab is intentionally destructive. That's the point. Every "break" is followed by a fix and an explanation of what a real attacker or misconfiguration could do with that gap.

---

## Scenario
You've built the infrastructure. Now you're wearing the red team hat for a day. Your job: find the weaknesses in your own environment before someone else does. You'll test management-plane protections, data-plane access controls, network rules, identity, and infrastructure-as-code failure modes — then document what you found.

---

## Objectives
- Apply and verify a resource lock — then watch a delete attempt fail
- Misconfigure an NSG rule and observe the traffic it breaks
- Break RBAC by scoping a permission wrong, then fix it
- Test storage account access controls (public access, SAS tokens, key rotation)
- Force a Terraform state conflict and recover from it
- Practice the "blast radius" mindset: know what each failure actually affects

---

## Setup: Deploy the Lab Environment

Before you can break things, you need something to break. This creates one resource group with a storage account, a VNet with one subnet, and a single small VM.

### Step 1: Create the Resource Group and Storage Account

```bash
az group create --name break-lab-rg --location eastus2

az storage account create \
  --name breaklab$RANDOM \
  --resource-group break-lab-rg \
  --location eastus2 \
  --sku Standard_LRS \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false
```

> **Save the storage account name** — the `$RANDOM` suffix makes it unique. Run `az storage account list -g break-lab-rg --query "[].name" -o tsv` to get it back if you forget.

### Step 2: Create the VNet and VM

```bash
az network vnet create \
  --resource-group break-lab-rg \
  --name break-vnet \
  --address-prefix 10.99.0.0/16 \
  --subnet-name break-subnet \
  --subnet-prefix 10.99.1.0/24

az network nsg create \
  --resource-group break-lab-rg \
  --name break-nsg

az network nsg rule create \
  --resource-group break-lab-rg \
  --nsg-name break-nsg \
  --name AllowSSH \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 22

az network vnet subnet update \
  --resource-group break-lab-rg \
  --vnet-name break-vnet \
  --name break-subnet \
  --network-security-group break-nsg

az vm create \
  --resource-group break-lab-rg \
  --name break-vm \
  --image Ubuntu2204 \
  --size Standard_D2als_v7 \
  --vnet-name break-vnet \
  --subnet break-subnet \
  --nsg "" \
  --admin-username labadmin \
  --generate-ssh-keys \
  --no-wait
```

> The VM takes 2–3 minutes to provision. While it's deploying, move on to Part 1 — you won't need the VM until Part 2.

---

## Part 1: Resource Locks — The Management Plane Seatbelt

Resource locks prevent deletion or modification at the management plane level, regardless of RBAC. Even subscription owners can't delete a locked resource without first removing the lock. This is your last line of defense against runaway automation or accidental `az group delete`.

### Step 1: Apply a Delete Lock

```bash
az lock create \
  --name "do-not-delete" \
  --resource-group break-lab-rg \
  --lock-type CanNotDelete \
  --notes "Lab 8 — testing lock enforcement"
```

### Step 2: Try to Delete the Resource Group

```bash
az group delete --name break-lab-rg --yes
```

**Expected result**: The command will fail with a `ScopeLocked` error. Azure refuses the operation — not because of permissions, but because of the lock. Write down the exact error message you see.

**What this protects against in the real world:**
- Automation scripts running `az group delete` on the wrong resource group (wrong variable, wrong subscription)
- A new team member with Owner permissions who doesn't know the resources are production
- Terraform `destroy` targeting the wrong workspace

### Step 3: Remove the Lock and Confirm It Works

```bash
# Get the lock ID
az lock list --resource-group break-lab-rg --query "[].id" -o tsv

# Delete the lock by ID
az lock delete --ids <lock-id-from-above>

# Now try deleting a resource (not the whole group — you still need it)
# Delete a test object inside the group to confirm the lock is gone
az storage account list -g break-lab-rg --query "[].name" -o tsv
```

**Reflection question**: If a CanNotDelete lock is applied at the resource group level, what happens to resources inside the group? Does each resource also become locked, or just the group itself? Test your assumption by trying to delete the storage account while the lock is on the group.

---

## Part 2: NSG Misconfiguration — Breaking Your Own Network

NSG rules have priority numbers. Lower numbers win. A single misconfigured deny rule in the wrong position can silently block all traffic — or an Allow-All rule in the wrong position can open everything up.

### Step 1: Get the VM's Public IP

```bash
az vm show -d -g break-lab-rg -n break-vm --query publicIps -o tsv
```

Wait for it to show an IP — if nothing appears yet, the VM is still provisioning. Try again in 30 seconds.

### Step 2: Confirm SSH Works (Baseline)

```bash
ssh -o ConnectTimeout=5 labadmin@<PUBLIC-IP>
```

You should connect successfully. Type `exit` to leave. This is your baseline — SSH works.

### Step 3: Block Yourself

Add a Deny rule with a *lower priority number* than your Allow rule. Lower number = higher priority.

```bash
az network nsg rule create \
  --resource-group break-lab-rg \
  --nsg-name break-nsg \
  --name BlockSSH \
  --priority 90 \
  --direction Inbound \
  --access Deny \
  --protocol Tcp \
  --destination-port-ranges 22
```

### Step 4: Try to Connect Again

```bash
ssh -o ConnectTimeout=5 labadmin@<PUBLIC-IP>
```

**Expected result**: Connection times out. The Deny rule at priority 90 is evaluated before the Allow rule at priority 100, so the packet gets dropped.

**What this looks like in the real world**: A security team adds a Deny rule to harden an NSG but sets the wrong priority. Applications that were working start timing out. The team checks if the service is down, checks the application logs — everything looks fine. This is a common 30-minute debugging session. Now you know to check NSG effective rules first.

### Step 5: Diagnose It the Right Way

Don't just fix the rule — use the diagnostic tool first:

1. Portal → search **Network Watcher** → **IP Flow Verify**
2. Fill in:
   - **VM**: break-vm
   - **Direction**: Inbound
   - **Protocol**: TCP
   - **Local port**: 22
   - **Remote IP**: your machine's IP (find it at `curl -s ifconfig.me`)
   - **Remote port**: any random port (e.g. 52000)
3. Run the check — it will tell you exactly which NSG rule is blocking traffic

### Step 6: Fix It

```bash
az network nsg rule delete \
  --resource-group break-lab-rg \
  --nsg-name break-nsg \
  --name BlockSSH
```

Re-run the IP Flow Verify check and confirm it now shows Allow.

---

## Part 3: Storage Access Controls — Three Ways In, Three Ways to Mess It Up

Storage accounts have multiple access layers: account keys, SAS tokens, Azure AD / RBAC, and network rules. Most misconfigurations happen when teams don't know which layer is actually protecting their data.

### Step 1: Create a Test Container and Upload a File

```bash
STORAGE_ACCOUNT=$(az storage account list -g break-lab-rg --query "[0].name" -o tsv)

az storage container create \
  --name test-container \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

echo "this is sensitive data" > /tmp/secret.txt

az storage blob upload \
  --account-name $STORAGE_ACCOUNT \
  --container-name test-container \
  --name secret.txt \
  --file /tmp/secret.txt \
  --auth-mode login
```

### Step 2: Break #1 — Try Public Access

Public blob access is disabled on this account (you set `--allow-blob-public-access false` during setup). Try to enable it anyway:

```bash
az storage account update \
  --name $STORAGE_ACCOUNT \
  --resource-group break-lab-rg \
  --allow-blob-public-access true
```

Then try to make the container public:

```bash
az storage container set-permission \
  --name test-container \
  --account-name $STORAGE_ACCOUNT \
  --public-access blob \
  --auth-mode login
```

**Expected result**: The container permission update will either fail or, if it succeeds, the blob will now be publicly readable by anyone with the URL. This demonstrates why `--allow-blob-public-access false` matters — it's a preventive control, not just a default you leave on.

**Re-disable public access:**
```bash
az storage account update \
  --name $STORAGE_ACCOUNT \
  --resource-group break-lab-rg \
  --allow-blob-public-access false
```

### Step 3: Break #2 — Generate a SAS Token with Too Many Permissions

A SAS (Shared Access Signature) token grants access without requiring Azure AD credentials. If you generate one carelessly, you hand anyone who finds the URL read/write/delete access to your data.

```bash
# Generate a SAS with all permissions, expiring far in the future
az storage blob generate-sas \
  --account-name $STORAGE_ACCOUNT \
  --container-name test-container \
  --name secret.txt \
  --permissions racwdlx \
  --expiry 2030-01-01 \
  --auth-mode login \
  --as-user \
  --full-uri
```

You now have a URL that grants full access to that blob for years. Anyone who finds this URL in a git commit, Slack message, or log file can access your data.

**The fix**: Always scope SAS tokens to the minimum permissions needed and the shortest practical expiry. For read-only access to one blob, `--permissions r` with `--expiry` set to hours or days is correct.

### Step 4: Break #3 — Rotate the Account Key

Account keys are the nuclear option — they grant full access to everything in the storage account. If an application is using key-based auth and you rotate the key, the application breaks instantly.

```bash
# Rotate key1
az storage account keys renew \
  --resource-group break-lab-rg \
  --account-name $STORAGE_ACCOUNT \
  --key key1
```

Now try to use the old key for anything — it will return an `AuthenticationFailed` error. This is what happens in production when someone rotates keys without updating the application config first.

**The lesson**: Use Azure AD authentication (`--auth-mode login`) instead of keys wherever possible. Managed Identity for apps running in Azure means you never manage keys at all.

---

## Part 4: RBAC — Scoping Permissions Wrong

RBAC misconfigurations usually go one of two directions: too permissive (user has Contributor on a subscription when they only needed Storage Blob Data Reader on one container) or too restrictive (correct role, wrong scope, so nothing works).

### Step 1: Get Your User Object ID

```bash
az ad signed-in-user show --query id -o tsv
```

### Step 2: Assign a Role at the Wrong Scope

Assign yourself Storage Blob Data Reader on the storage account — but only on a container that *doesn't* contain your test file:

```bash
az storage container create \
  --name empty-container \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee $(az ad signed-in-user show --query id -o tsv) \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/break-lab-rg/storageAccounts/$STORAGE_ACCOUNT/blobServices/default/containers/empty-container"
```

Now try to read a blob from `test-container` using only Azure AD auth (not the account key):

```bash
az storage blob download \
  --account-name $STORAGE_ACCOUNT \
  --container-name test-container \
  --name secret.txt \
  --file /tmp/downloaded.txt \
  --auth-mode login
```

**Expected result**: `AuthorizationPermissionMismatch` — you have the right role, but the wrong scope. You're authorized on `empty-container`, not `test-container`.

This is one of the most common IAM mistakes: the role looks right, but the scope is off. Always verify both role *and* scope when troubleshooting access denied errors.

### Step 3: Fix the Scope

```bash
az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee $(az ad signed-in-user show --query id -o tsv) \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/break-lab-rg/storageAccounts/$STORAGE_ACCOUNT/blobServices/default/containers/test-container"
```

Re-run the download — it should succeed now.

---

## Part 5: Terraform State Conflict — When Two People Run Apply at Once

Terraform remote state includes a lease/lock mechanism (via Azure Storage blob leasing). If two `terraform apply` runs happen simultaneously, one should block. This simulates what happens when state gets corrupted or the lock isn't released properly.

### Step 1: Set Up a Minimal Terraform Config

If you still have the state backend from Lab 7, use a new key (file name within the container) so you don't disturb previous state.

```hcl
# main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "tf-state-rg"
    storage_account_name = "YOUR_STATE_ACCOUNT"
    container_name       = "tfstate"
    key                  = "lab8/chaos.tfstate"
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "chaos" {
  name     = "chaos-test-rg"
  location = "East US 2"
}
```

```bash
terraform init
terraform apply -auto-approve
```

### Step 2: Simulate a Lock Conflict

Open a second terminal. While the first `apply` is running (or immediately after), run:

```bash
# Manually acquire the blob lease to simulate a stuck lock
STORAGE_ACCOUNT=YOUR_STATE_ACCOUNT
BLOB_NAME="lab8/chaos.tfstate"

az storage blob lease acquire \
  --blob-name "$BLOB_NAME" \
  --container-name tfstate \
  --account-name $STORAGE_ACCOUNT \
  --lease-duration 60
```

Now try `terraform apply` from the first terminal again:

```bash
terraform apply
```

**Expected result**: Terraform reports a state lock error — someone else (or a stuck process) holds the lock. It will show you the lock ID and who holds it.

### Step 3: Force-Unlock (Use Carefully)

Terraform provides a command to force-release a lock. In production, only use this when you are *certain* no other process is running.

```bash
terraform force-unlock <LOCK-ID-FROM-ERROR-MESSAGE>
```

Re-run `terraform apply` — it should succeed now.

**What this looks like in production**: A CI/CD pipeline gets killed mid-apply (server restart, timeout, someone hitting cancel). The state lock never gets released. The next deploy fails with a lock error. The team either waits for the lease to expire (up to 60 seconds for Azure Storage leases) or force-unlocks. Knowing how to diagnose and safely recover from this is a real operational skill.

### Step 4: Tear Down the Chaos Resources

```bash
terraform destroy -auto-approve
```

---

## Part 6: Dependency Deletion — Azure's Error Messages Are Actually Helpful

Azure prevents you from deleting resources that other resources depend on. Knowing what error to expect and how to read it saves debugging time.

### Step 1: Try to Delete the VNet While the VM Exists

```bash
az network vnet delete \
  --resource-group break-lab-rg \
  --name break-vnet
```

**Expected result**: Azure returns an error listing every resource currently using the VNet (NIC, subnet association, etc.). Read the full error — it tells you the exact dependency chain.

### Step 2: Try to Delete the NSG While It's Attached to a Subnet

```bash
az network nsg delete \
  --resource-group break-lab-rg \
  --name break-nsg
```

**Expected result**: Similar failure — the NSG is associated with the subnet and Azure won't orphan it.

**The pattern**: Azure enforces deletion order. When tearing down a full environment manually, the correct order is: VMs → NICs → Public IPs → NSG associations → NSGs → Subnets → VNets. Or just delete the resource group and let Azure figure out the order, which is why `az group delete` is the standard teardown method.

---

## Teardown: Clean Everything Up

```bash
az group delete --name break-lab-rg --yes --no-wait
az group delete --name chaos-test-rg --yes --no-wait
```

Confirm both groups are gone within a few minutes:

```bash
az group list --query "[?contains(name, 'break') || contains(name, 'chaos')].name" -o tsv
```

When this returns nothing, you're done.

---

## What You Broke and What It Taught You

| Break | Root Cause | Real-World Scenario |
|-------|-----------|-------------------|
| Resource lock blocked delete | Lock on RG propagates to contents | Automation script targeting wrong environment |
| NSG priority blocked SSH | Lower-numbered deny beats higher-numbered allow | Security hardening rule applied at wrong priority |
| SAS token over-permissioned | `rwdl` when `r` was needed | Token committed to git, found by scanner |
| Key rotation broke access | App using key auth, not managed identity | Rotation during on-call causes outage |
| RBAC wrong scope | Right role, wrong container | Ticket says "give access to storage" without specifying which container |
| Terraform lock conflict | Stuck lease from killed pipeline | CI/CD timeout leaves state locked for next deploy |
| Dependency delete blocked | Azure enforces deletion order | Manual teardown attempted in wrong order |

---

## Reflection

Before finishing, write down answers to these — even rough notes count:

1. Which break surprised you most? What assumption did it challenge?
2. If you were writing a runbook for the Terraform lock scenario, what steps would you list?
3. Which of the seven breaks could cause a security incident (not just an outage) if it happened in production? Why?
4. What's one monitoring alert you'd add to catch one of these issues before it becomes an incident?

---

## What's Next

You've now built, managed, secured, observed, automated, and broken Azure infrastructure. The natural next steps branch depending on where you're heading:

- **Security Engineering**: Azure Defender for Cloud, Microsoft Sentinel, Azure Policy (deny effects and compliance scoring)
- **Platform Engineering**: Azure Kubernetes Service, Azure Container Apps, private endpoints, hub-and-spoke networking
- **SRE / Operations**: Azure Site Recovery, availability zones and zone-redundant deployments, SLA calculations
- **Certifications**: AZ-900 (foundation), AZ-104 (administrator), AZ-500 (security engineer) — these labs cover significant AZ-104 and AZ-500 ground
