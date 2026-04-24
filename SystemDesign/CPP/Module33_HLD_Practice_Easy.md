# Module 33: HLD Practice Problems (Easy)

> These are the entry-level system design problems that appear frequently in interviews. Each problem is presented as a complete design walkthrough — requirements, estimation, high-level architecture, deep dive into key components, and trade-offs. Practice these until you can walk through each one in 35-45 minutes.

---

## 33.1 Design a URL Shortener (TinyURL)

> **Problem:** Design a service that takes a long URL and returns a short URL (e.g., `https://tinyurl.com/abc123`). When users visit the short URL, they are redirected to the original long URL.

---

### Step 1: Requirements

**Functional Requirements:**
- Given a long URL, generate a unique short URL
- Given a short URL, redirect to the original long URL
- Short URLs should expire after a configurable time (optional)
- Custom short URLs (optional — user picks the alias)

**Non-Functional Requirements:**
- High availability (URL redirection should never be down)
- Low latency (redirect should be fast — < 100ms)
- Short URLs should not be guessable (not sequential)

**Scale:**
- 100 million new URLs per month
- 10:1 read:write ratio (1 billion redirects per month)

---

### Step 2: Estimation

```
Write QPS:
  100M URLs/month ÷ (30 days × 86,400 sec) ≈ 40 writes/sec
  Peak: 40 × 3 = 120 writes/sec

Read QPS:
  1B redirects/month ÷ (30 × 86,400) ≈ 400 reads/sec
  Peak: 400 × 3 = 1,200 reads/sec

Storage (5 years):
  100M/month × 12 × 5 = 6 billion URLs
  Average record: 500 bytes (short URL + long URL + metadata)
  Total: 6B × 500 bytes = 3 TB

Cache:
  20% of daily URLs: 100M/30 × 20% = 670K URLs × 500 bytes ≈ 335 MB
  → Easily fits in a single Redis instance
```

---

### Step 3: High-Level Architecture

```
  +--------+     +-------------+     +------------------+     +----------+
  | Client | ──→ | API Gateway | ──→ | URL Service      | ──→ | Database |
  |        |     | (rate limit,|     | (shorten/redirect)|    | (NoSQL)  |
  |        |     |  auth)      |     +--------+---------+     +----------+
  +--------+     +-------------+              |
                                     +--------v---------+
                                     | Cache (Redis)    |
                                     +------------------+
```

**API Design:**

```
POST /api/v1/shorten
  Request:  { "long_url": "https://example.com/very/long/path", "expiry": "2026-01-01" }
  Response: { "short_url": "https://tinyurl.com/abc123", "expires_at": "2026-01-01" }

GET /{short_code}  (e.g., GET /abc123)
  Response: 301 Redirect → Location: https://example.com/very/long/path
```

---

### Step 4: Key Design Decisions

**Short URL Generation — Base62 Encoding:**

```
Approach: Generate a unique ID → encode in Base62

Base62 characters: [a-z, A-Z, 0-9] = 62 characters
  6 characters: 62^6 = 56.8 billion combinations (enough for 6B URLs)
  7 characters: 62^7 = 3.5 trillion combinations (massive headroom)

ID Generation Options:
  1. Auto-increment counter (simple but predictable — sequential URLs)
  2. Hash (MD5/SHA-256) of long URL → take first 6-7 chars (collision risk)
  3. Pre-generated unique IDs (Snowflake ID → Base62 encode)
  4. Random generation + collision check (simple, works well at this scale)

Recommended: Counter-based with Base62 encoding
  Counter: 1 → "1"
  Counter: 1000 → "g8"
  Counter: 1000000 → "4c92"
  Counter: 56800235584 → "zzzzzz" (6 chars exhausted)
```

**Handling Collisions (if using hash-based approach):**

```
1. Hash the long URL: MD5("https://example.com/...") → "d41d8cd98f00b204..."
2. Take first 7 characters: "d41d8cd"
3. Check database: does "d41d8cd" already exist?
   → No: insert and return
   → Yes: append a character and retry: "d41d8cd9", check again
```

**301 vs 302 Redirect:**

| Code | Type | Browser Caches? | Analytics Impact |
|------|------|----------------|-----------------|
| **301** | Permanent Redirect | Yes (browser skips our server next time) | Lose visibility into repeat visits |
| **302** | Temporary Redirect | No (browser always hits our server) | Track every visit |

**Recommendation:** Use **302** if you need analytics (track clicks). Use **301** if you want to reduce server load (browser caches the redirect).

---

### Step 5: Database Choice

```
Schema:
  short_code (PK): VARCHAR(7)     — "abc123"
  long_url: VARCHAR(2048)          — the original URL
  user_id: BIGINT                  — who created it (optional)
  created_at: TIMESTAMP
  expires_at: TIMESTAMP (nullable)
  click_count: BIGINT DEFAULT 0

Database Options:
  ✅ DynamoDB: key-value access pattern (lookup by short_code); auto-scaling; managed
  ✅ Cassandra: high write throughput; distributed; good for this access pattern
  ✅ PostgreSQL: if scale is moderate; ACID for counter updates
  
  Recommended: DynamoDB or Cassandra (key-value access pattern, high scale)
```

---

### Step 6: Caching Strategy

```
Read Path (with cache):
  1. Client → GET /abc123
  2. Check Redis cache: GET url:abc123
     → HIT: return long_url (fast!)
     → MISS: query database → store in Redis (TTL=24h) → return long_url

Cache-aside pattern with TTL.
Cache the top 20% of URLs (Pareto — 80% of traffic goes to 20% of URLs).
```

---

### Step 7: Complete Architecture

```
  +--------+     +-------------+     +------------------+
  | Client | ──→ | API Gateway | ──→ | URL Service      |
  +--------+     | - Rate limit|     | - Shorten API    |
                 | - Auth      |     | - Redirect API   |
                 +-------------+     +--------+---------+
                                              |
                                    +---------+---------+
                                    |                   |
                              +-----v-----+    +--------v-------+
                              | Redis     |    | DynamoDB       |
                              | Cache     |    | (short_code →  |
                              | (hot URLs)|    |  long_url)     |
                              +-----------+    +----------------+
                                    
                              +------------------+
                              | Analytics Service|
                              | (click tracking, |
                              |  Kafka → S3)     |
                              +------------------+
```

---


## 33.2 Design a Pastebin

> **Problem:** Design a service where users can paste text content and get a unique URL to share it (like pastebin.com). Content can be viewed by anyone with the URL.

---

### Requirements & Estimation

**Functional:** Create paste (text, optional expiry), read paste by URL, optional syntax highlighting, optional password protection.

**Non-Functional:** High availability for reads, low latency, content should be durable.

**Scale:**
- 5 million new pastes per day
- Read:write ratio: 5:1
- Average paste size: 10 KB
- Max paste size: 10 MB

```
Write QPS: 5M / 86,400 ≈ 58/sec (peak: ~175/sec)
Read QPS: 25M / 86,400 ≈ 290/sec (peak: ~870/sec)

Storage (5 years):
  5M/day × 365 × 5 = 9.1 billion pastes
  Metadata: 9.1B × 200 bytes = 1.8 TB
  Content: 9.1B × 10 KB = 91 TB (but most pastes are small — median ~2 KB)
  Realistic content: ~20 TB
```

---

### Architecture

```
  +--------+     +-------------+     +------------------+     +------------------+
  | Client | ──→ | API Gateway | ──→ | Paste Service    | ──→ | Metadata DB      |
  +--------+     +-------------+     |                  |     | (PostgreSQL/     |
                                     |                  |     |  DynamoDB)       |
                                     +--------+---------+     +------------------+
                                              |
                                     +--------v---------+
                                     | Object Storage   |
                                     | (S3)             |
                                     | - Paste content  |
                                     +------------------+
                                              |
                                     +--------v---------+
                                     | CDN (CloudFront) |
                                     | - Cache popular  |
                                     |   pastes         |
                                     +------------------+
```

**Key Design Decisions:**

| Decision | Choice | Why |
|----------|--------|-----|
| **Content storage** | S3 (object storage) | Cheap, durable (11 nines), handles large pastes; don't store 10 MB blobs in a database |
| **Metadata storage** | PostgreSQL or DynamoDB | Small records (paste_id, user_id, created_at, expires_at, s3_key) |
| **URL generation** | Same as URL shortener (Base62 of unique ID) | 6-8 character unique paste ID |
| **Expiration** | S3 lifecycle policy + DB TTL index | Auto-delete expired pastes |
| **CDN** | CloudFront in front of S3 | Cache popular pastes at edge; reduce S3 reads |

**API Design:**

```
POST /api/v1/pastes
  Request:  { "content": "print('hello')", "language": "python", "expires_in": 86400 }
  Response: { "paste_id": "abc123", "url": "https://paste.example.com/abc123" }

GET /api/v1/pastes/{paste_id}
  Response: { "paste_id": "abc123", "content": "print('hello')", "language": "python", 
              "created_at": "2025-03-15T10:30:00Z" }

GET /raw/{paste_id}
  Response: print('hello')  (plain text, for embedding/curl)
```

**Write Flow:**
1. Client sends paste content to Paste Service
2. Paste Service generates unique paste_id (Base62)
3. Paste Service uploads content to S3: `s3://pastes/{paste_id}`
4. Paste Service stores metadata in DB: `{paste_id, s3_key, language, created_at, expires_at}`
5. Return paste URL to client

**Read Flow:**
1. Client requests `GET /abc123`
2. CDN cache check → HIT: return cached content
3. MISS: Paste Service reads metadata from DB → fetches content from S3 → returns to client (CDN caches it)

---


## 33.3 Design a Rate Limiter

> **Problem:** Design a distributed rate limiter that limits the number of requests a client can make to an API within a time window. It should work across multiple API servers.

---

### Requirements & Estimation

**Functional:** Limit requests per user/IP/API key per time window. Return 429 when limit exceeded. Support different limits per endpoint. Configurable rules.

**Non-Functional:** Low latency (< 1ms overhead per request). High availability. Accurate counting (minimal over/under-counting). Distributed (works across multiple servers).

**Scale:**
- 10 million active users
- 1 million requests per second (peak)
- Rate limit: 100 requests per minute per user

---

### Rate Limiting Algorithms

**1. Token Bucket:**

```
Bucket capacity: 100 tokens
Refill rate: 100 tokens per minute (1.67 tokens/second)

Request arrives:
  If tokens > 0: consume 1 token → ALLOW
  If tokens = 0: → REJECT (429 Too Many Requests)

Tokens refill continuously at the refill rate.
Allows bursts up to bucket capacity.

Timeline:
  T=0:  100 tokens (full bucket)
  T=0:  50 requests arrive → 50 tokens consumed → 50 remaining → all ALLOWED
  T=1s: 1.67 tokens refilled → 51.67 tokens
  T=30s: 50 tokens refilled → bucket back to 100 (capped at capacity)
```

| Pros | Cons |
|------|------|
| Allows controlled bursts | Slightly complex to implement |
| Smooth rate limiting | Must track per-user state |
| Memory efficient (2 values per user: tokens, last_refill_time) | |

**2. Sliding Window Log:**

```
Store the timestamp of every request in a sorted set.
For each new request:
  1. Remove all timestamps older than (now - window_size)
  2. Count remaining timestamps
  3. If count < limit → ALLOW (add timestamp)
  4. If count >= limit → REJECT

Example (limit: 5 requests per minute):
  Timestamps: [10:00:15, 10:00:30, 10:00:45, 10:01:00, 10:01:10]
  New request at 10:01:20:
    Remove timestamps before 10:00:20: remove 10:00:15
    Remaining: [10:00:30, 10:00:45, 10:01:00, 10:01:10] = 4 requests
    4 < 5 → ALLOW (add 10:01:20)
```

| Pros | Cons |
|------|------|
| Very accurate (no boundary issues) | High memory (stores every timestamp) |
| Smooth sliding window | O(n) cleanup per request |

**3. Sliding Window Counter:**

```
Combine fixed window counts with a weighted average for the sliding window.

Previous window count: 80 requests (10:00 - 10:01)
Current window count: 30 requests (10:01 - 10:02)
Current position: 10:01:15 (25% into current window)

Estimated count = previous × (1 - position%) + current × position%
                = 80 × 0.75 + 30 × 0.25
                = 60 + 7.5 = 67.5

If limit = 100: 67.5 < 100 → ALLOW
```

| Pros | Cons |
|------|------|
| Memory efficient (2 counters per user per window) | Approximate (not exact) |
| Fast (O(1) per request) | Slight inaccuracy at window boundaries |
| Good balance of accuracy and performance | |

---

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ──→ | API Gateway /    | ──→ | Rate Limiter     |
  +--------+     | Load Balancer    |     | Middleware       |
                 +------------------+     +--------+---------+
                                                   |
                                          +--------v---------+
                                          | Redis Cluster    |
                                          | (rate limit      |
                                          |  counters)       |
                                          +------------------+
                                                   |
                                          +--------v---------+
                                          | Rules DB         |
                                          | (rate limit      |
                                          |  configurations) |
                                          +------------------+
```

**Redis Implementation (Token Bucket):**

```
For each request from user_123:

  -- Lua script (atomic in Redis)
  local key = "rate_limit:user_123"
  local now = tonumber(ARGV[1])
  local rate = 100          -- tokens per minute
  local capacity = 100      -- max tokens
  
  local bucket = redis.call("HMGET", key, "tokens", "last_refill")
  local tokens = tonumber(bucket[1]) or capacity
  local last_refill = tonumber(bucket[2]) or now
  
  -- Refill tokens based on elapsed time
  local elapsed = now - last_refill
  local new_tokens = math.min(capacity, tokens + elapsed * rate / 60)
  
  if new_tokens >= 1 then
    -- Allow request
    redis.call("HMSET", key, "tokens", new_tokens - 1, "last_refill", now)
    redis.call("EXPIRE", key, 120)  -- cleanup after 2 minutes of inactivity
    return 1  -- ALLOWED
  else
    -- Reject request
    return 0  -- REJECTED
  end
```

**Why Redis?**
- Atomic operations (Lua scripts execute atomically — no race conditions)
- Sub-millisecond latency
- Shared across all API servers (distributed rate limiting)
- TTL for automatic cleanup of inactive users

**Rate Limit Headers:**

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 73
X-RateLimit-Reset: 1625097660

HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1625097660
```

---


## 33.4 Design a Key-Value Store

> **Problem:** Design a distributed key-value store (like DynamoDB or Redis) that supports `GET(key)` and `PUT(key, value)` operations with high availability and low latency.

---

### Requirements & Estimation

**Functional:** `PUT(key, value)`, `GET(key)`, `DELETE(key)`. Support large values (up to 1 MB). Configurable consistency (strong or eventual).

**Non-Functional:** High availability (99.99%), low latency (< 10ms P99), horizontal scalability, tunable consistency.

**Scale:**
- 10 billion key-value pairs
- Average key: 50 bytes, average value: 1 KB
- Read QPS: 500,000; Write QPS: 100,000

```
Storage:
  10B × (50 bytes + 1 KB) ≈ 10 TB
  With 3x replication: 30 TB
  Per node (2 TB each): 15 nodes minimum

Memory for index:
  10B keys × 50 bytes = 500 GB (too large for memory — need disk-based index)
```

---

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ──→ | Coordinator      | ──→ | Storage Node 1   |
  +--------+     | (any node can    |     | (owns hash range |
                 |  be coordinator) |     |  0-3000)         |
                 +------------------+     +------------------+
                          |               +------------------+
                          +──────────────→| Storage Node 2   |
                          |               | (owns hash range |
                          |               |  3001-6000)      |
                          |               +------------------+
                          |               +------------------+
                          +──────────────→| Storage Node 3   |
                                          | (owns hash range |
                                          |  6001-9000)      |
                                          +------------------+
```

**Key Components:**

| Component | Responsibility |
|-----------|---------------|
| **Coordinator** | Receives client requests; determines which nodes own the key (consistent hashing); forwards requests; collects responses |
| **Storage Nodes** | Store key-value pairs; handle replication; run compaction |
| **Consistent Hash Ring** | Maps keys to nodes; virtual nodes for even distribution |
| **Gossip Protocol** | Membership management; failure detection; metadata propagation |
| **Replication** | Each key stored on N nodes (typically N=3) for durability |

---

### Data Partitioning (Consistent Hashing)

```
Hash Ring with virtual nodes:

  Node A: positions [100, 400, 700]
  Node B: positions [200, 500, 800]
  Node C: positions [300, 600, 900]

  Key "user:123" → hash = 350 → clockwise → Node A (position 400)
  Replicas: Node A (primary), Node B (position 500), Node C (position 600)
```

---

### Write Path

```
PUT("user:123", "{name: Alice}")

1. Client → Coordinator (any node)
2. Coordinator hashes "user:123" → determines primary node (A) and replicas (B, C)
3. Coordinator forwards write to A, B, C in parallel
4. Each node:
   a. Write to commit log (WAL) — sequential append, durable
   b. Write to memtable (in-memory sorted structure)
   c. ACK to coordinator
5. Coordinator waits for W acknowledgments (W=2 for quorum)
6. Coordinator responds to client: "Write successful"

Background:
  When memtable is full → flush to SSTable on disk
  Periodic compaction merges SSTables
```

---

### Read Path

```
GET("user:123")

1. Client → Coordinator
2. Coordinator hashes "user:123" → determines nodes A, B, C
3. Coordinator sends read to R nodes (R=2 for quorum)
4. Each node:
   a. Check memtable (in-memory)
   b. Check Bloom filters for each SSTable
   c. Read from SSTables (disk)
   d. Return value + version/timestamp
5. Coordinator compares responses:
   → If all agree: return value
   → If conflict: return latest version (or all versions for client resolution)
6. Read repair: if a node had stale data, send it the latest version
```

---

### Consistency & Replication

```
N = 3 (replication factor)
W = 2 (write quorum)
R = 2 (read quorum)
W + R = 4 > N = 3 → Strong consistency

Tunable per request:
  Strong consistency: W=2, R=2 (overlap guaranteed)
  Eventual consistency: W=1, R=1 (fastest, may read stale)
  Write-heavy: W=1, R=3 (fast writes, slow reads)
  Read-heavy: W=3, R=1 (slow writes, fast reads)
```

---

### Failure Handling

| Failure | Detection | Recovery |
|---------|-----------|---------|
| **Node temporarily down** | Gossip protocol (heartbeat timeout) | Hinted handoff — write to another node, forward when recovered |
| **Node permanently failed** | Extended timeout (e.g., 24 hours) | Replace node; re-replicate data from remaining replicas |
| **Network partition** | Gossip detects unreachable nodes | Sloppy quorum — write to available nodes; reconcile after partition heals |
| **Data corruption** | Checksum verification (Merkle trees) | Read repair; anti-entropy repair from healthy replicas |

**Merkle Trees for Anti-Entropy:**
Each node maintains a Merkle tree (hash tree) of its data. Nodes periodically compare Merkle trees — if a subtree hash differs, only that subtree's data needs to be synchronized (efficient — don't need to compare every key).

---


## 33.5 Design a Unique ID Generator

> **Problem:** Design a system that generates globally unique IDs at scale — 10,000+ IDs per second across multiple data centers. IDs should be roughly sortable by time.

---

### Requirements

**Functional:** Generate unique 64-bit numeric IDs. IDs should be roughly time-ordered (newer IDs > older IDs). No coordination between ID generators (no single point of failure).

**Non-Functional:** High availability, low latency (< 1ms), no duplicates (ever), horizontally scalable.

---

### Approaches

**1. UUID (Universally Unique Identifier):**

```
UUID v4 (random): 550e8400-e29b-41d4-a716-446655440000
  128 bits = 32 hex characters + 4 hyphens
  
  Probability of collision: ~1 in 2^122 (virtually impossible)
```

| Pros | Cons |
|------|------|
| No coordination needed (generate anywhere) | 128 bits (16 bytes) — too large for some use cases |
| Virtually zero collision probability | Not sortable by time (random) |
| Simple to implement | Poor index performance (random → B-Tree page splits) |
| Available in every language | Not human-friendly (long, hard to read) |

**Best for:** Distributed systems where size doesn't matter and time-ordering isn't needed.

**UUID v7 (time-ordered):** Encodes a Unix timestamp in the first 48 bits, followed by random bits. Time-sortable AND unique. Best UUID variant for databases.

---

**2. Database Auto-Increment:**

```
INSERT INTO ids DEFAULT VALUES RETURNING id;
→ id = 1, 2, 3, 4, 5, ...
```

| Pros | Cons |
|------|------|
| Simple | Single point of failure (one database) |
| Sequential (great for B-Tree indexes) | Doesn't scale horizontally |
| Small (64-bit integer) | Predictable (security concern — user can guess IDs) |

**Multi-master auto-increment:**
```
Server 1: generates 1, 3, 5, 7, 9, ...  (odd numbers)
Server 2: generates 2, 4, 6, 8, 10, ... (even numbers)

General: Server K generates IDs where id % N == K (N = number of servers)
```

| Pros | Cons |
|------|------|
| Scales to N servers | Adding/removing servers changes the pattern |
| No coordination needed | IDs are not globally sorted by time |
| Simple | Hard to add more servers later |

---

**3. Snowflake ID (Twitter's Approach) — Recommended:**

```
Snowflake ID: 64-bit integer

  +--+-------------------+----------+------------+
  |0 | Timestamp (41 bits)| Machine  | Sequence   |
  |  | (milliseconds     | ID       | Number     |
  |  |  since epoch)     | (10 bits)| (12 bits)  |
  +--+-------------------+----------+------------+
   1        41                10          12
  bit      bits              bits        bits

  Timestamp (41 bits): milliseconds since custom epoch
    → 2^41 ms = ~69 years of unique timestamps
    
  Machine ID (10 bits): identifies the generator instance
    → 2^10 = 1,024 unique machines
    
  Sequence Number (12 bits): counter within the same millisecond
    → 2^12 = 4,096 IDs per millisecond per machine
    
  Total capacity: 1,024 machines × 4,096 IDs/ms = 4,194,304 IDs/ms
                  = ~4 billion IDs per second (across all machines)
```

```
Example Snowflake ID:
  Timestamp: 1625097600000 (2021-07-01 00:00:00 UTC)
  Machine ID: 42
  Sequence: 7
  
  Binary: 0 | 10111101001001010000000000000000000000000 | 0000101010 | 000000000111
  Decimal: 6820940800000172039 (fits in a 64-bit integer)
```

| Pros | Cons |
|------|------|
| 64-bit integer (compact, fits in BIGINT) | Requires machine ID assignment (ZooKeeper or config) |
| Time-sortable (newer IDs > older IDs) | Clock skew can cause issues (non-monotonic timestamps) |
| No coordination for ID generation | 69-year limit (from custom epoch) |
| 4K IDs/ms per machine (very high throughput) | Machine ID must be unique across the cluster |
| Decentralized (each machine generates independently) | |

**Handling Clock Skew:**
```
If the system clock moves backward (NTP adjustment):
  Option 1: Wait until the clock catches up (block ID generation briefly)
  Option 2: Use the last known timestamp (don't go backward)
  Option 3: Use a logical clock component that always increments
```

---

**4. ULID (Universally Unique Lexicographically Sortable Identifier):**

```
ULID: 01ARZ3NDEKTSV4RRFFQ69G5FAV
  128 bits total:
    Timestamp: 48 bits (milliseconds, ~8,900 years)
    Randomness: 80 bits
  
  Encoded as 26 Crockford Base32 characters
  Lexicographically sortable (string comparison = time comparison)
```

| Pros | Cons |
|------|------|
| Time-sortable (like Snowflake) | 128 bits (larger than Snowflake's 64 bits) |
| No machine ID needed (random component) | Slightly higher collision risk than Snowflake |
| String-sortable (useful for databases that sort by string) | Less common than UUID |
| No coordination needed | |

---

### Architecture (Snowflake-based)

```
  +------------------+
  | ZooKeeper / etcd |  (assigns unique machine IDs to each generator)
  +--------+---------+
           |
  +--------v---------+  +------------------+  +------------------+
  | ID Generator 1   |  | ID Generator 2   |  | ID Generator 3   |
  | Machine ID: 1    |  | Machine ID: 2    |  | Machine ID: 3    |
  | (generates IDs   |  | (generates IDs   |  | (generates IDs   |
  |  independently)  |  |  independently)  |  |  independently)  |
  +------------------+  +------------------+  +------------------+
           ↑                     ↑                     ↑
    Service A calls        Service B calls        Service C calls
    for new IDs            for new IDs            for new IDs
```

**Machine ID Assignment:**
- On startup, each generator requests a unique machine ID from ZooKeeper (ephemeral node)
- If the generator crashes, the ephemeral node is deleted → machine ID is released
- New generator gets a new (or recycled) machine ID

---

### Comparison Summary

| Approach | Size | Sortable | Coordination | Throughput | Best For |
|----------|------|----------|-------------|-----------|----------|
| **UUID v4** | 128 bits | No | None | Unlimited | Simple, no ordering needed |
| **UUID v7** | 128 bits | Yes | None | Unlimited | Databases, time-ordered UUIDs |
| **Auto-increment** | 64 bits | Yes | Database (SPOF) | Limited by DB | Small scale, single DB |
| **Snowflake** | 64 bits | Yes | Machine ID only | 4K/ms/machine | Large scale, compact IDs |
| **ULID** | 128 bits | Yes | None | Unlimited | String-sortable, no coordination |

**Recommendation:** Use **Snowflake IDs** for most large-scale systems (compact, sortable, high throughput). Use **UUID v7** if you don't want to manage machine IDs. Use **auto-increment** for simple, single-database systems.

---

## Summary & Key Takeaways

| Problem | Key Design Decisions |
|---------|---------------------|
| **URL Shortener** | Base62 encoding of unique ID; 301 vs 302 redirect; cache hot URLs in Redis; DynamoDB for key-value access |
| **Pastebin** | Store content in S3 (not database); metadata in DB; CDN for popular pastes; lifecycle policies for expiration |
| **Rate Limiter** | Token Bucket or Sliding Window Counter; Redis for distributed state; Lua scripts for atomicity; return 429 + headers |
| **Key-Value Store** | Consistent hashing for partitioning; quorum reads/writes (W+R>N); gossip for membership; LSM-tree storage; Merkle trees for anti-entropy |
| **Unique ID Generator** | Snowflake ID (64-bit: timestamp + machine ID + sequence); ZooKeeper for machine ID assignment; handle clock skew |

---

## Interview Tips for Module 33

1. **Start with requirements** — clarify functional and non-functional requirements before designing
2. **Do the math** — estimate QPS, storage, bandwidth before choosing architecture
3. **URL Shortener** — know Base62 encoding, 301 vs 302 trade-off, caching strategy
4. **Pastebin** — separate metadata (DB) from content (S3); mention CDN for popular pastes
5. **Rate Limiter** — explain Token Bucket algorithm step by step; show Redis Lua script for atomicity; mention distributed challenges
6. **Key-Value Store** — draw the consistent hash ring; explain quorum (W+R>N); explain write path (WAL → memtable → SSTable)
7. **ID Generator** — explain Snowflake ID bit layout; mention clock skew handling; compare with UUID and auto-increment
8. **Always mention** — caching, monitoring, scaling, failure handling for every problem
9. **Draw diagrams** — architecture diagrams, data flow, sequence diagrams
10. **Discuss trade-offs** — every decision has trade-offs; articulate them clearly
