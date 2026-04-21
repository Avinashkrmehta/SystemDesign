# Module 2: Object-Oriented Programming (OOP) in C++

> This module covers the four pillars of OOP — Encapsulation, Inheritance, Polymorphism, and Abstraction — along with practical C++ implementation details essential for Low-Level Design. Each topic is explained in depth with multiple examples, edge cases, internal workings, and common pitfalls.

---

## 2.1 Classes & Objects

### Class Definition and Access Specifiers

A **class** is a user-defined type that encapsulates data (member variables/attributes) and behavior (member functions/methods) into a single unit. An **object** is an instance of a class — it occupies memory and has a concrete state.

**Syntax:**
```cpp
class ClassName {
    // members (default: private)
};

struct StructName {
    // members (default: public)
};
```

**Difference between `class` and `struct`:**
- The ONLY difference in C++ is the default access level
- `class` → default is `private`
- `struct` → default is `public`
- Convention: Use `struct` for plain data holders (POD), `class` for objects with behavior

**Complete Example:**
```cpp
class Employee {
private:
    // Data members — the state
    string name;
    int id;
    double salary;
    static int totalEmployees;  // shared across all objects

protected:
    string department;  // accessible in derived classes

public:
    // Constructor
    Employee(const string& name, int id, double salary, const string& dept)
        : name(name), id(id), salary(salary), department(dept) {
        totalEmployees++;
    }

    // Destructor
    ~Employee() { totalEmployees--; }

    // Public interface — what the outside world can do
    string getName() const { return name; }
    int getId() const { return id; }
    double getSalary() const { return salary; }

    void giveRaise(double percent) {
        if (percent > 0 && percent <= 50) {
            salary += salary * percent / 100.0;
        }
    }

    static int getTotalEmployees() { return totalEmployees; }
};

int Employee::totalEmployees = 0;  // static member definition
```

**Access Specifiers in Detail:**

| Specifier | Within Class | Derived Class | Outside Class | Use Case |
|-----------|:---:|:---:|:---:|---|
| `public` | ✓ | ✓ | ✓ | Interface methods, public API |
| `protected` | ✓ | ✓ | ✗ | Members needed by subclasses |
| `private` | ✓ | ✗ | ✗ | Implementation details, data |

**Multiple access specifier sections:**
```cpp
class MyClass {
public:
    void publicMethod1();
    void publicMethod2();

private:
    int data1;
    int data2;

public:  // you can have multiple public/private sections
    void anotherPublicMethod();

private:
    void helperMethod();
};
```

**Object Creation:**
```cpp
// Stack allocation (automatic storage duration)
Employee emp1("Alice", 101, 75000, "Engineering");

// Heap allocation (dynamic storage duration)
Employee* emp2 = new Employee("Bob", 102, 80000, "Design");
// ... use emp2 ...
delete emp2;  // must manually free

// Modern C++ — use smart pointers for heap objects
auto emp3 = std::make_unique<Employee>("Charlie", 103, 90000, "PM");
// automatically freed when emp3 goes out of scope
```

**Object Size and Memory Layout:**
```cpp
class Empty {};           // sizeof(Empty) == 1 (minimum 1 byte for unique address)

class Simple {
    int x;                // 4 bytes
    double y;             // 8 bytes
    char c;               // 1 byte + 7 bytes padding (alignment)
};                        // sizeof(Simple) == 24 (with typical alignment)

class WithVirtual {
    virtual void foo();   // adds vptr (8 bytes on 64-bit)
    int x;                // 4 bytes + 4 padding
};                        // sizeof(WithVirtual) == 16
```

---

### Constructors

Constructors are special member functions that initialize objects. They have the same name as the class, no return type, and are called automatically when an object is created.

#### Default Constructor

A constructor with no parameters (or all parameters have defaults).

```cpp
class Widget {
    int value;
    string name;
public:
    // User-defined default constructor
    Widget() : value(0), name("unnamed") {
        cout << "Default constructor called" << endl;
    }
};

Widget w;  // default constructor called
```

**Compiler-generated default constructor:**
- If you define NO constructors at all, the compiler generates a default constructor
- It default-initializes all members (calls their default constructors)
- Primitive types (int, double, etc.) are left UNINITIALIZED (garbage values!)
- Once you define ANY constructor, the compiler no longer generates a default one

```cpp
class NoDefault {
public:
    NoDefault(int x) {}  // parameterized constructor defined
};

// NoDefault obj;  // ERROR: no default constructor available

// To bring it back explicitly:
class WithDefault {
public:
    WithDefault() = default;  // explicitly request compiler-generated default
    WithDefault(int x) {}
};
```

**`= default` vs `= delete`:**
```cpp
class NonCopyable {
public:
    NonCopyable() = default;                          // use compiler-generated
    NonCopyable(const NonCopyable&) = delete;         // prevent copying
    NonCopyable& operator=(const NonCopyable&) = delete;  // prevent copy assignment
};
```

---

#### Parameterized Constructor

Takes arguments to initialize the object with specific values.

```cpp
class Rectangle {
    double width, height;
public:
    // Parameterized constructor
    Rectangle(double w, double h) : width(w), height(h) {
        if (w <= 0 || h <= 0)
            throw invalid_argument("Dimensions must be positive");
    }

    double area() const { return width * height; }
};

Rectangle r(5.0, 3.0);  // parameterized constructor
```

---

#### Member Initializer List

The preferred way to initialize members — happens BEFORE the constructor body executes.

```cpp
class Student {
    const int id;           // must be initialized in initializer list
    string& schoolRef;      // references must be initialized in initializer list
    string name;
    vector<int> grades;

public:
    // Member initializer list (after colon, before body)
    Student(int id, string& school, const string& name)
        : id(id),              // const members MUST be here
          schoolRef(school),   // references MUST be here
          name(name),          // more efficient than assignment in body
          grades()             // explicitly default-construct
    {
        // Constructor body — runs AFTER all members are initialized
        cout << "Student " << name << " created" << endl;
    }
};
```

**Why use initializer list over assignment in body?**
```cpp
class Expensive {
    string data;
public:
    // BAD: default-constructs data, then assigns (two operations)
    Expensive(const string& d) {
        data = d;  // default ctor + copy assignment
    }

    // GOOD: directly constructs data with value (one operation)
    Expensive(const string& d) : data(d) {}  // copy ctor only
};
```

**MUST use initializer list for:**
- `const` members
- Reference members
- Members without default constructors
- Base class constructors

**Initialization order:** Members are initialized in the order they are DECLARED in the class, NOT the order in the initializer list.

```cpp
class Gotcha {
    int a;
    int b;
public:
    Gotcha(int val) : b(val), a(b) {}  // WARNING: a is initialized FIRST (undefined!)
    // a is declared before b, so a is initialized first
    // But a(b) uses b which hasn't been initialized yet!
};
```

---

#### Copy Constructor

Creates a new object as a copy of an existing object.

```cpp
class DynamicArray {
    int* data;
    int size;

public:
    DynamicArray(int sz) : size(sz), data(new int[sz]()) {}

    // SHALLOW COPY (default behavior — DANGEROUS with raw pointers)
    // DynamicArray(const DynamicArray& other) = default;
    // This would copy the pointer, not the data — two objects share same memory!

    // DEEP COPY (correct for classes managing resources)
    DynamicArray(const DynamicArray& other) : size(other.size) {
        data = new int[size];
        for (int i = 0; i < size; i++) {
            data[i] = other.data[i];  // copy each element
        }
    }

    // Copy Assignment Operator (also needed — Rule of Three)
    DynamicArray& operator=(const DynamicArray& other) {
        if (this == &other) return *this;  // self-assignment check

        delete[] data;  // free old memory

        size = other.size;
        data = new int[size];
        for (int i = 0; i < size; i++) {
            data[i] = other.data[i];
        }
        return *this;
    }

    ~DynamicArray() { delete[] data; }
};
```

**When is the copy constructor called?**
```cpp
DynamicArray a(10);
DynamicArray b = a;        // 1. Copy initialization
DynamicArray c(a);         // 2. Direct copy initialization
void func(DynamicArray x); // 3. Pass by value
func(a);                   // copy constructor called for parameter x
DynamicArray getArr() { DynamicArray temp(5); return temp; }  // 4. Return by value (may be elided)
```

**Shallow vs Deep Copy:**
```
SHALLOW COPY (default):
┌─────────┐     ┌─────────────────┐
│ obj1    │────→│ [1][2][3][4][5] │
└─────────┘     └─────────────────┘
                        ↑
┌─────────┐             │
│ obj2    │─────────────┘  (SAME memory! Double-free bug!)
└─────────┘

DEEP COPY (correct):
┌─────────┐     ┌─────────────────┐
│ obj1    │────→│ [1][2][3][4][5] │
└─────────┘     └─────────────────┘

┌─────────┐     ┌─────────────────┐
│ obj2    │────→│ [1][2][3][4][5] │  (separate memory, same values)
└─────────┘     └─────────────────┘
```

---

#### Move Constructor (C++11)

Transfers ownership of resources from a temporary (rvalue) to a new object — avoids expensive deep copies.

```cpp
class Buffer {
    int* data;
    size_t size;

public:
    Buffer(size_t sz) : size(sz), data(new int[sz]()) {
        cout << "Constructed (allocated " << sz << " ints)" << endl;
    }

    // Copy constructor (expensive — allocates + copies)
    Buffer(const Buffer& other) : size(other.size), data(new int[other.size]) {
        copy(other.data, other.data + size, data);
        cout << "Copied (allocated " << size << " ints)" << endl;
    }

    // Move constructor (cheap — just pointer swap)
    Buffer(Buffer&& other) noexcept : data(other.data), size(other.size) {
        other.data = nullptr;  // source must be left in valid state
        other.size = 0;
        cout << "Moved (no allocation!)" << endl;
    }

    // Move Assignment Operator
    Buffer& operator=(Buffer&& other) noexcept {
        if (this == &other) return *this;

        delete[] data;         // free current resource

        data = other.data;     // steal from source
        size = other.size;

        other.data = nullptr;  // leave source in valid state
        other.size = 0;

        return *this;
    }

    ~Buffer() { delete[] data; }
};

// Usage
Buffer createBuffer() {
    Buffer temp(1000000);
    return temp;  // move constructor (or copy elision)
}

Buffer b1(100);
Buffer b2 = std::move(b1);  // explicit move — b1 is now empty
Buffer b3 = createBuffer();  // move from temporary
```

**Key points about move semantics:**
- `std::move()` doesn't move anything — it casts to an rvalue reference
- After being moved from, the object must be in a valid but unspecified state
- Mark move operations `noexcept` — enables optimizations in STL containers
- Move is typically O(1), copy is O(n)

---

#### Delegating Constructors (C++11)

One constructor can call another constructor of the same class.

```cpp
class Connection {
    string host;
    int port;
    bool secure;

public:
    // Primary constructor
    Connection(const string& h, int p, bool s) : host(h), port(p), secure(s) {}

    // Delegating constructors
    Connection(const string& h, int p) : Connection(h, p, false) {}  // delegates
    Connection(const string& h) : Connection(h, 80, false) {}        // delegates
    Connection() : Connection("localhost", 80, false) {}              // delegates
};
```

---

#### Explicit Constructors

Prevents implicit conversions via single-argument constructors.

```cpp
class Fraction {
    int num, den;
public:
    // Without explicit: allows implicit conversion from int
    // Fraction(int n) : num(n), den(1) {}
    // Fraction f = 5;  // compiles! (implicit conversion)

    // With explicit: prevents implicit conversion
    explicit Fraction(int n, int d = 1) : num(n), den(d) {}
};

// Fraction f = 5;           // ERROR: implicit conversion not allowed
Fraction f(5);               // OK: direct initialization
Fraction f2 = Fraction(5);   // OK: explicit construction
```

**Rule:** Mark single-argument constructors `explicit` unless you intentionally want implicit conversions.

---

#### Rule of Zero, Three, and Five

**Rule of Three:** If you define any of destructor, copy constructor, or copy assignment operator — define all three.

**Rule of Five (C++11):** Add move constructor and move assignment operator to the Rule of Three.

**Rule of Zero (preferred):** Don't define any of the five — use smart pointers and RAII containers that handle resources for you.

```cpp
// Rule of Zero — BEST approach in modern C++
class ModernClass {
    string name;                    // manages its own memory
    vector<int> data;               // manages its own memory
    unique_ptr<Resource> resource;  // manages its own memory
    // No destructor, no copy/move operations needed!
    // Compiler-generated ones do the right thing.
};

// Rule of Five — when you MUST manage raw resources
class RawResourceClass {
    int* data;
    size_t size;
public:
    RawResourceClass(size_t sz);                                    // constructor
    ~RawResourceClass();                                            // 1. destructor
    RawResourceClass(const RawResourceClass& other);                // 2. copy ctor
    RawResourceClass& operator=(const RawResourceClass& other);     // 3. copy assign
    RawResourceClass(RawResourceClass&& other) noexcept;            // 4. move ctor
    RawResourceClass& operator=(RawResourceClass&& other) noexcept; // 5. move assign
};
```

---

### Destructor

The destructor is called when an object's lifetime ends. It performs cleanup — releasing memory, closing files, disconnecting sockets, etc.

```cpp
class DatabaseConnection {
    Connection* conn;
    string connString;

public:
    DatabaseConnection(const string& cs) : connString(cs) {
        conn = openConnection(cs);
        cout << "Connected to " << cs << endl;
    }

    ~DatabaseConnection() {
        if (conn) {
            closeConnection(conn);
            conn = nullptr;
        }
        cout << "Disconnected from " << connString << endl;
    }
};
```

**When is the destructor called?**
```cpp
void example() {
    DatabaseConnection db("localhost");  // constructor called

    {
        DatabaseConnection db2("remote");  // constructor called
    }  // db2 destructor called here (end of inner scope)

    DatabaseConnection* db3 = new DatabaseConnection("cloud");
    delete db3;  // destructor called explicitly via delete

    // If you forget delete — MEMORY LEAK (destructor never called)

}  // db destructor called here (end of function scope)
```

**Destruction order:**
1. Destructor body executes
2. Member objects are destroyed (in reverse order of declaration)
3. Base class destructor is called

**Virtual Destructor — CRITICAL for polymorphism:**
```cpp
class Base {
public:
    ~Base() { cout << "Base destroyed" << endl; }  // NON-virtual — BUG!
};

class Derived : public Base {
    int* data;
public:
    Derived() : data(new int[100]) {}
    ~Derived() { delete[] data; cout << "Derived destroyed" << endl; }
};

Base* ptr = new Derived();
delete ptr;  // ONLY calls Base::~Base()! Derived::~Derived() is NEVER called!
             // Memory leak! data is never freed!

// FIX: Make base destructor virtual
class Base {
public:
    virtual ~Base() { cout << "Base destroyed" << endl; }
};
// Now delete ptr correctly calls Derived::~Derived() then Base::~Base()
```

**Rule:** If a class has ANY virtual function, its destructor MUST be virtual.

---

### The `this` Pointer

Every non-static member function has an implicit hidden parameter: a pointer to the object on which the function was called.

```cpp
class Node {
    int value;
    Node* next;

public:
    Node(int v) : value(v), next(nullptr) {}

    // 1. Disambiguate member from parameter
    void setValue(int value) {
        this->value = value;  // this->value is the member, value is the parameter
    }

    // 2. Return *this for method chaining (fluent interface)
    Node& setNext(Node* n) {
        this->next = n;
        return *this;
    }

    // 3. Compare with another object
    bool isSameAs(const Node* other) const {
        return this == other;  // compare addresses
    }

    // 4. Pass this to external function
    void registerSelf(Registry& reg) {
        reg.add(this);
    }

    // 5. Prevent self-assignment
    Node& operator=(const Node& other) {
        if (this == &other) return *this;  // self-assignment guard
        value = other.value;
        next = other.next;
        return *this;
    }
};

// Method chaining in action
Node n1(1), n2(2), n3(3);
n1.setNext(&n2).setValue(10);  // chained calls
```

**Type of `this`:**
- In non-const method: `ClassName* const this` (pointer is const, object is not)
- In const method: `const ClassName* const this` (both pointer and object are const)
- `this` is an rvalue — you cannot take its address (`&this` is illegal)

---

### Static Members (Variables and Functions)

Static members belong to the class, not to any instance. There is exactly ONE copy shared by all objects.

#### Static Variables

```cpp
class Logger {
    static int instanceCount;       // declaration
    static vector<string> logs;     // declaration
    string name;

public:
    Logger(const string& n) : name(n) { instanceCount++; }
    ~Logger() { instanceCount--; }

    void log(const string& msg) {
        logs.push_back("[" + name + "] " + msg);
    }

    static int getInstanceCount() { return instanceCount; }
    static const vector<string>& getLogs() { return logs; }
    static void clearLogs() { logs.clear(); }
};

// MUST define static members outside the class (in .cpp file)
int Logger::instanceCount = 0;
vector<string> Logger::logs;

// Usage
Logger l1("App"), l2("DB");
l1.log("Started");
l2.log("Connected");
cout << Logger::getInstanceCount();  // 2
cout << Logger::getLogs().size();     // 2
```

**Inline static members (C++17):**
```cpp
class Config {
public:
    inline static int maxRetries = 3;          // definition right here!
    inline static string appName = "MyApp";    // no separate .cpp definition needed
};
```

#### Static Functions

```cpp
class MathUtils {
public:
    static double PI() { return 3.14159265358979; }

    static int factorial(int n) {
        if (n <= 1) return 1;
        return n * factorial(n - 1);
    }

    static bool isPrime(int n) {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++)
            if (n % i == 0) return false;
        return true;
    }
};

// Called on the class, not on an object
double area = MathUtils::PI() * r * r;
int fact5 = MathUtils::factorial(5);
```

**Static function restrictions:**
- Cannot access non-static members (no `this` pointer)
- Cannot be `const` (const applies to `this`, which doesn't exist)
- Cannot be `virtual` (virtual dispatch requires an object)
- CAN access other static members and functions

**Common use cases for static:**
- Object counters
- Shared configuration
- Factory methods
- Utility/helper functions
- Singleton pattern implementation

---

### Const Member Functions

A `const` member function guarantees it will not modify the object's observable state.

```cpp
class Matrix {
    vector<vector<double>> data;
    int rows, cols;

public:
    Matrix(int r, int c) : rows(r), cols(c), data(r, vector<double>(c, 0.0)) {}

    // Const member functions — can be called on const objects
    int getRows() const { return rows; }
    int getCols() const { return cols; }
    double get(int r, int c) const { return data[r][c]; }

    bool isSquare() const { return rows == cols; }

    double determinant() const {  // complex computation, but doesn't modify
        // ... calculation ...
        return 0.0;
    }

    // Non-const — modifies the object
    void set(int r, int c, double val) { data[r][c] = val; }
    void transpose() { /* modifies data */ }

    // Const overloading — different behavior for const vs non-const objects
    double& operator()(int r, int c) { return data[r][c]; }              // non-const version
    const double& operator()(int r, int c) const { return data[r][c]; }  // const version
};

void printMatrix(const Matrix& m) {  // receives const reference
    // m.set(0, 0, 5);  // ERROR: cannot call non-const method on const object
    cout << m.get(0, 0);  // OK: get() is const
    cout << m(0, 0);      // calls const version of operator()
}
```

**Const correctness rules:**
- If a method doesn't modify the object → mark it `const`
- `const` objects can only call `const` member functions
- `const` references/pointers can only call `const` member functions
- Returning by non-const reference from a const function is not allowed

---

### Mutable Keyword

`mutable` exempts a member from the const restriction. It can be modified even in `const` member functions.

```cpp
class CachedComputation {
    vector<int> data;
    mutable bool cacheValid = false;    // mutable!
    mutable double cachedResult = 0.0;  // mutable!
    mutable mutex mtx;                  // mutable! (for thread-safe const methods)

public:
    CachedComputation(const vector<int>& d) : data(d) {}

    void setData(const vector<int>& d) {
        data = d;
        cacheValid = false;  // invalidate cache
    }

    double computeAverage() const {  // const method!
        lock_guard<mutex> lock(mtx);  // OK: mtx is mutable

        if (!cacheValid) {
            // Expensive computation — cache the result
            double sum = 0;
            for (int x : data) sum += x;
            cachedResult = sum / data.size();  // OK: mutable member
            cacheValid = true;                  // OK: mutable member
        }
        return cachedResult;
    }
};
```

**Legitimate uses of `mutable`:**
- Caching/memoization (lazy evaluation)
- Mutexes in const methods (thread safety)
- Access counters / debugging instrumentation
- Internal bookkeeping that doesn't affect observable state

**Abuse of `mutable`:** If you're making many members mutable, your const design is probably wrong.

---

### Friend Functions and Friend Classes

The `friend` keyword grants a non-member function or another class access to private and protected members.

#### Friend Functions

```cpp
class Vector3D {
    double x, y, z;
public:
    Vector3D(double x, double y, double z) : x(x), y(y), z(z) {}

    // Friend function declaration
    friend double dotProduct(const Vector3D& a, const Vector3D& b);
    friend Vector3D crossProduct(const Vector3D& a, const Vector3D& b);
    friend ostream& operator<<(ostream& os, const Vector3D& v);
};

// Friend function definitions — NOT members of Vector3D
double dotProduct(const Vector3D& a, const Vector3D& b) {
    return a.x * b.x + a.y * b.y + a.z * b.z;  // accesses private members
}

Vector3D crossProduct(const Vector3D& a, const Vector3D& b) {
    return Vector3D(
        a.y * b.z - a.z * b.y,
        a.z * b.x - a.x * b.z,
        a.x * b.y - a.y * b.x
    );
}

ostream& operator<<(ostream& os, const Vector3D& v) {
    os << "(" << v.x << ", " << v.y << ", " << v.z << ")";
    return os;
}
```

#### Friend Classes

```cpp
class LinkedList {
    struct Node {
        int data;
        Node* next;
    };
    Node* head;

    friend class LinkedListIterator;  // iterator needs access to Node

public:
    LinkedList() : head(nullptr) {}
    void push_front(int val) {
        Node* n = new Node{val, head};
        head = n;
    }
};

class LinkedListIterator {
    LinkedList::Node* current;  // can access private Node type
public:
    LinkedListIterator(LinkedList& list) : current(list.head) {}  // accesses private head
    bool hasNext() const { return current != nullptr; }
    int next() {
        int val = current->data;
        current = current->next;
        return val;
    }
};
```

#### Friend Member Function (specific method of another class)

```cpp
class B;  // forward declaration

class A {
public:
    void showB(const B& b);  // will access B's private members
};

class B {
    int secret = 42;
    friend void A::showB(const B& b);  // only A::showB is a friend, not all of A
public:
    B() = default;
};

void A::showB(const B& b) {
    cout << b.secret;  // OK: this specific method is a friend of B
}
```

**Properties of friendship:**
| Property | Explanation |
|----------|-------------|
| Not inherited | If Base is a friend of X, Derived is NOT automatically a friend of X |
| Not mutual | A declares B as friend → B can access A's privates, but A cannot access B's |
| Not transitive | A friends B, B friends C → A does NOT have access to C |
| Declared in class | The class being accessed declares who its friends are |

**When to use friend:**
- Operator overloading (especially `<<`, `>>`, binary operators)
- Tightly coupled helper classes (iterators, builders)
- Unit testing (test classes as friends)
- Avoid when possible — it weakens encapsulation

---


## 2.2 Encapsulation

> Encapsulation is the first pillar of OOP. It means bundling data and the methods that operate on that data into a single unit (class), and restricting direct access to the internal state. The goal is to protect the integrity of the object's data and hide implementation details.

---

### Data Hiding

**Principle:** The internal representation of an object should be hidden from the outside. Only a well-defined interface (public methods) should be exposed.

**Why data hiding matters:**

```cpp
// BAD: No encapsulation — data is public
class BankAccountBad {
public:
    double balance;
    string owner;
};

BankAccountBad acc;
acc.balance = -1000000;  // No validation! Invalid state!
acc.balance = acc.balance * 2;  // Anyone can manipulate directly

// GOOD: Proper encapsulation
class BankAccount {
private:
    double balance;
    string owner;
    string accountNumber;
    vector<Transaction> history;

    void recordTransaction(const string& type, double amount) {
        history.push_back({type, amount, time(nullptr)});
    }

public:
    BankAccount(const string& owner, const string& accNum, double initialDeposit = 0)
        : owner(owner), accountNumber(accNum), balance(0) {
        if (initialDeposit > 0) deposit(initialDeposit);
    }

    bool deposit(double amount) {
        if (amount <= 0) return false;
        balance += amount;
        recordTransaction("DEPOSIT", amount);
        return true;
    }

    bool withdraw(double amount) {
        if (amount <= 0) return false;
        if (amount > balance) return false;  // insufficient funds
        balance -= amount;
        recordTransaction("WITHDRAWAL", amount);
        return true;
    }

    bool transfer(BankAccount& to, double amount) {
        if (withdraw(amount)) {
            to.deposit(amount);
            return true;
        }
        return false;
    }

    double getBalance() const { return balance; }
    string getOwner() const { return owner; }
};
```

**Benefits of data hiding:**
1. **Validation:** All modifications go through methods that validate input
2. **Consistency:** Object is always in a valid state
3. **Flexibility:** Internal representation can change without breaking client code
4. **Debugging:** All state changes go through known entry points
5. **Security:** Sensitive data cannot be accessed directly

**Implementation change without breaking clients:**
```cpp
// Version 1: balance stored as double
class Account_v1 {
    double balance;  // in dollars
public:
    double getBalance() const { return balance; }
};

// Version 2: balance stored as cents (int) for precision
class Account_v2 {
    long long balanceCents;  // in cents — avoids floating point issues
public:
    double getBalance() const { return balanceCents / 100.0; }  // same interface!
};
// Client code doesn't change at all!
```

---

### Getters and Setters

Getters (accessors) and setters (mutators) provide controlled access to private data.

```cpp
class Person {
    string firstName;
    string lastName;
    int age;
    string email;

public:
    Person(const string& first, const string& last, int age, const string& email)
        : firstName(first), lastName(last), age(age), email(email) {
        validateAge(age);
        validateEmail(email);
    }

    // --- Getters ---
    const string& getFirstName() const { return firstName; }
    const string& getLastName() const { return lastName; }
    string getFullName() const { return firstName + " " + lastName; }  // computed
    int getAge() const { return age; }
    const string& getEmail() const { return email; }

    // --- Setters with validation ---
    void setAge(int newAge) {
        validateAge(newAge);
        age = newAge;
    }

    void setEmail(const string& newEmail) {
        validateEmail(newEmail);
        email = newEmail;
    }

    // Name change might require additional business logic
    void changeName(const string& first, const string& last) {
        if (first.empty() || last.empty())
            throw invalid_argument("Name cannot be empty");
        firstName = first;
        lastName = last;
    }

private:
    void validateAge(int a) {
        if (a < 0 || a > 150)
            throw invalid_argument("Invalid age: " + to_string(a));
    }

    void validateEmail(const string& e) {
        if (e.find('@') == string::npos)
            throw invalid_argument("Invalid email: " + e);
    }
};
```

**Getter return types — choosing correctly:**
```cpp
class Container {
    vector<int> items;
    string name;

public:
    // Return by const reference — efficient, no copy, read-only access
    const vector<int>& getItems() const { return items; }
    const string& getName() const { return name; }

    // Return by value — for small types or when you want a copy
    int getSize() const { return items.size(); }
    bool isEmpty() const { return items.empty(); }

    // NEVER return non-const reference to private data (breaks encapsulation!)
    // vector<int>& getItems() { return items; }  // BAD! Allows direct modification
};
```

**When NOT to use getters/setters:**
- Don't add them reflexively for every field
- If a field should never be read externally → no getter
- If a field should never change → no setter (or make it const)
- Prefer behavior-rich methods over raw data access

```cpp
// BAD: Anemic class with getters/setters for everything
class RectangleBad {
    double width, height;
public:
    double getWidth() const { return width; }
    double getHeight() const { return height; }
    void setWidth(double w) { width = w; }
    void setHeight(double h) { height = h; }
};
// Client does: rect.setWidth(rect.getWidth() * 2);  // logic is OUTSIDE the class

// GOOD: Behavior-rich class
class Rectangle {
    double width, height;
public:
    Rectangle(double w, double h) : width(w), height(h) {}
    double area() const { return width * height; }
    double perimeter() const { return 2 * (width + height); }
    void scale(double factor) { width *= factor; height *= factor; }
    void resize(double w, double h) {
        if (w <= 0 || h <= 0) throw invalid_argument("Dimensions must be positive");
        width = w; height = h;
    }
};
```

---

### Invariant Enforcement

A **class invariant** is a logical condition that must be true for every object of the class, at all times between method calls. The constructor establishes the invariant, and every public method must preserve it.

```cpp
class SortedArray {
    vector<int> data;  // INVARIANT: data is always sorted in ascending order

    bool isSorted() const {
        for (size_t i = 1; i < data.size(); i++)
            if (data[i] < data[i-1]) return false;
        return true;
    }

public:
    SortedArray() = default;  // empty array is trivially sorted ✓

    void insert(int value) {
        // Insert in correct position to maintain sorted invariant
        auto pos = lower_bound(data.begin(), data.end(), value);
        data.insert(pos, value);
        // Invariant preserved: data is still sorted ✓
    }

    bool contains(int value) const {
        // Can use binary search because invariant guarantees sorted order
        return binary_search(data.begin(), data.end(), value);
    }

    void remove(int value) {
        auto pos = lower_bound(data.begin(), data.end(), value);
        if (pos != data.end() && *pos == value) {
            data.erase(pos);
        }
        // Invariant preserved: removing from sorted array keeps it sorted ✓
    }

    int getMin() const {
        if (data.empty()) throw runtime_error("Empty array");
        return data.front();  // guaranteed to be minimum due to invariant
    }

    int getMax() const {
        if (data.empty()) throw runtime_error("Empty array");
        return data.back();   // guaranteed to be maximum due to invariant
    }
};
```

**Another example — Circular Buffer with invariants:**
```cpp
class CircularBuffer {
    vector<int> buffer;
    int head = 0;       // index of first element
    int tail = 0;       // index of next write position
    int count = 0;      // number of elements
    int capacity;

    // INVARIANTS:
    // 1. 0 <= count <= capacity
    // 2. 0 <= head < capacity
    // 3. 0 <= tail < capacity
    // 4. tail == (head + count) % capacity

public:
    explicit CircularBuffer(int cap) : capacity(cap), buffer(cap) {
        if (cap <= 0) throw invalid_argument("Capacity must be positive");
        // All invariants satisfied: count=0, head=0, tail=0
    }

    void push(int value) {
        if (count == capacity) throw overflow_error("Buffer full");
        buffer[tail] = value;
        tail = (tail + 1) % capacity;
        count++;
        // All invariants maintained
    }

    int pop() {
        if (count == 0) throw underflow_error("Buffer empty");
        int value = buffer[head];
        head = (head + 1) % capacity;
        count--;
        return value;
        // All invariants maintained
    }

    bool isFull() const { return count == capacity; }
    bool isEmpty() const { return count == 0; }
    int size() const { return count; }
};
```

**Strategies for invariant enforcement:**
1. **Validate in constructor** — establish invariant from the start
2. **Validate in setters/mutators** — check before modifying
3. **Make invalid states unrepresentable** — use types that prevent violations
4. **Use assertions in debug builds** — `assert(invariant())` at end of each method

---

### Immutable Objects in C++

An **immutable object** cannot be modified after construction. All its state is fixed at creation time.

```cpp
class ImmutableConfig {
    const string host;
    const int port;
    const bool useTLS;
    const int timeout;
    const vector<string> allowedOrigins;

public:
    ImmutableConfig(const string& host, int port, bool tls, int timeout,
                    const vector<string>& origins)
        : host(host), port(port), useTLS(tls), timeout(timeout),
          allowedOrigins(origins) {}

    // Only getters — no setters
    const string& getHost() const { return host; }
    int getPort() const { return port; }
    bool isSecure() const { return useTLS; }
    int getTimeout() const { return timeout; }
    const vector<string>& getAllowedOrigins() const { return allowedOrigins; }

    // "Modification" returns a new object (like functional programming)
    ImmutableConfig withPort(int newPort) const {
        return ImmutableConfig(host, newPort, useTLS, timeout, allowedOrigins);
    }

    ImmutableConfig withTimeout(int newTimeout) const {
        return ImmutableConfig(host, port, useTLS, newTimeout, allowedOrigins);
    }
};

// Usage
ImmutableConfig config("api.example.com", 443, true, 30, {"*.example.com"});
auto devConfig = config.withPort(8080).withTimeout(60);  // new object, original unchanged
```

**Builder pattern for immutable objects:**
```cpp
class ImmutableUser {
    const string name;
    const string email;
    const int age;
    const string role;

    // Private constructor — only Builder can create
    ImmutableUser(const string& n, const string& e, int a, const string& r)
        : name(n), email(e), age(a), role(r) {}

public:
    class Builder {
        string name, email, role = "user";
        int age = 0;
    public:
        Builder& setName(const string& n) { name = n; return *this; }
        Builder& setEmail(const string& e) { email = e; return *this; }
        Builder& setAge(int a) { age = a; return *this; }
        Builder& setRole(const string& r) { role = r; return *this; }

        ImmutableUser build() const {
            if (name.empty() || email.empty())
                throw invalid_argument("Name and email are required");
            return ImmutableUser(name, email, age, role);
        }
    };

    const string& getName() const { return name; }
    const string& getEmail() const { return email; }
    int getAge() const { return age; }
    const string& getRole() const { return role; }
};

auto user = ImmutableUser::Builder()
    .setName("Alice")
    .setEmail("alice@example.com")
    .setAge(30)
    .build();
```

**Benefits of immutability:**
- **Thread-safe:** No synchronization needed (no writes)
- **Predictable:** State never changes unexpectedly
- **Safe sharing:** Can be freely shared between components
- **Hashable:** Can be used as keys in hash maps (hash never changes)
- **Easy to reason about:** No temporal coupling

---


## 2.3 Inheritance

> Inheritance is the mechanism by which one class (derived/child) acquires the properties and behaviors of another class (base/parent). It models "is-a" relationships and enables code reuse and polymorphism.

---

### Single Inheritance

The simplest form — one derived class inherits from exactly one base class.

```cpp
class Vehicle {
protected:
    string make;
    string model;
    int year;
    double fuelLevel;

public:
    Vehicle(const string& make, const string& model, int year)
        : make(make), model(model), year(year), fuelLevel(100.0) {}

    virtual void start() {
        cout << year << " " << make << " " << model << " started." << endl;
    }

    virtual void stop() {
        cout << make << " " << model << " stopped." << endl;
    }

    void refuel(double amount) {
        fuelLevel = min(100.0, fuelLevel + amount);
    }

    double getFuelLevel() const { return fuelLevel; }
    string getInfo() const { return to_string(year) + " " + make + " " + model; }

    virtual ~Vehicle() = default;
};

class Car : public Vehicle {
    int numDoors;
    bool isConvertible;

public:
    Car(const string& make, const string& model, int year, int doors, bool convertible)
        : Vehicle(make, model, year),  // call base constructor
          numDoors(doors), isConvertible(convertible) {}

    void start() override {
        cout << "Turning key... ";
        Vehicle::start();  // call base version
    }

    void openRoof() {
        if (isConvertible) cout << "Roof opened!" << endl;
        else cout << "This car is not a convertible." << endl;
    }
};

class Truck : public Vehicle {
    double payloadCapacity;  // in tons

public:
    Truck(const string& make, const string& model, int year, double payload)
        : Vehicle(make, model, year), payloadCapacity(payload) {}

    void start() override {
        cout << "Diesel engine warming up... ";
        Vehicle::start();
    }

    void loadCargo(double tons) {
        if (tons <= payloadCapacity)
            cout << "Loaded " << tons << " tons." << endl;
        else
            cout << "Exceeds capacity of " << payloadCapacity << " tons!" << endl;
    }
};
```

**What the derived class gets:**
- All public and protected members of the base
- Does NOT get: private members (they exist but are inaccessible), constructors, destructor, assignment operator

**What the derived class can do:**
- Add new members (data and functions)
- Override virtual functions
- Call base class methods using `Base::method()`
- Hide base class non-virtual functions (not recommended)

---

### Multiple Inheritance

A class inherits from two or more base classes simultaneously.

```cpp
// Interface-like classes (safe for multiple inheritance)
class Flyable {
public:
    virtual void fly() = 0;
    virtual double getAltitude() const = 0;
    virtual ~Flyable() = default;
};

class Swimmable {
public:
    virtual void swim() = 0;
    virtual double getDepth() const = 0;
    virtual ~Swimmable() = default;
};

class Walkable {
public:
    virtual void walk() = 0;
    virtual double getSpeed() const = 0;
    virtual ~Walkable() = default;
};

// Duck can fly, swim, and walk
class Duck : public Flyable, public Swimmable, public Walkable {
    double altitude = 0;
    double depth = 0;
    double speed = 0;

public:
    void fly() override { altitude = 100; cout << "Duck flying!" << endl; }
    double getAltitude() const override { return altitude; }

    void swim() override { depth = 2; cout << "Duck swimming!" << endl; }
    double getDepth() const override { return depth; }

    void walk() override { speed = 3; cout << "Duck walking!" << endl; }
    double getSpeed() const override { return speed; }
};
```

**Ambiguity in multiple inheritance:**
```cpp
class A {
public:
    void doSomething() { cout << "A::doSomething" << endl; }
};

class B {
public:
    void doSomething() { cout << "B::doSomething" << endl; }
};

class C : public A, public B {};

C obj;
// obj.doSomething();    // ERROR: ambiguous! Which one?
obj.A::doSomething();    // OK: explicitly specify
obj.B::doSomething();    // OK: explicitly specify
```

**Best practices for multiple inheritance:**
- Use it primarily for inheriting from multiple interfaces (pure virtual classes)
- Avoid inheriting from multiple classes with data members
- If you must, use virtual inheritance to avoid the diamond problem

---

### Multilevel Inheritance

A chain where each class inherits from the one above it.

```cpp
class Shape {
protected:
    string color;
public:
    Shape(const string& c = "black") : color(c) {}
    virtual double area() const = 0;
    virtual void describe() const {
        cout << "Shape [color=" << color << "]" << endl;
    }
    virtual ~Shape() = default;
};

class Polygon : public Shape {
protected:
    int numSides;
public:
    Polygon(int sides, const string& c = "black") : Shape(c), numSides(sides) {}
    int getSides() const { return numSides; }
    void describe() const override {
        cout << "Polygon [sides=" << numSides << ", color=" << color << "]" << endl;
    }
};

class RegularPolygon : public Polygon {
protected:
    double sideLength;
public:
    RegularPolygon(int sides, double length, const string& c = "black")
        : Polygon(sides, c), sideLength(length) {}

    double perimeter() const { return numSides * sideLength; }

    void describe() const override {
        cout << "RegularPolygon [sides=" << numSides
             << ", length=" << sideLength
             << ", color=" << color << "]" << endl;
    }
};

class Square : public RegularPolygon {
public:
    Square(double side, const string& c = "black") : RegularPolygon(4, side, c) {}

    double area() const override { return sideLength * sideLength; }

    void describe() const override {
        cout << "Square [side=" << sideLength << ", color=" << color << "]" << endl;
    }
};

// Inheritance chain: Shape → Polygon → RegularPolygon → Square
```

**Depth considerations:**
- Deep hierarchies (>3-4 levels) become hard to understand and maintain
- Prefer composition over deep inheritance chains
- Each level should add meaningful specialization

---

### Hierarchical Inheritance

Multiple derived classes inherit from the same base class (tree structure).

```cpp
class Employee {
protected:
    string name;
    int id;
    double baseSalary;

public:
    Employee(const string& n, int id, double salary)
        : name(n), id(id), baseSalary(salary) {}

    virtual double calculatePay() const = 0;
    virtual string getRole() const = 0;

    string getName() const { return name; }
    int getId() const { return id; }

    virtual void displayInfo() const {
        cout << getRole() << ": " << name << " (ID: " << id << ")" << endl;
        cout << "  Pay: $" << calculatePay() << endl;
    }

    virtual ~Employee() = default;
};

class FullTimeEmployee : public Employee {
    double bonus;
public:
    FullTimeEmployee(const string& n, int id, double salary, double bonus)
        : Employee(n, id, salary), bonus(bonus) {}

    double calculatePay() const override { return baseSalary + bonus; }
    string getRole() const override { return "Full-Time"; }
};

class PartTimeEmployee : public Employee {
    int hoursWorked;
    double hourlyRate;
public:
    PartTimeEmployee(const string& n, int id, int hours, double rate)
        : Employee(n, id, 0), hoursWorked(hours), hourlyRate(rate) {}

    double calculatePay() const override { return hoursWorked * hourlyRate; }
    string getRole() const override { return "Part-Time"; }
};

class Contractor : public Employee {
    int projectDays;
    double dailyRate;
public:
    Contractor(const string& n, int id, int days, double rate)
        : Employee(n, id, 0), projectDays(days), dailyRate(rate) {}

    double calculatePay() const override { return projectDays * dailyRate; }
    string getRole() const override { return "Contractor"; }
};

// Polymorphic usage
void processPayroll(const vector<Employee*>& employees) {
    double total = 0;
    for (const auto* emp : employees) {
        emp->displayInfo();
        total += emp->calculatePay();
    }
    cout << "Total payroll: $" << total << endl;
}
```

---

### Hybrid Inheritance

A combination of multiple inheritance types. Most commonly leads to the diamond problem.

```cpp
//         Animal
//        /      \
//    Mammal    Bird
//        \      /
//         Bat

class Animal {
protected:
    string name;
    int age;
public:
    Animal(const string& n, int a) : name(n), age(a) {
        cout << "Animal(" << n << ") constructed" << endl;
    }
    virtual void describe() const {
        cout << name << ", age " << age << endl;
    }
    virtual ~Animal() { cout << "Animal(" << name << ") destroyed" << endl; }
};

class Mammal : public Animal {  // WITHOUT virtual — causes diamond problem
    bool hasFur;
public:
    Mammal(const string& n, int a, bool fur) : Animal(n, a), hasFur(fur) {}
    void nurse() { cout << name << " is nursing young" << endl; }
};

class Bird : public Animal {  // WITHOUT virtual — causes diamond problem
    double wingspan;
public:
    Bird(const string& n, int a, double ws) : Animal(n, a), wingspan(ws) {}
    void layEggs() { cout << name << " is laying eggs" << endl; }
};

class Bat : public Mammal, public Bird {
public:
    // PROBLEM: Must construct Animal TWICE (once for Mammal, once for Bird)
    Bat(const string& n, int a, bool fur, double ws)
        : Mammal(n, a, fur), Bird(n, a, ws) {}

    // PROBLEM: Ambiguous access
    // void show() { cout << name; }  // ERROR: which name? Mammal::name or Bird::name?
    void show() { cout << Mammal::name; }  // must qualify
};
```

---

### Diamond Problem & Virtual Inheritance

**The Problem:**
```
Without virtual inheritance:
    Animal          Animal       ← TWO separate Animal objects!
      |               |
    Mammal          Bird
      \              /
         Bat                     ← Bat has TWO copies of Animal members
```

**The Solution — Virtual Inheritance:**
```cpp
class Animal {
protected:
    string name;
    int age;
public:
    Animal(const string& n = "", int a = 0) : name(n), age(a) {
        cout << "Animal constructed: " << n << endl;
    }
    virtual ~Animal() = default;
};

class Mammal : virtual public Animal {  // VIRTUAL inheritance
    bool hasFur;
public:
    Mammal(const string& n, int a, bool fur) : Animal(n, a), hasFur(fur) {
        cout << "Mammal constructed" << endl;
    }
};

class Bird : virtual public Animal {  // VIRTUAL inheritance
    double wingspan;
public:
    Bird(const string& n, int a, double ws) : Animal(n, a), wingspan(ws) {
        cout << "Bird constructed" << endl;
    }
};

class Bat : public Mammal, public Bird {
public:
    // With virtual inheritance, the MOST DERIVED class constructs the virtual base
    Bat(const string& n, int a, bool fur, double ws)
        : Animal(n, a),          // Bat directly constructs Animal (required!)
          Mammal(n, a, fur),     // Mammal's Animal() call is SKIPPED
          Bird(n, a, ws)         // Bird's Animal() call is SKIPPED
    {
        cout << "Bat constructed" << endl;
    }

    void show() {
        cout << name;  // No ambiguity! Only ONE copy of Animal::name
    }
};
```

**Construction order with virtual inheritance:**
```
Output for: Bat bat("Bruce", 5, true, 1.2);

Animal constructed: Bruce    ← virtual base constructed FIRST
Mammal constructed           ← then non-virtual bases in declaration order
Bird constructed
Bat constructed              ← finally the most-derived class
```

**Key rules:**
1. Virtual bases are always constructed FIRST, before any non-virtual bases
2. The most-derived class is responsible for calling the virtual base constructor
3. Intermediate classes' calls to the virtual base constructor are ignored
4. If the most-derived class doesn't explicitly construct the virtual base, the default constructor is used

**Cost of virtual inheritance:**
- Extra pointer(s) per object for virtual base offset
- Slightly slower access to virtual base members (extra indirection)
- More complex object layout

---

### Constructor/Destructor Call Order in Inheritance

Understanding the exact order is critical for resource management.

```cpp
class Member {
    string name;
public:
    Member(const string& n) : name(n) { cout << "  Member(" << name << ") ctor" << endl; }
    ~Member() { cout << "  Member(" << name << ") dtor" << endl; }
};

class Base {
    Member baseMember;
public:
    Base() : baseMember("base_m") { cout << "  Base ctor body" << endl; }
    virtual ~Base() { cout << "  Base dtor body" << endl; }
};

class Middle : public Base {
    Member middleMember;
public:
    Middle() : middleMember("mid_m") { cout << "  Middle ctor body" << endl; }
    ~Middle() { cout << "  Middle dtor body" << endl; }
};

class Derived : public Middle {
    Member derivedMember1;
    Member derivedMember2;
public:
    Derived() : derivedMember2("d_m2"), derivedMember1("d_m1") {
        // NOTE: despite initializer list order, d_m1 is constructed first
        // because it's declared first in the class
        cout << "  Derived ctor body" << endl;
    }
    ~Derived() { cout << "  Derived dtor body" << endl; }
};

// Creating: Derived d;
// CONSTRUCTION ORDER (top-down, members in declaration order):
//   Member(base_m) ctor        ← Base's member
//   Base ctor body             ← Base body
//   Member(mid_m) ctor         ← Middle's member
//   Middle ctor body           ← Middle body
//   Member(d_m1) ctor          ← Derived's first member (declaration order!)
//   Member(d_m2) ctor          ← Derived's second member
//   Derived ctor body          ← Derived body

// DESTRUCTION ORDER (exact reverse):
//   Derived dtor body
//   Member(d_m2) dtor
//   Member(d_m1) dtor
//   Middle dtor body
//   Member(mid_m) dtor
//   Base dtor body
//   Member(base_m) dtor
```

**Complete construction order rules:**
1. Virtual base classes (in depth-first, left-to-right order)
2. Non-virtual base classes (in declaration order)
3. Member variables (in declaration order, NOT initializer list order)
4. Constructor body

**Destruction is ALWAYS the exact reverse of construction.**

---

### Access Control in Inheritance (public, protected, private inheritance)

The inheritance access specifier determines how base class members are exposed through the derived class.

```cpp
class Base {
public:
    int publicVar = 1;
    void publicFunc() {}

protected:
    int protectedVar = 2;
    void protectedFunc() {}

private:
    int privateVar = 3;      // NEVER accessible in derived, regardless of inheritance type
    void privateFunc() {}
};
```

#### Public Inheritance (most common — "is-a")
```cpp
class PublicDerived : public Base {
public:
    void test() {
        publicVar = 10;      // OK: still public
        publicFunc();        // OK: still public
        protectedVar = 20;   // OK: still protected (accessible in derived)
        protectedFunc();     // OK: still protected
        // privateVar = 30;  // ERROR: private is never accessible
    }
};

PublicDerived obj;
obj.publicVar = 10;    // OK: public remains public
obj.publicFunc();      // OK
// obj.protectedVar;   // ERROR: protected not accessible from outside
```

#### Protected Inheritance ("implemented-in-terms-of")
```cpp
class ProtectedDerived : protected Base {
public:
    void test() {
        publicVar = 10;      // OK: now protected (was public)
        protectedVar = 20;   // OK: still protected
        // privateVar = 30;  // ERROR
    }
};

ProtectedDerived obj;
// obj.publicVar;       // ERROR: public became protected!
// obj.publicFunc();    // ERROR: not accessible from outside
```

#### Private Inheritance ("implemented-in-terms-of", strongest restriction)
```cpp
class PrivateDerived : private Base {
public:
    void test() {
        publicVar = 10;      // OK: now private (was public)
        protectedVar = 20;   // OK: now private (was protected)
        // privateVar = 30;  // ERROR
    }

    // Can selectively expose base members using 'using'
    using Base::publicFunc;  // makes publicFunc public again in PrivateDerived
};

PrivateDerived obj;
// obj.publicVar;       // ERROR: everything became private
obj.publicFunc();       // OK: explicitly re-exposed with 'using'
```

**Summary Table:**

| Base Member Access | `public` inheritance | `protected` inheritance | `private` inheritance |
|---|---|---|---|
| `public` | public | protected | private |
| `protected` | protected | protected | private |
| `private` | inaccessible | inaccessible | inaccessible |

**When to use each:**
- **public:** "is-a" relationship (Dog is-a Animal). 95% of cases.
- **protected:** Rarely used. Base interface available to further derived classes but not public.
- **private:** "implemented-in-terms-of". Use base class for implementation only. Prefer composition instead.

---

### Upcasting and Downcasting

#### Upcasting (Derived → Base)

Always safe, always implicit. This is the foundation of polymorphism.

```cpp
class Shape {
public:
    virtual void draw() const = 0;
    virtual string type() const = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
    double radius;
public:
    Circle(double r) : radius(r) {}
    void draw() const override { cout << "Drawing circle r=" << radius << endl; }
    string type() const override { return "Circle"; }
    double getRadius() const { return radius; }
};

class Square : public Shape {
    double side;
public:
    Square(double s) : side(s) {}
    void draw() const override { cout << "Drawing square s=" << side << endl; }
    string type() const override { return "Square"; }
    double getSide() const { return side; }
};

// Upcasting — implicit, safe
Circle c(5.0);
Shape* sPtr = &c;          // upcast: Circle* → Shape*
Shape& sRef = c;           // upcast: Circle& → Shape&
sPtr->draw();              // calls Circle::draw() (virtual dispatch)

// Polymorphic container
vector<unique_ptr<Shape>> shapes;
shapes.push_back(make_unique<Circle>(3.0));   // upcast
shapes.push_back(make_unique<Square>(4.0));   // upcast

for (const auto& s : shapes) {
    s->draw();  // correct version called for each
}
```

**Object slicing — the danger of upcasting by value:**
```cpp
Circle c(5.0);
Shape s = c;  // SLICING! Only the Shape part is copied. Circle data is LOST!
              // This compiles only if Shape is not abstract.
              // s.draw() would call Shape::draw(), NOT Circle::draw()

// ALWAYS use pointers or references for polymorphism
Shape& ref = c;   // OK: no slicing
Shape* ptr = &c;  // OK: no slicing
```

#### Downcasting (Base → Derived)

Potentially unsafe — the base pointer might not actually point to the expected derived type.

```cpp
void processShape(Shape* shape) {
    // Method 1: dynamic_cast (SAFE — runtime check)
    if (Circle* circle = dynamic_cast<Circle*>(shape)) {
        // shape actually points to a Circle
        cout << "Circle with radius: " << circle->getRadius() << endl;
    } else if (Square* square = dynamic_cast<Square*>(shape)) {
        // shape actually points to a Square
        cout << "Square with side: " << square->getSide() << endl;
    } else {
        cout << "Unknown shape type" << endl;
    }

    // Method 2: static_cast (UNSAFE — no runtime check)
    // Only use when you are 100% CERTAIN of the type
    Circle* c = static_cast<Circle*>(shape);  // undefined behavior if shape isn't a Circle!

    // Method 3: typeid (check type without casting)
    if (typeid(*shape) == typeid(Circle)) {
        Circle* c = static_cast<Circle*>(shape);  // safe because we checked
    }
}
```

**dynamic_cast with references:**
```cpp
void processRef(Shape& shape) {
    try {
        Circle& c = dynamic_cast<Circle&>(shape);  // throws if wrong type
        cout << "Radius: " << c.getRadius() << endl;
    } catch (const bad_cast& e) {
        cout << "Not a Circle: " << e.what() << endl;
    }
}
```

**Requirements for dynamic_cast:**
- Base class must have at least one virtual function (enables RTTI)
- Compile with RTTI enabled (default in most compilers)
- Only works with pointers and references
- Returns `nullptr` for pointer casts on failure
- Throws `std::bad_cast` for reference casts on failure

**Performance note:** `dynamic_cast` has runtime overhead (traverses type hierarchy). Avoid in performance-critical loops. Prefer virtual functions (polymorphism) over downcasting when possible.

---


## 2.4 Polymorphism

> Polymorphism ("many forms") is the ability of different objects to respond to the same message/interface in different ways. It is the most powerful feature of OOP, enabling flexible and extensible designs.

---

### Compile-Time Polymorphism (Static Binding / Early Binding)

The compiler determines which function to call at compile time. No runtime overhead.

---

#### Function Overloading

Multiple functions with the same name but different parameter signatures in the same scope.

```cpp
class Calculator {
public:
    // Overloaded by number of parameters
    int add(int a, int b) { return a + b; }
    int add(int a, int b, int c) { return a + b + c; }

    // Overloaded by parameter type
    double add(double a, double b) { return a + b; }
    string add(const string& a, const string& b) { return a + b; }

    // Overloaded by const qualification
    void display(int& x) { cout << "non-const ref: " << x << endl; }
    void display(const int& x) { cout << "const ref: " << x << endl; }
};

Calculator calc;
calc.add(1, 2);           // calls add(int, int)
calc.add(1, 2, 3);       // calls add(int, int, int)
calc.add(1.5, 2.5);      // calls add(double, double)
calc.add("hi", " there"); // calls add(string, string)

int x = 5;
const int y = 10;
calc.display(x);          // calls display(int&)
calc.display(y);          // calls display(const int&)
calc.display(42);         // calls display(const int&) — rvalue binds to const ref
```

**Overload resolution rules (simplified):**
1. Exact match (no conversion needed)
2. Promotion (char→int, float→double)
3. Standard conversion (int→double, derived*→base*)
4. User-defined conversion (conversion operators, converting constructors)

**Common pitfalls:**
```cpp
void func(int x) { cout << "int" << endl; }
void func(double x) { cout << "double" << endl; }

func(5);      // int — exact match
func(5.0);    // double — exact match
func(5.0f);   // double — float promotes to double
func('A');    // int — char promotes to int
// func(5L);  // AMBIGUOUS: long can convert to both int and double

// Cannot overload by return type alone:
// int getValue();
// double getValue();  // ERROR: same parameters
```

**Function overloading vs default arguments:**
```cpp
// Using overloading
void log(const string& msg) { log(msg, "INFO"); }
void log(const string& msg, const string& level) { /* ... */ }

// Using default arguments (often simpler)
void log(const string& msg, const string& level = "INFO") { /* ... */ }
```

---

#### Operator Overloading

Allows user-defined types to use operators (`+`, `-`, `==`, `<<`, etc.) with natural syntax.

**Rules for operator overloading:**
- Cannot create new operators
- Cannot change operator precedence or associativity
- Cannot change the number of operands
- At least one operand must be a user-defined type
- Some operators MUST be member functions: `=`, `()`, `[]`, `->`, `->*`
- Some operators SHOULD be non-member: `<<`, `>>`, symmetric binary operators

**Complete example — a Fraction class:**
```cpp
class Fraction {
    int numerator;
    int denominator;

    void simplify() {
        int g = __gcd(abs(numerator), abs(denominator));
        numerator /= g;
        denominator /= g;
        if (denominator < 0) { numerator = -numerator; denominator = -denominator; }
    }

public:
    Fraction(int num = 0, int den = 1) : numerator(num), denominator(den) {
        if (den == 0) throw invalid_argument("Denominator cannot be zero");
        simplify();
    }

    // --- Arithmetic operators (binary, return new object) ---
    Fraction operator+(const Fraction& other) const {
        return Fraction(
            numerator * other.denominator + other.numerator * denominator,
            denominator * other.denominator
        );
    }

    Fraction operator-(const Fraction& other) const {
        return Fraction(
            numerator * other.denominator - other.numerator * denominator,
            denominator * other.denominator
        );
    }

    Fraction operator*(const Fraction& other) const {
        return Fraction(numerator * other.numerator, denominator * other.denominator);
    }

    Fraction operator/(const Fraction& other) const {
        if (other.numerator == 0) throw invalid_argument("Division by zero");
        return Fraction(numerator * other.denominator, denominator * other.numerator);
    }

    // --- Compound assignment operators ---
    Fraction& operator+=(const Fraction& other) {
        *this = *this + other;
        return *this;
    }

    // --- Unary operators ---
    Fraction operator-() const { return Fraction(-numerator, denominator); }
    Fraction operator+() const { return *this; }

    // --- Pre/Post increment ---
    Fraction& operator++() {  // pre-increment: ++f
        numerator += denominator;
        simplify();
        return *this;
    }

    Fraction operator++(int) {  // post-increment: f++
        Fraction temp = *this;
        ++(*this);
        return temp;
    }

    // --- Comparison operators ---
    bool operator==(const Fraction& other) const {
        return numerator == other.numerator && denominator == other.denominator;
    }
    bool operator!=(const Fraction& other) const { return !(*this == other); }
    bool operator<(const Fraction& other) const {
        return numerator * other.denominator < other.numerator * denominator;
    }
    bool operator>(const Fraction& other) const { return other < *this; }
    bool operator<=(const Fraction& other) const { return !(other < *this); }
    bool operator>=(const Fraction& other) const { return !(*this < other); }

    // --- Spaceship operator (C++20) — generates all comparisons ---
    // auto operator<=>(const Fraction& other) const { ... }

    // --- Conversion operator ---
    explicit operator double() const {
        return static_cast<double>(numerator) / denominator;
    }

    // --- Function call operator (makes object callable) ---
    double operator()() const { return static_cast<double>(numerator) / denominator; }

    // --- Subscript operator (example for matrix-like access) ---
    // int& operator[](int index) { ... }

    // --- Stream operators (must be friend or non-member) ---
    friend ostream& operator<<(ostream& os, const Fraction& f) {
        if (f.denominator == 1) os << f.numerator;
        else os << f.numerator << "/" << f.denominator;
        return os;
    }

    friend istream& operator>>(istream& is, Fraction& f) {
        char slash;
        is >> f.numerator >> slash >> f.denominator;
        f.simplify();
        return is;
    }
};

// Usage
Fraction a(1, 2), b(1, 3);
Fraction c = a + b;        // 5/6
Fraction d = a * b;        // 1/6
cout << c << endl;          // prints: 5/6
cout << (a < b) << endl;   // prints: 0 (1/2 is not less than 1/3)
double val = static_cast<double>(a);  // 0.5 (explicit conversion)
```

**Operators that CANNOT be overloaded:**
- `::` (scope resolution)
- `.` (member access)
- `.*` (pointer-to-member access)
- `?:` (ternary)
- `sizeof`
- `typeid`
- `alignof`

**Member vs Non-member operator overloading:**
```cpp
// Member: left operand is always the object
class Complex {
public:
    Complex operator+(const Complex& rhs) const;  // this + rhs
    // Problem: 5 + complex won't work! (5 is not a Complex object)
};

// Non-member (friend): allows conversion on both sides
class Complex {
    double real, imag;
public:
    Complex(double r, double i = 0) : real(r), imag(i) {}  // converting ctor

    friend Complex operator+(const Complex& lhs, const Complex& rhs) {
        return Complex(lhs.real + rhs.real, lhs.imag + rhs.imag);
    }
};

Complex c(1, 2);
Complex r1 = c + 5;    // OK: 5 implicitly converts to Complex(5, 0)
Complex r2 = 5 + c;    // OK: same (wouldn't work with member operator+)
```

---

#### Templates as Compile-Time Polymorphism

Templates generate type-specific code at compile time — "parametric polymorphism."

```cpp
// Same algorithm works for any type that supports the required operations
template <typename Container, typename Predicate>
int countIf(const Container& c, Predicate pred) {
    int count = 0;
    for (const auto& elem : c) {
        if (pred(elem)) count++;
    }
    return count;
}

vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
int evens = countIf(nums, [](int x) { return x % 2 == 0; });  // 5
int big = countIf(nums, [](int x) { return x > 5; });          // 5

vector<string> words = {"hello", "hi", "hey", "world"};
int hWords = countIf(words, [](const string& s) { return s[0] == 'h'; });  // 3
```

---

### Runtime Polymorphism (Dynamic Binding / Late Binding)

The actual function called is determined at runtime based on the real type of the object. Requires virtual functions and pointers/references.

---

#### Virtual Functions — Deep Dive

```cpp
class Notification {
protected:
    string message;
    string recipient;

public:
    Notification(const string& msg, const string& to) : message(msg), recipient(to) {}

    // Virtual function — can be overridden by derived classes
    virtual void send() const {
        cout << "Generic notification to " << recipient << ": " << message << endl;
    }

    // Non-virtual function — CANNOT be overridden polymorphically
    void log() const {
        cout << "[LOG] Notification created for " << recipient << endl;
    }

    virtual ~Notification() = default;
};

class EmailNotification : public Notification {
    string subject;
public:
    EmailNotification(const string& msg, const string& to, const string& subj)
        : Notification(msg, to), subject(subj) {}

    void send() const override {
        cout << "EMAIL to " << recipient << endl;
        cout << "  Subject: " << subject << endl;
        cout << "  Body: " << message << endl;
    }
};

class SMSNotification : public Notification {
public:
    SMSNotification(const string& msg, const string& to) : Notification(msg, to) {}

    void send() const override {
        cout << "SMS to " << recipient << ": " << message << endl;
    }
};

class PushNotification : public Notification {
    string appName;
public:
    PushNotification(const string& msg, const string& to, const string& app)
        : Notification(msg, to), appName(app) {}

    void send() const override {
        cout << "PUSH [" << appName << "] to " << recipient << ": " << message << endl;
    }
};

// Polymorphic usage — the power of virtual functions
void broadcastAlert(const vector<unique_ptr<Notification>>& channels) {
    for (const auto& channel : channels) {
        channel->send();  // correct version called based on actual type
    }
}

// Usage
vector<unique_ptr<Notification>> alerts;
alerts.push_back(make_unique<EmailNotification>("Server down!", "admin@co.com", "ALERT"));
alerts.push_back(make_unique<SMSNotification>("Server down!", "+1234567890"));
alerts.push_back(make_unique<PushNotification>("Server down!", "admin", "OpsApp"));
broadcastAlert(alerts);
```

**Virtual vs Non-virtual behavior:**
```cpp
Notification* ptr = new EmailNotification("hi", "user@x.com", "Hello");
ptr->send();  // calls EmailNotification::send() — VIRTUAL (dynamic dispatch)
ptr->log();   // calls Notification::log() — NON-VIRTUAL (static dispatch)
delete ptr;
```

**When a virtual function is NOT dispatched dynamically:**
1. Called on an object directly (not through pointer/reference)
2. Called with explicit scope: `ptr->Base::send()`
3. Called from within a constructor or destructor (object is not fully formed)

```cpp
class Base {
public:
    virtual void whoAmI() { cout << "Base" << endl; }
    Base() { whoAmI(); }  // calls Base::whoAmI(), NOT Derived::whoAmI()!
};

class Derived : public Base {
public:
    void whoAmI() override { cout << "Derived" << endl; }
    Derived() { whoAmI(); }  // calls Derived::whoAmI()
};

Derived d;
// Output:
// Base       ← from Base constructor (Derived part doesn't exist yet!)
// Derived    ← from Derived constructor
```

**Rule:** Never call virtual functions from constructors or destructors — the derived part doesn't exist yet (ctor) or has already been destroyed (dtor).

---

#### Pure Virtual Functions and Abstract Classes

A **pure virtual function** has no implementation in the base class. It forces every concrete derived class to provide its own implementation.

```cpp
class PaymentProcessor {
public:
    // Pure virtual functions — define the interface
    virtual bool processPayment(double amount) = 0;
    virtual bool refund(double amount) = 0;
    virtual string getProviderName() const = 0;
    virtual double getTransactionFee(double amount) const = 0;

    // Can still have concrete methods
    void printReceipt(double amount) const {
        cout << "Payment of $" << amount << " processed via " << getProviderName() << endl;
        cout << "Fee: $" << getTransactionFee(amount) << endl;
    }

    // Pure virtual with a default implementation (unusual but valid)
    virtual void logTransaction(double amount) = 0;

    virtual ~PaymentProcessor() = default;
};

// Pure virtual WITH default implementation — derived must still override,
// but can call the base version
void PaymentProcessor::logTransaction(double amount) {
    cout << "[DEFAULT LOG] " << getProviderName() << ": $" << amount << endl;
}

class StripeProcessor : public PaymentProcessor {
public:
    bool processPayment(double amount) override {
        cout << "Stripe: charging $" << amount << endl;
        return true;
    }

    bool refund(double amount) override {
        cout << "Stripe: refunding $" << amount << endl;
        return true;
    }

    string getProviderName() const override { return "Stripe"; }

    double getTransactionFee(double amount) const override {
        return amount * 0.029 + 0.30;  // 2.9% + $0.30
    }

    void logTransaction(double amount) override {
        PaymentProcessor::logTransaction(amount);  // call base default
        cout << "[STRIPE LOG] Additional stripe-specific logging" << endl;
    }
};

// PaymentProcessor pp;  // ERROR: cannot instantiate abstract class
StripeProcessor sp;       // OK: all pure virtuals are implemented
PaymentProcessor* pp = &sp;  // OK: pointer to abstract type
pp->processPayment(100);
pp->printReceipt(100);
```

**Abstract class rules:**
- Cannot be instantiated (but can have pointers/references to it)
- Can have constructors (called by derived classes)
- Can have data members
- Can have concrete (non-pure) methods
- A derived class is also abstract if it doesn't implement ALL pure virtual functions

---

#### VTable and VPtr — Internal Mechanism of Virtual Dispatch

Every class with at least one virtual function gets a **VTable** (Virtual Method Table) — a static array of function pointers. Every object of such a class gets a hidden **vptr** (virtual pointer) that points to its class's VTable.

**Detailed memory layout:**
```
class Base {                          class Derived : public Base {
    virtual void f1();                    void f1() override;
    virtual void f2();                    void f3();  // new virtual
    int x;                                int y;
};                                    };

Base object memory:                   Derived object memory:
┌────────────────┐                    ┌────────────────┐
│ vptr ──────────┼──→ Base VTable     │ vptr ──────────┼──→ Derived VTable
├────────────────┤                    ├────────────────┤
│ x              │                    │ x (from Base)  │
└────────────────┘                    ├────────────────┤
                                      │ y              │
                                      └────────────────┘

Base VTable:                          Derived VTable:
┌─────────────────────┐               ┌─────────────────────┐
│ [0] → Base::f1()    │               │ [0] → Derived::f1() │  ← overridden!
│ [1] → Base::f2()    │               │ [1] → Base::f2()    │  ← inherited
│ [2] → Base::~Base() │               │ [2] → Derived::~()  │  ← destructor
└─────────────────────┘               │ [3] → Derived::f3() │  ← new
                                      └─────────────────────┘
```

**How a virtual call works (step by step):**
```cpp
Base* ptr = new Derived();
ptr->f1();  // virtual call

// Compiler generates (pseudo-code):
// 1. Load vptr from object:     vptr = ptr->__vptr
// 2. Index into vtable:         func_ptr = vptr[0]  (f1 is at index 0)
// 3. Call through pointer:      func_ptr(ptr)       (passes 'this')
// Result: Derived::f1() is called
```

**Performance cost of virtual dispatch:**
- Memory: +8 bytes per object (vptr on 64-bit), +one VTable per class
- Speed: One extra pointer dereference per virtual call
- Cache: VTable lookup may cause cache miss
- Inlining: Virtual calls generally cannot be inlined (compiler doesn't know the target)

**When virtual dispatch does NOT happen:**
```cpp
Derived d;
d.f1();           // NO virtual dispatch — compiler knows exact type
                  // Direct call to Derived::f1()

Base* ptr = &d;
ptr->f1();        // Virtual dispatch — compiler doesn't know runtime type
ptr->Base::f1();  // NO virtual dispatch — explicit base call
```

---

#### `override` and `final` Keywords

**`override` (C++11):** Explicitly states that a function is meant to override a base class virtual function. The compiler verifies this.

```cpp
class Base {
public:
    virtual void process(int x) const;
    virtual void handle(double d);
    virtual void cleanup();
};

class Derived : public Base {
public:
    // WITHOUT override — silent bugs:
    void process(int x);          // OOPS: missing 'const' — this is a NEW function, not an override!
    void handle(float d);         // OOPS: float != double — new function, not override!

    // WITH override — compiler catches errors:
    // void process(int x) override;          // ERROR: doesn't match base signature (missing const)
    // void handle(float d) override;         // ERROR: parameter type mismatch
    void process(int x) const override;       // OK: correctly overrides
    void handle(double d) override;           // OK: correctly overrides
    void cleanup() override;                  // OK: correctly overrides
};
```

**`final` (C++11):** Prevents further overriding of a virtual function, or prevents inheritance from a class.

```cpp
// final on a virtual function
class Base {
public:
    virtual void critical() {}
};

class Middle : public Base {
public:
    void critical() override final {}  // no further overriding allowed
};

class Bottom : public Middle {
public:
    // void critical() override {}  // ERROR: critical() is final in Middle
};

// final on a class — prevents any inheritance
class Singleton final {
    static Singleton* instance;
    Singleton() {}
public:
    static Singleton* getInstance() {
        if (!instance) instance = new Singleton();
        return instance;
    }
};
// class MySingleton : public Singleton {};  // ERROR: Singleton is final
```

**Performance benefit of `final`:** The compiler can devirtualize calls to `final` functions (call them directly without vtable lookup) because it knows they cannot be overridden.

---

#### Covariant Return Types

When overriding a virtual function, the return type can be a more derived type (pointer or reference only).

```cpp
class Document {
public:
    virtual Document* clone() const {
        return new Document(*this);
    }
    virtual ~Document() = default;
};

class PDFDocument : public Document {
    string pdfData;
public:
    PDFDocument(const string& data) : pdfData(data) {}

    // Covariant return: PDFDocument* instead of Document*
    PDFDocument* clone() const override {
        return new PDFDocument(*this);
    }
};

class WordDocument : public Document {
    string wordData;
public:
    WordDocument(const string& data) : wordData(data) {}

    WordDocument* clone() const override {
        return new WordDocument(*this);
    }
};

// Usage benefit:
PDFDocument pdf("content");
PDFDocument* pdfCopy = pdf.clone();  // returns PDFDocument* directly, no cast needed!

Document* doc = &pdf;
Document* docCopy = doc->clone();    // returns Document* (but actually a PDFDocument)
```

**Rules for covariant returns:**
- Only works with pointers and references (not values)
- The derived return type must be publicly derived from the base return type
- Access level must be the same or less restrictive

---

### RTTI (Runtime Type Information)

RTTI allows you to determine the actual type of an object at runtime. Requires at least one virtual function in the hierarchy.

#### `dynamic_cast` — Safe Downcasting

```cpp
class Media {
public:
    virtual void play() = 0;
    virtual ~Media() = default;
};

class Audio : public Media {
    int bitrate;
public:
    Audio(int br) : bitrate(br) {}
    void play() override { cout << "Playing audio at " << bitrate << "kbps" << endl; }
    int getBitrate() const { return bitrate; }
};

class Video : public Media {
    int width, height;
public:
    Video(int w, int h) : width(w), height(h) {}
    void play() override { cout << "Playing video " << width << "x" << height << endl; }
    pair<int,int> getResolution() const { return {width, height}; }
};

class Subtitle : public Media {
    string language;
public:
    Subtitle(const string& lang) : language(lang) {}
    void play() override { cout << "Showing subtitles in " << language << endl; }
};

// Using dynamic_cast for safe downcasting
void analyzeMedia(Media* m) {
    // Pointer cast — returns nullptr on failure
    if (Audio* audio = dynamic_cast<Audio*>(m)) {
        cout << "Audio bitrate: " << audio->getBitrate() << endl;
    } else if (Video* video = dynamic_cast<Video*>(m)) {
        auto [w, h] = video->getResolution();
        cout << "Video resolution: " << w << "x" << h << endl;
    } else {
        cout << "Other media type" << endl;
    }
}

// Reference cast — throws std::bad_cast on failure
void mustBeAudio(Media& m) {
    try {
        Audio& audio = dynamic_cast<Audio&>(m);
        cout << "Bitrate: " << audio.getBitrate() << endl;
    } catch (const bad_cast& e) {
        cerr << "Not an Audio object: " << e.what() << endl;
    }
}

// Cross-casting (sideways cast in multiple inheritance)
class Playable { public: virtual ~Playable() = default; };
class Recordable { public: virtual ~Recordable() = default; };
class DVR : public Playable, public Recordable {};

Playable* p = new DVR();
Recordable* r = dynamic_cast<Recordable*>(p);  // cross-cast: works!
```

#### `typeid` Operator

Returns a `std::type_info` object describing the type.

```cpp
#include <typeinfo>

Media* m = new Video(1920, 1080);

// With polymorphic types (has virtual functions): gives DYNAMIC type
cout << typeid(*m).name() << endl;  // "Video" (or mangled name)

// With non-polymorphic types or non-dereferenced pointers: gives STATIC type
cout << typeid(m).name() << endl;   // "Media*" (pointer type, not object type)

// Comparing types
if (typeid(*m) == typeid(Video)) {
    cout << "It's a Video!" << endl;
}

// typeid with nullptr dereference throws std::bad_typeid
Media* null = nullptr;
// typeid(*null);  // throws std::bad_typeid
```

**`dynamic_cast` vs `typeid`:**
| Feature | `dynamic_cast` | `typeid` |
|---------|---|---|
| Purpose | Cast to derived type | Query type info |
| Checks inheritance | Yes (works with base classes) | No (exact type only) |
| Cross-casting | Yes | No |
| Returns | Pointer/reference or null/exception | `type_info` object |
| Use case | When you need to USE the derived interface | When you need to IDENTIFY the type |

**When to avoid RTTI:**
- If you find yourself using `dynamic_cast` frequently, your design may need refactoring
- Prefer virtual functions (let polymorphism do the work)
- RTTI has runtime cost and can be disabled with compiler flags (`-fno-rtti`)

---


## 2.5 Abstraction

> Abstraction is the process of exposing only the essential features of an entity while hiding the unnecessary complexity. In C++, abstraction is achieved through abstract classes and interfaces (pure virtual classes).

---

### Abstract Classes vs Interfaces (Pure Virtual Classes)

C++ does not have a dedicated `interface` keyword like Java or C#. Instead, interfaces are implemented as classes containing ONLY pure virtual functions.

**Abstract Class — partial abstraction:**
```cpp
class DatabaseConnection {
protected:
    string connectionString;
    bool connected = false;
    int queryCount = 0;

public:
    DatabaseConnection(const string& connStr) : connectionString(connStr) {}

    // Pure virtual — derived classes MUST implement
    virtual bool connect() = 0;
    virtual void disconnect() = 0;
    virtual ResultSet executeQuery(const string& sql) = 0;

    // Concrete methods — shared implementation
    bool isConnected() const { return connected; }
    int getQueryCount() const { return queryCount; }

    void logQuery(const string& sql) {
        queryCount++;
        cout << "[Query #" << queryCount << "] " << sql << endl;
    }

    // Template Method pattern — algorithm skeleton with customizable steps
    ResultSet safeQuery(const string& sql) {
        if (!connected) connect();
        logQuery(sql);
        auto result = executeQuery(sql);  // delegated to derived
        return result;
    }

    virtual ~DatabaseConnection() {
        if (connected) disconnect();
    }
};

class PostgresConnection : public DatabaseConnection {
    // pg_conn* pgConn;  // PostgreSQL-specific handle
public:
    PostgresConnection(const string& connStr) : DatabaseConnection(connStr) {}

    bool connect() override {
        // pgConn = PQconnectdb(connectionString.c_str());
        connected = true;
        cout << "Connected to PostgreSQL: " << connectionString << endl;
        return true;
    }

    void disconnect() override {
        // PQfinish(pgConn);
        connected = false;
        cout << "Disconnected from PostgreSQL" << endl;
    }

    ResultSet executeQuery(const string& sql) override {
        // return PQexec(pgConn, sql.c_str());
        cout << "PostgreSQL executing: " << sql << endl;
        return {};
    }
};
```

**Interface — complete abstraction (pure virtual class):**
```cpp
// Interface: ONLY pure virtual functions, no data, no implementation
class IRepository {
public:
    virtual void save(const Entity& entity) = 0;
    virtual Entity findById(int id) = 0;
    virtual vector<Entity> findAll() = 0;
    virtual void deleteById(int id) = 0;
    virtual bool exists(int id) = 0;
    virtual ~IRepository() = default;
};

class ICache {
public:
    virtual void put(const string& key, const string& value) = 0;
    virtual optional<string> get(const string& key) = 0;
    virtual void invalidate(const string& key) = 0;
    virtual void clear() = 0;
    virtual ~ICache() = default;
};

class IEventListener {
public:
    virtual void onEvent(const Event& event) = 0;
    virtual ~IEventListener() = default;
};
```

**Key differences:**

| Aspect | Abstract Class | Interface (Pure Virtual Class) |
|--------|---|---|
| Pure virtual functions | At least one | ALL functions are pure virtual |
| Concrete methods | Yes | No |
| Data members | Yes | No (typically) |
| Constructor | Yes (initializes data) | No (nothing to initialize) |
| Multiple inheritance | Can cause diamond problem | Safe (no data duplication) |
| Purpose | Partial implementation + contract | Pure contract/API definition |
| When to use | Shared behavior among related classes | Defining capabilities/roles |

---

### Designing Interfaces in C++

**Principles of good interface design:**

1. **Single Responsibility:** Each interface should represent one cohesive capability
2. **Small and focused:** Prefer many small interfaces over one large one
3. **Stable:** Interfaces should change rarely (they are contracts)
4. **Complete:** Should provide all operations needed for the capability

```cpp
// GOOD: Small, focused interfaces representing distinct capabilities

class IReadable {
public:
    virtual string read() = 0;
    virtual bool hasMore() const = 0;
    virtual void reset() = 0;
    virtual ~IReadable() = default;
};

class IWritable {
public:
    virtual void write(const string& data) = 0;
    virtual void flush() = 0;
    virtual ~IWritable() = default;
};

class ISeekable {
public:
    virtual void seek(long position) = 0;
    virtual long tell() const = 0;
    virtual long size() const = 0;
    virtual ~ISeekable() = default;
};

class ICloseable {
public:
    virtual void close() = 0;
    virtual bool isClosed() const = 0;
    virtual ~ICloseable() = default;
};

// Compose interfaces as needed
class FileStream : public IReadable, public IWritable, public ISeekable, public ICloseable {
    fstream file;
    bool closed = false;

public:
    FileStream(const string& path) { file.open(path, ios::in | ios::out); }

    // IReadable
    string read() override { string s; getline(file, s); return s; }
    bool hasMore() const override { return !file.eof(); }
    void reset() override { file.seekg(0); }

    // IWritable
    void write(const string& data) override { file << data; }
    void flush() override { file.flush(); }

    // ISeekable
    void seek(long pos) override { file.seekg(pos); file.seekp(pos); }
    long tell() const override { return const_cast<fstream&>(file).tellg(); }
    long size() const override { /* ... */ return 0; }

    // ICloseable
    void close() override { file.close(); closed = true; }
    bool isClosed() const override { return closed; }
};

// A network stream might only be readable and writable (not seekable)
class NetworkStream : public IReadable, public IWritable, public ICloseable {
    // ... only implements what makes sense for network I/O
};
```

**Interface naming conventions:**
- Prefix with `I`: `ISerializable`, `IComparable`, `IObserver`
- Suffix with `able`: `Printable`, `Drawable`, `Sortable`
- Use descriptive verbs: `EventHandler`, `DataProvider`

---

### Interface Segregation in Practice

**The Interface Segregation Principle (ISP):** No client should be forced to depend on methods it does not use.

**Problem — Fat Interface:**
```cpp
// BAD: One huge interface forces all implementations to handle everything
class IWorker {
public:
    virtual void work() = 0;
    virtual void eat() = 0;
    virtual void sleep() = 0;
    virtual void attendMeeting() = 0;
    virtual void writeReport() = 0;
    virtual ~IWorker() = default;
};

// A Robot worker doesn't eat or sleep!
class Robot : public IWorker {
public:
    void work() override { /* OK */ }
    void eat() override { /* meaningless! */ }       // forced to implement
    void sleep() override { /* meaningless! */ }     // forced to implement
    void attendMeeting() override { /* maybe */ }
    void writeReport() override { /* maybe */ }
};
```

**Solution — Segregated Interfaces:**
```cpp
// GOOD: Split into cohesive, focused interfaces
class IWorkable {
public:
    virtual void work() = 0;
    virtual ~IWorkable() = default;
};

class IFeedable {
public:
    virtual void eat() = 0;
    virtual void drink() = 0;
    virtual ~IFeedable() = default;
};

class ISleepable {
public:
    virtual void sleep() = 0;
    virtual void wake() = 0;
    virtual ~ISleepable() = default;
};

class IReportable {
public:
    virtual void writeReport() = 0;
    virtual void attendMeeting() = 0;
    virtual ~IReportable() = default;
};

// Human worker implements all relevant interfaces
class HumanWorker : public IWorkable, public IFeedable, public ISleepable, public IReportable {
public:
    void work() override { cout << "Human working" << endl; }
    void eat() override { cout << "Human eating" << endl; }
    void drink() override { cout << "Human drinking" << endl; }
    void sleep() override { cout << "Human sleeping" << endl; }
    void wake() override { cout << "Human waking" << endl; }
    void writeReport() override { cout << "Writing report" << endl; }
    void attendMeeting() override { cout << "In meeting" << endl; }
};

// Robot only implements what makes sense
class RobotWorker : public IWorkable {
public:
    void work() override { cout << "Robot working 24/7" << endl; }
    // No eat, sleep, or meeting methods — not forced to implement them!
};

// Functions depend only on the interfaces they need
void assignTask(IWorkable& worker) { worker.work(); }
void scheduleLunch(IFeedable& worker) { worker.eat(); }
```

**Benefits of interface segregation:**
- Classes only implement what they actually need
- Changes to one interface don't affect unrelated implementations
- Easier to test (mock only the relevant interface)
- Clearer dependencies (function signatures show exactly what's needed)

---

### Abstract Factory using Abstract Classes

The Abstract Factory pattern uses abstract classes to create families of related objects without specifying their concrete types.

```cpp
// --- Abstract Products ---
class Button {
public:
    virtual void render() const = 0;
    virtual void onClick(function<void()> handler) = 0;
    virtual string getType() const = 0;
    virtual ~Button() = default;
};

class TextBox {
public:
    virtual void render() const = 0;
    virtual void setText(const string& text) = 0;
    virtual string getText() const = 0;
    virtual ~TextBox() = default;
};

class CheckBox {
public:
    virtual void render() const = 0;
    virtual void toggle() = 0;
    virtual bool isChecked() const = 0;
    virtual ~CheckBox() = default;
};

// --- Abstract Factory ---
class UIFactory {
public:
    virtual unique_ptr<Button> createButton(const string& label) = 0;
    virtual unique_ptr<TextBox> createTextBox(const string& placeholder) = 0;
    virtual unique_ptr<CheckBox> createCheckBox(const string& label) = 0;
    virtual ~UIFactory() = default;
};

// --- Concrete Products (Windows) ---
class WindowsButton : public Button {
    string label;
    function<void()> handler;
public:
    WindowsButton(const string& l) : label(l) {}
    void render() const override { cout << "[Win Button: " << label << "]" << endl; }
    void onClick(function<void()> h) override { handler = h; }
    string getType() const override { return "WindowsButton"; }
};

class WindowsTextBox : public TextBox {
    string text, placeholder;
public:
    WindowsTextBox(const string& ph) : placeholder(ph) {}
    void render() const override {
        cout << "[Win TextBox: " << (text.empty() ? placeholder : text) << "]" << endl;
    }
    void setText(const string& t) override { text = t; }
    string getText() const override { return text; }
};

class WindowsCheckBox : public CheckBox {
    string label;
    bool checked = false;
public:
    WindowsCheckBox(const string& l) : label(l) {}
    void render() const override {
        cout << "[Win CheckBox: " << (checked ? "[X]" : "[ ]") << " " << label << "]" << endl;
    }
    void toggle() override { checked = !checked; }
    bool isChecked() const override { return checked; }
};

// --- Concrete Factory (Windows) ---
class WindowsUIFactory : public UIFactory {
public:
    unique_ptr<Button> createButton(const string& label) override {
        return make_unique<WindowsButton>(label);
    }
    unique_ptr<TextBox> createTextBox(const string& placeholder) override {
        return make_unique<WindowsTextBox>(placeholder);
    }
    unique_ptr<CheckBox> createCheckBox(const string& label) override {
        return make_unique<WindowsCheckBox>(label);
    }
};

// --- Concrete Products (macOS) ---
class MacButton : public Button {
    string label;
    function<void()> handler;
public:
    MacButton(const string& l) : label(l) {}
    void render() const override { cout << "(Mac Button: " << label << ")" << endl; }
    void onClick(function<void()> h) override { handler = h; }
    string getType() const override { return "MacButton"; }
};

// ... MacTextBox, MacCheckBox similarly ...

class MacUIFactory : public UIFactory {
public:
    unique_ptr<Button> createButton(const string& label) override {
        return make_unique<MacButton>(label);
    }
    unique_ptr<TextBox> createTextBox(const string& placeholder) override {
        // return make_unique<MacTextBox>(placeholder);
        return nullptr;
    }
    unique_ptr<CheckBox> createCheckBox(const string& label) override {
        // return make_unique<MacCheckBox>(label);
        return nullptr;
    }
};

// --- Client code — works with ANY factory ---
class LoginForm {
    unique_ptr<TextBox> usernameField;
    unique_ptr<TextBox> passwordField;
    unique_ptr<Button> loginButton;
    unique_ptr<CheckBox> rememberMe;

public:
    LoginForm(UIFactory& factory) {
        usernameField = factory.createTextBox("Enter username");
        passwordField = factory.createTextBox("Enter password");
        loginButton = factory.createButton("Login");
        rememberMe = factory.createCheckBox("Remember me");
    }

    void render() const {
        usernameField->render();
        passwordField->render();
        rememberMe->render();
        loginButton->render();
    }
};

// Usage — switch entire UI theme by changing factory
unique_ptr<UIFactory> factory;
#ifdef _WIN32
    factory = make_unique<WindowsUIFactory>();
#else
    factory = make_unique<MacUIFactory>();
#endif

LoginForm form(*factory);
form.render();
```

---


## 2.6 Composition vs Inheritance

> One of the most important design decisions in OOP is choosing between inheritance ("is-a") and composition ("has-a"). The Gang of Four (GoF) famously advises: "Favor object composition over class inheritance."

---

### "Has-A" vs "Is-A" Relationships

**"Is-A" (Inheritance):** The derived class IS a specialized version of the base class. It should satisfy the Liskov Substitution Principle — anywhere a base class is expected, a derived class should work correctly.

**"Has-A" (Composition):** The containing class HAS a component as part of its implementation. The component provides functionality that the container uses.

```cpp
// IS-A: A Dog IS an Animal
class Animal {
public:
    virtual void eat() { cout << "Eating" << endl; }
    virtual void sleep() { cout << "Sleeping" << endl; }
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    void eat() override { cout << "Dog eating kibble" << endl; }
    void bark() { cout << "Woof!" << endl; }
};

// HAS-A: A Car HAS an Engine, Transmission, and Wheels
class Engine {
    int horsepower;
    bool running = false;
public:
    Engine(int hp) : horsepower(hp) {}
    void start() { running = true; cout << "Engine started (" << horsepower << "hp)" << endl; }
    void stop() { running = false; cout << "Engine stopped" << endl; }
    bool isRunning() const { return running; }
    int getHP() const { return horsepower; }
};

class Transmission {
    int currentGear = 0;
    int maxGears;
public:
    Transmission(int gears) : maxGears(gears) {}
    void shiftUp() { if (currentGear < maxGears) currentGear++; }
    void shiftDown() { if (currentGear > 0) currentGear--; }
    int getGear() const { return currentGear; }
};

class GPS {
    double lat, lon;
public:
    void updatePosition(double la, double lo) { lat = la; lon = lo; }
    pair<double, double> getPosition() const { return {lat, lon}; }
    void navigate(const string& destination) {
        cout << "Navigating to " << destination << endl;
    }
};

class Car {
    Engine engine;              // composition: Car OWNS engine
    Transmission transmission; // composition: Car OWNS transmission
    GPS* gps;                  // aggregation: GPS can exist independently

public:
    Car(int hp, int gears, GPS* gpsDevice = nullptr)
        : engine(hp), transmission(gears), gps(gpsDevice) {}

    void start() {
        engine.start();
        cout << "Car ready to drive" << endl;
    }

    void drive() {
        if (!engine.isRunning()) {
            cout << "Start the engine first!" << endl;
            return;
        }
        transmission.shiftUp();
        cout << "Driving in gear " << transmission.getGear() << endl;
    }

    void navigateTo(const string& dest) {
        if (gps) gps->navigate(dest);
        else cout << "No GPS installed" << endl;
    }
};
```

**How to decide:**
| Question | If Yes → | If No → |
|----------|----------|---------|
| Can you say "B is a type of A"? | Inheritance | Composition |
| Does Liskov Substitution hold? | Inheritance | Composition |
| Do you need polymorphism? | Inheritance (or composition + interfaces) | Composition |
| Might the relationship change at runtime? | Composition | Either |
| Is the base class designed for inheritance? | Inheritance | Composition |

---

### When to Prefer Composition over Inheritance

**Problems with inheritance:**

1. **Tight coupling:** Derived class depends on base class implementation details
2. **Fragile base class problem:** Changes to base class can break derived classes
3. **Inflexible:** Relationship is fixed at compile time
4. **Explosion of classes:** Combining features requires many subclasses

```cpp
// PROBLEM: Inheritance explosion
// Want: Coffee with different sizes, milks, and toppings
// With inheritance, you'd need:
//   SmallCoffeeWithWholeMillAndWhippedCream
//   LargeCoffeeWithOatMilkAndCaramel
//   MediumCoffeeWithAlmondMilk
//   ... hundreds of combinations!

// SOLUTION: Composition
class Milk {
public:
    virtual string name() const = 0;
    virtual double cost() const = 0;
    virtual ~Milk() = default;
};

class WholeMilk : public Milk {
public:
    string name() const override { return "Whole Milk"; }
    double cost() const override { return 0.50; }
};

class OatMilk : public Milk {
public:
    string name() const override { return "Oat Milk"; }
    double cost() const override { return 0.75; }
};

class Topping {
public:
    virtual string name() const = 0;
    virtual double cost() const = 0;
    virtual ~Topping() = default;
};

class WhippedCream : public Topping {
public:
    string name() const override { return "Whipped Cream"; }
    double cost() const override { return 0.60; }
};

class Caramel : public Topping {
public:
    string name() const override { return "Caramel"; }
    double cost() const override { return 0.50; }
};

enum class Size { Small, Medium, Large };

class Coffee {
    Size size;
    unique_ptr<Milk> milk;                // composition
    vector<unique_ptr<Topping>> toppings; // composition

public:
    Coffee(Size s) : size(s) {}

    void setMilk(unique_ptr<Milk> m) { milk = std::move(m); }
    void addTopping(unique_ptr<Topping> t) { toppings.push_back(std::move(t)); }

    double totalCost() const {
        double base = (size == Size::Small) ? 3.0 : (size == Size::Medium) ? 4.0 : 5.0;
        if (milk) base += milk->cost();
        for (const auto& t : toppings) base += t->cost();
        return base;
    }

    void describe() const {
        cout << "Coffee (" << (int)size << ")";
        if (milk) cout << " + " << milk->name();
        for (const auto& t : toppings) cout << " + " << t->name();
        cout << " = $" << totalCost() << endl;
    }
};

// Any combination without class explosion!
Coffee c(Size::Large);
c.setMilk(make_unique<OatMilk>());
c.addTopping(make_unique<WhippedCream>());
c.addTopping(make_unique<Caramel>());
c.describe();  // Coffee (2) + Oat Milk + Whipped Cream + Caramel = $6.85
```

**Composition enables runtime flexibility:**
```cpp
// Strategy pattern — behavior can change at runtime
class SortStrategy {
public:
    virtual void sort(vector<int>& data) = 0;
    virtual ~SortStrategy() = default;
};

class QuickSort : public SortStrategy {
public:
    void sort(vector<int>& data) override {
        std::sort(data.begin(), data.end());
        cout << "Sorted with QuickSort" << endl;
    }
};

class MergeSort : public SortStrategy {
public:
    void sort(vector<int>& data) override {
        stable_sort(data.begin(), data.end());
        cout << "Sorted with MergeSort" << endl;
    }
};

class BubbleSort : public SortStrategy {
public:
    void sort(vector<int>& data) override {
        for (size_t i = 0; i < data.size(); i++)
            for (size_t j = 0; j < data.size() - i - 1; j++)
                if (data[j] > data[j+1]) swap(data[j], data[j+1]);
        cout << "Sorted with BubbleSort" << endl;
    }
};

class DataProcessor {
    unique_ptr<SortStrategy> strategy;  // composition — can change at runtime!

public:
    void setStrategy(unique_ptr<SortStrategy> s) {
        strategy = std::move(s);
    }

    void process(vector<int>& data) {
        if (!strategy) throw runtime_error("No sort strategy set");
        strategy->sort(data);
    }
};

// Change behavior at runtime — impossible with inheritance alone
DataProcessor processor;
vector<int> data = {5, 2, 8, 1, 9};

processor.setStrategy(make_unique<QuickSort>());
processor.process(data);  // uses QuickSort

processor.setStrategy(make_unique<BubbleSort>());
processor.process(data);  // now uses BubbleSort — same object, different behavior!
```

---

### Aggregation vs Composition

Both are "has-a" relationships, but they differ in **ownership** and **lifecycle management**.

#### Composition (Strong "Has-A" — Owns)

The container **owns** the component. The component's lifetime is tied to the container. When the container is destroyed, the component is destroyed too.

```cpp
class Heart {
public:
    void beat() { cout << "Beating..." << endl; }
};

class Brain {
public:
    void think() { cout << "Thinking..." << endl; }
};

class Human {
    Heart heart;    // composition: Human OWNS heart (value member)
    Brain brain;    // composition: Human OWNS brain

    // Or with dynamic allocation:
    unique_ptr<Liver> liver = make_unique<Liver>();  // still composition (unique ownership)

public:
    void live() {
        heart.beat();
        brain.think();
    }
};
// When Human is destroyed, Heart and Brain are automatically destroyed
// Heart and Brain cannot exist without a Human (in this model)
```

**Implementation patterns for composition:**
- Value members (most common): `Heart heart;`
- `unique_ptr` members: `unique_ptr<Heart> heart;`
- Objects created in constructor, destroyed in destructor

#### Aggregation (Weak "Has-A" — Uses)

The container **uses** the component but does NOT own it. The component can exist independently and may be shared.

```cpp
class Professor {
    string name;
    string department;
public:
    Professor(const string& n, const string& d) : name(n), department(d) {}
    string getName() const { return name; }
};

class Student {
    string name;
    int id;
public:
    Student(const string& n, int id) : name(n), id(id) {}
    string getName() const { return name; }
};

class Course {
    string title;
    Professor* instructor;          // aggregation: Course doesn't own Professor
    vector<Student*> students;      // aggregation: Course doesn't own Students

public:
    Course(const string& t, Professor* prof) : title(t), instructor(prof) {}

    void enroll(Student* s) { students.push_back(s); }

    void display() const {
        cout << title << " taught by " << instructor->getName() << endl;
        cout << "Students: ";
        for (auto* s : students) cout << s->getName() << ", ";
        cout << endl;
    }
};

// Aggregation in action:
Professor prof("Dr. Smith", "CS");
Student s1("Alice", 1), s2("Bob", 2), s3("Charlie", 3);

Course cs101("Intro to CS", &prof);
Course cs201("Data Structures", &prof);  // same professor, two courses

cs101.enroll(&s1);
cs101.enroll(&s2);
cs201.enroll(&s2);  // same student in two courses
cs201.enroll(&s3);

// When cs101 is destroyed, prof, s1, s2 still exist!
// Professor and Students have independent lifetimes
```

**Comparison Table:**

| Aspect | Composition | Aggregation |
|--------|---|---|
| Ownership | Strong (container owns part) | Weak (container uses part) |
| Lifecycle | Part dies with container | Part outlives container |
| Multiplicity | Part belongs to ONE container | Part can be shared |
| UML | Filled diamond ◆───── | Empty diamond ◇───── |
| C++ implementation | Value member or `unique_ptr` | Raw pointer, reference, or `shared_ptr` |
| Example | Body-Heart, House-Room | Team-Player, Course-Student |
| Destruction | Container destroys parts | Container does NOT destroy parts |

**Association (weakest relationship):**
```cpp
// Association: temporary "uses" relationship (not even aggregation)
class Logger {
public:
    void log(const string& msg) { cout << msg << endl; }
};

class OrderService {
    // No stored reference to Logger — just uses it temporarily
public:
    void processOrder(Logger& logger) {  // association via parameter
        logger.log("Processing order...");
        // ... process ...
        logger.log("Order complete");
    }
};
```

---

### Dependency Injection Basics

**Dependency Injection (DI)** is a technique where an object receives its dependencies from the outside rather than creating them internally. This is the practical application of the Dependency Inversion Principle.

**Without DI (tight coupling):**
```cpp
class EmailService {
public:
    void send(const string& to, const string& msg) {
        cout << "Sending email to " << to << ": " << msg << endl;
    }
};

class OrderProcessor {
    EmailService emailService;  // HARDCODED dependency — cannot change or test!

public:
    void processOrder(const string& orderId) {
        // ... process order ...
        emailService.send("customer@email.com", "Order " + orderId + " confirmed");
    }
};
// Problems:
// - Cannot use a different notification service (SMS, Push)
// - Cannot test without actually sending emails
// - OrderProcessor is tightly coupled to EmailService
```

**With DI (loose coupling):**
```cpp
// Step 1: Define abstraction (interface)
class INotificationService {
public:
    virtual void notify(const string& recipient, const string& message) = 0;
    virtual ~INotificationService() = default;
};

// Step 2: Concrete implementations
class EmailNotificationService : public INotificationService {
public:
    void notify(const string& recipient, const string& message) override {
        cout << "EMAIL to " << recipient << ": " << message << endl;
    }
};

class SMSNotificationService : public INotificationService {
public:
    void notify(const string& recipient, const string& message) override {
        cout << "SMS to " << recipient << ": " << message << endl;
    }
};

class MockNotificationService : public INotificationService {
    vector<pair<string, string>> sentMessages;
public:
    void notify(const string& recipient, const string& message) override {
        sentMessages.push_back({recipient, message});  // just record, don't send
    }
    const vector<pair<string, string>>& getSent() const { return sentMessages; }
};

// Step 3: Depend on abstraction, inject from outside
class OrderProcessor {
    INotificationService& notifier;  // depends on INTERFACE, not concrete class

public:
    // Constructor Injection — dependency provided at construction
    OrderProcessor(INotificationService& ns) : notifier(ns) {}

    void processOrder(const string& orderId, const string& customer) {
        // ... process order logic ...
        notifier.notify(customer, "Order " + orderId + " confirmed");
    }
};

// Usage — inject the desired implementation
EmailNotificationService emailService;
OrderProcessor prodProcessor(emailService);  // production: real emails
prodProcessor.processOrder("ORD-001", "alice@email.com");

MockNotificationService mockService;
OrderProcessor testProcessor(mockService);   // testing: no real emails sent
testProcessor.processOrder("ORD-002", "test@test.com");
// Verify: mockService.getSent() contains the expected message
```

#### Types of Dependency Injection

**1. Constructor Injection (preferred):**
```cpp
class Service {
    IDatabase& db;
    ILogger& logger;
    ICache& cache;
public:
    // All dependencies provided at construction — object is always fully initialized
    Service(IDatabase& db, ILogger& logger, ICache& cache)
        : db(db), logger(logger), cache(cache) {}
};
```

**2. Setter Injection:**
```cpp
class Service {
    IDatabase* db = nullptr;
    ILogger* logger = nullptr;
public:
    // Dependencies can be set/changed after construction
    void setDatabase(IDatabase* database) { db = database; }
    void setLogger(ILogger* log) { logger = log; }

    void doWork() {
        if (!db) throw runtime_error("Database not set!");
        // ...
    }
};
```

**3. Interface Injection:**
```cpp
class IDatabaseAware {
public:
    virtual void injectDatabase(IDatabase& db) = 0;
    virtual ~IDatabaseAware() = default;
};

class Service : public IDatabaseAware {
    IDatabase* db = nullptr;
public:
    void injectDatabase(IDatabase& database) override {
        db = &database;
    }
};
```

**Comparison of injection types:**

| Type | Pros | Cons | When to use |
|------|------|------|-------------|
| Constructor | Object always valid; immutable deps; clear requirements | Many params for many deps | Default choice; required dependencies |
| Setter | Flexible; optional deps; can change at runtime | Object may be in invalid state | Optional dependencies; reconfigurable |
| Interface | Standardized injection contract | Extra interface to implement | Framework-managed injection |

**Benefits of Dependency Injection:**
1. **Testability:** Inject mocks/stubs for unit testing
2. **Flexibility:** Swap implementations without changing client code
3. **Decoupling:** Client depends on abstraction, not concrete class
4. **Single Responsibility:** Object doesn't create its own dependencies
5. **Open/Closed:** Add new implementations without modifying existing code

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Classes | Blueprint for objects; use access specifiers to control visibility |
| Constructors | Default, Parameterized, Copy (deep!), Move (steal resources); use initializer lists |
| Rule of Five | If you define any special member, define all five (or use Rule of Zero) |
| Destructor | Virtual in base classes; called in reverse order of construction |
| Static Members | Belong to class; no `this`; defined outside class (or `inline static` in C++17) |
| `const` methods | Promise not to modify object; required for const objects/references |
| `mutable` | Exempts member from const; use for caching, mutexes |
| `friend` | Grants private access; not inherited, not mutual, not transitive |
| Encapsulation | Hide data, expose behavior; enforce invariants; don't over-use getters/setters |
| Immutability | All const members, no setters, return new objects; thread-safe by default |
| Inheritance types | Single, Multiple, Multilevel, Hierarchical, Hybrid |
| Diamond Problem | Virtual inheritance shares one base instance; most-derived constructs virtual base |
| Construction order | Virtual bases → Non-virtual bases → Members (declaration order) → Body |
| Access in inheritance | public (is-a), protected (rare), private (prefer composition) |
| Upcasting | Always safe, implicit; foundation of polymorphism |
| Downcasting | Use `dynamic_cast` for safety; avoid when possible |
| Object Slicing | Pass by pointer/reference for polymorphism, never by value |
| Function Overloading | Same name, different params; resolved at compile time |
| Operator Overloading | Natural syntax for user types; follow conventions |
| Virtual Functions | Runtime dispatch via VTable/VPtr; one indirection per call |
| Pure Virtual | `= 0`; makes class abstract; forces derived to implement |
| VTable/VPtr | Hidden pointer per object → class's function pointer table |
| `override`/`final` | Catch errors at compile time; enable devirtualization |
| Covariant Returns | Override can return more-derived pointer/reference |
| RTTI | `dynamic_cast` (safe downcast), `typeid` (type query); needs virtual functions |
| Abstract Class | Has pure virtuals + can have data/concrete methods |
| Interface | All pure virtual, no data; safe for multiple inheritance |
| ISP | Small focused interfaces > one fat interface |
| Composition | "Has-a"; flexible, runtime-changeable; prefer over inheritance |
| Aggregation | Weak "has-a"; part outlives container; shared ownership |
| Dependency Injection | Inject dependencies from outside; depend on abstractions |

---

## Interview Tips for Module 2

1. **Virtual destructor** — If a class has ANY virtual function, make the destructor virtual. Explain WHY (prevent resource leaks when deleting through base pointer).

2. **Rule of Five/Zero** — Know when to implement all five special members vs. relying on compiler-generated ones. Prefer Rule of Zero with smart pointers.

3. **Object slicing** — Explain what happens when you pass a derived object by value to a function expecting a base object. Always use pointers/references for polymorphism.

4. **`override` keyword** — Always use it. Explain how it catches bugs (signature mismatches that would silently create new functions instead of overriding).

5. **Composition over inheritance** — Be ready to explain with examples. Key reasons: flexibility, loose coupling, runtime behavior changes, avoiding class explosion.

6. **Diamond problem** — Draw the diagram, explain the ambiguity, explain virtual inheritance solution, explain who constructs the virtual base.

7. **VTable mechanism** — Draw the memory layout. Explain vptr, vtable, and the indirection. Know the performance cost (one pointer per object, one indirection per call, prevents inlining).

8. **Abstract class vs Interface** — C++ has no `interface` keyword. Interface = class with only pure virtual functions and virtual destructor. Know when to use each.

9. **`dynamic_cast` vs `static_cast`** — `dynamic_cast` is safe (runtime check, returns null on failure), `static_cast` is unsafe (no check, undefined behavior on wrong type). `dynamic_cast` requires RTTI and virtual functions.

10. **Encapsulation** — Not just getters/setters! It's about maintaining invariants, hiding implementation details, and providing a meaningful interface. Explain the difference between an anemic class (all getters/setters) and a well-encapsulated class (behavior-rich methods).

11. **Copy vs Move** — Explain when each is called, the performance difference (O(n) vs O(1)), and why move leaves the source in a "valid but unspecified" state.

12. **Dependency Injection** — Explain constructor injection, why it enables testing (inject mocks), and how it implements the Dependency Inversion Principle.

13. **`const` correctness** — Explain const member functions, const objects, const references, and why marking functions const is important for API design.

14. **Construction/Destruction order** — Base before derived (construction), derived before base (destruction). Members in declaration order. Virtual bases first.

15. **When NOT to use inheritance** — If you can't say "B is-a A" and satisfy Liskov Substitution, use composition instead. Private inheritance is almost always better replaced by composition.