# Day 66 – Provision an EKS Cluster with Terraform Modules

## Overview

You built Kubernetes clusters manually during the Kubernetes week. Today you provision one the DevOps way — fully automated, repeatable, and destroyable with a single command using Terraform registry modules.

> **Cost warning:** NAT Gateway charges ~$0.045/hour. EKS control plane charges ~$0.10/hour. Destroy everything when done.
> 

---

## Project Structure

```
terraform-eks/
├── providers.tf        # AWS + Kubernetes provider config
├── vpc.tf              # VPC module call
├── eks.tf              # EKS module call
├── variables.tf        # All input variables
├── outputs.tf          # Cluster outputs
├── terraform.tfvars    # Variable values
└── k8s/
    └── nginx-deployment.yaml
```

---

## Task 1: Project Setup

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

```hcl
# variables.tf
variable "region"             { type = string; default = "ap-south-1" }
variable "cluster_name"       { type = string; default = "terraweek-eks" }
variable "cluster_version"    { type = string; default = "1.31" }
variable "node_instance_type" { type = string; default = "t3.medium" }
variable "node_desired_count" { type = number; default = 2 }
variable "vpc_cidr"           { type = string; default = "10.0.0.0/16" }
```

```hcl
# terraform.tfvars
region             = "ap-south-1"
cluster_name       = "terraweek-eks"
node_instance_type = "t3.medium"
node_desired_count = 2
```

---

## Task 2: Create the VPC

```hcl
# vpc.tf
data "aws_availability_zones" "available" {
  state = "available"
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"
  cidr = var.vpc_cidr

  azs             = slice(data.aws_availability_zones.available.names, 0, 2)
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true      # one NAT saves cost in dev
  enable_dns_hostnames = true

  # Required EKS subnet tags
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }

  tags = {
    Project   = "TerraWeek"
    ManagedBy = "Terraform"
  }
}
```

```bash
terraform init
terraform plan   # verify VPC config before adding EKS
```

### Why EKS Needs Public AND Private Subnets

| Subnet type | What runs here |
| --- | --- |
| **Private** | Worker nodes (EC2 instances) — no direct internet exposure |
| **Public** | Load balancers (ELBs created by `type: LoadBalancer` Services) |

Nodes in private subnets reach the internet via the **NAT Gateway** in the public subnet — for pulling container images, calling AWS APIs, etc. The subnet tags tell EKS which subnets to use for internal vs external load balancers.

---

## Task 3: Create the EKS Cluster

```hcl
# eks.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets  # nodes go in private subnets

  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    terraweek_nodes = {
      ami_type       = "AL2_x86_64"
      instance_types = [var.node_instance_type]

      min_size     = 1
      max_size     = 3
      desired_size = var.node_desired_count
    }
  }

  tags = {
    Environment = "dev"
    Project     = "TerraWeek"
    ManagedBy   = "Terraform"
  }
}
```

```bash
terraform init      # downloads EKS module + dependencies
terraform plan      # review ~30+ resources before applying
```

Review the plan carefully. You will see: EKS cluster, IAM roles for nodes and cluster, managed node group, security groups, OIDC provider, launch template, and more — all created automatically by the module.

---

## Task 4: Apply and Connect kubectl

```bash
# Apply -- takes 10-15 minutes
terraform apply

# Update kubeconfig
aws eks update-kubeconfig --name terraweek-eks --region ap-south-1

# Verify
kubectl get nodes
# NAME                                         STATUS   ROLES    AGE
# ip-10-0-3-x.ap-south-1.compute.internal     Ready    <none>   2m
# ip-10-0-4-x.ap-south-1.compute.internal     Ready    <none>   2m

kubectl get pods -A
kubectl cluster-info
```

```hcl
# outputs.tf
output "cluster_name" {
  value = module.eks.cluster_name
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "cluster_region" {
  value = var.region
}

output "configure_kubectl" {
  value       = "aws eks update-kubeconfig --name ${module.eks.cluster_name} --region ${var.region}"
  description = "Command to configure kubectl"
}
```

---

## Task 5: Deploy a Workload

```yaml
# k8s/nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-terraweek
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f k8s/nginx-deployment.yaml

# Wait for LoadBalancer external IP/hostname
kubectl get svc nginx-service -w
# NAME            TYPE           CLUSTER-IP     EXTERNAL-IP
# nginx-service   LoadBalancer   172.20.x.x     <pending>
# nginx-service   LoadBalancer   172.20.x.x     a1b2c3d4.ap-south-1.elb.amazonaws.com

# Full picture
kubectl get nodes
kubectl get deployments
kubectl get pods
kubectl get svc

# Access the app
curl http://a1b2c3d4.ap-south-1.elb.amazonaws.com
```

---

## Task 6: Destroy Everything (Critical)

EKS + NAT Gateway costs money. Always clean up.

```bash
# Step 1: Delete Kubernetes resources FIRST
# (deletes the AWS LoadBalancer before terraform destroy)
kubectl delete -f k8s/nginx-deployment.yaml

# Wait for LB to be fully removed -- check EC2 > Load Balancers in console
# Confirm it's gone before proceeding

# Step 2: Destroy all Terraform resources (10-15 minutes)
terraform destroy

# Step 3: Verify in AWS console
# EKS: no clusters
# EC2: no node group instances
# VPC: terraweek-eks-vpc gone
# NAT Gateways: deleted
# Elastic IPs: released
```

### Why Delete K8s Resources Before `terraform destroy`?

When you create a `type: LoadBalancer` Service, Kubernetes asks AWS to create an ELB. This ELB is **not** managed by Terraform — it was created by the cloud controller manager. If you run `terraform destroy` first, it tries to delete the VPC, but the ELB is still attached to the VPC's subnets. AWS refuses to delete the VPC, and `terraform destroy` gets stuck. Always delete LoadBalancer Services first.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `Unauthorized` with kubectl | Re-run `aws eks update-kubeconfig --name terraweek-eks --region ap-south-1` |
| `terraform destroy` stuck | Check for leftover ELBs, ENIs, or security groups in the VPC |
| Nodes not Ready | `kubectl get events --sort-by=.metadata.creationTimestamp` |
| Pods pending | `kubectl describe pod <name>` — check for insufficient resources |
| LoadBalancer stays `<pending>` | Wait 2-3 minutes; ELB provisioning is slow |

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| EKS managed node group | AWS manages the EC2 instances, scaling, and AMI updates |
| Private subnets for nodes | Nodes have no public IP; reach internet via NAT Gateway |
| Public subnets for LBs | ELBs need public subnets; tagged with `kubernetes.io/role/elb` |
| `cluster_endpoint_public_access` | Allows `kubectl` from your machine; restrict in production |
| EKS module IAM roles | Module auto-creates all required IAM roles and policies |
| Delete LBs before destroy | K8s-created ELBs block VPC deletion if not removed first |
| `aws eks update-kubeconfig` | Updates `~/.kube/config` with cluster credentials |

---

## Quick Reference

```bash
# Provision
terraform init
terraform plan
terraform apply

# Connect kubectl
aws eks update-kubeconfig --name terraweek-eks --region ap-south-1
kubectl get nodes

# Deploy workload
kubectl apply -f k8s/nginx-deployment.yaml
kubectl get svc nginx-service -w

# Clean up (ALWAYS in this order)
kubectl delete -f k8s/nginx-deployment.yaml
# wait for ELB to disappear in AWS console
terraform destroy
```
