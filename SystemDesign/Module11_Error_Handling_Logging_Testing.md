# Module 11: Error Handling, Logging & Testing

> Robust software doesn't just work when everything goes right — it handles failures gracefully, logs what's happening for debugging and monitoring, and has tests that prove correctness. These three pillars — error handling, logging, and testing — are what separate production-quality code from prototype code. In LLD interviews, you're expected to handle edge cases, design observable systems, and write testable code. This module covers all three in depth with C++ implementations.

---

## 11.1 Error Handling in C++

> Error handling is about anticipating what can go wrong and deciding how your program should respond. C++ offers two primary mechanisms: **exceptions** (for exceptional, unexpected failures) and **error codes** (for expected, recoverable conditions). Choosing the right approach — and implementing it correctly — is critical for building reliable systems.

---

### Exceptions (try, catch, throw)

Exceptions are C++'s primary mechanism for reporting and handling errors. When an error occurs, you **throw** an exception object. The runtime unwinds the call stack until it finds a matching **catch** block.

```cpp
#include <iostream>
#include <stdexcept>
#include <string>
using namespace std;

// Throwing an exception
double divide(double a, double b) {
    if (b == 0.0) {
        throw runtime_error("Division by zero");
        // Execution STOPS here — control transfers to the nearest catch block
        // Code after throw is never executed
    }
    return a / b;
}

// Catching an exception
int main() {
    try {
        double result = divide(10.0, 0.0);
        cout << "Result: " << result << endl;  // never reached
    }
    catch (const runtime_error& e) {
        // Catches runtime_error and its derived classes
        cerr << "Error: " << e.what() << endl;  // "Error: Division by zero"
    }

    cout << "Program continues after handling the error" << endl;
    return 0;
}
```

**How exception propagation works:**

```cpp
void function_c() {
    throw runtime_error("Something went wrong in C");
    // Stack unwinding begins here
}

void function_b() {
    cout << "B: before calling C" << endl;
    function_c();  // exception thrown inside
    cout << "B: after calling C" << endl;  // NEVER executed — stack is unwinding
}

void function_a() {
    try {
        cout << "A: before calling B" << endl;
        function_b();
        cout << "A: after calling B" << endl;  // NEVER executed
    }
    catch (const runtime_error& e) {
        // Exception caught here — stack unwinding stops
        cerr << "A caught: " << e.what() << endl;
    }
    cout << "A: continues after catch" << endl;  // executed normally
}

// Output:
// A: before calling B
// B: before calling C
// A caught: Something went wrong in C
// A: continues after catch
```

**Multiple catch blocks — catching different exception types:**

```cpp
void process_input(const string& input) {
    try {
        if (input.empty()) {
            throw invalid_argument("Input cannot be empty");
        }

        int value = stoi(input);  // may throw invalid_argument or out_of_range

        if (value < 0) {
            throw out_of_range("Value must be non-negative");
        }

        cout << "Processing value: " << value << endl;
    }
    catch (const out_of_range& e) {
        // Catches out_of_range specifically
        cerr << "Out of range: " << e.what() << endl;
    }
    catch (const invalid_argument& e) {
        // Catches invalid_argument
        cerr << "Invalid argument: " << e.what() << endl;
    }
    catch (const exception& e) {
        // Catches any std::exception not caught above
        cerr << "General error: " << e.what() << endl;
    }
    catch (...) {
        // Catches ANYTHING (including non-exception types like int, string)
        cerr << "Unknown error occurred" << endl;
    }
}
```

**Important rules:**
- Catch blocks are checked **top to bottom** — put more specific types first
- Catch by **const reference** (`const exception& e`) to avoid slicing and unnecessary copies
- `catch (...)` catches everything — use as a last resort
- An uncaught exception calls `std::terminate()` — your program crashes

**Re-throwing exceptions:**

```cpp
void middle_layer() {
    try {
        // ... some operation that might throw ...
        throw runtime_error("database connection failed");
    }
    catch (const exception& e) {
        // Log the error at this level
        cerr << "middle_layer caught: " << e.what() << endl;

        // Re-throw the SAME exception (preserves the original type)
        throw;  // NOT throw e; — that would slice to std::exception

        // Or wrap in a higher-level exception
        // throw runtime_error(string("Service error: ") + e.what());
    }
}

void top_layer() {
    try {
        middle_layer();
    }
    catch (const runtime_error& e) {
        cerr << "top_layer caught: " << e.what() << endl;
    }
}
```

**throw vs throw e:**

```cpp
try {
    throw runtime_error("original error");
}
catch (const exception& e) {
    throw;    // GOOD: re-throws the original runtime_error object
    // throw e; // BAD: throws a COPY of e, sliced to std::exception
}
```

---

### Exception Hierarchy (std::exception, std::runtime_error, etc.)

C++ provides a hierarchy of standard exception classes, all derived from `std::exception`.

```
std::exception                          ← base class, has what()
├── std::logic_error                    ← programmer errors (bugs)
│   ├── std::invalid_argument           ← bad function argument
│   ├── std::domain_error               ← math domain error
│   ├── std::length_error               ← exceeded max size
│   ├── std::out_of_range               ← index out of bounds
│   └── std::future_error               ← future/promise errors
├── std::runtime_error                  ← errors detectable only at runtime
│   ├── std::overflow_error             ← arithmetic overflow
│   ├── std::underflow_error            ← arithmetic underflow
│   ├── std::range_error                ← range error
│   └── std::system_error               ← OS-level errors
│       └── std::filesystem::filesystem_error
├── std::bad_alloc                      ← memory allocation failure (new)
├── std::bad_cast                       ← failed dynamic_cast
├── std::bad_typeid                     ← typeid on null pointer
└── std::bad_function_call              ← calling empty std::function
```

**When to use which:**

```cpp
#include <stdexcept>
using namespace std;

// logic_error — indicates a bug in the program (should be fixed, not caught)
void set_age(int age) {
    if (age < 0 || age > 150) {
        throw invalid_argument("Age must be between 0 and 150");
    }
}

// runtime_error — indicates a failure that can't be predicted at compile time
string read_file(const string& path) {
    ifstream file(path);
    if (!file.is_open()) {
        throw runtime_error("Failed to open file: " + path);
    }
    // ...
}

// out_of_range — index or key not found
int get_element(const vector<int>& v, int index) {
    if (index < 0 || index >= (int)v.size()) {
        throw out_of_range("Index " + to_string(index) + " out of range [0, "
                          + to_string(v.size() - 1) + "]");
    }
    return v[index];
}
```

**The what() method:**

Every `std::exception` has a `what()` method that returns a C-string describing the error.

```cpp
try {
    throw runtime_error("connection timed out");
}
catch (const exception& e) {
    cout << e.what() << endl;  // "connection timed out"
}
```

---

### Custom Exception Classes

For real-world systems, you'll want domain-specific exception classes that carry additional context beyond a simple message.

```cpp
#include <stdexcept>
#include <string>
using namespace std;

// Simple custom exception
class DatabaseException : public runtime_error {
public:
    DatabaseException(const string& msg) : runtime_error(msg) {}
};

class ConnectionException : public DatabaseException {
    string host;
    int port;
public:
    ConnectionException(const string& host, int port, const string& msg)
        : DatabaseException("Connection to " + host + ":" + to_string(port) + " failed: " + msg),
          host(host), port(port) {}

    const string& get_host() const { return host; }
    int get_port() const { return port; }
};

class QueryException : public DatabaseException {
    string query;
    int error_code;
public:
    QueryException(const string& query, int code, const string& msg)
        : DatabaseException("Query failed (code " + to_string(code) + "): " + msg),
          query(query), error_code(code) {}

    const string& get_query() const { return query; }
    int get_error_code() const { return error_code; }
};

// Usage
void execute_query(const string& sql) {
    // Simulate a query failure
    throw QueryException(sql, 1045, "Access denied for user 'root'");
}

void connect_to_db(const string& host, int port) {
    throw ConnectionException(host, port, "Connection refused");
}

int main() {
    try {
        connect_to_db("localhost", 3306);
    }
    catch (const ConnectionException& e) {
        cerr << e.what() << endl;
        cerr << "Host: " << e.get_host() << ", Port: " << e.get_port() << endl;
        // Retry logic, fallback to replica, etc.
    }
    catch (const DatabaseException& e) {
        cerr << "Database error: " << e.what() << endl;
    }
}
```

**Exception hierarchy for an LLD system:**

```cpp
// Base exception for the entire application
class AppException : public runtime_error {
    int error_code;
    string context;
public:
    AppException(int code, const string& ctx, const string& msg)
        : runtime_error(msg), error_code(code), context(ctx) {}

    int code() const { return error_code; }
    const string& get_context() const { return context; }
};

// Domain-specific exceptions
class AuthenticationException : public AppException {
public:
    AuthenticationException(const string& msg)
        : AppException(401, "auth", msg) {}
};

class AuthorizationException : public AppException {
public:
    AuthorizationException(const string& msg)
        : AppException(403, "auth", msg) {}
};

class NotFoundException : public AppException {
    string resource_type;
    string resource_id;
public:
    NotFoundException(const string& type, const string& id)
        : AppException(404, "resource",
                       type + " with ID '" + id + "' not found"),
          resource_type(type), resource_id(id) {}

    const string& get_resource_type() const { return resource_type; }
    const string& get_resource_id() const { return resource_id; }
};

class ValidationException : public AppException {
    string field;
public:
    ValidationException(const string& field, const string& msg)
        : AppException(400, "validation", "Validation failed for '" + field + "': " + msg),
          field(field) {}

    const string& get_field() const { return field; }
};

// Usage in a service layer
class UserService {
public:
    User get_user(const string& id) {
        auto user = db.find_user(id);
        if (!user) {
            throw NotFoundException("User", id);
        }
        return *user;
    }

    void create_user(const string& name, const string& email) {
        if (name.empty()) {
            throw ValidationException("name", "cannot be empty");
        }
        if (email.find('@') == string::npos) {
            throw ValidationException("email", "invalid format");
        }
        // ...
    }
};
```

---

### Exception Safety Guarantees (Basic, Strong, No-throw)

Exception safety defines what guarantees a function provides if an exception is thrown during its execution. This is critical for writing correct, maintainable code.

**The three levels of exception safety:**

| Guarantee | Promise | Example |
|-----------|---------|---------|
| **No-throw** | Function never throws | Destructors, `swap()`, simple getters |
| **Strong** | If exception thrown, state is unchanged (rollback) | `push_back` (if copy throws, vector unchanged) |
| **Basic** | If exception thrown, no resource leaks, object in valid (but unspecified) state | Most standard library operations |
| **No guarantee** | Anything can happen — resource leaks, corrupted state | Poorly written code |

```cpp
#include <vector>
#include <string>
using namespace std;

// NO-THROW guarantee — never throws
class Account {
    double balance;
public:
    // Getters should never throw
    double get_balance() const noexcept { return balance; }

    // Swap should never throw (used for strong guarantee)
    void swap(Account& other) noexcept {
        std::swap(balance, other.balance);
    }
};

// STRONG guarantee — all or nothing (commit/rollback)
class TransactionManager {
    vector<string> log;

public:
    // If this throws, the log is unchanged
    void add_entry(const string& entry) {
        // vector::push_back provides strong guarantee:
        // if reallocation fails (bad_alloc), the vector is unchanged
        log.push_back(entry);
    }

    // Strong guarantee using copy-and-swap idiom
    void replace_log(const vector<string>& new_log) {
        vector<string> temp = new_log;  // copy — might throw (strong: original unchanged)
        swap(log, temp);                // swap — noexcept (commit)
        // If copy throws, log is untouched
        // If swap succeeds, log has new data
    }
};

// BASIC guarantee — no leaks, valid state, but state may have changed
class UserList {
    vector<string> users;

public:
    // Basic guarantee: if exception thrown mid-way, some users may be added
    // but no resource leaks occur
    void add_users(const vector<string>& new_users) {
        for (const auto& user : new_users) {
            users.push_back(user);  // might throw on any iteration
            // If it throws on the 3rd user, the first 2 are already added
            // Object is valid but in a partially-updated state
        }
    }
};
```

**Copy-and-swap idiom for strong guarantee:**

```cpp
class Database {
    vector<Record> records;
    unordered_map<int, size_t> index;

public:
    // Strong guarantee: either fully succeeds or state is unchanged
    void bulk_insert(const vector<Record>& new_records) {
        // Step 1: Make copies (might throw — original state untouched)
        vector<Record> new_records_copy = records;
        unordered_map<int, size_t> new_index = index;

        // Step 2: Modify the copies (might throw — original state untouched)
        for (const auto& record : new_records) {
            new_index[record.id] = new_records_copy.size();
            new_records_copy.push_back(record);
        }

        // Step 3: Commit — swap is noexcept
        swap(records, new_records_copy);   // noexcept
        swap(index, new_index);            // noexcept
        // If we reach here, the operation fully succeeded
    }
};
```

---

### RAII for Exception Safety

**RAII (Resource Acquisition Is Initialization)** is the most important C++ idiom for exception safety. Resources are acquired in constructors and released in destructors. Since destructors are called during stack unwinding, resources are always cleaned up — even when exceptions are thrown.

```cpp
#include <fstream>
#include <mutex>
#include <memory>
using namespace std;

// BAD: Manual resource management — not exception-safe
void bad_example() {
    int* data = new int[1000];
    FILE* file = fopen("data.txt", "r");

    // If this throws, data and file are LEAKED
    process_data(data, file);

    fclose(file);   // never reached if exception thrown
    delete[] data;  // never reached if exception thrown
}

// GOOD: RAII — exception-safe
void good_example() {
    auto data = make_unique<int[]>(1000);  // unique_ptr — auto-deleted
    ifstream file("data.txt");              // ifstream — auto-closed

    // Even if this throws, data is deleted and file is closed
    // because their destructors run during stack unwinding
    process_data(data.get(), file);

    // No manual cleanup needed
}

// RAII for mutex locking
void thread_safe_operation() {
    static mutex mtx;

    // BAD: manual lock/unlock
    // mtx.lock();
    // do_work();    // if this throws, mutex is NEVER unlocked → deadlock
    // mtx.unlock();

    // GOOD: RAII lock guard
    lock_guard<mutex> lock(mtx);
    do_work();  // if this throws, lock_guard destructor unlocks the mutex
}

// Custom RAII wrapper
class FileHandle {
    FILE* file;
public:
    FileHandle(const string& path, const string& mode)
        : file(fopen(path.c_str(), mode.c_str())) {
        if (!file) throw runtime_error("Failed to open: " + path);
    }

    ~FileHandle() {
        if (file) fclose(file);  // always closed, even on exception
    }

    FILE* get() { return file; }

    // Non-copyable, movable
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept : file(other.file) { other.file = nullptr; }
};

// RAII for database transactions
class Transaction {
    Database& db;
    bool committed = false;

public:
    Transaction(Database& db) : db(db) {
        db.begin_transaction();
    }

    ~Transaction() {
        if (!committed) {
            db.rollback();  // auto-rollback if not committed
        }
    }

    void commit() {
        db.commit_transaction();
        committed = true;
    }
};

void transfer_money(Database& db, int from, int to, double amount) {
    Transaction txn(db);  // begins transaction

    db.debit(from, amount);   // might throw
    db.credit(to, amount);    // might throw

    txn.commit();  // only reached if both succeed
    // If any exception is thrown, ~Transaction() calls rollback()
}
```

**RAII summary:**
- Acquire resources in constructors
- Release resources in destructors
- Destructors are called during stack unwinding (exception propagation)
- Use smart pointers (`unique_ptr`, `shared_ptr`) for heap memory
- Use lock guards (`lock_guard`, `unique_lock`) for mutexes
- Use RAII wrappers for file handles, database connections, transactions

---

### noexcept Specifier

The `noexcept` specifier tells the compiler (and other programmers) that a function will never throw an exception. If a `noexcept` function does throw, `std::terminate()` is called immediately — no stack unwinding.

```cpp
// Declaring a function as noexcept
void safe_function() noexcept {
    // This function promises to never throw
    // If it does throw, std::terminate() is called
}

// Conditional noexcept
template <typename T>
void swap_values(T& a, T& b) noexcept(noexcept(T(std::move(a)))) {
    // noexcept if T's move constructor is noexcept
    T temp = std::move(a);
    a = std::move(b);
    b = std::move(temp);
}

// Checking if a function is noexcept
static_assert(noexcept(safe_function()), "safe_function should be noexcept");
```

**When to use noexcept:**

```cpp
class MyClass {
public:
    // ALWAYS noexcept: destructors (implicitly noexcept in C++11+)
    ~MyClass() noexcept {
        // cleanup — must never throw
    }

    // ALWAYS noexcept: move operations (enables optimizations)
    MyClass(MyClass&& other) noexcept
        : data(std::move(other.data)) {}

    MyClass& operator=(MyClass&& other) noexcept {
        data = std::move(other.data);
        return *this;
    }

    // ALWAYS noexcept: swap
    void swap(MyClass& other) noexcept {
        std::swap(data, other.data);
    }

    // SHOULD be noexcept: simple getters
    int get_value() const noexcept { return value; }
    bool is_valid() const noexcept { return valid; }

    // Should NOT be noexcept: functions that allocate or do I/O
    void load_data(const string& path) {
        // might throw bad_alloc, runtime_error, etc.
    }

private:
    vector<int> data;
    int value = 0;
    bool valid = true;
};
```

**Why noexcept matters for performance:**

```cpp
// std::vector uses move operations when resizing IF they are noexcept
// If move constructor is NOT noexcept, vector falls back to COPYING (slower but safe)

class FastMove {
public:
    FastMove(FastMove&&) noexcept = default;  // vector will MOVE — fast
};

class SlowMove {
public:
    SlowMove(SlowMove&&) {}  // NOT noexcept — vector will COPY — slow
};

// vector<FastMove> resizing: moves elements — O(n) with small constant
// vector<SlowMove> resizing: copies elements — O(n) with large constant
```

---

### Error Codes vs Exceptions (When to Use Which)

This is a design decision that affects your entire codebase. Both approaches have trade-offs.

**Error codes — return a status indicating success or failure:**

```cpp
#include <system_error>
using namespace std;

// C-style error codes
enum class ErrorCode {
    Success = 0,
    NotFound,
    PermissionDenied,
    InvalidInput,
    ConnectionFailed,
    Timeout
};

struct Result {
    ErrorCode error;
    string data;

    bool ok() const { return error == ErrorCode::Success; }
};

Result find_user(const string& id) {
    if (id.empty()) {
        return {ErrorCode::InvalidInput, ""};
    }
    // ... lookup ...
    if (!found) {
        return {ErrorCode::NotFound, ""};
    }
    return {ErrorCode::Success, user_data};
}

void caller() {
    auto result = find_user("123");
    if (!result.ok()) {
        // Handle error
        switch (result.error) {
            case ErrorCode::NotFound:
                cout << "User not found" << endl;
                break;
            case ErrorCode::InvalidInput:
                cout << "Invalid input" << endl;
                break;
            default:
                cout << "Unknown error" << endl;
        }
        return;
    }
    // Use result.data
}
```

**When to use exceptions vs error codes:**

| Criteria | Exceptions | Error Codes |
|----------|-----------|-------------|
| **Frequency** | Rare, unexpected failures | Common, expected conditions |
| **Examples** | Out of memory, file corruption, network down | User not found, invalid input, empty result |
| **Performance** | Zero cost when not thrown; expensive when thrown | Small constant cost on every call |
| **Propagation** | Automatic (stack unwinding) | Manual (must check at every level) |
| **Ignorability** | Hard to ignore (crashes if uncaught) | Easy to ignore (silent bugs) |
| **Code clarity** | Clean happy path, errors handled separately | Error checks interleaved with logic |
| **Real-time systems** | Avoid (unpredictable timing) | Preferred (predictable) |
| **Constructors** | Only option (constructors can't return values) | N/A |

**Best practice — use both:**

```cpp
// Exceptions for truly exceptional situations
void connect_to_database() {
    // Network failure, authentication failure — exceptional
    throw ConnectionException("host", 3306, "Connection refused");
}

// Error codes / optional for expected "not found" cases
optional<User> find_user(const string& id) {
    auto it = users.find(id);
    if (it == users.end()) return nullopt;  // not found is expected, not exceptional
    return it->second;
}

// Caller
void handle_request(const string& user_id) {
    auto user = find_user(user_id);  // expected to sometimes return nullopt
    if (!user) {
        cout << "User not found" << endl;
        return;
    }
    // Use *user
}
```

---

### std::optional, std::expected (C++23)

**std::optional (C++17)** — represents a value that may or may not be present. Perfect for "not found" scenarios.

```cpp
#include <optional>
#include <string>
using namespace std;

class UserRepository {
    unordered_map<int, string> users;

public:
    // Returns the user if found, or empty optional if not
    optional<string> find_by_id(int id) const {
        auto it = users.find(id);
        if (it != users.end()) {
            return it->second;  // has value
        }
        return nullopt;  // no value
    }

    // Optional with default
    string find_or_default(int id, const string& default_name) const {
        return find_by_id(id).value_or(default_name);
    }
};

void usage() {
    UserRepository repo;

    // Check and use
    auto user = repo.find_by_id(42);
    if (user) {
        cout << "Found: " << *user << endl;       // dereference with *
        cout << "Found: " << user.value() << endl; // or .value() (throws if empty)
    } else {
        cout << "Not found" << endl;
    }

    // value_or — provide a default
    string name = repo.find_by_id(42).value_or("Unknown");

    // has_value() check
    if (user.has_value()) {
        cout << user.value() << endl;
    }
}
```

**std::expected (C++23)** — like optional but carries an error value when the operation fails.

```cpp
// C++23 std::expected — not yet widely available, but here's the concept
// expected<T, E> holds either a T (success) or an E (error)

#include <expected>  // C++23
using namespace std;

enum class ParseError {
    EmptyInput,
    InvalidFormat,
    OutOfRange
};

expected<int, ParseError> parse_int(const string& input) {
    if (input.empty()) {
        return unexpected(ParseError::EmptyInput);
    }

    try {
        int value = stoi(input);
        if (value < 0 || value > 1000) {
            return unexpected(ParseError::OutOfRange);
        }
        return value;  // success
    }
    catch (...) {
        return unexpected(ParseError::InvalidFormat);
    }
}

void usage() {
    auto result = parse_int("42");

    if (result) {
        cout << "Parsed: " << *result << endl;
    } else {
        switch (result.error()) {
            case ParseError::EmptyInput:
                cout << "Empty input" << endl;
                break;
            case ParseError::InvalidFormat:
                cout << "Invalid format" << endl;
                break;
            case ParseError::OutOfRange:
                cout << "Out of range" << endl;
                break;
        }
    }
}
```

**Pre-C++23 alternative — Result type:**

```cpp
// Custom Result type (works in C++17)
template <typename T, typename E>
class Result {
    variant<T, E> data;

public:
    // Success
    static Result ok(const T& value) {
        Result r;
        r.data = value;
        return r;
    }

    // Error
    static Result err(const E& error) {
        Result r;
        r.data = error;
        return r;
    }

    bool is_ok() const { return holds_alternative<T>(data); }
    bool is_err() const { return holds_alternative<E>(data); }

    const T& value() const { return get<T>(data); }
    const E& error() const { return get<E>(data); }

    // Map — transform the success value
    template <typename F>
    auto map(F func) const -> Result<decltype(func(value())), E> {
        if (is_ok()) {
            return Result<decltype(func(value())), E>::ok(func(value()));
        }
        return Result<decltype(func(value())), E>::err(error());
    }
};

// Usage
Result<User, string> find_user(int id) {
    if (id <= 0) return Result<User, string>::err("Invalid ID");
    // ...
    return Result<User, string>::ok(user);
}
```

---


## 11.2 Logging

> Logging is the practice of recording events, errors, and diagnostic information during program execution. Good logging is essential for debugging, monitoring, auditing, and understanding system behavior in production. In LLD, designing a logger is a classic interview problem that combines Singleton, Strategy, Observer, and Chain of Responsibility patterns.

---

### Logging Levels (TRACE, DEBUG, INFO, WARN, ERROR, FATAL)

Logging levels control the verbosity of log output. Each level represents a severity — you configure the minimum level, and only messages at that level or above are recorded.

```cpp
enum class LogLevel {
    TRACE = 0,   // Most verbose — detailed execution flow
    DEBUG = 1,   // Debugging information — variable values, state changes
    INFO  = 2,   // General information — startup, shutdown, config loaded
    WARN  = 3,   // Warning — something unexpected but recoverable
    ERROR = 4,   // Error — operation failed but system continues
    FATAL = 5    // Fatal — system cannot continue, about to crash
};

string level_to_string(LogLevel level) {
    switch (level) {
        case LogLevel::TRACE: return "TRACE";
        case LogLevel::DEBUG: return "DEBUG";
        case LogLevel::INFO:  return "INFO";
        case LogLevel::WARN:  return "WARN";
        case LogLevel::ERROR: return "ERROR";
        case LogLevel::FATAL: return "FATAL";
        default: return "UNKNOWN";
    }
}
```

**When to use each level:**

| Level | Purpose | Example |
|-------|---------|---------|
| TRACE | Extremely detailed flow | "Entering function X with params a=1, b=2" |
| DEBUG | Diagnostic information | "Cache hit for key 'user_123'" |
| INFO | Normal operations | "Server started on port 8080" |
| WARN | Unexpected but handled | "Retry attempt 2/3 for API call" |
| ERROR | Operation failed | "Failed to save user: database timeout" |
| FATAL | System-level failure | "Out of memory — shutting down" |

**Production environments** typically run at INFO or WARN level. DEBUG and TRACE are enabled during development or when investigating issues.

---

### Logger Design (Singleton Logger)

A logger is a classic Singleton — there should be one centralized logging system that all components use.

```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <mutex>
#include <chrono>
#include <iomanip>
#include <sstream>
#include <vector>
#include <memory>
using namespace std;

// Forward declaration
class LogSink;

class Logger {
public:
    static Logger& instance() {
        static Logger logger;
        return logger;
    }

    void set_level(LogLevel level) { min_level = level; }
    LogLevel get_level() const { return min_level; }

    void add_sink(shared_ptr<LogSink> sink) {
        lock_guard<mutex> lock(mtx);
        sinks.push_back(sink);
    }

    void log(LogLevel level, const string& message,
             const string& file = "", int line = 0) {
        if (level < min_level) return;  // filter by level

        string formatted = format_message(level, message, file, line);

        lock_guard<mutex> lock(mtx);
        for (auto& sink : sinks) {
            sink->write(formatted, level);
        }
    }

    // Convenience methods
    void trace(const string& msg, const string& file = "", int line = 0) {
        log(LogLevel::TRACE, msg, file, line);
    }
    void debug(const string& msg, const string& file = "", int line = 0) {
        log(LogLevel::DEBUG, msg, file, line);
    }
    void info(const string& msg, const string& file = "", int line = 0) {
        log(LogLevel::INFO, msg, file, line);
    }
    void warn(const string& msg, const string& file = "", int line = 0) {
        log(LogLevel::WARN, msg, file, line);
    }
    void error(const string& msg, const string& file = "", int line = 0) {
        log(LogLevel::ERROR, msg, file, line);
    }
    void fatal(const string& msg, const string& file = "", int line = 0) {
        log(LogLevel::FATAL, msg, file, line);
    }

private:
    Logger() : min_level(LogLevel::INFO) {}
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;

    string format_message(LogLevel level, const string& message,
                          const string& file, int line) {
        auto now = chrono::system_clock::now();
        auto time = chrono::system_clock::to_time_t(now);
        auto ms = chrono::duration_cast<chrono::milliseconds>(
            now.time_since_epoch()) % 1000;

        ostringstream oss;
        oss << put_time(localtime(&time), "%Y-%m-%d %H:%M:%S")
            << "." << setfill('0') << setw(3) << ms.count()
            << " [" << level_to_string(level) << "]";

        if (!file.empty()) {
            oss << " [" << file << ":" << line << "]";
        }

        oss << " " << message;
        return oss.str();
    }

    LogLevel min_level;
    vector<shared_ptr<LogSink>> sinks;
    mutex mtx;
};

// Macro for automatic file and line info
#define LOG_TRACE(msg) Logger::instance().trace(msg, __FILE__, __LINE__)
#define LOG_DEBUG(msg) Logger::instance().debug(msg, __FILE__, __LINE__)
#define LOG_INFO(msg)  Logger::instance().info(msg, __FILE__, __LINE__)
#define LOG_WARN(msg)  Logger::instance().warn(msg, __FILE__, __LINE__)
#define LOG_ERROR(msg) Logger::instance().error(msg, __FILE__, __LINE__)
#define LOG_FATAL(msg) Logger::instance().fatal(msg, __FILE__, __LINE__)
```

---

### Log Sinks (Multiple Output Targets)

A **sink** (or appender) is a destination for log messages. Using the Strategy pattern, the logger can write to multiple sinks simultaneously.

```cpp
// Abstract sink interface
class LogSink {
public:
    virtual ~LogSink() = default;
    virtual void write(const string& message, LogLevel level) = 0;
};

// Console sink — writes to stdout/stderr
class ConsoleSink : public LogSink {
public:
    void write(const string& message, LogLevel level) override {
        if (level >= LogLevel::ERROR) {
            cerr << message << endl;  // errors to stderr
        } else {
            cout << message << endl;  // everything else to stdout
        }
    }
};

// File sink — writes to a log file
class FileSink : public LogSink {
    ofstream file;
    mutex file_mtx;

public:
    FileSink(const string& path) : file(path, ios::app) {
        if (!file.is_open()) {
            throw runtime_error("Failed to open log file: " + path);
        }
    }

    void write(const string& message, LogLevel level) override {
        lock_guard<mutex> lock(file_mtx);
        file << message << endl;
        file.flush();  // ensure it's written immediately
    }
};

// Rotating file sink — creates new file when size limit is reached
class RotatingFileSink : public LogSink {
    string base_path;
    size_t max_size;
    int max_files;
    ofstream current_file;
    size_t current_size = 0;
    int file_index = 0;
    mutex mtx;

    void rotate() {
        current_file.close();
        file_index = (file_index + 1) % max_files;
        string path = base_path + "." + to_string(file_index);
        current_file.open(path, ios::trunc);
        current_size = 0;
    }

public:
    RotatingFileSink(const string& path, size_t max_bytes = 10 * 1024 * 1024,
                     int max_file_count = 5)
        : base_path(path), max_size(max_bytes), max_files(max_file_count) {
        current_file.open(path, ios::app);
        if (!current_file.is_open()) {
            throw runtime_error("Failed to open log file: " + path);
        }
    }

    void write(const string& message, LogLevel level) override {
        lock_guard<mutex> lock(mtx);
        if (current_size + message.size() > max_size) {
            rotate();
        }
        current_file << message << endl;
        current_size += message.size() + 1;
    }
};

// Filtered sink — only writes messages at or above a certain level
class FilteredSink : public LogSink {
    shared_ptr<LogSink> inner;
    LogLevel min_level;

public:
    FilteredSink(shared_ptr<LogSink> sink, LogLevel level)
        : inner(sink), min_level(level) {}

    void write(const string& message, LogLevel level) override {
        if (level >= min_level) {
            inner->write(message, level);
        }
    }
};

// Setup example
void setup_logging() {
    auto& logger = Logger::instance();
    logger.set_level(LogLevel::DEBUG);

    // Console: show INFO and above
    logger.add_sink(make_shared<FilteredSink>(
        make_shared<ConsoleSink>(), LogLevel::INFO));

    // File: log everything (DEBUG and above)
    logger.add_sink(make_shared<FileSink>("app.log"));

    // Rotating file: errors only, max 10MB per file, keep 5 files
    logger.add_sink(make_shared<FilteredSink>(
        make_shared<RotatingFileSink>("error.log", 10 * 1024 * 1024, 5),
        LogLevel::ERROR));

    LOG_INFO("Logging system initialized");
}
```

---

### Log Formatting and Rotation

**Log format components:**

```
2025-01-15 14:30:45.123 [INFO] [UserService.cpp:42] [req-abc123] User login successful: user_id=456
│                        │      │                    │            │
│                        │      │                    │            └── Message
│                        │      │                    └── Correlation/Request ID
│                        │      └── Source location (file:line)
│                        └── Log level
└── Timestamp with milliseconds
```

**Configurable log formatter:**

```cpp
class LogFormatter {
public:
    virtual ~LogFormatter() = default;
    virtual string format(LogLevel level, const string& message,
                          const string& file, int line,
                          const string& thread_id) = 0;
};

// Simple text formatter
class TextFormatter : public LogFormatter {
public:
    string format(LogLevel level, const string& message,
                  const string& file, int line,
                  const string& thread_id) override {
        ostringstream oss;
        oss << get_timestamp()
            << " [" << level_to_string(level) << "]"
            << " [thread:" << thread_id << "]"
            << " [" << file << ":" << line << "]"
            << " " << message;
        return oss.str();
    }

private:
    string get_timestamp() {
        auto now = chrono::system_clock::now();
        auto time = chrono::system_clock::to_time_t(now);
        auto ms = chrono::duration_cast<chrono::milliseconds>(
            now.time_since_epoch()) % 1000;

        ostringstream oss;
        oss << put_time(localtime(&time), "%Y-%m-%d %H:%M:%S")
            << "." << setfill('0') << setw(3) << ms.count();
        return oss.str();
    }
};
```

**Log rotation strategies:**

| Strategy | How It Works | Use Case |
|----------|-------------|----------|
| Size-based | Rotate when file exceeds N bytes | General purpose |
| Time-based | Rotate daily/hourly | Compliance, archival |
| Count-based | Keep at most N log files | Disk space management |
| Compress on rotate | gzip old log files | Long-term storage |

---

### Structured Logging

Traditional logging produces human-readable text. **Structured logging** produces machine-parseable records (JSON, key-value pairs) that can be easily searched, filtered, and analyzed by log aggregation tools (ELK stack, Splunk, Datadog).

```cpp
#include <sstream>
#include <unordered_map>
using namespace std;

// Structured log entry
class LogEntry {
    LogLevel level;
    string message;
    unordered_map<string, string> fields;
    chrono::system_clock::time_point timestamp;

public:
    LogEntry(LogLevel lvl, const string& msg)
        : level(lvl), message(msg), timestamp(chrono::system_clock::now()) {}

    // Fluent API for adding fields
    LogEntry& with(const string& key, const string& value) {
        fields[key] = value;
        return *this;
    }

    LogEntry& with(const string& key, int value) {
        fields[key] = to_string(value);
        return *this;
    }

    LogEntry& with(const string& key, double value) {
        fields[key] = to_string(value);
        return *this;
    }

    // Format as JSON
    string to_json() const {
        ostringstream oss;
        oss << "{";
        oss << "\"timestamp\":\"" << get_timestamp() << "\"";
        oss << ",\"level\":\"" << level_to_string(level) << "\"";
        oss << ",\"message\":\"" << escape_json(message) << "\"";

        for (const auto& [key, value] : fields) {
            oss << ",\"" << key << "\":\"" << escape_json(value) << "\"";
        }

        oss << "}";
        return oss.str();
    }

    // Format as key-value pairs
    string to_kv() const {
        ostringstream oss;
        oss << "timestamp=" << get_timestamp()
            << " level=" << level_to_string(level)
            << " msg=\"" << message << "\"";

        for (const auto& [key, value] : fields) {
            oss << " " << key << "=\"" << value << "\"";
        }
        return oss.str();
    }

    LogLevel get_level() const { return level; }

private:
    string get_timestamp() const {
        auto time = chrono::system_clock::to_time_t(timestamp);
        ostringstream oss;
        oss << put_time(gmtime(&time), "%Y-%m-%dT%H:%M:%SZ");
        return oss.str();
    }

    static string escape_json(const string& s) {
        string result;
        for (char c : s) {
            switch (c) {
                case '"':  result += "\\\""; break;
                case '\\': result += "\\\\"; break;
                case '\n': result += "\\n"; break;
                case '\t': result += "\\t"; break;
                default:   result += c;
            }
        }
        return result;
    }
};

// Usage
void handle_request(const string& user_id, const string& endpoint) {
    auto start = chrono::steady_clock::now();

    // ... process request ...

    auto elapsed = chrono::duration_cast<chrono::milliseconds>(
        chrono::steady_clock::now() - start);

    LogEntry entry(LogLevel::INFO, "Request processed");
    entry.with("user_id", user_id)
         .with("endpoint", endpoint)
         .with("duration_ms", (int)elapsed.count())
         .with("status", 200);

    // JSON output:
    // {"timestamp":"2025-01-15T14:30:45Z","level":"INFO",
    //  "message":"Request processed","user_id":"user_123",
    //  "endpoint":"/api/users","duration_ms":"42","status":"200"}
    cout << entry.to_json() << endl;
}
```

**Benefits of structured logging:**
- **Searchable** — query by any field (`user_id=123 AND status=500`)
- **Aggregatable** — compute averages, percentiles (`avg(duration_ms)`)
- **Alertable** — trigger alerts on patterns (`error_count > 100 in 5min`)
- **Parseable** — no regex needed to extract fields from log lines

---


## 11.3 Unit Testing (C++)

> Unit testing verifies that individual components (functions, classes, methods) work correctly in isolation. Tests catch bugs early, enable safe refactoring, and serve as living documentation. In C++, **Google Test (gtest)** is the most widely used testing framework, and **Google Mock (gmock)** provides mocking capabilities for testing components with dependencies.

---

### Google Test (gtest) Framework

Google Test provides macros for defining tests, making assertions, and organizing test suites.

**Basic test structure:**

```cpp
// calculator.h
class Calculator {
public:
    int add(int a, int b) { return a + b; }
    int subtract(int a, int b) { return a - b; }
    int multiply(int a, int b) { return a * b; }

    double divide(double a, double b) {
        if (b == 0.0) throw std::invalid_argument("Division by zero");
        return a / b;
    }
};
```

```cpp
// calculator_test.cpp
#include <gtest/gtest.h>
#include "calculator.h"

// TEST(TestSuiteName, TestName)
TEST(CalculatorTest, AddPositiveNumbers) {
    Calculator calc;
    EXPECT_EQ(calc.add(2, 3), 5);
}

TEST(CalculatorTest, AddNegativeNumbers) {
    Calculator calc;
    EXPECT_EQ(calc.add(-2, -3), -5);
}

TEST(CalculatorTest, AddZero) {
    Calculator calc;
    EXPECT_EQ(calc.add(5, 0), 5);
    EXPECT_EQ(calc.add(0, 5), 5);
    EXPECT_EQ(calc.add(0, 0), 0);
}

TEST(CalculatorTest, SubtractNumbers) {
    Calculator calc;
    EXPECT_EQ(calc.subtract(10, 3), 7);
    EXPECT_EQ(calc.subtract(3, 10), -7);
}

TEST(CalculatorTest, MultiplyNumbers) {
    Calculator calc;
    EXPECT_EQ(calc.multiply(4, 5), 20);
    EXPECT_EQ(calc.multiply(-2, 3), -6);
    EXPECT_EQ(calc.multiply(0, 100), 0);
}

TEST(CalculatorTest, DivideNumbers) {
    Calculator calc;
    EXPECT_DOUBLE_EQ(calc.divide(10.0, 3.0), 10.0 / 3.0);
    EXPECT_DOUBLE_EQ(calc.divide(0.0, 5.0), 0.0);
}

TEST(CalculatorTest, DivideByZeroThrows) {
    Calculator calc;
    EXPECT_THROW(calc.divide(10.0, 0.0), std::invalid_argument);
}

// Run all tests
// int main(int argc, char** argv) {
//     testing::InitGoogleTest(&argc, argv);
//     return RUN_ALL_TESTS();
// }
```

**Compiling and running:**

```bash
# Compile with gtest
g++ -std=c++17 calculator_test.cpp -lgtest -lgtest_main -pthread -o test
./test

# Output:
# [==========] Running 7 tests from 1 test suite.
# [----------] 7 tests from CalculatorTest
# [ RUN      ] CalculatorTest.AddPositiveNumbers
# [       OK ] CalculatorTest.AddPositiveNumbers (0 ms)
# ...
# [==========] 7 tests from 1 test suite ran. (1 ms total)
# [  PASSED  ] 7 tests.
```

---

### Assertions (EXPECT_EQ, ASSERT_TRUE, etc.)

Google Test provides two families of assertions:
- **EXPECT_*** — records failure but continues the test (non-fatal)
- **ASSERT_*** — records failure and stops the test immediately (fatal)

```cpp
TEST(AssertionExamples, BooleanAssertions) {
    bool connected = true;
    EXPECT_TRUE(connected);
    EXPECT_FALSE(!connected);

    // ASSERT stops the test if it fails — use when continuing makes no sense
    int* ptr = new int(42);
    ASSERT_NE(ptr, nullptr);  // if null, don't dereference below
    EXPECT_EQ(*ptr, 42);      // safe because ASSERT above passed
    delete ptr;
}

TEST(AssertionExamples, EqualityAssertions) {
    // Integer equality
    EXPECT_EQ(2 + 2, 4);      // ==
    EXPECT_NE(2 + 2, 5);      // !=

    // Comparison
    EXPECT_LT(3, 5);          // <
    EXPECT_LE(3, 3);          // <=
    EXPECT_GT(5, 3);          // >
    EXPECT_GE(5, 5);          // >=
}

TEST(AssertionExamples, FloatingPointAssertions) {
    double result = 0.1 + 0.2;

    // EXPECT_EQ would fail due to floating-point precision
    // EXPECT_EQ(result, 0.3);  // FAILS!

    // Use EXPECT_DOUBLE_EQ (4 ULPs tolerance) or EXPECT_NEAR
    EXPECT_DOUBLE_EQ(1.0 / 3.0, 1.0 / 3.0);
    EXPECT_NEAR(result, 0.3, 1e-10);  // within tolerance
    EXPECT_FLOAT_EQ(0.1f + 0.2f, 0.3f);  // for floats
}

TEST(AssertionExamples, StringAssertions) {
    string greeting = "Hello, World!";

    EXPECT_EQ(greeting, "Hello, World!");
    EXPECT_NE(greeting, "hello, world!");

    // C-string specific assertions
    EXPECT_STREQ("hello", "hello");       // strcmp == 0
    EXPECT_STRNE("hello", "world");       // strcmp != 0
    EXPECT_STRCASEEQ("Hello", "hello");   // case-insensitive equal
    EXPECT_STRCASENE("Hello", "World");   // case-insensitive not equal
}

TEST(AssertionExamples, ExceptionAssertions) {
    // Expect a specific exception type
    EXPECT_THROW(throw runtime_error("oops"), runtime_error);

    // Expect any exception
    EXPECT_ANY_THROW(throw 42);

    // Expect no exception
    EXPECT_NO_THROW(int x = 2 + 2);
}

TEST(AssertionExamples, CustomFailureMessages) {
    int value = 42;
    EXPECT_EQ(value, 42) << "Value should be 42 but got " << value;

    // Custom failure
    if (value != 42) {
        FAIL() << "Unexpected value: " << value;
    }

    // Custom success (rarely needed)
    SUCCEED();
}
```

**EXPECT vs ASSERT — when to use which:**

| Assertion | On Failure | Use When |
|-----------|-----------|----------|
| `EXPECT_*` | Records failure, continues test | Multiple independent checks in one test |
| `ASSERT_*` | Records failure, stops test | Subsequent checks depend on this one |

```cpp
TEST(Example, UseAssertForPrerequisites) {
    vector<int> data = get_data();

    // ASSERT: if empty, accessing data[0] below would crash
    ASSERT_FALSE(data.empty()) << "Data should not be empty";

    // EXPECT: these are independent checks
    EXPECT_GE(data[0], 0);
    EXPECT_LE(data[0], 100);
    EXPECT_EQ(data.size(), 10);
}
```

---

### Test Fixtures

A **test fixture** is a class that sets up common test state. Tests in the fixture share the setup/teardown logic but each test gets a fresh instance.

```cpp
#include <gtest/gtest.h>
#include <vector>
#include <string>
using namespace std;

// System under test
class UserRepository {
    unordered_map<int, string> users;
    int next_id = 1;

public:
    int add_user(const string& name) {
        int id = next_id++;
        users[id] = name;
        return id;
    }

    optional<string> find_user(int id) const {
        auto it = users.find(id);
        if (it != users.end()) return it->second;
        return nullopt;
    }

    bool remove_user(int id) {
        return users.erase(id) > 0;
    }

    size_t count() const { return users.size(); }
};

// Test fixture — inherits from ::testing::Test
class UserRepositoryTest : public ::testing::Test {
protected:
    UserRepository repo;
    int alice_id;
    int bob_id;

    // Called before EACH test
    void SetUp() override {
        alice_id = repo.add_user("Alice");
        bob_id = repo.add_user("Bob");
    }

    // Called after EACH test
    void TearDown() override {
        // Cleanup if needed (e.g., close connections, delete temp files)
    }
};

// TEST_F (not TEST) — uses the fixture
TEST_F(UserRepositoryTest, FindExistingUser) {
    auto user = repo.find_user(alice_id);
    ASSERT_TRUE(user.has_value());
    EXPECT_EQ(*user, "Alice");
}

TEST_F(UserRepositoryTest, FindNonExistingUser) {
    auto user = repo.find_user(999);
    EXPECT_FALSE(user.has_value());
}

TEST_F(UserRepositoryTest, RemoveUser) {
    EXPECT_TRUE(repo.remove_user(alice_id));
    EXPECT_FALSE(repo.find_user(alice_id).has_value());
    EXPECT_EQ(repo.count(), 1);  // only Bob remains
}

TEST_F(UserRepositoryTest, RemoveNonExistingUser) {
    EXPECT_FALSE(repo.remove_user(999));
    EXPECT_EQ(repo.count(), 2);  // nothing removed
}

TEST_F(UserRepositoryTest, CountAfterSetup) {
    // Each test gets a fresh fixture — SetUp() runs again
    EXPECT_EQ(repo.count(), 2);  // Alice and Bob from SetUp()
}
```

**Parameterized tests — run the same test with different inputs:**

```cpp
// Test with multiple inputs
class IsPrimeTest : public ::testing::TestWithParam<pair<int, bool>> {};

bool is_prime(int n) {
    if (n < 2) return false;
    for (int i = 2; i * i <= n; i++) {
        if (n % i == 0) return false;
    }
    return true;
}

TEST_P(IsPrimeTest, CheckPrimality) {
    auto [input, expected] = GetParam();
    EXPECT_EQ(is_prime(input), expected)
        << "is_prime(" << input << ") should be " << expected;
}

INSTANTIATE_TEST_SUITE_P(
    PrimeNumbers,
    IsPrimeTest,
    ::testing::Values(
        make_pair(0, false),
        make_pair(1, false),
        make_pair(2, true),
        make_pair(3, true),
        make_pair(4, false),
        make_pair(5, true),
        make_pair(97, true),
        make_pair(100, false)
    )
);
```

---

### Mocking with Google Mock (gmock)

**Mocking** replaces real dependencies with controlled substitutes. This lets you test a class in isolation without needing a real database, network, or file system.

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
using namespace std;
using ::testing::Return;
using ::testing::_;
using ::testing::AtLeast;
using ::testing::Throw;

// Interface (abstract class) — the dependency
class IDatabase {
public:
    virtual ~IDatabase() = default;
    virtual bool connect(const string& host, int port) = 0;
    virtual optional<string> get(const string& key) = 0;
    virtual bool set(const string& key, const string& value) = 0;
    virtual void disconnect() = 0;
};

// Mock class — generated by gmock
class MockDatabase : public IDatabase {
public:
    MOCK_METHOD(bool, connect, (const string& host, int port), (override));
    MOCK_METHOD(optional<string>, get, (const string& key), (override));
    MOCK_METHOD(bool, set, (const string& key, const string& value), (override));
    MOCK_METHOD(void, disconnect, (), (override));
};

// System under test — depends on IDatabase
class UserService {
    IDatabase& db;

public:
    UserService(IDatabase& db) : db(db) {}

    string get_user_name(const string& user_id) {
        auto result = db.get("user:" + user_id);
        if (result) return *result;
        return "Unknown";
    }

    bool save_user(const string& user_id, const string& name) {
        return db.set("user:" + user_id, name);
    }
};

// Tests using mocks
class UserServiceTest : public ::testing::Test {
protected:
    MockDatabase mock_db;
    unique_ptr<UserService> service;

    void SetUp() override {
        service = make_unique<UserService>(mock_db);
    }
};

TEST_F(UserServiceTest, GetUserNameReturnsNameWhenFound) {
    // Set up expectation: when get("user:123") is called, return "Alice"
    EXPECT_CALL(mock_db, get("user:123"))
        .WillOnce(Return(optional<string>("Alice")));

    string name = service->get_user_name("123");
    EXPECT_EQ(name, "Alice");
}

TEST_F(UserServiceTest, GetUserNameReturnsUnknownWhenNotFound) {
    EXPECT_CALL(mock_db, get("user:456"))
        .WillOnce(Return(nullopt));

    string name = service->get_user_name("456");
    EXPECT_EQ(name, "Unknown");
}

TEST_F(UserServiceTest, SaveUserCallsDatabase) {
    EXPECT_CALL(mock_db, set("user:123", "Alice"))
        .WillOnce(Return(true));

    bool result = service->save_user("123", "Alice");
    EXPECT_TRUE(result);
}

TEST_F(UserServiceTest, SaveUserReturnsFalseOnDbFailure) {
    EXPECT_CALL(mock_db, set(_, _))  // _ matches any argument
        .WillOnce(Return(false));

    bool result = service->save_user("123", "Alice");
    EXPECT_FALSE(result);
}
```

**Common gmock matchers and actions:**

```cpp
// Matchers — specify expected argument values
EXPECT_CALL(mock, method("exact_value"));     // exact match
EXPECT_CALL(mock, method(_));                  // any value
EXPECT_CALL(mock, method(Gt(5)));             // greater than 5
EXPECT_CALL(mock, method(HasSubstr("hello"))); // string contains "hello"
EXPECT_CALL(mock, method(Not(IsEmpty())));     // not empty

// Cardinality — how many times the method should be called
EXPECT_CALL(mock, method(_)).Times(3);         // exactly 3 times
EXPECT_CALL(mock, method(_)).Times(AtLeast(1)); // at least once
EXPECT_CALL(mock, method(_)).Times(AtMost(5));  // at most 5 times
EXPECT_CALL(mock, method(_)).Times(Between(2, 4)); // 2 to 4 times

// Actions — what to do when called
EXPECT_CALL(mock, method(_))
    .WillOnce(Return(42))           // first call returns 42
    .WillOnce(Return(43))           // second call returns 43
    .WillRepeatedly(Return(0));     // all subsequent calls return 0

EXPECT_CALL(mock, method(_))
    .WillOnce(Throw(runtime_error("fail")));  // throw an exception

// Sequences — enforce call order
{
    InSequence seq;
    EXPECT_CALL(mock, connect(_, _)).WillOnce(Return(true));
    EXPECT_CALL(mock, get(_)).WillOnce(Return("data"));
    EXPECT_CALL(mock, disconnect());
    // connect must be called before get, get before disconnect
}
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

**TDD example — building a Stack:**

```cpp
// Step 1: RED — Write a failing test
TEST(StackTest, NewStackIsEmpty) {
    Stack<int> s;
    EXPECT_TRUE(s.is_empty());
    EXPECT_EQ(s.size(), 0);
}
// This won't compile — Stack doesn't exist yet

// Step 2: GREEN — Write minimum code to pass
template <typename T>
class Stack {
public:
    bool is_empty() const { return true; }
    size_t size() const { return 0; }
};
// Test passes!

// Step 3: RED — Next test
TEST(StackTest, PushMakesStackNonEmpty) {
    Stack<int> s;
    s.push(42);
    EXPECT_FALSE(s.is_empty());
    EXPECT_EQ(s.size(), 1);
}
// Fails — push doesn't exist, is_empty always returns true

// Step 4: GREEN — Implement push
template <typename T>
class Stack {
    vector<T> data;
public:
    void push(const T& value) { data.push_back(value); }
    bool is_empty() const { return data.empty(); }
    size_t size() const { return data.size(); }
};
// Test passes!

// Step 5: RED — Test pop
TEST(StackTest, PopReturnsLastPushed) {
    Stack<int> s;
    s.push(10);
    s.push(20);
    EXPECT_EQ(s.pop(), 20);
    EXPECT_EQ(s.pop(), 10);
    EXPECT_TRUE(s.is_empty());
}

// Step 6: GREEN — Implement pop
template <typename T>
class Stack {
    vector<T> data;
public:
    void push(const T& value) { data.push_back(value); }
    T pop() {
        T value = data.back();
        data.pop_back();
        return value;
    }
    bool is_empty() const { return data.empty(); }
    size_t size() const { return data.size(); }
};

// Step 7: RED — Test edge case
TEST(StackTest, PopOnEmptyThrows) {
    Stack<int> s;
    EXPECT_THROW(s.pop(), runtime_error);
}

// Step 8: GREEN — Handle edge case
T pop() {
    if (data.empty()) throw runtime_error("Stack is empty");
    T value = data.back();
    data.pop_back();
    return value;
}

// Step 9: REFACTOR — Clean up, add top(), etc.
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

```cpp
// BAD: Hard to test — creates its own dependencies
class OrderService_Bad {
    MySQLDatabase db;          // concrete dependency — can't mock
    SmtpEmailSender email;     // concrete dependency — can't mock

public:
    OrderService_Bad()
        : db("localhost", 3306),
          email("smtp.example.com") {}

    void place_order(const Order& order) {
        db.save(order);                    // hits real database
        email.send(order.customer_email,   // sends real email
                   "Order confirmed");
    }
};
// To test this, you need a real MySQL database and SMTP server!

// GOOD: Testable — dependencies are injected
class IOrderRepository {
public:
    virtual ~IOrderRepository() = default;
    virtual void save(const Order& order) = 0;
    virtual optional<Order> find(int id) = 0;
};

class IEmailService {
public:
    virtual ~IEmailService() = default;
    virtual void send(const string& to, const string& subject,
                      const string& body) = 0;
};

class OrderService {
    IOrderRepository& repo;
    IEmailService& email;

public:
    // Dependencies injected through constructor
    OrderService(IOrderRepository& repo, IEmailService& email)
        : repo(repo), email(email) {}

    void place_order(const Order& order) {
        repo.save(order);
        email.send(order.customer_email, "Order Confirmed",
                   "Your order #" + to_string(order.id) + " has been placed.");
    }
};

// Now we can test with mocks!
class MockOrderRepo : public IOrderRepository {
public:
    MOCK_METHOD(void, save, (const Order& order), (override));
    MOCK_METHOD(optional<Order>, find, (int id), (override));
};

class MockEmailService : public IEmailService {
public:
    MOCK_METHOD(void, send, (const string& to, const string& subject,
                             const string& body), (override));
};

TEST(OrderServiceTest, PlaceOrderSavesAndSendsEmail) {
    MockOrderRepo mock_repo;
    MockEmailService mock_email;
    OrderService service(mock_repo, mock_email);

    Order order{1, "alice@example.com", 99.99};

    // Expect save to be called with the order
    EXPECT_CALL(mock_repo, save(_)).Times(1);

    // Expect email to be sent to the customer
    EXPECT_CALL(mock_email, send("alice@example.com", _, _)).Times(1);

    service.place_order(order);
}

TEST(OrderServiceTest, PlaceOrderHandlesDbFailure) {
    MockOrderRepo mock_repo;
    MockEmailService mock_email;
    OrderService service(mock_repo, mock_email);

    Order order{1, "alice@example.com", 99.99};

    // Simulate database failure
    EXPECT_CALL(mock_repo, save(_))
        .WillOnce(Throw(runtime_error("DB connection lost")));

    // Email should NOT be sent if save fails
    EXPECT_CALL(mock_email, send(_, _, _)).Times(0);

    EXPECT_THROW(service.place_order(order), runtime_error);
}
```

**Principles for testable code:**

| Principle | Description | Example |
|-----------|-------------|---------|
| **Depend on abstractions** | Use interfaces, not concrete classes | `IDatabase&` not `MySQL` |
| **Inject dependencies** | Pass dependencies in, don't create them | Constructor injection |
| **Single Responsibility** | Each class does one thing | Easier to test in isolation |
| **Pure functions** | Same input → same output, no side effects | Easiest to test |
| **Avoid global state** | No singletons in business logic | Pass logger as dependency |
| **Small methods** | Each method does one thing | Easier to write focused tests |

---


## Summary & Key Takeaways

| Topic | Core Concept | Key Tools / Patterns |
|-------|-------------|---------------------|
| Exceptions | Throw on failure, catch to handle | `try/catch/throw`, exception hierarchy |
| Custom Exceptions | Domain-specific error types with context | Inherit from `runtime_error` or `logic_error` |
| Exception Safety | Guarantees about state after an exception | Basic, Strong (copy-and-swap), No-throw |
| RAII | Resources tied to object lifetime | Smart pointers, lock guards, custom wrappers |
| noexcept | Promise that a function won't throw | Destructors, move ops, swap, getters |
| Error Codes vs Exceptions | Choose based on expected vs exceptional | `optional` for "not found", exceptions for failures |
| std::optional / expected | Type-safe "value or nothing/error" | `nullopt`, `value_or()`, `has_value()` |
| Logging Levels | Control verbosity | TRACE, DEBUG, INFO, WARN, ERROR, FATAL |
| Logger Design | Singleton + Strategy (sinks) | Console, file, rotating file sinks |
| Structured Logging | Machine-parseable log records | JSON format, key-value fields |
| Google Test | Test framework for C++ | `TEST()`, `TEST_F()`, `EXPECT_*`, `ASSERT_*` |
| Test Fixtures | Shared setup/teardown for tests | `::testing::Test`, `SetUp()`, `TearDown()` |
| Google Mock | Mock dependencies for isolation | `MOCK_METHOD`, `EXPECT_CALL`, matchers |
| TDD | Write tests first, then implement | Red → Green → Refactor cycle |
| Testable Code | Dependency injection, abstractions | Interfaces, constructor injection |

---

## Interview Tips for Module 11

1. **"What are the exception safety guarantees?"** Three levels: (a) **No-throw** — function never throws (destructors, swap, move ops). (b) **Strong** — if exception thrown, state is unchanged (copy-and-swap idiom). (c) **Basic** — no resource leaks, object in valid but possibly modified state. Always aim for at least basic guarantee. Use RAII to ensure resources are cleaned up.

2. **"What is RAII and why is it important?"** Resource Acquisition Is Initialization — acquire resources in constructors, release in destructors. Since destructors run during stack unwinding, resources are always cleaned up even when exceptions are thrown. Examples: `unique_ptr` for memory, `lock_guard` for mutexes, `ifstream` for files. RAII is the foundation of exception-safe C++.

3. **"When would you use exceptions vs error codes?"** Exceptions for rare, unexpected failures (network down, out of memory, file corruption). Error codes or `std::optional` for expected conditions (user not found, invalid input, empty result). Exceptions have zero cost when not thrown but are expensive when thrown. Error codes have a small constant cost on every call. Use both in the same codebase — exceptions for infrastructure failures, optional/error codes for business logic.

4. **"How would you design a logging system?"** Singleton Logger with configurable log level and multiple sinks (Strategy pattern). Sinks: console, file, rotating file, network. Each sink can have its own minimum level (FilteredSink decorator). Use structured logging (JSON) for production — enables searching and alerting. Include timestamp, level, source location, and correlation ID in every log entry.

5. **"What is the difference between EXPECT and ASSERT in Google Test?"** `EXPECT_*` records a failure but continues the test — use for independent checks. `ASSERT_*` records a failure and stops the test — use when subsequent checks depend on this one (e.g., assert pointer is not null before dereferencing). Prefer `EXPECT_*` by default; use `ASSERT_*` only for prerequisites.

6. **"How do you test code that depends on a database?"** Use Dependency Injection: define an interface (`IDatabase`), inject it into the class under test. In tests, create a mock (`MockDatabase` using gmock) that returns controlled values. This tests your logic without needing a real database. Set expectations with `EXPECT_CALL` to verify the correct queries are made.

7. **"What is Test-Driven Development?"** Write a failing test first (Red), write minimum code to pass (Green), then refactor while keeping tests green. Benefits: forces testable design, catches bugs immediately, tests serve as documentation, gives confidence to refactor. The key discipline is writing the test before the implementation.

8. **"How do you make code testable?"** (a) Depend on abstractions (interfaces), not concrete classes. (b) Inject dependencies through constructors. (c) Follow Single Responsibility — each class does one thing. (d) Avoid global state and singletons in business logic. (e) Prefer pure functions where possible. (f) Keep methods small and focused.

9. **"What is a custom exception and when would you create one?"** Create custom exceptions when you need domain-specific error types with additional context (error codes, field names, resource IDs). Inherit from `std::runtime_error` or `std::logic_error`. Build a hierarchy: `AppException` → `AuthException`, `NotFoundException`, `ValidationException`. This enables catching specific error types at different layers.

10. **"Explain noexcept and when to use it."** `noexcept` tells the compiler a function will never throw. If it does throw, `std::terminate()` is called. Use for: destructors (implicit), move constructors/assignment (enables vector optimizations), swap, and simple getters. Don't use for functions that allocate memory or do I/O. `noexcept` move operations are critical — `std::vector` falls back to copying if move isn't `noexcept`.

11. **"What is structured logging?"** Producing log records as machine-parseable data (JSON, key-value pairs) instead of free-form text. Each log entry has typed fields (user_id, duration_ms, status_code) that can be searched, filtered, and aggregated by log analysis tools. Benefits: no regex needed to parse logs, enables dashboards and alerts, supports correlation across services using request IDs.

12. **"How would you handle errors in a multi-layered system?"** Each layer catches exceptions it can handle and re-throws or wraps for the layer above. Repository layer throws `DatabaseException`. Service layer catches it and throws `ServiceException` with business context. Controller/API layer catches `ServiceException` and returns appropriate HTTP status codes. Use RAII at every layer to prevent resource leaks during unwinding.

