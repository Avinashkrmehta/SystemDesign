# Module 8: Advanced OOP & C++ Concepts

> This module covers advanced C++ techniques that go beyond basic OOP — move semantics for performance, type erasure for flexible polymorphism, CRTP for static dispatch, the Pimpl idiom for compilation firewalls, policy-based design for compile-time configurability, and strong typing to prevent bugs. These are the tools that separate production-quality C++ from textbook C++.

---

## 8.1 Move Semantics & Perfect Forwarding

> Move semantics allow resources to be *transferred* from one object to another instead of being *copied*. This is one of the most impactful features introduced in C++11 — it eliminates millions of unnecessary deep copies, making C++ code dramatically faster without sacrificing safety.

---

### Lvalues vs Rvalues

Every expression in C++ is either an **lvalue** (has a persistent identity/address) or an **rvalue** (a temporary with no persistent address).

```cpp
int x = 10;          // x is an lvalue (has an address, persists)
int y = x + 5;       // (x + 5) is an rvalue (temporary result, no address)

string name = "Alice";          // name is an lvalue
string greeting = "Hello " + name;  // "Hello " + name is an rvalue (temporary)

// You can take the address of an lvalue
int* p = &x;         // OK — x has an address

// You CANNOT take the address of an rvalue
// int* q = &(x + 5);  // ERROR — (x + 5) is a temporary, no address

// Lvalue references bind to lvalues
int& ref = x;        // OK
// int& ref2 = 42;   // ERROR — can't bind lvalue reference to rvalue

// Rvalue references bind to rvalues
int&& rref = 42;     // OK — rvalue reference extends the temporary's lifetime
// int&& rref2 = x;  // ERROR — can't bind rvalue reference to lvalue

// const lvalue references bind to BOTH
const int& cref1 = x;   // OK — binds to lvalue
const int& cref2 = 42;  // OK — binds to rvalue (extends lifetime)
```

**Value categories in C++11:**
```
            expression
           /          \
        glvalue      rvalue
       /      \     /     \
    lvalue   xvalue      prvalue

lvalue:  has identity, NOT movable from (variables, dereferenced pointers)
xvalue:  has identity, movable from (std::move(x), function returning T&&)
prvalue: no identity, movable from (literals, temporaries, function returning T)
```

**Practical rule of thumb:**
- If it has a name → lvalue
- If it's a temporary → rvalue
- `std::move(x)` casts an lvalue to an rvalue (xvalue)

---

### The Problem Move Semantics Solves

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <cstring>
using namespace std;

// Without move semantics — expensive copies everywhere
class HeavyObject {
    int* data;
    size_t size;

public:
    HeavyObject(size_t s) : size(s), data(new int[s]) {
        cout << "Constructor: allocated " << size << " ints" << endl;
    }

    // Copy constructor — DEEP COPY (expensive!)
    HeavyObject(const HeavyObject& other) : size(other.size), data(new int[other.size]) {
        memcpy(data, other.data, size * sizeof(int));
        cout << "Copy constructor: deep copied " << size << " ints (EXPENSIVE)" << endl;
    }

    // Copy assignment — DEEP COPY (expensive!)
    HeavyObject& operator=(const HeavyObject& other) {
        if (this != &other) {
            delete[] data;
            size = other.size;
            data = new int[size];
            memcpy(data, other.data, size * sizeof(int));
            cout << "Copy assignment: deep copied " << size << " ints (EXPENSIVE)" << endl;
        }
        return *this;
    }

    ~HeavyObject() {
        delete[] data;
    }
};

// This triggers COPY even though the returned object is a temporary!
HeavyObject createHeavy() {
    HeavyObject obj(1000000);  // 1 million ints
    return obj;  // Without move: copies 4MB of data!
}

int main() {
    HeavyObject a = createHeavy();  // Copy from temporary — wasteful!
    HeavyObject b = a;              // Copy from lvalue — necessary
    // The copy from createHeavy() is pure waste — the temporary is about to die anyway
}
```

**The insight:** When the source is a temporary (rvalue) that's about to be destroyed, we don't need to *copy* its resources — we can *steal* them. This is what move semantics enable.

---

### Move Constructor and Move Assignment Operator

```cpp
#include <iostream>
#include <cstring>
#include <utility>
using namespace std;

class Buffer {
    char* data;
    size_t size;
    string name;

public:
    // Regular constructor
    Buffer(const string& n, size_t s) : name(n), size(s), data(new char[s]) {
        memset(data, 0, size);
        cout << "  [" << name << "] Constructed (" << size << " bytes)" << endl;
    }

    // Copy constructor — deep copy
    Buffer(const Buffer& other)
        : name(other.name + "_copy"), size(other.size), data(new char[other.size])
    {
        memcpy(data, other.data, size);
        cout << "  [" << name << "] COPY constructed from [" << other.name
             << "] (" << size << " bytes copied)" << endl;
    }

    // MOVE constructor — steal resources!
    Buffer(Buffer&& other) noexcept
        : name(std::move(other.name)), size(other.size), data(other.data)
    {
        // Steal the pointer — no allocation, no copy!
        other.data = nullptr;  // leave source in valid but empty state
        other.size = 0;
        cout << "  [" << name << "] MOVE constructed (" << size << " bytes STOLEN)" << endl;
    }

    // Copy assignment
    Buffer& operator=(const Buffer& other) {
        if (this != &other) {
            delete[] data;
            size = other.size;
            data = new char[size];
            memcpy(data, other.data, size);
            name = other.name + "_assigned";
            cout << "  [" << name << "] COPY assigned (" << size << " bytes copied)" << endl;
        }
        return *this;
    }

    // MOVE assignment — steal resources!
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data;          // free our current resources
            data = other.data;      // steal their pointer
            size = other.size;
            name = std::move(other.name);
            other.data = nullptr;   // leave source empty
            other.size = 0;
            cout << "  [" << name << "] MOVE assigned (" << size << " bytes STOLEN)" << endl;
        }
        return *this;
    }

    ~Buffer() {
        if (data) {
            cout << "  [" << name << "] Destroyed (" << size << " bytes freed)" << endl;
        } else {
            cout << "  [moved-from] Destroyed (empty — was moved from)" << endl;
        }
        delete[] data;
    }

    size_t getSize() const { return size; }
    bool isEmpty() const { return data == nullptr; }
};

int main() {
    cout << "=== Construction ===" << endl;
    Buffer a("A", 1024);

    cout << "\n=== Copy ===" << endl;
    Buffer b = a;  // copy constructor — a is an lvalue

    cout << "\n=== Move ===" << endl;
    Buffer c = std::move(a);  // move constructor — std::move casts a to rvalue
    // a is now in a "moved-from" state — valid but empty

    cout << "\n=== Move from temporary ===" << endl;
    Buffer d = Buffer("temp", 2048);  // move constructor from temporary (rvalue)

    cout << "\n=== Move assignment ===" << endl;
    Buffer e("E", 512);
    e = std::move(b);  // move assignment — steals b's resources

    cout << "\n=== Destruction ===" << endl;
    return 0;
}
```

**Key rules for move operations:**
1. Move constructor/assignment should be marked `noexcept` — STL containers only use move if it's `noexcept`
2. Leave the moved-from object in a **valid but unspecified state** (typically empty/null)
3. The moved-from object must still be safely destructible
4. Move is an optimization of copy — if move isn't available, copy is used

---

### std::move and std::forward

**`std::move`** doesn't actually move anything — it's just a cast to rvalue reference:

```cpp
// std::move is essentially this:
template <typename T>
typename remove_reference<T>::type&& move(T&& arg) noexcept {
    return static_cast<typename remove_reference<T>::type&&>(arg);
}

// It says: "I'm done with this object, you can steal its resources"
string a = "Hello";
string b = std::move(a);  // casts a to rvalue → triggers move constructor
// a is now empty (moved-from state)
```

**`std::forward`** preserves the value category of a forwarded argument (perfect forwarding):

```cpp
// Without perfect forwarding — always copies
template <typename T>
void wrapper(T arg) {
    target(arg);  // arg is always an lvalue here (it has a name!)
}

// With perfect forwarding — preserves lvalue/rvalue nature
template <typename T>
void wrapper(T&& arg) {  // universal/forwarding reference
    target(std::forward<T>(arg));  // forwards as lvalue or rvalue
}

// How it works:
// If called with lvalue:  T = string&,  forward<string&>(arg) → lvalue
// If called with rvalue:  T = string,   forward<string>(arg)  → rvalue
```

**Perfect forwarding example — factory function:**

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <utility>
using namespace std;

class Widget {
    string name;
    int value;
public:
    Widget(const string& n, int v) : name(n), value(v) {
        cout << "Widget(const string&, int) — copy of name" << endl;
    }
    Widget(string&& n, int v) : name(std::move(n)), value(v) {
        cout << "Widget(string&&, int) — move of name" << endl;
    }
    void print() const { cout << "Widget: " << name << " = " << value << endl; }
};

// Perfect forwarding factory — forwards args exactly as received
template <typename T, typename... Args>
unique_ptr<T> make(Args&&... args) {
    return make_unique<T>(std::forward<Args>(args)...);
}

int main() {
    string name = "Counter";

    // Passes lvalue — Widget receives const string&
    auto w1 = make<Widget>(name, 42);
    w1->print();

    // Passes rvalue — Widget receives string&&
    auto w2 = make<Widget>(string("Timer"), 100);
    w2->print();

    // Passes moved lvalue — Widget receives string&&
    auto w3 = make<Widget>(std::move(name), 7);
    w3->print();

    return 0;
}
```

---

### Rule of Zero, Rule of Three, Rule of Five

These rules guide when and how to define special member functions.

**Rule of Three (pre-C++11):**
If you define any of these, you should define all three:
1. Destructor
2. Copy constructor
3. Copy assignment operator

```cpp
// Rule of Three — if you manage a resource manually
class RuleOfThree {
    int* data;
    size_t size;
public:
    RuleOfThree(size_t s) : size(s), data(new int[s]) {}

    ~RuleOfThree() { delete[] data; }                          // 1. Destructor

    RuleOfThree(const RuleOfThree& other)                      // 2. Copy constructor
        : size(other.size), data(new int[other.size]) {
        memcpy(data, other.data, size * sizeof(int));
    }

    RuleOfThree& operator=(const RuleOfThree& other) {         // 3. Copy assignment
        if (this != &other) {
            delete[] data;
            size = other.size;
            data = new int[size];
            memcpy(data, other.data, size * sizeof(int));
        }
        return *this;
    }
};
```

**Rule of Five (C++11):**
If you define any of these, you should define all five:
1. Destructor
2. Copy constructor
3. Copy assignment operator
4. **Move constructor**
5. **Move assignment operator**

```cpp
// Rule of Five — complete resource management
class RuleOfFive {
    int* data;
    size_t size;
public:
    RuleOfFive(size_t s) : size(s), data(new int[s]) {}

    ~RuleOfFive() { delete[] data; }                            // 1

    RuleOfFive(const RuleOfFive& o)                             // 2
        : size(o.size), data(new int[o.size]) {
        memcpy(data, o.data, size * sizeof(int));
    }

    RuleOfFive& operator=(const RuleOfFive& o) {                // 3
        if (this != &o) {
            delete[] data;
            size = o.size;
            data = new int[size];
            memcpy(data, o.data, size * sizeof(int));
        }
        return *this;
    }

    RuleOfFive(RuleOfFive&& o) noexcept                         // 4
        : size(o.size), data(o.data) {
        o.data = nullptr;
        o.size = 0;
    }

    RuleOfFive& operator=(RuleOfFive&& o) noexcept {            // 5
        if (this != &o) {
            delete[] data;
            data = o.data;
            size = o.size;
            o.data = nullptr;
            o.size = 0;
        }
        return *this;
    }
};
```

**Rule of Zero (preferred):**
Don't define any special member functions — use RAII wrappers (smart pointers, `std::vector`, `std::string`) that handle resources for you. The compiler-generated defaults do the right thing.

```cpp
// Rule of Zero — the BEST approach
class RuleOfZero {
    vector<int> data;       // vector handles memory
    string name;            // string handles memory
    unique_ptr<Widget> ptr; // unique_ptr handles ownership

public:
    RuleOfZero(const string& n, size_t s)
        : name(n), data(s), ptr(make_unique<Widget>(n, 0)) {}

    // NO destructor needed — members clean up themselves
    // NO copy/move needed — compiler generates correct ones
    //   (unique_ptr makes it move-only automatically)
};
```

**Summary table:**

| Rule | When | Define |
|------|------|--------|
| Rule of Zero | Class doesn't manage raw resources | Nothing — use RAII members |
| Rule of Three | Class manages resources (pre-C++11) | Destructor + Copy ctor + Copy assign |
| Rule of Five | Class manages resources (C++11+) | Destructor + Copy ctor + Copy assign + Move ctor + Move assign |

**Always prefer Rule of Zero.** If you find yourself writing a destructor, ask: "Can I use a smart pointer or RAII wrapper instead?"

---

### Move Semantics with STL Containers

```cpp
#include <iostream>
#include <vector>
#include <string>
using namespace std;

class Resource {
    string name;
public:
    Resource(const string& n) : name(n) {
        cout << "  Constructed: " << name << endl;
    }
    Resource(const Resource& o) : name(o.name) {
        cout << "  COPIED: " << name << endl;
    }
    Resource(Resource&& o) noexcept : name(std::move(o.name)) {
        cout << "  MOVED: " << name << endl;
    }
    string getName() const { return name; }
};

int main() {
    vector<Resource> vec;
    vec.reserve(4);  // pre-allocate to avoid reallocation moves

    cout << "=== push_back with lvalue (copies) ===" << endl;
    Resource r1("Alpha");
    vec.push_back(r1);  // COPIES r1 into the vector

    cout << "\n=== push_back with rvalue (moves) ===" << endl;
    vec.push_back(Resource("Beta"));  // MOVES temporary into vector

    cout << "\n=== push_back with std::move (moves) ===" << endl;
    Resource r2("Gamma");
    vec.push_back(std::move(r2));  // MOVES r2 into vector — r2 is now empty

    cout << "\n=== emplace_back (constructs in-place) ===" << endl;
    vec.emplace_back("Delta");  // Constructs directly in vector — NO copy or move!

    cout << "\n=== Contents ===" << endl;
    for (const auto& r : vec) {
        cout << "  " << r.getName() << endl;
    }

    return 0;
}
```

**Key takeaway:** Use `emplace_back` over `push_back` when constructing new objects, and `std::move` when transferring existing objects into containers.

---


## 8.2 Type Erasure

> Type erasure is a technique that hides the concrete type of an object behind a uniform interface, enabling polymorphism **without** requiring a common base class or inheritance hierarchy. It lets you store and manipulate objects of different types through a single type, as long as they satisfy certain requirements. The standard library uses this extensively: `std::function`, `std::any`, and `std::variant`.

---

### Why Type Erasure?

Traditional polymorphism (virtual functions) requires inheritance — all types must derive from a common base class. This is intrusive and limiting:
- Can't make `int` or `std::string` derive from your interface
- Third-party classes can't be retrofitted into your hierarchy
- Inheritance creates tight coupling
- Virtual dispatch has overhead (vtable indirection)

**The problem with inheritance-only polymorphism:**
```cpp
// BAD: Must inherit from Drawable to be used polymorphically
class Drawable {
public:
    virtual void draw() const = 0;
    virtual ~Drawable() = default;
};

class Circle : public Drawable {
public:
    void draw() const override { cout << "Drawing circle" << endl; }
};

// Can't make std::string or int "Drawable" without wrapping them!
// Can't make a third-party class "Drawable" without modifying it!
```

Type erasure solves this: any type that has a `draw()` method (or satisfies some requirement) can be used — no inheritance needed.

---

### std::function — The Standard Type Eraser for Callables

`std::function` is the most common type erasure in C++. It can hold any callable — function pointer, lambda, functor, member function — behind a uniform interface.

```cpp
#include <iostream>
#include <functional>
#include <string>
#include <vector>
using namespace std;

// Regular function
int add(int a, int b) { return a + b; }

// Functor (function object)
struct Multiplier {
    int factor;
    int operator()(int a, int b) const { return a * b * factor; }
};

class Calculator {
public:
    int compute(int a, int b) const { return a - b; }
};

int main() {
    // std::function<int(int, int)> can hold ANY callable with this signature
    vector<function<int(int, int)>> operations;

    // 1. Regular function
    operations.push_back(add);

    // 2. Lambda
    operations.push_back([](int a, int b) { return a - b; });

    // 3. Lambda with capture
    int offset = 100;
    operations.push_back([offset](int a, int b) { return a + b + offset; });

    // 4. Functor
    operations.push_back(Multiplier{3});

    // 5. Member function (via lambda)
    Calculator calc;
    operations.push_back([&calc](int a, int b) { return calc.compute(a, b); });

    // All stored in the same vector, called through the same interface!
    for (const auto& op : operations) {
        cout << "Result: " << op(10, 5) << endl;
    }
    // Output: 15, 5, 115, 150, 5

    return 0;
}
```

**How `std::function` works internally (simplified):**
```
┌─────────────────────────────────────────┐
│  std::function<int(int,int)>            │
├─────────────────────────────────────────┤
│  ┌─────────────────────────────────┐    │
│  │  Concept (type-erased wrapper)  │    │
│  │  ┌───────────────────────────┐  │    │
│  │  │  virtual invoke(int,int)  │  │    │
│  │  │  virtual clone()          │  │    │
│  │  │  virtual ~destructor()    │  │    │
│  │  └───────────────────────────┘  │    │
│  │           ▲                     │    │
│  │           │                     │    │
│  │  ┌────────┴────────┐           │    │
│  │  │ Model<Lambda>   │           │    │
│  │  │ stores: lambda  │           │    │
│  │  │ invoke → lambda()│          │    │
│  │  └─────────────────┘           │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

It uses an internal abstract base class (Concept) with a templated derived class (Model) that wraps the actual callable. This is the core type erasure pattern.

---

### std::any — Type-Safe Void Pointer

`std::any` (C++17) can hold a value of any type. It's like `void*` but type-safe — you must cast to the correct type to retrieve the value.

```cpp
#include <iostream>
#include <any>
#include <string>
#include <vector>
#include <typeinfo>
using namespace std;

int main() {
    // std::any can hold any type
    any value;

    value = 42;
    cout << "int: " << any_cast<int>(value) << endl;

    value = string("Hello");
    cout << "string: " << any_cast<string>(value) << endl;

    value = 3.14;
    cout << "double: " << any_cast<double>(value) << endl;

    // Type checking
    cout << "Type: " << value.type().name() << endl;
    cout << "Has value: " << value.has_value() << endl;

    // Wrong cast throws std::bad_any_cast
    try {
        int x = any_cast<int>(value);  // value holds double, not int!
    } catch (const bad_any_cast& e) {
        cout << "Bad cast: " << e.what() << endl;
    }

    // Safe cast with pointer (returns nullptr on mismatch)
    if (auto* p = any_cast<double>(&value)) {
        cout << "Safe cast: " << *p << endl;
    }
    if (auto* p = any_cast<int>(&value)) {
        cout << "This won't print" << endl;
    }

    // Heterogeneous container
    vector<any> bag = {42, string("hello"), 3.14, true};
    for (const auto& item : bag) {
        if (item.type() == typeid(int)) {
            cout << "int: " << any_cast<int>(item) << endl;
        } else if (item.type() == typeid(string)) {
            cout << "string: " << any_cast<string>(item) << endl;
        } else if (item.type() == typeid(double)) {
            cout << "double: " << any_cast<double>(item) << endl;
        } else if (item.type() == typeid(bool)) {
            cout << "bool: " << any_cast<bool>(item) << endl;
        }
    }

    return 0;
}
```

**When to use `std::any`:**
- Property maps / configuration stores with mixed types
- Plugin systems where value types aren't known at compile time
- Message passing with arbitrary payloads

**When NOT to use:** When you know the set of possible types at compile time — use `std::variant` instead (it's faster and safer).

---

### std::variant — Type-Safe Union

`std::variant` (C++17) holds one of a fixed set of types. It's a type-safe replacement for C unions.

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <vector>
using namespace std;

// A variant that can hold int, double, or string
using Value = variant<int, double, string>;

// --- Visitor pattern with std::visit ---
struct PrintVisitor {
    void operator()(int val) const { cout << "int: " << val << endl; }
    void operator()(double val) const { cout << "double: " << val << endl; }
    void operator()(const string& val) const { cout << "string: \"" << val << "\"" << endl; }
};

// Generic visitor using overloaded lambdas
template <typename... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template <typename... Ts>
overloaded(Ts...) -> overloaded<Ts...>;

int main() {
    Value v = 42;
    cout << "Holds int: " << get<int>(v) << endl;

    v = 3.14;
    cout << "Holds double: " << get<double>(v) << endl;

    v = string("Hello");
    cout << "Holds string: " << get<string>(v) << endl;

    // Check which type is held
    cout << "Index: " << v.index() << endl;  // 2 (string is at index 2)
    cout << "Holds string? " << holds_alternative<string>(v) << endl;

    // Safe access with get_if (returns pointer or nullptr)
    if (auto* p = get_if<string>(&v)) {
        cout << "Got string: " << *p << endl;
    }

    // --- std::visit — the power of variant ---
    vector<Value> values = {42, 3.14, string("world"), 100, string("hello")};

    // Visit with functor
    cout << "\nWith PrintVisitor:" << endl;
    for (const auto& val : values) {
        visit(PrintVisitor{}, val);
    }

    // Visit with overloaded lambdas (modern C++ idiom)
    cout << "\nWith overloaded lambdas:" << endl;
    for (const auto& val : values) {
        visit(overloaded{
            [](int v)            { cout << "int: " << v << endl; },
            [](double v)         { cout << "double: " << v << endl; },
            [](const string& v)  { cout << "string: \"" << v << "\"" << endl; }
        }, val);
    }

    // Compute with visit
    cout << "\nDouble all numeric values:" << endl;
    for (auto& val : values) {
        visit(overloaded{
            [](int& v)           { v *= 2; },
            [](double& v)        { v *= 2; },
            [](string& v)        { v += v; }  // concatenate strings
        }, val);
    }
    for (const auto& val : values) {
        visit(PrintVisitor{}, val);
    }

    return 0;
}
```

**`std::variant` vs `std::any` vs inheritance:**

| Aspect | `std::variant` | `std::any` | Inheritance |
|--------|---------------|------------|-------------|
| Types known at compile time | Yes (fixed set) | No (any type) | Yes (hierarchy) |
| Type safety | Compile-time checked | Runtime checked | Compile-time (via vtable) |
| Performance | No heap allocation (usually) | May heap-allocate | Vtable indirection |
| Extensibility | Must modify variant definition | Fully open | Add new subclass |
| Visitor support | `std::visit` | Manual type checking | Virtual functions |
| Use case | Closed set of types | Open/unknown types | Shared interface |

---

### Building Your Own Type Erasure

The core pattern: an abstract inner class (Concept) with a templated concrete class (Model) that wraps any type satisfying the requirements.

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>
using namespace std;

// Type-erased "Printable" — any type with a print() method or operator<<
class Printable {
    // --- Inner abstract base (Concept) ---
    struct Concept {
        virtual void print(ostream& os) const = 0;
        virtual unique_ptr<Concept> clone() const = 0;
        virtual ~Concept() = default;
    };

    // --- Inner templated wrapper (Model) ---
    template <typename T>
    struct Model : Concept {
        T object;

        Model(T obj) : object(std::move(obj)) {}

        void print(ostream& os) const override {
            os << object;  // requires T to support operator<<
        }

        unique_ptr<Concept> clone() const override {
            return make_unique<Model<T>>(object);
        }
    };

    unique_ptr<Concept> pimpl;

public:
    // Constructor accepts ANY type that supports operator<<
    template <typename T>
    Printable(T obj) : pimpl(make_unique<Model<T>>(std::move(obj))) {}

    // Copy
    Printable(const Printable& other) : pimpl(other.pimpl->clone()) {}
    Printable& operator=(const Printable& other) {
        pimpl = other.pimpl->clone();
        return *this;
    }

    // Move
    Printable(Printable&&) noexcept = default;
    Printable& operator=(Printable&&) noexcept = default;

    // The type-erased operation
    friend ostream& operator<<(ostream& os, const Printable& p) {
        p.pimpl->print(os);
        return os;
    }
};

// --- Types that DON'T share a base class ---
struct Point {
    int x, y;
    friend ostream& operator<<(ostream& os, const Point& p) {
        return os << "(" << p.x << ", " << p.y << ")";
    }
};

struct Color {
    int r, g, b;
    friend ostream& operator<<(ostream& os, const Color& c) {
        return os << "rgb(" << c.r << "," << c.g << "," << c.b << ")";
    }
};

int main() {
    // Store completely unrelated types in the same container!
    vector<Printable> items;
    items.push_back(42);                    // int
    items.push_back(3.14);                  // double
    items.push_back(string("Hello"));       // string
    items.push_back(Point{10, 20});         // custom struct
    items.push_back(Color{255, 128, 0});    // another custom struct

    // All printed through the same interface — no common base class!
    for (const auto& item : items) {
        cout << item << endl;
    }

    return 0;
}
```

**The type erasure pattern:**
```
┌─────────────────────────────────────┐
│  Printable (public interface)        │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  Concept (abstract inner)   │    │
│  │  + print() = 0              │    │
│  │  + clone() = 0              │    │
│  └─────────────────────────────┘    │
│           ▲                         │
│           │                         │
│  ┌────────┴────────────────┐        │
│  │  Model<T> (templated)   │        │
│  │  - object: T            │        │
│  │  + print() → os << obj  │        │
│  │  + clone() → new Model  │        │
│  └─────────────────────────┘        │
└─────────────────────────────────────┘

Key insight: The template parameter T exists only inside Model<T>.
The outer class Printable has NO template parameter — it's a regular class.
This is how the type is "erased" — hidden behind the Concept interface.
```

---

### When to Use Type Erasure

| Scenario | Technique |
|----------|-----------|
| Store any callable | `std::function` |
| Store any type (open set) | `std::any` |
| Store one of N known types | `std::variant` |
| Store any type satisfying a concept | Custom type erasure (Concept/Model) |
| Need polymorphism without inheritance | Custom type erasure |
| Third-party types that can't inherit from your base | Type erasure wraps them |

**Trade-offs:**

| Advantage | Disadvantage |
|-----------|-------------|
| No inheritance required | Heap allocation (usually) |
| Works with any type | More complex implementation |
| Non-intrusive (doesn't modify wrapped types) | Harder to debug (type hidden) |
| Value semantics (copyable, movable) | `std::function` has overhead vs direct call |

---


## 8.3 CRTP (Curiously Recurring Template Pattern)

> CRTP is a C++ idiom where a class `Derived` inherits from a template base class `Base<Derived>`, passing itself as the template argument. This enables **static polymorphism** — the base class can call derived class methods at compile time without virtual functions, achieving polymorphic behavior with zero runtime overhead.

---

### Why CRTP?

Virtual functions have runtime cost:
- Vtable pointer in every object (8 bytes on 64-bit)
- Indirect function call through vtable (cache miss potential)
- Prevents inlining (compiler can't see through virtual dispatch)
- Prevents certain optimizations

For performance-critical code (game engines, numerical libraries, embedded systems), CRTP provides polymorphism resolved entirely at compile time.

**The cost of virtual dispatch:**
```cpp
// Virtual polymorphism — runtime overhead
class Shape {
public:
    virtual double area() const = 0;  // vtable lookup at runtime
    virtual ~Shape() = default;       // vtable pointer in every object
};

class Circle : public Shape {
    double radius;
public:
    Circle(double r) : radius(r) {}
    double area() const override { return 3.14159 * radius * radius; }
};

// In a tight loop processing millions of shapes:
for (auto& shape : shapes) {
    total += shape->area();  // virtual call — can't inline, cache miss risk
}
```

---

### Basic CRTP Structure

```cpp
// The "curiously recurring" part: Derived passes itself to Base
template <typename Derived>
class Base {
public:
    void interface() {
        // Call derived class method — resolved at COMPILE TIME
        static_cast<Derived*>(this)->implementation();
    }

    // Optional: provide default implementation
    void implementation() {
        cout << "Base default implementation" << endl;
    }
};

class Concrete : public Base<Concrete> {
public:
    void implementation() {
        cout << "Concrete implementation" << endl;
    }
};

class AnotherConcrete : public Base<AnotherConcrete> {
    // Uses default implementation from Base
};

int main() {
    Concrete c;
    c.interface();  // calls Concrete::implementation() — no virtual!

    AnotherConcrete ac;
    ac.interface();  // calls Base::implementation() (default)
}
```

**How it works:**
```
Base<Concrete> knows that Derived = Concrete at compile time.
static_cast<Derived*>(this) → static_cast<Concrete*>(this)
The compiler resolves the call at compile time — no vtable needed.
The call can be inlined for maximum performance.
```

---

### Static Polymorphism

```cpp
#include <iostream>
#include <cmath>
#include <chrono>
using namespace std;

// --- CRTP Base ---
template <typename Derived>
class Shape {
public:
    double area() const {
        return static_cast<const Derived*>(this)->areaImpl();
    }

    double perimeter() const {
        return static_cast<const Derived*>(this)->perimeterImpl();
    }

    void describe() const {
        cout << static_cast<const Derived*>(this)->name()
             << ": area=" << area()
             << ", perimeter=" << perimeter() << endl;
    }
};

// --- Derived classes ---
class Circle : public Shape<Circle> {
    double radius;
public:
    Circle(double r) : radius(r) {}

    double areaImpl() const { return M_PI * radius * radius; }
    double perimeterImpl() const { return 2 * M_PI * radius; }
    string name() const { return "Circle(r=" + to_string(radius) + ")"; }
};

class Rectangle : public Shape<Rectangle> {
    double width, height;
public:
    Rectangle(double w, double h) : width(w), height(h) {}

    double areaImpl() const { return width * height; }
    double perimeterImpl() const { return 2 * (width + height); }
    string name() const { return "Rect(" + to_string(width) + "x" + to_string(height) + ")"; }
};

class Triangle : public Shape<Triangle> {
    double a, b, c;  // sides
public:
    Triangle(double s1, double s2, double s3) : a(s1), b(s2), c(s3) {}

    double areaImpl() const {
        double s = (a + b + c) / 2;
        return sqrt(s * (s - a) * (s - b) * (s - c));  // Heron's formula
    }
    double perimeterImpl() const { return a + b + c; }
    string name() const { return "Triangle(" + to_string(a) + "," + to_string(b) + "," + to_string(c) + ")"; }
};

// --- Function template that works with any CRTP shape ---
template <typename ShapeType>
void printShapeInfo(const Shape<ShapeType>& shape) {
    shape.describe();
}

int main() {
    Circle c(5.0);
    Rectangle r(3.0, 4.0);
    Triangle t(3.0, 4.0, 5.0);

    printShapeInfo(c);  // Circle(r=5): area=78.5398, perimeter=31.4159
    printShapeInfo(r);  // Rect(3x4): area=12, perimeter=14
    printShapeInfo(t);  // Triangle(3,4,5): area=6, perimeter=12

    return 0;
}
```

**Limitation:** CRTP shapes can't be stored in a single container (each `Shape<T>` is a different type). For heterogeneous collections, you still need virtual functions or type erasure.

---

### Mixin Classes with CRTP

CRTP is excellent for adding reusable functionality (mixins) to classes without virtual overhead:

```cpp
#include <iostream>
#include <string>
#include <chrono>
using namespace std;

// --- Mixin: Add comparison operators from < ---
template <typename Derived>
class Comparable {
public:
    bool operator>(const Derived& other) const {
        return other < static_cast<const Derived&>(*this);
    }
    bool operator<=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) > other);
    }
    bool operator>=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) < other);
    }
    bool operator==(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) < other) &&
               !(other < static_cast<const Derived&>(*this));
    }
    bool operator!=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) == other);
    }
};

// --- Mixin: Add print capability ---
template <typename Derived>
class Printable {
public:
    void print(ostream& os = cout) const {
        os << static_cast<const Derived*>(this)->toString() << endl;
    }

    friend ostream& operator<<(ostream& os, const Derived& obj) {
        os << obj.toString();
        return os;
    }
};

// --- Mixin: Add clone capability ---
template <typename Derived>
class Cloneable {
public:
    Derived clone() const {
        return Derived(static_cast<const Derived&>(*this));
    }
};

// --- Mixin: Instance counter ---
template <typename Derived>
class InstanceCounter {
    static int count;
public:
    InstanceCounter() { count++; }
    InstanceCounter(const InstanceCounter&) { count++; }
    ~InstanceCounter() { count--; }
    static int getInstanceCount() { return count; }
};
template <typename Derived>
int InstanceCounter<Derived>::count = 0;

// --- Use multiple mixins ---
class Temperature
    : public Comparable<Temperature>
    , public Printable<Temperature>
    , public Cloneable<Temperature>
    , public InstanceCounter<Temperature>
{
    double celsius;
public:
    Temperature(double c) : celsius(c) {}

    double getCelsius() const { return celsius; }
    double getFahrenheit() const { return celsius * 9.0 / 5.0 + 32; }

    // Required by Comparable
    bool operator<(const Temperature& other) const {
        return celsius < other.celsius;
    }

    // Required by Printable
    string toString() const {
        return to_string(celsius) + "°C (" + to_string(getFahrenheit()) + "°F)";
    }
};

int main() {
    Temperature t1(100.0);
    Temperature t2(37.5);
    Temperature t3(0.0);

    // Printable mixin
    cout << "t1: " << t1 << endl;
    cout << "t2: " << t2 << endl;

    // Comparable mixin — only defined operator<, got all others free!
    cout << "t1 > t2: " << (t1 > t2) << endl;    // true
    cout << "t3 <= t2: " << (t3 <= t2) << endl;   // true
    cout << "t1 == t1: " << (t1 == t1) << endl;   // true
    cout << "t1 != t2: " << (t1 != t2) << endl;   // true

    // Cloneable mixin
    Temperature t4 = t1.clone();
    cout << "Cloned: " << t4 << endl;

    // InstanceCounter mixin
    cout << "Instances: " << Temperature::getInstanceCount() << endl;  // 4

    return 0;
}
```

---

### Performance Benefits over Virtual Dispatch

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <numeric>
using namespace std;

// --- Virtual version ---
class VirtualShape {
public:
    virtual double area() const = 0;
    virtual ~VirtualShape() = default;
};

class VirtualCircle : public VirtualShape {
    double r;
public:
    VirtualCircle(double radius) : r(radius) {}
    double area() const override { return 3.14159265 * r * r; }
};

// --- CRTP version ---
template <typename Derived>
class CRTPShape {
public:
    double area() const {
        return static_cast<const Derived*>(this)->areaImpl();
    }
};

class CRTPCircle : public CRTPShape<CRTPCircle> {
    double r;
public:
    CRTPCircle(double radius) : r(radius) {}
    double areaImpl() const { return 3.14159265 * r * r; }
};

int main() {
    const int N = 10000000;

    // Benchmark virtual dispatch
    vector<unique_ptr<VirtualShape>> vShapes;
    for (int i = 0; i < N; i++) {
        vShapes.push_back(make_unique<VirtualCircle>(i * 0.001));
    }

    auto start = chrono::high_resolution_clock::now();
    double vTotal = 0;
    for (const auto& s : vShapes) {
        vTotal += s->area();  // virtual call — can't inline
    }
    auto vTime = chrono::high_resolution_clock::now() - start;

    // Benchmark CRTP (static dispatch)
    vector<CRTPCircle> cShapes;
    cShapes.reserve(N);
    for (int i = 0; i < N; i++) {
        cShapes.emplace_back(i * 0.001);
    }

    start = chrono::high_resolution_clock::now();
    double cTotal = 0;
    for (const auto& s : cShapes) {
        cTotal += s.area();  // static call — compiler can inline!
    }
    auto cTime = chrono::high_resolution_clock::now() - start;

    cout << "Virtual: " << chrono::duration_cast<chrono::milliseconds>(vTime).count()
         << "ms (total=" << vTotal << ")" << endl;
    cout << "CRTP:    " << chrono::duration_cast<chrono::milliseconds>(cTime).count()
         << "ms (total=" << cTotal << ")" << endl;

    // CRTP is typically 2-5x faster due to inlining and cache friendliness
    return 0;
}
```

**Why CRTP is faster:**
1. **Inlining:** Compiler knows the exact function at compile time → can inline
2. **No vtable pointer:** Objects are smaller → better cache utilization
3. **No indirect call:** No pointer dereference through vtable
4. **Better optimization:** Compiler can see through the call and optimize

---

### CRTP vs Virtual Polymorphism

| Aspect | CRTP (Static) | Virtual (Dynamic) |
|--------|--------------|-------------------|
| Dispatch time | Compile time | Runtime |
| Performance | No overhead (inlinable) | Vtable indirection |
| Object size | No vtable pointer | +8 bytes per object |
| Heterogeneous containers | Not directly possible | Yes (`vector<Base*>`) |
| Runtime flexibility | No (type fixed at compile time) | Yes (can change at runtime) |
| Code complexity | More template complexity | Simpler, familiar |
| Compilation time | Slower (templates) | Faster |
| Error messages | Template errors (verbose) | Clear virtual errors |
| Best for | Performance-critical, known types | Flexible, extensible hierarchies |

**Use CRTP when:** Performance matters, types are known at compile time, and you don't need heterogeneous containers.

**Use virtual when:** You need runtime flexibility, heterogeneous collections, or simpler code.

---


## 8.4 Pimpl Idiom (Pointer to Implementation)

> The Pimpl (Pointer to Implementation) idiom hides a class's implementation details behind an opaque pointer, creating a **compilation firewall** between the public interface and the private implementation. Changes to the implementation don't require recompilation of code that uses the class.

---

### Why Pimpl?

In C++, the header file exposes everything — including private members. This creates problems:

**The problem without Pimpl:**
```cpp
// widget.h — every user of Widget sees ALL of this
#include <string>
#include <vector>
#include <map>
#include <memory>
#include "database.h"      // heavy dependency!
#include "network.h"       // heavy dependency!
#include "crypto.h"        // heavy dependency!

class Widget {
public:
    void doWork();

private:
    // These are PRIVATE but still visible in the header!
    // Any change here forces ALL users of Widget to recompile!
    string name;
    vector<int> data;
    map<string, string> cache;
    Database db;             // pulls in database.h for ALL users
    NetworkClient client;    // pulls in network.h for ALL users
    CryptoEngine crypto;     // pulls in crypto.h for ALL users
    int internalState;
    bool isInitialized;
};
```

**Problems:**
1. **Compilation cascade:** Changing a private member recompiles ALL files that include `widget.h`
2. **Header bloat:** Users must include all headers needed by private members
3. **ABI instability:** Changing private members changes the class layout → breaks binary compatibility
4. **Exposed internals:** Private members are visible (even if not accessible) in the header

---

### Structure

```
Before Pimpl:                          After Pimpl:

widget.h                               widget.h
┌──────────────────────┐               ┌──────────────────────┐
│ #include "database.h"│               │ class Widget {       │
│ #include "network.h" │               │   class Impl;        │  ← forward declaration only
│                      │               │   unique_ptr<Impl> p;│  ← opaque pointer
│ class Widget {       │               │ public:              │
│   Database db;       │               │   void doWork();     │
│   NetworkClient net; │               │ };                   │
│   string name;       │               └──────────────────────┘
│   vector<int> data;  │
│ public:              │               widget.cpp
│   void doWork();     │               ┌──────────────────────┐
│ };                   │               │ #include "database.h"│  ← heavy includes
└──────────────────────┘               │ #include "network.h" │    only in .cpp!
                                       │                      │
                                       │ class Widget::Impl { │
                                       │   Database db;       │
                                       │   NetworkClient net; │
                                       │   string name;       │
                                       │   vector<int> data;  │
                                       │ };                   │
                                       └──────────────────────┘
```

---

### Basic Implementation

**widget.h — clean, minimal header:**
```cpp
#pragma once
#include <memory>
#include <string>

class Widget {
public:
    // Constructor / Destructor
    Widget(const std::string& name, int value);
    ~Widget();  // MUST be declared here, defined in .cpp

    // Move operations
    Widget(Widget&& other) noexcept;
    Widget& operator=(Widget&& other) noexcept;

    // Copy operations (if needed)
    Widget(const Widget& other);
    Widget& operator=(const Widget& other);

    // Public interface
    void setName(const std::string& name);
    std::string getName() const;
    void setValue(int value);
    int getValue() const;
    void process();
    void print() const;

private:
    class Impl;                    // forward declaration — no definition needed!
    std::unique_ptr<Impl> pImpl;   // opaque pointer to implementation
};
```

**widget.cpp — all implementation details hidden here:**
```cpp
#include "widget.h"
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
// Heavy headers only included in .cpp — not exposed to users!

using namespace std;

// --- The actual implementation class ---
class Widget::Impl {
public:
    string name;
    int value;
    vector<int> history;       // internal data structure
    bool processed = false;    // internal state

    Impl(const string& n, int v) : name(n), value(v) {
        history.push_back(v);
    }

    void doProcessing() {
        // Complex internal logic — hidden from header
        for (int i = 0; i < value; i++) {
            history.push_back(i * i);
        }
        processed = true;
        cout << "Processed " << name << " with " << history.size() << " entries" << endl;
    }

    void printDetails() const {
        cout << "Widget[" << name << "] value=" << value
             << " processed=" << processed
             << " history_size=" << history.size() << endl;
    }
};

// --- Widget methods delegate to Impl ---
Widget::Widget(const string& name, int value)
    : pImpl(make_unique<Impl>(name, value)) {}

// Destructor MUST be defined in .cpp where Impl is complete
Widget::~Widget() = default;

// Move operations
Widget::Widget(Widget&& other) noexcept = default;
Widget& Widget::operator=(Widget&& other) noexcept = default;

// Copy operations — must deep copy the Impl
Widget::Widget(const Widget& other)
    : pImpl(make_unique<Impl>(*other.pImpl)) {}

Widget& Widget::operator=(const Widget& other) {
    if (this != &other) {
        pImpl = make_unique<Impl>(*other.pImpl);
    }
    return *this;
}

// Public methods delegate to Impl
void Widget::setName(const string& name) { pImpl->name = name; }
string Widget::getName() const { return pImpl->name; }
void Widget::setValue(int value) { pImpl->value = value; }
int Widget::getValue() const { return pImpl->value; }

void Widget::process() { pImpl->doProcessing(); }
void Widget::print() const { pImpl->printDetails(); }
```

**main.cpp — user code:**
```cpp
#include "widget.h"  // clean header — no heavy dependencies!

int main() {
    Widget w("Alpha", 5);
    w.print();
    w.process();
    w.print();

    // Copy
    Widget w2 = w;
    w2.setName("Beta");
    w2.print();

    // Move
    Widget w3 = std::move(w);
    w3.print();
    // w is now in moved-from state

    return 0;
}
```

---

### Compilation Firewall

The key benefit — changes to `Impl` don't trigger recompilation of users:

```
Scenario: Add a new private member to Widget

WITHOUT Pimpl:
  Change widget.h → ALL files that #include "widget.h" must recompile
  In a large project: hundreds of files recompile for a private change!

WITH Pimpl:
  Change widget.cpp (Impl class) → ONLY widget.cpp recompiles
  All other files see the same widget.h → no recompilation needed!

Build time impact on a large project:
  Without Pimpl: change private member → 5 minute rebuild
  With Pimpl:    change private member → 2 second rebuild
```

---

### ABI Stability

Pimpl provides **binary compatibility** — you can change the implementation without breaking existing compiled code.

```cpp
// Version 1.0 of your library
class Widget::Impl {
    string name;
    int value;
};
// sizeof(Widget) = sizeof(unique_ptr<Impl>) = 8 bytes (always!)

// Version 2.0 — added new members
class Widget::Impl {
    string name;
    int value;
    vector<int> history;     // NEW
    Database* db;            // NEW
    map<string, string> cache; // NEW
};
// sizeof(Widget) = sizeof(unique_ptr<Impl>) = 8 bytes (STILL!)

// The Widget class layout hasn't changed!
// Old compiled code still works with the new library binary.
// No recompilation needed for users — just swap the .so/.dll file.
```

**This is critical for:**
- Shared libraries (.so / .dll) — users don't need to recompile
- Plugin systems — plugins compiled against old headers still work
- System libraries — OS updates don't break applications

---

### Reducing Header Dependencies

```cpp
// WITHOUT Pimpl — widget.h pulls in everything
// Anyone who includes widget.h also includes all of these:
#include <boost/asio.hpp>           // 50,000+ lines
#include <openssl/ssl.h>            // 10,000+ lines
#include "internal_database.h"      // 5,000+ lines
#include "proprietary_protocol.h"   // 3,000+ lines

// WITH Pimpl — widget.h is minimal
#include <memory>
#include <string>
// That's it! Heavy headers are only in widget.cpp
```

**Impact on compile times:**
- Each heavy header is parsed once (in widget.cpp) instead of in every file that uses Widget
- Precompiled headers become more effective
- Incremental builds are much faster

---

### Pimpl with `unique_ptr` — Important Details

```cpp
// WHY must the destructor be in the .cpp file?

// widget.h
class Widget {
    class Impl;
    unique_ptr<Impl> pImpl;
public:
    Widget();
    ~Widget();  // MUST declare here, define in .cpp!
    // If you use ~Widget() = default; in the header, it won't compile!
    // Because unique_ptr needs to see the complete Impl type to delete it,
    // and Impl is only forward-declared in the header.
};

// widget.cpp
class Widget::Impl { /* ... */ };

Widget::Widget() : pImpl(make_unique<Impl>()) {}
Widget::~Widget() = default;  // OK here — Impl is complete in this file

// Same applies to move operations:
Widget::Widget(Widget&&) noexcept = default;            // must be in .cpp
Widget& Widget::operator=(Widget&&) noexcept = default; // must be in .cpp
```

---

### When to Use Pimpl

| Scenario | Why Pimpl Helps |
|----------|----------------|
| Library/API development | ABI stability across versions |
| Large projects with slow builds | Compilation firewall reduces rebuild times |
| Hiding implementation details | Users can't see private members at all |
| Reducing header dependencies | Heavy includes only in .cpp |
| Platform-specific implementations | Different Impl per platform, same header |

**When NOT to use Pimpl:**
- Small, simple classes (overhead not worth it)
- Performance-critical inner loops (pointer indirection adds latency)
- Header-only libraries (no .cpp file to hide implementation)
- Classes that are rarely changed

**Costs of Pimpl:**
- Heap allocation for every object (can use custom allocator to mitigate)
- Pointer indirection on every method call
- More boilerplate code (delegation methods)
- Slightly harder to debug (implementation hidden behind pointer)

---


## 8.5 Policy-Based Design

> Policy-based design uses template parameters to configure a class's behavior at compile time. Instead of using virtual functions (runtime strategy), policies inject behavior through templates — the compiler generates optimized code for each specific combination of policies. It's essentially the **Strategy pattern resolved at compile time**.

---

### Why Policy-Based Design?

The Strategy pattern (runtime polymorphism) has overhead:
- Virtual function calls can't be inlined
- Vtable pointer adds to object size
- Runtime indirection hurts cache performance

Policy-based design achieves the same flexibility with zero runtime cost — the compiler generates specialized code for each policy combination.

**Runtime Strategy vs Compile-Time Policy:**
```cpp
// Runtime Strategy — flexible but has overhead
class Sorter {
    SortStrategy* strategy;  // virtual dispatch at runtime
public:
    void sort(vector<int>& data) {
        strategy->sort(data);  // indirect call — can't inline
    }
};

// Compile-Time Policy — zero overhead
template <typename SortPolicy>
class Sorter {
public:
    void sort(vector<int>& data) {
        SortPolicy::sort(data);  // resolved at compile time — inlined!
    }
};
```

---

### Basic Policy-Based Design

```cpp
#include <iostream>
#include <string>
#include <mutex>
#include <vector>
using namespace std;

// ============================================================
// POLICIES — small classes that provide specific behavior
// ============================================================

// --- Output Policies ---
struct ConsoleOutput {
    static void write(const string& message) {
        cout << message << endl;
    }
};

struct FileOutput {
    static void write(const string& message) {
        // Simulated file write
        cout << "[FILE] " << message << endl;
    }
};

struct NullOutput {
    static void write(const string& message) {
        // Discard — do nothing
    }
};

// --- Formatting Policies ---
struct PlainFormat {
    static string format(const string& level, const string& message) {
        return "[" + level + "] " + message;
    }
};

struct TimestampFormat {
    static string format(const string& level, const string& message) {
        return "2026-04-21 10:30:00 [" + level + "] " + message;
    }
};

struct JSONFormat {
    static string format(const string& level, const string& message) {
        return "{\"level\":\"" + level + "\",\"msg\":\"" + message + "\"}";
    }
};

// --- Threading Policies ---
struct SingleThreaded {
    struct Lock {
        Lock() {}  // no-op
    };
};

struct MultiThreaded {
    static mutex mtx;
    struct Lock {
        lock_guard<mutex> guard;
        Lock() : guard(mtx) {}
    };
};
mutex MultiThreaded::mtx;

// ============================================================
// HOST CLASS — configured by policies
// ============================================================

template <
    typename OutputPolicy = ConsoleOutput,
    typename FormatPolicy = PlainFormat,
    typename ThreadPolicy = SingleThreaded
>
class Logger {
public:
    void debug(const string& msg) { log("DEBUG", msg); }
    void info(const string& msg)  { log("INFO", msg); }
    void warn(const string& msg)  { log("WARN", msg); }
    void error(const string& msg) { log("ERROR", msg); }

private:
    void log(const string& level, const string& message) {
        typename ThreadPolicy::Lock lock;  // thread safety (or no-op)
        string formatted = FormatPolicy::format(level, message);  // format
        OutputPolicy::write(formatted);  // output
    }
};

// ============================================================
// Usage — mix and match policies at compile time
// ============================================================

int main() {
    // Console + Plain + SingleThreaded (default)
    Logger<> defaultLogger;
    defaultLogger.info("Default logger message");

    // Console + Timestamp format
    Logger<ConsoleOutput, TimestampFormat> timestampLogger;
    timestampLogger.info("With timestamp");

    // File + JSON format + Thread-safe
    Logger<FileOutput, JSONFormat, MultiThreaded> productionLogger;
    productionLogger.error("Database connection failed");

    // Null output (discard all logs) — useful for testing
    Logger<NullOutput> silentLogger;
    silentLogger.error("This goes nowhere");

    // Console + JSON format
    Logger<ConsoleOutput, JSONFormat> jsonConsoleLogger;
    jsonConsoleLogger.warn("Disk usage high");

    return 0;
}
```

**Output:**
```
[INFO] Default logger message
2026-04-21 10:30:00 [INFO] With timestamp
[FILE] {"level":"ERROR","msg":"Database connection failed"}
{"level":"WARN","msg":"Disk usage high"}
```

---

### Template Policies — Smart Pointer Example

A more sophisticated example showing how policies compose to create different smart pointer behaviors:

```cpp
#include <iostream>
#include <atomic>
#include <mutex>
using namespace std;

// ============================================================
// OWNERSHIP POLICIES
// ============================================================

// Deep copy on assignment
template <typename T>
struct DeepCopyPolicy {
    static T* clone(const T* ptr) {
        return ptr ? new T(*ptr) : nullptr;
    }
    static void release(T* ptr) {
        delete ptr;
    }
};

// Reference counting
template <typename T>
struct RefCountPolicy {
    struct ControlBlock {
        T* ptr;
        int count;
        ControlBlock(T* p) : ptr(p), count(1) {}
    };

    ControlBlock* cb = nullptr;

    void acquire(T* ptr) {
        cb = new ControlBlock(ptr);
    }

    void share(const RefCountPolicy& other) {
        cb = other.cb;
        if (cb) cb->count++;
    }

    bool release() {
        if (cb && --cb->count == 0) {
            delete cb->ptr;
            delete cb;
            cb = nullptr;
            return true;
        }
        return false;
    }

    T* get() const { return cb ? cb->ptr : nullptr; }
    int useCount() const { return cb ? cb->count : 0; }
};

// ============================================================
// CHECKING POLICIES
// ============================================================

template <typename T>
struct NoCheck {
    static void check(T* ptr) {
        // No checking — maximum performance
    }
};

template <typename T>
struct AssertCheck {
    static void check(T* ptr) {
        if (!ptr) {
            cerr << "ASSERTION FAILED: null pointer dereference!" << endl;
            abort();
        }
    }
};

template <typename T>
struct ThrowCheck {
    static void check(T* ptr) {
        if (!ptr) {
            throw runtime_error("Null pointer dereference");
        }
    }
};

// ============================================================
// POLICY-BASED SMART POINTER
// ============================================================

template <
    typename T,
    template <typename> class CheckPolicy = NoCheck
>
class SmartPtr {
    T* ptr;

public:
    explicit SmartPtr(T* p = nullptr) : ptr(p) {}

    ~SmartPtr() { delete ptr; }

    // Move only (simplified)
    SmartPtr(SmartPtr&& other) noexcept : ptr(other.ptr) { other.ptr = nullptr; }
    SmartPtr& operator=(SmartPtr&& other) noexcept {
        if (this != &other) {
            delete ptr;
            ptr = other.ptr;
            other.ptr = nullptr;
        }
        return *this;
    }

    SmartPtr(const SmartPtr&) = delete;
    SmartPtr& operator=(const SmartPtr&) = delete;

    T& operator*() const {
        CheckPolicy<T>::check(ptr);  // policy decides how to check
        return *ptr;
    }

    T* operator->() const {
        CheckPolicy<T>::check(ptr);
        return ptr;
    }

    T* get() const { return ptr; }
    explicit operator bool() const { return ptr != nullptr; }
};

// ============================================================
// Usage
// ============================================================

struct Widget {
    string name;
    Widget(const string& n) : name(n) {}
    void greet() const { cout << "Hello from " << name << endl; }
};

int main() {
    // No checking — fastest, but unsafe
    SmartPtr<Widget, NoCheck> fast(new Widget("Fast"));
    fast->greet();

    // Assert checking — crashes on null (debug builds)
    SmartPtr<Widget, AssertCheck> safe(new Widget("Safe"));
    safe->greet();

    // Throw checking — throws exception on null
    SmartPtr<Widget, ThrowCheck> throwing(new Widget("Throwing"));
    throwing->greet();

    // Test null access with ThrowCheck
    SmartPtr<Widget, ThrowCheck> nullPtr;
    try {
        nullPtr->greet();  // throws!
    } catch (const runtime_error& e) {
        cout << "Caught: " << e.what() << endl;
    }

    return 0;
}
```

---

### std::allocator as a Policy Example

The STL uses policy-based design extensively. `std::allocator` is a policy that controls how containers allocate memory:

```cpp
#include <iostream>
#include <vector>
#include <memory>
using namespace std;

// Custom allocator policy — tracks allocations
template <typename T>
struct TrackingAllocator {
    using value_type = T;

    static int totalAllocations;
    static int totalDeallocations;
    static size_t totalBytesAllocated;

    TrackingAllocator() = default;

    template <typename U>
    TrackingAllocator(const TrackingAllocator<U>&) {}

    T* allocate(size_t n) {
        totalAllocations++;
        totalBytesAllocated += n * sizeof(T);
        cout << "  [Alloc] " << n << " x " << sizeof(T)
             << " bytes = " << n * sizeof(T) << " bytes" << endl;
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }

    void deallocate(T* ptr, size_t n) {
        totalDeallocations++;
        cout << "  [Dealloc] " << n * sizeof(T) << " bytes" << endl;
        ::operator delete(ptr);
    }

    static void printStats() {
        cout << "Allocations: " << totalAllocations
             << ", Deallocations: " << totalDeallocations
             << ", Total bytes: " << totalBytesAllocated << endl;
    }
};

template <typename T> int TrackingAllocator<T>::totalAllocations = 0;
template <typename T> int TrackingAllocator<T>::totalDeallocations = 0;
template <typename T> size_t TrackingAllocator<T>::totalBytesAllocated = 0;

int main() {
    cout << "=== vector with tracking allocator ===" << endl;

    // vector uses the allocator policy to manage memory
    vector<int, TrackingAllocator<int>> vec;

    for (int i = 0; i < 10; i++) {
        vec.push_back(i);
    }

    cout << "\nVector contents: ";
    for (int v : vec) cout << v << " ";
    cout << endl;

    TrackingAllocator<int>::printStats();

    return 0;
}
```

---

### Policy-Based Design vs Runtime Strategy

| Aspect | Policy-Based (Compile-Time) | Strategy (Runtime) |
|--------|---------------------------|-------------------|
| Dispatch | Compile time (templates) | Runtime (virtual functions) |
| Performance | Zero overhead (inlined) | Virtual call overhead |
| Flexibility | Fixed at compile time | Changeable at runtime |
| Binary size | May increase (template instantiation) | Smaller (shared vtable) |
| Compilation | Slower (templates) | Faster |
| Error messages | Template errors (verbose) | Clear virtual errors |
| Combinability | Easy to combine multiple policies | Harder to combine strategies |
| Best for | Libraries, performance-critical code | Application logic, plugins |

**Use policy-based design when:**
- Behavior is known at compile time and won't change at runtime
- Performance is critical (inner loops, high-frequency calls)
- Building libraries or frameworks with configurable behavior
- Multiple orthogonal behaviors need to be combined

**Use runtime strategy when:**
- Behavior must change at runtime (user selection, configuration)
- Simplicity is more important than performance
- Plugin systems where new strategies are loaded dynamically

---


## 8.6 Enum Classes & Strong Types

> C++11 introduced `enum class` (scoped enumerations) to fix the problems of plain `enum`. Beyond enums, strong typing techniques prevent entire categories of bugs by making the type system work harder at compile time — catching errors that would otherwise be runtime bugs.

---

### enum class vs Plain enum

**Problems with plain `enum`:**
```cpp
// BAD: Plain enums — many problems
enum Color { RED, GREEN, BLUE };
enum TrafficLight { RED, YELLOW, GREEN };  // ERROR! RED and GREEN conflict!

enum Fruit { APPLE, BANANA, CHERRY };
enum Planet { MERCURY, VENUS, EARTH };

Fruit f = APPLE;
Planet p = MERCURY;

// Plain enums implicitly convert to int — allows nonsensical comparisons
if (f == p) {  // Compares APPLE (0) with MERCURY (0) — "equal"?!
    cout << "Fruit equals planet?!" << endl;  // This actually prints!
}

// Implicit conversion to int
int x = APPLE;  // OK — but should it be?
int y = RED;    // Which RED? Color::RED or TrafficLight::RED?

// Enumerators pollute the enclosing scope
// Can't have two enums with the same enumerator name
```

**`enum class` fixes all of these:**
```cpp
#include <iostream>
using namespace std;

// Scoped enumerations — no name conflicts, no implicit conversions
enum class Color { Red, Green, Blue };
enum class TrafficLight { Red, Yellow, Green };  // OK! No conflict with Color::Red

enum class Fruit { Apple, Banana, Cherry };
enum class Planet { Mercury, Venus, Earth };

int main() {
    Color c = Color::Red;
    TrafficLight t = TrafficLight::Red;

    // No implicit conversion to int
    // int x = c;           // ERROR! Can't implicitly convert
    int x = static_cast<int>(c);  // OK — explicit cast required

    // No comparison between different enum types
    // if (c == t) {}       // ERROR! Can't compare Color with TrafficLight
    // if (c == 0) {}       // ERROR! Can't compare Color with int

    // Same-type comparison works
    if (c == Color::Red) {
        cout << "Color is Red" << endl;
    }

    // Scoped names — no pollution
    Color color = Color::Green;      // must use Color:: prefix
    TrafficLight light = TrafficLight::Green;  // different Green, no conflict

    return 0;
}
```

---

### enum class Features

```cpp
#include <iostream>
#include <string>
#include <type_traits>
using namespace std;

// --- Specify underlying type ---
enum class HttpStatus : uint16_t {
    OK = 200,
    Created = 201,
    BadRequest = 400,
    Unauthorized = 401,
    Forbidden = 403,
    NotFound = 404,
    InternalError = 500,
    ServiceUnavailable = 503
};

// --- Forward declaration (possible with enum class) ---
enum class ErrorCode : int;  // forward declare — define later

// --- Helper functions ---
string httpStatusToString(HttpStatus status) {
    switch (status) {
        case HttpStatus::OK:                 return "200 OK";
        case HttpStatus::Created:            return "201 Created";
        case HttpStatus::BadRequest:         return "400 Bad Request";
        case HttpStatus::Unauthorized:       return "401 Unauthorized";
        case HttpStatus::Forbidden:          return "403 Forbidden";
        case HttpStatus::NotFound:           return "404 Not Found";
        case HttpStatus::InternalError:      return "500 Internal Server Error";
        case HttpStatus::ServiceUnavailable: return "503 Service Unavailable";
    }
    return "Unknown";
}

bool isSuccess(HttpStatus status) {
    auto code = static_cast<uint16_t>(status);
    return code >= 200 && code < 300;
}

bool isClientError(HttpStatus status) {
    auto code = static_cast<uint16_t>(status);
    return code >= 400 && code < 500;
}

// --- Using enum class as flags with bitwise operators ---
enum class Permission : uint8_t {
    None    = 0,
    Read    = 1 << 0,  // 0001
    Write   = 1 << 1,  // 0010
    Execute = 1 << 2,  // 0100
    All     = Read | Write | Execute  // ERROR without operator overloads!
};

// Must define operators explicitly for enum class (no implicit int conversion)
Permission operator|(Permission a, Permission b) {
    return static_cast<Permission>(
        static_cast<uint8_t>(a) | static_cast<uint8_t>(b));
}

Permission operator&(Permission a, Permission b) {
    return static_cast<Permission>(
        static_cast<uint8_t>(a) & static_cast<uint8_t>(b));
}

bool hasPermission(Permission perms, Permission check) {
    return (perms & check) == check;
}

int main() {
    HttpStatus status = HttpStatus::NotFound;
    cout << httpStatusToString(status) << endl;
    cout << "Is success: " << isSuccess(status) << endl;
    cout << "Is client error: " << isClientError(status) << endl;

    // Permissions
    Permission userPerms = Permission::Read | Permission::Write;
    cout << "\nHas Read: " << hasPermission(userPerms, Permission::Read) << endl;
    cout << "Has Execute: " << hasPermission(userPerms, Permission::Execute) << endl;

    return 0;
}
```

---

### enum class vs plain enum Summary

| Aspect | Plain `enum` | `enum class` |
|--------|-------------|--------------|
| Scope | Enumerators in enclosing scope | Enumerators scoped to enum name |
| Name conflicts | Yes (same names clash) | No (scoped, e.g., `Color::Red`) |
| Implicit int conversion | Yes (dangerous) | No (must `static_cast`) |
| Cross-type comparison | Allowed (nonsensical) | Compile error |
| Underlying type | Implementation-defined | Specifiable (`enum class X : uint8_t`) |
| Forward declaration | Not always possible | Always possible |
| Type safety | Weak | Strong |

**Rule: Always use `enum class` in modern C++.** There's no good reason to use plain `enum` in new code.

---

### Strong Typedefs

C++ `typedef` and `using` create aliases, not new types. This means the compiler can't distinguish between them:

```cpp
// BAD: Type aliases don't prevent mixing up parameters
using Meters = double;
using Seconds = double;
using MetersPerSecond = double;

MetersPerSecond calculateSpeed(Meters distance, Seconds time) {
    return distance / time;
}

int main() {
    Meters d = 100.0;
    Seconds t = 9.58;

    // BUG: Arguments swapped — compiles fine, wrong result!
    auto speed = calculateSpeed(t, d);  // passes seconds as meters!
    // Compiler sees double, double → no error!
}
```

**Strong typedefs create distinct types that the compiler enforces:**

```cpp
#include <iostream>
#include <string>
using namespace std;

// --- Strong typedef template ---
template <typename T, typename Tag>
class StrongType {
    T value;
public:
    explicit StrongType(T v) : value(v) {}

    T get() const { return value; }

    // Comparison operators
    bool operator==(const StrongType& other) const { return value == other.value; }
    bool operator!=(const StrongType& other) const { return value != other.value; }
    bool operator<(const StrongType& other) const { return value < other.value; }
    bool operator>(const StrongType& other) const { return value > other.value; }
    bool operator<=(const StrongType& other) const { return value <= other.value; }
    bool operator>=(const StrongType& other) const { return value >= other.value; }

    // Arithmetic (only with same type)
    StrongType operator+(const StrongType& other) const {
        return StrongType(value + other.value);
    }
    StrongType operator-(const StrongType& other) const {
        return StrongType(value - other.value);
    }

    friend ostream& operator<<(ostream& os, const StrongType& st) {
        return os << st.value;
    }
};

// --- Define distinct types using tags ---
struct MetersTag {};
struct SecondsTag {};
struct KilogramsTag {};
struct MetersPerSecondTag {};

using Meters = StrongType<double, MetersTag>;
using Seconds = StrongType<double, SecondsTag>;
using Kilograms = StrongType<double, KilogramsTag>;
using MetersPerSecond = StrongType<double, MetersPerSecondTag>;

// Function with strong types — can't mix up parameters!
MetersPerSecond calculateSpeed(Meters distance, Seconds time) {
    return MetersPerSecond(distance.get() / time.get());
}

// Another example: user IDs
struct UserIdTag {};
struct OrderIdTag {};
using UserId = StrongType<int, UserIdTag>;
using OrderId = StrongType<int, OrderIdTag>;

void processOrder(UserId user, OrderId order) {
    cout << "Processing order " << order << " for user " << user << endl;
}

int main() {
    Meters distance(100.0);
    Seconds time(9.58);

    auto speed = calculateSpeed(distance, time);
    cout << "Speed: " << speed << " m/s" << endl;

    // These would be COMPILE ERRORS:
    // calculateSpeed(time, distance);     // ERROR! Seconds is not Meters
    // calculateSpeed(distance, distance); // ERROR! Meters is not Seconds

    // Can't mix Meters and Seconds
    // Meters m = distance + time;  // ERROR! Different types

    // Can add same types
    Meters total = Meters(50.0) + Meters(30.0);
    cout << "Total distance: " << total << " m" << endl;

    // User/Order IDs can't be mixed
    UserId user(42);
    OrderId order(1001);
    processOrder(user, order);

    // processOrder(order, user);  // ERROR! OrderId is not UserId

    return 0;
}
```

---

### Preventing Implicit Conversions

Beyond strong typedefs, C++ offers several mechanisms to prevent dangerous implicit conversions:

```cpp
#include <iostream>
#include <string>
using namespace std;

// --- 1. explicit constructors ---
class Temperature {
    double celsius;
public:
    // Without explicit: Temperature t = 37.5; would work (implicit conversion)
    explicit Temperature(double c) : celsius(c) {}

    double getCelsius() const { return celsius; }
    double getFahrenheit() const { return celsius * 9.0 / 5.0 + 32; }

    friend ostream& operator<<(ostream& os, const Temperature& t) {
        return os << t.celsius << "°C";
    }
};

void checkFever(Temperature temp) {
    if (temp.getCelsius() > 37.5) {
        cout << temp << " — Fever detected!" << endl;
    } else {
        cout << temp << " — Normal" << endl;
    }
}

// --- 2. explicit conversion operators ---
class Percentage {
    double value;
public:
    explicit Percentage(double v) : value(v) {}

    // Without explicit: could accidentally use Percentage as double
    explicit operator double() const { return value; }

    // Without explicit: could accidentally use Percentage as bool
    explicit operator bool() const { return value != 0.0; }

    friend ostream& operator<<(ostream& os, const Percentage& p) {
        return os << p.value << "%";
    }
};

// --- 3. Deleted functions to prevent specific conversions ---
class SafeInt {
    int value;
public:
    SafeInt(int v) : value(v) {}

    // Delete constructors from other types
    SafeInt(double) = delete;       // prevent: SafeInt x = 3.14;
    SafeInt(bool) = delete;         // prevent: SafeInt x = true;
    SafeInt(char) = delete;         // prevent: SafeInt x = 'A';
    SafeInt(long long) = delete;    // prevent: SafeInt x = 1LL;

    int get() const { return value; }
};

int main() {
    // explicit constructor
    Temperature t(37.0);       // OK — direct initialization
    // Temperature t2 = 37.0;  // ERROR — implicit conversion blocked!
    checkFever(Temperature(38.5));  // OK — explicit construction
    // checkFever(38.5);            // ERROR — can't implicitly convert double to Temperature

    // explicit conversion operator
    Percentage p(85.0);
    // double d = p;            // ERROR — implicit conversion blocked
    double d = static_cast<double>(p);  // OK — explicit cast
    // if (p) {}                // ERROR without explicit — but with explicit:
    if (static_cast<bool>(p)) {
        cout << p << " is non-zero" << endl;
    }

    // Deleted conversions
    SafeInt si(42);            // OK
    // SafeInt si2 = 3.14;     // ERROR — double constructor deleted
    // SafeInt si3 = true;     // ERROR — bool constructor deleted
    // SafeInt si4 = 'A';      // ERROR — char constructor deleted

    return 0;
}
```

---

### Practical Example — Units Library

```cpp
#include <iostream>
#include <cmath>
using namespace std;

// --- Unit-safe physical quantities ---
template <int M, int KG, int S>  // dimensions: meters, kilograms, seconds
class Quantity {
    double value;
public:
    explicit Quantity(double v) : value(v) {}
    double get() const { return value; }

    // Same-dimension arithmetic
    Quantity operator+(const Quantity& other) const {
        return Quantity(value + other.value);
    }
    Quantity operator-(const Quantity& other) const {
        return Quantity(value - other.value);
    }
    Quantity operator*(double scalar) const {
        return Quantity(value * scalar);
    }
    Quantity operator/(double scalar) const {
        return Quantity(value / scalar);
    }

    // Comparison
    bool operator<(const Quantity& other) const { return value < other.value; }
    bool operator>(const Quantity& other) const { return value > other.value; }
    bool operator==(const Quantity& other) const { return value == other.value; }
};

// Multiplication of different dimensions → new dimension
template <int M1, int KG1, int S1, int M2, int KG2, int S2>
Quantity<M1+M2, KG1+KG2, S1+S2>
operator*(const Quantity<M1,KG1,S1>& a, const Quantity<M2,KG2,S2>& b) {
    return Quantity<M1+M2, KG1+KG2, S1+S2>(a.get() * b.get());
}

// Division of different dimensions → new dimension
template <int M1, int KG1, int S1, int M2, int KG2, int S2>
Quantity<M1-M2, KG1-KG2, S1-S2>
operator/(const Quantity<M1,KG1,S1>& a, const Quantity<M2,KG2,S2>& b) {
    return Quantity<M1-M2, KG1-KG2, S1-S2>(a.get() / b.get());
}

// --- Type aliases for common units ---
//                        M  KG  S
using Dimensionless = Quantity<0, 0, 0>;
using Meters        = Quantity<1, 0, 0>;
using Kilograms     = Quantity<0, 1, 0>;
using Seconds       = Quantity<0, 0, 1>;
using MetersPerSec  = Quantity<1, 0, -1>;   // m/s  = m * s^-1
using Acceleration  = Quantity<1, 0, -2>;   // m/s² = m * s^-2
using Newtons       = Quantity<1, 1, -2>;   // N = kg * m * s^-2
using SquareMeters  = Quantity<2, 0, 0>;    // m²

int main() {
    Meters distance(100.0);
    Seconds time(9.58);

    // Speed = distance / time → automatically gets correct dimension!
    MetersPerSec speed = distance / time;
    cout << "Speed: " << speed.get() << " m/s" << endl;

    // Force = mass * acceleration
    Kilograms mass(75.0);
    Acceleration accel(9.81);
    Newtons force = mass * accel;
    cout << "Force: " << force.get() << " N" << endl;

    // Area = length * width
    Meters length(10.0);
    Meters width(5.0);
    SquareMeters area = length * width;
    cout << "Area: " << area.get() << " m²" << endl;

    // These would be COMPILE ERRORS — dimension mismatch:
    // Meters m = distance + time;        // ERROR! Can't add meters and seconds
    // MetersPerSec v = distance + time;  // ERROR! Addition doesn't give m/s
    // Newtons f = mass + accel;          // ERROR! Can't add kg and m/s²

    // Can add same dimensions
    Meters total = distance + Meters(50.0);
    cout << "Total distance: " << total.get() << " m" << endl;

    return 0;
}
```

**The compiler catches physics errors at compile time!** You literally cannot add meters to seconds or assign a force to a velocity. The dimensional analysis is enforced by the type system with zero runtime cost.

---

### When to Use Strong Types

| Scenario | Technique |
|----------|-----------|
| Named constants with type safety | `enum class` |
| Preventing parameter mix-ups | Strong typedefs (tagged types) |
| Physical units | Dimensional analysis types |
| Preventing narrowing conversions | `explicit` constructors |
| Preventing specific type conversions | `= delete` on constructors |
| Domain-specific IDs (UserId, OrderId) | Strong typedefs |
| Bit flags with type safety | `enum class` with operator overloads |

**Benefits of strong typing:**
- Bugs caught at compile time instead of runtime
- Self-documenting code (types express intent)
- IDE autocompletion shows valid operations
- Zero runtime cost (all checks at compile time)
- Prevents entire categories of bugs (wrong argument order, unit mismatch)

**The principle:** Make illegal states unrepresentable. If the type system prevents a bug, you never need to write a test for it.

---
