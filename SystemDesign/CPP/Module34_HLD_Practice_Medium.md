# Module 34: HLD Practice Problems (Medium)

> These medium-difficulty problems are the most commonly asked in system design interviews at top tech companies. Each problem requires combining multiple concepts — databases, caching, messaging, scaling, and trade-offs. For each problem, we cover: requirements, estimation, architecture, key components deep dive, and critical trade-offs.

---

## 34.1 Design Twitter / News Feed

> **Problem:** Design a social media platform where users can post tweets, follow other users, and view a personalized timeline (news feed) of tweets from people they follow.

---

### Requirements & Scale

**Functional:** Post tweets (280 chars + media), follow/unfollow users, view home timeline (feed of followed users' tweets), search tweets, like/retweet.

**Scale:** 500M users, 300M DAU, 600K tweets/sec read, 6K tweets/sec write.

### Estimation

```
Write: 300M DAU × 2 tweets/day / 86,400 ≈ 7,000 tweets/sec
Read (timeline): 300M DAU × 10 timeline views/day / 86,400 ≈ 35,000 reads/sec
Storage: 7K tweets/sec × 86,400 × 365 × 500 bytes ≈ 110 TB/year (text only)
```

### Architecture

```
  +--------+     +-----------+     +----------------+     +-------------+
  | Client | ──→ | API GW    | ──→ | Tweet Service  | ──→ | Tweet DB    |
  +--------+     +-----------+     +----------------+     | (sharded by |
                                          |               | tweet_id)   |
                                   +------v-------+       +-------------+
                                   | Fan-out      |
                                   | Service      |       +-------------+
                                   +------+-------+       | Timeline    |
                                          |               | Cache       |
                                   +------v-------+       | (Redis —    |
                                   | Message Queue|       |  per-user   |
                                   | (Kafka)      |       |  sorted set)|
                                   +--------------+       +-------------+
```

### Key Design Decision: Fan-out on Write vs Fan-out on Read

**Fan-out on Write (Push Model):**
When a user posts a tweet, push it to all followers' timeline caches immediately.

```
User A posts tweet → Fan-out Service:
  Get A's followers: [B, C, D, ..., 10,000 followers]
  For each follower: ZADD timeline:{follower_id} {timestamp} {tweet_id}
  
  User B opens timeline → read from Redis: ZREVRANGE timeline:B 0 49
  → Instant! Timeline is pre-computed.
```

**Fan-out on Read (Pull Model):**
When a user opens their timeline, fetch tweets from all followed users on the fly.

```
User B opens timeline:
  Get B's following list: [A, C, D, ..., 500 users]
  For each followed user: get latest tweets
  Merge and sort by timestamp
  Return top 50
  
  → Slow! Must query 500 users' tweets and merge in real-time.
```

**Hybrid Approach (Twitter's actual approach):**

| User Type | Strategy | Why |
|-----------|----------|-----|
| Regular users (< 10K followers) | Fan-out on write | Pre-compute timelines; fast reads |
| Celebrities (> 10K followers) | Fan-out on read | Pushing to millions of followers is too slow/expensive |

```
User B opens timeline:
  1. Read pre-computed timeline from Redis (fan-out on write entries)
  2. Fetch latest tweets from celebrities B follows (fan-out on read)
  3. Merge both sets, sort by time, return top 50
```

### Database Design

```
Tweets Table (sharded by tweet_id):
  tweet_id (PK), user_id, content, media_urls, created_at, like_count, retweet_count

Users Table (sharded by user_id):
  user_id (PK), username, name, bio, follower_count, following_count

Follows Table (sharded by follower_id):
  follower_id, followee_id, created_at
  Index on (followee_id) for "get all followers of X"

Timeline Cache (Redis sorted sets):
  Key: timeline:{user_id}
  Members: tweet_ids, scored by timestamp
  Keep only the latest 800 tweets per user
```

---


## 34.2 Design Instagram / Photo Sharing

> **Problem:** Design a photo-sharing platform where users can upload photos, view a feed of photos from followed users, like/comment, and search.

---

### Requirements & Scale

**Functional:** Upload photos, view feed, like/comment, follow users, search by hashtag/location, stories (24h expiry).

**Scale:** 1B users, 500M DAU, 100M photos uploaded/day, 2B likes/day.

### Estimation

```
Upload: 100M photos/day / 86,400 ≈ 1,160 uploads/sec
Storage: 100M × 2 MB (average photo) = 200 TB/day = 73 PB/year
Read (feed): 500M DAU × 20 feed views / 86,400 ≈ 116,000 reads/sec
```

### Architecture

```
  +--------+     +-----------+     +------------------+     +----------+
  | Client | ──→ | API GW    | ──→ | Upload Service   | ──→ | S3       |
  +--------+     +-----------+     | (pre-signed URL) |     | (photos) |
                      |            +------------------+     +----------+
                      |                    |
                      |            +-------v--------+       +----------+
                      |            | Image Processing| ──→  | S3       |
                      |            | (resize, filter)|      | (thumbs) |
                      |            +----------------+       +----------+
                      |
                      +──→ +----------------+     +------------------+
                           | Feed Service   | ──→ | Feed Cache       |
                           +----------------+     | (Redis)          |
                                                  +------------------+
                      +──→ +----------------+
                           | Search Service | ──→ Elasticsearch
                           +----------------+
```

### Key Design Decisions

**Photo Upload Flow:**
```
1. Client → Upload Service: "I want to upload a photo"
2. Upload Service → generates pre-signed S3 URL
3. Upload Service → Client: pre-signed URL
4. Client → S3: upload photo directly (bypasses our servers!)
5. S3 → triggers Lambda/event → Image Processing Service
6. Image Processing: generate thumbnails (150×150, 300×300, 600×600, 1080×1080)
7. Store thumbnails in S3, metadata in DB
8. Publish "PhotoUploaded" event → Feed Service updates followers' feeds
```

**Why Pre-Signed URLs?**
- Photos are large (2-10 MB) — don't route through application servers
- S3 handles the upload bandwidth (scales infinitely)
- Application servers stay lightweight (handle only metadata)

**Image Processing Pipeline:**

```
Original Photo (4032×3024, 5 MB)
  ↓
  ├── Thumbnail: 150×150 (10 KB) — feed grid
  ├── Small: 320×320 (30 KB) — mobile feed
  ├── Medium: 640×640 (80 KB) — tablet feed
  ├── Large: 1080×1080 (200 KB) — desktop feed
  └── Original: stored as-is (backup)

Processing: async via message queue (SQS/Kafka)
Storage: S3 with CloudFront CDN in front
```

**Feed Generation:** Same hybrid fan-out approach as Twitter (Section 34.1).

**Storage Strategy:**
- **Photos:** S3 (object storage) — 11 nines durability, cheap, CDN-friendly
- **Metadata:** PostgreSQL/DynamoDB — photo_id, user_id, caption, location, hashtags, created_at
- **Feed cache:** Redis sorted sets — pre-computed per-user feeds
- **Search index:** Elasticsearch — search by hashtag, location, caption

---

## 34.3 Design WhatsApp / Chat System

> **Problem:** Design a real-time messaging system supporting 1-on-1 chat, group chat, online/offline status, message delivery receipts, and end-to-end encryption.

---

### Requirements & Scale

**Functional:** 1-on-1 messaging, group chat (up to 256 members), online/offline presence, read receipts (sent/delivered/read), media sharing, end-to-end encryption.

**Scale:** 2B users, 500M DAU, 100B messages/day, 50M concurrent connections.

### Estimation

```
Messages: 100B/day / 86,400 ≈ 1.16M messages/sec
Connections: 50M concurrent WebSocket connections
Storage: 100B × 100 bytes (avg text) = 10 TB/day text
         100B × 5% media × 200 KB = 1 PB/day media
```

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ←─→ | WebSocket Gateway| ←─→ | Chat Service     |
  | (app)  | WS  | (connection mgmt)|     | (message routing)|
  +--------+     +--------+---------+     +--------+---------+
                          |                        |
                 +--------v---------+     +--------v---------+
                 | Connection       |     | Message Queue    |
                 | Registry         |     | (Kafka)          |
                 | (Redis — which   |     +--------+---------+
                 |  user is on      |              |
                 |  which server)   |     +--------v---------+
                 +------------------+     | Message Store    |
                                          | (Cassandra)      |
                 +------------------+     +------------------+
                 | Presence Service |
                 | (online/offline) |     +------------------+
                 +------------------+     | Media Service    |
                                          | (S3 + CDN)       |
                                          +------------------+
```

### Key Design Decisions

**Message Delivery Flow (1-on-1):**

```
Alice sends message to Bob:

Case 1: Bob is ONLINE (connected to WebSocket Gateway)
  1. Alice → WS Gateway A → Chat Service
  2. Chat Service → Connection Registry: "Where is Bob?"
     → Redis: "Bob is on WS Gateway B"
  3. Chat Service → WS Gateway B → Bob (real-time delivery!)
  4. Chat Service → Kafka → Message Store (persist for history)
  5. Bob's client → ACK → Chat Service → Alice gets "delivered" ✓✓

Case 2: Bob is OFFLINE
  1. Alice → WS Gateway A → Chat Service
  2. Chat Service → Connection Registry: "Bob is offline"
  3. Chat Service → Kafka → Message Store (persist)
  4. Chat Service → Push Notification Service → sends push to Bob's phone
  5. Bob comes online → Chat Service → fetch undelivered messages from Message Store
  6. Bob receives messages → ACK → Alice gets "delivered" ✓✓
```

**Group Chat:**

```
Group: "Family" (members: Alice, Bob, Charlie, Dave)
Alice sends message to group:

  1. Alice → Chat Service
  2. Chat Service → get group members: [Bob, Charlie, Dave]
  3. For each member:
     → Check Connection Registry (online/offline)
     → Online: deliver via WebSocket
     → Offline: store + push notification
  4. Persist message: Cassandra (partition key: group_id, clustering key: timestamp)
```

**Message Storage (Cassandra):**

```sql
CREATE TABLE messages (
    conversation_id UUID,      -- partition key (1-on-1 or group)
    message_id TIMEUUID,       -- clustering key (sorted by time)
    sender_id UUID,
    content TEXT,               -- encrypted
    media_url TEXT,
    status TEXT,                -- sent, delivered, read
    created_at TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Efficient query: get latest 50 messages for a conversation
SELECT * FROM messages WHERE conversation_id = ? LIMIT 50;
```

**End-to-End Encryption (E2EE):**

```
Signal Protocol (used by WhatsApp):
  1. Each user generates a key pair (public + private)
  2. Public keys are exchanged via the server
  3. Messages are encrypted with the recipient's public key
  4. Only the recipient's private key can decrypt
  5. Server NEVER sees plaintext — only encrypted blobs
  
  Group E2EE: Sender encrypts the message separately for each group member
  (using each member's public key)
```

---

## 34.4 Design Notification System

> **Problem:** Design a system that sends notifications to users across multiple channels (push, SMS, email) with priority, rate limiting, and templates.

---

### Requirements & Scale

**Functional:** Send push notifications, SMS, email. Support templates. Priority levels (urgent, high, normal, low). Rate limiting per user. Scheduling (send at specific time). Analytics (delivery rate, open rate).

**Scale:** 10B notifications/day, 100M users, 3 channels.

### Architecture

```
  +------------------+     +------------------+     +------------------+
  | Trigger Sources  | ──→ | Notification     | ──→ | Priority Queue   |
  | - API calls      |     | Service          |     | (Kafka topics    |
  | - Events (Kafka) |     | - Validate       |     |  per priority)   |
  | - Scheduled jobs |     | - Rate limit     |     +--------+---------+
  +------------------+     | - Template render|              |
                           | - Dedup          |     +--------v---------+
                           +------------------+     | Channel Workers  |
                                                    | ┌──────────────┐ |
                                                    | │ Push Worker  │ |
                                                    | │ (APNs, FCM)  │ |
                                                    | ├──────────────┤ |
                                                    | │ SMS Worker   │ |
                                                    | │ (Twilio)     │ |
                                                    | ├──────────────┤ |
                                                    | │ Email Worker │ |
                                                    | │ (SES/SendGrid)│|
                                                    | └──────────────┘ |
                                                    +------------------+
                                                             |
                                                    +--------v---------+
                                                    | Analytics        |
                                                    | (delivery, open, |
                                                    |  click tracking) |
                                                    +------------------+
```

**Key Design Decisions:**

| Decision | Choice | Why |
|----------|--------|-----|
| **Queue per priority** | Separate Kafka topics: urgent, high, normal, low | Urgent notifications processed first; low-priority doesn't block urgent |
| **Rate limiting** | Per-user limits (max 5 push/hour, 2 SMS/day) | Prevent notification fatigue; respect user preferences |
| **Deduplication** | Idempotency key per notification | Prevent sending the same notification twice (retry safety) |
| **Template engine** | Server-side rendering with variables | "Hi {{name}}, your order {{order_id}} has shipped" |
| **Retry with backoff** | Exponential backoff per channel | APNs/FCM/Twilio may be temporarily unavailable |
| **DLQ (Dead Letter Queue)** | Failed notifications after max retries → DLQ | Investigate and retry manually; don't lose notifications |
| **User preferences** | Stored in DB; checked before sending | User opted out of marketing emails? Don't send. |

---

## 34.5 Design a Web Crawler

> **Problem:** Design a web crawler that systematically browses the internet, downloading web pages for indexing by a search engine.

---

### Requirements & Scale

**Functional:** Crawl web pages starting from seed URLs, extract links, follow links, store page content, respect robots.txt, handle duplicates.

**Scale:** 1 billion pages/month, ~400 pages/second.

### Architecture

```
  +------------------+     +------------------+     +------------------+
  | URL Frontier     | ──→ | Fetcher          | ──→ | Parser           |
  | (priority queue  |     | (HTTP client,    |     | (extract links,  |
  |  of URLs to      |     |  respect robots, |     |  text, metadata) |
  |  crawl)          |     |  politeness)     |     +--------+---------+
  +--------+---------+     +------------------+              |
           ↑                                        +--------v---------+
           |                                        | Dedup Service    |
           +────────────────────────────────────────| (Bloom filter /  |
              new URLs (not yet crawled)             |  URL fingerprint)|
                                                    +--------+---------+
                                                             |
                                                    +--------v---------+
                                                    | Content Store    |
                                                    | (S3 / HDFS)     |
                                                    +------------------+
```

**Key Design Decisions:**

**URL Frontier (Priority Queue):**
```
Front queues (priority-based):
  High priority: important domains (news sites, popular sites)
  Normal priority: regular pages
  Low priority: deep pages, less important domains

Back queues (politeness-based):
  One queue per domain: ensures we don't overwhelm any single server
  Delay between requests to same domain: 1-10 seconds (respect robots.txt crawl-delay)
```

**Politeness:**
- Parse `robots.txt` for each domain (which paths are allowed/disallowed)
- Respect `Crawl-delay` directive
- One queue per domain — never send more than 1 request/second to the same domain
- Identify as a crawler in User-Agent header

**Deduplication:**
- **URL dedup:** Bloom filter (have we seen this URL before?) — prevents re-crawling
- **Content dedup:** SimHash or MinHash fingerprint of page content — detects near-duplicate pages (same content, different URLs)

**Handling Scale (1B pages/month):**
```
400 pages/sec × average page size 500 KB = 200 MB/sec = 17 TB/day
Storage: 17 TB/day × 30 = 510 TB/month (compressed: ~100 TB)

Fetcher instances: 400 pages/sec ÷ 2 pages/sec per thread × 100 threads/instance ≈ 2 instances
(But with politeness delays, need more: ~20 fetcher instances)
```

---

## 34.6 Design Search Autocomplete / Typeahead

> **Problem:** Design a system that suggests search queries as the user types, showing the top 5-10 most popular completions for the typed prefix.

---

### Requirements & Scale

**Functional:** Given a prefix (e.g., "sys"), return top 10 suggestions (e.g., "system design", "system of a down", "system32"). Update suggestions based on search frequency. Personalization (optional).

**Scale:** 10K QPS, 5B search queries/day for data collection, < 100ms latency.

### Architecture

```
  +--------+     +-----------+     +------------------+     +----------+
  | Client | ──→ | API GW    | ──→ | Suggestion       | ──→ | Trie     |
  | (types |     +-----------+     | Service          |     | Cache    |
  |  "sys")|                       +------------------+     | (Redis)  |
  +--------+                              |                 +----------+
                                   +------v-------+
                                   | Data Pipeline|         +----------+
                                   | (aggregate   | ──→     | Trie DB  |
                                   |  search logs)|         | (rebuilt |
                                   +--------------+         |  hourly) |
                                                            +----------+
```

**Key Design: Trie Data Structure**

```
Trie for autocomplete:

  Root
  ├── s
  │   ├── y
  │   │   ├── s → "sys" (prefix)
  │   │   │   ├── t
  │   │   │   │   ├── e
  │   │   │   │   │   ├── m → "system" (freq: 50,000)
  │   │   │   │   │   │   ├── " design" → "system design" (freq: 30,000)
  │   │   │   │   │   │   ├── " of" → "system of a down" (freq: 10,000)
  │   │   │   │   │   │   └── "32" → "system32" (freq: 5,000)

Each node stores:
  - Character
  - Top K suggestions for this prefix (pre-computed!)
  - Frequency counts
```

**Pre-computing Top-K at Each Node:**
Instead of traversing the entire subtree at query time, store the top 10 suggestions at every node. When the user types "sys", return the pre-computed list immediately — O(1) lookup.

**Data Pipeline (Offline):**
```
1. Collect search queries (Kafka → S3)
2. Aggregate: count frequency per query (Spark job, hourly)
3. Build/update the Trie with new frequencies
4. Serialize Trie → distribute to Suggestion Service instances
5. Cache hot prefixes in Redis (top 1000 prefixes)
```

**Optimization:**
- **Cache top prefixes:** The top 1000 prefixes (1-3 characters) cover 80%+ of queries → cache in Redis
- **Don't query on every keystroke:** Debounce — wait 100-200ms after the user stops typing
- **Client-side caching:** Cache recent suggestions in the browser/app

---


## 34.7 Design a Distributed Cache

> **Problem:** Design a distributed caching system (like Redis Cluster or Memcached) that provides sub-millisecond reads across millions of operations per second.

---

### Requirements & Scale

**Functional:** GET/SET/DELETE by key, TTL expiration, eviction (LRU), support for data structures (strings, lists, sets, sorted sets).

**Scale:** 10M ops/sec, 1 TB total cache capacity, < 1ms P99 latency, 99.99% availability.

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ──→ | Client Library   | ──→ | Cache Node 1     |
  +--------+     | (consistent hash,|     | (slots 0-5460)   |
                 |  connection pool)|     +------------------+
                 +------------------+ ──→ | Cache Node 2     |
                                          | (slots 5461-10922)|
                                    + ──→ +------------------+
                                          | Cache Node 3     |
                                          | (slots 10923-16383)|
                                          +------------------+
                                          
  Each node has a replica for HA:
    Node 1 → Replica 1a
    Node 2 → Replica 2a
    Node 3 → Replica 3a
```

**Key Design Decisions:**

| Component | Design | Why |
|-----------|--------|-----|
| **Partitioning** | Consistent hashing (16,384 slots) | Even distribution; minimal redistribution on node add/remove |
| **Replication** | Async primary → replica | HA without write latency penalty |
| **Eviction** | LRU (approximated — sample 5 keys, evict the least recently used) | O(1) approximate LRU; Redis's actual approach |
| **Persistence** | Optional RDB snapshots + AOF | Survive restarts; not required for pure cache |
| **Failover** | Replica promotion on primary failure (Sentinel/Cluster) | Automatic HA |
| **Client routing** | Smart client caches slot→node mapping | Avoid extra hop through proxy |

**Handling Hot Keys:**
```
Problem: One key gets 1M reads/sec → single node overwhelmed

Solutions:
  1. Local cache (L1): Cache hot keys in application memory (Caffeine)
  2. Key replication: Replicate hot key to all nodes; client reads from random node
  3. Key splitting: "hot_key:shard:0" through "hot_key:shard:9" — 10 copies on different nodes
```

---

## 34.8 Design an API Rate Limiter (Distributed)

> **Problem:** Design a distributed rate limiter for a large-scale API platform with 100K+ clients, supporting per-user, per-endpoint, and tiered rate limits.

---

### Key Differences from 33.3 (Basic Rate Limiter)

This is the **distributed** version — multiple API gateway instances must share rate limit state consistently.

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ──→ | API Gateway 1    | ──→ | Redis Cluster    |
  +--------+     | (rate limit      |     | (shared counters)|
                 |  middleware)     |     +------------------+
  +--------+     +------------------+
  | Client | ──→ | API Gateway 2    | ──→ (same Redis Cluster)
  +--------+     +------------------+
  
                 +------------------+
                 | Rules Service    |  (rate limit configurations)
                 | - Free: 100/min  |
                 | - Pro: 1000/min  |
                 | - Enterprise:    |
                 |   10000/min      |
                 +------------------+
```

**Distributed Challenges:**

| Challenge | Solution |
|-----------|---------|
| **Race conditions** | Redis Lua scripts (atomic increment + check) |
| **Clock skew** | Use Redis server time (EVAL with redis.call('TIME')), not client time |
| **Redis failure** | Fail open (allow requests) or fail closed (reject) — configurable per endpoint |
| **Sync across regions** | Local Redis per region (approximate); or global Redis with higher latency |
| **Rule updates** | Rules Service pushes updates; gateways cache rules with short TTL |

**Sliding Window Counter (Redis implementation):**

```lua
-- Atomic sliding window counter in Redis
local key = KEYS[1]                    -- "rate:user_123:endpoint:/api/users"
local window = tonumber(ARGV[1])       -- 60 (seconds)
local limit = tonumber(ARGV[2])        -- 100 (requests per window)
local now = tonumber(ARGV[3])          -- current timestamp

-- Remove old entries outside the window
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- Count current entries
local count = redis.call('ZCARD', key)

if count < limit then
    -- Allow: add this request
    redis.call('ZADD', key, now, now .. ':' .. math.random(1000000))
    redis.call('EXPIRE', key, window)
    return {1, limit - count - 1}      -- {allowed, remaining}
else
    -- Reject
    return {0, 0}                       -- {rejected, remaining=0}
end
```

---

## 34.9 Design Uber / Ride Sharing

> **Problem:** Design a ride-sharing platform where riders request rides and nearby drivers are matched in real-time.

---

### Requirements & Scale

**Functional:** Rider requests ride (pickup, destination), match with nearby driver, real-time tracking, ETA calculation, fare estimation, payment, ride history, surge pricing.

**Scale:** 20M rides/day, 5M concurrent drivers, 20M concurrent riders, location updates every 4 seconds.

### Estimation

```
Location updates: 5M drivers × 1 update/4 sec = 1.25M updates/sec
Ride requests: 20M/day / 86,400 ≈ 230 rides/sec (peak: ~700/sec)
```

### Architecture

```
  +--------+     +-----------+     +------------------+
  | Rider  | ──→ | API GW    | ──→ | Ride Service     |
  | App    |     +-----------+     | (request, match, |
  +--------+                       |  track, complete) |
                                   +--------+---------+
  +--------+     +-----------+              |
  | Driver | ──→ | API GW    |     +--------v---------+
  | App    |     +-----------+     | Matching Service  |
  +--------+                       | (find nearby      |
       |                           |  drivers)         |
       |                           +--------+---------+
       |                                    |
       |    +------------------+   +--------v---------+
       +──→ | Location Service | → | Geospatial Index |
            | (track driver    |   | (Redis GEO or    |
            |  positions)      |   |  QuadTree/       |
            +------------------+   |  GeoHash)        |
                                   +------------------+
  
  +------------------+  +------------------+  +------------------+
  | Payment Service  |  | Pricing Service  |  | ETA Service      |
  | (Stripe/internal)|  | (surge pricing)  |  | (maps + traffic) |
  +------------------+  +------------------+  +------------------+
```

### Key Design: Geospatial Matching

**Finding Nearby Drivers:**

```
Rider requests ride at location (37.7749, -122.4194)

Option 1: GeoHash
  - Encode location to GeoHash: "9q8yyk" (precision ~1.2 km)
  - Query: find all drivers in GeoHash "9q8yyk" and adjacent cells
  - Redis: GEOSEARCH drivers FROMLONLAT -122.4194 37.7749 BYRADIUS 5 km

Option 2: QuadTree
  - Divide the map into quadrants recursively
  - Each leaf node contains drivers in that area
  - Search the leaf containing the rider + neighboring leaves

Option 3: S2 Geometry (Google's approach)
  - Divide Earth's surface into cells at multiple levels
  - Efficient range queries for "find all points within X km"
```

**Matching Algorithm:**

```
1. Rider requests ride at location L
2. Find all available drivers within 5 km radius
3. Filter: driver is available, vehicle type matches, driver rating > threshold
4. Rank by: distance (closest first), ETA, driver rating
5. Send ride request to top driver
6. Driver has 15 seconds to accept
   → Accept: match confirmed, start ride
   → Decline/timeout: send to next driver
   → After 3 rejections: expand search radius to 10 km
```

**Driver Location Tracking:**

```
Every 4 seconds, driver app sends:
  { driver_id, lat, lng, heading, speed, timestamp }

Location Service:
  1. Update Redis GEO: GEOADD drivers {lng} {lat} {driver_id}
  2. Publish to Kafka: "driver_location" topic (for ride tracking, ETA updates)
  3. If driver is on a ride: push location to rider's app via WebSocket

Scale: 1.25M updates/sec → Redis handles this easily (single-threaded, in-memory)
```

**Surge Pricing:**

```
Divide city into hexagonal zones (H3 cells)
For each zone, every minute:
  demand = ride_requests_in_zone / time_window
  supply = available_drivers_in_zone
  surge_multiplier = demand / supply (capped at 3x-5x)
  
  If surge > 1.0: show surge pricing to rider before confirming
```

---

## 34.10 Design a Hotel / Flight Booking System

> **Problem:** Design a booking system (like Booking.com or Expedia) where users can search for hotels/flights, view availability, and make reservations without double-booking.

---

### Requirements & Scale

**Functional:** Search hotels by location/date/guests, view availability and pricing, book a room, payment processing, cancellation, confirmation notifications.

**Scale:** 1M bookings/day, 50M searches/day, 1M hotels with 100 rooms each = 100M room-nights to manage.

### Architecture

```
  +--------+     +-----------+     +------------------+     +------------------+
  | Client | ──→ | API GW    | ──→ | Search Service   | ──→ | Elasticsearch    |
  +--------+     +-----------+     | (search hotels,  |     | (hotel search    |
                      |            |  filter, sort)    |     |  index)          |
                      |            +------------------+     +------------------+
                      |
                      +──→ +------------------+     +------------------+
                           | Booking Service  | ──→ | Inventory DB     |
                           | (reserve, book,  |     | (PostgreSQL —    |
                           |  cancel)         |     |  room availability|
                           +--------+---------+     |  with row locking)|
                                    |               +------------------+
                           +--------v---------+
                           | Payment Service  |     +------------------+
                           | (charge, refund) |     | Notification Svc |
                           +------------------+     | (email confirm)  |
                                                    +------------------+
```

### Key Design: Preventing Double Booking

```
Problem: Two users try to book the last room at the same time.

Solution: Pessimistic Locking (SELECT FOR UPDATE)

  BEGIN;
  -- Lock the specific room-night row
  SELECT * FROM room_inventory
  WHERE hotel_id = 123 AND room_type = 'deluxe' AND date = '2025-07-15'
  AND available_count > 0
  FOR UPDATE;  -- exclusive lock!
  
  -- If row found and available_count > 0:
  UPDATE room_inventory
  SET available_count = available_count - 1
  WHERE hotel_id = 123 AND room_type = 'deluxe' AND date = '2025-07-15';
  
  INSERT INTO bookings (user_id, hotel_id, room_type, check_in, check_out, status)
  VALUES (456, 123, 'deluxe', '2025-07-15', '2025-07-17', 'confirmed');
  
  COMMIT;
  
  -- Second user's SELECT FOR UPDATE BLOCKS until first transaction commits
  -- Then sees available_count = 0 → booking fails gracefully
```

**Alternative: Optimistic Locking**

```sql
-- Read current version
SELECT available_count, version FROM room_inventory
WHERE hotel_id = 123 AND room_type = 'deluxe' AND date = '2025-07-15';
-- Returns: available_count=1, version=5

-- Try to update with version check
UPDATE room_inventory
SET available_count = available_count - 1, version = version + 1
WHERE hotel_id = 123 AND room_type = 'deluxe' AND date = '2025-07-15'
AND version = 5 AND available_count > 0;

-- If affected_rows = 1 → success
-- If affected_rows = 0 → conflict (someone else booked) → retry or show "sold out"
```

**Temporary Hold (Reservation Timeout):**

```
1. User selects room → system creates a TEMPORARY HOLD (5-10 minutes)
   UPDATE room_inventory SET held_count = held_count + 1 WHERE ...
   INSERT INTO holds (user_id, hotel_id, room_type, date, expires_at) VALUES (..., NOW() + INTERVAL '10 minutes')

2. User completes payment within 10 minutes → convert hold to booking
   UPDATE room_inventory SET held_count = held_count - 1, booked_count = booked_count + 1

3. Hold expires (user didn't pay) → release the hold
   Background job: DELETE FROM holds WHERE expires_at < NOW()
   UPDATE room_inventory SET held_count = held_count - 1

This prevents the scenario where a user adds to cart but never pays,
blocking the room for other users indefinitely.
```

**Search Optimization:**

```
Search: "Hotels in Paris, July 15-17, 2 guests, under $200/night"

1. Elasticsearch query:
   - Filter: city=Paris, max_guests>=2, price<=200
   - Filter: available on July 15, 16 (check availability index)
   - Sort: by relevance (rating, reviews, price)
   - Return: top 20 results with photos, ratings, prices

2. Availability is pre-computed and indexed in Elasticsearch
   (updated every few minutes from the inventory DB)
   
3. Real-time availability check happens only when user clicks "Book"
   (query the inventory DB with locking at booking time)
```

---

## Summary & Key Takeaways

| Problem | Key Design Decisions |
|---------|---------------------|
| **Twitter/Feed** | Hybrid fan-out (push for regular users, pull for celebrities); Redis sorted sets for timeline cache; shard tweets by tweet_id |
| **Instagram** | Pre-signed URLs for direct S3 upload; async image processing pipeline (thumbnails); CDN for serving images |
| **WhatsApp/Chat** | WebSocket for real-time; Connection Registry (Redis) for routing; Cassandra for message storage (partition by conversation); E2EE with Signal Protocol |
| **Notifications** | Priority queues (Kafka topics per priority); channel workers (push/SMS/email); rate limiting per user; DLQ for failures |
| **Web Crawler** | URL Frontier (priority + politeness queues); Bloom filter for URL dedup; respect robots.txt; SimHash for content dedup |
| **Autocomplete** | Trie with pre-computed top-K at each node; offline data pipeline (Spark); cache hot prefixes in Redis; debounce client requests |
| **Distributed Cache** | Consistent hashing (16K slots); async replication; approximate LRU; hot key mitigation (local cache, key splitting) |
| **Rate Limiter (Distributed)** | Redis Lua scripts for atomicity; sliding window counter; fail open/closed policy; per-region Redis |
| **Uber/Ride Sharing** | Redis GEO for driver locations; geospatial matching (GeoHash/S2); surge pricing per zone; WebSocket for real-time tracking |
| **Hotel Booking** | Pessimistic locking (SELECT FOR UPDATE) to prevent double booking; temporary holds with timeout; Elasticsearch for search; real-time availability check at booking time |

---

## Interview Tips for Module 34

1. **Start with requirements and estimation** — always, for every problem
2. **Twitter** — the fan-out question is the core; explain hybrid approach; draw the timeline cache
3. **Instagram** — pre-signed URLs for upload is the key insight; mention image processing pipeline
4. **WhatsApp** — WebSocket connection management is the core challenge; explain message delivery for online vs offline users
5. **Notifications** — priority queues and channel abstraction; mention rate limiting and user preferences
6. **Web Crawler** — URL Frontier design (priority + politeness) is the core; mention Bloom filter for dedup
7. **Autocomplete** — Trie with pre-computed top-K; mention offline data pipeline and caching
8. **Distributed Cache** — consistent hashing and hot key handling are the key topics
9. **Uber** — geospatial indexing (Redis GEO or GeoHash) is the core; explain the matching algorithm
10. **Hotel Booking** — double booking prevention is the core; explain pessimistic vs optimistic locking; mention temporary holds
