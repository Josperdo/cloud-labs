# Lab 6: Terraform Fundamentals — Deploying Azure Infrastructure as Code

Check box if done: []

## Overview
Clicking through the Azure portal works for learning and one-offs. But on a real team, you need infrastructure that's repeatable, reviewable in code review, and deployable in minutes without manual steps. Terraform is the industry-standard tool for this. Once you know Terraform, the skills transfer directly to AWS, GCP, or any other cloud — only the provider changes, not the workflow.

This lab deploys the same networking setup from Lab 1 (VNet, subnets, NSGs) entirely through Terraform. By the end, you'll understand the full Terraform workflow and why teams use it instead of clicking through portals.

**Estimated time**: 60–75 minutes
**Cost**: ~$0.50 (networking resources are nearly free; tear down at the end)

> **Free Trial Note**: This lab creates only VNets, subnets, NSGs, and peerings — **no VMs**. You will not hit vCPU quotas or VM count limits. This lab is safe to run on an Azure free trial.

> **Note**: This lab requires Terraform and the Azure CLI installed on your machine. See the Prerequisites section below before starting.

---

## Scenario
Your team is standardizing on Terraform for all infrastructure. A senior engineer has asked you to recreate the application network from Lab 1 (web VNet, database VNet, NSGs, and peering) as a Terraform configuration, so future environments can be deployed with a single command instead of 30 minutes of portal clicking.

---

## Objectives
- Understand the Terraform workflow: write → init → plan → apply → destroy
- Write HCL (HashiCorp Configuration Language) to define Azure resources
- Use variables to make configurations reusable
- Use outputs to extract information from deployed infrastructure
- Modify deployed infrastructure and see how Terraform handles changes
- Understand Terraform state and why it matters

---

## Prerequisites: Install Terraform

### Install on macOS (Homebrew)
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform -version
```

### Install on Windows (Chocolatey)
```powershell
choco install terraform
terraform -version
```

### Install on Ubuntu/Debian
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
terraform -version
```

You also need the **Azure CLI** installed and logged in:
```bash
az login
az account show  # Confirm you're in the right subscription
```

---

## Part 1: Set Up Your Working Directory

### Step 1: Create a Project Directory
```bash
mkdir tf-azure-network
cd tf-azure-network
```

Everything in this lab lives in this directory. Terraform reads all `.tf` files in the current directory as a single configuration.

### Step 2: Understand the Files You'll Create

| File | Purpose |
|------|---------|
| `main.tf` | The resources you want to create |
| `variables.tf` | Input variables so values aren't hardcoded |
| `outputs.tf` | Values to print after deployment |
| `terraform.tfvars` | Actual values for your variables |

---

## Part 2: Configure the Azure Provider

### Step 3: Create `main.tf` — Provider Block
Create a file called `main.tf` with this content:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  required_version = ">= 1.5"
}

provider "azurerm" {
  features {}
}
```

**What this does**: Declares that this configuration uses the `azurerm` provider (Terraform's Azure plugin) at version 3.x. The `features {}` block is required by the azurerm provider even when empty.

**Transferable concept**: On AWS you'd write `provider "aws" { region = "us-east-1" }`. The pattern is identical — only the provider name changes.

---

## Part 3: Define the Infrastructure

### Step 4: Add a Resource Group
Add this block to `main.tf` below the provider block:

```hcl
resource "azurerm_resource_group" "network" {
  name     = var.resource_group_name
  location = var.location

  tags = {
    environment = var.environment
    managed_by  = "terraform"
  }
}
```

**Reading this**: `resource` is the keyword. `"azurerm_resource_group"` is the resource type (from the provider docs). `"network"` is your local name for this resource — you use it to reference this resource elsewhere in your config.

### Step 5: Add the Application VNet and Subnets
Add this to `main.tf`:

```hcl
resource "azurerm_virtual_network" "app" {
  name                = "app-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
}

resource "azurerm_subnet" "web" {
  name                 = "web-subnet"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.app.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "db" {
  name                 = "db-subnet"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.app.name
  address_prefixes     = ["10.0.2.0/24"]
}
```

**Notice the references**: `azurerm_resource_group.network.location` refers to the `location` attribute of your `azurerm_resource_group` resource named `"network"`. Terraform builds a dependency graph from these references — it knows to create the resource group before the VNet, because the VNet depends on it.

### Step 6: Add the Dev VNet
```hcl
resource "azurerm_virtual_network" "dev" {
  name                = "dev-vnet"
  address_space       = ["10.1.0.0/16"]
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
}

resource "azurerm_subnet" "dev" {
  name                 = "dev-subnet"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.dev.name
  address_prefixes     = ["10.1.1.0/24"]
}
```

### Step 7: Add the Web NSG with Rules
```hcl
resource "azurerm_network_security_group" "web" {
  name                = "web-nsg"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name

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
}
```

**SSH is scoped to your IP**: The `var.admin_ip_address` variable means you set this once and it's used wherever SSH access needs to be restricted. You'll define this variable next.

### Step 8: Add the Database NSG
```hcl
resource "azurerm_network_security_group" "db" {
  name                = "db-nsg"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name

  security_rule {
    name                       = "AllowDBFromWebTier"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "5432"
    source_address_prefix      = "10.0.1.0/24"
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
}
```

### Step 9: Associate NSGs with Subnets and Add VNet Peering
```hcl
resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.web.id
  network_security_group_id = azurerm_network_security_group.web.id
}

resource "azurerm_subnet_network_security_group_association" "db" {
  subnet_id                 = azurerm_subnet.db.id
  network_security_group_id = azurerm_network_security_group.db.id
}

resource "azurerm_virtual_network_peering" "app_to_dev" {
  name                      = "app-to-dev"
  resource_group_name       = azurerm_resource_group.network.name
  virtual_network_name      = azurerm_virtual_network.app.name
  remote_virtual_network_id = azurerm_virtual_network.dev.id
}

resource "azurerm_virtual_network_peering" "dev_to_app" {
  name                      = "dev-to-app"
  resource_group_name       = azurerm_resource_group.network.name
  virtual_network_name      = azurerm_virtual_network.dev.name
  remote_virtual_network_id = azurerm_virtual_network.app.id
}
```

---

## Part 4: Add Variables and Outputs

### Step 10: Create `variables.tf`
Create a new file called `variables.tf`:

```hcl
variable "resource_group_name" {
  description = "Name of the resource group to create"
  type        = string
  default     = "tf-network-rg"
}

variable "location" {
  description = "Azure region for all resources"
  type        = string
  default     = "East US"
}

variable "environment" {
  description = "Environment label (e.g. dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "admin_ip_address" {
  description = "Your public IP address for SSH access (e.g. 203.0.113.42/32)"
  type        = string
}
```

**Note**: `admin_ip_address` has no default — Terraform will prompt you for it at runtime, or you can set it in `terraform.tfvars`.

### Step 11: Create `terraform.tfvars`
Create a file called `terraform.tfvars` — this is where you put your actual values:

```hcl
resource_group_name = "tf-network-rg"
location            = "East US"
environment         = "dev"
admin_ip_address    = "YOUR_PUBLIC_IP/32"  # Replace with your actual IP
```

> **Find your IP**: Run `curl -s https://api.ipify.org` in your terminal.

**Security note**: `terraform.tfvars` can contain sensitive values. Add it to `.gitignore` if you're committing this directory to git:
```bash
echo "terraform.tfvars" >> .gitignore
```

### Step 12: Create `outputs.tf`
Create a file called `outputs.tf`:

```hcl
output "resource_group_name" {
  description = "Name of the created resource group"
  value       = azurerm_resource_group.network.name
}

output "app_vnet_id" {
  description = "Resource ID of the application VNet"
  value       = azurerm_virtual_network.app.id
}

output "web_subnet_id" {
  description = "Resource ID of the web subnet"
  value       = azurerm_subnet.web.id
}

output "db_subnet_id" {
  description = "Resource ID of the database subnet"
  value       = azurerm_subnet.db.id
}
```

Outputs let you see key information after deployment, and other Terraform configurations can reference them as inputs.

---

## Part 5: The Terraform Workflow

### Step 13: Initialize the Working Directory
```bash
terraform init
```

**What this does**: Downloads the `azurerm` provider plugin from the Terraform registry and sets up the local backend (state storage). You must run `init` before anything else, and again whenever you change providers or versions.

You'll see output like:
```
Initializing provider plugins...
- Finding hashicorp/azurerm versions matching "~> 3.0"...
- Installing hashicorp/azurerm v3.x.x...

Terraform has been successfully initialized!
```

### Step 14: Plan the Deployment
```bash
terraform plan
```

**What this does**: Terraform compares your configuration against the current state of Azure and shows exactly what it would create, modify, or destroy — without touching anything yet.

Read the output carefully:
- Lines starting with `+` are resources to **create**
- Lines starting with `~` are resources to **modify**
- Lines starting with `-` are resources to **destroy**

You should see ~12 resources to create. Review them before applying.

**This is the power of Terraform**: A developer proposes a change, runs `terraform plan`, and the output becomes the code review artifact. The team can see exactly what will change before anything touches production.

### Step 15: Apply the Configuration
```bash
terraform apply
```

Terraform will show the plan again and prompt you to confirm:
```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

Type `yes` and press Enter. Terraform creates all resources in the correct order, waiting for dependencies to complete before starting dependents.

**Wait ~2–3 minutes**. When complete, you'll see your outputs:
```
Outputs:

app_vnet_id    = "/subscriptions/.../resourceGroups/tf-network-rg/..."
db_subnet_id   = "/subscriptions/.../resourceGroups/tf-network-rg/..."
...
```

### Step 16: Verify in the Portal
1. Open the Azure portal → **Resource groups** → `tf-network-rg`
2. Confirm all resources are present: two VNets, subnets, two NSGs, NSG associations, and peerings
3. Check the NSG rules match what you wrote in the configuration

---

## Part 6: Modify Infrastructure and See How Terraform Handles Changes

### Step 17: Add a Tag to the Resource Group
Open `main.tf` and modify the resource group's tags block:

```hcl
  tags = {
    environment = var.environment
    managed_by  = "terraform"
    cost_center = "engineering"   # Add this line
  }
```

Run `terraform plan` again:
```bash
terraform plan
```

You'll see:
```
  ~ resource "azurerm_resource_group" "network" {
      ~ tags = {
          + "cost_center" = "engineering"
            # (other attributes unchanged)
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

Only the tag change is shown. Terraform knows the difference between what's deployed and what you declared. Apply it:
```bash
terraform apply
```

**What this demonstrates**: Infrastructure changes are predictable and auditable. You don't hunt through a portal to figure out what changed — it's all in git history.

### Step 18: Understand the State File
```bash
ls -la
```

You'll see a file called `terraform.tfstate`. Open it to see what's inside:
```bash
cat terraform.tfstate
```

The state file is JSON that maps your configuration to real Azure resources. Terraform uses it to:
- Know what's already deployed when you run `plan`
- Track resource IDs for resources it manages
- Determine what to destroy when you run `destroy`

**Never edit the state file manually.** In Lab 7, you'll move this to Azure Storage so your whole team shares a single state file instead of each having their own local copy.

---

## Part 7: Destroy the Infrastructure

### Step 19: Tear Down Everything
```bash
terraform destroy
```

Terraform shows everything it will delete. Type `yes` to confirm. All 12+ resources are deleted in the correct order (peerings before VNets, associations before NSGs, etc.).

**Compare to portal**: Deleting a resource group in the portal takes 2–3 minutes and deletes everything inside. `terraform destroy` does the same thing but also cleans up the state file so Terraform knows the resources are gone.

---

## What You Practiced

| Task | Why It Matters on the Job |
|------|--------------------------|
| **Provider configuration** | Every Terraform project starts here; the pattern is identical across AWS, GCP, and Azure |
| **Resource references and dependency graph** | Terraform builds the correct deployment order automatically — you don't have to |
| **Variables and tfvars** | Makes configurations reusable across environments; no hardcoded values |
| **terraform plan as a review artifact** | Teams review the plan output before applying; it's the code review for infrastructure |
| **State file** | Understanding state is critical — losing or corrupting it is a major operational incident |
| **terraform destroy** | Clean teardown of all resources in the correct order |

---

## Common Mistakes to Avoid
- **Editing the state file by hand**: The state file is Terraform's source of truth; manual edits corrupt it
- **Running `apply` without reading `plan`**: The plan shows exactly what will change — always review it
- **Storing `terraform.tfvars` in git with real credentials**: Put it in `.gitignore`; use environment variables for secrets
- **Hardcoding subscription IDs and resource group names**: Use variables; that's what they're for
- **Leaving state files local**: A local state file means only you can make changes; Lab 7 fixes this with remote state
- **Not pinning the provider version**: `~> 3.0` means any 3.x version. Without pinning, a provider update can break your configuration

---

## Next Steps
- **Lab 7** (next): Move state to Azure Storage and organize configurations into reusable modules
- Add a `count` or `for_each` to create multiple subnets from a list
- Use `terraform workspace` to manage dev and prod environments from the same configuration
- Import an existing Azure resource into Terraform with `terraform import`
