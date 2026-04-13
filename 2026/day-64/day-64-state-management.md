# Day 64 – Terraform State Management and Remote Backends

## Overview

The state file is the single most important thing in Terraform. It is the source of truth — the map between your `.tf` files and what actually exists in the cloud. Lose it and Terraform forgets everything. Corrupt it and your next apply could destroy production.

Today you learn to manage state like a professional: remote backends, locking, importing existing resources, and handling drift.

---

## Task 1: Inspect Your Current State

```bash
terraform show                                    # full state, human-readable
terraform state list                              # all resources tracked
terraform state show aws_instance.web             # every attribute of the instance
terraform state show aws_vpc.main                 # every attribute of the VPC
```

### What the State Stores for an EC2 Instance

You defined maybe 5 fields in your `.tf` file. The state stores everything AWS returned:

```
id, arn, public_ip, private_ip, public_dns, private_dns,
subnet_id, vpc_id, availability_zone, instance_state,
cpu_core_count, cpu_threads_per_core, ebs_block_device,
root_block_device, security_groups, ami, instance_type,
key_name, monitoring, tenancy, launch_time, tags_all...
```

This is why you never manually edit the state — Terraform uses all of these attributes internally for dependency resolution and change detection.

### The Serial Number

Every time Terraform successfully writes to the state file, it increments the `serial` counter. This is used for optimistic concurrency — if two processes try to write state simultaneously, the one with the lower serial number is rejected. It also tells you how many times the state has been modified.

---

## Task 2: Set Up S3 Remote Backend

Local state is dangerous: one deleted file and you lose everything, and teams can't share it. Move state to S3.

### Step 1 — Create the Backend Infrastructure

```bash
# Create S3 bucket for state
aws s3api create-bucket \
  --bucket terraweek-state-yourname \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Enable versioning (recover previous state files if corrupted)
aws s3api put-bucket-versioning \
  --bucket terraweek-state-yourname \
  --versioning-configuration Status=Enabled

# Block public access (state files are sensitive)
aws s3api put-public-access-block \
  --bucket terraweek-state-yourname \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraweek-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1
```

### Step 2 — Add Backend Block to Terraform Config

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "terraweek-state-yourname"
    key            = "dev/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraweek-state-lock"
    encrypt        = true
  }
}
```

### Step 3 — Migrate State

```bash
terraform init
# Terraform detects the new backend and prompts:
# "Do you want to copy existing state to the new backend?"
# Type: yes

# Verify migration
aws s3 ls s3://terraweek-state-yourname/dev/
# dev/terraform.tfstate  (should appear)

terraform plan
# Should show: No changes. (state migrated correctly)
```

### What `-migrate-state` Does

`terraform init -migrate-state` forces state migration even if Terraform doesn't automatically detect a backend change. Use this when switching between backends or when `terraform init` doesn't prompt automatically.

---
<img width="1577" height="642" alt="image" src="https://github.com/user-attachments/assets/0cf00beb-1b51-488f-98af-0e1affc9ee05" />

## Task 3: Test State Locking

State locking prevents two concurrent `terraform apply` runs from writing to the state simultaneously and corrupting it.

```bash
# Terminal 1
terraform apply
# (waiting for 'yes' confirmation...)

# Terminal 2 -- while Terminal 1 is waiting
terraform plan
# Error: Error acquiring the state lock
# Error message: ConditionalCheckFailedException
# Lock Info:
#   ID:        xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
#   Path:      terraweek-state-yourname/dev/terraform.tfstate
#   Operation: OperationTypePlan
#   Who:       user@hostname
#   Created:   2026-04-13 09:00:00
```

### Why Locking Is Critical for Teams

Without locking, two engineers running `terraform apply` simultaneously could interleave state writes, resulting in a corrupted state file that references resources that no longer exist or have wrong attributes. Recovery from a corrupted state file is painful and risky.

```bash
# If you get a stale lock (process died without releasing)
terraform force-unlock <LOCK_ID>
# Only use this when you are 100% sure no other operation is running
```

---
<img width="1572" height="446" alt="image" src="https://github.com/user-attachments/assets/a955468d-4f74-4472-9dad-4d76085ef43e" />

## Task 4: Import an Existing Resource

Not everything starts with Terraform. Import brings existing AWS resources under Terraform management.

```bash
# 1. Manually create an S3 bucket in AWS console
#    Name: terraweek-import-test-yourname

# 2. Write the resource block in your config
```

```hcl
# Add to main.tf
resource "aws_s3_bucket" "imported" {
  bucket = "terraweek-import-test-yourname"
}
```

```bash
# 3. Import it -- syntax: terraform import <resource_address> <real_id>
terraform import aws_s3_bucket.imported terraweek-import-test-yourname

# 4. Run plan to check alignment
terraform plan
# If "No changes" -- perfect match
# If changes shown -- your .tf doesn't match reality
# Update .tf to match, plan again until "No changes"

# 5. Verify it appears in state
terraform state list
# aws_s3_bucket.imported  <-- should appear
```

### `terraform import` vs Creating from Scratch

|  | `terraform import` | Creating from scratch |
| --- | --- | --- |
| Creates resource in AWS? | No — resource must already exist | Yes — Terraform creates it |
| Writes to state? | Yes — records the existing resource | Yes — after creation |
| Generates `.tf` config? | No — you write the config manually | Config is what you write |
| Use case | Adopt existing infra into Terraform | Green-field infrastructure |
| Risk | If config doesn't match, next plan shows diffs | No drift risk on first apply |

> **Terraform 1.5+ note:** `terraform import` blocks in config files allow importing without CLI commands. This is the modern approach.
> 

---
<img width="1380" height="555" alt="image" src="https://github.com/user-attachments/assets/504f7b89-9a5f-4863-baa4-eca3edb3de91" />

## Task 5: State Surgery — `mv` and `rm`

```bash
# Rename a resource (after refactoring .tf file)
terraform state list
# aws_s3_bucket.imported

terraform state mv aws_s3_bucket.imported aws_s3_bucket.logs_bucket
# Moved aws_s3_bucket.imported to aws_s3_bucket.logs_bucket

# Update .tf file: rename resource block to match
# resource "aws_s3_bucket" "logs_bucket" { ... }

terraform plan
# No changes -- state name matches .tf name
```

```bash
# Remove from state (without destroying in AWS)
terraform state rm aws_s3_bucket.logs_bucket
# Removed aws_s3_bucket.logs_bucket from state

terraform plan
# + aws_s3_bucket.logs_bucket  (Terraform wants to create it -- it forgot it exists!)

# Re-import to bring it back
terraform import aws_s3_bucket.logs_bucket terraweek-import-test-yourname
terraform plan
# No changes
```

### When to Use `state mv` and `state rm`

| Command | When to use |
| --- | --- |
| `state mv` | Refactoring `.tf` files — renaming a resource, moving it into a module, reorganizing without destroying infra |
| `state rm` | Handing off a resource to another Terraform workspace or team; intentionally removing from management without destroying; emergency: remove a broken resource from state so you can recreate it |

---
<img width="1485" height="395" alt="image" src="https://github.com/user-attachments/assets/631c51db-5bae-46c2-a631-734efcffd086" />

## Task 6: Simulate and Fix State Drift

State drift happens when someone changes infrastructure outside of Terraform — through the console, CLI, or another tool.

```bash
# 1. Apply so everything is in sync
terraform apply

# 2. Go to AWS console and manually:
#    - Change EC2 instance Name tag to "ManuallyChanged"
#    - Add a tag: Source = "ConsoleEdit"

# 3. Terraform detects the drift
terraform plan
# ~ aws_instance.web
#   ~ tags = {
#       ~ "Name"   = "ManuallyChanged" -> "terraweek-dev-server"
#       - "Source" = "ConsoleEdit"     -> null
#     }

# Option A: Reconcile -- force reality back to match config
terraform apply
# Tags restored to what .tf says

# Option B: Accept -- update .tf to match manual change
# Edit tags in main.tf, then:
terraform plan  # should show No changes
```

### Refresh-Only Mode

```bash
# Update state to match reality WITHOUT making any changes
terraform apply -refresh-only
# Updates state file to reflect the manual tag change
# Does NOT apply your .tf config
# Use this to see what drifted before deciding how to handle it
```

### How Teams Prevent State Drift in Production

| Strategy | Implementation |
| --- | --- |
| Restrict console access | IAM policies: engineers can read, only CI/CD can write |
| All changes via CI/CD | GitHub Actions / GitLab CI runs `terraform apply` — no manual applies |
| Drift detection | Scheduled CI job runs `terraform plan` daily and alerts if output != "No changes" |
| Require PR approval | Terraform plan output posted as PR comment; requires review before apply |
| Use Terraform Cloud | Built-in drift detection, run history, and RBAC |

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| State file | Source of truth; maps Terraform resources to real cloud resources |
| `serial` | Increments on every successful state write; prevents concurrent corruption |
| S3 backend | Remote state storage; survives local machine failure; shareable |
| DynamoDB locking | Prevents two concurrent applies from corrupting state |
| `encrypt = true` | State is encrypted at rest in S3 using SSE |
| `terraform import` | Brings existing resources into Terraform state without creating them |
| `state mv` | Renames a resource in state; use after refactoring `.tf` names |
| `state rm` | Removes from state without destroying; use when handing off resources |
| State drift | Reality diverged from state; detected by `terraform plan` showing unexpected diffs |
| `-refresh-only` | Updates state to match reality without applying config changes |
| `force-unlock` | Manually release a stale lock; only when no operation is running |

---

## Quick Reference

```bash
# State inspection
terraform show
terraform state list
terraform state show <resource>

# State migration
terraform init                    # prompts to migrate if backend changed
terraform init -migrate-state     # force migration

# State surgery
terraform state mv <old> <new>    # rename in state
terraform state rm <resource>     # remove from state (no destroy)
terraform import <resource> <id>  # import existing resource

# Drift
terraform plan                    # detect drift
terraform apply -refresh-only     # sync state to reality
terraform apply                   # force reality to match config

# Locking
terraform force-unlock <LOCK_ID>  # release stale lock
```

```hcl
# S3 remote backend
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "env/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```
