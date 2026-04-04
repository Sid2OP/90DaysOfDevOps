# Day 59 – Helm — Kubernetes Package Manager

## Overview

Over the past eight days you wrote Deployments, Services, ConfigMaps, Secrets, PVCs, and more — all as individual YAML files. For a real application you might have dozens. **Helm** is the package manager for Kubernetes, like `apt` for Ubuntu. One command deploys a full stack, and another rolls it back.

---

## Three Core Helm Concepts

| Concept | Meaning |
| --- | --- |
| **Chart** | A package of Kubernetes manifest templates (like a `.deb` package) |
| **Release** | A specific installation of a chart in your cluster (you can install the same chart multiple times with different names) |
| **Repository** | A collection of charts hosted at a URL (like a package registry) |

---

## Task 1: Install Helm

```bash
# macOS
brew install helm

# Linux (official script)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows (Chocolatey)
choco install kubernetes-helm

# Verify
helm version
helm env
```

`helm env` shows where Helm stores its cache, config, and data directories on your system.

---

## Task 2: Add a Repository and Search

```bash
# Add Bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami

# Fetch latest chart index
helm repo update

# Search within added repos
helm search repo nginx
helm search repo bitnami

# Search Artifact Hub (public chart registry)
helm search hub nginx
```

Bitnami maintains hundreds of production-grade charts for common software (nginx, postgresql, redis, kafka, wordpress, etc.).

---
<img width="1435" height="792" alt="image" src="https://github.com/user-attachments/assets/bc6f45e7-8f07-4394-a503-7253589a9f1e" />

## Task 3: Install a Chart

```bash
# Install nginx — release name: my-nginx
helm install my-nginx bitnami/nginx

# See everything Helm created
kubectl get all

# List all releases in the cluster
helm list

# Check release status and notes
helm status my-nginx

# See the rendered YAML that was applied
helm get manifest my-nginx

# See default values used
helm get values my-nginx
helm get values my-nginx --all   # includes defaults
```

One command replaced writing a Deployment, Service, ConfigMap, and ServiceAccount by hand.

---
<img width="1651" height="767" alt="image" src="https://github.com/user-attachments/assets/7ac10845-065e-4074-8ce6-f4bbb2ab1d05" />

<img width="1363" height="696" alt="image" src="https://github.com/user-attachments/assets/9f85bf06-3ad7-449f-af6a-6b1923134eb0" />

<img width="1042" height="732" alt="image" src="https://github.com/user-attachments/assets/069d731f-9840-4089-ae48-cd1b13554e7c" />

## Task 4: Customize with Values

```bash
# See all configurable values for a chart
helm show values bitnami/nginx

# Install with inline overrides
helm install nginx-custom bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort

# Nested values use dot notation
helm install nginx-custom bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort \
  --set resources.limits.cpu=250m
```

```yaml
# custom-values.yaml
replicaCount: 3

service:
  type: NodePort
  nodePorts:
    http: 30080

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi
```

```bash
# Install using values file
helm install nginx-file bitnami/nginx -f custom-values.yaml

# Combine file + inline (inline takes precedence)
helm install nginx-mixed bitnami/nginx -f custom-values.yaml --set replicaCount=5

# Verify overrides applied
helm get values nginx-file
```

### `--set` vs `-f values.yaml`

|  | `--set` | `-f values.yaml` |
| --- | --- | --- |
| Best for | Single quick overrides | Multiple values, complex nested config |
| Version control | Not tracked | Committed to git |
| Readability | Cluttered on CLI | Clean YAML |
| Production use | Not recommended | Preferred |

---

## Task 5: Upgrade and Rollback

```bash
# Upgrade to 5 replicas
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# Check revision history
helm history my-nginx
# REVISION   STATUS      DESCRIPTION
# 1          superseded  Install complete
# 2          deployed    Upgrade complete

# Rollback to revision 1
helm rollback my-nginx 1

# Check history again
helm history my-nginx
# REVISION   STATUS      DESCRIPTION
# 1          superseded  Install complete
# 2          superseded  Upgrade complete
# 3          deployed    Rollback to 1   <-- new revision!
```

**Rollback creates a new revision (3)**, it does not overwrite revision 2. This means you have a full audit trail of every change — same philosophy as `kubectl rollout history`.

---
<img width="1592" height="668" alt="image" src="https://github.com/user-attachments/assets/6c538aea-a93f-460e-b068-95efca8934d7" />
<img width="1286" height="773" alt="image" src="https://github.com/user-attachments/assets/c3947b54-f594-4b67-832e-d7aa7339f87f" />
<img width="1177" height="227" alt="image" src="https://github.com/user-attachments/assets/1a654ac1-e7a8-4a82-a3c7-bc99c598282d" />

## Task 6: Create Your Own Chart

```bash
# Scaffold a new chart
helm create my-app

# Explore the structure
tree my-app/
# my-app/
# ├── Chart.yaml          # chart metadata (name, version, appVersion)
# ├── values.yaml         # default values
# ├── charts/             # sub-charts (dependencies)
# ├── templates/
# │   ├── deployment.yaml   # uses Go templates
# │   ├── service.yaml
# │   ├── hpa.yaml
# │   ├── ingress.yaml
# │   ├── serviceaccount.yaml
# │   ├── _helpers.tpl      # reusable template functions
# │   └── NOTES.txt         # printed after install
# └── .helmignore
```

### Edit `values.yaml`

```yaml
# values.yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi
```

### Go Template Syntax (in `templates/`)

```yaml
# Inside templates/deployment.yaml
replicas: {{ .Values.replicaCount }}       # from values.yaml
name: {{ .Chart.Name }}                    # from Chart.yaml
name: {{ .Release.Name }}-{{ .Chart.Name }} # release-chart combo
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}

# Conditionals
{{- if .Values.autoscaling.enabled }}
...
{{- end }}

# Range (loop)
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value }}
{{- end }}
```

```bash
# Validate chart structure
helm lint my-app

# Preview rendered YAML without installing
helm template my-release ./my-app
helm template my-release ./my-app --set replicaCount=5

# Install from local chart
helm install my-release ./my-app

# Verify 3 replicas
kubectl get pods

# Upgrade to 5 replicas
helm upgrade my-release ./my-app --set replicaCount=5

# Verify 5 replicas
kubectl get pods
```

---
<img width="802" height="400" alt="image" src="https://github.com/user-attachments/assets/1fc8f632-98cb-4c85-9367-ad2782c655d5" />
<img width="908" height="702" alt="image" src="https://github.com/user-attachments/assets/3c0f7d39-2e98-4154-acd2-fdbb1b46f593" />

## Task 7: Clean Up

```bash
# Uninstall each release (removes all Kubernetes resources)
helm uninstall my-nginx
helm uninstall nginx-custom
helm uninstall nginx-file
helm uninstall my-release

# Keep history for auditing (resources deleted, history retained)
helm uninstall my-nginx --keep-history

# Verify no releases remain
helm list
helm list --all   # includes failed/uninstalled if --keep-history used

# Clean up local files
rm -rf my-app custom-values.yaml
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Chart | Packaged Kubernetes manifests with Go templates |
| Release | One installed instance of a chart; can have multiple releases per chart |
| `helm install` | Creates a new release |
| `helm upgrade` | Updates an existing release; creates a new revision |
| `helm rollback` | Rolls back to a previous revision; creates a new revision (doesn't overwrite) |
| `helm history` | Full audit trail of all revisions for a release |
| `--set` | Inline value override; not version-controlled |
| `-f values.yaml` | File-based overrides; production-preferred, committed to git |
| `helm template` | Renders manifests without installing; great for debugging |
| `helm lint` | Validates chart structure and syntax before install |
| `helm get manifest` | Shows the exact YAML that was applied to the cluster |

---

## Quick Reference

```bash
# Repository management
helm repo add <name> <url>
helm repo update
helm repo list
helm search repo <keyword>

# Release lifecycle
helm install <release> <chart>
helm install <release> <chart> --set key=val
helm install <release> <chart> -f values.yaml
helm upgrade <release> <chart>
helm rollback <release> <revision>
helm uninstall <release>
helm uninstall <release> --keep-history

# Inspection
helm list
helm status <release>
helm history <release>
helm get manifest <release>
helm get values <release>
helm get values <release> --all
helm show values <chart>

# Local chart development
helm create <chart-name>
helm lint <chart-dir>
helm template <release> <chart-dir>
helm install <release> <chart-dir>
helm upgrade <release> <chart-dir>
```
