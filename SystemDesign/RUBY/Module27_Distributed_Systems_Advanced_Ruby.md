# Module 27: Distributed Systems — Advanced

> This module covers the advanced building blocks of distributed systems — the algorithms and data structures that power databases like Cassandra and DynamoDB, caching systems like Redis Cluster, and big data platforms like Hadoop and Spark. Consistent hashing, gossip protocols, quorums, CRDTs, MapReduce, and probabilistic data structures are the tools that make planet-scale systems possible.

> **Ruby Context:** Ruby developers interact with these advanced distributed concepts through the systems they use daily. **Redis Cluster** uses consistent hashing and HyperLogLog (`redis-rb` gem). **Cassandra** uses gossip protocols, consistent hashing, and quorums (`cassandra-driver` gem). **Sidekiq** distributes work across workers. **Elasticsearch** uses sharding based on consistent hashing (`elasticsearch-ruby` gem). Understanding these internals helps you configure, debug, and scale the infrastructure your Ruby applications depend on. For data processing, Ruby integrates with Spark via **spark-ruby** or delegates to Python/Scala jobs, while gems like **bloomfilter-rb** and **redis-bloomfilter** bring probabilistic data structures into Ruby applications.

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

> **Ruby Context:** This problem is exactly what happens if you use a naive `Digest::MD5.hexdigest(key).to_i(16) % server_count` approach for distributing cache keys across Memcached or Redis instances. The `dalli` gem (Memcached client) uses consistent hashing by default to avoid this problem. The `redis-rb` gem with `redis-cluster` also handles this transparently.

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

> **Ruby Context:** Here's a Ruby implementation of a consistent hash ring that you can use for distributing work across servers, cache nodes, or any set of resources.

```ruby
# Consistent hash ring implementation in Ruby
require 'digest'
require 'sorted_set'

class ConsistentHashRing
  def initialize(nodes: [], replicas: 150)
    @replicas = replicas       # Virtual nodes per physical node
    @ring = {}                 # hash_position → node
    @sorted_keys = []          # Sorted positions for binary search
    @nodes = []

    nodes.each { |node| add_node(node) }
  end

  def add_node(node)
    @nodes << node
    @replicas.times do |i|
      hash_key = hash_for("#{node}:#{i}")
      @ring[hash_key] = node
    end
    @sorted_keys = @ring.keys.sort
  end

  def remove_node(node)
    @nodes.delete(node)
    @replicas.times do |i|
      hash_key = hash_for("#{node}:#{i}")
      @ring.delete(hash_key)
    end
    @sorted_keys = @ring.keys.sort
  end

  # Find the node responsible for a given key
  def get_node(key)
    return nil if @ring.empty?

    hash_key = hash_for(key)

    # Binary search for the first position >= hash_key
    idx = @sorted_keys.bsearch_index { |pos| pos >= hash_key }
    idx ||= 0  # Wrap around to the first node

    @ring[@sorted_keys[idx]]
  end

  # Get N nodes for replication (walk clockwise, skip duplicates)
  def get_nodes(key, count: 3)
    return @nodes.dup if @nodes.size <= count

    hash_key = hash_for(key)
    idx = @sorted_keys.bsearch_index { |pos| pos >= hash_key } || 0

    result = []
    seen = Set.new
    @sorted_keys.size.times do |i|
      pos = @sorted_keys[(idx + i) % @sorted_keys.size]
      node = @ring[pos]
      unless seen.include?(node)
        result << node
        seen << node
        break if result.size == count
      end
    end
    result
  end

  private

  def hash_for(key)
    Digest::MD5.hexdigest(key.to_s).to_i(16)
  end
end

# Usage
ring = ConsistentHashRing.new(nodes: ['redis-1', 'redis-2', 'redis-3'], replicas: 150)

ring.get_node('user:123')        # => "redis-2"
ring.get_node('user:456')        # => "redis-1"
ring.get_nodes('user:123', count: 2)  # => ["redis-2", "redis-3"]

# Add a node — only ~1/N keys move
ring.add_node('redis-4')
ring.get_node('user:123')        # Most keys stay on the same node
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

> **Ruby Context:** The `ConsistentHashRing` implementation above uses the `replicas` parameter for virtual nodes (default 150). The `dalli` gem for Memcached uses a similar approach internally. Redis Cluster uses a fixed 16,384 hash slots (a variation of virtual nodes) distributed across masters.

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

> **Ruby Context:** When using `redis-rb` with Redis Cluster, the gem handles hash slot routing automatically. For Memcached via `dalli`, consistent hashing is the default distribution strategy. For Cassandra via `cassandra-driver`, the driver is aware of the token ring and routes queries to the correct coordinator node.

```ruby
# Redis Cluster — consistent hashing handled by the driver
require 'redis-clustering'

redis = RedisClient.cluster(nodes: [
  'redis://redis-1:6379',
  'redis://redis-2:6379',
  'redis://redis-3:6379'
]).new_client

# The driver automatically routes to the correct node based on hash slot
redis.call('SET', 'user:123', 'Alice')  # Routed to the node owning slot CRC16("user:123") % 16384
redis.call('GET', 'user:123')           # Routed to the same node

# Dalli (Memcached) — consistent hashing by default
require 'dalli'

cache = Dalli::Client.new(['memcached-1:11211', 'memcached-2:11211', 'memcached-3:11211'])
cache.set('user:123', user_data)  # Consistent hashing determines which server
cache.get('user:123')             # Same server, even if cluster changes
```

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

> **Ruby Context:** While you typically don't implement gossip protocols in Ruby application code, understanding them helps you reason about how Cassandra, Consul, and Redis Cluster propagate state. For example, when you add a new Cassandra node, gossip is how the cluster discovers it — and it takes O(log N) rounds (seconds) for all nodes to learn about the new member. The `diplomat` gem (Consul) exposes membership information that Consul maintains via its Serf gossip layer.

```ruby
# Gossip protocol simulation in Ruby (educational)
class GossipNode
  attr_reader :id, :state

  def initialize(id)
    @id = id
    @state = { id => { heartbeat: 0, status: :alive, updated_at: Time.now } }
    @mutex = Mutex.new
  end

  # Increment local heartbeat
  def heartbeat
    @mutex.synchronize do
      @state[@id][:heartbeat] += 1
      @state[@id][:updated_at] = Time.now
    end
  end

  # Push-pull gossip exchange with another node
  def gossip_with(other_node)
    # Push: send our state
    remote_state = other_node.receive_gossip(@state.dup)

    # Pull: merge their state into ours
    merge_state(remote_state)
  end

  # Receive gossip from another node, return our state
  def receive_gossip(incoming_state)
    @mutex.synchronize do
      merge_state_internal(incoming_state)
      @state.dup
    end
  end

  # Detect failed nodes (no heartbeat update for threshold)
  def detect_failures(threshold: 10)
    @mutex.synchronize do
      @state.each do |node_id, info|
        next if node_id == @id
        if Time.now - info[:updated_at] > threshold
          info[:status] = :suspected
        end
      end
    end
  end

  private

  def merge_state(remote_state)
    @mutex.synchronize { merge_state_internal(remote_state) }
  end

  def merge_state_internal(remote_state)
    remote_state.each do |node_id, remote_info|
      local_info = @state[node_id]
      if local_info.nil? || remote_info[:heartbeat] > local_info[:heartbeat]
        @state[node_id] = remote_info.dup
      end
    end
  end
end
```

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

> **Ruby Context:** When using the `cassandra-driver` gem, you can configure the reconnection policy and understand that Cassandra's gossip-based failure detection (phi accrual) determines which nodes the driver considers available. If a node is marked down by gossip, the driver routes queries to other replicas.

```ruby
# Cassandra driver — aware of gossip-based failure detection
require 'cassandra'

cluster = Cassandra.cluster(
  hosts: ['cassandra-1', 'cassandra-2', 'cassandra-3'],
  reconnection_policy: Cassandra::Reconnection::Policies::Exponential.new(0.5, 30, 2),
  # The driver is aware of the cluster topology via gossip
  # It automatically routes to healthy nodes
  load_balancing_policy: Cassandra::LoadBalancing::Policies::TokenAware.new(
    Cassandra::LoadBalancing::Policies::DCAwareRoundRobin.new('us-east-1')
  )
)

session = cluster.connect('my_keyspace')
# Queries are routed to the correct node based on partition key (consistent hashing)
# and away from nodes that gossip has marked as down
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

> **Ruby Context:** Consul uses the SWIM protocol (via Serf) for cluster membership. When you query Consul for healthy service instances using the `diplomat` gem, the results reflect the current gossip-based membership state.

```ruby
# Consul membership via diplomat — reflects gossip state
require 'diplomat'

# Get all members of the Consul cluster (maintained by SWIM gossip)
members = Diplomat::Members.get
members.each do |member|
  puts "#{member['Name']}: #{member['Status']} (#{member['Addr']})"
  # Status: 1 = alive, 2 = leaving, 3 = left, 4 = failed
end

# Get only healthy instances of a service (gossip-aware)
healthy = Diplomat::Health.service('order-service', passing: true)
puts "#{healthy.size} healthy instances of order-service"
```

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

> **Ruby Context:** When using the `cassandra-driver` gem, you configure consistency levels that map directly to quorum settings. Understanding W, R, and N helps you choose the right consistency level for each query.

```ruby
# Cassandra quorum configuration via cassandra-driver gem
require 'cassandra'

cluster = Cassandra.cluster(hosts: ['cassandra-1', 'cassandra-2', 'cassandra-3'])
session = cluster.connect('my_keyspace')

# Write with QUORUM (W = ⌊N/2⌋ + 1 = 2 for RF=3)
session.execute(
  "INSERT INTO users (id, name, email) VALUES (?, ?, ?)",
  arguments: [user_id, 'Alice', 'alice@example.com'],
  consistency: :quorum  # Must write to 2 of 3 replicas
)

# Read with QUORUM (R = 2 for RF=3) — W + R > N → strong consistency
result = session.execute(
  "SELECT * FROM users WHERE id = ?",
  arguments: [user_id],
  consistency: :quorum  # Must read from 2 of 3 replicas
)

# Read with ONE (R = 1) — faster but eventually consistent
result = session.execute(
  "SELECT * FROM users WHERE id = ?",
  arguments: [user_id],
  consistency: :one  # Read from any 1 replica (may be stale)
)

# Write with ALL (W = N = 3) — slowest but all replicas have the data
session.execute(
  "INSERT INTO users (id, name) VALUES (?, ?)",
  arguments: [user_id, 'Alice'],
  consistency: :all  # Must write to ALL 3 replicas
)
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

> **Ruby Context:** Cassandra consistency levels map to these quorum values:

| Cassandra Consistency | W or R Value | Ruby Symbol |
|----------------------|-------------|-------------|
| `ONE` | 1 | `:one` |
| `TWO` | 2 | `:two` |
| `THREE` | 3 | `:three` |
| `QUORUM` | ⌊N/2⌋ + 1 | `:quorum` |
| `LOCAL_QUORUM` | ⌊N_local/2⌋ + 1 | `:local_quorum` |
| `ALL` | N | `:all` |
| `ANY` | 1 (including hinted handoff) | `:any` |

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

> **Ruby Context:** When using Cassandra with the `:any` consistency level, writes can succeed via hinted handoff even if none of the designated replicas are available. This is the most available but least consistent option. In practice, `:quorum` or `:local_quorum` are the most common choices for Ruby applications.

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

> **Ruby Context:** Cassandra uses LWW by default — each cell has a timestamp, and the highest timestamp wins during reads. When using the `cassandra-driver` gem, be aware that concurrent writes to the same column will silently discard all but the latest. For counters, Cassandra provides a dedicated counter column type that avoids LWW semantics.

```ruby
# LWW in Cassandra — concurrent writes, latest timestamp wins
session.execute(
  "INSERT INTO users (id, name) VALUES (?, ?) USING TIMESTAMP ?",
  arguments: [user_id, 'Alice', 1000]
)

# Concurrent write from another service with a later timestamp
session.execute(
  "INSERT INTO users (id, name) VALUES (?, ?) USING TIMESTAMP ?",
  arguments: [user_id, 'Bob', 1001]
)

# Read returns "Bob" — "Alice" is silently lost
result = session.execute("SELECT name FROM users WHERE id = ?", arguments: [user_id])
result.first['name']  # => "Bob"

# For counters, use Cassandra's counter type (not LWW)
session.execute(
  "UPDATE page_views SET count = count + 1 WHERE page_id = ?",
  arguments: [page_id]
)
```

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

> **Ruby Context:** See Module 26.6 for a Ruby `VectorClock` implementation. In practice, most Ruby applications use databases (PostgreSQL, Cassandra) that handle conflict resolution internally. However, understanding vector clocks helps when designing multi-region systems or working with CouchDB/Riak where conflicts are exposed to the application.

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

> **Ruby Context:** Here's a Ruby implementation of common CRDTs that you can use for distributed state management across Ruby services.

```ruby
# G-Counter CRDT in Ruby
class GCounter
  attr_reader :counts

  def initialize(node_id)
    @node_id = node_id
    @counts = Hash.new(0)
  end

  def increment(amount = 1)
    @counts[@node_id] += amount
  end

  def value
    @counts.values.sum
  end

  def merge(other)
    all_keys = (@counts.keys + other.counts.keys).uniq
    all_keys.each do |key|
      @counts[key] = [@counts[key], other.counts[key] || 0].max
    end
    self
  end

  def to_h
    @counts.dup
  end
end

# Usage across distributed services
counter_a = GCounter.new(:service_a)
counter_b = GCounter.new(:service_b)

counter_a.increment(3)  # Service A counted 3 page views
counter_b.increment(5)  # Service B counted 5 page views

# Merge (order doesn't matter — commutative and associative)
counter_a.merge(counter_b)
counter_a.value  # => 8
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

```ruby
# PN-Counter CRDT in Ruby
class PNCounter
  def initialize(node_id)
    @p = GCounter.new(node_id)  # Positive (increments)
    @n = GCounter.new(node_id)  # Negative (decrements)
  end

  def increment(amount = 1)
    @p.increment(amount)
  end

  def decrement(amount = 1)
    @n.increment(amount)
  end

  def value
    @p.value - @n.value
  end

  def merge(other)
    @p.merge(other.instance_variable_get(:@p))
    @n.merge(other.instance_variable_get(:@n))
    self
  end
end

# Usage: distributed upvote/downvote counter
votes_a = PNCounter.new(:server_a)
votes_b = PNCounter.new(:server_b)

votes_a.increment(10)  # 10 upvotes on server A
votes_a.decrement(3)   # 3 downvotes on server A
votes_b.increment(7)   # 7 upvotes on server B
votes_b.decrement(2)   # 2 downvotes on server B

votes_a.merge(votes_b)
votes_a.value  # => 12 (17 upvotes - 5 downvotes)
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

```ruby
# OR-Set CRDT in Ruby
class ORSet
  def initialize(node_id)
    @node_id = node_id
    @elements = {}   # element → Set of tags
    @tombstones = {} # element → Set of removed tags
    @counter = 0
  end

  def add(element)
    @counter += 1
    tag = "#{@node_id}:#{@counter}"
    @elements[element] ||= Set.new
    @elements[element] << tag
  end

  def remove(element)
    return unless @elements[element]
    @tombstones[element] ||= Set.new
    @tombstones[element].merge(@elements[element])  # Remove all observed tags
    @elements[element] = Set.new
  end

  def include?(element)
    return false unless @elements[element]
    # Element exists if it has tags not in tombstones
    live_tags = @elements[element] - (@tombstones[element] || Set.new)
    live_tags.any?
  end

  def to_a
    @elements.select { |elem, _| include?(elem) }.keys
  end

  def merge(other)
    # Merge elements (union of tags)
    other.instance_variable_get(:@elements).each do |elem, tags|
      @elements[elem] ||= Set.new
      @elements[elem].merge(tags)
    end
    # Merge tombstones (union of removed tags)
    other.instance_variable_get(:@tombstones).each do |elem, tags|
      @tombstones[elem] ||= Set.new
      @tombstones[elem].merge(tags)
    end
    self
  end
end

# Usage: distributed shopping cart
cart_a = ORSet.new(:server_a)
cart_b = ORSet.new(:server_b)

cart_a.add('apple')
cart_a.add('banana')
cart_b.add('cherry')
cart_b.remove('apple')  # Won't affect A's add (hasn't observed it)

cart_a.merge(cart_b)
cart_a.to_a  # => ["apple", "banana", "cherry"]
# apple survives because A's add was concurrent with B's remove
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

> **Ruby Context:** Redis Enterprise's active-active geo-replication uses CRDTs internally. When using `redis-rb` with Redis Enterprise's CRDB (Conflict-free Replicated Database), your Ruby application benefits from CRDT-based conflict resolution transparently. For application-level CRDTs, the implementations above can be serialized to JSON and stored in Redis or a database for cross-service merging.

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

> **Ruby Context:** Ruby's `Enumerable` methods (`map`, `group_by`, `reduce`) mirror the MapReduce paradigm for in-process data. For distributed MapReduce, Ruby applications typically delegate to Hadoop/Spark jobs or use Ruby-based alternatives like **Sidekiq** for parallel processing across workers. The `parallel` gem enables multi-process map operations on a single machine.

```ruby
# MapReduce concept in Ruby (single-process, educational)
log_lines = [
  "GET /home 200",
  "GET /about 200",
  "GET /home 200",
  "POST /login 401",
  "GET /home 200",
  "GET /about 404"
]

# MAP: extract key-value pairs
mapped = log_lines.map do |line|
  method, path, status = line.split
  [path, 1]
end
# => [["/home", 1], ["/about", 1], ["/home", 1], ["/login", 1], ["/home", 1], ["/about", 1]]

# SHUFFLE: group by key
shuffled = mapped.group_by(&:first).transform_values { |pairs| pairs.map(&:last) }
# => {"/home" => [1, 1, 1], "/about" => [1, 1], "/login" => [1]}

# REDUCE: aggregate values per key
result = shuffled.transform_values(&:sum)
# => {"/home" => 3, "/about" => 2, "/login" => 1}

# Distributed MapReduce with Sidekiq (fan-out pattern)
# Split work across Sidekiq workers, aggregate results
class MapWorker
  include Sidekiq::Job

  def perform(batch_id, chunk)
    # MAP phase: process a chunk of data
    counts = chunk.each_with_object(Hash.new(0)) do |line, acc|
      path = line.split[1]
      acc[path] += 1
    end

    # Store partial results in Redis
    counts.each do |path, count|
      Redis.current.hincrby("mapreduce:#{batch_id}:results", path, count)
    end
  end
end

# Fan out: split data into chunks and distribute
chunks = log_lines.each_slice(1000).to_a
batch_id = SecureRandom.uuid
chunks.each { |chunk| MapWorker.perform_async(batch_id, chunk) }
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

> **Ruby Context:** Ruby is not a first-class Spark language, but Ruby applications commonly interact with Spark in several ways:
> - **Trigger Spark jobs** from Ruby via shell commands or REST API (Livy)
> - **Read Spark output** from S3/HDFS/databases after batch processing
> - **Feed data to Spark** by writing to Kafka (via Karafka) or S3
> - Use **PySpark** or **Scala Spark** for the processing, with Ruby handling the web layer
>
> For simpler parallel processing within Ruby, the `parallel` gem, Sidekiq fan-out, or Ractor (Ruby 3+) can handle moderate-scale data processing.

```ruby
# Triggering a Spark job from Ruby (via Livy REST API)
class SparkJobLauncher
  def initialize
    @livy_url = ENV['LIVY_URL']  # e.g., "http://spark-master:8998"
    @conn = Faraday.new(url: @livy_url) do |f|
      f.request :json
      f.response :json
    end
  end

  def submit_batch(jar_path:, class_name:, args: [])
    response = @conn.post('/batches') do |req|
      req.body = {
        file: jar_path,
        className: class_name,
        args: args,
        driverMemory: '2g',
        executorMemory: '4g',
        numExecutors: 10
      }
    end
    response.body  # Returns batch ID for tracking
  end

  def check_status(batch_id)
    response = @conn.get("/batches/#{batch_id}")
    response.body['state']  # "running", "success", "dead"
  end
end

# Parallel processing within Ruby (parallel gem)
require 'parallel'

# Process 1 million records across all CPU cores
results = Parallel.map(records.each_slice(10_000), in_processes: 8) do |chunk|
  chunk.each_with_object(Hash.new(0)) do |record, counts|
    counts[record[:category]] += 1
  end
end

# Merge partial results
final = results.each_with_object(Hash.new(0)) do |partial, total|
  partial.each { |key, count| total[key] += count }
end
```

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

> **Ruby Context:** The `bloomfilter-rb` gem provides a native Ruby Bloom filter implementation. Redis also supports Bloom filters via the RedisBloom module (`BF.ADD`, `BF.EXISTS`), accessible from Ruby via `redis-rb`. For high-performance needs, the `bloomer` gem uses a C extension.

```ruby
# Bloom filter with bloomfilter-rb gem
require 'bloomfilter-rb'

# Create a Bloom filter for 1 million elements with 1% false positive rate
bf = BloomFilter::Native.new(size: 1_000_000, hashes: 7, seed: 42)

# Insert elements
bf.insert('user:123')
bf.insert('user:456')
bf.insert('user:789')

# Query
bf.include?('user:123')  # => true (definitely or probably in the set)
bf.include?('user:999')  # => false (DEFINITELY not in the set)
bf.include?('user:555')  # => false or true (might be a false positive)

# Redis Bloom filter (RedisBloom module)
require 'redis'

redis = Redis.new

# Add elements
redis.call('BF.ADD', 'visited_urls', 'https://example.com/page1')
redis.call('BF.ADD', 'visited_urls', 'https://example.com/page2')

# Check membership
redis.call('BF.EXISTS', 'visited_urls', 'https://example.com/page1')  # => 1 (probably yes)
redis.call('BF.EXISTS', 'visited_urls', 'https://example.com/page99') # => 0 (definitely no)

# Practical use: avoid duplicate processing in Sidekiq
class DeduplicatedWorker
  include Sidekiq::Job

  BLOOM_KEY = 'processed_events'

  def perform(event_id, payload)
    # Quick check: definitely not processed? (Bloom filter)
    already_seen = Redis.current.call('BF.EXISTS', BLOOM_KEY, event_id) == 1

    if already_seen
      # Might be a false positive — do an exact check
      return if ProcessedEvent.exists?(event_id: event_id)
    end

    # Process the event
    process_event(payload)

    # Mark as processed (both exact and Bloom filter)
    ProcessedEvent.create!(event_id: event_id)
    Redis.current.call('BF.ADD', BLOOM_KEY, event_id)
  end
end
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

> **Ruby Context:** Redis has built-in HyperLogLog support via `PFADD`, `PFCOUNT`, and `PFMERGE`. This is the most common way Ruby applications use HyperLogLog — through Redis. The `redis-rb` gem exposes these commands directly.

```ruby
# HyperLogLog with Redis (redis-rb gem)
require 'redis'

redis = Redis.new

# Track unique visitors per page per day
date = Date.today.to_s

# Add visitor IDs (PFADD)
redis.pfadd("unique_visitors:#{date}:/home", 'user_123')
redis.pfadd("unique_visitors:#{date}:/home", 'user_456')
redis.pfadd("unique_visitors:#{date}:/home", 'user_123')  # Duplicate — not counted again
redis.pfadd("unique_visitors:#{date}:/home", 'user_789')

# Count unique visitors (PFCOUNT) — ~12 KB memory regardless of count!
count = redis.pfcount("unique_visitors:#{date}:/home")
# => 3 (with ~0.81% standard error for large counts)

# Merge multiple HyperLogLogs (PFMERGE) — union of sets
redis.pfadd("unique_visitors:#{date}:/about", 'user_123')
redis.pfadd("unique_visitors:#{date}:/about", 'user_999')

# Total unique visitors across all pages (some visitors may overlap)
redis.pfmerge("unique_visitors:#{date}:total",
              "unique_visitors:#{date}:/home",
              "unique_visitors:#{date}:/about")
total = redis.pfcount("unique_visitors:#{date}:total")
# => 4 (user_123 counted only once across both pages)

# Practical use in a Rails controller
class AnalyticsTracker
  def track_page_view(page_path, user_id)
    date = Date.today.to_s
    key = "unique_visitors:#{date}:#{page_path}"

    # HyperLogLog — O(1) add, ~12 KB per counter
    Redis.current.pfadd(key, user_id)

    # Set TTL so old counters are cleaned up
    Redis.current.expire(key, 7.days.to_i)
  end

  def unique_visitors(page_path, date: Date.today)
    Redis.current.pfcount("unique_visitors:#{date}:#{page_path}")
  end
end
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

> **Ruby Context:** Redis supports Count-Min Sketch via the RedisBloom module (`CMS.INCRBY`, `CMS.QUERY`). For pure Ruby, you can implement a Count-Min Sketch or use the `countmin` gem.

```ruby
# Count-Min Sketch implementation in Ruby
require 'digest'

class CountMinSketch
  def initialize(width: 1000, depth: 5)
    @width = width
    @depth = depth
    @table = Array.new(depth) { Array.new(width, 0) }
    @seeds = (0...depth).map { |i| "cms_seed_#{i}" }
  end

  def add(element, count = 1)
    @depth.times do |i|
      position = hash_for(element, i)
      @table[i][position] += count
    end
  end

  def estimate(element)
    @depth.times.map do |i|
      position = hash_for(element, i)
      @table[i][position]
    end.min  # Take the minimum across all rows
  end

  def merge!(other)
    @depth.times do |i|
      @width.times do |j|
        @table[i][j] += other.instance_variable_get(:@table)[i][j]
      end
    end
    self
  end

  private

  def hash_for(element, row)
    Digest::MD5.hexdigest("#{@seeds[row]}:#{element}").to_i(16) % @width
  end
end

# Usage: track search query frequency
sketch = CountMinSketch.new(width: 10_000, depth: 7)

# Record search queries
queries = ['ruby gems', 'rails tutorial', 'ruby gems', 'sidekiq', 'ruby gems']
queries.each { |q| sketch.add(q) }

sketch.estimate('ruby gems')     # => 3 (exact or slight overestimate)
sketch.estimate('rails tutorial') # => 1
sketch.estimate('unknown query')  # => 0 (or small overestimate)

# Redis Count-Min Sketch (RedisBloom module)
redis = Redis.new
redis.call('CMS.INITBYPROB', 'search_queries', 0.001, 0.01)  # error, probability
redis.call('CMS.INCRBY', 'search_queries', 'ruby gems', 1)
redis.call('CMS.QUERY', 'search_queries', 'ruby gems')  # => [1]
```

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

> **Ruby Context:** Redis uses skip lists internally for its Sorted Set data type (`ZADD`, `ZRANGE`, `ZRANGEBYSCORE`). When you use Redis sorted sets from Ruby via `redis-rb`, you're leveraging skip lists under the hood. Understanding skip lists helps you reason about the O(log n) performance of sorted set operations.

```ruby
# Skip list implementation in Ruby (educational)
class SkipListNode
  attr_accessor :key, :value, :forward

  def initialize(key, value, level)
    @key = key
    @value = value
    @forward = Array.new(level + 1)  # Pointers to next nodes at each level
  end
end

class SkipList
  MAX_LEVEL = 16
  P = 0.5  # Probability of promotion

  def initialize
    @header = SkipListNode.new(nil, nil, MAX_LEVEL)
    @level = 0
  end

  def insert(key, value)
    update = Array.new(MAX_LEVEL + 1)
    current = @header

    # Find position at each level
    @level.downto(0) do |i|
      while current.forward[i] && current.forward[i].key < key
        current = current.forward[i]
      end
      update[i] = current
    end

    current = current.forward[0]

    if current && current.key == key
      current.value = value  # Update existing
    else
      new_level = random_level
      @level = new_level if new_level > @level

      new_node = SkipListNode.new(key, value, new_level)
      (0..new_level).each do |i|
        new_node.forward[i] = update[i]&.forward&.[](i)
        update[i]&.forward&.[]=(i, new_node) if update[i]
      end
    end
  end

  def search(key)
    current = @header
    @level.downto(0) do |i|
      while current.forward[i] && current.forward[i].key < key
        current = current.forward[i]
      end
    end
    current = current.forward[0]
    current&.key == key ? current.value : nil
  end

  private

  def random_level
    level = 0
    level += 1 while rand < P && level < MAX_LEVEL
    level
  end
end

# Usage
list = SkipList.new
list.insert(10, 'ten')
list.insert(5, 'five')
list.insert(20, 'twenty')
list.insert(15, 'fifteen')

list.search(15)  # => "fifteen"
list.search(99)  # => nil

# In practice, use Redis sorted sets (backed by skip lists)
redis = Redis.new
redis.zadd('leaderboard', 100, 'alice')
redis.zadd('leaderboard', 250, 'bob')
redis.zadd('leaderboard', 175, 'charlie')

# O(log n) operations powered by skip lists
redis.zrangebyscore('leaderboard', 100, 200)  # => ["alice", "charlie"]
redis.zrevrange('leaderboard', 0, 2, with_scores: true)
# => [["bob", 250.0], ["charlie", 175.0], ["alice", 100.0]]
```

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

**Ruby-Specific Takeaways:**

| Area | Ruby Tools & Gems |
|------|-------------------|
| **Consistent Hashing** | `dalli` (Memcached), `redis-clustering` (Redis Cluster), custom `ConsistentHashRing` |
| **Gossip/Membership** | `diplomat` (Consul/Serf), `cassandra-driver` (Cassandra topology) |
| **Quorum Configuration** | `cassandra-driver` consistency levels (`:quorum`, `:one`, `:all`) |
| **CRDTs** | Custom Ruby implementations, Redis Enterprise CRDB |
| **Bloom Filters** | `bloomfilter-rb`, `bloomer`, Redis `BF.ADD`/`BF.EXISTS` (RedisBloom) |
| **HyperLogLog** | Redis `PFADD`/`PFCOUNT`/`PFMERGE` via `redis-rb` |
| **Count-Min Sketch** | Custom Ruby implementation, Redis `CMS.INCRBY`/`CMS.QUERY` (RedisBloom) |
| **Skip Lists** | Redis sorted sets (`ZADD`, `ZRANGE`) via `redis-rb` |
| **Parallel Processing** | `parallel` gem, Sidekiq fan-out, Ractor (Ruby 3+) |
| **Spark Integration** | Livy REST API, shell commands, read Spark output from S3/DB |

---

## Interview Tips for Module 27

1. **Consistent hashing** — draw the hash ring; explain virtual nodes; show that adding a node moves only ~1/N keys; mention Cassandra, Redis Cluster, and the `dalli` gem
2. **Gossip protocol** — explain push-pull gossip; mention O(log N) convergence; explain phi accrual failure detector; reference Consul's Serf layer
3. **Quorum formula** — W + R > N for strong consistency; give examples with N=3; explain the trade-off between W and R; reference Cassandra consistency levels in Ruby
4. **Sloppy quorum + hinted handoff** — explain how they improve availability; mention the consistency trade-off
5. **CRDTs** — explain G-Counter and OR-Set with examples; explain why they converge automatically; mention Redis Enterprise active-active; show Ruby implementations
6. **LWW vs Vector Clocks vs CRDTs** — know the trade-offs; LWW is simple but loses data; vector clocks detect conflicts; CRDTs resolve automatically
7. **MapReduce** — explain the three phases with a word count example; know that Spark has largely replaced it; show how Ruby's `map`/`group_by`/`reduce` mirrors the paradigm
8. **Spark vs MapReduce** — Spark is in-memory (10-100x faster); supports streaming; better API; Ruby integrates via Livy or reads Spark output
9. **Bloom filter** — explain how it works (bit array + k hash functions); "definitely not" or "maybe yes"; mention Redis `BF.ADD`/`BF.EXISTS` and `bloomfilter-rb` gem
10. **HyperLogLog** — explain the leading zeros intuition; ~12 KB for billions of elements; mention Redis `PFCOUNT`; show the Ruby/Redis usage pattern
11. **Count-Min Sketch** — explain the grid of counters; MIN across rows; used for heavy hitters / top-K
12. **Skip list** — explain the multi-level linked list; O(log n); used by Redis sorted sets; easier concurrency than balanced trees