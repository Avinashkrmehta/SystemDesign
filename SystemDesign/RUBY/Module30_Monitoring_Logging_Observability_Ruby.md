# Module 30: Monitoring, Logging & Observability

> You can't fix what you can't see. In a distributed system with dozens of services, understanding what's happening — why a request is slow, where errors originate, which service is the bottleneck — requires **observability**. This module covers the three pillars of observability (metrics, logs, traces), the tools and patterns for implementing each, and the operational practices (SLIs, SLOs, error budgets) that turn raw data into actionable insights.

> **Ruby Context:** Ruby has a mature observability ecosystem. Key tools include **Prometheus client** (`prometheus-client` gem for metrics exposition), **Lograge** / **Semantic Logger** (structured logging for Rails), **OpenTelemetry** (`opentelemetry-sdk` + auto-instrumentation gems for traces, metrics, and logs), **Yabeda** (metrics framework with Prometheus/Datadog adapters), **Sentry** / **Honeybadger** / **Bugsnag** (error tracking), and **rails_health_check** / custom health endpoints for Kubernetes probes. Rails 7.1+ includes built-in health check endpoints. Puma (the standard Rails app server) supports graceful shutdown natively.

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
  Error rate: errors_total / http_requests_total
```

**Gauge:**
A value that can go up or down. Represents a current state or snapshot.

```
Examples:
  cpu_usage_percent = 73.5
  memory_used_bytes = 8,589,934,592
  active_connections = 1,247
  queue_depth = 342
  sidekiq_queue_size = 1,500
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
    P50 (median): ~30ms
    P95: ~200ms
    P99: ~800ms
```

**Histogram vs Summary:**

| Feature | Histogram | Summary |
|---------|-----------|---------|
| Percentile calculation | Server-side (approximate, from buckets) | Client-side (exact, pre-computed) |
| Aggregation across instances | ✅ Yes (merge bucket counts) | ❌ No (can't merge pre-computed percentiles) |
| Recommendation | ✅ Preferred (aggregatable) | Use when exact percentiles are critical |

---

### Prometheus + Grafana

**Prometheus** is the de facto standard for metrics collection in cloud-native systems. **Grafana** is the standard for visualization.

```
Prometheus Architecture:

  +------------------+     Scrape (pull)     +------------------+
  | Puma (Rails)     | ←──────────────────── | Prometheus       |
  | /metrics endpoint|                       | Server           |
  +------------------+                       |                  |
  +------------------+     Scrape            | - Time-series DB |
  | Sidekiq          | ←──────────────────── | - PromQL queries |
  | /metrics endpoint|                       | - Alert rules    |
  +------------------+                       +--------+---------+
  +------------------+     Scrape                     |
  | Redis            | ←──────────────────── ─────────+
  | (via exporter)   |                       |
  +------------------+                       |
                                    +--------v---------+
                                    | Grafana          |
                                    | - Dashboards     |
                                    +------------------+
                                    +------------------+
                                    | Alertmanager     |
                                    | - PagerDuty/Slack|
                                    +------------------+
```

> **Ruby Context:** The `prometheus-client` gem exposes a `/metrics` endpoint from your Rails app. The **Yabeda** framework provides a higher-level API with adapters for Prometheus, Datadog, and NewRelic. For Sidekiq metrics, `yabeda-sidekiq` auto-instruments queue sizes, job durations, and error rates.

```ruby
# Option 1: prometheus-client gem (low-level)
# Gemfile: gem 'prometheus-client'

require 'prometheus/client'

# Create a registry and metrics
prometheus = Prometheus::Client.registry

http_requests = prometheus.counter(
  :http_requests_total,
  docstring: 'Total HTTP requests',
  labels: [:method, :path, :status]
)

http_duration = prometheus.histogram(
  :http_request_duration_seconds,
  docstring: 'HTTP request latency',
  labels: [:method, :path],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

active_connections = prometheus.gauge(
  :active_connections,
  docstring: 'Current active connections'
)

# Rack middleware to expose /metrics endpoint
# config/application.rb
require 'prometheus/middleware/exporter'
require 'prometheus/middleware/collector'

config.middleware.use Prometheus::Middleware::Collector  # Auto-collect HTTP metrics
config.middleware.use Prometheus::Middleware::Exporter   # Expose /metrics endpoint

# Option 2: Yabeda framework (higher-level, recommended)
# Gemfile: gem 'yabeda-prometheus'
#          gem 'yabeda-rails'
#          gem 'yabeda-sidekiq'
#          gem 'yabeda-puma-plugin'

# config/initializers/yabeda.rb
Yabeda.configure do
  # Custom business metrics
  group :orders do
    counter :created_total, comment: 'Total orders created', tags: [:payment_method]
    histogram :processing_duration, comment: 'Order processing time',
              unit: :seconds, buckets: [0.1, 0.5, 1, 2, 5, 10]
  end

  group :payments do
    counter :processed_total, comment: 'Total payments processed', tags: [:status]
    gauge :pending_count, comment: 'Payments pending processing'
  end
end

# Instrument in application code
class OrderService
  def create_order(params)
    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    order = Order.create!(params)
    process_payment(order)

    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time
    Yabeda.orders.created_total.increment({ payment_method: order.payment_method })
    Yabeda.orders.processing_duration.observe(duration)

    order
  rescue PaymentError => e
    Yabeda.payments.processed_total.increment({ status: 'failed' })
    raise
  end
end

# Mount /metrics endpoint
# config/routes.rb
mount Yabeda::Prometheus::Exporter => '/metrics' if Rails.env.production?
```

**PromQL Examples:**

```promql
# Request rate (requests per second over the last 5 minutes)
rate(http_requests_total[5m])

# Error rate (percentage)
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100

# P99 latency (from histogram)
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Sidekiq queue depth
sidekiq_queue_size{queue="default"}

# Puma thread utilization
puma_running / puma_max_threads * 100

# ActiveRecord connection pool usage
active_record_connection_pool_size - active_record_connection_pool_available
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
| **Scout APM** | SaaS | Ruby-focused APM, N+1 detection, memory bloat tracking | Ruby/Rails-specific |
| **Skylight** | SaaS | Rails-focused performance monitoring, smart UI | Rails-specific |

> **Ruby Context:** **Scout APM** and **Skylight** are Ruby/Rails-specific APM tools that provide deep Rails insights (N+1 query detection, memory bloat, slow view rendering) without the complexity of general-purpose tools. For larger deployments, Datadog or New Relic with their Ruby agents provide unified observability.

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
# Prometheus alert rules for a Rails application
groups:
  - name: rails-alerts
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

      - alert: SidekiqQueueBacklog
        expr: sidekiq_queue_size{queue="default"} > 10000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Sidekiq default queue has > 10,000 jobs"

      - alert: PumaThreadSaturation
        expr: (puma_running / puma_max_threads) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Puma thread utilization > 90% — may need more workers"

      - alert: ActiveRecordPoolExhausted
        expr: active_record_connection_pool_available == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "ActiveRecord connection pool exhausted"
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
A **contract** between a service provider and customer that specifies SLOs and the consequences of not meeting them.

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

---


## 30.3 Logging

> **Logs** are discrete, timestamped records of events that happened in the system. Unlike metrics (aggregated numbers), logs contain **detailed context** — the specific request that failed, the exact error message, the user who triggered it. Logs are essential for debugging, auditing, and forensic analysis.

---

### Structured Logging (JSON)

**Unstructured logs** (plain text) are hard to parse, search, and analyze at scale:

```
❌ Unstructured (Rails default):
  Started GET "/api/users/123" for 10.0.1.5 at 2025-03-15 10:30:05 +0000
  Processing by UsersController#show as JSON
    Parameters: {"id"=>"123"}
    User Load (2.3ms)  SELECT "users".* FROM "users" WHERE "users"."id" = $1
  Completed 200 OK in 15ms (Views: 3.2ms | ActiveRecord: 2.3ms)

  Problems:
  - Multi-line (hard to ship to log aggregation)
  - Hard to parse programmatically
  - No request ID, user context, or trace ID
```

**Structured logs** (JSON) are machine-parseable and easily searchable:

```json
✅ Structured (JSON with Lograge):
{
  "timestamp": "2025-03-15T10:30:05.123Z",
  "level": "INFO",
  "method": "GET",
  "path": "/api/users/123",
  "format": "json",
  "controller": "UsersController",
  "action": "show",
  "status": 200,
  "duration": 15.2,
  "view": 3.2,
  "db": 2.3,
  "request_id": "req_xyz789",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "user_id": "user_456",
  "ip": "10.0.1.5",
  "params": {"id": "123"}
}
```

> **Ruby Context:** **Lograge** is the most popular gem for converting Rails' verbose multi-line logs into single-line structured JSON. **Semantic Logger** is a more comprehensive alternative that provides structured logging across the entire application (not just Rails requests).

```ruby
# Option 1: Lograge (most common for Rails)
# Gemfile: gem 'lograge'

# config/initializers/lograge.rb
Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.formatter = Lograge::Formatters::Json.new

  # Add custom fields to every log line
  config.lograge.custom_options = lambda do |event|
    {
      request_id: event.payload[:request_id],
      user_id: event.payload[:user_id],
      trace_id: event.payload[:trace_id],
      ip: event.payload[:ip],
      user_agent: event.payload[:user_agent],
      params: event.payload[:params]&.except('controller', 'action', 'format')
    }.compact
  end

  # Add custom payload from controllers
  config.lograge.custom_payload do |controller|
    {
      user_id: controller.current_user&.id,
      trace_id: controller.request.headers['X-Trace-Id'],
      ip: controller.request.remote_ip,
      user_agent: controller.request.user_agent
    }
  end
end

# Option 2: Semantic Logger (comprehensive structured logging)
# Gemfile: gem 'rails_semantic_logger'

# config/application.rb
config.rails_semantic_logger.semantic = true
config.rails_semantic_logger.started = false  # Don't log "Started GET..."
config.rails_semantic_logger.processing = false
config.rails_semantic_logger.rendered = false
config.semantic_logger.add_appender(
  io: $stdout,
  formatter: :json
)

# Usage anywhere in the app
Rails.logger.info('Order created',
  order_id: order.id,
  user_id: current_user.id,
  total: order.total,
  payment_method: order.payment_method
)
# Output: {"timestamp":"2025-03-15T10:30:05.123Z","level":"info",
#          "message":"Order created","order_id":456,"user_id":123,
#          "total":99.99,"payment_method":"credit_card"}

# Tagged logging (built into Rails)
Rails.logger.tagged('OrderService', "user:#{user.id}") do
  Rails.logger.info("Processing order #{order.id}")
end
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
  | Puma (Rails)     |     | Sidekiq Worker   |     | Other Service    |
  | (stdout JSON)    |     | (stdout JSON)    |     | (stdout JSON)    |
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
                           | (parse, enrich)  |
                           +--------+---------+
                                    |
                           +--------v---------+
                           | Elasticsearch    |
                           | (store, index)   |
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

> **Ruby Context:** For Kubernetes-based Rails deployments, the standard pattern is: Rails logs to stdout (JSON via Lograge) → Fluentd/Filebeat collects from container stdout → ships to Elasticsearch or Loki. No file-based logging needed — Kubernetes captures stdout automatically.

---

### Log Levels

| Level | When to Use | Example |
|-------|-------------|---------|
| **TRACE** | Very detailed debugging | `TRACE: ActiveRecord query plan for users#show` |
| **DEBUG** | Detailed information for debugging | `DEBUG: Cache miss for key user:123, fetching from DB` |
| **INFO** | Normal operational events | `INFO: Order ord_456 created for user user_123, total=$99.99` |
| **WARN** | Something unexpected but not an error | `WARN: Retry 2/3 for payment service call, timeout after 3s` |
| **ERROR** | An error occurred; the operation failed | `ERROR: Failed to charge payment for order ord_456` |
| **FATAL** | Critical error; the application is about to crash | `FATAL: Cannot connect to database after 10 retries` |

> **Ruby Context:** Rails uses `Rails.logger` which supports all standard log levels. Set the level per environment in `config/environments/*.rb`.

```ruby
# config/environments/production.rb
config.log_level = :info  # INFO and above in production

# config/environments/development.rb
config.log_level = :debug  # DEBUG and above in development

# Dynamic log level change at runtime (without restart)
# Useful for debugging production issues temporarily
Rails.logger.level = Logger::DEBUG  # Enable debug logging
# ... debug the issue ...
Rails.logger.level = Logger::INFO   # Restore normal level

# Don't log sensitive data
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :password, :secret, :token, :_key, :ssn, :credit_card
]
# Logs show: Parameters: {"email"=>"alice@example.com", "password"=>"[FILTERED]"}
```

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

> **Ruby Context:** Rails generates a unique `request_id` for every request (via `ActionDispatch::RequestId` middleware). You can propagate it to downstream services and include it in all logs.

```ruby
# Rails auto-generates request_id (X-Request-Id header)
# Access in controllers:
request.request_id  # => "req_abc123-def456-..."

# Propagate to downstream services via Faraday
class ServiceClient
  def initialize
    @conn = Faraday.new do |f|
      f.request :json
      f.response :json
      # Propagate request ID and trace ID to downstream services
      f.use :request_id_propagation
    end
  end
end

# Custom Faraday middleware to propagate correlation IDs
class RequestIdPropagation < Faraday::Middleware
  def call(env)
    env.request_headers['X-Request-Id'] = Thread.current[:request_id]
    env.request_headers['X-Trace-Id'] = Thread.current[:trace_id]
    @app.call(env)
  end
end

# Set correlation IDs in a before_action
class ApplicationController < ActionController::Base
  before_action :set_correlation_ids

  private

  def set_correlation_ids
    Thread.current[:request_id] = request.request_id
    Thread.current[:trace_id] = request.headers['X-Trace-Id'] || SecureRandom.hex(16)
  end
end

# Include in Sidekiq jobs
class ApplicationJob < ActiveJob::Base
  around_perform do |job, block|
    Thread.current[:trace_id] = job.arguments.last[:trace_id] if job.arguments.last.is_a?(Hash)
    block.call
  end
end
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

---

### OpenTelemetry

**OpenTelemetry (OTel)** is the industry standard for generating, collecting, and exporting telemetry data (traces, metrics, logs). It's a CNCF project that unifies the previously separate OpenTracing and OpenCensus projects.

```
OpenTelemetry Architecture:

  +------------------+     +------------------+     +------------------+
  | Rails App        |     | Sidekiq Worker   |     | Payment Service  |
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

> **Ruby Context:** OpenTelemetry has excellent Ruby support. The `opentelemetry-sdk` gem provides the core SDK, and auto-instrumentation gems automatically trace Rails, ActiveRecord, Faraday, Redis, Sidekiq, and more — with zero code changes.

```ruby
# OpenTelemetry setup for Rails
# Gemfile:
# gem 'opentelemetry-sdk'
# gem 'opentelemetry-exporter-otlp'
# gem 'opentelemetry-instrumentation-all'  # Auto-instrument everything

# config/initializers/opentelemetry.rb
require 'opentelemetry/sdk'
require 'opentelemetry/exporter/otlp'
require 'opentelemetry/instrumentation/all'

OpenTelemetry::SDK.configure do |c|
  c.service_name = 'order-service'
  c.service_version = ENV.fetch('APP_VERSION', '1.0.0')

  # Auto-instrument Rails, ActiveRecord, Faraday, Redis, Sidekiq, Net::HTTP
  c.use_all({
    'OpenTelemetry::Instrumentation::Rails' => {
      enable_recognize_route: true
    },
    'OpenTelemetry::Instrumentation::ActiveRecord' => {
      # Captures SQL queries as span attributes
    },
    'OpenTelemetry::Instrumentation::Faraday' => {
      # Traces outgoing HTTP calls and propagates context
    },
    'OpenTelemetry::Instrumentation::Sidekiq' => {
      # Traces job enqueue and perform
    }
  })

  # Export to OTel Collector (or directly to Jaeger/Datadog)
  c.add_span_processor(
    OpenTelemetry::SDK::Trace::Export::BatchSpanProcessor.new(
      OpenTelemetry::Exporter::OTLP::Exporter.new(
        endpoint: ENV.fetch('OTEL_EXPORTER_OTLP_ENDPOINT', 'http://otel-collector:4318')
      )
    )
  )
end

# Auto-instrumentation creates spans automatically for:
# - Every Rails controller action
# - Every ActiveRecord query
# - Every Faraday HTTP call (with trace context propagation)
# - Every Redis command
# - Every Sidekiq job enqueue and perform

# Manual span creation for custom business logic
class OrderService
  def create_order(params)
    tracer = OpenTelemetry.tracer_provider.tracer('order-service')

    tracer.in_span('create_order', attributes: {
      'order.user_id' => params[:user_id],
      'order.items_count' => params[:items].size
    }) do |span|
      order = Order.create!(params)

      # Add event to span
      span.add_event('order.created', attributes: { 'order.id' => order.id })

      # Nested span for payment processing
      tracer.in_span('process_payment', attributes: {
        'payment.amount' => order.total,
        'payment.method' => order.payment_method
      }) do |payment_span|
        result = PaymentClient.new.charge(order)
        payment_span.set_attribute('payment.transaction_id', result.transaction_id)
      end

      order
    rescue => e
      span&.record_exception(e)
      span&.status = OpenTelemetry::Trace::Status.error(e.message)
      raise
    end
  end
end
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

**Ruby Auto-Instrumentation Gems:**

| Gem | What It Traces |
|-----|---------------|
| `opentelemetry-instrumentation-rails` | Controller actions, middleware |
| `opentelemetry-instrumentation-active_record` | SQL queries |
| `opentelemetry-instrumentation-faraday` | Outgoing HTTP calls + context propagation |
| `opentelemetry-instrumentation-net_http` | Net::HTTP calls + context propagation |
| `opentelemetry-instrumentation-redis` | Redis commands |
| `opentelemetry-instrumentation-sidekiq` | Job enqueue and perform |
| `opentelemetry-instrumentation-pg` | PostgreSQL queries |
| `opentelemetry-instrumentation-rack` | Rack middleware |
| `opentelemetry-instrumentation-action_pack` | Action Controller |
| `opentelemetry-instrumentation-active_job` | ActiveJob enqueue and perform |

---

### Tracing Tools

| Tool | Type | Key Features | Best For |
|------|------|-------------|----------|
| **Jaeger** (Uber) | Open-source | Distributed tracing, service dependency graph | Kubernetes, open-source stack |
| **Zipkin** (Twitter) | Open-source | Distributed tracing, simpler than Jaeger | Simpler deployments |
| **Tempo** (Grafana) | Open-source | Trace storage backend; integrates with Grafana | Grafana ecosystem |
| **Datadog APM** | SaaS | Traces + metrics + logs unified | Full observability platform |
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
```

**W3C Trace Context** is the standard header format (supported by OpenTelemetry, Jaeger, Zipkin, Datadog).

**Propagation in Different Protocols:**

| Protocol | How Context is Propagated |
|----------|--------------------------|
| HTTP | `traceparent` header (W3C standard) |
| gRPC | Metadata (headers) |
| Kafka | Message headers (Karafka propagates automatically with OTel) |
| RabbitMQ | Message headers/properties |
| Sidekiq | Job arguments metadata (OTel Sidekiq instrumentation handles this) |

> **Ruby Context:** OpenTelemetry's Faraday and Net::HTTP instrumentation automatically injects `traceparent` headers into outgoing requests. The Sidekiq instrumentation propagates trace context through job metadata. No manual header management needed.

---

### Sampling Strategies

At high traffic volumes, tracing **every** request is too expensive. **Sampling** reduces the volume while preserving useful traces.

| Strategy | Description | Pros | Cons |
|----------|-------------|------|------|
| **Head-based sampling** | Decide at the start of the trace | Simple; low overhead | May miss interesting traces |
| **Tail-based sampling** | Decide after the trace is complete | Captures interesting traces | Higher overhead |
| **Rate-based** | Sample N traces per second | Predictable volume | May miss rare events |
| **Probabilistic** | Sample X% of traces randomly | Simple; representative | May miss rare events |
| **Always sample errors** | Always trace requests that result in errors | Never miss errors | Doesn't help with latency debugging |

```ruby
# Configure sampling in OpenTelemetry for Ruby
OpenTelemetry::SDK.configure do |c|
  # Probabilistic sampling — trace 10% of requests
  c.add_span_processor(
    OpenTelemetry::SDK::Trace::Export::BatchSpanProcessor.new(
      OpenTelemetry::Exporter::OTLP::Exporter.new
    )
  )

  # Use parent-based sampler with 10% probability for root spans
  c.traces.sampler = OpenTelemetry::SDK::Trace::Samplers.parent_based(
    root: OpenTelemetry::SDK::Trace::Samplers.trace_id_ratio_based(0.1)
  )
end

# For tail-based sampling, configure it in the OTel Collector (not in the app)
# The collector buffers spans and makes sampling decisions after the trace completes
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

> **Ruby Context:** Rails 7.1+ includes a built-in health check endpoint at `/up`. For older Rails versions or custom health checks, create a simple controller.

```ruby
# Rails 7.1+ built-in health check
# GET /up → 200 OK (automatically configured)
# config/routes.rb — already included by default in Rails 7.1+

# Custom liveness probe (for older Rails or custom logic)
# config/routes.rb
get '/health/live', to: 'health#liveness'

# app/controllers/health_controller.rb
class HealthController < ActionController::API
  # Liveness — is the process alive?
  # Do NOT check external dependencies here!
  def liveness
    render json: { status: 'alive', pid: Process.pid }, status: :ok
  end
end
```

```yaml
# Kubernetes liveness probe
livenessProbe:
  httpGet:
    path: /health/live
    port: 3000
  initialDelaySeconds: 15    # wait 15s after container starts
  periodSeconds: 10          # check every 10 seconds
  timeoutSeconds: 3          # timeout after 3 seconds
  failureThreshold: 3        # restart after 3 consecutive failures
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

What to check:
  ✅ Database connection is healthy
  ✅ Redis connection is healthy
  ✅ Required configuration is loaded
  ✅ Warm-up is complete (caches populated)
```

> **Ruby Context:** The readiness probe should verify that ActiveRecord can connect to the database and Redis is reachable. The `rails_health_check` gem provides configurable health checks, or you can build a custom one.

```ruby
# Custom readiness probe
# app/controllers/health_controller.rb
class HealthController < ActionController::API
  # Readiness — can we serve traffic?
  def readiness
    checks = {}

    # Check database
    begin
      ActiveRecord::Base.connection.execute('SELECT 1')
      checks[:database] = { status: 'healthy' }
    rescue => e
      checks[:database] = { status: 'unhealthy', error: e.message }
      return render json: { status: 'not_ready', checks: checks }, status: :service_unavailable
    end

    # Check Redis
    begin
      Redis.current.ping
      checks[:redis] = { status: 'healthy' }
    rescue => e
      checks[:redis] = { status: 'unhealthy', error: e.message }
      return render json: { status: 'not_ready', checks: checks }, status: :service_unavailable
    end

    # Check Sidekiq connectivity (can we enqueue jobs?)
    begin
      Sidekiq.redis { |conn| conn.ping }
      checks[:sidekiq] = { status: 'healthy' }
    rescue => e
      checks[:sidekiq] = { status: 'unhealthy', error: e.message }
      # Sidekiq being down might be acceptable — depends on your app
    end

    render json: { status: 'ready', checks: checks }, status: :ok
  end
end

# Using rails_health_check gem (more structured)
# Gemfile: gem 'rails_health_check'

# config/initializers/health_check.rb
HealthCheck.setup do |config|
  config.standard_checks = ['database', 'redis', 'sidekiq-redis']
  config.full_checks = ['database', 'redis', 'sidekiq-redis', 'migrations']
end

# Routes: GET /health_check → checks all standard checks
#         GET /health_check/full → checks all full checks
```

```yaml
# Kubernetes readiness probe
readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
  initialDelaySeconds: 5     # wait 5s before first check
  periodSeconds: 5           # check every 5 seconds
  timeoutSeconds: 3          # timeout after 3 seconds
  failureThreshold: 3        # remove from LB after 3 failures
  successThreshold: 1        # add back after 1 success
```

---

### Startup Probes

**"Has the application finished starting?"** — For applications with long startup times. The startup probe runs **only during startup**; once it succeeds, liveness and readiness probes take over.

```yaml
# Kubernetes startup probe for Rails
startupProbe:
  httpGet:
    path: /health/live
    port: 3000
  initialDelaySeconds: 0
  periodSeconds: 5
  failureThreshold: 30       # 30 × 5s = 150 seconds max startup time
  # Liveness and readiness probes don't start until startup probe succeeds
```

> **Ruby Context:** Rails apps can have slow startup times due to eager loading, asset compilation, or database migrations. The startup probe prevents Kubernetes from killing the pod during this initialization phase. Puma's `on_booted` callback can signal when the app is fully started.

---

### Liveness vs Readiness vs Startup

| Probe | Question | On Failure | Check |
|-------|----------|-----------|-------|
| **Liveness** | Is the process alive? | **Restart** the container | Process health only (no external deps) |
| **Readiness** | Can it serve traffic? | **Remove** from load balancer | External dependencies (DB, Redis) |
| **Startup** | Has it finished starting? | **Restart** (after threshold) | Application initialization complete |

```
Container Lifecycle:

  Container starts → Puma boots → Rails initializes
       |
  [Startup Probe] ← runs until success (or failure threshold → restart)
       |
  Startup succeeds (Rails is loaded, Puma is listening)
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
  3. Puma receives SIGTERM:
     a. Stop accepting new connections
     b. Finish processing in-flight requests (drain)
     c. Close database connections (ActiveRecord pool)
     d. Close Redis connections
     e. Flush logs and metrics
     f. Exit with code 0
  4. If the application doesn't exit within terminationGracePeriodSeconds (default: 30s):
     → Kubernetes sends SIGKILL (forced kill)
```

> **Ruby Context:** Puma handles graceful shutdown natively — it traps SIGTERM, stops accepting new connections, and waits for in-flight requests to complete. Sidekiq also handles SIGTERM gracefully, finishing the current job before shutting down. You can add custom shutdown hooks for cleanup.

```ruby
# Puma graceful shutdown configuration (config/puma.rb)
workers ENV.fetch('WEB_CONCURRENCY', 2).to_i
threads_count = ENV.fetch('RAILS_MAX_THREADS', 5).to_i
threads threads_count, threads_count

preload_app!

on_worker_boot do
  ActiveRecord::Base.establish_connection
end

on_worker_shutdown do
  # Custom cleanup on worker shutdown
  Rails.logger.info('Puma worker shutting down — flushing metrics')
  # Flush any buffered metrics/traces
  OpenTelemetry.tracer_provider.force_flush if defined?(OpenTelemetry)
end

# Puma handles SIGTERM automatically:
# 1. Stops accepting new connections
# 2. Waits for in-flight requests to complete (up to worker_timeout)
# 3. Shuts down workers gracefully

# Sidekiq graceful shutdown (config/sidekiq.yml)
# :timeout: 25  # Wait up to 25 seconds for current job to finish
#               # (less than K8s terminationGracePeriodSeconds of 30)

# Custom shutdown hooks for Rails
# config/initializers/shutdown_hooks.rb
at_exit do
  Rails.logger.info('Application shutting down')

  # Flush OpenTelemetry spans
  if defined?(OpenTelemetry)
    OpenTelemetry.tracer_provider.force_flush
    OpenTelemetry.tracer_provider.shutdown
  end

  # Close custom connections
  EventPublisher.instance.close if defined?(EventPublisher)
end

# Kubernetes pod spec for Rails
# spec:
#   terminationGracePeriodSeconds: 60  # Give 60 seconds for graceful shutdown
#   containers:
#     - name: rails
#       lifecycle:
#         preStop:
#           exec:
#             command: ["/bin/sh", "-c", "sleep 5"]
#             # Wait 5 seconds for load balancer to stop sending traffic
```

**Key Point:** Without graceful shutdown, users see connection reset errors during deployments. With graceful shutdown, deployments are invisible to users — zero-downtime deployments.

> **Ruby Context:** For zero-downtime deployments with Puma on Kubernetes:
> 1. `preStop` hook sleeps 5s (lets LB drain connections)
> 2. Puma receives SIGTERM → stops accepting → drains in-flight requests
> 3. Sidekiq receives SIGTERM → finishes current job → exits
> 4. `terminationGracePeriodSeconds` should be longer than Puma's drain time + Sidekiq's timeout

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
| Readiness Probe | "Can it serve traffic?" → remove from LB on failure; check DB, Redis, dependencies |
| Startup Probe | "Has it finished starting?" → prevents liveness probe from killing slow-starting apps |
| Graceful Shutdown | SIGTERM → stop accepting → drain in-flight → close connections → exit; enables zero-downtime deploys |

**Ruby-Specific Takeaways:**

| Area | Ruby Tools & Gems |
|------|-------------------|
| **Metrics (Prometheus)** | `prometheus-client`, `yabeda-prometheus`, `yabeda-rails`, `yabeda-sidekiq`, `yabeda-puma-plugin` |
| **Metrics (APM)** | Scout APM, Skylight, New Relic Ruby agent, Datadog `ddtrace` |
| **Structured Logging** | Lograge (Rails request logs), Semantic Logger (comprehensive), `rails_semantic_logger` |
| **Log Filtering** | `config.filter_parameters` (built-in Rails) |
| **Correlation IDs** | `ActionDispatch::RequestId` (built-in Rails), custom Faraday middleware |
| **Distributed Tracing** | `opentelemetry-sdk`, `opentelemetry-instrumentation-all` (auto-instruments Rails, AR, Faraday, Redis, Sidekiq) |
| **Error Tracking** | Sentry (`sentry-ruby`), Honeybadger, Bugsnag, Rollbar |
| **Health Checks** | Rails 7.1+ `/up` endpoint, `rails_health_check` gem, custom `HealthController` |
| **Graceful Shutdown** | Puma (native SIGTERM handling), Sidekiq (native SIGTERM + timeout) |
| **Alerting** | Prometheus Alertmanager, PagerDuty, OpsGenie, Slack integrations |

---

## Interview Tips for Module 30

1. **Three pillars** — explain metrics, logs, traces and how they work together; give the debugging flow example
2. **Metric types** — explain counter, gauge, histogram; know when to use each; reference Yabeda/prometheus-client for Ruby
3. **Prometheus** — explain the pull model; write basic PromQL queries (rate, histogram_quantile); mention the `/metrics` endpoint from Rails
4. **SLI/SLO/SLA** — explain the relationship; give concrete examples; mention error budgets
5. **Error budgets** — explain how they balance feature velocity and reliability; mention the budget policy
6. **Structured logging** — explain why JSON; mention Lograge for Rails; mention correlation IDs via `request_id`
7. **ELK vs Loki** — ELK indexes content (fast search, expensive); Loki indexes labels only (cheaper, slower content search)
8. **Distributed tracing** — draw a trace waterfall; explain trace ID, span ID, parent span ID; reference OpenTelemetry Ruby SDK
9. **OpenTelemetry** — mention as the industry standard; instrument once, export to any backend; list Ruby auto-instrumentation gems (Rails, ActiveRecord, Faraday, Sidekiq)
10. **Sampling** — explain why you can't trace everything; always sample errors; probabilistic for normal traffic
11. **Health checks** — explain liveness vs readiness vs startup; warn about checking external deps in liveness (cascading failure!); mention Rails 7.1+ `/up` endpoint
12. **Graceful shutdown** — explain the SIGTERM → drain → close → exit sequence; mention Puma's native support; explain `preStop` hook and `terminationGracePeriodSeconds` for zero-downtime deploys