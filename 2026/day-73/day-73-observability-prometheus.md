# Day 73 – Introduction to Observability and Prometheus

## Overview

You have built infrastructure with Terraform, configured servers with Ansible, and containerized apps with Docker. But once everything is running — how do you know it is healthy? How do you find out why something broke at 3 AM? That is **observability**.

---

## Task 1: Understand Observability

### Monitoring vs Observability

|  | Traditional Monitoring | Observability |
| --- | --- | --- |
| **Tells you** | *What* is broken (alert fired) | *Why* it is broken (explore and correlate) |
| **Approach** | Pre-defined thresholds and dashboards | Ask arbitrary questions about system state |
| **Limitation** | Only catches known failure modes | Handles unknown unknowns |

### The Three Pillars

| Pillar | What it is | Answers | Tools |
| --- | --- | --- | --- |
| **Metrics** | Numerical measurements over time | Is something wrong? How bad? | Prometheus, Datadog, CloudWatch |
| **Logs** | Timestamped text records of events | Why did it break? What happened? | Loki, ELK Stack, Fluentd |
| **Traces** | The journey of a request across services | Where did it break? Which service? | OpenTelemetry, Jaeger, Zipkin |

**Why you need all three:**

- Metrics tell you the error rate on `/api/users` spiked
- Logs tell you the stack trace showing a database timeout
- Traces tell you the payment service call took 12 seconds

### Architecture (What You Will Build This Week)

```
[Your App]  --> metrics --> [Prometheus]        --> [Grafana Dashboards]
[Your App]  --> logs    --> [Promtail]   --> [Loki]  --> [Grafana]
[Your App]  --> traces  --> [OTEL Collector]    --> [Grafana/Debug]
[Host]      --> metrics --> [Node Exporter]     --> [Prometheus]
[Docker]    --> metrics --> [cAdvisor]          --> [Prometheus]
```

---

## Task 2: Set Up Prometheus with Docker

```bash
mkdir observability-stack && cd observability-stack
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s       # pull metrics every 15 seconds
  evaluation_interval: 15s   # evaluate alerting rules every 15 seconds

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

```yaml
# docker-compose.yml
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

volumes:
  prometheus_data:
```

```bash
docker compose up -d
```

Open [**http://localhost:9090**](http://localhost:9090) → Status → Targets. You should see one target (`prometheus`) with state **UP**.

---

## Task 3: Prometheus Concepts

### Metric Types

| Type | Behavior | Real-world example |
| --- | --- | --- |
| **Counter** | Only goes up (resets on restart) | `http_requests_total`, `errors_total` |
| **Gauge** | Goes up and down freely | `cpu_usage_percent`, `memory_bytes`, `active_connections` |
| **Histogram** | Counts values in pre-defined buckets | Request duration: how many took <100ms, <500ms, <1s |
| **Summary** | Like histogram but percentiles calculated client-side | `request_duration_seconds{quantile="0.99"}` |

**Counter vs Gauge:**

- **Counter** — `http_requests_total` starts at 0 and only increments. To get a useful rate, use `rate()`. Never use rate() on a gauge.
- **Gauge** — `node_memory_MemAvailable_bytes` floats freely. Query it directly.

### Labels and Time Series

Labels add dimensions to metrics:

```
http_requests_total{method="GET", status="200", handler="/api"} 1523
http_requests_total{method="POST", status="500", handler="/api"} 12
```

Each unique combination of `metric name + labels` = one **time series**.

### Explore in the Prometheus UI

```
# How many metrics is Prometheus collecting about itself?
count({__name__=~".+"})

# Memory Prometheus is using
process_resident_memory_bytes

# Total HTTP requests to Prometheus
prometheus_http_requests_total

# Filter by handler
prometheus_http_requests_total{handler="/api/v1/query"}
```

---

## Task 4: PromQL Basics

```
# Instant vector -- current value
up
# Returns 1 (healthy) or 0 (down) for each scrape target

# Range vector -- values over a time window
prometheus_http_requests_total[5m]
# All values from the last 5 minutes

# Rate -- per-second speed of a counter
rate(prometheus_http_requests_total[5m])
# Most common function -- converts always-increasing counters to useful rates

# Sum across all label combinations
sum(rate(prometheus_http_requests_total[5m]))

# Filter by label
prometheus_http_requests_total{code="200"}
prometheus_http_requests_total{code!="200"}

# Arithmetic
process_resident_memory_bytes / 1024 / 1024
# Converts bytes to megabytes

# Top-K
topk(5, prometheus_http_requests_total)

# Exercise answer: per-second rate of non-200 requests over 5 minutes
rate(prometheus_http_requests_total{code!="200"}[5m])
```

### PromQL Rules

| Rule | Why |
| --- | --- |
| Always use `rate()` before `sum()` | `sum(rate(...))` is correct. `rate(sum(...))` is wrong and gives bad results |
| `rate()` only on counters | Applying rate to a gauge gives meaningless results |
| Use `[5m]` or longer windows | Shorter windows are noisy; 5m is the standard minimum |
| `irate()` for spikes | `irate()` uses last 2 samples -- more responsive but noisier |

---

## Task 5: Add a Sample Application as a Scrape Target

```yaml
# docker-compose.yml
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

  notes-app:
    image: trainwithshubham/notes-app:latest
    container_name: notes-app
    ports:
      - "8000:8000"
    restart: unless-stopped

  cadviser:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
    - 8080:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

volumes:
  prometheus_data:

```

```yaml
 prometheus.yml
global:
  scrape_interval: 15s       # pull metrics every 15 seconds
  evaluation_interval: 15s   # evaluate alerting rules every 15 seconds

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "notes-app"
    static_configs:
      - targets: ["cadviser:8080"]

```

```bash
docker compose up -d

# Generate some traffic
curl http://localhost:8000
curl http://localhost:8000
curl http://localhost:8000
```
<img width="1652" height="792" alt="image" src="https://github.com/user-attachments/assets/44c68f84-586f-4a2f-9e57-3edf47d484a5" />

Status → Targets should now show **two targets** both in state UP.

### Why Targets Show as DOWN

| Cause | Fix |
| --- | --- |
| Container not running | `docker ps` — check if it started |
| Wrong port | Verify `targets` in prometheus.yml matches exposed port |
| Different Docker network | Services must be on the same network; Docker Compose handles this automatically |
| App doesn't expose `/metrics` | Needs an exporter (Node Exporter, cAdvisor, etc.) |

---
## Task 6: Data Retention and Storage

```bash
# Check disk usage
docker exec prometheus du -sh /prometheus

# Check TSDB status
# UI: Status > TSDB Status
```

```yaml
# Configure retention in docker-compose.yml
command:
  - '--config.file=/etc/prometheus/prometheus.yml'
  - '--storage.tsdb.retention.time=30d'    # keep 30 days of data
  - '--storage.tsdb.retention.size=1GB'    # or limit by size
```

**What happens when retention is exceeded:** Prometheus deletes the oldest data blocks automatically to stay within the limit. It runs compaction in the background — no manual intervention needed.

**Why the volume mount matters:** Without `prometheus_data:/prometheus`, all collected metrics are stored inside the container. When the container restarts or is recreated, all historical data is lost. The named volume persists data across container restarts, upgrades, and re-creates.

### Prometheus Storage Architecture

```
/prometheus/
  chunks_head/    # recent in-memory data being written
  01HXY.../       # 2-hour data blocks, compacted
  01HXZ.../       # older blocks
  wal/            # write-ahead log (crash recovery)
  lock            # prevents multiple Prometheus instances writing simultaneously
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Observability vs monitoring | Monitoring = what; Observability = why |
| Three pillars | Metrics (numbers), Logs (text), Traces (request flow) |
| Pull model | Prometheus scrapes targets on its schedule; targets don't push |
| `up` metric | Auto-created for every target; 1 = healthy, 0 = unreachable |
| Counter | Only goes up; use `rate()` to get meaningful values |
| Gauge | Goes up and down; query directly |
| Labels | Dimensions on metrics; each unique combo = one time series |
| `rate()` | Converts counter to per-second speed; always use before `sum()` |
| Retention | Default 15 days; configurable by time or size |
| Volume mount | Required for data persistence across restarts |

---

## Quick Reference

```bash
# Start stack
docker compose up -d
docker compose down

# Check targets
open http://localhost:9090/targets

# Reload config without restart (requires --web.enable-lifecycle flag)
curl -X POST http://localhost:9090/-/reload

# Check Prometheus logs
docker logs prometheus

# Disk usage
docker exec prometheus du -sh /prometheus
```

```
# Essential PromQL patterns
up                                              # target health
rate(metric_total[5m])                          # per-second rate
sum(rate(metric_total[5m])) by (label)          # rate aggregated by label
metric{label="value"}                           # filter by label
metric / 1024 / 1024                            # bytes to MB
topk(5, metric)                                 # top 5 values
count({__name__=~".+"})                         # total metric count
```
