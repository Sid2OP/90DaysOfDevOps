# Day 52 – Kubernetes Namespaces and Deployments

## Overview

Yesterday you created standalone Pods that disappear forever when deleted. Today you fix that with **Deployments** — the real way to run applications in Kubernetes. You also learn **Namespaces**, which let you organize and isolate resources inside a single cluster.

---

## Task 1: Explore Default Namespaces

```bash
# List all namespaces
kubectl get namespaces
```

### Built-in Namespaces

| Namespace | Purpose |
| --- | --- |
| `default` | Where your resources go if you don't specify a namespace |
| `kube-system` | Kubernetes internal components (API server, scheduler, etcd, coredns) |
| `kube-public` | Publicly readable resources; mostly used for cluster info |
| `kube-node-lease` | Node heartbeat objects for faster node failure detection |

```bash
# See the control plane running as pods
kubectl get pods -n kube-system
```

You'll see pods like `etcd-*`, `kube-apiserver-*`, `kube-scheduler-*`, `kube-controller-manager-*`, `coredns-*`, `kube-proxy-*`. These are the architecture components from Day 50 — running as pods inside the cluster itself.

> **Do not modify anything in `kube-system`.** Breaking these pods breaks your entire cluster.
> 

---
<img width="988" height="463" alt="image" src="https://github.com/user-attachments/assets/54dfcc65-062b-4c46-9e48-6125d0db5d3a" />


## Task 2: Create and Use Custom Namespaces

```bash
# Imperative creation
kubectl create namespace dev
kubectl create namespace staging

# Verify
kubectl get namespaces
```

```yaml
# production-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

```bash
# Declarative creation
kubectl apply -f production-namespace.yaml

# Run pods in specific namespaces
kubectl run nginx-dev --image=nginx:latest -n dev
kubectl run nginx-staging --image=nginx:latest -n staging

# This shows NOTHING (only default namespace)
kubectl get pods

# This shows everything across all namespaces
kubectl get pods -A
```

### Namespace Isolation Rules

- Resources in different namespaces are **logically isolated** — a Service in `dev` is not visible to pods in `staging` by default.
- Namespaces share the same cluster nodes and networking infrastructure.
- **Resource quotas** can be applied per namespace to limit CPU, memory, and object counts.
- `kubectl` commands always target the `default` namespace unless you specify `-n <namespace>` or `-A`.

---
<img width="937" height="121" alt="image" src="https://github.com/user-attachments/assets/249943a5-1a67-4100-9e15-649e8a0b6c67" />


## Task 4: Self-Healing — Delete a Pod and Watch It Come Back

```bash
# List current pods
kubectl get pods -n dev

# Delete one pod (use the actual name from your output)
kubectl delete pod nginx-deployment-xxxxx-yyyyy -n dev

# Watch immediately
kubectl get pods -n dev
```

**What happens:** The ReplicaSet controller detects only 2 of 3 desired replicas exist. Within seconds, it creates a new pod. The new pod gets a **different random name** — it's a fresh pod, not a restart of the old one.

```
Deleted:  nginx-deployment-7d6f5b8c4-abcde  (gone forever)
Created:  nginx-deployment-7d6f5b8c4-xyzwv  (new pod, same spec)
```

This is the fundamental value of Deployments: **self-healing without human intervention**.

---

## Task 5: Scale the Deployment

```bash
# Scale up imperatively
kubectl scale deployment nginx-deployment --replicas=5 -n dev
kubectl get pods -n dev
# 5 pods now running

# Scale down
kubectl scale deployment nginx-deployment --replicas=2 -n dev
kubectl get pods -n dev
# 3 pods are Terminating, 2 remain Running
```

When scaling down, Kubernetes **terminates the excess pods gracefully** — it sends SIGTERM, waits for the container to finish in-flight requests (up to `terminationGracePeriodSeconds`, default 30s), then force-kills.

### Declarative Scaling (Preferred)

Edit `nginx-deployment.yaml`, change `replicas: 4`, then:

```bash
kubectl apply -f nginx-deployment.yaml
```

This is the correct production approach — the YAML is your source of truth.

---

## Task 6: Rolling Update

```bash
# Update the image version (triggers a rolling update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 -n dev

# Watch the rollout in real time
kubectl rollout status deployment/nginx-deployment -n dev
# Waiting for deployment "nginx-deployment" rollout to finish...
# 1 out of 3 new replicas have been updated...
# 2 out of 3 new replicas have been updated...
# deployment "nginx-deployment" successfully rolled out
```

### How Rolling Update Works

```
Old RS (nginx:1.24):  3 pods  →  2 pods  →  1 pod  →  0 pods
New RS (nginx:1.25):  0 pods  →  1 pod   →  2 pods →  3 pods
```

New pods must pass **readiness checks** before old pods are terminated. This guarantees zero downtime — there are always healthy pods serving traffic.

```bash
# Check rollout history
kubectl rollout history deployment/nginx-deployment -n dev
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# Roll back to the previous version
kubectl rollout undo deployment/nginx-deployment -n dev

# Verify the rollback
kubectl rollout status deployment/nginx-deployment -n dev
kubectl describe deployment nginx-deployment -n dev | grep Image
# Image: nginx:1.24
```

### Rollout Commands Summary

```bash
kubectl rollout status deployment/<n> -n <ns>    # watch live progress
kubectl rollout history deployment/<n> -n <ns>   # show revision history
kubectl rollout undo deployment/<n> -n <ns>      # roll back one revision
kubectl rollout undo deployment/<n> --to-revision=1 -n <ns>  # roll back to specific revision
kubectl rollout pause deployment/<n> -n <ns>     # pause mid-rollout
kubectl rollout resume deployment/<n> -n <ns>    # resume paused rollout
```

---

## Task 7: Clean Up

```bash
kubectl delete deployment nginx-deployment -n dev
kubectl delete pod nginx-dev -n dev
kubectl delete pod nginx-staging -n staging

# Delete entire namespaces (removes EVERYTHING inside)
kubectl delete namespace dev staging production

# Verify
kubectl get namespaces
kubectl get pods -A
```

> **Production warning:** `kubectl delete namespace` is irreversible and removes every resource inside it. Always double-check which namespace you're targeting.
> 

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Namespace | Logical isolation inside a cluster; scope for names, quotas, and RBAC |
| `default` namespace | Where resources go when you don't specify `-n` |
| `kube-system` | Never modify — control plane lives here |
| Deployment | Manages a ReplicaSet which manages pods; provides self-healing, scaling, rolling updates |
| `selector.matchLabels` | Must exactly match `template.metadata.labels` — this is how the Deployment finds its pods |
| `READY 3/3` | current running / desired replicas |
| Self-healing | Delete a pod → ReplicaSet creates a new one immediately; new pod gets a new name |
| Scaling | `kubectl scale --replicas=N` or edit YAML and re-apply |
| Rolling update | New ReplicaSet scales up while old one scales down; zero downtime |
| `rollout undo` | Instantly rolls back to the previous ReplicaSet |
| Namespace delete | Removes all resources inside it — irreversible |

---

## Quick Reference

```bash
# Namespaces
kubectl create namespace <n>
kubectl get namespaces
kubectl delete namespace <n>
kubectl get pods -n <namespace>
kubectl get pods -A

# Deployments
kubectl apply -f deployment.yaml
kubectl get deployments -n <ns>
kubectl describe deployment <n> -n <ns>
kubectl delete deployment <n> -n <ns>

# Scaling
kubectl scale deployment <n> --replicas=5 -n <ns>

# Rolling updates
kubectl set image deployment/<n> <container>=<image>:<tag> -n <ns>
kubectl rollout status deployment/<n> -n <ns>
kubectl rollout history deployment/<n> -n <ns>
kubectl rollout undo deployment/<n> -n <ns>

# ReplicaSets (Deployment creates these automatically)
kubectl get replicasets -n <ns>

# Set default namespace for current context
kubectl config set-context --current --namespace=dev
```
