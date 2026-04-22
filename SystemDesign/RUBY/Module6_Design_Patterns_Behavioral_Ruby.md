# Module 6: Design Patterns — Behavioral (Ruby)

> Behavioral patterns deal with communication between objects — they define how objects interact, distribute responsibility, and coordinate workflows. Unlike creational patterns (which handle object creation) or structural patterns (which handle object composition), behavioral patterns focus on **algorithms and the assignment of responsibilities** between objects. They help make complex control flows understandable by encapsulating behavior in objects and letting you vary it independently. Ruby's dynamic nature — duck typing, blocks, Procs, lambdas, modules, and `method_missing` — makes many behavioral patterns more concise and idiomatic than in statically typed languages.

---

## 6.1 Strategy Pattern

> The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. It lets the algorithm vary independently from the clients that use it. The client selects which algorithm to use at runtime, without knowing the implementation details.

---

### Why Strategy?

Many situations require choosing between different algorithms or behaviors at runtime:
- Sorting: quicksort vs mergesort vs heapsort depending on data characteristics
- Payment: credit card vs PayPal vs crypto vs bank transfer
- Compression: gzip vs brotli vs lz4 depending on speed/size tradeoff
- Navigation: shortest path vs fastest route vs avoid tolls
- Pricing: regular vs premium vs student discount

**The problem without Strategy:**
```ruby
# BAD: Conditional logic scattered everywhere — violates OCP and SRP
class ShippingCalculator
  def calculate(method, weight, distance)
    case method
    when "ground"
      weight * 1.5 + distance * 0.5
    when "air"
      weight * 3.0 + distance * 1.2
    when "express"
      weight * 5.0 + distance * 2.0 + 10.0  # surcharge
    when "drone"
      # New method — must modify this class!
      weight * 4.0 + distance * 0.8
    else
      raise ArgumentError, "Unknown method: #{method}"
    end
    # Every new shipping method means modifying this class
    # Violates Open/Closed Principle!
    # Testing requires testing ALL branches in one class
  end
end
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
│      Context          │        │   Strategy (duck type)   │
├──────────────────────┤        ├─────────────────────────┤
│ - strategy            │───────→│ + execute(data): result  │
├──────────────────────┤        └─────────────────────────┘
│ + strategy=           │                    ▲
│ + execute_strategy    │                    │
└──────────────────────┘         ┌───────────┼───────────┐
                                 │           │           │
                          ConcreteA    ConcreteB    ConcreteC
                          + execute()  + execute()  + execute()
```

**Participants:**
- **Strategy (duck type):** Any object responding to the expected method (no formal interface needed in Ruby)
- **ConcreteStrategy:** Implements a specific algorithm
- **Context:** Maintains a reference to a Strategy object; delegates work to it

---

### Basic Implementation — Sorting Strategies

```ruby
# --- Concrete Strategies (duck typing — no interface needed) ---
class BubbleSort
  def sort(data)
    arr = data.dup
    n = arr.size
    (n - 1).times do |i|
      (n - i - 1).times do |j|
        arr[j], arr[j + 1] = arr[j + 1], arr[j] if arr[j] > arr[j + 1]
      end
    end
    arr
  end

  def name = "BubbleSort"
end

class QuickSort
  def sort(data)
    return data if data.size <= 1

    pivot = data[data.size / 2]
    left  = data.select { |x| x < pivot }
    mid   = data.select { |x| x == pivot }
    right = data.select { |x| x > pivot }
    sort(left) + mid + sort(right)
  end

  def name = "QuickSort"
end

class MergeSort
  def sort(data)
    return data if data.size <= 1

    mid = data.size / 2
    left  = sort(data[0...mid])
    right = sort(data[mid..])
    merge(left, right)
  end

  def name = "MergeSort"

  private

  def merge(left, right)
    result = []
    until left.empty? || right.empty?
      result << (left.first <= right.first ? left.shift : right.shift)
    end
    result + left + right
  end
end

# --- Context ---
class Sorter
  attr_accessor :strategy

  def initialize(strategy = nil)
    @strategy = strategy
  end

  def sort(data)
    raise "No sorting strategy set!" unless @strategy

    puts "Sorting with #{@strategy.name}..."
    @strategy.sort(data)
  end
end

# --- Client code ---
data = [38, 27, 43, 3, 9, 82, 10]

sorter = Sorter.new

# Use BubbleSort for small data
sorter.strategy = BubbleSort.new
p sorter.sort(data)  # => [3, 9, 10, 27, 38, 43, 82]

# Switch to QuickSort for larger data
sorter.strategy = QuickSort.new
p sorter.sort(data)

# Switch to MergeSort when stability matters
sorter.strategy = MergeSort.new
p sorter.sort(data)
```

---

### Real-World Example — Payment Processing

```ruby
# --- Concrete Strategies ---
class CreditCardPayment
  attr_reader :card_number, :expiry, :cvv

  def initialize(card_number:, expiry:, cvv:)
    @card_number = card_number
    @expiry = expiry
    @cvv = cvv
  end

  def validate?
    return false if card_number.length < 13 || card_number.length > 19
    return false unless [3, 4].include?(cvv.length)

    puts "  Credit card validated (Luhn check passed)"
    true
  end

  def pay(amount)
    unless validate?
      puts "  Credit card validation failed!"
      return false
    end
    puts "  Paid $#{amount} via Credit Card ending in #{card_number[-4..]}"
    true
  end

  def method_name = "Credit Card"
end

class PayPalPayment
  def initialize(email:, password:)
    @email = email
    @password = password
  end

  def validate?
    return false unless @email.include?("@")

    puts "  PayPal account validated"
    true
  end

  def pay(amount)
    unless validate?
      puts "  PayPal validation failed!"
      return false
    end
    puts "  Paid $#{amount} via PayPal (#{@email})"
    true
  end

  def method_name = "PayPal"
end

class CryptoPayment
  def initialize(wallet_address:, currency:)
    @wallet_address = wallet_address
    @currency = currency
  end

  def validate?
    return false if @wallet_address.length < 26

    puts "  Crypto wallet validated (#{@currency})"
    true
  end

  def pay(amount)
    unless validate?
      puts "  Crypto wallet validation failed!"
      return false
    end
    crypto_amount = amount / 50_000.0
    puts "  Paid #{crypto_amount.round(6)} #{@currency} (≈$#{amount}) to wallet #{@wallet_address[0, 8]}..."
    true
  end

  def method_name = "Crypto (#{@currency})"
end

class BankTransferPayment
  def initialize(bank_name:, account_number:, routing_number:)
    @bank_name = bank_name
    @account_number = account_number
    @routing_number = routing_number
  end

  def validate?
    return false if @account_number.empty? || @routing_number.empty?

    puts "  Bank account validated (#{@bank_name})"
    true
  end

  def pay(amount)
    unless validate?
      puts "  Bank transfer validation failed!"
      return false
    end
    puts "  Transferred $#{amount} via #{@bank_name} (account ending #{@account_number[-4..]})"
    true
  end

  def method_name = "Bank Transfer (#{@bank_name})"
end

# --- Context ---
class PaymentProcessor
  attr_accessor :strategy

  def process_payment(amount)
    unless @strategy
      puts "No payment method selected!"
      return false
    end

    puts "Processing $#{amount} via #{@strategy.method_name}:"
    success = @strategy.pay(amount)
    puts success ? "  Payment successful!" : "  Payment failed!"
    success
  end
end

# --- Client code ---
processor = PaymentProcessor.new

processor.strategy = CreditCardPayment.new(
  card_number: "4111111111111111", expiry: "12/25", cvv: "123"
)
processor.process_payment(99.99)

puts

processor.strategy = PayPalPayment.new(email: "user@example.com", password: "secret")
processor.process_payment(49.99)

puts

processor.strategy = CryptoPayment.new(
  wallet_address: "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa", currency: "BTC"
)
processor.process_payment(250.00)
```

---

### Strategy with Blocks and Lambdas (Idiomatic Ruby Approach)

Ruby's blocks, Procs, and lambdas make lightweight strategies natural — no class hierarchy needed:

```ruby
# Strategy as Proc/Lambda — no class hierarchy needed
regular_pricing  = ->(price, qty) { price * qty }
premium_discount = ->(price, qty) { price * qty * 0.9 }
bulk_discount    = ->(price, qty) do
  case qty
  when 100.. then price * qty * 0.7   # 30% off
  when 50..  then price * qty * 0.8   # 20% off
  when 10..  then price * qty * 0.9   # 10% off
  else price * qty
  end
end
seasonal_sale = ->(price, qty) { price * qty * 0.75 }

# --- Context ---
class ShoppingCart
  attr_accessor :pricing_strategy

  Item = Struct.new(:name, :price, :quantity)

  def initialize(strategy = regular_pricing)
    @pricing_strategy = strategy
    @items = []
  end

  def add_item(name, price, quantity)
    @items << Item.new(name, price, quantity)
  end

  def total
    @items.sum { |item| @pricing_strategy.call(item.price, item.quantity) }
  end

  def print_receipt
    puts "=== Receipt ==="
    @items.each do |item|
      cost = @pricing_strategy.call(item.price, item.quantity)
      puts "#{item.name} x#{item.quantity} @ $#{item.price} = $#{cost}"
    end
    puts "Total: $#{total}"
  end
end

# --- Client code ---
cart = ShoppingCart.new(regular_pricing)
cart.add_item("Widget", 10.0, 5)
cart.add_item("Gadget", 25.0, 3)

puts "Regular pricing:"
cart.print_receipt

puts "\nPremium discount:"
cart.pricing_strategy = premium_discount
cart.print_receipt

puts "\nSeasonal sale:"
cart.pricing_strategy = seasonal_sale
cart.print_receipt

# Inline strategy — no class or variable needed
cart.pricing_strategy = ->(price, qty) { price * qty * 0.5 }
puts "\nFlash sale:"
cart.print_receipt
```

---

### Strategy with Blocks (Even More Idiomatic)

Ruby blocks let you pass strategies directly to methods:

```ruby
class TextFormatter
  def initialize
    @strategies = {}
    register_defaults
  end

  def register(name, &block)
    @strategies[name] = block
  end

  def format(text, style)
    strategy = @strategies[style]
    raise ArgumentError, "Unknown strategy: #{style}" unless strategy

    strategy.call(text)
  end

  private

  def register_defaults
    @strategies[:uppercase]  = ->(text) { text.upcase }
    @strategies[:lowercase]  = ->(text) { text.downcase }
    @strategies[:titlecase]  = ->(text) { text.split.map(&:capitalize).join(" ") }
    @strategies[:reverse]    = ->(text) { text.reverse }
  end
end

formatter = TextFormatter.new

puts formatter.format("hello world", :uppercase)   # HELLO WORLD
puts formatter.format("hello world", :titlecase)    # Hello World

# Add a custom strategy at runtime — OCP!
formatter.register(:leetspeak) do |text|
  text.tr("aeiosAEIOS", "43105431o5")
end
puts formatter.format("Strategy Pattern", :leetspeak)  # Str4t3gy P4tt3rn
```

**When to use classes vs lambdas vs blocks:**

| Aspect | Class Hierarchy | Lambda/Proc | Block |
|--------|----------------|-------------|-------|
| State | Strategies can hold complex state | Lambdas capture closure state | Captures closure state |
| Reusability | Easy to reuse across projects | Tied to specific signature | Single-use, inline |
| Extensibility | New strategies via new classes | New strategies via new lambdas | Passed at call site |
| Testing | Easy to mock/test individually | Harder to name and test | Hardest to test in isolation |
| Best for | Complex strategies with state | Simple, stateless algorithms | One-off customization |

---

### When to Use Strategy

| Scenario | Why Strategy Helps |
|----------|-------------------|
| Multiple algorithms for the same task | Encapsulate each, swap at runtime |
| Conditional logic selecting behavior | Replace if/else chains with polymorphism |
| Algorithm details should be hidden from client | Client only knows the interface (duck type) |
| Need to add new algorithms without modifying existing code | OCP compliance |
| Different behaviors needed in different contexts | Configure context with appropriate strategy |
| Testing individual algorithms in isolation | Each strategy is independently testable |

**Strategy vs Template Method:**

| Aspect | Strategy | Template Method |
|--------|----------|-----------------|
| Mechanism | Composition (has-a) | Inheritance (is-a) |
| Granularity | Entire algorithm is swapped | Only steps of algorithm vary |
| Runtime flexibility | Can change at runtime | Fixed at class definition time |
| Class count | More classes (one per strategy) | Fewer classes (subclasses) |
| Coupling | Loose (duck typing) | Tight (inheritance-based) |
| Ruby idiom | Lambdas, Procs, duck-typed objects | Subclasses overriding hook methods |

---


## 6.2 Observer Pattern

> The Observer pattern defines a one-to-many dependency between objects so that when one object (the Subject) changes state, all its dependents (Observers) are notified and updated automatically. It's the foundation of event-driven programming and the publish-subscribe model. Ruby's standard library includes an `Observable` module that provides this pattern out of the box.

---

### Why Observer?

When one object's state change should trigger updates in other objects, but you don't want tight coupling between them:
- Stock price changes → update portfolio display, trigger alerts, log history
- User posts a message → notify followers, update feed, send push notifications
- Temperature sensor reading → update display, check thresholds, log data
- Order status changes → notify customer, update dashboard, trigger shipping

**The problem without Observer:**
```ruby
# BAD: Tight coupling — WeatherStation knows about every display
class WeatherStation
  def initialize
    @temp_display = TemperatureDisplay.new
    @humidity_display = HumidityDisplay.new
    @forecast_display = ForecastDisplay.new
    @mobile_app = MobileApp.new
    # ... every new display requires modifying this class!
  end

  def set_measurements(temp, humidity, pressure)
    # Must update ALL displays manually — tightly coupled!
    @temp_display.update(temp)
    @humidity_display.update(humidity)
    @forecast_display.update(temp, humidity, pressure)
    @mobile_app.refresh(temp, humidity, pressure)
    # Adding a new display means modifying this class — violates OCP!
  end
end
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
│       Subject            │       │     Observer (duck type)  │
├──────────────────────────┤       ├──────────────────────────┤
│ - observers: Array       │       │ + update(data)           │
├──────────────────────────┤       └──────────────────────────┘
│ + attach(observer)       │                    ▲
│ + detach(observer)       │                    │
│ + notify                 │         ┌──────────┼──────────┐
└──────────────────────────┘         │          │          │
         │ notifies                Display   Logger    AlertSystem
         └──────────────────────→  + update() + update() + update()
```

**Participants:**
- **Subject (Observable):** Maintains a list of observers and notifies them of state changes
- **Observer:** Any object responding to `update` (duck typing — no formal interface needed)
- **ConcreteSubject:** Stores state of interest; sends notifications when state changes
- **ConcreteObserver:** Implements `update` to react to notifications

---

### Push Model Implementation

In the **push model**, the Subject sends detailed data to observers in the notification. Observers receive all the data they might need.

```ruby
# --- Subject (Observable) ---
class WeatherStation
  attr_reader :temperature, :humidity, :pressure

  def initialize
    @observers = []
    @temperature = 0
    @humidity = 0
    @pressure = 0
  end

  def attach(observer)
    @observers << observer unless @observers.include?(observer)
    puts "[WeatherStation] #{observer.name} subscribed."
  end

  def detach(observer)
    @observers.delete(observer)
    puts "[WeatherStation] #{observer.name} unsubscribed."
  end

  def notify
    puts "[WeatherStation] Notifying #{@observers.size} observers..."
    @observers.each do |observer|
      observer.update(@temperature, @humidity, @pressure)
    end
  end

  def set_measurements(temp, humidity, pressure)
    puts "\n[WeatherStation] New measurements: " \
         "temp=#{temp}°C, humidity=#{humidity}%, pressure=#{pressure} hPa"
    @temperature = temp
    @humidity = humidity
    @pressure = pressure
    notify
  end
end

# --- Concrete Observers ---
class TemperatureDisplay
  def name = "TemperatureDisplay"

  def update(temperature, humidity, pressure)
    puts "  [TempDisplay] Current temperature: #{temperature}°C"
  end
end

class StatisticsDisplay
  def initialize
    @min_temp = Float::INFINITY
    @max_temp = -Float::INFINITY
    @sum_temp = 0.0
    @num_readings = 0
  end

  def name = "StatisticsDisplay"

  def update(temperature, humidity, pressure)
    @sum_temp += temperature
    @num_readings += 1
    @min_temp = [@min_temp, temperature].min
    @max_temp = [@max_temp, temperature].max
    avg = (@sum_temp / @num_readings).round(1)
    puts "  [StatsDisplay] Avg/Min/Max: #{avg}/#{@min_temp}/#{@max_temp}°C"
  end
end

class AlertSystem
  def initialize(temp_threshold)
    @temp_threshold = temp_threshold
  end

  def name = "AlertSystem"

  def update(temperature, humidity, pressure)
    if temperature > @temp_threshold
      puts "  [ALERT] Temperature #{temperature}°C exceeds threshold #{@temp_threshold}°C!"
    end
    if humidity > 90
      puts "  [ALERT] High humidity: #{humidity}%!"
    end
  end
end

class ForecastDisplay
  def initialize
    @last_pressure = 0
  end

  def name = "ForecastDisplay"

  def update(temperature, humidity, pressure)
    forecast = if pressure > @last_pressure
                 "Improving weather on the way!"
               elsif pressure < @last_pressure
                 "Watch out for cooler, rainy weather."
               else
                 "More of the same."
               end
    puts "  [Forecast] #{forecast}"
    @last_pressure = pressure
  end
end

# --- Client code ---
station = WeatherStation.new

temp_display = TemperatureDisplay.new
stats_display = StatisticsDisplay.new
alert_system = AlertSystem.new(35.0)
forecast_display = ForecastDisplay.new

station.attach(temp_display)
station.attach(stats_display)
station.attach(alert_system)
station.attach(forecast_display)

station.set_measurements(28.5, 65.0, 1013.0)
station.set_measurements(32.0, 70.0, 1010.0)
station.set_measurements(37.0, 92.0, 1007.0)  # triggers alerts!

station.detach(temp_display)
station.set_measurements(25.0, 55.0, 1015.0)  # temp_display won't be notified
```

---

### Pull Model Implementation

In the **pull model**, the Subject sends minimal notification, and observers pull the data they need from the Subject.

```ruby
# --- Subject with Pull Model ---
class StockMarket
  def initialize
    @observers = []
    @prices = {}
  end

  def attach(observer)  = @observers << observer
  def detach(observer)  = @observers.delete(observer)

  def notify
    @observers.each(&:update)  # no data passed — observer pulls what it needs
  end

  def set_price(symbol, price)
    @prices[symbol] = price
    notify
  end

  def price(symbol)   = @prices.fetch(symbol, 0.0)
  def symbols         = @prices.keys
end

# --- Concrete Observer (Pull Model) ---
class PortfolioTracker
  def initialize(market)
    @market = market  # reference to subject — for pulling data
    @holdings = {}
  end

  def add_holding(symbol, qty)
    @holdings[symbol] = qty
  end

  def update
    total = @holdings.sum { |symbol, qty| @market.price(symbol) * qty }
    puts "[Portfolio] Total value: $#{total.round(2)}"
  end
end

class PriceAlertObserver
  def initialize(market, symbol, alert_price)
    @market = market
    @watch_symbol = symbol
    @alert_price = alert_price
  end

  def update
    current = @market.price(@watch_symbol)
    if current >= @alert_price
      puts "[ALERT] #{@watch_symbol} reached $#{current} (target: $#{@alert_price})"
    end
  end
end
```

---

### Push vs Pull Model

| Aspect | Push Model | Pull Model |
|--------|-----------|------------|
| Data flow | Subject pushes data to observers | Observers pull data from subject |
| Observer interface | `update(data1, data2, ...)` | `update` — no parameters |
| Coupling | Observer coupled to data format | Observer coupled to Subject interface |
| Efficiency | May send unnecessary data | Observer fetches only what it needs |
| Flexibility | Subject decides what to send | Observer decides what to read |
| Complexity | Simpler observer implementation | Observer needs reference to Subject |
| Best for | All observers need same data | Observers need different subsets of data |

---

### Ruby's Built-in Observable Module

Ruby's standard library provides `Observable` — a mixin that adds Subject behavior to any class:

```ruby
require 'observer'

class WeatherData
  include Observable

  attr_reader :temperature, :humidity, :pressure

  def set_measurements(temp, humidity, pressure)
    @temperature = temp
    @humidity = humidity
    @pressure = pressure
    changed           # mark as changed (required before notify)
    notify_observers(self)  # pass self so observers can pull data
  end
end

class CurrentConditionsDisplay
  def update(weather_data)
    puts "Current: #{weather_data.temperature}°C, " \
         "#{weather_data.humidity}% humidity"
  end
end

class HeatIndexDisplay
  def update(weather_data)
    t = weather_data.temperature
    h = weather_data.humidity
    # Simplified heat index
    heat_index = (t + 0.5 * (h - 50)).round(1)
    puts "Heat index: #{heat_index}°C"
  end
end

# --- Usage ---
weather = WeatherData.new

current_display = CurrentConditionsDisplay.new
heat_display = HeatIndexDisplay.new

weather.add_observer(current_display)
weather.add_observer(heat_display)

weather.set_measurements(28.5, 65.0, 1013.0)
# Current: 28.5°C, 65.0% humidity
# Heat index: 36.0°C

weather.delete_observer(heat_display)
weather.set_measurements(32.0, 70.0, 1010.0)
# Current: 32.0°C, 70.0% humidity
```

**What `include Observable` provides:**
- `add_observer(obj)` — subscribe an observer
- `delete_observer(obj)` — unsubscribe
- `delete_observers` — remove all observers
- `changed(state = true)` — mark the object as changed (must call before `notify_observers`)
- `notify_observers(*args)` — notify all observers by calling their `update` method
- `changed?` — check if the object has been marked as changed
- `count_observers` — number of registered observers

---

### Event System Implementation (Generic Observer with Blocks)

A more flexible event system using Ruby blocks and Procs:

```ruby
class EventEmitter
  def initialize
    @listeners = Hash.new { |h, k| h[k] = [] }
    @next_id = 0
  end

  # Subscribe to an event — returns handler ID for unsubscribing
  def on(event, &handler)
    id = @next_id += 1
    @listeners[event] << { id: id, handler: handler, once: false }
    id
  end

  # Subscribe for one-time notification
  def once(event, &handler)
    id = @next_id += 1
    @listeners[event] << { id: id, handler: handler, once: true }
    id
  end

  # Unsubscribe
  def off(event, id)
    @listeners[event].reject! { |entry| entry[:id] == id }
  end

  # Emit event to all subscribers
  def emit(event, *data)
    return unless @listeners.key?(event)

    # Copy to avoid mutation during iteration
    handlers = @listeners[event].dup
    handlers.each do |entry|
      entry[:handler].call(*data)
      off(event, entry[:id]) if entry[:once]
    end
  end

  def listener_count(event) = @listeners[event].size
end

# --- Usage ---
emitter = EventEmitter.new

id1 = emitter.on("user:login") { |data| puts "Logger: User logged in — #{data}" }
id2 = emitter.on("user:login") { |data| puts "Analytics: Track login — #{data}" }
emitter.on("user:logout") { |data| puts "Cleanup: Session ended — #{data}" }

# One-time handler
emitter.once("user:login") { |data| puts "Welcome bonus: First login! — #{data}" }

puts "--- First login ---"
emitter.emit("user:login", "user123")

puts "\n--- Second login ---"
emitter.emit("user:login", "user456")  # "once" handler won't fire

emitter.off("user:login", id1)
puts "\n--- Third login (after unsubscribe) ---"
emitter.emit("user:login", "user789")  # only id2 handler fires
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
- **Thread safety:** Concurrent attach/detach/notify needs synchronization (use `Mutex`)

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
```ruby
# BAD: Button is tightly coupled to specific actions
class Button
  def click
    # What does this button do? It's hardcoded!
    document.save  # Can't reuse Button for other actions
  end
end

# BAD: No way to undo operations
class TextEditor
  def bold = # apply bold
  def italic = # apply italic
  def delete_text = # delete selected text — how to undo?
  # No history, no undo, no redo
end
```

---

### Structure

```
┌──────────────┐     ┌──────────────────────┐     ┌──────────────┐
│   Invoker     │     │   Command (duck type) │     │   Receiver    │
├──────────────┤     ├──────────────────────┤     ├──────────────┤
│ - command    │────→│ + execute             │────→│ + action      │
├──────────────┤     │ + undo                │     └──────────────┘
│ + invoke     │     └──────────────────────┘
└──────────────┘              ▲
                              │
                   ┌──────────┼──────────┐
                   │          │          │
              InsertCmd    PasteCmd   DeleteCmd
              + execute    + execute   + execute
              + undo       + undo      + undo
```

**Participants:**
- **Command (duck type):** Any object responding to `execute` and optionally `undo`
- **ConcreteCommand:** Binds a Receiver to an action; implements `execute` by calling Receiver methods
- **Invoker:** Asks the command to carry out the request
- **Receiver:** Knows how to perform the actual work
- **Client:** Creates ConcreteCommand and sets its Receiver

---

### Basic Implementation — Text Editor with Undo/Redo

```ruby
# --- Receiver ---
class TextDocument
  attr_reader :content

  def initialize
    @content = ""
  end

  def insert_text(position, text)
    position = [0, [position, @content.length].min].max
    @content.insert(position, text)
  end

  def delete_text(position, length)
    @content.slice!(position, length) if position >= 0 && position < @content.length
  end

  def to_s = "Document: \"#{@content}\""
end

# --- Concrete Commands ---
class InsertCommand
  def initialize(document, position, text)
    @document = document
    @position = position
    @text = text
  end

  def execute
    @document.insert_text(@position, @text)
  end

  def undo
    @document.delete_text(@position, @text.length)
  end

  def description = "Insert \"#{@text}\" at position #{@position}"
end

class DeleteCommand
  def initialize(document, position, length)
    @document = document
    @position = position
    @length = length
    @deleted_text = nil
  end

  def execute
    @deleted_text = @document.content[@position, @length]
    @document.delete_text(@position, @length)
  end

  def undo
    @document.insert_text(@position, @deleted_text)
  end

  def description = "Delete #{@length} chars at position #{@position}"
end

class ReplaceCommand
  def initialize(document, position, length, new_text)
    @document = document
    @position = position
    @length = length
    @new_text = new_text
    @old_text = nil
  end

  def execute
    @old_text = @document.content[@position, @length]
    @document.delete_text(@position, @length)
    @document.insert_text(@position, @new_text)
  end

  def undo
    @document.delete_text(@position, @new_text.length)
    @document.insert_text(@position, @old_text)
  end

  def description = "Replace \"#{@old_text}\" with \"#{@new_text}\""
end

# --- Invoker with Undo/Redo ---
class TextEditor
  def initialize(document)
    @document = document
    @undo_stack = []
    @redo_stack = []
  end

  def execute_command(cmd)
    puts "Execute: #{cmd.description}"
    cmd.execute
    @undo_stack.push(cmd)
    @redo_stack.clear  # new action invalidates redo history
    puts @document
  end

  def undo
    if @undo_stack.empty?
      puts "Nothing to undo!"
      return
    end
    cmd = @undo_stack.pop
    puts "Undo: #{cmd.description}"
    cmd.undo
    @redo_stack.push(cmd)
    puts @document
  end

  def redo
    if @redo_stack.empty?
      puts "Nothing to redo!"
      return
    end
    cmd = @redo_stack.pop
    puts "Redo: #{cmd.description}"
    cmd.execute
    @undo_stack.push(cmd)
    puts @document
  end

  def undo_count = @undo_stack.size
  def redo_count = @redo_stack.size
end

# --- Client code ---
doc = TextDocument.new
editor = TextEditor.new(doc)

editor.execute_command(InsertCommand.new(doc, 0, "Hello"))
editor.execute_command(InsertCommand.new(doc, 5, " World"))
editor.execute_command(InsertCommand.new(doc, 11, "!"))
# Document: "Hello World!"

editor.undo  # removes "!"
editor.undo  # removes " World"
# Document: "Hello"

editor.redo  # re-adds " World"
# Document: "Hello World"

editor.execute_command(ReplaceCommand.new(doc, 6, 5, "Ruby"))
# Document: "Hello Ruby"

editor.undo  # undo replace
# Document: "Hello World"
```

---

### Macro Commands (Composite Command)

A macro command is a command that contains a sequence of other commands, executed as a single unit.

```ruby
class MacroCommand
  def initialize(name)
    @name = name
    @commands = []
  end

  def add_command(cmd)
    @commands << cmd
    self  # enable chaining
  end

  def execute
    puts "--- Executing macro: #{@name} ---"
    @commands.each(&:execute)
  end

  def undo
    puts "--- Undoing macro: #{@name} ---"
    @commands.reverse_each(&:undo)  # undo in reverse order!
  end

  def description = "Macro: #{@name} (#{@commands.size} commands)"
end

# --- Usage ---
doc = TextDocument.new
editor = TextEditor.new(doc)

template_macro = MacroCommand.new("Document Template")
  .add_command(InsertCommand.new(doc, 0, "Title: "))
  .add_command(InsertCommand.new(doc, 7, "Untitled"))
  .add_command(InsertCommand.new(doc, 15, "\n\nBody: "))
  .add_command(InsertCommand.new(doc, 23, "Enter text here..."))

editor.execute_command(template_macro)
# Document: "Title: Untitled\n\nBody: Enter text here..."

editor.undo  # undoes the ENTIRE macro in one step
# Document: ""
```

---

### Command with Blocks (Idiomatic Ruby)

For simple commands, Ruby blocks make the pattern lightweight:

```ruby
class BlockCommand
  def initialize(description, &execute_block)
    @description = description
    @execute_block = execute_block
    @undo_block = nil
  end

  def on_undo(&block)
    @undo_block = block
    self
  end

  def execute = @execute_block.call
  def undo    = @undo_block&.call
  def description = @description
end

# --- Usage ---
light_state = { on: false }

turn_on = BlockCommand.new("Turn on light") { light_state[:on] = true; puts "Light ON" }
  .on_undo { light_state[:on] = false; puts "Light OFF" }

turn_on.execute  # Light ON
turn_on.undo     # Light OFF
```

---

### Command Queue (Deferred Execution)

Commands can be queued for later execution, enabling task scheduling and batch processing.

```ruby
class CommandQueue
  def initialize
    @pending = []
    @executed = []
  end

  def enqueue(cmd)
    puts "Queued: #{cmd.description}"
    @pending << cmd
  end

  def process_next
    if @pending.empty?
      puts "Queue is empty!"
      return
    end
    cmd = @pending.shift
    puts "Processing: #{cmd.description}"
    cmd.execute
    @executed << cmd
  end

  def process_all
    puts "Processing #{@pending.size} commands..."
    process_next until @pending.empty?
    puts "All commands processed."
  end

  def undo_last
    if @executed.empty?
      puts "No commands to undo!"
      return
    end
    cmd = @executed.pop
    puts "Undoing: #{cmd.description}"
    cmd.undo
  end

  def pending_count  = @pending.size
  def executed_count = @executed.size
end
```

---

### Real-World Example — Remote Control (Home Automation)

```ruby
# --- Receivers ---
class Light
  attr_reader :location, :brightness

  def initialize(location)
    @location = location
    @brightness = 0
  end

  def on
    @brightness = 100
    puts "#{@location} light ON (100%)"
  end

  def off
    @brightness = 0
    puts "#{@location} light OFF"
  end

  def dim(level)
    @brightness = level
    puts "#{@location} light dimmed to #{level}%"
  end
end

class Thermostat
  attr_reader :temperature

  def initialize
    @temperature = 20.0
  end

  def set_temperature(temp)
    puts "Thermostat set to #{temp}°C"
    @temperature = temp
  end
end

# --- Concrete Commands ---
class LightOnCommand
  def initialize(light)
    @light = light
    @previous_brightness = 0
  end

  def execute
    @previous_brightness = @light.brightness
    @light.on
  end

  def undo
    @previous_brightness.zero? ? @light.off : @light.dim(@previous_brightness)
  end

  def description = "#{@light.location} Light ON"
end

class LightOffCommand
  def initialize(light)
    @light = light
    @previous_brightness = 0
  end

  def execute
    @previous_brightness = @light.brightness
    @light.off
  end

  def undo
    @previous_brightness.positive? ? @light.dim(@previous_brightness) : @light.on
  end

  def description = "#{@light.location} Light OFF"
end

class ThermostatCommand
  def initialize(thermostat, new_temp)
    @thermostat = thermostat
    @new_temp = new_temp
    @previous_temp = 0
  end

  def execute
    @previous_temp = @thermostat.temperature
    @thermostat.set_temperature(@new_temp)
  end

  def undo
    @thermostat.set_temperature(@previous_temp)
  end

  def description = "Set thermostat to #{@new_temp.to_i}°C"
end

# --- Invoker: Remote Control ---
class RemoteControl
  NUM_SLOTS = 5

  def initialize
    @on_commands  = Array.new(NUM_SLOTS)
    @off_commands = Array.new(NUM_SLOTS)
  end

  def set_command(slot, on_cmd, off_cmd)
    return unless slot.between?(0, NUM_SLOTS - 1)

    @on_commands[slot]  = on_cmd
    @off_commands[slot] = off_cmd
  end

  def press_on(slot)
    return unless slot.between?(0, NUM_SLOTS - 1) && @on_commands[slot]

    print "[Slot #{slot} ON] "
    @on_commands[slot].execute
  end

  def press_off(slot)
    return unless slot.between?(0, NUM_SLOTS - 1) && @off_commands[slot]

    print "[Slot #{slot} OFF] "
    @off_commands[slot].execute
  end

  def print_slots
    puts "\n--- Remote Control ---"
    NUM_SLOTS.times do |i|
      on_desc  = @on_commands[i]&.description || "---"
      off_desc = @off_commands[i]&.description || "---"
      puts "Slot #{i}: #{on_desc} / #{off_desc}"
    end
  end
end

# --- Client code ---
living_room = Light.new("Living Room")
bedroom = Light.new("Bedroom")
thermostat = Thermostat.new

remote = RemoteControl.new
remote.set_command(0, LightOnCommand.new(living_room), LightOffCommand.new(living_room))
remote.set_command(1, LightOnCommand.new(bedroom), LightOffCommand.new(bedroom))
remote.set_command(2, ThermostatCommand.new(thermostat, 24.0), ThermostatCommand.new(thermostat, 20.0))

remote.print_slots

puts
remote.press_on(0)   # Living Room light ON
remote.press_on(1)   # Bedroom light ON
remote.press_on(2)   # Thermostat to 24°C
remote.press_off(0)  # Living Room light OFF
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
```ruby
# BAD: Massive conditional logic based on state
class VendingMachine
  IDLE = :idle
  HAS_MONEY = :has_money
  DISPENSING = :dispensing
  OUT_OF_STOCK = :out_of_stock

  def initialize
    @state = IDLE
    @balance = 0
  end

  def insert_money(amount)
    case @state
    when IDLE
      @balance += amount
      @state = HAS_MONEY
    when HAS_MONEY
      @balance += amount
    when DISPENSING
      puts "Please wait, dispensing..."
    when OUT_OF_STOCK
      puts "Machine is out of stock, returning money"
    end
    # Every method has this same case statement for EVERY state!
  end

  def select_product(id)
    case @state
    when IDLE then # ...
    when HAS_MONEY then # ...
    when DISPENSING then # ...
    when OUT_OF_STOCK then # ...
    end
    # Duplicated state checks everywhere!
  end

  # Adding a new state means modifying EVERY method!
end
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
│      Context          │       │     State (duck type)     │
├──────────────────────┤       ├──────────────────────────┤
│ - state               │──────→│ + handle_insert_money     │
├──────────────────────┤       │ + handle_select_product   │
│ + state=              │       │ + handle_dispense         │
│ + insert_money        │       └──────────────────────────┘
│ + select_product      │                    ▲
│ + dispense            │                    │
└──────────────────────┘       ┌─────────────┼─────────────┐
                               │             │             │
                          IdleState    HasMoneyState  DispensingState
```

**Participants:**
- **State (duck type):** Defines interface for state-specific behavior
- **ConcreteState:** Implements behavior for a particular state; may trigger state transitions
- **Context:** Maintains a reference to the current state; delegates requests to it

---

### Vending Machine Example

```ruby
# --- Context ---
class VendingMachine
  attr_reader :balance, :selected_product

  Product = Struct.new(:name, :price, :quantity)

  def initialize
    @products = {}
    @balance = 0.0
    @selected_product = nil
    @state = IdleState.new
  end

  def state=(new_state)
    puts "  [State: #{@state.name} → #{new_state.name}]"
    @state = new_state
  end

  # Delegate to current state
  def insert_money(amount)
    puts "\n> Insert $#{amount}"
    @state.insert_money(self, amount)
  end

  def select_product(id)
    puts "\n> Select product ##{id}"
    @state.select_product(self, id)
  end

  def dispense
    @state.dispense(self)
  end

  def cancel_transaction
    puts "\n> Cancel transaction"
    @state.cancel_transaction(self)
  end

  # Internal methods used by states
  def add_balance(amount)    = @balance += amount
  def reset_balance          = @balance = 0.0
  def selected_product=(id)  = @selected_product = id

  def add_product(id, name, price, qty)
    @products[id] = Product.new(name, price, qty)
  end

  def has_product?(id)
    @products.key?(id) && @products[id].quantity.positive?
  end

  def product(id) = @products.fetch(id)

  def decrement_product(id)
    @products[id].quantity -= 1 if @products.key?(id)
  end

  def any_products?
    @products.values.any? { |p| p.quantity.positive? }
  end

  def print_status
    puts "  Balance: $#{@balance} | State: #{@state.name}"
  end
end

# --- Concrete States ---
class IdleState
  def name = "Idle"

  def insert_money(machine, amount)
    machine.add_balance(amount)
    puts "  Money accepted. Balance: $#{machine.balance}"
    machine.state = HasMoneyState.new
  end

  def select_product(machine, _id)
    puts "  Please insert money first."
  end

  def dispense(machine)
    puts "  Please insert money and select a product."
  end

  def cancel_transaction(machine)
    puts "  No transaction to cancel."
  end
end

class HasMoneyState
  def name = "HasMoney"

  def insert_money(machine, amount)
    machine.add_balance(amount)
    puts "  Additional money accepted. Balance: $#{machine.balance}"
  end

  def select_product(machine, product_id)
    unless machine.has_product?(product_id)
      puts "  Product ##{product_id} is out of stock."
      return
    end

    product = machine.product(product_id)
    if machine.balance < product.price
      puts "  Insufficient funds. #{product.name} costs $#{product.price}, " \
           "balance: $#{machine.balance}"
      return
    end

    machine.selected_product = product_id
    puts "  Selected: #{product.name} ($#{product.price})"
    machine.state = DispensingState.new
    machine.dispense  # auto-dispense after selection
  end

  def dispense(machine)
    puts "  Please select a product first."
  end

  def cancel_transaction(machine)
    puts "  Transaction cancelled. Returning $#{machine.balance}"
    machine.reset_balance
    machine.state = IdleState.new
  end
end

class DispensingState
  def name = "Dispensing"

  def insert_money(machine, _amount)
    puts "  Please wait, dispensing product..."
  end

  def select_product(machine, _id)
    puts "  Please wait, dispensing product..."
  end

  def dispense(machine)
    product = machine.product(machine.selected_product)
    machine.decrement_product(machine.selected_product)
    change = machine.balance - product.price
    machine.reset_balance

    puts "  *** Dispensing: #{product.name} ***"
    puts "  Returning change: $#{change}" if change.positive?

    machine.state = machine.any_products? ? IdleState.new : OutOfStockState.new
  end

  def cancel_transaction(machine)
    puts "  Cannot cancel, already dispensing."
  end
end

class OutOfStockState
  def name = "OutOfStock"

  def insert_money(machine, amount)
    puts "  Machine is out of stock. Returning $#{amount}"
  end

  def select_product(machine, _id)
    puts "  Machine is out of stock."
  end

  def dispense(machine)
    puts "  Machine is out of stock."
  end

  def cancel_transaction(machine)
    puts "  No transaction in progress."
  end
end

# --- Client code ---
machine = VendingMachine.new
machine.add_product(1, "Cola", 1.50, 2)
machine.add_product(2, "Chips", 2.00, 1)
machine.add_product(3, "Water", 1.00, 1)

machine.insert_money(2.00)
machine.select_product(1)   # Cola $1.50 — get $0.50 change

machine.insert_money(1.00)
machine.select_product(2)   # Chips $2.00 — not enough!
machine.insert_money(1.00)
machine.select_product(2)   # Now enough

machine.insert_money(5.00)
machine.cancel_transaction  # Get $5.00 back

machine.insert_money(1.50)
machine.select_product(1)   # Last Cola

machine.insert_money(1.00)
machine.select_product(3)   # Last Water — machine now out of stock

machine.insert_money(1.00)  # Should be rejected — out of stock
```

---

### State Transition Table

A state transition table documents all valid transitions:

```
┌─────────────────┬──────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ Current State    │ insert_money     │ select_product   │ dispense         │ cancel           │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Idle            │ → HasMoney       │ "Insert money"   │ "Insert money"   │ "Nothing to do"  │
│ HasMoney        │ Add to balance   │ → Dispensing     │ "Select product" │ → Idle (refund)  │
│ Dispensing      │ "Wait..."        │ "Wait..."        │ → Idle/OutOfStock│ "Can't cancel"   │
│ OutOfStock      │ Return money     │ "Out of stock"   │ "Out of stock"   │ "Nothing to do"  │
└─────────────────┴──────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

### TCP Connection Example

```ruby
class TCPConnection
  def initialize
    @state = ClosedTCPState.new
  end

  def state=(new_state)
    puts "  [TCP: #{@state.name} → #{new_state.name}]"
    @state = new_state
  end

  def open        = @state.open(self)
  def close       = @state.close(self)
  def acknowledge = @state.acknowledge(self)
  def send_data(data) = @state.send_data(self, data)
  def state_name  = @state.name
end

class ClosedTCPState
  def name = "CLOSED"

  def open(conn)
    puts "  Opening connection... sending SYN"
    conn.state = ListeningTCPState.new
  end

  def close(conn)       = puts("  Already closed.")
  def acknowledge(conn) = puts("  No connection to acknowledge.")
  def send_data(conn, _data) = puts("  Cannot send — connection is closed.")
end

class ListeningTCPState
  def name = "LISTENING"

  def open(conn)  = puts("  Already opening...")

  def close(conn)
    puts "  Aborting connection."
    conn.state = ClosedTCPState.new
  end

  def acknowledge(conn)
    puts "  SYN-ACK received. Connection established!"
    conn.state = EstablishedTCPState.new
  end

  def send_data(conn, _data) = puts("  Cannot send — still establishing connection.")
end

class EstablishedTCPState
  def name = "ESTABLISHED"

  def open(conn) = puts("  Connection already established.")

  def close(conn)
    puts "  Sending FIN... closing connection."
    conn.state = ClosedTCPState.new
  end

  def acknowledge(conn) = puts("  ACK received.")

  def send_data(conn, data)
    puts "  Sending data: \"#{data}\" (#{data.bytesize} bytes)"
  end
end

# --- Client code ---
conn = TCPConnection.new

conn.send_data("Hello")  # Can't send — closed
conn.open                 # CLOSED → LISTENING
conn.send_data("Hello")  # Can't send — still connecting
conn.acknowledge          # LISTENING → ESTABLISHED
conn.send_data("Hello")  # Sends data
conn.send_data("World")  # Sends data
conn.close                # ESTABLISHED → CLOSED
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
| Large conditional blocks based on state | Replaces if/else or case with polymorphism |
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
```ruby
# BAD: Duplicated algorithm structure across classes
class CSVDataMiner
  def mine(path)
    raw_data = read_file(path)        # same
    data = parse_csv(raw_data)        # different
    analysis = analyze_data(data)     # same
    generate_report(analysis)         # same
  end
end

class JSONDataMiner
  def mine(path)
    raw_data = read_file(path)        # same (duplicated!)
    data = parse_json(raw_data)       # different
    analysis = analyze_data(data)     # same (duplicated!)
    generate_report(analysis)         # same (duplicated!)
  end
end

class XMLDataMiner
  def mine(path)
    raw_data = read_file(path)        # same (duplicated!)
    data = parse_xml(raw_data)        # different
    analysis = analyze_data(data)     # same (duplicated!)
    generate_report(analysis)         # same (duplicated!)
  end
end
# The algorithm structure (read → parse → analyze → report) is duplicated!
# Common steps are copy-pasted. Bug fix in one? Must fix all three!
```

---

### Structure

```
┌──────────────────────────────────┐
│     AbstractClass                │
├──────────────────────────────────┤
│ + mine()            ← final      │  ← defines the algorithm skeleton
│ # read_data()       ← concrete   │  ← common implementation
│ # parse_data()      ← abstract   │  ← subclasses MUST override
│ # analyze_data()    ← concrete   │  ← common implementation
│ # hook()            ← virtual    │  ← subclasses CAN override (optional)
└──────────────────────────────────┘
                ▲
                │
     ┌──────────┴──────────┐
     │                     │
ConcreteClassA         ConcreteClassB
# parse_data() override  # parse_data() override
# hook() override         (uses default hook)
```

**Participants:**
- **AbstractClass:** Defines the template method (algorithm skeleton) and abstract/hook steps
- **ConcreteClass:** Implements the abstract steps; optionally overrides hooks
- **Template Method:** The method that defines the algorithm — should not be overridden

**Types of methods in the base class:**
1. **Concrete methods:** Common steps with default implementation (not overridden)
2. **Abstract methods:** Steps that MUST be implemented by subclasses (raise `NotImplementedError`)
3. **Hook methods:** Steps with default (often empty) implementation that CAN be overridden

---

### Basic Implementation — Data Mining Framework

```ruby
# --- Abstract Class with Template Method ---
class DataMiner
  # THE TEMPLATE METHOD — defines the algorithm skeleton
  def mine(path)
    raw_data = read_data(path)          # step 1: read (concrete)
    records = parse_data(raw_data)      # step 2: parse (abstract)

    if should_filter?                   # hook: optional filtering
      records = filter_data(records)    # step 3: filter (hook)
    end

    analysis = analyze_data(records)    # step 4: analyze (concrete)
    generate_report(analysis)           # step 5: report (concrete)

    on_complete                         # hook: post-processing
  end

  private

  # --- Concrete method (common implementation) ---
  def read_data(path)
    puts "Reading data from: #{path}"
    "raw data from #{path}"
  end

  # --- Abstract method (subclasses MUST implement) ---
  def parse_data(_raw_data)
    raise NotImplementedError, "#{self.class} must implement #parse_data"
  end

  # --- Concrete method (common implementation) ---
  def analyze_data(records)
    puts "Analyzing #{records.size} records..."
    records.map { |r| "Analyzed: #{r}" }
  end

  # --- Concrete method (common implementation) ---
  def generate_report(analysis)
    puts "=== Report ==="
    analysis.each { |item| puts "  #{item}" }
    puts "=== End Report ==="
  end

  # --- Hook methods (optional override) ---
  def should_filter? = false

  def filter_data(records) = records

  def on_complete
    # default: do nothing — subclasses can override for cleanup, logging, etc.
  end
end

# --- Concrete Class: CSV Miner ---
class CSVDataMiner < DataMiner
  private

  def parse_data(_raw_data)
    puts "Parsing CSV data..."
    ["row1:col1,col2", "row2:col1,col2", "row3:col1,col2"]
  end

  def should_filter? = true

  def filter_data(records)
    puts "Filtering CSV records (removing empty rows)..."
    records.reject(&:empty?)
  end

  def on_complete
    puts "CSV mining complete. Closing file handles."
  end
end

# --- Concrete Class: JSON Miner ---
class JSONDataMiner < DataMiner
  private

  def parse_data(_raw_data)
    puts "Parsing JSON data..."
    ['{"name":"Alice"}', '{"name":"Bob"}']
  end

  def on_complete
    puts "JSON mining complete."
  end
end

# --- Concrete Class: XML Miner ---
class XMLDataMiner < DataMiner
  private

  def parse_data(_raw_data)
    puts "Parsing XML data..."
    ["<record>Alice</record>", "<record>Bob</record>"]
  end
  # Uses default hooks (no filtering, no on_complete action)
end

# --- Client code ---
puts "=== CSV Mining ==="
CSVDataMiner.new.mine("data.csv")

puts "\n=== JSON Mining ==="
JSONDataMiner.new.mine("data.json")

puts "\n=== XML Mining ==="
XMLDataMiner.new.mine("data.xml")
```

---

### Hook Methods

Hooks are methods with a default (often empty) implementation that subclasses can optionally override. They provide extension points without forcing subclasses to implement them.

```ruby
class GameAI
  # Template method
  def take_turn
    collect_resources
    build_structures

    if should_attack?       # HOOK — subclasses decide
      attack
    else
      defend
    end

    on_turn_end             # HOOK — optional post-turn logic
  end

  private

  # Concrete step
  def collect_resources
    puts "Collecting resources from owned territories"
  end

  # Abstract steps — MUST override
  def build_structures = raise(NotImplementedError)
  def attack           = raise(NotImplementedError)
  def defend           = raise(NotImplementedError)

  # Hook methods — CAN override
  def should_attack? = true
  def on_turn_end = nil
end

class AggressiveAI < GameAI
  private

  def build_structures = puts("Building barracks and weapons factory")
  def attack           = puts("Launching full-scale attack with all units!")
  def defend           = puts("Minimal defense — attack is the best defense")
  def should_attack?   = true
end

class DefensiveAI < GameAI
  def initialize
    super
    @turns_without_attack = 0
  end

  private

  def build_structures = puts("Building walls and defensive towers")
  def attack           = puts("Sending small raiding party")
  def defend           = puts("Fortifying all positions")

  def should_attack?
    @turns_without_attack += 1
    @turns_without_attack > 5  # only attack after 5 defensive turns
  end

  def on_turn_end = puts("Checking perimeter defenses...")
end
```

---

### Template Method with Modules (Ruby Idiom)

Ruby modules can serve as an alternative to abstract base classes for the Template Method pattern:

```ruby
module Reportable
  def generate
    header
    body
    footer
    on_complete
  end

  private

  def header = puts("=== #{report_title} ===")
  def footer = puts("=== End ===")

  # Abstract — must be implemented by including class
  def report_title = raise(NotImplementedError)
  def body         = raise(NotImplementedError)

  # Hook
  def on_complete = nil
end

class SalesReport
  include Reportable

  private

  def report_title = "Sales Report"

  def body
    puts "  Total sales: $10,000"
    puts "  Top product: Widget"
  end

  def on_complete = puts("Sent to management.")
end

class InventoryReport
  include Reportable

  private

  def report_title = "Inventory Report"

  def body
    puts "  Items in stock: 500"
    puts "  Low stock alerts: 3"
  end
end

SalesReport.new.generate
puts
InventoryReport.new.generate
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
│  mine() {                           │
│      read_data          ← concrete  │
│      parse_data         ← calls subclass (abstract)
│      if should_filter?  ← calls subclass (hook)
│          filter_data    ← calls subclass (abstract)
│      end                            │
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
│  parse_data { # my implementation } │
│  filter_data { # my implementation }│
│  should_filter? { true }            │
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
| Flexibility | Fixed at class definition time | Changeable at runtime |
| Control | Base class controls flow | Client controls which strategy |
| Coupling | Tight (inheritance) | Loose (duck typing) |
| Class count | Fewer (just subclasses) | More (strategy objects + context) |
| Ruby idiom | Subclasses or modules | Lambdas, Procs, duck-typed objects |

---


## 6.6 Iterator Pattern

> The Iterator pattern provides a way to access the elements of an aggregate object sequentially without exposing its underlying representation. It separates the traversal logic from the collection, allowing multiple traversal strategies over the same collection. Ruby's `Enumerable` module and `each` convention make this pattern deeply embedded in the language.

---

### Why Iterator?

When you need to traverse a collection without knowing its internal structure:
- Iterate over a tree (in-order, pre-order, post-order) without knowing it's a tree
- Traverse a graph (BFS, DFS) through a uniform interface
- Access elements of a custom data structure like a standard container
- Support multiple simultaneous traversals over the same collection

**The problem without Iterator:**
```ruby
# BAD: Client must know internal structure to traverse
class BinaryTree
  attr_accessor :root

  Node = Struct.new(:value, :left, :right)
end

# Client code — tightly coupled to tree internals
def print_all(node)
  return unless node

  print_all(node.left)
  print "#{node.value} "
  print_all(node.right)
end
# What if we change from BinaryTree to a different structure? Client breaks!
```

---

### Structure

```
┌──────────────────────┐       ┌──────────────────────────┐
│   Collection          │       │   Iterator (duck type)    │
├──────────────────────┤       ├──────────────────────────┤
│ + each { |item| }    │──────→│ + next: Element           │
│ + include Enumerable │       │ + has_next?: bool         │
└──────────────────────┘       └──────────────────────────┘
                                            ▲
                                            │
                                ┌───────────┼───────────┐
                                │                       │
                         ExternalIterator        Enumerator
```

---

### Internal vs External Iterators

**External Iterator:** The client controls the iteration (calls `next` explicitly).
**Internal Iterator:** The collection controls the iteration (accepts a block to apply to each element).

```ruby
class NumberCollection
  include Enumerable

  def initialize
    @numbers = []
  end

  def add(n)
    @numbers << n
    self
  end

  # --- Internal Iterator (collection controls) ---
  # This is THE Ruby way — define `each` and include Enumerable
  def each(&block)
    @numbers.each(&block)
  end

  # Internal iterator with early termination
  def each_until
    @numbers.each do |n|
      break unless yield(n)  # stop if block returns false
    end
  end

  # --- External Iterator (client controls) ---
  def create_iterator
    ExternalIterator.new(@numbers)
  end

  class ExternalIterator
    def initialize(data)
      @data = data
      @index = 0
    end

    def has_next? = @index < @data.size
    def next      = @data[@index].tap { @index += 1 }
    def current   = @data[@index]
    def reset     = @index = 0
  end
end

collection = NumberCollection.new
collection.add(10).add(20).add(30).add(40)

# Internal iterator — collection controls (idiomatic Ruby)
puts "Internal iterator (each):"
collection.each { |n| print "#{n} " }
puts

# Enumerable methods — all available because we defined `each`
puts "Sum: #{collection.sum}"
puts "Select > 20: #{collection.select { |n| n > 20 }}"
puts "Map doubled: #{collection.map { |n| n * 2 }}"
puts "Min/Max: #{collection.minmax}"

# External iterator — client controls
puts "\nExternal iterator:"
it = collection.create_iterator
while it.has_next?
  print "#{it.next} "
end
puts

# Ruby's built-in Enumerator (external iterator for free)
puts "\nRuby Enumerator:"
enum = collection.each  # returns an Enumerator when no block given
puts enum.next  # 10
puts enum.next  # 20
puts enum.next  # 30
```

---

### The Power of Enumerable

By defining `each` and including `Enumerable`, your collection gets 50+ methods for free:

```ruby
class WordCollection
  include Enumerable

  def initialize(*words)
    @words = words
  end

  def each(&block)
    @words.each(&block)
  end
end

words = WordCollection.new("apple", "banana", "cherry", "date", "elderberry")

# All of these work automatically:
puts words.sort                          # alphabetical sort
puts words.select { |w| w.length > 5 }  # filter
puts words.map(&:upcase)                 # transform
puts words.count                         # count
puts words.min_by(&:length)              # shortest word
puts words.group_by { |w| w[0] }        # group by first letter
puts words.flat_map(&:chars).uniq        # unique characters
puts words.any? { |w| w.start_with?("b") }  # any banana?
puts words.reduce { |a, b| a.length > b.length ? a : b }  # longest
puts words.each_with_object([]) { |w, arr| arr << w.reverse }  # reverse all
puts words.zip([1, 2, 3, 4, 5])         # pair with indices
puts words.take(3)                       # first 3
puts words.drop_while { |w| w < "c" }   # drop until "c"
```

---

### Custom Linked List with Enumerable

```ruby
class LinkedList
  include Enumerable

  Node = Struct.new(:data, :next_node)

  def initialize
    @head = nil
    @tail = nil
    @count = 0
  end

  def push_back(value)
    new_node = Node.new(value, nil)
    if @tail
      @tail.next_node = new_node
      @tail = new_node
    else
      @head = @tail = new_node
    end
    @count += 1
    self
  end

  def push_front(value)
    @head = Node.new(value, @head)
    @tail = @head unless @tail
    @count += 1
    self
  end

  def size  = @count
  def empty? = @count.zero?

  # THE key method — enables all Enumerable methods
  def each
    return enum_for(:each) unless block_given?

    current = @head
    while current
      yield current.data
      current = current.next_node
    end
  end

  # Reverse iteration
  def reverse_each
    return enum_for(:reverse_each) unless block_given?

    to_a.reverse_each { |item| yield item }
  end

  def to_s = "[#{to_a.join(' -> ')}]"
end

# --- Usage ---
list = LinkedList.new
list.push_back(10).push_back(20).push_back(30).push_front(5)

puts list  # [5 -> 10 -> 20 -> 30]

# Range-based iteration — works because we defined each!
list.each { |val| print "#{val} " }
puts  # 5 10 20 30

# All Enumerable methods work
puts "Sum: #{list.sum}"                    # 65
puts "Include 20? #{list.include?(20)}"    # true
puts "Sorted: #{list.sort}"               # [5, 10, 20, 30]
puts "Select > 10: #{list.select { |v| v > 10 }}"  # [20, 30]

# External iterator via Enumerator
enum = list.each
puts enum.next  # 5
puts enum.next  # 10

# Works with Ruby's built-in methods
list.each_with_index { |val, i| puts "  [#{i}] #{val}" }
```

---

### Tree Iterator (Multiple Traversal Strategies)

```ruby
class BinaryTree
  Node = Struct.new(:value, :left, :right)

  attr_reader :root

  def initialize(root)
    @root = root
  end

  # --- In-Order Iterator (Left → Root → Right) ---
  def in_order
    return enum_for(:in_order) unless block_given?

    traverse_in_order(@root) { |val| yield val }
  end

  # --- Pre-Order Iterator (Root → Left → Right) ---
  def pre_order
    return enum_for(:pre_order) unless block_given?

    traverse_pre_order(@root) { |val| yield val }
  end

  # --- Level-Order Iterator (BFS) ---
  def level_order
    return enum_for(:level_order) unless block_given?

    queue = [@root].compact
    until queue.empty?
      node = queue.shift
      yield node.value
      queue << node.left if node.left
      queue << node.right if node.right
    end
  end

  # Default iteration is in-order
  include Enumerable
  def each(&block) = in_order(&block)

  private

  def traverse_in_order(node, &block)
    return unless node

    traverse_in_order(node.left, &block)
    yield node.value
    traverse_in_order(node.right, &block)
  end

  def traverse_pre_order(node, &block)
    return unless node

    yield node.value
    traverse_pre_order(node.left, &block)
    traverse_pre_order(node.right, &block)
  end
end

# --- Client code ---
#       4
#      / \
#     2   6
#    / \ / \
#   1  3 5  7
root = BinaryTree::Node.new(4,
  BinaryTree::Node.new(2,
    BinaryTree::Node.new(1),
    BinaryTree::Node.new(3)
  ),
  BinaryTree::Node.new(6,
    BinaryTree::Node.new(5),
    BinaryTree::Node.new(7)
  )
)

tree = BinaryTree.new(root)

print "InOrder:    "; tree.in_order    { |v| print "#{v} " }; puts  # 1 2 3 4 5 6 7
print "PreOrder:   "; tree.pre_order   { |v| print "#{v} " }; puts  # 4 2 1 3 6 5 7
print "LevelOrder: "; tree.level_order { |v| print "#{v} " }; puts  # 4 2 6 1 3 5 7

# Enumerator-based (external iteration)
enum = tree.in_order
puts enum.next  # 1
puts enum.next  # 2

# Enumerable methods work on default (in-order) traversal
puts "Sum: #{tree.sum}"                    # 28
puts "Sorted: #{tree.sort}"               # [1, 2, 3, 4, 5, 6, 7]
puts "Select > 4: #{tree.select { |v| v > 4 }}"  # [5, 6, 7]
```

---

### Lazy Iterators

Ruby's `Enumerator::Lazy` enables lazy evaluation for potentially infinite or expensive sequences:

```ruby
# Infinite Fibonacci sequence — lazy, computed on demand
def fibonacci
  Enumerator.new do |yielder|
    a, b = 0, 1
    loop do
      yielder.yield a
      a, b = b, a + b
    end
  end.lazy
end

# Take only what you need — no infinite loop!
puts fibonacci.first(10).to_a.inspect
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Chain lazy operations
puts fibonacci.select(&:odd?).first(5).to_a.inspect
# [1, 1, 3, 5, 13]

puts fibonacci.select { |n| n > 100 }.first(3).to_a.inspect
# [144, 233, 377]

# Lazy file processing — reads only what's needed
# File.open("huge.log").each_line.lazy
#   .select { |line| line.include?("ERROR") }
#   .map(&:strip)
#   .first(10)
```

---

### When to Use Iterator

| Scenario | Why Iterator Helps |
|----------|-------------------|
| Hide collection internals from clients | Clients use uniform interface regardless of structure |
| Support multiple traversal strategies | Different iterators for same collection |
| Provide uniform traversal interface | Same `each`/`next` for lists, trees, graphs |
| Enable simultaneous traversals | Each Enumerator maintains its own position |
| Make custom collections work with Enumerable | Define `each`, include `Enumerable`, get 50+ methods |
| Decouple algorithms from data structures | Algorithms work with iterators, not specific containers |

**Ruby Iterator Patterns:**

| Pattern | Description | Example |
|---------|------------|---------|
| Internal (block) | Collection controls iteration | `collection.each { \|x\| ... }` |
| External (Enumerator) | Client controls iteration | `enum = collection.each; enum.next` |
| Lazy | Deferred evaluation | `collection.lazy.select { ... }.first(5)` |
| Chained | Pipeline of transformations | `collection.map { ... }.select { ... }.sort` |
| Custom | Define `each` + include `Enumerable` | Full Enumerable API for free |

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
```ruby
# BAD: Every component knows about every other component
class TextBox
  attr_accessor :checkbox, :submit_btn, :error_label, :dropdown

  def on_change
    # Must coordinate with every other component directly
    if text.empty?
      submit_btn.disable
      error_label.show("Required field")
    else
      submit_btn.enable
      error_label.hide
      dropdown.refresh if checkbox.checked?
    end
    # Tight coupling! Adding a new component means modifying ALL existing ones
  end
end
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
│   Mediator            │       │   Colleague (duck type)  │
├──────────────────────┤       ├──────────────────────────┤
│ + notify(sender,     │◄─────│ - mediator                │
│         event)       │       │ + mediator=               │
└──────────────────────┘       └──────────────────────────┘
         ▲                                  ▲
         │                                  │
  ConcreteMediator              ┌───────────┼───────────┐
  (knows all colleagues)        │           │           │
                           ColleagueA  ColleagueB  ColleagueC
```

---

### Chat Room Example

```ruby
# --- Colleague (User) ---
class User
  attr_reader :name
  attr_accessor :mediator

  def initialize(name)
    @name = name
    @mediator = nil
    @message_history = []
  end

  def send_message(message, to: nil)
    print "[#{@name} sends]: #{message}"
    puts to ? " (to #{to})" : ""
    @mediator.send_message(message, sender: self, recipient: to)
  end

  def receive(from, message)
    full_message = "#{from}: #{message}"
    @message_history << full_message
    puts "  [#{@name} receives]: #{full_message}"
  end

  def show_history
    puts "\n--- #{@name}'s message history ---"
    @message_history.each { |msg| puts "  #{msg}" }
  end
end

# --- Concrete Mediator: Chat Room ---
class ChatRoom
  def initialize(name)
    @room_name = name
    @users = {}
    puts "Chat room \"#{@room_name}\" created."
  end

  def add_user(user)
    user.mediator = self

    # Notify existing users
    @users.each_value do |existing_user|
      existing_user.receive("System", "#{user.name} has joined the chat")
    end

    @users[user.name] = user
    puts "[#{@room_name}] #{user.name} joined the room."
  end

  def remove_user(name)
    @users.delete(name)
    @users.each_value do |user|
      user.receive("System", "#{name} has left the chat")
    end
    puts "[#{@room_name}] #{name} left the room."
  end

  def send_message(message, sender:, recipient: nil)
    if recipient.nil?
      # Broadcast to all except sender
      @users.each do |name, user|
        user.receive(sender.name, message) unless name == sender.name
      end
    else
      # Direct message
      target = @users[recipient]
      if target
        target.receive("#{sender.name} (DM)", message)
      else
        sender.receive("System", "User #{recipient} not found")
      end
    end
  end
end

# --- Client code ---
chat_room = ChatRoom.new("General")

alice   = User.new("Alice")
bob     = User.new("Bob")
charlie = User.new("Charlie")

chat_room.add_user(alice)
chat_room.add_user(bob)
chat_room.add_user(charlie)

puts

alice.send_message("Hello everyone!")

puts

bob.send_message("Hey Alice, how are you?", to: "Alice")

puts

charlie.send_message("Good morning!")

alice.show_history
bob.show_history
```

---

### Air Traffic Controller Example

```ruby
class Aircraft
  attr_reader :call_sign, :altitude
  attr_accessor :atc

  def initialize(call_sign, altitude)
    @call_sign = call_sign
    @altitude = altitude
    @atc = nil
  end

  def altitude=(alt)
    @altitude = alt
  end

  def request_landing
    puts "#{@call_sign}: Requesting landing clearance. Altitude: #{@altitude}ft"
    @atc.request_landing(self)
  end

  def request_takeoff
    puts "#{@call_sign}: Requesting takeoff clearance."
    @atc.request_takeoff(self)
  end

  def receive_message(from, message)
    puts "  #{@call_sign} receives from #{from}: #{message}"
  end
end

class ControlTower
  def initialize
    @aircraft = []
    @runway_free = true
    @active_runway = "27L"
  end

  def register(ac)
    ac.atc = self
    @aircraft << ac
    puts "[Tower] #{ac.call_sign} registered with tower."
  end

  def request_landing(ac)
    if @runway_free
      @runway_free = false
      ac.receive_message("Tower", "Cleared to land runway #{@active_runway}")
      notify_all("#{ac.call_sign} is landing on runway #{@active_runway}. Hold positions.", except: ac)
    else
      ac.receive_message("Tower", "Runway occupied. Enter holding pattern at #{ac.altitude}ft")
    end
  end

  def request_takeoff(ac)
    if @runway_free
      @runway_free = false
      ac.receive_message("Tower", "Cleared for takeoff runway #{@active_runway}")
      notify_all("#{ac.call_sign} is departing runway #{@active_runway}. Hold short.", except: ac)
    else
      ac.receive_message("Tower", "Runway occupied. Hold short of runway #{@active_runway}")
    end
  end

  def runway_clear
    @runway_free = true
    puts "[Tower] Runway #{@active_runway} is now clear."
    notify_all("Runway #{@active_runway} is clear.")
  end

  private

  def notify_all(message, except: nil)
    @aircraft.each do |ac|
      ac.receive_message("Tower", message) unless ac == except
    end
  end
end

# --- Client code ---
tower = ControlTower.new

flight1 = Aircraft.new("UA-123", 10_000)
flight2 = Aircraft.new("AA-456", 12_000)
flight3 = Aircraft.new("DL-789", 0)

tower.register(flight1)
tower.register(flight2)
tower.register(flight3)

puts
flight1.request_landing    # Cleared — runway was free
puts
flight2.request_landing    # Denied — runway occupied
puts
tower.runway_clear         # Runway freed
puts
flight3.request_takeoff    # Cleared — runway is free now
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
```ruby
# BAD: Caller must know which handler to use
class SupportSystem
  def handle_ticket(ticket)
    case ticket.severity
    when "low"      then @level1_support.handle(ticket)
    when "medium"   then @level2_support.handle(ticket)
    when "high"     then @level3_support.handle(ticket)
    when "critical" then @manager.handle(ticket)
    end
    # Caller is coupled to ALL handlers and the routing logic
    # Adding a new level means modifying this code
  end
end
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
│   Handler (base class)    │
├──────────────────────────┤
│ - next_handler            │
├──────────────────────────┤
│ + set_next(handler)       │
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

```ruby
# --- Log Levels ---
module LogLevel
  DEBUG = 0
  INFO  = 1
  WARN  = 2
  ERROR = 3
  FATAL = 4

  NAMES = { DEBUG => "DEBUG", INFO => "INFO ", WARN => "WARN ",
            ERROR => "ERROR", FATAL => "FATAL" }.freeze

  def self.name_for(level) = NAMES.fetch(level, "?????")
end

# --- Handler Base Class ---
class LogHandler
  def initialize(level)
    @handler_level = level
    @next_handler = nil
  end

  def set_next(handler)
    @next_handler = handler
    handler  # return handler for chaining
  end

  def handle(level, message)
    write(level, message) if level >= @handler_level

    # Pass to next handler regardless (each handler decides independently)
    @next_handler&.handle(level, message)
  end

  private

  def timestamp = Time.now.strftime("%H:%M:%S")

  def write(level, message)
    raise NotImplementedError
  end
end

# --- Concrete Handlers ---
class ConsoleLogHandler < LogHandler
  private

  def write(level, message)
    puts "[#{timestamp}] [CONSOLE] [#{LogLevel.name_for(level)}] #{message}"
  end
end

class FileLogHandler < LogHandler
  def initialize(level, filename)
    super(level)
    @filename = filename
  end

  private

  def write(level, message)
    puts "[#{timestamp}] [FILE:#{@filename}] [#{LogLevel.name_for(level)}] #{message}"
  end
end

class EmailLogHandler < LogHandler
  def initialize(level, recipient)
    super(level)
    @recipient = recipient
  end

  private

  def write(level, message)
    puts "[#{timestamp}] [EMAIL→#{@recipient}] [#{LogLevel.name_for(level)}] #{message}"
  end
end

class SlackLogHandler < LogHandler
  def initialize(level, channel)
    super(level)
    @channel = channel
  end

  private

  def write(level, message)
    puts "[#{timestamp}] [SLACK:##{@channel}] [#{LogLevel.name_for(level)}] #{message}"
  end
end

# --- Client code ---
# Build the chain:
# Console (DEBUG+) → File (INFO+) → Slack (WARN+) → Email (ERROR+)
console = ConsoleLogHandler.new(LogLevel::DEBUG)
file    = FileLogHandler.new(LogLevel::INFO, "app.log")
slack   = SlackLogHandler.new(LogLevel::WARN, "alerts")
email   = EmailLogHandler.new(LogLevel::ERROR, "admin@example.com")

console.set_next(file).set_next(slack).set_next(email)

puts "=== DEBUG message ==="
console.handle(LogLevel::DEBUG, "Variable x = 42")
# Only Console handles it

puts "\n=== INFO message ==="
console.handle(LogLevel::INFO, "User logged in")
# Console + File handle it

puts "\n=== WARN message ==="
console.handle(LogLevel::WARN, "Disk usage at 85%")
# Console + File + Slack handle it

puts "\n=== ERROR message ==="
console.handle(LogLevel::ERROR, "Database connection failed!")
# All four handlers handle it
```

---

### Middleware Pipeline Example (Idiomatic Ruby with Lambdas)

A common real-world use: HTTP request processing pipeline where each middleware can process or reject the request.

```ruby
# --- Request/Response ---
HttpRequest = Struct.new(:method, :path, :headers, :body, :client_ip, :user_id, keyword_init: true)
HttpResponse = Struct.new(:status_code, :body, keyword_init: true)

# --- Middleware Base ---
class Middleware
  def initialize
    @next = nil
  end

  def set_next(middleware)
    @next = middleware
    middleware
  end

  def handle(request)
    @next ? @next.handle(request) : HttpResponse.new(status_code: 200, body: "OK")
  end
end

# --- Concrete Middleware ---
class AuthenticationMiddleware < Middleware
  def handle(request)
    puts "  [Auth] Checking authentication..."

    token = request.headers["Authorization"]
    unless token
      puts "  [Auth] REJECTED — No auth token"
      return HttpResponse.new(status_code: 401, body: "Unauthorized: Missing authentication token")
    end

    unless token.start_with?("Bearer ")
      puts "  [Auth] REJECTED — Invalid token format"
      return HttpResponse.new(status_code: 401, body: "Unauthorized: Invalid token format")
    end

    request.user_id = "user_#{token[7, 4]}"
    puts "  [Auth] Authenticated as #{request.user_id}"

    super  # pass to next
  end
end

class RateLimitMiddleware < Middleware
  def initialize(max_requests)
    super()
    @max_requests = max_requests
    @request_counts = Hash.new(0)
  end

  def handle(request)
    puts "  [RateLimit] Checking rate limit for #{request.client_ip}..."

    @request_counts[request.client_ip] += 1
    if @request_counts[request.client_ip] > @max_requests
      puts "  [RateLimit] REJECTED — Rate limit exceeded"
      return HttpResponse.new(status_code: 429, body: "Too Many Requests")
    end

    puts "  [RateLimit] OK (#{@request_counts[request.client_ip]}/#{@max_requests})"
    super
  end
end

class ValidationMiddleware < Middleware
  def handle(request)
    puts "  [Validation] Validating request..."

    if %w[POST PUT].include?(request.method)
      if request.body.nil? || request.body.empty?
        puts "  [Validation] REJECTED — Empty body for #{request.method}"
        return HttpResponse.new(status_code: 400, body: "Bad Request: Body required for #{request.method}")
      end
      unless request.headers.key?("Content-Type")
        puts "  [Validation] REJECTED — Missing Content-Type"
        return HttpResponse.new(status_code: 400, body: "Bad Request: Content-Type header required")
      end
    end

    puts "  [Validation] OK"
    super
  end
end

class LoggingMiddleware < Middleware
  def handle(request)
    puts "  [Log] #{request.method} #{request.path} from #{request.client_ip}"
    response = super
    puts "  [Log] Response: #{response.status_code}"
    response
  end
end

class RequestHandler < Middleware
  def handle(request)
    puts "  [Handler] Processing #{request.method} #{request.path}"
    HttpResponse.new(status_code: 200, body: "Success! Processed by #{request.user_id}")
  end
end

# --- Build middleware chain ---
logging    = LoggingMiddleware.new
rate_limit = RateLimitMiddleware.new(3)
auth       = AuthenticationMiddleware.new
validation = ValidationMiddleware.new
handler    = RequestHandler.new

logging.set_next(rate_limit).set_next(auth).set_next(validation).set_next(handler)

# Test 1: Valid GET request
puts "=== Request 1: Valid GET ==="
req1 = HttpRequest.new(method: "GET", path: "/api/users",
  headers: { "Authorization" => "Bearer abc123" }, body: "", client_ip: "192.168.1.1")
resp1 = logging.handle(req1)
puts "Result: #{resp1.status_code} — #{resp1.body}\n\n"

# Test 2: Missing auth
puts "=== Request 2: No Auth ==="
req2 = HttpRequest.new(method: "GET", path: "/api/users",
  headers: {}, body: "", client_ip: "192.168.1.2")
resp2 = logging.handle(req2)
puts "Result: #{resp2.status_code} — #{resp2.body}\n\n"

# Test 3: POST without body
puts "=== Request 3: POST without body ==="
req3 = HttpRequest.new(method: "POST", path: "/api/users",
  headers: { "Authorization" => "Bearer xyz789" }, body: "", client_ip: "192.168.1.1")
resp3 = logging.handle(req3)
puts "Result: #{resp3.status_code} — #{resp3.body}\n\n"
```

---

### Chain with Rack-Style Middleware (Ruby Idiom)

Ruby's Rack web framework uses a middleware pattern that's essentially Chain of Responsibility:

```ruby
class MiddlewareStack
  def initialize(app)
    @app = app
    @middlewares = []
  end

  def use(middleware_class, *args)
    @middlewares << [middleware_class, args]
    self
  end

  def call(request)
    # Build chain from inside out
    handler = @app
    @middlewares.reverse_each do |klass, args|
      handler = klass.new(handler, *args)
    end
    handler.call(request)
  end
end

# Each middleware wraps the next handler
class TimingMiddleware
  def initialize(app)
    @app = app
  end

  def call(request)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    response = @app.call(request)
    elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
    puts "  [Timing] #{(elapsed * 1000).round(2)}ms"
    response
  end
end

class CacheMiddleware
  def initialize(app, ttl: 60)
    @app = app
    @cache = {}
    @ttl = ttl
  end

  def call(request)
    key = "#{request[:method]}:#{request[:path]}"
    if @cache.key?(key)
      puts "  [Cache] HIT for #{key}"
      return @cache[key]
    end

    puts "  [Cache] MISS for #{key}"
    response = @app.call(request)
    @cache[key] = response
    response
  end
end

# Final handler
final_app = ->(request) { { status: 200, body: "Hello from #{request[:path]}" } }

stack = MiddlewareStack.new(final_app)
stack.use(TimingMiddleware)
stack.use(CacheMiddleware, ttl: 30)

puts stack.call(method: "GET", path: "/api/data")
puts stack.call(method: "GET", path: "/api/data")  # cache hit
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

> The Visitor pattern lets you add new operations to existing object structures without modifying the classes of the elements on which it operates. It achieves this through a technique called **double dispatch** — the operation executed depends on both the type of the visitor and the type of the element being visited. In Ruby, duck typing and `method_missing` offer alternative approaches to the classic Visitor.

---

### Why Visitor?

When you need to perform many unrelated operations on objects in a structure, and you don't want to pollute their classes with these operations:
- Compiler: AST nodes need type checking, code generation, optimization, pretty printing
- File system: files/directories need size calculation, search, permission check, backup
- Document: elements need rendering, exporting, spell checking, word counting
- Shape hierarchy: shapes need area calculation, drawing, serialization, collision detection

**The problem without Visitor:**
```ruby
# BAD: Every new operation requires modifying ALL element classes
class Circle
  def draw = # ...
  def area = # ...
  def to_json = # ... added later
  def to_xml  = # ... added later
  def optimize = # ... added later
  # Every new operation means modifying Circle, Rectangle, Triangle, etc.!
end
```

**Problems:**
1. Adding a new operation requires modifying every element class
2. Unrelated operations clutter the element classes (violates SRP)
3. Element classes become bloated with operations they shouldn't know about

---

### Structure

```
┌──────────────────────────┐       ┌──────────────────────────┐
│   Element (duck type)     │       │   Visitor (duck type)     │
├──────────────────────────┤       ├──────────────────────────┤
│ + accept(visitor)         │       │ + visit_circle(circle)    │
└──────────────────────────┘       │ + visit_rectangle(rect)   │
          ▲                        │ + visit_triangle(tri)     │
          │                        └──────────────────────────┘
   ┌──────┼──────┐                          ▲
   │      │      │                          │
Circle  Rect  Triangle              ┌───────┼───────┐
                                    │               │
                              AreaVisitor    DrawVisitor

Double Dispatch:
  element.accept(visitor)  →  visitor.visit_circle(self)
  1st dispatch: accept()   →  resolved by element's type
  2nd dispatch: visit_x()  →  resolved by visitor's type
```

---

### Double Dispatch Explained

Ruby uses single dispatch — the method called depends only on the receiver's type. Visitor achieves double dispatch:

```ruby
# Single dispatch: method depends on ONE object's type
shape.draw  # calls Circle#draw or Rectangle#draw

# Double dispatch: operation depends on TWO objects' types
# Step 1: element.accept(visitor) — dispatches on element type
# Step 2: visitor.visit_circle(self) — dispatches on visitor type

# The combination of element type + visitor type determines the operation
```

---

### Complete Implementation — Shape Operations

```ruby
# --- Concrete Elements ---
class Circle
  attr_reader :radius, :x, :y

  def initialize(radius, x: 0, y: 0)
    @radius = radius
    @x = x
    @y = y
  end

  def accept(visitor)
    visitor.visit_circle(self)  # double dispatch
  end

  def name = "Circle"
end

class Rectangle
  attr_reader :width, :height, :x, :y

  def initialize(width, height, x: 0, y: 0)
    @width = width
    @height = height
    @x = x
    @y = y
  end

  def accept(visitor)
    visitor.visit_rectangle(self)
  end

  def name = "Rectangle"
end

class Triangle
  attr_reader :base, :height, :x, :y

  def initialize(base, height, x: 0, y: 0)
    @base = base
    @height = height
    @x = x
    @y = y
  end

  def accept(visitor)
    visitor.visit_triangle(self)
  end

  def name = "Triangle"
end

# --- Concrete Visitors (new operations without modifying shapes) ---

# Visitor 1: Calculate area
class AreaCalculator
  attr_reader :total_area

  def initialize
    @total_area = 0.0
  end

  def visit_circle(c)
    area = Math::PI * c.radius**2
    puts "  Circle area: #{area.round(2)}"
    @total_area += area
  end

  def visit_rectangle(r)
    area = r.width * r.height
    puts "  Rectangle area: #{area.round(2)}"
    @total_area += area
  end

  def visit_triangle(t)
    area = 0.5 * t.base * t.height
    puts "  Triangle area: #{area.round(2)}"
    @total_area += area
  end
end

# Visitor 2: Export to JSON
class JSONExporter
  def initialize
    @entries = []
  end

  def visit_circle(c)
    @entries << { type: "circle", radius: c.radius, x: c.x, y: c.y }
  end

  def visit_rectangle(r)
    @entries << { type: "rectangle", width: r.width, height: r.height, x: r.x, y: r.y }
  end

  def visit_triangle(t)
    @entries << { type: "triangle", base: t.base, height: t.height, x: t.x, y: t.y }
  end

  def to_json
    require 'json'
    JSON.pretty_generate(@entries)
  end
end

# Visitor 3: Draw (render) shapes
class DrawVisitor
  def visit_circle(c)
    puts "  Drawing circle at (#{c.x},#{c.y}) with radius #{c.radius}"
  end

  def visit_rectangle(r)
    puts "  Drawing rectangle at (#{r.x},#{r.y}) size #{r.width}x#{r.height}"
  end

  def visit_triangle(t)
    puts "  Drawing triangle at (#{t.x},#{t.y}) base=#{t.base} height=#{t.height}"
  end
end

# Visitor 4: Calculate perimeter
class PerimeterCalculator
  attr_reader :total_perimeter

  def initialize
    @total_perimeter = 0.0
  end

  def visit_circle(c)
    p = 2 * Math::PI * c.radius
    puts "  Circle perimeter: #{p.round(2)}"
    @total_perimeter += p
  end

  def visit_rectangle(r)
    p = 2 * (r.width + r.height)
    puts "  Rectangle perimeter: #{p.round(2)}"
    @total_perimeter += p
  end

  def visit_triangle(t)
    side = Math.sqrt((t.base / 2.0)**2 + t.height**2)
    p = t.base + 2 * side
    puts "  Triangle perimeter: #{p.round(2)}"
    @total_perimeter += p
  end
end

# --- Client code ---
shapes = [
  Circle.new(5.0, x: 10, y: 20),
  Rectangle.new(4.0, 6.0, x: 30, y: 40),
  Triangle.new(3.0, 4.0, x: 50, y: 60),
  Circle.new(2.5, x: 70, y: 80)
]

puts "=== Area Calculation ==="
area_calc = AreaCalculator.new
shapes.each { |shape| shape.accept(area_calc) }
puts "Total area: #{area_calc.total_area.round(2)}"

puts "\n=== Drawing ==="
drawer = DrawVisitor.new
shapes.each { |shape| shape.accept(drawer) }

puts "\n=== JSON Export ==="
exporter = JSONExporter.new
shapes.each { |shape| shape.accept(exporter) }
puts exporter.to_json

puts "\n=== Perimeter Calculation ==="
perim_calc = PerimeterCalculator.new
shapes.each { |shape| shape.accept(perim_calc) }
puts "Total perimeter: #{perim_calc.total_perimeter.round(2)}"
```

---

### Dynamic Visitor with method_missing (Ruby Idiom)

Ruby's `method_missing` can automate the double dispatch, removing the need for explicit `visit_*` methods:

```ruby
class DynamicVisitor
  def visit(element)
    method_name = "visit_#{element.class.name.downcase}"
    if respond_to?(method_name, true)
      send(method_name, element)
    else
      visit_default(element)
    end
  end

  private

  def visit_default(element)
    puts "  No visitor method for #{element.class}"
  end
end

class DescriptionVisitor < DynamicVisitor
  private

  def visit_circle(c)
    "A circle with radius #{c.radius}"
  end

  def visit_rectangle(r)
    "A #{r.width}x#{r.height} rectangle"
  end

  def visit_triangle(t)
    "A triangle with base #{t.base} and height #{t.height}"
  end
end

# Elements just need a simple accept
module Visitable
  def accept(visitor)
    visitor.visit(self)
  end
end

# Reopen classes to include Visitable
[Circle, Rectangle, Triangle].each { |klass| klass.include(Visitable) }

visitor = DescriptionVisitor.new
shapes.each { |shape| puts shape.accept(visitor) }
```

---

### AST Traversal Example

A classic use case — visiting nodes of an Abstract Syntax Tree in a compiler:

```ruby
# --- AST Nodes ---
class NumberNode
  attr_reader :value

  def initialize(value)
    @value = value
  end

  def accept(visitor) = visitor.visit_number(self)
end

class BinaryOpNode
  attr_reader :op, :left, :right

  def initialize(op, left, right)
    @op = op
    @left = left
    @right = right
  end

  def accept(visitor) = visitor.visit_binary_op(self)
end

class UnaryOpNode
  attr_reader :op, :operand

  def initialize(op, operand)
    @op = op
    @operand = operand
  end

  def accept(visitor) = visitor.visit_unary_op(self)
end

# --- Visitor: Evaluate expression ---
class EvaluatorVisitor
  def visit_number(node) = node.value

  def visit_binary_op(node)
    left_val  = node.left.accept(self)
    right_val = node.right.accept(self)

    case node.op
    when "+" then left_val + right_val
    when "-" then left_val - right_val
    when "*" then left_val * right_val
    when "/" then left_val / right_val.to_f
    end
  end

  def visit_unary_op(node)
    val = node.operand.accept(self)
    node.op == "-" ? -val : val
  end
end

# --- Visitor: Pretty print expression ---
class PrintVisitor
  def visit_number(node) = node.value.to_s

  def visit_binary_op(node)
    left  = node.left.accept(self)
    right = node.right.accept(self)
    "(#{left} #{node.op} #{right})"
  end

  def visit_unary_op(node)
    operand = node.operand.accept(self)
    "(#{node.op}#{operand})"
  end
end

# --- Client code ---
# Build AST for: (3 + 4) * -(2 - 1)
ast = BinaryOpNode.new("*",
  BinaryOpNode.new("+",
    NumberNode.new(3),
    NumberNode.new(4)
  ),
  UnaryOpNode.new("-",
    BinaryOpNode.new("-",
      NumberNode.new(2),
      NumberNode.new(1)
    )
  )
)

evaluator = EvaluatorVisitor.new
puts "Result: #{ast.accept(evaluator)}"  # -7

printer = PrintVisitor.new
puts "Expression: #{ast.accept(printer)}"  # ((3 + 4) * (-(2 - 1)))
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

**Rule of thumb:** Use Visitor when the element hierarchy is stable but operations change frequently. Avoid it when new element types are added often. In Ruby, consider using `method_missing` or open classes as lighter alternatives.

---


## 6.10 Memento Pattern

> The Memento pattern captures and externalizes an object's internal state so that the object can be restored to this state later, without violating encapsulation. It provides the ability to undo/restore an object to a previous state. Ruby's `Marshal` and `clone`/`dup` make state snapshots straightforward.

---

### Why Memento?

When you need to save and restore an object's state:
- Text editor: save document state for undo
- Game: save checkpoints for loading later
- Database: transaction rollback
- Configuration: save settings before experimental changes
- Drawing app: save canvas state for undo

**The problem without Memento:**
```ruby
# BAD: Exposing internal state to save/restore
class Editor
  attr_accessor :content, :cursor_pos, :font_name
  # All public — anyone can read/modify!
  # To save state, external code must know ALL internal fields
  # and copy them manually. If we add a new field, all save/restore
  # code must be updated!
end
```

---

### Structure

```
┌──────────────────────┐     ┌──────────────────────┐     ┌──────────────────────┐
│     Originator        │     │      Memento          │     │     Caretaker         │
├──────────────────────┤     ├──────────────────────┤     ├──────────────────────┤
│ - state              │     │ - state (snapshot)    │     │ - mementos: Array    │
├──────────────────────┤     ├──────────────────────┤     ├──────────────────────┤
│ + save               │────→│ + description         │←────│ + save(memento)      │
│ + restore(memento)   │     │   (metadata only)     │     │ + undo / redo        │
└──────────────────────┘     └──────────────────────┘     └──────────────────────┘
```

**Participants:**
- **Originator:** The object whose state needs to be saved. Creates mementos and restores from them.
- **Memento:** Stores the internal state of the Originator. Opaque to other objects (encapsulation preserved).
- **Caretaker:** Manages mementos (stores, retrieves) but never examines or modifies their contents.

---

### Basic Implementation — Text Editor

```ruby
# --- Memento ---
class EditorMemento
  attr_reader :timestamp, :description

  def initialize(content, cursor_position, timestamp)
    @content = content.dup.freeze
    @cursor_position = cursor_position
    @timestamp = timestamp

    preview = content.length > 30 ? "#{content[0, 30]}..." : content
    @description = "[#{timestamp}] \"#{preview}\" (cursor: #{cursor_position})"
  end

  # Only the Originator should access these — use Ruby's access control
  protected

  attr_reader :content, :cursor_position
end

# --- Originator ---
class TextEditor
  def initialize
    @content = ""
    @cursor_position = 0
    @font_name = "Arial"
    @font_size = 12
    @save_counter = 0
  end

  # --- Normal editor operations ---
  def type(text)
    @content.insert(@cursor_position, text)
    @cursor_position += text.length
  end

  def delete_back(count)
    if @cursor_position >= count
      @content.slice!(@cursor_position - count, count)
      @cursor_position -= count
    end
  end

  def move_cursor(position)
    @cursor_position = [[position, 0].max, @content.length].min
  end

  def set_font(font, size)
    @font_name = font
    @font_size = size
  end

  def to_s
    "Content: \"#{@content}\"\n" \
    "Cursor: #{@cursor_position} | Font: #{@font_name} #{@font_size}pt"
  end

  # --- Memento operations ---
  def save
    @save_counter += 1
    puts "  [Saving state...]"
    EditorMemento.new(@content, @cursor_position, "T#{@save_counter}")
  end

  def restore(memento)
    @content = memento.send(:content).dup
    @cursor_position = memento.send(:cursor_position)
    puts "  [Restored to: #{memento.description}]"
  end
end

# --- Caretaker ---
class EditorHistory
  def initialize
    @history = []
    @current_index = -1
  end

  def save(memento)
    # Remove any "future" states (if we undid and then made new changes)
    @history = @history[0..@current_index]
    @history << memento
    @current_index = @history.size - 1
  end

  def undo
    if @current_index <= 0
      puts "  Nothing to undo!"
      return nil
    end
    @current_index -= 1
    @history[@current_index]
  end

  def redo
    if @current_index >= @history.size - 1
      puts "  Nothing to redo!"
      return nil
    end
    @current_index += 1
    @history[@current_index]
  end

  def show_history
    puts "\n--- History ---"
    @history.each_with_index do |memento, i|
      marker = i == @current_index ? " → " : "   "
      puts "#{marker}#{memento.description}"
    end
  end
end

# --- Client code ---
editor = TextEditor.new
history = EditorHistory.new

# Save initial state
history.save(editor.save)

editor.type("Hello")
puts editor
history.save(editor.save)

editor.type(" World")
puts editor
history.save(editor.save)

editor.type("! How are you?")
puts editor
history.save(editor.save)

history.show_history

# Undo
puts "\n--- Undo ---"
if (memento = history.undo)
  editor.restore(memento)
  puts editor
end

puts "\n--- Undo again ---"
if (memento = history.undo)
  editor.restore(memento)
  puts editor
end

# Redo
puts "\n--- Redo ---"
if (memento = history.redo)
  editor.restore(memento)
  puts editor
end

# New action after undo (discards redo history)
puts "\n--- New action after undo ---"
editor.type(" Ruby")
puts editor
history.save(editor.save)

history.show_history
```

---

### Memento with Marshal (Ruby Idiom)

Ruby's `Marshal` can serialize entire object state, making deep snapshots trivial:

```ruby
class GameCharacter
  attr_reader :name, :health, :level, :location, :inventory

  def initialize(name)
    @name = name
    @health = 100
    @max_health = 100
    @mana = 50
    @level = 1
    @experience = 0
    @location = "Starting Village"
    @inventory = {}
  end

  def take_damage(dmg)
    @health = [0, @health - dmg].max
    puts "#{@name} takes #{dmg} damage! HP: #{@health}/#{@max_health}"
  end

  def heal(amount)
    @health = [@max_health, @health + amount].min
    puts "#{@name} heals #{amount}! HP: #{@health}/#{@max_health}"
  end

  def gain_experience(xp)
    @experience += xp
    puts "#{@name} gains #{xp} XP! Total: #{@experience}"
    while @experience >= @level * 100
      @experience -= @level * 100
      @level += 1
      @max_health += 20
      @health = @max_health
      @mana += 10
      puts "  *** LEVEL UP! Now level #{@level} ***"
    end
  end

  def move_to(loc)
    @location = loc
    puts "#{@name} moves to #{@location}"
  end

  def add_item(item, count = 1)
    @inventory[item] = (@inventory[item] || 0) + count
    puts "#{@name} obtained #{item} x#{count}"
  end

  def to_s
    inv = @inventory.map { |item, count| "#{item}x#{count}" }.join(" ")
    <<~STATUS
      === #{@name} ===
      Level: #{@level} | XP: #{@experience}
      HP: #{@health}/#{@max_health} | MP: #{@mana}
      Location: #{@location}
      Inventory: #{inv}
    STATUS
  end

  # --- Memento via Marshal ---
  def save_game(save_name = nil)
    save_name ||= "Save_#{Time.now.strftime('%H:%M:%S')}"
    puts "\n[Game saved: #{save_name}]"
    { name: save_name, data: Marshal.dump(self), timestamp: Time.now }
  end

  def self.load_game(save)
    puts "\n[Game loaded: #{save[:name]}]"
    Marshal.load(save[:data])
  end
end

# --- Caretaker ---
class SaveManager
  def initialize
    @saves = []
  end

  def add_save(save)
    @saves << save
  end

  def latest_save = @saves.last

  def save_at(index) = @saves[index]

  def list_saves
    puts "\n--- Save Files ---"
    @saves.each_with_index do |save, i|
      puts "  [#{i}] #{save[:name]} — #{save[:timestamp]}"
    end
  end
end

# --- Client code ---
hero = GameCharacter.new("Warrior")
save_manager = SaveManager.new

hero.move_to("Dark Forest")
hero.gain_experience(80)
hero.add_item("Iron Sword")
save_manager.add_save(hero.save_game("Before Boss"))

# Fight a boss (things go badly)
hero.take_damage(60)
hero.take_damage(30)
puts hero

# Load the save!
puts "\n--- Loading save... ---"
hero = GameCharacter.load_game(save_manager.latest_save)
puts hero

# Try again with better strategy
hero.gain_experience(120)
hero.add_item("Health Potion", 3)
save_manager.add_save(hero.save_game("After Level Up"))

hero.take_damage(40)
hero.heal(50)
hero.gain_experience(200)
hero.move_to("Castle")
save_manager.add_save(hero.save_game("Victory"))

save_manager.list_saves
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
| Ruby approach | `Marshal.dump` / `Marshal.load` | Command objects with `undo` method |
| Best for | Complex state, hard to compute inverse | Simple operations with clear inverses |

**Pitfalls:**
- **Memory consumption:** Storing full state snapshots can be expensive for large objects
- **Serialization:** Objects with IO handles, Procs, or singleton methods can't be marshaled
- **Frequency:** Saving too often wastes memory; too rarely loses granularity

---


## 6.11 Interpreter Pattern

> The Interpreter pattern defines a representation for a language's grammar along with an interpreter that uses the representation to interpret sentences in the language. It maps a domain-specific language (DSL) to a class hierarchy where each rule in the grammar becomes a class. Ruby's metaprogramming and block-based DSLs make this pattern particularly natural.

---

### Why Interpreter?

When you have a simple language or grammar that needs to be evaluated:
- Mathematical expressions: `3 + 4 * 2`
- Boolean expressions: `(true AND false) OR true`
- SQL-like queries: `SELECT name WHERE age > 25`
- Configuration rules: `IF user.role == "admin" THEN allow`
- Regular expressions (conceptually)

**The problem without Interpreter:**
```ruby
# BAD: Hardcoded evaluation logic — can't extend the language
def evaluate(expression)
  # Must parse and evaluate in one monolithic method
  # Adding new operators means rewriting the parser
  # No reusable grammar representation
  # Can't compose expressions dynamically
end
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
- **AbstractExpression:** Declares `interpret` method
- **TerminalExpression:** Implements interpret for terminal symbols (literals, variables)
- **NonTerminalExpression:** Implements interpret for grammar rules (operators, compounds)
- **Context:** Contains information that's global to the interpreter (variable values, etc.)

---

### Simple Expression Evaluator

```ruby
# --- Context ---
class Context
  def initialize
    @variables = {}
  end

  def set(name, value)
    @variables[name] = value
  end

  def get(name)
    raise "Undefined variable: #{name}" unless @variables.key?(name)

    @variables[name]
  end

  def has?(name) = @variables.key?(name)
end

# --- Terminal Expressions ---
class NumberExpression
  def initialize(value)
    @value = value
  end

  def interpret(_context) = @value
  def to_s = @value.to_s
end

class VariableExpression
  def initialize(name)
    @name = name
  end

  def interpret(context) = context.get(@name)
  def to_s = @name
end

# --- Non-Terminal Expressions (Operators) ---
class AddExpression
  def initialize(left, right)
    @left = left
    @right = right
  end

  def interpret(context)
    @left.interpret(context) + @right.interpret(context)
  end

  def to_s = "(#{@left} + #{@right})"
end

class SubtractExpression
  def initialize(left, right)
    @left = left
    @right = right
  end

  def interpret(context)
    @left.interpret(context) - @right.interpret(context)
  end

  def to_s = "(#{@left} - #{@right})"
end

class MultiplyExpression
  def initialize(left, right)
    @left = left
    @right = right
  end

  def interpret(context)
    @left.interpret(context) * @right.interpret(context)
  end

  def to_s = "(#{@left} * #{@right})"
end

class DivideExpression
  def initialize(left, right)
    @left = left
    @right = right
  end

  def interpret(context)
    divisor = @right.interpret(context)
    raise "Division by zero" if divisor.zero?

    @left.interpret(context).to_f / divisor
  end

  def to_s = "(#{@left} / #{@right})"
end

# --- Build and evaluate expressions ---
context = Context.new
context.set("x", 10)
context.set("y", 5)
context.set("z", 3)

# Build expression: (x + y) * z = (10 + 5) * 3 = 45
expr1 = MultiplyExpression.new(
  AddExpression.new(
    VariableExpression.new("x"),
    VariableExpression.new("y")
  ),
  VariableExpression.new("z")
)

puts "#{expr1} = #{expr1.interpret(context)}"
# ((x + y) * z) = 45

# Build expression: x * y - z / 2 = 10 * 5 - 3 / 2 = 48.5
expr2 = SubtractExpression.new(
  MultiplyExpression.new(
    VariableExpression.new("x"),
    VariableExpression.new("y")
  ),
  DivideExpression.new(
    VariableExpression.new("z"),
    NumberExpression.new(2)
  )
)

puts "#{expr2} = #{expr2.interpret(context)}"

# Change variables and re-evaluate
context.set("x", 20)
puts "\nAfter x = 20:"
puts "#{expr1} = #{expr1.interpret(context)}"
# ((x + y) * z) = (20 + 5) * 3 = 75
```

---

### Boolean Expression Interpreter

```ruby
# --- Context ---
class BoolContext
  def initialize(vars = {})
    @variables = vars
  end

  def set(name, value) = @variables[name] = value
  def get(name) = @variables.fetch(name) { raise "Undefined: #{name}" }
end

# --- Terminal ---
class BoolConstant
  def initialize(value)
    @value = value
  end

  def interpret(_ctx) = @value
  def to_s = @value.to_s
end

class BoolVariable
  def initialize(name)
    @name = name
  end

  def interpret(ctx) = ctx.get(@name)
  def to_s = @name
end

# --- Non-Terminal ---
class AndExpression
  def initialize(left, right)
    @left = left
    @right = right
  end

  def interpret(ctx) = @left.interpret(ctx) && @right.interpret(ctx)
  def to_s = "(#{@left} AND #{@right})"
end

class OrExpression
  def initialize(left, right)
    @left = left
    @right = right
  end

  def interpret(ctx) = @left.interpret(ctx) || @right.interpret(ctx)
  def to_s = "(#{@left} OR #{@right})"
end

class NotExpression
  def initialize(operand)
    @operand = operand
  end

  def interpret(ctx) = !@operand.interpret(ctx)
  def to_s = "(NOT #{@operand})"
end

# --- Client code ---
ctx = BoolContext.new(
  "isAdmin" => true,
  "isLoggedIn" => true,
  "isBanned" => false
)

# Rule: (isLoggedIn AND isAdmin) AND (NOT isBanned)
rule = AndExpression.new(
  AndExpression.new(
    BoolVariable.new("isLoggedIn"),
    BoolVariable.new("isAdmin")
  ),
  NotExpression.new(
    BoolVariable.new("isBanned")
  )
)

puts "Rule: #{rule}"
puts "Result: #{rule.interpret(ctx) ? 'ALLOWED' : 'DENIED'}"
# Result: ALLOWED

ctx.set("isBanned", true)
puts "\nAfter banning:"
puts "Result: #{rule.interpret(ctx) ? 'ALLOWED' : 'DENIED'}"
# Result: DENIED
```

---

### DSL-Style Interpreter (Idiomatic Ruby)

Ruby's blocks and metaprogramming enable a more natural DSL approach:

```ruby
class RuleEngine
  def initialize(&block)
    @rules = []
    instance_eval(&block) if block_given?
  end

  def rule(name, &condition)
    @rules << { name: name, condition: condition }
  end

  def evaluate(context)
    @rules.each do |r|
      result = r[:condition].call(context)
      puts "#{r[:name]}: #{result ? 'PASS' : 'FAIL'}"
    end
  end
end

engine = RuleEngine.new do
  rule("Age check")     { |ctx| ctx[:age] >= 18 }
  rule("Balance check") { |ctx| ctx[:balance] > 0 }
  rule("Active check")  { |ctx| ctx[:active] == true }
  rule("Combined")      { |ctx| ctx[:age] >= 18 && ctx[:balance] > 100 }
end

engine.evaluate(age: 25, balance: 150, active: true)
# Age check: PASS
# Balance check: PASS
# Active check: PASS
# Combined: PASS

engine.evaluate(age: 16, balance: 50, active: true)
# Age check: FAIL
# Balance check: PASS
# Active check: PASS
# Combined: FAIL
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

Each node is an Expression object with an interpret method.
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
- Complex grammars — use parser generators (Treetop, Parslet, ANTLR) instead
- Performance-critical evaluation — interpreter overhead is significant
- Grammar changes frequently — class hierarchy must change with grammar

**Trade-offs:**

| Advantage | Disadvantage |
|-----------|-------------|
| Easy to implement simple grammars | Class explosion for complex grammars |
| Easy to extend (add new expressions) | Slow for complex expressions |
| Grammar rules are explicit in code | Hard to maintain for large grammars |
| Can compose expressions dynamically | Better alternatives exist for complex languages |
| Ruby DSLs offer a lighter alternative | DSLs can be harder to debug |

---


## 6.12 Null Object Pattern

> The Null Object pattern provides an object with defined neutral ("do nothing") behavior as a surrogate for the absence of an object. Instead of using `nil` to represent the absence of an object, you use a special object that implements the expected interface but does nothing. This eliminates nil checks throughout the code. Ruby's `NilClass` can be extended, but a dedicated Null Object is cleaner and more explicit.

---

### Why Null Object?

Nil references are a constant source of bugs and defensive code:
- Forgetting a nil check causes `NoMethodError: undefined method for nil:NilClass`
- Nil checks clutter the code and obscure the real logic
- Every consumer of an object must independently check for nil
- Tony Hoare called null references his "billion-dollar mistake"

**The problem without Null Object:**
```ruby
# BAD: Nil checks everywhere!
class OrderService
  def initialize(logger: nil, notifier: nil, auditor: nil)
    @logger = logger       # might be nil!
    @notifier = notifier   # might be nil!
    @auditor = auditor     # might be nil!
  end

  def process_order(order)
    # Must check EVERY dependency for nil before using it
    @logger&.log("Processing order #{order.id}")

    # ... actual business logic ...

    @notifier&.notify("Order processed: #{order.id}")

    @auditor&.record("ORDER_PROCESSED", order.id)

    # Safe navigation (&.) everywhere! Easy to forget one → crash!
    # Business logic is buried under defensive code
  end
end
```

---

### Structure

```
┌──────────────────────────┐
│   Interface (duck type)   │
├──────────────────────────┤
│ + operation               │
└──────────────────────────┘
             ▲
             │
    ┌────────┴────────┐
    │                 │
RealObject        NullObject
+ operation       + operation
  (does work)       (does nothing)
```

---

### Basic Implementation

```ruby
# --- Real Implementations ---
class ConsoleLogger
  def log(message)   = puts("[LOG] #{message}")
  def warn(message)  = puts("[WARN] #{message}")
  def error(message) = puts("[ERROR] #{message}")
  def null?          = false
end

class FileLogger
  def initialize(filename)
    @filename = filename
  end

  def log(message)   = puts("[FILE:#{@filename}] #{message}")
  def warn(message)  = puts("[FILE:#{@filename} WARN] #{message}")
  def error(message) = puts("[FILE:#{@filename} ERROR] #{message}")
  def null?          = false
end

# --- Null Object ---
class NullLogger
  def log(_message)   = nil  # intentionally empty — does nothing
  def warn(_message)  = nil
  def error(_message) = nil
  def null?           = true
end

# --- Service that uses Logger ---
class OrderService
  def initialize(logger:)
    @logger = logger  # never nil — always a valid logger (real or null)
  end

  def process_order(order_id)
    # NO nil checks needed! Just use the logger directly.
    @logger.log("Processing order: #{order_id}")

    # ... business logic ...
    @logger.log("Validating payment...")
    @logger.log("Updating inventory...")

    @logger.log("Order #{order_id} processed successfully")
  end
end

# --- Client code ---
# With real logger — logs everything
puts "=== With Console Logger ==="
service1 = OrderService.new(logger: ConsoleLogger.new)
service1.process_order("ORD-001")

puts

# With null logger — silently does nothing
puts "=== With Null Logger ==="
service2 = OrderService.new(logger: NullLogger.new)
service2.process_order("ORD-002")
puts "(no output — NullLogger does nothing)"
```

---

### Comprehensive Example — Notification System

```ruby
# --- Notification Interface (duck type) ---
class EmailNotifier
  def send_notification(recipient, message)
    puts "  EMAIL → #{recipient}: #{message}"
  end

  def type = "Email"
  def null? = false
end

class SMSNotifier
  def send_notification(recipient, message)
    puts "  SMS → #{recipient}: #{message}"
  end

  def type = "SMS"
  def null? = false
end

class PushNotifier
  def send_notification(recipient, message)
    puts "  PUSH → #{recipient}: #{message}"
  end

  def type = "Push"
  def null? = false
end

# --- Null Object ---
class NullNotifier
  def send_notification(_recipient, _message) = nil
  def type = "None (disabled)"
  def null? = true
end

# --- Discount Strategy ---
class PercentageDiscount
  def initialize(percent)
    @percent = percent
  end

  def apply(price) = price * (1 - @percent / 100.0)
  def description = "#{@percent.to_i}% off"
end

# --- Null Discount (no discount applied) ---
class NullDiscount
  def apply(price) = price  # returns price unchanged
  def description = "No discount"
end

# --- User with optional features ---
class User
  def initialize(name, email, notifier: nil, discount: nil)
    @name = name
    @email = email
    # Use Null Objects instead of nil
    @notifier = notifier || NullNotifier.new
    @discount = discount || NullDiscount.new
  end

  def place_order(item, price)
    final_price = @discount.apply(price)

    puts "#{@name} orders #{item}:"
    puts "  Original: $#{price}"
    puts "  Discount: #{@discount.description}"
    puts "  Final: $#{'%.2f' % final_price}"

    # No nil check needed! NullNotifier just does nothing.
    @notifier.send_notification(@email, "Order confirmed: #{item} for $#{'%.2f' % final_price}")
  end
end

# --- Client code ---
# Premium user with email notifications and discount
premium = User.new("Alice", "alice@example.com",
  notifier: EmailNotifier.new,
  discount: PercentageDiscount.new(20))

# Basic user with no notifications and no discount
# NullNotifier and NullDiscount are used automatically
basic = User.new("Bob", "bob@example.com")

# User with SMS notifications but no discount
sms_user = User.new("Charlie", "charlie@example.com",
  notifier: SMSNotifier.new)

puts "=== Premium User ==="
premium.place_order("Laptop", 999.99)

puts "\n=== Basic User ==="
basic.place_order("Mouse", 29.99)

puts "\n=== SMS User ==="
sms_user.place_order("Keyboard", 79.99)
```

---

### Null Object with method_missing (Ruby Idiom)

Ruby's `method_missing` can create a universal Null Object that absorbs any method call:

```ruby
class BlackHole
  def method_missing(method_name, *args, &block)
    self  # return self so chained calls also do nothing
  end

  def respond_to_missing?(_method_name, _include_private = false)
    true
  end

  def nil?  = false
  def null? = true
  def to_s  = ""
  def to_a  = []
  def to_i  = 0
  def to_f  = 0.0
  def inspect = "#<BlackHole>"
end

# Usage — absorbs any method call chain
null = BlackHole.new
null.log("test")                    # does nothing
null.foo.bar.baz                    # does nothing, returns self each time
null.send_email("x", "y").track    # does nothing
```

---

### Null Object vs nil

| Aspect | nil | Null Object |
|--------|-----|-------------|
| Safety | `NoMethodError` if not checked | Always safe to call methods |
| Code clarity | `&.` and `nil?` checks clutter code | Clean, no defensive code |
| Polymorphism | No — nil doesn't implement your interface | Yes — implements the interface |
| Default behavior | None — must handle explicitly | Defined "do nothing" behavior |
| Debugging | `NoMethodError` at runtime | Silent no-op (can be harder to debug) |
| Type safety | Any variable can be nil | Strongly typed (implements interface) |
| Ruby idiom | `&.` (safe navigation) is common | More explicit, better for complex cases |

---

### When to Use Null Object

| Scenario | Why Null Object Helps |
|----------|----------------------|
| Optional dependencies | Provide null implementation instead of nil |
| Default behavior when feature is disabled | Null object does nothing gracefully |
| Eliminating nil checks | Code is cleaner and less error-prone |
| Strategy pattern with "no strategy" option | NullStrategy returns input unchanged |
| Logging that can be disabled | NullLogger silently discards messages |
| Plugin systems with optional plugins | NullPlugin does nothing |

**Pitfalls:**
- **Silent failures:** Null objects hide the absence of real behavior — bugs can go unnoticed
- **Debugging difficulty:** Hard to tell if a null object is being used unintentionally
- **Not always appropriate:** Sometimes nil genuinely means "error" and should be handled explicitly
- **`null?` method:** Adding `null?` partially defeats the purpose — try to avoid checking it

**Best practice:** Use Null Object for optional, non-critical behavior (logging, notifications, metrics). Use explicit error handling (exceptions, `raise`) for required behavior where absence is an error. In Ruby, the safe navigation operator (`&.`) is a lighter alternative for simple cases, but Null Object is better when you need consistent polymorphic behavior.

---
