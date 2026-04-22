# Module 1: Ruby Fundamentals (Prerequisites)

> This module covers all the foundational Ruby concepts you must master before diving into Low-Level Design. These are the building blocks for writing clean, expressive, and well-structured object-oriented code in Ruby.

---

## 1.1 Basics

### Variables and Data Types

Ruby is a **dynamically typed** language — you don't declare types explicitly. The interpreter determines the type at runtime.

**Variable Types by Scope:**

| Variable | Prefix | Scope | Example |
|----------|--------|-------|---------|
| Local | none (lowercase) | Block/method | `name = "Alice"` |
| Instance | `@` | Object instance | `@age = 25` |
| Class | `@@` | Entire class | `@@count = 0` |
| Global | `$` | Entire program | `$debug = true` |
| Constant | UPPERCASE | Lexical scope | `PI = 3.14159` |

**Core Data Types:**

| Type | Example | Notes |
|------|---------|-------|
| `Integer` | `42`, `1_000_000` | Arbitrary precision (no overflow!) |
| `Float` | `3.14`, `2.5e10` | 64-bit double precision |
| `String` | `"hello"`, `'world'` | Mutable by default |
| `Symbol` | `:name`, `:status` | Immutable, interned strings (fast comparison) |
| `Boolean` | `true`, `false` | TrueClass / FalseClass |
| `NilClass` | `nil` | Absence of value (only `nil` and `false` are falsy) |
| `Array` | `[1, 2, 3]` | Ordered, heterogeneous collection |
| `Hash` | `{a: 1, b: 2}` | Key-value pairs (like a dictionary) |
| `Range` | `1..10`, `1...10` | Inclusive (`..`) vs exclusive (`...`) of end |

**Everything is an object in Ruby:**
```ruby
42.class          # => Integer
3.14.class        # => Float
"hello".class     # => String
nil.class         # => NilClass
true.class        # => TrueClass
[1,2,3].class     # => Array

# Even methods are objects
method(:puts).class  # => Method
```

**Type Checking and Conversion:**
```ruby
# Type checking
x = 42
x.is_a?(Integer)    # => true
x.is_a?(Numeric)    # => true (Integer inherits from Numeric)
x.is_a?(String)     # => false
x.class             # => Integer
x.instance_of?(Integer)  # => true (exact class, not ancestors)

# Explicit conversion (strict — raises error if impossible)
Integer("42")       # => 42
Integer("hello")    # => ArgumentError!
Float("3.14")       # => 3.14
String(42)          # => "42"
Array(nil)          # => []
Array({a: 1})       # => [[:a, 1]]

# Implicit conversion (lenient — returns nil or best effort)
"42".to_i           # => 42
"hello".to_i        # => 0 (no error!)
"3.14".to_f         # => 3.14
42.to_s             # => "42"
42.to_f             # => 42.0
[1,2,3].to_s        # => "[1, 2, 3]"

# Duck typing — Ruby doesn't care about type, only behavior
# If it responds to the method, it works
def double(x)
  x + x  # works for Integer, Float, String, Array...
end

double(5)        # => 10
double("hi")     # => "hihi"
double([1, 2])   # => [1, 2, 1, 2]
```

**Symbols vs Strings:**
```ruby
# Symbols are immutable and unique (same symbol = same object in memory)
:name.object_id == :name.object_id   # => true (same object!)
"name".object_id == "name".object_id # => false (different objects!)

# Symbols are ideal for:
# - Hash keys (faster lookup)
# - Method names
# - Identifiers that don't change

# Strings are ideal for:
# - Text that changes
# - User input
# - Display/output

# Conversion
"hello".to_sym    # => :hello
:hello.to_s       # => "hello"
```

**Frozen (Immutable) Objects:**
```ruby
str = "hello"
str.freeze          # makes the object immutable
# str << " world"   # => FrozenError: can't modify frozen String

# In Ruby 3+, string literals in files with this magic comment are frozen:
# frozen_string_literal: true
```

---

### Operators

| Category | Operators | Notes |
|----------|-----------|-------|
| Arithmetic | `+`, `-`, `*`, `/`, `%`, `**` | `**` is exponentiation |
| Comparison | `==`, `!=`, `<`, `>`, `<=`, `>=`, `<=>` | `<=>` is spaceship operator |
| Logical | `&&`, `||`, `!`, `and`, `or`, `not` | Word versions have lower precedence |
| Bitwise | `&`, `|`, `^`, `~`, `<<`, `>>` | Operate on bits |
| Assignment | `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `**=`, `||=`, `&&=` | `||=` is conditional assign |
| Range | `..`, `...` | Inclusive vs exclusive end |
| Ternary | `condition ? expr1 : expr2` | Inline if-else |

**Key Ruby-specific operators:**

```ruby
# Spaceship operator <=> (returns -1, 0, or 1)
5 <=> 3    # => 1  (left is greater)
3 <=> 5    # => -1 (left is lesser)
5 <=> 5    # => 0  (equal)
# Used by sort: [3, 1, 2].sort { |a, b| a <=> b }

# Conditional assignment ||=
x = nil
x ||= 10   # x is now 10 (assigns only if x is nil or false)
x ||= 20   # x is still 10 (already has a truthy value)

# Safe navigation operator &. (Ruby 2.3+)
user = nil
user&.name   # => nil (no NoMethodError!)
# Equivalent to: user && user.name

# Double splat ** (hash decomposition)
def configure(**options)
  options  # => {timeout: 30, retries: 3}
end
configure(timeout: 30, retries: 3)

# Pattern matching (Ruby 3.0+)
case [1, 2, 3]
in [Integer => a, Integer => b, Integer => c]
  puts "#{a}, #{b}, #{c}"
end
```

**Truthiness in Ruby:**
```ruby
# ONLY nil and false are falsy. Everything else is truthy!
# This is different from many languages:
if 0          # truthy! (0 is NOT falsy in Ruby)
if ""         # truthy! (empty string is NOT falsy in Ruby)
if []         # truthy! (empty array is NOT falsy in Ruby)
if nil        # falsy
if false      # falsy
```

---

### Control Flow

**Conditional Statements:**
```ruby
# if-elsif-else
if score >= 90
  grade = "A"
elsif score >= 80
  grade = "B"
elsif score >= 70
  grade = "C"
else
  grade = "F"
end

# Single-line (modifier form) — idiomatic Ruby
puts "Adult" if age >= 18
puts "Minor" unless age >= 18

# unless (opposite of if — use for negative conditions)
unless logged_in?
  redirect_to login_path
end

# case-when (Ruby's switch — much more powerful)
case value
when 1..5
  puts "low"
when 6..10
  puts "high"
when String
  puts "it's a string!"
when /^hello/
  puts "starts with hello"
when ->(x) { x.even? }
  puts "it's even (using lambda)"
else
  puts "something else"
end

# case can match types, ranges, regex, lambdas — not just equality!

# Ternary
status = age >= 18 ? "adult" : "minor"

# Pattern matching (Ruby 3.0+)
case {name: "Alice", age: 30, role: "admin"}
in {name: String => name, role: "admin"}
  puts "Admin: #{name}"
in {name: String => name, role: "user"}
  puts "User: #{name}"
end
```

**Loops:**
```ruby
# while
while condition
  # ...
end

# until (opposite of while)
until queue.empty?
  process(queue.pop)
end

# for (rarely used in idiomatic Ruby)
for i in 1..5
  puts i
end

# Idiomatic Ruby uses iterators instead of loops:
5.times { |i| puts i }              # 0, 1, 2, 3, 4
1.upto(5) { |i| puts i }            # 1, 2, 3, 4, 5
10.downto(1) { |i| puts i }         # 10, 9, ..., 1
(1..10).each { |i| puts i }         # 1 through 10

# loop (infinite loop — use break to exit)
loop do
  input = gets.chomp
  break if input == "quit"
end

# Modifier form
puts "working..." while processing?
```

**Important:** `break` exits the loop, `next` skips to next iteration (like `continue` in C++), `redo` restarts the current iteration.

---

### Methods (Functions)

In Ruby, functions defined inside a class are called **methods**. Standalone functions are actually methods on `main` (the top-level object).

```ruby
# Method definition
def greet(name)
  "Hello, #{name}!"  # last expression is implicitly returned
end

# Explicit return (used for early exit)
def divide(a, b)
  return nil if b == 0  # early return
  a.to_f / b
end

# Default arguments
def connect(host, port = 3000, ssl: false)
  puts "Connecting to #{host}:#{port} (SSL: #{ssl})"
end
connect("localhost")              # port=3000, ssl=false
connect("api.com", 443, ssl: true)

# Variable arguments (splat operator)
def sum(*numbers)
  numbers.reduce(0, :+)
end
sum(1, 2, 3, 4, 5)  # => 15

# Keyword arguments (Ruby 2.0+)
def create_user(name:, email:, role: "user")
  { name: name, email: email, role: role }
end
create_user(name: "Alice", email: "alice@example.com")

# Double splat (arbitrary keyword arguments)
def configure(**options)
  options.each { |key, val| puts "#{key}: #{val}" }
end
configure(timeout: 30, retries: 3, verbose: true)
```

**Method naming conventions:**
```ruby
# Predicate methods (return boolean) end with ?
def empty?
  @items.length == 0
end

# Dangerous/mutating methods end with !
def sort!    # modifies in place
  @data.sort!
end

# Setter methods end with =
def name=(new_name)
  @name = new_name
end
```

**Blocks, Procs, and Lambdas:**
```ruby
# Block — anonymous chunk of code passed to a method
[1, 2, 3].each { |x| puts x }          # single-line block
[1, 2, 3].each do |x|                   # multi-line block
  puts x
end

# Yielding to a block
def with_logging
  puts "Starting..."
  result = yield  # execute the block
  puts "Done!"
  result
end
with_logging { expensive_operation }

# Proc — block stored as an object
square = Proc.new { |x| x ** 2 }
square.call(5)   # => 25
square.(5)       # => 25 (shorthand)
square[5]        # => 25 (another shorthand)

# Lambda — stricter Proc (checks argument count, return behavior)
multiply = ->(a, b) { a * b }
multiply.call(3, 4)  # => 12
# multiply.call(3)   # => ArgumentError (lambda checks arity)

# Proc vs Lambda:
# - Lambda checks argument count, Proc doesn't
# - return in Lambda returns from the lambda
# - return in Proc returns from the ENCLOSING method
```

**Comparison of callable objects:**

| Feature | Block | Proc | Lambda |
|---------|-------|------|--------|
| Object? | No (syntax) | Yes | Yes |
| Arity check | No | No | Yes |
| `return` behavior | N/A | Returns from enclosing method | Returns from lambda |
| Creation | `{ }` or `do...end` | `Proc.new { }` | `->() { }` or `lambda { }` |
| Passing to method | Implicit (yield) | Explicit argument | Explicit argument |

---

### Recursion

```ruby
# Factorial
def factorial(n)
  return 1 if n <= 1   # base case
  n * factorial(n - 1) # recursive case
end

# Fibonacci with memoization
def fib(n, memo = {})
  return n if n <= 1
  memo[n] ||= fib(n - 1, memo) + fib(n - 2, memo)
end

# Ruby doesn't optimize tail recursion by default, but you can enable it:
# RubyVM::InstructionSequence.compile_option = { tailcall_optimization: true }

# Practical recursion: flatten nested array
def deep_flatten(arr)
  arr.each_with_object([]) do |elem, result|
    if elem.is_a?(Array)
      result.concat(deep_flatten(elem))
    else
      result << elem
    end
  end
end

deep_flatten([1, [2, [3, [4]]], 5])  # => [1, 2, 3, 4, 5]
```

**Key Concepts:**
- **Base case:** Condition that stops recursion
- **Recursive case:** Method calls itself with a smaller input
- **SystemStackError:** Ruby's equivalent of stack overflow (default stack ~10,000 frames)
- **Memoization:** Cache results to avoid redundant computation
- **Tail recursion:** Not optimized by default in MRI Ruby (use iteration for deep recursion)

---

## 1.2 Memory & Object Model

> Ruby manages memory automatically through garbage collection. Unlike C++, you don't manually allocate or free memory. However, understanding Ruby's object model and memory behavior is essential for writing efficient code and avoiding leaks.

---

### Object Identity and References

In Ruby, **variables hold references** to objects, not the objects themselves. Every object has a unique `object_id`.

```ruby
a = "hello"
b = a          # b references the SAME object as a

a.object_id == b.object_id  # => true (same object!)

b << " world"  # mutates the shared object
puts a         # => "hello world" (a is affected!)

# To get an independent copy:
c = a.dup      # shallow copy
d = a.clone    # shallow copy (also copies frozen state and singleton methods)

c << " ruby"
puts a         # => "hello world" (a is NOT affected)
```

**Object identity vs equality:**
```ruby
a = "hello"
b = "hello"

a == b         # => true  (same VALUE — content equality)
a.equal?(b)    # => false (different OBJECTS — identity equality)
a.eql?(b)      # => true  (same value and same type — used by Hash)

# For symbols and small integers, same value = same object:
:foo.equal?(:foo)  # => true (symbols are interned)
1.equal?(1)        # => true (small integers are cached)
```

---

### Mutable vs Immutable Objects

```ruby
# Immutable objects (cannot be changed after creation):
# - Integer, Float, Symbol, true, false, nil, Frozen objects

x = 42
# x itself is immutable — you can't change the integer 42
# You can only reassign x to point to a different object
x = 43  # x now references a different Integer object

# Mutable objects (can be changed in place):
# - String, Array, Hash (by default)

str = "hello"
str << " world"    # mutates str in place
str.upcase!        # mutates str in place

arr = [1, 2, 3]
arr.push(4)        # mutates arr in place
arr[0] = 99        # mutates arr in place

# Freezing makes any object immutable:
str = "hello"
str.freeze
# str << " world"  # => FrozenError!
str.frozen?        # => true

# Frozen objects cannot be modified, but the variable can be reassigned:
str = "new string"  # OK — str now points to a different (unfrozen) object
```

---

### Garbage Collection

Ruby uses a **mark-and-sweep** garbage collector (with generational and incremental improvements in modern Ruby).

```ruby
# Objects are automatically freed when no references point to them

def create_objects
  arr = Array.new(1_000_000) { |i| "string_#{i}" }
  # arr goes out of scope when method returns
  # All 1,000,000 strings become eligible for GC
end

create_objects
GC.start  # manually trigger garbage collection (rarely needed)

# GC statistics
GC.stat   # => hash with GC metrics
GC.count  # => number of GC runs so far
```

**Common memory concerns in Ruby:**

```ruby
# 1. Unintentional object retention (memory leak)
class EventBus
  def initialize
    @listeners = []
  end

  def subscribe(listener)
    @listeners << listener  # listener is retained forever!
  end
  # Fix: provide an unsubscribe method, or use WeakRef
end

# 2. String duplication
# BAD: creates a new string object every time
1000.times { process("constant_key") }  # 1000 String objects!

# GOOD: use a frozen constant or symbol
KEY = "constant_key".freeze
1000.times { process(KEY) }  # 1 String object reused

# Or use symbols (always interned):
1000.times { process(:constant_key) }

# 3. Large collections — use lazy enumerators
# BAD: creates entire intermediate array in memory
(1..1_000_000).select { |x| x.even? }.first(10)

# GOOD: lazy evaluation — only processes what's needed
(1..1_000_000).lazy.select { |x| x.even? }.first(10)
```

---

### WeakRef (Weak References)

```ruby
require 'weakref'

class Cache
  def initialize
    @store = {}
  end

  def put(key, value)
    @store[key] = WeakRef.new(value)
    # WeakRef doesn't prevent garbage collection
  end

  def get(key)
    ref = @store[key]
    return nil unless ref

    begin
      ref.__getobj__  # get the actual object
    rescue WeakRef::RefError
      @store.delete(key)  # object was garbage collected
      nil
    end
  end
end
```

---

### Object Copying (Shallow vs Deep)

```ruby
# Shallow copy: dup / clone
original = { name: "Alice", hobbies: ["reading", "coding"] }
copy = original.dup

copy[:name] = "Bob"           # doesn't affect original
copy[:hobbies] << "gaming"    # DOES affect original! (shared nested object)

puts original[:hobbies]  # => ["reading", "coding", "gaming"] — modified!

# Deep copy: Marshal (serialize + deserialize)
deep_copy = Marshal.load(Marshal.dump(original))
deep_copy[:hobbies] << "swimming"
puts original[:hobbies]  # => ["reading", "coding", "gaming"] — NOT affected

# Note: Marshal can't handle Procs, lambdas, IO objects, or singleton methods
```

**dup vs clone:**

| Feature | `dup` | `clone` |
|---------|-------|---------|
| Copies instance variables | Yes | Yes |
| Copies frozen state | No | Yes |
| Copies singleton methods | No | Yes |
| Calls `initialize_copy` | Yes | Yes |

---

### ObjectSpace

`ObjectSpace` lets you inspect all living objects in the Ruby process.

```ruby
# Count objects by type
ObjectSpace.count_objects
# => {TOTAL: 50000, FREE: 10000, T_STRING: 15000, T_ARRAY: 5000, ...}

# Iterate over all objects of a type
ObjectSpace.each_object(String) { |s| puts s if s.include?("secret") }

# Find what references an object (debugging memory leaks)
require 'objspace'
ObjectSpace.trace_object_allocations_start
# ... run code ...
ObjectSpace.allocation_sourcefile(obj)  # where was this object created?
ObjectSpace.allocation_sourceline(obj)  # which line?
ObjectSpace.memsize_of(obj)             # memory size in bytes
```

---

## 1.3 Collections & Enumerable

> Ruby's collections (Array, Hash, Set) combined with the Enumerable module provide the same power as C++'s STL — but with a more expressive, functional style.

---

### Array (Dynamic, Ordered Collection)

Ruby's `Array` is equivalent to C++'s `std::vector` — a dynamically-sized, ordered collection.

```ruby
# Creation
arr = [1, 2, 3, 4, 5]
arr = Array.new(5, 0)         # [0, 0, 0, 0, 0]
arr = Array.new(5) { |i| i * 2 }  # [0, 2, 4, 6, 8]
arr = %w[apple banana cherry]  # ["apple", "banana", "cherry"]
arr = %i[foo bar baz]          # [:foo, :bar, :baz]

# Access
arr[0]          # first element
arr[-1]         # last element
arr[1..3]       # slice (elements at index 1, 2, 3)
arr.first(3)    # first 3 elements
arr.last(2)     # last 2 elements
arr.sample      # random element
arr.dig(0, 1)   # safe nested access (returns nil if path doesn't exist)

# Modification
arr.push(6)         # add to end (alias: arr << 6)
arr.pop             # remove from end, returns removed element
arr.unshift(0)      # add to front
arr.shift           # remove from front
arr.insert(2, 99)   # insert 99 at index 2
arr.delete_at(2)    # delete element at index 2
arr.delete(99)      # delete all occurrences of 99
arr.compact         # remove all nil values (returns new array)
arr.compact!        # remove all nil values (in place)
arr.flatten         # flatten nested arrays
arr.uniq            # remove duplicates

# Size
arr.length      # or arr.size or arr.count
arr.empty?      # true if no elements

# Searching
arr.include?(5)         # true/false
arr.index(5)            # index of first occurrence (nil if not found)
arr.rindex(5)           # index of last occurrence
arr.find { |x| x > 3 } # first element matching condition
arr.count { |x| x > 3 } # count elements matching condition

# Sorting
arr.sort                          # ascending (returns new array)
arr.sort { |a, b| b <=> a }      # descending
arr.sort_by { |x| x.length }     # sort by computed value
arr.sort!                         # sort in place
arr.min                           # minimum element
arr.max                           # maximum element
arr.minmax                        # [min, max]

# Transformation
arr.map { |x| x * 2 }            # transform each element (alias: collect)
arr.select { |x| x.even? }       # filter (alias: filter)
arr.reject { |x| x.even? }       # inverse filter
arr.reduce(0) { |sum, x| sum + x }  # fold/accumulate (alias: inject)
arr.flat_map { |x| [x, x * 2] } # map + flatten
arr.zip([10, 20, 30])            # [[1,10], [2,20], [3,30]]
arr.each_with_index { |val, i| } # iterate with index
arr.each_with_object({}) { |x, hash| hash[x] = x * 2 }
arr.group_by { |x| x % 2 == 0 ? :even : :odd }
arr.chunk { |x| x > 3 }         # group consecutive elements
arr.tally                        # count occurrences {elem => count} (Ruby 2.7+)

# Set operations
[1,2,3] & [2,3,4]    # intersection: [2, 3]
[1,2,3] | [2,3,4]    # union: [1, 2, 3, 4]
[1,2,3] - [2,3,4]    # difference: [1]

# Conversion
arr.join(", ")        # "1, 2, 3, 4, 5"
arr.to_s              # "[1, 2, 3, 4, 5]"
```

**Performance characteristics:**

| Operation | Time Complexity |
|-----------|----------------|
| Access by index | O(1) |
| Push/Pop (end) | O(1) amortized |
| Unshift/Shift (front) | O(n) |
| Insert/Delete (middle) | O(n) |
| Include? (search) | O(n) |
| Sort | O(n log n) |

---

### Hash (Key-Value Store)

Ruby's `Hash` is equivalent to C++'s `std::unordered_map` — a hash table with O(1) average lookup.

```ruby
# Creation
hash = { "name" => "Alice", "age" => 30 }     # string keys
hash = { name: "Alice", age: 30 }              # symbol keys (preferred)
hash = Hash.new(0)                              # default value 0 for missing keys
hash = Hash.new { |h, k| h[k] = [] }           # default block (creates new array per key)

# Access
hash[:name]             # => "Alice"
hash[:missing]          # => nil (or default value)
hash.fetch(:name)       # => "Alice" (raises KeyError if missing)
hash.fetch(:missing, "default")  # => "default" (with fallback)
hash.dig(:user, :address, :city) # safe nested access

# Modification
hash[:email] = "alice@example.com"  # add/update
hash.delete(:age)                    # remove key, returns value
hash.merge(other_hash)               # combine (returns new hash)
hash.merge!(other_hash)              # combine in place (alias: update)
hash.transform_values { |v| v.to_s } # transform all values
hash.transform_keys { |k| k.to_s }   # transform all keys
hash.select { |k, v| v > 10 }        # filter
hash.reject { |k, v| v.nil? }        # inverse filter
hash.compact                          # remove nil values

# Querying
hash.key?(:name)        # true (alias: has_key?, include?)
hash.value?("Alice")    # true (alias: has_value?)
hash.keys               # array of all keys
hash.values             # array of all values
hash.length             # number of key-value pairs
hash.empty?             # true if no pairs
hash.any? { |k, v| v > 25 }  # true if any pair matches

# Iteration
hash.each { |key, value| puts "#{key}: #{value}" }
hash.each_pair { |k, v| }    # same as each
hash.each_key { |k| }
hash.each_value { |v| }
hash.map { |k, v| [k, v.upcase] }.to_h  # transform to new hash

# Destructuring
name, age = hash.values_at(:name, :age)

# Useful patterns
# Word frequency counter
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
freq = words.tally  # => {"apple"=>3, "banana"=>2, "cherry"=>1}

# Grouping
people = [{name: "Alice", dept: "Eng"}, {name: "Bob", dept: "Eng"}, {name: "Carol", dept: "HR"}]
by_dept = people.group_by { |p| p[:dept] }
# => {"Eng"=>[{...}, {...}], "HR"=>[{...}]}
```

**Hash ordering:** Since Ruby 1.9, Hashes maintain insertion order.

---

### Set

Ruby's `Set` is equivalent to C++'s `std::unordered_set` — a collection of unique elements with O(1) lookup.

```ruby
require 'set'

# Creation
s = Set.new([1, 2, 3, 3, 4])  # => #<Set: {1, 2, 3, 4}> (duplicates removed)
s = Set[1, 2, 3]
s = [1, 2, 2, 3].to_set

# Operations
s.add(5)            # add element (alias: <<)
s.delete(3)         # remove element
s.include?(2)       # membership test — O(1)!
s.length            # number of elements

# Set operations
a = Set[1, 2, 3, 4]
b = Set[3, 4, 5, 6]

a & b               # intersection: #<Set: {3, 4}>
a | b               # union: #<Set: {1, 2, 3, 4, 5, 6}>
a - b               # difference: #<Set: {1, 2}>
a ^ b               # symmetric difference: #<Set: {1, 2, 5, 6}>

a.subset?(b)        # is a a subset of b?
a.superset?(b)      # is a a superset of b?
a.intersect?(b)     # do they share any elements?
a.disjoint?(b)      # do they share NO elements?
```

**When to use Set vs Array:**
- Use `Set` when you need fast membership testing (`include?`)
- Use `Set` when you need uniqueness guaranteed
- Use `Array` when order matters or you need index access
- Use `Array` when you have few elements (Set overhead not worth it for < ~10 elements)

---

### Stack and Queue Behavior

Ruby doesn't have dedicated Stack/Queue classes — use Array with specific methods:

```ruby
# Stack (LIFO) — use push/pop
stack = []
stack.push(1)    # [1]
stack.push(2)    # [1, 2]
stack.push(3)    # [1, 2, 3]
stack.last       # peek: 3
stack.pop        # 3, stack is now [1, 2]

# Queue (FIFO) — use push/shift
queue = []
queue.push(1)    # [1]
queue.push(2)    # [1, 2]
queue.push(3)    # [1, 2, 3]
queue.first      # peek: 1
queue.shift      # 1, queue is now [2, 3]

# Thread-safe queue (for concurrent programming)
require 'thread'
q = Queue.new
q.push("task1")
q.push("task2")
q.pop            # "task1" (blocks if empty)
q.size           # 1

# SizedQueue (bounded buffer)
sq = SizedQueue.new(5)  # max 5 elements
sq.push("item")         # blocks if full
```

---

### Comparable and Enumerable Modules

**Comparable** — mix in to get comparison operators from `<=>`:
```ruby
class Temperature
  include Comparable

  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  # Define <=> and get <, <=, ==, >=, >, between? for free
  def <=>(other)
    @degrees <=> other.degrees
  end
end

t1 = Temperature.new(100)
t2 = Temperature.new(50)
t1 > t2          # => true
t1.between?(Temperature.new(0), Temperature.new(200))  # => true
[t2, t1, Temperature.new(75)].sort  # sorted by degrees
```

**Enumerable** — mix in to get 50+ collection methods from `each`:
```ruby
class NumberRange
  include Enumerable

  def initialize(from, to)
    @from = from
    @to = to
  end

  # Define each and get map, select, reduce, sort, min, max, etc. for free!
  def each
    current = @from
    while current <= @to
      yield current
      current += 1
    end
  end
end

range = NumberRange.new(1, 10)
range.map { |x| x * 2 }        # => [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
range.select { |x| x.even? }   # => [2, 4, 6, 8, 10]
range.reduce(:+)                # => 55
range.min                       # => 1
range.max                       # => 10
range.to_a                      # => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
range.count                     # => 10
range.include?(5)               # => true
range.sort_by { |x| -x }       # => [10, 9, 8, ..., 1]
```

---

### Lazy Enumerators

For large or infinite sequences, lazy evaluation processes elements one at a time:

```ruby
# Without lazy: creates entire intermediate arrays in memory
(1..Float::INFINITY).select { |x| x.even? }.first(5)  # hangs forever!

# With lazy: processes only what's needed
(1..Float::INFINITY).lazy.select { |x| x.even? }.first(5)
# => [2, 4, 6, 8, 10]

# Chaining lazy operations
File.open("huge_file.log").each_line.lazy
  .select { |line| line.include?("ERROR") }
  .map { |line| line.strip }
  .first(10)
# Only reads lines until 10 errors are found — doesn't load entire file!

# Creating custom lazy enumerators
fib = Enumerator.new do |yielder|
  a, b = 0, 1
  loop do
    yielder.yield a
    a, b = b, a + b
  end
end

fib.lazy.select { |x| x.even? }.first(5)  # => [0, 2, 8, 34, 144]
```

---

## 1.4 Modules, Mixins & Metaprogramming

> Where C++ uses templates for generic programming, Ruby uses modules (mixins) and metaprogramming. These are Ruby's most powerful features for code reuse and building flexible, DRY designs.

---

### Modules as Namespaces

Modules group related constants, methods, and classes to prevent name collisions (like C++ namespaces).

```ruby
module Payments
  TAX_RATE = 0.08

  class CreditCard
    def charge(amount)
      puts "Charging $#{amount} to credit card"
    end
  end

  class PayPal
    def charge(amount)
      puts "Charging $#{amount} via PayPal"
    end
  end

  # Module method (called on the module itself)
  def self.supported_methods
    ["credit_card", "paypal", "bank_transfer"]
  end
end

# Usage
card = Payments::CreditCard.new
card.charge(99.99)
Payments::TAX_RATE          # => 0.08
Payments.supported_methods  # => ["credit_card", "paypal", "bank_transfer"]
```

**Nested modules:**
```ruby
module Company
  module Engineering
    module Backend
      class APIServer
        # ...
      end
    end
  end
end

# Access
server = Company::Engineering::Backend::APIServer.new

# Shorthand (Ruby 3.0+ compact nesting)
module Company::Engineering::Backend
  class APIServer; end
end
```

---

### Modules as Mixins (include, extend, prepend)

Mixins are Ruby's answer to multiple inheritance — share behavior across unrelated classes without inheritance hierarchies.

**`include` — adds module methods as instance methods:**
```ruby
module Loggable
  def log(message)
    puts "[#{self.class.name}] #{message}"
  end

  def log_error(message)
    puts "[ERROR][#{self.class.name}] #{message}"
  end
end

module Serializable
  def to_json
    instance_variables.each_with_object({}) do |var, hash|
      hash[var.to_s.delete('@')] = instance_variable_get(var)
    end.to_json
  end

  def to_yaml
    # YAML serialization
  end
end

class User
  include Loggable       # User instances get log() and log_error()
  include Serializable   # User instances get to_json() and to_yaml()

  def initialize(name, email)
    @name = name
    @email = email
  end

  def save
    log("Saving user #{@name}")
    # ... save logic ...
  end
end

user = User.new("Alice", "alice@example.com")
user.log("Hello!")     # => [User] Hello!
user.save              # => [User] Saving user Alice
puts user.to_json      # => {"name":"Alice","email":"alice@example.com"}
```

**`extend` — adds module methods as class methods (or singleton methods):**
```ruby
module ClassMethods
  def find(id)
    puts "Finding #{name} with id #{id}"
  end

  def all
    puts "Returning all #{name} records"
  end
end

class Product
  extend ClassMethods  # Product.find(), Product.all()

  def initialize(name)
    @name = name
  end
end

Product.find(42)    # => "Finding Product with id 42"
Product.all         # => "Returning all Product records"

# extend on a single object (singleton methods):
obj = Object.new
obj.extend(Loggable)
obj.log("works on this specific object only")
```

**`prepend` — inserts module BEFORE the class in method lookup (for method wrapping):**
```ruby
module Timing
  def process
    start = Time.now
    result = super  # calls the original method in the class
    elapsed = Time.now - start
    puts "#{self.class}#process took #{elapsed}s"
    result
  end
end

class DataProcessor
  prepend Timing  # Timing#process is called FIRST, then DataProcessor#process via super

  def process
    sleep(0.1)
    "processed data"
  end
end

dp = DataProcessor.new
dp.process
# Output: DataProcessor#process took 0.1s
# Returns: "processed data"
```

**Method lookup order (Method Resolution Order — MRO):**
```ruby
module M1; end
module M2; end

class Parent; end
class Child < Parent
  include M1
  include M2
  prepend Timing
end

Child.ancestors
# => [Timing, Child, M2, M1, Parent, Object, Kernel, BasicObject]
#     ↑ prepend goes BEFORE class
#              ↑ class itself
#                   ↑ included modules (last included = first in chain)
#                          ↑ parent class
#                                  ↑ Object (root of all classes)
```

**The `included` and `extended` hooks:**
```ruby
module Hookable
  def self.included(base)
    puts "#{self} was included in #{base}"
    base.extend(ClassMethods)  # common pattern: add class methods on include
  end

  module ClassMethods
    def class_level_method
      puts "I'm a class method added via include!"
    end
  end

  def instance_level_method
    puts "I'm an instance method"
  end
end

class MyClass
  include Hookable  # triggers included hook
end

MyClass.class_level_method      # works!
MyClass.new.instance_level_method  # works!
```

---

### Metaprogramming Basics

Metaprogramming is writing code that writes code. Ruby's open classes and reflection make this natural.

**`method_missing` — handle calls to undefined methods:**
```ruby
class DynamicProxy
  def initialize(target)
    @target = target
  end

  def method_missing(method_name, *args, &block)
    if @target.respond_to?(method_name)
      puts "Delegating #{method_name} to target"
      @target.send(method_name, *args, &block)
    else
      super  # raise NoMethodError for truly missing methods
    end
  end

  # Always override respond_to_missing? alongside method_missing
  def respond_to_missing?(method_name, include_private = false)
    @target.respond_to?(method_name, include_private) || super
  end
end

proxy = DynamicProxy.new([1, 2, 3])
proxy.length    # => Delegating length to target => 3
proxy.reverse   # => Delegating reverse to target => [3, 2, 1]
```

**`define_method` — create methods dynamically:**
```ruby
class Model
  FIELDS = [:name, :email, :age]

  FIELDS.each do |field|
    # Dynamically define getter
    define_method(field) do
      instance_variable_get("@#{field}")
    end

    # Dynamically define setter
    define_method("#{field}=") do |value|
      instance_variable_set("@#{field}", value)
    end

    # Dynamically define predicate
    define_method("#{field}?") do
      !instance_variable_get("@#{field}").nil?
    end
  end
end

m = Model.new
m.name = "Alice"
m.name       # => "Alice"
m.name?      # => true
m.email?     # => false
```

**`class_eval` and `instance_eval`:**
```ruby
# class_eval — evaluate code in the context of a class
String.class_eval do
  def shout
    upcase + "!!!"
  end
end
"hello".shout  # => "HELLO!!!"

# instance_eval — evaluate code in the context of an object
obj = Object.new
obj.instance_eval do
  def greet
    "Hello from #{self}"
  end
end
obj.greet  # => "Hello from #<Object:0x...>"
```

**`send` and `public_send` — call methods by name:**
```ruby
class Calculator
  def add(a, b) = a + b
  def subtract(a, b) = a - b
  def multiply(a, b) = a * b

  private
  def secret() = "hidden"
end

calc = Calculator.new
operation = :add
calc.send(operation, 3, 4)         # => 7 (calls calc.add(3, 4))
calc.public_send(operation, 3, 4)  # => 7 (same, but respects visibility)
calc.send(:secret)                 # => "hidden" (bypasses private!)
# calc.public_send(:secret)        # => NoMethodError (respects private)
```

---

### Open Classes (Monkey Patching)

Ruby classes are never closed — you can reopen and modify any class at any time.

```ruby
# Adding methods to built-in classes
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

5.factorial   # => 120
7.prime?      # => true

# Adding methods to String
class String
  def palindrome?
    self == self.reverse
  end

  def word_count
    split.length
  end
end

"racecar".palindrome?           # => true
"hello world foo".word_count    # => 3
```

**Refinements (safer alternative to monkey patching — Ruby 2.0+):**
```ruby
module StringExtensions
  refine String do
    def palindrome?
      self == self.reverse
    end
  end
end

# Refinement is only active where explicitly used:
class MyClass
  using StringExtensions  # activate refinement in this scope only

  def check(word)
    word.palindrome?  # works here
  end
end

# "hello".palindrome?  # NoMethodError — refinement not active globally!
```

---

### attr_accessor, attr_reader, attr_writer

These are metaprogramming methods that generate getter/setter methods:

```ruby
class Person
  attr_reader :name        # generates: def name; @name; end
  attr_writer :email       # generates: def email=(val); @email = val; end
  attr_accessor :age       # generates both getter AND setter

  def initialize(name, email, age)
    @name = name
    @email = email
    @age = age
  end
end

p = Person.new("Alice", "alice@example.com", 30)
p.name          # => "Alice" (getter)
# p.name = "Bob"  # NoMethodError (no setter — attr_reader only)
p.email = "new@example.com"  # setter works (attr_writer)
# p.email       # NoMethodError (no getter — attr_writer only)
p.age           # => 30 (getter from attr_accessor)
p.age = 31      # setter works (attr_accessor)
```

**Custom attribute methods with validation:**
```ruby
class Product
  attr_reader :name, :price

  def name=(new_name)
    raise ArgumentError, "Name can't be blank" if new_name.nil? || new_name.strip.empty?
    @name = new_name.strip
  end

  def price=(new_price)
    raise ArgumentError, "Price must be positive" unless new_price.is_a?(Numeric) && new_price > 0
    @price = new_price.round(2)
  end

  def initialize(name, price)
    self.name = name    # uses the setter (with validation)
    self.price = price  # uses the setter (with validation)
  end
end
```

---

## 1.5 Error Handling, File I/O & Project Structure

> This section covers Ruby's exception handling system, file operations, and how Ruby projects are organized — the equivalent of C++'s preprocessor, header guards, and namespaces for project structure.

---

### Exception Handling

Ruby uses `begin/rescue/ensure/end` (equivalent to C++'s `try/catch/finally`).

```ruby
# Basic exception handling
begin
  result = 10 / 0
rescue ZeroDivisionError => e
  puts "Error: #{e.message}"  # => "Error: divided by 0"
end

# Multiple rescue clauses (most specific first)
begin
  data = JSON.parse(input)
  file = File.open(data["path"])
rescue JSON::ParserError => e
  puts "Invalid JSON: #{e.message}"
rescue Errno::ENOENT => e
  puts "File not found: #{e.message}"
rescue StandardError => e
  puts "Unexpected error: #{e.class} - #{e.message}"
ensure
  # Always runs (like finally) — cleanup code
  file&.close
end

# retry — re-execute the begin block
attempts = 0
begin
  attempts += 1
  connect_to_database
rescue ConnectionError => e
  retry if attempts < 3  # try up to 3 times
  raise  # re-raise after 3 failures
end

# raise — throw an exception
def withdraw(amount)
  raise ArgumentError, "Amount must be positive" if amount <= 0
  raise InsufficientFundsError, "Not enough balance" if amount > @balance
  @balance -= amount
end

# Inline rescue (for simple cases)
value = Integer(input) rescue 0  # returns 0 if conversion fails
```

**Exception hierarchy:**
```
Exception
├── NoMemoryError
├── ScriptError
│   ├── LoadError
│   ├── SyntaxError
│   └── NotImplementedError
├── SignalException
│   └── Interrupt (Ctrl+C)
├── SystemExit
└── StandardError          ← rescue without specifying class catches this
    ├── ArgumentError
    ├── IOError
    ├── IndexError
    ├── KeyError
    ├── NameError
    │   └── NoMethodError
    ├── RangeError
    ├── RuntimeError       ← default when you just say `raise "message"`
    ├── TypeError
    ├── ZeroDivisionError
    └── ... (many more)
```

**Custom exceptions:**
```ruby
# Define custom exception classes
module MyApp
  class Error < StandardError; end  # base error for the app

  class ValidationError < Error
    attr_reader :field, :value

    def initialize(field, value, message = nil)
      @field = field
      @value = value
      super(message || "Validation failed for #{field}: #{value}")
    end
  end

  class NotFoundError < Error
    attr_reader :resource, :id

    def initialize(resource, id)
      @resource = resource
      @id = id
      super("#{resource} with id #{id} not found")
    end
  end

  class AuthenticationError < Error; end
  class AuthorizationError < Error; end
end

# Usage
def find_user(id)
  user = database.find(id)
  raise MyApp::NotFoundError.new("User", id) unless user
  user
end

begin
  find_user(999)
rescue MyApp::NotFoundError => e
  puts e.message       # => "User with id 999 not found"
  puts e.resource      # => "User"
  puts e.id            # => 999
  puts e.backtrace.first(5)  # stack trace
end
```

**Exception safety patterns:**
```ruby
# Ensure cleanup with ensure
def process_file(path)
  file = File.open(path)
  # ... process ...
  file.read
ensure
  file&.close  # always close, even if exception occurred
end

# Better: use blocks (RAII-like pattern in Ruby)
def process_file(path)
  File.open(path) do |file|
    file.read
    # file is automatically closed when block ends
  end
end

# throw/catch — non-local exit (NOT for errors, for flow control)
result = catch(:done) do
  (1..100).each do |i|
    (1..100).each do |j|
      throw :done, [i, j] if i * j == 42  # immediately exit both loops
    end
  end
end
puts result.inspect  # => [1, 42] or [2, 21] etc.
```

---

### File I/O

```ruby
# Reading files
content = File.read("data.txt")              # entire file as string
lines = File.readlines("data.txt")           # array of lines
lines = File.readlines("data.txt", chomp: true)  # without newlines

# Reading with block (auto-closes file)
File.open("data.txt", "r") do |file|
  file.each_line do |line|
    puts line.strip
  end
end

# Writing files
File.write("output.txt", "Hello, World!")    # overwrite
File.write("output.txt", "more", mode: "a") # append

File.open("output.txt", "w") do |file|
  file.puts "Line 1"
  file.puts "Line 2"
  file.print "No newline"
  file.write "Raw bytes"
end

# File modes
# "r"  — read only (default)
# "w"  — write only (truncates existing file)
# "a"  — append (write to end)
# "r+" — read and write
# "w+" — read and write (truncates)
# "a+" — read and append

# File operations
File.exist?("path")       # check existence
File.size("path")         # file size in bytes
File.delete("path")       # delete file
File.rename("old", "new") # rename/move
File.directory?("path")   # is it a directory?
File.basename("/a/b/c.txt")  # => "c.txt"
File.dirname("/a/b/c.txt")   # => "/a/b"
File.extname("file.rb")      # => ".rb"
File.join("path", "to", "file.rb")  # => "path/to/file.rb"

# Directory operations
Dir.mkdir("new_dir")
Dir.entries(".")          # list directory contents
Dir.glob("**/*.rb")       # find all .rb files recursively
Dir.pwd                   # current working directory

# Tempfiles (auto-cleanup)
require 'tempfile'
Tempfile.create("prefix") do |file|
  file.write("temporary data")
  file.path  # => "/tmp/prefix20210101-1234-abc.tmp"
end  # file is automatically deleted
```

---

### Require, Load, and Project Structure

Ruby's equivalent of `#include` and header guards:

```ruby
# require — loads a file ONCE (like #include with header guards)
require 'json'                    # load from $LOAD_PATH (gems, stdlib)
require_relative 'lib/my_class'   # load relative to current file

# require won't load the same file twice:
require 'json'  # loaded
require 'json'  # skipped (already loaded)

# load — loads a file EVERY time (like #include without guards)
load 'config.rb'   # always re-executes the file
load 'config.rb'   # executes again

# autoload — lazy loading (loads file when constant is first accessed)
autoload :HeavyClass, 'lib/heavy_class'
# HeavyClass file is only loaded when HeavyClass is first used
```

**Typical Ruby project structure:**
```
my_project/
├── Gemfile              # dependencies (like package.json or CMakeLists.txt)
├── Gemfile.lock         # locked dependency versions
├── Rakefile             # task runner (like Makefile)
├── my_project.gemspec   # gem specification (if building a gem)
├── lib/                 # source code
│   ├── my_project.rb   # entry point (requires everything)
│   └── my_project/
│       ├── version.rb
│       ├── models/
│       │   ├── user.rb
│       │   └── order.rb
│       ├── services/
│       │   ├── payment_service.rb
│       │   └── notification_service.rb
│       └── utils/
│           └── validator.rb
├── spec/                # tests (RSpec)
│   ├── spec_helper.rb
│   ├── models/
│   │   └── user_spec.rb
│   └── services/
│       └── payment_service_spec.rb
├── bin/                 # executables
│   └── my_project
└── README.md
```

**Entry point pattern:**
```ruby
# lib/my_project.rb — loads all components
require_relative 'my_project/version'
require_relative 'my_project/models/user'
require_relative 'my_project/models/order'
require_relative 'my_project/services/payment_service'
require_relative 'my_project/services/notification_service'

module MyProject
  class Error < StandardError; end
  # ...
end
```

---

### Gems and Bundler

Gems are Ruby's package system (like npm packages or C++ libraries).

```ruby
# Gemfile — declare dependencies
source "https://rubygems.org"

gem "rails", "~> 7.0"        # ~> means >= 7.0 and < 8.0
gem "pg", ">= 1.0"           # PostgreSQL adapter
gem "redis", "~> 5.0"
gem "sidekiq"                 # latest version

group :development, :test do
  gem "rspec", "~> 3.12"
  gem "rubocop", require: false
end

group :test do
  gem "factory_bot"
  gem "faker"
end
```

```bash
# Install dependencies
bundle install

# Run code with bundled gems
bundle exec ruby my_script.rb
bundle exec rspec

# Add a new gem
bundle add nokogiri
```

---

### Frozen String Literals and Performance

```ruby
# At the top of a file — makes all string literals frozen (immutable)
# frozen_string_literal: true

# This prevents accidental string mutation and improves performance
# (frozen strings can be shared/reused by the interpreter)

str = "hello"
# str << " world"  # => FrozenError!
str = str + " world"  # OK — creates a new string

# Without the magic comment, use .freeze explicitly:
CONSTANT_KEY = "my_key".freeze
```

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Dynamic Typing | Ruby determines types at runtime; duck typing over explicit types |
| Everything is an Object | Even integers, nil, and true/false are objects with methods |
| Symbols vs Strings | Symbols (`:name`) are immutable and interned; use for identifiers and hash keys |
| Truthiness | Only `nil` and `false` are falsy — `0`, `""`, `[]` are all truthy! |
| Blocks/Procs/Lambdas | Blocks are syntax, Procs/Lambdas are objects; lambdas check arity |
| Enumerable | Include `Enumerable` + define `each` to get 50+ collection methods free |
| Modules (Mixins) | `include` for instance methods, `extend` for class methods, `prepend` for wrapping |
| Method Lookup | prepended modules → class → included modules → parent → Object → BasicObject |
| Metaprogramming | `define_method`, `method_missing`, `send`, open classes — powerful but use carefully |
| Memory | Automatic GC; use `freeze` for immutability; `WeakRef` for caches |
| Exceptions | `begin/rescue/ensure`; inherit from `StandardError` for custom errors |
| Project Structure | `lib/` for source, `spec/` for tests, `Gemfile` for dependencies |
| Require | `require` loads once, `require_relative` for local files, `load` always re-executes |
| Refinements | Safer alternative to monkey patching — scoped class modifications |

---

## C++ to Ruby Quick Reference

| C++ Concept | Ruby Equivalent |
|-------------|-----------------|
| `std::vector` | `Array` |
| `std::unordered_map` | `Hash` |
| `std::unordered_set` | `Set` (require 'set') |
| `std::map` (ordered) | `Hash` (insertion-ordered since Ruby 1.9) or SortedSet |
| `std::stack` | `Array` with `push`/`pop` |
| `std::queue` | `Array` with `push`/`shift` or `Queue` class |
| `std::priority_queue` | No built-in; use sorted array or gem |
| Templates | Duck typing + modules (no generics needed) |
| `#include` | `require` / `require_relative` |
| Header guards | Not needed (require loads once) |
| Namespaces | Modules |
| `new`/`delete` | Automatic (GC handles deallocation) |
| Smart pointers | Not needed (GC + references) |
| RAII | Blocks with `ensure` or `File.open { }` pattern |
| `const` | `.freeze` / `# frozen_string_literal: true` |
| Function pointers | Procs, Lambdas, method objects |
| `std::function` | `Proc` or `Lambda` |
| Operator overloading | Define methods like `def +(other)`, `def <=>(other)` |
| Multiple inheritance | Mixins (`include` multiple modules) |
| Abstract class | Module with `raise NotImplementedError` in methods |
| Interface | Module defining method signatures (duck typing) |
| `static` members | Class variables (`@@var`) or class methods (`self.method`) |
| `friend` | No equivalent (use `send` to bypass visibility if needed) |
| Destructor | `ObjectSpace.define_finalizer` (rarely used; prefer ensure/blocks) |

---

## Interview Tips for Module 1

1. **Know Ruby's object model** — everything is an object, single inheritance + mixins, method lookup chain (ancestors). Be able to explain `include` vs `extend` vs `prepend`.

2. **Understand blocks, Procs, and Lambdas** — know the differences (arity checking, return behavior). Be able to write methods that accept blocks with `yield` or `&block`.

3. **Enumerable mastery** — know `map`, `select`, `reduce`, `flat_map`, `group_by`, `each_with_object`, `tally`. These replace most explicit loops in idiomatic Ruby.

4. **Symbol vs String** — explain when to use each, why symbols are faster for hash keys, and the memory implications.

5. **Duck typing** — Ruby doesn't check types, it checks behavior. "If it walks like a duck and quacks like a duck, it's a duck." Explain `respond_to?` over `is_a?`.

6. **Metaprogramming** — understand `method_missing`, `define_method`, `send`, `respond_to_missing?`. Know when metaprogramming is appropriate vs when it hurts readability.

7. **Exception handling** — know the exception hierarchy, why you should rescue `StandardError` not `Exception`, and the `retry` pattern for transient failures.

8. **Frozen objects and immutability** — explain `freeze`, `frozen_string_literal: true`, and why immutability helps with thread safety and performance.

9. **Comparable and Enumerable** — know how to include these modules and what you need to implement (`<=>` for Comparable, `each` for Enumerable).

10. **Memory and GC** — understand that Ruby uses mark-and-sweep GC, variables hold references (not values), and how to avoid memory leaks (unsubscribe patterns, WeakRef).

11. **Method visibility** — `public`, `private`, `protected` in Ruby. Know that `private` means "can only be called without an explicit receiver" (different from C++).

12. **Idiomatic Ruby** — prefer iterators over loops, use guard clauses (`return if`), use `?` and `!` naming conventions, prefer symbols for hash keys, use `&.` safe navigation.
