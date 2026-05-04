# Day 74 – Node Exporter, cAdvisor, and Grafana Dashboards

## Overview

Prometheus is running but only monitoring itself. Today you add **Node Exporter** for host metrics (CPU, memory, disk), **cAdvisor** for container metrics, and **Grafana** to visualize everything in dashboards instead of raw PromQL.

---

## Task 1: Node Exporter — Host Metrics

Node Exporter exposes Linux system metrics in Prometheus format. It reads from `/proc` and `/sys` which the kernel uses to expose system state.

```yaml
# Add to docker-compose.yml
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro     # CPU stats, memory info, process data
      - /sys:/host/sys:ro       # hardware and driver details
      - /:/rootfs:ro            # filesystem usage
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped
```

```yaml
# prometheus.yml — add scrape target
  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

```bash
docker compose up -d

# Verify
curl http://localhost:9100/metrics | head -20
# Check Status > Targets in Prometheus UI
```

### Key Node Exporter Queries

```
# CPU idle percentage per core
node_cpu_seconds_total{mode="idle"}

# Memory: total vs available
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes

# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk filesystem usage percentage
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100

# Network bytes received per second
rate(node_network_receive_bytes_total[5m])
```

---

## Task 2: cAdvisor — Container Metrics

cAdvisor (Container Advisor) monitors resource usage of every running Docker container via Linux cgroups.

```yaml
# Add to docker-compose.yml
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro  # discover containers
      - /sys:/sys:ro                                   # cgroup stats
      - /var/lib/docker/:/var/lib/docker:ro            # container filesystem info
    restart: unless-stopped
```

```yaml
# prometheus.yml — add scrape target
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
```

```bash
docker compose up -d
open http://localhost:8080   # cAdvisor web UI
```

### Key cAdvisor Queries

```
# CPU usage per container
rate(container_cpu_usage_seconds_total{name!=""}[5m])

# Memory usage per container
container_memory_usage_bytes{name!=""}

# Network received per container
rate(container_network_receive_bytes_total{name!=""}[5m])

# Which container uses the most memory?
topk(3, container_memory_usage_bytes{name!=""})
```

`{name!=""}` filters out system-level aggregated entries and shows only named containers.

### Node Exporter vs cAdvisor

|  | Node Exporter | cAdvisor |
| --- | --- | --- |
| **Monitors** | The host machine (Linux OS) | Individual Docker containers |
| **Metrics prefix** | `node_` | `container_` |
| **Data source** | `/proc`, `/sys`, kernel interfaces | Docker socket, cgroups |
| **Use when** | You need host CPU/memory/disk/network | You need per-container resource usage |

---

## Task 3: Set Up Grafana

```yaml
# Add to docker-compose.yml
  grafana:
    image: grafana/grafana-enterprise:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

```bash
docker compose up -d
open http://localhost:3000
# Login: admin / admin123
```

**Add Prometheus datasource manually:**

1. Connections → Data Sources → Add data source
2. Select Prometheus
3. URL: `http://prometheus:9090` (container name, NOT [localhost](http://localhost))
4. Save & Test → "Successfully queried the Prometheus API"

> **Why container name, not [localhost](http://localhost)?** Containers communicate over Docker's internal network using service names as DNS. `localhost` inside the Grafana container refers to Grafana itself, not Prometheus.
> 

---

## Task 4: Build Your First Dashboard

Dashboards → New Dashboard → Add Visualization → Select Prometheus

**Panel 1 — CPU Usage (Gauge)**

```
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

- Visualization: Gauge
- Thresholds: green < 60, yellow < 80, red ≥ 80

**Panel 2 — Memory Usage (Gauge)**

```
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

- Visualization: Gauge

**Panel 3 — Container CPU Usage (Time Series)**

```
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
```

- Legend: `{{name}}`

**Panel 4 — Container Memory Usage (Bar Chart)**

```
container_memory_usage_bytes{name!=""} / 1024 / 1024
```

- Legend: `{{name}}`

**Panel 5 — Disk Usage (Stat)**

```
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```

Save as **"DevOps Observability Overview"**.

---

## Task 5: Auto-Provision Datasources with YAML

```bash
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards
```

```yaml
# grafana/provisioning/datasources/datasources.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

```bash
docker compose up -d grafana
# Prometheus datasource is already configured -- no manual UI steps
```

### Why Provisioning via YAML is Better Than Manual UI

|  | Manual UI | YAML Provisioning |
| --- | --- | --- |
| Reproducible | No — must redo after each rebuild | Yes — commit to Git, applies automatically |
| Team setup | Every teammate configures manually | `git clone`  • `docker compose up` = done |
| CI/CD compatible | No | Yes — no human interaction required |
| Disaster recovery | Reconfigure from memory | Restore from Git |

---

## Task 6: Import Community Dashboards

```
Dashboards > New > Import
```

| Dashboard ID | Name | What it shows |
| --- | --- | --- |
| **1860** | Node Exporter Full | Full host metrics: CPU, memory, disk, network, load average |
| **193** | Docker monitoring (cAdvisor) | Per-container CPU, memory, network, disk I/O |

Dashboard 1860 is the industry standard — almost every team imports it immediately after setting up Node Exporter.

---

## Complete docker-compose.yml

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped

  grafana:
    image: grafana/grafana-enterprise:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    restart: unless-stopped

  notes-app:
    image: trainwithshubham/notes-app:latest
    container_name: notes-app
    ports:
      - "8000:8000"
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

```bash
docker compose ps   # verify all 5 services are Up
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Node Exporter | Exposes host OS metrics via `/proc` and `/sys`; `node_` prefix |
| cAdvisor | Exposes per-container metrics via Docker socket + cgroups; `container_` prefix |
| `{name!=""}` filter | Removes cAdvisor system-level aggregates; shows only named containers |
| Grafana datasource | Always use container name (`prometheus:9090`), not `localhost` |
| YAML provisioning | Repeatable datasource/dashboard setup; commit to Git |
| Dashboard ID 1860 | Industry-standard Node Exporter Full dashboard |
| Dashboard ID 193 | Docker container monitoring via cAdvisor |

---

## Quick Reference

```bash
# Start/stop stack
docker compose up -d
docker compose ps
docker compose down

# Verify exporters
curl http://localhost:9100/metrics | head -20   # Node Exporter
curl http://localhost:8080/metrics | head -20   # cAdvisor

# UIs
open http://localhost:9090   # Prometheus
open http://localhost:8080   # cAdvisor
open http://localhost:3000   # Grafana
```

```
# CPU usage %
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage %
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk usage %
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Top 3 containers by memory
topk(3, container_memory_usage_bytes{name!=""})

# Container CPU %
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
```
<img width="1648" height="830" alt="image" src="https://github.com/user-attachments/assets/dc7baccf-cae7-48f7-be77-59905b9dd9c9" />

<img width="1652" height="838" alt="image" src="https://github.com/user-attachments/assets/c8203eb5-01fb-4712-a506-d6bb7d7363b1" />
