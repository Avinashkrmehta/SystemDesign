# Module 32: Back-of-the-Envelope Estimation

> Back-of-the-envelope estimation is a critical skill in system design interviews. Before diving into architecture, you need to estimate the **scale** of the system — how many requests per second, how much storage, how much bandwidth, how many servers. These rough calculations guide architectural decisions: do you need sharding? How big should the cache be? How many servers do you need? This module provides the numbers, formulas, and practice problems to master estimation.

> **Ruby Context:** Ruby/Rails applications have specific performance characteristics that affect estimation. Puma (the standard app server) handles ~1,000-5,000 requests/second per instance depending on workload. Sidekiq processes ~500-5,000 jobs/second per process. A single Rails process typically uses 200-500 MB of RAM. Understanding these Ruby-specific numbers helps you estimate server counts, memory requirements, and connection pool sizes accurately. The GVL (Global VM Lock) in CRuby means each Ruby thread can only execute Ruby code one at a time, making I/O-bound workloads (typical for Rails) scale better than CPU-bound ones.

---

## 32.1 Latency Numbers Every Engineer Should Know

> These numbers give you an intuitive sense of how fast different operations are. Memorize the order of magnitude — you don't need exact values, but you need to know that a network call is ~1,000,000x slower than a memory access.

---

### The Latency Table

| Operation | Latency | Relative | Notes |
|-----------|---------|----------|-------|
| L1 cache reference | ~1 ns | 1x | CPU cache — fastest possible access |
| Branch mispredict | ~3 ns | 3x | CPU pipeline flush |
| L2 cache reference | ~4 ns | 4x | Still on-chip, slightly slower |
| Mutex lock/unlock | ~17 ns | 17x | Thread synchronization overhead |
| Main memory reference (RAM) | ~100 ns | 100x | Off-chip, but still fast |
| Compress 1 KB with Snappy | ~3,000 ns (3 μs) | 3,000x | Fast compression |
| Read 1 MB sequentially from RAM | ~3,000 ns (3 μs) | 3,000x | Memory bandwidth |
| SSD random read | ~16,000 ns (16 μs) | 16,000x | 100x faster than HDD |
| Read 1 MB sequentially from SSD | ~49,000 ns (49 μs) | 49,000x | SSD sequential throughput |
| Round trip within same datacenter | ~500,000 ns (0.5 ms) | 500,000x | Network hop within DC |
| Read 1 MB sequentially from HDD | ~825,000 ns (0.8 ms) | 825,000x | Spinning disk sequential |
| HDD seek | ~4,000,000 ns (4 ms) | 4,000,000x | Spinning disk random access |
| Disk seek (HDD) | ~4,000,000 ns (4 ms) | 4,000,000x | Mechanical movement |
| Send packet CA → Netherlands → CA | ~150,000,000 ns (150 ms) | 150,000,000x | Cross-continent round trip |

---

### Visualizing the Scale

```
If L1 cache access = 1 second (human scale):

  L1 cache reference:        1 second
  L2 cache reference:        4 seconds
  Main memory reference:     1.5 minutes
  SSD random read:           4.5 hours
  HDD seek:                  46 days
  Same-datacenter round trip: 5.8 days
  Cross-continent round trip: 4.8 YEARS
```

---

### Key Takeaways for System Design

| Insight | Implication |
|---------|------------|
| **Memory is 1000x faster than SSD** | Cache hot data in memory (Redis, `Rails.cache`) |
| **SSD is 100x faster than HDD** | Use SSDs for databases; HDDs only for cold storage/archives |
| **Network within DC is 500,000x slower than memory** | Minimize network round trips; batch requests; use local caches |
| **Cross-continent is 300x slower than same-DC** | Use CDNs, edge caching, multi-region deployments |
| **Sequential reads are 100x faster than random reads** | Design for sequential access (append-only logs, LSM trees) |
| **Compression is cheap** | Compress data before sending over network (3 μs to compress 1 KB vs 500 μs network round trip) |

> **Ruby Context:** A typical Rails request involves: Ruby code execution (~1-10 ms), ActiveRecord query (~1-50 ms), Redis cache lookup (~0.5-2 ms), and possibly an external HTTP call (~50-500 ms). The external HTTP call dominates — this is why caching and reducing network calls matter most for Rails performance.

```ruby
# Typical Rails request latency breakdown
# Ruby code execution:     1-10 ms   (in-process, fast)
# ActiveRecord query:      1-50 ms   (network to DB + query time)
# Redis cache hit:         0.5-2 ms  (network to Redis + lookup)
# Redis cache miss + DB:   5-50 ms   (cache miss → DB query → cache write)
# External HTTP call:      50-500 ms (network to external service)
# Total typical request:   10-100 ms (well-optimized Rails app)

# Benchmark in Rails
require 'benchmark'

Benchmark.measure { User.find(1) }                    # ~2-5 ms (cached connection)
Benchmark.measure { Rails.cache.read('key') }          # ~0.5-1 ms (Redis)
Benchmark.measure { Faraday.get('http://api/users/1') } # ~50-200 ms (external)
```

---

### Quick Reference: Time Units

| Unit | Abbreviation | Value |
|------|-------------|-------|
| Nanosecond | ns | 10^-9 seconds |
| Microsecond | μs | 10^-6 seconds = 1,000 ns |
| Millisecond | ms | 10^-3 seconds = 1,000 μs = 1,000,000 ns |
| Second | s | 1,000 ms |

---


## 32.2 Storage Estimation

> Estimating storage requirements is one of the most common calculations in system design interviews. You need to know the size of common data types and be able to estimate how much storage a system needs over time.

---

### Data Type Sizes

| Data Type | Size | Notes |
|-----------|------|-------|
| **Character (ASCII)** | 1 byte | English characters, digits, symbols |
| **Character (UTF-8)** | 1-4 bytes | 1 byte for ASCII, 2-3 bytes for most international characters |
| **Boolean** | 1 byte | (1 bit logically, but stored as 1 byte in most systems) |
| **Short Integer** | 2 bytes | -32,768 to 32,767 |
| **Integer (int32)** | 4 bytes | -2.1 billion to 2.1 billion |
| **Long Integer (int64)** | 8 bytes | -9.2 × 10^18 to 9.2 × 10^18 |
| **Float** | 4 bytes | ~7 decimal digits precision |
| **Double** | 8 bytes | ~15 decimal digits precision |
| **UUID** | 16 bytes | 128-bit unique identifier (36 chars as string with hyphens) |
| **Timestamp** | 8 bytes | Unix timestamp (seconds or milliseconds since epoch) |
| **IPv4 address** | 4 bytes | (15 chars as string: "255.255.255.255") |
| **IPv6 address** | 16 bytes | (39 chars as string) |
| **MD5 hash** | 16 bytes | (32 hex chars as string) |
| **SHA-256 hash** | 32 bytes | (64 hex chars as string) |

> **Ruby Context:** Ruby objects have overhead beyond the raw data size. A Ruby `String` object uses ~40 bytes of overhead plus the string content. A Ruby `Integer` (Fixnum) uses 8 bytes in-process. When estimating memory for Ruby processes (Puma, Sidekiq), account for this object overhead — a Rails process holding 10,000 ActiveRecord objects in memory uses significantly more than `10,000 × row_size`.

---

### Common Object Sizes

| Object | Typical Size | Breakdown |
|--------|-------------|-----------|
| **Tweet** | ~300 bytes | text (280 chars) + metadata (user_id, timestamp, etc.) |
| **URL** | ~100 bytes | Average URL length |
| **Email address** | ~30 bytes | Average email length |
| **User profile** | ~1 KB | name, email, bio, preferences, timestamps |
| **JSON API response** | 1-10 KB | Typical REST API response |
| **Web page (HTML)** | 50-200 KB | HTML + inline CSS/JS |
| **Thumbnail image** | 10-50 KB | Small preview image |
| **Compressed image (JPEG)** | 200 KB - 2 MB | Typical photo |
| **High-res image** | 2-10 MB | Professional photography, raw |
| **Short video (1 min, compressed)** | 5-50 MB | Depends on resolution and codec |
| **HD video (1 min, 1080p)** | 100-200 MB | H.264/H.265 compressed |
| **4K video (1 min)** | 300-500 MB | H.265 compressed |
| **Audio (1 min, MP3)** | 1-2 MB | 128-256 kbps |
| **PDF document** | 100 KB - 10 MB | Depends on content (text vs images) |
| **Database row (typical)** | 100-500 bytes | Depends on schema |
| **Log entry** | 100-500 bytes | Structured JSON log |
| **Kafka message** | 100 bytes - 1 MB | Depends on payload |

---

### Powers of 2 — Quick Reference

| Power | Exact Value | Approximate | Name |
|-------|------------|-------------|------|
| 2^10 | 1,024 | ~1 Thousand | 1 KB (Kilobyte) |
| 2^20 | 1,048,576 | ~1 Million | 1 MB (Megabyte) |
| 2^30 | 1,073,741,824 | ~1 Billion | 1 GB (Gigabyte) |
| 2^40 | 1,099,511,627,776 | ~1 Trillion | 1 TB (Terabyte) |
| 2^50 | ~1.13 × 10^15 | ~1 Quadrillion | 1 PB (Petabyte) |

**Useful Approximations:**

```
1 KB ≈ 1,000 bytes (10^3)
1 MB ≈ 1,000 KB ≈ 1,000,000 bytes (10^6)
1 GB ≈ 1,000 MB ≈ 1,000,000,000 bytes (10^9)
1 TB ≈ 1,000 GB ≈ 1,000,000,000,000 bytes (10^12)
1 PB ≈ 1,000 TB ≈ 1,000,000,000,000,000 bytes (10^15)

For estimation, use powers of 10 (not powers of 2) — close enough and easier to calculate.
```

---

### Storage Estimation Formula

```
Total Storage = (data per item) × (items per day) × (days) × (replication factor)

Example: Twitter-like service
  Tweets per day: 500 million
  Average tweet size: 300 bytes (text + metadata)
  Media: 10% of tweets have an image (average 500 KB)
  Retention: 5 years
  Replication: 3x

  Text storage per day:
    500M × 300 bytes = 150 GB/day

  Media storage per day:
    500M × 10% × 500 KB = 25 TB/day

  5-year storage:
    Text: 150 GB/day × 365 × 5 = 274 TB
    Media: 25 TB/day × 365 × 5 = 45.6 PB
    Total (with 3x replication): (274 TB + 45.6 PB) × 3 ≈ 137 PB

  → Media dominates storage! Text is negligible compared to media.
```

---


## 32.3 Traffic Estimation

> Traffic estimation determines how many requests your system needs to handle. This drives decisions about server count, database capacity, caching strategy, and whether you need sharding.

---

### Key Metrics

**DAU (Daily Active Users):**

```
MAU (Monthly Active Users) → DAU:
  Typical ratio: DAU ≈ MAU / 3 to MAU / 5
  
  Example: 100M MAU → DAU ≈ 20-33M
```

**QPS (Queries Per Second):**

```
QPS = DAU × average_queries_per_user / seconds_per_day

seconds_per_day = 86,400 ≈ 100,000 (for easy calculation, use 10^5)

Example: Twitter-like service
  DAU: 300 million
  Average actions per user per day: 10 (view timeline, post, like, etc.)
  
  Total daily requests: 300M × 10 = 3 billion
  QPS = 3,000,000,000 / 86,400 ≈ 35,000 QPS
  
  Simplified: 3B / 100,000 = 30,000 QPS (close enough for estimation)
```

**Peak QPS:**

```
Peak QPS = Average QPS × peak_multiplier

Peak multiplier: typically 2x to 5x average
  - Social media: 2-3x (gradual daily pattern)
  - E-commerce (Black Friday): 5-10x
  - Live events (Super Bowl): 10-100x

Example:
  Average QPS: 35,000
  Peak QPS (3x): 105,000
  
  Design for peak, not average!
```

**Read:Write Ratio:**

```
Most systems are read-heavy:
  Social media (Twitter): 100:1 (100 reads per 1 write)
  E-commerce (Amazon): 50:1 (many views per purchase)
  Chat (WhatsApp): 1:1 (roughly equal reads and writes)
  IoT sensors: 1:10 (write-heavy — sensors push data)
  
Example: Twitter at 35,000 QPS with 100:1 read:write
  Write QPS: 35,000 / 101 ≈ 350 writes/sec
  Read QPS: 35,000 - 350 ≈ 34,650 reads/sec
```

---

### Traffic Estimation Template

```
Given:
  MAU = X million
  DAU = MAU / 4 (or given)
  Actions per user per day = Y
  Read:Write ratio = R:W

Calculate:
  Total daily requests = DAU × Y
  Average QPS = Total daily requests / 86,400
  Peak QPS = Average QPS × 3 (or given multiplier)
  Write QPS = Average QPS / (R + W) × W
  Read QPS = Average QPS / (R + W) × R
```

---

### Useful Time Constants

| Period | Seconds | Approximation |
|--------|---------|---------------|
| 1 minute | 60 | ~60 |
| 1 hour | 3,600 | ~4 × 10^3 |
| 1 day | 86,400 | ~10^5 (use this!) |
| 1 week | 604,800 | ~6 × 10^5 |
| 1 month | 2,592,000 | ~2.5 × 10^6 |
| 1 year | 31,536,000 | ~3 × 10^7 |

**Pro Tip:** Use `86,400 ≈ 100,000 = 10^5` for daily calculations. It's only 16% off and makes mental math much easier.

---


## 32.4 Bandwidth Estimation

> Bandwidth estimation determines how much data flows through the system per second. This affects network infrastructure, CDN requirements, and cost.

---

### Bandwidth Formula

```
Bandwidth = QPS × average_request_or_response_size

Incoming bandwidth (upload):
  = Write QPS × average request size

Outgoing bandwidth (download):
  = Read QPS × average response size
```

**Example: Image Sharing Service (Instagram-like)**

```
Given:
  DAU: 100 million
  Photos uploaded per day: 10 million (10% of users upload 1 photo)
  Average photo size: 1 MB
  Average user views 50 photos per day
  Thumbnail size: 50 KB
  Full-size image: 1 MB

Upload bandwidth:
  10M photos/day × 1 MB = 10 TB/day
  10 TB / 86,400 seconds ≈ 116 MB/s ≈ 930 Mbps ≈ 1 Gbps

Download bandwidth (thumbnails in feed):
  100M users × 50 thumbnails × 50 KB = 250 TB/day
  250 TB / 86,400 ≈ 2.9 GB/s ≈ 23 Gbps

Download bandwidth (full-size views):
  Assume 10% of thumbnails are clicked for full view:
  100M × 50 × 10% × 1 MB = 500 TB/day
  500 TB / 86,400 ≈ 5.8 GB/s ≈ 46 Gbps

Total outgoing: ~70 Gbps (peak: ~200 Gbps)

→ This is why CDNs are essential! Serving 200 Gbps from your own servers is extremely expensive.
   CDN offloads 80-95% of this bandwidth.
```

---

### CDN Offloading

```
Without CDN:
  All 200 Gbps served from your origin servers
  Cost: $0.05-0.09/GB × 500 TB/day = $25,000-45,000/day!

With CDN (90% cache hit rate):
  CDN serves: 200 Gbps × 90% = 180 Gbps (from edge, cheap)
  Origin serves: 200 Gbps × 10% = 20 Gbps (cache misses only)
```

---

### Bandwidth Quick Reference

| Metric | Value |
|--------|-------|
| 1 Mbps | 1 million bits per second = 125 KB/s |
| 1 Gbps | 1 billion bits per second = 125 MB/s |
| 10 Gbps | Typical server NIC |
| 25-100 Gbps | High-performance server NIC |
| 1 Tbps | Large CDN edge capacity |

**Conversion:**
```
Bytes to bits: multiply by 8
  1 MB/s = 8 Mbps
  1 GB/s = 8 Gbps
  
Bits to bytes: divide by 8
  100 Mbps = 12.5 MB/s
  1 Gbps = 125 MB/s
```

---


## 32.5 Ruby/Rails-Specific Performance Numbers

> These numbers are specific to the Ruby/Rails ecosystem and are essential for estimating server counts, memory requirements, and connection pool sizes for Rails applications.

---

### Puma (Web Server) Performance

| Metric | Typical Value | Notes |
|--------|--------------|-------|
| **Requests/sec per Puma worker** | 200-1,000 | Depends on request complexity |
| **Requests/sec per Puma instance** | 1,000-5,000 | With 2-4 workers, 5 threads each |
| **Memory per Puma worker** | 200-500 MB | Rails app with gems loaded |
| **Memory per Puma instance** | 500 MB - 2 GB | 2-4 workers |
| **Threads per worker** | 5 (default) | Limited by GVL for CPU work; I/O releases GVL |
| **Workers per instance** | 2-4 | Typically = number of CPU cores |
| **Request latency (simple)** | 5-20 ms | Cache hit, simple DB query |
| **Request latency (moderate)** | 20-100 ms | Multiple DB queries, some computation |
| **Request latency (complex)** | 100-500 ms | External API calls, heavy computation |

```
Puma capacity estimation:

  Instance: 2 workers × 5 threads = 10 concurrent requests
  Average request time: 50 ms
  Throughput: 10 / 0.05 = 200 requests/sec per instance
  
  With faster requests (20 ms average):
  Throughput: 10 / 0.02 = 500 requests/sec per instance
  
  With 4 workers × 5 threads = 20 concurrent:
  Throughput: 20 / 0.05 = 400 requests/sec per instance
```

---

### Sidekiq (Background Jobs) Performance

| Metric | Typical Value | Notes |
|--------|--------------|-------|
| **Jobs/sec per Sidekiq process** | 500-5,000 | Depends on job complexity |
| **Memory per Sidekiq process** | 200-500 MB | Similar to Puma worker |
| **Default concurrency** | 10 threads | Configurable via `-c` flag |
| **Simple job (cache update)** | 1-5 ms | Redis + simple logic |
| **Moderate job (DB + email)** | 50-200 ms | DB queries + external API |
| **Heavy job (report generation)** | 1-60 seconds | Complex computation or large data |

```
Sidekiq capacity estimation:

  Process: 10 threads, average job time: 100 ms
  Throughput: 10 / 0.1 = 100 jobs/sec per process
  
  With 25 threads, average job time: 50 ms:
  Throughput: 25 / 0.05 = 500 jobs/sec per process
  
  Need to process 10,000 jobs/sec?
  10,000 / 500 = 20 Sidekiq processes
```

---

### ActiveRecord / Database Performance

| Metric | Typical Value | Notes |
|--------|--------------|-------|
| **Simple query (indexed)** | 1-5 ms | `User.find(id)`, `WHERE id = ?` |
| **Complex query (join)** | 5-50 ms | Multi-table join with conditions |
| **N+1 query (100 records)** | 100-500 ms | 100 individual queries instead of 1 |
| **Connection pool size** | 5 per worker | Matches Puma thread count |
| **Max connections (PostgreSQL)** | 100 (default) | Each uses ~10 MB RAM |
| **PgBouncer connections** | 20-50 to DB | Multiplexes hundreds of app connections |

```
Database connection estimation:

  Puma: 3 instances × 2 workers × 5 threads = 30 connections needed
  Sidekiq: 2 processes × 10 threads = 20 connections needed
  Total: 50 connections
  
  PostgreSQL default max_connections: 100 → OK for now
  
  At scale (20 Puma instances + 10 Sidekiq processes):
  20 × 2 × 5 + 10 × 10 = 300 connections needed
  PostgreSQL max_connections: 100 → NOT ENOUGH!
  
  Solution: PgBouncer
  300 app connections → PgBouncer → 50 DB connections
  (PgBouncer multiplexes and reuses connections)
```

---

### Redis Performance

| Metric | Typical Value | Notes |
|--------|--------------|-------|
| **GET/SET latency** | 0.5-1 ms | Network round trip + operation |
| **Operations/sec (single instance)** | 100,000-200,000 | Single-threaded but very fast |
| **Memory per key (small value)** | ~100 bytes overhead | Key + value + Redis metadata |
| **Max memory (practical)** | 25-50 GB | Beyond this, consider clustering |
| **Sidekiq Redis memory** | 1-5 GB | Job queue + retry set + scheduled set |
| **Rails.cache Redis memory** | 1-50 GB | Depends on cached data volume |

```
Redis memory estimation for Rails.cache:

  Cached objects: 1 million
  Average cached value: 2 KB (serialized JSON)
  Redis overhead per key: ~100 bytes
  
  Total: 1M × (2 KB + 100 bytes) ≈ 2.1 GB
  
  → Single Redis instance handles this easily
```

---

### Rails Memory Estimation

```
Rails process memory breakdown:

  Ruby runtime:           ~30 MB
  Rails framework:        ~50 MB
  Gems (typical Gemfile):  ~50-150 MB
  Application code:       ~10-30 MB
  Per-request overhead:   ~1-5 MB (GC reclaims between requests)
  
  Total per Puma worker:  ~200-400 MB
  Total per Puma instance (2 workers): ~400-800 MB
  Total per Puma instance (4 workers): ~800 MB - 1.6 GB

Server sizing:
  t3.medium (4 GB RAM):  2 Puma workers comfortably
  t3.large (8 GB RAM):   4 Puma workers
  t3.xlarge (16 GB RAM): 4 Puma workers + 2 Sidekiq processes
  m5.xlarge (16 GB RAM): Same, better CPU for compute-heavy apps
```

---

### Ruby/Rails Server Count Estimation

```
Formula:
  Puma instances = Peak QPS / QPS_per_instance
  Sidekiq processes = Peak jobs/sec / jobs_per_process

Example: E-commerce Rails application
  DAU: 1 million
  Actions per user per day: 20 (browse, search, add to cart, checkout)
  Average QPS: 1M × 20 / 86,400 ≈ 230 QPS
  Peak QPS (3x): 690 QPS
  
  Puma throughput: 300 req/sec per instance (moderate complexity)
  Puma instances needed: 690 / 300 ≈ 3 instances
  With 2x headroom: 6 Puma instances
  
  Background jobs: 10% of requests generate a job = 69 jobs/sec peak
  Sidekiq throughput: 200 jobs/sec per process
  Sidekiq processes: 69 / 200 ≈ 1 process
  With headroom: 2 Sidekiq processes
  
  Memory:
  6 Puma instances × 800 MB = 4.8 GB
  2 Sidekiq processes × 400 MB = 800 MB
  Total: ~5.6 GB → 2 × t3.large (8 GB each) with room to spare

Example: High-traffic social media Rails API
  DAU: 50 million
  Actions per user per day: 30
  Average QPS: 50M × 30 / 86,400 ≈ 17,000 QPS
  Peak QPS (3x): 51,000 QPS
  
  Puma throughput: 500 req/sec per instance (mostly cached reads)
  Puma instances needed: 51,000 / 500 = 102 instances
  With headroom: ~130 Puma instances across 3 AZs (~44 per AZ)
  
  Memory: 130 × 800 MB = 104 GB
  → ~17 × m5.xlarge (16 GB each) or use Kubernetes with resource limits
```

---


## 32.6 Common Calculations

> These are the calculations you'll use most frequently in system design interviews. Practice them until they're second nature.

---

### How Many Servers Needed?

```
Formula:
  Servers = Peak QPS / QPS_per_server

Typical server capacity (depends on workload):
  Simple API (read from cache): 10,000-50,000 QPS per server
  API with DB queries: 1,000-5,000 QPS per server
  CPU-intensive (image processing): 100-500 QPS per server
  ML inference (GPU): 10-100 QPS per server

Ruby/Rails-specific:
  Rails API (cached reads): 500-2,000 QPS per Puma instance
  Rails API (DB queries): 200-500 QPS per Puma instance
  Rails API (complex, external calls): 50-200 QPS per Puma instance
  Sidekiq (simple jobs): 500-2,000 jobs/sec per process
  Sidekiq (moderate jobs): 100-500 jobs/sec per process

Example: Social media API (Rails)
  Peak QPS: 100,000
  QPS per Puma instance (cached reads): 1,000
  Puma instances needed: 100,000 / 1,000 = 100 instances
  
  With 2x headroom: 200 instances
  With 3 availability zones: ~67 instances per AZ
  Memory: 200 × 800 MB = 160 GB → ~25 × m5.xlarge (16 GB each)
```

---

### How Much Storage for N Years?

```
Formula:
  Storage = daily_data × 365 × years × replication_factor

Example: Chat application (WhatsApp-like)
  Messages per day: 100 billion
  Average message size: 100 bytes (text)
  Media messages: 5% with average 200 KB
  Retention: 5 years
  Replication: 3x

  Text per day: 100B × 100 bytes = 10 TB/day
  Media per day: 100B × 5% × 200 KB = 1 PB/day
  
  5-year text: 10 TB × 365 × 5 = 18.25 PB
  5-year media: 1 PB × 365 × 5 = 1,825 PB ≈ 1.8 EB (exabytes!)
  
  With 3x replication:
    Text: 18.25 PB × 3 = 54.75 PB
    Media: 1.8 EB × 3 = 5.4 EB
  
  → Media storage dominates by 100x
  → Use object storage (S3) for media, database for metadata only
```

---

### How Many Shards Needed?

```
Formula:
  Shards = Total data size / Max data per shard
  
  OR
  
  Shards = Peak write QPS / Max write QPS per shard

Typical shard capacity:
  PostgreSQL: 500 GB - 2 TB per shard (comfortable)
  MySQL: 500 GB - 1 TB per shard
  Cassandra: 1-5 TB per node
  DynamoDB: auto-sharded (managed)

Example: User database (Rails + PostgreSQL)
  Total users: 500 million
  Average user record: 1 KB
  Total data: 500M × 1 KB = 500 GB
  
  With indexes and overhead: 500 GB × 2 = 1 TB
  Max per shard: 500 GB
  Shards needed: 1 TB / 500 GB = 2 shards
  
  With growth (2x in 3 years): 4 shards
  With replication (3x): 4 × 3 = 12 database instances
```

---

### Cache Size Estimation

```
Formula:
  Cache size = daily_active_data × cache_ratio

  Where cache_ratio = percentage of data that should be cached
  Typically: cache the top 20% of data (Pareto principle — 80/20 rule)

Example: URL shortener (Rails)
  Total URLs: 1 billion
  Daily active URLs (accessed at least once): 100 million (10%)
  Average URL record: 200 bytes (short URL + long URL + metadata)
  
  Cache the top 20% of daily active URLs:
    100M × 20% × 200 bytes = 4 GB
  
  With overhead (Redis metadata): 4 GB × 1.5 = 6 GB
  
  → A single Redis instance (64 GB) can easily handle this!

Example: Rails.cache for e-commerce product pages
  Total products: 5 million
  Daily active products: 500,000 (10%)
  Average cached response: 5 KB (serialized JSON)
  Cache top 20%: 500K × 20% × 5 KB = 500 MB
  
  → Easily fits in a single ElastiCache Redis instance
  → Cache hit ratio: ~80% (Pareto)
  → Database only handles 20% of reads
```

---

### Number of Connections (Rails-Specific)

```
Formula:
  Total DB connections = puma_instances × workers × threads + sidekiq_processes × concurrency

Example: Medium Rails application
  Puma: 10 instances × 2 workers × 5 threads = 100 connections
  Sidekiq: 5 processes × 10 threads = 50 connections
  Rails console / migrations / cron: ~5 connections
  Total: 155 connections
  
  PostgreSQL max_connections default: 100 → NOT ENOUGH!
  
  Solutions:
  1. PgBouncer: 155 app connections → 30 DB connections (transaction pooling)
  2. Increase max_connections to 200 (each uses ~10 MB RAM = 2 GB total)
  3. Reduce Sidekiq concurrency to 5

Large Rails application:
  Puma: 50 instances × 4 workers × 5 threads = 1,000 connections
  Sidekiq: 20 processes × 15 threads = 300 connections
  Total: 1,300 connections
  
  → PgBouncer is mandatory at this scale
  → PgBouncer: 1,300 app connections → 50-100 DB connections
  → Consider read replicas to split read traffic
```

---

### Quick Estimation Cheat Sheet

| Question | Formula | Example |
|----------|---------|---------|
| QPS | DAU × actions / 86,400 | 100M × 10 / 100K = 10K QPS |
| Peak QPS | QPS × 3 | 10K × 3 = 30K QPS |
| Storage/day | items/day × size | 10M × 1 KB = 10 GB/day |
| Storage/year | storage/day × 365 | 10 GB × 365 = 3.65 TB/year |
| Bandwidth | QPS × response size | 10K × 10 KB = 100 MB/s |
| Puma instances | Peak QPS / QPS per instance | 30K / 500 = 60 instances |
| Sidekiq processes | Peak jobs/sec / jobs per process | 1K / 200 = 5 processes |
| DB connections | (puma × workers × threads) + (sidekiq × concurrency) | (10×2×5) + (5×10) = 150 |
| Shards | Total data / max per shard | 10 TB / 500 GB = 20 shards |
| Cache size | Active data × 20% | 100 GB × 20% = 20 GB |

---


## 32.7 Practice Problems

> Practice these estimation problems until you can do them quickly and confidently. In interviews, show your work — the process matters more than the exact numbers.

---

### Practice 1: Estimate Storage for Twitter (1 Year)

```
Given:
  DAU: 300 million
  Tweets per day: 500 million
  Average tweet: 280 characters = 280 bytes (text only)
  10% of tweets have an image (average 500 KB)
  1% of tweets have a video (average 5 MB)
  Metadata per tweet: 200 bytes (user_id, timestamp, likes, retweets, etc.)
  Replication factor: 3

Text + Metadata:
  500M tweets × (280 + 200) bytes = 500M × 480 bytes = 240 GB/day
  Per year: 240 GB × 365 = 87.6 TB/year

Images:
  500M × 10% × 500 KB = 25 TB/day
  Per year: 25 TB × 365 = 9,125 TB ≈ 9.1 PB/year

Videos:
  500M × 1% × 5 MB = 25 TB/day
  Per year: 25 TB × 365 = 9,125 TB ≈ 9.1 PB/year

Total (before replication):
  Text: 87.6 TB
  Images: 9.1 PB
  Videos: 9.1 PB
  Total: ~18.3 PB/year

With 3x replication: ~55 PB/year

Key Insight: Media (images + videos) is 99.5% of storage. Text is negligible.
Storage Strategy: Text/metadata in PostgreSQL; media in S3 (ActiveStorage).
```

---

### Practice 2: Estimate QPS for Google Search

```
Given:
  Global internet users: ~5 billion
  Google's market share: ~90%
  Google users: ~4.5 billion
  DAU: ~1.5 billion (not everyone searches every day)
  Average searches per user per day: ~5

Total daily searches:
  1.5B × 5 = 7.5 billion searches/day

Average QPS:
  7.5B / 86,400 ≈ 87,000 QPS

Peak QPS (3x):
  ~260,000 QPS

Per search, Google may make:
  - 1 query to the search index
  - 1 query to the ads system
  - 1 query to the knowledge graph
  - 1 query to autocomplete
  Total internal QPS: 260,000 × 4 = ~1 million internal QPS

Key Insight: The public-facing QPS is just the tip of the iceberg.
Internal fan-out multiplies the actual load significantly.
```

---

### Practice 3: Estimate Bandwidth for YouTube

```
Given:
  DAU: 2 billion
  Average watch time: 40 minutes/day
  Average video bitrate: 5 Mbps (1080p)
  Upload: 500 hours of video uploaded per minute

Download bandwidth:
  Peak concurrent viewers: 10% of DAU = 200 million
  200M × 5 Mbps = 1,000 Tbps = 1 Pbps (petabit per second!)
  
  With CDN serving 95%: Origin serves 50 Tbps
  CDN serves: 950 Tbps across thousands of edge servers worldwide

Upload bandwidth:
  500 hours/min × 60 min/hour = 30,000 hours/hour
  Average upload bitrate: 10 Mbps
  30,000 hours × 3,600 sec × 10 Mbps ≈ 1 Tbps upload

Key Insight: Download bandwidth is 1000x upload bandwidth.
CDN is absolutely essential — no single data center can serve 1 Pbps.
```

---

### Practice 4: Estimate Rails Servers for an E-Commerce Platform

```
Given:
  DAU: 5 million
  Actions per user per day: 30 (browse, search, add to cart, checkout)
  Read:Write ratio: 50:1
  Average request latency: 40 ms
  Puma config: 2 workers × 5 threads = 10 concurrent requests per instance

Average QPS:
  5M × 30 / 86,400 ≈ 1,740 QPS

Peak QPS (5x for e-commerce — flash sales):
  1,740 × 5 = 8,700 QPS

Puma throughput per instance:
  10 concurrent / 0.04s = 250 requests/sec

Puma instances needed:
  8,700 / 250 = 35 instances
  With 2x headroom: 70 Puma instances
  Across 3 AZs: ~24 per AZ

Memory:
  70 instances × 800 MB = 56 GB
  → 9 × m5.xlarge (16 GB, 4 vCPU) or 18 × t3.large (8 GB, 2 vCPU)

Sidekiq (background jobs):
  10% of requests generate a job: 870 jobs/sec peak
  Average job time: 200 ms
  Sidekiq throughput: 10 threads / 0.2s = 50 jobs/sec per process
  Processes needed: 870 / 50 = 18 Sidekiq processes
  Memory: 18 × 400 MB = 7.2 GB → 2 × t3.xlarge (16 GB each)

Database connections:
  Puma: 70 × 2 × 5 = 700 connections
  Sidekiq: 18 × 10 = 180 connections
  Total: 880 connections → PgBouncer mandatory
  PgBouncer: 880 app connections → 50 DB connections

Redis:
  Cache: ~2 GB (product pages, user sessions)
  Sidekiq: ~1 GB (job queues)
  Total: ~3 GB → single ElastiCache Redis instance (cache.r6g.large)

PostgreSQL:
  Products: 1M × 2 KB = 2 GB
  Users: 5M × 1 KB = 5 GB
  Orders: 50K/day × 500 bytes × 365 × 3 years = 27 GB
  Total with indexes: ~100 GB → single RDS instance (db.r6g.xlarge)
  Read replicas: 2 (for read-heavy traffic)
```

---

### Practice 5: Estimate Servers for a Chat Application

```
Given:
  DAU: 500 million
  Peak concurrent users: 10% of DAU = 50 million
  Messages per user per day: 50
  Peak messages per second: 500M × 50 / 86,400 × 3 (peak) ≈ 870,000 msg/sec

WebSocket connections:
  50 million concurrent connections
  Each connection: ~10 KB memory (buffers, state)
  
  Connections per server: ~100,000 (practical limit with 16 GB RAM for connections)
  Servers needed: 50M / 100K = 500 WebSocket servers

Message processing:
  870,000 messages/sec
  Each server handles: ~5,000 messages/sec
  Servers needed: 870,000 / 5,000 = 174 message processing servers

Storage:
  500M users × 50 messages × 100 bytes = 2.5 TB/day
  Per year: 2.5 TB × 365 = 912 TB ≈ 1 PB/year

Key Insight: WebSocket connections are the primary scaling challenge.
Need 500+ servers just for maintaining persistent connections.

Ruby Note: For WebSocket-heavy applications, consider using Action Cable
with AnyCable (Go-based WebSocket server) instead of pure Ruby — AnyCable
handles 10-50x more connections per server than Action Cable's Ruby server.
```

---

### Estimation Tips for Interviews

| Tip | Why |
|-----|-----|
| **Round aggressively** | Use 10^5 instead of 86,400; use 300M instead of 314M. Precision doesn't matter. |
| **Show your work** | Write down assumptions and calculations. The process matters more than the answer. |
| **State assumptions** | "I'll assume 300M DAU and 10 actions per user per day" — makes your reasoning clear |
| **Use powers of 10** | Much easier mental math than exact numbers |
| **Sanity check** | Does the answer make sense? If you calculate 1 million servers, something is wrong. |
| **Start with what you know** | DAU → QPS → storage → bandwidth → servers (in that order) |
| **Identify the bottleneck** | Is it storage? QPS? Bandwidth? Connections? Focus on the limiting factor. |
| **Consider media separately** | Text is tiny; images/videos dominate storage and bandwidth |
| **Account for replication** | Storage × 3 for typical replication factor |
| **Design for peak** | Size infrastructure for peak load (3-5x average), not average |
| **Know Ruby numbers** | Puma: 200-1,000 req/sec per instance; Sidekiq: 100-500 jobs/sec per process; 200-500 MB per process |

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Latency Numbers | Memory: 100 ns; SSD: 16 μs; Network (DC): 0.5 ms; Network (cross-continent): 150 ms |
| Key Insight | Memory is 1000x faster than SSD; SSD is 100x faster than HDD; network is 500,000x slower than memory |
| Data Sizes | Tweet: 300 bytes; Image: 500 KB; Video (1 min): 5-50 MB; User profile: 1 KB |
| Powers of 2 | KB=10^3, MB=10^6, GB=10^9, TB=10^12, PB=10^15 |
| QPS Formula | DAU × actions_per_user / 86,400 (use 10^5 for easy math) |
| Peak QPS | Average QPS × 3 (typical) to × 10 (extreme events) |
| Read:Write | Social media: 100:1; E-commerce: 50:1; Chat: 1:1; IoT: 1:10 |
| Storage | daily_data × 365 × years × replication_factor; media dominates text by 100x |
| Bandwidth | QPS × response_size; CDN offloads 80-95% of download bandwidth |
| Servers | Peak QPS / QPS_per_server; typical: 1K-10K QPS per server |
| Shards | Total_data / max_per_shard; or Peak_write_QPS / max_writes_per_shard |
| Cache Size | Active_data × 20% (Pareto); single Redis handles up to ~50 GB |
| Seconds/Day | 86,400 ≈ 10^5 (use this approximation!) |
| Estimation Process | DAU → QPS → Storage → Bandwidth → Servers → Shards → Cache |

**Ruby-Specific Numbers:**

| Metric | Value |
|--------|-------|
| **Puma req/sec per instance** | 200-1,000 (depends on request complexity) |
| **Puma memory per worker** | 200-500 MB |
| **Puma workers per instance** | 2-4 (= CPU cores) |
| **Puma threads per worker** | 5 (default) |
| **Sidekiq jobs/sec per process** | 100-500 (moderate jobs) |
| **Sidekiq memory per process** | 200-500 MB |
| **Sidekiq default concurrency** | 10 threads |
| **Rails request latency (simple)** | 5-20 ms |
| **Rails request latency (moderate)** | 20-100 ms |
| **ActiveRecord query (indexed)** | 1-5 ms |
| **Redis GET/SET from Rails** | 0.5-2 ms |
| **External HTTP call** | 50-500 ms |
| **DB connection pool per worker** | 5 (= thread count) |
| **PgBouncer needed at** | > 100 total connections |

---

## Interview Tips for Module 32

1. **Memorize latency numbers** — know the order of magnitude for memory, SSD, HDD, network (same DC), network (cross-continent)
2. **Know data sizes** — tweet (~300 bytes), image (~500 KB), video (~5 MB/min), user profile (~1 KB)
3. **Powers of 2** — KB, MB, GB, TB, PB; use powers of 10 for estimation
4. **QPS calculation** — DAU × actions / 86,400; use 10^5 for easy math; multiply by 3 for peak
5. **Read:Write ratio** — know typical ratios for different systems; this determines if you need read replicas or write sharding
6. **Storage estimation** — always separate text from media; media dominates by 100x; account for replication
7. **Bandwidth** — QPS × response size; mention CDN for offloading download bandwidth
8. **Server count (Rails)** — Peak QPS / Puma throughput; know that Puma does 200-1,000 req/sec per instance; account for headroom and AZs
9. **Sidekiq sizing** — Peak jobs/sec / Sidekiq throughput; know that Sidekiq does 100-500 jobs/sec per process
10. **DB connections** — (Puma instances × workers × threads) + (Sidekiq processes × concurrency); mention PgBouncer when > 100 connections
11. **Cache sizing** — 20% of active data (Pareto); check if it fits in a single Redis instance
12. **Show your work** — write down assumptions, formulas, and calculations step by step; sanity check the result