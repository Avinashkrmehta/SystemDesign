# Module 25: Microservices Architecture

> Microservices architecture structures an application as a collection of small, autonomous services, each owning its own data and deployable independently. It's the dominant architecture for large-scale systems at companies like Netflix, Amazon, Uber, and Spotify. But microservices come with significant complexity — distributed transactions, service discovery, inter-service communication, and operational overhead. This module covers when and how to use microservices, the patterns that make them work, and the pitfalls that make them fail.

---

## 25.1 Microservices Fundamentals

> A **microservice** is a small, independently deployable service that focuses on a single business capability. Microservices communicate over the network (HTTP, gRPC, messaging) and each owns its own data store. The key idea: **organize around business capabilities, not technical layers**.

---

### Monolith vs Microservices

```
Monolith:                              Microservices:
+----------------------------+         +--------+ +--------+ +--------+
|  User Module               |         | User   | | Order  | | Payment|
|  Order Module              |         | Service| | Service| | Service|
|  Payment Module            |         +---+----+ +---+----+ +---+----+
|  Notification Module       |             |          |          |
|  Search Module             |         +---+----+ +---+----+ +---+----+
|                            |         | User   | | Order  | | Payment|
|  Single Database           |         | DB     | | DB     | | DB     |
+----------------------------+         +--------+ +--------+ +--------+
  One deployment unit                  Independent deployment units
  One database                         Database per service
  In-process function calls            Network calls (HTTP, gRPC, events)
```

**Detailed Comparison:**

| Aspect | Monolith | Microservices |
|--------|---------|---------------|
| **Deployment** | Single unit — deploy everything together | Independent — deploy each service separately |
| **Scaling** | Scale entire application | Scale individual services based on their load |
| **Technology** | One language/framework for everything | Each service can use different tech (polyglot) |
| **Data** | Single shared database | Database per service (data isolation) |
| **Communication** | In-process function calls (nanoseconds) | Network calls — HTTP, gRPC, messaging (milliseconds) |
| **Transactions** | ACID transactions across all modules | Distributed transactions (saga pattern, eventual consistency) |
| **Team structure** | One team or multiple teams on one codebase | Small autonomous teams per service (2-pizza teams) |
| **Complexity** | Simple architecture, complex codebase (as it grows) | Complex architecture, simpler individual codebases |
| **Testing** | End-to-end tests are straightforward | Integration testing across services is hard |
| **Debugging** | Single process — stack traces show full flow | Distributed tracing needed (Jaeger, Zipkin) |
| **Failure** | One bug can crash everything | Failure is isolated to one service (if designed well) |
| **Latency** | Low (in-process calls) | Higher (network hops between services) |
| **Operational cost** | Low (one thing to deploy, monitor, maintain) | High (dozens of services to deploy, monitor, maintain) |

---

### Characteristics of Microservices

| Characteristic | Description |
|---------------|-------------|
| **Single responsibility** | Each service does one thing well — one business capability |
| **Independently deployable** | Deploy, update, and scale each service without affecting others |
| **Owns its data** | Each service has its own database — no shared database |
| **Loosely coupled** | Services interact through well-defined APIs; internal changes don't affect others |
| **Organized around business capabilities** | Not technical layers (UI, logic, data) but business domains (users, orders, payments) |
| **Small team ownership** | Each service is owned by a small team (2-pizza team: 6-10 people) |
| **Decentralized governance** | Teams choose their own technology, tools, and processes |
| **Design for failure** | Assume any service can fail at any time; build resilience into every interaction |
| **Observable** | Centralized logging, metrics, and distributed tracing across all services |
| **Automated** | CI/CD pipelines, automated testing, infrastructure as code for each service |

---

### Bounded Context (Domain-Driven Design)

A **bounded context** is a central pattern in Domain-Driven Design (DDD) that defines the boundary within which a particular domain model applies. Each microservice should align with one bounded context.

```
E-Commerce Domain:

  +------------------+    +------------------+    +------------------+
  | User Context     |    | Order Context    |    | Payment Context  |
  |                  |    |                  |    |                  |
  | - User           |    | - Order          |    | - Payment        |
  | - Profile        |    | - OrderItem      |    | - Transaction    |
  | - Address        |    | - Cart           |    | - Refund         |
  | - Preferences    |    | - Shipping       |    | - Invoice        |
  |                  |    |                  |    |                  |
  | "User" = account |    | "User" = buyer   |    | "User" = payer   |
  | details, prefs   |    | with shipping    |    | with payment     |
  |                  |    | address          |    | method           |
  +------------------+    +------------------+    +------------------+
  
  The same real-world concept ("User") has DIFFERENT meanings in each context.
  Each context has its own model of "User" with only the attributes it needs.
```

**Key DDD Concepts for Microservices:**

| Concept | Description | Example |
|---------|-------------|---------|
| **Bounded Context** | A boundary where a domain model is consistent and complete | Order Service has its own model of "Product" (id, name, price) — not the full product catalog |
| **Ubiquitous Language** | Shared vocabulary within a bounded context | "Order" means different things in Order Context vs Shipping Context |
| **Aggregate** | A cluster of domain objects treated as a single unit for data changes | Order + OrderItems = Order Aggregate (always consistent together) |
| **Aggregate Root** | The entry point to an aggregate; external references only point to the root | Order is the root; you never access OrderItem directly from outside |
| **Domain Event** | Something that happened in the domain that other contexts care about | "OrderPlaced", "PaymentCompleted", "ItemShipped" |
| **Anti-Corruption Layer** | A translation layer between two bounded contexts with different models | Order Service translates its "Product" model when calling Product Service |

**How to Identify Bounded Contexts:**
1. Look at the **organizational structure** — teams often align with bounded contexts
2. Identify **different meanings** of the same term across the business
3. Find **natural boundaries** where data models diverge
4. Look for areas that **change independently** and at different rates
5. Use **event storming** — workshop technique to discover domain events and boundaries

---

### Single Responsibility per Service

Each microservice should have a **single reason to change** — it owns one business capability completely.

**Good Service Boundaries:**

| Service | Responsibility | Owns |
|---------|---------------|------|
| User Service | User registration, authentication, profile management | Users table, auth tokens |
| Order Service | Order creation, status tracking, order history | Orders table, order items |
| Payment Service | Payment processing, refunds, invoicing | Payments table, transactions |
| Inventory Service | Stock management, reservations, restocking | Inventory table |
| Notification Service | Email, SMS, push notifications | Notification templates, delivery logs |
| Search Service | Product search, autocomplete, filtering | Search index (Elasticsearch) |
| Recommendation Service | Personalized recommendations | ML models, user behavior data |

**Bad Service Boundaries (too granular):**

| Service | Problem |
|---------|---------|
| EmailService, SMSService, PushService | Too fine-grained — should be one Notification Service |
| OrderCreationService, OrderStatusService | Same domain — should be one Order Service |
| UserReadService, UserWriteService | CQRS is an internal pattern, not a service boundary |

**Bad Service Boundaries (too broad):**

| Service | Problem |
|---------|---------|
| BackendService | That's a monolith with a different name |
| DataService | Too generic — what data? Which domain? |
| APIService | Technical layer, not business capability |

**Rule of Thumb:** A microservice should be small enough that a single team (6-10 people) can own it completely, but large enough that it represents a meaningful business capability. If two services always change together and always deploy together, they should probably be one service.

---

### Independent Deployment

The most important benefit of microservices: each service can be deployed **independently** without coordinating with other teams or services.

```
Monolith Deployment:
  Change in Payment module → must deploy ENTIRE application
  → Requires coordination with User team, Order team, Search team
  → Risk: a bug in Payment breaks User functionality
  → Deployment frequency: weekly or monthly (slow, risky)

Microservice Deployment:
  Change in Payment Service → deploy ONLY Payment Service
  → No coordination needed with other teams
  → Risk: isolated to Payment Service
  → Deployment frequency: multiple times per day (fast, low risk)
```

**Requirements for Independent Deployment:**
- **Backward-compatible APIs** — new version must work with old clients
- **Database per service** — no shared schema that requires coordinated migrations
- **Feature flags** — deploy code without activating it; activate when ready
- **Contract testing** — verify that API changes don't break consumers
- **CI/CD per service** — each service has its own build, test, and deploy pipeline
- **Canary deployments** — roll out to a small percentage of traffic first

---


## 25.2 Communication Patterns

> Microservices communicate over the network, not through in-process function calls. The choice between synchronous and asynchronous communication fundamentally affects system behavior — latency, coupling, reliability, and complexity.

---

### Synchronous Communication (REST, gRPC)

The caller sends a request and **waits** for a response before continuing.

```
Synchronous (request-response):
  Order Service → POST /payments (REST) → Payment Service
  Order Service waits... (blocked)
  Payment Service processes... (100ms)
  Payment Service → 200 OK → Order Service continues
```

**REST (HTTP/JSON):**

| Pros | Cons |
|------|------|
| Simple, well-understood | Tight coupling (caller depends on callee being available) |
| Easy to debug (curl, Postman) | Cascading failures (if Payment is down, Order is stuck) |
| HTTP caching | Higher latency (network round trip per call) |
| Universal (every language has HTTP client) | Synchronous blocking reduces throughput |

**gRPC (HTTP/2 + Protobuf):**

| Pros | Cons |
|------|------|
| 3-10x smaller payloads (binary protobuf) | Harder to debug (binary format) |
| Strong typing with code generation | Requires protobuf tooling |
| Bidirectional streaming | Not browser-friendly (needs grpc-web proxy) |
| Built-in deadline propagation | More complex than REST |
| Lower latency than REST | Tighter coupling than async |

**When to Use Synchronous:**
- The caller **needs the response** to continue (e.g., Order Service needs payment confirmation)
- Simple request-response patterns
- Low-latency requirements where async overhead isn't justified
- Internal service-to-service calls with circuit breakers

---

### Asynchronous Communication (Message Queues, Events)

The caller sends a message and **doesn't wait** for a response. The message is processed later by the consumer.

```
Asynchronous (event-driven):
  Order Service → publish "OrderCreated" event → Message Queue (Kafka)
  Order Service continues immediately (not blocked!)
  
  Payment Service → subscribes to "OrderCreated" → processes payment
  Inventory Service → subscribes to "OrderCreated" → reserves stock
  Notification Service → subscribes to "OrderCreated" → sends confirmation email
```

```
  +--------+     Publish      +---------------+     Subscribe    +----------+
  | Order  | ───────────────→ | Message Queue | ───────────────→ | Payment  |
  | Service|   "OrderCreated" | (Kafka/SQS)   |                  | Service  |
  +--------+                  +---------------+ ───────────────→ +----------+
  (continues                                                     | Inventory|
   immediately)                                                  | Service  |
                                                                 +----------+
                                                   ───────────→ | Notif.   |
                                                                 | Service  |
                                                                 +----------+
```

**Message Queue (Point-to-Point):**
One producer, one consumer. Each message is processed by exactly one consumer.

```
Producer → Queue → Consumer A (processes message)
                   Consumer B (idle — message already consumed by A)
```

**Pub/Sub (Publish-Subscribe):**
One producer, multiple subscribers. Each subscriber gets a copy of every message.

```
Producer → Topic → Subscriber A (gets a copy)
                 → Subscriber B (gets a copy)
                 → Subscriber C (gets a copy)
```

| Pros | Cons |
|------|------|
| Loose coupling (producer doesn't know about consumers) | More complex architecture |
| Resilience (if consumer is down, messages queue up) | Eventual consistency (not immediate) |
| Scalability (add more consumers to handle load) | Harder to debug (async flow, message ordering) |
| Natural load leveling (queue absorbs traffic spikes) | Message delivery guarantees add complexity |
| One event can trigger multiple actions (fan-out) | Monitoring and observability are harder |

**When to Use Asynchronous:**
- The caller **doesn't need an immediate response** (notifications, analytics, logging)
- Multiple services need to react to the same event (fan-out)
- You need to **decouple** services (producer doesn't depend on consumer availability)
- You need to **absorb traffic spikes** (queue buffers messages)
- Long-running operations (video processing, report generation)

---

### Request-Reply Pattern (Async with Response)

Sometimes you need async communication but still need a response. The **request-reply pattern** sends a request via a message queue and receives the response on a separate reply queue.

```
Order Service → Request Queue: "Process payment for order 123"
                                (includes reply_to: "order-replies" and correlation_id: "abc")

Payment Service → picks up message → processes payment
Payment Service → Reply Queue ("order-replies"): "Payment successful" (correlation_id: "abc")

Order Service → picks up reply from "order-replies" → matches correlation_id "abc" → continues
```

**Use Case:** When you want the decoupling benefits of async but still need a response (e.g., payment processing where you need confirmation but don't want tight coupling).

---

### Choreography vs Orchestration

Two approaches to coordinating multi-service workflows:

**Choreography (Decentralized — Event-Driven):**

Each service reacts to events and publishes its own events. No central coordinator.

```
1. Order Service → publishes "OrderCreated"
2. Payment Service → hears "OrderCreated" → charges payment → publishes "PaymentCompleted"
3. Inventory Service → hears "PaymentCompleted" → reserves stock → publishes "StockReserved"
4. Shipping Service → hears "StockReserved" → creates shipment → publishes "ShipmentCreated"
5. Notification Service → hears "ShipmentCreated" → sends tracking email

Each service knows only about the events it cares about.
No service knows the full workflow.
```

**Orchestration (Centralized — Coordinator):**

A central **orchestrator** service directs the workflow, telling each service what to do.

```
Order Orchestrator:
  1. → Order Service: "Create order" → receives confirmation
  2. → Payment Service: "Charge $99.99" → receives confirmation
  3. → Inventory Service: "Reserve items" → receives confirmation
  4. → Shipping Service: "Create shipment" → receives confirmation
  5. → Notification Service: "Send confirmation email"

The orchestrator knows the full workflow.
Each service is a simple worker that does what it's told.
```

**Comparison:**

| Feature | Choreography | Orchestration |
|---------|-------------|---------------|
| **Coordination** | Decentralized (events) | Centralized (orchestrator) |
| **Coupling** | Very loose (services don't know about each other) | Tighter (orchestrator knows all services) |
| **Visibility** | Hard to see the full workflow | Easy to see the full workflow in one place |
| **Adding steps** | Add a new subscriber (no changes to existing services) | Modify the orchestrator |
| **Error handling** | Each service handles its own errors; compensating events | Orchestrator handles errors centrally; compensating commands |
| **Debugging** | Hard (follow events across services, distributed tracing) | Easier (orchestrator has full state) |
| **Single point of failure** | None | Orchestrator is a SPOF (must be highly available) |
| **Complexity** | Grows with number of services and events | Grows with workflow complexity |
| **Best for** | Simple workflows (2-4 steps), high decoupling | Complex workflows (5+ steps), branching logic, error handling |

**Practical Recommendation:**
- Start with **choreography** for simple, linear workflows
- Switch to **orchestration** when workflows become complex (branching, error handling, timeouts)
- Many systems use **both** — orchestration for the main workflow, choreography for side effects (notifications, analytics)

---

### API Gateway Pattern

An **API Gateway** is the single entry point for all external client requests. It routes requests to the appropriate microservice, handles cross-cutting concerns, and can aggregate responses.

```
Without API Gateway:
  Mobile App → User Service (port 8001)
  Mobile App → Order Service (port 8002)
  Mobile App → Payment Service (port 8003)
  (Client must know about every service and its address)

With API Gateway:
  Mobile App → API Gateway (single endpoint)
    /api/users/*    → User Service
    /api/orders/*   → Order Service
    /api/payments/* → Payment Service
  (Client knows only the gateway)
```

**API Gateway Responsibilities:**

| Responsibility | Description |
|---------------|-------------|
| **Routing** | Route requests to the correct service based on path, headers, method |
| **Authentication** | Validate JWT/API keys once at the gateway |
| **Rate limiting** | Enforce per-user/per-API rate limits |
| **SSL termination** | Handle HTTPS at the gateway; internal traffic can be HTTP |
| **Response aggregation** | Combine responses from multiple services into one |
| **Protocol translation** | External REST → internal gRPC |
| **Caching** | Cache responses for frequently requested data |
| **Logging/Monitoring** | Centralized request logging and metrics |
| **Circuit breaking** | Protect backend services from cascading failures |

Covered in depth in Module 16.4.

---


## 25.3 Service Discovery

> In a microservices architecture, services are deployed across multiple servers, containers, or pods. Their IP addresses and ports change dynamically (auto-scaling, deployments, failures). **Service discovery** is the mechanism by which services find each other's network locations.

---

### The Problem

```
Static Configuration (doesn't work in dynamic environments):
  Order Service config: payment_service_url = "http://10.0.1.5:8080"
  
  What happens when:
  - Payment Service scales to 3 instances? (which IP?)
  - Payment Service is redeployed? (new IP!)
  - Payment Service instance crashes? (stale IP!)
  - Payment Service moves to a different host? (wrong IP!)
```

---

### Client-Side Discovery

The **client** queries a **service registry** to get the list of available instances, then selects one using a load balancing algorithm.

```
1. Payment Service instances register with Service Registry:
   Registry: payment-service → [10.0.1.5:8080, 10.0.1.6:8080, 10.0.1.7:8080]

2. Order Service queries the registry:
   Order Service → Registry: "Where is payment-service?"
   Registry → Order Service: [10.0.1.5:8080, 10.0.1.6:8080, 10.0.1.7:8080]

3. Order Service selects one (client-side load balancing):
   Order Service → 10.0.1.6:8080 (round robin / least connections)
```

```
  +--------+     Register     +------------------+
  |Payment | ───────────────→ | Service Registry |
  |Service | (on startup)     | (Consul, Eureka) |
  |Instance|                  +--------+---------+
  +--------+                           |
                                       | Query
  +--------+                  +--------v---------+
  | Order  | ←── Instances ── | Service Registry |
  | Service|                  +------------------+
  +---+----+
      |
      | Direct call (client picks one)
      ↓
  +--------+
  |Payment |
  |Instance|
  +--------+
```

| Pros | Cons |
|------|------|
| No single point of failure for routing | Client must implement discovery + LB logic |
| Client can make smart routing decisions | Every client language needs a discovery library |
| Lower latency (direct connection) | Client cache of instances can become stale |

---

### Server-Side Discovery

The client sends requests to a **load balancer** or **router**, which queries the service registry and routes the request.

```
1. Payment Service instances register with Service Registry
2. Order Service sends request to Load Balancer / Router
3. Load Balancer queries Registry → gets instance list → routes request

  +--------+     Request      +---------------+     Route      +--------+
  | Order  | ───────────────→ | Load Balancer | ────────────→ | Payment|
  | Service|                  | / Router      |               | Service|
  +--------+                  +-------+-------+               +--------+
                                      |
                              +-------v--------+
                              | Service Registry|
                              +----------------+
```

| Pros | Cons |
|------|------|
| Client is simple (just sends to LB) | Load balancer is a single point of failure (must be HA) |
| No discovery logic in client code | Extra network hop (client → LB → service) |
| Centralized routing decisions | LB must be kept in sync with registry |

**Kubernetes uses server-side discovery by default:** Services are accessed via a stable DNS name (e.g., `payment-service.default.svc.cluster.local`), and kube-proxy routes to a healthy pod.

---

### Service Registry

The **service registry** is a database of available service instances and their network locations.

| Registry | Type | Key Features |
|----------|------|-------------|
| **Consul** (HashiCorp) | CP (consistent) | Service discovery, health checks, KV store, service mesh (Connect) |
| **Eureka** (Netflix) | AP (available) | Service discovery for Spring Cloud; self-preservation mode |
| **etcd** | CP (consistent) | Distributed KV store; used by Kubernetes for cluster state |
| **ZooKeeper** | CP (consistent) | Distributed coordination; used by Kafka, HBase, Hadoop |
| **Kubernetes DNS** | Built-in | Automatic DNS for services and pods; no external registry needed |

**Service Registration Patterns:**

| Pattern | How It Works | Example |
|---------|-------------|---------|
| **Self-registration** | Service registers itself on startup, deregisters on shutdown | Spring Cloud + Eureka |
| **Third-party registration** | A separate registrar watches for new instances and registers them | Kubernetes (kubelet registers pods), Consul agent |

**Health Check Integration:**
The registry must know if an instance is healthy. Two approaches:
- **Heartbeat:** Instance sends periodic heartbeats; registry removes it if heartbeats stop
- **Health check:** Registry (or agent) actively checks the instance's health endpoint

---

### DNS-Based Discovery

Use DNS to resolve service names to IP addresses. The simplest form of service discovery.

```
Order Service → DNS query: "payment-service.internal"
DNS → A records: [10.0.1.5, 10.0.1.6, 10.0.1.7]
Order Service → connects to 10.0.1.5 (first record)
```

**Kubernetes DNS:**
```
Service name: payment-service
Namespace: default

DNS name: payment-service.default.svc.cluster.local
→ Resolves to the ClusterIP (virtual IP)
→ kube-proxy routes to a healthy pod (iptables/IPVS rules)
```

| Pros | Cons |
|------|------|
| Simple — every language has DNS support | DNS TTL caching can serve stale IPs |
| No special client library needed | Limited load balancing (DNS round-robin) |
| Works everywhere | No health-check-aware routing (basic DNS) |
| Kubernetes DNS is automatic | Can't route based on load or latency |

**Key Point:** In Kubernetes, DNS-based discovery is usually sufficient. Outside Kubernetes, use Consul or a similar service registry for dynamic environments.

---


## 25.4 Data Management

> Data management is the **hardest problem** in microservices. Each service owning its own database means no cross-service JOINs, no cross-service ACID transactions, and data duplication across services. This section covers the patterns that make it work.

---

### Database per Service Pattern

Each microservice has its own database that only it can access. Other services must go through the service's API.

```
✅ Correct (Database per Service):
  Order Service → Order DB (only Order Service reads/writes)
  Payment Service → Payment DB (only Payment Service reads/writes)
  
  Order Service needs payment info? → calls Payment Service API

❌ Wrong (Shared Database):
  Order Service → Shared DB ← Payment Service
  (Both services read/write the same tables — tight coupling!)
```

**Benefits:**
- Services are truly independent (different schemas, different databases, different technologies)
- Schema changes in one service don't affect others
- Each service can choose the best database for its needs (polyglot persistence)
- Failure isolation — one database going down doesn't affect other services

**Challenges:**
- No cross-service JOINs — must use API calls or data replication
- No cross-service transactions — must use Saga pattern
- Data duplication — some data exists in multiple services
- Queries spanning multiple services are complex (API composition or CQRS)

---

### Shared Database (Anti-Pattern)

Multiple services sharing the same database is an **anti-pattern** because it creates tight coupling.

```
❌ Shared Database Problems:
  Order Service → reads users table directly
  User Service → owns users table
  
  If User Service changes the users table schema:
  → Order Service breaks!
  → Must coordinate deployments between teams
  → Defeats the purpose of microservices
```

**Why It's an Anti-Pattern:**
- **Tight coupling** — services depend on each other's schema
- **No independent deployment** — schema changes require coordinated releases
- **No technology freedom** — all services must use the same database
- **Scaling bottleneck** — one database for all services
- **Ownership ambiguity** — who owns the schema? Who can change it?

**Exception:** During migration from monolith to microservices, a shared database is a temporary stepping stone. Extract services one at a time, migrating their data to separate databases.

---

### Saga Pattern for Distributed Transactions

When a business operation spans multiple services, you can't use a traditional ACID transaction. The **Saga pattern** breaks it into a sequence of local transactions with compensating actions. Covered in depth in Module 22.5.

**Quick Recap:**

```
Order Saga (happy path):
  1. Order Service: Create order (PENDING)
  2. Payment Service: Charge payment
  3. Inventory Service: Reserve stock
  4. Order Service: Confirm order (CONFIRMED)

Order Saga (payment fails):
  1. Order Service: Create order (PENDING)
  2. Payment Service: Charge fails!
  3. Compensate: Order Service → Cancel order (CANCELLED)
```

**Saga Execution Coordinator (SEC):**
A dedicated component that tracks the saga state and triggers compensating actions on failure.

```
Saga State Machine:
  STARTED → PAYMENT_PENDING → PAYMENT_COMPLETED → STOCK_RESERVED → CONFIRMED
                             → PAYMENT_FAILED → COMPENSATING → CANCELLED
```

---

### Event Sourcing and CQRS

Both covered in depth in Module 22.5. In the microservices context:

**Event Sourcing** enables services to communicate through domain events:
```
Order Service → publishes "OrderCreated" event
Payment Service → subscribes → processes payment → publishes "PaymentCompleted"
Inventory Service → subscribes → reserves stock → publishes "StockReserved"
```

**CQRS** enables efficient cross-service queries:
```
Write: Each service writes to its own database
Read:  A dedicated read service builds a denormalized view from events across all services

Example: "Order Dashboard" needs data from Order, Payment, and Shipping services
  → Each service publishes events
  → Dashboard Service subscribes to all events
  → Builds a denormalized read model with all the data it needs
  → No cross-service API calls for reads!
```

---

### Data Consistency Strategies

| Strategy | Consistency | Complexity | Use Case |
|----------|-----------|-----------|----------|
| **Synchronous API calls** | Strong (if using transactions) | Low | Simple, low-scale operations |
| **Saga pattern** | Eventual | High | Multi-service business transactions |
| **Event-driven data replication** | Eventual | Medium | Services need read access to other services' data |
| **CQRS with event sourcing** | Eventual | High | Complex read patterns across services |
| **Change Data Capture (CDC)** | Eventual | Medium | Replicate data changes from DB to other services |
| **API composition** | Strong (at query time) | Medium | Aggregate data from multiple services for a single response |

**Change Data Capture (CDC):**
Capture changes from a service's database and publish them as events — without modifying the service's code.

```
Order Service → writes to Order DB
                     ↓
              Debezium (CDC) → reads DB change log (WAL/binlog)
                     ↓
              Kafka → "order_changes" topic
                     ↓
              Analytics Service → consumes changes → updates analytics DB
              Search Service → consumes changes → updates search index
```

**CDC Tools:** Debezium (open-source, Kafka-based), AWS DMS, Maxwell (MySQL), Bottled Water (PostgreSQL)

**Key Point:** There is no silver bullet for data consistency in microservices. Accept eventual consistency for most operations. Use sagas for critical multi-service transactions. Use CDC for data replication without coupling services.

---


## 25.5 Resilience Patterns

> In a microservices architecture, **any service can fail at any time**. Network calls fail, services crash, databases become slow. Resilience patterns prevent a single failure from cascading through the entire system.

---

### Circuit Breaker

The **circuit breaker** pattern prevents a service from repeatedly calling a failing downstream service. It "trips" after a threshold of failures and returns a fallback response immediately — giving the failing service time to recover.

**Three States:**

```
+----------+     Failure threshold     +--------+     Timeout      +-----------+
|  CLOSED  | ────────────────────────→ |  OPEN  | ──────────────→ | HALF-OPEN |
| (normal) |                           | (fail  |                  | (testing) |
|          | ←──────────────────────── | fast)  | ←────────────── |           |
+----------+     Reset (success)       +--------+   Failure        +-----------+
                                                     (back to OPEN)     |
                                                                        | Success
                                                                        ↓
                                                                   +----------+
                                                                   |  CLOSED  |
                                                                   | (normal) |
                                                                   +----------+
```

| State | Behavior |
|-------|----------|
| **CLOSED** | Normal operation. Requests pass through. Failures are counted. |
| **OPEN** | All requests **fail immediately** with a fallback response. No calls to the downstream service. Timer starts. |
| **HALF-OPEN** | After the timer expires, allow **one test request** through. If it succeeds → CLOSED. If it fails → OPEN again. |

**Configuration Parameters:**

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| Failure threshold | Number of failures before opening the circuit | 5-10 failures |
| Failure rate threshold | Percentage of failures before opening | 50% |
| Timeout (open duration) | How long to stay OPEN before trying HALF-OPEN | 30-60 seconds |
| Half-open max requests | Number of test requests in HALF-OPEN state | 1-3 |
| Sliding window | Time window for counting failures | 10-60 seconds |

**Example Flow:**

```
Request 1 → Payment Service → 200 OK (success, circuit CLOSED)
Request 2 → Payment Service → 200 OK (success)
Request 3 → Payment Service → 500 Error (failure count: 1)
Request 4 → Payment Service → Timeout (failure count: 2)
Request 5 → Payment Service → 500 Error (failure count: 3)
Request 6 → Payment Service → Connection Refused (failure count: 4)
Request 7 → Payment Service → Timeout (failure count: 5 → THRESHOLD!)

Circuit OPENS:
Request 8 → Circuit breaker → return fallback immediately (no call to Payment Service)
Request 9 → Circuit breaker → return fallback immediately
... (for 30 seconds)

Circuit goes HALF-OPEN:
Request 100 → Payment Service → 200 OK (test request succeeds!)
Circuit CLOSES → normal operation resumes
```

**Fallback Strategies:**

| Strategy | Description | Example |
|----------|-------------|---------|
| **Default value** | Return a sensible default | Return cached product price |
| **Cached response** | Return the last successful response | Return cached user profile |
| **Degraded response** | Return partial data | Show product without reviews |
| **Error response** | Return a clear error to the client | "Payment service temporarily unavailable" |
| **Alternative service** | Call a backup service | Use a secondary payment provider |
| **Queue for later** | Queue the request for retry | Queue the payment for processing when service recovers |

---

### Retry with Exponential Backoff

Automatically retry failed requests with increasing delays between attempts.

```
Attempt 1: Call Payment Service → Timeout
  Wait 1 second
Attempt 2: Call Payment Service → 503 Service Unavailable
  Wait 2 seconds
Attempt 3: Call Payment Service → 503 Service Unavailable
  Wait 4 seconds
Attempt 4: Call Payment Service → 200 OK (success!)

Exponential backoff: delay = base_delay × 2^(attempt - 1)
With jitter:         delay = base_delay × 2^(attempt - 1) + random(0, base_delay)
```

**Why Jitter Matters:**
Without jitter, all clients retry at the exact same time → thundering herd → service overloaded again.
With jitter, retries are spread out → gradual recovery.

**Retry Configuration:**

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| Max retries | Maximum number of retry attempts | 3-5 |
| Base delay | Initial delay before first retry | 100ms - 1s |
| Max delay | Maximum delay (cap the exponential growth) | 30-60 seconds |
| Jitter | Random variation added to delay | 0 to base_delay |
| Retryable errors | Which errors trigger a retry | 429, 500, 502, 503, 504, timeouts, connection errors |
| Non-retryable errors | Which errors should NOT be retried | 400, 401, 403, 404 (client errors — retrying won't help) |

**Important:** Only retry **idempotent** operations. Retrying a non-idempotent POST could create duplicate orders or double-charge a payment.

---

### Timeout

Set a **maximum time** to wait for a response. If the timeout expires, fail the request and move on.

```
Without timeout:
  Order Service → Payment Service (stuck, never responds)
  Order Service thread is blocked FOREVER
  → Thread pool exhausted → Order Service can't handle any requests → cascading failure!

With timeout:
  Order Service → Payment Service (timeout: 5 seconds)
  After 5 seconds → timeout error → Order Service returns error to client
  Thread is freed → Order Service continues handling other requests
```

**Timeout Types:**

| Type | Description | Typical Value |
|------|-------------|---------------|
| **Connection timeout** | Max time to establish a TCP connection | 1-5 seconds |
| **Read timeout** | Max time to wait for a response after connection is established | 5-30 seconds |
| **Overall timeout** | Max time for the entire operation (including retries) | 10-60 seconds |

**Deadline Propagation (gRPC):**
In a chain of service calls, the deadline propagates — each service knows how much time is left.

```
Client sets deadline: 10 seconds
  → API Gateway (1s used, 9s remaining)
    → Order Service (2s used, 7s remaining)
      → Payment Service (must complete within 7s)
```

---

### Bulkhead

The **bulkhead** pattern isolates different parts of the system so that a failure in one part doesn't bring down the entire system. Named after the watertight compartments in a ship's hull.

```
Without Bulkhead:
  Order Service has 100 threads in a single thread pool
  Payment Service becomes slow → 90 threads stuck waiting for Payment
  → Only 10 threads left for ALL other operations
  → Order Service effectively down for everything!

With Bulkhead:
  Order Service has separate thread pools:
    Payment calls: 30 threads (max)
    Inventory calls: 30 threads (max)
    User calls: 20 threads (max)
    Other: 20 threads (max)
  
  Payment Service becomes slow → 30 Payment threads stuck
  → 70 threads still available for Inventory, User, and other operations!
  → Order Service continues working for non-payment operations
```

**Bulkhead Types:**

| Type | Description | Example |
|------|-------------|---------|
| **Thread pool isolation** | Separate thread pools for different downstream services | Hystrix-style thread pool per service |
| **Semaphore isolation** | Limit concurrent calls to a service using a semaphore | Max 50 concurrent calls to Payment Service |
| **Process isolation** | Run different workloads in separate processes/containers | Separate pods for API and background workers |
| **Infrastructure isolation** | Separate infrastructure for different tenants or workloads | Separate Redis clusters for cache vs sessions |

---

### Fallback

Provide an **alternative response** when the primary operation fails. Often used in combination with circuit breaker and timeout.

```
try:
    product = product_service.get_product(id)
except (Timeout, CircuitOpen, ServiceUnavailable):
    product = cache.get(f"product:{id}")       # fallback: cached version
    if not product:
        product = default_product_response(id)  # fallback: default values
    product.is_stale = True                     # flag that data may be stale
return product
```

---

### Rate Limiting

Protect services from being overwhelmed by too many requests. Covered in Module 16.1 (rate limiting headers) and Module 14 (rate limiter design).

**Rate Limiting in Microservices:**

```
Client → API Gateway (rate limit: 1000 req/min per user)
  → Order Service (rate limit: 500 req/min to Payment Service)
    → Payment Service (rate limit: 100 req/min to external payment provider)
```

Each layer can have its own rate limits:
- **API Gateway:** Per-user or per-API-key limits (protect the system from external abuse)
- **Service-to-service:** Per-service limits (prevent one service from overwhelming another)
- **External API calls:** Per-provider limits (respect third-party API rate limits)

---

### Resilience Pattern Combination

In practice, these patterns are used **together**:

```
Order Service calling Payment Service:

  1. Rate Limiter: Check if we're within the rate limit for Payment Service
     → If exceeded: return 429 (don't even try)
  
  2. Circuit Breaker: Check if the circuit is open
     → If open: return fallback immediately (don't even try)
  
  3. Bulkhead: Check if the Payment thread pool has available threads
     → If full: return 503 (don't queue, fail fast)
  
  4. Timeout: Set a 5-second timeout for the call
  
  5. Retry: If the call fails with a retryable error, retry up to 3 times
     with exponential backoff + jitter
  
  6. Fallback: If all retries fail, return a fallback response
```

**Resilience Libraries:**

| Library | Language | Features |
|---------|----------|----------|
| **Resilience4j** | Java | Circuit breaker, retry, rate limiter, bulkhead, time limiter |
| **Polly** | .NET | Circuit breaker, retry, timeout, bulkhead, fallback |
| **Hystrix** (Netflix) | Java | Circuit breaker, bulkhead (deprecated — use Resilience4j) |
| **go-kit** | Go | Circuit breaker, rate limiter, retry |
| **Envoy** (service mesh) | Any | Circuit breaker, retry, timeout, rate limiting (infrastructure-level) |

---


## 25.6 Service Mesh

> A **service mesh** is a dedicated infrastructure layer for handling service-to-service communication. It uses **sidecar proxies** deployed alongside each service instance to handle networking concerns — load balancing, retries, circuit breaking, mTLS, and observability — without changing application code.

---

### Sidecar Proxy Pattern

Every service instance gets a **sidecar proxy** (typically Envoy) that intercepts all inbound and outbound network traffic.

```
Without Service Mesh:
  Service A → (application code handles retries, circuit breaking, TLS) → Service B
  Every service must implement resilience patterns in its own code.

With Service Mesh:
  Service A → Sidecar Proxy A → Sidecar Proxy B → Service B
  The sidecar handles ALL networking concerns.
  Service A and B just make plain HTTP/gRPC calls.
```

```
+------------------+          +------------------+
| Pod A            |          | Pod B            |
| +------+ +----+ |          | +----+ +------+  |
| |Svc A | |Envoy|──── mTLS ───|Envoy| |Svc B |  |
| |      | |Proxy| |          | |Proxy| |      |  |
| +------+ +----+ |          | +----+ +------+  |
+------------------+          +------------------+
     ↑                              ↑
     |         +------------+       |
     +─────────| Control    |───────+
               | Plane      |
               | (istiod)   |
               +------------+
               Pushes config to all sidecars:
               - Routing rules
               - Retry policies
               - Circuit breaker settings
               - mTLS certificates
```

**Data Plane vs Control Plane:**

| Component | Role | Example |
|-----------|------|---------|
| **Data Plane** | Sidecar proxies that handle actual traffic | Envoy proxies in every pod |
| **Control Plane** | Manages and configures the data plane | istiod (Istio), linkerd-control-plane (Linkerd) |

---

### Service Mesh Features

**Traffic Management:**

| Feature | Description |
|---------|-------------|
| **Load balancing** | Client-side LB across service instances (round robin, least request, etc.) |
| **Traffic splitting** | Route 95% to v1, 5% to v2 (canary deployment) |
| **Traffic mirroring** | Copy live traffic to a test service (shadow testing) |
| **Retries** | Automatic retries with configurable policies |
| **Timeouts** | Per-route timeout configuration |
| **Circuit breaking** | Per-service circuit breaker settings |
| **Fault injection** | Inject delays or errors for chaos testing |
| **Rate limiting** | Per-service rate limits |

**Mutual TLS (mTLS):**

```
Without mTLS:
  Service A → plain HTTP → Service B
  (Anyone on the network can intercept or impersonate)

With mTLS (service mesh):
  Service A → Sidecar A → encrypted (mTLS) → Sidecar B → Service B
  Both sides verify each other's identity via certificates.
  Certificates are automatically rotated by the control plane.
  Zero-trust networking: every service call is authenticated and encrypted.
```

**mTLS Benefits:**
- **Encryption:** All inter-service traffic is encrypted (even within the same data center)
- **Authentication:** Each service proves its identity (prevents impersonation)
- **Authorization:** Control which services can talk to which (service-level access control)
- **Automatic:** Certificate provisioning, rotation, and revocation handled by the mesh

**Observability:**

| Signal | What the Mesh Provides |
|--------|----------------------|
| **Metrics** | Request rate, error rate, latency (P50/P95/P99) per service, per route |
| **Distributed tracing** | Automatic trace propagation across services (Jaeger, Zipkin) |
| **Access logs** | Detailed logs for every request (source, destination, status, latency) |
| **Service graph** | Visual map of service dependencies and traffic flow |
| **Health monitoring** | Real-time health status of every service instance |

---

### Istio vs Linkerd

| Feature | Istio | Linkerd |
|---------|-------|---------|
| **Sidecar proxy** | Envoy (C++) | linkerd2-proxy (Rust) |
| **Resource overhead** | Higher (~50-100 MB per sidecar) | Lower (~10-20 MB per sidecar) |
| **Complexity** | High (many features, many CRDs) | Low (simple, opinionated) |
| **Learning curve** | Steep | Gentle |
| **Traffic management** | Very rich (VirtualService, DestinationRule) | Basic (ServiceProfile, TrafficSplit) |
| **mTLS** | Automatic, configurable | Automatic, on by default |
| **Multi-cluster** | Supported | Supported |
| **Observability** | Excellent (Kiali dashboard, Prometheus, Jaeger) | Good (built-in dashboard, Prometheus) |
| **Best for** | Complex environments needing fine-grained control | Simpler environments prioritizing ease of use |

**When to Use a Service Mesh:**
- 10+ microservices with complex inter-service communication
- Need mTLS for zero-trust networking
- Need consistent resilience patterns (retries, circuit breaking) across all services
- Need observability without modifying application code
- Running on Kubernetes

**When NOT to Use a Service Mesh:**
- Small number of services (< 5) — overhead isn't justified
- Simple communication patterns — a library (Resilience4j) may be sufficient
- Not on Kubernetes — service mesh is primarily a Kubernetes pattern
- Team can't handle the operational complexity

---


## 25.7 Decomposition Strategies

> The hardest part of microservices isn't building them — it's deciding **where to draw the boundaries**. Bad boundaries lead to a "distributed monolith" — all the complexity of microservices with none of the benefits. This section covers strategies for decomposing a system into well-bounded microservices.

---

### Decompose by Business Capability

Identify the core **business capabilities** of the organization and create one service per capability.

```
E-Commerce Business Capabilities:
  - User Management → User Service
  - Product Catalog → Product Service
  - Order Management → Order Service
  - Payment Processing → Payment Service
  - Inventory Management → Inventory Service
  - Shipping & Delivery → Shipping Service
  - Notifications → Notification Service
  - Search → Search Service
  - Recommendations → Recommendation Service
  - Analytics → Analytics Service
```

**How to Identify Business Capabilities:**
1. Look at the **organizational structure** — departments often map to capabilities
2. Ask: "What does the business **do**?" (not "what technology do we use?")
3. Each capability should be **independently valuable** to the business
4. Capabilities should have **minimal overlap**

| Good Boundary | Why |
|--------------|-----|
| Order Service (create, track, cancel orders) | Complete business capability; changes independently |
| Payment Service (charge, refund, invoice) | Separate domain; different compliance requirements |
| Notification Service (email, SMS, push) | Cross-cutting but independent; different scaling needs |

| Bad Boundary | Why |
|-------------|-----|
| Database Service | Technical layer, not business capability |
| Validation Service | Cross-cutting concern, not a standalone capability |
| CRUD Service | Too generic; no business meaning |

---

### Decompose by Subdomain (DDD)

Use **Domain-Driven Design** to identify subdomains and align services with them.

**Subdomain Types:**

| Type | Description | Investment | Example |
|------|-------------|-----------|---------|
| **Core Domain** | The competitive advantage; what makes the business unique | High (custom development, best engineers) | Recommendation algorithm, pricing engine |
| **Supporting Domain** | Necessary but not differentiating; supports the core | Medium (custom but simpler) | Order management, inventory tracking |
| **Generic Domain** | Common functionality that any business needs | Low (buy or use open-source) | Authentication, email sending, payment processing |

```
E-Commerce Subdomains:

  Core Domain (competitive advantage):
    - Recommendation Engine (personalized suggestions)
    - Dynamic Pricing (real-time price optimization)
    - Search Ranking (relevance algorithm)

  Supporting Domain (necessary, custom):
    - Order Management
    - Inventory Management
    - Shipping & Logistics

  Generic Domain (buy/reuse):
    - Authentication (Auth0, Cognito)
    - Payment Processing (Stripe, PayPal)
    - Email/SMS (SendGrid, Twilio)
    - Analytics (Segment, Amplitude)
```

**Key Insight:** Invest the most in your **core domain** (custom-built, best engineers). Use third-party services for **generic domains** (don't build your own payment processor). Build **supporting domains** with moderate investment.

---

### Strangler Fig Pattern (Migrating from Monolith)

Incrementally replace parts of a monolith with microservices, routing traffic through a facade that decides whether to send requests to the old monolith or the new service.

```
Phase 1: All traffic goes to monolith
  Client → Facade → Monolith (handles everything)

Phase 2: Extract Payment Service
  Client → Facade → /payments/* → New Payment Service
                  → everything else → Monolith

Phase 3: Extract more services
  Client → Facade → /payments/* → Payment Service
                  → /users/*    → User Service
                  → /orders/*   → Order Service
                  → everything else → Monolith (shrinking)

Phase 4: Monolith is empty
  Client → Facade → All services are microservices
  Decommission the monolith.
```

**Strangler Fig Steps:**

| Step | Action | Risk |
|------|--------|------|
| 1. **Identify** | Choose a module to extract (clear boundaries, high value) | Low |
| 2. **Build** | Build the new microservice alongside the monolith | Low |
| 3. **Route** | Route traffic to the new service (feature flag or path-based routing) | Medium |
| 4. **Verify** | Run both in parallel; compare results (shadow traffic) | Low |
| 5. **Migrate data** | Move data from shared DB to the service's own DB | High |
| 6. **Switch** | Route all traffic to the new service | Medium |
| 7. **Remove** | Delete the old code from the monolith | Low |

**Key Rules:**
- **Never do a big-bang rewrite** — extract one service at a time
- **Keep the monolith running** during the entire migration
- **Start with the easiest extraction** to build confidence and tooling
- **Expect the migration to take months or years** for large monoliths

---

### Anti-Corruption Layer (ACL)

An **Anti-Corruption Layer** is a translation layer between two systems (or bounded contexts) with different models. It prevents the legacy system's model from "corrupting" the new service's clean domain model.

```
Without ACL:
  New Order Service directly calls Legacy Monolith API
  → New service must understand legacy data formats, naming conventions, quirks
  → Legacy model "corrupts" the new service's clean design

With ACL:
  New Order Service → ACL (translator) → Legacy Monolith API
  → ACL translates between new model and legacy model
  → New service has a clean domain model, unaffected by legacy
```

```
+------------------+     Clean API     +-----+     Legacy API     +------------------+
| New Order        | ────────────────→ | ACL | ────────────────→ | Legacy Monolith  |
| Service          |                   |     |                    |                  |
| (clean model:    | ←──── Clean ───── |     | ←──── Legacy ──── | (legacy model:   |
|  Order, Item,    |     response      |     |     response      |  ORDER_REC,      |
|  Amount)         |                   +-----+                    |  ITEM_LINE,      |
+------------------+                  Translates                  |  AMT_DUE)        |
                                      between models              +------------------+
```

**ACL Responsibilities:**
- **Data format translation** — convert between legacy and new data formats
- **Naming translation** — map legacy field names to new domain names
- **Protocol translation** — convert between legacy protocols (SOAP, XML) and modern (REST, JSON)
- **Error translation** — convert legacy error codes to meaningful domain errors
- **Filtering** — expose only the data the new service needs (not the entire legacy model)

**When to Use ACL:**
- Integrating with a legacy system during migration
- Calling a third-party API with a different data model
- Two bounded contexts with different models of the same concept
- Protecting a clean domain model from external complexity

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Microservices | Small, independently deployable services organized around business capabilities |
| Monolith vs Microservices | Start with monolith; migrate to microservices when you have a clear reason (scale, teams, deployment) |
| Bounded Context | Each service aligns with one bounded context; same concept can have different models in different contexts |
| Single Responsibility | One service = one business capability; not too granular, not too broad |
| Sync Communication | REST/gRPC — simple but creates coupling; use circuit breakers |
| Async Communication | Message queues/events — loose coupling, resilience, but eventual consistency |
| Choreography vs Orchestration | Choreography for simple workflows; orchestration for complex ones; many systems use both |
| Service Discovery | Client-side (query registry) or server-side (load balancer queries registry); Kubernetes DNS is usually sufficient |
| Database per Service | Each service owns its data; no shared database; use sagas for cross-service transactions |
| Saga Pattern | Sequence of local transactions with compensating actions; choreography or orchestration |
| CDC | Capture database changes as events without modifying service code (Debezium) |
| Circuit Breaker | Three states: CLOSED → OPEN → HALF-OPEN; prevents cascading failures |
| Retry + Backoff | Exponential backoff with jitter; only retry idempotent operations |
| Timeout | Always set timeouts; deadline propagation in gRPC |
| Bulkhead | Isolate thread pools per downstream service; prevent one slow service from blocking everything |
| Service Mesh | Sidecar proxies (Envoy) handle networking; mTLS, retries, circuit breaking, observability |
| Istio vs Linkerd | Istio = feature-rich, complex; Linkerd = simple, lightweight |
| Decompose by Capability | Align services with business capabilities, not technical layers |
| Strangler Fig | Incrementally extract services from monolith; never big-bang rewrite |
| Anti-Corruption Layer | Translation layer between new service and legacy system; protects clean domain model |

---

## Interview Tips for Module 25

1. **Monolith vs Microservices** — don't default to microservices; explain trade-offs; recommend starting with a modular monolith
2. **Bounded Context** — explain with an example; show how "User" means different things in different contexts
3. **Communication patterns** — explain sync vs async; know when to use each; mention choreography vs orchestration
4. **Service discovery** — explain client-side vs server-side; mention Kubernetes DNS as the default in K8s
5. **Database per service** — explain why shared database is an anti-pattern; discuss data consistency challenges
6. **Saga pattern** — draw the flow for a multi-service transaction; show compensating actions on failure
7. **Circuit breaker** — explain the three states with a concrete example; mention fallback strategies
8. **Retry with backoff** — explain exponential backoff + jitter; warn about retrying non-idempotent operations
9. **Bulkhead** — explain with the thread pool isolation example; show how it prevents cascading failures
10. **Service mesh** — explain the sidecar pattern; mention mTLS, observability; know when to use vs when it's overkill
11. **Strangler Fig** — explain the incremental migration approach; never suggest a big-bang rewrite
12. **Decomposition** — explain by business capability and by subdomain; identify core vs supporting vs generic domains
