# Day 60 – Capstone: Deploy WordPress + MySQL on Kubernetes

## Overview

Ten days of Kubernetes — clusters, Pods, Deployments, Services, Secrets, ConfigMaps, PVCs, StatefulSets, resource management, autoscaling, and Helm. Today you put it all together in one real deployment: **WordPress + MySQL** using every major concept you have learned.

---

## Concepts Used in This Capstone

| Concept | Day | Where Used |
| --- | --- | --- |
| Namespace | 52 | `capstone` namespace isolates all resources |
| Secret | 54 | MySQL credentials |
| ConfigMap | 54 | WordPress DB connection config |
| StatefulSet | 56 | MySQL with stable identity |
| Headless Service | 56 | Per-pod DNS for MySQL |
| PVC (volumeClaimTemplates) | 55 | MySQL data persistence |
| Deployment | 52 | WordPress with 2 replicas |
| `envFrom` / `secretKeyRef` | 54 | Inject config into pods |
| Resource requests + limits | 57 | Both MySQL and WordPress |
| Liveness + readiness probes | 57 | WordPress health checks |
| NodePort Service | 53 | External access to WordPress |
| HPA | 58 | Auto-scale WordPress under load |

---

## Architecture

```
[Browser]
    |
    | :30080
    v
[NodePort Service: wordpress]
    |
    v
[WordPress Deployment: 2 replicas]
    |  envFrom: ConfigMap (DB_HOST, DB_NAME)
    |  secretKeyRef: Secret (DB_USER, DB_PASS)
    |  liveness + readiness probes
    |
    | mysql-0.mysql.capstone.svc.cluster.local:3306
    v
[Headless Service: mysql]
    |
    v
[StatefulSet: mysql-0]
    |  envFrom: Secret
    |  requests: 250m CPU / 512Mi RAM
    |
    v
[PVC: mysql-data-mysql-0 (1Gi)]
```

---

## Task 1: Create the Namespace

```bash
kubectl create namespace capstone

# Set as default so you don't need -n capstone on every command
kubectl config set-context --current --namespace=capstone

# Verify
kubectl config view --minify | grep namespace
```

---

## Task 2: Deploy MySQL

### Secret

```yaml
# mysql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: capstone
stringData:
  MYSQL_ROOT_PASSWORD: rootpassword123
  MYSQL_DATABASE: wordpress
  MYSQL_USER: wpuser
  MYSQL_PASSWORD: wppassword123
```

### Headless Service

```yaml
# mysql-headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: capstone
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

### StatefulSet

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: capstone
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        envFrom:
        - secretRef:
            name: mysql-secret
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "localhost"]
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          exec:
            command: ["mysql", "-u", "root", "-prootpassword123", "-e", "SELECT 1"]
          initialDelaySeconds: 30
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 1Gi
```

```bash
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-headless-svc.yaml
kubectl apply -f mysql-statefulset.yaml

# Watch MySQL start up
kubectl get pods -w

# Verify database exists
kubectl exec -it mysql-0 -- mysql -u wpuser -pwppassword123 -e "SHOW DATABASES;"
# Should show: wordpress
```

---

## Task 3: Deploy WordPress

### ConfigMap

```yaml
# wordpress-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-config
  namespace: capstone
data:
  WORDPRESS_DB_HOST: "mysql-0.mysql.capstone.svc.cluster.local:3306"
  WORDPRESS_DB_NAME: "wordpress"
```

> The host must follow the StatefulSet DNS pattern: `<pod>.<headless-svc>.<namespace>.svc.cluster.local`
> 

### Deployment

```yaml
# wordpress-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: capstone
  labels:
    app: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: wordpress-config
        env:
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /wp-login.php
            port: 80
          initialDelaySeconds: 60
          periodSeconds: 15
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /wp-login.php
            port: 80
          initialDelaySeconds: 60
          periodSeconds: 10
          failureThreshold: 3
```

```bash
kubectl apply -f wordpress-configmap.yaml
kubectl apply -f wordpress-deployment.yaml

# Wait until both pods are 1/1
kubectl get pods -w
# wordpress-xxxxx   0/1   Running   (waiting for MySQL + WP boot)
# wordpress-xxxxx   1/1   Running
```

> WordPress takes ~60 seconds to initialize on first boot. The `initialDelaySeconds: 60` gives it time before probes start.
> 

---

## Task 4: Expose WordPress

```yaml
# wordpress-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: capstone
spec:
  type: NodePort
  selector:
    app: wordpress
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
kubectl apply -f wordpress-service.yaml

# Minikube
minikube service wordpress -n capstone

# Kind
kubectl port-forward svc/wordpress 8080:80 -n capstone
# Then open: http://localhost:8080

# Docker Desktop
curl http://localhost:30080
```

Complete the WordPress setup wizard and create a blog post. You will verify this post survives pod deletion in Task 5.

---

## Task 5: Test Self-Healing and Persistence

```bash
# Test 1: WordPress pod self-healing
kubectl get pods
kubectl delete pod <wordpress-pod-name>

# Watch it come back immediately
kubectl get pods -w
# Deployment controller recreates it within seconds
# Refresh browser -- site is still up (other replica handles traffic)

# Test 2: MySQL pod deletion (StatefulSet)
kubectl delete pod mysql-0

kubectl get pods -w
# mysql-0 terminates, then comes back with the SAME name
# Reconnects to the SAME PVC (mysql-data-mysql-0)

# After MySQL recovers (~30s), refresh WordPress
# Your blog post should still be there
kubectl exec -it mysql-0 -- mysql -u wpuser -pwppassword123 wordpress -e "SELECT post_title FROM wp_posts WHERE post_status='publish';"
```

**What this proves:**

- Deployment: stateless pods are recreated instantly, traffic continues via the second replica
- StatefulSet: `mysql-0` always reconnects to `mysql-data-mysql-0` — data is never lost

---

## Task 6: Set Up HPA

```yaml
# wordpress-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress-hpa
  namespace: capstone
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

```bash
kubectl apply -f wordpress-hpa.yaml

kubectl get hpa -n capstone
# NAME            REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS
# wordpress-hpa   Deployment/wordpress   3%/50%    2         10        2

# Full picture
kubectl get all -n capstone
```

---

## Task 7: Compare with Helm (Bonus)

```bash
# Deploy WordPress via Helm in a separate namespace
kubectl create namespace helm-wp
helm install wp-helm bitnami/wordpress -n helm-wp

# Count what Helm created
kubectl get all -n helm-wp

# Compare
helm get manifest wp-helm -n helm-wp | grep "^kind:" | sort | uniq -c

# Clean up
helm uninstall wp-helm -n helm-wp
kubectl delete namespace helm-wp
```

### Manual vs Helm Comparison

|  | Manual (this capstone) | Helm (bitnami/wordpress) |
| --- | --- | --- |
| Resources created | You control every field | Auto-generated from values |
| Customization | Complete control | Via `values.yaml` overrides |
| Learning value | High — you understand every line | Lower — abstracted away |
| Production speed | Slow (write everything) | Fast (one command) |
| Database | MySQL 8.0 | MariaDB (MySQL-compatible) |

---

## Task 8: Clean Up and Reflect

```bash
# Final full view
kubectl get all -n capstone
kubectl get pvc -n capstone
kubectl get secrets -n capstone
kubectl get configmaps -n capstone
kubectl get hpa -n capstone

# Delete EVERYTHING in one command
kubectl delete namespace capstone

# Verify it is all gone
kubectl get namespaces
kubectl get pods -A

# Reset default namespace
kubectl config set-context --current --namespace=default
```

**Yes — deleting the namespace removes everything inside it**: Pods, Deployments, StatefulSets, Services, ConfigMaps, Secrets, HPAs, and PVCs. This is the nuclear option. Use with extreme caution in production.

---

## Troubleshooting

| Symptom | Check |
| --- | --- |
| MySQL takes long to start | `kubectl logs mysql-0 -n capstone` |
| WordPress pods `0/1` | `kubectl describe pod <name> -n capstone` — check probe events |
| PVC stays `Pending` | `kubectl get storageclass` — ensure a default exists |
| Can't reach WordPress | Verify `nodePort: 30080` is in 30000-32767 range |
| HPA shows `<unknown>` | Metrics Server not installed or no CPU requests set |
| WordPress can't reach MySQL | Check `WORDPRESS_DB_HOST` matches exact StatefulSet DNS pattern |

---

## Key Concepts Summary

This single deployment used **12 Kubernetes concepts**:

1. **Namespace** — isolated environment for all capstone resources
2. **Secret** — MySQL credentials stored encrypted, injected via `envFrom`
3. **ConfigMap** — WordPress DB host and name as non-sensitive config
4. **StatefulSet** — MySQL with stable pod name (`mysql-0`)
5. **Headless Service** — per-pod DNS for StatefulSet (`mysql-0.mysql.capstone.svc.cluster.local`)
6. **PVC via volumeClaimTemplates** — 1Gi dedicated storage for MySQL that survives pod deletion
7. **Deployment** — WordPress with 2 replicas and self-healing
8. **envFrom + secretKeyRef** — clean separation of config from container spec
9. **Resource requests + limits** — scheduler placement and runtime enforcement
10. **Liveness + readiness probes** — automatic health checking on `/wp-login.php`
11. **NodePort Service** — external access on port 30080
12. **HPA** — auto-scale WordPress from 2 to 10 replicas based on CPU

---

## Quick Reference — All Files

```
capstone/
├── mysql-secret.yaml
├── mysql-headless-svc.yaml
├── mysql-statefulset.yaml
├── wordpress-configmap.yaml
├── wordpress-deployment.yaml
├── wordpress-service.yaml
└── wordpress-hpa.yaml

Apply order:
1. mysql-secret.yaml          (Secret first -- others depend on it)
2. mysql-headless-svc.yaml    (Service before StatefulSet)
3. mysql-statefulset.yaml     (MySQL must be Running before WordPress)
4. wordpress-configmap.yaml
5. wordpress-deployment.yaml
6. wordpress-service.yaml
7. wordpress-hpa.yaml
```
<img width="1612" height="662" alt="image" src="https://github.com/user-attachments/assets/1e6438ad-8a68-4cbe-af5a-6f1bcaf07cc5" />
