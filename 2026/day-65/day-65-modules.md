# Day 65 – Terraform Modules: Build Reusable Infrastructure

## Overview

Writing everything in one big `main.tf` works for learning, but real teams manage dozens of environments with hundreds of resources. **Modules** are the way to package, reuse, and share infrastructure code — write once, call many times, like functions in programming.

---

## Task 1: Understand Module Structure

```
terraform-modules/
├── main.tf              # Root module -- calls child modules
├── variables.tf
├── outputs.tf
├── providers.tf
├── locals.tf
└── modules/
    ├── ec2-instance/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── README.md
    └── security-group/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── README.md
```

### Root Module vs Child Module

|  | Root Module | Child Module |
| --- | --- | --- |
| Location | Where you run `terraform apply` | Called via `module {}` block |
| Has provider config? | Yes | No — inherits from root |
| Has backend config? | Yes | No |
| Called by | Nothing — entry point | Root or another module |
| Reusable? | No | Yes — call multiple times with different inputs |

---

## Task 2: Build a Custom EC2 Module

```hcl
# modules/ec2-instance/variables.tf
variable "ami_id" {
  type = string
}

variable "subnet_cidr" {
  description = "Public subnet CIDR"
  type        = string
  default     = "10.0.1.0/24"
}

variable "instance_type"{
  type = string
  default = "t2.micro"
}
variable "subnet_id"          { type = string }
variable "security_group_ids" { type = list(string) }
variable "instance_name"      { type = string }
variable "tags"               {
  type = map(string)
   default = {}
}
variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "environment" {

  type = string
  default =  "dev"
}

variable "project_name" {
  description = "Project name prefix for all resources"
  type        = string
  default     = "Terraform-modules-project"
  # No default -- Terraform will PROMPT for this value
}

```

```hcl
# modules/ec2-instance/main.tf
locals {
  name_prefix = var.project_name

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.subnet_cidr
  availability_zone = data.aws_availability_zones.available.names[0]  # first AZ
  # ...
}

resource "aws_instance" "this" {

  ami = var.ami_id
  instance_type               = var.instance_type
  subnet_id                   = var.subnet_id
  vpc_security_group_ids      = var.security_group_ids
  associate_public_ip_address = true

  tags = merge(var.tags, {
    Name = var.instance_name
  })
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = { Name = "${var.project_name}-vpc" }
}


```

```hcl
# modules/ec2-instance/outputs.tf
output "instance_id" { value = aws_instance.this.id }
output "public_ip"   { value = aws_instance.this.public_ip }
output "private_ip"  { value = aws_instance.this.private_ip }
```

---

## Task 3: Build a Custom Security Group Module

```hcl
# modules/security-group/variables.tf
variable "vpc_id"         { type = string }
variable "sg_name"        { type = string }
variable "ingress_ports"  { type = list(number); default = [22, 80] }
variable "tags"           { type = map(string); default = {} }
```

```hcl
# modules/security-group/main.tf
resource "aws_security_group" "this" {
  name   = var.sg_name
  vpc_id = var.vpc_id

  # dynamic block: generates one ingress rule per port in the list
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

  tags = merge(var.tags, { Name = var.sg_name })
}
```

```hcl
# modules/security-group/outputs.tf
output "sg_id" { value = aws_security_group.this.id }
```

### How `dynamic` Blocks Work

```hcl
# Without dynamic — repetitive, not reusable
ingress { from_port = 22  ... }
ingress { from_port = 80  ... }

# With dynamic — loops over a list
dynamic "ingress" {
  for_each = [22, 80, 443]
  content {
    from_port = ingress.value
    to_port   = ingress.value
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---
## Task 4: Call Your Modules from Root

```hcl
# root main.tf
# Fetch the latest Amazon Linux 2 AMI dynamically
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

# Fetch available AZs in the current region
data "aws_availability_zones" "available" {
  state = "available"
}



locals {
  name_prefix = "${var.project_name}-${var.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.subnet_cidr
  availability_zone = data.aws_availability_zones.available.names[0]  # first AZ
  # ...
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

module "web_sg" {

  source = "./modules/security-group"
  vpc_id = aws_vpc.main.id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = local.common_tags
}

module "web_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = aws_subnet.public.id
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-web"
  tags               = local.common_tags
}

module "api_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = aws_subnet.public.id
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-api"
  tags               = local.common_tags
}


```

```hcl
# root outputs.tf
output "web_server_ip" { value = module.web_server.public_ip }
output "api_server_ip" { value = module.api_server.public_ip }
```

```bash
terraform init    # links local modules
terraform plan
terraform apply

terraform state list
# module.web_server.aws_instance.this
# module.api_server.aws_instance.this
# module.web_sg.aws_security_group.this
```
<img width="1525" height="527" alt="image" src="https://github.com/user-attachments/assets/ddd6bc80-ebb2-45dc-b1c7-51439290d782" />

---
## Task 5: Use a Public Registry Module

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${local.name_prefix}-vpc"
  cidr = var.vpc_cidr

  azs             = ["ap-south-1a", "ap-south-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway   = false
  enable_dns_hostnames = true
  tags                 = local.common_tags
}

# Reference outputs
module "web_sg"    { vpc_id   = module.vpc.vpc_id }
module "web_server" { subnet_id = module.vpc.public_subnets[0] }
```

```bash
terraform init     # downloads to .terraform/modules/
terraform plan && terraform apply
ls .terraform/modules/
# vpc/  web_server/  api_server/  web_sg/
```

### Hand-Written VPC vs Registry Module

|  | Day 62 hand-written | terraform-aws-modules/vpc |
| --- | --- | --- |
| Resources created | 5 | 20+ (multi-AZ, NACLs, etc.) |
| Lines of HCL | ~60 | ~15 (the module call) |
| Multi-AZ support | Manual | Built-in |
| Best practices | You implement | Pre-baked |

---

## Task 6: Module Versioning and Best Practices

```hcl
version = "5.1.0"           # exact — guaranteed reproducible
version = "~> 5.0"          # any 5.x — safe, allows improvements
version = ">= 5.0, < 6.0"   # explicit range
```

```bash
terraform init -upgrade   # check for newer versions
terraform destroy
```

### Five Module Best Practices

| Practice | Why |
| --- | --- |
| Always pin registry module versions | Unpinned modules can change and break your apply |
| Keep modules focused — one concern per module | Single responsibility makes them reusable in different contexts |
| Use variables for everything, hardcode nothing | A module with hardcoded values is only usable once |
| Always define outputs | Callers need to reference your resources; without outputs the module is a dead end |
| Add [README.md](http://README.md) to every custom module | Document inputs, outputs, and usage examples |

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Module | A directory of `.tf` files; reusable unit of infrastructure |
| Root module | Entry point; has provider + backend config |
| Child module | Called with `module {}` block; no provider config |
| Local module | `source = "./modules/name"` |
| Registry module | `source = "org/name/provider"` — downloaded on `terraform init` |
| Module output reference | `module.<n>.<output_name>` |
| `dynamic` block | Loops over a list to generate repeated nested blocks |
| State prefix | `module.name.resource_type.resource_name` |
| `terraform init -upgrade` | Checks for newer module and provider versions |

---

## Quick Reference

```hcl
# Local module
module "name" {
  source   = "./modules/ec2-instance"
  variable = "value"
}

# Registry module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

# Reference output
subnet_id = module.vpc.public_subnets[0]
vpc_id    = module.vpc.vpc_id

# Dynamic block
dynamic "ingress" {
  for_each = var.ingress_ports
  content {
    from_port   = ingress.value
    to_port     = ingress.value
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

```bash
terraform init              # link/download modules
terraform init -upgrade     # check newer versions
terraform get               # download modules only
terraform state list        # see module.* prefixes
```
