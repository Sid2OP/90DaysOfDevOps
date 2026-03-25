# Day 50 – Kubernetes Architecture and Cluster Setup

## Overview

Docker runs containers on a single machine. Kubernetes orchestrates containers across **many machines** — handling scheduling, self-healing, scaling, networking, and rolling updates automatically. Today you understand the architecture, set up a local cluster, and run your first `kubectl` commands.

---

## Task 1: The Kubernetes Story

### Why Was Kubernetes Created?

Docker solved *building and running* containers. But at scale — hundreds of services, thousands of containers, multiple servers — you immediately hit problems Docker alone cannot solve:

- How do you decide *which server* runs each container?
- What happens when a container crashes at 3 AM?
- How do you roll out a new version without downtime?
- How do containers on different servers find and talk to each other?
- How do you scale a service up when traffic spikes?

Kubernetes solves all of these. It is a **container orchestrator** — a system that manages the full lifecycle of containerized workloads across a cluster of machines.

### Who Created It?

Google created Kubernetes, open-sourcing it in 2014. It was directly inspired by **Borg**, Google's internal cluster management system that had been running their production workloads (Search, Gmail, YouTube) at massive scale for over a decade. The engineers who built Borg took those lessons and designed Kubernetes for the open-source world.

Kubernetes was donated to the **Cloud Native Computing Foundation (CNCF)** in 2016 and has since become the industry standard for container orchestration.

### What Does the Name Mean?

"Kubernetes" comes from the Greek word **κυβερνήτης** meaning *helmsman* or *pilot* — the person who steers a ship. The logo is a ship's wheel. The abbreviation **k8s** comes from replacing the 8 middle letters of "kubernetes" with the number 8.

---

## Task 2: Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                        │
│                                                         │
│  ┌─────────────┐  ┌──────┐  ┌───────────┐  ┌────────┐  │
│  │ API Server  │  │ etcd │  │ Scheduler │  │  Ctrl  │  │
│  │ (front door)│  │ (DB) │  │(where pod │  │ Manager│  │
│  │ all kubectl │  │all   │  │ goes?)    │  │(desired│  │
│  │ commands hit│  │state │  │           │  │ state) │  │
│  └──────┬──────┘  └──────┘  └───────────┘  └────────┘  │
└─────────┼───────────────────────────────────────────────┘
          │  HTTPS
    ┌─────┼──────────────────────────────────┐
    │     │         WORKER NODE              │
    │  ┌──▼──────┐  ┌────────────┐           │
    │  │ kubelet │  │ kube-proxy │           │
    │  │ (agent) │  │(networking)│           │
    │  └──┬──────┘  └────────────┘           │
    │     │                                  │
    │  ┌──▼──────────────┐                   │
    │  │ Container Runtime│  (containerd)    │
    │  └─────────────────┘                   │
    │     │                                  │
    │  ┌──▼──┐ ┌─────┐ ┌─────┐              │
    │  │ Pod │ │ Pod │ │ Pod │              │
    │  └─────┘ └─────┘ └─────┘              │
    └─────────────────────────────────────────┘
```

### Control Plane Components

| Component | Role |
| --- | --- |
| **API Server** | The single entry point for all cluster operations. Every `kubectl` command, every controller, every node — all talk to the API server. It validates requests and writes state to etcd. |
| **etcd** | A distributed key-value store. The source of truth for all cluster state — what pods exist, what nodes are registered, what services are defined. If etcd goes down, the cluster still *runs* but nothing can be changed. |
| **Scheduler** | Watches for newly created pods with no assigned node. Evaluates resource requests, node capacity, affinity rules, and taints to pick the best node. |
| **Controller Manager** | Runs control loops (controllers) that watch cluster state and take action to reconcile desired vs actual state. Examples: ReplicaSet controller (ensures N replicas running), Node controller (detects node failures). |

### Worker Node Components

| Component | Role |
| --- | --- |
| **kubelet** | An agent running on every node. Receives pod specs from the API server and ensures the containers described in those specs are running and healthy. Reports node/pod status back up. |
| **kube-proxy** | Maintains network rules on each node so pods can reach Services. Handles load-balancing across pod endpoints using iptables or IPVS. |
| **Container Runtime** | The engine that actually pulls images and runs containers. Kubernetes uses the CRI (Container Runtime Interface) — most clusters use **containerd** or CRI-O. Docker is no longer used directly. |

### Tracing `kubectl apply -f pod.yaml`

```
1. kubectl reads pod.yaml and sends POST request to API Server
2. API Server authenticates & validates the request
3. API Server writes the pod object to etcd (status: Pending)
4. Scheduler notices a new pod with no node assignment
5. Scheduler scores all nodes, picks the best one
6. Scheduler updates the pod object in etcd with nodeName: <node>
7. kubelet on that node sees the pod is assigned to it
8. kubelet calls the Container Runtime (containerd) to pull the image
9. Container Runtime starts the container
10. kubelet updates pod status to Running in etcd via API Server
11. kubectl get pod now shows Running
```

### Failure Scenarios

**API Server goes down:** The cluster keeps running — existing pods continue. But nothing can be changed: no new deployments, no scaling, no `kubectl` commands work until it recovers.

**Worker node goes down:** The Node controller detects it after ~40 seconds. Pods on that node are marked as `Unknown`/`Terminating`. If those pods belong to a Deployment or ReplicaSet, the controller reschedules them on healthy nodes.

---

## Task 3: Install kubectl

```bash
# macOS
brew install kubectl

# Linux (amd64)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Windows (Chocolatey)
choco install kubernetes-cli

# Verify
kubectl version --client
```

---

## Task 4: Set Up Your Local Cluster

### Option A: kind (Kubernetes in Docker)

kind runs a full Kubernetes cluster using Docker containers as nodes — no VM required, fast to spin up, great for CI.

```bash
# Install kind
# macOS
brew install kind

# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create a cluster
kind create cluster --name devops-cluster

# Verify
kubectl cluster-info
kubectl get nodes
```

### Option B: minikube

minikube creates a single-node cluster in a VM or container. Has more features out of the box (dashboard, addons, LoadBalancer support).

```bash
# Install minikube
# macOS
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start

# Verify
kubectl cluster-info
kubectl get nodes
```

### Which to Choose and Why

|  | kind | minikube |
| --- | --- | --- |
| **Speed** | Faster startup | Slightly slower |
| **Driver** | Docker only | Docker, VirtualBox, Hyperkit |
| **Multi-node** | Yes (config file) | Limited |
| **Addons** | Manual | Built-in (dashboard, ingress) |
| **Best for** | CI, scripting, multi-node testing | Learning, local development |

**Recommended for this journey:** `kind` — it's lightweight, Docker-native, and mirrors how clusters are tested in CI pipelines.

---

## Task 5: Explore Your Cluster

```bash
# Cluster info
kubectl cluster-info

# List all nodes
kubectl get nodes
kubectl get nodes -o wide    # extra detail: IP, OS, container runtime

# Describe a node (events, capacity, allocated resources)
kubectl describe node <node-name>

# List all namespaces
kubectl get namespaces

# All pods across all namespaces
kubectl get pods -A

# Pods in kube-system only
kubectl get pods -n kube-system
```

### Matching kube-system Pods to Architecture Components

| Pod name prefix | Architecture component |
| --- | --- |
| `etcd-*` | etcd — cluster database |
| `kube-apiserver-*` | API Server |
| `kube-scheduler-*` | Scheduler |
| `kube-controller-manager-*` | Controller Manager |
| `kube-proxy-*` | kube-proxy (one per node) |
| `coredns-*` | CoreDNS — internal DNS for service discovery |
| `kindnet-*` / `calico-*` | CNI plugin — pod networking |

The control plane components (etcd, apiserver, scheduler, controller-manager) run as **static pods** managed directly by kubelet on the control plane node — not by the API server itself. This is why they can bootstrap the cluster.

---

## Task 6: Cluster Lifecycle & kubeconfig

```bash
# Delete cluster
kind delete cluster --name devops-cluster
# or
minikube delete

# Recreate
kind create cluster --name devops-cluster
# or
minikube start

# Verify
kubectl get nodes

# Context management
kubectl config current-context      # which cluster am I talking to?
kubectl config get-contexts         # list all clusters/contexts
kubectl config use-context <name>   # switch to a different cluster
kubectl config view                 # show full kubeconfig
```

### What is a kubeconfig?

A kubeconfig is a YAML file that tells `kubectl` **how to connect to a cluster**. It contains:

- **clusters** — API server addresses and CA certificates
- **users** — credentials (client certificates, tokens, or exec plugins)
- **contexts** — a named pairing of cluster + user + namespace
- **current-context** — which context is active

**Default location:** `~/.kube/config`

When you run `kind create cluster` or `minikube start`, the tool automatically writes (or merges) a new context into `~/.kube/config` and sets it as the current context. You can override the location with the `KUBECONFIG` environment variable or `--kubeconfig` flag.

```bash
# Example: use a different kubeconfig file
export KUBECONFIG=/path/to/my-cluster.yaml
kubectl get nodes

# Merge multiple kubeconfig files
export KUBECONFIG=~/.kube/config:~/.kube/work-cluster.yaml
kubectl config get-contexts
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Kubernetes purpose | Orchestrates containers at scale: scheduling, self-healing, scaling, networking |
| Origin | Google, 2014. Inspired by internal Borg system. Now CNCF. |
| Name meaning | Greek for "helmsman" / pilot. k8s = 8 middle letters. |
| API Server | Single entry point; all kubectl commands and controllers go through it |
| etcd | Cluster database; source of truth for all state |
| Scheduler | Picks which node a new pod runs on |
| Controller Manager | Reconciles desired vs actual state in control loops |
| kubelet | Agent on each node; makes sure containers match pod specs |
| kube-proxy | Network rules for Service routing and load balancing |
| Container runtime | Runs containers (containerd, CRI-O — not Docker directly) |
| kind | Kubernetes in Docker — fast, lightweight, great for local dev and CI |
| minikube | Feature-rich local cluster with built-in addons |
| kubeconfig | `~/.kube/config` — stores cluster connection info and credentials |
| context | A named cluster+user+namespace combo in kubeconfig |

---

## Quick Reference

```bash
# Cluster setup
kind create cluster --name devops-cluster
kind delete cluster --name devops-cluster
kind get clusters

# Basic exploration
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl get namespaces
kubectl get pods -A
kubectl get pods -n kube-system

# Context management
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <name>
kubectl config view

# Flags
-o wide          # extra columns
-o yaml          # full YAML output
-n <namespace>   # specify namespace
-A               # all namespaces
```

<img width="1795" height="558" alt="image" src="https://github.com/user-attachments/assets/8d0f65cd-9d4c-4187-992f-9f3eea39c178" />

<img width="1846" height="770" alt="image" src="https://github.com/user-attachments/assets/64e7f39a-8c9d-42c5-aea3-899edaaa6902" />

<img width="1312" height="532" alt="image" src="https://github.com/user-attachments/assets/ee43bea2-245f-44ec-b153-f2edb4704bbe" />





