# Module 35: HLD Practice Problems (Hard)

> These are the most challenging system design problems — asked at senior-level interviews at top tech companies. Each problem involves complex trade-offs, multiple interacting subsystems, and requires deep knowledge of distributed systems concepts.

> **Ruby Context:** These problems are largely architecture-focused and language-agnostic. Ruby/Rails implementation notes are included where they add practical value — particularly for video processing pipelines (Sidekiq + FFmpeg), collaborative editing (Action Cable/AnyCable), payment systems (Stripe integration + idempotency), leaderboards (Redis sorted sets), and content moderation (ML service integration). For compute-heavy subsystems (transcoding, ML inference, routing algorithms), Ruby typically orchestrates while delegating to specialized services.

---

## 35.1 Design YouTube / Video Streaming

### Requirements & Scale

**Scale:** 800M DAU, 500 hours uploaded/min, 1B views/day, 250 Tbps streaming bandwidth.

### Architecture

```
  Upload Path:
  Client → Upload Service (pre-signed S3 URL, chunked) → S3 (raw) → Transcode Pipeline → S3 (transcoded) → CDN

  Streaming Path:
  Viewer → CDN (CloudFront) → Origin (S3) → Adaptive Bitrate (HLS/DASH)
```

### Key Design: Transcoding Pipeline

```
Raw Video (1080p, 1 GB) → Message Queue → Transcoding Workers (auto-scaled):
  ├── 240p  (400 Kbps)   → 50 MB
  ├── 360p  (800 Kbps)   → 100 MB
  ├── 480p  (1.5 Mbps)   → 200 MB
  ├── 720p  (3 Mbps)     → 400 MB
  ├── 1080p (5 Mbps)     → 650 MB
  └── 1080p (AV1, 3 Mbps)→ 400 MB

Each resolution split into 2-10 second segments for adaptive bitrate streaming.
```

> **Ruby Context:** Ruby orchestrates the pipeline; FFmpeg does the actual transcoding. Sidekiq manages the job queue.

```ruby
# Video upload triggers transcoding via Sidekiq
class VideoTranscodeWorker
  include Sidekiq::Job
  sidekiq_options queue: :transcoding, retry: 3

  RESOLUTIONS = {
    '240p'  => { width: 426,  height: 240,  bitrate: '400k' },
    '360p'  => { width: 640,  height: 360,  bitrate: '800k' },
    '480p'  => { width: 854,  height: 480,  bitrate: '1500k' },
    '720p'  => { width: 1280, height: 720,  bitrate: '3000k' },
    '1080p' => { width: 1920, height: 1080, bitrate: '5000k' }
  }

  def perform(video_id)
    video = Video.find(video_id)
    raw_path = download_from_s3(video.raw_s3_key)

    RESOLUTIONS.each do |name, opts|
      # Transcode to HLS segments using FFmpeg
      output_dir = "/tmp/transcode/#{video_id}/#{name}"
      system("ffmpeg -i #{raw_path} -vf scale=#{opts[:width]}:#{opts[:height]} " \
             "-b:v #{opts[:bitrate]} -hls_time 6 -hls_list_size 0 " \
             "-f hls #{output_dir}/playlist.m3u8")

      # Upload segments to S3
      upload_directory_to_s3(output_dir, "videos/#{video_id}/#{name}/")
    end

    # Generate master manifest (lists all resolutions)
    generate_master_manifest(video_id)
    video.update!(status: :ready, transcoded_at: Time.current)
  end
end

# Upload controller with pre-signed URL
class Api::V1::VideosController < ApplicationController
  def create
    video = Video.create!(user: current_user, status: :uploading, title: params[:title])
    key = "raw/#{video.id}/#{params[:filename]}"
    url = Aws::S3::Presigner.new.presigned_url(:put_object,
      bucket: ENV['S3_RAW_BUCKET'], key: key, expires_in: 3600)

    render json: { video_id: video.id, upload_url: url, key: key }
  end

  # Called after upload completes (or via S3 event notification)
  def transcode
    video = Video.find(params[:id])
    video.update!(status: :transcoding)
    VideoTranscodeWorker.perform_async(video.id)
    render json: { status: 'transcoding' }
  end
end
```

**Adaptive Bitrate Streaming:** Client requests HLS manifest (.m3u8), picks resolution based on bandwidth. Switches mid-stream without interruption.

---

## 35.2 Design Google Docs / Collaborative Editing

### Requirements & Scale

**Scale:** 100M documents, 10M concurrent editors, < 100ms edit propagation.

### Key Design: Conflict Resolution

```
Alice and Bob edit "Hello World" simultaneously:
  Alice (pos 6): inserts "Beautiful " → "Hello Beautiful World"
  Bob (pos 5): deletes "o" → "Hell World"

OT (Operational Transformation) — Google Docs' approach:
  Transform operations against each other so both orderings produce the same result.

CRDTs — Figma/Yjs approach:
  Each character has a unique ordered ID. Operations commute naturally.
```

| Feature | OT | CRDT |
|---------|----|----|
| Server requirement | Central server transforms ops | Can work peer-to-peer |
| Offline support | Limited | Excellent |
| Proven at scale | Google Docs | Figma, Apple Notes |

> **Ruby Context:** Use **AnyCable** for WebSocket connections (high concurrency) and a collaboration engine. For simpler real-time features, Action Cable works. The `yjs` library (JavaScript CRDT) handles client-side; Ruby manages persistence and access control.

```ruby
# Collaborative editing channel (AnyCable)
class DocumentChannel < ApplicationCable::Channel
  def subscribed
    document = Document.find(params[:id])
    authorize! :edit, document
    stream_from "document:#{document.id}"
    # Track active editors
    Redis.current.sadd("editors:#{document.id}", current_user.id)
  end

  def receive(data)
    # Broadcast operation to all other editors
    ActionCable.server.broadcast(
      "document:#{data['document_id']}",
      { type: 'operation', operation: data['operation'], user_id: current_user.id }
    )
    # Persist operation for version history (async)
    DocumentOperationWorker.perform_async(data['document_id'], data['operation'])
  end

  def unsubscribed
    Redis.current.srem("editors:#{params[:id]}", current_user.id)
  end
end
```

---

## 35.3 Design Google Maps

### Requirements & Scale

**Scale:** 100M DAU, 1M concurrent navigation sessions, real-time traffic from millions of devices.

### Key Designs

**Map Tiles:** Pre-rendered at 18+ zoom levels, served via CDN. Vector tiles (modern) send data for client-side rendering.

**Route Planning:** Contraction Hierarchies (pre-processed graph) — millisecond queries for continent-scale routes. Much faster than Dijkstra/A*.

**Real-Time Traffic:** GPS data from phones → map-match to road segments → aggregate speed → update routing graph edge weights → color-coded traffic layer.

> **Ruby Context:** Ruby would serve as the API layer (user accounts, saved places, search) while routing and tile rendering are handled by specialized services (OSRM for routing, Mapbox/custom for tiles). The `rgeo` gem handles geospatial operations in Ruby.

---

## 35.4 Design Google Search

### Requirements & Scale

**Scale:** 100K QPS, 100B+ pages indexed, < 500ms query latency.

### Key Designs

**Inverted Index:** word → list of documents containing it (with positions). Sharded by document across thousands of servers. Query fans out to all shards in parallel.

**PageRank:** Iterative algorithm over the web graph (computed offline via Spark). One of hundreds of ranking signals alongside content relevance, freshness, engagement.

> **Ruby Context:** Ruby is not used for search engine internals. However, Rails apps commonly integrate with Elasticsearch via `searchkick` or `elasticsearch-ruby` for application-level search that mirrors these concepts (inverted index, relevance scoring, sharding).

---

## 35.5 Design Dropbox / Google Drive

### Requirements & Scale

**Scale:** 100M DAU, 1B files synced/day, avg 500 KB, max 10 GB.

### Key Design: Chunking + Dedup + Delta Sync

```
File (100 MB) → split into 4 MB chunks → hash each chunk
  Only upload chunks that changed (delta sync)
  Same chunk across users = stored once (deduplication)
  Content-Defined Chunking (rolling hash) for stable boundaries

Sync: file watcher → detect change → compute chunk hashes → upload new chunks → notify other devices via WebSocket
Conflict: save both versions ("report.docx" + "report (conflict).docx")
```

> **Ruby Context:** ActiveStorage handles simple file uploads. For Dropbox-like sync, you'd build a custom service. The chunking/dedup logic runs in the client (desktop app), while Ruby manages metadata, permissions, and notifications.

```ruby
# File metadata service (Rails)
class SyncService
  def self.upload_chunks(user_id, file_path, chunk_hashes)
    file_record = SyncedFile.find_or_initialize_by(user_id: user_id, path: file_path)

    # Determine which chunks are new (not already in storage)
    existing = ChunkStore.where(hash: chunk_hashes).pluck(:hash).to_set
    new_chunks = chunk_hashes.reject { |h| existing.include?(h) }

    # Return pre-signed URLs for new chunks only (dedup!)
    urls = new_chunks.map do |hash|
      { hash: hash, url: presign_upload("chunks/#{hash}") }
    end

    file_record.update!(chunk_hashes: chunk_hashes, version: file_record.version + 1)

    # Notify other devices
    ActionCable.server.broadcast("sync:#{user_id}", {
      type: 'file_changed', path: file_path, version: file_record.version
    })

    { new_chunks_to_upload: urls, version: file_record.version }
  end
end
```

---

## 35.6 Design a Distributed Message Queue (Kafka-like)

### Requirements & Scale

**Scale:** Millions of messages/sec, petabytes of storage, thousands of topics.

### Key Designs

**Append-Only Log:** Sequential writes only → disk throughput (not IOPS) is the limit. Zero-copy (`sendfile`) for network transfer.

**Partitioning:** Topic split into partitions. Each partition = ordered log. One consumer per partition per group. Parallelism = number of partitions.

**Replication:** Leader handles reads/writes. Followers replicate. `acks=all` for no data loss. ISR (In-Sync Replicas) for leader election.

> **Ruby Context:** Ruby apps interact with Kafka via **Karafka** (consumer framework) or **rdkafka-ruby** (low-level client). You don't build Kafka in Ruby — you use it.

```ruby
# Karafka consumer (consuming from Kafka)
class OrderEventsConsumer < Karafka::BaseConsumer
  def consume
    messages.each do |message|
      event = JSON.parse(message.payload)
      case event['type']
      when 'order_created' then ProcessOrderWorker.perform_async(event)
      when 'order_paid'    then FulfillOrderWorker.perform_async(event)
      end
    end
  end
end

# Producing to Kafka
Karafka.producer.produce_async(
  topic: 'order_events',
  payload: { type: 'order_created', order_id: order.id }.to_json,
  key: order.id.to_s  # Partition key — ensures ordering per order
)
```

---

## 35.7 Design a Payment System

### Requirements & Scale

**Scale:** 1M transactions/day, zero tolerance for double-charging or lost payments.

### Key Designs

**Idempotency:** Client sends unique key with each request. Server checks if already processed before charging.

**Double-Entry Ledger:** Every payment = DEBIT + CREDIT entries that sum to zero. Append-only. Complete audit trail.

**State Machine:** CREATED → PROCESSING → SUCCEEDED/FAILED → REFUND_PENDING → REFUNDED.

**Reconciliation:** Daily comparison of internal ledger vs payment provider settlement reports.

> **Ruby Context:** Stripe is the most common payment provider for Ruby apps. The `stripe` gem + idempotency keys + webhooks handle most of the complexity.

```ruby
# Payment service with idempotency
class PaymentService
  def self.charge(order_id:, amount:, idempotency_key:)
    # Check if already processed
    existing = Payment.find_by(idempotency_key: idempotency_key)
    return existing if existing&.succeeded?

    payment = Payment.create!(
      order_id: order_id, amount: amount,
      idempotency_key: idempotency_key, status: :processing
    )

    begin
      # Stripe handles its own idempotency with the key
      charge = Stripe::Charge.create(
        { amount: (amount * 100).to_i, currency: 'usd',
          customer: order.user.stripe_customer_id,
          metadata: { order_id: order_id } },
        { idempotency_key: idempotency_key }
      )

      payment.update!(status: :succeeded, provider_id: charge.id)

      # Double-entry ledger
      Ledger.record!(
        debit_account: "user:#{order.user_id}", credit_account: "merchant:#{order.merchant_id}",
        amount: amount, reference: payment.id
      )

      payment
    rescue Stripe::CardError => e
      payment.update!(status: :failed, error_message: e.message)
      payment
    end
  end
end

# Webhook handler for async payment events
class StripeWebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token

  def create
    event = Stripe::Webhook.construct_event(
      request.body.read, request.env['HTTP_STRIPE_SIGNATURE'], ENV['STRIPE_WEBHOOK_SECRET']
    )

    case event.type
    when 'charge.succeeded'
      PaymentConfirmationWorker.perform_async(event.data.object.id)
    when 'charge.refunded'
      RefundProcessingWorker.perform_async(event.data.object.id)
    when 'charge.disputed'
      DisputeAlertWorker.perform_async(event.data.object.id)
    end

    head :ok
  rescue Stripe::SignatureVerificationError
    head :bad_request
  end
end

# Reconciliation job (daily)
class ReconciliationWorker
  include Sidekiq::Job
  sidekiq_options queue: :finance

  def perform(date = Date.yesterday.to_s)
    # Fetch settlement from Stripe
    stripe_charges = Stripe::Charge.list(created: { gte: date.to_time.to_i, lt: (date.to_date + 1).to_time.to_i })

    # Compare with internal records
    stripe_charges.auto_paging_each do |charge|
      internal = Payment.find_by(provider_id: charge.id)
      if internal.nil?
        ReconciliationAlert.create!(type: :missing_internal, provider_id: charge.id)
      elsif internal.amount != charge.amount / 100.0
        ReconciliationAlert.create!(type: :amount_mismatch, payment_id: internal.id)
      end
    end
  end
end
```

---

## 35.8 Design a Social Network (Facebook)

### Key Designs

**Social Graph:** Adjacency list in database (or graph DB for complex queries). Facebook's TAO: custom graph storage optimized for read-heavy social graph queries, cached in distributed memcache.

**News Feed:** Hybrid fan-out (same as Twitter, Module 34.1). ML-based ranking: relationship strength, post type, recency, engagement, content quality.

> **Ruby Context:** Same patterns as Twitter (Module 34.1) — Sidekiq for fan-out, Redis sorted sets for feed cache, Karafka for event streaming. For graph queries (mutual friends, people you may know), consider Neo4j with the `neo4j` gem or PostgreSQL recursive CTEs.

---

## 35.9 Design a Ticket Booking System (Ticketmaster)

### The Core Challenge

500K users competing for 80K seats simultaneously.

### Key Designs

**Virtual Waiting Room:** Queue users, admit in batches (1000 at a time). Prevents system overload. Users see position in queue.

**Seat Locking:** `UPDATE seats SET status='HELD', held_until=NOW()+10min WHERE status='AVAILABLE'` — atomic. Background job releases expired holds.

> **Ruby Context:**

```ruby
# Virtual waiting room with Redis
class WaitingRoomService
  def self.join(event_id, user_id)
    position = Redis.current.zadd("waiting_room:#{event_id}", Time.now.to_f, user_id)
    rank = Redis.current.zrank("waiting_room:#{event_id}", user_id)
    { position: rank + 1, estimated_wait: rank * 0.5 } # ~0.5 sec per person
  end

  def self.admit_batch(event_id, batch_size: 1000)
    # Admit next batch from waiting room
    users = Redis.current.zrangebyscore("waiting_room:#{event_id}", '-inf', '+inf',
                                         limit: [0, batch_size])
    users.each do |user_id|
      Redis.current.zrem("waiting_room:#{event_id}", user_id)
      Redis.current.setex("admitted:#{event_id}:#{user_id}", 600, '1') # 10 min session
      ActionCable.server.broadcast("waiting:#{user_id}", { type: 'admitted', event_id: event_id })
    end
  end
end

# Seat locking with ActiveRecord
class SeatService
  def self.hold(event_id:, seat_id:, user_id:)
    ActiveRecord::Base.transaction do
      seat = Seat.lock('FOR UPDATE').find_by!(event_id: event_id, seat_id: seat_id)
      raise SeatUnavailableError unless seat.available?

      seat.update!(status: :held, held_by: user_id, held_until: 10.minutes.from_now)
      ReleaseSeatWorker.perform_at(10.minutes.from_now, seat.id)
      seat
    end
  end

  def self.book(event_id:, seat_id:, user_id:)
    ActiveRecord::Base.transaction do
      seat = Seat.lock('FOR UPDATE').find_by!(event_id: event_id, seat_id: seat_id, held_by: user_id)
      raise SeatExpiredError if seat.held_until < Time.current

      seat.update!(status: :booked, booked_by: user_id)
      BookingConfirmationWorker.perform_async(user_id, seat.id)
    end
  end
end
```

---

## 35.10 Design a Real-Time Gaming Leaderboard

### Key Design: Redis Sorted Sets

```
ZADD leaderboard 1500 "player:alice"     — set score
ZREVRANK leaderboard "player:alice"      — get rank (0-indexed)
ZREVRANGE leaderboard 0 9 WITHSCORES    — top 10
ZINCRBY leaderboard 100 "player:alice"   — increment score

All operations: O(log N) — fast with 100M members!
```

> **Ruby Context:**

```ruby
class LeaderboardService
  def self.update_score(game_id, player_id, score)
    Redis.current.zadd("leaderboard:#{game_id}", score, player_id)
  end

  def self.increment_score(game_id, player_id, delta)
    Redis.current.zincrby("leaderboard:#{game_id}", delta, player_id)
  end

  def self.rank(game_id, player_id)
    rank = Redis.current.zrevrank("leaderboard:#{game_id}", player_id)
    rank ? rank + 1 : nil  # 1-indexed for display
  end

  def self.top(game_id, count: 10)
    Redis.current.zrevrange("leaderboard:#{game_id}", 0, count - 1, with_scores: true)
      .map { |id, score| { player_id: id, score: score.to_i } }
  end

  def self.neighborhood(game_id, player_id, range: 5)
    rank = Redis.current.zrevrank("leaderboard:#{game_id}", player_id)
    return [] unless rank
    start = [rank - range, 0].max
    Redis.current.zrevrange("leaderboard:#{game_id}", start, rank + range, with_scores: true)
  end
end

# Time-based leaderboards (daily/weekly)
# Key: leaderboard:game1:daily:2025-03-15 (TTL: 7 days)
# Key: leaderboard:game1:weekly:2025-W11 (TTL: 30 days)
```

**Scaling:** Shard by game/region. For 1M updates/sec, buffer writes via Kafka → batch ZADD every second.

---

## 35.11 Design a Content Moderation System

### Architecture

```
Post Created → ML Pipeline (parallel):
  ├── Text: hate speech, spam, toxicity classifiers
  ├── Image: nudity, violence, OCR → text analysis
  └── Video: sample frames + audio transcription

Decision Engine:
  score > 0.95 → AUTO-REMOVE
  score 0.70-0.95 → HUMAN REVIEW queue
  score < 0.30 → AUTO-ALLOW
```

> **Ruby Context:** Ruby orchestrates the pipeline; ML models run as separate services (Python/TensorFlow). Sidekiq manages the workflow.

```ruby
# Content moderation pipeline
class ModerationPipelineWorker
  include Sidekiq::Job

  def perform(post_id)
    post = Post.find(post_id)
    scores = {}

    # Call ML services in parallel (using futures or async HTTP)
    threads = []
    threads << Thread.new { scores[:text] = TextModerationClient.analyze(post.content) }
    threads << Thread.new { scores[:image] = ImageModerationClient.analyze(post.image_url) } if post.image_url
    threads.each(&:join)

    max_score = scores.values.map { |s| s[:violation_score] }.max

    case max_score
    when 0.95..1.0
      post.update!(moderation_status: :removed, removal_reason: 'auto_detected')
    when 0.70...0.95
      HumanReviewQueue.enqueue(post_id: post.id, scores: scores, priority: max_score)
    else
      post.update!(moderation_status: :approved)
    end
  end
end
```

---

## 35.12 Design a Metrics/Monitoring System (Prometheus-like)

### Key Designs

**Time-Series Storage:** Gorilla compression (8x reduction). Delta-of-delta for timestamps, XOR for values. Time-based partitioning (2-hour blocks).

**Alerting:** Evaluate rules every 15-30s. Fire after "FOR" duration. Group related alerts. Route by severity (PagerDuty/Slack/email).

> **Ruby Context:** Ruby apps expose metrics via `prometheus-client` or `yabeda` gems (see Module 30). The monitoring system itself is typically Prometheus + Grafana (infrastructure-level, not built in Ruby). Ruby's role is being a good metrics source.

---

## Summary & Key Takeaways

| Problem | Core Challenge | Key Design | Ruby Tools |
|---------|---------------|-----------|-----------|
| **YouTube** | Transcoding at scale | Async pipeline; HLS/DASH; CDN | Sidekiq + FFmpeg, aws-sdk-s3 |
| **Google Docs** | Real-time collaboration | OT or CRDTs; WebSocket | AnyCable, Action Cable |
| **Google Maps** | Continent-scale routing | Contraction Hierarchies; map tiles; traffic | rgeo gem, API layer only |
| **Google Search** | 100B+ pages in ms | Inverted index; PageRank; sharding | Searchkick (app-level search) |
| **Dropbox** | File sync | Chunking + dedup + delta sync | ActiveStorage, custom sync service |
| **Message Queue** | High-throughput durable messaging | Append-only log; partitions; ISR | Karafka, rdkafka-ruby |
| **Payment System** | No double charges | Idempotency keys; double-entry ledger | stripe gem, ActiveRecord transactions |
| **Social Network** | Feed at 3B scale | Hybrid fan-out; graph storage; ML ranking | Sidekiq, Redis, neo4j gem |
| **Ticketmaster** | 500K users, 80K seats | Virtual waiting room; seat locking | Redis (queue), ActiveRecord lock |
| **Leaderboard** | Real-time 100M players | Redis sorted sets | redis-rb (ZADD, ZREVRANK) |
| **Content Moderation** | 1B posts/day | Parallel ML + confidence routing | Sidekiq, HTTP clients to ML services |
| **Metrics System** | 1M points/sec | Gorilla compression; time partitioning | prometheus-client, yabeda |

---

## Interview Tips for Module 35

1. **YouTube** — transcoding pipeline (async, multiple resolutions) + adaptive bitrate + CDN; show Sidekiq + FFmpeg orchestration
2. **Google Docs** — OT vs CRDT clearly; draw conflict example; mention AnyCable for WebSocket scale
3. **Google Maps** — Contraction Hierarchies (not Dijkstra); map tiles + zoom levels; real-time traffic from GPS
4. **Google Search** — inverted index; PageRank intuition; document-based sharding
5. **Dropbox** — chunking + dedup are key; delta sync; conflict = save both versions
6. **Message Queue** — append-only log; partition = parallelism; consumer groups; ISR replication
7. **Payment System** — idempotency keys (#1 concern); double-entry ledger; Stripe webhook handling; reconciliation
8. **Social Network** — hybrid fan-out; social graph (TAO); ML feed ranking
9. **Ticketmaster** — virtual waiting room (queue users, admit in batches); seat locking with timeout
10. **Leaderboard** — Redis sorted sets (ZADD, ZREVRANK, ZREVRANGE); sharding for scale
11. **Content Moderation** — parallel ML models; confidence thresholds; human review queue
12. **Metrics System** — Gorilla compression; time-based partitioning; alert grouping