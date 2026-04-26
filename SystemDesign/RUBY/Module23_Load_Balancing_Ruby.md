# Module 23: Load Balancing

> A **load balancer** distributes incoming network traffic across multiple servers to ensure no single server is overwhelmed. It's one of the most fundamental components in any scalable architecture — sitting between clients and servers, it improves availability, reliability, and performance. This module covers load balancing from basics to advanced patterns including algorithms, health checks, global load balancing, high availability, and production tools.

---

## 23.1 Load Balancer Basics

> A **load balancer** acts as a traffic cop, receiving all incoming requests and distributing them across a pool of backend servers (also called targets, upstreams, or a server farm). Clients talk to the load balancer's IP address — they never know about the individual backend servers.

> **🔹 Ruby Context:** In a typical Ruby on Rails deployment, Nginx or HAProxy sits in front of multiple Puma (or Unicorn) application server processes. Since Ruby's GIL (Global Interpreter Lock) limits true parallelism in MRI Ruby, running multiple Puma workers behind a load balancer is the standard way to achieve concurrency. A common production stack is: **Nginx → multiple Puma workers → Rails app**. Cloud deployments often use AWS ALB or NLB in front of ECS/EKS containers running Puma.

---

### What is a Load Balancer?

```
Without Load Balancer:
  Client A ──→ Server 1 (overloaded!)
  Client B ──→ Server 1 (overloaded!)
  Client C ──→ Server 1 (overloaded!)
  Server 2: idle
  Server 3: idle

With Load Balancer:
  Client A ──┐
  Client B ──┼──→ Load Balancer ──→ Server 1 (33% load)
  Client C ──┘                  ──→ Server 2 (33% load)
                                ──→ Server 3 (33% load)
```

**What a Load Balancer Does:**

| Function | Description |
|----------|-------------|
| **Traffic distribution** | Spreads requests across multiple servers |
| **Health monitoring** | Detects unhealthy servers and stops sending traffic to them |
| **High availability** | If a server fails, traffic is automatically routed to healthy servers |
| **SSL/TLS termination** | Handles encryption/decryption, offloading this work from backend servers |
| **Session persistence** | Routes requests from the same client to the same server (sticky sessions) |
| **Connection management** | Manages connection pooling, keep-alive, and connection draining |
| **Security** | Absorbs DDoS attacks, hides backend server IPs, enforces rate limits |

---

### Layer 4 (Transport) vs Layer 7 (Application) Load Balancing

Load balancers operate at different layers of the OSI model, with fundamentally different capabilities:

**Layer 4 (Transport Layer) Load Balancing:**

Operates at the TCP/UDP level. Routes traffic based on **IP addresses and port numbers** without inspecting the content of the request.

```
Client → L4 Load Balancer → Backend Server

L4 LB sees:
  Source IP: 203.0.113.50
  Dest IP: 10.0.1.100 (LB's IP)
  Source Port: 52431
  Dest Port: 443
  Protocol: TCP

L4 LB does NOT see:
  HTTP method (GET, POST)
  URL path (/api/users)
  Headers (Host, Cookie, Authorization)
  Request body
```

**How L4 Works:**
1. Client initiates a TCP connection to the load balancer's IP
2. Load balancer selects a backend server (based on algorithm)
3. Load balancer either:
   - **NAT mode:** Rewrites the destination IP to the backend server's IP, forwards the packet
   - **DSR (Direct Server Return):** Forwards the packet; backend responds directly to the client (bypassing the LB for responses — faster)
   - **Tunnel mode:** Encapsulates the packet and sends it to the backend

**Layer 7 (Application Layer) Load Balancing:**

Operates at the HTTP/HTTPS level. Can inspect the **full content** of the request — URL, headers, cookies, body — and make routing decisions based on it.

```
Client → L7 Load Balancer → Backend Server

L7 LB sees EVERYTHING:
  HTTP Method: GET
  URL: /api/users/123
  Host: api.example.com
  Headers: Authorization: Bearer <token>
  Cookies: session_id=abc123
  Body: (for POST/PUT)
```

**What L7 Can Do That L4 Cannot:**

| Capability | L4 | L7 |
|-----------|----|----|
| Route by URL path | ❌ | ✅ `/api/*` → API servers, `/static/*` → CDN |
| Route by hostname | ❌ | ✅ `api.example.com` → API, `www.example.com` → Web |
| Route by HTTP header | ❌ | ✅ Route by `Authorization`, `Accept-Language`, custom headers |
| Route by cookie | ❌ | ✅ Session affinity based on cookie value |
| Content-based routing | ❌ | ✅ Route based on request body content |
| URL rewriting | ❌ | ✅ Rewrite `/v1/users` to `/users` before forwarding |
| Request/response modification | ❌ | ✅ Add/remove headers, compress responses |
| SSL/TLS termination | ✅ (TCP passthrough) | ✅ (full termination and re-encryption) |
| WebSocket support | ✅ (TCP level) | ✅ (HTTP upgrade aware) |
| gRPC support | ✅ (TCP level) | ✅ (HTTP/2 aware, per-RPC routing) |
| Rate limiting per endpoint | ❌ | ✅ Different limits for `/api/search` vs `/api/users` |
| A/B testing | ❌ | ✅ Route 10% of traffic to canary based on header/cookie |

**L4 vs L7 Comparison:**

| Feature | Layer 4 | Layer 7 |
|---------|---------|---------|
| **Speed** | Faster (no content inspection) | Slower (must parse HTTP) |
| **Throughput** | Higher (less processing per packet) | Lower (more processing) |
| **Flexibility** | Limited (IP/port only) | Rich (URL, headers, cookies, body) |
| **SSL termination** | Pass-through or terminate | Full termination with inspection |
| **Use case** | High-throughput TCP/UDP traffic, non-HTTP protocols | HTTP/HTTPS APIs, web applications, microservices |
| **Examples** | AWS NLB, HAProxy (TCP mode) | AWS ALB, Nginx, HAProxy (HTTP mode), Envoy |

**When to Use Which:**

| Scenario | Recommendation |
|----------|---------------|
| Non-HTTP protocols (database, MQTT, gaming) | L4 |
| Maximum throughput, minimum latency | L4 |
| HTTP/HTTPS with content-based routing | L7 |
| Microservices with path-based routing | L7 |
| A/B testing, canary deployments | L7 |
| gRPC with per-RPC load balancing | L7 |
| Simple TCP load balancing | L4 |

**Common Pattern — L4 in Front of L7:**

```
Internet → L4 Load Balancer (NLB) → L7 Load Balancers (ALB/Nginx) → Backend Servers
           (handles raw TCP,          (handles HTTP routing,
            DDoS protection,           SSL termination,
            high throughput)           content-based routing)
```

> **🔹 Ruby Context:** For Rails applications, the most common pattern is an L7 load balancer (Nginx or ALB) in front of Puma workers. Nginx handles SSL termination, static asset serving, and request buffering, then proxies dynamic requests to Puma via Unix sockets or TCP. In high-traffic Rails deployments, an NLB (L4) may sit in front of multiple Nginx instances for additional scalability.

---

### Hardware vs Software Load Balancers

| Feature | Hardware LB | Software LB |
|---------|------------|-------------|
| **Form factor** | Physical appliance (rack-mounted) | Software running on commodity servers or VMs |
| **Cost** | $10,000 - $100,000+ | Free (open-source) or pay-per-use (cloud) |
| **Performance** | Very high (custom ASICs) | High (but limited by server hardware) |
| **Flexibility** | Limited (vendor-specific features) | Highly flexible (open-source, customizable) |
| **Scaling** | Buy bigger appliance | Add more instances |
| **Deployment** | Data center only | Anywhere (cloud, on-prem, edge) |
| **Examples** | F5 BIG-IP, Citrix ADC, A10 Networks | Nginx, HAProxy, Envoy, AWS ALB/NLB |
| **Trend** | Declining (legacy) | Dominant (cloud-native) |

**Key Point:** Software load balancers have largely replaced hardware load balancers. Cloud-managed load balancers (AWS ALB/NLB, GCP Cloud Load Balancing) are the most common choice today.

---


## 23.2 Load Balancing Algorithms

> The load balancing algorithm determines **which backend server** receives each incoming request. The right algorithm depends on your workload characteristics — whether servers are homogeneous, whether requests have varying costs, and whether session affinity is needed.

---

### Round Robin

Distributes requests sequentially across servers in a circular order. The simplest algorithm.

```
Servers: A, B, C

Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (back to the beginning)
Request 5 → Server B
Request 6 → Server C
...
```

| Pros | Cons |
|------|------|
| Simplest to implement | Ignores server capacity (all servers treated equally) |
| Even distribution (if servers are identical) | Ignores current server load |
| No state needed | Long-running requests can cause uneven load |
| Predictable | Not suitable for heterogeneous servers |

**Best for:** Homogeneous servers with similar request costs.

---

### Weighted Round Robin

Like round robin, but servers with higher weights receive proportionally more requests.

```
Servers: A (weight=5), B (weight=3), C (weight=2)
Total weight: 10

Distribution: A gets 50%, B gets 30%, C gets 20%

Request sequence: A, A, A, A, A, B, B, B, C, C, A, A, A, A, A, B, B, B, C, C, ...
```

| Pros | Cons |
|------|------|
| Accounts for different server capacities | Still ignores current load |
| Simple to configure | Weights must be manually set and updated |
| Good for heterogeneous servers | Doesn't adapt to changing conditions |

**Best for:** Servers with different capacities (e.g., mix of large and small instances). Also useful for **canary deployments** — give the canary server a weight of 1 and production servers a weight of 99.

---

### Least Connections

Routes each request to the server with the **fewest active connections**. Adapts to varying request durations.

```
Server A: 15 active connections
Server B: 8 active connections   ← next request goes here (fewest)
Server C: 12 active connections

New request → Server B (8 connections, now 9)
```

| Pros | Cons |
|------|------|
| Adapts to varying request durations | Slightly more overhead (must track connection counts) |
| Naturally balances slow and fast servers | Doesn't account for server capacity differences |
| Good for long-lived connections (WebSocket, database) | New server gets all traffic initially (0 connections) |

**Best for:** Workloads with varying request durations (some requests take 10ms, others take 5 seconds).

> **🔹 Ruby Context:** Least connections is particularly useful for Rails apps where request durations vary widely — e.g., a fast index page vs. a slow report generation endpoint. Nginx's `least_conn` directive works well with Puma upstreams for this pattern.

---

### Weighted Least Connections

Combines least connections with server weights. Routes to the server with the lowest `connections / weight` ratio.

```
Server A: 15 connections, weight=5 → ratio = 15/5 = 3.0
Server B: 8 connections, weight=3  → ratio = 8/3 = 2.67  ← lowest ratio, gets next request
Server C: 12 connections, weight=2 → ratio = 12/2 = 6.0
```

**Best for:** Heterogeneous servers with varying request durations.

---

### IP Hash

Hashes the client's IP address to determine which server receives the request. The same client IP always goes to the same server.

```
hash(client_ip) % num_servers = server_index

hash("203.0.113.50") % 3 = 1 → Server B (always)
hash("198.51.100.25") % 3 = 0 → Server A (always)
hash("192.0.2.100")   % 3 = 2 → Server C (always)
```

| Pros | Cons |
|------|------|
| Session affinity without cookies | Uneven distribution if some IPs generate more traffic |
| No state needed (deterministic) | Adding/removing servers changes all mappings |
| Simple | Clients behind NAT share an IP → all go to same server |

**Best for:** Simple session affinity when you can't use cookies. Not recommended for production — use cookie-based affinity or external session stores instead.

> **🔹 Ruby Context:** Rails applications typically avoid IP hash in favor of cookie-based sessions backed by Redis or Memcached (via `ActionDispatch::Session::CacheStore` or the `redis-session-store` gem). This decouples session state from server affinity entirely, which is the recommended pattern for Rails deployments.

---

### Consistent Hashing

Maps both servers and requests onto a hash ring. Each request is routed to the nearest server clockwise on the ring. Adding or removing a server only affects ~1/N of requests.

```
Hash Ring:
        Server A (pos 0)
       /                \
  Server D (pos 270)   Server B (pos 90)
       \                /
        Server C (pos 180)

Request hash = 75 → clockwise → Server B
Request hash = 200 → clockwise → Server D

Adding Server E at position 135:
  Only requests between 90-135 move from Server C to Server E
  All other requests stay on their current server
```

| Pros | Cons |
|------|------|
| Minimal disruption when adding/removing servers | More complex to implement |
| Good for distributed caches (minimize cache misses on scaling) | Can be uneven without virtual nodes |
| Deterministic routing | Requires consistent hashing library |

**Best for:** Distributed caches (Redis, Memcached), stateful services where you want to minimize redistribution when scaling.

> **🔹 Ruby Context:** The `hash_ring` gem or `consistent_hashing` gem provides consistent hashing for Ruby applications. The `dalli` gem (Memcached client for Ruby) uses consistent hashing internally to distribute keys across multiple Memcached servers. Redis Cluster also uses a form of hash slots for key distribution, accessible via the `redis-clustering` gem.

---

### Random

Selects a server at random for each request.

```
Request 1 → random(A, B, C) → Server C
Request 2 → random(A, B, C) → Server A
Request 3 → random(A, B, C) → Server A
Request 4 → random(A, B, C) → Server B
```

| Pros | Cons |
|------|------|
| Simplest possible implementation | No guarantee of even distribution (short-term) |
| No state needed | Ignores server load and capacity |
| Statistically even over large numbers | Can be unlucky (all requests to one server) |

**Best for:** Large server pools where statistical averaging ensures even distribution. Sometimes used as a tiebreaker in more complex algorithms.

---

### Least Response Time

Routes to the server with the **lowest average response time** AND fewest active connections. Combines latency awareness with connection awareness.

```
Server A: avg response time = 50ms, 10 connections
Server B: avg response time = 20ms, 15 connections  ← fastest response time
Server C: avg response time = 80ms, 5 connections

Score = response_time × active_connections
Server A: 50 × 10 = 500
Server B: 20 × 15 = 300  ← lowest score, gets next request
Server C: 80 × 5 = 400
```

| Pros | Cons |
|------|------|
| Routes to the fastest server | Must continuously measure response times |
| Adapts to server performance changes | Measurement overhead |
| Good for heterogeneous environments | Cold servers may get all traffic initially |

**Best for:** When backend servers have varying performance characteristics (different hardware, different workloads, different geographic locations).

---

### Resource-Based (Adaptive)

The load balancer queries backend servers for their current resource utilization (CPU, memory, connections) and routes to the least loaded server.

```
Server A reports: CPU=80%, Memory=60%, Connections=150
Server B reports: CPU=30%, Memory=40%, Connections=50   ← least loaded
Server C reports: CPU=50%, Memory=70%, Connections=100

Next request → Server B
```

**Implementation:** Backend servers expose a health/status endpoint that reports resource metrics. The load balancer periodically polls this endpoint.

```
GET /health
{
  "status": "healthy",
  "cpu_percent": 30,
  "memory_percent": 40,
  "active_connections": 50,
  "request_queue_depth": 2
}
```

> **🔹 Ruby Context:** In Rails, you can implement a health endpoint using a simple controller or the `rails-health-check` gem. Puma also exposes runtime stats (thread pool usage, backlog) via its control app, which can be queried for adaptive load balancing decisions:

```ruby
# config/routes.rb
get '/health', to: 'health#show'

# app/controllers/health_controller.rb
class HealthController < ApplicationController
  # Skip authentication for health checks
  skip_before_action :authenticate_user!, if: -> { defined?(authenticate_user!) }

  def show
    checks = {
      status: 'healthy',
      database: database_healthy?,
      redis: redis_healthy?,
      puma_stats: puma_stats
    }

    if checks[:database] && checks[:redis]
      render json: checks, status: :ok
    else
      render json: checks.merge(status: 'unhealthy'), status: :service_unavailable
    end
  end

  private

  def database_healthy?
    ActiveRecord::Base.connection.execute('SELECT 1')
    true
  rescue StandardError
    false
  end

  def redis_healthy?
    Redis.current.ping == 'PONG'
  rescue StandardError
    false
  end

  def puma_stats
    # Puma exposes stats via its control app
    stats = Puma.stats rescue nil
    return {} unless stats

    parsed = JSON.parse(stats)
    {
      workers: parsed['workers'],
      booted_workers: parsed['booted_workers'],
      backlog: parsed['backlog'],
      running: parsed['running'],
      pool_capacity: parsed['pool_capacity'],
      max_threads: parsed['max_threads']
    }
  rescue StandardError
    {}
  end
end
```

| Pros | Cons |
|------|------|
| Most accurate load distribution | Requires health endpoint on every server |
| Adapts to real-time conditions | Polling overhead |
| Accounts for all resource types | More complex to implement |

**Best for:** Environments where server load varies significantly and you need the most accurate distribution.

---

### Algorithm Comparison Summary

| Algorithm | Complexity | State Needed | Adapts to Load | Session Affinity | Best For |
|-----------|-----------|-------------|----------------|-----------------|----------|
| **Round Robin** | Lowest | None | No | No | Homogeneous servers, simple setup |
| **Weighted Round Robin** | Low | Weights | No | No | Heterogeneous servers, canary deploys |
| **Least Connections** | Medium | Connection counts | Yes | No | Varying request durations |
| **Weighted Least Conn** | Medium | Counts + weights | Yes | No | Heterogeneous + varying durations |
| **IP Hash** | Low | None | No | Yes (IP-based) | Simple session affinity |
| **Consistent Hashing** | Medium | Hash ring | No | Yes (key-based) | Distributed caches, stateful services |
| **Random** | Lowest | None | No | No | Large pools, simplicity |
| **Least Response Time** | High | Response times | Yes | No | Performance-sensitive workloads |
| **Resource-Based** | Highest | Server metrics | Yes | No | Maximum accuracy, complex environments |

---


## 23.3 Health Checks

> Health checks allow the load balancer to detect unhealthy backend servers and stop sending traffic to them. Without health checks, the load balancer would continue routing requests to crashed or overloaded servers, causing errors for users.

---

### Active Health Checks

The load balancer **proactively sends requests** to backend servers at regular intervals to verify they're healthy.

```
Load Balancer → Server A: GET /health → 200 OK ✅ (healthy)
Load Balancer → Server B: GET /health → 200 OK ✅ (healthy)
Load Balancer → Server C: GET /health → timeout ❌ (unhealthy!)

→ Remove Server C from the pool
→ Route all traffic to Server A and Server B

Later:
Load Balancer → Server C: GET /health → 200 OK ✅ (recovered!)
→ Add Server C back to the pool
```

**Health Check Configuration:**

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| **Interval** | How often to check each server | 5-30 seconds |
| **Timeout** | Max time to wait for a response | 2-5 seconds |
| **Unhealthy threshold** | Consecutive failures before marking unhealthy | 2-3 failures |
| **Healthy threshold** | Consecutive successes before marking healthy again | 2-3 successes |
| **Path** | HTTP endpoint to check | `/health`, `/healthz`, `/status` |
| **Expected status** | HTTP status code that indicates healthy | 200 |
| **Protocol** | Health check protocol | HTTP, HTTPS, TCP |

**Types of Active Health Checks:**

| Type | How It Works | What It Verifies |
|------|-------------|-----------------|
| **TCP check** | Open a TCP connection to the server | Server is reachable and accepting connections |
| **HTTP check** | Send an HTTP GET request, check status code | Application is running and responding |
| **Deep health check** | HTTP check that also verifies dependencies (DB, cache) | Application AND its dependencies are healthy |
| **Script-based** | Run a custom script on the server | Custom health logic |

**Shallow vs Deep Health Checks:**

```
Shallow Health Check (GET /health):
  → Returns 200 if the application process is running
  → Does NOT check database, cache, or external dependencies
  → Fast (< 1ms)
  → Use for: liveness checks (is the process alive?)

Deep Health Check (GET /health/ready):
  → Checks database connectivity
  → Checks Redis connectivity
  → Checks external API availability
  → Checks disk space
  → Returns 200 only if ALL dependencies are healthy
  → Slower (10-100ms)
  → Use for: readiness checks (can this server handle traffic?)
```

```json
// Deep health check response
{
  "status": "healthy",
  "checks": {
    "database": { "status": "healthy", "latency_ms": 2 },
    "redis": { "status": "healthy", "latency_ms": 1 },
    "disk": { "status": "healthy", "free_gb": 45 },
    "external_api": { "status": "degraded", "latency_ms": 500 }
  },
  "uptime_seconds": 86400
}
```

**Warning:** Deep health checks can cause cascading failures. If the database is slow, ALL servers report unhealthy, and the load balancer removes ALL servers from the pool → total outage. Use deep checks for readiness (should this server receive traffic?) and shallow checks for liveness (is this server alive?).

> **🔹 Ruby Context:** Rails 7.1+ includes a built-in health check endpoint at `/up` (via `Rails::HealthController`). For more sophisticated checks, the `health_check` gem or `okcomputer` gem provides configurable health check endpoints that can verify database, Redis, Sidekiq, and custom dependencies. These integrate naturally with Nginx or ALB health check configurations.

---

### Passive Health Checks

The load balancer monitors **actual traffic** to detect unhealthy servers — no separate health check requests needed.

```
Load Balancer observes:
  Server A: 500 requests, 2 errors (0.4% error rate) → healthy
  Server B: 500 requests, 250 errors (50% error rate) → UNHEALTHY!
  Server C: 500 requests, 5 errors (1% error rate) → healthy

→ Remove Server B from the pool (too many errors)
```

**Passive Health Check Signals:**

| Signal | Unhealthy Indicator |
|--------|-------------------|
| **HTTP 5xx responses** | Server returning 500, 502, 503 errors |
| **Connection failures** | TCP connection refused or timed out |
| **Response timeouts** | Server didn't respond within the timeout |
| **Connection resets** | Server abruptly closed the connection |

**Active vs Passive Health Checks:**

| Feature | Active | Passive |
|---------|--------|---------|
| Detection method | Periodic probe requests | Monitor actual traffic |
| Detection speed | Depends on interval (5-30s) | Immediate (detects on real traffic) |
| Overhead | Extra requests to servers | No extra requests |
| Can detect idle server issues | Yes (checks even with no traffic) | No (needs traffic to detect) |
| False positives | Possible (health endpoint works but app is broken) | Less likely (based on real traffic) |

**Best Practice:** Use **both** active and passive health checks together. Active checks detect servers that are completely down. Passive checks detect servers that are up but returning errors.

---

### Graceful Degradation

When a server is marked unhealthy, the load balancer should handle the transition gracefully:

**Connection Draining (Deregistration Delay):**

```
Server B is being removed (unhealthy or scaling down):

1. Stop sending NEW requests to Server B
2. Allow EXISTING in-flight requests to complete (drain period: 30-300 seconds)
3. After drain period, forcefully close remaining connections
4. Remove Server B from the pool

Without draining:
  Server B removed immediately → in-flight requests get connection reset → errors!

With draining:
  Server B stops receiving new requests → existing requests finish normally → clean removal
```

**Graceful Shutdown Sequence:**

```
1. Load balancer marks server as "draining" (no new requests)
2. Server receives SIGTERM signal
3. Server stops accepting new connections
4. Server finishes processing in-flight requests
5. Server closes database connections, flushes logs
6. Server exits cleanly
7. Load balancer removes server from pool
```

> **🔹 Ruby Context:** Puma handles graceful shutdown natively. When Puma receives `SIGTERM`, it stops accepting new connections, waits for in-flight requests to complete (configurable via `worker_shutdown_timeout`), and then exits. This integrates well with Kubernetes' `terminationGracePeriodSeconds` and ALB deregistration delay. Unicorn also supports graceful restarts via `SIGUSR2` (fork new master) and `SIGQUIT` (graceful shutdown of old workers). Sidekiq similarly handles `SIGTERM` by finishing current jobs before shutting down.

```ruby
# config/puma.rb — Graceful shutdown configuration
workers ENV.fetch('WEB_CONCURRENCY', 2).to_i
threads_count = ENV.fetch('RAILS_MAX_THREADS', 5).to_i
threads threads_count, threads_count

# Time to wait for workers to finish before force-killing
worker_shutdown_timeout 30

# Preload app for faster worker boot (copy-on-write)
preload_app!

on_worker_boot do
  # Re-establish database connections in forked workers
  ActiveRecord::Base.establish_connection
end

before_fork do
  # Close parent connections before forking
  ActiveRecord::Base.connection_pool.disconnect!
end
```

**Retry on Failure:**

```
Client → Load Balancer → Server A → 502 Bad Gateway
Load Balancer retries → Server B → 200 OK → return to client

The client never sees the error — the load balancer transparently retried.
```

**Retry Configuration:**
- Retry on: 502, 503, 504, connection errors, timeouts
- Max retries: 1-2 (avoid retry storms)
- Retry on different server (not the same one that failed)
- **Idempotent requests only** — never retry POST/PUT that might cause duplicates

---


## 23.4 Advanced Load Balancing

> Beyond basic load balancing within a single data center, modern systems need global traffic management, service mesh integration, and client-side load balancing for microservices.

---

### Global Server Load Balancing (GSLB)

**GSLB** distributes traffic across multiple data centers or regions worldwide. It operates at the DNS level, routing users to the nearest or healthiest data center.

```
User in Tokyo → GSLB → Asia-Pacific Data Center (lowest latency)
User in London → GSLB → Europe Data Center (lowest latency)
User in New York → GSLB → US East Data Center (lowest latency)

If US East Data Center fails:
  User in New York → GSLB → US West Data Center (failover)
```

```
                    +------------------+
                    |      GSLB        |
                    | (DNS-based)      |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v------+ +----v-------+ +----v-------+
     | US East DC    | | EU West DC | | APAC DC    |
     |  +---------+  | |  +-------+ | |  +-------+ |
     |  | L7 LB   |  | |  | L7 LB| | |  | L7 LB| |
     |  +----+----+  | |  +---+---+ | |  +---+---+ |
     |  | Servers |   | |  |Servers| | |  |Servers| |
     |  +---------+  | |  +-------+ | |  +-------+ |
     +---------------+ +------------+ +------------+
```

**GSLB Routing Methods:**

| Method | Description | Use Case |
|--------|-------------|----------|
| **Geographic** | Route based on client's geographic location | Data residency compliance (GDPR) |
| **Latency-based** | Route to the data center with lowest measured latency | Best user experience |
| **Weighted** | Distribute traffic by percentage across data centers | Gradual migration, capacity management |
| **Failover** | Primary data center with automatic failover to secondary | Disaster recovery |
| **Health-based** | Route only to healthy data centers | Automatic failure handling |

---

### DNS-Based Load Balancing

The simplest form of global load balancing — return different IP addresses from DNS based on routing policy.

```
DNS Query: api.example.com

Response (round-robin):
  A record: 54.1.2.3   (US East)
  A record: 35.4.5.6   (EU West)
  A record: 13.7.8.9   (APAC)
  
  Client connects to the first IP in the list.
  DNS rotates the order for each query.
```

**Limitations of DNS-Based Load Balancing:**
- **No health checks** (basic DNS) — returns IPs of dead servers
- **TTL caching** — clients cache DNS for minutes; changes are slow to propagate
- **No real-time adaptation** — can't react to sudden load changes
- **Coarse-grained** — routes at the data center level, not the server level

**Managed DNS with Health Checks (AWS Route 53, Cloudflare):**
- Health checks monitor each endpoint
- Unhealthy endpoints are automatically removed from DNS responses
- Supports latency-based, geographic, weighted, and failover routing
- Still limited by DNS TTL caching

---

### Anycast

**Anycast** assigns the **same IP address** to multiple servers in different locations. The network (BGP routing) automatically routes each client to the **nearest** server.

```
Anycast IP: 1.2.3.4

Server in US East: announces 1.2.3.4 via BGP
Server in EU West: announces 1.2.3.4 via BGP
Server in APAC:    announces 1.2.3.4 via BGP

Client in Tokyo → network routes to APAC server (nearest via BGP)
Client in London → network routes to EU West server (nearest via BGP)
Client in New York → network routes to US East server (nearest via BGP)
```

**How Anycast Differs from Unicast:**

| Feature | Unicast | Anycast |
|---------|---------|---------|
| IP assignment | One IP = one server | One IP = many servers |
| Routing | Always goes to the same server | Goes to the nearest server (BGP) |
| Failover | Must change DNS/IP | Automatic (BGP withdraws route) |
| Use case | Most services | CDN, DNS, DDoS protection |

**Anycast Use Cases:**
- **CDN edge servers** — Cloudflare, Akamai use anycast to route users to the nearest edge
- **DNS servers** — Root DNS servers use anycast (13 root server IPs, hundreds of actual servers)
- **DDoS protection** — Anycast distributes attack traffic across all locations, diluting the impact

**Anycast Limitation:** Anycast works well for **stateless, short-lived connections** (DNS queries, CDN requests). For **stateful, long-lived connections** (WebSocket, TCP sessions), BGP route changes can cause connections to be routed to a different server mid-session.

---

### Service Mesh Load Balancing (Envoy, Istio)

In a microservices architecture, a **service mesh** handles load balancing between services using **sidecar proxies** deployed alongside each service instance.

```
Traditional Load Balancing:
  Service A → Central Load Balancer → Service B instances

Service Mesh Load Balancing:
  Service A → Sidecar Proxy (Envoy) → Sidecar Proxy (Envoy) → Service B
              (runs alongside A)       (runs alongside B)

Every service instance has its own sidecar proxy.
The sidecar handles: load balancing, retries, circuit breaking, mTLS, observability.
```

```
+------------------+          +------------------+
| Pod A            |          | Pod B-1          |
| +------+ +----+ |          | +------+ +----+  |
| |Svc A | |Envoy|----────────|Envoy | |Svc B|  |
| +------+ +----+ |          | +------+ +----+  |
+------------------+          +------------------+
                              +------------------+
                              | Pod B-2          |
                              | +------+ +----+  |
                    ──────────|Envoy | |Svc B|  |
                              | +------+ +----+  |
                              +------------------+
```

**Service Mesh Load Balancing Features:**

| Feature | Description |
|---------|-------------|
| **Client-side LB** | Each sidecar maintains a list of healthy endpoints and load balances locally |
| **Circuit breaking** | Stop sending traffic to a failing service; return fallback |
| **Retries** | Automatically retry failed requests (with budget limits) |
| **Timeouts** | Enforce request timeouts per service |
| **mTLS** | Mutual TLS between all services (zero-trust networking) |
| **Observability** | Distributed tracing, metrics, access logs — all from the sidecar |
| **Traffic splitting** | Route 5% to canary, 95% to stable (for canary deployments) |
| **Fault injection** | Inject delays or errors for chaos testing |

**Service Mesh Solutions:**

| Solution | Sidecar Proxy | Control Plane | Key Feature |
|----------|--------------|--------------|-------------|
| **Istio** | Envoy | istiod | Most feature-rich, complex |
| **Linkerd** | linkerd2-proxy (Rust) | linkerd-control-plane | Lightweight, simple, fast |
| **Consul Connect** | Envoy or built-in | Consul | Integrates with HashiCorp ecosystem |
| **AWS App Mesh** | Envoy | AWS managed | AWS-native |

> **🔹 Ruby Context:** Ruby/Rails microservices deployed on Kubernetes benefit from service meshes like Istio or Linkerd. The sidecar proxy handles all the cross-cutting concerns (retries, circuit breaking, mTLS) transparently — the Rails app just makes normal HTTP calls. This is especially valuable in Ruby where you might not want to add circuit breaker gems (like `circuitbox` or `stoplight`) to every service. The mesh handles it at the infrastructure layer instead.

---

### Client-Side Load Balancing

Instead of a central load balancer, the **client** (calling service) maintains a list of server endpoints and selects one for each request.

```
Central Load Balancer:
  Client → Load Balancer → Server (extra network hop)

Client-Side Load Balancing:
  Client → Server (direct connection, no extra hop)
  
  Client maintains:
    Server A: 10.0.1.1:8080 (healthy, latency=5ms)
    Server B: 10.0.1.2:8080 (healthy, latency=8ms)
    Server C: 10.0.1.3:8080 (unhealthy, removed)
  
  Client selects Server A (lowest latency) and connects directly.
```

**How Client Gets Server List:**
- **Service discovery** — query Consul, etcd, or Kubernetes DNS for current endpoints
- **DNS** — resolve the service name to multiple A records
- **Static configuration** — hardcoded list (not recommended for dynamic environments)

**Client-Side LB Advantages:**
- No single point of failure (no central LB)
- Lower latency (no extra network hop)
- Per-client load balancing decisions (can use local metrics)

**Client-Side LB Disadvantages:**
- Every client must implement LB logic (or use a library)
- Harder to update LB algorithm (must update all clients)
- Client must handle service discovery and health checking

**gRPC Client-Side Load Balancing:**
gRPC has built-in client-side load balancing:
- `pick_first` — connect to the first available server
- `round_robin` — rotate across all servers
- Custom: implement your own balancer (e.g., weighted round robin based on server metrics)

> **🔹 Ruby Context:** For Ruby gRPC services (via the `grpc` gem), client-side load balancing is available through gRPC's built-in name resolution and load balancing policies. For HTTP-based Ruby microservices, gems like `faraday` with custom middleware can implement client-side load balancing by resolving service endpoints from Consul or Kubernetes DNS and rotating between them.

---


## 23.5 High Availability

> **High availability (HA)** means the system continues to operate even when components fail. For load balancers — which are the entry point for all traffic — HA is critical. A load balancer failure without HA means a total system outage.

---

### Active-Passive (Failover)

One load balancer is **active** (handling all traffic). A second is **passive** (standby), ready to take over if the active one fails.

```
Normal Operation:
  Internet → Active LB (handling traffic) → Backend Servers
             Passive LB (standby, monitoring active)

Failover (Active LB fails):
  Internet → Passive LB (now active, takes over) → Backend Servers
             Failed LB (being repaired)

Detection: Passive LB sends heartbeats to Active LB.
           If heartbeats stop → Passive takes over the Virtual IP.
```

```
  +------------------+     Heartbeat     +------------------+
  | Active LB        | ←──────────────→ | Passive LB       |
  | VIP: 10.0.0.100  |                  | (standby)        |
  +--------+---------+                  +--------+---------+
           |                                      |
           ↓                                      |
    Backend Servers                                |
                                                   |
  Active LB fails:                                 |
    Passive LB claims VIP: 10.0.0.100 ←───────────┘
    Traffic now flows through Passive LB (now Active)
```

| Pros | Cons |
|------|------|
| Simple to set up | Passive LB is wasted capacity (idle until failover) |
| Clear failover behavior | Brief downtime during failover (seconds) |
| No split-brain risk | Only 1x capacity (active handles all traffic) |

**Failover Time:** Typically 5-30 seconds (heartbeat detection + VIP migration + connection re-establishment).

---

### Active-Active

**Both** load balancers are active and handling traffic simultaneously. Traffic is distributed between them (typically via DNS round-robin or anycast).

```
Normal Operation:
  Internet → DNS returns both IPs
    50% traffic → Active LB 1 → Backend Servers
    50% traffic → Active LB 2 → Backend Servers

LB 1 Fails:
  DNS health check removes LB 1's IP
  100% traffic → Active LB 2 → Backend Servers
```

```
  +------------------+                  +------------------+
  | Active LB 1      |                  | Active LB 2      |
  | IP: 10.0.0.101   |                  | IP: 10.0.0.102   |
  +--------+---------+                  +--------+---------+
           |                                      |
           +──────────────┬───────────────────────+
                          |
                   Backend Servers
```

| Pros | Cons |
|------|------|
| 2x capacity (both LBs handle traffic) | More complex configuration |
| No wasted resources | Must handle session synchronization |
| Faster failover (other LB already active) | Potential split-brain if LBs disagree on backend health |
| Better resource utilization | Need DNS or another mechanism to distribute between LBs |

---

### Floating IP / Virtual IP (VIP)

A **Virtual IP (VIP)** is an IP address that can be moved between servers. It's the mechanism that enables failover — the VIP always points to the active load balancer.

```
VIP: 10.0.0.100 (this is what clients connect to)

Normal:
  VIP 10.0.0.100 → assigned to LB 1 (active)
  LB 2 (passive) monitors LB 1

Failover:
  LB 1 fails → LB 2 claims VIP 10.0.0.100
  Clients still connect to 10.0.0.100 → now reaches LB 2
  No DNS change needed — same IP, different server
```

**How VIP Failover Works:**
1. Active LB holds the VIP (responds to ARP requests for that IP)
2. Passive LB sends heartbeats to Active LB
3. If heartbeats stop (Active LB is down):
   - Passive LB sends a **Gratuitous ARP** announcing it now owns the VIP
   - Network switches update their ARP tables
   - Traffic to the VIP now reaches the Passive LB
4. Failover complete — typically 1-5 seconds

**Tools for VIP Management:**
- **Keepalived** — VRRP (Virtual Router Redundancy Protocol) implementation for Linux
- **Pacemaker/Corosync** — Cluster resource manager for HA
- **AWS Elastic IP** — Cloud-native VIP (can be reassigned between instances via API)
- **GCP Forwarding Rules** — Cloud-native traffic routing

**Keepalived Configuration Example:**

```
# /etc/keepalived/keepalived.conf on LB 1 (master)
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100              # higher priority = master
    advert_int 1              # heartbeat every 1 second
    
    authentication {
        auth_type PASS
        auth_pass secret123
    }
    
    virtual_ipaddress {
        10.0.0.100/24         # the VIP
    }
    
    track_script {
        check_nginx           # monitor nginx health
    }
}

vrrp_script check_nginx {
    script "/usr/bin/pgrep nginx"
    interval 2                # check every 2 seconds
    weight -20                # reduce priority by 20 if nginx is down
}
```

> **🔹 Ruby Context:** In a typical Rails HA deployment, Keepalived manages VIP failover between two Nginx instances, each of which proxies to Puma workers. On cloud platforms (AWS, GCP), managed load balancers (ALB/NLB) handle HA automatically — you don't need to manage VIPs or Keepalived yourself. For on-premises Rails deployments, the Keepalived + Nginx + Puma stack is a proven HA pattern.

---

### Redundancy at Every Layer

High availability requires redundancy at **every layer** of the stack — not just the load balancer.

```
Fully Redundant Architecture:

  DNS: Multiple DNS providers (Route 53 + Cloudflare)
    ↓
  CDN: Multiple edge locations (CloudFront)
    ↓
  Load Balancers: Active-Active pair (2+ LBs)
    ↓
  Application Servers: Multiple instances (3+ servers, auto-scaling)
    ↓
  Cache: Redis Cluster with replicas (3 masters + 3 replicas)
    ↓
  Database: Primary + Sync Replica + Async Replicas (multi-AZ)
    ↓
  Object Storage: S3 (11 nines durability, cross-region replication)
```

> **🔹 Ruby Context:** A fully redundant Rails architecture typically looks like: **Route 53 → CloudFront (assets) → ALB → Auto Scaling Group of Puma instances → ElastiCache Redis (for caching + Sidekiq) → RDS PostgreSQL (primary + replica) → S3 (ActiveStorage)**. Tools like Capistrano, Kamal (formerly MRSK), or Kubernetes handle zero-downtime deployments across multiple app servers.

**Availability Math:**

```
Single component availability: 99.9% (8.76 hours downtime/year)

Serial components (all must work):
  LB (99.9%) × App (99.9%) × DB (99.9%) = 99.7% (26.3 hours downtime/year)

With redundancy (parallel components):
  LB pair: 1 - (1-0.999)² = 99.9999% 
  App cluster (3): 1 - (1-0.999)³ = 99.999999%
  DB pair: 1 - (1-0.999)² = 99.9999%
  
  Combined: 99.9999% × 99.999999% × 99.9999% ≈ 99.9998% (< 1 minute downtime/year)
```

**Availability Targets:**

| Availability | Downtime/Year | Downtime/Month | Common Name |
|-------------|--------------|----------------|-------------|
| 99% | 3.65 days | 7.3 hours | Two nines |
| 99.9% | 8.76 hours | 43.8 minutes | Three nines |
| 99.95% | 4.38 hours | 21.9 minutes | Three and a half nines |
| 99.99% | 52.6 minutes | 4.38 minutes | Four nines |
| 99.999% | 5.26 minutes | 26.3 seconds | Five nines |
| 99.9999% | 31.5 seconds | 2.63 seconds | Six nines |

**Most SLAs target 99.9% to 99.99%.** Five nines (99.999%) requires significant investment in redundancy, automation, and operational excellence.

---


## 23.6 Load Balancer Tools

> Choosing the right load balancer depends on your protocol requirements, scale, cloud provider, and operational preferences. This section covers the most popular load balancer tools and when to use each.

---

### Nginx

**Nginx** is the most popular open-source web server and reverse proxy, widely used as an L7 load balancer.

| Feature | Details |
|---------|---------|
| **Type** | Web server + reverse proxy + L7 load balancer |
| **Protocols** | HTTP/1.1, HTTP/2, WebSocket, gRPC, TCP/UDP (stream module) |
| **Algorithms** | Round robin, weighted, least connections, IP hash, random with two choices |
| **Health checks** | Passive (open-source), active (Nginx Plus) |
| **SSL/TLS** | Full termination, SNI support, OCSP stapling |
| **Caching** | Built-in response caching with configurable TTL |
| **Rate limiting** | Per-IP, per-endpoint rate limiting |
| **Static files** | Excellent static file serving |
| **Config** | Static config file, reload without downtime (`nginx -s reload`) |
| **Best for** | Web applications, API reverse proxy, static file serving |

**Nginx Plus** (commercial) adds: active health checks, session persistence, dynamic reconfiguration API, JWT authentication, advanced monitoring.

> **🔹 Ruby Context:** Nginx is the most common reverse proxy for Rails applications. A typical Nginx + Puma configuration proxies requests via Unix sockets for minimal overhead:

```nginx
# /etc/nginx/sites-available/myapp
upstream puma_backend {
    # Unix socket connection to Puma (lowest latency)
    server unix:///var/www/myapp/shared/tmp/sockets/puma.sock;
    
    # Or TCP with multiple Puma instances for load balancing:
    # server 127.0.0.1:3000;
    # server 127.0.0.1:3001;
    # server 127.0.0.1:3002;
    
    # Use least connections for varying request durations
    # least_conn;
}

server {
    listen 80;
    server_name myapp.com;
    root /var/www/myapp/current/public;

    # Serve static assets directly (bypass Puma)
    location ^~ /assets/ {
        gzip_static on;
        expires max;
        add_header Cache-Control public;
    }

    # Proxy dynamic requests to Puma
    location / {
        proxy_pass http://puma_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

### HAProxy

**HAProxy** is a high-performance, open-source load balancer focused on reliability and throughput.

| Feature | Details |
|---------|---------|
| **Type** | L4/L7 load balancer and proxy |
| **Protocols** | HTTP/1.1, HTTP/2, TCP, WebSocket |
| **Algorithms** | Round robin, weighted, least connections, source hash, URI hash, random |
| **Health checks** | Active (HTTP, TCP, script-based) + passive |
| **SSL/TLS** | Full termination |
| **Connection management** | Excellent connection pooling and multiplexing |
| **Stats** | Built-in stats dashboard with real-time metrics |
| **Config** | Static config file, runtime API for dynamic changes |
| **Performance** | Handles millions of connections; used by GitHub, Stack Overflow, Reddit |
| **Best for** | High-performance TCP/HTTP load balancing, database proxying |

> **🔹 Ruby Context:** HAProxy is used by several large-scale Rails deployments (GitHub famously uses HAProxy). It's particularly useful when you need active health checks (free, unlike Nginx Plus), advanced load balancing algorithms, or TCP-level load balancing for non-HTTP services like database read replicas. HAProxy can also load balance across multiple Puma instances with more granular control than Nginx's upstream module.

**HAProxy vs Nginx:**

| Feature | Nginx | HAProxy |
|---------|-------|---------|
| Primary role | Web server + proxy | Load balancer + proxy |
| Static file serving | Excellent | Not supported |
| Load balancing | Good | Excellent |
| Health checks (free) | Passive only | Active + passive |
| Stats/monitoring | Basic (access logs) | Excellent (built-in dashboard) |
| Configuration | Simpler | More complex but more powerful |
| TCP load balancing | Supported (stream module) | Native, excellent |
| Best for | Web serving + simple LB | Dedicated high-performance LB |

---

### AWS Elastic Load Balancing (ALB, NLB, CLB)

AWS offers three managed load balancer types:

**ALB (Application Load Balancer) — Layer 7:**

| Feature | Details |
|---------|---------|
| **Layer** | 7 (HTTP/HTTPS) |
| **Routing** | Path-based, host-based, header-based, query string-based |
| **Targets** | EC2 instances, ECS containers, Lambda functions, IP addresses |
| **Health checks** | HTTP/HTTPS with configurable path and thresholds |
| **SSL/TLS** | Full termination, SNI for multiple certificates |
| **WebSocket** | Supported |
| **gRPC** | Supported |
| **Sticky sessions** | Cookie-based (application or duration-based) |
| **Authentication** | Built-in OIDC/Cognito authentication |
| **WAF integration** | AWS WAF for web application firewall |
| **Best for** | HTTP/HTTPS microservices, container-based applications |

**NLB (Network Load Balancer) — Layer 4:**

| Feature | Details |
|---------|---------|
| **Layer** | 4 (TCP/UDP/TLS) |
| **Performance** | Millions of requests/sec, ultra-low latency (~100μs) |
| **Static IP** | Supports Elastic IPs (static IP per AZ) |
| **Targets** | EC2 instances, IP addresses, ALB (NLB in front of ALB) |
| **Health checks** | TCP, HTTP, HTTPS |
| **TLS** | TLS termination or passthrough |
| **Preserve source IP** | Yes (client IP visible to backend) |
| **Best for** | Non-HTTP protocols, extreme performance, static IPs, gaming, IoT |

**CLB (Classic Load Balancer) — Legacy:**

| Feature | Details |
|---------|---------|
| **Layer** | 4 and 7 (limited) |
| **Status** | Legacy — use ALB or NLB for new deployments |
| **Best for** | Nothing new — migrate to ALB/NLB |

> **🔹 Ruby Context:** For Rails on AWS, ALB is the standard choice. It integrates with ECS (Fargate or EC2) for containerized Puma deployments, supports path-based routing for microservices, and provides built-in health checks against Rails' `/up` endpoint. A typical setup: **ALB → ECS Service (multiple Puma tasks) → RDS + ElastiCache**. For ActionCable (WebSocket), ALB's native WebSocket support works seamlessly with Puma's WebSocket handling.

**AWS LB Decision Guide:**

| Need | Use |
|------|-----|
| HTTP/HTTPS with path-based routing | ALB |
| Microservices with container targets | ALB |
| Lambda function targets | ALB |
| Non-HTTP protocols (TCP, UDP, TLS) | NLB |
| Ultra-low latency (< 1ms) | NLB |
| Static IP addresses | NLB |
| Millions of connections/sec | NLB |
| WebSocket or gRPC | ALB |
| Built-in authentication (OIDC) | ALB |
| NLB in front of ALB (static IP + L7 routing) | NLB → ALB |

---

### Envoy Proxy

**Envoy** is a high-performance, open-source edge and service proxy designed for cloud-native applications. It's the default sidecar proxy in Istio service mesh.

| Feature | Details |
|---------|---------|
| **Type** | L4/L7 proxy, service mesh sidecar |
| **Protocols** | HTTP/1.1, HTTP/2, gRPC (native), TCP, UDP, WebSocket, MongoDB, Redis, Thrift |
| **Configuration** | Dynamic via xDS API (no restart needed) |
| **Service discovery** | Integrates with Consul, Kubernetes, etcd |
| **Load balancing** | Round robin, least request, ring hash, random, Maglev |
| **Health checks** | Active (HTTP, gRPC, TCP) + passive (outlier detection) |
| **Circuit breaking** | Per-service circuit breakers |
| **Retries** | Configurable retry policies with budgets |
| **Observability** | Distributed tracing (Jaeger, Zipkin), metrics (Prometheus), access logs |
| **mTLS** | Automatic mutual TLS between services |
| **Best for** | Kubernetes, service mesh (Istio), microservices |

---

### Traefik

**Traefik** is a modern, cloud-native reverse proxy and load balancer with automatic service discovery.

| Feature | Details |
|---------|---------|
| **Type** | L7 reverse proxy and load balancer |
| **Auto-discovery** | Automatically discovers services from Docker, Kubernetes, Consul, etcd |
| **Let's Encrypt** | Automatic SSL certificate provisioning and renewal |
| **Configuration** | Dynamic — no restart needed; auto-configures from container labels |
| **Dashboard** | Built-in web dashboard |
| **Middleware** | Rate limiting, authentication, circuit breaker, retry, compression |
| **Best for** | Docker/Kubernetes environments with auto-discovery needs |

> **🔹 Ruby Context:** Traefik is a popular choice for Rails apps deployed with Docker Compose or Kubernetes. It auto-discovers Puma containers and configures routing from Docker labels — no manual Nginx config needed. For small-to-medium Rails deployments using Docker, Traefik with Let's Encrypt provides automatic SSL and load balancing with minimal configuration. Kamal (Rails' official deployment tool, formerly MRSK) uses Traefik as its default reverse proxy for zero-downtime deployments.

---

### Tool Selection Guide

| Scenario | Recommended Tool |
|----------|-----------------|
| Simple web app reverse proxy | Nginx |
| High-performance dedicated load balancer | HAProxy |
| AWS cloud-native HTTP/HTTPS | ALB |
| AWS cloud-native TCP/UDP | NLB |
| Kubernetes service mesh | Envoy (with Istio or Linkerd) |
| Docker/Kubernetes with auto-discovery | Traefik |
| gRPC-native load balancing | Envoy |
| Static file serving + reverse proxy | Nginx |
| Need built-in stats dashboard | HAProxy |
| Multi-cloud or on-premises | Nginx or HAProxy |

> **🔹 Ruby Context — Rails Deployment Patterns:**
>
> | Deployment Style | Recommended LB Stack |
> |-----------------|---------------------|
> | **Traditional (Capistrano + VPS)** | Nginx + Puma (Unix sockets) |
> | **Docker Compose (Kamal/MRSK)** | Traefik + Puma containers |
> | **AWS ECS/Fargate** | ALB + Puma containers |
> | **Kubernetes** | Nginx Ingress or Traefik + Puma pods |
> | **Heroku** | Heroku Router (managed, no config needed) |
> | **High-traffic on-prem** | HAProxy + Nginx + Puma |

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Load Balancer | Distributes traffic across servers; improves availability, performance, and fault tolerance |
| L4 vs L7 | L4 routes by IP/port (fast, simple); L7 routes by HTTP content (flexible, powerful) |
| L4 Use Case | Non-HTTP protocols, maximum throughput, TCP passthrough |
| L7 Use Case | HTTP routing by path/host/header, SSL termination, A/B testing, microservices |
| Round Robin | Simplest algorithm; good for homogeneous servers |
| Least Connections | Adapts to varying request durations; good for long-lived connections |
| Consistent Hashing | Minimal disruption when adding/removing servers; good for caches |
| Least Response Time | Routes to fastest server; best for heterogeneous environments |
| Active Health Checks | LB probes servers periodically; detects down servers |
| Passive Health Checks | LB monitors real traffic; detects errors in real-time |
| Deep vs Shallow Health | Shallow = process alive; Deep = dependencies healthy; use both |
| Connection Draining | Finish in-flight requests before removing a server |
| GSLB | Global traffic distribution across data centers via DNS |
| Anycast | Same IP on multiple servers; network routes to nearest; used by CDNs and DNS |
| Service Mesh | Sidecar proxies (Envoy) handle LB, retries, circuit breaking, mTLS between microservices |
| Client-Side LB | Client selects server directly; no central LB; used in gRPC and service mesh |
| Active-Passive HA | One active LB, one standby; simple but wastes capacity |
| Active-Active HA | Both LBs handle traffic; 2x capacity; more complex |
| VIP / Floating IP | IP that moves between servers for seamless failover |
| Availability Math | Serial: multiply availabilities; Parallel: 1 - (1-A)^N; redundancy at every layer |
| Nginx | Web server + reverse proxy; best for web apps and simple LB |
| HAProxy | Dedicated high-performance LB; best for TCP/HTTP load balancing |
| AWS ALB | Managed L7 LB; path/host routing, container targets, Lambda targets |
| AWS NLB | Managed L4 LB; ultra-low latency, static IPs, millions of connections |
| Envoy | Cloud-native proxy; service mesh sidecar; gRPC-native; dynamic configuration |

---

## Interview Tips for Module 23

1. **L4 vs L7** — explain the difference clearly; know when to use each; mention the L4-in-front-of-L7 pattern
2. **Algorithms** — know round robin, least connections, consistent hashing, and IP hash; explain trade-offs
3. **Consistent hashing** — explain the hash ring, virtual nodes, and why it minimizes redistribution
4. **Health checks** — explain active vs passive; shallow vs deep; warn about cascading failures with deep checks
5. **Connection draining** — explain why it matters and how it works during deployments and failures
6. **GSLB** — explain DNS-based global load balancing; mention latency-based vs geographic routing
7. **Anycast** — explain how it works (same IP, BGP routing); mention CDN and DNS use cases
8. **Service mesh** — explain the sidecar pattern (Envoy); mention circuit breaking, retries, mTLS
9. **High availability** — explain active-passive vs active-active; VIP failover with Keepalived
10. **Availability math** — calculate combined availability for serial and parallel components
11. **AWS ALB vs NLB** — know the differences; ALB for HTTP, NLB for TCP/UDP and static IPs
12. **Draw the full picture** — in HLD interviews, show DNS → GSLB → CDN → L4 LB → L7 LB → App Servers → DB

> **🔹 Ruby Context — Interview Additions:**
> 13. **Rails deployment stack** — be ready to explain the Nginx → Puma pipeline, why multiple Puma workers are needed (GIL), and how Puma's thread pool works
> 14. **Kamal/MRSK** — know that Rails' official deployment tool uses Traefik for zero-downtime deploys with Docker
> 15. **Health endpoints** — mention Rails 7.1's built-in `/up` endpoint and gems like `okcomputer` for deep health checks
> 16. **Session management** — explain why Rails apps use Redis-backed sessions instead of sticky sessions for better scalability
