# Day 54 – Kubernetes ConfigMaps and Secrets

## Overview

Hardcoding configuration into container images means rebuilding every time a value changes. Kubernetes solves this with **ConfigMaps** for non-sensitive config and **Secrets** for sensitive data — both can be injected into pods as environment variables or mounted as files.

---

## Task 1: Create a ConfigMap from Literals

```bash
# Imperative creation with --from-literal
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080

# Inspect
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

### What the YAML looks like

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_DEBUG: "false"
  APP_ENV: production
  APP_PORT: "8080"
```
<img width="947" height="597" alt="image" src="https://github.com/user-attachments/assets/20de97ea-024f-4839-9a15-686237733bc1" />

## Task 2: Create a ConfigMap from a File

```
# health.conf
server {
    listen 80;
    location /health {
        return 200 'healthy';
        add_header Content-Type text/plain;
    }
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}
```

```bash
# Create ConfigMap from the file
# The key name (default.conf) becomes the filename when mounted
kubectl create configmap nginx-config --from-file=default.conf=health.conf

# Verify the file contents are stored
kubectl get configmap nginx-config -o yaml
```

### What the YAML looks like

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen 80;
        location /health {
            return 200 'healthy';
            ...
        }
    }
```

The entire file content is stored as the value of the `default.conf` key.

---
<img width="1223" height="573" alt="image" src="https://github.com/user-attachments/assets/93e5b67b-1037-41d7-b300-1dfef0d66c47" />

## Task 3: Use ConfigMaps in a Pod

### Method 1: `envFrom` — Inject All Keys as Env Vars

```yaml
# env-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "echo APP_ENV=$APP_ENV; echo APP_DEBUG=$APP_DEBUG; echo APP_PORT=$APP_PORT; sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config    # injects ALL keys as env vars
```

```bash
kubectl apply -f env-pod.yaml
kubectl logs env-pod
# APP_ENV=production
# APP_DEBUG=false
# APP_PORT=8080
```

### Method 2: Individual Key with `valueFrom`

```yaml
env:
- name: MY_ENV          # name in the container (can differ from ConfigMap key)
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_ENV      # which key to pull from the ConfigMap
```

### Method 3: Volume Mount — Full Config Files

```yaml
# nginx-configmap-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-config-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-conf
      mountPath: /etc/nginx/conf.d    # each ConfigMap key becomes a file here
      readOnly: true
  volumes:
  - name: nginx-conf
    configMap:
      name: nginx-config              # mounts default.conf into the directory
```

```bash
kubectl apply -f nginx-configmap-pod.yaml
kubectl get pods

# Test the /health endpoint
kubectl exec nginx-config-pod -- curl -s http://localhost/health
# healthy
```

### When to Use Each Method

| Method | Use for |
| --- | --- |
| `envFrom` (all keys) | Simple key-value app config, 12-factor apps |
| `env`  • `valueFrom` (single key) | Picking specific values, renaming keys |
| Volume mount | Full config files (nginx.conf, application.yml, .env files) |

---
<img width="1126" height="311" alt="image" src="https://github.com/user-attachments/assets/d4696cf8-5fd0-47fd-a4bf-6dd347469b0f" />

## Task 4: Create a Secret

```bash
# Imperative creation
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cureP@ssw0rd

# Inspect — values are base64-encoded
kubectl get secret db-credentials -o yaml
```

### What the YAML looks like

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  DB_PASSWORD: czNjdXJlUEBzc3cwcmQ=    # base64 encoded
  DB_USER: YWRtaW4=
```

```bash
# Decode a value
echo 'czNjdXJlUEBzc3cwcmQ=' | base64 --decode
# s3cureP@ssw0rd

# One-liner extract + decode
kubectl get secret db-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
```

### base64 Is Encoding, NOT Encryption

Anyone with `kubectl get secret` access can decode these values instantly. The real advantages of Secrets over ConfigMaps are:

| Advantage | Detail |
| --- | --- |
| **RBAC separation** | You can grant access to ConfigMaps but restrict Secrets to specific roles |
| **tmpfs storage** | Secrets mounted as volumes are stored in memory (tmpfs), not on disk |
| **Encryption at rest** | Kubernetes supports encrypting etcd data; Secret type is the target |
| **Audit logging** | Access to Secrets can be separately audited |
| **Third-party integrations** | Vault, Sealed Secrets, and External Secrets Operator all target the Secret type |

> **Always use `-n` (no newline) when base64 encoding manually:** `echo -n 'value' | base64`. Without `-n`, the trailing newline gets encoded too and the decoded value will have a hidden `\n`.
> 

---
<img width="1008" height="372" alt="image" src="https://github.com/user-attachments/assets/c84ccb9b-d97b-43ff-b9b3-80c68767ddcb" />

## Task 5: Use Secrets in a Pod

```yaml
# secret-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sh", "-c", "echo DB_USER=$DB_USER; ls /etc/db-credentials/; cat /etc/db-credentials/DB_PASSWORD; sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_USER
    volumeMounts:
    - name: db-secret-vol
      mountPath: /etc/db-credentials
      readOnly: true
  volumes:
  - name: db-secret-vol
    secret:
      secretName: db-credentials
```

```bash
kubectl apply -f secret-pod.yaml
kubectl logs secret-pod
# DB_USER=admin
# DB_PASSWORD
# DB_USER
# s3cureP@ssw0rd     <-- plaintext! Kubernetes decodes base64 when mounting
```

### Key Insight: Mounted Secret Values Are Plaintext

Kubernetes automatically base64-**decodes** Secret values when:

- Mounting as a volume — each key becomes a file, content is decoded plaintext
- Injecting as an env var via `secretKeyRef` — value is decoded plaintext

The base64 encoding only exists in etcd storage and in `kubectl get secret -o yaml` output.

---
## Task 6: Update a ConfigMap and Observe Propagation

```yaml
# live-config-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: live-config-pod
spec:
  containers:
  - name: watcher
    image: busybox:latest
    command: ["sh", "-c", "while true; do cat /config/message; echo; sleep 5; done"]
    volumeMounts:
    - name: config-vol
      mountPath: /config
  volumes:
  - name: config-vol
    configMap:
      name: live-config
```

```bash
# Create the ConfigMap
kubectl create configmap live-config --from-literal=message=hello

# Apply the pod
kubectl apply -f live-config-pod.yaml

# Watch the logs
kubectl logs live-config-pod -f
# hello
# hello
# hello

# In another terminal, update the ConfigMap
kubectl patch configmap live-config \
  --type merge \
  -p '{"data":{"message":"world"}}'

# After 30-60 seconds, the pod logs change:
# world
# world
```

### ConfigMap Update Propagation Rules

| Injection method | Auto-updates? | When? |
| --- | --- | --- |
| Volume mount | ✅ Yes | ~30–60 seconds (kubelet sync interval) |
| `envFrom` | ❌ No | Only set at pod startup; requires pod restart |
| `env`  • `valueFrom` | ❌ No | Only set at pod startup; requires pod restart |

This is a critical production gotcha: if you update a ConfigMap and expect an env-var-based app to pick up the change, **you must restart the pods**.

```bash
# Force restart pods in a deployment after ConfigMap change
kubectl rollout restart deployment/<name>
```

---
<img width="1126" height="257" alt="image" src="https://github.com/user-attachments/assets/edbc1d0d-7a2e-42d4-bafe-8d97880bd886" />

## Task 7: Clean Up

```bash
kubectl delete pod env-pod nginx-config-pod secret-pod live-config-pod
kubectl delete configmap app-config nginx-config live-config
kubectl delete secret db-credentials

# Verify
kubectl get pods
kubectl get configmaps
kubectl get secrets
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| ConfigMap | Stores non-sensitive config as plain text key-value pairs |
| Secret | Stores sensitive data base64-encoded in etcd; decoded on injection |
| `--from-literal` | Creates key-value pairs from CLI arguments |
| `--from-file=key=file` | File content becomes the value; key is the filename in the mount |
| `envFrom` | Injects ALL ConfigMap/Secret keys as env vars |
| `valueFrom` | Injects a single specific key as an env var |
| Volume mount | Each key becomes a file; content is decoded plaintext for Secrets |
| Auto-update | Volume mounts auto-update in ~60s; env vars require pod restart |
| base64 ≠ encryption | Anyone with `kubectl get secret` can decode; use RBAC to restrict access |
| `echo -n` | Always use `-n` when manually base64 encoding to avoid trailing newline |

---

## Quick Reference

```bash
# ConfigMap creation
kubectl create configmap <name> --from-literal=KEY=VALUE
kubectl create configmap <name> --from-file=key=filename
kubectl apply -f configmap.yaml

# ConfigMap inspection
kubectl get configmap <name>
kubectl get configmap <name> -o yaml
kubectl describe configmap <name>

# Secret creation
kubectl create secret generic <name> --from-literal=KEY=VALUE
kubectl create secret generic <name> --from-file=key=filename

# Secret inspection + decode
kubectl get secret <name> -o yaml
kubectl get secret <name> -o jsonpath='{.data.KEY}' | base64 --decode

# Patch a ConfigMap
kubectl patch configmap <name> --type merge -p '{"data":{"key":"newvalue"}}'

# Force pod restart after ConfigMap change
kubectl rollout restart deployment/<name>

# Base64 encoding (always -n)
echo -n 'value' | base64
echo 'encodedvalue' | base64 --decode
```
Notice: values are stored as **plain text** — no encoding, no encryption. ConfigMaps are not for secrets.

---
