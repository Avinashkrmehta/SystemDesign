# Module 9: Concurrency & Multithreading in Ruby

> Concurrency is the ability of a program to manage multiple tasks at the same time. Ruby provides several mechanisms for concurrency — from native Threads and Fibers to the modern Ractor model introduced in Ruby 3.0. Understanding Ruby's concurrency model, including the Global VM Lock (GVL), is essential for building performant, correct, and scalable systems. This module covers threading primitives, synchronization, cooperative concurrency, true parallelism, and practical patterns used in production Ruby applications.

---

## 9.1 Thread Basics

> A **Thread** is the smallest unit of execution that can be scheduled by the operating system. Ruby's `Thread` class provides a way to create and manage threads within a single process. All threads share the same memory space (heap, global data) but each has its own stack and program counter. Ruby threads are native OS threads (since Ruby 1.9), but their parallel execution is constrained by the Global VM Lock (GVL).

---

### Thread Creation and Joining

**Creating a thread** requires passing a block to `Thread.new`. The thread begins executing the block immediately upon creation.

```ruby
# Simple function to run in a thread
def greet(name)
  puts "Hello from thread! Name: #{name}"
end

# Create a thread that runs greet()
t1 = Thread.new { greet("Alice") }

# Main thread continues executing here...
puts "Hello from main thread!"

# MUST join to wait for t1 to finish
t1.join  # blocks until t1 finishes
```

**What happens if you don't join?**

Unlike C++, Ruby won't crash if a thread isn't joined. However, if the main thread exits, all other threads are **killed immediately** — their work is lost.

```ruby
def dangerous
  t = Thread.new do
    sleep(2)
    puts "This may never print!"
  end
  # t is not joined — if main exits, t is killed
end

dangerous
# Program exits immediately — thread is killed before it prints
```

**join and value:**

| Method | Behavior | When to use |
|--------|----------|-------------|
| `join` | Blocks the calling thread until the target thread finishes; returns the thread | When you need to ensure the thread completes |
| `join(timeout)` | Blocks for at most `timeout` seconds; returns `nil` on timeout | When you can't afford to block indefinitely |
| `value` | Blocks until the thread finishes and returns the **last expression** evaluated in the block | When you need the thread's result |

```ruby
# join — wait for thread to complete
def example_join
  t = Thread.new do
    sleep(2)
    puts "Thread finished work"
  end

  puts "Waiting for thread..."
  t.join  # blocks here for ~2 seconds
  puts "Thread done, continuing"
end

# value — get the thread's return value
def example_value
  t = Thread.new do
    sleep(1)
    42  # last expression is the return value
  end

  puts "Computing in background..."
  result = t.value  # blocks until thread finishes, returns 42
  puts "Result: #{result}"  # => Result: 42
end

# join with timeout
def example_timeout
  t = Thread.new { sleep(10) }

  if t.join(2)  # wait at most 2 seconds
    puts "Thread finished in time"
  else
    puts "Thread timed out!"  # this prints — 2 < 10
  end
end
```

**Passing arguments to threads:**

Arguments are passed directly to `Thread.new` and received as block parameters. Ruby **captures the values at thread creation time** (by reference for objects, but the binding is immediate).

```ruby
# Passing arguments via block parameters
t = Thread.new("Alice", 25) do |name, age|
  puts "Name: #{name}, Age: #{age}"
end
t.join

# CAUTION: Closures capture variables by reference
counter = 0
threads = 5.times.map do |i|
  Thread.new do
    # 'i' is the block parameter — safe, each thread gets its own copy
    # 'counter' is a shared closure variable — NOT safe without synchronization
    counter += 1  # DATA RACE!
  end
end
threads.each(&:join)
puts counter  # Might not be 5 due to race condition!
```

**Creating multiple threads:**

```ruby
# Launch multiple threads and collect results
threads = 10.times.map do |i|
  Thread.new(i) do |thread_id|
    sleep(rand * 0.5)
    "Result from thread #{thread_id}"
  end
end

# Collect all results
results = threads.map(&:value)
results.each { |r| puts r }
```

---

### The Global VM Lock (GVL / GIL)

The **Global VM Lock** (GVL), historically called the Global Interpreter Lock (GIL), is the single most important concept in Ruby concurrency. It is a mutex held by the Ruby VM that **prevents multiple threads from executing Ruby code simultaneously**.

**What the GVL means:**

```
Without GVL (true parallelism):
  Thread 1: [====Ruby====][====Ruby====]
  Thread 2: [====Ruby====][====Ruby====]
  (Both execute Ruby code at the same time on different CPU cores)

With GVL (Ruby's reality):
  Thread 1: [====Ruby====]............[====Ruby====]
  Thread 2: .............[====Ruby====].............[====Ruby====]
  (Only ONE thread executes Ruby code at any given moment)
```

**Key implications:**

| Aspect | Impact |
|--------|--------|
| CPU-bound work | Threads provide NO speedup — only one thread runs Ruby at a time |
| I/O-bound work | Threads DO help — the GVL is released during I/O operations |
| Thread safety | The GVL does NOT make your code thread-safe — context switches happen between Ruby operations |
| C extensions | Well-written C extensions release the GVL during computation |

**When the GVL is released:**

```ruby
# The GVL is released during blocking I/O operations:
# - File I/O (File.read, File.write)
# - Network I/O (HTTP requests, socket operations)
# - sleep()
# - Waiting on subprocesses
# - Some C extension operations (e.g., database queries)

# This means I/O-bound programs DO benefit from threads:
require 'net/http'
require 'uri'

urls = [
  "https://example.com",
  "https://example.org",
  "https://example.net"
]

# Sequential — slow (each request waits for the previous one)
start = Time.now
urls.each { |url| Net::HTTP.get(URI(url)) }
puts "Sequential: #{Time.now - start}s"

# Concurrent — fast (requests happen in parallel during I/O)
start = Time.now
threads = urls.map do |url|
  Thread.new { Net::HTTP.get(URI(url)) }
end
threads.each(&:join)
puts "Concurrent: #{Time.now - start}s"
# Concurrent version is ~3x faster because the GVL is released during network I/O
```

**GVL vs true parallelism — when to use what:**

| Workload | Threads (GVL) | Ractor (Ruby 3.0+) | Process (fork) |
|----------|--------------|---------------------|----------------|
| I/O-bound (HTTP, DB, files) | ✅ Great | ✅ Works | ✅ Works |
| CPU-bound (computation) | ❌ No speedup | ✅ True parallelism | ✅ True parallelism |
| Shared mutable state | ✅ Easy (with sync) | ❌ No sharing | ❌ No sharing |
| Memory efficiency | ✅ Shared memory | ✅ Shared memory | ❌ Separate memory |

---

### Thread Lifecycle and Status

Every thread has a lifecycle and can be in one of several states.

```ruby
t = Thread.new do
  sleep(5)
  "done"
end

# Thread status
puts t.status   # => "sleep" (sleeping), "run" (running), "aborting", false (finished), nil (exception)
puts t.alive?   # => true (running or sleeping)
puts t.stop?    # => true (sleeping or dead)

# Thread identity
puts t.object_id          # unique object ID
puts Thread.current       # the currently executing thread
puts Thread.main          # the main thread
puts Thread.list          # all living threads
puts Thread.list.count    # number of living threads
```

**Thread states:**

| Status | Meaning |
|--------|---------|
| `"run"` | Thread is currently running |
| `"sleep"` | Thread is sleeping or waiting (I/O, `sleep`, `Mutex#lock`, etc.) |
| `"aborting"` | Thread is being killed |
| `false` | Thread terminated normally |
| `nil` | Thread terminated with an exception |

**Controlling threads:**

```ruby
t = Thread.new do
  loop do
    puts "Working..."
    sleep(1)
  end
end

sleep(3)

# Kill a thread (use with caution — no cleanup!)
t.kill    # also: t.terminate, t.exit
# or
Thread.kill(t)

# Raise an exception in a thread
t.raise("Stop!")  # raises RuntimeError in the thread

# Thread priority (hint to the scheduler — not guaranteed)
t.priority = 3     # higher = more CPU time (default is 0)
puts t.priority    # => 3
```

**Thread-local variables:**

Each thread can have its own local storage, useful for per-thread context (like request IDs in a web server).

```ruby
# Thread-local variables using thread-local storage
Thread.current[:request_id] = "abc-123"
puts Thread.current[:request_id]  # => "abc-123"

# Thread-local variables using thread_variable_set/get (fiber-local vs thread-local)
Thread.current.thread_variable_set(:user_id, 42)
puts Thread.current.thread_variable_get(:user_id)  # => 42

# Difference:
# Thread.current[:key]              — fiber-local (different per Fiber within the same Thread)
# Thread.current.thread_variable_get(:key) — thread-local (same across all Fibers in the Thread)
```

---

### Exception Handling in Threads

By default, exceptions in threads are **silently swallowed** — the thread dies, but the main thread doesn't know unless it calls `join` or `value`.

```ruby
# BAD: Exception is silently lost
t = Thread.new do
  raise "Something went wrong!"
end
sleep(1)
puts "Main thread continues — never knew about the exception"

# GOOD: join or value re-raises the exception in the calling thread
t = Thread.new do
  raise "Something went wrong!"
end

begin
  t.join  # re-raises the exception here
rescue => e
  puts "Caught from thread: #{e.message}"
end

# ALTERNATIVE: Make all thread exceptions fatal
Thread.abort_on_exception = true
# Now ANY unhandled exception in ANY thread kills the entire program

# Or per-thread:
t = Thread.new { raise "boom" }
t.abort_on_exception = true

# Or use Thread.report_on_exception (Ruby 2.5+, default: true)
# Prints a warning when a thread dies with an exception
Thread.report_on_exception = true  # default since Ruby 2.5
```

**Safe thread pattern with exception handling:**

```ruby
def safe_thread(&block)
  Thread.new do
    block.call
  rescue => e
    puts "[Thread Error] #{e.class}: #{e.message}"
    puts e.backtrace.first(5).join("\n")
    nil  # return nil on error
  end
end

t = safe_thread { 1 / 0 }
result = t.value  # => nil (exception was caught and logged)
```

---

### Practical Example: Parallel Computation

```ruby
# Parallel sum: divide array into chunks, sum each in a separate thread
# NOTE: Due to the GVL, this won't be faster than sequential for CPU-bound work
# This is for demonstration — use Ractor for true CPU parallelism

data = (1..10_000_000).to_a
num_threads = 4
chunk_size = data.size / num_threads

threads = num_threads.times.map do |i|
  start_idx = i * chunk_size
  end_idx = (i == num_threads - 1) ? data.size : start_idx + chunk_size
  chunk = data[start_idx...end_idx]

  Thread.new(chunk) do |my_chunk|
    my_chunk.sum  # each thread sums its own chunk
  end
end

total = threads.map(&:value).sum
puts "Sum: #{total}"
# Expected: 50000005000000

# For I/O-bound parallel work (where threads DO help):
require 'net/http'

def fetch_url(url)
  uri = URI(url)
  Net::HTTP.get(uri)
end

urls = ["https://example.com"] * 10

# Parallel fetch — genuinely faster due to I/O concurrency
start = Time.now
threads = urls.map { |url| Thread.new { fetch_url(url) } }
results = threads.map(&:value)
puts "Fetched #{results.size} pages in #{Time.now - start}s"
```

---

### Common Pitfalls with Ruby Threads

1. **Assuming threads give CPU parallelism** — the GVL prevents it for Ruby code
2. **Not joining threads** — main thread exits, killing all child threads
3. **Shared mutable state without synchronization** — data races
4. **Silent exceptions** — threads die quietly unless you join/value or set `abort_on_exception`
5. **Closure variable capture** — threads share closure variables by reference

```ruby
# PITFALL: Closure variable capture
results = []
threads = 5.times.map do |i|
  Thread.new do
    results << i * 2  # RACE CONDITION — Array#<< is not thread-safe!
  end
end
threads.each(&:join)
puts results.inspect  # May have missing or duplicate values!

# FIX: Use a Mutex or thread-safe data structure
mutex = Mutex.new
results = []
threads = 5.times.map do |i|
  Thread.new do
    mutex.synchronize { results << i * 2 }
  end
end
threads.each(&:join)
puts results.sort.inspect  # => [0, 2, 4, 6, 8]
```

---


## 9.2 Synchronization Primitives

> When multiple threads access shared data, **synchronization** is required to prevent data races and ensure correctness. A **data race** occurs when two or more threads access the same memory location concurrently, at least one access is a write, and there is no synchronization. Even with the GVL, Ruby threads can interleave between any two Ruby operations, so shared mutable state still requires protection.

---

### Mutex

A **Mutex** (mutual exclusion) is the most fundamental synchronization primitive in Ruby. It ensures that only one thread can execute a critical section at a time.

**Basic Mutex usage:**

```ruby
shared_counter = 0
mutex = Mutex.new

# WITHOUT mutex — data race
threads = 10.times.map do
  Thread.new do
    10_000.times { shared_counter += 1 }  # NOT atomic!
  end
end
threads.each(&:join)
puts shared_counter  # Often less than 100_000 due to race conditions!

# WITH mutex — correct
shared_counter = 0
mutex = Mutex.new

threads = 10.times.map do
  Thread.new do
    10_000.times do
      mutex.synchronize { shared_counter += 1 }
    end
  end
end
threads.each(&:join)
puts shared_counter  # Always 100_000
```

**Why is `counter += 1` not safe even with the GVL?**

`counter += 1` is actually three operations:
1. **Read** the current value of `counter`
2. **Add** 1 to the value
3. **Write** the new value back to `counter`

The GVL can be released between any of these steps (at a context switch point), allowing another thread to read the stale value.

**Mutex methods:**

```ruby
mutex = Mutex.new

# lock / unlock — manual locking (error-prone, prefer synchronize)
mutex.lock
# ... critical section ...
mutex.unlock

# synchronize — RAII-style block (ALWAYS prefer this)
mutex.synchronize do
  # ... critical section ...
  # mutex is automatically unlocked when the block exits
  # even if an exception is raised!
end

# try_lock — non-blocking attempt to acquire the lock
if mutex.try_lock
  begin
    # ... critical section ...
  ensure
    mutex.unlock
  end
else
  puts "Lock is busy, doing something else"
end

# locked? — check if the mutex is currently locked
puts mutex.locked?  # => true/false

# owned? — check if the current thread owns the lock
puts mutex.owned?   # => true/false
```

**IMPORTANT: Never call `synchronize` on a Mutex you already hold — it will deadlock!**

```ruby
mutex = Mutex.new

mutex.synchronize do
  # BAD: This will deadlock! Ruby's Mutex is NOT re-entrant
  mutex.synchronize do
    puts "This never executes"
  end
end
# ThreadError: deadlock; recursive locking
```

---

### Monitor

A **Monitor** is a re-entrant mutex — the same thread can lock it multiple times without deadlocking. It must unlock the same number of times before other threads can acquire it.

```ruby
require 'monitor'

# Method 1: Include Monitor as a mixin
class SafeCounter
  include MonitorMixin

  def initialize
    super  # MUST call super to initialize the monitor
    @count = 0
  end

  def increment
    synchronize do
      @count += 1
      log_change  # calls another synchronized method — safe with Monitor!
    end
  end

  def value
    synchronize { @count }
  end

  private

  def log_change
    synchronize do  # re-entrant — same thread can lock again
      puts "Count changed to #{@count}"
    end
  end
end

counter = SafeCounter.new
threads = 5.times.map { Thread.new { 100.times { counter.increment } } }
threads.each(&:join)
puts counter.value  # => 500

# Method 2: Use Monitor.new directly
monitor = Monitor.new
shared_data = []

threads = 5.times.map do |i|
  Thread.new do
    monitor.synchronize do
      shared_data << i
      # Can call synchronize again — re-entrant
      monitor.synchronize do
        puts "Added #{i}, total: #{shared_data.size}"
      end
    end
  end
end
threads.each(&:join)

# Method 3: Extend an object with MonitorMixin
hash = {}
hash.extend(MonitorMixin)

hash.synchronize do
  hash[:key] = "value"
end
```

**When to use Monitor vs Mutex:**

| Feature | `Mutex` | `Monitor` |
|---------|---------|-----------|
| Re-entrant | ❌ No (deadlocks) | ✅ Yes |
| Performance | Faster (simpler) | Slightly slower |
| Condition variables | Use `ConditionVariable` | Use `Monitor::ConditionVariable` (built-in) |
| Mixin support | No | Yes (`MonitorMixin`) |
| Use case | Simple critical sections | Recursive locking, complex objects |

**Rule of thumb:** Use `Mutex` for simple cases. Use `Monitor` when methods that hold the lock need to call other methods that also need the lock.

---

### ConditionVariable

A **ConditionVariable** allows threads to wait for a specific condition to become true. It's used with a Mutex to avoid busy-waiting (spinning in a loop checking a condition).

**The pattern:**
1. **Waiting thread:** locks a mutex, checks a condition, and if false, waits (releasing the mutex)
2. **Notifying thread:** locks the mutex, changes the shared state, signals the waiting thread

```ruby
mutex = Mutex.new
cv = ConditionVariable.new
queue = []
finished = false

# Producer: generates data and notifies consumer
producer = Thread.new do
  10.times do |i|
    mutex.synchronize do
      queue << i
      puts "Produced: #{i}"
      cv.signal  # wake up one waiting consumer
    end
    sleep(0.1)
  end

  mutex.synchronize do
    finished = true
    cv.broadcast  # wake up ALL waiting consumers
  end
end

# Consumer: waits for data and processes it
consumer = Thread.new do
  loop do
    mutex.synchronize do
      # Wait until there's data OR producer is finished
      while queue.empty? && !finished
        cv.wait(mutex)  # releases mutex, sleeps, re-acquires mutex on wake
      end

      break if queue.empty? && finished

      # Process all available data
      until queue.empty?
        value = queue.shift
        puts "Consumed: #{value}"
      end
    end
  end
end

producer.join
consumer.join
```

**How cv.wait(mutex) works internally:**

```ruby
# cv.wait(mutex) does:
# 1. Atomically releases the mutex and suspends the thread
# 2. When signaled (or spuriously woken), re-acquires the mutex
# 3. Returns (caller should re-check the condition)

# cv.wait(mutex, timeout) — wait with a timeout (seconds)
cv.wait(mutex, 5.0)  # wait at most 5 seconds
```

**signal vs broadcast:**

| Method | Behavior | When to use |
|--------|----------|-------------|
| `signal` | Wakes up ONE waiting thread | When only one thread can make progress (e.g., one item in queue) |
| `broadcast` | Wakes up ALL waiting threads | When multiple threads might be able to proceed, or when the condition changes for everyone |

**Spurious wakeups — always use a loop:**

```ruby
# BAD: No loop — vulnerable to spurious wakeups
mutex.synchronize do
  cv.wait(mutex)
  # Thread might wake up even though the condition isn't true!
  process(queue.shift)  # queue might still be empty!
end

# GOOD: Loop re-checks the condition after every wakeup
mutex.synchronize do
  while queue.empty?
    cv.wait(mutex)
  end
  # Guaranteed: queue is not empty here
  process(queue.shift)
end
```

**Monitor with built-in ConditionVariable:**

```ruby
require 'monitor'

class BoundedBuffer
  include MonitorMixin

  def initialize(capacity)
    super()
    @buffer = []
    @capacity = capacity
    @not_full = new_cond   # Monitor's built-in condition variable
    @not_empty = new_cond
  end

  def produce(item)
    synchronize do
      @not_full.wait_while { @buffer.size >= @capacity }
      @buffer << item
      @not_empty.signal
    end
  end

  def consume
    synchronize do
      @not_empty.wait_while { @buffer.empty? }
      item = @buffer.shift
      @not_full.signal
      item
    end
  end

  def size
    synchronize { @buffer.size }
  end
end
```

---

### Thread-Safe Data Structures: Queue and SizedQueue

Ruby's standard library provides **thread-safe** queue implementations that handle all synchronization internally — no need for external mutexes.

**Queue — unbounded thread-safe FIFO queue:**

```ruby
queue = Thread::Queue.new  # or just Queue.new

# Producer
producer = Thread.new do
  10.times do |i|
    queue << i  # also: queue.push(i), queue.enq(i)
    puts "Produced: #{i}"
    sleep(0.1)
  end
  queue.close  # signal that no more items will be added
end

# Consumer
consumer = Thread.new do
  # pop blocks if queue is empty, returns nil when closed and empty
  while (item = queue.pop)  # also: queue.shift, queue.deq
    puts "Consumed: #{item}"
  end
  puts "Queue closed, consumer exiting"
end

producer.join
consumer.join
```

**SizedQueue — bounded thread-safe FIFO queue:**

```ruby
# SizedQueue blocks producers when full (backpressure)
queue = Thread::SizedQueue.new(5)  # capacity of 5

producer = Thread.new do
  20.times do |i|
    queue << i  # blocks if queue is full (size >= 5)
    puts "Produced: #{i} (queue size: #{queue.size})"
  end
  queue.close
end

consumer = Thread.new do
  while (item = queue.pop)  # blocks if queue is empty
    puts "Consumed: #{item}"
    sleep(0.2)  # slow consumer — producer will block when queue fills up
  end
end

producer.join
consumer.join
```

**Queue methods:**

| Method | Behavior |
|--------|----------|
| `push(item)` / `<<` / `enq` | Add item (blocks on SizedQueue if full) |
| `pop` / `shift` / `deq` | Remove and return item (blocks if empty) |
| `pop(true)` | Non-blocking pop — raises `ThreadError` if empty |
| `close` | Prevents further pushes; pop returns `nil` when empty |
| `closed?` | Check if the queue is closed |
| `size` / `length` | Number of items in the queue |
| `empty?` | Check if the queue is empty |
| `num_waiting` | Number of threads waiting on the queue |
| `clear` | Remove all items |

**Queue vs manual Mutex + ConditionVariable:**

```ruby
# Manual approach — error-prone, verbose
mutex = Mutex.new
cv = ConditionVariable.new
buffer = []

# Push
mutex.synchronize do
  buffer << item
  cv.signal
end

# Pop
mutex.synchronize do
  while buffer.empty?
    cv.wait(mutex)
  end
  buffer.shift
end

# Queue approach — simple, correct, built-in
queue = Queue.new
queue << item     # push
item = queue.pop  # pop (blocks if empty)

# ALWAYS prefer Queue/SizedQueue over manual synchronization for producer-consumer patterns
```

---

### Deadlock Prevention

A **deadlock** occurs when two or more threads are each waiting for a lock held by the other, creating a circular dependency where no thread can proceed.

```ruby
# DEADLOCK EXAMPLE:
mutex_a = Mutex.new
mutex_b = Mutex.new

thread1 = Thread.new do
  mutex_a.synchronize do
    sleep(0.1)  # give thread2 time to lock mutex_b
    mutex_b.synchronize do
      puts "Thread 1: got both locks"
    end
  end
end

thread2 = Thread.new do
  mutex_b.synchronize do
    sleep(0.1)  # give thread1 time to lock mutex_a
    mutex_a.synchronize do
      puts "Thread 2: got both locks"
    end
  end
end

# Thread 1 holds A, waits for B
# Thread 2 holds B, waits for A
# → DEADLOCK — both threads blocked forever
# Ruby will detect this and raise: fatal: No live threads left. Deadlock?
thread1.join
thread2.join
```

**Prevention Strategy 1: Consistent lock ordering**

Always acquire mutexes in the same order across all threads.

```ruby
mutex_a = Mutex.new
mutex_b = Mutex.new

# GOOD: Both threads lock A first, then B
thread1 = Thread.new do
  mutex_a.synchronize do
    mutex_b.synchronize do
      puts "Thread 1: got both locks"
    end
  end
end

thread2 = Thread.new do
  mutex_a.synchronize do  # same order: A first
    mutex_b.synchronize do  # then B
      puts "Thread 2: got both locks"
    end
  end
end

thread1.join
thread2.join
```

**Prevention Strategy 2: try_lock with backoff**

```ruby
mutex_a = Mutex.new
mutex_b = Mutex.new

def acquire_both(m1, m2)
  loop do
    m1.lock
    if m2.try_lock
      return  # got both locks
    else
      m1.unlock  # release first lock and retry
      sleep(rand * 0.01)  # random backoff to avoid livelock
    end
  end
end

thread1 = Thread.new do
  acquire_both(mutex_a, mutex_b)
  begin
    puts "Thread 1: got both locks"
  ensure
    mutex_b.unlock
    mutex_a.unlock
  end
end

thread2 = Thread.new do
  acquire_both(mutex_a, mutex_b)
  begin
    puts "Thread 2: got both locks"
  ensure
    mutex_b.unlock
    mutex_a.unlock
  end
end

thread1.join
thread2.join
```

**Prevention Strategy 3: Lock hierarchy**

Assign a numeric level to each mutex. A thread can only lock a mutex with a LOWER level than the lowest mutex it currently holds.

```ruby
# Conceptual hierarchy:
# Level 3: database_mutex
# Level 2: cache_mutex
# Level 1: log_mutex
# Rule: always lock higher levels first

class HierarchicalMutex
  attr_reader :level

  def initialize(level)
    @level = level
    @mutex = Mutex.new
  end

  def synchronize(&block)
    current_level = Thread.current[:mutex_level] || Float::INFINITY
    if @level >= current_level
      raise "Lock hierarchy violation! Cannot lock level #{@level} while holding level #{current_level}"
    end

    @mutex.synchronize do
      old_level = Thread.current[:mutex_level]
      Thread.current[:mutex_level] = @level
      begin
        block.call
      ensure
        Thread.current[:mutex_level] = old_level
      end
    end
  end
end

db_mutex = HierarchicalMutex.new(3)
cache_mutex = HierarchicalMutex.new(2)
log_mutex = HierarchicalMutex.new(1)

# GOOD: database (3) → cache (2) → log (1)
db_mutex.synchronize do
  cache_mutex.synchronize do
    log_mutex.synchronize do
      puts "All three locks acquired in order"
    end
  end
end

# BAD: log (1) → database (3) — raises exception
# log_mutex.synchronize do
#   db_mutex.synchronize do  # RuntimeError: Lock hierarchy violation!
#     puts "This never executes"
#   end
# end
```

**Deadlock conditions (all four must hold):**
1. **Mutual exclusion** — resource held exclusively
2. **Hold and wait** — thread holds one resource while waiting for another
3. **No preemption** — resources can't be forcibly taken
4. **Circular wait** — circular chain of threads waiting for each other

Breaking any one condition prevents deadlock.

---


## 9.3 Atomic Operations & Thread-Safe Patterns

> An **atomic operation** is an operation that completes in a single step relative to other threads — no other thread can observe the operation in a partially-completed state. Ruby's standard library doesn't have a built-in `Atomic` class like C++ or Java, but the `concurrent-ruby` gem provides robust atomic primitives. This section covers both native approaches and the `concurrent-ruby` gem.

---

### Thread-Safety of Core Operations

Understanding which Ruby operations are "thread-safe" is critical. The GVL ensures that individual VM instructions are atomic, but **compound operations are NOT atomic**.

```ruby
# ATOMIC (single VM instruction — safe without mutex):
x = 42           # simple assignment
y = x             # simple read
arr = [1, 2, 3]   # object creation

# NOT ATOMIC (multiple VM instructions — NOT safe):
x += 1            # read + add + write (3 operations)
arr << item       # read array + append + update size
hash[key] = val   # hash lookup + insert/update
arr.pop           # read array + remove last + update size

# Even "simple" operations can be unsafe:
# x += 1 is actually:
# 1. temp = x       (read)
# 2. temp = temp + 1 (compute)
# 3. x = temp       (write)
# Another thread can read x between steps 1 and 3!
```

**Thread-safe patterns without external gems:**

```ruby
# Pattern 1: Mutex for all shared mutable state
class ThreadSafeCounter
  def initialize
    @mutex = Mutex.new
    @count = 0
  end

  def increment
    @mutex.synchronize { @count += 1 }
  end

  def decrement
    @mutex.synchronize { @count -= 1 }
  end

  def value
    @mutex.synchronize { @count }
  end
end

# Pattern 2: Immutable objects (frozen) — inherently thread-safe
CONFIG = {
  host: "localhost",
  port: 3000,
  debug: false
}.freeze

# CONFIG[:host] = "other"  # => FrozenError — can't modify

# Pattern 3: Thread-local storage — no sharing, no races
Thread.current[:counter] = 0
Thread.current[:counter] += 1  # safe — only this thread accesses it
```

---

### Concurrent-Ruby Gem: Atomic Primitives

The `concurrent-ruby` gem is the de facto standard for advanced concurrency in Ruby. It provides atomic variables, thread-safe collections, and high-level concurrency abstractions.

```ruby
# gem install concurrent-ruby
require 'concurrent'

# AtomicFixnum — thread-safe integer with atomic operations
counter = Concurrent::AtomicFixnum.new(0)

threads = 10.times.map do
  Thread.new do
    10_000.times { counter.increment }
  end
end
threads.each(&:join)
puts counter.value  # Always 100_000 — no mutex needed!
```

**Atomic types in concurrent-ruby:**

```ruby
require 'concurrent'

# AtomicFixnum — atomic integer
num = Concurrent::AtomicFixnum.new(0)
num.increment          # atomically: num += 1, returns new value
num.decrement          # atomically: num -= 1, returns new value
num.increment(5)       # atomically: num += 5
num.value              # read current value
num.value = 42         # atomic write
num.compare_and_set(42, 100)  # CAS: if value == 42, set to 100, return true

# AtomicBoolean — atomic boolean
flag = Concurrent::AtomicBoolean.new(false)
flag.make_true         # atomically set to true
flag.make_false        # atomically set to false
flag.true?             # read
flag.false?            # read
flag.value = true      # atomic write

# AtomicReference — atomic reference to any object (like AtomicReference in Java)
ref = Concurrent::AtomicReference.new("initial")
ref.set("new_value")   # atomic write
ref.get                # atomic read
old = ref.get_and_set("updated")  # atomic swap, returns old value
ref.compare_and_set("updated", "final")  # CAS

# Atom — like Clojure's atom, applies a function atomically
atom = Concurrent::Atom.new(0)
atom.swap { |current| current + 1 }  # atomically apply function
atom.swap { |current| current * 2 }
puts atom.value  # => 2

# Atom with validation
validated_atom = Concurrent::Atom.new(0, validator: ->(val) { val >= 0 })
validated_atom.swap { |v| v + 10 }  # OK
validated_atom.swap { |v| v - 20 }  # fails validation — value stays at 10
```

**Compare-and-Swap (CAS) pattern:**

```ruby
require 'concurrent'

ref = Concurrent::AtomicReference.new(0)

# CAS loop — the foundation of lock-free algorithms
def lock_free_increment(ref)
  loop do
    old_val = ref.get
    new_val = old_val + 1
    return new_val if ref.compare_and_set(old_val, new_val)
    # If CAS fails, another thread modified the value — retry
  end
end

threads = 10.times.map do
  Thread.new do
    1000.times { lock_free_increment(ref) }
  end
end
threads.each(&:join)
puts ref.get  # => 10_000
```

---

### Thread-Safe Collections

```ruby
require 'concurrent'

# Concurrent::Map — thread-safe hash (like Java's ConcurrentHashMap)
map = Concurrent::Map.new

threads = 10.times.map do |i|
  Thread.new do
    100.times do |j|
      map["key_#{i}_#{j}"] = i * j
    end
  end
end
threads.each(&:join)
puts map.size  # => 1000

# Key methods:
map.put(:key, "value")           # insert/update
map.get(:key)                    # read (returns nil if not found)
map.delete(:key)                 # remove
map.put_if_absent(:key, "val")   # insert only if key doesn't exist
map.compute_if_absent(:key) { "computed" }  # lazy compute if absent
map.compute_if_present(:key) { |val| val.upcase }  # update if present
map.compute(:key) { |val| val ? val + 1 : 0 }  # compute unconditionally
map.each_pair { |k, v| puts "#{k}: #{v}" }

# Concurrent::Array — thread-safe array
arr = Concurrent::Array.new
threads = 10.times.map do |i|
  Thread.new { arr << i }
end
threads.each(&:join)
puts arr.sort.inspect  # => [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# Concurrent::Hash — thread-safe hash (simpler than Map)
hash = Concurrent::Hash.new
hash[:key] = "value"

# Concurrent::Set — thread-safe set
set = Concurrent::Set.new
set << "item1"
set << "item2"
puts set.include?("item1")  # => true
```

**Comparison of thread-safe collection approaches:**

| Approach | Thread-Safe | Performance | Use Case |
|----------|------------|-------------|----------|
| `Mutex` + regular Hash/Array | ✅ (manual) | Good (coarse-grained) | Simple cases, full control |
| `Concurrent::Map` | ✅ (built-in) | Best (fine-grained locking) | High-concurrency hash maps |
| `Concurrent::Array` | ✅ (built-in) | Good | Shared arrays |
| Frozen objects | ✅ (immutable) | Best (no locking) | Read-only configuration |
| `Queue` / `SizedQueue` | ✅ (built-in) | Good | Producer-consumer patterns |

---


## 9.4 Fiber & Cooperative Concurrency

> A **Fiber** is a lightweight concurrency primitive that provides **cooperative** (non-preemptive) multitasking. Unlike threads, fibers don't run in parallel — they explicitly yield control to each other. Fibers are the foundation of Ruby's async I/O ecosystem and are much cheaper to create than threads (no OS thread overhead).

---

### Fiber Basics

A Fiber is a block of code that can be paused and resumed. It runs on the same thread as the caller and only switches when explicitly told to.

```ruby
# Create a fiber
fiber = Fiber.new do
  puts "Step 1"
  Fiber.yield        # pause here, return control to caller
  puts "Step 2"
  Fiber.yield        # pause again
  puts "Step 3"
end

fiber.resume  # => "Step 1" (runs until first Fiber.yield)
fiber.resume  # => "Step 2" (resumes from first yield, runs until second)
fiber.resume  # => "Step 3" (resumes from second yield, runs to end)
# fiber.resume  # FiberError: dead fiber called — fiber has finished
```

**Passing values with Fiber.yield and resume:**

```ruby
# Values can flow in both directions
fiber = Fiber.new do |first|
  puts "Received: #{first}"          # "Received: hello"
  second = Fiber.yield("yielded 1")  # pause, return "yielded 1" to caller
  puts "Received: #{second}"         # "Received: world"
  "final result"                     # returned by last resume
end

result1 = fiber.resume("hello")   # => "yielded 1"
result2 = fiber.resume("world")   # => "final result"
puts result1  # => "yielded 1"
puts result2  # => "final result"
```

**Fiber as an Enumerator — infinite sequences:**

```ruby
# Fibonacci sequence using a Fiber
fib = Fiber.new do
  a, b = 0, 1
  loop do
    Fiber.yield a
    a, b = b, a + b
  end
end

10.times { puts fib.resume }
# 0, 1, 1, 2, 3, 5, 8, 13, 21, 34

# Ruby's Enumerator is actually built on Fibers!
enum = Enumerator.new do |yielder|
  a, b = 0, 1
  loop do
    yielder.yield a
    a, b = b, a + b
  end
end

enum.take(10)  # => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

---

### Fiber vs Thread

| Feature | Fiber | Thread |
|---------|-------|--------|
| Scheduling | Cooperative (explicit yield) | Preemptive (OS/VM scheduled) |
| Parallelism | None (same thread) | Limited by GVL |
| Context switch cost | Very low (~100ns) | Higher (~1-10μs) |
| Memory overhead | ~4KB initial stack | ~1MB stack (OS thread) |
| Max count | Millions | Thousands (OS limit) |
| Shared state | Same thread — no races | Shared memory — races possible |
| Use case | Generators, async I/O, coroutines | Background work, I/O concurrency |
| Error handling | Exceptions propagate to caller | Exceptions are per-thread |

```ruby
# Fibers are MUCH cheaper than threads
require 'benchmark'

Benchmark.bm do |x|
  x.report("1000 fibers:") do
    fibers = 1000.times.map { Fiber.new { Fiber.yield } }
    fibers.each(&:resume)
  end

  x.report("1000 threads:") do
    threads = 1000.times.map { Thread.new { sleep(0) } }
    threads.each(&:join)
  end
end
# Fibers are typically 10-100x faster to create and switch
```

---

### Fiber Scheduler (Ruby 3.0+)

Ruby 3.0 introduced the **Fiber Scheduler** interface, which allows fibers to be automatically suspended and resumed during blocking I/O operations. This enables **async I/O without callbacks** — code looks synchronous but runs concurrently.

```ruby
# The Fiber Scheduler is an interface — you need an implementation
# Popular implementations: async gem, evt gem

# Conceptual example of how the scheduler works:
# 1. A fiber calls a blocking operation (e.g., IO.read, sleep)
# 2. The scheduler intercepts the call
# 3. The scheduler suspends the fiber and runs another fiber
# 4. When the I/O completes, the scheduler resumes the original fiber

# With the 'async' gem (most popular implementation):
# gem install async
require 'async'

Async do |task|
  # These run concurrently — the scheduler switches fibers during I/O
  task1 = task.async do
    sleep(1)  # scheduler suspends this fiber
    puts "Task 1 done"
    "result 1"
  end

  task2 = task.async do
    sleep(1)  # scheduler suspends this fiber
    puts "Task 2 done"
    "result 2"
  end

  # Both tasks complete in ~1 second (not 2), because they run concurrently
  puts task1.wait  # => "result 1"
  puts task2.wait  # => "result 2"
end
```

**Fiber Scheduler with HTTP requests:**

```ruby
require 'async'
require 'async/http/internet'

Async do |task|
  internet = Async::HTTP::Internet.new

  urls = [
    "https://example.com",
    "https://example.org",
    "https://example.net"
  ]

  # Launch all requests concurrently using fibers
  tasks = urls.map do |url|
    task.async do
      response = internet.get(url)
      body = response.read
      puts "#{url}: #{body.length} bytes"
      body.length
    end
  end

  # Wait for all results
  results = tasks.map(&:wait)
  puts "Total bytes: #{results.sum}"
ensure
  internet&.close
end
# All 3 requests happen concurrently — total time ≈ slowest single request
```

---

### Practical Fiber Patterns

**Pattern 1: Pipeline processing with Fibers**

```ruby
# Producer fiber
producer = Fiber.new do
  (1..10).each do |i|
    Fiber.yield i
  end
  nil  # signal end
end

# Transformer fiber
transformer = Fiber.new do
  while (value = producer.resume)
    Fiber.yield value * 2
  end
  nil
end

# Consumer
while (result = transformer.resume)
  puts "Got: #{result}"
end
# Got: 2, 4, 6, 8, 10, 12, 14, 16, 18, 20
```

**Pattern 2: Cooperative task scheduler**

```ruby
class SimpleScheduler
  def initialize
    @tasks = []
  end

  def spawn(&block)
    @tasks << Fiber.new(&block)
  end

  def run
    until @tasks.empty?
      task = @tasks.shift
      begin
        task.resume
        @tasks << task if task.alive?
      rescue FiberError
        # Fiber is dead, skip it
      end
    end
  end
end

scheduler = SimpleScheduler.new

scheduler.spawn do
  3.times do |i|
    puts "Task A: step #{i}"
    Fiber.yield
  end
end

scheduler.spawn do
  3.times do |i|
    puts "Task B: step #{i}"
    Fiber.yield
  end
end

scheduler.run
# Output interleaves:
# Task A: step 0
# Task B: step 0
# Task A: step 1
# Task B: step 1
# Task A: step 2
# Task B: step 2
```

---


## 9.5 Ractor: True Parallelism (Ruby 3.0+)

> **Ractor** (Ruby Actor) is Ruby 3.0's answer to true parallelism. Each Ractor has its own GVL, meaning multiple Ractors can execute Ruby code simultaneously on different CPU cores. Ractors achieve thread safety by **not sharing mutable state** — they communicate through message passing, similar to Erlang's actor model.

---

### Ractor Basics

```ruby
# Create a Ractor — it runs in parallel with the main Ractor
r = Ractor.new do
  # This code runs in a separate Ractor with its own GVL
  heavy_computation = (1..1_000_000).sum
  heavy_computation  # return value is sent to the caller
end

# Get the result (blocks until the Ractor finishes)
result = r.take
puts result  # => 500000500000
```

**Ractor creation and communication:**

```ruby
# Method 1: Ractor.new with a block — the block runs in the new Ractor
r = Ractor.new { "hello from ractor" }
puts r.take  # => "hello from ractor"

# Method 2: Pass arguments to the Ractor (must be shareable or copied)
r = Ractor.new(42) do |value|
  value * 2
end
puts r.take  # => 84

# Method 3: Send messages to a Ractor
r = Ractor.new do
  msg = Ractor.receive  # blocks until a message is sent
  "Received: #{msg}"
end
r.send("hello")  # send a message to the Ractor
puts r.take       # => "Received: hello"

# Method 4: Bidirectional communication
r = Ractor.new do
  loop do
    msg = Ractor.receive
    break if msg == :quit
    Ractor.yield "Processed: #{msg}"  # send result back
  end
end

r.send("task1")
puts r.take  # => "Processed: task1"

r.send("task2")
puts r.take  # => "Processed: task2"

r.send(:quit)
```

---

### Ractor Communication Model

Ractors communicate through two channels: **incoming port** (receive messages) and **outgoing port** (yield results).

```
Main Ractor                    Worker Ractor
    |                               |
    |--- send("data") ------------>|  (incoming port)
    |                               |  Ractor.receive
    |                               |  ... process ...
    |<--- Ractor.yield("result") --|  (outgoing port)
    |  r.take                       |
```

**Message passing rules:**

| Rule | Description |
|------|-------------|
| **Copy semantics** | By default, messages are deep-copied between Ractors |
| **Move semantics** | Use `Ractor.yield(obj, move: true)` or `r.send(obj, move: true)` to transfer ownership |
| **Shareable objects** | Frozen objects, `Ractor.make_shareable(obj)`, and some built-in types can be shared without copying |

```ruby
# Copy semantics (default) — safe but potentially expensive
r = Ractor.new do
  data = Ractor.receive
  data << 4  # modifies the COPY, not the original
  data
end

original = [1, 2, 3]
r.send(original)
result = r.take
puts original.inspect  # => [1, 2, 3] — unchanged
puts result.inspect    # => [1, 2, 3, 4]

# Move semantics — transfers ownership (original becomes unusable)
r = Ractor.new do
  data = Ractor.receive
  data.sum
end

arr = [1, 2, 3]
r.send(arr, move: true)
# arr is now inaccessible — using it raises Ractor::MovedError
# puts arr.inspect  # => Ractor::MovedError
puts r.take  # => 6

# Shareable objects — frozen objects can be shared without copying
FROZEN_CONFIG = { host: "localhost", port: 3000 }.freeze
Ractor.make_shareable(FROZEN_CONFIG)  # mark as shareable

r = Ractor.new(FROZEN_CONFIG) do |config|
  "Connecting to #{config[:host]}:#{config[:port]}"
end
puts r.take  # => "Connecting to localhost:3000"
```

---

### Parallel Computation with Ractors

```ruby
# True parallel computation — each Ractor runs on a different CPU core
def parallel_sum(data, num_ractors)
  chunk_size = data.size / num_ractors

  ractors = num_ractors.times.map do |i|
    start_idx = i * chunk_size
    end_idx = (i == num_ractors - 1) ? data.size : start_idx + chunk_size
    chunk = data[start_idx...end_idx]

    Ractor.new(chunk) do |my_chunk|
      my_chunk.sum
    end
  end

  # Collect results from all Ractors
  ractors.map(&:take).sum
end

data = (1..10_000_000).to_a
result = parallel_sum(data, 4)
puts "Sum: #{result}"  # => 50000005000000
# This is genuinely parallel — ~4x speedup on 4 cores!
```

**Ractor pool pattern:**

```ruby
# Create a pool of worker Ractors
def ractor_pool(num_workers, tasks)
  # Create a pipe Ractor to distribute tasks
  pipe = Ractor.new do
    loop do
      Ractor.yield(Ractor.receive)
    end
  end

  # Create worker Ractors
  workers = num_workers.times.map do |i|
    Ractor.new(pipe, i) do |pipe_r, id|
      loop do
        task = pipe_r.take
        break if task == :done
        result = task ** 2  # example computation
        Ractor.yield([id, task, result])
      end
    end
  end

  # Send tasks to the pipe
  tasks.each { |task| pipe.send(task) }
  num_workers.times { pipe.send(:done) }

  # Collect results
  results = []
  workers.each do |worker|
    loop do
      begin
        results << worker.take
      rescue Ractor::ClosedError
        break
      end
    end
  end
  results
end

results = ractor_pool(4, (1..20).to_a)
results.each { |id, task, result| puts "Worker #{id}: #{task}² = #{result}" }
```

---

### Ractor Limitations and Gotchas

```ruby
# 1. Cannot access outer variables (no closures!)
x = 42
# Ractor.new { puts x }  # => Ractor::IsolationError

# Must pass as argument:
Ractor.new(x) { |val| puts val }  # OK

# 2. Cannot share mutable objects
arr = [1, 2, 3]
# Ractor.new(arr) { |a| a }  # copies arr (expensive for large objects)

# 3. Many gems are not Ractor-safe (they use global state)
# Ractor.new { require 'some_gem' }  # may fail

# 4. Class variables and global variables are not accessible
# @@class_var and $global_var cannot be read from non-main Ractors

# 5. Constants must be shareable (frozen)
MUTABLE_CONST = [1, 2, 3]  # not frozen
# Ractor.new { MUTABLE_CONST }  # => Ractor::IsolationError

FROZEN_CONST = [1, 2, 3].freeze
Ractor.new { FROZEN_CONST }  # OK
```

**Ractor vs Thread vs Process:**

| Feature | Thread | Ractor | Process (fork) |
|---------|--------|--------|----------------|
| True parallelism | ❌ (GVL) | ✅ | ✅ |
| Shared memory | ✅ | ❌ (message passing) | ❌ (copy-on-write) |
| Communication | Shared variables + mutex | `send` / `receive` / `yield` / `take` | Pipes, sockets, shared memory |
| Memory overhead | Low (shared heap) | Medium (isolated heap) | High (full process copy) |
| Gem compatibility | ✅ Full | ⚠️ Limited | ✅ Full |
| Maturity | Stable | Experimental (Ruby 3.0+) | Stable |
| Use case | I/O concurrency | CPU parallelism | CPU parallelism, isolation |

---


## 9.6 Async Programming

> Asynchronous programming in Ruby allows you to launch tasks that run concurrently and retrieve their results later. The `concurrent-ruby` gem provides `Future`, `Promise`, `Async`, and other high-level abstractions similar to C++'s `std::async` and `std::future`. Ruby 3.0+ also provides native async support through the Fiber Scheduler.

---

### Future (concurrent-ruby)

A **Future** represents a value that will be computed asynchronously. It launches a block in a thread pool and provides a way to retrieve the result later.

```ruby
require 'concurrent'

# Launch an async computation — runs in a thread pool
future = Concurrent::Future.execute do
  sleep(2)  # simulate expensive work
  42
end

# Main thread continues doing other work...
puts "Computing in background..."
sleep(1)
puts "Still working on main thread..."

# Get the result — blocks if not ready yet
puts future.value  # blocks until the future completes => 42

# Check status without blocking
puts future.state       # :fulfilled, :rejected, or :pending
puts future.fulfilled?  # true
puts future.rejected?   # false
```

**Future with error handling:**

```ruby
require 'concurrent'

# Future that raises an exception
future = Concurrent::Future.execute do
  raise "Something went wrong!"
end

sleep(0.1)  # give it time to execute

puts future.state     # => :rejected
puts future.rejected? # => true
puts future.reason    # => #<RuntimeError: Something went wrong!>
puts future.value     # => nil (returns nil for rejected futures)

# Future with timeout
future = Concurrent::Future.execute { sleep(10); 42 }
result = future.value(2)  # wait at most 2 seconds
puts result  # => nil (timed out)
```

**Multiple futures — parallel execution:**

```ruby
require 'concurrent'

# Launch multiple tasks in parallel
futures = 5.times.map do |i|
  Concurrent::Future.execute do
    sleep(rand * 2)
    "Result from task #{i}"
  end
end

# Collect all results (blocks on each if not ready)
results = futures.map(&:value)
results.each { |r| puts r }
```

---

### Promise (concurrent-ruby)

A **Promise** is similar to a Future but supports **chaining** — you can attach callbacks that execute when the promise is fulfilled or rejected.

```ruby
require 'concurrent'

# Create a promise chain
promise = Concurrent::Promise.execute { 10 }
  .then { |value| value * 2 }      # 20
  .then { |value| value + 5 }      # 25
  .then { |value| "Result: #{value}" }  # "Result: 25"

# Wait for the chain to complete
promise.wait
puts promise.value  # => "Result: 25"

# Promise with error handling
promise = Concurrent::Promise.execute { raise "Error!" }
  .then { |v| v * 2 }  # skipped due to error
  .rescue { |e| "Recovered from: #{e.message}" }

promise.wait
puts promise.value  # => "Recovered from: Error!"

# Chaining with rescue
Concurrent::Promise.execute { 42 }
  .then { |v| v / 0 }  # raises ZeroDivisionError
  .rescue { |e| -1 }   # recover with default value
  .then { |v| puts "Final: #{v}" }  # "Final: -1"
  .wait
```

**Promise.zip — wait for multiple promises:**

```ruby
require 'concurrent'

p1 = Concurrent::Promise.execute { sleep(1); "A" }
p2 = Concurrent::Promise.execute { sleep(2); "B" }
p3 = Concurrent::Promise.execute { sleep(1); "C" }

# Wait for all promises and combine results
combined = Concurrent::Promise.zip(p1, p2, p3).then do |a, b, c|
  "#{a}-#{b}-#{c}"
end

combined.wait
puts combined.value  # => "A-B-C" (after ~2 seconds)
```

---

### Async (concurrent-ruby)

`Concurrent::Async` is a mixin that makes any method call asynchronous — it wraps method calls in futures automatically.

```ruby
require 'concurrent'

class AsyncWorker
  include Concurrent::Async

  def heavy_computation(n)
    sleep(1)  # simulate work
    n * n
  end

  def fetch_data(url)
    sleep(0.5)  # simulate I/O
    "Data from #{url}"
  end
end

worker = AsyncWorker.new

# Synchronous call (normal)
result = worker.heavy_computation(5)  # blocks for 1 second
puts result  # => 25

# Asynchronous call — returns an IVar (future-like)
future = worker.async.heavy_computation(5)  # returns immediately
puts "Doing other work..."
puts future.value  # blocks until complete => 25

# Multiple async calls
f1 = worker.async.heavy_computation(10)
f2 = worker.async.fetch_data("https://example.com")

puts f1.value  # => 100
puts f2.value  # => "Data from https://example.com"

# Await — block until complete, return the value
result = worker.await.heavy_computation(7)
puts result  # => 49 (blocks, then returns directly)
```

---

### Comparison: Future vs Promise vs Async

| Feature | `Future` | `Promise` | `Async` mixin |
|---------|----------|-----------|---------------|
| Execution | Immediate (thread pool) | Immediate (thread pool) | On method call |
| Chaining | ❌ | ✅ (`.then`, `.rescue`) | ❌ |
| Error handling | `.reason` | `.rescue` chain | `.reason` on IVar |
| Use case | Simple async tasks | Complex async pipelines | Making objects async |
| Return type | `Future` | `Promise` | `IVar` |

---

### TimerTask and ScheduledTask

```ruby
require 'concurrent'

# TimerTask — runs a block periodically
task = Concurrent::TimerTask.new(execution_interval: 2) do
  puts "Tick: #{Time.now}"
end
task.execute
sleep(7)
task.shutdown
# Output: Tick every 2 seconds for 7 seconds

# ScheduledTask — runs a block after a delay
scheduled = Concurrent::ScheduledTask.execute(3) do
  puts "Executed after 3 seconds"
  42
end

puts "Waiting..."
puts scheduled.value  # blocks until task executes => 42
```

---


## 9.7 Classic Concurrency Problems

> These are well-known problems in concurrent programming that illustrate common challenges: synchronization, deadlock avoidance, resource sharing, and fairness. Understanding these problems and their solutions is essential for interviews and real-world concurrent system design.

---

### Producer-Consumer Problem

The **Producer-Consumer** (or Bounded Buffer) problem involves one or more producers generating data and placing it into a shared buffer, and one or more consumers removing data from the buffer. The challenge is to synchronize access so that:
- Producers don't add to a full buffer
- Consumers don't remove from an empty buffer
- No data races occur

**Solution 1: Using SizedQueue (idiomatic Ruby)**

```ruby
# SizedQueue handles ALL synchronization internally — the simplest solution
queue = SizedQueue.new(5)  # buffer capacity of 5

# Producer thread
producer = Thread.new do
  20.times do |i|
    queue << i  # blocks if queue is full
    puts "[Producer] Produced: #{i} (queue size: #{queue.size})"
    sleep(0.05)
  end
  queue.close  # signal no more items
end

# Consumer threads
consumers = 2.times.map do |id|
  Thread.new do
    while (item = queue.pop)  # blocks if empty, returns nil when closed
      puts "[Consumer #{id}] Consumed: #{item} (queue size: #{queue.size})"
      sleep(0.1 + rand * 0.05)  # simulate processing
    end
    puts "[Consumer #{id}] Exiting"
  end
end

producer.join
consumers.each(&:join)
puts "All done!"
```

**Solution 2: Using Mutex + ConditionVariable (manual implementation)**

```ruby
class ProducerConsumerQueue
  def initialize(max_size)
    @buffer = []
    @max_size = max_size
    @mutex = Mutex.new
    @not_full = ConditionVariable.new
    @not_empty = ConditionVariable.new
    @done = false
  end

  def produce(item)
    @mutex.synchronize do
      while @buffer.size >= @max_size
        @not_full.wait(@mutex)
      end

      @buffer << item
      puts "[Producer] Produced: #{item} (buffer size: #{@buffer.size})"

      @not_empty.signal
    end
  end

  def consume
    @mutex.synchronize do
      while @buffer.empty? && !@done
        @not_empty.wait(@mutex)
      end

      return nil if @buffer.empty? && @done

      item = @buffer.shift
      puts "[Consumer] Consumed: #{item} (buffer size: #{@buffer.size})"

      @not_full.signal
      item
    end
  end

  def finish
    @mutex.synchronize do
      @done = true
      @not_empty.broadcast  # wake all consumers so they can exit
    end
  end
end

queue = ProducerConsumerQueue.new(5)

producer = Thread.new do
  20.times do |i|
    queue.produce(i)
    sleep(0.05)
  end
  queue.finish
end

consumers = 2.times.map do
  Thread.new do
    loop do
      item = queue.consume
      break if item.nil?
      sleep(0.1)
    end
  end
end

producer.join
consumers.each(&:join)
```

---

### Reader-Writer Problem

Multiple threads need to access a shared resource. **Readers** can access concurrently (they don't modify data), but **writers** need exclusive access. The challenge is to maximize concurrency while ensuring correctness.

```ruby
class ReadWriteLock
  def initialize
    @mutex = Mutex.new
    @cv = ConditionVariable.new
    @active_readers = 0
    @writer_active = false
    @waiting_writers = 0
  end

  def read_lock
    @mutex.synchronize do
      # Wait if a writer is active or writers are waiting (writers-preference)
      while @writer_active || @waiting_writers > 0
        @cv.wait(@mutex)
      end
      @active_readers += 1
    end
  end

  def read_unlock
    @mutex.synchronize do
      @active_readers -= 1
      @cv.broadcast if @active_readers == 0  # wake waiting writers
    end
  end

  def write_lock
    @mutex.synchronize do
      @waiting_writers += 1
      while @writer_active || @active_readers > 0
        @cv.wait(@mutex)
      end
      @waiting_writers -= 1
      @writer_active = true
    end
  end

  def write_unlock
    @mutex.synchronize do
      @writer_active = false
      @cv.broadcast  # wake waiting readers and writers
    end
  end

  # Convenience methods with blocks
  def with_read_lock
    read_lock
    yield
  ensure
    read_unlock
  end

  def with_write_lock
    write_lock
    yield
  ensure
    write_unlock
  end
end

# Usage
class SharedDatabase
  def initialize
    @lock = ReadWriteLock.new
    @data = 0
  end

  def read
    @lock.with_read_lock do
      puts "[Reader #{Thread.current.object_id}] Read: #{@data}"
      sleep(0.1)  # simulate read time
      @data
    end
  end

  def write(value)
    @lock.with_write_lock do
      puts "[Writer #{Thread.current.object_id}] Writing: #{value}"
      @data = value
      sleep(0.2)  # simulate write time
    end
  end
end

db = SharedDatabase.new

# Multiple reader threads
readers = 5.times.map do
  Thread.new do
    3.times do
      db.read
      sleep(0.05)
    end
  end
end

# Writer threads
writers = 2.times.map do |i|
  Thread.new do
    3.times do |j|
      db.write(i * 100 + j)
      sleep(0.3)
    end
  end
end

readers.each(&:join)
writers.each(&:join)
```

**Using concurrent-ruby's ReadWriteLock:**

```ruby
require 'concurrent'

class SharedCache
  def initialize
    @lock = Concurrent::ReadWriteLock.new
    @cache = {}
  end

  def get(key)
    @lock.with_read_lock { @cache[key] }
  end

  def set(key, value)
    @lock.with_write_lock { @cache[key] = value }
  end

  def delete(key)
    @lock.with_write_lock { @cache.delete(key) }
  end

  def size
    @lock.with_read_lock { @cache.size }
  end
end
```

---

### Dining Philosophers Problem

Five philosophers sit around a table. Each philosopher alternates between thinking and eating. To eat, a philosopher needs two forks (one on each side). The challenge is to prevent **deadlock** (all philosophers pick up their left fork and wait for the right) and **starvation** (a philosopher never gets to eat).

```ruby
NUM_PHILOSOPHERS = 5
forks = Array.new(NUM_PHILOSOPHERS) { Mutex.new }

# SOLUTION 1: Resource ordering — always pick up the lower-numbered fork first
def philosopher(id, forks)
  left = id
  right = (id + 1) % NUM_PHILOSOPHERS

  # Always lock the lower-numbered fork first to prevent deadlock
  first = [left, right].min
  second = [left, right].max

  3.times do |meal|
    # Think
    puts "Philosopher #{id} is thinking"
    sleep(rand * 0.2)

    # Pick up forks (in consistent order)
    forks[first].synchronize do
      forks[second].synchronize do
        # Eat
        puts "Philosopher #{id} is eating (meal #{meal + 1})"
        sleep(rand * 0.2)
      end
    end

    puts "Philosopher #{id} finished eating"
  end
end

threads = NUM_PHILOSOPHERS.times.map do |i|
  Thread.new { philosopher(i, forks) }
end
threads.each(&:join)
puts "Dinner is over!"
```

**SOLUTION 2: Using try_lock with backoff**

```ruby
NUM_PHILOSOPHERS = 5
forks = Array.new(NUM_PHILOSOPHERS) { Mutex.new }

def philosopher_v2(id, forks)
  left = id
  right = (id + 1) % NUM_PHILOSOPHERS

  3.times do |meal|
    puts "Philosopher #{id} is thinking"
    sleep(rand * 0.2)

    # Try to pick up both forks without deadlocking
    loop do
      forks[left].lock
      if forks[right].try_lock
        # Got both forks — eat!
        puts "Philosopher #{id} is eating (meal #{meal + 1})"
        sleep(rand * 0.2)
        forks[right].unlock
        forks[left].unlock
        break
      else
        # Couldn't get right fork — release left and retry
        forks[left].unlock
        sleep(rand * 0.01)  # random backoff
      end
    end

    puts "Philosopher #{id} finished eating"
  end
end

threads = NUM_PHILOSOPHERS.times.map do |i|
  Thread.new { philosopher_v2(i, forks) }
end
threads.each(&:join)
puts "Dinner is over!"
```

---

### Sleeping Barber Problem

A barber shop has one barber, one barber chair, and N waiting chairs. If there are no customers, the barber sleeps. When a customer arrives, they wake the barber (if sleeping) or sit in a waiting chair (if the barber is busy). If all waiting chairs are full, the customer leaves.

```ruby
class BarberShop
  def initialize(num_chairs)
    @num_chairs = num_chairs
    @mutex = Mutex.new
    @barber_cv = ConditionVariable.new
    @customer_cv = ConditionVariable.new
    @waiting_customers = []
    @barber_sleeping = true
    @haircut_done = false
    @shop_open = true
  end

  def enter_shop(customer_id)
    @mutex.synchronize do
      if @waiting_customers.size >= @num_chairs
        puts "Customer #{customer_id}: No seats available, leaving"
        return
      end

      @waiting_customers << customer_id
      puts "Customer #{customer_id}: Sitting in waiting room (#{@waiting_customers.size}/#{@num_chairs})"

      # Wake the barber if sleeping
      @barber_cv.signal if @barber_sleeping

      # Wait for haircut
      @haircut_done = false
      until @haircut_done
        @customer_cv.wait(@mutex)
      end
      puts "Customer #{customer_id}: Haircut done, leaving happy"
    end
  end

  def barber_work
    loop do
      @mutex.synchronize do
        # Sleep if no customers
        while @waiting_customers.empty? && @shop_open
          @barber_sleeping = true
          puts "Barber: Sleeping..."
          @barber_cv.wait(@mutex)
        end
        @barber_sleeping = false

        break if !@shop_open && @waiting_customers.empty?

        customer = @waiting_customers.shift
        puts "Barber: Cutting hair of customer #{customer}"
      end

      # Cut hair (outside the lock — takes time)
      sleep(0.2)

      @mutex.synchronize do
        @haircut_done = true
        @customer_cv.broadcast
      end
    end
    puts "Barber: Going home"
  end

  def close_shop
    @mutex.synchronize do
      @shop_open = false
      @barber_cv.signal
    end
  end
end

shop = BarberShop.new(3)  # 3 waiting chairs

barber = Thread.new { shop.barber_work }

customers = 10.times.map do |i|
  Thread.new do
    sleep(i * 0.08)
    shop.enter_shop(i + 1)
  end
end

customers.each(&:join)
shop.close_shop
barber.join
```

---

### Summary of Classic Problems

| Problem | Key Challenge | Ruby Solution |
|---------|--------------|---------------|
| Producer-Consumer | Synchronize access to shared buffer | `SizedQueue` (built-in) or Mutex + ConditionVariable |
| Reader-Writer | Allow concurrent reads, exclusive writes | `ReadWriteLock` (concurrent-ruby) or manual implementation |
| Dining Philosophers | Prevent deadlock with circular resource dependency | Lock ordering or try_lock with backoff |
| Sleeping Barber | Coordinate barber sleep/wake with customer arrivals | Mutex + ConditionVariable |

---


## 9.8 Thread Pool

> A **thread pool** is a collection of pre-created worker threads that wait for tasks to execute. Instead of creating and destroying threads for each task (which is expensive), tasks are submitted to the pool and executed by available threads. Thread pools are fundamental to high-performance servers, task schedulers, and parallel computation frameworks.

---

### Why Thread Pools?

**Problem with creating threads per task:**

```ruby
# BAD: Creating a new thread for every request
def handle_request(request)
  Thread.new do
    process(request)
  end
  # Problems:
  # 1. Thread creation overhead (~1ms per thread)
  # 2. No limit on concurrent threads — can exhaust system resources
  # 3. Context switching overhead with thousands of threads
  # 4. No way to control or cancel tasks
  # 5. If main exits, all threads are killed
end
```

**Thread pool benefits:**
- **Reuse threads** — avoid creation/destruction overhead
- **Limit concurrency** — prevent resource exhaustion
- **Task queuing** — buffer work when all threads are busy
- **Graceful shutdown** — wait for in-progress tasks to complete
- **Better resource utilization** — match thread count to hardware

---

### Custom Thread Pool Implementation

```ruby
class ThreadPool
  def initialize(size)
    @size = size
    @queue = Queue.new
    @workers = []
    @shutdown = false

    # Create worker threads
    @size.times do
      @workers << Thread.new do
        loop do
          task = @queue.pop
          break if task == :shutdown
          begin
            task.call
          rescue => e
            puts "[ThreadPool] Task error: #{e.message}"
          end
        end
      end
    end
  end

  # Submit a task (callable/block) to the pool
  def submit(&block)
    raise "ThreadPool is shut down" if @shutdown
    @queue << block
  end

  # Submit a task and get a future-like object for the result
  def submit_with_result(&block)
    raise "ThreadPool is shut down" if @shutdown
    result_queue = Queue.new

    @queue << -> {
      begin
        result_queue << [:ok, block.call]
      rescue => e
        result_queue << [:error, e]
      end
    }

    # Return a simple future
    -> {
      status, value = result_queue.pop
      raise value if status == :error
      value
    }
  end

  # Graceful shutdown — finish all queued tasks, then stop
  def shutdown
    @shutdown = true
    @size.times { @queue << :shutdown }  # send shutdown signal to each worker
    @workers.each(&:join)
  end

  # Immediate shutdown — discard pending tasks
  def shutdown!
    @shutdown = true
    @queue.clear
    @size.times { @queue << :shutdown }
    @workers.each { |w| w.join(5) || w.kill }
  end

  def queue_size
    @queue.size
  end
end

# Usage
pool = ThreadPool.new(4)  # 4 worker threads

# Submit tasks
20.times do |i|
  pool.submit do
    puts "Task #{i} running on #{Thread.current.object_id}"
    sleep(0.1)
  end
end

# Submit with result
futures = 10.times.map do |i|
  pool.submit_with_result { i * i }
end

# Collect results
results = futures.map(&:call)
puts "Results: #{results.inspect}"  # => [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# Graceful shutdown
pool.shutdown
```

---

### concurrent-ruby Thread Pools

The `concurrent-ruby` gem provides production-ready thread pool implementations.

```ruby
require 'concurrent'

# FixedThreadPool — fixed number of worker threads
pool = Concurrent::FixedThreadPool.new(4)

20.times do |i|
  pool.post do
    puts "Task #{i} on thread #{Thread.current.object_id}"
    sleep(0.1)
  end
end

pool.shutdown
pool.wait_for_termination

# CachedThreadPool — creates threads as needed, reuses idle threads
pool = Concurrent::CachedThreadPool.new

10.times do |i|
  pool.post { puts "Task #{i}" }
end

pool.shutdown
pool.wait_for_termination

# ThreadPoolExecutor — full control over pool configuration
pool = Concurrent::ThreadPoolExecutor.new(
  min_threads: 2,           # minimum number of threads
  max_threads: 10,          # maximum number of threads
  max_queue: 100,           # maximum number of queued tasks
  fallback_policy: :caller_runs  # what to do when queue is full
)

100.times do |i|
  pool.post { sleep(0.1); puts "Task #{i}" }
end

pool.shutdown
pool.wait_for_termination
```

**Fallback policies (when the queue is full):**

| Policy | Behavior |
|--------|----------|
| `:abort` | Raise `Concurrent::RejectedExecutionError` |
| `:discard` | Silently discard the task |
| `:caller_runs` | Execute the task in the calling thread (backpressure) |

**Thread pool comparison:**

| Pool Type | Threads | Queue | Use Case |
|-----------|---------|-------|----------|
| `FixedThreadPool` | Fixed count | Unbounded | Known workload, predictable resource usage |
| `CachedThreadPool` | Dynamic (0 to ∞) | No queue (immediate) | Bursty workloads, many short tasks |
| `ThreadPoolExecutor` | Configurable min/max | Bounded | Production systems, full control |

---

### Priority Task Queue

```ruby
require 'concurrent'

class PriorityThreadPool
  def initialize(num_workers)
    @mutex = Mutex.new
    @cv = ConditionVariable.new
    @tasks = []  # sorted by priority
    @shutdown = false

    @workers = num_workers.times.map do
      Thread.new { worker_loop }
    end
  end

  def submit(priority, &block)
    @mutex.synchronize do
      raise "Pool is shut down" if @shutdown
      @tasks << { priority: priority, task: block }
      @tasks.sort_by! { |t| -t[:priority] }  # highest priority first
      @cv.signal
    end
  end

  def shutdown
    @mutex.synchronize do
      @shutdown = true
      @cv.broadcast
    end
    @workers.each(&:join)
  end

  private

  def worker_loop
    loop do
      task = nil
      @mutex.synchronize do
        while @tasks.empty? && !@shutdown
          @cv.wait(@mutex)
        end
        return if @shutdown && @tasks.empty?
        task = @tasks.shift[:task]
      end
      task&.call
    end
  end
end

# Usage
pool = PriorityThreadPool.new(2)
pool.submit(1) { puts "Low priority task" }
pool.submit(10) { puts "HIGH priority task" }
pool.submit(5) { puts "Medium priority task" }
sleep(0.1)
pool.shutdown
# HIGH priority runs first, then medium, then low
```

---

### Work Stealing (Concept)

In a basic thread pool, all workers share a single task queue — this creates contention when many workers try to dequeue simultaneously. **Work stealing** gives each worker its own local queue, and idle workers "steal" tasks from busy workers' queues.

```
Basic Thread Pool:
  [Shared Queue] ← all workers contend for this
       ↓
  [Worker 1] [Worker 2] [Worker 3] [Worker 4]

Work-Stealing Thread Pool:
  [Worker 1: local queue] → tasks
  [Worker 2: local queue] → tasks
  [Worker 3: local queue] → (empty — steals from Worker 1)
  [Worker 4: local queue] → tasks
```

**Conceptual implementation in Ruby:**

```ruby
class WorkStealingPool
  def initialize(num_workers)
    @queues = num_workers.times.map { Queue.new }
    @workers = num_workers.times.map do |i|
      Thread.new { worker_loop(i) }
    end
    @next_queue = Concurrent::AtomicFixnum.new(0)
    @shutdown = false
  end

  def submit(&block)
    # Round-robin assignment to worker queues
    idx = @next_queue.increment % @queues.size
    @queues[idx] << block
  end

  def shutdown
    @shutdown = true
    @queues.each { |q| q << :shutdown }
    @workers.each(&:join)
  end

  private

  def worker_loop(my_id)
    loop do
      # Try own queue first (non-blocking)
      task = nil
      begin
        task = @queues[my_id].pop(true)  # non-blocking
      rescue ThreadError
        # Own queue empty — try stealing from others
        task = steal_from_others(my_id)
      end

      return if task == :shutdown
      next sleep(0.001) if task.nil?  # nothing to steal, brief sleep

      begin
        task.call
      rescue => e
        puts "[Worker #{my_id}] Error: #{e.message}"
      end
    end
  end

  def steal_from_others(my_id)
    @queues.each_with_index do |queue, i|
      next if i == my_id
      begin
        return queue.pop(true)  # non-blocking steal
      rescue ThreadError
        next  # queue empty, try next
      end
    end
    nil  # nothing to steal
  end
end
```

**Benefits of work stealing:**
- Reduces contention on the shared queue
- Automatic load balancing (idle workers steal from busy ones)
- Better cache locality (workers process their own tasks)
- Used in: Java ForkJoinPool, Go runtime, Tokio (Rust)

---

### Thread Pool Design Considerations

| Consideration | Options |
|--------------|---------|
| **Pool size** | CPU count for CPU-bound; larger (CPU × 2-5) for I/O-bound |
| **Queue type** | Unbounded (risk OOM), bounded (backpressure), priority |
| **Rejection policy** | Block caller, raise exception, discard, run in caller's thread |
| **Shutdown** | Graceful (finish queued tasks) vs immediate (discard pending) |
| **Task cancellation** | Via Future, cancellation tokens, or flags |
| **Exception handling** | Propagate via Future, log and continue, or re-raise |

**Sizing guidelines:**

```ruby
# CPU-bound tasks: threads ≈ number of CPU cores
# (More threads won't help due to GVL — use Ractors instead)
cpu_pool_size = Etc.nprocessors

# I/O-bound tasks: threads ≈ CPU cores × ratio of wait time to compute time
# For mostly-waiting tasks (HTTP, DB): 2-5× CPU cores
io_pool_size = Etc.nprocessors * 3

# Mixed workload: separate pools for CPU and I/O tasks
cpu_pool = Concurrent::FixedThreadPool.new(Etc.nprocessors)
io_pool = Concurrent::FixedThreadPool.new(Etc.nprocessors * 3)
```

---


## 9.9 Concurrent Data Structures

> Standard Ruby collections (`Array`, `Hash`, `Set`) are NOT thread-safe. Concurrent access requires either external synchronization (Mutex) or purpose-built thread-safe data structures. This section covers common concurrent data structures and their implementation patterns in Ruby.

---

### Thread-Safe Queue (Built-in)

Ruby's `Queue` and `SizedQueue` are the most commonly needed concurrent data structures — used in thread pools, producer-consumer patterns, and message passing. They are built into the standard library and handle all synchronization internally.

```ruby
# Thread-safe unbounded queue
queue = Queue.new

# Thread-safe bounded queue (with backpressure)
sized_queue = SizedQueue.new(10)

# Both support:
queue.push(item)       # add item (SizedQueue blocks if full)
queue.pop              # remove item (blocks if empty)
queue.pop(true)        # non-blocking pop (raises ThreadError if empty)
queue.size             # current size
queue.empty?           # check if empty
queue.close            # prevent further pushes
queue.clear            # remove all items
```

See section 9.2 for detailed Queue/SizedQueue usage.

---

### Thread-Safe Hash Map

**Simple approach — Mutex wrapper:**

```ruby
class ThreadSafeHash
  def initialize
    @hash = {}
    @mutex = Mutex.new
  end

  def [](key)
    @mutex.synchronize { @hash[key] }
  end

  def []=(key, value)
    @mutex.synchronize { @hash[key] = value }
  end

  def delete(key)
    @mutex.synchronize { @hash.delete(key) }
  end

  def key?(key)
    @mutex.synchronize { @hash.key?(key) }
  end

  def size
    @mutex.synchronize { @hash.size }
  end

  def each(&block)
    @mutex.synchronize { @hash.each(&block) }
  end

  # Atomic compute-if-absent
  def compute_if_absent(key)
    @mutex.synchronize do
      unless @hash.key?(key)
        @hash[key] = yield
      end
      @hash[key]
    end
  end
end
```

**Better approach — ReadWriteLock for read-heavy workloads:**

```ruby
require 'concurrent'

class ReadOptimizedHash
  def initialize
    @hash = {}
    @lock = Concurrent::ReadWriteLock.new
  end

  def [](key)
    @lock.with_read_lock { @hash[key] }
  end

  def []=(key, value)
    @lock.with_write_lock { @hash[key] = value }
  end

  def delete(key)
    @lock.with_write_lock { @hash.delete(key) }
  end

  def size
    @lock.with_read_lock { @hash.size }
  end

  def each(&block)
    # Take a snapshot under read lock, iterate outside the lock
    snapshot = @lock.with_read_lock { @hash.dup }
    snapshot.each(&block)
  end
end

# Best approach — use Concurrent::Map (striped locking internally)
map = Concurrent::Map.new
map[:key] = "value"
puts map[:key]  # => "value"
```

**Striped locking — one mutex per bucket (high concurrency):**

```ruby
class StripedHashMap
  NUM_STRIPES = 16

  def initialize
    @stripes = NUM_STRIPES.times.map { { mutex: Mutex.new, data: {} } }
  end

  def [](key)
    stripe = stripe_for(key)
    stripe[:mutex].synchronize { stripe[:data][key] }
  end

  def []=(key, value)
    stripe = stripe_for(key)
    stripe[:mutex].synchronize { stripe[:data][key] = value }
  end

  def delete(key)
    stripe = stripe_for(key)
    stripe[:mutex].synchronize { stripe[:data].delete(key) }
  end

  def size
    @stripes.sum do |stripe|
      stripe[:mutex].synchronize { stripe[:data].size }
    end
  end

  private

  def stripe_for(key)
    @stripes[key.hash.abs % NUM_STRIPES]
  end
end

# With 16 stripes, up to 16 threads can operate on different stripes simultaneously!
map = StripedHashMap.new

threads = 10.times.map do |i|
  Thread.new do
    100.times do |j|
      map["key_#{i}_#{j}"] = i * j
    end
  end
end
threads.each(&:join)
puts map.size  # => 1000
```

---

### Thread-Safe Counter

```ruby
# Simple Mutex-based counter
class ThreadSafeCounter
  def initialize(initial = 0)
    @value = initial
    @mutex = Mutex.new
  end

  def increment(by = 1)
    @mutex.synchronize { @value += by }
  end

  def decrement(by = 1)
    @mutex.synchronize { @value -= by }
  end

  def value
    @mutex.synchronize { @value }
  end

  def reset
    @mutex.synchronize do
      old = @value
      @value = 0
      old
    end
  end
end

# Better: Use Concurrent::AtomicFixnum (lock-free)
require 'concurrent'

counter = Concurrent::AtomicFixnum.new(0)
threads = 10.times.map do
  Thread.new { 10_000.times { counter.increment } }
end
threads.each(&:join)
puts counter.value  # => 100_000 (always correct, no mutex needed)
```

---

### Thread-Safe Singleton

Ensuring a singleton is initialized exactly once across threads:

```ruby
# Method 1: Ruby's built-in Singleton module (thread-safe)
require 'singleton'

class Configuration
  include Singleton

  attr_accessor :host, :port

  def initialize
    @host = "localhost"
    @port = 3000
    puts "Configuration initialized"
  end
end

# Thread-safe — only one instance is ever created
threads = 10.times.map do
  Thread.new { Configuration.instance }
end
threads.each(&:join)
# "Configuration initialized" prints exactly once

# Method 2: Mutex + double-checked locking
class DatabasePool
  @instance = nil
  @mutex = Mutex.new

  def self.instance
    return @instance if @instance  # fast path — no lock needed

    @mutex.synchronize do
      @instance ||= new  # double-check inside the lock
    end
  end

  private_class_method :new

  def initialize
    puts "DatabasePool created"
    @connections = []
  end
end

# Method 3: Concurrent::Delay (lazy, thread-safe initialization)
require 'concurrent'

DB_POOL = Concurrent::Delay.new { DatabasePool.new }

# First call initializes, subsequent calls return cached value
pool = DB_POOL.value  # thread-safe lazy initialization
```

---

### Choosing the Right Concurrent Data Structure

| Data Structure | Approach | Throughput | Complexity | Use Case |
|---------------|----------|-----------|-----------|----------|
| `Mutex` + Hash/Array | Simple wrapper | Low under contention | Easy | Prototyping, low contention |
| `ReadWriteLock` + Hash | concurrent-ruby | Good for read-heavy | Easy | Caches, config stores |
| `Concurrent::Map` | Striped locking | High | Easy (built-in) | General-purpose concurrent hash |
| `StripedHashMap` | Lock per partition | High | Medium | Custom hash maps |
| `Queue` / `SizedQueue` | Built-in thread-safe | Good | Easy | Producer-consumer patterns |
| `Concurrent::AtomicFixnum` | Lock-free | Very high | Easy | Counters, flags |
| Frozen objects | Immutable | Best (no locking) | Easy | Read-only configuration |

**Guidelines:**
1. Start with a Mutex-based solution — it's correct and easy to reason about
2. Use `Queue`/`SizedQueue` for producer-consumer patterns (built-in, correct)
3. Use `Concurrent::Map` for concurrent hash maps (best general choice)
4. Use `Concurrent::AtomicFixnum`/`AtomicBoolean` for simple counters and flags
5. Use `ReadWriteLock` if reads vastly outnumber writes
6. Use frozen/immutable objects wherever possible — no synchronization needed

---


## 9.10 Practical Patterns & Best Practices

> This section covers real-world concurrency patterns commonly used in Ruby applications — from web servers to background job processors.

---

### Pattern 1: Worker Pool with Job Queue

The most common pattern in Ruby applications — a pool of workers processing jobs from a queue (used by Sidekiq, Resque, etc.).

```ruby
class WorkerPool
  def initialize(num_workers:, queue_size: 100)
    @queue = SizedQueue.new(queue_size)
    @workers = num_workers.times.map do |i|
      Thread.new { worker_loop(i) }
    end
  end

  def enqueue(job)
    @queue << job
  end

  def shutdown
    @queue.close
    @workers.each(&:join)
  end

  private

  def worker_loop(id)
    while (job = @queue.pop)
      begin
        puts "[Worker #{id}] Processing: #{job}"
        job.call
      rescue => e
        puts "[Worker #{id}] Error: #{e.message}"
      end
    end
    puts "[Worker #{id}] Shutting down"
  end
end

# Usage
pool = WorkerPool.new(num_workers: 4)

20.times do |i|
  pool.enqueue(-> {
    sleep(rand * 0.2)
    puts "Job #{i} complete"
  })
end

sleep(2)
pool.shutdown
```

---

### Pattern 2: Parallel Map (concurrent collection processing)

```ruby
require 'concurrent'

module Enumerable
  def parallel_map(pool_size: Etc.nprocessors)
    pool = Concurrent::FixedThreadPool.new(pool_size)

    futures = map do |item|
      Concurrent::Future.execute(executor: pool) { yield item }
    end

    results = futures.map(&:value!)  # value! raises on error
    pool.shutdown
    results
  end
end

# Usage
urls = ["https://example.com"] * 10
results = urls.parallel_map { |url| Net::HTTP.get(URI(url)).length }
puts results.inspect
```

---

### Pattern 3: Circuit Breaker (fault tolerance)

```ruby
require 'concurrent'

class CircuitBreaker
  STATES = %i[closed open half_open].freeze

  def initialize(threshold: 5, timeout: 30)
    @threshold = threshold
    @timeout = timeout
    @failure_count = Concurrent::AtomicFixnum.new(0)
    @state = Concurrent::AtomicReference.new(:closed)
    @last_failure_time = Concurrent::AtomicReference.new(nil)
    @mutex = Mutex.new
  end

  def call(&block)
    case @state.get
    when :open
      if Time.now - @last_failure_time.get > @timeout
        @state.set(:half_open)
        try_call(&block)
      else
        raise "Circuit is OPEN — request rejected"
      end
    when :closed, :half_open
      try_call(&block)
    end
  end

  def state
    @state.get
  end

  private

  def try_call(&block)
    result = block.call
    on_success
    result
  rescue => e
    on_failure
    raise
  end

  def on_success
    @failure_count.value = 0
    @state.set(:closed)
  end

  def on_failure
    @last_failure_time.set(Time.now)
    count = @failure_count.increment
    @state.set(:open) if count >= @threshold
  end
end

# Usage
breaker = CircuitBreaker.new(threshold: 3, timeout: 10)

5.times do |i|
  begin
    breaker.call do
      raise "Service unavailable" if rand < 0.7
      "Success!"
    end
  rescue => e
    puts "Attempt #{i + 1}: #{e.message} (state: #{breaker.state})"
  end
end
```

---

### Pattern 4: Rate Limiter (Token Bucket)

```ruby
class TokenBucketRateLimiter
  def initialize(rate:, burst:)
    @rate = rate.to_f       # tokens per second
    @burst = burst.to_f     # max tokens (bucket size)
    @tokens = burst.to_f    # start full
    @last_refill = Time.now
    @mutex = Mutex.new
  end

  def acquire(tokens = 1)
    @mutex.synchronize do
      refill
      if @tokens >= tokens
        @tokens -= tokens
        true
      else
        false
      end
    end
  end

  # Blocking acquire — waits until tokens are available
  def acquire!(tokens = 1)
    loop do
      return if acquire(tokens)
      sleep(tokens / @rate)  # wait for enough tokens to regenerate
    end
  end

  def available_tokens
    @mutex.synchronize do
      refill
      @tokens
    end
  end

  private

  def refill
    now = Time.now
    elapsed = now - @last_refill
    @tokens = [@tokens + elapsed * @rate, @burst].min
    @last_refill = now
  end
end

# Usage: 10 requests per second, burst of 20
limiter = TokenBucketRateLimiter.new(rate: 10, burst: 20)

threads = 50.times.map do |i|
  Thread.new do
    if limiter.acquire
      puts "Request #{i}: allowed"
    else
      puts "Request #{i}: rate limited"
    end
  end
end
threads.each(&:join)
```

---

### Pattern 5: Publish-Subscribe with Threads

```ruby
class EventBus
  def initialize
    @subscribers = {}
    @mutex = Mutex.new
  end

  def subscribe(event, &handler)
    @mutex.synchronize do
      @subscribers[event] ||= []
      @subscribers[event] << handler
    end
  end

  def publish(event, *args)
    handlers = @mutex.synchronize { (@subscribers[event] || []).dup }

    # Notify subscribers in separate threads
    threads = handlers.map do |handler|
      Thread.new { handler.call(*args) }
    end
    threads.each(&:join)
  end

  # Async publish — fire and forget
  def publish_async(event, *args)
    handlers = @mutex.synchronize { (@subscribers[event] || []).dup }

    handlers.each do |handler|
      Thread.new do
        handler.call(*args)
      rescue => e
        puts "[EventBus] Handler error: #{e.message}"
      end
    end
  end
end

bus = EventBus.new

bus.subscribe(:user_created) { |user| puts "Send welcome email to #{user}" }
bus.subscribe(:user_created) { |user| puts "Create audit log for #{user}" }
bus.subscribe(:user_created) { |user| puts "Notify admin about #{user}" }

bus.publish(:user_created, "Alice")
```

---

### Best Practices Summary

| Practice | Why |
|----------|-----|
| Use `Queue`/`SizedQueue` for producer-consumer | Built-in, correct, handles all synchronization |
| Prefer `Mutex#synchronize` over manual `lock`/`unlock` | Exception-safe, can't forget to unlock |
| Use `concurrent-ruby` for complex patterns | Battle-tested, well-documented, performant |
| Use Ractors for CPU-bound parallelism | Only way to bypass the GVL in pure Ruby |
| Use threads for I/O-bound concurrency | GVL is released during I/O — threads help |
| Minimize shared mutable state | Less sharing = fewer bugs |
| Use immutable (frozen) objects when possible | No synchronization needed |
| Set `Thread.abort_on_exception = true` in development | Don't let thread errors go unnoticed |
| Size thread pools based on workload type | CPU-bound: core count; I/O-bound: core count × 2-5 |
| Always use a predicate loop with ConditionVariable | Protects against spurious wakeups |

---


## Summary & Key Takeaways

| Topic | Core Concept | Key Tools |
|-------|-------------|-----------|
| Thread Basics | Create, join, get value from threads; understand GVL | `Thread.new`, `join`, `value` |
| GVL/GIL | Only one thread executes Ruby at a time; I/O releases GVL | Threads for I/O, Ractors for CPU |
| Synchronization | Protect shared data from data races | `Mutex`, `Monitor`, `ConditionVariable` |
| Thread-Safe Queues | Built-in synchronized producer-consumer | `Queue`, `SizedQueue` |
| Atomic Operations | Lock-free thread-safe operations on single variables | `Concurrent::AtomicFixnum`, `AtomicReference`, `Atom` |
| Fibers | Cooperative concurrency, lightweight coroutines | `Fiber.new`, `Fiber.yield`, `resume` |
| Fiber Scheduler | Async I/O without callbacks (Ruby 3.0+) | `async` gem, `Fiber::Scheduler` |
| Ractors | True parallelism bypassing the GVL (Ruby 3.0+) | `Ractor.new`, `send`, `receive`, `take` |
| Async Programming | Launch tasks and retrieve results later | `Concurrent::Future`, `Promise`, `Async` |
| Classic Problems | Producer-Consumer, Reader-Writer, Dining Philosophers | `SizedQueue`, `ReadWriteLock`, lock ordering |
| Thread Pool | Reuse threads for task execution | `Concurrent::FixedThreadPool`, `ThreadPoolExecutor` |
| Concurrent Data Structures | Thread-safe containers | `Concurrent::Map`, `Queue`, `AtomicFixnum` |

---

## Interview Tips for Module 9

1. **"What is the GVL/GIL in Ruby?"** The Global VM Lock is a mutex that prevents multiple threads from executing Ruby code simultaneously. Only one thread runs Ruby at a time. However, the GVL is released during I/O operations (file, network, sleep), so threads DO help for I/O-bound work. For CPU-bound parallelism, use Ractors (Ruby 3.0+) or fork processes.

2. **"Are Ruby threads real OS threads?"** Yes, since Ruby 1.9 (MRI/CRuby). They are native POSIX threads mapped 1:1 to OS threads. However, the GVL limits their parallel execution of Ruby code. JRuby and TruffleRuby don't have a GVL and support true thread parallelism.

3. **"How do you prevent data races in Ruby?"** Use `Mutex#synchronize` to protect shared mutable state. Use `Queue`/`SizedQueue` for producer-consumer patterns. Use `Concurrent::Map` or `Concurrent::AtomicFixnum` for lock-free thread-safe operations. Best of all — minimize shared mutable state and use immutable (frozen) objects.

4. **"What's the difference between a Mutex and a Monitor?"** A Mutex deadlocks if the same thread tries to lock it twice (non-reentrant). A Monitor allows the same thread to lock it multiple times (reentrant) — useful when synchronized methods call other synchronized methods. Mutex is faster for simple cases.

5. **"Explain the Producer-Consumer problem in Ruby."** Use `SizedQueue` — it handles everything: blocks producers when full, blocks consumers when empty, and is thread-safe. For manual implementation, use a Mutex + two ConditionVariables (not_full, not_empty). Producers wait on not_full, signal not_empty. Consumers wait on not_empty, signal not_full.

6. **"What is a spurious wakeup?"** A thread waiting on a ConditionVariable wakes up without being signaled. Always use a `while` loop to re-check the condition: `while queue.empty?; cv.wait(mutex); end`. Never use a bare `cv.wait` without a condition check.

7. **"What are Fibers and how do they differ from Threads?"** Fibers are lightweight cooperative concurrency primitives. They run on the same thread and only switch when explicitly yielded (`Fiber.yield`). They're much cheaper than threads (~4KB vs ~1MB stack), can't run in parallel, and don't need synchronization. Used for generators, coroutines, and async I/O (via Fiber Scheduler in Ruby 3.0+).

8. **"What are Ractors?"** Ractors (Ruby 3.0+) provide true parallelism by giving each Ractor its own GVL. They communicate through message passing (send/receive) and cannot share mutable state. Think of them as Ruby's actor model. They're still experimental and many gems aren't Ractor-safe.

9. **"How would you implement a thread pool in Ruby?"** Create N worker threads that loop on a `Queue`. Submit tasks by pushing callables to the queue. Workers pop tasks and execute them. For shutdown, close the queue or push sentinel values. In production, use `Concurrent::FixedThreadPool` or `ThreadPoolExecutor` from concurrent-ruby.

10. **"What is the concurrent-ruby gem?"** It's the de facto standard library for advanced concurrency in Ruby. It provides: atomic variables (`AtomicFixnum`, `AtomicBoolean`, `AtomicReference`), thread-safe collections (`Concurrent::Map`, `Concurrent::Array`), futures and promises, thread pools, read-write locks, and many other concurrency utilities.

11. **"How do you handle exceptions in threads?"** By default, exceptions in threads are silently swallowed — the thread dies but the main thread doesn't know. Call `join` or `value` to re-raise the exception. Set `Thread.abort_on_exception = true` to make all thread exceptions fatal. Use `Thread.report_on_exception = true` (default since Ruby 2.5) to print warnings.

12. **"When would you use threads vs processes vs Ractors?"** Threads for I/O-bound concurrency (HTTP requests, DB queries) — simple, shared memory. Processes (fork) for CPU-bound parallelism with full gem compatibility — isolated, stable. Ractors for CPU-bound parallelism with lower memory overhead — message passing, experimental. For most Ruby web apps, threads + process forking (Puma, Unicorn) is the standard approach.

13. **"How do you prevent deadlock?"** Four strategies: (a) consistent lock ordering — always acquire mutexes in the same order, (b) try_lock with backoff — release and retry if you can't get all locks, (c) lock hierarchy — assign levels to mutexes, only lock lower levels, (d) use `Queue`/`SizedQueue` instead of manual locking. Ruby detects simple deadlocks and raises `fatal: No live threads left. Deadlock?`.

14. **"What is the Reader-Writer problem?"** Multiple readers can access data concurrently (no modification), but writers need exclusive access. Use `Concurrent::ReadWriteLock`: `with_read_lock` for readers (concurrent), `with_write_lock` for writers (exclusive). Discuss trade-offs: reader-preference (writers may starve) vs writer-preference (readers may starve).

15. **"How does Ruby's Fiber Scheduler work?"** Ruby 3.0 introduced the Fiber Scheduler interface. When a fiber performs blocking I/O, the scheduler intercepts it, suspends the fiber, and runs another fiber. When the I/O completes, the original fiber is resumed. This enables async I/O without callbacks — code looks synchronous but runs concurrently. The `async` gem is the most popular implementation.
