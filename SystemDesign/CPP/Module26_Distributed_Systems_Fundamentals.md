# Module 26: Distributed Systems Fundamentals

> A distributed system is a collection of independent computers that appears to its users as a single coherent system. Every large-scale application — from Google Search to Netflix streaming to your bank's transaction system — is a distributed system. This module covers the foundational theory: why distributed systems are hard, the consistency models that govern them, the consensus algorithms that keep them coordinated, and the transaction patterns that keep them correct.

---

## 26.1 Core Concepts

> Distributed systems are fundamentally different from single-machine systems. They introduce problems that simply don't exist on a single computer: network failures, partial failures, clock skew, and the impossibility of global state. Understanding these challenges is the first step to designing systems that work correctly at scale.

---

### What Makes a System "Distributed"?

A system is distributed when its components run on **multiple networked computers** that coordinate their actions by **passing messages**. The components don't share memory or a clock.

```
Single Machine:                     Distributed System:
+------------------+                +--------+     +--------+
| CPU  | Memory    |                | Node A |←───→| Node B |
|      |           |                | (App)  | net | (App)  |
| Disk | Clock     |                +---+----+     +---+----+
+------------------+                    |              |
                                    +---+----+     +---+----+
Shared memory: yes                  | Node C |←───→| Node D |
Shared clock: yes                   | (DB)   | net | (Cache)|
Network: no                         +--------+     +--------+
Partial failure: no
                                    Shared memory: NO
                                    Shared clock: NO (clocks drift)
                                    Network: YES (unreliable)
                                    Partial failure: YES
```

**Key Properties of Distributed Systems:**
- **No shared memory** — nodes communicate only by sending messages over the network
- **No shared clock** — each node has its own clock, and clocks drift apart
- **Independent failure** — any node can fail independently while others continue
- **Network unreliability** — messages can be lost, delayed, duplicated, or reordered
- **Concurrency** — multiple nodes process requests simultaneously

---

### Fallacies of Distributed Computing (8 Fallacies)

In 1994, Peter Deutsch (and later James Gosling) identified **8 false assumptions** that developers make about distributed systems. Violating these assumptions causes bugs, outages, and architectural failures.

| # | Fallacy | Reality | Consequence of Assuming It |
|---|---------|---------|---------------------------|
| 1 | **The network is reliable** | Networks fail: packets are lost, cables are cut, switches crash | Must handle retries, timeouts, circuit breakers |
| 2 | **Latency is zero** | Network calls take 0.5-200ms (vs nanoseconds for local calls) | Must minimize network round trips; use caching, batching |
| 3 | **Bandwidth is infinite** | Bandwidth is limited and shared; large payloads cause congestion | Must compress data, paginate responses, use efficient serialization |
| 4 | **The network is secure** | Networks can be intercepted, spoofed, or attacked | Must encrypt (TLS), authenticate (mTLS), authorize |
| 5 | **Topology doesn't change** | Servers are added/removed, IPs change, routes change | Must use service discovery, not hardcoded addresses |
| 6 | **There is one administrator** | Multiple teams, cloud providers, ISPs manage different parts | Must handle heterogeneous environments, different SLAs |
| 7 | **Transport cost is zero** | Serialization, network transfer, and deserialization have real costs | Must consider data format (JSON vs protobuf), payload size |
| 8 | **The network is homogeneous** | Different hardware, OS, protocols, and versions coexist | Must use standard protocols, handle version differences |

**Practical Impact:**

```
Fallacy 1 (network is reliable):
  Developer writes: result = remote_service.call(request)
  Assumes: the call always succeeds
  Reality: the call can timeout, return an error, or never return at all
  Fix: result = retry_with_backoff(remote_service.call, request, max_retries=3, timeout=5s)

Fallacy 2 (latency is zero):
  Developer writes: for each user: fetch_profile(user.id)  // 100 users × 10ms = 1 second!
  Assumes: remote calls are as fast as local calls
  Reality: each network call adds 1-100ms of latency
  Fix: batch_fetch_profiles([user.id for user in users])  // 1 call × 15ms = 15ms
```

---

### Network Partitions

A **network partition** occurs when the network between two groups of nodes fails, splitting the system into isolated groups that can't communicate with each other.

```
Normal Operation:
  Node A ←──→ Node B ←──→ Node C
  (all nodes can communicate)

Network Partition:
  Node A ←──→ Node B    ✕    Node C
  (A and B can talk to each other, but neither can reach C)
  
  Partition 1: {A, B}
  Partition 2: {C}
```

**What Happens During a Partition:**
- Nodes on each side of the partition continue running
- They can't tell if the other side has crashed or if the network is broken
- Requests that need data from the other side will fail or timeout
- The system must choose: **reject requests** (consistency) or **serve potentially stale data** (availability) — this is the CAP theorem

**Partition Types:**

| Type | Description | Example |
|------|-------------|---------|
| **Complete partition** | No communication between groups | Data center network link cut |
| **Partial partition** | Some nodes can communicate, others can't | Node A can reach B and C, but B can't reach C |
| **Asymmetric partition** | A can send to B, but B can't send to A | Firewall misconfiguration, one-way link failure |
| **Intermittent partition** | Network flaps — works sometimes, fails sometimes | Congested network, flaky switch |

---

### Partial Failures

In a distributed system, **part of the system can fail while the rest continues working**. This is fundamentally different from a single machine, where either everything works or nothing works.

```
Single Machine:
  CPU fails → entire machine is down → clear failure

Distributed System:
  Node C fails → Nodes A, B, D continue working
  But: requests that need Node C will fail
       Node C might be slow (not crashed) — is it failed or just slow?
       Node C might have processed the request but the response was lost
```

**The Three Outcomes of a Network Request:**

```
Client sends request to Server:

  Outcome 1: SUCCESS
    Client → Request → Server → Response → Client
    (Clear: request succeeded)

  Outcome 2: FAILURE
    Client → Request → Server → Error → Client
    (Clear: request failed)

  Outcome 3: UNKNOWN (the hard case!)
    Client → Request → Server → ??? → Client (timeout)
    
    Did the server:
    a) Never receive the request? (safe to retry)
    b) Receive and process the request, but the response was lost? (NOT safe to retry — would duplicate!)
    c) Receive the request but crash before processing? (safe to retry)
    d) Receive the request, process it slowly, and will eventually respond? (retry would duplicate!)
    
    The client CANNOT distinguish between these cases!
    This is why idempotency is critical in distributed systems.
```

**Handling Partial Failures:**

| Strategy | Description |
|----------|-------------|
| **Timeouts** | Don't wait forever; fail after a reasonable time |
| **Retries** | Retry failed requests (with exponential backoff + jitter) |
| **Idempotency** | Make operations safe to retry (same result if executed multiple times) |
| **Circuit breakers** | Stop calling a failing service; return fallback |
| **Graceful degradation** | Return partial results when some components are unavailable |
| **Health checks** | Detect failed nodes and route traffic away from them |

---

### Clock Synchronization Issues

In a distributed system, each node has its own clock, and **clocks are never perfectly synchronized**. This creates problems when you need to order events across nodes.

**Why Clocks Drift:**
- Quartz crystal oscillators in computers drift by 10-100 parts per million
- That's 1-10 milliseconds per day, or up to 1 second per month
- NTP (Network Time Protocol) synchronizes clocks, but only to within 1-50ms accuracy
- Network delays make perfect synchronization impossible

**Problems Caused by Clock Skew:**

```
Problem 1: Event Ordering
  Node A: "User updated profile" at T=1000.000
  Node B: "User deleted account" at T=999.998
  
  Which happened first? Node B's clock is 2ms behind.
  If we use timestamps to order events, we get the WRONG order!
  The update appears to happen AFTER the delete.

Problem 2: Distributed Locks with Expiry
  Node A acquires lock at T=1000, expires at T=1005 (5 second lease)
  Node A's clock is slow by 3 seconds
  Node B's clock says T=1005 → lock expired → Node B acquires lock
  Node A's clock says T=1002 → lock still valid → Node A continues working
  → BOTH nodes think they hold the lock! Data corruption!

Problem 3: Cache TTL
  Node A sets cache entry with TTL=60s at T=1000 (expires at T=1060)
  Node B's clock is 30 seconds ahead
  Node B checks at its T=1060 (Node A's T=1030) → thinks entry expired
  → Premature cache invalidation
```

**Solutions:**

| Solution | Description | Accuracy |
|----------|-------------|----------|
| **NTP** | Synchronize clocks via network time servers | 1-50ms (LAN), 10-100ms (WAN) |
| **PTP (Precision Time Protocol)** | Hardware-assisted time sync | < 1 microsecond |
| **GPS clocks** | Dedicated GPS receivers for time | < 100 nanoseconds |
| **Google TrueTime** | GPS + atomic clocks in every data center; exposes uncertainty interval | < 7ms uncertainty |
| **Logical clocks** | Don't use physical time; use logical ordering (Lamport, Vector clocks) | Perfect ordering (no physical time) |

**Key Point:** Never rely on physical timestamps for ordering events in a distributed system. Use **logical clocks** (Lamport timestamps, vector clocks) or **consensus-based ordering** (Raft, Paxos) instead.

---


## 26.2 Consistency Models

> A **consistency model** defines the rules about when and how updates become visible to readers. Stronger consistency makes the system easier to reason about but harder to scale. Weaker consistency enables higher performance and availability but requires the application to handle inconsistencies.

---

### The Consistency Spectrum

```
Strongest ←──────────────────────────────────────────────→ Weakest
(easiest to program)                              (highest performance)

Linearizable → Sequential → Causal → Read-your-writes → Monotonic → Eventual
```

---

### Linearizability (Strongest)

**Linearizability** (also called "atomic consistency" or "strong consistency") guarantees that every operation appears to take effect **instantaneously** at some point between its invocation and completion. All nodes see the same order of operations, and that order is consistent with real-time.

```
Linearizable:
  T1: Client A writes X=1          (takes effect at some instant between start and end)
  T2: Client B writes X=2          (takes effect after A's write)
  T3: Client C reads X → gets 2    (sees the latest write)
  T4: Client D reads X → gets 2    (also sees the latest write — can never see 1)

  Once ANY client sees X=2, NO client can ever see X=1 again.
```

**Properties:**
- Every read returns the most recent write
- All operations have a total order consistent with real-time
- Once a value is read, all subsequent reads return that value or a newer one
- Behaves as if there's a single copy of the data

**Cost:** Requires coordination between nodes (consensus) → higher latency, lower throughput

**Used by:** Google Spanner, CockroachDB, etcd, ZooKeeper (for writes)

---

### Sequential Consistency

All operations appear to execute in **some sequential order**, and each client's operations appear in the order they were issued. But the global order doesn't need to match real-time.

```
Sequential Consistency:
  Client A: write X=1, then write X=2
  Client B: write X=3, then write X=4

  Valid sequential order: A:X=1, B:X=3, A:X=2, B:X=4 → final X=4
  Valid sequential order: B:X=3, A:X=1, A:X=2, B:X=4 → final X=4
  Valid sequential order: A:X=1, A:X=2, B:X=3, B:X=4 → final X=4
  
  All are valid as long as each client's operations maintain their order.
  (A's writes are always in order: X=1 before X=2)
  (B's writes are always in order: X=3 before X=4)
```

**Difference from Linearizability:** Sequential consistency doesn't require the order to match real-time. If Client A's write completes before Client B's write starts (in real-time), sequential consistency doesn't guarantee A's write appears first.

---

### Causal Consistency

Operations that are **causally related** are seen in the correct order by all nodes. Operations that are **not causally related** (concurrent) can be seen in different orders by different nodes.

```
Causal Relationship:
  Client A posts: "What's for lunch?"     (Event 1)
  Client B replies: "Pizza!"              (Event 2 — caused by Event 1)
  
  Causal consistency guarantees: ALL nodes see Event 1 before Event 2
  (You never see the reply before the question)

Concurrent (no causal relationship):
  Client A posts: "What's for lunch?"     (Event 1)
  Client C posts: "Nice weather today"    (Event 3 — independent of Event 1)
  
  Different nodes may see Event 1 and Event 3 in different orders.
  (That's OK — they're unrelated)
```

**How Causality is Tracked:**
- **Lamport timestamps** — partial ordering (can detect "happened before" but not concurrency)
- **Vector clocks** — full causal ordering (can detect both "happened before" and "concurrent")

**Used by:** MongoDB (causal consistency sessions), some Cassandra configurations

---

### Read-Your-Writes Consistency

A client always sees its own writes. After a client writes a value, subsequent reads by **that same client** will return the written value (or a newer one). Other clients may still see stale data.

```
Read-Your-Writes:
  Client A writes X=5 → reads X → gets 5 (always sees own write)
  Client B reads X → might get 3 (hasn't seen A's write yet — that's OK)
```

**Implementation:**
- Route the client's reads to the same replica that received the write
- Or track the client's last write timestamp and only read from replicas that are caught up
- Or use sticky sessions (route all of a client's requests to the same node)

**Used by:** DynamoDB (session consistency), many web applications

---

### Monotonic Reads

Once a client reads a value, it will **never see an older value** in subsequent reads. The client's view of the data only moves forward, never backward.

```
Monotonic Reads:
  Client reads X from Replica A → gets 5
  Client reads X from Replica B → gets 5 or newer (never 3)

Without Monotonic Reads:
  Client reads X from Replica A → gets 5
  Client reads X from Replica B → gets 3 (older! time went backward!)
  (This happens when Replica B is lagging behind Replica A)
```

**Implementation:** Route all of a client's reads to the same replica (sticky sessions), or track the version the client last read and reject older versions.

---

### Eventual Consistency (Weakest)

If no new updates are made, **all replicas will eventually converge** to the same value. There's no guarantee about when this will happen or what intermediate states clients will see.

```
Eventual Consistency:
  T1: Client writes X=5 to Node A
  T2: Client reads X from Node B → gets 3 (stale — replication hasn't reached B)
  T3: Client reads X from Node C → gets 3 (stale)
  T4: Replication reaches B and C
  T5: Client reads X from Node B → gets 5 (converged!)
  T6: Client reads X from Node C → gets 5 (converged!)
  
  The "eventual" window (T1 to T4) is typically milliseconds to seconds.
```

**Used by:** Cassandra (default), DynamoDB (default), DNS, CDN caches

---

### Linearizability vs Serializability

These terms are often confused but mean different things:

| Feature | Linearizability | Serializability |
|---------|----------------|-----------------|
| **Scope** | Single object (one register, one key) | Multiple objects (a transaction touching multiple rows) |
| **Guarantee** | Operations on one object appear instantaneous and ordered | Transactions appear to execute in some serial order |
| **Real-time** | Yes — order matches real-time | No — order just needs to be some valid serial order |
| **Context** | Distributed systems, replicated data | Database transactions, isolation levels |
| **Example** | "Read of X always returns the latest write to X" | "Transfer $100: debit and credit appear atomic" |

**Strict Serializability** = Linearizability + Serializability (both guarantees combined). This is what Google Spanner provides.

---


## 26.3 Consensus Algorithms

> **Consensus** is the problem of getting multiple nodes to agree on a single value, even when some nodes fail or messages are lost. It's the foundation of replicated state machines, leader election, distributed locks, and atomic broadcast. Consensus is provably impossible in a fully asynchronous system with even one faulty node (FLP impossibility result), but practical algorithms work by using timeouts to detect failures.

---

### Why Consensus is Hard

```
The Problem:
  3 nodes must agree on a value (e.g., "who is the leader?")
  
  Node A proposes: "I am the leader"
  Node B proposes: "I am the leader"
  Node C: network partition — can't communicate
  
  Requirements:
  1. Agreement: All non-faulty nodes decide on the SAME value
  2. Validity: The decided value was proposed by some node
  3. Termination: All non-faulty nodes eventually decide
  4. Integrity: Each node decides at most once
  
  Challenge: Node C might have already decided before the partition.
             Nodes A and B can't tell if C is crashed or just partitioned.
             If A and B decide without C, and C already decided differently → disagreement!
```

**FLP Impossibility (1985):**
Fischer, Lynch, and Paterson proved that in a **purely asynchronous** system (no timeouts, no clock), consensus is impossible if even **one node can crash**. This is because you can't distinguish a crashed node from a very slow one.

**Practical Solution:** Use **timeouts** to suspect failures. This makes the system partially synchronous — consensus algorithms work correctly in practice even though they can't guarantee termination in all theoretical scenarios.

---

### Paxos (Basic Idea)

**Paxos** (Leslie Lamport, 1989) is the foundational consensus algorithm. It's notoriously difficult to understand but is the basis for many production systems.

**Roles:**
- **Proposer:** Proposes a value
- **Acceptor:** Votes on proposals (majority needed)
- **Learner:** Learns the decided value

**Two Phases:**

```
Phase 1: PREPARE (proposer asks for permission)
  Proposer → all Acceptors: "Prepare(n)" (n = proposal number, must be unique and increasing)
  Acceptor → Proposer: "Promise(n)" (I promise not to accept proposals with number < n)
                        + any previously accepted value

Phase 2: ACCEPT (proposer asks for agreement)
  Proposer → all Acceptors: "Accept(n, value)" (please accept this value with proposal number n)
  Acceptor → Proposer: "Accepted(n, value)" (I accept)
  
  If a MAJORITY of acceptors accept → value is DECIDED
  Proposer → all Learners: "Decided(value)"
```

```
Proposer          Acceptor 1       Acceptor 2       Acceptor 3
   |                  |                |                |
   |-- Prepare(1) -->|                |                |
   |-- Prepare(1) ------------------>|                |
   |-- Prepare(1) ---------------------------------->|
   |                  |                |                |
   |<- Promise(1) ---|                |                |
   |<- Promise(1) -------------------|                |
   |<- Promise(1) -----------------------------------|
   |                  |                |                |
   | (majority promised — proceed to phase 2)          |
   |                  |                |                |
   |-- Accept(1,v) ->|                |                |
   |-- Accept(1,v) ------------------>|                |
   |-- Accept(1,v) ---------------------------------->|
   |                  |                |                |
   |<- Accepted(1,v)-|                |                |
   |<- Accepted(1,v)-----------------|                |
   |                  |                |                |
   | (majority accepted — value v is DECIDED)          |
```

**Why Paxos is Hard:**
- Multiple proposers can conflict (dueling proposals)
- Requires careful handling of proposal numbers
- Multi-Paxos (for a sequence of decisions) is even more complex
- The original paper is famously difficult to understand

---

### Raft (Leader Election, Log Replication, Safety)

**Raft** (Diego Ongaro, 2014) was designed to be an **understandable** alternative to Paxos. It achieves the same guarantees but with a clearer structure.

**Key Idea:** Raft decomposes consensus into three sub-problems:
1. **Leader Election** — choose one node as the leader
2. **Log Replication** — leader replicates its log to followers
3. **Safety** — ensure all nodes agree on the same log

**Node States:**

```
+----------+     Timeout      +-----------+     Wins election     +--------+
| Follower | ───────────────→ | Candidate | ─────────────────────→| Leader |
|          | ←─────────────── |           | ←───────────────────── |        |
+----------+  Discovers leader+-----------+  Discovers higher term +--------+
      ↑                                                                |
      |                    Discovers higher term                       |
      +────────────────────────────────────────────────────────────────+
```

**Leader Election:**

```
1. All nodes start as FOLLOWERS
2. Followers expect heartbeats from the leader
3. If a follower doesn't receive a heartbeat within the ELECTION TIMEOUT (150-300ms random):
   → It becomes a CANDIDATE
   → Increments its TERM number
   → Votes for itself
   → Sends RequestVote to all other nodes

4. Other nodes vote:
   → Each node votes for at most ONE candidate per term
   → Vote for the candidate if its log is at least as up-to-date

5. If candidate receives votes from a MAJORITY:
   → Becomes the LEADER
   → Sends heartbeats to all followers
   → Handles all client requests

6. If no candidate wins (split vote):
   → Election timeout expires → new election with higher term
   → Random timeouts prevent repeated split votes
```

```
Term 1:                    Term 2:
  Leader: Node A             Leader: Node B (A failed)
  
  Node A: Leader ──→ Crash!
  Node B: Follower ──→ Timeout ──→ Candidate ──→ Wins election ──→ Leader
  Node C: Follower ──→ Votes for B ──→ Follower (of B)
```

**Log Replication:**

```
1. Client sends command to Leader
2. Leader appends command to its log
3. Leader sends AppendEntries RPC to all followers
4. Followers append to their logs and acknowledge
5. When a MAJORITY has acknowledged:
   → Leader COMMITS the entry (applies to state machine)
   → Leader responds to client
   → Leader notifies followers to commit

Leader Log:    [1:SET X=1] [2:SET Y=2] [3:SET X=3] ← committed
Follower A:   [1:SET X=1] [2:SET Y=2] [3:SET X=3] ← replicated
Follower B:   [1:SET X=1] [2:SET Y=2]              ← slightly behind (will catch up)
```

**Safety Guarantees:**
- **Election Safety:** At most one leader per term
- **Leader Append-Only:** Leader never overwrites or deletes log entries
- **Log Matching:** If two logs contain an entry with the same index and term, all preceding entries are identical
- **Leader Completeness:** If an entry is committed, it will be present in the logs of all future leaders
- **State Machine Safety:** If a node applies a log entry at a given index, no other node applies a different entry at that index

**Raft in Production:**

| System | Uses Raft For |
|--------|--------------|
| **etcd** | Distributed key-value store (Kubernetes cluster state) |
| **CockroachDB** | Distributed SQL database (range-level consensus) |
| **TiKV** | Distributed key-value store (TiDB storage layer) |
| **Consul** | Service discovery and configuration |
| **HashiCorp Vault** | Secrets management (HA mode) |
| **RabbitMQ** | Quorum queues (since 3.8) |

---

### ZAB (ZooKeeper Atomic Broadcast)

**ZAB** is the consensus protocol used by Apache ZooKeeper. It's similar to Raft but was developed independently and has some differences.

**Key Differences from Raft:**

| Feature | Raft | ZAB |
|---------|------|-----|
| Leader election | Random timeouts, majority vote | Epoch-based, longest log wins |
| Log replication | AppendEntries RPC | Proposal + Commit (two-phase) |
| Recovery | Leader sends missing entries to followers | Synchronization phase after election |
| Terminology | Term, Leader, Follower | Epoch, Leader, Follower |

**ZAB is used by:** Apache ZooKeeper (which is used by Kafka, HBase, Hadoop, Solr)

---

### Byzantine Fault Tolerance (BFT) — Overview

All the algorithms above assume **crash failures** — nodes either work correctly or crash. **Byzantine failures** are worse: nodes can behave **arbitrarily** — sending wrong data, lying, or acting maliciously.

```
Crash Failure:
  Node fails → stops responding → other nodes detect the failure
  (Honest failure — node doesn't lie)

Byzantine Failure:
  Node is compromised → sends WRONG data to different nodes
  Node A tells Node B: "The value is 5"
  Node A tells Node C: "The value is 7"
  (Malicious or buggy — node actively lies)
```

**Byzantine Generals Problem:**
- N generals must agree on a battle plan (attack or retreat)
- Some generals are traitors (send conflicting messages)
- Requirement: all loyal generals agree on the same plan
- **Result:** Requires at least 3f+1 nodes to tolerate f Byzantine failures (need 2/3 majority)

**BFT Algorithms:**

| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| **PBFT** (Practical BFT) | Classic BFT algorithm; 3f+1 nodes for f failures | Permissioned blockchains |
| **Tendermint** | BFT consensus for blockchains | Cosmos blockchain |
| **HotStuff** | Optimized BFT with linear communication | Diem/Libra blockchain |

**When BFT is Needed:**
- Blockchain / cryptocurrency (untrusted participants)
- Multi-party computation (mutually distrusting organizations)
- Safety-critical systems (aerospace, nuclear)

**When BFT is NOT Needed (most systems):**
- Internal microservices (you trust your own code)
- Cloud infrastructure (you trust the cloud provider)
- Traditional databases (crash failures are sufficient)

**Key Point:** Most system design interviews and production systems only need crash fault tolerance (Raft, Paxos). BFT is for specialized use cases.

---


## 26.4 Distributed Coordination

> Distributed coordination services provide primitives like **leader election**, **distributed locking**, **configuration management**, and **service discovery** that are essential for building reliable distributed systems. Instead of implementing these primitives yourself (which is extremely error-prone), use a battle-tested coordination service.

---

### ZooKeeper

**Apache ZooKeeper** is a centralized service for distributed coordination. It provides a hierarchical key-value store (like a file system) with strong consistency guarantees.

**ZooKeeper Data Model:**

```
/                           (root)
├── /services               (service registry)
│   ├── /services/user-service
│   │   ├── /services/user-service/instance-001  → "10.0.1.5:8080"
│   │   └── /services/user-service/instance-002  → "10.0.1.6:8080"
│   └── /services/order-service
│       └── /services/order-service/instance-001 → "10.0.2.5:8080"
├── /config                 (configuration)
│   ├── /config/database-url → "postgres://..."
│   └── /config/feature-flags → '{"dark_mode": true}'
├── /locks                  (distributed locks)
│   └── /locks/order-processing → (ephemeral node, held by lock owner)
└── /election               (leader election)
    ├── /election/candidate-001  → (ephemeral sequential)
    └── /election/candidate-002  → (ephemeral sequential)
```

**ZNode Types:**

| Type | Description | Use Case |
|------|-------------|----------|
| **Persistent** | Exists until explicitly deleted | Configuration, metadata |
| **Ephemeral** | Automatically deleted when the creating session ends (client disconnects) | Service registration, locks, leader election |
| **Sequential** | ZooKeeper appends a monotonically increasing number to the name | Leader election (lowest number wins), queue ordering |
| **Ephemeral + Sequential** | Combines both properties | Leader election with automatic failover |

---

**Leader Election with ZooKeeper:**

```
1. Each candidate creates an EPHEMERAL SEQUENTIAL node under /election:
   Node A creates: /election/candidate-00001
   Node B creates: /election/candidate-00002
   Node C creates: /election/candidate-00003

2. The node with the LOWEST sequence number is the leader:
   /election/candidate-00001 (Node A) → LEADER
   /election/candidate-00002 (Node B) → follower (watches 00001)
   /election/candidate-00003 (Node C) → follower (watches 00002)

3. If the leader (Node A) crashes:
   → Ephemeral node /election/candidate-00001 is automatically deleted
   → Node B's watch fires → Node B checks: am I the lowest? YES → Node B becomes LEADER
   → Node C's watch fires → Node C checks: am I the lowest? NO → watches Node B

4. Herd effect prevention:
   Each node watches ONLY the node immediately before it (not the leader).
   When the leader dies, only the next node is notified — not all nodes.
```

---

**Distributed Locking with ZooKeeper:**

```
Acquiring a Lock:
  1. Client creates EPHEMERAL SEQUENTIAL node: /locks/resource-001/lock-00001
  2. Client gets all children of /locks/resource-001
  3. If client's node has the LOWEST sequence number → lock acquired!
  4. Otherwise → watch the node with the next-lower sequence number → wait

Releasing a Lock:
  1. Client deletes its node: /locks/resource-001/lock-00001
  2. Next waiter's watch fires → it checks if it now has the lowest number → acquires lock

Crash Safety:
  If the lock holder crashes → ephemeral node is automatically deleted
  → Next waiter acquires the lock → no deadlock!
```

**Why ZooKeeper Locks are Better Than Redis Locks:**
- **Ephemeral nodes** — lock is automatically released if the holder crashes (no stale locks)
- **Sequential ordering** — fair lock (FIFO ordering of waiters)
- **Watch mechanism** — efficient notification (no polling)
- **Strong consistency** — ZooKeeper uses ZAB consensus; no split-brain

---

**Configuration Management:**

```
Store configuration in ZooKeeper:
  SET /config/database-url = "postgres://primary:5432/mydb"
  SET /config/feature-flags = '{"dark_mode": true, "new_checkout": false}'

Applications watch for changes:
  WATCH /config/database-url
  → When the value changes, all watchers are notified immediately
  → Applications reload configuration without restart
```

---

### etcd

**etcd** is a distributed key-value store that uses **Raft** for consensus. It's the backbone of Kubernetes (stores all cluster state).

| Feature | Details |
|---------|---------|
| **Consensus** | Raft (simpler than ZAB) |
| **Data model** | Flat key-value (with prefix-based "directories") |
| **API** | gRPC + HTTP/JSON |
| **Watch** | Efficient watch on key prefixes (streaming changes) |
| **Lease** | TTL-based ephemeral keys (like ZooKeeper ephemeral nodes) |
| **Transactions** | Mini-transactions (if-then-else on key comparisons) |
| **Used by** | Kubernetes (cluster state), CoreDNS, Vitess |

**etcd vs ZooKeeper:**

| Feature | etcd | ZooKeeper |
|---------|------|-----------|
| Consensus | Raft | ZAB |
| Language | Go | Java |
| Data model | Flat key-value | Hierarchical (tree) |
| API | gRPC (modern) | Custom protocol |
| Watch | Prefix-based, streaming | Per-node watches (must re-register) |
| Ephemeral keys | Lease-based TTL | Session-based |
| Kubernetes | Native (built for K8s) | Not used by K8s |
| Operational | Simpler | More complex |

---

### Consul

**Consul** (HashiCorp) is a service mesh and distributed coordination tool that combines service discovery, configuration, and segmentation.

| Feature | Details |
|---------|---------|
| **Service discovery** | HTTP and DNS interfaces; health checking built-in |
| **Key-value store** | Distributed KV with watches and transactions |
| **Service mesh** | Consul Connect — sidecar proxies with mTLS |
| **Health checking** | HTTP, TCP, script-based, gRPC health checks |
| **Multi-datacenter** | Native multi-DC support with WAN gossip |
| **Consensus** | Raft (for KV store and leader election) |
| **Gossip** | Serf (for membership and failure detection) |

**When to Use Which:**

| Need | Recommendation |
|------|---------------|
| Kubernetes cluster state | etcd (built-in) |
| Service discovery + health checks | Consul |
| Distributed locking with strong guarantees | ZooKeeper |
| Configuration management | etcd or Consul |
| Service mesh | Consul Connect or Istio |
| Kafka/HBase coordination | ZooKeeper (legacy) or KRaft (Kafka without ZK) |

---


## 26.5 Distributed Transactions

> When a single business operation spans multiple services or databases, you need a way to ensure that either **all parts succeed** or **all parts are rolled back**. Distributed transactions are much harder than local transactions because there's no single database to coordinate the ACID guarantees.

---

### Two-Phase Commit (2PC)

**2PC** is a protocol that ensures all participants in a distributed transaction either **all commit** or **all abort**.

```
Phase 1: PREPARE (voting phase)
  Coordinator → Participant A: "Can you commit?"
  Coordinator → Participant B: "Can you commit?"
  Coordinator → Participant C: "Can you commit?"
  
  Each participant:
    - Executes the transaction locally (but doesn't commit yet)
    - Writes to its local WAL (for recovery)
    - Responds: "YES, I can commit" or "NO, I must abort"

Phase 2: COMMIT or ABORT (decision phase)
  If ALL participants voted YES:
    Coordinator → All: "COMMIT"
    Each participant commits and releases locks
  
  If ANY participant voted NO (or timed out):
    Coordinator → All: "ABORT"
    Each participant rolls back and releases locks
```

```
Coordinator        Participant A      Participant B
    |                   |                  |
    |-- Prepare ------->|                  |
    |-- Prepare ------------------------------>|
    |                   |                  |
    |<-- YES -----------|                  |
    |<-- YES ----------------------------------|
    |                   |                  |
    | (all voted YES)   |                  |
    |                   |                  |
    |-- Commit -------->|                  |
    |-- Commit ------------------------------>|
    |                   |                  |
    |<-- ACK -----------|                  |
    |<-- ACK ----------------------------------|
    |                   |                  |
    | DONE              |                  |
```

**2PC Problems:**

| Problem | Description |
|---------|-------------|
| **Blocking** | If the coordinator crashes after Phase 1 (after participants voted YES), participants are stuck — they can't commit or abort until the coordinator recovers. They hold locks the entire time. |
| **Single point of failure** | The coordinator is a SPOF. If it crashes, the entire transaction is stuck. |
| **Latency** | Two round trips (prepare + commit) add significant latency |
| **Reduced availability** | ALL participants must be available for the transaction to succeed |
| **Lock contention** | Participants hold locks during both phases — reduces throughput |

---

### Three-Phase Commit (3PC)

**3PC** adds a **pre-commit** phase to reduce the blocking problem of 2PC.

```
Phase 1: CAN-COMMIT (voting)
  Coordinator → Participants: "Can you commit?"
  Participants → Coordinator: "YES" or "NO"

Phase 2: PRE-COMMIT (prepare to commit)
  If all voted YES:
    Coordinator → Participants: "Pre-commit" (prepare, but don't commit yet)
    Participants → Coordinator: "ACK"
  
  If any voted NO:
    Coordinator → Participants: "Abort"

Phase 3: DO-COMMIT (final commit)
  Coordinator → Participants: "Commit"
  Participants commit and acknowledge
```

**3PC vs 2PC:**

| Feature | 2PC | 3PC |
|---------|-----|-----|
| Phases | 2 | 3 |
| Blocking | Yes (if coordinator crashes after prepare) | Reduced (participants can decide based on pre-commit state) |
| Latency | 2 round trips | 3 round trips (higher) |
| Complexity | Moderate | Higher |
| Used in practice | Yes (widely used) | Rarely (3PC is mostly theoretical) |

**Key Point:** 3PC is rarely used in practice because it adds latency and complexity. Modern systems prefer the **Saga pattern** for distributed transactions.

---

### Saga Pattern

The **Saga pattern** breaks a distributed transaction into a sequence of **local transactions**, each with a **compensating action** for rollback. Covered in depth in Module 22.5 and Module 25.4.

**Quick Comparison with 2PC:**

| Feature | 2PC | Saga |
|---------|-----|------|
| Consistency | Strong (ACID) | Eventual (each step is a local transaction) |
| Availability | Low (all participants must be available) | High (each step is independent) |
| Latency | High (2 round trips, locks held) | Lower (no global locks) |
| Complexity | Protocol complexity | Business logic complexity (compensating actions) |
| Lock duration | Long (entire transaction) | Short (each local transaction) |
| Isolation | Full (global locks) | None (intermediate states are visible) |
| Best for | Short transactions, few participants | Long-running transactions, microservices |

---

### Compensating Transactions

A **compensating transaction** undoes the effect of a previously committed transaction. It's the "rollback" mechanism in the Saga pattern.

```
Forward Transaction:          Compensating Transaction:
  Create Order                  Cancel Order
  Charge Payment                Refund Payment
  Reserve Inventory             Release Inventory
  Create Shipment               Cancel Shipment
  Send Confirmation Email       Send Cancellation Email
```

**Compensating Transaction Challenges:**

| Challenge | Description | Solution |
|-----------|-------------|---------|
| **Not always possible** | You can't "unsend" an email or "unship" a package | Design compensations that are "good enough" (send cancellation email, recall shipment) |
| **Semantic rollback** | Compensation may not restore the exact original state | Accept that compensation is a "best effort" undo |
| **Ordering** | Compensations must execute in reverse order | Saga orchestrator tracks the order |
| **Idempotency** | Compensation may be retried (network failures) | Make compensating transactions idempotent |
| **Concurrency** | Other transactions may have read intermediate state | Accept eventual consistency; use version checks |

---

### Outbox Pattern

The **Outbox pattern** ensures that a database write and a message publish happen **atomically** — either both succeed or neither does. This solves the dual-write problem.

**The Dual-Write Problem:**

```
❌ Dual Write (unsafe):
  1. Write to database: INSERT INTO orders (id, ...) VALUES (1, ...)  → SUCCESS
  2. Publish to Kafka: "OrderCreated" event                           → FAILS!
  
  Result: Order exists in DB but no event was published.
  Other services never learn about the order!

  Or worse:
  1. Publish to Kafka: "OrderCreated" event                           → SUCCESS
  2. Write to database: INSERT INTO orders (id, ...) VALUES (1, ...)  → FAILS!
  
  Result: Event published but order doesn't exist in DB.
  Other services process a phantom order!
```

**Outbox Pattern Solution:**

```
✅ Outbox Pattern (safe):
  1. Within a SINGLE database transaction:
     a. INSERT INTO orders (id, ...) VALUES (1, ...)
     b. INSERT INTO outbox (id, event_type, payload) VALUES (uuid, 'OrderCreated', '{...}')
     COMMIT;  (both writes are atomic — same database, same transaction)

  2. A separate process (Outbox Relay / CDC) reads the outbox table and publishes to Kafka:
     SELECT * FROM outbox WHERE published = false;
     → Publish to Kafka
     → UPDATE outbox SET published = true WHERE id = ...
     (or use CDC like Debezium to capture outbox changes automatically)
```

```
  +--------+     Single TX      +----------+
  | App    | ──────────────────→| Database |
  | Server |  INSERT order      | orders   |
  |        |  INSERT outbox     | outbox   |
  +--------+                    +----+-----+
                                     |
                              +------v------+
                              | Outbox Relay|  (polls outbox table)
                              | or Debezium |  (CDC — reads WAL)
                              +------+------+
                                     |
                              +------v------+
                              |   Kafka     |
                              | "OrderCreated"|
                              +-------------+
```

**Outbox Pattern Guarantees:**
- **At-least-once delivery:** The relay may publish the same event twice (if it crashes after publishing but before marking as published). Consumers must be idempotent.
- **Ordering:** Events are published in the order they were written to the outbox (if using a single partition or ordered relay).
- **No data loss:** The event is in the database (durable) before it's published.

---


## 26.6 Time & Ordering

> Ordering events correctly is one of the hardest problems in distributed systems. Physical clocks are unreliable (they drift), and there's no global clock that all nodes share. **Logical clocks** solve this by tracking causality — the "happened before" relationship — without relying on physical time.

---

### Physical Clocks vs Logical Clocks

| Feature | Physical Clocks | Logical Clocks |
|---------|----------------|---------------|
| **What they measure** | Wall-clock time (seconds since epoch) | Causal ordering (what happened before what) |
| **Accuracy** | Imperfect (drift, NTP sync within 1-50ms) | Perfect (no drift — it's a counter) |
| **Cross-node comparison** | Unreliable (clocks may disagree) | Reliable (consistent ordering) |
| **Use case** | Timestamps for humans, TTLs, rough ordering | Event ordering, conflict detection, causality tracking |
| **Examples** | `System.currentTimeMillis()`, `time.time()` | Lamport timestamps, vector clocks |

---

### Lamport Timestamps

**Lamport timestamps** (Leslie Lamport, 1978) assign a logical timestamp to every event such that if event A **happened before** event B, then `timestamp(A) < timestamp(B)`.

**Rules:**
1. Each node maintains a counter `C` (starts at 0)
2. Before each local event: `C = C + 1`
3. When sending a message: attach `C` to the message
4. When receiving a message with timestamp `T`: `C = max(C, T) + 1`

```
Node A (counter=0)          Node B (counter=0)          Node C (counter=0)
    |                           |                           |
    | Event a1 (C=1)            |                           |
    |                           |                           |
    |--- msg (T=1) ----------->|                           |
    |                           | Receive: C=max(0,1)+1=2  |
    |                           | Event b1 (C=2)            |
    |                           |                           |
    |                           |--- msg (T=2) ----------->|
    |                           |                           | Receive: C=max(0,2)+1=3
    |                           |                           | Event c1 (C=3)
    |                           |                           |
    | Event a2 (C=2)            |                           |
    |                           |                           |
    |--- msg (T=2) ---------------------------------------->|
    |                           |                           | Receive: C=max(3,2)+1=4
    |                           |                           | Event c2 (C=4)
```

**Lamport Timestamp Properties:**
- If A happened before B: `L(A) < L(B)` ✅ (guaranteed)
- If `L(A) < L(B)`: A **might** have happened before B, or they might be concurrent ⚠️
- **Cannot detect concurrent events** — this is the limitation

**The "happened before" relation (→):**
- If A and B are events on the same node and A occurs before B: A → B
- If A is a send event and B is the corresponding receive event: A → B
- Transitivity: if A → B and B → C, then A → C
- If neither A → B nor B → A: A and B are **concurrent** (A ‖ B)

---

### Vector Clocks

**Vector clocks** extend Lamport timestamps to detect **concurrent events**. Each node maintains a vector of counters — one counter per node in the system.

**Rules:**
1. Each node `i` maintains a vector `V[i]` of size N (one entry per node)
2. Before each local event on node `i`: `V[i][i] = V[i][i] + 1`
3. When sending a message: attach the entire vector `V[i]`
4. When receiving a message with vector `V_msg` on node `j`:
   - For each entry `k`: `V[j][k] = max(V[j][k], V_msg[k])`
   - Then: `V[j][j] = V[j][j] + 1`

```
Node A                    Node B                    Node C
V=[0,0,0]                V=[0,0,0]                V=[0,0,0]
    |                         |                         |
    | Event a1                |                         |
    | V=[1,0,0]               |                         |
    |                         |                         |
    |--- msg V=[1,0,0] ----->|                         |
    |                         | Receive:                |
    |                         | V=max([0,0,0],[1,0,0])  |
    |                         |  =[1,0,0]               |
    |                         | V[B]++ → V=[1,1,0]      |
    |                         |                         |
    |                         |--- msg V=[1,1,0] ----->|
    |                         |                         | Receive:
    |                         |                         | V=[1,1,1]
    |                         |                         |
    | Event a2                |                         |
    | V=[2,0,0]               |                         |
```

**Comparing Vector Clocks:**

```
V1 = [2, 3, 1]
V2 = [1, 4, 2]

V1 ≤ V2? Check each entry: 2≤1? NO → V1 is NOT ≤ V2
V2 ≤ V1? Check each entry: 1≤2? YES, 4≤3? NO → V2 is NOT ≤ V1

Neither V1 ≤ V2 nor V2 ≤ V1 → CONCURRENT events!

V3 = [1, 2, 1]
V4 = [2, 3, 2]

V3 ≤ V4? 1≤2? YES, 2≤3? YES, 1≤2? YES → V3 happened before V4
```

**Ordering Rules:**
- `V1 < V2` (V1 happened before V2): every entry in V1 is ≤ the corresponding entry in V2, and at least one is strictly less
- `V1 = V2`: all entries are equal (same event)
- `V1 ‖ V2` (concurrent): neither V1 ≤ V2 nor V2 ≤ V1

**Vector Clock Limitations:**
- Vector size grows with the number of nodes (N entries per vector)
- Not practical for systems with thousands of nodes
- Solutions: dotted version vectors, interval tree clocks

**Used by:** Amazon Dynamo (original paper), Riak, conflict detection in multi-master replication

---

### Hybrid Logical Clocks (HLC)

**HLC** combines the best of physical and logical clocks:
- Uses physical time as the base (human-readable, roughly ordered)
- Adds a logical component to handle cases where physical clocks are insufficient
- Bounded by physical time (HLC timestamp is always close to real time)

```
HLC = (physical_time, logical_counter)

Node A at physical time 100: HLC = (100, 0)
Node A sends message at physical time 100: HLC = (100, 0)
Node B receives at physical time 99 (clock behind!):
  HLC = (max(99, 100), 0+1) = (100, 1)
  (Uses the sender's physical time since it's higher, increments logical counter)

Node B local event at physical time 101:
  HLC = (101, 0)  (physical time advanced past the logical component)
```

**HLC Properties:**
- Always close to physical time (bounded drift)
- Captures causality (like Lamport timestamps)
- Compact (just two numbers, not a vector)
- Human-readable (the physical component is a real timestamp)

**Used by:** CockroachDB, MongoDB (internal), YugabyteDB

---

### NTP (Network Time Protocol)

**NTP** synchronizes clocks across the internet using a hierarchy of time servers.

```
NTP Hierarchy (Stratum):
  Stratum 0: Atomic clocks, GPS receivers (reference clocks)
  Stratum 1: Servers directly connected to Stratum 0 (< 1ms accuracy)
  Stratum 2: Servers synced to Stratum 1 (< 10ms accuracy)
  Stratum 3: Servers synced to Stratum 2 (< 50ms accuracy)
  ...
  Stratum 15: Maximum (Stratum 16 = unsynchronized)
```

**NTP Accuracy:**

| Environment | Typical Accuracy |
|-------------|-----------------|
| LAN (same data center) | 0.1 - 1 ms |
| WAN (across internet) | 1 - 50 ms |
| Congested network | 10 - 100 ms |
| Google TrueTime (GPS + atomic) | < 7 ms (with known uncertainty bounds) |

**Google TrueTime:**
Google's TrueTime API returns a time **interval** `[earliest, latest]` instead of a single timestamp. The true time is guaranteed to be within this interval. Spanner uses this to implement linearizable transactions — it waits for the uncertainty interval to pass before committing.

```
TrueTime.now() → [T - ε, T + ε]  where ε ≈ 1-7ms

Spanner commit protocol:
  1. Assign commit timestamp T
  2. Wait until TrueTime.now().earliest > T  (wait out the uncertainty)
  3. Commit is now guaranteed to be in the past for all nodes
  → Linearizable without consensus for read-only transactions!
```

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Distributed System | Multiple networked computers with no shared memory or clock; communicate by messages |
| 8 Fallacies | Network is unreliable, latency is non-zero, bandwidth is finite — design for failure |
| Network Partitions | Inevitable in distributed systems; must choose consistency or availability (CAP) |
| Partial Failures | Part of the system can fail while the rest works; the "unknown" outcome is the hardest case |
| Clock Skew | Physical clocks drift; never use timestamps for ordering events across nodes |
| Linearizability | Strongest consistency — every read returns the latest write; requires consensus |
| Eventual Consistency | Weakest — replicas converge eventually; highest performance and availability |
| Causal Consistency | Causally related events are ordered; concurrent events may differ across nodes |
| Linearizability vs Serializability | Linearizability = single object, real-time; Serializability = multi-object transactions |
| Paxos | Foundational consensus algorithm; correct but hard to understand and implement |
| Raft | Understandable consensus; leader election + log replication; used by etcd, CockroachDB |
| ZAB | ZooKeeper's consensus protocol; similar to Raft |
| Byzantine Fault Tolerance | Handles malicious nodes; needs 3f+1 nodes; used in blockchains, not typical systems |
| ZooKeeper | Distributed coordination: leader election, locks, config, service discovery |
| etcd | Raft-based KV store; Kubernetes cluster state; simpler than ZooKeeper |
| Consul | Service discovery + KV store + service mesh; multi-datacenter |
| 2PC | All-or-nothing distributed commit; blocking if coordinator crashes |
| Saga Pattern | Sequence of local transactions with compensating actions; preferred over 2PC for microservices |
| Outbox Pattern | Atomic DB write + event publish; solves the dual-write problem |
| Lamport Timestamps | Logical clock; captures "happened before" but can't detect concurrency |
| Vector Clocks | Logical clock; captures both "happened before" and "concurrent"; size grows with nodes |
| HLC | Hybrid of physical + logical; bounded by real time; used by CockroachDB |
| NTP | Synchronizes physical clocks; 1-50ms accuracy; not sufficient for event ordering |
| Google TrueTime | GPS + atomic clocks; returns uncertainty interval; enables Spanner's linearizability |

---

## Interview Tips for Module 26

1. **8 Fallacies** — mention at least 3-4 fallacies and explain their practical impact
2. **Partial failures** — explain the "unknown" outcome and why idempotency is critical
3. **Consistency models** — draw the spectrum from linearizable to eventual; explain each level with an example
4. **Linearizability vs Serializability** — know the difference; this is a common trick question
5. **Raft** — explain leader election and log replication; draw the state diagram (follower → candidate → leader)
6. **Paxos vs Raft** — know that Raft is the practical choice; Paxos is the theoretical foundation
7. **ZooKeeper** — explain leader election with ephemeral sequential nodes; explain distributed locking
8. **etcd vs ZooKeeper** — etcd is simpler, Raft-based, used by Kubernetes; ZooKeeper is older, used by Kafka
9. **2PC** — explain the two phases; explain the blocking problem; explain why Saga is preferred for microservices
10. **Outbox pattern** — explain the dual-write problem and how the outbox solves it; mention Debezium for CDC
11. **Lamport vs Vector clocks** — Lamport can't detect concurrency; vector clocks can but don't scale
12. **HLC** — mention as the practical choice; combines physical time with logical ordering; used by CockroachDB
