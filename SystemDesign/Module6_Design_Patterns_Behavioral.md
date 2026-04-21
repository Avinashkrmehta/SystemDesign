# Module 6: Design Patterns — Behavioral

> Behavioral patterns deal with communication between objects — they define how objects interact, distribute responsibility, and coordinate workflows. Unlike creational patterns (which handle object creation) or structural patterns (which handle object composition), behavioral patterns focus on **algorithms and the assignment of responsibilities** between objects. They help make complex control flows understandable by encapsulating behavior in objects and letting you vary it independently.

---

## 6.1 Strategy Pattern

> The Strategy pattern defines a family of algorithms, encapsulates each one in its own class, and makes them interchangeable. It lets the algorithm vary independently from the clients that use it. The client selects which algorithm to use at runtime, without knowing the implementation details.

---

### Why Strategy?

Many situations require choosing between different algorithms or behaviors at runtime:
- Sorting: quicksort vs mergesort vs heapsort depending on data characteristics
- Payment: credit card vs PayPal vs crypto vs bank transfer
- Compression: gzip vs brotli vs lz4 depending on speed/size tradeoff
- Navigation: shortest path vs fastest route vs avoid tolls
- Pricing: regular vs premium vs student discount

**The problem without Strategy:**
```cpp
// BAD: Conditional logic scattered everywhere — violates OCP and SRP
class ShippingCalculator {
public:
    double calculate(const string& method, double weight, double distance) {
        if (method == "ground") {
            return weight * 1.5 + distance * 0.5;
        } else if (method == "air") {
            return weight * 3.0 + distance * 1.2;
        } else if (method == "express") {
            return weight * 5.0 + distance * 2.0 + 10.0;  // surcharge
        } else if (method == "drone") {
            // New method — must modify this class!
            return weight * 4.0 + distance * 0.8;
        }
        // Every new shipping method means modifying this class
        // Violates Open/Closed Principle!
        // Testing requires testing ALL branches in one class
        throw invalid_argument("Unknown method: " + method);
    }
};
```

**Problems:**
1. Violates **Open/Closed Principle** — must modify existing code to add new algorithms
2. Violates **Single Responsibility Principle** — one class knows all algorithms
3. Conditional chains grow and become hard to maintain
4. Can't reuse individual algorithms elsewhere
5. Hard to test — must test all branches in one class

---

### Structure

```
┌──────────────────────┐        ┌─────────────────────────┐
│      Context          │        │     Strategy (interface) │
├──────────────────────┤        ├─────────────────────────┤
│ - strategy: Strategy* │───────→│ + execute(data): result  │
├──────────────────────┤        └─────────────────────────┘
│ + setStrategy(s)      │                    ▲
│ + executeStrategy()   │                    │
└──────────────────────┘         ┌───────────┼───────────┐
                                 │           │           │
                          ConcreteA    ConcreteB    ConcreteC
                          + execute()  + execute()  + execute()
```

**Participants:**
- **Strategy (interface):** Declares the common interface for all algorithms
- **ConcreteStrategy:** Implements a specific algorithm
- **Context:** Maintains a reference to a Strategy object; delegates work to it

---

### Basic Implementation — Sorting Strategies

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <memory>
using namespace std;

// --- Strategy Interface ---
class SortStrategy {
public:
    virtual void sort(vector<int>& data) = 0;
    virtual string name() const = 0;
    virtual ~SortStrategy() = default;
};

// --- Concrete Strategies ---
class BubbleSort : public SortStrategy {
public:
    void sort(vector<int>& data) override {
        int n = data.size();
        for (int i = 0; i < n - 1; i++) {
            for (int j = 0; j < n - i - 1; j++) {
                if (data[j] > data[j + 1]) {
                    swap(data[j], data[j + 1]);
                }
            }
        }
    }
    string name() const override { return "BubbleSort"; }
};

class QuickSort : public SortStrategy {
    void quicksort(vector<int>& data, int low, int high) {
        if (low < high) {
            int pivot = data[high];
            int i = low - 1;
            for (int j = low; j < high; j++) {
                if (data[j] <= pivot) {
                    swap(data[++i], data[j]);
                }
            }
            swap(data[i + 1], data[high]);
            int pi = i + 1;
            quicksort(data, low, pi - 1);
            quicksort(data, pi + 1, high);
        }
    }
public:
    void sort(vector<int>& data) override {
        if (!data.empty()) {
            quicksort(data, 0, data.size() - 1);
        }
    }
    string name() const override { return "QuickSort"; }
};

class MergeSort : public SortStrategy {
    void merge(vector<int>& data, int left, int mid, int right) {
        vector<int> temp(right - left + 1);
        int i = left, j = mid + 1, k = 0;
        while (i <= mid && j <= right) {
            temp[k++] = (data[i] <= data[j]) ? data[i++] : data[j++];
        }
        while (i <= mid) temp[k++] = data[i++];
        while (j <= right) temp[k++] = data[j++];
        for (int idx = 0; idx < k; idx++) {
            data[left + idx] = temp[idx];
        }
    }
    void mergesort(vector<int>& data, int left, int right) {
        if (left < right) {
            int mid = left + (right - left) / 2;
            mergesort(data, left, mid);
            mergesort(data, mid + 1, right);
            merge(data, left, mid, right);
        }
    }
public:
    void sort(vector<int>& data) override {
        if (!data.empty()) {
            mergesort(data, 0, data.size() - 1);
        }
    }
    string name() const override { return "MergeSort"; }
};

// --- Context ---
class Sorter {
    unique_ptr<SortStrategy> strategy;
public:
    void setStrategy(unique_ptr<SortStrategy> s) {
        strategy = std::move(s);
    }

    void sort(vector<int>& data) {
        if (!strategy) {
            throw runtime_error("No sorting strategy set!");
        }
        cout << "Sorting with " << strategy->name() << "..." << endl;
        strategy->sort(data);
    }
};

// --- Client code ---
int main() {
    vector<int> data = {38, 27, 43, 3, 9, 82, 10};

    Sorter sorter;

    // Use BubbleSort for small data
    sorter.setStrategy(make_unique<BubbleSort>());
    sorter.sort(data);

    // Switch to QuickSort for larger data
    data = {38, 27, 43, 3, 9, 82, 10, 55, 1, 99, 23};
    sorter.setStrategy(make_unique<QuickSort>());
    sorter.sort(data);

    // Switch to MergeSort when stability matters
    data = {38, 27, 43, 3, 9, 82, 10};
    sorter.setStrategy(make_unique<MergeSort>());
    sorter.sort(data);

    return 0;
}
```

---

### Real-World Example — Payment Processing

```cpp
#include <iostream>
#include <string>
#include <memory>
using namespace std;

// --- Strategy Interface ---
class PaymentStrategy {
public:
    virtual bool pay(double amount) = 0;
    virtual bool validate() const = 0;
    virtual string getMethodName() const = 0;
    virtual ~PaymentStrategy() = default;
};

// --- Concrete Strategies ---
class CreditCardPayment : public PaymentStrategy {
    string cardNumber;
    string expiry;
    string cvv;
public:
    CreditCardPayment(const string& card, const string& exp, const string& c)
        : cardNumber(card), expiry(exp), cvv(c) {}

    bool validate() const override {
        // Luhn algorithm check (simplified)
        if (cardNumber.length() < 13 || cardNumber.length() > 19) return false;
        if (cvv.length() != 3 && cvv.length() != 4) return false;
        cout << "  Credit card validated (Luhn check passed)" << endl;
        return true;
    }

    bool pay(double amount) override {
        if (!validate()) {
            cout << "  Credit card validation failed!" << endl;
            return false;
        }
        cout << "  Paid $" << amount << " via Credit Card ending in "
             << cardNumber.substr(cardNumber.length() - 4) << endl;
        return true;
    }

    string getMethodName() const override { return "Credit Card"; }
};

class PayPalPayment : public PaymentStrategy {
    string email;
    string password;
    bool authenticated = false;
public:
    PayPalPayment(const string& e, const string& p)
        : email(e), password(p) {}

    bool validate() const override {
        if (email.find('@') == string::npos) return false;
        cout << "  PayPal account validated" << endl;
        return true;
    }

    bool pay(double amount) override {
        if (!validate()) {
            cout << "  PayPal validation failed!" << endl;
            return false;
        }
        cout << "  Paid $" << amount << " via PayPal (" << email << ")" << endl;
        return true;
    }

    string getMethodName() const override { return "PayPal"; }
};

class CryptoPayment : public PaymentStrategy {
    string walletAddress;
    string currency;  // BTC, ETH, etc.
public:
    CryptoPayment(const string& wallet, const string& curr)
        : walletAddress(wallet), currency(curr) {}

    bool validate() const override {
        if (walletAddress.length() < 26) return false;
        cout << "  Crypto wallet validated (" << currency << ")" << endl;
        return true;
    }

    bool pay(double amount) override {
        if (!validate()) {
            cout << "  Crypto wallet validation failed!" << endl;
            return false;
        }
        // Convert to crypto amount (simplified)
        double cryptoAmount = amount / 50000.0;  // simplified BTC rate
        cout << "  Paid " << cryptoAmount << " " << currency
             << " (≈$" << amount << ") to wallet " << walletAddress.substr(0, 8)
             << "..." << endl;
        return true;
    }

    string getMethodName() const override { return "Crypto (" + currency + ")"; }
};

class BankTransferPayment : public PaymentStrategy {
    string bankName;
    string accountNumber;
    string routingNumber;
public:
    BankTransferPayment(const string& bank, const string& account, const string& routing)
        : bankName(bank), accountNumber(account), routingNumber(routing) {}

    bool validate() const override {
        if (accountNumber.empty() || routingNumber.empty()) return false;
        cout << "  Bank account validated (" << bankName << ")" << endl;
        return true;
    }

    bool pay(double amount) override {
        if (!validate()) {
            cout << "  Bank transfer validation failed!" << endl;
            return false;
        }
        cout << "  Transferred $" << amount << " via " << bankName
             << " (account ending " << accountNumber.substr(accountNumber.length() - 4)
             << ")" << endl;
        return true;
    }

    string getMethodName() const override { return "Bank Transfer (" + bankName + ")"; }
};

// --- Context ---
class PaymentProcessor {
    unique_ptr<PaymentStrategy> strategy;
public:
    void setPaymentMethod(unique_ptr<PaymentStrategy> s) {
        strategy = std::move(s);
    }

    bool processPayment(double amount) {
        if (!strategy) {
            cout << "No payment method selected!" << endl;
            return false;
        }
        cout << "Processing $" << amount << " via " << strategy->getMethodName() << ":" << endl;
        bool success = strategy->pay(amount);
        if (success) {
            cout << "  Payment successful!" << endl;
        } else {
            cout << "  Payment failed!" << endl;
        }
        return success;
    }
};

// --- Client code ---
int main() {
    PaymentProcessor processor;

    // Pay with credit card
    processor.setPaymentMethod(
        make_unique<CreditCardPayment>("4111111111111111", "12/25", "123"));
    processor.processPayment(99.99);

    cout << endl;

    // Switch to PayPal
    processor.setPaymentMethod(
        make_unique<PayPalPayment>("user@example.com", "secret"));
    processor.processPayment(49.99);

    cout << endl;

    // Switch to crypto
    processor.setPaymentMethod(
        make_unique<CryptoPayment>("1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa", "BTC"));
    processor.processPayment(250.00);

    return 0;
}
```

---

### Strategy with `std::function` (Modern C++ Approach)

Instead of creating a class hierarchy, you can use `std::function` for lightweight strategies:

```cpp
#include <iostream>
#include <functional>
#include <string>
#include <vector>
#include <cmath>
using namespace std;

// Strategy as std::function — no class hierarchy needed
using PricingStrategy = function<double(double basePrice, int quantity)>;

// --- Define strategies as lambdas or functions ---
PricingStrategy regularPricing = [](double price, int qty) {
    return price * qty;
};

PricingStrategy premiumDiscount = [](double price, int qty) {
    return price * qty * 0.9;  // 10% off
};

PricingStrategy bulkDiscount = [](double price, int qty) {
    if (qty >= 100) return price * qty * 0.7;       // 30% off
    else if (qty >= 50) return price * qty * 0.8;    // 20% off
    else if (qty >= 10) return price * qty * 0.9;    // 10% off
    return price * qty;
};

PricingStrategy seasonalSale = [](double price, int qty) {
    return price * qty * 0.75;  // 25% off seasonal
};

// --- Context ---
class ShoppingCart {
    PricingStrategy pricingStrategy;
    struct Item {
        string name;
        double price;
        int quantity;
    };
    vector<Item> items;

public:
    ShoppingCart(PricingStrategy strategy) : pricingStrategy(strategy) {}

    void setPricingStrategy(PricingStrategy strategy) {
        pricingStrategy = strategy;
    }

    void addItem(const string& name, double price, int quantity) {
        items.push_back({name, price, quantity});
    }

    double calculateTotal() const {
        double total = 0;
        for (const auto& item : items) {
            total += pricingStrategy(item.price, item.quantity);
        }
        return total;
    }

    void printReceipt() const {
        cout << "=== Receipt ===" << endl;
        for (const auto& item : items) {
            double cost = pricingStrategy(item.price, item.quantity);
            cout << item.name << " x" << item.quantity
                 << " @ $" << item.price << " = $" << cost << endl;
        }
        cout << "Total: $" << calculateTotal() << endl;
    }
};

int main() {
    ShoppingCart cart(regularPricing);
    cart.addItem("Widget", 10.0, 5);
    cart.addItem("Gadget", 25.0, 3);

    cout << "Regular pricing:" << endl;
    cart.printReceipt();

    cout << "\nPremium discount:" << endl;
    cart.setPricingStrategy(premiumDiscount);
    cart.printReceipt();

    cout << "\nSeasonal sale:" << endl;
    cart.setPricingStrategy(seasonalSale);
    cart.printReceipt();

    // Can even create inline strategies
    cart.setPricingStrategy([](double price, int qty) {
        return price * qty * 0.5;  // 50% flash sale!
    });
    cout << "\nFlash sale:" << endl;
    cart.printReceipt();

    return 0;
}
```

**When to use `std::function` vs class hierarchy:**

| Aspect | Class Hierarchy | `std::function` |
|--------|----------------|-----------------|
| State | Strategies can hold complex state | Lambdas can capture state, but less structured |
| Reusability | Easy to reuse across projects | Tied to specific function signature |
| Extensibility | New strategies via new classes | New strategies via new lambdas/functions |
| Overhead | Virtual dispatch (vtable) | `std::function` has type-erasure overhead |
| Complexity | More boilerplate (classes, files) | Lightweight, inline |
| Testing | Easy to mock/test individually | Harder to name and test individually |
| Best for | Complex strategies with state | Simple, stateless algorithms |

---

### Eliminating Conditional Statements with Strategy

**Before (conditional mess):**
```cpp
class TextFormatter {
public:
    string format(const string& text, const string& style) {
        if (style == "uppercase") {
            string result = text;
            transform(result.begin(), result.end(), result.begin(), ::toupper);
            return result;
        } else if (style == "lowercase") {
            string result = text;
            transform(result.begin(), result.end(), result.begin(), ::tolower);
            return result;
        } else if (style == "titlecase") {
            string result = text;
            bool newWord = true;
            for (char& c : result) {
                if (newWord && isalpha(c)) { c = toupper(c); newWord = false; }
                else { c = tolower(c); }
                if (c == ' ') newWord = true;
            }
            return result;
        } else if (style == "reverse") {
            string result = text;
            reverse(result.begin(), result.end());
            return result;
        }
        return text;
    }
};
```

**After (Strategy pattern):**
```cpp
#include <functional>
#include <unordered_map>
#include <algorithm>

using FormatStrategy = function<string(const string&)>;

class TextFormatter {
    unordered_map<string, FormatStrategy> strategies;
    FormatStrategy currentStrategy;

public:
    TextFormatter() {
        // Register built-in strategies
        strategies["uppercase"] = [](const string& text) {
            string r = text;
            transform(r.begin(), r.end(), r.begin(), ::toupper);
            return r;
        };
        strategies["lowercase"] = [](const string& text) {
            string r = text;
            transform(r.begin(), r.end(), r.begin(), ::tolower);
            return r;
        };
        strategies["titlecase"] = [](const string& text) {
            string r = text;
            bool newWord = true;
            for (char& c : r) {
                if (newWord && isalpha(c)) { c = toupper(c); newWord = false; }
                else { c = tolower(c); }
                if (c == ' ') newWord = true;
            }
            return r;
        };
        strategies["reverse"] = [](const string& text) {
            string r = text;
            reverse(r.begin(), r.end());
            return r;
        };

        currentStrategy = strategies["lowercase"];  // default
    }

    // Add new strategies without modifying existing code — OCP!
    void registerStrategy(const string& name, FormatStrategy strategy) {
        strategies[name] = strategy;
    }

    void setStrategy(const string& name) {
        auto it = strategies.find(name);
        if (it == strategies.end()) throw invalid_argument("Unknown strategy: " + name);
        currentStrategy = it->second;
    }

    string format(const string& text) {
        return currentStrategy(text);
    }
};

int main() {
    TextFormatter formatter;

    formatter.setStrategy("uppercase");
    cout << formatter.format("hello world") << endl;  // HELLO WORLD

    formatter.setStrategy("titlecase");
    cout << formatter.format("hello world") << endl;  // Hello World

    // Add a custom strategy at runtime — no modification needed!
    formatter.registerStrategy("leetspeak", [](const string& text) {
        string r = text;
        for (char& c : r) {
            switch (tolower(c)) {
                case 'a': c = '4'; break;
                case 'e': c = '3'; break;
                case 'i': c = '1'; break;
                case 'o': c = '0'; break;
                case 's': c = '5'; break;
                case 't': c = '7'; break;
            }
        }
        return r;
    });
    formatter.setStrategy("leetspeak");
    cout << formatter.format("Strategy Pattern") << endl;  // 57r473gy P4773rn

    return 0;
}
```

---

### When to Use Strategy

| Scenario | Why Strategy Helps |
|----------|-------------------|
| Multiple algorithms for the same task | Encapsulate each, swap at runtime |
| Conditional logic selecting behavior | Replace if/else chains with polymorphism |
| Algorithm details should be hidden from client | Client only knows the interface |
| Need to add new algorithms without modifying existing code | OCP compliance |
| Different behaviors needed in different contexts | Configure context with appropriate strategy |
| Testing individual algorithms in isolation | Each strategy is independently testable |

**Strategy vs Template Method:**

| Aspect | Strategy | Template Method |
|--------|----------|-----------------|
| Mechanism | Composition (has-a) | Inheritance (is-a) |
| Granularity | Entire algorithm is swapped | Only steps of algorithm vary |
| Runtime flexibility | Can change at runtime | Fixed at compile time |
| Class count | More classes (one per strategy) | Fewer classes (subclasses) |
| Coupling | Loose (interface-based) | Tight (inheritance-based) |

---


## 6.2 Observer Pattern

> The Observer pattern defines a one-to-many dependency between objects so that when one object (the Subject) changes state, all its dependents (Observers) are notified and updated automatically. It's the foundation of event-driven programming and the publish-subscribe model.

---

### Why Observer?

When one object's state change should trigger updates in other objects, but you don't want tight coupling between them:
- Stock price changes → update portfolio display, trigger alerts, log history
- User posts a message → notify followers, update feed, send push notifications
- Temperature sensor reading → update display, check thresholds, log data
- Order status changes → notify customer, update dashboard, trigger shipping

**The problem without Observer:**
```cpp
// BAD: Tight coupling — WeatherStation knows about every display
class WeatherStation {
    TemperatureDisplay* tempDisplay;
    HumidityDisplay* humidDisplay;
    PressureDisplay* pressDisplay;
    ForecastDisplay* forecastDisplay;
    MobileApp* mobileApp;
    WebDashboard* webDashboard;
    // ... every new display requires modifying this class!

public:
    void setMeasurements(float temp, float humidity, float pressure) {
        // Must update ALL displays manually — tightly coupled!
        tempDisplay->update(temp);
        humidDisplay->update(humidity);
        pressDisplay->update(pressure);
        forecastDisplay->update(temp, humidity, pressure);
        mobileApp->refresh(temp, humidity, pressure);
        webDashboard->refresh(temp, humidity, pressure);
        // Adding a new display means modifying this class — violates OCP!
    }
};
```

**Problems:**
1. WeatherStation is coupled to every concrete display type
2. Adding/removing displays requires modifying WeatherStation
3. Can't add displays at runtime
4. Violates Open/Closed Principle and Single Responsibility Principle

---

### Structure

```
┌──────────────────────────┐       ┌──────────────────────────┐
│       Subject            │       │     Observer (interface)  │
├──────────────────────────┤       ├──────────────────────────┤
│ - observers: list<Obs*>  │       │ + update(data)           │
├──────────────────────────┤       └──────────────────────────┘
│ + attach(observer)       │                    ▲
│ + detach(observer)       │                    │
│ + notify()               │         ┌──────────┼──────────┐
└──────────────────────────┘         │          │          │
         │ notifies                Display   Logger    AlertSystem
         └──────────────────────→  + update() + update() + update()
```

**Participants:**
- **Subject (Observable):** Maintains a list of observers and notifies them of state changes
- **Observer:** Defines an interface for objects that should be notified
- **ConcreteSubject:** Stores state of interest; sends notifications when state changes
- **ConcreteObserver:** Implements the Observer interface to react to updates

---

### Push Model Implementation

In the **push model**, the Subject sends detailed data to observers in the notification. Observers receive all the data they might need.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <memory>
using namespace std;

// --- Observer Interface ---
class WeatherObserver {
public:
    virtual void update(float temperature, float humidity, float pressure) = 0;
    virtual string getName() const = 0;
    virtual ~WeatherObserver() = default;
};

// --- Subject (Observable) ---
class WeatherStation {
    vector<WeatherObserver*> observers;
    float temperature = 0;
    float humidity = 0;
    float pressure = 0;

public:
    void attach(WeatherObserver* observer) {
        observers.push_back(observer);
        cout << "[WeatherStation] " << observer->getName() << " subscribed." << endl;
    }

    void detach(WeatherObserver* observer) {
        observers.erase(
            remove(observers.begin(), observers.end(), observer),
            observers.end()
        );
        cout << "[WeatherStation] " << observer->getName() << " unsubscribed." << endl;
    }

    void notify() {
        cout << "[WeatherStation] Notifying " << observers.size() << " observers..." << endl;
        for (auto* observer : observers) {
            observer->update(temperature, humidity, pressure);
        }
    }

    void setMeasurements(float temp, float hum, float press) {
        cout << "\n[WeatherStation] New measurements: "
             << "temp=" << temp << "°C, humidity=" << hum
             << "%, pressure=" << press << " hPa" << endl;
        temperature = temp;
        humidity = hum;
        pressure = press;
        notify();  // automatically notify all observers
    }

    // Getters for pull model
    float getTemperature() const { return temperature; }
    float getHumidity() const { return humidity; }
    float getPressure() const { return pressure; }
};

// --- Concrete Observers ---
class TemperatureDisplay : public WeatherObserver {
public:
    void update(float temperature, float humidity, float pressure) override {
        cout << "  [TempDisplay] Current temperature: " << temperature << "°C" << endl;
    }
    string getName() const override { return "TemperatureDisplay"; }
};

class StatisticsDisplay : public WeatherObserver {
    float minTemp = 999, maxTemp = -999, sumTemp = 0;
    int numReadings = 0;
public:
    void update(float temperature, float humidity, float pressure) override {
        sumTemp += temperature;
        numReadings++;
        minTemp = min(minTemp, temperature);
        maxTemp = max(maxTemp, temperature);
        cout << "  [StatsDisplay] Avg/Min/Max: "
             << (sumTemp / numReadings) << "/" << minTemp << "/" << maxTemp << "°C" << endl;
    }
    string getName() const override { return "StatisticsDisplay"; }
};

class AlertSystem : public WeatherObserver {
    float tempThreshold;
public:
    AlertSystem(float threshold) : tempThreshold(threshold) {}

    void update(float temperature, float humidity, float pressure) override {
        if (temperature > tempThreshold) {
            cout << "  [ALERT] Temperature " << temperature
                 << "°C exceeds threshold " << tempThreshold << "°C!" << endl;
        }
        if (humidity > 90) {
            cout << "  [ALERT] High humidity: " << humidity << "%!" << endl;
        }
    }
    string getName() const override { return "AlertSystem"; }
};

class ForecastDisplay : public WeatherObserver {
    float lastPressure = 0;
public:
    void update(float temperature, float humidity, float pressure) override {
        string forecast;
        if (pressure > lastPressure) {
            forecast = "Improving weather on the way!";
        } else if (pressure < lastPressure) {
            forecast = "Watch out for cooler, rainy weather.";
        } else {
            forecast = "More of the same.";
        }
        cout << "  [Forecast] " << forecast << endl;
        lastPressure = pressure;
    }
    string getName() const override { return "ForecastDisplay"; }
};

// --- Client code ---
int main() {
    WeatherStation station;

    TemperatureDisplay tempDisplay;
    StatisticsDisplay statsDisplay;
    AlertSystem alertSystem(35.0);
    ForecastDisplay forecastDisplay;

    // Subscribe observers
    station.attach(&tempDisplay);
    station.attach(&statsDisplay);
    station.attach(&alertSystem);
    station.attach(&forecastDisplay);

    // Simulate weather changes
    station.setMeasurements(28.5, 65.0, 1013.0);
    station.setMeasurements(32.0, 70.0, 1010.0);
    station.setMeasurements(37.0, 92.0, 1007.0);  // triggers alerts!

    // Unsubscribe an observer
    station.detach(&tempDisplay);
    station.setMeasurements(25.0, 55.0, 1015.0);  // tempDisplay won't be notified

    return 0;
}
```

---

### Pull Model Implementation

In the **pull model**, the Subject sends minimal notification (or none), and observers pull the data they need from the Subject.

```cpp
// --- Observer Interface (Pull Model) ---
class Observer {
public:
    virtual void update() = 0;  // no data passed — observer pulls what it needs
    virtual ~Observer() = default;
};

// --- Subject with Pull Model ---
class StockMarket {
    vector<Observer*> observers;
    unordered_map<string, double> prices;

public:
    void attach(Observer* obs) { observers.push_back(obs); }
    void detach(Observer* obs) {
        observers.erase(remove(observers.begin(), observers.end(), obs), observers.end());
    }

    void notify() {
        for (auto* obs : observers) {
            obs->update();  // just a signal — no data
        }
    }

    void setPrice(const string& symbol, double price) {
        prices[symbol] = price;
        notify();
    }

    // Observers call these to pull data they need
    double getPrice(const string& symbol) const {
        auto it = prices.find(symbol);
        return (it != prices.end()) ? it->second : 0.0;
    }

    vector<string> getSymbols() const {
        vector<string> symbols;
        for (const auto& [sym, _] : prices) symbols.push_back(sym);
        return symbols;
    }
};

// --- Concrete Observer (Pull Model) ---
class PortfolioTracker : public Observer {
    StockMarket& market;  // reference to subject — for pulling data
    unordered_map<string, int> holdings;  // symbol → quantity

public:
    PortfolioTracker(StockMarket& m) : market(m) {}

    void addHolding(const string& symbol, int qty) {
        holdings[symbol] = qty;
    }

    void update() override {
        // Pull only the data we care about
        double totalValue = 0;
        for (const auto& [symbol, qty] : holdings) {
            double price = market.getPrice(symbol);  // PULL from subject
            totalValue += price * qty;
        }
        cout << "[Portfolio] Total value: $" << totalValue << endl;
    }
};

class PriceAlertObserver : public Observer {
    StockMarket& market;
    string watchSymbol;
    double alertPrice;

public:
    PriceAlertObserver(StockMarket& m, const string& symbol, double price)
        : market(m), watchSymbol(symbol), alertPrice(price) {}

    void update() override {
        double currentPrice = market.getPrice(watchSymbol);  // PULL
        if (currentPrice >= alertPrice) {
            cout << "[ALERT] " << watchSymbol << " reached $"
                 << currentPrice << " (target: $" << alertPrice << ")" << endl;
        }
    }
};
```

---

### Push vs Pull Model

| Aspect | Push Model | Pull Model |
|--------|-----------|------------|
| Data flow | Subject pushes data to observers | Observers pull data from subject |
| Observer interface | `update(data1, data2, ...)` | `update()` — no parameters |
| Coupling | Observer coupled to data format | Observer coupled to Subject interface |
| Efficiency | May send unnecessary data | Observer fetches only what it needs |
| Flexibility | Subject decides what to send | Observer decides what to read |
| Complexity | Simpler observer implementation | Observer needs reference to Subject |
| Best for | All observers need same data | Observers need different subsets of data |

---

### Event System Implementation (Generic Observer)

A more flexible, type-safe event system using templates:

```cpp
#include <iostream>
#include <functional>
#include <unordered_map>
#include <vector>
#include <string>
#include <algorithm>
using namespace std;

// --- Generic Event System ---
class EventEmitter {
public:
    using EventHandler = function<void(const string& eventData)>;
    using HandlerId = size_t;

private:
    struct HandlerEntry {
        HandlerId id;
        EventHandler handler;
    };

    unordered_map<string, vector<HandlerEntry>> listeners;
    HandlerId nextId = 0;

public:
    // Subscribe to an event — returns handler ID for unsubscribing
    HandlerId on(const string& event, EventHandler handler) {
        HandlerId id = nextId++;
        listeners[event].push_back({id, handler});
        return id;
    }

    // Subscribe for one-time notification
    HandlerId once(const string& event, EventHandler handler) {
        HandlerId id = nextId++;
        // Wrap handler to auto-remove after first call
        listeners[event].push_back({id, [this, event, id, handler](const string& data) {
            handler(data);
            off(event, id);
        }});
        return id;
    }

    // Unsubscribe
    void off(const string& event, HandlerId id) {
        auto it = listeners.find(event);
        if (it != listeners.end()) {
            auto& handlers = it->second;
            handlers.erase(
                remove_if(handlers.begin(), handlers.end(),
                    [id](const HandlerEntry& entry) { return entry.id == id; }),
                handlers.end()
            );
        }
    }

    // Emit event to all subscribers
    void emit(const string& event, const string& data = "") {
        auto it = listeners.find(event);
        if (it != listeners.end()) {
            // Copy handlers in case they modify the list during iteration
            auto handlers = it->second;
            for (const auto& entry : handlers) {
                entry.handler(data);
            }
        }
    }

    // Get subscriber count for an event
    size_t listenerCount(const string& event) const {
        auto it = listeners.find(event);
        return (it != listeners.end()) ? it->second.size() : 0;
    }
};

// --- Usage ---
int main() {
    EventEmitter emitter;

    // Subscribe to events
    auto id1 = emitter.on("user:login", [](const string& data) {
        cout << "Logger: User logged in — " << data << endl;
    });

    auto id2 = emitter.on("user:login", [](const string& data) {
        cout << "Analytics: Track login event — " << data << endl;
    });

    emitter.on("user:logout", [](const string& data) {
        cout << "Cleanup: Session ended — " << data << endl;
    });

    // One-time handler
    emitter.once("user:login", [](const string& data) {
        cout << "Welcome bonus: First login detected! — " << data << endl;
    });

    // Emit events
    cout << "--- First login ---" << endl;
    emitter.emit("user:login", "user123");

    cout << "\n--- Second login ---" << endl;
    emitter.emit("user:login", "user456");  // "once" handler won't fire

    // Unsubscribe
    emitter.off("user:login", id1);
    cout << "\n--- Third login (after unsubscribe) ---" << endl;
    emitter.emit("user:login", "user789");  // only id2 handler fires

    return 0;
}
```

---

### When to Use Observer

| Scenario | Why Observer Helps |
|----------|-------------------|
| State changes should trigger updates in other objects | Decouples subject from observers |
| Number of dependent objects is unknown or changes dynamically | Observers can subscribe/unsubscribe at runtime |
| Broadcasting events to multiple listeners | One-to-many notification |
| Building event-driven systems | Foundation for event buses, reactive programming |
| Implementing MVC architecture | Model notifies Views of changes |
| Loose coupling between components | Subject doesn't know concrete observer types |

**Pitfalls to watch for:**
- **Memory leaks:** Observers that forget to unsubscribe (dangling references)
- **Notification order:** Observers are notified in subscription order — don't depend on it
- **Cascading updates:** Observer A's update triggers Subject B, which notifies Observer C... can cause infinite loops
- **Performance:** Many observers or frequent notifications can be expensive
- **Thread safety:** Concurrent attach/detach/notify needs synchronization

---


## 6.3 Command Pattern

> The Command pattern encapsulates a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations. It turns a method call into a stand-alone object that contains all information about the request.

---

### Why Command?

When you need to decouple the object that invokes an operation from the object that performs it:
- Text editor: undo/redo operations
- GUI: buttons, menus, shortcuts all trigger the same action
- Task scheduling: queue commands for later execution
- Transaction logging: record operations for replay
- Macro recording: store a sequence of commands

**The problem without Command:**
```cpp
// BAD: Button is tightly coupled to specific actions
class Button {
public:
    void click() {
        // What does this button do? It's hardcoded!
        document.save();  // Can't reuse Button for other actions
    }
};

// BAD: No way to undo operations
class TextEditor {
public:
    void bold() { /* apply bold */ }
    void italic() { /* apply italic */ }
    void deleteText() { /* delete selected text — how to undo? */ }
    // No history, no undo, no redo
};
```

---

### Structure

```
┌──────────────┐     ┌──────────────────────┐     ┌──────────────┐
│   Invoker     │     │   Command (interface) │     │   Receiver    │
├──────────────┤     ├──────────────────────┤     ├──────────────┤
│ - command    │────→│ + execute()           │────→│ + action()    │
├──────────────┤     │ + undo()              │     └──────────────┘
│ + invoke()   │     └──────────────────────┘
└──────────────┘              ▲
                              │
                   ┌──────────┼──────────┐
                   │          │          │
              CopyCommand  PasteCmd   DeleteCmd
              + execute()  + execute() + execute()
              + undo()     + undo()    + undo()
```

**Participants:**
- **Command (interface):** Declares `execute()` and optionally `undo()`
- **ConcreteCommand:** Binds a Receiver to an action; implements `execute()` by calling Receiver methods
- **Invoker:** Asks the command to carry out the request
- **Receiver:** Knows how to perform the actual work
- **Client:** Creates ConcreteCommand and sets its Receiver

---

### Basic Implementation — Text Editor with Undo/Redo

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <stack>
#include <memory>
using namespace std;

// --- Receiver ---
class TextDocument {
    string content;
public:
    void insertText(int position, const string& text) {
        if (position < 0 || position > (int)content.size()) {
            position = content.size();
        }
        content.insert(position, text);
    }

    void deleteText(int position, int length) {
        if (position >= 0 && position < (int)content.size()) {
            content.erase(position, length);
        }
    }

    string getText() const { return content; }
    int length() const { return content.size(); }

    void print() const {
        cout << "Document: \"" << content << "\"" << endl;
    }
};

// --- Command Interface ---
class Command {
public:
    virtual void execute() = 0;
    virtual void undo() = 0;
    virtual string description() const = 0;
    virtual ~Command() = default;
};

// --- Concrete Commands ---
class InsertCommand : public Command {
    TextDocument& document;
    int position;
    string text;
public:
    InsertCommand(TextDocument& doc, int pos, const string& t)
        : document(doc), position(pos), text(t) {}

    void execute() override {
        document.insertText(position, text);
    }

    void undo() override {
        document.deleteText(position, text.length());
    }

    string description() const override {
        return "Insert \"" + text + "\" at position " + to_string(position);
    }
};

class DeleteCommand : public Command {
    TextDocument& document;
    int position;
    int length;
    string deletedText;  // saved for undo
public:
    DeleteCommand(TextDocument& doc, int pos, int len)
        : document(doc), position(pos), length(len) {}

    void execute() override {
        // Save deleted text for undo
        deletedText = document.getText().substr(position, length);
        document.deleteText(position, length);
    }

    void undo() override {
        document.insertText(position, deletedText);
    }

    string description() const override {
        return "Delete " + to_string(length) + " chars at position " + to_string(position);
    }
};

class ReplaceCommand : public Command {
    TextDocument& document;
    int position;
    int length;
    string newText;
    string oldText;  // saved for undo
public:
    ReplaceCommand(TextDocument& doc, int pos, int len, const string& replacement)
        : document(doc), position(pos), length(len), newText(replacement) {}

    void execute() override {
        oldText = document.getText().substr(position, length);
        document.deleteText(position, length);
        document.insertText(position, newText);
    }

    void undo() override {
        document.deleteText(position, newText.length());
        document.insertText(position, oldText);
    }

    string description() const override {
        return "Replace \"" + oldText + "\" with \"" + newText + "\"";
    }
};

// --- Invoker with Undo/Redo ---
class TextEditor {
    TextDocument& document;
    stack<unique_ptr<Command>> undoStack;
    stack<unique_ptr<Command>> redoStack;

public:
    TextEditor(TextDocument& doc) : document(doc) {}

    void executeCommand(unique_ptr<Command> cmd) {
        cout << "Execute: " << cmd->description() << endl;
        cmd->execute();
        undoStack.push(std::move(cmd));
        // Clear redo stack — new action invalidates redo history
        while (!redoStack.empty()) redoStack.pop();
        document.print();
    }

    void undo() {
        if (undoStack.empty()) {
            cout << "Nothing to undo!" << endl;
            return;
        }
        auto cmd = std::move(undoStack.top());
        undoStack.pop();
        cout << "Undo: " << cmd->description() << endl;
        cmd->undo();
        redoStack.push(std::move(cmd));
        document.print();
    }

    void redo() {
        if (redoStack.empty()) {
            cout << "Nothing to redo!" << endl;
            return;
        }
        auto cmd = std::move(redoStack.top());
        redoStack.pop();
        cout << "Redo: " << cmd->description() << endl;
        cmd->execute();
        undoStack.push(std::move(cmd));
        document.print();
    }

    int undoCount() const { return undoStack.size(); }
    int redoCount() const { return redoStack.size(); }
};

// --- Client code ---
int main() {
    TextDocument doc;
    TextEditor editor(doc);

    editor.executeCommand(make_unique<InsertCommand>(doc, 0, "Hello"));
    editor.executeCommand(make_unique<InsertCommand>(doc, 5, " World"));
    editor.executeCommand(make_unique<InsertCommand>(doc, 11, "!"));
    // Document: "Hello World!"

    editor.undo();  // removes "!"
    editor.undo();  // removes " World"
    // Document: "Hello"

    editor.redo();  // re-adds " World"
    // Document: "Hello World"

    editor.executeCommand(make_unique<ReplaceCommand>(doc, 6, 5, "C++"));
    // Document: "Hello C++!"  — wait, no "!" since we undid it
    // Document: "Hello C++"

    editor.undo();  // undo replace
    // Document: "Hello World"

    return 0;
}
```

---

### Macro Commands (Composite Command)

A macro command is a command that contains a sequence of other commands, executed as a single unit.

```cpp
class MacroCommand : public Command {
    vector<unique_ptr<Command>> commands;
    string macroName;

public:
    MacroCommand(const string& name) : macroName(name) {}

    void addCommand(unique_ptr<Command> cmd) {
        commands.push_back(std::move(cmd));
    }

    void execute() override {
        cout << "--- Executing macro: " << macroName << " ---" << endl;
        for (auto& cmd : commands) {
            cmd->execute();
        }
    }

    void undo() override {
        cout << "--- Undoing macro: " << macroName << " ---" << endl;
        // Undo in reverse order!
        for (int i = commands.size() - 1; i >= 0; i--) {
            commands[i]->undo();
        }
    }

    string description() const override {
        return "Macro: " + macroName + " (" + to_string(commands.size()) + " commands)";
    }
};

// Usage
int main() {
    TextDocument doc;
    TextEditor editor(doc);

    // Create a macro that sets up a document template
    auto templateMacro = make_unique<MacroCommand>("Document Template");
    templateMacro->addCommand(make_unique<InsertCommand>(doc, 0, "Title: "));
    templateMacro->addCommand(make_unique<InsertCommand>(doc, 7, "Untitled"));
    templateMacro->addCommand(make_unique<InsertCommand>(doc, 15, "\n\nBody: "));
    templateMacro->addCommand(make_unique<InsertCommand>(doc, 23, "Enter text here..."));

    editor.executeCommand(std::move(templateMacro));
    // Document: "Title: Untitled\n\nBody: Enter text here..."

    editor.undo();  // undoes the ENTIRE macro in one step
    // Document: ""

    return 0;
}
```

---

### Command Queue (Deferred Execution)

Commands can be queued for later execution, enabling task scheduling and batch processing.

```cpp
#include <queue>
#include <thread>
#include <chrono>

class CommandQueue {
    queue<unique_ptr<Command>> pendingCommands;
    vector<unique_ptr<Command>> executedCommands;  // history

public:
    void enqueue(unique_ptr<Command> cmd) {
        cout << "Queued: " << cmd->description() << endl;
        pendingCommands.push(std::move(cmd));
    }

    void processNext() {
        if (pendingCommands.empty()) {
            cout << "Queue is empty!" << endl;
            return;
        }
        auto cmd = std::move(pendingCommands.front());
        pendingCommands.pop();
        cout << "Processing: " << cmd->description() << endl;
        cmd->execute();
        executedCommands.push_back(std::move(cmd));
    }

    void processAll() {
        cout << "Processing " << pendingCommands.size() << " commands..." << endl;
        while (!pendingCommands.empty()) {
            processNext();
        }
        cout << "All commands processed." << endl;
    }

    int pendingCount() const { return pendingCommands.size(); }
    int executedCount() const { return executedCommands.size(); }

    // Undo last executed command
    void undoLast() {
        if (executedCommands.empty()) {
            cout << "No commands to undo!" << endl;
            return;
        }
        auto& cmd = executedCommands.back();
        cout << "Undoing: " << cmd->description() << endl;
        cmd->undo();
        executedCommands.pop_back();
    }
};

// Usage
int main() {
    TextDocument doc;
    CommandQueue queue;

    // Queue up commands (not executed yet)
    queue.enqueue(make_unique<InsertCommand>(doc, 0, "First "));
    queue.enqueue(make_unique<InsertCommand>(doc, 6, "Second "));
    queue.enqueue(make_unique<InsertCommand>(doc, 13, "Third"));

    cout << "Pending: " << queue.pendingCount() << endl;

    // Process all at once
    queue.processAll();
    doc.print();  // "First Second Third"

    // Undo last
    queue.undoLast();
    doc.print();  // "First Second "

    return 0;
}
```

---

### Real-World Example — Remote Control (Home Automation)

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <string>
using namespace std;

// --- Receivers ---
class Light {
    string location;
    int brightness = 0;
public:
    Light(const string& loc) : location(loc) {}
    void on() { brightness = 100; cout << location << " light ON (100%)" << endl; }
    void off() { brightness = 0; cout << location << " light OFF" << endl; }
    void dim(int level) { brightness = level; cout << location << " light dimmed to " << level << "%" << endl; }
    int getBrightness() const { return brightness; }
    string getLocation() const { return location; }
};

class Thermostat {
    double temperature = 20.0;
public:
    void setTemperature(double temp) {
        cout << "Thermostat set to " << temp << "°C" << endl;
        temperature = temp;
    }
    double getTemperature() const { return temperature; }
};

class MusicPlayer {
    string currentSong;
    int volume = 50;
    bool playing = false;
public:
    void play(const string& song) {
        currentSong = song;
        playing = true;
        cout << "Playing: " << song << " (volume: " << volume << ")" << endl;
    }
    void stop() {
        playing = false;
        cout << "Music stopped" << endl;
    }
    void setVolume(int vol) {
        volume = vol;
        cout << "Volume set to " << volume << endl;
    }
    bool isPlaying() const { return playing; }
    int getVolume() const { return volume; }
};

// --- Command Interface ---
class SmartCommand {
public:
    virtual void execute() = 0;
    virtual void undo() = 0;
    virtual string description() const = 0;
    virtual ~SmartCommand() = default;
};

// --- Concrete Commands ---
class LightOnCommand : public SmartCommand {
    Light& light;
    int previousBrightness;
public:
    LightOnCommand(Light& l) : light(l), previousBrightness(0) {}
    void execute() override {
        previousBrightness = light.getBrightness();
        light.on();
    }
    void undo() override {
        if (previousBrightness == 0) light.off();
        else light.dim(previousBrightness);
    }
    string description() const override { return light.getLocation() + " Light ON"; }
};

class LightOffCommand : public SmartCommand {
    Light& light;
    int previousBrightness;
public:
    LightOffCommand(Light& l) : light(l), previousBrightness(0) {}
    void execute() override {
        previousBrightness = light.getBrightness();
        light.off();
    }
    void undo() override {
        if (previousBrightness > 0) light.dim(previousBrightness);
        else light.on();
    }
    string description() const override { return light.getLocation() + " Light OFF"; }
};

class ThermostatCommand : public SmartCommand {
    Thermostat& thermostat;
    double newTemp;
    double previousTemp;
public:
    ThermostatCommand(Thermostat& t, double temp)
        : thermostat(t), newTemp(temp), previousTemp(0) {}
    void execute() override {
        previousTemp = thermostat.getTemperature();
        thermostat.setTemperature(newTemp);
    }
    void undo() override {
        thermostat.setTemperature(previousTemp);
    }
    string description() const override {
        return "Set thermostat to " + to_string((int)newTemp) + "°C";
    }
};

// --- Invoker: Remote Control ---
class RemoteControl {
    static const int NUM_SLOTS = 5;
    unique_ptr<SmartCommand> onCommands[NUM_SLOTS];
    unique_ptr<SmartCommand> offCommands[NUM_SLOTS];
    unique_ptr<SmartCommand> lastCommand;

public:
    void setCommand(int slot, unique_ptr<SmartCommand> onCmd, unique_ptr<SmartCommand> offCmd) {
        if (slot >= 0 && slot < NUM_SLOTS) {
            onCommands[slot] = std::move(onCmd);
            offCommands[slot] = std::move(offCmd);
        }
    }

    void pressOn(int slot) {
        if (slot >= 0 && slot < NUM_SLOTS && onCommands[slot]) {
            cout << "[Slot " << slot << " ON] ";
            onCommands[slot]->execute();
            // Clone for undo (simplified — store reference)
        }
    }

    void pressOff(int slot) {
        if (slot >= 0 && slot < NUM_SLOTS && offCommands[slot]) {
            cout << "[Slot " << slot << " OFF] ";
            offCommands[slot]->execute();
        }
    }

    void printSlots() const {
        cout << "\n--- Remote Control ---" << endl;
        for (int i = 0; i < NUM_SLOTS; i++) {
            cout << "Slot " << i << ": "
                 << (onCommands[i] ? onCommands[i]->description() : "---")
                 << " / "
                 << (offCommands[i] ? offCommands[i]->description() : "---")
                 << endl;
        }
    }
};

// --- Client code ---
int main() {
    Light livingRoom("Living Room");
    Light bedroom("Bedroom");
    Thermostat thermostat;

    RemoteControl remote;

    remote.setCommand(0,
        make_unique<LightOnCommand>(livingRoom),
        make_unique<LightOffCommand>(livingRoom));

    remote.setCommand(1,
        make_unique<LightOnCommand>(bedroom),
        make_unique<LightOffCommand>(bedroom));

    remote.setCommand(2,
        make_unique<ThermostatCommand>(thermostat, 24.0),
        make_unique<ThermostatCommand>(thermostat, 20.0));

    remote.printSlots();

    cout << endl;
    remote.pressOn(0);   // Living Room light ON
    remote.pressOn(1);   // Bedroom light ON
    remote.pressOn(2);   // Thermostat to 24°C
    remote.pressOff(0);  // Living Room light OFF

    return 0;
}
```

---

### When to Use Command

| Scenario | Why Command Helps |
|----------|------------------|
| Undo/Redo functionality | Commands store state needed to reverse operations |
| Queuing and scheduling operations | Commands are objects that can be stored and executed later |
| Logging and auditing | Commands can be serialized and logged |
| Macro recording | Composite commands store sequences |
| Callback abstraction | Commands encapsulate callbacks as objects |
| Transactional behavior | Execute a batch; undo all if one fails |
| Decoupling invoker from receiver | Invoker doesn't know what action is performed |

**Command vs Strategy:**

| Aspect | Command | Strategy |
|--------|---------|----------|
| Intent | Encapsulate a request as an object | Encapsulate an algorithm |
| Undo support | Yes (stores state for reversal) | No (stateless algorithm swap) |
| History | Commands can be stored, queued, logged | Strategies are typically not stored |
| Receiver | Command knows its receiver | Strategy doesn't have a receiver |
| Granularity | Single operation | Entire algorithm |

---


## 6.4 State Pattern

> The State pattern allows an object to alter its behavior when its internal state changes. The object will appear to change its class. It encapsulates state-specific behavior into separate state objects and delegates behavior to the current state object.

---

### Why State?

When an object's behavior depends on its state and must change at runtime based on that state:
- Vending machine: idle → has money → dispensing → out of stock
- TCP connection: closed → listening → established → closing
- Document: draft → review → approved → published
- Player character: standing → running → jumping → falling

**The problem without State:**
```cpp
// BAD: Massive conditional logic based on state
class VendingMachine {
    enum State { IDLE, HAS_MONEY, DISPENSING, OUT_OF_STOCK };
    State currentState = IDLE;

public:
    void insertMoney(double amount) {
        if (currentState == IDLE) {
            balance += amount;
            currentState = HAS_MONEY;
        } else if (currentState == HAS_MONEY) {
            balance += amount;
        } else if (currentState == DISPENSING) {
            cout << "Please wait, dispensing..." << endl;
        } else if (currentState == OUT_OF_STOCK) {
            cout << "Machine is out of stock, returning money" << endl;
        }
        // Every method has this same switch/if-else for EVERY state!
    }

    void selectProduct(int id) {
        if (currentState == IDLE) { /* ... */ }
        else if (currentState == HAS_MONEY) { /* ... */ }
        else if (currentState == DISPENSING) { /* ... */ }
        else if (currentState == OUT_OF_STOCK) { /* ... */ }
        // Duplicated state checks everywhere!
    }

    // Adding a new state means modifying EVERY method!
};
```

**Problems:**
1. State logic scattered across all methods
2. Adding a new state requires modifying every method
3. Hard to understand which behaviors belong to which state
4. Violates Open/Closed Principle and Single Responsibility Principle

---

### Structure

```
┌──────────────────────┐       ┌──────────────────────────┐
│      Context          │       │     State (interface)     │
├──────────────────────┤       ├──────────────────────────┤
│ - state: State*       │──────→│ + handleInsertMoney()     │
├──────────────────────┤       │ + handleSelectProduct()   │
│ + setState(s)         │       │ + handleDispense()        │
│ + insertMoney()       │       └──────────────────────────┘
│ + selectProduct()     │                    ▲
│ + dispense()          │                    │
└──────────────────────┘       ┌─────────────┼─────────────┐
                               │             │             │
                          IdleState    HasMoneyState  DispensingState
```

**Participants:**
- **State (interface):** Defines interface for state-specific behavior
- **ConcreteState:** Implements behavior for a particular state; may trigger state transitions
- **Context:** Maintains a reference to the current state; delegates requests to it

---

### Vending Machine Example

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <unordered_map>
using namespace std;

// Forward declaration
class VendingMachine;

// --- State Interface ---
class VendingState {
public:
    virtual void insertMoney(VendingMachine& machine, double amount) = 0;
    virtual void selectProduct(VendingMachine& machine, int productId) = 0;
    virtual void dispense(VendingMachine& machine) = 0;
    virtual void cancelTransaction(VendingMachine& machine) = 0;
    virtual string getName() const = 0;
    virtual ~VendingState() = default;
};

// --- Context ---
class VendingMachine {
    unique_ptr<VendingState> currentState;
    double balance = 0;

    struct Product {
        string name;
        double price;
        int quantity;
    };
    unordered_map<int, Product> products;
    int selectedProduct = -1;

public:
    VendingMachine();  // defined after state classes

    void setState(unique_ptr<VendingState> state) {
        cout << "  [State: " << currentState->getName()
             << " → " << state->getName() << "]" << endl;
        currentState = std::move(state);
    }

    // Delegate to current state
    void insertMoney(double amount) {
        cout << "\n> Insert $" << amount << endl;
        currentState->insertMoney(*this, amount);
    }

    void selectProduct(int id) {
        cout << "\n> Select product #" << id << endl;
        currentState->selectProduct(*this, id);
    }

    void dispense() {
        currentState->dispense(*this);
    }

    void cancelTransaction() {
        cout << "\n> Cancel transaction" << endl;
        currentState->cancelTransaction(*this);
    }

    // Internal methods used by states
    void addBalance(double amount) { balance += amount; }
    double getBalance() const { return balance; }
    void resetBalance() { balance = 0; }

    void addProduct(int id, const string& name, double price, int qty) {
        products[id] = {name, price, qty};
    }

    bool hasProduct(int id) const {
        auto it = products.find(id);
        return it != products.end() && it->second.quantity > 0;
    }

    Product getProduct(int id) const { return products.at(id); }

    void decrementProduct(int id) {
        if (products.count(id)) products[id].quantity--;
    }

    void setSelectedProduct(int id) { selectedProduct = id; }
    int getSelectedProduct() const { return selectedProduct; }

    bool hasAnyProducts() const {
        for (const auto& [id, prod] : products) {
            if (prod.quantity > 0) return true;
        }
        return false;
    }

    void printStatus() const {
        cout << "  Balance: $" << balance
             << " | State: " << currentState->getName() << endl;
    }
};

// --- Concrete States ---
class IdleState : public VendingState {
public:
    void insertMoney(VendingMachine& machine, double amount) override {
        machine.addBalance(amount);
        cout << "  Money accepted. Balance: $" << machine.getBalance() << endl;
        machine.setState(make_unique<class HasMoneyState>());
    }

    void selectProduct(VendingMachine& machine, int productId) override {
        cout << "  Please insert money first." << endl;
    }

    void dispense(VendingMachine& machine) override {
        cout << "  Please insert money and select a product." << endl;
    }

    void cancelTransaction(VendingMachine& machine) override {
        cout << "  No transaction to cancel." << endl;
    }

    string getName() const override { return "Idle"; }
};

class HasMoneyState : public VendingState {
public:
    void insertMoney(VendingMachine& machine, double amount) override {
        machine.addBalance(amount);
        cout << "  Additional money accepted. Balance: $" << machine.getBalance() << endl;
    }

    void selectProduct(VendingMachine& machine, int productId) override {
        if (!machine.hasProduct(productId)) {
            cout << "  Product #" << productId << " is out of stock." << endl;
            return;
        }

        auto product = machine.getProduct(productId);
        if (machine.getBalance() < product.price) {
            cout << "  Insufficient funds. " << product.name << " costs $"
                 << product.price << ", balance: $" << machine.getBalance() << endl;
            return;
        }

        machine.setSelectedProduct(productId);
        cout << "  Selected: " << product.name << " ($" << product.price << ")" << endl;
        machine.setState(make_unique<class DispensingState>());
        machine.dispense();  // auto-dispense after selection
    }

    void dispense(VendingMachine& machine) override {
        cout << "  Please select a product first." << endl;
    }

    void cancelTransaction(VendingMachine& machine) override {
        cout << "  Transaction cancelled. Returning $" << machine.getBalance() << endl;
        machine.resetBalance();
        machine.setState(make_unique<IdleState>());
    }

    string getName() const override { return "HasMoney"; }
};

class DispensingState : public VendingState {
public:
    void insertMoney(VendingMachine& machine, double amount) override {
        cout << "  Please wait, dispensing product..." << endl;
    }

    void selectProduct(VendingMachine& machine, int productId) override {
        cout << "  Please wait, dispensing product..." << endl;
    }

    void dispense(VendingMachine& machine) override {
        int productId = machine.getSelectedProduct();
        auto product = machine.getProduct(productId);

        machine.decrementProduct(productId);
        double change = machine.getBalance() - product.price;
        machine.resetBalance();

        cout << "  *** Dispensing: " << product.name << " ***" << endl;
        if (change > 0) {
            cout << "  Returning change: $" << change << endl;
        }

        if (machine.hasAnyProducts()) {
            machine.setState(make_unique<IdleState>());
        } else {
            machine.setState(make_unique<class OutOfStockState>());
        }
    }

    void cancelTransaction(VendingMachine& machine) override {
        cout << "  Cannot cancel, already dispensing." << endl;
    }

    string getName() const override { return "Dispensing"; }
};

class OutOfStockState : public VendingState {
public:
    void insertMoney(VendingMachine& machine, double amount) override {
        cout << "  Machine is out of stock. Returning $" << amount << endl;
    }

    void selectProduct(VendingMachine& machine, int productId) override {
        cout << "  Machine is out of stock." << endl;
    }

    void dispense(VendingMachine& machine) override {
        cout << "  Machine is out of stock." << endl;
    }

    void cancelTransaction(VendingMachine& machine) override {
        cout << "  No transaction in progress." << endl;
    }

    string getName() const override { return "OutOfStock"; }
};

// --- VendingMachine constructor (defined after state classes) ---
VendingMachine::VendingMachine() : currentState(make_unique<IdleState>()) {}

// --- Client code ---
int main() {
    VendingMachine machine;
    machine.addProduct(1, "Cola", 1.50, 2);
    machine.addProduct(2, "Chips", 2.00, 1);
    machine.addProduct(3, "Water", 1.00, 1);

    // Normal purchase
    machine.insertMoney(2.00);
    machine.selectProduct(1);  // Cola $1.50 — get $0.50 change

    // Insufficient funds
    machine.insertMoney(1.00);
    machine.selectProduct(2);  // Chips $2.00 — not enough!
    machine.insertMoney(1.00);
    machine.selectProduct(2);  // Now enough

    // Cancel transaction
    machine.insertMoney(5.00);
    machine.cancelTransaction();  // Get $5.00 back

    // Buy remaining items to trigger OutOfStock
    machine.insertMoney(1.50);
    machine.selectProduct(1);  // Last Cola

    machine.insertMoney(1.00);
    machine.selectProduct(3);  // Last Water — machine now out of stock

    machine.insertMoney(1.00);  // Should be rejected — out of stock

    return 0;
}
```

---

### State Transition Table

A state transition table documents all valid transitions:

```
┌─────────────────┬──────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ Current State    │ insertMoney      │ selectProduct    │ dispense         │ cancel           │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Idle            │ → HasMoney       │ "Insert money"   │ "Insert money"   │ "Nothing to do"  │
│ HasMoney        │ Add to balance   │ → Dispensing     │ "Select product" │ → Idle (refund)  │
│ Dispensing      │ "Wait..."        │ "Wait..."        │ → Idle/OutOfStock│ "Can't cancel"   │
│ OutOfStock      │ Return money     │ "Out of stock"   │ "Out of stock"   │ "Nothing to do"  │
└─────────────────┴──────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

### TCP Connection Example

```cpp
#include <iostream>
#include <memory>
using namespace std;

class TCPConnection;

class TCPState {
public:
    virtual void open(TCPConnection& conn) = 0;
    virtual void close(TCPConnection& conn) = 0;
    virtual void acknowledge(TCPConnection& conn) = 0;
    virtual void send(TCPConnection& conn, const string& data) = 0;
    virtual string getName() const = 0;
    virtual ~TCPState() = default;
};

class TCPConnection {
    unique_ptr<TCPState> state;
public:
    TCPConnection();

    void setState(unique_ptr<TCPState> s) {
        cout << "  [TCP: " << state->getName() << " → " << s->getName() << "]" << endl;
        state = std::move(s);
    }

    void open() { state->open(*this); }
    void close() { state->close(*this); }
    void acknowledge() { state->acknowledge(*this); }
    void send(const string& data) { state->send(*this, data); }
    string getStateName() const { return state->getName(); }
};

class ClosedState : public TCPState {
public:
    void open(TCPConnection& conn) override {
        cout << "  Opening connection... sending SYN" << endl;
        conn.setState(make_unique<class ListeningState>());
    }
    void close(TCPConnection& conn) override {
        cout << "  Already closed." << endl;
    }
    void acknowledge(TCPConnection& conn) override {
        cout << "  No connection to acknowledge." << endl;
    }
    void send(TCPConnection& conn, const string& data) override {
        cout << "  Cannot send — connection is closed." << endl;
    }
    string getName() const override { return "CLOSED"; }
};

class ListeningState : public TCPState {
public:
    void open(TCPConnection& conn) override {
        cout << "  Already opening..." << endl;
    }
    void close(TCPConnection& conn) override {
        cout << "  Aborting connection." << endl;
        conn.setState(make_unique<ClosedState>());
    }
    void acknowledge(TCPConnection& conn) override {
        cout << "  SYN-ACK received. Connection established!" << endl;
        conn.setState(make_unique<class EstablishedState>());
    }
    void send(TCPConnection& conn, const string& data) override {
        cout << "  Cannot send — still establishing connection." << endl;
    }
    string getName() const override { return "LISTENING"; }
};

class EstablishedState : public TCPState {
public:
    void open(TCPConnection& conn) override {
        cout << "  Connection already established." << endl;
    }
    void close(TCPConnection& conn) override {
        cout << "  Sending FIN... closing connection." << endl;
        conn.setState(make_unique<ClosedState>());
    }
    void acknowledge(TCPConnection& conn) override {
        cout << "  ACK received." << endl;
    }
    void send(TCPConnection& conn, const string& data) override {
        cout << "  Sending data: \"" << data << "\" (" << data.size() << " bytes)" << endl;
    }
    string getName() const override { return "ESTABLISHED"; }
};

TCPConnection::TCPConnection() : state(make_unique<ClosedState>()) {}

int main() {
    TCPConnection conn;

    conn.send("Hello");      // Can't send — closed
    conn.open();             // CLOSED → LISTENING
    conn.send("Hello");      // Can't send — still connecting
    conn.acknowledge();      // LISTENING → ESTABLISHED
    conn.send("Hello");      // Sends data
    conn.send("World");      // Sends data
    conn.close();            // ESTABLISHED → CLOSED

    return 0;
}
```

---

### State vs Strategy

| Aspect | State | Strategy |
|--------|-------|----------|
| Intent | Object changes behavior based on internal state | Client selects algorithm at runtime |
| Who triggers change | State objects trigger transitions themselves | Client explicitly sets the strategy |
| Awareness | States know about each other (for transitions) | Strategies are independent of each other |
| Number of behaviors | All behaviors change together (per state) | One behavior is swapped |
| Lifecycle | States transition automatically | Strategy is set and stays until changed |
| Analogy | Finite state machine | Pluggable algorithm |

---

### When to Use State

| Scenario | Why State Helps |
|----------|----------------|
| Object behavior depends on its state | Encapsulates state-specific behavior |
| Large conditional blocks based on state | Replaces if/else or switch with polymorphism |
| State transitions follow defined rules | Each state knows valid transitions |
| Need to add new states without modifying existing ones | OCP compliance |
| Finite state machine implementation | Natural mapping to State pattern |
| Workflow or process management | Each step is a state with defined transitions |

---


## 6.5 Template Method Pattern

> The Template Method pattern defines the skeleton of an algorithm in a base class method, deferring some steps to subclasses. It lets subclasses redefine certain steps of an algorithm without changing the algorithm's overall structure. This is the Hollywood Principle in action: "Don't call us, we'll call you."

---

### Why Template Method?

When multiple classes share the same algorithm structure but differ in specific steps:
- Data mining: read data → parse → analyze → report (steps differ by format)
- Game AI: initialize → take turn → check win → cleanup (steps differ by game)
- Build process: compile → link → test → package (steps differ by language)
- Document generation: header → body → footer (steps differ by format)

**The problem without Template Method:**
```cpp
// BAD: Duplicated algorithm structure across classes
class CSVDataMiner {
public:
    void mine(const string& path) {
        string rawData = readFile(path);        // same
        auto data = parseCSV(rawData);          // different
        auto analysis = analyzeData(data);      // same
        generateReport(analysis);               // same
    }
};

class JSONDataMiner {
public:
    void mine(const string& path) {
        string rawData = readFile(path);        // same (duplicated!)
        auto data = parseJSON(rawData);         // different
        auto analysis = analyzeData(data);      // same (duplicated!)
        generateReport(analysis);               // same (duplicated!)
    }
};

class XMLDataMiner {
public:
    void mine(const string& path) {
        string rawData = readFile(path);        // same (duplicated!)
        auto data = parseXML(rawData);          // different
        auto analysis = analyzeData(data);      // same (duplicated!)
        generateReport(analysis);               // same (duplicated!)
    }
};
// The algorithm structure (read → parse → analyze → report) is duplicated!
// Common steps are copy-pasted. Bug fix in one? Must fix all three!
```

---

### Structure

```
┌──────────────────────────────────┐
│     AbstractClass                │
├──────────────────────────────────┤
│ + templateMethod()  ← final     │  ← defines the algorithm skeleton
│ # step1()           ← concrete  │  ← common implementation
│ # step2()           ← abstract  │  ← subclasses MUST override
│ # step3()           ← abstract  │  ← subclasses MUST override
│ # hook()            ← virtual   │  ← subclasses CAN override (optional)
└──────────────────────────────────┘
                ▲
                │
     ┌──────────┴──────────┐
     │                     │
ConcreteClassA         ConcreteClassB
# step2() override     # step2() override
# step3() override     # step3() override
# hook() override      (uses default hook)
```

**Participants:**
- **AbstractClass:** Defines the template method (algorithm skeleton) and abstract/hook steps
- **ConcreteClass:** Implements the abstract steps; optionally overrides hooks
- **Template Method:** The method that defines the algorithm — typically `final` to prevent overriding

**Types of methods in the base class:**
1. **Concrete methods:** Common steps with default implementation (not overridden)
2. **Abstract methods:** Steps that MUST be implemented by subclasses (pure virtual)
3. **Hook methods:** Steps with default (often empty) implementation that CAN be overridden

---

### Basic Implementation — Data Mining Framework

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <sstream>
#include <fstream>
using namespace std;

// --- Abstract Class with Template Method ---
class DataMiner {
public:
    // THE TEMPLATE METHOD — defines the algorithm skeleton
    // 'final' prevents subclasses from changing the algorithm structure
    void mine(const string& path) final {
        string rawData = readData(path);          // step 1: read (concrete)
        auto records = parseData(rawData);         // step 2: parse (abstract)

        if (shouldFilter()) {                      // hook: optional filtering
            records = filterData(records);          // step 3: filter (hook)
        }

        auto analysis = analyzeData(records);      // step 4: analyze (concrete)
        generateReport(analysis);                  // step 5: report (concrete)

        onComplete();                              // hook: post-processing
    }

    virtual ~DataMiner() = default;

protected:
    // --- Concrete method (common implementation) ---
    string readData(const string& path) {
        cout << "Reading data from: " << path << endl;
        // Simulated file reading
        return "raw data from " + path;
    }

    // --- Abstract methods (subclasses MUST implement) ---
    virtual vector<string> parseData(const string& rawData) = 0;

    // --- Concrete method (common implementation) ---
    vector<string> analyzeData(const vector<string>& records) {
        cout << "Analyzing " << records.size() << " records..." << endl;
        vector<string> results;
        for (const auto& record : records) {
            results.push_back("Analyzed: " + record);
        }
        return results;
    }

    // --- Concrete method (common implementation) ---
    void generateReport(const vector<string>& analysis) {
        cout << "=== Report ===" << endl;
        for (const auto& item : analysis) {
            cout << "  " << item << endl;
        }
        cout << "=== End Report ===" << endl;
    }

    // --- Hook methods (optional override) ---
    virtual bool shouldFilter() { return false; }  // default: no filtering

    virtual vector<string> filterData(const vector<string>& records) {
        return records;  // default: no filtering
    }

    virtual void onComplete() {
        // default: do nothing — subclasses can override for cleanup, logging, etc.
    }
};

// --- Concrete Class: CSV Miner ---
class CSVDataMiner : public DataMiner {
protected:
    vector<string> parseData(const string& rawData) override {
        cout << "Parsing CSV data..." << endl;
        // Simulate CSV parsing
        return {"row1:col1,col2", "row2:col1,col2", "row3:col1,col2"};
    }

    bool shouldFilter() override { return true; }

    vector<string> filterData(const vector<string>& records) override {
        cout << "Filtering CSV records (removing empty rows)..." << endl;
        vector<string> filtered;
        for (const auto& r : records) {
            if (!r.empty()) filtered.push_back(r);
        }
        return filtered;
    }

    void onComplete() override {
        cout << "CSV mining complete. Closing file handles." << endl;
    }
};

// --- Concrete Class: JSON Miner ---
class JSONDataMiner : public DataMiner {
protected:
    vector<string> parseData(const string& rawData) override {
        cout << "Parsing JSON data..." << endl;
        return {"{\"name\":\"Alice\"}", "{\"name\":\"Bob\"}"};
    }

    void onComplete() override {
        cout << "JSON mining complete." << endl;
    }
};

// --- Concrete Class: XML Miner ---
class XMLDataMiner : public DataMiner {
protected:
    vector<string> parseData(const string& rawData) override {
        cout << "Parsing XML data..." << endl;
        return {"<record>Alice</record>", "<record>Bob</record>"};
    }
    // Uses default hooks (no filtering, no onComplete action)
};

// --- Client code ---
int main() {
    cout << "=== CSV Mining ===" << endl;
    CSVDataMiner csvMiner;
    csvMiner.mine("data.csv");

    cout << "\n=== JSON Mining ===" << endl;
    JSONDataMiner jsonMiner;
    jsonMiner.mine("data.json");

    cout << "\n=== XML Mining ===" << endl;
    XMLDataMiner xmlMiner;
    xmlMiner.mine("data.xml");

    return 0;
}
```

---

### Hook Methods

Hooks are methods with a default (often empty) implementation that subclasses can optionally override. They provide extension points without forcing subclasses to implement them.

```cpp
class GameAI {
public:
    // Template method
    void takeTurn() final {
        collectResources();
        buildStructures();

        if (shouldAttack()) {       // HOOK — subclasses decide
            attack();
        } else {
            defend();
        }

        onTurnEnd();                // HOOK — optional post-turn logic
    }

    virtual ~GameAI() = default;

protected:
    // Concrete steps
    void collectResources() {
        cout << "Collecting resources from owned territories" << endl;
    }

    // Abstract steps — MUST override
    virtual void buildStructures() = 0;
    virtual void attack() = 0;
    virtual void defend() = 0;

    // Hook methods — CAN override
    virtual bool shouldAttack() { return true; }  // default: always attack
    virtual void onTurnEnd() {}                    // default: do nothing
};

class AggressiveAI : public GameAI {
protected:
    void buildStructures() override {
        cout << "Building barracks and weapons factory" << endl;
    }
    void attack() override {
        cout << "Launching full-scale attack with all units!" << endl;
    }
    void defend() override {
        cout << "Minimal defense — attack is the best defense" << endl;
    }
    bool shouldAttack() override {
        return true;  // always attack
    }
};

class DefensiveAI : public GameAI {
    int turnsWithoutAttack = 0;
protected:
    void buildStructures() override {
        cout << "Building walls and defensive towers" << endl;
    }
    void attack() override {
        cout << "Sending small raiding party" << endl;
    }
    void defend() override {
        cout << "Fortifying all positions" << endl;
    }
    bool shouldAttack() override {
        turnsWithoutAttack++;
        return turnsWithoutAttack > 5;  // only attack after 5 defensive turns
    }
    void onTurnEnd() override {
        cout << "Checking perimeter defenses..." << endl;
    }
};
```

---

### Hollywood Principle

> "Don't call us, we'll call you."

The base class (framework) calls the subclass methods, not the other way around. The subclass doesn't control the flow — it just fills in the blanks.

```
Traditional approach:
  Subclass calls base class methods → subclass controls flow

Hollywood Principle (Template Method):
  Base class calls subclass methods → base class controls flow
  Subclass just provides implementations for specific steps

┌─────────────────────────────────────┐
│         Base Class (Framework)       │
│                                     │
│  templateMethod() {                 │
│      step1();          ← concrete   │
│      step2();          ← calls subclass (abstract)
│      if (hook()) {     ← calls subclass (hook)
│          step3();      ← calls subclass (abstract)
│      }                              │
│  }                                  │
│                                     │
│  "I control the algorithm.          │
│   I'll call you when I need you."   │
└─────────────────────────────────────┘
         │ calls
         ▼
┌─────────────────────────────────────┐
│         Subclass                     │
│                                     │
│  step2() { /* my implementation */ }│
│  step3() { /* my implementation */ }│
│  hook()  { return true; }           │
│                                     │
│  "I just provide the pieces.        │
│   The base class calls me."         │
└─────────────────────────────────────┘
```

---

### When to Use Template Method

| Scenario | Why Template Method Helps |
|----------|--------------------------|
| Multiple classes share the same algorithm structure | Eliminates code duplication |
| Want to control the algorithm flow from the base class | Subclasses can't change the order of steps |
| Some steps are common, others vary | Common steps in base, varying steps in subclasses |
| Framework design | Framework defines the flow, users customize steps |
| Need extension points without forcing implementation | Hook methods provide optional customization |

**Template Method vs Strategy:**

| Aspect | Template Method | Strategy |
|--------|----------------|----------|
| Mechanism | Inheritance (is-a) | Composition (has-a) |
| Granularity | Varies steps within an algorithm | Swaps the entire algorithm |
| Flexibility | Fixed at compile time | Changeable at runtime |
| Control | Base class controls flow | Client controls which strategy |
| Coupling | Tight (inheritance) | Loose (interface) |
| Class count | Fewer (just subclasses) | More (strategy classes + context) |

---


## 6.6 Iterator Pattern

> The Iterator pattern provides a way to access the elements of an aggregate object sequentially without exposing its underlying representation. It separates the traversal logic from the collection, allowing multiple traversal strategies over the same collection.

---

### Why Iterator?

When you need to traverse a collection without knowing its internal structure:
- Iterate over a tree (in-order, pre-order, post-order) without knowing it's a tree
- Traverse a graph (BFS, DFS) through a uniform interface
- Access elements of a custom data structure like a standard container
- Support multiple simultaneous traversals over the same collection

**The problem without Iterator:**
```cpp
// BAD: Client must know internal structure to traverse
class BinaryTree {
public:
    struct Node {
        int value;
        Node* left;
        Node* right;
    };
    Node* root;

    // Client must write traversal logic themselves!
    // And must know it's a tree with left/right pointers
};

// Client code — tightly coupled to tree internals
void printAll(BinaryTree::Node* node) {
    if (!node) return;
    printAll(node->left);
    cout << node->value << " ";
    printAll(node->right);
}
// What if we change from BinaryTree to a different structure? Client breaks!
```

---

### Structure

```
┌──────────────────────┐       ┌──────────────────────────┐
│   Iterable           │       │   Iterator (interface)    │
│   (Collection)       │       ├──────────────────────────┤
├──────────────────────┤       │ + hasNext(): bool         │
│ + createIterator()   │──────→│ + next(): Element         │
└──────────────────────┘       │ + current(): Element      │
                               └──────────────────────────┘
                                            ▲
                                            │
                                ┌───────────┼───────────┐
                                │                       │
                         ForwardIterator         ReverseIterator
```

---

### Internal vs External Iterators

**External Iterator:** The client controls the iteration (calls `next()` explicitly).
**Internal Iterator:** The collection controls the iteration (accepts a function to apply to each element).

```cpp
#include <iostream>
#include <vector>
#include <functional>
using namespace std;

class NumberCollection {
    vector<int> numbers;
public:
    void add(int n) { numbers.push_back(n); }

    // --- External Iterator (client controls) ---
    class Iterator {
        const vector<int>& data;
        size_t index;
    public:
        Iterator(const vector<int>& d, size_t i = 0) : data(d), index(i) {}
        bool hasNext() const { return index < data.size(); }
        int next() { return data[index++]; }
        int current() const { return data[index]; }
        void reset() { index = 0; }
    };

    Iterator createIterator() const {
        return Iterator(numbers);
    }

    // --- Internal Iterator (collection controls) ---
    void forEach(function<void(int)> action) const {
        for (int n : numbers) {
            action(n);
        }
    }

    // Internal iterator with early termination
    void forEachUntil(function<bool(int)> action) const {
        for (int n : numbers) {
            if (!action(n)) break;  // stop if action returns false
        }
    }
};

int main() {
    NumberCollection collection;
    collection.add(10);
    collection.add(20);
    collection.add(30);
    collection.add(40);

    // External iterator — client controls
    cout << "External iterator:" << endl;
    auto it = collection.createIterator();
    while (it.hasNext()) {
        cout << it.next() << " ";
    }
    cout << endl;

    // Internal iterator — collection controls
    cout << "Internal iterator:" << endl;
    collection.forEach([](int n) {
        cout << n << " ";
    });
    cout << endl;

    // Internal iterator with early termination
    cout << "First 2 elements:" << endl;
    int count = 0;
    collection.forEachUntil([&count](int n) {
        cout << n << " ";
        return ++count < 2;
    });
    cout << endl;

    return 0;
}
```

---

### Custom Iterator Compatible with Range-Based For Loop

To make a custom collection work with C++ range-based `for` loops, you need to provide `begin()` and `end()` methods that return iterators supporting `operator++`, `operator*`, and `operator!=`.

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>
using namespace std;

// --- Custom Linked List with STL-compatible Iterator ---
template <typename T>
class LinkedList {
    struct Node {
        T data;
        Node* next;
        Node(const T& d, Node* n = nullptr) : data(d), next(n) {}
    };

    Node* head = nullptr;
    Node* tail = nullptr;
    size_t count = 0;

public:
    ~LinkedList() {
        while (head) {
            Node* temp = head;
            head = head->next;
            delete temp;
        }
    }

    void pushBack(const T& value) {
        Node* newNode = new Node(value);
        if (!tail) {
            head = tail = newNode;
        } else {
            tail->next = newNode;
            tail = newNode;
        }
        count++;
    }

    void pushFront(const T& value) {
        head = new Node(value, head);
        if (!tail) tail = head;
        count++;
    }

    size_t size() const { return count; }
    bool empty() const { return count == 0; }

    // --- Iterator class (STL-compatible) ---
    class Iterator {
        Node* current;
    public:
        // Iterator traits (needed for STL compatibility)
        using iterator_category = forward_iterator_tag;
        using value_type = T;
        using difference_type = ptrdiff_t;
        using pointer = T*;
        using reference = T&;

        Iterator(Node* node) : current(node) {}

        // Dereference
        T& operator*() { return current->data; }
        const T& operator*() const { return current->data; }

        // Arrow operator
        T* operator->() { return &current->data; }

        // Pre-increment
        Iterator& operator++() {
            if (current) current = current->next;
            return *this;
        }

        // Post-increment
        Iterator operator++(int) {
            Iterator temp = *this;
            ++(*this);
            return temp;
        }

        // Comparison
        bool operator==(const Iterator& other) const { return current == other.current; }
        bool operator!=(const Iterator& other) const { return current != other.current; }
    };

    // --- Const Iterator ---
    class ConstIterator {
        const Node* current;
    public:
        using iterator_category = forward_iterator_tag;
        using value_type = T;
        using difference_type = ptrdiff_t;
        using pointer = const T*;
        using reference = const T&;

        ConstIterator(const Node* node) : current(node) {}

        const T& operator*() const { return current->data; }
        const T* operator->() const { return &current->data; }

        ConstIterator& operator++() {
            if (current) current = current->next;
            return *this;
        }

        ConstIterator operator++(int) {
            ConstIterator temp = *this;
            ++(*this);
            return temp;
        }

        bool operator==(const ConstIterator& other) const { return current == other.current; }
        bool operator!=(const ConstIterator& other) const { return current != other.current; }
    };

    // --- begin() and end() for range-based for loop ---
    Iterator begin() { return Iterator(head); }
    Iterator end() { return Iterator(nullptr); }
    ConstIterator begin() const { return ConstIterator(head); }
    ConstIterator end() const { return ConstIterator(nullptr); }
    ConstIterator cbegin() const { return ConstIterator(head); }
    ConstIterator cend() const { return ConstIterator(nullptr); }
};

// --- Usage ---
int main() {
    LinkedList<int> list;
    list.pushBack(10);
    list.pushBack(20);
    list.pushBack(30);
    list.pushFront(5);

    // Range-based for loop — works because we have begin() and end()!
    cout << "Range-based for:" << endl;
    for (int val : list) {
        cout << val << " ";
    }
    cout << endl;  // 5 10 20 30

    // Manual iterator usage
    cout << "Manual iterator:" << endl;
    for (auto it = list.begin(); it != list.end(); ++it) {
        cout << *it << " ";
    }
    cout << endl;

    // Works with STL algorithms too
    cout << "Using std::find:" << endl;
    auto found = find(list.begin(), list.end(), 20);
    if (found != list.end()) {
        cout << "Found: " << *found << endl;
    }

    // Const iteration
    const LinkedList<int>& constList = list;
    for (const int& val : constList) {
        cout << val << " ";
    }
    cout << endl;

    return 0;
}
```

---

### Tree Iterator (Multiple Traversal Strategies)

```cpp
#include <iostream>
#include <stack>
#include <queue>
#include <memory>
#include <functional>
using namespace std;

// --- Binary Tree ---
struct TreeNode {
    int value;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int v, TreeNode* l = nullptr, TreeNode* r = nullptr)
        : value(v), left(l), right(r) {}
};

// --- Iterator Interface ---
class TreeIterator {
public:
    virtual bool hasNext() = 0;
    virtual int next() = 0;
    virtual string name() const = 0;
    virtual ~TreeIterator() = default;
};

// --- In-Order Iterator (Left → Root → Right) ---
class InOrderIterator : public TreeIterator {
    stack<TreeNode*> stk;

    void pushLeft(TreeNode* node) {
        while (node) {
            stk.push(node);
            node = node->left;
        }
    }

public:
    InOrderIterator(TreeNode* root) {
        pushLeft(root);
    }

    bool hasNext() override {
        return !stk.empty();
    }

    int next() override {
        TreeNode* node = stk.top();
        stk.pop();
        pushLeft(node->right);
        return node->value;
    }

    string name() const override { return "InOrder"; }
};

// --- Pre-Order Iterator (Root → Left → Right) ---
class PreOrderIterator : public TreeIterator {
    stack<TreeNode*> stk;
public:
    PreOrderIterator(TreeNode* root) {
        if (root) stk.push(root);
    }

    bool hasNext() override {
        return !stk.empty();
    }

    int next() override {
        TreeNode* node = stk.top();
        stk.pop();
        if (node->right) stk.push(node->right);
        if (node->left) stk.push(node->left);
        return node->value;
    }

    string name() const override { return "PreOrder"; }
};

// --- Level-Order Iterator (BFS) ---
class LevelOrderIterator : public TreeIterator {
    queue<TreeNode*> q;
public:
    LevelOrderIterator(TreeNode* root) {
        if (root) q.push(root);
    }

    bool hasNext() override {
        return !q.empty();
    }

    int next() override {
        TreeNode* node = q.front();
        q.pop();
        if (node->left) q.push(node->left);
        if (node->right) q.push(node->right);
        return node->value;
    }

    string name() const override { return "LevelOrder"; }
};

// --- Binary Tree Collection ---
class BinaryTree {
    TreeNode* root;

    void deleteTree(TreeNode* node) {
        if (!node) return;
        deleteTree(node->left);
        deleteTree(node->right);
        delete node;
    }

public:
    BinaryTree(TreeNode* r) : root(r) {}
    ~BinaryTree() { deleteTree(root); }

    unique_ptr<TreeIterator> createInOrderIterator() {
        return make_unique<InOrderIterator>(root);
    }

    unique_ptr<TreeIterator> createPreOrderIterator() {
        return make_unique<PreOrderIterator>(root);
    }

    unique_ptr<TreeIterator> createLevelOrderIterator() {
        return make_unique<LevelOrderIterator>(root);
    }
};

// --- Client code ---
void printWithIterator(TreeIterator& it) {
    cout << it.name() << ": ";
    while (it.hasNext()) {
        cout << it.next() << " ";
    }
    cout << endl;
}

int main() {
    //       4
    //      / \
    //     2   6
    //    / \ / \
    //   1  3 5  7
    TreeNode* root = new TreeNode(4,
        new TreeNode(2, new TreeNode(1), new TreeNode(3)),
        new TreeNode(6, new TreeNode(5), new TreeNode(7))
    );

    BinaryTree tree(root);

    auto inOrder = tree.createInOrderIterator();
    printWithIterator(*inOrder);    // InOrder: 1 2 3 4 5 6 7

    auto preOrder = tree.createPreOrderIterator();
    printWithIterator(*preOrder);   // PreOrder: 4 2 1 3 6 5 7

    auto levelOrder = tree.createLevelOrderIterator();
    printWithIterator(*levelOrder); // LevelOrder: 4 2 6 1 3 5 7

    return 0;
}
```

---

### When to Use Iterator

| Scenario | Why Iterator Helps |
|----------|-------------------|
| Hide collection internals from clients | Clients use uniform interface regardless of structure |
| Support multiple traversal strategies | Different iterators for same collection |
| Provide uniform traversal interface | Same `hasNext()`/`next()` for lists, trees, graphs |
| Enable simultaneous traversals | Each iterator maintains its own position |
| Make custom collections work with STL | Implement `begin()`/`end()` with proper operators |
| Decouple algorithms from data structures | Algorithms work with iterators, not specific containers |

**C++ STL Iterator Categories:**

| Category | Operations | Examples |
|----------|-----------|----------|
| Input Iterator | Read, `++`, `==`, `!=` | `istream_iterator` |
| Output Iterator | Write, `++` | `ostream_iterator` |
| Forward Iterator | Read/Write, `++` | `forward_list::iterator` |
| Bidirectional Iterator | Forward + `--` | `list::iterator`, `set::iterator` |
| Random Access Iterator | Bidirectional + `[]`, `+`, `-`, `<` | `vector::iterator`, `deque::iterator` |

---


## 6.7 Mediator Pattern

> The Mediator pattern defines an object that encapsulates how a set of objects interact. It promotes loose coupling by keeping objects from referring to each other explicitly, and lets you vary their interaction independently. Instead of objects communicating directly (many-to-many), they communicate through a central mediator (one-to-many).

---

### Why Mediator?

When multiple objects need to communicate with each other, direct references create a tangled web of dependencies:
- Chat room: users send messages to each other through the room
- Air traffic control: planes communicate through the tower, not directly
- GUI dialog: form fields interact through the dialog (checkbox enables/disables a text field)
- Trading system: buyers and sellers interact through an exchange

**The problem without Mediator:**
```cpp
// BAD: Every component knows about every other component
class TextBox {
    Checkbox* checkbox;
    Button* submitBtn;
    Label* errorLabel;
    Dropdown* dropdown;
    // ... references to ALL related components!

public:
    void onChange() {
        // Must coordinate with every other component directly
        if (text.empty()) {
            submitBtn->disable();
            errorLabel->show("Required field");
        } else {
            submitBtn->enable();
            errorLabel->hide();
            if (checkbox->isChecked()) {
                dropdown->refresh();
            }
        }
        // Tight coupling! Adding a new component means modifying ALL existing ones
    }
};
```

**Problems:**
1. Every component coupled to every other component (N×N dependencies)
2. Adding/removing a component requires modifying many classes
3. Can't reuse components independently
4. Complex interaction logic scattered across all components

---

### Structure

```
Without Mediator (many-to-many):        With Mediator (one-to-many):

    A ←──→ B                                A
    ↕ ╲  ╱ ↕                                ↕
    D ←──→ C                            B ← M → D
                                            ↕
                                            C

┌──────────────────────┐       ┌──────────────────────────┐
│   Mediator           │       │   Colleague (interface)   │
├──────────────────────┤       ├──────────────────────────┤
│ + notify(sender,     │◄─────│ - mediator: Mediator*     │
│         event)       │       │ + setMediator(m)          │
└──────────────────────┘       └──────────────────────────┘
         ▲                                  ▲
         │                                  │
  ConcreteMediator              ┌───────────┼───────────┐
  (knows all colleagues)        │           │           │
                           ColleagueA  ColleagueB  ColleagueC
```

---

### Chat Room Example

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <memory>
#include <algorithm>
#include <ctime>
using namespace std;

// Forward declaration
class ChatMediator;

// --- Colleague (User) ---
class User {
protected:
    string name;
    ChatMediator* mediator;

public:
    User(const string& n) : name(n), mediator(nullptr) {}
    virtual ~User() = default;

    void setMediator(ChatMediator* m) { mediator = m; }
    string getName() const { return name; }

    virtual void send(const string& message, const string& to = "") = 0;
    virtual void receive(const string& from, const string& message) = 0;
};

// --- Mediator Interface ---
class ChatMediator {
public:
    virtual void sendMessage(const string& message, User* sender, const string& recipient = "") = 0;
    virtual void addUser(shared_ptr<User> user) = 0;
    virtual void removeUser(const string& name) = 0;
    virtual ~ChatMediator() = default;
};

// --- Concrete Colleague ---
class ChatUser : public User {
    vector<string> messageHistory;
public:
    ChatUser(const string& name) : User(name) {}

    void send(const string& message, const string& to = "") override {
        cout << "[" << name << " sends]: " << message;
        if (!to.empty()) cout << " (to " << to << ")";
        cout << endl;
        mediator->sendMessage(message, this, to);
    }

    void receive(const string& from, const string& message) override {
        string fullMessage = from + ": " + message;
        messageHistory.push_back(fullMessage);
        cout << "  [" << name << " receives]: " << fullMessage << endl;
    }

    void showHistory() const {
        cout << "\n--- " << name << "'s message history ---" << endl;
        for (const auto& msg : messageHistory) {
            cout << "  " << msg << endl;
        }
    }
};

// --- Concrete Mediator: Chat Room ---
class ChatRoom : public ChatMediator {
    string roomName;
    unordered_map<string, shared_ptr<User>> users;

public:
    ChatRoom(const string& name) : roomName(name) {
        cout << "Chat room \"" << roomName << "\" created." << endl;
    }

    void addUser(shared_ptr<User> user) override {
        user->setMediator(this);
        cout << "[" << roomName << "] " << user->getName() << " joined the room." << endl;

        // Notify existing users
        for (auto& [name, existingUser] : users) {
            existingUser->receive("System", user->getName() + " has joined the chat");
        }

        users[user->getName()] = user;
    }

    void removeUser(const string& name) override {
        users.erase(name);
        // Notify remaining users
        for (auto& [userName, user] : users) {
            user->receive("System", name + " has left the chat");
        }
        cout << "[" << roomName << "] " << name << " left the room." << endl;
    }

    void sendMessage(const string& message, User* sender, const string& recipient = "") override {
        if (recipient.empty()) {
            // Broadcast to all except sender
            for (auto& [name, user] : users) {
                if (name != sender->getName()) {
                    user->receive(sender->getName(), message);
                }
            }
        } else {
            // Direct message
            auto it = users.find(recipient);
            if (it != users.end()) {
                it->second->receive(sender->getName() + " (DM)", message);
            } else {
                sender->receive("System", "User " + recipient + " not found");
            }
        }
    }
};

// --- Client code ---
int main() {
    auto chatRoom = make_shared<ChatRoom>("General");

    auto alice = make_shared<ChatUser>("Alice");
    auto bob = make_shared<ChatUser>("Bob");
    auto charlie = make_shared<ChatUser>("Charlie");

    chatRoom->addUser(alice);
    chatRoom->addUser(bob);
    chatRoom->addUser(charlie);

    cout << endl;

    // Broadcast message
    alice->send("Hello everyone!");

    cout << endl;

    // Direct message
    bob->send("Hey Alice, how are you?", "Alice");

    cout << endl;

    // Another broadcast
    charlie->send("Good morning!");

    // Show history
    alice->showHistory();
    bob->showHistory();

    return 0;
}
```

---

### Air Traffic Controller Example

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <unordered_map>
using namespace std;

class ATCMediator;

// --- Colleague: Aircraft ---
class Aircraft {
protected:
    string callSign;
    int altitude;
    ATCMediator* atc;

public:
    Aircraft(const string& sign, int alt)
        : callSign(sign), altitude(alt), atc(nullptr) {}
    virtual ~Aircraft() = default;

    void setMediator(ATCMediator* m) { atc = m; }
    string getCallSign() const { return callSign; }
    int getAltitude() const { return altitude; }
    void setAltitude(int alt) { altitude = alt; }

    virtual void requestLanding() = 0;
    virtual void requestTakeoff() = 0;
    virtual void receiveMessage(const string& from, const string& message) = 0;
};

// --- Mediator ---
class ATCMediator {
public:
    virtual void requestLanding(Aircraft* aircraft) = 0;
    virtual void requestTakeoff(Aircraft* aircraft) = 0;
    virtual void registerAircraft(shared_ptr<Aircraft> aircraft) = 0;
    virtual void notifyAll(const string& message, Aircraft* except = nullptr) = 0;
    virtual ~ATCMediator() = default;
};

// --- Concrete Colleague ---
class CommercialAircraft : public Aircraft {
public:
    CommercialAircraft(const string& sign, int alt) : Aircraft(sign, alt) {}

    void requestLanding() override {
        cout << callSign << ": Requesting landing clearance. Altitude: " << altitude << "ft" << endl;
        atc->requestLanding(this);
    }

    void requestTakeoff() override {
        cout << callSign << ": Requesting takeoff clearance." << endl;
        atc->requestTakeoff(this);
    }

    void receiveMessage(const string& from, const string& message) override {
        cout << "  " << callSign << " receives from " << from << ": " << message << endl;
    }
};

// --- Concrete Mediator: Control Tower ---
class ControlTower : public ATCMediator {
    vector<shared_ptr<Aircraft>> aircraft;
    bool runwayFree = true;
    string activeRunway = "27L";

public:
    void registerAircraft(shared_ptr<Aircraft> ac) override {
        ac->setMediator(this);
        aircraft.push_back(ac);
        cout << "[Tower] " << ac->getCallSign() << " registered with tower." << endl;
    }

    void requestLanding(Aircraft* ac) override {
        if (runwayFree) {
            runwayFree = false;
            ac->receiveMessage("Tower", "Cleared to land runway " + activeRunway);
            notifyAll(ac->getCallSign() + " is landing on runway " + activeRunway + ". Hold positions.", ac);
        } else {
            ac->receiveMessage("Tower", "Runway occupied. Enter holding pattern at " +
                to_string(ac->getAltitude()) + "ft");
        }
    }

    void requestTakeoff(Aircraft* ac) override {
        if (runwayFree) {
            runwayFree = false;
            ac->receiveMessage("Tower", "Cleared for takeoff runway " + activeRunway);
            notifyAll(ac->getCallSign() + " is departing runway " + activeRunway + ". Hold short.", ac);
        } else {
            ac->receiveMessage("Tower", "Runway occupied. Hold short of runway " + activeRunway);
        }
    }

    void notifyAll(const string& message, Aircraft* except = nullptr) override {
        for (auto& ac : aircraft) {
            if (ac.get() != except) {
                ac->receiveMessage("Tower", message);
            }
        }
    }

    void runwayClear() {
        runwayFree = true;
        cout << "[Tower] Runway " << activeRunway << " is now clear." << endl;
        notifyAll("Runway " + activeRunway + " is clear.");
    }
};

// --- Client code ---
int main() {
    auto tower = make_shared<ControlTower>();

    auto flight1 = make_shared<CommercialAircraft>("UA-123", 10000);
    auto flight2 = make_shared<CommercialAircraft>("AA-456", 12000);
    auto flight3 = make_shared<CommercialAircraft>("DL-789", 0);

    tower->registerAircraft(flight1);
    tower->registerAircraft(flight2);
    tower->registerAircraft(flight3);

    cout << endl;

    flight1->requestLanding();   // Cleared — runway was free
    cout << endl;

    flight2->requestLanding();   // Denied — runway occupied
    cout << endl;

    tower->runwayClear();        // Runway freed
    cout << endl;

    flight3->requestTakeoff();   // Cleared — runway is free now
    cout << endl;

    return 0;
}
```

---

### When to Use Mediator

| Scenario | Why Mediator Helps |
|----------|-------------------|
| Many objects communicate in complex ways | Centralizes communication logic |
| Objects are tightly coupled through direct references | Reduces N×N to N×1 dependencies |
| Reusing components is difficult due to dependencies | Components only depend on mediator interface |
| Interaction logic should be customizable | Swap mediator implementation |
| GUI dialog coordination | Form fields interact through dialog mediator |
| Distributed system coordination | Central coordinator manages interactions |

**Mediator vs Observer:**

| Aspect | Mediator | Observer |
|--------|----------|----------|
| Communication | Bidirectional (colleagues ↔ mediator) | Unidirectional (subject → observers) |
| Awareness | Mediator knows all colleagues | Subject doesn't know concrete observers |
| Purpose | Coordinate complex interactions | Broadcast state changes |
| Coupling | Colleagues coupled to mediator | Observers loosely coupled to subject |
| Complexity | Can become a God object | Simpler, more focused |

**Pitfall:** The mediator can become a "God object" that knows too much and does too much. Keep mediators focused on coordination, not business logic.

---


## 6.8 Chain of Responsibility Pattern

> The Chain of Responsibility pattern lets you pass requests along a chain of handlers. Upon receiving a request, each handler decides either to process the request or to pass it to the next handler in the chain. It decouples the sender of a request from its receivers by giving more than one object a chance to handle the request.

---

### Why Chain of Responsibility?

When a request should be handled by one of several handlers, but you don't know which one in advance:
- Logging: DEBUG → INFO → WARN → ERROR (each level decides whether to log)
- Middleware: authentication → authorization → validation → rate limiting → handler
- Support tickets: Level 1 → Level 2 → Level 3 → Manager
- Event handling: child widget → parent widget → window → application

**The problem without Chain of Responsibility:**
```cpp
// BAD: Caller must know which handler to use
class SupportSystem {
public:
    void handleTicket(Ticket& ticket) {
        if (ticket.severity == "low") {
            level1Support.handle(ticket);
        } else if (ticket.severity == "medium") {
            level2Support.handle(ticket);
        } else if (ticket.severity == "high") {
            level3Support.handle(ticket);
        } else if (ticket.severity == "critical") {
            manager.handle(ticket);
        }
        // Caller is coupled to ALL handlers and the routing logic
        // Adding a new level means modifying this code
    }
};
```

---

### Structure

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Handler A    │───→│  Handler B    │───→│  Handler C    │───→│  Handler D    │
│              │    │              │    │              │    │              │
│ Can handle?  │    │ Can handle?  │    │ Can handle?  │    │ Can handle?  │
│ Yes → process│    │ Yes → process│    │ Yes → process│    │ Yes → process│
│ No → pass on │    │ No → pass on │    │ No → pass on │    │ No → end     │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘

┌──────────────────────────┐
│   Handler (interface)     │
├──────────────────────────┤
│ - next: Handler*          │
├──────────────────────────┤
│ + setNext(handler)        │
│ + handle(request)         │
└──────────────────────────┘
             ▲
             │
  ┌──────────┼──────────┐
  │          │          │
HandlerA  HandlerB  HandlerC
```

---

### Logging Levels Example

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <ctime>
#include <sstream>
#include <iomanip>
using namespace std;

// --- Log Levels ---
enum class LogLevel {
    DEBUG = 0,
    INFO = 1,
    WARN = 2,
    ERROR = 3,
    FATAL = 4
};

string logLevelToString(LogLevel level) {
    switch (level) {
        case LogLevel::DEBUG: return "DEBUG";
        case LogLevel::INFO:  return "INFO ";
        case LogLevel::WARN:  return "WARN ";
        case LogLevel::ERROR: return "ERROR";
        case LogLevel::FATAL: return "FATAL";
    }
    return "?????";
}

// --- Handler Interface ---
class LogHandler {
protected:
    unique_ptr<LogHandler> nextHandler;
    LogLevel handlerLevel;

    string getTimestamp() const {
        auto now = time(nullptr);
        stringstream ss;
        ss << put_time(localtime(&now), "%H:%M:%S");
        return ss.str();
    }

public:
    LogHandler(LogLevel level) : handlerLevel(level) {}
    virtual ~LogHandler() = default;

    LogHandler* setNext(unique_ptr<LogHandler> next) {
        nextHandler = std::move(next);
        return nextHandler.get();
    }

    void handle(LogLevel level, const string& message) {
        if (level >= handlerLevel) {
            write(level, message);
        }
        // Pass to next handler regardless (each handler decides independently)
        if (nextHandler) {
            nextHandler->handle(level, message);
        }
    }

protected:
    virtual void write(LogLevel level, const string& message) = 0;
};

// --- Concrete Handlers ---
class ConsoleLogHandler : public LogHandler {
public:
    ConsoleLogHandler(LogLevel level) : LogHandler(level) {}

protected:
    void write(LogLevel level, const string& message) override {
        cout << "[" << getTimestamp() << "] [CONSOLE] ["
             << logLevelToString(level) << "] " << message << endl;
    }
};

class FileLogHandler : public LogHandler {
    string filename;
public:
    FileLogHandler(LogLevel level, const string& file)
        : LogHandler(level), filename(file) {}

protected:
    void write(LogLevel level, const string& message) override {
        // Simulated file write
        cout << "[" << getTimestamp() << "] [FILE:" << filename << "] ["
             << logLevelToString(level) << "] " << message << endl;
    }
};

class EmailLogHandler : public LogHandler {
    string recipient;
public:
    EmailLogHandler(LogLevel level, const string& email)
        : LogHandler(level), recipient(email) {}

protected:
    void write(LogLevel level, const string& message) override {
        cout << "[" << getTimestamp() << "] [EMAIL→" << recipient << "] ["
             << logLevelToString(level) << "] " << message << endl;
    }
};

class SlackLogHandler : public LogHandler {
    string channel;
public:
    SlackLogHandler(LogLevel level, const string& ch)
        : LogHandler(level), channel(ch) {}

protected:
    void write(LogLevel level, const string& message) override {
        cout << "[" << getTimestamp() << "] [SLACK:#" << channel << "] ["
             << logLevelToString(level) << "] " << message << endl;
    }
};

// --- Client code ---
int main() {
    // Build the chain:
    // Console (DEBUG+) → File (INFO+) → Slack (WARN+) → Email (ERROR+)
    auto console = make_unique<ConsoleLogHandler>(LogLevel::DEBUG);
    auto file = make_unique<FileLogHandler>(LogLevel::INFO, "app.log");
    auto slack = make_unique<SlackLogHandler>(LogLevel::WARN, "alerts");
    auto email = make_unique<EmailLogHandler>(LogLevel::ERROR, "admin@example.com");

    // Chain them
    auto* filePtr = console->setNext(std::move(file));
    auto* slackPtr = filePtr->setNext(std::move(slack));
    slackPtr->setNext(std::move(email));

    cout << "=== DEBUG message ===" << endl;
    console->handle(LogLevel::DEBUG, "Variable x = 42");
    // Only Console handles it

    cout << "\n=== INFO message ===" << endl;
    console->handle(LogLevel::INFO, "User logged in");
    // Console + File handle it

    cout << "\n=== WARN message ===" << endl;
    console->handle(LogLevel::WARN, "Disk usage at 85%");
    // Console + File + Slack handle it

    cout << "\n=== ERROR message ===" << endl;
    console->handle(LogLevel::ERROR, "Database connection failed!");
    // All four handlers handle it

    cout << "\n=== FATAL message ===" << endl;
    console->handle(LogLevel::FATAL, "System crash — out of memory!");
    // All four handlers handle it

    return 0;
}
```

---

### Middleware Pipeline Example

A common real-world use: HTTP request processing pipeline where each middleware can process or reject the request.

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <unordered_map>
using namespace std;

// --- Request object ---
struct HttpRequest {
    string method;
    string path;
    unordered_map<string, string> headers;
    string body;
    string clientIP;
    string userId;  // set by auth middleware
};

struct HttpResponse {
    int statusCode = 200;
    string body;
};

// --- Middleware Handler ---
class Middleware {
protected:
    unique_ptr<Middleware> next;

public:
    virtual ~Middleware() = default;

    Middleware* setNext(unique_ptr<Middleware> nextMiddleware) {
        next = std::move(nextMiddleware);
        return next.get();
    }

    virtual HttpResponse handle(HttpRequest& request) {
        if (next) {
            return next->handle(request);
        }
        return {200, "OK"};
    }
};

// --- Concrete Middleware ---
class AuthenticationMiddleware : public Middleware {
public:
    HttpResponse handle(HttpRequest& request) override {
        cout << "  [Auth] Checking authentication..." << endl;

        auto it = request.headers.find("Authorization");
        if (it == request.headers.end()) {
            cout << "  [Auth] REJECTED — No auth token" << endl;
            return {401, "Unauthorized: Missing authentication token"};
        }

        string token = it->second;
        if (token.substr(0, 7) != "Bearer ") {
            cout << "  [Auth] REJECTED — Invalid token format" << endl;
            return {401, "Unauthorized: Invalid token format"};
        }

        // Simulate token validation
        request.userId = "user_" + token.substr(7, 4);
        cout << "  [Auth] Authenticated as " << request.userId << endl;

        return Middleware::handle(request);  // pass to next
    }
};

class RateLimitMiddleware : public Middleware {
    unordered_map<string, int> requestCounts;
    int maxRequests;

public:
    RateLimitMiddleware(int max) : maxRequests(max) {}

    HttpResponse handle(HttpRequest& request) override {
        cout << "  [RateLimit] Checking rate limit for " << request.clientIP << "..." << endl;

        requestCounts[request.clientIP]++;
        if (requestCounts[request.clientIP] > maxRequests) {
            cout << "  [RateLimit] REJECTED — Rate limit exceeded" << endl;
            return {429, "Too Many Requests"};
        }

        cout << "  [RateLimit] OK (" << requestCounts[request.clientIP]
             << "/" << maxRequests << ")" << endl;

        return Middleware::handle(request);
    }
};

class ValidationMiddleware : public Middleware {
public:
    HttpResponse handle(HttpRequest& request) override {
        cout << "  [Validation] Validating request..." << endl;

        if (request.method == "POST" || request.method == "PUT") {
            if (request.body.empty()) {
                cout << "  [Validation] REJECTED — Empty body for " << request.method << endl;
                return {400, "Bad Request: Body required for " + request.method};
            }
            auto ct = request.headers.find("Content-Type");
            if (ct == request.headers.end()) {
                cout << "  [Validation] REJECTED — Missing Content-Type" << endl;
                return {400, "Bad Request: Content-Type header required"};
            }
        }

        cout << "  [Validation] OK" << endl;
        return Middleware::handle(request);
    }
};

class LoggingMiddleware : public Middleware {
public:
    HttpResponse handle(HttpRequest& request) override {
        cout << "  [Log] " << request.method << " " << request.path
             << " from " << request.clientIP << endl;

        auto response = Middleware::handle(request);

        cout << "  [Log] Response: " << response.statusCode << endl;
        return response;
    }
};

// --- Final Handler (the actual endpoint) ---
class RequestHandler : public Middleware {
public:
    HttpResponse handle(HttpRequest& request) override {
        cout << "  [Handler] Processing " << request.method << " " << request.path << endl;
        return {200, "Success! Processed by " + request.userId};
    }
};

// --- Client code ---
int main() {
    // Build middleware chain:
    // Logging → RateLimit → Auth → Validation → Handler
    auto logging = make_unique<LoggingMiddleware>();
    auto rateLimit = make_unique<RateLimitMiddleware>(3);
    auto auth = make_unique<AuthenticationMiddleware>();
    auto validation = make_unique<ValidationMiddleware>();
    auto handler = make_unique<RequestHandler>();

    auto* rl = logging->setNext(std::move(rateLimit));
    auto* au = rl->setNext(std::move(auth));
    auto* va = au->setNext(std::move(validation));
    va->setNext(std::move(handler));

    // Test 1: Valid GET request
    cout << "=== Request 1: Valid GET ===" << endl;
    HttpRequest req1{"GET", "/api/users", {{"Authorization", "Bearer abc123"}}, "", "192.168.1.1", ""};
    auto resp1 = logging->handle(req1);
    cout << "Result: " << resp1.statusCode << " — " << resp1.body << "\n" << endl;

    // Test 2: Missing auth
    cout << "=== Request 2: No Auth ===" << endl;
    HttpRequest req2{"GET", "/api/users", {}, "", "192.168.1.2", ""};
    auto resp2 = logging->handle(req2);
    cout << "Result: " << resp2.statusCode << " — " << resp2.body << "\n" << endl;

    // Test 3: POST without body
    cout << "=== Request 3: POST without body ===" << endl;
    HttpRequest req3{"POST", "/api/users", {{"Authorization", "Bearer xyz789"}}, "", "192.168.1.1", ""};
    auto resp3 = logging->handle(req3);
    cout << "Result: " << resp3.statusCode << " — " << resp3.body << "\n" << endl;

    return 0;
}
```

---

### When to Use Chain of Responsibility

| Scenario | Why Chain of Responsibility Helps |
|----------|----------------------------------|
| Multiple handlers for a request, unknown which will handle | Request flows until a handler processes it |
| Processing pipeline with optional steps | Each handler decides to process or pass |
| Decoupling sender from receivers | Sender doesn't know which handler processes |
| Dynamic handler configuration | Chain can be built/modified at runtime |
| Logging with multiple levels/destinations | Each logger handles its level and passes on |
| Middleware/filter pipelines | Authentication → authorization → validation → handler |

**Chain of Responsibility vs Decorator:**

| Aspect | Chain of Responsibility | Decorator |
|--------|------------------------|-----------|
| Intent | Find the right handler for a request | Add behavior to an object |
| Flow | Can stop at any handler | Always passes through all decorators |
| Awareness | Handlers independent of each other | Decorators wrap the same interface |
| Result | One handler processes the request | All decorators contribute to the result |

---


## 6.9 Visitor Pattern

> The Visitor pattern lets you add new operations to existing object structures without modifying the classes of the elements on which it operates. It achieves this through a technique called **double dispatch** — the operation executed depends on both the type of the visitor and the type of the element being visited.

---

### Why Visitor?

When you need to perform many unrelated operations on objects in a structure, and you don't want to pollute their classes with these operations:
- Compiler: AST nodes need type checking, code generation, optimization, pretty printing
- File system: files/directories need size calculation, search, permission check, backup
- Document: elements need rendering, exporting, spell checking, word counting
- Shape hierarchy: shapes need area calculation, drawing, serialization, collision detection

**The problem without Visitor:**
```cpp
// BAD: Every new operation requires modifying ALL element classes
class Circle {
public:
    void draw() { /* ... */ }
    double area() { /* ... */ }
    string toJSON() { /* ... */ }        // added later
    string toXML() { /* ... */ }         // added later
    void optimize() { /* ... */ }        // added later
    bool collidesWith(Shape*) { /* ... */ } // added later
    // Every new operation means modifying Circle, Rectangle, Triangle, etc.!
};
```

**Problems:**
1. Adding a new operation requires modifying every element class
2. Unrelated operations clutter the element classes (violates SRP)
3. Element classes become bloated with operations they shouldn't know about

---

### Structure

```
┌──────────────────────────┐       ┌──────────────────────────┐
│   Element (interface)     │       │   Visitor (interface)     │
├──────────────────────────┤       ├──────────────────────────┤
│ + accept(visitor)         │       │ + visitCircle(Circle)     │
└──────────────────────────┘       │ + visitRect(Rectangle)    │
          ▲                        │ + visitTriangle(Triangle) │
          │                        └──────────────────────────┘
   ┌──────┼──────┐                          ▲
   │      │      │                          │
Circle  Rect  Triangle              ┌───────┼───────┐
                                    │               │
                              AreaVisitor    DrawVisitor

Double Dispatch:
  element.accept(visitor)  →  visitor.visitCircle(this)
  1st dispatch: accept()   →  resolved by element's type
  2nd dispatch: visitX()   →  resolved by visitor's type
```

---

### Double Dispatch Explained

C++ uses single dispatch — the method called depends only on the object's type (via vtable). Visitor achieves double dispatch:

```cpp
// Single dispatch: method depends on ONE object's type
shape->draw();  // calls Circle::draw() or Rectangle::draw()

// Double dispatch: operation depends on TWO objects' types
// Step 1: element.accept(visitor) — dispatches on element type
// Step 2: visitor.visitCircle(this) — dispatches on visitor type

// The combination of element type + visitor type determines the operation
```

---

### Complete Implementation — Shape Operations

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <cmath>
#include <sstream>
using namespace std;

// Forward declarations
class Circle;
class Rectangle;
class Triangle;

// --- Visitor Interface ---
class ShapeVisitor {
public:
    virtual void visitCircle(const Circle& circle) = 0;
    virtual void visitRectangle(const Rectangle& rect) = 0;
    virtual void visitTriangle(const Triangle& triangle) = 0;
    virtual ~ShapeVisitor() = default;
};

// --- Element Interface ---
class Shape {
public:
    virtual void accept(ShapeVisitor& visitor) const = 0;
    virtual string name() const = 0;
    virtual ~Shape() = default;
};

// --- Concrete Elements ---
class Circle : public Shape {
    double radius;
    double x, y;  // center position
public:
    Circle(double r, double cx = 0, double cy = 0)
        : radius(r), x(cx), y(cy) {}

    void accept(ShapeVisitor& visitor) const override {
        visitor.visitCircle(*this);  // double dispatch: calls visitor's Circle-specific method
    }

    double getRadius() const { return radius; }
    double getX() const { return x; }
    double getY() const { return y; }
    string name() const override { return "Circle"; }
};

class Rectangle : public Shape {
    double width, height;
    double x, y;  // top-left position
public:
    Rectangle(double w, double h, double px = 0, double py = 0)
        : width(w), height(h), x(px), y(py) {}

    void accept(ShapeVisitor& visitor) const override {
        visitor.visitRectangle(*this);
    }

    double getWidth() const { return width; }
    double getHeight() const { return height; }
    double getX() const { return x; }
    double getY() const { return y; }
    string name() const override { return "Rectangle"; }
};

class Triangle : public Shape {
    double base, height;
    double x, y;
public:
    Triangle(double b, double h, double px = 0, double py = 0)
        : base(b), height(h), x(px), y(py) {}

    void accept(ShapeVisitor& visitor) const override {
        visitor.visitTriangle(*this);
    }

    double getBase() const { return base; }
    double getHeight() const { return height; }
    double getX() const { return x; }
    double getY() const { return y; }
    string name() const override { return "Triangle"; }
};

// --- Concrete Visitors (new operations without modifying shapes) ---

// Visitor 1: Calculate area
class AreaCalculator : public ShapeVisitor {
    double totalArea = 0;
public:
    void visitCircle(const Circle& c) override {
        double area = M_PI * c.getRadius() * c.getRadius();
        cout << "  Circle area: " << area << endl;
        totalArea += area;
    }

    void visitRectangle(const Rectangle& r) override {
        double area = r.getWidth() * r.getHeight();
        cout << "  Rectangle area: " << area << endl;
        totalArea += area;
    }

    void visitTriangle(const Triangle& t) override {
        double area = 0.5 * t.getBase() * t.getHeight();
        cout << "  Triangle area: " << area << endl;
        totalArea += area;
    }

    double getTotalArea() const { return totalArea; }
};

// Visitor 2: Export to JSON
class JSONExporter : public ShapeVisitor {
    stringstream json;
    int count = 0;
public:
    void visitCircle(const Circle& c) override {
        if (count > 0) json << ",\n";
        json << "  {\"type\":\"circle\",\"radius\":" << c.getRadius()
             << ",\"x\":" << c.getX() << ",\"y\":" << c.getY() << "}";
        count++;
    }

    void visitRectangle(const Rectangle& r) override {
        if (count > 0) json << ",\n";
        json << "  {\"type\":\"rectangle\",\"width\":" << r.getWidth()
             << ",\"height\":" << r.getHeight()
             << ",\"x\":" << r.getX() << ",\"y\":" << r.getY() << "}";
        count++;
    }

    void visitTriangle(const Triangle& t) override {
        if (count > 0) json << ",\n";
        json << "  {\"type\":\"triangle\",\"base\":" << t.getBase()
             << ",\"height\":" << t.getHeight()
             << ",\"x\":" << t.getX() << ",\"y\":" << t.getY() << "}";
        count++;
    }

    string getJSON() const {
        return "[\n" + json.str() + "\n]";
    }
};

// Visitor 3: Draw (render) shapes
class DrawVisitor : public ShapeVisitor {
public:
    void visitCircle(const Circle& c) override {
        cout << "  Drawing circle at (" << c.getX() << "," << c.getY()
             << ") with radius " << c.getRadius() << endl;
    }

    void visitRectangle(const Rectangle& r) override {
        cout << "  Drawing rectangle at (" << r.getX() << "," << r.getY()
             << ") size " << r.getWidth() << "x" << r.getHeight() << endl;
    }

    void visitTriangle(const Triangle& t) override {
        cout << "  Drawing triangle at (" << t.getX() << "," << t.getY()
             << ") base=" << t.getBase() << " height=" << t.getHeight() << endl;
    }
};

// Visitor 4: Calculate perimeter
class PerimeterCalculator : public ShapeVisitor {
    double totalPerimeter = 0;
public:
    void visitCircle(const Circle& c) override {
        double p = 2 * M_PI * c.getRadius();
        cout << "  Circle perimeter: " << p << endl;
        totalPerimeter += p;
    }

    void visitRectangle(const Rectangle& r) override {
        double p = 2 * (r.getWidth() + r.getHeight());
        cout << "  Rectangle perimeter: " << p << endl;
        totalPerimeter += p;
    }

    void visitTriangle(const Triangle& t) override {
        // Simplified: assuming isosceles triangle
        double side = sqrt(pow(t.getBase() / 2, 2) + pow(t.getHeight(), 2));
        double p = t.getBase() + 2 * side;
        cout << "  Triangle perimeter: " << p << endl;
        totalPerimeter += p;
    }

    double getTotalPerimeter() const { return totalPerimeter; }
};

// --- Client code ---
int main() {
    // Create a collection of shapes
    vector<unique_ptr<Shape>> shapes;
    shapes.push_back(make_unique<Circle>(5.0, 10, 20));
    shapes.push_back(make_unique<Rectangle>(4.0, 6.0, 30, 40));
    shapes.push_back(make_unique<Triangle>(3.0, 4.0, 50, 60));
    shapes.push_back(make_unique<Circle>(2.5, 70, 80));

    // Apply different visitors — NO modification to shape classes!

    cout << "=== Area Calculation ===" << endl;
    AreaCalculator areaCalc;
    for (const auto& shape : shapes) {
        shape->accept(areaCalc);
    }
    cout << "Total area: " << areaCalc.getTotalArea() << endl;

    cout << "\n=== Drawing ===" << endl;
    DrawVisitor drawer;
    for (const auto& shape : shapes) {
        shape->accept(drawer);
    }

    cout << "\n=== JSON Export ===" << endl;
    JSONExporter exporter;
    for (const auto& shape : shapes) {
        shape->accept(exporter);
    }
    cout << exporter.getJSON() << endl;

    cout << "\n=== Perimeter Calculation ===" << endl;
    PerimeterCalculator perimCalc;
    for (const auto& shape : shapes) {
        shape->accept(perimCalc);
    }
    cout << "Total perimeter: " << perimCalc.getTotalPerimeter() << endl;

    return 0;
}
```

---

### AST Traversal Example

A classic use case — visiting nodes of an Abstract Syntax Tree in a compiler:

```cpp
#include <iostream>
#include <memory>
#include <vector>
using namespace std;

// Forward declarations
class NumberNode;
class BinaryOpNode;
class UnaryOpNode;

// --- Visitor Interface ---
class ASTVisitor {
public:
    virtual void visitNumber(const NumberNode& node) = 0;
    virtual void visitBinaryOp(const BinaryOpNode& node) = 0;
    virtual void visitUnaryOp(const UnaryOpNode& node) = 0;
    virtual ~ASTVisitor() = default;
};

// --- AST Node Interface ---
class ASTNode {
public:
    virtual void accept(ASTVisitor& visitor) const = 0;
    virtual ~ASTNode() = default;
};

// --- Concrete Nodes ---
class NumberNode : public ASTNode {
    double value;
public:
    NumberNode(double v) : value(v) {}
    void accept(ASTVisitor& visitor) const override { visitor.visitNumber(*this); }
    double getValue() const { return value; }
};

class BinaryOpNode : public ASTNode {
    char op;
    unique_ptr<ASTNode> left, right;
public:
    BinaryOpNode(char o, unique_ptr<ASTNode> l, unique_ptr<ASTNode> r)
        : op(o), left(std::move(l)), right(std::move(r)) {}
    void accept(ASTVisitor& visitor) const override { visitor.visitBinaryOp(*this); }
    char getOp() const { return op; }
    const ASTNode& getLeft() const { return *left; }
    const ASTNode& getRight() const { return *right; }
};

class UnaryOpNode : public ASTNode {
    char op;
    unique_ptr<ASTNode> operand;
public:
    UnaryOpNode(char o, unique_ptr<ASTNode> child)
        : op(o), operand(std::move(child)) {}
    void accept(ASTVisitor& visitor) const override { visitor.visitUnaryOp(*this); }
    char getOp() const { return op; }
    const ASTNode& getOperand() const { return *operand; }
};

// --- Visitor: Evaluate expression ---
class EvaluatorVisitor : public ASTVisitor {
    double result = 0;
public:
    void visitNumber(const NumberNode& node) override {
        result = node.getValue();
    }

    void visitBinaryOp(const BinaryOpNode& node) override {
        node.getLeft().accept(*this);
        double leftVal = result;
        node.getRight().accept(*this);
        double rightVal = result;

        switch (node.getOp()) {
            case '+': result = leftVal + rightVal; break;
            case '-': result = leftVal - rightVal; break;
            case '*': result = leftVal * rightVal; break;
            case '/': result = leftVal / rightVal; break;
        }
    }

    void visitUnaryOp(const UnaryOpNode& node) override {
        node.getOperand().accept(*this);
        if (node.getOp() == '-') result = -result;
    }

    double getResult() const { return result; }
};

// --- Visitor: Pretty print expression ---
class PrintVisitor : public ASTVisitor {
    string output;
public:
    void visitNumber(const NumberNode& node) override {
        output += to_string((int)node.getValue());
    }

    void visitBinaryOp(const BinaryOpNode& node) override {
        output += "(";
        node.getLeft().accept(*this);
        output += " ";
        output += node.getOp();
        output += " ";
        node.getRight().accept(*this);
        output += ")";
    }

    void visitUnaryOp(const UnaryOpNode& node) override {
        output += "(";
        output += node.getOp();
        node.getOperand().accept(*this);
        output += ")";
    }

    string getOutput() const { return output; }
};

// --- Client code ---
int main() {
    // Build AST for: (3 + 4) * -(2 - 1)
    auto ast = make_unique<BinaryOpNode>('*',
        make_unique<BinaryOpNode>('+',
            make_unique<NumberNode>(3),
            make_unique<NumberNode>(4)
        ),
        make_unique<UnaryOpNode>('-',
            make_unique<BinaryOpNode>('-',
                make_unique<NumberNode>(2),
                make_unique<NumberNode>(1)
            )
        )
    );

    // Evaluate
    EvaluatorVisitor evaluator;
    ast->accept(evaluator);
    cout << "Result: " << evaluator.getResult() << endl;  // -7

    // Pretty print
    PrintVisitor printer;
    ast->accept(printer);
    cout << "Expression: " << printer.getOutput() << endl;  // ((3 + 4) * (-(2 - 1)))

    return 0;
}
```

---

### When to Use Visitor

| Scenario | Why Visitor Helps |
|----------|------------------|
| Need to add operations to a stable class hierarchy | New visitors = new operations, no class changes |
| Many unrelated operations on the same structure | Each visitor encapsulates one operation |
| Object structure rarely changes, operations change often | Visitor is ideal when elements are stable |
| Need double dispatch | Operation depends on both element and visitor type |
| Compiler/interpreter AST processing | Type checking, code gen, optimization as visitors |
| Reporting/exporting in multiple formats | Each format is a visitor |

**Trade-offs:**

| Advantage | Disadvantage |
|-----------|-------------|
| Easy to add new operations (new visitor) | Hard to add new element types (must update ALL visitors) |
| Related behavior grouped in one visitor | Breaks encapsulation (visitor needs access to element data) |
| Visitor can accumulate state across elements | Double dispatch adds complexity |
| Clean separation of concerns | Overkill for simple hierarchies |

**Rule of thumb:** Use Visitor when the element hierarchy is stable but operations change frequently. Avoid it when new element types are added often.

---


## 6.10 Memento Pattern

> The Memento pattern captures and externalizes an object's internal state so that the object can be restored to this state later, without violating encapsulation. It provides the ability to undo/restore an object to a previous state.

---

### Why Memento?

When you need to save and restore an object's state:
- Text editor: save document state for undo
- Game: save checkpoints for loading later
- Database: transaction rollback
- Configuration: save settings before experimental changes
- Drawing app: save canvas state for undo

**The problem without Memento:**
```cpp
// BAD: Exposing internal state to save/restore
class Editor {
public:
    string content;     // public — anyone can read/modify!
    int cursorPos;      // public — breaks encapsulation!
    string fontName;    // public — no control over state!

    // To save state, external code must know ALL internal fields
    // and copy them manually. If we add a new field, all save/restore
    // code must be updated!
};
```

---

### Structure

```
┌──────────────────────┐     ┌──────────────────────┐     ┌──────────────────────┐
│     Originator        │     │      Memento          │     │     Caretaker         │
├──────────────────────┤     ├──────────────────────┤     ├──────────────────────┤
│ - state              │     │ - state (snapshot)    │     │ - mementos: list     │
├──────────────────────┤     ├──────────────────────┤     ├──────────────────────┤
│ + createMemento()    │────→│ + getState()          │←────│ + addMemento(m)      │
│ + restore(memento)   │     │   (only for Originator)│    │ + getMemento(index)  │
└──────────────────────┘     └──────────────────────┘     └──────────────────────┘
```

**Participants:**
- **Originator:** The object whose state needs to be saved. Creates mementos and restores from them.
- **Memento:** Stores the internal state of the Originator. Opaque to other objects (encapsulation preserved).
- **Caretaker:** Manages mementos (stores, retrieves) but never examines or modifies their contents.

---

### Basic Implementation — Text Editor

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
using namespace std;

// --- Memento ---
class EditorMemento {
    // Only the Originator (Editor) should access these
    friend class TextEditor;

    string content;
    int cursorPosition;
    string timestamp;

    EditorMemento(const string& c, int pos, const string& ts)
        : content(c), cursorPosition(pos), timestamp(ts) {}

public:
    // Public interface — only metadata, not the actual state
    string getTimestamp() const { return timestamp; }
    string getDescription() const {
        string preview = content.substr(0, 30);
        if (content.length() > 30) preview += "...";
        return "[" + timestamp + "] \"" + preview + "\" (cursor: " + to_string(cursorPosition) + ")";
    }
};

// --- Originator ---
class TextEditor {
    string content;
    int cursorPosition = 0;
    string fontName = "Arial";
    int fontSize = 12;

    string getCurrentTimestamp() const {
        // Simplified timestamp
        static int counter = 0;
        return "T" + to_string(++counter);
    }

public:
    // --- Normal editor operations ---
    void type(const string& text) {
        content.insert(cursorPosition, text);
        cursorPosition += text.length();
    }

    void deleteBack(int count) {
        if (cursorPosition >= count) {
            content.erase(cursorPosition - count, count);
            cursorPosition -= count;
        }
    }

    void moveCursor(int position) {
        cursorPosition = max(0, min(position, (int)content.length()));
    }

    void setFont(const string& font, int size) {
        fontName = font;
        fontSize = size;
    }

    void print() const {
        cout << "Content: \"" << content << "\"" << endl;
        cout << "Cursor: " << cursorPosition
             << " | Font: " << fontName << " " << fontSize << "pt" << endl;
    }

    // --- Memento operations ---
    unique_ptr<EditorMemento> save() {
        cout << "  [Saving state...]" << endl;
        return unique_ptr<EditorMemento>(
            new EditorMemento(content, cursorPosition, getCurrentTimestamp())
        );
    }

    void restore(const EditorMemento& memento) {
        content = memento.content;
        cursorPosition = memento.cursorPosition;
        cout << "  [Restored to: " << memento.getDescription() << "]" << endl;
    }
};

// --- Caretaker ---
class EditorHistory {
    vector<unique_ptr<EditorMemento>> history;
    int currentIndex = -1;

public:
    void save(unique_ptr<EditorMemento> memento) {
        // Remove any "future" states (if we undid and then made new changes)
        while ((int)history.size() > currentIndex + 1) {
            history.pop_back();
        }
        history.push_back(std::move(memento));
        currentIndex = history.size() - 1;
    }

    const EditorMemento* undo() {
        if (currentIndex <= 0) {
            cout << "  Nothing to undo!" << endl;
            return nullptr;
        }
        currentIndex--;
        return history[currentIndex].get();
    }

    const EditorMemento* redo() {
        if (currentIndex >= (int)history.size() - 1) {
            cout << "  Nothing to redo!" << endl;
            return nullptr;
        }
        currentIndex++;
        return history[currentIndex].get();
    }

    void showHistory() const {
        cout << "\n--- History ---" << endl;
        for (int i = 0; i < (int)history.size(); i++) {
            cout << (i == currentIndex ? " → " : "   ")
                 << history[i]->getDescription() << endl;
        }
    }
};

// --- Client code ---
int main() {
    TextEditor editor;
    EditorHistory history;

    // Save initial state
    history.save(editor.save());

    // Type some text
    editor.type("Hello");
    editor.print();
    history.save(editor.save());

    editor.type(" World");
    editor.print();
    history.save(editor.save());

    editor.type("! How are you?");
    editor.print();
    history.save(editor.save());

    history.showHistory();

    // Undo
    cout << "\n--- Undo ---" << endl;
    if (auto* memento = history.undo()) {
        editor.restore(*memento);
        editor.print();
    }

    cout << "\n--- Undo again ---" << endl;
    if (auto* memento = history.undo()) {
        editor.restore(*memento);
        editor.print();
    }

    // Redo
    cout << "\n--- Redo ---" << endl;
    if (auto* memento = history.redo()) {
        editor.restore(*memento);
        editor.print();
    }

    // New action after undo (discards redo history)
    cout << "\n--- New action after undo ---" << endl;
    editor.type(" C++");
    editor.print();
    history.save(editor.save());

    history.showHistory();

    return 0;
}
```

---

### Game Save System Example

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <unordered_map>
using namespace std;

// --- Memento ---
class GameSave {
    friend class GameCharacter;

    int health;
    int mana;
    int level;
    int experience;
    string location;
    unordered_map<string, int> inventory;
    string saveName;
    string timestamp;

    GameSave(int hp, int mp, int lvl, int xp, const string& loc,
             const unordered_map<string, int>& inv, const string& name, const string& ts)
        : health(hp), mana(mp), level(lvl), experience(xp),
          location(loc), inventory(inv), saveName(name), timestamp(ts) {}

public:
    string getSaveName() const { return saveName; }
    string getTimestamp() const { return timestamp; }
    string getDescription() const {
        return saveName + " [" + timestamp + "] — Lvl " + to_string(level)
               + " at " + location + " (HP:" + to_string(health) + ")";
    }
};

// --- Originator ---
class GameCharacter {
    string name;
    int health, maxHealth;
    int mana, maxMana;
    int level;
    int experience;
    string location;
    unordered_map<string, int> inventory;

    static int saveCounter;

public:
    GameCharacter(const string& n)
        : name(n), health(100), maxHealth(100), mana(50), maxMana(50),
          level(1), experience(0), location("Starting Village") {}

    // Game actions
    void takeDamage(int dmg) {
        health = max(0, health - dmg);
        cout << name << " takes " << dmg << " damage! HP: " << health << "/" << maxHealth << endl;
    }

    void heal(int amount) {
        health = min(maxHealth, health + amount);
        cout << name << " heals " << amount << "! HP: " << health << "/" << maxHealth << endl;
    }

    void gainExperience(int xp) {
        experience += xp;
        cout << name << " gains " << xp << " XP! Total: " << experience << endl;
        while (experience >= level * 100) {
            experience -= level * 100;
            level++;
            maxHealth += 20;
            health = maxHealth;
            maxMana += 10;
            mana = maxMana;
            cout << "  *** LEVEL UP! Now level " << level << " ***" << endl;
        }
    }

    void moveTo(const string& loc) {
        location = loc;
        cout << name << " moves to " << location << endl;
    }

    void addItem(const string& item, int count = 1) {
        inventory[item] += count;
        cout << name << " obtained " << item << " x" << count << endl;
    }

    void print() const {
        cout << "\n=== " << name << " ===" << endl;
        cout << "Level: " << level << " | XP: " << experience << endl;
        cout << "HP: " << health << "/" << maxHealth
             << " | MP: " << mana << "/" << maxMana << endl;
        cout << "Location: " << location << endl;
        cout << "Inventory: ";
        for (const auto& [item, count] : inventory) {
            cout << item << "x" << count << " ";
        }
        cout << endl;
    }

    // --- Memento operations ---
    unique_ptr<GameSave> saveGame(const string& saveName = "") {
        string sName = saveName.empty() ? "Save_" + to_string(++saveCounter) : saveName;
        string ts = "2026-04-21 " + to_string(10 + saveCounter) + ":00";
        cout << "\n[Game saved: " << sName << "]" << endl;
        return unique_ptr<GameSave>(
            new GameSave(health, mana, level, experience, location, inventory, sName, ts)
        );
    }

    void loadGame(const GameSave& save) {
        health = save.health;
        mana = save.mana;
        level = save.level;
        experience = save.experience;
        location = save.location;
        inventory = save.inventory;
        cout << "\n[Game loaded: " << save.getSaveName() << "]" << endl;
    }
};

int GameCharacter::saveCounter = 0;

// --- Caretaker ---
class SaveManager {
    vector<unique_ptr<GameSave>> saves;
public:
    void addSave(unique_ptr<GameSave> save) {
        saves.push_back(std::move(save));
    }

    const GameSave* getSave(int index) const {
        if (index >= 0 && index < (int)saves.size()) {
            return saves[index].get();
        }
        return nullptr;
    }

    const GameSave* getLatestSave() const {
        return saves.empty() ? nullptr : saves.back().get();
    }

    void listSaves() const {
        cout << "\n--- Save Files ---" << endl;
        for (int i = 0; i < (int)saves.size(); i++) {
            cout << "  [" << i << "] " << saves[i]->getDescription() << endl;
        }
    }
};

// --- Client code ---
int main() {
    GameCharacter hero("Warrior");
    SaveManager saveManager;

    // Play the game
    hero.moveTo("Dark Forest");
    hero.gainExperience(80);
    hero.addItem("Iron Sword");
    saveManager.addSave(hero.saveGame("Before Boss"));

    // Fight a boss (things go badly)
    hero.takeDamage(60);
    hero.takeDamage(30);
    hero.print();

    // Load the save!
    cout << "\n--- Loading save... ---" << endl;
    if (auto* save = saveManager.getLatestSave()) {
        hero.loadGame(*save);
        hero.print();
    }

    // Try again with better strategy
    hero.gainExperience(120);  // level up first!
    hero.addItem("Health Potion", 3);
    saveManager.addSave(hero.saveGame("After Level Up"));

    hero.takeDamage(40);
    hero.heal(50);
    hero.gainExperience(200);
    hero.moveTo("Castle");
    saveManager.addSave(hero.saveGame("Victory"));

    saveManager.listSaves();

    return 0;
}
```

---

### When to Use Memento

| Scenario | Why Memento Helps |
|----------|------------------|
| Undo/Redo functionality | Save state before each operation, restore on undo |
| Checkpointing / Save games | Capture full state at specific points |
| Transaction rollback | Save state before transaction, restore on failure |
| Snapshot for comparison | Compare current state with a previous snapshot |
| Preserving encapsulation | State is saved without exposing internals |

**Memento vs Command (for undo):**

| Aspect | Memento | Command |
|--------|---------|---------|
| What's stored | Complete state snapshot | Operation + inverse operation |
| Memory usage | Higher (full state per save) | Lower (just the operation) |
| Complexity | Simpler (just save/restore) | More complex (must implement undo logic) |
| Granularity | Entire object state | Individual operations |
| Best for | Complex state, hard to compute inverse | Simple operations with clear inverses |

**Pitfalls:**
- **Memory consumption:** Storing full state snapshots can be expensive for large objects
- **Serialization:** Complex objects with pointers/references are hard to snapshot
- **Frequency:** Saving too often wastes memory; too rarely loses granularity

---


## 6.11 Interpreter Pattern

> The Interpreter pattern defines a representation for a language's grammar along with an interpreter that uses the representation to interpret sentences in the language. It maps a domain-specific language (DSL) to a class hierarchy where each rule in the grammar becomes a class.

---

### Why Interpreter?

When you have a simple language or grammar that needs to be evaluated:
- Mathematical expressions: `3 + 4 * 2`
- Boolean expressions: `(true AND false) OR true`
- SQL-like queries: `SELECT name WHERE age > 25`
- Configuration rules: `IF user.role == "admin" THEN allow`
- Regular expressions (conceptually)

**The problem without Interpreter:**
```cpp
// BAD: Hardcoded evaluation logic — can't extend the language
double evaluate(const string& expression) {
    // Must parse and evaluate in one monolithic function
    // Adding new operators means rewriting the parser
    // No reusable grammar representation
    // Can't compose expressions dynamically
}
```

---

### Structure

```
┌──────────────────────────┐
│  AbstractExpression       │
├──────────────────────────┤
│ + interpret(context)      │
└──────────────────────────┘
             ▲
             │
    ┌────────┴────────┐
    │                 │
TerminalExpression  NonTerminalExpression
(leaf: number,      (composite: add, multiply,
 variable, true)     and, or, if-then)
```

**Participants:**
- **AbstractExpression:** Declares `interpret()` method
- **TerminalExpression:** Implements interpret for terminal symbols (literals, variables)
- **NonTerminalExpression:** Implements interpret for grammar rules (operators, compounds)
- **Context:** Contains information that's global to the interpreter (variable values, etc.)

---

### Simple Expression Evaluator

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <unordered_map>
#include <stack>
#include <sstream>
using namespace std;

// --- Context ---
class Context {
    unordered_map<string, double> variables;
public:
    void setVariable(const string& name, double value) {
        variables[name] = value;
    }

    double getVariable(const string& name) const {
        auto it = variables.find(name);
        if (it == variables.end()) {
            throw runtime_error("Undefined variable: " + name);
        }
        return it->second;
    }

    bool hasVariable(const string& name) const {
        return variables.count(name) > 0;
    }
};

// --- Abstract Expression ---
class Expression {
public:
    virtual double interpret(const Context& context) const = 0;
    virtual string toString() const = 0;
    virtual ~Expression() = default;
};

// --- Terminal Expressions ---
class NumberExpression : public Expression {
    double value;
public:
    NumberExpression(double v) : value(v) {}

    double interpret(const Context& context) const override {
        return value;
    }

    string toString() const override {
        return to_string((int)value);
    }
};

class VariableExpression : public Expression {
    string name;
public:
    VariableExpression(const string& n) : name(n) {}

    double interpret(const Context& context) const override {
        return context.getVariable(name);
    }

    string toString() const override {
        return name;
    }
};

// --- Non-Terminal Expressions (Operators) ---
class AddExpression : public Expression {
    unique_ptr<Expression> left, right;
public:
    AddExpression(unique_ptr<Expression> l, unique_ptr<Expression> r)
        : left(std::move(l)), right(std::move(r)) {}

    double interpret(const Context& context) const override {
        return left->interpret(context) + right->interpret(context);
    }

    string toString() const override {
        return "(" + left->toString() + " + " + right->toString() + ")";
    }
};

class SubtractExpression : public Expression {
    unique_ptr<Expression> left, right;
public:
    SubtractExpression(unique_ptr<Expression> l, unique_ptr<Expression> r)
        : left(std::move(l)), right(std::move(r)) {}

    double interpret(const Context& context) const override {
        return left->interpret(context) - right->interpret(context);
    }

    string toString() const override {
        return "(" + left->toString() + " - " + right->toString() + ")";
    }
};

class MultiplyExpression : public Expression {
    unique_ptr<Expression> left, right;
public:
    MultiplyExpression(unique_ptr<Expression> l, unique_ptr<Expression> r)
        : left(std::move(l)), right(std::move(r)) {}

    double interpret(const Context& context) const override {
        return left->interpret(context) * right->interpret(context);
    }

    string toString() const override {
        return "(" + left->toString() + " * " + right->toString() + ")";
    }
};

class DivideExpression : public Expression {
    unique_ptr<Expression> left, right;
public:
    DivideExpression(unique_ptr<Expression> l, unique_ptr<Expression> r)
        : left(std::move(l)), right(std::move(r)) {}

    double interpret(const Context& context) const override {
        double divisor = right->interpret(context);
        if (divisor == 0) throw runtime_error("Division by zero");
        return left->interpret(context) / divisor;
    }

    string toString() const override {
        return "(" + left->toString() + " / " + right->toString() + ")";
    }
};

// --- Build and evaluate expressions ---
int main() {
    Context context;
    context.setVariable("x", 10);
    context.setVariable("y", 5);
    context.setVariable("z", 3);

    // Build expression: (x + y) * z = (10 + 5) * 3 = 45
    auto expr1 = make_unique<MultiplyExpression>(
        make_unique<AddExpression>(
            make_unique<VariableExpression>("x"),
            make_unique<VariableExpression>("y")
        ),
        make_unique<VariableExpression>("z")
    );

    cout << expr1->toString() << " = " << expr1->interpret(context) << endl;
    // ((x + y) * z) = 45

    // Build expression: x * y - z / 2 = 10 * 5 - 3 / 2 = 48.5
    auto expr2 = make_unique<SubtractExpression>(
        make_unique<MultiplyExpression>(
            make_unique<VariableExpression>("x"),
            make_unique<VariableExpression>("y")
        ),
        make_unique<DivideExpression>(
            make_unique<VariableExpression>("z"),
            make_unique<NumberExpression>(2)
        )
    );

    cout << expr2->toString() << " = " << expr2->interpret(context) << endl;

    // Change variables and re-evaluate
    context.setVariable("x", 20);
    cout << "\nAfter x = 20:" << endl;
    cout << expr1->toString() << " = " << expr1->interpret(context) << endl;
    // ((x + y) * z) = (20 + 5) * 3 = 75

    return 0;
}
```

---

### Boolean Expression Interpreter

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <unordered_map>
using namespace std;

// --- Context ---
class BoolContext {
    unordered_map<string, bool> variables;
public:
    void set(const string& name, bool value) { variables[name] = value; }
    bool get(const string& name) const {
        auto it = variables.find(name);
        if (it == variables.end()) throw runtime_error("Undefined: " + name);
        return it->second;
    }
};

// --- Abstract Expression ---
class BoolExpression {
public:
    virtual bool interpret(const BoolContext& ctx) const = 0;
    virtual string toString() const = 0;
    virtual ~BoolExpression() = default;
};

// --- Terminal ---
class BoolConstant : public BoolExpression {
    bool value;
public:
    BoolConstant(bool v) : value(v) {}
    bool interpret(const BoolContext& ctx) const override { return value; }
    string toString() const override { return value ? "true" : "false"; }
};

class BoolVariable : public BoolExpression {
    string name;
public:
    BoolVariable(const string& n) : name(n) {}
    bool interpret(const BoolContext& ctx) const override { return ctx.get(name); }
    string toString() const override { return name; }
};

// --- Non-Terminal ---
class AndExpression : public BoolExpression {
    unique_ptr<BoolExpression> left, right;
public:
    AndExpression(unique_ptr<BoolExpression> l, unique_ptr<BoolExpression> r)
        : left(std::move(l)), right(std::move(r)) {}
    bool interpret(const BoolContext& ctx) const override {
        return left->interpret(ctx) && right->interpret(ctx);
    }
    string toString() const override {
        return "(" + left->toString() + " AND " + right->toString() + ")";
    }
};

class OrExpression : public BoolExpression {
    unique_ptr<BoolExpression> left, right;
public:
    OrExpression(unique_ptr<BoolExpression> l, unique_ptr<BoolExpression> r)
        : left(std::move(l)), right(std::move(r)) {}
    bool interpret(const BoolContext& ctx) const override {
        return left->interpret(ctx) || right->interpret(ctx);
    }
    string toString() const override {
        return "(" + left->toString() + " OR " + right->toString() + ")";
    }
};

class NotExpression : public BoolExpression {
    unique_ptr<BoolExpression> operand;
public:
    NotExpression(unique_ptr<BoolExpression> op) : operand(std::move(op)) {}
    bool interpret(const BoolContext& ctx) const override {
        return !operand->interpret(ctx);
    }
    string toString() const override {
        return "(NOT " + operand->toString() + ")";
    }
};

// --- Client code ---
int main() {
    BoolContext ctx;
    ctx.set("isAdmin", true);
    ctx.set("isLoggedIn", true);
    ctx.set("isBanned", false);

    // Rule: (isLoggedIn AND isAdmin) AND (NOT isBanned)
    auto rule = make_unique<AndExpression>(
        make_unique<AndExpression>(
            make_unique<BoolVariable>("isLoggedIn"),
            make_unique<BoolVariable>("isAdmin")
        ),
        make_unique<NotExpression>(
            make_unique<BoolVariable>("isBanned")
        )
    );

    cout << "Rule: " << rule->toString() << endl;
    cout << "Result: " << (rule->interpret(ctx) ? "ALLOWED" : "DENIED") << endl;
    // Result: ALLOWED

    // Change context
    ctx.set("isBanned", true);
    cout << "\nAfter banning:" << endl;
    cout << "Result: " << (rule->interpret(ctx) ? "ALLOWED" : "DENIED") << endl;
    // Result: DENIED

    return 0;
}
```

---

### Abstract Syntax Tree

The Interpreter pattern naturally creates an AST where:
- **Leaf nodes** are terminal expressions (numbers, variables, constants)
- **Internal nodes** are non-terminal expressions (operators, compounds)

```
Expression: (x + 5) * (y - 2)

        [*]
       /   \
     [+]   [-]
    /   \  /   \
  [x]  [5] [y]  [2]

Each node is an Expression object with an interpret() method.
The tree is evaluated bottom-up: leaves return values,
internal nodes combine their children's results.
```

---

### When to Use Interpreter

| Scenario | Why Interpreter Helps |
|----------|----------------------|
| Simple grammar that needs evaluation | Maps grammar rules to classes |
| Domain-specific languages (DSLs) | Each rule is a class, easy to extend |
| Expression evaluation | Natural tree structure for expressions |
| Configuration rules | Boolean/conditional rules as expression trees |
| Query languages | Simple SQL-like or filter expressions |

**When NOT to use Interpreter:**
- Complex grammars — use parser generators (ANTLR, Bison) instead
- Performance-critical evaluation — interpreter overhead is significant
- Grammar changes frequently — class hierarchy must change with grammar

**Trade-offs:**

| Advantage | Disadvantage |
|-----------|-------------|
| Easy to implement simple grammars | Class explosion for complex grammars |
| Easy to extend (add new expressions) | Slow for complex expressions |
| Grammar rules are explicit in code | Hard to maintain for large grammars |
| Can compose expressions dynamically | Better alternatives exist for complex languages |

---


## 6.12 Null Object Pattern

> The Null Object pattern provides an object with defined neutral ("do nothing") behavior as a surrogate for the absence of an object. Instead of using `nullptr` (or `NULL`) to represent the absence of an object, you use a special object that implements the expected interface but does nothing. This eliminates null checks throughout the code.

---

### Why Null Object?

Null references are a constant source of bugs and defensive code:
- Forgetting a null check causes crashes (segfault, NullPointerException)
- Null checks clutter the code and obscure the real logic
- Every consumer of an object must independently check for null
- Tony Hoare called null references his "billion-dollar mistake"

**The problem without Null Object:**
```cpp
// BAD: Null checks everywhere!
class OrderService {
    Logger* logger;       // might be null!
    Notifier* notifier;   // might be null!
    Auditor* auditor;     // might be null!

public:
    void processOrder(const Order& order) {
        // Must check EVERY dependency for null before using it
        if (logger != nullptr) {
            logger->log("Processing order " + order.id);
        }

        // ... actual business logic ...

        if (notifier != nullptr) {
            notifier->notify("Order processed: " + order.id);
        }

        if (auditor != nullptr) {
            auditor->record("ORDER_PROCESSED", order.id);
        }

        // Null checks everywhere! Easy to forget one → crash!
        // Business logic is buried under defensive code
    }
};
```

---

### Structure

```
┌──────────────────────────┐
│   Interface               │
├──────────────────────────┤
│ + operation()             │
└──────────────────────────┘
             ▲
             │
    ┌────────┴────────┐
    │                 │
RealObject        NullObject
+ operation()     + operation()
  (does work)       (does nothing)
```

---

### Basic Implementation

```cpp
#include <iostream>
#include <string>
#include <memory>
using namespace std;

// --- Interface ---
class Logger {
public:
    virtual void log(const string& message) = 0;
    virtual void warn(const string& message) = 0;
    virtual void error(const string& message) = 0;
    virtual bool isNull() const { return false; }
    virtual ~Logger() = default;
};

// --- Real Implementation ---
class ConsoleLogger : public Logger {
public:
    void log(const string& message) override {
        cout << "[LOG] " << message << endl;
    }
    void warn(const string& message) override {
        cout << "[WARN] " << message << endl;
    }
    void error(const string& message) override {
        cout << "[ERROR] " << message << endl;
    }
};

class FileLogger : public Logger {
    string filename;
public:
    FileLogger(const string& file) : filename(file) {}
    void log(const string& message) override {
        cout << "[FILE:" << filename << "] " << message << endl;
    }
    void warn(const string& message) override {
        cout << "[FILE:" << filename << " WARN] " << message << endl;
    }
    void error(const string& message) override {
        cout << "[FILE:" << filename << " ERROR] " << message << endl;
    }
};

// --- Null Object ---
class NullLogger : public Logger {
public:
    void log(const string& message) override {
        // intentionally empty — does nothing
    }
    void warn(const string& message) override {
        // intentionally empty
    }
    void error(const string& message) override {
        // intentionally empty
    }
    bool isNull() const override { return true; }
};

// --- Service that uses Logger ---
class OrderService {
    Logger& logger;  // reference — never null!

public:
    // Always receives a valid logger (real or null)
    OrderService(Logger& log) : logger(log) {}

    void processOrder(const string& orderId) {
        // NO null checks needed! Just use the logger directly.
        logger.log("Processing order: " + orderId);

        // ... business logic ...
        logger.log("Validating payment...");
        logger.log("Updating inventory...");

        logger.log("Order " + orderId + " processed successfully");
    }
};

// --- Client code ---
int main() {
    // With real logger — logs everything
    ConsoleLogger consoleLogger;
    OrderService service1(consoleLogger);
    cout << "=== With Console Logger ===" << endl;
    service1.processOrder("ORD-001");

    cout << endl;

    // With null logger — silently does nothing
    NullLogger nullLogger;
    OrderService service2(nullLogger);
    cout << "=== With Null Logger ===" << endl;
    service2.processOrder("ORD-002");
    cout << "(no output — NullLogger does nothing)" << endl;

    return 0;
}
```

---

### Comprehensive Example — Notification System

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <vector>
using namespace std;

// --- Notification Interface ---
class Notifier {
public:
    virtual void send(const string& recipient, const string& message) = 0;
    virtual string getType() const = 0;
    virtual bool isNull() const { return false; }
    virtual ~Notifier() = default;
};

// --- Real Implementations ---
class EmailNotifier : public Notifier {
public:
    void send(const string& recipient, const string& message) override {
        cout << "  EMAIL → " << recipient << ": " << message << endl;
    }
    string getType() const override { return "Email"; }
};

class SMSNotifier : public Notifier {
public:
    void send(const string& recipient, const string& message) override {
        cout << "  SMS → " << recipient << ": " << message << endl;
    }
    string getType() const override { return "SMS"; }
};

class PushNotifier : public Notifier {
public:
    void send(const string& recipient, const string& message) override {
        cout << "  PUSH → " << recipient << ": " << message << endl;
    }
    string getType() const override { return "Push"; }
};

// --- Null Object ---
class NullNotifier : public Notifier {
public:
    void send(const string& recipient, const string& message) override {
        // intentionally empty — notification disabled
    }
    string getType() const override { return "None (disabled)"; }
    bool isNull() const override { return true; }
};

// --- Discount Strategy Interface ---
class DiscountStrategy {
public:
    virtual double apply(double price) const = 0;
    virtual string description() const = 0;
    virtual ~DiscountStrategy() = default;
};

class PercentageDiscount : public DiscountStrategy {
    double percent;
public:
    PercentageDiscount(double p) : percent(p) {}
    double apply(double price) const override { return price * (1 - percent / 100); }
    string description() const override { return to_string((int)percent) + "% off"; }
};

// --- Null Discount (no discount applied) ---
class NullDiscount : public DiscountStrategy {
public:
    double apply(double price) const override { return price; }  // returns price unchanged
    string description() const override { return "No discount"; }
};

// --- User with optional features ---
class User {
    string name;
    string email;
    unique_ptr<Notifier> notifier;
    unique_ptr<DiscountStrategy> discount;

public:
    User(const string& n, const string& e,
         unique_ptr<Notifier> notif = nullptr,
         unique_ptr<DiscountStrategy> disc = nullptr)
        : name(n), email(e)
    {
        // Use Null Objects instead of nullptr
        notifier = notif ? std::move(notif) : make_unique<NullNotifier>();
        discount = disc ? std::move(disc) : make_unique<NullDiscount>();
    }

    void placeOrder(const string& item, double price) {
        double finalPrice = discount->apply(price);

        cout << name << " orders " << item << ":" << endl;
        cout << "  Original: $" << price << endl;
        cout << "  Discount: " << discount->description() << endl;
        cout << "  Final: $" << finalPrice << endl;

        // No null check needed! NullNotifier just does nothing.
        notifier->send(email, "Order confirmed: " + item + " for $" + to_string(finalPrice));
    }
};

// --- Client code ---
int main() {
    // Premium user with email notifications and discount
    User premium("Alice", "alice@example.com",
        make_unique<EmailNotifier>(),
        make_unique<PercentageDiscount>(20));

    // Basic user with no notifications and no discount
    // NullNotifier and NullDiscount are used automatically
    User basic("Bob", "bob@example.com");

    // User with SMS notifications but no discount
    User smsUser("Charlie", "charlie@example.com",
        make_unique<SMSNotifier>());

    cout << "=== Premium User ===" << endl;
    premium.placeOrder("Laptop", 999.99);

    cout << "\n=== Basic User ===" << endl;
    basic.placeOrder("Mouse", 29.99);

    cout << "\n=== SMS User ===" << endl;
    smsUser.placeOrder("Keyboard", 79.99);

    return 0;
}
```

---

### Null Object vs nullptr

| Aspect | nullptr | Null Object |
|--------|---------|-------------|
| Safety | Crashes if not checked | Always safe to call methods |
| Code clarity | Null checks clutter code | Clean, no defensive code |
| Polymorphism | No — null is not polymorphic | Yes — implements the interface |
| Default behavior | None — must handle explicitly | Defined "do nothing" behavior |
| Debugging | NullPointerException at runtime | Silent no-op (can be harder to debug) |
| Type safety | Weakly typed (any pointer can be null) | Strongly typed (implements interface) |

---

### When to Use Null Object

| Scenario | Why Null Object Helps |
|----------|----------------------|
| Optional dependencies | Provide null implementation instead of nullptr |
| Default behavior when feature is disabled | Null object does nothing gracefully |
| Eliminating null checks | Code is cleaner and less error-prone |
| Strategy pattern with "no strategy" option | NullStrategy returns input unchanged |
| Logging that can be disabled | NullLogger silently discards messages |
| Plugin systems with optional plugins | NullPlugin does nothing |

**Pitfalls:**
- **Silent failures:** Null objects hide the absence of real behavior — bugs can go unnoticed
- **Debugging difficulty:** Hard to tell if a null object is being used unintentionally
- **Not always appropriate:** Sometimes null genuinely means "error" and should be handled explicitly
- **`isNull()` method:** Adding `isNull()` partially defeats the purpose — try to avoid checking it

**Best practice:** Use Null Object for optional, non-critical behavior (logging, notifications, metrics). Use explicit error handling (exceptions, `std::optional`) for required behavior where absence is an error.

---
