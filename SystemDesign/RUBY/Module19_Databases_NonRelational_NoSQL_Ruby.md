# Module 19: Databases — Non-Relational (NoSQL)

> Not all data fits neatly into tables with rows and columns. NoSQL databases emerged to handle use cases where relational databases struggle — massive scale, flexible schemas, high write throughput, graph relationships, and real-time search. This module covers the major NoSQL categories, their data models, internal architectures, and most importantly, when to choose each one over a relational database.

> **Ruby Context:** Ruby has a rich ecosystem of gems for interacting with NoSQL databases: `redis` (redis-rb) for Redis, `mongoid` or `mongo` for MongoDB, `cassandra-driver` for Cassandra, `neo4j` (neo4j-ruby-driver or activegraph) for Neo4j, `aws-sdk-dynamodb` for DynamoDB, and `elasticsearch-ruby` for Elasticsearch. Rails applications commonly use these alongside ActiveRecord for polyglot persistence.

---

## 19.1 NoSQL Categories

> **NoSQL** ("Not Only SQL") is an umbrella term for databases that don't use the traditional relational table model. NoSQL databases are designed for specific access patterns and trade-offs — there is no single "NoSQL database" that does everything. Each category excels at a different type of workload.

---

### Overview of NoSQL Categories

```
+-------------------+------------------+-------------------+------------------+
|   Key-Value       |   Document       |   Column-Family   |   Graph          |
|   Stores          |   Stores         |   Stores          |   Databases      |
+-------------------+------------------+-------------------+------------------+
| key → value       | key → JSON doc   | row key →         | nodes + edges    |
|                   |                  |   column families  |   + properties   |
| Redis             | MongoDB          | Cassandra          | Neo4j            |
| DynamoDB          | CouchDB          | HBase              | Amazon Neptune   |
| Memcached         | Firestore        | ScyllaDB           | ArangoDB         |
+-------------------+------------------+-------------------+------------------+

+-------------------+------------------+
|   Time-Series     |   Search Engines |
|   Databases       |                  |
+-------------------+------------------+
| timestamp → value | document → index |
|                   |   (inverted)     |
| InfluxDB          | Elasticsearch    |
| TimescaleDB       | Solr             |
| Prometheus        | Meilisearch      |
+-------------------+------------------+
```

---

### Category Comparison

| Category | Data Model | Query Pattern | Strengths | Weaknesses |
|----------|-----------|---------------|-----------|-----------|
| **Key-Value** | Simple key → value pairs | Get/Set by key | Extreme speed, simplicity, caching | No complex queries, no relationships |
| **Document** | JSON/BSON documents | Query by any field, nested data | Flexible schema, rich queries | Joins are expensive, denormalization needed |
| **Column-Family** | Row key → column families | Range scans by row key | Massive write throughput, horizontal scale | Limited query flexibility, no joins |
| **Graph** | Nodes + edges + properties | Traverse relationships | Relationship queries are fast | Not great for bulk analytics, smaller ecosystem |
| **Time-Series** | Timestamp → measurements | Time-range queries, aggregations | Optimized for time-ordered data, compression | Limited to time-series patterns |
| **Search Engine** | Documents → inverted index | Full-text search, faceted search | Fast text search, relevance ranking | Not a primary database, eventual consistency |

---

### When to Use Each Category

| Use Case | Best Category | Why |
|----------|--------------|-----|
| Caching, session storage | Key-Value (Redis) | Sub-millisecond reads, simple get/set |
| User profiles, product catalogs | Document (MongoDB) | Flexible schema, nested data, rich queries |
| IoT sensor data, metrics, logs | Column-Family (Cassandra) or Time-Series (InfluxDB) | High write throughput, time-range queries |
| Social networks, recommendations | Graph (Neo4j) | Relationship traversal is O(1) per hop |
| Full-text search, autocomplete | Search Engine (Elasticsearch) | Inverted index, relevance scoring |
| Shopping cart, feature flags | Key-Value (Redis, DynamoDB) | Simple access pattern, high availability |
| Content management, blog posts | Document (MongoDB, Firestore) | Flexible schema, embedded comments/tags |
| Financial transactions, banking | Relational (PostgreSQL) | ACID transactions, strong consistency |
| Reporting, analytics | Relational or Column-Family | Complex joins (SQL) or pre-aggregated data (NoSQL) |

---


## 19.2 Key-Value Stores

> A **key-value store** is the simplest NoSQL data model — every piece of data is stored as a key-value pair. You look up values by their key, like a giant hash map. Key-value stores are optimized for extreme speed and simplicity, making them ideal for caching, session storage, and real-time counters.

> **Ruby Context:** The `redis` gem (redis-rb) is the standard Ruby client for Redis. In Rails, `ActiveSupport::Cache::RedisCacheStore` provides built-in Redis caching, and `Sidekiq` uses Redis as its job queue backend. For DynamoDB, the `aws-sdk-dynamodb` gem or the `dynamoid` gem (an ActiveRecord-like ORM for DynamoDB) are commonly used.

---

### Data Model

```
+---------------------------+-------------------------------------------+
|          Key              |                Value                      |
+---------------------------+-------------------------------------------+
| "user:123"                | "{\"name\":\"Alice\",\"age\":30}"         |
| "session:abc-def-ghi"     | "{\"user_id\":123,\"expires\":16250...}"  |
| "cache:product:456"       | "<serialized product object>"             |
| "counter:page_views"      | "1847293"                                 |
| "rate_limit:user:123"     | "47"                                      |
+---------------------------+-------------------------------------------+
```

**Characteristics:**
- **Simple API:** `GET(key)`, `SET(key, value)`, `DELETE(key)` — that's the core
- **No schema:** Values can be strings, JSON, binary blobs, serialized objects — the database doesn't interpret them
- **No relationships:** No joins, no foreign keys, no references between keys
- **No secondary indexes:** You can only look up by the exact key (some key-value stores add limited secondary index support)
- **Blazing fast:** O(1) lookups by key, typically sub-millisecond latency

---

### Use Cases

| Use Case | Key Pattern | Value | Why Key-Value |
|----------|------------|-------|---------------|
| **Caching** | `cache:user:123` | Serialized user object | Sub-ms reads, TTL expiration |
| **Session storage** | `session:abc-def-ghi` | Session data (JSON) | Fast lookup by session ID, TTL |
| **Rate limiting** | `rate:user:123:minute:202503151430` | Request count | Atomic increment, TTL |
| **Shopping cart** | `cart:user:123` | Cart items (JSON) | Fast read/write, per-user isolation |
| **Feature flags** | `feature:dark_mode` | `"enabled"` or `"disabled"` | Simple get, instant propagation |
| **Leaderboards** | `leaderboard:game:1` | Sorted set of scores | Redis sorted sets — O(log n) rank queries |
| **Distributed locks** | `lock:order:456` | Lock holder ID + expiry | Atomic SET with NX (set if not exists) |
| **Pub/Sub messaging** | `channel:notifications` | Message stream | Redis Pub/Sub or Streams |

---

### Redis — Deep Dive

**Redis** (Remote Dictionary Server) is the most popular key-value store. It's an **in-memory** data structure server that supports rich data types beyond simple strings.

**Redis Data Structures:**

| Data Structure | Description | Commands | Use Case |
|---------------|-------------|----------|----------|
| **String** | Binary-safe string (up to 512 MB) | `SET`, `GET`, `INCR`, `DECR`, `APPEND`, `SETEX` | Caching, counters, flags |
| **List** | Doubly linked list of strings | `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `LRANGE`, `LLEN` | Message queues, activity feeds, recent items |
| **Set** | Unordered collection of unique strings | `SADD`, `SREM`, `SMEMBERS`, `SISMEMBER`, `SINTER`, `SUNION` | Tags, unique visitors, mutual friends |
| **Sorted Set (ZSet)** | Set with a score for each member (ordered by score) | `ZADD`, `ZREM`, `ZRANGE`, `ZRANK`, `ZRANGEBYSCORE` | Leaderboards, priority queues, time-based feeds |
| **Hash** | Map of field-value pairs (like a mini object) | `HSET`, `HGET`, `HMSET`, `HGETALL`, `HINCRBY` | User profiles, object attributes, counters per field |
| **Stream** | Append-only log of entries with auto-generated IDs | `XADD`, `XREAD`, `XRANGE`, `XGROUP`, `XACK` | Event sourcing, message queues, activity logs |
| **Bitmap** | Bit array operations on strings | `SETBIT`, `GETBIT`, `BITCOUNT`, `BITOP` | Daily active users, feature flags per user |
| **HyperLogLog** | Probabilistic cardinality estimation | `PFADD`, `PFCOUNT`, `PFMERGE` | Unique visitor counting (approximate) |
| **Geospatial** | Latitude/longitude with distance queries | `GEOADD`, `GEODIST`, `GEORADIUS`, `GEOSEARCH` | Nearby restaurants, driver matching |

**Redis Data Structure Examples (using redis-rb gem):**

```ruby
require 'redis'
require 'json'

redis = Redis.new(host: 'localhost', port: 6379)

# String: Simple caching with TTL
redis.set('cache:user:123', { name: 'Alice', age: 30 }.to_json, ex: 3600)  # expires in 1 hour
user_data = JSON.parse(redis.get('cache:user:123'))                         # returns parsed hash

# String: Atomic counter
redis.incr('page_views:homepage')          # atomically increment by 1 → returns new value
redis.incrby('page_views:homepage', 10)    # atomically increment by 10

# Hash: User profile (access individual fields without deserializing)
redis.hset('user:123', 'name', 'Alice', 'age', '30', 'email', 'alice@example.com')
redis.hget('user:123', 'name')             # returns "Alice" (no need to fetch entire object)
redis.hincrby('user:123', 'age', 1)        # atomically increment age

# Sorted Set: Leaderboard
redis.zadd('leaderboard', 1500, 'player:alice')
redis.zadd('leaderboard', 2300, 'player:bob')
redis.zadd('leaderboard', 1800, 'player:charlie')
redis.zrevrange('leaderboard', 0, 9)                    # top 10 players (highest score first)
redis.zrank('leaderboard', 'player:bob')                 # bob's rank (0-indexed)
redis.zrangebyscore('leaderboard', 1000, 2000)           # players with score between 1000-2000

# Set: Mutual friends
redis.sadd('friends:alice', ['bob', 'charlie', 'dave'])
redis.sadd('friends:bob', ['alice', 'charlie', 'eve'])
redis.sinter('friends:alice', 'friends:bob')             # mutual friends → ["charlie"]

# List: Recent activity feed (capped at 100 items)
redis.lpush('feed:user:123', { action: 'liked', post: 456 }.to_json)
redis.ltrim('feed:user:123', 0, 99)         # keep only the 100 most recent items
redis.lrange('feed:user:123', 0, 9)          # get the 10 most recent items

# Stream: Event log
redis.xadd('events:orders', { action: 'created', order_id: '789', user_id: '123' })
redis.xadd('events:orders', { action: 'paid', order_id: '789', amount: '99.99' })
redis.xrange('events:orders', '-', '+')      # read all events
```

---

### Redis Persistence (RDB, AOF)

Redis is an in-memory database, but it offers two persistence mechanisms to survive restarts:

**RDB (Redis Database Backup) — Point-in-Time Snapshots:**

```
How it works:
  1. Redis forks the process (copy-on-write)
  2. Child process writes the entire dataset to a binary .rdb file
  3. When complete, the new .rdb file replaces the old one

Configuration:
  save 900 1      -- snapshot if at least 1 key changed in 900 seconds
  save 300 10     -- snapshot if at least 10 keys changed in 300 seconds
  save 60 10000   -- snapshot if at least 10000 keys changed in 60 seconds
```

| Pros | Cons |
|------|------|
| Compact binary file — fast to load on restart | Data loss between snapshots (up to minutes) |
| Great for backups and disaster recovery | Fork can be slow for large datasets (memory spike) |
| Minimal performance impact during normal operation | Not suitable when you can't afford any data loss |

**AOF (Append-Only File) — Write Log:**

```
How it works:
  1. Every write command is appended to a log file
  2. On restart, Redis replays the log to reconstruct the dataset

Configuration:
  appendonly yes
  appendfsync always    -- fsync after every write (safest, slowest)
  appendfsync everysec  -- fsync every second (good balance — default)
  appendfsync no        -- let OS decide when to fsync (fastest, least safe)
```

| Pros | Cons |
|------|------|
| Minimal data loss (at most 1 second with `everysec`) | Larger file than RDB (stores every command) |
| Append-only — no corruption risk from partial writes | Slower restart (must replay all commands) |
| Human-readable log file | AOF rewrite needed periodically to compact the file |

**RDB vs AOF:**

| Feature | RDB | AOF |
|---------|-----|-----|
| Data loss risk | Minutes (between snapshots) | ~1 second (`everysec`) or zero (`always`) |
| File size | Compact (binary) | Larger (command log) |
| Restart speed | Fast (load binary) | Slower (replay commands) |
| Performance impact | Periodic fork (memory spike) | Continuous append (slight write overhead) |
| Best for | Backups, disaster recovery | Durability, minimal data loss |

**Recommendation:** Use **both** RDB + AOF. RDB for fast backups and disaster recovery. AOF for minimal data loss. On restart, Redis uses AOF (more complete) if available.

---

### Redis Cluster

**Redis Cluster** provides horizontal scaling by distributing data across multiple Redis nodes using **hash slots**.

```
Redis Cluster Architecture:

  +------------------+    +------------------+    +------------------+
  | Master A         |    | Master B         |    | Master C         |
  | Slots: 0-5460    |    | Slots: 5461-10922|    | Slots: 10923-16383|
  +--------+---------+    +--------+---------+    +--------+---------+
           |                       |                       |
  +--------+---------+    +--------+---------+    +--------+---------+
  | Replica A1       |    | Replica B1       |    | Replica C1       |
  +------------------+    +------------------+    +------------------+

Total: 16384 hash slots distributed across masters
Key assignment: slot = CRC16(key) % 16384
```

**How Redis Cluster Works:**
1. The key space is divided into **16,384 hash slots**
2. Each master node owns a subset of slots
3. When a client sends a command, the cluster calculates `CRC16(key) % 16384` to determine which node owns that slot
4. If the client connects to the wrong node, it receives a `MOVED` redirect to the correct node
5. Smart clients cache the slot-to-node mapping to avoid redirects

> **Ruby Context:** The `redis-clustering` gem (or `redis` gem v5+ with cluster mode) supports Redis Cluster natively. You can connect with `Redis.new(cluster: ['redis://node1:6379', 'redis://node2:6379'])`. The client automatically handles slot mapping and `MOVED` redirects.

**Cluster Features:**
- **Automatic failover:** If a master dies, its replica is promoted automatically
- **Resharding:** Slots can be moved between nodes to rebalance (online, no downtime)
- **Multi-key operations:** Only supported if all keys hash to the same slot (use hash tags: `{user:123}:profile` and `{user:123}:orders` hash to the same slot)

**Redis Cluster Limitations:**
- Multi-key operations across different slots are not atomic
- No support for `SELECT` (multiple databases) — only database 0
- Lua scripts must operate on keys in the same slot
- Larger cluster = more gossip protocol overhead

---

### Other Key-Value Stores

**Amazon DynamoDB:**

| Feature | Details |
|---------|---------|
| **Type** | Managed key-value + document store (AWS) |
| **Data model** | Partition key + optional sort key → attributes (JSON-like) |
| **Scaling** | Automatic horizontal scaling, on-demand or provisioned capacity |
| **Consistency** | Eventually consistent (default) or strongly consistent reads |
| **Global tables** | Multi-region, multi-active replication |
| **Streams** | Change data capture (CDC) for event-driven architectures |
| **DAX** | In-memory caching layer (microsecond reads) |
| **Best for** | Serverless apps, high-scale key-value access, AWS-native architectures |

> **Ruby Context:** Use the `aws-sdk-dynamodb` gem for direct DynamoDB access, or the `dynamoid` gem which provides an ActiveRecord-like ORM interface with familiar methods like `find`, `where`, and `create`. Dynamoid supports auto-scaling, global secondary indexes, and integrates well with Rails.

**DynamoDB Data Model:**

```
Table: Orders
  Partition Key: user_id (determines which partition stores the item)
  Sort Key: order_id (orders items within a partition)

  +----------+----------+--------+--------+---------------------+
  | user_id  | order_id | total  | status | created_at          |
  | (PK)     | (SK)     |        |        |                     |
  +----------+----------+--------+--------+---------------------+
  | user_123 | ord_001  |  99.99 | active | 2025-01-15T10:30:00 |
  | user_123 | ord_002  |  49.99 | shipped| 2025-02-20T14:45:00 |
  | user_456 | ord_003  | 149.99 | active | 2025-03-10T09:15:00 |
  +----------+----------+--------+--------+---------------------+

Queries:
  GET item:     user_id = "user_123" AND order_id = "ord_001"     → single item
  Query range:  user_id = "user_123" AND order_id BEGINS_WITH "ord"  → all orders for user
  Scan:         Reads entire table (expensive — avoid in production)
```

**DynamoDB Access in Ruby:**

```ruby
require 'aws-sdk-dynamodb'

dynamodb = Aws::DynamoDB::Client.new(region: 'us-east-1')

# Get a single item
result = dynamodb.get_item(
  table_name: 'Orders',
  key: { 'user_id' => 'user_123', 'order_id' => 'ord_001' }
)
puts result.item  # => {"user_id"=>"user_123", "order_id"=>"ord_001", "total"=>99.99, ...}

# Query all orders for a user
result = dynamodb.query(
  table_name: 'Orders',
  key_condition_expression: 'user_id = :uid AND begins_with(order_id, :prefix)',
  expression_attribute_values: { ':uid' => 'user_123', ':prefix' => 'ord' }
)
result.items.each { |item| puts item }

# Put a new item
dynamodb.put_item(
  table_name: 'Orders',
  item: {
    'user_id'    => 'user_789',
    'order_id'   => 'ord_004',
    'total'      => 79.99,
    'status'     => 'active',
    'created_at' => Time.now.iso8601
  }
)
```

**DynamoDB Single-Table Design:**
In DynamoDB, the best practice is often to store multiple entity types in a **single table** using generic partition key and sort key names (PK, SK). This enables efficient access patterns without joins.

```
+------------------+------------------+----------+--------+
| PK               | SK               | name     | total  |
+------------------+------------------+----------+--------+
| USER#123         | PROFILE          | Alice    |        |
| USER#123         | ORDER#2025-01-15 |          |  99.99 |
| USER#123         | ORDER#2025-02-20 |          |  49.99 |
| PRODUCT#456      | METADATA         | Widget   |        |
| PRODUCT#456      | REVIEW#USER#123  |          |        |
+------------------+------------------+----------+--------+

Query: PK = "USER#123" AND SK BEGINS_WITH "ORDER#"
→ Returns all orders for user 123 (sorted by date)
```

**Memcached:**

| Feature | Details |
|---------|---------|
| **Type** | In-memory key-value cache (no persistence) |
| **Data model** | Simple key → string value |
| **Scaling** | Client-side sharding (consistent hashing) |
| **Persistence** | None — data is lost on restart |
| **Data structures** | Strings only (no lists, sets, sorted sets) |
| **Best for** | Simple caching when you don't need Redis's data structures |

> **Ruby Context:** The `dalli` gem is the standard Ruby client for Memcached. Rails integrates with it via `ActiveSupport::Cache::MemCacheStore`. However, most Ruby/Rails projects prefer Redis over Memcached due to Redis's richer data structures and the ubiquity of the `redis` gem.

**Redis vs Memcached:**

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Rich (strings, lists, sets, sorted sets, hashes, streams) | Strings only |
| Persistence | RDB + AOF | None |
| Replication | Master-replica | None (client-side) |
| Clustering | Redis Cluster (server-side) | Client-side sharding |
| Pub/Sub | Yes | No |
| Lua scripting | Yes | No |
| Memory efficiency | Good | Slightly better for simple strings (slab allocator) |
| Multi-threaded | Single-threaded (io-threads in 6.0+) | Multi-threaded |
| Use case | Feature-rich caching, data structures, messaging | Simple, high-throughput string caching |

**Key Point:** Redis has largely replaced Memcached for most use cases. Use Memcached only when you need the simplest possible cache with multi-threaded performance and don't need any of Redis's advanced features.

---


## 19.3 Document Stores

> A **document store** organizes data as semi-structured **documents** (typically JSON or BSON). Each document is a self-contained unit that can have a different structure from other documents in the same collection. This flexibility makes document stores ideal for applications with evolving schemas and nested data.

> **Ruby Context:** The `mongoid` gem is the most popular MongoDB ODM (Object-Document Mapper) for Ruby/Rails. It provides an ActiveRecord-like interface with model definitions, validations, callbacks, and associations. The lower-level `mongo` gem provides direct driver access. For Firestore, use the `google-cloud-firestore` gem.

---

### JSON/BSON Documents

**Document Structure:**

```json
// A single document in a "users" collection
{
  "_id": "user_123",
  "name": "Alice Smith",
  "email": "alice@example.com",
  "age": 30,
  "address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zip": "94102"
  },
  "tags": ["premium", "early-adopter"],
  "orders": [
    {
      "order_id": "ord_001",
      "total": 99.99,
      "items": [
        { "product_id": "prod_456", "name": "Widget", "quantity": 2 }
      ],
      "created_at": "2025-01-15T10:30:00Z"
    }
  ],
  "preferences": {
    "notifications": true,
    "theme": "dark",
    "language": "en"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2025-03-10T09:15:00Z"
}
```

**Key Characteristics:**
- Documents are **self-contained** — all related data can be embedded in one document
- **No fixed schema** — different documents in the same collection can have different fields
- **Nested data** — objects and arrays can be nested to any depth
- **Rich queries** — query by any field, including nested fields and array elements

**BSON (Binary JSON) — MongoDB's format:**
- Binary encoding of JSON — more compact and faster to parse
- Supports additional types: `Date`, `ObjectId`, `Decimal128`, `Binary`, `Regex`
- Maximum document size: **16 MB** (MongoDB)

---

### Schema Flexibility

Document stores are often called "schemaless," but a more accurate term is **schema-on-read** — the database doesn't enforce a schema, but your application code expects a certain structure.

**Schema-on-Write (SQL) vs Schema-on-Read (Document):**

| Aspect | Schema-on-Write (SQL) | Schema-on-Read (Document) |
|--------|----------------------|--------------------------|
| Schema definition | Defined upfront (CREATE TABLE) | Implicit in application code |
| Schema changes | ALTER TABLE (can be slow, may lock table) | Just start writing new fields |
| Data validation | Database enforces constraints | Application must validate |
| Consistency | All rows have the same structure | Documents can vary |
| Migration | Requires migration scripts | Gradual migration (handle old + new format) |
| Best for | Stable, well-understood data models | Rapidly evolving schemas, varied data |

> **Ruby Context:** Mongoid provides schema-like field definitions in Ruby models, giving you the best of both worlds — flexible MongoDB storage with Ruby-level type checking and validation:

```ruby
class User
  include Mongoid::Document
  include Mongoid::Timestamps  # adds created_at, updated_at

  field :name,  type: String
  field :email, type: String
  field :age,   type: Integer
  field :tags,  type: Array, default: []

  embeds_one  :address
  embeds_many :orders

  validates :name,  presence: true
  validates :email, presence: true, uniqueness: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :age,   numericality: { greater_than: 0, less_than: 150 }, allow_nil: true

  index({ email: 1 }, { unique: true })
  index({ tags: 1 })
end

class Address
  include Mongoid::Document

  field :street, type: String
  field :city,   type: String
  field :state,  type: String
  field :zip,    type: String

  embedded_in :user
end
```

**Schema Validation (MongoDB):**
Even in document stores, you can optionally enforce a schema:

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email"],
      properties: {
        name: { bsonType: "string", maxLength: 100 },
        email: { bsonType: "string", pattern: "^.+@.+\\..+$" },
        age: { bsonType: "int", minimum: 0, maximum: 150 }
      }
    }
  }
});
```

---

### Embedded Documents vs References

The most important design decision in document databases: **embed related data** inside the document or **reference** it with an ID (like a foreign key)?

**Embedding (Denormalized):**

```json
// Order document with embedded items and shipping address
{
  "_id": "ord_001",
  "user_id": "user_123",
  "user_name": "Alice Smith",
  "items": [
    { "product_id": "prod_456", "name": "Widget", "price": 29.99, "quantity": 2 },
    { "product_id": "prod_789", "name": "Gadget", "price": 49.99, "quantity": 1 }
  ],
  "shipping_address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA"
  },
  "total": 109.97
}
```

**Referencing (Normalized):**

```json
// Order document with references
{
  "_id": "ord_001",
  "user_id": "user_123",
  "item_ids": ["item_001", "item_002"],
  "shipping_address_id": "addr_001",
  "total": 109.97
}

// Separate items collection
{ "_id": "item_001", "product_id": "prod_456", "name": "Widget", "price": 29.99, "quantity": 2 }
{ "_id": "item_002", "product_id": "prod_789", "name": "Gadget", "price": 49.99, "quantity": 1 }
```

> **Ruby Context:** Mongoid supports both patterns with `embeds_one`/`embeds_many` (embedding) and `has_one`/`has_many`/`belongs_to` (referencing). Embedded documents use `embedded_in` on the child side.

```ruby
# Embedding: items stored inside the order document
class Order
  include Mongoid::Document
  field :user_id, type: String
  field :total,   type: Float
  embeds_many :items
  embeds_one  :shipping_address
end

class Item
  include Mongoid::Document
  field :product_id, type: String
  field :name,       type: String
  field :price,      type: Float
  field :quantity,    type: Integer
  embedded_in :order
end

# Referencing: items in a separate collection
class Order
  include Mongoid::Document
  field :total, type: Float
  has_many :items
  belongs_to :user
end

class Item
  include Mongoid::Document
  field :product_id, type: String
  field :name,       type: String
  field :price,      type: Float
  field :quantity,    type: Integer
  belongs_to :order
end
```

**When to Embed vs Reference:**

| Embed When | Reference When |
|-----------|---------------|
| Data is always accessed together | Data is accessed independently |
| One-to-few relationship (1 user → 3 addresses) | One-to-many with unbounded growth (1 user → millions of orders) |
| Data doesn't change frequently | Data changes frequently (update in one place) |
| Document stays under 16 MB | Embedded data would exceed document size limit |
| Read performance is critical | Write performance is critical (avoid updating large documents) |
| Data belongs to the parent (composition) | Data is shared across documents (many-to-many) |

**Rule of Thumb:**
- **Embed** for data that is read together and doesn't grow unboundedly
- **Reference** for data that is shared, grows unboundedly, or changes independently

---

### MongoDB Indexing

MongoDB supports rich indexing similar to relational databases:

| Index Type | Description | Example |
|-----------|-------------|---------|
| **Single field** | Index on one field | `db.users.createIndex({ email: 1 })` |
| **Compound** | Index on multiple fields (leftmost prefix rule applies) | `db.orders.createIndex({ user_id: 1, created_at: -1 })` |
| **Multikey** | Automatically indexes array elements | `db.users.createIndex({ tags: 1 })` — indexes each tag |
| **Text** | Full-text search index | `db.articles.createIndex({ title: "text", body: "text" })` |
| **Geospatial** | 2D/2DSphere indexes for location queries | `db.places.createIndex({ location: "2dsphere" })` |
| **Hashed** | Hash-based index for equality queries (used for sharding) | `db.users.createIndex({ user_id: "hashed" })` |
| **TTL** | Automatically deletes documents after a time period | `db.sessions.createIndex({ created_at: 1 }, { expireAfterSeconds: 3600 })` |
| **Partial** | Index only documents matching a filter | `db.orders.createIndex({ total: 1 }, { partialFilterExpression: { status: "active" } })` |
| **Unique** | Enforces uniqueness | `db.users.createIndex({ email: 1 }, { unique: true })` |

> **Ruby Context:** In Mongoid, indexes are declared in the model class and created with `rake db:mongoid:create_indexes`:

```ruby
class User
  include Mongoid::Document

  field :email, type: String
  field :tags,  type: Array

  index({ email: 1 }, { unique: true })                          # unique single field
  index({ tags: 1 })                                              # multikey (array)
  index({ created_at: 1 }, { expire_after_seconds: 3600 })       # TTL index
end

class Order
  include Mongoid::Document

  field :user_id,    type: String
  field :created_at, type: Time
  field :total,      type: Float
  field :status,     type: String

  index({ user_id: 1, created_at: -1 })                          # compound index
  index({ total: 1 }, { partial_filter_expression: { status: 'active' } })  # partial
end
```


---

### MongoDB Aggregation Pipeline

The **aggregation pipeline** is MongoDB's framework for data transformation and analysis — similar to SQL's GROUP BY, JOIN, and window functions.

```javascript
// Example: Top 5 customers by total spending in the last 30 days
db.orders.aggregate([
  // Stage 1: Filter orders from the last 30 days
  { $match: {
      created_at: { $gte: new Date(Date.now() - 30*24*60*60*1000) },
      status: "completed"
  }},

  // Stage 2: Group by user, calculate totals
  { $group: {
      _id: "$user_id",
      total_spent: { $sum: "$total" },
      order_count: { $sum: 1 },
      avg_order_value: { $avg: "$total" }
  }},

  // Stage 3: Sort by total spending (descending)
  { $sort: { total_spent: -1 } },

  // Stage 4: Limit to top 5
  { $limit: 5 },

  // Stage 5: Join with users collection to get names
  { $lookup: {
      from: "users",
      localField: "_id",
      foreignField: "_id",
      as: "user_info"
  }},

  // Stage 6: Flatten the joined array
  { $unwind: "$user_info" },

  // Stage 7: Project final output
  { $project: {
      _id: 0,
      user_name: "$user_info.name",
      total_spent: 1,
      order_count: 1,
      avg_order_value: { $round: ["$avg_order_value", 2] }
  }}
]);
```

> **Ruby Context:** The same aggregation pipeline can be executed using the `mongo` gem or Mongoid:

```ruby
# Using Mongoid's aggregation pipeline
pipeline = [
  { '$match' => {
      'created_at' => { '$gte' => 30.days.ago },
      'status' => 'completed'
  }},
  { '$group' => {
      '_id' => '$user_id',
      'total_spent' => { '$sum' => '$total' },
      'order_count' => { '$sum' => 1 },
      'avg_order_value' => { '$avg' => '$total' }
  }},
  { '$sort' => { 'total_spent' => -1 } },
  { '$limit' => 5 },
  { '$lookup' => {
      'from' => 'users',
      'localField' => '_id',
      'foreignField' => '_id',
      'as' => 'user_info'
  }},
  { '$unwind' => '$user_info' },
  { '$project' => {
      '_id' => 0,
      'user_name' => '$user_info.name',
      'total_spent' => 1,
      'order_count' => 1,
      'avg_order_value' => { '$round' => ['$avg_order_value', 2] }
  }}
]

results = Order.collection.aggregate(pipeline).to_a
results.each do |doc|
  puts "#{doc['user_name']}: $#{doc['total_spent']} (#{doc['order_count']} orders)"
end
```

**Common Aggregation Stages:**

| Stage | Purpose | SQL Equivalent |
|-------|---------|---------------|
| `$match` | Filter documents | `WHERE` |
| `$group` | Group and aggregate | `GROUP BY` + aggregate functions |
| `$sort` | Sort results | `ORDER BY` |
| `$limit` / `$skip` | Pagination | `LIMIT` / `OFFSET` |
| `$project` | Select/reshape fields | `SELECT` |
| `$lookup` | Join with another collection | `LEFT JOIN` |
| `$unwind` | Flatten arrays | Unnest |
| `$addFields` | Add computed fields | Computed columns |
| `$bucket` | Group into ranges | Histogram |
| `$facet` | Multiple aggregation pipelines in parallel | Multiple GROUP BYs |

---

### Use Cases for Document Stores

| Use Case | Why Document Store |
|----------|-------------------|
| **Content management** | Articles have varying structures (text, images, videos, metadata) |
| **Product catalogs** | Different product types have different attributes (electronics vs clothing) |
| **User profiles** | Flexible preferences, settings, nested addresses |
| **Event logging** | Events have varying payloads; schema evolves over time |
| **Mobile app backends** | JSON-native; flexible schema matches rapid mobile development |
| **Real-time analytics** | Aggregation pipeline for on-the-fly analysis |
| **Configuration storage** | Nested, hierarchical configuration data |

---


## 19.4 Column-Family Stores

> **Column-family stores** (also called wide-column stores) organize data into rows and column families, but unlike relational databases, each row can have a different set of columns. They are designed for **massive write throughput**, **horizontal scalability**, and **time-series data**. Apache Cassandra is the most widely used column-family store.

> **Ruby Context:** The `cassandra-driver` gem is the official DataStax Ruby driver for Apache Cassandra. It supports prepared statements, batch operations, automatic node discovery, load balancing policies, and retry policies. For higher-level ORM-like access, the `cequel` gem provides an ActiveRecord-inspired interface for Cassandra.

---

### Wide-Column Model

```
Traditional Row-Oriented Storage (SQL):
  Row 1: [id=1, name="Alice", age=30, email="alice@..."]
  Row 2: [id=2, name="Bob",   age=25, email="bob@..."]
  Row 3: [id=3, name="Charlie",age=35, email="charlie@..."]
  → All columns stored together per row

Wide-Column Storage (Cassandra):
  Row Key: "user_123"
    Column Family: "profile"
      name: "Alice"
      age: 30
      email: "alice@example.com"
    Column Family: "activity"
      last_login: "2025-03-15T10:30:00Z"
      page_views: 1847

  Row Key: "user_456"
    Column Family: "profile"
      name: "Bob"
      age: 25
      // no email column — sparse! Each row can have different columns
    Column Family: "activity"
      last_login: "2025-03-14T08:00:00Z"
```

**Key Concepts:**
- **Row Key (Partition Key):** Determines which node stores the data (like a hash key)
- **Column Family:** A group of related columns (like a table in SQL)
- **Column:** A name-value pair with a timestamp
- **Sparse:** Rows don't need to have the same columns — no storage wasted on NULL values
- **Sorted:** Columns within a row are sorted by name (or by clustering key in Cassandra)

---

### Partition Key and Clustering Key (Cassandra)

In Cassandra, the **primary key** has two parts:

```sql
CREATE TABLE orders (
    user_id UUID,           -- PARTITION KEY: determines which node stores this row
    order_date TIMESTAMP,   -- CLUSTERING KEY: determines sort order within the partition
    order_id UUID,          -- CLUSTERING KEY: further sorts within the same date
    total DECIMAL,
    status TEXT,
    PRIMARY KEY ((user_id), order_date, order_id)
);
--              ↑ partition key    ↑ clustering keys (sorted within partition)
```

**Partition Key:**
- Determines **which node** stores the data (hash of partition key → token → node)
- All rows with the same partition key are stored on the same node
- Queries MUST include the partition key (otherwise Cassandra must scan all nodes)

**Clustering Key:**
- Determines the **sort order** of rows within a partition
- Enables efficient range queries within a partition

```
Partition: user_id = "user_123"
  +---------------------+----------+--------+--------+
  | order_date          | order_id | total  | status |
  +---------------------+----------+--------+--------+
  | 2025-01-15 10:30:00 | ord_001  |  99.99 | active |  ← sorted by order_date
  | 2025-02-20 14:45:00 | ord_002  |  49.99 | shipped|
  | 2025-03-10 09:15:00 | ord_003  | 149.99 | active |
  +---------------------+----------+--------+--------+

Efficient queries:
  ✅ SELECT * FROM orders WHERE user_id = 'user_123';                    -- full partition
  ✅ SELECT * FROM orders WHERE user_id = 'user_123' AND order_date > '2025-02-01';  -- range within partition
  ❌ SELECT * FROM orders WHERE order_date > '2025-02-01';               -- no partition key! Full cluster scan!
  ❌ SELECT * FROM orders WHERE total > 100;                             -- not part of primary key!
```

**Key Rule:** In Cassandra, you **design the table around your queries**, not around your data model. Each query pattern may need its own table (denormalization is expected).

**Cassandra Access in Ruby:**

```ruby
require 'cassandra'

cluster = Cassandra.cluster(hosts: ['127.0.0.1'])
session = cluster.connect('my_keyspace')

# Create table
session.execute(<<~CQL)
  CREATE TABLE IF NOT EXISTS orders (
    user_id UUID,
    order_date TIMESTAMP,
    order_id UUID,
    total DECIMAL,
    status TEXT,
    PRIMARY KEY ((user_id), order_date, order_id)
  ) WITH CLUSTERING ORDER BY (order_date DESC, order_id ASC)
CQL

# Insert with prepared statement (recommended for performance)
insert = session.prepare(
  'INSERT INTO orders (user_id, order_date, order_id, total, status) VALUES (?, ?, ?, ?, ?)'
)
session.execute(insert, arguments: [
  Cassandra::Uuid.new('550e8400-e29b-41d4-a716-446655440000'),
  Time.now,
  Cassandra::TimeUuid::Generator.new.now,
  99.99,
  'active'
])

# Query all orders for a user
select = session.prepare('SELECT * FROM orders WHERE user_id = ?')
results = session.execute(select, arguments: [
  Cassandra::Uuid.new('550e8400-e29b-41d4-a716-446655440000')
])
results.each do |row|
  puts "Order #{row['order_id']}: $#{row['total']} (#{row['status']})"
end

# Range query within a partition
range_select = session.prepare(
  'SELECT * FROM orders WHERE user_id = ? AND order_date > ?'
)
results = session.execute(range_select, arguments: [
  Cassandra::Uuid.new('550e8400-e29b-41d4-a716-446655440000'),
  Time.new(2025, 2, 1)
])
```

---

### Cassandra Architecture

Cassandra uses a **peer-to-peer** architecture with no single point of failure — every node is equal.

```
Cassandra Ring (6 nodes):

         Node A (tokens: 0-16)
        /                      \
  Node F (tokens: 81-96)    Node B (tokens: 17-32)
       |                        |
  Node E (tokens: 65-80)    Node C (tokens: 33-48)
        \                      /
         Node D (tokens: 49-64)

Data placement (Replication Factor = 3):
  Key "user_123" → hash = 25 → primary: Node B
                                replica 1: Node C
                                replica 2: Node D
```

**Key Architecture Components:**

| Component | Description |
|-----------|-------------|
| **Ring** | Nodes arranged in a logical ring; each owns a range of tokens (hash values) |
| **Consistent Hashing** | Partition key is hashed to a token; token determines which node owns the data |
| **Gossip Protocol** | Nodes exchange state information (alive/dead, token ranges) every second |
| **Replication Factor (RF)** | Number of copies of each piece of data (typically RF=3) |
| **Snitch** | Determines which data center and rack a node belongs to (for replica placement) |
| **Coordinator** | Any node can be the coordinator for a request — routes to the correct replicas |

**Consistency Levels:**

| Level | Reads | Writes | Description |
|-------|-------|--------|-------------|
| `ONE` | 1 replica | 1 replica | Fastest, lowest consistency |
| `QUORUM` | ⌊RF/2⌋ + 1 replicas | ⌊RF/2⌋ + 1 replicas | Strong consistency if R + W > RF |
| `ALL` | All replicas | All replicas | Strongest consistency, lowest availability |
| `LOCAL_QUORUM` | Quorum in local DC | Quorum in local DC | Strong consistency within a data center |
| `EACH_QUORUM` | Quorum in each DC | Quorum in each DC | Strong consistency across all data centers |

**Strong Consistency Formula:** `R + W > RF` (Read consistency + Write consistency > Replication Factor)
- RF=3, R=QUORUM(2), W=QUORUM(2): 2+2=4 > 3 → **strongly consistent**
- RF=3, R=ONE(1), W=ONE(1): 1+1=2 < 3 → **eventually consistent**

> **Ruby Context:** The `cassandra-driver` gem supports consistency levels per query:

```ruby
# Set consistency level per query
session.execute(select, arguments: [user_id], consistency: :quorum)

# Or set default consistency for the session
cluster = Cassandra.cluster(
  hosts: ['127.0.0.1'],
  consistency: :local_quorum
)
```

---

### Write Path and Read Path

**Write Path:**

```
Client → Coordinator Node → Commit Log (append, sequential write — fast)
                          → Memtable (in-memory sorted structure)
                          → Acknowledge to client (write is durable)

Background (later):
  Memtable fills up → Flush to SSTable on disk (immutable, sorted file)
  Multiple SSTables accumulate → Compaction merges them into fewer, larger SSTables
```

```
1. Client sends write to coordinator
2. Coordinator forwards to replica nodes (based on RF)
3. Each replica:
   a. Appends to Commit Log (WAL — sequential write, ensures durability)
   b. Writes to Memtable (in-memory sorted map)
   c. Acknowledges to coordinator
4. Coordinator acknowledges to client (when enough replicas respond per consistency level)
5. Background: Memtable → SSTable flush → Compaction
```

**Why writes are fast:**
- Commit log is a sequential append (no random I/O)
- Memtable is in-memory (no disk I/O)
- No read-before-write (unlike B-Tree updates)
- Acknowledgment happens before data reaches disk (commit log ensures durability)

**Read Path:**

```
1. Client sends read to coordinator
2. Coordinator determines which replicas have the data
3. Coordinator sends read request to replicas (based on consistency level)
4. Each replica:
   a. Check Memtable (in-memory — fastest)
   b. Check Bloom Filters for each SSTable (quickly eliminate SSTables that don't contain the key)
   c. Check SSTable indexes → read data from disk
   d. Merge results from Memtable + SSTables (most recent timestamp wins)
5. Coordinator compares responses from replicas (read repair if inconsistent)
6. Return result to client
```

**Why reads can be slower than writes:**
- May need to check multiple SSTables (mitigated by Bloom filters and compaction)
- Read repair adds overhead when replicas are inconsistent
- Range scans may touch many SSTables

---

### Use Cases for Column-Family Stores

| Use Case | Why Column-Family |
|----------|------------------|
| **Time-series data** | Partition by device/sensor, cluster by timestamp — efficient range queries |
| **IoT sensor data** | Massive write throughput from millions of devices |
| **Event logging** | Append-only writes, time-ordered reads |
| **Messaging / chat history** | Partition by conversation, cluster by timestamp |
| **User activity tracking** | High write volume, time-ordered reads per user |
| **Recommendation data** | Pre-computed recommendations stored per user |
| **DNS data** | High read throughput, simple key-value access pattern |

**When NOT to use Cassandra:**
- Need complex joins or aggregations → use SQL
- Need strong consistency for all operations → use PostgreSQL
- Need ad-hoc queries on arbitrary columns → use SQL or Elasticsearch
- Small dataset (< 10 GB) → SQL is simpler and sufficient
- Need transactions across multiple partitions → use SQL

---


## 19.5 Graph Databases

> A **graph database** stores data as **nodes** (entities), **edges** (relationships), and **properties** (attributes). Unlike relational databases where relationship queries require expensive JOINs across multiple tables, graph databases traverse relationships in O(1) per hop — making them ideal for social networks, recommendation engines, and fraud detection.

> **Ruby Context:** For Neo4j, the `activegraph` gem (formerly `neo4j` gem) provides an ActiveRecord-like ORM for graph models in Ruby/Rails. The lower-level `neo4j-ruby-driver` gem provides direct Bolt protocol access. For Amazon Neptune, use the `aws-sdk-neptunedata` gem or Gremlin-based clients.

---

### Nodes, Edges, Properties

```
Graph Model:

  (Alice)---[:FRIENDS_WITH {since: 2020}]--->(Bob)
     |                                         |
     |                                         |
  [:LIKES]                                [:WORKS_AT]
     |                                         |
     v                                         v
  (Post: "Hello World")                    (Company: Google)
     ^                                         ^
     |                                         |
  [:WROTE]                                [:WORKS_AT {role: "SDE", since: 2022}]
     |                                         |
  (Alice)                                   (Charlie)
```

**Components:**

| Component | Description | Example |
|-----------|-------------|---------|
| **Node** | An entity (like a row in SQL) | Person, Product, Company, Post |
| **Edge (Relationship)** | A connection between two nodes (directed) | FRIENDS_WITH, PURCHASED, WORKS_AT |
| **Property** | Key-value attributes on nodes or edges | `name: "Alice"`, `since: 2020` |
| **Label** | A type/category for nodes | `:Person`, `:Product`, `:Company` |

**Key Difference from SQL:**
In SQL, relationships are implicit (foreign keys + JOINs). In a graph database, relationships are **first-class citizens** — they are stored explicitly and can have their own properties.

```
SQL: "Find friends of friends of Alice"
  SELECT DISTINCT f3.name
  FROM users u1
  JOIN friendships f1 ON u1.id = f1.user_id
  JOIN friendships f2 ON f1.friend_id = f2.user_id
  JOIN users f3 ON f2.friend_id = f3.id
  WHERE u1.name = 'Alice'
    AND f3.id != u1.id;
  -- Multiple JOINs, gets exponentially slower with more hops

Graph: "Find friends of friends of Alice"
  MATCH (alice:Person {name: "Alice"})-[:FRIENDS_WITH*2]->(fof:Person)
  WHERE fof <> alice
  RETURN DISTINCT fof.name;
  -- Constant time per hop, regardless of total graph size
```

---

### Cypher Query Language (Neo4j)

**Cypher** is Neo4j's declarative query language for graph databases. It uses ASCII art patterns to describe graph structures.

**Pattern Syntax:**
```
(node)                          -- a node
(node:Label)                    -- a node with a label
(node:Label {prop: "value"})    -- a node with properties
-[rel:TYPE]->                   -- a directed relationship
-[rel:TYPE {prop: "value"}]->   -- a relationship with properties
-[rel:TYPE*1..3]->              -- variable-length path (1 to 3 hops)
```

**CRUD Operations:**

```cypher
// CREATE nodes and relationships
CREATE (alice:Person {name: "Alice", age: 30})
CREATE (bob:Person {name: "Bob", age: 25})
CREATE (google:Company {name: "Google", founded: 1998})
CREATE (alice)-[:FRIENDS_WITH {since: 2020}]->(bob)
CREATE (bob)-[:WORKS_AT {role: "SDE", since: 2022}]->(google)

// READ: Find Alice's friends
MATCH (alice:Person {name: "Alice"})-[:FRIENDS_WITH]->(friend:Person)
RETURN friend.name, friend.age

// READ: Find friends of friends (2 hops)
MATCH (alice:Person {name: "Alice"})-[:FRIENDS_WITH*2]->(fof:Person)
WHERE fof.name <> "Alice"
RETURN DISTINCT fof.name

// READ: Shortest path between two people
MATCH path = shortestPath(
  (alice:Person {name: "Alice"})-[:FRIENDS_WITH*]-(bob:Person {name: "Bob"})
)
RETURN path, length(path)

// UPDATE
MATCH (alice:Person {name: "Alice"})
SET alice.age = 31, alice.updated_at = datetime()

// DELETE a relationship
MATCH (alice:Person {name: "Alice"})-[r:FRIENDS_WITH]->(bob:Person {name: "Bob"})
DELETE r

// DELETE a node (must delete relationships first)
MATCH (alice:Person {name: "Alice"})
DETACH DELETE alice  -- deletes node and all its relationships
```

> **Ruby Context:** Using the `activegraph` gem, you can define graph models with an ActiveRecord-like syntax and execute Cypher queries:

```ruby
# Model definitions with ActiveGraph (formerly Neo4j.rb)
class Person
  include ActiveGraph::Node

  property :name, type: String
  property :age,  type: Integer

  has_many :out, :friends, rel_class: :FriendsWith, model_class: :Person
  has_many :out, :employers, rel_class: :WorksAt, model_class: :Company
end

class Company
  include ActiveGraph::Node

  property :name,    type: String
  property :founded, type: Integer

  has_many :in, :employees, rel_class: :WorksAt, model_class: :Person
end

class FriendsWith
  include ActiveGraph::Relationship

  from_class :Person
  to_class   :Person
  type 'FRIENDS_WITH'

  property :since, type: Integer
end

class WorksAt
  include ActiveGraph::Relationship

  from_class :Person
  to_class   :Company
  type 'WORKS_AT'

  property :role,  type: String
  property :since, type: Integer
end

# CRUD operations
alice = Person.create(name: 'Alice', age: 30)
bob   = Person.create(name: 'Bob', age: 25)
google = Company.create(name: 'Google', founded: 1998)

# Create relationships
FriendsWith.create(from_node: alice, to_node: bob, since: 2020)
WorksAt.create(from_node: bob, to_node: google, role: 'SDE', since: 2022)

# Query: Find Alice's friends
alice.friends.each { |friend| puts "#{friend.name}, age #{friend.age}" }

# Raw Cypher query for complex traversals
results = ActiveGraph::Base.query(
  "MATCH (alice:Person {name: $name})-[:FRIENDS_WITH*2]->(fof:Person)
   WHERE fof.name <> $name
   RETURN DISTINCT fof.name AS name",
  name: 'Alice'
)
results.each { |row| puts row.name }
```


**Advanced Queries:**

```cypher
// Recommendation: "People who bought X also bought Y"
MATCH (user:Person)-[:PURCHASED]->(product:Product {name: "Laptop"})
MATCH (user)-[:PURCHASED]->(other:Product)
WHERE other.name <> "Laptop"
RETURN other.name, COUNT(*) AS frequency
ORDER BY frequency DESC
LIMIT 5

// Fraud detection: Find circular money transfers
MATCH path = (a:Account)-[:TRANSFERRED*3..6]->(a)
WHERE ALL(r IN relationships(path) WHERE r.amount > 10000)
RETURN path

// Social influence: Find the most connected people (degree centrality)
MATCH (p:Person)-[:FRIENDS_WITH]-()
RETURN p.name, COUNT(*) AS connections
ORDER BY connections DESC
LIMIT 10

// Community detection: Find clusters of connected people
MATCH (p1:Person)-[:FRIENDS_WITH]-(p2:Person)-[:FRIENDS_WITH]-(p3:Person)-[:FRIENDS_WITH]-(p1)
RETURN p1.name, p2.name, p3.name  -- triangles indicate tight communities
```

---

### Graph Traversal

Graph databases excel at **traversal** — following relationships from node to node. The key advantage is **index-free adjacency**: each node directly references its neighbors, so traversal is O(1) per hop regardless of the total graph size.

**Index-Free Adjacency:**

```
SQL approach (with JOINs):
  To find friends of friends:
  1. Scan friendships table for Alice's friends → O(n) or O(log n) with index
  2. For each friend, scan friendships table again → O(n) or O(log n) per friend
  3. Total: O(k * log n) where k = number of friends, n = total rows
  4. Gets exponentially worse with more hops

Graph approach (index-free adjacency):
  1. Go to Alice's node → follow FRIENDS_WITH pointers → O(k) where k = Alice's friends
  2. For each friend, follow their FRIENDS_WITH pointers → O(k) per friend
  3. Total: O(k^depth) — depends on connectivity, NOT on total graph size
  4. A graph with 1 billion nodes performs the same as one with 1000 nodes
     (for the same traversal depth and connectivity)
```

**Traversal Algorithms:**

| Algorithm | Purpose | Use Case |
|-----------|---------|----------|
| **BFS (Breadth-First Search)** | Find shortest path, explore level by level | Shortest path between users, degrees of separation |
| **DFS (Depth-First Search)** | Explore as deep as possible first | Cycle detection, topological sorting |
| **Dijkstra's** | Shortest weighted path | Route planning, minimum cost path |
| **PageRank** | Rank nodes by importance (link analysis) | Search ranking, influence scoring |
| **Community Detection (Louvain)** | Find clusters of densely connected nodes | Social communities, market segmentation |
| **Betweenness Centrality** | Find nodes that bridge communities | Key influencers, network bottlenecks |

---

### Use Cases for Graph Databases

| Use Case | Why Graph | Example |
|----------|----------|---------|
| **Social networks** | Friend recommendations, mutual connections, degrees of separation | Facebook, LinkedIn |
| **Recommendation engines** | "People who liked X also liked Y" — collaborative filtering via graph traversal | Netflix, Amazon |
| **Fraud detection** | Detect circular transactions, suspicious patterns, connected fraud rings | Banking, insurance |
| **Knowledge graphs** | Entities and their relationships (people, places, concepts) | Google Knowledge Graph, Wikipedia |
| **Network/IT infrastructure** | Map dependencies between servers, services, and configurations | Impact analysis, root cause analysis |
| **Identity and access management** | Who has access to what, through which roles and groups | RBAC traversal, permission checking |
| **Supply chain** | Track products through suppliers, manufacturers, distributors | Provenance, recall tracking |
| **Master data management** | Connect and deduplicate entities across systems | Customer 360, data integration |

**When NOT to use a Graph Database:**
- Simple CRUD with no relationship queries → use SQL or document store
- Bulk analytics and aggregations → use SQL, data warehouse, or column store
- High write throughput with simple access patterns → use Cassandra or DynamoDB
- Full-text search → use Elasticsearch
- Time-series data → use InfluxDB or TimescaleDB

---


## 19.6 SQL vs NoSQL Decision Framework

> Choosing between SQL and NoSQL is one of the most important decisions in system design. There is no universally "better" option — the right choice depends on your data model, access patterns, consistency requirements, scale, and team expertise. Many production systems use **both** SQL and NoSQL databases for different parts of the system (polyglot persistence).

> **Ruby Context:** Rails applications commonly start with PostgreSQL (via ActiveRecord) and add NoSQL databases as specific needs arise. The Ruby ecosystem has mature gems for all major databases, making polyglot persistence straightforward. Gems like `redis`, `mongoid`, `elasticsearch-ruby`, and `cassandra-driver` integrate cleanly alongside ActiveRecord.

---

### Data Structure Requirements

| Data Characteristic | SQL | NoSQL |
|--------------------|-----|-------|
| **Structured, tabular data** | ✅ Excellent — tables with fixed schema | ⚠️ Possible but not ideal |
| **Semi-structured data** (varying fields) | ⚠️ Possible with JSON columns | ✅ Document stores (MongoDB) |
| **Hierarchical / nested data** | ⚠️ Requires JOINs or JSON | ✅ Document stores (embed naturally) |
| **Highly connected data** (relationships) | ⚠️ JOINs work but get slow at depth | ✅ Graph databases (Neo4j) |
| **Simple key-value access** | ⚠️ Overkill | ✅ Key-value stores (Redis, DynamoDB) |
| **Time-ordered data** | ⚠️ Possible with indexes | ✅ Time-series DBs or column-family stores |
| **Full-text search** | ⚠️ Basic support | ✅ Search engines (Elasticsearch) |

**Question to ask:** "What does my data look like, and how will I access it?"

---

### Consistency vs Availability Needs

| Requirement | SQL | NoSQL |
|------------|-----|-------|
| **Strong consistency** (every read returns the latest write) | ✅ ACID transactions by default | ⚠️ Some support it (MongoDB, DynamoDB strong reads) but not the default |
| **Eventual consistency** (reads may return stale data briefly) | ⚠️ Not the default model | ✅ Cassandra, DynamoDB (default), most distributed NoSQL |
| **ACID transactions** (multi-row, multi-table) | ✅ Native support | ⚠️ Limited (MongoDB supports multi-document transactions since 4.0, but with performance cost) |
| **High availability** (survive node failures) | ⚠️ Requires replication setup | ✅ Built-in (Cassandra, DynamoDB — designed for availability) |
| **Partition tolerance** (survive network splits) | ⚠️ Single-node is CA; replicated setups vary | ✅ Designed for it (AP or CP depending on configuration) |

**CAP Theorem Quick Reference:**

```
        Consistency
           /\
          /  \
         /    \
        / CP   \
       /  systems\
      /    (HBase,\
     /   MongoDB   \
    /   strong mode)\
   /________________\
  Availability ---- Partition Tolerance
       AP systems
    (Cassandra, DynamoDB,
     CouchDB)

You can only guarantee 2 of 3 during a network partition.
In practice, P is non-negotiable in distributed systems,
so the real choice is between C and A.
```

| System | CAP Classification | Behavior During Partition |
|--------|-------------------|--------------------------|
| PostgreSQL (single node) | CA | No partition tolerance (single node) |
| PostgreSQL (with replication) | CP | Replicas may become unavailable to maintain consistency |
| MongoDB (default) | CP | Primary election; writes blocked during election |
| Cassandra | AP | Continues serving reads/writes; reconciles later |
| DynamoDB | AP (default) or CP (strong reads) | Configurable per-request |
| Redis Cluster | CP | Minority partition becomes unavailable |

---

### Read/Write Patterns

| Pattern | SQL | NoSQL |
|---------|-----|-------|
| **Read-heavy** (100:1 read:write) | ✅ Good with read replicas and indexes | ✅ Good (Redis cache, DynamoDB) |
| **Write-heavy** (high ingestion rate) | ⚠️ B-Tree indexes slow down writes | ✅ Cassandra, DynamoDB (LSM-tree, append-only) |
| **Complex queries** (JOINs, aggregations, window functions) | ✅ SQL is designed for this | ❌ Most NoSQL databases lack JOIN support |
| **Simple lookups** (get by key) | ⚠️ Works but overkill | ✅ Key-value stores (sub-millisecond) |
| **Range scans** (time ranges, pagination) | ✅ B-Tree indexes | ✅ Cassandra (clustering keys), DynamoDB (sort keys) |
| **Full-text search** | ⚠️ Basic (PostgreSQL is decent) | ✅ Elasticsearch (purpose-built) |
| **Graph traversal** (friends of friends) | ❌ JOINs get exponentially slower | ✅ Graph databases (O(1) per hop) |
| **Ad-hoc queries** (unknown query patterns) | ✅ SQL is flexible | ❌ NoSQL requires pre-designed access patterns |

**Key Insight:** If you don't know your query patterns upfront, start with SQL. If you know your exact access patterns and need extreme scale, NoSQL may be better.

---

### Scale Requirements

| Scale Dimension | SQL | NoSQL |
|----------------|-----|-------|
| **Vertical scaling** (bigger server) | ✅ Simple — add more CPU/RAM | ✅ Also works |
| **Horizontal scaling** (more servers) | ⚠️ Complex (sharding, read replicas, Vitess, Citus) | ✅ Built-in (Cassandra, DynamoDB, MongoDB sharding) |
| **Data volume** (terabytes to petabytes) | ⚠️ Possible with sharding but complex | ✅ Designed for it (Cassandra handles petabytes) |
| **Throughput** (millions of ops/sec) | ⚠️ Possible with caching + read replicas | ✅ DynamoDB, Cassandra handle millions of ops/sec natively |
| **Geographic distribution** (multi-region) | ⚠️ Complex (CockroachDB, Spanner) | ✅ Cassandra, DynamoDB Global Tables |

**Scale Thresholds (rough guidelines):**

| Data Size | Throughput | Recommendation |
|-----------|-----------|---------------|
| < 100 GB | < 10K QPS | SQL (PostgreSQL, MySQL) — don't over-engineer |
| 100 GB - 1 TB | 10K - 100K QPS | SQL with read replicas + caching (Redis) |
| 1 TB - 10 TB | 100K - 1M QPS | Consider NoSQL for specific access patterns; SQL with sharding for complex queries |
| > 10 TB | > 1M QPS | NoSQL for primary workload; SQL for transactional/analytical needs |

---

### Team Expertise

| Factor | SQL | NoSQL |
|--------|-----|-------|
| **Learning curve** | Low — SQL is widely known | Medium-High — each NoSQL DB has unique concepts |
| **Tooling** | Mature (decades of tools, ORMs, admin UIs) | Growing (varies by database) |
| **Hiring** | Easy — most developers know SQL | Harder — need specialists for Cassandra, Neo4j, etc. |
| **Debugging** | Easier (EXPLAIN, query plans, mature profiling) | Harder (distributed tracing, eventual consistency issues) |
| **Operations** | Well-understood (backups, replication, monitoring) | Complex (compaction tuning, consistency level management, cluster rebalancing) |

> **Ruby Context:** Ruby/Rails teams typically have strong SQL/ActiveRecord expertise. Adding NoSQL databases increases operational complexity. Start with PostgreSQL + Redis (for caching/sessions), and add specialized databases only when specific access patterns demand it.

---

### Decision Flowchart

```
Start
  │
  ├── Need ACID transactions across multiple entities?
  │     YES → SQL (PostgreSQL, MySQL)
  │     NO ↓
  │
  ├── Data is highly connected (relationships are the query)?
  │     YES → Graph Database (Neo4j)
  │     NO ↓
  │
  ├── Need full-text search with relevance ranking?
  │     YES → Search Engine (Elasticsearch) + primary DB
  │     NO ↓
  │
  ├── Simple key-value access pattern (get/set by key)?
  │     YES → Key-Value Store (Redis, DynamoDB)
  │     NO ↓
  │
  ├── Time-series data (metrics, IoT, logs)?
  │     YES → Time-Series DB (InfluxDB, TimescaleDB) or Cassandra
  │     NO ↓
  │
  ├── Need flexible schema with nested documents?
  │     YES → Document Store (MongoDB)
  │     NO ↓
  │
  ├── Need massive write throughput (millions/sec)?
  │     YES → Column-Family Store (Cassandra, ScyllaDB)
  │     NO ↓
  │
  └── Default → SQL (PostgreSQL)
        (Start simple, optimize later)
```

---

### Polyglot Persistence — Using Multiple Databases

Most real-world systems use **multiple databases**, each optimized for a specific workload:

```
E-Commerce System:

  ┌─────────────────────────────────────────────────────────┐
  │                    Application Layer                     │
  └──────┬──────────┬──────────┬──────────┬────────────────┘
         │          │          │          │
    ┌────▼────┐ ┌───▼───┐ ┌───▼───┐ ┌───▼────┐
    │PostgreSQL│ │ Redis │ │Elastic│ │Cassandra│
    │         │ │       │ │search │ │        │
    │ Orders  │ │ Cache │ │Product│ │ Event  │
    │ Users   │ │Session│ │Search │ │ Logs   │
    │ Payments│ │ Cart  │ │       │ │Analytics│
    │ (ACID)  │ │(fast) │ │(search│ │(writes)│
    └─────────┘ └───────┘ └───────┘ └────────┘
```

| Component | Database | Why |
|-----------|----------|-----|
| Orders, Users, Payments | PostgreSQL | ACID transactions, complex queries, data integrity |
| Cache, Sessions, Cart | Redis | Sub-millisecond reads, TTL, data structures |
| Product Search | Elasticsearch | Full-text search, faceted filtering, relevance ranking |
| Event Logs, Analytics | Cassandra | High write throughput, time-series access pattern |
| Recommendations | Neo4j | Graph traversal for "customers also bought" |
| File Storage | S3 | Object storage for images, videos, documents |

> **Ruby Context:** A typical Rails e-commerce app might use this gem stack for polyglot persistence:

```ruby
# Gemfile — polyglot persistence in a Rails application
gem 'pg'                      # PostgreSQL via ActiveRecord (orders, users, payments)
gem 'redis'                   # Redis for caching, sessions, Sidekiq
gem 'sidekiq'                 # Background jobs backed by Redis
gem 'mongoid'                 # MongoDB for flexible document storage
gem 'elasticsearch-model'     # Elasticsearch for product search
gem 'elasticsearch-rails'     # Rails integration for Elasticsearch
gem 'cassandra-driver'        # Cassandra for event logs / analytics
gem 'activegraph'             # Neo4j for recommendations / graph queries
gem 'aws-sdk-dynamodb'        # DynamoDB for serverless key-value access
gem 'aws-sdk-s3'              # S3 for file storage

# config/initializers/redis.rb
REDIS = Redis.new(url: ENV['REDIS_URL'])

# config/initializers/elasticsearch.rb
Elasticsearch::Model.client = Elasticsearch::Client.new(
  url: ENV['ELASTICSEARCH_URL'],
  log: Rails.env.development?
)
```

**Key Point:** Don't force one database to do everything. Use the right tool for each job. The complexity of managing multiple databases is offset by the performance and scalability benefits.

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| NoSQL Categories | Key-Value, Document, Column-Family, Graph, Time-Series, Search — each for a different access pattern |
| Key-Value Stores | Simplest model (key → value); Redis is the standard; sub-millisecond reads |
| Redis Data Structures | Strings, Lists, Sets, Sorted Sets, Hashes, Streams — far more than simple key-value |
| Redis Persistence | RDB (snapshots) + AOF (write log) — use both for durability |
| Redis Cluster | 16,384 hash slots distributed across masters; automatic failover |
| Document Stores | JSON documents with flexible schema; MongoDB is the standard |
| Embed vs Reference | Embed for data accessed together; reference for shared or unbounded data |
| MongoDB Aggregation | Pipeline of stages ($match, $group, $sort, $lookup) — like SQL GROUP BY + JOIN |
| Column-Family Stores | Cassandra — massive write throughput, peer-to-peer, eventual consistency |
| Cassandra Data Model | Design tables around queries; partition key determines node; clustering key determines sort |
| Cassandra Write Path | Commit log → Memtable → SSTable — writes are fast (sequential + in-memory) |
| Graph Databases | Nodes + edges + properties; O(1) per hop traversal; Neo4j + Cypher |
| Graph Use Cases | Social networks, recommendations, fraud detection, knowledge graphs |
| SQL vs NoSQL | SQL for ACID, complex queries, unknown patterns; NoSQL for scale, specific access patterns |
| CAP Theorem | CP (consistency + partition tolerance) or AP (availability + partition tolerance) — choose based on requirements |
| Polyglot Persistence | Use multiple databases — right tool for each job |
| Default Choice | Start with PostgreSQL; add NoSQL databases as specific needs arise |
| Ruby Ecosystem | `redis`, `mongoid`, `cassandra-driver`, `activegraph`, `elasticsearch-ruby`, `aws-sdk-dynamodb` — mature gems for all major NoSQL databases |

---

## Interview Tips for Module 19

1. **Know the categories** — be able to name the 6 NoSQL categories and give an example database and use case for each
2. **Redis data structures** — know when to use Strings vs Hashes vs Sorted Sets vs Streams; be ready to design a leaderboard or rate limiter with Redis
3. **Embed vs Reference** — in document store design, explain the trade-offs and give examples of when to use each
4. **Cassandra data modeling** — explain partition key vs clustering key; design a table for a given query pattern
5. **Cassandra consistency** — explain consistency levels and the formula R + W > RF for strong consistency
6. **Graph databases** — explain index-free adjacency and why graph traversal is O(1) per hop; write a Cypher query for "friends of friends"
7. **SQL vs NoSQL** — don't say "NoSQL is better" or "SQL is better"; articulate specific trade-offs for the given scenario
8. **CAP theorem** — explain with examples; know which databases are CP vs AP
9. **Polyglot persistence** — in HLD interviews, use multiple databases for different components and explain why
10. **DynamoDB** — know the partition key + sort key model, single-table design, and when to use it (AWS-native, serverless)
11. **Write path** — explain Cassandra's write path (commit log → memtable → SSTable) and why writes are fast
12. **Default to PostgreSQL** — when in doubt, start with PostgreSQL and add specialized databases as needed
13. **Ruby gems** — know the standard Ruby gems for each NoSQL database (`redis`, `mongoid`, `cassandra-driver`, `activegraph`, `elasticsearch-ruby`) and when to reach for them in a Rails application