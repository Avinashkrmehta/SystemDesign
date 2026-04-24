# Module 27: Distributed Systems — Advanced

> This module covers the advanced building blocks of distributed systems — the algorithms and data structures that power databases like Cassandra and DynamoDB, caching systems like Redis Cluster, and big data platforms like Hadoop and Spark. Consistent hashing, gossip protocols, quorums, CRDTs, MapReduce, and probabilistic data structures are the tools that make planet-scale systems possible.

---

## 27.1 Consistent Hashing

> **Consistent hashing** is a technique for distributing data across a cluster of nodes such that adding or removing a node only requires moving a minimal fraction of the data. It's the foundation of distributed caches (Redis Cluster, Memcached), distributed databases (Cassandra, DynamoDB), and load balancers.

---

### The Problem with Simple Hashing

```
Simple hash-based distribution:
  node = hash(key) % N   (N = number of nodes)

  N = 3 nodes:
    hash("user:1") % 3 = 1 → Node 1
    hash("user:2") % 3 = 0 → Node 0
    hash("user:3") % 3 = 2 → Node 2

  Add a 4th node (N = 4):
    hash("user:1") % 4 = 1 → Node 1 (same)
    hash("user:2") % 4 = 2 → Node 2 (MOVED from Node 0!)
    hash("user:3") % 4 = 3 → Node 3 (MOVED from Node 2!)

  Almost ALL keys are remapped! For a cache, this means almost all entries
  become cache misses → thundering herd to the database.
  
  With N nodes, adding one node remaps ~(N-1)/N of all keys.
  For 100 nodes: ~99% of keys are remapped!
```

---

### Hash Ring

Consistent hashing maps both **nodes** and **keys** onto a circular hash space (ring). Each key is assigned to the **first node encountered clockwise** on the ring.

```
Hash Ring (0 to 2^32 - 1, wrapping around):

                    Node A (pos: 1000)
                   /
                  /
  Node D (pos: 750) ←─── Key "user:5" (hash: 800) → goes to Node A
                  \
                   \
                    Node B (pos: 250)
                   /
                  /
  Node C (pos: 500) ←─── Key "user:3" (hash: 400) → goes to Node C
```

```
Position on ring:  0 ──── 250 ──── 500 ──── 750 ──── 1000 ──── 0 (wraps)
                          B         C         D          A
                          
Key hash = 100 → clockwise → first node is B (250)
Key hash = 300 → clockwise → first node is C (500)
Key hash = 600 → clockwise → first node is D (750)
Key hash = 800 → clockwise → first node is A (1000)
Key hash = 950 → clockwise → wraps → first node is B (250)
```

---

### Adding/Removing Nodes

**Adding Node E at position 600:**

```
Before:
  Keys 500-750 → Node D
  
After:
  Keys 500-600 → Node E (NEW — only these keys move!)
  Keys 600-750 → Node D (unchanged)
  
  All other keys stay on their current nodes.
  Only keys between C (500) and E (600) are affected.
  That's approximately 1/N of all keys (where N = number of nodes).
```

```
Before:  ... ── C(500) ────────────── D(750) ── ...
                         ↑
                    Keys 500-750 → D

After:   ... ── C(500) ── E(600) ── D(750) ── ...
                    ↑          ↑
              Keys 500-600  Keys 600-750
                  → E          → D (unchanged)
```

**Removing Node C (position 500):**

```
Before:
  Keys 250-500 → Node C
  
After:
  Keys 250-500 → Node D (next clockwise node absorbs C's keys)
  
  Only C's keys are redistributed. All other keys stay put.
```

**Key Advantage:** Adding or removing a node only moves ~1/N of the keys (where N is the number of nodes). With simple `hash % N`, adding a node moves ~(N-1)/N of the keys.

| Operation | Simple Hash (% N) | Consistent Hashing |
|-----------|-------------------|-------------------|
| Add 1 node to 100 | ~99% keys remapped | ~1% keys remapped |
| Remove 1 node from 100 | ~99% keys remapped | ~1% keys remapped |

---

### Virtual Nodes (vnodes)

With only a few physical nodes, the distribution on the ring can be **uneven** — one node might own a much larger arc than others.

```
Problem (3 physical nodes, uneven distribution):
  Node A at position 100 → owns arc 900-100 (200 units)
  Node B at position 300 → owns arc 100-300 (200 units)
  Node C at position 900 → owns arc 300-900 (600 units!)  ← 3x more data than A or B!
```

**Solution: Virtual Nodes**

Each physical node is mapped to **multiple positions** on the ring (virtual nodes). This spreads each node's responsibility across the ring, resulting in a more even distribution.

```
Physical Node A → Virtual nodes at positions: 50, 350, 650, 850
Physical Node B → Virtual nodes at positions: 150, 450, 550, 950
Physical Node C → Virtual nodes at positions: 250, 400, 700, 800

Ring: 50(A) 150(B) 250(C) 350(A) 400(C) 450(B) 550(B) 650(A) 700(C) 800(C) 850(A) 950(B)

Each physical node owns ~4/12 = 33% of the ring (much more even!)
```

**Virtual Node Benefits:**

| Benefit | Description |
|---------|-------------|
| **Even distribution** | More virtual nodes → more even data distribution |
| **Heterogeneous nodes** | Powerful nodes get more virtual nodes → handle more data proportionally |
| **Smoother rebalancing** | Adding a node adds multiple virtual nodes → takes small chunks from many nodes instead of one big chunk from one node |

**Typical Configuration:** 100-256 virtual nodes per physical node.

---

### Applications

| System | How It Uses Consistent Hashing |
|--------|-------------------------------|
| **Cassandra** | Partition key → token (hash) → ring position → responsible node(s) |
| **DynamoDB** | Partition key → hash → partition → storage node |
| **Redis Cluster** | 16,384 hash slots distributed across masters (CRC16 hash) |
| **Memcached** | Client-side consistent hashing for key distribution |
| **Akamai CDN** | Route requests to nearest cache server |
| **Amazon S3** | Distribute objects across storage nodes |
| **Load Balancers** | Consistent hashing for session affinity (same client → same server) |

---


## 27.2 Gossip Protocol

> The **gossip protocol** (also called epidemic protocol) is a peer-to-peer communication mechanism where nodes periodically exchange state information with random peers. Like how rumors spread in a social network, information eventually reaches all nodes through repeated random exchanges. It's used for failure detection, membership management, and data dissemination in systems like Cassandra, DynamoDB, and Consul.

---

### How Gossip Works

```
Every T seconds (gossip interval, typically 1 second):
  1. Node A randomly selects a peer (Node C)
  2. Node A sends its state to Node C
  3. Node C merges A's state with its own
  4. Node C sends its merged state back to A
  5. Node A merges C's state with its own
  
  Both nodes now have the union of their knowledge.
```

```
Time T1: Node A knows: {A: alive, B: alive}
         Node C knows: {C: alive, D: alive}
         
         A gossips with C:
         A sends {A: alive, B: alive} to C
         C merges → {A: alive, B: alive, C: alive, D: alive}
         C sends {A: alive, B: alive, C: alive, D: alive} to A
         A merges → {A: alive, B: alive, C: alive, D: alive}
         
Time T1: Both A and C now know about all 4 nodes!

Time T2: A gossips with D, C gossips with B
         → After 2 rounds, ALL nodes know about ALL other nodes
```

**Convergence Speed:**
- With N nodes, information reaches all nodes in **O(log N)** gossip rounds
- 1000 nodes → ~10 rounds (10 seconds with 1-second interval)
- 1,000,000 nodes → ~20 rounds (20 seconds)
- This is remarkably fast and scales logarithmically

**Gossip Variants:**

| Variant | Description | Use Case |
|---------|-------------|----------|
| **Push** | Sender pushes its state to a random peer | Fast initial spread |
| **Pull** | Node requests state from a random peer | Efficient when most nodes already have the info |
| **Push-Pull** | Both sides exchange state (most common) | Best convergence; used by Cassandra, Consul |

---

### Failure Detection

Gossip protocols are commonly used for **failure detection** — determining which nodes in the cluster are alive and which have failed.

**Heartbeat-Based Detection:**

```
Each node maintains a heartbeat counter that it increments periodically.
Nodes gossip their heartbeat counters to peers.

Node A's view:
  Node A: heartbeat=150 (local, always incrementing)
  Node B: heartbeat=148 (received via gossip 2 seconds ago)
  Node C: heartbeat=145 (received via gossip 5 seconds ago)
  Node D: heartbeat=130 (received via gossip 20 seconds ago — SUSPICIOUS!)

If a node's heartbeat hasn't increased for T_fail seconds → mark as SUSPECTED
If still no update after T_cleanup seconds → mark as DEAD and remove
```

**Phi Accrual Failure Detector (Cassandra):**

Instead of a binary alive/dead decision, the phi accrual detector calculates a **suspicion level** (φ) based on the statistical distribution of heartbeat arrival times.

```
φ = -log10(probability that the node is still alive)

φ = 1  → 10% chance the node has failed (probably fine)
φ = 2  → 1% chance the node is alive (probably failed)
φ = 5  → 0.001% chance the node is alive (definitely failed)
φ = 8  → threshold in Cassandra (mark as down)

Advantages:
  - Adapts to network conditions (high-latency networks get higher thresholds)
  - No fixed timeout — uses statistical analysis of actual heartbeat intervals
  - Fewer false positives than fixed-timeout detectors
```

---

### Membership Protocol

The gossip protocol maintains a **membership list** — the set of nodes currently in the cluster.

```
Membership State per Node:
  +--------+--------+----------+------------------+
  | Node   | Status | Heartbeat| Last Updated     |
  +--------+--------+----------+------------------+
  | Node A | ALIVE  | 150      | 2025-03-15 10:00 |
  | Node B | ALIVE  | 148      | 2025-03-15 09:59 |
  | Node C | SUSPECT| 145      | 2025-03-15 09:55 |
  | Node D | DEAD   | 130      | 2025-03-15 09:40 |
  +--------+--------+----------+------------------+

Node Lifecycle:
  JOIN → ALIVE → SUSPECT (heartbeat timeout) → DEAD (cleanup timeout) → REMOVED
                    ↓
              ALIVE (heartbeat resumes — false alarm)
```

**SWIM Protocol (Scalable Weakly-consistent Infection-style Membership):**
An improved gossip-based membership protocol that combines:
1. **Direct probing:** Ping a random node directly
2. **Indirect probing:** If direct ping fails, ask K other nodes to ping the suspect
3. **Dissemination:** Piggyback membership updates on ping/ack messages (no extra messages)

Used by: Consul (via Serf), Memberlist (HashiCorp)

---

### Gossip Protocol Properties

| Property | Description |
|----------|-------------|
| **Scalable** | O(log N) convergence; each node communicates with a fixed number of peers per round |
| **Fault tolerant** | No single point of failure; works even if many nodes fail |
| **Decentralized** | No leader or coordinator; all nodes are equal |
| **Eventually consistent** | All nodes eventually converge to the same state |
| **Simple** | Easy to implement; no complex coordination logic |
| **Bandwidth efficient** | Each node sends a small, fixed amount of data per round |

**Gossip Trade-offs:**

| Pros | Cons |
|------|------|
| Highly scalable (logarithmic convergence) | Eventually consistent (not immediate) |
| No single point of failure | Convergence time is probabilistic (not guaranteed) |
| Tolerates network partitions | Bandwidth overhead (constant background traffic) |
| Simple to implement | False positives in failure detection |

---

### Systems Using Gossip

| System | What Gossip Is Used For |
|--------|------------------------|
| **Cassandra** | Cluster membership, failure detection, schema changes, token metadata |
| **DynamoDB** | Membership, failure detection (internal) |
| **Consul** | Cluster membership (Serf), failure detection, event broadcasting |
| **Redis Cluster** | Node discovery, failure detection, slot migration |
| **Riak** | Ring state, membership, handoff coordination |
| **CockroachDB** | Node liveness, cluster metadata |
| **Amazon S3** | Internal cluster management |

---


## 27.3 Quorum-Based Systems

> A **quorum** is the minimum number of nodes that must participate in a read or write operation for it to be considered successful. Quorum-based systems allow you to tune the trade-off between consistency and availability on a per-operation basis.

---

### Read Quorum, Write Quorum

In a system with **N replicas**, you configure:
- **W** = number of replicas that must acknowledge a **write** (write quorum)
- **R** = number of replicas that must respond to a **read** (read quorum)

```
N = 3 replicas (Node A, Node B, Node C)

Write with W=2:
  Client → write to Node A → ACK ✅
  Client → write to Node B → ACK ✅  (W=2 achieved → write succeeds)
  Client → write to Node C → (async, may succeed later)

Read with R=2:
  Client → read from Node A → value=5 (latest)
  Client → read from Node B → value=5 (latest)  (R=2 achieved → return 5)
  (Node C might still have value=3 — stale — but we don't need it)
```

---

### W + R > N for Strong Consistency

The **quorum intersection formula**: if `W + R > N`, then every read is guaranteed to see the latest write, because the read quorum and write quorum must **overlap** — at least one node has the latest value.

```
N = 3, W = 2, R = 2:  W + R = 4 > 3 → STRONGLY CONSISTENT

Write quorum: {A, B}     (2 of 3 nodes have the latest value)
Read quorum:  {B, C}     (reads from 2 of 3 nodes)
Overlap:      {B}        (Node B has the latest value → read returns it)

No matter which 2 nodes you read from, at least 1 was in the write quorum.
```

```
Visualization (N=3):

  Write quorum (W=2):     Read quorum (R=2):      Overlap:
  +---+  +---+  +---+     +---+  +---+  +---+     +---+  +---+  +---+
  | A |  | B |  | C |     | A |  | B |  | C |     | A |  | B |  | C |
  | ✅ |  | ✅ |  |   |     |   |  | ✅ |  | ✅ |     |   |  | ✅ |  |   |
  +---+  +---+  +---+     +---+  +---+  +---+     +---+  +---+  +---+
  
  B is in BOTH quorums → read always sees the latest write
```

**Common Quorum Configurations:**

| Configuration | W | R | W+R vs N | Consistency | Availability | Use Case |
|--------------|---|---|----------|-------------|-------------|----------|
| **Strong consistency** | ⌊N/2⌋+1 | ⌊N/2⌋+1 | > N | Strong | Moderate | Default for most systems |
| **Write-heavy** | 1 | N | > N | Strong | Low for reads | Logging, metrics (fast writes) |
| **Read-heavy** | N | 1 | > N | Strong | Low for writes | Read-heavy with rare writes |
| **Eventual consistency** | 1 | 1 | < N | Eventual | Highest | When staleness is acceptable |

**Example with N=3:**

| W | R | W+R | Consistency | Trade-off |
|---|---|-----|-------------|-----------|
| 1 | 1 | 2 < 3 | Eventual | Fastest reads and writes; may read stale data |
| 1 | 3 | 4 > 3 | Strong | Fast writes, slow reads (must read all replicas) |
| 3 | 1 | 4 > 3 | Strong | Slow writes (all replicas), fast reads |
| 2 | 2 | 4 > 3 | Strong | Balanced; tolerates 1 node failure for both reads and writes |
| 1 | 2 | 3 = 3 | Strong* | Borderline; works if no concurrent writes |

---

### Sloppy Quorum

A **strict quorum** requires W or R responses from the **designated replicas** for a key. If a designated replica is down, the operation fails.

A **sloppy quorum** allows the operation to succeed by writing to **any available node** — even nodes that don't normally own that key. This improves availability at the cost of consistency.

```
Strict Quorum (N=3, W=2):
  Key "user:123" → designated replicas: Node A, Node B, Node C
  Node C is down → can only write to A and B → W=2 achieved → OK
  Node B AND C are down → can only write to A → W=1 < 2 → FAIL!

Sloppy Quorum (N=3, W=2):
  Key "user:123" → designated replicas: Node A, Node B, Node C
  Node B AND C are down → write to A and Node D (not a designated replica) → W=2 achieved → OK!
  Node D holds the data temporarily until B or C recover (hinted handoff)
```

| Feature | Strict Quorum | Sloppy Quorum |
|---------|--------------|---------------|
| Availability | Lower (designated replicas must be available) | Higher (any node can participate) |
| Consistency | Stronger (W+R>N guarantees overlap) | Weaker (non-designated nodes may not be read) |
| Use case | When consistency is critical | When availability is critical (Dynamo-style systems) |

---

### Hinted Handoff

When a sloppy quorum writes to a non-designated node, that node stores the data as a **hint** — a temporary copy that should be forwarded to the correct node when it recovers.

```
1. Key "user:123" should be on Nodes A, B, C
2. Node C is down → write goes to Node D (with a hint: "this belongs to Node C")
3. Node D stores: { key: "user:123", value: ..., hint: "deliver to Node C" }
4. Node C recovers
5. Node D detects C is back → forwards the data to Node C
6. Node D deletes the hint
7. Node C now has the correct data
```

```
  Node C down:
    Client → Node A: write ✅
    Client → Node D: write ✅ (hint: "for Node C")
    
  Node C recovers:
    Node D → Node C: "Here's data that belongs to you"
    Node D: delete hint
    Node C: now has the data ✅
```

**Hinted Handoff Properties:**
- Improves **write availability** — writes succeed even when some replicas are down
- Data is **eventually** delivered to the correct node
- Hints have a **TTL** — if the node doesn't recover within the TTL, the hint is discarded (data loss risk)
- Does NOT guarantee consistency — the hinted node may not be queried during reads

**Used by:** Cassandra, DynamoDB, Riak

---


## 27.4 Conflict Resolution

> In distributed systems with multiple writers (multi-master replication, sloppy quorums), **conflicts** are inevitable — two nodes may update the same data simultaneously. The system must detect and resolve these conflicts. This section covers the major conflict resolution strategies.

---

### Last-Write-Wins (LWW)

The simplest strategy: each write has a timestamp, and the write with the **latest timestamp wins**. All other writes are silently discarded.

```
Node A: SET user:123 = "Alice" at T=1000
Node B: SET user:123 = "Bob"   at T=1001

Resolution: T=1001 > T=1000 → "Bob" wins
Result: user:123 = "Bob"
"Alice" is silently lost!
```

| Pros | Cons |
|------|------|
| Simplest to implement | **Silent data loss** — losing writes without notification |
| No conflict detection needed | Depends on synchronized clocks (clock skew → wrong winner) |
| Automatic resolution | Concurrent writes are not detected — one is arbitrarily discarded |
| Low overhead | Not suitable when all writes matter (e.g., adding items to a cart) |

**Used by:** Cassandra (default), DynamoDB (default for conflicting writes)

**When LWW is Acceptable:**
- Immutable data (write once, never update)
- Last-state-wins semantics (user profile — latest update is the "correct" one)
- Idempotent operations (setting a value, not incrementing)

**When LWW is NOT Acceptable:**
- Counters (incrementing — lost increments)
- Sets (adding items — lost additions)
- Any operation where all writes must be preserved

---

### Vector Clocks (for Conflict Detection)

Vector clocks (covered in Module 26.6) detect **concurrent writes** — writes that happened independently on different nodes without knowledge of each other.

```
Node A: SET user:123 = "Alice", vector = [A:1, B:0]
Node B: SET user:123 = "Bob",   vector = [A:0, B:1]

Compare vectors:
  [A:1, B:0] vs [A:0, B:1]
  Neither dominates → CONCURRENT CONFLICT DETECTED!

Resolution options:
  1. Return BOTH versions to the client → client resolves (Amazon's shopping cart approach)
  2. Merge automatically (if possible)
  3. Use application-specific logic
```

**Amazon Dynamo's Approach:**
- Detect conflicts using vector clocks
- Return ALL conflicting versions (called "siblings") to the client
- Client merges the siblings and writes back the resolved version
- Example: shopping cart — merge = union of all items from all versions

---

### CRDTs (Conflict-free Replicated Data Types)

**CRDTs** are data structures that can be **replicated across multiple nodes** and **merged automatically** without conflicts. They are mathematically guaranteed to converge to the same state, regardless of the order in which updates are applied.

**Key Property:** For any two replicas, merging their states always produces the same result, regardless of the order of operations or the order of merges. This is achieved by requiring operations to be **commutative**, **associative**, and **idempotent**.

---

**G-Counter (Grow-only Counter):**

Each node maintains its own counter. The total is the sum of all node counters.

```
3 nodes: A, B, C

Node A increments 3 times: {A: 3, B: 0, C: 0}
Node B increments 5 times: {A: 0, B: 5, C: 0}
Node C increments 2 times: {A: 0, B: 0, C: 2}

Merge: take MAX of each entry:
  {A: max(3,0,0), B: max(0,5,0), C: max(0,0,2)} = {A: 3, B: 5, C: 2}
  Total = 3 + 5 + 2 = 10

No conflicts! Order of merges doesn't matter!
```

---

**PN-Counter (Positive-Negative Counter):**

Two G-Counters: one for increments (P), one for decrements (N). Value = P - N.

```
Node A: increment 5 times, decrement 2 times
  P = {A: 5, B: 0, C: 0}
  N = {A: 2, B: 0, C: 0}
  Value = 5 - 2 = 3

Node B: increment 3 times, decrement 1 time
  P = {A: 0, B: 3, C: 0}
  N = {A: 0, B: 1, C: 0}
  Value = 3 - 1 = 2

Merge:
  P = {A: 5, B: 3, C: 0} → sum = 8
  N = {A: 2, B: 1, C: 0} → sum = 3
  Value = 8 - 3 = 5
```

---

**G-Set (Grow-only Set):**

Elements can only be **added**, never removed. Merge = union of all elements.

```
Node A: {apple, banana}
Node B: {banana, cherry}

Merge: {apple, banana, cherry}  (union — no conflicts possible)
```

---

**OR-Set (Observed-Remove Set):**

Elements can be added AND removed. Each add is tagged with a unique ID. A remove only removes the specific tagged additions that the remover has observed.

```
Node A: add("apple", tag=1), add("banana", tag=2)
  Set: {(apple, 1), (banana, 2)}

Node B: observes {(apple, 1), (banana, 2)}, removes "apple"
  Removes: (apple, 1)
  Set: {(banana, 2)}

Node A: add("apple", tag=3)  (concurrent with B's remove)
  Set: {(apple, 1), (banana, 2), (apple, 3)}

Merge A and B:
  A: {(apple, 1), (banana, 2), (apple, 3)}
  B: {(banana, 2)}  (removed (apple, 1))
  
  Result: {(banana, 2), (apple, 3)}
  
  (apple, 1) was removed by B → gone
  (apple, 3) was added by A after B's observation → kept!
  "Add wins" over concurrent remove — intuitive behavior
```

---

**LWW-Register (Last-Writer-Wins Register):**

A single value with a timestamp. The value with the latest timestamp wins.

```
Node A: SET value="Alice" at T=1000
Node B: SET value="Bob" at T=1001

Merge: T=1001 > T=1000 → value = "Bob"
```

This is the CRDT formalization of LWW. It's a valid CRDT because the merge function (take the latest timestamp) is commutative, associative, and idempotent.

---

**CRDT Summary:**

| CRDT | Operations | Merge Strategy | Use Case |
|------|-----------|---------------|----------|
| **G-Counter** | Increment only | Sum of per-node counters | Page views, likes (no decrement) |
| **PN-Counter** | Increment + Decrement | P counter - N counter | Upvotes/downvotes, inventory |
| **G-Set** | Add only | Union | Tags, followers (no unfollow) |
| **OR-Set** | Add + Remove | Union with tombstones | Shopping cart, friend lists |
| **LWW-Register** | Set value | Latest timestamp wins | User profile, settings |
| **LWW-Map** | Set key-value | LWW per key | Configuration, metadata |
| **MV-Register** | Set value | Keep all concurrent values | When all versions matter |

**CRDT Trade-offs:**

| Pros | Cons |
|------|------|
| Automatic conflict resolution — no application logic needed | Limited data types (not all operations can be CRDTs) |
| Mathematically guaranteed convergence | Higher storage overhead (metadata: tags, vectors, tombstones) |
| No coordination needed (no consensus, no locking) | Tombstone accumulation (deleted items leave metadata) |
| Works offline (merge when reconnected) | Semantics may be surprising (OR-Set "add wins" over remove) |

**Used by:** Redis (CRDT module for active-active geo-replication), Riak, Automerge (collaborative editing), Yjs (collaborative editing)

---

### Application-Level Conflict Resolution

When automatic resolution isn't possible, return all conflicting versions to the **application** and let it decide.

```
Database returns: [version_A: "Alice", version_B: "Bob"]

Application logic:
  if both are user profile updates:
    merge fields: take non-null fields from each version
  if both are shopping cart updates:
    merge items: union of items from both versions
  if both are counter updates:
    sum the deltas from both versions
  if irreconcilable:
    prompt the user to choose (like Git merge conflicts)
```

**Used by:** CouchDB (returns all conflicting revisions), Amazon Dynamo (returns siblings)

---


## 27.5 MapReduce & Distributed Computing

> **MapReduce** is a programming model for processing large datasets in parallel across a cluster of machines. It was popularized by Google's 2004 paper and implemented in Apache Hadoop. While MapReduce itself has been largely superseded by Apache Spark, the concepts remain fundamental to distributed data processing.

---

### MapReduce Paradigm

MapReduce breaks a computation into two phases:
1. **Map:** Process each input record independently, producing key-value pairs
2. **Reduce:** Group all values by key and combine them

```
Input Data (web server logs):
  "GET /home 200"
  "GET /about 200"
  "GET /home 200"
  "POST /login 401"
  "GET /home 200"
  "GET /about 404"

Goal: Count page views per URL

MAP Phase (runs on each input record independently):
  "GET /home 200"   → ("/home", 1)
  "GET /about 200"  → ("/about", 1)
  "GET /home 200"   → ("/home", 1)
  "POST /login 401" → ("/login", 1)
  "GET /home 200"   → ("/home", 1)
  "GET /about 404"  → ("/about", 1)

SHUFFLE Phase (group by key — handled by the framework):
  "/home"  → [1, 1, 1]
  "/about" → [1, 1]
  "/login" → [1]

REDUCE Phase (combine values for each key):
  "/home"  → sum([1, 1, 1]) = 3
  "/about" → sum([1, 1])    = 2
  "/login" → sum([1])       = 1

Output:
  /home: 3
  /about: 2
  /login: 1
```

---

### Map Phase, Shuffle Phase, Reduce Phase

```
+------------------+     +------------------+     +------------------+
|   Input Data     |     |   Intermediate   |     |   Output Data    |
|   (distributed)  |     |   (shuffled)     |     |   (aggregated)   |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
    +----v----+              +----v----+              +----v----+
    |  MAP    |              | SHUFFLE |              | REDUCE  |
    | Worker 1|              | (sort & |              | Worker 1|
    +---------+              | group   |              +---------+
    +----v----+              | by key) |              +----v----+
    |  MAP    |              +---------+              | REDUCE  |
    | Worker 2|                                       | Worker 2|
    +---------+                                       +---------+
    +----v----+                                       +----v----+
    |  MAP    |                                       | REDUCE  |
    | Worker 3|                                       | Worker 3|
    +---------+                                       +---------+
```

**Phase Details:**

| Phase | What Happens | Parallelism |
|-------|-------------|-------------|
| **Map** | Each worker processes a chunk of input data; applies the map function to each record; outputs key-value pairs | Embarrassingly parallel — each record is independent |
| **Shuffle** | Framework sorts and groups intermediate key-value pairs by key; transfers data between map and reduce workers | Network-intensive — data moves between machines |
| **Reduce** | Each worker receives all values for a subset of keys; applies the reduce function to combine values | Parallel by key — each key is independent |

**Classic MapReduce Examples:**

| Problem | Map Output | Reduce Operation |
|---------|-----------|-----------------|
| Word count | (word, 1) | Sum all 1s per word |
| Inverted index | (word, document_id) | Collect all doc IDs per word |
| Sort | (sort_key, record) | Identity (shuffle does the sorting) |
| Join | (join_key, tagged_record) | Match records with same key from different tables |
| PageRank | (page, partial_rank) | Sum partial ranks per page |

---

### Hadoop Ecosystem (Overview)

**Apache Hadoop** is the open-source implementation of MapReduce + distributed storage.

```
Hadoop Ecosystem:

  +----------------------------------------------------------+
  |  Applications: Hive (SQL), Pig (scripting), Mahout (ML)  |
  +----------------------------------------------------------+
  |  Processing: MapReduce / Tez / Spark                      |
  +----------------------------------------------------------+
  |  Resource Management: YARN (Yet Another Resource Negotiator)|
  +----------------------------------------------------------+
  |  Storage: HDFS (Hadoop Distributed File System)           |
  +----------------------------------------------------------+
```

| Component | Purpose |
|-----------|---------|
| **HDFS** | Distributed file system; stores data in 128 MB blocks across nodes; 3x replication |
| **YARN** | Resource manager; schedules and manages compute resources across the cluster |
| **MapReduce** | Original processing engine; batch-only, disk-based (slow) |
| **Hive** | SQL interface on top of Hadoop; translates SQL to MapReduce/Tez/Spark jobs |
| **Pig** | Scripting language for data transformation (largely replaced by Spark) |
| **HBase** | Column-family database on top of HDFS (like Cassandra but on Hadoop) |
| **ZooKeeper** | Distributed coordination (leader election, locks, config) |

**Hadoop Limitations:**
- MapReduce writes intermediate results to disk between stages → slow
- Not suitable for iterative algorithms (ML) — each iteration is a full MapReduce job
- High latency — batch processing only, not real-time
- Complex to operate and tune

---

### Spark (Overview)

**Apache Spark** is the successor to Hadoop MapReduce. It processes data **in memory**, making it 10-100x faster for many workloads.

```
Spark Architecture:

  +------------------+
  | Driver Program   |  (your code — defines the computation)
  +--------+---------+
           |
  +--------v---------+
  | Cluster Manager  |  (YARN, Kubernetes, Mesos, or Spark Standalone)
  +--------+---------+
           |
  +--------v---------+  +------------------+  +------------------+
  | Executor 1       |  | Executor 2       |  | Executor 3       |
  | (Worker Node)    |  | (Worker Node)    |  | (Worker Node)    |
  | - Tasks          |  | - Tasks          |  | - Tasks          |
  | - Cache (RAM)    |  | - Cache (RAM)    |  | - Cache (RAM)    |
  +------------------+  +------------------+  +------------------+
```

**Spark vs MapReduce:**

| Feature | MapReduce | Spark |
|---------|-----------|-------|
| Processing | Disk-based (read/write between stages) | In-memory (cache intermediate results) |
| Speed | Slow (disk I/O between stages) | 10-100x faster (in-memory) |
| Iterative algorithms | Very slow (full job per iteration) | Fast (cache data in memory across iterations) |
| Real-time | Batch only | Batch + streaming (Spark Streaming, Structured Streaming) |
| API | Low-level (Map + Reduce functions) | High-level (DataFrame, SQL, ML pipelines) |
| Languages | Java | Scala, Python, Java, R, SQL |
| Ecosystem | Hadoop only | Standalone, YARN, Kubernetes, Mesos |

**Spark Components:**

| Component | Purpose |
|-----------|---------|
| **Spark Core** | Distributed task scheduling, memory management, fault recovery |
| **Spark SQL** | SQL queries on structured data (DataFrames) |
| **Spark Streaming** | Real-time stream processing (micro-batches) |
| **MLlib** | Machine learning library (classification, regression, clustering) |
| **GraphX** | Graph processing (PageRank, connected components) |

**Key Point for System Design:** MapReduce/Spark are for **batch processing** of large datasets (ETL, analytics, ML training). They're not for real-time request handling. In system design interviews, mention Spark for offline data processing pipelines, not for serving user requests.

---


## 27.6 Bloom Filters & Probabilistic Data Structures

> **Probabilistic data structures** trade perfect accuracy for dramatic improvements in space and time efficiency. They answer questions like "have I seen this element before?" or "how many unique elements are there?" using a fraction of the memory that exact data structures would require. They're used extensively in databases, caches, networking, and big data systems.

---

### Bloom Filter

A **Bloom filter** is a space-efficient probabilistic data structure that tests whether an element is a **member of a set**. It can tell you:
- **"Definitely NOT in the set"** — guaranteed correct (no false negatives)
- **"Probably in the set"** — might be wrong (possible false positives)

**How It Works:**

```
1. Start with a bit array of M bits, all set to 0:
   [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]  (M=10)

2. Use K hash functions (K=3 in this example)

3. INSERT "apple":
   hash1("apple") % 10 = 2
   hash2("apple") % 10 = 5
   hash3("apple") % 10 = 8
   Set bits 2, 5, 8 to 1:
   [0, 0, 1, 0, 0, 1, 0, 0, 1, 0]

4. INSERT "banana":
   hash1("banana") % 10 = 1
   hash2("banana") % 10 = 5  (already 1 — no change)
   hash3("banana") % 10 = 7
   Set bits 1, 5, 7 to 1:
   [0, 1, 1, 0, 0, 1, 0, 1, 1, 0]

5. QUERY "apple":
   Check bits 2, 5, 8 → all are 1 → "PROBABLY in the set" ✅

6. QUERY "cherry":
   hash1("cherry") % 10 = 3
   hash2("cherry") % 10 = 6
   hash3("cherry") % 10 = 8
   Check bits 3, 6, 8 → bit 3 is 0 → "DEFINITELY NOT in the set" ✅

7. QUERY "grape":
   hash1("grape") % 10 = 1
   hash2("grape") % 10 = 2
   hash3("grape") % 10 = 5
   Check bits 1, 2, 5 → all are 1 → "PROBABLY in the set"
   But "grape" was never inserted! This is a FALSE POSITIVE.
   (Bits 1, 2, 5 were set by "apple" and "banana" coincidentally)
```

**False Positive Rate:**

```
Probability of false positive ≈ (1 - e^(-kn/m))^k

Where:
  m = number of bits in the array
  n = number of elements inserted
  k = number of hash functions

For 1% false positive rate with 1 million elements:
  m ≈ 9.6 million bits ≈ 1.2 MB
  k ≈ 7 hash functions

For 0.1% false positive rate with 1 million elements:
  m ≈ 14.4 million bits ≈ 1.8 MB
  k ≈ 10 hash functions
```

**Bloom Filter Properties:**

| Property | Value |
|----------|-------|
| Space | O(m) bits — much smaller than storing actual elements |
| Insert | O(k) — compute k hashes, set k bits |
| Query | O(k) — compute k hashes, check k bits |
| False negatives | **IMPOSSIBLE** — if an element was inserted, the query always returns "probably yes" |
| False positives | Possible — probability depends on m, n, k |
| Delete | **NOT SUPPORTED** — can't unset bits (other elements may share them) |

**Bloom Filter Use Cases:**

| Use Case | How It's Used |
|----------|--------------|
| **Database reads (LSM-Tree)** | Check if a key might be in an SSTable before reading from disk (saves disk I/O) |
| **Web crawlers** | Check if a URL has already been crawled (avoid re-crawling) |
| **Spam filtering** | Check if an email address is in the spam list |
| **CDN cache** | Check if a resource is cached before querying the origin |
| **Weak password detection** | Check if a password is in a list of known compromised passwords |
| **Network routing** | Check if a packet should be forwarded (Bloom filter in router) |
| **Duplicate detection** | Check if a message/event has already been processed |

---

### Counting Bloom Filter

A **counting Bloom filter** replaces each bit with a **counter**, enabling **deletions**.

```
Standard Bloom Filter:  [0, 1, 1, 0, 0, 1, 0, 1, 1, 0]  (bits — can't delete)

Counting Bloom Filter:  [0, 2, 1, 0, 0, 3, 0, 1, 2, 0]  (counters)

INSERT "apple" (positions 2, 5, 8): increment counters at 2, 5, 8
DELETE "apple" (positions 2, 5, 8): decrement counters at 2, 5, 8

A position is "set" if its counter > 0
```

| Pros | Cons |
|------|------|
| Supports deletion | 3-4x more memory (counters instead of bits) |
| Same false positive guarantees | Counter overflow risk (use 4-bit counters minimum) |
| Same query performance | More complex implementation |

---

### HyperLogLog (Cardinality Estimation)

**HyperLogLog** estimates the number of **distinct elements** (cardinality) in a dataset using very little memory.

```
Problem: How many unique visitors did our website have today?

Exact approach: Store all visitor IDs in a set → 100 million IDs × 16 bytes = 1.6 GB
HyperLogLog:   ~12 KB of memory → estimates 100 million unique visitors with ~0.81% error!
```

**How It Works (Simplified):**

```
Observation: If you hash random values, the probability of seeing a hash
that starts with k leading zeros is 1/2^k.

If the longest run of leading zeros you've seen is k,
you've probably seen about 2^k distinct elements.

Example:
  hash("user_1") = 0010110...  → 2 leading zeros
  hash("user_2") = 0000011...  → 5 leading zeros
  hash("user_3") = 0100101...  → 1 leading zero
  hash("user_4") = 0000001...  → 6 leading zeros  ← maximum!

  Maximum leading zeros = 6 → estimate ≈ 2^6 = 64 distinct elements

HyperLogLog improves accuracy by:
  1. Splitting the hash space into M buckets (registers)
  2. Each bucket tracks the maximum leading zeros for its subset
  3. The final estimate is the harmonic mean of all bucket estimates
  4. Bias correction for small and large cardinalities
```

**HyperLogLog Properties:**

| Property | Value |
|----------|-------|
| Memory | ~12 KB for billions of elements (with 16,384 registers) |
| Error rate | ~0.81% standard error (with 16,384 registers) |
| Operations | Add element: O(1), Count: O(M), Merge: O(M) |
| Merge | Two HyperLogLogs can be merged (union of sets) |

**Use Cases:**
- Counting unique visitors per page/day
- Counting unique search queries
- Counting distinct IP addresses
- Any "count distinct" operation on large datasets

**Used by:** Redis (`PFADD`, `PFCOUNT`, `PFMERGE`), Presto/Trino, BigQuery (APPROX_COUNT_DISTINCT), Elasticsearch

---

### Count-Min Sketch (Frequency Estimation)

**Count-Min Sketch** estimates the **frequency** of elements in a data stream — "how many times has this element appeared?"

```
Problem: What are the top 10 most searched queries today?

Exact approach: HashMap of query → count → unbounded memory
Count-Min Sketch: Fixed memory → approximate counts with bounded error
```

**How It Works:**

```
D hash functions, each mapping to a row of W counters:

         W counters
    [0] [0] [0] [0] [0]   ← row 1 (hash function 1)
    [0] [0] [0] [0] [0]   ← row 2 (hash function 2)
    [0] [0] [0] [0] [0]   ← row 3 (hash function 3)

INSERT "apple":
  hash1("apple") % 5 = 2 → increment row 1, position 2
  hash2("apple") % 5 = 0 → increment row 2, position 0
  hash3("apple") % 5 = 4 → increment row 3, position 4

    [0] [0] [1] [0] [0]
    [1] [0] [0] [0] [0]
    [0] [0] [0] [0] [1]

QUERY "apple":
  Check row 1[2]=1, row 2[0]=1, row 3[4]=1
  Estimate = MIN(1, 1, 1) = 1 ✅

The estimate is always ≥ the true count (never underestimates).
Taking the MIN across rows minimizes overestimation from hash collisions.
```

**Properties:**
- **Never underestimates** — the estimate is always ≥ the true count
- **May overestimate** — due to hash collisions (other elements increment the same counters)
- **Fixed memory** — D × W counters, regardless of the number of distinct elements
- **Mergeable** — two sketches can be merged (element-wise addition)

**Use Cases:**
- Heavy hitters detection (top-K most frequent elements)
- Network traffic monitoring (most frequent source IPs)
- Query frequency estimation (most popular search queries)
- Anomaly detection (sudden frequency spikes)

---

### Skip List

A **skip list** is a probabilistic data structure that provides O(log n) search, insert, and delete — like a balanced binary search tree but simpler to implement and naturally concurrent.

```
Skip List Structure (4 levels):

Level 3:  HEAD ──────────────────────────────── 50 ──────────────────── NIL
Level 2:  HEAD ──────────── 20 ──────────────── 50 ──────── 70 ──────── NIL
Level 1:  HEAD ──── 10 ──── 20 ──── 30 ──────── 50 ──── 60 ── 70 ── 80 ── NIL
Level 0:  HEAD ── 5 ── 10 ── 15 ── 20 ── 25 ── 30 ── 40 ── 50 ── 55 ── 60 ── 70 ── 75 ── 80 ── NIL

Search for 55:
  Level 3: HEAD → 50 (50 < 55, go right) → NIL (too far, go down)
  Level 2: 50 → 70 (70 > 55, go down)
  Level 1: 50 → 60 (60 > 55, go down)
  Level 0: 50 → 55 → FOUND!

  Only visited 4 nodes instead of scanning all 13!
```

**How It Works:**
- Level 0 is a sorted linked list containing all elements
- Higher levels are "express lanes" that skip over elements
- Each element is promoted to the next level with probability p (typically 0.5 or 0.25)
- Search starts at the highest level and drops down when the next element is too large

**Skip List Properties:**

| Property | Value |
|----------|-------|
| Search | O(log n) expected |
| Insert | O(log n) expected |
| Delete | O(log n) expected |
| Space | O(n) expected (with p=0.5, ~2n pointers total) |
| Concurrency | Naturally supports concurrent access (lock individual nodes) |

**Skip List vs Balanced BST (Red-Black Tree, AVL):**

| Feature | Skip List | Balanced BST |
|---------|-----------|-------------|
| Implementation | Simpler (linked list + random levels) | Complex (rotations, rebalancing) |
| Concurrency | Easy (lock-free or fine-grained locking) | Hard (rebalancing affects many nodes) |
| Cache performance | Worse (pointer chasing) | Better (more compact) |
| Range queries | Natural (follow level 0 links) | Requires in-order traversal |
| Deterministic | No (probabilistic) | Yes |

**Used by:** Redis (sorted sets), LevelDB/RocksDB (memtable), Cassandra (memtable), ConcurrentSkipListMap (Java)

---

### Probabilistic Data Structure Summary

| Data Structure | Question It Answers | Space | Error Type | Used By |
|---------------|-------------------|-------|-----------|---------|
| **Bloom Filter** | "Is X in the set?" | O(m) bits | False positives (no false negatives) | Cassandra, Chrome, CDNs |
| **Counting Bloom** | "Is X in the set?" + delete | O(m × c) bits | False positives | Network monitoring |
| **HyperLogLog** | "How many distinct elements?" | ~12 KB | ±0.81% error | Redis, BigQuery, Presto |
| **Count-Min Sketch** | "How many times has X appeared?" | O(d × w) counters | Overestimation | Network monitoring, top-K |
| **Skip List** | Sorted set operations | O(n) | None (exact) | Redis, LevelDB, RocksDB |

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Consistent Hashing | Hash ring; adding a node moves only ~1/N keys; virtual nodes for even distribution |
| Virtual Nodes | Each physical node maps to 100-256 positions on the ring; ensures even distribution |
| Gossip Protocol | Peer-to-peer state exchange; O(log N) convergence; no single point of failure |
| Phi Accrual Detector | Statistical failure detection; adapts to network conditions; used by Cassandra |
| Quorum | W + R > N for strong consistency; tune W and R for read-heavy or write-heavy workloads |
| Sloppy Quorum | Write to any available node (not just designated replicas); improves availability |
| Hinted Handoff | Temporarily store data on non-designated node; forward when correct node recovers |
| Last-Write-Wins | Simplest conflict resolution; silent data loss; depends on synchronized clocks |
| Vector Clocks | Detect concurrent writes; return all versions for application-level resolution |
| CRDTs | Conflict-free data structures; automatic merge; G-Counter, PN-Counter, OR-Set |
| G-Counter | Grow-only counter; per-node counters; merge = max per entry, total = sum |
| OR-Set | Add + remove set; tagged additions; "add wins" over concurrent remove |
| MapReduce | Map (transform) → Shuffle (group by key) → Reduce (aggregate); batch processing |
| Hadoop | HDFS + YARN + MapReduce; disk-based; slow but handles petabytes |
| Spark | In-memory processing; 10-100x faster than MapReduce; batch + streaming |
| Bloom Filter | "Definitely not" or "maybe yes"; no false negatives; ~1.2 MB for 1M elements at 1% FP |
| HyperLogLog | Count distinct elements; ~12 KB for billions of elements; ±0.81% error |
| Count-Min Sketch | Estimate element frequency; fixed memory; never underestimates |
| Skip List | Probabilistic sorted data structure; O(log n); easy concurrency; used by Redis sorted sets |

---

## Interview Tips for Module 27

1. **Consistent hashing** — draw the hash ring; explain virtual nodes; show that adding a node moves only ~1/N keys; mention Cassandra and Redis Cluster
2. **Gossip protocol** — explain push-pull gossip; mention O(log N) convergence; explain phi accrual failure detector
3. **Quorum formula** — W + R > N for strong consistency; give examples with N=3; explain the trade-off between W and R
4. **Sloppy quorum + hinted handoff** — explain how they improve availability; mention the consistency trade-off
5. **CRDTs** — explain G-Counter and OR-Set with examples; explain why they converge automatically; mention Redis active-active
6. **LWW vs Vector Clocks vs CRDTs** — know the trade-offs; LWW is simple but loses data; vector clocks detect conflicts; CRDTs resolve automatically
7. **MapReduce** — explain the three phases with a word count example; know that Spark has largely replaced it
8. **Spark vs MapReduce** — Spark is in-memory (10-100x faster); supports streaming; better API
9. **Bloom filter** — explain how it works (bit array + k hash functions); "definitely not" or "maybe yes"; mention SSTable lookups
10. **HyperLogLog** — explain the leading zeros intuition; ~12 KB for billions of elements; mention Redis PFCOUNT
11. **Count-Min Sketch** — explain the grid of counters; MIN across rows; used for heavy hitters / top-K
12. **Skip list** — explain the multi-level linked list; O(log n); used by Redis sorted sets; easier concurrency than balanced trees
