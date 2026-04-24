# Module 38: Common Mistakes & Tips

> This final module consolidates the most important advice for system design interviews — the general tips that apply to every problem, the resources to study, and a structured study plan to go from zero to interview-ready. Think of this as your pre-interview checklist and long-term preparation guide.

---

## 38.1 General Tips

> These tips apply to both LLD and HLD interviews. They're the habits that separate candidates who get offers from those who don't.

---

### Tip 1: Practice Drawing Diagrams

System design is a **visual** exercise. You'll be drawing on a whiteboard, shared screen, or virtual canvas. Practice until drawing is second nature.

**What to Practice:**

| Diagram Type | When to Use | Tools |
|-------------|-------------|-------|
| **Architecture diagram** | HLD — show components and data flow | draw.io, Excalidraw, whiteboard |
| **Class diagram** | LLD — show classes, relationships, patterns | draw.io, PlantUML, whiteboard |
| **Sequence diagram** | Show request flow between components | draw.io, PlantUML |
| **Data flow diagram** | Show how data moves through the system | draw.io, whiteboard |
| **ER diagram** | Show database schema and relationships | draw.io, dbdiagram.io |

**Diagram Best Practices:**

```
✅ Good Diagram:
  - Clear labels on every box and arrow
  - Data flow direction shown with arrows
  - Protocol/format labeled on arrows (HTTP, gRPC, async/Kafka)
  - Components grouped logically (frontend, backend, data layer)
  - Not too many boxes (6-10 for HLD, 8-15 classes for LLD)

❌ Bad Diagram:
  - Unlabeled boxes ("what is this box?")
  - No arrows (how does data flow?)
  - Too many components (overwhelming, can't explain in time)
  - Messy layout (hard to follow)
```

---

### Tip 2: Think Out Loud

**The interviewer can't read your mind.** If you're thinking silently, they don't know if you're stuck or making progress. Narrate your thought process.

```
✅ Good:
  "I'm thinking about the database choice. We have a key-value access pattern —
   look up by short_code. DynamoDB is a natural fit for this because it's
   optimized for key-value lookups and scales automatically. The alternative
   would be PostgreSQL, which gives us more query flexibility but requires
   more operational work for scaling. Given our simple access pattern,
   I'll go with DynamoDB."

❌ Bad:
  *30 seconds of silence*
  "I'll use DynamoDB."
  (Interviewer doesn't know WHY you chose DynamoDB)
```

**What to Narrate:**
- What you're considering and why
- The options you see and their trade-offs
- Why you're choosing one option over another
- What you're deferring for later ("I'll come back to caching after the database design")

---

### Tip 3: Start Simple, Then Add Complexity

**Don't try to design the final architecture in one shot.** Start with the simplest possible design that works, then iteratively add complexity.

```
Iteration 1 (simplest):
  Client → API Server → Database
  "This handles the basic functionality. Now let me add caching..."

Iteration 2 (add caching):
  Client → API Server → Cache (Redis) → Database
  "This handles the read-heavy workload. Now let me think about scaling..."

Iteration 3 (add scaling):
  Client → Load Balancer → API Servers (3x) → Cache → Database (primary + replicas)
  "This handles our current scale. For 10x growth, we'd add sharding..."

Iteration 4 (add async processing):
  + Message Queue (Kafka) for analytics, notifications
  + CDN for static content
  + Monitoring (Prometheus + Grafana)
```

**Why this works:**
- You always have a working design (even if incomplete)
- The interviewer can steer you ("go deeper on caching" or "let's talk about scaling")
- You demonstrate that you can prioritize (most important components first)
- You don't get lost in complexity

---

### Tip 4: Always Discuss Trade-offs

**Every design decision has trade-offs.** Stating them explicitly is the single most impactful thing you can do in an interview.

**The Trade-off Formula:**

```
"I chose [X] because [reason]. The trade-off is [what we give up].
 This is acceptable because [why it's OK for our use case]."

Example:
  "I chose eventual consistency for the timeline cache because it gives us
   lower latency and higher availability. The trade-off is that a user might
   not see a new tweet for a few seconds. This is acceptable because social
   media feeds don't require real-time consistency — a few seconds delay
   is imperceptible to users."
```

**Common Trade-offs to Discuss:**

| Trade-off | When to Discuss |
|-----------|----------------|
| **Consistency vs Availability** | Database choice, caching strategy, replication |
| **Latency vs Throughput** | Sync vs async processing, batching |
| **Cost vs Performance** | Cache size, number of replicas, CDN coverage |
| **Complexity vs Simplicity** | Monolith vs microservices, single DB vs sharded |
| **Read vs Write optimization** | Normalization vs denormalization, CQRS |
| **Accuracy vs Speed** | Exact count vs approximate (HyperLogLog), sampling |
| **Durability vs Speed** | Sync vs async replication, WAL flush frequency |

---

### Tip 5: Know Your Numbers

**Memorize the key numbers** so you can do estimation quickly and confidently.

**Latency Numbers:**

```
L1 cache:           ~1 ns
RAM:                ~100 ns
SSD random read:    ~16 μs
Network (same DC):  ~0.5 ms
HDD seek:           ~4 ms
Network (cross-continent): ~150 ms
```

**Scale Numbers:**

```
Seconds per day:    86,400 ≈ 10^5
Seconds per month:  2.6M ≈ 2.5 × 10^6
Seconds per year:   31.5M ≈ 3 × 10^7

1 KB = 10^3 bytes
1 MB = 10^6 bytes
1 GB = 10^9 bytes
1 TB = 10^12 bytes
1 PB = 10^15 bytes
```

**Capacity Numbers:**

```
Single PostgreSQL:  10K-50K QPS (depends on query complexity)
Single Redis:       100K+ ops/sec
Single Kafka broker: 100K+ messages/sec
Single web server:  1K-10K QPS (depends on workload)
CDN edge:           millions of requests/sec (distributed)
```

---

### Tip 6: Study Real-World Architectures

**The best way to learn system design is to study how real companies solve real problems.**

| Company | What to Study | Key Insight |
|---------|-------------|-------------|
| **Netflix** | Microservices, CDN, chaos engineering | Resilience patterns (circuit breaker, bulkhead) |
| **Uber** | Geospatial matching, real-time systems | Location tracking at scale, surge pricing |
| **Twitter** | News feed, fan-out, timeline caching | Hybrid fan-out (push for regular, pull for celebrities) |
| **Instagram** | Photo storage, CDN, feed ranking | Pre-signed URLs for upload, image processing pipeline |
| **WhatsApp** | Messaging, WebSocket, E2EE | Connection management at 2B users |
| **Stripe** | Payment processing, idempotency | Double-entry ledger, idempotency keys |
| **Google** | Search, MapReduce, Spanner | Inverted index, PageRank, TrueTime |
| **Amazon** | DynamoDB, S3, microservices | Dynamo paper (consistent hashing, quorums) |
| **Meta** | Social graph, TAO, news feed | Graph storage, feed ranking ML |
| **Cloudflare** | CDN, DNS, DDoS protection | Anycast, edge computing |

**Where to Find Architecture Details:**
- Company engineering blogs (Netflix Tech Blog, Uber Engineering, etc.)
- Conference talks (QCon, Strange Loop, InfoQ)
- Research papers (Google's Bigtable, Dynamo, Spanner, MapReduce)
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
  6. Compare with reference solutions
  7. Repeat 2-3 times per week for 4-6 weeks
```

---

### Tip 8: Have a "Go-To" Tech Stack

Don't waste time deciding between technologies in the interview. Have a default stack that you know well.

**Recommended Default Stack:**

| Component | Default Choice | When to Switch |
|-----------|---------------|---------------|
| **SQL Database** | PostgreSQL | MySQL if team prefers; Aurora for AWS-native |
| **NoSQL (key-value)** | Redis (cache) or DynamoDB (persistent) | Cassandra for write-heavy time-series |
| **Message Queue** | Kafka | SQS for simple queues; RabbitMQ for complex routing |
| **Cache** | Redis | Memcached only for simplest caching |
| **Search** | Elasticsearch | PostgreSQL full-text for simple search |
| **Object Storage** | S3 | GCS for GCP |
| **CDN** | CloudFront | Cloudflare for non-AWS |
| **Load Balancer** | ALB (L7) or NLB (L4) | Nginx for on-prem |
| **Container Orchestration** | Kubernetes (EKS) | ECS for simpler setups |
| **Monitoring** | Prometheus + Grafana | Datadog for managed |
| **CI/CD** | GitHub Actions | GitLab CI, Jenkins |

**Why a default stack helps:** You spend zero time on "which database should I use?" and more time on the actual design. You can always justify switching: "My default is PostgreSQL, but for this write-heavy workload, Cassandra is a better fit because..."

---


## 38.2 Resources

> These are the highest-value resources for system design preparation, organized by type. You don't need all of them — pick the ones that match your learning style.

---

### Books (Ranked by Priority)

| Priority | Book | Author | What You'll Learn |
|----------|------|--------|------------------|
| **#1 (Must Read)** | Designing Data-Intensive Applications (DDIA) | Martin Kleppmann | The bible of distributed systems — replication, partitioning, consistency, batch/stream processing |
| **#2 (Must Read)** | System Design Interview Vol 1 & 2 | Alex Xu | Step-by-step solutions to 25+ HLD problems; great for interview prep |
| **#3** | Clean Code | Robert C. Martin | Writing readable, maintainable code; naming, functions, classes |
| **#4** | Design Patterns (GoF) | Gamma, Helm, Johnson, Vlissides | The 23 classic design patterns; essential for LLD |
| **#5** | Head First Design Patterns | Freeman & Robson | Beginner-friendly introduction to design patterns (Java examples, concepts apply to C++) |
| **#6** | Clean Architecture | Robert C. Martin | Architecture principles; dependency rule; use cases; boundaries |
| **#7** | The C++ Programming Language | Bjarne Stroustrup | Comprehensive C++ reference (for C++-specific LLD) |
| **#8** | Effective Modern C++ | Scott Meyers | Modern C++ best practices (move semantics, smart pointers, lambdas) |

**Reading Order for Interview Prep:**
1. DDIA (chapters 1-9 for HLD fundamentals)
2. System Design Interview Vol 1 (practice problems)
3. Head First Design Patterns or GoF (for LLD)
4. System Design Interview Vol 2 (more practice)
5. Clean Code + Clean Architecture (for code quality)

---

### Online Resources

**Free Resources:**

| Resource | Type | What You'll Learn |
|----------|------|------------------|
| **System Design Primer** (GitHub) | Guide | Comprehensive overview of system design concepts with diagrams |
| **ByteByteGo** (Alex Xu) | Newsletter + YouTube | Visual explanations of system design concepts; weekly newsletter |
| **Engineering Blogs** | Articles | How real companies solve real problems at scale |
| **InfoQ** | Conference talks | Architecture talks from industry experts |
| **High Scalability** | Blog | Case studies of large-scale system architectures |
| **Martin Fowler's Blog** | Articles | Architecture patterns, microservices, refactoring |

**Paid Resources:**

| Resource | Type | What You'll Learn |
|----------|------|------------------|
| **Grokking the System Design Interview** (Educative) | Course | Structured HLD problems with step-by-step solutions |
| **Grokking the OOD Interview** (Educative) | Course | Structured LLD problems with class diagrams and code |
| **AlgoExpert SystemsExpert** | Video course | Video explanations of 25+ system design problems |
| **Exponent** | Mock interviews | Practice with real interviewers; video solutions |

---

### Engineering Blogs to Follow

| Company | Blog URL | Notable Posts |
|---------|----------|--------------|
| **Netflix** | netflixtechblog.com | Microservices, chaos engineering, data pipeline |
| **Uber** | eng.uber.com | Geospatial systems, real-time matching, Kafka at scale |
| **Meta** | engineering.fb.com | TAO (social graph), news feed, Cassandra at scale |
| **Google** | cloud.google.com/blog | Spanner, Bigtable, Borg, SRE practices |
| **Amazon/AWS** | aws.amazon.com/blogs | DynamoDB, S3 internals, Lambda, Aurora |
| **Airbnb** | medium.com/airbnb-engineering | Search, payments, service-oriented architecture |
| **Stripe** | stripe.com/blog/engineering | Payment systems, idempotency, API design |
| **Spotify** | engineering.atspotify.com | Microservices, data pipelines, ML infrastructure |
| **LinkedIn** | engineering.linkedin.com | Kafka (created at LinkedIn), feed ranking, graph |
| **Cloudflare** | blog.cloudflare.com | CDN, DNS, DDoS, edge computing, networking |

---

### Key Research Papers (For Deep Understanding)

| Paper | Year | Key Concept |
|-------|------|-------------|
| **Google MapReduce** | 2004 | Distributed batch processing paradigm |
| **Google File System (GFS)** | 2003 | Distributed file system design |
| **Google Bigtable** | 2006 | Wide-column store; LSM-tree storage |
| **Amazon Dynamo** | 2007 | Consistent hashing, quorums, vector clocks, sloppy quorum |
| **Google Spanner** | 2012 | Globally distributed SQL; TrueTime for linearizability |
| **Apache Kafka** | 2011 | Distributed log-based messaging |
| **Raft Consensus** | 2014 | Understandable consensus algorithm |
| **Facebook TAO** | 2013 | Social graph storage and caching |
| **Google Borg** | 2015 | Cluster management (predecessor to Kubernetes) |
| **CRDTs** | 2011 | Conflict-free replicated data types |

**You don't need to read all papers.** DDIA (the book) summarizes the key ideas from most of these papers in an accessible way.

---


## 38.3 Study Plan

> This study plan takes you from zero to interview-ready in 26 weeks (6 months). If you already know C++ and OOP, you can skip to week 7 and finish in ~12 weeks.

---

### 26-Week Comprehensive Plan

| Week | Focus | Modules | Key Activities |
|------|-------|---------|---------------|
| **1-2** | C++ Fundamentals | 1 | Pointers, memory, STL, templates, RAII, smart pointers |
| **3-4** | OOP + SOLID | 2-3 | Classes, inheritance, polymorphism, SOLID principles |
| **5-6** | Design Patterns (Creational + Structural) | 4-5 | Singleton, Factory, Builder, Adapter, Decorator, Proxy |
| **7** | Design Patterns (Behavioral) + UML | 6-7 | Strategy, Observer, Command, State; class diagrams |
| **8** | Advanced OOP + Concurrency | 8-9 | Move semantics, CRTP, threads, mutexes, atomics |
| **9** | DS/Algo for LLD + Error Handling | 10-11 | LRU cache, rate limiter, testing, logging |
| **10-11** | LLD Practice (Easy + Medium) | 12-13 | Parking lot, vending machine, elevator, library system |
| **12** | LLD Practice (Hard) | 14 | Chess, file system, spreadsheet, text editor |
| **13-14** | Networking + APIs + Web Architecture | 15-17 | OSI, TCP/IP, HTTP, REST, GraphQL, gRPC, CDN, proxy |
| **15-16** | Databases (SQL + NoSQL + Deep Dive) | 18-20 | Schema, indexing, ACID, CAP, replication, sharding |
| **17** | Caching + Scalability | 21-22 | Cache strategies, eviction, CQRS, event sourcing |
| **18** | Load Balancing + Message Queues | 23-24 | Algorithms, health checks, Kafka, event-driven |
| **19** | Microservices + Distributed Systems | 25-26 | Service discovery, saga, circuit breaker, consensus |
| **20** | Advanced Distributed + Storage | 27-28 | Consistent hashing, gossip, CRDTs, S3, serialization |
| **21** | Security + Monitoring + DevOps | 29-31 | OAuth, JWT, encryption, Prometheus, Docker, K8s, CI/CD |
| **22** | Estimation + HLD Practice (Easy) | 32-33 | Back-of-envelope, URL shortener, rate limiter, KV store |
| **23-24** | HLD Practice (Medium) | 34 | Twitter, Instagram, WhatsApp, Uber, notifications |
| **25** | HLD Practice (Hard) | 35 | YouTube, Google Docs, Google Maps, payment system |
| **26** | Interview Frameworks + Mock Interviews | 36-38 | LLD framework, HLD framework, mock interviews |

---

### 12-Week Accelerated Plan (Already Know C++ and OOP)

| Week | Focus | Modules | Key Activities |
|------|-------|---------|---------------|
| **1** | SOLID + Design Patterns | 3-6 | All patterns; focus on Strategy, Observer, Factory, Singleton |
| **2** | LLD Practice | 12-14 | 6-8 LLD problems (parking lot through chess) |
| **3** | Networking + APIs | 15-16 | HTTP, REST, GraphQL, gRPC, API Gateway |
| **4** | Databases | 18-20 | SQL, NoSQL, CAP, replication, sharding, indexing |
| **5** | Caching + Scalability | 21-22 | Cache strategies, CQRS, performance optimization |
| **6** | Load Balancing + Messaging | 23-24 | LB algorithms, Kafka, event-driven architecture |
| **7** | Microservices + Distributed Systems | 25-27 | Service mesh, consensus, consistent hashing, CRDTs |
| **8** | Storage + Security + DevOps | 28-31 | S3, encryption, Docker, K8s, CI/CD |
| **9** | Estimation + HLD Easy | 32-33 | Back-of-envelope, URL shortener, KV store, ID generator |
| **10** | HLD Medium | 34 | Twitter, WhatsApp, Uber, notifications, autocomplete |
| **11** | HLD Hard | 35 | YouTube, Google Docs, payment system, Kafka design |
| **12** | Interview Prep + Mock Interviews | 36-38 | Frameworks, mock interviews, review weak areas |

---

### 4-Week Crash Course (Experienced Engineer, Quick Refresh)

| Week | Focus | Activities |
|------|-------|-----------|
| **1** | Core Concepts Review | DDIA chapters 1-9; review CAP, replication, sharding, indexing, caching |
| **2** | HLD Practice (8 problems) | URL shortener, Twitter, WhatsApp, Uber, YouTube, payment system, notification, autocomplete |
| **3** | LLD Practice (6 problems) | Parking lot, elevator, chess, LRU cache, rate limiter, pub-sub |
| **4** | Mock Interviews + Review | 4-6 mock interviews; review weak areas; practice estimation |

---

### Daily Practice Routine

```
Weekday (1-2 hours):
  - 30 min: Read one section of DDIA or System Design Interview book
  - 30 min: Practice one concept (draw an architecture, write a class diagram)
  - 30 min: Review flashcards (latency numbers, design patterns, trade-offs)

Weekend (3-4 hours):
  - 45 min: Full HLD problem (timed, on whiteboard/paper)
  - 45 min: Full LLD problem (timed, write code)
  - 30 min: Review and compare with reference solutions
  - 30 min: Study one engineering blog post or conference talk
  - 30 min: Update notes, create flashcards for new concepts
```

---

### How to Know You're Ready

```
You're ready for LLD interviews when you can:
  □ Design a parking lot system in 30 minutes with class diagram and code
  □ Name 10+ design patterns and explain when to use each
  □ Apply SOLID principles and explain violations
  □ Handle concurrency concerns (mutex, thread safety)
  □ Draw a clean class diagram with relationships and multiplicity

You're ready for HLD interviews when you can:
  □ Design a URL shortener in 35 minutes (requirements → estimation → architecture → deep dive)
  □ Estimate QPS, storage, and cache size for any system
  □ Explain CAP theorem with examples of CP and AP systems
  □ Design a caching strategy (cache-aside, eviction, invalidation)
  □ Explain database sharding (strategies, shard key selection, cross-shard queries)
  □ Design a message queue-based async pipeline
  □ Discuss trade-offs for every design decision
  □ Handle failure scenarios (what if X goes down?)
  □ Mention security, monitoring, and observability
  □ Complete 10+ HLD problems under time pressure
```

---

## Summary & Key Takeaways

| Tip | Key Point |
|-----|-----------|
| **Draw diagrams** | Practice until drawing architecture and class diagrams is second nature |
| **Think out loud** | Narrate your thought process; the interviewer evaluates reasoning, not just the result |
| **Start simple** | Begin with 4-6 components; add complexity iteratively |
| **Trade-offs** | Every decision has trade-offs; state them explicitly using the formula |
| **Know your numbers** | Latency (ns/μs/ms), storage (KB/MB/GB/TB), capacity (QPS per server/Redis/Kafka) |
| **Study real systems** | Netflix, Uber, Twitter, Stripe — learn how they solve problems at scale |
| **Practice timed** | 45 minutes per problem; practice on whiteboard/paper, not just in your head |
| **Default tech stack** | Have a go-to stack (PostgreSQL, Redis, Kafka, S3, K8s) — switch only with justification |
| **DDIA is #1** | "Designing Data-Intensive Applications" is the single most valuable resource |
| **Mock interviews** | Practice with a partner or service; get feedback on communication and structure |

---

### Final Checklist — The Night Before Your Interview

```
Knowledge:
  □ Can explain CAP theorem, ACID vs BASE, consistency models
  □ Can estimate QPS, storage, bandwidth for any system
  □ Know 10+ design patterns and when to use each
  □ Know the trade-offs of SQL vs NoSQL, cache strategies, sharding strategies
  □ Can design 5+ HLD problems end-to-end under time pressure
  □ Can design 5+ LLD problems with class diagrams and code

Preparation:
  □ Reviewed latency numbers and data size reference
  □ Have a default tech stack ready
  □ Practiced drawing diagrams (whiteboard or digital)
  □ Practiced thinking out loud
  □ Got a good night's sleep

During the Interview:
  □ Clarify requirements FIRST (3-5 minutes)
  □ Do estimation BEFORE designing
  □ Start simple, add complexity
  □ Discuss trade-offs for every decision
  □ Mention failure scenarios
  □ Mention security and monitoring
  □ Ask the interviewer: "Should I go deeper on any component?"
```

---

> **You've completed the entire System Design: Zero to Hero syllabus.**
> **38 modules. From C++ fundamentals to designing YouTube at scale.**
> **Now go practice, and ace that interview.**
