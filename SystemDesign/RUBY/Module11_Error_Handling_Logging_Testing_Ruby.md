# Module 11: Error Handling, Logging & Testing in Ruby

> Robust software doesn't just work when everything goes right — it handles failures gracefully, logs what's happening for debugging and monitoring, and has tests that prove correctness. These three pillars — error handling, logging, and testing — are what separate production-quality code from prototype code. In LLD interviews, you're expected to handle edge cases, design observable systems, and write testable code. This module covers all three in depth with Ruby implementations.

---

## 11.1 Error Handling in Ruby

> Error handling is about anticipating what can go wrong and deciding how your program should respond. Ruby uses **exceptions** as its primary error handling mechanism, with `begin/rescue/ensure` blocks. Ruby also supports return-value-based error handling using `nil`, symbols, or custom Result objects for expected conditions. Choosing the right approach — and implementing it correctly — is critical for building reliable systems.

---

### Exceptions (begin, rescue, raise)

Exceptions are Ruby's primary mechanism for reporting and handling errors. When an error occurs, you **raise** an exception object. The runtime unwinds the call stack until it finds a matching **rescue** block.

```ruby
# Raising an exception
def divide(a, b)
  if b == 0.0
    raise "Division by zero"
    # Execution STOPS here — control transfers to the nearest rescue block
    # Code after raise is never executed
  end
  a / b
end

# Rescuing an exception
begin
  result = divide(10.0, 0.0)
  puts "Result: #{result}"  # never reached
rescue RuntimeError => e
  # Catches RuntimeError and its subclasses
  $stderr.puts "Error: #{e.message}"  # "Error: Division by zero"
end

puts "Program continues after handling the error"
```

**How exception propagation works:**

```ruby
def function_c
  raise RuntimeError, "Something went wrong in C"
  # Stack unwinding begins here
end

def function_b
  puts "B: before calling C"
  function_c  # exception raised inside
  puts "B: after calling C"  # NEVER executed — stack is unwinding
end

def function_a
  begin
    puts "A: before calling B"
    function_b
    puts "A: after calling B"  # NEVER executed
  rescue RuntimeError => e
    # Exception caught here — stack unwinding stops
    $stderr.puts "A caught: #{e.message}"
  end
  puts "A: continues after rescue"  # executed normally
end

function_a
# Output:
# A: before calling B
# B: before calling C
# A caught: Something went wrong in C
# A: continues after rescue
```

**Multiple rescue clauses — catching different exception types:**

```ruby
def process_input(input)
  begin
    raise ArgumentError, "Input cannot be empty" if input.empty?

    value = Integer(input)  # may raise ArgumentError or TypeError

    raise RangeError, "Value must be non-negative" if value < 0

    puts "Processing value: #{value}"
  rescue RangeError => e
    # Catches RangeError specifically
    $stderr.puts "Out of range: #{e.message}"
  rescue ArgumentError => e
    # Catches ArgumentError
    $stderr.puts "Invalid argument: #{e.message}"
  rescue StandardError => e
    # Catches any StandardError not caught above
    $stderr.puts "General error: #{e.message}"
  rescue Exception => e
    # Catches ANYTHING (including SignalException, SystemExit)
    # Almost never do this — it catches Ctrl+C and kill signals!
    $stderr.puts "Unknown error: #{e.message}"
  end
end
```

**Important rules:**
- Rescue clauses are checked **top to bottom** — put more specific types first
- Always rescue specific exception classes, not bare `rescue` (which catches `StandardError`)
- `rescue Exception` catches everything including signals — almost never use it
- An unrescued exception crashes your program with a stack trace
- Methods implicitly act as `begin/end` blocks — you can use `rescue` directly in a method

```ruby
# Rescue directly in a method (no begin/end needed)
def safe_divide(a, b)
  a / b
rescue ZeroDivisionError => e
  $stderr.puts "Cannot divide by zero: #{e.message}"
  nil
end
```

**Re-raising exceptions:**

```ruby
def middle_layer
  # ... some operation that might raise ...
  raise RuntimeError, "database connection failed"
rescue StandardError => e
  # Log the error at this level
  $stderr.puts "middle_layer caught: #{e.message}"

  # Re-raise the SAME exception (preserves the original type and backtrace)
  raise

  # Or wrap in a higher-level exception
  # raise RuntimeError, "Service error: #{e.message}"
end

def top_layer
  middle_layer
rescue RuntimeError => e
  $stderr.puts "top_layer caught: #{e.message}"
end
```

**raise vs raise e:**

```ruby
begin
  raise RuntimeError, "original error"
rescue StandardError => e
  raise        # GOOD: re-raises the original exception with original backtrace
  # raise e    # WORKS but resets the backtrace to this line — less useful for debugging
end
```

**The ensure and retry keywords:**

```ruby
# ensure — always runs, whether exception occurred or not (like C++ destructors / finally)
def read_file(path)
  file = File.open(path, "r")
  data = file.read
  data
rescue Errno::ENOENT => e
  $stderr.puts "File not found: #{e.message}"
  nil
ensure
  file&.close  # always runs — even if exception was raised
  puts "Cleanup complete"
end

# retry — re-execute the begin block (useful for transient failures)
def fetch_with_retry(url, max_retries: 3)
  attempts = 0
  begin
    attempts += 1
    puts "Attempt #{attempts}..."
    response = Net::HTTP.get(URI(url))
    response
  rescue Net::OpenTimeout, Net::ReadTimeout => e
    if attempts < max_retries
      $stderr.puts "Timeout, retrying... (#{attempts}/#{max_retries})"
      sleep(2 ** attempts)  # exponential backoff
      retry  # goes back to begin
    else
      raise  # re-raise after max retries exhausted
    end
  end
end
```

---

### Exception Hierarchy (Exception, StandardError, RuntimeError, etc.)

Ruby provides a hierarchy of built-in exception classes, all derived from `Exception`.

```
Exception                                ← base class (DO NOT rescue this broadly)
├── NoMemoryError                        ← out of memory
├── ScriptError                          ← errors in the script itself
│   ├── LoadError                        ← require/load failed
│   ├── NotImplementedError              ← method not implemented
│   └── SyntaxError                      ← syntax error in code
├── SecurityError                        ← security violation
├── SignalException                      ← OS signals (Ctrl+C, kill)
│   └── Interrupt                        ← Ctrl+C (SIGINT)
├── SystemExit                           ← exit / exit! called
├── SystemStackError                     ← stack overflow (deep recursion)
└── StandardError                        ← base for most application errors
    ├── ArgumentError                    ← bad method argument
    ├── EncodingError                    ← encoding mismatch
    ├── FiberError                       ← invalid Fiber operation
    ├── IOError                          ← I/O failure
    │   └── EOFError                     ← unexpected end of file
    ├── IndexError                       ← index out of bounds
    │   ├── KeyError                     ← hash key not found (fetch)
    │   └── StopIteration               ← iterator exhausted
    ├── LocalJumpError                   ← invalid yield/return/break
    ├── NameError                        ← undefined variable/method
    │   └── NoMethodError                ← method doesn't exist
    ├── RangeError                       ← value out of range
    │   └── FloatDomainError             ← invalid float operation
    ├── RegexpError                      ← invalid regex
    ├── RuntimeError                     ← generic runtime error (default for raise)
    │   └── FrozenError                  ← modifying a frozen object
    ├── SystemCallError                  ← OS-level errors (Errno::*)
    │   ├── Errno::ENOENT                ← file not found
    │   ├── Errno::EACCES                ← permission denied
    │   ├── Errno::ECONNREFUSED          ← connection refused
    │   └── Errno::ETIMEDOUT             ← connection timed out
    ├── ThreadError                      ← thread-related error
    ├── TypeError                        ← wrong type
    └── ZeroDivisionError                ← division by zero
```

**When to use which:**

```ruby
# ArgumentError — bad method argument (programmer error)
def set_age(age)
  raise ArgumentError, "Age must be between 0 and 150" unless (0..150).include?(age)
  @age = age
end

# RuntimeError — generic runtime failure (default for bare raise)
def read_file(path)
  raise "Failed to open file: #{path}" unless File.exist?(path)
  # bare raise creates a RuntimeError
  File.read(path)
end

# IndexError / RangeError — out of bounds
def get_element(array, index)
  if index < 0 || index >= array.size
    raise IndexError, "Index #{index} out of range [0, #{array.size - 1}]"
  end
  array[index]
end

# Errno::* — OS-level errors (raised automatically by IO operations)
begin
  File.read("/nonexistent/file.txt")
rescue Errno::ENOENT => e
  puts "File not found: #{e.message}"
rescue Errno::EACCES => e
  puts "Permission denied: #{e.message}"
end
```

**The message method:**

Every exception has a `message` method that returns a string describing the error, and a `backtrace` method for the stack trace.

```ruby
begin
  raise RuntimeError, "connection timed out"
rescue StandardError => e
  puts e.message    # "connection timed out"
  puts e.class      # RuntimeError
  puts e.backtrace.first(3).join("\n")  # first 3 lines of stack trace
end
```

---

### Custom Exception Classes

For real-world systems, you'll want domain-specific exception classes that carry additional context beyond a simple message.

```ruby
# Simple custom exception
class DatabaseError < StandardError; end

class ConnectionError < DatabaseError
  attr_reader :host, :port

  def initialize(host, port, msg)
    @host = host
    @port = port
    super("Connection to #{host}:#{port} failed: #{msg}")
  end
end

class QueryError < DatabaseError
  attr_reader :query, :error_code

  def initialize(query, code, msg)
    @query = query
    @error_code = code
    super("Query failed (code #{code}): #{msg}")
  end
end

# Usage
def execute_query(sql)
  # Simulate a query failure
  raise QueryError.new(sql, 1045, "Access denied for user 'root'")
end

def connect_to_db(host, port)
  raise ConnectionError.new(host, port, "Connection refused")
end

begin
  connect_to_db("localhost", 3306)
rescue ConnectionError => e
  $stderr.puts e.message
  $stderr.puts "Host: #{e.host}, Port: #{e.port}"
  # Retry logic, fallback to replica, etc.
rescue DatabaseError => e
  $stderr.puts "Database error: #{e.message}"
end
```

**Exception hierarchy for an LLD system:**

```ruby
# Base exception for the entire application
class AppError < StandardError
  attr_reader :code, :context

  def initialize(code, context, msg)
    @code = code
    @context = context
    super(msg)
  end
end

# Domain-specific exceptions
class AuthenticationError < AppError
  def initialize(msg)
    super(401, "auth", msg)
  end
end

class AuthorizationError < AppError
  def initialize(msg)
    super(403, "auth", msg)
  end
end

class NotFoundError < AppError
  attr_reader :resource_type, :resource_id

  def initialize(type, id)
    @resource_type = type
    @resource_id = id
    super(404, "resource", "#{type} with ID '#{id}' not found")
  end
end

class ValidationError < AppError
  attr_reader :field

  def initialize(field, msg)
    @field = field
    super(400, "validation", "Validation failed for '#{field}': #{msg}")
  end
end

# Usage in a service layer
class UserService
  def initialize(db)
    @db = db
  end

  def get_user(id)
    user = @db.find_user(id)
    raise NotFoundError.new("User", id) unless user
    user
  end

  def create_user(name, email)
    raise ValidationError.new("name", "cannot be empty") if name.nil? || name.empty?
    raise ValidationError.new("email", "invalid format") unless email.include?("@")
    # ...
  end
end
```

---

### Exception Safety in Ruby (ensure, Block Patterns)

Ruby doesn't have C++'s RAII or destructors, but it achieves exception safety through **ensure blocks**, **block-based resource management**, and **idiomatic patterns**. The key principle is the same: resources must be cleaned up even when exceptions occur.

**The ensure block — Ruby's equivalent of RAII/finally:**

```ruby
# BAD: Manual resource management — not exception-safe
def bad_example
  file = File.open("data.txt", "r")
  data = process_data(file)  # if this raises, file is LEAKED
  file.close                 # never reached if exception raised
  data
end

# GOOD: ensure block — exception-safe
def good_example_ensure
  file = File.open("data.txt", "r")
  process_data(file)
rescue StandardError => e
  $stderr.puts "Error: #{e.message}"
  nil
ensure
  file&.close  # always runs — even if exception was raised
end

# BEST: Block form — automatic cleanup (Ruby's idiomatic RAII)
def best_example
  File.open("data.txt", "r") do |file|
    process_data(file)
  end
  # File is automatically closed when the block exits — even on exception
end
```

**Block-based resource management — Ruby's idiomatic RAII:**

```ruby
# File.open with block — auto-closes
File.open("output.txt", "w") do |f|
  f.write("Hello")
  raise "oops"  # file is STILL closed automatically
end

# Mutex#synchronize — auto-unlocks
mutex = Mutex.new
mutex.synchronize do
  do_work  # if this raises, mutex is STILL unlocked
end

# Custom RAII-style wrapper using blocks
class FileHandle
  def self.open(path, mode)
    handle = new(path, mode)
    yield handle
  ensure
    handle&.close
  end

  def initialize(path, mode)
    @file = File.open(path, mode)
    raise "Failed to open: #{path}" unless @file
  end

  def close
    @file&.close
    @file = nil
  end

  def read
    @file.read
  end

  def write(data)
    @file.write(data)
  end
end

# Usage — file is always closed
FileHandle.open("data.txt", "r") do |fh|
  data = fh.read
  process(data)
end

# Database transaction pattern — auto-rollback on exception
class Transaction
  def initialize(db)
    @db = db
    @committed = false
  end

  def self.wrap(db)
    txn = new(db)
    db.begin_transaction
    yield txn
  rescue StandardError => e
    db.rollback unless txn.committed?
    raise
  ensure
    # If not committed and no exception, still rollback (safety net)
    db.rollback unless txn.committed?
  end

  def commit
    @db.commit_transaction
    @committed = true
  end

  def committed?
    @committed
  end
end

def transfer_money(db, from, to, amount)
  Transaction.wrap(db) do |txn|
    db.debit(from, amount)   # might raise
    db.credit(to, amount)    # might raise
    txn.commit               # only reached if both succeed
    # If any exception is raised, Transaction.wrap auto-rolls back
  end
end
```

**Exception safety levels in Ruby:**

| Guarantee | Ruby Approach | Example |
|-----------|--------------|---------|
| **No-raise** | Simple getters, frozen objects | `attr_reader :name` |
| **Strong** | Clone-and-swap, transactions | Modify copy, then replace original |
| **Basic** | ensure blocks, block patterns | No resource leaks, valid state |

```ruby
# STRONG guarantee — clone-and-swap
class UserList
  def initialize
    @users = []
  end

  # Strong guarantee: either fully succeeds or state is unchanged
  def replace_all(new_users)
    temp = new_users.dup  # copy — might raise (original unchanged)
    @users = temp         # assignment is atomic in Ruby
  end

  # Strong guarantee using transaction-style approach
  def bulk_add(new_users)
    temp = @users.dup  # copy current state
    new_users.each do |user|
      temp << user  # modify the copy
    end
    @users = temp  # commit — atomic swap
  end
end

# BASIC guarantee — ensure cleanup
class Connection
  def execute(query)
    acquire_lock
    result = run_query(query)  # might raise
    result
  ensure
    release_lock  # always runs — no resource leak
  end
end
```

---

### Return Values vs Exceptions (When to Use Which)

This is a design decision that affects your entire codebase. Ruby is exception-friendly, but not everything should be an exception.

**Return values — return nil, symbols, or Result objects for expected conditions:**

```ruby
# Using nil for "not found" (very common in Ruby)
class UserRepository
  def find_by_id(id)
    @users[id]  # returns nil if not found — this is expected, not exceptional
  end

  def find_by_id!(id)
    @users[id] || raise(NotFoundError.new("User", id))
    # Convention: bang (!) methods raise on failure
  end
end

# Using symbols for status
def create_user(name, email)
  return :invalid_name if name.nil? || name.empty?
  return :invalid_email unless email.include?("@")
  return :duplicate if @users.key?(email)

  @users[email] = { name: name, email: email }
  :ok
end

# Caller
status = create_user("Alice", "alice@example.com")
case status
when :ok then puts "User created"
when :invalid_name then puts "Name is required"
when :invalid_email then puts "Invalid email"
when :duplicate then puts "User already exists"
end
```

**When to use exceptions vs return values:**

| Criteria | Exceptions | Return Values (nil/symbols/Result) |
|----------|-----------|-----------------------------------|
| **Frequency** | Rare, unexpected failures | Common, expected conditions |
| **Examples** | Network down, file corruption, out of memory | User not found, invalid input, empty result |
| **Performance** | Expensive when raised | Negligible cost |
| **Propagation** | Automatic (stack unwinding) | Manual (must check at every level) |
| **Ignorability** | Hard to ignore (crashes if unrescued) | Easy to ignore (silent nil bugs) |
| **Code clarity** | Clean happy path | Checks interleaved with logic |
| **Ruby convention** | `method!` (bang methods) | `method` (query methods) |

**Best practice — use both:**

```ruby
# Exceptions for truly exceptional situations
def connect_to_database(host, port)
  # Network failure, authentication failure — exceptional
  raise ConnectionError.new(host, port, "Connection refused")
end

# Return nil for expected "not found" cases
def find_user(id)
  @users[id]  # returns nil if not found — expected, not exceptional
end

# Bang method raises for callers who want exceptions
def find_user!(id)
  find_user(id) || raise(NotFoundError.new("User", id))
end

# Caller chooses the style
user = find_user(user_id)
if user
  process(user)
else
  puts "User not found"
end

# Or use the bang version when absence is unexpected
user = find_user!(user_id)  # raises if not found
process(user)
```

---

### Result Objects and Monadic Error Handling

Ruby doesn't have `std::optional` or `std::expected`, but you can build equivalent patterns with Result objects. This is popular in service layers where you want structured success/failure without exceptions.

**Custom Result type:**

```ruby
class Result
  attr_reader :value, :error

  def initialize(value: nil, error: nil)
    @value = value
    @error = error
  end

  def self.ok(value)
    new(value: value)
  end

  def self.err(error)
    new(error: error)
  end

  def ok?
    @error.nil?
  end

  def err?
    !ok?
  end

  # Transform the success value
  def map
    return self if err?
    Result.ok(yield(@value))
  end

  # Chain operations that return Results
  def and_then
    return self if err?
    yield(@value)
  end

  # Provide a default value on error
  def unwrap_or(default)
    ok? ? @value : default
  end

  # Raise if error, return value if ok
  def unwrap!
    raise @error if err? && @error.is_a?(Exception)
    raise RuntimeError, @error.to_s if err?
    @value
  end

  def to_s
    ok? ? "Ok(#{@value})" : "Err(#{@error})"
  end
end

# Usage
def parse_int(input)
  return Result.err(:empty_input) if input.nil? || input.empty?

  begin
    value = Integer(input)
    return Result.err(:out_of_range) if value < 0 || value > 1000
    Result.ok(value)
  rescue ArgumentError
    Result.err(:invalid_format)
  end
end

result = parse_int("42")
if result.ok?
  puts "Parsed: #{result.value}"
else
  case result.error
  when :empty_input then puts "Empty input"
  when :invalid_format then puts "Invalid format"
  when :out_of_range then puts "Out of range"
  end
end

# Chaining with map and and_then
def find_user(id)
  user = @users[id]
  user ? Result.ok(user) : Result.err(:not_found)
end

def get_user_email(id)
  find_user(id)
    .map { |user| user[:email] }
    .and_then { |email| email ? Result.ok(email) : Result.err(:no_email) }
end

# unwrap_or for defaults
name = find_user(42).map { |u| u[:name] }.unwrap_or("Unknown")
```

**Ruby's built-in nil-safe navigation (the &. operator):**

```ruby
# Safe navigation operator — Ruby's lightweight "optional"
user = find_user(42)
email = user&.email          # nil if user is nil, otherwise user.email
name = user&.profile&.name   # chains safely through nil

# dig — safe nested hash/array access
config = { database: { host: "localhost", port: 3306 } }
host = config.dig(:database, :host)  # "localhost"
missing = config.dig(:database, :ssl)  # nil (no exception)
```

---


## 11.2 Logging

> Logging is the practice of recording events, errors, and diagnostic information during program execution. Good logging is essential for debugging, monitoring, auditing, and understanding system behavior in production. In LLD, designing a logger is a classic interview problem that combines Singleton, Strategy, Observer, and Chain of Responsibility patterns.

---

### Logging Levels (TRACE, DEBUG, INFO, WARN, ERROR, FATAL)

Logging levels control the verbosity of log output. Each level represents a severity — you configure the minimum level, and only messages at that level or above are recorded.

```ruby
module LogLevel
  TRACE = 0  # Most verbose — detailed execution flow
  DEBUG = 1  # Debugging information — variable values, state changes
  INFO  = 2  # General information — startup, shutdown, config loaded
  WARN  = 3  # Warning — something unexpected but recoverable
  ERROR = 4  # Error — operation failed but system continues
  FATAL = 5  # Fatal — system cannot continue, about to crash

  NAMES = {
    TRACE => "TRACE",
    DEBUG => "DEBUG",
    INFO  => "INFO",
    WARN  => "WARN",
    ERROR => "ERROR",
    FATAL => "FATAL"
  }.freeze

  def self.to_s(level)
    NAMES.fetch(level, "UNKNOWN")
  end
end
```

**Ruby's built-in Logger already defines severity levels:**

```ruby
require 'logger'

# Ruby's stdlib Logger uses these levels:
# Logger::DEBUG   = 0
# Logger::INFO    = 1
# Logger::WARN    = 2
# Logger::ERROR   = 3
# Logger::FATAL   = 4
# Logger::UNKNOWN = 5
```

**When to use each level:**

| Level | Purpose | Example |
|-------|---------|---------|
| TRACE | Extremely detailed flow | "Entering method X with params a=1, b=2" |
| DEBUG | Diagnostic information | "Cache hit for key 'user_123'" |
| INFO | Normal operations | "Server started on port 8080" |
| WARN | Unexpected but handled | "Retry attempt 2/3 for API call" |
| ERROR | Operation failed | "Failed to save user: database timeout" |
| FATAL | System-level failure | "Out of memory — shutting down" |

**Production environments** typically run at INFO or WARN level. DEBUG and TRACE are enabled during development or when investigating issues.

---

### Ruby's Built-in Logger

Ruby ships with a `Logger` class in its standard library. It's simple, thread-safe, and sufficient for many applications.

```ruby
require 'logger'

# Basic usage
logger = Logger.new($stdout)
logger.level = Logger::INFO

logger.debug("This won't appear — level is INFO")
logger.info("Server started on port 8080")
logger.warn("Disk usage at 85%")
logger.error("Failed to connect to database")
logger.fatal("Out of memory — shutting down")

# Output:
# I, [2025-01-15T14:30:45.123456 #12345]  INFO -- : Server started on port 8080
# W, [2025-01-15T14:30:45.123789 #12345]  WARN -- : Disk usage at 85%
# ...

# Log to a file
file_logger = Logger.new("app.log")
file_logger.info("Application started")

# Log to a file with rotation
# Rotate when file exceeds 10MB, keep 5 old files
rotating_logger = Logger.new("app.log", 5, 10 * 1024 * 1024)

# Daily rotation
daily_logger = Logger.new("app.log", "daily")

# Custom format
logger.formatter = proc do |severity, datetime, progname, msg|
  "#{datetime.strftime('%Y-%m-%d %H:%M:%S.%L')} [#{severity}] #{msg}\n"
end

logger.info("Custom formatted message")
# Output: 2025-01-15 14:30:45.123 [INFO] Custom formatted message
```

---

### Logger Design (Singleton Logger with Multiple Sinks)

For LLD interviews, you'll want to design a logger from scratch that demonstrates Singleton, Strategy (sinks), and other patterns.

```ruby
require 'singleton'
require 'time'

class AppLogger
  include Singleton

  def initialize
    @min_level = LogLevel::INFO
    @sinks = []
    @mutex = Mutex.new
  end

  def level=(level)
    @min_level = level
  end

  def level
    @min_level
  end

  def add_sink(sink)
    @mutex.synchronize { @sinks << sink }
  end

  def log(level, message, file: nil, line: nil)
    return if level < @min_level

    formatted = format_message(level, message, file: file, line: line)

    @mutex.synchronize do
      @sinks.each { |sink| sink.write(formatted, level) }
    end
  end

  # Convenience methods
  def trace(msg, **opts) = log(LogLevel::TRACE, msg, **opts)
  def debug(msg, **opts) = log(LogLevel::DEBUG, msg, **opts)
  def info(msg, **opts)  = log(LogLevel::INFO, msg, **opts)
  def warn(msg, **opts)  = log(LogLevel::WARN, msg, **opts)
  def error(msg, **opts) = log(LogLevel::ERROR, msg, **opts)
  def fatal(msg, **opts) = log(LogLevel::FATAL, msg, **opts)

  private

  def format_message(level, message, file: nil, line: nil)
    now = Time.now
    timestamp = now.strftime("%Y-%m-%d %H:%M:%S.") + format("%03d", now.usec / 1000)

    parts = [timestamp, "[#{LogLevel.to_s(level)}]"]
    parts << "[#{file}:#{line}]" if file
    parts << message
    parts.join(" ")
  end
end

# Global convenience method (optional)
def log
  AppLogger.instance
end
```

---

### Log Sinks (Multiple Output Targets)

A **sink** (or appender) is a destination for log messages. Using the Strategy pattern, the logger can write to multiple sinks simultaneously.

```ruby
# Abstract sink interface
class LogSink
  def write(message, level)
    raise NotImplementedError, "#{self.class}#write must be implemented"
  end
end

# Console sink — writes to stdout/stderr
class ConsoleSink < LogSink
  def write(message, level)
    if level >= LogLevel::ERROR
      $stderr.puts message  # errors to stderr
    else
      $stdout.puts message  # everything else to stdout
    end
  end
end

# File sink — writes to a log file
class FileSink < LogSink
  def initialize(path)
    @file = File.open(path, "a")
    @mutex = Mutex.new
    raise "Failed to open log file: #{path}" unless @file
  end

  def write(message, level)
    @mutex.synchronize do
      @file.puts message
      @file.flush  # ensure it's written immediately
    end
  end

  def close
    @file&.close
  end
end

# Rotating file sink — creates new file when size limit is reached
class RotatingFileSink < LogSink
  def initialize(path, max_bytes: 10 * 1024 * 1024, max_files: 5)
    @base_path = path
    @max_size = max_bytes
    @max_files = max_files
    @current_size = 0
    @file_index = 0
    @mutex = Mutex.new
    @current_file = File.open(path, "a")
    @current_size = @current_file.size
  end

  def write(message, level)
    @mutex.synchronize do
      rotate if @current_size + message.bytesize > @max_size
      @current_file.puts message
      @current_size += message.bytesize + 1
    end
  end

  private

  def rotate
    @current_file.close
    @file_index = (@file_index + 1) % @max_files
    path = "#{@base_path}.#{@file_index}"
    @current_file = File.open(path, "w")  # truncate old file
    @current_size = 0
  end
end

# Filtered sink — only writes messages at or above a certain level
class FilteredSink < LogSink
  def initialize(inner_sink, min_level)
    @inner = inner_sink
    @min_level = min_level
  end

  def write(message, level)
    @inner.write(message, level) if level >= @min_level
  end
end

# Setup example
def setup_logging
  logger = AppLogger.instance
  logger.level = LogLevel::DEBUG

  # Console: show INFO and above
  logger.add_sink(FilteredSink.new(ConsoleSink.new, LogLevel::INFO))

  # File: log everything (DEBUG and above)
  logger.add_sink(FileSink.new("app.log"))

  # Rotating file: errors only, max 10MB per file, keep 5 files
  logger.add_sink(FilteredSink.new(
    RotatingFileSink.new("error.log", max_bytes: 10 * 1024 * 1024, max_files: 5),
    LogLevel::ERROR
  ))

  logger.info("Logging system initialized")
end
```

---

### Log Formatting and Rotation

**Log format components:**

```
2025-01-15 14:30:45.123 [INFO] [user_service.rb:42] [req-abc123] User login successful: user_id=456
│                        │      │                    │            │
│                        │      │                    │            └── Message
│                        │      │                    └── Correlation/Request ID
│                        │      └── Source location (file:line)
│                        └── Log level
└── Timestamp with milliseconds
```

**Configurable log formatter:**

```ruby
# Abstract formatter
class LogFormatter
  def format(level, message, file: nil, line: nil, thread_id: nil)
    raise NotImplementedError
  end
end

# Simple text formatter
class TextFormatter < LogFormatter
  def format(level, message, file: nil, line: nil, thread_id: nil)
    parts = [timestamp]
    parts << "[#{LogLevel.to_s(level)}]"
    parts << "[thread:#{thread_id}]" if thread_id
    parts << "[#{file}:#{line}]" if file
    parts << message
    parts.join(" ")
  end

  private

  def timestamp
    now = Time.now
    now.strftime("%Y-%m-%d %H:%M:%S.") + format("%03d", now.usec / 1000)
  end
end

# JSON formatter (for structured logging — see below)
class JsonFormatter < LogFormatter
  def format(level, message, file: nil, line: nil, thread_id: nil, **fields)
    entry = {
      timestamp: Time.now.utc.iso8601(3),
      level: LogLevel.to_s(level),
      message: message
    }
    entry[:file] = file if file
    entry[:line] = line if line
    entry[:thread_id] = thread_id if thread_id
    entry.merge!(fields)

    require 'json'
    JSON.generate(entry)
  end
end
```

**Log rotation strategies:**

| Strategy | How It Works | Use Case |
|----------|-------------|----------|
| Size-based | Rotate when file exceeds N bytes | General purpose |
| Time-based | Rotate daily/hourly | Compliance, archival |
| Count-based | Keep at most N log files | Disk space management |
| Compress on rotate | gzip old log files | Long-term storage |

**Ruby's built-in Logger handles rotation:**

```ruby
require 'logger'

# Size-based rotation: 5 files, 10MB each
logger = Logger.new("app.log", 5, 10 * 1024 * 1024)

# Time-based rotation
daily_logger = Logger.new("app.log", "daily")
weekly_logger = Logger.new("app.log", "weekly")
monthly_logger = Logger.new("app.log", "monthly")
```

---

### Structured Logging

Traditional logging produces human-readable text. **Structured logging** produces machine-parseable records (JSON, key-value pairs) that can be easily searched, filtered, and analyzed by log aggregation tools (ELK stack, Splunk, Datadog).

```ruby
require 'json'
require 'time'

# Structured log entry with fluent API
class LogEntry
  def initialize(level, message)
    @level = level
    @message = message
    @fields = {}
    @timestamp = Time.now.utc
  end

  # Fluent API for adding fields
  def with(key, value)
    @fields[key] = value
    self
  end

  # Format as JSON
  def to_json
    entry = {
      timestamp: @timestamp.iso8601(3),
      level: LogLevel.to_s(@level),
      message: @message
    }.merge(@fields)

    JSON.generate(entry)
  end

  # Format as key-value pairs
  def to_kv
    parts = [
      "timestamp=#{@timestamp.iso8601(3)}",
      "level=#{LogLevel.to_s(@level)}",
      "msg=\"#{@message}\""
    ]

    @fields.each do |key, value|
      parts << "#{key}=\"#{value}\""
    end

    parts.join(" ")
  end

  attr_reader :level
end

# Usage
def handle_request(user_id, endpoint)
  start = Process.clock_gettime(Process::CLOCK_MONOTONIC)

  # ... process request ...

  elapsed_ms = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start) * 1000).round

  entry = LogEntry.new(LogLevel::INFO, "Request processed")
    .with(:user_id, user_id)
    .with(:endpoint, endpoint)
    .with(:duration_ms, elapsed_ms)
    .with(:status, 200)

  # JSON output:
  # {"timestamp":"2025-01-15T14:30:45.123Z","level":"INFO",
  #  "message":"Request processed","user_id":"user_123",
  #  "endpoint":"/api/users","duration_ms":42,"status":200}
  puts entry.to_json
end
```

**Benefits of structured logging:**
- **Searchable** — query by any field (`user_id=123 AND status=500`)
- **Aggregatable** — compute averages, percentiles (`avg(duration_ms)`)
- **Alertable** — trigger alerts on patterns (`error_count > 100 in 5min`)
- **Parseable** — no regex needed to extract fields from log lines

---


## 11.3 Unit Testing (Ruby)

> Unit testing verifies that individual components (functions, classes, methods) work correctly in isolation. Tests catch bugs early, enable safe refactoring, and serve as living documentation. In Ruby, **Minitest** is the built-in testing framework (ships with Ruby), and **RSpec** is the most popular community framework. Both support mocking and stubbing for testing components with dependencies.

---

### Minitest Framework

Minitest ships with Ruby — no gem installation needed. It provides a simple, fast testing framework with assertions and mocking.

**Basic test structure:**

```ruby
# calculator.rb
class Calculator
  def add(a, b)
    a + b
  end

  def subtract(a, b)
    a - b
  end

  def multiply(a, b)
    a * b
  end

  def divide(a, b)
    raise ArgumentError, "Division by zero" if b == 0
    a.to_f / b
  end
end
```

```ruby
# calculator_test.rb
require 'minitest/autorun'
require_relative 'calculator'

class CalculatorTest < Minitest::Test
  def setup
    @calc = Calculator.new
  end

  def test_add_positive_numbers
    assert_equal 5, @calc.add(2, 3)
  end

  def test_add_negative_numbers
    assert_equal(-5, @calc.add(-2, -3))
  end

  def test_add_zero
    assert_equal 5, @calc.add(5, 0)
    assert_equal 5, @calc.add(0, 5)
    assert_equal 0, @calc.add(0, 0)
  end

  def test_subtract_numbers
    assert_equal 7, @calc.subtract(10, 3)
    assert_equal(-7, @calc.subtract(3, 10))
  end

  def test_multiply_numbers
    assert_equal 20, @calc.multiply(4, 5)
    assert_equal(-6, @calc.multiply(-2, 3))
    assert_equal 0, @calc.multiply(0, 100)
  end

  def test_divide_numbers
    assert_in_delta 3.333, @calc.divide(10, 3), 0.001
    assert_equal 0.0, @calc.divide(0, 5)
  end

  def test_divide_by_zero_raises
    assert_raises(ArgumentError) { @calc.divide(10, 0) }
  end
end
```

**Running tests:**

```bash
# Run a single test file
ruby calculator_test.rb

# Output:
# Run options: --seed 12345
#
# # Running:
#
# .......
#
# Finished in 0.001234s, 5672.3456 runs/s, 8102.1234 assertions/s.
#
# 7 runs, 10 assertions, 0 failures, 0 errors, 0 skips
```

---

### RSpec Framework

RSpec is Ruby's most popular testing framework. It uses a descriptive, behavior-driven syntax that reads like documentation.

**Basic test structure:**

```ruby
# spec/calculator_spec.rb
require_relative '../calculator'

RSpec.describe Calculator do
  subject(:calc) { described_class.new }

  describe '#add' do
    it 'adds positive numbers' do
      expect(calc.add(2, 3)).to eq(5)
    end

    it 'adds negative numbers' do
      expect(calc.add(-2, -3)).to eq(-5)
    end

    it 'adds zero' do
      expect(calc.add(5, 0)).to eq(5)
      expect(calc.add(0, 5)).to eq(5)
      expect(calc.add(0, 0)).to eq(0)
    end
  end

  describe '#subtract' do
    it 'subtracts numbers' do
      expect(calc.subtract(10, 3)).to eq(7)
      expect(calc.subtract(3, 10)).to eq(-7)
    end
  end

  describe '#multiply' do
    it 'multiplies numbers' do
      expect(calc.multiply(4, 5)).to eq(20)
      expect(calc.multiply(-2, 3)).to eq(-6)
      expect(calc.multiply(0, 100)).to eq(0)
    end
  end

  describe '#divide' do
    it 'divides numbers' do
      expect(calc.divide(10, 3)).to be_within(0.001).of(3.333)
      expect(calc.divide(0, 5)).to eq(0.0)
    end

    it 'raises ArgumentError when dividing by zero' do
      expect { calc.divide(10, 0) }.to raise_error(ArgumentError, "Division by zero")
    end
  end
end
```

**Running RSpec:**

```bash
# Install RSpec
gem install rspec

# Run all specs
rspec

# Run a specific file
rspec spec/calculator_spec.rb

# Run with documentation format
rspec --format documentation

# Output:
# Calculator
#   #add
#     adds positive numbers
#     adds negative numbers
#     adds zero
#   #subtract
#     subtracts numbers
#   #multiply
#     multiplies numbers
#   #divide
#     divides numbers
#     raises ArgumentError when dividing by zero
#
# 7 examples, 0 failures
```

---

### Assertions (Minitest) and Matchers (RSpec)

**Minitest assertions:**

```ruby
class AssertionExamplesTest < Minitest::Test
  # Boolean assertions
  def test_boolean_assertions
    connected = true
    assert connected                    # truthy
    refute !connected                   # not falsy (same as assert)
    assert_equal true, connected        # strict equality
  end

  # Equality assertions
  def test_equality_assertions
    assert_equal 4, 2 + 2              # ==
    refute_equal 5, 2 + 2              # !=
  end

  # Comparison assertions
  def test_comparison_assertions
    assert_operator 3, :<, 5           # 3 < 5
    assert_operator 3, :<=, 3          # 3 <= 3
    assert_operator 5, :>, 3           # 5 > 3
    assert_operator 5, :>=, 5          # 5 >= 5
  end

  # Floating-point assertions
  def test_floating_point_assertions
    result = 0.1 + 0.2
    # assert_equal 0.3, result  # FAILS due to floating-point precision!
    assert_in_delta 0.3, result, 1e-10   # within tolerance
    assert_in_epsilon 0.3, result, 0.01  # within relative tolerance
  end

  # String assertions
  def test_string_assertions
    greeting = "Hello, World!"
    assert_equal "Hello, World!", greeting
    refute_equal "hello, world!", greeting
    assert_match(/Hello/, greeting)       # regex match
    assert_includes greeting, "World"     # string contains
  end

  # Exception assertions
  def test_exception_assertions
    # Expect a specific exception type
    error = assert_raises(RuntimeError) { raise "oops" }
    assert_equal "oops", error.message

    # Expect a specific exception class
    assert_raises(ArgumentError) { Integer("not a number") }
  end

  # Nil assertions
  def test_nil_assertions
    assert_nil nil
    refute_nil "something"
  end

  # Type assertions
  def test_type_assertions
    assert_instance_of String, "hello"     # exact class
    assert_kind_of Numeric, 42             # class or superclass
    assert_respond_to "hello", :upcase     # duck typing
  end

  # Collection assertions
  def test_collection_assertions
    list = [1, 2, 3, 4, 5]
    assert_includes list, 3                # element in collection
    assert_empty []                        # collection is empty
    refute_empty list                      # collection is not empty
  end

  # Custom failure messages
  def test_custom_messages
    value = 42
    assert_equal 42, value, "Value should be 42 but got #{value}"
  end
end
```

**RSpec matchers:**

```ruby
RSpec.describe "Matcher Examples" do
  # Equality matchers
  it "equality" do
    expect(2 + 2).to eq(4)          # ==
    expect(2 + 2).not_to eq(5)      # !=
    expect("hello").to eql("hello")  # eql? (type + value)
    expect([1, 2]).to equal([1, 2])  # equal? would fail (different objects)
  end

  # Comparison matchers
  it "comparison" do
    expect(3).to be < 5
    expect(3).to be <= 3
    expect(5).to be > 3
    expect(5).to be >= 5
    expect(5).to be_between(1, 10)
  end

  # Floating-point matchers
  it "floating point" do
    expect(0.1 + 0.2).to be_within(1e-10).of(0.3)
  end

  # String matchers
  it "strings" do
    expect("Hello, World!").to eq("Hello, World!")
    expect("Hello, World!").to match(/Hello/)
    expect("Hello, World!").to include("World")
    expect("Hello, World!").to start_with("Hello")
    expect("Hello, World!").to end_with("World!")
  end

  # Exception matchers
  it "exceptions" do
    expect { raise RuntimeError, "oops" }.to raise_error(RuntimeError)
    expect { raise RuntimeError, "oops" }.to raise_error(RuntimeError, "oops")
    expect { raise RuntimeError, "oops" }.to raise_error(RuntimeError, /oops/)
    expect { 2 + 2 }.not_to raise_error
  end

  # Collection matchers
  it "collections" do
    expect([1, 2, 3]).to include(2)
    expect([1, 2, 3]).to contain_exactly(3, 1, 2)  # any order
    expect([1, 2, 3]).to match_array([3, 2, 1])     # any order
    expect([]).to be_empty
    expect([1, 2, 3]).to all(be > 0)
    expect([1, 2, 3]).to have_attributes(size: 3)    # for objects
  end

  # Type matchers
  it "types" do
    expect("hello").to be_a(String)
    expect(42).to be_a(Numeric)
    expect("hello").to respond_to(:upcase)
  end

  # Truthiness matchers
  it "truthiness" do
    expect(true).to be true
    expect(false).to be false
    expect(nil).to be_nil
    expect("hello").to be_truthy
    expect(nil).to be_falsy
  end

  # Change matchers
  it "change" do
    array = [1, 2, 3]
    expect { array.push(4) }.to change { array.size }.from(3).to(4)
    expect { array.push(5) }.to change { array.size }.by(1)
  end

  # Predicate matchers (any method ending in ? becomes a matcher)
  it "predicates" do
    expect([]).to be_empty          # calls empty?
    expect(1).to be_odd             # calls odd?
    expect(2).to be_even            # calls even?
    expect(nil).to be_nil           # calls nil?
    expect("hello").to be_frozen    # calls frozen? (if frozen)
  end
end
```

---

### Test Fixtures (setup/teardown and before/after)

A **test fixture** sets up common test state. Tests in the fixture share the setup/teardown logic but each test gets a fresh instance.

**Minitest fixtures:**

```ruby
require 'minitest/autorun'

class UserRepository
  def initialize
    @users = {}
    @next_id = 1
  end

  def add_user(name)
    id = @next_id
    @next_id += 1
    @users[id] = name
    id
  end

  def find_user(id)
    @users[id]
  end

  def remove_user(id)
    @users.delete(id) != nil
  end

  def count
    @users.size
  end
end

class UserRepositoryTest < Minitest::Test
  # Called before EACH test
  def setup
    @repo = UserRepository.new
    @alice_id = @repo.add_user("Alice")
    @bob_id = @repo.add_user("Bob")
  end

  # Called after EACH test
  def teardown
    # Cleanup if needed (e.g., close connections, delete temp files)
  end

  def test_find_existing_user
    user = @repo.find_user(@alice_id)
    assert_equal "Alice", user
  end

  def test_find_non_existing_user
    user = @repo.find_user(999)
    assert_nil user
  end

  def test_remove_user
    assert @repo.remove_user(@alice_id)
    assert_nil @repo.find_user(@alice_id)
    assert_equal 1, @repo.count  # only Bob remains
  end

  def test_remove_non_existing_user
    refute @repo.remove_user(999)
    assert_equal 2, @repo.count  # nothing removed
  end

  def test_count_after_setup
    # Each test gets a fresh fixture — setup runs again
    assert_equal 2, @repo.count  # Alice and Bob from setup
  end
end
```

**RSpec fixtures (before/after, let, subject):**

```ruby
RSpec.describe UserRepository do
  # let — lazy-evaluated, memoized per test
  let(:repo) { UserRepository.new }
  let(:alice_id) { repo.add_user("Alice") }
  let(:bob_id) { repo.add_user("Bob") }

  # before — runs before each test (like setup)
  before do
    # Force evaluation of lazy lets
    alice_id
    bob_id
  end

  # after — runs after each test (like teardown)
  after do
    # Cleanup if needed
  end

  describe '#find_user' do
    it 'returns the user when found' do
      expect(repo.find_user(alice_id)).to eq("Alice")
    end

    it 'returns nil when not found' do
      expect(repo.find_user(999)).to be_nil
    end
  end

  describe '#remove_user' do
    it 'removes an existing user' do
      expect(repo.remove_user(alice_id)).to be true
      expect(repo.find_user(alice_id)).to be_nil
      expect(repo.count).to eq(1)
    end

    it 'returns false for non-existing user' do
      expect(repo.remove_user(999)).to be false
      expect(repo.count).to eq(2)
    end
  end

  describe '#count' do
    it 'returns the number of users after setup' do
      expect(repo.count).to eq(2)
    end
  end
end
```

**RSpec shared examples and contexts:**

```ruby
# Shared examples — reusable test groups
RSpec.shared_examples "a collection" do
  it "starts empty" do
    expect(subject).to be_empty
  end

  it "can add items" do
    subject.add("item")
    expect(subject).not_to be_empty
  end
end

RSpec.describe Stack do
  it_behaves_like "a collection"
end

RSpec.describe Queue do
  it_behaves_like "a collection"
end

# Shared context — reusable setup
RSpec.shared_context "with database" do
  let(:db) { MockDatabase.new }
  before { db.connect }
  after { db.disconnect }
end

RSpec.describe UserService do
  include_context "with database"

  it "uses the shared database" do
    expect(db).to be_connected
  end
end
```

**Parameterized tests in RSpec:**

```ruby
def prime?(n)
  return false if n < 2
  (2..Math.sqrt(n).to_i).none? { |i| n % i == 0 }
end

RSpec.describe '#prime?' do
  [
    [0, false],
    [1, false],
    [2, true],
    [3, true],
    [4, false],
    [5, true],
    [97, true],
    [100, false]
  ].each do |input, expected|
    it "returns #{expected} for #{input}" do
      expect(prime?(input)).to eq(expected)
    end
  end
end
```

---

### Mocking and Stubbing

**Mocking** replaces real dependencies with controlled substitutes. This lets you test a class in isolation without needing a real database, network, or file system.

**Minitest mocking:**

```ruby
require 'minitest/autorun'

# Interface (duck type) — the dependency
class UserService
  def initialize(db, email_service)
    @db = db
    @email_service = email_service
  end

  def get_user_name(user_id)
    result = @db.get("user:#{user_id}")
    result || "Unknown"
  end

  def save_user(user_id, name)
    @db.set("user:#{user_id}", name)
  end
end

class UserServiceTest < Minitest::Test
  def test_get_user_name_returns_name_when_found
    # Create a mock
    mock_db = Minitest::Mock.new
    mock_db.expect(:get, "Alice", ["user:123"])

    service = UserService.new(mock_db, nil)
    name = service.get_user_name("123")

    assert_equal "Alice", name
    mock_db.verify  # ensures all expectations were met
  end

  def test_get_user_name_returns_unknown_when_not_found
    mock_db = Minitest::Mock.new
    mock_db.expect(:get, nil, ["user:456"])

    service = UserService.new(mock_db, nil)
    name = service.get_user_name("456")

    assert_equal "Unknown", name
    mock_db.verify
  end

  # Using stub for simpler cases
  def test_save_user_calls_database
    called_with = nil
    mock_db = Object.new
    mock_db.define_singleton_method(:set) do |key, value|
      called_with = [key, value]
      true
    end

    service = UserService.new(mock_db, nil)
    result = service.save_user("123", "Alice")

    assert result
    assert_equal ["user:123", "Alice"], called_with
  end
end
```

**RSpec mocking (doubles, stubs, expectations):**

```ruby
RSpec.describe UserService do
  # Create test doubles
  let(:mock_db) { instance_double("Database") }
  let(:mock_email) { instance_double("EmailService") }
  let(:service) { UserService.new(mock_db, mock_email) }

  describe '#get_user_name' do
    it 'returns the name when found' do
      # Stub: when get("user:123") is called, return "Alice"
      allow(mock_db).to receive(:get).with("user:123").and_return("Alice")

      name = service.get_user_name("123")
      expect(name).to eq("Alice")
    end

    it 'returns Unknown when not found' do
      allow(mock_db).to receive(:get).with("user:456").and_return(nil)

      name = service.get_user_name("456")
      expect(name).to eq("Unknown")
    end
  end

  describe '#save_user' do
    it 'calls the database' do
      # Expectation: set MUST be called with these arguments
      expect(mock_db).to receive(:set).with("user:123", "Alice").and_return(true)

      result = service.save_user("123", "Alice")
      expect(result).to be true
    end

    it 'returns false on db failure' do
      allow(mock_db).to receive(:set).and_return(false)

      result = service.save_user("123", "Alice")
      expect(result).to be false
    end
  end
end
```

**Common RSpec mock patterns:**

```ruby
# allow (stub) — set up return values without asserting calls
allow(mock).to receive(:method).and_return(value)
allow(mock).to receive(:method).with(arg1, arg2).and_return(value)
allow(mock).to receive(:method).and_raise(RuntimeError, "fail")
allow(mock).to receive(:method).and_yield(block_arg)

# expect (mock) — assert that a method IS called
expect(mock).to receive(:method).once
expect(mock).to receive(:method).twice
expect(mock).to receive(:method).exactly(3).times
expect(mock).to receive(:method).at_least(:once)
expect(mock).to receive(:method).at_most(5).times
expect(mock).to receive(:method).with(anything)
expect(mock).to receive(:method).with(hash_including(key: "value"))
expect(mock).to receive(:method).with(instance_of(String))

# Argument matchers
expect(mock).to receive(:method).with(anything)                    # any single arg
expect(mock).to receive(:method).with(any_args)                    # any number of args
expect(mock).to receive(:method).with(no_args)                     # no arguments
expect(mock).to receive(:method).with(hash_including(name: "Alice")) # hash subset
expect(mock).to receive(:method).with(instance_of(String))         # type check
expect(mock).to receive(:method).with(/pattern/)                   # regex match

# Return values
allow(mock).to receive(:method).and_return(1, 2, 3)  # returns 1, then 2, then 3
allow(mock).to receive(:method) { |arg| arg.upcase }  # computed return value

# Ordered expectations
expect(mock).to receive(:connect).ordered
expect(mock).to receive(:query).ordered
expect(mock).to receive(:disconnect).ordered
# connect must be called before query, query before disconnect

# Spy — verify after the fact
allow(mock).to receive(:method)
# ... do stuff ...
expect(mock).to have_received(:method).with("arg")
```

**Full mocking example — testing with dependency injection:**

```ruby
# Interfaces (duck types in Ruby)
class OrderService
  def initialize(repo:, email_service:)
    @repo = repo
    @email_service = email_service
  end

  def place_order(order)
    @repo.save(order)
    @email_service.send_email(
      to: order[:customer_email],
      subject: "Order Confirmed",
      body: "Your order ##{order[:id]} has been placed."
    )
  end
end

RSpec.describe OrderService do
  let(:mock_repo) { instance_double("OrderRepository") }
  let(:mock_email) { instance_double("EmailService") }
  let(:service) { OrderService.new(repo: mock_repo, email_service: mock_email) }
  let(:order) { { id: 1, customer_email: "alice@example.com", total: 99.99 } }

  describe '#place_order' do
    it 'saves the order and sends confirmation email' do
      expect(mock_repo).to receive(:save).with(order).once
      expect(mock_email).to receive(:send_email).with(
        to: "alice@example.com",
        subject: "Order Confirmed",
        body: "Your order #1 has been placed."
      ).once

      service.place_order(order)
    end

    it 'does not send email if save fails' do
      allow(mock_repo).to receive(:save).and_raise(RuntimeError, "DB connection lost")
      expect(mock_email).not_to receive(:send_email)

      expect { service.place_order(order) }.to raise_error(RuntimeError)
    end
  end
end
```

---

### Test-Driven Development (TDD) Basics

**TDD** is a development practice where you write tests before writing the implementation. The cycle is:

```
1. RED    — Write a failing test for the next piece of functionality
2. GREEN  — Write the minimum code to make the test pass
3. REFACTOR — Clean up the code while keeping tests green
4. Repeat
```

**TDD example — building a Stack (RSpec):**

```ruby
# Step 1: RED — Write a failing test
RSpec.describe Stack do
  describe 'new stack' do
    it 'is empty' do
      stack = Stack.new
      expect(stack).to be_empty
      expect(stack.size).to eq(0)
    end
  end
end
# This fails — Stack doesn't exist yet

# Step 2: GREEN — Write minimum code to pass
class Stack
  def empty?
    true
  end

  def size
    0
  end
end
# Test passes!

# Step 3: RED — Next test
RSpec.describe Stack do
  it 'is not empty after push' do
    stack = Stack.new
    stack.push(42)
    expect(stack).not_to be_empty
    expect(stack.size).to eq(1)
  end
end
# Fails — push doesn't exist, empty? always returns true

# Step 4: GREEN — Implement push
class Stack
  def initialize
    @data = []
  end

  def push(value)
    @data.push(value)
  end

  def empty?
    @data.empty?
  end

  def size
    @data.size
  end
end
# Test passes!

# Step 5: RED — Test pop
RSpec.describe Stack do
  it 'pops the last pushed element' do
    stack = Stack.new
    stack.push(10)
    stack.push(20)
    expect(stack.pop).to eq(20)
    expect(stack.pop).to eq(10)
    expect(stack).to be_empty
  end
end

# Step 6: GREEN — Implement pop
class Stack
  def pop
    @data.pop
  end
end

# Step 7: RED — Test edge case
RSpec.describe Stack do
  it 'raises on pop when empty' do
    stack = Stack.new
    expect { stack.pop }.to raise_error(RuntimeError, "Stack is empty")
  end
end

# Step 8: GREEN — Handle edge case
class Stack
  def pop
    raise "Stack is empty" if @data.empty?
    @data.pop
  end
end

# Step 9: REFACTOR — Clean up, add peek, etc.
class Stack
  def initialize
    @data = []
  end

  def push(value)
    @data.push(value)
    self  # enable chaining
  end

  def pop
    raise "Stack is empty" if empty?
    @data.pop
  end

  def peek
    raise "Stack is empty" if empty?
    @data.last
  end

  def empty?
    @data.empty?
  end

  def size
    @data.size
  end
end
```

**TDD benefits:**
- Forces you to think about the interface before the implementation
- Produces code that is testable by design
- Catches bugs immediately
- Tests serve as documentation
- Gives confidence to refactor

---

### Writing Testable Code (Dependency Injection)

Code is testable when you can replace its dependencies with mocks. The key technique is **Dependency Injection (DI)** — pass dependencies in rather than creating them internally.

```ruby
# BAD: Hard to test — creates its own dependencies
class OrderServiceBad
  def initialize
    @db = MySQLDatabase.new("localhost", 3306)  # concrete dependency — can't mock
    @email = SmtpEmailSender.new("smtp.example.com")  # concrete dependency
  end

  def place_order(order)
    @db.save(order)                    # hits real database
    @email.send(order[:customer_email],  # sends real email
                "Order confirmed")
  end
end
# To test this, you need a real MySQL database and SMTP server!

# GOOD: Testable — dependencies are injected
class OrderService
  def initialize(repo:, email_service:)
    @repo = repo
    @email_service = email_service
  end

  def place_order(order)
    @repo.save(order)
    @email_service.send_email(
      to: order[:customer_email],
      subject: "Order Confirmed",
      body: "Your order ##{order[:id]} has been placed."
    )
  end
end

# Now we can test with mocks!
RSpec.describe OrderService do
  let(:mock_repo) { instance_double("OrderRepository") }
  let(:mock_email) { instance_double("EmailService") }
  let(:service) { OrderService.new(repo: mock_repo, email_service: mock_email) }

  it 'saves the order and sends email' do
    order = { id: 1, customer_email: "alice@example.com", total: 99.99 }

    expect(mock_repo).to receive(:save).with(order)
    expect(mock_email).to receive(:send_email).with(
      to: "alice@example.com",
      subject: anything,
      body: anything
    )

    service.place_order(order)
  end

  it 'does not send email if save fails' do
    order = { id: 1, customer_email: "alice@example.com", total: 99.99 }

    allow(mock_repo).to receive(:save).and_raise(RuntimeError, "DB connection lost")
    expect(mock_email).not_to receive(:send_email)

    expect { service.place_order(order) }.to raise_error(RuntimeError)
  end
end
```

**Principles for testable code:**

| Principle | Description | Example |
|-----------|-------------|---------|
| **Depend on duck types** | Use interfaces (duck typing), not concrete classes | Inject `repo:` not `MySQLDatabase.new` |
| **Inject dependencies** | Pass dependencies in, don't create them | Constructor / keyword argument injection |
| **Single Responsibility** | Each class does one thing | Easier to test in isolation |
| **Pure methods** | Same input → same output, no side effects | Easiest to test |
| **Avoid global state** | No singletons in business logic | Pass logger as dependency |
| **Small methods** | Each method does one thing | Easier to write focused tests |
| **Use blocks for resources** | Block-based cleanup patterns | `File.open { |f| ... }` |

---


## Summary & Key Takeaways

| Topic | Core Concept | Key Tools / Patterns |
|-------|-------------|---------------------|
| Exceptions | Raise on failure, rescue to handle | `begin/rescue/ensure/raise`, exception hierarchy |
| Custom Exceptions | Domain-specific error types with context | Inherit from `StandardError` |
| Exception Safety | Guarantees about state after an exception | ensure blocks, block-based RAII, clone-and-swap |
| Resource Cleanup | Resources tied to block lifetime | `File.open { }`, `Mutex#synchronize`, ensure |
| Return Values vs Exceptions | Choose based on expected vs exceptional | `nil` for "not found", exceptions for failures |
| Result Objects | Type-safe "value or error" | Custom `Result` class, `&.` safe navigation |
| Logging Levels | Control verbosity | TRACE, DEBUG, INFO, WARN, ERROR, FATAL |
| Logger Design | Singleton + Strategy (sinks) | Ruby's `Logger`, custom sinks |
| Structured Logging | Machine-parseable log records | JSON format, key-value fields |
| Minitest | Built-in test framework | `assert_*`, `refute_*`, `Minitest::Mock` |
| RSpec | BDD-style test framework | `expect`, `describe`, `it`, `let`, `before` |
| Test Fixtures | Shared setup/teardown for tests | `setup`/`teardown` (Minitest), `before`/`after` (RSpec) |
| Mocking | Mock dependencies for isolation | `instance_double`, `allow`, `expect().to receive` |
| TDD | Write tests first, then implement | Red → Green → Refactor cycle |
| Testable Code | Dependency injection, duck typing | Constructor injection, keyword arguments |

---

## Interview Tips for Module 11

1. **"How does Ruby handle errors?"** Ruby uses exceptions with `begin/rescue/ensure/raise`. The exception hierarchy starts at `Exception`, but you almost always rescue from `StandardError` or its subclasses. Never rescue `Exception` broadly — it catches signals and system exits. Use `ensure` for cleanup (like C++ destructors or Java's finally). Ruby also supports `retry` to re-execute the begin block.

2. **"What is the Ruby exception hierarchy?"** `Exception` is the root. `StandardError` is the base for most application errors — `RuntimeError`, `ArgumentError`, `TypeError`, `NameError`, `IOError`, `IndexError`, etc. `ScriptError`, `SignalException`, and `SystemExit` are siblings of `StandardError` under `Exception`. Bare `rescue` catches `StandardError`. Always rescue specific classes, not `Exception`.

3. **"When would you use exceptions vs return values in Ruby?"** Exceptions for rare, unexpected failures (network down, file corruption, permission denied). Return `nil` or symbols for expected conditions (user not found, invalid input, empty result). Ruby convention: `find` returns nil, `find!` raises. Use both in the same codebase — exceptions for infrastructure failures, nil/symbols for business logic.

4. **"How would you design a logging system?"** Singleton Logger with configurable log level and multiple sinks (Strategy pattern). Sinks: console, file, rotating file, network. Each sink can have its own minimum level (FilteredSink decorator). Use structured logging (JSON) for production — enables searching and alerting. Include timestamp, level, source location, and correlation ID in every log entry. Ruby's stdlib `Logger` handles basic needs; custom implementations add multi-sink support.

5. **"What is ensure and how does it relate to RAII?"** `ensure` is Ruby's equivalent of C++ destructors / Java's finally — code that always runs, whether an exception occurred or not. Ruby's idiomatic RAII is block-based: `File.open { |f| ... }` auto-closes the file, `Mutex#synchronize { ... }` auto-unlocks. Write custom RAII wrappers using class methods that yield to a block and clean up in ensure.

6. **"What is the difference between Minitest and RSpec?"** Minitest is Ruby's built-in framework — simple, fast, uses `assert_*` methods. RSpec is a community framework with BDD-style syntax — `describe`, `it`, `expect().to`. RSpec has richer matchers, shared examples, and built-in mocking. Minitest is lighter and faster. Both are production-quality. Choose based on team preference.

7. **"How do you test code that depends on a database?"** Use Dependency Injection: pass the database as a constructor argument (duck typing). In tests, create a mock/double that returns controlled values. RSpec: `instance_double("Database")` with `allow/expect` for stubs and expectations. Minitest: `Minitest::Mock` with `expect`. This tests your logic without needing a real database.

8. **"What is Test-Driven Development?"** Write a failing test first (Red), write minimum code to pass (Green), then refactor while keeping tests green. Benefits: forces testable design, catches bugs immediately, tests serve as documentation, gives confidence to refactor. The key discipline is writing the test before the implementation.

9. **"How do you make Ruby code testable?"** (a) Depend on duck types, not concrete classes. (b) Inject dependencies through constructor or keyword arguments. (c) Follow Single Responsibility — each class does one thing. (d) Avoid global state and singletons in business logic. (e) Prefer pure methods where possible. (f) Keep methods small and focused. (g) Use block-based patterns for resource management.

10. **"What is a custom exception and when would you create one?"** Create custom exceptions when you need domain-specific error types with additional context (error codes, field names, resource IDs). Inherit from `StandardError` (not `Exception`). Build a hierarchy: `AppError` → `AuthenticationError`, `NotFoundError`, `ValidationError`. Add `attr_reader` for extra context fields. This enables rescuing specific error types at different layers.

11. **"What is structured logging?"** Producing log records as machine-parseable data (JSON, key-value pairs) instead of free-form text. Each log entry has typed fields (user_id, duration_ms, status_code) that can be searched, filtered, and aggregated by log analysis tools. Benefits: no regex needed to parse logs, enables dashboards and alerts, supports correlation across services using request IDs.

12. **"How would you handle errors in a multi-layered Ruby system?"** Each layer rescues exceptions it can handle and re-raises or wraps for the layer above. Repository layer raises `DatabaseError`. Service layer rescues it and raises `ServiceError` with business context. Controller/API layer rescues `ServiceError` and returns appropriate HTTP status codes. Use ensure at every layer to prevent resource leaks. Use the bang method convention (`save!` raises, `save` returns boolean) to give callers a choice.