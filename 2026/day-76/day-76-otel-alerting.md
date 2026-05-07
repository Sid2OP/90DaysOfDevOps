# Day 76 – OpenTelemetry and Alerting

## Overview

You have metrics (Prometheus) and logs (Loki). Today you add the third pillar — **traces** via OpenTelemetry — and set up **alerting** so your system notifies you when something goes wrong instead of you staring at dashboards all day.

---

## Task 1: Understand OpenTelemetry

### What is OpenTelemetry (OTEL)?

A vendor-neutral, open-source framework for generating, collecting, and exporting telemetry data (metrics, logs, traces). It is **not a backend** — it collects and ships data to backends like Prometheus, Jaeger, Loki, or Datadog. Think of it as a universal adapter between your app and any observability backend.

### The OTEL Collector Pipeline

```
[Your App / curl]
       |
       | OTLP (gRPC :4317 or HTTP :4318)
       v
  [OTEL Collector]
   Receivers       --> accept data (OTLP, Prometheus, Jaeger formats)
   Processors      --> transform (batch, filter, sample)
   Exporters       --> send to backends (Prometheus, debug, Jaeger)
       |
       |--metrics--> [Prometheus :8889] --> scraped by Prometheus
       |--traces --> [Debug console / Jaeger / Tempo]
       |--logs  --> [Debug console / Loki]
```

### What is OTLP?

OpenTelemetry Protocol — the standard wire format for sending telemetry.

- **gRPC:** port 4317 (binary, efficient, production)
- **HTTP/JSON:** port 4318 (human-readable, easy to test with curl)

### What are Distributed Traces?

A trace tracks a single request as it travels through multiple services. Each step = a **span**.

```
User Request
  └─ Span 1: API Gateway       (50ms)
       └─ Span 2: Auth Service    (20ms)
       └─ Span 3: Database query  (180ms)  <-- this is the bottleneck
```

Spans contain: trace ID, span ID, parent span ID, start time, duration, attributes.

---

## Task 2: Add the OpenTelemetry Collector

```bash
mkdir -p otel-collector
```

```yaml
# otel-collector/otel-collector-config.yml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"   # Prometheus scrapes this
  debug:
    verbosity: detailed         # prints traces/logs to stdout

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```

```yaml
# Add to docker-compose.yml
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus exporter endpoint
    volumes:
      - ./otel-collector/otel-collector-config.yml:/etc/otelcol-contrib/config.yaml
    restart: unless-stopped
```

```yaml
# Add to prometheus.yml scrape_configs
  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8889"]
```

```bash
docker compose up -d

# Verify collector is running
docker logs otel-collector 2>&1 | tail -5

# Prometheus Targets should show otel-collector as UP
```

---

## Task 3: Send Test Traces and Metrics

```bash
# Send a test trace via OTLP HTTP
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{"key": "service.name", "value": {"stringValue": "my-test-service"}}]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "5b8efff798038103d269b633813fc60c",
          "spanId": "eee19b7ec3c1b174",
          "name": "test-span",
          "kind": 1,
          "startTimeUnixNano": "1544712660000000000",
          "endTimeUnixNano": "1544712661000000000",
          "attributes": [
            {"key": "http.method", "value": {"stringValue": "GET"}},
            {"key": "http.status_code", "value": {"intValue": "200"}}
          ]
        }]
      }]
    }]
  }'

# See the trace in collector debug output
docker logs otel-collector 2>&1 | grep -A 10 "test-span"
```

```bash
# Send a test metric via OTLP HTTP
curl -X POST http://localhost:4318/v1/metrics \
  -H "Content-Type: application/json" \
  -d '{
    "resourceMetrics": [{
      "resource": {
        "attributes": [{"key": "service.name", "value": {"stringValue": "my-test-service"}}]
      },
      "scopeMetrics": [{
        "metrics": [{
          "name": "test_requests_total",
          "sum": {
            "dataPoints": [{"asInt": "42", "startTimeUnixNano": "1544712660000000000", "timeUnixNano": "1544712661000000000"}],
            "aggregationTemporality": 2,
            "isMonotonic": true
          }
        }]
      }]
    }]
  }'

# Query in Prometheus UI
test_requests_total
```

**Data flow:** `curl` → OTEL Collector (OTLP receiver) → Prometheus exporter → Prometheus scrapes → query in UI. This is how OTEL bridges different telemetry formats.

---

## Task 4: Prometheus Alerting Rules

```yaml
# alert-rules.yml
groups:
  - name: system-alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU above 80% for 2+ minutes. Current: {{ $value }}%"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Memory above 85%. Current: {{ $value }}%"

      - alert: ContainerDown
        expr: absent(container_last_seen{name="notes-app"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container is down"
          description: "notes-app has not been seen for over 1 minute"

      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Scrape target is down"
          description: "{{ $labels.job }} target {{ $labels.instance }} is unreachable"

      - alert: HighDiskUsage
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space critically low"
          description: "Root filesystem above 90%. Current: {{ $value }}%"
```

### Alert Rule Fields

| Field | Purpose |
| --- | --- |
| `expr` | PromQL condition that must be true to trigger |
| `for` | How long condition must hold before firing (avoids flapping) |
| `labels` | Metadata for routing (`severity: warning/critical`) |
| `annotations` | Human-readable summary and description; `{{ $value }}` inserts current metric value |

### Alert States

```
Inactive  -->  Pending  -->  Firing
  (condition   (condition     (condition true
  not met)      true but       for >= 'for'
                < 'for')       duration)
```

```yaml
# Update prometheus.yml to load rules
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/alert-rules.yml

scrape_configs:
  # ... existing scrape configs ...
```

```yaml
# Mount rules file in docker-compose.yml Prometheus service
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert-rules.yml:/etc/prometheus/alert-rules.yml
      - prometheus_data:/prometheus
```

```bash
docker compose up -d prometheus

# Check rules: Prometheus UI > Status > Rules
# Check alerts: Prometheus UI > Alerts

# Test ContainerDown alert
docker compose stop notes-app
# Wait 1-2 minutes, check Alerts UI -- TargetDown should fire
docker compose start notes-app
```

---

## Task 5: Grafana Alerts

### Create a Contact Point

1. Grafana → Alerting → Contact points → Add contact point
2. Name: "DevOps Team"
3. Integration: Email or Slack webhook
4. Save

### Create an Alert Rule

1. Alerting → Alert rules → New alert rule
2. Name: "High Container Memory"
3. Query: `container_memory_usage_bytes{name="notes-app"} / 1024 / 1024`
4. Condition: IS ABOVE 100 (MB)
5. Evaluation: every 1m, for 2m
6. Label: `severity = warning`
7. Contact point: "DevOps Team"
8. Save

### Create a Notification Policy

1. Alerting → Notification policies
2. Set default contact point: "DevOps Team"
3. Add nested policy: `severity=critical` → different contact or higher urgency settings

### Prometheus Alerts vs Grafana Alerts

|  | Prometheus Alerts | Grafana Alerts |
| --- | --- | --- |
| **Defined in** | `alert-rules.yml` files | Grafana UI or provisioning YAML |
| **Evaluates** | Only Prometheus metrics | Any datasource (Prometheus, Loki, etc.) |
| **Notifications** | Requires Alertmanager | Built-in — email, Slack, PagerDuty, etc. |
| **Best for** | Infrastructure alerts, GitOps workflows | Application alerts, easier setup for small teams |
| **Routing** | Alertmanager routes by labels | Notification policies route by labels |

**Rule of thumb:** Use Prometheus + Alertmanager for production GitOps workflows. Use Grafana alerts for quick setup, cross-datasource alerting, and smaller teams.

---

## Task 6: Full Stack Architecture

```
                    METRICS PIPELINE
[Node Exporter] -------> [Prometheus] -------> [Grafana Dashboards]
[cAdvisor] ------------> [Prometheus] -------> [Grafana Dashboards]
[OTEL Collector :8889] -> [Prometheus] -------> [Grafana Dashboards]
                                       -------> [Alert Rules -> Notifications]

                    LOGS PIPELINE
[Docker Containers] -> [Promtail] -> [Loki] -> [Grafana Explore]

                    TRACES PIPELINE
[App / curl OTLP] --> [OTEL Collector] -> [Debug / Jaeger / Tempo]
```

### Services Reference

| Service | Port | Purpose |
| --- | --- | --- |
| Prometheus | 9090 | Metrics storage and querying |
| Node Exporter | 9100 | Host system metrics |
| cAdvisor | 8080 | Container metrics |
| Grafana | 3000 | Visualization and alerting UI |
| Loki | 3100 | Log storage |
| Promtail | 9080 | Log collection agent |
| OTEL Collector | 4317/4318/8889 | Telemetry collection + Prometheus export |
| Notes App | 8000 | Sample application |

```bash
# Verify all 8 containers are healthy
docker compose ps
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| OTEL | Vendor-neutral telemetry framework; not a backend |
| OTEL Collector | Receives, processes, and exports telemetry to any backend |
| OTLP | Standard wire format; gRPC :4317 or HTTP :4318 |
| Trace | End-to-end journey of one request; made of spans |
| Span | One unit of work in a trace; has duration and attributes |
| `debug` exporter | Prints telemetry to stdout; use `docker logs otel-collector` to see |
| Alert `expr` | PromQL condition that triggers the alert |
| Alert `for` | Pending duration; prevents flapping on brief spikes |
| `absent()` | Fires when a time series disappears; detects dead containers |
| Alert states | Inactive → Pending → Firing |

---

## Quick Reference

```bash
# Start full stack
docker compose up -d
docker compose ps

# Test OTEL trace
curl -X POST http://localhost:4318/v1/traces -H "Content-Type: application/json" -d '{...}'
docker logs otel-collector 2>&1 | grep -A 10 "test-span"

# Test alert firing
docker compose stop notes-app
# Wait 1-2 min, check http://localhost:9090/alerts
docker compose start notes-app

# Reload Prometheus config (if --web.enable-lifecycle is set)
curl -X POST http://localhost:9090/-/reload
```

```yaml
# Alert rule skeleton
- alert: AlertName
  expr: <promql_condition>
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Short description"
    description: "Detailed message. Value: {{ $value }}"
```
<img width="1642" height="796" alt="image" src="https://github.com/user-attachments/assets/343825ba-cdf4-47b0-8179-11fba0a34d24" />

<img width="1655" height="763" alt="image" src="https://github.com/user-attachments/assets/35fa40b9-04d6-4206-b6df-ed8985d20f54" />
