# Day 63 – Variables, Outputs, Data Sources and Expressions

## Overview

Your Day 62 config works but is full of hardcoded values — region, CIDR blocks, AMI IDs, instance types. Today you make your Terraform configs **dynamic, reusable, and environment-aware** using variables, outputs, data sources, locals, and expressions.

---

## Task 1: Extract Variables

```hcl
# variables.tf
variable "region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_cidr" {
  description = "Public subnet CIDR"
  type        = string
  default     = "10.0.1.0/24"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "project_name" {
  description = "Project name prefix for all resources"
  type        = string
  # No default -- Terraform will PROMPT for this value
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

variable "allowed_ports" {
  description = "Ports to allow inbound"
  type        = list(number)
  default     = [22, 80, 443]
}

variable "extra_tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

```hcl
# main.tf -- using variables
provider "aws" {
  region = var.region
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = { Name = "${var.project_name}-vpc" }
}

resource "aws_instance" "web" {
  instance_type = var.instance_type
  # ...
}
```

### The Five Variable Types

| Type | Example | Use case |
| --- | --- | --- |
| `string` | `"t2.micro"` | Region, instance type, names |
| `number` | `3` | Replica counts, port numbers |
| `bool` | `true` | Feature flags, enable/disable |
| `list(type)` | `[22, 80, 443]` | Multiple ports, AZ lists |
| `map(type)` | `{env = "dev"}` | Tags, key-value config |

---
<img width="1496" height="801" alt="image" src="https://github.com/user-attachments/assets/addb9c2a-965f-4f5a-aeb8-29d3b160585d" />

## Task 2: Variable Files and Precedence

```hcl
# terraform.tfvars  (auto-loaded)
project_name  = "terraweek"
environment   = "dev"
instance_type = "t2.micro"
```

```hcl
# prod.tfvars
project_name  = "terraweek"
environment   = "prod"
instance_type = "t3.small"
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
```

```bash
# Uses terraform.tfvars automatically
terraform plan

# Uses prod.tfvars
terraform plan -var-file="prod.tfvars"

# CLI flag overrides everything
terraform plan -var="instance_type=t2.nano"

# Environment variable
export TF_VAR_environment="staging"
terraform plan
```

### Variable Precedence (lowest → highest)

| Priority | Source |
| --- | --- |
| 1 (lowest) | `default` value in `variable {}` block |
| 2 | `terraform.tfvars` (auto-loaded) |
| 3 | `*.auto.tfvars` files (auto-loaded, alphabetically) |
| 4 | `-var-file="file.tfvars"` flag |
| 5 | `-var="key=value"` CLI flag |
| 6 (highest) | `TF_VAR_<name>` environment variables |

Higher priority always wins. `-var` flag overrides everything including `.tfvars` files.

---
<img width="1438" height="705" alt="image" src="https://github.com/user-attachments/assets/7b2a6bdf-fc91-4daa-b3c9-d484ef19bff6" />

## Task 3: Add Outputs

```hcl
# outputs.tf
output "vpc_id" {
  description = "The VPC ID"
  value       = aws_vpc.main.id
}

output "subnet_id" {
  description = "Public subnet ID"
  value       = aws_subnet.public.id
}

output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "instance_public_dns" {
  description = "Public DNS name"
  value       = aws_instance.web.public_dns
}

output "security_group_id" {
  description = "Security group ID"
  value       = aws_security_group.web_sg.id
}
```

```bash
terraform apply
# Outputs printed automatically at the end

terraform output                       # show all outputs
terraform output instance_public_ip    # show specific output
terraform output -json                 # JSON format for scripting
terraform output -raw instance_public_ip  # raw value (no quotes)
```

---
<img width="1318" height="757" alt="image" src="https://github.com/user-attachments/assets/834dafa2-76fb-47d9-a49b-2fbd39441277" />

## Task 4: Use Data Sources

Data sources are **read-only** — they fetch existing information from AWS, they don't create anything.

```hcl
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

# Use in resources
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id          # dynamic AMI
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id
  # ...
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.subnet_cidr
  availability_zone = data.aws_availability_zones.available.names[0]  # first AZ
  # ...
}
```

### Resource vs Data Source

|  | Resource | Data Source |
| --- | --- | --- |
| **Keyword** | `resource` | `data` |
| **Reference** | `aws_instance.web.id` | `data.aws_ami.amazon_linux.id` |
| **Creates anything?** | Yes | No -- read only |
| **Tracked in state?** | Yes | No |
| **Destroyed on `terraform destroy`?** | Yes | No |
| **Use case** | Create and manage infrastructure | Look up existing info (AMIs, AZs, VPCs not managed by this config) |

---

## Task 5: Locals for Dynamic Values

Locals are computed values defined once and reused. They reduce repetition and make changes easier.

```hcl
# locals block (usually at top of main.tf or in a locals.tf)
locals {
  name_prefix = "${var.project_name}-${var.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Use locals in resources
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

resource "aws_instance" "web" {
  # ...
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-server"
    Role = "webserver"
  })
}
```

`merge()` combines two maps — keys in the second map override the first. Every resource gets `Project`, `Environment`, `ManagedBy` automatically plus its own `Name`.

---

## Task 6: Built-in Functions and Conditionals

```bash
# Interactive REPL to test expressions
terraform console
```

### String Functions

```hcl
upper("terraweek")                          # "TERRAWEEK"
lower("PROD")                               # "prod"
join("-", ["terra", "week", "2026"])        # "terra-week-2026"
split(",", "a,b,c")                         # ["a", "b", "c"]
format("arn:aws:s3:::%s", "my-bucket")     # full ARN string
trimspace("  hello  ")                      # "hello"
replace("hello-world", "-", "_")           # "hello_world"
```

### Collection Functions

```hcl
length(["a", "b", "c"])                     # 3
toset(["a", "b", "a"])                      # {"a", "b"} -- deduplicates
lookup({dev = "t2.micro", prod = "t3.small"}, "dev", "t2.micro")  # "t2.micro"
merge({a = 1}, {b = 2})                     # {a=1, b=2}
flat(["a", ["b", "c"]])                    # ["a", "b", "c"]
contains(["a", "b", "c"], "b")             # true
```

### Networking Function

```hcl
cidrsubnet("10.0.0.0/16", 8, 1)            # "10.0.1.0/24"
cidrsubnet("10.0.0.0/16", 8, 2)            # "10.0.2.0/24"
# Great for generating subnet CIDRs dynamically from a VPC CIDR
```

### Conditional Expression

```hcl
# Syntax: condition ? true_value : false_value
instance_type = var.environment == "prod" ? "t3.small" : "t2.micro"

# In a tag
enabled = var.environment == "prod" ? "true" : "false"

# Nested
instance_type = var.environment == "prod" ? "t3.large" : var.environment == "staging" ? "t3.small" : "t2.micro"
```

### Five Most Useful Functions

| Function | What it does |
| --- | --- |
| `merge(map1, map2)` | Combines two maps; second map wins on conflict. Essential for tag merging. |
| `cidrsubnet(cidr, bits, index)` | Calculates a subnet CIDR from a parent. Eliminates hardcoding subnet CIDRs. |
| `lookup(map, key, default)` | Safe map lookup with fallback. Use for environment-to-instance-type mappings. |
| `join(sep, list)` | Joins a list into a string. Useful for building ARNs, IAM policy conditions. |
| `toset(list)` | Converts list to set (deduplicates). Required for `for_each` over lists with potential duplicates. |

---

## File Structure for a Parameterized Config

```
terraform-aws-infra/
├── providers.tf        # terraform block + provider
├── variables.tf        # all input variable declarations
├── locals.tf           # computed local values
├── main.tf             # all resources
├── data.tf             # all data sources
├── outputs.tf          # all outputs
├── terraform.tfvars    # default values (dev) -- auto-loaded
├── prod.tfvars         # production overrides
├── .terraform.lock.hcl # committed to git
└── .gitignore          # excludes *.tfstate, .terraform/
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| `variable` | Input parameter; no default = required; `var.name` to reference |
| `terraform.tfvars` | Auto-loaded; contains actual values for variables |
| Variable precedence | default < tfvars < auto.tfvars < -var-file < -var < TF_VAR_* |
| `output` | Prints values after apply; accessible via `terraform output` |
| `data` source | Read-only lookup; never creates or destroys; `data.type.name.attr` |
| `locals` | Computed values defined once, reused everywhere; `local.name` |
| `merge()` | Combines maps; essential for consistent tagging |
| `cidrsubnet()` | Dynamic subnet calculation from parent CIDR |
| Conditional `? :` | Ternary expression for environment-aware config |
| `terraform console` | Interactive REPL for testing functions and expressions |

---

## Quick Reference

```bash
# Apply with different environments
terraform plan                          # uses terraform.tfvars
terraform plan -var-file="prod.tfvars"
terraform plan -var="environment=staging"
export TF_VAR_environment="staging" && terraform plan

# Outputs
terraform output
terraform output instance_public_ip
terraform output -json
terraform output -raw instance_public_ip

# Interactive console
terraform console
> upper("hello")
> cidrsubnet("10.0.0.0/16", 8, 1)
> merge({a="1"}, {b="2"})
```

```hcl
# Reference cheat sheet
var.name                                    # input variable
local.name                                  # local value
data.aws_ami.amazon_linux.id               # data source attribute
aws_instance.web.public_ip                 # resource attribute
"${var.project}-${var.environment}"        # string interpolation
var.env == "prod" ? "t3.small" : "t2.micro" # conditional
```
