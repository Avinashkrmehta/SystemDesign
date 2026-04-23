# Module 17: Web Architecture Basics

> Before diving into databases, caching, and distributed systems, you need a solid understanding of how web applications are structured at a high level. This module covers the foundational architectural patterns — client-server, monolithic, N-tier — as well as the critical infrastructure components that sit between clients and servers: proxy servers, CDNs, and DNS-based routing. These concepts form the skeleton on which every system design is built.

---

## 17.1 Client-Server Architecture

> The **client-server model** is the most fundamental architecture pattern on the web. A **client** (browser, mobile app, desktop app) sends requests to a **server**, which processes them and returns responses. Nearly every web application follows this model.

---

### Request-Response Model

The web is built on a **request-response** cycle:

```
+--------+                          +--------+
| Client |  ---- HTTP Request --->  | Server |
|        |                          |        |
| (Browser,  <--- HTTP Response --- | (Web   |
|  Mobile,   |                      |  App,  |
|  Desktop)  |                      |  API)  |
+--------+                          +--------+
```

**How it works:**
1. Client initiates a request (e.g., `GET /api/users`)
2. Request travels over the network (DNS resolution → TCP connection → TLS handshake → HTTP request)
3. Server receives the request, processes it (business logic, database queries, etc.)
4. Server sends back a response (status code, headers, body)
5. Client receives and renders/processes the response

**Key Characteristics:**
- **Client initiates** — the server never spontaneously sends data to the client (in basic HTTP)
- **Synchronous** — client waits for the response before proceeding (though async patterns exist)
- **Separation of concerns** — client handles presentation, server handles data and logic

**Modern Variations:**

| Pattern | Description | Example |
|---------|-------------|---------|
| Traditional request-response | Client sends request, waits for response | REST API call |
| Long polling | Server holds the request open until data is available | Chat notifications |
| Server-Sent Events (SSE) | Server pushes events to client over a persistent connection | Live score updates |
| WebSocket | Full-duplex bidirectional communication | Real-time chat, gaming |
| Webhooks | Server calls client's URL when an event occurs | Payment confirmation callback |

---

### Stateless vs Stateful Servers

**Stateless Server:**
The server does **not** store any information about previous requests. Each request is independent and self-contained.

```
Request 1: GET /api/profile
            Authorization: Bearer <token>
            → Server validates token, fetches profile, responds

Request 2: GET /api/orders
            Authorization: Bearer <token>
            → Server validates token again, fetches orders, responds
            → Server has NO memory of Request 1
```

**Characteristics of Stateless Servers:**

| Aspect | Detail |
|--------|--------|
| Session storage | None on the server — client sends identity with every request (JWT, API key) |
| Horizontal scaling | Easy — any server can handle any request |
| Load balancing | Simple round-robin — no sticky sessions needed |
| Fault tolerance | High — if a server dies, others pick up seamlessly |
| Complexity | Client must send all context with every request |

**Stateful Server:**
The server **remembers** information about the client between requests (e.g., server-side sessions).

```
Request 1: POST /login
            → Server creates session, stores in memory: { session_id: "abc", user: "Alice" }
            → Response: Set-Cookie: session_id=abc

Request 2: GET /api/profile
            Cookie: session_id=abc
            → Server looks up session "abc" in memory → finds user "Alice"
            → Returns Alice's profile
```

**Characteristics of Stateful Servers:**

| Aspect | Detail |
|--------|--------|
| Session storage | Server memory, local disk, or in-process cache |
| Horizontal scaling | Hard — requests must go to the same server (sticky sessions) |
| Load balancing | Requires session affinity (sticky sessions) |
| Fault tolerance | Low — if the server dies, all sessions are lost |
| Complexity | Simpler client logic (server remembers context) |

**Making Stateful Systems Scalable:**

The solution is to **externalize state** — move session data out of the server into a shared store:

```
Stateful (bad for scaling):
  Client → Server A (session in memory)
  Client → Server B (no session — fails!)

Externalized State (scalable):
  Client → Server A → Redis (shared session store)
  Client → Server B → Redis (same session found — works!)
```

| Approach | How It Works | Trade-off |
|----------|-------------|-----------|
| **Sticky sessions** | Load balancer routes same client to same server | Simple but fragile — server death loses sessions |
| **External session store** | Sessions stored in Redis/Memcached | Adds dependency but enables true statelessness |
| **JWT tokens** | Session data encoded in the token itself | Truly stateless but tokens can't be easily revoked |
| **Encrypted cookies** | Session data encrypted in cookies | Stateless, limited by cookie size (4KB) |

**Key Point for System Design:** Always design servers to be **stateless** when possible. Externalize state to a shared store (Redis, database). This enables horizontal scaling, simple load balancing, and fault tolerance.

---

### Thin Client vs Thick Client

The division of work between client and server varies:

**Thin Client (Server-Side Rendering):**
- Server does most of the work — generates complete HTML pages
- Client just renders the HTML
- Minimal client-side logic

```
Client: "Give me the users page"
Server: Fetches data, renders HTML template, returns complete page
Client: Displays the HTML
```

**Examples:** Traditional server-rendered apps (PHP, Ruby on Rails, Django templates, JSP)

**Thick Client (Client-Side Rendering / SPA):**
- Client does most of the work — renders UI, manages state, handles routing
- Server provides data via APIs (REST, GraphQL)
- Rich, interactive user experience

```
Client: Loads JavaScript application (SPA)
Client: "Give me the users data" → GET /api/users
Server: Returns JSON data
Client: Renders the data into HTML using JavaScript
```

**Examples:** React, Angular, Vue.js single-page applications (SPAs)

**Comparison:**

| Feature | Thin Client (SSR) | Thick Client (SPA) |
|---------|-------------------|-------------------|
| Initial load | Fast (server sends ready HTML) | Slower (must download JS bundle first) |
| Subsequent navigation | Slower (full page reload) | Faster (client-side routing, API calls only) |
| SEO | Good (HTML is crawlable) | Requires SSR/SSG for SEO |
| Server load | Higher (renders HTML for every request) | Lower (serves static files + API) |
| Interactivity | Limited | Rich, app-like experience |
| Offline capability | None | Possible (service workers, local storage) |
| Complexity | Simpler frontend | Complex frontend (state management, routing) |

**Modern Hybrid Approaches:**

| Approach | Description | Example |
|----------|-------------|---------|
| **SSR + Hydration** | Server renders initial HTML, client "hydrates" it with JavaScript | Next.js, Nuxt.js |
| **SSG (Static Site Generation)** | Pages pre-rendered at build time, served as static files | Gatsby, Hugo, Next.js static export |
| **ISR (Incremental Static Regeneration)** | Static pages regenerated on-demand after a TTL | Next.js ISR |
| **Islands Architecture** | Mostly static HTML with interactive "islands" of JavaScript | Astro |

**Key Point for System Design:** Most modern systems use a **thick client (SPA)** that communicates with a **stateless API backend**. The API serves JSON data, and the client handles rendering. For SEO-critical pages, SSR or SSG is used.

---


## 17.2 Monolithic Architecture

> A **monolith** is an application where all components — UI, business logic, data access, background jobs — are built, deployed, and scaled as a **single unit**. Despite the industry's enthusiasm for microservices, monoliths remain the right choice for many applications, especially in the early stages.

---

### Single Deployable Unit

In a monolithic architecture, the entire application is packaged and deployed as one artifact:

```
+----------------------------------------------------------+
|                    Monolithic Application                  |
|                                                          |
|  +-------------+  +-------------+  +------------------+  |
|  | User Module |  | Order Module|  | Payment Module   |  |
|  +-------------+  +-------------+  +------------------+  |
|  +-------------+  +-------------+  +------------------+  |
|  | Auth Module |  | Search      |  | Notification     |  |
|  |             |  | Module      |  | Module           |  |
|  +-------------+  +-------------+  +------------------+  |
|                                                          |
|  +---------------------------------------------------+   |
|  |              Shared Database                       |   |
|  +---------------------------------------------------+   |
+----------------------------------------------------------+
         |
    Single Deployment
    Single Process
    Single Database
```

**Characteristics:**
- All modules run in the **same process**
- Modules communicate via **function calls** (in-process, not over the network)
- Single **shared database** for all modules
- Deployed as a **single artifact** (JAR, WAR, binary, Docker image)
- Scaled by running **multiple copies** of the entire application behind a load balancer

---

### Advantages and Disadvantages

**Advantages:**

| Advantage | Explanation |
|-----------|-------------|
| **Simple development** | One codebase, one IDE, one build system — easy to understand and navigate |
| **Simple deployment** | Deploy one artifact — no orchestration of multiple services |
| **Simple testing** | End-to-end tests run against one application — no service mocking needed |
| **Low latency** | Module-to-module calls are in-process function calls (nanoseconds, not milliseconds) |
| **No network complexity** | No service discovery, no inter-service auth, no distributed tracing |
| **Easy debugging** | Single process — stack traces show the full call chain |
| **ACID transactions** | Single database — transactions span all modules naturally |
| **Low operational overhead** | One application to monitor, log, and maintain |
| **Team simplicity** | Small teams can work effectively in one codebase |

**Disadvantages:**

| Disadvantage | Explanation |
|-------------|-------------|
| **Scaling limitations** | Must scale the entire application even if only one module is the bottleneck |
| **Deployment risk** | A bug in one module requires redeploying everything; one bad deploy affects all features |
| **Technology lock-in** | Entire application must use the same language, framework, and runtime |
| **Long build/deploy times** | As the codebase grows, builds and deployments become slower |
| **Tight coupling risk** | Without discipline, modules become entangled — changes in one break others |
| **Team scaling** | Large teams stepping on each other's toes in the same codebase |
| **Reliability** | A memory leak or crash in one module takes down the entire application |
| **Partial deployment impossible** | Can't deploy just the payment module — must deploy everything |

---

### When Monolith is the Right Choice

| Scenario | Why Monolith |
|----------|-------------|
| **Early-stage startup** | Speed of development matters most; don't over-engineer |
| **Small team (< 10 developers)** | One codebase is easier to manage than 20 microservices |
| **Simple domain** | If the business logic isn't complex, microservices add unnecessary overhead |
| **Unclear domain boundaries** | You don't know where to split yet — premature decomposition is worse than a monolith |
| **Tight deadlines** | Monolith is faster to build, test, and deploy |
| **Low scale requirements** | If you don't need to handle millions of requests, a monolith on a beefy server works fine |

**The Golden Rule:** Start with a monolith. Split into microservices only when you have a clear reason — scaling bottleneck, team scaling, or deployment independence.

> "If you can't build a well-structured monolith, what makes you think you can build a well-structured set of microservices?" — Simon Brown

---

### Modular Monolith

A **modular monolith** is a monolith with well-defined internal boundaries. Each module has its own domain, its own data, and communicates with other modules through well-defined interfaces — but it's still deployed as a single unit.

```
+----------------------------------------------------------+
|                    Modular Monolith                        |
|                                                          |
|  +----------------+  +----------------+  +--------------+ |
|  | User Module    |  | Order Module   |  | Payment      | |
|  |                |  |                |  | Module       | |
|  | - UserService  |  | - OrderService |  | - PayService | |
|  | - UserRepo     |  | - OrderRepo    |  | - PayRepo    | |
|  | - User tables  |  | - Order tables |  | - Pay tables | |
|  |                |  |                |  |              | |
|  | Public API:    |  | Public API:    |  | Public API:  | |
|  | - getUser()    |  | - createOrder()|  | - charge()   | |
|  | - createUser() |  | - getOrders()  |  | - refund()   | |
|  +-------↑--------+  +-------↑--------+  +------↑-------+ |
|          |                    |                   |         |
|          +-------- Internal APIs (function calls) +         |
|                                                          |
|  +---------------------------------------------------+   |
|  |              Shared Database (but separate schemas)|   |
|  +---------------------------------------------------+   |
+----------------------------------------------------------+
```

**Key Principles:**
- Each module **owns its data** (separate tables/schemas, no cross-module direct DB access)
- Modules communicate through **public interfaces** (not by reaching into each other's internals)
- **No circular dependencies** between modules
- Each module can be **tested independently**
- If you later need to extract a module into a microservice, the boundaries are already clean

**Modular Monolith vs Microservices:**

| Feature | Modular Monolith | Microservices |
|---------|-----------------|---------------|
| Deployment | Single unit | Independent per service |
| Communication | In-process function calls | Network calls (HTTP, gRPC, messaging) |
| Data isolation | Separate schemas, same DB | Separate databases |
| Latency | Nanoseconds (function call) | Milliseconds (network) |
| Transactions | ACID (single DB) | Distributed (saga, eventual consistency) |
| Operational complexity | Low | High |
| Team independence | Moderate | High |
| Scaling granularity | Whole application | Per service |

**Key Point for System Design:** The modular monolith is often the best starting point. It gives you clean boundaries (making future extraction easy) without the operational complexity of microservices. Many successful companies (Shopify, Basecamp) run modular monoliths at scale.

---

### Monolith to Microservices — Migration Strategies

When a monolith outgrows its architecture, you need a strategy for incremental migration — never a big-bang rewrite.

**Strangler Fig Pattern:**

Named after the strangler fig tree that grows around a host tree and eventually replaces it. You incrementally replace parts of the monolith with microservices while keeping the monolith running.

```
Phase 1: Monolith handles everything
  Client → Monolith (Users, Orders, Payments, Notifications)

Phase 2: Extract one service, route traffic through a facade
  Client → API Gateway / Facade
            ├── /users/*        → Monolith (still handles users)
            ├── /orders/*       → Monolith (still handles orders)
            ├── /payments/*     → New Payment Service  ← extracted!
            └── /notifications  → Monolith

Phase 3: Continue extracting
  Client → API Gateway
            ├── /users/*        → User Service         ← extracted!
            ├── /orders/*       → Order Service         ← extracted!
            ├── /payments/*     → Payment Service
            └── /notifications  → Notification Service  ← extracted!

Phase 4: Monolith is empty — decommission it
```

**Steps for Each Extraction:**
1. **Identify the module** with the clearest boundaries and highest value for extraction
2. **Define the API** between the new service and the monolith
3. **Duplicate the code** into the new service (don't delete from monolith yet)
4. **Migrate the data** — new service gets its own database
5. **Route traffic** to the new service (feature flag or API gateway routing)
6. **Verify** the new service works correctly (shadow traffic, canary deployment)
7. **Remove the old code** from the monolith

**Anti-Corruption Layer:**
When the new microservice needs to communicate with the monolith, use an anti-corruption layer to translate between the monolith's data model and the microservice's domain model. This prevents the monolith's legacy design from leaking into the new service.

**What to Extract First:**

| Good Candidates | Why |
|----------------|-----|
| Modules with clear boundaries | Easier to separate, fewer cross-dependencies |
| Modules with different scaling needs | Payment processing needs different scaling than user profiles |
| Modules with different deployment cadence | Notification templates change daily; user auth changes monthly |
| Modules with different technology needs | ML recommendation engine needs Python; core API is in Java |
| Modules causing the most pain | Frequent bugs, slow deployments, team conflicts |

| Bad Candidates | Why |
|---------------|-----|
| Tightly coupled modules | Too many cross-dependencies; extraction creates a distributed monolith |
| Core domain with unclear boundaries | You'll get the boundaries wrong and create worse problems |
| Modules with shared database tables | Data migration is extremely complex |

**Common Mistakes:**
- **Distributed monolith:** Extracting services that are still tightly coupled — you get all the complexity of microservices with none of the benefits
- **Big-bang rewrite:** Trying to rewrite everything at once instead of incrementally
- **Shared database:** Multiple services reading/writing the same tables — defeats the purpose of separation
- **Too many services too fast:** Each service adds operational overhead (deployment, monitoring, debugging)

---


## 17.3 N-Tier Architecture

> **N-Tier (or multi-tier) architecture** separates an application into **logical layers**, each with a distinct responsibility. The most common form is **3-tier architecture**: Presentation, Business Logic, and Data Access. This separation improves maintainability, testability, and allows each layer to evolve independently.

---

### The Three Tiers

```
+----------------------------------------------------------+
|                   Presentation Layer                      |
|  (UI, Controllers, Views, API Endpoints)                 |
|  - Handles user interaction and request/response          |
|  - Validates input format                                 |
|  - Renders output (HTML, JSON)                            |
+---------------------------+------------------------------+
                            |
                            ↓
+----------------------------------------------------------+
|                  Business Logic Layer                      |
|  (Services, Domain Models, Use Cases)                     |
|  - Core application logic and rules                       |
|  - Orchestrates workflows                                 |
|  - Enforces business constraints                          |
|  - Independent of UI and database                         |
+---------------------------+------------------------------+
                            |
                            ↓
+----------------------------------------------------------+
|                   Data Access Layer                        |
|  (Repositories, DAOs, ORM, Database Clients)              |
|  - Communicates with the database                         |
|  - Abstracts storage details from business logic          |
|  - Handles queries, transactions, connection pooling      |
+---------------------------+------------------------------+
                            |
                            ↓
+----------------------------------------------------------+
|                      Database                             |
|  (PostgreSQL, MySQL, MongoDB, Redis)                      |
+----------------------------------------------------------+
```

---

### Presentation Layer

The **Presentation Layer** (also called the UI Layer or Interface Layer) is the entry point for all client interactions.

**Responsibilities:**
- Receive and parse incoming requests (HTTP, gRPC, WebSocket)
- Input validation (format, required fields — NOT business rules)
- Authentication (verify identity — who are you?)
- Route requests to the appropriate business logic
- Format and return responses (JSON, HTML, protobuf)
- Handle HTTP status codes and error formatting

**What belongs here:**
- API controllers / route handlers
- Request/response DTOs (Data Transfer Objects)
- Input validation (format checks)
- Serialization / deserialization
- CORS configuration
- Rate limiting (or delegated to API Gateway)

**What does NOT belong here:**
- Business rules (e.g., "users can only have 3 active orders")
- Database queries
- Complex calculations

```
// Pseudocode — Presentation Layer (Controller)
class UserController {
    UserService userService;  // injected dependency

    Response getUser(Request req) {
        string userId = req.pathParam("id");       // parse input
        if (!isValidUUID(userId))                   // format validation
            return Response(400, "Invalid user ID");

        User user = userService.getUserById(userId); // delegate to business layer
        if (user == null)
            return Response(404, "User not found");

        return Response(200, toJSON(user));          // format output
    }
};
```

---

### Business Logic Layer

The **Business Logic Layer** (also called the Service Layer or Domain Layer) contains the core application logic — the rules that define what the application does.

**Responsibilities:**
- Implement business rules and workflows
- Orchestrate operations across multiple repositories
- Enforce domain constraints and invariants
- Transaction management
- Authorization (verify permissions — are you allowed to do this?)

**What belongs here:**
- Service classes (UserService, OrderService, PaymentService)
- Domain models / entities
- Business validation (e.g., "order total must be positive", "user can't order more than inventory")
- Workflow orchestration (e.g., "create order → reserve inventory → charge payment → send confirmation")

**What does NOT belong here:**
- HTTP-specific logic (status codes, headers, request parsing)
- Database-specific logic (SQL queries, connection management)
- UI rendering

```
// Pseudocode — Business Logic Layer (Service)
class OrderService {
    OrderRepository orderRepo;
    InventoryService inventoryService;
    PaymentService paymentService;
    NotificationService notificationService;

    Order createOrder(string userId, vector<OrderItem> items) {
        // Business validation
        if (items.empty())
            throw BusinessException("Order must have at least one item");

        double total = calculateTotal(items);
        if (total <= 0)
            throw BusinessException("Order total must be positive");

        // Business workflow
        inventoryService.reserve(items);           // Step 1: Reserve inventory
        Payment payment = paymentService.charge(userId, total);  // Step 2: Charge
        Order order = orderRepo.save(Order(userId, items, total, payment.id));  // Step 3: Save
        notificationService.sendOrderConfirmation(order);  // Step 4: Notify

        return order;
    }
};
```

---

### Data Access Layer

The **Data Access Layer** (also called the Repository Layer or Persistence Layer) abstracts all database interactions behind a clean interface.

**Responsibilities:**
- Execute database queries (CRUD operations)
- Map between domain objects and database rows/documents
- Manage database connections and connection pooling
- Handle database-specific concerns (SQL dialects, query optimization)

**What belongs here:**
- Repository classes (UserRepository, OrderRepository)
- ORM configuration and mappings
- Raw SQL queries (if not using ORM)
- Database migration scripts
- Connection pool management

**What does NOT belong here:**
- Business rules
- Input validation
- HTTP handling

```
// Pseudocode — Data Access Layer (Repository)
class OrderRepository {
    DatabaseConnection db;

    Order findById(string id) {
        auto row = db.query("SELECT * FROM orders WHERE id = ?", id);
        return mapToOrder(row);
    }

    vector<Order> findByUserId(string userId) {
        auto rows = db.query("SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC", userId);
        return mapToOrders(rows);
    }

    Order save(Order order) {
        db.execute("INSERT INTO orders (id, user_id, total, status) VALUES (?, ?, ?, ?)",
                   order.id, order.userId, order.total, order.status);
        return order;
    }

    void updateStatus(string orderId, string status) {
        db.execute("UPDATE orders SET status = ?, updated_at = NOW() WHERE id = ?",
                   status, orderId);
    }
};
```

---

### Benefits of Separation

| Benefit | Explanation |
|---------|-------------|
| **Maintainability** | Changes to one layer don't ripple through others — change the database without touching business logic |
| **Testability** | Each layer can be tested independently — mock the data layer to test business logic |
| **Reusability** | Business logic can be reused across different presentation layers (REST API, gRPC, CLI, WebSocket) |
| **Team organization** | Frontend team works on presentation, backend team on business logic, DBA on data layer |
| **Technology flexibility** | Swap the database (MySQL → PostgreSQL) by changing only the data layer |
| **Separation of concerns** | Each layer has one job — easier to reason about and debug |
| **Scalability** | Layers can be scaled independently (e.g., more API servers, read replicas for data layer) |

**The Dependency Rule:**
Dependencies should point **inward** — outer layers depend on inner layers, never the reverse:

```
Presentation → Business Logic → Data Access → Database

✅ Controller calls Service
✅ Service calls Repository
❌ Repository should NOT call Service
❌ Service should NOT know about HTTP status codes
```

This follows the **Dependency Inversion Principle** (Module 3, Section 3.5) — high-level modules (business logic) should not depend on low-level modules (data access). Both should depend on abstractions (interfaces).

---

### Beyond 3-Tier: Common Variations

| Architecture | Layers | Use Case |
|-------------|--------|----------|
| **2-Tier** | Client + Database (no middle tier) | Simple desktop apps, prototypes |
| **3-Tier** | Presentation + Business + Data | Most web applications |
| **4-Tier** | Client + API Gateway + Application + Database | Microservices with API gateway |
| **Clean Architecture** | Entities → Use Cases → Interface Adapters → Frameworks | Domain-driven, highly testable |
| **Hexagonal (Ports & Adapters)** | Core domain + Ports (interfaces) + Adapters (implementations) | Flexible, swappable infrastructure |

---

### Clean Architecture (Deep Dive)

**Clean Architecture** (by Robert C. Martin) organizes code in concentric circles where dependencies point **inward**. The inner circles know nothing about the outer circles.

```
+------------------------------------------------------------------+
|  Frameworks & Drivers (outermost)                                |
|  (Web framework, Database driver, UI, External APIs)             |
|                                                                  |
|  +------------------------------------------------------------+  |
|  |  Interface Adapters                                        |  |
|  |  (Controllers, Presenters, Gateways, Repositories impl)   |  |
|  |                                                            |  |
|  |  +------------------------------------------------------+  |  |
|  |  |  Application Business Rules (Use Cases)              |  |  |
|  |  |  (Application-specific logic, orchestration)         |  |  |
|  |  |                                                      |  |  |
|  |  |  +------------------------------------------------+  |  |  |
|  |  |  |  Enterprise Business Rules (Entities)          |  |  |  |
|  |  |  |  (Core domain objects, business rules)         |  |  |  |
|  |  |  |  (No dependencies on anything external)        |  |  |  |
|  |  |  +------------------------------------------------+  |  |  |
|  |  +------------------------------------------------------+  |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+

Dependency Rule: Dependencies point INWARD only
  Frameworks → Adapters → Use Cases → Entities
  ✅ Controller depends on UseCase
  ✅ UseCase depends on Entity
  ❌ Entity NEVER depends on UseCase
  ❌ UseCase NEVER depends on Controller or Database
```

**Layer Responsibilities:**

| Layer | Contains | Example |
|-------|----------|---------|
| **Entities** | Core business objects and rules that exist independent of any application | `Order`, `User`, `Product` with validation rules |
| **Use Cases** | Application-specific business rules; orchestrates entities | `CreateOrderUseCase`, `CancelOrderUseCase` |
| **Interface Adapters** | Converts data between use cases and external formats | Controllers (HTTP → UseCase), Repository implementations (UseCase → SQL) |
| **Frameworks & Drivers** | External tools and frameworks | Express.js, PostgreSQL driver, React, gRPC server |

**Key Benefit:** The core business logic (Entities + Use Cases) has **zero dependencies** on frameworks, databases, or UI. You can swap PostgreSQL for MongoDB, or REST for gRPC, without changing a single line of business logic.

---

### Hexagonal Architecture (Ports & Adapters)

**Hexagonal Architecture** (by Alistair Cockburn) achieves the same goal as Clean Architecture but uses different terminology: **Ports** (interfaces) and **Adapters** (implementations).

```
                    +------------------+
                    |   HTTP Adapter   |  (REST Controller)
                    +--------+---------+
                             |
                    +--------v---------+
                    |    Input Port    |  (UseCase interface)
                    +--------+---------+
                             |
              +--------------v--------------+
              |                             |
              |      Application Core       |
              |   (Domain Logic, Entities,  |
              |    Use Cases)               |
              |                             |
              +--------------+--------------+
                             |
                    +--------v---------+
                    |   Output Port    |  (Repository interface)
                    +--------+---------+
                             |
              +--------------+--------------+
              |                             |
    +---------v---------+    +--------------v---------+
    | PostgreSQL Adapter |    | Redis Cache Adapter   |
    | (implements repo)  |    | (implements cache)    |
    +--------------------+    +------------------------+
```

**Ports:** Interfaces defined by the application core — they describe what the core needs from the outside world
- **Input Ports (Driving):** How the outside world talks to the core (e.g., `CreateOrderUseCase` interface)
- **Output Ports (Driven):** How the core talks to the outside world (e.g., `OrderRepository` interface)

**Adapters:** Concrete implementations of ports
- **Input Adapters (Driving):** REST controller, gRPC handler, CLI command, message consumer
- **Output Adapters (Driven):** PostgreSQL repository, Redis cache, SMTP email sender, Kafka producer

**Key Benefit:** You can test the entire application core by providing mock adapters. You can swap any adapter without touching the core. The same core can be driven by REST, gRPC, CLI, or message queues simultaneously.

---


## 17.4 Proxy Servers

> A **proxy server** is an intermediary that sits between clients and servers, forwarding requests and responses. Proxies are fundamental infrastructure components used for caching, security, load balancing, anonymity, and traffic management. Understanding the difference between forward and reverse proxies is essential for system design.

---

### Forward Proxy vs Reverse Proxy

**Forward Proxy — sits in front of clients:**

A forward proxy acts on behalf of **clients**. The client sends requests to the proxy, and the proxy forwards them to the destination server. The server sees the proxy's IP, not the client's.

```
+--------+     +---------------+     +--------+
| Client | --> | Forward Proxy | --> | Server |
+--------+     +---------------+     +--------+
                (acts for client)

Client knows about the proxy.
Server does NOT know about the client.
```

**Use Cases for Forward Proxy:**

| Use Case | How It Works |
|----------|-------------|
| **Anonymity / Privacy** | Hides client's IP address from the server |
| **Content filtering** | Corporate proxy blocks access to certain websites |
| **Caching** | Caches frequently accessed content to reduce bandwidth |
| **Access control** | Restricts which external sites employees can access |
| **Bypassing geo-restrictions** | Client appears to be in the proxy's location |
| **Logging / Monitoring** | Logs all outbound requests for compliance |

**Examples:** Squid, corporate HTTP proxies, VPN (acts as a forward proxy at the network level)

---

**Reverse Proxy — sits in front of servers:**

A reverse proxy acts on behalf of **servers**. The client sends requests to the reverse proxy (thinking it's the actual server), and the proxy forwards them to the appropriate backend server.

```
+--------+     +---------------+     +-----------+
| Client | --> | Reverse Proxy | --> | Server A  |
+--------+     +---------------+     +-----------+
                (acts for server)  --> | Server B  |
                                      +-----------+
                                   --> | Server C  |
                                      +-----------+

Client does NOT know about the backend servers.
Server knows about the proxy (proxy adds headers like X-Forwarded-For).
```

**Use Cases for Reverse Proxy:**

| Use Case | How It Works |
|----------|-------------|
| **Load balancing** | Distributes requests across multiple backend servers |
| **SSL/TLS termination** | Handles HTTPS encryption/decryption, forwards plain HTTP to backends |
| **Caching** | Caches static content and API responses |
| **Compression** | Compresses responses (gzip, brotli) before sending to client |
| **Security** | Hides backend server details, blocks malicious requests (WAF) |
| **Rate limiting** | Limits requests per client/IP |
| **Static file serving** | Serves static assets (images, CSS, JS) directly without hitting the application server |
| **URL rewriting** | Rewrites URLs before forwarding to backends |
| **A/B testing** | Routes a percentage of traffic to different backend versions |

---

### Comparison

| Feature | Forward Proxy | Reverse Proxy |
|---------|--------------|---------------|
| Acts on behalf of | Client | Server |
| Client awareness | Client knows about proxy | Client doesn't know about proxy |
| Server awareness | Server doesn't know about client | Server knows about proxy |
| Typical location | Client's network | Server's network |
| Primary purpose | Privacy, filtering, caching for clients | Load balancing, security, caching for servers |
| Configuration | Client must be configured to use it | Transparent to client (DNS points to proxy) |

---

### Use Cases in System Design

**1. Caching:**

```
Client → Reverse Proxy (Nginx)
         ├── Cache HIT → Return cached response (fast, no backend hit)
         └── Cache MISS → Forward to backend → Cache response → Return to client
```

- Cache static assets (images, CSS, JS) with long TTLs
- Cache API responses with short TTLs
- Reduces backend load significantly

**2. Security:**

```
Client → Reverse Proxy (WAF + Rate Limiter)
         ├── Malicious request → Block (403 Forbidden)
         ├── Rate limit exceeded → Reject (429 Too Many Requests)
         └── Valid request → Forward to backend
```

- Web Application Firewall (WAF) blocks SQL injection, XSS, etc.
- DDoS protection by absorbing/filtering traffic
- Hide backend server IPs and architecture details

**3. Load Balancing:**

```
Client → Reverse Proxy (Load Balancer)
         ├── Server A (healthy, low load) → Route here
         ├── Server B (healthy, high load) → Skip
         └── Server C (unhealthy) → Remove from pool
```

- Distribute traffic across multiple backend instances
- Health checks to remove unhealthy servers
- Various algorithms: round-robin, least connections, IP hash

**4. SSL/TLS Termination:**

```
Client ──HTTPS──→ Reverse Proxy ──HTTP──→ Backend Servers
                  (handles TLS)          (plain HTTP, faster)
```

- Proxy handles the expensive TLS handshake and encryption
- Backend servers communicate over plain HTTP within the trusted internal network
- Simplifies certificate management (one place to manage certs)

---

### Nginx, HAProxy, Envoy

| Feature | Nginx | HAProxy | Envoy |
|---------|-------|---------|-------|
| **Primary role** | Web server + reverse proxy | Load balancer + proxy | Service mesh sidecar proxy |
| **Layer** | L7 (HTTP) + L4 (TCP) | L4 (TCP) + L7 (HTTP) | L4 + L7 |
| **Static file serving** | Excellent | No | No |
| **Load balancing** | Good | Excellent | Excellent |
| **Health checks** | Basic | Advanced (active + passive) | Advanced |
| **gRPC support** | Yes (since 1.13) | Yes | Native (built for gRPC) |
| **WebSocket support** | Yes | Yes | Yes |
| **Configuration** | Static config file (reload) | Static config file (reload) | Dynamic (xDS API, hot reload) |
| **Service discovery** | Manual / DNS | Manual / DNS | Dynamic (integrates with Consul, K8s) |
| **Observability** | Basic (access logs) | Good (stats page) | Excellent (distributed tracing, metrics) |
| **TLS termination** | Yes | Yes | Yes (+ mTLS) |
| **Best for** | Web serving + simple reverse proxy | High-performance load balancing | Kubernetes, service mesh (Istio) |

**When to use which:**

| Scenario | Recommendation |
|----------|---------------|
| Serve static files + reverse proxy | Nginx |
| High-performance TCP/HTTP load balancing | HAProxy |
| Kubernetes / service mesh | Envoy (with Istio) |
| Simple setup, general purpose | Nginx |
| Need advanced health checks and failover | HAProxy |
| Need dynamic configuration and observability | Envoy |

---

### Practical Nginx Configuration Example

A real-world Nginx configuration for a web application with reverse proxy, load balancing, SSL termination, caching, and rate limiting:

```nginx
# Define upstream backend servers
upstream app_servers {
    least_conn;                          # Load balancing algorithm
    server 10.0.1.10:8080 weight=3;     # Higher weight = more traffic
    server 10.0.1.11:8080 weight=2;
    server 10.0.1.12:8080 weight=1;     # Canary server (less traffic)
    server 10.0.1.13:8080 backup;       # Only used when others are down

    keepalive 64;                        # Connection pool to backends
}

# Rate limiting zone (10 requests/second per IP)
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

# Caching configuration
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m
                 max_size=1g inactive=60m use_temp_path=off;

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # SSL/TLS Configuration
    ssl_certificate     /etc/ssl/certs/api.example.com.crt;
    ssl_certificate_key /etc/ssl/private/api.example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Gzip compression
    gzip on;
    gzip_types application/json text/plain text/css application/javascript;
    gzip_min_length 1000;

    # Static files — served directly by Nginx
    location /static/ {
        root /var/www;
        expires 1y;                      # Cache for 1 year
        add_header Cache-Control "public, immutable";
    }

    # API — reverse proxy to backend with rate limiting
    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;  # Allow burst of 20

        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-ID $request_id;

        # Timeouts
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 10s;

        # Caching for GET requests
        proxy_cache api_cache;
        proxy_cache_methods GET;
        proxy_cache_valid 200 5m;        # Cache 200 responses for 5 minutes
        proxy_cache_valid 404 1m;        # Cache 404 responses for 1 minute
        proxy_cache_use_stale error timeout updating;  # Serve stale on error
        add_header X-Cache-Status $upstream_cache_status;
    }

    # WebSocket support
    location /ws/ {
        proxy_pass http://app_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;        # Keep WebSocket alive for 1 hour
    }

    # Health check endpoint (not proxied)
    location /health {
        return 200 '{"status": "healthy"}';
        add_header Content-Type application/json;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$server_name$request_uri;
}
```

**Connection Pooling:**

Without connection pooling, every request from the proxy to the backend opens a new TCP connection (expensive — 3-way handshake + potential TLS). Connection pooling maintains a pool of persistent connections that are reused across requests.

```
Without Connection Pooling:
  Request 1: Nginx → TCP Handshake → Backend → Response → Close
  Request 2: Nginx → TCP Handshake → Backend → Response → Close
  Request 3: Nginx → TCP Handshake → Backend → Response → Close

With Connection Pooling (keepalive):
  Nginx maintains pool of 64 connections to backends
  Request 1: Nginx → Reuse Connection A → Backend → Response
  Request 2: Nginx → Reuse Connection B → Backend → Response
  Request 3: Nginx → Reuse Connection A → Backend → Response
  (Connections stay open and are reused)
```

**Impact:** Connection pooling can reduce backend latency by 5-20ms per request (eliminating TCP/TLS handshake overhead) and significantly reduce the number of connections backends need to handle.

---


## 17.5 CDN (Content Delivery Network)

> A **CDN (Content Delivery Network)** is a geographically distributed network of servers that caches and delivers content to users from the server closest to them. CDNs dramatically reduce latency, offload traffic from origin servers, and improve availability. Every large-scale web application uses a CDN.

---

### How CDNs Work

```
Without CDN:
  User in Tokyo → Request travels to Origin Server in US East → ~200ms latency
  User in London → Request travels to Origin Server in US East → ~100ms latency

With CDN:
  User in Tokyo → CDN Edge Server in Tokyo → ~10ms latency (cached)
  User in London → CDN Edge Server in London → ~10ms latency (cached)
  
  If not cached at edge:
  User in Tokyo → CDN Edge in Tokyo → Origin Server in US East → Cache at edge → Return
  (First request is slow, subsequent requests are fast)
```

**CDN Architecture:**

```
                        +------------------+
                        |  Origin Server   |
                        |  (Your Server)   |
                        +--------+---------+
                                 |
                    +------------+------------+
                    |                         |
            +-------+-------+       +--------+------+
            | CDN Edge      |       | CDN Edge      |
            | (US West)     |       | (Europe)      |
            +-------+-------+       +--------+------+
                    |                         |
              +-----+-----+           +------+------+
              |           |           |            |
         +----+----+ +----+----+ +----+----+ +----+----+
         |User     | |User     | |User     | |User     |
         |(LA)     | |(SF)     | |(London) | |(Berlin) |
         +---------+ +---------+ +---------+ +---------+
```

**How a CDN Request Works:**

1. User requests `https://cdn.example.com/images/logo.png`
2. DNS resolves `cdn.example.com` to the nearest CDN edge server (using GeoDNS/Anycast)
3. Edge server checks its cache:
   - **Cache HIT:** Returns the cached content immediately (fast!)
   - **Cache MISS:** Fetches from origin server, caches it, then returns to user
4. Subsequent requests from nearby users are served from the edge cache

---

### Push vs Pull CDN

**Pull CDN (Origin Pull) — Most Common:**

The CDN **pulls** content from the origin server on demand — only when a user requests it.

```
1. User requests /image.png from CDN edge
2. Edge doesn't have it (cache miss)
3. Edge fetches /image.png from origin server
4. Edge caches it and returns to user
5. Next user requesting /image.png gets it from edge cache (cache hit)
```

| Pros | Cons |
|------|------|
| Simple setup — just point DNS to CDN | First request for each asset is slow (cache miss) |
| Only caches what's actually requested | Origin must handle cache miss traffic |
| No need to manage what's on the CDN | Cache expiration can serve stale content |
| Automatic — CDN handles everything | Cold start after cache purge |

**Push CDN:**

You **push** content to the CDN proactively — uploading files directly to CDN storage before users request them.

```
1. You upload /image.png to CDN storage
2. CDN distributes it to all edge servers
3. User requests /image.png — already available at edge (always a cache hit)
```

| Pros | Cons |
|------|------|
| No cache miss — content is always available | Must manage what's uploaded to CDN |
| Origin server never receives traffic for CDN content | Storage costs for all content on all edges |
| Predictable performance | More complex deployment pipeline |
| Good for large, infrequently changing files | Must handle cache invalidation manually |

**When to Use Which:**

| Scenario | Recommendation |
|----------|---------------|
| Dynamic website with many assets | Pull CDN (automatic, simple) |
| Video streaming platform | Push CDN (pre-distribute large files) |
| Static site / documentation | Either (Push for predictability, Pull for simplicity) |
| Frequently updated content | Pull CDN (automatic refresh based on TTL) |
| Large files (videos, software downloads) | Push CDN (avoid origin load) |

**Most CDNs support both models.** In practice, Pull CDN is the default for most web applications.

---

### Edge Servers

**Edge servers** (also called Points of Presence — PoPs) are the CDN servers distributed around the world, located as close to end users as possible.

**What Edge Servers Do:**
- Cache and serve static content (images, CSS, JS, videos)
- Cache API responses (with appropriate cache headers)
- Terminate TLS connections (reducing latency for HTTPS)
- Compress content (gzip, brotli)
- Apply security rules (WAF, DDoS protection, bot detection)
- Execute edge functions (Cloudflare Workers, Lambda@Edge)

**Edge Computing:**
Modern CDNs go beyond caching — they can run code at the edge:

| Feature | Description | Example |
|---------|-------------|---------|
| **Edge Functions** | Run serverless functions at CDN edge | Cloudflare Workers, Lambda@Edge, Vercel Edge Functions |
| **Edge Rendering** | Render HTML at the edge (SSR at edge) | Next.js on Vercel Edge |
| **Edge Databases** | Distributed databases at the edge | Cloudflare D1, Turso |
| **A/B Testing** | Route users to different versions at the edge | Feature flags at edge |
| **Personalization** | Customize content based on location, device | Geo-targeted content |
| **Auth at Edge** | Validate JWT tokens at the edge | Reject unauthorized requests before hitting origin |

---

### Cache Invalidation in CDN

Cache invalidation is one of the hardest problems in computer science. With CDNs, it's even harder because content is cached across hundreds of edge servers worldwide.

**Strategies:**

**1. TTL-Based Expiration (Time To Live):**
```
Cache-Control: max-age=3600    → Edge caches for 1 hour
Cache-Control: s-maxage=86400  → CDN caches for 1 day (overrides max-age for shared caches)
```
- Simple and automatic
- Content may be stale until TTL expires
- Good for content that changes infrequently

**2. Cache Purge / Invalidation:**
```
# Purge a specific URL
POST /purge
{ "url": "https://cdn.example.com/images/logo.png" }

# Purge by pattern (wildcard)
POST /purge
{ "pattern": "https://cdn.example.com/images/*" }

# Purge everything
POST /purge-all
```
- Immediate invalidation across all edge servers
- Most CDNs provide APIs and dashboards for purging
- Purge propagation takes seconds to minutes (not instant)

**3. Versioned URLs (Cache Busting):**
```
# Instead of purging, change the URL:
/styles.css?v=1.0    → /styles.css?v=1.1
/bundle.abc123.js    → /bundle.def456.js  (content hash in filename)
```
- No purging needed — new URL = new cache entry
- Old versions naturally expire based on TTL
- Best practice for static assets (CSS, JS, images)
- Build tools (Webpack, Vite) generate hashed filenames automatically

**4. Stale-While-Revalidate:**
```
Cache-Control: max-age=60, stale-while-revalidate=3600
```
- Serve stale content immediately while fetching fresh content in the background
- User gets fast response (stale), next user gets fresh content
- Great balance between freshness and performance

**Best Practices:**
- Use **versioned URLs** for static assets (CSS, JS, images) — set very long TTL (1 year)
- Use **short TTL** for API responses that change frequently
- Use **stale-while-revalidate** for content where slight staleness is acceptable
- Use **cache purge** for urgent updates (security patches, critical content changes)
- Never cache **personalized content** (user profiles, shopping carts) at the CDN level

---

### Popular CDN Providers

| Provider | Key Features | Best For |
|----------|-------------|----------|
| **CloudFront (AWS)** | Deep AWS integration, Lambda@Edge, global network | AWS-native architectures |
| **Cloudflare** | Free tier, Workers (edge compute), DDoS protection, DNS | General purpose, security-focused |
| **Akamai** | Largest network (350K+ servers), enterprise features | Enterprise, media streaming |
| **Fastly** | Real-time purging (~150ms), VCL/Compute@Edge | Real-time content, API caching |
| **Google Cloud CDN** | GCP integration, Anycast, Cloud Armor | GCP-native architectures |
| **Azure CDN** | Azure integration, multiple CDN providers | Azure-native architectures |
| **Vercel Edge Network** | Optimized for Next.js, edge functions | Frontend/Jamstack applications |

**Choosing a CDN:**

| Consideration | Recommendation |
|--------------|---------------|
| AWS infrastructure | CloudFront |
| Security + free tier + edge compute | Cloudflare |
| Enterprise media streaming | Akamai |
| Need instant cache purging | Fastly |
| GCP infrastructure | Google Cloud CDN |
| Next.js / frontend apps | Vercel |

---


## 17.6 Domain Name System (DNS) in Architecture

> DNS is not just for resolving domain names to IP addresses — it's a powerful tool for **load balancing, geographic routing, and failover** in system design. By leveraging DNS strategically, you can route users to the nearest data center, distribute traffic across servers, and automatically redirect traffic away from failed infrastructure.

---

### DNS-Based Load Balancing

DNS can distribute traffic across multiple servers by returning different IP addresses for the same domain name.

**DNS Round-Robin:**

The simplest form of DNS load balancing — the DNS server returns multiple A records, rotating the order with each query.

```
Query: api.example.com

Response 1: [192.168.1.1, 192.168.1.2, 192.168.1.3]
Response 2: [192.168.1.2, 192.168.1.3, 192.168.1.1]
Response 3: [192.168.1.3, 192.168.1.1, 192.168.1.2]

Clients typically connect to the first IP in the list.
```

**Pros:**
- Simple to set up — just add multiple A records
- No special infrastructure needed
- Distributes traffic roughly evenly

**Cons:**
- No health checking — DNS keeps returning IPs of dead servers
- No awareness of server load — can't route to least-loaded server
- DNS caching means changes are slow to propagate
- Clients may cache the IP and always connect to the same server
- No session affinity

**Weighted DNS:**

Assign different weights to different servers — useful for gradual rollouts or servers with different capacities.

```
api.example.com:
  192.168.1.1  weight: 70   (70% of traffic — powerful server)
  192.168.1.2  weight: 20   (20% of traffic — medium server)
  192.168.1.3  weight: 10   (10% of traffic — canary deployment)
```

**Use Cases:**
- **Canary deployments:** Route 5% of traffic to the new version
- **Capacity-based routing:** Route more traffic to more powerful servers
- **Gradual migration:** Slowly shift traffic from old to new infrastructure

---

### GeoDNS

**GeoDNS** (Geographic DNS) returns different IP addresses based on the **geographic location** of the client making the DNS query. This routes users to the nearest data center, reducing latency.

```
User in New York → DNS query → Returns IP of US East data center
User in Tokyo    → DNS query → Returns IP of Asia Pacific data center
User in London   → DNS query → Returns IP of EU West data center
```

```
                    +------------------+
                    |   GeoDNS Server  |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
     +--------+------+ +----+-------+ +----+-------+
     | US East DC    | | EU West DC | | APAC DC    |
     | 54.x.x.x     | | 52.x.x.x  | | 13.x.x.x  |
     +---------------+ +------------+ +------------+
           ↑                  ↑              ↑
     Users in Americas   Users in Europe  Users in Asia
```

**How GeoDNS Works:**
1. Client makes a DNS query
2. DNS server determines the client's location (from the resolver's IP or EDNS Client Subnet)
3. Returns the IP address of the nearest data center
4. Client connects to the nearest server

**Implementation:**
- **AWS Route 53:** Geolocation routing, latency-based routing
- **Cloudflare DNS:** Geo-steering, load balancing
- **Google Cloud DNS:** Geolocation routing policies
- **NS1:** Advanced traffic management with geo-routing

**Geolocation vs Latency-Based Routing:**

| Approach | Routes Based On | Pros | Cons |
|----------|----------------|------|------|
| **Geolocation** | Client's geographic location | Predictable, compliance (data residency) | Nearest geographically ≠ lowest latency |
| **Latency-based** | Measured network latency | Optimal performance | Requires latency measurements, more complex |

**Key Point:** Latency-based routing is generally better for performance because geographic proximity doesn't always correlate with network latency (e.g., a user in Mexico might have lower latency to a US West server than a US East server, even though US East is geographically closer).

---

### DNS Failover

**DNS failover** automatically removes unhealthy servers from DNS responses and redirects traffic to healthy ones.

```
Normal Operation:
  api.example.com → [Server A: 10.0.1.1, Server B: 10.0.2.1]

Server A fails:
  Health check detects Server A is down
  DNS updated: api.example.com → [Server B: 10.0.2.1]
  All new DNS queries return only Server B

Server A recovers:
  Health check detects Server A is healthy
  DNS updated: api.example.com → [Server A: 10.0.1.1, Server B: 10.0.2.1]
```

**How DNS Failover Works:**

```
+------------------+
|  DNS Provider    |
|  (Route 53, etc.)|
+--------+---------+
         |
    Health Checks (every 10-30 seconds)
         |
    +----+----+----+
    |         |    |
+---+---+ +--+--+ +---+---+
|Server A| |Srvr B| |Server C|
| (UP)   | |(DOWN)| | (UP)   |
+--------+ +------+ +--------+

DNS Response: [Server A, Server C]  (Server B excluded)
```

**Configuration (AWS Route 53 Example):**
- Create health checks for each endpoint (HTTP, HTTPS, TCP)
- Configure failover routing policy:
  - **Primary:** Main server/region
  - **Secondary:** Backup server/region (used when primary fails health check)
- Set health check interval (10s or 30s)
- Set failure threshold (e.g., 3 consecutive failures = unhealthy)

**Active-Passive Failover:**
```
Primary (US East) ← All traffic normally goes here
Secondary (US West) ← Only receives traffic when primary is down

DNS health check detects primary is down → DNS switches to secondary
Primary recovers → DNS switches back to primary
```

**Active-Active Failover:**
```
Region A (US East) ← Receives traffic
Region B (EU West) ← Receives traffic
Region C (APAC)    ← Receives traffic

If Region A fails → Traffic redistributed to B and C
All regions are active simultaneously
```

**Limitations of DNS Failover:**

| Limitation | Impact | Mitigation |
|-----------|--------|-----------|
| **TTL delay** | Clients cache DNS for TTL duration — may still connect to dead server | Use low TTL (60s) for failover records |
| **Not instant** | Health check interval + TTL = minutes of downtime | Combine with application-level health checks |
| **Client caching** | Some clients ignore TTL and cache longer | Use short TTL + application-level retry logic |
| **No connection draining** | Existing connections aren't migrated | Use load balancer for graceful connection handling |

**Key Point for System Design:** DNS failover is a coarse-grained mechanism — it works at the DNS level with delays measured in seconds to minutes. For faster failover (milliseconds), use a **load balancer** with health checks. In practice, use **both**: DNS failover for region-level failover, load balancers for server-level failover.

---

### DNS in a Complete Architecture

Here's how DNS fits into a typical large-scale web architecture:

```
User types "www.example.com"
         |
         ↓
+------------------+
|    DNS (Route 53)|  ← GeoDNS routes to nearest region
+--------+---------+
         |
         ↓
+------------------+
|    CDN (CloudFront)| ← Serves cached static content
+--------+---------+
         |  (cache miss)
         ↓
+------------------+
| Load Balancer    | ← Distributes across application servers
| (ALB/NLB)        |
+--------+---------+
         |
    +----+----+
    |         |
+---+---+ +---+---+
|App Srv| |App Srv| ← Stateless application servers
|  A    | |  B    |
+---+---+ +---+---+
    |         |
    +----+----+
         |
+--------+---------+
|    Database      | ← Primary + Read Replicas
|    (RDS/Aurora)  |
+------------------+
```

**DNS responsibilities in this architecture:**
1. **GeoDNS** routes users to the nearest CDN edge / region
2. **CDN DNS** (CNAME) points to CDN provider's domain
3. **Internal DNS** resolves service names within the VPC (service discovery)
4. **Failover DNS** redirects traffic if an entire region goes down

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Client-Server | Fundamental web model — client sends request, server returns response |
| Stateless Servers | No server-side session state — enables horizontal scaling, simple load balancing |
| Stateful → Stateless | Externalize state to Redis/Memcached; use JWT for truly stateless auth |
| Thin vs Thick Client | SSR (server renders HTML) vs SPA (client renders with JS + API data) |
| Monolith | Single deployable unit — simple, fast to develop, right choice for most startups |
| Modular Monolith | Monolith with clean internal boundaries — best of both worlds |
| Monolith → Microservices | Start monolith, split only when you have a clear reason (scale, team, deployment) |
| N-Tier Architecture | Presentation → Business Logic → Data Access — separation of concerns |
| Dependency Rule | Dependencies point inward — controllers call services, services call repositories |
| Forward Proxy | Acts for clients — anonymity, content filtering, caching |
| Reverse Proxy | Acts for servers — load balancing, SSL termination, caching, security |
| Nginx vs HAProxy vs Envoy | Nginx (web server + proxy), HAProxy (load balancer), Envoy (service mesh) |
| CDN | Geographically distributed cache — reduces latency, offloads origin |
| Push vs Pull CDN | Pull (on-demand, most common) vs Push (pre-distribute, large files) |
| Cache Invalidation | Versioned URLs for assets, TTL for APIs, purge for urgent updates |
| DNS Load Balancing | Round-robin, weighted — simple but no health checks |
| GeoDNS | Route users to nearest data center based on location |
| DNS Failover | Remove unhealthy servers from DNS — coarse-grained, use with load balancers |

---

## Interview Tips for Module 17

1. **Stateless vs Stateful** — always design for stateless servers; explain how to externalize state
2. **Monolith vs Microservices** — don't default to microservices; explain when each is appropriate
3. **Modular Monolith** — a strong answer that shows nuance beyond the monolith/microservices binary
4. **N-Tier layers** — know what belongs in each layer; don't put business logic in controllers
5. **Forward vs Reverse Proxy** — know the difference and use cases for each
6. **CDN** — explain how CDNs work, push vs pull, and cache invalidation strategies
7. **Cache busting** — versioned URLs with content hashes is the best practice for static assets
8. **DNS in architecture** — explain GeoDNS, DNS failover, and how DNS fits into the overall system
9. **SSL termination** — explain why it's done at the reverse proxy / load balancer level
10. **Draw the full picture** — in HLD interviews, show DNS → CDN → Load Balancer → App Servers → Database


