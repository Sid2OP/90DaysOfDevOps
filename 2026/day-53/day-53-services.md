# Day 53 – Kubernetes Services

## Overview

You have Deployments running multiple Pods, but Pods get random IPs that change every restart. **Services** solve this by providing a stable network endpoint, a consistent DNS name, and automatic load balancing across all matching Pods.

---

## Why Services?

```
Without a Service:
  Client → 10.244.0.5 (Pod 1)   ← IP changes on restart!
  Client → 10.244.0.6 (Pod 2)
  Client → 10.244.0.7 (Pod 3)

With a Service:
  Client → Service (10.96.0.1, stable) → Pod 1
                                        → Pod 2
                                        → Pod 3
```

A Service provides:

- A **stable IP and DNS name** that never changes, even as pods restart
- **Load balancing** across all pods that match its label selector
- **Service discovery** via Kubernetes DNS

---

## Task 1: Deploy the Application

```yaml
# app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f app-deployment.yaml
kubectl get pods -o wide    # note the individual pod IPs
```
<img width="1475" height="465" alt="image" src="https://github.com/user-attachments/assets/0b8bbbce-8a47-4492-b539-f6329c37ccad" />

## Task 2: ClusterIP Service (Internal Access)

ClusterIP is the **default** Service type. It gives pods a stable internal IP that is only reachable from within the cluster.

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app        # routes to all pods with this label
  ports:
  - port: 80            # port the Service listens on
    targetPort: 80      # port on the Pod to forward to
```

```bash
kubectl apply -f clusterip-service.yaml
kubectl get services
# NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
# web-app-clusterip   ClusterIP   10.96.145.200   <none>        80/TCP

# See which pod IPs the Service is routing to
kubectl get endpoints web-app-clusterip

# Test from inside the cluster (temporary pod)
kubectl run test-client --image=busybox:latest --rm -it --restart=Never -- sh
# Inside the pod:
wget -qO- http://web-app-clusterip
exit
```

### Key Fields

| Field | Purpose |
| --- | --- |
| `selector.app: web-app` | Routes traffic to all pods with this label |
| `port: 80` | The port the Service itself listens on |
| `targetPort: 80` | The port on the Pod to forward traffic to (can differ from `port`) |
| `CLUSTER-IP` | The stable virtual IP assigned to this Service |

> **`port` and `targetPort` don't have to match.** Example: `port: 8080, targetPort: 80` — clients hit 8080, pods receive on 80.
> 

---
<img width="927" height="91" alt="image" src="https://github.com/user-attachments/assets/7ba65a88-3583-41c5-a216-5b42d7c3ed37" />

## Task 3: Service Discovery with DNS

Kubernetes runs **CoreDNS**, which gives every Service an automatic DNS entry:

```
<service-name>.<namespace>.svc.cluster.local
```

```bash
kubectl run dns-test --image=busybox:latest --rm -it --restart=Never -- sh

# Short name (within same namespace)
wget -qO- http://web-app-clusterip

# Full DNS name (works across namespaces)
wget -qO- http://web-app-clusterip.default.svc.cluster.local

# DNS lookup
nslookup web-app-clusterip
# Server: 10.96.0.10    (CoreDNS ClusterIP)
# Address: 10.96.145.200  (matches kubectl get services CLUSTER-IP)

exit
```

### DNS Name Patterns

| Scope | DNS name | Example |
| --- | --- | --- |
| Same namespace | `<service>` | `web-app-clusterip` |
| Same namespace | `<service>.<ns>` | `web-app-clusterip.default` |
| Cross-namespace | `<service>.<ns>.svc.cluster.local` | `web-app-clusterip.default.svc.cluster.local` |

In practice: use the short name within the same namespace, full name when reaching across namespaces.

---
<img width="1342" height="687" alt="image" src="https://github.com/user-attachments/assets/6d69a622-883c-403d-b28e-6b9340df2097" />

## Task 4: NodePort Service (External Access via Node)

A NodePort opens a port on **every node** in the cluster, making the service reachable from outside.

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80          # ClusterIP port (internal)
    targetPort: 80    # Pod port
    nodePort: 30080   # External port on every node (30000-32767)
```

```bash
kubectl apply -f nodeport-service.yaml
kubectl get services

# Access externally:
# Minikube
minikube service web-app-nodeport --url

# Kind — get node IP first
kubectl get nodes -o wide
curl <node-internal-ip>:30080

# Docker Desktop
curl http://localhost:30080
```

**Traffic flow:** `<NodeIP>:30080` → Service → Pod:80

**NodePort range:** 30000–32767. If you omit `nodePort`, Kubernetes picks one automatically.

---
<img width="1640" height="731" alt="image" src="https://github.com/user-attachments/assets/e906b9b9-e8f5-471c-8d2e-8f862eea1293" />

## Task 5: LoadBalancer Service (Cloud External Access)

A LoadBalancer Service provisions a **real external load balancer** from your cloud provider (AWS ELB, GCP CLB, Azure LB) and routes external traffic to your nodes.

```yaml
# loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f loadbalancer-service.yaml
kubectl get services
# NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP
# web-app-loadbalancer   LoadBalancer   10.96.210.33   <pending>
```

**Why `<pending>` on local clusters?** Kind, Minikube, and Docker Desktop have no cloud provider integration, so there is nothing to provision an external IP. The field stays `<pending>` indefinitely.

```bash
# Minikube can simulate it:
minikube tunnel     # run in a separate terminal
kubectl get services   # EXTERNAL-IP now shows an IP
```

On a real cloud cluster (EKS, GKE, AKS), the EXTERNAL-IP is a public hostname or IP provisioned automatically within ~60 seconds.

---

## Task 6: Service Types Side by Side

```bash
kubectl get services -o wide
kubectl describe service web-app-loadbalancer
# Notice: LoadBalancer has a ClusterIP AND a NodePort AND the LB config
```

### Service Types Comparison

| Type | Accessible From | Use Case |
| --- | --- | --- |
| `ClusterIP` | Inside the cluster only | Internal service-to-service communication |
| `NodePort` | Outside via `<NodeIP>:<NodePort>` | Development, testing, bare-metal clusters |
| `LoadBalancer` | Outside via cloud load balancer | Production traffic in cloud environments |

### Service Types Build on Each Other

```
LoadBalancer
  └─ creates a NodePort
       └─ creates a ClusterIP
```

A `LoadBalancer` service always has a ClusterIP and a NodePort assigned internally. You can verify this with `kubectl describe service web-app-loadbalancer` — all three IPs and ports are listed.

### Endpoints — How a Service Knows Where to Send Traffic

```bash
# See the pod IPs a Service is currently routing to
kubectl get endpoints web-app-clusterip
# NAME                ENDPOINTS
# web-app-clusterip   10.244.0.5:80,10.244.0.6:80,10.244.0.7:80
```

When a pod is deleted and replaced, its new IP is automatically added to the endpoints list and the old one removed. The Service's ClusterIP never changes.

---

## Task 7: Clean Up

```bash
kubectl delete -f app-deployment.yaml
kubectl delete -f clusterip-service.yaml
kubectl delete -f nodeport-service.yaml
kubectl delete -f loadbalancer-service.yaml

kubectl get pods
kubectl get services
# Only the built-in 'kubernetes' service should remain
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Why Services exist | Pod IPs are ephemeral; Services provide stable IPs and DNS names |
| `selector` | Must match pod labels exactly; if no pods match, the Service routes to nothing |
| `port` vs `targetPort` | `port` = Service listens here; `targetPort` = Pod receives here; can differ |
| ClusterIP | Default type; internal only; stable virtual IP + DNS |
| NodePort | Opens a port (30000–32767) on every node; externally accessible |
| LoadBalancer | Provisions cloud LB; `<pending>` on local clusters without a cloud provider |
| Endpoints | The real pod IPs a Service routes to; auto-updated as pods come and go |
| CoreDNS | Built-in DNS server; resolves `<svc>.<ns>.svc.cluster.local` to ClusterIP |
| LB ⊃ NodePort ⊃ ClusterIP | Each type builds on the previous one |

---

## Quick Reference

```bash
# Apply services
kubectl apply -f service.yaml
kubectl get services
kubectl get services -o wide
kubectl describe service <name>

# Check endpoints (which pod IPs the service routes to)
kubectl get endpoints <service-name>

# Test ClusterIP from inside cluster
kubectl run test --image=busybox --rm -it --restart=Never -- sh
wget -qO- http://<service-name>
nslookup <service-name>

# Minikube: get NodePort URL
minikube service <service-name> --url

# Minikube: simulate LoadBalancer
minikube tunnel

# Delete
kubectl delete service <name>
kubectl delete -f service.yaml
```
The pod IPs you see here are **ephemeral** — they change every time a pod restarts or gets replaced. This is exactly the problem Services fix.

---
