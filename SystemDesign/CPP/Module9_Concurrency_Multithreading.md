# Module 9: Concurrency & Multithreading in C++

> Concurrency is the ability of a program to manage multiple tasks at the same time. Multithreading is one way to achieve concurrency — by running multiple threads of execution within a single process. Modern C++ (C++11 and beyond) provides a rich standard library for threading, synchronization, atomic operations, and asynchronous programming. Mastering these concepts is essential for building performant, correct, and scalable systems.

---

## 9.1 Thread Basics

> A **thread** is the smallest unit of execution that can be scheduled by the operating system. Multiple threads within the same process share the same memory space (heap, global data) but each has its own stack and program counter. C++11 introduced `std::thread` as a portable, standard way to create and manage threads.

---

### std::thread Creation and Joining

**Creating a thread** requires passing a callable (function, lambda, functor, or member function) to the `std::thread` constructor. The thread begins executing immediately upon construction.

```cpp
#include <iostream>
#include <thread>
using namespace std;

// Simple function to run in a thread
void greet(const string& name) {
    cout << "Hello from thread! Name: " << name << endl;
}

int main() {
    // Create a thread that runs greet()
    thread t1(greet, "Alice");

    // Main thread continues executing here...
    cout << "Hello from main thread!" << endl;

    // MUST join or detach before t1 goes out of scope
    t1.join();  // blocks until t1 finishes

    return 0;
}
```

**What happens if you don't join or detach?**

If a `std::thread` object is destroyed while still **joinable** (i.e., it represents a running thread and hasn't been joined or detached), `std::terminate()` is called — your program crashes.

```cpp
void dangerous() {
    thread t([] { cout << "Working..." << endl; });
    // t goes out of scope without join() or detach()
    // → std::terminate() is called → PROGRAM CRASHES
}
```

**join() vs detach():**

| Method | Behavior | When to use |
|--------|----------|-------------|
| `join()` | Blocks the calling thread until the target thread finishes | When you need the thread's result or must ensure it completes |
| `detach()` | Separates the thread from the `std::thread` object; thread runs independently | For fire-and-forget tasks (logging, background cleanup) |

```cpp
// join() — wait for thread to complete
void example_join() {
    thread t([] {
        this_thread::sleep_for(chrono::seconds(2));
        cout << "Thread finished work" << endl;
    });

    cout << "Waiting for thread..." << endl;
    t.join();  // blocks here for ~2 seconds
    cout << "Thread done, continuing" << endl;
}

// detach() — fire and forget
void example_detach() {
    thread t([] {
        this_thread::sleep_for(chrono::seconds(2));
        cout << "Background thread done" << endl;
    });

    t.detach();  // thread runs independently
    cout << "Main continues immediately" << endl;
    // WARNING: if main() exits before the detached thread finishes,
    // the detached thread is killed (undefined behavior for its resources)
}
```

**Checking if a thread is joinable:**

```cpp
thread t(some_function);

if (t.joinable()) {
    t.join();  // safe to call
}

// After join() or detach(), joinable() returns false
// Calling join() on a non-joinable thread throws std::system_error
```

**Using lambdas (most common in modern C++):**

```cpp
int main() {
    int result = 0;

    thread t([&result] {
        // Capture by reference to write back results
        result = 42;
        cout << "Thread computed result" << endl;
    });

    t.join();
    cout << "Result: " << result << endl;  // 42
}
```

**Using functors (function objects):**

```cpp
class Worker {
    int id;
public:
    Worker(int id) : id(id) {}

    void operator()() const {
        cout << "Worker " << id << " executing" << endl;
    }
};

int main() {
    Worker w(1);
    thread t1(w);       // copies the functor
    thread t2(Worker(2)); // move constructs — use extra parens or braces to avoid "most vexing parse"
    thread t3{Worker(3)}; // brace initialization avoids ambiguity

    t1.join();
    t2.join();
    t3.join();
}
```

**Using member functions:**

```cpp
class Printer {
public:
    void print(const string& msg) {
        cout << "Printer says: " << msg << endl;
    }
};

int main() {
    Printer p;
    // Pass: &ClassName::method, pointer-to-object, arguments...
    thread t(&Printer::print, &p, "Hello from member function");
    t.join();
}
```

---

### Detached Threads

A **detached thread** runs independently of the `std::thread` object. Once detached, you cannot join it or check its status. The thread continues running until its function returns, and its resources are automatically cleaned up.

```cpp
void background_task() {
    while (true) {
        // Periodic cleanup, logging, etc.
        this_thread::sleep_for(chrono::minutes(1));
        cout << "Background cleanup..." << endl;
    }
}

int main() {
    thread t(background_task);
    t.detach();  // runs forever in the background

    // Main thread continues...
    // When main() exits, detached threads are terminated
}
```

**Dangers of detached threads:**
- If the detached thread accesses local variables of the creating function, and that function returns, you get **dangling references** (undefined behavior)
- No way to know when the thread finishes
- No way to catch exceptions from the thread
- If `main()` exits, all detached threads are killed immediately

```cpp
// BAD: Dangling reference with detached thread
void bad_detach() {
    int local_data = 42;

    thread t([&local_data] {
        this_thread::sleep_for(chrono::seconds(1));
        cout << local_data << endl;  // UNDEFINED BEHAVIOR!
        // bad_detach() has already returned, local_data is destroyed
    });

    t.detach();
    // Function returns, local_data is destroyed
    // But the thread is still running and will access destroyed memory!
}

// GOOD: Copy data into the thread
void good_detach() {
    int local_data = 42;

    thread t([local_data] {  // capture by VALUE — safe copy
        this_thread::sleep_for(chrono::seconds(1));
        cout << local_data << endl;  // safe — thread owns its own copy
    });

    t.detach();
}
```

---

### Thread Arguments (by Value, by Reference with std::ref)

By default, `std::thread` **copies** all arguments into the new thread's internal storage, even if the function signature takes a reference. To pass by reference, you must use `std::ref()`.

```cpp
// Function that modifies its argument
void increment(int& value) {
    value += 10;
}

int main() {
    int x = 5;

    // BAD: This won't compile or won't modify x!
    // thread t(increment, x);  // x is COPIED into the thread
    // The thread's internal copy is passed to increment, not x itself

    // GOOD: Use std::ref to pass by reference
    thread t(increment, ref(x));
    t.join();

    cout << x << endl;  // 15 — x was modified by the thread
}
```

**Why does std::thread copy by default?**
- Safety: if the calling function returns before the thread finishes, the original variable is destroyed. Copying prevents dangling references.
- `std::ref` is an explicit opt-in that says "I guarantee this reference will remain valid."

```cpp
// Passing multiple arguments
void process(int id, const string& name, double value) {
    cout << "Processing #" << id << ": " << name << " = " << value << endl;
}

int main() {
    thread t(process, 1, "temperature", 98.6);
    t.join();
}

// Passing by const reference — std::cref
void read_data(const vector<int>& data) {
    cout << "Data size: " << data.size() << endl;
}

int main() {
    vector<int> big_data(1000000, 42);

    // Without cref: copies 1 million ints into the thread — wasteful!
    // thread t1(read_data, big_data);

    // With cref: passes a reference — efficient (but you must ensure big_data outlives the thread)
    thread t(read_data, cref(big_data));
    t.join();
}
```

**Moving arguments into threads:**

```cpp
void take_ownership(unique_ptr<int> ptr) {
    cout << "Thread owns: " << *ptr << endl;
}

int main() {
    auto ptr = make_unique<int>(42);

    // unique_ptr can't be copied, must be moved
    thread t(take_ownership, std::move(ptr));
    t.join();

    // ptr is now nullptr — ownership transferred to the thread
}
```

---

### Thread ID and Hardware Concurrency

Every thread has a unique ID, and you can query how many hardware threads the system supports.

```cpp
#include <thread>
#include <iostream>
using namespace std;

void worker() {
    // Get the current thread's ID
    cout << "Worker thread ID: " << this_thread::get_id() << endl;
}

int main() {
    cout << "Main thread ID: " << this_thread::get_id() << endl;

    // How many threads can truly run in parallel?
    unsigned int cores = thread::hardware_concurrency();
    cout << "Hardware concurrency: " << cores << endl;
    // Returns 0 if the value is not computable or not well-defined

    // Create one thread per core
    vector<thread> threads;
    for (unsigned int i = 0; i < cores; i++) {
        threads.emplace_back(worker);
    }

    for (auto& t : threads) {
        t.join();
    }
}
```

**`this_thread` namespace utilities:**

```cpp
// Get current thread's ID
auto id = this_thread::get_id();

// Sleep for a duration
this_thread::sleep_for(chrono::milliseconds(100));

// Sleep until a time point
auto wake_time = chrono::steady_clock::now() + chrono::seconds(5);
this_thread::sleep_until(wake_time);

// Yield execution to another thread (hint to the scheduler)
this_thread::yield();
```

**Thread ownership and move semantics:**

`std::thread` is **movable but not copyable** — each thread object uniquely owns its thread of execution.

```cpp
thread create_thread() {
    thread t([] { cout << "Hello from moved thread" << endl; });
    return t;  // moved out of the function
}

int main() {
    thread t1 = create_thread();  // move construction
    t1.join();

    thread t2([] { cout << "t2" << endl; });
    // thread t3 = t2;  // ERROR: threads are not copyable
    thread t3 = std::move(t2);  // OK: transfer ownership
    // t2 is now empty (not joinable)
    t3.join();
}
```

---

### Practical Example: Parallel Computation

```cpp
#include <thread>
#include <vector>
#include <numeric>
#include <iostream>
using namespace std;

// Parallel sum: divide array into chunks, sum each in a separate thread
void partial_sum(const vector<int>& data, int start, int end, long long& result) {
    result = 0;
    for (int i = start; i < end; i++) {
        result += data[i];
    }
}

int main() {
    const int N = 10000000;
    vector<int> data(N);
    iota(data.begin(), data.end(), 1);  // fill with 1, 2, 3, ..., N

    unsigned int num_threads = thread::hardware_concurrency();
    if (num_threads == 0) num_threads = 4;  // fallback

    int chunk_size = N / num_threads;
    vector<thread> threads;
    vector<long long> results(num_threads, 0);

    // Launch threads
    for (unsigned int i = 0; i < num_threads; i++) {
        int start = i * chunk_size;
        int end = (i == num_threads - 1) ? N : start + chunk_size;
        threads.emplace_back(partial_sum, cref(data), start, end, ref(results[i]));
    }

    // Wait for all threads
    for (auto& t : threads) {
        t.join();
    }

    // Combine results
    long long total = 0;
    for (auto r : results) {
        total += r;
    }

    cout << "Sum: " << total << endl;
    // Expected: N*(N+1)/2 = 50000005000000
}
```

---

### Common Pitfalls with std::thread

1. **Forgetting to join or detach** → `std::terminate()` crash
2. **Dangling references** with detached threads or `std::ref`
3. **Data races** — multiple threads accessing shared data without synchronization
4. **Exception safety** — if an exception is thrown before `join()`, the thread is never joined

```cpp
// Exception-safe thread management using RAII
class ThreadGuard {
    thread& t;
public:
    explicit ThreadGuard(thread& t) : t(t) {}

    ~ThreadGuard() {
        if (t.joinable()) {
            t.join();  // always join on destruction
        }
    }

    // Non-copyable
    ThreadGuard(const ThreadGuard&) = delete;
    ThreadGuard& operator=(const ThreadGuard&) = delete;
};

void safe_function() {
    int data = 0;
    thread t([&data] { data = 42; });
    ThreadGuard guard(t);  // will join even if an exception is thrown

    // ... code that might throw ...
    // guard's destructor calls t.join() automatically
}
```

**C++20 `std::jthread` — auto-joining thread:**

```cpp
#include <thread>

void work() {
    // ...
}

int main() {
    jthread t(work);
    // No need to call join()!
    // jthread automatically joins in its destructor
    // Also supports cooperative cancellation via stop_token
}
```

---


## 9.2 Synchronization Primitives

> When multiple threads access shared data, **synchronization** is required to prevent data races and ensure correctness. A **data race** occurs when two or more threads access the same memory location concurrently, at least one access is a write, and there is no synchronization. Data races cause undefined behavior in C++.

---

### std::mutex, std::recursive_mutex, std::timed_mutex

A **mutex** (mutual exclusion) is the most fundamental synchronization primitive. It ensures that only one thread can access a critical section at a time.

**std::mutex — basic mutual exclusion:**

```cpp
#include <mutex>
#include <thread>
#include <iostream>
using namespace std;

int shared_counter = 0;
mutex mtx;

void increment(int times) {
    for (int i = 0; i < times; i++) {
        mtx.lock();       // acquire the lock
        shared_counter++;  // critical section — only one thread at a time
        mtx.unlock();     // release the lock
    }
}

int main() {
    thread t1(increment, 100000);
    thread t2(increment, 100000);

    t1.join();
    t2.join();

    cout << "Counter: " << shared_counter << endl;  // always 200000
}
```

**Without the mutex:**
```cpp
// BAD: Data race — undefined behavior
int shared_counter = 0;

void increment_unsafe(int times) {
    for (int i = 0; i < times; i++) {
        shared_counter++;  // NOT atomic! Read-modify-write is 3 operations
        // Thread 1 reads 5, Thread 2 reads 5
        // Thread 1 writes 6, Thread 2 writes 6
        // Counter should be 7, but it's 6 — lost update!
    }
}
// Result: unpredictable, often less than 200000
```

**std::recursive_mutex — allows the same thread to lock multiple times:**

A regular `mutex` will **deadlock** if the same thread tries to lock it twice. A `recursive_mutex` allows re-entrant locking — the same thread can lock it multiple times, but must unlock the same number of times.

```cpp
#include <mutex>
using namespace std;

class TreeNode {
    int value;
    TreeNode* left;
    TreeNode* right;
    mutable recursive_mutex mtx;

public:
    TreeNode(int v) : value(v), left(nullptr), right(nullptr) {}

    int sum() const {
        lock_guard<recursive_mutex> lock(mtx);
        int total = value;
        if (left) total += left->sum();    // recursive call — re-locks the mutex
        if (right) total += right->sum();  // same thread, same mutex — OK with recursive_mutex
        return total;
    }

    void set_value(int v) {
        lock_guard<recursive_mutex> lock(mtx);
        value = v;
    }
};
```

**When to use recursive_mutex:**
- When a function that holds a lock calls another function that also needs the same lock
- Common in recursive data structures (trees, graphs)
- **Prefer redesigning to avoid the need** — recursive_mutex is slower and often indicates a design issue

**std::timed_mutex — lock with a timeout:**

```cpp
#include <mutex>
#include <chrono>
using namespace std;

timed_mutex tmtx;

void try_work() {
    // Try to acquire the lock for up to 100ms
    if (tmtx.try_lock_for(chrono::milliseconds(100))) {
        // Got the lock — do work
        cout << "Lock acquired, working..." << endl;
        this_thread::sleep_for(chrono::milliseconds(50));
        tmtx.unlock();
    } else {
        // Couldn't get the lock in time
        cout << "Timeout! Lock not acquired." << endl;
    }
}

void try_work_until() {
    auto deadline = chrono::steady_clock::now() + chrono::seconds(1);

    if (tmtx.try_lock_until(deadline)) {
        // Got the lock before the deadline
        cout << "Lock acquired before deadline" << endl;
        tmtx.unlock();
    } else {
        cout << "Deadline passed, lock not acquired" << endl;
    }
}
```

**Mutex comparison:**

| Mutex Type | Re-entrant | Timed | Use Case |
|-----------|-----------|-------|----------|
| `std::mutex` | No | No | Default choice — simple, fast |
| `std::recursive_mutex` | Yes | No | Recursive functions needing the same lock |
| `std::timed_mutex` | No | Yes | When you can't afford to block indefinitely |
| `std::recursive_timed_mutex` | Yes | Yes | Recursive + timeout (rare) |
| `std::shared_mutex` (C++17) | No | No | Multiple readers, single writer |

---

### std::lock_guard, std::unique_lock, std::scoped_lock

Manually calling `lock()` and `unlock()` is error-prone — if an exception is thrown between them, the mutex is never unlocked (deadlock). **RAII lock wrappers** solve this.

**std::lock_guard — simplest RAII lock:**

```cpp
#include <mutex>
using namespace std;

mutex mtx;
int balance = 1000;

void withdraw(int amount) {
    lock_guard<mutex> lock(mtx);  // locks mutex on construction
    // Critical section
    if (balance >= amount) {
        balance -= amount;
        cout << "Withdrew " << amount << ", balance: " << balance << endl;
    } else {
        cout << "Insufficient funds" << endl;
    }
    // lock_guard destructor unlocks mutex — even if an exception is thrown!
}
```

**std::unique_lock — flexible RAII lock:**

`unique_lock` provides more control than `lock_guard`: deferred locking, timed locking, manual lock/unlock, and transferable ownership.

```cpp
#include <mutex>
using namespace std;

mutex mtx;

void flexible_locking() {
    // Deferred locking — construct without locking
    unique_lock<mutex> lock(mtx, defer_lock);

    // ... do some non-critical work ...

    lock.lock();  // manually lock when ready
    // ... critical section ...
    lock.unlock();  // manually unlock early

    // ... more non-critical work ...

    lock.lock();  // re-lock for another critical section
    // ... another critical section ...
    // destructor unlocks if still locked
}

void timed_locking() {
    timed_mutex tmtx;
    unique_lock<timed_mutex> lock(tmtx, chrono::milliseconds(100));

    if (lock.owns_lock()) {
        cout << "Got the lock!" << endl;
    } else {
        cout << "Timeout" << endl;
    }
}

void try_locking() {
    unique_lock<mutex> lock(mtx, try_to_lock);

    if (lock.owns_lock()) {
        cout << "Got the lock!" << endl;
    } else {
        cout << "Lock is busy, doing something else" << endl;
    }
}

// unique_lock is MOVABLE (lock_guard is not)
unique_lock<mutex> get_lock() {
    unique_lock<mutex> lock(mtx);
    // ... prepare data ...
    return lock;  // transfer ownership to caller
}
```

**std::scoped_lock (C++17) — lock multiple mutexes without deadlock:**

```cpp
#include <mutex>
using namespace std;

mutex mtx1, mtx2;

void transfer(int& from, int& to, int amount) {
    // Locks BOTH mutexes atomically — no deadlock possible
    scoped_lock lock(mtx1, mtx2);

    if (from >= amount) {
        from -= amount;
        to += amount;
    }
    // Both mutexes unlocked when lock goes out of scope
}
```

**Comparison of lock wrappers:**

| Feature | `lock_guard` | `unique_lock` | `scoped_lock` (C++17) |
|---------|-------------|--------------|----------------------|
| RAII lock/unlock | ✅ | ✅ | ✅ |
| Deferred locking | ❌ | ✅ | ❌ |
| Try locking | ❌ | ✅ | ❌ |
| Timed locking | ❌ | ✅ | ❌ |
| Manual lock/unlock | ❌ | ✅ | ❌ |
| Movable | ❌ | ✅ | ❌ |
| Multiple mutexes | ❌ | ❌ | ✅ |
| Use with condition_variable | ❌ | ✅ | ❌ |
| Overhead | Lowest | Slightly higher | Low |

**Rule of thumb:**
- Use `lock_guard` for simple, single-mutex locking
- Use `unique_lock` when you need flexibility (deferred lock, condition variables, timed lock)
- Use `scoped_lock` when locking multiple mutexes

---

### Deadlock Prevention (Lock Ordering, std::lock)

A **deadlock** occurs when two or more threads are each waiting for a lock held by the other, creating a circular dependency where no thread can proceed.

```cpp
// DEADLOCK EXAMPLE:
mutex mtx_a, mtx_b;

void thread1() {
    lock_guard<mutex> lock_a(mtx_a);  // locks A
    this_thread::sleep_for(chrono::milliseconds(1));
    lock_guard<mutex> lock_b(mtx_b);  // waits for B — but thread2 holds B!
}

void thread2() {
    lock_guard<mutex> lock_b(mtx_b);  // locks B
    this_thread::sleep_for(chrono::milliseconds(1));
    lock_guard<mutex> lock_a(mtx_a);  // waits for A — but thread1 holds A!
}

// Thread 1 holds A, waits for B
// Thread 2 holds B, waits for A
// → DEADLOCK — both threads blocked forever
```

**Prevention Strategy 1: Consistent lock ordering**

Always acquire mutexes in the same order across all threads.

```cpp
// GOOD: Both threads lock A first, then B
void thread1_fixed() {
    lock_guard<mutex> lock_a(mtx_a);  // always lock A first
    lock_guard<mutex> lock_b(mtx_b);  // then B
    // ... work ...
}

void thread2_fixed() {
    lock_guard<mutex> lock_a(mtx_a);  // same order: A first
    lock_guard<mutex> lock_b(mtx_b);  // then B
    // ... work ...
}
```

**Prevention Strategy 2: std::lock — lock multiple mutexes atomically**

`std::lock` uses a deadlock-avoidance algorithm (try-and-back-off) to lock multiple mutexes without deadlock, regardless of order.

```cpp
mutex mtx_a, mtx_b;

void safe_transfer() {
    // std::lock locks both mutexes without deadlock
    // Uses internal algorithm to avoid circular waiting
    unique_lock<mutex> lock_a(mtx_a, defer_lock);
    unique_lock<mutex> lock_b(mtx_b, defer_lock);

    lock(lock_a, lock_b);  // atomically locks both — no deadlock

    // ... critical section using both resources ...
    // Both unlocked when unique_locks go out of scope
}
```

**Prevention Strategy 3: std::scoped_lock (C++17) — simplest approach**

```cpp
void safest_transfer() {
    scoped_lock lock(mtx_a, mtx_b);  // locks both, deadlock-free
    // ... work ...
}
```

**Prevention Strategy 4: Lock hierarchy**

Assign a numeric level to each mutex. A thread can only lock a mutex with a LOWER level than the lowest mutex it currently holds.

```cpp
// Conceptual hierarchy:
// Level 3: database_mutex
// Level 2: cache_mutex
// Level 1: log_mutex
// Rule: always lock higher levels first

// GOOD: database (3) → cache (2) → log (1)
// BAD:  log (1) → database (3) — violates hierarchy
```

**Deadlock conditions (all four must hold):**
1. **Mutual exclusion** — resource held exclusively
2. **Hold and wait** — thread holds one resource while waiting for another
3. **No preemption** — resources can't be forcibly taken
4. **Circular wait** — circular chain of threads waiting for each other

Breaking any one condition prevents deadlock.

---

### std::condition_variable (wait, notify_one, notify_all)

A **condition variable** allows threads to wait for a specific condition to become true. It's used with a mutex to avoid busy-waiting (spinning in a loop checking a condition).

**The pattern:**
1. **Waiting thread:** locks a mutex, checks a condition, and if false, waits (releasing the mutex)
2. **Notifying thread:** locks the mutex, changes the shared state, notifies the waiting thread

```cpp
#include <mutex>
#include <condition_variable>
#include <queue>
#include <thread>
#include <iostream>
using namespace std;

mutex mtx;
condition_variable cv;
queue<int> data_queue;
bool finished = false;

// Producer: generates data and notifies consumer
void producer() {
    for (int i = 0; i < 10; i++) {
        {
            lock_guard<mutex> lock(mtx);
            data_queue.push(i);
            cout << "Produced: " << i << endl;
        }
        cv.notify_one();  // wake up one waiting consumer
        this_thread::sleep_for(chrono::milliseconds(100));
    }

    {
        lock_guard<mutex> lock(mtx);
        finished = true;
    }
    cv.notify_all();  // wake up ALL waiting consumers
}

// Consumer: waits for data and processes it
void consumer() {
    while (true) {
        unique_lock<mutex> lock(mtx);

        // Wait until there's data OR producer is finished
        cv.wait(lock, [] {
            return !data_queue.empty() || finished;
        });

        // Process all available data
        while (!data_queue.empty()) {
            int value = data_queue.front();
            data_queue.pop();
            cout << "Consumed: " << value << endl;
        }

        if (finished && data_queue.empty()) break;
    }
}

int main() {
    thread prod(producer);
    thread cons(consumer);

    prod.join();
    cons.join();
}
```

**How cv.wait() works internally:**

```cpp
// cv.wait(lock, predicate) is equivalent to:
while (!predicate()) {
    cv.wait(lock);
}

// cv.wait(lock) does:
// 1. Atomically releases the mutex and suspends the thread
// 2. When notified (or spuriously woken), re-acquires the mutex
// 3. Returns (caller should re-check the condition)
```

**notify_one vs notify_all:**

| Method | Behavior | When to use |
|--------|----------|-------------|
| `notify_one()` | Wakes up ONE waiting thread | When only one thread can make progress (e.g., one item in queue) |
| `notify_all()` | Wakes up ALL waiting threads | When multiple threads might be able to proceed, or when the condition changes for everyone |

```cpp
// notify_one: efficient when one consumer can handle the work
void producer_single() {
    {
        lock_guard<mutex> lock(mtx);
        data_queue.push(42);
    }
    cv.notify_one();  // wake one consumer
}

// notify_all: necessary when all threads need to re-evaluate
void shutdown() {
    {
        lock_guard<mutex> lock(mtx);
        finished = true;
    }
    cv.notify_all();  // wake ALL consumers so they can exit
}
```

**Timed waiting:**

```cpp
unique_lock<mutex> lock(mtx);

// Wait for up to 1 second
if (cv.wait_for(lock, chrono::seconds(1), [] { return !data_queue.empty(); })) {
    // Condition became true within 1 second
    int value = data_queue.front();
    data_queue.pop();
} else {
    // Timeout — condition still false after 1 second
    cout << "Timed out waiting for data" << endl;
}

// Wait until a specific time point
auto deadline = chrono::steady_clock::now() + chrono::seconds(5);
cv.wait_until(lock, deadline, [] { return !data_queue.empty(); });
```

---

### Spurious Wakeups

A **spurious wakeup** is when a thread waiting on a condition variable is woken up even though no thread called `notify_one()` or `notify_all()`. This can happen due to OS-level implementation details.

**This is why you ALWAYS use a predicate with wait():**

```cpp
// BAD: No predicate — vulnerable to spurious wakeups
cv.wait(lock);
// Thread might wake up even though the condition isn't true!
// If you proceed without checking, you'll process invalid state

// GOOD: Predicate re-checks the condition after every wakeup
cv.wait(lock, [] { return !data_queue.empty(); });
// If spuriously woken, the predicate returns false, and wait() goes back to sleep
// Only returns when the predicate is actually true
```

**The predicate form is equivalent to a while loop:**

```cpp
// These are equivalent:
cv.wait(lock, predicate);

// Same as:
while (!predicate()) {
    cv.wait(lock);
}
```

---

### Practical Example: Thread-Safe Bounded Buffer

```cpp
#include <mutex>
#include <condition_variable>
#include <queue>
using namespace std;

template <typename T>
class BoundedBuffer {
    queue<T> buffer;
    size_t capacity;
    mutex mtx;
    condition_variable not_full;
    condition_variable not_empty;

public:
    BoundedBuffer(size_t cap) : capacity(cap) {}

    void produce(const T& item) {
        unique_lock<mutex> lock(mtx);
        not_full.wait(lock, [this] { return buffer.size() < capacity; });

        buffer.push(item);

        lock.unlock();
        not_empty.notify_one();
    }

    T consume() {
        unique_lock<mutex> lock(mtx);
        not_empty.wait(lock, [this] { return !buffer.empty(); });

        T item = buffer.front();
        buffer.pop();

        lock.unlock();
        not_full.notify_one();

        return item;
    }

    size_t size() {
        lock_guard<mutex> lock(mtx);
        return buffer.size();
    }
};
```

---


## 9.3 Atomic Operations

> An **atomic operation** is an operation that completes in a single step relative to other threads — no other thread can observe the operation in a partially-completed state. `std::atomic<T>` provides lock-free (or at least thread-safe) operations on individual variables without needing a mutex.

---

### std::atomic\<T\>

`std::atomic` wraps a type and guarantees that reads and writes are atomic — no data races, no torn reads/writes.

```cpp
#include <atomic>
#include <thread>
#include <iostream>
using namespace std;

atomic<int> counter(0);  // atomic integer, initialized to 0

void increment(int times) {
    for (int i = 0; i < times; i++) {
        counter++;  // atomic increment — thread-safe without a mutex!
        // Equivalent to: counter.fetch_add(1);
    }
}

int main() {
    thread t1(increment, 100000);
    thread t2(increment, 100000);

    t1.join();
    t2.join();

    cout << "Counter: " << counter << endl;  // always 200000
}
```

**Why is `counter++` on a regular `int` not safe?**

`counter++` on a non-atomic `int` is actually three operations:
1. **Read** the current value from memory
2. **Increment** the value in a register
3. **Write** the new value back to memory

Two threads can interleave these steps, causing lost updates. `std::atomic` ensures the entire read-modify-write happens as one indivisible operation.

**Common atomic operations:**

```cpp
atomic<int> x(0);

// Load and store
int val = x.load();          // atomic read
x.store(42);                 // atomic write

// Read-modify-write operations
x.fetch_add(5);              // atomically: x += 5, returns old value
x.fetch_sub(3);              // atomically: x -= 3, returns old value
x.fetch_and(0xFF);           // atomically: x &= 0xFF
x.fetch_or(0x01);            // atomically: x |= 0x01
x.fetch_xor(0x10);           // atomically: x ^= 0x10

// Operator overloads (convenience)
x++;                         // fetch_add(1)
x--;                         // fetch_sub(1)
x += 10;                     // fetch_add(10)
x -= 5;                      // fetch_sub(5)

// Exchange
int old = x.exchange(100);   // atomically: set x to 100, return old value
```

**Atomic with other types:**

```cpp
atomic<bool> flag(false);
atomic<double> temperature(0.0);  // may not be lock-free on all platforms
atomic<int*> ptr(nullptr);

// Check if the atomic is lock-free (hardware-supported)
cout << "int is lock-free: " << atomic<int>{}.is_lock_free() << endl;
cout << "double is lock-free: " << atomic<double>{}.is_lock_free() << endl;

// atomic_flag — guaranteed lock-free boolean
atomic_flag spinlock = ATOMIC_FLAG_INIT;
```

**Atomic flag — the simplest atomic type:**

```cpp
// atomic_flag is the ONLY type guaranteed to be lock-free on all platforms
atomic_flag lock_flag = ATOMIC_FLAG_INIT;

void spinlock_acquire() {
    while (lock_flag.test_and_set(memory_order_acquire)) {
        // Spin — busy wait until the flag is cleared
        // test_and_set: atomically sets flag to true, returns previous value
        // If previous was true → someone else holds the lock → keep spinning
        // If previous was false → we just acquired the lock → exit loop
    }
}

void spinlock_release() {
    lock_flag.clear(memory_order_release);
}
```

---

### Memory Ordering (relaxed, acquire, release, seq_cst)

Memory ordering controls how atomic operations interact with non-atomic memory accesses. Modern CPUs and compilers can **reorder** memory operations for performance. Memory ordering constraints tell the hardware and compiler what reorderings are allowed.

**Why does this matter?**

```cpp
// Without proper memory ordering, Thread 2 might see x == 0 even after
// Thread 1 sets ready to true, because the CPU/compiler reordered the writes

// Thread 1:
x = 42;           // (A)
ready = true;     // (B) — might be reordered BEFORE (A) by the CPU!

// Thread 2:
if (ready) {      // (C)
    assert(x == 42);  // (D) — might FAIL if (A) and (B) were reordered!
}
```

**Memory ordering options (from weakest to strongest):**

**1. `memory_order_relaxed` — no ordering guarantees**

Only guarantees atomicity. No synchronization with other threads. Operations can be freely reordered.

```cpp
atomic<int> counter(0);

void relaxed_increment() {
    // Only need atomicity, don't care about ordering relative to other variables
    counter.fetch_add(1, memory_order_relaxed);
}

// Use case: simple counters, statistics where exact ordering doesn't matter
```

**2. `memory_order_acquire` / `memory_order_release` — producer-consumer synchronization**

- **Release:** All writes BEFORE this store are visible to any thread that acquires the same atomic variable. (Prevents reordering of earlier writes past this point.)
- **Acquire:** All reads AFTER this load see the writes that happened before the corresponding release. (Prevents reordering of later reads before this point.)

```cpp
atomic<bool> ready(false);
int data = 0;

// Producer thread
void producer() {
    data = 42;                                    // (A) write data
    ready.store(true, memory_order_release);      // (B) release — guarantees (A) is visible
}

// Consumer thread
void consumer() {
    while (!ready.load(memory_order_acquire)) {}  // (C) acquire — synchronizes with (B)
    assert(data == 42);                           // (D) guaranteed to see data == 42
}

// The acquire-release pair creates a "happens-before" relationship:
// (A) happens-before (B) [program order + release]
// (B) synchronizes-with (C) [release-acquire on 'ready']
// (C) happens-before (D) [program order + acquire]
// Therefore: (A) happens-before (D) → data == 42 is guaranteed
```

**3. `memory_order_acq_rel` — both acquire and release**

Used for read-modify-write operations that both read and write.

```cpp
atomic<int> sync_counter(0);

void read_modify_write() {
    // fetch_add is both a read and a write
    // acq_rel ensures:
    // - All prior writes are visible (release)
    // - All subsequent reads see the latest values (acquire)
    sync_counter.fetch_add(1, memory_order_acq_rel);
}
```

**4. `memory_order_seq_cst` — sequential consistency (default)**

The strongest ordering. All `seq_cst` operations appear to execute in a single total order agreed upon by all threads. This is the default for all atomic operations.

```cpp
atomic<int> x(0), y(0);

// Thread 1
void thread1() {
    x.store(1, memory_order_seq_cst);  // default
}

// Thread 2
void thread2() {
    y.store(1, memory_order_seq_cst);
}

// Thread 3
void thread3() {
    while (x.load() == 0) {}
    if (y.load() == 0) {
        // With seq_cst: if we see x == 1, and thread2 ran after thread1,
        // then y must also be 1. Total order is guaranteed.
    }
}
```

**Memory ordering summary:**

| Ordering | Guarantees | Performance | Use Case |
|----------|-----------|-------------|----------|
| `relaxed` | Atomicity only | Fastest | Counters, statistics |
| `acquire` | Reads after this see prior writes | Fast | Consumer side of producer-consumer |
| `release` | Writes before this are visible | Fast | Producer side of producer-consumer |
| `acq_rel` | Both acquire and release | Medium | Read-modify-write operations |
| `seq_cst` | Total ordering across all threads | Slowest | Default, when in doubt |

**Rule of thumb:** Use `seq_cst` (the default) unless you have a proven performance need and deeply understand the memory model. Incorrect use of weaker orderings causes subtle, hard-to-reproduce bugs.

---

### Compare-and-Swap (CAS)

**Compare-and-swap** (CAS) is the fundamental building block of lock-free programming. It atomically:
1. Compares the current value with an expected value
2. If they match, replaces the current value with a new value
3. If they don't match, loads the current value into the expected variable

```cpp
atomic<int> value(0);

void cas_example() {
    int expected = 0;
    int desired = 42;

    // compare_exchange_strong:
    // If value == expected (0), set value = desired (42), return true
    // If value != expected, set expected = current value, return false
    bool success = value.compare_exchange_strong(expected, desired);

    if (success) {
        cout << "CAS succeeded: value is now " << desired << endl;
    } else {
        cout << "CAS failed: value was " << expected << ", not 0" << endl;
    }
}
```

**compare_exchange_strong vs compare_exchange_weak:**

```cpp
// strong: guaranteed to succeed if value == expected
// Only fails if value != expected
bool success = value.compare_exchange_strong(expected, desired);

// weak: may SPURIOUSLY fail even if value == expected
// More efficient on some architectures (ARM, PowerPC)
// Must be used in a loop
bool success = value.compare_exchange_weak(expected, desired);
```

**CAS loop pattern — the foundation of lock-free algorithms:**

```cpp
atomic<int> counter(0);

void lock_free_increment() {
    int old_val = counter.load();
    while (!counter.compare_exchange_weak(old_val, old_val + 1)) {
        // CAS failed — another thread modified counter
        // old_val is updated to the current value
        // Loop retries with the new value
    }
}

// This is exactly what fetch_add does internally on many platforms
```

**Practical CAS example — lock-free stack push:**

```cpp
template <typename T>
struct Node {
    T data;
    Node* next;
    Node(const T& d) : data(d), next(nullptr) {}
};

template <typename T>
class LockFreeStack {
    atomic<Node<T>*> head{nullptr};

public:
    void push(const T& data) {
        Node<T>* new_node = new Node<T>(data);
        new_node->next = head.load();

        // CAS loop: try to set head to new_node
        // If head changed since we read it, retry
        while (!head.compare_exchange_weak(new_node->next, new_node)) {
            // new_node->next is updated to current head automatically
        }
    }

    bool pop(T& result) {
        Node<T>* old_head = head.load();

        while (old_head && !head.compare_exchange_weak(old_head, old_head->next)) {
            // old_head is updated to current head automatically
        }

        if (old_head) {
            result = old_head->data;
            delete old_head;  // simplified — real lock-free code needs hazard pointers or epoch-based reclamation
            return true;
        }
        return false;
    }
};
```

---

### Lock-Free Programming Basics

**Lock-free** means that at least one thread is guaranteed to make progress in a finite number of steps, even if other threads are suspended. No thread can block another indefinitely.

**Levels of non-blocking guarantees:**

| Guarantee | Definition | Example |
|-----------|-----------|---------|
| **Wait-free** | Every thread completes in bounded steps | `atomic::fetch_add` on most hardware |
| **Lock-free** | At least one thread makes progress | CAS-based algorithms |
| **Obstruction-free** | A thread makes progress if no other thread is active | Weakest non-blocking guarantee |
| **Blocking** | Threads can be blocked indefinitely | Mutex-based code |

```cpp
// Lock-free counter (wait-free on most platforms)
atomic<long long> ops_counter(0);

void record_operation() {
    ops_counter.fetch_add(1, memory_order_relaxed);
    // This is wait-free: completes in a fixed number of CPU instructions
    // No loops, no retries, no possibility of blocking
}

// Lock-free queue (simplified single-producer, single-consumer)
template <typename T, size_t Size>
class SPSCQueue {
    array<T, Size> buffer;
    atomic<size_t> read_pos{0};
    atomic<size_t> write_pos{0};

public:
    bool push(const T& item) {
        size_t write = write_pos.load(memory_order_relaxed);
        size_t next_write = (write + 1) % Size;

        if (next_write == read_pos.load(memory_order_acquire)) {
            return false;  // full
        }

        buffer[write] = item;
        write_pos.store(next_write, memory_order_release);
        return true;
    }

    bool pop(T& item) {
        size_t read = read_pos.load(memory_order_relaxed);

        if (read == write_pos.load(memory_order_acquire)) {
            return false;  // empty
        }

        item = buffer[read];
        read_pos.store((read + 1) % Size, memory_order_release);
        return true;
    }
};
```

**When to use lock-free programming:**
- High-contention scenarios where mutex overhead is significant
- Real-time systems where blocking is unacceptable
- Simple data structures (counters, flags, single-producer/single-consumer queues)

**When NOT to use lock-free programming:**
- Complex data structures (use mutexes — correctness is more important)
- When you don't have proven performance problems with mutexes
- When the team doesn't deeply understand memory ordering
- Lock-free code is extremely hard to get right and even harder to debug

---


## 9.4 Async Programming

> Asynchronous programming in C++ allows you to launch tasks that run concurrently and retrieve their results later. The `<future>` header provides `std::async`, `std::future`, `std::promise`, `std::packaged_task`, and `std::shared_future` — higher-level abstractions over raw threads.

---

### std::async, std::future, std::promise

**std::async — the simplest way to run a task asynchronously:**

`std::async` launches a function (potentially in a new thread) and returns a `std::future` that will hold the result.

```cpp
#include <future>
#include <iostream>
#include <chrono>
using namespace std;

// A function that takes time to compute
int expensive_computation(int x) {
    this_thread::sleep_for(chrono::seconds(2));
    return x * x;
}

int main() {
    // Launch asynchronously — may run in a new thread
    future<int> result = async(launch::async, expensive_computation, 42);

    // Main thread continues doing other work...
    cout << "Computing in background..." << endl;
    this_thread::sleep_for(chrono::seconds(1));
    cout << "Still working on main thread..." << endl;

    // Get the result — blocks if not ready yet
    int value = result.get();  // blocks until the async task completes
    cout << "Result: " << value << endl;  // 1764

    // IMPORTANT: get() can only be called ONCE on a future
    // Calling get() again throws std::future_error
}
```

**Launch policies:**

```cpp
// launch::async — guaranteed to run in a new thread
future<int> f1 = async(launch::async, expensive_computation, 10);

// launch::deferred — lazy evaluation, runs when get() or wait() is called
// Runs in the CALLING thread, not a new thread
future<int> f2 = async(launch::deferred, expensive_computation, 20);
// Nothing happens yet...
int val = f2.get();  // NOW it runs, in the current thread

// Default (launch::async | launch::deferred) — implementation decides
future<int> f3 = async(expensive_computation, 30);
// The runtime chooses whether to launch a new thread or defer
```

**Checking if a future is ready:**

```cpp
future<int> result = async(launch::async, expensive_computation, 42);

// Non-blocking check
auto status = result.wait_for(chrono::milliseconds(0));
if (status == future_status::ready) {
    cout << "Result ready: " << result.get() << endl;
} else if (status == future_status::timeout) {
    cout << "Still computing..." << endl;
} else if (status == future_status::deferred) {
    cout << "Task is deferred (hasn't started)" << endl;
}

// Wait with timeout
if (result.wait_for(chrono::seconds(5)) == future_status::ready) {
    cout << "Got result: " << result.get() << endl;
} else {
    cout << "Timed out after 5 seconds" << endl;
}
```

**Multiple async tasks:**

```cpp
int main() {
    // Launch multiple tasks in parallel
    auto f1 = async(launch::async, [] { return download_file("file1.txt"); });
    auto f2 = async(launch::async, [] { return download_file("file2.txt"); });
    auto f3 = async(launch::async, [] { return download_file("file3.txt"); });

    // All three downloads happen concurrently
    // Collect results (blocks on each if not ready)
    string r1 = f1.get();
    string r2 = f2.get();
    string r3 = f3.get();

    cout << "All downloads complete" << endl;
}
```

**std::promise — manually set a future's value from another thread:**

A `promise` is the "write" end, and a `future` is the "read" end of a one-time communication channel between threads.

```cpp
#include <future>
#include <thread>
using namespace std;

void compute(promise<int> prom) {
    try {
        // Do some work...
        this_thread::sleep_for(chrono::seconds(1));
        int result = 42;

        // Set the result — the associated future becomes ready
        prom.set_value(result);
    } catch (...) {
        // If an exception occurs, propagate it to the future
        prom.set_exception(current_exception());
    }
}

int main() {
    promise<int> prom;
    future<int> fut = prom.get_future();  // get the associated future

    // Launch thread with the promise
    thread t(compute, std::move(prom));  // promise is move-only

    // Wait for the result
    try {
        int result = fut.get();  // blocks until promise is fulfilled
        cout << "Result: " << result << endl;
    } catch (const exception& e) {
        cout << "Exception: " << e.what() << endl;
    }

    t.join();
}
```

**Promise with exception propagation:**

```cpp
void risky_computation(promise<double> prom) {
    try {
        double result = 1.0 / 0.0;  // some computation
        if (isinf(result)) {
            throw runtime_error("Division by zero");
        }
        prom.set_value(result);
    } catch (...) {
        prom.set_exception(current_exception());
        // The exception is stored in the future
        // When the consumer calls get(), the exception is re-thrown
    }
}

int main() {
    promise<double> prom;
    future<double> fut = prom.get_future();

    thread t(risky_computation, std::move(prom));

    try {
        double val = fut.get();  // re-throws the exception from the other thread!
    } catch (const runtime_error& e) {
        cout << "Caught from other thread: " << e.what() << endl;
    }

    t.join();
}
```

---

### std::packaged_task

A `packaged_task` wraps a callable and provides a `future` for its return value. Unlike `std::async`, it doesn't automatically launch a thread — you control when and where it runs.

```cpp
#include <future>
#include <thread>
#include <functional>
using namespace std;

int add(int a, int b) {
    this_thread::sleep_for(chrono::seconds(1));
    return a + b;
}

int main() {
    // Wrap the function in a packaged_task
    packaged_task<int(int, int)> task(add);

    // Get the future BEFORE moving the task
    future<int> result = task.get_future();

    // Run the task in a thread (must move — packaged_task is not copyable)
    thread t(std::move(task), 3, 4);

    // Get the result
    cout << "Result: " << result.get() << endl;  // 7

    t.join();
}
```

**packaged_task with a task queue:**

```cpp
#include <queue>
#include <mutex>
#include <future>
using namespace std;

class TaskQueue {
    queue<packaged_task<void()>> tasks;
    mutex mtx;

public:
    // Submit a task and get a future for its result
    template <typename F>
    future<typename result_of<F()>::type> submit(F func) {
        using ResultType = typename result_of<F()>::type;

        packaged_task<ResultType()> task(std::move(func));
        future<ResultType> result = task.get_future();

        {
            lock_guard<mutex> lock(mtx);
            // Wrap in a void() task for uniform storage
            tasks.push(packaged_task<void()>(std::move(task)));
        }

        return result;
    }

    // Execute the next task in the queue
    void execute_next() {
        packaged_task<void()> task;
        {
            lock_guard<mutex> lock(mtx);
            if (tasks.empty()) return;
            task = std::move(tasks.front());
            tasks.pop();
        }
        task();  // execute the task
    }
};

int main() {
    TaskQueue queue;

    auto f1 = queue.submit([] { return 42; });
    auto f2 = queue.submit([] { return string("hello"); });

    queue.execute_next();  // runs first task
    queue.execute_next();  // runs second task

    cout << f1.get() << endl;  // 42
    cout << f2.get() << endl;  // "hello"
}
```

**Comparison: async vs promise vs packaged_task:**

| Feature | `std::async` | `std::promise` | `std::packaged_task` |
|---------|-------------|---------------|---------------------|
| Creates thread? | Optionally (launch policy) | No | No |
| Who sets the value? | Automatically (return value) | Manually (`set_value`) | Automatically (return value) |
| When does it run? | Immediately or deferred | When you call `set_value` | When you invoke `operator()` |
| Use case | Simple async tasks | Manual inter-thread communication | Task queues, deferred execution |
| Flexibility | Low | High | Medium |

---

### std::shared_future

A regular `std::future` can only be read once (`get()` moves the value out). A `std::shared_future` allows multiple threads to wait for and read the same result.

```cpp
#include <future>
#include <thread>
#include <vector>
using namespace std;

int main() {
    // Create a shared_future from a regular future
    shared_future<int> shared = async(launch::async, [] {
        this_thread::sleep_for(chrono::seconds(2));
        return 42;
    }).share();  // .share() converts future to shared_future

    // Multiple threads can wait for the same result
    vector<thread> threads;
    for (int i = 0; i < 5; i++) {
        threads.emplace_back([shared, i] {
            // Each thread can call get() — all get the same value
            int result = shared.get();
            cout << "Thread " << i << " got: " << result << endl;
        });
    }

    for (auto& t : threads) {
        t.join();
    }
}
```

**shared_future use cases:**
- Broadcasting a result to multiple waiting threads
- One-time initialization that multiple threads depend on
- Barrier-like synchronization where all threads wait for a signal

```cpp
// Example: Multiple threads waiting for configuration to be loaded
shared_future<Config> config_future;

void worker(shared_future<Config> config) {
    // Block until config is ready
    const Config& cfg = config.get();
    // All workers see the same config
    cout << "Worker using config: " << cfg.name << endl;
}

int main() {
    promise<Config> config_promise;
    config_future = config_promise.get_future().share();

    // Launch workers — they'll all wait for config
    vector<thread> workers;
    for (int i = 0; i < 10; i++) {
        workers.emplace_back(worker, config_future);
    }

    // Load config (simulated)
    this_thread::sleep_for(chrono::seconds(1));
    config_promise.set_value(Config{"production", 8080});
    // All 10 workers unblock and proceed with the same config

    for (auto& w : workers) {
        w.join();
    }
}
```

---


## 9.5 Classic Concurrency Problems

> These are well-known problems in concurrent programming that illustrate common challenges: synchronization, deadlock avoidance, resource sharing, and fairness. Understanding these problems and their solutions is essential for interviews and real-world concurrent system design.

---

### Producer-Consumer Problem

The **Producer-Consumer** (or Bounded Buffer) problem involves one or more producers generating data and placing it into a shared buffer, and one or more consumers removing data from the buffer. The challenge is to synchronize access so that:
- Producers don't add to a full buffer
- Consumers don't remove from an empty buffer
- No data races occur

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <iostream>
using namespace std;

template <typename T>
class ProducerConsumerQueue {
    queue<T> buffer;
    size_t max_size;
    mutex mtx;
    condition_variable not_full;   // signaled when buffer is not full
    condition_variable not_empty;  // signaled when buffer is not empty
    bool done = false;             // signals producers are finished

public:
    ProducerConsumerQueue(size_t max) : max_size(max) {}

    void produce(const T& item) {
        unique_lock<mutex> lock(mtx);
        not_full.wait(lock, [this] { return buffer.size() < max_size; });

        buffer.push(item);
        cout << "[Producer] Produced: " << item
             << " (buffer size: " << buffer.size() << ")" << endl;

        lock.unlock();
        not_empty.notify_one();
    }

    bool consume(T& item) {
        unique_lock<mutex> lock(mtx);
        not_empty.wait(lock, [this] { return !buffer.empty() || done; });

        if (buffer.empty() && done) return false;  // no more items

        item = buffer.front();
        buffer.pop();
        cout << "[Consumer] Consumed: " << item
             << " (buffer size: " << buffer.size() << ")" << endl;

        lock.unlock();
        not_full.notify_one();
        return true;
    }

    void finish() {
        {
            lock_guard<mutex> lock(mtx);
            done = true;
        }
        not_empty.notify_all();  // wake all consumers so they can exit
    }
};

int main() {
    ProducerConsumerQueue<int> queue(5);  // buffer capacity of 5

    // Producer thread
    thread producer([&queue] {
        for (int i = 1; i <= 20; i++) {
            queue.produce(i);
            this_thread::sleep_for(chrono::milliseconds(50));
        }
        queue.finish();
    });

    // Consumer threads
    thread consumer1([&queue] {
        int item;
        while (queue.consume(item)) {
            this_thread::sleep_for(chrono::milliseconds(100));  // simulate processing
        }
    });

    thread consumer2([&queue] {
        int item;
        while (queue.consume(item)) {
            this_thread::sleep_for(chrono::milliseconds(150));
        }
    });

    producer.join();
    consumer1.join();
    consumer2.join();

    cout << "All done!" << endl;
}
```

---

### Reader-Writer Problem

Multiple threads need to access a shared resource. **Readers** can access concurrently (they don't modify data), but **writers** need exclusive access. The challenge is to maximize concurrency while ensuring correctness.

**Three variants:**
1. **Readers-preference:** Readers never wait if the resource is being read (writers may starve)
2. **Writers-preference:** Writers get priority (readers may starve)
3. **Fair:** Neither readers nor writers starve

```cpp
#include <shared_mutex>
#include <thread>
#include <iostream>
#include <vector>
using namespace std;

class SharedDatabase {
    mutable shared_mutex rw_mutex;  // C++17 shared_mutex
    int data = 0;

public:
    // Multiple readers can read simultaneously
    int read() const {
        shared_lock<shared_mutex> lock(rw_mutex);  // shared (read) lock
        cout << "[Reader " << this_thread::get_id() << "] Read: " << data << endl;
        this_thread::sleep_for(chrono::milliseconds(100));  // simulate read time
        return data;
    }

    // Only one writer at a time, no readers during write
    void write(int value) {
        unique_lock<shared_mutex> lock(rw_mutex);  // exclusive (write) lock
        cout << "[Writer " << this_thread::get_id() << "] Writing: " << value << endl;
        data = value;
        this_thread::sleep_for(chrono::milliseconds(200));  // simulate write time
    }
};

int main() {
    SharedDatabase db;

    // Multiple reader threads
    vector<thread> readers;
    for (int i = 0; i < 5; i++) {
        readers.emplace_back([&db, i] {
            for (int j = 0; j < 3; j++) {
                db.read();
                this_thread::sleep_for(chrono::milliseconds(50));
            }
        });
    }

    // Writer threads
    vector<thread> writers;
    for (int i = 0; i < 2; i++) {
        writers.emplace_back([&db, i] {
            for (int j = 0; j < 3; j++) {
                db.write(i * 100 + j);
                this_thread::sleep_for(chrono::milliseconds(300));
            }
        });
    }

    for (auto& r : readers) r.join();
    for (auto& w : writers) w.join();
}
```

**Manual implementation (without shared_mutex):**

```cpp
class ReadWriteLock {
    mutex mtx;
    condition_variable cv;
    int active_readers = 0;
    bool writer_active = false;
    int waiting_writers = 0;

public:
    void read_lock() {
        unique_lock<mutex> lock(mtx);
        // Wait if a writer is active or writers are waiting (writers-preference)
        cv.wait(lock, [this] { return !writer_active && waiting_writers == 0; });
        active_readers++;
    }

    void read_unlock() {
        unique_lock<mutex> lock(mtx);
        active_readers--;
        if (active_readers == 0) {
            cv.notify_all();  // wake waiting writers
        }
    }

    void write_lock() {
        unique_lock<mutex> lock(mtx);
        waiting_writers++;
        cv.wait(lock, [this] { return !writer_active && active_readers == 0; });
        waiting_writers--;
        writer_active = true;
    }

    void write_unlock() {
        unique_lock<mutex> lock(mtx);
        writer_active = false;
        cv.notify_all();  // wake waiting readers and writers
    }
};
```

---

### Dining Philosophers Problem

Five philosophers sit around a table. Each philosopher alternates between thinking and eating. To eat, a philosopher needs two forks (one on each side). The challenge is to prevent **deadlock** (all philosophers pick up their left fork and wait for the right) and **starvation** (a philosopher never gets to eat).

```cpp
#include <thread>
#include <mutex>
#include <iostream>
#include <vector>
#include <chrono>
using namespace std;

const int NUM_PHILOSOPHERS = 5;
mutex forks[NUM_PHILOSOPHERS];

// SOLUTION 1: Resource ordering — always pick up the lower-numbered fork first
void philosopher(int id) {
    int left = id;
    int right = (id + 1) % NUM_PHILOSOPHERS;

    // Always lock the lower-numbered fork first to prevent deadlock
    int first = min(left, right);
    int second = max(left, right);

    for (int meal = 0; meal < 3; meal++) {
        // Think
        cout << "Philosopher " << id << " is thinking" << endl;
        this_thread::sleep_for(chrono::milliseconds(100 + rand() % 200));

        // Pick up forks (in consistent order)
        lock_guard<mutex> lock1(forks[first]);
        lock_guard<mutex> lock2(forks[second]);

        // Eat
        cout << "Philosopher " << id << " is eating (meal " << meal + 1 << ")" << endl;
        this_thread::sleep_for(chrono::milliseconds(100 + rand() % 200));

        // Forks are released when lock_guards go out of scope
        cout << "Philosopher " << id << " finished eating" << endl;
    }
}

int main() {
    vector<thread> philosophers;
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        philosophers.emplace_back(philosopher, i);
    }
    for (auto& p : philosophers) {
        p.join();
    }
    cout << "Dinner is over!" << endl;
}
```

**SOLUTION 2: Using std::lock to avoid deadlock:**

```cpp
void philosopher_v2(int id) {
    int left = id;
    int right = (id + 1) % NUM_PHILOSOPHERS;

    for (int meal = 0; meal < 3; meal++) {
        // Think
        this_thread::sleep_for(chrono::milliseconds(100));

        // Lock both forks atomically — no deadlock possible
        scoped_lock lock(forks[left], forks[right]);

        // Eat
        cout << "Philosopher " << id << " is eating" << endl;
        this_thread::sleep_for(chrono::milliseconds(100));
    }
}
```

**SOLUTION 3: Limit concurrency — allow at most N-1 philosophers to try eating:**

```cpp
#include <semaphore>  // C++20

counting_semaphore<NUM_PHILOSOPHERS - 1> table_seats(NUM_PHILOSOPHERS - 1);

void philosopher_v3(int id) {
    int left = id;
    int right = (id + 1) % NUM_PHILOSOPHERS;

    for (int meal = 0; meal < 3; meal++) {
        this_thread::sleep_for(chrono::milliseconds(100));  // think

        table_seats.acquire();  // at most 4 philosophers can try to eat

        lock_guard<mutex> lock1(forks[left]);
        lock_guard<mutex> lock2(forks[right]);

        cout << "Philosopher " << id << " is eating" << endl;
        this_thread::sleep_for(chrono::milliseconds(100));

        table_seats.release();
    }
}
```

---

### Sleeping Barber Problem

A barber shop has one barber, one barber chair, and N waiting chairs. If there are no customers, the barber sleeps. When a customer arrives, they wake the barber if sleeping, or sit in a waiting chair if the barber is busy. If all waiting chairs are full, the customer leaves.

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <iostream>
using namespace std;

class BarberShop {
    int num_waiting_chairs;
    queue<int> waiting_customers;
    mutex mtx;
    condition_variable barber_cv;    // barber waits for customers
    condition_variable customer_cv;  // customer waits for haircut
    bool barber_sleeping = true;
    bool shop_open = true;
    int current_customer = -1;
    bool haircut_done = false;

public:
    BarberShop(int chairs) : num_waiting_chairs(chairs) {}

    // Called by customer threads
    bool enter_shop(int customer_id) {
        unique_lock<mutex> lock(mtx);

        if (!shop_open) return false;

        if ((int)waiting_customers.size() >= num_waiting_chairs) {
            cout << "Customer " << customer_id << " leaves (no seats)" << endl;
            return false;  // no waiting chairs available
        }

        waiting_customers.push(customer_id);
        cout << "Customer " << customer_id << " sits down (waiting: "
             << waiting_customers.size() << ")" << endl;

        // Wake the barber if sleeping
        if (barber_sleeping) {
            barber_cv.notify_one();
        }

        // Wait for this customer's haircut to be done
        customer_cv.wait(lock, [this, customer_id] {
            return current_customer == customer_id && haircut_done;
        });

        cout << "Customer " << customer_id << " leaves with a fresh haircut!" << endl;
        haircut_done = false;
        return true;
    }

    // Called by barber thread
    void barber_work() {
        while (true) {
            unique_lock<mutex> lock(mtx);

            // Sleep if no customers
            barber_sleeping = true;
            barber_cv.wait(lock, [this] {
                return !waiting_customers.empty() || !shop_open;
            });
            barber_sleeping = false;

            if (!shop_open && waiting_customers.empty()) break;

            // Get next customer
            current_customer = waiting_customers.front();
            waiting_customers.pop();

            cout << "Barber cutting hair of customer " << current_customer << endl;
            lock.unlock();

            // Cut hair (takes time)
            this_thread::sleep_for(chrono::milliseconds(200));

            // Signal customer that haircut is done
            lock.lock();
            haircut_done = true;
            customer_cv.notify_all();
        }
        cout << "Barber goes home" << endl;
    }

    void close_shop() {
        {
            lock_guard<mutex> lock(mtx);
            shop_open = false;
        }
        barber_cv.notify_one();
    }
};

int main() {
    BarberShop shop(3);  // 3 waiting chairs

    thread barber([&shop] { shop.barber_work(); });

    // Customers arrive at random intervals
    vector<thread> customers;
    for (int i = 1; i <= 10; i++) {
        customers.emplace_back([&shop, i] {
            this_thread::sleep_for(chrono::milliseconds(i * 80));
            shop.enter_shop(i);
        });
    }

    for (auto& c : customers) c.join();

    shop.close_shop();
    barber.join();
}
```

---

### Bounded Buffer Problem

The Bounded Buffer is a generalization of the Producer-Consumer problem with a fixed-size buffer. This is the same as the `BoundedBuffer` class shown in section 9.2, but here's a version using semaphores (C++20):

```cpp
#include <semaphore>  // C++20
#include <mutex>
#include <thread>
#include <array>
#include <iostream>
using namespace std;

template <typename T, size_t N>
class BoundedBuffer {
    array<T, N> buffer;
    size_t read_pos = 0;
    size_t write_pos = 0;
    mutex mtx;
    counting_semaphore<N> empty_slots{N};  // initially N empty slots
    counting_semaphore<N> full_slots{0};   // initially 0 full slots

public:
    void produce(const T& item) {
        empty_slots.acquire();  // wait for an empty slot (decrements)

        {
            lock_guard<mutex> lock(mtx);
            buffer[write_pos] = item;
            write_pos = (write_pos + 1) % N;
        }

        full_slots.release();  // signal that a slot is now full (increments)
    }

    T consume() {
        full_slots.acquire();  // wait for a full slot (decrements)

        T item;
        {
            lock_guard<mutex> lock(mtx);
            item = buffer[read_pos];
            read_pos = (read_pos + 1) % N;
        }

        empty_slots.release();  // signal that a slot is now empty (increments)
        return item;
    }
};

int main() {
    BoundedBuffer<int, 5> buffer;

    thread producer([&buffer] {
        for (int i = 0; i < 20; i++) {
            buffer.produce(i);
            cout << "Produced: " << i << endl;
        }
    });

    thread consumer([&buffer] {
        for (int i = 0; i < 20; i++) {
            int val = buffer.consume();
            cout << "Consumed: " << val << endl;
        }
    });

    producer.join();
    consumer.join();
}
```

---

### Summary of Classic Problems

| Problem | Key Challenge | Core Mechanism |
|---------|--------------|----------------|
| Producer-Consumer | Synchronize access to shared buffer | Condition variables or semaphores |
| Reader-Writer | Allow concurrent reads, exclusive writes | shared_mutex or manual read-write lock |
| Dining Philosophers | Prevent deadlock with circular resource dependency | Lock ordering, std::lock, or limiting concurrency |
| Sleeping Barber | Coordinate barber sleep/wake with customer arrivals | Condition variables |
| Bounded Buffer | Fixed-size buffer with blocking on full/empty | Semaphores or condition variables |

---


## 9.6 Thread Pool

> A **thread pool** is a collection of pre-created worker threads that wait for tasks to execute. Instead of creating and destroying threads for each task (which is expensive), tasks are submitted to the pool and executed by available threads. Thread pools are fundamental to high-performance servers, task schedulers, and parallel computation frameworks.

---

### Why Thread Pools?

**Problem with creating threads per task:**

```cpp
// BAD: Creating a new thread for every request
void handle_request(Request req) {
    thread t([req] {
        process(req);
    });
    t.detach();
    // Problems:
    // 1. Thread creation overhead (~1ms per thread on Linux)
    // 2. No limit on concurrent threads — can exhaust system resources
    // 3. Context switching overhead with thousands of threads
    // 4. No way to control or cancel tasks
    // 5. Detached threads are hard to manage
}
```

**Thread pool benefits:**
- **Reuse threads** — avoid creation/destruction overhead
- **Limit concurrency** — prevent resource exhaustion
- **Task queuing** — buffer work when all threads are busy
- **Graceful shutdown** — wait for in-progress tasks to complete
- **Better resource utilization** — match thread count to hardware

---

### Custom Thread Pool Implementation in C++

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <functional>
#include <vector>
#include <future>
#include <iostream>
using namespace std;

class ThreadPool {
    vector<thread> workers;
    queue<function<void()>> tasks;

    mutex queue_mutex;
    condition_variable condition;
    bool stop = false;

public:
    // Create a pool with the specified number of worker threads
    ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; i++) {
            workers.emplace_back([this] {
                worker_loop();
            });
        }
    }

    // Submit a task and get a future for its result
    template <typename F, typename... Args>
    auto submit(F&& func, Args&&... args) -> future<decltype(func(args...))> {
        using ReturnType = decltype(func(args...));

        // Wrap the task in a shared_ptr<packaged_task> so it can be stored in std::function
        auto task = make_shared<packaged_task<ReturnType()>>(
            bind(forward<F>(func), forward<Args>(args)...)
        );

        future<ReturnType> result = task->get_future();

        {
            lock_guard<mutex> lock(queue_mutex);
            if (stop) {
                throw runtime_error("Cannot submit to stopped ThreadPool");
            }
            tasks.push([task] { (*task)(); });
        }

        condition.notify_one();  // wake one worker
        return result;
    }

    // Graceful shutdown — finish all queued tasks, then stop
    ~ThreadPool() {
        {
            lock_guard<mutex> lock(queue_mutex);
            stop = true;
        }
        condition.notify_all();  // wake all workers so they can exit

        for (auto& worker : workers) {
            worker.join();
        }
    }

private:
    void worker_loop() {
        while (true) {
            function<void()> task;

            {
                unique_lock<mutex> lock(queue_mutex);
                condition.wait(lock, [this] {
                    return stop || !tasks.empty();
                });

                if (stop && tasks.empty()) return;  // exit the worker

                task = std::move(tasks.front());
                tasks.pop();
            }

            task();  // execute the task outside the lock
        }
    }
};

// Usage
int main() {
    ThreadPool pool(4);  // 4 worker threads

    // Submit various tasks
    vector<future<int>> results;

    for (int i = 0; i < 20; i++) {
        results.push_back(pool.submit([i] {
            cout << "Task " << i << " running on thread "
                 << this_thread::get_id() << endl;
            this_thread::sleep_for(chrono::milliseconds(100));
            return i * i;
        }));
    }

    // Collect results
    for (auto& result : results) {
        cout << "Result: " << result.get() << endl;
    }

    // ThreadPool destructor waits for all tasks to complete
}
```

---

### Task Queue with Condition Variables

The task queue is the heart of the thread pool. Workers block on the condition variable when the queue is empty, and are woken when new tasks arrive.

```cpp
// A more detailed look at the task queue mechanics

template <typename T>
class TaskQueue {
    queue<T> queue_;
    mutable mutex mutex_;
    condition_variable not_empty_;
    bool shutdown_ = false;

public:
    // Add a task to the queue
    void push(T task) {
        {
            lock_guard<mutex> lock(mutex_);
            if (shutdown_) throw runtime_error("Queue is shut down");
            queue_.push(std::move(task));
        }
        not_empty_.notify_one();
    }

    // Remove a task from the queue (blocks if empty)
    // Returns false if the queue is shut down and empty
    bool pop(T& task) {
        unique_lock<mutex> lock(mutex_);
        not_empty_.wait(lock, [this] {
            return !queue_.empty() || shutdown_;
        });

        if (queue_.empty()) return false;  // shutdown and empty

        task = std::move(queue_.front());
        queue_.pop();
        return true;
    }

    // Try to pop without blocking
    bool try_pop(T& task) {
        lock_guard<mutex> lock(mutex_);
        if (queue_.empty()) return false;
        task = std::move(queue_.front());
        queue_.pop();
        return true;
    }

    // Signal shutdown — workers will drain remaining tasks and exit
    void shutdown() {
        {
            lock_guard<mutex> lock(mutex_);
            shutdown_ = true;
        }
        not_empty_.notify_all();
    }

    size_t size() const {
        lock_guard<mutex> lock(mutex_);
        return queue_.size();
    }

    bool empty() const {
        lock_guard<mutex> lock(mutex_);
        return queue_.empty();
    }
};
```

**Priority task queue:**

```cpp
#include <queue>
#include <functional>

struct PrioritizedTask {
    int priority;  // higher = more important
    function<void()> task;

    bool operator<(const PrioritizedTask& other) const {
        return priority < other.priority;  // max-heap: highest priority first
    }
};

class PriorityThreadPool {
    vector<thread> workers;
    priority_queue<PrioritizedTask> tasks;
    mutex mtx;
    condition_variable cv;
    bool stop = false;

public:
    PriorityThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; i++) {
            workers.emplace_back([this] {
                while (true) {
                    PrioritizedTask task;
                    {
                        unique_lock<mutex> lock(mtx);
                        cv.wait(lock, [this] { return stop || !tasks.empty(); });
                        if (stop && tasks.empty()) return;
                        task = std::move(const_cast<PrioritizedTask&>(tasks.top()));
                        tasks.pop();
                    }
                    task.task();
                }
            });
        }
    }

    void submit(int priority, function<void()> task) {
        {
            lock_guard<mutex> lock(mtx);
            tasks.push({priority, std::move(task)});
        }
        cv.notify_one();
    }

    ~PriorityThreadPool() {
        { lock_guard<mutex> lock(mtx); stop = true; }
        cv.notify_all();
        for (auto& w : workers) w.join();
    }
};

// Usage
PriorityThreadPool pool(4);
pool.submit(1, [] { cout << "Low priority task" << endl; });
pool.submit(10, [] { cout << "HIGH priority task" << endl; });
pool.submit(5, [] { cout << "Medium priority task" << endl; });
// HIGH priority runs first, then medium, then low
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

**How work stealing works:**
1. Each worker has its own **deque** (double-ended queue) of tasks
2. New tasks are pushed to the **back** of the local deque
3. The worker pops tasks from the **back** (LIFO — good for cache locality)
4. When a worker's deque is empty, it **steals** from the **front** of another worker's deque (FIFO — steals oldest/largest tasks)

```cpp
// Conceptual work-stealing deque (simplified)
template <typename T>
class WorkStealingDeque {
    deque<T> tasks;
    mutex mtx;

public:
    // Owner pushes and pops from the back (fast path — often uncontended)
    void push(T task) {
        lock_guard<mutex> lock(mtx);
        tasks.push_back(std::move(task));
    }

    bool pop(T& task) {
        lock_guard<mutex> lock(mtx);
        if (tasks.empty()) return false;
        task = std::move(tasks.back());
        tasks.pop_back();
        return true;
    }

    // Thieves steal from the front (contended — but rare)
    bool steal(T& task) {
        lock_guard<mutex> lock(mtx);
        if (tasks.empty()) return false;
        task = std::move(tasks.front());
        tasks.pop_front();
        return true;
    }

    bool empty() const {
        lock_guard<mutex> lock(mtx);
        return tasks.empty();
    }
};
```

**Benefits of work stealing:**
- Reduces contention on the shared queue
- Better cache locality (workers process their own tasks)
- Automatic load balancing (idle workers steal from busy ones)
- Used in: Intel TBB, Java ForkJoinPool, Go runtime, Tokio (Rust)

**When to use work stealing:**
- CPU-bound tasks with many small work items
- Recursive divide-and-conquer algorithms (fork-join parallelism)
- When the basic shared-queue thread pool becomes a bottleneck

---

### Thread Pool Design Considerations

| Consideration | Options |
|--------------|---------|
| **Pool size** | `hardware_concurrency()` for CPU-bound; larger for I/O-bound |
| **Queue type** | Unbounded (risk OOM), bounded (backpressure), priority |
| **Rejection policy** | Block caller, throw exception, discard oldest, run in caller's thread |
| **Shutdown** | Graceful (finish queued tasks) vs immediate (discard pending) |
| **Task cancellation** | Via `std::future`, cancellation tokens, or atomic flags |
| **Exception handling** | Propagate via `std::future`, log and continue, or terminate |

---


## 9.7 Concurrent Data Structures

> Standard library containers (`vector`, `map`, `queue`, etc.) are NOT thread-safe. Concurrent access requires either external synchronization (mutexes) or purpose-built thread-safe data structures. This section covers common concurrent data structures and their implementation patterns.

---

### Thread-Safe Queue

A thread-safe queue is the most commonly needed concurrent data structure — used in thread pools, producer-consumer patterns, and message passing.

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <optional>
using namespace std;

template <typename T>
class ThreadSafeQueue {
    queue<T> queue_;
    mutable mutex mutex_;
    condition_variable cv_;

public:
    ThreadSafeQueue() = default;

    // Non-copyable (mutex is non-copyable)
    ThreadSafeQueue(const ThreadSafeQueue&) = delete;
    ThreadSafeQueue& operator=(const ThreadSafeQueue&) = delete;

    // Push an item (thread-safe)
    void push(T item) {
        {
            lock_guard<mutex> lock(mutex_);
            queue_.push(std::move(item));
        }
        cv_.notify_one();
    }

    // Blocking pop — waits until an item is available
    T pop() {
        unique_lock<mutex> lock(mutex_);
        cv_.wait(lock, [this] { return !queue_.empty(); });
        T item = std::move(queue_.front());
        queue_.pop();
        return item;
    }

    // Non-blocking try_pop — returns empty optional if queue is empty
    optional<T> try_pop() {
        lock_guard<mutex> lock(mutex_);
        if (queue_.empty()) return nullopt;
        T item = std::move(queue_.front());
        queue_.pop();
        return item;
    }

    // Timed pop — waits up to the specified duration
    optional<T> pop_for(chrono::milliseconds timeout) {
        unique_lock<mutex> lock(mutex_);
        if (!cv_.wait_for(lock, timeout, [this] { return !queue_.empty(); })) {
            return nullopt;  // timeout
        }
        T item = std::move(queue_.front());
        queue_.pop();
        return item;
    }

    bool empty() const {
        lock_guard<mutex> lock(mutex_);
        return queue_.empty();
    }

    size_t size() const {
        lock_guard<mutex> lock(mutex_);
        return queue_.size();
    }
};

// Usage
ThreadSafeQueue<int> tsq;

// Producer thread
thread producer([&tsq] {
    for (int i = 0; i < 100; i++) {
        tsq.push(i);
    }
});

// Consumer thread
thread consumer([&tsq] {
    for (int i = 0; i < 100; i++) {
        int val = tsq.pop();  // blocks if empty
        cout << "Got: " << val << endl;
    }
});

producer.join();
consumer.join();
```

**Fine-grained locking queue (separate head and tail locks):**

For higher throughput, use separate locks for the head and tail of the queue so producers and consumers don't contend with each other.

```cpp
template <typename T>
class FineGrainedQueue {
    struct Node {
        shared_ptr<T> data;
        unique_ptr<Node> next;
    };

    mutex head_mutex;
    unique_ptr<Node> head;

    mutex tail_mutex;
    Node* tail;  // raw pointer — tail is always valid (dummy node)

    Node* get_tail() {
        lock_guard<mutex> lock(tail_mutex);
        return tail;
    }

public:
    FineGrainedQueue() : head(make_unique<Node>()), tail(head.get()) {}

    // Push only locks the tail
    void push(T value) {
        auto new_data = make_shared<T>(std::move(value));
        auto new_node = make_unique<Node>();
        Node* new_tail = new_node.get();

        lock_guard<mutex> lock(tail_mutex);
        tail->data = new_data;
        tail->next = std::move(new_node);
        tail = new_tail;
    }

    // Pop only locks the head
    shared_ptr<T> try_pop() {
        lock_guard<mutex> lock(head_mutex);
        if (head.get() == get_tail()) {
            return nullptr;  // empty
        }
        shared_ptr<T> result = head->data;
        unique_ptr<Node> old_head = std::move(head);
        head = std::move(old_head->next);
        return result;
    }
};

// Producers lock tail_mutex, consumers lock head_mutex
// They don't contend with each other — higher throughput!
```

---

### Thread-Safe HashMap

A thread-safe hash map allows concurrent reads and writes. The simplest approach uses a single mutex, but **striped locking** (one mutex per bucket) provides much better concurrency.

**Simple approach — single mutex:**

```cpp
template <typename K, typename V>
class SimpleThreadSafeMap {
    unordered_map<K, V> map_;
    mutable shared_mutex mutex_;  // shared_mutex for read-write locking

public:
    void insert(const K& key, const V& value) {
        unique_lock<shared_mutex> lock(mutex_);  // exclusive lock for writes
        map_[key] = value;
    }

    optional<V> find(const K& key) const {
        shared_lock<shared_mutex> lock(mutex_);  // shared lock for reads
        auto it = map_.find(key);
        if (it != map_.end()) return it->second;
        return nullopt;
    }

    bool erase(const K& key) {
        unique_lock<shared_mutex> lock(mutex_);
        return map_.erase(key) > 0;
    }

    size_t size() const {
        shared_lock<shared_mutex> lock(mutex_);
        return map_.size();
    }
};
```

**Striped locking — one mutex per bucket (much better concurrency):**

```cpp
template <typename K, typename V, size_t NumBuckets = 64>
class StripedHashMap {
    struct Bucket {
        list<pair<K, V>> entries;
        mutable shared_mutex mutex;

        optional<V> find(const K& key) const {
            shared_lock<shared_mutex> lock(mutex);
            for (const auto& [k, v] : entries) {
                if (k == key) return v;
            }
            return nullopt;
        }

        void insert(const K& key, const V& value) {
            unique_lock<shared_mutex> lock(mutex);
            for (auto& [k, v] : entries) {
                if (k == key) {
                    v = value;  // update existing
                    return;
                }
            }
            entries.emplace_back(key, value);  // insert new
        }

        bool erase(const K& key) {
            unique_lock<shared_mutex> lock(mutex);
            auto it = find_if(entries.begin(), entries.end(),
                             [&key](const auto& p) { return p.first == key; });
            if (it != entries.end()) {
                entries.erase(it);
                return true;
            }
            return false;
        }
    };

    array<Bucket, NumBuckets> buckets;

    size_t get_bucket_index(const K& key) const {
        return hash<K>{}(key) % NumBuckets;
    }

public:
    void insert(const K& key, const V& value) {
        buckets[get_bucket_index(key)].insert(key, value);
    }

    optional<V> find(const K& key) const {
        return buckets[get_bucket_index(key)].find(key);
    }

    bool erase(const K& key) {
        return buckets[get_bucket_index(key)].erase(key);
    }
};

// With 64 buckets, up to 64 threads can operate on different buckets simultaneously!
// Much better than a single mutex for the entire map.

// Usage
StripedHashMap<string, int> map;

// Multiple threads can insert/read concurrently with minimal contention
thread t1([&map] {
    for (int i = 0; i < 1000; i++) {
        map.insert("key" + to_string(i), i);
    }
});

thread t2([&map] {
    for (int i = 0; i < 1000; i++) {
        auto val = map.find("key" + to_string(i));
        if (val) cout << *val << " ";
    }
});

t1.join();
t2.join();
```

---

### Lock-Free Stack (Basics)

A lock-free stack uses atomic CAS operations instead of mutexes. This is the simplest lock-free data structure.

```cpp
#include <atomic>
#include <memory>
using namespace std;

template <typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(const T& d) : data(d), next(nullptr) {}
    };

    atomic<Node*> head{nullptr};
    atomic<size_t> size_{0};

public:
    void push(const T& data) {
        Node* new_node = new Node(data);
        new_node->next = head.load(memory_order_relaxed);

        // CAS loop: atomically set head to new_node
        while (!head.compare_exchange_weak(
            new_node->next,  // expected (updated on failure)
            new_node,        // desired
            memory_order_release,
            memory_order_relaxed
        )) {
            // If CAS fails, new_node->next is updated to current head
            // Loop retries
        }

        size_.fetch_add(1, memory_order_relaxed);
    }

    bool pop(T& result) {
        Node* old_head = head.load(memory_order_relaxed);

        while (old_head && !head.compare_exchange_weak(
            old_head,          // expected
            old_head->next,    // desired (new head = old head's next)
            memory_order_acquire,
            memory_order_relaxed
        )) {
            // old_head is updated to current head on failure
        }

        if (!old_head) return false;  // stack was empty

        result = old_head->data;
        size_.fetch_sub(1, memory_order_relaxed);

        // WARNING: Deleting old_head here is UNSAFE in a concurrent environment!
        // Another thread might still be reading old_head->next.
        // Real implementations use hazard pointers or epoch-based reclamation.
        delete old_head;  // simplified — not safe in production

        return true;
    }

    bool empty() const {
        return head.load(memory_order_relaxed) == nullptr;
    }

    size_t size() const {
        return size_.load(memory_order_relaxed);
    }
};
```

**The ABA problem:**

A critical issue with lock-free data structures using CAS:

```
1. Thread A reads head = Node(A) → next = Node(B)
2. Thread A is preempted
3. Thread B pops Node(A), pops Node(B), pushes Node(C), pushes Node(A) back
4. Head is now: Node(A) → Node(C)
5. Thread A resumes, CAS succeeds (head is still Node(A))
6. But head->next is now Node(C), not Node(B)!
7. Thread A sets head = Node(B) — which was already freed!
```

**Solutions to ABA:**
- **Tagged pointers:** Combine the pointer with a version counter; CAS checks both
- **Hazard pointers:** Threads publish which nodes they're accessing; nodes aren't freed until safe
- **Epoch-based reclamation:** Defer deletion until all threads have passed a safe point
- **Reference counting:** Use `shared_ptr` with atomic operations (C++20 `atomic<shared_ptr>`)

```cpp
// Tagged pointer approach (simplified)
struct TaggedPtr {
    Node* ptr;
    size_t tag;  // version counter
};

atomic<TaggedPtr> head{{nullptr, 0}};

void push(const T& data) {
    Node* new_node = new Node(data);
    TaggedPtr old_head = head.load();

    TaggedPtr new_head;
    do {
        new_node->next = old_head.ptr;
        new_head = {new_node, old_head.tag + 1};  // increment tag
    } while (!head.compare_exchange_weak(old_head, new_head));
    // CAS now checks BOTH pointer AND tag — ABA detected if tag differs
}
```

---

### Choosing the Right Concurrent Data Structure

| Data Structure | Approach | Throughput | Complexity | Use Case |
|---------------|----------|-----------|-----------|----------|
| Single-mutex wrapper | Simple, correct | Low under contention | Easy | Prototyping, low contention |
| Reader-writer lock wrapper | `shared_mutex` | Good for read-heavy | Easy | Caches, config stores |
| Striped/bucketed locking | Lock per partition | High | Medium | Hash maps, databases |
| Lock-free stack/queue | CAS-based | Very high | Very hard | High-performance systems |
| SPSC queue | No locks needed | Highest | Medium | Single-producer, single-consumer |

**Guidelines:**
1. Start with a mutex-based solution — it's correct and easy to reason about
2. Profile to identify if synchronization is actually a bottleneck
3. Try `shared_mutex` if reads vastly outnumber writes
4. Try striped locking if contention is on a hash map
5. Only use lock-free structures if you have proven need and deep expertise

---

### Thread-Safe Singleton (Revisited)

A common concurrent data structure need — ensuring a singleton is initialized exactly once across threads:

```cpp
// Method 1: Meyer's Singleton (C++11 guarantees thread-safe static initialization)
class Singleton {
public:
    static Singleton& getInstance() {
        static Singleton instance;  // thread-safe in C++11+
        return instance;
    }

    void doWork() { cout << "Working" << endl; }

private:
    Singleton() { cout << "Singleton created" << endl; }
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
};

// Method 2: std::call_once (explicit control)
class SingletonV2 {
    static once_flag init_flag;
    static SingletonV2* instance;

public:
    static SingletonV2& getInstance() {
        call_once(init_flag, [] {
            instance = new SingletonV2();
        });
        return *instance;
    }

private:
    SingletonV2() { cout << "SingletonV2 created" << endl; }
};

once_flag SingletonV2::init_flag;
SingletonV2* SingletonV2::instance = nullptr;
```

---


## Summary & Key Takeaways

| Topic | Core Concept | Key Tools |
|-------|-------------|-----------|
| Thread Basics | Create, join, detach threads; pass arguments | `std::thread`, `std::ref`, `std::jthread` (C++20) |
| Synchronization | Protect shared data from data races | `mutex`, `lock_guard`, `unique_lock`, `scoped_lock` |
| Condition Variables | Wait for conditions without busy-waiting | `condition_variable`, `wait`, `notify_one/all` |
| Deadlock Prevention | Avoid circular lock dependencies | Lock ordering, `std::lock`, `scoped_lock` |
| Atomic Operations | Lock-free thread-safe operations on single variables | `std::atomic`, `fetch_add`, `compare_exchange` |
| Memory Ordering | Control visibility of writes across threads | `relaxed`, `acquire/release`, `seq_cst` |
| Async Programming | Launch tasks and retrieve results later | `async`, `future`, `promise`, `packaged_task` |
| Classic Problems | Producer-Consumer, Reader-Writer, Dining Philosophers | Condition variables, semaphores, shared_mutex |
| Thread Pool | Reuse threads for task execution | Task queue + condition variable + worker threads |
| Concurrent Data Structures | Thread-safe containers | Mutex wrappers, striped locking, lock-free CAS |

---

## Interview Tips for Module 9

1. **"What is a data race?"** Two or more threads access the same memory location concurrently, at least one is a write, and there's no synchronization. Data races cause undefined behavior in C++. Fix with mutexes, atomics, or by eliminating shared mutable state.

2. **"What's the difference between a mutex and an atomic?"** A mutex protects a critical section (multiple operations). An atomic protects a single variable (one operation). Atomics are faster but limited to simple operations. Use mutex for compound operations, atomic for counters and flags.

3. **"How do you prevent deadlock?"** Four strategies: (a) consistent lock ordering, (b) `std::lock` or `scoped_lock` to lock multiple mutexes atomically, (c) lock hierarchy, (d) try-lock with timeout. Explain the four conditions for deadlock (mutual exclusion, hold-and-wait, no preemption, circular wait).

4. **"Explain the Producer-Consumer problem."** Shared buffer, producers add items, consumers remove items. Use a mutex + two condition variables (not_full, not_empty). Producers wait on not_full, signal not_empty. Consumers wait on not_empty, signal not_full. Always use predicates with `wait()` to handle spurious wakeups.

5. **"What is a spurious wakeup?"** A thread waiting on a condition variable wakes up without being notified. Always use the predicate form of `wait()`: `cv.wait(lock, predicate)`. This re-checks the condition after every wakeup.

6. **"What's the difference between `lock_guard` and `unique_lock`?"** `lock_guard` is simple RAII — locks on construction, unlocks on destruction. `unique_lock` adds flexibility: deferred locking, try-lock, timed lock, manual lock/unlock, and is movable. Use `unique_lock` with condition variables.

7. **"Explain `std::async` vs `std::thread`."** `std::async` returns a `future` for the result and handles exceptions. `std::thread` gives you a raw thread with no built-in result mechanism. `async` can also defer execution. Use `async` for tasks that produce results; use `thread` for long-running background work.

8. **"What is a thread pool and why use one?"** A pool of pre-created worker threads that execute tasks from a queue. Benefits: avoids thread creation overhead, limits concurrency, provides task queuing. Implementation: worker threads loop on a condition variable, wake when tasks are submitted.

9. **"Explain memory ordering in atomics."** `seq_cst` (default) provides total ordering — safest but slowest. `acquire/release` provides synchronization between a producer and consumer. `relaxed` only guarantees atomicity — fastest but no ordering. Use `seq_cst` unless you have a proven performance need.

10. **"What is compare-and-swap (CAS)?"** Atomically: if current value equals expected, replace with desired and return true; otherwise, load current value into expected and return false. Foundation of lock-free programming. Used in a loop: read current value, compute new value, CAS to update.

11. **"What is the ABA problem?"** In CAS-based algorithms, a value changes from A to B and back to A. CAS sees A and succeeds, but the state has changed. Solutions: tagged pointers (version counter), hazard pointers, epoch-based reclamation.

12. **"How would you implement a thread-safe hash map?"** Start with `shared_mutex` wrapper (simple, correct). For better concurrency, use striped locking: divide into N buckets, each with its own mutex. Threads accessing different buckets don't contend. Choose N based on expected concurrency.

13. **"Explain the Reader-Writer problem."** Multiple readers can access data concurrently (no modification). Writers need exclusive access. Use `shared_mutex`: `shared_lock` for readers, `unique_lock` for writers. Discuss trade-offs: reader-preference (writers may starve) vs writer-preference (readers may starve).

14. **"What is the Dining Philosophers problem?"** Five philosophers, five forks, each needs two forks to eat. Naive approach deadlocks (all pick up left fork). Solutions: consistent lock ordering (always pick up lower-numbered fork first), `scoped_lock` for both forks, or limit concurrency (at most N-1 philosophers try to eat).

15. **"When would you use lock-free programming?"** Only when mutex-based code is a proven bottleneck, for simple data structures (counters, stacks, SPSC queues), and when the team deeply understands memory ordering. Lock-free code is extremely hard to get right. Start with mutexes, profile, then optimize.
