# Module 36: LLD Interview Framework

> Low-Level Design (LLD) interviews test your ability to design the internal structure of a system — classes, modules, relationships, and design patterns. Unlike HLD (which focuses on distributed architecture), LLD focuses on **object-oriented design** and **clean code**. This module provides a step-by-step framework for approaching any LLD interview problem, with examples and common mistakes to avoid.

> **Ruby Context:** Ruby's dynamic nature, duck typing, mixins (modules), and expressive syntax make LLD in Ruby different from statically-typed languages. Key Ruby idioms for LLD: **modules for mixins** (instead of multiple inheritance), **duck typing** (respond_to? over type checking), **blocks/procs** for strategy-like patterns, **attr_accessor/reader/writer** for encapsulation, **Comparable/Enumerable** for standard interfaces, and **SOLID principles** expressed through Ruby conventions. Ruby favors composition over inheritance and uses dependency injection via constructor arguments or configuration blocks.

---

## 36.1 Step-by-Step Approach

> Follow this framework for every LLD interview. It keeps you organized, ensures you cover all aspects, and demonstrates a structured thought process to the interviewer.

---

### Step 1: Clarify Requirements (2-3 minutes)

**Never start designing without understanding what you're building.**

| Category | Example Questions |
|----------|------------------|
| **Functional** | "What are the core features?" "Should users be able to X?" |
| **Non-Functional** | "How many concurrent users?" "Does it need to be thread-safe?" |
| **Scope** | "Should I design the full system or focus on the core?" |
| **Constraints** | "Any specific language/framework?" "Memory constraints?" |

**Example — Parking Lot System:**

```
Questions to ask:
  - How many floors? How many spots per floor?
  - What vehicle types? (Car, Bike, Truck)
  - What spot types? (Small, Medium, Large)
  - Do we need payment? Hourly rate?
  - Multiple entry/exit gates?
  - Is concurrency a concern? (multiple cars entering simultaneously)

Scope decision:
  ✅ In scope: Vehicle parking, spot allocation, payment calculation
  ❌ Out of scope: UI, database persistence, payment gateway integration
```

---

### Step 2: Identify Core Objects/Entities (3-5 minutes)

```
Nouns → Classes
  "A parking lot has multiple floors. Each floor has parking spots.
   Vehicles enter through gates and park in spots. A ticket is issued."
  
  Classes: ParkingLot, Floor, ParkingSpot, Vehicle, Gate, Ticket

Verbs → Methods
  "Vehicles enter, park, exit. Tickets are issued and paid."
  
  Methods: enter, park, exit, issue_ticket, pay

Adjectives → Attributes or Constants/Symbols
  "Small, medium, large spots. Car, bike, truck vehicles."
  
  Constants: SPOT_TYPES = [:small, :medium, :large]
             VEHICLE_TYPES = [:car, :bike, :truck]
```

> **Ruby Context:** In Ruby, prefer symbols (`:small`, `:car`) or string enums over separate classes for simple type distinctions. Use classes only when types have genuinely different behavior.

---

### Step 3: Define Relationships (3-5 minutes)

| Relationship | Ruby Implementation | Example |
|-------------|-------------------|---------|
| **Inheritance (is-a)** | `class Car < Vehicle` | Car is-a Vehicle |
| **Composition (has-a, strong)** | Instance created inside owner | ParkingLot creates its Floors |
| **Aggregation (has-a, weak)** | Passed in, exists independently | Floor holds ParkingSpots |
| **Mixin (behaves-like)** | `include Payable` | Ticket includes Payable behavior |
| **Association (uses)** | Reference via attribute | Ticket references a Vehicle |
| **Dependency (uses temporarily)** | Method parameter | `calculate_price(ticket)` |

> **Ruby Context:** Ruby uses **modules (mixins)** where other languages use interfaces or multiple inheritance. A module defines a contract (set of methods) that any class can include.

```ruby
# Ruby relationships example
module Parkable
  def can_fit?(spot)
    raise NotImplementedError
  end
end

class Vehicle
  include Parkable
  attr_reader :license_plate, :vehicle_type
end

class Car < Vehicle  # Inheritance
  def can_fit?(spot)
    spot.size != :small
  end
end

class ParkingLot
  attr_reader :floors  # Composition — lot owns floors

  def initialize(num_floors:, spots_per_floor:)
    @floors = Array.new(num_floors) { |i| Floor.new(number: i + 1, spots: spots_per_floor) }
  end
end
```

---

### Step 4: Apply Design Patterns (5-10 minutes)

**Common Patterns in LLD Interviews (Ruby idioms):**

| Pattern | When to Use | Ruby Idiom |
|---------|-------------|-----------|
| **Singleton** | Only one instance | `include Singleton` (stdlib) |
| **Strategy** | Multiple algorithms, swap at runtime | Pass a proc/lambda, or inject a strategy object |
| **Observer** | Notify on state changes | `Observable` module (stdlib), or custom callbacks |
| **Factory** | Create objects without specifying class | Class method (`.create`) or separate factory class |
| **State** | Behavior changes based on state | State objects or `aasm` gem |
| **Template Method** | Algorithm skeleton with customizable steps | Base class with methods that subclasses override |
| **Decorator** | Add behavior dynamically | `SimpleDelegator`, `module_eval`, or wrapper classes |
| **Command** | Encapsulate actions, support undo | Command objects with `execute` and `undo` methods |

> **Ruby Context:** Ruby's blocks, procs, and lambdas often replace the need for formal Strategy or Command pattern classes. Duck typing reduces the need for explicit interfaces.

```ruby
# Strategy pattern — Ruby style (multiple approaches)

# Approach 1: Inject a strategy object (classic OOP)
class ParkingLot
  def initialize(pricing_strategy:)
    @pricing_strategy = pricing_strategy
  end

  def calculate_price(ticket)
    @pricing_strategy.calculate(ticket)
  end
end

class HourlyPricing
  RATES = { car: 2.0, bike: 1.0, truck: 4.0 }

  def calculate(ticket)
    hours = ((ticket.exit_time - ticket.entry_time) / 3600.0).ceil
    hours * RATES.fetch(ticket.vehicle.vehicle_type)
  end
end

lot = ParkingLot.new(pricing_strategy: HourlyPricing.new)

# Approach 2: Pass a block/lambda (Ruby-idiomatic for simple strategies)
class ParkingLot
  def initialize(&pricing_block)
    @pricing = pricing_block
  end

  def calculate_price(ticket)
    @pricing.call(ticket)
  end
end

lot = ParkingLot.new { |ticket| ticket.duration_hours.ceil * 2.0 }

# Approach 3: Configuration block (Rails-style)
class ParkingLot
  class << self
    attr_accessor :pricing_strategy
  end

  def calculate_price(ticket)
    self.class.pricing_strategy.calculate(ticket)
  end
end

ParkingLot.pricing_strategy = HourlyPricing.new
```

**How to Present Patterns:**
- Name the pattern
- Explain WHY it fits (what problem it solves)
- Show the Ruby-idiomatic implementation

---

### Step 5: Draw Class Diagram (5-10 minutes)

```
+---------------------------+
| <<Singleton>>             |
| ParkingLot                |
+---------------------------+
| - floors: Array<Floor>    |
| - pricing_strategy        |
+---------------------------+
| + park_vehicle(vehicle)   |
| + unpark_vehicle(ticket)  |
| + available_spots         |
+---------------------------+
         ◆ 1..*
         |
+---------------------------+
| Floor                     |
+---------------------------+
| - number: Integer         |
| - spots: Array<ParkingSpot>|
+---------------------------+
| + find_available(vehicle) |
| + park(vehicle)           |
+---------------------------+
         ◆ 1..*
         |
+---------------------------+       +---------------------------+
| ParkingSpot               |       | <<module>>                |
+---------------------------+       | PricingStrategy           |
| - spot_id: String         |       +---------------------------+
| - size: Symbol            |       | + calculate(ticket)       |
| - vehicle: Vehicle/nil    |       +---------------------------+
+---------------------------+            ▲           ▲
| + available?              |            |           |
| + park(vehicle)           |   +--------+--+ +-----+--------+
| + unpark                  |   |HourlyPricing| |FlatPricing  |
| + can_fit?(vehicle)       |   +-------------+ +-------------+
+---------------------------+

+---------------------------+
| Vehicle                   |
+---------------------------+
| - license_plate: String   |
| - vehicle_type: Symbol    |
+---------------------------+
| + can_fit?(spot)          |
+---------------------------+
     ▲        ▲        ▲
     |        |        |
  +-----+  +-----+  +-------+
  | Car |  | Bike|  | Truck |
  +-----+  +-----+  +-------+

+---------------------------+
| Ticket                    |
+---------------------------+
| - ticket_id: String       |
| - vehicle: Vehicle        |
| - spot: ParkingSpot       |
| - entry_time: Time        |
| - exit_time: Time/nil     |
+---------------------------+
| + duration_hours          |
| + paid?                   |
+---------------------------+
```

---

### Step 6: Write Core Code (10-15 minutes)

> **Ruby Context:** Focus on clean, idiomatic Ruby. Use `attr_reader`/`attr_accessor` for encapsulation, `raise` for error handling, `freeze` for constants, and meaningful method names.

```ruby
# Parking Lot LLD — Complete Ruby Implementation

# Vehicle hierarchy
class Vehicle
  attr_reader :license_plate, :vehicle_type

  def initialize(license_plate:, vehicle_type:)
    @license_plate = license_plate
    @vehicle_type = vehicle_type
  end
end

class Car < Vehicle
  def initialize(license_plate:)
    super(license_plate: license_plate, vehicle_type: :car)
  end
end

class Bike < Vehicle
  def initialize(license_plate:)
    super(license_plate: license_plate, vehicle_type: :bike)
  end
end

class Truck < Vehicle
  def initialize(license_plate:)
    super(license_plate: license_plate, vehicle_type: :truck)
  end
end

# Parking Spot
class ParkingSpot
  attr_reader :spot_id, :size, :vehicle

  def initialize(spot_id:, size:)
    @spot_id = spot_id
    @size = size  # :small, :medium, :large
    @vehicle = nil
  end

  def available?
    @vehicle.nil?
  end

  def can_fit?(vehicle)
    return false unless available?

    case vehicle.vehicle_type
    when :bike  then true                    # fits anywhere
    when :car   then size != :small          # medium or large
    when :truck then size == :large          # large only
    else false
    end
  end

  def park(vehicle)
    raise 'Spot is occupied' unless can_fit?(vehicle)
    @vehicle = vehicle
    true
  end

  def unpark
    raise 'Spot is empty' if available?
    v = @vehicle
    @vehicle = nil
    v
  end
end

# Ticket
class Ticket
  attr_reader :ticket_id, :vehicle, :spot, :entry_time
  attr_accessor :exit_time

  def initialize(vehicle:, spot:)
    @ticket_id = SecureRandom.uuid
    @vehicle = vehicle
    @spot = spot
    @entry_time = Time.now
    @exit_time = nil
  end

  def duration_hours
    end_time = exit_time || Time.now
    ((end_time - entry_time) / 3600.0).ceil
  end

  def paid?
    !@exit_time.nil?
  end
end

# Pricing Strategy (Strategy Pattern)
class HourlyPricing
  RATES = { car: 2.0, bike: 1.0, truck: 4.0 }.freeze

  def calculate(ticket)
    ticket.duration_hours * RATES.fetch(ticket.vehicle.vehicle_type)
  end
end

class FlatRatePricing
  RATES = { car: 10.0, bike: 5.0, truck: 20.0 }.freeze

  def calculate(ticket)
    RATES.fetch(ticket.vehicle.vehicle_type)
  end
end

# Floor
class Floor
  attr_reader :number, :spots

  def initialize(number:, spot_config:)
    @number = number
    @spots = build_spots(spot_config)
  end

  def find_available_spot(vehicle)
    @spots.find { |spot| spot.can_fit?(vehicle) }
  end

  def available_spots_count
    @spots.count(&:available?)
  end

  private

  def build_spots(config)
    spots = []
    config.each do |size, count|
      count.times { |i| spots << ParkingSpot.new(spot_id: "#{number}-#{size}-#{i + 1}", size: size) }
    end
    spots
  end
end

# Parking Lot (Singleton)
require 'singleton'

class ParkingLot
  include Singleton

  attr_reader :floors
  attr_accessor :pricing_strategy

  def initialize
    @floors = []
    @pricing_strategy = HourlyPricing.new
    @tickets = {}  # ticket_id → ticket
    @mutex = Mutex.new
  end

  def add_floor(spot_config:)
    @floors << Floor.new(number: @floors.size + 1, spot_config: spot_config)
  end

  def park_vehicle(vehicle)
    @mutex.synchronize do
      @floors.each do |floor|
        spot = floor.find_available_spot(vehicle)
        if spot
          spot.park(vehicle)
          ticket = Ticket.new(vehicle: vehicle, spot: spot)
          @tickets[ticket.ticket_id] = ticket
          return ticket
        end
      end
      raise 'No available spot for this vehicle'
    end
  end

  def unpark_vehicle(ticket_id)
    @mutex.synchronize do
      ticket = @tickets.fetch(ticket_id) { raise 'Invalid ticket' }
      ticket.exit_time = Time.now
      price = @pricing_strategy.calculate(ticket)
      ticket.spot.unpark
      @tickets.delete(ticket_id)
      { ticket: ticket, price: price }
    end
  end

  def available_spots
    @floors.sum(&:available_spots_count)
  end
end

# Usage
lot = ParkingLot.instance
lot.add_floor(spot_config: { small: 10, medium: 20, large: 5 })
lot.add_floor(spot_config: { small: 10, medium: 20, large: 5 })

car = Car.new(license_plate: 'ABC-123')
ticket = lot.park_vehicle(car)
puts "Parked at spot: #{ticket.spot.spot_id}"

# Later...
result = lot.unpark_vehicle(ticket.ticket_id)
puts "Price: $#{result[:price]}"
```

**Code Tips (Ruby-specific):**
- Use `attr_reader` for read-only, `attr_accessor` only when mutation is needed
- Use `freeze` on constants (`RATES = { ... }.freeze`)
- Use `raise` with descriptive messages for error cases
- Use `Mutex` for thread safety (mention it, implement if asked)
- Use `SecureRandom.uuid` for unique IDs
- Prefer composition: pass strategy objects via constructor
- Use symbols (`:car`, `:small`) instead of string constants

---

### Step 7: Discuss Trade-offs (2-3 minutes)

| Aspect | Discussion |
|--------|-----------|
| **Extensibility** | "Adding Electric vehicle: new subclass, override `can_fit?` — no changes to ParkingLot (OCP)." |
| **Testability** | "PricingStrategy is injected — mock it in tests. `can_fit?` is a pure method — easy to unit test with RSpec." |
| **Concurrency** | "Used Mutex in ParkingLot. For higher throughput, per-floor Mutex instead of global." |
| **Performance** | "Finding a spot is O(n). Could use a free-list or priority queue for O(1) allocation." |
| **Persistence** | "Currently in-memory. Add a Repository pattern with ActiveRecord for database backing." |

---


## 36.2 Common Mistakes in LLD Interviews

---

### Mistake 1: Jumping to Code Without Clarifying Requirements

```
❌ Bad: Immediately starts writing classes
✅ Good: "Before I start, let me clarify — how many floors? Vehicle types? Payment needed?"
```

---

### Mistake 2: Not Using Design Patterns Where Appropriate

```ruby
# ❌ Bad — giant conditional for pricing
def calculate_price(vehicle_type, hours)
  if vehicle_type == :car && weekend?
    hours * 3.0
  elsif vehicle_type == :car
    hours * 2.0
  elsif vehicle_type == :bike && weekend?
    hours * 1.5
  # ... 20 more conditions
  end
end

# ✅ Good — Strategy pattern
class WeekendPricing
  def calculate(ticket)
    ticket.duration_hours * rates[ticket.vehicle.vehicle_type]
  end
end

lot.pricing_strategy = weekend? ? WeekendPricing.new : WeekdayPricing.new
```

---

### Mistake 3: Over-Engineering (Too Many Patterns)

**Rule of Thumb:** 2-3 patterns per LLD problem is typical. If you're using more than 4, you're probably over-engineering.

---

### Mistake 4: Ignoring Concurrency

```ruby
# ❌ Bad — race condition
def park_vehicle(vehicle)
  spot = find_available_spot(vehicle)  # Thread 1 finds spot A
  spot.park(vehicle)                    # Thread 2 also found spot A!
end

# ✅ Good — thread-safe
def park_vehicle(vehicle)
  @mutex.synchronize do
    spot = find_available_spot(vehicle)
    raise 'No spot available' unless spot
    spot.park(vehicle)
  end
end
```

**When to mention concurrency:** Parking lots, elevators, booking systems, any shared mutable state.

---

### Mistake 5: Not Considering Edge Cases

```
Edge cases to always consider:
  - What if input is nil? (use guard clauses: `return unless vehicle`)
  - What if the system is full? (raise descriptive error)
  - What if the same vehicle tries to park twice? (check if already parked)
  - What if payment fails? (keep ticket active, allow retry)
  - What about invalid input? (validate in initializer)
```

---

### Mistake 6: Tight Coupling Between Classes

```ruby
# ❌ Bad — depends on concrete class
class ParkingLot
  def initialize
    @pricing = HourlyPricing.new  # Can't change without modifying ParkingLot
  end
end

# ✅ Good — depends on duck type (any object with #calculate)
class ParkingLot
  def initialize(pricing_strategy: HourlyPricing.new)
    @pricing_strategy = pricing_strategy  # Injected, swappable, testable
  end
end
```

> **Ruby Context:** Ruby's duck typing means you don't need explicit interfaces. Any object that responds to `#calculate` works as a pricing strategy. In tests, you can pass a simple lambda: `ParkingLot.new(pricing_strategy: ->(ticket) { 0.0 })`.

---

### Mistake 7: Not Explaining Your Thought Process

```
✅ Good:
  "I'll make Vehicle a base class because Car, Bike, and Truck share
   license_plate and vehicle_type but have different parking rules via can_fit?.
   
   For pricing, I'll use dependency injection — pass the strategy into ParkingLot's
   constructor. This way we can swap pricing at runtime and mock it in tests.
   In Ruby, any object responding to #calculate works — no interface needed."
```

---

### Ruby-Specific LLD Tips

| Tip | Why |
|-----|-----|
| **Use modules for shared behavior** | `include Comparable`, `include Enumerable` — Ruby's version of interfaces |
| **Prefer composition over inheritance** | Inject collaborators; use modules for cross-cutting concerns |
| **Duck typing over type checking** | Don't check `is_a?(PricingStrategy)` — just call `#calculate` |
| **Use `freeze` for immutable constants** | `RATES = { car: 2.0 }.freeze` prevents accidental mutation |
| **Guard clauses for edge cases** | `return unless vehicle` instead of nested if/else |
| **Raise descriptive errors** | `raise ArgumentError, 'Vehicle cannot be nil'` |
| **Use `Struct` for simple value objects** | `Ticket = Struct.new(:id, :vehicle, :spot, :entry_time, keyword_init: true)` |
| **Use `attr_reader` by default** | Only use `attr_accessor` when external mutation is intended |
| **Name methods with `?` for booleans** | `available?`, `paid?`, `can_fit?` |
| **Name methods with `!` for dangerous ops** | `park!` (raises on failure), `park` (returns nil on failure) |

---

### LLD Interview Checklist

```
Before coding:
  □ Clarified requirements (functional + non-functional + scope)
  □ Identified core entities (nouns → classes)
  □ Defined relationships (inheritance, composition, modules)
  □ Identified applicable design patterns (2-3 max)

During coding:
  □ Used modules/duck typing for extensibility (not rigid interfaces)
  □ Applied SOLID principles (SRP, OCP, DIP)
  □ Handled edge cases (guard clauses, raise on invalid state)
  □ Considered thread safety (Mutex if applicable)
  □ Used symbols for type constants (:car, :small)
  □ Named methods clearly (? for booleans, ! for dangerous)

After coding:
  □ Discussed trade-offs (extensibility, testability, performance)
  □ Mentioned what you'd improve with more time
  □ Explained design pattern choices and WHY
```

---

### Time Management (45-minute LLD Interview)

| Phase | Time | Activity |
|-------|------|----------|
| **Requirements** | 2-3 min | Clarify scope, ask questions |
| **Entities & Relationships** | 3-5 min | List classes, define relationships |
| **Design Patterns** | 3-5 min | Identify patterns, explain why |
| **Class Diagram** | 5-10 min | Draw classes, attributes, methods, relationships |
| **Core Code** | 15-20 min | Write key classes and methods (Ruby) |
| **Trade-offs** | 2-3 min | Discuss extensibility, testability, performance |
| **Buffer** | 2-5 min | Questions, edge cases, improvements |

**Key Insight:** The interviewer cares more about your **design decisions** (steps 1-4) than your **code** (step 6). A great design with pseudocode beats a poor design with perfect syntax.

---

## Summary & Key Takeaways

| Step | Key Point |
|------|-----------|
| **1. Requirements** | Always clarify before designing; ask about scope, scale, constraints |
| **2. Entities** | Nouns → classes, verbs → methods, adjectives → symbols/constants |
| **3. Relationships** | Inheritance, composition, modules (mixins), association |
| **4. Patterns** | 2-3 patterns that solve real problems; Ruby idioms (blocks, DI, duck typing) |
| **5. Class Diagram** | Show classes, attributes, methods, relationships |
| **6. Code** | Idiomatic Ruby: attr_reader, guard clauses, raise, freeze, Mutex |
| **7. Trade-offs** | Extensibility, testability, concurrency, performance |

| Mistake | Ruby Fix |
|---------|----------|
| Jumping to code | Spend 2-3 minutes on requirements first |
| No design patterns | Use Strategy (inject object or lambda), Observer, Factory |
| Over-engineering | 2-3 patterns max; Ruby's simplicity reduces need for heavy patterns |
| Ignoring concurrency | `Mutex.new` + `synchronize` for shared mutable state |
| Missing edge cases | Guard clauses: `return unless`, `raise ArgumentError` |
| Tight coupling | Dependency injection via constructor; duck typing |
| Silent coding | Think out loud; explain every decision |

---

## Interview Tips for Module 36

1. **Requirements first** — always spend 2-3 minutes clarifying before designing
2. **Think out loud** — explain your reasoning; the process matters more than the result
3. **Ruby idioms** — use `attr_reader`, symbols, `freeze`, `?`/`!` methods, guard clauses
4. **Modules over interfaces** — Ruby uses duck typing; `include` modules for shared behavior
5. **Dependency injection** — pass collaborators via constructor; enables testing and swapping
6. **2-3 patterns** — Strategy (lambda or object), Singleton (`include Singleton`), Observer
7. **Draw before coding** — class diagram helps you and the interviewer see the big picture
8. **Handle edge cases** — `raise` with descriptive messages; guard clauses for nil/invalid
9. **Mention concurrency** — `Mutex#synchronize` for shared state; mention per-resource locks for performance
10. **Discuss testability** — "I can mock the pricing strategy in RSpec by injecting a double"