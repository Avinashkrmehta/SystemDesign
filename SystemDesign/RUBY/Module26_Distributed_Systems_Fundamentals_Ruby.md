# Module 26: Distributed Systems Fundamentals

> A distributed system is a collection of independent computers that appears to its users as a single coherent system. Every large-scale application — from Google Search to Netflix streaming to your bank's transaction system — is a distributed system. This module covers the foundational theory: why distributed systems are hard, the consistency models that govern them, the consensus algorithms that keep them coordinated, and the transaction patterns that keep them correct.

> **Ruby Context:** Ruby applications frequently operate within distributed systems — a Rails app backed by PostgreSQL replicas, Redis caching, Sidekiq workers, and Kafka event streams is already a distributed system. Understanding distributed fundamentals helps Ruby developers reason about replication lag, cache consistency, job idempotency, and failure modes. Key Ruby tools in this space include **redis-rb** (distributed caching and locking), **redlock-rb** (distributed locks via Redis), **diplomat** (Consul client), **etcdv3** (etcd client), **zk** / **zookeeper** gems (ZooKeeper client), **Sidekiq** (at-least-once job processing), and **Karafka** (Kafka consumers with offset management). Rails itself provides `ActiveRecord` transactions, `Rails.cache` with configurable backends, and `ActiveJob` for async processing — all of which interact with distributed system concerns.

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

> **Ruby Context:** A typical Rails production stack is already distributed: the Rails app server (Puma), PostgreSQL primary + replicas, Redis (cache + Sidekiq broker), Elasticsearch, and possibly Kafka. Each of these runs on separate machines with separate clocks and communicates over the network. Understanding distributed system properties helps explain why your Sidekiq job might process the same message twice, why a read from a PostgreSQL replica might return stale data, or why a Redis lock might not be as safe as you think.

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
  Developer writes: users.each { |u| fetch_profile(u.id) }  # 100 users × 10ms = 1 second!
  Assumes: remote calls are as fast as local calls
  Reality: each network call adds 1-100ms of latency
  Fix: batch_fetch_profiles(users.map(&:id))  # 1 call × 15ms = 15ms
```

> **Ruby Context:** Ruby developers commonly hit Fallacy 2 with N+1 queries across services. Within a single Rails app, `includes` solves N+1 for database queries. But when calling external services in a loop (e.g., fetching user profiles from a User Service for each order), there's no ORM to help — you must explicitly batch requests. Faraday with `faraday-retry` middleware addresses Fallacy 1 by adding retries and timeouts to HTTP calls.

```ruby
# Fallacy 1: Assuming the network is reliable
# ❌ Bad — no timeout, no retry, no error handling
response = Faraday.get("#{ENV['USER_SERVICE_URL']}/api/v1/users/#{user_id}")

# ✅ Good — timeout, retry, error handling
client = Faraday.new(url: ENV['USER_SERVICE_URL']) do |f|
  f.request :retry, max: 3, interval: 0.5, backoff_factor: 2,
                    retry_statuses: [429, 500, 502, 503, 504]
  f.options.timeout = 5
  f.options.open_timeout = 2
  f.response :raise_error
end

begin
  response = client.get("/api/v1/users/#{user_id}")
rescue Faraday::Error => e
  Rails.logger.error("User service call failed: #{e.message}")
  nil  # Graceful degradation
end

# Fallacy 2: Assuming latency is zero
# ❌ Bad — N+1 across services
orders.each do |order|
  order.user_profile = UserServiceClient.fetch(order.user_id)  # 100 calls!
end

# ✅ Good — batch request
user_ids = orders.map(&:user_id).uniq
profiles = UserServiceClient.batch_fetch(user_ids)  # 1 call
orders.each { |order| order.user_profile = profiles[order.user_id] }
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

> **Ruby Context:** A common partition scenario for Rails apps: the app server can reach Redis but not PostgreSQL (or vice versa). If Sidekiq can't reach the database but can reach Redis, jobs will be dequeued but fail to process. If the app can reach PostgreSQL but not Redis, caching and Sidekiq job enqueuing will fail. Designing for these partial failures means having proper timeouts, circuit breakers (Stoplight gem), and graceful degradation paths.

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

> **Ruby Context:** This "unknown" outcome is exactly why Sidekiq jobs must be idempotent. When a Sidekiq worker processes a job and crashes before acknowledging completion, the job is retried — potentially causing duplicate processing. The same applies to webhook handlers, payment callbacks, and any operation triggered by an external event.

```ruby
# The "unknown" outcome in practice — Sidekiq job idempotency
class ChargePaymentWorker
  include Sidekiq::Job
  sidekiq_options retry: 5

  def perform(order_id, idempotency_key)
    # Idempotency check — was this already processed?
    return if ProcessedPayment.exists?(idempotency_key: idempotency_key)

    ActiveRecord::Base.transaction do
      order = Order.lock.find(order_id)
      return if order.paid?  # Double-check within transaction

      result = PaymentGateway.charge(
        amount: order.total,
        idempotency_key: idempotency_key  # Payment gateway also uses idempotency key
      )

      order.update!(status: :paid, transaction_id: result.transaction_id)
      ProcessedPayment.create!(idempotency_key: idempotency_key, order_id: order_id)
    end
  end
end
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

> **Ruby Context:** Clock skew affects Rails apps in subtle ways. `Rails.cache` TTLs, Sidekiq job scheduling (`perform_at`), Redis key expiration, and `ActiveRecord` timestamps all rely on system clocks. If your Rails app servers and Redis/database servers have drifting clocks, you might see cache entries expiring early, scheduled jobs firing at wrong times, or `updated_at` timestamps that don't reflect true ordering. Always use NTP on all servers and prefer relative durations (`expires_in: 1.hour`) over absolute timestamps where possible.

```ruby
# Clock skew problem with distributed locks in Ruby
# ❌ Unsafe — relies on clock synchronization between Redis and app servers
def acquire_lock_unsafe(key, ttl: 5)
  expiry = Time.now.to_f + ttl
  if Redis.current.setnx(key, expiry)
    true
  else
    stored_expiry = Redis.current.get(key).to_f
    if Time.now.to_f > stored_expiry  # Clock skew makes this unreliable!
      Redis.current.getset(key, Time.now.to_f + ttl)
      true
    else
      false
    end
  end
end

# ✅ Safer — use Redis SET with EX (server-side expiry, no clock comparison)
def acquire_lock_safe(key, ttl: 5)
  # Redis handles expiry on its own clock — no cross-machine clock comparison
  Redis.current.set(key, SecureRandom.uuid, nx: true, ex: ttl)
end
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

> **Ruby Context:** When your Rails app reads from a PostgreSQL primary, you get linearizable reads for that single database. But the moment you introduce read replicas, you lose linearizability — a write to the primary may not be visible on the replica yet. This is why after creating a record, redirecting to a page that reads from a replica might show "not found."

```ruby
# Linearizability problem with read replicas in Rails
class OrdersController < ApplicationController
  def create
    @order = Order.create!(order_params)  # Writes to PRIMARY
    redirect_to @order                     # Next request might read from REPLICA
    # → User might see "Order not found" because replica hasn't caught up!
  end

  def show
    @order = Order.find(params[:id])  # Might read from REPLICA (stale!)
  end
end

# Fix: Force read from primary after write (Rails 6+ multi-database)
class OrdersController < ApplicationController
  def show
    # ActiveRecord automatically reads from primary for a short window after writes
    # Or explicitly:
    ActiveRecord::Base.connected_to(role: :writing) do
      @order = Order.find(params[:id])  # Reads from PRIMARY
    end
  end
end
```

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

> **Ruby Context:** Rails 6+ has built-in support for read-your-writes consistency with multi-database setups. After a write, Rails automatically routes subsequent reads to the primary for a configurable delay period, then switches back to the replica.

```ruby
# Rails 6+ automatic read-your-writes with multi-database
# config/database.yml
# production:
#   primary:
#     adapter: postgresql
#     url: <%= ENV['PRIMARY_DATABASE_URL'] %>
#   primary_replica:
#     adapter: postgresql
#     url: <%= ENV['REPLICA_DATABASE_URL'] %>
#     replica: true

# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: { writing: :primary, reading: :primary_replica }
end

# Rails automatically switches to primary after writes:
# config/application.rb
config.active_record.database_selector = { delay: 2.seconds }
config.active_record.database_resolver = ActiveRecord::Middleware::DatabaseSelector::Resolver
config.active_record.database_resolver_context = ActiveRecord::Middleware::DatabaseSelector::Resolver::Session

# After a write, reads go to primary for 2 seconds, then back to replica.
# This provides read-your-writes consistency for the writing user.
```

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

> **Ruby Context:** Without sticky sessions, a Rails app behind a load balancer might route consecutive requests to different app servers connected to different replicas. A user could see their new order on one page load, then see it disappear on the next. Sticky sessions (via load balancer cookie) or the Rails database selector (which tracks the last write timestamp in the session) prevent this.

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

> **Ruby Context:** `Rails.cache` with Memcached or Redis clusters is eventually consistent — a write to one cache node may not be immediately visible on another. DNS-based service discovery is eventually consistent. Even PostgreSQL streaming replication has an "eventual" window where replicas lag behind the primary (typically < 100ms, but can spike under load).

```ruby
# Eventual consistency with Rails.cache
Rails.cache.write('user:123:profile', user_profile, expires_in: 1.hour)
# Another app server might still read the OLD cached value until
# the cache entry propagates or expires.

# Eventual consistency with PostgreSQL replicas
# Write to primary:
user.update!(name: 'New Name')  # Goes to primary

# Immediate read from replica might return old name:
ActiveRecord::Base.connected_to(role: :reading) do
  User.find(user.id).name  # Might still return 'Old Name' for a few ms
end
```

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

> **Ruby Context:** In Rails, `ActiveRecord::Base.transaction` with PostgreSQL's default `READ COMMITTED` isolation does NOT provide serializability. For serializable transactions, you must explicitly set the isolation level. This matters for operations like inventory reservation where concurrent transactions could oversell.

```ruby
# Default Rails transaction — READ COMMITTED (not serializable)
ActiveRecord::Base.transaction do
  account = Account.find(1)
  account.update!(balance: account.balance - 100)
  # Another concurrent transaction could read the old balance!
end

# Serializable isolation — prevents anomalies but may cause retries
ActiveRecord::Base.transaction(isolation: :serializable) do
  account = Account.lock.find(1)  # SELECT ... FOR UPDATE
  account.update!(balance: account.balance - 100)
rescue ActiveRecord::SerializationFailure
  retry  # PostgreSQL aborts one of the conflicting transactions
end
```

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

> **Ruby Context:** While you won't implement Raft in Ruby for production, you interact with Raft-based systems constantly: etcd (if using Kubernetes), Consul (service discovery via `diplomat` gem), and RabbitMQ quorum queues (via `bunny` gem). Understanding Raft helps you reason about why these systems require a majority of nodes to be available, why writes are slower than reads, and what happens during leader failover.

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

> **Ruby Context:** The `zk` gem (or `zookeeper` gem) provides a Ruby client for ZooKeeper. It supports creating znodes, watches, ephemeral nodes, and recipes for leader election and distributed locking.

```ruby
# ZooKeeper client in Ruby (zk gem)
require 'zk'

zk = ZK.new('localhost:2181')

# Create a persistent configuration node
zk.create('/config/database-url', 'postgres://primary:5432/mydb', or: :set)

# Watch for configuration changes
zk.register('/config/database-url') do |event|
  new_value = zk.get('/config/database-url').first
  Rails.logger.info("Database URL changed to: #{new_value}")
  DatabaseConnectionPool.reconnect!(new_value)
end

# Service registration with ephemeral node (auto-deleted on disconnect)
zk.create(
  '/services/order-service/instance-',
  "#{Socket.ip_address_list.detect(&:ipv4_private?).ip_address}:3000",
  mode: :ephemeral_sequential
)

# Cleanup on shutdown
at_exit { zk.close }
```

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

> **Ruby Context:** The `zk` gem provides a built-in `LeaderElection` recipe that handles the ephemeral sequential node creation and watch logic.

```ruby
# Leader election with ZK gem
require 'zk'

zk = ZK.new('localhost:2181')
election = zk.election_candidate('order-processor', Socket.gethostname)

election.on_winning_election do
  Rails.logger.info("This node is now the LEADER for order processing")
  OrderProcessingLeader.start  # Start processing orders
end

election.on_losing_election do
  Rails.logger.info("This node is a FOLLOWER — standing by")
  OrderProcessingLeader.stop   # Stop processing, wait for leadership
end

election.vote!  # Participate in the election
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

```ruby
# Distributed locking with ZK gem
require 'zk'

zk = ZK.new('localhost:2181')

# Acquire an exclusive lock
lock = zk.locker('/locks/inventory-update')

if lock.lock!(wait: 10)  # Wait up to 10 seconds for the lock
  begin
    # Critical section — only one process executes this at a time
    inventory = Inventory.find(product_id)
    inventory.update!(quantity: inventory.quantity - reserved_quantity)
  ensure
    lock.unlock!
  end
else
  raise 'Could not acquire inventory lock within timeout'
end
```

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

> **Ruby Context:** The `etcdv3` gem provides a Ruby client for etcd v3 API. It supports key-value operations, watches, leases, and transactions.

```ruby
# etcd client in Ruby (etcdv3 gem)
require 'etcdv3'

conn = Etcdv3.new(endpoints: 'http://localhost:2379')

# Store configuration
conn.put('/config/database-url', 'postgres://primary:5432/mydb')
conn.put('/config/feature-flags', '{"dark_mode": true}')

# Read configuration
value = conn.get('/config/database-url').kvs.first.value
# => "postgres://primary:5432/mydb"

# Watch for changes (blocking)
Thread.new do
  conn.watch('/config/', range_end: '/config0') do |events|
    events.each do |event|
      Rails.logger.info("Config changed: #{event.kv.key} = #{event.kv.value}")
      ConfigReloader.reload!(event.kv.key, event.kv.value)
    end
  end
end

# Lease-based ephemeral key (service registration)
lease = conn.lease_grant(30)  # 30-second TTL
conn.put('/services/order-service/instance-1', 'http://10.0.1.5:3000', lease: lease.ID)

# Keep-alive thread (renew lease periodically)
Thread.new do
  loop do
    sleep 10
    conn.lease_keep_alive(lease.ID)
  end
end
```

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

> **Ruby Context:** The `diplomat` gem is the standard Ruby client for Consul. It provides service discovery, KV store access, health checks, and session management. Consul is popular in the Ruby ecosystem because it provides service discovery, configuration, and distributed locking in one tool.

```ruby
# Consul client in Ruby (diplomat gem)
require 'diplomat'

# Configure Diplomat
Diplomat.configure do |config|
  config.url = ENV.fetch('CONSUL_URL', 'http://localhost:8500')
end

# Service discovery
services = Diplomat::Health.service('payment-service', passing: true)
healthy_instance = services.sample  # Random healthy instance
url = "http://#{healthy_instance.ServiceAddress}:#{healthy_instance.ServicePort}"

# Key-value store
Diplomat::Kv.put('config/database-url', 'postgres://primary:5432/mydb')
value = Diplomat::Kv.get('config/database-url')

# Distributed locking with Consul sessions
session_id = Diplomat::Session.create(
  Name: 'order-processor-lock',
  TTL: '30s',
  Behavior: 'delete'  # Delete lock key when session expires
)

if Diplomat::Lock.acquire('/locks/order-processor', session_id)
  begin
    # Critical section
    process_orders
  ensure
    Diplomat::Lock.release('/locks/order-processor', session_id)
  end
end
```

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

> **Ruby Context:** PostgreSQL supports 2PC via `PREPARE TRANSACTION` and `COMMIT PREPARED`, and ActiveRecord can interact with it. However, 2PC across multiple databases or services is rarely used in Ruby applications. The Saga pattern (covered in Module 25.4) is strongly preferred for Ruby microservices.

```ruby
# 2PC with PostgreSQL prepared transactions (rarely used in practice)
# This is for educational purposes — prefer Saga pattern in production

class TwoPhaseCommitCoordinator
  def execute(order_id, amount)
    transaction_id = "tx_#{SecureRandom.hex(8)}"

    begin
      # Phase 1: PREPARE
      order_prepared = prepare_order(transaction_id, order_id)
      payment_prepared = prepare_payment(transaction_id, order_id, amount)

      if order_prepared && payment_prepared
        # Phase 2: COMMIT
        commit_order(transaction_id)
        commit_payment(transaction_id)
        { status: :committed }
      else
        # Phase 2: ABORT
        rollback_order(transaction_id) if order_prepared
        rollback_payment(transaction_id) if payment_prepared
        { status: :aborted }
      end
    rescue => e
      # Crash recovery — must check prepared transaction state
      Rails.logger.error("2PC failed: #{e.message}")
      rollback_order(transaction_id) rescue nil
      rollback_payment(transaction_id) rescue nil
      { status: :error, message: e.message }
    end
  end

  private

  def prepare_order(tx_id, order_id)
    OrderDB.connection.execute("BEGIN")
    OrderDB.connection.execute("UPDATE orders SET status = 'processing' WHERE id = #{order_id}")
    OrderDB.connection.execute("PREPARE TRANSACTION '#{tx_id}_order'")
    true
  rescue => e
    false
  end

  def commit_order(tx_id)
    OrderDB.connection.execute("COMMIT PREPARED '#{tx_id}_order'")
  end

  def rollback_order(tx_id)
    OrderDB.connection.execute("ROLLBACK PREPARED '#{tx_id}_order'")
  end

  # Similar methods for payment...
end
```

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

> **Ruby Context:** The Saga pattern is the preferred approach for distributed transactions in Ruby microservices. See Module 25.4 for detailed Ruby implementations using AASM, Statesman, and Sidekiq workflows.

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

> **Ruby Context:** Compensating transactions in Ruby are typically implemented as Sidekiq workers with idempotency checks. Each forward action records enough state to enable its compensation.

```ruby
# Compensating transactions in Ruby
class OrderSagaCompensator
  def compensate(saga_id)
    saga = OrderSaga.find(saga_id)
    completed_steps = saga.completed_steps.reverse  # Compensate in reverse order

    completed_steps.each do |step|
      case step
      when 'payment_charged'
        RefundPaymentWorker.perform_async(saga.order_id, saga.transaction_id)
      when 'inventory_reserved'
        ReleaseInventoryWorker.perform_async(saga.order_id)
      when 'order_created'
        CancelOrderWorker.perform_async(saga.order_id)
      when 'confirmation_sent'
        SendCancellationEmailWorker.perform_async(saga.order_id)
      end
    end
  end
end

# Idempotent compensating worker
class RefundPaymentWorker
  include Sidekiq::Job
  sidekiq_options retry: 5

  def perform(order_id, transaction_id)
    # Idempotency — don't refund twice
    return if Refund.exists?(transaction_id: transaction_id)

    result = PaymentGateway.refund(transaction_id: transaction_id)
    Refund.create!(
      transaction_id: transaction_id,
      order_id: order_id,
      refund_id: result.refund_id,
      status: :completed
    )
  end
end
```

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

> **Ruby Context:** The outbox pattern is very natural in Rails because ActiveRecord transactions make it easy to write to multiple tables atomically. A Sidekiq worker or a dedicated process polls the outbox table and publishes events.

```ruby
# Outbox pattern in Rails

# Migration
class CreateOutboxEntries < ActiveRecord::Migration[7.1]
  def change
    create_table :outbox_entries do |t|
      t.string :aggregate_type, null: false
      t.string :aggregate_id, null: false
      t.string :event_type, null: false
      t.jsonb :payload, null: false, default: {}
      t.datetime :published_at
      t.timestamps
    end
    add_index :outbox_entries, :published_at, where: 'published_at IS NULL'
  end
end

# Write order + outbox entry in the same transaction
class OrderService
  def create_order(params)
    ActiveRecord::Base.transaction do
      order = Order.create!(params)

      OutboxEntry.create!(
        aggregate_type: 'Order',
        aggregate_id: order.id.to_s,
        event_type: 'order_created',
        payload: {
          order_id: order.id,
          user_id: order.user_id,
          total: order.total.to_f,
          items: order.order_items.map { |i|
            { product_id: i.product_id, quantity: i.quantity }
          }
        }
      )

      order
    end
  end
end

# Outbox relay worker — polls and publishes
class OutboxRelayWorker
  include Sidekiq::Job
  sidekiq_options queue: :outbox, retry: 3

  def perform
    OutboxEntry.where(published_at: nil).order(:created_at).limit(100).each do |entry|
      Karafka.producer.produce_sync(
        topic: "#{entry.aggregate_type.underscore}_events",
        payload: entry.payload.merge(
          event_type: entry.event_type,
          event_id: entry.id,
          timestamp: entry.created_at.iso8601
        ).to_json,
        key: entry.aggregate_id
      )

      entry.update!(published_at: Time.current)
    rescue => e
      Rails.logger.error("Failed to publish outbox entry #{entry.id}: #{e.message}")
      raise  # Sidekiq will retry
    end
  end
end

# Schedule the relay to run frequently
# config/sidekiq_scheduler.yml (with sidekiq-scheduler gem)
# outbox_relay:
#   every: '5s'
#   class: OutboxRelayWorker
#   queue: outbox
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
| **Examples** | `Time.now`, `Process.clock_gettime(Process::CLOCK_REALTIME)` | Lamport timestamps, vector clocks |

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

> **Ruby Context:** You can implement Lamport timestamps in Ruby for ordering events across distributed services. Each service maintains a counter and includes it in messages.

```ruby
# Lamport clock implementation in Ruby
class LamportClock
  def initialize
    @counter = 0
    @mutex = Mutex.new
  end

  # Increment before a local event
  def tick
    @mutex.synchronize do
      @counter += 1
      @counter
    end
  end

  # Update when receiving a message with a timestamp
  def receive(remote_timestamp)
    @mutex.synchronize do
      @counter = [@counter, remote_timestamp].max + 1
      @counter
    end
  end

  # Current timestamp (for sending messages)
  def current
    @mutex.synchronize { @counter }
  end
end

# Usage in a service
CLOCK = LamportClock.new

# When publishing an event
def publish_event(event_type, payload)
  timestamp = CLOCK.tick
  Karafka.producer.produce_async(
    topic: 'events',
    payload: payload.merge(
      lamport_timestamp: timestamp,
      event_type: event_type
    ).to_json
  )
end

# When consuming an event
class EventConsumer < Karafka::BaseConsumer
  def consume
    messages.each do |message|
      event = JSON.parse(message.payload)
      CLOCK.receive(event['lamport_timestamp'])
      process_event(event)
    end
  end
end
```

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

> **Ruby Context:** Vector clocks are useful for conflict detection in distributed Ruby systems. The implementation below can be used to track causality across microservices.

```ruby
# Vector clock implementation in Ruby
class VectorClock
  attr_reader :clock

  def initialize(node_id, nodes)
    @node_id = node_id
    @clock = nodes.each_with_object({}) { |n, h| h[n] = 0 }
    @mutex = Mutex.new
  end

  # Increment before a local event
  def tick
    @mutex.synchronize do
      @clock[@node_id] += 1
      @clock.dup
    end
  end

  # Update when receiving a message with a remote vector clock
  def receive(remote_clock)
    @mutex.synchronize do
      remote_clock.each do |node, timestamp|
        @clock[node] = [@clock[node] || 0, timestamp].max
      end
      @clock[@node_id] += 1
      @clock.dup
    end
  end

  # Compare two vector clocks
  def self.compare(v1, v2)
    all_keys = (v1.keys + v2.keys).uniq
    v1_leq_v2 = all_keys.all? { |k| (v1[k] || 0) <= (v2[k] || 0) }
    v2_leq_v1 = all_keys.all? { |k| (v2[k] || 0) <= (v1[k] || 0) }

    if v1_leq_v2 && v2_leq_v1
      :equal
    elsif v1_leq_v2
      :before  # v1 happened before v2
    elsif v2_leq_v1
      :after   # v1 happened after v2
    else
      :concurrent  # v1 and v2 are concurrent — conflict!
    end
  end
end

# Usage
clock = VectorClock.new(:order_service, [:order_service, :payment_service, :inventory_service])
timestamp = clock.tick  # => { order_service: 1, payment_service: 0, inventory_service: 0 }

# When receiving a message from payment service
remote_ts = { order_service: 1, payment_service: 2, inventory_service: 0 }
merged = clock.receive(remote_ts)
# => { order_service: 2, payment_service: 2, inventory_service: 0 }

# Conflict detection
VectorClock.compare({ a: 2, b: 3 }, { a: 1, b: 4 })  # => :concurrent (conflict!)
VectorClock.compare({ a: 1, b: 2 }, { a: 2, b: 3 })  # => :before
```

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

> **Ruby Context:** HLC is useful when you need both human-readable timestamps and causal ordering. Here's a Ruby implementation:

```ruby
# Hybrid Logical Clock implementation in Ruby
class HybridLogicalClock
  attr_reader :physical_time, :logical_counter

  def initialize
    @physical_time = 0
    @logical_counter = 0
    @mutex = Mutex.new
  end

  # Generate a new timestamp for a local event or send
  def now
    @mutex.synchronize do
      pt = current_physical_time
      if pt > @physical_time
        @physical_time = pt
        @logical_counter = 0
      else
        @logical_counter += 1
      end
      [@physical_time, @logical_counter]
    end
  end

  # Update when receiving a message with a remote HLC timestamp
  def receive(remote_pt, remote_lc)
    @mutex.synchronize do
      pt = current_physical_time
      if pt > @physical_time && pt > remote_pt
        @physical_time = pt
        @logical_counter = 0
      elsif remote_pt > @physical_time
        @physical_time = remote_pt
        @logical_counter = remote_lc + 1
      elsif @physical_time == remote_pt
        @logical_counter = [@logical_counter, remote_lc].max + 1
      else
        @logical_counter += 1
      end
      [@physical_time, @logical_counter]
    end
  end

  # Compare two HLC timestamps
  def self.compare(ts1, ts2)
    pt1, lc1 = ts1
    pt2, lc2 = ts2
    return -1 if pt1 < pt2 || (pt1 == pt2 && lc1 < lc2)
    return 1 if pt1 > pt2 || (pt1 == pt2 && lc1 > lc2)
    0
  end

  private

  def current_physical_time
    Process.clock_gettime(Process::CLOCK_REALTIME, :millisecond)
  end
end

# Usage
hlc = HybridLogicalClock.new
ts = hlc.now  # => [1714200000000, 0]  (milliseconds since epoch, counter)

# Include in messages
event = { data: '...', hlc_pt: ts[0], hlc_lc: ts[1] }

# On receive
remote_ts = hlc.receive(event[:hlc_pt], event[:hlc_lc])
```

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

> **Ruby Context:** Ruby provides several clock sources via `Process.clock_gettime`. For distributed systems work, prefer monotonic clocks for measuring durations and wall clocks only for human-readable timestamps.

```ruby
# Ruby clock sources
Time.now                                                    # Wall clock (affected by NTP adjustments)
Process.clock_gettime(Process::CLOCK_REALTIME)              # Wall clock (float seconds)
Process.clock_gettime(Process::CLOCK_REALTIME, :millisecond) # Wall clock (integer ms)
Process.clock_gettime(Process::CLOCK_MONOTONIC)             # Monotonic (for measuring durations)

# ✅ Use monotonic clock for timeouts and durations
start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
perform_operation
elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
Rails.logger.info("Operation took #{elapsed.round(3)}s")

# ❌ Don't use wall clock for durations (NTP can adjust it backward!)
start = Time.now
perform_operation
elapsed = Time.now - start  # Could be negative if NTP adjusts the clock!
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

**Ruby-Specific Takeaways:**

| Area | Ruby Tools & Gems |
|------|-------------------|
| **Distributed Locking** | redlock-rb (Redis), zk gem (ZooKeeper), diplomat (Consul) |
| **Service Discovery** | diplomat (Consul), eureka-client, Kubernetes DNS |
| **Configuration** | etcdv3 gem, diplomat (Consul KV), Rails credentials |
| **Coordination** | zk gem (ZooKeeper), etcdv3, diplomat (Consul sessions) |
| **Idempotency** | sidekiq-unique-jobs, custom idempotency keys, database constraints |
| **Outbox Pattern** | ActiveRecord transactions + Sidekiq relay, Debezium (CDC) |
| **Event Ordering** | Karafka (Kafka partition ordering), custom Lamport/vector clocks |
| **Multi-DB Consistency** | Rails multi-database (database_selector), ActiveRecord transactions |
| **Resilience** | Stoplight (circuit breaker), faraday-retry, connection_pool |
| **Observability** | OpenTelemetry, Prometheus client, Lograge |
| **Clock Sources** | `Process.clock_gettime(CLOCK_MONOTONIC)` for durations, `Time.now` for wall clock |

---

## Interview Tips for Module 26

1. **8 Fallacies** — mention at least 3-4 fallacies and explain their practical impact; give Ruby examples (N+1 across services, Faraday timeouts)
2. **Partial failures** — explain the "unknown" outcome and why idempotency is critical; reference Sidekiq job retries as a concrete example
3. **Consistency models** — draw the spectrum from linearizable to eventual; explain each level with an example; mention Rails read replicas for read-your-writes
4. **Linearizability vs Serializability** — know the difference; this is a common trick question; reference PostgreSQL isolation levels
5. **Raft** — explain leader election and log replication; draw the state diagram (follower → candidate → leader); mention etcd and Consul as Ruby-relevant systems
6. **Paxos vs Raft** — know that Raft is the practical choice; Paxos is the theoretical foundation
7. **ZooKeeper** — explain leader election with ephemeral sequential nodes; explain distributed locking; reference the `zk` gem
8. **etcd vs ZooKeeper** — etcd is simpler, Raft-based, used by Kubernetes; ZooKeeper is older, used by Kafka; reference `etcdv3` and `diplomat` gems
9. **2PC** — explain the two phases; explain the blocking problem; explain why Saga is preferred for microservices
10. **Outbox pattern** — explain the dual-write problem and how the outbox solves it; show the Rails ActiveRecord transaction approach; mention Debezium for CDC
11. **Lamport vs Vector clocks** — Lamport can't detect concurrency; vector clocks can but don't scale
12. **HLC** — mention as the practical choice; combines physical time with logical ordering; used by CockroachDB