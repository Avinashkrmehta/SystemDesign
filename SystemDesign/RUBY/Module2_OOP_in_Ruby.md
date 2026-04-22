# Module 2: Object-Oriented Programming (OOP) in Ruby

> This module covers the four pillars of OOP — Encapsulation, Inheritance, Polymorphism, and Abstraction — along with practical Ruby implementation details essential for Low-Level Design. Each topic is explained in depth with multiple examples, edge cases, internal workings, and common pitfalls.

---

## 2.1 Classes & Objects

### Class Definition and Access Control

A **class** is a blueprint that encapsulates data (instance variables/attributes) and behavior (methods) into a single unit. An **object** is an instance of a class — it occupies memory and has a concrete state.

**Syntax:**
```ruby
class ClassName
  # members
end
```

**Key differences from other languages:**
- Everything in Ruby is an object (even integers, `nil`, `true`, `false`)
- Classes themselves are objects (instances of `Class`)
- No separate `struct` keyword — use `Struct` class or plain classes
- Instance variables are ALWAYS private; access is through methods only
- Ruby uses `@` for instance variables, `@@` for class variables, and `$` for globals

**Complete Example:**
```ruby
class Employee
  # Class variable — shared across all instances
  @@total_employees = 0

  # attr_reader generates getter methods
  # attr_writer generates setter methods
  # attr_accessor generates both
  attr_reader :name, :id
  attr_reader :department  # accessible but not writable from outside

  def initialize(name, id, salary, department)
    @name = name
    @id = id
    @salary = salary          # no getter/setter — truly private
    @department = department
    @@total_employees += 1
  end

  # Public method — part of the interface
  def salary
    @salary
  end

  def give_raise(percent)
    if percent > 0 && percent <= 50
      @salary += @salary * percent / 100.0
    end
  end

  # Class method (like static in other languages)
  def self.total_employees
    @@total_employees
  end

  # Protected method — accessible in same class and subclasses
  protected

  def department_internal
    @department
  end

  # Private method — only accessible within the instance itself
  private

  def calculate_bonus
    @salary * 0.1
  end
end
```

**Access Control in Detail:**

| Specifier | Within Class | Subclass | Outside Class | Use Case |
|-----------|:---:|:---:|:---:|---|
| `public` | ✓ | ✓ | ✓ | Interface methods, public API |
| `protected` | ✓ | ✓ | ✗ | Methods needed by subclasses or same-class instances |
| `private` | ✓ | ✓* | ✗ | Implementation details |

*In Ruby, private methods ARE inherited by subclasses (unlike C++). The restriction is that private methods cannot be called with an explicit receiver (not even `self`, prior to Ruby 2.7).

**Multiple access control styles:**
```ruby
class MyClass
  # Style 1: Section-based (like C++)
  public

  def public_method1; end
  def public_method2; end

  private

  def helper_method; end

  protected

  def shared_method; end

  # Style 2: Inline declaration
  def another_public_method; end
  private :another_public_method  # make it private after definition

  # Style 3: Using symbols (Ruby 2.1+)
  private def inline_private
    # this method is private
  end
end
```

**Object Creation:**
```ruby
# Standard instantiation (all objects are heap-allocated in Ruby)
emp1 = Employee.new("Alice", 101, 75000, "Engineering")

# Ruby has no stack vs heap distinction for objects
# All objects are garbage collected automatically

# Using Struct for simple data holders (like C++ POD structs)
Point = Struct.new(:x, :y)
p = Point.new(3, 4)
puts p.x  # 3
```

**Object Introspection (Ruby's reflection capabilities):**
```ruby
emp = Employee.new("Alice", 101, 75000, "Engineering")

emp.class                    # => Employee
emp.is_a?(Employee)          # => true
emp.respond_to?(:give_raise) # => true
emp.instance_variables       # => [:@name, :@id, :@salary, :@department]
emp.methods                  # => list of all public methods
emp.private_methods          # => list of private methods
emp.object_id                # => unique integer identifier
```

---

### Constructors (initialize)

Ruby uses the `initialize` method as the constructor. It is called automatically by `Class.new`.

#### Default Constructor

```ruby
class Widget
  def initialize
    @value = 0
    @name = "unnamed"
    puts "Default constructor called"
  end
end

w = Widget.new  # initialize called automatically
```

**If you define no `initialize`:** Ruby provides an empty default one (inherited from `Object`). Instance variables simply don't exist until assigned.

```ruby
class NoExplicitInit
  def show
    puts @x.inspect  # nil — uninitialized instance variables are nil, not garbage!
  end
end

NoExplicitInit.new.show  # => nil
```

**Key difference from C++:** In Ruby, uninitialized instance variables are always `nil` (never garbage values).

---

#### Parameterized Constructor

```ruby
class Rectangle
  attr_reader :width, :height

  def initialize(width, height)
    raise ArgumentError, "Dimensions must be positive" if width <= 0 || height <= 0
    @width = width
    @height = height
  end

  def area
    @width * @height
  end
end

r = Rectangle.new(5.0, 3.0)
```

---

#### Default Parameter Values (replacing overloaded constructors)

Ruby doesn't support method overloading. Instead, use default parameters, keyword arguments, or variable arguments.

```ruby
class Connection
  attr_reader :host, :port, :secure

  # Default parameters replace multiple constructors
  def initialize(host = "localhost", port = 80, secure: false)
    @host = host
    @port = port
    @secure = secure
  end
end

# Various ways to instantiate
Connection.new                              # localhost:80, not secure
Connection.new("api.example.com")           # api.example.com:80, not secure
Connection.new("api.example.com", 443, secure: true)  # full specification
```

**Keyword arguments (Ruby 2.0+) — named parameters:**
```ruby
class Connection
  def initialize(host:, port: 80, secure: false)
    @host = host
    @port = port
    @secure = secure
  end
end

Connection.new(host: "api.example.com", secure: true)
```

---

#### Copy Constructor Equivalent — `dup` and `clone`

Ruby doesn't have explicit copy constructors. Instead, it provides `dup` and `clone` methods.

```ruby
class DynamicArray
  attr_reader :data

  def initialize(size_or_data = 0)
    if size_or_data.is_a?(Array)
      @data = size_or_data.dup  # deep copy the array
    else
      @data = Array.new(size_or_data, 0)
    end
  end

  # Called by dup/clone to customize copying behavior
  def initialize_copy(original)
    @data = original.data.dup  # DEEP copy — each instance gets its own array
  end

  def [](index)
    @data[index]
  end

  def []=(index, value)
    @data[index] = value
  end
end

a = DynamicArray.new(5)
a[0] = 42

b = a.dup        # calls initialize_copy — deep copy
b[0] = 99
puts a[0]        # 42 — not affected (deep copy worked)
```

**`dup` vs `clone`:**
```ruby
# dup:   copies instance variables, does NOT copy frozen state or singleton methods
# clone: copies instance variables, DOES copy frozen state and singleton methods

obj = "hello"
obj.freeze

duped = obj.dup
duped.frozen?    # => false (dup doesn't preserve frozen state)

cloned = obj.clone
cloned.frozen?   # => true (clone preserves frozen state)
```

**Shallow vs Deep Copy:**
```
SHALLOW COPY (default dup/clone without initialize_copy):
┌─────────┐     ┌─────────────────┐
│ obj1    │────→│ [1, 2, 3, 4, 5] │
└─────────┘     └─────────────────┘
                        ↑
┌─────────┐             │
│ obj2    │─────────────┘  (SAME array object!)
└─────────┘

DEEP COPY (with initialize_copy or Marshal):
┌─────────┐     ┌─────────────────┐
│ obj1    │────→│ [1, 2, 3, 4, 5] │
└─────────┘     └─────────────────┘

┌─────────┐     ┌─────────────────┐
│ obj2    │────→│ [1, 2, 3, 4, 5] │  (separate array, same values)
└─────────┘     └─────────────────┘
```

**Deep copy using Marshal (Ruby's serialization):**
```ruby
# Quick and dirty deep copy (works for most objects)
original = { a: [1, 2, 3], b: { nested: "value" } }
deep_copy = Marshal.load(Marshal.dump(original))
# Limitation: doesn't work with Procs, IO objects, singleton methods
```

---

#### Freezing Objects (Immutability)

Ruby's equivalent of making objects immutable after construction.

```ruby
class Config
  attr_reader :host, :port, :use_tls

  def initialize(host, port, use_tls)
    @host = host
    @port = port
    @use_tls = use_tls
    freeze  # make this object immutable after construction!
  end
end

config = Config.new("api.example.com", 443, true)
# config.instance_variable_set(:@port, 8080)  # RuntimeError: can't modify frozen Config
```

---

### Destructor Equivalent — Finalizers

Ruby uses garbage collection, so there's no deterministic destructor. However, you can register finalizers for cleanup.

```ruby
class DatabaseConnection
  def initialize(conn_string)
    @conn_string = conn_string
    @conn = open_connection(conn_string)
    puts "Connected to #{conn_string}"

    # Register a destructor-like callback (weak reference via object_id)
    ObjectSpace.define_finalizer(self, self.class.destructor(@conn_string))
  end

  # Must be a class method or Proc that doesn't reference self
  # (otherwise it prevents garbage collection!)
  def self.destructor(conn_string)
    proc { puts "Disconnected from #{conn_string}" }
  end

  def close
    # Explicit cleanup is preferred over finalizers
    if @conn
      close_connection(@conn)
      @conn = nil
    end
  end

  private

  def open_connection(cs)
    # ... open connection ...
    Object.new  # placeholder
  end

  def close_connection(conn)
    # ... close connection ...
  end
end
```

**Best practice:** Don't rely on finalizers. Use explicit `close` methods and ensure blocks:
```ruby
# Ruby idiom: block-based resource management (like RAII)
class DatabaseConnection
  def self.open(conn_string)
    conn = new(conn_string)
    if block_given?
      begin
        yield conn
      ensure
        conn.close  # always called, even if exception raised
      end
    else
      conn
    end
  end
end

# Usage — guaranteed cleanup
DatabaseConnection.open("localhost") do |db|
  db.query("SELECT * FROM users")
end  # automatically closed here
```

---

### The `self` Keyword (equivalent of `this`)

In Ruby, `self` refers to the current object (receiver of the method call).

```ruby
class Node
  attr_accessor :value, :next_node

  def initialize(value)
    @value = value
    @next_node = nil
  end

  # 1. Disambiguate — though in Ruby, @var already refers to instance variable
  def value=(value)
    @value = value  # @value is the instance variable, value is the parameter
    # self.value = value  # This would cause infinite recursion! (calls this setter again)
  end

  # 2. Return self for method chaining (fluent interface)
  def set_next(node)
    @next_node = node
    self  # return self to enable chaining
  end

  # 3. Compare with another object
  def same_as?(other)
    self.equal?(other)  # compare object identity (same object_id)
  end

  # 4. self in class context defines class methods
  def self.create_chain(*values)
    nodes = values.map { |v| Node.new(v) }
    nodes.each_cons(2) { |a, b| a.set_next(b) }
    nodes.first
  end
end

# Method chaining in action
n1 = Node.new(1)
n2 = Node.new(2)
n3 = Node.new(3)
n1.set_next(n2).value = 10  # chained call
```

**Where `self` matters in Ruby:**
```ruby
class Example
  attr_accessor :name

  def demonstrate
    # Inside instance method: self is the instance
    puts self.class  # => Example
    puts self        # => #<Example:0x...>

    # Must use self for setter calls (otherwise Ruby thinks it's local variable)
    self.name = "hello"  # calls the setter method
    name = "hello"       # creates a LOCAL VARIABLE (doesn't call setter!)
  end

  # At class level: self is the class itself
  def self.class_method
    puts self  # => Example (the class object)
  end
end
```

---

### Class Variables and Class Methods (Static Equivalent)

#### Class Variables (`@@`)

```ruby
class Logger
  @@instance_count = 0
  @@logs = []

  attr_reader :name

  def initialize(name)
    @name = name
    @@instance_count += 1
  end

  def log(msg)
    @@logs << "[#{@name}] #{msg}"
  end

  # Class methods — called on the class, not instances
  def self.instance_count
    @@instance_count
  end

  def self.logs
    @@logs.dup  # return a copy to prevent external modification
  end

  def self.clear_logs
    @@logs.clear
  end
end

l1 = Logger.new("App")
l2 = Logger.new("DB")
l1.log("Started")
l2.log("Connected")
puts Logger.instance_count  # 2
puts Logger.logs.size       # 2
```

**WARNING about `@@` class variables:**
```ruby
class Animal
  @@count = 0
  def self.count; @@count; end
end

class Dog < Animal
  @@count = 0  # This SHARES the same @@count with Animal!
end

# @@class variables are shared across the ENTIRE inheritance hierarchy!
# This is a common source of bugs. Prefer class instance variables instead.
```

**Class Instance Variables (`@` at class level) — safer alternative:**
```ruby
class Animal
  @count = 0  # class instance variable — belongs to Animal class object only

  class << self
    attr_accessor :count
  end

  def initialize
    self.class.count += 1
  end
end

class Dog < Animal
  @count = 0  # separate counter for Dog — not shared with Animal!
end

Animal.new
Dog.new
Dog.new
puts Animal.count  # 1
puts Dog.count     # 2 (independent!)
```

#### Class Methods (Static Methods)

```ruby
class MathUtils
  PI = 3.14159265358979  # constant

  def self.factorial(n)
    return 1 if n <= 1
    n * factorial(n - 1)
  end

  def self.prime?(n)
    return false if n < 2
    (2..Math.sqrt(n).to_i).none? { |i| n % i == 0 }
  end

  # Alternative syntax using class << self block
  class << self
    def fibonacci(n)
      return n if n <= 1
      fibonacci(n - 1) + fibonacci(n - 2)
    end
  end
end

area = MathUtils::PI * r ** 2
fact5 = MathUtils.factorial(5)
```

**Class method restrictions:**
- Cannot access instance variables (`@var`) — no instance context
- Cannot call instance methods directly
- CAN access class variables (`@@var`) and class instance variables
- CAN call other class methods

**Common use cases for class methods:**
- Factory methods (`User.create_from_hash(data)`)
- Utility/helper functions
- Singleton access (`Config.instance`)
- Object counters and registries

---

### Frozen and Const — Immutability in Ruby

```ruby
class Matrix
  attr_reader :rows, :cols

  def initialize(rows, cols)
    @rows = rows
    @cols = cols
    @data = Array.new(rows) { Array.new(cols, 0.0) }
  end

  def get(r, c)
    @data[r][c]
  end

  def set(r, c, val)
    raise "Matrix is frozen" if frozen?
    @data[r][c] = val
  end

  def square?
    @rows == @cols
  end
end

m = Matrix.new(3, 3)
m.set(0, 0, 5.0)
m.freeze
# m.set(1, 1, 3.0)  # RuntimeError: Matrix is frozen
puts m.get(0, 0)     # OK: reading is fine
```

**Constants in Ruby:**
```ruby
class Config
  MAX_RETRIES = 3          # constant (uppercase)
  DEFAULT_TIMEOUT = 30

  # Constants CAN be reassigned (with a warning)
  # MAX_RETRIES = 5  # warning: already initialized constant

  # To truly prevent reassignment, freeze the value
  API_URL = "https://api.example.com".freeze
end

puts Config::MAX_RETRIES  # 3
```

---

### Mutable Keyword Equivalent — Instance Variable Mutability

Ruby doesn't have a `mutable` keyword. Since instance variables are always accessible within the class, you can modify them even in methods that are conceptually "read-only." The convention is to use `freeze` for true immutability.

```ruby
class CachedComputation
  def initialize(data)
    @data = data.dup
    @cache_valid = false
    @cached_result = nil
    @mutex = Mutex.new
  end

  def data=(new_data)
    @data = new_data.dup
    @cache_valid = false
  end

  # Conceptually a "read" operation, but modifies cache internally
  def compute_average
    @mutex.synchronize do
      unless @cache_valid
        sum = @data.sum
        @cached_result = sum.to_f / @data.size
        @cache_valid = true
      end
      @cached_result
    end
  end
end
```

**Ruby's approach:** There's no language-level distinction between "logically const" and "physically const." The convention is documentation and method naming (query methods end with `?`, bang methods end with `!`).

---

### Friend Equivalent — Access Control Workarounds

Ruby doesn't have a `friend` keyword. Instead, it offers several mechanisms for controlled access:

#### Using `send` to bypass access control (testing/debugging)

```ruby
class Secret
  private

  def hidden_method
    "secret value"
  end
end

obj = Secret.new
# obj.hidden_method        # NoMethodError: private method called
obj.send(:hidden_method)   # => "secret value" (bypasses access control)
```

#### Using `protected` for same-class instance access

```ruby
class Vector3D
  def initialize(x, y, z)
    @x = x
    @y = y
    @z = z
  end

  def dot_product(other)
    # Can call protected methods on other instances of same class
    @x * other.x + @y * other.y + @z * other.z
  end

  def to_s
    "(#{@x}, #{@y}, #{@z})"
  end

  protected

  # Protected: accessible from instances of the same class (or subclasses)
  def x; @x; end
  def y; @y; end
  def z; @z; end
end

a = Vector3D.new(1, 2, 3)
b = Vector3D.new(4, 5, 6)
puts a.dot_product(b)  # 32
# puts a.x             # NoMethodError: protected method 'x' called
```

#### Using modules for shared access patterns

```ruby
class LinkedList
  Node = Struct.new(:data, :next_node)  # accessible to friends via nesting

  attr_reader :head

  def initialize
    @head = nil
  end

  def push_front(val)
    @head = Node.new(val, @head)
  end
end

class LinkedListIterator
  def initialize(list)
    @current = list.head  # accesses public reader
  end

  def has_next?
    !@current.nil?
  end

  def next_value
    val = @current.data
    @current = @current.next_node
    val
  end
end
```

**Properties of Ruby's access control:**
| Property | Explanation |
|----------|-------------|
| Private is inherited | Subclasses inherit private methods (can call them without explicit receiver) |
| Protected allows same-class access | Protected methods can be called on other instances of the same class |
| `send` bypasses all | Any method can be called via `send` (useful for testing) |
| Access is per-method | Each method individually declared public/protected/private |

---


## 2.2 Encapsulation

> Encapsulation is the first pillar of OOP. It means bundling data and the methods that operate on that data into a single unit (class), and restricting direct access to the internal state. The goal is to protect the integrity of the object's data and hide implementation details.

---

### Data Hiding

**Principle:** The internal representation of an object should be hidden from the outside. Only a well-defined interface (public methods) should be exposed.

**Why data hiding matters:**

```ruby
# BAD: No encapsulation — using Struct with all public access
BankAccountBad = Struct.new(:balance, :owner)

acc = BankAccountBad.new(1000, "Alice")
acc.balance = -1000000  # No validation! Invalid state!
acc.balance = acc.balance * 2  # Anyone can manipulate directly

# GOOD: Proper encapsulation
class BankAccount
  attr_reader :owner, :balance

  def initialize(owner, account_number, initial_deposit = 0)
    @owner = owner
    @account_number = account_number  # no reader — truly hidden
    @balance = 0
    @history = []
    deposit(initial_deposit) if initial_deposit > 0
  end

  def deposit(amount)
    return false if amount <= 0
    @balance += amount
    record_transaction("DEPOSIT", amount)
    true
  end

  def withdraw(amount)
    return false if amount <= 0
    return false if amount > @balance  # insufficient funds
    @balance -= amount
    record_transaction("WITHDRAWAL", amount)
    true
  end

  def transfer(to_account, amount)
    if withdraw(amount)
      to_account.deposit(amount)
      true
    else
      false
    end
  end

  private

  def record_transaction(type, amount)
    @history << { type: type, amount: amount, time: Time.now }
  end
end
```

**Benefits of data hiding:**
1. **Validation:** All modifications go through methods that validate input
2. **Consistency:** Object is always in a valid state
3. **Flexibility:** Internal representation can change without breaking client code
4. **Debugging:** All state changes go through known entry points
5. **Security:** Sensitive data cannot be accessed directly

**Implementation change without breaking clients:**
```ruby
# Version 1: balance stored as Float
class AccountV1
  def balance
    @balance  # stored as dollars (Float)
  end
end

# Version 2: balance stored as Integer (cents) for precision
class AccountV2
  def balance
    @balance_cents / 100.0  # same interface! Clients don't know the difference
  end
end
```

**Ruby's unique approach to encapsulation:**
```ruby
# In Ruby, instance variables are ALWAYS private
# There is NO way to access @var from outside without a method

class Example
  def initialize
    @secret = 42
  end
end

obj = Example.new
# obj.@secret          # SyntaxError
# obj.secret           # NoMethodError (no getter defined)
obj.instance_variable_get(:@secret)  # 42 — reflection bypass (avoid in production!)
```

---

### Getters and Setters (Accessors)

Ruby provides `attr_reader`, `attr_writer`, and `attr_accessor` to generate accessor methods. But you can also write custom ones with validation.

```ruby
class Person
  attr_reader :first_name, :last_name

  def initialize(first, last, age, email)
    @first_name = first
    @last_name = last
    self.age = age      # use setter for validation
    self.email = email  # use setter for validation
  end

  # --- Custom Getters ---
  def full_name
    "#{@first_name} #{@last_name}"  # computed property
  end

  def age
    @age
  end

  def email
    @email
  end

  # --- Custom Setters with validation ---
  def age=(new_age)
    validate_age(new_age)
    @age = new_age
  end

  def email=(new_email)
    validate_email(new_email)
    @email = new_email
  end

  def change_name(first, last)
    raise ArgumentError, "Name cannot be empty" if first.empty? || last.empty?
    @first_name = first
    @last_name = last
  end

  private

  def validate_age(a)
    raise ArgumentError, "Invalid age: #{a}" unless (0..150).include?(a)
  end

  def validate_email(e)
    raise ArgumentError, "Invalid email: #{e}" unless e.include?("@")
  end
end
```

**How `attr_accessor` works internally:**
```ruby
# attr_reader :name is equivalent to:
def name
  @name
end

# attr_writer :name is equivalent to:
def name=(value)
  @name = value
end

# attr_accessor :name generates BOTH
```

**When NOT to use `attr_accessor`:**
```ruby
# BAD: Anemic class with accessors for everything
class RectangleBad
  attr_accessor :width, :height
end
# Client does: rect.width = rect.width * 2  # logic is OUTSIDE the class

# GOOD: Behavior-rich class
class Rectangle
  attr_reader :width, :height

  def initialize(width, height)
    raise ArgumentError, "Dimensions must be positive" if width <= 0 || height <= 0
    @width = width
    @height = height
  end

  def area
    @width * @height
  end

  def perimeter
    2 * (@width + @height)
  end

  def scale(factor)
    raise ArgumentError, "Factor must be positive" if factor <= 0
    @width *= factor
    @height *= factor
    self
  end

  def resize(w, h)
    raise ArgumentError, "Dimensions must be positive" if w <= 0 || h <= 0
    @width = w
    @height = h
    self
  end
end
```

---

### Invariant Enforcement

A **class invariant** is a logical condition that must be true for every object of the class, at all times between method calls. The constructor establishes the invariant, and every public method must preserve it.

```ruby
class SortedArray
  include Enumerable

  def initialize
    @data = []  # INVARIANT: @data is always sorted in ascending order
  end

  def insert(value)
    # Insert in correct position to maintain sorted invariant
    index = @data.bsearch_index { |x| x >= value } || @data.size
    @data.insert(index, value)
    # Invariant preserved: data is still sorted ✓
    self
  end

  def contains?(value)
    # Can use binary search because invariant guarantees sorted order
    !!@data.bsearch { |x| value <=> x }
  end

  def remove(value)
    index = @data.bsearch_index { |x| x >= value }
    if index && @data[index] == value
      @data.delete_at(index)
    end
    # Invariant preserved: removing from sorted array keeps it sorted ✓
    self
  end

  def min
    raise "Empty array" if @data.empty?
    @data.first  # guaranteed minimum due to invariant
  end

  def max
    raise "Empty array" if @data.empty?
    @data.last   # guaranteed maximum due to invariant
  end

  def each(&block)
    @data.each(&block)
  end

  def size
    @data.size
  end
end
```

**Another example — Circular Buffer with invariants:**
```ruby
class CircularBuffer
  # INVARIANTS:
  # 1. 0 <= @count <= @capacity
  # 2. 0 <= @head < @capacity
  # 3. 0 <= @tail < @capacity
  # 4. @tail == (@head + @count) % @capacity

  def initialize(capacity)
    raise ArgumentError, "Capacity must be positive" if capacity <= 0
    @capacity = capacity
    @buffer = Array.new(capacity)
    @head = 0
    @tail = 0
    @count = 0
  end

  def push(value)
    raise "Buffer full" if full?
    @buffer[@tail] = value
    @tail = (@tail + 1) % @capacity
    @count += 1
    self
  end

  def pop
    raise "Buffer empty" if empty?
    value = @buffer[@head]
    @head = (@head + 1) % @capacity
    @count -= 1
    value
  end

  def full?
    @count == @capacity
  end

  def empty?
    @count == 0
  end

  def size
    @count
  end
end
```

**Strategies for invariant enforcement:**
1. **Validate in constructor** — establish invariant from the start
2. **Validate in setters/mutators** — check before modifying
3. **Make invalid states unrepresentable** — use types/freezing that prevent violations
4. **Use assertions in development** — `raise` or custom assertion methods

---

### Immutable Objects in Ruby

An **immutable object** cannot be modified after construction. In Ruby, use `freeze` to enforce immutability.

```ruby
class ImmutableConfig
  attr_reader :host, :port, :use_tls, :timeout, :allowed_origins

  def initialize(host, port, use_tls, timeout, allowed_origins)
    @host = host.freeze
    @port = port
    @use_tls = use_tls
    @timeout = timeout
    @allowed_origins = allowed_origins.map(&:freeze).freeze
    freeze  # freeze the entire object!
  end

  # "Modification" returns a new object (functional style)
  def with_port(new_port)
    self.class.new(@host, new_port, @use_tls, @timeout, @allowed_origins)
  end

  def with_timeout(new_timeout)
    self.class.new(@host, @port, @use_tls, new_timeout, @allowed_origins)
  end
end

config = ImmutableConfig.new("api.example.com", 443, true, 30, ["*.example.com"])
dev_config = config.with_port(8080).with_timeout(60)  # new object, original unchanged
```

**Builder pattern for immutable objects:**
```ruby
class ImmutableUser
  attr_reader :name, :email, :age, :role

  # Private constructor — only Builder can create
  private_class_method :new

  def initialize(name, email, age, role)
    @name = name.freeze
    @email = email.freeze
    @age = age
    @role = role.freeze
    freeze
  end

  class Builder
    def initialize
      @name = nil
      @email = nil
      @age = 0
      @role = "user"
    end

    def set_name(name)
      @name = name
      self
    end

    def set_email(email)
      @email = email
      self
    end

    def set_age(age)
      @age = age
      self
    end

    def set_role(role)
      @role = role
      self
    end

    def build
      raise ArgumentError, "Name and email are required" if @name.nil? || @email.nil?
      ImmutableUser.send(:new, @name, @email, @age, @role)
    end
  end
end

user = ImmutableUser::Builder.new
  .set_name("Alice")
  .set_email("alice@example.com")
  .set_age(30)
  .build
```

**Benefits of immutability:**
- **Thread-safe:** No synchronization needed (no writes)
- **Predictable:** State never changes unexpectedly
- **Safe sharing:** Can be freely shared between components
- **Hashable:** Can be used as hash keys (hash never changes)
- **Easy to reason about:** No temporal coupling

---


## 2.3 Inheritance

> Inheritance is the mechanism by which one class (child/subclass) acquires the properties and behaviors of another class (parent/superclass). It models "is-a" relationships and enables code reuse and polymorphism.

---

### Single Inheritance

Ruby supports ONLY single inheritance (one parent class). Multiple inheritance is achieved through Mixins (modules).

```ruby
class Vehicle
  attr_reader :make, :model, :year, :fuel_level

  def initialize(make, model, year)
    @make = make
    @model = model
    @year = year
    @fuel_level = 100.0
  end

  def start
    puts "#{@year} #{@make} #{@model} started."
  end

  def stop
    puts "#{@make} #{@model} stopped."
  end

  def refuel(amount)
    @fuel_level = [@fuel_level + amount, 100.0].min
  end

  def info
    "#{@year} #{@make} #{@model}"
  end
end

class Car < Vehicle
  attr_reader :num_doors, :convertible

  def initialize(make, model, year, doors, convertible)
    super(make, model, year)  # call parent constructor
    @num_doors = doors
    @convertible = convertible
  end

  def start
    print "Turning key... "
    super  # call parent's start method
  end

  def open_roof
    if @convertible
      puts "Roof opened!"
    else
      puts "This car is not a convertible."
    end
  end
end

class Truck < Vehicle
  attr_reader :payload_capacity

  def initialize(make, model, year, payload)
    super(make, model, year)
    @payload_capacity = payload
  end

  def start
    print "Diesel engine warming up... "
    super
  end

  def load_cargo(tons)
    if tons <= @payload_capacity
      puts "Loaded #{tons} tons."
    else
      puts "Exceeds capacity of #{@payload_capacity} tons!"
    end
  end
end
```

**What the child class gets:**
- All public and protected methods of the parent
- Does NOT get: private methods are inherited but can only be called without explicit receiver
- Instance variables are NOT inherited — they're created when assigned in `initialize`

**What the child class can do:**
- Add new methods and instance variables
- Override parent methods
- Call parent methods using `super`
- Change access control of inherited methods

---

### Mixins — Ruby's Answer to Multiple Inheritance

Since Ruby only allows single inheritance, **modules** (mixins) provide a way to share behavior across unrelated classes.

```ruby
# Modules define capabilities/behaviors
module Flyable
  def fly
    @altitude = 100
    puts "#{self.class} flying!"
  end

  def altitude
    @altitude || 0
  end
end

module Swimmable
  def swim
    @depth = 2
    puts "#{self.class} swimming!"
  end

  def depth
    @depth || 0
  end
end

module Walkable
  def walk
    @speed = 3
    puts "#{self.class} walking!"
  end

  def speed
    @speed || 0
  end
end

# Duck can fly, swim, and walk — include multiple modules
class Duck
  include Flyable
  include Swimmable
  include Walkable

  def initialize(name)
    @name = name
  end

  def to_s
    @name
  end
end

donald = Duck.new("Donald")
donald.fly    # Duck flying!
donald.swim   # Duck swimming!
donald.walk   # Duck walking!

# Check capabilities
donald.is_a?(Flyable)    # => true
donald.is_a?(Swimmable)  # => true
Duck.ancestors           # => [Duck, Walkable, Swimmable, Flyable, Object, Kernel, BasicObject]
```

**`include` vs `extend` vs `prepend`:**
```ruby
module Greetable
  def greet
    "Hello, I'm #{name}"
  end
end

class Person
  include Greetable   # adds as INSTANCE methods
  attr_reader :name

  def initialize(name)
    @name = name
  end
end

Person.new("Alice").greet  # => "Hello, I'm Alice"

class Robot
  extend Greetable    # adds as CLASS methods

  def self.name
    "Robot"
  end
end

Robot.greet  # => "Hello, I'm Robot" (called on class, not instance)

# prepend: inserts module BEFORE the class in method lookup
module Logging
  def process
    puts "Before process"
    super  # calls the original class method
    puts "After process"
  end
end

class Worker
  prepend Logging  # Logging methods take priority over Worker methods

  def process
    puts "Working..."
  end
end

Worker.new.process
# Output:
# Before process
# Working...
# After process
```

---

### Method Lookup Path (Method Resolution Order)

Ruby searches for methods in a specific order. Understanding this is critical for debugging.

```ruby
module M1
  def who_am_i
    "M1"
  end
end

module M2
  def who_am_i
    "M2"
  end
end

class Parent
  def who_am_i
    "Parent"
  end
end

class Child < Parent
  include M1
  include M2  # last included wins (for same-named methods)

  def who_am_i
    "Child"
  end
end

Child.ancestors
# => [Child, M2, M1, Parent, Object, Kernel, BasicObject]

# Method lookup order:
# 1. Child (the class itself)
# 2. M2 (last included module)
# 3. M1 (first included module)
# 4. Parent (superclass)
# 5. Object (default parent)
# 6. Kernel (module included in Object)
# 7. BasicObject (root of hierarchy)
```

**With `prepend`:**
```ruby
class Child < Parent
  prepend M1  # prepend goes BEFORE the class
  include M2  # include goes AFTER the class

  def who_am_i
    "Child"
  end
end

Child.ancestors
# => [M1, Child, M2, Parent, Object, Kernel, BasicObject]
# M1 is checked FIRST (before Child itself)
```

---

### Multilevel Inheritance

A chain where each class inherits from the one above it.

```ruby
class Shape
  attr_reader :color

  def initialize(color = "black")
    @color = color
  end

  def area
    raise NotImplementedError, "#{self.class} must implement #area"
  end

  def describe
    "Shape [color=#{@color}]"
  end
end

class Polygon < Shape
  attr_reader :num_sides

  def initialize(sides, color = "black")
    super(color)
    @num_sides = sides
  end

  def describe
    "Polygon [sides=#{@num_sides}, color=#{@color}]"
  end
end

class RegularPolygon < Polygon
  attr_reader :side_length

  def initialize(sides, length, color = "black")
    super(sides, color)
    @side_length = length
  end

  def perimeter
    @num_sides * @side_length
  end

  def describe
    "RegularPolygon [sides=#{@num_sides}, length=#{@side_length}, color=#{@color}]"
  end
end

class Square < RegularPolygon
  def initialize(side, color = "black")
    super(4, side, color)
  end

  def area
    @side_length ** 2
  end

  def describe
    "Square [side=#{@side_length}, color=#{@color}]"
  end
end

# Inheritance chain: Shape → Polygon → RegularPolygon → Square
```

**Depth considerations:**
- Deep hierarchies (>3-4 levels) become hard to understand and maintain
- Prefer composition and mixins over deep inheritance chains
- Each level should add meaningful specialization

---

### Hierarchical Inheritance

Multiple child classes inherit from the same parent class (tree structure).

```ruby
class Employee
  attr_reader :name, :id

  def initialize(name, id, base_salary = 0)
    @name = name
    @id = id
    @base_salary = base_salary
  end

  def calculate_pay
    raise NotImplementedError, "#{self.class} must implement #calculate_pay"
  end

  def role
    raise NotImplementedError, "#{self.class} must implement #role"
  end

  def display_info
    puts "#{role}: #{@name} (ID: #{@id})"
    puts "  Pay: $#{'%.2f' % calculate_pay}"
  end
end

class FullTimeEmployee < Employee
  def initialize(name, id, salary, bonus)
    super(name, id, salary)
    @bonus = bonus
  end

  def calculate_pay
    @base_salary + @bonus
  end

  def role
    "Full-Time"
  end
end

class PartTimeEmployee < Employee
  def initialize(name, id, hours_worked, hourly_rate)
    super(name, id)
    @hours_worked = hours_worked
    @hourly_rate = hourly_rate
  end

  def calculate_pay
    @hours_worked * @hourly_rate
  end

  def role
    "Part-Time"
  end
end

class Contractor < Employee
  def initialize(name, id, project_days, daily_rate)
    super(name, id)
    @project_days = project_days
    @daily_rate = daily_rate
  end

  def calculate_pay
    @project_days * @daily_rate
  end

  def role
    "Contractor"
  end
end

# Polymorphic usage
def process_payroll(employees)
  total = 0
  employees.each do |emp|
    emp.display_info
    total += emp.calculate_pay
  end
  puts "Total payroll: $#{'%.2f' % total}"
end

employees = [
  FullTimeEmployee.new("Alice", 1, 80000, 10000),
  PartTimeEmployee.new("Bob", 2, 80, 25),
  Contractor.new("Charlie", 3, 20, 500)
]
process_payroll(employees)
```

---

### The Diamond Problem — Not an Issue in Ruby

Ruby avoids the diamond problem entirely because it only supports single inheritance. Modules are linearized in the ancestor chain.

```ruby
module Animal
  def describe
    "I am an animal"
  end
end

module Mammal
  include Animal

  def nurse
    puts "Nursing young"
  end
end

module Bird
  include Animal

  def lay_eggs
    puts "Laying eggs"
  end
end

class Bat
  include Mammal
  include Bird  # Animal is included only ONCE in the ancestor chain

  def describe
    "I am a bat"
  end
end

Bat.ancestors
# => [Bat, Bird, Animal, Mammal, Object, Kernel, BasicObject]
# Note: Animal appears only ONCE! Ruby's linearization prevents duplication.

bat = Bat.new
bat.nurse      # from Mammal
bat.lay_eggs   # from Bird
```

**How Ruby resolves the diamond:**
- Ruby uses C3 linearization to create a single, unambiguous method lookup path
- Each module/class appears only ONCE in the ancestor chain
- Last `include` wins for same-named methods (unless using `prepend`)
- No ambiguity, no duplicate state — the problem simply doesn't exist

---

### Constructor Call Order in Inheritance

```ruby
class Base
  def initialize
    puts "  Base initialize"
    @base_member = "base_m"
  end
end

class Middle < Base
  def initialize
    super  # MUST explicitly call super (not automatic like C++)
    puts "  Middle initialize"
    @middle_member = "mid_m"
  end
end

class Derived < Middle
  def initialize
    super  # calls Middle#initialize, which calls Base#initialize
    puts "  Derived initialize"
    @derived_member1 = "d_m1"
    @derived_member2 = "d_m2"
  end
end

# Creating: Derived.new
# Output:
#   Base initialize
#   Middle initialize
#   Derived initialize
```

**Key difference from C++:** In Ruby, you MUST explicitly call `super` to invoke the parent constructor. It is NOT automatic. If you don't call `super`, the parent's `initialize` is never called.

```ruby
class Parent
  def initialize
    @important_setup = true
    puts "Parent setup done"
  end
end

class Child < Parent
  def initialize
    # Forgot to call super!
    @child_data = "hello"
  end
end

c = Child.new
c.instance_variable_get(:@important_setup)  # => nil (parent never initialized!)
```

**`super` variations:**
```ruby
class Parent
  def initialize(x, y)
    @x = x
    @y = y
  end
end

class Child < Parent
  def initialize(x, y, z)
    super(x, y)    # passes x, y to parent
    @z = z
  end
end

class Child2 < Parent
  def initialize(x, y, z)
    super          # passes ALL arguments (x, y, z) to parent — may cause ArgumentError!
    @z = z
  end
end

class Child3 < Parent
  def initialize(x, y, z)
    super()        # passes NO arguments to parent — may cause ArgumentError!
    @z = z
  end
end
```

---

### Access Control in Inheritance

```ruby
class Base
  def public_method
    "public"
  end

  protected

  def protected_method
    "protected"
  end

  private

  def private_method
    "private"
  end
end

class Child < Base
  def test_access
    puts public_method      # OK: inherited, still public
    puts protected_method   # OK: inherited, accessible in subclass
    puts private_method     # OK: inherited, accessible without explicit receiver
    # puts self.private_method  # ERROR in Ruby < 2.7: private can't use explicit receiver
  end

  # Can change access control of inherited methods
  public :protected_method  # make parent's protected method public in Child
end

child = Child.new
child.public_method       # OK
child.protected_method    # OK (we made it public in Child)
# child.private_method    # NoMethodError: private method
```

**Ruby's access control is different from C++:**
- There's no `public`/`protected`/`private` inheritance specifier
- All inheritance is effectively "public" — the parent's public methods remain public
- You CAN change access control in the child class using `public`, `protected`, `private` with method names

---

### Type Checking and Duck Typing

Ruby is dynamically typed and uses **duck typing** — "If it walks like a duck and quacks like a duck, it's a duck."

#### Upcasting Equivalent — Polymorphism via Duck Typing

```ruby
class Shape
  def draw
    raise NotImplementedError
  end
end

class Circle < Shape
  def initialize(radius)
    @radius = radius
  end

  def draw
    puts "Drawing circle r=#{@radius}"
  end

  def radius
    @radius
  end
end

class Square < Shape
  def initialize(side)
    @side = side
  end

  def draw
    puts "Drawing square s=#{@side}"
  end

  def side
    @side
  end
end

# No explicit upcasting needed — Ruby uses duck typing
shapes = [Circle.new(5), Square.new(4)]
shapes.each { |s| s.draw }  # correct version called for each

# You don't even need a common base class!
class Triangle
  def draw
    puts "Drawing triangle"
  end
end

shapes << Triangle.new  # works fine — it responds to #draw
shapes.each { |s| s.draw }
```

**No object slicing in Ruby:** Since Ruby always uses references (never value copies for method calls), there's no slicing problem.

#### Downcasting Equivalent — Type Checking

```ruby
def process_shape(shape)
  # Method 1: is_a? / kind_of? (checks class hierarchy)
  case shape
  when Circle
    puts "Circle with radius: #{shape.radius}"
  when Square
    puts "Square with side: #{shape.side}"
  else
    puts "Unknown shape type"
  end

  # Method 2: respond_to? (duck typing — preferred!)
  if shape.respond_to?(:radius)
    puts "Has radius: #{shape.radius}"
  end

  # Method 3: instance_of? (exact class, not subclasses)
  if shape.instance_of?(Circle)
    puts "Exactly a Circle (not a subclass)"
  end
end
```

**Type checking methods comparison:**
| Method | Checks | Example |
|--------|--------|---------|
| `is_a?` / `kind_of?` | Class + ancestors + modules | `obj.is_a?(Enumerable)` |
| `instance_of?` | Exact class only | `obj.instance_of?(Array)` |
| `respond_to?` | Has method? (duck typing) | `obj.respond_to?(:each)` |
| `===` (case equality) | Used in `case` statements | `Integer === 42` |

**Prefer duck typing over type checking:**
```ruby
# BAD: Checking types explicitly
def process(item)
  if item.is_a?(String)
    item.upcase
  elsif item.is_a?(Array)
    item.map(&:to_s)
  end
end

# GOOD: Duck typing — just call the method
def process(item)
  item.to_s  # works for any object that responds to to_s
end

# GOOD: Check capability, not type
def serialize(obj)
  if obj.respond_to?(:to_json)
    obj.to_json
  elsif obj.respond_to?(:to_s)
    obj.to_s
  end
end
```

---


## 2.4 Polymorphism

> Polymorphism ("many forms") is the ability of different objects to respond to the same message/interface in different ways. In Ruby, polymorphism is natural and pervasive due to duck typing — any object that responds to a method can be used interchangeably.

---

### Duck Typing — Ruby's Primary Polymorphism

In Ruby, polymorphism doesn't require inheritance or interfaces. If an object responds to the expected method, it works.

```ruby
# These classes have NO common ancestor (beyond Object), yet work polymorphically
class Dog
  def speak
    "Woof!"
  end
end

class Cat
  def speak
    "Meow!"
  end
end

class Duck
  def speak
    "Quack!"
  end
end

class Person
  def speak
    "Hello!"
  end
end

# Polymorphic usage — no base class needed!
animals = [Dog.new, Cat.new, Duck.new, Person.new]
animals.each { |a| puts a.speak }

# Any function that calls #speak works with ANY object that has #speak
def make_noise(speaker)
  puts speaker.speak
end

make_noise(Dog.new)     # works
make_noise(Person.new)  # works
make_noise("hello")     # NoMethodError — String doesn't have #speak
```

---

### Method Overriding (Runtime Polymorphism)

When a child class defines a method with the same name as the parent, the child's version is called.

```ruby
class Notification
  attr_reader :message, :recipient

  def initialize(message, recipient)
    @message = message
    @recipient = recipient
  end

  def send_notification
    puts "Generic notification to #{@recipient}: #{@message}"
  end

  # Non-overridable behavior (by convention — Ruby can't truly prevent it)
  def log
    puts "[LOG] Notification created for #{@recipient}"
  end
end

class EmailNotification < Notification
  attr_reader :subject

  def initialize(message, recipient, subject)
    super(message, recipient)
    @subject = subject
  end

  def send_notification
    puts "EMAIL to #{@recipient}"
    puts "  Subject: #{@subject}"
    puts "  Body: #{@message}"
  end
end

class SMSNotification < Notification
  def send_notification
    puts "SMS to #{@recipient}: #{@message}"
  end
end

class PushNotification < Notification
  def initialize(message, recipient, app_name)
    super(message, recipient)
    @app_name = app_name
  end

  def send_notification
    puts "PUSH [#{@app_name}] to #{@recipient}: #{@message}"
  end
end

# Polymorphic usage
def broadcast_alert(channels)
  channels.each do |channel|
    channel.send_notification  # correct version called based on actual type
  end
end

alerts = [
  EmailNotification.new("Server down!", "admin@co.com", "ALERT"),
  SMSNotification.new("Server down!", "+1234567890"),
  PushNotification.new("Server down!", "admin", "OpsApp")
]
broadcast_alert(alerts)
```

**Calling the parent version with `super`:**
```ruby
class Base
  def process(data)
    puts "Base processing: #{data}"
    data.upcase
  end
end

class Enhanced < Base
  def process(data)
    puts "Enhanced pre-processing"
    result = super  # calls Base#process with same arguments
    puts "Enhanced post-processing"
    "#{result}!!!"
  end
end

Enhanced.new.process("hello")
# Enhanced pre-processing
# Base processing: hello
# Enhanced post-processing
# => "HELLO!!!"
```

---

### Method Overloading — Not Supported (Alternatives)

Ruby does NOT support method overloading (multiple methods with same name, different parameters). The last definition wins.

```ruby
# BAD: This doesn't work like C++ overloading
class Calculator
  def add(a, b)
    a + b
  end

  def add(a, b, c)  # This REPLACES the previous add method!
    a + b + c
  end
end

calc = Calculator.new
# calc.add(1, 2)     # ArgumentError: wrong number of arguments (given 2, expected 3)
calc.add(1, 2, 3)    # 6
```

**Ruby alternatives to method overloading:**

```ruby
class Calculator
  # Alternative 1: Default parameters
  def add(a, b, c = 0)
    a + b + c
  end

  # Alternative 2: Variable arguments (splat)
  def sum(*numbers)
    numbers.reduce(0, :+)
  end

  # Alternative 3: Duck typing — works with any type that supports +
  def combine(a, b)
    a + b  # works for Integer, Float, String, Array...
  end

  # Alternative 4: Keyword arguments for clarity
  def calculate(operation:, values:)
    case operation
    when :add then values.sum
    when :multiply then values.reduce(1, :*)
    end
  end

  # Alternative 5: Check argument types (less idiomatic)
  def flexible_add(a, b)
    case [a.class, b.class]
    when [Integer, Integer] then a + b
    when [String, String] then "#{a}#{b}"
    when [Array, Array] then a + b
    else raise TypeError, "Cannot add #{a.class} and #{b.class}"
    end
  end
end

calc = Calculator.new
calc.sum(1, 2)           # 3
calc.sum(1, 2, 3, 4, 5) # 15
calc.combine(1, 2)       # 3
calc.combine("hi", " there")  # "hi there"
calc.combine([1, 2], [3, 4])  # [1, 2, 3, 4]
```

---

### Operator Overloading

Ruby allows operator overloading by defining methods with operator names. Operators are just syntactic sugar for method calls.

```ruby
class Fraction
  include Comparable

  attr_reader :numerator, :denominator

  def initialize(num, den = 1)
    raise ArgumentError, "Denominator cannot be zero" if den == 0
    # Simplify and normalize sign
    gcd = num.gcd(den)
    @numerator = num / gcd
    @denominator = den / gcd
    if @denominator < 0
      @numerator = -@numerator
      @denominator = -@denominator
    end
  end

  # --- Arithmetic operators ---
  def +(other)
    other = coerce_to_fraction(other)
    Fraction.new(
      @numerator * other.denominator + other.numerator * @denominator,
      @denominator * other.denominator
    )
  end

  def -(other)
    other = coerce_to_fraction(other)
    Fraction.new(
      @numerator * other.denominator - other.numerator * @denominator,
      @denominator * other.denominator
    )
  end

  def *(other)
    other = coerce_to_fraction(other)
    Fraction.new(@numerator * other.numerator, @denominator * other.denominator)
  end

  def /(other)
    other = coerce_to_fraction(other)
    raise ZeroDivisionError, "Division by zero" if other.numerator == 0
    Fraction.new(@numerator * other.denominator, @denominator * other.numerator)
  end

  # --- Unary operators ---
  def -@
    Fraction.new(-@numerator, @denominator)
  end

  def +@
    self
  end

  # --- Comparison (via Comparable mixin) ---
  def <=>(other)
    other = coerce_to_fraction(other)
    (@numerator * other.denominator) <=> (other.numerator * @denominator)
  end

  # --- Equality ---
  def ==(other)
    return false unless other.respond_to?(:numerator) && other.respond_to?(:denominator)
    @numerator == other.numerator && @denominator == other.denominator
  end

  # --- Conversion ---
  def to_f
    @numerator.to_f / @denominator
  end

  def to_s
    @denominator == 1 ? @numerator.to_s : "#{@numerator}/#{@denominator}"
  end

  def inspect
    "Fraction(#{to_s})"
  end

  # --- Coercion (allows 2 + Fraction.new(1, 3) to work) ---
  def coerce(other)
    [Fraction.new(other), self]
  end

  private

  def coerce_to_fraction(other)
    case other
    when Fraction then other
    when Integer then Fraction.new(other)
    else raise TypeError, "Cannot coerce #{other.class} to Fraction"
    end
  end
end

# Usage
a = Fraction.new(1, 2)
b = Fraction.new(1, 3)
puts a + b        # 5/6
puts a * b        # 1/6
puts a > b        # true
puts a.to_f       # 0.5
puts 2 + a        # 5/2 (works because of #coerce)
```

**Operators that CAN be overloaded in Ruby:**
```ruby
# Arithmetic: +  -  *  /  %  **
# Comparison: ==  <=>  <  >  <=  >=
# Bitwise:    &  |  ^  ~  <<  >>
# Unary:      +@  -@
# Array:      []  []=
# Conversion: to_s  to_i  to_f  to_a  to_h
# Equality:   ==  eql?  equal?
# Spaceship:  <=> (used by Comparable mixin for all comparisons)
```

**Operators that CANNOT be overloaded:**
- `=` (assignment)
- `..` and `...` (range)
- `not`, `and`, `or` (logical — but `!`, `&&`, `||` can't be overloaded either)
- `?:` (ternary)
- `defined?`
- `::` (scope resolution)

**The `Comparable` mixin — get all comparisons from `<=>`:**
```ruby
class Temperature
  include Comparable

  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  # Define only <=> and get <, >, <=, >=, between? for free!
  def <=>(other)
    @degrees <=> other.degrees
  end
end

t1 = Temperature.new(100)
t2 = Temperature.new(50)
puts t1 > t2    # true
puts t1 <= t2   # false
puts t2.between?(Temperature.new(0), Temperature.new(75))  # true
[t1, t2].sort   # works! (uses <=>)
```

---

### Dynamic Dispatch — How Ruby Resolves Methods

Ruby uses dynamic dispatch for ALL method calls (there's no "non-virtual" like C++). Every method call goes through the method lookup chain at runtime.

```ruby
class Animal
  def speak
    "..."
  end

  def describe
    # speak is resolved dynamically — will call the overridden version
    "I say: #{speak}"
  end
end

class Dog < Animal
  def speak
    "Woof!"
  end
end

puts Dog.new.describe  # "I say: Woof!" (not "I say: ...")
# describe calls speak, which is resolved to Dog#speak at runtime
```

**Key difference from C++:** In Ruby, ALL methods are effectively "virtual." There's no concept of static dispatch for instance methods. The method lookup always starts from the actual class of the object.

**Method lookup internals:**
```
Dog.new.speak

1. Check Dog for #speak → FOUND → call it
   (if not found, continue...)
2. Check included modules in Dog (reverse include order)
3. Check Parent class (Animal)
4. Check modules included in Parent
5. Continue up to Object → Kernel → BasicObject
6. If still not found → call #method_missing
```

---

### `method_missing` — Ruby's Meta-Polymorphism

When Ruby can't find a method anywhere in the lookup chain, it calls `method_missing`. This enables powerful dynamic behavior.

```ruby
class DynamicProxy
  def initialize(target)
    @target = target
    @call_count = Hash.new(0)
  end

  def method_missing(method_name, *args, &block)
    if @target.respond_to?(method_name)
      @call_count[method_name] += 1
      puts "Proxying #{method_name} (call ##{@call_count[method_name]})"
      @target.send(method_name, *args, &block)
    else
      super  # raise NoMethodError for truly missing methods
    end
  end

  # ALWAYS define respond_to_missing? alongside method_missing
  def respond_to_missing?(method_name, include_private = false)
    @target.respond_to?(method_name, include_private) || super
  end

  def call_stats
    @call_count.dup
  end
end

# Usage
proxy = DynamicProxy.new("hello world")
puts proxy.upcase      # Proxying upcase → "HELLO WORLD"
puts proxy.length      # Proxying length → 11
puts proxy.reverse     # Proxying reverse → "dlrow olleh"
puts proxy.call_stats  # {upcase: 1, length: 1, reverse: 1}
```

**Rules for `method_missing`:**
1. Always call `super` for methods you don't handle (preserves NoMethodError)
2. Always define `respond_to_missing?` alongside it
3. Use sparingly — it's slower than regular methods and harder to debug
4. Consider defining the method dynamically with `define_method` for repeated calls

---

### Open Classes and Monkey Patching

Ruby classes are "open" — you can add or modify methods on existing classes at any time. This is a form of polymorphism unique to dynamic languages.

```ruby
# Add a method to the built-in String class
class String
  def palindrome?
    self == self.reverse
  end

  def word_count
    split.size
  end
end

"racecar".palindrome?        # => true
"hello world".word_count     # => 2

# Refinements (Ruby 2.0+) — safer alternative to monkey patching
module StringExtensions
  refine String do
    def shout
      upcase + "!!!"
    end
  end
end

# Only active where explicitly used
class MyApp
  using StringExtensions

  def greet
    "hello".shout  # => "HELLO!!!" (only works in this scope)
  end
end

# "hello".shout  # NoMethodError (refinement not active here)
```

---

### Abstract Methods — Enforcing Interface Contracts

Ruby doesn't have `abstract` or `pure virtual` keywords. Instead, use conventions and runtime errors.

```ruby
# Approach 1: Raise NotImplementedError
class PaymentProcessor
  def process_payment(amount)
    raise NotImplementedError, "#{self.class}#process_payment must be implemented"
  end

  def refund(amount)
    raise NotImplementedError, "#{self.class}#refund must be implemented"
  end

  def provider_name
    raise NotImplementedError, "#{self.class}#provider_name must be implemented"
  end

  def transaction_fee(amount)
    raise NotImplementedError, "#{self.class}#transaction_fee must be implemented"
  end

  # Concrete method — shared implementation (Template Method pattern)
  def print_receipt(amount)
    puts "Payment of $#{'%.2f' % amount} processed via #{provider_name}"
    puts "Fee: $#{'%.2f' % transaction_fee(amount)}"
  end

  # Template Method — algorithm skeleton with customizable steps
  def safe_process(amount)
    validate_amount(amount)
    result = process_payment(amount)
    log_transaction(amount)
    result
  end

  private

  def validate_amount(amount)
    raise ArgumentError, "Amount must be positive" if amount <= 0
  end

  def log_transaction(amount)
    puts "[LOG] #{provider_name}: $#{'%.2f' % amount}"
  end
end

class StripeProcessor < PaymentProcessor
  def process_payment(amount)
    puts "Stripe: charging $#{'%.2f' % amount}"
    true
  end

  def refund(amount)
    puts "Stripe: refunding $#{'%.2f' % amount}"
    true
  end

  def provider_name
    "Stripe"
  end

  def transaction_fee(amount)
    amount * 0.029 + 0.30  # 2.9% + $0.30
  end
end

# PaymentProcessor.new.process_payment(100)  # NotImplementedError!
sp = StripeProcessor.new
sp.safe_process(100)
sp.print_receipt(100)
```

**Approach 2: Module with contract enforcement:**
```ruby
module AbstractInterface
  def self.included(klass)
    klass.instance_variable_set(:@abstract_methods, [])
    klass.extend(ClassMethods)
  end

  module ClassMethods
    def abstract_method(*methods)
      @abstract_methods.concat(methods)
      methods.each do |method|
        define_method(method) do |*args|
          raise NotImplementedError,
            "#{self.class}##{method} is abstract and must be implemented"
        end
      end
    end

    def abstract_methods
      @abstract_methods
    end
  end
end

class Shape
  include AbstractInterface

  abstract_method :area, :perimeter, :draw

  def describe
    "#{self.class}: area=#{area}, perimeter=#{perimeter}"
  end
end

class Circle < Shape
  def initialize(radius)
    @radius = radius
  end

  def area
    Math::PI * @radius ** 2
  end

  def perimeter
    2 * Math::PI * @radius
  end

  def draw
    puts "Drawing circle with radius #{@radius}"
  end
end
```

---


## 2.5 Abstraction

> Abstraction is the process of exposing only the essential features of an entity while hiding the unnecessary complexity. In Ruby, abstraction is achieved through abstract classes (convention-based), modules as interfaces, and duck typing.

---

### Abstract Classes vs Interfaces (Modules)

Ruby doesn't have dedicated `abstract` or `interface` keywords. Instead:
- **Abstract classes** = classes with methods that raise `NotImplementedError`
- **Interfaces** = modules that define a contract (expected methods)

**Abstract Class — partial abstraction:**
```ruby
class DatabaseConnection
  attr_reader :query_count

  def initialize(connection_string)
    @connection_string = connection_string
    @connected = false
    @query_count = 0
  end

  # Abstract methods — derived classes MUST implement
  def connect
    raise NotImplementedError, "#{self.class}#connect must be implemented"
  end

  def disconnect
    raise NotImplementedError, "#{self.class}#disconnect must be implemented"
  end

  def execute_query(sql)
    raise NotImplementedError, "#{self.class}#execute_query must be implemented"
  end

  # Concrete methods — shared implementation
  def connected?
    @connected
  end

  def log_query(sql)
    @query_count += 1
    puts "[Query ##{@query_count}] #{sql}"
  end

  # Template Method pattern — algorithm skeleton with customizable steps
  def safe_query(sql)
    connect unless connected?
    log_query(sql)
    execute_query(sql)
  end

  # Ensure cleanup
  def close
    disconnect if connected?
  end
end

class PostgresConnection < DatabaseConnection
  def connect
    # @pg_conn = PG.connect(@connection_string)
    @connected = true
    puts "Connected to PostgreSQL: #{@connection_string}"
  end

  def disconnect
    # @pg_conn.close
    @connected = false
    puts "Disconnected from PostgreSQL"
  end

  def execute_query(sql)
    # @pg_conn.exec(sql)
    puts "PostgreSQL executing: #{sql}"
    []
  end
end

class MySQLConnection < DatabaseConnection
  def connect
    @connected = true
    puts "Connected to MySQL: #{@connection_string}"
  end

  def disconnect
    @connected = false
    puts "Disconnected from MySQL"
  end

  def execute_query(sql)
    puts "MySQL executing: #{sql}"
    []
  end
end
```

**Interface — using modules as contracts:**
```ruby
# Interface: defines expected behavior (contract)
module Repository
  def save(entity)
    raise NotImplementedError, "#{self.class}#save must be implemented"
  end

  def find_by_id(id)
    raise NotImplementedError, "#{self.class}#find_by_id must be implemented"
  end

  def find_all
    raise NotImplementedError, "#{self.class}#find_all must be implemented"
  end

  def delete_by_id(id)
    raise NotImplementedError, "#{self.class}#delete_by_id must be implemented"
  end

  def exists?(id)
    raise NotImplementedError, "#{self.class}#exists? must be implemented"
  end
end

module Cacheable
  def put(key, value)
    raise NotImplementedError, "#{self.class}#put must be implemented"
  end

  def get(key)
    raise NotImplementedError, "#{self.class}#get must be implemented"
  end

  def invalidate(key)
    raise NotImplementedError, "#{self.class}#invalidate must be implemented"
  end

  def clear
    raise NotImplementedError, "#{self.class}#clear must be implemented"
  end
end

module EventListener
  def on_event(event)
    raise NotImplementedError, "#{self.class}#on_event must be implemented"
  end
end
```

**Key differences:**

| Aspect | Abstract Class | Module (Interface) |
|--------|---|---|
| Inheritance | Single (class < Parent) | Multiple (include many modules) |
| State | Can have instance variables | Can have instance variables (when included) |
| Concrete methods | Yes | Yes (but prefer keeping them minimal) |
| Constructor | Yes (initialize) | No (but can hook into including class) |
| Instantiation | Can be instantiated (no language prevention) | Cannot be instantiated |
| Purpose | Partial implementation + contract | Pure contract/shared behavior |
| When to use | Shared state and behavior among related classes | Defining capabilities/roles |

---

### Designing Interfaces with Modules

**Principles of good interface design:**

1. **Single Responsibility:** Each module should represent one cohesive capability
2. **Small and focused:** Prefer many small modules over one large one
3. **Stable:** Interfaces should change rarely (they are contracts)
4. **Complete:** Should provide all operations needed for the capability

```ruby
# GOOD: Small, focused modules representing distinct capabilities

module Readable
  def read
    raise NotImplementedError, "#{self.class}#read must be implemented"
  end

  def has_more?
    raise NotImplementedError, "#{self.class}#has_more? must be implemented"
  end

  def reset
    raise NotImplementedError, "#{self.class}#reset must be implemented"
  end
end

module Writable
  def write(data)
    raise NotImplementedError, "#{self.class}#write must be implemented"
  end

  def flush
    raise NotImplementedError, "#{self.class}#flush must be implemented"
  end
end

module Seekable
  def seek(position)
    raise NotImplementedError, "#{self.class}#seek must be implemented"
  end

  def tell
    raise NotImplementedError, "#{self.class}#tell must be implemented"
  end

  def size
    raise NotImplementedError, "#{self.class}#size must be implemented"
  end
end

module Closeable
  def close
    raise NotImplementedError, "#{self.class}#close must be implemented"
  end

  def closed?
    raise NotImplementedError, "#{self.class}#closed? must be implemented"
  end
end

# Compose interfaces as needed
class FileStream
  include Readable
  include Writable
  include Seekable
  include Closeable

  def initialize(path, mode = "r+")
    @file = File.open(path, mode)
    @closed = false
  end

  # Readable
  def read
    @file.gets
  end

  def has_more?
    !@file.eof?
  end

  def reset
    @file.rewind
  end

  # Writable
  def write(data)
    @file.write(data)
  end

  def flush
    @file.flush
  end

  # Seekable
  def seek(pos)
    @file.seek(pos)
  end

  def tell
    @file.tell
  end

  def size
    @file.size
  end

  # Closeable
  def close
    @file.close
    @closed = true
  end

  def closed?
    @closed
  end
end

# A network stream might only be readable and writable (not seekable)
class NetworkStream
  include Readable
  include Writable
  include Closeable
  # Does NOT include Seekable — networks aren't seekable
end
```

**Module naming conventions:**
- Adjective-like: `Enumerable`, `Comparable`, `Serializable`
- Capability-based: `Readable`, `Writable`, `Cacheable`
- Role-based: `Observer`, `EventHandler`, `DataProvider`

---

### Interface Segregation in Practice

**The Interface Segregation Principle (ISP):** No client should be forced to depend on methods it does not use.

**Problem — Fat Interface:**
```ruby
# BAD: One huge module forces all implementations to handle everything
module Worker
  def work; raise NotImplementedError; end
  def eat; raise NotImplementedError; end
  def sleep; raise NotImplementedError; end
  def attend_meeting; raise NotImplementedError; end
  def write_report; raise NotImplementedError; end
end

# A Robot worker doesn't eat or sleep!
class Robot
  include Worker

  def work
    puts "Robot working"
  end

  def eat; end           # meaningless! forced to implement
  def sleep; end         # meaningless! forced to implement
  def attend_meeting; end
  def write_report; end
end
```

**Solution — Segregated Interfaces:**
```ruby
# GOOD: Split into cohesive, focused modules
module Workable
  def work
    raise NotImplementedError, "#{self.class}#work must be implemented"
  end
end

module Feedable
  def eat
    raise NotImplementedError, "#{self.class}#eat must be implemented"
  end

  def drink
    raise NotImplementedError, "#{self.class}#drink must be implemented"
  end
end

module Sleepable
  def sleep_now
    raise NotImplementedError, "#{self.class}#sleep_now must be implemented"
  end

  def wake
    raise NotImplementedError, "#{self.class}#wake must be implemented"
  end
end

module Reportable
  def write_report
    raise NotImplementedError, "#{self.class}#write_report must be implemented"
  end

  def attend_meeting
    raise NotImplementedError, "#{self.class}#attend_meeting must be implemented"
  end
end

# Human worker implements all relevant interfaces
class HumanWorker
  include Workable
  include Feedable
  include Sleepable
  include Reportable

  def work
    puts "Human working"
  end

  def eat
    puts "Human eating"
  end

  def drink
    puts "Human drinking"
  end

  def sleep_now
    puts "Human sleeping"
  end

  def wake
    puts "Human waking"
  end

  def write_report
    puts "Writing report"
  end

  def attend_meeting
    puts "In meeting"
  end
end

# Robot only implements what makes sense
class RobotWorker
  include Workable

  def work
    puts "Robot working 24/7"
  end
  # No eat, sleep, or meeting methods — not forced to implement them!
end

# Functions depend only on the interfaces they need
def assign_task(worker)
  # Only requires Workable
  worker.work
end

def schedule_lunch(worker)
  # Only requires Feedable
  worker.eat
end
```

---

### Abstract Factory using Abstract Classes

The Abstract Factory pattern uses abstract classes/modules to create families of related objects without specifying their concrete types.

```ruby
# --- Abstract Products ---
module Button
  def render
    raise NotImplementedError
  end

  def on_click(&handler)
    raise NotImplementedError
  end

  def type
    raise NotImplementedError
  end
end

module TextBox
  def render
    raise NotImplementedError
  end

  def text=(value)
    raise NotImplementedError
  end

  def text
    raise NotImplementedError
  end
end

module CheckBox
  def render
    raise NotImplementedError
  end

  def toggle
    raise NotImplementedError
  end

  def checked?
    raise NotImplementedError
  end
end

# --- Abstract Factory ---
class UIFactory
  def create_button(label)
    raise NotImplementedError
  end

  def create_text_box(placeholder)
    raise NotImplementedError
  end

  def create_check_box(label)
    raise NotImplementedError
  end
end

# --- Concrete Products (Windows) ---
class WindowsButton
  include Button

  def initialize(label)
    @label = label
    @handler = nil
  end

  def render
    puts "[Win Button: #{@label}]"
  end

  def on_click(&handler)
    @handler = handler
  end

  def type
    "WindowsButton"
  end
end

class WindowsTextBox
  include TextBox

  def initialize(placeholder)
    @placeholder = placeholder
    @text = ""
  end

  def render
    display = @text.empty? ? @placeholder : @text
    puts "[Win TextBox: #{display}]"
  end

  def text=(value)
    @text = value
  end

  def text
    @text
  end
end

class WindowsCheckBox
  include CheckBox

  def initialize(label)
    @label = label
    @checked = false
  end

  def render
    mark = @checked ? "X" : " "
    puts "[Win CheckBox: [#{mark}] #{@label}]"
  end

  def toggle
    @checked = !@checked
  end

  def checked?
    @checked
  end
end

# --- Concrete Factory (Windows) ---
class WindowsUIFactory < UIFactory
  def create_button(label)
    WindowsButton.new(label)
  end

  def create_text_box(placeholder)
    WindowsTextBox.new(placeholder)
  end

  def create_check_box(label)
    WindowsCheckBox.new(label)
  end
end

# --- Concrete Products (Mac) ---
class MacButton
  include Button

  def initialize(label)
    @label = label
    @handler = nil
  end

  def render
    puts "(Mac Button: #{@label})"
  end

  def on_click(&handler)
    @handler = handler
  end

  def type
    "MacButton"
  end
end

class MacUIFactory < UIFactory
  def create_button(label)
    MacButton.new(label)
  end

  def create_text_box(placeholder)
    # MacTextBox.new(placeholder)
    nil
  end

  def create_check_box(label)
    # MacCheckBox.new(label)
    nil
  end
end

# --- Client code — works with ANY factory ---
class LoginForm
  def initialize(factory)
    @username_field = factory.create_text_box("Enter username")
    @password_field = factory.create_text_box("Enter password")
    @login_button = factory.create_button("Login")
    @remember_me = factory.create_check_box("Remember me")
  end

  def render
    @username_field&.render
    @password_field&.render
    @remember_me&.render
    @login_button&.render
  end
end

# Usage — switch entire UI theme by changing factory
factory = if RUBY_PLATFORM =~ /win/
            WindowsUIFactory.new
          else
            MacUIFactory.new
          end

form = LoginForm.new(factory)
form.render
```

---


## 2.6 Composition vs Inheritance

> One of the most important design decisions in OOP is choosing between inheritance ("is-a") and composition ("has-a"). The Gang of Four (GoF) famously advises: "Favor object composition over class inheritance."

---

### "Has-A" vs "Is-A" Relationships

**"Is-A" (Inheritance):** The child class IS a specialized version of the parent class. It should satisfy the Liskov Substitution Principle — anywhere a parent class is expected, a child class should work correctly.

**"Has-A" (Composition):** The containing class HAS a component as part of its implementation. The component provides functionality that the container uses.

```ruby
# IS-A: A Dog IS an Animal
class Animal
  def eat
    puts "Eating"
  end

  def sleep
    puts "Sleeping"
  end
end

class Dog < Animal
  def eat
    puts "Dog eating kibble"
  end

  def bark
    puts "Woof!"
  end
end

# HAS-A: A Car HAS an Engine, Transmission, and GPS
class Engine
  attr_reader :horsepower

  def initialize(hp)
    @horsepower = hp
    @running = false
  end

  def start
    @running = true
    puts "Engine started (#{@horsepower}hp)"
  end

  def stop
    @running = false
    puts "Engine stopped"
  end

  def running?
    @running
  end
end

class Transmission
  attr_reader :current_gear

  def initialize(max_gears)
    @max_gears = max_gears
    @current_gear = 0
  end

  def shift_up
    @current_gear += 1 if @current_gear < @max_gears
  end

  def shift_down
    @current_gear -= 1 if @current_gear > 0
  end
end

class GPS
  attr_reader :lat, :lon

  def initialize
    @lat = 0.0
    @lon = 0.0
  end

  def update_position(lat, lon)
    @lat = lat
    @lon = lon
  end

  def navigate(destination)
    puts "Navigating to #{destination}"
  end
end

class Car
  attr_reader :engine, :transmission

  def initialize(hp, gears, gps = nil)
    @engine = Engine.new(hp)              # composition: Car OWNS engine
    @transmission = Transmission.new(gears)  # composition: Car OWNS transmission
    @gps = gps                            # aggregation: GPS can exist independently
  end

  def start
    @engine.start
    puts "Car ready to drive"
  end

  def drive
    unless @engine.running?
      puts "Start the engine first!"
      return
    end
    @transmission.shift_up
    puts "Driving in gear #{@transmission.current_gear}"
  end

  def navigate_to(destination)
    if @gps
      @gps.navigate(destination)
    else
      puts "No GPS installed"
    end
  end
end
```

**How to decide:**
| Question | If Yes → | If No → |
|----------|----------|---------|
| Can you say "B is a type of A"? | Inheritance | Composition |
| Does Liskov Substitution hold? | Inheritance | Composition |
| Do you need polymorphism? | Inheritance (or composition + duck typing) | Composition |
| Might the relationship change at runtime? | Composition | Either |
| Is the parent class designed for inheritance? | Inheritance | Composition |

---

### When to Prefer Composition over Inheritance

**Problems with inheritance:**

1. **Tight coupling:** Child class depends on parent class implementation details
2. **Fragile base class problem:** Changes to parent class can break child classes
3. **Inflexible:** Relationship is fixed at definition time
4. **Single inheritance limit:** Ruby only allows one parent class

```ruby
# PROBLEM: Inheritance explosion
# Want: Coffee with different sizes, milks, and toppings
# With inheritance, you'd need many subclasses for each combination

# SOLUTION: Composition
class Milk
  def name
    raise NotImplementedError
  end

  def cost
    raise NotImplementedError
  end
end

class WholeMilk < Milk
  def name; "Whole Milk"; end
  def cost; 0.50; end
end

class OatMilk < Milk
  def name; "Oat Milk"; end
  def cost; 0.75; end
end

class Topping
  def name
    raise NotImplementedError
  end

  def cost
    raise NotImplementedError
  end
end

class WhippedCream < Topping
  def name; "Whipped Cream"; end
  def cost; 0.60; end
end

class Caramel < Topping
  def name; "Caramel"; end
  def cost; 0.50; end
end

class Coffee
  SIZES = { small: 3.0, medium: 4.0, large: 5.0 }.freeze

  attr_reader :size

  def initialize(size)
    @size = size
    @milk = nil
    @toppings = []
  end

  def milk=(milk)
    @milk = milk
    self
  end

  def add_topping(topping)
    @toppings << topping
    self
  end

  def total_cost
    base = SIZES[@size] || 3.0
    base += @milk.cost if @milk
    @toppings.each { |t| base += t.cost }
    base
  end

  def describe
    parts = ["Coffee (#{@size})"]
    parts << "+ #{@milk.name}" if @milk
    @toppings.each { |t| parts << "+ #{t.name}" }
    parts << "= $#{'%.2f' % total_cost}"
    puts parts.join(" ")
  end
end

# Any combination without class explosion!
coffee = Coffee.new(:large)
coffee.milk = OatMilk.new
coffee.add_topping(WhippedCream.new)
coffee.add_topping(Caramel.new)
coffee.describe  # Coffee (large) + Oat Milk + Whipped Cream + Caramel = $6.85
```

**Composition enables runtime flexibility — Strategy Pattern:**
```ruby
# Strategy pattern — behavior can change at runtime
module SortStrategy
  def sort(data)
    raise NotImplementedError
  end
end

class QuickSort
  include SortStrategy

  def sort(data)
    data.sort  # Ruby's built-in sort (Quicksort-based)
  end

  def to_s
    "QuickSort"
  end
end

class BubbleSort
  include SortStrategy

  def sort(data)
    arr = data.dup
    n = arr.size
    (n - 1).times do |i|
      (n - 1 - i).times do |j|
        arr[j], arr[j + 1] = arr[j + 1], arr[j] if arr[j] > arr[j + 1]
      end
    end
    arr
  end

  def to_s
    "BubbleSort"
  end
end

class InsertionSort
  include SortStrategy

  def sort(data)
    arr = data.dup
    (1...arr.size).each do |i|
      key = arr[i]
      j = i - 1
      while j >= 0 && arr[j] > key
        arr[j + 1] = arr[j]
        j -= 1
      end
      arr[j + 1] = key
    end
    arr
  end

  def to_s
    "InsertionSort"
  end
end

class DataProcessor
  attr_writer :strategy

  def initialize(strategy = nil)
    @strategy = strategy
  end

  def process(data)
    raise "No sort strategy set" unless @strategy
    puts "Sorting with #{@strategy}..."
    @strategy.sort(data)
  end
end

# Change behavior at runtime — impossible with inheritance alone
processor = DataProcessor.new(QuickSort.new)
data = [5, 2, 8, 1, 9]

result = processor.process(data)  # uses QuickSort
puts result.inspect

processor.strategy = BubbleSort.new
result = processor.process(data)  # now uses BubbleSort — same object, different behavior!
puts result.inspect
```

---

### Aggregation vs Composition

Both are "has-a" relationships, but they differ in **ownership** and **lifecycle management**.

#### Composition (Strong "Has-A" — Owns)

The container **owns** the component. The component's lifetime is tied to the container. When the container is destroyed (garbage collected), the component goes with it.

```ruby
class Heart
  def beat
    puts "Beating..."
  end
end

class Brain
  def think
    puts "Thinking..."
  end
end

class Human
  def initialize
    @heart = Heart.new   # composition: Human OWNS heart (created internally)
    @brain = Brain.new   # composition: Human OWNS brain
  end

  def live
    @heart.beat
    @brain.think
  end
end
# When Human is garbage collected, Heart and Brain are too
# (no other references to them exist)
```

**Implementation patterns for composition:**
- Create components inside the constructor
- Components are not shared with other objects
- No external references to the components

#### Aggregation (Weak "Has-A" — Uses)

The container **uses** the component but does NOT own it. The component can exist independently and may be shared.

```ruby
class Professor
  attr_reader :name, :department

  def initialize(name, department)
    @name = name
    @department = department
  end
end

class Student
  attr_reader :name, :id

  def initialize(name, id)
    @name = name
    @id = id
  end
end

class Course
  attr_reader :title

  def initialize(title, instructor)
    @title = title
    @instructor = instructor    # aggregation: Course doesn't own Professor
    @students = []              # aggregation: Course doesn't own Students
  end

  def enroll(student)
    @students << student
  end

  def display
    puts "#{@title} taught by #{@instructor.name}"
    puts "Students: #{@students.map(&:name).join(', ')}"
  end
end

# Aggregation in action:
prof = Professor.new("Dr. Smith", "CS")
s1 = Student.new("Alice", 1)
s2 = Student.new("Bob", 2)
s3 = Student.new("Charlie", 3)

cs101 = Course.new("Intro to CS", prof)
cs201 = Course.new("Data Structures", prof)  # same professor, two courses

cs101.enroll(s1)
cs101.enroll(s2)
cs201.enroll(s2)  # same student in two courses
cs201.enroll(s3)

# When cs101 is garbage collected, prof, s1, s2 still exist!
# Professor and Students have independent lifetimes
```

**Comparison Table:**

| Aspect | Composition | Aggregation |
|--------|---|---|
| Ownership | Strong (container owns part) | Weak (container uses part) |
| Lifecycle | Part dies with container | Part outlives container |
| Multiplicity | Part belongs to ONE container | Part can be shared |
| UML | Filled diamond ◆───── | Empty diamond ◇───── |
| Ruby implementation | Create internally, no external refs | Receive via constructor/setter |
| Example | Body-Heart, House-Room | Team-Player, Course-Student |
| Destruction | Part is GC'd with container | Part is NOT GC'd with container |

**Association (weakest relationship):**
```ruby
# Association: temporary "uses" relationship (not even aggregation)
class Logger
  def log(msg)
    puts msg
  end
end

class OrderService
  # No stored reference to Logger — just uses it temporarily
  def process_order(order, logger)  # association via parameter
    logger.log("Processing order...")
    # ... process ...
    logger.log("Order complete")
  end
end
```

---

### Dependency Injection in Ruby

**Dependency Injection (DI)** is a technique where an object receives its dependencies from the outside rather than creating them internally. This is the practical application of the Dependency Inversion Principle.

**Without DI (tight coupling):**
```ruby
class EmailService
  def send_email(to, msg)
    puts "Sending email to #{to}: #{msg}"
  end
end

class OrderProcessor
  def initialize
    @email_service = EmailService.new  # HARDCODED dependency — cannot change or test!
  end

  def process_order(order_id)
    # ... process order ...
    @email_service.send_email("customer@email.com", "Order #{order_id} confirmed")
  end
end
# Problems:
# - Cannot use a different notification service (SMS, Push)
# - Cannot test without actually sending emails
# - OrderProcessor is tightly coupled to EmailService
```

**With DI (loose coupling):**
```ruby
# Step 1: Define abstraction (interface via duck typing)
# In Ruby, we don't need a formal interface — just document the expected method
# Any object with a #notify(recipient, message) method works

# Step 2: Concrete implementations
class EmailNotificationService
  def notify(recipient, message)
    puts "EMAIL to #{recipient}: #{message}"
  end
end

class SMSNotificationService
  def notify(recipient, message)
    puts "SMS to #{recipient}: #{message}"
  end
end

class MockNotificationService
  attr_reader :sent_messages

  def initialize
    @sent_messages = []
  end

  def notify(recipient, message)
    @sent_messages << { recipient: recipient, message: message }
  end
end

# Step 3: Depend on duck type, inject from outside
class OrderProcessor
  def initialize(notifier)
    @notifier = notifier  # depends on anything with #notify method
  end

  def process_order(order_id, customer)
    # ... process order logic ...
    @notifier.notify(customer, "Order #{order_id} confirmed")
  end
end

# Usage — inject the desired implementation
email_service = EmailNotificationService.new
prod_processor = OrderProcessor.new(email_service)  # production: real emails
prod_processor.process_order("ORD-001", "alice@email.com")

mock_service = MockNotificationService.new
test_processor = OrderProcessor.new(mock_service)   # testing: no real emails sent
test_processor.process_order("ORD-002", "test@test.com")
# Verify: mock_service.sent_messages contains the expected message
puts mock_service.sent_messages.inspect
```

#### Types of Dependency Injection

**1. Constructor Injection (preferred):**
```ruby
class Service
  def initialize(db:, logger:, cache:)
    @db = db
    @logger = logger
    @cache = cache
  end

  def perform
    @logger.info("Starting service")
    data = @cache.get("key") || @db.query("SELECT ...")
    @logger.info("Done")
    data
  end
end

# All dependencies provided at construction — object is always fully initialized
service = Service.new(
  db: PostgresDB.new,
  logger: FileLogger.new,
  cache: RedisCache.new
)
```

**2. Setter Injection:**
```ruby
class Service
  attr_writer :database, :logger

  def perform
    raise "Database not set!" unless @database
    @logger&.info("Performing...")
    @database.query("SELECT ...")
  end
end

service = Service.new
service.database = PostgresDB.new
service.logger = FileLogger.new  # optional
service.perform
```

**3. Block/Proc Injection (Ruby-specific):**
```ruby
class DataProcessor
  def initialize(&formatter)
    @formatter = formatter || ->(data) { data.to_s }
  end

  def process(data)
    result = transform(data)
    @formatter.call(result)
  end

  private

  def transform(data)
    data.map { |x| x * 2 }
  end
end

# Inject behavior via block
json_processor = DataProcessor.new { |data| data.to_json }
csv_processor = DataProcessor.new { |data| data.join(",") }

puts json_processor.process([1, 2, 3])  # [2,4,6]
puts csv_processor.process([1, 2, 3])   # 2,4,6
```

**4. Module Injection (Mixin-based DI):**
```ruby
module ConsoleLogger
  def log(msg)
    puts "[LOG] #{msg}"
  end
end

module FileLogger
  def log(msg)
    File.open("app.log", "a") { |f| f.puts "[#{Time.now}] #{msg}" }
  end
end

class Application
  include ConsoleLogger  # swap to FileLogger for production

  def run
    log("Application starting")
    # ...
    log("Application finished")
  end
end
```

**Comparison of injection types:**

| Type | Pros | Cons | When to use |
|------|------|------|-------------|
| Constructor | Object always valid; clear requirements | Many params for many deps | Default choice; required dependencies |
| Setter | Flexible; optional deps; can change at runtime | Object may be in invalid state | Optional dependencies; reconfigurable |
| Block/Proc | Very Ruby-idiomatic; lightweight | Limited to single behavior | Simple strategy injection |
| Module | No runtime overhead; compile-time decision | Can't change at runtime | Fixed behavior selection |

**Benefits of Dependency Injection:**
1. **Testability:** Inject mocks/stubs for unit testing
2. **Flexibility:** Swap implementations without changing client code
3. **Decoupling:** Client depends on duck type, not concrete class
4. **Single Responsibility:** Object doesn't create its own dependencies
5. **Open/Closed:** Add new implementations without modifying existing code

---


## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Classes | Blueprint for objects; instance variables always private; use `attr_*` for access |
| Constructors | `initialize` method; must call `super` explicitly; no overloading (use defaults/kwargs) |
| Copying | `dup`/`clone` for copies; override `initialize_copy` for deep copy; `Marshal` for full deep copy |
| Freezing | `freeze` makes objects immutable; `frozen?` checks state; no "unfreeze" |
| Access Control | `public`, `protected`, `private`; private methods ARE inherited; `send` bypasses all |
| `self` | Current object in instance methods; current class in class context; needed for setter calls |
| Class Methods | `def self.method_name`; no instance access; use for factories, utilities |
| Class Variables | `@@var` shared across hierarchy (dangerous!); prefer class instance variables (`@var` at class level) |
| Constants | UPPERCASE; can be reassigned (with warning); `freeze` for true immutability |
| Encapsulation | Instance vars always private; `attr_*` for controlled access; behavior-rich over anemic |
| Immutability | `freeze` entire object; return new objects for "modifications"; thread-safe by default |
| Inheritance | Single inheritance only; `<` for subclassing; `super` for parent methods |
| Mixins | `include`/`extend`/`prepend` modules; Ruby's answer to multiple inheritance |
| Method Lookup | Class → Modules (reverse include order) → Parent → ... → BasicObject |
| Diamond Problem | Doesn't exist in Ruby; C3 linearization ensures single path |
| Construction Order | Must explicitly call `super`; not automatic like C++ |
| Duck Typing | "responds_to?" over "is_a?"; no type declarations needed for polymorphism |
| Method Overriding | All methods are "virtual"; dynamic dispatch always; `super` calls parent |
| No Overloading | Use default params, splat (`*args`), keyword args, or duck typing instead |
| Operator Overloading | Define methods like `+`, `-`, `<=>`, `[]`; use `Comparable` mixin for comparisons |
| `method_missing` | Catch-all for undefined methods; always pair with `respond_to_missing?` |
| Open Classes | Can modify any class at runtime; use Refinements for safer scoping |
| Abstract Methods | Convention: raise `NotImplementedError`; no language enforcement |
| Modules as Interfaces | Pure method contracts via modules; safe "multiple inheritance" |
| ISP | Small focused modules > one fat module |
| Composition | "Has-a"; flexible, runtime-changeable; prefer over inheritance |
| Aggregation | Weak "has-a"; part outlives container; shared references |
| Dependency Injection | Inject dependencies from outside; leverage duck typing for loose coupling |

---

## Interview Tips for Module 2

1. **Everything is an object** — In Ruby, even `nil`, `true`, `42`, and classes themselves are objects. Explain how this differs from languages with primitives.

2. **Access control** — Ruby's `private` means "cannot be called with an explicit receiver" (not "cannot be inherited"). Private methods ARE available in subclasses. Explain the difference from C++/Java.

3. **No method overloading** — Ruby doesn't support it. Explain alternatives: default parameters, keyword arguments, splat operators, and duck typing.

4. **Single inheritance + Mixins** — Explain why Ruby chose this over multiple inheritance. Modules provide shared behavior without the diamond problem. Know the method lookup order (MRO).

5. **Duck typing vs type checking** — "If it quacks like a duck..." Prefer `respond_to?` over `is_a?`. Explain how this enables polymorphism without inheritance hierarchies.

6. **`dup` vs `clone` vs `Marshal`** — Know the differences: `dup` doesn't copy frozen state or singleton methods; `clone` copies everything; `Marshal.load(Marshal.dump(obj))` for true deep copy.

7. **Composition over inheritance** — Be ready to explain with examples. Key reasons: flexibility, loose coupling, runtime behavior changes, avoiding single-inheritance limitation.

8. **`include` vs `extend` vs `prepend`** — `include` adds instance methods (after class in MRO); `extend` adds class methods; `prepend` adds instance methods (before class in MRO, enabling method wrapping).

9. **`method_missing` and metaprogramming** — Explain how it works, when to use it (proxies, DSLs), and the requirement to also define `respond_to_missing?`. Know the performance implications.

10. **Encapsulation in Ruby** — Instance variables are ALWAYS private (no public fields). Access is only through methods. Explain why `attr_accessor` for everything is an anti-pattern (anemic classes).

11. **`freeze` and immutability** — Explain how `freeze` works, that it's shallow (nested objects aren't frozen), and how to achieve deep freeze. Explain benefits for thread safety.

12. **Dependency Injection** — Explain constructor injection with duck typing. Ruby doesn't need interfaces — any object with the right methods works. Show how this enables testing with mocks.

13. **`super` keyword** — Explain the three forms: `super` (passes all args), `super()` (passes no args), `super(x, y)` (passes specific args). Forgetting `super` in `initialize` is a common bug.

14. **Open classes and Refinements** — Explain monkey patching (adding methods to existing classes), its dangers, and how Refinements (Ruby 2.0+) provide scoped, safer alternatives.

15. **Abstract classes in Ruby** — No language support. Convention is `raise NotImplementedError`. Explain how modules can serve as interface contracts and how duck typing often makes formal interfaces unnecessary.

---

## Ruby vs C++ OOP Comparison

| Feature | C++ | Ruby |
|---------|-----|------|
| Typing | Static, compile-time | Dynamic, runtime |
| Inheritance | Multiple | Single + Mixins |
| Polymorphism | Virtual functions + vtable | All methods are dynamic dispatch |
| Interfaces | Pure virtual classes | Modules / duck typing |
| Abstract classes | `= 0` pure virtual | Convention (`raise NotImplementedError`) |
| Access control | `public`/`protected`/`private` sections | Methods: `public`/`protected`/`private`; vars: always private |
| Memory management | Manual / RAII / smart pointers | Garbage collection |
| Constructors | Multiple (overloading) | Single `initialize` (use defaults/kwargs) |
| Destructors | Deterministic (`~Class`) | Non-deterministic (finalizers, GC) |
| Operator overloading | Member/friend functions | Define methods (`+`, `-`, `[]`, etc.) |
| Method overloading | Yes (by signature) | No (last definition wins) |
| Object slicing | Yes (pass by value) | No (always references) |
| `const` correctness | Yes (`const` methods, objects) | `freeze` (runtime, not compile-time) |
| Templates/Generics | Templates (compile-time) | Not needed (duck typing) |
| RTTI | `dynamic_cast`, `typeid` | `is_a?`, `instance_of?`, `respond_to?` |
| Friend access | `friend` keyword | `send` method / protected methods |
| Static members | `static` keyword | Class methods (`def self.x`) / class variables (`@@x`) |
