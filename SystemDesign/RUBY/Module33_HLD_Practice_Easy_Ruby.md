# Module 33: HLD Practice Problems (Easy)

> These are the entry-level system design problems that appear frequently in interviews. Each problem is presented as a complete design walkthrough — requirements, estimation, high-level architecture, deep dive into key components, and trade-offs. Practice these until you can walk through each one in 35-45 minutes.

> **Ruby Context:** Each problem includes Ruby/Rails implementation notes showing how you'd build these systems with the Ruby ecosystem. Key tools referenced: **Rails API mode** (lightweight services), **Sidekiq** (background jobs), **Redis** (`redis-rb` for caching and rate limiting), **ActiveRecord** / **DynamoDB** (`aws-sdk-dynamodb`), **Karafka** (Kafka events), **aws-sdk-s3** (object storage), and **Flipper** (feature flags). For ID generation, Ruby has `SecureRandom`, the `ulid` gem, and custom Snowflake implementations.

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

Rails server estimation:
  Peak read QPS: 1,200 (very manageable)
  Puma throughput: ~1,000 req/sec per instance (cached reads)
  → 2 Puma instances with headroom
```

---

### Step 3: High-Level Architecture

```
  +--------+     +-------------+     +------------------+     +----------+
  | Client | ──→ | API Gateway | ──→ | URL Service      | ──→ | Database |
  |        |     | (rate limit,|     | (Rails API)      |     | (DynamoDB|
  |        |     |  auth)      |     +--------+---------+     | or PG)   |
  +--------+     +-------------+              |               +----------+
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
```

> **Ruby Context:** Here's a Rails implementation of the core URL shortening logic.

```ruby
# app/services/url_shortener_service.rb
class UrlShortenerService
  BASE62_CHARS = ('a'..'z').to_a + ('A'..'Z').to_a + ('0'..'9').to_a

  def self.shorten(long_url, expires_at: nil)
    # Generate unique short code
    short_code = generate_short_code

    # Store in database
    shortened_url = ShortenedUrl.create!(
      short_code: short_code,
      long_url: long_url,
      expires_at: expires_at
    )

    # Warm the cache
    Rails.cache.write("url:#{short_code}", long_url, expires_in: 24.hours)

    shortened_url
  end

  def self.resolve(short_code)
    # Check cache first
    long_url = Rails.cache.fetch("url:#{short_code}", expires_in: 24.hours) do
      # Cache miss — query database
      record = ShortenedUrl.find_by(short_code: short_code)
      return nil unless record
      return nil if record.expires_at&.past?
      record.long_url
    end

    # Track click asynchronously (don't slow down the redirect)
    ClickTrackingWorker.perform_async(short_code) if long_url

    long_url
  end

  private

  def self.generate_short_code(length: 7)
    loop do
      code = Array.new(length) { BASE62_CHARS.sample }.join
      break code unless ShortenedUrl.exists?(short_code: code)
    end
  end

  # Alternative: Base62 encode a counter/Snowflake ID
  def self.base62_encode(number)
    return BASE62_CHARS[0] if number.zero?
    result = ''
    while number > 0
      result.prepend(BASE62_CHARS[number % 62])
      number /= 62
    end
    result
  end
end

# app/controllers/api/v1/urls_controller.rb
class Api::V1::UrlsController < ApplicationController
  def create
    result = UrlShortenerService.shorten(
      params[:long_url],
      expires_at: params[:expiry]
    )
    render json: {
      short_url: "https://#{request.host}/#{result.short_code}",
      expires_at: result.expires_at
    }, status: :created
  end
end

# app/controllers/redirects_controller.rb
class RedirectsController < ApplicationController
  def show
    long_url = UrlShortenerService.resolve(params[:short_code])
    if long_url
      redirect_to long_url, status: :moved_permanently, allow_other_host: true
    else
      render plain: 'URL not found', status: :not_found
    end
  end
end

# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :urls, only: [:create]
    end
  end
  get '/:short_code', to: 'redirects#show', constraints: { short_code: /[a-zA-Z0-9]{6,7}/ }
end
```

**301 vs 302 Redirect:**

| Code | Type | Browser Caches? | Analytics Impact |
|------|------|----------------|-----------------|
| **301** | Permanent Redirect | Yes (browser skips our server next time) | Lose visibility into repeat visits |
| **302** | Temporary Redirect | No (browser always hits our server) | Track every visit |

**Recommendation:** Use **302** if you need analytics (track clicks). Use **301** if you want to reduce server load.

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

> **Ruby Context:** For moderate scale, PostgreSQL with ActiveRecord works well. For high scale, use DynamoDB via `aws-sdk-dynamodb`.

```ruby
# ActiveRecord model (PostgreSQL)
class ShortenedUrl < ApplicationRecord
  validates :short_code, presence: true, uniqueness: true
  validates :long_url, presence: true

  scope :active, -> { where('expires_at IS NULL OR expires_at > ?', Time.current) }
end

# DynamoDB alternative (aws-sdk-dynamodb)
class DynamoUrlStore
  def initialize
    @client = Aws::DynamoDB::Client.new
    @table = 'shortened_urls'
  end

  def put(short_code, long_url, expires_at: nil)
    @client.put_item(
      table_name: @table,
      item: {
        'short_code' => short_code,
        'long_url' => long_url,
        'created_at' => Time.now.to_i,
        'expires_at' => expires_at&.to_i
      }.compact
    )
  end

  def get(short_code)
    resp = @client.get_item(table_name: @table, key: { 'short_code' => short_code })
    resp.item
  end
end
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
  | Client | ──→ | API Gateway | ──→ | Rails API        |
  +--------+     | - Rate limit|     | (Puma)           |
                 | - Auth      |     | - Shorten API    |
                 +-------------+     | - Redirect API   |
                                     +--------+---------+
                                              |
                                    +---------+---------+
                                    |                   |
                              +-----v-----+    +--------v-------+
                              | Redis     |    | DynamoDB /     |
                              | Cache     |    | PostgreSQL     |
                              | (hot URLs)|    | (short_code →  |
                              +-----------+    |  long_url)     |
                                               +----------------+
                              +------------------+
                              | Sidekiq Workers  |
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
  +--------+     +-------------+     | (Rails API)      |     | (PostgreSQL)     |
                                     |                  |     +------------------+
                                     +--------+---------+
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

> **Ruby Context:** This is a natural fit for Rails with ActiveStorage (S3 backend) or direct `aws-sdk-s3` usage.

```ruby
# app/models/paste.rb
class Paste < ApplicationRecord
  validates :paste_id, presence: true, uniqueness: true
  validates :language, inclusion: { in: %w[text ruby python javascript go java] }, allow_nil: true

  scope :active, -> { where('expires_at IS NULL OR expires_at > ?', Time.current) }

  def content_url
    # Pre-signed S3 URL for private content
    s3_presigner.presigned_url(:get_object, bucket: ENV['S3_BUCKET'], key: s3_key, expires_in: 3600)
  end

  def s3_key
    "pastes/#{paste_id}"
  end

  private

  def s3_presigner
    @s3_presigner ||= Aws::S3::Presigner.new
  end
end

# app/services/paste_service.rb
class PasteService
  def self.create(content:, language: nil, expires_in: nil)
    paste_id = generate_paste_id
    s3_key = "pastes/#{paste_id}"

    # Upload content to S3
    s3_client.put_object(
      bucket: ENV['S3_BUCKET'],
      key: s3_key,
      body: content,
      content_type: 'text/plain',
      server_side_encryption: 'AES256'
    )

    # Store metadata in database
    Paste.create!(
      paste_id: paste_id,
      s3_key: s3_key,
      language: language,
      content_size: content.bytesize,
      expires_at: expires_in ? Time.current + expires_in : nil
    )
  end

  def self.read(paste_id)
    paste = Paste.active.find_by!(paste_id: paste_id)

    # Fetch content from S3 (CDN handles caching)
    response = s3_client.get_object(bucket: ENV['S3_BUCKET'], key: paste.s3_key)
    content = response.body.read

    { paste: paste, content: content }
  end

  private

  def self.generate_paste_id
    # Same Base62 approach as URL shortener
    loop do
      id = SecureRandom.alphanumeric(8)
      break id unless Paste.exists?(paste_id: id)
    end
  end

  def self.s3_client
    @s3_client ||= Aws::S3::Client.new
  end
end

# app/controllers/api/v1/pastes_controller.rb
class Api::V1::PastesController < ApplicationController
  def create
    result = PasteService.create(
      content: params[:content],
      language: params[:language],
      expires_in: params[:expires_in]&.to_i
    )
    render json: {
      paste_id: result.paste_id,
      url: "https://paste.example.com/#{result.paste_id}"
    }, status: :created
  end

  def show
    data = PasteService.read(params[:id])
    render json: {
      paste_id: data[:paste].paste_id,
      content: data[:content],
      language: data[:paste].language,
      created_at: data[:paste].created_at.iso8601
    }
  rescue ActiveRecord::RecordNotFound
    render json: { error: 'Paste not found' }, status: :not_found
  end
end

# Background job to clean up expired pastes
class ExpiredPasteCleanupWorker
  include Sidekiq::Job
  sidekiq_options queue: :maintenance

  def perform
    Paste.where('expires_at < ?', Time.current).find_each do |paste|
      s3_client.delete_object(bucket: ENV['S3_BUCKET'], key: paste.s3_key)
      paste.destroy
    end
  end
end
```

**Write Flow:**
1. Client sends paste content to Rails API
2. PasteService generates unique paste_id
3. Uploads content to S3: `s3://pastes/{paste_id}`
4. Stores metadata in PostgreSQL
5. Returns paste URL to client

**Read Flow:**
1. Client requests `GET /abc123`
2. CDN cache check → HIT: return cached content
3. MISS: Rails reads metadata from DB → fetches content from S3 → returns (CDN caches it)

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
                = 80 × 0.75 + 30 × 0.25 = 67.5

If limit = 100: 67.5 < 100 → ALLOW
```

| Pros | Cons |
|------|------|
| Memory efficient (2 counters per user per window) | Approximate (not exact) |
| Fast (O(1) per request) | Slight inaccuracy at window boundaries |

---

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ──→ | Puma (Rails)     | ──→ | Rack::Attack     |
  +--------+     | + API Gateway    |     | Middleware        |
                 +------------------+     +--------+---------+
                                                   |
                                          +--------v---------+
                                          | Redis Cluster    |
                                          | (rate limit      |
                                          |  counters)       |
                                          +------------------+
```

> **Ruby Context:** **Rack::Attack** is the standard rate limiting solution for Rails. For custom algorithms, you can implement them directly with Redis Lua scripts.

```ruby
# Option 1: Rack::Attack (simplest, production-ready)
# config/initializers/rack_attack.rb
class Rack::Attack
  # Token bucket style — 100 requests per minute per user
  throttle('api/per-user', limit: 100, period: 60) do |req|
    if req.path.start_with?('/api/')
      req.env['HTTP_X_API_KEY'] || req.ip
    end
  end

  # Different limits per endpoint
  throttle('api/create-url', limit: 10, period: 60) do |req|
    req.ip if req.path == '/api/v1/urls' && req.post?
  end

  # Custom response
  self.throttled_responder = ->(req) {
    match_data = req.env['rack.attack.match_data']
    retry_after = match_data[:period] - (Time.now.to_i % match_data[:period])
    [
      429,
      {
        'Content-Type' => 'application/json',
        'Retry-After' => retry_after.to_s,
        'X-RateLimit-Limit' => match_data[:limit].to_s,
        'X-RateLimit-Remaining' => '0',
        'X-RateLimit-Reset' => (Time.now.to_i + retry_after).to_s
      },
      [{ error: 'Rate limit exceeded', retry_after: retry_after }.to_json]
    ]
  }
end

# Option 2: Custom Token Bucket with Redis Lua script
class TokenBucketRateLimiter
  LUA_SCRIPT = <<~LUA
    local key = KEYS[1]
    local now = tonumber(ARGV[1])
    local rate = tonumber(ARGV[2])
    local capacity = tonumber(ARGV[3])

    local bucket = redis.call("HMGET", key, "tokens", "last_refill")
    local tokens = tonumber(bucket[1]) or capacity
    local last_refill = tonumber(bucket[2]) or now

    -- Refill tokens based on elapsed time
    local elapsed = now - last_refill
    local new_tokens = math.min(capacity, tokens + elapsed * rate / 60)

    if new_tokens >= 1 then
      redis.call("HMSET", key, "tokens", new_tokens - 1, "last_refill", now)
      redis.call("EXPIRE", key, 120)
      return 1  -- ALLOWED
    else
      redis.call("HMSET", key, "tokens", new_tokens, "last_refill", now)
      redis.call("EXPIRE", key, 120)
      return 0  -- REJECTED
    end
  LUA

  def initialize(redis: Redis.current, rate: 100, capacity: 100)
    @redis = redis
    @rate = rate
    @capacity = capacity
    @sha = @redis.script(:load, LUA_SCRIPT)
  end

  def allow?(identifier)
    result = @redis.evalsha(@sha, keys: ["rate_limit:#{identifier}"],
                                  argv: [Time.now.to_f, @rate, @capacity])
    result == 1
  end
end

# Usage in a Rails before_action
class Api::BaseController < ApplicationController
  before_action :check_rate_limit

  private

  def check_rate_limit
    limiter = TokenBucketRateLimiter.new(rate: 100, capacity: 100)
    identifier = request.headers['X-API-Key'] || request.remote_ip

    unless limiter.allow?(identifier)
      render json: { error: 'Rate limit exceeded' }, status: :too_many_requests
    end
  end
end
```

**Why Redis?**
- Atomic operations (Lua scripts execute atomically — no race conditions)
- Sub-millisecond latency
- Shared across all Puma instances (distributed rate limiting)
- TTL for automatic cleanup of inactive users

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

> **Ruby Context:** See Module 27 for a Ruby `ConsistentHashRing` implementation. For a production key-value store, you'd use Cassandra (`cassandra-driver` gem) or DynamoDB (`aws-sdk-dynamodb`) rather than building your own. But understanding the internals helps you configure and debug these systems.

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
Each node maintains a Merkle tree (hash tree) of its data. Nodes periodically compare Merkle trees — if a subtree hash differs, only that subtree's data needs to be synchronized.

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
```

| Pros | Cons |
|------|------|
| No coordination needed | 128 bits — too large for some use cases |
| Virtually zero collision probability | Not sortable by time (random) |
| Simple to implement | Poor index performance (random → B-Tree page splits) |

> **Ruby Context:** `SecureRandom.uuid` generates UUID v4. For UUID v7 (time-ordered), use the `uuid7` gem.

```ruby
# UUID v4 (built-in Ruby)
SecureRandom.uuid  # => "550e8400-e29b-41d4-a716-446655440000"

# UUID v7 (time-ordered, uuid7 gem)
require 'uuid7'
UUID7.generate  # => "018e4f5c-1a2b-7000-8000-abc123def456" (time-sortable)
```

---

**2. Database Auto-Increment:**

```
INSERT INTO ids DEFAULT VALUES RETURNING id;
→ id = 1, 2, 3, 4, 5, ...
```

| Pros | Cons |
|------|------|
| Simple | Single point of failure |
| Sequential (great for B-Tree indexes) | Doesn't scale horizontally |
| Small (64-bit integer) | Predictable (security concern) |

> **Ruby Context:** ActiveRecord auto-increment is the default for Rails models. Works well for single-database applications.

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

  Timestamp (41 bits): milliseconds since custom epoch → ~69 years
  Machine ID (10 bits): 1,024 unique machines
  Sequence Number (12 bits): 4,096 IDs per millisecond per machine
  Total capacity: ~4 billion IDs per second (across all machines)
```

> **Ruby Context:** Here's a Ruby Snowflake ID generator.

```ruby
# lib/snowflake_id_generator.rb
class SnowflakeIdGenerator
  EPOCH = 1_700_000_000_000  # Custom epoch (2023-11-14, in milliseconds)
  MACHINE_ID_BITS = 10
  SEQUENCE_BITS = 12

  MAX_MACHINE_ID = (1 << MACHINE_ID_BITS) - 1  # 1023
  MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1        # 4095

  MACHINE_ID_SHIFT = SEQUENCE_BITS               # 12
  TIMESTAMP_SHIFT = SEQUENCE_BITS + MACHINE_ID_BITS  # 22

  def initialize(machine_id)
    raise ArgumentError, "Machine ID must be 0-#{MAX_MACHINE_ID}" unless (0..MAX_MACHINE_ID).include?(machine_id)
    @machine_id = machine_id
    @sequence = 0
    @last_timestamp = -1
    @mutex = Mutex.new
  end

  def generate
    @mutex.synchronize do
      timestamp = current_time_ms

      if timestamp == @last_timestamp
        @sequence = (@sequence + 1) & MAX_SEQUENCE
        # Sequence exhausted for this millisecond — wait for next ms
        timestamp = wait_next_ms(@last_timestamp) if @sequence == 0
      elsif timestamp < @last_timestamp
        # Clock moved backward — wait until it catches up
        timestamp = wait_next_ms(@last_timestamp)
        @sequence = 0
      else
        @sequence = 0
      end

      @last_timestamp = timestamp

      ((timestamp - EPOCH) << TIMESTAMP_SHIFT) |
        (@machine_id << MACHINE_ID_SHIFT) |
        @sequence
    end
  end

  private

  def current_time_ms
    (Process.clock_gettime(Process::CLOCK_REALTIME, :millisecond))
  end

  def wait_next_ms(last_ts)
    ts = current_time_ms
    ts = current_time_ms while ts <= last_ts
    ts
  end
end

# Usage
generator = SnowflakeIdGenerator.new(machine_id: 1)
id = generator.generate  # => 7189283746283520001 (64-bit integer)

# In a Rails initializer
# config/initializers/snowflake.rb
SNOWFLAKE = SnowflakeIdGenerator.new(
  machine_id: ENV.fetch('MACHINE_ID', 1).to_i
)

# In a model
class Order < ApplicationRecord
  before_create :assign_snowflake_id

  private

  def assign_snowflake_id
    self.id = SNOWFLAKE.generate
  end
end
```

| Pros | Cons |
|------|------|
| 64-bit integer (compact, fits in BIGINT) | Requires machine ID assignment |
| Time-sortable (newer IDs > older IDs) | Clock skew can cause issues |
| No coordination for ID generation | 69-year limit (from custom epoch) |
| 4K IDs/ms per machine (very high throughput) | Machine ID must be unique across cluster |

---

**4. ULID (Universally Unique Lexicographically Sortable Identifier):**

```
ULID: 01ARZ3NDEKTSV4RRFFQ69G5FAV
  128 bits total:
    Timestamp: 48 bits (milliseconds, ~8,900 years)
    Randomness: 80 bits
  Encoded as 26 Crockford Base32 characters
  Lexicographically sortable
```

> **Ruby Context:** The `ulid` gem provides ULID generation for Ruby.

```ruby
# Gemfile: gem 'ulid'
require 'ulid'

ULID.generate  # => "01ARZ3NDEKTSV4RRFFQ69G5FAV"

# Time-sortable: later ULIDs are lexicographically greater
id1 = ULID.generate
sleep 0.001
id2 = ULID.generate
id2 > id1  # => true (sortable by time)

# Use as primary key in Rails
class Order < ApplicationRecord
  before_create { self.id = ULID.generate }
end
```

---

### Comparison Summary

| Approach | Size | Sortable | Coordination | Throughput | Best For |
|----------|------|----------|-------------|-----------|----------|
| **UUID v4** | 128 bits | No | None | Unlimited | Simple, no ordering needed |
| **UUID v7** | 128 bits | Yes | None | Unlimited | Databases, time-ordered UUIDs |
| **Auto-increment** | 64 bits | Yes | Database (SPOF) | Limited by DB | Small scale, single DB |
| **Snowflake** | 64 bits | Yes | Machine ID only | 4K/ms/machine | Large scale, compact IDs |
| **ULID** | 128 bits | Yes | None | Unlimited | String-sortable, no coordination |

**Recommendation:** Use **Snowflake IDs** for most large-scale systems (compact, sortable, high throughput). Use **UUID v7** or **ULID** if you don't want to manage machine IDs. Use **auto-increment** for simple, single-database Rails apps.

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
    Rails App 1 calls      Rails App 2 calls      Sidekiq calls
    for new IDs            for new IDs            for new IDs
```

> **Ruby Context:** For most Rails applications, auto-increment (default) or ULID is sufficient. Snowflake IDs are needed when you have multiple databases/shards and need globally unique, sortable, compact IDs. The `snowflake-rb` gem or the custom implementation above works well.

---

## Summary & Key Takeaways

| Problem | Key Design Decisions |
|---------|---------------------|
| **URL Shortener** | Base62 encoding of unique ID; 301 vs 302 redirect; cache hot URLs in Redis; DynamoDB or PostgreSQL for storage |
| **Pastebin** | Store content in S3 (not database); metadata in PostgreSQL; CDN for popular pastes; lifecycle policies for expiration |
| **Rate Limiter** | Token Bucket or Sliding Window Counter; Redis for distributed state; Lua scripts for atomicity; Rack::Attack for Rails |
| **Key-Value Store** | Consistent hashing for partitioning; quorum reads/writes (W+R>N); gossip for membership; LSM-tree storage; Merkle trees for anti-entropy |
| **Unique ID Generator** | Snowflake ID (64-bit: timestamp + machine ID + sequence); ULID for no-coordination alternative; auto-increment for simple Rails apps |

**Ruby Implementation Summary:**

| Problem | Ruby Tools |
|---------|-----------|
| **URL Shortener** | Rails API, `SecureRandom`, Redis (`rails cache`), ActiveRecord or `aws-sdk-dynamodb` |
| **Pastebin** | Rails API, `aws-sdk-s3` (content storage), ActiveRecord (metadata), Sidekiq (cleanup jobs) |
| **Rate Limiter** | Rack::Attack (simple), custom Redis Lua scripts (advanced), `redis-rb` |
| **Key-Value Store** | `cassandra-driver`, `aws-sdk-dynamodb`, or custom with `redis-rb` |
| **Unique ID Generator** | `SecureRandom.uuid` (UUID), `ulid` gem (ULID), custom Snowflake class |

---

## Interview Tips for Module 33

1. **Start with requirements** — clarify functional and non-functional requirements before designing
2. **Do the math** — estimate QPS, storage, bandwidth before choosing architecture; use Ruby-specific numbers (Puma: 200-1,000 req/sec)
3. **URL Shortener** — know Base62 encoding, 301 vs 302 trade-off, caching strategy; show Rails controller + service code
4. **Pastebin** — separate metadata (PostgreSQL) from content (S3); mention CDN for popular pastes; show ActiveStorage or `aws-sdk-s3`
5. **Rate Limiter** — explain Token Bucket algorithm; show Rack::Attack for Rails; show Redis Lua script for custom implementation
6. **Key-Value Store** — draw the consistent hash ring; explain quorum (W+R>N); explain write path (WAL → memtable → SSTable)
7. **ID Generator** — explain Snowflake ID bit layout; show Ruby implementation; compare with UUID and ULID; mention `ulid` gem
8. **Always mention** — caching (Redis), monitoring (Yabeda/Prometheus), scaling (Puma instances), failure handling (Sidekiq retries)
9. **Draw diagrams** — architecture diagrams showing Rails API, Sidekiq, Redis, PostgreSQL, S3
10. **Discuss trade-offs** — every decision has trade-offs; articulate them clearly