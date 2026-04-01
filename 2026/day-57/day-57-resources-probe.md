# Day 57 – Resource Requests, Limits, and Probes

## Overview

Your Pods are running, but Kubernetes has no idea how much CPU or memory they need — and no way to detect if they are stuck or unhealthy. Today you fix both: **resource requests/limits** for smart scheduling, and **probes** for automatic failure detection and recovery.

---

## Resource Requests vs Limits

```
Requests = what the scheduler uses to PLACE the pod (guaranteed minimum)
Limits   = what kubelet ENFORCES at runtime (hard ceiling)

        Node capacity: 4 CPU, 8Gi RAM
        ┌──────────────────────────────────┐
        │  Pod A: request 1 CPU / limit 2  │
        │  Pod B: request 1 CPU / limit 2  │
        │  Pod C: request 1 CPU / limit 2  │
        │  (1 CPU left unscheduled)        │
        └──────────────────────────────────┘
```

- **CPU** is *compressible* — if a container exceeds its limit, it is **throttled** (slowed down), not killed.
- **Memory** is *incompressible* — if a container exceeds its limit, it is **killed** (OOMKilled). No mercy.

---

## Task 1: Resource Requests and Limits

```yaml
# resource-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: 100m       # 0.1 CPU guaranteed
        memory: 128Mi
      limits:
        cpu: 250m       # 0.25 CPU max
        memory: 256Mi
```

```bash
kubectl apply -f resource-pod.yaml
kubectl describe pod resource-pod
# Look for:
#   Requests:  cpu: 100m  memory: 128Mi
#   Limits:    cpu: 250m  memory: 256Mi
#   QoS Class: Burstable
```

### CPU and Memory Units

| Unit | Meaning |
| --- | --- |
| `100m` CPU | 100 millicores = 0.1 CPU core |
| `1` CPU | 1 full core = 1000m |
| `128Mi` | 128 mebibytes (1 Mi = 1,048,576 bytes) |
| `1Gi` | 1 gibibyte = 1024 Mi |

### QoS Classes

| QoS Class | Condition | Priority under pressure |
| --- | --- | --- |
| `Guaranteed` | `requests == limits` for all containers | Highest — last to be evicted |
| `Burstable` | `requests < limits` (at least one container) | Medium |
| `BestEffort` | No requests or limits set at all | Lowest — first to be evicted |

The QoS class is automatically assigned by Kubernetes based on your resource spec.

---
<img width="1440" height="772" alt="image" src="https://github.com/user-attachments/assets/57d1b985-81af-4351-b61d-76bd36bd4b53" />

## Task 2: OOMKilled — Exceeding Memory Limits

```yaml
# oom-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      limits:
        memory: 100Mi     # limit is 100Mi
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
    # tries to allocate 200M -- twice the limit
```

```bash
kubectl apply -f oom-pod.yaml
kubectl get pod oom-pod -w
# oom-pod   0/1   OOMKilled   1   5s
# oom-pod   0/1   CrashLoopBackOff  ...

kubectl describe pod oom-pod
# State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

### Exit Code 137

```
Exit code 137 = 128 + 9 (SIGKILL)
```

When the Linux kernel OOM killer terminates a process, it sends SIGKILL (signal 9). The container exit code is `128 + signal_number = 137`. This is the definitive sign of an out-of-memory kill.

| Exit Code | Meaning |
| --- | --- |
| `0` | Clean exit |
| `1` | Application error |
| `137` | OOMKilled (128 + SIGKILL) |
| `143` | Graceful termination (128 + SIGTERM) |

---
<img width="926" height="487" alt="image" src="https://github.com/user-attachments/assets/fad666b9-8526-4a91-8f9f-b815463435f8" />

## Task 3: Pending Pod — Requesting Too Much

```yaml
# impossible-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: impossible-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: 100       # 100 full CPU cores!
        memory: 128Gi  # 128 GiB RAM
```

```bash
kubectl apply -f impossible-pod.yaml
kubectl get pod impossible-pod
# NAME             STATUS    REASON
# impossible-pod   Pending

kubectl describe pod impossible-pod
# Events:
#   Warning  FailedScheduling  0/1 nodes are available:
#   1 Insufficient cpu, 1 Insufficient memory.
```

The scheduler cannot find any node with enough capacity. The pod stays `Pending` forever until resources become available or the request is reduced.

---

## Task 4: Liveness Probe

A **liveness probe** detects stuck or broken containers. If it fails `failureThreshold` times in a row, Kubernetes **restarts** the container.

```yaml
# liveness-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
  - name: app
    image: busybox:latest
    command:
    - sh
    - -c
    - |
      touch /tmp/healthy
      sleep 30
      rm /tmp/healthy
      sleep 3600
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5    # wait 5s before first check
      periodSeconds: 5          # check every 5s
      failureThreshold: 3       # restart after 3 consecutive failures
```

```bash
kubectl apply -f liveness-pod.yaml
kubectl get pod liveness-pod -w
# At ~30s: /tmp/healthy is deleted
# At ~45s: 3rd failure triggers restart
# RESTARTS column increments from 0 to 1

kubectl describe pod liveness-pod
# Liveness probe failed: cat: /tmp/healthy: No such file or directory
# Container will be restarted
```

---
<img width="1347" height="777" alt="image" src="https://github.com/user-attachments/assets/f43e02e8-4cc1-4af6-b49b-24c1e5fbce5b" />

## Task 5: Readiness Probe

A **readiness probe** controls traffic routing. Failure **removes** the Pod from Service endpoints but does **not** restart it.

```yaml
# readiness-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
  labels:
    app: readiness-test
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

```bash
kubectl apply -f readiness-pod.yaml
kubectl expose pod readiness-pod --port=80 --name=readiness-svc

# Pod is READY and receiving traffic
kubectl get pods
# readiness-pod   1/1   Running
kubectl get endpoints readiness-svc
# 10.244.0.5:80   <-- pod IP listed

# Break it: delete the index.html file
kubectl exec readiness-pod -- rm /usr/share/nginx/html/index.html

# Wait ~15 seconds
kubectl get pods
# readiness-pod   0/1   Running    <-- 0/1 = not ready, but NOT restarted!
kubectl get endpoints readiness-svc
# <none>   <-- removed from load balancer!
```

**Key difference from liveness:** The container is still running (RESTARTS stays at 0), but traffic is no longer sent to it. Fix the issue and readiness probe passes again — pod rejoins endpoints automatically.

---
<img width="1497" height="772" alt="image" src="https://github.com/user-attachments/assets/23cf536d-d0e7-4a67-a64e-500dcb723aa6" />

## Task 6: Startup Probe

A **startup probe** gives slow-starting containers extra time to initialize. While the startup probe is running, **liveness and readiness probes are disabled**. If the startup probe fails, the container is killed.

```yaml
# startup-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-pod
spec:
  containers:
  - name: slow-start
    image: busybox:latest
    command:
    - sh
    - -c
    - |
      echo "Initializing..."
      sleep 20
      touch /tmp/started
      echo "Ready!"
      sleep 3600
    startupProbe:
      exec:
        command: ["cat", "/tmp/started"]
      periodSeconds: 5
      failureThreshold: 12   # 5s * 12 = 60 second budget for startup
    livenessProbe:
      exec:
        command: ["cat", "/tmp/started"]
      periodSeconds: 10
      failureThreshold: 3
```

```bash
kubectl apply -f startup-pod.yaml
kubectl get pod startup-pod -w
# Pod stays 0/1 for ~20s while sleeping
# startupProbe fails for ~4 checks (fine, under failureThreshold: 12)
# At 20s: /tmp/started is created
# startupProbe passes
# livenessProbe takes over
# Pod shows 1/1 Running
```

**What if `failureThreshold: 2`?** The startup probe would fail at 10 seconds (2 checks × 5s each). The container gets killed before it finishes its 20-second initialization. It would go into `CrashLoopBackOff`.

---

## Probe Types Comparison

| Probe | What it does on failure | Container restarted? | Traffic removed? |
| --- | --- | --- | --- |
| `livenessProbe` | Detects stuck/broken containers | **Yes** | N/A |
| `readinessProbe` | Detects containers not ready for traffic | No | **Yes** |
| `startupProbe` | Gives slow containers time to initialize | **Yes** (if exceeded) | Yes (never ready) |

### Probe Methods

| Method | How it works |
| --- | --- |
| `exec` | Runs a command inside the container; success = exit code 0 |
| `httpGet` | HTTP GET request; success = status code 200–399 |
| `tcpSocket` | TCP connection; success = connection accepted |

### Probe Timing Fields

| Field | Default | Meaning |
| --- | --- | --- |
| `initialDelaySeconds` | 0 | Wait this long before first probe |
| `periodSeconds` | 10 | How often to probe |
| `timeoutSeconds` | 1 | Probe times out after this many seconds |
| `successThreshold` | 1 | Consecutive successes needed to pass |
| `failureThreshold` | 3 | Consecutive failures needed to act |

---
<img width="886" height="157" alt="image" src="https://github.com/user-attachments/assets/1a4409a5-52ef-478c-8011-e7b7929cc98c" />

## Task 7: Clean Up

```bash
kubectl delete pod resource-pod oom-pod impossible-pod liveness-pod readiness-pod startup-pod
kubectl delete service readiness-svc
kubectl get pods
kubectl get services
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| `requests` | Scheduler uses this to decide placement; guaranteed minimum |
| `limits` | kubelet enforces at runtime; hard ceiling |
| CPU throttling | Container slowed down when over CPU limit; not killed |
| OOMKilled | Container killed when over memory limit; exit code 137 |
| QoS: Guaranteed | requests == limits; last to be evicted under pressure |
| QoS: Burstable | requests < limits; medium eviction priority |
| QoS: BestEffort | no limits set; first to be evicted |
| Liveness probe | Failure = container restart |
| Readiness probe | Failure = removed from Service endpoints; NOT restarted |
| Startup probe | Disables liveness + readiness until startup passes; failure = kill |
| `failureThreshold` | How many consecutive failures before action is taken |

---

## Quick Reference

```yaml
# Resource requests and limits
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

# Liveness probe (exec)
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3

# Readiness probe (httpGet)
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

# Startup probe (for slow containers)
startupProbe:
  exec:
    command: ["cat", "/tmp/started"]
  periodSeconds: 5
  failureThreshold: 24   # 24 * 5s = 2 min budget

# TCP probe
livenessProbe:
  tcpSocket:
    port: 5432
  periodSeconds: 10
```

```bash
# Inspect resource usage
kubectl describe pod <n>          # see Requests/Limits/QoS
kubectl top pod                   # live CPU + memory (requires metrics-server)
kubectl top node                  # node resource usage

# Debug OOMKilled
kubectl describe pod <n>          # Reason: OOMKilled, Exit Code: 137
kubectl logs <n> --previous       # logs from the killed container
```
