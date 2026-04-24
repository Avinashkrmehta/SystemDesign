# Module 30: Monitoring, Logging & Observability

> You can't fix what you can't see. In a distributed system with dozens of services, understanding what's happening — why a request is slow, where errors originate, which service is the bottleneck — requires **observability**. This module covers the three pillars of observability (metrics, logs, traces), the tools and patterns for implementing each, and the operational practices (SLIs, SLOs, error budgets) that turn raw data into actionable insights.

---

## 30.1 Three Pillars of Observability

> **Observability** is the ability to understand the internal state of a system by examining its external outputs. The three pillars — **metrics**, **logs**, and **traces** — each provide a different lens into system behavior. Together, they give you a complete picture.

---

### Metrics, Logs, Traces — Overview

```
The Three Pillars:

  METRICS                    LOGS                      TRACES
  (What is happening?)       (What happened?)          (How did it happen?)
  
  "Request rate: 500/sec"    "Error: DB timeout at     "Request abc123:
   "P99 latency: 200ms"      10:30:05 for user 456"    API Gateway → 5ms
   "Error rate: 0.5%"        "User 789 logged in       Order Service → 50ms
   "CPU: 75%"                 from IP 203.0.113.50"      → Payment Service → 200ms
   "Queue depth: 1500"       "Order 123 created,         → DB Query → 150ms
                               total: $99.99"            Total: 255ms"
  
  Aggregated numbers         Individual events          Request flow across
  over time                  with context               services
  
  "How much?"                "What exactly?"            "Where is the bottleneck?"
```

**How They Work Together:**

```
Alert fires: "P99 latency > 500ms" (METRIC)
  → Check dashboard: Order Service latency spiked at 10:30 (METRIC)
  → Search logs: "timeout" errors in Order Service at 10:30 (LOG)
  → Find trace: Request abc123 took 2 seconds (TRACE)
  → Trace shows: Payment Service call took 1.8 seconds (TRACE)
  → Check Payment Service logs: "Connection pool exhausted" (LOG)
  → Check Payment Service metrics: Active connections = 100/100 (METRIC)
  → Root cause: Payment Service connection pool is too small
```

| Pillar | What It Tells You | Granularity | Volume | Retention |
|--------|------------------|-------------|--------|-----------|
| **Metrics** | Aggregated system health (rates, counts, distributions) | Low (aggregated) | Low | Long (months/years) |
| **Logs** | Detailed event records with context | High (per event) | Very high | Medium (weeks/months) |
| **Traces** | Request flow across services with timing | Medium (per request) | High (sampled) | Short (days/weeks) |

---

### When to Use Each Pillar

| Question | Use |
|----------|-----|
| "Is the system healthy right now?" | Metrics (dashboards) |
| "What's the error rate over the last hour?" | Metrics (counters) |
| "Why did this specific request fail?" | Logs (search by request ID) |
| "What happened at 10:30 that caused the spike?" | Logs (time-range search) |
| "Which service is causing the latency?" | Traces (waterfall view) |
| "Is the database the bottleneck?" | Traces (span timing) + Metrics (DB latency) |
| "How many unique users hit this error?" | Logs (aggregate by user ID) |
| "What's the P99 latency trend this week?" | Metrics (histogram over time) |

---


## 30.2 Metrics & Monitoring

> **Metrics** are numeric measurements collected over time. They tell you the **aggregate health** of your system — request rates, error rates, latencies, resource utilization. Metrics are the foundation of dashboards, alerts, and capacity planning.

---

### Metric Types

**Counter:**
A monotonically increasing value that only goes up (or resets to zero on restart). Used for counting events.

```
Examples:
  http_requests_total = 1,547,832
  errors_total = 2,341
  orders_created_total = 89,456
  bytes_transferred_total = 1,234,567,890

Usage:
  Rate: requests_per_second = rate(http_requests_total[5m])
  → "How many requests per second over the last 5 minutes?"
  
  Error rate: errors_total / http_requests_total
  → "What percentage of requests are errors?"
```

**Gauge:**
A value that can go up or down. Represents a current state or snapshot.

```
Examples:
  cpu_usage_percent = 73.5
  memory_used_bytes = 8,589,934,592
  active_connections = 1,247
  queue_depth = 342
  temperature_celsius = 42.1

Usage:
  "What is the current CPU usage?"
  "How many items are in the queue right now?"
```

**Histogram:**
Measures the distribution of values by counting observations in configurable buckets. Used for latencies and sizes.

```
Example: HTTP request latency histogram

  Buckets:
    ≤ 10ms:   5,000 requests
    ≤ 50ms:   15,000 requests
    ≤ 100ms:  18,000 requests
    ≤ 250ms:  19,500 requests
    ≤ 500ms:  19,800 requests
    ≤ 1000ms: 19,950 requests
    ≤ +Inf:   20,000 requests (total)

  From this, you can calculate:
    P50 (median): ~30ms (50% of requests ≤ 30ms)
    P95: ~200ms (95% of requests ≤ 200ms)
    P99: ~800ms (99% of requests ≤ 800ms)
    Average: sum / count
```

**Summary:**
Similar to histogram but calculates percentiles on the client side (pre-computed). Less flexible than histograms but more accurate for specific percentiles.

```
Example: HTTP request latency summary
  P50: 28ms
  P90: 150ms
  P95: 210ms
  P99: 780ms
  Count: 20,000
  Sum: 1,200,000ms
```

**Histogram vs Summary:**

| Feature | Histogram | Summary |
|---------|-----------|---------|
| Percentile calculation | Server-side (approximate, from buckets) | Client-side (exact, pre-computed) |
| Aggregation across instances | ✅ Yes (merge bucket counts) | ❌ No (can't merge pre-computed percentiles) |
| Configuration | Define bucket boundaries | Define target percentiles |
| Flexibility | Can calculate any percentile after the fact | Only pre-defined percentiles |
| Recommendation | ✅ Preferred (aggregatable) | Use when exact percentiles are critical |

---

### Prometheus + Grafana

**Prometheus** is the de facto standard for metrics collection in cloud-native systems. **Grafana** is the standard for visualization and dashboarding.

```
Prometheus Architecture:

  +------------------+     Scrape (pull)     +------------------+
  | App Server 1     | ←──────────────────── | Prometheus       |
  | /metrics endpoint|                       | Server           |
  +------------------+                       |                  |
  +------------------+     Scrape            | - Time-series DB |
  | App Server 2     | ←──────────────────── | - PromQL queries |
  | /metrics endpoint|                       | - Alert rules    |
  +------------------+                       +--------+---------+
  +------------------+     Scrape                     |
  | Redis            | ←──────────────────── ─────────+
  | (via exporter)   |                       |
  +------------------+                       |
                                    +--------v---------+
                                    | Grafana          |
                                    | - Dashboards     |
                                    | - Visualizations |
                                    +------------------+
                                    +------------------+
                                    | Alertmanager     |
                                    | - Route alerts   |
                                    | - PagerDuty/Slack|
                                    +------------------+
```

**How Prometheus Works:**
1. Applications expose metrics at a `/metrics` HTTP endpoint (in Prometheus text format)
2. Prometheus **scrapes** (pulls) metrics from all targets at a configured interval (typically 15-30 seconds)
3. Metrics are stored in Prometheus's built-in **time-series database**
4. Users query metrics using **PromQL** (Prometheus Query Language)
5. Grafana connects to Prometheus and visualizes metrics in dashboards
6. **Alertmanager** evaluates alert rules and sends notifications (PagerDuty, Slack, email)

**PromQL Examples:**

```promql
# Request rate (requests per second over the last 5 minutes)
rate(http_requests_total[5m])

# Error rate (percentage)
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100

# P99 latency (from histogram)
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# CPU usage per instance
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Top 5 endpoints by request rate
topk(5, sum by (endpoint) (rate(http_requests_total[5m])))

# Memory usage percentage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
```

**Prometheus Metrics Endpoint Example:**

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/users",status="200"} 15234
http_requests_total{method="GET",endpoint="/api/users",status="500"} 23
http_requests_total{method="POST",endpoint="/api/orders",status="201"} 8921

# HELP http_request_duration_seconds HTTP request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{endpoint="/api/users",le="0.01"} 5000
http_request_duration_seconds_bucket{endpoint="/api/users",le="0.05"} 14500
http_request_duration_seconds_bucket{endpoint="/api/users",le="0.1"} 15000
http_request_duration_seconds_bucket{endpoint="/api/users",le="0.25"} 15200
http_request_duration_seconds_bucket{endpoint="/api/users",le="+Inf"} 15234
http_request_duration_seconds_sum{endpoint="/api/users"} 456.78
http_request_duration_seconds_count{endpoint="/api/users"} 15234

# HELP active_connections Current number of active connections
# TYPE active_connections gauge
active_connections 1247
```

---

### Other Monitoring Tools

| Tool | Type | Key Features | Best For |
|------|------|-------------|----------|
| **Prometheus + Grafana** | Open-source | Pull-based, PromQL, rich ecosystem | Kubernetes, cloud-native |
| **Datadog** | SaaS | Metrics + logs + traces (unified), 700+ integrations | Full observability platform |
| **CloudWatch** | AWS managed | Native AWS integration, logs + metrics + alarms | AWS-native |
| **New Relic** | SaaS | APM, infrastructure, logs, browser monitoring | Full-stack observability |
| **Grafana Cloud** | SaaS | Managed Prometheus + Loki + Tempo | Open-source stack, managed |
| **InfluxDB + Telegraf** | Open-source | Push-based, SQL-like query language (Flux) | IoT, time-series heavy |
| **VictoriaMetrics** | Open-source | Prometheus-compatible, better performance/storage | High-cardinality metrics |

---

### Alerting

**Alert Design Principles:**

| Principle | Description |
|-----------|-------------|
| **Alert on symptoms, not causes** | Alert on "error rate > 1%" (symptom), not "CPU > 80%" (cause — may not affect users) |
| **Actionable** | Every alert should have a clear action — if you can't do anything, don't alert |
| **Low noise** | Too many alerts → alert fatigue → real alerts get ignored |
| **Severity levels** | Critical (page someone NOW), Warning (investigate soon), Info (FYI) |
| **Runbooks** | Every alert should link to a runbook with diagnosis and remediation steps |

**Alert Examples:**

```yaml
# Prometheus alert rules
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate (> 1%) for {{ $labels.service }}"
          runbook: "https://wiki.example.com/runbooks/high-error-rate"

      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency > 1s for {{ $labels.service }}"

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk space < 10% on {{ $labels.instance }}"
```

---

### SLI, SLO, SLA

**SLI (Service Level Indicator):**
A quantitative measure of a specific aspect of service quality.

```
Examples:
  - Request latency P99 < 200ms
  - Availability: successful requests / total requests
  - Error rate: 5xx responses / total responses
  - Throughput: requests per second
```

**SLO (Service Level Objective):**
A target value for an SLI — the level of service you aim to provide.

```
Examples:
  - "99.9% of requests will complete in < 200ms" (latency SLO)
  - "99.95% of requests will succeed" (availability SLO)
  - "Error rate will be < 0.1%" (error rate SLO)
```

**SLA (Service Level Agreement):**
A **contract** between a service provider and customer that specifies SLOs and the consequences of not meeting them (refunds, credits).

```
Example SLA:
  "We guarantee 99.9% monthly uptime. If we fail to meet this:
   - 99.0% - 99.9%: 10% service credit
   - 95.0% - 99.0%: 25% service credit
   - < 95.0%: 50% service credit"
```

**Relationship:**

```
SLI: "What we measure"     → P99 latency, availability, error rate
SLO: "What we target"      → P99 < 200ms, 99.95% availability
SLA: "What we promise"     → 99.9% uptime with financial penalties
     (external contract)

SLO is stricter than SLA (internal target > external promise)
  SLO: 99.95% availability (internal target)
  SLA: 99.9% availability (external promise — gives buffer)
```

---

### Error Budgets

An **error budget** is the amount of unreliability you can tolerate while still meeting your SLO. It's calculated as `1 - SLO`.

```
SLO: 99.9% availability
Error Budget: 1 - 0.999 = 0.1% = 43.8 minutes of downtime per month

If you've used 30 minutes of your error budget this month:
  Remaining: 13.8 minutes
  → Be cautious with deployments
  → Prioritize reliability over features

If you've used 0 minutes of your error budget:
  → You're "too reliable" — you could be deploying faster
  → Use the budget to ship features, experiment, take risks
```

**Error Budget Policy:**

| Budget Status | Action |
|--------------|--------|
| Budget remaining > 50% | Ship features, experiment, deploy frequently |
| Budget remaining 20-50% | Normal operations; be careful with risky changes |
| Budget remaining < 20% | Freeze feature deployments; focus on reliability |
| Budget exhausted (0%) | Stop all deployments; all engineering effort on reliability |

**Key Insight:** Error budgets align engineering and product teams. Product wants to ship features (risk). Engineering wants reliability (safety). The error budget gives both teams a shared, objective framework for making trade-off decisions.

---


## 30.3 Logging

> **Logs** are discrete, timestamped records of events that happened in the system. Unlike metrics (aggregated numbers), logs contain **detailed context** — the specific request that failed, the exact error message, the user who triggered it. Logs are essential for debugging, auditing, and forensic analysis.

---

### Structured Logging (JSON)

**Unstructured logs** (plain text) are hard to parse, search, and analyze at scale:

```
❌ Unstructured:
  2025-03-15 10:30:05 ERROR OrderService - Failed to create order for user 123: DB timeout after 5000ms

  Problems:
  - Hard to parse programmatically (regex needed)
  - Inconsistent format across services
  - Can't easily filter by user_id, error_type, latency
```

**Structured logs** (JSON) are machine-parseable and easily searchable:

```json
✅ Structured (JSON):
{
  "timestamp": "2025-03-15T10:30:05.123Z",
  "level": "ERROR",
  "service": "order-service",
  "instance": "order-service-pod-abc123",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "request_id": "req_xyz789",
  "user_id": "user_123",
  "method": "POST",
  "path": "/api/orders",
  "message": "Failed to create order",
  "error": {
    "type": "DatabaseTimeoutException",
    "message": "Connection timed out after 5000ms",
    "stack_trace": "at OrderRepository.save(OrderRepository.java:45)..."
  },
  "duration_ms": 5023,
  "order_id": null,
  "metadata": {
    "items_count": 3,
    "total_amount": 149.97
  }
}
```

**Benefits of Structured Logging:**
- **Searchable:** Filter by any field (`user_id = "user_123" AND level = "ERROR"`)
- **Parseable:** Log aggregation tools (Elasticsearch, Loki) can index and query JSON fields
- **Consistent:** Same format across all services
- **Correlatable:** `trace_id` and `request_id` link logs across services

---

### Centralized Logging (ELK Stack)

In a microservices architecture, logs are scattered across dozens of services and hundreds of instances. **Centralized logging** collects all logs into a single, searchable system.

**ELK Stack (Elasticsearch + Logstash + Kibana):**

```
ELK Architecture:

  +------------------+     +------------------+     +------------------+
  | Service A        |     | Service B        |     | Service C        |
  | (stdout/file)    |     | (stdout/file)    |     | (stdout/file)    |
  +--------+---------+     +--------+---------+     +--------+---------+
           |                        |                        |
  +--------v---------+     +--------v---------+     +--------v---------+
  | Filebeat /       |     | Filebeat /       |     | Filebeat /       |
  | Fluentd          |     | Fluentd          |     | Fluentd          |
  | (log shipper)    |     | (log shipper)    |     | (log shipper)    |
  +--------+---------+     +--------+---------+     +--------+---------+
           |                        |                        |
           +------------------------+------------------------+
                                    |
                           +--------v---------+
                           | Logstash /       |
                           | Fluentd          |
                           | (parse, enrich,  |
                           |  transform)      |
                           +--------+---------+
                                    |
                           +--------v---------+
                           | Elasticsearch    |
                           | (store, index,   |
                           |  search)         |
                           +--------+---------+
                                    |
                           +--------v---------+
                           | Kibana           |
                           | (visualize,      |
                           |  search, alert)  |
                           +------------------+
```

**Modern Alternative — Grafana Loki:**

| Feature | ELK (Elasticsearch) | Grafana Loki |
|---------|-------------------|-------------|
| Indexing | Full-text index (indexes log content) | Label-based index only (doesn't index content) |
| Storage cost | High (full-text index is large) | Low (stores compressed log chunks) |
| Query speed | Fast (indexed) | Slower for content search (grep-like) |
| Query language | KQL (Kibana Query Language) | LogQL (similar to PromQL) |
| Integration | Kibana | Grafana (same dashboard as metrics) |
| Complexity | High (Elasticsearch cluster management) | Low (simpler architecture) |
| Best for | Large-scale log analytics, full-text search | Cost-effective logging, Grafana users |

---

### Log Levels

| Level | When to Use | Example |
|-------|-------------|---------|
| **TRACE** | Very detailed debugging (method entry/exit, variable values) | `TRACE: Entering calculateTotal() with items=[...]` |
| **DEBUG** | Detailed information for debugging | `DEBUG: Cache miss for key user:123, fetching from DB` |
| **INFO** | Normal operational events | `INFO: Order ord_456 created for user user_123, total=$99.99` |
| **WARN** | Something unexpected but not an error; system can continue | `WARN: Retry 2/3 for payment service call, timeout after 3s` |
| **ERROR** | An error occurred; the operation failed | `ERROR: Failed to charge payment for order ord_456: insufficient funds` |
| **FATAL** | Critical error; the application is about to crash | `FATAL: Cannot connect to database after 10 retries, shutting down` |

**Log Level Best Practices:**
- **Production:** INFO and above (INFO, WARN, ERROR, FATAL)
- **Staging:** DEBUG and above
- **Development:** TRACE and above (everything)
- **Dynamic log levels:** Allow changing log level at runtime without restarting (useful for debugging production issues)
- **Don't log sensitive data:** Never log passwords, tokens, credit card numbers, PII

---

### Correlation IDs for Distributed Tracing

A **correlation ID** (also called request ID or trace ID) is a unique identifier that follows a request across all services, linking all related logs together.

```
Without Correlation ID:
  Order Service log:  "ERROR: Payment failed"
  Payment Service log: "ERROR: Timeout connecting to bank API"
  → How do you know these two logs are related to the same request?

With Correlation ID:
  Order Service log:  { "trace_id": "abc123", "message": "Payment failed" }
  Payment Service log: { "trace_id": "abc123", "message": "Timeout connecting to bank API" }
  → Search for trace_id = "abc123" → see the complete request flow!
```

**Implementation:**

```
1. API Gateway generates a unique trace_id for each incoming request
   (or uses the client-provided X-Request-Id header)

2. trace_id is passed to every downstream service via HTTP header:
   X-Trace-Id: abc123

3. Every service includes trace_id in all log entries

4. To debug: search for trace_id in centralized logging → see all logs for that request
```

---

### Log Aggregation and Retention

**Log Volume Estimation:**

```
100 services × 10 instances each × 100 log lines/second per instance
= 100,000 log lines/second
= 8.64 billion log lines/day
= ~1 TB/day (at ~120 bytes per log line, compressed)
= ~30 TB/month
```

**Retention Strategy:**

| Tier | Retention | Storage | Use Case |
|------|-----------|---------|----------|
| **Hot** | 7-14 days | Elasticsearch / Loki (fast search) | Active debugging, recent incidents |
| **Warm** | 30-90 days | Cheaper storage (S3 + Athena) | Incident investigation, compliance |
| **Cold** | 1-7 years | Archive (S3 Glacier) | Regulatory compliance, audit |
| **Delete** | After retention period | — | Reduce storage costs |

**Cost Optimization:**
- **Sample verbose logs** — log 10% of DEBUG-level logs in production
- **Aggregate before storing** — count errors per minute instead of storing every error log
- **Compress** — gzip/zstd compression reduces log storage by 80-90%
- **Tiered storage** — move old logs to cheaper storage automatically
- **Drop low-value logs** — health check logs, heartbeat logs add volume without value

---


## 30.4 Distributed Tracing

> In a microservices architecture, a single user request may traverse 5-20 services. When that request is slow or fails, you need to know **which service** is the bottleneck and **where** the error occurred. **Distributed tracing** follows a request across all services, recording timing and metadata at each step.

---

### Trace Concepts

```
Trace (one complete request):
  Trace ID: abc123

  ┌─────────────────────────────────────────────────────────────┐
  │ Span: API Gateway (5ms)                                     │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ Span: Order Service (250ms)                          │   │
  │  │  ┌───────────────────────────────┐                   │   │
  │  │  │ Span: Payment Service (180ms) │                   │   │
  │  │  │  ┌────────────────────┐       │                   │   │
  │  │  │  │ Span: DB Query     │       │                   │   │
  │  │  │  │ (150ms)            │       │                   │   │
  │  │  │  └────────────────────┘       │                   │   │
  │  │  └───────────────────────────────┘                   │   │
  │  │  ┌──────────────────────┐                            │   │
  │  │  │ Span: Inventory (30ms)│                           │   │
  │  │  └──────────────────────┘                            │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
  
  Total: 255ms
  Bottleneck: Payment Service → DB Query (150ms = 59% of total time)
```

**Key Concepts:**

| Concept | Description |
|---------|-------------|
| **Trace** | The complete journey of a request across all services; identified by a unique Trace ID |
| **Span** | A single unit of work within a trace (one service call, one DB query, one cache lookup) |
| **Span ID** | Unique identifier for each span |
| **Parent Span ID** | Links a span to its parent (creates the tree structure) |
| **Tags/Attributes** | Key-value metadata on a span (HTTP method, status code, user ID, SQL query) |
| **Events/Logs** | Timestamped events within a span (error occurred, retry attempted) |
| **Baggage** | Key-value pairs propagated across all spans in a trace (user ID, tenant ID) |

**Span Structure:**

```json
{
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "parent_span_id": "a3ce929d0e0e4736",
  "operation_name": "POST /api/payments",
  "service_name": "payment-service",
  "start_time": "2025-03-15T10:30:05.100Z",
  "duration_ms": 180,
  "status": "OK",
  "tags": {
    "http.method": "POST",
    "http.status_code": 200,
    "http.url": "/api/payments",
    "user.id": "user_123",
    "payment.amount": 99.99,
    "db.type": "postgresql",
    "db.statement": "INSERT INTO payments ..."
  },
  "events": [
    { "time": "2025-03-15T10:30:05.150Z", "name": "db.query.start" },
    { "time": "2025-03-15T10:30:05.280Z", "name": "db.query.end" }
  ]
}
```

---

### OpenTelemetry

**OpenTelemetry (OTel)** is the industry standard for generating, collecting, and exporting telemetry data (traces, metrics, logs). It's a CNCF project that unifies the previously separate OpenTracing and OpenCensus projects.

```
OpenTelemetry Architecture:

  +------------------+     +------------------+     +------------------+
  | Service A        |     | Service B        |     | Service C        |
  | (OTel SDK)       |     | (OTel SDK)       |     | (OTel SDK)       |
  | - Auto-instrument|     | - Auto-instrument|     | - Auto-instrument|
  | - Manual spans   |     | - Manual spans   |     | - Manual spans   |
  +--------+---------+     +--------+---------+     +--------+---------+
           |                        |                        |
           +------------------------+------------------------+
                                    |
                           +--------v---------+
                           | OTel Collector   |
                           | - Receive        |
                           | - Process        |
                           | - Export         |
                           +--------+---------+
                                    |
                    +---------------+---------------+
                    |               |               |
           +--------v------+ +-----v------+ +------v------+
           | Jaeger        | | Prometheus | | Loki        |
           | (traces)      | | (metrics)  | | (logs)      |
           +---------------+ +------------+ +-------------+
```

**OTel Components:**

| Component | Description |
|-----------|-------------|
| **SDK** | Library added to your application; auto-instruments HTTP, gRPC, DB calls; allows manual span creation |
| **API** | Vendor-neutral API for creating spans, metrics, logs |
| **Collector** | Receives telemetry data, processes it (filter, batch, enrich), exports to backends |
| **Exporters** | Send data to specific backends (Jaeger, Zipkin, Prometheus, Datadog, etc.) |
| **Auto-instrumentation** | Automatically creates spans for common libraries (HTTP clients, DB drivers, message queues) |

**Key Benefit:** Instrument once with OpenTelemetry, export to any backend. Switch from Jaeger to Datadog without changing application code.

---

### Tracing Tools

| Tool | Type | Key Features | Best For |
|------|------|-------------|----------|
| **Jaeger** (Uber) | Open-source | Distributed tracing, service dependency graph, root cause analysis | Kubernetes, open-source stack |
| **Zipkin** (Twitter) | Open-source | Distributed tracing, simpler than Jaeger | Simpler deployments |
| **Tempo** (Grafana) | Open-source | Trace storage backend; integrates with Grafana | Grafana ecosystem |
| **Datadog APM** | SaaS | Traces + metrics + logs unified; auto-instrumentation | Full observability platform |
| **AWS X-Ray** | AWS managed | Distributed tracing for AWS services | AWS-native |
| **Honeycomb** | SaaS | High-cardinality exploration, BubbleUp analysis | Debugging complex systems |

---

### Trace Propagation

For distributed tracing to work, the **trace context** (trace ID, span ID) must be propagated from service to service via HTTP headers.

```
Service A → HTTP Request → Service B
  Headers:
    traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
                 ↑version  ↑trace-id                    ↑parent-span-id  ↑sampled

  Service B reads the traceparent header:
    → Creates a new span with the same trace_id
    → Sets parent_span_id to the received span_id
    → Generates a new span_id for its own span
```

**W3C Trace Context** is the standard header format (supported by OpenTelemetry, Jaeger, Zipkin, Datadog):
- `traceparent`: `{version}-{trace-id}-{parent-span-id}-{trace-flags}`
- `tracestate`: vendor-specific key-value pairs

**Propagation in Different Protocols:**

| Protocol | How Context is Propagated |
|----------|--------------------------|
| HTTP | `traceparent` header (W3C standard) |
| gRPC | Metadata (headers) |
| Kafka | Message headers |
| RabbitMQ | Message headers/properties |
| AWS SQS | Message attributes |

---

### Sampling Strategies

At high traffic volumes, tracing **every** request is too expensive (storage, network, processing). **Sampling** reduces the volume while preserving useful traces.

| Strategy | Description | Pros | Cons |
|----------|-------------|------|------|
| **Head-based sampling** | Decide at the start of the trace whether to sample it | Simple; low overhead | May miss interesting traces (errors, slow requests) |
| **Tail-based sampling** | Decide after the trace is complete (based on duration, errors, etc.) | Captures interesting traces | Higher overhead (must buffer all spans until decision) |
| **Rate-based** | Sample N traces per second | Predictable volume | May miss rare events |
| **Probabilistic** | Sample X% of traces randomly | Simple; representative | May miss rare events |
| **Always sample errors** | Always trace requests that result in errors | Never miss errors | Doesn't help with latency debugging |
| **Adaptive** | Adjust sampling rate based on traffic volume | Consistent volume regardless of traffic | More complex |

**Recommended Approach:**
- **Always sample errors** and slow requests (P99+ latency)
- **Probabilistic sampling** for normal requests (1-10% depending on volume)
- **Tail-based sampling** if you can afford the infrastructure (best quality traces)

```
Sampling Decision:
  if request.has_error:
    sample = True (always trace errors)
  elif request.duration > p99_threshold:
    sample = True (always trace slow requests)
  elif random() < 0.05:
    sample = True (5% random sampling)
  else:
    sample = False (don't trace)
```

---


## 30.5 Health Checks & Readiness

> Health checks allow infrastructure (load balancers, Kubernetes, service mesh) to determine whether a service instance is **alive**, **ready to serve traffic**, and **fully started**. Proper health checks prevent routing traffic to broken instances and enable graceful deployments.

---

### Liveness Probes

**"Is the process alive?"** — Checks if the application process is running and not deadlocked. If the liveness probe fails, the infrastructure **restarts** the instance.

```
Liveness Probe:
  GET /health/live → 200 OK (process is alive)
  GET /health/live → no response (process is deadlocked or crashed)
  → Kubernetes restarts the pod

What to check:
  ✅ Process is running
  ✅ Main thread is responsive
  ✅ Not deadlocked
  
  ❌ Do NOT check external dependencies (database, cache, external APIs)
     → If the database is down, restarting the app won't fix it
     → All instances would restart simultaneously → cascading failure!
```

```yaml
# Kubernetes liveness probe
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 15    # wait 15s after container starts
  periodSeconds: 10          # check every 10 seconds
  timeoutSeconds: 3          # timeout after 3 seconds
  failureThreshold: 3        # restart after 3 consecutive failures
```

**Liveness Probe Implementation:**

```python
@app.get("/health/live")
def liveness():
    # Simple check: can the process respond to HTTP?
    # If this endpoint responds, the process is alive.
    return {"status": "alive"}
```

---

### Readiness Probes

**"Can this instance serve traffic?"** — Checks if the application is ready to handle requests. If the readiness probe fails, the instance is **removed from the load balancer** (but NOT restarted).

```
Readiness Probe:
  GET /health/ready → 200 OK (ready to serve traffic)
  GET /health/ready → 503 Service Unavailable (not ready)
  → Kubernetes removes pod from Service endpoints (no traffic routed to it)
  → Pod stays running; readiness is checked again later
  → When ready again → pod is added back to Service endpoints

What to check:
  ✅ Database connection is healthy
  ✅ Cache (Redis) connection is healthy
  ✅ Required configuration is loaded
  ✅ Warm-up is complete (caches populated, models loaded)
  
  These are the dependencies needed to serve requests correctly.
```

```yaml
# Kubernetes readiness probe
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5     # wait 5s before first check
  periodSeconds: 5           # check every 5 seconds
  timeoutSeconds: 3          # timeout after 3 seconds
  failureThreshold: 3        # remove from LB after 3 failures
  successThreshold: 1        # add back after 1 success
```

**Readiness Probe Implementation:**

```python
@app.get("/health/ready")
def readiness():
    checks = {}
    
    # Check database
    try:
        db.execute("SELECT 1")
        checks["database"] = {"status": "healthy", "latency_ms": 2}
    except Exception as e:
        checks["database"] = {"status": "unhealthy", "error": str(e)}
        return JSONResponse(status_code=503, content={"status": "not_ready", "checks": checks})
    
    # Check Redis
    try:
        redis.ping()
        checks["redis"] = {"status": "healthy", "latency_ms": 1}
    except Exception as e:
        checks["redis"] = {"status": "unhealthy", "error": str(e)}
        return JSONResponse(status_code=503, content={"status": "not_ready", "checks": checks})
    
    return {"status": "ready", "checks": checks}
```

---

### Startup Probes

**"Has the application finished starting?"** — For applications with long startup times (loading ML models, warming caches, running migrations). The startup probe runs **only during startup**; once it succeeds, liveness and readiness probes take over.

```yaml
# Kubernetes startup probe
startupProbe:
  httpGet:
    path: /health/started
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 5
  failureThreshold: 30       # 30 × 5s = 150 seconds max startup time
  # Liveness and readiness probes don't start until startup probe succeeds
```

**Why Startup Probes Exist:**
Without a startup probe, a slow-starting application might fail the liveness probe during startup → Kubernetes restarts it → it starts again → fails liveness again → restart loop!

The startup probe gives the application time to start without being killed by the liveness probe.

---

### Liveness vs Readiness vs Startup

| Probe | Question | On Failure | Check |
|-------|----------|-----------|-------|
| **Liveness** | Is the process alive? | **Restart** the container | Process health only (no external deps) |
| **Readiness** | Can it serve traffic? | **Remove** from load balancer | External dependencies (DB, cache) |
| **Startup** | Has it finished starting? | **Restart** (after threshold) | Application initialization complete |

```
Container Lifecycle:

  Container starts
       |
  [Startup Probe] ← runs until success (or failure threshold → restart)
       |
  Startup succeeds
       |
  [Liveness Probe] ← runs continuously (failure → restart)
  [Readiness Probe] ← runs continuously (failure → remove from LB)
       |
  Container running and serving traffic
       |
  SIGTERM received (deployment, scaling down)
       |
  [Graceful Shutdown]
       |
  Container stops
```

---

### Graceful Shutdown

When a container is being stopped (deployment, scaling down, node drain), it should **finish in-flight requests** before exiting — not abruptly drop connections.

```
Graceful Shutdown Sequence:

  1. Kubernetes sends SIGTERM to the container
  2. Kubernetes removes the pod from Service endpoints (no NEW traffic)
  3. Application receives SIGTERM:
     a. Stop accepting new connections
     b. Finish processing in-flight requests (drain)
     c. Close database connections
     d. Flush logs and metrics
     e. Exit with code 0
  4. If the application doesn't exit within terminationGracePeriodSeconds (default: 30s):
     → Kubernetes sends SIGKILL (forced kill)
```

```yaml
# Kubernetes pod spec
spec:
  terminationGracePeriodSeconds: 60  # give 60 seconds for graceful shutdown
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]
            # Wait 5 seconds for load balancer to stop sending traffic
            # (there's a brief window where LB still routes to the pod after SIGTERM)
```

**Graceful Shutdown Implementation:**

```python
import signal
import sys

def graceful_shutdown(signum, frame):
    print("SIGTERM received, starting graceful shutdown...")
    
    # 1. Stop accepting new connections
    server.stop_accepting()
    
    # 2. Wait for in-flight requests to complete (max 30 seconds)
    server.drain(timeout=30)
    
    # 3. Close database connections
    db.close()
    
    # 4. Close Redis connections
    redis.close()
    
    # 5. Flush logs
    logger.flush()
    
    print("Graceful shutdown complete")
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

**Key Point:** Without graceful shutdown, users see connection reset errors during deployments. With graceful shutdown, deployments are invisible to users — zero-downtime deployments.

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Three Pillars | Metrics (aggregated numbers), Logs (detailed events), Traces (request flow) — use all three together |
| Metrics | Counter (monotonic), Gauge (up/down), Histogram (distribution) — Prometheus is the standard |
| PromQL | `rate()` for request rate, `histogram_quantile()` for percentiles, label-based filtering |
| Alerting | Alert on symptoms (error rate), not causes (CPU); every alert must be actionable with a runbook |
| SLI/SLO/SLA | SLI = what you measure; SLO = what you target; SLA = what you promise (with penalties) |
| Error Budgets | 1 - SLO = allowed unreliability; budget remaining guides feature vs reliability trade-offs |
| Structured Logging | JSON format; searchable fields; consistent across services |
| Centralized Logging | ELK (Elasticsearch + Logstash + Kibana) or Grafana Loki; collect all logs in one place |
| Log Levels | TRACE/DEBUG (development), INFO (normal ops), WARN (unexpected), ERROR (failures), FATAL (crash) |
| Correlation IDs | Unique trace_id propagated across all services; links related logs together |
| Log Retention | Hot (7-14 days, fast search), Warm (30-90 days, cheaper), Cold (1-7 years, archive) |
| Distributed Tracing | Trace = full request journey; Span = one unit of work; Trace ID links all spans |
| OpenTelemetry | Industry standard for instrumentation; instrument once, export to any backend |
| Trace Propagation | W3C `traceparent` header; propagated via HTTP headers, gRPC metadata, message headers |
| Sampling | Always sample errors + slow requests; probabilistic for normal traffic; tail-based for best quality |
| Liveness Probe | "Is the process alive?" → restart on failure; do NOT check external dependencies |
| Readiness Probe | "Can it serve traffic?" → remove from LB on failure; check DB, cache, dependencies |
| Startup Probe | "Has it finished starting?" → prevents liveness probe from killing slow-starting apps |
| Graceful Shutdown | SIGTERM → stop accepting → drain in-flight → close connections → exit; enables zero-downtime deploys |

---

## Interview Tips for Module 30

1. **Three pillars** — explain metrics, logs, traces and how they work together; give the debugging flow example
2. **Metric types** — explain counter, gauge, histogram; know when to use each
3. **Prometheus** — explain the pull model; write basic PromQL queries (rate, histogram_quantile)
4. **SLI/SLO/SLA** — explain the relationship; give concrete examples; mention error budgets
5. **Error budgets** — explain how they balance feature velocity and reliability; mention the budget policy
6. **Structured logging** — explain why JSON; mention correlation IDs for cross-service debugging
7. **ELK vs Loki** — ELK indexes content (fast search, expensive); Loki indexes labels only (cheaper, slower content search)
8. **Distributed tracing** — draw a trace waterfall; explain trace ID, span ID, parent span ID
9. **OpenTelemetry** — mention as the industry standard; instrument once, export to any backend
10. **Sampling** — explain why you can't trace everything; always sample errors; probabilistic for normal traffic
11. **Health checks** — explain liveness vs readiness vs startup; warn about checking external deps in liveness (cascading failure!)
12. **Graceful shutdown** — explain the SIGTERM → drain → close → exit sequence; mention zero-downtime deployments
