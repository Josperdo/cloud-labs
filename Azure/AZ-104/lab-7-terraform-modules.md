# Lab 7: Terraform Modules and Remote State — Team-Ready Infrastructure

Check box if done: []

## Overview
A single `main.tf` works fine for a lab. On a real team, you need configurations that multiple engineers can work on simultaneously without overwriting each other's changes, organized into reusable pieces so you don't copy-paste the same networking setup into every project. This lab covers the two things that make Terraform production-grade: remote state (shared, locked, versioned) and modules (reusable infrastructure components).

**Estimated time**: 65–80 minutes
**Cost**: ~$1–2 (one storage account for state + networking resources; tear down at the end)

> **Free Trial Note**: This lab creates one storage account (LRS, ~$0.02/month) and VNets/NSGs. **No VMs** — you will not hit vCPU or VM count limits.

> **Prerequisites**: Complete Lab 6 first. This lab builds on those concepts directly.

---

## Scenario
Your team just approved Terraform as the standard for all Azure infrastructure. Two problems need solving immediately:

1. The state file is stuck on one engineer's laptop. If they're on vacation and something needs to change, no one else can run Terraform safely.
2. Three different projects are each copy-pasting the same VNet/subnet/NSG configuration. When the security team changes the SSH rule, someone has to update it in three places.

You're going to fix both problems: move state to Azure Storage (so the whole team shares it), and refactor the networking config into a module (so all three projects use the same code).

---

## Objectives
- Create an Azure Storage backend for Terraform remote state
- Configure state locking to prevent simultaneous deployments
- Build a reusable networking module
- Use the module to deploy two environments (dev and prod) from the same code
- Understand how to pass data between Terraform configurations
- Import an existing resource into Terraform state

---

## Part 1: Set Up Remote State Storage

### Step 1: Create the State Storage Account
Remote state needs to exist before your Terraform configurations can use it. You'll create this manually (a rare case where clicking through the portal is appropriate — the state backend can't manage its own state).

**Pick a name for your storage account now.** It must be globally unique, 3–24 lowercase letters/numbers only, no hyphens. Write it down — you'll use it in several places throughout this lab.

Example: `tfstate` + your initials + a 4-digit number → `tfstatejs2847`

**macOS / Linux** — run these commands, replacing `YOUR_ACCOUNT_NAME` with the name you chose:
```bash
az group create --name tf-state-rg --location eastus

az storage account create \
  --name YOUR_ACCOUNT_NAME \
  --resource-group tf-state-rg \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

az storage container create \
  --name tfstate \
  --account-name YOUR_ACCOUNT_NAME
```

**Windows (PowerShell)** — same commands work in PowerShell with the Azure CLI installed:
```powershell
az group create --name tf-state-rg --location eastus

az storage account create `
  --name YOUR_ACCOUNT_NAME `
  --resource-group tf-state-rg `
  --location eastus `
  --sku Standard_LRS `
  --kind StorageV2 `
  --min-tls-version TLS1_2 `
  --allow-blob-public-access false

az storage container create --name tfstate --account-name YOUR_ACCOUNT_NAME
```

**Or use the portal** — if CLI gives you trouble, you can create the storage account and container manually (same as Lab 3 Part 1), then come back here. Just use `Standard LRS`, disable blob public access, and create a container called `tfstate`.

**Why a dedicated resource group for state?** State files contain the IDs of every resource you manage. If you delete the state accidentally (say, by deleting the resource group it's in alongside other things), Terraform loses track of your infrastructure. State storage lives alone in its own resource group so it's never caught in a cleanup sweep.

### Step 2: Enable Versioning on the State Container
**macOS / Linux:**
```bash
az storage account blob-service-properties update \
  --account-name YOUR_ACCOUNT_NAME \
  --resource-group tf-state-rg \
  --enable-versioning true
```

**Windows (PowerShell):**
```powershell
az storage account blob-service-properties update `
  --account-name YOUR_ACCOUNT_NAME `
  --resource-group tf-state-rg `
  --enable-versioning true
```

**Why versioning?** If a `terraform apply` fails halfway through, the state file can end up in a partial state. With versioning enabled, you can roll back to the previous version of the state file rather than manually reconstructing it.

---

## Part 2: Build a Reusable Networking Module

### Step 3: Set Up the Directory Structure
Create a new directory structure.

**macOS / Linux:**
```bash
mkdir -p tf-project/modules/networking
mkdir -p tf-project/environments/dev
mkdir -p tf-project/environments/prod
cd tf-project
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Path tf-project\modules\networking -Force
New-Item -ItemType Directory -Path tf-project\environments\dev -Force
New-Item -ItemType Directory -Path tf-project\environments\prod -Force
cd tf-project
```

Your layout should look like this:
```
tf-project/
├── modules/
│   └── networking/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── dev/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── terraform.tfvars
    └── prod/
        ├── main.tf
        ├── variables.tf
        └── terraform.tfvars
```

**This structure separates concerns**: modules contain reusable components; environments contain the specific configuration for each deployment. The module doesn't know about environments — environments just call the module with different inputs.

### Step 4: Write the Networking Module — `modules/networking/variables.tf`
```hcl
variable "resource_group_name" {
  description = "Name of the resource group to create resources in"
  type        = string
}

variable "location" {
  description = "Azure region for all resources"
  type        = string
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
}

variable "vnet_address_space" {
  description = "Address space for the application VNet"
  type        = string
  # No default — callers must provide this to avoid IP conflicts
}

variable "web_subnet_prefix" {
  description = "CIDR range for the web tier subnet"
  type        = string
}

variable "db_subnet_prefix" {
  description = "CIDR range for the database tier subnet"
  type        = string
}

variable "admin_ip_address" {
  description = "IP address allowed SSH access (CIDR notation, e.g. 1.2.3.4/32)"
  type        = string
}

variable "db_port" {
  description = "Database port to allow from web tier (e.g. 5432 for PostgreSQL)"
  type        = number
  default     = 5432
}
```

### Step 5: Write the Networking Module — `modules/networking/main.tf`
```hcl
resource "azurerm_virtual_network" "app" {
  name                = "${var.environment}-app-vnet"
  address_space       = [var.vnet_address_space]
  location            = var.location
  resource_group_name = var.resource_group_name

  tags = {
    environment = var.environment
    managed_by  = "terraform"
  }
}

resource "azurerm_subnet" "web" {
  name                 = "web-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.app.name
  address_prefixes     = [var.web_subnet_prefix]
}

resource "azurerm_subnet" "db" {
  name                 = "db-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.app.name
  address_prefixes     = [var.db_subnet_prefix]
}

resource "azurerm_network_security_group" "web" {
  name                = "${var.environment}-web-nsg"
  location            = var.location
  resource_group_name = var.resource_group_name

  security_rule {
    name                       = "AllowHTTP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "AllowHTTPS"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "AllowSSH"
    priority                   = 120
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = var.admin_ip_address
    destination_address_prefix = "*"
  }

  tags = {
    environment = var.environment
    managed_by  = "terraform"
  }
}

resource "azurerm_network_security_group" "db" {
  name                = "${var.environment}-db-nsg"
  location            = var.location
  resource_group_name = var.resource_group_name

  security_rule {
    name                       = "AllowDBFromWebTier"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = tostring(var.db_port)
    source_address_prefix      = var.web_subnet_prefix
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "DenyAllOther"
    priority                   = 4000
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = {
    environment = var.environment
    managed_by  = "terraform"
  }
}

resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.web.id
  network_security_group_id = azurerm_network_security_group.web.id
}

resource "azurerm_subnet_network_security_group_association" "db" {
  subnet_id                 = azurerm_subnet.db.id
  network_security_group_id = azurerm_network_security_group.db.id
}
```

**Notice**: Resource names are prefixed with `var.environment` (e.g., `"${var.environment}-app-vnet"`). This prevents naming conflicts when both dev and prod use the same module.

### Step 6: Write the Networking Module — `modules/networking/outputs.tf`
```hcl
output "vnet_id" {
  description = "Resource ID of the application VNet"
  value       = azurerm_virtual_network.app.id
}

output "vnet_name" {
  description = "Name of the application VNet"
  value       = azurerm_virtual_network.app.name
}

output "web_subnet_id" {
  description = "Resource ID of the web tier subnet"
  value       = azurerm_subnet.web.id
}

output "db_subnet_id" {
  description = "Resource ID of the database tier subnet"
  value       = azurerm_subnet.db.id
}

output "web_nsg_id" {
  description = "Resource ID of the web NSG"
  value       = azurerm_network_security_group.web.id
}
```

---

## Part 3: Configure the Dev Environment

### Step 7: Write `environments/dev/main.tf`
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }

  backend "azurerm" {
    resource_group_name  = "tf-state-rg"
    storage_account_name = "YOUR_STORAGE_ACCOUNT_NAME"  # Replace this
    container_name       = "tfstate"
    key                  = "dev/networking.tfstate"
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "dev" {
  name     = var.resource_group_name
  location = var.location

  tags = {
    environment = "dev"
    managed_by  = "terraform"
  }
}

module "networking" {
  source = "../../modules/networking"

  resource_group_name = azurerm_resource_group.dev.name
  location            = azurerm_resource_group.dev.location
  environment         = "dev"
  vnet_address_space  = var.vnet_address_space
  web_subnet_prefix   = var.web_subnet_prefix
  db_subnet_prefix    = var.db_subnet_prefix
  admin_ip_address    = var.admin_ip_address
  db_port             = 5432
}
```

**The `backend` block**: This tells Terraform to store the state file in Azure Blob Storage instead of locally. The `key` is the path within the container — `"dev/networking.tfstate"` keeps dev and prod state separate.

**The `module` block**: This calls your networking module, passing in environment-specific values. The module code lives in one place; the environment configuration lives separately.

### Step 8: Write `environments/dev/variables.tf`
```hcl
variable "resource_group_name" {
  description = "Name of the dev resource group"
  type        = string
  default     = "dev-network-rg"
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "East US"
}

variable "vnet_address_space" {
  description = "Address space for dev VNet"
  type        = string
}

variable "web_subnet_prefix" {
  type = string
}

variable "db_subnet_prefix" {
  type = string
}

variable "admin_ip_address" {
  description = "Your IP for SSH access"
  type        = string
}
```

### Step 9: Write `environments/dev/terraform.tfvars`
```hcl
resource_group_name = "dev-network-rg"
location            = "East US"
vnet_address_space  = "10.10.0.0/16"
web_subnet_prefix   = "10.10.1.0/24"
db_subnet_prefix    = "10.10.2.0/24"
admin_ip_address    = "YOUR_PUBLIC_IP/32"  # Replace with your IP
```

---

## Part 4: Configure the Prod Environment

### Step 10: Write `environments/prod/main.tf`
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }

  backend "azurerm" {
    resource_group_name  = "tf-state-rg"
    storage_account_name = "YOUR_STORAGE_ACCOUNT_NAME"  # Same account, different key
    container_name       = "tfstate"
    key                  = "prod/networking.tfstate"     # Different key = different state
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "prod" {
  name     = var.resource_group_name
  location = var.location

  tags = {
    environment = "prod"
    managed_by  = "terraform"
  }
}

module "networking" {
  source = "../../modules/networking"

  resource_group_name = azurerm_resource_group.prod.name
  location            = azurerm_resource_group.prod.location
  environment         = "prod"
  vnet_address_space  = var.vnet_address_space
  web_subnet_prefix   = var.web_subnet_prefix
  db_subnet_prefix    = var.db_subnet_prefix
  admin_ip_address    = var.admin_ip_address
  db_port             = 5432
}
```

### Step 11: Write `environments/prod/variables.tf` and `terraform.tfvars`

`environments/prod/variables.tf` — same as dev's:
```hcl
variable "resource_group_name" {
  type    = string
  default = "prod-network-rg"
}

variable "location" {
  type    = string
  default = "East US"
}

variable "vnet_address_space" {
  type = string
}

variable "web_subnet_prefix" {
  type = string
}

variable "db_subnet_prefix" {
  type = string
}

variable "admin_ip_address" {
  type = string
}
```

`environments/prod/terraform.tfvars` — **different IP ranges** (never overlap with dev):
```hcl
resource_group_name = "prod-network-rg"
location            = "East US"
vnet_address_space  = "10.20.0.0/16"   # Different from dev's 10.10.0.0/16
web_subnet_prefix   = "10.20.1.0/24"
db_subnet_prefix    = "10.20.2.0/24"
admin_ip_address    = "YOUR_PUBLIC_IP/32"
```

---

## Part 5: Deploy Both Environments

### Step 12: Deploy Dev
```bash
cd environments/dev
terraform init
terraform plan
terraform apply
```

When `init` runs with the `backend` block configured, Terraform connects to Azure Storage and creates (or reads) the state file at `tfstate/dev/networking.tfstate`.

### Step 13: Deploy Prod
```bash
cd ../prod
terraform init
terraform plan
terraform apply
```

Prod gets its own state file at `tfstate/prod/networking.tfstate`. The two environments are completely independent — changing one doesn't affect the other.

### Step 14: Verify State Storage in the Portal
1. Open the Azure portal → **Storage accounts** → your `tfstate<random>` account
2. Left sidebar → **Containers** → `tfstate`
3. You should see two state files:
   - `dev/networking.tfstate`
   - `prod/networking.tfstate`

Click on one to see its properties. The blob is a JSON file containing your entire infrastructure state. You can download it and inspect it, but never edit it manually.

### Step 15: Understand State Locking
While a `terraform apply` is running, Terraform creates a lease on the blob to prevent another engineer from running `apply` at the same time. If two people tried to apply simultaneously, the state file could end up corrupted.

**What happens when a lock is stuck**: If a Terraform run is interrupted (killed process, network failure), the lock sometimes remains. You can see and break it with:
```bash
terraform force-unlock <LOCK_ID>
```

Only do this after confirming no one else is actually running Terraform — breaking an active lock can corrupt state.

---

## Part 6: Updating the Module Across Both Environments

### Step 16: Simulate a Security Policy Change
The security team says SSH should be removed from the NSG entirely — all access should go through Azure Bastion instead. This change needs to go to both dev and prod.

In `modules/networking/main.tf`, remove (or comment out) the SSH security rule from the web NSG:

```hcl
# Remove this entire block:
# security_rule {
#   name                       = "AllowSSH"
#   ...
# }
```

Now apply the change to dev first:
```bash
cd environments/dev
terraform plan    # Review: should show 1 change (removing SSH rule from web NSG)
terraform apply
```

After verifying dev looks correct, apply to prod:
```bash
cd ../prod
terraform plan
terraform apply
```

**What this demonstrates**: One change to the module propagates to all environments that use it. If you had copied the NSG configuration into each project separately, you'd have to find and update every copy manually — and inevitably miss one.

---

## Part 7: Import an Existing Resource

### Step 17: Understand terraform import
Sometimes Azure resources exist that weren't created by Terraform — a team member clicked through the portal, or they were created before Terraform was adopted. `terraform import` brings an existing resource into Terraform management.

**Scenario**: Suppose the `tf-state-rg` resource group was created manually (it was — you did that in Step 1). Let's practice importing it.

First, add a resource block to a temporary `import-test.tf` file in the `environments/dev` directory:

```hcl
resource "azurerm_resource_group" "state" {
  name     = "tf-state-rg"
  location = "East US"
}
```

Now run the import (replace `<subscription_id>` with your actual subscription ID):
```bash
az account show --query id --output tsv   # Get your subscription ID

terraform import \
  azurerm_resource_group.state \
  /subscriptions/<subscription_id>/resourceGroups/tf-state-rg
```

Run `terraform plan` — if the resource block matches what's in Azure, the plan should show no changes. If there are differences (like tags), the plan will show what would be modified to make them match.

**When to use import**: When you're adopting Terraform for an existing environment. The workflow is: write a resource block that matches the real resource → import it → run plan → fix any drift → the resource is now under Terraform management.

Remove the `import-test.tf` file after you're done:
```bash
rm import-test.tf
```

---

## Part 8: Cleanup

### Step 18: Destroy Both Environments
```bash
cd environments/dev
terraform destroy

cd ../prod
terraform destroy
```

### Step 19: Clean Up State Storage
After both environments are destroyed, you can delete the state storage if you're done with this lab. If you plan to continue using Terraform, keep it.

```bash
az group delete --name tf-state-rg --yes
```

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|--------------------------|
| **Remote state in Azure Storage** | Local state files block collaboration; remote state is the only production-viable option |
| **State locking** | Prevents concurrent applies from corrupting state; critical in teams with CI/CD pipelines |
| **Reusable modules** | One module, many environments; a security fix in the module propagates everywhere |
| **Environment-specific tfvars** | Dev and prod use the same code with different IP ranges, names, and settings |
| **Separate state per environment** | Dev and prod are independent; a failed prod apply doesn't corrupt dev state |
| **terraform import** | Brings manually-created resources under Terraform management |

---

## Common Mistakes to Avoid
- **Sharing state between environments**: If dev and prod share a state file, a `terraform destroy` in dev can delete prod resources. Always use separate state files.
- **Deleting the state storage account**: The state is the map Terraform uses to find your resources. Losing it means you have to `import` everything from scratch.
- **Committing state files to git**: State files contain resource IDs and sometimes sensitive values (outputs from secret resources). They belong in blob storage, not git.
- **Not using state locking**: Azure Blob Storage provides locking automatically when you use the `azurerm` backend — never work around it.
- **Calling modules with absolute paths**: Use `../../modules/networking` (relative) instead of `/home/user/modules/networking` (absolute). Absolute paths break on other machines.
- **One giant module for everything**: Modules should be focused. A "networking" module should do networking. A "compute" module should do compute. Monolithic modules are hard to test and reuse.

---

## Transferring These Skills to AWS

When you move to AWS, the concepts are identical — only the syntax changes:

| Azure (this lab) | AWS equivalent |
|-----------------|---------------|
| `provider "azurerm"` | `provider "aws"` |
| Azure Blob Storage backend | S3 backend with DynamoDB for locking |
| `azurerm_resource_group` | (no direct equivalent; use `aws_resourcegroups_group` or just tags) |
| `azurerm_virtual_network` | `aws_vpc` |
| `azurerm_subnet` | `aws_subnet` |
| `azurerm_network_security_group` | `aws_security_group` |

The module structure, state management, and `plan → apply → destroy` workflow are exactly the same. The investment in learning Terraform on Azure pays off immediately when you start on AWS.

---

## Next Steps
- Add a `versions.tf` with `required_providers` locked to exact versions for reproducible builds
- Set up a CI/CD pipeline (GitHub Actions or Azure DevOps) that runs `terraform plan` on PRs and `terraform apply` on merge to main
- Use `terraform_remote_state` data source to read outputs from one environment into another (e.g., prod's VNet ID into a peering configuration)
- Explore the Terraform registry for community modules: `registry.terraform.io`
- Add `moved` blocks when renaming resources to avoid destroying and recreating them
