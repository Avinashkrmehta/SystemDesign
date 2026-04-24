# Module 36: LLD Interview Framework

> Low-Level Design (LLD) interviews test your ability to design the internal structure of a system — classes, interfaces, relationships, and design patterns. Unlike HLD (which focuses on distributed architecture), LLD focuses on **object-oriented design** and **clean code**. This module provides a step-by-step framework for approaching any LLD interview problem, with examples and common mistakes to avoid.

---

## 36.1 Step-by-Step Approach

> Follow this framework for every LLD interview. It keeps you organized, ensures you cover all aspects, and demonstrates a structured thought process to the interviewer.

---

### Step 1: Clarify Requirements (2-3 minutes)

**Never start designing without understanding what you're building.** Ask questions to narrow the scope.

**What to Ask:**

| Category | Example Questions |
|----------|------------------|
| **Functional** | "What are the core features?" "Should users be able to X?" "Do we need Y?" |
| **Non-Functional** | "How many concurrent users?" "Does it need to be thread-safe?" "What about persistence?" |
| **Scope** | "Should I design the full system or focus on the core?" "Is the UI in scope?" |
| **Constraints** | "Any specific language/framework?" "Memory constraints?" "Real-time requirements?" |

**Example — Parking Lot System:**

```
Questions to ask:
  - How many floors? How many spots per floor?
  - What vehicle types? (Car, Bike, Truck)
  - What spot types? (Small, Medium, Large)
  - Do we need payment? Hourly rate?
  - Multiple entry/exit gates?
  - Do we need to find the nearest available spot?
  - Is concurrency a concern? (multiple cars entering simultaneously)

Scope decision:
  ✅ In scope: Vehicle parking, spot allocation, payment calculation
  ❌ Out of scope: UI, database persistence, payment gateway integration
```

**Key Rule:** Spend 2-3 minutes here. Don't rush. The interviewer wants to see that you think before you code.

---

### Step 2: Identify Core Objects/Entities (3-5 minutes)

Extract **classes** from the requirements using a simple linguistic technique:

```
Nouns → Classes
  "A parking lot has multiple floors. Each floor has parking spots.
   Vehicles enter through gates and park in spots. A ticket is issued."
  
  Classes: ParkingLot, Floor, ParkingSpot, Vehicle, Gate, Ticket

Verbs → Methods
  "Vehicles enter, park, exit. Tickets are issued and paid."
  
  Methods: enter(), park(), exit(), issueTicket(), pay()

Adjectives → Attributes or Enums
  "Small, medium, large spots. Car, bike, truck vehicles."
  
  Enums: SpotType {SMALL, MEDIUM, LARGE}, VehicleType {CAR, BIKE, TRUCK}
```

**List the Core Entities:**

```
Parking Lot System — Core Entities:
  1. ParkingLot       — the main system (Singleton)
  2. Floor            — a level in the parking lot
  3. ParkingSpot      — an individual spot (Small, Medium, Large)
  4. Vehicle          — a vehicle (Car, Bike, Truck)
  5. Ticket           — issued on entry, used for payment
  6. Gate             — entry/exit point
  7. Payment          — payment calculation
  8. DisplayBoard     — shows available spots per floor
```

---

### Step 3: Define Relationships (3-5 minutes)

Determine how classes relate to each other:

| Relationship | Meaning | UML | Example |
|-------------|---------|-----|---------|
| **Inheritance (is-a)** | Subclass extends superclass | `──▷` (hollow arrow) | `Car` is-a `Vehicle` |
| **Composition (has-a, strong)** | Part cannot exist without the whole; lifecycle dependency | `──◆` (filled diamond) | `ParkingLot` has `Floor`s (floors don't exist without the lot) |
| **Aggregation (has-a, weak)** | Part can exist independently | `──◇` (hollow diamond) | `Floor` has `ParkingSpot`s |
| **Association (uses)** | One class uses another | `──` (plain line) | `Ticket` references a `Vehicle` |
| **Dependency (uses temporarily)** | One class uses another in a method | `--→` (dashed arrow) | `PaymentService` uses `Ticket` to calculate amount |

**Parking Lot Relationships:**

```
ParkingLot ◆── Floor (1..*)         — composition (lot owns floors)
Floor ◆── ParkingSpot (1..*)        — composition (floor owns spots)
ParkingSpot ── Vehicle (0..1)       — association (spot may have a vehicle)
ParkingLot ◆── Gate (1..*)          — composition
Gate ── Ticket                      — association (gate issues tickets)
Ticket ── Vehicle                   — association (ticket references vehicle)

Vehicle ──▷ Car, Bike, Truck        — inheritance
ParkingSpot ──▷ SmallSpot, MediumSpot, LargeSpot — inheritance (or use enum)
```

---

### Step 4: Apply Design Patterns (5-10 minutes)

Identify which design patterns fit the problem. **Don't force patterns** — use them only when they solve a real problem.

**Common Patterns in LLD Interviews:**

| Pattern | When to Use | Example |
|---------|-------------|---------|
| **Singleton** | Only one instance should exist | ParkingLot, Logger, Configuration |
| **Factory** | Create objects without specifying exact class | VehicleFactory, SpotFactory |
| **Strategy** | Multiple algorithms, swap at runtime | PricingStrategy (hourly, flat, weekend), ParkingStrategy (nearest, random) |
| **Observer** | Notify multiple objects when state changes | DisplayBoard observes ParkingLot for spot availability changes |
| **State** | Object behavior changes based on state | VendingMachine states (Idle, HasMoney, Dispensing) |
| **Command** | Encapsulate actions as objects, support undo | Text editor (type, delete, undo, redo) |
| **Decorator** | Add behavior dynamically | Pizza toppings, coffee add-ons |
| **Chain of Responsibility** | Pass request through a chain of handlers | Logger (DEBUG → INFO → ERROR), ATM dispenser (100s → 50s → 20s) |
| **Template Method** | Define algorithm skeleton, let subclasses fill steps | Game turn (roll dice → move → action → end turn) |
| **Iterator** | Traverse a collection without exposing internals | Custom collection traversal |

**Parking Lot Patterns:**

```
1. Singleton: ParkingLot (only one instance)
2. Strategy: PricingStrategy (different rates for different vehicle types, time of day)
3. Factory: VehicleFactory (create Car, Bike, Truck based on type)
4. Observer: DisplayBoard observes spot availability changes
```

**How to Present Patterns:**
- Name the pattern
- Explain WHY it fits (what problem it solves)
- Show the trade-off (what you gain vs what complexity it adds)

```
"I'll use the Strategy pattern for pricing because we have multiple pricing
algorithms (hourly, flat rate, weekend rate) and we want to swap them at
runtime without modifying the ParkingLot class. This follows the Open/Closed
Principle — we can add new pricing strategies without changing existing code."
```

---

### Step 5: Draw Class Diagram (5-10 minutes)

Draw the classes with their attributes, methods, and relationships.

```
+---------------------------+
| <<Singleton>>             |
| ParkingLot                |
+---------------------------+
| - floors: List<Floor>     |
| - gates: List<Gate>       |
| - displayBoard: DisplayBoard |
+---------------------------+
| + getInstance(): ParkingLot |
| + addFloor(floor): void   |
| + parkVehicle(vehicle): Ticket |
| + unparkVehicle(ticket): Payment |
| + getAvailableSpots(): int |
+---------------------------+
         ◆ 1..*
         |
+---------------------------+
| Floor                     |
+---------------------------+
| - floorNumber: int        |
| - spots: List<ParkingSpot>|
+---------------------------+
| + getAvailableSpots(type): List<ParkingSpot> |
| + parkVehicle(vehicle): ParkingSpot |
+---------------------------+
         ◆ 1..*
         |
+---------------------------+       +---------------------------+
| <<abstract>>              |       | <<interface>>             |
| ParkingSpot               |       | PricingStrategy           |
+---------------------------+       +---------------------------+
| - spotId: string          |       | + calculatePrice(         |
| - spotType: SpotType      |       |     ticket: Ticket): double|
| - vehicle: Vehicle (null) |       +---------------------------+
| - isAvailable: bool       |            ▲           ▲
+---------------------------+            |           |
| + park(vehicle): bool     |   +--------+--+ +-----+--------+
| + unpark(): Vehicle       |   |HourlyPricing| |FlatPricing  |
| + canFit(vehicle): bool   |   +-------------+ +-------------+
+---------------------------+
     ▲        ▲        ▲
     |        |        |
+--------+ +--------+ +--------+
|SmallSpot| |MediumSpot| |LargeSpot|
+--------+ +--------+ +--------+

+---------------------------+
| <<abstract>>              |
| Vehicle                   |
+---------------------------+
| - licensePlate: string    |
| - vehicleType: VehicleType|
+---------------------------+
     ▲        ▲        ▲
     |        |        |
  +-----+  +-----+  +-------+
  | Car |  | Bike|  | Truck |
  +-----+  +-----+  +-------+

+---------------------------+
| Ticket                    |
+---------------------------+
| - ticketId: string        |
| - vehicle: Vehicle        |
| - spot: ParkingSpot       |
| - entryTime: DateTime     |
| - exitTime: DateTime      |
| - isPaid: bool            |
+---------------------------+
| + calculateDuration(): int|
+---------------------------+
```

**Tips for Class Diagrams:**
- Show **access modifiers** (+ public, - private, # protected)
- Mark **abstract classes** and **interfaces**
- Show **multiplicity** (1, 0..1, 1..*, 0..*)
- Keep it focused — don't draw every getter/setter
- Use **stereotypes** (`<<Singleton>>`, `<<abstract>>`, `<<interface>>`)

---

### Step 6: Write Core Code (10-15 minutes)

Write the most important classes and methods. Focus on **core logic**, not boilerplate.

```cpp
// Enums
enum class VehicleType { CAR, BIKE, TRUCK };
enum class SpotType { SMALL, MEDIUM, LARGE };

// Vehicle hierarchy
class Vehicle {
protected:
    string licensePlate;
    VehicleType type;
public:
    Vehicle(string plate, VehicleType t) : licensePlate(plate), type(t) {}
    VehicleType getType() const { return type; }
    string getLicensePlate() const { return licensePlate; }
    virtual ~Vehicle() = default;
};

class Car : public Vehicle {
public:
    Car(string plate) : Vehicle(plate, VehicleType::CAR) {}
};

// Parking Spot
class ParkingSpot {
    string spotId;
    SpotType spotType;
    Vehicle* vehicle = nullptr;
    
public:
    ParkingSpot(string id, SpotType type) : spotId(id), spotType(type) {}
    
    bool isAvailable() const { return vehicle == nullptr; }
    
    bool canFit(const Vehicle& v) const {
        if (!isAvailable()) return false;
        switch (v.getType()) {
            case VehicleType::BIKE:  return true; // fits in any spot
            case VehicleType::CAR:   return spotType != SpotType::SMALL;
            case VehicleType::TRUCK: return spotType == SpotType::LARGE;
        }
        return false;
    }
    
    bool park(Vehicle* v) {
        if (!canFit(*v)) return false;
        vehicle = v;
        return true;
    }
    
    Vehicle* unpark() {
        Vehicle* v = vehicle;
        vehicle = nullptr;
        return v;
    }
};

// Pricing Strategy (Strategy Pattern)
class PricingStrategy {
public:
    virtual double calculatePrice(int durationMinutes, VehicleType type) = 0;
    virtual ~PricingStrategy() = default;
};

class HourlyPricing : public PricingStrategy {
    unordered_map<VehicleType, double> rates = {
        {VehicleType::BIKE, 1.0},
        {VehicleType::CAR, 2.0},
        {VehicleType::TRUCK, 4.0}
    };
public:
    double calculatePrice(int durationMinutes, VehicleType type) override {
        int hours = (durationMinutes + 59) / 60; // round up
        return hours * rates[type];
    }
};

// Parking Lot (Singleton)
class ParkingLot {
    static ParkingLot* instance;
    vector<Floor> floors;
    unique_ptr<PricingStrategy> pricingStrategy;
    mutex mtx; // thread safety
    
    ParkingLot() : pricingStrategy(make_unique<HourlyPricing>()) {}
    
public:
    static ParkingLot* getInstance() {
        if (!instance) instance = new ParkingLot();
        return instance;
    }
    
    Ticket* parkVehicle(Vehicle* vehicle) {
        lock_guard<mutex> lock(mtx);
        for (auto& floor : floors) {
            ParkingSpot* spot = floor.findAvailableSpot(*vehicle);
            if (spot) {
                spot->park(vehicle);
                return new Ticket(vehicle, spot);
            }
        }
        return nullptr; // no spot available
    }
    
    double unparkVehicle(Ticket* ticket) {
        lock_guard<mutex> lock(mtx);
        ticket->setExitTime(chrono::system_clock::now());
        int duration = ticket->calculateDurationMinutes();
        double price = pricingStrategy->calculatePrice(
            duration, ticket->getVehicle()->getType()
        );
        ticket->getSpot()->unpark();
        return price;
    }
};
```

**Code Tips:**
- Write **interfaces first** (Strategy, Observer) — shows design thinking
- Use **SOLID principles** — especially SRP and OCP
- Handle **edge cases** (no spot available, invalid vehicle type)
- Mention **thread safety** if concurrency is relevant (mutex, lock_guard)
- Don't write getters/setters — focus on business logic

---

### Step 7: Discuss Trade-offs (2-3 minutes)

End by discussing what you'd improve or what trade-offs you made:

| Aspect | Discussion |
|--------|-----------|
| **Extensibility** | "Adding a new vehicle type (e.g., Electric) only requires a new subclass — no changes to existing code (OCP)." |
| **Testability** | "PricingStrategy is injected — we can mock it in tests. ParkingSpot.canFit() is a pure function — easy to unit test." |
| **Concurrency** | "I used a mutex in ParkingLot for thread safety. For higher throughput, we could use per-floor locks instead of a global lock." |
| **Performance** | "Finding an available spot is O(n) — we could use a min-heap or free list for O(1) allocation." |
| **Persistence** | "Currently in-memory. For persistence, we'd add a Repository layer (Repository pattern) with a database backend." |

---


## 36.2 Common Mistakes in LLD Interviews

> Knowing what NOT to do is as important as knowing what to do. These are the mistakes that cost candidates the most points.

---

### Mistake 1: Jumping to Code Without Clarifying Requirements

```
❌ Bad:
  Interviewer: "Design a parking lot system."
  Candidate: *immediately starts writing classes*
  
  Problem: You might design for the wrong requirements.
  Maybe the interviewer wanted a multi-floor lot with payment,
  but you designed a simple single-floor lot.

✅ Good:
  Interviewer: "Design a parking lot system."
  Candidate: "Before I start, let me clarify a few things.
    How many floors? What vehicle types? Do we need payment?
    Multiple entry/exit gates? Should it be thread-safe?"
  
  Then: "Based on your answers, here's the scope I'll focus on..."
```

**Why it matters:** Requirements clarification shows maturity and prevents wasted time designing the wrong thing.

---

### Mistake 2: Not Using Design Patterns Where Appropriate

```
❌ Bad:
  // Giant if-else chain for pricing
  double calculatePrice(VehicleType type, int hours) {
      if (type == CAR && isWeekend()) return hours * 3.0;
      else if (type == CAR && !isWeekend()) return hours * 2.0;
      else if (type == BIKE && isWeekend()) return hours * 1.5;
      // ... 20 more conditions
  }

✅ Good:
  // Strategy pattern — each pricing strategy is a separate class
  class WeekendPricing : public PricingStrategy { ... };
  class WeekdayPricing : public PricingStrategy { ... };
  
  // Swap at runtime
  parkingLot.setPricingStrategy(isWeekend() ? new WeekendPricing() : new WeekdayPricing());
```

**Why it matters:** Design patterns demonstrate that you can write maintainable, extensible code — not just code that works.

---

### Mistake 3: Over-Engineering (Too Many Patterns)

```
❌ Bad:
  "I'll use Singleton for the ParkingLot, Factory for creating spots,
   Abstract Factory for creating vehicle families, Builder for constructing
   tickets, Decorator for adding features to spots, Proxy for lazy-loading
   floors, Mediator for communication between gates..."
  
  → Too many patterns! The design is now harder to understand than the problem.

✅ Good:
  "I'll use Singleton for ParkingLot (only one instance needed) and
   Strategy for pricing (multiple algorithms, swappable at runtime).
   These two patterns solve the key design challenges."
  
  → Use patterns that solve REAL problems, not patterns for the sake of patterns.
```

**Rule of Thumb:** 2-3 patterns per LLD problem is typical. If you're using more than 4, you're probably over-engineering.

---

### Mistake 4: Ignoring Concurrency

```
❌ Bad:
  ParkingSpot* findAndPark(Vehicle* v) {
      ParkingSpot* spot = findAvailableSpot(v);  // Thread 1 finds spot A
      spot->park(v);                              // Thread 2 also found spot A!
      return spot;                                // Both threads park in the same spot!
  }

✅ Good:
  ParkingSpot* findAndPark(Vehicle* v) {
      lock_guard<mutex> lock(mtx);  // Only one thread at a time
      ParkingSpot* spot = findAvailableSpot(v);
      if (spot) spot->park(v);
      return spot;
  }
```

**When to mention concurrency:**
- Parking lot (multiple cars entering simultaneously)
- Elevator system (multiple requests at the same time)
- Booking systems (multiple users booking the same resource)
- Any system with shared mutable state

**What to say:** "This operation needs to be thread-safe because multiple threads could try to park simultaneously. I'll use a mutex to ensure atomicity. For better performance, we could use per-floor locks instead of a global lock."

---

### Mistake 5: Not Considering Edge Cases

```
Edge cases to always consider:
  - What if the input is null/empty?
  - What if the system is full? (no parking spots available)
  - What if the same vehicle tries to park twice?
  - What if payment fails?
  - What if the system crashes mid-operation?
  - What about overflow? (counter exceeds max value)
  - What about invalid input? (negative price, future date in the past)
```

**How to handle:** You don't need to implement every edge case, but **mention** them. "I'd add validation here to handle the case where..." shows the interviewer you think about robustness.

---

### Mistake 6: Tight Coupling Between Classes

```
❌ Bad (tight coupling):
  class ParkingLot {
      HourlyPricing pricing;  // directly depends on concrete class
      
      double calculatePrice(Ticket t) {
          return pricing.calculate(t);  // can't change pricing without modifying ParkingLot
      }
  };

✅ Good (loose coupling via interface):
  class ParkingLot {
      unique_ptr<PricingStrategy> pricing;  // depends on interface
      
      ParkingLot(unique_ptr<PricingStrategy> strategy) 
          : pricing(move(strategy)) {}  // injected via constructor
      
      double calculatePrice(Ticket t) {
          return pricing->calculate(t);  // works with ANY pricing strategy
      }
  };
```

**Principles to follow:**
- **Depend on abstractions, not concretions** (Dependency Inversion Principle)
- **Program to an interface, not an implementation**
- **Inject dependencies** (constructor injection) instead of creating them internally
- **Single Responsibility** — each class does one thing

---

### Mistake 7: Not Explaining Your Thought Process

```
❌ Bad:
  *silently writes code for 20 minutes*
  Interviewer has no idea what you're thinking or why you made certain decisions.

✅ Good:
  "I'm going to start with the Vehicle hierarchy. I'll make Vehicle abstract
   because we never instantiate a generic Vehicle — it's always a Car, Bike,
   or Truck. I'll use inheritance here because they share common attributes
   (license plate, type) but may have different behaviors (size, parking rules)."
  
  "For pricing, I'll use the Strategy pattern because we have multiple pricing
   algorithms and we want to swap them without modifying the ParkingLot class.
   This follows the Open/Closed Principle."
```

**Why it matters:** The interviewer is evaluating your **thought process**, not just the final code. Thinking out loud lets them see your reasoning, give hints, and steer you if needed.

---

### LLD Interview Checklist

Use this checklist to make sure you've covered everything:

```
Before coding:
  □ Clarified requirements (functional + non-functional + scope)
  □ Identified core entities (nouns → classes)
  □ Defined relationships (inheritance, composition, association)
  □ Identified applicable design patterns (2-3 max)

During coding:
  □ Used interfaces/abstract classes for extensibility
  □ Applied SOLID principles (especially SRP, OCP, DIP)
  □ Handled edge cases (or mentioned them)
  □ Considered thread safety (if applicable)
  □ Used enums for fixed sets of values
  □ Named classes and methods clearly

After coding:
  □ Discussed trade-offs (extensibility, testability, performance)
  □ Mentioned what you'd improve with more time
  □ Explained your design pattern choices and WHY
```

---

### Time Management (45-minute LLD Interview)

| Phase | Time | Activity |
|-------|------|----------|
| **Requirements** | 2-3 min | Clarify scope, ask questions |
| **Entities & Relationships** | 3-5 min | List classes, define relationships |
| **Design Patterns** | 3-5 min | Identify patterns, explain why |
| **Class Diagram** | 5-10 min | Draw classes, attributes, methods, relationships |
| **Core Code** | 15-20 min | Write key classes and methods |
| **Trade-offs** | 2-3 min | Discuss extensibility, testability, performance |
| **Buffer** | 2-5 min | Questions, edge cases, improvements |

**Key Insight:** The interviewer cares more about your **design decisions** (steps 1-4) than your **code** (step 6). A great design with pseudocode beats a poor design with perfect syntax.

---

## Summary & Key Takeaways

| Step | Key Point |
|------|-----------|
| **1. Requirements** | Always clarify before designing; ask about scope, scale, constraints |
| **2. Entities** | Nouns → classes, verbs → methods, adjectives → attributes/enums |
| **3. Relationships** | Inheritance (is-a), composition (has-a, strong), aggregation (has-a, weak), association (uses) |
| **4. Patterns** | Use 2-3 patterns that solve real problems; don't force patterns |
| **5. Class Diagram** | Show classes, attributes, methods, relationships, multiplicity |
| **6. Code** | Focus on core logic, not boilerplate; use interfaces; handle edge cases |
| **7. Trade-offs** | Discuss extensibility, testability, concurrency, performance |

| Mistake | Fix |
|---------|-----|
| Jumping to code | Spend 2-3 minutes on requirements first |
| No design patterns | Use Strategy, Observer, Factory, Singleton where they fit |
| Over-engineering | 2-3 patterns max; patterns should solve real problems |
| Ignoring concurrency | Mention thread safety for shared mutable state |
| Missing edge cases | Mention null inputs, full capacity, duplicate operations |
| Tight coupling | Depend on interfaces; inject dependencies |
| Silent coding | Think out loud; explain every decision |

---

## Interview Tips for Module 36

1. **Requirements first** — always spend 2-3 minutes clarifying before designing
2. **Think out loud** — explain your reasoning at every step; the process matters more than the result
3. **Start with interfaces** — define the contracts before the implementations
4. **Use SOLID** — especially SRP (one class, one job) and OCP (extend without modifying)
5. **2-3 patterns** — identify the right patterns; explain WHY they fit; don't over-engineer
6. **Draw before coding** — a class diagram helps you and the interviewer see the big picture
7. **Handle edge cases** — mention them even if you don't implement them all
8. **Mention concurrency** — if the system has shared state, discuss thread safety
9. **Discuss trade-offs** — extensibility vs simplicity, performance vs readability
10. **Practice** — do 10-15 LLD problems (Modules 12-14) until the framework is second nature
