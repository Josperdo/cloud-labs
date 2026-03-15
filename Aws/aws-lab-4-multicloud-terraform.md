# AWS Lab 4: Multi-Cloud Terraform Project (Capstone)

## Overview
This is your capstone lab. You'll refactor your existing Azure Terraform project to work on **both Azure and AWS** with a single codebase. This is what separates "I took AWS labs" from "I'm a cloud engineer."

**Estimated time**: 4–6 hours (depends on your existing project complexity)  
**Cost**: ~$2–$5 (minimal; tear down at the end)  
**Key learning**: Writing infrastructure code that's cloud-agnostic, handling cloud-specific differences, and proving you can think like an architect.

---

## Objectives
- Refactor existing Terraform code to support multiple cloud providers
- Understand provider-specific syntax and resource differences
- Create a variables file that switches between clouds
- Deploy identical 3-tier architecture to both Azure and AWS
- Document your architecture and decision-making
- Build your primary portfolio project

---

## Prerequisites
- You have an existing Azure Terraform project that works
- Terraform is installed locally (`terraform version` returns a version)
- AWS CLI is configured (`aws sts get-caller-identity` returns your account info)
- Azure CLI is configured (`az account show` returns your subscription)

---

## Part 1: Understand Your Current Azure Project

### Step 1: Audit Your Existing Code
1. Open your Azure Terraform project locally
2. List all files:
   ```bash
   ls -la
   ```
3. You should have (at minimum):
   - `main.tf` (resource definitions)
   - `variables.tf` (input variables)
   - `outputs.tf` (outputs)
   - `terraform.tfvars` (variable values)

4. Review `main.tf`:
   - What's the architecture? (list the resources)
   - Example:
     ```hcl
     resource "azurerm_resource_group" "rg" { ... }
     resource "azurerm_virtual_network" "vnet" { ... }
     resource "azurerm_subnet" "subnet" { ... }
     # etc
     ```

**Document this**. You'll need it to map Azure resources → AWS equivalents.

---

## Part 2: Map Azure Resources to AWS Equivalents

### Step 2: Create Architecture Mapping Document

**Create a file called `ARCHITECTURE.md`** in your project:

```markdown
# Multi-Cloud Architecture

## Azure → AWS Resource Mapping

| Component | Azure | AWS | Purpose |
|-----------|-------|-----|---------|
| **Networking** | | | |
| Network | azurerm_virtual_network | aws_vpc | Isolated network |
| Subnet | azurerm_subnet | aws_subnet | Network segment |
| Firewall | azurerm_network_security_group | aws_security_group | Traffic control |
| **Compute** | | | |
| VM | azurerm_linux_virtual_machine | aws_instance | Compute resource |
| Fleet | (manual) | aws_autoscaling_group | Auto-scaling |
| LB | azurerm_lb | aws_lb | Load balancer |
| **Storage** | | | |
| Blob | azurerm_storage_account | aws_s3_bucket | Object storage |
| **Identity** | | | |
| Role | azurerm_role_assignment | aws_iam_role_policy | Access control |

## Deployment Strategy

- **Single Terraform workspace** with cloud selector variable
- **Separate provider blocks** for Azure and AWS
- **Conditional resource creation** based on `var.cloud_provider`
- **Outputs that map between clouds** for easy reference

## Security Posture

- **Networking**: VPC with public/private subnets; no direct internet access to databases
- **Compute**: Auto-scaling with health checks; stateless instances
- **Storage**: Encryption at rest; access controlled via IAM/policies
- **Identity**: Least privilege; no overly permissive roles
```

---

## Part 3: Refactor Terraform Code for Multi-Cloud

### Step 3: Update `variables.tf`

Add a cloud selector variable at the top:

```hcl
variable "cloud_provider" {
  description = "Cloud provider: 'azure' or 'aws'"
  type        = string
  default     = "azure"
  
  validation {
    condition     = contains(["azure", "aws"], var.cloud_provider)
    error_message = "cloud_provider must be 'azure' or 'aws'."
  }
}

variable "project_name" {
  description = "Project name (used for naming resources)"
  type        = string
  default     = "lab4-project"
}

variable "environment" {
  description = "Environment: 'dev' or 'prod'"
  type        = string
  default     = "dev"
}

# Azure-specific variables
variable "azure_location" {
  description = "Azure region"
  type        = string
  default     = "eastus"
}

variable "azure_subscription_id" {
  description = "Azure subscription ID"
  type        = string
}

# AWS-specific variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "instance_type_azure" {
  description = "Azure VM size"
  type        = string
  default     = "Standard_B2s"
}

variable "instance_type_aws" {
  description = "AWS instance type"
  type        = string
  default     = "t3.small"
}

variable "instance_count" {
  description = "Number of instances to deploy"
  type        = number
  default     = 2
}
```

### Step 4: Update `terraform.tfvars`

```hcl
cloud_provider       = "azure"  # Change to "aws" to deploy to AWS
project_name         = "lab4"
environment          = "dev"
azure_subscription_id = "your-subscription-id-here"
aws_region           = "us-east-1"
instance_count       = 2
```

### Step 5: Create `main.tf` with Conditional Logic

This is the big one. You'll use `count` and `providers` to conditionally create resources:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure Azure provider
provider "azurerm" {
  features {}
  subscription_id = var.azure_subscription_id
}

# Configure AWS provider
provider "aws" {
  region = var.aws_region
}

# ======================
# AZURE RESOURCES
# ======================

resource "azurerm_resource_group" "rg" {
  count            = var.cloud_provider == "azure" ? 1 : 0
  name             = "rg-${var.project_name}-${var.environment}"
  location         = var.azure_location
}

resource "azurerm_virtual_network" "vnet" {
  count               = var.cloud_provider == "azure" ? 1 : 0
  name                = "vnet-${var.project_name}"
  location            = azurerm_resource_group.rg[0].location
  resource_group_name = azurerm_resource_group.rg[0].name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "public" {
  count                = var.cloud_provider == "azure" ? 1 : 0
  name                 = "subnet-public"
  resource_group_name  = azurerm_resource_group.rg[0].name
  virtual_network_name = azurerm_virtual_network.vnet[0].name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "private" {
  count                = var.cloud_provider == "azure" ? 1 : 0
  name                 = "subnet-private"
  resource_group_name  = azurerm_resource_group.rg[0].name
  virtual_network_name = azurerm_virtual_network.vnet[0].name
  address_prefixes     = ["10.0.2.0/24"]
}

resource "azurerm_network_security_group" "web_nsg" {
  count               = var.cloud_provider == "azure" ? 1 : 0
  name                = "nsg-web-${var.project_name}"
  location            = azurerm_resource_group.rg[0].location
  resource_group_name = azurerm_resource_group.rg[0].name

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
    source_address_prefix      = "*"  # In production, restrict this
    destination_address_prefix = "*"
  }
}

# Azure VMs (simplified; in production, use azurerm_linux_virtual_machine)
resource "azurerm_public_ip" "vm_pip" {
  count               = var.cloud_provider == "azure" ? var.instance_count : 0
  name                = "pip-vm-${count.index}"
  location            = azurerm_resource_group.rg[0].location
  resource_group_name = azurerm_resource_group.rg[0].name
  allocation_method   = "Static"
}

# ======================
# AWS RESOURCES
# ======================

resource "aws_vpc" "vpc" {
  count            = var.cloud_provider == "aws" ? 1 : 0
  cidr_block       = "10.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true

  tags = {
    Name = "vpc-${var.project_name}"
  }
}

resource "aws_subnet" "public" {
  count             = var.cloud_provider == "aws" ? 1 : 0
  vpc_id            = aws_vpc.vpc[0].id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "${var.aws_region}a"

  tags = {
    Name = "subnet-public-${var.project_name}"
  }
}

resource "aws_subnet" "private" {
  count             = var.cloud_provider == "aws" ? 1 : 0
  vpc_id            = aws_vpc.vpc[0].id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "${var.aws_region}a"

  tags = {
    Name = "subnet-private-${var.project_name}"
  }
}

resource "aws_internet_gateway" "igw" {
  count  = var.cloud_provider == "aws" ? 1 : 0
  vpc_id = aws_vpc.vpc[0].id

  tags = {
    Name = "igw-${var.project_name}"
  }
}

resource "aws_route_table" "public" {
  count  = var.cloud_provider == "aws" ? 1 : 0
  vpc_id = aws_vpc.vpc[0].id

  route {
    cidr_block      = "0.0.0.0/0"
    gateway_id      = aws_internet_gateway.igw[0].id
  }

  tags = {
    Name = "rt-public-${var.project_name}"
  }
}

resource "aws_route_table_association" "public" {
  count          = var.cloud_provider == "aws" ? 1 : 0
  subnet_id      = aws_subnet.public[0].id
  route_table_id = aws_route_table.public[0].id
}

resource "aws_security_group" "web_sg" {
  count       = var.cloud_provider == "aws" ? 1 : 0
  name        = "sg-web-${var.project_name}"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.vpc[0].id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "sg-web-${var.project_name}"
  }
}

# AWS EC2 instances
resource "aws_instance" "web" {
  count             = var.cloud_provider == "aws" ? var.instance_count : 0
  ami               = "ami-0c02fb55956c7d316"  # Ubuntu 22.04 in us-east-1; update for your region
  instance_type     = var.instance_type_aws
  subnet_id         = aws_subnet.public[0].id
  security_groups   = [aws_security_group.web_sg[0].id]

  user_data = base64encode(<<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
    echo "<h1>Welcome to $(hostname -f)</h1>" > /var/www/html/index.html
  EOF
  )

  tags = {
    Name = "web-instance-${count.index}"
  }
}

# ======================
# OUTPUTS
# ======================

output "cloud_provider_deployed" {
  value = var.cloud_provider
}

output "azure_resource_group_name" {
  value = var.cloud_provider == "azure" ? azurerm_resource_group.rg[0].name : "N/A"
}

output "azure_vnet_name" {
  value = var.cloud_provider == "azure" ? azurerm_virtual_network.vnet[0].name : "N/A"
}

output "aws_vpc_id" {
  value = var.cloud_provider == "aws" ? aws_vpc.vpc[0].id : "N/A"
}

output "aws_instance_ids" {
  value = var.cloud_provider == "aws" ? aws_instance.web[*].id : []
}

output "aws_instance_public_ips" {
  value = var.cloud_provider == "aws" ? aws_instance.web[*].public_ip_address : []
}
```

---

## Part 4: Deploy and Test

### Step 6: Initialize Terraform (Azure)
```bash
# Set to deploy to Azure
terraform init

# Validate
terraform validate

# Plan (review what will be created)
terraform plan -out=tfplan-azure
```

### Step 7: Apply to Azure
```bash
terraform apply tfplan-azure
```

**Wait 3–5 minutes**. Check outputs:
```bash
terraform output
```

You should see Azure resource group name, VNet name, etc.

### Step 8: Destroy Azure Resources
```bash
terraform destroy -auto-approve
```

### Step 9: Switch to AWS and Repeat
1. Edit `terraform.tfvars`:
   ```hcl
   cloud_provider = "aws"
   ```

2. Plan:
   ```bash
   terraform plan -out=tfplan-aws
   ```

3. Apply:
   ```bash
   terraform apply tfplan-aws
   ```

**Wait 3–5 minutes**. Check outputs:
```bash
terraform output
```

You should see VPC ID, instance IDs, public IPs.

### Step 10: Verify AWS Deployment
```bash
# List instances
aws ec2 describe-instances --region us-east-1 \
  --filters "Name=tag:Name,Values=web-instance-*" \
  --query 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress]'
```

### Step 11: Cleanup
```bash
terraform destroy -auto-approve
```

---

## Part 5: Document and Polish

### Step 12: Write a Comprehensive README

Create `README.md`:

```markdown
# Multi-Cloud Infrastructure as Code

A Terraform project that deploys identical 3-tier infrastructure to both Azure and AWS.

## Architecture

- **Networking**: VPC/VNet with public and private subnets
- **Compute**: EC2 instances (AWS) or VMs (Azure) in public subnet
- **Security**: Security Groups (AWS) or NSGs (Azure) controlling inbound/outbound traffic
- **Scalability**: Configured for easy scaling (variables for instance count, sizes, etc.)

## File Structure

```
.
├── main.tf              # Resource definitions (Azure + AWS)
├── variables.tf         # Input variables
├── outputs.tf           # Output values
├── terraform.tfvars     # Variable values
├── ARCHITECTURE.md      # Design decisions
├── README.md            # This file
└── .gitignore           # Git ignore rules
```

## Prerequisites

- Terraform >= 1.0
- Azure CLI configured (`az login`)
- AWS CLI configured (`aws configure`)
- Azure subscription ID
- AWS account with EC2/VPC permissions

## Usage

### Deploy to Azure

```bash
# Edit terraform.tfvars
cloud_provider = "azure"

# Plan and apply
terraform plan -out=tfplan
terraform apply tfplan

# View outputs
terraform output
```

### Deploy to AWS

```bash
# Edit terraform.tfvars
cloud_provider = "aws"

# Plan and apply
terraform plan -out=tfplan
terraform apply tfplan

# View outputs
terraform output
```

### Cleanup

```bash
terraform destroy -auto-approve
```

## Cost Estimates

- **Azure**: ~$5/month (B2s VM × 2)
- **AWS**: ~$2/month (t3.small × 2)

Costs will vary based on region and instance size.

## Design Decisions

1. **Cloud-agnostic code**: Single `main.tf` supports both clouds using `count` expressions
2. **Separate provider blocks**: Azure and AWS providers are independent
3. **Conditional resources**: Resources only created if `cloud_provider` matches
4. **Consistent naming**: Both clouds use similar resource names for clarity
5. **Security by default**: NSGs/security groups restrict inbound traffic; no overly permissive rules

## Known Limitations

- AMI ID hardcoded for AWS (update for different region)
- No load balancer (simplified for MVP)
- No RDS/database tier (future enhancement)
- No monitoring/logging (future enhancement)

## Future Enhancements

- [ ] Add Application Load Balancer (AWS) / Azure Load Balancer
- [ ] Add RDS database in private subnet
- [ ] Enable CloudWatch (AWS) / Azure Monitor logging
- [ ] Implement auto-scaling groups (AWS ASG / Azure VMSS)
- [ ] Add S3 bucket for backups
- [ ] Implement multi-region deployment

## Lessons Learned

### Azure

- Resource Group is the container; all resources must belong to one
- NSGs can be attached at subnet or NIC level (chose NIC for this project)
- Public IPs are separate resources
- VNets are free; compute and storage incur costs

### AWS

- VPC is the container; subnets are tied to AZs
- Security Groups are instance-level; NACLs are subnet-level
- Internet Gateway must be created and attached to VPC explicitly
- Route tables control traffic routing; must explicitly route internet traffic to IGW
- Public IPs can be elastic or ephemeral; used elastic for demo

## Interview Talking Points

1. **Multi-cloud thinking**: "I can write code that targets multiple clouds. I understand cloud-agnostic patterns."
2. **IaC best practices**: "I use variables, outputs, and conditional logic to keep code DRY."
3. **Security**: "I understand network segmentation, least privilege, and cloud-native security models."
4. **Problem-solving**: "I mapped Azure resources to AWS equivalents and built a unified codebase."

## Next Steps

- Implement auto-scaling for both clouds
- Add load balancing
- Deploy a real application (e.g., Node.js app, Django app)
- Implement CI/CD (Terraform Cloud, GitHub Actions)
- Set up monitoring and alerting

## License

Personal project for portfolio/interview prep.

---

**Questions?** Reach out or check the ARCHITECTURE.md file for detailed design decisions.
```

### Step 13: Update `ARCHITECTURE.md` with Deployment Info

Add a section at the end:

```markdown
## Deployment Results

### Azure Deployment
- Resource Group: `rg-lab4-dev`
- VNet: `vnet-lab4`
- Subnets: `subnet-public`, `subnet-private`
- NSG: `nsg-web-lab4`

### AWS Deployment
- VPC ID: `vpc-xxxxx`
- Public Subnet: `subnet-xxxxx`
- Private Subnet: `subnet-xxxxx`
- Security Group: `sg-web-lab4`
- EC2 Instances: 2x `t3.small`

### Architecture Diagram

```
┌─────────────────────────────────────────┐
│           Cloud Provider                │
│      (Azure or AWS via Terraform)       │
├─────────────────────────────────────────┤
│  VPC/VNet (10.0.0.0/16)                 │
│  ├─ Public Subnet (10.0.1.0/24)         │
│  │  └─ Web Servers (2x) w/ Security SG  │
│  └─ Private Subnet (10.0.2.0/24)        │
│     └─ Database (future)                │
└─────────────────────────────────────────┘
```

## Terraform Execution Summary

**Cloud Switching**:
```bash
# Deploy to Azure
terraform plan -var="cloud_provider=azure"
terraform apply

# Deploy to AWS
terraform plan -var="cloud_provider=aws"
terraform apply
```

**Key Learnings**:
1. Cloud providers differ in resource APIs but share architectural concepts
2. Conditional logic (`count`) enables code reuse across clouds
3. Outputs need to handle "N/A" values when resources don't exist
4. Resource naming conventions help with multi-cloud clarity
```

---

## Part 6: Git and Portfolio

### Step 14: Initialize Git and Push
```bash
cd /path/to/your/project

# Initialize git
git init

# Create .gitignore
cat > .gitignore <<EOF
.terraform/
.terraform.lock.hcl
*.tfvars
*.tfplan
*.tfstate*
.env
.DS_Store
EOF

# Add files
git add .

# Commit
git commit -m "Initial multi-cloud infrastructure project

- Support for both Azure and AWS
- Single codebase with cloud selector variable
- Deploys VPC/VNet with public/private subnets
- Security groups configured for web tier
- Auto-cleanup via terraform destroy"

# (If pushing to GitHub)
git remote add origin https://github.com/YOUR-USERNAME/multi-cloud-infrastructure.git
git push -u origin main
```

### Step 15: Update Your Portfolio Website

Add a project card:

```markdown
### Multi-Cloud Infrastructure as Code

**GitHub**: [multi-cloud-infrastructure](https://github.com/your-username/multi-cloud-infrastructure)

Terraform project that deploys identical infrastructure to both Azure and AWS using a single codebase. Demonstrates:
- Cloud-agnostic IaC patterns (conditional resource creation)
- VPC/VNet architecture with public/private subnets
- Security hardening (NSGs/Security Groups)
- AWS and Azure API/CLI fluency

**Technologies**: Terraform, Azure, AWS, Infrastructure as Code

**Key skills**: IaC, multi-cloud thinking, DevOps, networking, security
```

---

## Part 7: Interview Prep

### Step 16: Prepare Your Talking Points

**Narrative for interviews**:

> "I built a Terraform project that deploys infrastructure to both Azure and AWS. Instead of learning each cloud separately, I created a unified codebase using Terraform's conditional logic to switch between providers. This demonstrates that I understand cloud-agnostic architectural patterns and can quickly transfer knowledge across cloud platforms.
>
> The infrastructure is a 3-tier pattern with VPC/VNet, public/private subnets, and security groups/NSGs. I've documented my architecture decisions and mapped Azure resources to AWS equivalents.
>
> This project shows I can write production-ready IaC, not just click through portals."

**Interview questions to prepare for**:

1. "Walk us through your multi-cloud project. Why did you do it?"
2. "What's the difference between Azure NSGs and AWS Security Groups?"
3. "How did you handle cloud-specific differences in Terraform?"
4. "What would you do next with this infrastructure?"
5. "How would you scale this to production?"

---

## Key Concepts to Understand

| Concept | Importance |
|---------|-----------|
| **Terraform count** | Essential for conditional resource creation |
| **Provider blocks** | Required for multi-cloud support |
| **Variables and outputs** | Enable code reuse and readability |
| **Resource mapping** | Azure ≠ AWS; understand equivalents |
| **IaC best practices** | DRY, modular, documented code |

---

## Cleanup

```bash
# Destroy all resources
terraform destroy -auto-approve

# Delete local state files (if not in .gitignore)
rm -rf .terraform/ .terraform.lock.hcl terraform.tfstate*
```

---

## Next Steps (Advanced)

- [ ] Add Terraform modules (separate networking, compute, storage modules)
- [ ] Implement CI/CD (GitHub Actions → Terraform Plan/Apply)
- [ ] Add monitoring (CloudWatch for AWS, Monitor for Azure)
- [ ] Deploy actual application (e.g., Node.js via Docker or systemd)
- [ ] Multi-region support
- [ ] High availability (load balancer + ASG/VMSS)
- [ ] Data persistence (RDS for AWS, SQL Database for Azure)

---

**This is your capstone project. Use it as the centerpiece of your "I'm a Cloud Engineer" story.**
