# Module 20: Database Deep Dive

> This module goes beneath the surface of databases — into the distributed systems theory, replication strategies, sharding techniques, and storage engine internals that determine how databases actually work at scale. Understanding CAP theorem, replication lag, shard key selection, LSM trees vs B-Trees, and OLTP vs OLAP is what separates surface-level system design from truly informed architectural decisions.

---

## 20.1 CAP Theorem

> The **CAP Theorem** (Brewer's Theorem, 2000) states that a distributed data store can provide at most **two out of three** guarantees simultaneously: **Consistency**, **Availability**, and **Partition Tolerance**. Since network partitions are inevitable in distributed systems, the real choice is between consistency and availability during a partition.

---

### Consistency, Availability, Partition Tolerance

**Consistency (C):**
Every read receives the **most recent write** or an error. All nodes see the same data at the same time.

```
Consistent System:
  Client writes X=5 to Node A
  Client reads X from Node B → gets 5 (latest value)
  Client reads X from Node C → gets 5 (latest value)
  All nodes agree on the current value at all times.
```

**Availability (A):**
Every request receives a **non-error response**, without guarantee that it contains the most recent write. The system always responds, even if the data might be stale.

```
Available System:
  Client writes X=5 to Node A
  Network partition isolates Node B
  Client reads X from Node B → gets 3 (stale value, but still responds!)
  The system never returns an error — it always gives an answer.
```

**Partition Tolerance (P):**
The system continues to operate despite **network partitions** — messages between nodes being lost or delayed. In any real distributed system, network partitions will happen (cables get cut, switches fail, data centers lose connectivity).

```
Network Partition:
  +--------+          +--------+
  | Node A |  ✕ ✕ ✕   | Node B |    ← Network link is broken
  +--------+          +--------+
  
  Both nodes are alive, but they can't communicate.
  The system must decide: reject requests (sacrifice A) or serve stale data (sacrifice C)?
```

---

### The Real Trade-off: CP vs AP

Since network partitions are **unavoidable** in distributed systems, partition tolerance (P) is non-negotiable. The real choice during a partition is:

**CP (Consistency + Partition Tolerance):**
- During a partition, the system **rejects requests** (or blocks) rather than return stale data
- Guarantees that every successful read returns the latest write
- Sacrifices availability — some requests will fail or timeout

```
CP System During Partition:
  Client → Node A (isolated from Node B)
  Client writes X=5 to Node A
  Client reads X from Node B → ERROR or TIMEOUT
  (Node B refuses to answer because it can't confirm it has the latest data)
```

**AP (Availability + Partition Tolerance):**
- During a partition, the system **always responds**, even with potentially stale data
- Guarantees that every request gets a response
- Sacrifices consistency — different nodes may return different values

```
AP System During Partition:
  Client → Node A (isolated from Node B)
  Client writes X=5 to Node A
  Client reads X from Node B → returns 3 (stale but responds!)
  (Node B serves its local copy, even though it's outdated)
  After partition heals → Nodes reconcile and converge to X=5
```

---

### Database CAP Classifications

| Database | CAP | Behavior During Partition |
|----------|-----|--------------------------|
| **PostgreSQL** (single node) | CA | No partition (single node) — not distributed |
| **PostgreSQL** (streaming replication) | CP | Sync replicas block writes if replica is unreachable |
| **MySQL** (Group Replication) | CP | Minority partition becomes read-only |
| **MongoDB** (replica set) | CP | Primary election blocks writes for seconds; minority can't accept writes |
| **HBase** | CP | Regions become unavailable if RegionServer fails (until failover) |
| **CockroachDB** | CP | Ranges with minority replicas become unavailable |
| **Google Spanner** | CP | Uses TrueTime for global consistency; sacrifices availability during partitions |
| **Cassandra** | AP | Continues serving reads/writes on both sides of partition; reconciles later |
| **DynamoDB** | AP (default) | Eventually consistent reads by default; strong consistency optional |
| **CouchDB** | AP | Multi-master with conflict resolution on reconciliation |
| **Riak** | AP | Continues serving; uses vector clocks for conflict resolution |
| **Redis Cluster** | CP | Minority partition stops accepting writes |

**Important Nuance:** CAP classification is not absolute — many databases are configurable:
- **MongoDB:** CP by default, but you can read from secondaries (AP-like behavior)
- **DynamoDB:** AP by default, but supports strongly consistent reads (CP-like per request)
- **Cassandra:** AP by default, but `QUORUM` consistency level provides CP-like behavior

---

### PACELC Theorem (Extension of CAP)

The **PACELC theorem** extends CAP by addressing what happens when there is **no partition** (the normal case):

```
If there is a Partition (P):
  Choose between Availability (A) and Consistency (C)
Else (E) — normal operation:
  Choose between Latency (L) and Consistency (C)
```

**PACELC = PAC / ELC**

| Database | During Partition (PAC) | Normal Operation (ELC) | Full Classification |
|----------|----------------------|----------------------|-------------------|
| **PostgreSQL** | PC (consistency) | EC (consistency) | PC/EC |
| **MongoDB** | PC (consistency) | EC (consistency) | PC/EC |
| **Cassandra** | PA (availability) | EL (low latency) | PA/EL |
| **DynamoDB** | PA (availability) | EL (low latency) | PA/EL |
| **CockroachDB** | PC (consistency) | EC (consistency) | PC/EC |
| **Google Spanner** | PC (consistency) | EC (consistency, higher latency) | PC/EC |
| **Cosmos DB** | Configurable | Configurable | Configurable |

**Why PACELC matters:** CAP only describes behavior during partitions, which are rare. PACELC also captures the **latency vs consistency** trade-off during normal operation, which affects every single request.

- **Cassandra (PA/EL):** During partitions, stays available. During normal operation, prioritizes low latency (eventual consistency by default).
- **Spanner (PC/EC):** During partitions, sacrifices availability. During normal operation, still prioritizes consistency (higher latency due to TrueTime synchronization).

---


## 20.2 ACID vs BASE

> **ACID** and **BASE** represent two fundamentally different approaches to data consistency. ACID prioritizes correctness and strong consistency (typical of SQL databases). BASE prioritizes availability and performance, accepting temporary inconsistency (typical of distributed NoSQL databases). Understanding when to use each is critical for system design.

---

### ACID (Atomicity, Consistency, Isolation, Durability)

ACID properties were covered in depth in Module 18.5. Here's a quick recap in the context of distributed systems:

| Property | Guarantee | Cost |
|----------|-----------|------|
| **Atomicity** | All operations succeed or all fail | Requires coordination (2PC, WAL) |
| **Consistency** | Database moves from one valid state to another | Requires constraint checking |
| **Isolation** | Concurrent transactions don't interfere | Requires locking or MVCC (reduces throughput) |
| **Durability** | Committed data survives crashes | Requires disk writes (fsync) — adds latency |

**ACID in Distributed Systems:**
Maintaining ACID across multiple nodes is **extremely expensive**:
- **Atomicity** across nodes requires **Two-Phase Commit (2PC)** — a blocking protocol
- **Consistency** across nodes requires **consensus** (Paxos, Raft) — adds latency
- **Isolation** across nodes requires **distributed locking** — reduces throughput
- **Durability** across nodes requires **synchronous replication** — adds latency

This is why most distributed NoSQL databases relax ACID guarantees.

---

### BASE (Basically Available, Soft state, Eventually consistent)

BASE is the consistency model used by most distributed NoSQL databases:

| Property | Meaning |
|----------|---------|
| **Basically Available** | The system guarantees availability — it always responds, even if the data is stale or inconsistent |
| **Soft State** | The state of the system may change over time, even without new input (due to background reconciliation) |
| **Eventually Consistent** | If no new updates are made, all replicas will eventually converge to the same value |

**Eventually Consistent — What It Means:**

```
Time T1: Client writes X=5 to Node A
Time T2: Node A starts replicating to Node B and Node C (asynchronous)
Time T3: Client reads X from Node B → gets 3 (stale! replication hasn't reached B yet)
Time T4: Replication reaches Node B and Node C
Time T5: Client reads X from Node B → gets 5 (consistent now!)

The "eventual" window (T2 to T4) is typically milliseconds to seconds.
```

**Consistency Spectrum:**

```
Strong ←──────────────────────────────────────────→ Weak
Consistency                                        Consistency

Linearizable → Sequential → Causal → Read-your-writes → Monotonic → Eventual
    |              |           |            |                |           |
 Spanner      PostgreSQL   Cassandra    DynamoDB        Cassandra   Cassandra
 CockroachDB  (single node) (QUORUM)   (session        (ONE)       (ONE, async
                                        consistency)                 replication)
```

| Consistency Level | Guarantee | Example |
|------------------|-----------|---------|
| **Linearizable** | Every read returns the absolute latest write, globally ordered | Google Spanner |
| **Sequential** | All operations appear in some sequential order consistent with program order | PostgreSQL (Serializable) |
| **Causal** | Operations that are causally related are seen in the correct order | Cassandra with QUORUM |
| **Read-your-writes** | A client always sees its own writes | DynamoDB (session consistency) |
| **Monotonic reads** | Once a client reads a value, it never sees an older value | Most databases with sticky sessions |
| **Eventual** | All replicas converge eventually; no ordering guarantees | Cassandra with ONE, DNS |

---

### When to Choose ACID vs BASE

| Scenario | Choose | Why |
|----------|--------|-----|
| Financial transactions (banking, payments) | ACID | Cannot tolerate inconsistency — money must balance |
| Inventory management (last item in stock) | ACID | Double-selling is unacceptable |
| User registration (unique email) | ACID | Duplicate accounts cause problems |
| Social media feed | BASE | Slight delay in seeing a new post is acceptable |
| Analytics / metrics | BASE | Approximate counts are fine; eventual accuracy is sufficient |
| Shopping cart | BASE | Cart can be eventually consistent; final checkout needs ACID |
| Session storage | BASE | Stale session data is briefly acceptable |
| IoT sensor data | BASE | Missing or delayed readings are tolerable |
| Notification delivery | BASE | Slight delay is acceptable; at-least-once is fine |
| Audit logs | ACID | Must be complete and accurate for compliance |

**Hybrid Approach (most common in practice):**
Use ACID for the **critical path** (payments, inventory, user accounts) and BASE for **non-critical paths** (feeds, analytics, notifications, caching).

```
E-Commerce Checkout:
  1. Validate cart items → BASE (read from cache, eventually consistent)
  2. Reserve inventory → ACID (must be strongly consistent — prevent overselling)
  3. Process payment → ACID (must be atomic — charge AND record)
  4. Send confirmation email → BASE (eventual delivery is fine)
  5. Update analytics → BASE (eventual consistency is fine)
  6. Update recommendation engine → BASE (eventual consistency is fine)
```

---


## 20.3 Database Replication

> **Replication** is the process of copying data from one database server (primary) to one or more other servers (replicas). Replication provides **high availability** (if the primary fails, a replica takes over), **read scalability** (distribute read queries across replicas), and **disaster recovery** (replicas in different data centers survive regional failures).

---

### Master-Slave (Primary-Replica) Replication

The most common replication topology. One **primary** (master) handles all writes. One or more **replicas** (slaves) receive copies of the data and handle read queries.

```
                    Writes
  Client ──────────→ Primary (Master)
                        |
              ┌─────────┼─────────┐
              ↓         ↓         ↓
          Replica 1  Replica 2  Replica 3
              ↑         ↑         ↑
  Client ─── Reads ─────┘─────────┘
```

**How It Works:**
1. Client sends all writes (INSERT, UPDATE, DELETE) to the primary
2. Primary writes to its WAL (Write-Ahead Log)
3. Primary streams WAL changes to replicas
4. Replicas apply the changes to their local copy
5. Clients can read from any replica (or the primary)

**Advantages:**
- Simple to set up and understand
- Read scalability — add more replicas to handle more read traffic
- No write conflicts — only one node accepts writes

**Disadvantages:**
- Single point of failure for writes (until failover)
- Replication lag — replicas may serve stale data
- Failover complexity — promoting a replica to primary requires coordination

---

### Master-Master (Multi-Primary) Replication

Multiple nodes accept writes simultaneously. Each node replicates its changes to all other nodes.

```
  Client A ──→ Primary 1 ←──→ Primary 2 ←── Client B
                   ↕               ↕
               Primary 3 ←──→ Primary 4
                   ↑               ↑
              Client C         Client D
```

**Advantages:**
- No single point of failure for writes
- Write scalability — distribute writes across nodes
- Geographic distribution — users write to the nearest node

**Disadvantages:**
- **Write conflicts** — two nodes may update the same row simultaneously
- Conflict resolution is complex and error-prone
- Higher replication overhead (every node sends to every other node)
- Harder to maintain consistency

**When to Use Multi-Primary:**
- Multi-region deployments where each region needs local write capability
- Systems that can tolerate eventual consistency and have a clear conflict resolution strategy
- Collaborative editing (with OT or CRDT for conflict resolution)

---

### Synchronous vs Asynchronous Replication

**Synchronous Replication:**

```
Client → Write to Primary → Replicate to Replica → Replica ACKs → Primary ACKs Client
         (waits for replica confirmation before acknowledging the write)

Timeline:
  T1: Client sends write
  T2: Primary writes locally (1ms)
  T3: Primary sends to replica (network: 1-100ms)
  T4: Replica writes locally (1ms)
  T5: Replica ACKs to primary
  T6: Primary ACKs to client
  Total: 4-104ms (includes network round trip to replica)
```

| Pros | Cons |
|------|------|
| Zero data loss — replica is always up to date | Higher write latency (must wait for replica) |
| Strong consistency — reads from replica are current | Reduced availability — if replica is down, writes block |
| Simple failover — replica has all data | Throughput limited by slowest replica |

**Asynchronous Replication:**

```
Client → Write to Primary → Primary ACKs Client → (later) Replicate to Replica
         (acknowledges immediately, replicates in background)

Timeline:
  T1: Client sends write
  T2: Primary writes locally (1ms)
  T3: Primary ACKs to client (immediate!)
  T4: (background) Primary sends to replica
  T5: (background) Replica writes locally
  Total client wait: 2ms (no network round trip to replica)
```

| Pros | Cons |
|------|------|
| Low write latency (no waiting for replica) | Data loss risk — if primary crashes before replication, writes are lost |
| High availability — replica being down doesn't affect writes | Replication lag — replicas may serve stale data |
| Higher throughput | Weaker consistency guarantees |

**Semi-Synchronous Replication:**
A compromise — wait for **at least one** replica to acknowledge, but not all:

```
Client → Write to Primary → Replicate to Replicas → At least 1 Replica ACKs → Primary ACKs Client
         (waits for one replica, not all)
```

- Used by MySQL (semi-sync replication) and PostgreSQL (synchronous_standby_names)
- Balances durability (at least one copy) with performance (don't wait for all replicas)

---

### Replication Lag

**Replication lag** is the delay between a write on the primary and that write becoming visible on a replica. It's the fundamental challenge of asynchronous replication.

```
Primary:  Write X=5 at T1
Replica:  Still shows X=3 at T1 (lag!)
Replica:  Shows X=5 at T1 + lag (typically milliseconds to seconds)
```

**Problems Caused by Replication Lag:**

**1. Read-After-Write Inconsistency:**
```
User updates profile (write → primary)
User refreshes page (read → replica)
→ User sees OLD profile! (replica hasn't caught up)

Solution: Read-your-writes consistency
  - Read from primary for data the user just wrote
  - Or track write timestamp and only read from replicas that are caught up
```

**2. Monotonic Read Violations:**
```
User reads X=5 from Replica A (caught up)
User reads X=3 from Replica B (lagging)
→ User sees time go backward!

Solution: Monotonic reads
  - Route a user's reads to the same replica (sticky sessions)
  - Or track the last-read version and only accept newer versions
```

**3. Causal Ordering Violations:**
```
User A posts a comment (write → primary)
User B sees the comment (read → Replica 1, caught up)
User B replies to the comment (write → primary)
User C reads from Replica 2 (lagging) → sees the reply but NOT the original comment!

Solution: Causal consistency
  - Track causal dependencies between operations
  - Ensure replicas apply operations in causal order
```

**Monitoring Replication Lag:**

```sql
-- PostgreSQL: Check replication lag
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       (sent_lsn - replay_lsn) AS replication_lag_bytes
FROM pg_stat_replication;

-- MySQL: Check seconds behind master
SHOW SLAVE STATUS\G
-- Look for: Seconds_Behind_Master
```

**Acceptable Lag Thresholds:**

| Use Case | Acceptable Lag | Strategy |
|----------|---------------|----------|
| Analytics dashboards | Minutes | Async replication, read from any replica |
| Social media feeds | Seconds | Async replication, read-your-writes for own content |
| E-commerce product pages | Seconds | Async replication + cache invalidation |
| Financial balances | Zero | Sync replication or read from primary |
| Inventory counts | Zero | Read from primary for stock checks |

---

### Read Replicas

**Read replicas** are replicas specifically designated to handle read traffic, offloading the primary.

```
Write Traffic (10%)                Read Traffic (90%)
      |                           /       |       \
      v                          v        v        v
  +--------+              +---------+ +---------+ +---------+
  | Primary|  ──async──→  |Replica 1| |Replica 2| |Replica 3|
  +--------+              +---------+ +---------+ +---------+
```

**Scaling Reads with Replicas:**
- Primary handles all writes (10% of traffic)
- Read replicas handle all reads (90% of traffic)
- Add more replicas to handle more read traffic
- Each replica can handle the same read throughput as the primary

**Cloud Managed Read Replicas:**

| Cloud Service | Max Read Replicas | Cross-Region | Auto-Failover |
|--------------|-------------------|-------------|---------------|
| AWS RDS | 15 | Yes | Yes (Multi-AZ) |
| AWS Aurora | 15 | Yes (Global Database) | Yes |
| GCP Cloud SQL | 10 | Yes | Yes |
| Azure SQL | 4 | Yes (Geo-Replication) | Yes |

---

### Conflict Resolution in Multi-Master

When multiple primaries accept writes simultaneously, **conflicts** are inevitable — two nodes may update the same row at the same time.

**Types of Conflicts:**

| Conflict Type | Example |
|--------------|---------|
| **Write-Write** | Node A sets X=5, Node B sets X=10 at the same time |
| **Delete-Update** | Node A deletes row, Node B updates the same row |
| **Insert-Insert** | Node A and Node B both insert a row with the same key |

**Conflict Resolution Strategies:**

**1. Last-Write-Wins (LWW):**
- Each write has a timestamp; the write with the latest timestamp wins
- Simple but can lose data silently

```
Node A: SET X=5 at T1=1000
Node B: SET X=10 at T2=1001
Resolution: X=10 (T2 > T1, Node B wins)
Problem: Node A's write is silently discarded
```

**2. Vector Clocks:**
- Each node maintains a vector of logical timestamps (one per node)
- Can detect concurrent writes (neither happened before the other)
- Concurrent writes are flagged for application-level resolution

```
Node A: X=5, vector=[A:2, B:1]
Node B: X=10, vector=[A:1, B:2]
Neither dominates → CONFLICT DETECTED
Application must resolve (e.g., merge, prompt user, pick one)
```

**3. CRDTs (Conflict-free Replicated Data Types):**
- Data structures that can be merged automatically without conflicts
- Mathematically guaranteed to converge
- Examples: G-Counter (grow-only counter), OR-Set (observed-remove set), LWW-Register

```
G-Counter (each node has its own counter):
  Node A: {A: 5, B: 0} → total = 5
  Node B: {A: 0, B: 3} → total = 3
  Merge:  {A: 5, B: 3} → total = 8 (no conflict!)
```

**4. Application-Level Resolution:**
- The database stores all conflicting versions
- The application reads all versions and decides how to merge
- Used by CouchDB, Riak

| Strategy | Pros | Cons | Used By |
|----------|------|------|---------|
| Last-Write-Wins | Simple, automatic | Silent data loss | Cassandra, DynamoDB |
| Vector Clocks | Detects conflicts accurately | Complex, requires app-level resolution | Riak, Dynamo |
| CRDTs | Automatic, no conflicts | Limited data types, higher storage | Redis (CRDT module), Riak |
| Application-Level | Full control | Complex application logic | CouchDB |

---


## 20.4 Database Sharding / Partitioning

> **Sharding** (horizontal partitioning) splits a large database into smaller, independent pieces called **shards**, each stored on a separate server. Sharding is the primary technique for scaling databases beyond what a single server can handle — when you have terabytes of data or millions of queries per second.

---

### Horizontal Partitioning (Sharding) vs Vertical Partitioning

**Horizontal Partitioning (Sharding):**
Split rows across multiple servers. Each shard has the same schema but different rows.

```
Before (single server):
  users table: 100 million rows on one server

After (3 shards):
  Shard 1: users with id 1 - 33,333,333
  Shard 2: users with id 33,333,334 - 66,666,666
  Shard 3: users with id 66,666,667 - 100,000,000

Each shard is an independent database server with the same table structure.
```

**Vertical Partitioning:**
Split columns across multiple servers. Each partition has different columns for the same rows.

```
Before (single server):
  users table: id, name, email, bio, avatar_url, preferences, activity_log

After (vertical partitioning):
  Server 1 (hot data):  id, name, email          ← frequently accessed
  Server 2 (cold data): id, bio, avatar_url       ← less frequently accessed
  Server 3 (large data): id, preferences, activity_log  ← large, rarely accessed
```

| Feature | Horizontal (Sharding) | Vertical |
|---------|----------------------|----------|
| Split by | Rows | Columns |
| Schema | Same on all shards | Different on each partition |
| Scale | Virtually unlimited (add more shards) | Limited (finite number of columns) |
| Complexity | High (routing, cross-shard queries) | Moderate (join across partitions) |
| Use case | Large datasets, high throughput | Separate hot/cold data, large columns |

---

### Sharding Strategies

**1. Range-Based Sharding:**

Assign rows to shards based on a range of the shard key.

```
Shard Key: user_id
  Shard 1: user_id 1 - 1,000,000
  Shard 2: user_id 1,000,001 - 2,000,000
  Shard 3: user_id 2,000,001 - 3,000,000

Shard Key: created_at (date)
  Shard 1: January 2025
  Shard 2: February 2025
  Shard 3: March 2025
```

| Pros | Cons |
|------|------|
| Simple to implement and understand | **Hotspots** — new users/data always go to the latest shard |
| Efficient range queries (all data in one shard) | Uneven distribution — some ranges may have more data |
| Easy to add new shards (extend the range) | Requires rebalancing when shards become uneven |

**2. Hash-Based Sharding:**

Apply a hash function to the shard key and use the result to determine the shard.

```
Shard Key: user_id
Hash Function: shard = hash(user_id) % num_shards

  hash("user_123") % 3 = 1 → Shard 1
  hash("user_456") % 3 = 0 → Shard 0
  hash("user_789") % 3 = 2 → Shard 2
```

| Pros | Cons |
|------|------|
| Even distribution — no hotspots | Range queries require querying ALL shards (scatter-gather) |
| Simple routing logic | Adding/removing shards requires **resharding** (rehashing all keys) |
| Works well for random access patterns | Loss of data locality (related data may be on different shards) |

**Consistent Hashing** solves the resharding problem — when adding a shard, only ~1/N of keys need to move (instead of all keys). See Module 27 for details.

**3. Directory-Based Sharding:**

A lookup table (directory) maps each key to its shard. The directory is stored in a separate service.

```
Directory Service:
  user_123 → Shard 2
  user_456 → Shard 1
  user_789 → Shard 3

Client → Directory Service: "Where is user_123?"
Directory → "Shard 2"
Client → Shard 2: GET user_123
```

| Pros | Cons |
|------|------|
| Maximum flexibility — any key can be on any shard | Directory is a single point of failure |
| Easy to rebalance — just update the directory | Extra hop for every query (directory lookup) |
| Supports complex routing logic | Directory must be highly available and fast |

**4. Geographic Sharding:**

Assign data to shards based on geographic location.

```
  Shard US-East:  Users in North America
  Shard EU-West:  Users in Europe
  Shard AP-South: Users in Asia-Pacific

Routing: Based on user's country/region
```

| Pros | Cons |
|------|------|
| Low latency — data is near the user | Cross-region queries are slow |
| Data residency compliance (GDPR — EU data stays in EU) | Uneven distribution (more users in some regions) |
| Natural disaster isolation | Complex routing logic |

---

### Shard Key Selection

The **shard key** is the most important decision in sharding. A bad shard key leads to hotspots, uneven distribution, and cross-shard queries.

**Good Shard Key Properties:**

| Property | Why It Matters |
|----------|---------------|
| **High cardinality** | Many distinct values → even distribution across shards |
| **Even distribution** | Each shard gets roughly the same amount of data and traffic |
| **Query alignment** | Most queries include the shard key → single-shard queries (fast) |
| **Write distribution** | Writes are spread across shards (no single hot shard) |
| **Immutable** | Shard key should never change (moving data between shards is expensive) |

**Shard Key Examples:**

| Use Case | Good Shard Key | Bad Shard Key | Why |
|----------|---------------|--------------|-----|
| User data | `user_id` | `country` | Country has low cardinality; US shard would be huge |
| Orders | `user_id` | `order_date` | Date causes hotspot on latest shard; user_id distributes evenly |
| Chat messages | `conversation_id` | `sender_id` | Messages in a conversation should be on the same shard |
| IoT sensor data | `device_id` | `timestamp` | Timestamp causes all writes to go to the latest shard |
| Multi-tenant SaaS | `tenant_id` | `user_id` | Tenant data should be co-located for isolation and queries |

**Compound Shard Key:**
Sometimes a single column isn't enough. Use a compound key:

```
Shard Key: (tenant_id, user_id)
  → Data is first partitioned by tenant (co-location)
  → Then distributed by user within each tenant (even distribution)
```

---

### Cross-Shard Queries

The biggest challenge with sharding: queries that span multiple shards.

```
Single-Shard Query (fast):
  SELECT * FROM orders WHERE user_id = 123;
  → Router knows user_id=123 is on Shard 2 → query Shard 2 only

Cross-Shard Query (slow — scatter-gather):
  SELECT * FROM orders WHERE total > 100 ORDER BY created_at DESC LIMIT 10;
  → Must query ALL shards → merge results → sort → return top 10
```

**Scatter-Gather Pattern:**

```
Client → Router
  Router → Shard 1: SELECT * FROM orders WHERE total > 100 ORDER BY created_at DESC LIMIT 10
  Router → Shard 2: SELECT * FROM orders WHERE total > 100 ORDER BY created_at DESC LIMIT 10
  Router → Shard 3: SELECT * FROM orders WHERE total > 100 ORDER BY created_at DESC LIMIT 10
  
  Router collects results from all shards
  Router merges, sorts, and returns top 10 to client
```

**Problems with Cross-Shard Queries:**
- **Performance:** Must query all shards and merge results
- **JOINs:** Joining data across shards is extremely expensive (or impossible)
- **Transactions:** ACID transactions across shards require 2PC (slow, complex)
- **Aggregations:** COUNT, SUM, AVG across shards require scatter-gather
- **Pagination:** OFFSET-based pagination across shards is unreliable

**Mitigation Strategies:**
- Design the shard key so most queries are single-shard
- Denormalize data to avoid cross-shard JOINs
- Use a separate analytics database (data warehouse) for cross-shard aggregations
- Use application-level joins (query each shard, join in application code)

---

### Resharding Challenges

**Resharding** is the process of changing the number of shards or the sharding strategy. It's one of the most painful operations in database management.

**When Resharding is Needed:**
- A shard has grown too large (data or traffic)
- Adding capacity (more shards for more throughput)
- Hotspot mitigation (rebalancing data away from a hot shard)
- Changing the shard key (rare, extremely painful)

**Resharding Approaches:**

| Approach | Description | Downtime |
|----------|-------------|----------|
| **Stop-the-world** | Stop writes, move data, restart | Yes (minutes to hours) |
| **Double-write** | Write to both old and new shards during migration | No (but complex) |
| **Shadow migration** | Copy data in background, switch reads when caught up | Minimal |
| **Consistent hashing** | Only ~1/N of keys move when adding a shard | Minimal |
| **Vitess / ProxySQL** | Middleware handles resharding transparently | Minimal |

**Key Point:** Resharding is so painful that you should plan your initial sharding strategy carefully. Over-shard initially (more shards than you need) — it's easier to merge shards than to split them.

---


## 20.5 Database Internals

> Understanding how databases store and retrieve data internally helps you make better decisions about schema design, indexing, and database selection. The two dominant storage engine architectures are **B-Tree** (optimized for reads) and **LSM-Tree** (optimized for writes). This section covers the key internal components that power modern databases.

---

### Write-Ahead Log (WAL)

The **WAL** (also called redo log, transaction log, or binlog) is the foundation of database durability. Every change is written to the WAL **before** being applied to the actual data pages.

```
Write Operation:
  1. Client sends: UPDATE users SET name='Bob' WHERE id=1
  2. Database writes to WAL: "UPDATE users SET name='Bob' WHERE id=1" (sequential append — fast)
  3. Database acknowledges to client: "Committed!" (data is durable in WAL)
  4. (Later) Database applies the change to the actual data page on disk (random I/O — slow)

Crash Recovery:
  1. Database restarts after crash
  2. Reads the WAL from the last checkpoint
  3. Replays all committed transactions that weren't applied to data pages
  4. Rolls back any uncommitted transactions
  5. Database is now consistent — no data loss for committed transactions
```

**Why WAL is Fast:**
- WAL writes are **sequential appends** to a single file
- Sequential I/O is 100-1000x faster than random I/O on HDDs, and 10-100x faster on SSDs
- The actual data page updates (random I/O) happen in the background
- This is why databases can acknowledge commits quickly — they only need to write to the WAL

**WAL in Different Databases:**

| Database | WAL Name | Details |
|----------|----------|---------|
| PostgreSQL | WAL (Write-Ahead Log) | 16 MB segments, used for replication and point-in-time recovery |
| MySQL/InnoDB | Redo Log + Binary Log | Redo log for crash recovery, binlog for replication |
| SQLite | WAL mode | Optional (default is rollback journal) |
| Cassandra | Commit Log | Append-only, per-node |
| MongoDB | Oplog (Operations Log) | Capped collection, used for replication |

---

### B-Tree vs LSM-Tree Storage Engines

The two dominant storage engine architectures:

**B-Tree (Read-Optimized):**

```
B-Tree Structure:
                    [50]
                   /    \
              [20|30]   [70|80]
             / | \      / | \
          [10][25][35] [60][75][90]  ← leaf nodes contain data (or pointers to data)

Write: Find the correct leaf node → update in place (random I/O)
Read:  Traverse from root to leaf → O(log n) (typically 3-4 disk reads)
```

**How B-Tree Writes Work:**
1. Find the correct leaf page (O(log n) — traverse the tree)
2. Update the page in place (random I/O — write to a specific disk location)
3. If the page is full, split it and update the parent (cascading splits)
4. Write the change to WAL for durability

**LSM-Tree (Write-Optimized):**

```
LSM-Tree Structure:

  Memory:
    +------------------+
    | Memtable         |  ← sorted in-memory structure (Red-Black tree or skip list)
    | (write buffer)   |     All writes go here first (fast — in-memory)
    +------------------+
           |
           | (flush when full)
           ↓
  Disk Level 0:
    +--------+ +--------+ +--------+
    |SSTable1| |SSTable2| |SSTable3|  ← immutable sorted files
    +--------+ +--------+ +--------+
           |
           | (compaction — merge sorted files)
           ↓
  Disk Level 1:
    +------------------+ +------------------+
    |   SSTable A      | |   SSTable B      |  ← larger, merged SSTables
    +------------------+ +------------------+
           |
           | (compaction)
           ↓
  Disk Level 2:
    +----------------------------------------+
    |           SSTable X (largest)           |
    +----------------------------------------+
```

**How LSM-Tree Writes Work:**
1. Write to the Memtable (in-memory — extremely fast)
2. Write to the WAL/Commit Log (sequential append — for durability)
3. When Memtable is full, flush it to disk as an **SSTable** (Sorted String Table)
4. SSTables are **immutable** — never modified after creation
5. Background **compaction** merges multiple SSTables into fewer, larger ones

**How LSM-Tree Reads Work:**
1. Check the Memtable (in-memory — fastest)
2. Check each SSTable level, from newest to oldest
3. Use **Bloom filters** to skip SSTables that definitely don't contain the key
4. Use **sparse indexes** within SSTables to find the right block
5. Return the most recent version of the key

---

**B-Tree vs LSM-Tree Comparison:**

| Feature | B-Tree | LSM-Tree |
|---------|--------|----------|
| **Write performance** | Moderate (random I/O for in-place updates) | Excellent (sequential writes, in-memory buffer) |
| **Read performance** | Excellent (O(log n), single location) | Good (may check multiple SSTables) |
| **Write amplification** | Low (update in place) | Higher (data written multiple times during compaction) |
| **Read amplification** | Low (one tree traversal) | Higher (may read from multiple SSTables) |
| **Space amplification** | Low (data stored once) | Higher (multiple versions until compaction) |
| **Range queries** | Excellent (leaf nodes are linked) | Good (within SSTables, but may span multiple) |
| **Concurrency** | Good (page-level locking) | Excellent (writes don't block reads) |
| **Compaction overhead** | None | Background CPU and I/O for compaction |
| **Used by** | PostgreSQL, MySQL/InnoDB, Oracle, SQL Server | Cassandra, HBase, RocksDB, LevelDB, ScyllaDB |
| **Best for** | Read-heavy, OLTP, complex queries | Write-heavy, time-series, logs, IoT |

---

### SSTables and Compaction

**SSTable (Sorted String Table):**
An immutable, sorted file of key-value pairs written to disk when the Memtable is flushed.

```
SSTable Structure:
  +--------------------------------------------------+
  | Key: "alice" → Value: {name: "Alice", age: 30}   |
  | Key: "bob"   → Value: {name: "Bob", age: 25}     |
  | Key: "charlie" → Value: {name: "Charlie", age: 35}|
  +--------------------------------------------------+
  | Sparse Index: alice→offset:0, charlie→offset:200  |
  +--------------------------------------------------+
  | Bloom Filter: [bit array for quick key existence check] |
  +--------------------------------------------------+
```

**Properties:**
- **Sorted:** Keys are in sorted order → efficient range scans and merging
- **Immutable:** Never modified after creation → no locking needed, safe for concurrent reads
- **Indexed:** Sparse index for fast key lookup within the file
- **Filtered:** Bloom filter for quick "does this key exist?" checks

**Compaction Strategies:**

**Size-Tiered Compaction (STCS):**
```
When N SSTables of similar size accumulate → merge them into one larger SSTable

Level 0: [1MB] [1MB] [1MB] [1MB]  → merge into →  Level 1: [4MB]
Level 1: [4MB] [4MB] [4MB] [4MB]  → merge into →  Level 2: [16MB]
```
- Good for write-heavy workloads
- Can have high space amplification (old data persists until compaction)
- Used by Cassandra (default)

**Leveled Compaction (LCS):**
```
Level 0: SSTables from memtable flushes (may overlap)
Level 1: Non-overlapping SSTables, each ~10MB, total ~10MB
Level 2: Non-overlapping SSTables, each ~10MB, total ~100MB (10x Level 1)
Level 3: Non-overlapping SSTables, each ~10MB, total ~1GB (10x Level 2)

Compaction: merge one SSTable from Level N with overlapping SSTables in Level N+1
```
- Better read performance (fewer SSTables to check per level)
- Lower space amplification
- Higher write amplification (more compaction work)
- Used by LevelDB, RocksDB

---

### Bloom Filters

A **Bloom filter** is a probabilistic data structure that answers: "Is this key **possibly** in this SSTable?" It can say "definitely not" (no false negatives) or "maybe yes" (possible false positives).

```
Bloom Filter for SSTable:
  Contains keys: alice, bob, charlie

  Query: "Is 'dave' in this SSTable?"
  Bloom filter: "Definitely NOT" → skip this SSTable entirely (saved a disk read!)

  Query: "Is 'alice' in this SSTable?"
  Bloom filter: "Maybe YES" → read the SSTable to confirm

  Query: "Is 'eve' in this SSTable?"
  Bloom filter: "Maybe YES" (false positive!) → read the SSTable → not found
  (Wasted one disk read, but this is rare with a well-sized Bloom filter)
```

**How Bloom Filters Work:**
1. A bit array of size M (all zeros initially)
2. K hash functions that map keys to positions in the bit array
3. **Insert:** Hash the key with all K functions → set those K bits to 1
4. **Query:** Hash the key with all K functions → if ALL K bits are 1, "maybe yes"; if ANY bit is 0, "definitely no"

```
Bit array (M=10): [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]

Insert "alice" (hash1=2, hash2=5, hash3=8):
  [0, 0, 1, 0, 0, 1, 0, 0, 1, 0]

Insert "bob" (hash1=1, hash2=5, hash3=7):
  [0, 1, 1, 0, 0, 1, 0, 1, 1, 0]

Query "charlie" (hash1=3, hash2=6, hash3=9):
  Positions 3, 6, 9 → [0, ?, 0] → bit 3 is 0 → DEFINITELY NOT in set

Query "dave" (hash1=1, hash2=2, hash3=5):
  Positions 1, 2, 5 → [1, 1, 1] → all bits are 1 → MAYBE in set (false positive!)
```

**Bloom Filter in Databases:**
- Each SSTable has its own Bloom filter
- Before reading an SSTable from disk, check the Bloom filter (in memory)
- If "definitely not" → skip the SSTable (save a disk read)
- False positive rate is configurable (typically 1% — use more memory for lower rate)
- Dramatically reduces read amplification in LSM-Tree databases

---

### Page Cache and Buffer Pool

**Page Cache (OS-level):**
The operating system caches recently accessed disk pages in RAM. When the database reads a page, the OS may serve it from cache instead of disk.

```
Database reads page → OS checks page cache
  Cache HIT → return from RAM (microseconds)
  Cache MISS → read from disk (milliseconds) → store in page cache → return
```

**Buffer Pool (Database-level):**
The database maintains its own cache of data pages in memory, separate from the OS page cache. This gives the database more control over caching behavior.

```
PostgreSQL: shared_buffers (typically 25% of RAM)
MySQL/InnoDB: innodb_buffer_pool_size (typically 70-80% of RAM)

Query → Check buffer pool
  HIT → return from buffer pool (fastest)
  MISS → read from disk (or OS page cache) → load into buffer pool → return
```

**Buffer Pool Management:**
- Uses a **clock sweep** or **LRU** algorithm to evict cold pages
- **Dirty pages** (modified but not yet written to disk) are flushed periodically
- **Pin count** prevents eviction of pages currently in use
- **Prefetching** loads adjacent pages for sequential scans

**Key Point for System Design:** The buffer pool is why databases perform well even with large datasets — frequently accessed data stays in memory. Size the buffer pool appropriately (70-80% of RAM for dedicated database servers).

---

### Query Optimizer Basics

The **query optimizer** decides **how** to execute a SQL query — which indexes to use, which join algorithm, what order to join tables, etc. The same query can be executed in many different ways with vastly different performance.

**What the Optimizer Decides:**

| Decision | Options |
|----------|---------|
| **Access method** | Full table scan, index scan, index-only scan, bitmap scan |
| **Join algorithm** | Nested loop, hash join, merge join |
| **Join order** | Which table to scan first (for multi-table joins) |
| **Index selection** | Which index to use (or none) |
| **Sort method** | In-memory sort, external sort, index-based sort |
| **Aggregation method** | Hash aggregation, sort-based aggregation |

**Cost-Based Optimization:**
The optimizer estimates the **cost** of each possible execution plan and chooses the cheapest one. Cost is based on:
- **Table statistics:** Row count, data distribution, distinct values per column
- **Index statistics:** Index size, selectivity, correlation
- **I/O cost:** Number of disk pages to read
- **CPU cost:** Number of rows to process, sort, hash

```sql
-- PostgreSQL: View the optimizer's chosen plan
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
GROUP BY u.name
ORDER BY order_count DESC
LIMIT 10;

-- Output shows:
-- Limit (cost=1234.56..1234.78 rows=10)
--   → Sort (cost=1234.56..1234.78 rows=500)
--       Sort Key: (count(o.id)) DESC
--       → HashAggregate (cost=1200.00..1205.00 rows=500)
--           Group Key: u.name
--           → Hash Join (cost=100.00..1100.00 rows=5000)
--               Hash Cond: (o.user_id = u.id)
--               → Seq Scan on orders o (cost=0.00..800.00 rows=50000)
--               → Hash (cost=50.00..50.00 rows=500)
--                   → Index Scan using idx_users_status on users u (cost=0.00..50.00 rows=500)
--                       Index Cond: (status = 'active')
```

**Keeping Statistics Up to Date:**

```sql
-- PostgreSQL: Update statistics (run periodically or after large data changes)
ANALYZE users;
ANALYZE orders;

-- MySQL: Update statistics
ANALYZE TABLE users;
ANALYZE TABLE orders;
```

Stale statistics → bad query plans → slow queries. Most databases run auto-analyze, but after bulk loads or major data changes, run ANALYZE manually.

---


## 20.6 Data Warehousing & OLAP

> Production databases (OLTP) are optimized for fast, small transactions — inserting an order, updating a profile. But when you need to analyze millions of rows for business intelligence — "What was our revenue by region last quarter?" — you need a different architecture. **Data warehouses** and **OLAP** systems are designed for exactly this.

---

### OLTP vs OLAP

| Feature | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
|---------|--------------------------------------|--------------------------------------|
| **Purpose** | Day-to-day operations | Business intelligence, analytics, reporting |
| **Queries** | Simple, short (INSERT, UPDATE, SELECT by key) | Complex, long (aggregations, JOINs across millions of rows) |
| **Data volume per query** | Few rows (one order, one user) | Millions to billions of rows |
| **Users** | Application servers, end users | Data analysts, BI tools, data scientists |
| **Optimization** | Write performance, low latency | Read performance, throughput |
| **Schema** | Normalized (3NF) | Denormalized (star/snowflake schema) |
| **Data freshness** | Real-time (current state) | Near real-time to batch (historical data) |
| **Storage** | Row-oriented | Column-oriented |
| **Examples** | PostgreSQL, MySQL, DynamoDB | Snowflake, BigQuery, Redshift, ClickHouse |

```
OLTP Query (fast, simple):
  SELECT name, email FROM users WHERE id = 123;
  → Reads 1 row, returns in < 1ms

OLAP Query (slow, complex):
  SELECT region, product_category,
         SUM(revenue) AS total_revenue,
         COUNT(DISTINCT customer_id) AS unique_customers,
         AVG(order_value) AS avg_order_value
  FROM sales_facts
  JOIN dim_products ON sales_facts.product_id = dim_products.id
  JOIN dim_regions ON sales_facts.region_id = dim_regions.id
  WHERE sale_date BETWEEN '2024-01-01' AND '2024-12-31'
  GROUP BY region, product_category
  ORDER BY total_revenue DESC;
  → Scans 500 million rows, returns in 5-30 seconds
```

---

### Star Schema and Snowflake Schema

Data warehouses use **dimensional modeling** — organizing data into **fact tables** (measurements/events) and **dimension tables** (descriptive attributes).

**Star Schema:**

```
                    +----------------+
                    | dim_products   |
                    | product_id (PK)|
                    | name           |
                    | category       |
                    | brand          |
                    +-------+--------+
                            |
+----------------+  +-------+--------+  +----------------+
| dim_customers  |  | fact_sales     |  | dim_time       |
| customer_id(PK)|--| sale_id (PK)   |--| date_id (PK)   |
| name           |  | customer_id(FK)|  | date           |
| email          |  | product_id (FK)|  | day_of_week    |
| segment        |  | time_id (FK)   |  | month          |
| region         |  | region_id (FK) |  | quarter        |
+----------------+  | quantity       |  | year           |
                    | unit_price     |  +----------------+
                    | total_amount   |
                    | discount       |
+----------------+  +----------------+
| dim_regions    |          |
| region_id (PK) |----------+
| region_name    |
| country        |
| continent      |
+----------------+
```

**Star Schema Characteristics:**
- **Fact table** at the center — contains measurements (revenue, quantity, clicks)
- **Dimension tables** around it — contain descriptive attributes (who, what, when, where)
- Fact table has foreign keys to all dimension tables
- Dimension tables are **denormalized** (no further normalization)
- Simple JOINs — fact table joins directly to each dimension (one hop)

**Snowflake Schema:**

Like a star schema, but dimension tables are **normalized** — they have their own sub-dimensions.

```
  dim_products → dim_categories → dim_departments
  (product_id,   (category_id,    (department_id,
   name,          name,            name)
   category_id)   department_id)
```

| Feature | Star Schema | Snowflake Schema |
|---------|------------|-----------------|
| Dimension normalization | Denormalized (flat) | Normalized (sub-dimensions) |
| Query complexity | Simpler (fewer JOINs) | More complex (more JOINs) |
| Query performance | Faster (fewer JOINs) | Slower (more JOINs) |
| Storage | More (redundant data in dimensions) | Less (normalized) |
| Maintenance | Easier | Harder (more tables to manage) |
| Preferred | Most data warehouses | When dimension tables are very large |

**Key Point:** Star schema is preferred for most data warehouses because query performance matters more than storage savings.

---

### ETL (Extract, Transform, Load)

**ETL** is the process of moving data from operational databases (OLTP) to the data warehouse (OLAP).

```
Source Systems          ETL Pipeline              Data Warehouse
+------------+                                    +-------------+
| PostgreSQL | ──Extract──→ ┌──────────┐          |             |
+------------+              │Transform │          |  Snowflake  |
+------------+              │          │──Load──→ |  BigQuery   |
| MongoDB    | ──Extract──→ │ Clean    │          |  Redshift   |
+------------+              │ Validate │          |             |
+------------+              │ Aggregate│          +-------------+
| API Logs   | ──Extract──→ │ Join     │
+------------+              │ Dedupe   │
                            └──────────┘
```

**ETL Stages:**

| Stage | Description | Examples |
|-------|-------------|---------|
| **Extract** | Pull data from source systems | Read from databases, APIs, files, message queues |
| **Transform** | Clean, validate, reshape, aggregate, join | Remove duplicates, convert formats, calculate metrics, join datasets |
| **Load** | Write transformed data to the data warehouse | Bulk insert, upsert, append |

**ETL vs ELT:**

| Approach | Description | When to Use |
|----------|-------------|-------------|
| **ETL** | Transform before loading into warehouse | When warehouse compute is expensive; when transformations are complex |
| **ELT** | Load raw data first, transform inside the warehouse | Modern cloud warehouses (Snowflake, BigQuery) with cheap compute; faster iteration |

**ELT is the modern approach** — load raw data into the warehouse, then use SQL to transform it. Cloud warehouses have massive compute power, making in-warehouse transformations fast and cost-effective.

**ETL/ELT Tools:**

| Tool | Type | Description |
|------|------|-------------|
| **Apache Airflow** | Orchestrator | Schedule and monitor ETL pipelines (Python-based DAGs) |
| **dbt (data build tool)** | Transform (ELT) | SQL-based transformations inside the warehouse |
| **Apache Spark** | Processing engine | Large-scale data processing (batch and streaming) |
| **Fivetran / Airbyte** | Extract + Load | Managed connectors for extracting data from sources |
| **AWS Glue** | Managed ETL | Serverless ETL on AWS |
| **Kafka Connect** | Streaming ETL | Real-time data integration via Kafka |

---

### Column-Oriented Storage

OLAP queries typically read a few columns from millions of rows. **Column-oriented storage** stores data by column instead of by row, making these queries dramatically faster.

**Row-Oriented Storage (OLTP):**

```
Row storage on disk:
  [id=1, name="Alice", age=30, city="SF"]
  [id=2, name="Bob",   age=25, city="NYC"]
  [id=3, name="Charlie",age=35, city="LA"]

Query: SELECT AVG(age) FROM users;
  Must read entire rows (including name, city) to get age values
  Reads: 3 full rows × 4 columns = 12 values read
```

**Column-Oriented Storage (OLAP):**

```
Column storage on disk:
  id column:   [1, 2, 3]
  name column: ["Alice", "Bob", "Charlie"]
  age column:  [30, 25, 35]
  city column: ["SF", "NYC", "LA"]

Query: SELECT AVG(age) FROM users;
  Only reads the age column!
  Reads: 3 values (just the age column)
```

**Why Column Storage is Faster for Analytics:**

| Advantage | Explanation |
|-----------|-------------|
| **Read only needed columns** | Query reads only the columns it needs (not entire rows) |
| **Better compression** | Same-type values compress much better (all integers, all strings) |
| **Vectorized processing** | CPU can process a column of integers with SIMD instructions |
| **Cache efficiency** | Sequential access to same-type data is cache-friendly |

**Compression in Column Storage:**

```
city column (row-oriented): ["SF", "NYC", "LA", "SF", "SF", "NYC", "SF", "LA", "SF", "NYC"]
  → 10 strings, ~40 bytes

city column (column-oriented with dictionary encoding):
  Dictionary: {0: "SF", 1: "NYC", 2: "LA"}
  Values: [0, 1, 2, 0, 0, 1, 0, 2, 0, 1]
  → 10 integers + small dictionary, ~15 bytes

city column (with run-length encoding after sorting):
  Sorted: ["LA", "LA", "NYC", "NYC", "NYC", "SF", "SF", "SF", "SF", "SF"]
  RLE: [(LA, 2), (NYC, 3), (SF, 5)]
  → 3 pairs, ~12 bytes (70% compression!)
```

**Column-Oriented Databases:**

| Database | Type | Key Features |
|----------|------|-------------|
| **Snowflake** | Cloud data warehouse | Separation of storage and compute, auto-scaling |
| **Google BigQuery** | Cloud data warehouse | Serverless, pay-per-query, Dremel engine |
| **Amazon Redshift** | Cloud data warehouse | Columnar, MPP (Massively Parallel Processing) |
| **ClickHouse** | Open-source OLAP | Extremely fast for real-time analytics |
| **Apache Parquet** | File format | Columnar file format for data lakes (used with Spark, Hive) |
| **Apache ORC** | File format | Columnar file format (optimized for Hive) |
| **DuckDB** | Embedded OLAP | SQLite for analytics — in-process, columnar |

---

### Data Lake vs Data Warehouse

| Feature | Data Warehouse | Data Lake |
|---------|---------------|-----------|
| **Data type** | Structured (tables, schemas) | Structured + semi-structured + unstructured (JSON, CSV, images, logs) |
| **Schema** | Schema-on-write (defined before loading) | Schema-on-read (defined when querying) |
| **Processing** | SQL queries | SQL + Python + Spark + ML |
| **Users** | Business analysts, BI tools | Data engineers, data scientists |
| **Storage cost** | Higher (optimized storage) | Lower (cheap object storage like S3) |
| **Query performance** | Fast (optimized for queries) | Variable (depends on format and engine) |
| **Data quality** | High (cleaned, validated) | Variable (raw data, may be messy) |
| **Examples** | Snowflake, BigQuery, Redshift | S3 + Athena, GCS + BigQuery, Azure Data Lake |

**Data Lakehouse (Modern Approach):**
Combines the best of both — cheap storage of a data lake with the query performance and ACID transactions of a data warehouse.

```
Data Lakehouse Architecture:
  Raw Data (S3/GCS) → Open File Format (Parquet/ORC) → Table Format (Delta Lake/Iceberg/Hudi)
                                                         ↑
                                                    ACID transactions
                                                    Schema enforcement
                                                    Time travel
                                                    Efficient queries
```

| Technology | Description |
|-----------|-------------|
| **Delta Lake** (Databricks) | ACID transactions on data lakes, time travel, schema evolution |
| **Apache Iceberg** (Netflix) | Open table format with hidden partitioning, schema evolution |
| **Apache Hudi** (Uber) | Incremental processing, upserts on data lakes |

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| CAP Theorem | During a partition: choose Consistency (CP) or Availability (AP); P is non-negotiable |
| PACELC | Extends CAP: during normal operation, choose Latency or Consistency |
| CP Systems | MongoDB, HBase, CockroachDB, Spanner — reject requests during partitions |
| AP Systems | Cassandra, DynamoDB, CouchDB — serve stale data during partitions |
| ACID | Strong consistency — atomicity, consistency, isolation, durability; expensive in distributed systems |
| BASE | Eventual consistency — basically available, soft state; used by most distributed NoSQL |
| Replication | Primary-replica (simple, read scaling) vs multi-primary (write scaling, conflict resolution) |
| Sync vs Async | Sync = zero data loss, higher latency; Async = possible data loss, lower latency |
| Replication Lag | The delay between primary write and replica visibility; causes read-after-write inconsistency |
| Sharding | Split rows across servers; the shard key determines distribution and query routing |
| Shard Key | High cardinality, even distribution, query-aligned, immutable — the most important sharding decision |
| Range Sharding | Simple but causes hotspots on the latest shard |
| Hash Sharding | Even distribution but no range queries; consistent hashing avoids full resharding |
| Cross-Shard Queries | Scatter-gather across all shards — expensive; design to minimize them |
| WAL | Sequential append log for durability; enables fast commits and crash recovery |
| B-Tree | Read-optimized; in-place updates; used by PostgreSQL, MySQL |
| LSM-Tree | Write-optimized; sequential writes + compaction; used by Cassandra, RocksDB |
| SSTables | Immutable sorted files; merged by compaction; Bloom filters skip irrelevant files |
| Bloom Filters | Probabilistic "definitely not" or "maybe yes" — saves disk reads in LSM-Tree databases |
| Buffer Pool | Database-level page cache; size to 70-80% of RAM on dedicated servers |
| OLTP vs OLAP | OLTP = transactions (row-oriented); OLAP = analytics (column-oriented) |
| Star Schema | Fact table + dimension tables; denormalized dimensions; preferred for data warehouses |
| ETL vs ELT | ETL = transform before loading; ELT = load raw, transform in warehouse (modern approach) |
| Column Storage | Read only needed columns; better compression; vectorized processing; 10-100x faster for analytics |
| Data Lakehouse | Combines data lake (cheap storage) with warehouse (ACID, performance) — Delta Lake, Iceberg |

---

## Interview Tips for Module 20

1. **CAP Theorem** — explain with a concrete example; know which databases are CP vs AP; mention PACELC for bonus points
2. **ACID vs BASE** — explain when to use each; the hybrid approach (ACID for critical path, BASE for non-critical) is the best answer
3. **Replication** — draw primary-replica topology; explain sync vs async trade-offs; discuss replication lag problems and solutions
4. **Sharding** — explain the 4 strategies; design a shard key for a given system; discuss cross-shard query challenges
5. **Shard key selection** — this comes up in every HLD interview; practice choosing shard keys for e-commerce, social media, messaging
6. **B-Tree vs LSM-Tree** — explain the trade-offs; know which databases use which; recommend based on read-heavy vs write-heavy workload
7. **Bloom filters** — explain how they work and why they matter for LSM-Tree read performance
8. **WAL** — explain why sequential writes are fast and how WAL enables crash recovery
9. **OLTP vs OLAP** — know the differences; explain star schema; recommend column-oriented storage for analytics
10. **Consistency spectrum** — go beyond "strong vs eventual"; mention linearizable, causal, read-your-writes, monotonic reads
11. **Conflict resolution** — explain LWW, vector clocks, CRDTs; know the trade-offs of each
12. **Data lakehouse** — mention Delta Lake/Iceberg as the modern approach combining lakes and warehouses
