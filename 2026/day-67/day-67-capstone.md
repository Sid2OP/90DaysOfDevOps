# Day 67 – TerraWeek Capstone: Multi-Environment Infrastructure with Workspaces and Modules

## Overview

Seven days of Terraform come together in one production-grade project. One codebase, three environments (dev, staging, prod) using **Terraform workspaces** and custom modules. This is how infrastructure teams operate at scale.

---

## Task 1: Learn Terraform Workspaces

```bash
mkdir terraweek-capstone && cd terraweek-capstone
terraform init

terraform workspace show          # default

terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

terraform workspace list
# * default
#   dev
#   staging
#   prod

terraform workspace select dev
terraform workspace select staging
terraform workspace select prod
```

### Workspace Internals

`terraform.workspace` is a built-in variable available anywhere in your config. When you're on the `dev` workspace, `terraform.workspace` returns `"dev"`.

**State file locations:**

```
Local:   .terraform.tfstate.d/<workspace>/terraform.tfstate
S3:      env:/<workspace>/terraform.tfstate
```

### Workspaces vs Separate Directories

|  | Workspaces | Separate directories |
| --- | --- | --- |
| Codebase | Single shared codebase | One copy per environment |
| State | Isolated per workspace | Isolated per directory |
| Risk | Accidental wrong workspace | Accidental wrong directory |
| Best for | Similar environments, same config | Radically different configs per env |

---

## Task 2: Project Structure

```
terraweek-capstone/
├── main.tf                # Root module
├── variables.tf
├── outputs.tf
├── providers.tf
├── locals.tf              # Workspace-aware locals
├── dev.tfvars
├── staging.tfvars
├── prod.tfvars
├── .gitignore
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── security-group/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ec2-instance/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

```bash
# .gitignore
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
.terraform.lock.hcl
```

**Why this structure is best practice:** Each file has a single responsibility. `locals.tf` is the only place workspace logic lives. Modules are portable and reusable. `.tfvars` are gitignored so secrets don't leak. State is never committed.

---

## Task 3: Custom Modules

### Module 1: vpc

```hcl
# modules/vpc/variables.tf
variable "cidr"               { type = string }
variable "public_subnet_cidr" { type = string }
variable "environment"        { type = string }
variable "project_name"       { type = string }
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block = var.cidr
  tags = { Name = "${var.project_name}-${var.environment}-vpc", Environment = var.environment }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidr
  map_public_ip_on_launch = true
  tags = { Name = "${var.project_name}-${var.environment}-subnet", Environment = var.environment }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.this.id
  tags   = { Name = "${var.project_name}-${var.environment}-igw" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  route { cidr_block = "0.0.0.0/0"; gateway_id = aws_internet_gateway.igw.id }
  tags = { Name = "${var.project_name}-${var.environment}-rt" }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id"    { value = aws_vpc.this.id }
output "subnet_id" { value = aws_subnet.public.id }
```

### Module 2: security-group

```hcl
# modules/security-group/variables.tf
variable "vpc_id" {
  description = "VPC ID to create the security group in"
  type        = string
}

variable "ingress_ports" {
  description = "List of TCP ports to allow inbound"
  type        = list(number)
  default     = [22, 80]
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}

variable "project_name" {
  description = "Project name used in resource naming"
  type        = string
}
```

```hcl
# modules/security-group/main.tf
resource "aws_security_group" "this" {
  name        = "${var.project_name}-${var.environment}-sg"
  description = "Security group for ${var.environment} environment"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "Port ${ingress.value}"
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-${var.environment}-sg"
    Environment = var.environment
    Project     = var.project_name
  }
}
```

```hcl
# modules/security-group/outputs.tf
output "sg_id" { value = aws_security_group.this.id }
```

### Module 3: ec2-instance

```hcl
# modules/ec2-instance/variables.tf
variable "ami_id" {
  description = "AMI ID for the EC2 instance"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "subnet_id" {
  description = "Subnet ID to launch the instance in"
  type        = string
}

variable "security_group_ids" {
  description = "List of security group IDs to attach"
  type        = list(string)
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}

variable "project_name" {
  description = "Project name used in resource naming"
  type        = string
}
```

```hcl
# modules/ec2-instance/main.tf
resource "aws_instance" "this" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
  subnet_id                   = var.subnet_id
  vpc_security_group_ids      = var.security_group_ids
  associate_public_ip_address = true

  tags = {
    Name        = "${var.project_name}-${var.environment}-server"
    Environment = var.environment
    Project     = var.project_name
  }
}
```

```hcl
# modules/ec2-instance/outputs.tf
output "instance_id" { value = aws_instance.this.id }
output "public_ip"   { value = aws_instance.this.public_ip }
```

```bash
terraform validate   # validate all modules
```

---

## Task 4: Workspace-Aware Root Config

```hcl
# locals.tf
locals {
  environment = terraform.workspace
  name_prefix = "${var.project_name}-${local.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = local.environment
    ManagedBy   = "Terraform"
    Workspace   = terraform.workspace
  }
}
```

```hcl
# main.tf
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter { name = "name"; values = ["amzn2-ami-hvm-*-x86_64-gp2"] }
}

module "vpc" {
  source            = "./modules/vpc"
  cidr              = var.vpc_cidr
  public_subnet_cidr = var.subnet_cidr
  environment       = local.environment
  project_name      = var.project_name
}

module "sg" {
  source        = "./modules/security-group"
  vpc_id        = module.vpc.vpc_id
  ingress_ports = var.ingress_ports
  environment   = local.environment
  project_name  = var.project_name
}

module "server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = var.instance_type
  subnet_id          = module.vpc.subnet_id
  security_group_ids = [module.sg.sg_id]
  environment        = local.environment
  project_name       = var.project_name
}
```

### Environment tfvars

```hcl
# dev.tfvars
vpc_cidr      = "10.0.0.0/16"
subnet_cidr   = "10.0.1.0/24"
instance_type = "t2.micro"
ingress_ports = [22, 80]       # SSH allowed in dev
```

```hcl
# staging.tfvars
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
instance_type = "t2.small"
ingress_ports = [22, 80, 443]
```

```hcl
# prod.tfvars
vpc_cidr      = "10.2.0.0/16"
subnet_cidr   = "10.2.1.0/24"
instance_type = "t3.small"
ingress_ports = [80, 443]      # NO SSH in prod
```

Different VPC CIDRs prevent accidental peering conflicts. Instance types scale up per environment. Dev allows SSH; prod does not.

---

## Task 5: Deploy All Three Environments

```bash
# Dev
terraform workspace select dev
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"

# Staging
terraform workspace select staging
terraform plan -var-file="staging.tfvars"
terraform apply -var-file="staging.tfvars"

# Prod
terraform workspace select prod
terraform plan -var-file="prod.tfvars"
terraform apply -var-file="prod.tfvars"

# Verify all three
terraform workspace select dev     && terraform output
terraform workspace select staging && terraform output
terraform workspace select prod    && terraform output
```

**Verify in AWS console:**

- Three VPCs: `10.0.0.0/16`, `10.1.0.0/16`, `10.2.0.0/16`
- Three EC2 instances: `terraweek-dev-server`, `terraweek-staging-server`, `terraweek-prod-server`
- Different instance types per environment

---
<img width="1495" height="770" alt="image" src="https://github.com/user-attachments/assets/e2bcb997-54a5-428b-aa80-2cec4302f2b3" />

<img width="1498" height="707" alt="image" src="https://github.com/user-attachments/assets/2dce3f78-163c-4662-8751-e1fbff75b65f" />

<img width="1520" height="641" alt="image" src="https://github.com/user-attachments/assets/80cbb15c-83d8-4327-9d57-46269085a5e1" />

<img width="1608" height="367" alt="1" src="https://github.com/user-attachments/assets/117dcc44-ddd9-4850-9f12-2004ce93f152" />

<img width="1608" height="367" alt="2" src="https://github.com/user-attachments/assets/723ca4bb-afc7-4aba-9027-4ae355932174" />


## Task 6: Terraform Best Practices Guide

| Category | Best Practice |
| --- | --- |
| **File structure** | Separate files: `providers.tf`, `variables.tf`, `outputs.tf`, `main.tf`, `locals.tf` |
| **State** | Always use remote backend (S3 + DynamoDB); enable versioning and encryption |
| **Variables** | Never hardcode; use `.tfvars` per environment; add `validation {}` blocks for constraints |
| **Modules** | One concern per module; always define inputs/outputs; pin registry versions |
| **Workspaces** | Use for environment isolation; reference `terraform.workspace` in locals |
| **Security** | `.gitignore` state and `.tfvars`; encrypt state at rest; restrict backend access |
| **Workflow** | Always `plan` before `apply`; run `fmt` and `validate` before committing |
| **Tagging** | Tag every resource with `Project`, `Environment`, `ManagedBy` |
| **Naming** | Consistent pattern: `<project>-<environment>-<resource>` |
| **Cleanup** | `terraform destroy` non-production environments when not in use |

---

## Task 7: Destroy All Environments

```bash
# Destroy in reverse order (prod → staging → dev)
terraform workspace select prod
terraform destroy -var-file="prod.tfvars"

terraform workspace select staging
terraform destroy -var-file="staging.tfvars"

terraform workspace select dev
terraform destroy -var-file="dev.tfvars"

# Delete workspaces (must be on default first)
terraform workspace select default
terraform workspace delete dev
terraform workspace delete staging
terraform workspace delete prod

# Verify
terraform workspace list
# * default
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| `terraform.workspace` | Built-in variable; returns current workspace name |
| Workspace state isolation | Each workspace has its own state file; resources are completely separate |
| `-var-file` | Must be specified explicitly; doesn't auto-load `terraform.tfvars` |
| `terraform workspace show` | Check which workspace you're on before any apply |
| Different CIDRs per env | Prevents accidental VPC peering conflicts between environments |
| Can't delete current workspace | Switch to `default` first |
| `locals.tf` | Single place for workspace-derived values; keeps `main.tf` clean |

---

## Quick Reference

```bash
# Workspace management
terraform workspace show
terraform workspace list
terraform workspace new <name>
terraform workspace select <name>
terraform workspace delete <name>   # must not be current

# Deploy per environment
terraform workspace select dev
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"
terraform destroy -var-file="dev.tfvars"

# Validate before applying
terraform fmt
terraform validate
terraform plan -var-file="<env>.tfvars"
```
