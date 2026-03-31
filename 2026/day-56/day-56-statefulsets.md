# Day 56 – Kubernetes StatefulSets

## Overview

Deployments work great for stateless apps where any pod is interchangeable. But databases need **stable identity** — fixed pod names, ordered startup, and dedicated storage per replica. StatefulSets provide exactly that.

---

## Deployment vs StatefulSet

| Feature | Deployment | StatefulSet |
| --- | --- | --- |
| Pod names | Random (`app-7d6f-abc`) | Stable, ordered (`web-0`, `web-1`, `web-2`) |
| Startup order | All at once (parallel) | Ordered: `pod-0` → `pod-1` → `pod-2` |
| Shutdown order | Random | Reverse: `pod-2` → `pod-1` → `pod-0` |
| Storage | Shared PVC (all pods same volume) | Each pod gets its own dedicated PVC |
| Network identity | No stable hostname | Stable DNS per pod |
| Use case | Stateless: web servers, APIs | Stateful: databases, Kafka, Zookeeper |

---

## Task 1: Understand the Problem

```bash
# Create a Deployment and observe random pod names
kubectl create deployment demo --image=nginx --replicas=3
kubectl get pods
# demo-7d6f5b8c4-abcde
# demo-7d6f5b8c4-fghij
# demo-7d6f5b8c4-xyzwv

# Delete one pod
kubectl delete pod demo-7d6f5b8c4-abcde

# Replacement gets a DIFFERENT random name
kubectl get pods
# demo-7d6f5b8c4-mnopq   <-- new name!

# Clean up before continuing
kubectl delete deployment demo
```

### Why Random Names Are a Problem for Databases

In a database cluster (e.g., PostgreSQL primary + replicas, Kafka brokers), each node has a **fixed role and identity**:

- `db-0` is always the primary writer
- `db-1` and `db-2` are always read replicas
- Other services connect to `db-0` specifically for writes
- Config files reference nodes by name

If `db-0` restarts as `db-7f3k2-xvz`, the entire cluster loses its routing logic. StatefulSets guarantee that `web-0` is always `web-0`.

---

## Task 2: Create a Headless Service

A Headless Service (`clusterIP: None`) does NOT load-balance to a single virtual IP. Instead, DNS returns the individual pod IPs directly, enabling per-pod DNS names.

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None       # this makes it Headless
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f headless-service.yaml
kubectl get services
# NAME   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
# web    ClusterIP   None         <none>        80/TCP
```

**CLUSTER-IP shows `None`** — this is correct for a Headless Service. StatefulSets require a Headless Service to create individual DNS records for each pod.

---
<img width="815" height="192" alt="image" src="https://github.com/user-attachments/assets/c6298ac3-6037-4754-9291-34bb92c54797" />

## Task 3: Create a StatefulSet

```yaml
# web-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web          # must match the Headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:      # each pod gets its OWN PVC
  - metadata:
      name: web-data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 100Mi
```

```bash
kubectl apply -f web-statefulset.yaml

# Watch ordered creation
kubectl get pods -l app=web -w
# web-0   0/1   ContainerCreating
# web-0   1/1   Running           <-- web-0 Ready first
# web-1   0/1   ContainerCreating <-- then web-1 starts
# web-1   1/1   Running
# web-2   0/1   ContainerCreating <-- then web-2
# web-2   1/1   Running

# Check auto-created PVCs
kubectl get pvc
# web-data-web-0   Bound   ...
# web-data-web-1   Bound   ...
# web-data-web-2   Bound   ...
```

### PVC Naming Pattern

```
<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>

web-data - web - 0  ==>  web-data-web-0
web-data - web - 1  ==>  web-data-web-1
web-data - web - 2  ==>  web-data-web-2
```

Each pod is permanently bonded to its own PVC. If `web-0` is deleted and recreated, it reconnects to `web-data-web-0` — not any other PVC.

---
<img width="856" height="412" alt="image" src="https://github.com/user-attachments/assets/d945d101-33b5-4deb-a153-cf6a35602016" />

## Task 4: Stable Network Identity

Each StatefulSet pod gets a DNS name in this format:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local

web-0.web.default.svc.cluster.local
web-1.web.default.svc.cluster.local
web-2.web.default.svc.cluster.local
```

```bash
# Test DNS resolution from inside the cluster
kubectl run dns-test --image=busybox:latest --rm -it --restart=Never -- sh

# Resolve each pod individually
nslookup web-0.web.default.svc.cluster.local
nslookup web-1.web.default.svc.cluster.local
nslookup web-2.web.default.svc.cluster.local

# Short form (within same namespace)
nslookup web-0.web

exit

# Verify IPs match pod IPs
kubectl get pods -o wide
```

The `nslookup` results map directly to the individual pod IPs shown in `kubectl get pods -o wide` — not a shared virtual IP like a regular ClusterIP Service.

### Regular Service vs Headless Service DNS

|  | Regular ClusterIP Service | Headless Service |
| --- | --- | --- |
| DNS resolves to | Single virtual ClusterIP | Individual pod IPs |
| Load balancing | Kubernetes kube-proxy | Client-side (or none) |
| Per-pod DNS | Not possible | `pod-0.svc`, `pod-1.svc`, etc. |

---
<img width="1470" height="726" alt="image" src="https://github.com/user-attachments/assets/65eb01d2-83b4-4028-bdcf-b4d12a972e59" />

## Task 5: Stable Storage — Data Survives Pod Deletion

```bash
# Write unique data to each pod
kubectl exec web-0 -- sh -c "echo 'Data from web-0' > /usr/share/nginx/html/index.html"
kubectl exec web-1 -- sh -c "echo 'Data from web-1' > /usr/share/nginx/html/index.html"
kubectl exec web-2 -- sh -c "echo 'Data from web-2' > /usr/share/nginx/html/index.html"

# Verify
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
# Data from web-0

# Delete web-0
kubectl delete pod web-0

# Watch it come back with the SAME name
kubectl get pods -w
# web-0   0/1   Terminating
# web-0   0/1   Pending
# web-0   0/1   ContainerCreating
# web-0   1/1   Running

# Check data is still there
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
# Data from web-0   <-- same data, same PVC!
```

**What happened:** The new `web-0` pod started up and reconnected to the `web-data-web-0` PVC automatically. The data written by the previous `web-0` was never lost.

---
<img width="1418" height="427" alt="image" src="https://github.com/user-attachments/assets/224fe006-081a-43d7-875a-e676fd1e922d" />

## Task 6: Ordered Scaling

```bash
# Scale up to 5 (adds web-3, then web-4 in order)
kubectl scale statefulset web --replicas=5
kubectl get pods -w
# web-3 starts only after web-2 is Running
# web-4 starts only after web-3 is Running

kubectl get pvc
# web-data-web-0  Bound
# web-data-web-1  Bound
# web-data-web-2  Bound
# web-data-web-3  Bound  <-- new
# web-data-web-4  Bound  <-- new

# Scale down to 3 (terminates web-4 first, then web-3)
kubectl scale statefulset web --replicas=3
kubectl get pods -w
# Reverse order: web-4 terminates first, then web-3

# Check PVCs after scale-down
kubectl get pvc
# ALL 5 PVCs still exist!
# web-data-web-3 and web-data-web-4 are retained
```

### Why PVCs Survive Scale-Down

Kubernetes intentionally keeps PVCs when scaling down a StatefulSet. If you scale back up, `web-3` reconnects to `web-data-web-3` and your data is still there. This is a safety feature — accidental scale-down doesn't destroy data.

To actually delete the PVCs, you must do so manually.

---

## Task 7: Clean Up

```bash
# Step 1: Delete the StatefulSet
kubectl delete statefulset web

# Step 2: Check PVCs - they are NOT auto-deleted
kubectl get pvc
# web-data-web-0   Bound   (still here!)
# web-data-web-1   Bound
# web-data-web-2   Bound
# web-data-web-3   Bound
# web-data-web-4   Bound

# Step 3: Delete PVCs manually
kubectl delete pvc web-data-web-0 web-data-web-1 web-data-web-2 web-data-web-3 web-data-web-4
# Or all at once:
kubectl delete pvc -l app=web

# Step 4: Delete the Headless Service
kubectl delete service web

# Verify everything is clean
kubectl get pvc
kubectl get pods
kubectl get services
```

**PVCs are NOT auto-deleted when you delete a StatefulSet.** This is intentional — your data is protected from accidental deletion. Always clean up PVCs separately.

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| StatefulSet purpose | Stable pod identity + ordered operations + per-pod storage |
| `serviceName` | Must reference an existing Headless Service (`clusterIP: None`) |
| Headless Service | No virtual IP; DNS returns individual pod IPs directly |
| Pod names | Always `<name>-0`, `<name>-1`, `<name>-2` — never random |
| PVC naming | `<template>-<statefulset>-<ordinal>` e.g. `web-data-web-0` |
| Pod DNS | `<pod>.<service>.<namespace>.svc.cluster.local` |
| Startup order | Sequential: 0, 1, 2 — each waits for previous to be Ready |
| Shutdown order | Reverse: 2, 1, 0 |
| Scale-down PVCs | Not deleted automatically — data preserved if you scale back up |
| StatefulSet deletion | Does NOT delete PVCs — clean up manually |
| `kubectl get sts` | Short name for StatefulSets |

---

## Quick Reference

```bash
# StatefulSet commands
kubectl get sts
kubectl describe sts <n>
kubectl delete sts <n>
kubectl scale statefulset <n> --replicas=5

# Watch ordered creation
kubectl get pods -l app=<label> -w

# Check per-pod PVCs
kubectl get pvc

# Test per-pod DNS
kubectl run dns-test --image=busybox --rm -it --restart=Never -- sh
nslookup web-0.web.default.svc.cluster.local

# Write to specific pod
kubectl exec web-0 -- sh -c "echo 'data' > /path/file"
kubectl exec web-0 -- cat /path/file

# Clean up everything (StatefulSet does NOT delete PVCs!)
kubectl delete sts web
kubectl delete pvc -l app=web
kubectl delete svc web
```

```yaml
# StatefulSet skeleton
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web          # headless service name
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: app
        image: nginx:1.25
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 1Gi

# Headless Service (required)
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
```
