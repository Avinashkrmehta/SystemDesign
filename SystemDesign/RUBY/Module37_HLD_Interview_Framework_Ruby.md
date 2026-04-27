# Module 37: HLD Interview Framework

> High-Level Design (HLD) interviews test your ability to design the architecture of a large-scale distributed system — choosing the right databases, caching strategies, messaging patterns, and scaling techniques. Unlike LLD (which focuses on classes and code), HLD focuses on **components, data flow, and trade-offs**. This module provides a battle-tested framework for approaching any HLD interview problem.

> **Ruby Context:** HLD interviews are largely language-agnostic — the architecture is the same whether you implement in Ruby, Java, or Go. However, knowing Ruby/Rails-specific performance characteristics helps you make informed decisions: Puma handles 200-1,000 req/sec per instance, Sidekiq processes 100-500 jobs/sec, a Rails process uses 200-500 MB RAM, and PgBouncer is needed beyond ~100 database connections. Reference these numbers during estimation. For technology choices, mention Rails-ecosystem tools: Redis (cache + Sidekiq), PostgreSQL/DynamoDB (database), Karafka (Kafka), Searchkick (Elasticsearch), aws-sdk-s3 (storage), and Rack::Attack (rate limiting).

---

## 37.1 Step-by-Step Approach

> This 6-step framework works for every HLD problem. Follow it consistently and you'll never get lost in an interview.

---

### Step 1: Clarify Requirements (3-5 minutes)

**The most important step.** Ask questions to understand what you're building and at what scale.

**Functional Requirements — "What does the system do?"**

```
URL Shortener Example:
  ✅ In scope:
    - Shorten a long URL → short URL
    - Redirect short URL → long URL
    - URL expiration (optional)
    - Analytics (click count) (optional)
  
  ❌ Out of scope:
    - User accounts / authentication
    - URL editing after creation
    - Admin dashboard
```

**Non-Functional Requirements — "How well does it need to work?"**

| Requirement | Question to Ask | URL Shortener Answer |
|-------------|----------------|---------------------|
| **Latency** | "What's the acceptable response time?" | Redirect: < 100ms |
| **Throughput** | "How many requests per second?" | 1000 reads/sec, 100 writes/sec |
| **Availability** | "Can the system be down?" | 99.99% (redirect must always work) |
| **Consistency** | "Is eventual consistency OK?" | Yes (few seconds delay for new URLs is fine) |
| **Durability** | "Can we lose data?" | No — URLs must be durable |

**Scale — "How big is this?"**

```
  - 100M new URLs per month
  - 10:1 read:write ratio → 1B redirects per month
  - Store for 5 years → 6 billion URLs total
```

**Key Rule:** Write down the requirements. This becomes your contract with the interviewer.

---

### Step 2: Back-of-the-Envelope Estimation (3-5 minutes)

**Do the math before designing.** Estimation drives architectural decisions.

```
URL Shortener Estimation:

  Write QPS: 100M/month ÷ (30 × 86,400) ≈ 40/sec (peak 3x: ~120/sec)
  Read QPS: 1B/month ÷ (30 × 86,400) ≈ 400/sec (peak 3x: ~1,200/sec)
  Storage (5 years): 6B × 500 bytes = 3 TB
  Cache (20% of daily): 3.3M × 20% × 500 bytes = 330 MB → single Redis

  Rails-specific:
    Peak 1,200 reads/sec ÷ 1,000 req/sec per Puma instance = 2 instances (with headroom: 4)
    → Very manageable scale for Rails
```

> **Ruby Context:** Always translate QPS into Puma instances needed. This grounds your design in reality and shows you understand Rails performance characteristics.

**Estimation Cheat Sheet:**

| Metric | Formula |
|--------|---------|
| QPS | items_per_day / 86,400 (use 10^5 for easy math) |
| Peak QPS | Average QPS × 3 |
| Storage | items × size × retention_years × 365 |
| Cache | daily_active_items × 20% × item_size |
| Puma instances | Peak QPS / 500 (moderate complexity) |
| Sidekiq processes | Peak jobs/sec / 200 (moderate jobs) |
| DB connections | (puma_instances × workers × threads) + (sidekiq × concurrency) |

---

### Step 3: High-Level Architecture (5-10 minutes)

**Draw the big picture.** Start simple — 4-6 components.

```
URL Shortener — Architecture:

  +--------+     +-------------+     +------------------+     +----------+
  | Client | ──→ | API Gateway | ──→ | Rails API        | ──→ | Database |
  +--------+     | (Rack::Attack|    | (Puma)           |     | (PG/Dynamo)
                 |  rate limit) |     +--------+---------+     +----------+
                 +-------------+              |
                                     +--------v---------+
                                     | Redis Cache      |
                                     +------------------+
                                     
                                     +------------------+
                                     | Sidekiq Workers  |
                                     | (analytics,      |
                                     |  async tasks)    |
                                     +------------------+
```

**API Design:**

```
POST /api/v1/urls
  Request:  { "long_url": "https://example.com/very/long/path" }
  Response: { "short_url": "https://short.ly/abc123" }
  Status: 201 Created

GET /{short_code}
  Response: 301/302 Redirect → Location: https://example.com/very/long/path
```

**Tips:**
- Draw boxes for components, arrows for data flow
- Label arrows (HTTP, async/Sidekiq, Kafka event)
- Start with 4-6 components; add more in deep dive
- For Rails: always show Puma (web), Sidekiq (workers), Redis, Database

---

### Step 4: Deep Dive into Components (10-15 minutes)

The interviewer will ask you to go deeper. Be ready to discuss:

**Database Schema and Choice:**

```
Table: urls
  short_code  VARCHAR(7) PRIMARY KEY
  long_url    VARCHAR(2048) NOT NULL
  created_at  TIMESTAMP
  expires_at  TIMESTAMP (nullable)
  click_count BIGINT DEFAULT 0

Choice: DynamoDB (key-value access pattern) or PostgreSQL (moderate scale, ACID)
```

**Caching Strategy:**

```
Cache-Aside with Redis:
  1. GET /abc123 → check Redis: GET url:abc123
     HIT → return (fast!)
     MISS → query DB → store in Redis (TTL=24h) → return

  Rails implementation: Rails.cache.fetch("url:#{short_code}", expires_in: 24.hours) { db_lookup }
  Expected hit ratio: 90%+ (Pareto — 20% of URLs get 80% of traffic)
```

**Key Algorithm:**

```
Short URL generation: Counter + Base62 encoding
  7 chars of Base62 = 3.5 trillion combinations
  Or: SecureRandom.alphanumeric(7) + collision check
```

---

### Step 5: Address Bottlenecks & Scale (5-10 minutes)

```
Scaling Progression:

  Stage 1 (current): Single PG + Redis cache
    Handles: 1,200 reads/sec → 2-4 Puma instances

  Stage 2 (10x): Add read replicas
    Rails multi-database: reads from replica, writes to primary
    Handles: 12,000 reads/sec

  Stage 3 (100x): Shard the database
    Shard by short_code hash
    PgBouncer for connection pooling

  Stage 4 (1000x): CDN + edge caching
    CloudFront caches popular redirects
    Most requests never reach Rails
```

**Common Scaling Techniques:**

| Technique | When to Use | Ruby Tool |
|-----------|-------------|-----------|
| **Caching** | Read-heavy; hot data fits in memory | Redis via `Rails.cache` |
| **Read replicas** | Read:write > 10:1 | Rails multi-database |
| **Sharding** | Data > 10 TB or write throughput exceeded | Application-level or Citus |
| **CDN** | Static content; geo-distributed users | CloudFront + S3 |
| **Message queues** | Async processing; decouple services | Sidekiq (Redis) or Karafka (Kafka) |
| **Auto-scaling** | Variable traffic | Kubernetes HPA or AWS ECS |
| **Connection pooling** | > 100 DB connections | PgBouncer |

**Failure Scenarios:**

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| Database down | Can't create/redirect URLs | Read replicas for reads; RDS Multi-AZ failover |
| Redis down | All requests hit database | DB handles load (cache is optimization); Redis Sentinel for HA |
| Puma crash | Reduced capacity | Multiple instances behind ALB; auto-scaling |
| Sidekiq down | Background jobs queue up | Jobs persist in Redis; process when Sidekiq recovers |

---

### Step 6: Discuss Trade-offs (3-5 minutes)

**Every design decision has trade-offs.**

```
Consistency vs Availability:
  Chose eventual consistency (cache may serve stale data briefly).
  ✅ Higher availability, lower latency
  ❌ New URL might not redirect for a few seconds
  Acceptable for URL shortener; NOT for payments.

301 vs 302 Redirect:
  301: Browser caches → fewer requests → less load → but lose analytics
  302: Browser always hits us → track every click → but more load
  Trade-off: 302 if analytics matter; 301 for minimal load.

Single DB vs Sharded:
  Single: simple, ACID, easy to manage (fits at 3 TB)
  Sharded: scales horizontally, but complex (routing, cross-shard queries)
  Chose: single for now; design shard key for future.
```

---

### Complete Architecture (Final)

```
  +--------+     +------------------+     +------------------+
  | Client | ──→ | CDN (CloudFront) | ──→ | ALB              |
  +--------+     | (popular URLs)   |     +--------+---------+
                 +------------------+              |
                                          +--------v---------+
                                          | Rails API (Puma) |
                                          | (stateless,      |
                                          |  auto-scaled)    |
                                          +--------+---------+
                                                   |
                                    +--------------+--------------+
                                    |                             |
                              +-----v-----+              +-------v-------+
                              | Redis     |              | PostgreSQL /  |
                              | (cache +  |              | DynamoDB      |
                              |  Sidekiq) |              +-------+-------+
                              +-----------+                      |
                                                         +-------v-------+
                              +------------------+       | Read Replicas |
                              | Sidekiq Workers  |       +---------------+
                              | (analytics,      |
                              |  cleanup jobs)   |
                              +------------------+
                              
                              +------------------+
                              | Monitoring       |
                              | (Prometheus +    |
                              |  Grafana + Loki) |
                              +------------------+
```

---

## 37.2 Common Mistakes in HLD Interviews

---

### Mistake 1: Not Clarifying Requirements Upfront

```
❌ "Design Twitter." → *immediately draws 15 boxes*
✅ "Twitter has many features. Which should I focus on — timeline, search, notifications?"
```

---

### Mistake 2: Skipping Estimation

```
❌ "Let's use Cassandra." → "Why?" → "It's good for large-scale."
✅ "300M DAU × 10 views = 35K reads/sec. With Redis cache (90% hit rate),
    DB handles 3,500 reads/sec. PostgreSQL with read replicas handles this fine."
```

> **Ruby Context:** Ground estimates in Rails numbers: "35K reads/sec ÷ 500 req/sec per Puma instance = 70 instances. With 90% cache hit rate, only 3,500 reach the DB — 7 Puma instances handle this."

---

### Mistake 3: Designing for the Wrong Scale

```
❌ Too small: "Single SQLite" (for 100M users)
❌ Too large: "1000 sharded nodes" (for 10K users)
✅ Right-sized: "Based on estimation, 3 TB fits in one PostgreSQL. Single DB with read replicas."
```

---

### Mistake 4: Not Discussing Trade-offs

```
❌ "We'll use Cassandra." (no explanation)
✅ "Cassandra for messages because: write-heavy (100B/day), partition by conversation_id,
    eventual consistency is OK for chat. Trade-off: no ACID, no complex queries.
    For user profiles (need strong consistency), we'll use PostgreSQL."
```

---

### Mistake 5: Ignoring Failure Scenarios

```
✅ "If the primary DB goes down:
    → RDS Multi-AZ promotes standby (30-60 sec failover)
    → Read replicas continue serving reads
    → Sidekiq jobs queue in Redis until DB recovers
    → Puma returns 503 for writes during failover"
```

---

### Mistake 6: Over-Focusing on One Component

```
❌ Spend 25 minutes on database schema → no time for caching, scaling, monitoring
✅ 5 minutes per component: DB, cache, API, scaling, trade-offs, monitoring
```

---

### Mistake 7: Not Considering Security

```
✅ "Security: JWT at API Gateway, Rack::Attack rate limiting (100 req/min per user),
    HTTPS everywhere (force_ssl), input validation (sanitize URLs),
    encrypted at rest (RDS encryption, S3 SSE)"
```

---

### Mistake 8: Forgetting Monitoring

```
✅ "Observability:
    - Metrics: Yabeda + Prometheus + Grafana (request rate, error rate, P99 latency)
    - Logging: Lograge (JSON) → Loki (centralized)
    - Tracing: OpenTelemetry → Jaeger (distributed tracing)
    - Alerts: error rate > 1%, P99 > 500ms, Sidekiq queue > 10K
    - Health: /up (Rails 7.1+), /health/ready (DB + Redis check)"
```

---

### HLD Interview Checklist

```
Before designing:
  □ Clarified functional requirements
  □ Clarified non-functional requirements (latency, availability, consistency)
  □ Clarified scale (users, QPS, data volume)
  □ Estimated QPS, storage, cache size, Puma instances needed

During designing:
  □ Drew high-level architecture (Puma, Sidekiq, Redis, DB, CDN)
  □ Defined API endpoints
  □ Chose database(s) with justification
  □ Designed caching strategy (Rails.cache + Redis)
  □ Discussed scaling (replicas, sharding, CDN, PgBouncer)

After designing:
  □ Discussed failure scenarios and mitigation
  □ Discussed trade-offs (consistency vs availability, cost vs performance)
  □ Mentioned security (auth, rate limiting, encryption)
  □ Mentioned monitoring (Prometheus, Lograge, OpenTelemetry, alerts)
  □ Summarized key decisions
```

---

### Time Management (45-minute HLD Interview)

| Phase | Time | Activity |
|-------|------|----------|
| **Requirements** | 3-5 min | Clarify functional, non-functional, scale |
| **Estimation** | 3-5 min | QPS, storage, cache, Puma instances |
| **Architecture** | 5-10 min | Draw components, data flow, API design |
| **Deep Dive** | 10-15 min | Database, caching, key algorithms |
| **Scaling & Failures** | 5-10 min | Bottlenecks, scaling plan, failure scenarios |
| **Trade-offs** | 3-5 min | Consistency, cost, complexity |
| **Buffer** | 2-5 min | Security, monitoring, questions |

---

### What Interviewers Are Looking For

| Signal | How to Demonstrate |
|--------|-------------------|
| **Structured thinking** | Follow the 6-step framework consistently |
| **Scale awareness** | Do estimation; right-size the design; know Rails performance numbers |
| **Trade-off analysis** | "I chose X because... the trade-off is..." |
| **Breadth** | Cover DB, cache, API, scaling, monitoring, security |
| **Depth** | Go deep on any component when asked |
| **Communication** | Think out loud; draw diagrams; label everything |
| **Real-world awareness** | Mention failures, monitoring, PgBouncer, Sidekiq retries |

---

## Summary & Key Takeaways

| Step | Key Point |
|------|-----------|
| **1. Requirements** | Clarify functional, non-functional, and scale BEFORE designing |
| **2. Estimation** | QPS, storage, cache, Puma instances — justifies technology choices |
| **3. Architecture** | Start simple (Puma + Sidekiq + Redis + DB); show data flow; define APIs |
| **4. Deep Dive** | Database choice + schema, caching (Rails.cache), key algorithms |
| **5. Scale** | Read replicas → sharding → CDN; PgBouncer for connections; discuss failures |
| **6. Trade-offs** | Consistency vs availability, cost vs performance, complexity vs simplicity |

| Mistake | Fix |
|---------|-----|
| No requirements | Spend 3-5 minutes asking questions first |
| No estimation | Calculate QPS, storage, Puma instances — show your work |
| Wrong scale | Design for requirements, not for Google (unless it IS Google) |
| No trade-offs | Every decision has trade-offs — articulate them |
| No failure discussion | "What happens when X goes down?" for every component |
| Over-focus one component | 5 minutes per component; cover breadth |
| No security | Auth (JWT/Devise), rate limiting (Rack::Attack), encryption |
| No monitoring | Yabeda/Prometheus, Lograge, OpenTelemetry, alerts, health checks |

---

## Interview Tips for Module 37

1. **Follow the framework** — requirements → estimation → architecture → deep dive → scale → trade-offs
2. **Write requirements on the board** — this is your contract with the interviewer
3. **Show estimation work** — include Rails-specific numbers (Puma throughput, Sidekiq capacity)
4. **Start simple** — draw Puma + Sidekiq + Redis + DB first; add complexity as needed
5. **Justify every choice** — "I chose PostgreSQL because..." not just "I'll use PostgreSQL"
6. **Discuss trade-offs** — for every decision, explain what you gain and what you give up
7. **Think about failures** — "What if the database goes down?" for every component
8. **Mention security** — JWT, Rack::Attack, force_ssl, input validation
9. **Mention monitoring** — Prometheus/Yabeda, Lograge, OpenTelemetry, health checks
10. **Think out loud** — the interviewer wants to see your reasoning
11. **Know your Rails numbers** — Puma: 200-1,000 req/sec; Sidekiq: 100-500 jobs/sec; 200-500 MB/process; PgBouncer at 100+ connections
12. **Practice** — do 20+ HLD problems (Modules 33-35) until the framework is automatic