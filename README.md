# System Design: Zero to Hero Syllabus (C++)

> A comprehensive syllabus covering every topic in Low-Level Design (LLD) and High-Level Design (HLD) from absolute beginner to interview-ready.

---

## Table of Contents

- [PART A: LOW-LEVEL DESIGN (LLD)](#part-a-low-level-design-lld)
  - [Module 1: C++ Fundamentals (Prerequisites)](#module-1-c-fundamentals-prerequisites)
  - [Module 2: Object-Oriented Programming (OOP) in C++](#module-2-object-oriented-programming-oop-in-c)
  - [Module 3: SOLID Principles](#module-3-solid-principles)
  - [Module 4: Design Patterns — Creational](#module-4-design-patterns--creational)
  - [Module 5: Design Patterns — Structural](#module-5-design-patterns--structural)
  - [Module 6: Design Patterns — Behavioral](#module-6-design-patterns--behavioral)
  - [Module 7: UML Diagrams & Modeling](#module-7-uml-diagrams--modeling)
  - [Module 8: Advanced OOP & C++ Concepts](#module-8-advanced-oop--c-concepts)
  - [Module 9: Concurrency & Multithreading in C++](#module-9-concurrency--multithreading-in-c)
  - [Module 10: Data Structures & Algorithms for LLD](#module-10-data-structures--algorithms-for-lld)
  - [Module 11: Error Handling, Logging & Testing](#module-11-error-handling-logging--testing)
  - [Module 12: LLD Practice Problems (Easy)](#module-12-lld-practice-problems-easy)
  - [Module 13: LLD Practice Problems (Medium)](#module-13-lld-practice-problems-medium)
  - [Module 14: LLD Practice Problems (Hard)](#module-14-lld-practice-problems-hard)
- [PART B: HIGH-LEVEL DESIGN (HLD)](#part-b-high-level-design-hld)
  - [Module 15: Networking Fundamentals](#module-15-networking-fundamentals)
  - [Module 16: Application Layer Protocols & APIs](#module-16-application-layer-protocols--apis)
  - [Module 17: Web Architecture Basics](#module-17-web-architecture-basics)
  - [Module 18: Databases — Relational (SQL)](#module-18-databases--relational-sql)
  - [Module 19: Databases — Non-Relational (NoSQL)](#module-19-databases--non-relational-nosql)
  - [Module 20: Database Deep Dive](#module-20-database-deep-dive)
  - [Module 21: Caching](#module-21-caching)
  - [Module 22: Scalability](#module-22-scalability)
  - [Module 23: Load Balancing](#module-23-load-balancing)
  - [Module 24: Message Queues & Event-Driven Architecture](#module-24-message-queues--event-driven-architecture)
  - [Module 25: Microservices Architecture](#module-25-microservices-architecture)
  - [Module 26: Distributed Systems Fundamentals](#module-26-distributed-systems-fundamentals)
  - [Module 27: Distributed Systems — Advanced](#module-27-distributed-systems--advanced)
  - [Module 28: Storage & File Systems](#module-28-storage--file-systems)
  - [Module 29: Security & Authentication](#module-29-security--authentication)
  - [Module 30: Monitoring, Logging & Observability](#module-30-monitoring-logging--observability)
  - [Module 31: DevOps & Deployment](#module-31-devops--deployment)
  - [Module 32: Back-of-the-Envelope Estimation](#module-32-back-of-the-envelope-estimation)
  - [Module 33: HLD Practice Problems (Easy)](#module-33-hld-practice-problems-easy)
  - [Module 34: HLD Practice Problems (Medium)](#module-34-hld-practice-problems-medium)
  - [Module 35: HLD Practice Problems (Hard)](#module-35-hld-practice-problems-hard)
- [PART C: INTERVIEW PREPARATION](#part-c-interview-preparation)
  - [Module 36: LLD Interview Framework](#module-36-lld-interview-framework)
  - [Module 37: HLD Interview Framework](#module-37-hld-interview-framework)
  - [Module 38: Common Mistakes & Tips](#module-38-common-mistakes--tips)

---


# PART A: LOW-LEVEL DESIGN (LLD)

> LLD focuses on the internal design of individual components — classes, interfaces, design patterns, and how objects interact at the code level.

---

## Module 1: C++ Fundamentals (Prerequisites)

### 1.1 Basics
- Variables, Data Types, Type Casting
- Operators (Arithmetic, Logical, Bitwise, Ternary)
- Control Flow (if/else, switch, loops)
- Functions (Pass by Value, Pass by Reference, Pass by Pointer)
- Default Arguments, Function Overloading
- Inline Functions
- Recursion

### 1.2 Pointers & Memory
- Pointers and References
- Pointer Arithmetic
- Dynamic Memory Allocation (new, delete, new[], delete[])
- Memory Layout (Stack, Heap, Data, Code segments)
- Dangling Pointers, Memory Leaks
- Smart Pointers (unique_ptr, shared_ptr, weak_ptr)
- RAII (Resource Acquisition Is Initialization)

### 1.3 STL (Standard Template Library)
- Containers: vector, list, deque, stack, queue, priority_queue
- Associative Containers: set, multiset, map, multimap
- Unordered Containers: unordered_set, unordered_map
- Iterators (begin, end, rbegin, rend)
- Algorithms (sort, find, binary_search, lower_bound, upper_bound)
- Functors and Lambda Expressions
- std::pair, std::tuple
- std::string and string operations

### 1.4 Templates
- Function Templates
- Class Templates
- Template Specialization (Full & Partial)
- Variadic Templates
- SFINAE (Substitution Failure Is Not An Error) — basics
- Type Traits (std::is_same, std::enable_if)

### 1.5 Namespaces & Preprocessor
- Namespaces (named, unnamed, nested)
- using declarations vs using directives
- Preprocessor Directives (#define, #include, #ifdef, #pragma once)
- Header Guards

---

## Module 2: Object-Oriented Programming (OOP) in C++

### 2.1 Classes & Objects
- Class Definition, Access Specifiers (public, private, protected)
- Constructors (Default, Parameterized, Copy, Move)
- Destructor
- this pointer
- Static Members (variables and functions)
- Const Member Functions
- Mutable keyword
- Friend Functions and Friend Classes

### 2.2 Encapsulation
- Data Hiding
- Getters and Setters
- Invariant Enforcement
- Immutable Objects in C++

### 2.3 Inheritance
- Single Inheritance
- Multiple Inheritance
- Multilevel Inheritance
- Hierarchical Inheritance
- Hybrid Inheritance
- Diamond Problem & Virtual Inheritance
- Constructor/Destructor Call Order in Inheritance
- Access Control in Inheritance (public, protected, private inheritance)
- Upcasting and Downcasting

### 2.4 Polymorphism
- Compile-Time Polymorphism
  - Function Overloading
  - Operator Overloading (unary, binary, stream operators)
  - Templates (parametric polymorphism)
- Runtime Polymorphism
  - Virtual Functions
  - Pure Virtual Functions
  - Abstract Classes
  - VTable and VPtr (how virtual dispatch works internally)
  - override and final keywords
  - Covariant Return Types
- RTTI (Runtime Type Information)
  - dynamic_cast
  - typeid operator

### 2.5 Abstraction
- Abstract Classes vs Interfaces (pure virtual classes)
- Designing Interfaces in C++
- Interface Segregation in Practice
- Abstract Factory using Abstract Classes

### 2.6 Composition vs Inheritance
- "Has-A" vs "Is-A" Relationships
- When to Prefer Composition over Inheritance
- Aggregation vs Composition
- Dependency Injection basics

---

## Module 3: SOLID Principles

### 3.1 Single Responsibility Principle (SRP)
- One class, one reason to change
- Identifying responsibilities
- Refactoring God Classes
- C++ examples with before/after

### 3.2 Open/Closed Principle (OCP)
- Open for extension, closed for modification
- Using inheritance and polymorphism for OCP
- Strategy pattern as OCP example
- Template Method as OCP example

### 3.3 Liskov Substitution Principle (LSP)
- Subtypes must be substitutable for base types
- Rectangle-Square problem
- Preconditions and Postconditions
- Covariance and Contravariance
- Behavioral subtyping

### 3.4 Interface Segregation Principle (ISP)
- No client should depend on methods it doesn't use
- Fat interfaces vs thin interfaces
- Splitting interfaces in C++ (multiple inheritance of pure virtual classes)
- Role interfaces

### 3.5 Dependency Inversion Principle (DIP)
- High-level modules should not depend on low-level modules
- Both should depend on abstractions
- Dependency Injection in C++ (Constructor, Setter, Interface injection)
- Inversion of Control (IoC) concept

### 3.6 Other Important Principles
- DRY (Don't Repeat Yourself)
- KISS (Keep It Simple, Stupid)
- YAGNI (You Aren't Gonna Need It)
- Law of Demeter (Principle of Least Knowledge)
- Composition over Inheritance
- Program to an Interface, not an Implementation
- Encapsulate What Varies
- Favor Object Composition over Class Inheritance

---

## Module 4: Design Patterns — Creational

> Creational patterns deal with object creation mechanisms.

### 4.1 Singleton Pattern
- Eager vs Lazy Initialization
- Thread-Safe Singleton (Meyer's Singleton, Double-Checked Locking)
- Singleton with std::call_once
- When to use and when to avoid
- Singleton vs Global Variable
- C++ implementation

### 4.2 Factory Method Pattern
- Define interface for creating objects
- Let subclasses decide which class to instantiate
- Simple Factory vs Factory Method
- Parameterized Factory
- C++ implementation

### 4.3 Abstract Factory Pattern
- Family of related objects
- Factory of Factories
- Cross-platform UI example
- C++ implementation

### 4.4 Builder Pattern
- Step-by-step construction of complex objects
- Director class
- Fluent Builder (method chaining)
- Builder vs Constructor with many parameters
- C++ implementation

### 4.5 Prototype Pattern
- Cloning existing objects
- Deep Copy vs Shallow Copy
- Prototype Registry
- clone() method using virtual functions
- C++ implementation (copy constructors, clone())

### 4.6 Object Pool Pattern
- Reusing expensive objects
- Connection Pool, Thread Pool concepts
- C++ implementation

---

## Module 5: Design Patterns — Structural

> Structural patterns deal with object composition and relationships.

### 5.1 Adapter Pattern
- Convert interface of a class to another interface
- Class Adapter (multiple inheritance) vs Object Adapter (composition)
- Two-Way Adapter
- C++ implementation

### 5.2 Bridge Pattern
- Decouple abstraction from implementation
- Abstraction hierarchy + Implementation hierarchy
- Bridge vs Adapter
- C++ implementation

### 5.3 Composite Pattern
- Tree structures (part-whole hierarchies)
- Leaf vs Composite nodes
- Uniform treatment of individual and composite objects
- File System example
- C++ implementation

### 5.4 Decorator Pattern
- Add responsibilities dynamically
- Decorator vs Inheritance
- Stacking decorators
- Pizza/Coffee topping example
- C++ implementation

### 5.5 Facade Pattern
- Simplified interface to a complex subsystem
- Reducing coupling
- Home Theater / Computer Boot example
- C++ implementation

### 5.6 Flyweight Pattern
- Sharing common state to reduce memory
- Intrinsic vs Extrinsic state
- Flyweight Factory
- Text Editor character rendering example
- C++ implementation

### 5.7 Proxy Pattern
- Placeholder for another object
- Types: Virtual Proxy, Protection Proxy, Remote Proxy, Caching Proxy, Logging Proxy
- Proxy vs Decorator
- Smart Pointer as Proxy
- C++ implementation

---

## Module 6: Design Patterns — Behavioral

> Behavioral patterns deal with communication between objects.

### 6.1 Strategy Pattern
- Define family of algorithms, make them interchangeable
- Eliminate conditional statements
- Sorting strategies example
- Payment method example
- C++ implementation (using polymorphism and std::function)

### 6.2 Observer Pattern
- One-to-many dependency (publish-subscribe)
- Subject and Observer interfaces
- Push vs Pull model
- Event system example
- C++ implementation

### 6.3 Command Pattern
- Encapsulate request as an object
- Undo/Redo functionality
- Macro Commands
- Command Queue
- C++ implementation

### 6.4 State Pattern
- Object behavior changes based on internal state
- State Transition Table
- Vending Machine example
- TCP Connection example
- State vs Strategy
- C++ implementation

### 6.5 Template Method Pattern
- Define skeleton of algorithm, let subclasses fill in steps
- Hook methods
- Hollywood Principle ("Don't call us, we'll call you")
- C++ implementation

### 6.6 Iterator Pattern
- Sequential access to elements without exposing internals
- Internal vs External iterators
- Custom iterator in C++ (begin, end, operator++, operator*)
- Range-based for loop compatibility
- C++ implementation

### 6.7 Mediator Pattern
- Centralized communication between objects
- Reduce many-to-many to one-to-many
- Chat Room example
- Air Traffic Controller example
- C++ implementation

### 6.8 Chain of Responsibility Pattern
- Pass request along a chain of handlers
- Each handler decides to process or pass
- Logging levels example
- Middleware pipeline example
- C++ implementation

### 6.9 Visitor Pattern
- Add new operations without modifying classes
- Double Dispatch
- Element + Visitor interfaces
- AST traversal example
- C++ implementation

### 6.10 Memento Pattern
- Capture and restore object state
- Originator, Memento, Caretaker
- Undo functionality
- C++ implementation

### 6.11 Interpreter Pattern
- Define grammar and interpret sentences
- Abstract Syntax Tree
- Simple expression evaluator
- C++ implementation

### 6.12 Null Object Pattern
- Provide default "do nothing" behavior
- Avoid null checks
- C++ implementation

---

## Module 7: UML Diagrams & Modeling

### 7.1 Class Diagrams
- Classes, Attributes, Methods
- Visibility (+public, -private, #protected)
- Static members (underlined)
- Abstract classes (italics)

### 7.2 Relationships
- Association (uses)
- Aggregation (has-a, weak ownership)
- Composition (has-a, strong ownership, lifecycle dependency)
- Inheritance / Generalization (is-a)
- Realization / Implementation (implements interface)
- Dependency (uses temporarily)
- Multiplicity (1, 0..1, 0..*, 1..*)

### 7.3 Sequence Diagrams
- Lifelines, Activation bars
- Synchronous vs Asynchronous messages
- Return messages
- Loops, Conditions (alt, opt, loop fragments)

### 7.4 Other UML Diagrams (Overview)
- Use Case Diagrams
- Activity Diagrams
- State Machine Diagrams
- Component Diagrams
- Deployment Diagrams

### 7.5 Practical UML
- Drawing class diagrams for design patterns
- Drawing class diagrams for LLD problems
- Tools: draw.io, PlantUML, Lucidchart

---

## Module 8: Advanced OOP & C++ Concepts

### 8.1 Move Semantics & Perfect Forwarding
- Lvalues vs Rvalues
- Move Constructor, Move Assignment Operator
- std::move, std::forward
- Rule of Zero, Rule of Three, Rule of Five

### 8.2 Type Erasure
- std::function
- std::any, std::variant
- Concept of type erasure for polymorphism without inheritance

### 8.3 CRTP (Curiously Recurring Template Pattern)
- Static Polymorphism
- Mixin classes
- Performance benefits over virtual dispatch

### 8.4 Pimpl Idiom (Pointer to Implementation)
- Compilation firewall
- ABI stability
- Reducing header dependencies

### 8.5 Policy-Based Design
- Compile-time strategy selection
- Template policies
- std::allocator as policy example

### 8.6 Enum Classes & Strong Types
- enum class vs plain enum
- Strong typedefs
- Preventing implicit conversions

---

## Module 9: Concurrency & Multithreading in C++

### 9.1 Thread Basics
- std::thread creation and joining
- Detached threads
- Thread arguments (by value, by reference with std::ref)
- Thread ID, Hardware concurrency

### 9.2 Synchronization Primitives
- std::mutex, std::recursive_mutex, std::timed_mutex
- std::lock_guard, std::unique_lock, std::scoped_lock
- Deadlock prevention (lock ordering, std::lock)
- std::condition_variable (wait, notify_one, notify_all)
- Spurious wakeups

### 9.3 Atomic Operations
- std::atomic<T>
- Memory ordering (relaxed, acquire, release, seq_cst)
- Compare-and-swap (CAS)
- Lock-free programming basics

### 9.4 Async Programming
- std::async, std::future, std::promise
- std::packaged_task
- std::shared_future

### 9.5 Classic Concurrency Problems
- Producer-Consumer Problem
- Reader-Writer Problem
- Dining Philosophers Problem
- Sleeping Barber Problem
- Bounded Buffer Problem

### 9.6 Thread Pool
- Custom Thread Pool implementation in C++
- Task Queue with condition variables
- Work stealing (concept)

### 9.7 Concurrent Data Structures
- Thread-safe Queue
- Thread-safe HashMap
- Lock-free Stack (basics)

---

## Module 10: Data Structures & Algorithms for LLD

### 10.1 Essential Data Structures
- Arrays, Vectors, Linked Lists
- Stacks, Queues, Deques
- Hash Maps, Hash Sets
- Trees (Binary Tree, BST, AVL, Red-Black)
- Heaps (Min-Heap, Max-Heap)
- Graphs (Adjacency List, Adjacency Matrix)
- Tries

### 10.2 Algorithms Relevant to LLD
- Sorting (for leaderboards, rankings)
- Searching (for lookup systems)
- Graph Traversal BFS/DFS (for social networks, dependency resolution)
- Shortest Path (for maps, routing)
- Hashing (for caching, deduplication)
- LRU Cache implementation (HashMap + Doubly Linked List)
- LFU Cache implementation
- Consistent Hashing
- Rate Limiting algorithms (Token Bucket, Sliding Window)

---

## Module 11: Error Handling, Logging & Testing

### 11.1 Error Handling in C++
- Exceptions (try, catch, throw)
- Exception Hierarchy (std::exception, std::runtime_error, etc.)
- Custom Exception Classes
- Exception Safety Guarantees (Basic, Strong, No-throw)
- RAII for exception safety
- noexcept specifier
- Error Codes vs Exceptions (when to use which)
- std::optional, std::expected (C++23)

### 11.2 Logging
- Logging Levels (TRACE, DEBUG, INFO, WARN, ERROR, FATAL)
- Logger Design (Singleton Logger)
- Log formatting and rotation
- Structured logging

### 11.3 Unit Testing (C++)
- Google Test (gtest) framework
- Test fixtures
- Assertions (EXPECT_EQ, ASSERT_TRUE, etc.)
- Mocking with Google Mock (gmock)
- Test-Driven Development (TDD) basics
- Writing testable code (dependency injection)

---


## Module 12: LLD Practice Problems (Easy)

### 12.1 Parking Lot System
- Vehicle types (Car, Bike, Truck)
- Parking spots (Small, Medium, Large)
- Entry/Exit gates, Ticketing
- Payment calculation
- Design patterns: Strategy, Factory

### 12.2 Tic-Tac-Toe
- Board representation
- Player turns
- Win condition checking
- Extensible to NxN board
- Design patterns: State

### 12.3 Vending Machine
- Product inventory
- Coin/Note handling
- State management (Idle, HasMoney, Dispensing)
- Design patterns: State, Strategy

### 12.4 Stack Overflow (Simplified)
- Users, Questions, Answers, Comments
- Voting system
- Tags and search
- Design patterns: Observer, Composite

### 12.5 Logger System
- Multiple log levels
- Multiple output targets (Console, File, Network)
- Chain of Responsibility for log filtering
- Singleton Logger
- Design patterns: Singleton, Chain of Responsibility, Observer

### 12.6 ATM Machine
- Card authentication
- Balance inquiry, Withdrawal, Deposit
- Transaction states
- Design patterns: State, Chain of Responsibility

---

## Module 13: LLD Practice Problems (Medium)

### 13.1 Library Management System
- Books, Members, Librarians
- Issue, Return, Reserve books
- Fine calculation
- Search (by title, author, ISBN)
- Design patterns: Observer, Strategy, Factory

### 13.2 Elevator System
- Multiple elevators, multiple floors
- Scheduling algorithms (SCAN, LOOK, FCFS)
- Direction management
- Capacity handling
- Design patterns: Strategy, State, Observer

### 13.3 Snake and Ladder Game
- Board with snakes and ladders
- Dice rolling
- Multiple players
- Win condition
- Design patterns: Strategy, Observer

### 13.4 Splitwise / Expense Sharing
- Users, Groups, Expenses
- Split types (Equal, Exact, Percentage)
- Balance calculation and simplification
- Design patterns: Strategy, Observer

### 13.5 Movie Ticket Booking (BookMyShow)
- Theaters, Screens, Shows, Seats
- Seat selection and locking
- Concurrent booking handling
- Payment integration
- Design patterns: Strategy, Observer, Singleton

### 13.6 Online Shopping (Amazon)
- Products, Cart, Orders
- Inventory management
- Payment processing
- Order tracking
- Discount/Coupon system
- Design patterns: Strategy, Observer, Factory, Decorator

### 13.7 Car Rental System
- Vehicles, Reservations, Customers
- Vehicle types and pricing
- Availability checking
- Insurance and add-ons
- Design patterns: Strategy, Factory, Builder

### 13.8 Hotel Booking System
- Rooms, Reservations, Guests
- Room types and pricing
- Check-in/Check-out
- Amenities
- Design patterns: Strategy, Factory, Observer

---

## Module 14: LLD Practice Problems (Hard)

### 14.1 Chess Game
- Board, Pieces (King, Queen, Rook, Bishop, Knight, Pawn)
- Move validation per piece type
- Check, Checkmate, Stalemate detection
- Castling, En Passant, Pawn Promotion
- Design patterns: Strategy, Command, Observer

### 14.2 Design a File System (In-Memory)
- Files and Directories (Composite pattern)
- CRUD operations
- Path resolution
- Permissions
- Design patterns: Composite, Iterator, Visitor

### 14.3 Design a Spreadsheet (Excel)
- Cells with formulas
- Dependency graph between cells
- Circular dependency detection
- Recalculation on change
- Design patterns: Observer, Composite, Memento

### 14.4 Design a Text Editor
- Insert, Delete, Cursor movement
- Undo/Redo (Command + Memento)
- Copy/Paste (Clipboard)
- Find and Replace
- Design patterns: Command, Memento, Iterator

### 14.5 Design a Card Game Framework
- Deck, Cards, Players, Hands
- Extensible for Poker, Blackjack, UNO
- Turn management
- Design patterns: Template Method, Strategy, Factory

### 14.6 Design an LRU Cache
- HashMap + Doubly Linked List
- O(1) get and put
- Eviction policy
- Thread-safe version

### 14.7 Design a Pub-Sub Messaging System
- Topics, Publishers, Subscribers
- Message ordering
- Acknowledgment
- Design patterns: Observer, Mediator

### 14.8 Design a Task Scheduler
- Task with priority and dependencies
- DAG-based execution order
- Concurrent execution
- Retry and timeout
- Design patterns: Strategy, Observer, Command

### 14.9 Design a Rate Limiter
- Token Bucket Algorithm
- Sliding Window Algorithm
- Fixed Window Counter
- Per-user and per-API limiting
- Thread-safe implementation

### 14.10 Design a Multi-Level Cache
- L1 (in-memory), L2 (distributed)
- Cache-aside, Write-through, Write-back policies
- Eviction strategies (LRU, LFU, FIFO)
- Design patterns: Chain of Responsibility, Strategy, Proxy

---

---

# PART B: HIGH-LEVEL DESIGN (HLD)

> HLD focuses on the overall system architecture — how services, databases, caches, queues, and other infrastructure components work together at scale.

---

## Module 15: Networking Fundamentals

### 15.1 OSI Model (7 Layers)
- Physical, Data Link, Network, Transport, Session, Presentation, Application
- What happens at each layer
- Protocols at each layer

### 15.2 TCP/IP Model
- Link, Internet, Transport, Application layers
- TCP vs UDP
  - Three-way handshake
  - Reliability, ordering, flow control
  - When to use TCP vs UDP

### 15.3 IP Addressing & DNS
- IPv4, IPv6
- Subnetting basics
- DNS resolution process (recursive, iterative)
- DNS record types (A, AAAA, CNAME, MX, NS, TXT)
- DNS caching (TTL)

### 15.4 HTTP/HTTPS
- HTTP methods (GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS)
- Status codes (1xx, 2xx, 3xx, 4xx, 5xx)
- Headers (Content-Type, Authorization, Cache-Control, ETag)
- Cookies and Sessions
- HTTPS, TLS/SSL handshake
- HTTP/1.1 vs HTTP/2 vs HTTP/3 (QUIC)
- Keep-Alive connections

### 15.5 WebSockets
- Full-duplex communication
- WebSocket handshake (upgrade from HTTP)
- Use cases (chat, live updates, gaming)
- WebSocket vs Long Polling vs Server-Sent Events (SSE)

### 15.6 Other Protocols
- FTP, SMTP, IMAP/POP3
- MQTT (IoT)
- AMQP (Message Queuing)

---

## Module 16: Application Layer Protocols & APIs

### 16.1 REST (Representational State Transfer)
- REST principles (Stateless, Client-Server, Cacheable, Uniform Interface)
- Resource naming conventions
- HATEOAS
- Versioning (URL, Header, Query Param)
- Pagination (Offset, Cursor-based)
- Filtering, Sorting, Searching
- Rate Limiting headers
- Idempotency

### 16.2 GraphQL
- Schema Definition Language (SDL)
- Queries, Mutations, Subscriptions
- Resolvers
- N+1 problem and DataLoader
- GraphQL vs REST (trade-offs)

### 16.3 gRPC
- Protocol Buffers (protobuf)
- Unary, Server Streaming, Client Streaming, Bidirectional Streaming
- gRPC vs REST
- When to use gRPC (microservices, low latency)

### 16.4 API Gateway
- Routing, Rate Limiting, Authentication
- Request/Response transformation
- API composition
- Examples: Kong, AWS API Gateway, Nginx

### 16.5 API Design Best Practices
- Consistent naming
- Error response format
- Backward compatibility
- API documentation (OpenAPI/Swagger)
- Contract-first vs Code-first

---

## Module 17: Web Architecture Basics

### 17.1 Client-Server Architecture
- Request-Response model
- Stateless vs Stateful servers
- Thin client vs Thick client

### 17.2 Monolithic Architecture
- Single deployable unit
- Advantages and disadvantages
- When monolith is the right choice
- Modular monolith

### 17.3 N-Tier Architecture
- Presentation Layer
- Business Logic Layer
- Data Access Layer
- Benefits of separation

### 17.4 Proxy Servers
- Forward Proxy vs Reverse Proxy
- Use cases (caching, security, load balancing)
- Nginx, HAProxy, Envoy

### 17.5 CDN (Content Delivery Network)
- How CDNs work
- Push vs Pull CDN
- Edge servers
- Cache invalidation in CDN
- Examples: CloudFront, Cloudflare, Akamai

### 17.6 Domain Name System (DNS) in Architecture
- DNS-based load balancing
- GeoDNS
- DNS failover

---

## Module 18: Databases — Relational (SQL)

### 18.1 Relational Database Fundamentals
- Tables, Rows, Columns
- Primary Key, Foreign Key, Composite Key
- Constraints (NOT NULL, UNIQUE, CHECK, DEFAULT)
- Data Types

### 18.2 SQL Deep Dive
- DDL, DML, DCL, TCL
- JOINs (INNER, LEFT, RIGHT, FULL, CROSS, SELF)
- Subqueries, CTEs (Common Table Expressions)
- Window Functions (ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD)
- Aggregations (GROUP BY, HAVING)
- Views, Materialized Views

### 18.3 Normalization & Denormalization
- 1NF, 2NF, 3NF, BCNF
- When to denormalize
- Trade-offs (read performance vs write complexity)

### 18.4 Indexing
- B-Tree Index, B+ Tree Index
- Hash Index
- Composite Index
- Covering Index
- Partial Index
- Full-Text Index
- Index selectivity
- When NOT to index

### 18.5 Transactions & ACID
- Atomicity, Consistency, Isolation, Durability
- Isolation Levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable)
- Dirty Read, Non-Repeatable Read, Phantom Read
- Optimistic vs Pessimistic Locking
- MVCC (Multi-Version Concurrency Control)
- Two-Phase Locking (2PL)
- Deadlock detection and prevention

### 18.6 Popular SQL Databases
- MySQL / MariaDB
- PostgreSQL
- Oracle
- SQL Server
- SQLite

---

## Module 19: Databases — Non-Relational (NoSQL)

### 19.1 NoSQL Categories
- Key-Value Stores (Redis, DynamoDB, Memcached)
- Document Stores (MongoDB, CouchDB)
- Column-Family Stores (Cassandra, HBase)
- Graph Databases (Neo4j, Amazon Neptune)
- Time-Series Databases (InfluxDB, TimescaleDB)
- Search Engines (Elasticsearch, Solr)

### 19.2 Key-Value Stores
- Data model
- Use cases (caching, session storage, counters)
- Redis data structures (Strings, Lists, Sets, Sorted Sets, Hashes, Streams)
- Redis persistence (RDB, AOF)
- Redis Cluster

### 19.3 Document Stores
- JSON/BSON documents
- Schema flexibility
- Embedded documents vs References
- MongoDB indexing, aggregation pipeline
- Use cases (content management, catalogs)

### 19.4 Column-Family Stores
- Wide-column model
- Partition Key, Clustering Key
- Cassandra architecture (ring, gossip protocol, consistent hashing)
- Write path, Read path
- Use cases (time-series, IoT, logging)

### 19.5 Graph Databases
- Nodes, Edges, Properties
- Cypher query language (Neo4j)
- Graph traversal
- Use cases (social networks, recommendation engines, fraud detection)

### 19.6 SQL vs NoSQL Decision Framework
- Data structure requirements
- Consistency vs Availability needs
- Read/Write patterns
- Scale requirements
- Team expertise

---

## Module 20: Database Deep Dive

### 20.1 CAP Theorem
- Consistency, Availability, Partition Tolerance
- CP systems (HBase, MongoDB with strong consistency)
- AP systems (Cassandra, DynamoDB)
- CA systems (single-node RDBMS)
- PACELC theorem (extension of CAP)

### 20.2 ACID vs BASE
- ACID (Atomicity, Consistency, Isolation, Durability)
- BASE (Basically Available, Soft state, Eventually consistent)
- When to choose which

### 20.3 Database Replication
- Master-Slave (Primary-Replica) Replication
- Master-Master (Multi-Primary) Replication
- Synchronous vs Asynchronous Replication
- Replication Lag
- Read Replicas
- Conflict Resolution in Multi-Master

### 20.4 Database Sharding / Partitioning
- Horizontal Partitioning (Sharding)
- Vertical Partitioning
- Sharding Strategies
  - Range-based sharding
  - Hash-based sharding
  - Directory-based sharding
  - Geographic sharding
- Shard key selection
- Cross-shard queries
- Resharding challenges

### 20.5 Database Internals
- Write-Ahead Log (WAL)
- B-Tree vs LSM Tree storage engines
- SSTables and Compaction
- Bloom Filters
- Page cache
- Buffer pool
- Query optimizer basics

### 20.6 Data Warehousing & OLAP
- OLTP vs OLAP
- Star Schema, Snowflake Schema
- ETL (Extract, Transform, Load)
- Column-oriented storage
- Data Lake vs Data Warehouse

---

## Module 21: Caching

### 21.1 Caching Fundamentals
- Why cache? (Latency, Throughput, Cost)
- Cache Hit, Cache Miss
- Hit Ratio
- Cache Warm-up

### 21.2 Caching Strategies
- Cache-Aside (Lazy Loading)
- Read-Through
- Write-Through
- Write-Behind (Write-Back)
- Write-Around
- Refresh-Ahead

### 21.3 Cache Eviction Policies
- LRU (Least Recently Used)
- LFU (Least Frequently Used)
- FIFO (First In First Out)
- TTL (Time To Live)
- Random Replacement
- ARC (Adaptive Replacement Cache)

### 21.4 Caching Layers
- Browser Cache
- CDN Cache
- Application-Level Cache (in-process)
- Distributed Cache (Redis, Memcached)
- Database Query Cache
- OS Page Cache

### 21.5 Cache Invalidation
- TTL-based expiration
- Event-based invalidation
- Version-based invalidation
- Cache stampede / Thundering herd problem
  - Locking
  - Probabilistic early expiration

### 21.6 Distributed Caching
- Redis Cluster
- Memcached
- Consistent Hashing for cache distribution
- Cache replication
- Cache partitioning

### 21.7 Common Caching Problems
- Cache Penetration (querying non-existent keys)
- Cache Breakdown (hot key expiration)
- Cache Avalanche (mass expiration)
- Solutions: Bloom filters, mutex locks, staggered TTL

---

## Module 22: Scalability

### 22.1 Vertical Scaling (Scale Up)
- Adding more CPU, RAM, Disk
- Limits of vertical scaling
- When vertical scaling is appropriate

### 22.2 Horizontal Scaling (Scale Out)
- Adding more machines
- Stateless services
- Session management (sticky sessions, external session store)
- Data partitioning

### 22.3 Database Scaling
- Read Replicas
- Sharding
- Connection Pooling
- Query optimization
- Denormalization for read performance

### 22.4 Application Scaling
- Stateless application design
- Horizontal Pod Autoscaling
- Auto-scaling groups
- Scaling microservices independently

### 22.5 Scaling Patterns
- CQRS (Command Query Responsibility Segregation)
- Event Sourcing
- Materialized Views
- Database per Service
- Saga Pattern for distributed transactions

### 22.6 Performance Optimization
- Latency vs Throughput
- P50, P95, P99 latencies
- Amdahl's Law
- Connection pooling
- Batch processing
- Async processing
- Compression (gzip, brotli)

---

## Module 23: Load Balancing

### 23.1 Load Balancer Basics
- What is a Load Balancer?
- Layer 4 (Transport) vs Layer 7 (Application) Load Balancing
- Hardware vs Software Load Balancers

### 23.2 Load Balancing Algorithms
- Round Robin
- Weighted Round Robin
- Least Connections
- Weighted Least Connections
- IP Hash
- Consistent Hashing
- Random
- Least Response Time
- Resource-Based (Adaptive)

### 23.3 Health Checks
- Active health checks (ping, HTTP check)
- Passive health checks (monitoring failures)
- Graceful degradation

### 23.4 Advanced Load Balancing
- Global Server Load Balancing (GSLB)
- DNS-based load balancing
- Anycast
- Service Mesh load balancing (Envoy, Istio)
- Client-side load balancing

### 23.5 High Availability
- Active-Passive (Failover)
- Active-Active
- Floating IP / Virtual IP
- Redundancy at every layer

### 23.6 Load Balancer Tools
- Nginx
- HAProxy
- AWS ELB (ALB, NLB, CLB)
- Envoy Proxy
- Traefik

---

## Module 24: Message Queues & Event-Driven Architecture

### 24.1 Message Queue Fundamentals
- Producer, Consumer, Broker
- Point-to-Point vs Publish-Subscribe
- Message ordering
- At-most-once, At-least-once, Exactly-once delivery
- Dead Letter Queue (DLQ)
- Message acknowledgment

### 24.2 Popular Message Queues
- Apache Kafka
  - Topics, Partitions, Consumer Groups
  - Log-based storage
  - Offset management
  - Kafka Streams
- RabbitMQ
  - Exchanges (Direct, Fanout, Topic, Headers)
  - Queues, Bindings
  - AMQP protocol
- Amazon SQS / SNS
- Apache Pulsar
- Redis Streams

### 24.3 Event-Driven Architecture
- Events vs Commands vs Queries
- Event Sourcing
- CQRS (Command Query Responsibility Segregation)
- Event Store
- Eventual Consistency
- Saga Pattern (Orchestration vs Choreography)

### 24.4 Stream Processing
- Batch Processing vs Stream Processing
- Apache Kafka Streams
- Apache Flink (concepts)
- Windowing (Tumbling, Sliding, Session)
- Watermarks and Late Data

### 24.5 Use Cases
- Order processing pipeline
- Notification system
- Log aggregation
- Real-time analytics
- Decoupling microservices

---

## Module 25: Microservices Architecture

### 25.1 Microservices Fundamentals
- Monolith vs Microservices
- Characteristics of Microservices
- Bounded Context (Domain-Driven Design)
- Single Responsibility per Service
- Independent deployment

### 25.2 Communication Patterns
- Synchronous (REST, gRPC)
- Asynchronous (Message Queues, Events)
- Request-Reply pattern
- Choreography vs Orchestration
- API Gateway pattern

### 25.3 Service Discovery
- Client-side discovery
- Server-side discovery
- Service Registry (Consul, Eureka, etcd, ZooKeeper)
- DNS-based discovery

### 25.4 Data Management
- Database per Service pattern
- Shared Database (anti-pattern)
- Saga Pattern for distributed transactions
- Event Sourcing
- CQRS
- Data consistency strategies

### 25.5 Resilience Patterns
- Circuit Breaker (Open, Half-Open, Closed states)
- Retry with Exponential Backoff
- Timeout
- Bulkhead
- Fallback
- Rate Limiting

### 25.6 Service Mesh
- Sidecar Proxy pattern
- Istio, Linkerd, Envoy
- Traffic management
- Mutual TLS (mTLS)
- Observability

### 25.7 Decomposition Strategies
- Decompose by Business Capability
- Decompose by Subdomain
- Strangler Fig Pattern (migrating from monolith)
- Anti-Corruption Layer

---

## Module 26: Distributed Systems Fundamentals

### 26.1 Core Concepts
- What makes a system "distributed"?
- Fallacies of Distributed Computing (8 fallacies)
- Network partitions
- Partial failures
- Clock synchronization issues

### 26.2 Consistency Models
- Strong Consistency
- Eventual Consistency
- Causal Consistency
- Read-your-writes Consistency
- Monotonic Reads
- Linearizability vs Serializability

### 26.3 Consensus Algorithms
- Why consensus is hard
- Paxos (basic idea)
- Raft (Leader Election, Log Replication, Safety)
- ZAB (ZooKeeper Atomic Broadcast)
- Byzantine Fault Tolerance (BFT) — overview

### 26.4 Distributed Coordination
- ZooKeeper
  - Leader election
  - Distributed locking
  - Configuration management
  - Service discovery
- etcd
- Consul

### 26.5 Distributed Transactions
- Two-Phase Commit (2PC)
- Three-Phase Commit (3PC)
- Saga Pattern
- Compensating Transactions
- Outbox Pattern

### 26.6 Time & Ordering
- Physical Clocks vs Logical Clocks
- Lamport Timestamps
- Vector Clocks
- Hybrid Logical Clocks (HLC)
- NTP (Network Time Protocol)

---

## Module 27: Distributed Systems — Advanced

### 27.1 Consistent Hashing
- Hash Ring
- Virtual Nodes (vnodes)
- Adding/Removing nodes
- Applications (distributed caches, databases, load balancing)

### 27.2 Gossip Protocol
- How gossip works
- Failure detection
- Membership protocol
- Used in: Cassandra, DynamoDB, Consul

### 27.3 Quorum-Based Systems
- Read Quorum, Write Quorum
- W + R > N for strong consistency
- Sloppy Quorum
- Hinted Handoff

### 27.4 Conflict Resolution
- Last-Write-Wins (LWW)
- Vector Clocks
- CRDTs (Conflict-free Replicated Data Types)
  - G-Counter, PN-Counter
  - G-Set, OR-Set
  - LWW-Register
- Application-level conflict resolution

### 27.5 MapReduce & Distributed Computing
- MapReduce paradigm
- Map phase, Shuffle phase, Reduce phase
- Hadoop ecosystem (overview)
- Spark (overview)

### 27.6 Bloom Filters & Probabilistic Data Structures
- Bloom Filter (false positives, no false negatives)
- Counting Bloom Filter
- HyperLogLog (cardinality estimation)
- Count-Min Sketch (frequency estimation)
- Skip List

---

## Module 28: Storage & File Systems

### 28.1 Storage Types
- Block Storage (EBS, SAN)
- File Storage (NFS, EFS)
- Object Storage (S3, GCS, Azure Blob)
- When to use which

### 28.2 Distributed File Systems
- HDFS (Hadoop Distributed File System)
- GFS (Google File System) — paper concepts
- Ceph
- Architecture: NameNode, DataNode, Replication

### 28.3 Object Storage
- S3-like architecture
- Buckets, Objects, Keys
- Versioning
- Lifecycle policies
- Pre-signed URLs
- Multipart upload

### 28.4 Data Serialization
- JSON, XML
- Protocol Buffers (protobuf)
- Apache Avro
- Apache Thrift
- MessagePack
- Schema evolution and backward compatibility

### 28.5 Data Compression
- gzip, brotli, snappy, lz4, zstd
- Compression vs Speed trade-offs
- When to compress (storage, network transfer)

---

## Module 29: Security & Authentication

### 29.1 Authentication
- Username/Password
- Multi-Factor Authentication (MFA)
- OAuth 2.0 (Authorization Code, Client Credentials, Implicit, PKCE)
- OpenID Connect (OIDC)
- SAML
- JWT (JSON Web Tokens) — structure, signing, verification
- Session-based vs Token-based authentication

### 29.2 Authorization
- RBAC (Role-Based Access Control)
- ABAC (Attribute-Based Access Control)
- ACL (Access Control Lists)
- Policy engines

### 29.3 Encryption
- Symmetric Encryption (AES)
- Asymmetric Encryption (RSA, ECC)
- Hashing (SHA-256, bcrypt, argon2)
- Encryption at rest vs in transit
- TLS/SSL
- Certificate management

### 29.4 Common Security Concerns in System Design
- SQL Injection
- XSS (Cross-Site Scripting)
- CSRF (Cross-Site Request Forgery)
- DDoS protection
- Rate limiting
- Input validation
- Secrets management (Vault, AWS Secrets Manager)

### 29.5 Network Security
- Firewalls, Security Groups
- VPC, Subnets (Public, Private)
- VPN, Bastion Host
- WAF (Web Application Firewall)
- Zero Trust Architecture

---

## Module 30: Monitoring, Logging & Observability

### 30.1 Three Pillars of Observability
- Metrics (quantitative measurements)
- Logs (discrete events)
- Traces (request flow across services)

### 30.2 Metrics & Monitoring
- Types: Counter, Gauge, Histogram, Summary
- Prometheus + Grafana
- CloudWatch, Datadog
- Alerting (thresholds, anomaly detection)
- SLI, SLO, SLA
- Error budgets

### 30.3 Logging
- Structured logging (JSON)
- Centralized logging (ELK Stack: Elasticsearch, Logstash, Kibana)
- Log levels
- Correlation IDs for distributed tracing
- Log aggregation and retention

### 30.4 Distributed Tracing
- OpenTelemetry
- Jaeger, Zipkin
- Trace ID, Span ID
- Trace propagation
- Sampling strategies

### 30.5 Health Checks & Readiness
- Liveness probes
- Readiness probes
- Startup probes
- Graceful shutdown

---

## Module 31: DevOps & Deployment

### 31.1 Containerization
- Docker basics (images, containers, Dockerfile)
- Docker Compose
- Container registries
- Multi-stage builds

### 31.2 Orchestration
- Kubernetes basics (Pods, Services, Deployments, ReplicaSets)
- Horizontal Pod Autoscaler
- ConfigMaps, Secrets
- Ingress controllers
- Namespaces

### 31.3 CI/CD
- Continuous Integration
- Continuous Delivery vs Continuous Deployment
- Pipeline stages (Build, Test, Deploy)
- Blue-Green Deployment
- Canary Deployment
- Rolling Deployment
- Feature Flags

### 31.4 Infrastructure as Code
- Terraform (concepts)
- CloudFormation
- Ansible
- Immutable infrastructure

### 31.5 Cloud Services (Overview)
- Compute (EC2, Lambda, ECS, EKS)
- Storage (S3, EBS, EFS)
- Database (RDS, DynamoDB, ElastiCache)
- Networking (VPC, Route53, CloudFront, ELB)
- Messaging (SQS, SNS, EventBridge)

---

## Module 32: Back-of-the-Envelope Estimation

### 32.1 Latency Numbers Every Engineer Should Know
- L1 cache reference: ~1 ns
- L2 cache reference: ~4 ns
- Main memory reference: ~100 ns
- SSD random read: ~16 us
- HDD seek: ~4 ms
- Network round trip (same datacenter): ~0.5 ms
- Network round trip (cross-continent): ~150 ms

### 32.2 Storage Estimation
- Character = 1 byte (ASCII) or 2-4 bytes (UTF-8)
- Integer = 4 bytes, Long = 8 bytes
- UUID = 16 bytes
- Timestamp = 8 bytes
- Image (compressed) = 200 KB - 2 MB
- Video (1 min, compressed) = 5-50 MB

### 32.3 Traffic Estimation
- DAU (Daily Active Users)
- QPS (Queries Per Second) = DAU * avg_queries / 86400
- Peak QPS = QPS * 2-5x
- Read:Write ratio

### 32.4 Bandwidth Estimation
- Bandwidth = QPS * request/response size
- Upload vs Download bandwidth
- CDN offloading

### 32.5 Common Calculations
- How many servers needed?
- How much storage for 5 years?
- How many shards needed?
- Cache size estimation
- Powers of 2 (KB, MB, GB, TB, PB)

### 32.6 Practice
- Estimate storage for Twitter (1 year)
- Estimate QPS for Google Search
- Estimate bandwidth for YouTube
- Estimate cache size for URL shortener

---


## Module 33: HLD Practice Problems (Easy)

### 33.1 Design a URL Shortener (TinyURL)
- Key topics: Hashing, Base62 encoding, Database choice, Caching, Analytics
- Scale: 100M URLs/month, 10:1 read:write ratio
- Components: API Gateway, Application Server, Database, Cache, Analytics

### 33.2 Design a Pastebin
- Key topics: Object storage, Expiration, Access control
- Scale: 5M pastes/day
- Components: API, Metadata DB, Object Storage (S3), CDN

### 33.3 Design a Rate Limiter
- Key topics: Token Bucket, Sliding Window, Distributed rate limiting
- Scale: Millions of requests/second
- Components: API Gateway, Redis, Rules Engine

### 33.4 Design a Key-Value Store
- Key topics: Consistent hashing, Replication, Conflict resolution
- Scale: High throughput, Low latency
- Components: Coordinator, Storage nodes, Gossip protocol

### 33.5 Design a Unique ID Generator
- Key topics: UUID, Snowflake ID, Database auto-increment, Clock sync
- Scale: 10K IDs/second across multiple datacenters
- Components: ID service, ZooKeeper for coordination

---

## Module 34: HLD Practice Problems (Medium)

### 34.1 Design Twitter / News Feed
- Key topics: Fan-out on write vs Fan-out on read, Timeline generation, Caching
- Scale: 500M users, 600K tweets/second read
- Components: Tweet Service, Fan-out Service, Timeline Cache, Search, Notification

### 34.2 Design Instagram / Photo Sharing
- Key topics: Object storage, CDN, News feed, Image processing
- Scale: 1B users, 100M photos/day
- Components: Upload Service, Image Processing, CDN, Feed Service, Search

### 34.3 Design WhatsApp / Chat System
- Key topics: WebSockets, Message queue, Presence, Group chat, E2E encryption
- Scale: 2B users, 100B messages/day
- Components: Chat Server, WebSocket Manager, Message Queue, Presence Service

### 34.4 Design Notification System
- Key topics: Push, SMS, Email channels, Priority, Rate limiting, Templates
- Scale: 10B notifications/day
- Components: Notification Service, Channel Adapters, Message Queue, Analytics

### 34.5 Design a Web Crawler
- Key topics: BFS/DFS crawling, URL frontier, Politeness, Deduplication
- Scale: 1B pages/month
- Components: URL Frontier, Fetcher, Parser, Dedup Service, DNS Resolver

### 34.6 Design Search Autocomplete / Typeahead
- Key topics: Trie, Prefix matching, Ranking, Caching
- Scale: 10K QPS
- Components: Trie Service, Aggregation Service, Cache, Data Collection

### 34.7 Design a Distributed Cache
- Key topics: Consistent hashing, Eviction, Replication, Partitioning
- Scale: Millions of ops/second
- Components: Cache nodes, Coordinator, Health checker

### 34.8 Design an API Rate Limiter (Distributed)
- Key topics: Token bucket, Sliding window log, Redis, Race conditions
- Scale: 100K+ clients
- Components: Rate Limiter Service, Redis Cluster, Rules DB

### 34.9 Design Uber / Ride Sharing
- Key topics: Location tracking, Matching algorithm, ETA, Surge pricing, Maps
- Scale: 20M rides/day
- Components: Location Service, Matching Service, Trip Service, Payment, Maps/ETA

### 34.10 Design a Hotel / Flight Booking System
- Key topics: Inventory management, Double booking prevention, Payment, Search
- Scale: 1M bookings/day
- Components: Search Service, Booking Service, Inventory Service, Payment, Notification

---

## Module 35: HLD Practice Problems (Hard)

### 35.1 Design YouTube / Video Streaming
- Key topics: Video upload, Transcoding, Adaptive bitrate streaming, CDN, Recommendation
- Scale: 2B users, 500 hours video uploaded/minute
- Components: Upload Service, Transcoding Pipeline, CDN, Streaming Service, Recommendation Engine, Search

### 35.2 Design Google Docs / Collaborative Editing
- Key topics: OT (Operational Transformation), CRDT, WebSockets, Conflict resolution
- Scale: 100M documents, 10M concurrent editors
- Components: Document Service, Collaboration Engine, WebSocket Server, Version History

### 35.3 Design Google Maps
- Key topics: Graph algorithms, Tile rendering, Geospatial indexing, ETA, Traffic
- Scale: 1B users, real-time traffic
- Components: Map Tile Service, Routing Service, Geocoding, Traffic Service, Search

### 35.4 Design Google Search
- Key topics: Web crawling, Indexing, PageRank, Query processing, Spell check
- Scale: 8.5B searches/day
- Components: Crawler, Indexer, Query Processor, Ranking Service, Spell Checker, Cache

### 35.5 Design Dropbox / Google Drive
- Key topics: File sync, Chunking, Deduplication, Conflict resolution, Versioning
- Scale: 1B files/day synced
- Components: Block Server, Metadata Service, Sync Service, Notification, Cold Storage

### 35.6 Design a Distributed Message Queue (Kafka-like)
- Key topics: Log-based storage, Partitioning, Replication, Consumer groups, Ordering
- Scale: Millions of messages/second
- Components: Broker, ZooKeeper/Controller, Producer, Consumer, Partition Manager

### 35.7 Design a Payment System
- Key topics: Idempotency, Double-spend prevention, Reconciliation, Ledger
- Scale: 1M transactions/day
- Components: Payment Gateway, Ledger Service, Risk Engine, Reconciliation, Notification

### 35.8 Design a Social Network (Facebook)
- Key topics: News feed, Friend graph, Notifications, Search, Ads
- Scale: 3B users
- Components: User Service, Graph Service, Feed Service, Search, Notification, Ad Service

### 35.9 Design a Ticket Booking System (Ticketmaster)
- Key topics: Seat locking, High concurrency, Waiting queue, Fairness
- Scale: 50K concurrent users for popular events
- Components: Event Service, Booking Service, Queue Service, Payment, Notification

### 35.10 Design a Real-Time Gaming Leaderboard
- Key topics: Sorted sets, Real-time updates, Sharding, Consistency
- Scale: 100M players, 1M score updates/second
- Components: Score Ingestion, Leaderboard Service (Redis Sorted Sets), Sharding Layer

### 35.11 Design a Content Moderation System
- Key topics: ML pipeline, Human review, Queue management, Appeals
- Scale: 1B posts/day
- Components: ML Service, Review Queue, Human Review Tool, Appeals Service, Rules Engine

### 35.12 Design a Metrics/Monitoring System (Prometheus-like)
- Key topics: Time-series DB, Aggregation, Alerting, Dashboards
- Scale: 10M metrics, 1M data points/second
- Components: Collector, Time-Series DB, Query Engine, Alert Manager, Dashboard

---

---

# PART C: INTERVIEW PREPARATION

---

## Module 36: LLD Interview Framework

### 36.1 Step-by-Step Approach
1. **Clarify Requirements** (2-3 min)
   - Functional requirements (what the system does)
   - Non-functional requirements (scale, performance)
   - Scope (what to include, what to exclude)

2. **Identify Core Objects/Entities** (3-5 min)
   - Nouns from requirements become classes
   - Verbs become methods
   - Adjectives become attributes

3. **Define Relationships** (3-5 min)
   - Inheritance (is-a)
   - Composition (has-a, strong)
   - Aggregation (has-a, weak)
   - Association (uses)

4. **Apply Design Patterns** (5-10 min)
   - Identify which patterns fit
   - Justify your choice
   - Show trade-offs

5. **Draw Class Diagram** (5-10 min)
   - Classes with attributes and methods
   - Relationships with multiplicity
   - Interfaces and abstract classes

6. **Write Core Code** (10-15 min)
   - Key classes and interfaces
   - Critical methods
   - Pattern implementations

7. **Discuss Trade-offs** (2-3 min)
   - Extensibility
   - Testability
   - Performance

### 36.2 Common Mistakes in LLD Interviews
- Jumping to code without clarifying requirements
- Not using design patterns where appropriate
- Over-engineering (too many patterns)
- Ignoring concurrency
- Not considering edge cases
- Tight coupling between classes

---

## Module 37: HLD Interview Framework

### 37.1 Step-by-Step Approach
1. **Clarify Requirements** (3-5 min)
   - Functional requirements
   - Non-functional requirements (latency, throughput, availability, consistency)
   - Scale (users, data, QPS)
   - Constraints

2. **Back-of-the-Envelope Estimation** (3-5 min)
   - QPS (read and write)
   - Storage (daily, yearly, 5-year)
   - Bandwidth
   - Cache size

3. **High-Level Architecture** (5-10 min)
   - Draw main components
   - Show data flow
   - API design (key endpoints)

4. **Deep Dive into Components** (10-15 min)
   - Database schema and choice
   - Caching strategy
   - Data partitioning
   - Replication

5. **Address Bottlenecks & Scale** (5-10 min)
   - Identify bottlenecks
   - Horizontal scaling
   - Load balancing
   - CDN, Message queues

6. **Discuss Trade-offs** (3-5 min)
   - Consistency vs Availability
   - Latency vs Throughput
   - Cost vs Performance
   - Complexity vs Simplicity

### 37.2 Common Mistakes in HLD Interviews
- Not clarifying requirements upfront
- Skipping estimation
- Designing for too small or too large scale
- Not discussing trade-offs
- Ignoring failure scenarios
- Over-focusing on one component
- Not considering security
- Forgetting monitoring and alerting

---

## Module 38: Common Mistakes & Tips

### 38.1 General Tips
- Practice drawing diagrams (whiteboard or draw.io)
- Think out loud during interviews
- Start simple, then add complexity
- Always discuss trade-offs
- Know your numbers (latency, storage, QPS)
- Study real-world architectures (Netflix, Uber, Twitter engineering blogs)

### 38.2 Resources
- Books:
  - "Designing Data-Intensive Applications" by Martin Kleppmann
  - "System Design Interview" by Alex Xu (Vol 1 & 2)
  - "Clean Code" by Robert C. Martin
  - "Design Patterns: Elements of Reusable Object-Oriented Software" (GoF)
  - "Head First Design Patterns"
  - "Clean Architecture" by Robert C. Martin
  - "The C++ Programming Language" by Bjarne Stroustrup
  - "Effective Modern C++" by Scott Meyers

- Online:
  - System Design Primer (GitHub)
  - Grokking the System Design Interview
  - ByteByteGo (Alex Xu)
  - Engineering blogs (Netflix, Uber, Airbnb, Meta, Google)
  - LeetCode (for LLD problems)

### 38.3 Study Plan
| Week | Focus |
|------|-------|
| 1-2 | C++ OOP Fundamentals (Modules 1-2) |
| 3-4 | SOLID Principles + Creational Patterns (Modules 3-4) |
| 5-6 | Structural + Behavioral Patterns (Modules 5-6) |
| 7 | UML + Advanced OOP (Modules 7-8) |
| 8 | Concurrency + DS/Algo for LLD (Modules 9-10) |
| 9-10 | LLD Practice Problems Easy + Medium (Modules 12-13) |
| 11 | LLD Practice Problems Hard (Module 14) |
| 12-13 | Networking + APIs + Web Architecture (Modules 15-17) |
| 14-15 | Databases SQL + NoSQL + Deep Dive (Modules 18-20) |
| 16 | Caching + Scalability (Modules 21-22) |
| 17 | Load Balancing + Message Queues (Modules 23-24) |
| 18 | Microservices + Distributed Systems (Modules 25-26) |
| 19 | Advanced Distributed Systems + Storage (Modules 27-28) |
| 20 | Security + Monitoring + DevOps (Modules 29-31) |
| 21 | Estimation + HLD Practice Easy (Modules 32-33) |
| 22-23 | HLD Practice Medium (Module 34) |
| 24-25 | HLD Practice Hard (Module 35) |
| 26 | Interview Frameworks + Mock Interviews (Modules 36-38) |

---

> **Total Duration: ~26 weeks (6 months) for comprehensive coverage**
> **Accelerated Path: ~12 weeks if you already know C++ and OOP basics**

---

*Generated for System Design preparation with C++ focus.*
*Last updated: April 2026*
