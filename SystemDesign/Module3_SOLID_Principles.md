# Module 3: SOLID Principles

> SOLID is an acronym for five design principles that make software designs more understandable, flexible, and maintainable. These principles were introduced by Robert C. Martin (Uncle Bob) and form the foundation of good object-oriented design. Each principle addresses a specific aspect of class design and dependency management, and together they guide developers toward code that is easy to extend, test, and refactor.

---

## 3.1 Single Responsibility Principle (SRP)

> "A class should have one, and only one, reason to change." — Robert C. Martin

The Single Responsibility Principle states that every class should have exactly one responsibility — one axis of change. If a class has more than one reason to change, it has more than one responsibility, and those responsibilities are coupled. A change to one responsibility may impair or inhibit the class's ability to meet the others.

---

### One Class, One Reason to Change

**What is a "reason to change"?**

A "reason to change" corresponds to a **stakeholder** or **actor** who might request a change. If two different actors can request changes to the same class, that class has two responsibilities.

```cpp
// BAD: This class has THREE reasons to change:
// 1. Business logic changes (how salary is calculated)
// 2. Persistence changes (how employee is saved to DB)
// 3. Presentation changes (how employee is displayed/reported)

class Employee {
    string name;
    string department;
    double baseSalary;
    double bonus;

public:
    Employee(const string& n, const string& dept, double salary, double bonus)
        : name(n), department(dept), baseSalary(salary), bonus(bonus) {}

    // Responsibility 1: Business logic (HR department cares about this)
    double calculatePay() const {
        return baseSalary + bonus;
    }

    double calculateTax() const {
        double pay = calculatePay();
        if (pay > 100000) return pay * 0.30;
        if (pay > 50000) return pay * 0.20;
        return pay * 0.10;
    }

    // Responsibility 2: Persistence (DBA/Infrastructure team cares about this)
    void saveToDatabase() {
        // SQL query to save employee
        string sql = "INSERT INTO employees (name, dept, salary, bonus) VALUES ('"
                     + name + "', '" + department + "', "
                     + to_string(baseSalary) + ", " + to_string(bonus) + ")";
        // execute(sql);
    }

    void loadFromDatabase(int id) {
        // SQL query to load employee
        // ... parse results and set fields ...
    }

    // Responsibility 3: Presentation (UI/Report team cares about this)
    string generateReport() const {
        return "Employee Report\n"
               "Name: " + name + "\n"
               "Department: " + department + "\n"
               "Salary: $" + to_string(calculatePay()) + "\n"
               "Tax: $" + to_string(calculateTax()) + "\n";
    }

    void printToConsole() const {
        cout << generateReport();
    }

    string toJSON() const {
        return "{\"name\":\"" + name + "\",\"salary\":" + to_string(calculatePay()) + "}";
    }
};
```

**Why is this problematic?**
- If the report format changes, you modify the same class that handles business logic
- If the database schema changes, you risk breaking salary calculations
- If tax rules change, you might accidentally affect the persistence layer
- Testing one responsibility requires setting up infrastructure for all three
- Multiple developers working on different features will create merge conflicts

---

**GOOD: Each class has exactly one responsibility:**

```cpp
// Responsibility 1: Business logic — salary and tax calculations
class Employee {
    string name;
    string department;
    double baseSalary;
    double bonus;

public:
    Employee(const string& n, const string& dept, double salary, double bonus)
        : name(n), department(dept), baseSalary(salary), bonus(bonus) {}

    string getName() const { return name; }
    string getDepartment() const { return department; }
    double getBaseSalary() const { return baseSalary; }
    double getBonus() const { return bonus; }

    double calculatePay() const {
        return baseSalary + bonus;
    }

    double calculateTax() const {
        double pay = calculatePay();
        if (pay > 100000) return pay * 0.30;
        if (pay > 50000) return pay * 0.20;
        return pay * 0.10;
    }
};

// Responsibility 2: Persistence — saving and loading from database
class EmployeeRepository {
    // Database connection, ORM, etc.
public:
    void save(const Employee& emp) {
        // Parameterized query for safety
        // PreparedStatement stmt("INSERT INTO employees (name, dept, salary, bonus) VALUES (?, ?, ?, ?)");
        // stmt.bind(1, emp.getName());
        // stmt.bind(2, emp.getDepartment());
        // stmt.bind(3, emp.getBaseSalary());
        // stmt.bind(4, emp.getBonus());
        // stmt.execute();
        cout << "Saved " << emp.getName() << " to database" << endl;
    }

    Employee findById(int id) {
        // Query database and construct Employee
        // ...
        return Employee("loaded", "dept", 0, 0);
    }

    vector<Employee> findByDepartment(const string& dept) {
        // ...
        return {};
    }
};

// Responsibility 3: Presentation — formatting and display
class EmployeeReportGenerator {
public:
    string generateTextReport(const Employee& emp) const {
        return "Employee Report\n"
               "Name: " + emp.getName() + "\n"
               "Department: " + emp.getDepartment() + "\n"
               "Salary: $" + to_string(emp.calculatePay()) + "\n"
               "Tax: $" + to_string(emp.calculateTax()) + "\n";
    }

    string generateJSON(const Employee& emp) const {
        return "{\"name\":\"" + emp.getName() + "\","
               "\"department\":\"" + emp.getDepartment() + "\","
               "\"salary\":" + to_string(emp.calculatePay()) + ","
               "\"tax\":" + to_string(emp.calculateTax()) + "}";
    }

    string generateCSV(const Employee& emp) const {
        return emp.getName() + "," + emp.getDepartment() + ","
               + to_string(emp.calculatePay()) + "," + to_string(emp.calculateTax());
    }
};

// Usage — each class is independent and focused
Employee emp("Alice", "Engineering", 85000, 15000);

EmployeeRepository repo;
repo.save(emp);

EmployeeReportGenerator reporter;
cout << reporter.generateTextReport(emp);
cout << reporter.generateJSON(emp);
```

---

### Identifying Responsibilities

**Techniques to identify if a class has too many responsibilities:**

1. **The "and" test:** Describe what the class does. If you use "and" or "or", it likely has multiple responsibilities.
   - "This class calculates pay AND saves to database AND generates reports" → 3 responsibilities

2. **The "who cares" test:** Identify which stakeholders/actors would request changes.
   - HR wants to change tax calculation → business logic
   - DBA wants to change table schema → persistence
   - UI team wants a new report format → presentation

3. **The "reason to change" test:** List all possible reasons the class might need modification.

4. **The cohesion test:** Do all methods use all (or most) of the class's data? If some methods only use a subset of fields, those methods might belong to a different class.

```cpp
// Identifying responsibilities through cohesion analysis

class UserManager {
    // Group A: Authentication data
    string username;
    string passwordHash;
    string salt;
    int failedAttempts;
    time_t lockoutUntil;

    // Group B: Profile data
    string displayName;
    string email;
    string avatarUrl;
    string bio;

    // Group C: Notification preferences
    bool emailNotifications;
    bool pushNotifications;
    vector<string> subscribedTopics;

public:
    // Methods using Group A (Authentication)
    bool authenticate(const string& password) { /* uses passwordHash, salt, failedAttempts */ }
    void resetPassword(const string& newPassword) { /* uses passwordHash, salt */ }
    bool isLockedOut() const { /* uses failedAttempts, lockoutUntil */ }

    // Methods using Group B (Profile)
    void updateProfile(const string& name, const string& bio) { /* uses displayName, bio */ }
    void changeEmail(const string& newEmail) { /* uses email */ }
    void setAvatar(const string& url) { /* uses avatarUrl */ }

    // Methods using Group C (Notifications)
    void subscribe(const string& topic) { /* uses subscribedTopics */ }
    void setEmailNotifications(bool enabled) { /* uses emailNotifications */ }
    vector<string> getSubscriptions() const { /* uses subscribedTopics */ }
};

// REFACTORED: Three cohesive classes
class UserAuthentication {
    string username;
    string passwordHash;
    string salt;
    int failedAttempts = 0;
    time_t lockoutUntil = 0;

public:
    UserAuthentication(const string& user, const string& passHash, const string& s)
        : username(user), passwordHash(passHash), salt(s) {}

    bool authenticate(const string& password);
    void resetPassword(const string& newPassword);
    bool isLockedOut() const;
    void recordFailedAttempt();
    void unlock();
};

class UserProfile {
    string userId;
    string displayName;
    string email;
    string avatarUrl;
    string bio;

public:
    UserProfile(const string& id, const string& name, const string& email)
        : userId(id), displayName(name), email(email) {}

    void updateDisplayName(const string& name);
    void changeEmail(const string& newEmail);
    void setAvatar(const string& url);
    void updateBio(const string& newBio);

    const string& getDisplayName() const { return displayName; }
    const string& getEmail() const { return email; }
};

class NotificationPreferences {
    string userId;
    bool emailEnabled = true;
    bool pushEnabled = true;
    vector<string> subscribedTopics;

public:
    NotificationPreferences(const string& id) : userId(id) {}

    void enableEmail(bool enabled) { emailEnabled = enabled; }
    void enablePush(bool enabled) { pushEnabled = enabled; }
    void subscribe(const string& topic);
    void unsubscribe(const string& topic);
    bool shouldNotifyViaEmail() const { return emailEnabled; }
    bool shouldNotifyViaPush() const { return pushEnabled; }
};
```

---

### Refactoring God Classes

A **God Class** (or Blob) is a class that knows too much or does too much. It violates SRP by accumulating responsibilities over time. Refactoring a God Class involves extracting cohesive groups of data and behavior into separate classes.

**Step-by-step refactoring process:**

```cpp
// BEFORE: God Class — an online store "does everything" class
class OnlineStore {
    // Product data
    vector<Product> products;
    map<int, int> inventory;  // productId -> quantity

    // User data
    vector<User> users;
    map<int, vector<int>> userOrders;  // userId -> orderIds

    // Order data
    vector<Order> orders;
    int nextOrderId = 1;

    // Payment data
    string stripeApiKey;
    string paypalClientId;

    // Email data
    string smtpServer;
    string fromAddress;

public:
    // Product management
    void addProduct(const Product& p) { products.push_back(p); }
    void updateInventory(int productId, int qty) { inventory[productId] = qty; }
    bool isInStock(int productId) const { return inventory.at(productId) > 0; }

    // User management
    void registerUser(const User& u) { users.push_back(u); }
    User* findUser(int userId) { /* ... */ }
    bool authenticateUser(const string& email, const string& password) { /* ... */ }

    // Order processing
    int createOrder(int userId, const vector<pair<int,int>>& items) { /* ... */ }
    void cancelOrder(int orderId) { /* ... */ }
    double calculateOrderTotal(int orderId) { /* ... */ }

    // Payment processing
    bool chargeStripe(int orderId, const string& cardToken) { /* ... */ }
    bool chargePaypal(int orderId, const string& paypalEmail) { /* ... */ }
    void processRefund(int orderId) { /* ... */ }

    // Email notifications
    void sendOrderConfirmation(int orderId) { /* ... */ }
    void sendShippingNotification(int orderId, const string& trackingNum) { /* ... */ }
    void sendPasswordReset(int userId) { /* ... */ }

    // Reporting
    double getTotalRevenue() const { /* ... */ }
    vector<Product> getTopSellingProducts(int n) const { /* ... */ }
    map<string, int> getSalesByCategory() const { /* ... */ }
};
```

**AFTER: Refactored into focused classes:**

```cpp
// Each class has ONE responsibility and ONE reason to change

class ProductCatalog {
    vector<Product> products;
public:
    void addProduct(const Product& p);
    void removeProduct(int productId);
    Product* findProduct(int productId);
    vector<Product> search(const string& query) const;
};

class InventoryManager {
    map<int, int> stock;  // productId -> quantity
public:
    void setStock(int productId, int quantity);
    bool isAvailable(int productId, int quantity = 1) const;
    void reserve(int productId, int quantity);
    void release(int productId, int quantity);
    int getStock(int productId) const;
};

class UserService {
    vector<User> users;
public:
    void registerUser(const User& u);
    User* findById(int userId);
    User* findByEmail(const string& email);
    bool authenticate(const string& email, const string& password);
    void updateProfile(int userId, const UserProfile& profile);
};

class OrderService {
    vector<Order> orders;
    int nextOrderId = 1;
    InventoryManager& inventory;  // dependency injection

public:
    OrderService(InventoryManager& inv) : inventory(inv) {}

    int createOrder(int userId, const vector<OrderItem>& items);
    void cancelOrder(int orderId);
    Order* findOrder(int orderId);
    double calculateTotal(int orderId) const;
    vector<Order> getOrdersByUser(int userId) const;
};

class PaymentService {
    string stripeApiKey;
    string paypalClientId;
public:
    PaymentService(const string& stripeKey, const string& paypalId)
        : stripeApiKey(stripeKey), paypalClientId(paypalId) {}

    bool chargeCard(double amount, const string& cardToken);
    bool chargePaypal(double amount, const string& paypalEmail);
    bool refund(const string& transactionId, double amount);
};

class NotificationService {
    string smtpServer;
    string fromAddress;
public:
    NotificationService(const string& smtp, const string& from)
        : smtpServer(smtp), fromAddress(from) {}

    void sendOrderConfirmation(const Order& order, const string& email);
    void sendShippingUpdate(const Order& order, const string& trackingNum);
    void sendPasswordReset(const string& email, const string& resetToken);
};

class SalesReportService {
    const vector<Order>& orders;
    const ProductCatalog& catalog;
public:
    SalesReportService(const vector<Order>& ord, const ProductCatalog& cat)
        : orders(ord), catalog(cat) {}

    double getTotalRevenue() const;
    vector<Product> getTopSelling(int n) const;
    map<string, double> getRevenueByCategory() const;
};
```

**Benefits of the refactoring:**
- Each class can be tested independently
- Changes to payment processing don't risk breaking order logic
- Different teams can work on different classes without conflicts
- New notification channels (SMS, Push) only affect NotificationService
- Database changes only affect the relevant repository/service

---

### C++ Examples with Before/After

**Example 1: Logger with too many responsibilities**

```cpp
// BEFORE: Logger handles formatting, filtering, AND output
class Logger {
    ofstream file;
    string filename;
    int minLevel;  // 0=DEBUG, 1=INFO, 2=WARN, 3=ERROR

public:
    Logger(const string& fname, int level) : filename(fname), minLevel(level) {
        file.open(fname, ios::app);
    }

    void log(int level, const string& message) {
        if (level < minLevel) return;  // filtering

        // formatting
        string levelStr;
        switch (level) {
            case 0: levelStr = "DEBUG"; break;
            case 1: levelStr = "INFO"; break;
            case 2: levelStr = "WARN"; break;
            case 3: levelStr = "ERROR"; break;
        }

        time_t now = time(nullptr);
        string timestamp = ctime(&now);
        timestamp.pop_back();  // remove newline

        string formatted = "[" + timestamp + "] [" + levelStr + "] " + message;

        // output to file
        file << formatted << endl;

        // output to console (hardcoded!)
        if (level >= 2) {
            cerr << formatted << endl;
        }
    }

    ~Logger() { file.close(); }
};
```

```cpp
// AFTER: Separated into focused components

// Responsibility: Format log messages
class LogFormatter {
public:
    virtual string format(int level, const string& message) const = 0;
    virtual ~LogFormatter() = default;
};

class StandardFormatter : public LogFormatter {
public:
    string format(int level, const string& message) const override {
        static const char* levels[] = {"DEBUG", "INFO", "WARN", "ERROR"};
        time_t now = time(nullptr);
        string timestamp = ctime(&now);
        timestamp.pop_back();
        return "[" + timestamp + "] [" + string(levels[level]) + "] " + message;
    }
};

class JSONFormatter : public LogFormatter {
public:
    string format(int level, const string& message) const override {
        return "{\"level\":" + to_string(level) + ",\"message\":\"" + message + "\"}";
    }
};

// Responsibility: Filter log messages by level
class LogFilter {
    int minLevel;
public:
    LogFilter(int level) : minLevel(level) {}
    bool shouldLog(int level) const { return level >= minLevel; }
    void setMinLevel(int level) { minLevel = level; }
};

// Responsibility: Write log messages to a destination
class LogOutput {
public:
    virtual void write(const string& formattedMessage) = 0;
    virtual ~LogOutput() = default;
};

class FileOutput : public LogOutput {
    ofstream file;
public:
    FileOutput(const string& filename) { file.open(filename, ios::app); }
    void write(const string& msg) override { file << msg << endl; }
    ~FileOutput() { file.close(); }
};

class ConsoleOutput : public LogOutput {
public:
    void write(const string& msg) override { cerr << msg << endl; }
};

class NetworkOutput : public LogOutput {
    string serverUrl;
public:
    NetworkOutput(const string& url) : serverUrl(url) {}
    void write(const string& msg) override {
        // Send to log aggregation service
        cout << "[NETWORK -> " << serverUrl << "] " << msg << endl;
    }
};

// Orchestrator: Composes the above components
class Logger {
    unique_ptr<LogFormatter> formatter;
    LogFilter filter;
    vector<unique_ptr<LogOutput>> outputs;

public:
    Logger(unique_ptr<LogFormatter> fmt, int minLevel)
        : formatter(std::move(fmt)), filter(minLevel) {}

    void addOutput(unique_ptr<LogOutput> output) {
        outputs.push_back(std::move(output));
    }

    void log(int level, const string& message) {
        if (!filter.shouldLog(level)) return;

        string formatted = formatter->format(level, message);

        for (auto& output : outputs) {
            output->write(formatted);
        }
    }
};

// Usage — flexible composition
auto logger = Logger(make_unique<StandardFormatter>(), 1);  // INFO and above
logger.addOutput(make_unique<FileOutput>("app.log"));
logger.addOutput(make_unique<ConsoleOutput>());
logger.addOutput(make_unique<NetworkOutput>("https://logs.example.com"));

logger.log(1, "Application started");
logger.log(3, "Database connection failed");
```

**Example 2: Invoice class refactoring**

```cpp
// BEFORE: Invoice handles calculation, persistence, AND printing
class Invoice {
    vector<LineItem> items;
    double taxRate;
    string customerName;
    string customerEmail;

public:
    // Calculation (business logic)
    double getSubtotal() const {
        double sum = 0;
        for (const auto& item : items) sum += item.price * item.quantity;
        return sum;
    }
    double getTax() const { return getSubtotal() * taxRate; }
    double getTotal() const { return getSubtotal() + getTax(); }

    // Persistence (infrastructure concern)
    void saveToFile(const string& filename) {
        ofstream f(filename);
        f << "Customer: " << customerName << endl;
        for (const auto& item : items)
            f << item.name << "," << item.price << "," << item.quantity << endl;
        f << "Total: " << getTotal() << endl;
    }

    // Printing (presentation concern)
    void print() const {
        cout << "=== INVOICE ===" << endl;
        cout << "Customer: " << customerName << endl;
        for (const auto& item : items)
            cout << "  " << item.name << " x" << item.quantity
                 << " @ $" << item.price << endl;
        cout << "Subtotal: $" << getSubtotal() << endl;
        cout << "Tax: $" << getTax() << endl;
        cout << "TOTAL: $" << getTotal() << endl;
    }

    // Email sending (notification concern)
    void emailToCustomer() {
        // SMTP logic here...
    }
};
```

```cpp
// AFTER: Clean separation

class Invoice {
    vector<LineItem> items;
    double taxRate;
    string customerId;

public:
    Invoice(const string& custId, double tax) : customerId(custId), taxRate(tax) {}

    void addItem(const LineItem& item) { items.push_back(item); }

    double getSubtotal() const {
        double sum = 0;
        for (const auto& item : items) sum += item.price * item.quantity;
        return sum;
    }
    double getTax() const { return getSubtotal() * taxRate; }
    double getTotal() const { return getSubtotal() + getTax(); }

    const vector<LineItem>& getItems() const { return items; }
    const string& getCustomerId() const { return customerId; }
    double getTaxRate() const { return taxRate; }
};

class InvoiceRepository {
public:
    virtual void save(const Invoice& invoice) = 0;
    virtual Invoice load(const string& invoiceId) = 0;
    virtual ~InvoiceRepository() = default;
};

class FileInvoiceRepository : public InvoiceRepository {
    string directory;
public:
    FileInvoiceRepository(const string& dir) : directory(dir) {}
    void save(const Invoice& invoice) override { /* write to file */ }
    Invoice load(const string& invoiceId) override { /* read from file */ }
};

class InvoicePrinter {
public:
    virtual void print(const Invoice& invoice) const = 0;
    virtual ~InvoicePrinter() = default;
};

class ConsoleInvoicePrinter : public InvoicePrinter {
public:
    void print(const Invoice& invoice) const override {
        cout << "=== INVOICE ===" << endl;
        for (const auto& item : invoice.getItems())
            cout << "  " << item.name << " x" << item.quantity
                 << " @ $" << item.price << endl;
        cout << "TOTAL: $" << invoice.getTotal() << endl;
    }
};

class PDFInvoicePrinter : public InvoicePrinter {
public:
    void print(const Invoice& invoice) const override {
        // Generate PDF...
    }
};
```

---

### Common Pitfalls and Guidelines

**Over-applying SRP:**
- Don't create a class for every single method — that's fragmentation, not SRP
- A class with 5 methods that all operate on the same data for the same purpose is fine
- SRP is about cohesion at the responsibility level, not at the method level

**Under-applying SRP:**
- If a class has 20+ methods spanning different concerns, it needs splitting
- If changing one feature frequently requires modifying unrelated code in the same class, split it

**Granularity guideline:**
- A responsibility is a "reason to change" tied to a specific actor/stakeholder
- If two things always change together for the same reason, they belong together
- If two things change at different times for different reasons, they should be separate

---


## 3.2 Open/Closed Principle (OCP)

> "Software entities (classes, modules, functions) should be open for extension, but closed for modification." — Bertrand Meyer

The Open/Closed Principle means that once a class is written, tested, and deployed, its source code should not need to be modified to add new behavior. Instead, new behavior should be added by writing new code (new classes, new implementations) that extends the existing system.

**Why?**
- Modifying existing code risks introducing bugs in already-working functionality
- Existing code may already be tested and deployed — changes require re-testing everything
- Multiple developers adding features to the same class creates merge conflicts
- Closed for modification = stable, reliable foundation

---

### Open for Extension, Closed for Modification

**The problem — violating OCP:**

```cpp
// BAD: Every new shape requires modifying this function
class AreaCalculator {
public:
    double calculateArea(const vector<void*>& shapes, const vector<string>& types) {
        double total = 0;
        for (size_t i = 0; i < shapes.size(); i++) {
            if (types[i] == "circle") {
                struct Circle { double radius; };
                Circle* c = static_cast<Circle*>(shapes[i]);
                total += 3.14159 * c->radius * c->radius;
            }
            else if (types[i] == "rectangle") {
                struct Rectangle { double width, height; };
                Rectangle* r = static_cast<Rectangle*>(shapes[i]);
                total += r->width * r->height;
            }
            else if (types[i] == "triangle") {
                struct Triangle { double base, height; };
                Triangle* t = static_cast<Triangle*>(shapes[i]);
                total += 0.5 * t->base * t->height;
            }
            // Adding a new shape (e.g., Pentagon) requires MODIFYING this function!
            // This violates OCP.
        }
        return total;
    }
};
```

**Problems with the above:**
- Adding a new shape means modifying `calculateArea` — violates "closed for modification"
- The function grows unboundedly with each new shape
- Risk of breaking existing shape calculations when adding new ones
- Cannot add shapes from external libraries without modifying this code

**GOOD: Open for extension, closed for modification:**

```cpp
// Abstract base — the stable, closed part
class Shape {
public:
    virtual double area() const = 0;
    virtual string name() const = 0;
    virtual ~Shape() = default;
};

// Extensions — new shapes added WITHOUT modifying existing code
class Circle : public Shape {
    double radius;
public:
    Circle(double r) : radius(r) {}
    double area() const override { return 3.14159265 * radius * radius; }
    string name() const override { return "Circle"; }
};

class Rectangle : public Shape {
    double width, height;
public:
    Rectangle(double w, double h) : width(w), height(h) {}
    double area() const override { return width * height; }
    string name() const override { return "Rectangle"; }
};

class Triangle : public Shape {
    double base, height;
public:
    Triangle(double b, double h) : base(b), height(h) {}
    double area() const override { return 0.5 * base * height; }
    string name() const override { return "Triangle"; }
};

// This class is CLOSED for modification — never needs to change
class AreaCalculator {
public:
    double totalArea(const vector<unique_ptr<Shape>>& shapes) const {
        double total = 0;
        for (const auto& shape : shapes) {
            total += shape->area();
        }
        return total;
    }

    void printReport(const vector<unique_ptr<Shape>>& shapes) const {
        for (const auto& shape : shapes) {
            cout << shape->name() << ": area = " << shape->area() << endl;
        }
        cout << "Total area: " << totalArea(shapes) << endl;
    }
};

// Adding a new shape — NO modification to AreaCalculator or any existing shape!
class Pentagon : public Shape {
    double side;
public:
    Pentagon(double s) : side(s) {}
    double area() const override {
        return 0.25 * sqrt(5 * (5 + 2 * sqrt(5))) * side * side;
    }
    string name() const override { return "Pentagon"; }
};

class Ellipse : public Shape {
    double a, b;  // semi-major and semi-minor axes
public:
    Ellipse(double a, double b) : a(a), b(b) {}
    double area() const override { return 3.14159265 * a * b; }
    string name() const override { return "Ellipse"; }
};

// Usage — AreaCalculator works with ALL shapes, past and future
vector<unique_ptr<Shape>> shapes;
shapes.push_back(make_unique<Circle>(5.0));
shapes.push_back(make_unique<Rectangle>(3.0, 4.0));
shapes.push_back(make_unique<Pentagon>(2.0));
shapes.push_back(make_unique<Ellipse>(3.0, 2.0));

AreaCalculator calc;
calc.printReport(shapes);  // works without any modification to AreaCalculator
```

---

### Using Inheritance and Polymorphism for OCP

The most common way to achieve OCP in C++ is through **abstract base classes** (interfaces) and **virtual functions**. The base class defines the contract (closed), and derived classes provide new implementations (open for extension).

**Example: Payment processing system**

```cpp
// Closed for modification — this interface is stable
class PaymentMethod {
public:
    virtual bool pay(double amount) = 0;
    virtual bool refund(double amount) = 0;
    virtual string getMethodName() const = 0;
    virtual double getTransactionFee(double amount) const = 0;
    virtual ~PaymentMethod() = default;
};

// Existing implementations — already tested and deployed
class CreditCardPayment : public PaymentMethod {
    string cardNumber;
    string expiryDate;
public:
    CreditCardPayment(const string& card, const string& expiry)
        : cardNumber(card), expiryDate(expiry) {}

    bool pay(double amount) override {
        double fee = getTransactionFee(amount);
        cout << "Charging $" << (amount + fee) << " to card ending in "
             << cardNumber.substr(cardNumber.size() - 4) << endl;
        return true;
    }

    bool refund(double amount) override {
        cout << "Refunding $" << amount << " to card" << endl;
        return true;
    }

    string getMethodName() const override { return "Credit Card"; }
    double getTransactionFee(double amount) const override { return amount * 0.029 + 0.30; }
};

class PayPalPayment : public PaymentMethod {
    string email;
public:
    PayPalPayment(const string& e) : email(e) {}

    bool pay(double amount) override {
        cout << "PayPal payment of $" << amount << " from " << email << endl;
        return true;
    }

    bool refund(double amount) override {
        cout << "PayPal refund of $" << amount << " to " << email << endl;
        return true;
    }

    string getMethodName() const override { return "PayPal"; }
    double getTransactionFee(double amount) const override { return amount * 0.034 + 0.30; }
};

// Payment processor — CLOSED for modification
class PaymentProcessor {
public:
    bool processPayment(PaymentMethod& method, double amount) {
        cout << "Processing " << method.getMethodName() << " payment..." << endl;
        double fee = method.getTransactionFee(amount);
        cout << "Transaction fee: $" << fee << endl;

        bool success = method.pay(amount);
        if (success) {
            cout << "Payment successful!" << endl;
        } else {
            cout << "Payment failed!" << endl;
        }
        return success;
    }
};

// EXTENSION: Adding cryptocurrency payment — NO modification to PaymentProcessor!
class CryptoPayment : public PaymentMethod {
    string walletAddress;
    string currency;  // BTC, ETH, etc.
public:
    CryptoPayment(const string& wallet, const string& curr)
        : walletAddress(wallet), currency(curr) {}

    bool pay(double amount) override {
        cout << "Sending " << amount << " USD equivalent in " << currency
             << " to " << walletAddress << endl;
        return true;
    }

    bool refund(double amount) override {
        cout << "Crypto refund of $" << amount << " in " << currency << endl;
        return true;
    }

    string getMethodName() const override { return "Crypto (" + currency + ")"; }
    double getTransactionFee(double amount) const override { return amount * 0.01; }
};

// EXTENSION: Adding bank transfer — NO modification needed anywhere!
class BankTransferPayment : public PaymentMethod {
    string accountNumber;
    string routingNumber;
public:
    BankTransferPayment(const string& acc, const string& routing)
        : accountNumber(acc), routingNumber(routing) {}

    bool pay(double amount) override {
        cout << "Bank transfer of $" << amount << " from account " << accountNumber << endl;
        return true;
    }

    bool refund(double amount) override {
        cout << "Bank refund of $" << amount << " to account " << accountNumber << endl;
        return true;
    }

    string getMethodName() const override { return "Bank Transfer"; }
    double getTransactionFee(double amount) const override { return 0.25; }  // flat fee
};

// Usage — PaymentProcessor handles ALL payment types without modification
PaymentProcessor processor;

CreditCardPayment card("4111111111111111", "12/25");
processor.processPayment(card, 99.99);

CryptoPayment crypto("0xABC123...", "ETH");
processor.processPayment(crypto, 50.00);

BankTransferPayment bank("123456789", "021000021");
processor.processPayment(bank, 1000.00);
```

---

### Strategy Pattern as OCP Example

The **Strategy Pattern** is one of the most natural implementations of OCP. It allows you to define a family of algorithms, encapsulate each one, and make them interchangeable — all without modifying the context class.

```cpp
// Strategy interface — closed for modification
class CompressionStrategy {
public:
    virtual vector<uint8_t> compress(const vector<uint8_t>& data) = 0;
    virtual vector<uint8_t> decompress(const vector<uint8_t>& data) = 0;
    virtual string getName() const = 0;
    virtual double getCompressionRatio() const = 0;
    virtual ~CompressionStrategy() = default;
};

// Concrete strategies — each is an extension
class ZipCompression : public CompressionStrategy {
public:
    vector<uint8_t> compress(const vector<uint8_t>& data) override {
        cout << "Compressing with ZIP algorithm..." << endl;
        // ... ZIP compression logic ...
        return data;  // simplified
    }

    vector<uint8_t> decompress(const vector<uint8_t>& data) override {
        cout << "Decompressing ZIP..." << endl;
        return data;
    }

    string getName() const override { return "ZIP"; }
    double getCompressionRatio() const override { return 0.6; }
};

class GzipCompression : public CompressionStrategy {
public:
    vector<uint8_t> compress(const vector<uint8_t>& data) override {
        cout << "Compressing with GZIP algorithm..." << endl;
        return data;
    }

    vector<uint8_t> decompress(const vector<uint8_t>& data) override {
        cout << "Decompressing GZIP..." << endl;
        return data;
    }

    string getName() const override { return "GZIP"; }
    double getCompressionRatio() const override { return 0.55; }
};

class LZ4Compression : public CompressionStrategy {
public:
    vector<uint8_t> compress(const vector<uint8_t>& data) override {
        cout << "Compressing with LZ4 (fast)..." << endl;
        return data;
    }

    vector<uint8_t> decompress(const vector<uint8_t>& data) override {
        cout << "Decompressing LZ4..." << endl;
        return data;
    }

    string getName() const override { return "LZ4"; }
    double getCompressionRatio() const override { return 0.75; }  // less compression, more speed
};

// Context class — CLOSED for modification
class FileArchiver {
    unique_ptr<CompressionStrategy> strategy;

public:
    FileArchiver(unique_ptr<CompressionStrategy> s) : strategy(std::move(s)) {}

    void setStrategy(unique_ptr<CompressionStrategy> s) {
        strategy = std::move(s);
    }

    void archiveFile(const string& filename) {
        cout << "Archiving " << filename << " using " << strategy->getName() << endl;
        cout << "Expected compression ratio: " << strategy->getCompressionRatio() << endl;

        // Read file data (simplified)
        vector<uint8_t> data = {/* file contents */};

        // Compress using current strategy
        auto compressed = strategy->compress(data);

        cout << "Archive complete." << endl;
    }

    void extractFile(const string& archiveName) {
        cout << "Extracting " << archiveName << " using " << strategy->getName() << endl;
        vector<uint8_t> data = {/* archive contents */};
        auto decompressed = strategy->decompress(data);
        cout << "Extraction complete." << endl;
    }
};

// Adding a new compression algorithm — ZERO modification to FileArchiver
class BrotliCompression : public CompressionStrategy {
public:
    vector<uint8_t> compress(const vector<uint8_t>& data) override {
        cout << "Compressing with Brotli (web-optimized)..." << endl;
        return data;
    }

    vector<uint8_t> decompress(const vector<uint8_t>& data) override {
        cout << "Decompressing Brotli..." << endl;
        return data;
    }

    string getName() const override { return "Brotli"; }
    double getCompressionRatio() const override { return 0.50; }
};

// Usage
FileArchiver archiver(make_unique<GzipCompression>());
archiver.archiveFile("document.pdf");

// Switch strategy at runtime — no modification to FileArchiver
archiver.setStrategy(make_unique<BrotliCompression>());
archiver.archiveFile("image.png");
```

---

### Template Method as OCP Example

The **Template Method Pattern** defines the skeleton of an algorithm in a base class, letting subclasses override specific steps without changing the algorithm's structure. The base class is closed for modification (the algorithm structure is fixed), but open for extension (individual steps can be customized).

```cpp
// Template Method — the algorithm skeleton is CLOSED for modification
class DataMiner {
public:
    // Template method — defines the algorithm structure (final = cannot be overridden)
    void mine(const string& path) {
        string rawData = extractData(path);
        vector<string> data = parseData(rawData);
        auto analysis = analyzeData(data);
        string report = generateReport(analysis);
        sendReport(report);
    }

protected:
    // Steps that subclasses MUST implement (pure virtual = open for extension)
    virtual string extractData(const string& path) = 0;
    virtual vector<string> parseData(const string& rawData) = 0;

    // Steps with default implementation that CAN be overridden (hook methods)
    virtual map<string, int> analyzeData(const vector<string>& data) {
        // Default: word frequency analysis
        map<string, int> freq;
        for (const auto& word : data) freq[word]++;
        return freq;
    }

    virtual string generateReport(const map<string, int>& analysis) {
        string report = "Analysis Report:\n";
        for (const auto& [key, value] : analysis) {
            report += "  " + key + ": " + to_string(value) + "\n";
        }
        return report;
    }

    virtual void sendReport(const string& report) {
        cout << report;  // default: print to console
    }

public:
    virtual ~DataMiner() = default;
};

// Extension: CSV data mining — only implements what's different
class CSVDataMiner : public DataMiner {
protected:
    string extractData(const string& path) override {
        cout << "Reading CSV file: " << path << endl;
        // Read CSV file contents
        return "name,age,city\nAlice,30,NYC\nBob,25,LA";
    }

    vector<string> parseData(const string& rawData) override {
        vector<string> result;
        // Parse CSV rows
        stringstream ss(rawData);
        string line;
        while (getline(ss, line)) {
            result.push_back(line);
        }
        return result;
    }
};

// Extension: PDF data mining — different extraction and parsing
class PDFDataMiner : public DataMiner {
protected:
    string extractData(const string& path) override {
        cout << "Extracting text from PDF: " << path << endl;
        // Use PDF library to extract text
        return "extracted pdf text content here";
    }

    vector<string> parseData(const string& rawData) override {
        vector<string> result;
        // Split by whitespace for PDF text
        stringstream ss(rawData);
        string word;
        while (ss >> word) {
            result.push_back(word);
        }
        return result;
    }

    // Override hook: send report via email instead of console
    void sendReport(const string& report) override {
        cout << "[EMAIL] Sending PDF analysis report..." << endl;
        cout << report;
    }
};

// Extension: Database data mining
class DatabaseDataMiner : public DataMiner {
protected:
    string extractData(const string& path) override {
        cout << "Querying database: " << path << endl;
        // Execute SQL query
        return "row1|row2|row3";
    }

    vector<string> parseData(const string& rawData) override {
        vector<string> result;
        // Parse pipe-delimited data
        stringstream ss(rawData);
        string item;
        while (getline(ss, item, '|')) {
            result.push_back(item);
        }
        return result;
    }
};

// Usage — the mine() algorithm never changes, but behavior varies
CSVDataMiner csvMiner;
csvMiner.mine("data.csv");

PDFDataMiner pdfMiner;
pdfMiner.mine("report.pdf");

DatabaseDataMiner dbMiner;
dbMiner.mine("SELECT * FROM analytics");
```

---

### OCP with Function Objects and std::function

In modern C++, OCP can also be achieved without inheritance using **function objects**, **lambdas**, and **std::function**:

```cpp
#include <functional>

// Closed for modification — accepts any validation logic
class InputValidator {
    vector<pair<string, function<bool(const string&)>>> rules;

public:
    void addRule(const string& name, function<bool(const string&)> rule) {
        rules.push_back({name, rule});
    }

    bool validate(const string& input) const {
        for (const auto& [name, rule] : rules) {
            if (!rule(input)) {
                cout << "Validation failed: " << name << endl;
                return false;
            }
        }
        return true;
    }
};

// Open for extension — add new rules without modifying InputValidator
InputValidator emailValidator;
emailValidator.addRule("not_empty", [](const string& s) { return !s.empty(); });
emailValidator.addRule("has_at_sign", [](const string& s) { return s.find('@') != string::npos; });
emailValidator.addRule("has_dot", [](const string& s) { return s.find('.') != string::npos; });
emailValidator.addRule("min_length", [](const string& s) { return s.size() >= 5; });

// Later, add more rules without touching InputValidator:
emailValidator.addRule("no_spaces", [](const string& s) {
    return s.find(' ') == string::npos;
});
emailValidator.addRule("valid_domain", [](const string& s) {
    auto atPos = s.find('@');
    if (atPos == string::npos) return false;
    string domain = s.substr(atPos + 1);
    return domain.find('.') != string::npos && domain.size() > 3;
});

cout << emailValidator.validate("user@example.com") << endl;  // 1 (true)
cout << emailValidator.validate("invalid") << endl;            // 0 (false)
```

---

### Key Takeaways for OCP

| Technique | How it achieves OCP | When to use |
|-----------|---|---|
| Inheritance + Virtual Functions | New behavior via new derived classes | When you have a clear type hierarchy |
| Strategy Pattern | Swap algorithms without modifying context | When behavior varies independently of the object |
| Template Method | Override steps without changing algorithm structure | When algorithm skeleton is fixed but steps vary |
| std::function / Lambdas | Inject behavior as callable objects | When you want lightweight, flexible extension |
| Decorator Pattern | Add responsibilities without modifying original | When you need to layer behaviors dynamically |

**When OCP might be overkill:**
- Simple scripts or one-off utilities
- Code that genuinely won't need extension
- Premature abstraction (don't add extension points "just in case")
- Apply OCP where you've seen or expect change, not everywhere

---


## 3.3 Liskov Substitution Principle (LSP)

> "Objects of a superclass should be replaceable with objects of a subclass without affecting the correctness of the program." — Barbara Liskov, 1987

The Liskov Substitution Principle is the most formal of the SOLID principles. It states that if `S` is a subtype of `T`, then objects of type `T` may be replaced with objects of type `S` without altering any of the desirable properties of the program (correctness, task performed, etc.).

In simpler terms: **A derived class must be usable through the base class interface without the caller knowing the difference.**

---

### Subtypes Must Be Substitutable for Base Types

**What does "substitutable" mean?**

If code works correctly with a base class pointer/reference, it must continue to work correctly when that pointer/reference actually points to any derived class object. The derived class must honor the contract established by the base class.

```cpp
// A function that works with the base type
void processShape(const Shape& shape) {
    double area = shape.area();
    assert(area >= 0);  // we expect area to be non-negative
    cout << "Area: " << area << endl;
}

// LSP says: ANY subclass of Shape passed to processShape() must work correctly
// - Circle, Rectangle, Triangle — all should return valid non-negative areas
// - No subclass should throw unexpected exceptions
// - No subclass should violate the contract (e.g., return negative area)
```

**A correct LSP-compliant hierarchy:**

```cpp
class Bird {
public:
    virtual string getName() const = 0;
    virtual string getHabitat() const = 0;
    virtual void eat() = 0;
    virtual ~Bird() = default;
};

class FlyingBird : public Bird {
public:
    virtual void fly() = 0;
    virtual double getAltitude() const = 0;
};

class Sparrow : public FlyingBird {
public:
    string getName() const override { return "Sparrow"; }
    string getHabitat() const override { return "Urban areas"; }
    void eat() override { cout << "Sparrow eating seeds" << endl; }
    void fly() override { cout << "Sparrow flying" << endl; }
    double getAltitude() const override { return 50.0; }
};

class Penguin : public Bird {  // NOT FlyingBird — penguins can't fly!
public:
    string getName() const override { return "Penguin"; }
    string getHabitat() const override { return "Antarctic"; }
    void eat() override { cout << "Penguin eating fish" << endl; }
    void swim() { cout << "Penguin swimming" << endl; }
};

// LSP-compliant usage:
void describeBird(const Bird& bird) {
    cout << bird.getName() << " lives in " << bird.getHabitat() << endl;
    // This works for ALL birds — Sparrow, Penguin, Eagle, etc.
}

void makeFly(FlyingBird& bird) {
    bird.fly();
    cout << "Altitude: " << bird.getAltitude() << "m" << endl;
    // This works for ALL flying birds — Sparrow, Eagle, etc.
    // Penguin is NOT a FlyingBird, so it can't be passed here — correct!
}
```

**LSP violation — the classic bad design:**

```cpp
// BAD: Penguin inherits from a Bird class that promises fly()
class BirdBad {
public:
    virtual void fly() = 0;  // ALL birds must fly? Wrong assumption!
    virtual ~BirdBad() = default;
};

class PenguinBad : public BirdBad {
public:
    void fly() override {
        throw runtime_error("Penguins can't fly!");  // VIOLATES LSP!
        // Any code expecting BirdBad::fly() to work will break
    }
};

void migrateBirds(vector<BirdBad*>& flock) {
    for (auto* bird : flock) {
        bird->fly();  // CRASH if any bird is a Penguin!
    }
}
```

---

### The Rectangle-Square Problem

This is the most famous example of an LSP violation. Mathematically, a square IS a rectangle (a rectangle with equal sides). But in OOP, making `Square` inherit from `Rectangle` violates LSP.

**The violation:**

```cpp
class Rectangle {
protected:
    double width;
    double height;

public:
    Rectangle(double w, double h) : width(w), height(h) {}

    virtual void setWidth(double w) { width = w; }
    virtual void setHeight(double h) { height = h; }

    double getWidth() const { return width; }
    double getHeight() const { return height; }
    double area() const { return width * height; }
};

class Square : public Rectangle {
public:
    Square(double side) : Rectangle(side, side) {}

    // Must keep width == height to maintain square invariant
    void setWidth(double w) override {
        width = w;
        height = w;  // forced to change both!
    }

    void setHeight(double h) override {
        width = h;   // forced to change both!
        height = h;
    }
};

// This function works correctly for Rectangle but BREAKS for Square
void testRectangle(Rectangle& r) {
    r.setWidth(5);
    r.setHeight(4);

    // For a Rectangle, we expect:
    assert(r.getWidth() == 5);   // FAILS for Square! (width became 4)
    assert(r.getHeight() == 4);  // passes
    assert(r.area() == 20);      // FAILS for Square! (area is 16)

    cout << "Expected area 20, got " << r.area() << endl;
}

Rectangle rect(3, 3);
testRectangle(rect);  // works: area = 20

Square sq(3);
testRectangle(sq);    // FAILS: area = 16, not 20!
// Square violates the contract that setWidth and setHeight are independent
```

**Why this violates LSP:**
- `Rectangle` establishes a contract: `setWidth` changes only width, `setHeight` changes only height
- `Square` breaks this contract: setting one dimension changes both
- Code written for `Rectangle` produces incorrect results when given a `Square`
- The subtype is NOT substitutable for the base type

**Solutions to the Rectangle-Square problem:**

**Solution 1: Make shapes immutable (no setters)**
```cpp
class Shape {
public:
    virtual double area() const = 0;
    virtual double perimeter() const = 0;
    virtual ~Shape() = default;
};

class Rectangle : public Shape {
    const double width;
    const double height;
public:
    Rectangle(double w, double h) : width(w), height(h) {}
    double area() const override { return width * height; }
    double perimeter() const override { return 2 * (width + height); }
    double getWidth() const { return width; }
    double getHeight() const { return height; }

    // "Modification" returns a new object
    Rectangle withWidth(double w) const { return Rectangle(w, height); }
    Rectangle withHeight(double h) const { return Rectangle(width, h); }
};

class Square : public Shape {
    const double side;
public:
    Square(double s) : side(s) {}
    double area() const override { return side * side; }
    double perimeter() const override { return 4 * side; }
    double getSide() const { return side; }

    Square withSide(double s) const { return Square(s); }
};

// No LSP violation — no mutable state to create inconsistency
// Square is NOT a subtype of Rectangle — they're siblings under Shape
```

**Solution 2: Separate hierarchies (no inheritance between Rectangle and Square)**
```cpp
class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

class Rectangle : public Shape {
    double width, height;
public:
    Rectangle(double w, double h) : width(w), height(h) {}
    void setWidth(double w) { width = w; }
    void setHeight(double h) { height = h; }
    double area() const override { return width * height; }
};

class Square : public Shape {
    double side;
public:
    Square(double s) : side(s) {}
    void setSide(double s) { side = s; }  // clear, unambiguous interface
    double area() const override { return side * side; }
};

// Square and Rectangle are independent — no LSP issue
// Both are Shapes, but Square is NOT a Rectangle
```

**Solution 3: Use a factory method if you need both**
```cpp
class Rectangle {
    double width, height;
public:
    Rectangle(double w, double h) : width(w), height(h) {}

    // Factory method for creating a square (a rectangle with equal sides)
    static Rectangle createSquare(double side) {
        return Rectangle(side, side);
    }

    void setWidth(double w) { width = w; }
    void setHeight(double h) { height = h; }
    double area() const { return width * height; }
    bool isSquare() const { return width == height; }
};

// No inheritance, no LSP violation
// A "square" is just a rectangle that happens to have equal sides
Rectangle sq = Rectangle::createSquare(5);
sq.setWidth(10);  // perfectly fine — it's just a rectangle now
```

---

### Preconditions and Postconditions

LSP can be formally defined in terms of **preconditions** (what must be true before a method is called) and **postconditions** (what must be true after a method returns).

**LSP Rules:**
1. **Preconditions cannot be strengthened in a subtype** — the derived class cannot demand MORE from the caller than the base class does
2. **Postconditions cannot be weakened in a subtype** — the derived class must guarantee at least as much as the base class promises

```cpp
// Base class contract:
class Account {
protected:
    double balance;

public:
    Account(double initial) : balance(initial) {}

    // Precondition: amount > 0
    // Postcondition: balance increases by exactly 'amount'
    virtual void deposit(double amount) {
        if (amount <= 0) throw invalid_argument("Amount must be positive");
        balance += amount;
    }

    // Precondition: amount > 0 AND amount <= balance
    // Postcondition: balance decreases by exactly 'amount', returns true
    //                OR balance unchanged, returns false (insufficient funds)
    virtual bool withdraw(double amount) {
        if (amount <= 0) throw invalid_argument("Amount must be positive");
        if (amount > balance) return false;
        balance -= amount;
        return true;
    }

    double getBalance() const { return balance; }
    virtual ~Account() = default;
};
```

**LSP-COMPLIANT subclass (weaker precondition, stronger postcondition):**
```cpp
class PremiumAccount : public Account {
    double overdraftLimit;

public:
    PremiumAccount(double initial, double overdraft)
        : Account(initial), overdraftLimit(overdraft) {}

    // WEAKER precondition: allows withdrawal even when amount > balance
    // (up to overdraft limit) — this is FINE, it accepts MORE inputs
    bool withdraw(double amount) override {
        if (amount <= 0) throw invalid_argument("Amount must be positive");
        if (amount > balance + overdraftLimit) return false;
        balance -= amount;  // balance can go negative (up to -overdraftLimit)
        return true;
    }

    // STRONGER postcondition: deposit also earns 1% bonus
    // Guarantees MORE than base class — this is FINE
    void deposit(double amount) override {
        if (amount <= 0) throw invalid_argument("Amount must be positive");
        balance += amount * 1.01;  // 1% bonus — gives MORE than promised
    }
};
```

**LSP-VIOLATING subclass (stronger precondition):**
```cpp
class RestrictedAccount : public Account {
    double maxWithdrawal;

public:
    RestrictedAccount(double initial, double maxW)
        : Account(initial), maxWithdrawal(maxW) {}

    // STRONGER precondition: amount must also be <= maxWithdrawal
    // This VIOLATES LSP — demands MORE from the caller!
    bool withdraw(double amount) override {
        if (amount <= 0) throw invalid_argument("Amount must be positive");
        if (amount > maxWithdrawal)
            throw runtime_error("Exceeds maximum withdrawal limit");  // NEW restriction!
        if (amount > balance) return false;
        balance -= amount;
        return true;
    }
};

// Code that works with Account will BREAK with RestrictedAccount:
void withdrawAll(Account& acc) {
    double balance = acc.getBalance();
    bool success = acc.withdraw(balance);  // should work per Account's contract
    // But RestrictedAccount throws if balance > maxWithdrawal!
    // LSP VIOLATED
}
```

**LSP-VIOLATING subclass (weaker postcondition):**
```cpp
class BrokenAccount : public Account {
public:
    BrokenAccount(double initial) : Account(initial) {}

    // WEAKER postcondition: doesn't guarantee balance increases by 'amount'
    void deposit(double amount) override {
        if (amount <= 0) throw invalid_argument("Amount must be positive");
        // "Processing fee" — balance increases by LESS than amount
        balance += amount * 0.95;  // only 95% deposited — VIOLATES postcondition!
    }
};

// Code expecting Account's contract:
void doubleBalance(Account& acc) {
    double current = acc.getBalance();
    acc.deposit(current);
    assert(acc.getBalance() == 2 * current);  // FAILS with BrokenAccount!
}
```

**Summary of precondition/postcondition rules:**

| Rule | Allowed | Violation |
|------|---------|-----------|
| Precondition | Same or WEAKER (accept more) | STRONGER (reject what base accepts) |
| Postcondition | Same or STRONGER (guarantee more) | WEAKER (guarantee less than base) |
| Invariant | Must be maintained | Breaking class invariant |
| Exceptions | Same or fewer exceptions | New unexpected exceptions |

---

### Covariance and Contravariance

These concepts describe how type relationships are preserved (or reversed) in the context of method signatures.

**Covariance (return types):** If `Dog` is a subtype of `Animal`, then a method returning `Dog` can override a method returning `Animal`. The return type varies in the SAME direction as the class hierarchy.

```cpp
class Animal {
public:
    virtual Animal* clone() const = 0;
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    // Covariant return type: Dog* instead of Animal*
    Dog* clone() const override {
        return new Dog(*this);
    }
};

class Cat : public Animal {
public:
    Cat* clone() const override {
        return new Cat(*this);
    }
};

// Covariance is LSP-compliant because:
// - Code expecting Animal* gets a Dog* (which IS an Animal*) — valid
// - The return type is MORE specific, not less — stronger postcondition
```

**Contravariance (parameter types):** If `Dog` is a subtype of `Animal`, then a method accepting `Animal` can override a method accepting `Dog`. The parameter type varies in the OPPOSITE direction. (Note: C++ does NOT support contravariant parameters in overriding — this is a conceptual explanation.)

```cpp
// Conceptual example (not valid C++ override syntax, but illustrates the principle)

class EventHandler {
public:
    // Base accepts specific type
    virtual void handle(MouseEvent& event) = 0;
    virtual ~EventHandler() = default;
};

// Contravariance would mean: derived accepts MORE general type
// This is LSP-compliant because it accepts everything the base accepts (and more)
// class GeneralHandler : public EventHandler {
//     void handle(Event& event) override;  // accepts any Event, including MouseEvent
// };

// In C++, achieve this through templates or separate overloads:
class GeneralHandler : public EventHandler {
public:
    void handle(MouseEvent& event) override {
        handleAny(event);  // delegate to general handler
    }

    void handleAny(Event& event) {
        // handles any event type
    }
};
```

**Covariance and contravariance in practice (with smart pointers):**

```cpp
class AnimalShelter {
public:
    // Returns a general type
    virtual unique_ptr<Animal> adopt() = 0;
    virtual ~AnimalShelter() = default;
};

class DogShelter : public AnimalShelter {
public:
    // Note: C++ doesn't support covariant smart pointers directly
    // Must return unique_ptr<Animal>, not unique_ptr<Dog>
    unique_ptr<Animal> adopt() override {
        return make_unique<Dog>();  // returns Dog wrapped in Animal pointer
    }

    // If you need the specific type, provide a separate method:
    unique_ptr<Dog> adoptDog() {
        return make_unique<Dog>();
    }
};
```

---

### Behavioral Subtyping

LSP is ultimately about **behavioral subtyping** — the subclass must behave in a way that is consistent with the expectations set by the base class. This goes beyond just matching method signatures.

**Behavioral contract includes:**
1. Method signatures (enforced by compiler)
2. Preconditions and postconditions (logical contracts)
3. Invariants (conditions that must always hold)
4. History constraint (how the object's state evolves over time)

```cpp
// Base class with a behavioral contract
class Stack {
public:
    // Invariant: size() >= 0
    // Invariant: after push(x), top() == x
    // Invariant: after push(x), size() increases by 1
    // Invariant: after pop(), size() decreases by 1

    virtual void push(int value) = 0;
    virtual int pop() = 0;
    virtual int top() const = 0;
    virtual int size() const = 0;
    virtual bool isEmpty() const = 0;
    virtual ~Stack() = default;
};

// LSP-COMPLIANT: Bounded stack (adds a limit but doesn't break existing behavior)
class BoundedStack : public Stack {
    vector<int> data;
    int maxSize;

public:
    BoundedStack(int max) : maxSize(max) {}

    void push(int value) override {
        if (data.size() >= maxSize)
            throw overflow_error("Stack is full");  // NEW precondition? 
        // This is acceptable IF documented in the base class that push() may throw
        // OR if the base class says "push succeeds if there's capacity"
        data.push_back(value);
    }

    int pop() override {
        if (data.empty()) throw underflow_error("Stack is empty");
        int val = data.back();
        data.pop_back();
        return val;
    }

    int top() const override {
        if (data.empty()) throw underflow_error("Stack is empty");
        return data.back();
    }

    int size() const override { return data.size(); }
    bool isEmpty() const override { return data.empty(); }
};

// LSP-VIOLATING: A "stack" that doesn't maintain LIFO order
class RandomStack : public Stack {
    vector<int> data;

public:
    void push(int value) override {
        data.push_back(value);
    }

    int pop() override {
        if (data.empty()) throw underflow_error("Empty");
        // Returns a RANDOM element instead of the last one pushed!
        int idx = rand() % data.size();
        int val = data[idx];
        data.erase(data.begin() + idx);
        return val;  // VIOLATES: after push(x), pop() should return x (if no other push)
    }

    int top() const override {
        if (data.empty()) throw underflow_error("Empty");
        return data.back();  // inconsistent with pop()!
    }

    int size() const override { return data.size(); }
    bool isEmpty() const override { return data.empty(); }
};

// Code that relies on Stack's behavioral contract:
void undoLastAction(Stack& stack) {
    if (!stack.isEmpty()) {
        int lastAction = stack.pop();  // expects LIFO behavior
        cout << "Undoing action: " << lastAction << endl;
    }
}
// RandomStack breaks this — it might "undo" a random action, not the last one!
```

**The History Constraint:**

The history constraint says that a subtype should not allow state transitions that the base type would not allow.

```cpp
// Base: Immutable point (once created, coordinates don't change)
class ImmutablePoint {
    const int x, y;
public:
    ImmutablePoint(int x, int y) : x(x), y(y) {}
    int getX() const { return x; }
    int getY() const { return y; }
    virtual ~ImmutablePoint() = default;
};

// VIOLATION: Mutable subclass breaks the immutability contract
class MutablePoint : public ImmutablePoint {
    int mx, my;  // shadow the base class members
public:
    MutablePoint(int x, int y) : ImmutablePoint(x, y), mx(x), my(y) {}
    void setX(int x) { mx = x; }  // allows mutation!
    void setY(int y) { my = y; }
    // Now getX()/getY() from base return original values,
    // but the object's "real" state has changed — confusing and broken
};
// If code stores an ImmutablePoint reference and expects it to never change,
// a MutablePoint subclass violates that expectation.
```

---

### Signs of LSP Violations

Watch for these code smells that indicate LSP violations:

1. **Type checking in client code:**
```cpp
// If you see this, LSP is likely violated
void process(Animal* a) {
    if (dynamic_cast<Penguin*>(a)) {
        // special handling because Penguin doesn't fit the Animal contract
    } else {
        a->fly();  // assumes all animals can fly
    }
}
```

2. **Empty or throwing method implementations:**
```cpp
class ReadOnlyCollection : public Collection {
    void add(int item) override {
        throw UnsupportedOperationException();  // LSP violation!
    }
};
```

3. **Conditional behavior based on subtype:**
```cpp
void calculateArea(Shape* s) {
    if (typeid(*s) == typeid(Square)) {
        // special formula for square
    } else if (typeid(*s) == typeid(Rectangle)) {
        // different formula for rectangle
    }
}
```

4. **Documentation saying "don't call X on this subtype"**

**How to fix LSP violations:**
- Redesign the hierarchy (separate interfaces for different capabilities)
- Use composition instead of inheritance
- Make the base class contract more precise
- Split into smaller, more focused interfaces (ISP helps here)

---


## 3.4 Interface Segregation Principle (ISP)

> "No client should be forced to depend on methods it does not use." — Robert C. Martin

The Interface Segregation Principle states that large, monolithic interfaces should be split into smaller, more specific ones so that clients only need to know about the methods that are relevant to them. A class should not be forced to implement interfaces it doesn't use.

---

### No Client Should Depend on Methods It Doesn't Use

**The problem with fat interfaces:**

When a class depends on an interface with methods it doesn't need, it becomes coupled to those methods. Changes to unused methods can force recompilation, and implementors are burdened with providing meaningless implementations.

```cpp
// BAD: Fat interface — forces ALL implementations to handle everything
class IMachine {
public:
    virtual void print(const Document& doc) = 0;
    virtual void scan(const Document& doc) = 0;
    virtual void fax(const Document& doc) = 0;
    virtual void staple(const Document& doc) = 0;
    virtual void copy(const Document& doc, int copies) = 0;
    virtual ~IMachine() = default;
};

// A simple printer is FORCED to implement scan, fax, staple, copy!
class SimplePrinter : public IMachine {
public:
    void print(const Document& doc) override {
        cout << "Printing: " << doc.getTitle() << endl;
    }

    // Forced to provide meaningless implementations:
    void scan(const Document& doc) override {
        throw runtime_error("SimplePrinter cannot scan!");  // BAD
    }
    void fax(const Document& doc) override {
        throw runtime_error("SimplePrinter cannot fax!");   // BAD
    }
    void staple(const Document& doc) override {
        // do nothing — silent failure is even worse!
    }
    void copy(const Document& doc, int copies) override {
        throw runtime_error("SimplePrinter cannot copy!");  // BAD
    }
};

// A scanner is FORCED to implement print, fax, staple!
class SimpleScanner : public IMachine {
public:
    void scan(const Document& doc) override {
        cout << "Scanning: " << doc.getTitle() << endl;
    }

    void print(const Document& doc) override {
        throw runtime_error("SimpleScanner cannot print!");
    }
    void fax(const Document& doc) override {
        throw runtime_error("SimpleScanner cannot fax!");
    }
    void staple(const Document& doc) override {
        throw runtime_error("SimpleScanner cannot staple!");
    }
    void copy(const Document& doc, int copies) override {
        throw runtime_error("SimpleScanner cannot copy!");
    }
};
```

**Problems:**
- `SimplePrinter` is polluted with methods it can never meaningfully implement
- Client code that only needs printing still depends on the scan/fax/staple interface
- Adding a new method to `IMachine` forces ALL implementations to change
- Violates LSP: calling `scan()` on a `SimplePrinter` throws unexpectedly

---

### Fat Interfaces vs Thin Interfaces

**Fat Interface:** A single large interface with many methods covering multiple concerns. Forces implementors to deal with all methods even if they only need a subset.

**Thin Interface:** A small, focused interface with a few cohesive methods representing a single capability. Implementors only implement what they actually support.

```cpp
// GOOD: Thin, focused interfaces — each represents ONE capability

class IPrinter {
public:
    virtual void print(const Document& doc) = 0;
    virtual void printMultiple(const Document& doc, int copies) = 0;
    virtual bool isPaperLoaded() const = 0;
    virtual ~IPrinter() = default;
};

class IScanner {
public:
    virtual Document scan() = 0;
    virtual Document scanWithResolution(int dpi) = 0;
    virtual bool isReady() const = 0;
    virtual ~IScanner() = default;
};

class IFax {
public:
    virtual bool sendFax(const Document& doc, const string& number) = 0;
    virtual Document receiveFax() = 0;
    virtual ~IFax() = default;
};

class IStapler {
public:
    virtual void staple(const Document& doc) = 0;
    virtual int getStaplesRemaining() const = 0;
    virtual ~IStapler() = default;
};

// Now each device implements ONLY what it supports:

class SimplePrinter : public IPrinter {
public:
    void print(const Document& doc) override {
        cout << "Printing: " << doc.getTitle() << endl;
    }
    void printMultiple(const Document& doc, int copies) override {
        for (int i = 0; i < copies; i++) print(doc);
    }
    bool isPaperLoaded() const override { return true; }
};

class SimpleScanner : public IScanner {
public:
    Document scan() override {
        cout << "Scanning at default resolution..." << endl;
        return Document("Scanned Document");
    }
    Document scanWithResolution(int dpi) override {
        cout << "Scanning at " << dpi << " DPI..." << endl;
        return Document("High-res Scan");
    }
    bool isReady() const override { return true; }
};

// Multi-function device implements multiple interfaces
class MultiFunctionPrinter : public IPrinter, public IScanner, public IFax, public IStapler {
public:
    // IPrinter
    void print(const Document& doc) override {
        cout << "MFP printing: " << doc.getTitle() << endl;
    }
    void printMultiple(const Document& doc, int copies) override {
        for (int i = 0; i < copies; i++) print(doc);
    }
    bool isPaperLoaded() const override { return true; }

    // IScanner
    Document scan() override {
        cout << "MFP scanning..." << endl;
        return Document("MFP Scan");
    }
    Document scanWithResolution(int dpi) override {
        cout << "MFP scanning at " << dpi << " DPI..." << endl;
        return Document("MFP High-res Scan");
    }
    bool isReady() const override { return true; }

    // IFax
    bool sendFax(const Document& doc, const string& number) override {
        cout << "MFP faxing to " << number << endl;
        return true;
    }
    Document receiveFax() override {
        return Document("Received Fax");
    }

    // IStapler
    void staple(const Document& doc) override {
        cout << "MFP stapling: " << doc.getTitle() << endl;
    }
    int getStaplesRemaining() const override { return 100; }
};
```

**Client code depends only on what it needs:**

```cpp
// This function only needs printing — doesn't know or care about scanning/faxing
void printReport(IPrinter& printer, const Document& report) {
    if (printer.isPaperLoaded()) {
        printer.print(report);
    } else {
        cout << "Please load paper!" << endl;
    }
}

// This function only needs scanning
void digitizeDocument(IScanner& scanner) {
    if (scanner.isReady()) {
        Document scanned = scanner.scanWithResolution(300);
        // ... process scanned document ...
    }
}

// Both SimplePrinter and MultiFunctionPrinter work with printReport()
// Both SimpleScanner and MultiFunctionPrinter work with digitizeDocument()
// Each function depends on the MINIMUM interface it needs
```

---

### Splitting Interfaces in C++ (Multiple Inheritance of Pure Virtual Classes)

C++ achieves ISP through **multiple inheritance of pure virtual classes** (interfaces). Since pure virtual classes have no data members, multiple inheritance is safe (no diamond problem with data).

**Real-world example: Plugin system**

```cpp
// Segregated interfaces for a plugin system

class IInitializable {
public:
    virtual bool initialize(const Config& config) = 0;
    virtual void shutdown() = 0;
    virtual bool isInitialized() const = 0;
    virtual ~IInitializable() = default;
};

class IUpdatable {
public:
    virtual void update(double deltaTime) = 0;
    virtual ~IUpdatable() = default;
};

class IRenderable {
public:
    virtual void render() const = 0;
    virtual void setVisible(bool visible) = 0;
    virtual bool isVisible() const = 0;
    virtual ~IRenderable() = default;
};

class ISerializable {
public:
    virtual string serialize() const = 0;
    virtual void deserialize(const string& data) = 0;
    virtual ~ISerializable() = default;
};

class IEventListener {
public:
    virtual void onEvent(const Event& event) = 0;
    virtual vector<string> getSubscribedEvents() const = 0;
    virtual ~IEventListener() = default;
};

class IConfigurable {
public:
    virtual void configure(const map<string, string>& settings) = 0;
    virtual map<string, string> getConfiguration() const = 0;
    virtual ~IConfigurable() = default;
};

// A physics plugin: needs initialization, updating, and event handling
// Does NOT need rendering or serialization
class PhysicsPlugin : public IInitializable, public IUpdatable, public IEventListener {
    bool initialized = false;

public:
    bool initialize(const Config& config) override {
        cout << "Physics engine initialized" << endl;
        initialized = true;
        return true;
    }
    void shutdown() override { initialized = false; }
    bool isInitialized() const override { return initialized; }

    void update(double dt) override {
        // Update physics simulation
        cout << "Physics step: " << dt << "s" << endl;
    }

    void onEvent(const Event& event) override {
        // Handle collision events, etc.
    }
    vector<string> getSubscribedEvents() const override {
        return {"collision", "force_applied"};
    }
};

// A UI widget: needs rendering, events, and configuration
// Does NOT need physics updates or serialization
class UIWidget : public IRenderable, public IEventListener, public IConfigurable {
    bool visible = true;
    map<string, string> config;

public:
    void render() const override {
        if (visible) cout << "Rendering widget" << endl;
    }
    void setVisible(bool v) override { visible = v; }
    bool isVisible() const override { return visible; }

    void onEvent(const Event& event) override {
        // Handle click, hover, etc.
    }
    vector<string> getSubscribedEvents() const override {
        return {"click", "hover", "key_press"};
    }

    void configure(const map<string, string>& settings) override {
        config = settings;
    }
    map<string, string> getConfiguration() const override {
        return config;
    }
};

// A save-game system: needs serialization and initialization only
class SaveSystem : public IInitializable, public ISerializable {
    bool initialized = false;
    string gameState;

public:
    bool initialize(const Config& config) override {
        initialized = true;
        return true;
    }
    void shutdown() override { initialized = false; }
    bool isInitialized() const override { return initialized; }

    string serialize() const override { return gameState; }
    void deserialize(const string& data) override { gameState = data; }
};

// The engine processes each interface independently:
class GameEngine {
    vector<IInitializable*> initializables;
    vector<IUpdatable*> updatables;
    vector<IRenderable*> renderables;
    vector<IEventListener*> listeners;

public:
    void registerInitializable(IInitializable* obj) { initializables.push_back(obj); }
    void registerUpdatable(IUpdatable* obj) { updatables.push_back(obj); }
    void registerRenderable(IRenderable* obj) { renderables.push_back(obj); }
    void registerListener(IEventListener* obj) { listeners.push_back(obj); }

    void initializeAll(const Config& config) {
        for (auto* obj : initializables) obj->initialize(config);
    }

    void updateAll(double dt) {
        for (auto* obj : updatables) obj->update(dt);
    }

    void renderAll() {
        for (auto* obj : renderables) {
            if (obj->isVisible()) obj->render();
        }
    }

    void broadcastEvent(const Event& event) {
        for (auto* listener : listeners) listener->onEvent(event);
    }
};
```

---

### Role Interfaces

A **role interface** defines a specific role that an object can play in a particular context. Instead of defining interfaces based on what an object IS, define them based on what role the object PLAYS.

```cpp
// Role-based interfaces — defined by what the object DOES in a specific context

// Role: Something that can be persisted to storage
class IPersistable {
public:
    virtual string getEntityId() const = 0;
    virtual map<string, string> toRecord() const = 0;
    virtual void fromRecord(const map<string, string>& record) = 0;
    virtual ~IPersistable() = default;
};

// Role: Something that can be validated
class IValidatable {
public:
    virtual bool isValid() const = 0;
    virtual vector<string> getValidationErrors() const = 0;
    virtual ~IValidatable() = default;
};

// Role: Something that can be audited (track changes)
class IAuditable {
public:
    virtual string getLastModifiedBy() const = 0;
    virtual time_t getLastModifiedAt() const = 0;
    virtual vector<string> getChangeHistory() const = 0;
    virtual ~IAuditable() = default;
};

// Role: Something that can be displayed to users
class IDisplayable {
public:
    virtual string getDisplayName() const = 0;
    virtual string getDescription() const = 0;
    virtual string getIconPath() const = 0;
    virtual ~IDisplayable() = default;
};

// Role: Something that can be searched/indexed
class ISearchable {
public:
    virtual vector<string> getSearchableFields() const = 0;
    virtual string getSearchSummary() const = 0;
    virtual ~ISearchable() = default;
};

// A Product plays multiple roles depending on context
class Product : public IPersistable, public IValidatable,
                public IAuditable, public IDisplayable, public ISearchable {
    string id;
    string name;
    string description;
    double price;
    int stock;
    string lastModifier;
    time_t lastModified;
    vector<string> history;

public:
    Product(const string& id, const string& name, double price)
        : id(id), name(name), price(price), stock(0),
          lastModified(time(nullptr)) {}

    // IPersistable role
    string getEntityId() const override { return id; }
    map<string, string> toRecord() const override {
        return {{"id", id}, {"name", name}, {"price", to_string(price)},
                {"stock", to_string(stock)}};
    }
    void fromRecord(const map<string, string>& record) override {
        name = record.at("name");
        price = stod(record.at("price"));
        stock = stoi(record.at("stock"));
    }

    // IValidatable role
    bool isValid() const override {
        return !name.empty() && price > 0 && stock >= 0;
    }
    vector<string> getValidationErrors() const override {
        vector<string> errors;
        if (name.empty()) errors.push_back("Name is required");
        if (price <= 0) errors.push_back("Price must be positive");
        if (stock < 0) errors.push_back("Stock cannot be negative");
        return errors;
    }

    // IAuditable role
    string getLastModifiedBy() const override { return lastModifier; }
    time_t getLastModifiedAt() const override { return lastModified; }
    vector<string> getChangeHistory() const override { return history; }

    // IDisplayable role
    string getDisplayName() const override { return name; }
    string getDescription() const override { return description; }
    string getIconPath() const override { return "/icons/product.png"; }

    // ISearchable role
    vector<string> getSearchableFields() const override {
        return {name, description, id};
    }
    string getSearchSummary() const override {
        return name + " - $" + to_string(price);
    }
};

// Each subsystem only depends on the role it cares about:

class Repository {
public:
    void save(IPersistable& entity) {
        auto record = entity.toRecord();
        cout << "Saving entity " << entity.getEntityId() << endl;
        // ... save to database ...
    }
};

class ValidationService {
public:
    bool validate(const IValidatable& entity) {
        if (!entity.isValid()) {
            for (const auto& error : entity.getValidationErrors()) {
                cerr << "Validation error: " << error << endl;
            }
            return false;
        }
        return true;
    }
};

class SearchEngine {
public:
    void index(const ISearchable& entity) {
        auto fields = entity.getSearchableFields();
        cout << "Indexing: " << entity.getSearchSummary() << endl;
        // ... add to search index ...
    }
};

class AuditLogger {
public:
    void logChange(const IAuditable& entity) {
        cout << "Change by " << entity.getLastModifiedBy()
             << " at " << entity.getLastModifiedAt() << endl;
    }
};
```

---

### Guidelines for Interface Segregation

**How to identify interfaces that need splitting:**

1. **Look for "throw UnsupportedOperationException" patterns** — if implementors throw for methods they can't support, the interface is too fat

2. **Look for empty method bodies** — if implementors leave methods empty, they don't need those methods

3. **Look for groups of methods used together** — methods that are always called together belong in the same interface; methods called independently should be in separate interfaces

4. **Apply the "Interface Pollution" test** — if adding a new implementor requires implementing methods that make no sense for it, the interface needs splitting

**Granularity balance:**

```cpp
// TOO GRANULAR — one method per interface is excessive
class ICanAdd { virtual void add(int x) = 0; };
class ICanRemove { virtual void remove(int x) = 0; };
class ICanFind { virtual bool find(int x) = 0; };
class ICanSize { virtual int size() = 0; };
// This creates too many tiny interfaces — hard to work with

// TOO COARSE — everything in one interface
class IEverything {
    virtual void add(int x) = 0;
    virtual void remove(int x) = 0;
    virtual bool find(int x) = 0;
    virtual int size() = 0;
    virtual void sort() = 0;
    virtual void serialize() = 0;
    virtual void render() = 0;
    // ... 20 more methods
};

// JUST RIGHT — cohesive groups of related methods
class ICollection {
    virtual void add(int x) = 0;
    virtual void remove(int x) = 0;
    virtual bool contains(int x) const = 0;
    virtual int size() const = 0;
    virtual bool isEmpty() const = 0;
};

class ISortable {
    virtual void sort() = 0;
    virtual void sortBy(function<bool(int,int)> comparator) = 0;
};

class ISerializable {
    virtual string serialize() const = 0;
    virtual void deserialize(const string& data) = 0;
};
```

**ISP and the Dependency Inversion Principle work together:**
- ISP tells you to make interfaces small and focused
- DIP tells you to depend on those interfaces rather than concrete classes
- Together, they create loosely coupled, highly cohesive designs

---


## 3.5 Dependency Inversion Principle (DIP)

> "A. High-level modules should not depend on low-level modules. Both should depend on abstractions."
> "B. Abstractions should not depend on details. Details should depend on abstractions."
> — Robert C. Martin

The Dependency Inversion Principle is about reversing the traditional dependency direction. Instead of high-level business logic depending directly on low-level implementation details (database, file system, network), both should depend on abstract interfaces. This makes the system flexible, testable, and resilient to change.

---

### High-Level Modules Should Not Depend on Low-Level Modules

**What are high-level and low-level modules?**
- **High-level modules:** Contain business logic, policies, and orchestration. They define WHAT the system does.
- **Low-level modules:** Contain implementation details — database access, file I/O, network calls, UI rendering. They define HOW things are done.

**The problem — direct dependency (traditional layering):**

```cpp
// LOW-LEVEL: Concrete database implementation
class MySQLDatabase {
public:
    void connect(const string& connectionString) {
        cout << "Connected to MySQL: " << connectionString << endl;
    }

    vector<map<string, string>> query(const string& sql) {
        cout << "MySQL executing: " << sql << endl;
        return {};  // results
    }

    void execute(const string& sql) {
        cout << "MySQL executing: " << sql << endl;
    }
};

// LOW-LEVEL: Concrete email service
class SendGridEmailService {
    string apiKey;
public:
    SendGridEmailService(const string& key) : apiKey(key) {}

    void sendEmail(const string& to, const string& subject, const string& body) {
        cout << "SendGrid: sending to " << to << endl;
    }
};

// HIGH-LEVEL: Business logic — DIRECTLY depends on low-level modules!
class OrderService {
    MySQLDatabase database;              // DIRECT dependency on MySQL!
    SendGridEmailService emailService;   // DIRECT dependency on SendGrid!

public:
    OrderService() : emailService("api-key-123") {
        database.connect("mysql://localhost/orders");
    }

    void placeOrder(const string& userId, const vector<string>& items) {
        // Business logic mixed with infrastructure details
        database.execute("INSERT INTO orders ...");  // tied to MySQL
        emailService.sendEmail(userId, "Order Confirmed", "...");  // tied to SendGrid

        // What if we want to:
        // - Switch to PostgreSQL? Must modify OrderService!
        // - Switch to Mailgun? Must modify OrderService!
        // - Test without a real database? Impossible!
        // - Test without sending real emails? Impossible!
    }
};
```

**Problems with direct dependencies:**
- Cannot switch database without modifying business logic
- Cannot switch email provider without modifying business logic
- Cannot unit test business logic in isolation
- High-level policy is coupled to low-level implementation details
- Changes in infrastructure ripple up to business logic

---

### Both Should Depend on Abstractions

**The solution — invert the dependency direction:**

```cpp
// ABSTRACTIONS (interfaces) — owned by the high-level module

class IOrderRepository {
public:
    virtual void save(const Order& order) = 0;
    virtual Order findById(const string& orderId) = 0;
    virtual vector<Order> findByUser(const string& userId) = 0;
    virtual void updateStatus(const string& orderId, OrderStatus status) = 0;
    virtual ~IOrderRepository() = default;
};

class INotificationService {
public:
    virtual void sendOrderConfirmation(const string& userId, const Order& order) = 0;
    virtual void sendShippingUpdate(const string& userId, const string& trackingNum) = 0;
    virtual void sendCancellationNotice(const string& userId, const Order& order) = 0;
    virtual ~INotificationService() = default;
};

class IPaymentGateway {
public:
    virtual PaymentResult charge(const string& userId, double amount) = 0;
    virtual PaymentResult refund(const string& transactionId, double amount) = 0;
    virtual ~IPaymentGateway() = default;
};

class IInventoryService {
public:
    virtual bool isAvailable(const string& productId, int quantity) = 0;
    virtual void reserve(const string& productId, int quantity) = 0;
    virtual void release(const string& productId, int quantity) = 0;
    virtual ~IInventoryService() = default;
};

// HIGH-LEVEL: Business logic depends ONLY on abstractions
class OrderService {
    IOrderRepository& repository;
    INotificationService& notifications;
    IPaymentGateway& payments;
    IInventoryService& inventory;

public:
    OrderService(IOrderRepository& repo, INotificationService& notif,
                 IPaymentGateway& pay, IInventoryService& inv)
        : repository(repo), notifications(notif), payments(pay), inventory(inv) {}

    Order placeOrder(const string& userId, const vector<OrderItem>& items) {
        // Pure business logic — no infrastructure details!

        // 1. Check inventory
        for (const auto& item : items) {
            if (!inventory.isAvailable(item.productId, item.quantity)) {
                throw out_of_stock_error(item.productId);
            }
        }

        // 2. Calculate total
        double total = 0;
        for (const auto& item : items) {
            total += item.price * item.quantity;
        }

        // 3. Process payment
        auto paymentResult = payments.charge(userId, total);
        if (!paymentResult.success) {
            throw payment_failed_error(paymentResult.errorMessage);
        }

        // 4. Reserve inventory
        for (const auto& item : items) {
            inventory.reserve(item.productId, item.quantity);
        }

        // 5. Create and save order
        Order order(userId, items, total, paymentResult.transactionId);
        repository.save(order);

        // 6. Send confirmation
        notifications.sendOrderConfirmation(userId, order);

        return order;
    }

    void cancelOrder(const string& orderId) {
        auto order = repository.findById(orderId);

        // Refund payment
        payments.refund(order.getTransactionId(), order.getTotal());

        // Release inventory
        for (const auto& item : order.getItems()) {
            inventory.release(item.productId, item.quantity);
        }

        // Update status
        repository.updateStatus(orderId, OrderStatus::Cancelled);

        // Notify customer
        notifications.sendCancellationNotice(order.getUserId(), order);
    }
};

// LOW-LEVEL: Concrete implementations depend on the same abstractions

class MySQLOrderRepository : public IOrderRepository {
    // MySQL-specific connection, queries, etc.
public:
    void save(const Order& order) override {
        cout << "MySQL: INSERT INTO orders ..." << endl;
    }
    Order findById(const string& orderId) override {
        cout << "MySQL: SELECT * FROM orders WHERE id = ..." << endl;
        return Order{};
    }
    vector<Order> findByUser(const string& userId) override {
        return {};
    }
    void updateStatus(const string& orderId, OrderStatus status) override {
        cout << "MySQL: UPDATE orders SET status = ..." << endl;
    }
};

class PostgresOrderRepository : public IOrderRepository {
    // PostgreSQL-specific implementation
public:
    void save(const Order& order) override {
        cout << "PostgreSQL: INSERT INTO orders ..." << endl;
    }
    Order findById(const string& orderId) override { return Order{}; }
    vector<Order> findByUser(const string& userId) override { return {}; }
    void updateStatus(const string& orderId, OrderStatus status) override {}
};

class EmailNotificationService : public INotificationService {
public:
    void sendOrderConfirmation(const string& userId, const Order& order) override {
        cout << "Email: Order confirmation to " << userId << endl;
    }
    void sendShippingUpdate(const string& userId, const string& trackingNum) override {
        cout << "Email: Shipping update to " << userId << endl;
    }
    void sendCancellationNotice(const string& userId, const Order& order) override {
        cout << "Email: Cancellation notice to " << userId << endl;
    }
};

class SMSNotificationService : public INotificationService {
public:
    void sendOrderConfirmation(const string& userId, const Order& order) override {
        cout << "SMS: Order confirmation to " << userId << endl;
    }
    void sendShippingUpdate(const string& userId, const string& trackingNum) override {
        cout << "SMS: Shipping update to " << userId << endl;
    }
    void sendCancellationNotice(const string& userId, const Order& order) override {
        cout << "SMS: Cancellation notice to " << userId << endl;
    }
};

class StripePaymentGateway : public IPaymentGateway {
    string apiKey;
public:
    StripePaymentGateway(const string& key) : apiKey(key) {}
    PaymentResult charge(const string& userId, double amount) override {
        cout << "Stripe: charging $" << amount << endl;
        return {true, "txn_123", ""};
    }
    PaymentResult refund(const string& transactionId, double amount) override {
        cout << "Stripe: refunding $" << amount << endl;
        return {true, transactionId, ""};
    }
};
```

**The dependency direction is now INVERTED:**

```
BEFORE (traditional):
  OrderService ──depends on──→ MySQLDatabase
  OrderService ──depends on──→ SendGridEmailService

AFTER (inverted):
  OrderService ──depends on──→ IOrderRepository ←──implements── MySQLOrderRepository
  OrderService ──depends on──→ INotificationService ←──implements── EmailNotificationService

Both high-level and low-level depend on the ABSTRACTION (interface).
The abstraction is owned by the high-level module.
```

---

### Dependency Injection in C++ (Constructor, Setter, Interface Injection)

Dependency Injection (DI) is the practical mechanism for implementing DIP. It's how you actually provide the concrete implementations to the high-level module at runtime.

#### Constructor Injection (Preferred)

Dependencies are provided through the constructor. The object is always fully initialized.

```cpp
class ReportGenerator {
    IDataSource& dataSource;
    IFormatter& formatter;
    IOutputTarget& output;

public:
    // All dependencies injected at construction — object is immediately usable
    ReportGenerator(IDataSource& ds, IFormatter& fmt, IOutputTarget& out)
        : dataSource(ds), formatter(fmt), output(out) {}

    void generateReport(const string& reportType) {
        auto data = dataSource.fetchData(reportType);
        auto formatted = formatter.format(data);
        output.write(formatted);
    }
};

// Production configuration
MySQLDataSource prodData("mysql://prod-server/analytics");
HTMLFormatter htmlFmt;
FileOutputTarget fileOut("/reports/daily.html");
ReportGenerator prodReport(prodData, htmlFmt, fileOut);
prodReport.generateReport("daily_sales");

// Test configuration — completely different implementations, same ReportGenerator
MockDataSource testData;
PlainTextFormatter textFmt;
StringOutputTarget stringOut;
ReportGenerator testReport(testData, textFmt, stringOut);
testReport.generateReport("test");
// Verify: stringOut.getContent() contains expected output
```

**Advantages of constructor injection:**
- Object is always in a valid state (all dependencies present from creation)
- Dependencies are clearly visible in the constructor signature
- Immutable dependencies (using references) — cannot be changed after construction
- Easy to see when a class has too many dependencies (constructor parameter count)

#### Setter Injection

Dependencies are provided through setter methods after construction. Useful for optional dependencies or when dependencies need to change at runtime.

```cpp
class NotificationManager {
    IEmailSender* emailSender = nullptr;
    ISMSSender* smsSender = nullptr;
    IPushNotifier* pushNotifier = nullptr;

public:
    NotificationManager() = default;

    // Optional dependencies — set only what's available
    void setEmailSender(IEmailSender* sender) { emailSender = sender; }
    void setSMSSender(ISMSSender* sender) { smsSender = sender; }
    void setPushNotifier(IPushNotifier* notifier) { pushNotifier = notifier; }

    void notifyUser(const string& userId, const string& message) {
        if (emailSender) emailSender->send(userId, message);
        if (smsSender) smsSender->send(userId, message);
        if (pushNotifier) pushNotifier->notify(userId, message);
    }

    bool hasAnyChannel() const {
        return emailSender || smsSender || pushNotifier;
    }
};

// Configure with available channels
NotificationManager manager;
SendGridSender email("api-key");
manager.setEmailSender(&email);

TwilioSMSSender sms("account-sid", "auth-token");
manager.setSMSSender(&sms);
// Push notifications not available in this environment — that's fine

manager.notifyUser("user123", "Your order has shipped!");
```

**When to use setter injection:**
- Optional dependencies (not all implementations need all dependencies)
- Dependencies that change at runtime (strategy pattern)
- Circular dependencies (rare, usually a design smell)
- Framework-managed lifecycle where construction and configuration are separate steps

#### Interface Injection

A dedicated interface defines how dependencies are injected. The container/framework calls the injection method.

```cpp
// Injection interfaces
class IDatabaseAware {
public:
    virtual void setDatabase(IDatabase& db) = 0;
    virtual ~IDatabaseAware() = default;
};

class ILoggerAware {
public:
    virtual void setLogger(ILogger& logger) = 0;
    virtual ~ILoggerAware() = default;
};

class ICacheAware {
public:
    virtual void setCache(ICache& cache) = 0;
    virtual ~ICacheAware() = default;
};

// Service implements injection interfaces to declare its dependencies
class UserService : public IDatabaseAware, public ILoggerAware, public ICacheAware {
    IDatabase* db = nullptr;
    ILogger* logger = nullptr;
    ICache* cache = nullptr;

public:
    void setDatabase(IDatabase& database) override { db = &database; }
    void setLogger(ILogger& log) override { logger = &log; }
    void setCache(ICache& c) override { cache = &c; }

    User findUser(const string& id) {
        // Check cache first
        if (cache) {
            auto cached = cache->get("user:" + id);
            if (cached) {
                logger->log("Cache hit for user " + id);
                return deserialize(*cached);
            }
        }

        // Query database
        logger->log("Cache miss, querying DB for user " + id);
        auto user = db->query("SELECT * FROM users WHERE id = " + id);

        // Store in cache
        if (cache) cache->put("user:" + id, serialize(user));

        return user;
    }
};

// A simple DI container that wires dependencies
class DIContainer {
    IDatabase* database = nullptr;
    ILogger* logger = nullptr;
    ICache* cache = nullptr;

public:
    void registerDatabase(IDatabase* db) { database = db; }
    void registerLogger(ILogger* log) { logger = log; }
    void registerCache(ICache* c) { cache = c; }

    template <typename T>
    void inject(T& service) {
        if constexpr (is_base_of_v<IDatabaseAware, T>) {
            if (database) service.setDatabase(*database);
        }
        if constexpr (is_base_of_v<ILoggerAware, T>) {
            if (logger) service.setLogger(*logger);
        }
        if constexpr (is_base_of_v<ICacheAware, T>) {
            if (cache) service.setCache(*cache);
        }
    }
};

// Usage
PostgresDatabase db("postgres://localhost/app");
ConsoleLogger logger;
RedisCache cache("redis://localhost");

DIContainer container;
container.registerDatabase(&db);
container.registerLogger(&logger);
container.registerCache(&cache);

UserService userService;
container.inject(userService);  // automatically wires all dependencies

auto user = userService.findUser("user-42");
```

---

### Inversion of Control (IoC) Concept

**Inversion of Control** is the broader principle that DIP and DI implement. In traditional programming, your code calls library code. With IoC, the framework calls your code. You give up control of the flow to the framework.

**Traditional control flow (you call the library):**
```cpp
// YOU decide when to create objects and call methods
int main() {
    MySQLDatabase db;
    db.connect("localhost");

    EmailService email;
    email.configure("smtp.gmail.com");

    OrderService orders(db, email);
    orders.processOrder("order-1");

    return 0;
}
```

**Inversion of Control (the framework calls you):**
```cpp
// Framework/container manages object creation and lifecycle

class Application {
public:
    virtual void configure(ServiceRegistry& registry) = 0;
    virtual void onStart() = 0;
    virtual void onShutdown() = 0;
    virtual ~Application() = default;
};

class MyApp : public Application {
public:
    void configure(ServiceRegistry& registry) override {
        // DECLARE what you need — the framework creates and wires them
        registry.registerSingleton<IDatabase, PostgresDatabase>();
        registry.registerSingleton<INotificationService, EmailNotificationService>();
        registry.registerTransient<IOrderRepository, PostgresOrderRepository>();
        registry.registerSingleton<OrderService>();
    }

    void onStart() override {
        // Framework has already created and wired everything
        auto& orderService = ServiceLocator::get<OrderService>();
        cout << "Application started, OrderService ready" << endl;
    }

    void onShutdown() override {
        cout << "Application shutting down" << endl;
    }
};

// The framework controls the lifecycle:
// 1. Creates MyApp
// 2. Calls configure() to learn about dependencies
// 3. Creates all registered services in correct order
// 4. Injects dependencies automatically
// 5. Calls onStart()
// 6. ... application runs ...
// 7. Calls onShutdown()
// 8. Destroys services in reverse order
```

**A simple IoC container in C++:**

```cpp
class ServiceContainer {
    map<string, function<void*()>> factories;
    map<string, void*> singletons;

public:
    // Register a factory for creating instances
    template <typename Interface, typename Implementation>
    void registerTransient() {
        string key = typeid(Interface).name();
        factories[key] = []() -> void* {
            return new Implementation();
        };
    }

    // Register a singleton (created once, shared)
    template <typename Interface, typename Implementation>
    void registerSingleton() {
        string key = typeid(Interface).name();
        factories[key] = [this, key]() -> void* {
            if (singletons.find(key) == singletons.end()) {
                singletons[key] = new Implementation();
            }
            return singletons[key];
        };
    }

    // Resolve a dependency
    template <typename Interface>
    Interface* resolve() {
        string key = typeid(Interface).name();
        if (factories.find(key) == factories.end()) {
            throw runtime_error("Service not registered: " + key);
        }
        return static_cast<Interface*>(factories[key]());
    }
};

// Usage
ServiceContainer container;
container.registerSingleton<ILogger, FileLogger>();
container.registerSingleton<IDatabase, PostgresDatabase>();
container.registerTransient<IOrderRepository, PostgresOrderRepository>();

auto* logger = container.resolve<ILogger>();
auto* db = container.resolve<IDatabase>();
auto* repo = container.resolve<IOrderRepository>();
```

---

### DIP in Practice — Complete Example

```cpp
// A complete example showing DIP applied to a notification system

// --- Abstractions (owned by high-level module) ---

class IMessageFormatter {
public:
    virtual string format(const string& templateName,
                         const map<string, string>& variables) const = 0;
    virtual ~IMessageFormatter() = default;
};

class IDeliveryChannel {
public:
    virtual bool deliver(const string& recipient, const string& subject,
                        const string& body) = 0;
    virtual string getChannelName() const = 0;
    virtual bool isAvailable() const = 0;
    virtual ~IDeliveryChannel() = default;
};

class IRecipientResolver {
public:
    virtual string resolveAddress(const string& userId,
                                  const string& channelType) const = 0;
    virtual bool hasChannel(const string& userId,
                           const string& channelType) const = 0;
    virtual ~IRecipientResolver() = default;
};

class INotificationLogger {
public:
    virtual void logSent(const string& userId, const string& channel,
                        const string& subject) = 0;
    virtual void logFailed(const string& userId, const string& channel,
                          const string& reason) = 0;
    virtual ~INotificationLogger() = default;
};

// --- High-level module: Notification orchestration ---

class NotificationEngine {
    IMessageFormatter& formatter;
    vector<IDeliveryChannel*> channels;
    IRecipientResolver& resolver;
    INotificationLogger& logger;

public:
    NotificationEngine(IMessageFormatter& fmt, IRecipientResolver& res,
                      INotificationLogger& log)
        : formatter(fmt), resolver(res), logger(log) {}

    void addChannel(IDeliveryChannel* channel) {
        channels.push_back(channel);
    }

    bool notify(const string& userId, const string& templateName,
                const map<string, string>& variables) {
        string body = formatter.format(templateName, variables);
        string subject = variables.count("subject") ? variables.at("subject") : "Notification";

        bool anySent = false;

        for (auto* channel : channels) {
            if (!channel->isAvailable()) continue;

            string channelType = channel->getChannelName();
            if (!resolver.hasChannel(userId, channelType)) continue;

            string address = resolver.resolveAddress(userId, channelType);

            if (channel->deliver(address, subject, body)) {
                logger.logSent(userId, channelType, subject);
                anySent = true;
            } else {
                logger.logFailed(userId, channelType, "Delivery failed");
            }
        }

        if (!anySent) {
            logger.logFailed(userId, "all", "No channel delivered successfully");
        }

        return anySent;
    }
};

// --- Low-level modules: Concrete implementations ---

class HandlebarsFormatter : public IMessageFormatter {
    map<string, string> templates;
public:
    void loadTemplate(const string& name, const string& tmpl) {
        templates[name] = tmpl;
    }

    string format(const string& templateName,
                 const map<string, string>& variables) const override {
        string result = templates.at(templateName);
        for (const auto& [key, value] : variables) {
            string placeholder = "{{" + key + "}}";
            size_t pos;
            while ((pos = result.find(placeholder)) != string::npos) {
                result.replace(pos, placeholder.size(), value);
            }
        }
        return result;
    }
};

class SMTPEmailChannel : public IDeliveryChannel {
    string smtpServer;
    int port;
public:
    SMTPEmailChannel(const string& server, int p) : smtpServer(server), port(p) {}

    bool deliver(const string& recipient, const string& subject,
                const string& body) override {
        cout << "SMTP: Sending to " << recipient << " via " << smtpServer << endl;
        return true;
    }

    string getChannelName() const override { return "email"; }
    bool isAvailable() const override { return true; }
};

class TwilioSMSChannel : public IDeliveryChannel {
    string accountSid;
    string authToken;
public:
    TwilioSMSChannel(const string& sid, const string& token)
        : accountSid(sid), authToken(token) {}

    bool deliver(const string& recipient, const string& subject,
                const string& body) override {
        cout << "Twilio SMS: Sending to " << recipient << endl;
        return true;
    }

    string getChannelName() const override { return "sms"; }
    bool isAvailable() const override { return true; }
};

class SlackChannel : public IDeliveryChannel {
    string webhookUrl;
public:
    SlackChannel(const string& url) : webhookUrl(url) {}

    bool deliver(const string& recipient, const string& subject,
                const string& body) override {
        cout << "Slack: Posting to " << recipient << endl;
        return true;
    }

    string getChannelName() const override { return "slack"; }
    bool isAvailable() const override { return true; }
};

class DatabaseRecipientResolver : public IRecipientResolver {
public:
    string resolveAddress(const string& userId,
                         const string& channelType) const override {
        // Query user preferences from database
        if (channelType == "email") return userId + "@example.com";
        if (channelType == "sms") return "+1234567890";
        if (channelType == "slack") return "@" + userId;
        return "";
    }

    bool hasChannel(const string& userId, const string& channelType) const override {
        return true;  // simplified
    }
};

class FileNotificationLogger : public INotificationLogger {
    ofstream logFile;
public:
    FileNotificationLogger(const string& path) { logFile.open(path, ios::app); }

    void logSent(const string& userId, const string& channel,
                const string& subject) override {
        logFile << "[SENT] " << userId << " via " << channel << ": " << subject << endl;
    }

    void logFailed(const string& userId, const string& channel,
                  const string& reason) override {
        logFile << "[FAILED] " << userId << " via " << channel << ": " << reason << endl;
    }
};

// --- Wiring it all together ---

void setupNotificationSystem() {
    // Create low-level implementations
    HandlebarsFormatter formatter;
    formatter.loadTemplate("order_confirm",
        "Hi {{name}}, your order #{{orderId}} has been confirmed!");
    formatter.loadTemplate("shipping_update",
        "{{name}}, your order #{{orderId}} is on its way! Tracking: {{tracking}}");

    DatabaseRecipientResolver resolver;
    FileNotificationLogger logger("notifications.log");

    SMTPEmailChannel email("smtp.example.com", 587);
    TwilioSMSChannel sms("AC123", "token456");
    SlackChannel slack("https://hooks.slack.com/...");

    // Wire high-level module with low-level implementations
    NotificationEngine engine(formatter, resolver, logger);
    engine.addChannel(&email);
    engine.addChannel(&sms);
    engine.addChannel(&slack);

    // Use the system — high-level module doesn't know about SMTP, Twilio, or Slack
    engine.notify("alice", "order_confirm", {
        {"subject", "Order Confirmed"},
        {"name", "Alice"},
        {"orderId", "ORD-789"}
    });
}
```

---

### Key Benefits of DIP

| Benefit | Explanation |
|---------|-------------|
| Testability | Inject mock implementations for unit testing |
| Flexibility | Swap implementations without changing business logic |
| Parallel development | Teams can work on high-level and low-level modules independently |
| Plugin architecture | New implementations can be added without modifying existing code |
| Configuration-driven | Choose implementations at deployment time (dev vs prod) |
| Reduced coupling | Changes in low-level modules don't ripple up to business logic |

**DIP vs DI vs IoC:**

| Concept | What it is | Level |
|---------|-----------|-------|
| DIP (Dependency Inversion Principle) | Design principle — depend on abstractions | Architectural guideline |
| DI (Dependency Injection) | Technique — pass dependencies from outside | Implementation pattern |
| IoC (Inversion of Control) | Broader concept — framework controls flow | Architectural pattern |

DIP is the principle, DI is one way to implement it, and IoC is the general pattern that encompasses both.

---


## 3.6 Other Important Principles

> Beyond SOLID, several other design principles guide good object-oriented design. These principles complement SOLID and are frequently referenced in design discussions and interviews.

---

### DRY (Don't Repeat Yourself)

> "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."

DRY is about eliminating duplication of **knowledge** (not just code). If the same logic, rule, or concept exists in multiple places, a change requires updating all of them — and forgetting one creates bugs.

```cpp
// BAD: Duplicated validation logic
class UserRegistration {
public:
    bool registerUser(const string& email, const string& password) {
        // Email validation duplicated here
        if (email.empty() || email.find('@') == string::npos ||
            email.find('.') == string::npos) {
            return false;
        }
        // Password validation duplicated here
        if (password.size() < 8 || !hasUppercase(password) || !hasDigit(password)) {
            return false;
        }
        // ... register ...
        return true;
    }
};

class UserProfileUpdate {
public:
    bool updateEmail(const string& newEmail) {
        // SAME email validation duplicated!
        if (newEmail.empty() || newEmail.find('@') == string::npos ||
            newEmail.find('.') == string::npos) {
            return false;
        }
        // ... update ...
        return true;
    }
};

class PasswordReset {
public:
    bool resetPassword(const string& newPassword) {
        // SAME password validation duplicated!
        if (newPassword.size() < 8 || !hasUppercase(newPassword) || !hasDigit(newPassword)) {
            return false;
        }
        // ... reset ...
        return true;
    }
};
```

```cpp
// GOOD: Single source of truth for validation rules
class EmailValidator {
public:
    static bool isValid(const string& email) {
        if (email.empty()) return false;
        if (email.find('@') == string::npos) return false;
        auto atPos = email.find('@');
        string domain = email.substr(atPos + 1);
        return domain.find('.') != string::npos && domain.size() > 3;
    }

    static vector<string> getErrors(const string& email) {
        vector<string> errors;
        if (email.empty()) errors.push_back("Email is required");
        else if (email.find('@') == string::npos) errors.push_back("Missing @ symbol");
        return errors;
    }
};

class PasswordValidator {
public:
    static bool isValid(const string& password) {
        return password.size() >= 8 && hasUppercase(password) &&
               hasLowercase(password) && hasDigit(password);
    }

    static vector<string> getErrors(const string& password) {
        vector<string> errors;
        if (password.size() < 8) errors.push_back("Must be at least 8 characters");
        if (!hasUppercase(password)) errors.push_back("Must contain uppercase letter");
        if (!hasDigit(password)) errors.push_back("Must contain a digit");
        return errors;
    }

private:
    static bool hasUppercase(const string& s) {
        return any_of(s.begin(), s.end(), ::isupper);
    }
    static bool hasLowercase(const string& s) {
        return any_of(s.begin(), s.end(), ::islower);
    }
    static bool hasDigit(const string& s) {
        return any_of(s.begin(), s.end(), ::isdigit);
    }
};

// Now all classes use the single source of truth:
class UserRegistration {
public:
    bool registerUser(const string& email, const string& password) {
        if (!EmailValidator::isValid(email)) return false;
        if (!PasswordValidator::isValid(password)) return false;
        // ... register ...
        return true;
    }
};
```

**DRY is NOT just about code duplication:**
- Two identical code blocks that represent DIFFERENT concepts should NOT be merged
- If they change for different reasons, they are not true duplicates
- "Accidental duplication" (same code, different purposes) is fine to keep separate

---

### KISS (Keep It Simple, Stupid)

> "The simplest solution that works is usually the best."

KISS advocates for simplicity in design. Don't over-engineer, don't add unnecessary abstractions, and don't use complex patterns when a straightforward solution works.

```cpp
// BAD: Over-engineered for a simple task
// Just need to check if a number is even...

class NumberClassificationStrategyFactory {
    static map<string, unique_ptr<INumberClassificationStrategy>> strategies;
public:
    static INumberClassificationStrategy& getStrategy(const string& type) {
        return *strategies[type];
    }
};

class INumberClassificationStrategy {
public:
    virtual bool classify(int n) = 0;
    virtual ~INumberClassificationStrategy() = default;
};

class EvenNumberStrategy : public INumberClassificationStrategy {
public:
    bool classify(int n) override { return n % 2 == 0; }
};

// ... 50 lines of code to check if a number is even

// GOOD: Simple and clear
bool isEven(int n) {
    return n % 2 == 0;
}
```

**When complexity IS justified:**
- When requirements genuinely demand it (multiple payment methods → Strategy pattern)
- When you have evidence of future extension needs (not speculation)
- When the complexity reduces overall system complexity (a well-designed abstraction)

**KISS in practice:**
```cpp
// KISS: Use the simplest data structure that works
// Need to store unique items and check membership? Use unordered_set, not a custom B-tree.
unordered_set<string> activeUsers;
activeUsers.insert("alice");
bool isActive = activeUsers.count("alice") > 0;

// KISS: Use standard library algorithms instead of reimplementing
vector<int> nums = {5, 2, 8, 1, 9};
sort(nums.begin(), nums.end());  // don't write your own sort!
auto it = find(nums.begin(), nums.end(), 8);  // don't write your own search!
```

---

### YAGNI (You Aren't Gonna Need It)

> "Don't implement something until you actually need it."

YAGNI warns against adding functionality based on speculation about future requirements. Premature features add complexity, maintenance burden, and often turn out to be wrong when the actual requirement arrives.

```cpp
// BAD: Adding "just in case" features that aren't needed yet

class UserService {
    IDatabase& db;
    ICache& cache;
    IMessageQueue& queue;           // "We might need async processing someday"
    IAnalyticsService& analytics;   // "We might want to track this later"
    IAuditLog& auditLog;            // "Compliance might require this eventually"
    IEncryptionService& encryption; // "We might need to encrypt data at rest"

public:
    // Constructor with 6 dependencies for a service that currently just does CRUD!
    UserService(IDatabase& db, ICache& cache, IMessageQueue& q,
                IAnalyticsService& a, IAuditLog& al, IEncryptionService& e)
        : db(db), cache(cache), queue(q), analytics(a), auditLog(al), encryption(e) {}

    User createUser(const string& name, const string& email) {
        auto encrypted = encryption.encrypt(email);  // not needed yet!
        User user(name, encrypted);
        db.save(user);
        cache.invalidate("users");
        queue.publish("user.created", user.toJSON());  // no consumer exists yet!
        analytics.track("user_created", {{"name", name}});  // no dashboard yet!
        auditLog.log("CREATE", "user", user.getId());  // no compliance requirement yet!
        return user;
    }
};
```

```cpp
// GOOD: Only what's needed NOW

class UserService {
    IDatabase& db;

public:
    UserService(IDatabase& db) : db(db) {}

    User createUser(const string& name, const string& email) {
        User user(name, email);
        db.save(user);
        return user;
    }

    // Add caching WHEN performance becomes an issue
    // Add analytics WHEN the business requests it
    // Add encryption WHEN compliance requires it
    // Add async processing WHEN load demands it
};
```

**YAGNI does NOT mean:**
- Don't plan ahead at all
- Don't write clean, extensible code
- Ignore known upcoming requirements

**YAGNI DOES mean:**
- Don't implement features until they're actually needed
- Don't add abstractions for hypothetical future scenarios
- Keep the design clean so features CAN be added later (OCP helps here)

---

### Law of Demeter (Principle of Least Knowledge)

> "Only talk to your immediate friends. Don't talk to strangers."

A method should only call methods on:
1. Its own object (`this`)
2. Objects passed as parameters
3. Objects it creates
4. Its direct component objects (members)

It should NOT call methods on objects returned by other methods (no "train wrecks").

```cpp
// BAD: Violating Law of Demeter — reaching through objects
class Address {
public:
    string getCity() const { return city; }
    string getZipCode() const { return zipCode; }
private:
    string city, zipCode;
};

class Customer {
public:
    Address& getAddress() { return address; }
private:
    Address address;
};

class Order {
public:
    Customer& getCustomer() { return customer; }
private:
    Customer customer;
};

// "Train wreck" — reaching deep into the object graph
void printShippingLabel(Order& order) {
    // BAD: order → customer → address → city (3 levels deep!)
    string city = order.getCustomer().getAddress().getCity();
    string zip = order.getCustomer().getAddress().getZipCode();
    cout << city << ", " << zip << endl;
}
```

```cpp
// GOOD: Each object provides what's needed directly

class Order {
    Customer customer;
public:
    // Order provides shipping info directly — hides internal structure
    string getShippingCity() const { return customer.getCity(); }
    string getShippingZip() const { return customer.getZipCode(); }
    string getShippingLabel() const {
        return customer.getCity() + ", " + customer.getZipCode();
    }
};

class Customer {
    Address address;
public:
    // Customer delegates to Address — hides Address from callers
    string getCity() const { return address.getCity(); }
    string getZipCode() const { return address.getZipCode(); }
};

void printShippingLabel(Order& order) {
    // GOOD: only talks to the Order object directly
    cout << order.getShippingLabel() << endl;
}
```

**Benefits:**
- Reduces coupling (caller doesn't know about Address class)
- If Address structure changes, only Customer needs updating (not all callers)
- Easier to test (mock only the immediate dependency)

**When it's OK to "violate" Law of Demeter:**
- Fluent interfaces / method chaining (builder pattern): `builder.setName("x").setAge(5).build()`
- Data structures (DTOs, value objects): accessing nested data is their purpose
- Standard library: `vec.begin()->second` is idiomatic C++

---

### Composition over Inheritance

> "Favor object composition over class inheritance." — Gang of Four

This principle advises using composition (has-a) to achieve code reuse and polymorphism rather than inheritance (is-a), because composition is more flexible and creates less coupling.

```cpp
// BAD: Deep inheritance hierarchy for behavior combinations
class Animal { /* ... */ };
class FlyingAnimal : public Animal { virtual void fly() = 0; };
class SwimmingAnimal : public Animal { virtual void swim() = 0; };
class FlyingSwimmingAnimal : public FlyingAnimal, public SwimmingAnimal { /* diamond! */ };
// What about a walking-swimming-flying animal? Explosion of classes!

// GOOD: Compose behaviors
class IMovementBehavior {
public:
    virtual void move() = 0;
    virtual ~IMovementBehavior() = default;
};

class FlyBehavior : public IMovementBehavior {
public:
    void move() override { cout << "Flying through the air" << endl; }
};

class SwimBehavior : public IMovementBehavior {
public:
    void move() override { cout << "Swimming in water" << endl; }
};

class WalkBehavior : public IMovementBehavior {
public:
    void move() override { cout << "Walking on land" << endl; }
};

class Animal {
    string name;
    vector<unique_ptr<IMovementBehavior>> movements;  // composition!

public:
    Animal(const string& n) : name(n) {}

    void addMovement(unique_ptr<IMovementBehavior> behavior) {
        movements.push_back(std::move(behavior));
    }

    void moveAll() {
        cout << name << ": ";
        for (auto& m : movements) m->move();
    }
};

// Any combination without class explosion!
Animal duck("Duck");
duck.addMovement(make_unique<FlyBehavior>());
duck.addMovement(make_unique<SwimBehavior>());
duck.addMovement(make_unique<WalkBehavior>());
duck.moveAll();
```

---

### Program to an Interface, Not an Implementation

> "Depend on abstractions, not on concretions."

This principle (closely related to DIP) means that variables, parameters, and return types should be declared as abstract types (interfaces) rather than concrete classes whenever possible.

```cpp
// BAD: Programming to implementation
void processData(vector<int>& data) {  // tied to vector specifically
    // What if we want to use a list or deque?
}

MySQLDatabase* db = new MySQLDatabase();  // tied to MySQL

// GOOD: Programming to interface
void processData(const ICollection<int>& data) {  // works with any collection
}

unique_ptr<IDatabase> db = make_unique<PostgresDatabase>();  // can swap implementations

// In function signatures:
void saveUser(IUserRepository& repo, const User& user) {
    repo.save(user);  // works with ANY repository implementation
}
```

---

### Encapsulate What Varies

> "Identify the aspects of your application that vary and separate them from what stays the same."

Find the parts of your code that are likely to change and encapsulate them behind stable interfaces. This way, changes are isolated and don't ripple through the system.

```cpp
// What varies: tax calculation rules (different per country, change frequently)
// What stays the same: the order processing flow

class ITaxCalculator {
public:
    virtual double calculateTax(double amount, const string& category) const = 0;
    virtual ~ITaxCalculator() = default;
};

class USTaxCalculator : public ITaxCalculator {
public:
    double calculateTax(double amount, const string& category) const override {
        if (category == "food") return amount * 0.0;   // no tax on food
        if (category == "luxury") return amount * 0.12;
        return amount * 0.08;  // default sales tax
    }
};

class UKTaxCalculator : public ITaxCalculator {
public:
    double calculateTax(double amount, const string& category) const override {
        if (category == "food") return amount * 0.0;
        if (category == "luxury") return amount * 0.20;
        return amount * 0.20;  // VAT
    }
};

// Order processing is STABLE — doesn't change when tax rules change
class OrderProcessor {
    ITaxCalculator& taxCalc;  // the varying part is encapsulated

public:
    OrderProcessor(ITaxCalculator& calc) : taxCalc(calc) {}

    double calculateTotal(const vector<OrderItem>& items) {
        double subtotal = 0;
        double tax = 0;
        for (const auto& item : items) {
            subtotal += item.price * item.quantity;
            tax += taxCalc.calculateTax(item.price * item.quantity, item.category);
        }
        return subtotal + tax;
    }
};
```

---

### Favor Object Composition over Class Inheritance

This is the Gang of Four's formulation of "Composition over Inheritance." The key insight is that inheritance creates a compile-time, static relationship that cannot change, while composition creates a runtime, dynamic relationship that can be reconfigured.

```cpp
// Inheritance: behavior is fixed at compile time
class Logger {
public:
    virtual void log(const string& msg) = 0;
};

class TimestampLogger : public Logger {
public:
    void log(const string& msg) override {
        cout << "[" << time(nullptr) << "] " << msg << endl;
    }
};

class EncryptedTimestampLogger : public TimestampLogger {
    // What if we want encryption WITHOUT timestamp?
    // What if we want timestamp + encryption + compression?
    // Inheritance doesn't compose well!
};

// Composition: behavior can be combined freely at runtime (Decorator pattern)
class ILogger {
public:
    virtual void log(const string& msg) = 0;
    virtual ~ILogger() = default;
};

class ConsoleLogger : public ILogger {
public:
    void log(const string& msg) override { cout << msg << endl; }
};

class LoggerDecorator : public ILogger {
protected:
    unique_ptr<ILogger> wrapped;
public:
    LoggerDecorator(unique_ptr<ILogger> logger) : wrapped(std::move(logger)) {}
};

class TimestampDecorator : public LoggerDecorator {
public:
    using LoggerDecorator::LoggerDecorator;
    void log(const string& msg) override {
        wrapped->log("[" + to_string(time(nullptr)) + "] " + msg);
    }
};

class EncryptionDecorator : public LoggerDecorator {
public:
    using LoggerDecorator::LoggerDecorator;
    void log(const string& msg) override {
        wrapped->log(encrypt(msg));
    }
private:
    string encrypt(const string& msg) { return "ENC(" + msg + ")"; }
};

class CompressionDecorator : public LoggerDecorator {
public:
    using LoggerDecorator::LoggerDecorator;
    void log(const string& msg) override {
        wrapped->log(compress(msg));
    }
private:
    string compress(const string& msg) { return "ZIP(" + msg + ")"; }
};

// Compose ANY combination at runtime!
auto logger = make_unique<TimestampDecorator>(
    make_unique<EncryptionDecorator>(
        make_unique<ConsoleLogger>()
    )
);
logger->log("Hello");  // outputs: [1234567890] ENC(Hello)

// Different combination — no new classes needed:
auto logger2 = make_unique<CompressionDecorator>(
    make_unique<TimestampDecorator>(
        make_unique<ConsoleLogger>()
    )
);
logger2->log("Hello");  // outputs: ZIP([1234567890] Hello)
```

---

### How These Principles Relate to SOLID

| Principle | Supports SOLID | How |
|-----------|---|---|
| DRY | SRP | Eliminates duplicated responsibilities |
| KISS | All | Prevents over-engineering that obscures design |
| YAGNI | OCP, SRP | Prevents premature abstractions |
| Law of Demeter | DIP, ISP | Reduces coupling between modules |
| Composition over Inheritance | OCP, LSP, DIP | Enables flexible, substitutable designs |
| Program to Interface | DIP, OCP | Depends on abstractions, enables extension |
| Encapsulate What Varies | OCP, SRP | Isolates change behind stable interfaces |

---


## Summary & Key Takeaways

| Principle | Core Idea | Key Technique |
|-----------|-----------|---------------|
| SRP | One class, one reason to change | Extract classes by responsibility/actor |
| OCP | Open for extension, closed for modification | Inheritance, Strategy, Template Method |
| LSP | Subtypes must be substitutable for base types | Honor contracts, preconditions, postconditions |
| ISP | No client depends on methods it doesn't use | Split fat interfaces into thin, focused ones |
| DIP | Depend on abstractions, not concretions | Dependency Injection, IoC containers |
| DRY | Single source of truth for every piece of knowledge | Extract shared logic into reusable components |
| KISS | Simplest solution that works | Avoid over-engineering |
| YAGNI | Don't build what you don't need yet | Implement on demand, not speculation |
| Law of Demeter | Only talk to immediate friends | Delegate, don't reach through objects |
| Composition over Inheritance | Has-a over is-a for flexibility | Strategy, Decorator, Delegation |

---

## Interview Tips for Module 3

1. **SRP — "What is a reason to change?"** Explain that a "reason to change" maps to a stakeholder/actor. If HR and IT both request changes to the same class, it has two responsibilities. Give the Employee example (business logic vs persistence vs presentation).

2. **OCP — "How do you add features without modifying existing code?"** Explain using abstract base classes and polymorphism. The Strategy pattern is the cleanest example. Show how a new payment method or compression algorithm can be added by creating a new class without touching existing ones.

3. **LSP — "What's the Rectangle-Square problem?"** Explain that Square inheriting from Rectangle violates LSP because `setWidth`/`setHeight` have different semantics. Code expecting independent width/height breaks with Square. Solutions: immutable shapes, separate hierarchies, or factory methods.

4. **LSP — "What are preconditions and postconditions?"** Preconditions cannot be strengthened (subclass can't demand more), postconditions cannot be weakened (subclass can't guarantee less). Give the Account example where a restricted account adds new restrictions that break callers.

5. **ISP — "What's wrong with a fat interface?"** Implementors are forced to provide meaningless implementations (throw or no-op). Clients depend on methods they don't use, creating unnecessary coupling. Show the Printer/Scanner/Fax example and how splitting into IPrinter, IScanner, IFax solves it.

6. **DIP — "What's the difference between DIP, DI, and IoC?"** DIP is the principle (depend on abstractions). DI is the technique (inject dependencies from outside). IoC is the broader pattern (framework controls flow). DIP tells you WHAT to do, DI tells you HOW to do it.

7. **DIP — "How do you test code that uses a database?"** Define an IRepository interface. In production, inject a real database implementation. In tests, inject a mock/stub that returns predefined data. The business logic doesn't know or care which one it's using.

8. **Composition over Inheritance — "When would you use inheritance vs composition?"** Use inheritance for genuine "is-a" relationships where LSP holds (Dog is-a Animal). Use composition for "has-a" relationships, when behavior needs to change at runtime, or when you'd need multiple inheritance to combine features.

9. **SOLID violations in practice — "How do you identify violations?"** SRP: class has methods serving different actors. OCP: adding a feature requires modifying existing switch/if-else chains. LSP: type checks or "unsupported operation" exceptions. ISP: empty method implementations. DIP: `new ConcreteClass()` inside business logic.

10. **Trade-offs — "Can you over-apply SOLID?"** Yes! Over-applying SRP creates too many tiny classes. Over-applying OCP creates premature abstractions. Over-applying DIP adds unnecessary indirection. Apply SOLID where you see or expect change, not everywhere. KISS and YAGNI balance SOLID.

11. **Real-world application — "How do SOLID principles help in system design?"** They make code testable (DIP + DI), extensible (OCP), maintainable (SRP), and flexible (LSP + ISP). In interviews, show how you'd structure a class hierarchy for a parking lot, elevator system, or payment processor using these principles.

12. **Law of Demeter — "What's a train wreck?"** `order.getCustomer().getAddress().getCity()` — reaching through multiple objects. Fix by having Order expose `getShippingCity()` directly. Reduces coupling and makes refactoring easier.

13. **DRY vs Premature Abstraction** — Not all code that looks similar IS duplicate. Two functions with identical code that serve different purposes and change for different reasons should remain separate. DRY applies to knowledge/concepts, not just syntax.

14. **Applying SOLID to a design problem** — In an interview, when designing a system:
    - Identify responsibilities → SRP (separate classes)
    - Identify extension points → OCP (use interfaces/abstract classes)
    - Verify substitutability → LSP (can any subclass replace the base?)
    - Check interface size → ISP (are clients forced to depend on unused methods?)
    - Check dependency direction → DIP (does business logic depend on infrastructure?)
