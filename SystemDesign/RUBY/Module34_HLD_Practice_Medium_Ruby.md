# Module 34: HLD Practice Problems (Medium)

> These medium-difficulty problems are the most commonly asked in system design interviews at top tech companies. Each problem requires combining multiple concepts — databases, caching, messaging, scaling, and trade-offs.

> **Ruby Context:** Each problem includes Ruby/Rails implementation notes. Key tools: **Rails API** (Puma), **Sidekiq** (fan-out, background jobs), **Redis** (caching, sorted sets, GEO, pub/sub), **Karafka** (Kafka), **Action Cable** / **AnyCable** (WebSockets), **ActiveRecord** (locking, transactions), **aws-sdk-s3** (media), **Searchkick** / **elasticsearch-ruby** (search), and **Faraday** (service-to-service HTTP).

---

## 34.1 Design Twitter / News Feed

### Requirements & Scale

**Functional:** Post tweets (280 chars + media), follow/unfollow, view home timeline, search, like/retweet.

**Scale:** 500M users, 300M DAU, 35K reads/sec, 7K writes/sec.

### Architecture

```
  +--------+     +-----------+     +----------------+     +-------------+
  | Client | ──→ | API GW    | ──→ | Tweet Service  | ──→ | Tweet DB    |
  +--------+     +-----------+     | (Rails API)    |     | (sharded)   |
                                          |               +-------------+
                                   +------v-------+       +-------------+
                                   | Fan-out Svc  |       | Timeline    |
                                   | (Sidekiq)    |       | Cache       |
                                   +------+-------+       | (Redis      |
                                          |               |  sorted set)|
                                   +------v-------+       +-------------+
                                   | Kafka        |
                                   +--------------+
```

### Key Design: Hybrid Fan-out

| User Type | Strategy | Why |
|-----------|----------|-----|
| Regular users (< 10K followers) | Fan-out on write (Sidekiq pushes to Redis) | Pre-compute timelines; fast reads |
| Celebrities (> 10K followers) | Fan-out on read | Pushing to millions is too slow |

> **Ruby Context:**

```ruby
# Fan-out on write with Sidekiq
class FanOutWorker
  include Sidekiq::Job
  sidekiq_options queue: :fan_out

  CELEBRITY_THRESHOLD = 10_000

  def perform(tweet_id, author_id)
    return if User.find(author_id).follower_count >= CELEBRITY_THRESHOLD

    Follow.where(followee_id: author_id).find_each(batch_size: 1000) do |follow|
      Redis.current.zadd("timeline:#{follow.follower_id}", Time.now.to_f, tweet_id)
      Redis.current.zremrangebyrank("timeline:#{follow.follower_id}", 0, -801)
    end
  end
end

# Timeline read — merge pre-computed + celebrity tweets
class TimelineService
  def self.fetch(user_id, limit: 50)
    # Pre-computed entries from Redis
    ids = Redis.current.zrevrange("timeline:#{user_id}", 0, limit - 1)

    # Celebrity tweets fetched on-read
    celeb_ids = Follow.joins(:followee)
                      .where(follower_id: user_id)
                      .where('users.follower_count >= ?', 10_000)
                      .pluck(:followee_id)
    if celeb_ids.any?
      celeb_tweet_ids = Tweet.where(user_id: celeb_ids)
                             .order(created_at: :desc).limit(limit).pluck(:id)
      ids = (ids.map(&:to_i) + celeb_tweet_ids).uniq
    end

    Tweet.where(id: ids).order(created_at: :desc).limit(limit)
  end
end
```

---

## 34.2 Design Instagram / Photo Sharing

### Requirements & Scale

**Scale:** 500M DAU, 100M photos/day, 200 TB/day storage, 116K reads/sec.

### Key Design: Pre-Signed Upload + Async Processing

```
1. Client → Rails API: "I want to upload"
2. Rails → generates pre-signed S3 URL (aws-sdk-s3)
3. Client → S3: upload directly (bypasses Rails!)
4. S3 event → Sidekiq/Lambda → Image Processing
5. Generate thumbnails (150×150, 320×320, 640×640, 1080×1080)
6. Publish "PhotoUploaded" event → fan-out to followers' feeds
```

> **Ruby Context:**

```ruby
# Pre-signed URL generation
class UploadsController < ApplicationController
  def presign
    key = "photos/#{current_user.id}/#{SecureRandom.uuid}.jpg"
    url = Aws::S3::Presigner.new.presigned_url(:put_object,
      bucket: ENV['S3_BUCKET'], key: key, expires_in: 900, content_type: 'image/jpeg')
    render json: { upload_url: url, key: key }
  end
end

# Async image processing after upload
class ImageProcessingWorker
  include Sidekiq::Job

  SIZES = { thumbnail: [150, 150], small: [320, 320], medium: [640, 640], large: [1080, 1080] }

  def perform(photo_id)
    photo = Photo.find(photo_id)
    original = download_from_s3(photo.s3_key)

    SIZES.each do |name, (w, h)|
      resized = ImageProcessing::MiniMagick.source(original).resize_to_fill(w, h).call
      upload_to_s3("#{photo.s3_key}_#{name}", resized)
    end

    photo.update!(processed: true)
    FeedFanOutWorker.perform_async(photo_id, photo.user_id)
  end
end
```

**Storage:** S3 (photos) + CloudFront CDN + PostgreSQL (metadata) + Redis (feed cache) + Elasticsearch (search by hashtag/location).

---

## 34.3 Design WhatsApp / Chat System

### Requirements & Scale

**Scale:** 500M DAU, 1.16M messages/sec, 50M concurrent WebSocket connections.

### Architecture

```
  +--------+     +------------------+     +------------------+
  | Client | ←─→ | WebSocket Gateway| ←─→ | Chat Service     |
  +--------+ WS  | (AnyCable)       |     | (Rails API)      |
                 +--------+---------+     +--------+---------+
                          |                        |
                 +--------v---------+     +--------v---------+
                 | Connection       |     | Kafka            |
                 | Registry (Redis) |     +--------+---------+
                 +------------------+              |
                                          +--------v---------+
                 +------------------+     | Message Store    |
                 | Presence Service |     | (Cassandra)      |
                 +------------------+     +------------------+
```

### Key Design: Message Delivery

```
Alice sends to Bob:
  Online:  Alice → WS Gateway → Chat Service → Redis lookup → WS Gateway B → Bob
  Offline: Alice → Chat Service → Kafka → Cassandra + Push Notification
           Bob comes online → fetch undelivered from Cassandra
```

> **Ruby Context:** Use **AnyCable** (Go-based WebSocket server) instead of Action Cable for high connection counts. AnyCable handles 10-50x more connections per server than pure Ruby Action Cable.

```ruby
# AnyCable channel for chat
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chat:#{current_user.id}"
    Redis.current.hset('connections', current_user.id, AnyCable.server_id)
  end

  def receive(data)
    ChatService.deliver(
      sender_id: current_user.id,
      recipient_id: data['to'],
      content: data['content']
    )
  end

  def unsubscribed
    Redis.current.hdel('connections', current_user.id)
  end
end

class ChatService
  def self.deliver(sender_id:, recipient_id:, content:)
    message = persist_message(sender_id, recipient_id, content)

    server_id = Redis.current.hget('connections', recipient_id)
    if server_id  # Online — deliver via WebSocket
      ActionCable.server.broadcast("chat:#{recipient_id}", message.as_json)
    else          # Offline — push notification
      PushNotificationWorker.perform_async(recipient_id, message.id)
    end
  end
end
```

---

## 34.4 Design Notification System

### Requirements & Scale

**Scale:** 10B notifications/day, 100M users, 3 channels (push, SMS, email).

### Architecture

```
  Trigger Sources → Notification Service → Priority Queues (Kafka) → Channel Workers
                    (validate, rate limit,    (urgent/high/normal)    (Push/SMS/Email)
                     template, dedup)
```

> **Ruby Context:**

```ruby
# Notification service with Sidekiq priority queues
class NotificationService
  def self.send(user_id:, type:, template:, data:, priority: :normal, channels: [:push])
    user = User.find(user_id)
    return unless user.notification_enabled?(type)  # Check preferences
    return if rate_limited?(user_id, type)          # Rate limit check
    return if duplicate?(user_id, type, data)       # Dedup check

    rendered = render_template(template, data)

    channels.each do |channel|
      queue = "notifications_#{priority}"
      case channel
      when :push  then PushWorker.set(queue: queue).perform_async(user_id, rendered)
      when :sms   then SmsWorker.set(queue: queue).perform_async(user_id, rendered)
      when :email then EmailWorker.set(queue: queue).perform_async(user_id, rendered)
      end
    end
  end
end

# Sidekiq processes urgent queue first
# config/sidekiq.yml:
#   :queues:
#     - [notifications_urgent, 10]
#     - [notifications_high, 5]
#     - [notifications_normal, 2]
#     - [notifications_low, 1]
```

**Key decisions:** Priority queues (Sidekiq weighted queues or Kafka topics), per-user rate limiting (Rack::Attack or Redis counters), DLQ for failures, user preference checks before sending.

---

## 34.5 Design a Web Crawler

### Requirements & Scale

**Scale:** 1B pages/month (~400 pages/sec), 17 TB/day storage.

### Architecture

```
  URL Frontier → Fetcher → Parser → Dedup (Bloom filter) → Content Store (S3)
  (priority +     (HTTP,     (extract    (URL + content       
   politeness)    robots.txt) links)      fingerprint)
       ↑                                      |
       +──────── new URLs (not yet crawled) ──+
```

**Key decisions:**
- **URL Frontier:** Priority queues (important domains first) + politeness queues (one per domain, 1 req/sec max)
- **Dedup:** Bloom filter for URL dedup; SimHash for content dedup
- **Politeness:** Parse `robots.txt`, respect `Crawl-delay`, identify as crawler in User-Agent

> **Ruby Context:** While crawlers are typically built in Go/Java for performance, Ruby can be used for smaller-scale crawlers with `mechanize`, `nokogiri` (HTML parsing), and `bloomfilter-rb` (dedup). For production scale, Ruby orchestrates the pipeline (Sidekiq scheduling, S3 storage) while fetching is done by faster services.

---

## 34.6 Design Search Autocomplete / Typeahead

### Requirements & Scale

**Scale:** 10K QPS, < 100ms latency, 5B queries/day for data collection.

### Key Design: Trie with Pre-computed Top-K

```
Each trie node stores pre-computed top 10 suggestions.
User types "sys" → O(1) lookup → return pre-computed list instantly.

Data Pipeline (offline):
  Search logs → Kafka → Spark (aggregate frequencies hourly) → rebuild Trie → distribute
  
Hot prefixes (top 1000, 1-3 chars) cached in Redis for 80%+ of queries.
```

> **Ruby Context:**

```ruby
# Autocomplete with Redis sorted sets (simple approach)
class AutocompleteService
  def self.suggest(prefix, limit: 10)
    # Hot prefixes cached in Redis sorted set (scored by frequency)
    results = Redis.current.zrevrange("autocomplete:#{prefix.downcase}", 0, limit - 1)
    return results if results.any?

    # Fallback: query Elasticsearch prefix search
    SearchQuery.search(prefix, fields: [:query_text], match_type: :phrase_prefix, limit: limit)
  end

  # Offline job: rebuild autocomplete index from search logs
  def self.rebuild_index
    SearchLog.group(:query).where('created_at > ?', 7.days.ago)
             .order('count_all DESC').count.each do |query, count|
      # Index every prefix of this query
      (1..query.length).each do |i|
        prefix = query[0...i].downcase
        Redis.current.zadd("autocomplete:#{prefix}", count, query)
        Redis.current.zremrangebyrank("autocomplete:#{prefix}", 0, -11) # Keep top 10
      end
    end
  end
end
```

---

## 34.7 Design a Distributed Cache

### Requirements & Scale

**Scale:** 10M ops/sec, 1 TB capacity, < 1ms P99, 99.99% availability.

### Key Design Decisions

| Component | Design | Why |
|-----------|--------|-----|
| **Partitioning** | Consistent hashing (16,384 slots) | Even distribution; minimal redistribution |
| **Replication** | Async primary → replica | HA without write latency penalty |
| **Eviction** | Approximate LRU (sample 5 keys) | O(1); Redis's actual approach |
| **Failover** | Replica promotion (Sentinel/Cluster) | Automatic HA |
| **Client routing** | Smart client caches slot→node mapping | Avoid proxy hop |

**Hot Key Solutions:**
1. **L1 local cache** in Rails process (`ActiveSupport::Cache::MemoryStore` for hot keys)
2. **Key splitting:** `hot_key:shard:0` through `hot_key:shard:9` on different nodes
3. **Read from replicas** for hot read-only keys

> **Ruby Context:** Use `redis-clustering` gem for Redis Cluster. For L1 cache, Rails multi-level caching:

```ruby
# Multi-level cache: L1 (memory) + L2 (Redis)
config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }

# Hot key with local memory cache (avoids Redis round-trip)
def get_hot_config(key)
  Rails.cache.fetch("config:#{key}", expires_in: 30.seconds, race_condition_ttl: 5.seconds) do
    # Only hits Redis on cache miss
    Config.find_by(key: key)&.value
  end
end
```

---

## 34.8 Design an API Rate Limiter (Distributed)

### Key Differences from 33.3

Multiple API gateway instances share state via Redis Cluster. Tiered limits (Free: 100/min, Pro: 1000/min, Enterprise: 10000/min).

**Distributed Challenges & Solutions:**

| Challenge | Solution |
|-----------|---------|
| Race conditions | Redis Lua scripts (atomic) |
| Clock skew | Use Redis server time, not client |
| Redis failure | Fail open (allow) or fail closed (reject) — configurable |
| Multi-region | Local Redis per region (approximate) |

> **Ruby Context:** See Module 33.3 for Rack::Attack and custom Lua script implementations. For tiered limits:

```ruby
# Tiered rate limiting based on user plan
class Rack::Attack
  throttle('api/tiered', limit: proc { |req|
    user = User.find_by(api_key: req.env['HTTP_X_API_KEY'])
    case user&.plan
    when 'enterprise' then 10_000
    when 'pro'        then 1_000
    else                   100
    end
  }, period: 60) do |req|
    req.env['HTTP_X_API_KEY'] if req.path.start_with?('/api/')
  end
end
```

---

## 34.9 Design Uber / Ride Sharing

### Requirements & Scale

**Scale:** 20M rides/day, 5M concurrent drivers, 1.25M location updates/sec.

### Key Design: Geospatial Matching

```
Rider requests ride → Redis GEOSEARCH within 5 km → filter available drivers → rank by ETA → offer to closest
```

> **Ruby Context:**

```ruby
# Driver location tracking with Redis GEO
class LocationService
  def self.update_driver(driver_id, lat, lng)
    Redis.current.geoadd('drivers:available', lng, lat, driver_id)
  end

  def self.find_nearby_drivers(lat, lng, radius_km: 5, limit: 10)
    Redis.current.geosearch(
      'drivers:available', 'FROMLONLAT', lng, lat,
      'BYRADIUS', radius_km, 'km', 'ASC', 'COUNT', limit, 'WITHDIST'
    )
  end

  def self.remove_driver(driver_id)
    Redis.current.zrem('drivers:available', driver_id)
  end
end

# Matching service
class MatchingService
  def self.match_rider(rider_id, pickup_lat, pickup_lng)
    nearby = LocationService.find_nearby_drivers(pickup_lat, pickup_lng)
    nearby.each do |driver_id, distance|
      driver = Driver.find(driver_id)
      next unless driver.available? && driver.vehicle_matches?(ride_type)

      # Offer ride to driver (15 sec timeout)
      RideOfferWorker.perform_async(driver_id, rider_id)
      return driver_id
    end
    nil # No drivers available — expand radius or queue
  end
end
```

**Surge Pricing:** Divide city into H3 hexagonal cells. Every minute: `surge = demand / supply` per cell. Cap at 3-5x.

**Real-time tracking:** Driver location updates via WebSocket (AnyCable) pushed to rider's app.

---

## 34.10 Design a Hotel / Flight Booking System

### Requirements & Scale

**Scale:** 1M bookings/day, 50M searches/day, 100M room-nights to manage.

### Key Design: Preventing Double Booking

> **Ruby Context:** ActiveRecord supports both pessimistic and optimistic locking natively.

```ruby
# Pessimistic locking (SELECT FOR UPDATE) — prevents double booking
class BookingService
  def self.book(user_id:, hotel_id:, room_type:, check_in:, check_out:)
    ActiveRecord::Base.transaction do
      dates = (check_in...check_out).to_a

      dates.each do |date|
        inventory = RoomInventory.lock('FOR UPDATE')  # Pessimistic lock!
          .find_by!(hotel_id: hotel_id, room_type: room_type, date: date)

        raise SoldOutError if inventory.available_count <= 0
        inventory.decrement!(:available_count)
      end

      Booking.create!(
        user_id: user_id, hotel_id: hotel_id, room_type: room_type,
        check_in: check_in, check_out: check_out, status: :confirmed
      )
    end
  end
end

# Optimistic locking alternative (lock_version column)
class RoomInventory < ApplicationRecord
  # Requires `lock_version` integer column in the table
  # ActiveRecord automatically uses optimistic locking when this column exists

  def reserve!
    with_lock do  # or use optimistic: decrement + check lock_version
      raise SoldOutError if available_count <= 0
      decrement!(:available_count)
    end
  rescue ActiveRecord::StaleObjectError
    retry # Someone else booked — retry (will fail if sold out)
  end
end

# Temporary hold with expiry (Sidekiq scheduled job releases it)
class HoldService
  def self.create_hold(user_id:, hotel_id:, room_type:, date:)
    ActiveRecord::Base.transaction do
      inventory = RoomInventory.lock.find_by!(hotel_id: hotel_id, room_type: room_type, date: date)
      raise SoldOutError if inventory.available_count <= 0

      inventory.increment!(:held_count)
      hold = Hold.create!(user_id: user_id, hotel_id: hotel_id, room_type: room_type,
                          date: date, expires_at: 10.minutes.from_now)

      # Schedule release if not converted to booking
      ReleaseHoldWorker.perform_at(hold.expires_at, hold.id)
      hold
    end
  end
end
```

**Search:** Elasticsearch (via `searchkick` gem) for hotel search with filters (location, dates, price, guests). Availability pre-indexed, real-time check only at booking time.

---

## Summary & Key Takeaways

| Problem | Key Design Decisions | Ruby Tools |
|---------|---------------------|-----------|
| **Twitter/Feed** | Hybrid fan-out; Redis sorted sets for timeline | Sidekiq (fan-out), Redis, Karafka |
| **Instagram** | Pre-signed S3 upload; async image processing; CDN | aws-sdk-s3, ImageProcessing gem, Sidekiq |
| **WhatsApp/Chat** | WebSocket + Connection Registry; Cassandra storage | AnyCable, Redis, Cassandra driver |
| **Notifications** | Priority queues; channel workers; rate limiting; DLQ | Sidekiq (weighted queues), APNs/FCM gems |
| **Web Crawler** | URL Frontier; Bloom filter dedup; politeness | Nokogiri, bloomfilter-rb, Sidekiq |
| **Autocomplete** | Trie with pre-computed top-K; Redis cache | Redis sorted sets, Searchkick |
| **Distributed Cache** | Consistent hashing; LRU; hot key mitigation | redis-clustering, MemoryStore (L1) |
| **Rate Limiter** | Redis Lua scripts; tiered limits; fail open/closed | Rack::Attack, custom Lua scripts |
| **Uber** | Redis GEO; geospatial matching; surge pricing | Redis GEOSEARCH, AnyCable, Sidekiq |
| **Hotel Booking** | Pessimistic locking; temporary holds; Elasticsearch | ActiveRecord lock/transaction, Searchkick |

---

## Interview Tips for Module 34

1. **Twitter** — explain hybrid fan-out; show Sidekiq worker pushing to Redis sorted sets
2. **Instagram** — pre-signed URLs bypass app servers; async image processing pipeline
3. **WhatsApp** — AnyCable for high connection counts; Connection Registry in Redis; online vs offline delivery
4. **Notifications** — Sidekiq weighted queues for priority; per-user rate limiting; DLQ
5. **Web Crawler** — URL Frontier (priority + politeness); Bloom filter; robots.txt
6. **Autocomplete** — Redis sorted sets for hot prefixes; offline rebuild from search logs
7. **Distributed Cache** — consistent hashing; hot key solutions (L1 cache, key splitting)
8. **Uber** — Redis GEO commands (`GEOADD`, `GEOSEARCH`); matching algorithm
9. **Hotel Booking** — ActiveRecord `lock('FOR UPDATE')` for pessimistic locking; temporary holds with Sidekiq scheduled release
10. **Always mention** — caching, monitoring, failure handling, scaling for every problem