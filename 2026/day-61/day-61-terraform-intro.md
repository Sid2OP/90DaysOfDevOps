# Day 61 – Introduction to Terraform and Your First AWS Infrastructure

## Overview

You've been deploying containers, writing CI/CD pipelines, and orchestrating workloads on Kubernetes. But who creates the servers, networks, and clusters underneath? Today you start your **Infrastructure as Code** journey with Terraform — the tool that lets you define, provision, and manage cloud infrastructure by writing code.

---

## Task 1: Understanding Infrastructure as Code

### What is IaC and Why Does It Matter?

Infrastructure as Code means treating your servers, networks, databases, and cloud resources the same way you treat application code — written in files, version-controlled in Git, reviewed in PRs, and deployed automatically. Instead of clicking through the AWS console to create a VPC, you write a `.tf` file that describes what you want, and Terraform makes it happen.

In DevOps, IaC matters because infrastructure becomes **reproducible, auditable, and automated**. The same code that creates your staging environment creates production — no more "works on my machine" for infrastructure.

### Problems IaC Solves vs Manual Console Work

| Manual Console | IaC (Terraform) |
| --- | --- |
| No record of what was created or why | Git history shows every change, who made it, when |
| Can't reproduce exactly — human error guaranteed | Run `terraform apply` and get identical infra every time |
| Deleting resources is risky and manual | `terraform destroy` cleans up everything it created |
| Staging ≠ Production (configuration drift) | Same code, same result, every environment |
| No review process | PRs, CI checks, and plan reviews before changes apply |

### Terraform vs Other IaC Tools

| Tool | Approach | Scope | Who makes it |
| --- | --- | --- | --- |
| **Terraform** | Declarative, cloud-agnostic HCL | Any cloud + SaaS | HashiCorp |
| **CloudFormation** | Declarative, AWS-only JSON/YAML | AWS only | AWS |
| **Ansible** | Procedural (how to do it), agentless | Config mgmt, not infra | Red Hat |
| **Pulumi** | Imperative using real languages (Python, Go, TS) | Any cloud | Pulumi Corp |

### Declarative and Cloud-Agnostic

**Declarative** means you describe *what* you want, not *how* to create it. You write `resource "aws_instance" "web" { instance_type = "t2.micro" }` and Terraform figures out the API calls, ordering, and dependencies.

**Cloud-agnostic** means the same Terraform workflow (`init -> plan -> apply`) works for AWS, GCP, Azure, Kubernetes, GitHub, Datadog, or any of 3,000+ providers. You only change the provider block and resource types.

---

## Task 2: Install Terraform and Configure AWS

```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Linux (amd64)
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Windows
choco install terraform

# Verify
terraform -version
terraform env
```

```bash
# Configure AWS CLI
aws configure
# AWS Access Key ID:     <your key>
# AWS Secret Access Key: <your secret>
# Default region:        ap-south-1
# Output format:         json

# Verify
aws sts get-caller-identity
# Returns: Account ID, UserId, ARN
```

---

## Task 3: Your First Terraform Config -- S3 Bucket

```bash
mkdir terraform-basics && cd terraform-basics
```

```hcl
# main.tf
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

resource "aws_s3_bucket" "my_bucket" {
  bucket = "terraweek-yourname-2026"  # must be globally unique

  tags = {
    Name        = "TerraWeek Bucket"
    Environment = "Dev"
  }
}
```

```bash
terraform init      # downloads the AWS provider plugin
terraform fmt       # auto-format .tf files
terraform validate  # check syntax without connecting to AWS
terraform plan      # preview what will be created
terraform apply     # create the bucket (type 'yes')
```

### What `terraform init` Downloads

`terraform init` reads `required_providers` and downloads the provider binary into `.terraform/`. It also creates `.terraform.lock.hcl` which pins exact provider versions.

```
terraform-basics/
├── main.tf
├── .terraform/
│   └── providers/
│       └── registry.terraform.io/hashicorp/aws/5.x.x/
│           └── terraform-provider-aws_v5.x.x
├── .terraform.lock.hcl
└── terraform.tfstate        # created after first apply
```

---

## Task 4: Add an EC2 Instance

```hcl
# Add to main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

resource "aws_s3_bucket" "my-bucket" {
  bucket = "sid-terra-2op"
  tags = {
    name = "terrweek bucket"
    env  = "Dev"
  }
}

resource "aws_instance" "web" {

  tags = {
    Name = "terra-automate-server"
  }
  ami = "ami-0d76b909de1a0595d"

  instance_type = "t3.micro"

  key_name = aws_key_pair.my_key_pair.key_name

  vpc_security_group_ids = [aws_security_group.my_security_group.id]

  root_block_device {
volume_size = 10
    volume_type = "gp3"
  }



}

resource "aws_key_pair" "my_key_pair" {

  key_name   = "terra-automate-key"
  public_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEAKFS4oqOgYNZTY/KDk4gw9Nz2aUGm6wa7Soppi4GeJ sid@sid-VMware-Virtual-Platform"
}


resource "aws_default_vpc" "default" {
}


resource "aws_security_group" "my_security_group" {

  name        = "terra-security-group"
  vpc_id      = aws_default_vpc.default.id
  description = "This is inbound and outbound rules"

}
resource "aws_vpc_security_group_ingress_rule" "allow_http" {
  security_group_id = aws_security_group.my_security_group.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 80
  ip_protocol       = "tcp"
  to_port           = 80
}

resource "aws_vpc_security_group_egress_rule" "allow_all_traffic" {
  security_group_id = aws_security_group.my_security_group.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"

}

```

```bash
terraform plan
# Output:
#   ~ aws_s3_bucket.my_bucket  (no changes -- already in state)
#   + aws_instance.web         (will be created)

terraform apply
```

### How Terraform Knows What Already Exists

Terraform compares **desired state** (your `.tf` files) against **current state** (`terraform.tfstate`). After the first apply, the state file records every resource created. On the next plan, Terraform reads the state file and only shows differences -- the S3 bucket is already tracked, so no changes needed.

---

## Task 5: Understand the State File

```bash
terraform show                                    # human-readable state
terraform state list                              # list all managed resources
terraform state show aws_s3_bucket.my_bucket      # one resource in detail
terraform state show aws_instance.web
```

### What the State File Stores

For each resource, `terraform.tfstate` records:

- Terraform resource address (`aws_instance.web`)
- Every attribute returned by AWS after creation (instance ID, private IP, public IP, ARN, AZ, etc.)
- Provider used and schema version
- Dependencies between resources

### Critical State File Rules

**Never manually edit the state file.** Terraform uses internal IDs and checksums. Manual edits corrupt state, causing phantom creates or destroys.

**Never commit `terraform.tfstate` to Git.** It contains sensitive data (passwords, keys, IPs). In production, store state remotely in S3 + DynamoDB locking.

```bash
# .gitignore for all Terraform projects
*.tfstate
*.tfstate.backup
.terraform/
```

---

## Task 6: Modify, Plan, and Destroy

```hcl
# Change the tag in main.tf
tags = {
  Name = "TerraWeek-Modified"
}
```

```bash
terraform plan
# ~ aws_instance.web  (in-place update, tag only)

terraform apply
```

### Plan Output Symbols

| Symbol | Meaning |
| --- | --- |
| `+` | Resource will be **created** |
| `-` | Resource will be **destroyed** |
| `~` | Resource will be **updated in-place** |
| `-/+` | Resource will be **destroyed and recreated** |

A tag change is `~` (in-place). Changing the AMI would be `-/+` (destroy + recreate). Terraform tells you which it is before you apply.

```bash
# Destroy everything
terraform destroy
# Shows destruction plan, type 'yes' to confirm
# Both S3 bucket and EC2 instance are gone
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| IaC | Infrastructure in code -- version-controlled, reproducible, auditable |
| Declarative | Describe *what* you want; Terraform figures out *how* |
| Cloud-agnostic | Same workflow for AWS, GCP, Azure, Kubernetes, 3,000+ providers |
| `terraform init` | Downloads provider plugins; creates lock file |
| `terraform plan` | Previews changes without applying; always run before apply |
| `terraform apply` | Applies changes; updates state file |
| `terraform destroy` | Destroys all managed resources |
| State file | Source of truth; never edit manually; never commit to git |
| `~` in plan | In-place update (no destroy) |
| `-/+` in plan | Destroy + recreate required |
| Provider | Plugin that translates HCL into cloud API calls |

---

## Quick Reference

```bash
# Lifecycle
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
terraform destroy

# State
terraform show
terraform state list
terraform state show <resource>
terraform state rm <resource>
terraform import <resource> <id>

# Flags
terraform plan -out=tfplan
terraform apply tfplan
terraform apply -auto-approve
terraform destroy -target=aws_instance.web
```

```hcl
# Minimal main.tf skeleton
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

resource "aws_s3_bucket" "example" {
  bucket = "my-unique-bucket-name-2026"
  tags   = { Name = "Example" }
}

resource "aws_instance" "example" {
  ami           = "ami-0f5ee92e2d63afc18"
  instance_type = "t2.micro"
  tags          = { Name = "Example" }
}
```

<img width="1577" height="742" alt="image" src="https://github.com/user-attachments/assets/ec2e091b-66a2-45d9-a253-d5fc01e3f407" />

<img width="1456" height="720" alt="image" src="https://github.com/user-attachments/assets/e09461c6-6fdb-4f12-bae3-63a76e724bd5" />

<img width="1406" height="712" alt="image" src="https://github.com/user-attachments/assets/5719af01-dc7e-46dc-9c38-598049e3171a" />
