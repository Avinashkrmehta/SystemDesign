# Module 21: Caching

> Caching is the single most impactful technique for improving system performance. A well-designed caching strategy can reduce database load by 90%+, cut response times from hundreds of milliseconds to single-digit milliseconds, and save significant infrastructure costs. But caching also introduces complexity — stale data, cache invalidation, thundering herds, and consistency challenges. This module covers caching from fundamentals to production-grade distributed caching patterns.

---

## 21.1 Caching Fundamentals

> A **cache** is a high-speed data storage layer that stores a subset of data — typically transient — so that future requests for that data are served faster than accessing the primary data source. Caching works because of the **Pareto principle**: roughly 20% of data accounts for 80% of access.

---

### Why Cache? (Latency, Throughput, Cost)

**Latency Reduction:**

```
Without cache:
  Client → App Server → Database (10-100ms) → App Server → Client
  Total: 50-200ms

With cache:
  Client → App Server → Cache (0.1-1ms) → App Server → Client
  Total: 5-20ms (10-50x faster)
```

| Storage | Typical Latency | Relative Speed |
|---------|----------------|---------------|
| L1 CPU Cache | ~1 ns | 1x (baseline) |
| L2 CPU Cache | ~4 ns | 4x |
| RAM (in-process cache) | ~100 ns | 100x |
| Redis (network + RAM) | 0.1-1 ms | 100,000-1,000,000x |
| SSD (database) | 0.1-1 ms | 100,000-1,000,000x |
| HDD (database) | 5-20 ms | 5,000,000-20,000,000x |
| Network (cross-region DB) | 50-200 ms | 50,000,000-200,000,000x |

**Throughput Increase:**
- A single Redis instance handles **100,000+ operations/second**
- A single PostgreSQL instance handles **10,000-50,000 queries/second** (depending on complexity)
- Caching offloads 80-95% of reads from the database → database handles only cache misses

**Cost Reduction:**
- Database instances are expensive (compute-optimized, storage-optimized)
- Cache instances are cheaper (memory-optimized, smaller)
- Reducing database load means fewer/smaller database instances needed
- Example: 10 database read replicas → 2 replicas + 1 Redis cluster (significant cost savings)

---

### Cache Hit, Cache Miss

```
Cache Hit (fast path):
  Client → App → Cache: "Do you have user:123?"
                 Cache: "Yes! Here's the data" → return to client
                 (No database query needed)

Cache Miss (slow path):
  Client → App → Cache: "Do you have user:123?"
                 Cache: "No, I don't have it"
           App → Database: "SELECT * FROM users WHERE id = 123"
           App → Cache: "Store this for next time" (SET user:123 = data, TTL=3600)
           App → Client: return data
```

**Cache Hit Flow:**
```
Request → Check Cache → HIT → Return cached data
                                (fast: 0.1-1ms)
```

**Cache Miss Flow:**
```
Request → Check Cache → MISS → Query Database → Store in Cache → Return data
                                (slow: 10-100ms)    (for next time)
```

---

### Hit Ratio

The **hit ratio** (or hit rate) is the percentage of requests served from the cache. It's the most important metric for cache effectiveness.

```
Hit Ratio = Cache Hits / (Cache Hits + Cache Misses) × 100%

Example:
  1,000,000 total requests
  950,000 cache hits
  50,000 cache misses
  Hit Ratio = 950,000 / 1,000,000 = 95%
```

**Hit Ratio Benchmarks:**

| Hit Ratio | Assessment | Impact |
|-----------|-----------|--------|
| < 50% | Poor — cache is not effective | Minimal benefit; investigate cache key design |
| 50-80% | Moderate — room for improvement | Noticeable latency reduction |
| 80-95% | Good — typical for well-designed caches | Significant database load reduction |
| 95-99% | Excellent — most production systems target this | Database handles only 1-5% of reads |
| > 99% | Outstanding — highly cacheable workload | Database is almost entirely offloaded |

**Factors Affecting Hit Ratio:**

| Factor | Impact |
|--------|--------|
| **Cache size** | Larger cache → more data fits → higher hit ratio |
| **TTL** | Longer TTL → data stays longer → higher hit ratio (but more staleness) |
| **Access pattern** | Skewed access (few hot keys) → higher hit ratio; uniform access → lower |
| **Eviction policy** | LRU works well for temporal locality; LFU for frequency-based access |
| **Cache key design** | Good keys → high reuse; bad keys → low reuse |
| **Data volatility** | Frequently changing data → more invalidations → lower hit ratio |

---

### Cache Warm-up

When a cache is empty (cold start — after deployment, restart, or cache flush), **all requests are cache misses**, causing a sudden spike in database load.

```
Cold Cache (after restart):
  Time T0: Cache is empty
  Time T1: 100% cache misses → all requests hit database → database overloaded!
  Time T2: Cache starts filling up → 50% hit ratio
  Time T3: Cache is warm → 95% hit ratio → database load normalizes
```

**Warm-up Strategies:**

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| **Lazy warm-up** | Let cache fill naturally from misses | Simple; acceptable if database can handle the initial spike |
| **Pre-loading** | Load popular data into cache before serving traffic | When you know the hot keys (top products, popular users) |
| **Snapshot restore** | Dump cache to disk periodically; restore on restart | Redis RDB/AOF; fast warm-up from last known state |
| **Gradual traffic shift** | Route a small percentage of traffic to the new cache, increase gradually | During cache migration or new deployment |
| **Shadow traffic** | Replay production traffic against the new cache without serving responses | Testing cache effectiveness before going live |

```
Pre-loading Example:
  1. Deploy new cache instance
  2. Run warm-up script:
     SELECT id, data FROM products WHERE is_popular = true;
     → For each product: SET cache:product:{id} = data, TTL=3600
  3. Start serving traffic (cache is already warm for popular items)
```

---


## 21.2 Caching Strategies

> Caching strategies define **how** data flows between the application, cache, and database. The right strategy depends on your read/write ratio, consistency requirements, and tolerance for stale data. There are six main strategies, each with different trade-offs.

---

### Cache-Aside (Lazy Loading)

The **application** manages the cache explicitly. It checks the cache first, and on a miss, queries the database and populates the cache.

```
Read Path:
  App → Cache: GET user:123
    HIT  → return cached data
    MISS → App → Database: SELECT * FROM users WHERE id=123
           App → Cache: SET user:123 = data, TTL=3600
           App → return data

Write Path:
  App → Database: UPDATE users SET name='Bob' WHERE id=123
  App → Cache: DELETE user:123  (invalidate — next read will repopulate)
```

```
  +--------+     1. Check     +-------+
  |  App   | ───────────────→ | Cache |
  |        | ←── 2a. HIT ──── |       |
  |        |                   +-------+
  |        |     2b. MISS
  |        | ───────────────→ +----------+
  |        | ←── 3. Data ──── | Database |
  |        |                   +----------+
  |        | ─── 4. Populate → | Cache |
  +--------+                   +-------+
```

| Pros | Cons |
|------|------|
| Simple to implement | Cache miss penalty (extra round trip to DB) |
| Cache only contains requested data (no wasted space) | Stale data possible (DB updated, cache not yet invalidated) |
| Cache failure doesn't break the system (falls back to DB) | Application must manage cache logic |
| Works with any database | Initial requests after cold start are slow |

**Best for:** General-purpose caching. The most common strategy. Good default choice.

> **Ruby Context:** In Rails, `Rails.cache.fetch` implements cache-aside out of the box. It checks the cache, and on a miss, executes the block, stores the result, and returns it. The `redis` gem or `dalli` gem (for Memcached) serve as the backing store.

---

### Read-Through

The **cache** sits between the application and database. The application always reads from the cache. On a miss, the **cache itself** fetches from the database and stores the result.

```
Read Path:
  App → Cache: GET user:123
    HIT  → return cached data
    MISS → Cache → Database: SELECT * FROM users WHERE id=123
           Cache stores the result
           Cache → App: return data

  (Application never talks to the database directly for reads)
```

```
  +--------+     Always read    +-------+     On miss     +----------+
  |  App   | ─────────────────→ | Cache | ──────────────→ | Database |
  |        | ←── Return data ── |       | ←── Fetch ───── |          |
  +--------+                    +-------+                  +----------+
```

| Pros | Cons |
|------|------|
| Simpler application code (no cache management logic) | Cache library/provider must support read-through |
| Cache handles miss logic transparently | Less control over cache population |
| Consistent read path | Same stale data risk as cache-aside |

**Best for:** When using a cache provider that supports read-through natively (e.g., Hazelcast, NCache, Caffeine).

> **Ruby Context:** `Rails.cache.fetch("key") { expensive_query }` effectively acts as a read-through pattern — the block is the "fetch from source" logic. For more explicit read-through behavior, you can wrap a cache store with a custom class that auto-populates on miss.

---

### Write-Through

Every write goes to **both** the cache and the database **synchronously**. The write is only acknowledged after both succeed.

```
Write Path:
  App → Cache: SET user:123 = new_data
  Cache → Database: UPDATE users SET ... WHERE id=123
  (Both succeed) → Acknowledge to App

Read Path:
  App → Cache: GET user:123 → always a HIT (data is always in cache)
```

```
  +--------+     Write     +-------+     Write     +----------+
  |  App   | ────────────→ | Cache | ────────────→ | Database |
  |        | ←── ACK ───── |       | ←── ACK ───── |          |
  +--------+               +-------+               +----------+
```

| Pros | Cons |
|------|------|
| Cache is always consistent with database | Higher write latency (write to both cache AND DB) |
| No stale data (cache is always up to date) | Every write goes through cache (even data that's never read) |
| Simple read path (always hits cache) | Cache may store data that's never read (wasted space) |

**Best for:** Read-heavy workloads where consistency is critical and you can tolerate slightly higher write latency.

---

### Write-Behind (Write-Back)

Writes go to the **cache first** and are acknowledged immediately. The cache **asynchronously** writes to the database in the background.

```
Write Path:
  App → Cache: SET user:123 = new_data → ACK immediately (fast!)
  Cache → (async, batched) → Database: UPDATE users SET ... WHERE id=123

Read Path:
  App → Cache: GET user:123 → HIT (data is in cache)
```

```
  +--------+     Write     +-------+     Async write    +----------+
  |  App   | ────────────→ | Cache | ─ ─ ─ ─ ─ ─ ─ ─→ | Database |
  |        | ←── ACK ───── |       |    (background)    |          |
  +--------+  (immediate)  +-------+                    +----------+
```

| Pros | Cons |
|------|------|
| Lowest write latency (write to cache only, async to DB) | **Data loss risk** — if cache crashes before writing to DB, data is lost |
| Batch writes to DB (reduces DB load) | Complex — must handle async failures, retries, ordering |
| Absorbs write spikes (cache buffers writes) | Cache is the source of truth (dangerous if cache is volatile) |
| Excellent for write-heavy workloads | Eventual consistency between cache and DB |

**Best for:** Write-heavy workloads where slight data loss is acceptable (metrics, analytics, activity logs). **Not suitable for financial data.**

> **Ruby Context:** Write-behind can be implemented in Ruby using background job frameworks like Sidekiq or GoodJob. Write to Redis immediately, then enqueue a job to persist to the database asynchronously.

---

### Write-Around

Writes go **directly to the database**, bypassing the cache. The cache is only populated on reads (cache-aside for reads).

```
Write Path:
  App → Database: UPDATE users SET ... WHERE id=123
  (Cache is NOT updated — may become stale)

Read Path (cache-aside):
  App → Cache: GET user:123
    HIT  → return (may be stale until TTL expires or invalidated)
    MISS → App → Database → App → Cache: SET user:123 = data
```

| Pros | Cons |
|------|------|
| Writes are fast (no cache overhead) | Stale data in cache until TTL expires or invalidated |
| Cache only contains read data (no write-only data wasting space) | Cache miss after every write (until next read) |
| Simple write path | Higher read latency after writes (cache miss) |

**Best for:** Write-heavy workloads where recently written data is rarely read immediately (e.g., log ingestion, batch processing).

---

### Refresh-Ahead

The cache **proactively refreshes** entries that are about to expire, before they're actually requested. This prevents cache misses for popular data.

```
Entry TTL = 60 seconds
Refresh window = last 10 seconds (50-60s mark)

Time 0s:  Cache stores user:123 (TTL=60s)
Time 50s: Cache proactively refreshes user:123 from DB (before expiry)
Time 60s: Old entry would have expired, but fresh data is already in cache
          → No cache miss! No latency spike!
```

| Pros | Cons |
|------|------|
| Eliminates cache miss latency for popular keys | Complexity — must predict which keys will be requested |
| Smooth, predictable latency | Wasted refreshes for keys that won't be requested again |
| No thundering herd on expiry | Requires background refresh mechanism |

**Best for:** Hot keys with predictable access patterns (homepage data, popular products, configuration).

> **Ruby Context:** Refresh-ahead can be implemented with a recurring Sidekiq job or the `clockwork` / `rufus-scheduler` gem that periodically refreshes hot cache keys before they expire.

---

### Strategy Comparison

| Strategy | Read Latency | Write Latency | Consistency | Complexity | Best For |
|----------|-------------|--------------|-------------|-----------|----------|
| **Cache-Aside** | Miss penalty | Low (DB only) | Eventual | Low | General purpose (default choice) |
| **Read-Through** | Miss penalty | Low (DB only) | Eventual | Low | When cache provider supports it |
| **Write-Through** | Always fast | Higher (cache + DB) | Strong | Medium | Read-heavy, consistency critical |
| **Write-Behind** | Always fast | Lowest (cache only) | Eventual | High | Write-heavy, loss-tolerant |
| **Write-Around** | Low (DB only) | Low (DB only) | Eventual | Low | Write-heavy, rarely read after write |
| **Refresh-Ahead** | Always fast | Varies | Near real-time | High | Hot keys, predictable access |

**Most Common Combination:** **Cache-Aside** for reads + **invalidate on write** (delete cache entry when DB is updated). This is what 90% of production systems use.

---


## 21.3 Cache Eviction Policies

> When the cache is full and a new entry needs to be stored, the cache must **evict** (remove) an existing entry to make room. The eviction policy determines **which** entry to remove. The right policy depends on your access pattern.

---

### LRU (Least Recently Used)

Evicts the entry that was **accessed least recently**. Based on the assumption that recently accessed data is likely to be accessed again soon (temporal locality).

```
Cache capacity: 3

Access sequence: A, B, C, D, A, E

Step 1: GET A → miss → cache: [A]
Step 2: GET B → miss → cache: [A, B]
Step 3: GET C → miss → cache: [A, B, C]  (full)
Step 4: GET D → miss → evict A (least recently used) → cache: [B, C, D]
Step 5: GET A → miss → evict B (least recently used) → cache: [C, D, A]
Step 6: GET E → miss → evict C (least recently used) → cache: [D, A, E]
```

**Implementation:** Hash (O(1) lookup) + Doubly Linked List (O(1) move to front / remove from back)

> **Ruby Context:** Ruby's `Hash` maintains insertion order (since Ruby 1.9), which makes it a natural fit for LRU cache implementations. The `lru_redux` gem provides a high-performance LRU cache. Rails also provides `ActiveSupport::Cache::MemoryStore` with a built-in size limit.

```
Most Recently Used ←→ ... ←→ Least Recently Used
     [HEAD]                        [TAIL]
       ↓                             ↓
      [A] ←→ [D] ←→ [C] ←→ [B]
       
On access: move to HEAD
On eviction: remove from TAIL
On insert (full): remove TAIL, insert at HEAD
```

| Pros | Cons |
|------|------|
| Simple and effective for most workloads | Doesn't consider frequency (a one-time access moves to front) |
| O(1) for all operations | Scan pollution — a full table scan evicts all hot data |
| Good for temporal locality | Not optimal for frequency-based access patterns |

**LRU is the default eviction policy for Redis and most caching systems.**

---

### LFU (Least Frequently Used)

Evicts the entry that was **accessed least frequently**. Based on the assumption that frequently accessed data is more valuable than recently accessed data.

```
Cache capacity: 3

Access sequence: A, A, A, B, B, C, D

Step 1-3: GET A (3 times) → cache: [A(3)]
Step 4-5: GET B (2 times) → cache: [A(3), B(2)]
Step 6:   GET C (1 time)  → cache: [A(3), B(2), C(1)]  (full)
Step 7:   GET D → evict C (frequency=1, lowest) → cache: [A(3), B(2), D(1)]
```

| Pros | Cons |
|------|------|
| Better than LRU for frequency-based access | Higher implementation complexity |
| Resistant to scan pollution | Stale popular items stay forever (frequency never decreases) |
| Good for stable popularity distributions | New items are immediately eviction candidates (cold start) |

**LFU with Aging:** To address the staleness problem, some implementations decay frequency counts over time, so old popular items eventually get evicted.

---

### FIFO (First In First Out)

Evicts the **oldest** entry (the one that was inserted first), regardless of access pattern.

```
Cache capacity: 3

Access sequence: A, B, C, D, A, E

Step 1: INSERT A → cache: [A]
Step 2: INSERT B → cache: [A, B]
Step 3: INSERT C → cache: [A, B, C]  (full)
Step 4: INSERT D → evict A (first in) → cache: [B, C, D]
Step 5: ACCESS A → miss → evict B (first in) → cache: [C, D, A]
Step 6: INSERT E → evict C (first in) → cache: [D, A, E]
```

| Pros | Cons |
|------|------|
| Simplest to implement (queue) | Ignores access pattern entirely |
| Predictable behavior | Hot items may be evicted if they were inserted early |
| Low overhead | Generally worse hit ratio than LRU or LFU |

**Use case:** When all items have roughly equal access probability, or when simplicity is more important than optimal hit ratio.

---

### TTL (Time To Live)

Not strictly an eviction policy but a **expiration mechanism**. Each entry has a TTL — after the TTL expires, the entry is automatically removed (or marked as stale).

```
SET user:123 = data, TTL=3600  (expires in 1 hour)

Time 0:    Entry is fresh → cache hit returns data
Time 3599: Entry is still fresh → cache hit
Time 3600: Entry expires → cache miss → fetch from DB → re-cache with new TTL
```

**TTL is typically combined with an eviction policy (e.g., LRU + TTL):**
- TTL ensures data doesn't become too stale
- LRU handles memory pressure when the cache is full
- An entry can be evicted by LRU (memory pressure) OR by TTL (staleness), whichever comes first

**Choosing TTL Values:**

| Data Type | Suggested TTL | Reasoning |
|-----------|--------------|-----------|
| User session | 30 min - 24 hours | Balance security with convenience |
| Product catalog | 5-60 minutes | Products change infrequently |
| User profile | 5-15 minutes | Profiles change occasionally |
| Search results | 1-5 minutes | Results change frequently |
| Configuration | 1-5 minutes | Changes should propagate quickly |
| Static assets (via CDN) | 1 day - 1 year | Rarely change; use cache busting for updates |
| Real-time data (stock prices) | 1-10 seconds | Must be very fresh |
| API rate limit counters | Window duration (1 min, 1 hour) | Must match the rate limit window |

> **Ruby Context:** Both `Rails.cache` and the `redis` gem support TTL natively. In Rails: `Rails.cache.write("key", value, expires_in: 5.minutes)`. With the `redis` gem: `redis.setex("key", 300, value)`. The `dalli` gem (Memcached) also supports TTL: `dalli.set("key", value, 300)`.

---

### Other Eviction Policies

**Random Replacement:**
- Evicts a random entry
- Surprisingly effective in some workloads (avoids worst-case patterns of LRU)
- Very simple to implement
- Used as a baseline for comparison

**ARC (Adaptive Replacement Cache):**
- Combines LRU and LFU adaptively
- Maintains two LRU lists: one for recently accessed items, one for frequently accessed items
- Dynamically adjusts the balance between recency and frequency based on the workload
- Better hit ratio than pure LRU or LFU in many workloads
- More complex to implement; patented by IBM

**Eviction Policy Comparison:**

| Policy | Best For | Hit Ratio | Complexity | Scan Resistant |
|--------|---------|-----------|-----------|---------------|
| **LRU** | General purpose, temporal locality | Good | Low | No |
| **LFU** | Frequency-based access, stable popularity | Better | Medium | Yes |
| **FIFO** | Simple systems, uniform access | Fair | Lowest | No |
| **TTL** | Staleness control (combined with LRU/LFU) | N/A | Low | N/A |
| **Random** | Baseline, simple systems | Fair | Lowest | Yes |
| **ARC** | Mixed workloads, adaptive | Best | High | Yes |

---


## 21.4 Caching Layers

> Caching doesn't happen at just one level — a typical web request passes through **multiple caching layers**, each serving a different purpose. Understanding where caching happens helps you design an effective multi-layer caching strategy.

---

### The Full Caching Stack

```
User's Browser
    ↓
+------------------+
| Browser Cache    |  ← Static assets (CSS, JS, images), DNS, HTTP responses
+------------------+
    ↓
+------------------+
| CDN Cache        |  ← Static content, API responses (edge servers worldwide)
+------------------+
    ↓
+------------------+
| Load Balancer    |  ← (usually pass-through, some do SSL session caching)
+------------------+
    ↓
+------------------+
| Application Cache|  ← In-process cache (Hash, LruRedux, ActiveSupport::Cache)
| (In-Process)     |     Fastest after CPU cache — no network hop
+------------------+
    ↓
+------------------+
| Distributed Cache|  ← Redis, Memcached — shared across app instances
| (Redis)          |     Network hop but still much faster than DB
+------------------+
    ↓
+------------------+
| Database Cache   |  ← Query cache, buffer pool, page cache
| (Buffer Pool)    |     Managed by the database itself
+------------------+
    ↓
+------------------+
| OS Page Cache    |  ← OS caches disk pages in RAM
+------------------+
    ↓
+------------------+
| Disk (SSD/HDD)   |  ← Actual persistent storage
+------------------+
```

---

### Browser Cache

The browser caches HTTP responses locally based on caching headers.

```
First visit:
  Browser → GET /style.css → Server responds with:
    Cache-Control: max-age=31536000, immutable
    ETag: "abc123"
  Browser stores the response locally

Second visit:
  Browser → checks local cache → /style.css is cached and not expired
  → Uses local copy (NO network request at all!)

After expiry (or with no-cache):
  Browser → GET /style.css
    If-None-Match: "abc123"
  Server → 304 Not Modified (use your cached copy)
  (Small network request, but no data transfer)
```

**What Browsers Cache:**
- Static assets: CSS, JavaScript, images, fonts, videos
- API responses (if Cache-Control headers allow it)
- DNS lookups (browser DNS cache, typically 1-2 minutes)
- TLS session tickets (avoid full TLS handshake on reconnection)

> **Ruby Context:** In Rails, the Asset Pipeline (Sprockets) or Propshaft automatically fingerprints assets (e.g., `application-abc123.css`), enabling aggressive browser caching with `Cache-Control: max-age=31536000, immutable`. Rails controllers support `stale?` and `fresh_when` helpers for conditional GET (ETag/Last-Modified) responses.

---

### CDN Cache

CDN edge servers cache content close to users worldwide. Covered in detail in Module 17.5.

**What CDNs Cache:**
- Static assets (images, CSS, JS, videos)
- API responses (with appropriate Cache-Control headers)
- HTML pages (for static or semi-static sites)
- DNS responses

**CDN Cache Headers:**
```
Cache-Control: public, max-age=3600, s-maxage=86400
  → Browser caches for 1 hour
  → CDN caches for 24 hours (s-maxage overrides max-age for shared caches)
```

---

### Application-Level Cache (In-Process)

An **in-process cache** lives inside the application's memory space — no network hop, no serialization. It's the fastest cache after CPU caches.

```
Application Process:
  +------------------------------------------+
  |  Application Code                        |
  |                                          |
  |  +------------------------------------+  |
  |  | In-Process Cache (Hash/LruRedux)   |  |
  |  | user:123 → {name: "Alice", ...}   |  |
  |  | product:456 → {name: "Widget",...} |  |
  |  +------------------------------------+  |
  |                                          |
  +------------------------------------------+
```

**Popular In-Process Cache Libraries:**

| Language | Library | Features |
|----------|---------|----------|
| Java | Caffeine | High-performance, near-optimal hit ratio, async loading |
| Java | Guava Cache | Simple, widely used, part of Google Guava |
| Java | Ehcache | Supports disk overflow, distributed mode |
| Python | `functools.lru_cache` | Built-in decorator, simple LRU |
| Python | cachetools | TTL, LRU, LFU, configurable |
| Go | groupcache | Distributed, singleflight (prevents thundering herd) |
| Ruby | `Hash` + custom eviction / `lru_redux` gem | Manual or gem-based LRU implementation |
| Ruby | `ActiveSupport::Cache::MemoryStore` | Built-in Rails in-process cache with size limits |
| Node.js | node-cache, lru-cache | Simple key-value with TTL |

> **Ruby Context:** Ruby's `Hash` preserves insertion order, making it a natural building block for LRU caches. The `lru_redux` gem provides a fast, thread-safe LRU cache. In Rails, `ActiveSupport::Cache::MemoryStore` is the built-in in-process cache store, configured via `config.cache_store = :memory_store, { size: 64.megabytes }`.

**In-Process vs Distributed Cache:**

| Feature | In-Process | Distributed (Redis) |
|---------|-----------|-------------------|
| Latency | ~100 ns (no network) | ~0.5 ms (network round trip) |
| Shared across instances | No (each instance has its own) | Yes (all instances share) |
| Memory usage | Duplicated across instances | Single shared store |
| Consistency | Each instance may have different data | Single source of truth |
| Failure impact | Lost on process restart | Survives app restarts |
| Best for | Hot data, configuration, small datasets | Shared state, sessions, large datasets |

**Common Pattern — Two-Level Cache (L1 + L2):**

```
Request → L1 (In-Process, MemoryStore) → HIT → return (fastest)
                                        → MISS
          L2 (Distributed, Redis)       → HIT → store in L1 → return
                                        → MISS
          Database                      → store in L2 → store in L1 → return
```

This gives you the speed of in-process caching with the consistency of distributed caching.

> **Ruby Context:** Rails supports multi-layer caching natively. You can configure `config.cache_store = :redis_cache_store` for L2 and use `ActiveSupport::Cache::MemoryStore` as an L1 wrapper. The `identity_cache` gem by Shopify provides a two-level cache (in-process + Memcached) for ActiveRecord models.

---

### Distributed Cache (Redis, Memcached)

A shared cache accessible by all application instances over the network. Covered in detail in Module 19.2 (Redis).

**Key Characteristics:**
- Shared across all application instances (consistent view)
- Survives application restarts (data persists in Redis)
- Network hop adds ~0.5-1ms latency (still 10-100x faster than database)
- Can store much more data than in-process cache (dedicated memory)
- Supports advanced data structures (Redis: sorted sets, streams, pub/sub)

> **Ruby Context:** The `redis` gem is the standard Ruby client for Redis. The `dalli` gem is the standard client for Memcached. In Rails, configure with `config.cache_store = :redis_cache_store, { url: ENV["REDIS_URL"] }` or `config.cache_store = :mem_cache_store, ENV["MEMCACHED_URL"]`. The `connection_pool` gem is recommended for thread-safe Redis connections in multi-threaded servers like Puma.

---

### Database Query Cache

Some databases cache query results internally.

**MySQL Query Cache (deprecated in 8.0):**
- Cached the exact result of SELECT queries
- Any write to a table invalidated ALL cached queries for that table
- Caused contention on high-write workloads (global mutex)
- **Removed in MySQL 8.0** — use application-level caching instead

**PostgreSQL — No Query Cache:**
- PostgreSQL does not have a query cache
- Relies on the buffer pool (shared_buffers) to cache data pages
- Application-level caching (Redis) is the recommended approach

**Buffer Pool / Shared Buffers:**
- Caches data pages (not query results) in memory
- Managed by the database engine (LRU-based eviction)
- PostgreSQL: `shared_buffers` (typically 25% of RAM)
- MySQL/InnoDB: `innodb_buffer_pool_size` (typically 70-80% of RAM)

---

### OS Page Cache

The operating system caches recently accessed disk pages in RAM. This is transparent to the application and database.

```
Database reads a page:
  1. Database → OS: "Read page at offset X"
  2. OS checks page cache:
     HIT  → return from RAM (microseconds)
     MISS → read from disk (milliseconds) → store in page cache → return
  3. Subsequent reads of the same page → served from page cache
```

**Key Point:** The OS page cache and the database buffer pool may cache the same data (double caching). Some databases (like PostgreSQL) rely on the OS page cache for data not in shared_buffers. Others (like MySQL/InnoDB with `O_DIRECT`) bypass the OS page cache to avoid double caching.

---


## 21.5 Cache Invalidation

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

Cache invalidation is the process of removing or updating stale data in the cache when the underlying data changes. Getting it wrong means serving stale data; getting it right is one of the hardest problems in distributed systems.

---

### TTL-Based Expiration

The simplest invalidation strategy — set a TTL on every cache entry. After the TTL expires, the entry is automatically removed.

```
SET user:123 = data, TTL=300  (5 minutes)

Time 0:   Fresh data in cache
Time 300: Entry expires → next read is a cache miss → fetch fresh data from DB
```

| Pros | Cons |
|------|------|
| Simple — no invalidation logic needed | Data is stale for up to TTL duration |
| Automatic — cache manages expiration | Short TTL → more cache misses → more DB load |
| Predictable behavior | Long TTL → more stale data |

**Best for:** Data that changes infrequently and where slight staleness is acceptable (product catalogs, configuration, user profiles).

> **Ruby Context:** In Rails, TTL is set via `expires_in`: `Rails.cache.fetch("user:#{id}", expires_in: 5.minutes) { User.find(id) }`. ActiveRecord model caching with `touch: true` on associations can be combined with TTL to auto-expire parent caches when children change.

---

### Event-Based Invalidation

Invalidate the cache **immediately** when the underlying data changes, triggered by an event.

```
1. App updates user in database
2. App publishes event: "user:123 updated"
3. Cache subscriber receives event → DELETE user:123 from cache
4. Next read → cache miss → fetch fresh data from DB

Or with a message queue:
  App → Database: UPDATE user
  App → Kafka/SNS: publish "user_updated" event
  Cache Invalidation Service → subscribes to events → DELETE from cache
```

```
  +--------+     Write     +----------+
  |  App   | ────────────→ | Database |
  |        |               +----------+
  |        |     Publish event
  |        | ────────────→ +----------------+
  +--------+               | Message Queue  |
                           | (Kafka/SNS)    |
                           +-------+--------+
                                   |
                           +-------v--------+
                           | Cache Invalidation |
                           | Service            |
                           +-------+--------+
                                   |
                           +-------v--------+
                           |     Cache      |
                           | DELETE user:123|
                           +----------------+
```

| Pros | Cons |
|------|------|
| Near real-time invalidation | Complex — requires event infrastructure |
| Minimal stale data window | Event delivery failures can leave stale data |
| Precise — only invalidates changed data | Must ensure event ordering and at-least-once delivery |

**Best for:** Data where freshness is critical (user permissions, pricing, inventory counts).

> **Ruby Context:** Rails provides `ActiveSupport::Notifications` and ActiveRecord callbacks (`after_commit`) for event-based cache invalidation. For distributed systems, use Sidekiq workers subscribing to Redis Pub/Sub, or Kafka consumers via the `karafka` gem. Example: `after_commit { Rails.cache.delete("user:#{id}") }`.

---

### Version-Based Invalidation

Include a **version number** or **hash** in the cache key. When data changes, increment the version — the old cache entry is effectively abandoned (never accessed again).

```
Version 1: cache key = "user:123:v1" → data
Version 2: cache key = "user:123:v2" → new data

Old key "user:123:v1" is never accessed again → eventually evicted by LRU/TTL
```

**Common Implementation:**
```
# Store the current version in a fast lookup (Redis or in-memory)
SET user:123:version = 5

# Cache key includes the version
GET user:123:v5
  HIT → return data
  MISS → fetch from DB → SET user:123:v5 = data

# On update: increment version
INCR user:123:version  → now 6
# Next read uses key "user:123:v6" → cache miss → fresh data
```

| Pros | Cons |
|------|------|
| No explicit invalidation needed | Old versions waste cache space until evicted |
| Atomic — version change instantly invalidates | Requires a version counter (extra storage/lookup) |
| Works well with CDN caching | More complex cache key management |

**Best for:** CDN caching (versioned URLs), API response caching, configuration caching.

> **Ruby Context:** Rails uses version-based cache keys extensively. ActiveRecord's `cache_key_with_version` method (e.g., `"users/123-20231015120000"`) includes the `updated_at` timestamp, so any model update automatically generates a new cache key. This is the foundation of Russian Doll caching in Rails views.

---

### Cache Stampede / Thundering Herd Problem

A **cache stampede** (thundering herd) occurs when a popular cache entry expires and **many concurrent requests** simultaneously experience a cache miss, all hitting the database at once.

```
Normal operation:
  1000 requests/sec for user:123 → all served from cache → DB load: 0

Cache entry expires:
  1000 requests/sec → ALL get cache miss simultaneously
  → 1000 concurrent database queries for the same data!
  → Database overloaded → cascading failure

Timeline:
  T0: Cache entry expires
  T1: Request 1 → cache miss → query DB
  T1: Request 2 → cache miss → query DB
  T1: Request 3 → cache miss → query DB
  ...
  T1: Request 1000 → cache miss → query DB
  → Database receives 1000 identical queries simultaneously!
```

**Solution 1: Locking (Mutex)**

Only one request fetches from the database; others wait for the result.

```
Request 1 → cache miss → acquire lock → query DB → store in cache → release lock
Request 2 → cache miss → lock is held → wait...
Request 3 → cache miss → lock is held → wait...
...
Lock released → Requests 2, 3, ... → cache hit! (data is now cached)

Only 1 database query instead of 1000!
```

```ruby
# Ruby implementation using Redis for distributed locking
def get_user(user_id)
  cache_key = "user:#{user_id}"
  data = Rails.cache.read(cache_key)
  return data if data

  # Cache miss — try to acquire lock
  lock_key = "lock:user:#{user_id}"
  if redis.set(lock_key, "1", nx: true, ex: 10) # SET if Not eXists, 10s expiry
    # Got the lock — fetch from DB
    data = User.find(user_id)
    Rails.cache.write(cache_key, data, expires_in: 1.hour)
    redis.del(lock_key) # release lock
    data
  else
    # Lock is held by another request — wait and retry
    sleep(0.05) # 50ms
    get_user(user_id) # retry (will likely hit cache now)
  end
end
```

> **Ruby Context:** The `redis-mutex` gem and `redlock` gem provide robust distributed mutex implementations for Ruby. Rails 7+ also supports `Rails.cache.fetch` with the `race_condition_ttl` option, which mitigates stampedes by extending the TTL briefly while one process regenerates the value.

**Solution 2: Probabilistic Early Expiration (Stale-While-Revalidate)**

Refresh the cache **before** it actually expires. Each request has a small probability of triggering a background refresh as the TTL approaches.

```
TTL = 3600 seconds (1 hour)
At time T (seconds remaining until expiry):
  probability_of_refresh = exp(-T / beta)
  
  T = 3600 (just cached):  probability ≈ 0% (don't refresh)
  T = 60 (1 min left):     probability ≈ 5% (small chance of refresh)
  T = 10 (10 sec left):    probability ≈ 30% (likely to refresh)
  T = 1 (1 sec left):      probability ≈ 90% (almost certain to refresh)
```

This ensures that **one** request refreshes the cache before expiry, preventing the stampede.

**Solution 3: Stale-While-Revalidate (HTTP Header)**

```
Cache-Control: max-age=3600, stale-while-revalidate=60

Behavior:
  0-3600s:    Serve from cache (fresh)
  3600-3660s: Serve STALE data immediately, refresh in background
  3660s+:     Cache miss (must fetch fresh data)
```

The user gets an instant response (stale data), and the cache is refreshed in the background for the next request.

> **Ruby Context:** In Rails, set these headers in controllers: `expires_in 1.hour, stale_while_revalidate: 60.seconds`. For application-level caching, `Rails.cache.fetch("key", race_condition_ttl: 60.seconds) { expensive_query }` provides similar behavior — it serves stale data while one process regenerates.

**Solution 4: Pre-computation / Refresh-Ahead**

A background job refreshes popular cache entries before they expire.

```
Background Worker (runs every 5 minutes):
  1. Get list of hot keys (keys with high access count)
  2. For each hot key approaching expiry:
     a. Fetch fresh data from database
     b. Update cache with new TTL
  3. Hot keys never expire → no stampede possible
```

> **Ruby Context:** Implement with a recurring Sidekiq job using `sidekiq-cron` or `sidekiq-scheduler`:

```ruby
# app/jobs/cache_refresh_job.rb
class CacheRefreshJob
  include Sidekiq::Job

  def perform
    hot_product_ids = Product.where(popular: true).pluck(:id)

    hot_product_ids.each do |id|
      product = Product.find(id)
      Rails.cache.write("product:#{id}", product, expires_in: 1.hour)
    end
  end
end

# config/sidekiq_cron.yml
# cache_refresh:
#   cron: "*/5 * * * *"   # every 5 minutes
#   class: CacheRefreshJob
```

---


## 21.6 Distributed Caching

> When your application runs on multiple servers, an in-process cache on each server leads to inconsistency (each server has different cached data) and wasted memory (same data cached N times). A **distributed cache** is a shared cache accessible by all application instances, providing a single consistent view of cached data.

---

### Architecture

```
Without Distributed Cache:
  App Server 1: [user:123 = Alice (v1)]     ← stale!
  App Server 2: [user:123 = Alice (v2)]     ← fresh
  App Server 3: [user:123 = not cached]     ← cache miss
  (Each server has its own inconsistent cache)

With Distributed Cache:
  App Server 1 ──┐
  App Server 2 ──┼──→ Redis Cluster: [user:123 = Alice (v2)]
  App Server 3 ──┘    (single source of truth)
```

> **Ruby Context:** In a typical Rails deployment with Puma (multi-threaded) or multiple Unicorn/Passenger workers, each process has its own memory space. A distributed cache (Redis or Memcached) is essential for sharing cached data across all workers and servers. Configure in `config/environments/production.rb`:

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  pool_size: ENV.fetch("RAILS_MAX_THREADS", 5).to_i,
  connect_timeout: 2,
  read_timeout: 1,
  write_timeout: 1,
  reconnect_attempts: 1,
  error_handler: ->(method:, returning:, exception:) {
    Rails.logger.error("Redis cache error: #{exception.message}")
    Sentry.capture_exception(exception) # or your error tracker
  }
}

# Or with Memcached via dalli:
# config.cache_store = :mem_cache_store,
#   ENV["MEMCACHED_URL"],
#   { pool_size: 5, compress: true }
```

---

### Consistent Hashing for Cache Distribution

When distributing cache entries across multiple cache nodes, you need to decide which node stores which key. **Consistent hashing** ensures that adding or removing a node only moves ~1/N of the keys (instead of rehashing everything).

```
Hash Ring:

        Node A (position 0)
       /                    \
  Node D (position 270)    Node B (position 90)
       \                    /
        Node C (position 180)

Key "user:123" → hash = 75 → clockwise → lands on Node B
Key "user:456" → hash = 200 → clockwise → lands on Node D
Key "user:789" → hash = 350 → clockwise → lands on Node A
```

**Adding a Node (Node E at position 135):**
```
Before: Keys 91-180 → Node C
After:  Keys 91-135 → Node E (new)
        Keys 136-180 → Node C (unchanged)

Only keys between 91-135 move to Node E.
All other keys stay on their current nodes.
```

**Virtual Nodes (vnodes):**
With only a few physical nodes, the distribution can be uneven. Virtual nodes solve this by mapping each physical node to multiple positions on the ring.

```
Physical Node A → Virtual nodes at positions: 10, 95, 210, 310
Physical Node B → Virtual nodes at positions: 45, 150, 260, 350

More virtual nodes → more even distribution
Typical: 100-200 virtual nodes per physical node
```

> **Ruby Context:** Redis Cluster handles consistent hashing automatically using 16384 hash slots. The `redis-clustering` gem (or `redis` gem v5+ with cluster mode) supports Redis Cluster natively. For Memcached, the `dalli` gem uses consistent hashing by default across multiple server nodes:

```ruby
# Dalli with multiple Memcached nodes — consistent hashing is automatic
config.cache_store = :mem_cache_store,
  ["memcached1.example.com:11211", "memcached2.example.com:11211"],
  { pool_size: 5 }

# Redis Cluster with the redis gem (v5+)
redis = Redis.new(cluster: [
  "redis://node1:6379",
  "redis://node2:6379",
  "redis://node3:6379"
])
```


---

### Cache Replication

Replicate cache data across multiple nodes for **high availability** and **read scalability**.

```
Redis Replication:
  +--------+     Replication     +-----------+
  | Master | ──────────────────→ | Replica 1 |  ← reads
  |        | ──────────────────→ | Replica 2 |  ← reads
  +--------+                     +-----------+
   ↑ writes

  Writes go to master only
  Reads can go to any replica
  If master fails → promote a replica to master (automatic with Redis Sentinel)
```

**Redis Sentinel (High Availability):**
- Monitors Redis master and replicas
- Automatically promotes a replica to master if the master fails
- Notifies clients of the new master
- Provides a single endpoint that always points to the current master

> **Ruby Context:** The `redis` gem supports Sentinel natively. Configure it to auto-discover the current master:

```ruby
redis = Redis.new(
  url: "redis://mymaster",
  sentinels: [
    { host: "sentinel1.example.com", port: 26379 },
    { host: "sentinel2.example.com", port: 26379 },
    { host: "sentinel3.example.com", port: 26379 }
  ],
  role: :master  # or :slave for read replicas
)

# In Rails:
config.cache_store = :redis_cache_store, {
  url: "redis://mymaster",
  sentinels: [
    { host: "sentinel1.example.com", port: 26379 },
    { host: "sentinel2.example.com", port: 26379 }
  ],
  role: :master
}
```

---

### Cache Partitioning (Sharding)

Distribute cache data across multiple nodes to increase total cache capacity and throughput.

```
Redis Cluster (3 masters, each with a replica):

  Master A (slots 0-5460)      ←→  Replica A1
  Master B (slots 5461-10922)  ←→  Replica B1
  Master C (slots 10923-16383) ←→  Replica C1

  Total capacity: 3x single node
  Total throughput: 3x single node
  
  Key routing: slot = CRC16(key) % 16384
```

**Client-Side Sharding vs Server-Side Sharding:**

| Approach | Description | Used By |
|----------|-------------|---------|
| **Client-side** | Client hashes the key and connects to the correct node | Memcached, Redis (with client library) |
| **Server-side** | Cluster routes requests internally | Redis Cluster |
| **Proxy-based** | A proxy (Twemproxy, Envoy) routes requests | Twemproxy, Codis |

---


## 21.7 Common Caching Problems

> Even with a well-designed caching strategy, several failure modes can cause performance degradation or system outages. Understanding these problems and their solutions is essential for building resilient caching systems.

---

### Cache Penetration (Querying Non-Existent Keys)

**Problem:** Requests for data that **doesn't exist** in the database always result in cache misses (because there's nothing to cache). If an attacker sends millions of requests for non-existent keys, every request hits the database.

```
Attacker → GET user:999999999 → cache miss → DB query → not found → nothing to cache
Attacker → GET user:999999998 → cache miss → DB query → not found → nothing to cache
Attacker → GET user:999999997 → cache miss → DB query → not found → nothing to cache
... (millions of requests, all hitting the database)
```

**Solutions:**

**1. Cache Null Results:**
```
GET user:999999999 → cache miss → DB query → not found
→ Cache: SET user:999999999 = NULL, TTL=60  (cache the "not found" result)

Next request: GET user:999999999 → cache hit → return NULL (no DB query!)
```

Short TTL (60 seconds) prevents caching stale "not found" results if the data is later created.

> **Ruby Context:** In Rails, cache null results explicitly:

```ruby
def find_user(id)
  Rails.cache.fetch("user:#{id}", expires_in: 5.minutes) do
    user = User.find_by(id: id)
    # Cache the nil result too — returns nil on cache hit for non-existent users
    user  # nil is cached, preventing repeated DB queries
  end
end
```

**2. Bloom Filter:**
Use a Bloom filter to check if a key **could exist** before querying the cache or database.

```
Request → Bloom Filter: "Does user:999999999 possibly exist?"
  → "Definitely NOT" → return 404 immediately (no cache or DB query!)
  → "Maybe YES" → proceed to cache → DB (normal flow)
```

```
  +--------+     Check      +---------------+
  |  App   | ──────────────→| Bloom Filter  |
  |        |                | (in memory)   |
  |        | ←── "No" ──── |               |  → Return 404 immediately
  |        | ←── "Maybe" ──|               |  → Proceed to cache/DB
  +--------+                +---------------+
```

The Bloom filter is populated with all valid keys from the database. It uses very little memory (10 bits per key ≈ 1.2 GB for 1 billion keys).

> **Ruby Context:** The `bloomer` or `bloom-filter` gem provides Bloom filter implementations in Ruby. Redis also has a Bloom filter module (`RedisBloom`) accessible via the `redis` gem:

```ruby
# Using the bloomer gem for in-process Bloom filter
require 'bloomer'

bloom = Bloomer.new(1_000_000, 0.01) # 1M items, 1% false positive rate
User.pluck(:id).each { |id| bloom.add("user:#{id}") }

def find_user(id)
  return nil unless bloom.include?("user:#{id}") # Definitely not in DB
  Rails.cache.fetch("user:#{id}", expires_in: 5.minutes) { User.find_by(id: id) }
end
```

**3. Input Validation:**
Validate request parameters before querying. If the ID format is invalid (e.g., negative number, wrong format), reject immediately.

---

### Cache Breakdown (Hot Key Expiration)

**Problem:** A single **extremely popular** cache key expires, and thousands of concurrent requests simultaneously hit the database for the same key. This is a specific case of the cache stampede problem.

```
Hot key: "homepage_data" (10,000 requests/second)

TTL expires → 10,000 concurrent cache misses → 10,000 identical DB queries
→ Database overloaded → cascading failure
```

**Solutions:**

**1. Mutex Lock (covered in 21.5):**
Only one request fetches from DB; others wait.

**2. Never Expire Hot Keys:**
Set no TTL on critical hot keys. Refresh them proactively via a background job.

```ruby
# Background job to refresh hot keys (using Sidekiq)
class HomepageDataRefreshJob
  include Sidekiq::Job

  def perform
    data = HomepageQuery.fetch_all # expensive DB query
    Rails.cache.write("homepage_data", data) # no expires_in — never expires
  end
end

# Schedule every 5 minutes via sidekiq-cron
```

**3. Logical Expiration:**
Store the expiration time inside the cached value (not as a Redis TTL). When a request sees the logical expiry has passed, it triggers a background refresh while returning the stale data.

```json
{
  "data": { ... },
  "logical_expiry": "2025-03-15T15:00:00Z"
}

Request at 15:01:
  → Data is logically expired
  → Return stale data immediately (fast!)
  → Trigger async background refresh
  → Next request gets fresh data
```

```ruby
# Ruby implementation of logical expiration
def get_homepage_data
  cached = Rails.cache.read("homepage_data")

  if cached
    if Time.current > cached[:logical_expiry]
      # Logically expired — return stale data, refresh in background
      HomepageDataRefreshJob.perform_async
    end
    return cached[:data]
  end

  # True cache miss — fetch synchronously
  data = HomepageQuery.fetch_all
  Rails.cache.write("homepage_data", {
    data: data,
    logical_expiry: 5.minutes.from_now
  })
  data
end
```

---

### Cache Avalanche (Mass Expiration)

**Problem:** Many cache entries expire **at the same time**, causing a massive spike in database queries. This can happen when:
- Cache is populated in bulk with the same TTL
- Cache server restarts (all entries lost)
- TTL is set to a round number (e.g., exactly 1 hour for all entries)

```
Time 0:    Bulk load 100,000 entries with TTL=3600
Time 3600: ALL 100,000 entries expire simultaneously
           → 100,000 cache misses → database overwhelmed
```

**Solutions:**

**1. Staggered TTL (Jitter):**
Add random jitter to TTL values so entries expire at different times.

```ruby
base_ttl = 3600 # 1 hour
jitter = rand(0..600) # 0-10 minutes of random jitter
actual_ttl = base_ttl + jitter

Rails.cache.write("product:#{id}", product, expires_in: actual_ttl.seconds)

# Entries expire between 3600-4200 seconds (spread over 10 minutes)
# Instead of all expiring at exactly 3600 seconds
```

**2. Multi-Level Cache (L1 + L2):**
If the distributed cache (L2) fails, the in-process cache (L1) still serves requests.

```
L2 (Redis) goes down:
  Request → L1 (in-process, MemoryStore) → HIT → return (still works!)
  Request → L1 → MISS → Database (only for L1 misses, not all traffic)
```

**3. Circuit Breaker:**
If the database is overwhelmed, stop sending requests and return a fallback response (cached stale data, default values, or a degraded experience).

```
Normal:     Request → Cache miss → Database → return data
Overloaded: Request → Cache miss → Circuit breaker OPEN → return fallback/stale data
Recovery:   Request → Cache miss → Circuit breaker HALF-OPEN → try database → if OK, close circuit
```

> **Ruby Context:** The `circuitbox` or `stoplight` gem provides circuit breaker implementations for Ruby:

```ruby
require 'stoplight'

light = Stoplight('database-queries') do
  User.find(id)
end.with_fallback do |error|
  Rails.cache.read("user:#{id}:stale") || { error: "Service degraded" }
end

result = light.run
```

**4. Cache Pre-warming After Restart:**
After a cache restart, pre-load popular data before serving traffic (see Section 21.1 — Cache Warm-up).

---

### Hot Key Problem

**Problem:** A single key receives a disproportionate amount of traffic, overwhelming the single cache node that stores it.

```
"celebrity_post:123" → 1,000,000 requests/second
All requests go to the same Redis node (because the key hashes to one node)
→ That single node is overwhelmed while others are idle
```

**Solutions:**

**1. Local Cache (L1) for Hot Keys:**
Cache the hot key in each application server's in-process cache. Most requests never reach Redis.

**2. Key Replication (Read Replicas):**
Replicate the hot key across multiple cache nodes. Distribute reads across replicas.

**3. Key Splitting:**
Split the hot key into multiple sub-keys and randomly select one for each request.

```
Instead of: GET celebrity_post:123
Use:        GET celebrity_post:123:shard:{random(0-9)}

10 sub-keys distributed across different Redis nodes
Each sub-key has the same data
Requests are spread across 10 nodes instead of 1
```

```ruby
# Ruby implementation of key splitting for hot keys
def get_celebrity_post(post_id)
  shard = rand(0..9)
  cache_key = "celebrity_post:#{post_id}:shard:#{shard}"

  Rails.cache.fetch(cache_key, expires_in: 5.minutes) do
    Post.find(post_id)
  end
end

# When updating, invalidate all shards
def invalidate_celebrity_post(post_id)
  (0..9).each do |shard|
    Rails.cache.delete("celebrity_post:#{post_id}:shard:#{shard}")
  end
end
```

---

### Summary of Caching Problems and Solutions

| Problem | Cause | Impact | Solutions |
|---------|-------|--------|----------|
| **Cache Penetration** | Queries for non-existent keys | Every request hits DB | Cache null results, Bloom filter, input validation |
| **Cache Breakdown** | Hot key expires | Thousands of simultaneous DB queries | Mutex lock, never-expire hot keys, logical expiration |
| **Cache Avalanche** | Mass expiration at same time | Database overwhelmed | Staggered TTL (jitter), multi-level cache, circuit breaker |
| **Hot Key** | Single key gets extreme traffic | One cache node overwhelmed | Local cache, key replication, key splitting |
| **Cache Stampede** | Popular key expires | Thundering herd to DB | Locking, probabilistic early refresh, stale-while-revalidate |
| **Stale Data** | Cache not invalidated after DB update | Users see outdated data | Event-based invalidation, short TTL, version-based keys |
| **Cold Start** | Empty cache after deployment/restart | All requests hit DB | Pre-warming, snapshot restore, gradual traffic shift |

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Why Cache | 10-100x latency reduction; 80-95% database load reduction; significant cost savings |
| Hit Ratio | Target 90-99%; affected by cache size, TTL, access pattern, eviction policy |
| Cache-Aside | App manages cache; most common strategy; invalidate on write |
| Write-Through | Write to cache + DB synchronously; strong consistency but higher write latency |
| Write-Behind | Write to cache, async to DB; lowest write latency but data loss risk |
| Write-Around | Write to DB only, cache on read; good for write-heavy, rarely-read data |
| LRU | Default eviction policy; evicts least recently used; O(1) with Hash + linked list |
| LFU | Evicts least frequently used; better for stable popularity; more complex |
| TTL | Expiration timer; combine with LRU; add jitter to prevent avalanche |
| Caching Layers | Browser → CDN → In-Process (L1) → Distributed/Redis (L2) → DB Buffer Pool → OS Page Cache |
| Two-Level Cache | L1 (in-process, fastest) + L2 (Redis, shared); best of both worlds |
| TTL Invalidation | Simple but stale; use short TTL for volatile data |
| Event Invalidation | Near real-time; requires event infrastructure (Kafka, SNS) |
| Cache Stampede | Hot key expires → thundering herd; solve with mutex lock or probabilistic early refresh |
| Consistent Hashing | Distribute keys across cache nodes; adding a node moves only ~1/N keys |
| Cache Penetration | Non-existent keys bypass cache; solve with Bloom filter or cache null results |
| Cache Breakdown | Hot key expires; solve with mutex, never-expire, or logical expiration |
| Cache Avalanche | Mass expiration; solve with staggered TTL (jitter) and multi-level cache |
| Hot Key | Single key overwhelms one node; solve with local cache, key splitting, or replication |

---

## Interview Tips for Module 21

1. **Caching strategy** — know all 6 strategies; cache-aside + invalidate-on-write is the default answer; explain when to use write-through or write-behind
2. **Cache-aside implementation** — be ready to write code for cache-aside with TTL and invalidation (in Ruby, use `Rails.cache.fetch` with `expires_in`)
3. **Eviction policies** — explain LRU vs LFU with examples; know that LRU is the default for Redis
4. **LRU implementation** — classic interview question: implement LRU cache with O(1) get/put using Hash + doubly linked list (Ruby's ordered Hash makes this natural)
5. **Cache invalidation** — explain TTL vs event-based vs version-based; discuss the trade-offs of each
6. **Cache stampede** — explain the thundering herd problem and at least 2 solutions (mutex lock, probabilistic early refresh)
7. **Cache penetration** — explain the problem and Bloom filter solution
8. **Cache avalanche** — explain staggered TTL (jitter) as the primary solution
9. **Caching layers** — draw the full stack (browser → CDN → L1 → L2 → DB); explain what each layer caches
10. **Consistent hashing** — explain how it distributes keys and why adding a node only moves ~1/N keys
11. **Redis vs Memcached** — know the key differences (data structures, persistence, replication)
12. **Two-level cache** — explain L1 (in-process) + L2 (Redis) pattern and why it's used
