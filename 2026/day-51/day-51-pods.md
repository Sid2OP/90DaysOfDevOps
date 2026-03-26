# Day 51 – Kubernetes Manifests and Your First Pods

## Overview

Yesterday you set up a cluster. Today you **actually deploy something**. You'll learn the structure of a Kubernetes manifest, create Pods hands-on, and understand the declarative vs imperative approaches. By the end you should be able to write a Pod manifest from scratch without looking at docs.

---

## The Anatomy of a Kubernetes Manifest

Every Kubernetes resource is a YAML file with four required top-level fields:

```yaml
apiVersion: v1        # Which API group/version to use
kind: Pod             # What type of resource
metadata:             # Identity: name, labels, namespace
  name: my-pod
  labels:
    app: my-app
spec:                 # Desired state — what you want Kubernetes to run
  containers:
  - name: my-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

| Field | Purpose |
| --- | --- |
| `apiVersion` | Which API group handles this resource. `v1` for core resources (Pod, Service, ConfigMap). `apps/v1` for Deployments. |
| `kind` | Resource type. Today: `Pod`. Later: `Deployment`, `Service`, `ConfigMap`, etc. |
| `metadata` | Identity of the resource. `name` is required. `labels` are key-value pairs for organization and selection. |
| `spec` | The desired state. For a Pod: which containers, which images, which ports, resource limits, volumes, etc. |

---

## Task 1: Create Your First Pod (Nginx)

```yaml
# nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
# Apply the manifest
kubectl apply -f nginx-pod.yaml

# Check status
kubectl get pods
kubectl get pods -o wide          # shows node IP and which node it landed on

# Detailed event log (great for debugging)
kubectl describe pod nginx-pod

# Read container logs
kubectl logs nginx-pod

# Shell into the container
kubectl exec -it nginx-pod -- /bin/bash

# Inside the container:
curl localhost:80
exit
```

**Verify:** `curl localhost:80` from inside the container returns the Nginx welcome HTML page.

**What `kubectl describe` shows:**

- `Node:` — which node the pod was scheduled to
- `IP:` — the pod's cluster IP
- `Containers:` — image, ports, state, restart count
- `Events:` — the scheduling + pull + start timeline, and any errors

---
<img width="1322" height="752" alt="image" src="https://github.com/user-attachments/assets/589336e0-19a8-458d-934a-ec5b328fa19b" />
<img width="1582" height="775" alt="image" src="https://github.com/user-attachments/assets/ebe126db-235c-4615-98ac-9861ee32bf49" />

## Task 2: Create a Custom Pod (BusyBox)

```yaml
# busybox-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
  labels:
    app: busybox
    environment: dev
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "echo Hello from BusyBox && sleep 3600"]
```

```bash
kubectl apply -f busybox-pod.yaml
kubectl get pods
kubectl logs busybox-pod
# Output: Hello from BusyBox
```

### Why the `command` Field Is Required Here

BusyBox has no default long-lived process. Without `command`, the container starts, immediately exits with code 0, and Kubernetes restarts it — over and over. This produces `CrashLoopBackOff`. The `sleep 3600` keeps the container alive for 1 hour so you can interact with it.

```bash
# Shell into BusyBox (no bash — use sh)
kubectl exec -it busybox-pod -- /bin/sh
whoami
hostname
ls /
exit
```

---
<img width="1102" height="572" alt="image" src="https://github.com/user-attachments/assets/9f5591c0-4eaf-46d5-8326-0af856251393" />

## Task 3: Imperative vs Declarative

### Declarative (YAML + `kubectl apply`) — Recommended

```bash
# Create or update from a file
kubectl apply -f nginx-pod.yaml

# Update: edit the file, re-apply — Kubernetes figures out what changed
kubectl apply -f nginx-pod.yaml
```

The file IS the source of truth. Check it into git. Repeat apply = idempotent.

### Imperative (CLI commands) — Quick + Dirty

```bash
# Create a pod without any YAML
kubectl run redis-pod --image=redis:latest

kubectl get pods
```

### The Dry-Run Scaffolding Trick

```bash
# Generate YAML without creating anything
kubectl run test-pod --image=nginx --dry-run=client -o yaml

# Save it as a starting point
kubectl run test-pod --image=nginx --dry-run=client -o yaml > test-pod.yaml
```

This is the fastest way to scaffold a manifest. Generate, then customize.

### What Kubernetes Adds Automatically

When you do `kubectl get pod redis-pod -o yaml`, you see fields you never wrote:

```yaml
metadata:
  uid: 3f2a1b4c-...           # unique ID assigned by Kubernetes
  resourceVersion: "12345"   # version counter for optimistic locking
  creationTimestamp: "..."   # when it was created
spec:
  nodeName: devops-cluster   # which node the scheduler chose
  tolerations: [...]          # default tolerations added automatically
status:
  phase: Running
  podIP: 10.244.0.5
  containerStatuses:
  - ready: true
    restartCount: 0
```

Your YAML defines `spec`. Kubernetes fills in `status` and most of `metadata`.

---

## Task 4: Validate Before Applying

```bash
# Client-side validation (checks YAML structure only, no cluster needed)
kubectl apply -f nginx-pod.yaml --dry-run=client

# Server-side validation (cluster checks against its API schema)
kubectl apply -f nginx-pod.yaml --dry-run=server
```

### Intentional Break — Missing `image` Field

```yaml
# broken-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
spec:
  containers:
  - name: nginx
    # image field removed!
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f broken-pod.yaml --dry-run=client
# Error: spec.containers[0].image: Required value
```

Kubernetes tells you exactly which field is missing and that it is required. This is far better than applying and watching it fail at runtime.

### Other Validation Errors to Know

| Mistake | Error message |
| --- | --- |
| Missing `image` | `spec.containers[0].image: Required value` |
| Invalid `apiVersion` | `no matches for kind "Pod" in version "v999"` |
| Wrong indentation | YAML parse error before it even hits Kubernetes |
| Unknown field | `unknown field "spec.containers[0].imagen"` (server-side) |

---

## Task 5: Pod Labels and Filtering

```yaml
# multi-label-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-label-pod
  labels:
    app: payments-service
    environment: staging
    team: backend
    version: v2
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sh", "-c", "sleep 3600"]
```

```bash
# Show all pods with their labels
kubectl get pods --show-labels

# Filter by single label
kubectl get pods -l app=nginx
kubectl get pods -l environment=dev

# Filter by multiple labels (AND logic)
kubectl get pods -l app=payments-service,environment=staging

# Filter with set-based selector
kubectl get pods -l 'environment in (dev, staging)'
kubectl get pods -l 'environment notin (production)'

# Add a label to an existing pod
kubectl label pod nginx-pod environment=production

# Overwrite an existing label (requires --overwrite)
kubectl label pod nginx-pod environment=production --overwrite

# Remove a label (trailing hyphen)
kubectl label pod nginx-pod environment-

# Verify
kubectl get pods --show-labels
```

### Why Labels Matter

Labels have no inherent meaning to Kubernetes — they are arbitrary key-value pairs YOU define. Their power comes from **selectors**:

- **Services** use label selectors to find which pods to route traffic to
- **Deployments** use label selectors to manage their pods
- **kubectl** uses `-l` to filter what you see
- **NetworkPolicies** use label selectors to control traffic

A pod with `app: nginx` is just a pod until a Service selects `app: nginx` — then it suddenly gets traffic.

---

## Task 6: Clean Up

```bash
# Delete by name
kubectl delete pod nginx-pod
kubectl delete pod busybox-pod
kubectl delete pod redis-pod
kubectl delete pod multi-label-pod

# Delete using the manifest file
kubectl delete -f nginx-pod.yaml

# Delete all pods in the default namespace
kubectl delete pods --all

# Verify
kubectl get pods
# No resources found in default namespace.
```

### The Critical Lesson

When you delete a standalone Pod, **it is gone permanently**. There is no controller watching it. No automatic recreation. This is exactly why production workloads never use bare Pods — they use **Deployments**, which wrap Pods in a controller that ensures the desired number always runs. (Coming on Day 52.)

```
Bare Pod deleted  →  Gone forever
Deployment pod deleted  →  Controller recreates it immediately
```

---

## Declarative vs Imperative Summary

|  | Declarative (`apply -f`) | Imperative (`run`, `create`) |
| --- | --- | --- |
| **Source of truth** | YAML file in git | Your memory / shell history |
| **Idempotent** | Yes — apply again = no change | No — run again = error (already exists) |
| **Best for** | Production, team environments | Quick testing, scaffolding |
| **Audit trail** | Git history | None |
| **Kubernetes prefers** | ✅ Yes | ❌ Discouraged in production |

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Four manifest fields | `apiVersion`, `kind`, `metadata`, `spec` — all required |
| `kubectl apply` | Declarative; creates or updates; idempotent |
| `kubectl run` | Imperative; creates pod directly without YAML |
| `--dry-run=client -o yaml` | Scaffolds a manifest template without creating anything |
| `kubectl describe pod` | Shows events, node, IP, container state — primary debug tool |
| `kubectl logs` | Shows container stdout/stderr |
| `kubectl exec -it` | Opens an interactive shell inside a running container |
| `CrashLoopBackOff` | Container keeps crashing and restarting — check logs first |
| Labels | Arbitrary key-value pairs; power comes from selectors |
| Bare pod deletion | Permanent — no controller recreates it |

---

## Quick Reference

```bash
# Apply / Delete
kubectl apply -f pod.yaml
kubectl delete -f pod.yaml
kubectl delete pod <name>

# Inspect
kubectl get pods
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl describe pod <name>
kubectl logs <name>
kubectl logs <name> -f          # follow/stream logs
kubectl logs <name> --previous  # logs from previous crash

# Exec
kubectl exec -it <name> -- /bin/bash
kubectl exec -it <name> -- /bin/sh   # if bash not available
kubectl exec <name> -- env           # non-interactive

# Labels
kubectl get pods -l key=value
kubectl label pod <name> key=value
kubectl label pod <name> key=value --overwrite
kubectl label pod <name> key-         # remove label

# Dry-run + scaffold
kubectl apply -f pod.yaml --dry-run=client
kubectl run mypod --image=nginx --dry-run=client -o yaml > mypod.yaml
```
