# Module 22: Scalability

> Scalability is the ability of a system to handle increased load by adding resources. A scalable system maintains acceptable performance as traffic, data volume, or user count grows — from hundreds to millions of users. This module covers the two fundamental scaling approaches (vertical and horizontal), database and application scaling techniques, architectural patterns that enable scale (CQRS, Event Sourcing, Saga), and performance optimization strategies.

---

## 22.1 Vertical Scaling (Scale Up)

> **Vertical scaling** means increasing the capacity of a single machine — adding more CPU, RAM, disk, or upgrading to faster hardware. It's the simplest way to handle more load, but it has hard limits.

---

### Adding More CPU, RAM, Disk

```
Before (small instance):                After (scaled up):
  +------------------+                   +------------------+
  | 4 CPU cores      |                   | 64 CPU cores     |
  | 16 GB RAM        |     Scale Up      | 512 GB RAM       |
  | 500 GB SSD       |  ──────────────→  | 4 TB NVMe SSD    |
  | 1 Gbps network   |                   | 25 Gbps network  |
  +------------------+                   +------------------+
  Handles: 1,000 req/s                   Handles: 20,000 req/s
```

**What Vertical Scaling Improves:**

| Resource | Impact |
|----------|--------|
| **More CPU cores** | Handle more concurrent requests, faster computation |
| **More RAM** | Larger database buffer pool, more in-process caching, fewer disk reads |
| **Faster disk (NVMe SSD)** | Faster database reads/writes, reduced I/O latency |
| **More network bandwidth** | Handle more concurrent connections, faster data transfer |
| **Faster CPU (higher clock)** | Faster single-threaded operations (many DB operations are single-threaded) |

**Cloud Instance Scaling Examples:**

| Cloud | Small | Medium | Large | Largest |
|-------|-------|--------|-------|---------|
| **AWS EC2** | t3.medium (2 vCPU, 4 GB) | m6i.xlarge (4 vCPU, 16 GB) | r6i.8xlarge (32 vCPU, 256 GB) | u-24tb1.metal (448 vCPU, 24 TB) |
| **AWS RDS** | db.t3.medium | db.r6g.xlarge | db.r6g.8xlarge | db.r6g.16xlarge (64 vCPU, 512 GB) |
| **GCP** | n2-standard-2 | n2-standard-8 | n2-highmem-64 | m2-ultramem-416 (416 vCPU, 12 TB) |

---

### Limits of Vertical Scaling

| Limitation | Details |
|-----------|---------|
| **Hardware ceiling** | There's a maximum machine size — you can't buy a server with 10,000 CPU cores |
| **Cost curve** | Cost increases non-linearly — a 2x bigger machine costs 3-4x more |
| **Single point of failure** | One machine = one failure point; if it goes down, everything goes down |
| **Downtime for upgrades** | Resizing often requires a restart (minutes of downtime) |
| **Diminishing returns** | Doubling RAM doesn't double performance if the bottleneck is CPU or I/O |
| **No geographic distribution** | A single machine can only be in one data center |

**Cost Non-Linearity Example:**

```
AWS RDS PostgreSQL (us-east-1, on-demand):
  db.r6g.large    (2 vCPU, 16 GB):   ~$0.26/hr  → $190/month
  db.r6g.xlarge   (4 vCPU, 32 GB):   ~$0.52/hr  → $380/month   (2x resources, 2x cost)
  db.r6g.4xlarge  (16 vCPU, 128 GB): ~$2.07/hr  → $1,510/month (8x resources, 8x cost)
  db.r6g.16xlarge (64 vCPU, 512 GB): ~$8.28/hr  → $6,044/month (32x resources, 32x cost)

  Going from 2 vCPU to 64 vCPU (32x) costs 32x more
  But performance doesn't scale 32x — maybe 10-15x due to contention, locking, etc.
```

---

### When Vertical Scaling is Appropriate

| Scenario | Why Vertical Scaling Works |
|----------|--------------------------|
| **Early-stage startup** | Simple, no architectural changes needed; focus on product, not infrastructure |
| **Database server** | Databases are hard to horizontally scale; a bigger machine is often the easiest solution |
| **Single-threaded workloads** | Some workloads (Redis, single-query PostgreSQL) benefit more from faster CPU than more CPUs |
| **Low traffic** (< 10K req/s) | A single beefy server handles this easily |
| **Quick fix** | Need more capacity NOW; vertical scaling takes minutes, horizontal takes days/weeks |
| **Stateful applications** | Applications with in-memory state are hard to distribute |

**Key Point:** Vertical scaling is the right **first step**. Don't prematurely optimize for horizontal scaling. Scale vertically until you hit the limits, then add horizontal scaling for the specific bottleneck.

---


## 22.2 Horizontal Scaling (Scale Out)

> **Horizontal scaling** means adding more machines to distribute the load. Instead of one powerful server, you use many smaller servers working together. This is the foundation of modern distributed systems and cloud-native architectures.

---

### Adding More Machines

```
Vertical Scaling:                    Horizontal Scaling:
  +------------------+                +--------+ +--------+ +--------+
  |  One BIG server  |                | Server | | Server | | Server |
  |  64 CPU, 512 GB  |                |   1    | |   2    | |   3    |
  |                  |                +--------+ +--------+ +--------+
  |  All traffic     |                +--------+ +--------+ +--------+
  |  goes here       |                | Server | | Server | | Server |
  +------------------+                |   4    | |   5    | |   6    |
                                      +--------+ +--------+ +--------+
  Limit: hardware ceiling              Limit: virtually unlimited
  Cost: expensive                      Cost: linear (add more cheap servers)
  Failure: total outage                Failure: one server down, others continue
```

**Horizontal Scaling Advantages:**

| Advantage | Details |
|-----------|---------|
| **No hardware ceiling** | Add as many servers as needed — theoretically unlimited |
| **Linear cost** | 2x servers ≈ 2x cost ≈ 2x capacity (roughly) |
| **Fault tolerance** | One server fails → others continue serving; no single point of failure |
| **Geographic distribution** | Place servers in multiple regions for lower latency |
| **Incremental** | Add capacity gradually as traffic grows |
| **Cloud-native** | Auto-scaling groups add/remove instances based on demand |

**Horizontal Scaling Challenges:**

| Challenge | Details |
|-----------|---------|
| **Complexity** | Distributed systems are inherently more complex than single servers |
| **Data consistency** | Keeping data consistent across multiple servers is hard (CAP theorem) |
| **Stateful services** | Services with in-memory state can't simply be replicated |
| **Network overhead** | Inter-server communication adds latency |
| **Operational overhead** | More servers to monitor, deploy, and maintain |
| **Data partitioning** | Must decide how to split data across servers (sharding) |

---

### Stateless Services

The key to horizontal scaling is making services **stateless** — no server-specific state that ties a request to a particular server.

```
Stateful Service (hard to scale):
  Server A: { session: "user_123 → Alice's cart" }
  Server B: { session: (empty) }
  
  If user_123's request goes to Server B → session not found → error!
  Must use sticky sessions (ties user to specific server)

Stateless Service (easy to scale):
  Server A: (no local state)
  Server B: (no local state)
  
  All state stored externally:
    Redis: { "session:user_123" → "Alice's cart" }
  
  Any server can handle any request → simple round-robin load balancing
```

**Making Services Stateless:**

| State Type | Where to Move It |
|-----------|-----------------|
| **User sessions** | Redis, Memcached, or JWT tokens |
| **Shopping carts** | Redis or database |
| **File uploads** | Object storage (S3) — not local disk |
| **Temporary files** | Object storage or shared filesystem |
| **Configuration** | Config service, environment variables, or config files (not in-memory) |
| **Caches** | Distributed cache (Redis) instead of in-process cache |

---

### Session Management

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Sticky Sessions** | Load balancer routes same user to same server (cookie/IP-based) | Simple; no external store needed | Server failure loses sessions; uneven load distribution |
| **External Session Store** | Sessions stored in Redis/Memcached; any server can read them | True statelessness; fault tolerant | Extra infrastructure (Redis); network hop for every request |
| **JWT Tokens** | Session data encoded in a signed token; client sends with every request | No server-side storage; truly stateless | Can't revoke easily; token size grows with data; not suitable for large session data |
| **Encrypted Cookies** | Session data encrypted and stored in browser cookies | No server-side storage | Limited to 4KB; encryption overhead |

**Recommended Approach:** External session store (Redis) for most applications. JWT for stateless APIs where session revocation isn't critical.

```
External Session Store Pattern:

  Client → Load Balancer → Any App Server → Redis (session store)
  
  1. Client sends request with session cookie: session_id=abc123
  2. Load balancer routes to ANY server (round-robin)
  3. App server reads session from Redis: GET session:abc123
  4. App server processes request using session data
  5. App server updates session in Redis: SET session:abc123 = updated_data, EX 1800
```

---

### Data Partitioning

When a single database can't handle the load, partition (shard) the data across multiple database servers. Covered in depth in Module 20.4.

**Quick Summary:**

```
Single Database (bottleneck):
  All 100M users on one server → too much data, too many queries

Partitioned Database:
  Shard 1: Users A-H (25M users)
  Shard 2: Users I-P (25M users)
  Shard 3: Users Q-Z (25M users)
  Shard 4: Users 0-9 (25M users)
  
  Each shard handles 1/4 of the load → 4x capacity
```

**Scaling Strategy Progression:**

```
Stage 1: Single server (vertical scaling)
  → Good for 0-10K users

Stage 2: Read replicas + caching
  → Good for 10K-100K users
  → Add Redis cache, add 2-3 read replicas

Stage 3: Application horizontal scaling + load balancer
  → Good for 100K-1M users
  → Stateless app servers behind a load balancer

Stage 4: Database sharding + CDN
  → Good for 1M-10M users
  → Shard the database, add CDN for static content

Stage 5: Microservices + message queues + multi-region
  → Good for 10M+ users
  → Split into services, async processing, geographic distribution
```

---


## 22.3 Database Scaling

> The database is almost always the first bottleneck in a growing system. Application servers are easy to scale horizontally (stateless), but databases hold state and are much harder to scale. This section covers the primary techniques for scaling databases.

---

### Read Replicas

The simplest database scaling technique — add read-only copies of the database to handle read traffic.

```
Before (single DB handles everything):
  App Servers → Primary DB (reads + writes) → 10,000 QPS capacity

After (read replicas):
  App Servers → Primary DB (writes only) → 2,000 writes/sec
             → Replica 1 (reads) → 10,000 reads/sec
             → Replica 2 (reads) → 10,000 reads/sec
             → Replica 3 (reads) → 10,000 reads/sec
  Total read capacity: 30,000 reads/sec (3x improvement)
```

**Implementation Pattern:**

```python
# Application code routes reads and writes to different connections
class DatabaseRouter:
    def get_connection(self, query_type):
        if query_type == "write":
            return primary_connection      # always write to primary
        else:
            return random.choice(replica_connections)  # read from any replica
```

**When Read Replicas Are Enough:**
- Read-heavy workloads (90%+ reads) — most web applications
- Reporting queries that can tolerate slight staleness
- Geographic distribution (replica in each region for lower latency)

**When Read Replicas Are NOT Enough:**
- Write-heavy workloads — replicas don't help with write throughput
- Single large dataset that doesn't fit on one server — need sharding
- Need strong consistency for all reads — replicas have replication lag

---

### Sharding

Split the database into multiple independent pieces, each on its own server. Covered in depth in Module 20.4.

**Quick Decision Guide:**

| Question | Answer |
|----------|--------|
| Is the bottleneck reads? | Try read replicas + caching first |
| Is the bottleneck writes? | Sharding is likely needed |
| Is the data too large for one server? | Sharding is needed |
| Can you tolerate replication lag? | Read replicas may be sufficient |
| Do you need cross-shard queries? | Consider if sharding is worth the complexity |

---

### Connection Pooling

Each database connection consumes server resources (memory, file descriptors, process/thread). Without connection pooling, each application request opens a new connection — expensive and wasteful.

```
Without Connection Pooling:
  Request 1 → Open connection → Query → Close connection (50ms overhead)
  Request 2 → Open connection → Query → Close connection (50ms overhead)
  Request 3 → Open connection → Query → Close connection (50ms overhead)
  
  1000 concurrent requests → 1000 simultaneous connections → database overwhelmed!

With Connection Pooling:
  Pool of 50 connections (pre-opened, reused)
  Request 1 → Borrow connection from pool → Query → Return to pool (0ms overhead)
  Request 2 → Borrow connection from pool → Query → Return to pool
  Request 3 → Borrow connection from pool → Query → Return to pool
  
  1000 concurrent requests → 50 connections (requests queue for available connections)
```

**Connection Pool Configuration:**

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| **Min connections** | Minimum connections kept open (even when idle) | 5-10 |
| **Max connections** | Maximum connections allowed | 20-100 per app instance |
| **Idle timeout** | Close idle connections after this duration | 5-10 minutes |
| **Connection timeout** | Max time to wait for a connection from the pool | 5-30 seconds |
| **Max lifetime** | Close connections after this total lifetime (prevents stale connections) | 30-60 minutes |

**Connection Pool Sizing Formula:**

```
Total connections = Number of app instances × Max pool size per instance

Example:
  10 app servers × 20 connections each = 200 total database connections
  PostgreSQL default max_connections = 100 → NOT ENOUGH!
  
  Solutions:
  1. Increase max_connections (but each connection uses ~10 MB RAM)
  2. Reduce pool size per instance
  3. Use a connection pooler (PgBouncer, ProxySQL)
```

**External Connection Poolers:**

| Tool | Database | Description |
|------|----------|-------------|
| **PgBouncer** | PostgreSQL | Lightweight connection pooler; multiplexes many client connections to few DB connections |
| **PgPool-II** | PostgreSQL | Connection pooling + load balancing + replication management |
| **ProxySQL** | MySQL | Connection pooling + query routing + read/write splitting |
| **Amazon RDS Proxy** | AWS RDS | Managed connection pooler for RDS databases |

```
Without PgBouncer:
  100 app instances × 20 connections = 2,000 DB connections → DB overwhelmed

With PgBouncer:
  100 app instances × 20 connections → PgBouncer (2,000 client connections)
  PgBouncer → Database (50 actual connections)
  PgBouncer multiplexes 2,000 client connections onto 50 DB connections
```

---

### Query Optimization

Before scaling hardware, optimize your queries — a single slow query can bring down a database.

**Common Optimization Techniques:**

| Technique | Impact | Example |
|-----------|--------|---------|
| **Add indexes** | 100-1000x faster reads | `CREATE INDEX idx_user_email ON users(email)` |
| **Use EXPLAIN** | Identify slow query plans | `EXPLAIN ANALYZE SELECT ...` |
| **Avoid SELECT *** | Read only needed columns | `SELECT name, email FROM users` instead of `SELECT *` |
| **Limit result sets** | Reduce data transfer | `LIMIT 20` instead of returning all rows |
| **Use pagination** | Avoid loading entire tables | Cursor-based pagination for large datasets |
| **Optimize JOINs** | Ensure JOIN columns are indexed | Index foreign keys |
| **Avoid N+1 queries** | Batch related queries | Use JOINs or IN clauses instead of loops |
| **Use prepared statements** | Avoid re-parsing queries | Parameterized queries |
| **Partition large tables** | Faster scans on partitioned data | Partition by date for time-series data |
| **Archive old data** | Keep active tables small | Move old orders to an archive table |

**N+1 Query Problem:**

```python
# BAD: N+1 queries (1 query for users + N queries for orders)
users = db.query("SELECT * FROM users LIMIT 100")
for user in users:
    orders = db.query(f"SELECT * FROM orders WHERE user_id = {user.id}")
# Total: 101 queries!

# GOOD: 2 queries (1 for users + 1 for all orders)
users = db.query("SELECT * FROM users LIMIT 100")
user_ids = [u.id for u in users]
orders = db.query(f"SELECT * FROM orders WHERE user_id IN ({','.join(user_ids)})")
# Total: 2 queries!

# BEST: 1 query with JOIN
results = db.query("""
    SELECT u.*, o.id as order_id, o.total
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    LIMIT 100
""")
# Total: 1 query!
```

---

### Denormalization for Read Performance

Trade write complexity for read performance by storing redundant data. Covered in Module 18.3.

```
Normalized (3 JOINs for order details):
  SELECT o.id, u.name, p.name, oi.quantity
  FROM orders o
  JOIN users u ON o.user_id = u.id
  JOIN order_items oi ON o.id = oi.order_id
  JOIN products p ON oi.product_id = p.id
  WHERE o.id = 123;

Denormalized (single table read):
  SELECT id, user_name, product_name, quantity
  FROM order_details_denormalized
  WHERE order_id = 123;
  
  (user_name and product_name are duplicated in the denormalized table)
```

**When to Denormalize:**
- Read:write ratio > 10:1
- JOINs are the performance bottleneck
- Data changes infrequently
- You can tolerate eventual consistency between the normalized and denormalized copies

---


## 22.4 Application Scaling

> Scaling the application layer is typically easier than scaling the database because application servers can be made **stateless**. Once stateless, you simply add more instances behind a load balancer.

---

### Stateless Application Design

**Principles for Stateless Applications:**

| Principle | Implementation |
|-----------|---------------|
| No local session state | Store sessions in Redis or use JWT |
| No local file storage | Use S3/GCS for file uploads |
| No in-memory caches that must be consistent | Use distributed cache (Redis) for shared state |
| Configuration from environment | Environment variables, config service, not local files |
| Idempotent operations | Safe to retry requests (important when load balancer retries) |
| Graceful shutdown | Finish in-flight requests before stopping (drain connections) |

**The Twelve-Factor App (key principles for scalable apps):**

| Factor | Description |
|--------|-------------|
| **Codebase** | One codebase, many deploys (dev, staging, prod) |
| **Dependencies** | Explicitly declare and isolate dependencies |
| **Config** | Store config in environment variables (not in code) |
| **Backing services** | Treat databases, caches, queues as attached resources |
| **Build, release, run** | Strictly separate build and run stages |
| **Processes** | Execute the app as stateless processes |
| **Port binding** | Export services via port binding |
| **Concurrency** | Scale out via the process model (more instances) |
| **Disposability** | Fast startup and graceful shutdown |
| **Dev/prod parity** | Keep development, staging, and production as similar as possible |
| **Logs** | Treat logs as event streams (stdout, not files) |
| **Admin processes** | Run admin/management tasks as one-off processes |

---

### Horizontal Pod Autoscaling (Kubernetes)

In Kubernetes, **Horizontal Pod Autoscaler (HPA)** automatically adjusts the number of pod replicas based on observed metrics.

```
HPA Configuration:
  Target: Deployment "api-server"
  Min replicas: 3
  Max replicas: 50
  Target CPU utilization: 70%

Behavior:
  CPU at 30% (low traffic):   HPA scales down to 3 pods
  CPU at 70% (normal):        HPA maintains current count
  CPU at 90% (high traffic):  HPA scales up (adds pods)
  CPU at 95% (spike):         HPA scales up aggressively

  Scale-up:  React quickly (seconds)
  Scale-down: React slowly (minutes) — avoid flapping
```

```yaml
# Kubernetes HPA manifest
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30    # wait 30s before scaling up
      policies:
      - type: Percent
        value: 100                       # can double pods in one step
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10                        # remove max 10% of pods per step
        periodSeconds: 60
```

**Custom Metrics for HPA:**
- **CPU utilization** — default, good for compute-bound workloads
- **Memory utilization** — good for memory-bound workloads
- **Request rate (QPS)** — scale based on incoming traffic
- **Queue depth** — scale workers based on pending messages
- **Response latency (P99)** — scale when latency exceeds threshold

---

### Auto-Scaling Groups (AWS)

AWS **Auto Scaling Groups (ASG)** automatically manage EC2 instances based on scaling policies.

```
Auto Scaling Group:
  Min instances: 2
  Desired instances: 4
  Max instances: 20

Scaling Policies:
  Scale out: Add 2 instances when avg CPU > 70% for 5 minutes
  Scale in:  Remove 1 instance when avg CPU < 30% for 10 minutes

  +--------+  +--------+  +--------+  +--------+
  | EC2-1  |  | EC2-2  |  | EC2-3  |  | EC2-4  |  ← desired: 4
  +--------+  +--------+  +--------+  +--------+
                                                    Traffic spike!
  +--------+  +--------+  +--------+  +--------+  +--------+  +--------+
  | EC2-1  |  | EC2-2  |  | EC2-3  |  | EC2-4  |  | EC2-5  |  | EC2-6  |  ← scaled to 6
  +--------+  +--------+  +--------+  +--------+  +--------+  +--------+
```

**Scaling Policy Types:**

| Policy Type | Description | Use Case |
|-------------|-------------|----------|
| **Target Tracking** | Maintain a target metric value (e.g., CPU at 70%) | Most common; simple and effective |
| **Step Scaling** | Add/remove instances in steps based on alarm thresholds | Fine-grained control over scaling behavior |
| **Scheduled Scaling** | Scale at specific times (e.g., more instances during business hours) | Predictable traffic patterns |
| **Predictive Scaling** | ML-based prediction of future traffic; pre-scales before the spike | Recurring patterns (daily, weekly) |

---

### Scaling Microservices Independently

One of the key benefits of microservices: each service can be scaled independently based on its own load.

```
Monolith Scaling:
  Scale the ENTIRE application (even if only one module is the bottleneck)
  [User + Order + Payment + Notification] × 10 instances

Microservice Scaling:
  Scale each service based on its own demand:
  User Service:         × 3 instances  (low traffic)
  Order Service:        × 10 instances (high traffic — Black Friday)
  Payment Service:      × 5 instances  (moderate traffic)
  Notification Service: × 20 instances (sending millions of emails)
```

**Per-Service Scaling Strategies:**

| Service Type | Scaling Strategy | Metric |
|-------------|-----------------|--------|
| **API services** (synchronous) | HPA based on CPU/request rate | Requests per second, CPU utilization |
| **Worker services** (async) | HPA based on queue depth | Pending messages in queue |
| **Batch processing** | Scheduled scaling or job-based | Job queue size, time of day |
| **Real-time services** (WebSocket) | Connection-based scaling | Active WebSocket connections |
| **ML inference** | GPU-based scaling | Inference queue depth, GPU utilization |

---


## 22.5 Scaling Patterns

> As systems grow, simple scaling techniques (more servers, read replicas) aren't enough. You need architectural patterns that fundamentally change how data flows through the system. These patterns enable massive scale but add significant complexity — use them only when simpler approaches are insufficient.

---

### CQRS (Command Query Responsibility Segregation)

**CQRS** separates the **write model** (commands) from the **read model** (queries). Each model is optimized independently — the write model for consistency and the read model for performance.

```
Traditional (single model):
  Client → API → Single Database (handles both reads and writes)
  
  Problem: Read queries (complex JOINs, aggregations) compete with writes
           for the same database resources.

CQRS (separate models):
  Write Path (Commands):
    Client → API → Command Handler → Write Database (normalized, ACID)
                                         |
                                    Event/Change Stream
                                         |
                                         ↓
  Read Path (Queries):                Projection
    Client → API → Query Handler → Read Database (denormalized, optimized for reads)
```

```
  +--------+     Commands      +----------------+     Events     +------------------+
  | Client | ────────────────→ | Write Model    | ─────────────→ | Event Bus        |
  |        |                   | (PostgreSQL)   |                | (Kafka/SNS)      |
  |        |                   | Normalized     |                +--------+---------+
  |        |                   | ACID           |                         |
  |        |                   +----------------+                         |
  |        |                                                    +--------v---------+
  |        |     Queries       +----------------+               | Projection       |
  |        | ←────────────── | Read Model     | ←───────────── | Service          |
  |        |                   | (Elasticsearch |               | (builds read     |
  |        |                   |  or Redis or   |               |  model from      |
  |        |                   |  DynamoDB)     |               |  events)         |
  +--------+                   | Denormalized   |               +------------------+
                               +----------------+
```

**When to Use CQRS:**

| Use When | Don't Use When |
|----------|---------------|
| Read and write workloads have very different characteristics | Simple CRUD application |
| Read model needs different structure than write model | Read and write patterns are similar |
| Need to scale reads and writes independently | Team is small and can't handle the complexity |
| Complex read queries are slowing down writes | Data model is simple |
| Different consistency requirements for reads vs writes | Strong consistency is required for all reads |

**CQRS Trade-offs:**

| Pros | Cons |
|------|------|
| Independent scaling of reads and writes | Increased complexity (two models to maintain) |
| Optimized data models for each path | Eventual consistency between write and read models |
| Better performance (no read/write contention) | More infrastructure (event bus, projection service) |
| Flexibility (different databases for each model) | Debugging is harder (data flows through multiple systems) |

---

### Event Sourcing

**Event Sourcing** stores the **sequence of events** that led to the current state, rather than storing the current state directly. The current state is derived by replaying all events.

```
Traditional (state-based):
  Account table: { id: 1, balance: 750 }
  (Only the current balance is stored — history is lost)

Event Sourcing:
  Event Store:
    1. AccountCreated  { id: 1, initial_balance: 1000 }
    2. MoneyWithdrawn  { id: 1, amount: 200 }
    3. MoneyDeposited  { id: 1, amount: 50 }
    4. MoneyWithdrawn  { id: 1, amount: 100 }
  
  Current state = replay events: 1000 - 200 + 50 - 100 = 750
  (Full history is preserved — you can reconstruct state at any point in time)
```

**Event Store Properties:**
- **Append-only** — events are never modified or deleted
- **Ordered** — events have a sequence number or timestamp
- **Immutable** — once written, an event is permanent
- **Complete** — the event store is the single source of truth

**Benefits of Event Sourcing:**

| Benefit | Description |
|---------|-------------|
| **Complete audit trail** | Every change is recorded — who did what, when |
| **Time travel** | Reconstruct the state at any point in time |
| **Event replay** | Rebuild read models, fix bugs by replaying events with corrected logic |
| **Debugging** | Reproduce any bug by replaying the exact sequence of events |
| **Analytics** | Rich event data for business intelligence |
| **Decoupling** | Other services can subscribe to events and build their own views |

**Challenges:**

| Challenge | Description |
|-----------|-------------|
| **Complexity** | Fundamentally different programming model |
| **Event schema evolution** | Old events must remain readable as the schema evolves |
| **Eventual consistency** | Read models are derived asynchronously from events |
| **Storage growth** | Event store grows indefinitely (mitigated by snapshots) |
| **Querying** | Can't query the event store directly for current state — need projections |

**Snapshots:**
To avoid replaying millions of events, periodically save a **snapshot** of the current state:

```
Events: 1, 2, 3, ..., 999,999, 1,000,000
Snapshot at event 999,000: { balance: 742 }

To get current state:
  Load snapshot (balance: 742)
  Replay events 999,001 to 1,000,000 (only 1,000 events instead of 1,000,000)
```

**Event Sourcing + CQRS:**
Event Sourcing and CQRS are often used together:
- **Write side:** Append events to the event store
- **Read side:** Project events into denormalized read models
- **Event bus:** Connects the write side to the read side

---

### Materialized Views

A **materialized view** is a pre-computed query result stored as a physical table. It's refreshed periodically or on-demand. Covered in Module 18.2.

**In the context of scaling:**
- Materialized views offload expensive aggregation queries from the primary database
- They serve as the "read model" in CQRS
- They can be stored in a different database optimized for reads (Elasticsearch, Redis, DynamoDB)

```
Write Path:
  App → PostgreSQL (normalized, ACID)
  
  Background Job (every 5 minutes):
    REFRESH MATERIALIZED VIEW monthly_revenue;
    REFRESH MATERIALIZED VIEW top_products;
    REFRESH MATERIALIZED VIEW user_activity_summary;

Read Path:
  Dashboard → SELECT * FROM monthly_revenue WHERE month = '2025-03';
  (Reads from pre-computed materialized view — fast!)
```

---

### Database per Service

In a microservices architecture, each service owns its own database. No service directly accesses another service's database.

```
Monolith (shared database):
  User Service  ──┐
  Order Service ──┼──→ Single Database (all tables)
  Payment Service─┘

Microservices (database per service):
  User Service    → User DB (PostgreSQL)
  Order Service   → Order DB (PostgreSQL)
  Payment Service → Payment DB (PostgreSQL)
  Search Service  → Search DB (Elasticsearch)
  Cache Service   → Cache DB (Redis)
```

**Benefits:**
- Services are truly independent — can deploy, scale, and evolve independently
- Each service can choose the best database for its needs (polyglot persistence)
- No shared schema — no cross-team coordination for schema changes
- Failure isolation — one database going down doesn't affect other services

**Challenges:**
- No cross-service JOINs — must use API calls or event-driven data synchronization
- No cross-service transactions — must use the Saga pattern
- Data duplication — some data is replicated across services
- Increased operational complexity — more databases to manage

---

### Saga Pattern for Distributed Transactions

When a business operation spans multiple services (each with its own database), you can't use a traditional ACID transaction. The **Saga pattern** breaks the operation into a sequence of local transactions, each with a compensating action for rollback.

**Choreography-Based Saga (event-driven):**

```
Order Saga (happy path):
  1. Order Service: Create order (status=pending) → publish "OrderCreated" event
  2. Payment Service: Charge payment → publish "PaymentCompleted" event
  3. Inventory Service: Reserve items → publish "InventoryReserved" event
  4. Order Service: Update order (status=confirmed) → publish "OrderConfirmed" event

Order Saga (failure — payment fails):
  1. Order Service: Create order (status=pending) → publish "OrderCreated" event
  2. Payment Service: Charge fails → publish "PaymentFailed" event
  3. Order Service: Compensate → cancel order (status=cancelled)
```

```
  Order Svc          Payment Svc         Inventory Svc
      |                   |                    |
      | OrderCreated      |                    |
      |──────────────────→|                    |
      |                   | PaymentCompleted   |
      |                   |───────────────────→|
      |                   |                    | InventoryReserved
      |←──────────────────────────────────────|
      | OrderConfirmed    |                    |
      |                   |                    |
```

**Orchestration-Based Saga (central coordinator):**

```
  +------------------+
  | Saga Orchestrator|
  +--------+---------+
           |
    1. Create Order  → Order Service
    2. Charge Payment → Payment Service
    3. Reserve Items  → Inventory Service
    4. Confirm Order  → Order Service
    
    If step 2 fails:
    → Compensate step 1: Cancel Order → Order Service
    
    If step 3 fails:
    → Compensate step 2: Refund Payment → Payment Service
    → Compensate step 1: Cancel Order → Order Service
```

**Choreography vs Orchestration:**

| Feature | Choreography | Orchestration |
|---------|-------------|---------------|
| Coordination | Decentralized (events) | Centralized (orchestrator) |
| Coupling | Loose (services don't know about each other) | Tighter (orchestrator knows all services) |
| Complexity | Harder to understand (distributed logic) | Easier to understand (centralized logic) |
| Single point of failure | None | Orchestrator is a SPOF |
| Debugging | Harder (follow events across services) | Easier (orchestrator has full view) |
| Best for | Simple sagas (2-3 steps) | Complex sagas (4+ steps, branching logic) |

---


## 22.6 Performance Optimization

> Scaling isn't just about adding more servers — it's also about making each server do more with less. Performance optimization reduces the resources needed per request, which directly translates to handling more traffic with the same infrastructure (or the same traffic with less infrastructure and lower cost).

---

### Latency vs Throughput

**Latency:** The time it takes to complete a single request (measured in milliseconds).
**Throughput:** The number of requests the system can handle per unit of time (measured in requests per second).

```
Analogy — Highway:
  Latency = how long it takes ONE car to travel from A to B
  Throughput = how many cars pass a point per hour

  A narrow highway with no traffic: low latency, low throughput
  A wide highway with heavy traffic: higher latency (congestion), high throughput
```

**The Relationship:**
- Increasing throughput often increases latency (more contention, queuing)
- Reducing latency doesn't always increase throughput (may be limited by other resources)
- The goal is to optimize **both** — low latency AND high throughput

**Latency Breakdown of a Typical Web Request:**

```
Client → DNS lookup (1-50ms, cached: 0ms)
       → TCP handshake (1-100ms, keep-alive: 0ms)
       → TLS handshake (1-100ms, session resumption: 0ms)
       → HTTP request transfer (1-10ms)
       → Server processing:
           → Load balancer routing (0.1-1ms)
           → Application logic (1-50ms)
           → Cache lookup (0.1-1ms)
           → Database query (1-100ms)
       → HTTP response transfer (1-100ms)
       → Client rendering (10-500ms)

Total: 10-1000ms (depending on caching, network, query complexity)
```

---

### P50, P95, P99 Latencies (Percentile Latencies)

**Average latency is misleading** — it hides the experience of the slowest requests. Percentile latencies give a more accurate picture.

```
1000 requests with latencies (sorted):
  Request 1-500:   10ms each  (50th percentile = P50)
  Request 501-950: 50ms each  (95th percentile = P95)
  Request 951-990: 200ms each (99th percentile = P99)
  Request 991-999: 1000ms each (99.9th percentile = P999)
  Request 1000:    5000ms     (max)

  Average: ~45ms (looks fine!)
  P50: 10ms (median — half of requests are faster than this)
  P95: 50ms (95% of requests are faster than this)
  P99: 200ms (99% of requests are faster than this)
  P999: 1000ms (99.9% of requests are faster than this)
```

**Why Percentiles Matter:**

| Metric | What It Tells You |
|--------|------------------|
| **P50 (median)** | The "typical" user experience |
| **P95** | The experience of 1 in 20 users — important for user satisfaction |
| **P99** | The experience of 1 in 100 users — often your most important/active users |
| **P999** | The experience of 1 in 1000 users — tail latency, often caused by GC pauses, retries, or cold caches |
| **Average** | Misleading — a few slow requests can skew the average while most users are fine |

**SLA/SLO Targets (typical):**

| Service Type | P50 Target | P95 Target | P99 Target |
|-------------|-----------|-----------|-----------|
| API endpoint | < 50ms | < 200ms | < 500ms |
| Web page load | < 500ms | < 2s | < 5s |
| Database query | < 10ms | < 50ms | < 200ms |
| Cache lookup | < 1ms | < 5ms | < 10ms |

**Tail Latency Amplification:**
In microservices, a single user request may fan out to multiple services. The overall latency is determined by the **slowest** service call.

```
User request fans out to 5 services in parallel:
  Service A: P99 = 50ms
  Service B: P99 = 50ms
  Service C: P99 = 50ms
  Service D: P99 = 50ms
  Service E: P99 = 50ms

  Probability that ALL 5 are under P99: 0.99^5 = 95.1%
  Probability that at least ONE exceeds P99: 4.9%
  
  → The combined P99 is WORSE than any individual service's P99
  → With 50 service calls: 0.99^50 = 60.5% chance all are under P99
     → 39.5% of requests hit at least one tail latency!
```

**Mitigation:** Hedged requests (send the same request to multiple replicas, use the first response), timeouts, circuit breakers.

---

### Amdahl's Law

**Amdahl's Law** defines the theoretical maximum speedup from parallelization. If a fraction `P` of a task can be parallelized and the rest `(1-P)` is sequential, the maximum speedup with `N` processors is:

```
Speedup = 1 / ((1 - P) + P/N)

Where:
  P = fraction of the task that can be parallelized
  N = number of processors (or servers)
  1-P = fraction that must remain sequential
```

**Examples:**

```
If 90% of the work can be parallelized (P=0.9):
  N=2:   Speedup = 1 / (0.1 + 0.9/2)  = 1.82x
  N=4:   Speedup = 1 / (0.1 + 0.9/4)  = 3.08x
  N=10:  Speedup = 1 / (0.1 + 0.9/10) = 5.26x
  N=100: Speedup = 1 / (0.1 + 0.9/100)= 9.17x
  N=∞:   Speedup = 1 / 0.1            = 10x (maximum!)

If 50% of the work can be parallelized (P=0.5):
  N=∞:   Speedup = 1 / 0.5 = 2x (maximum — no matter how many servers!)
```

**Key Insight:** The sequential portion of the workload limits the maximum speedup. Even with infinite servers, if 10% of the work is sequential, you can never achieve more than 10x speedup.

**System Design Implications:**
- Identify the sequential bottleneck (database writes, single-threaded operations, global locks)
- Optimize the sequential portion first — it has the highest impact
- Adding more servers has diminishing returns if the sequential portion is large
- This is why database writes (sequential, single-primary) are often the bottleneck

---

### Connection Pooling

Covered in Section 22.3. Reusing database and HTTP connections instead of creating new ones for each request.

**Key Numbers:**
- TCP connection setup: 1-3ms (3-way handshake)
- TLS handshake: 5-50ms
- Database connection setup: 10-50ms (authentication, session initialization)
- Connection from pool: 0.01-0.1ms

---

### Batch Processing

Process multiple items in a single operation instead of one at a time.

```
Without Batching:
  INSERT INTO orders VALUES (1, ...);  → 1ms
  INSERT INTO orders VALUES (2, ...);  → 1ms
  INSERT INTO orders VALUES (3, ...);  → 1ms
  ... (1000 inserts)
  Total: 1000ms (1000 round trips)

With Batching:
  INSERT INTO orders VALUES (1, ...), (2, ...), (3, ...), ... (1000 rows);
  Total: 10ms (1 round trip, bulk insert)
  → 100x faster!
```

**Where to Apply Batching:**

| Operation | Without Batching | With Batching |
|-----------|-----------------|---------------|
| Database inserts | 1 INSERT per row | Multi-row INSERT or COPY |
| API calls | 1 HTTP request per item | Batch API endpoint |
| Message publishing | 1 message at a time | Batch publish (Kafka batch producer) |
| Email sending | 1 email at a time | Batch send (SES batch API) |
| Cache operations | 1 GET/SET at a time | MGET/MSET (Redis pipeline) |

---

### Async Processing

Move non-critical work out of the request path into background jobs.

```
Synchronous (slow — user waits for everything):
  User places order:
    1. Validate order (10ms)
    2. Charge payment (500ms)     ← slow external API
    3. Send confirmation email (200ms)  ← slow
    4. Update analytics (50ms)    ← not critical
    5. Notify warehouse (100ms)   ← not critical
  Total: 860ms (user waits for all 5 steps)

Asynchronous (fast — user waits only for critical steps):
  User places order:
    1. Validate order (10ms)
    2. Charge payment (500ms)     ← critical, must be synchronous
    3. Publish "OrderCreated" event to message queue (5ms)
  Total: 515ms (user gets response)

  Background workers (async):
    4. Send confirmation email (from queue)
    5. Update analytics (from queue)
    6. Notify warehouse (from queue)
  (User doesn't wait for these)
```

```
  +--------+     Sync      +----------+     Async     +---------------+
  | Client | ────────────→ | API      | ────────────→ | Message Queue |
  |        | ←── Response ─| Server   |               | (Kafka/SQS)  |
  +--------+  (fast: 515ms)+----------+               +-------+-------+
                                                              |
                                                    +---------+---------+
                                                    |                   |
                                              +-----v-----+    +-------v-------+
                                              | Email      |    | Analytics     |
                                              | Worker     |    | Worker        |
                                              +------------+    +---------------+
```

**What to Make Async:**
- Email/SMS/push notifications
- Analytics and logging
- Image/video processing
- Report generation
- Third-party API calls (non-critical)
- Search index updates
- Recommendation engine updates

**What to Keep Sync:**
- Authentication and authorization
- Payment processing (must confirm before responding)
- Data validation
- Core business logic that the user needs to see the result of

---

### Compression

Reduce the size of data transferred over the network.

| Algorithm | Compression Ratio | Speed | Use Case |
|-----------|------------------|-------|----------|
| **gzip** | Good (60-80% reduction) | Moderate | HTTP responses (most common) |
| **Brotli** | Better (10-25% better than gzip) | Slower compression, fast decompression | Static assets (pre-compressed) |
| **zstd** | Excellent | Very fast | Logs, data transfer, database compression |
| **snappy** | Moderate | Fastest | Real-time data (Kafka, database pages) |
| **lz4** | Moderate | Very fast | Real-time data, database pages |

```
HTTP Response Compression:
  Request:  Accept-Encoding: gzip, br
  Response: Content-Encoding: gzip
            (body is gzip-compressed)

  Uncompressed JSON: 50 KB
  gzip compressed:   8 KB (84% reduction)
  Brotli compressed: 6 KB (88% reduction)
```

**Where to Apply Compression:**
- HTTP responses (gzip/brotli — handled by Nginx/CDN)
- Database storage (zstd/lz4 — PostgreSQL, MySQL, Cassandra support page compression)
- Message queues (snappy/lz4 — Kafka supports producer-side compression)
- Log files (gzip/zstd — compress before archiving)
- Data transfer between services (protobuf is already compact; compress large payloads)

**When NOT to Compress:**
- Already compressed data (images, videos, encrypted data) — compression adds CPU overhead with no benefit
- Very small payloads (< 1 KB) — compression overhead exceeds savings
- Latency-critical paths where CPU time matters more than bandwidth

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Vertical Scaling | Add more CPU/RAM to one server; simple but has hardware ceiling and cost non-linearity |
| Horizontal Scaling | Add more servers; virtually unlimited but requires stateless design and adds complexity |
| Scale Vertically First | Don't prematurely optimize; scale up until you hit limits, then scale out |
| Stateless Services | Move all state to external stores (Redis, S3); any server can handle any request |
| Session Management | External session store (Redis) or JWT; avoid sticky sessions |
| Scaling Progression | Single server → read replicas + cache → horizontal app scaling → sharding → microservices |
| Read Replicas | Simplest DB scaling; add read-only copies for read-heavy workloads |
| Connection Pooling | Reuse DB connections; use PgBouncer/ProxySQL to multiplex connections |
| Query Optimization | Add indexes, avoid N+1 queries, use EXPLAIN, limit result sets |
| HPA (Kubernetes) | Auto-scale pods based on CPU, memory, or custom metrics |
| Auto Scaling Groups | AWS auto-scaling for EC2 instances based on scaling policies |
| CQRS | Separate write model (normalized, ACID) from read model (denormalized, fast) |
| Event Sourcing | Store events, not state; complete audit trail; replay to rebuild state |
| Saga Pattern | Distributed transactions across microservices; choreography or orchestration |
| Database per Service | Each microservice owns its database; no cross-service DB access |
| Percentile Latencies | P50 (median), P95, P99 matter more than average; tail latency amplification in microservices |
| Amdahl's Law | Sequential portion limits max speedup; optimize the bottleneck first |
| Batch Processing | Process multiple items in one operation; 10-100x faster than one-at-a-time |
| Async Processing | Move non-critical work to background queues; reduce user-facing latency |
| Compression | gzip/brotli for HTTP; snappy/lz4 for real-time; 60-88% size reduction |

---

## Interview Tips for Module 22

1. **Vertical vs Horizontal** — explain both; start with vertical, move to horizontal when limits are hit
2. **Stateless design** — explain how to make services stateless (externalize sessions, files, caches)
3. **Scaling progression** — walk through the stages: single server → replicas + cache → horizontal → sharding → microservices
4. **Connection pooling** — explain why it matters; mention PgBouncer for PostgreSQL
5. **N+1 queries** — explain the problem and show the fix (JOINs or IN clauses)
6. **CQRS** — draw the architecture; explain when to use it and the eventual consistency trade-off
7. **Event Sourcing** — explain the concept; mention snapshots for performance; pair with CQRS
8. **Saga Pattern** — explain choreography vs orchestration; draw the compensating transaction flow
9. **Percentile latencies** — explain P50/P95/P99; mention tail latency amplification in microservices
10. **Amdahl's Law** — explain why adding more servers has diminishing returns; identify the sequential bottleneck
11. **Async processing** — identify which parts of a workflow can be async (notifications, analytics) vs must be sync (payment)
12. **Auto-scaling** — explain HPA in Kubernetes or ASG in AWS; mention scaling metrics (CPU, queue depth, request rate)
