# Module 38: Common Mistakes & Tips

> This final module consolidates the most important advice for system design interviews — the general tips that apply to every problem, the resources to study, and a structured study plan to go from zero to interview-ready. Think of this as your pre-interview checklist and long-term preparation guide.

> **Ruby Context:** This module adapts the general tips, default tech stack, and study plan for Ruby/Rails engineers. The core principles (think out loud, discuss trade-offs, start simple) are universal. The Ruby-specific additions cover: Rails performance numbers to memorize, Ruby-ecosystem tools as your default stack, Ruby/Rails-specific books and resources, and a study plan that accounts for Ruby's OOP model (modules, duck typing, mixins vs interfaces/abstract classes).

---

## 38.1 General Tips

---

### Tip 1: Practice Drawing Diagrams

| Diagram Type | When to Use | Tools |
|-------------|-------------|-------|
| **Architecture diagram** | HLD — show components and data flow | draw.io, Excalidraw, whiteboard |
| **Class diagram** | LLD — show classes, modules, relationships | draw.io, PlantUML, whiteboard |
| **Sequence diagram** | Show request flow between components | draw.io, PlantUML |
| **Data flow diagram** | Show how data moves through the system | draw.io, whiteboard |
| **ER diagram** | Show database schema and relationships | draw.io, dbdiagram.io |

**Diagram Best Practices:**

```
✅ Good Diagram:
  - Clear labels on every box and arrow
  - Data flow direction shown with arrows
  - Protocol labeled (HTTP, Sidekiq/async, Kafka, WebSocket)
  - Components grouped (frontend, Rails API, workers, data layer)
  - 6-10 boxes for HLD, 8-15 classes for LLD

❌ Bad Diagram:
  - Unlabeled boxes
  - No arrows (how does data flow?)
  - Too many components (can't explain in time)
```

---

### Tip 2: Think Out Loud

```
✅ Good:
  "I'm thinking about the database choice. We have a key-value access pattern —
   look up by short_code. DynamoDB is a natural fit. The alternative is PostgreSQL
   with ActiveRecord, which gives us more query flexibility and is simpler to start
   with. Given our moderate scale (3 TB, 1,200 reads/sec), PostgreSQL with Redis
   caching handles this easily. I'll start with PostgreSQL and note that we'd
   migrate to DynamoDB if we hit 10x scale."

❌ Bad:
  *30 seconds of silence*
  "I'll use PostgreSQL."
```

---

### Tip 3: Start Simple, Then Add Complexity

```
Iteration 1 (simplest):
  Client → Rails API (Puma) → PostgreSQL
  "This handles basic functionality. Now let me add caching..."

Iteration 2 (add caching):
  Client → Rails API → Redis (Rails.cache) → PostgreSQL
  "This handles read-heavy workload. Now scaling..."

Iteration 3 (add scaling):
  Client → ALB → Rails API (3 instances) → Redis → PostgreSQL (primary + 2 replicas)
  "This handles current scale. For async work..."

Iteration 4 (add async + observability):
  + Sidekiq workers (background jobs)
  + Kafka/Karafka (event streaming)
  + CDN (CloudFront for static assets)
  + Prometheus + Grafana (monitoring)
```

---

### Tip 4: Always Discuss Trade-offs

**The Trade-off Formula:**

```
"I chose [X] because [reason]. The trade-off is [what we give up].
 This is acceptable because [why it's OK for our use case]."

Example:
  "I chose eventual consistency for the timeline cache because it gives us
   lower latency and higher availability. The trade-off is that a user might
   not see a new tweet for a few seconds. This is acceptable because social
   media feeds don't require real-time consistency."
```

**Common Trade-offs:**

| Trade-off | When to Discuss |
|-----------|----------------|
| **Consistency vs Availability** | Database choice, caching, replication |
| **Latency vs Throughput** | Sync vs async (Sidekiq), batching |
| **Cost vs Performance** | Cache size, replicas, CDN coverage |
| **Complexity vs Simplicity** | Monolith vs microservices, single DB vs sharded |
| **Read vs Write optimization** | Normalization vs denormalization, CQRS |
| **Ruby vs other languages** | When Ruby's performance is/isn't sufficient |

---

### Tip 5: Know Your Numbers

**Latency Numbers:**

```
L1 cache:           ~1 ns
RAM:                ~100 ns
SSD random read:    ~16 μs
Network (same DC):  ~0.5 ms
HDD seek:           ~4 ms
Cross-continent:    ~150 ms
```

**Scale Numbers:**

```
Seconds per day:    86,400 ≈ 10^5
1 KB = 10^3, 1 MB = 10^6, 1 GB = 10^9, 1 TB = 10^12, 1 PB = 10^15
```

**Ruby/Rails-Specific Numbers (MEMORIZE THESE):**

```
Puma throughput:         200-1,000 req/sec per instance (depends on complexity)
Puma memory per worker:  200-500 MB
Puma workers:            2-4 per instance (= CPU cores)
Puma threads per worker: 5 (default)
Sidekiq throughput:      100-500 jobs/sec per process (moderate jobs)
Sidekiq memory:          200-500 MB per process
Rails request latency:   5-20 ms (simple), 20-100 ms (moderate), 100-500 ms (complex)
ActiveRecord query:      1-5 ms (indexed), 5-50 ms (complex join)
Redis GET/SET:           0.5-2 ms (from Rails)
External HTTP call:      50-500 ms
PgBouncer needed at:     > 100 total DB connections
Single PostgreSQL:       5K-20K QPS (simple queries)
Single Redis:            100K+ ops/sec
```

**Capacity Numbers:**

```
Single PostgreSQL:  5K-20K QPS (depends on query)
Single Redis:       100K+ ops/sec
Single Kafka broker: 100K+ messages/sec
Single Puma instance: 200-1,000 QPS
CDN edge:           millions of requests/sec (distributed)
```

---

### Tip 6: Study Real-World Architectures

| Company | What to Study | Key Insight |
|---------|-------------|-------------|
| **Shopify** | Rails at scale, modular monolith, Packwerk | You CAN scale Rails to massive traffic |
| **GitHub** | Rails monolith → services, Actions | Gradual extraction, not big-bang rewrite |
| **Basecamp/37signals** | Rails, Kamal, Hotwire, Solid Queue | Rails can do more than you think without microservices |
| **Netflix** | Microservices, CDN, chaos engineering | Resilience patterns |
| **Uber** | Geospatial matching, real-time systems | Location tracking at scale |
| **Twitter** | News feed, fan-out, timeline caching | Hybrid fan-out |
| **Stripe** | Payment processing, idempotency | Double-entry ledger, idempotency keys |
| **Airbnb** | Search, payments, SOA (originally Rails) | Service extraction from Rails monolith |
| **Instacart** | Rails at scale, Sidekiq, real-time | High-throughput Rails with background processing |
| **Cookpad** | Large Rails app, Ruby performance | Ruby optimization techniques |

**Where to Find Architecture Details:**
- Company engineering blogs
- Conference talks (RailsConf, RubyConf, RubyKaigi, QCon)
- Research papers (Dynamo, Spanner, Kafka, Raft)
- Books (DDIA by Martin Kleppmann — the single best resource)

---

### Tip 7: Practice Under Time Pressure

```
Real interview: 45 minutes for one problem
  - 5 min: requirements + estimation
  - 10 min: high-level architecture
  - 15 min: deep dive
  - 10 min: scaling + trade-offs
  - 5 min: buffer

Practice routine:
  1. Pick a problem (Modules 33-35)
  2. Set a 45-minute timer
  3. Design on paper/whiteboard (not just in your head)
  4. Record yourself explaining (or practice with a friend)
  5. Review: did you cover requirements, estimation, architecture, deep dive, trade-offs?
  6. Repeat 2-3 times per week for 4-6 weeks
```

---

### Tip 8: Have a "Go-To" Tech Stack

**Ruby/Rails Default Stack:**

| Component | Default Choice | When to Switch |
|-----------|---------------|---------------|
| **Web Framework** | Rails API (Puma) | Sinatra/Roda for tiny services; Go for CPU-intensive |
| **SQL Database** | PostgreSQL | Aurora for AWS-native; MySQL if team prefers |
| **NoSQL (key-value)** | DynamoDB (persistent) | Cassandra for write-heavy time-series |
| **Cache** | Redis (via Rails.cache) | Memcached only for simplest caching |
| **Background Jobs** | Sidekiq (Redis-backed) | Shoryuken (SQS) for AWS-native; Karafka (Kafka) for streaming |
| **Message Queue** | Kafka (via Karafka) | SQS for simple queues; RabbitMQ (Bunny/Sneakers) for routing |
| **Search** | Elasticsearch (Searchkick) | PostgreSQL full-text for simple search |
| **Object Storage** | S3 (ActiveStorage or aws-sdk-s3) | GCS for GCP |
| **CDN** | CloudFront | Cloudflare for non-AWS |
| **WebSockets** | AnyCable (production) | Action Cable (simple/low-scale) |
| **Load Balancer** | ALB | Nginx for on-prem; Traefik with Kamal |
| **Deployment** | Kamal (simple) or EKS (complex) | Heroku (prototype); ECS (medium) |
| **Monitoring** | Prometheus + Grafana (Yabeda) | Datadog for managed; Scout/Skylight for Rails APM |
| **CI/CD** | GitHub Actions | GitLab CI |
| **Feature Flags** | Flipper | LaunchDarkly for enterprise |
| **Rate Limiting** | Rack::Attack | Custom Redis Lua for advanced |
| **Auth** | Devise + OmniAuth | JWT (devise-jwt) for API-only |
| **Authorization** | Pundit | CanCanCan for simpler RBAC |

**Why a default stack helps:** Zero time on "which tool?" → more time on actual design. Switch only with justification: "My default is PostgreSQL, but for this write-heavy workload with 100B messages/day, Cassandra is better because..."

---

## 38.2 Resources

---

### Books (Ranked by Priority for Ruby Engineers)

| Priority | Book | Author | What You'll Learn |
|----------|------|--------|------------------|
| **#1 (Must Read)** | Designing Data-Intensive Applications (DDIA) | Martin Kleppmann | The bible of distributed systems — replication, partitioning, consistency |
| **#2 (Must Read)** | System Design Interview Vol 1 & 2 | Alex Xu | Step-by-step HLD solutions; great for interview prep |
| **#3** | Practical Object-Oriented Design in Ruby (POODR) | Sandi Metz | OOP principles in Ruby; essential for LLD interviews |
| **#4** | 99 Bottles of OOP | Sandi Metz | Refactoring and design patterns through a single problem |
| **#5** | Design Patterns in Ruby | Russ Olsen | GoF patterns adapted for Ruby's dynamic nature |
| **#6** | Ruby Under a Microscope | Pat Shaughnessy | How Ruby works internally (GC, objects, VM) |
| **#7** | The Rails Way / Rails guides | Various | Rails conventions, ActiveRecord, caching, background jobs |
| **#8** | Clean Code / Clean Architecture | Robert C. Martin | Architecture principles (language-agnostic) |

**Reading Order for Interview Prep:**
1. DDIA (chapters 1-9 for HLD fundamentals)
2. POODR (for LLD/OOP thinking in Ruby)
3. System Design Interview Vol 1 (practice problems)
4. Design Patterns in Ruby (for LLD pattern knowledge)
5. System Design Interview Vol 2 (more practice)

---

### Online Resources

**Free Resources:**

| Resource | Type | What You'll Learn |
|----------|------|------------------|
| **System Design Primer** (GitHub) | Guide | Comprehensive system design overview |
| **ByteByteGo** (Alex Xu) | Newsletter + YouTube | Visual system design explanations |
| **Shopify Engineering Blog** | Articles | Rails at massive scale |
| **37signals Dev Blog** | Articles | Rails philosophy, Kamal, Solid Queue |
| **Ruby Weekly** | Newsletter | Ruby ecosystem updates |
| **RailsConf / RubyConf talks** | Videos | Architecture talks from Ruby experts |
| **Martin Fowler's Blog** | Articles | Architecture patterns, refactoring |

**Paid Resources:**

| Resource | Type | What You'll Learn |
|----------|------|------------------|
| **Grokking the System Design Interview** | Course | Structured HLD problems |
| **Grokking the OOD Interview** | Course | Structured LLD problems |
| **GoRails / Drifting Ruby** | Video courses | Rails-specific patterns and techniques |
| **Upcase (thoughtbot)** | Video courses | Ruby design patterns, testing, refactoring |

---

### Engineering Blogs to Follow (Ruby-Relevant)

| Company | Blog | Notable Topics |
|---------|------|---------------|
| **Shopify** | shopify.engineering | Rails at scale, Packwerk, modular monolith |
| **GitHub** | github.blog/engineering | Rails → services migration, Actions |
| **37signals** | dev.37signals.com | Kamal, Solid Queue, Rails 8, Hotwire |
| **Instacart** | tech.instacart.com | Rails performance, Sidekiq at scale |
| **Gusto** | engineering.gusto.com | Rails, microservices, payments |
| **Stripe** | stripe.com/blog/engineering | Payments, API design, Ruby services |
| **Netflix** | netflixtechblog.com | Microservices, resilience (concepts apply) |
| **Uber** | eng.uber.com | Geospatial, real-time (concepts apply) |

---

## 38.3 Study Plan

---

### 26-Week Comprehensive Plan (Ruby/Rails)

| Week | Focus | Modules | Key Activities |
|------|-------|---------|---------------|
| **1-2** | Ruby Fundamentals | 1 | Blocks, procs, lambdas, modules, Enumerable, metaprogramming |
| **3-4** | OOP in Ruby + SOLID | 2-3 | Classes, mixins, duck typing, SOLID with Ruby examples |
| **5-6** | Design Patterns (Ruby) | 4-6 | Strategy (lambdas), Observer, Factory, Singleton, Decorator |
| **7** | UML + Advanced OOP | 7-8 | Class diagrams, composition vs inheritance, modules |
| **8** | Concurrency + DS/Algo | 9-10 | Threads, Mutex, Ractor; LRU cache, rate limiter in Ruby |
| **9** | Error Handling + Testing | 11 | RSpec, exception handling, logging |
| **10-11** | LLD Practice (Easy + Medium) | 12-13 | Parking lot, vending machine, elevator, library (Ruby) |
| **12** | LLD Practice (Hard) | 14 | Chess, file system, spreadsheet (Ruby) |
| **13-14** | Networking + APIs | 15-17 | HTTP, REST, GraphQL, gRPC, CDN, API Gateway |
| **15-16** | Databases (SQL + NoSQL) | 18-20 | PostgreSQL, ActiveRecord, DynamoDB, Cassandra, sharding |
| **17** | Caching + Scalability | 21-22 | Rails.cache, Redis, CQRS, event sourcing |
| **18** | Load Balancing + Message Queues | 23-24 | ALB, Sidekiq, Kafka/Karafka, event-driven |
| **19** | Microservices + Distributed Systems | 25-26 | Service discovery, saga, circuit breaker (Stoplight), consensus |
| **20** | Advanced Distributed + Storage | 27-28 | Consistent hashing, gossip, CRDTs, S3, serialization |
| **21** | Security + Monitoring + DevOps | 29-31 | Devise, JWT, Rack::Attack, Prometheus, Docker, K8s, Kamal |
| **22** | Estimation + HLD Easy | 32-33 | Back-of-envelope (Rails numbers), URL shortener, rate limiter |
| **23-24** | HLD Medium | 34 | Twitter, Instagram, WhatsApp, Uber, notifications |
| **25** | HLD Hard | 35 | YouTube, Google Docs, payment system, leaderboard |
| **26** | Interview Frameworks + Mocks | 36-38 | LLD framework, HLD framework, mock interviews |

---

### 12-Week Accelerated Plan (Already Know Ruby/Rails)

| Week | Focus | Modules | Key Activities |
|------|-------|---------|---------------|
| **1** | SOLID + Design Patterns (Ruby) | 3-6 | POODR review; Strategy, Observer, Factory in Ruby |
| **2** | LLD Practice | 12-14 | 6-8 LLD problems in Ruby (parking lot through chess) |
| **3** | Networking + APIs | 15-16 | HTTP, REST, gRPC; Faraday, Rails API mode |
| **4** | Databases | 18-20 | PostgreSQL deep dive, ActiveRecord, NoSQL, sharding |
| **5** | Caching + Scalability | 21-22 | Rails.cache, Redis patterns, read replicas, PgBouncer |
| **6** | Messaging + Microservices | 23-25 | Sidekiq, Karafka, service extraction, Stoplight |
| **7** | Distributed Systems | 26-27 | Consensus, consistent hashing, CRDTs, quorums |
| **8** | Storage + Security + DevOps | 28-31 | S3, Devise/JWT, Docker, Kamal/K8s |
| **9** | Estimation + HLD Easy | 32-33 | Rails performance numbers, URL shortener, KV store |
| **10** | HLD Medium | 34 | Twitter, WhatsApp, Uber, notifications, autocomplete |
| **11** | HLD Hard | 35 | YouTube, payment system, leaderboard, content moderation |
| **12** | Interview Prep + Mocks | 36-38 | Frameworks, mock interviews, review weak areas |

---

### 4-Week Crash Course (Experienced Rails Engineer)

| Week | Focus | Activities |
|------|-------|-----------|
| **1** | Core Concepts | DDIA ch 1-9; review CAP, replication, sharding, caching; memorize Rails numbers |
| **2** | HLD Practice (8 problems) | URL shortener, Twitter, WhatsApp, Uber, YouTube, payment, notification, autocomplete |
| **3** | LLD Practice (6 problems) | Parking lot, elevator, chess, LRU cache, rate limiter, pub-sub (all in Ruby) |
| **4** | Mock Interviews + Review | 4-6 mocks; review weak areas; practice estimation with Rails numbers |

---

### Daily Practice Routine

```
Weekday (1-2 hours):
  - 30 min: Read DDIA or System Design Interview book
  - 30 min: Practice one concept (draw architecture, write Ruby class diagram)
  - 30 min: Review flashcards (latency numbers, Rails numbers, patterns)

Weekend (3-4 hours):
  - 45 min: Full HLD problem (timed, whiteboard, mention Rails tools)
  - 45 min: Full LLD problem (timed, write Ruby code)
  - 30 min: Review and compare with reference solutions
  - 30 min: Read one Shopify/GitHub/37signals engineering blog post
  - 30 min: Update notes, create flashcards
```

---

### How to Know You're Ready

```
You're ready for LLD interviews when you can:
  □ Design a parking lot in 30 minutes with Ruby classes and clean OOP
  □ Name 10+ design patterns and show Ruby-idiomatic implementations
  □ Apply SOLID principles; explain violations in Ruby code
  □ Use modules, duck typing, and composition effectively
  □ Handle concurrency (Mutex, thread safety) when relevant
  □ Draw a clean class diagram with relationships

You're ready for HLD interviews when you can:
  □ Design a URL shortener in 35 minutes (requirements → estimation → architecture)
  □ Estimate QPS, storage, Puma instances, Sidekiq processes for any system
  □ Explain CAP theorem with examples (PostgreSQL = CP, Cassandra = AP)
  □ Design a caching strategy (Rails.cache, Redis, cache-aside, invalidation)
  □ Explain database sharding and when PgBouncer is needed
  □ Design an async pipeline (Sidekiq or Kafka/Karafka)
  □ Discuss trade-offs for every decision
  □ Handle failure scenarios (DB down, Redis down, Sidekiq down)
  □ Mention security (Devise, Rack::Attack, force_ssl) and monitoring (Yabeda, Lograge, OTel)
  □ Complete 10+ HLD problems under time pressure
```

---

## Summary & Key Takeaways

| Tip | Key Point |
|-----|-----------|
| **Draw diagrams** | Practice architecture diagrams with Rails components (Puma, Sidekiq, Redis, PG) |
| **Think out loud** | Narrate reasoning; mention Ruby/Rails trade-offs explicitly |
| **Start simple** | Client → Rails API → DB; then add Redis, Sidekiq, CDN iteratively |
| **Trade-offs** | State them explicitly; "I chose X because... the trade-off is..." |
| **Know your numbers** | Rails-specific: Puma 200-1K req/sec, Sidekiq 100-500 jobs/sec, 200-500 MB/process |
| **Study real systems** | Shopify, GitHub, 37signals — Rails at scale; Netflix, Uber — distributed concepts |
| **Practice timed** | 45 minutes per problem; whiteboard/paper; record yourself |
| **Default tech stack** | PostgreSQL, Redis, Sidekiq, Kafka/Karafka, S3, Searchkick, Kamal/K8s |
| **DDIA + POODR** | DDIA for HLD; POODR for LLD — the two must-read books for Ruby engineers |
| **Mock interviews** | Practice with a partner; get feedback on communication and structure |

---

### Final Checklist — The Night Before Your Interview

```
Knowledge:
  □ Can explain CAP theorem, ACID vs BASE, consistency models
  □ Can estimate QPS, storage, Puma instances for any system
  □ Know 10+ design patterns with Ruby implementations
  □ Know trade-offs of SQL vs NoSQL, cache strategies, sharding
  □ Can design 5+ HLD problems end-to-end (with Rails ecosystem tools)
  □ Can design 5+ LLD problems with Ruby class diagrams and code

Preparation:
  □ Reviewed latency numbers and Rails performance numbers
  □ Have default Ruby tech stack ready (PG, Redis, Sidekiq, Kafka, S3)
  □ Practiced drawing diagrams
  □ Practiced thinking out loud
  □ Got a good night's sleep

During the Interview:
  □ Clarify requirements FIRST (3-5 minutes)
  □ Do estimation BEFORE designing (include Puma/Sidekiq sizing)
  □ Start simple (Rails API + DB), add complexity
  □ Discuss trade-offs for every decision
  □ Mention failure scenarios
  □ Mention security (Devise, Rack::Attack) and monitoring (Yabeda, Lograge)
  □ Ask: "Should I go deeper on any component?"
```

---

> **You've completed the entire System Design: Zero to Hero syllabus (Ruby Edition).**
> **38 modules. From Ruby fundamentals to designing YouTube at scale.**
> **Now go practice, and ace that interview.**