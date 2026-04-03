# Day 58 – Metrics Server and Horizontal Pod Autoscaler (HPA)

## Overview

Yesterday you set resource requests and limits. Today you put them to work. Install the **Metrics Server** so Kubernetes can see actual resource usage, then configure a **Horizontal Pod Autoscaler (HPA)** that scales pods up under load and back down automatically.

---

## How HPA Works

```
Metrics Server polls kubelets every 15s
        ↓
HPA controller checks metrics every 15s
        ↓
desiredReplicas = ceil(currentReplicas * (currentUsage / targetUsage))
        ↓
HPA updates Deployment replicas field
        ↓
Deployment controller creates/deletes pods
```

HPA **requires** `resources.requests` on your containers. Without requests, utilization percentages cannot be calculated and TARGETS shows `<unknown>`.

---

## Task 1: Install the Metrics Server

```bash
# Check if already running
kubectl get pods -n kube-system | grep metrics-server

# Minikube
minikube addons enable metrics-server

# Kind or kubeadm — apply official manifest with insecure TLS flag for local clusters
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For Kind (needs --kubelet-insecure-tls flag)
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Wait 60 seconds, then verify
kubectl top nodes
kubectl top pods -A
```

> `--kubelet-insecure-tls` skips TLS verification between the metrics server and kubelets. **Never use this in production** — local clusters use self-signed certs that the metrics server rejects by default.
> 

---

## Task 2: Explore kubectl top

```bash
# Node resource usage
kubectl top nodes
# NAME                 CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# devops-cluster       112m         2%     512Mi           13%

# All pods across all namespaces
kubectl top pods -A

# Sort by CPU
kubectl top pods -A --sort-by=cpu

# Sort by memory
kubectl top pods -A --sort-by=memory

# Specific namespace
kubectl top pods -n kube-system
```

### `kubectl top` vs `kubectl describe`

| Command | Shows |
| --- | --- |
| `kubectl top pod` | **Actual live usage** (from Metrics Server, updated every 15s) |
| `kubectl describe pod` | **Configured requests/limits** (what you declared in the YAML) |

These are completely different numbers. A pod can have `requests.cpu: 500m` but actually be using only `12m` at rest.

---
<img width="1461" height="318" alt="image" src="https://github.com/user-attachments/assets/9f98bb30-fad9-47cb-8480-f5730e98de5a" />
<img width="1172" height="498" alt="image" src="https://github.com/user-attachments/assets/0572b97b-14d0-452a-bfcd-224808c27080" />
<img width="1013" height="706" alt="image" src="https://github.com/user-attachments/assets/92f40f3f-aa54-4018-b513-bfd6a4a2e54d" />

## Task 3: Create a Deployment with CPU Requests

```yaml
# php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m    # HPA needs this to calculate utilization %
          limits:
            cpu: 500m
```

```bash
kubectl apply -f php-apache.yaml

# Expose as a Service
kubectl expose deployment php-apache --port=80 --name=php-apache

# Check current CPU usage
kubectl top pod -l app=php-apache
# php-apache-xxxxx   2m    (idle, almost nothing)
```

---
<img width="1118" height="242" alt="image" src="https://github.com/user-attachments/assets/226332ee-3dbf-4632-b681-bec567f91f42" />

## Task 4: Create an HPA (Imperative)

```bash
# Create HPA: scale between 1 and 10 replicas, target 50% CPU utilization
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

# Check it
kubectl get hpa
# NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS
# php-apache   Deployment/php-apache   <unknown>/50%   1         10        1

# Wait 30-60 seconds for metrics to arrive
kubectl get hpa
# NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS
# php-apache   Deployment/php-apache   2%/50%    1         10        1

# Detailed view
kubectl describe hpa php-apache
```

<img width="938" height="61" alt="image" src="https://github.com/user-attachments/assets/7e5b42ba-34c9-44d0-beb3-9fe30b711b7b" />

### TARGETS: `2%/50%` means:

- `2%` = current average CPU utilization across all pods
- `50%` = target threshold — scale up if current exceeds this

---

## Task 5: Generate Load and Watch Autoscaling

```bash
# Start load generator in a new terminal
kubectl run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

# Watch HPA in another terminal
kubectl get hpa php-apache --watch
# TARGETS      REPLICAS
# 2%/50%       1
# 68%/50%      1
# 250%/50%     4       <-- scaling up!
# 185%/50%     5
# 98%/50%      5
# 52%/50%      5
# 48%/50%      5       <-- stabilized

# Watch pods being created
kubectl get pods -l app=php-apache --watch

# Stop the load
kubectl delete pod load-generator

# Scale-down has a 5-minute stabilization window
# HPA waits to confirm load is truly gone before scaling down
kubectl get hpa php-apache --watch
# Eventually: REPLICAS drops back to 1
```

### Scale-Up vs Scale-Down Speed

| Direction | Default behavior |
| --- | --- |
| Scale-up | Fast — responds within 1-2 metric cycles (~30s) |
| Scale-down | Slow — 5-minute stabilization window to prevent flapping |

---
<img width="1110" height="690" alt="image" src="https://github.com/user-attachments/assets/8c804ce9-e207-4956-842b-3d43dd4be55b" />

## Task 6: HPA from YAML (Declarative — autoscaling/v2)

```bash
# Delete the imperative HPA first
kubectl delete hpa php-apache
```

```yaml
# hpa-v2.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0       # scale up immediately
      policies:
      - type: Pods
        value: 4                          # add up to 4 pods per 15s
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300     # wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10                         # remove at most 10% of pods per 60s
        periodSeconds: 60
```

```bash
kubectl apply -f hpa-v2.yaml
kubectl describe hpa php-apache
```

### What the `behavior` Section Controls

| Field | Effect |
| --- | --- |
| `scaleUp.stabilizationWindowSeconds: 0` | Scale up immediately with no delay |
| `scaleUp.policies[].value: 4` | Add at most 4 pods per 15-second window |
| `scaleDown.stabilizationWindowSeconds: 300` | Wait 5 minutes of sustained low usage before scaling down |
| `scaleDown.policies[].value: 10` | Remove at most 10% of current pods per 60 seconds |

### autoscaling/v1 vs autoscaling/v2

|  | `autoscaling/v1` | `autoscaling/v2` |
| --- | --- | --- |
| CPU metrics | Yes | Yes |
| Memory metrics | No | Yes |
| Custom metrics | No | Yes (Prometheus, Datadog, etc.) |
| `behavior` tuning | No | Yes |
| Recommended | No (deprecated) | Yes |

---
<img width="1037" height="635" alt="image" src="https://github.com/user-attachments/assets/65e65b1c-a2b8-43ac-9f97-94f53454f448" />

## Task 7: Clean Up

```bash
kubectl delete hpa php-apache
kubectl delete service php-apache
kubectl delete deployment php-apache
kubectl delete pod load-generator --ignore-not-found

# Leave metrics-server running
kubectl get pods -n kube-system | grep metrics-server

kubectl get pods
kubectl get services
kubectl get hpa
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Metrics Server | Collects real-time CPU/memory from kubelets; required for `kubectl top` and HPA |
| `kubectl top` | Shows actual live usage; completely separate from requests/limits in YAML |
| HPA requirement | `resources.requests` must be set; without it, TARGETS shows `<unknown>` |
| HPA formula | `desiredReplicas = ceil(current * currentUsage / targetUsage)` |
| Scale-up | Fast; responds within ~30 seconds |
| Scale-down | Slow; 5-minute default stabilization window prevents flapping |
| `autoscaling/v2` | Supports CPU, memory, custom metrics, and fine-grained `behavior` tuning |
| `behavior.scaleUp` | Controls speed and burst size when scaling out |
| `behavior.scaleDown` | Controls how cautiously pods are removed |
| `--kubelet-insecure-tls` | Required on local clusters (Kind/minikube); never in production |

---

## Quick Reference

```bash
# Metrics Server
kubectl top nodes
kubectl top pods -A
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# HPA imperative
kubectl autoscale deployment <n> --cpu-percent=50 --min=1 --max=10
kubectl get hpa
kubectl get hpa --watch
kubectl describe hpa <n>
kubectl delete hpa <n>

# HPA declarative
kubectl apply -f hpa.yaml

# Load generator (quick)
kubectl run load \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://<service>; done"
```

```yaml
# Minimal HPA v2
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```
