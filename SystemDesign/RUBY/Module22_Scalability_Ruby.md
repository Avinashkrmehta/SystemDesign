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

> **Ruby Context:** Ruby's MRI (CRuby) has a Global Interpreter Lock (GIL/GVL), meaning only one thread executes Ruby code at a time. This makes vertical scaling with faster CPUs particularly impactful for Ruby applications, since single-core performance directly affects throughput. Puma uses multiple workers (processes) to utilize multiple CPU cores, so more cores = more Puma workers = more concurrent requests.

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

> **Ruby Context:** Many Rails applications start on a single server with Puma and can handle significant traffic through vertical scaling alone. A common early-stage Rails setup is a single server with Puma (4-8 workers), PostgreSQL, and Redis — this can comfortably handle thousands of requests per second before needing horizontal scaling.

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

> **Ruby Context:** Rails makes it straightforward to externalize session state. Configure `config.session_store :redis_store` (via the `redis-session-store` gem) or use `config.session_store :cookie_store` with encrypted cookies. For file uploads, Active Storage supports S3, GCS, and Azure out of the box. For caching, Rails' `config.cache_store = :redis_cache_store` moves the cache to Redis.

---

### Session Management

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Sticky Sessions** | Load balancer routes same user to same server (cookie/IP-based) | Simple; no external store needed | Server failure loses sessions; uneven load distribution |
| **External Session Store** | Sessions stored in Redis/Memcached; any server can read them | True statelessness; fault tolerant | Extra infrastructure (Redis); network hop for every request |
| **JWT Tokens** | Session data encoded in a signed token; client sends with every request | No server-side storage; truly stateless | Can't revoke easily; token size grows with data; not suitable for large session data |
| **Encrypted Cookies** | Session data encrypted and stored in browser cookies | No server-side storage | Limited to 4KB; encryption overhead |

**Recommended Approach:** External session store (Redis) for most applications. JWT for stateless APIs where session revocation isn't critical.

> **Ruby Context:** Rails defaults to encrypted cookie sessions (`config.session_store :cookie_store`), which is already stateless. For larger session data or when you need server-side session management, switch to Redis: `config.session_store :redis_session_store, key: '_app_session', redis: { url: ENV['REDIS_URL'] }`.

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

> **Ruby Context:** The typical Rails scaling progression follows this pattern closely. Stage 1-2 is a single Heroku dyno or server with Puma + PostgreSQL + Redis. Stage 3 adds multiple Puma instances behind Nginx or an ALB. Stage 4 introduces database sharding (Rails 6+ has built-in multi-database support with `connects_to` and automatic role switching). Stage 5 may involve extracting services from the Rails monolith, using Sidekiq for async processing, and Kafka/RabbitMQ for inter-service communication.

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
