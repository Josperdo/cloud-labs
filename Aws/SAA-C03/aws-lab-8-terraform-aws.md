# AWS Lab 8: Terraform on AWS

Check box if done: []

## Overview
Labs 1–7 were built by clicking through the console — good for building a mental model of what each service does, bad for reproducibility (Lab 7 called this out directly: no way to redeploy without redoing every click). This lab rebuilds a slice of Lab 5's architecture in Terraform, with a proper module structure and remote state backed by S3 + DynamoDB locking — the same pattern [Azure Lab 6–7](../../Azure/AZ-104/lab-6-terraform-fundamentals.md) used, so you can compare the two clouds' Terraform ergonomics directly.

**Estimated time**: 4–5 hours
**Cost**: ~$0.50–$2 (S3 + DynamoDB for state cost fractions of a cent; the VPC/EC2 resources you deploy briefly cost a small amount — destroy at the end)

---

## Objectives
- Set up an S3 + DynamoDB remote state backend with locking
- Structure Terraform code into reusable modules (networking, compute) instead of one flat `main.tf`
- Deploy a VPC and an Auto Scaling Group through modules with variables
- Force a state lock conflict and recover from it — same exercise as the Azure chaos lab, different cloud
- Compare the AWS and Azure Terraform provider patterns directly

---

## Part 1: Remote State Backend

### Step 1: Why Remote State Instead of Local
A local `terraform.tfstate` file only exists on your machine — nobody else can run `terraform plan` against the same infrastructure without it, and if your laptop dies, so does your only record of what Terraform manages. Remote state in S3, combined with DynamoDB for locking, solves both: state lives centrally, and the lock table prevents two people (or two CI runs) from applying simultaneously and corrupting state.

### Step 2: Create the Backend Resources
These need to exist *before* any Terraform config can use them as a backend — bootstrap them directly via CLI, not Terraform itself (a classic chicken-and-egg problem: you can't store Terraform's own state in a bucket Terraform hasn't created yet).

```bash
BUCKET_NAME="tf-state-$(aws sts get-caller-identity --query Account --output text)"

aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket $BUCKET_NAME \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws dynamodb create-table \
  --table-name tf-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

**Note the security defaults**: versioning (recover a previous state if something corrupts it), encryption at rest, and blocked public access — a Terraform state file often contains sensitive values (resource IDs, sometimes secrets if a module isn't careful), so it deserves the same hardening as any other sensitive data store.

---

## Part 2: Module Structure

### Step 3: Set Up the Project Layout

```bash
mkdir -p aws-tf-project/{modules/networking,modules/compute}
cd aws-tf-project
```

```
aws-tf-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
└── modules/
    ├── networking/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── compute/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

This mirrors the module structure from [Azure Lab 7](../../Azure/AZ-104/lab-7-terraform-modules.md) — networking and compute as separate, reusable modules instead of one monolithic file.

### Step 4: Write the Networking Module

`modules/networking/variables.tf`:
```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.1.0.0/16"
}

variable "project_name" {
  type = string
}
```

`modules/networking/main.tf`:
```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = { Name = "${var.project_name}-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.project_name}-igw" }
}

resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = { Name = "${var.project_name}-public-${count.index}" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${var.project_name}-public-rt" }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web" {
  name_prefix = "${var.project_name}-web-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project_name}-web-sg" }
}
```

`modules/networking/outputs.tf`:
```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "web_security_group_id" {
  value = aws_security_group.web.id
}
```

### Step 5: Write the Compute Module

`modules/compute/variables.tf`:
```hcl
variable "project_name" { type = string }
variable "vpc_id" { type = string }
variable "subnet_ids" { type = list(string) }
variable "security_group_id" { type = string }
variable "instance_type" {
  type    = string
  default = "t3.micro"
}
```

`modules/compute/main.tf`:
```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_launch_template" "web" {
  name_prefix   = "${var.project_name}-web-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  vpc_security_group_ids = [var.security_group_id]

  user_data = base64encode(<<-EOF
    #!/bin/bash
    dnf install -y nginx
    echo "<h1>Hello from Terraform-managed $(hostname -f)</h1>" > /usr/share/nginx/html/index.html
    systemctl enable nginx
    systemctl start nginx
  EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags           = { Name = "${var.project_name}-web" }
  }
}

resource "aws_autoscaling_group" "web" {
  name                = "${var.project_name}-asg"
  desired_capacity    = 2
  min_size            = 2
  max_size            = 4
  vpc_zone_identifier = var.subnet_ids

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "${var.project_name}-web"
    propagate_at_launch = true
  }
}
```

`modules/compute/outputs.tf`:
```hcl
output "asg_name" {
  value = aws_autoscaling_group.web.name
}
```

---

## Part 3: Root Module and Backend Configuration

### Step 6: Configure the Backend

`backend.tf`:
```hcl
terraform {
  backend "s3" {
    bucket         = "REPLACE_WITH_YOUR_BUCKET_NAME"
    key            = "aws-tf-project/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-state-lock"
    encrypt        = true
  }
}
```

> Replace `REPLACE_WITH_YOUR_BUCKET_NAME` with the `$BUCKET_NAME` value from Step 2.

### Step 7: Write the Root Module

`main.tf`:
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

module "networking" {
  source       = "./modules/networking"
  project_name = var.project_name
  vpc_cidr     = var.vpc_cidr
}

module "compute" {
  source             = "./modules/compute"
  project_name       = var.project_name
  vpc_id             = module.networking.vpc_id
  subnet_ids         = module.networking.public_subnet_ids
  security_group_id  = module.networking.web_security_group_id
  instance_type      = var.instance_type
}
```

`variables.tf`:
```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "project_name" {
  type    = string
  default = "aws-tf-lab8"
}

variable "vpc_cidr" {
  type    = string
  default = "10.1.0.0/16"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}
```

`outputs.tf`:
```hcl
output "vpc_id" {
  value = module.networking.vpc_id
}

output "asg_name" {
  value = module.compute.asg_name
}
```

### Step 8: Initialize and Apply

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

**Validation checkpoint**: `terraform output` shows a `vpc_id` and `asg_name`. Confirm in the console (**EC2** → **Auto Scaling Groups**) that instances launched and are healthy — same outcome as clicking through Lab 5's Part 1, produced by `terraform apply` this time.

---

## Part 4: Force a State Lock Conflict

### Step 9: Simulate a Stuck Lock
Open a second terminal in the same directory. In terminal 1, start an apply that will take a moment (e.g., after changing `desired_capacity` to force a change):

```bash
# Terminal 1
terraform apply
```

While it's running (or immediately after, using the lock ID from the state), in terminal 2:

```bash
# Terminal 2 — try to run another operation against the same state
terraform plan
```

**Expected result**: Terraform reports the state is locked, showing who holds the lock (DynamoDB `tf-state-lock` table entry) and when it was acquired.

### Step 10: Inspect the Lock Directly
```bash
aws dynamodb scan --table-name tf-state-lock
```

You'll see the lock item, including the operation and the `Info` field describing what's holding it.

### Step 11: Force-Unlock (Only When You're Certain Nothing Else Is Running)
```bash
terraform force-unlock <LOCK-ID-FROM-ERROR-MESSAGE>
```

**This is the same operational skill as [Azure Lab 8's Terraform lock exercise](../../Azure/AZ-104/lab-8-break-things.md)** — a CI pipeline killed mid-apply leaves a stuck lock behind regardless of which cloud's backend you're using, and knowing how to diagnose and safely recover is a cloud-agnostic skill.

---

## Part 5: Compare AWS and Azure Terraform Patterns

### Step 12: Note the Differences Directly
Now that you've built comparable infrastructure in both clouds, write down the concrete differences you hit:

| Aspect | AWS (`hashicorp/aws`) | Azure (`hashicorp/azurerm`) |
|--------|------------------------|-------------------------------|
| **State locking mechanism** | Separate DynamoDB table | Built into the Azure Storage blob lease — no second resource needed |
| **Resource container** | No equivalent to a Resource Group — resources exist directly in the account/region | Every resource belongs to an `azurerm_resource_group` |
| **AMI/image lookup** | Explicit `data "aws_ami"` query with filters | Image reference specified inline on the VM resource |
| **Auto-scaling resource** | `aws_autoscaling_group` references a separate `aws_launch_template` | `azurerm_linux_virtual_machine_scale_set` is more self-contained |

This table — filled in from firsthand experience rather than memorized — is exactly the kind of concrete comparison that makes a multi-cloud interview answer credible.

---

## Part 6: Cleanup

```bash
terraform destroy -auto-approve
```

Then remove the backend resources (only if you're done with this project entirely — otherwise leave them for future Terraform work):

```bash
aws dynamodb delete-table --table-name tf-state-lock

aws s3 rm s3://$BUCKET_NAME --recursive
aws s3api delete-bucket --bucket $BUCKET_NAME
```

---

## Key Concepts to Understand

| Concept | Definition |
|---------|-----------|
| **Remote state with locking** | S3 stores the state file; DynamoDB provides the lock so concurrent applies can't corrupt it |
| **Module composition** | Root module wires together reusable child modules via input variables and outputs, rather than one flat file |
| **`terraform force-unlock`** | Manually clears a stuck lock — a real operational skill, not just an exam topic, and identical in spirit across cloud backends |
| **AWS has no Resource Group equivalent** | Unlike Azure, AWS resources aren't grouped into a single deletable container by default — tagging (see [Lab 7](aws-lab-7-well-architected-cost.md)) is how you simulate that grouping for cost/cleanup purposes |

---

## Interview Prep: Common Questions

1. **"Why remote state instead of local state?"** — Team collaboration (shared source of truth), durability (survives a lost laptop), and locking (prevents concurrent-apply corruption).
2. **"How do you structure Terraform for reusability?"** — Modules with clear input variables and outputs, composed in a root module — the same networking module could deploy a second, differently-configured VPC by just calling it again with different variables.
3. **"What's a real operational issue you've hit with Terraform state?"** — The forced lock conflict in Part 4, and the diagnosis/recovery process, is a concrete answer instead of a textbook one.
4. **"How does AWS's Terraform provider differ from Azure's?"** — Reference the comparison table you built firsthand in Part 5.

---

## Next Steps
- Add a `staging`/`production` workspace split using Terraform workspaces or separate state keys
- Wire this project into a GitHub Actions pipeline running `terraform plan` on pull requests and `terraform apply` on merge to main
- This closes out the SAA-C03 track's new labs — continue to [AWS SCS-C02](../SCS-C02/README.md) to build the equivalent security-specialty depth on top of this same AWS foundation
