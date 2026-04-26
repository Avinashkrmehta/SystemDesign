# Module 16: Application Layer Protocols & APIs

> APIs (Application Programming Interfaces) are the contracts that define how different software components communicate. In system design, choosing the right API paradigm — REST, GraphQL, or gRPC — and designing clean, scalable APIs is just as important as choosing the right database or caching strategy. This module covers the major API styles, their trade-offs, API gateway patterns, and best practices for designing APIs that are consistent, versioned, and production-ready.

> **Ruby Context:** Ruby has a rich ecosystem for building APIs. **Ruby on Rails** is the dominant framework for REST APIs (with `rails new --api` for API-only apps). **Grape** is a popular lightweight REST framework. For GraphQL, the **graphql-ruby** gem is the standard. For gRPC, the **grpc** gem provides full support. Ruby's expressiveness and convention-over-configuration philosophy make it particularly well-suited for rapid API development.

---

## 16.1 REST (Representational State Transfer)

> **REST** is an architectural style for designing networked applications. It was introduced by Roy Fielding in his 2000 doctoral dissertation. REST is not a protocol — it's a set of constraints that, when followed, produce scalable, stateless, and cacheable web services. REST over HTTP is the most common API style on the web today.

---

### REST Principles (Constraints)

REST defines **six architectural constraints**. An API that follows all of them is considered "RESTful":

**1. Client-Server Separation:**
- The client (UI) and server (data/logic) are independent
- They communicate only through the API
- Either side can evolve independently without breaking the other

**2. Statelessness:**
- Each request from the client must contain **all information** needed to process it
- The server does not store any client session state between requests
- Every request is self-contained

```
STATEFUL (bad for REST):
  Request 1: POST /login → Server stores session
  Request 2: GET /profile → Server uses stored session to identify user

STATELESS (RESTful):
  Request 1: POST /login → Server returns JWT token
  Request 2: GET /profile
             Authorization: Bearer <jwt-token>  ← client sends identity with every request
```

**Benefits of statelessness:**
- Any server can handle any request (easy horizontal scaling)
- No session affinity / sticky sessions needed
- Simpler server logic — no session management
- Better fault tolerance — server crash doesn't lose session state

> **Ruby Context:** Rails traditionally uses cookie-based sessions (stateful), but for REST APIs, the convention is to use token-based authentication. The **devise-jwt** or **doorkeeper** gems handle JWT/OAuth2 token flows. When building API-only Rails apps (`rails new myapp --api`), session middleware is excluded by default, encouraging stateless design.

**3. Cacheability:**
- Responses must explicitly indicate whether they are cacheable or not
- Cacheable responses can be reused by clients and intermediaries (CDN, proxy)
- Uses HTTP caching headers: `Cache-Control`, `ETag`, `Expires`, `Last-Modified`

**4. Uniform Interface:**
The most fundamental constraint — it simplifies and decouples the architecture. It has four sub-constraints:

| Sub-constraint | Meaning |
|---------------|---------|
| **Resource Identification** | Resources are identified by URIs (e.g., `/users/123`) |
| **Resource Manipulation through Representations** | Client manipulates resources via representations (JSON, XML) sent with HTTP methods |
| **Self-descriptive Messages** | Each message contains enough information to describe how to process it (Content-Type, status codes) |
| **HATEOAS** | Hypermedia as the Engine of Application State (see below) |

**5. Layered System:**
- The client cannot tell whether it's connected directly to the server or through intermediaries (load balancers, CDN, proxies)
- Layers can be added for security, caching, load balancing without changing the API

**6. Code on Demand (Optional):**
- Server can extend client functionality by sending executable code (e.g., JavaScript)
- This is the only optional constraint
- Rarely used in API design

---

### Resource Naming Conventions

In REST, everything is a **resource** identified by a **URI (Uniform Resource Identifier)**. Good resource naming is critical for a clean, intuitive API.

**Rules:**

| Rule | Good ✅ | Bad ❌ | Why |
|------|---------|--------|-----|
| Use nouns, not verbs | `/users` | `/getUsers` | HTTP methods provide the verb |
| Use plural nouns | `/users` | `/user` | Consistent — collection is plural |
| Use lowercase | `/users/123/orders` | `/Users/123/Orders` | URIs are case-sensitive; lowercase is convention |
| Use hyphens for readability | `/user-profiles` | `/user_profiles` or `/userProfiles` | Hyphens are URI-friendly |
| Nest for relationships | `/users/123/orders` | `/orders?userId=123` | Shows hierarchy (though query params are also valid) |
| No trailing slashes | `/users` | `/users/` | Trailing slashes create ambiguity |
| No file extensions | `/users/123` | `/users/123.json` | Use `Accept` header for content negotiation |

> **Ruby Context:** Rails routing conventions align closely with REST resource naming. `resources :users` in `config/routes.rb` automatically generates RESTful routes (`/users`, `/users/:id`, etc.). Rails uses underscores in path helpers (`user_orders_path`) but hyphens or underscores in URLs depending on configuration. Nested resources are expressed as `resources :users { resources :orders }`.

**Resource Hierarchy Examples:**

```
GET    /users                    → List all users
GET    /users/123                → Get user 123
POST   /users                    → Create a new user
PUT    /users/123                → Replace user 123
PATCH  /users/123                → Partially update user 123
DELETE /users/123                → Delete user 123

GET    /users/123/orders         → List orders for user 123
GET    /users/123/orders/456     → Get order 456 for user 123
POST   /users/123/orders         → Create an order for user 123
```

**Actions that don't map to CRUD:**

Sometimes you need endpoints for actions that aren't simple CRUD. Common approaches:

```
# Approach 1: Use a sub-resource
POST /users/123/activate         → Activate user 123
POST /orders/456/cancel          → Cancel order 456

# Approach 2: Use a verb endpoint (pragmatic, not strictly RESTful)
POST /users/123/send-verification-email

# Approach 3: Treat the action as a resource
POST /user-activations           → { "userId": 123 }
POST /order-cancellations        → { "orderId": 456 }
```

> **Ruby Context:** In Rails, custom actions are defined with `member` and `collection` routes:
> ```ruby
> resources :users do
>   member do
>     post :activate
>     post :send_verification_email
>   end
> end
> resources :orders do
>   member do
>     post :cancel
>   end
> end
> ```

---

### HATEOAS (Hypermedia as the Engine of Application State)

**HATEOAS** is the most advanced (and most debated) REST constraint. It means that API responses should include **links** to related actions and resources, so the client can discover what it can do next by following links — rather than hardcoding URLs.

**Example without HATEOAS:**
```json
{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com",
  "status": "active"
}
```
The client must know in advance that to get Alice's orders, it should call `GET /users/123/orders`.

**Example with HATEOAS:**
```json
{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com",
  "status": "active",
  "_links": {
    "self": { "href": "/users/123" },
    "orders": { "href": "/users/123/orders" },
    "deactivate": { "href": "/users/123/deactivate", "method": "POST" },
    "update": { "href": "/users/123", "method": "PUT" }
  }
}
```

**Benefits:**
- Client doesn't need to hardcode URLs — follows links from responses
- Server can change URL structure without breaking clients
- API is self-documenting and discoverable

**Reality:**
- Most real-world REST APIs do **not** implement full HATEOAS
- It adds complexity and payload size
- Useful for public APIs where discoverability matters
- Internal microservice APIs rarely need it

> **Ruby Context:** The **roar** gem and **jsonapi-serializer** (formerly fast_jsonapi) support HATEOAS-style links in Ruby. The JSON:API specification (implemented by the **jsonapi-resources** gem) includes link generation by default. Rails URL helpers (`user_url(user)`) make generating absolute links straightforward.

---

### Versioning

APIs evolve over time. Versioning ensures that changes don't break existing clients.

**Versioning Strategies:**

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URL Path** | `GET /v1/users` | Simple, explicit, easy to route | Pollutes URL, hard to sunset |
| **Query Parameter** | `GET /users?version=1` | Doesn't change URL structure | Easy to miss, not standard |
| **Custom Header** | `X-API-Version: 1` | Clean URLs | Hidden, harder to test in browser |
| **Accept Header** (Content Negotiation) | `Accept: application/vnd.myapi.v1+json` | Most RESTful, clean URLs | Complex, harder to implement |

**URL Path Versioning (most common):**
```
GET /v1/users/123    → Version 1 response format
GET /v2/users/123    → Version 2 response format (may have different fields)
```

**When to version:**
- Breaking changes to response format (removing/renaming fields)
- Breaking changes to request format
- Changing behavior of existing endpoints

**When NOT to version:**
- Adding new optional fields to responses (backward compatible)
- Adding new endpoints
- Bug fixes

**Best Practice:** Use URL path versioning for simplicity. Support at most 2-3 versions simultaneously. Deprecate old versions with clear timelines and `Sunset` headers.

> **Ruby Context:** In Rails, URL path versioning is typically implemented with namespace routing:
> ```ruby
> # config/routes.rb
> namespace :v1 do
>   resources :users
> end
> namespace :v2 do
>   resources :users
> end
> ```
> Controllers live in `app/controllers/v1/users_controller.rb` and `app/controllers/v2/users_controller.rb`. The **versionist** gem provides additional versioning strategies (header-based, Accept header, etc.).

---

### Pagination

When a collection has many items, returning all of them in one response is impractical. Pagination breaks the results into manageable pages.

**Offset-Based Pagination:**

```
GET /users?offset=0&limit=20     → Users 1-20
GET /users?offset=20&limit=20    → Users 21-40
GET /users?offset=40&limit=20    → Users 41-60
```

```json
{
  "data": [ ... ],
  "pagination": {
    "offset": 20,
    "limit": 20,
    "total": 1000
  }
}
```

| Pros | Cons |
|------|------|
| Simple to implement | Inconsistent results if data changes between pages (items shift) |
| Can jump to any page | Slow for large offsets (`OFFSET 100000` is expensive in SQL) |
| Total count available | Requires counting total rows (expensive for large tables) |

**Cursor-Based Pagination:**

```
GET /users?limit=20                          → First 20 users
GET /users?limit=20&cursor=eyJpZCI6MjB9     → Next 20 users after cursor
```

```json
{
  "data": [ ... ],
  "pagination": {
    "next_cursor": "eyJpZCI6NDB9",
    "has_more": true
  }
}
```

The cursor is typically an encoded reference to the last item (e.g., base64-encoded `{"id": 20}`).

| Pros | Cons |
|------|------|
| Consistent results even if data changes | Cannot jump to arbitrary page |
| Efficient for large datasets (uses `WHERE id > cursor`) | No total count (without extra query) |
| No performance degradation with deep pages | Cursor is opaque — harder to debug |

**When to use which:**

| Scenario | Recommendation |
|----------|---------------|
| Admin dashboards, small datasets | Offset-based (users expect page numbers) |
| Infinite scroll, feeds, timelines | Cursor-based (consistent, performant) |
| Real-time data, frequently changing | Cursor-based (offset would skip/duplicate items) |
| Large datasets (millions of rows) | Cursor-based (offset becomes slow) |

> **Ruby Context:** Rails provides built-in pagination support through gems. **Kaminari** and **will_paginate** are the most popular for offset-based pagination. For cursor-based pagination, **pagy** (the fastest pagination gem) supports both offset and cursor modes. In Rails controllers:
> ```ruby
> # Offset-based with Kaminari
> @users = User.page(params[:page]).per(20)
>
> # Cursor-based with Pagy
> @pagy, @users = pagy_cursor(User.order(:id), after: params[:cursor])
> ```

---

### Filtering, Sorting, Searching

**Filtering:**
```
GET /users?status=active                     → Filter by status
GET /users?role=admin&status=active          → Multiple filters (AND)
GET /orders?created_after=2025-01-01         → Date range filter
GET /products?price_min=10&price_max=100     → Range filter
GET /users?status=active,inactive            → Multiple values (OR)
```

**Sorting:**
```
GET /users?sort=name                         → Sort by name ascending (default)
GET /users?sort=-created_at                  → Sort by created_at descending (- prefix)
GET /users?sort=status,-name                 → Sort by status ASC, then name DESC
```

Alternative syntax:
```
GET /users?sort_by=name&sort_order=desc
```

**Searching:**
```
GET /users?search=alice                      → Full-text search across relevant fields
GET /users?q=alice                           → Short form (common convention)
GET /products?search=laptop&category=electronics  → Search + filter combined
```

**Best Practice:** Keep filtering, sorting, and searching as query parameters. Don't create separate endpoints for each filter combination.

> **Ruby Context:** The **ransack** gem provides powerful filtering and sorting for Rails. **pg_search** adds PostgreSQL full-text search. For API-specific filtering, **has_scope** or custom query objects are common patterns:
> ```ruby
> # Using Ransack in a controller
> @q = User.ransack(params[:q])
> @users = @q.result.page(params[:page])
>
> # Custom query object pattern
> class UserFilter
>   def initialize(params)
>     @scope = User.all
>     @scope = @scope.where(status: params[:status]) if params[:status]
>     @scope = @scope.where(role: params[:role]) if params[:role]
>     @scope = @scope.order(params[:sort] || :created_at)
>   end
>
>   def results
>     @scope
>   end
> end
> ```

---

### Rate Limiting Headers

Rate limiting protects your API from abuse and ensures fair usage. The server communicates rate limit status through response headers:

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000          → Maximum requests allowed in the window
X-RateLimit-Remaining: 742       → Requests remaining in the current window
X-RateLimit-Reset: 1625097600    → Unix timestamp when the window resets

--- When limit is exceeded: ---

HTTP/1.1 429 Too Many Requests
Retry-After: 60                  → Seconds until the client can retry
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1625097600
```

**Common Rate Limiting Strategies:**

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Per-user | Each authenticated user gets N requests/window | User-facing APIs |
| Per-API-key | Each API key gets N requests/window | Third-party integrations |
| Per-IP | Each IP address gets N requests/window | Unauthenticated endpoints |
| Per-endpoint | Different limits for different endpoints | Expensive operations get lower limits |
| Tiered | Different limits based on subscription plan | SaaS APIs (free vs paid tiers) |

**Rate Limiting Algorithms** (covered in detail in Module 10):
- **Token Bucket:** Tokens refill at a steady rate; each request consumes a token
- **Sliding Window Log:** Track timestamps of all requests; count within the window
- **Sliding Window Counter:** Approximate count using current and previous window weights
- **Fixed Window Counter:** Count requests in fixed time windows (simple but has burst issues at window boundaries)

> **Ruby Context:** The **rack-attack** gem is the standard for rate limiting in Ruby/Rails applications. It provides throttling, blocklisting, and safelisting as Rack middleware:
> ```ruby
> # config/initializers/rack_attack.rb
> Rack::Attack.throttle("api/ip", limit: 1000, period: 1.hour) do |req|
>   req.ip if req.path.start_with?("/api/")
> end
>
> Rack::Attack.throttle("api/token", limit: 5000, period: 1.hour) do |req|
>   req.env["HTTP_AUTHORIZATION"]&.split(" ")&.last if req.path.start_with?("/api/")
> end
>
> # Custom response with rate limit headers
> Rack::Attack.throttled_responder = lambda do |req|
>   match_data = req.env["rack.attack.match_data"]
>   headers = {
>     "X-RateLimit-Limit" => match_data[:limit].to_s,
>     "X-RateLimit-Remaining" => "0",
>     "Retry-After" => match_data[:period].to_s
>   }
>   [429, headers, ["Rate limit exceeded. Retry later.\n"]]
> end
> ```

---

### Idempotency

An operation is **idempotent** if performing it multiple times produces the same result as performing it once. This is critical for reliability — if a network error occurs, the client can safely retry without causing duplicate side effects.

**HTTP Method Idempotency:**

| Method | Idempotent? | Explanation |
|--------|------------|-------------|
| GET | Yes | Reading data doesn't change state |
| PUT | Yes | Replacing a resource with the same data yields the same result |
| DELETE | Yes | Deleting an already-deleted resource is a no-op |
| PATCH | No* | Depends on implementation (e.g., `{"increment": 1}` is not idempotent) |
| POST | No | Creating a resource twice creates two resources |

**Making POST Idempotent with Idempotency Keys:**

```
# Client generates a unique key for each logical operation
POST /payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "amount": 100.00,
  "currency": "USD",
  "recipient": "user_456"
}
```

**How it works:**
1. Client generates a unique `Idempotency-Key` (typically a UUID) for each operation
2. Server checks if it has seen this key before
3. If **new key:** Process the request, store the key + response
4. If **duplicate key:** Return the stored response without re-processing

```
First request:  POST /payments (Key: abc-123) → 201 Created (payment processed)
Retry request:  POST /payments (Key: abc-123) → 201 Created (same response, NOT processed again)
New request:    POST /payments (Key: def-456) → 201 Created (new payment processed)
```

**Key Point for System Design:** Idempotency is essential for payment systems, order processing, and any operation where duplicates cause real-world harm. Stripe, PayPal, and AWS all use idempotency keys.

**Idempotency Key Storage and Lifecycle:**

```
+--------+                    +------------+                  +-----------+
| Client |  POST /payments    | API Server |  Check key       | Key Store |
|        | ---- Key: abc ---> |            | ---- abc? -----> | (Redis)   |
|        |                    |            | <--- NOT FOUND -- |           |
|        |                    |            |                   |           |
|        |                    | Process payment...             |           |
|        |                    |            |                   |           |
|        |                    |            | ---- Store -----> |           |
|        |                    |            |  key=abc          |           |
|        |                    |            |  response=201     |           |
|        |                    |            |  ttl=24h          |           |
|        | <--- 201 Created - |            |                   |           |
+--------+                    +------------+                  +-----------+

Retry (network error, client didn't get response):
+--------+                    +------------+                  +-----------+
| Client |  POST /payments    | API Server |  Check key       | Key Store |
|        | ---- Key: abc ---> |            | ---- abc? -----> | (Redis)   |
|        |                    |            | <--- FOUND ---    |           |
|        |                    |            |  response=201     |           |
|        | <--- 201 Created - |            |                   |           |
|        |  (same response,   |            |  (NOT processed   |           |
|        |   no duplicate)    |            |   again)          |           |
+--------+                    +------------+                  +-----------+
```

**Implementation Considerations:**
- Store idempotency keys in Redis with a TTL (typically 24-48 hours)
- The stored value should include the full response (status code, headers, body)
- Handle concurrent requests with the same key using distributed locks
- Return the stored response for duplicate keys, even if the original request failed (store error responses too)
- Idempotency keys should be scoped per-user or per-API-key to prevent cross-user conflicts

> **Ruby Context:** Implementing idempotency in Rails typically uses Redis for key storage. The **idempotent-request** gem provides middleware-level support. A manual implementation:
> ```ruby
> class IdempotencyMiddleware
>   def initialize(app)
>     @app = app
>     @redis = Redis.new
>   end
>
>   def call(env)
>     key = env["HTTP_IDEMPOTENCY_KEY"]
>     return @app.call(env) unless key
>
>     cached = @redis.get("idempotency:#{key}")
>     return JSON.parse(cached) if cached
>
>     status, headers, body = @app.call(env)
>     response = [status, headers, body.to_a]
>     @redis.setex("idempotency:#{key}", 86_400, response.to_json)
>     [status, headers, body]
>   end
> end
> ```

---

### Designing a Complete REST API — Practical Example

Here's a complete REST API design for an e-commerce order system, demonstrating all the concepts above:

**Resource Hierarchy:**
```
/v1/users
/v1/users/{userId}
/v1/users/{userId}/orders
/v1/users/{userId}/orders/{orderId}
/v1/users/{userId}/orders/{orderId}/items
/v1/products
/v1/products/{productId}
/v1/products/{productId}/reviews
```

**Endpoint Design:**

```
# Users
GET    /v1/users?status=active&sort=-created_at&limit=20&cursor=eyJpZCI6MTAwfQ
POST   /v1/users
GET    /v1/users/{userId}
PATCH  /v1/users/{userId}
DELETE /v1/users/{userId}

# Orders
GET    /v1/users/{userId}/orders?status=pending&sort=-created_at
POST   /v1/users/{userId}/orders
       Idempotency-Key: <uuid>
GET    /v1/users/{userId}/orders/{orderId}
PATCH  /v1/users/{userId}/orders/{orderId}

# Order Actions (non-CRUD)
POST   /v1/users/{userId}/orders/{orderId}/cancel
POST   /v1/users/{userId}/orders/{orderId}/refund

# Products
GET    /v1/products?category=electronics&price_min=10&price_max=500&search=laptop&sort=price
GET    /v1/products/{productId}
```

> **Ruby Context:** In Rails, this resource hierarchy maps to routes and controllers:
> ```ruby
> # config/routes.rb
> namespace :v1 do
>   resources :users do
>     resources :orders do
>       member do
>         post :cancel
>         post :refund
>       end
>       resources :items, only: [:index]
>     end
>   end
>   resources :products, only: [:index, :show] do
>     resources :reviews
>   end
> end
> ```

**Request/Response Examples:**

```
POST /v1/users/user_123/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
X-Request-Id: req_abc123

{
  "items": [
    { "product_id": "prod_456", "quantity": 2 },
    { "product_id": "prod_789", "quantity": 1 }
  ],
  "shipping_address_id": "addr_012",
  "payment_method_id": "pm_345"
}
```

```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /v1/users/user_123/orders/order_678
X-Request-Id: req_abc123
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 997
X-RateLimit-Reset: 1625097600

{
  "id": "order_678",
  "user_id": "user_123",
  "status": "pending",
  "items": [
    {
      "product_id": "prod_456",
      "name": "Wireless Mouse",
      "quantity": 2,
      "unit_price": 29.99,
      "subtotal": 59.98
    },
    {
      "product_id": "prod_789",
      "name": "USB-C Cable",
      "quantity": 1,
      "unit_price": 12.99,
      "subtotal": 12.99
    }
  ],
  "subtotal": 72.97,
  "tax": 6.57,
  "shipping": 5.99,
  "total": 85.53,
  "currency": "USD",
  "created_at": "2025-03-15T14:30:00Z",
  "updated_at": "2025-03-15T14:30:00Z"
}
```

> **Ruby Context:** In a Rails controller, this would be handled with serializers for consistent JSON output:
> ```ruby
> # app/controllers/v1/orders_controller.rb
> module V1
>   class OrdersController < ApplicationController
>     before_action :authenticate_user!
>
>     def create
>       order = OrderCreationService.new(
>         user: current_user,
>         items: order_params[:items],
>         shipping_address_id: order_params[:shipping_address_id],
>         payment_method_id: order_params[:payment_method_id]
>       ).call
>
>       render json: OrderSerializer.new(order).serializable_hash,
>              status: :created,
>              location: v1_user_order_url(current_user, order)
>     end
>
>     private
>
>     def order_params
>       params.permit(items: [:product_id, :quantity],
>                     :shipping_address_id, :payment_method_id)
>     end
>   end
> end
> ```

**Error Response Example:**

```
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json
X-Request-Id: req_def456

{
  "error": {
    "code": "INSUFFICIENT_INVENTORY",
    "message": "One or more items do not have sufficient inventory.",
    "details": [
      {
        "field": "items[0].quantity",
        "message": "Requested 2 units of prod_456, but only 1 available.",
        "product_id": "prod_456",
        "requested": 2,
        "available": 1
      }
    ],
    "request_id": "req_def456",
    "documentation_url": "https://api.example.com/docs/errors#INSUFFICIENT_INVENTORY"
  }
}
```

> **Ruby Context:** Rails uses `rescue_from` for consistent error handling across controllers:
> ```ruby
> class ApplicationController < ActionController::API
>   rescue_from ActiveRecord::RecordNotFound, with: :not_found
>   rescue_from InsufficientInventoryError, with: :unprocessable
>
>   private
>
>   def not_found(exception)
>     render json: {
>       error: {
>         code: "NOT_FOUND",
>         message: exception.message,
>         request_id: request.request_id
>       }
>     }, status: :not_found
>   end
>
>   def unprocessable(exception)
>     render json: {
>       error: {
>         code: exception.code,
>         message: exception.message,
>         details: exception.details,
>         request_id: request.request_id
>       }
>     }, status: :unprocessable_entity
>   end
> end
> ```

---


## 16.2 GraphQL

> **GraphQL** is a query language for APIs and a runtime for executing those queries. Developed by Facebook in 2012 (open-sourced in 2015), GraphQL gives clients the power to request **exactly the data they need** — no more, no less. Unlike REST, where the server defines the response shape, in GraphQL the **client** defines the shape of the response.

> **Ruby Context:** The **graphql-ruby** gem (by Robert Mosolgo) is the standard GraphQL implementation for Ruby. It integrates seamlessly with Rails and provides a class-based DSL for defining schemas, types, and resolvers. Install with `gem 'graphql'` and generate boilerplate with `rails generate graphql:install`.

---

### Schema Definition Language (SDL)

The **Schema Definition Language (SDL)** is how you define the types, queries, mutations, and subscriptions available in a GraphQL API. The schema is the contract between client and server.

**Type Definitions:**

```graphql
# Object type — represents a resource
type User {
  id: ID!                    # ! means non-nullable
  name: String!
  email: String!
  age: Int
  posts: [Post!]!            # List of non-nullable Posts (list itself is non-nullable)
  profile: Profile
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  createdAt: String!
}

type Comment {
  id: ID!
  text: String!
  author: User!
}

type Profile {
  bio: String
  avatarUrl: String
  website: String
}
```

> **Ruby Context:** In graphql-ruby, types are defined as Ruby classes:
> ```ruby
> # app/graphql/types/user_type.rb
> module Types
>   class UserType < Types::BaseObject
>     field :id, ID, null: false
>     field :name, String, null: false
>     field :email, String, null: false
>     field :age, Integer, null: true
>     field :posts, [Types::PostType], null: false
>     field :profile, Types::ProfileType, null: true
>   end
> end
>
> # app/graphql/types/post_type.rb
> module Types
>   class PostType < Types::BaseObject
>     field :id, ID, null: false
>     field :title, String, null: false
>     field :content, String, null: false
>     field :author, Types::UserType, null: false
>     field :comments, [Types::CommentType], null: false
>     field :created_at, GraphQL::Types::ISO8601DateTime, null: false
>   end
> end
> ```

**Scalar Types:**

| Type | Description | Example |
|------|-------------|---------|
| `Int` | 32-bit integer | `42` |
| `Float` | Double-precision floating point | `3.14` |
| `String` | UTF-8 string | `"hello"` |
| `Boolean` | true/false | `true` |
| `ID` | Unique identifier (serialized as String) | `"user_123"` |

**Custom Scalars:**
```graphql
scalar DateTime
scalar JSON
scalar URL
```

> **Ruby Context:** graphql-ruby provides built-in custom scalars like `GraphQL::Types::ISO8601DateTime` and `GraphQL::Types::JSON`. Custom scalars are defined as:
> ```ruby
> class Types::Url < Types::BaseScalar
>   def self.coerce_input(input_value, _context)
>     URI.parse(input_value)
>   rescue URI::InvalidURIError
>     raise GraphQL::CoercionError, "#{input_value.inspect} is not a valid URL"
>   end
>
>   def self.coerce_result(ruby_value, _context)
>     ruby_value.to_s
>   end
> end
> ```

**Enums:**
```graphql
enum Role {
  ADMIN
  USER
  MODERATOR
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}
```

> **Ruby Context:** Enums in graphql-ruby:
> ```ruby
> class Types::RoleEnum < Types::BaseEnum
>   value "ADMIN", "Administrator role"
>   value "USER", "Regular user role"
>   value "MODERATOR", "Moderator role"
> end
> ```

**Input Types (for mutations):**
```graphql
input CreateUserInput {
  name: String!
  email: String!
  age: Int
  role: Role = USER    # default value
}

input UpdateUserInput {
  name: String
  email: String
  age: Int
}
```

> **Ruby Context:** Input types in graphql-ruby:
> ```ruby
> class Types::CreateUserInput < Types::BaseInputObject
>   argument :name, String, required: true
>   argument :email, String, required: true
>   argument :age, Integer, required: false
>   argument :role, Types::RoleEnum, required: false, default_value: "USER"
> end
> ```

**Interfaces and Union Types:**
```graphql
# Interface — shared fields across types
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String!
}

type Post implements Node {
  id: ID!
  title: String!
}

# Union — one of several types (no shared fields required)
union SearchResult = User | Post | Comment

type Query {
  search(query: String!): [SearchResult!]!
}
```

> **Ruby Context:** Interfaces and unions in graphql-ruby:
> ```ruby
> module Types::NodeInterface
>   include Types::BaseInterface
>   field :id, ID, null: false
>
>   orphan_types Types::UserType, Types::PostType
> end
>
> class Types::SearchResultUnion < Types::BaseUnion
>   possible_types Types::UserType, Types::PostType, Types::CommentType
>
>   def self.resolve_type(object, _context)
>     case object
>     when User then Types::UserType
>     when Post then Types::PostType
>     when Comment then Types::CommentType
>     end
>   end
> end
> ```

---

### Queries, Mutations, Subscriptions

GraphQL has three root operation types:

**1. Queries — Reading Data:**

```graphql
# Schema definition
type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  post(id: ID!): Post
}
```

```graphql
# Client query — request exactly what you need
query GetUser {
  user(id: "123") {
    name
    email
    posts {
      title
      createdAt
    }
  }
}
```

```json
// Response — matches the query shape exactly
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@example.com",
      "posts": [
        { "title": "GraphQL Basics", "createdAt": "2025-01-15" },
        { "title": "REST vs GraphQL", "createdAt": "2025-02-20" }
      ]
    }
  }
}
```

**Key Advantage:** The client asked for `name`, `email`, and `posts.title` + `posts.createdAt` — and that's exactly what it got. No over-fetching of unused fields.

> **Ruby Context:** In graphql-ruby, query fields are defined in the QueryType:
> ```ruby
> # app/graphql/types/query_type.rb
> module Types
>   class QueryType < Types::BaseObject
>     field :user, Types::UserType, null: true do
>       argument :id, ID, required: true
>     end
>
>     field :users, [Types::UserType], null: false do
>       argument :limit, Integer, required: false, default_value: 20
>       argument :offset, Integer, required: false, default_value: 0
>     end
>
>     def user(id:)
>       User.find_by(id: id)
>     end
>
>     def users(limit:, offset:)
>       User.limit(limit).offset(offset)
>     end
>   end
> end
> ```

**2. Mutations — Writing Data:**

```graphql
# Schema definition
type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  publishPost(id: ID!): Post!
}
```

```graphql
# Client mutation
mutation CreateNewUser {
  createUser(input: { name: "Bob", email: "bob@example.com", age: 30 }) {
    id
    name
    email
  }
}
```

```json
// Response
{
  "data": {
    "createUser": {
      "id": "456",
      "name": "Bob",
      "email": "bob@example.com"
    }
  }
}
```

> **Ruby Context:** Mutations in graphql-ruby are typically defined as separate classes:
> ```ruby
> # app/graphql/mutations/create_user.rb
> module Mutations
>   class CreateUser < Mutations::BaseMutation
>     argument :input, Types::CreateUserInput, required: true
>
>     field :user, Types::UserType, null: true
>     field :errors, [String], null: false
>
>     def resolve(input:)
>       user = User.new(input.to_h)
>       if user.save
>         { user: user, errors: [] }
>       else
>         { user: nil, errors: user.errors.full_messages }
>       end
>     end
>   end
> end
> ```

**3. Subscriptions — Real-Time Data:**

Subscriptions use WebSockets to push data to clients when events occur.

```graphql
# Schema definition
type Subscription {
  messageAdded(channelId: ID!): Message!
  userStatusChanged(userId: ID!): UserStatus!
}
```

```graphql
# Client subscription
subscription OnNewMessage {
  messageAdded(channelId: "general") {
    id
    text
    author {
      name
    }
    createdAt
  }
}
```

When a new message is added to the "general" channel, the server pushes it to all subscribed clients in real-time.

> **Ruby Context:** graphql-ruby supports subscriptions via **ActionCable** (Rails WebSocket framework) or **Pusher**:
> ```ruby
> # app/graphql/types/subscription_type.rb
> module Types
>   class SubscriptionType < Types::BaseObject
>     field :message_added, Types::MessageType, null: false do
>       argument :channel_id, ID, required: true
>     end
>
>     def message_added(channel_id:)
>       # This is triggered when an event is published
>       object
>     end
>   end
> end
>
> # Triggering from anywhere in the app:
> MyAppSchema.subscriptions.trigger(:message_added, { channel_id: channel.id }, message)
> ```

---

### Resolvers

**Resolvers** are functions that populate the data for each field in the schema. Every field in a GraphQL schema has a corresponding resolver function.

**Resolver Structure:**

```
resolver(parent, args, context, info)
```

| Parameter | Description |
|-----------|-------------|
| `parent` | The result of the parent resolver (for nested fields) |
| `args` | Arguments passed to the field (e.g., `id` in `user(id: "123")`) |
| `context` | Shared data across all resolvers (auth info, database connections, dataloaders) |
| `info` | Information about the query (field name, schema, etc.) |

**Example Resolver Chain:**

```
Query: user(id: "123") → User Resolver → fetches user from DB
  └── User.posts → Posts Resolver → fetches posts for this user
       └── Post.author → Author Resolver → fetches author for each post
            └── User.profile → Profile Resolver → fetches profile
```

> **Ruby Context:** In graphql-ruby, resolvers can be inline methods on type classes or standalone resolver classes:

```ruby
# Inline resolver (method on the type class)
class Types::UserType < Types::BaseObject
  field :posts, [Types::PostType], null: false

  def posts
    # `object` is the parent (the User instance)
    object.posts.published
  end
end

# Standalone resolver class (for complex logic)
class Resolvers::UserResolver < Resolvers::BaseResolver
  type Types::UserType, null: true
  argument :id, ID, required: true

  def resolve(id:)
    User.find_by(id: id)
  end
end

# Using the standalone resolver in QueryType
class Types::QueryType < Types::BaseObject
  field :user, resolver: Resolvers::UserResolver
end
```

> In graphql-ruby, `object` refers to the parent value, keyword arguments replace `args`, and `context` is available as a method. The `info` parameter is accessible but rarely needed directly.

---

### N+1 Problem and DataLoader

The **N+1 problem** is the most common performance issue in GraphQL. It occurs when resolving a list of items triggers a separate database query for each item's related data.

**The Problem:**

```graphql
query {
  users {          # 1 query: SELECT * FROM users
    name
    posts {        # N queries: SELECT * FROM posts WHERE author_id = ? (one per user)
      title
    }
  }
}
```

If there are 100 users, this executes **101 queries** (1 for users + 100 for posts) — the N+1 problem.

**The Solution — DataLoader:**

DataLoader **batches** and **caches** database requests within a single request cycle.

```
Without DataLoader:
  User 1 → SELECT * FROM posts WHERE author_id = 1
  User 2 → SELECT * FROM posts WHERE author_id = 2
  User 3 → SELECT * FROM posts WHERE author_id = 3
  ... (100 separate queries)

With DataLoader:
  Collect all user IDs: [1, 2, 3, ..., 100]
  Single batch query: SELECT * FROM posts WHERE author_id IN (1, 2, 3, ..., 100)
  Distribute results to each resolver
```

**How DataLoader Works:**

```
1. Resolver for User 1's posts calls: postLoader.load(1)
2. Resolver for User 2's posts calls: postLoader.load(2)
3. Resolver for User 3's posts calls: postLoader.load(3)
   ... (all within the same tick of the event loop)

4. DataLoader batches all IDs: [1, 2, 3, ..., N]
5. Executes ONE batch function: SELECT * FROM posts WHERE author_id IN (1, 2, 3, ..., N)
6. Distributes results back to each resolver

Result: 1 + 1 = 2 queries instead of 1 + N = 101 queries
```

> **Ruby Context:** graphql-ruby provides built-in **GraphQL::Dataloader** (inspired by Facebook's DataLoader). It uses Ruby Fibers for lazy batching:
> ```ruby
> # app/graphql/sources/posts_by_author.rb
> class Sources::PostsByAuthor < GraphQL::Dataloader::Source
>   def fetch(author_ids)
>     posts = Post.where(author_id: author_ids).group_by(&:author_id)
>     author_ids.map { |id| posts[id] || [] }
>   end
> end
>
> # In the UserType
> class Types::UserType < Types::BaseObject
>   field :posts, [Types::PostType], null: false
>
>   def posts
>     dataloader.with(Sources::PostsByAuthor).load(object.id)
>   end
> end
> ```
> Alternatively, the **graphql-batch** gem (by Shopify) provides a similar batching mechanism with a Promise-based API. Rails' built-in `ActiveRecord::Associations::Preloader` can also help when used with `lookahead` to preload associations based on the query shape.

**Key Point:** DataLoader is essential for any production GraphQL server. Without it, GraphQL can be significantly slower than REST due to the N+1 problem.

---

### GraphQL vs REST (Trade-offs)

| Feature | REST | GraphQL |
|---------|------|---------|
| **Data fetching** | Server defines response shape | Client defines response shape |
| **Over-fetching** | Common (server returns all fields) | Eliminated (client requests only needed fields) |
| **Under-fetching** | Common (need multiple endpoints) | Eliminated (single query gets all related data) |
| **Endpoints** | Multiple (one per resource) | Single endpoint (`/graphql`) |
| **Versioning** | URL versioning (`/v1/`, `/v2/`) | Schema evolution (deprecate fields, add new ones) |
| **Caching** | Easy (HTTP caching, CDN) | Hard (POST requests, dynamic queries) |
| **Error handling** | HTTP status codes | Always 200 OK; errors in response body |
| **File upload** | Native (multipart/form-data) | Requires workarounds (multipart spec) |
| **Real-time** | Requires WebSocket/SSE separately | Built-in subscriptions |
| **Tooling** | Mature (Postman, curl, Swagger) | Growing (GraphiQL, Apollo DevTools) |
| **Learning curve** | Low | Medium-High |
| **Performance** | Predictable (fixed endpoints) | Can be unpredictable (complex nested queries) |
| **N+1 problem** | Controlled by server | Must be handled (DataLoader) |

**When to Use REST:**

| Scenario | Why REST |
|----------|---------|
| Simple CRUD APIs | REST maps naturally to CRUD operations |
| Public APIs | Easier to cache, document, and rate-limit |
| File-heavy operations | Native file upload/download support |
| Microservice-to-microservice | Simple, well-understood, cacheable |
| CDN-friendly content | HTTP caching works out of the box |

**When to Use GraphQL:**

| Scenario | Why GraphQL |
|----------|------------|
| Mobile apps (bandwidth-sensitive) | Request only needed fields — smaller payloads |
| Complex, nested data | Single query replaces multiple REST calls |
| Rapidly evolving frontend | Frontend can change data requirements without backend changes |
| Multiple client types | Different clients (web, mobile, TV) need different data shapes |
| Real-time features | Built-in subscription support |

**When to Use Both:**
Many large systems use REST for simple, cacheable endpoints and GraphQL for complex, client-driven data fetching. They're not mutually exclusive.

> **Ruby Context:** In the Ruby ecosystem, Rails is the dominant REST framework, while graphql-ruby powers GraphQL. Many Rails applications mount a GraphQL endpoint alongside REST routes:
> ```ruby
> # config/routes.rb
> Rails.application.routes.draw do
>   # REST endpoints
>   namespace :v1 do
>     resources :users
>     resources :products
>   end
>
>   # GraphQL endpoint (coexists with REST)
>   post "/graphql", to: "graphql#execute"
> end
> ```
> Shopify, GitHub, and many Ruby-based companies use graphql-ruby in production at scale.

---

### GraphQL Security Concerns

GraphQL's flexibility introduces unique security challenges that don't exist in REST:

**1. Query Depth Attacks:**
A malicious client can send deeply nested queries that cause exponential database load:

```graphql
# Malicious query — exponential depth
query {
  user(id: "1") {
    posts {
      author {
        posts {
          author {
            posts {
              author {
                # ... infinite nesting
              }
            }
          }
        }
      }
    }
  }
}
```

**Mitigation:** Set a maximum query depth limit (e.g., depth ≤ 10). Libraries like `graphql-depth-limit` enforce this.

> **Ruby Context:** graphql-ruby has built-in support for query depth limiting:
> ```ruby
> class MyAppSchema < GraphQL::Schema
>   max_depth 10
>   max_complexity 300
>
>   query Types::QueryType
>   mutation Types::MutationType
> end
> ```

**2. Query Complexity Attacks:**
Even without deep nesting, a query can request massive amounts of data:

```graphql
# Expensive query — requests thousands of records
query {
  users(limit: 10000) {
    posts(limit: 1000) {
      comments(limit: 1000) {
        author { name }
      }
    }
  }
}
```

**Mitigation:** Assign a **cost** to each field and set a maximum query cost:

| Field | Cost |
|-------|------|
| Scalar field (name, email) | 1 |
| Object field (profile) | 2 |
| List field (posts) | 2 × limit |
| Nested list (posts.comments) | 2 × parent_limit × child_limit |

Reject queries that exceed the cost budget (e.g., max cost = 1000).

> **Ruby Context:** graphql-ruby supports field-level complexity annotations:
> ```ruby
> class Types::UserType < Types::BaseObject
>   field :name, String, null: false, complexity: 1
>   field :posts, [Types::PostType], null: false, complexity: 10 do
>     argument :limit, Integer, required: false, default_value: 20
>   end
>
>   # Dynamic complexity based on arguments
>   field :comments, [Types::CommentType], null: false do
>     argument :limit, Integer, required: false, default_value: 10
>     complexity ->(ctx, args, child_complexity) {
>       args[:limit] * child_complexity
>     }
>   end
> end
> ```

**3. Introspection in Production:**
GraphQL supports introspection — clients can query the schema itself to discover all types, fields, and relationships. This is useful for development but exposes your entire API surface in production.

```graphql
# Introspection query — reveals entire schema
{
  __schema {
    types {
      name
      fields { name type { name } }
    }
  }
}
```

**Mitigation:** Disable introspection in production.

> **Ruby Context:** Disable introspection in graphql-ruby:
> ```ruby
> class MyAppSchema < GraphQL::Schema
>   # Disable introspection in production
>   disable_introspection_entry_points if Rails.env.production?
> end
> ```
> Alternatively, use the **graphql-schema_comparator** gem to track schema changes and the **graphql-guard** gem for field-level authorization, ensuring sensitive fields are protected even if introspection is enabled in staging environments.

**4. Additional Security Best Practices:**

| Concern | Mitigation |
|---------|-----------|
| **Authentication** | Validate tokens in context setup before resolvers execute |
| **Authorization** | Use field-level authorization (graphql-ruby's `authorized?` method or **pundit** integration) |
| **Persisted Queries** | Only allow pre-registered queries in production (prevents arbitrary queries) |
| **Timeout** | Set query execution timeouts to prevent long-running queries |
| **Rate Limiting** | Limit by query complexity, not just request count |

> **Ruby Context:** A complete security setup in graphql-ruby:
> ```ruby
> class MyAppSchema < GraphQL::Schema
>   max_depth 10
>   max_complexity 300
>   default_max_page_size 50
>
>   # Query timeout
>   use GraphQL::Schema::Timeout, max_seconds: 5
>
>   query Types::QueryType
>   mutation Types::MutationType
>   subscription Types::SubscriptionType
> end
>
> # Field-level authorization in types
> class Types::UserType < Types::BaseObject
>   field :email, String, null: false
>
>   def email
>     # Only show email to the user themselves or admins
>     if context[:current_user]&.admin? || context[:current_user]&.id == object.id
>       object.email
>     else
>       nil
>     end
>   end
> end
>
> # Context setup in the controller
> class GraphqlController < ApplicationController
>   def execute
>     context = {
>       current_user: current_user,
>       ip_address: request.remote_ip
>     }
>     result = MyAppSchema.execute(params[:query],
>       variables: params[:variables],
>       context: context,
>       operation_name: params[:operationName]
>     )
>     render json: result
>   end
> end
> ```

**4. Batching Attacks:**
GraphQL allows sending multiple operations in a single request:

```json
[
  { "query": "{ user(id: \"1\") { name } }" },
  { "query": "{ user(id: \"2\") { name } }" },
  { "query": "{ user(id: \"3\") { name } }" }
  // ... thousands of queries in one HTTP request
]
```

**Mitigation:** Limit the number of operations per request (e.g., max 10 queries per batch).

---

### GraphQL Caching Strategies

Caching is harder in GraphQL than REST because all queries go to a single `POST /graphql` endpoint with dynamic query bodies. HTTP caching (CDN, browser cache) doesn't work out of the box.

**Client-Side Caching (Normalized Cache):**
- Apollo Client and Relay use a **normalized cache** — they break responses into individual objects identified by `id` + `__typename`
- When a query returns `User { id: 1, name: "Alice" }`, it's stored as a cache entry
- Subsequent queries that reference User 1 get the cached data without a network request
- Cache updates propagate automatically to all components displaying that data

**Server-Side Caching:**

| Strategy | How It Works |
|----------|-------------|
| **Persisted Queries** | Client sends a hash of the query instead of the full query text. Server looks up the query by hash. Enables GET requests → HTTP caching works |
| **Automatic Persisted Queries (APQ)** | Client sends hash first; if server doesn't recognize it, client sends full query; server caches the mapping |
| **Response Caching** | Cache full responses keyed by query + variables. Works for public, non-personalized data |
| **Field-Level Caching** | Cache individual resolver results with TTLs. Fine-grained but complex |
| **CDN Caching with GET** | Use persisted queries with GET method → CDN can cache based on URL |

> **Ruby Context:** graphql-ruby supports persisted queries via the **graphql-persisted_queries** gem, which stores query mappings in Redis or Memcached. For server-side response caching, Rails' built-in caching (`Rails.cache`) integrates well with resolver-level caching:
> ```ruby
> class Types::ProductType < Types::BaseObject
>   field :reviews_summary, Types::ReviewsSummaryType, null: false
>
>   def reviews_summary
>     Rails.cache.fetch("product:#{object.id}:reviews_summary", expires_in: 1.hour) do
>       object.compute_reviews_summary
>     end
>   end
> end
> ```

---


## 16.3 gRPC

> **gRPC** (gRPC Remote Procedure Call) is a high-performance, open-source RPC framework developed by Google. It uses **Protocol Buffers (protobuf)** for serialization and **HTTP/2** for transport, making it significantly faster than REST for inter-service communication. gRPC is the go-to choice for low-latency microservice communication.

> **Ruby Context:** The **grpc** gem provides full gRPC support for Ruby, including client and server implementations. The **grpc-tools** gem includes the `protoc` compiler plugin for generating Ruby code from `.proto` files. While Ruby's gRPC performance is lower than Go or C++ implementations, it's suitable for many microservice use cases and integrates well with the Ruby ecosystem.

---

### Protocol Buffers (protobuf)

**Protocol Buffers** are Google's language-neutral, platform-neutral mechanism for serializing structured data. They serve as both the **Interface Definition Language (IDL)** and the **serialization format** for gRPC.

**Defining a Service (.proto file):**

```protobuf
syntax = "proto3";

package userservice;

// Service definition — defines the RPC methods
service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
  rpc DeleteUser (DeleteUserRequest) returns (Empty);
}

// Message definitions — define the data structures
message GetUserRequest {
  string id = 1;              // field number (not value) — used for binary encoding
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  int32 age = 3;
  Role role = 4;
}

message UserResponse {
  string id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  Role role = 5;
  string created_at = 6;
}

message ListUsersRequest {
  int32 limit = 1;
  int32 offset = 2;
}

message ListUsersResponse {
  repeated UserResponse users = 1;    // repeated = list/array
  int32 total = 2;
}

message DeleteUserRequest {
  string id = 1;
}

message Empty {}

enum Role {
  USER = 0;          // first enum value must be 0 in proto3
  ADMIN = 1;
  MODERATOR = 2;
}
```

> **Ruby Context:** Generate Ruby code from `.proto` files using the `grpc_tools_ruby_protoc` command:
> ```bash
> grpc_tools_ruby_protoc -I ./protos \
>   --ruby_out=./lib \
>   --grpc_out=./lib \
>   ./protos/user_service.proto
> ```
> This generates `user_service_pb.rb` (message classes) and `user_service_services_pb.rb` (service stubs). The generated Ruby classes use the `google-protobuf` gem for serialization.

**Why Protocol Buffers over JSON?**

| Feature | JSON | Protocol Buffers |
|---------|------|-----------------|
| Format | Text (human-readable) | Binary (compact) |
| Size | Larger (field names repeated) | 3-10x smaller (field numbers, no names) |
| Parsing speed | Slower (text parsing) | 5-100x faster (binary parsing) |
| Schema | Optional (JSON Schema) | Required (.proto file) |
| Type safety | Weak (everything is string/number) | Strong (typed fields, enums) |
| Code generation | Manual or third-party | Built-in (generates client/server code) |
| Human readability | Excellent | Poor (binary) |
| Browser support | Native | Requires grpc-web proxy |

**Field Numbers:**
- Each field has a unique number (1, 2, 3...) used in the binary encoding
- Numbers 1-15 use 1 byte to encode (use for frequently used fields)
- Numbers 16-2047 use 2 bytes
- Never reuse or change field numbers (breaks backward compatibility)
- To remove a field, mark it as `reserved` instead of deleting

```protobuf
message User {
  reserved 3, 5;                    // these field numbers can never be reused
  reserved "old_field_name";        // this field name can never be reused
  string id = 1;
  string name = 2;
  // field 3 was removed (reserved)
  string email = 4;
  // field 5 was removed (reserved)
}
```

---

### Communication Patterns

gRPC supports **four communication patterns**, all built on HTTP/2:

**1. Unary RPC (Request-Response):**

The simplest pattern — one request, one response. Similar to a REST API call.

```protobuf
rpc GetUser (GetUserRequest) returns (UserResponse);
```

```
Client ---- Request ----> Server
Client <--- Response ---- Server
```

**Use case:** Simple CRUD operations, lookups, authentication.

> **Ruby Context:** Implementing a unary RPC server and client in Ruby:
> ```ruby
> # Server implementation
> class UserServiceServer < Userservice::UserService::Service
>   def get_user(request, _call)
>     user = UserRepository.find(request.id)
>     Userservice::UserResponse.new(
>       id: user.id,
>       name: user.name,
>       email: user.email,
>       age: user.age
>     )
>   end
> end
>
> # Start the server
> server = GRPC::RpcServer.new
> server.add_http2_port("0.0.0.0:50051", :this_port_is_insecure)
> server.handle(UserServiceServer.new)
> server.run_till_terminated
>
> # Client usage
> stub = Userservice::UserService::Stub.new("localhost:50051", :this_channel_is_insecure)
> response = stub.get_user(Userservice::GetUserRequest.new(id: "123"))
> puts response.name  # => "Alice"
> ```

**2. Server Streaming RPC:**

Client sends one request, server returns a stream of responses.

```protobuf
rpc ListUsers (ListUsersRequest) returns (stream UserResponse);
```

```
Client ---- Request ---------> Server
Client <--- Response 1 ------- Server
Client <--- Response 2 ------- Server
Client <--- Response 3 ------- Server
Client <--- ... --------------- Server
Client <--- End of Stream ---- Server
```

**Use case:** Downloading large datasets, real-time feeds, log streaming, search results.

> **Ruby Context:** Server streaming in Ruby:
> ```ruby
> # Server implementation
> class UserServiceServer < Userservice::UserService::Service
>   def list_users(request, _call)
>     users = UserRepository.all(limit: request.limit, offset: request.offset)
>     # Return an Enumerator that yields responses one at a time
>     Enumerator.new do |yielder|
>       users.each do |user|
>         yielder.yield Userservice::UserResponse.new(
>           id: user.id, name: user.name, email: user.email
>         )
>       end
>     end
>   end
> end
>
> # Client usage
> stub.list_users(Userservice::ListUsersRequest.new(limit: 100)).each do |user|
>   puts user.name
> end
> ```

**3. Client Streaming RPC:**

Client sends a stream of requests, server returns one response after receiving all data.

```protobuf
rpc UploadUsersBatch (stream CreateUserRequest) returns (BatchUploadResponse);
```

```
Client ---- Request 1 ------> Server
Client ---- Request 2 ------> Server
Client ---- Request 3 ------> Server
Client ---- End of Stream --> Server
Client <--- Response -------- Server
```

**Use case:** File upload, batch data ingestion, sensor data collection.

**4. Bidirectional Streaming RPC:**

Both client and server send streams of messages independently. They can read and write in any order.

```protobuf
rpc Chat (stream ChatMessage) returns (stream ChatMessage);
```

```
Client ---- Message 1 ------> Server
Client <--- Message A -------- Server
Client ---- Message 2 ------> Server
Client ---- Message 3 ------> Server
Client <--- Message B -------- Server
Client <--- Message C -------- Server
  ... (both sides send independently) ...
```

**Use case:** Chat applications, real-time collaboration, interactive gaming, live data sync.

> **Ruby Context:** Bidirectional streaming in Ruby:
> ```ruby
> # Server implementation
> class ChatServiceServer < Chatservice::ChatService::Service
>   def chat(requests)
>     # requests is an Enumerable of incoming messages
>     Enumerator.new do |yielder|
>       requests.each do |message|
>         # Process incoming message and yield responses
>         response = process_message(message)
>         yielder.yield response
>       end
>     end
>   end
> end
> ```

---

### gRPC vs REST

| Feature | REST | gRPC |
|---------|------|------|
| **Protocol** | HTTP/1.1 or HTTP/2 | HTTP/2 (always) |
| **Data format** | JSON (text) | Protocol Buffers (binary) |
| **Payload size** | Larger | 3-10x smaller |
| **Serialization speed** | Slower | 5-100x faster |
| **Streaming** | Limited (SSE, WebSocket separate) | Native (4 patterns) |
| **Code generation** | Optional (OpenAPI) | Built-in (protoc compiler) |
| **Browser support** | Native | Requires grpc-web proxy |
| **Human readability** | Easy (JSON, curl) | Hard (binary, needs tools) |
| **Caching** | Easy (HTTP caching) | Hard (binary, HTTP/2) |
| **Load balancing** | Simple (L7 HTTP) | Complex (HTTP/2 multiplexing, L7 gRPC-aware) |
| **Error handling** | HTTP status codes | gRPC status codes (16 codes) |
| **Deadline/Timeout** | Client-side only | Built-in (propagated across services) |
| **Interceptors** | Middleware | Built-in interceptor chain |
| **Contract** | Loose (documentation) | Strict (.proto file) |

**gRPC Status Codes:**

| Code | Name | Description |
|------|------|-------------|
| 0 | OK | Success |
| 1 | CANCELLED | Operation cancelled by caller |
| 2 | UNKNOWN | Unknown error |
| 3 | INVALID_ARGUMENT | Client sent invalid argument |
| 4 | DEADLINE_EXCEEDED | Operation timed out |
| 5 | NOT_FOUND | Resource not found |
| 7 | PERMISSION_DENIED | Caller doesn't have permission |
| 8 | RESOURCE_EXHAUSTED | Rate limit or quota exceeded |
| 12 | UNIMPLEMENTED | Method not implemented |
| 13 | INTERNAL | Internal server error |
| 14 | UNAVAILABLE | Service temporarily unavailable |
| 16 | UNAUTHENTICATED | Missing or invalid authentication |

---

### When to Use gRPC

| Scenario | Why gRPC |
|----------|---------|
| **Microservice-to-microservice** | Low latency, strong contracts, code generation |
| **Low-latency requirements** | Binary serialization, HTTP/2 multiplexing |
| **Polyglot environments** | Code generation for 10+ languages from one .proto file |
| **Streaming data** | Native support for all 4 streaming patterns |
| **Mobile backends** | Smaller payloads save bandwidth |
| **Internal APIs** | Strong typing, backward compatibility, no need for browser support |

| Scenario | Why NOT gRPC |
|----------|-------------|
| **Public-facing APIs** | Browsers can't call gRPC natively; REST is more accessible |
| **Simple CRUD** | REST is simpler and sufficient |
| **Need HTTP caching** | gRPC doesn't work well with CDN/HTTP caches |
| **Debugging** | Binary format is harder to inspect than JSON |
| **Small teams** | Learning curve and tooling overhead may not be worth it |

**Common Architecture Pattern:**
```
External Clients (Browser, Mobile)
        |
    [API Gateway]  ← REST / GraphQL (public-facing)
        |
  [Microservices]  ← gRPC (internal communication)
   /    |    \
  Svc A  Svc B  Svc C  ← gRPC between services
```

> **Ruby Context:** In a Ruby microservices architecture, you might use Rails for the public-facing REST/GraphQL API gateway and gRPC for internal service communication. The **gruf** gem provides a Rails-like framework for building gRPC services in Ruby, with interceptors, error handling, and a familiar structure:
> ```ruby
> # Using gruf for a more Rails-like gRPC experience
> class UserService < Gruf::Controllers::Base
>   bind Userservice::UserService::Service
>
>   def get_user
>     user = User.find(request.message.id)
>     Userservice::UserResponse.new(
>       id: user.id.to_s,
>       name: user.name,
>       email: user.email
>     )
>   rescue ActiveRecord::RecordNotFound
>     fail!(:not_found, :user_not_found, "User not found")
>   end
> end
> ```

---

### gRPC Advanced Features

**Deadline Propagation:**

gRPC has built-in support for **deadlines** (timeouts) that propagate across service calls. If Service A calls Service B with a 5-second deadline, and Service B calls Service C, Service C automatically inherits the remaining deadline.

```
Client sets deadline: 5 seconds

Client → Service A (remaining: 5s)
  Service A does 1s of work
  Service A → Service B (remaining: 4s)  ← deadline propagated automatically
    Service B does 1s of work
    Service B → Service C (remaining: 3s)  ← deadline propagated again
      Service C does 1s of work
      Service C responds (remaining: 2s)
    Service B responds (remaining: 2s)
  Service A responds (remaining: 2s)
Client receives response in ~3s ✅

If Service C takes 4s:
  Service C exceeds deadline → DEADLINE_EXCEEDED error
  Error propagates back through the chain
  Client gets DEADLINE_EXCEEDED after 5s
```

**Why this matters:** In REST, each service sets its own timeout independently. If Service A has a 5s timeout to B, and B has a 5s timeout to C, the total could be 10s — but the client only expected 5s. gRPC's deadline propagation prevents this cascading timeout problem.

> **Ruby Context:** Setting deadlines in Ruby gRPC clients:
> ```ruby
> # Set a 5-second deadline
> stub = Userservice::UserService::Stub.new("localhost:50051", :this_channel_is_insecure)
> deadline = Time.now + 5  # 5 seconds from now
> response = stub.get_user(
>   Userservice::GetUserRequest.new(id: "123"),
>   deadline: deadline
> )
> rescue GRPC::DeadlineExceeded => e
>   puts "Request timed out: #{e.message}"
> ```

**Interceptors (Middleware):**

gRPC interceptors are like HTTP middleware — they run before and after each RPC call. They're used for logging, authentication, metrics, tracing, and error handling.

```
Client Request
    ↓
[Client Interceptor 1: Add auth token]
    ↓
[Client Interceptor 2: Log request]
    ↓
  ──── Network ────
    ↓
[Server Interceptor 1: Validate auth token]
    ↓
[Server Interceptor 2: Log request + start timer]
    ↓
[Server Interceptor 3: Rate limiting]
    ↓
[Actual RPC Handler]
    ↓
[Server Interceptor 2: Log response + stop timer]
    ↓
  ──── Network ────
    ↓
[Client Interceptor 2: Log response]
    ↓
Client receives response
```

**Types of interceptors:**
- **Unary Interceptor:** For request-response RPCs
- **Stream Interceptor:** For streaming RPCs
- Both client-side and server-side interceptors are supported

> **Ruby Context:** gRPC interceptors in Ruby:
> ```ruby
> # Server interceptor for logging
> class LoggingInterceptor < GRPC::ServerInterceptor
>   def request_response(request:, call:, method:, &block)
>     start_time = Time.now
>     Rails.logger.info("gRPC request: #{method}")
>     response = yield
>     elapsed = Time.now - start_time
>     Rails.logger.info("gRPC response: #{method} (#{elapsed.round(3)}s)")
>     response
>   end
> end
>
> # Auth interceptor
> class AuthInterceptor < GRPC::ServerInterceptor
>   def request_response(request:, call:, method:, &block)
>     token = call.metadata["authorization"]
>     raise GRPC::BadStatus.new(GRPC::Core::StatusCodes::UNAUTHENTICATED, "Missing token") unless token
>     yield
>   end
> end
>
> # Register interceptors
> server = GRPC::RpcServer.new(interceptors: [LoggingInterceptor.new, AuthInterceptor.new])
> ```

**gRPC Load Balancing Challenges:**

HTTP/2 multiplexing creates a unique load balancing challenge. A single HTTP/2 connection carries multiple streams (RPCs). If a Layer 4 load balancer routes based on connections, all RPCs from one client go to the same server — defeating the purpose of load balancing.

```
Problem with L4 Load Balancing:
  Client → L4 LB → Server A (single HTTP/2 connection)
  All 1000 RPCs go to Server A
  Server B and C are idle

Solutions:
  1. L7 Load Balancing (gRPC-aware): Route individual RPCs, not connections
     - Envoy, Nginx (with gRPC module), Linkerd
  
  2. Client-Side Load Balancing: Client maintains connections to all servers
     - gRPC has built-in support (pick_first, round_robin)
     - Requires service discovery (DNS, Consul, etcd)
  
  3. Lookaside Load Balancing: Separate load balancer service advises the client
     - Client asks LB service "which server should I use?"
     - LB service returns a server based on load/health
```

**Protobuf Schema Evolution (Backward Compatibility):**

Protocol Buffers are designed for safe schema evolution:

| Change | Safe? | Details |
|--------|-------|---------|
| Add a new field | ✅ Yes | Old clients ignore unknown fields; new clients use default values for missing fields |
| Remove a field | ✅ Yes | Mark as `reserved` to prevent reuse; old clients ignore the missing field |
| Rename a field | ✅ Yes | Wire format uses field numbers, not names |
| Change field number | ❌ No | Breaks all existing clients |
| Change field type | ❌ No | Binary encoding is type-dependent |
| Change `repeated` to scalar | ❌ No | Wire format is different |

---


## 16.4 API Gateway

> An **API Gateway** is a server that acts as the single entry point for all client requests to a backend system. It sits between external clients and internal microservices, handling cross-cutting concerns like routing, authentication, rate limiting, and request transformation. In a microservices architecture, the API Gateway is a critical infrastructure component.

---

### Core Responsibilities

```
                    +------------------+
  Mobile App  ---->|                  |----> User Service
  Web App    ---->|   API Gateway    |----> Order Service
  3rd Party  ---->|                  |----> Payment Service
                    +------------------+     Product Service
                     |  Routing        |     Notification Service
                     |  Auth           |
                     |  Rate Limiting  |
                     |  Transformation |
                     |  Logging        |
                     |  Caching        |
                     +------------------+
```

**Routing:**
- Routes incoming requests to the appropriate backend service based on URL path, headers, or method
- `/api/users/*` → User Service
- `/api/orders/*` → Order Service
- `/api/payments/*` → Payment Service

**Rate Limiting:**
- Enforces request limits per user, API key, or IP
- Protects backend services from being overwhelmed
- Returns `429 Too Many Requests` when limits are exceeded
- Can implement different tiers (free: 100 req/min, paid: 10,000 req/min)

**Authentication & Authorization:**
- Validates authentication tokens (JWT, API keys, OAuth tokens) at the gateway
- Rejects unauthenticated requests before they reach backend services
- Can attach user identity to the request for downstream services
- Centralizes auth logic — individual services don't need to implement it

```
Without API Gateway:
  Client → User Service (validates JWT)
  Client → Order Service (validates JWT)
  Client → Payment Service (validates JWT)
  (Each service duplicates auth logic)

With API Gateway:
  Client → API Gateway (validates JWT once) → User Service (trusts gateway)
                                             → Order Service (trusts gateway)
                                             → Payment Service (trusts gateway)
```

> **Ruby Context:** In the Ruby ecosystem, you can build a lightweight API gateway using **Rails** with **rack-proxy** for request forwarding, or use dedicated gateway solutions. For authentication at the gateway level, **doorkeeper** (OAuth2) and **warden** are common choices. Many Ruby shops use Kong or Nginx as the gateway in front of their Rails services.

---

### Request/Response Transformation

The API Gateway can modify requests and responses as they pass through:

**Request Transformation:**
- Add headers (e.g., `X-Request-ID` for tracing, `X-User-ID` from JWT)
- Convert between protocols (REST → gRPC, HTTP → WebSocket)
- Validate and sanitize input
- Transform request body format

**Response Transformation:**
- Remove internal headers before sending to client
- Convert response formats (XML → JSON, protobuf → JSON)
- Add CORS headers
- Aggregate responses from multiple services

**Protocol Translation:**
```
External (REST/JSON)          Internal (gRPC/protobuf)
  Client → API Gateway  →  Microservices
           (translates)
```

This allows internal services to use gRPC for performance while exposing a REST API to external clients.

> **Ruby Context:** Rails middleware is well-suited for request/response transformation. The **rack-cors** gem handles CORS headers, and custom Rack middleware can add tracing headers:
> ```ruby
> # Custom middleware for request transformation
> class RequestEnricher
>   def initialize(app)
>     @app = app
>   end
>
>   def call(env)
>     # Add tracing header
>     env["HTTP_X_REQUEST_ID"] ||= SecureRandom.uuid
>
>     # Extract user ID from JWT and add as header
>     if (token = env["HTTP_AUTHORIZATION"]&.split(" ")&.last)
>       payload = JWT.decode(token, secret_key).first
>       env["HTTP_X_USER_ID"] = payload["sub"]
>     end
>
>     @app.call(env)
>   end
> end
> ```

---

### API Composition (Backend for Frontend — BFF)

The API Gateway can **aggregate** data from multiple backend services into a single response, reducing the number of round trips for the client.

**Without API Composition:**
```
Mobile App:
  GET /users/123          → User Service     → { name, email }
  GET /users/123/orders   → Order Service    → [ orders ]
  GET /users/123/profile  → Profile Service  → { bio, avatar }
  (3 separate API calls from the client)
```

**With API Composition:**
```
Mobile App:
  GET /api/user-dashboard/123  → API Gateway
    → User Service     → { name, email }
    → Order Service    → [ orders ]
    → Profile Service  → { bio, avatar }
  ← Combined response: { user, orders, profile }
  (1 API call from the client)
```

**Backend for Frontend (BFF) Pattern:**
Different clients (web, mobile, TV) often need different data shapes. The BFF pattern creates a dedicated API gateway (or gateway layer) for each client type:

```
Web App    → Web BFF Gateway    → Microservices
Mobile App → Mobile BFF Gateway → Microservices
TV App     → TV BFF Gateway     → Microservices
```

Each BFF is optimized for its client's needs — the mobile BFF returns smaller payloads, the web BFF returns richer data.

> **Ruby Context:** The BFF pattern is commonly implemented in Ruby using separate Rails API apps or namespaced controllers per client type. Alternatively, GraphQL naturally serves as a BFF since each client can request exactly the data it needs:
> ```ruby
> # BFF-style controller that aggregates from multiple services
> class Api::Mobile::DashboardController < ApplicationController
>   def show
>     user = UserService.find(params[:id])
>     orders = OrderService.recent_for(params[:id], limit: 5)  # Mobile gets fewer
>     profile = ProfileService.find_by_user(params[:id])
>
>     render json: {
>       user: MobileUserSerializer.new(user),       # Compact serializer
>       recent_orders: orders.map { |o| MobileOrderSerializer.new(o) },
>       profile: { avatar_url: profile.avatar_url }  # Minimal profile data
>     }
>   end
> end
> ```

---

### Additional Gateway Features

**Load Balancing:**
- Distributes requests across multiple instances of a service
- Health checks to remove unhealthy instances
- Supports round-robin, least connections, weighted routing

**Circuit Breaking:**
- If a backend service is failing, the gateway can "open the circuit" and return a fallback response
- Prevents cascading failures across the system
- Automatically retries when the service recovers

**Caching:**
- Cache responses for frequently requested, rarely changing data
- Reduces load on backend services
- Configurable per-endpoint TTL

**Logging & Monitoring:**
- Centralized request/response logging
- Metrics collection (latency, error rates, throughput)
- Distributed tracing (inject trace IDs)

**SSL/TLS Termination:**
- Handle HTTPS at the gateway level
- Internal communication can use plain HTTP (within a trusted network)
- Reduces TLS overhead on backend services

**Request Validation:**
- Validate request schema before forwarding to backend
- Reject malformed requests early
- Reduces load on backend services

> **Ruby Context:** For circuit breaking in Ruby, the **circuitbox** or **stoplight** gems provide circuit breaker patterns. For distributed tracing, **opentelemetry-ruby** integrates with Rails and gRPC services:
> ```ruby
> # Circuit breaker with Circuitbox
> Circuitbox.circuit(:user_service, exceptions: [Timeout::Error, Faraday::Error]) do
>   UserServiceClient.get_user(user_id)
> end
> ```

---

### Popular API Gateway Solutions

| Gateway | Type | Key Features | Best For |
|---------|------|-------------|----------|
| **Kong** | Open-source / Enterprise | Plugin ecosystem, Lua-based, DB-backed | General purpose, extensible |
| **AWS API Gateway** | Managed (cloud) | Lambda integration, WebSocket, REST & HTTP APIs | AWS-native architectures |
| **Nginx** | Open-source | High performance, reverse proxy, load balancing | High-throughput, simple routing |
| **Envoy** | Open-source | Service mesh sidecar, gRPC-native, observability | Kubernetes, service mesh (Istio) |
| **Traefik** | Open-source | Auto-discovery, Docker/K8s native, Let's Encrypt | Container-based deployments |
| **Apigee** | Managed (Google Cloud) | Analytics, monetization, developer portal | Enterprise API management |
| **Azure API Management** | Managed (Azure) | Policy engine, developer portal | Azure-native architectures |
| **KrakenD** | Open-source | Stateless, high performance, API aggregation | High-performance aggregation |

**Choosing an API Gateway:**

| Consideration | Recommendation |
|--------------|---------------|
| AWS-native, serverless | AWS API Gateway |
| Kubernetes / service mesh | Envoy + Istio |
| High customization, plugins | Kong |
| Simple reverse proxy + routing | Nginx |
| Container auto-discovery | Traefik |
| Enterprise API management | Apigee or Azure APIM |

---

### API Gateway Anti-Patterns

| Anti-Pattern | Problem | Solution |
|-------------|---------|---------|
| **God Gateway** | Too much business logic in the gateway | Keep gateway thin — routing, auth, rate limiting only |
| **Single point of failure** | Gateway goes down, everything goes down | Deploy multiple gateway instances behind a load balancer |
| **Tight coupling** | Gateway knows too much about service internals | Use service discovery; gateway routes by path, not by service details |
| **No circuit breaking** | One slow service blocks all requests | Implement circuit breakers and timeouts |
| **Over-aggregation** | Gateway does too much data composition | Use BFF pattern or let clients make parallel calls |

---


## 16.5 API Design Best Practices

> Good API design is the difference between an API that developers love and one they dread. A well-designed API is consistent, predictable, well-documented, and resilient to change. These best practices apply regardless of whether you're building REST, GraphQL, or gRPC APIs.

---

### Consistent Naming

Consistency is the most important quality of a good API. Once you establish a convention, follow it everywhere.

**Naming Conventions:**

| Element | Convention | Good ✅ | Bad ❌ |
|---------|-----------|---------|--------|
| URL paths | lowercase, hyphens | `/user-profiles` | `/UserProfiles`, `/user_profiles` |
| Query params | snake_case or camelCase (pick one) | `?sort_by=name` | Mixed: `?sortBy=name&page_size=10` |
| JSON fields | camelCase (JavaScript convention) | `"firstName"` | `"first_name"` (unless your ecosystem prefers it) |
| JSON fields | snake_case (Ruby convention) | `"first_name"` | Mixed conventions in same API |
| Headers | Kebab-Case | `X-Request-Id` | `x_request_id` |
| Protobuf fields | snake_case | `user_name` | `userName` |

> **Ruby Context:** The Ruby/Rails ecosystem strongly favors **snake_case** for JSON fields. Rails serializers output snake_case by default. If your API consumers are primarily JavaScript clients, consider using a serializer that converts to camelCase (the **oj** gem with `Oj.mimic_JSON` or **olive_branch** middleware for automatic case conversion):
> ```ruby
> # config/initializers/olive_branch.rb
> # Automatically converts between camelCase (client) and snake_case (server)
> Rails.application.config.middleware.use OliveBranch::Middleware,
>   inflection: "camel"
> ```

**Key Rule:** Pick one convention and use it consistently across your entire API. Inconsistency is worse than any particular convention.

**Naming Tips:**
- Use **domain language** — name things what your users call them
- Be **specific** — `created_at` is better than `date`; `total_amount` is better than `amount`
- Avoid **abbreviations** — `configuration` not `config`, `application` not `app` (unless universally understood)
- Use **boolean prefixes** — `is_active`, `has_permission`, `can_edit`
- Avoid **redundancy** — in a `User` object, use `name` not `user_name`

> **Ruby Context:** Ruby convention uses `?` suffix for boolean methods (`active?`, `admin?`), but in JSON API responses, use `is_active` or `active` as field names since `?` is not valid in JSON keys.

---

### Error Response Format

A consistent, informative error response format helps clients handle errors gracefully.

**Standard Error Response Structure:**

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request body contains invalid fields.",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address.",
        "value": "not-an-email"
      },
      {
        "field": "age",
        "message": "Must be a positive integer.",
        "value": -5
      }
    ],
    "request_id": "req_abc123def456",
    "documentation_url": "https://api.example.com/docs/errors#VALIDATION_ERROR"
  }
}
```

**Error Response Best Practices:**

| Practice | Why |
|----------|-----|
| Use a consistent structure for ALL errors | Clients can write one error handler |
| Include a machine-readable error code | `"VALIDATION_ERROR"` is parseable; `"The email field is invalid"` is not |
| Include a human-readable message | For developers debugging issues |
| Include field-level details for validation errors | Clients can highlight specific form fields |
| Include a request ID | For correlating with server logs |
| Include a documentation URL | Helps developers self-serve |
| Never expose internal details | No stack traces, SQL queries, or internal paths in production |

> **Ruby Context:** Rails provides built-in validation error formatting. A common pattern is a base error serializer:
> ```ruby
> # app/serializers/error_serializer.rb
> class ErrorSerializer
>   def initialize(resource)
>     @resource = resource
>   end
>
>   def serialize
>     {
>       error: {
>         code: error_code,
>         message: error_message,
>         details: field_errors,
>         request_id: Current.request_id
>       }
>     }
>   end
>
>   private
>
>   def field_errors
>     return [] unless @resource.respond_to?(:errors)
>
>     @resource.errors.map do |error|
>       {
>         field: error.attribute.to_s,
>         message: error.full_message,
>         value: @resource.send(error.attribute)
>       }
>     end
>   end
> end
> ```

**HTTP Status Code + Error Code Mapping:**

```
400 Bad Request        → VALIDATION_ERROR, INVALID_PARAMETER, MISSING_FIELD
401 Unauthorized       → AUTHENTICATION_REQUIRED, TOKEN_EXPIRED, INVALID_TOKEN
403 Forbidden          → PERMISSION_DENIED, INSUFFICIENT_SCOPE
404 Not Found          → RESOURCE_NOT_FOUND, ENDPOINT_NOT_FOUND
409 Conflict           → DUPLICATE_RESOURCE, VERSION_CONFLICT
422 Unprocessable      → BUSINESS_RULE_VIOLATION
429 Too Many Requests  → RATE_LIMIT_EXCEEDED
500 Internal Error     → INTERNAL_ERROR (never expose details)
503 Service Unavailable → SERVICE_UNAVAILABLE, MAINTENANCE_MODE
```

---

### Backward Compatibility

APIs are contracts. Breaking changes force all clients to update simultaneously, which is often impossible (especially for mobile apps and third-party integrations).

**What is a Breaking Change?**

| Change Type | Breaking? | Example |
|-------------|-----------|---------|
| Adding a new optional field to response | No ✅ | Adding `"avatar_url"` to User response |
| Adding a new optional query parameter | No ✅ | Adding `?include_deleted=true` |
| Adding a new endpoint | No ✅ | Adding `GET /users/123/preferences` |
| Removing a field from response | **Yes** ❌ | Removing `"email"` from User response |
| Renaming a field | **Yes** ❌ | Changing `"name"` to `"full_name"` |
| Changing a field's type | **Yes** ❌ | Changing `"age"` from integer to string |
| Changing URL structure | **Yes** ❌ | Changing `/users/123` to `/user/123` |
| Making an optional field required | **Yes** ❌ | Requiring `"phone"` in CreateUser |
| Changing error response format | **Yes** ❌ | Changing error structure |
| Changing authentication mechanism | **Yes** ❌ | Switching from API key to OAuth |

**Strategies for Backward Compatibility:**

**1. Additive Changes Only (Preferred):**
```json
// Version 1 response
{ "id": 1, "name": "Alice" }

// Version 1.1 response (backward compatible — new field added)
{ "id": 1, "name": "Alice", "avatar_url": "https://..." }

// Old clients ignore "avatar_url" — nothing breaks
```

**2. Deprecation Before Removal:**
```json
// Step 1: Add new field, mark old field as deprecated
{
  "name": "Alice",           // deprecated — use full_name instead
  "full_name": "Alice Smith" // new field
}

// Step 2: After migration period, remove old field in next major version
{
  "full_name": "Alice Smith"
}
```

**3. Sunset Header (RFC 8594):**
```
HTTP/1.1 200 OK
Sunset: Sat, 01 Jan 2026 00:00:00 GMT
Deprecation: true
Link: <https://api.example.com/v2/users>; rel="successor-version"
```

> **Ruby Context:** Adding sunset headers in Rails:
> ```ruby
> class V1::UsersController < ApplicationController
>   before_action :add_deprecation_headers
>
>   private
>
>   def add_deprecation_headers
>     response.headers["Sunset"] = "Sat, 01 Jan 2026 00:00:00 GMT"
>     response.headers["Deprecation"] = "true"
>     response.headers["Link"] = '<https://api.example.com/v2/users>; rel="successor-version"'
>   end
> end
> ```

**4. Versioning for Breaking Changes:**
When you must make breaking changes, release a new API version and maintain the old one during a migration period.

---

### API Documentation (OpenAPI / Swagger)

Good documentation is essential for API adoption. **OpenAPI Specification (formerly Swagger)** is the industry standard for describing REST APIs.

**OpenAPI Specification Example:**

```yaml
openapi: 3.0.3
info:
  title: User Service API
  description: API for managing users
  version: 1.0.0
  contact:
    email: api-support@example.com

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging

paths:
  /users:
    get:
      summary: List all users
      operationId: listUsers
      tags:
        - Users
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
        - name: status
          in: query
          schema:
            type: string
            enum: [active, inactive, suspended]
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '429':
          $ref: '#/components/responses/RateLimited'

    post:
      summary: Create a new user
      operationId: createUser
      tags:
        - Users
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/ValidationError'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          example: "user_123"
        name:
          type: string
          example: "Alice Smith"
        email:
          type: string
          format: email
          example: "alice@example.com"
        status:
          type: string
          enum: [active, inactive, suspended]
        created_at:
          type: string
          format: date-time

    CreateUserRequest:
      type: object
      required:
        - name
        - email
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
        email:
          type: string
          format: email
        age:
          type: integer
          minimum: 0
          maximum: 150

    Pagination:
      type: object
      properties:
        offset:
          type: integer
        limit:
          type: integer
        total:
          type: integer

  responses:
    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    ValidationError:
      description: Invalid request body
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    RateLimited:
      description: Rate limit exceeded
      headers:
        Retry-After:
          schema:
            type: integer
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    Error:
      type: object
      properties:
        code:
          type: string
        message:
          type: string
        details:
          type: array
          items:
            type: object

  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - BearerAuth: []
```

> **Ruby Context:** Several gems generate OpenAPI specs from Rails code (code-first approach):
> - **rswag** — integrates with RSpec to generate OpenAPI specs from tests
> - **rspec-openapi** — auto-generates OpenAPI docs from request specs
> - **grape-swagger** — generates Swagger docs for Grape APIs
>
> For contract-first, use **openapi_first** gem to validate requests/responses against an OpenAPI spec:
> ```ruby
> # Using rswag for test-driven API documentation
> RSpec.describe "Users API", type: :request do
>   path "/v1/users" do
>     get "List all users" do
>       tags "Users"
>       produces "application/json"
>       parameter name: :limit, in: :query, type: :integer, required: false
>       parameter name: :status, in: :query, type: :string, required: false
>
>       response "200", "successful" do
>         schema type: :object, properties: {
>           data: { type: :array, items: { "$ref" => "#/components/schemas/User" } }
>         }
>         run_test!
>       end
>     end
>   end
> end
> ```

**Documentation Tools:**

| Tool | Purpose |
|------|---------|
| **Swagger UI** | Interactive API documentation from OpenAPI spec |
| **Redoc** | Beautiful, responsive API documentation |
| **Postman** | API testing + auto-generated documentation |
| **Stoplight** | Visual API design + documentation platform |
| **GraphiQL** | Interactive GraphQL IDE and documentation |
| **Apollo Studio** | GraphQL schema registry and documentation |
| **gRPC reflection** | Runtime service discovery for gRPC |
| **Buf** | Protobuf linting, breaking change detection, documentation |

---

### Contract-First vs Code-First

There are two approaches to API development:

**Contract-First (Design-First):**
1. Design the API specification first (OpenAPI YAML, .proto file, GraphQL schema)
2. Review and agree on the contract with stakeholders
3. Generate server stubs and client SDKs from the spec
4. Implement the server logic

```
Design API Spec → Review → Generate Code → Implement Logic
```

**Code-First (Implementation-First):**
1. Write the server code first
2. Generate the API specification from the code (annotations, decorators)
3. Documentation is derived from the implementation

```
Write Code → Generate Spec → Documentation
```

**Comparison:**

| Aspect | Contract-First | Code-First |
|--------|---------------|-----------|
| API consistency | High (spec is the source of truth) | Medium (depends on developer discipline) |
| Parallel development | Yes (frontend and backend work from spec) | No (frontend waits for backend) |
| Breaking change detection | Easy (diff the spec) | Harder (must compare generated specs) |
| Initial effort | Higher (design upfront) | Lower (just start coding) |
| Documentation quality | High (spec IS the documentation) | Variable (depends on annotations) |
| Flexibility | Less (must update spec first) | More (iterate quickly) |
| Best for | Public APIs, large teams, microservices | Internal APIs, small teams, prototyping |

**Recommendation:**
- **Public APIs and microservices:** Contract-first — the spec is the contract between teams
- **Internal tools and prototypes:** Code-first — faster iteration
- **gRPC:** Always contract-first (the .proto file IS the contract)
- **GraphQL:** Always contract-first (the schema IS the contract)

> **Ruby Context:** The Ruby ecosystem leans toward code-first due to Rails' convention-over-configuration philosophy. However, for public APIs and microservices, contract-first is gaining traction. The **openapi_first** gem validates incoming requests against an OpenAPI spec, and **committee** gem provides middleware for request/response validation against OpenAPI or JSON Schema specs.

---

### Additional Best Practices

**1. Use Proper HTTP Status Codes:**
Don't return `200 OK` for everything. Use the right status code for the right situation (see Module 15, Section 15.4).

**2. Support Content Negotiation:**
```
Request:  Accept: application/json
Response: Content-Type: application/json

Request:  Accept: application/xml
Response: Content-Type: application/xml
```

> **Ruby Context:** Rails supports content negotiation with `respond_to`:
> ```ruby
> def show
>   @user = User.find(params[:id])
>   respond_to do |format|
>     format.json { render json: @user }
>     format.xml  { render xml: @user }
>   end
> end
> ```

**3. Use ISO 8601 for Dates:**
```json
{
  "created_at": "2025-03-15T14:30:00Z",
  "updated_at": "2025-03-15T14:30:00+05:30"
}
```
Always use UTC or include timezone offset. Never use ambiguous formats like `03/15/2025`.

> **Ruby Context:** Rails uses ISO 8601 by default when serializing `DateTime` and `Time` objects to JSON. Ensure `config.time_zone` is set appropriately and use `Time.current` instead of `Time.now` for timezone-aware timestamps.

**4. Use UUIDs for Public IDs:**
```json
{ "id": "550e8400-e29b-41d4-a716-446655440000" }
```
Don't expose auto-increment database IDs — they leak information (total count, creation order) and are guessable.

> **Ruby Context:** Rails supports UUID primary keys natively with PostgreSQL:
> ```ruby
> # In migration
> class CreateUsers < ActiveRecord::Migration[7.1]
>   def change
>     create_table :users, id: :uuid do |t|
>       t.string :name
>       t.string :email
>       t.timestamps
>     end
>   end
> end
>
> # Or enable globally in config/initializers/generators.rb
> Rails.application.config.generators do |g|
>   g.orm :active_record, primary_key_type: :uuid
> end
> ```

**5. Implement Health Check Endpoints:**
```
GET /health          → 200 OK { "status": "healthy" }
GET /health/ready    → 200 OK (ready to serve traffic)
GET /health/live     → 200 OK (process is alive)
```

> **Ruby Context:** The **rails-health-check** gem or a simple custom route:
> ```ruby
> # config/routes.rb
> get "/health", to: proc { [200, {}, ['{"status":"healthy"}']] }
>
> # Or with Rails 7.1+ built-in health check
> Rails.application.routes.draw do
>   get "up" => "rails/health#show", as: :rails_health_check
> end
> ```

**6. Support Bulk Operations:**
```
POST /users/bulk
{
  "users": [
    { "name": "Alice", "email": "alice@example.com" },
    { "name": "Bob", "email": "bob@example.com" }
  ]
}
```
Reduces round trips for batch operations.

> **Ruby Context:** Rails supports bulk inserts with `insert_all`:
> ```ruby
> def bulk_create
>   users_data = params[:users].map { |u| u.permit(:name, :email).to_h }
>   result = User.insert_all(users_data, returning: %w[id name email])
>   render json: { created: result.rows.length }, status: :created
> end
> ```

**7. Use Correlation IDs for Tracing:**
```
Request:  X-Request-Id: req_abc123
Response: X-Request-Id: req_abc123
```
Include in all logs for distributed tracing across services.

> **Ruby Context:** Rails automatically generates and propagates `X-Request-Id` headers via the `ActionDispatch::RequestId` middleware. Access it with `request.request_id` in controllers or `Current.request_id` with `CurrentAttributes`.

**8. Implement Graceful Degradation:**
- Return partial results when some backend services are down
- Use fallback/default values when possible
- Communicate degraded state to clients

**9. Design for Idempotency:**
- All GET, PUT, DELETE should be idempotent
- Use idempotency keys for POST operations (see Section 16.1)

**10. Security Checklist:**
- Always use HTTPS
- Validate and sanitize all input
- Use parameterized queries (prevent SQL injection)
- Implement rate limiting
- Use short-lived tokens with refresh mechanism
- Don't expose sensitive data in URLs (use headers or body)
- Implement CORS properly
- Log all authentication failures

> **Ruby Context:** Rails provides many security features out of the box: parameterized queries via ActiveRecord, CSRF protection, strong parameters for input validation, and `force_ssl` for HTTPS. The **brakeman** gem performs static analysis for security vulnerabilities in Rails apps. The **bundler-audit** gem checks for known vulnerabilities in dependencies.

---


## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| REST | Architectural style with 6 constraints — stateless, cacheable, uniform interface |
| REST Resources | Use nouns (not verbs), plural, lowercase, hyphens — `/users/123/orders` |
| HATEOAS | Responses include links to related actions — most APIs skip this in practice |
| API Versioning | URL path versioning (`/v1/`) is most common; only version for breaking changes |
| Pagination | Offset-based for simple UIs; cursor-based for feeds, large datasets, real-time data |
| Rate Limiting | Use `429` status code with `Retry-After` header; implement per-user/per-key limits |
| Idempotency | GET, PUT, DELETE are idempotent; use idempotency keys for POST |
| GraphQL | Client defines response shape — eliminates over/under-fetching |
| GraphQL N+1 | Use DataLoader to batch database queries — essential for production |
| GraphQL vs REST | GraphQL for complex, nested data; REST for simple CRUD and caching |
| gRPC | Binary (protobuf) over HTTP/2 — 3-10x smaller, 5-100x faster than REST/JSON |
| gRPC Streaming | 4 patterns: Unary, Server Streaming, Client Streaming, Bidirectional |
| gRPC Use Case | Internal microservice communication; REST/GraphQL for external clients |
| API Gateway | Single entry point — routing, auth, rate limiting, transformation |
| BFF Pattern | Dedicated gateway per client type (web, mobile, TV) |
| API Documentation | OpenAPI/Swagger for REST; GraphQL schema is self-documenting; .proto for gRPC |
| Contract-First | Design the spec first, then implement — best for public APIs and large teams |
| Backward Compatibility | Additive changes only; deprecate before removing; version for breaking changes |
| Error Responses | Consistent structure with machine-readable code, human message, and details |
| Naming | Pick one convention and use it everywhere — consistency beats any particular style |

> **Ruby Ecosystem Summary:** Rails (`rails new --api`) for REST APIs, **graphql-ruby** for GraphQL, **grpc** gem for gRPC, **rack-attack** for rate limiting, **Kaminari/Pagy** for pagination, **rswag** for OpenAPI documentation, **circuitbox** for circuit breaking, and **doorkeeper** for OAuth2. Ruby's convention-over-configuration philosophy and rich gem ecosystem make it one of the most productive languages for API development.

---

## Interview Tips for Module 16

1. **REST principles** — know all 6 constraints, especially statelessness and uniform interface
2. **Design a REST API** — be ready to design resource URLs, methods, status codes, and pagination for any system
3. **Pagination** — know offset vs cursor trade-offs; cursor-based is preferred for large-scale systems
4. **Idempotency** — explain why it matters and how to implement it for POST with idempotency keys
5. **GraphQL vs REST** — articulate clear trade-offs; don't just say "GraphQL is better"
6. **N+1 problem** — explain the problem and how DataLoader solves it
7. **gRPC** — know when to use it (internal services) vs REST (external APIs); explain protobuf advantages
8. **API Gateway** — explain its role in microservices; know the difference between gateway and service mesh
9. **Versioning** — know the strategies and when to version (breaking changes only)
10. **Error handling** — design a consistent error response format with codes, messages, and details
11. **Rate limiting** — know the algorithms (Token Bucket, Sliding Window) and how to communicate limits via headers
12. **Backward compatibility** — explain what constitutes a breaking change and how to avoid them
