# Module 1: C++ Fundamentals (Prerequisites)

> This module covers all the foundational C++ concepts you must master before diving into Low-Level Design. These are the building blocks for writing clean, efficient, and well-structured object-oriented code.

---

## 1.1 Basics

### Variables and Data Types

**Variables** are named storage locations in memory that hold a value of a specific type.

| Data Type | Size (typical) | Range | Example |
|-----------|---------------|-------|---------|
| `bool` | 1 byte | true/false | `bool flag = true;` |
| `char` | 1 byte | -128 to 127 | `char ch = 'A';` |
| `int` | 4 bytes | -2^31 to 2^31-1 | `int x = 42;` |
| `float` | 4 bytes | ~7 decimal digits | `float f = 3.14f;` |
| `double` | 8 bytes | ~15 decimal digits | `double d = 3.14159;` |
| `long long` | 8 bytes | -2^63 to 2^63-1 | `long long big = 1e18;` |

**Type Casting:**
- **Implicit (Widening):** Compiler automatically converts smaller type to larger type
  ```cpp
  int a = 10;
  double b = a; // int -> double (implicit)
  ```
- **Explicit (Narrowing):** Programmer forces conversion
  ```cpp
  double x = 3.99;
  int y = (int)x;           // C-style cast -> y = 3
  int z = static_cast<int>(x); // C++ style (preferred)
  ```
- **C++ Cast Operators:**
  - `static_cast<T>()` — compile-time checked, most common
  - `dynamic_cast<T>()` — runtime checked, for polymorphic types
  - `const_cast<T>()` — add/remove const
  - `reinterpret_cast<T>()` — low-level bit reinterpretation (dangerous)

---

### Operators

| Category | Operators | Notes |
|----------|-----------|-------|
| Arithmetic | `+`, `-`, `*`, `/`, `%` | `%` only for integers |
| Relational | `==`, `!=`, `<`, `>`, `<=`, `>=` | Return bool |
| Logical | `&&`, `||`, `!` | Short-circuit evaluation |
| Bitwise | `&`, `|`, `^`, `~`, `<<`, `>>` | Operate on bits |
| Assignment | `=`, `+=`, `-=`, `*=`, `/=`, `%=` | Compound assignment |
| Ternary | `condition ? expr1 : expr2` | Inline if-else |
| Increment/Decrement | `++`, `--` | Pre vs Post (++i vs i++) |

**Key Points:**
- `&&` and `||` use short-circuit evaluation (second operand not evaluated if result is determined)
- Bitwise operators are useful for flags, masks, and low-level optimization
- Pre-increment (`++i`) is generally preferred in C++ (avoids creating a temporary copy)

---

### Control Flow

**Conditional Statements:**
```cpp
// if-else
if (condition) {
    // ...
} else if (another_condition) {
    // ...
} else {
    // ...
}

// switch (works with int, char, enum)
switch (value) {
    case 1: /* ... */ break;
    case 2: /* ... */ break;
    default: /* ... */ break;
}
```

**Loops:**
```cpp
// for loop
for (int i = 0; i < n; i++) { }

// range-based for (C++11)
for (auto& elem : container) { }

// while loop
while (condition) { }

// do-while (executes at least once)
do { } while (condition);
```

**Important:** `break` exits the loop, `continue` skips to next iteration.

---

### Functions

**Pass by Value:**
```cpp
void func(int x) { x = 10; } // original unchanged
```

**Pass by Reference:**
```cpp
void func(int& x) { x = 10; } // original modified
```

**Pass by Pointer:**
```cpp
void func(int* x) { *x = 10; } // original modified via dereferencing
```

**Comparison:**
| Method | Copies Data? | Can Modify Original? | Can be NULL? |
|--------|-------------|---------------------|--------------|
| By Value | Yes | No | N/A |
| By Reference | No | Yes | No |
| By Pointer | No (copies address) | Yes | Yes |

**Default Arguments:**
```cpp
void print(int x, int y = 10, int z = 20); // defaults from right to left
```

**Function Overloading:**
- Same function name, different parameter lists (type, number, or order)
- Return type alone cannot differentiate overloaded functions
```cpp
int add(int a, int b);
double add(double a, double b);
int add(int a, int b, int c);
```

**Inline Functions:**
```cpp
inline int square(int x) { return x * x; }
```
- Suggests compiler to replace function call with function body
- Reduces function call overhead for small functions
- Compiler may ignore the suggestion for large functions

---

### Recursion

A function that calls itself to solve smaller subproblems.

```cpp
int factorial(int n) {
    if (n <= 1) return 1;       // Base case
    return n * factorial(n - 1); // Recursive case
}
```

**Key Concepts:**
- **Base case:** Condition that stops recursion (prevents infinite loop)
- **Recursive case:** Function calls itself with a smaller/simpler input
- **Stack overflow:** Occurs if recursion is too deep (no base case or too many calls)
- **Tail recursion:** Recursive call is the last operation (can be optimized by compiler)
- **Time complexity:** Often exponential without memoization
- **Space complexity:** O(depth) due to call stack

---

## 1.2 Pointers & Memory

### Pointers and References

**Pointer:** A variable that stores the memory address of another variable.
```cpp
int x = 10;
int* ptr = &x;   // ptr holds address of x
cout << *ptr;     // Dereferencing: prints 10
*ptr = 20;        // x is now 20
```

**Reference:** An alias (another name) for an existing variable.
```cpp
int x = 10;
int& ref = x;    // ref is an alias for x
ref = 20;         // x is now 20
```

**Key Differences:**
| Feature | Pointer | Reference |
|---------|---------|-----------|
| Can be NULL | Yes | No (must be initialized) |
| Can be reassigned | Yes | No (bound at creation) |
| Needs dereferencing | Yes (`*ptr`) | No (used directly) |
| Can have pointer to pointer | Yes (`int**`) | No |
| Memory | Has its own address | Same address as original |
| Arithmetic | Supported | Not supported |

---

### Pointer Arithmetic

```cpp
int arr[] = {10, 20, 30, 40, 50};
int* p = arr;       // points to arr[0]
p++;                // now points to arr[1] (moves by sizeof(int) = 4 bytes)
cout << *(p + 2);   // prints arr[3] = 40
cout << p[2];       // same as *(p + 2)
```

- Incrementing a pointer moves it by `sizeof(type)` bytes
- Array name decays to a pointer to its first element
- `ptr + n` moves pointer by `n * sizeof(*ptr)` bytes

---

### Dynamic Memory Allocation

**Stack vs Heap:**
- **Stack:** Automatic allocation/deallocation, limited size, fast
- **Heap:** Manual allocation/deallocation, large size, slower

```cpp
// Single variable
int* p = new int(42);    // allocate on heap
delete p;                 // free memory

// Array
int* arr = new int[100]; // allocate array on heap
delete[] arr;             // free array (MUST use delete[])
```

**Rules:**
- Every `new` must have a matching `delete`
- Every `new[]` must have a matching `delete[]`
- Never use `delete` on stack memory
- Never `delete` the same memory twice (double free)

---

### Memory Layout

```
+------------------+  High Address
|      Stack       |  (local variables, function calls, grows downward)
+------------------+
|        ↓         |
|                  |
|        ↑         |
+------------------+
|       Heap       |  (dynamic allocation, grows upward)
+------------------+
|   BSS Segment    |  (uninitialized global/static variables)
+------------------+
|   Data Segment   |  (initialized global/static variables)
+------------------+
|   Code/Text      |  (compiled program instructions)
+------------------+  Low Address
```

---

### Dangling Pointers and Memory Leaks

**Dangling Pointer:** Points to memory that has been freed.
```cpp
int* p = new int(10);
delete p;
// p is now dangling — accessing *p is undefined behavior
p = nullptr; // Good practice: set to nullptr after delete
```

**Memory Leak:** Allocated memory that is never freed.
```cpp
void leak() {
    int* p = new int(10);
    // function returns without delete — memory leaked!
}
```

**Prevention:**
- Always pair `new`/`delete`
- Set pointers to `nullptr` after deletion
- Use smart pointers (preferred in modern C++)
- Use tools like Valgrind or AddressSanitizer

---

### Smart Pointers (C++11)

Smart pointers automatically manage memory lifetime using RAII.

**`std::unique_ptr`** — Exclusive ownership (only one owner)
```cpp
#include <memory>
std::unique_ptr<int> p1 = std::make_unique<int>(42);
// auto p2 = p1;          // ERROR: cannot copy
auto p2 = std::move(p1);  // OK: transfer ownership (p1 is now nullptr)
// Memory automatically freed when p2 goes out of scope
```

**`std::shared_ptr`** — Shared ownership (reference counted)
```cpp
std::shared_ptr<int> p1 = std::make_shared<int>(42);
std::shared_ptr<int> p2 = p1; // OK: both own the resource
// Reference count = 2
// Memory freed when last shared_ptr is destroyed
cout << p1.use_count(); // prints 2
```

**`std::weak_ptr`** — Non-owning observer (breaks circular references)
```cpp
std::shared_ptr<int> sp = std::make_shared<int>(42);
std::weak_ptr<int> wp = sp; // does NOT increase ref count

if (auto locked = wp.lock()) { // convert to shared_ptr to use
    cout << *locked;
}
```

**When to use which:**
| Smart Pointer | Use Case |
|---------------|----------|
| `unique_ptr` | Single owner, most common, zero overhead |
| `shared_ptr` | Multiple owners need the same resource |
| `weak_ptr` | Break circular references, caching, observers |

---

### RAII (Resource Acquisition Is Initialization)

**Core Idea:** Tie resource lifetime to object lifetime.
- Acquire resource in constructor
- Release resource in destructor
- When object goes out of scope, destructor is called automatically

```cpp
class FileHandler {
    FILE* file;
public:
    FileHandler(const char* name) { file = fopen(name, "r"); }
    ~FileHandler() { if (file) fclose(file); } // auto cleanup
};

void process() {
    FileHandler fh("data.txt"); // resource acquired
    // ... use file ...
} // fh destroyed here -> file automatically closed
```

**Benefits:**
- No resource leaks (even if exceptions are thrown)
- No need for explicit cleanup code
- Smart pointers are RAII wrappers for dynamic memory
- `std::lock_guard` is RAII for mutexes
- `std::fstream` is RAII for file I/O

---

## 1.3 STL (Standard Template Library)

### Sequence Containers

| Container | Underlying Structure | Access | Insert/Delete | Use Case |
|-----------|---------------------|--------|---------------|----------|
| `vector` | Dynamic array | O(1) random | O(1) amortized at end, O(n) middle | Default choice |
| `deque` | Double-ended queue | O(1) random | O(1) at both ends | Need front/back insertion |
| `list` | Doubly linked list | O(n) | O(1) anywhere (with iterator) | Frequent mid-insertions |
| `forward_list` | Singly linked list | O(n) | O(1) after position | Memory-constrained |
| `array` | Fixed-size array | O(1) random | N/A (fixed size) | Compile-time known size |

**Vector (most commonly used):**
```cpp
#include <vector>
vector<int> v = {1, 2, 3, 4, 5};
v.push_back(6);          // add to end
v.pop_back();            // remove from end
v[0];                    // access by index (no bounds check)
v.at(0);                 // access with bounds check (throws exception)
v.size();                // number of elements
v.empty();               // check if empty
v.clear();               // remove all elements
v.begin(), v.end();      // iterators
v.insert(v.begin()+2, 10); // insert at position
v.erase(v.begin()+1);   // erase at position
```

---

### Container Adaptors

| Adaptor | Underlying | Behavior |
|---------|-----------|----------|
| `stack` | deque (default) | LIFO (Last In First Out) |
| `queue` | deque (default) | FIFO (First In First Out) |
| `priority_queue` | vector + heap | Largest element on top (max-heap) |

```cpp
#include <stack>
#include <queue>

stack<int> s;
s.push(10); s.push(20);
s.top();  // 20
s.pop();  // removes 20

queue<int> q;
q.push(10); q.push(20);
q.front(); // 10
q.back();  // 20
q.pop();   // removes 10

priority_queue<int> pq;                    // max-heap
priority_queue<int, vector<int>, greater<int>> min_pq; // min-heap
pq.push(30); pq.push(10); pq.push(20);
pq.top(); // 30
```

---

### Associative Containers (Ordered — based on Red-Black Tree)

| Container | Duplicates? | Key-Value? | Ordering |
|-----------|-------------|------------|----------|
| `set` | No | No | Sorted |
| `multiset` | Yes | No | Sorted |
| `map` | No (keys) | Yes | Sorted by key |
| `multimap` | Yes (keys) | Yes | Sorted by key |

```cpp
#include <set>
#include <map>

set<int> s = {3, 1, 4, 1, 5}; // stores {1, 3, 4, 5} (no duplicates, sorted)
s.insert(2);
s.find(3);    // returns iterator (or s.end() if not found)
s.count(1);   // 1 (exists) or 0 (not exists)
s.erase(4);

map<string, int> m;
m["apple"] = 5;
m["banana"] = 3;
m.insert({"cherry", 7});
for (auto& [key, val] : m) { } // structured bindings (C++17)
```

**Time Complexity:** O(log n) for insert, find, erase.

---

### Unordered Containers (Hash-based)

| Container | Duplicates? | Key-Value? | Average Time |
|-----------|-------------|------------|--------------|
| `unordered_set` | No | No | O(1) |
| `unordered_multiset` | Yes | No | O(1) |
| `unordered_map` | No (keys) | Yes | O(1) |
| `unordered_multimap` | Yes (keys) | Yes | O(1) |

```cpp
#include <unordered_map>
unordered_map<string, int> um;
um["key1"] = 100;
um.count("key1"); // 1 if exists
// Worst case: O(n) due to hash collisions
```

**When to use ordered vs unordered:**
- Use `map`/`set` when you need sorted order or range queries
- Use `unordered_map`/`unordered_set` when you need fastest average lookup

---

### Iterators

Iterators provide a uniform way to traverse containers.

```cpp
vector<int> v = {10, 20, 30, 40};

// Forward iteration
for (auto it = v.begin(); it != v.end(); ++it) {
    cout << *it << " ";
}

// Reverse iteration
for (auto it = v.rbegin(); it != v.rend(); ++it) {
    cout << *it << " ";
}

// Const iterators (read-only)
for (auto it = v.cbegin(); it != v.cend(); ++it) {
    // *it = 5; // ERROR: cannot modify through const_iterator
}
```

**Iterator Categories:**
1. **Input Iterator** — Read forward only (single pass)
2. **Output Iterator** — Write forward only (single pass)
3. **Forward Iterator** — Read/Write forward (multi-pass)
4. **Bidirectional Iterator** — Forward + backward (list, set, map)
5. **Random Access Iterator** — Any position in O(1) (vector, deque, array)

---

### Algorithms

```cpp
#include <algorithm>
#include <numeric>

vector<int> v = {5, 2, 8, 1, 9, 3};

// Sorting
sort(v.begin(), v.end());                    // ascending
sort(v.begin(), v.end(), greater<int>());    // descending
sort(v.begin(), v.end(), [](int a, int b) { return a > b; }); // custom

// Searching (on sorted containers)
binary_search(v.begin(), v.end(), 5);        // returns true/false
lower_bound(v.begin(), v.end(), 5);          // iterator to first >= 5
upper_bound(v.begin(), v.end(), 5);          // iterator to first > 5

// Finding (any container)
auto it = find(v.begin(), v.end(), 8);       // linear search

// Min/Max
*min_element(v.begin(), v.end());
*max_element(v.begin(), v.end());

// Others
reverse(v.begin(), v.end());
accumulate(v.begin(), v.end(), 0);           // sum (from <numeric>)
count(v.begin(), v.end(), 5);                // count occurrences
unique(v.begin(), v.end());                  // remove consecutive duplicates
next_permutation(v.begin(), v.end());        // next lexicographic permutation
```

---

### Functors and Lambda Expressions

**Functor (Function Object):** A class/struct with `operator()` overloaded.
```cpp
struct Multiply {
    int factor;
    Multiply(int f) : factor(f) {}
    int operator()(int x) const { return x * factor; }
};

Multiply triple(3);
cout << triple(5); // 15
```

**Lambda Expression (C++11):** Anonymous inline function.
```cpp
// Syntax: [capture](parameters) -> return_type { body }

auto add = [](int a, int b) { return a + b; };
cout << add(3, 4); // 7

int x = 10;
auto addX = [x](int a) { return a + x; };     // capture by value
auto modX = [&x](int a) { x += a; };          // capture by reference
auto all  = [=]() { };                          // capture all by value
auto allRef = [&]() { };                        // capture all by reference

// Used with algorithms
sort(v.begin(), v.end(), [](int a, int b) { return a > b; });
```

---

### std::pair and std::tuple

```cpp
#include <utility>  // pair
#include <tuple>    // tuple

// pair — holds exactly 2 values
pair<int, string> p = {1, "hello"};
cout << p.first << " " << p.second;
auto p2 = make_pair(42, 3.14);

// tuple — holds any number of values
tuple<int, double, string> t = {1, 3.14, "hi"};
cout << get<0>(t); // 1
cout << get<2>(t); // "hi"

// Structured bindings (C++17)
auto [a, b, c] = t; // a=1, b=3.14, c="hi"
```

---

### std::string and String Operations

```cpp
#include <string>

string s = "Hello, World!";
s.length();          // or s.size() — 13
s[0];                // 'H'
s.substr(7, 5);      // "World"
s.find("World");     // 7 (position) or string::npos if not found
s.append(" C++");    // "Hello, World! C++"
s += " C++";         // same as append
s.insert(5, " Dear");
s.erase(5, 5);       // erase 5 chars starting at position 5
s.replace(0, 5, "Hi"); // replace "Hello" with "Hi"
s.compare("other");  // 0 if equal, <0 or >0 otherwise

// Conversion
to_string(42);       // int -> string
stoi("42");          // string -> int
stod("3.14");        // string -> double

// Iteration
for (char c : s) { }
```

---

## 1.4 Templates

### Function Templates

Templates enable **generic programming** — write code once, use with any type.

```cpp
template <typename T>
T getMax(T a, T b) {
    return (a > b) ? a : b;
}

// Usage — compiler generates specific versions
cout << getMax<int>(3, 7);       // explicit type
cout << getMax(3.14, 2.71);      // type deduced as double
```

**Multiple template parameters:**
```cpp
template <typename T, typename U>
auto add(T a, U b) -> decltype(a + b) {
    return a + b;
}
```

---

### Class Templates

```cpp
template <typename T>
class Stack {
    vector<T> data;
public:
    void push(const T& val) { data.push_back(val); }
    T pop() {
        T top = data.back();
        data.pop_back();
        return top;
    }
    bool empty() const { return data.empty(); }
};

// Usage
Stack<int> intStack;
Stack<string> strStack;
```

---

### Template Specialization

**Full Specialization:** Provide a completely different implementation for a specific type.
```cpp
template <typename T>
class Printer {
public:
    void print(T val) { cout << val; }
};

// Full specialization for bool
template <>
class Printer<bool> {
public:
    void print(bool val) { cout << (val ? "true" : "false"); }
};
```

**Partial Specialization:** Specialize for a subset of types (only for class templates).
```cpp
// General template
template <typename T, typename U>
class Pair { /* ... */ };

// Partial specialization: when both types are the same
template <typename T>
class Pair<T, T> { /* different implementation */ };

// Partial specialization: when second type is a pointer
template <typename T, typename U>
class Pair<T, U*> { /* different implementation */ };
```

---

### Variadic Templates (C++11)

Templates that accept a variable number of arguments.

```cpp
// Base case
void print() { cout << endl; }

// Recursive variadic template
template <typename T, typename... Args>
void print(T first, Args... rest) {
    cout << first << " ";
    print(rest...);  // recursively process remaining args
}

// Usage
print(1, 2.5, "hello", 'c'); // prints: 1 2.5 hello c
```

**Fold Expressions (C++17):**
```cpp
template <typename... Args>
auto sum(Args... args) {
    return (args + ...); // fold expression
}
cout << sum(1, 2, 3, 4); // 10
```

---

### SFINAE (Substitution Failure Is Not An Error)

When the compiler tries to substitute template arguments and fails, it simply discards that overload instead of producing an error.

```cpp
#include <type_traits>

// Only enabled for integral types
template <typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
onlyForIntegers(T val) {
    return val * 2;
}

// C++17 simplified with if constexpr
template <typename T>
auto process(T val) {
    if constexpr (std::is_integral_v<T>) {
        return val * 2;
    } else {
        return val;
    }
}
```

---

### Type Traits

Type traits provide compile-time information about types.

```cpp
#include <type_traits>

std::is_same<int, int>::value;          // true
std::is_integral<float>::value;         // false
std::is_pointer<int*>::value;           // true
std::is_class<std::string>::value;      // true
std::is_base_of<Base, Derived>::value;  // true

// C++17 shorthand: _v suffix
std::is_same_v<int, int>;              // true

// enable_if for conditional compilation
template <typename T, typename = std::enable_if_t<std::is_arithmetic_v<T>>>
T square(T x) { return x * x; }
```

**Common Type Traits:**
- `is_same`, `is_integral`, `is_floating_point`, `is_arithmetic`
- `is_pointer`, `is_reference`, `is_array`
- `is_class`, `is_enum`, `is_abstract`
- `is_const`, `is_volatile`
- `is_base_of`, `is_convertible`
- `remove_const`, `remove_reference`, `add_pointer`

---

## 1.5 Namespaces & Preprocessor

### Namespaces

Namespaces prevent name collisions by grouping entities under a name.

**Named Namespace:**
```cpp
namespace Math {
    const double PI = 3.14159;
    double square(double x) { return x * x; }
}

// Usage
double area = Math::PI * Math::square(5.0);
```

**Unnamed (Anonymous) Namespace:**
```cpp
namespace {
    int helper() { return 42; } // internal linkage (like static)
}
// Only accessible within this translation unit (file)
```

**Nested Namespace (C++17):**
```cpp
namespace A::B::C {
    void func() { }
}
// Equivalent to: namespace A { namespace B { namespace C { ... } } }
```

---

### using Declarations vs using Directives

```cpp
// using declaration — brings ONE specific name into scope
using std::cout;
using std::endl;
cout << "Hello" << endl;

// using directive — brings ALL names from namespace into scope
using namespace std; // AVOID in header files (pollutes global namespace)
```

**Best Practices:**
- Never use `using namespace` in header files
- Prefer `using` declarations over directives
- Use full qualification (`std::vector`) in headers
- `using namespace` is acceptable in `.cpp` files or limited scopes

---

### Preprocessor Directives

The preprocessor runs before compilation, performing text substitution.

```cpp
// Macro definition
#define PI 3.14159
#define MAX(a, b) ((a) > (b) ? (a) : (b))  // macro function (use carefully!)
#define DEBUG_LOG(msg) cout << "[DEBUG] " << msg << endl

// Conditional compilation
#ifdef DEBUG
    // compiled only if DEBUG is defined
#endif

#ifndef RELEASE
    // compiled only if RELEASE is NOT defined
#endif

#if VERSION >= 2
    // compiled if condition is true
#elif VERSION == 1
    // ...
#else
    // ...
#endif

// Include
#include <iostream>    // system headers
#include "myfile.h"   // local headers

// Pragma
#pragma once           // include guard (non-standard but widely supported)
```

---

### Header Guards

Prevent multiple inclusion of the same header file.

**Traditional approach:**
```cpp
// myclass.h
#ifndef MYCLASS_H
#define MYCLASS_H

class MyClass {
    // ...
};

#endif // MYCLASS_H
```

**Modern approach:**
```cpp
// myclass.h
#pragma once

class MyClass {
    // ...
};
```

**`#pragma once` vs `#ifndef` guards:**
| Feature | `#pragma once` | `#ifndef` guards |
|---------|---------------|-----------------|
| Standard | Non-standard (but universal) | Standard C++ |
| Simplicity | One line | Three lines |
| Name collision | Impossible | Possible if guard name reused |
| Performance | Slightly faster (compiler optimization) | Standard |

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Data Types | Know sizes, ranges, and when to use each |
| Pass by ref/ptr | Use references for cleaner code, pointers when NULL is needed |
| Memory | Stack = automatic, Heap = manual (use smart pointers) |
| Smart Pointers | `unique_ptr` (default), `shared_ptr` (shared ownership), `weak_ptr` (break cycles) |
| RAII | Tie resource lifetime to object lifetime |
| STL Containers | `vector` (default), `map`/`set` (ordered), `unordered_map`/`unordered_set` (fast lookup) |
| Templates | Enable generic, type-safe code |
| SFINAE | Failed substitution removes overload, doesn't cause error |
| Namespaces | Prevent name collisions, avoid `using namespace` in headers |
| Preprocessor | `#pragma once` for guards, minimize macro usage in modern C++ |

---

## Interview Tips for Module 1

1. **Always prefer smart pointers** over raw `new`/`delete` in modern C++
2. **Know the difference** between `vector`, `list`, `deque` and when to use each
3. **Understand move semantics** — why `std::move` exists and when to use it
4. **Lambda expressions** are heavily used in modern C++ (know capture semantics)
5. **Template metaprogramming** questions are common in C++ interviews
6. **Memory layout** questions (stack vs heap, where variables live) are frequent
7. **RAII** is the foundation of resource management in C++ — understand it deeply
8. **STL algorithms** — know `sort`, `find`, `binary_search`, `lower_bound`, `upper_bound`
9. **Const correctness** — understand `const` with pointers, references, and member functions
10. **Undefined behavior** — know common causes (dangling pointers, buffer overflow, signed overflow)
