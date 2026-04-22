# Module 8: Advanced OOP & Ruby Concepts

> This module covers advanced Ruby techniques that go beyond basic OOP — object references and immutability for safe data handling, duck typing for flexible polymorphism, metaprogramming for powerful abstractions, the eigenclass (singleton class) for per-object behavior, module composition for compile-time-like configurability, DSL building for expressive APIs, and type checking patterns to prevent bugs. These are the tools that separate production-quality Ruby from textbook Ruby.

---

## 8.1 Object References, Immutability & Object Lifecycle

> Ruby is a pass-by-reference-value language — variables hold references to objects, not the objects themselves. Understanding how references, copying, freezing, and garbage collection work is essential for writing correct, performant Ruby. This section is the Ruby equivalent of C++ move semantics and smart pointers — it covers how Ruby manages object ownership and lifecycle.

---

### Object References and object_id

Every object in Ruby has a unique `object_id`. Variables are labels (references) that point to objects — they are NOT the objects themselves.

```ruby
# Variables are references (labels) pointing to objects
a = "hello"
b = a          # b points to the SAME object as a

puts a.object_id        # e.g., 260
puts b.object_id        # same as a — 260
puts a.equal?(b)        # true — same object in memory

# Mutating through one reference affects the other
b << " world"
puts a                  # "hello world" — a sees the change!
puts b                  # "hello world"

# Assignment creates a NEW reference, not a new object
c = "hello"
d = "hello"
puts c.equal?(d)        # false — different objects (same content, different identity)
puts c == d             # true  — same VALUE (content equality)
puts c.eql?(d)          # true  — same value and type

# Reassignment points the variable to a NEW object
a = "goodbye"
puts a                  # "goodbye" — a now points to a new object
puts b                  # "hello world" — b still points to the old object
```

**Key distinction — three kinds of equality:**

| Method | Checks | Example |
|--------|--------|---------|
| `equal?` | Same object (identity) | `a.equal?(b)` — same `object_id`? |
| `==` | Same value (content) | `"hi" == "hi"` → `true` |
| `eql?` | Same value AND same type | `1.eql?(1.0)` → `false` (Integer vs Float) |

---

### Immediate Values vs Reference Types

Some Ruby objects are stored directly in the variable (immediate values) — they don't have separate heap allocations:

```ruby
# Immediate values — stored directly, not on the heap
# These are the SAME object everywhere (singleton-like)
puts 42.object_id          # always 85 (2n+1 for integers)
puts 42.object_id          # still 85 — same object
puts true.object_id        # 2
puts false.object_id       # 0
puts nil.object_id         # 8
puts :hello.object_id      # consistent across calls (symbol table)

x = 42
y = 42
puts x.equal?(y)           # true — same object! (immediate value)

# Reference types — each instance is a separate heap object
s1 = "hello"
s2 = "hello"
puts s1.equal?(s2)         # false — different objects on the heap

a1 = [1, 2, 3]
a2 = [1, 2, 3]
puts a1.equal?(a2)         # false — different array objects
```

**Immediate values in Ruby:**

| Type | Immediate? | Notes |
|------|-----------|-------|
| `Integer` (small) | Yes | Stored as tagged pointer (Fixnum internally) |
| `Symbol` | Yes | Interned in global symbol table |
| `true` / `false` | Yes | Singleton objects |
| `nil` | Yes | Singleton object |
| `Float` | Yes (CRuby 2.0+) | Stored as tagged pointer on 64-bit |
| `String` | No | Always heap-allocated (mutable) |
| `Array` / `Hash` | No | Always heap-allocated |
| Custom objects | No | Always heap-allocated |

**Why this matters:** Immediate values can't be mutated (they're frozen by nature), and you can't define singleton methods on them. Understanding this prevents subtle bugs.

---

### dup vs clone — Shallow Copying

Ruby provides two methods for copying objects. Both create **shallow copies** — the outer object is new, but nested objects are still shared.

```ruby
class Document
  attr_accessor :title, :tags, :metadata

  def initialize(title, tags, metadata)
    @title = title
    @tags = tags
    @metadata = metadata
  end

  def to_s
    "Document(#{@title}, tags=#{@tags}, meta=#{@metadata})"
  end
end

original = Document.new("Report", ["urgent", "finance"], { author: "Alice" })

# --- dup: copies object, does NOT copy frozen state or singleton methods ---
duped = original.dup

puts original.equal?(duped)                    # false — different objects
puts original.title.equal?(duped.title)        # true  — same String object (shallow!)
puts original.tags.equal?(duped.tags)          # true  — same Array object (shallow!)

# Mutating a nested object affects both
duped.tags << "reviewed"
puts original.tags    # ["urgent", "finance", "reviewed"] — AFFECTED!

# Reassigning a top-level attribute does NOT affect the original
duped.title = "New Report"
puts original.title   # "Report" — not affected (reassignment, not mutation)

# --- clone: copies object, ALSO copies frozen state and singleton methods ---
original.freeze

cloned = original.clone
puts cloned.frozen?   # true — clone preserves frozen state

duped2 = original.dup
puts duped2.frozen?   # false — dup does NOT preserve frozen state

# Singleton methods
obj = Object.new
def obj.special_method
  "I'm special"
end

cloned_obj = obj.clone
puts cloned_obj.special_method  # "I'm special" — clone copies singleton methods

duped_obj = obj.dup
# duped_obj.special_method      # NoMethodError — dup does NOT copy singleton methods
```

**dup vs clone summary:**

| Aspect | `dup` | `clone` |
|--------|-------|---------|
| Creates new object | Yes | Yes |
| Shallow copy | Yes | Yes |
| Copies frozen state | No | Yes |
| Copies singleton methods | No | Yes |
| Calls `initialize_copy` | Yes | Yes |
| Use case | "Give me a mutable copy" | "Give me an exact replica" |

---

### Deep Copy

Neither `dup` nor `clone` performs a deep copy. For true independence, you need to deep copy manually or use Marshal:

```ruby
# --- Marshal trick for deep copy (works for most objects) ---
original = { name: "Alice", scores: [95, 87, 92], address: { city: "NYC" } }

deep = Marshal.load(Marshal.dump(original))

deep[:scores] << 100
deep[:address][:city] = "LA"

puts original[:scores]         # [95, 87, 92] — NOT affected
puts original[:address][:city] # "NYC" — NOT affected
puts deep[:scores]             # [95, 87, 92, 100]
puts deep[:address][:city]     # "LA"

# --- Limitations of Marshal ---
# Cannot marshal: Procs, lambdas, IO objects, singleton methods, Bindings
lambda_hash = { action: -> { puts "hi" } }
# Marshal.dump(lambda_hash)  # TypeError: no _dump_data is defined for class Proc

# --- Custom deep copy ---
class Document
  attr_accessor :title, :tags, :metadata

  def initialize(title, tags, metadata)
    @title = title
    @tags = tags
    @metadata = metadata
  end

  def deep_dup
    Document.new(
      @title.dup,
      @tags.map(&:dup),
      @metadata.transform_values(&:dup)
    )
  end
end

doc1 = Document.new("Report", ["urgent"], { author: "Alice" })
doc2 = doc1.deep_dup

doc2.tags << "archived"
puts doc1.tags    # ["urgent"] — not affected
puts doc2.tags    # ["urgent", "archived"]
```

---

### freeze, frozen?, and Immutability

`freeze` makes an object immutable — no further modifications are allowed. This is Ruby's primary mechanism for creating safe, read-only data.

```ruby
# --- Basic freeze ---
str = "hello"
str.freeze

# str << " world"     # FrozenError: can't modify frozen String
# str.upcase!         # FrozenError
# str[0] = "H"        # FrozenError

puts str.frozen?       # true
puts str               # "hello" — still accessible, just not modifiable

# --- freeze is permanent — cannot unfreeze ---
# There is NO unfreeze method. Once frozen, always frozen.
# To get a mutable version, use dup:
mutable = str.dup
mutable << " world"    # OK — dup creates an unfrozen copy
puts mutable           # "hello world"
puts mutable.frozen?   # false

# --- Freezing collections ---
arr = [1, 2, 3]
arr.freeze

# arr << 4             # FrozenError
# arr[0] = 99          # FrozenError
# arr.push(4)          # FrozenError

# BUT: freeze is SHALLOW — nested objects are NOT frozen!
nested = [[1, 2], [3, 4]]
nested.freeze

# nested << [5, 6]     # FrozenError — can't modify outer array
nested[0] << 99         # OK! Inner array is NOT frozen!
puts nested.inspect     # [[1, 2, 99], [3, 4]]

# --- Deep freeze (manual) ---
def deep_freeze(obj)
  case obj
  when Array
    obj.each { |item| deep_freeze(item) }
  when Hash
    obj.each_value { |val| deep_freeze(val) }
  when String, Symbol
    # freeze the string
  end
  obj.freeze
end

config = {
  database: { host: "localhost", port: 5432 },
  features: ["auth", "logging"]
}
deep_freeze(config)

# config[:database][:host] = "remote"  # FrozenError — deeply frozen!
# config[:features] << "caching"       # FrozenError — deeply frozen!
```

---

### Frozen String Literals

Ruby 2.3+ introduced a magic comment that makes all string literals frozen by default — a significant performance optimization:

```ruby
# frozen_string_literal: true

# With this comment, ALL string literals in this file are frozen
str = "hello"
puts str.frozen?    # true

# str << " world"   # FrozenError!
# str.upcase!       # FrozenError!

# To create a mutable string, use String.new or unary +
mutable1 = String.new("hello")
mutable1 << " world"   # OK

mutable2 = +"hello"    # unary + creates an unfrozen copy
mutable2 << " world"   # OK

# To explicitly freeze (redundant with the magic comment, but explicit):
frozen = -"hello"      # unary - returns a frozen, deduplicated string

# Why use frozen_string_literal: true?
# 1. Performance: frozen strings can be shared/deduplicated (less GC pressure)
# 2. Safety: prevents accidental mutation of string literals
# 3. Future-proofing: Ruby 3.x may make this the default
```

---

### Garbage Collection and Object Lifecycle

Ruby uses automatic garbage collection (GC) — you don't manually allocate or free memory. This is the Ruby equivalent of C++ smart pointers, but fully automatic.

```ruby
# --- Object lifecycle ---
# 1. Allocation: Ruby allocates memory when you create objects
# 2. Usage: You use the object through references
# 3. Unreachable: When no references point to the object, it becomes garbage
# 4. Collection: The GC reclaims the memory (eventually)

def create_objects
  a = "temporary"     # object created
  b = [1, 2, 3]       # another object
  return b            # b is returned (still referenced), a becomes garbage
end

result = create_objects
# "temporary" string is now unreachable → eligible for GC
# The array [1, 2, 3] is still alive (referenced by result)

# --- Finalizers — cleanup callbacks (like C++ destructors, but weaker) ---
class ManagedResource
  def initialize(name)
    @name = name
    # Register a destructor-like callback
    # IMPORTANT: The proc must NOT reference self (would prevent GC!)
    weak_name = @name.dup
    ObjectSpace.define_finalizer(self, proc { |id|
      puts "Cleaning up resource: #{weak_name} (object_id: #{id})"
    })
  end
end

r = ManagedResource.new("database_connection")
r = nil  # remove reference — object becomes eligible for GC
GC.start # suggest garbage collection (finalizer may run)

# --- WeakRef — references that don't prevent GC ---
require 'weakref'

strong = "I am strong"
weak = WeakRef.new(strong)

puts weak.to_s          # "I am strong" — still alive
puts weak.weakref_alive? # true

strong = nil
GC.start

# weak.to_s             # WeakRef::RefError — object was garbage collected!
# weak.weakref_alive?   # false

# --- GC tuning ---
GC.disable              # disable GC (dangerous — memory will grow!)
# ... performance-critical section ...
GC.enable               # re-enable GC
GC.start                # force a GC cycle

# GC statistics
stats = GC.stat
puts "Heap pages: #{stats[:heap_allocated_pages]}"
puts "Total objects: #{stats[:heap_live_slots]}"
puts "GC count: #{stats[:count]}"
```

**Ruby GC vs C++ memory management:**

| Aspect | Ruby (GC) | C++ (Manual/Smart Pointers) |
|--------|-----------|---------------------------|
| Memory management | Automatic (GC) | Manual or RAII (smart pointers) |
| Deallocation timing | Non-deterministic (GC decides) | Deterministic (scope exit / `delete`) |
| Memory leaks | Rare (circular refs handled) | Common if not careful |
| Performance overhead | GC pauses (stop-the-world) | Zero overhead (compile-time) |
| Finalizers/Destructors | Weak (`define_finalizer`) | Strong (deterministic destructors) |
| Copying | Shallow by default (`dup`/`clone`) | Deep copy via copy constructor |
| Move semantics | Not needed (references are cheap) | Critical for performance |
| Ownership model | Shared (GC tracks all refs) | Unique/Shared/Weak (`unique_ptr`, `shared_ptr`) |

**Key insight:** Ruby doesn't need move semantics because variables hold references (pointers), not objects. "Moving" a C++ object avoids an expensive deep copy — in Ruby, assignment just copies a reference (8 bytes), which is already as cheap as a move.

---


## 8.2 Duck Typing & Flexible Polymorphism

> "If it walks like a duck and quacks like a duck, then it must be a duck." Duck typing is Ruby's approach to polymorphism — objects are defined by what they *can do* (which methods they respond to), not by what they *are* (their class hierarchy). This is the Ruby equivalent of C++ type erasure — it achieves polymorphism without requiring a common base class or inheritance hierarchy, but it does so naturally as a core language feature rather than a design pattern.

---

### Why Duck Typing?

In statically typed languages like C++, polymorphism requires inheritance — all types must derive from a common base class. This is intrusive and limiting:
- Can't make built-in types conform to your interface
- Third-party classes can't be retrofitted into your hierarchy
- Inheritance creates tight coupling

**Ruby's duck typing solves this naturally:**

```ruby
# In C++, you'd need a common base class or type erasure.
# In Ruby, ANY object that responds to the right methods just works.

class Duck
  def quack
    "Quack!"
  end

  def swim
    "Swimming gracefully"
  end
end

class Person
  def quack
    "I'm quacking like a duck!"
  end

  def swim
    "Splashing around"
  end
end

class RubberDuck
  def quack
    "Squeak!"
  end

  def swim
    "Floating"
  end
end

# This method works with ANY object that responds to quack and swim
# No inheritance, no interface declaration, no type constraints
def duck_test(duck_like)
  puts duck_like.quack
  puts duck_like.swim
end

duck_test(Duck.new)        # Quack! / Swimming gracefully
duck_test(Person.new)      # I'm quacking like a duck! / Splashing around
duck_test(RubberDuck.new)  # Squeak! / Floating
```

---

### respond_to? — Runtime Type Checking

Duck typing doesn't mean "no checking" — it means checking *capabilities* instead of *types*:

```ruby
# --- Defensive duck typing with respond_to? ---
def process(item)
  if item.respond_to?(:each)
    item.each { |x| puts "  Item: #{x}" }
  elsif item.respond_to?(:to_a)
    item.to_a.each { |x| puts "  Item: #{x}" }
  else
    puts "  Single item: #{item}"
  end
end

process([1, 2, 3])        # iterates with each
process(1..5)              # Range responds to each
process("hello")           # single item (String#each was removed in Ruby 1.9)
process(42)                # single item

# --- Real-world example: flexible serialization ---
class JsonSerializer
  def serialize(data)
    if data.respond_to?(:to_json)
      data.to_json
    elsif data.respond_to?(:to_hash)
      serialize_hash(data.to_hash)
    elsif data.respond_to?(:to_a)
      serialize_array(data.to_a)
    elsif data.respond_to?(:to_s)
      "\"#{data}\""
    else
      raise ArgumentError, "Cannot serialize #{data.class}"
    end
  end

  private

  def serialize_hash(hash)
    pairs = hash.map { |k, v| "\"#{k}\": #{serialize(v)}" }
    "{ #{pairs.join(', ')} }"
  end

  def serialize_array(arr)
    items = arr.map { |item| serialize(item) }
    "[#{items.join(', ')}]"
  end
end

serializer = JsonSerializer.new
puts serializer.serialize({ name: "Alice", scores: [95, 87] })
# { "name": "Alice", "scores": [95, 87] }
```

---

### Protocol-Based Design (Implicit Interfaces)

Ruby doesn't have formal interfaces, but conventions establish implicit protocols — sets of methods that objects are expected to implement:

```ruby
# --- The Comparable protocol ---
# Implement <=> and get <, >, <=, >=, ==, between?, clamp for free
class Temperature
  include Comparable

  attr_reader :celsius

  def initialize(celsius)
    @celsius = celsius
  end

  # The ONLY method you need to implement
  def <=>(other)
    @celsius <=> other.celsius
  end

  def to_s
    "#{@celsius}°C"
  end
end

temps = [Temperature.new(100), Temperature.new(0), Temperature.new(37)]
puts temps.sort.map(&:to_s).join(", ")  # 0°C, 37°C, 100°C
puts temps.min                           # 0°C
puts temps.max                           # 100°C
puts Temperature.new(50).between?(Temperature.new(0), Temperature.new(100))  # true

# --- The Enumerable protocol ---
# Implement each and get map, select, reject, reduce, sort, min, max, etc.
class NumberRange
  include Enumerable

  def initialize(start, stop)
    @start = start
    @stop = stop
  end

  # The ONLY method you need to implement
  def each(&block)
    current = @start
    while current <= @stop
      yield current
      current += 1
    end
  end
end

range = NumberRange.new(1, 10)
puts range.select(&:even?).inspect   # [2, 4, 6, 8, 10]
puts range.map { |n| n ** 2 }.inspect # [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
puts range.reduce(:+)                 # 55
puts range.min                         # 1
puts range.count                       # 10

# --- Custom protocol: Renderable ---
# Convention: any object with a #render method can be rendered
module Renderable
  def render_to(output)
    output << render  # calls the object's render method
  end
end

class HtmlParagraph
  include Renderable

  def initialize(text)
    @text = text
  end

  def render
    "<p>#{@text}</p>"
  end
end

class MarkdownHeading
  include Renderable

  def initialize(text, level = 1)
    @text = text
    @level = level
  end

  def render
    "#{"#" * @level} #{@text}"
  end
end

# Works with any Renderable — no common base class needed
def render_page(components)
  output = []
  components.each { |c| c.render_to(output) }
  output.join("\n")
end

page = render_page([
  MarkdownHeading.new("Title"),
  HtmlParagraph.new("Hello world"),
  MarkdownHeading.new("Section", 2)
])
puts page
# # Title
# <p>Hello world</p>
# ## Section
```

---

### Coercion Protocols — to_s, to_i, to_a, to_hash, to_proc

Ruby uses naming conventions for type coercion. Objects that implement these methods can be used wherever that type is expected:

```ruby
# --- to_s: String coercion (used by puts, string interpolation) ---
class Money
  attr_reader :amount, :currency

  def initialize(amount, currency = "USD")
    @amount = amount
    @currency = currency
  end

  def to_s
    "$#{'%.2f' % @amount} #{@currency}"
  end
end

m = Money.new(42.5)
puts m                    # $42.50 USD (puts calls to_s)
puts "Balance: #{m}"      # Balance: $42.50 USD (interpolation calls to_s)

# --- to_a: Array coercion (used by splat operator, Array()) ---
class Pair
  def initialize(first, second)
    @first = first
    @second = second
  end

  def to_a
    [@first, @second]
  end

  def to_ary  # implicit array coercion (for parallel assignment)
    [@first, @second]
  end
end

pair = Pair.new("hello", "world")
a, b = pair              # parallel assignment uses to_ary
puts "#{a}, #{b}"        # hello, world
puts Array(pair).inspect # ["hello", "world"] — Array() uses to_a

# --- to_proc: Proc coercion (used by & operator) ---
class Multiplier
  def initialize(factor)
    @factor = factor
  end

  def to_proc
    ->(x) { x * @factor }
  end
end

doubler = Multiplier.new(2)
puts [1, 2, 3, 4].map(&doubler).inspect  # [2, 4, 6, 8]

tripler = Multiplier.new(3)
puts [1, 2, 3, 4].map(&tripler).inspect  # [3, 6, 9, 12]

# Symbol#to_proc — the most common example
puts [1, 2, 3].map(&:to_s).inspect       # ["1", "2", "3"]
# &:to_s calls :to_s.to_proc → creates ->(x) { x.to_s }
```

---

### Duck Typing vs C++ Type Erasure

| Aspect | Ruby Duck Typing | C++ Type Erasure |
|--------|-----------------|-----------------|
| Mechanism | Runtime method lookup | Concept/Model pattern (templates + virtual) |
| Type checking | Runtime (`respond_to?`, `NoMethodError`) | Compile-time (template constraints) |
| Performance | Method lookup overhead | Heap allocation + virtual dispatch |
| Inheritance required | No | No |
| Explicit interface | No (implicit by convention) | No (but can use concepts in C++20) |
| Error timing | Runtime (`NoMethodError`) | Compile-time (template error) |
| Flexibility | Maximum (any object, any method) | High (any type satisfying requirements) |
| Safety | Lower (errors at runtime) | Higher (errors at compile time) |
| Heterogeneous collections | Natural (arrays hold any object) | Requires type erasure wrapper |

**Key insight:** What C++ achieves through complex type erasure patterns (Concept/Model, `std::function`, `std::any`), Ruby achieves naturally through duck typing. The trade-off is that Ruby catches type errors at runtime instead of compile time.

---


## 8.3 Metaprogramming — Ruby's Power Tool

> Metaprogramming is writing code that writes code. Ruby's object model is so open and reflective that you can define methods at runtime, modify classes after they're created, intercept method calls, and generate entire APIs dynamically. This is the Ruby equivalent of C++ templates and CRTP — but far more dynamic and flexible. Where C++ generates code at compile time through templates, Ruby generates code at runtime through metaprogramming.

---

### define_method — Dynamic Method Definition

`define_method` creates methods at runtime. This eliminates repetitive method definitions and enables powerful abstractions:

```ruby
# --- Basic define_method ---
class Greeter
  %w[english spanish french german].each do |lang|
    define_method("greet_in_#{lang}") do |name|
      greeting = case lang
                 when "english" then "Hello"
                 when "spanish" then "Hola"
                 when "french"  then "Bonjour"
                 when "german"  then "Hallo"
                 end
      "#{greeting}, #{name}!"
    end
  end
end

g = Greeter.new
puts g.greet_in_english("Alice")   # Hello, Alice!
puts g.greet_in_spanish("Bob")     # Hola, Bob!
puts g.greet_in_french("Charlie")  # Bonjour, Charlie!
puts g.greet_in_german("Diana")    # Hallo, Diana!

# --- Real-world: attribute accessors with validation ---
class Model
  def self.validated_attr(name, &validator)
    # Define getter
    define_method(name) do
      instance_variable_get("@#{name}")
    end

    # Define setter with validation
    define_method("#{name}=") do |value|
      if validator && !validator.call(value)
        raise ArgumentError, "Invalid value for #{name}: #{value.inspect}"
      end
      instance_variable_set("@#{name}", value)
    end
  end
end

class User < Model
  validated_attr(:name) { |v| v.is_a?(String) && !v.empty? }
  validated_attr(:age)  { |v| v.is_a?(Integer) && v.between?(0, 150) }
  validated_attr(:email) { |v| v.is_a?(String) && v.include?("@") }

  def initialize(name, age, email)
    self.name = name
    self.age = age
    self.email = email
  end
end

user = User.new("Alice", 30, "alice@example.com")
puts user.name   # Alice
puts user.age    # 30

# user.age = -5          # ArgumentError: Invalid value for age: -5
# user.name = ""         # ArgumentError: Invalid value for name: ""
# user.email = "invalid" # ArgumentError: Invalid value for email: "invalid"

# --- Generating CRUD methods dynamically ---
class Repository
  def self.crud_for(resource)
    define_method("create_#{resource}") do |attrs|
      puts "Creating #{resource} with #{attrs}"
    end

    define_method("find_#{resource}") do |id|
      puts "Finding #{resource} ##{id}"
    end

    define_method("update_#{resource}") do |id, attrs|
      puts "Updating #{resource} ##{id} with #{attrs}"
    end

    define_method("delete_#{resource}") do |id|
      puts "Deleting #{resource} ##{id}"
    end
  end
end

class UserRepository < Repository
  crud_for :user
  crud_for :post
  crud_for :comment
end

repo = UserRepository.new
repo.create_user(name: "Alice")     # Creating user with {name: "Alice"}
repo.find_post(42)                  # Finding post #42
repo.delete_comment(7)              # Deleting comment #7
```

---

### method_missing — Intercepting Undefined Method Calls

When Ruby can't find a method, it calls `method_missing` on the receiver. Overriding this lets you handle arbitrary method calls dynamically:

```ruby
# --- Basic method_missing ---
class DynamicConfig
  def initialize
    @data = {}
  end

  def method_missing(name, *args)
    method_name = name.to_s

    if method_name.end_with?("=")
      # Setter: config.database_url = "..."
      key = method_name.chomp("=").to_sym
      @data[key] = args.first
    elsif @data.key?(name)
      # Getter: config.database_url
      @data[name]
    else
      super  # IMPORTANT: call super for truly missing methods
    end
  end

  # ALWAYS override respond_to_missing? alongside method_missing
  def respond_to_missing?(name, include_private = false)
    method_name = name.to_s
    method_name.end_with?("=") || @data.key?(name) || super
  end

  def to_h
    @data.dup
  end
end

config = DynamicConfig.new
config.database_url = "postgres://localhost/mydb"
config.redis_host = "localhost"
config.redis_port = 6379

puts config.database_url   # postgres://localhost/mydb
puts config.redis_port     # 6379
puts config.to_h.inspect   # {:database_url=>"postgres://localhost/mydb", ...}

# respond_to? works correctly because we defined respond_to_missing?
puts config.respond_to?(:database_url)  # true
puts config.respond_to?(:unknown)       # false

# --- Ghost methods with method_missing ---
class FluentQuery
  attr_reader :conditions

  def initialize
    @conditions = {}
  end

  def method_missing(name, *args)
    method_name = name.to_s

    if method_name.start_with?("where_")
      field = method_name.sub("where_", "").to_sym
      @conditions[field] = args.first
      self  # return self for chaining
    elsif method_name.start_with?("and_")
      field = method_name.sub("and_", "").to_sym
      @conditions[field] = args.first
      self
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    name.to_s.start_with?("where_", "and_") || super
  end

  def to_sql
    clauses = @conditions.map { |k, v| "#{k} = '#{v}'" }
    "SELECT * WHERE #{clauses.join(' AND ')}"
  end
end

query = FluentQuery.new
  .where_name("Alice")
  .and_age(30)
  .and_city("NYC")

puts query.to_sql
# SELECT * WHERE name = 'Alice' AND age = '30' AND city = 'NYC'
```

**Critical rules for method_missing:**

| Rule | Why |
|------|-----|
| Always call `super` for unhandled methods | Prevents swallowing real errors |
| Always override `respond_to_missing?` | Makes `respond_to?` work correctly |
| Prefer `define_method` when possible | `method_missing` is slower (full method lookup fails first) |
| Keep the logic simple | Complex `method_missing` is hard to debug |
| Document the ghost methods | IDE/tools can't discover them automatically |

---

### class_eval and instance_eval

These methods change the context in which code is evaluated — they're the foundation of many metaprogramming techniques:

```ruby
# --- class_eval (aka module_eval): evaluates in the context of a CLASS ---
# Methods defined inside become INSTANCE methods of the class

class Dog; end

Dog.class_eval do
  def bark
    "Woof!"
  end

  def fetch(item)
    "Fetching #{item}!"
  end
end

puts Dog.new.bark        # Woof!
puts Dog.new.fetch("ball") # Fetching ball!

# class_eval with a string (can define methods with dynamic names)
Dog.class_eval <<-RUBY, __FILE__, __LINE__ + 1
  def sit
    "Sitting!"
  end
RUBY

puts Dog.new.sit         # Sitting!

# --- instance_eval: evaluates in the context of an OBJECT ---
# self becomes the receiver object

obj = Object.new

obj.instance_eval do
  # self is obj here
  @secret = 42  # sets instance variable on obj

  # Defines a singleton method on obj (not on Object class)
  def reveal
    @secret
  end
end

puts obj.reveal          # 42
# Object.new.reveal      # NoMethodError — reveal is only on obj

# --- Practical use: configuration DSL ---
class ServerConfig
  attr_reader :host, :port, :ssl

  def initialize(&block)
    @host = "localhost"
    @port = 80
    @ssl = false
    instance_eval(&block) if block_given?
  end

  private

  def host(value)
    @host = value
  end

  def port(value)
    @port = value
  end

  def ssl(value)
    @ssl = value
  end
end

config = ServerConfig.new do
  host "api.example.com"
  port 443
  ssl true
end

puts "#{config.host}:#{config.port} (SSL: #{config.ssl})"
# api.example.com:443 (SSL: true)
```

**class_eval vs instance_eval:**

| Aspect | `class_eval` | `instance_eval` |
|--------|-------------|-----------------|
| `self` becomes | The class/module | The specific object |
| `def` creates | Instance methods of the class | Singleton methods on the object |
| Use case | Adding methods to a class | Configuring a specific object |
| Equivalent to | Opening the class with `class ... end` | Defining singleton methods |
| Common in | Frameworks, plugins, mixins | DSLs, configuration blocks |

---

### class_eval vs instance_eval — Deep Dive

```ruby
# The difference is subtle but critical:

class MyClass; end
obj = MyClass.new

# class_eval: self = MyClass, def creates instance methods
MyClass.class_eval do
  puts self          # MyClass
  def greet          # instance method of MyClass
    "Hello from #{self.class}"
  end
end

puts obj.greet       # Hello from MyClass
puts MyClass.new.greet # Hello from MyClass — ALL instances get it

# instance_eval: self = obj, def creates singleton methods
obj.instance_eval do
  puts self          # #<MyClass:0x...> (the specific object)
  def special        # singleton method on THIS object only
    "I'm special"
  end
end

puts obj.special     # I'm special
# MyClass.new.special # NoMethodError — only obj has this method

# --- On a class object (classes are objects too!) ---
# instance_eval on a class: self = the class object
# def creates singleton methods on the class = CLASS METHODS
MyClass.instance_eval do
  def class_method
    "I'm a class method"
  end
end

puts MyClass.class_method  # I'm a class method
# This is equivalent to def MyClass.class_method or def self.class_method inside the class
```

---

### open classes and Monkey Patching

Ruby classes are never closed — you can reopen and modify any class at any time, including built-in classes:

```ruby
# --- Reopening a class to add methods ---
class String
  def palindrome?
    self == self.reverse
  end

  def word_count
    split(/\s+/).length
  end
end

puts "racecar".palindrome?              # true
puts "hello world foo".word_count       # 3

# --- Adding methods to Integer ---
class Integer
  def factorial
    return 1 if self <= 1
    self * (self - 1).factorial
  end

  def prime?
    return false if self < 2
    (2..Math.sqrt(self).to_i).none? { |i| self % i == 0 }
  end
end

puts 5.factorial    # 120
puts 17.prime?      # true
puts 15.prime?      # false

# --- DANGER: Monkey patching can break things ---
# BAD: Overriding existing methods
class Array
  # This REPLACES the built-in length method!
  def length
    "I broke Array#length!"
  end
end

# puts [1, 2, 3].length  # "I broke Array#length!" — everything using length is broken!

# RULE: Never override existing methods in monkey patches.
# Only ADD new methods, and namespace them to avoid conflicts.
```

**Monkey patching risks:**

| Risk | Description |
|------|-------------|
| Name collisions | Your method may conflict with a gem or future Ruby version |
| Silent breakage | Overriding existing methods breaks code that depends on them |
| Hard to debug | The source of the method isn't where you'd expect |
| Load order dependency | Behavior depends on which file loads last |
| Testing difficulty | Tests may pass/fail depending on load order |

**Use Refinements instead** (covered in section 8.5) — they scope monkey patches to specific files.

---


## 8.4 Eigenclass (Singleton Class) & Per-Object Behavior

> Every object in Ruby has a hidden class called its **eigenclass** (also called singleton class or metaclass). This invisible class sits between the object and its actual class in the method lookup chain, allowing individual objects to have unique methods and behavior. Understanding the eigenclass is key to understanding how Ruby's object model really works — class methods, singleton methods, and `extend` all operate through the eigenclass.

---

### What is the Eigenclass?

```ruby
# Every object has a hidden singleton class (eigenclass)
obj = Object.new

# Define a method on just this one object
def obj.greet
  "Hello from #{self}"
end

puts obj.greet           # Hello from #<Object:0x...>
# Object.new.greet       # NoMethodError — only obj has this method

# Where does this method live? In obj's eigenclass!
eigen = obj.singleton_class
puts eigen                        # #<Class:#<Object:0x...>>
puts eigen.instance_methods(false) # [:greet]
puts eigen.superclass             # Object

# The eigenclass is inserted into the lookup chain:
# obj → eigenclass → Object → Kernel → BasicObject
```

**Method lookup with eigenclass:**
```
Without eigenclass:          With eigenclass:

obj                          obj
 │                            │
 └──→ Object                  └──→ eigenclass (#<Class:#<Object>>)
       │                            │
       └──→ Kernel                  └──→ Object
              │                           │
              └──→ BasicObject            └──→ Kernel
                                                │
                                                └──→ BasicObject
```

---

### Singleton Methods

Singleton methods are methods defined on a single object. They live in the object's eigenclass:

```ruby
class Animal
  def speak
    "..."
  end
end

cat = Animal.new
dog = Animal.new

# Define singleton method on dog only
def dog.speak
  "Woof!"
end

# Alternative syntax using singleton_class
cat.singleton_class.define_method(:speak) do
  "Meow!"
end

puts dog.speak   # Woof!
puts cat.speak   # Meow!

# New animals still use the original method
puts Animal.new.speak  # ...

# Check for singleton methods
puts dog.singleton_methods.inspect  # [:speak]
puts cat.singleton_methods.inspect  # [:speak]

# --- Class methods ARE singleton methods on the class object ---
class User
  def self.count
    42
  end
end

# User.count is a singleton method on the User object
puts User.singleton_methods.include?(:count)  # true
puts User.singleton_class.instance_methods(false).include?(:count)  # true

# These are ALL equivalent ways to define class methods:
class MyClass
  # Way 1: def self.method_name
  def self.way1
    "class method via self"
  end

  # Way 2: inside singleton class block
  class << self
    def way2
      "class method via class << self"
    end
  end
end

# Way 3: outside the class
def MyClass.way3
  "class method defined outside"
end

# Way 4: using singleton_class
MyClass.singleton_class.define_method(:way4) do
  "class method via singleton_class"
end

puts MyClass.way1  # class method via self
puts MyClass.way2  # class method via class << self
puts MyClass.way3  # class method defined outside
puts MyClass.way4  # class method via singleton_class
```

---

### class << self — Opening the Eigenclass

The `class << obj` syntax opens an object's eigenclass for modification:

```ruby
class Configuration
  # Open the eigenclass of Configuration (the class object)
  class << self
    attr_accessor :environment, :debug_mode

    def configure
      yield self
    end

    def reset!
      @environment = "development"
      @debug_mode = false
    end

    private

    def secret_key
      "abc123"
    end
  end

  # Instance methods
  def initialize
    @settings = {}
  end
end

Configuration.configure do |config|
  config.environment = "production"
  config.debug_mode = true
end

puts Configuration.environment  # production
puts Configuration.debug_mode   # true
Configuration.reset!
puts Configuration.environment  # development

# --- Per-object customization ---
class Button
  attr_accessor :label

  def initialize(label)
    @label = label
  end

  def click
    puts "Button '#{@label}' clicked"
  end
end

submit = Button.new("Submit")
cancel = Button.new("Cancel")

# Customize just the submit button
class << submit
  def click
    puts "SUBMITTING FORM!"
    super  # can still call the original
  end

  def double_click
    puts "Double-clicked submit!"
  end
end

submit.click        # SUBMITTING FORM! / Button 'Submit' clicked
submit.double_click # Double-clicked submit!
cancel.click        # Button 'Cancel' clicked
# cancel.double_click # NoMethodError — only submit has this
```

---

### The Full Method Lookup Path

Understanding the complete lookup chain is essential for advanced Ruby:

```ruby
module Greetable
  def greet
    "Hello from Greetable"
  end
end

class Animal
  def speak
    "..."
  end
end

class Dog < Animal
  include Greetable

  def speak
    "Woof!"
  end
end

rex = Dog.new

# Define singleton method
def rex.speak
  "WOOF WOOF!"
end

# Full lookup path for rex.speak:
# 1. rex's eigenclass (#<Class:#<Dog:0x...>>) → FOUND: "WOOF WOOF!"
# 2. Dog
# 3. Greetable (included module)
# 4. Animal
# 5. Object
# 6. Kernel (mixed into Object)
# 7. BasicObject

# Verify the ancestors chain:
puts rex.singleton_class.ancestors.inspect
# [#<Class:#<Dog:0x...>>, Dog, Greetable, Animal, Object, Kernel, BasicObject]

# For class methods, the lookup goes through eigenclass ancestors:
puts Dog.singleton_class.ancestors.inspect
# [#<Class:Dog>, #<Class:Animal>, #<Class:Object>, #<Class:BasicObject>,
#  Class, Module, Object, Kernel, BasicObject]
```

**Complete method lookup diagram:**
```
Instance method lookup:

rex (object)
 │
 └──→ #<Class:#<Dog>> (rex's eigenclass)
       │
       └──→ Dog (class)
             │
             └──→ Greetable (included module)
                   │
                   └──→ Animal (superclass)
                         │
                         └──→ Object
                               │
                               └──→ Kernel
                                     │
                                     └──→ BasicObject
                                           │
                                           └──→ method_missing

Class method lookup:

Dog (class object)
 │
 └──→ #<Class:Dog> (Dog's eigenclass)
       │
       └──→ #<Class:Animal> (Animal's eigenclass)
             │
             └──→ #<Class:Object>
                   │
                   └──→ #<Class:BasicObject>
                         │
                         └──→ Class
                               │
                               └──→ Module
                                     │
                                     └──→ Object → Kernel → BasicObject
```

---

### extend — Adding Module Methods to a Single Object

`extend` adds a module's methods as singleton methods on a specific object:

```ruby
module Logging
  def log(message)
    puts "[LOG] #{self}: #{message}"
  end
end

module Serializable
  def to_json_string
    vars = instance_variables.map do |var|
      "\"#{var.to_s.delete('@')}\": \"#{instance_variable_get(var)}\""
    end
    "{ #{vars.join(', ')} }"
  end
end

class User
  attr_accessor :name, :email

  def initialize(name, email)
    @name = name
    @email = email
  end

  def to_s
    "User(#{@name})"
  end
end

alice = User.new("Alice", "alice@example.com")
bob = User.new("Bob", "bob@example.com")

# Extend only alice with Logging
alice.extend(Logging)
alice.log("signed in")   # [LOG] User(Alice): signed in
# bob.log("test")        # NoMethodError — bob doesn't have Logging

# Extend only bob with Serializable
bob.extend(Serializable)
puts bob.to_json_string  # { "name": "Bob", "email": "bob@example.com" }
# alice.to_json_string   # NoMethodError — alice doesn't have Serializable

# extend on a class adds CLASS methods (because classes are objects)
class Product
  extend Logging  # Logging methods become class methods
end

Product.log("initialized")  # [LOG] Product: initialized
```

**include vs extend vs prepend:**

| Method | Where methods go | Effect |
|--------|-----------------|--------|
| `include` | Instance methods of the class | All instances get the methods |
| `extend` | Singleton methods of the object | Only that object gets the methods |
| `prepend` | Before the class in lookup chain | Overrides class methods (can call `super`) |

---


## 8.5 Refinements — Scoped Monkey Patching

> Refinements (introduced in Ruby 2.0, stabilized in 2.1) provide a way to modify classes in a **scoped, controlled** manner — unlike monkey patching, which affects the entire program. Refinements let you add or override methods on existing classes, but the changes are only visible within the file or module where they're activated. This is Ruby's answer to the dangers of open classes.

---

### Why Refinements?

Monkey patching is powerful but dangerous — changes are global and permanent:

```ruby
# BAD: Global monkey patch — affects ALL code everywhere
class String
  def to_slug
    downcase.strip.gsub(/\s+/, '-').gsub(/[^a-z0-9\-]/, '')
  end
end

# This affects every String in the entire program, including gems!
# A gem might define its own to_slug differently → conflict!
```

Refinements scope the changes:

```ruby
# GOOD: Refinement — only active where explicitly used
module StringExtensions
  refine String do
    def to_slug
      downcase.strip.gsub(/\s+/, '-').gsub(/[^a-z0-9\-]/, '')
    end

    def truncate(length, omission: "...")
      return self if self.length <= length
      self[0, length - omission.length] + omission
    end
  end
end

# Without `using`, the methods don't exist:
# "Hello World".to_slug  # NoMethodError

# Activate the refinement in this scope:
using StringExtensions

puts "Hello World!".to_slug           # hello-world
puts "A very long string".truncate(10) # A very...

# The refinement is ONLY active in this file, after the `using` statement.
# Other files, gems, and libraries are completely unaffected.
```

---

### Refinement Scope Rules

```ruby
module MathExtensions
  refine Integer do
    def factorial
      return 1 if self <= 1
      self * (self - 1).factorial
    end
  end

  refine Float do
    def round_to(places)
      (self * 10**places).round / 10.0**places
    end
  end
end

# --- Scope rule 1: using activates from that point to end of file ---
# 5.factorial  # NoMethodError — not yet activated

using MathExtensions

puts 5.factorial           # 120
puts 3.14159.round_to(2)   # 3.14

# --- Scope rule 2: Refinements are lexically scoped ---
class Calculator
  using MathExtensions  # active only inside this class definition

  def compute_factorial(n)
    n.factorial  # works here
  end
end

# --- Scope rule 3: Refinements are NOT inherited ---
class ScientificCalculator < Calculator
  # MathExtensions is NOT automatically active here
  # Must use `using MathExtensions` again if needed
end

# --- Scope rule 4: Refinements don't affect dynamic dispatch ---
module Demo
  refine String do
    def shout
      upcase + "!!!"
    end
  end
end

using Demo

puts "hello".shout         # HELLO!!!

# But send bypasses refinements:
# "hello".send(:shout)     # NoMethodError in some Ruby versions
# This is because send uses the regular method lookup, not the refined one
```

---

### Practical Refinement Patterns

```ruby
# --- Pattern 1: Safe extensions for specific contexts ---
module JsonParsing
  refine String do
    def parse_json
      require 'json'
      JSON.parse(self)
    end

    def valid_json?
      require 'json'
      JSON.parse(self)
      true
    rescue JSON::ParserError
      false
    end
  end

  refine Hash do
    def to_pretty_json
      require 'json'
      JSON.pretty_generate(self)
    end
  end
end

# Only used in files that need JSON parsing
using JsonParsing

data = '{"name": "Alice", "age": 30}'.parse_json
puts data["name"]                    # Alice
puts '{"valid": true}'.valid_json?   # true
puts 'not json'.valid_json?          # false
puts({ name: "Bob" }.to_pretty_json)

# --- Pattern 2: Testing helpers ---
module TestingExtensions
  refine Object do
    def must_equal(expected)
      unless self == expected
        raise "Expected #{expected.inspect}, got #{self.inspect}"
      end
      true
    end

    def must_be_kind_of(klass)
      unless self.is_a?(klass)
        raise "Expected #{klass}, got #{self.class}"
      end
      true
    end
  end
end

# In test files only:
using TestingExtensions

"hello".must_equal("hello")     # true
42.must_be_kind_of(Integer)     # true
# "hello".must_equal("world")   # raises: Expected "world", got "hello"

# --- Pattern 3: Domain-specific enhancements ---
module MoneyExtensions
  refine Integer do
    def dollars
      { amount: self * 100, currency: "USD" }
    end

    def euros
      { amount: self * 100, currency: "EUR" }
    end
  end
end

using MoneyExtensions

price = 42.dollars
puts price.inspect  # {:amount=>4200, :currency=>"USD"}
```

---

### Refinements vs Monkey Patching

| Aspect | Monkey Patching | Refinements |
|--------|----------------|-------------|
| Scope | Global (entire program) | Lexical (file/class where `using` is called) |
| Visibility | All code sees changes | Only code after `using` sees changes |
| Conflict risk | High (gems, stdlib may conflict) | Low (scoped to specific files) |
| Debugging | Hard (source unclear) | Easier (explicit `using` shows what's active) |
| Performance | No overhead | Slight overhead (extra lookup step) |
| Dynamic dispatch | Works with `send` | May not work with `send` |
| Inheritance | Inherited by subclasses | NOT inherited by subclasses |
| Use case | Quick prototyping, legacy code | Production code, libraries, safe extensions |

**Rule: Prefer refinements over monkey patching in production code.** Use monkey patching only for quick prototyping or when refinements aren't sufficient.

---


## 8.6 Module Composition & Policy-Based Design

> Ruby's module system enables a form of policy-based design similar to C++ template policies — but resolved at runtime through mixins, blocks, and composition rather than at compile time through templates. Modules act as reusable units of behavior that can be mixed into classes in various combinations, and blocks/procs enable strategy injection. This section covers Ruby's equivalent of C++ policy-based design and the Pimpl idiom.

---

### Modules as Policies (Mixins)

```ruby
# ============================================================
# POLICIES — modules that provide specific behavior
# ============================================================

# --- Output Policies ---
module ConsoleOutput
  def write_output(message)
    puts message
  end
end

module FileOutput
  def write_output(message)
    # Simulated file write
    puts "[FILE] #{message}"
  end
end

module NullOutput
  def write_output(_message)
    # Discard — do nothing
  end
end

# --- Formatting Policies ---
module PlainFormat
  def format_message(level, message)
    "[#{level}] #{message}"
  end
end

module TimestampFormat
  def format_message(level, message)
    "#{Time.now.strftime('%Y-%m-%d %H:%M:%S')} [#{level}] #{message}"
  end
end

module JsonFormat
  def format_message(level, message)
    "{\"level\":\"#{level}\",\"msg\":\"#{message}\"}"
  end
end

# --- Threading Policies ---
module SingleThreaded
  def with_lock
    yield  # no-op — just execute the block
  end
end

module MultiThreaded
  def self.included(base)
    base.instance_variable_set(:@mutex, Mutex.new)
  end

  def with_lock(&block)
    self.class.instance_variable_get(:@mutex).synchronize(&block)
  end
end

# ============================================================
# HOST CLASS — configured by including policy modules
# ============================================================

class Logger
  def initialize(output_policy, format_policy, thread_policy = SingleThreaded)
    # Extend this specific instance with the chosen policies
    extend output_policy
    extend format_policy
    extend thread_policy
  end

  def debug(msg) = log("DEBUG", msg)
  def info(msg)  = log("INFO", msg)
  def warn(msg)  = log("WARN", msg)
  def error(msg) = log("ERROR", msg)

  private

  def log(level, message)
    with_lock do
      formatted = format_message(level, message)
      write_output(formatted)
    end
  end
end

# ============================================================
# Usage — mix and match policies
# ============================================================

# Console + Plain + SingleThreaded
default_logger = Logger.new(ConsoleOutput, PlainFormat)
default_logger.info("Default logger message")
# [INFO] Default logger message

# Console + Timestamp format
timestamp_logger = Logger.new(ConsoleOutput, TimestampFormat)
timestamp_logger.info("With timestamp")
# 2026-04-21 10:30:00 [INFO] With timestamp

# File + JSON format + Thread-safe
production_logger = Logger.new(FileOutput, JsonFormat, MultiThreaded)
production_logger.error("Database connection failed")
# [FILE] {"level":"ERROR","msg":"Database connection failed"}

# Null output (discard all logs) — useful for testing
silent_logger = Logger.new(NullOutput, PlainFormat)
silent_logger.error("This goes nowhere")
```

---

### Strategy via Blocks and Procs

Ruby's blocks provide an elegant alternative to the Strategy pattern — no classes or modules needed for simple strategies:

```ruby
# --- Strategy via blocks ---
class Sorter
  def initialize(&strategy)
    @strategy = strategy || ->(a, b) { a <=> b }  # default: natural order
  end

  def sort(data)
    data.sort(&@strategy)
  end
end

numbers = [3, 1, 4, 1, 5, 9, 2, 6]

# Default sort
puts Sorter.new.sort(numbers).inspect
# [1, 1, 2, 3, 4, 5, 6, 9]

# Reverse sort
puts Sorter.new { |a, b| b <=> a }.sort(numbers).inspect
# [9, 6, 5, 4, 3, 2, 1, 1]

# Sort by digit sum
puts Sorter.new { |a, b| a.digits.sum <=> b.digits.sum }.sort(numbers).inspect

# --- Multiple strategies via Proc objects ---
class PricingEngine
  STRATEGIES = {
    standard: ->(price, _ctx) { price },
    discount: ->(price, ctx) { price * (1 - ctx[:discount_rate]) },
    premium:  ->(price, ctx) { price * (1 + ctx[:premium_rate]) },
    bulk:     ->(price, ctx) {
      qty = ctx[:quantity] || 1
      qty >= 100 ? price * 0.8 : qty >= 10 ? price * 0.9 : price
    }
  }.freeze

  def initialize(strategy_name = :standard)
    @strategy = STRATEGIES.fetch(strategy_name) do
      raise ArgumentError, "Unknown strategy: #{strategy_name}"
    end
  end

  def calculate(price, context = {})
    @strategy.call(price, context)
  end
end

standard = PricingEngine.new(:standard)
puts standard.calculate(100)  # 100

discount = PricingEngine.new(:discount)
puts discount.calculate(100, discount_rate: 0.2)  # 80.0

bulk = PricingEngine.new(:bulk)
puts bulk.calculate(100, quantity: 150)  # 80.0
puts bulk.calculate(100, quantity: 50)   # 90.0
puts bulk.calculate(100, quantity: 5)    # 100
```

---

### Composable Modules with Hooks

Ruby modules can use hooks to configure the including class — this enables sophisticated composition patterns:

```ruby
# --- Module with self.included hook ---
module Timestamps
  def self.included(base)
    base.attr_accessor :created_at, :updated_at

    base.define_method(:touch) do
      @updated_at = Time.now
    end

    # Add class method via extend
    base.extend(ClassMethods)
  end

  module ClassMethods
    def create(**attrs)
      obj = new(**attrs)
      obj.created_at = Time.now
      obj.updated_at = Time.now
      obj
    end
  end
end

module SoftDeletable
  def self.included(base)
    base.attr_accessor :deleted_at

    base.extend(ClassMethods)
  end

  def soft_delete
    @deleted_at = Time.now
  end

  def deleted?
    !@deleted_at.nil?
  end

  def restore
    @deleted_at = nil
  end

  module ClassMethods
    def active_scope(collection)
      collection.reject(&:deleted?)
    end
  end
end

module Validatable
  def self.included(base)
    base.instance_variable_set(:@validations, [])
    base.extend(ClassMethods)
  end

  module ClassMethods
    def validates(field, **options)
      @validations << { field: field, options: options }
    end

    def validations
      @validations
    end
  end

  def valid?
    self.class.validations.all? do |v|
      value = send(v[:field])
      opts = v[:options]

      valid = true
      valid &&= !value.nil? && !value.to_s.empty? if opts[:presence]
      valid &&= value.is_a?(Integer) && value >= opts[:minimum] if opts[:minimum]
      valid &&= value.length <= opts[:max_length] if opts[:max_length] && value.respond_to?(:length)
      valid
    end
  end
end

# --- Compose modules into a model ---
class Article
  include Timestamps
  include SoftDeletable
  include Validatable

  attr_accessor :title, :body, :word_count

  validates :title, presence: true, max_length: 200
  validates :word_count, minimum: 1

  def initialize(title: nil, body: nil, word_count: 0)
    @title = title
    @body = body
    @word_count = word_count
  end
end

article = Article.create(title: "Ruby Metaprogramming", body: "...", word_count: 500)
puts article.created_at          # current time
puts article.valid?              # true
puts article.deleted?            # false

article.soft_delete
puts article.deleted?            # true
article.restore
puts article.deleted?            # false
```

---

### Ruby's Encapsulation Patterns (Pimpl Equivalent)

C++ uses the Pimpl idiom to hide implementation details behind an opaque pointer. Ruby achieves similar encapsulation through different mechanisms:

```ruby
# --- Pattern 1: Private class hidden inside the public class ---
class PaymentProcessor
  # Public interface
  def initialize(api_key)
    @impl = Implementation.new(api_key)
  end

  def charge(amount, currency: "USD")
    @impl.charge(amount, currency)
  end

  def refund(transaction_id)
    @impl.refund(transaction_id)
  end

  def status(transaction_id)
    @impl.status(transaction_id)
  end

  # Implementation is completely hidden
  private

  class Implementation
    def initialize(api_key)
      @api_key = api_key
      @transactions = {}
      @next_id = 1
    end

    def charge(amount, currency)
      id = "txn_#{@next_id}"
      @next_id += 1
      @transactions[id] = {
        amount: amount,
        currency: currency,
        status: :completed,
        timestamp: Time.now
      }
      { transaction_id: id, status: :completed }
    end

    def refund(transaction_id)
      txn = @transactions[transaction_id]
      return { error: "Transaction not found" } unless txn
      txn[:status] = :refunded
      { transaction_id: transaction_id, status: :refunded }
    end

    def status(transaction_id)
      @transactions[transaction_id] || { error: "Not found" }
    end
  end
end

processor = PaymentProcessor.new("sk_test_123")
result = processor.charge(99.99)
puts result.inspect  # {:transaction_id=>"txn_1", :status=>:completed}

# PaymentProcessor::Implementation  # NameError — it's private!
# Users can't access or depend on the implementation details.

# --- Pattern 2: Delegation with Forwardable ---
require 'forwardable'

class PublicAPI
  extend Forwardable

  def initialize
    @engine = InternalEngine.new
  end

  # Only expose specific methods from the internal engine
  def_delegators :@engine, :search, :count
  # :process, :internal_optimize are NOT exposed

  private

  class InternalEngine
    def search(query)
      "Results for: #{query}"
    end

    def count
      42
    end

    def process
      "Internal processing..."
    end

    def internal_optimize
      "Optimizing internals..."
    end
  end
end

api = PublicAPI.new
puts api.search("ruby")  # Results for: ruby
puts api.count            # 42
# api.process             # NoMethodError — not delegated
```

**Ruby encapsulation vs C++ Pimpl:**

| Aspect | C++ Pimpl | Ruby Encapsulation |
|--------|-----------|-------------------|
| Mechanism | Opaque pointer to forward-declared class | Private inner class or delegation |
| Compilation firewall | Yes (main benefit) | N/A (Ruby is interpreted) |
| ABI stability | Yes | N/A (no binary interface) |
| Information hiding | Complete (header shows nothing) | Strong (private class, Forwardable) |
| Performance cost | Heap allocation + indirection | Object allocation (normal Ruby) |
| Use case | Libraries, large C++ projects | Clean APIs, gem design |

---

### Policy-Based Design vs Runtime Strategy in Ruby

| Aspect | Module Composition (Compile-Time-ish) | Block/Proc Strategy (Runtime) |
|--------|--------------------------------------|------------------------------|
| Configuration time | Object creation (`extend`, `include`) | Any time (pass new block) |
| Changeable at runtime | No (once extended, stays) | Yes (swap the proc) |
| Performance | Direct method call | Proc call overhead (minimal) |
| Complexity | More structured (modules) | Simpler (just a block) |
| Reusability | High (modules are named, testable) | Lower (anonymous blocks) |
| Best for | Cross-cutting concerns, frameworks | Simple strategies, one-off behavior |

---


## 8.7 Hooks & Callbacks — Ruby's Lifecycle Events

> Ruby provides a rich set of hooks (callback methods) that are triggered automatically when certain events occur in the object model — when a class is inherited, a module is included, a method is defined, or a constant is missing. These hooks enable frameworks like Rails to provide "magic" behavior and are essential for building sophisticated metaprogramming systems.

---

### Class and Module Hooks

```ruby
# --- inherited: called when a class is subclassed ---
class BaseModel
  def self.inherited(subclass)
    puts "#{subclass} inherits from #{self}"
    subclass.instance_variable_set(:@table_name, subclass.name.downcase + "s")
    (@descendants ||= []) << subclass
  end

  def self.descendants
    @descendants || []
  end

  def self.table_name
    @table_name
  end
end

class User < BaseModel; end       # User inherits from BaseModel
class Product < BaseModel; end    # Product inherits from BaseModel
class Admin < User; end           # Admin inherits from User

puts BaseModel.descendants.inspect  # [User, Product]
puts User.descendants.inspect       # [Admin]
puts User.table_name                # users
puts Product.table_name             # products

# --- included: called when a module is included in a class ---
module Auditable
  def self.included(base)
    puts "#{self} included in #{base}"

    # Add class methods
    base.extend(ClassMethods)

    # Add instance methods
    base.class_eval do
      attr_reader :audit_log

      define_method(:initialize_audit) do
        @audit_log = []
      end
    end
  end

  module ClassMethods
    def audited_methods(*methods)
      methods.each do |method_name|
        original = instance_method(method_name)

        define_method(method_name) do |*args, &block|
          @audit_log ||= []
          @audit_log << {
            method: method_name,
            args: args,
            timestamp: Time.now
          }
          original.bind(self).call(*args, &block)
        end
      end
    end
  end
end

class BankAccount
  include Auditable

  attr_reader :balance

  def initialize(balance = 0)
    @balance = balance
    initialize_audit
  end

  def deposit(amount)
    @balance += amount
  end

  def withdraw(amount)
    @balance -= amount if amount <= @balance
  end

  audited_methods :deposit, :withdraw
end

account = BankAccount.new(1000)
account.deposit(500)
account.withdraw(200)
puts account.balance                    # 1300
puts account.audit_log.length           # 2
puts account.audit_log.first[:method]   # deposit

# --- extended: called when a module is used to extend an object ---
module Cacheable
  def self.extended(obj)
    puts "#{self} extended #{obj}"
    obj.instance_variable_set(:@cache, {})
  end

  def cache_get(key)
    @cache[key]
  end

  def cache_set(key, value)
    @cache[key] = value
  end
end

service = Object.new
service.extend(Cacheable)  # Cacheable extended #<Object:0x...>
service.cache_set(:user_1, { name: "Alice" })
puts service.cache_get(:user_1).inspect  # {:name=>"Alice"}

# --- prepended: called when a module is prepended ---
module Logging
  def self.prepended(base)
    puts "#{self} prepended to #{base}"
  end

  def save
    puts "LOG: Saving #{self.class}..."
    super  # calls the original save method
    puts "LOG: Saved successfully"
  end
end

class Record
  def save
    puts "Saving record to database"
  end
end

Record.prepend(Logging)  # Logging prepended to Record

Record.new.save
# LOG: Saving Record...
# Saving record to database
# LOG: Saved successfully
```

---

### Method Hooks

```ruby
# --- method_added: called when an instance method is defined ---
class StrictClass
  ALLOWED_METHODS = %i[initialize name age to_s].freeze

  def self.method_added(method_name)
    return if @_adding_method  # prevent infinite recursion

    unless ALLOWED_METHODS.include?(method_name)
      @_adding_method = true
      remove_method(method_name)
      @_adding_method = false
      warn "WARNING: Method :#{method_name} is not allowed in #{self}"
    end
  end

  def name
    "allowed"
  end

  # def hack  # WARNING: Method :hack is not allowed in StrictClass
  #   "not allowed"
  # end
end

# --- method_removed / method_undefined ---
class Observable
  def self.method_removed(method_name)
    puts "Method :#{method_name} was removed from #{self}"
  end

  def self.method_undefined(method_name)
    puts "Method :#{method_name} was undefined in #{self}"
  end

  def temporary_method
    "I exist temporarily"
  end
end

# Observable.remove_method(:temporary_method)
# Method :temporary_method was removed from Observable

# --- singleton_method_added: called when a singleton method is defined ---
class Tracked
  def self.singleton_method_added(method_name)
    return if method_name == :singleton_method_added  # skip self
    puts "Class method added: #{method_name}"
  end

  def self.find(id)     # Class method added: find
    "Finding #{id}"
  end

  def self.create(attrs) # Class method added: create
    "Creating with #{attrs}"
  end
end

# --- method_missing and respond_to_missing? (covered in 8.3) ---
# These are also hooks — called when a method isn't found in the lookup chain
```

---

### Constant and Coercion Hooks

```ruby
# --- const_missing: called when a constant is not found ---
class AutoLoader
  def self.const_missing(name)
    puts "Auto-loading constant: #{name}"

    # Simulate auto-loading a class from a file
    case name
    when :UserModel
      const_set(:UserModel, Class.new {
        def initialize(name)
          @name = name
        end
        def to_s
          "User: #{@name}"
        end
      })
    when :ProductModel
      const_set(:ProductModel, Class.new {
        attr_reader :title
        def initialize(title)
          @title = title
        end
      })
    else
      super  # raise NameError for truly missing constants
    end
  end
end

# Constants are loaded on first access
user = AutoLoader::UserModel.new("Alice")  # Auto-loading constant: UserModel
puts user                                   # User: Alice

product = AutoLoader::ProductModel.new("Widget")  # Auto-loading constant: ProductModel
puts product.title                                  # Widget

# Second access doesn't trigger const_missing (constant is now defined)
user2 = AutoLoader::UserModel.new("Bob")  # no auto-load message
puts user2                                 # User: Bob

# --- coerce: called for numeric type coercion ---
class Celsius
  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees.to_f
  end

  def +(other)
    case other
    when Celsius
      Celsius.new(@degrees + other.degrees)
    when Numeric
      Celsius.new(@degrees + other)
    else
      raise TypeError, "Cannot add #{other.class} to Celsius"
    end
  end

  # Called when Ruby doesn't know how to compute: 5 + Celsius.new(10)
  def coerce(other)
    case other
    when Numeric
      [Celsius.new(other), self]
    else
      raise TypeError, "#{other.class} can't be coerced into Celsius"
    end
  end

  def to_s
    "#{@degrees}°C"
  end
end

temp = Celsius.new(20)
puts temp + Celsius.new(5)  # 25.0°C
puts temp + 10              # 30.0°C
puts 10 + temp              # 30.0°C (uses coerce!)
```

---

### Hook Summary Table

| Hook | Triggered When | Defined On |
|------|---------------|-----------|
| `inherited(subclass)` | Class is subclassed | Parent class (class method) |
| `included(base)` | Module is included | The module (module method) |
| `extended(obj)` | Module is used to extend an object | The module (module method) |
| `prepended(base)` | Module is prepended | The module (module method) |
| `method_added(name)` | Instance method defined | The class (class method) |
| `method_removed(name)` | Instance method removed | The class (class method) |
| `method_undefined(name)` | Instance method undefined | The class (class method) |
| `singleton_method_added(name)` | Singleton method defined | The object |
| `const_missing(name)` | Constant not found | The class/module (class method) |
| `method_missing(name, *args)` | Method not found | The object (instance method) |
| `respond_to_missing?(name, priv)` | `respond_to?` check for missing method | The object (instance method) |
| `coerce(other)` | Numeric operation with unknown type | The object (instance method) |

---


## 8.8 Delegation Patterns

> Delegation is a design technique where an object hands off work to a helper object. Ruby provides multiple delegation mechanisms — from the standard library's `Forwardable` and `Delegator` classes to manual delegation and the `SimpleDelegator` pattern. Delegation is often preferred over inheritance because it provides flexibility without the tight coupling of an "is-a" relationship.

---

### Forwardable — Selective Delegation

The `Forwardable` module lets you explicitly delegate specific methods to another object:

```ruby
require 'forwardable'

class Printer
  def print_document(doc)
    puts "Printing: #{doc}"
  end

  def print_status
    "Ready"
  end

  def calibrate
    puts "Calibrating..."
  end
end

class Office
  extend Forwardable

  def initialize
    @printer = Printer.new
    @employees = []
  end

  # Delegate specific methods to @printer
  def_delegator :@printer, :print_document, :print   # rename: print_document → print
  def_delegator :@printer, :print_status              # keep same name
  # calibrate is NOT delegated — it's an internal printer concern

  # Delegate multiple methods at once
  def_delegators :@employees, :size, :each, :<<, :map

  def add_employee(name)
    @employees << name
  end
end

office = Office.new
office.print("Quarterly Report")  # Printing: Quarterly Report
puts office.print_status           # Ready
# office.calibrate                 # NoMethodError — not delegated

office.add_employee("Alice")
office.add_employee("Bob")
puts office.size                   # 2
puts office.map(&:upcase).inspect  # ["ALICE", "BOB"]
```

---

### SimpleDelegator — Wrap and Override

`SimpleDelegator` wraps an object and forwards all method calls to it. You can selectively override methods:

```ruby
require 'delegate'

class User
  attr_accessor :name, :email, :role

  def initialize(name, email, role = :user)
    @name = name
    @email = email
    @role = role
  end

  def admin?
    @role == :admin
  end

  def to_s
    "#{@name} <#{@email}>"
  end
end

# --- Decorator using SimpleDelegator ---
class UserPresenter < SimpleDelegator
  def display_name
    admin? ? "👑 #{name}" : name
  end

  def masked_email
    parts = email.split("@")
    "#{parts[0][0..2]}***@#{parts[1]}"
  end

  # Override to_s
  def to_s
    "#{display_name} (#{masked_email})"
  end
end

user = User.new("Alice", "alice@example.com", :admin)
presenter = UserPresenter.new(user)

puts presenter.display_name   # 👑 Alice
puts presenter.masked_email   # ali***@example.com
puts presenter.to_s           # 👑 Alice (ali***@example.com)
puts presenter.name           # Alice (delegated to user)
puts presenter.admin?         # true (delegated to user)
puts presenter.class          # UserPresenter
puts presenter.__getobj__.class  # User (the wrapped object)

# --- Swapping the wrapped object ---
bob = User.new("Bob", "bob@example.com")
presenter.__setobj__(bob)
puts presenter.display_name   # Bob (now wrapping bob)

# --- Stacking decorators ---
class TimestampedUser < SimpleDelegator
  def to_s
    "#{super} [accessed: #{Time.now.strftime('%H:%M')}]"
  end
end

decorated = TimestampedUser.new(UserPresenter.new(user))
puts decorated.to_s  # 👑 Alice (ali***@example.com) [accessed: 14:30]
```

---

### DelegateClass — Type-Specific Delegation

`DelegateClass` creates a delegation class for a specific type, providing better documentation and type checking:

```ruby
require 'delegate'

class EnhancedArray < DelegateClass(Array)
  def initialize(*args)
    super(args)  # wrap a real Array
  end

  def mean
    return 0.0 if empty?
    sum.to_f / size
  end

  def median
    sorted = sort
    mid = size / 2
    size.odd? ? sorted[mid] : (sorted[mid - 1] + sorted[mid]) / 2.0
  end

  def mode
    freq = tally
    max_freq = freq.values.max
    freq.select { |_, v| v == max_freq }.keys
  end

  def to_s
    "EnhancedArray(#{super})"
  end
end

arr = EnhancedArray.new(3, 1, 4, 1, 5, 9, 2, 6)
puts arr.sort.inspect   # [1, 1, 2, 3, 4, 5, 6, 9] (delegated to Array)
puts arr.mean            # 3.875
puts arr.median          # 3.5
puts arr.mode.inspect    # [1]
puts arr.select(&:odd?).inspect  # [3, 1, 1, 5, 9] (delegated)
```

---

### Manual Delegation with method_missing

For maximum control, you can implement delegation manually:

```ruby
class LazyProxy
  def initialize(&factory)
    @factory = factory
    @target = nil
  end

  def method_missing(name, *args, **kwargs, &block)
    target.send(name, *args, **kwargs, &block)
  end

  def respond_to_missing?(name, include_private = false)
    target.respond_to?(name, include_private)
  end

  private

  def target
    @target ||= @factory.call
  end
end

# The expensive object is only created when first used
expensive_service = LazyProxy.new do
  puts "Creating expensive service..."
  sleep(0.1)  # simulate slow initialization
  { status: "ready", data: [1, 2, 3] }
end

puts "Service created (but not initialized yet)"
puts expensive_service[:status]  # Creating expensive service... / ready
puts expensive_service[:data].inspect  # [1, 2, 3] (no re-creation)
```

---

### Delegation Comparison

| Technique | Use Case | Pros | Cons |
|-----------|----------|------|------|
| `Forwardable` | Selective delegation | Explicit, clear API | Verbose for many methods |
| `SimpleDelegator` | Decorator pattern | Forwards everything, easy override | Wraps entire object |
| `DelegateClass` | Type-specific wrapper | Better docs, type-aware | Less flexible |
| `method_missing` | Dynamic/lazy delegation | Maximum flexibility | Harder to debug, slower |
| Manual delegation | Simple cases | No dependencies | Boilerplate |

---


## 8.9 DSL Building — Domain-Specific Languages

> A Domain-Specific Language (DSL) is a mini-language tailored to a specific problem domain. Ruby's flexible syntax — optional parentheses, blocks, `instance_eval`, and metaprogramming — makes it uniquely suited for building internal DSLs. Frameworks like Rails, RSpec, Rake, and Sinatra are all built on Ruby DSLs. This section teaches you how to build your own.

---

### Why DSLs?

DSLs make code read like natural language for the domain, hiding implementation complexity:

```ruby
# Without DSL — verbose, implementation-focused
router = Router.new
router.add_route("GET", "/users", UsersController, :index)
router.add_route("POST", "/users", UsersController, :create)
router.add_route("GET", "/users/:id", UsersController, :show)

# With DSL — reads like a specification
Router.draw do
  get  "/users",     to: "users#index"
  post "/users",     to: "users#create"
  get  "/users/:id", to: "users#show"
end
```

---

### Basic DSL with instance_eval

```ruby
# --- Configuration DSL ---
class AppConfig
  attr_reader :settings

  def initialize
    @settings = {}
  end

  def self.configure(&block)
    config = new
    config.instance_eval(&block)
    config
  end

  private

  def app_name(name)
    @settings[:app_name] = name
  end

  def version(ver)
    @settings[:version] = ver
  end

  def database(&block)
    db_config = DatabaseConfig.new
    db_config.instance_eval(&block)
    @settings[:database] = db_config.to_h
  end

  def cache(&block)
    cache_config = CacheConfig.new
    cache_config.instance_eval(&block)
    @settings[:cache] = cache_config.to_h
  end
end

class DatabaseConfig
  def initialize
    @config = {}
  end

  def adapter(val);  @config[:adapter] = val; end
  def host(val);     @config[:host] = val; end
  def port(val);     @config[:port] = val; end
  def database(val); @config[:database] = val; end
  def pool(val);     @config[:pool] = val; end

  def to_h
    @config
  end
end

class CacheConfig
  def initialize
    @config = {}
  end

  def store(val);    @config[:store] = val; end
  def ttl(val);      @config[:ttl] = val; end
  def namespace(val); @config[:namespace] = val; end

  def to_h
    @config
  end
end

# --- Usage: reads like a config file ---
config = AppConfig.configure do
  app_name "MyApp"
  version  "2.1.0"

  database do
    adapter  "postgresql"
    host     "localhost"
    port     5432
    database "myapp_production"
    pool     10
  end

  cache do
    store     "redis"
    ttl       3600
    namespace "myapp"
  end
end

puts config.settings.inspect
# {:app_name=>"MyApp", :version=>"2.1.0",
#  :database=>{:adapter=>"postgresql", :host=>"localhost", :port=>5432, ...},
#  :cache=>{:store=>"redis", :ttl=>3600, :namespace=>"myapp"}}
```

---

### Builder DSL with Method Chaining

```ruby
class HtmlBuilder
  def initialize
    @elements = []
  end

  def self.build(&block)
    builder = new
    builder.instance_eval(&block)
    builder.to_html
  end

  def div(attrs = {}, &block)
    @elements << Element.new("div", attrs, &block)
    self
  end

  def p(text = nil, attrs = {})
    @elements << Element.new("p", attrs, text: text)
    self
  end

  def h1(text, attrs = {})
    @elements << Element.new("h1", attrs, text: text)
    self
  end

  def h2(text, attrs = {})
    @elements << Element.new("h2", attrs, text: text)
    self
  end

  def ul(&block)
    list = ListBuilder.new
    list.instance_eval(&block)
    @elements << list
    self
  end

  def to_html(indent = 0)
    @elements.map { |e| e.to_html(indent) }.join("\n")
  end

  class Element
    def initialize(tag, attrs = {}, text: nil, &block)
      @tag = tag
      @attrs = attrs
      @text = text
      @children = []
      if block_given?
        child_builder = HtmlBuilder.new
        child_builder.instance_eval(&block)
        @children = child_builder.instance_variable_get(:@elements)
      end
    end

    def to_html(indent = 0)
      prefix = "  " * indent
      attr_str = @attrs.map { |k, v| " #{k}=\"#{v}\"" }.join

      if @children.any?
        inner = @children.map { |c| c.to_html(indent + 1) }.join("\n")
        "#{prefix}<#{@tag}#{attr_str}>\n#{inner}\n#{prefix}</#{@tag}>"
      elsif @text
        "#{prefix}<#{@tag}#{attr_str}>#{@text}</#{@tag}>"
      else
        "#{prefix}<#{@tag}#{attr_str} />"
      end
    end
  end

  class ListBuilder
    def initialize
      @items = []
    end

    def li(text)
      @items << text
    end

    def to_html(indent = 0)
      prefix = "  " * indent
      items = @items.map { |i| "#{prefix}  <li>#{i}</li>" }.join("\n")
      "#{prefix}<ul>\n#{items}\n#{prefix}</ul>"
    end
  end
end

# --- Usage: reads like HTML structure ---
html = HtmlBuilder.build do
  div(class: "container") do
    h1 "Welcome to Ruby DSLs"
    p "This page was built with a DSL.", class: "intro"

    div(class: "content") do
      h2 "Features"
      ul do
        li "Clean syntax"
        li "Metaprogramming power"
        li "Block-based configuration"
      end
    end
  end
end

puts html
# <div class="container">
#   <h1>Welcome to Ruby DSLs</h1>
#   <p class="intro">This page was built with a DSL.</p>
#   <div class="content">
#     <h2>Features</h2>
#     <ul>
#       <li>Clean syntax</li>
#       <li>Metaprogramming power</li>
#       <li>Block-based configuration</li>
#     </ul>
#   </div>
# </div>
```

---

### Testing DSL (RSpec-Style)

```ruby
# --- Build a mini testing framework DSL ---
module MiniSpec
  class Suite
    attr_reader :results

    def initialize(description)
      @description = description
      @examples = []
      @results = []
      @before_hooks = []
    end

    def before(&block)
      @before_hooks << block
    end

    def it(description, &block)
      @examples << { description: description, block: block }
    end

    def run
      puts "\n#{@description}"
      @examples.each do |example|
        context = ExampleContext.new
        @before_hooks.each { |hook| context.instance_eval(&hook) }
        begin
          context.instance_eval(&example[:block])
          puts "  ✓ #{example[:description]}"
          @results << { description: example[:description], status: :pass }
        rescue => e
          puts "  ✗ #{example[:description]} — #{e.message}"
          @results << { description: example[:description], status: :fail, error: e.message }
        end
      end
    end
  end

  class ExampleContext
    def expect(value)
      Expectation.new(value)
    end
  end

  class Expectation
    def initialize(actual)
      @actual = actual
    end

    def to(matcher)
      unless matcher.matches?(@actual)
        raise "Expected #{matcher.expected.inspect}, got #{@actual.inspect}"
      end
    end

    def not_to(matcher)
      if matcher.matches?(@actual)
        raise "Expected NOT #{matcher.expected.inspect}, got #{@actual.inspect}"
      end
    end
  end

  class EqualMatcher
    attr_reader :expected

    def initialize(expected)
      @expected = expected
    end

    def matches?(actual)
      actual == @expected
    end
  end

  class BeTruthyMatcher
    def expected; "truthy"; end
    def matches?(actual); !!actual; end
  end

  # DSL entry point
  def self.describe(description, &block)
    suite = Suite.new(description)
    suite.instance_eval(&block)
    suite.run
    suite
  end

  # Matcher helpers
  def self.eq(value)
    EqualMatcher.new(value)
  end

  def self.be_truthy
    BeTruthyMatcher.new
  end
end

# --- Usage: reads like RSpec ---
MiniSpec.describe "Calculator" do
  before do
    @calc_result = 2 + 3
  end

  it "adds two numbers" do
    expect(@calc_result).to MiniSpec.eq(5)
  end

  it "handles zero" do
    expect(0 + 0).to MiniSpec.eq(0)
  end

  it "returns truthy for non-zero" do
    expect(42).to MiniSpec.be_truthy
  end
end
# Calculator
#   ✓ adds two numbers
#   ✓ handles zero
#   ✓ returns truthy for non-zero
```

---

### Router DSL (Sinatra/Rails-Style)

```ruby
class Router
  Route = Struct.new(:method, :path, :handler, :constraints)

  def initialize
    @routes = []
  end

  def self.draw(&block)
    router = new
    router.instance_eval(&block)
    router
  end

  def get(path, to: nil, constraints: {}, &block)
    add_route(:GET, path, to || block, constraints)
  end

  def post(path, to: nil, constraints: {}, &block)
    add_route(:POST, path, to || block, constraints)
  end

  def put(path, to: nil, constraints: {}, &block)
    add_route(:PUT, path, to || block, constraints)
  end

  def delete(path, to: nil, constraints: {}, &block)
    add_route(:DELETE, path, to || block, constraints)
  end

  def namespace(prefix, &block)
    ns = NamespaceContext.new(prefix, self)
    ns.instance_eval(&block)
  end

  def resources(name, only: %i[index show create update destroy])
    only.each do |action|
      case action
      when :index   then get    "/#{name}",          to: "#{name}#index"
      when :show    then get    "/#{name}/:id",      to: "#{name}#show"
      when :create  then post   "/#{name}",          to: "#{name}#create"
      when :update  then put    "/#{name}/:id",      to: "#{name}#update"
      when :destroy then delete "/#{name}/:id",      to: "#{name}#destroy"
      end
    end
  end

  def match(method, path)
    @routes.find do |route|
      route.method == method && path_matches?(route.path, path)
    end
  end

  def print_routes
    @routes.each do |route|
      handler = route.handler.is_a?(String) ? route.handler : "block"
      printf "  %-8s %-25s → %s\n", route.method, route.path, handler
    end
  end

  # Allow NamespaceContext to add routes with prefixed paths
  def add_route(method, path, handler, constraints = {})
    @routes << Route.new(method, path, handler, constraints)
  end

  private

  def path_matches?(pattern, path)
    regex = pattern.gsub(/:(\w+)/, '(?<\1>[^/]+)')
    path.match?(/\A#{regex}\z/)
  end

  class NamespaceContext
    def initialize(prefix, router)
      @prefix = prefix
      @router = router
    end

    def get(path, **opts, &block)
      @router.add_route(:GET, "/#{@prefix}#{path}", opts[:to] || block)
    end

    def post(path, **opts, &block)
      @router.add_route(:POST, "/#{@prefix}#{path}", opts[:to] || block)
    end

    def resources(name, **opts)
      # Delegate to router with prefixed paths
      original_routes_count = @router.instance_variable_get(:@routes).size
      @router.resources(name, **opts)
      routes = @router.instance_variable_get(:@routes)
      # Prefix the newly added routes
      routes[original_routes_count..].each do |route|
        route.path = "/#{@prefix}#{route.path}"
      end
    end
  end
end

# --- Usage: reads like Rails routes ---
router = Router.draw do
  get  "/",          to: "pages#home"
  get  "/about",     to: "pages#about"

  resources :users, only: [:index, :show, :create]
  resources :posts

  namespace :api do
    get "/status", to: "api#status"
    resources :tokens, only: [:index, :create]
  end
end

puts "Routes:"
router.print_routes
# Routes:
#   GET      /                         → pages#home
#   GET      /about                    → pages#about
#   GET      /users                    → users#index
#   GET      /users/:id                → users#show
#   POST     /users                    → users#create
#   GET      /posts                    → posts#index
#   ...
```

---

### DSL Building Techniques Summary

| Technique | How | Best For |
|-----------|-----|----------|
| `instance_eval(&block)` | Evaluates block in object's context | Configuration DSLs |
| `class_eval` / `module_eval` | Evaluates in class context | Adding methods dynamically |
| `define_method` | Creates methods at runtime | Generating repetitive methods |
| `method_missing` | Catches undefined method calls | Fluid/ghost method DSLs |
| Method chaining (`return self`) | Each method returns the builder | Builder pattern DSLs |
| Blocks and Procs | Pass behavior as arguments | Callback-style DSLs |
| `Struct` / `OpenStruct` | Quick data containers | Simple configuration objects |

**DSL design principles:**
1. **Read like prose** — the DSL should be readable by non-programmers
2. **Fail loudly** — give clear error messages for invalid DSL usage
3. **Minimize surprise** — follow Ruby conventions and idioms
4. **Document the DSL** — ghost methods and `instance_eval` hide the API from IDEs
5. **Test the DSL** — test both valid usage and error cases

---


## 8.10 Type Checking Patterns & Contracts

> Ruby is dynamically typed — the interpreter doesn't enforce types at compile time. While duck typing is powerful, large codebases benefit from explicit type checking to catch bugs early, document intent, and prevent entire categories of errors. This section covers Ruby's type checking patterns — from simple runtime checks to design-by-contract, Sorbet type annotations, and strong typedef equivalents. This is the Ruby equivalent of C++ `enum class`, strong typedefs, and `explicit` constructors.

---

### Runtime Type Checking

```ruby
# --- Basic type checking with is_a? and kind_of? ---
def transfer(from_account, to_account, amount)
  raise TypeError, "from_account must be an Account" unless from_account.is_a?(Account)
  raise TypeError, "to_account must be an Account" unless to_account.is_a?(Account)
  raise TypeError, "amount must be Numeric" unless amount.is_a?(Numeric)
  raise ArgumentError, "amount must be positive" unless amount > 0

  from_account.withdraw(amount)
  to_account.deposit(amount)
end

# --- Duck type checking with respond_to? ---
def serialize(object)
  if object.respond_to?(:to_json)
    object.to_json
  elsif object.respond_to?(:to_hash)
    object.to_hash.to_s
  else
    raise ArgumentError, "#{object.class} is not serializable (needs #to_json or #to_hash)"
  end
end

# --- Pattern matching for type checking (Ruby 3.0+) ---
def process_value(value)
  case value
  in Integer => n if n > 0
    "Positive integer: #{n}"
  in Integer => n
    "Non-positive integer: #{n}"
  in String => s if s.length > 0
    "Non-empty string: #{s}"
  in Array => arr
    "Array with #{arr.length} elements"
  in { name: String => name, age: Integer => age }
    "Person: #{name}, age #{age}"
  else
    "Unknown type: #{value.class}"
  end
end

puts process_value(42)                          # Positive integer: 42
puts process_value(-5)                          # Non-positive integer: -5
puts process_value("hello")                     # Non-empty string: hello
puts process_value([1, 2, 3])                   # Array with 3 elements
puts process_value({ name: "Alice", age: 30 })  # Person: Alice, age 30
```

---

### Design by Contract

Design by Contract (DbC) enforces preconditions, postconditions, and invariants:

```ruby
# --- Contract module for Design by Contract ---
module Contracts
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def contract(method_name, pre: nil, post: nil)
      original = instance_method(method_name)

      define_method(method_name) do |*args, &block|
        # Check preconditions
        if pre
          unless instance_exec(*args, &pre)
            raise ContractViolation,
              "Precondition failed for #{self.class}##{method_name} with args: #{args.inspect}"
          end
        end

        # Execute the method
        result = original.bind(self).call(*args, &block)

        # Check postconditions
        if post
          unless instance_exec(result, *args, &post)
            raise ContractViolation,
              "Postcondition failed for #{self.class}##{method_name}: result=#{result.inspect}"
          end
        end

        result
      end
    end
  end

  class ContractViolation < RuntimeError; end
end

# --- Usage ---
class BankAccount
  include Contracts

  attr_reader :balance, :owner

  def initialize(owner, balance = 0)
    @owner = owner
    @balance = balance
  end

  def deposit(amount)
    @balance += amount
  end

  contract :deposit,
    pre:  ->(amount) { amount.is_a?(Numeric) && amount > 0 },
    post: ->(result, amount) { result == @balance && @balance >= amount }

  def withdraw(amount)
    @balance -= amount
    @balance
  end

  contract :withdraw,
    pre:  ->(amount) { amount.is_a?(Numeric) && amount > 0 && amount <= @balance },
    post: ->(result, _amount) { result >= 0 }

  def transfer_to(other, amount)
    withdraw(amount)
    other.deposit(amount)
  end

  contract :transfer_to,
    pre: ->(other, amount) {
      other.is_a?(BankAccount) && amount > 0 && amount <= @balance
    }
end

alice = BankAccount.new("Alice", 1000)
bob = BankAccount.new("Bob", 500)

alice.deposit(200)
puts alice.balance  # 1200

alice.transfer_to(bob, 300)
puts alice.balance  # 900
puts bob.balance    # 800

# alice.deposit(-50)     # ContractViolation: Precondition failed
# alice.withdraw(10000)  # ContractViolation: Precondition failed
```

---

### Value Objects — Strong Typedefs in Ruby

Ruby doesn't have strong typedefs like C++, but you can create value objects that prevent parameter mix-ups:

```ruby
# --- Strong typedef using a generic wrapper ---
class StrongType
  include Comparable

  attr_reader :value

  def initialize(value)
    @value = value
    freeze  # value objects should be immutable
  end

  def <=>(other)
    raise TypeError, "Cannot compare #{self.class} with #{other.class}" unless self.class == other.class
    @value <=> other.value
  end

  def ==(other)
    self.class == other.class && @value == other.value
  end
  alias eql? ==

  def hash
    [self.class, @value].hash
  end

  def to_s
    "#{self.class.name.split('::').last}(#{@value})"
  end

  def inspect
    to_s
  end

  # Prevent arithmetic between different types
  def +(other)
    raise TypeError, "Cannot add #{other.class} to #{self.class}" unless self.class == other.class
    self.class.new(@value + other.value)
  end

  def -(other)
    raise TypeError, "Cannot subtract #{other.class} from #{self.class}" unless self.class == other.class
    self.class.new(@value - other.value)
  end

  def *(scalar)
    raise TypeError, "Can only multiply by Numeric" unless scalar.is_a?(Numeric)
    self.class.new(@value * scalar)
  end
end

# --- Define distinct types ---
class Meters < StrongType; end
class Seconds < StrongType; end
class Kilograms < StrongType; end

class MetersPerSecond < StrongType; end

def calculate_speed(distance, time)
  raise TypeError, "Expected Meters, got #{distance.class}" unless distance.is_a?(Meters)
  raise TypeError, "Expected Seconds, got #{time.class}" unless time.is_a?(Seconds)

  MetersPerSecond.new(distance.value / time.value.to_f)
end

distance = Meters.new(100)
time = Seconds.new(9.58)

speed = calculate_speed(distance, time)
puts speed  # MetersPerSecond(10.438413361169102)

# These raise TypeError — can't mix up parameters!
# calculate_speed(time, distance)     # TypeError: Expected Meters, got Seconds
# calculate_speed(distance, distance) # TypeError: Expected Seconds, got Meters

# Can't add different types
total_distance = Meters.new(50) + Meters.new(30)
puts total_distance  # Meters(80)

# Meters.new(50) + Seconds.new(10)  # TypeError: Cannot add Seconds to Meters

# --- Domain IDs ---
class UserId < StrongType; end
class OrderId < StrongType; end

def process_order(user_id, order_id)
  raise TypeError, "Expected UserId" unless user_id.is_a?(UserId)
  raise TypeError, "Expected OrderId" unless order_id.is_a?(OrderId)
  puts "Processing order #{order_id} for user #{user_id}"
end

process_order(UserId.new(42), OrderId.new(1001))
# process_order(OrderId.new(1001), UserId.new(42))  # TypeError!
```

---

### Enum-Like Patterns in Ruby

Ruby doesn't have `enum class` like C++, but there are several patterns for type-safe enumerations:

```ruby
# --- Pattern 1: Frozen constants module ---
module HttpStatus
  OK                  = 200
  CREATED             = 201
  BAD_REQUEST         = 400
  UNAUTHORIZED        = 401
  FORBIDDEN           = 403
  NOT_FOUND           = 404
  INTERNAL_ERROR      = 500
  SERVICE_UNAVAILABLE = 503

  ALL = constants.map { |c| const_get(c) }.freeze

  def self.success?(code)
    code >= 200 && code < 300
  end

  def self.client_error?(code)
    code >= 400 && code < 500
  end

  def self.server_error?(code)
    code >= 500 && code < 600
  end

  def self.name_for(code)
    constants.find { |c| const_get(c) == code }&.to_s&.downcase&.tr('_', ' ')
  end

  freeze  # prevent adding new constants
end

puts HttpStatus::NOT_FOUND              # 404
puts HttpStatus.success?(200)           # true
puts HttpStatus.client_error?(404)      # true
puts HttpStatus.name_for(503)           # service unavailable

# --- Pattern 2: Type-safe enum with objects ---
class Color
  attr_reader :name, :hex

  REGISTRY = {}

  def initialize(name, hex)
    @name = name.freeze
    @hex = hex.freeze
    REGISTRY[name] = self
    freeze
  end

  RED   = new(:red,   "#FF0000")
  GREEN = new(:green, "#00FF00")
  BLUE  = new(:blue,  "#0000FF")

  def self.all
    REGISTRY.values
  end

  def self.from_name(name)
    REGISTRY.fetch(name) { raise ArgumentError, "Unknown color: #{name}" }
  end

  def to_s
    "Color(#{@name}: #{@hex})"
  end

  # Prevent creating new instances
  private_class_method :new
end

puts Color::RED           # Color(red: #FF0000)
puts Color.from_name(:blue) # Color(blue: #0000FF)
puts Color.all.map(&:to_s).inspect
# Color.new(:yellow, "#FFFF00")  # NoMethodError: private method 'new'

# --- Pattern 3: Comparable enum with Struct ---
Permission = Struct.new(:name, :level) do
  include Comparable

  def <=>(other)
    level <=> other.level
  end

  def to_s
    name.to_s
  end

  NONE    = new(:none, 0).freeze
  READ    = new(:read, 1).freeze
  WRITE   = new(:write, 2).freeze
  ADMIN   = new(:admin, 3).freeze

  ALL = [NONE, READ, WRITE, ADMIN].freeze
end

puts Permission::WRITE > Permission::READ  # true
puts Permission::ADMIN >= Permission::WRITE # true
puts Permission::ALL.sort.map(&:to_s).inspect # ["none", "read", "write", "admin"]
```

---

### RBS and Sorbet — Static Type Checking

Ruby 3.0+ includes RBS (Ruby Signature) for type annotations, and Sorbet is a popular third-party type checker:

```ruby
# --- RBS type signatures (in .rbs files) ---
# These go in sig/user.rbs, NOT in the Ruby file:
#
# class User
#   attr_reader name: String
#   attr_reader age: Integer
#   attr_reader email: String
#
#   def initialize: (String name, Integer age, String email) -> void
#   def adult?: () -> bool
#   def greeting: () -> String
#   def self.create: (String name, Integer age, String email) -> User
# end

# --- Sorbet type annotations (inline) ---
# With Sorbet gem installed:
#
# # typed: strict
# class User
#   extend T::Sig
#
#   sig { params(name: String, age: Integer, email: String).void }
#   def initialize(name, age, email)
#     @name = name
#     @age = age
#     @email = email
#   end
#
#   sig { returns(T::Boolean) }
#   def adult?
#     @age >= 18
#   end
#
#   sig { params(amount: Numeric).returns(String) }
#   def format_balance(amount)
#     "$#{'%.2f' % amount}"
#   end
#
#   sig { params(items: T::Array[String]).returns(T::Array[String]) }
#   def filter_items(items)
#     items.select { |item| item.length > 3 }
#   end
# end

# --- Practical: runtime type checking module (lightweight Sorbet alternative) ---
module TypeChecked
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def sig(method_name, params: {}, returns: nil)
      original = instance_method(method_name)

      define_method(method_name) do |*args, **kwargs, &block|
        # Check parameter types
        param_list = params.keys
        args.each_with_index do |arg, i|
          expected = params[param_list[i]]
          next unless expected
          unless arg.is_a?(expected)
            raise TypeError,
              "#{self.class}##{method_name}: expected #{param_list[i]} to be #{expected}, got #{arg.class}"
          end
        end

        result = original.bind(self).call(*args, **kwargs, &block)

        # Check return type
        if returns && !result.is_a?(returns)
          raise TypeError,
            "#{self.class}##{method_name}: expected return type #{returns}, got #{result.class}"
        end

        result
      end
    end
  end
end

class Calculator
  include TypeChecked

  def add(a, b)
    a + b
  end
  sig :add, params: { a: Numeric, b: Numeric }, returns: Numeric

  def concat(a, b)
    "#{a}#{b}"
  end
  sig :concat, params: { a: String, b: String }, returns: String
end

calc = Calculator.new
puts calc.add(3, 4)           # 7
puts calc.concat("hello", " world")  # hello world

# calc.add("3", 4)            # TypeError: expected a to be Numeric, got String
# calc.concat(3, 4)           # TypeError: expected a to be String, got Integer
```

---

### Type Checking Comparison

| Approach | When Checked | Overhead | Safety | Effort |
|----------|-------------|----------|--------|--------|
| Duck typing (`respond_to?`) | Runtime | Minimal | Low | Low |
| `is_a?` / `kind_of?` checks | Runtime | Minimal | Medium | Low |
| Design by Contract | Runtime | Low | High | Medium |
| Value objects (strong types) | Runtime | Low | High | Medium |
| Pattern matching | Runtime | Low | Medium | Low |
| RBS signatures | Static analysis (separate tool) | Zero runtime | High | Medium |
| Sorbet annotations | Static analysis + runtime | Optional runtime | Highest | High |

**When to use what:**

| Scenario | Recommended Approach |
|----------|---------------------|
| Quick scripts, prototypes | Duck typing (no checks) |
| Library public APIs | `is_a?` checks + clear error messages |
| Financial/safety-critical code | Design by Contract + Value objects |
| Large team projects | Sorbet or RBS |
| Preventing parameter mix-ups | Value objects (strong typedefs) |
| Domain modeling | Enum patterns + Value objects |

**The principle (same as C++):** Make illegal states unrepresentable. Even in a dynamic language, good type design prevents bugs before they happen.

---
