# Day 75 – Log Management with Loki and Promtail

## Overview

Metrics tell you *what* is broken. Logs tell you *why*. Today you add the second pillar of observability — **Grafana Loki** for log storage and **Promtail** as the log shipping agent. By the end, Grafana shows metrics and logs side by side.

---

## Task 1: Understand the Logging Pipeline

```
[Docker Containers]
       |
       | (write JSON logs to /var/lib/docker/containers/)
       v
  [Promtail]
       |
       | (reads logs, adds labels, pushes to Loki)
       v
    [Loki]
       |
       | (stores logs, indexes by labels only)
       v
   [Grafana]
       |
       | (queries Loki with LogQL, displays logs)
       v
   [You]
```

### Loki vs ELK Stack

|  | Loki | Elasticsearch (ELK) |
| --- | --- | --- |
| **Indexes** | Labels only (container name, job) | Full text of every log line |
| **Storage cost** | Low — only metadata indexed | High — inverted index of all words |
| **Query power** | Label filtering + grep-style | Full-text search, field extraction |
| **Complexity** | Simple — "Prometheus for logs" | Complex — cluster management required |
| **Best for** | DevOps log aggregation, cost-sensitive | Compliance, audit, complex search |

**The trade-off:** Loki can't search arbitrary fields efficiently. You must filter by label first, then grep the content. This makes it fast and cheap but less powerful than Elasticsearch for complex queries.

---

## Task 2: Add Loki to the Stack

```bash
mkdir -p loki
```

```yaml
# loki/loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /loki

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  filesystem:
    directory: /loki/chunks
```

### What This Config Does

| Field | Purpose |
| --- | --- |
| `auth_enabled: false` | Single-tenant mode, no auth needed |
| `store: tsdb` | Uses Loki's time-series database for the index |
| `object_store: filesystem` | Stores log chunks on local disk |
| `replication_factor: 1` | Single instance, no HA (fine for learning) |
| `period: 24h` | Creates a new index every 24 hours |

```yaml
# Add to docker-compose.yml
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml
      - loki_data:/loki
    command: -config.file=/etc/loki/loki-config.yml
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  loki_data:       # add this
```

```bash
docker compose up -d loki

# Verify Loki is ready
curl http://localhost:3100/ready
# ready
```

---

## Task 3: Add Promtail to Collect Container Logs

```bash
mkdir -p promtail
```

```yaml
# promtail/promtail-config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml    # bookmark -- tracks what was already sent

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*-json.log
    pipeline_stages:
      - docker: {}    # parses Docker JSON log format, extracts timestamp + stream
```

```yaml
# Add to docker-compose.yml
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    ports:
      - "9080:9080"
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/promtail-config.yml
    restart: unless-stopped
```

### Why These Volume Mounts?

| Mount | Purpose |
| --- | --- |
| `/var/lib/docker/containers` (read-only) | Where Docker writes JSON log files for each container |
| `/var/run/docker.sock` | Lets Promtail discover container names and labels via Docker API |

```bash
docker compose up -d

# Generate some logs
for i in $(seq 1 20); do curl -s http://localhost:8000 > /dev/null; done

# Check Promtail targets
curl http://localhost:9080/targets
```

---

## Task 4: Add Loki as a Grafana Datasource

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

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```

```bash
docker compose restart grafana
```

Grafana → Connections → Data Sources — you should see both **Prometheus** and **Loki** listed.

---

## Task 5: Query Logs with LogQL

Grafana → Explore (compass icon) → Select **Loki** as datasource.

```
# Stream selector -- all Docker container logs
{job="docker"}

# Filter by container name
{container_name="prometheus"}

# Filter by content -- lines containing "error"
{job="docker"} |= "error"

# Exclude lines -- filter out health check noise
{job="docker"} != "health"

# Regex filter -- HTTP 4xx or 5xx
{job="docker"} |~ "status=[45]\\d{2}"

# Case-insensitive search
{job="docker"} |~ "(?i)error"

# Count log lines over time (5 min windows)
count_over_time({job="docker"}[5m])

# Rate of logs per second
rate({job="docker"}[5m])

# Top containers by log volume
topk(5, sum by (container_name) (rate({job="docker"}[5m])))
```

### Exercise Answers

```
# Error logs from notes-app in last 1 hour
{container_name="notes-app"} |= "error"

# Count of error lines per minute
sum(count_over_time({container_name="notes-app"} |= "error" [1m]))
```

### LogQL Operators

| Operator | Meaning |  |
| --- | --- | --- |
| `\ | =` | Line contains (case-sensitive) |
| `!=` | Line does not contain |  |
| `\ | ~` | Line matches regex |
| `!~` | Line does not match regex |  |
| `\ | json` | Parse line as JSON, extract fields |
| `\ | logfmt` | Parse key=value logfmt format |

---

## Task 6: Correlate Metrics and Logs in Grafana

### Add a Logs Panel to Your Dashboard

1. Open your existing dashboard from Day 74
2. Add Panel → Select **Loki** datasource
3. Query: `{job="docker"}`
4. Visualization: **Logs**
5. Title: "Container Logs"

### Use the Split View for Real-Time Correlation

1. Go to **Explore**
2. Click the **Split** button (two panels side by side)
3. Left panel (Prometheus): `rate(container_cpu_usage_seconds_total{name="notes-app"}[5m])`
4. Right panel (Loki): `{container_name="notes-app"}`

Click on a CPU spike in the left panel — both panels automatically zoom to that time range. You see the metric anomaly and the exact log lines from that moment simultaneously.

### Why This Matters for Incident Response

| Separate systems | Grafana with Prometheus + Loki |
| --- | --- |
| Switch between browser tabs | Single UI with split view |
| Manually align time ranges | Time sync is automatic |
| Context lost between tools | Click a metric spike, see logs instantly |
| Slower MTTR | Faster root cause identification |

MTTR (Mean Time to Resolve) is the key metric for incident response. Having metrics and logs in one place with synchronized time ranges cuts MTTR significantly — you don't waste time copying timestamps between tools.

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Loki | Log aggregation system; indexes labels only, not full text |
| Promtail | Log shipping agent; reads Docker log files and pushes to Loki |
| `positions.yaml` | Promtail's bookmark; tracks what's been shipped; delete = re-read all logs |
| LogQL stream selector | `{label="value"}` — required first step in every query |
| `\ | =` filter |
| `count_over_time` | Count log lines in a time window (metric from logs) |
| `rate()` in LogQL | Rate of log lines per second |
| Low cardinality labels | Use container name, job — never user ID or request ID as labels |
| Provisioning | YAML-based Grafana datasource config; applied on restart |

---

## Quick Reference

```bash
# Start stack
docker compose up -d

# Verify Loki
curl http://localhost:3100/ready
curl http://localhost:3100/metrics

# Check Promtail targets
curl http://localhost:9080/targets

# Generate logs
for i in $(seq 1 20); do curl -s http://localhost:8000 > /dev/null; done

# Restart Grafana to pick up new datasource
docker compose restart grafana
```

```
# LogQL quick reference
{job="docker"}                              # all docker logs
{container_name="myapp"}                    # specific container
{job="docker"} |= "error"                  # contains 'error'
{job="docker"} != "debug"                  # excludes 'debug'
{job="docker"} |~ "(?i)exception"          # regex, case-insensitive
rate({job="docker"}[5m])                    # log rate per second
count_over_time({job="docker"}[5m])        # log count per window
sum by (container_name) (rate({job="docker"}[5m]))  # by container
```
<img width="1658" height="831" alt="image" src="https://github.com/user-attachments/assets/1cb5f9d9-9d39-4664-8db2-96d8e85631b0" />

<img width="1655" height="842" alt="image" src="https://github.com/user-attachments/assets/3e749ecb-da2f-4275-9aa0-0f3cd091047e" />
