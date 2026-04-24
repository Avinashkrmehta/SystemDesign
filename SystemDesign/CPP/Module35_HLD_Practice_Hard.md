# Module 35: HLD Practice Problems (Hard)

> These are the most challenging system design problems — asked at senior-level interviews at top tech companies. Each problem involves complex trade-offs, multiple interacting subsystems, and requires deep knowledge of distributed systems concepts. Master these and you can handle any system design interview.

---


## 35.1 Design YouTube / Video Streaming

> **Problem:** Design a video-sharing platform where users upload videos, the system transcodes them into multiple formats/resolutions, and viewers stream videos with adaptive bitrate.

---

### Requirements & Scale

**Functional:** Upload videos, transcode to multiple resolutions, stream with adaptive bitrate, search, recommendations, comments, likes, subscriptions, live streaming (optional).

**Scale:** 2B users, 800M DAU, 500 hours of video uploaded per minute, 1B video views per day.

### Estimation

```
Upload: 500 hours/min × 60 = 30,000 hours/hour
  Average raw video: 1 GB/hour → 30 TB/hour upload
  After transcoding (multiple resolutions): ~5x → 150 TB/hour stored

Storage (1 year):
  150 TB/hour × 24 × 365 = 1.3 EB (exabytes)/year

Streaming bandwidth:
  1B views/day, average 5 min watch, 5 Mbps (1080p)
  Peak concurrent: ~50M viewers
  50M × 5 Mbps = 250 Tbps → CDN is absolutely essential
```

### Architecture

```
  Upload Path:
  +--------+     +------------------+     +------------------+     +----------+
  | Client | ──→ | Upload Service   | ──→ | Object Storage   | ──→ | Transcode|
  +--------+     | (pre-signed URL, |     | (S3 — raw video) |     | Pipeline |
                 |  chunked upload) |     +------------------+     | (SQS +   |
                 +------------------+                               | FFmpeg   |
                                                                    | workers) |
                                                                    +----+-----+
                                                                         |
                                                                    +----v-----+
                                                                    | S3       |
                                                                    | (transcoded|
                                                                    |  videos)  |
                                                                    +----+-----+
                                                                         |
  Streaming Path:                                                   +----v-----+
  +--------+     +------------------+     +------------------+     | CDN      |
  | Viewer | ←── | CDN (CloudFront) | ←── | Origin (S3)      |     | (edge    |
  +--------+     +------------------+     +------------------+     |  cache)  |
                                                                    +----------+
```

### Key Design: Transcoding Pipeline

```
Raw Video Upload (1080p, H.264, 1 GB)
  ↓
Message Queue (SQS/Kafka): "transcode job for video_abc123"
  ↓
Transcoding Workers (auto-scaled):
  ├── 240p  (H.264, 400 Kbps)  → 50 MB
  ├── 360p  (H.264, 800 Kbps)  → 100 MB
  ├── 480p  (H.264, 1.5 Mbps)  → 200 MB
  ├── 720p  (H.264, 3 Mbps)    → 400 MB
  ├── 1080p (H.264, 5 Mbps)    → 650 MB
  └── 1080p (VP9/AV1, 3 Mbps)  → 400 MB (better codec, same quality)
  
Each resolution is split into small segments (2-10 seconds each)
for adaptive bitrate streaming (HLS/DASH).

Total storage per video: ~1.8 GB (all resolutions)
```

**Adaptive Bitrate Streaming (ABR):**

```
Client starts watching at 1080p (good network)
  → Requests 1080p segments: segment_001.ts, segment_002.ts, ...

Network degrades (buffering detected):
  → Client switches to 480p mid-stream: segment_015_480p.ts, segment_016_480p.ts
  → No interruption — just lower quality

Network recovers:
  → Client switches back to 1080p

Protocols:
  HLS (HTTP Live Streaming) — Apple, most common
  DASH (Dynamic Adaptive Streaming over HTTP) — open standard
  
Both work by: manifest file (.m3u8 or .mpd) lists all available resolutions
              and segment URLs. Client picks the best resolution dynamically.
```

**Video Recommendation:**
- Collaborative filtering: "users who watched X also watched Y"
- Content-based: similar tags, categories, creators
- ML models trained on watch history, likes, search queries
- Served from a pre-computed recommendation cache (updated hourly)

---

## 35.2 Design Google Docs / Collaborative Editing

> **Problem:** Design a real-time collaborative document editor where multiple users can edit the same document simultaneously, seeing each other's changes in real-time.

---

### Requirements & Scale

**Functional:** Create/edit documents, real-time collaboration (multiple cursors), version history, comments, sharing/permissions, offline editing with sync.

**Scale:** 100M documents, 10M concurrent editors, < 100ms latency for edit propagation.

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ←─→ | WebSocket Server | ←─→ | Collaboration    |
  | (editor|  WS | (per-document    |     | Engine           |
  |  app)  |     |  sessions)       |     | (OT or CRDT)    |
  +--------+     +--------+---------+     +--------+---------+
                          |                        |
                 +--------v---------+     +--------v---------+
                 | Session Manager  |     | Document Store   |
                 | (which users are |     | (PostgreSQL —    |
                 |  editing which   |     |  document meta)  |
                 |  document)       |     +------------------+
                 +------------------+     
                                          +------------------+
                                          | Object Storage   |
                                          | (S3 — document   |
                                          |  content/versions)|
                                          +------------------+
```

### Key Design: Conflict Resolution

**The Problem:**

```
Alice and Bob are editing the same sentence: "Hello World"

Alice (at position 6): inserts "Beautiful " → "Hello Beautiful World"
Bob (at position 5): deletes "o" → "Hell World"

Both edits happen simultaneously. How do we merge them correctly?
Without conflict resolution: one edit overwrites the other → data loss!
```

**Operational Transformation (OT) — Google Docs' approach:**

```
OT transforms operations against each other so they can be applied in any order
and produce the same result.

Alice's operation: INSERT("Beautiful ", position=6)
Bob's operation: DELETE(position=5, count=1)

Server receives Alice's op first, then Bob's op:
  1. Apply Alice's op: "Hello World" → "Hello Beautiful World"
  2. Transform Bob's op against Alice's op:
     Bob wanted to delete at position 5, but Alice inserted 10 chars before position 6
     Transformed: DELETE(position=5, count=1) (position unchanged — deletion is before insertion)
  3. Apply transformed Bob's op: "Hello Beautiful World" → "Hell Beautiful World"

If server received Bob's op first:
  1. Apply Bob's op: "Hello World" → "Hell World"
  2. Transform Alice's op against Bob's op:
     Alice wanted to insert at position 6, but Bob deleted 1 char before position 6
     Transformed: INSERT("Beautiful ", position=5) (position shifted left by 1)
  3. Apply transformed Alice's op: "Hell World" → "Hell Beautiful World"

Both orderings produce the same result: "Hell Beautiful World" ✅
```

**CRDTs (Conflict-free Replicated Data Types) — Alternative approach:**

```
Each character has a unique, ordered ID:
  "Hello World"
  H(1.0) e(2.0) l(3.0) l(4.0) o(5.0) (6.0)W(7.0) o(8.0) r(9.0) l(10.0) d(11.0)

Alice inserts "B" between position 6 and 7: ID = 6.5
Bob deletes character at ID 5.0 ("o")

These operations commute — order doesn't matter:
  Apply both: H e l l _ B e a u t i f u l _ W r l d
  (Same result regardless of order)

CRDTs used by: Figma, Apple Notes, Automerge, Yjs
```

**OT vs CRDT:**

| Feature | OT | CRDT |
|---------|----|----|
| Server requirement | Central server transforms operations | Can work peer-to-peer (no central server) |
| Complexity | Complex transformation functions | Complex data structure |
| Offline support | Limited (need server for transforms) | Excellent (merge when reconnected) |
| Proven at scale | Google Docs (billions of users) | Figma, Apple Notes |
| Consistency | Requires central ordering | Eventually consistent (automatic merge) |

**Version History:**
- Every N operations (or every save), snapshot the document state
- Store snapshots in S3 with timestamps
- Users can browse and restore any previous version
- Diff between versions computed on demand

---


## 35.3 Design Google Maps

> **Problem:** Design a mapping and navigation service that provides map rendering, route planning, real-time traffic, ETA estimation, and location search.

---

### Requirements & Scale

**Functional:** View maps (pan, zoom), search locations/businesses, get directions (driving, walking, transit), real-time traffic, ETA, street view (optional), offline maps.

**Scale:** 1B users, 100M DAU, 1M navigation sessions concurrent, real-time traffic from millions of devices.

### Architecture

```
  +--------+     +-----------+     +------------------+
  | Client | ──→ | API GW    | ──→ | Map Tile Service |  (pre-rendered map images)
  +--------+     +-----------+     +------------------+
                      |
                      +──→ +------------------+
                           | Routing Service  |  (shortest path, ETA)
                           +------------------+
                      +──→ +------------------+
                           | Geocoding Service|  (address ↔ coordinates)
                           +------------------+
                      +──→ +------------------+
                           | Search Service   |  (places, businesses)
                           +------------------+
                      +──→ +------------------+
                           | Traffic Service  |  (real-time traffic data)
                           +------------------+
```

### Key Design: Map Tile Rendering

```
The world map is divided into tiles at multiple zoom levels:

  Zoom 0: 1 tile (entire world)
  Zoom 1: 4 tiles (2×2)
  Zoom 2: 16 tiles (4×4)
  ...
  Zoom 18: 68 billion tiles (262,144 × 262,144) — street-level detail

Each tile: 256×256 pixels PNG/WebP image (~20-50 KB)

Client requests: GET /tiles/{zoom}/{x}/{y}.png
  Example: /tiles/15/5241/12661.png

Tiles are:
  - Pre-rendered and stored in object storage (S3)
  - Served via CDN (CloudFront) — most tiles are static and cacheable
  - Updated periodically (road changes, new buildings) — not real-time
  
Vector tiles (modern approach):
  Instead of raster images, send vector data (roads, buildings, labels)
  Client renders locally → smaller download, smoother zoom, customizable styling
  Used by: Mapbox, Google Maps (mobile)
```

### Key Design: Route Planning

```
Road network as a weighted graph:
  Nodes: intersections
  Edges: road segments (weight = travel time based on distance, speed limit, traffic)

Algorithms:
  1. Dijkstra's: shortest path, O((V+E) log V) — too slow for continent-scale
  2. A*: Dijkstra + heuristic (straight-line distance) — faster but still slow
  3. Contraction Hierarchies (CH): pre-process the graph by "contracting" unimportant nodes
     → Query time: milliseconds for continent-scale routes!
     → Used by: OSRM, Google Maps
  
  Pre-processing:
    - Rank nodes by importance (highways > local roads)
    - Contract low-importance nodes (add shortcut edges)
    - Build a hierarchy: query only searches "upward" then "downward"
    
  Query: San Francisco → New York
    - Search upward from SF (local roads → highways → interstates)
    - Search upward from NY (same)
    - Meet in the middle at the interstate level
    - Total: ~1000 nodes explored (vs millions with Dijkstra)
```

### Key Design: Real-Time Traffic

```
Data Sources:
  - GPS data from millions of phones (anonymized speed on road segments)
  - Traffic sensors on highways
  - Incident reports (accidents, construction)
  - Historical patterns (rush hour, weekends, holidays)

Processing:
  1. Phones send location + speed every 1-5 seconds
  2. Map-match GPS points to road segments (snap to nearest road)
  3. Aggregate speed per road segment (median speed of all devices)
  4. Compare to free-flow speed → calculate congestion level
  5. Update edge weights in the routing graph
  6. Push traffic layer to map tiles (color-coded: green/yellow/red)

ETA Calculation:
  ETA = sum of (segment_length / current_speed) for all segments in route
  + historical adjustment (this road is usually slower at 5 PM)
  + ML model (predict traffic 15-30 minutes ahead)
  
  Re-calculate ETA every 30-60 seconds during navigation
```

---

## 35.4 Design Google Search

> **Problem:** Design a web search engine that crawls the web, indexes pages, and returns relevant results for user queries in milliseconds.

---

### Requirements & Scale

**Functional:** Crawl web pages, build search index, process queries, rank results by relevance, spell correction, autocomplete, knowledge panels, ads.

**Scale:** 8.5B searches/day (~100K QPS), index of 100+ billion web pages, < 500ms query latency.

### Architecture

```
  Offline (Indexing Pipeline):
  +----------+     +----------+     +----------+     +----------+
  | Crawler  | ──→ | Parser   | ──→ | Indexer  | ──→ | Index    |
  | (fetch   |     | (extract |     | (build   |     | Storage  |
  |  pages)  |     |  text,   |     |  inverted|     | (sharded)|
  +----------+     |  links)  |     |  index)  |     +----------+
                   +----------+     +----------+

  Online (Query Processing):
  +--------+     +-----------+     +------------------+     +----------+
  | User   | ──→ | Query     | ──→ | Index Servers    | ──→ | Ranking  |
  +--------+     | Processor |     | (search inverted |     | Service  |
                 | (tokenize,|     |  index, retrieve |     | (PageRank|
                 |  spell    |     |  matching docs)  |     |  + ML)   |
                 |  correct) |     +------------------+     +----------+
                 +-----------+
```

### Key Design: Inverted Index

```
Forward Index (document → words):
  Doc 1: "the cat sat on the mat"
  Doc 2: "the dog sat on the log"
  Doc 3: "the cat chased the dog"

Inverted Index (word → documents):
  "the"    → [Doc1: pos(1,5), Doc2: pos(1,5), Doc3: pos(1,4)]
  "cat"    → [Doc1: pos(2), Doc3: pos(2)]
  "sat"    → [Doc1: pos(3), Doc2: pos(3)]
  "on"     → [Doc1: pos(4), Doc2: pos(4)]
  "mat"    → [Doc1: pos(6)]
  "dog"    → [Doc2: pos(2), Doc3: pos(5)]
  "log"    → [Doc2: pos(6)]
  "chased" → [Doc3: pos(3)]

Query: "cat sat"
  "cat" → [Doc1, Doc3]
  "sat" → [Doc1, Doc2]
  Intersection: [Doc1] — Doc1 contains both words!
  
  With positions: Doc1 has "cat" at pos 2 and "sat" at pos 3 → adjacent → phrase match!
```

**Index Sharding:**
```
100 billion pages → too large for one server

Sharding strategies:
  1. Document-based: Shard 1 has docs 1-10B, Shard 2 has docs 10B-20B, etc.
     → Query must hit ALL shards (scatter-gather) → merge results
     
  2. Term-based: Shard 1 has terms A-F, Shard 2 has terms G-L, etc.
     → Query for "cat sat" hits shard for "cat" and shard for "sat" → intersect
     → Uneven distribution (common terms have huge posting lists)

Google uses document-based sharding with thousands of index servers.
Each query fans out to all shards in parallel → results merged and ranked.
```

### Key Design: PageRank

```
PageRank: a page is important if important pages link to it.

  PR(A) = (1-d) + d × Σ (PR(Ti) / C(Ti))
  
  Where:
    d = damping factor (0.85)
    Ti = pages that link to A
    C(Ti) = number of outgoing links from Ti

Intuition:
  Page A has links from 3 pages:
    Page X (PR=0.8, 10 outgoing links) → contributes 0.8/10 = 0.08
    Page Y (PR=0.5, 5 outgoing links)  → contributes 0.5/5 = 0.10
    Page Z (PR=0.2, 2 outgoing links)  → contributes 0.2/2 = 0.10
  
  PR(A) = 0.15 + 0.85 × (0.08 + 0.10 + 0.10) = 0.15 + 0.238 = 0.388

Computed iteratively over the entire web graph (MapReduce/Spark job).
Modern search engines use PageRank as ONE of hundreds of ranking signals
(also: content relevance, freshness, user engagement, mobile-friendliness, etc.)
```

---


## 35.5 Design Dropbox / Google Drive

> **Problem:** Design a cloud file storage and synchronization service where users can upload, download, and sync files across multiple devices.

---

### Requirements & Scale

**Functional:** Upload/download files, sync across devices, file versioning, sharing (link, user), offline access, conflict resolution.

**Scale:** 500M users, 100M DAU, 1B files synced/day, average file size 500 KB, max file size 10 GB.

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ──→ | Block Server     | ──→ | Object Storage   |
  | (sync  |     | (chunking,       |     | (S3 — file       |
  |  agent)|     |  dedup, upload)  |     |  blocks)         |
  +--------+     +--------+---------+     +------------------+
       |                  |
       |         +--------v---------+     +------------------+
       |         | Metadata Service | ──→ | Metadata DB      |
       |         | (file tree,      |     | (PostgreSQL)     |
       |         |  versions, perms)|     +------------------+
       |         +------------------+
       |
       |         +------------------+
       +────────→| Sync Service     |  (push changes to other devices via WebSocket/long-poll)
                 | (Notification)   |
                 +------------------+
```

### Key Design: File Chunking and Deduplication

```
File Chunking:
  Large file (100 MB) → split into 4 MB chunks:
    Chunk 1: bytes 0 - 4MB       → hash: "abc123"
    Chunk 2: bytes 4MB - 8MB     → hash: "def456"
    ...
    Chunk 25: bytes 96MB - 100MB → hash: "xyz789"

Benefits:
  - Upload/download chunks in parallel (faster)
  - Resume interrupted transfers (retry only failed chunks)
  - Only upload CHANGED chunks on file edit (delta sync)
  - Deduplication across users (same chunk = same hash = store once)

Deduplication:
  User A uploads photo.jpg → chunks: [abc, def, ghi]
  User B uploads same photo.jpg → chunks: [abc, def, ghi]
  
  All chunks already exist in storage! User B's upload is instant.
  Storage: 1 copy instead of 2 → 50% savings for duplicate files.

Content-Defined Chunking (CDC):
  Instead of fixed 4 MB chunks, use a rolling hash (Rabin fingerprint)
  to find chunk boundaries based on content.
  
  Benefit: inserting 1 byte at the beginning of a file only changes 1 chunk
  (fixed chunking would change ALL chunks because boundaries shift)
```

### Key Design: Sync and Conflict Resolution

```
Sync Flow:
  1. User edits file on Device A
  2. Client detects change (file system watcher)
  3. Client computes chunk hashes → compares with server's chunk list
  4. Upload only changed/new chunks to Block Server
  5. Update metadata (new version, new chunk list)
  6. Sync Service notifies Device B (WebSocket push)
  7. Device B downloads changed chunks → reconstructs updated file

Conflict Resolution:
  User edits file on Device A (offline)
  User edits SAME file on Device B (offline)
  Both come online → CONFLICT!
  
  Resolution strategies:
    1. Last-write-wins (simple but loses data)
    2. Keep both versions: "report.docx" and "report (conflict).docx" (Dropbox's approach)
    3. Merge (only for text files — like Git merge)
    
  Dropbox approach: save both versions, let the user decide which to keep.
```

---

## 35.6 Design a Distributed Message Queue (Kafka-like)

> **Problem:** Design a distributed message queue that supports high-throughput, durable, ordered message delivery with consumer groups.

---

### Requirements & Scale

**Functional:** Producers publish messages to topics, consumers subscribe to topics, message ordering within partitions, consumer groups (each message processed by one consumer in the group), message retention (configurable, e.g., 7 days), replay (re-read old messages).

**Scale:** Millions of messages/second, petabytes of storage, thousands of topics.

### Architecture

```
  +----------+     +------------------+     +----------+
  | Producer | ──→ | Broker 1         | ──→ | Consumer |
  | (app)    |     | Partition 0 (L)  |     | Group A  |
  +----------+     | Partition 1 (F)  |     +----------+
                   +------------------+
  +----------+     +------------------+     +----------+
  | Producer | ──→ | Broker 2         | ──→ | Consumer |
  +----------+     | Partition 0 (F)  |     | Group B  |
                   | Partition 1 (L)  |     +----------+
                   +------------------+
                   
  L = Leader (handles reads/writes)
  F = Follower (replica for durability)
```

### Key Design: Log-Based Storage

```
Each partition is an append-only, immutable log:

  Partition 0:
  +-----+-----+-----+-----+-----+-----+-----+-----+
  | 0   | 1   | 2   | 3   | 4   | 5   | 6   | 7   |  ← offsets
  | msg | msg | msg | msg | msg | msg | msg | msg |
  +-----+-----+-----+-----+-----+-----+-----+-----+
                              ↑                   ↑
                        Consumer A            Producer
                        (offset=3)            (appends here)

  - Messages are NEVER deleted (until retention period expires)
  - Consumers track their own offset (position in the log)
  - Multiple consumer groups can read the same partition at different offsets
  - Replay: reset offset to 0 → re-read all messages
```

**Why Append-Only Logs are Fast:**
- Sequential writes only (no random I/O) → disk throughput, not IOPS, is the limit
- OS page cache is highly effective for sequential reads
- Zero-copy transfer: `sendfile()` syscall sends data directly from page cache to network socket (no user-space copy)
- Batching: producers batch messages → one write for many messages

### Key Design: Partitioning and Consumer Groups

```
Topic "orders" with 4 partitions:
  Partition 0: [order events for user_ids hashing to 0]
  Partition 1: [order events for user_ids hashing to 1]
  Partition 2: [order events for user_ids hashing to 2]
  Partition 3: [order events for user_ids hashing to 3]

Consumer Group "order-processors" with 4 consumers:
  Consumer A → reads Partition 0
  Consumer B → reads Partition 1
  Consumer C → reads Partition 2
  Consumer D → reads Partition 3
  
  Each partition is read by EXACTLY ONE consumer in the group.
  → Ordering is preserved within each partition.
  → Parallelism = number of partitions.

If Consumer C crashes:
  Rebalance: Consumer A → Partition 0, Consumer B → Partitions 1+2, Consumer D → Partition 3
  (Partition 2 is reassigned to a surviving consumer)
```

### Key Design: Replication

```
Replication Factor = 3:
  Partition 0: Leader on Broker 1, Followers on Broker 2 and Broker 3

Write path:
  1. Producer → Leader (Broker 1)
  2. Leader writes to local log
  3. Followers pull from Leader and write to their local logs
  4. When all in-sync replicas (ISR) have the message:
     → Message is "committed" → visible to consumers
  
  acks=0:   Producer doesn't wait for any ACK (fastest, may lose data)
  acks=1:   Producer waits for Leader ACK (fast, may lose if Leader crashes before replication)
  acks=all: Producer waits for ALL ISR ACKs (slowest, no data loss)

Leader failure:
  1. Controller detects Leader is down (ZooKeeper/KRaft)
  2. Promote an in-sync Follower to Leader
  3. Producers and consumers redirect to new Leader
  4. No data loss (if acks=all was used)
```

---


## 35.7 Design a Payment System

> **Problem:** Design a payment processing system that handles charges, refunds, and reconciliation with external payment providers — with zero tolerance for double-charging or lost payments.

---

### Requirements & Scale

**Functional:** Process payments (credit card, bank transfer, wallet), refunds, idempotency (no double charges), reconciliation with payment providers, ledger (audit trail), fraud detection.

**Scale:** 1M transactions/day (~12 TPS average, ~50 TPS peak).

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ──→ | Payment Gateway  | ──→ | Payment Service  |
  +--------+     | (API, idempotency|     | (orchestrate     |
                 |  key validation) |     |  payment flow)   |
                 +------------------+     +--------+---------+
                                                   |
                                          +--------v---------+
                                          | Ledger Service   |
                                          | (double-entry    |
                                          |  bookkeeping)    |
                                          +--------+---------+
                                                   |
                 +------------------+     +--------v---------+
                 | Risk Engine      |     | Payment Provider |
                 | (fraud detection)|     | Adapter          |
                 +------------------+     | (Stripe, PayPal, |
                                          |  bank APIs)      |
                                          +------------------+
                                          
                 +------------------+
                 | Reconciliation   |
                 | Service          |
                 | (match internal  |
                 |  records with    |
                 |  provider reports)|
                 +------------------+
```

### Key Design: Idempotency (No Double Charges)

```
Problem: Client sends payment request → network timeout → client retries
  Without idempotency: payment is charged TWICE!

Solution: Idempotency Key
  1. Client generates a unique key: "pay_abc123"
  2. Client sends: POST /payments, Idempotency-Key: pay_abc123
  3. Server checks: has "pay_abc123" been processed?
     → YES: return the stored result (no re-processing)
     → NO: process payment, store result keyed by "pay_abc123"
  4. Client retries with same key → gets same result (no double charge)

Storage: Redis (key: idempotency:{key}, value: response, TTL: 24-48 hours)
```

### Key Design: Double-Entry Ledger

```
Every payment creates TWO ledger entries that balance to zero:

  Payment of $100 from User to Merchant:
    Entry 1: DEBIT  User Account    -$100
    Entry 2: CREDIT Merchant Account +$100
    Sum: -100 + 100 = 0 ✅

  Refund of $100:
    Entry 3: DEBIT  Merchant Account -$100
    Entry 4: CREDIT User Account     +$100
    Sum: -100 + 100 = 0 ✅

  Ledger is APPEND-ONLY — entries are never modified or deleted.
  Balance = SUM of all entries for an account.
  
  This is the same system banks use — provides a complete, auditable history.
```

### Key Design: Payment State Machine

```
  CREATED → PROCESSING → SUCCEEDED
                ↓              ↓
            FAILED        REFUND_PENDING → REFUNDED
                               ↓
                          REFUND_FAILED

Each state transition is:
  1. Recorded in the database (with timestamp)
  2. Published as an event (Kafka) for downstream services
  3. Idempotent (re-processing the same transition is a no-op)
```

### Key Design: Reconciliation

```
Daily reconciliation process:
  1. Download settlement report from payment provider (Stripe, bank)
  2. Compare with internal ledger:
     - Match: internal record matches provider record ✅
     - Mismatch: amounts differ → flag for investigation
     - Missing: internal record exists but not in provider report → investigate
     - Extra: provider report has record not in our system → investigate
  3. Generate reconciliation report
  4. Alert on discrepancies > threshold

This catches:
  - Failed webhooks (payment succeeded at provider but we didn't record it)
  - Double charges (we charged twice but provider only processed once)
  - Fraud (unauthorized transactions)
```

---

## 35.8 Design a Social Network (Facebook)

> **Problem:** Design a social network with user profiles, friend connections, news feed, messaging, notifications, search, and ads.

---

### Requirements & Scale

**Functional:** User profiles, friend requests/connections, news feed (posts from friends), post creation (text, images, videos), likes/comments, messaging, notifications, search (people, posts), groups, events.

**Scale:** 3B users, 2B DAU, 100M posts/day, 10B feed views/day.

### Architecture (Simplified)

```
  +--------+     +-----------+
  | Client | ──→ | API GW    |
  +--------+     +-----+-----+
                       |
         +-------------+-------------+-------------+
         |             |             |             |
  +------v------+ +----v----+ +-----v-----+ +----v----+
  | User Service| | Feed    | | Post      | | Graph   |
  | (profiles,  | | Service | | Service   | | Service |
  |  auth)      | | (news   | | (create,  | | (friends|
  +-------------+ |  feed)  | |  media)   | |  social |
                  +---------+ +-----------+ |  graph) |
                                            +---------+
  +-------------+ +----------+ +----------+
  | Notification| | Search   | | Ad       |
  | Service     | | Service  | | Service  |
  +-------------+ +----------+ +----------+
```

### Key Design: Social Graph

```
Graph Storage:
  Adjacency list in a database (or graph database for complex queries):
  
  friendships table:
    user_id_1 | user_id_2 | status    | created_at
    123       | 456       | accepted  | 2024-01-15
    123       | 789       | accepted  | 2024-02-20
    456       | 789       | pending   | 2024-03-10

  Queries:
    "Get all friends of user 123":
      SELECT user_id_2 FROM friendships WHERE user_id_1 = 123 AND status = 'accepted'
      UNION
      SELECT user_id_1 FROM friendships WHERE user_id_2 = 123 AND status = 'accepted'
    
    "Mutual friends of 123 and 456":
      Intersection of friends(123) and friends(456)
    
    "Friends of friends" (people you may know):
      friends(friends(123)) - friends(123) - {123}

  For complex graph queries (2+ hops): use a graph database (Neo4j, TAO)
```

**Facebook TAO (The Associations and Objects):**
Facebook built a custom graph storage system optimized for the social graph:
- Objects: users, posts, comments, photos (nodes)
- Associations: friendships, likes, comments-on (edges)
- Cached in a massive distributed cache (memcache-based)
- Read-optimized: most queries are "get friends of X", "get posts liked by X"
- Write-through cache: writes go to database AND cache simultaneously

### Key Design: News Feed

Same hybrid fan-out approach as Twitter (Module 34.1):
- Fan-out on write for regular users (pre-compute feed)
- Fan-out on read for users with many friends (celebrities, public figures)
- Feed ranking: ML model scores each post (relevance, engagement prediction, recency, relationship strength)
- Feed cache: Redis sorted sets per user, top 500 posts

**Feed Ranking Signals:**

| Signal | Weight | Description |
|--------|--------|-------------|
| Relationship strength | High | How often you interact with the poster |
| Post type | Medium | Videos > photos > text (engagement data) |
| Recency | Medium | Newer posts ranked higher |
| Engagement | High | Posts with many likes/comments from your friends |
| Content quality | Medium | ML model predicts quality (not clickbait) |
| User preferences | Medium | Topics/people you engage with most |

---


## 35.9 Design a Ticket Booking System (Ticketmaster)

> **Problem:** Design a ticket booking system for concerts and events where thousands of users compete for limited seats simultaneously.

---

### Requirements & Scale

**Functional:** Browse events, view seat map, select seats, temporary hold during checkout, payment, confirmation, prevent double-booking, handle high concurrency for popular events.

**Scale:** 50K-500K concurrent users for popular events (Taylor Swift, World Cup), 10M tickets sold/year.

### The Core Challenge: High Concurrency for Limited Inventory

```
Problem: Taylor Swift concert — 80,000 seats, 500,000 users trying to buy simultaneously.
  
  If 500K users all hit "Buy" at the same time:
  → 500K concurrent requests for 80K seats
  → Must prevent double-booking (two users buying the same seat)
  → Must be fair (first-come, first-served)
  → Must handle payment timeout (user selected seat but didn't pay)
```

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ──→ | Virtual Waiting  | ──→ | Booking Service  |
  +--------+     | Room (Queue)     |     | (seat selection, |
                 | (rate-limit      |     |  hold, payment)  |
                 |  entry to system)|     +--------+---------+
                 +------------------+              |
                                          +--------v---------+
                                          | Inventory DB     |
                                          | (seat status:    |
                                          |  available/held/ |
                                          |  booked)         |
                                          +------------------+
```

### Key Design: Virtual Waiting Room

```
Instead of letting 500K users overwhelm the booking system:

1. All users enter a Virtual Waiting Room (queue)
2. Users are assigned a random position (lottery) or FIFO position
3. System admits users in batches (e.g., 1000 at a time)
4. Admitted users have 10 minutes to select seats and pay
5. If they don't complete → seats released → next batch admitted

Benefits:
  - Booking system handles only 1000 concurrent users (manageable)
  - Fair (everyone gets a chance)
  - No system overload
  - Users see their position in queue ("You are #15,234 in line")
```

### Key Design: Seat Locking

```
Seat States: AVAILABLE → HELD → BOOKED (or HELD → RELEASED)

1. User selects Seat A1:
   UPDATE seats SET status='HELD', held_by=user_123, held_until=NOW()+10min
   WHERE event_id=1 AND seat_id='A1' AND status='AVAILABLE';
   
   If affected_rows = 1 → seat held successfully (10-minute timer starts)
   If affected_rows = 0 → seat already taken → show error

2. User completes payment within 10 minutes:
   UPDATE seats SET status='BOOKED', booked_by=user_123
   WHERE event_id=1 AND seat_id='A1' AND held_by=user_123;

3. User doesn't pay within 10 minutes:
   Background job: UPDATE seats SET status='AVAILABLE', held_by=NULL
   WHERE held_until < NOW() AND status='HELD';
   → Seat released for other users
```

---

## 35.10 Design a Real-Time Gaming Leaderboard

> **Problem:** Design a leaderboard system that ranks millions of players by score in real-time, supporting score updates, rank queries, and top-N queries.

---

### Requirements & Scale

**Functional:** Update player score, get player's rank, get top N players, get players around a specific rank (neighborhood), leaderboard per game/region/time period.

**Scale:** 100M players, 1M score updates/second, rank query < 10ms.

### Key Design: Redis Sorted Sets

```
Redis Sorted Set: perfect data structure for leaderboards

  ZADD leaderboard 1500 "player:alice"
  ZADD leaderboard 2300 "player:bob"
  ZADD leaderboard 1800 "player:charlie"
  ZADD leaderboard 2100 "player:dave"

  Top 10 players (highest score first):
    ZREVRANGE leaderboard 0 9 WITHSCORES
    → bob:2300, dave:2100, charlie:1800, alice:1500

  Alice's rank:
    ZREVRANK leaderboard "player:alice"
    → 3 (0-indexed, so 4th place)

  Players around rank 50 (rank 45-55):
    ZREVRANGE leaderboard 45 55 WITHSCORES

  Update score:
    ZADD leaderboard 2000 "player:alice"  (replaces old score)
    or ZINCRBY leaderboard 100 "player:alice"  (increment by 100)

All operations: O(log N) — fast even with 100M members!
```

### Scaling Beyond One Redis Instance

```
Problem: 100M players × 50 bytes per entry = 5 GB → fits in one Redis instance
         But 1M updates/sec may exceed single Redis throughput (~100K ops/sec)

Solution 1: Sharding by game/region
  leaderboard:game1:us → Redis instance 1
  leaderboard:game1:eu → Redis instance 2
  leaderboard:game2:us → Redis instance 3
  
  Each shard handles a subset of players → lower QPS per shard

Solution 2: Write buffering
  Score updates → Kafka → batch consumer → ZADD to Redis every 1 second
  Trades real-time accuracy (1-second delay) for throughput

Solution 3: Hierarchical leaderboard
  Tier 1: Top 1000 players → single Redis (always accurate)
  Tier 2: All players → sharded, approximate ranking
  
  Most users only care about their approximate rank and the top players.
```

**Time-Based Leaderboards (daily, weekly, monthly):**

```
Daily leaderboard:
  Key: leaderboard:game1:daily:2025-03-15
  At midnight: create new key for the new day
  Old keys expire after 7 days (TTL)

Weekly leaderboard:
  Option 1: ZUNIONSTORE to merge daily leaderboards
  Option 2: Separate sorted set updated in parallel

All-time leaderboard:
  Persistent sorted set (no TTL)
```

---

## 35.11 Design a Content Moderation System

> **Problem:** Design a system that automatically detects and removes harmful content (hate speech, violence, spam, nudity) from a social media platform, with human review for edge cases.

---

### Requirements & Scale

**Functional:** Automated ML detection (text, image, video), confidence scoring, human review queue, appeals process, policy rules engine, audit trail.

**Scale:** 1B posts/day, < 1 second for automated detection, 100K human reviews/day.

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Post   | ──→ | Moderation       | ──→ | ML Pipeline      |
  | Created|     | Gateway          |     | (parallel checks)|
  | (event)|     | (route to checks)|     +--------+---------+
  +--------+     +------------------+              |
                                          +--------v---------+
                                          | Decision Engine  |
                                          | (combine ML      |
                                          |  scores + rules) |
                                          +--------+---------+
                                                   |
                                    +--------------+--------------+
                                    |              |              |
                              +-----v-----+  +----v----+  +-----v-----+
                              | Auto-Allow|  | Human   |  | Auto-Remove|
                              | (high     |  | Review  |  | (high      |
                              |  confidence| | Queue   |  |  confidence|
                              |  safe)    |  | (edge   |  |  violation)|
                              +-----------+  |  cases) |  +-----------+
                                             +---------+
                                                  |
                                             +----v----+
                                             | Appeals |
                                             | Service |
                                             +---------+
```

### Key Design: ML Pipeline

```
Each post goes through multiple ML models in parallel:

  Text Analysis:
    ├── Hate speech classifier (BERT-based) → score: 0.0-1.0
    ├── Spam detector → score: 0.0-1.0
    ├── Toxicity classifier → score: 0.0-1.0
    └── Language detection → language code

  Image Analysis:
    ├── Nudity/NSFW detector (CNN) → score: 0.0-1.0
    ├── Violence detector → score: 0.0-1.0
    ├── OCR → extract text → run text analysis on extracted text
    └── Logo/brand detection → score: 0.0-1.0

  Video Analysis:
    ├── Sample frames (1 per second) → run image analysis on each
    ├── Audio transcription → run text analysis on transcript
    └── Known harmful video fingerprint matching (hash-based)

Decision Engine:
  if any_score > 0.95 → AUTO-REMOVE (high confidence violation)
  if any_score > 0.70 → HUMAN REVIEW (uncertain — needs human judgment)
  if all_scores < 0.30 → AUTO-ALLOW (high confidence safe)
  else → HUMAN REVIEW (borderline)
```

### Key Design: Human Review Queue

```
Priority Queue (highest priority reviewed first):
  Priority 1: Reported by users (multiple reports = higher priority)
  Priority 2: ML flagged with high scores (0.70-0.95)
  Priority 3: ML flagged with medium scores (0.50-0.70)
  Priority 4: Random sampling (quality assurance — check ML accuracy)

Reviewer workflow:
  1. Reviewer picks next item from queue
  2. Sees: post content, ML scores, context (user history, reports)
  3. Decision: Allow / Remove / Escalate (to senior reviewer)
  4. Decision is recorded (audit trail) and applied
  5. ML model is retrained periodically using human decisions as labels

Quality assurance:
  - Same post reviewed by 2+ reviewers independently (for sensitive content)
  - Agreement rate tracked — disagreements escalated to senior reviewers
  - Reviewer accuracy tracked — low-accuracy reviewers get additional training
```

---

## 35.12 Design a Metrics/Monitoring System (Prometheus-like)

> **Problem:** Design a time-series metrics collection, storage, querying, and alerting system for monitoring infrastructure and applications.

---

### Requirements & Scale

**Functional:** Collect metrics from thousands of services, store time-series data, query with aggregations (rate, sum, avg, percentiles), dashboards, alerting with thresholds and anomaly detection.

**Scale:** 10M active time series, 1M data points/second ingestion, 1K queries/second, 90-day retention.

### Architecture

```
  +------------------+     +------------------+     +------------------+
  | Metric Sources   | ──→ | Collector /      | ──→ | Time-Series DB   |
  | - App /metrics   |     | Scraper          |     | (custom or       |
  | - Node exporter  |     | (pull every 15s) |     |  InfluxDB/       |
  | - Custom metrics |     +------------------+     |  VictoriaMetrics)|
  +------------------+                               +--------+---------+
                                                              |
                                                     +--------v---------+
                                                     | Query Engine     |
                                                     | (PromQL-like)    |
                                                     +--------+---------+
                                                              |
                                                    +---------+---------+
                                                    |                   |
                                              +-----v-----+    +-------v-------+
                                              | Dashboard  |    | Alert Manager |
                                              | (Grafana)  |    | (evaluate     |
                                              +-----------+    |  rules, route  |
                                                               |  notifications)|
                                                               +---------------+
```

### Key Design: Time-Series Storage

```
Time-series data structure:
  Metric: http_requests_total{service="api", method="GET", status="200"}
  
  Timestamp    | Value
  1625097600   | 15234
  1625097615   | 15289  (+55 in 15 seconds)
  1625097630   | 15342  (+53)
  1625097645   | 15401  (+59)
  ...

Storage optimization:
  1. Delta encoding: store differences instead of absolute values
     [15234, +55, +53, +59, ...] → much smaller than [15234, 15289, 15342, 15401]
  
  2. Gorilla compression (Facebook): 
     Timestamps: delta-of-delta encoding (most deltas are 0 or very small)
     Values: XOR encoding (consecutive values are often similar)
     Result: 12 bytes per data point → 1.37 bytes per data point (8.7x compression)
  
  3. Time-based partitioning:
     Each 2-hour block is a separate file on disk
     Old blocks are compacted and compressed
     Queries only read relevant time blocks
```

**Storage Estimation:**

```
10M time series × 1 sample every 15 seconds × 90 days retention

Samples per day: 10M × (86,400 / 15) = 57.6 billion samples/day
Samples for 90 days: 57.6B × 90 = 5.18 trillion samples

Uncompressed: 5.18T × 16 bytes (timestamp + value) = 83 TB
With Gorilla compression (~8x): 83 TB / 8 ≈ 10 TB

→ Fits on a small cluster of servers with SSDs
```

### Key Design: Alerting

```
Alert Rule:
  IF rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
  FOR 5 minutes
  THEN alert "HighErrorRate"

Evaluation:
  1. Alert Manager evaluates all rules every 15-30 seconds
  2. If condition is true → start "pending" timer
  3. If condition is true for the "FOR" duration (5 min) → fire alert
  4. Route alert based on labels:
     severity=critical → PagerDuty (page on-call engineer)
     severity=warning → Slack channel
     severity=info → email digest
  5. If condition becomes false → resolve alert

Alert Grouping:
  If 100 services all have high error rates at the same time:
  → Don't send 100 separate alerts!
  → Group by common label (e.g., cluster, datacenter)
  → Send ONE alert: "High error rate across 100 services in us-east-1"

Silencing:
  During planned maintenance: silence alerts for affected services
  Prevents alert fatigue during known events
```

---

## Summary & Key Takeaways

| Problem | Core Challenge | Key Design Decision |
|---------|---------------|-------------------|
| **YouTube** | Video transcoding at scale | Async transcoding pipeline; adaptive bitrate streaming (HLS/DASH); CDN for delivery |
| **Google Docs** | Real-time collaboration without conflicts | OT (Google's approach) or CRDTs (Figma's approach); WebSocket for real-time sync |
| **Google Maps** | Route planning at continent scale | Contraction Hierarchies for fast routing; pre-rendered map tiles; real-time traffic from GPS data |
| **Google Search** | Searching 100B+ pages in milliseconds | Inverted index (sharded); PageRank + ML for ranking; massive parallelism |
| **Dropbox** | File sync across devices | Content-defined chunking; deduplication; delta sync (only changed chunks) |
| **Message Queue** | High-throughput durable messaging | Append-only log; partitioning for parallelism; consumer groups; ISR replication |
| **Payment System** | No double charges, no lost payments | Idempotency keys; double-entry ledger; payment state machine; reconciliation |
| **Social Network** | News feed at 3B user scale | Hybrid fan-out; social graph (TAO); ML-based feed ranking |
| **Ticketmaster** | 500K users competing for 80K seats | Virtual waiting room (queue); seat locking with timeout; pessimistic locking |
| **Leaderboard** | Real-time ranking of 100M players | Redis sorted sets (ZADD, ZREVRANK); sharding by game/region |
| **Content Moderation** | 1B posts/day, detect harmful content | Parallel ML pipeline (text + image + video); confidence-based routing (auto/human/remove) |
| **Metrics System** | 1M data points/sec, 90-day retention | Time-series DB with Gorilla compression; delta encoding; time-based partitioning |

---

## Interview Tips for Module 35

1. **YouTube** — focus on the transcoding pipeline (async, multiple resolutions) and adaptive bitrate streaming; mention CDN is essential
2. **Google Docs** — explain OT vs CRDT clearly; draw the conflict resolution example; mention WebSocket for real-time
3. **Google Maps** — mention Contraction Hierarchies for routing (not just Dijkstra); explain map tiles and zoom levels
4. **Google Search** — draw the inverted index; explain PageRank intuitively; mention index sharding (document-based)
5. **Dropbox** — chunking and deduplication are the key insights; explain delta sync; mention conflict resolution
6. **Message Queue** — explain the append-only log; partition = unit of parallelism; consumer groups; ISR replication
7. **Payment System** — idempotency is the #1 concern; explain double-entry ledger; mention reconciliation
8. **Social Network** — hybrid fan-out for feed; social graph storage (TAO); ML-based feed ranking
9. **Ticketmaster** — virtual waiting room is the key insight; explain seat locking with timeout
10. **Leaderboard** — Redis sorted sets are the answer; explain ZADD, ZREVRANK, ZREVRANGE; discuss sharding for scale
11. **Content Moderation** — parallel ML models with confidence thresholds; human review for edge cases; appeals process
12. **Metrics System** — Gorilla compression for time-series; time-based partitioning; alert grouping to prevent fatigue
