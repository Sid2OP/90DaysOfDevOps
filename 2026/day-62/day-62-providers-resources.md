# Day 62 – Providers, Resources and Dependencies

## Overview

Yesterday you created standalone resources. But real infrastructure is connected — a server lives inside a subnet, a subnet inside a VPC, a security group controls traffic. Today you build a complete networking stack and learn how Terraform figures out what to create first.

---

## Task 1: Explore the AWS Provider

```bash
mkdir terraform-aws-infra && cd terraform-aws-infra
```

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

```bash
terraform init
# Installed hashicorp/aws v5.x.x

cat .terraform.lock.hcl
```

### Version Constraint Syntax

| Constraint | Meaning |
| --- | --- |
| `~> 5.0` | Allows `5.0`, `5.1`, `5.9` — but NOT `6.0`. Patch + minor updates only within major version. |
| `>= 5.0` | Any version 5.0 or higher — including 6.0, 7.0, etc. Risky. |
| `= 5.0.0` | Exactly this version, nothing else. |
| `>= 5.0, < 6.0` | Explicit range — same effect as `~> 5.0` but more verbose. |

**`~> 5.0` is the production standard** — you get bug fixes and new resources automatically, but breaking changes in a major version can't sneak in.

### What `.terraform.lock.hcl` Does

The lock file records the **exact** provider version installed (e.g., `5.43.0`) and its checksum hashes. On the next `terraform init`, Terraform installs that exact version — not just any `~> 5.0` version. This guarantees that your teammates and CI pipelines use identical provider binaries. Commit this file to Git.

---

## Task 2: Build a VPC from Scratch

```hcl
# main.tf

# 1. VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "TerraWeek-VPC"
  }
}

# 2. Subnet (references VPC -- implicit dependency)
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true

  tags = {
    Name = "TerraWeek-Public-Subnet"
  }
}

# 3. Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "TerraWeek-IGW"
  }
}

# 4. Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "TerraWeek-RT"
  }
}

# 5. Route Table Association
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

```bash
terraform plan
# Plan: 5 to add, 0 to change, 0 to destroy

terraform apply
```

---

## Task 3: Implicit Dependencies

### How Terraform Detects Order

When you write `vpc_id = aws_vpc.main.id`, Terraform sees a **reference** from one resource to another. It builds a dependency graph from all these references and creates resources in the correct order automatically — no manual ordering needed.

```
aws_vpc.main
  └── aws_subnet.public          (references aws_vpc.main.id)
  └── aws_internet_gateway.igw   (references aws_vpc.main.id)
       └── aws_route_table.public (references igw.id and vpc.id)
            └── aws_route_table_association.public (references RT and subnet)
```

### All Implicit Dependencies in This Config

| Resource | Depends on | Via |
| --- | --- | --- |
| `aws_subnet.public` | `aws_vpc.main` | `vpc_id = aws_vpc.main.id` |
| `aws_internet_gateway.igw` | `aws_vpc.main` | `vpc_id = aws_vpc.main.id` |
| `aws_route_table.public` | `aws_vpc.main` | `vpc_id = aws_vpc.main.id` |
| `aws_route_table.public` | `aws_internet_gateway.igw` | `gateway_id = aws_internet_gateway.igw.id` |
| `aws_route_table_association.public` | `aws_subnet.public` | `subnet_id = aws_subnet.public.id` |
| `aws_route_table_association.public` | `aws_route_table.public` | `route_table_id = aws_route_table.public.id` |

**What would happen if subnet was created before VPC?** The AWS API would reject it — a subnet cannot exist without a VPC. Terraform prevents this by resolving the graph first. If you somehow removed the reference and used a hardcoded ID that didn't exist yet, the apply would fail with an AWS API error.

---

## Task 4: Add a Security Group and EC2 Instance

```hcl
# Add to main.tf

resource "aws_security_group" "web_sg" {
  name        = "TerraWeek-SG"
  description = "Allow SSH and HTTP"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "SSH"
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound"
  }

  tags = {
    Name = "TerraWeek-SG"
  }
}

resource "aws_instance" "web" {
  ami                         = "ami-0f5ee92e2d63afc18"  # Amazon Linux 2, ap-south-1
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.web_sg.id]
  associate_public_ip_address = true

  tags = {
    Name = "TerraWeek-Server"
  }
}
```

```bash
terraform plan
# 2 to add: aws_security_group.web_sg, aws_instance.web

terraform apply

# Get the public IP
terraform output  # or:
terraform state show aws_instance.web | grep public_ip
```

---

## Task 5: Explicit Dependencies with `depends_on`

Sometimes Terraform cannot detect a dependency because there is no direct attribute reference between two resources.

```hcl
# Add to main.tf
resource "aws_s3_bucket" "app_logs" {
  bucket = "terraweek-app-logs-yourname-2026"

  tags = {
    Name = "TerraWeek-AppLogs"
  }

  depends_on = [aws_instance.web]  # no attribute reference exists
  # Terraform will create the EC2 instance BEFORE this bucket
}
```

```bash
terraform plan
# Terraform creates aws_instance.web first, then aws_s3_bucket.app_logs

# Visualize the full dependency graph
terraform graph | dot -Tpng > graph.png

# Without Graphviz installed -- paste DOT output here:
# https://webgraphviz.com
terraform graph
```

### When to Use `depends_on` in Real Projects

| Scenario | Why `depends_on` is needed |
| --- | --- |
| IAM role must exist before an EC2 instance profile that references it in a policy | No direct attribute reference between the two resources |
| An S3 bucket policy must wait for bucket replication configuration | The policy references the bucket by name (string), not by Terraform reference |
| A Lambda function must wait for its CloudWatch log group to exist | Lambda would auto-create the log group, breaking the managed one |
| Database must be ready before a migration script runs | The migration uses a `null_resource` with no attribute link to the DB resource |

---

## Task 6: Lifecycle Rules

```hcl
resource "aws_instance" "web" {
  ami                         = "ami-NEW-AMI-ID"  # changed AMI
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.web_sg.id]
  associate_public_ip_address = true

  tags = {
    Name = "TerraWeek-Server"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

```bash
terraform plan
# -/+ aws_instance.web (must be replaced)
# Plan: 1 to add, 0 to change, 1 to destroy
# With create_before_destroy: new instance created FIRST, then old one destroyed

terraform destroy
# Destroys in REVERSE dependency order:
# 1. aws_instance.web
# 2. aws_s3_bucket.app_logs
# 3. aws_security_group.web_sg
# 4. aws_route_table_association.public
# 5. aws_route_table.public
# 6. aws_internet_gateway.igw
# 7. aws_subnet.public
# 8. aws_vpc.main  (last -- everything else depends on it)
```

### Three Lifecycle Arguments

| Argument | What it does | When to use it |
| --- | --- | --- |
| `create_before_destroy = true` | Creates the replacement resource BEFORE destroying the old one | Stateful resources (EC2, RDS) where you want zero downtime during replacement |
| `prevent_destroy = true` | Blocks `terraform destroy` and any plan that would delete this resource | Production databases, S3 buckets with critical data -- adds a safety net |
| `ignore_changes = [tags]` | Ignores drift on specified attributes -- Terraform won't try to revert them | When other systems (e.g., AWS auto-tagging) modify resources and you don't want Terraform to fight them |

```hcl
# Example: protect a production database
resource "aws_db_instance" "prod" {
  # ... config ...
  lifecycle {
    prevent_destroy       = true
    ignore_changes        = [tags, latest_restorable_time]
  }
}
```

---

## Resource Reference Syntax

```
<resource_type>.<resource_name>.<attribute>

aws_vpc.main.id
aws_subnet.public.id
aws_security_group.web_sg.id
aws_instance.web.public_ip
aws_s3_bucket.app_logs.arn
```

All available attributes for a resource are listed in the Terraform AWS provider documentation under each resource type.

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| `~> 5.0` | Allows minor + patch updates within major version; production standard |
| `.terraform.lock.hcl` | Pins exact provider version + checksum; commit to Git |
| Implicit dependency | Terraform detects from attribute references between resources |
| Explicit dependency | `depends_on` used when no attribute reference exists between resources |
| Dependency graph | Terraform builds a DAG and creates resources in topological order |
| Destroy order | Always reverse of creation order |
| `create_before_destroy` | New resource created first; prevents downtime during replacement |
| `prevent_destroy` | Blocks accidental deletion; safety net for critical resources |
| `ignore_changes` | Ignores external modifications to specified attributes |
| `terraform graph` | Outputs DOT format dependency graph; visualize at [webgraphviz.com](http://webgraphviz.com) |

---

## Quick Reference

```hcl
# Reference another resource
vpc_id = aws_vpc.main.id

# Explicit dependency
depends_on = [aws_instance.web]

# Lifecycle block
lifecycle {
  create_before_destroy = true
  prevent_destroy       = true
  ignore_changes        = [tags, ami]
}
```

```bash
# Visualize dependency graph
terraform graph | dot -Tpng > graph.png
terraform graph   # paste at webgraphviz.com

# Destroy one resource only
terraform destroy -target=aws_instance.web

# Destroy everything
terraform destroy
```

<img width="1532" height="798" alt="image" src="https://github.com/user-attachments/assets/ed9aa5b6-7897-4d00-a68e-b99a8683ea4b" />
