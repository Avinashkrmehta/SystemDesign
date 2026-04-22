# Module 4: Design Patterns — Creational

> Creational patterns deal with object creation mechanisms — they abstract the instantiation process, making systems independent of how their objects are created, composed, and represented. Instead of instantiating objects directly with `new`, creational patterns provide more flexibility in deciding which objects to create, how to create them, and when to create them.

---

## 4.1 Singleton Pattern

> The Singleton pattern ensures that a class has only ONE instance throughout the application's lifetime and provides a global point of access to that instance.

---

### Why Singleton?

Some resources should exist exactly once in a system:
- Database connection pool
- Logger
- Configuration manager
- Thread pool
- File system manager
- Hardware interface (printer spooler, GPU context)

**The problem without Singleton:**
```cpp
// BAD: Multiple loggers writing to the same file — race conditions, duplicate handles
Logger log1;  // opens file
Logger log2;  // opens same file again — conflict!
Logger log3;  // and again...

// BAD: Multiple config managers — which one has the latest state?
ConfigManager c1;
c1.set("timeout", 30);
ConfigManager c2;
cout << c2.get("timeout");  // doesn't see c1's changes!
```

---

### Basic Singleton (Naive — NOT Thread-Safe)

```cpp
class Singleton {
private:
    static Singleton* instance;  // the single instance

    // Private constructor — prevents external instantiation
    Singleton() {
        cout << "Singleton created" << endl;
    }

    // Delete copy and move operations
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
    Singleton(Singleton&&) = delete;
    Singleton& operator=(Singleton&&) = delete;

public:
    static Singleton* getInstance() {
        if (instance == nullptr) {
            instance = new Singleton();
        }
        return instance;
    }

    void doWork() {
        cout << "Singleton doing work" << endl;
    }
};

// Static member definition
Singleton* Singleton::instance = nullptr;

// Usage
Singleton* s1 = Singleton::getInstance();
Singleton* s2 = Singleton::getInstance();
// s1 == s2 (same pointer, same object)
```

**Problems with this approach:**
1. **Not thread-safe:** Two threads could both see `instance == nullptr` and create two instances
2. **Memory leak:** `instance` is never deleted (no destructor call)
3. **No destruction order control:** Static pointer outlives the program

---

### Eager Initialization

The instance is created at program startup (before `main()` runs), avoiding thread-safety issues.

```cpp
class EagerSingleton {
private:
    static EagerSingleton instance;  // created at static initialization time

    EagerSingleton() {
        cout << "EagerSingleton created" << endl;
    }

    EagerSingleton(const EagerSingleton&) = delete;
    EagerSingleton& operator=(const EagerSingleton&) = delete;

public:
    static EagerSingleton& getInstance() {
        return instance;
    }

    void configure(const string& setting) {
        cout << "Configured: " << setting << endl;
    }
};

// Instance created here — before main() executes
EagerSingleton EagerSingleton::instance;
```

**Pros:**
- Thread-safe (created before any threads exist)
- Simple implementation

**Cons:**
- Instance created even if never used (wastes resources)
- No control over initialization order between translation units (Static Initialization Order Fiasco)
- Cannot pass parameters to constructor at creation time

**Static Initialization Order Fiasco:**
```cpp
// File: config.cpp
EagerSingleton EagerSingleton::instance;  // created first? or second?

// File: database.cpp
// This might try to use EagerSingleton before it's constructed!
DatabasePool DatabasePool::pool(EagerSingleton::getInstance().getConfig());
// UNDEFINED BEHAVIOR if EagerSingleton isn't constructed yet!
```

---

### Lazy Initialization — Meyer's Singleton (C++11, Thread-Safe)

The **recommended** approach in modern C++. Uses a local static variable, which C++11 guarantees is initialized exactly once, even in the presence of multiple threads.

```cpp
class Singleton {
private:
    Singleton() {
        cout << "Singleton created" << endl;
    }

    ~Singleton() {
        cout << "Singleton destroyed" << endl;
    }

    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;

public:
    static Singleton& getInstance() {
        static Singleton instance;  // created on first call, destroyed at program exit
        return instance;
    }

    void doWork() {
        cout << "Working..." << endl;
    }
};

// Usage
Singleton::getInstance().doWork();

// Cannot do:
// Singleton s;                    // ERROR: constructor is private
// Singleton s2 = Singleton::getInstance();  // ERROR: copy constructor is deleted
```

**Why this works:**
- C++11 §6.7: "If control enters the declaration concurrently while the variable is being initialized, the concurrent execution shall wait for completion of the initialization."
- The compiler inserts synchronization (typically a flag + mutex) to ensure one-time initialization
- Destruction happens automatically at program exit (reverse order of construction)

**This is the gold standard for Singleton in C++11 and later.**

---

### Thread-Safe Singleton with Double-Checked Locking (Pre-C++11 or Explicit Control)

Before C++11, Meyer's Singleton wasn't guaranteed thread-safe. The Double-Checked Locking Pattern (DCLP) was used:

```cpp
#include <mutex>
#include <atomic>

class ThreadSafeSingleton {
private:
    static atomic<ThreadSafeSingleton*> instance;
    static mutex mtx;

    ThreadSafeSingleton() {
        cout << "ThreadSafeSingleton created" << endl;
    }

    ThreadSafeSingleton(const ThreadSafeSingleton&) = delete;
    ThreadSafeSingleton& operator=(const ThreadSafeSingleton&) = delete;

public:
    static ThreadSafeSingleton* getInstance() {
        ThreadSafeSingleton* tmp = instance.load(memory_order_acquire);
        if (tmp == nullptr) {                    // First check (no lock)
            lock_guard<mutex> lock(mtx);
            tmp = instance.load(memory_order_relaxed);
            if (tmp == nullptr) {                // Second check (with lock)
                tmp = new ThreadSafeSingleton();
                instance.store(tmp, memory_order_release);
            }
        }
        return tmp;
    }

    void doWork() { cout << "Working..." << endl; }
};

atomic<ThreadSafeSingleton*> ThreadSafeSingleton::instance{nullptr};
mutex ThreadSafeSingleton::mtx;
```

**Why double-checked?**
1. **First check (without lock):** Fast path — if instance exists, return immediately (no lock overhead)
2. **Lock acquisition:** Only if instance might not exist
3. **Second check (with lock):** Another thread might have created it while we were waiting for the lock

**Why `atomic` and memory ordering?**
- Without `atomic`, the compiler/CPU might reorder operations
- A thread could see a non-null pointer to a partially constructed object
- `memory_order_acquire/release` ensures proper visibility of the constructed object

---

### Singleton with `std::call_once`

A cleaner alternative to DCLP using C++ standard library:

```cpp
#include <mutex>

class CallOnceSingleton {
private:
    static unique_ptr<CallOnceSingleton> instance;
    static once_flag initFlag;

    CallOnceSingleton(const string& config) {
        cout << "Singleton created with config: " << config << endl;
    }

    CallOnceSingleton(const CallOnceSingleton&) = delete;
    CallOnceSingleton& operator=(const CallOnceSingleton&) = delete;

public:
    static CallOnceSingleton& getInstance(const string& config = "") {
        call_once(initFlag, [&config]() {
            instance.reset(new CallOnceSingleton(config));
        });
        return *instance;
    }

    void doWork() { cout << "Working..." << endl; }
};

unique_ptr<CallOnceSingleton> CallOnceSingleton::instance;
once_flag CallOnceSingleton::initFlag;

// Usage
CallOnceSingleton::getInstance("production").doWork();
CallOnceSingleton::getInstance("ignored").doWork();  // config param ignored on subsequent calls
```

**`std::call_once` guarantees:**
- The callable is executed exactly once, even if called from multiple threads
- All threads that call `call_once` with the same `once_flag` will block until the one executing thread completes
- Exception safety: if the callable throws, another thread gets to try

---

### When to Use and When to Avoid Singleton

**Legitimate use cases:**
| Use Case | Why Singleton Makes Sense |
|----------|--------------------------|
| Logger | One log destination, consistent formatting, thread-safe writes |
| Configuration | One source of truth for app settings |
| Connection Pool | Manage limited resources centrally |
| Hardware Access | One interface to physical device |
| Cache Manager | Shared cache across components |
| Thread Pool | Centralized task scheduling |

**When to AVOID Singleton:**

1. **When you need testability:**
```cpp
// BAD: Hard to test — can't inject a mock
class OrderService {
    void processOrder() {
        Database::getInstance().save(order);  // tightly coupled!
    }
};

// GOOD: Dependency injection — testable
class OrderService {
    IDatabase& db;
public:
    OrderService(IDatabase& database) : db(database) {}
    void processOrder() {
        db.save(order);  // can inject mock for testing
    }
};
```

2. **When state makes testing unpredictable:**
```cpp
// Singleton state persists between tests — tests affect each other!
TEST(Test1) {
    Singleton::getInstance().setState(42);
}
TEST(Test2) {
    // Singleton still has state 42 from Test1! Tests are not independent!
}
```

3. **When you might need multiple instances later:**
   - "We'll only ever have one database" → Later: "We need read replicas"
   - "One cache is enough" → Later: "We need L1 and L2 caches"

4. **When it hides dependencies:**
```cpp
// BAD: Hidden dependency — not visible in constructor/interface
class ReportGenerator {
public:
    void generate() {
        auto data = Database::getInstance().query("...");  // hidden!
        auto config = Config::getInstance().get("format"); // hidden!
        Logger::getInstance().log("Generating report");    // hidden!
    }
};
// What does ReportGenerator depend on? You can't tell from its interface!
```

---

### Singleton vs Global Variable

| Aspect | Singleton | Global Variable |
|--------|-----------|-----------------|
| Initialization | Controlled (lazy or eager) | At program start (uncontrolled order) |
| Construction | Can have parameters, logic | Simple initialization only |
| Access control | Through `getInstance()` method | Direct access from anywhere |
| Inheritance | Can use polymorphism | No |
| Thread safety | Can be made thread-safe | No built-in safety |
| Lifetime | Controlled destruction | Destroyed at program exit (order issues) |
| Testability | Both are problematic | Both are problematic |
| Encapsulation | Private constructor, controlled interface | No encapsulation |

**Both share the fundamental problem:** They introduce global state, making code harder to test, reason about, and parallelize.

**Preferred alternative — Dependency Injection:**
```cpp
// Instead of Singleton, create one instance and pass it around
int main() {
    auto logger = make_shared<FileLogger>("app.log");
    auto config = make_shared<AppConfig>("config.yaml");
    auto db = make_shared<PostgresDB>(config->getDbUrl());

    // Pass dependencies explicitly
    OrderService orderService(db, logger);
    UserService userService(db, logger);
    // Clear dependencies, testable, no global state
}
```

---

### Complete Production-Ready Singleton Example

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <mutex>
#include <fstream>
#include <chrono>
#include <sstream>
#include <iomanip>

class Logger {
public:
    enum class Level { DEBUG, INFO, WARN, ERROR, FATAL };

private:
    Level minLevel = Level::INFO;
    ofstream logFile;
    mutex logMutex;
    bool consoleOutput = true;

    Logger() {
        logFile.open("application.log", ios::app);
    }

    ~Logger() {
        if (logFile.is_open()) {
            log(Level::INFO, "Logger shutting down");
            logFile.close();
        }
    }

    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;

    string levelToString(Level level) const {
        switch (level) {
            case Level::DEBUG: return "DEBUG";
            case Level::INFO:  return "INFO ";
            case Level::WARN:  return "WARN ";
            case Level::ERROR: return "ERROR";
            case Level::FATAL: return "FATAL";
        }
        return "?????";
    }

    string getTimestamp() const {
        auto now = chrono::system_clock::now();
        auto time = chrono::system_clock::to_time_t(now);
        stringstream ss;
        ss << put_time(localtime(&time), "%Y-%m-%d %H:%M:%S");
        return ss.str();
    }

public:
    static Logger& getInstance() {
        static Logger instance;
        return instance;
    }

    void setLevel(Level level) {
        lock_guard<mutex> lock(logMutex);
        minLevel = level;
    }

    void setConsoleOutput(bool enabled) {
        lock_guard<mutex> lock(logMutex);
        consoleOutput = enabled;
    }

    void log(Level level, const string& message) {
        if (level < minLevel) return;

        lock_guard<mutex> lock(logMutex);

        string formatted = "[" + getTimestamp() + "] [" + levelToString(level) + "] " + message;

        if (logFile.is_open()) {
            logFile << formatted << endl;
        }

        if (consoleOutput) {
            cout << formatted << endl;
        }
    }

    void debug(const string& msg) { log(Level::DEBUG, msg); }
    void info(const string& msg)  { log(Level::INFO, msg); }
    void warn(const string& msg)  { log(Level::WARN, msg); }
    void error(const string& msg) { log(Level::ERROR, msg); }
    void fatal(const string& msg) { log(Level::FATAL, msg); }
};

// Convenience macro
#define LOG_DEBUG(msg) Logger::getInstance().debug(msg)
#define LOG_INFO(msg)  Logger::getInstance().info(msg)
#define LOG_WARN(msg)  Logger::getInstance().warn(msg)
#define LOG_ERROR(msg) Logger::getInstance().error(msg)

// Usage
int main() {
    Logger::getInstance().setLevel(Logger::Level::DEBUG);
    LOG_INFO("Application started");
    LOG_DEBUG("Loading configuration...");
    LOG_WARN("Config file not found, using defaults");
    LOG_ERROR("Failed to connect to database");
    return 0;
}
```

---


## 4.2 Factory Method Pattern

> The Factory Method pattern defines an interface for creating an object, but lets subclasses decide which class to instantiate. It defers instantiation to subclasses, promoting loose coupling between the creator and the concrete products.

---

### The Problem Factory Method Solves

```cpp
// BAD: Client code directly creates concrete objects — tightly coupled
class NotificationService {
public:
    void sendAlert(const string& message, const string& type) {
        if (type == "email") {
            EmailNotification* n = new EmailNotification(message);  // hardcoded!
            n->send();
            delete n;
        } else if (type == "sms") {
            SMSNotification* n = new SMSNotification(message);      // hardcoded!
            n->send();
            delete n;
        } else if (type == "push") {
            PushNotification* n = new PushNotification(message);    // hardcoded!
            n->send();
            delete n;
        }
        // Adding a new type requires modifying this code — violates OCP!
    }
};
```

**Problems:**
1. Violates Open/Closed Principle — must modify existing code to add new types
2. Client is coupled to ALL concrete classes
3. Object creation logic is scattered and duplicated
4. Hard to test — can't substitute mock objects

---

### Simple Factory (Not a GoF Pattern, but Common)

A single class with a static method that creates objects based on input. Simple but limited.

```cpp
// --- Product hierarchy ---
class Transport {
public:
    virtual void deliver() const = 0;
    virtual double calculateCost(double distance) const = 0;
    virtual string getType() const = 0;
    virtual ~Transport() = default;
};

class Truck : public Transport {
public:
    void deliver() const override {
        cout << "Delivering by road in a truck" << endl;
    }
    double calculateCost(double distance) const override {
        return distance * 1.5;  // $1.5 per km
    }
    string getType() const override { return "Truck"; }
};

class Ship : public Transport {
public:
    void deliver() const override {
        cout << "Delivering by sea in a ship" << endl;
    }
    double calculateCost(double distance) const override {
        return distance * 0.8;  // $0.8 per km (cheaper but slower)
    }
    string getType() const override { return "Ship"; }
};

class Airplane : public Transport {
public:
    void deliver() const override {
        cout << "Delivering by air in an airplane" << endl;
    }
    double calculateCost(double distance) const override {
        return distance * 5.0;  // $5.0 per km (expensive but fast)
    }
    string getType() const override { return "Airplane"; }
};

// --- Simple Factory ---
class TransportFactory {
public:
    static unique_ptr<Transport> create(const string& type) {
        if (type == "truck") return make_unique<Truck>();
        if (type == "ship") return make_unique<Ship>();
        if (type == "airplane") return make_unique<Airplane>();
        throw invalid_argument("Unknown transport type: " + type);
    }
};

// Usage
auto transport = TransportFactory::create("truck");
transport->deliver();
cout << "Cost for 100km: $" << transport->calculateCost(100) << endl;
```

**Limitations of Simple Factory:**
- Still violates OCP (must modify the factory to add new types)
- All creation logic in one place — can become a God class
- No polymorphism in the creation process itself

---

### Factory Method Pattern (GoF)

The key insight: **make the factory method virtual** so subclasses can override it to create different products.

```cpp
// --- Product interface ---
class Document {
public:
    virtual void open() = 0;
    virtual void save() = 0;
    virtual void close() = 0;
    virtual string getExtension() const = 0;
    virtual ~Document() = default;
};

class PDFDocument : public Document {
    string content;
public:
    void open() override { cout << "Opening PDF document" << endl; }
    void save() override { cout << "Saving PDF (compressing...)" << endl; }
    void close() override { cout << "Closing PDF" << endl; }
    string getExtension() const override { return ".pdf"; }
};

class WordDocument : public Document {
    string content;
public:
    void open() override { cout << "Opening Word document" << endl; }
    void save() override { cout << "Saving Word document (formatting...)" << endl; }
    void close() override { cout << "Closing Word document" << endl; }
    string getExtension() const override { return ".docx"; }
};

class SpreadsheetDocument : public Document {
    vector<vector<string>> cells;
public:
    void open() override { cout << "Opening Spreadsheet" << endl; }
    void save() override { cout << "Saving Spreadsheet (recalculating...)" << endl; }
    void close() override { cout << "Closing Spreadsheet" << endl; }
    string getExtension() const override { return ".xlsx"; }
};

// --- Creator (abstract) with Factory Method ---
class Application {
public:
    // THE FACTORY METHOD — subclasses override this to create specific documents
    virtual unique_ptr<Document> createDocument() = 0;

    // Template Method that USES the factory method
    void newDocument() {
        auto doc = createDocument();  // calls the overridden factory method
        doc->open();
        documents.push_back(std::move(doc));
        cout << "Document created. Total: " << documents.size() << endl;
    }

    void saveAll() {
        for (auto& doc : documents) {
            doc->save();
        }
    }

    void closeAll() {
        for (auto& doc : documents) {
            doc->close();
        }
        documents.clear();
    }

    virtual ~Application() = default;

private:
    vector<unique_ptr<Document>> documents;
};

// --- Concrete Creators ---
class PDFApplication : public Application {
public:
    unique_ptr<Document> createDocument() override {
        cout << "PDFApplication creating PDF..." << endl;
        return make_unique<PDFDocument>();
    }
};

class WordApplication : public Application {
public:
    unique_ptr<Document> createDocument() override {
        cout << "WordApplication creating Word doc..." << endl;
        return make_unique<WordDocument>();
    }
};

class SpreadsheetApplication : public Application {
public:
    unique_ptr<Document> createDocument() override {
        cout << "SpreadsheetApplication creating spreadsheet..." << endl;
        return make_unique<SpreadsheetDocument>();
    }
};

// --- Client code ---
void workWithApp(Application& app) {
    app.newDocument();
    app.newDocument();
    app.saveAll();
    app.closeAll();
}

int main() {
    PDFApplication pdfApp;
    WordApplication wordApp;

    workWithApp(pdfApp);   // creates PDFs
    workWithApp(wordApp);  // creates Word docs
    // Client code doesn't know or care about concrete document types!
}
```

**Structure:**
```
┌─────────────────────┐         ┌──────────────────┐
│   Application       │         │    Document      │
│   (Creator)         │         │    (Product)     │
├─────────────────────┤         ├──────────────────┤
│ + newDocument()     │────────→│ + open()         │
│ + saveAll()         │ creates │ + save()         │
│ + createDocument()  │ ←──────→│ + close()        │
│   {abstract}        │         └──────────────────┘
└─────────────────────┘                  ▲
          ▲                              │
          │                    ┌─────────┼─────────┐
┌─────────┴──────────┐        │         │         │
│  PDFApplication    │   PDFDocument  WordDoc  SpreadsheetDoc
│  createDocument()  │
│  → returns PDF     │
└────────────────────┘
```

---

### Simple Factory vs Factory Method

| Aspect | Simple Factory | Factory Method |
|--------|---------------|----------------|
| Structure | One class with static/instance method | Abstract creator + concrete creators |
| Extensibility | Must modify factory to add types (violates OCP) | Add new creator subclass (OCP compliant) |
| Polymorphism | No polymorphism in creation | Creator is polymorphic |
| Complexity | Simple | More classes, more structure |
| Use case | Few types, unlikely to change | Many types, expected to grow |
| GoF pattern? | No (it's an idiom) | Yes |

---

### Parameterized Factory Method

The factory method takes parameters to decide which product to create, while still allowing subclass customization.

```cpp
class Notification {
public:
    virtual void send(const string& message) = 0;
    virtual ~Notification() = default;
};

class EmailNotification : public Notification {
    string recipient;
public:
    EmailNotification(const string& to) : recipient(to) {}
    void send(const string& message) override {
        cout << "EMAIL to " << recipient << ": " << message << endl;
    }
};

class SMSNotification : public Notification {
    string phoneNumber;
public:
    SMSNotification(const string& phone) : phoneNumber(phone) {}
    void send(const string& message) override {
        cout << "SMS to " << phoneNumber << ": " << message << endl;
    }
};

class PushNotification : public Notification {
    string deviceToken;
public:
    PushNotification(const string& token) : deviceToken(token) {}
    void send(const string& message) override {
        cout << "PUSH to device " << deviceToken << ": " << message << endl;
    }
};

class SlackNotification : public Notification {
    string channel;
public:
    SlackNotification(const string& ch) : channel(ch) {}
    void send(const string& message) override {
        cout << "SLACK #" << channel << ": " << message << endl;
    }
};

// --- Parameterized Factory Method in Creator ---
class AlertService {
public:
    enum class Channel { EMAIL, SMS, PUSH, SLACK };

    // Parameterized factory method — can be overridden
    virtual unique_ptr<Notification> createNotification(Channel channel,
                                                         const string& target) {
        switch (channel) {
            case Channel::EMAIL: return make_unique<EmailNotification>(target);
            case Channel::SMS:   return make_unique<SMSNotification>(target);
            case Channel::PUSH:  return make_unique<PushNotification>(target);
            case Channel::SLACK: return make_unique<SlackNotification>(target);
        }
        throw invalid_argument("Unknown channel");
    }

    void sendAlert(Channel channel, const string& target, const string& message) {
        auto notification = createNotification(channel, target);
        notification->send(message);
    }

    virtual ~AlertService() = default;
};

// --- Subclass can override to customize creation ---
class PremiumAlertService : public AlertService {
public:
    unique_ptr<Notification> createNotification(Channel channel,
                                                 const string& target) override {
        // Premium service might add logging, retry logic, or use enhanced versions
        cout << "[Premium] Creating notification..." << endl;
        auto notification = AlertService::createNotification(channel, target);
        // Could wrap in a decorator for retry, logging, etc.
        return notification;
    }
};
```

---

### Factory Method with Registration (Self-Registering Factory)

A more extensible approach where products register themselves with the factory — no switch/if-else needed.

```cpp
#include <functional>
#include <unordered_map>

class Shape {
public:
    virtual void draw() const = 0;
    virtual double area() const = 0;
    virtual string name() const = 0;
    virtual ~Shape() = default;
};

// --- Self-registering Factory ---
class ShapeFactory {
public:
    using Creator = function<unique_ptr<Shape>(const unordered_map<string, double>&)>;

private:
    static unordered_map<string, Creator>& getRegistry() {
        static unordered_map<string, Creator> registry;
        return registry;
    }

public:
    // Register a new shape type
    static bool registerShape(const string& typeName, Creator creator) {
        getRegistry()[typeName] = creator;
        return true;
    }

    // Create a shape by registered name
    static unique_ptr<Shape> create(const string& typeName,
                                     const unordered_map<string, double>& params = {}) {
        auto& registry = getRegistry();
        auto it = registry.find(typeName);
        if (it == registry.end()) {
            throw invalid_argument("Unknown shape type: " + typeName);
        }
        return it->second(params);
    }

    // List all registered types
    static vector<string> getRegisteredTypes() {
        vector<string> types;
        for (const auto& [name, _] : getRegistry()) {
            types.push_back(name);
        }
        return types;
    }
};

// --- Concrete shapes that self-register ---
class Circle : public Shape {
    double radius;
public:
    Circle(double r) : radius(r) {}
    void draw() const override { cout << "Drawing circle r=" << radius << endl; }
    double area() const override { return 3.14159 * radius * radius; }
    string name() const override { return "Circle"; }

    // Self-registration
    static bool registered;
};

bool Circle::registered = ShapeFactory::registerShape("circle",
    [](const unordered_map<string, double>& params) -> unique_ptr<Shape> {
        double r = params.count("radius") ? params.at("radius") : 1.0;
        return make_unique<Circle>(r);
    });

class Rectangle : public Shape {
    double width, height;
public:
    Rectangle(double w, double h) : width(w), height(h) {}
    void draw() const override { cout << "Drawing rect " << width << "x" << height << endl; }
    double area() const override { return width * height; }
    string name() const override { return "Rectangle"; }

    static bool registered;
};

bool Rectangle::registered = ShapeFactory::registerShape("rectangle",
    [](const unordered_map<string, double>& params) -> unique_ptr<Shape> {
        double w = params.count("width") ? params.at("width") : 1.0;
        double h = params.count("height") ? params.at("height") : 1.0;
        return make_unique<Rectangle>(w, h);
    });

// Usage — adding new shapes requires NO modification to factory or client
auto circle = ShapeFactory::create("circle", {{"radius", 5.0}});
auto rect = ShapeFactory::create("rectangle", {{"width", 3.0}, {"height", 4.0}});
circle->draw();  // Drawing circle r=5
rect->draw();    // Drawing rect 3x4
```

**Benefits of self-registering factory:**
- Truly Open/Closed — add new shapes by just creating a new class file
- No switch/if-else chains
- Plugins can register new types at runtime
- Factory doesn't need to know about concrete types

---

### When to Use Factory Method

| Scenario | Why Factory Method Helps |
|----------|--------------------------|
| Don't know exact types at compile time | Defers decision to runtime/subclass |
| Want to provide extension points | Users subclass creator to add new products |
| Creation logic is complex | Encapsulates creation complexity |
| Need to enforce creation constraints | Factory can validate, log, cache |
| Want to decouple client from concrete classes | Client works with abstract product |
| Framework/library design | Let users customize object creation |

**Real-world examples:**
- GUI frameworks: `createButton()`, `createWindow()` — platform-specific
- Database drivers: `createConnection()` — DB-specific
- Serialization: `createParser()` — format-specific (JSON, XML, YAML)
- Game engines: `createEnemy()`, `createWeapon()` — level-specific

---


## 4.3 Abstract Factory Pattern

> The Abstract Factory pattern provides an interface for creating **families of related or dependent objects** without specifying their concrete classes. It's essentially a "factory of factories" — a super-factory that creates other factories.

---

### The Problem Abstract Factory Solves

When you have multiple families of related products and need to ensure that products from the same family are used together:

```cpp
// BAD: Mixing products from different families
WindowsButton* btn = new WindowsButton();   // Windows style
MacScrollbar* scroll = new MacScrollbar();   // Mac style — INCONSISTENT!
LinuxMenu* menu = new LinuxMenu();           // Linux style — CHAOS!

// The UI looks broken because components from different platforms are mixed
```

**The requirement:** Create entire families of objects that are designed to work together, and make it easy to switch between families.

---

### Structure

```
┌─────────────────────────┐
│    AbstractFactory       │
├─────────────────────────┤
│ + createButton()        │──────→ AbstractButton
│ + createCheckbox()      │──────→ AbstractCheckbox
│ + createTextInput()     │──────→ AbstractTextInput
└─────────────────────────┘
          ▲
          │
    ┌─────┴──────┐
    │            │
WindowsFactory  MacFactory
creates:        creates:
WindowsButton   MacButton
WindowsCheckbox MacCheckbox
WindowsTextInput MacTextInput
```

---

### Complete Implementation — Cross-Platform UI

```cpp
// ============================================================
// ABSTRACT PRODUCTS — define interfaces for each product type
// ============================================================

class Button {
public:
    virtual void render() const = 0;
    virtual void onClick(function<void()> handler) = 0;
    virtual void setLabel(const string& label) = 0;
    virtual ~Button() = default;
};

class Checkbox {
public:
    virtual void render() const = 0;
    virtual void toggle() = 0;
    virtual bool isChecked() const = 0;
    virtual void setLabel(const string& label) = 0;
    virtual ~Checkbox() = default;
};

class TextInput {
public:
    virtual void render() const = 0;
    virtual void setText(const string& text) = 0;
    virtual string getText() const = 0;
    virtual void setPlaceholder(const string& placeholder) = 0;
    virtual ~TextInput() = default;
};

class Dialog {
public:
    virtual void show(const string& title, const string& message) const = 0;
    virtual ~Dialog() = default;
};

// ============================================================
// CONCRETE PRODUCTS — Windows Family
// ============================================================

class WindowsButton : public Button {
    string label;
    function<void()> clickHandler;
public:
    void render() const override {
        cout << "[ " << label << " ]  (Windows flat button)" << endl;
    }
    void onClick(function<void()> handler) override { clickHandler = handler; }
    void setLabel(const string& l) override { label = l; }
};

class WindowsCheckbox : public Checkbox {
    string label;
    bool checked = false;
public:
    void render() const override {
        cout << (checked ? "[X] " : "[ ] ") << label << "  (Windows checkbox)" << endl;
    }
    void toggle() override { checked = !checked; }
    bool isChecked() const override { return checked; }
    void setLabel(const string& l) override { label = l; }
};

class WindowsTextInput : public TextInput {
    string text;
    string placeholder;
public:
    void render() const override {
        string display = text.empty() ? placeholder : text;
        cout << "|" << display << "|  (Windows text field)" << endl;
    }
    void setText(const string& t) override { text = t; }
    string getText() const override { return text; }
    void setPlaceholder(const string& p) override { placeholder = p; }
};

class WindowsDialog : public Dialog {
public:
    void show(const string& title, const string& message) const override {
        cout << "╔══ " << title << " ══╗" << endl;
        cout << "║ " << message << endl;
        cout << "║        [OK]        ║" << endl;
        cout << "╚════════════════════╝  (Windows dialog)" << endl;
    }
};

// ============================================================
// CONCRETE PRODUCTS — macOS Family
// ============================================================

class MacButton : public Button {
    string label;
    function<void()> clickHandler;
public:
    void render() const override {
        cout << "( " << label << " )  (macOS rounded button)" << endl;
    }
    void onClick(function<void()> handler) override { clickHandler = handler; }
    void setLabel(const string& l) override { label = l; }
};

class MacCheckbox : public Checkbox {
    string label;
    bool checked = false;
public:
    void render() const override {
        cout << (checked ? "☑ " : "☐ ") << label << "  (macOS checkbox)" << endl;
    }
    void toggle() override { checked = !checked; }
    bool isChecked() const override { return checked; }
    void setLabel(const string& l) override { label = l; }
};

class MacTextInput : public TextInput {
    string text;
    string placeholder;
public:
    void render() const override {
        string display = text.empty() ? placeholder : text;
        cout << "⌈" << display << "⌉  (macOS text field)" << endl;
    }
    void setText(const string& t) override { text = t; }
    string getText() const override { return text; }
    void setPlaceholder(const string& p) override { placeholder = p; }
};

class MacDialog : public Dialog {
public:
    void show(const string& title, const string& message) const override {
        cout << "┌── " << title << " ──┐" << endl;
        cout << "│ " << message << endl;
        cout << "│      ( OK )       │" << endl;
        cout << "└───────────────────┘  (macOS dialog)" << endl;
    }
};

// ============================================================
// ABSTRACT FACTORY — interface for creating product families
// ============================================================

class UIFactory {
public:
    virtual unique_ptr<Button> createButton() = 0;
    virtual unique_ptr<Checkbox> createCheckbox() = 0;
    virtual unique_ptr<TextInput> createTextInput() = 0;
    virtual unique_ptr<Dialog> createDialog() = 0;
    virtual string getThemeName() const = 0;
    virtual ~UIFactory() = default;
};

// ============================================================
// CONCRETE FACTORIES — each creates a complete family
// ============================================================

class WindowsUIFactory : public UIFactory {
public:
    unique_ptr<Button> createButton() override {
        return make_unique<WindowsButton>();
    }
    unique_ptr<Checkbox> createCheckbox() override {
        return make_unique<WindowsCheckbox>();
    }
    unique_ptr<TextInput> createTextInput() override {
        return make_unique<WindowsTextInput>();
    }
    unique_ptr<Dialog> createDialog() override {
        return make_unique<WindowsDialog>();
    }
    string getThemeName() const override { return "Windows"; }
};

class MacUIFactory : public UIFactory {
public:
    unique_ptr<Button> createButton() override {
        return make_unique<MacButton>();
    }
    unique_ptr<Checkbox> createCheckbox() override {
        return make_unique<MacCheckbox>();
    }
    unique_ptr<TextInput> createTextInput() override {
        return make_unique<MacTextInput>();
    }
    unique_ptr<Dialog> createDialog() override {
        return make_unique<MacDialog>();
    }
    string getThemeName() const override { return "macOS"; }
};

// ============================================================
// CLIENT CODE — works with factories and products via interfaces
// ============================================================

class SettingsPage {
    unique_ptr<TextInput> nameField;
    unique_ptr<TextInput> emailField;
    unique_ptr<Checkbox> darkModeToggle;
    unique_ptr<Checkbox> notificationsToggle;
    unique_ptr<Button> saveButton;
    unique_ptr<Button> cancelButton;
    unique_ptr<Dialog> confirmDialog;

public:
    // Receives factory — doesn't know or care which platform
    SettingsPage(UIFactory& factory) {
        nameField = factory.createTextInput();
        nameField->setPlaceholder("Enter your name");

        emailField = factory.createTextInput();
        emailField->setPlaceholder("Enter your email");

        darkModeToggle = factory.createCheckbox();
        darkModeToggle->setLabel("Dark Mode");

        notificationsToggle = factory.createCheckbox();
        notificationsToggle->setLabel("Enable Notifications");

        saveButton = factory.createButton();
        saveButton->setLabel("Save");

        cancelButton = factory.createButton();
        cancelButton->setLabel("Cancel");

        confirmDialog = factory.createDialog();
    }

    void render() const {
        cout << "=== Settings ===" << endl;
        nameField->render();
        emailField->render();
        darkModeToggle->render();
        notificationsToggle->render();
        saveButton->render();
        cancelButton->render();
    }

    void save() {
        confirmDialog->show("Confirm", "Save changes?");
    }
};

// --- Selecting the factory at runtime ---
unique_ptr<UIFactory> createFactory() {
    #ifdef _WIN32
        return make_unique<WindowsUIFactory>();
    #elif __APPLE__
        return make_unique<MacUIFactory>();
    #else
        return make_unique<WindowsUIFactory>();  // default
    #endif
}

int main() {
    auto factory = createFactory();
    cout << "Using theme: " << factory->getThemeName() << endl << endl;

    SettingsPage settings(*factory);
    settings.render();
    settings.save();
}
```

---

### Factory of Factories

When you have multiple abstract factories and need to select one dynamically:

```cpp
class UIFactoryProvider {
    static unordered_map<string, function<unique_ptr<UIFactory>()>>& getRegistry() {
        static unordered_map<string, function<unique_ptr<UIFactory>()>> registry;
        return registry;
    }

public:
    static void registerFactory(const string& name, function<unique_ptr<UIFactory>()> creator) {
        getRegistry()[name] = creator;
    }

    static unique_ptr<UIFactory> getFactory(const string& name) {
        auto& registry = getRegistry();
        auto it = registry.find(name);
        if (it == registry.end()) {
            throw invalid_argument("Unknown UI factory: " + name);
        }
        return it->second();
    }
};

// Registration (could be in separate files)
struct FactoryRegistrar {
    FactoryRegistrar() {
        UIFactoryProvider::registerFactory("windows", []() {
            return make_unique<WindowsUIFactory>();
        });
        UIFactoryProvider::registerFactory("macos", []() {
            return make_unique<MacUIFactory>();
        });
    }
};
static FactoryRegistrar registrar;  // registers at startup

// Usage — select factory by name (from config, user preference, etc.)
string theme = readConfig("ui.theme");  // "windows" or "macos"
auto factory = UIFactoryProvider::getFactory(theme);
SettingsPage page(*factory);
```

---

### Abstract Factory vs Factory Method

| Aspect | Factory Method | Abstract Factory |
|--------|---------------|-----------------|
| Creates | ONE product | FAMILY of related products |
| Method | Single virtual method | Multiple creation methods |
| Inheritance | Subclass overrides one method | Subclass implements all creation methods |
| Focus | How to create one object | How to create a consistent set of objects |
| Complexity | Simpler | More complex |
| Use case | One product varies | Multiple products must be consistent |
| Example | `createButton()` | `createButton()` + `createCheckbox()` + `createDialog()` |

**When to use Abstract Factory:**
- System must be independent of how products are created
- System needs to work with multiple families of products
- Products in a family are designed to work together (consistency constraint)
- You want to provide a library of products, revealing only interfaces

---

### Another Example — Database Access Layer

```cpp
// Abstract products
class Connection {
public:
    virtual bool connect(const string& connectionString) = 0;
    virtual void disconnect() = 0;
    virtual bool isConnected() const = 0;
    virtual ~Connection() = default;
};

class Command {
public:
    virtual void setQuery(const string& sql) = 0;
    virtual void addParameter(const string& name, const string& value) = 0;
    virtual int executeNonQuery() = 0;  // INSERT, UPDATE, DELETE
    virtual ~Command() = default;
};

class ResultSet {
public:
    virtual bool next() = 0;
    virtual string getString(const string& column) = 0;
    virtual int getInt(const string& column) = 0;
    virtual bool isNull(const string& column) = 0;
    virtual ~ResultSet() = default;
};

// Abstract Factory
class DatabaseFactory {
public:
    virtual unique_ptr<Connection> createConnection() = 0;
    virtual unique_ptr<Command> createCommand(Connection& conn) = 0;
    virtual string getDatabaseName() const = 0;
    virtual ~DatabaseFactory() = default;
};

// Concrete: PostgreSQL family
class PgConnection : public Connection {
    bool connected = false;
public:
    bool connect(const string& cs) override {
        cout << "PostgreSQL connecting to: " << cs << endl;
        connected = true;
        return true;
    }
    void disconnect() override { connected = false; }
    bool isConnected() const override { return connected; }
};

class PgCommand : public Command {
    string query;
    vector<pair<string, string>> params;
public:
    void setQuery(const string& sql) override { query = sql; }
    void addParameter(const string& name, const string& value) override {
        params.push_back({name, value});
    }
    int executeNonQuery() override {
        cout << "PG executing: " << query << " with " << params.size() << " params" << endl;
        return 1;  // rows affected
    }
};

class PostgreSQLFactory : public DatabaseFactory {
public:
    unique_ptr<Connection> createConnection() override {
        return make_unique<PgConnection>();
    }
    unique_ptr<Command> createCommand(Connection& conn) override {
        return make_unique<PgCommand>();
    }
    string getDatabaseName() const override { return "PostgreSQL"; }
};

// Concrete: MySQL family
class MySQLConnection : public Connection {
    bool connected = false;
public:
    bool connect(const string& cs) override {
        cout << "MySQL connecting to: " << cs << endl;
        connected = true;
        return true;
    }
    void disconnect() override { connected = false; }
    bool isConnected() const override { return connected; }
};

class MySQLCommand : public Command {
    string query;
    vector<pair<string, string>> params;
public:
    void setQuery(const string& sql) override { query = sql; }
    void addParameter(const string& name, const string& value) override {
        params.push_back({name, value});
    }
    int executeNonQuery() override {
        cout << "MySQL executing: " << query << " with " << params.size() << " params" << endl;
        return 1;
    }
};

class MySQLFactory : public DatabaseFactory {
public:
    unique_ptr<Connection> createConnection() override {
        return make_unique<MySQLConnection>();
    }
    unique_ptr<Command> createCommand(Connection& conn) override {
        return make_unique<MySQLCommand>();
    }
    string getDatabaseName() const override { return "MySQL"; }
};

// Client code — completely database-agnostic
class UserRepository {
    DatabaseFactory& dbFactory;
    string connectionString;

public:
    UserRepository(DatabaseFactory& factory, const string& connStr)
        : dbFactory(factory), connectionString(connStr) {}

    void createUser(const string& name, const string& email) {
        auto conn = dbFactory.createConnection();
        conn->connect(connectionString);

        auto cmd = dbFactory.createCommand(*conn);
        cmd->setQuery("INSERT INTO users (name, email) VALUES ($1, $2)");
        cmd->addParameter("name", name);
        cmd->addParameter("email", email);
        cmd->executeNonQuery();

        conn->disconnect();
    }
};

// Switch databases by changing one line
PostgreSQLFactory pgFactory;
// MySQLFactory mysqlFactory;
UserRepository repo(pgFactory, "host=localhost dbname=myapp");
repo.createUser("Alice", "alice@example.com");
```

---


## 4.4 Builder Pattern

> The Builder pattern separates the construction of a complex object from its representation, allowing the same construction process to create different representations. It's ideal when an object requires many steps to create, or when you want to construct different variations of an object using the same building process.

---

### The Problem Builder Solves

```cpp
// BAD: Constructor with too many parameters — "telescoping constructor" anti-pattern
class Pizza {
public:
    Pizza(string size, string crust, string sauce, bool cheese, bool pepperoni,
          bool mushrooms, bool onions, bool olives, bool peppers, bool bacon,
          bool pineapple, bool extraCheese, bool thinCrust) {
        // Which bool is which?! Easy to mix up parameters.
    }
};

// Calling this is error-prone and unreadable:
Pizza p("large", "thin", "tomato", true, true, false, true, false, false, true, false, true, false);
// What does the 7th 'false' mean? Nobody knows without checking the constructor.
```

**Problems:**
1. Too many constructor parameters — hard to read, easy to mix up
2. Many parameters are optional — leads to multiple constructor overloads
3. Object might be in an inconsistent state during construction
4. No way to enforce construction order or validate intermediate state

---

### Basic Builder Pattern

```cpp
// --- The Product (complex object to build) ---
class HttpRequest {
public:
    string method;
    string url;
    unordered_map<string, string> headers;
    string body;
    int timeout;
    bool followRedirects;
    int maxRetries;
    string authToken;

    void send() const {
        cout << method << " " << url << endl;
        for (const auto& [key, value] : headers) {
            cout << "  " << key << ": " << value << endl;
        }
        if (!body.empty()) cout << "  Body: " << body << endl;
        cout << "  Timeout: " << timeout << "ms, Retries: " << maxRetries << endl;
    }
};

// --- The Builder ---
class HttpRequestBuilder {
    HttpRequest request;

public:
    HttpRequestBuilder() {
        // Set defaults
        request.method = "GET";
        request.timeout = 30000;
        request.followRedirects = true;
        request.maxRetries = 3;
    }

    HttpRequestBuilder& setMethod(const string& method) {
        request.method = method;
        return *this;
    }

    HttpRequestBuilder& setUrl(const string& url) {
        request.url = url;
        return *this;
    }

    HttpRequestBuilder& addHeader(const string& key, const string& value) {
        request.headers[key] = value;
        return *this;
    }

    HttpRequestBuilder& setBody(const string& body) {
        request.body = body;
        return *this;
    }

    HttpRequestBuilder& setTimeout(int ms) {
        if (ms <= 0) throw invalid_argument("Timeout must be positive");
        request.timeout = ms;
        return *this;
    }

    HttpRequestBuilder& setFollowRedirects(bool follow) {
        request.followRedirects = follow;
        return *this;
    }

    HttpRequestBuilder& setMaxRetries(int retries) {
        if (retries < 0) throw invalid_argument("Retries cannot be negative");
        request.maxRetries = retries;
        return *this;
    }

    HttpRequestBuilder& setAuthToken(const string& token) {
        request.authToken = token;
        request.headers["Authorization"] = "Bearer " + token;
        return *this;
    }

    // Build and validate
    HttpRequest build() {
        if (request.url.empty()) {
            throw runtime_error("URL is required");
        }
        if (request.method == "POST" || request.method == "PUT") {
            if (request.headers.find("Content-Type") == request.headers.end()) {
                request.headers["Content-Type"] = "application/json";
            }
        }
        return request;
    }
};

// Usage — clear, readable, self-documenting
HttpRequest req = HttpRequestBuilder()
    .setMethod("POST")
    .setUrl("https://api.example.com/users")
    .addHeader("Content-Type", "application/json")
    .addHeader("Accept", "application/json")
    .setBody(R"({"name": "Alice", "email": "alice@example.com"})")
    .setAuthToken("abc123xyz")
    .setTimeout(5000)
    .setMaxRetries(2)
    .build();

req.send();
```

---

### Director Class

The **Director** defines the ORDER in which to call building steps. It encapsulates specific construction recipes, while the builder provides the implementation of each step.

```cpp
// --- Product ---
class House {
public:
    string foundation;
    string structure;
    string roof;
    string interior;
    string exterior;
    int floors;
    bool hasGarage;
    bool hasSwimmingPool;
    bool hasGarden;
    string heatingSystem;

    void describe() const {
        cout << "House: " << floors << " floors" << endl;
        cout << "  Foundation: " << foundation << endl;
        cout << "  Structure: " << structure << endl;
        cout << "  Roof: " << roof << endl;
        cout << "  Interior: " << interior << endl;
        cout << "  Exterior: " << exterior << endl;
        cout << "  Heating: " << heatingSystem << endl;
        cout << "  Garage: " << (hasGarage ? "Yes" : "No") << endl;
        cout << "  Pool: " << (hasSwimmingPool ? "Yes" : "No") << endl;
        cout << "  Garden: " << (hasGarden ? "Yes" : "No") << endl;
    }
};

// --- Abstract Builder ---
class HouseBuilder {
protected:
    House house;

public:
    virtual ~HouseBuilder() = default;

    virtual HouseBuilder& buildFoundation() = 0;
    virtual HouseBuilder& buildStructure() = 0;
    virtual HouseBuilder& buildRoof() = 0;
    virtual HouseBuilder& buildInterior() = 0;
    virtual HouseBuilder& buildExterior() = 0;
    virtual HouseBuilder& setFloors(int n) { house.floors = n; return *this; }
    virtual HouseBuilder& addGarage() { house.hasGarage = true; return *this; }
    virtual HouseBuilder& addSwimmingPool() { house.hasSwimmingPool = true; return *this; }
    virtual HouseBuilder& addGarden() { house.hasGarden = true; return *this; }
    virtual HouseBuilder& setHeating(const string& type) { house.heatingSystem = type; return *this; }

    House getResult() {
        return house;
    }

    void reset() {
        house = House{};
    }
};

// --- Concrete Builder: Wooden House ---
class WoodenHouseBuilder : public HouseBuilder {
public:
    HouseBuilder& buildFoundation() override {
        house.foundation = "Concrete slab with wooden posts";
        return *this;
    }
    HouseBuilder& buildStructure() override {
        house.structure = "Timber frame with wooden walls";
        return *this;
    }
    HouseBuilder& buildRoof() override {
        house.roof = "Wooden shingles";
        return *this;
    }
    HouseBuilder& buildInterior() override {
        house.interior = "Hardwood floors, wooden paneling";
        return *this;
    }
    HouseBuilder& buildExterior() override {
        house.exterior = "Cedar wood siding";
        return *this;
    }
};

// --- Concrete Builder: Stone House ---
class StoneHouseBuilder : public HouseBuilder {
public:
    HouseBuilder& buildFoundation() override {
        house.foundation = "Deep concrete foundation with rebar";
        return *this;
    }
    HouseBuilder& buildStructure() override {
        house.structure = "Stone walls with steel reinforcement";
        return *this;
    }
    HouseBuilder& buildRoof() override {
        house.roof = "Slate tiles";
        return *this;
    }
    HouseBuilder& buildInterior() override {
        house.interior = "Marble floors, plastered walls";
        return *this;
    }
    HouseBuilder& buildExterior() override {
        house.exterior = "Natural stone facade";
        return *this;
    }
};

// --- Director: defines construction recipes ---
class ConstructionDirector {
    HouseBuilder* builder;

public:
    void setBuilder(HouseBuilder* b) { builder = b; }

    // Recipe: Simple house
    House constructSimpleHouse() {
        builder->reset();
        builder->buildFoundation()
               .buildStructure()
               .buildRoof()
               .buildInterior()
               .buildExterior()
               .setFloors(1)
               .setHeating("Electric");
        return builder->getResult();
    }

    // Recipe: Luxury house
    House constructLuxuryHouse() {
        builder->reset();
        builder->buildFoundation()
               .buildStructure()
               .buildRoof()
               .buildInterior()
               .buildExterior()
               .setFloors(3)
               .addGarage()
               .addSwimmingPool()
               .addGarden()
               .setHeating("Underfloor radiant");
        return builder->getResult();
    }

    // Recipe: Minimal cabin
    House constructCabin() {
        builder->reset();
        builder->buildFoundation()
               .buildStructure()
               .buildRoof()
               .setFloors(1)
               .setHeating("Wood stove");
        return builder->getResult();
    }
};

// Usage
WoodenHouseBuilder woodBuilder;
StoneHouseBuilder stoneBuilder;
ConstructionDirector director;

director.setBuilder(&woodBuilder);
House woodenCabin = director.constructCabin();
woodenCabin.describe();

director.setBuilder(&stoneBuilder);
House stoneMansion = director.constructLuxuryHouse();
stoneMansion.describe();
```

**Director benefits:**
- Encapsulates construction algorithms (recipes)
- Same director can work with different builders
- Client doesn't need to know the construction steps
- Easy to add new recipes without changing builders

---

### Fluent Builder (Method Chaining)

The most common modern usage — each setter returns `*this` (reference to the builder), enabling chained calls.

```cpp
class QueryBuilder {
    string table;
    vector<string> columns;
    vector<string> conditions;
    vector<string> orderBy;
    int limitVal = -1;
    int offsetVal = -1;
    vector<string> joins;
    bool distinct = false;

public:
    QueryBuilder& select(const vector<string>& cols) {
        columns = cols;
        return *this;
    }

    QueryBuilder& selectAll() {
        columns = {"*"};
        return *this;
    }

    QueryBuilder& from(const string& tableName) {
        table = tableName;
        return *this;
    }

    QueryBuilder& where(const string& condition) {
        conditions.push_back(condition);
        return *this;
    }

    QueryBuilder& andWhere(const string& condition) {
        conditions.push_back("AND " + condition);
        return *this;
    }

    QueryBuilder& orWhere(const string& condition) {
        conditions.push_back("OR " + condition);
        return *this;
    }

    QueryBuilder& join(const string& joinClause) {
        joins.push_back("JOIN " + joinClause);
        return *this;
    }

    QueryBuilder& leftJoin(const string& joinClause) {
        joins.push_back("LEFT JOIN " + joinClause);
        return *this;
    }

    QueryBuilder& orderByAsc(const string& column) {
        orderBy.push_back(column + " ASC");
        return *this;
    }

    QueryBuilder& orderByDesc(const string& column) {
        orderBy.push_back(column + " DESC");
        return *this;
    }

    QueryBuilder& limit(int n) {
        limitVal = n;
        return *this;
    }

    QueryBuilder& offset(int n) {
        offsetVal = n;
        return *this;
    }

    QueryBuilder& setDistinct(bool d = true) {
        distinct = d;
        return *this;
    }

    string build() const {
        if (table.empty()) throw runtime_error("Table name is required");

        string query = "SELECT ";
        if (distinct) query += "DISTINCT ";

        if (columns.empty()) query += "*";
        else {
            for (size_t i = 0; i < columns.size(); i++) {
                if (i > 0) query += ", ";
                query += columns[i];
            }
        }

        query += " FROM " + table;

        for (const auto& j : joins) {
            query += " " + j;
        }

        if (!conditions.empty()) {
            query += " WHERE ";
            for (size_t i = 0; i < conditions.size(); i++) {
                query += conditions[i];
                if (i < conditions.size() - 1) query += " ";
            }
        }

        if (!orderBy.empty()) {
            query += " ORDER BY ";
            for (size_t i = 0; i < orderBy.size(); i++) {
                if (i > 0) query += ", ";
                query += orderBy[i];
            }
        }

        if (limitVal > 0) query += " LIMIT " + to_string(limitVal);
        if (offsetVal > 0) query += " OFFSET " + to_string(offsetVal);

        return query;
    }
};

// Usage — reads like a DSL (Domain-Specific Language)
string query = QueryBuilder()
    .select({"u.name", "u.email", "o.total"})
    .from("users u")
    .leftJoin("orders o ON u.id = o.user_id")
    .where("u.active = true")
    .andWhere("o.total > 100")
    .orderByDesc("o.total")
    .limit(10)
    .offset(20)
    .build();

cout << query << endl;
// SELECT u.name, u.email, o.total FROM users u LEFT JOIN orders o ON u.id = o.user_id
// WHERE u.active = true AND o.total > 100 ORDER BY o.total DESC LIMIT 10 OFFSET 20
```

---

### Builder vs Constructor with Many Parameters

| Approach | Pros | Cons |
|----------|------|------|
| **Many-param constructor** | Simple, one line | Unreadable, error-prone, all params required |
| **Multiple constructors** | Different combos | Combinatorial explosion, still unclear |
| **Setter methods** | Flexible, clear names | Object in invalid state between setters |
| **Builder** | Readable, validates, immutable product | More code, extra class |

**When Builder wins:**
```cpp
// Constructor approach — which is width and which is height?
Window w(800, 600, true, false, "Main", nullptr, 2, true);

// Builder approach — self-documenting
Window w = WindowBuilder()
    .setWidth(800)
    .setHeight(600)
    .setResizable(true)
    .setFullscreen(false)
    .setTitle("Main")
    .setBorderWidth(2)
    .setVisible(true)
    .build();
```

---

### Builder for Immutable Objects

A common pattern: the builder is the ONLY way to create the object, and the resulting object is immutable.

```cpp
class ServerConfig {
    // All fields are const — truly immutable after construction
    const string host;
    const int port;
    const int maxConnections;
    const int timeout;
    const bool useTLS;
    const string certPath;
    const string keyPath;
    const vector<string> allowedOrigins;

    // Private constructor — only Builder can create
    ServerConfig(const string& h, int p, int mc, int t, bool tls,
                 const string& cert, const string& key, const vector<string>& origins)
        : host(h), port(p), maxConnections(mc), timeout(t), useTLS(tls),
          certPath(cert), keyPath(key), allowedOrigins(origins) {}

    friend class ServerConfigBuilder;  // only builder can access private ctor

public:
    // Only getters — no setters
    const string& getHost() const { return host; }
    int getPort() const { return port; }
    int getMaxConnections() const { return maxConnections; }
    int getTimeout() const { return timeout; }
    bool isTLS() const { return useTLS; }
    const string& getCertPath() const { return certPath; }
    const string& getKeyPath() const { return keyPath; }
    const vector<string>& getAllowedOrigins() const { return allowedOrigins; }
};

class ServerConfigBuilder {
    string host = "0.0.0.0";
    int port = 8080;
    int maxConnections = 100;
    int timeout = 30000;
    bool useTLS = false;
    string certPath;
    string keyPath;
    vector<string> allowedOrigins;

public:
    ServerConfigBuilder& setHost(const string& h) { host = h; return *this; }
    ServerConfigBuilder& setPort(int p) { port = p; return *this; }
    ServerConfigBuilder& setMaxConnections(int mc) { maxConnections = mc; return *this; }
    ServerConfigBuilder& setTimeout(int t) { timeout = t; return *this; }
    ServerConfigBuilder& enableTLS(const string& cert, const string& key) {
        useTLS = true;
        certPath = cert;
        keyPath = key;
        return *this;
    }
    ServerConfigBuilder& addAllowedOrigin(const string& origin) {
        allowedOrigins.push_back(origin);
        return *this;
    }

    ServerConfig build() const {
        // Validation
        if (host.empty()) throw runtime_error("Host is required");
        if (port <= 0 || port > 65535) throw runtime_error("Invalid port");
        if (useTLS && (certPath.empty() || keyPath.empty())) {
            throw runtime_error("TLS requires cert and key paths");
        }
        if (maxConnections <= 0) throw runtime_error("Max connections must be positive");

        return ServerConfig(host, port, maxConnections, timeout, useTLS,
                           certPath, keyPath, allowedOrigins);
    }
};

// Usage
ServerConfig config = ServerConfigBuilder()
    .setHost("api.example.com")
    .setPort(443)
    .setMaxConnections(1000)
    .setTimeout(5000)
    .enableTLS("/etc/ssl/cert.pem", "/etc/ssl/key.pem")
    .addAllowedOrigin("https://app.example.com")
    .addAllowedOrigin("https://admin.example.com")
    .build();

// config is now immutable — thread-safe, predictable
```

---

### When to Use Builder Pattern

| Scenario | Why Builder Helps |
|----------|-------------------|
| Object with many optional parameters | Avoids telescoping constructors |
| Complex construction with validation | Validates at build time |
| Need to create immutable objects | Builder accumulates state, product is frozen |
| Same construction process, different representations | Director + different builders |
| Object requires step-by-step construction | Each step is a method call |
| Want readable, self-documenting construction | Method names describe what's being set |

**Real-world examples:**
- HTTP request builders (OkHttp, libcurl wrappers)
- SQL query builders
- Configuration objects
- UI layout builders
- Test data builders (test fixtures)
- Protocol buffer messages

---


## 4.5 Prototype Pattern

> The Prototype pattern creates new objects by cloning an existing object (the prototype) rather than creating from scratch. It's useful when object creation is expensive, or when you want to create objects whose type is determined at runtime.

---

### The Problem Prototype Solves

```cpp
// Problem 1: Expensive object creation
// Creating a game character involves loading textures, animations, AI models...
// Creating 100 enemies from scratch = 100 expensive loads
// Cloning a prototype = 1 load + 99 cheap copies

// Problem 2: Unknown concrete type at compile time
void duplicateShape(Shape* shape) {
    // How do you create a copy? You don't know if it's Circle, Rectangle, etc.
    // Shape* copy = new ???(*shape);  // Can't do this!

    // With Prototype pattern:
    Shape* copy = shape->clone();  // Works regardless of actual type!
}
```

---

### Deep Copy vs Shallow Copy

Understanding the difference is critical for implementing Prototype correctly.

```cpp
class Document {
    string title;
    string* content;          // dynamically allocated
    vector<string> tags;
    map<string, string> metadata;

public:
    Document(const string& t, const string& c)
        : title(t), content(new string(c)) {}

    // SHALLOW COPY — copies pointers, not pointed-to data
    // Both original and copy share the same 'content' memory!
    Document shallowCopy() const {
        Document copy("", "");
        copy.title = title;
        copy.content = content;  // DANGER: shared pointer!
        copy.tags = tags;        // vector does deep copy (it's a value type)
        copy.metadata = metadata;
        return copy;
    }

    // DEEP COPY — copies everything, including pointed-to data
    Document deepCopy() const {
        Document copy(title, *content);  // new string allocated
        copy.tags = tags;
        copy.metadata = metadata;
        return copy;
    }

    ~Document() { delete content; }

    void setContent(const string& c) { *content = c; }
    string getContent() const { return *content; }
    string getTitle() const { return title; }
};
```

```
SHALLOW COPY:
┌──────────────┐          ┌─────────────────────┐
│ original     │          │ "Hello World"       │
│ content ─────┼─────────→│                     │
└──────────────┘          └─────────────────────┘
                                    ↑
┌──────────────┐                    │
│ copy         │                    │
│ content ─────┼────────────────────┘  (SHARED! Modifying one affects both!)
└──────────────┘

DEEP COPY:
┌──────────────┐          ┌─────────────────────┐
│ original     │          │ "Hello World"       │
│ content ─────┼─────────→│                     │
└──────────────┘          └─────────────────────┘

┌──────────────┐          ┌─────────────────────┐
│ copy         │          │ "Hello World"       │  (INDEPENDENT copy)
│ content ─────┼─────────→│                     │
└──────────────┘          └─────────────────────┘
```

---

### Prototype Pattern with Virtual `clone()`

The standard implementation uses a virtual `clone()` method that each concrete class overrides to copy itself.

```cpp
// --- Prototype interface ---
class Shape {
protected:
    int x, y;
    string color;

public:
    Shape() : x(0), y(0), color("black") {}
    Shape(int x, int y, const string& color) : x(x), y(y), color(color) {}

    // THE PROTOTYPE METHOD — virtual clone
    virtual unique_ptr<Shape> clone() const = 0;

    virtual void draw() const = 0;
    virtual string getType() const = 0;
    virtual double area() const = 0;

    void moveTo(int newX, int newY) { x = newX; y = newY; }
    void setColor(const string& c) { color = c; }

    virtual ~Shape() = default;
};

// --- Concrete Prototypes ---
class Circle : public Shape {
    double radius;

public:
    Circle(int x, int y, double r, const string& color)
        : Shape(x, y, color), radius(r) {}

    // Clone creates a deep copy of this specific type
    unique_ptr<Shape> clone() const override {
        return make_unique<Circle>(*this);  // uses copy constructor
    }

    void draw() const override {
        cout << "Circle at (" << x << "," << y << ") r=" << radius
             << " color=" << color << endl;
    }

    string getType() const override { return "Circle"; }
    double area() const override { return 3.14159 * radius * radius; }

    void setRadius(double r) { radius = r; }
};

class Rectangle : public Shape {
    double width, height;

public:
    Rectangle(int x, int y, double w, double h, const string& color)
        : Shape(x, y, color), width(w), height(h) {}

    unique_ptr<Shape> clone() const override {
        return make_unique<Rectangle>(*this);
    }

    void draw() const override {
        cout << "Rectangle at (" << x << "," << y << ") " << width << "x" << height
             << " color=" << color << endl;
    }

    string getType() const override { return "Rectangle"; }
    double area() const override { return width * height; }
};

class Triangle : public Shape {
    double base, height_val;

public:
    Triangle(int x, int y, double b, double h, const string& color)
        : Shape(x, y, color), base(b), height_val(h) {}

    unique_ptr<Shape> clone() const override {
        return make_unique<Triangle>(*this);
    }

    void draw() const override {
        cout << "Triangle at (" << x << "," << y << ") base=" << base
             << " height=" << height_val << " color=" << color << endl;
    }

    string getType() const override { return "Triangle"; }
    double area() const override { return 0.5 * base * height_val; }
};

// --- Using Prototype to duplicate unknown shapes ---
void duplicateAndMove(const Shape& original, int offsetX, int offsetY) {
    auto copy = original.clone();  // don't need to know concrete type!
    copy->moveTo(offsetX, offsetY);
    copy->draw();
}

// Usage
Circle c(10, 20, 5.0, "red");
Rectangle r(0, 0, 100, 50, "blue");

duplicateAndMove(c, 50, 50);  // clones the circle, moves copy
duplicateAndMove(r, 200, 100); // clones the rectangle, moves copy

// Polymorphic cloning
vector<unique_ptr<Shape>> shapes;
shapes.push_back(make_unique<Circle>(0, 0, 10, "red"));
shapes.push_back(make_unique<Rectangle>(0, 0, 20, 30, "green"));
shapes.push_back(make_unique<Triangle>(0, 0, 15, 20, "blue"));

// Clone all shapes
vector<unique_ptr<Shape>> copies;
for (const auto& shape : shapes) {
    copies.push_back(shape->clone());  // each clones correctly!
}
```

---

### Prototype Registry (Prototype Manager)

A registry that stores pre-configured prototypes by name, allowing clients to clone them by key.

```cpp
class ShapeRegistry {
    unordered_map<string, unique_ptr<Shape>> prototypes;

public:
    void registerPrototype(const string& name, unique_ptr<Shape> prototype) {
        prototypes[name] = std::move(prototype);
    }

    unique_ptr<Shape> create(const string& name) const {
        auto it = prototypes.find(name);
        if (it == prototypes.end()) {
            throw invalid_argument("Unknown prototype: " + name);
        }
        return it->second->clone();
    }

    bool hasPrototype(const string& name) const {
        return prototypes.count(name) > 0;
    }

    vector<string> getAvailablePrototypes() const {
        vector<string> names;
        for (const auto& [name, _] : prototypes) {
            names.push_back(name);
        }
        return names;
    }
};

// Setup registry with pre-configured prototypes
ShapeRegistry registry;
registry.registerPrototype("small-red-circle",
    make_unique<Circle>(0, 0, 5, "red"));
registry.registerPrototype("large-blue-circle",
    make_unique<Circle>(0, 0, 50, "blue"));
registry.registerPrototype("standard-button",
    make_unique<Rectangle>(0, 0, 120, 40, "gray"));
registry.registerPrototype("icon-frame",
    make_unique<Rectangle>(0, 0, 32, 32, "transparent"));

// Create shapes from registry — no need to know construction details
auto shape1 = registry.create("small-red-circle");
shape1->moveTo(100, 200);
shape1->draw();

auto shape2 = registry.create("standard-button");
shape2->moveTo(50, 300);
shape2->draw();

// Create many copies efficiently
for (int i = 0; i < 100; i++) {
    auto enemy = registry.create("small-red-circle");
    enemy->moveTo(i * 20, 0);
}
```

---

### Deep Clone with Complex Object Graphs

When objects contain pointers to other objects, cloning requires careful deep copying of the entire object graph.

```cpp
class Address {
public:
    string street;
    string city;
    string country;

    Address(const string& s, const string& c, const string& co)
        : street(s), city(c), country(co) {}

    unique_ptr<Address> clone() const {
        return make_unique<Address>(street, city, country);
    }
};

class Department {
public:
    string name;
    int floor;

    Department(const string& n, int f) : name(n), floor(f) {}

    unique_ptr<Department> clone() const {
        return make_unique<Department>(name, floor);
    }
};

class Employee {
    string name;
    int id;
    double salary;
    unique_ptr<Address> address;       // owned, must deep copy
    shared_ptr<Department> department;  // shared, shallow copy is OK

public:
    Employee(const string& n, int id, double sal,
             unique_ptr<Address> addr, shared_ptr<Department> dept)
        : name(n), id(id), salary(sal),
          address(std::move(addr)), department(dept) {}

    // Deep clone — copies address (owned), shares department (shared)
    unique_ptr<Employee> clone() const {
        return make_unique<Employee>(
            name,
            id,
            salary,
            address->clone(),    // deep copy of owned object
            department           // shared_ptr copy (shared ownership)
        );
    }

    // Clone with modifications
    unique_ptr<Employee> cloneWithNewId(int newId) const {
        auto copy = clone();
        copy->id = newId;
        return copy;
    }

    void setAddress(unique_ptr<Address> addr) { address = std::move(addr); }
    void setSalary(double s) { salary = s; }

    void print() const {
        cout << name << " (ID: " << id << ") $" << salary << endl;
        cout << "  Address: " << address->street << ", " << address->city << endl;
        cout << "  Department: " << department->name << " (Floor " << department->floor << ")" << endl;
    }
};

// Usage
auto dept = make_shared<Department>("Engineering", 3);
auto emp1 = make_unique<Employee>(
    "Alice", 101, 95000,
    make_unique<Address>("123 Main St", "Seattle", "USA"),
    dept
);

// Clone and customize
auto emp2 = emp1->cloneWithNewId(102);
emp2->setSalary(90000);
emp2->setAddress(make_unique<Address>("456 Oak Ave", "Portland", "USA"));

emp1->print();
emp2->print();
// emp1 and emp2 share the same Department but have independent Addresses
```

---

### Prototype with Copy Constructors

In C++, the copy constructor is the natural mechanism for cloning. The `clone()` method typically delegates to it:

```cpp
class GameCharacter {
    string name;
    int health;
    int attack;
    int defense;
    vector<string> inventory;
    unordered_map<string, int> skills;
    unique_ptr<Weapon> weapon;  // owned resource

public:
    GameCharacter(const string& n, int hp, int atk, int def)
        : name(n), health(hp), attack(atk), defense(def) {}

    // Copy constructor — performs deep copy
    GameCharacter(const GameCharacter& other)
        : name(other.name),
          health(other.health),
          attack(other.attack),
          defense(other.defense),
          inventory(other.inventory),    // vector copies elements
          skills(other.skills),          // map copies entries
          weapon(other.weapon ? other.weapon->clone() : nullptr)  // deep copy pointer
    {}

    // Clone method delegates to copy constructor
    unique_ptr<GameCharacter> clone() const {
        return make_unique<GameCharacter>(*this);
    }

    void addItem(const string& item) { inventory.push_back(item); }
    void addSkill(const string& skill, int level) { skills[skill] = level; }
    void setName(const string& n) { name = n; }

    void describe() const {
        cout << name << " [HP:" << health << " ATK:" << attack << " DEF:" << defense << "]" << endl;
        cout << "  Inventory: ";
        for (const auto& item : inventory) cout << item << ", ";
        cout << endl;
        cout << "  Skills: ";
        for (const auto& [skill, level] : skills) cout << skill << "(" << level << ") ";
        cout << endl;
    }
};

// Create a template character (prototype)
GameCharacter warriorTemplate("Warrior", 100, 15, 10);
warriorTemplate.addItem("Sword");
warriorTemplate.addItem("Shield");
warriorTemplate.addSkill("Slash", 1);
warriorTemplate.addSkill("Block", 1);

// Clone and customize for each instance
auto warrior1 = warriorTemplate.clone();
warrior1->setName("Aragorn");
warrior1->addSkill("Leadership", 3);

auto warrior2 = warriorTemplate.clone();
warrior2->setName("Boromir");
warrior2->addItem("Horn of Gondor");

warrior1->describe();
warrior2->describe();
// Both have base warrior stats/items, but with individual customizations
```

---

### When to Use Prototype Pattern

| Scenario | Why Prototype Helps |
|----------|---------------------|
| Object creation is expensive | Clone is cheaper than creating from scratch |
| Need to create objects whose type is unknown at compile time | `clone()` handles polymorphic copying |
| Want to avoid parallel class hierarchies of factories | One `clone()` method replaces factory classes |
| Need many similar objects with slight variations | Clone template, modify differences |
| System should be independent of how products are created | Clone decouples creation from concrete types |
| Want to add/remove products at runtime | Registry of prototypes can be modified dynamically |

**Real-world examples:**
- Game development: spawning enemies from templates
- Document editors: copy-paste operations
- GUI: duplicating widgets/components
- Database: creating records from templates
- Configuration: deriving configs from base templates

**Prototype vs other creational patterns:**
| Pattern | When to Use |
|---------|-------------|
| Prototype | When cloning is simpler/cheaper than constructing |
| Factory Method | When you know the creation steps but not the type |
| Abstract Factory | When you need families of consistent objects |
| Builder | When construction is complex and step-by-step |

---


## 4.6 Object Pool Pattern

> The Object Pool pattern manages a set of reusable objects. Instead of creating and destroying objects on demand (which can be expensive), the pool maintains a collection of pre-created objects that can be "checked out" when needed and "returned" when done.

---

### The Problem Object Pool Solves

```cpp
// BAD: Creating and destroying expensive objects repeatedly
void handleRequest(const string& query) {
    // Each request creates a new connection — EXPENSIVE!
    DatabaseConnection* conn = new DatabaseConnection("host=localhost");  // ~50ms
    conn->connect();                                                       // ~200ms
    auto result = conn->execute(query);
    conn->disconnect();                                                    // ~50ms
    delete conn;                                                           // cleanup
    // Total overhead: ~300ms per request just for connection management!
}

// With 1000 requests/second, that's 300 seconds of wasted time per second!
```

**Objects that are expensive to create:**
- Database connections (TCP handshake, authentication, SSL)
- Thread objects (OS kernel call, stack allocation)
- Network sockets
- Large memory buffers
- GPU resources (textures, shaders)
- File handles

---

### Basic Object Pool Implementation

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>

template <typename T>
class ObjectPool {
    queue<unique_ptr<T>> available;
    mutex poolMutex;
    condition_variable cv;
    int maxSize;
    int currentSize = 0;
    function<unique_ptr<T>()> factory;       // creates new objects
    function<void(T&)> resetFunc;            // resets object to clean state

public:
    ObjectPool(int maxPoolSize,
               function<unique_ptr<T>()> createFunc,
               function<void(T&)> reset = nullptr)
        : maxSize(maxPoolSize), factory(createFunc), resetFunc(reset) {}

    // Acquire an object from the pool
    unique_ptr<T> acquire() {
        unique_lock<mutex> lock(poolMutex);

        // If objects available, return one
        if (!available.empty()) {
            auto obj = std::move(available.front());
            available.pop();
            return obj;
        }

        // If pool not at max capacity, create a new object
        if (currentSize < maxSize) {
            currentSize++;
            lock.unlock();
            return factory();
        }

        // Pool exhausted — wait for an object to be returned
        cv.wait(lock, [this]() { return !available.empty(); });
        auto obj = std::move(available.front());
        available.pop();
        return obj;
    }

    // Acquire with timeout — returns nullptr if timeout expires
    unique_ptr<T> acquireWithTimeout(chrono::milliseconds timeout) {
        unique_lock<mutex> lock(poolMutex);

        if (!available.empty()) {
            auto obj = std::move(available.front());
            available.pop();
            return obj;
        }

        if (currentSize < maxSize) {
            currentSize++;
            lock.unlock();
            return factory();
        }

        // Wait with timeout
        if (cv.wait_for(lock, timeout, [this]() { return !available.empty(); })) {
            auto obj = std::move(available.front());
            available.pop();
            return obj;
        }

        return nullptr;  // timeout expired
    }

    // Return an object to the pool
    void release(unique_ptr<T> obj) {
        if (!obj) return;

        // Reset object to clean state before returning to pool
        if (resetFunc) {
            resetFunc(*obj);
        }

        {
            lock_guard<mutex> lock(poolMutex);
            available.push(std::move(obj));
        }
        cv.notify_one();  // wake up one waiting thread
    }

    // Pool statistics
    int getAvailableCount() const {
        lock_guard<mutex> lock(const_cast<mutex&>(poolMutex));
        return available.size();
    }

    int getTotalSize() const { return currentSize; }
    int getMaxSize() const { return maxSize; }
};
```

---

### Connection Pool Example

```cpp
class DatabaseConnection {
    string connectionString;
    bool connected = false;
    int queryCount = 0;

public:
    DatabaseConnection(const string& connStr) : connectionString(connStr) {
        cout << "  [DB] Creating connection (expensive!)..." << endl;
        // Simulate expensive creation
        this_thread::sleep_for(chrono::milliseconds(100));
    }

    ~DatabaseConnection() {
        if (connected) disconnect();
        cout << "  [DB] Connection destroyed" << endl;
    }

    void connect() {
        connected = true;
        cout << "  [DB] Connected" << endl;
    }

    void disconnect() {
        connected = false;
        cout << "  [DB] Disconnected" << endl;
    }

    string execute(const string& query) {
        if (!connected) throw runtime_error("Not connected!");
        queryCount++;
        cout << "  [DB] Executing: " << query << " (query #" << queryCount << ")" << endl;
        return "result";
    }

    void reset() {
        // Reset to clean state for reuse
        queryCount = 0;
        // Don't disconnect — that's the whole point of pooling!
    }

    bool isConnected() const { return connected; }
    bool isHealthy() const { return connected; }  // could do a ping
};

// --- Connection Pool using ObjectPool ---
class ConnectionPool {
    ObjectPool<DatabaseConnection> pool;

public:
    ConnectionPool(const string& connStr, int maxConnections)
        : pool(maxConnections,
               // Factory: create and connect
               [connStr]() {
                   auto conn = make_unique<DatabaseConnection>(connStr);
                   conn->connect();
                   return conn;
               },
               // Reset function
               [](DatabaseConnection& conn) {
                   conn.reset();
               })
    {}

    // RAII wrapper for automatic release
    class ConnectionGuard {
        ConnectionPool& pool;
        unique_ptr<DatabaseConnection> conn;

    public:
        ConnectionGuard(ConnectionPool& p, unique_ptr<DatabaseConnection> c)
            : pool(p), conn(std::move(c)) {}

        ~ConnectionGuard() {
            if (conn) pool.release(std::move(conn));
        }

        // Prevent copying
        ConnectionGuard(const ConnectionGuard&) = delete;
        ConnectionGuard& operator=(const ConnectionGuard&) = delete;

        // Allow moving
        ConnectionGuard(ConnectionGuard&& other) noexcept
            : pool(other.pool), conn(std::move(other.conn)) {}

        DatabaseConnection* operator->() { return conn.get(); }
        DatabaseConnection& operator*() { return *conn; }
    };

    ConnectionGuard getConnection() {
        auto conn = pool.acquire();
        return ConnectionGuard(*this, std::move(conn));
    }

    void release(unique_ptr<DatabaseConnection> conn) {
        pool.release(std::move(conn));
    }

    int getAvailable() const { return pool.getAvailableCount(); }
};

// Usage — connections are automatically returned to pool
void handleRequest(ConnectionPool& pool, const string& query) {
    auto conn = pool.getConnection();  // acquire from pool
    conn->execute(query);
    // ConnectionGuard destructor automatically returns connection to pool
}

int main() {
    ConnectionPool pool("host=localhost dbname=myapp", 5);

    // Simulate concurrent requests
    cout << "=== Handling requests ===" << endl;
    handleRequest(pool, "SELECT * FROM users");
    handleRequest(pool, "INSERT INTO logs VALUES(...)");
    handleRequest(pool, "UPDATE users SET active=true");
    // Connections are reused, not recreated each time!

    cout << "Available connections: " << pool.getAvailable() << endl;
}
```

---

### Thread Pool Concept

A thread pool is a classic application of the Object Pool pattern — threads are expensive to create, so we maintain a pool of worker threads.

```cpp
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <vector>
#include <future>

class ThreadPool {
    vector<thread> workers;
    queue<function<void()>> tasks;
    mutex queueMutex;
    condition_variable condition;
    bool stop = false;

public:
    ThreadPool(int numThreads) {
        for (int i = 0; i < numThreads; i++) {
            workers.emplace_back([this]() {
                while (true) {
                    function<void()> task;
                    {
                        unique_lock<mutex> lock(queueMutex);
                        condition.wait(lock, [this]() {
                            return stop || !tasks.empty();
                        });

                        if (stop && tasks.empty()) return;

                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();  // execute the task
                }
            });
        }
        cout << "ThreadPool created with " << numThreads << " threads" << endl;
    }

    // Submit a task and get a future for the result
    template <typename F, typename... Args>
    auto submit(F&& f, Args&&... args) -> future<decltype(f(args...))> {
        using ReturnType = decltype(f(args...));

        auto task = make_shared<packaged_task<ReturnType()>>(
            bind(forward<F>(f), forward<Args>(args)...)
        );

        future<ReturnType> result = task->get_future();

        {
            lock_guard<mutex> lock(queueMutex);
            if (stop) throw runtime_error("ThreadPool is stopped");
            tasks.emplace([task]() { (*task)(); });
        }

        condition.notify_one();
        return result;
    }

    // Get pool statistics
    size_t pendingTasks() const {
        lock_guard<mutex> lock(const_cast<mutex&>(queueMutex));
        return tasks.size();
    }

    size_t workerCount() const { return workers.size(); }

    ~ThreadPool() {
        {
            lock_guard<mutex> lock(queueMutex);
            stop = true;
        }
        condition.notify_all();
        for (auto& worker : workers) {
            if (worker.joinable()) worker.join();
        }
        cout << "ThreadPool destroyed" << endl;
    }
};

// Usage
ThreadPool pool(4);  // 4 worker threads

// Submit tasks
auto future1 = pool.submit([]() {
    this_thread::sleep_for(chrono::milliseconds(100));
    return 42;
});

auto future2 = pool.submit([](int a, int b) {
    return a + b;
}, 10, 20);

cout << "Result 1: " << future1.get() << endl;  // 42
cout << "Result 2: " << future2.get() << endl;  // 30
```

---

### Generic Object Pool with Health Checks

A production-ready pool that validates objects before handing them out and handles broken objects.

```cpp
template <typename T>
class RobustObjectPool {
public:
    struct PoolConfig {
        int minSize = 2;           // minimum objects to keep ready
        int maxSize = 10;          // maximum objects in pool
        chrono::seconds maxIdleTime = chrono::seconds(300);  // evict after 5 min idle
        chrono::seconds healthCheckInterval = chrono::seconds(30);
        int maxUsageCount = 1000;  // recreate after N uses
    };

private:
    struct PooledObject {
        unique_ptr<T> object;
        chrono::steady_clock::time_point lastUsed;
        int usageCount = 0;
    };

    queue<PooledObject> available;
    mutex poolMutex;
    condition_variable cv;
    int activeCount = 0;
    PoolConfig config;
    function<unique_ptr<T>()> factory;
    function<bool(const T&)> healthCheck;
    function<void(T&)> resetFunc;
    bool shutdown = false;

public:
    RobustObjectPool(PoolConfig cfg,
                     function<unique_ptr<T>()> createFunc,
                     function<bool(const T&)> validate,
                     function<void(T&)> reset)
        : config(cfg), factory(createFunc), healthCheck(validate), resetFunc(reset) {
        // Pre-populate with minimum objects
        for (int i = 0; i < config.minSize; i++) {
            PooledObject po;
            po.object = factory();
            po.lastUsed = chrono::steady_clock::now();
            available.push(std::move(po));
        }
    }

    unique_ptr<T> acquire() {
        unique_lock<mutex> lock(poolMutex);

        while (!available.empty()) {
            auto po = std::move(available.front());
            available.pop();

            // Validate object health
            if (healthCheck && !healthCheck(*po.object)) {
                cout << "[Pool] Discarding unhealthy object" << endl;
                continue;  // discard and try next
            }

            // Check if object has been used too many times
            if (po.usageCount >= config.maxUsageCount) {
                cout << "[Pool] Discarding overused object" << endl;
                continue;  // discard and create fresh
            }

            activeCount++;
            po.usageCount++;
            return std::move(po.object);
        }

        // No valid objects available — create new if under max
        int totalSize = activeCount + (int)available.size();
        if (totalSize < config.maxSize) {
            activeCount++;
            lock.unlock();
            return factory();
        }

        // Wait for return
        cv.wait(lock, [this]() { return !available.empty() || shutdown; });
        if (shutdown) throw runtime_error("Pool is shutting down");

        auto po = std::move(available.front());
        available.pop();
        activeCount++;
        po.usageCount++;
        return std::move(po.object);
    }

    void release(unique_ptr<T> obj) {
        if (!obj) return;

        lock_guard<mutex> lock(poolMutex);
        activeCount--;

        // Reset object state
        if (resetFunc) resetFunc(*obj);

        // Return to pool
        PooledObject po;
        po.object = std::move(obj);
        po.lastUsed = chrono::steady_clock::now();
        available.push(std::move(po));

        cv.notify_one();
    }

    void close() {
        lock_guard<mutex> lock(poolMutex);
        shutdown = true;
        while (!available.empty()) available.pop();
        cv.notify_all();
    }

    // Statistics
    int getActiveCount() const { return activeCount; }
    int getIdleCount() const {
        lock_guard<mutex> lock(const_cast<mutex&>(poolMutex));
        return available.size();
    }
    int getTotalCount() const { return activeCount + getIdleCount(); }
};
```

---

### When to Use Object Pool

| Scenario | Why Object Pool Helps |
|----------|----------------------|
| Object creation is expensive (>1ms) | Amortize creation cost across many uses |
| Objects are needed frequently | Avoid repeated create/destroy cycles |
| Limited resources (connections, threads) | Control maximum resource usage |
| Objects can be reused after reset | Clean state is cheaper than new creation |
| High-throughput systems | Reduce GC pressure and allocation overhead |

**When NOT to use Object Pool:**
- Object creation is cheap (simple POD objects)
- Objects hold state that's hard to reset cleanly
- Pool management overhead exceeds creation cost
- Objects are rarely reused (long-lived, unique state)

**Real-world examples:**
- Database connection pools (HikariCP, c3p0)
- Thread pools (Java ExecutorService, C++ `std::async`)
- HTTP connection pools (keep-alive connections)
- Game object pools (bullets, particles, enemies)
- Memory pools (pre-allocated buffers)
- Socket pools (network connections)

---

### Object Pool vs Other Patterns

| Pattern | Purpose | Relationship to Pool |
|---------|---------|---------------------|
| Singleton | One instance globally | Pool itself is often a Singleton |
| Factory | Creates new objects | Pool uses factory to create initial objects |
| Flyweight | Shares immutable state | Pool shares mutable objects (different!) |
| Prototype | Clones objects | Pool could clone a prototype to fill the pool |

---

## Summary & Key Takeaways

| Pattern | Intent | Key Mechanism | When to Use |
|---------|--------|---------------|-------------|
| **Singleton** | Exactly one instance | Private constructor + static access | Shared resources, global state |
| **Factory Method** | Defer creation to subclasses | Virtual creation method | Unknown types, extensibility |
| **Abstract Factory** | Create families of objects | Factory interface with multiple create methods | Consistent product families |
| **Builder** | Step-by-step construction | Separate builder class with fluent API | Complex objects, many parameters |
| **Prototype** | Clone existing objects | Virtual `clone()` method | Expensive creation, unknown types |
| **Object Pool** | Reuse expensive objects | Pool manages acquire/release lifecycle | Expensive creation, high throughput |

---

## Relationships Between Creational Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│                    CREATIONAL PATTERNS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Singleton ──── often used with ────→ Factory, Pool             │
│                                                                 │
│  Factory Method ──── evolves into ──→ Abstract Factory          │
│                                                                 │
│  Abstract Factory ── implemented with → Factory Method          │
│                  └── or with ────────→ Prototype                │
│                                                                 │
│  Builder ──── can use ──────────────→ Factory (for parts)       │
│          └── produces ──────────────→ Composite (complex obj)   │
│                                                                 │
│  Prototype ──── alternative to ─────→ Factory Method            │
│            └── used to fill ────────→ Object Pool               │
│                                                                 │
│  Object Pool ── uses ───────────────→ Factory (to create)       │
│             └── often is ───────────→ Singleton                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Evolution path:**
1. Start with **Simple Factory** (static method)
2. When you need extensibility → **Factory Method** (virtual method)
3. When you need product families → **Abstract Factory**
4. When construction is complex → **Builder**
5. When creation is expensive → **Prototype** or **Object Pool**
6. When you need exactly one → **Singleton** (use sparingly!)

---

## Interview Tips for Module 4

1. **Singleton thread safety** — Know Meyer's Singleton (C++11 local static), DCLP, and `call_once`. Explain why naive implementation fails with threads. Mention that Meyer's Singleton is the preferred modern approach.

2. **Singleton criticism** — Be ready to explain why Singleton is controversial: hidden dependencies, hard to test, global state. Know the alternative (dependency injection). Interviewers love candidates who understand trade-offs.

3. **Factory Method vs Abstract Factory** — Factory Method creates ONE product (single virtual method). Abstract Factory creates a FAMILY of products (multiple creation methods). Draw the UML difference.

4. **Simple Factory vs Factory Method** — Simple Factory is NOT a GoF pattern. It's a static method with if/else. Factory Method uses inheritance and polymorphism for extensibility (OCP compliant).

5. **Builder vs Constructor** — Builder shines when: many optional parameters, need validation at build time, want immutable products, construction is multi-step. Know the fluent interface pattern.

6. **Builder with Director** — Director encapsulates construction recipes. Same director + different builders = different representations. Same builder + different directors = different construction orders.

7. **Prototype deep vs shallow copy** — Always explain the difference. In C++, implement `clone()` using the copy constructor. Ensure deep copy of owned resources (unique_ptr members). Shared resources (shared_ptr) can be shallow-copied.

8. **Object Pool lifecycle** — Explain acquire/release/reset cycle. Know RAII guards for automatic release. Discuss health checks, max idle time, and pool sizing. Mention real examples (connection pools, thread pools).

9. **When NOT to use each pattern:**
   - Singleton: when testability matters, when you might need multiple instances later
   - Factory: when creation is simple and types are known
   - Builder: when object has few, required parameters
   - Prototype: when objects are cheap to create
   - Pool: when creation is cheap or objects can't be cleanly reset

10. **Combining patterns** — Abstract Factory often uses Factory Methods internally. Object Pool is often a Singleton. Builder can use Factory for creating parts. Show you understand how patterns compose.

11. **Open/Closed Principle** — Factory Method and Abstract Factory are the primary patterns for achieving OCP in object creation. Explain how adding a new product type requires only a new class, not modification of existing code.

12. **Memory management in C++** — Use `unique_ptr` for factory return types (clear ownership transfer). Use `shared_ptr` in pools when multiple references are needed. Always think about who owns the created object.
