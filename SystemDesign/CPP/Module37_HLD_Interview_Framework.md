# Module 37: HLD Interview Framework

> High-Level Design (HLD) interviews test your ability to design the architecture of a large-scale distributed system — choosing the right databases, caching strategies, messaging patterns, and scaling techniques. Unlike LLD (which focuses on classes and code), HLD focuses on **components, data flow, and trade-offs**. This module provides a battle-tested framework for approaching any HLD interview problem.

---

## 37.1 Step-by-Step Approach

> This 6-step framework works for every HLD problem. Follow it consistently and you'll never get lost in an interview. We'll use "Design a URL Shortener" as a running example.

---

### Step 1: Clarify Requirements (3-5 minutes)

**The most important step.** A great design for the wrong requirements is a failed interview. Ask questions to understand what you're building and at what scale.

**Functional Requirements — "What does the system do?"**

```
Ask:
  "What are the core features?"
  "Should users be able to X?"
  "Do we need feature Y?"
  "What's in scope vs out of scope?"

URL Shortener Example:
  ✅ In scope:
    - Shorten a long URL → short URL
    - Redirect short URL → long URL
    - Custom short URLs (optional)
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
| **Availability** | "Can the system be down? For how long?" | 99.99% (redirect must always work) |
| **Consistency** | "Is eventual consistency OK?" | Yes (a few seconds delay for new URLs is fine) |
| **Durability** | "Can we lose data?" | No — URLs must be durable |

**Scale — "How big is this?"**

```
Ask:
  "How many users?"
  "How many [items] per day/month?"
  "What's the read:write ratio?"
  "How much data do we need to store?"
  "How long do we retain data?"

URL Shortener:
  - 100M new URLs per month
  - 10:1 read:write ratio → 1B redirects per month
  - Store for 5 years
  - Total URLs: 6 billion
```

**Key Rule:** Write down the requirements on the whiteboard/screen. This becomes your contract with the interviewer — you'll design for exactly these requirements.

---

### Step 2: Back-of-the-Envelope Estimation (3-5 minutes)

**Do the math before designing.** Estimation drives architectural decisions — do you need sharding? How big should the cache be? How many servers?

```
URL Shortener Estimation:

  Write QPS:
    100M URLs/month ÷ (30 × 86,400) ≈ 40 writes/sec
    Peak (3x): ~120 writes/sec

  Read QPS:
    1B redirects/month ÷ (30 × 86,400) ≈ 400 reads/sec
    Peak (3x): ~1,200 reads/sec

  Storage (5 years):
    6B URLs × 500 bytes (short URL + long URL + metadata) = 3 TB
    → Fits on a single database server (no sharding needed yet)

  Bandwidth:
    Read: 1,200 req/sec × 500 bytes = 600 KB/sec (trivial)

  Cache (20% of daily URLs):
    3.3M daily URLs × 20% × 500 bytes = 330 MB
    → Easily fits in a single Redis instance

  Conclusion:
    - Scale is moderate — no need for complex sharding
    - Cache is small — single Redis instance
    - Read-heavy (10:1) — caching will be very effective
```

**Show your work.** Write the calculations on the whiteboard. The interviewer wants to see your reasoning, not just the final number.

**Estimation Cheat Sheet:**

| Metric | Formula |
|--------|---------|
| QPS | items_per_day / 86,400 (use 10^5 for easy math) |
| Peak QPS | Average QPS × 3 |
| Storage | items × size × retention_years × 365 |
| Cache | daily_active_items × 20% × item_size |
| Servers | Peak QPS / QPS_per_server |
| Bandwidth | QPS × response_size |

---

### Step 3: High-Level Architecture (5-10 minutes)

**Draw the big picture.** Show the main components and how data flows between them. Start simple — you'll add complexity later.

```
URL Shortener — Initial Architecture:

  +--------+     +-------------+     +------------------+     +----------+
  | Client | ──→ | API Gateway | ──→ | URL Service      | ──→ | Database |
  +--------+     | (rate limit)|     | (shorten/redirect)|    +----------+
                 +-------------+     +--------+---------+
                                              |
                                     +--------v---------+
                                     | Cache (Redis)    |
                                     +------------------+
```

**What to Include:**
- Client (browser, mobile app)
- Load balancer / API Gateway
- Application service(s)
- Database(s)
- Cache
- Message queue (if async processing is needed)
- CDN (if serving static content)

**API Design (key endpoints):**

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
- Label the arrows (HTTP, gRPC, async/event)
- Don't over-complicate — start with 4-6 components
- You'll add more components in the deep dive

---

### Step 4: Deep Dive into Components (10-15 minutes)

**This is the meat of the interview.** The interviewer will ask you to go deeper on specific components. Be ready to discuss:

**Database Schema and Choice:**

```
URL Shortener Schema:
  Table: urls
    short_code  VARCHAR(7) PRIMARY KEY  — "abc123"
    long_url    VARCHAR(2048) NOT NULL  — original URL
    user_id     BIGINT                  — who created it (optional)
    created_at  TIMESTAMP               — when created
    expires_at  TIMESTAMP               — when it expires (nullable)
    click_count BIGINT DEFAULT 0        — analytics

Database Choice:
  Option 1: PostgreSQL — ACID, reliable, good for moderate scale
  Option 2: DynamoDB — key-value access pattern, auto-scaling, managed
  
  Recommendation: DynamoDB (access pattern is pure key-value lookup by short_code)
```

**Caching Strategy:**

```
Cache-Aside Pattern:
  1. Client requests GET /abc123
  2. Check Redis: GET url:abc123
     → HIT: return long_url (fast!)
     → MISS: query database → store in Redis (TTL=24h) → return

Why cache works well here:
  - Read-heavy (10:1 ratio)
  - Hot URLs are accessed repeatedly (Pareto — 20% of URLs get 80% of traffic)
  - Cache is small (330 MB — fits in one Redis instance)
  - Expected hit ratio: 90%+ (most redirects are for popular URLs)
```

**Key Algorithm (Short URL Generation):**

```
Approach: Counter + Base62 Encoding
  1. Atomic counter: 1, 2, 3, ... (auto-increment or distributed counter)
  2. Encode counter in Base62: [a-z, A-Z, 0-9]
     Counter 1 → "1"
     Counter 1000 → "g8"
     Counter 56,800,235,584 → "zzzzzz" (6 chars, 56.8B combinations)
  3. 7 characters: 62^7 = 3.5 trillion combinations (more than enough)

Why Base62?
  - URL-safe characters only (no special chars)
  - Short (7 chars vs 32 chars for UUID)
  - Not guessable (if using a non-sequential counter or hash)
```

**Data Partitioning (if needed at scale):**

```
At 3 TB, a single database handles this fine.
If we grow to 30 TB:
  Shard by short_code (hash-based sharding)
  → Even distribution
  → Single-shard lookups (no cross-shard queries)
```

---

### Step 5: Address Bottlenecks & Scale (5-10 minutes)

**Identify what breaks first as traffic grows, and how to fix it.**

```
URL Shortener — Scaling Progression:

  Stage 1 (current): Single DB + Redis cache
    Handles: 1,200 reads/sec, 120 writes/sec
    Bottleneck: none at this scale

  Stage 2 (10x traffic): Add read replicas
    Handles: 12,000 reads/sec
    Read replicas handle redirect queries
    Primary handles writes

  Stage 3 (100x traffic): Shard the database
    Handles: 120,000 reads/sec
    Shard by short_code hash
    Each shard handles a subset of URLs

  Stage 4 (1000x traffic): CDN + edge caching
    Popular URLs cached at CDN edge (CloudFront)
    Most redirects never reach our servers
```

**Common Scaling Techniques to Mention:**

| Technique | When to Use |
|-----------|-------------|
| **Caching (Redis)** | Read-heavy workloads; hot data fits in memory |
| **Read replicas** | Read:write ratio > 10:1 |
| **Database sharding** | Data too large for one server; write throughput exceeded |
| **CDN** | Static content; geographically distributed users |
| **Message queues** | Async processing; decouple services; absorb traffic spikes |
| **Load balancing** | Multiple app server instances |
| **Auto-scaling** | Variable traffic patterns (peak hours, events) |

**Failure Scenarios to Discuss:**

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| Database down | Can't create or redirect URLs | Read replicas for reads; failover for writes |
| Cache down | All requests hit database | Database can handle the load (cache is optimization, not requirement); cache cluster with replicas |
| App server crash | Reduced capacity | Multiple instances behind load balancer; auto-scaling replaces failed instances |
| Network partition | Some users can't reach the service | Multi-region deployment; DNS failover |

---

### Step 6: Discuss Trade-offs (3-5 minutes)

**Every design decision has trade-offs.** Discussing them shows maturity and depth.

**Consistency vs Availability:**

```
URL Shortener:
  We chose eventual consistency (cache may serve stale data for a few seconds).
  
  Trade-off:
    ✅ Higher availability (cache serves even if DB is slow)
    ✅ Lower latency (cache hit = sub-millisecond)
    ❌ A newly created URL might not be immediately accessible (cache miss → DB)
    
  Acceptable because: a few seconds delay for a new URL is fine.
  NOT acceptable for: financial transactions, inventory counts.
```

**Latency vs Throughput:**

```
301 (permanent redirect) vs 302 (temporary redirect):
  301: Browser caches the redirect → fewer requests to our server → higher throughput
       But: we lose visibility into click analytics
  302: Browser always hits our server → we can track every click
       But: more load on our servers
  
  Trade-off: 302 if analytics matter; 301 if we want to minimize server load.
```

**Cost vs Performance:**

```
Cache everything vs cache selectively:
  Cache everything: highest hit ratio, lowest latency
    But: more Redis memory = higher cost
  Cache top 20%: good hit ratio (Pareto), lower cost
    But: 20% of requests still hit the database
  
  We chose: cache top 20% (330 MB) — excellent cost/performance balance.
```

**Complexity vs Simplicity:**

```
Single database vs sharded database:
  Single DB: simple, easy to manage, ACID transactions
    But: limited to ~10 TB, single point of failure
  Sharded DB: scales horizontally, higher throughput
    But: complex (routing, cross-shard queries, resharding)
  
  We chose: single DB for now (3 TB fits easily).
  Plan: shard when we approach 10 TB (design the shard key now).
```

---

### Complete Architecture (Final)

```
  +--------+     +------------------+     +------------------+
  | Client | ──→ | CDN (CloudFront) | ──→ | Load Balancer    |
  +--------+     | (cache popular   |     | (ALB)            |
                 |  redirects)      |     +--------+---------+
                 +------------------+              |
                                          +--------v---------+
                                          | URL Service      |
                                          | (stateless,      |
                                          |  auto-scaled)    |
                                          +--------+---------+
                                                   |
                                    +--------------+--------------+
                                    |                             |
                              +-----v-----+              +-------v-------+
                              | Redis     |              | DynamoDB      |
                              | Cache     |              | (short_code → |
                              | (hot URLs)|              |  long_url)    |
                              +-----------+              +---------------+
                              
                              +------------------+
                              | Analytics        |
                              | (Kafka → S3 →    |
                              |  Athena/Spark)   |
                              +------------------+
                              
                              +------------------+
                              | Monitoring       |
                              | (Prometheus +    |
                              |  Grafana)        |
                              +------------------+
```

---


## 37.2 Common Mistakes in HLD Interviews

> These mistakes are the difference between a "hire" and a "no hire." Avoid them and you'll stand out from most candidates.

---

### Mistake 1: Not Clarifying Requirements Upfront

```
❌ Bad:
  Interviewer: "Design Twitter."
  Candidate: *immediately draws 15 boxes and starts talking about Kafka*
  
  10 minutes later...
  Interviewer: "I actually wanted you to focus on the search feature."
  Candidate: *wasted 10 minutes on the wrong thing*

✅ Good:
  Interviewer: "Design Twitter."
  Candidate: "Twitter has many features — posting, timeline, search, notifications,
    messaging, trending topics. Which aspects should I focus on?"
  Interviewer: "Focus on the news feed / timeline."
  Candidate: "Got it. Let me clarify the requirements for the timeline..."
```

**Why it matters:** You have 35-45 minutes. Spending 3 minutes on requirements saves you from wasting 15 minutes on the wrong design.

---

### Mistake 2: Skipping Estimation

```
❌ Bad:
  "Let's use Cassandra for the database."
  Interviewer: "Why Cassandra?"
  "Because it's good for large-scale systems."
  Interviewer: "How large is this system?"
  "Uh... I'm not sure."

✅ Good:
  "Let me estimate the scale first.
   300M DAU × 10 timeline views/day = 3B reads/day ≈ 35K reads/sec.
   At this scale, a single PostgreSQL with read replicas and Redis cache
   handles it fine. We don't need Cassandra yet."
```

**Why it matters:** Estimation justifies your technology choices. Without it, you're guessing. With it, you're making informed decisions.

---

### Mistake 3: Designing for the Wrong Scale

```
❌ Too small:
  "I'll use a single SQLite database."
  (For a system with 100M users? That won't work.)

❌ Too large:
  "I'll shard across 1000 database nodes with a custom consensus protocol."
  (For a system with 10K users? That's massive over-engineering.)

✅ Right-sized:
  "Based on my estimation, we have 35K reads/sec and 3 TB of data.
   A single PostgreSQL primary with 3 read replicas and a Redis cache
   handles this comfortably. If we grow 10x, we'll add sharding."
```

**Why it matters:** Design for the requirements, not for Google's scale (unless you're designing for Google's scale). Over-engineering is as bad as under-engineering.

---

### Mistake 4: Not Discussing Trade-offs

```
❌ Bad:
  "We'll use Cassandra."
  (No explanation of why, or what you're giving up.)

✅ Good:
  "We'll use Cassandra for the message store because:
   - Write-heavy workload (100B messages/day) — Cassandra excels at writes
   - Partition by conversation_id — efficient range queries for chat history
   - Eventual consistency is acceptable for chat (messages may arrive slightly out of order)
   
   Trade-off: We lose ACID transactions and complex queries.
   For the user profile data (which needs strong consistency), we'll use PostgreSQL."
```

**Why it matters:** Every technology choice has trade-offs. Articulating them shows you understand the technology deeply, not just its name.

---

### Mistake 5: Ignoring Failure Scenarios

```
❌ Bad:
  *draws a beautiful architecture with no mention of what happens when things fail*

✅ Good:
  "What happens if the primary database goes down?
   → We have a synchronous replica in another AZ that gets promoted automatically.
   → Read replicas continue serving reads during failover.
   → Failover takes 30-60 seconds — during this time, writes fail but reads work.
   
   What happens if Redis goes down?
   → All requests fall through to the database.
   → The database can handle the load (cache is an optimization, not a requirement).
   → Redis Sentinel automatically promotes a replica."
```

**Why it matters:** Production systems fail. Showing that you've thought about failure scenarios demonstrates real-world experience.

---

### Mistake 6: Over-Focusing on One Component

```
❌ Bad:
  Candidate spends 25 minutes designing the database schema in extreme detail.
  No time left for caching, scaling, monitoring, or trade-offs.
  
  Interviewer wanted to see the big picture, not a perfect schema.

✅ Good:
  Spend 5 minutes on each major component:
  - Database: schema + choice + partitioning strategy (5 min)
  - Caching: strategy + cache size + invalidation (5 min)
  - API design: key endpoints + data flow (5 min)
  - Scaling: bottlenecks + solutions (5 min)
  - Trade-offs: consistency, availability, cost (5 min)
```

**Why it matters:** HLD interviews test **breadth** — can you design a complete system? Going too deep on one component means you miss others.

---

### Mistake 7: Not Considering Security

```
❌ Bad:
  *no mention of authentication, authorization, encryption, or rate limiting*

✅ Good:
  "For security:
   - Authentication: JWT tokens validated at the API Gateway
   - Rate limiting: 100 requests/minute per user at the API Gateway
   - HTTPS everywhere (TLS termination at the load balancer)
   - Input validation: sanitize URLs to prevent XSS/injection
   - Private data encrypted at rest (S3 SSE, RDS encryption)"
```

**Why it matters:** Security is a basic requirement for any production system. Mentioning it (even briefly) shows completeness.

---

### Mistake 8: Forgetting Monitoring and Alerting

```
❌ Bad:
  *no mention of how you'd know if the system is healthy or broken*

✅ Good:
  "For observability:
   - Metrics: Prometheus + Grafana (request rate, error rate, latency P50/P95/P99)
   - Logging: structured JSON logs → ELK or Loki (centralized, searchable)
   - Tracing: OpenTelemetry → Jaeger (distributed tracing across services)
   - Alerting: alert on error rate > 1%, P99 latency > 500ms, disk > 80%
   - Health checks: /health/live (liveness), /health/ready (readiness)"
```

**Why it matters:** A system without monitoring is a system you can't operate. Mentioning observability shows you think about the full lifecycle, not just the happy path.

---

### HLD Interview Checklist

```
Before designing:
  □ Clarified functional requirements (what the system does)
  □ Clarified non-functional requirements (latency, availability, consistency)
  □ Clarified scale (users, QPS, data volume)
  □ Estimated QPS, storage, bandwidth, cache size

During designing:
  □ Drew high-level architecture (4-6 components)
  □ Defined API endpoints (REST/gRPC)
  □ Chose database(s) with justification
  □ Designed caching strategy
  □ Discussed data partitioning (if needed)
  □ Addressed scaling (read replicas, sharding, CDN, queues)

After designing:
  □ Discussed failure scenarios and mitigation
  □ Discussed trade-offs (consistency vs availability, cost vs performance)
  □ Mentioned security (auth, rate limiting, encryption)
  □ Mentioned monitoring (metrics, logs, traces, alerts)
  □ Summarized the design and key decisions
```

---

### Time Management (45-minute HLD Interview)

| Phase | Time | Activity |
|-------|------|----------|
| **Requirements** | 3-5 min | Clarify functional, non-functional, scale |
| **Estimation** | 3-5 min | QPS, storage, bandwidth, cache size |
| **High-Level Architecture** | 5-10 min | Draw components, data flow, API design |
| **Deep Dive** | 10-15 min | Database, caching, key algorithms, partitioning |
| **Scaling & Failures** | 5-10 min | Bottlenecks, scaling plan, failure scenarios |
| **Trade-offs** | 3-5 min | Consistency vs availability, cost, complexity |
| **Buffer** | 2-5 min | Security, monitoring, questions |

**Key Insight:** The interviewer evaluates your **thought process** and **trade-off analysis**, not whether you draw the "correct" architecture. There is no single correct answer — there are many valid designs with different trade-offs.

---

### What Interviewers Are Looking For

| Signal | What It Means | How to Demonstrate |
|--------|-------------|-------------------|
| **Structured thinking** | Can you break down a complex problem systematically? | Follow the 6-step framework consistently |
| **Scale awareness** | Do you understand what changes at different scales? | Do estimation; right-size the design |
| **Trade-off analysis** | Can you articulate the pros and cons of each decision? | "I chose X because... the trade-off is..." |
| **Breadth** | Can you design a complete system, not just one component? | Cover DB, cache, API, scaling, monitoring |
| **Depth** | Can you go deep on a specific component when asked? | Be ready to explain any component in detail |
| **Communication** | Can you explain your design clearly? | Think out loud; draw diagrams; label everything |
| **Real-world awareness** | Do you know how production systems actually work? | Mention failure scenarios, monitoring, security |

---

## Summary & Key Takeaways

| Step | Key Point |
|------|-----------|
| **1. Requirements** | Clarify functional, non-functional, and scale BEFORE designing |
| **2. Estimation** | QPS, storage, bandwidth, cache — justifies technology choices |
| **3. Architecture** | Start simple (4-6 components); show data flow; define APIs |
| **4. Deep Dive** | Database choice + schema, caching strategy, key algorithms |
| **5. Scale** | Identify bottlenecks; plan scaling progression; discuss failures |
| **6. Trade-offs** | Consistency vs availability, cost vs performance, complexity vs simplicity |

| Mistake | Fix |
|---------|-----|
| No requirements clarification | Spend 3-5 minutes asking questions first |
| No estimation | Calculate QPS, storage, cache size — show your work |
| Wrong scale | Design for the requirements, not for Google (unless it IS Google) |
| No trade-offs | Every decision has trade-offs — articulate them |
| No failure discussion | "What happens when X goes down?" — answer for every component |
| Over-focus on one component | Spend 5 minutes per component; cover breadth |
| No security | Mention auth, rate limiting, encryption, input validation |
| No monitoring | Mention metrics, logs, traces, alerts |

---

## Interview Tips for Module 37

1. **Follow the framework** — requirements → estimation → architecture → deep dive → scale → trade-offs
2. **Write requirements on the board** — this is your contract with the interviewer
3. **Show estimation work** — write the math; the process matters more than exact numbers
4. **Start simple** — draw 4-6 boxes first; add complexity as needed
5. **Justify every choice** — "I chose PostgreSQL because..." not just "I'll use PostgreSQL"
6. **Discuss trade-offs** — for every decision, explain what you gain and what you give up
7. **Think about failures** — "What if the database goes down?" for every component
8. **Mention security** — auth, rate limiting, encryption (even briefly)
9. **Mention monitoring** — metrics, logs, traces, alerts (even briefly)
10. **Think out loud** — the interviewer wants to see your reasoning, not just the final diagram
11. **Ask for feedback** — "Does this make sense so far?" "Should I go deeper on any component?"
12. **Practice** — do 20+ HLD problems (Modules 33-35) until the framework is automatic
