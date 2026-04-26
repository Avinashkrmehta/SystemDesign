# Module 24: Message Queues & Event-Driven Architecture

> In any distributed system, services need to communicate. Synchronous communication (REST, gRPC) is simple but creates tight coupling — if the downstream service is slow or down, the caller is blocked. **Message queues** decouple producers from consumers, enabling asynchronous communication that improves reliability, scalability, and fault tolerance. **Event-driven architecture** takes this further — instead of services calling each other, they react to events, creating loosely coupled systems that can evolve independently. This module covers message queue fundamentals, deep dives into Kafka and RabbitMQ, event-driven patterns (Event Sourcing, CQRS, Saga), stream processing, and real-world use cases.

> **Ruby Context:** Ruby has a rich ecosystem for message queues and event-driven architecture. Key gems include **Sidekiq** (Redis-backed background jobs), **Sneakers** (RabbitMQ worker framework), **Karafka** (Kafka consumer framework for Ruby/Rails), **Bunny** (RabbitMQ client), **ruby-kafka** / **rdkafka-ruby** (Kafka clients), and **Rails ActiveJob** (unified interface for background job frameworks). Rails applications commonly use these tools to build scalable, event-driven systems.

---

## 24.1 Message Queue Fundamentals

> A **message queue** is a form of asynchronous service-to-service communication. Messages are stored in a queue until the receiving service (consumer) is ready to process them. The sending service (producer) doesn't wait for the consumer — it publishes the message and moves on. This decoupling is the foundation of scalable, resilient distributed systems.

---

### Producer, Consumer, Broker

The three core components of any messaging system:

```
+----------+     publish     +---------+     consume     +----------+
| Producer | ──────────────→ | Broker  | ──────────────→ | Consumer |
| (Sender) |                 | (Queue) |                 | (Receiver)|
+----------+                 +---------+                 +----------+

Producer: Creates and sends messages (e.g., Order Service publishes "order_created")
Broker:   Stores messages durably and delivers them to consumers (e.g., Kafka, RabbitMQ)
Consumer: Receives and processes messages (e.g., Payment Service processes the order)
```

**Detailed Flow:**

```
1. Producer creates a message:
   { "event": "order_created", "order_id": "ORD-123", "amount": 99.99 }

2. Producer sends the message to the broker (a specific queue or topic)

3. Broker:
   - Receives the message
   - Persists it to disk (durability)
   - Acknowledges receipt to the producer
   - Holds the message until a consumer is ready

4. Consumer:
   - Pulls the message from the broker (or broker pushes it)
   - Processes the message (e.g., charge the customer)
   - Acknowledges successful processing to the broker
   - Broker removes the message (or marks it as consumed)
```

**Why Not Just Call the Service Directly?**

```
Synchronous (REST/gRPC):
  Order Service → POST /payments → Payment Service
  
  Problems:
  - If Payment Service is down → Order Service gets an error → user sees failure
  - If Payment Service is slow → Order Service is blocked → thread pool exhaustion
  - If traffic spikes → Payment Service is overwhelmed → cascading failure
  - Tight coupling: Order Service must know Payment Service's API, URL, and availability

Asynchronous (Message Queue):
  Order Service → publish "order_created" → Broker → Payment Service consumes
  
  Benefits:
  - If Payment Service is down → messages queue up → processed when it recovers
  - If Payment Service is slow → messages buffer in the queue → no blocking
  - If traffic spikes → queue absorbs the burst → consumers process at their own pace
  - Loose coupling: Order Service doesn't know or care who processes the message
```

> **Ruby Context:** In Rails, the simplest way to move work off the request cycle is **ActiveJob** backed by **Sidekiq** (Redis) or **Sneakers** (RabbitMQ). For example, `OrderMailer.confirmation(order).deliver_later` enqueues a job that a Sidekiq worker picks up asynchronously. For cross-service messaging, **Bunny** (RabbitMQ) or **Karafka** (Kafka) are the go-to choices.

**Key Terminology:**

| Term | Description |
|------|-------------|
| **Message** | A unit of data sent from producer to consumer (payload + metadata) |
| **Queue** | An ordered collection of messages waiting to be processed |
| **Topic** | A named channel for messages (Kafka); consumers subscribe to topics |
| **Exchange** | A routing mechanism (RabbitMQ) that routes messages to queues based on rules |
| **Partition** | A subdivision of a topic for parallelism (Kafka) |
| **Consumer Group** | A group of consumers that share the work of consuming a topic (Kafka) |
| **Offset** | The position of a message within a partition (Kafka) |
| **Binding** | A rule that connects an exchange to a queue (RabbitMQ) |
| **Channel** | A virtual connection inside a TCP connection (RabbitMQ/AMQP) |

---

### Point-to-Point vs Publish-Subscribe

The two fundamental messaging patterns:

**Point-to-Point (Queue):**

Each message is consumed by **exactly one** consumer. Multiple consumers can listen on the same queue, but each message goes to only one of them (competing consumers).

```
                          +------------+
                     ┌──→ | Consumer A | (processes message 1, 3, 5)
+----------+        │    +------------+
| Producer | → [Queue]
+----------+        │    +------------+
                     └──→ | Consumer B | (processes message 2, 4, 6)
                          +------------+

Message 1 → Consumer A
Message 2 → Consumer B
Message 3 → Consumer A
Message 4 → Consumer B
(Each message processed by exactly ONE consumer — work is distributed)
```

**Use Cases:**
- Task distribution (process orders, send emails, resize images)
- Work queues where each task should be done exactly once
- Load balancing work across multiple workers

> **Ruby Context:** Sidekiq uses this pattern natively — multiple Sidekiq worker processes compete for jobs from the same Redis queue. Sneakers does the same with RabbitMQ queues.

**Publish-Subscribe (Pub/Sub):**

Each message is delivered to **all** subscribers. Every subscriber gets a copy of every message.

```
                          +------------+
                     ┌──→ | Subscriber A | (gets ALL messages)
+----------+        │    +------------+
| Publisher | → [Topic]
+----------+        │    +------------+
                     ├──→ | Subscriber B | (gets ALL messages)
                     │    +------------+
                     │    +------------+
                     └──→ | Subscriber C | (gets ALL messages)
                          +------------+

Message 1 → Subscriber A, Subscriber B, Subscriber C (all get it)
Message 2 → Subscriber A, Subscriber B, Subscriber C (all get it)
```

**Use Cases:**
- Event broadcasting (user signed up → notify email service, analytics, CRM)
- Real-time notifications
- Log aggregation (all services publish logs → multiple consumers: storage, alerting, analytics)

**Comparison:**

| Feature | Point-to-Point | Publish-Subscribe |
|---------|---------------|-------------------|
| **Delivery** | One consumer per message | All subscribers get every message |
| **Use case** | Task distribution, work queues | Event broadcasting, notifications |
| **Scaling** | Add consumers to process faster | Add subscribers for new functionality |
| **Example** | RabbitMQ queue, SQS | Kafka topic, SNS, RabbitMQ fanout exchange |
| **Coupling** | Producer knows the queue | Producer publishes to topic; doesn't know subscribers |

**Kafka's Hybrid Model (Consumer Groups):**

Kafka combines both patterns. A topic can have multiple consumer groups. Within a group, each message goes to one consumer (point-to-point). Across groups, each group gets all messages (pub/sub).

```
Topic: "orders"

Consumer Group A (Order Processing):
  Consumer A1 gets partition 0 messages
  Consumer A2 gets partition 1 messages
  (Each message processed by ONE consumer in the group)

Consumer Group B (Analytics):
  Consumer B1 gets partition 0 messages
  Consumer B2 gets partition 1 messages
  (Each message processed by ONE consumer in the group)

Both groups get ALL messages — but within each group, work is distributed.
```

> **Ruby Context:** Karafka supports consumer groups natively. You define consumer classes that subscribe to topics, and Karafka manages partition assignment within the group. Multiple Karafka applications (different consumer groups) can independently consume the same topic.

---

### Message Ordering

**Why Ordering Matters:**

```
Order events:
  1. order_created (order_id=123)
  2. payment_received (order_id=123)
  3. order_shipped (order_id=123)

If processed out of order:
  3. order_shipped → ERROR! Can't ship an order that hasn't been paid!
  1. order_created → OK
  2. payment_received → OK

Correct order is critical for stateful processing.
```

**Ordering Guarantees by System:**

| System | Ordering Guarantee |
|--------|-------------------|
| **Kafka** | Ordered within a partition (not across partitions) |
| **RabbitMQ** | Ordered within a single queue (with one consumer) |
| **SQS Standard** | Best-effort ordering (no guarantee) |
| **SQS FIFO** | Strict ordering within a message group |
| **Apache Pulsar** | Ordered within a partition (like Kafka) |
| **Redis Streams** | Ordered within a stream |

**Kafka Partition Ordering:**

```
Topic: "orders" (3 partitions)

Partition 0: [msg1, msg4, msg7, ...]  ← ordered within partition
Partition 1: [msg2, msg5, msg8, ...]  ← ordered within partition
Partition 2: [msg3, msg6, msg9, ...]  ← ordered within partition

Messages within a partition are strictly ordered.
Messages across partitions have NO ordering guarantee.

To ensure ordering for a specific entity (e.g., order_id=123):
  → Use order_id as the partition key
  → All messages for order_id=123 go to the SAME partition
  → They are processed in order
```

```
Producer sends:
  message(key="order-123", value="order_created")  → hash("order-123") % 3 = partition 1
  message(key="order-123", value="payment_received") → hash("order-123") % 3 = partition 1
  message(key="order-123", value="order_shipped")    → hash("order-123") % 3 = partition 1

All three messages go to partition 1 → processed in order by the consumer of partition 1.
```

**Ordering Trade-offs:**

| Approach | Ordering | Parallelism | Throughput |
|----------|----------|-------------|-----------|
| Single partition | Global order | None (1 consumer) | Low |
| Multiple partitions + key | Per-key order | High (1 consumer per partition) | High |
| Multiple partitions, no key | No order | Highest | Highest |

---

### At-Most-Once, At-Least-Once, Exactly-Once Delivery

The three delivery semantics define how many times a message is delivered to the consumer. This is one of the most important concepts in messaging systems.

**At-Most-Once:**

The message is delivered **zero or one time**. It may be lost, but it's never duplicated.

```
Producer → Broker → Consumer

1. Broker delivers message to Consumer
2. Broker immediately marks message as consumed (before consumer processes it)
3. If consumer crashes during processing → message is LOST (not redelivered)

Flow:
  Broker: "Here's message 42"
  Broker: (marks 42 as consumed immediately)
  Consumer: (starts processing... crashes!)
  Message 42 is LOST — broker already marked it as consumed
```

| Pros | Cons |
|------|------|
| Simplest, fastest | Messages can be lost |
| No duplicates | Not suitable for critical data |
| Low latency | No retry mechanism |

**Use cases:** Metrics, logs, monitoring data — where losing a few data points is acceptable.

**At-Least-Once:**

The message is delivered **one or more times**. It's never lost, but it may be duplicated.

```
Producer → Broker → Consumer

1. Broker delivers message to Consumer
2. Consumer processes the message
3. Consumer sends ACK (acknowledgment) to Broker
4. Broker marks message as consumed

If consumer crashes BEFORE sending ACK:
  Broker: "No ACK received for message 42 — redeliver!"
  Broker delivers message 42 AGAIN to a consumer
  → Message 42 is processed TWICE (duplicate!)

Flow:
  Broker: "Here's message 42"
  Consumer: (processes message 42... crashes before ACK!)
  Broker: (timeout, no ACK) → redeliver message 42
  Consumer: (processes message 42 again — DUPLICATE)
```

| Pros | Cons |
|------|------|
| No message loss | Duplicates possible |
| Reliable delivery | Consumer must handle duplicates (idempotency) |
| Most common choice | Slightly higher latency (wait for ACK) |

**Use cases:** Most production systems. Order processing, payment notifications, email sending — with idempotent consumers.

> **Ruby Context:** Sidekiq uses at-least-once delivery by default. If a worker raises an exception, the job is retried (with exponential backoff). Sneakers with RabbitMQ also uses manual ACK — if the worker crashes before acknowledging, the message is redelivered.

**Exactly-Once:**

The message is delivered **exactly one time**. No loss, no duplicates. The holy grail of messaging — but extremely hard to achieve in distributed systems.

```
True exactly-once is nearly impossible in distributed systems because:
  - Network can fail between consumer processing and ACK
  - Consumer can crash after processing but before ACK
  - Broker can crash after receiving ACK but before persisting it

What "exactly-once" usually means in practice:
  "At-least-once delivery" + "Idempotent processing" = "Effectively exactly-once"
```

**Approaches to Exactly-Once:**

| Approach | How It Works | Trade-off |
|----------|-------------|-----------|
| **Idempotent consumer** | Consumer detects and ignores duplicates (using message ID) | Consumer must track processed message IDs |
| **Transactional outbox** | Write message + business data in same DB transaction | Requires polling or CDC (Change Data Capture) |
| **Kafka transactions** | Kafka's built-in transactional producer + consumer | Higher latency, Kafka-specific |
| **Deduplication at broker** | Broker rejects duplicate messages (using producer ID + sequence number) | Broker must maintain dedup state |

**Idempotent Consumer Pattern:**

```
Consumer receives message:
  { "message_id": "msg-abc-123", "event": "charge_customer", "amount": 99.99 }

Consumer logic:
  1. Check: "Have I already processed msg-abc-123?"
     → Query processed_messages table: SELECT 1 FROM processed_messages WHERE id = 'msg-abc-123'
  2. If YES → skip (already processed, this is a duplicate)
  3. If NO →
     a. Process the message (charge the customer)
     b. INSERT INTO processed_messages (id, processed_at) VALUES ('msg-abc-123', NOW())
     c. ACK the message
     (Steps a and b should be in the SAME database transaction for atomicity)
```

> **Ruby Context:** In a Rails Sidekiq worker, idempotency can be implemented using a database uniqueness constraint or the `sidekiq-unique-jobs` gem. For Karafka consumers, you can track processed offsets or message IDs in your database within the same transaction as your business logic.

```ruby
# Ruby idempotent consumer example (Sidekiq worker)
class ChargeCustomerWorker
  include Sidekiq::Job
  sidekiq_options retry: 5

  def perform(message_id, order_id, amount)
    # Idempotency check — skip if already processed
    return if ProcessedMessage.exists?(message_id: message_id)

    ActiveRecord::Base.transaction do
      # Process the charge
      PaymentGateway.charge(order_id: order_id, amount: amount)

      # Record that we processed this message (same transaction)
      ProcessedMessage.create!(message_id: message_id, processed_at: Time.current)
    end
  end
end
```

**Delivery Semantics Comparison:**

| Semantic | Message Loss | Duplicates | Complexity | Use Case |
|----------|-------------|-----------|-----------|----------|
| **At-most-once** | Possible | Never | Lowest | Metrics, logs, non-critical data |
| **At-least-once** | Never | Possible | Medium | Most production systems (with idempotency) |
| **Exactly-once** | Never | Never | Highest | Financial transactions, critical workflows |

**Industry Standard:** Most production systems use **at-least-once delivery with idempotent consumers**. This gives you the reliability of exactly-once without the complexity and performance overhead.

---

### Dead Letter Queue (DLQ)

A **Dead Letter Queue** is a special queue where messages that cannot be processed successfully are sent. Instead of retrying forever or losing the message, it's moved to the DLQ for investigation.

```
Normal Flow:
  Queue → Consumer → Process successfully → ACK → message removed

DLQ Flow:
  Queue → Consumer → Process FAILS → retry 1 → FAILS → retry 2 → FAILS → retry 3 → FAILS
  → Message moved to Dead Letter Queue (DLQ)
  → Alert operations team
  → Investigate and fix
  → Replay messages from DLQ back to the original queue
```

```
+----------+     +-------------+     +----------+
| Producer | ──→ | Main Queue  | ──→ | Consumer |
+----------+     +------+------+     +----+-----+
                        |                  |
                        |    Failed after  |
                        |    max retries   |
                        ↓                  |
                 +------+------+           |
                 | Dead Letter |  ←────────┘
                 |   Queue     |
                 +------+------+
                        |
                        ↓
                 +------+------+
                 | Investigation|
                 | & Replay     |
                 +-------------+
```

**Why Messages End Up in DLQ:**

| Reason | Example |
|--------|---------|
| **Malformed message** | Invalid JSON, missing required fields |
| **Business logic error** | Order references a product that doesn't exist |
| **Transient failure exhausted** | Database was down for all retry attempts |
| **Poison message** | Message that always causes consumer to crash |
| **Schema mismatch** | Producer sends v2 schema, consumer expects v1 |
| **Timeout** | Processing takes longer than the visibility timeout |

**DLQ Configuration:**

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| **Max retries** | Number of processing attempts before moving to DLQ | 3-5 |
| **Retry delay** | Time between retries (often exponential backoff) | 1s, 5s, 30s, 5min |
| **DLQ retention** | How long messages stay in the DLQ | 7-14 days |
| **Alert threshold** | Alert when DLQ depth exceeds threshold | 10-100 messages |

> **Ruby Context:** Sidekiq has built-in retry with exponential backoff and a "Dead" set (its DLQ equivalent). After exhausting retries (default 25), jobs move to the Dead set visible in the Sidekiq Web UI. Sneakers supports DLQ via RabbitMQ's dead-letter exchange (`x-dead-letter-exchange`). Karafka provides a `dead_letter_queue` feature that routes failed messages to a separate Kafka topic after N retries.

```ruby
# Sidekiq DLQ configuration
class ProcessOrderWorker
  include Sidekiq::Job
  sidekiq_options retry: 5  # Max 5 retries before moving to Dead set

  # Called when all retries are exhausted (message goes to Dead set)
  sidekiq_retries_exhausted do |job, exception|
    Rails.logger.error("Order processing permanently failed: #{job['args']} - #{exception.message}")
    AlertService.notify("DLQ: Order processing failed after all retries", job: job)
  end

  def perform(order_id)
    Order.find(order_id).process!
  end
end
```

**DLQ Best Practices:**
- Always set up a DLQ — never let messages retry infinitely
- Monitor DLQ depth — a growing DLQ indicates a systemic problem
- Include original error information with the DLQ message (why it failed)
- Build tooling to inspect and replay DLQ messages
- Set up alerts when messages land in the DLQ

---

### Message Acknowledgment

**Acknowledgment (ACK)** is how the consumer tells the broker that a message has been successfully processed. The broker uses this to decide whether to remove the message or redeliver it.

```
ACK Flow:
  1. Broker delivers message to Consumer
  2. Consumer processes the message
  3. Consumer sends ACK to Broker
  4. Broker removes the message from the queue

NACK (Negative Acknowledgment) Flow:
  1. Broker delivers message to Consumer
  2. Consumer tries to process but FAILS
  3. Consumer sends NACK to Broker
  4. Broker requeues the message (or sends to DLQ after max retries)
```

**ACK Modes:**

| Mode | Description | Risk |
|------|-------------|------|
| **Auto ACK** | Broker marks message as consumed immediately on delivery | Message loss if consumer crashes before processing |
| **Manual ACK** | Consumer explicitly ACKs after successful processing | Duplicates if consumer crashes after processing but before ACK |
| **Batch ACK** | Consumer ACKs multiple messages at once (Kafka: commit offset) | All unACKed messages redelivered on crash |

> **Ruby Context:** Bunny (RabbitMQ) supports both auto and manual ACK. Sneakers defaults to manual ACK — your worker returns `:ack`, `:reject`, or `:requeue`. Karafka manages Kafka offset commits automatically but also supports manual offset management for fine-grained control.

```ruby
# Sneakers worker with manual ACK (RabbitMQ)
class PaymentWorker
  include Sneakers::Worker
  from_queue 'payments', ack: true  # manual ACK mode

  def work(raw_message)
    message = JSON.parse(raw_message)
    PaymentService.process(message)
    ack!    # Acknowledge — message removed from queue
  rescue PaymentError => e
    reject! # Negative ACK — message requeued or sent to DLQ
  end
end
```

**Kafka Offset Commit (ACK equivalent):**

```
Kafka doesn't use traditional ACK. Instead, consumers commit their offset — the position
of the last successfully processed message.

Partition: [msg0, msg1, msg2, msg3, msg4, msg5, msg6, ...]
                                     ^
                              committed offset = 3
                              (messages 0-3 are processed)
                              (messages 4+ are pending)

If consumer crashes and restarts:
  → Reads committed offset (3)
  → Starts consuming from offset 4
  → Messages 0-3 are NOT redelivered (already committed)

If consumer crashes BEFORE committing offset 4:
  → Restarts from offset 4 (last committed + 1)
  → Message 4 is redelivered (duplicate — consumer must be idempotent)
```

**Visibility Timeout (SQS):**

```
SQS uses a "visibility timeout" instead of explicit ACK:

1. Consumer receives message → message becomes INVISIBLE to other consumers
2. Visibility timeout starts (e.g., 30 seconds)
3. If consumer processes and deletes the message within 30s → done
4. If consumer crashes or doesn't delete within 30s:
   → Message becomes VISIBLE again → another consumer picks it up

This prevents two consumers from processing the same message simultaneously.
```

---


## 24.2 Popular Message Queues

> Choosing the right message queue is a critical architectural decision. Each system has different strengths — Kafka excels at high-throughput log-based streaming, RabbitMQ at flexible routing and traditional messaging, SQS at managed simplicity, and Pulsar at multi-tenancy and tiered storage. This section provides deep dives into the most popular options.

> **Ruby Context:** The most common pairings in the Ruby ecosystem are: **Sidekiq + Redis** for background jobs within a single application, **Bunny/Sneakers + RabbitMQ** for cross-service messaging with flexible routing, and **Karafka/rdkafka-ruby + Kafka** for high-throughput event streaming. Rails ActiveJob provides a unified API that can target any of these backends.

---

### Apache Kafka

Kafka is a **distributed, log-based streaming platform** designed for high-throughput, fault-tolerant, real-time data streaming. Originally built at LinkedIn, it's now the de facto standard for event streaming.

**Core Architecture:**

```
+-------------------+     +-------------------+     +-------------------+
|   Kafka Broker 1  |     |   Kafka Broker 2  |     |   Kafka Broker 3  |
|                   |     |                   |     |                   |
| Topic: orders     |     | Topic: orders     |     | Topic: orders     |
| ┌───────────────┐ |     | ┌───────────────┐ |     | ┌───────────────┐ |
| │ Partition 0   │ |     | │ Partition 1   │ |     | │ Partition 2   │ |
| │ (Leader)      │ |     | │ (Leader)      │ |     | │ (Leader)      │ |
| └───────────────┘ |     | └───────────────┘ |     | └───────────────┘ |
| ┌───────────────┐ |     | ┌───────────────┐ |     | ┌───────────────┐ |
| │ Partition 1   │ |     | │ Partition 2   │ |     | │ Partition 0   │ |
| │ (Replica)     │ |     | │ (Replica)     │ |     | │ (Replica)     │ |
| └───────────────┘ |     | └───────────────┘ |     | └───────────────┘ |
+-------------------+     +-------------------+     +-------------------+
         ↑                         ↑                         ↑
         |                         |                         |
    ZooKeeper / KRaft (metadata, leader election, cluster coordination)
```

---

#### Topics, Partitions, Consumer Groups

**Topics:**

A topic is a **named stream of messages** — a logical category or feed name. Producers publish to topics, consumers subscribe to topics.

```
Topic: "orders"        → all order-related events
Topic: "payments"      → all payment-related events
Topic: "user-activity" → all user activity events
Topic: "notifications" → all notification events
```

**Partitions:**

A topic is divided into **partitions** — ordered, immutable sequences of messages. Partitions are the unit of parallelism in Kafka.

```
Topic: "orders" (3 partitions)

Partition 0: [offset 0] [offset 1] [offset 2] [offset 3] → ...
Partition 1: [offset 0] [offset 1] [offset 2] → ...
Partition 2: [offset 0] [offset 1] [offset 2] [offset 3] [offset 4] → ...

Each partition:
  - Is an ordered, append-only log
  - Lives on one broker (leader) with replicas on other brokers
  - Can be consumed by ONE consumer within a consumer group
  - Has its own offset counter (independent of other partitions)
```

**How Messages Are Assigned to Partitions:**

```
1. With a key:
   partition = hash(key) % num_partitions
   
   message(key="order-123") → hash("order-123") % 3 = 1 → Partition 1
   message(key="order-456") → hash("order-456") % 3 = 0 → Partition 0
   message(key="order-123") → hash("order-123") % 3 = 1 → Partition 1 (same partition!)
   
   → All messages with the same key go to the same partition → ordering guaranteed per key

2. Without a key (null key):
   → Round-robin across partitions (or sticky partitioning in newer Kafka)
   → No ordering guarantee
```

**Consumer Groups:**

A consumer group is a set of consumers that **cooperatively consume** a topic. Each partition is assigned to exactly one consumer in the group.

```
Topic: "orders" (4 partitions: P0, P1, P2, P3)

Consumer Group A (3 consumers):
  Consumer A1 → P0, P1
  Consumer A2 → P2
  Consumer A3 → P3
  (Each partition assigned to exactly ONE consumer in the group)
  (Consumer A1 gets 2 partitions because there are more partitions than consumers)

Consumer Group B (2 consumers):
  Consumer B1 → P0, P1
  Consumer B2 → P2, P3
  (Independent of Group A — both groups get ALL messages)
```

**Scaling Consumers:**

```
Partitions: 4 (P0, P1, P2, P3)

1 consumer:  C1 → P0, P1, P2, P3  (one consumer handles everything)
2 consumers: C1 → P0, P1 | C2 → P2, P3
3 consumers: C1 → P0, P1 | C2 → P2 | C3 → P3
4 consumers: C1 → P0 | C2 → P1 | C3 → P2 | C4 → P3  (optimal: 1:1)
5 consumers: C1 → P0 | C2 → P1 | C3 → P2 | C4 → P3 | C5 → IDLE!

Rule: Max useful consumers = number of partitions
      Adding more consumers than partitions → idle consumers (wasted resources)
```

> **Ruby Context:** Karafka manages consumer groups and partition assignment automatically. You configure the consumer group name in `karafka.rb`, and Karafka handles rebalancing when consumers join or leave. Each Karafka process can run multiple consumer threads to maximize throughput.

```ruby
# Karafka consumer group configuration (karafka.rb)
class KarafkaApp < Karafka::App
  setup do |config|
    config.kafka = { 'bootstrap.servers': 'localhost:9092' }
    config.client_id = 'order-processing-app'
  end

  routes.draw do
    consumer_group :order_processors do
      topic :orders do
        consumer OrderConsumer
      end
    end
  end
end

# Karafka consumer
class OrderConsumer < Karafka::BaseConsumer
  def consume
    messages.each do |message|
      order_data = message.payload
      OrderService.process(order_data)
    end
  end
end
```

---

#### Log-Based Storage

Kafka stores messages as an **append-only, immutable log** on disk. This is fundamentally different from traditional message queues that delete messages after consumption.

```
Traditional Queue (RabbitMQ):
  [msg1] [msg2] [msg3] [msg4]
  Consumer reads msg1 → msg1 is DELETED from the queue
  msg1 is gone forever

Kafka Log:
  [msg1] [msg2] [msg3] [msg4] [msg5] → (append only)
  Consumer A reads msg1 → msg1 is NOT deleted
  Consumer B reads msg1 → msg1 is still there
  msg1 stays until retention period expires (e.g., 7 days)
```

```
Partition 0 (append-only log on disk):

  Offset: 0     1     2     3     4     5     6     7
         [msg] [msg] [msg] [msg] [msg] [msg] [msg] [msg] → new messages appended here
                       ↑                       ↑
                Consumer Group A            Consumer Group B
                (offset = 2)                (offset = 5)
                
  Each consumer group tracks its own offset independently.
  Messages are NOT deleted when consumed — they're retained for the configured period.
```

**Why Log-Based Storage is Powerful:**

| Feature | Benefit |
|---------|---------|
| **Replay** | Consumers can re-read old messages by resetting their offset |
| **Multiple consumers** | Multiple consumer groups can read the same data independently |
| **Audit trail** | Complete history of all events (useful for debugging, compliance) |
| **Reprocessing** | If a consumer has a bug, fix it and replay all messages |
| **Time travel** | Read messages from any point in time (within retention) |
| **High throughput** | Sequential disk writes are fast (append-only, no random I/O) |

**Retention Policies:**

| Policy | Description | Typical Value |
|--------|-------------|---------------|
| **Time-based** | Delete messages older than X | 7 days (default), 30 days, forever |
| **Size-based** | Delete oldest messages when partition exceeds X bytes | 1 GB, 10 GB |
| **Compacted** | Keep only the latest message per key (like a changelog) | Used for state/config topics |

**Log Compaction:**

```
Before compaction:
  Offset 0: key=user-1, value={name: "Alice"}
  Offset 1: key=user-2, value={name: "Bob"}
  Offset 2: key=user-1, value={name: "Alice Smith"}  ← newer value for user-1
  Offset 3: key=user-3, value={name: "Charlie"}
  Offset 4: key=user-2, value={name: "Bob Jones"}    ← newer value for user-2

After compaction:
  Offset 2: key=user-1, value={name: "Alice Smith"}  ← latest for user-1
  Offset 3: key=user-3, value={name: "Charlie"}      ← only value for user-3
  Offset 4: key=user-2, value={name: "Bob Jones"}    ← latest for user-2

Old values for user-1 and user-2 are removed.
The topic now represents the CURRENT STATE of all users.
```

---

#### Offset Management

The **offset** is a sequential ID assigned to each message within a partition. Consumers track their offset to know which messages they've processed.

```
Partition 0:
  Offset: 0  1  2  3  4  5  6  7  8  9
         [m] [m] [m] [m] [m] [m] [m] [m] [m] [m]
                            ↑
                     committed offset = 4
                     (messages 0-4 processed)
                     (next message to process: offset 5)
```

**Offset Commit Strategies:**

| Strategy | Description | Risk |
|----------|-------------|------|
| **Auto-commit** | Kafka periodically commits offsets (default: every 5 seconds) | Messages may be reprocessed if consumer crashes between auto-commits |
| **Manual commit (sync)** | Consumer explicitly commits after processing each message/batch | Slower but safer; duplicates only if crash between process and commit |
| **Manual commit (async)** | Consumer commits asynchronously (non-blocking) | Faster but may lose commits on crash |

**Offset Reset Policies (when no committed offset exists):**

| Policy | Behavior |
|--------|----------|
| `earliest` | Start from the beginning of the partition (read all historical messages) |
| `latest` | Start from the end (only read new messages from now on) |
| `none` | Throw an error if no committed offset exists |

> **Ruby Context:** Karafka handles offset management automatically by default (auto-commit). You can switch to manual offset management for critical consumers by calling `mark_as_consumed(message)` explicitly within your consumer, giving you fine-grained control over when offsets are committed.

---

#### Kafka Streams

**Kafka Streams** is a client library for building stream processing applications that read from and write to Kafka topics. It's not a separate cluster — it runs inside your application.

```
Input Topic → Kafka Streams Application → Output Topic

Example: Real-time order analytics
  Input: "orders" topic (raw order events)
  Processing: Count orders per product per 5-minute window
  Output: "order-counts" topic (aggregated counts)
```

**Key Concepts:**

| Concept | Description |
|---------|-------------|
| **KStream** | An unbounded stream of records (each record is independent) |
| **KTable** | A changelog stream (latest value per key — like a database table) |
| **GlobalKTable** | A KTable replicated to all instances (for joins with small datasets) |
| **Windowing** | Group records by time windows (tumbling, hopping, sliding, session) |
| **State Store** | Local storage for aggregations and joins (backed by a changelog topic) |

```
Topology Example:

  orders-topic
       |
       ↓
  [Filter: amount > 100]
       |
       ↓
  [GroupByKey: product_id]
       |
       ↓
  [WindowedBy: 5-minute tumbling window]
       |
       ↓
  [Count]
       |
       ↓
  high-value-order-counts-topic
```

> **Ruby Context:** Kafka Streams is a JVM library and not directly available in Ruby. However, **Karafka** provides similar stream-processing capabilities for Ruby applications — you can filter, transform, and route messages within Karafka consumers. For complex stream processing (windowing, joins, aggregations), Ruby teams typically use Kafka Streams (JVM), Apache Flink, or ksqlDB alongside their Ruby services.

---

### RabbitMQ

RabbitMQ is a **traditional message broker** implementing the **AMQP (Advanced Message Queuing Protocol)**. It excels at flexible message routing, priority queues, and complex routing patterns.

**Core Architecture:**

```
+----------+     +-------------------------------------------+     +----------+
| Producer | ──→ |              RabbitMQ Broker               | ──→ | Consumer |
+----------+     |                                           |     +----------+
                 |  +----------+     +---------+             |
                 |  | Exchange | ──→ | Binding | ──→ [Queue] |
                 |  +----------+     +---------+             |
                 |                                           |
                 +-------------------------------------------+
```

> **Ruby Context:** The **Bunny** gem is the most popular RabbitMQ client for Ruby. **Sneakers** builds on Bunny to provide a high-level worker framework (similar to Sidekiq but for RabbitMQ). For Rails, you can use ActiveJob with the `activejob-sneakers` adapter.

---

#### Exchanges (Direct, Fanout, Topic, Headers)

In RabbitMQ, producers **never send messages directly to queues**. They send to an **exchange**, which routes messages to queues based on **bindings** and **routing keys**.

**Direct Exchange:**

Routes messages to queues whose **binding key exactly matches** the message's routing key.

```
Producer sends: routing_key = "payment.success"

Direct Exchange:
  Binding: "payment.success" → Queue A  ✅ (exact match)
  Binding: "payment.failed"  → Queue B  ❌ (no match)
  Binding: "order.created"   → Queue C  ❌ (no match)

→ Message goes to Queue A only
```

**Fanout Exchange:**

Routes messages to **all bound queues**, ignoring the routing key. Pure broadcast.

```
Producer sends: (any routing key — ignored)

Fanout Exchange:
  → Queue A  ✅
  → Queue B  ✅
  → Queue C  ✅

→ Message goes to ALL queues (broadcast)
```

**Topic Exchange:**

Routes messages based on **pattern matching** between the routing key and binding patterns. Uses `*` (one word) and `#` (zero or more words) wildcards.

```
Producer sends: routing_key = "order.us.created"

Topic Exchange:
  Binding: "order.*.created"  → Queue A  ✅ (matches: * = "us")
  Binding: "order.#"          → Queue B  ✅ (matches: # = "us.created")
  Binding: "order.eu.created" → Queue C  ❌ (no match: "eu" ≠ "us")
  Binding: "payment.#"        → Queue D  ❌ (no match: "order" ≠ "payment")

→ Message goes to Queue A and Queue B
```

**Headers Exchange:**

Routes based on **message headers** (key-value pairs) instead of routing key. Supports `x-match: all` (all headers must match) or `x-match: any` (any header matches).

```
Producer sends: headers = { "region": "us", "type": "premium" }

Headers Exchange:
  Binding: { "region": "us", x-match: "any" }           → Queue A  ✅
  Binding: { "region": "eu", "type": "premium", x-match: "all" } → Queue B  ❌ (region mismatch)
  Binding: { "type": "premium", x-match: "any" }        → Queue C  ✅
```

**Exchange Comparison:**

| Exchange Type | Routing Based On | Use Case |
|--------------|-----------------|----------|
| **Direct** | Exact routing key match | Task routing by type (email, SMS, push) |
| **Fanout** | Broadcast to all queues | Event broadcasting, notifications |
| **Topic** | Pattern matching on routing key | Hierarchical routing (region.service.event) |
| **Headers** | Message header values | Complex routing based on message metadata |

```ruby
# Ruby example: Publishing to different RabbitMQ exchanges with Bunny
require 'bunny'

connection = Bunny.new
connection.start
channel = connection.create_channel

# Direct exchange — route by exact key
direct_exchange = channel.direct('notifications')
direct_exchange.publish('Payment received!', routing_key: 'payment.success')

# Fanout exchange — broadcast to all bound queues
fanout_exchange = channel.fanout('events.broadcast')
fanout_exchange.publish('User signed up!')

# Topic exchange — route by pattern
topic_exchange = channel.topic('orders.routing')
topic_exchange.publish('New US order', routing_key: 'order.us.created')

connection.close
```

---

#### Queues, Bindings

**Queues** store messages until consumers process them. **Bindings** connect exchanges to queues with routing rules.

```
Exchange ──[Binding: routing_key="order.*"]──→ Queue "order-processing"
Exchange ──[Binding: routing_key="payment.*"]──→ Queue "payment-processing"
```

**Queue Properties:**

| Property | Description |
|----------|-------------|
| **Durable** | Queue survives broker restart (persisted to disk) |
| **Exclusive** | Queue is used by only one connection and deleted when connection closes |
| **Auto-delete** | Queue is deleted when the last consumer unsubscribes |
| **TTL** | Messages expire after a specified time |
| **Max length** | Maximum number of messages in the queue |
| **Dead letter exchange** | Exchange to route messages that are rejected or expire |
| **Priority** | Queue supports message priorities (0-255) |

```ruby
# Ruby example: Declaring queues and bindings with Bunny
require 'bunny'

connection = Bunny.new
connection.start
channel = connection.create_channel

# Declare a durable queue with DLQ support
queue = channel.queue('order-processing',
  durable: true,
  arguments: {
    'x-dead-letter-exchange' => 'dlx.orders',
    'x-message-ttl' => 60_000,       # 60 second TTL
    'x-max-length' => 100_000        # Max 100K messages
  }
)

# Bind queue to a topic exchange
exchange = channel.topic('orders', durable: true)
queue.bind(exchange, routing_key: 'order.*.created')

# Consume messages
queue.subscribe(manual_ack: true) do |delivery_info, _properties, payload|
  puts "Received: #{payload}"
  channel.ack(delivery_info.delivery_tag)
end
```

---

#### AMQP Protocol

**AMQP (Advanced Message Queuing Protocol)** is the wire-level protocol that RabbitMQ implements. It defines how clients and brokers communicate.

```
AMQP Connection Model:

  +------------------+          +------------------+
  |    Application   |          |   RabbitMQ       |
  |                  |          |                  |
  | +-- Connection (TCP) ------+ Connection       |
  | |                |          |                  |
  | | +-- Channel 1 ----------+ Channel 1        |
  | | |              |          |                  |
  | | +-- Channel 2 ----------+ Channel 2        |
  | | |              |          |                  |
  | | +-- Channel 3 ----------+ Channel 3        |
  | |                |          |                  |
  +------------------+          +------------------+

  Connection: A TCP connection between client and broker
  Channel: A virtual connection inside a TCP connection (lightweight)
           Multiple channels share one TCP connection (multiplexing)
           Each channel is independent (own flow control, own error handling)
```

**Why Channels?**
- Opening a TCP connection is expensive (TLS handshake, authentication)
- Channels are lightweight — you can have hundreds per connection
- Each thread in your application uses its own channel
- If a channel encounters an error, only that channel is closed (not the entire connection)

> **Ruby Context:** Bunny manages connections and channels for you. A common pattern is to create one connection per process and one channel per thread. Sneakers handles this automatically — each worker thread gets its own channel on a shared connection.

---

### Amazon SQS / SNS

**Amazon SQS (Simple Queue Service)** is a fully managed message queue service. **Amazon SNS (Simple Notification Service)** is a fully managed pub/sub service. They're often used together.

```
SQS (Point-to-Point):
  Producer → SQS Queue → Consumer
  (Each message consumed by one consumer)

SNS (Pub/Sub):
  Publisher → SNS Topic → Subscriber 1 (SQS Queue)
                        → Subscriber 2 (Lambda Function)
                        → Subscriber 3 (HTTP Endpoint)
                        → Subscriber 4 (Email)

SNS + SQS (Fan-out Pattern):
  Publisher → SNS Topic → SQS Queue A → Consumer Group A
                        → SQS Queue B → Consumer Group B
                        → SQS Queue C → Consumer Group C
  (Each consumer group gets ALL messages, processes independently)
```

**SQS Standard vs FIFO:**

| Feature | SQS Standard | SQS FIFO |
|---------|-------------|----------|
| **Throughput** | Nearly unlimited | 300 msg/s (3,000 with batching) |
| **Ordering** | Best-effort (no guarantee) | Strict FIFO within message group |
| **Delivery** | At-least-once (may duplicate) | Exactly-once (deduplication) |
| **Use case** | High-throughput, order doesn't matter | Order matters, no duplicates |
| **Cost** | Lower | Higher |

> **Ruby Context:** The **aws-sdk-sqs** and **aws-sdk-sns** gems provide Ruby clients for SQS and SNS. **Shoryuken** is a popular Sidekiq-like worker gem specifically designed for SQS. Rails ActiveJob also has an SQS adapter via `active_elastic_job` or `aws-sdk-rails`.

```ruby
# Ruby example: SQS producer and consumer with aws-sdk
require 'aws-sdk-sqs'

sqs = Aws::SQS::Client.new(region: 'us-east-1')
queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789/orders'

# Send a message
sqs.send_message(
  queue_url: queue_url,
  message_body: { order_id: 'ORD-123', amount: 99.99 }.to_json
)

# Receive and process messages
loop do
  response = sqs.receive_message(queue_url: queue_url, max_number_of_messages: 10)
  response.messages.each do |msg|
    order = JSON.parse(msg.body)
    process_order(order)
    sqs.delete_message(queue_url: queue_url, receipt_handle: msg.receipt_handle)
  end
end
```

---

### Apache Pulsar

Apache Pulsar is a **cloud-native, multi-tenant messaging and streaming platform** that combines the best of Kafka (log-based streaming) and RabbitMQ (flexible queuing).

**Key Differentiators from Kafka:**

| Feature | Kafka | Pulsar |
|---------|-------|--------|
| **Storage** | Brokers store data (coupled) | Separate storage layer (Apache BookKeeper) |
| **Multi-tenancy** | Limited (topic-level ACLs) | Native (tenants, namespaces, topics) |
| **Geo-replication** | Requires MirrorMaker (complex) | Built-in geo-replication |
| **Subscription modes** | Consumer groups only | Exclusive, Shared, Failover, Key_Shared |
| **Message acknowledgment** | Offset-based (per partition) | Per-message ACK (more flexible) |
| **Tiered storage** | Requires custom setup | Built-in (offload old data to S3/GCS) |
| **Scaling** | Add brokers + rebalance partitions | Add brokers independently of storage |

```
Pulsar Architecture:

  +----------+     +-------------------+     +-------------------+
  | Producer | ──→ | Pulsar Broker 1   | ──→ | BookKeeper        |
  +----------+     | (stateless,       |     | (persistent       |
                   |  serves topics)   |     |  storage layer)   |
  +----------+     +-------------------+     +-------------------+
  | Consumer | ←── | Pulsar Broker 2   |
  +----------+     | (stateless)       |
                   +-------------------+

  Brokers are STATELESS — they don't store data.
  BookKeeper stores data — can scale independently.
  This separation allows independent scaling of compute and storage.
```

---

### Redis Streams

**Redis Streams** is a log-based data structure in Redis that provides messaging capabilities similar to Kafka but within Redis.

```
Redis Stream: "orders"

  Entry ID (timestamp-sequence):
  1681234567890-0: { "order_id": "123", "amount": "99.99" }
  1681234567891-0: { "order_id": "124", "amount": "49.99" }
  1681234567892-0: { "order_id": "125", "amount": "149.99" }

  Consumer Group "processors":
    Consumer A: reads entry 1681234567890-0
    Consumer B: reads entry 1681234567891-0
    Consumer A: reads entry 1681234567892-0
```

> **Ruby Context:** The **redis** gem supports Redis Streams natively (`XADD`, `XREAD`, `XREADGROUP`, `XACK`). Since many Ruby/Rails apps already use Redis (for caching, Sidekiq), Redis Streams can be a lightweight messaging solution without adding another infrastructure component.

```ruby
# Ruby example: Redis Streams producer and consumer
require 'redis'

redis = Redis.new

# Producer: add entries to a stream
redis.xadd('orders', { order_id: 'ORD-123', amount: '99.99' })
redis.xadd('orders', { order_id: 'ORD-124', amount: '49.99' })

# Create a consumer group
redis.xgroup(:create, 'orders', 'processors', '0', mkstream: true) rescue nil

# Consumer: read from the group
entries = redis.xreadgroup('processors', 'consumer-1', 'orders', '>')
entries.each do |stream, messages|
  messages.each do |id, fields|
    puts "Processing order: #{fields}"
    redis.xack('orders', 'processors', id)  # Acknowledge
  end
end
```

**Redis Streams vs Kafka:**

| Feature | Redis Streams | Kafka |
|---------|--------------|-------|
| **Deployment** | Part of Redis (simple) | Separate cluster (complex) |
| **Throughput** | Moderate (100K msg/s) | Very high (millions msg/s) |
| **Persistence** | Redis persistence (RDB/AOF) | Disk-based log (very durable) |
| **Consumer groups** | Yes | Yes |
| **Retention** | Memory-limited (or maxlen) | Disk-based (days/weeks/forever) |
| **Best for** | Small-medium scale, already using Redis | Large-scale event streaming |

---

### Message Queue Comparison Summary

| Feature | Kafka | RabbitMQ | SQS | Pulsar | Redis Streams |
|---------|-------|----------|-----|--------|--------------|
| **Model** | Log-based streaming | Traditional message broker | Managed queue | Log-based + queuing | Log-based (in Redis) |
| **Throughput** | Very high (millions/s) | Moderate (50K/s) | High (managed) | Very high | Moderate (100K/s) |
| **Ordering** | Per partition | Per queue | FIFO queues only | Per partition | Per stream |
| **Delivery** | At-least-once (exactly-once with transactions) | At-least-once, at-most-once | At-least-once (SQS), exactly-once (FIFO) | At-least-once, effectively exactly-once | At-least-once |
| **Retention** | Configurable (days/forever) | Until consumed | 4 days (default, max 14) | Tiered (hot + cold storage) | Memory-limited |
| **Replay** | Yes (reset offset) | No (message deleted after ACK) | No | Yes | Yes (by ID) |
| **Routing** | Topic + partition key | Exchanges (direct, fanout, topic, headers) | Queue-based | Topic + subscription modes | Stream-based |
| **Managed option** | Confluent Cloud, AWS MSK | CloudAMQP, AWS MQ | AWS SQS (fully managed) | StreamNative Cloud | AWS ElastiCache, Redis Cloud |
| **Best for** | Event streaming, log aggregation, high throughput | Complex routing, task queues, RPC | Simple queuing, serverless | Multi-tenant, geo-replicated | Simple messaging, already using Redis |

> **Ruby Context — Choosing for Ruby/Rails:** For most Rails apps, **Sidekiq + Redis** is the default starting point for background jobs. If you need cross-service messaging with flexible routing, **Bunny/Sneakers + RabbitMQ** is the natural choice. For high-throughput event streaming, **Karafka + Kafka** is the Ruby-native option. For AWS-native apps, **Shoryuken + SQS** integrates well. Redis Streams is a good middle ground if you already run Redis and need lightweight pub/sub.

---

## 24.3 Event-Driven Architecture

> In traditional request-driven architecture, services call each other directly (Service A calls Service B). In **event-driven architecture (EDA)**, services communicate by producing and consuming **events**. An event represents something that happened — "order was placed," "payment was received," "user signed up." Services react to events they care about, creating a loosely coupled, highly scalable system. EDA is the foundation of modern microservices architectures.

> **Ruby Context:** Ruby gems like **Wisper** (in-process pub/sub), **Rails Event Store** (event sourcing and CQRS for Rails), and **Dry::Events** (from the dry-rb ecosystem) provide event-driven patterns within Ruby applications. For cross-service events, Kafka (via Karafka) or RabbitMQ (via Bunny/Sneakers) are the transport layers.

---

### Events vs Commands vs Queries

Understanding the difference between events, commands, and queries is fundamental to designing event-driven systems.

**Event:**

An event is a **notification that something happened** in the past. It's a fact — immutable and irrevocable. The producer doesn't know or care who consumes it.

```
Event: "OrderPlaced"
{
  "event_type": "OrderPlaced",
  "event_id": "evt-abc-123",
  "timestamp": "2026-04-23T10:30:00Z",
  "data": {
    "order_id": "ORD-789",
    "customer_id": "CUST-456",
    "total": 149.99,
    "items": [...]
  }
}

Characteristics:
  - Past tense: "OrderPlaced" (not "PlaceOrder")
  - Immutable: once published, it cannot be changed
  - No expectation of response: producer doesn't wait for consumers
  - Multiple consumers: any service can subscribe and react
```

**Command:**

A command is a **request to do something**. It's directed at a specific service and expects a result.

```
Command: "PlaceOrder"
{
  "command_type": "PlaceOrder",
  "data": {
    "customer_id": "CUST-456",
    "items": [...],
    "shipping_address": "..."
  }
}

Characteristics:
  - Imperative: "PlaceOrder" (not "OrderPlaced")
  - Directed: sent to a specific service (Order Service)
  - Expects result: success or failure
  - One handler: exactly one service processes the command
```

**Query:**

A query is a **request for information**. It doesn't change state — it's a read operation.

```
Query: "GetOrderStatus"
{
  "query_type": "GetOrderStatus",
  "data": {
    "order_id": "ORD-789"
  }
}

Characteristics:
  - Read-only: doesn't change any state
  - Synchronous: caller waits for the response
  - Idempotent: calling it multiple times has no side effects
```

**Comparison:**

| Aspect | Event | Command | Query |
|--------|-------|---------|-------|
| **Intent** | "This happened" | "Do this" | "Tell me this" |
| **Tense** | Past (OrderPlaced) | Imperative (PlaceOrder) | Question (GetOrder) |
| **Direction** | Broadcast (any subscriber) | Targeted (specific handler) | Targeted (specific handler) |
| **Response** | None expected | Success/failure expected | Data expected |
| **Coupling** | Loose (producer doesn't know consumers) | Tight (sender knows receiver) | Tight (sender knows receiver) |
| **Mutability** | Immutable fact | May be rejected | No state change |

> **Ruby Context:** The **Wisper** gem models this distinction well. Events are broadcast via `publish`, commands are handled by specific command objects (often using the Command pattern from dry-rb or Trailblazer), and queries are plain read operations. **Rails Event Store** provides a full event bus with publish/subscribe semantics.

```ruby
# Ruby example: Events, Commands, and Queries with Wisper and Rails Event Store

# --- Event (broadcast, past tense) ---
class OrderService
  include Wisper::Publisher

  def place_order(order_params)
    order = Order.create!(order_params)
    # Broadcast event — any subscriber can react
    broadcast(:order_placed, order_id: order.id, total: order.total)
  end
end

# Subscribers react independently
class EmailNotifier
  def on_order_placed(order_id:, total:)
    OrderMailer.confirmation(order_id).deliver_later
  end
end

class AnalyticsTracker
  def on_order_placed(order_id:, total:)
    Analytics.track('order_placed', order_id: order_id, revenue: total)
  end
end

# Wire up subscribers
order_service = OrderService.new
order_service.subscribe(EmailNotifier.new)
order_service.subscribe(AnalyticsTracker.new)
order_service.place_order(customer_id: 'CUST-456', items: ['Widget'])

# --- Command (targeted, imperative) ---
# Using dry-rb command pattern
class PlaceOrder
  include Dry::Transaction

  step :validate
  step :create_order
  step :publish_event

  private

  def validate(input)
    if input[:items].any?
      Success(input)
    else
      Failure('Cart is empty')
    end
  end

  def create_order(input)
    order = Order.create!(input)
    Success(order)
  end

  def publish_event(order)
    EventStore.publish(OrderPlaced.new(data: { order_id: order.id }))
    Success(order)
  end
end

# --- Query (read-only, synchronous) ---
class GetOrderStatus
  def call(order_id)
    order = Order.find(order_id)
    { order_id: order.id, status: order.status, total: order.total }
  end
end
```

---

### Event Sourcing

**Event Sourcing** is a pattern where the state of an entity is determined by a **sequence of events** rather than storing the current state directly. Instead of updating a row in a database, you append an event to an event log. The current state is derived by replaying all events.

```
Traditional (State-Based):
  Database row: { order_id: "ORD-123", status: "shipped", total: 99.99 }
  
  UPDATE orders SET status = 'shipped' WHERE order_id = 'ORD-123';
  → Previous state is LOST (we only know the current state)

Event Sourcing:
  Event Log for ORD-123:
    1. OrderCreated    { order_id: "ORD-123", total: 99.99, items: [...] }
    2. PaymentReceived { order_id: "ORD-123", amount: 99.99, method: "credit_card" }
    3. OrderConfirmed  { order_id: "ORD-123" }
    4. OrderShipped    { order_id: "ORD-123", tracking: "TRK-456" }
  
  Current state = replay all events:
    Start: empty order
    Apply OrderCreated → { status: "created", total: 99.99 }
    Apply PaymentReceived → { status: "paid", total: 99.99 }
    Apply OrderConfirmed → { status: "confirmed", total: 99.99 }
    Apply OrderShipped → { status: "shipped", total: 99.99, tracking: "TRK-456" }
  
  → Complete history is preserved. We know EXACTLY what happened and when.
```

```
Event Store (append-only log):

  +--------+------------------+-------------------------------------------+
  | Seq #  | Event Type       | Data                                      |
  +--------+------------------+-------------------------------------------+
  | 1      | OrderCreated     | { order_id: "ORD-123", total: 99.99 }    |
  | 2      | PaymentReceived  | { order_id: "ORD-123", amount: 99.99 }   |
  | 3      | OrderConfirmed   | { order_id: "ORD-123" }                  |
  | 4      | ItemAdded        | { order_id: "ORD-123", item: "Widget" }  |
  | 5      | OrderShipped     | { order_id: "ORD-123", tracking: "..." } |
  +--------+------------------+-------------------------------------------+
  
  Events are NEVER updated or deleted. The log is append-only.
  To "undo" something, you append a compensating event (e.g., "OrderCancelled").
```

> **Ruby Context:** **Rails Event Store (RES)** is the go-to library for event sourcing in Ruby/Rails. It provides an event store backed by PostgreSQL (or other databases), event publishing, subscriptions, projections, and more. It integrates naturally with ActiveRecord and Rails.

```ruby
# Ruby example: Event Sourcing with Rails Event Store
require 'rails_event_store'

# Define events (immutable value objects)
class OrderCreated < RailsEventStore::Event; end
class PaymentReceived < RailsEventStore::Event; end
class OrderConfirmed < RailsEventStore::Event; end
class OrderShipped < RailsEventStore::Event; end
class OrderCancelled < RailsEventStore::Event; end

# Configure the event store (typically in config/initializers/event_store.rb)
Rails.configuration.event_store = RailsEventStore::Client.new

# Publish events to a stream (one stream per aggregate/entity)
event_store = Rails.configuration.event_store

event_store.publish(
  OrderCreated.new(data: { order_id: 'ORD-123', total: 99.99, items: ['Widget'] }),
  stream_name: 'Order$ORD-123'
)

event_store.publish(
  PaymentReceived.new(data: { order_id: 'ORD-123', amount: 99.99, method: 'credit_card' }),
  stream_name: 'Order$ORD-123'
)

event_store.publish(
  OrderShipped.new(data: { order_id: 'ORD-123', tracking: 'TRK-456' }),
  stream_name: 'Order$ORD-123'
)

# Rebuild state by replaying events
events = event_store.read.stream('Order$ORD-123').to_a
order_state = events.each_with_object({}) do |event, state|
  case event
  when OrderCreated
    state.merge!(status: 'created', total: event.data[:total], items: event.data[:items])
  when PaymentReceived
    state.merge!(status: 'paid', payment_method: event.data[:method])
  when OrderConfirmed
    state.merge!(status: 'confirmed')
  when OrderShipped
    state.merge!(status: 'shipped', tracking: event.data[:tracking])
  when OrderCancelled
    state.merge!(status: 'cancelled', cancel_reason: event.data[:reason])
  end
end
# => { status: "shipped", total: 99.99, items: ["Widget"],
#      payment_method: "credit_card", tracking: "TRK-456" }
```

```ruby
# Ruby example: Aggregate with Event Sourcing pattern
class OrderAggregate
  attr_reader :id, :status, :total, :items, :tracking

  def initialize(id)
    @id = id
    @status = nil
    @total = 0
    @items = []
    @tracking = nil
    @unpublished_events = []
  end

  # Command methods — validate and produce events
  def place(total:, items:)
    raise 'Order already placed' if @status
    apply(OrderCreated.new(data: { order_id: @id, total: total, items: items }))
  end

  def confirm_payment(amount:, method:)
    raise 'Order not in created state' unless @status == 'created'
    raise 'Amount mismatch' unless amount == @total
    apply(PaymentReceived.new(data: { order_id: @id, amount: amount, method: method }))
  end

  def ship(tracking:)
    raise 'Order not paid' unless @status == 'paid'
    apply(OrderShipped.new(data: { order_id: @id, tracking: tracking }))
  end

  # Rebuild from event history
  def self.from_stream(stream_name, event_store:)
    order_id = stream_name.split('$').last
    aggregate = new(order_id)
    event_store.read.stream(stream_name).each do |event|
      aggregate.send(:apply_event, event)
    end
    aggregate
  end

  private

  def apply(event)
    apply_event(event)
    @unpublished_events << event
  end

  def apply_event(event)
    case event
    when OrderCreated
      @status = 'created'
      @total = event.data[:total]
      @items = event.data[:items]
    when PaymentReceived
      @status = 'paid'
    when OrderShipped
      @status = 'shipped'
      @tracking = event.data[:tracking]
    end
  end
end
```

**Benefits of Event Sourcing:**

| Benefit | Description |
|---------|-------------|
| **Complete audit trail** | Every change is recorded — who did what, when |
| **Time travel** | Reconstruct the state at any point in time |
| **Debugging** | Replay events to reproduce bugs |
| **Event replay** | Rebuild read models, fix bugs, create new projections |
| **Decoupling** | Events can be consumed by multiple services independently |
| **Compliance** | Full history for regulatory requirements (finance, healthcare) |

**Challenges of Event Sourcing:**

| Challenge | Description | Mitigation |
|-----------|-------------|------------|
| **Complexity** | More complex than CRUD | Use only where the benefits justify it |
| **Event schema evolution** | Events are immutable — how to handle schema changes? | Versioned events, upcasting |
| **Replay performance** | Replaying millions of events is slow | Snapshots (periodically save current state) |
| **Eventual consistency** | Read models may lag behind the event store | Accept eventual consistency or use synchronous projections |
| **Query complexity** | Can't easily query the event store (it's a log, not a table) | Use CQRS with materialized read models |

**Snapshots (Performance Optimization):**

```
Without snapshots:
  To get current state of ORD-123:
  → Replay ALL 10,000 events from the beginning
  → Slow!

With snapshots:
  Snapshot at event #9,500: { status: "confirmed", total: 99.99, ... }
  → Load snapshot
  → Replay only events #9,501 to #10,000 (500 events instead of 10,000)
  → Fast!

Snapshot strategy:
  - Create a snapshot every N events (e.g., every 100 events)
  - Or create a snapshot every T time (e.g., every hour)
  - Store snapshots alongside the event log
```

> **Ruby Context:** Rails Event Store supports snapshots through custom projection handlers. You can build a snapshot mechanism that periodically materializes the current state into a regular ActiveRecord model, then only replay events after the snapshot point.

---

### CQRS (Command Query Responsibility Segregation)

**CQRS** separates the **write model** (commands) from the **read model** (queries). Instead of using the same data model for both reads and writes, you use different models optimized for each.

```
Traditional (Single Model):
  
  Client → API → Same Database Model → Database
  
  Reads and writes use the SAME tables, SAME schema.
  Trade-off: schema optimized for writes is bad for reads, and vice versa.

CQRS (Separate Models):

  Write Side:                          Read Side:
  Client → Command API → Write Model → Event Store
                                          |
                                     (events published)
                                          |
                                          ↓
                              Projection / Denormalizer
                                          |
                                          ↓
                              Read Model (optimized for queries)
                                          ↑
                              Client → Query API
```

```
+----------+     Command      +-------------+     Events     +-------------+
|  Client  | ───────────────→ | Write Model | ─────────────→ | Event Store |
+----------+                  | (Domain)    |                +------+------+
                              +-------------+                       |
                                                              (publish events)
                                                                    |
                                                                    ↓
+----------+     Query        +-------------+     Project    +------+------+
|  Client  | ←──────────────  | Read Model  | ←──────────── | Projector   |
+----------+                  | (Denorm.)   |               +-------------+
                              +-------------+
```

**Why Separate Read and Write Models?**

| Aspect | Write Model | Read Model |
|--------|------------|------------|
| **Optimized for** | Data integrity, business rules, validation | Fast queries, specific views |
| **Schema** | Normalized (3NF), domain-driven | Denormalized, pre-computed, materialized views |
| **Database** | Relational DB, Event Store | NoSQL, Elasticsearch, Redis, read replicas |
| **Scale** | Typically lower write volume | Typically much higher read volume |
| **Example** | `orders` table with normalized relations | `order_summary` view with all data pre-joined |

**CQRS Without Event Sourcing:**

CQRS doesn't require Event Sourcing. You can use CQRS with a traditional database:

```
Write Side:
  Command → Validate → Write to PostgreSQL (normalized tables)
  → Publish event to Kafka: "OrderCreated"

Read Side:
  Kafka consumer receives "OrderCreated"
  → Update denormalized read model in Elasticsearch / Redis / DynamoDB
  
Query:
  Client → Query API → Read from Elasticsearch (fast, denormalized)
```

**CQRS + Event Sourcing (Full Pattern):**

```
Write Side:
  Command → Validate → Append event to Event Store
  → Event Store publishes event

Read Side:
  Projector consumes events from Event Store
  → Builds/updates materialized views (read models)
  → Multiple projectors can build different views from the same events

Example:
  Event: "OrderPlaced"
  
  Projector 1: Updates "order-details" read model (for order detail page)
  Projector 2: Updates "order-summary" read model (for order list page)
  Projector 3: Updates "revenue-dashboard" read model (for analytics)
  Projector 4: Updates "search-index" in Elasticsearch (for search)
```

> **Ruby Context:** Rails Event Store has first-class support for CQRS. You define event handlers (projectors) that subscribe to events and update read models. The write side uses domain aggregates and the event store, while the read side uses denormalized ActiveRecord models or external stores like Elasticsearch.

```ruby
# Ruby example: CQRS with Rails Event Store

# --- Write Side (Command + Event Store) ---
class PlaceOrderCommand
  def initialize(event_store: Rails.configuration.event_store)
    @event_store = event_store
  end

  def call(customer_id:, items:, total:)
    order_id = SecureRandom.uuid

    # Validate business rules
    raise 'Empty cart' if items.empty?
    raise 'Invalid total' if total <= 0

    # Append event to event store (write model)
    @event_store.publish(
      OrderPlaced.new(data: {
        order_id: order_id,
        customer_id: customer_id,
        items: items,
        total: total
      }),
      stream_name: "Order$#{order_id}"
    )

    order_id
  end
end

# --- Read Side (Projectors build denormalized read models) ---

# Projector 1: Order details read model
class OrderDetailsProjector
  def call(event)
    case event
    when OrderPlaced
      OrderReadModel.create!(
        order_id: event.data[:order_id],
        customer_id: event.data[:customer_id],
        items: event.data[:items],
        total: event.data[:total],
        status: 'placed'
      )
    when PaymentReceived
      OrderReadModel.find_by!(order_id: event.data[:order_id])
                    .update!(status: 'paid')
    when OrderShipped
      OrderReadModel.find_by!(order_id: event.data[:order_id])
                    .update!(status: 'shipped', tracking: event.data[:tracking])
    end
  end
end

# Projector 2: Revenue dashboard read model
class RevenueDashboardProjector
  def call(event)
    case event
    when OrderPlaced
      DailyRevenue.find_or_create_by(date: Date.today).tap do |record|
        record.increment!(:order_count)
        record.increment!(:total_revenue, event.data[:total])
      end
    end
  end
end

# Register projectors as event subscribers
event_store = Rails.configuration.event_store
event_store.subscribe(OrderDetailsProjector.new, to: [OrderPlaced, PaymentReceived, OrderShipped])
event_store.subscribe(RevenueDashboardProjector.new, to: [OrderPlaced])

# --- Query Side (Read from denormalized models) ---
class OrderQueryService
  def order_details(order_id)
    OrderReadModel.find_by!(order_id: order_id)
  end

  def customer_orders(customer_id)
    OrderReadModel.where(customer_id: customer_id).order(created_at: :desc)
  end

  def daily_revenue(date)
    DailyRevenue.find_by(date: date)
  end
end
```

**When to Use CQRS:**

| Use CQRS When | Don't Use CQRS When |
|---------------|-------------------|
| Read and write patterns are very different | Simple CRUD with similar read/write patterns |
| Read volume >> write volume (10:1 or more) | Low traffic, simple domain |
| Complex queries that don't map to the write model | Team is small and unfamiliar with the pattern |
| Need different databases for reads and writes | Consistency requirements are strict (CQRS introduces eventual consistency) |
| Event Sourcing is already in use | The added complexity isn't justified |

---

### Event Store

An **Event Store** is a database optimized for storing events. It's an append-only log where events are the source of truth.

```
Event Store Structure:

  +----------+--------+------------------+-------------------+------------------+
  | StreamId | Version| EventType        | Timestamp         | Data (JSON)      |
  +----------+--------+------------------+-------------------+------------------+
  | ORD-123  | 1      | OrderCreated     | 2026-04-23 10:00  | { total: 99.99 } |
  | ORD-123  | 2      | PaymentReceived  | 2026-04-23 10:01  | { amount: 99.99 }|
  | ORD-456  | 1      | OrderCreated     | 2026-04-23 10:02  | { total: 49.99 } |
  | ORD-123  | 3      | OrderShipped     | 2026-04-23 10:05  | { tracking: ...} |
  | ORD-456  | 2      | OrderCancelled   | 2026-04-23 10:10  | { reason: "..." }|
  +----------+--------+------------------+-------------------+------------------+

  StreamId: Groups events for a specific entity (aggregate)
  Version: Sequential version number within a stream (for optimistic concurrency)
  EventType: The type of event
  Data: The event payload (typically JSON)
```

**Event Store Operations:**

| Operation | Description |
|-----------|-------------|
| **Append** | Add a new event to a stream (with expected version for concurrency) |
| **Read stream** | Read all events for a specific stream (entity) |
| **Read all** | Read all events across all streams (for projections) |
| **Subscribe** | Get notified when new events are appended (for real-time projections) |

**Event Store Implementations:**

| Implementation | Description |
|---------------|-------------|
| **EventStoreDB** | Purpose-built event store (open source, by Greg Young) |
| **Apache Kafka** | Log-based streaming platform (can serve as event store with compaction) |
| **PostgreSQL** | Use an events table with append-only semantics |
| **DynamoDB** | Use with stream ID as partition key, version as sort key |
| **Custom** | Build on any append-only storage |

> **Ruby Context:** Rails Event Store uses PostgreSQL (via ActiveRecord) as its default storage backend. It creates an `event_store_events` table that stores events as append-only rows with stream correlation. For high-throughput needs, you can configure RES to use EventStoreDB as an alternative backend.

```ruby
# Ruby example: Rails Event Store backed by PostgreSQL
# The migration (auto-generated by rails_event_store generator):
#
# create_table :event_store_events, id: :string do |t|
#   t.string   :event_type,  null: false
#   t.jsonb    :metadata
#   t.jsonb    :data,         null: false
#   t.datetime :created_at,   null: false, precision: 6
#   t.datetime :valid_at,     precision: 6
# end
#
# create_table :event_store_events_in_streams do |t|
#   t.string   :stream,       null: false
#   t.integer  :position
#   t.string   :event_id,     null: false
#   t.datetime :created_at,   null: false, precision: 6
# end

# Using the event store with optimistic concurrency
event_store = Rails.configuration.event_store
stream = 'Order$ORD-123'

# Append with expected version (optimistic locking)
event_store.publish(
  OrderCreated.new(data: { order_id: 'ORD-123', total: 99.99 }),
  stream_name: stream,
  expected_version: :none  # Stream must not exist yet
)

event_store.publish(
  PaymentReceived.new(data: { order_id: 'ORD-123', amount: 99.99 }),
  stream_name: stream,
  expected_version: 0  # Expect exactly 1 event already in stream
)

# Read all events in a stream
events = event_store.read.stream(stream).to_a

# Read all events across all streams (for projections)
all_events = event_store.read.each.to_a

# Subscribe to new events in real time
event_store.subscribe(to: [OrderCreated]) do |event|
  puts "New order: #{event.data[:order_id]}"
end
```

---

### Eventual Consistency

In event-driven systems, the read model is updated **asynchronously** after the write. This means there's a window where the read model is stale — this is **eventual consistency**.

```
Timeline:

T0: Client sends "PlaceOrder" command
T1: Write model appends "OrderPlaced" event to Event Store
T2: Write model returns success to client: "Order placed!"
T3: Client queries "GetMyOrders" → Read model doesn't have the order yet! (stale)
T4: Projector processes "OrderPlaced" event → updates read model
T5: Client queries "GetMyOrders" → Read model now has the order ✅

The window T2-T4 is the "inconsistency window" (typically milliseconds to seconds).
```

**Handling Eventual Consistency:**

| Strategy | Description |
|----------|-------------|
| **Read-your-writes** | After a write, read from the write model (not the read model) for that specific entity |
| **Polling** | Client polls until the read model is updated |
| **Optimistic UI** | Client assumes success and shows the result immediately (update UI before server confirms) |
| **Version checking** | Include a version number; client retries if read model version is behind |
| **Causal consistency** | Track causal dependencies; ensure reads reflect all causally related writes |

> **Ruby Context:** In Rails, a common approach is to use synchronous event handlers for the "happy path" read model update (so the read model is updated before the response is sent), and asynchronous handlers for secondary projections (analytics, search indexing). Rails Event Store supports both sync and async subscribers.

```ruby
# Ruby example: Handling eventual consistency in Rails

# Strategy 1: Synchronous projection (read-your-writes)
# The projector runs in the same transaction as the event publish
event_store = Rails.configuration.event_store

# Sync subscriber — updates read model immediately (same request cycle)
event_store.subscribe(OrderDetailsProjector.new, to: [OrderPlaced])

# Async subscriber — updates eventually (via Sidekiq)
event_store.subscribe(
  ->(event) { AnalyticsProjectorWorker.perform_async(event.event_id) },
  to: [OrderPlaced]
)

# Strategy 2: Optimistic UI in the controller
class OrdersController < ApplicationController
  def create
    order_id = PlaceOrderCommand.new.call(
      customer_id: current_user.id,
      items: params[:items],
      total: params[:total]
    )

    # Return the order_id immediately — client shows "Order placed!"
    # Read model may not be updated yet, but client has the ID
    render json: { order_id: order_id, status: 'placed' }, status: :created
  end

  def show
    # Try read model first (fast, denormalized)
    order = OrderReadModel.find_by(order_id: params[:id])

    if order.nil?
      # Fallback: rebuild from event store (read-your-writes)
      events = event_store.read.stream("Order$#{params[:id]}").to_a
      if events.any?
        order = rebuild_order_from_events(events)
      else
        head :not_found
        return
      end
    end

    render json: order
  end
end

# Strategy 3: Version checking
class OrderReadModel < ApplicationRecord
  # Table has a `version` column that tracks the last applied event version
  def up_to_date?(expected_version)
    version >= expected_version
  end
end
```

---

### Saga Pattern (Orchestration vs Choreography)

A **Saga** is a pattern for managing **distributed transactions** across multiple services. Instead of a single ACID transaction spanning multiple databases (which doesn't work in microservices), a saga breaks the transaction into a sequence of local transactions, each publishing events or commands to trigger the next step.

**The Problem:**

```
Monolith (single database, single transaction):
  BEGIN TRANSACTION
    INSERT INTO orders (...)
    UPDATE inventory SET quantity = quantity - 1
    INSERT INTO payments (...)
    INSERT INTO shipments (...)
  COMMIT
  → All succeed or all fail (ACID)

Microservices (multiple databases, NO distributed transaction):
  Order Service → creates order (own DB)
  Inventory Service → reserves stock (own DB)
  Payment Service → charges customer (own DB)
  Shipping Service → creates shipment (own DB)
  
  What if Payment fails after Inventory already reserved stock?
  → Need to COMPENSATE (undo the inventory reservation)
  → This is what the Saga pattern handles
```

**Choreography-Based Saga:**

Each service listens for events and decides what to do next. No central coordinator — services react to events independently.

```
1. Order Service: creates order → publishes "OrderCreated"
2. Inventory Service: hears "OrderCreated" → reserves stock → publishes "StockReserved"
3. Payment Service: hears "StockReserved" → charges customer → publishes "PaymentCompleted"
4. Shipping Service: hears "PaymentCompleted" → creates shipment → publishes "OrderShipped"

Compensation (if Payment fails):
3. Payment Service: charge fails → publishes "PaymentFailed"
2. Inventory Service: hears "PaymentFailed" → releases stock → publishes "StockReleased"
1. Order Service: hears "StockReleased" → cancels order → publishes "OrderCancelled"
```

```
  Order         Inventory       Payment        Shipping
  Service       Service         Service        Service
    |               |               |               |
    |--OrderCreated→|               |               |
    |               |--StockReserved→               |
    |               |               |--PaymentCompleted→
    |               |               |               |--OrderShipped
    |               |               |               |
    
  Compensation (Payment fails):
    |               |               |               |
    |               |               |--PaymentFailed→|
    |               |←StockReleased-|               |
    |←OrderCancelled|               |               |
```

> **Ruby Context:** Choreography sagas map naturally to Rails Event Store subscriptions or Karafka consumers. Each service subscribes to the events it cares about and publishes its own events.

```ruby
# Ruby example: Choreography-based Saga with Rails Event Store

# Events
class OrderCreated < RailsEventStore::Event; end
class StockReserved < RailsEventStore::Event; end
class StockReservationFailed < RailsEventStore::Event; end
class PaymentCompleted < RailsEventStore::Event; end
class PaymentFailed < RailsEventStore::Event; end
class StockReleased < RailsEventStore::Event; end
class OrderCancelled < RailsEventStore::Event; end
class ShipmentCreated < RailsEventStore::Event; end

# Inventory Service — reacts to OrderCreated
class InventoryHandler
  def call(event)
    case event
    when OrderCreated
      if InventoryService.reserve(event.data[:items])
        event_store.publish(StockReserved.new(data: {
          order_id: event.data[:order_id],
          items: event.data[:items]
        }))
      else
        event_store.publish(StockReservationFailed.new(data: {
          order_id: event.data[:order_id],
          reason: 'Insufficient stock'
        }))
      end
    when PaymentFailed
      # Compensation: release reserved stock
      InventoryService.release(event.data[:order_id])
      event_store.publish(StockReleased.new(data: { order_id: event.data[:order_id] }))
    end
  end

  private

  def event_store = Rails.configuration.event_store
end

# Payment Service — reacts to StockReserved
class PaymentHandler
  def call(event)
    case event
    when StockReserved
      result = PaymentGateway.charge(
        order_id: event.data[:order_id],
        amount: event.data[:amount]
      )
      if result.success?
        event_store.publish(PaymentCompleted.new(data: {
          order_id: event.data[:order_id],
          transaction_id: result.transaction_id
        }))
      else
        event_store.publish(PaymentFailed.new(data: {
          order_id: event.data[:order_id],
          reason: result.error_message
        }))
      end
    end
  end

  private

  def event_store = Rails.configuration.event_store
end

# Wire up subscriptions
event_store = Rails.configuration.event_store
event_store.subscribe(InventoryHandler.new, to: [OrderCreated, PaymentFailed])
event_store.subscribe(PaymentHandler.new, to: [StockReserved])
```

**Orchestration-Based Saga:**

A central **Saga Orchestrator** coordinates the entire flow. It sends commands to each service and handles responses.

```
Saga Orchestrator:
  1. Send "ReserveStock" command → Inventory Service
  2. Receive "StockReserved" → Send "ChargeCustomer" command → Payment Service
  3. Receive "PaymentCompleted" → Send "CreateShipment" command → Shipping Service
  4. Receive "ShipmentCreated" → Mark saga as COMPLETED

Compensation (if Payment fails):
  3. Receive "PaymentFailed" → Send "ReleaseStock" command → Inventory Service
  4. Receive "StockReleased" → Send "CancelOrder" command → Order Service
  5. Mark saga as COMPENSATED
```

```
                    Saga
                  Orchestrator
                      |
         +------------+------------+
         |            |            |
    ReserveStock  ChargeCustomer  CreateShipment
         |            |            |
         ↓            ↓            ↓
    Inventory     Payment      Shipping
    Service       Service      Service
```

> **Ruby Context:** For orchestration sagas, you can use a state machine gem like **AASM** or **Statesman** to model the saga's lifecycle, combined with Sidekiq workers for each step.

```ruby
# Ruby example: Orchestration-based Saga with state machine
class OrderSaga
  include AASM

  aasm column: :state do
    state :started, initial: true
    state :stock_reserved
    state :payment_completed
    state :shipped
    state :completed
    state :compensating
    state :failed

    event :reserve_stock do
      transitions from: :started, to: :stock_reserved, after: :do_charge_payment
    end

    event :complete_payment do
      transitions from: :stock_reserved, to: :payment_completed, after: :do_create_shipment
    end

    event :create_shipment do
      transitions from: :payment_completed, to: :shipped, after: :do_complete
    end

    event :complete do
      transitions from: :shipped, to: :completed
    end

    # Compensation events
    event :payment_failed do
      transitions from: :stock_reserved, to: :compensating, after: :do_release_stock
    end

    event :stock_released do
      transitions from: :compensating, to: :failed, after: :do_cancel_order
    end
  end

  def start(order_id:, items:, amount:)
    @order_id = order_id
    @items = items
    @amount = amount
    ReserveStockWorker.perform_async(order_id, items)
  end

  private

  def do_charge_payment
    ChargePaymentWorker.perform_async(@order_id, @amount)
  end

  def do_create_shipment
    CreateShipmentWorker.perform_async(@order_id)
  end

  def do_complete
    Rails.logger.info("Saga completed for order #{@order_id}")
  end

  def do_release_stock
    ReleaseStockWorker.perform_async(@order_id, @items)
  end

  def do_cancel_order
    CancelOrderWorker.perform_async(@order_id)
  end
end

# Saga orchestrator worker — handles responses from services
class SagaOrchestratorWorker
  include Sidekiq::Job

  def perform(saga_id, event_type, payload)
    saga = OrderSaga.find(saga_id)

    case event_type
    when 'stock_reserved'    then saga.reserve_stock!
    when 'payment_completed' then saga.complete_payment!
    when 'shipment_created'  then saga.create_shipment!
    when 'payment_failed'    then saga.payment_failed!
    when 'stock_released'    then saga.stock_released!
    end

    saga.save!
  end
end
```

**Choreography vs Orchestration:**

| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| **Coordination** | Decentralized (services react to events) | Centralized (orchestrator directs) |
| **Coupling** | Loose (services don't know about each other) | Medium (orchestrator knows all services) |
| **Complexity** | Simple for few steps; hard to track for many | Clear flow; easier to understand and debug |
| **Single point of failure** | None | Orchestrator is a SPOF (must be highly available) |
| **Visibility** | Hard to see the full flow (distributed across services) | Easy to see the full flow (in the orchestrator) |
| **Adding steps** | Add a new service that listens to events | Modify the orchestrator |
| **Best for** | Simple sagas (2-4 steps) | Complex sagas (5+ steps), need clear visibility |

**Saga Best Practices:**

| Practice | Description |
|----------|-------------|
| **Idempotent steps** | Each step must be idempotent (safe to retry) |
| **Compensating transactions** | Every step must have a compensating action (undo) |
| **Timeout handling** | Set timeouts for each step; trigger compensation on timeout |
| **Saga state persistence** | Store saga state durably (survives crashes) |
| **Monitoring** | Track saga progress, alert on stuck or failed sagas |
| **Dead letter handling** | Move permanently failed sagas to a DLQ for investigation |

---


## 24.4 Stream Processing

> **Stream processing** is the real-time processing of data as it arrives, as opposed to **batch processing** which processes data in large chunks at scheduled intervals. Message queues like Kafka produce streams of events — stream processing frameworks consume these streams, transform them, aggregate them, and produce results in real time.

> **Ruby Context:** Ruby is not traditionally a stream processing language (JVM-based tools like Kafka Streams and Flink dominate), but Ruby applications commonly produce and consume streams via **Karafka**, and can perform lightweight stream transformations within consumers. For heavy stream processing (windowing, joins, complex aggregations), Ruby teams typically pair their Ruby services with dedicated stream processors (Flink, ksqlDB, Kafka Streams) running on the JVM.

---

### Batch Processing vs Stream Processing

```
Batch Processing:
  Collect data for hours/days → Process all at once → Output results
  
  Example: Generate daily sales report
  11:59 PM: Collect all sales events from the day
  12:00 AM: Run batch job (MapReduce, Spark)
  12:30 AM: Report ready
  
  Timeline: [collect...collect...collect] → [PROCESS] → [result]
  Latency: Hours (data is stale by the time it's processed)

Stream Processing:
  Process each event as it arrives → Output results continuously
  
  Example: Real-time sales dashboard
  Each sale event → processed immediately → dashboard updated in real time
  
  Timeline: [event→process→result] [event→process→result] [event→process→result]
  Latency: Milliseconds to seconds (near real-time)
```

**Comparison:**

| Feature | Batch Processing | Stream Processing |
|---------|-----------------|-------------------|
| **Latency** | Minutes to hours | Milliseconds to seconds |
| **Data** | Bounded (finite dataset) | Unbounded (infinite stream) |
| **Processing** | All at once | Continuous, event-by-event |
| **Complexity** | Simpler (complete data available) | More complex (incomplete data, ordering, late arrivals) |
| **Use cases** | ETL, reports, ML training, data warehousing | Real-time analytics, alerting, fraud detection, live dashboards |
| **Tools** | Hadoop MapReduce, Spark (batch mode), Hive | Kafka Streams, Apache Flink, Spark Streaming, Apache Storm |
| **Fault tolerance** | Restart the job | Checkpointing, exactly-once semantics |

> **Ruby Context:** Ruby excels at batch processing with tools like **Rake tasks**, **ActiveJob**, and **Sidekiq** (for parallel batch workers). For stream processing, Karafka consumers act as lightweight stream processors. For the heavy lifting, Ruby apps typically delegate to JVM-based stream processors.

```ruby
# Ruby example: Batch vs Stream processing patterns

# --- Batch Processing (Rake task, runs nightly) ---
# lib/tasks/daily_report.rake
namespace :reports do
  desc 'Generate daily sales report'
  task daily_sales: :environment do
    yesterday = Date.yesterday

    orders = Order.where(created_at: yesterday.all_day)
    total_revenue = orders.sum(:total)
    order_count = orders.count
    avg_order_value = order_count > 0 ? total_revenue / order_count : 0

    DailySalesReport.create!(
      date: yesterday,
      total_revenue: total_revenue,
      order_count: order_count,
      avg_order_value: avg_order_value
    )

    puts "Report generated: #{order_count} orders, $#{total_revenue} revenue"
  end
end

# --- Stream Processing (Karafka consumer, processes in real time) ---
class SalesDashboardConsumer < Karafka::BaseConsumer
  def consume
    messages.each do |message|
      order = message.payload

      # Update real-time counters in Redis
      redis = Redis.new
      today = Date.today.to_s

      redis.incr("sales:#{today}:count")
      redis.incrbyfloat("sales:#{today}:revenue", order['total'])

      # Push update to live dashboard via ActionCable
      ActionCable.server.broadcast('sales_dashboard', {
        order_count: redis.get("sales:#{today}:count").to_i,
        total_revenue: redis.get("sales:#{today}:revenue").to_f
      })
    end
  end
end
```

**Lambda Architecture (Batch + Stream):**

```
                         +-------------------+
                         |   Incoming Data   |
                         +---------+---------+
                                   |
                    +--------------+--------------+
                    |                             |
           +--------v--------+          +---------v--------+
           |  Batch Layer    |          |  Speed Layer     |
           |  (MapReduce,    |          |  (Kafka Streams, |
           |   Spark Batch)  |          |   Flink)         |
           +--------+--------+          +---------+--------+
                    |                             |
           +--------v--------+          +---------v--------+
           |  Batch Views    |          |  Real-time Views |
           |  (accurate,     |          |  (approximate,   |
           |   high latency) |          |   low latency)   |
           +--------+--------+          +---------+--------+
                    |                             |
                    +--------------+--------------+
                                   |
                         +---------v---------+
                         |   Serving Layer   |
                         |   (merge batch +  |
                         |    real-time)      |
                         +-------------------+

Batch layer: Processes ALL historical data periodically (accurate but slow)
Speed layer: Processes new data in real time (fast but may be approximate)
Serving layer: Merges both views to serve queries
```

**Kappa Architecture (Stream Only):**

```
                         +-------------------+
                         |   Incoming Data   |
                         +---------+---------+
                                   |
                         +---------v---------+
                         |   Stream Layer    |
                         |   (Kafka Streams, |
                         |    Flink)         |
                         +---------+---------+
                                   |
                         +---------v---------+
                         |   Serving Layer   |
                         +-------------------+

Everything is a stream. No separate batch layer.
To reprocess historical data: replay the stream from the beginning.
Simpler than Lambda architecture (one codebase, one processing path).
```

| Architecture | Pros | Cons |
|-------------|------|------|
| **Lambda** | Accurate batch + fast real-time | Two codebases, complex to maintain |
| **Kappa** | Simple (one codebase), stream-only | Reprocessing requires replaying entire stream |

---

### Apache Kafka Streams

Covered in section 24.2 (Kafka Streams). Key points for stream processing:

```
Kafka Streams Processing Topology:

  Source Topic(s) → Processor Nodes → Sink Topic(s)

  Processor operations:
  - filter(): Keep only events matching a condition
  - map() / mapValues(): Transform events
  - flatMap(): One event → multiple events
  - groupByKey() / groupBy(): Group events by key
  - aggregate() / count() / reduce(): Aggregate grouped events
  - join(): Join two streams or a stream with a table
  - windowedBy(): Group events into time windows
  - branch(): Split a stream into multiple streams based on conditions
```

> **Ruby Context:** While Kafka Streams is JVM-only, Karafka consumers can replicate many of these operations. Karafka 2.x supports filtering, routing, and batch processing within consumers. For complex topologies, consider using **ksqlDB** (SQL-based stream processing on Kafka) alongside your Ruby services — it requires no JVM code and can be queried via REST API from Ruby.

```ruby
# Ruby example: Stream processing patterns with Karafka

# Filter + Transform (equivalent to Kafka Streams filter + map)
class HighValueOrderConsumer < Karafka::BaseConsumer
  def consume
    messages.each do |message|
      order = message.payload

      # Filter: only process high-value orders
      next unless order['total'].to_f > 100.0

      # Transform: enrich with customer data
      customer = CustomerService.fetch(order['customer_id'])
      enriched_order = order.merge(
        'customer_name' => customer.name,
        'customer_tier' => customer.tier,
        'processed_at' => Time.current.iso8601
      )

      # Produce to output topic (like a Kafka Streams sink)
      Karafka.producer.produce_sync(
        topic: 'high-value-orders-enriched',
        payload: enriched_order.to_json,
        key: order['order_id']
      )
    end
  end
end

# Branch: Split stream into multiple output topics
class OrderRouterConsumer < Karafka::BaseConsumer
  def consume
    messages.each do |message|
      order = message.payload

      topic = case order['region']
              when 'us' then 'orders-us'
              when 'eu' then 'orders-eu'
              else 'orders-other'
              end

      Karafka.producer.produce_sync(
        topic: topic,
        payload: message.raw_payload,
        key: order['order_id']
      )
    end
  end
end
```

---

### Apache Flink (Concepts)

**Apache Flink** is a distributed stream processing framework designed for **stateful computations** over unbounded and bounded data streams. It's more powerful than Kafka Streams for complex stream processing.

```
Flink Architecture:

  +------------------+
  |   Job Manager    |  (coordinates execution, manages checkpoints)
  +--------+---------+
           |
  +--------v---------+  +------------------+  +------------------+
  |  Task Manager 1  |  |  Task Manager 2  |  |  Task Manager 3  |
  |  (worker node)   |  |  (worker node)   |  |  (worker node)   |
  |  [Task] [Task]   |  |  [Task] [Task]   |  |  [Task] [Task]   |
  +------------------+  +------------------+  +------------------+
```

**Flink vs Kafka Streams:**

| Feature | Kafka Streams | Apache Flink |
|---------|--------------|-------------|
| **Deployment** | Library (runs in your app) | Standalone cluster |
| **Source** | Kafka only | Kafka, files, sockets, databases, custom |
| **Processing** | Stream only | Stream + batch (unified) |
| **State management** | RocksDB (local) | RocksDB + distributed snapshots |
| **Windowing** | Tumbling, hopping, sliding, session | All of the above + custom windows |
| **Exactly-once** | Within Kafka ecosystem | End-to-end exactly-once |
| **Complexity** | Lower (library, no cluster) | Higher (separate cluster to manage) |
| **Best for** | Kafka-centric stream processing | Complex event processing, large-scale analytics |

> **Ruby Context:** Flink is a JVM framework with no native Ruby client. Ruby services interact with Flink indirectly — typically by producing events to Kafka (which Flink consumes) and reading Flink's output from Kafka topics or databases. For teams that need Flink-level power but prefer not to write Java/Scala, **ksqlDB** or **Flink SQL** provide SQL interfaces that can be called from Ruby via REST APIs.

---

### Windowing (Tumbling, Sliding, Session)

When processing streams, you often need to aggregate events over **time windows** — for example, "count orders per 5-minute window" or "average response time per minute."

**Tumbling Window:**

Fixed-size, non-overlapping windows. Each event belongs to exactly one window.

```
Time:    |  0  |  1  |  2  |  3  |  4  |  5  |  6  |  7  |  8  |  9  |
Events:     A     B     C     D     E     F     G     H     I     J

Tumbling window (size = 3):
  Window 1: [0-2]  → events A, B, C
  Window 2: [3-5]  → events D, E, F
  Window 3: [6-8]  → events G, H, I
  Window 4: [9-11] → event J (so far)

  |----Window 1----|----Window 2----|----Window 3----|----Window 4----|
  |  A    B    C   |  D    E    F   |  G    H    I   |  J             |
```

**Use cases:** Hourly reports, daily aggregations, per-minute metrics.

**Sliding Window (Hopping Window):**

Fixed-size windows that **overlap**. Defined by window size and slide interval. An event can belong to multiple windows.

```
Time:    |  0  |  1  |  2  |  3  |  4  |  5  |  6  |  7  |
Events:     A     B     C     D     E     F     G     H

Sliding window (size = 4, slide = 2):
  Window 1: [0-3]  → events A, B, C, D
  Window 2: [2-5]  → events C, D, E, F       ← overlaps with Window 1
  Window 3: [4-7]  → events E, F, G, H       ← overlaps with Window 2

  |--------Window 1--------|
              |--------Window 2--------|
                          |--------Window 3--------|
  |  A    B  |  C    D   |  E    F   |  G    H    |
```

**Use cases:** Moving averages, trend detection, smoothed metrics.

**Session Window:**

Dynamic windows based on **activity gaps**. A session window closes when there's no activity for a specified gap duration. Window size varies.

```
Time:    |  0  |  1  |  2  |  3  |  4  |  5  |  6  |  7  |  8  |  9  | 10 | 11 | 12 |
Events:     A     B     C                 D     E                        F     G

Session window (gap = 2):
  Session 1: [0-2]  → events A, B, C  (gap of 3 after C → session closes)
  Session 2: [5-6]  → events D, E     (gap of 4 after E → session closes)
  Session 3: [10-11] → events F, G    (ongoing or closes after gap)

  |--Session 1--|     gap     |--Session 2--|     gap     |--Session 3--|
  |  A    B    C |             |  D    E     |             |  F    G     |
```

**Use cases:** User session analysis, click stream analysis, activity tracking.

**Window Comparison:**

| Window Type | Size | Overlap | Use Case |
|------------|------|---------|----------|
| **Tumbling** | Fixed | No | Periodic aggregations (hourly counts, daily totals) |
| **Sliding** | Fixed | Yes | Moving averages, trend detection |
| **Session** | Variable | No | User sessions, activity-based grouping |

> **Ruby Context:** Ruby doesn't have built-in windowing primitives like Flink or Kafka Streams, but you can implement simple windowing patterns using Redis for state and Karafka consumers for processing. For production-grade windowing, use a dedicated stream processor (Flink, ksqlDB) and consume the aggregated results in Ruby.

```ruby
# Ruby example: Simple tumbling window with Redis + Karafka
class OrderCountWindowConsumer < Karafka::BaseConsumer
  WINDOW_SIZE_SECONDS = 300  # 5-minute tumbling window

  def consume
    redis = Redis.new

    messages.each do |message|
      order = message.payload
      event_time = Time.parse(order['timestamp'])

      # Calculate which window this event belongs to
      window_start = (event_time.to_i / WINDOW_SIZE_SECONDS) * WINDOW_SIZE_SECONDS
      window_key = "order_count:window:#{window_start}"

      # Increment counter for this window
      redis.incr(window_key)
      redis.expire(window_key, WINDOW_SIZE_SECONDS * 3)  # TTL: 3 windows

      # Increment per-product counter
      product_key = "order_count:window:#{window_start}:product:#{order['product_id']}"
      redis.incr(product_key)
      redis.expire(product_key, WINDOW_SIZE_SECONDS * 3)
    end
  end
end

# Ruby example: Simple sliding window (moving average) with Redis
class MovingAverageCalculator
  def initialize(redis: Redis.new, window_seconds: 300)
    @redis = redis
    @window_seconds = window_seconds
  end

  def add_value(metric_name, value, timestamp: Time.current)
    key = "moving_avg:#{metric_name}"
    # Store value with timestamp as score in a sorted set
    @redis.zadd(key, timestamp.to_f, "#{timestamp.to_f}:#{value}")
    # Remove entries outside the window
    cutoff = (timestamp - @window_seconds).to_f
    @redis.zremrangebyscore(key, '-inf', cutoff)
  end

  def average(metric_name)
    key = "moving_avg:#{metric_name}"
    entries = @redis.zrange(key, 0, -1)
    return 0.0 if entries.empty?

    values = entries.map { |e| e.split(':').last.to_f }
    values.sum / values.size
  end
end
```

---

### Watermarks and Late Data

In real-world stream processing, events can arrive **out of order** or **late** due to network delays, retries, or clock skew. **Watermarks** help the system decide when a window is "complete" and can be processed.

**The Problem:**

```
Event time vs Processing time:

  Event A occurred at T=1, arrived at T=1 (on time)
  Event B occurred at T=2, arrived at T=5 (late! delayed by 3 seconds)
  Event C occurred at T=3, arrived at T=3 (on time)

  Tumbling window [0-3]:
    At processing time T=3: Window has events A, C
    At processing time T=5: Event B arrives (event time T=2, belongs to window [0-3])
    But window [0-3] was already processed!
    
  → Event B is LATE DATA. What do we do with it?
```

**Watermarks:**

A watermark is a **timestamp** that declares: "All events with event time ≤ this watermark have arrived." It's the system's estimate of how far behind the data stream is.

```
Stream of events (by event time):

  Event time:     1    2    3    4    5    6    7
  Arrival time:   1    5    3    4    6    8    7
                       ↑ (late)           ↑ (late)

Watermark (with allowed lateness = 2):
  At processing time 3: watermark = 1 (events up to T=1 are complete)
  At processing time 5: watermark = 3 (events up to T=3 are complete)
  At processing time 6: watermark = 4 (events up to T=4 are complete)

  Window [0-3] fires when watermark passes 3.
  Event B (event time=2) arrives at processing time 5 → watermark is 3 → B is ON TIME
  (because watermark hadn't passed 2 when B arrived)
```

**Handling Late Data:**

| Strategy | Description | Trade-off |
|----------|-------------|-----------|
| **Drop** | Ignore late events | Simple but data loss |
| **Allowed lateness** | Accept late events within a grace period; update the window result | More accurate but uses more memory (keep windows open longer) |
| **Side output** | Route late events to a separate stream for special handling | Flexible but adds complexity |
| **Reprocessing** | Periodically reprocess with batch to correct for late data (Lambda architecture) | Most accurate but complex |

```
Flink example (allowed lateness):

  Window [0-5] with allowed lateness of 10 seconds:
  
  T=5:  Window fires with events A, C, D (first result)
  T=8:  Late event B arrives (event time=2) → window UPDATES result (A, B, C, D)
  T=15: Watermark passes 5+10=15 → window is PURGED (no more updates accepted)
  T=20: Very late event E arrives (event time=4) → DROPPED (past allowed lateness)
```

> **Ruby Context:** Watermarks and late data handling are primarily concerns for dedicated stream processors (Flink, Kafka Streams). In Ruby Karafka consumers, you can implement basic late-data handling by checking event timestamps against window boundaries and routing late events to a separate topic or DLQ.

```ruby
# Ruby example: Basic late data handling in a Karafka consumer
class WindowedAggregationConsumer < Karafka::BaseConsumer
  WINDOW_SIZE = 300        # 5-minute window
  ALLOWED_LATENESS = 60    # Accept events up to 60 seconds late

  def consume
    redis = Redis.new

    messages.each do |message|
      order = message.payload
      event_time = Time.parse(order['timestamp'])
      now = Time.current

      window_start = (event_time.to_i / WINDOW_SIZE) * WINDOW_SIZE
      window_end = window_start + WINDOW_SIZE
      window_deadline = window_end + ALLOWED_LATENESS

      if now.to_i > window_deadline
        # Event is too late — route to late-data topic
        Karafka.producer.produce_sync(
          topic: 'orders-late-data',
          payload: message.raw_payload,
          key: order['order_id']
        )
        Rails.logger.warn("Late event dropped: #{order['order_id']} " \
                          "(event_time=#{event_time}, window_deadline=#{Time.at(window_deadline)})")
      else
        # Event is on time (or within allowed lateness) — aggregate
        window_key = "order_window:#{window_start}"
        redis.incr(window_key)
        redis.incrbyfloat("#{window_key}:revenue", order['total'].to_f)
        redis.expire(window_key, WINDOW_SIZE * 3)
      end
    end
  end
end
```

---


## 24.5 Use Cases

> Message queues and event-driven architecture are not theoretical concepts — they're the backbone of most large-scale production systems. This section covers the most common real-world use cases with architectural diagrams showing how the components fit together.

> **Ruby Context:** Ruby/Rails applications use these patterns extensively. Sidekiq processes billions of jobs daily across the Ruby ecosystem. Shopify, GitHub, Basecamp, and other large Rails applications rely heavily on background job queues and event-driven patterns for scalability.

---

### Order Processing Pipeline

An e-commerce order goes through multiple stages — validation, payment, inventory, shipping, notification. A message queue decouples these stages, making the system resilient and scalable.

```
                                    Message Queue (Kafka)
                                    
Customer → API Gateway → Order Service ──→ [order-events topic]
                              |                    |
                              |          +---------+---------+---------+
                              |          |         |         |         |
                              |          ↓         ↓         ↓         ↓
                              |     Inventory  Payment  Notification  Analytics
                              |     Service    Service  Service       Service
                              |          |         |         |         |
                              |          ↓         ↓         ↓         ↓
                              |     [stock-    [payment- [notif-    [analytics-
                              |      events]   events]   events]    events]
                              |
                              ↓
                         Order Database

Flow:
1. Customer places order → Order Service creates order, publishes "OrderCreated"
2. Inventory Service: reserves stock → publishes "StockReserved" or "OutOfStock"
3. Payment Service: charges customer → publishes "PaymentCompleted" or "PaymentFailed"
4. If payment fails → Inventory Service compensates (releases stock) [Saga pattern]
5. Notification Service: sends order confirmation email/SMS
6. Analytics Service: updates dashboards, reports

Benefits:
- Order Service responds immediately (doesn't wait for payment, inventory, etc.)
- If Payment Service is down, messages queue up and are processed when it recovers
- Each service scales independently (Notification Service can have 10 consumers during peak)
- Adding a new service (e.g., Fraud Detection) = just subscribe to "order-events" topic
```

> **Ruby Context:** A typical Rails e-commerce app might use Sidekiq for the notification and analytics steps, Karafka for cross-service event streaming, and Rails Event Store for the order aggregate's event sourcing.

```ruby
# Ruby example: Order processing pipeline with Sidekiq + Karafka

# Step 1: Rails controller creates order and publishes event
class OrdersController < ApplicationController
  def create
    order = Order.create!(order_params.merge(status: 'pending'))

    # Publish to Kafka for cross-service consumption
    Karafka.producer.produce_sync(
      topic: 'order-events',
      payload: {
        event_type: 'OrderCreated',
        order_id: order.id,
        customer_id: order.customer_id,
        items: order.items.as_json,
        total: order.total.to_f
      }.to_json,
      key: order.id.to_s
    )

    # Also enqueue immediate background jobs via Sidekiq
    SendOrderConfirmationWorker.perform_async(order.id)

    render json: { order_id: order.id, status: 'pending' }, status: :created
  end
end

# Step 2: Karafka consumer in Payment Service
class PaymentConsumer < Karafka::BaseConsumer
  def consume
    messages.each do |message|
      event = message.payload
      next unless event['event_type'] == 'OrderCreated'

      result = PaymentGateway.charge(
        order_id: event['order_id'],
        amount: event['total']
      )

      output_event = if result.success?
        { event_type: 'PaymentCompleted', order_id: event['order_id'],
          transaction_id: result.transaction_id }
      else
        { event_type: 'PaymentFailed', order_id: event['order_id'],
          reason: result.error }
      end

      Karafka.producer.produce_sync(
        topic: 'payment-events',
        payload: output_event.to_json,
        key: event['order_id']
      )
    end
  end
end

# Step 3: Sidekiq worker for notifications
class SendOrderConfirmationWorker
  include Sidekiq::Job
  sidekiq_options retry: 3, queue: 'notifications'

  def perform(order_id)
    order = Order.find(order_id)
    OrderMailer.confirmation(order).deliver_now
    PushNotificationService.send(order.customer, "Order #{order.id} confirmed!")
  end
end
```

---

### Notification System

A notification system must handle multiple channels (push, email, SMS), prioritization, rate limiting, and user preferences — all at massive scale.

```
                    +-------------------+
                    | Notification API  |
                    +--------+----------+
                             |
                    +--------v----------+
                    | Priority Router   |
                    | (high/medium/low) |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v------+ +----v-------+ +----v-------+
     | High Priority | | Med Priority| | Low Priority|
     | Queue         | | Queue       | | Queue       |
     +--------+------+ +----+-------+ +----+-------+
              |              |              |
              +--------------+--------------+
                             |
                    +--------v----------+
                    | Channel Router    |
                    | (based on user    |
                    |  preferences)     |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v------+ +----v-------+ +----v-------+
     | Push Service  | | Email      | | SMS        |
     | (FCM/APNs)    | | Service    | | Service    |
     +---------------+ | (SendGrid) | | (Twilio)   |
                       +------------+ +------------+

Features enabled by message queues:
- Priority queues: critical alerts processed before marketing emails
- Rate limiting: don't overwhelm SMS provider (100 msg/s limit)
- Retry with backoff: if push notification fails, retry after 1s, 5s, 30s
- Dead letter queue: permanently failed notifications for investigation
- Channel fallback: if push fails, try email; if email fails, try SMS
- Batching: batch 1000 emails into one API call to SendGrid
```

> **Ruby Context:** Sidekiq's priority queues map perfectly to this pattern. You define multiple queues with different priorities, and Sidekiq processes higher-priority queues first.

```ruby
# Ruby example: Notification system with Sidekiq priority queues

# config/sidekiq.yml
# :queues:
#   - [critical, 10]      # Weight 10 — processed first
#   - [notifications, 5]  # Weight 5
#   - [marketing, 1]      # Weight 1 — processed last

class NotificationService
  PRIORITY_MAP = {
    high: 'critical',
    medium: 'notifications',
    low: 'marketing'
  }.freeze

  def self.send(user:, message:, priority: :medium, channels: nil)
    channels ||= user.notification_preferences
    queue = PRIORITY_MAP[priority]

    channels.each do |channel|
      case channel
      when :push
        PushNotificationWorker.set(queue: queue).perform_async(user.id, message)
      when :email
        EmailNotificationWorker.set(queue: queue).perform_async(user.id, message)
      when :sms
        SmsNotificationWorker.set(queue: queue).perform_async(user.id, message)
      end
    end
  end
end

class PushNotificationWorker
  include Sidekiq::Job
  sidekiq_options retry: 5

  sidekiq_retry_in do |count|
    [1, 5, 30, 300, 3600][count] || 3600  # 1s, 5s, 30s, 5min, 1hr
  end

  sidekiq_retries_exhausted do |job, exception|
    # Fallback to email if push fails permanently
    user_id, message = job['args']
    EmailNotificationWorker.perform_async(user_id, message)
    AlertService.notify("Push notification permanently failed for user #{user_id}")
  end

  def perform(user_id, message)
    user = User.find(user_id)
    FCMClient.send(device_token: user.device_token, body: message)
  end
end

class SmsNotificationWorker
  include Sidekiq::Job
  sidekiq_options retry: 3, queue: 'notifications'

  # Rate limiting with sidekiq-limiter or custom logic
  def perform(user_id, message)
    user = User.find(user_id)

    # Simple rate limiting with Redis
    redis = Redis.new
    rate_key = "sms_rate:#{Time.current.to_i / 60}"  # Per-minute bucket
    current_count = redis.incr(rate_key)
    redis.expire(rate_key, 120)

    if current_count > 100  # Max 100 SMS per minute
      # Re-enqueue with delay
      self.class.perform_in(30, user_id, message)
      return
    end

    TwilioClient.send_sms(to: user.phone, body: message)
  end
end
```

---

### Log Aggregation

In a microservices architecture with hundreds of services, each producing logs, you need a centralized system to collect, process, and store logs.

```
+----------+     +----------+     +----------+
| Service A|     | Service B|     | Service C|
| (logs)   |     | (logs)   |     | (logs)   |
+----+-----+     +----+-----+     +----+-----+
     |                |                |
     +----------------+----------------+
                      |
             +--------v---------+
             | Log Shipper      |
             | (Filebeat,       |
             |  Fluentd,        |
             |  Vector)         |
             +--------+---------+
                      |
             +--------v---------+
             | Kafka            |
             | (logs topic)     |
             +--------+---------+
                      |
         +------------+------------+
         |            |            |
+--------v---+ +-----v------+ +---v-----------+
| Elasticsearch| | S3 / HDFS | | Alert Service |
| (search,    | | (long-term | | (real-time    |
|  dashboards)| |  archive)  | |  error alerts)|
+-------------+ +------------+ +---------------+

Why Kafka for logs:
- Handles massive throughput (millions of log lines/second)
- Decouples log producers from consumers
- Multiple consumers: Elasticsearch for search, S3 for archive, alerting for real-time
- Buffering: if Elasticsearch is slow, logs queue in Kafka (no data loss)
- Replay: reprocess logs if Elasticsearch index is corrupted
```

> **Ruby Context:** Rails applications typically use **Lograge** for structured JSON logging, which makes logs easy to ship to Kafka or Elasticsearch. The **Semantic Logger** gem is another popular choice for structured, high-performance logging. For log shipping, **Fluentd** (written in Ruby!) is a natural fit.

```ruby
# Ruby example: Structured logging with Lograge + Kafka shipping

# config/initializers/lograge.rb
Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.formatter = Lograge::Formatters::Json.new
  config.lograge.custom_payload do |controller|
    {
      host: Socket.gethostname,
      service: 'order-service',
      request_id: controller.request.request_id,
      user_id: controller.current_user&.id
    }
  end
  config.lograge.custom_options = lambda do |event|
    {
      params: event.payload[:params].except('controller', 'action'),
      exception: event.payload[:exception]&.first,
      duration_ms: event.duration.round(2)
    }
  end
end

# Output: {"method":"GET","path":"/orders","format":"json","controller":"OrdersController",
#          "action":"index","status":200,"duration_ms":45.23,"service":"order-service",
#          "host":"web-01","request_id":"abc-123","user_id":456}

# Optional: Ship logs directly to Kafka from Ruby (in addition to file-based shipping)
class KafkaLogSubscriber < ActiveSupport::LogSubscriber
  def process_action(event)
    payload = {
      service: 'order-service',
      level: 'info',
      method: event.payload[:method],
      path: event.payload[:path],
      status: event.payload[:status],
      duration_ms: event.duration.round(2),
      timestamp: Time.current.iso8601
    }

    Karafka.producer.produce_async(
      topic: 'application-logs',
      payload: payload.to_json
    )
  rescue StandardError => e
    # Never let log shipping break the application
    Rails.logger.warn("Failed to ship log to Kafka: #{e.message}")
  end
end
```

---

### Real-Time Analytics

Process events in real time to power dashboards, detect anomalies, and trigger alerts.

```
+----------+     +----------+     +----------+
| Web App  |     | Mobile   |     | IoT      |
| (clicks) |     | (events) |     | (sensors)|
+----+-----+     +----+-----+     +----+-----+
     |                |                |
     +----------------+----------------+
                      |
             +--------v---------+
             | Kafka            |
             | (events topic)   |
             +--------+---------+
                      |
         +------------+------------+
         |                         |
+--------v---------+     +---------v--------+
| Stream Processor |     | Stream Processor |
| (Flink/KStreams) |     | (Flink/KStreams) |
| - Count events   |     | - Detect anomaly |
| - Aggregate by   |     | - Fraud detection|
|   region/product |     | - Alert if       |
| - Windowed stats |     |   threshold      |
+--------+---------+     +---------+--------+
         |                         |
+--------v---------+     +---------v--------+
| Dashboard DB     |     | Alert Service    |
| (Redis, InfluxDB)|     | (PagerDuty,      |
| → Grafana        |     |  Slack, Email)   |
+------------------+     +------------------+

Example metrics computed in real time:
- Orders per minute by region
- Revenue per hour by product category
- Error rate per service (5-minute rolling window)
- Active users right now (session windows)
- Fraud score per transaction (real-time ML inference)
```

> **Ruby Context:** Ruby services produce events to Kafka, and Karafka consumers can compute lightweight real-time aggregations using Redis as the state store. For complex analytics, a dedicated stream processor (Flink) handles the heavy lifting, and Ruby reads the aggregated results from Redis or a dashboard database.

```ruby
# Ruby example: Real-time analytics with Karafka + Redis + ActionCable

class AnalyticsConsumer < Karafka::BaseConsumer
  def consume
    redis = Redis.new

    messages.each do |message|
      event = message.payload
      timestamp = Time.parse(event['timestamp'])
      minute_bucket = timestamp.strftime('%Y-%m-%d-%H-%M')

      case event['event_type']
      when 'page_view'
        redis.hincrby("analytics:pageviews:#{minute_bucket}", event['page'], 1)
        redis.pfadd("analytics:unique_users:#{minute_bucket}", event['user_id'])
      when 'order_placed'
        redis.hincrby("analytics:orders:#{minute_bucket}", event['region'], 1)
        redis.hincrbyfloat("analytics:revenue:#{minute_bucket}", event['region'], event['total'])
      when 'error'
        redis.hincrby("analytics:errors:#{minute_bucket}", event['service'], 1)
        check_error_threshold(redis, minute_bucket, event['service'])
      end

      # Set TTL on all keys (keep 24 hours of minute-level data)
      redis.expire("analytics:pageviews:#{minute_bucket}", 86_400)
      redis.expire("analytics:unique_users:#{minute_bucket}", 86_400)
      redis.expire("analytics:orders:#{minute_bucket}", 86_400)
      redis.expire("analytics:revenue:#{minute_bucket}", 86_400)
      redis.expire("analytics:errors:#{minute_bucket}", 86_400)
    end

    # Push latest stats to live dashboard
    broadcast_dashboard_update
  end

  private

  def check_error_threshold(redis, bucket, service)
    error_count = redis.hget("analytics:errors:#{bucket}", service).to_i
    if error_count > 50  # More than 50 errors per minute
      AlertService.trigger(
        severity: :high,
        message: "High error rate for #{service}: #{error_count} errors/min"
      )
    end
  end

  def broadcast_dashboard_update
    redis = Redis.new
    bucket = Time.current.strftime('%Y-%m-%d-%H-%M')

    ActionCable.server.broadcast('analytics_dashboard', {
      timestamp: bucket,
      orders: redis.hgetall("analytics:orders:#{bucket}"),
      revenue: redis.hgetall("analytics:revenue:#{bucket}"),
      errors: redis.hgetall("analytics:errors:#{bucket}"),
      unique_users: redis.pfcount("analytics:unique_users:#{bucket}")
    })
  end
end
```

---

### Decoupling Microservices

The most fundamental use case — using message queues to decouple services so they can evolve, deploy, and scale independently.

```
Tightly Coupled (Synchronous):

  Order Service → REST → Inventory Service → REST → Payment Service → REST → Shipping Service
  
  Problems:
  - If Shipping Service is down, the entire chain fails
  - Order Service must wait for ALL downstream services (high latency)
  - Adding a new service requires changing Order Service
  - All services must be up simultaneously
  - Cascading failures: one slow service slows everything

Loosely Coupled (Event-Driven):

  Order Service → publishes "OrderCreated" → Kafka
  
  Inventory Service → subscribes to "OrderCreated" → processes independently
  Payment Service → subscribes to "OrderCreated" → processes independently
  Shipping Service → subscribes to "PaymentCompleted" → processes independently
  Analytics Service → subscribes to "OrderCreated" → processes independently
  
  Benefits:
  - If Shipping Service is down, orders still get created and paid
  - Order Service responds immediately (doesn't wait for downstream)
  - Adding Analytics Service = just subscribe to the topic (no changes to Order Service)
  - Services can be deployed independently
  - No cascading failures: each service processes at its own pace
```

**When to Use Synchronous vs Asynchronous:**

| Use Synchronous (REST/gRPC) When | Use Asynchronous (Message Queue) When |
|----------------------------------|--------------------------------------|
| Client needs an immediate response | Client doesn't need an immediate response |
| Simple request-response pattern | Fire-and-forget or eventual processing |
| Low latency required for the response | Downstream processing can be delayed |
| Few downstream services | Many downstream services (fan-out) |
| Strong consistency required | Eventual consistency is acceptable |
| Example: "Get user profile" | Example: "Send welcome email after signup" |

> **Ruby Context:** A common Rails pattern is to use synchronous calls for reads (API queries) and asynchronous messaging for writes/side effects. The controller handles the primary write, then enqueues background jobs (Sidekiq) or publishes events (Karafka) for everything else.

```ruby
# Ruby example: Decoupled microservices pattern in Rails

# BEFORE (tightly coupled — synchronous calls to all services)
class OrdersController < ApplicationController
  def create
    order = Order.create!(order_params)

    # Synchronous calls — if any fail, the whole request fails
    InventoryClient.reserve(order.items)        # HTTP call — blocks
    PaymentClient.charge(order.total)            # HTTP call — blocks
    ShippingClient.create_shipment(order)         # HTTP call — blocks
    NotificationClient.send_confirmation(order)   # HTTP call — blocks
    AnalyticsClient.track_order(order)            # HTTP call — blocks

    render json: order, status: :created
  rescue Faraday::Error => e
    # Any downstream service failure = user sees an error
    render json: { error: 'Order failed' }, status: :service_unavailable
  end
end

# AFTER (loosely coupled — publish event, services react independently)
class OrdersController < ApplicationController
  def create
    order = Order.create!(order_params)

    # Publish event — returns immediately, no blocking
    Karafka.producer.produce_sync(
      topic: 'order-events',
      payload: {
        event_type: 'OrderCreated',
        order_id: order.id,
        customer_id: order.customer_id,
        items: order.items.as_json,
        total: order.total.to_f,
        timestamp: Time.current.iso8601
      }.to_json,
      key: order.id.to_s
    )

    # Response is immediate — downstream processing happens asynchronously
    render json: { order_id: order.id, status: 'pending' }, status: :created
  end
end

# Each service consumes independently — no coupling to Order Service
# Inventory Service (separate Rails app with Karafka)
class InventoryConsumer < Karafka::BaseConsumer
  def consume
    messages.each do |message|
      event = message.payload
      next unless event['event_type'] == 'OrderCreated'
      InventoryService.reserve(event['items'])
    end
  end
end

# Analytics Service (could be a simple Sidekiq worker in the same app)
class AnalyticsTrackingWorker
  include Sidekiq::Job

  def perform(event_type, event_data)
    case event_type
    when 'OrderCreated'
      AnalyticsDB.insert(event_type: event_type, data: event_data, tracked_at: Time.current)
    end
  end
end
```

---

### Design Considerations Summary

When designing systems with message queues and event-driven architecture, consider these key decisions:

| Decision | Options | Guidance |
|----------|---------|---------|
| **Queue vs Topic** | Point-to-point vs Pub/Sub | Use queue for task distribution; topic for event broadcasting |
| **Ordering** | Strict vs best-effort | Use partition keys for per-entity ordering; accept no global order |
| **Delivery** | At-most-once vs at-least-once vs exactly-once | Default to at-least-once with idempotent consumers |
| **Retention** | Delete after consume vs time-based vs size-based | Kafka: time-based (7 days default); RabbitMQ: delete after ACK |
| **Serialization** | JSON vs Avro vs Protobuf | JSON for simplicity; Avro/Protobuf for performance and schema evolution |
| **Schema management** | Schema registry vs ad-hoc | Use a schema registry (Confluent Schema Registry) for production |
| **Error handling** | Retry + DLQ vs drop | Always use DLQ; configure retry with exponential backoff |
| **Monitoring** | Consumer lag, queue depth, error rate | Alert on growing consumer lag (consumers falling behind) |
| **Scaling** | Partitions, consumer groups | More partitions = more parallelism; consumers ≤ partitions |
| **Technology** | Kafka vs RabbitMQ vs SQS vs Pulsar | Kafka for streaming; RabbitMQ for routing; SQS for managed simplicity |

> **Ruby Context — Technology Decision Guide:**

| Scenario | Recommended Stack |
|----------|------------------|
| Single Rails app, background jobs | **Sidekiq + Redis** |
| Single Rails app, event sourcing | **Rails Event Store + PostgreSQL** |
| Multiple Ruby services, flexible routing | **Bunny/Sneakers + RabbitMQ** |
| Multiple services, high-throughput streaming | **Karafka + Kafka** |
| AWS-native, serverless-friendly | **Shoryuken + SQS/SNS** |
| Already using Redis, lightweight pub/sub | **Redis Streams** |
| Complex stream processing alongside Ruby | **Karafka (produce) + Flink/ksqlDB (process)** |

---

### Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|----------------|
| **Consumer lag** | How far behind consumers are from the latest message | Growing lag = consumers can't keep up |
| **Queue depth** | Number of messages waiting in the queue | Sustained growth = processing bottleneck |
| **Message throughput** | Messages produced/consumed per second | Sudden drop = producer or consumer issue |
| **Processing latency** | Time from message publish to consumer processing | P99 > SLA = performance issue |
| **Error rate** | Percentage of messages that fail processing | > 1% = investigate |
| **DLQ depth** | Number of messages in the dead letter queue | Any messages = investigate |
| **Broker disk usage** | Disk space used by message storage | > 80% = increase retention or add storage |
| **Replication lag** | How far behind replicas are from the leader | Growing lag = replication issue |

> **Ruby Context:** For Sidekiq, monitor queue latency (time a job waits before processing), queue size, processed/failed job counts, and retry queue depth — all visible in the **Sidekiq Web UI** or via the Sidekiq API. For Karafka, monitor consumer lag via Kafka's built-in metrics or tools like **Karafka Web UI**, **Burrow**, or **Kafka Manager**. Export metrics to **Prometheus** + **Grafana** for dashboards and alerting.

```ruby
# Ruby example: Monitoring Sidekiq metrics with Prometheus
# Using the yabeda-sidekiq gem for automatic metric collection

# Gemfile
# gem 'yabeda-sidekiq'
# gem 'yabeda-prometheus'

# config/initializers/yabeda.rb
Yabeda.configure do
  group :sidekiq do
    gauge :queue_size, comment: 'Current queue size', tags: [:queue]
    gauge :retry_size, comment: 'Current retry queue size'
    gauge :dead_size, comment: 'Current dead set size'
    gauge :latency, comment: 'Queue latency in seconds', tags: [:queue]
  end
end

# Custom health check endpoint
class HealthController < ApplicationController
  def sidekiq_status
    stats = Sidekiq::Stats.new
    queues = Sidekiq::Queue.all.map do |q|
      { name: q.name, size: q.size, latency: q.latency.round(2) }
    end

    status = {
      processed: stats.processed,
      failed: stats.failed,
      retry_size: stats.retry_size,
      dead_size: stats.dead_size,
      queues: queues,
      healthy: queues.all? { |q| q[:latency] < 30 }  # Alert if any queue > 30s latency
    }

    render json: status, status: status[:healthy] ? :ok : :service_unavailable
  end
end
```

---

*This module covered the complete landscape of message queues and event-driven architecture — from fundamental concepts (producers, consumers, delivery semantics) through deep dives into Kafka and RabbitMQ, to advanced patterns (Event Sourcing, CQRS, Saga) and stream processing. Ruby's ecosystem provides excellent tooling at every level: Sidekiq for background jobs, Bunny/Sneakers for RabbitMQ, Karafka for Kafka, Rails Event Store for event sourcing and CQRS, and Wisper for in-process pub/sub. These concepts are essential for designing scalable, resilient distributed systems and are frequently tested in system design interviews.*