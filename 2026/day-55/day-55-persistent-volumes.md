# Day 55 – Persistent Volumes (PV) and Persistent Volume Claims (PVC)

## Overview

Containers are ephemeral — when a Pod dies, everything inside it disappears. That's catastrophic for databases and any app that needs data to survive restarts. Today you fix this with **Persistent Volumes (PV)** and **Persistent Volume Claims (PVC)**.

---

## The Storage Problem

```
Pod starts  →  writes /data/message.txt
Pod crashes →  new Pod starts
New Pod     →  /data/message.txt is GONE
```

Kubernetes separates **storage provisioning** (admin's job) from **storage consumption** (developer's job) using two resources:

- **PersistentVolume (PV)** — a piece of storage in the cluster. Cluster-wide, not namespaced.
- **PersistentVolumeClaim (PVC)** — a request for storage by a Pod. Namespaced.

```
Admin creates PV  →  Developer creates PVC  →  Pod mounts PVC
(the storage)        (the request)              (uses the storage)
```

---

## Task 1: See the Problem — Data Lost on Pod Deletion

```yaml
# ephemeral-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ephemeral-writer
spec:
  containers:
  - name: writer
    image: busybox:latest
    command: ["sh", "-c", "echo 'Written at: '$(date) > /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: temp-storage
      mountPath: /data
  volumes:
  - name: temp-storage
    emptyDir: {}       # wiped when the Pod is deleted
```

```bash
kubectl apply -f ephemeral-pod.yaml

# Verify the file exists
kubectl exec ephemeral-writer -- cat /data/message.txt
# Written at: Mon Mar 10 09:23:11 UTC 2025

# Delete and recreate the Pod
kubectl delete pod ephemeral-writer
kubectl apply -f ephemeral-pod.yaml

# Check the file again
kubectl exec ephemeral-writer -- cat /data/message.txt
# Written at: Mon Mar 10 09:24:55 UTC 2025  ← DIFFERENT timestamp!
```

**Result:** The old message is gone. `emptyDir` is tied to the Pod's lifetime — it is created when the Pod starts and destroyed when the Pod stops.

### Volume Types to Know

| Type | Lifetime | Use Case |
| --- | --- | --- |
| `emptyDir` | Pod lifetime | Scratch space, temp files, shared between containers in a Pod |
| `hostPath` | Node lifetime | Dev/testing only — data tied to a specific node |
| `persistentVolumeClaim` | Independent | Production data that survives Pod restarts |

---

## Task 2: Create a PersistentVolume (Static Provisioning)

```yaml
# manual-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/k8s-pv-data    # directory on the node
```

```bash
kubectl apply -f manual-pv.yaml
kubectl get pv
# NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS
# manual-pv   1Gi        RWO            Retain           Available   manual
```

### Access Modes

| Mode | Short | Meaning |
| --- | --- | --- |
| `ReadWriteOnce` | RWO | Read-write by a single node at a time |
| `ReadOnlyMany` | ROX | Read-only by many nodes simultaneously |
| `ReadWriteMany` | RWX | Read-write by many nodes simultaneously |

### Reclaim Policies

| Policy | What happens when PVC is deleted |
| --- | --- |
| `Retain` | PV stays; data preserved; must be manually reclaimed |
| `Delete` | PV and underlying storage are automatically deleted |
| `Recycle` | Deprecated — basic scrub (`rm -rf`) then made available again |

> `hostPath` stores data in a directory on the node. If the Pod moves to a different node, the data is inaccessible. Fine for learning, never for production.
> 

---
<img width="1408" height="140" alt="image" src="https://github.com/user-attachments/assets/b338f6e0-68e2-4f84-a10a-b0c0e4c4a247" />

## Task 3: Create a PersistentVolumeClaim

```yaml
# manual-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi    # requesting 500Mi from a 1Gi PV
```

```bash
kubectl apply -f manual-pvc.yaml

kubectl get pvc
# NAME         STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS
# manual-pvc   Bound    manual-pv   1Gi        RWO            manual

kubectl get pv
# NAME        CAPACITY   STATUS   CLAIM              STORAGECLASS
# manual-pv   1Gi        Bound    default/manual-pvc  manual
```

**Both show `Bound`** — Kubernetes matched the PVC to the PV by `storageClassName` and `accessModes`. The PVC requested 500Mi but the PV has 1Gi; Kubernetes binds the whole PV to the PVC (no partial binding in static provisioning).

### PV Status Lifecycle

```
Available  →  Bound  →  Released  →  (Reclaim policy applied)
```

| Status | Meaning |
| --- | --- |
| `Available` | PV exists, not yet claimed by any PVC |
| `Bound` | PV is claimed by a PVC |
| `Released` | PVC was deleted; PV still exists with old data (Retain policy) |
| `Failed` | Automatic reclamation failed |

### Why PVC Stays Pending

If your PVC shows `Pending`, check:

- `storageClassName` in PVC matches PV exactly
- `accessModes` in PVC is a subset of PV's access modes
- Requested capacity doesn't exceed PV capacity
- No other PVC already bound to the PV

---
<img width="1297" height="145" alt="image" src="https://github.com/user-attachments/assets/f593b031-b43c-40d1-bbc7-03d136bfc7ad" />

## Task 4: Use the PVC in a Pod — Data That Survives

```yaml
# persistent-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: persistent-writer
spec:
  containers:
  - name: writer
    image: busybox:latest
    command: ["sh", "-c", "echo 'Pod 1 wrote at: '$(date) >> /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: manual-pvc    # reference the PVC by name
```

```bash
kubectl apply -f persistent-pod.yaml

# Verify data exists
kubectl exec persistent-writer -- cat /data/message.txt
# Pod 1 wrote at: Mon Mar 10 09:30:00 UTC 2025

# Delete the Pod
kubectl delete pod persistent-writer

# Update command to say "Pod 2" and recreate
# (edit persistent-pod.yaml: change "Pod 1" to "Pod 2")
kubectl apply -f persistent-pod.yaml

# Check the file — should contain BOTH entries
kubectl exec persistent-writer -- cat /data/message.txt
# Pod 1 wrote at: Mon Mar 10 09:30:00 UTC 2025
# Pod 2 wrote at: Mon Mar 10 09:31:45 UTC 2025
```

**Result:** Both entries are present. The data on the PV survived Pod deletion and is accessible by the new Pod.

---
<img width="983" height="72" alt="image" src="https://github.com/user-attachments/assets/4b3997d6-daad-4b9e-b2f1-c951ab3e5bbd" />

## Task 5: StorageClasses and Dynamic Provisioning

```bash
kubectl get storageclass
# NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
# standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer

kubectl describe storageclass standard
```

### What is a StorageClass?

A StorageClass defines **how storage is provisioned** — which backend, which reclaim policy, and when to bind. With dynamic provisioning, developers only create PVCs; the StorageClass automatically creates the matching PV.

```
Static:   Admin creates PV manually  →  Developer creates PVC  →  Kubernetes binds them
Dynamic:  Developer creates PVC  →  StorageClass auto-creates PV  →  Kubernetes binds them
```

### StorageClass Key Fields

| Field | Meaning |
| --- | --- |
| `provisioner` | The plugin that creates storage (e.g., `rancher.io/local-path`, `ebs.csi.aws.com`) |
| `reclaimPolicy` | `Delete` (default) or `Retain` |
| `volumeBindingMode` | `Immediate` (provision immediately) or `WaitForFirstConsumer` (wait until a Pod needs it) |

---

## Task 6: Dynamic Provisioning

```yaml
# dynamic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: standard    # use your cluster's default StorageClass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
```

```bash
kubectl apply -f dynamic-pvc.yaml

# A PV is created automatically
kubectl get pv
# NAME          CAPACITY   STATUS   CLAIM                 STORAGECLASS
# manual-pv     1Gi        Bound    default/manual-pvc    manual
# pvc-a1b2c3d4  256Mi      Bound    default/dynamic-pvc   standard  ← auto-created!

kubectl get pvc
# manual-pvc    Bound   manual-pv     1Gi    RWO   manual
# dynamic-pvc   Bound   pvc-a1b2c3d4  256Mi  RWO   standard
```

```yaml
# pod-using-dynamic-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-writer
spec:
  containers:
  - name: writer
    image: busybox:latest
    command: ["sh", "-c", "echo 'Dynamic PV works!' > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: dynamic-storage
      mountPath: /data
  volumes:
  - name: dynamic-storage
    persistentVolumeClaim:
      claimName: dynamic-pvc
```

```bash
kubectl apply -f pod-using-dynamic-pvc.yaml
kubectl exec dynamic-writer -- cat /data/test.txt
# Dynamic PV works!
```

---

## Task 7: Clean Up — Reclaim Policy in Action

```bash
# Step 1: Delete pods first
kubectl delete pod persistent-writer dynamic-writer

# Step 2: Delete PVCs
kubectl delete pvc manual-pvc dynamic-pvc

# Step 3: Check what happened to the PVs
kubectl get pv
# NAME          CAPACITY   STATUS     STORAGECLASS
# manual-pv     1Gi        Released   manual        ← Retain: PV still exists!
# pvc-a1b2c3d4  (gone)                              ← Delete: PV auto-deleted!

# Step 4: Delete the retained PV manually
kubectl delete pv manual-pv
```

### Why the Difference?

|  | `manual-pv` | `pvc-a1b2c3d4` |
| --- | --- | --- |
| Reclaim policy | `Retain` | `Delete` |
| When PVC deleted | PV shows `Released`; data preserved | PV and storage auto-deleted |
| Manual cleanup needed | Yes | No |
| Use case | Critical data you want to keep | Ephemeral test/dev storage |

> After a PV is `Released`, you cannot immediately rebind it to a new PVC (even if you delete and recreate the PVC). You must manually delete the `claimRef` in the PV spec or delete and recreate the PV.
> 

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| `emptyDir` | Pod-lifetime scratch space; gone when Pod dies |
| PV | Cluster-wide storage resource; created by admin (static) or StorageClass (dynamic) |
| PVC | Namespaced storage request by a developer; binds to a matching PV |
| Static provisioning | Admin manually creates PVs; PVCs bind to existing PVs |
| Dynamic provisioning | StorageClass auto-creates PVs when a PVC is submitted |
| `Retain` policy | PV survives PVC deletion; data preserved; manual cleanup required |
| `Delete` policy | PV and storage auto-deleted when PVC is deleted |
| PV → `Released` | PVC was deleted but PV exists (Retain); not automatically reusable |
| `WaitForFirstConsumer` | PV is only provisioned when a Pod actually needs it (avoids AZ mismatch) |

---

## Quick Reference

```bash
# PVs (cluster-wide)
kubectl get pv
kubectl describe pv <name>
kubectl delete pv <name>

# PVCs (namespaced)
kubectl get pvc
kubectl get pvc -n <namespace>
kubectl describe pvc <name>
kubectl delete pvc <name>

# StorageClasses
kubectl get storageclass
kubectl describe storageclass <name>

# Debug a Pending PVC
kubectl describe pvc <name>   # Events section shows why it's pending
kubectl get pv                 # Check Available PVs and their attributes
```

```yaml
# PV skeleton
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/data

# PVC skeleton
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: manual
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 500Mi

# Mount PVC in Pod
volumes:
- name: my-vol
  persistentVolumeClaim:
    claimName: my-pvc
containers:
- volumeMounts:
  - name: my-vol
    mountPath: /data
```
