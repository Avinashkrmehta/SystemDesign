# Module 4: Design Patterns — Creational (Ruby)

> Creational patterns deal with object creation mechanisms — they abstract the instantiation process, making systems independent of how their objects are created, composed, and represented. Instead of instantiating objects directly with `.new`, creational patterns provide more flexibility in deciding which objects to create, how to create them, and when to create them.

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
```ruby
# BAD: Multiple loggers writing to the same file — race conditions, duplicate handles
log1 = Logger.new  # opens file
log2 = Logger.new  # opens same file again — conflict!
log3 = Logger.new  # and again...

# BAD: Multiple config managers — which one has the latest state?
c1 = ConfigManager.new
c1.set("timeout", 30)
c2 = ConfigManager.new
puts c2.get("timeout")  # doesn't see c1's changes!
```

---

### Basic Singleton (Naive — NOT Thread-Safe)

```ruby
class Singleton
  @instance = nil

  # Private constructor — prevents external instantiation
  private_class_method :new

  def self.instance
    @instance ||= new
    # NOT thread-safe: two threads could both see @instance as nil
  end

  def do_work
    puts "Singleton doing work"
  end
end

# Usage
s1 = Singleton.instance
s2 = Singleton.instance
puts s1.equal?(s2)  # true — same object
# Singleton.new      # => NoMethodError: private method 'new' called
```

**Problems with this approach:**
1. **Not thread-safe:** Two threads could both see `@instance` as nil and create two instances
2. **No clone/dup protection:** `s1.dup` or `s1.clone` would create copies

---

### Eager Initialization

The instance is created at class load time, avoiding thread-safety issues.

```ruby
class EagerSingleton
  # Instance created immediately when class is loaded
  @instance = new

  private_class_method :new

  def self.instance
    @instance
  end

  def configure(setting)
    puts "Configured: #{setting}"
  end
end

# Instance already exists before any call to .instance
EagerSingleton.instance.configure("database_url")
```

**Pros:**
- Thread-safe (created before any threads access it)
- Simple implementation

**Cons:**
- Instance created even if never used (wastes resources)
- Cannot pass parameters to constructor at creation time
- In Ruby, class loading order can be unpredictable in large apps

---

### Thread-Safe Singleton with Mutex

```ruby
require 'thread'

class ThreadSafeSingleton
  @instance = nil
  @mutex = Mutex.new

  private_class_method :new

  def self.instance
    # Double-checked locking pattern
    return @instance if @instance

    @mutex.synchronize do
      @instance ||= new
    end
  end

  def do_work
    puts "Working..."
  end
end

# Safe to call from multiple threads
threads = 10.times.map do
  Thread.new { ThreadSafeSingleton.instance }
end
results = threads.map(&:value)
puts results.uniq.size  # 1 — all threads got the same instance
```

**Why double-checked?**
1. **First check (without lock):** Fast path — if instance exists, return immediately (no lock overhead)
2. **Lock acquisition:** Only if instance might not exist
3. **Second check (with lock):** Another thread might have created it while we were waiting for the lock

---

### Ruby's Built-in Singleton Module (The Recommended Approach)

Ruby provides a `Singleton` module in the standard library that handles all the thread-safety and copy-prevention concerns. **This is the gold standard for Singleton in Ruby.**

```ruby
require 'singleton'

class AppConfig
  include Singleton

  def initialize
    @settings = {}
    puts "AppConfig created"
  end

  def set(key, value)
    @settings[key] = value
  end

  def get(key)
    @settings[key]
  end

  def all
    @settings.dup
  end
end

# Usage
config = AppConfig.instance
config.set("timeout", 30)
config.set("retries", 3)

# Same instance everywhere
puts AppConfig.instance.get("timeout")  # 30

# Cannot instantiate directly
# AppConfig.new    # => NoMethodError: private method 'new' called
# config.clone     # => TypeError: can't clone instance of singleton
# config.dup       # => TypeError: can't dup instance of singleton
```

**What `include Singleton` does:**
- Makes `.new` private
- Provides `.instance` class method (thread-safe, lazy initialization)
- Prevents `clone` and `dup`
- Uses `Mutex` internally for thread safety
- Handles marshaling (serialization) correctly

---

### Singleton with Clone/Dup Protection (Manual Implementation)

If you don't want to use the `Singleton` module, protect against cloning manually:

```ruby
class ManualSingleton
  @instance = nil
  @mutex = Mutex.new

  private_class_method :new

  def self.instance
    return @instance if @instance
    @mutex.synchronize { @instance ||= new }
  end

  # Prevent cloning
  def clone
    raise TypeError, "can't clone instance of singleton #{self.class}"
  end

  # Prevent duplication
  def dup
    raise TypeError, "can't dup instance of singleton #{self.class}"
  end

  private

  def initialize
    @data = {}
  end
end
```

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
```ruby
# BAD: Hard to test — can't inject a mock
class OrderService
  def process_order(order)
    Database.instance.save(order)  # tightly coupled!
  end
end

# GOOD: Dependency injection — testable
class OrderService
  def initialize(database)
    @db = database
  end

  def process_order(order)
    @db.save(order)  # can inject mock for testing
  end
end
```

2. **When state makes testing unpredictable:**
```ruby
# Singleton state persists between tests — tests affect each other!
RSpec.describe "Test1" do
  it "sets state" do
    AppConfig.instance.set("mode", "test")
  end
end

RSpec.describe "Test2" do
  it "sees leftover state" do
    # AppConfig still has "mode" => "test" from Test1! Tests are not independent!
    expect(AppConfig.instance.get("mode")).to eq("test")  # SURPRISE!
  end
end
```

3. **When you might need multiple instances later:**
   - "We'll only ever have one database" → Later: "We need read replicas"
   - "One cache is enough" → Later: "We need L1 and L2 caches"

4. **When it hides dependencies:**
```ruby
# BAD: Hidden dependency — not visible in constructor/interface
class ReportGenerator
  def generate
    data = Database.instance.query("...")     # hidden!
    config = Config.instance.get("format")   # hidden!
    Logger.instance.log("Generating report") # hidden!
  end
end
# What does ReportGenerator depend on? You can't tell from its interface!
```

---

### Singleton vs Global Variable

| Aspect | Singleton | Global Variable ($global) |
|--------|-----------|---------------------------|
| Initialization | Controlled (lazy or eager) | At assignment time |
| Construction | Can have logic in initialize | Simple assignment only |
| Access control | Through `.instance` method | Direct access from anywhere |
| Inheritance | Can use polymorphism | No |
| Thread safety | Can be made thread-safe | No built-in safety |
| Encapsulation | Private constructor, controlled interface | No encapsulation |
| Testability | Both are problematic | Both are problematic |

**Both share the fundamental problem:** They introduce global state, making code harder to test, reason about, and parallelize.

**Preferred alternative — Dependency Injection:**
```ruby
# Instead of Singleton, create one instance and pass it around
logger = FileLogger.new("app.log")
config = AppConfig.new("config.yaml")
db = PostgresDB.new(config.db_url)

# Pass dependencies explicitly
order_service = OrderService.new(db: db, logger: logger)
user_service = UserService.new(db: db, logger: logger)
# Clear dependencies, testable, no global state
```

---

### Complete Production-Ready Singleton Example

```ruby
require 'singleton'
require 'thread'
require 'time'

class Logger
  include Singleton

  LEVELS = { debug: 0, info: 1, warn: 2, error: 3, fatal: 4 }.freeze

  def initialize
    @min_level = :info
    @log_file = File.open("application.log", "a")
    @mutex = Mutex.new
    @console_output = true
  end

  def level=(level)
    @mutex.synchronize { @min_level = level }
  end

  def console_output=(enabled)
    @mutex.synchronize { @console_output = enabled }
  end

  def log(level, message)
    return if LEVELS[level] < LEVELS[@min_level]

    @mutex.synchronize do
      formatted = "[#{timestamp}] [#{level.to_s.upcase.ljust(5)}] #{message}"

      @log_file.puts(formatted)
      @log_file.flush

      puts formatted if @console_output
    end
  end

  def debug(msg) = log(:debug, msg)
  def info(msg)  = log(:info, msg)
  def warn(msg)  = log(:warn, msg)
  def error(msg) = log(:error, msg)
  def fatal(msg) = log(:fatal, msg)

  def close
    @mutex.synchronize do
      info("Logger shutting down")
      @log_file.close
    end
  end

  private

  def timestamp
    Time.now.strftime("%Y-%m-%d %H:%M:%S")
  end
end

# Usage
Logger.instance.level = :debug
Logger.instance.info("Application started")
Logger.instance.debug("Loading configuration...")
Logger.instance.warn("Config file not found, using defaults")
Logger.instance.error("Failed to connect to database")

# At shutdown
at_exit { Logger.instance.close }
```

---


## 4.2 Factory Method Pattern

> The Factory Method pattern defines an interface for creating an object, but lets subclasses decide which class to instantiate. It defers instantiation to subclasses, promoting loose coupling between the creator and the concrete products.

---

### The Problem Factory Method Solves

```ruby
# BAD: Client code directly creates concrete objects — tightly coupled
class NotificationService
  def send_alert(message, type)
    case type
    when "email"
      n = EmailNotification.new(message)  # hardcoded!
      n.send
    when "sms"
      n = SMSNotification.new(message)    # hardcoded!
      n.send
    when "push"
      n = PushNotification.new(message)   # hardcoded!
      n.send
    end
    # Adding a new type requires modifying this code — violates OCP!
  end
end
```

**Problems:**
1. Violates Open/Closed Principle — must modify existing code to add new types
2. Client is coupled to ALL concrete classes
3. Object creation logic is scattered and duplicated
4. Hard to test — can't substitute mock objects

---

### Simple Factory (Not a GoF Pattern, but Common)

A single class with a method that creates objects based on input. Simple but limited.

```ruby
# --- Product hierarchy ---
class Transport
  def deliver
    raise NotImplementedError
  end

  def calculate_cost(distance)
    raise NotImplementedError
  end

  def type
    raise NotImplementedError
  end
end

class Truck < Transport
  def deliver
    puts "Delivering by road in a truck"
  end

  def calculate_cost(distance)
    distance * 1.5  # $1.5 per km
  end

  def type = "Truck"
end

class Ship < Transport
  def deliver
    puts "Delivering by sea in a ship"
  end

  def calculate_cost(distance)
    distance * 0.8  # $0.8 per km (cheaper but slower)
  end

  def type = "Ship"
end

class Airplane < Transport
  def deliver
    puts "Delivering by air in an airplane"
  end

  def calculate_cost(distance)
    distance * 5.0  # $5.0 per km (expensive but fast)
  end

  def type = "Airplane"
end

# --- Simple Factory ---
class TransportFactory
  def self.create(type)
    case type
    when "truck"    then Truck.new
    when "ship"     then Ship.new
    when "airplane" then Airplane.new
    else raise ArgumentError, "Unknown transport type: #{type}"
    end
  end
end

# Usage
transport = TransportFactory.create("truck")
transport.deliver
puts "Cost for 100km: $#{transport.calculate_cost(100)}"
```

**Limitations of Simple Factory:**
- Still violates OCP (must modify the factory to add new types)
- All creation logic in one place — can become a God class
- No polymorphism in the creation process itself

---

### Factory Method Pattern (GoF)

The key insight: **make the factory method overridable** so subclasses can override it to create different products.

```ruby
# --- Product interface ---
class Document
  def open
    raise NotImplementedError
  end

  def save
    raise NotImplementedError
  end

  def close
    raise NotImplementedError
  end

  def extension
    raise NotImplementedError
  end
end

class PDFDocument < Document
  def open  = puts "Opening PDF document"
  def save  = puts "Saving PDF (compressing...)"
  def close = puts "Closing PDF"
  def extension = ".pdf"
end

class WordDocument < Document
  def open  = puts "Opening Word document"
  def save  = puts "Saving Word document (formatting...)"
  def close = puts "Closing Word document"
  def extension = ".docx"
end

class SpreadsheetDocument < Document
  def open  = puts "Opening Spreadsheet"
  def save  = puts "Saving Spreadsheet (recalculating...)"
  def close = puts "Closing Spreadsheet"
  def extension = ".xlsx"
end

# --- Creator (abstract) with Factory Method ---
class Application
  def initialize
    @documents = []
  end

  # THE FACTORY METHOD — subclasses override this to create specific documents
  def create_document
    raise NotImplementedError, "#{self.class} must implement create_document"
  end

  # Template Method that USES the factory method
  def new_document
    doc = create_document  # calls the overridden factory method
    doc.open
    @documents << doc
    puts "Document created. Total: #{@documents.size}"
  end

  def save_all
    @documents.each(&:save)
  end

  def close_all
    @documents.each(&:close)
    @documents.clear
  end
end

# --- Concrete Creators ---
class PDFApplication < Application
  def create_document
    puts "PDFApplication creating PDF..."
    PDFDocument.new
  end
end

class WordApplication < Application
  def create_document
    puts "WordApplication creating Word doc..."
    WordDocument.new
  end
end

class SpreadsheetApplication < Application
  def create_document
    puts "SpreadsheetApplication creating spreadsheet..."
    SpreadsheetDocument.new
  end
end

# --- Client code ---
def work_with_app(app)
  app.new_document
  app.new_document
  app.save_all
  app.close_all
end

pdf_app = PDFApplication.new
word_app = WordApplication.new

work_with_app(pdf_app)   # creates PDFs
work_with_app(word_app)  # creates Word docs
# Client code doesn't know or care about concrete document types!
```

**Structure:**
```
┌─────────────────────┐         ┌──────────────────┐
│   Application       │         │    Document      │
│   (Creator)         │         │    (Product)     │
├─────────────────────┤         ├──────────────────┤
│ + new_document()    │────────→│ + open()         │
│ + save_all()        │ creates │ + save()         │
│ + create_document() │ ←──────→│ + close()        │
│   {abstract}        │         └──────────────────┘
└─────────────────────┘                  ▲
          ▲                              │
          │                    ┌─────────┼─────────┐
┌─────────┴──────────┐        │         │         │
│  PDFApplication    │   PDFDocument  WordDoc  SpreadsheetDoc
│  create_document() │
│  → returns PDF     │
└────────────────────┘
```

---

### Simple Factory vs Factory Method

| Aspect | Simple Factory | Factory Method |
|--------|---------------|----------------|
| Structure | One class with class method | Abstract creator + concrete creators |
| Extensibility | Must modify factory to add types (violates OCP) | Add new creator subclass (OCP compliant) |
| Polymorphism | No polymorphism in creation | Creator is polymorphic |
| Complexity | Simple | More classes, more structure |
| Use case | Few types, unlikely to change | Many types, expected to grow |
| GoF pattern? | No (it's an idiom) | Yes |

---

### Parameterized Factory Method

The factory method takes parameters to decide which product to create, while still allowing subclass customization.

```ruby
class Notification
  def send(message)
    raise NotImplementedError
  end
end

class EmailNotification < Notification
  def initialize(recipient)
    @recipient = recipient
  end

  def send(message)
    puts "EMAIL to #{@recipient}: #{message}"
  end
end

class SMSNotification < Notification
  def initialize(phone_number)
    @phone_number = phone_number
  end

  def send(message)
    puts "SMS to #{@phone_number}: #{message}"
  end
end

class PushNotification < Notification
  def initialize(device_token)
    @device_token = device_token
  end

  def send(message)
    puts "PUSH to device #{@device_token}: #{message}"
  end
end

class SlackNotification < Notification
  def initialize(channel)
    @channel = channel
  end

  def send(message)
    puts "SLACK ##{@channel}: #{message}"
  end
end

# --- Parameterized Factory Method in Creator ---
class AlertService
  CHANNELS = %i[email sms push slack].freeze

  # Parameterized factory method — can be overridden
  def create_notification(channel, target)
    case channel
    when :email then EmailNotification.new(target)
    when :sms   then SMSNotification.new(target)
    when :push  then PushNotification.new(target)
    when :slack then SlackNotification.new(target)
    else raise ArgumentError, "Unknown channel: #{channel}"
    end
  end

  def send_alert(channel, target, message)
    notification = create_notification(channel, target)
    notification.send(message)
  end
end

# --- Subclass can override to customize creation ---
class PremiumAlertService < AlertService
  def create_notification(channel, target)
    puts "[Premium] Creating notification..."
    # Premium service might add logging, retry logic, or use enhanced versions
    super
  end
end
```

---

### Factory Method with Registration (Self-Registering Factory)

A more extensible approach where products register themselves with the factory — no case/when needed.

```ruby
class Shape
  def draw
    raise NotImplementedError
  end

  def area
    raise NotImplementedError
  end

  def name
    raise NotImplementedError
  end
end

# --- Self-registering Factory ---
class ShapeFactory
  @registry = {}

  class << self
    # Register a new shape type
    def register(type_name, &creator)
      @registry[type_name] = creator
    end

    # Create a shape by registered name
    def create(type_name, **params)
      creator = @registry[type_name]
      raise ArgumentError, "Unknown shape type: #{type_name}" unless creator
      creator.call(**params)
    end

    # List all registered types
    def registered_types
      @registry.keys
    end
  end
end

# --- Concrete shapes that self-register ---
class Circle < Shape
  attr_accessor :radius

  def initialize(radius: 1.0)
    @radius = radius
  end

  def draw = puts "Drawing circle r=#{@radius}"
  def area = Math::PI * @radius ** 2
  def name = "Circle"

  # Self-registration
  ShapeFactory.register("circle") { |**params| Circle.new(**params) }
end

class Rectangle < Shape
  attr_accessor :width, :height

  def initialize(width: 1.0, height: 1.0)
    @width = width
    @height = height
  end

  def draw = puts "Drawing rect #{@width}x#{@height}"
  def area = @width * @height
  def name = "Rectangle"

  # Self-registration
  ShapeFactory.register("rectangle") { |**params| Rectangle.new(**params) }
end

class Triangle < Shape
  attr_accessor :base, :height

  def initialize(base: 1.0, height: 1.0)
    @base = base
    @height = height
  end

  def draw = puts "Drawing triangle base=#{@base} height=#{@height}"
  def area = 0.5 * @base * @height
  def name = "Triangle"

  ShapeFactory.register("triangle") { |**params| Triangle.new(**params) }
end

# Usage — adding new shapes requires NO modification to factory or client
circle = ShapeFactory.create("circle", radius: 5.0)
rect = ShapeFactory.create("rectangle", width: 3.0, height: 4.0)
circle.draw  # Drawing circle r=5.0
rect.draw    # Drawing rect 3.0x4.0

puts ShapeFactory.registered_types.inspect  # ["circle", "rectangle", "triangle"]
```

**Benefits of self-registering factory:**
- Truly Open/Closed — add new shapes by just creating a new class file
- No case/when chains
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
- GUI frameworks: `create_button`, `create_window` — platform-specific
- Database drivers: `create_connection` — DB-specific
- Serialization: `create_parser` — format-specific (JSON, XML, YAML)
- Game engines: `create_enemy`, `create_weapon` — level-specific
- Rails: `ActiveRecord::Base.connection` — adapter-specific

---


## 4.3 Abstract Factory Pattern

> The Abstract Factory pattern provides an interface for creating **families of related or dependent objects** without specifying their concrete classes. It's essentially a "factory of factories" — a super-factory that creates other factories.

---

### The Problem Abstract Factory Solves

When you have multiple families of related products and need to ensure that products from the same family are used together:

```ruby
# BAD: Mixing products from different families
btn = WindowsButton.new      # Windows style
scroll = MacScrollbar.new    # Mac style — INCONSISTENT!
menu = LinuxMenu.new         # Linux style — CHAOS!

# The UI looks broken because components from different platforms are mixed
```

**The requirement:** Create entire families of objects that are designed to work together, and make it easy to switch between families.

---

### Structure

```
┌─────────────────────────┐
│    AbstractFactory       │
├─────────────────────────┤
│ + create_button()       │──────→ AbstractButton
│ + create_checkbox()     │──────→ AbstractCheckbox
│ + create_text_input()   │──────→ AbstractTextInput
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

```ruby
# ============================================================
# ABSTRACT PRODUCTS — define interfaces for each product type
# ============================================================

class Button
  def render      = raise(NotImplementedError)
  def on_click(&handler) = raise(NotImplementedError)
  def label=(label) = raise(NotImplementedError)
end

class Checkbox
  def render      = raise(NotImplementedError)
  def toggle      = raise(NotImplementedError)
  def checked?    = raise(NotImplementedError)
  def label=(label) = raise(NotImplementedError)
end

class TextInput
  def render      = raise(NotImplementedError)
  def text=(text) = raise(NotImplementedError)
  def text        = raise(NotImplementedError)
  def placeholder=(placeholder) = raise(NotImplementedError)
end

class Dialog
  def show(title, message) = raise(NotImplementedError)
end

# ============================================================
# CONCRETE PRODUCTS — Windows Family
# ============================================================

class WindowsButton < Button
  attr_writer :label

  def initialize
    @label = ""
    @click_handler = nil
  end

  def render
    puts "[ #{@label} ]  (Windows flat button)"
  end

  def on_click(&handler)
    @click_handler = handler
  end
end

class WindowsCheckbox < Checkbox
  attr_writer :label

  def initialize
    @label = ""
    @checked = false
  end

  def render
    mark = @checked ? "X" : " "
    puts "[#{mark}] #{@label}  (Windows checkbox)"
  end

  def toggle
    @checked = !@checked
  end

  def checked? = @checked
end

class WindowsTextInput < TextInput
  attr_accessor :text
  attr_writer :placeholder

  def initialize
    @text = ""
    @placeholder = ""
  end

  def render
    display = @text.empty? ? @placeholder : @text
    puts "|#{display}|  (Windows text field)"
  end
end

class WindowsDialog < Dialog
  def show(title, message)
    puts "╔══ #{title} ══╗"
    puts "║ #{message}"
    puts "║        [OK]        ║"
    puts "╚════════════════════╝  (Windows dialog)"
  end
end

# ============================================================
# CONCRETE PRODUCTS — macOS Family
# ============================================================

class MacButton < Button
  attr_writer :label

  def initialize
    @label = ""
    @click_handler = nil
  end

  def render
    puts "( #{@label} )  (macOS rounded button)"
  end

  def on_click(&handler)
    @click_handler = handler
  end
end

class MacCheckbox < Checkbox
  attr_writer :label

  def initialize
    @label = ""
    @checked = false
  end

  def render
    mark = @checked ? "☑" : "☐"
    puts "#{mark} #{@label}  (macOS checkbox)"
  end

  def toggle
    @checked = !@checked
  end

  def checked? = @checked
end

class MacTextInput < TextInput
  attr_accessor :text
  attr_writer :placeholder

  def initialize
    @text = ""
    @placeholder = ""
  end

  def render
    display = @text.empty? ? @placeholder : @text
    puts "⌈#{display}⌉  (macOS text field)"
  end
end

class MacDialog < Dialog
  def show(title, message)
    puts "┌── #{title} ──┐"
    puts "│ #{message}"
    puts "│      ( OK )       │"
    puts "└───────────────────┘  (macOS dialog)"
  end
end

# ============================================================
# ABSTRACT FACTORY — interface for creating product families
# ============================================================

class UIFactory
  def create_button     = raise(NotImplementedError)
  def create_checkbox   = raise(NotImplementedError)
  def create_text_input = raise(NotImplementedError)
  def create_dialog     = raise(NotImplementedError)
  def theme_name        = raise(NotImplementedError)
end

# ============================================================
# CONCRETE FACTORIES — each creates a complete family
# ============================================================

class WindowsUIFactory < UIFactory
  def create_button     = WindowsButton.new
  def create_checkbox   = WindowsCheckbox.new
  def create_text_input = WindowsTextInput.new
  def create_dialog     = WindowsDialog.new
  def theme_name        = "Windows"
end

class MacUIFactory < UIFactory
  def create_button     = MacButton.new
  def create_checkbox   = MacCheckbox.new
  def create_text_input = MacTextInput.new
  def create_dialog     = MacDialog.new
  def theme_name        = "macOS"
end

# ============================================================
# CLIENT CODE — works with factories and products via interfaces
# ============================================================

class SettingsPage
  def initialize(factory)
    @name_field = factory.create_text_input
    @name_field.placeholder = "Enter your name"

    @email_field = factory.create_text_input
    @email_field.placeholder = "Enter your email"

    @dark_mode_toggle = factory.create_checkbox
    @dark_mode_toggle.label = "Dark Mode"

    @notifications_toggle = factory.create_checkbox
    @notifications_toggle.label = "Enable Notifications"

    @save_button = factory.create_button
    @save_button.label = "Save"

    @cancel_button = factory.create_button
    @cancel_button.label = "Cancel"

    @confirm_dialog = factory.create_dialog
  end

  def render
    puts "=== Settings ==="
    @name_field.render
    @email_field.render
    @dark_mode_toggle.render
    @notifications_toggle.render
    @save_button.render
    @cancel_button.render
  end

  def save
    @confirm_dialog.show("Confirm", "Save changes?")
  end
end

# --- Selecting the factory at runtime ---
def create_factory
  case RUBY_PLATFORM
  when /darwin/  then MacUIFactory.new
  when /win|mingw/ then WindowsUIFactory.new
  else WindowsUIFactory.new  # default
  end
end

factory = create_factory
puts "Using theme: #{factory.theme_name}\n\n"

settings = SettingsPage.new(factory)
settings.render
settings.save
```

---

### Factory of Factories

When you have multiple abstract factories and need to select one dynamically:

```ruby
class UIFactoryProvider
  @registry = {}

  class << self
    def register(name, &creator)
      @registry[name] = creator
    end

    def get_factory(name)
      creator = @registry[name]
      raise ArgumentError, "Unknown UI factory: #{name}" unless creator
      creator.call
    end

    def available_factories
      @registry.keys
    end
  end
end

# Registration (could be in separate files)
UIFactoryProvider.register("windows") { WindowsUIFactory.new }
UIFactoryProvider.register("macos")   { MacUIFactory.new }

# Usage — select factory by name (from config, user preference, etc.)
theme = ENV.fetch("UI_THEME", "macos")
factory = UIFactoryProvider.get_factory(theme)
page = SettingsPage.new(factory)
```

---

### Abstract Factory vs Factory Method

| Aspect | Factory Method | Abstract Factory |
|--------|---------------|-----------------|
| Creates | ONE product | FAMILY of related products |
| Method | Single overridable method | Multiple creation methods |
| Inheritance | Subclass overrides one method | Subclass implements all creation methods |
| Focus | How to create one object | How to create a consistent set of objects |
| Complexity | Simpler | More complex |
| Use case | One product varies | Multiple products must be consistent |
| Example | `create_button` | `create_button` + `create_checkbox` + `create_dialog` |

**When to use Abstract Factory:**
- System must be independent of how products are created
- System needs to work with multiple families of products
- Products in a family are designed to work together (consistency constraint)
- You want to provide a library of products, revealing only interfaces

---

### Another Example — Database Access Layer

```ruby
# Abstract products
class Connection
  def connect(connection_string) = raise(NotImplementedError)
  def disconnect                 = raise(NotImplementedError)
  def connected?                 = raise(NotImplementedError)
end

class Command
  def query=(sql)                        = raise(NotImplementedError)
  def add_parameter(name, value)         = raise(NotImplementedError)
  def execute_non_query                  = raise(NotImplementedError)
end

# Abstract Factory
class DatabaseFactory
  def create_connection          = raise(NotImplementedError)
  def create_command(connection) = raise(NotImplementedError)
  def database_name              = raise(NotImplementedError)
end

# Concrete: PostgreSQL family
class PgConnection < Connection
  def initialize
    @connected = false
  end

  def connect(connection_string)
    puts "PostgreSQL connecting to: #{connection_string}"
    @connected = true
  end

  def disconnect
    @connected = false
  end

  def connected? = @connected
end

class PgCommand < Command
  def initialize
    @query = ""
    @params = []
  end

  def query=(sql)
    @query = sql
  end

  def add_parameter(name, value)
    @params << [name, value]
  end

  def execute_non_query
    puts "PG executing: #{@query} with #{@params.size} params"
    1  # rows affected
  end
end

class PostgreSQLFactory < DatabaseFactory
  def create_connection          = PgConnection.new
  def create_command(_connection) = PgCommand.new
  def database_name              = "PostgreSQL"
end

# Concrete: MySQL family
class MySQLConnection < Connection
  def initialize
    @connected = false
  end

  def connect(connection_string)
    puts "MySQL connecting to: #{connection_string}"
    @connected = true
  end

  def disconnect
    @connected = false
  end

  def connected? = @connected
end

class MySQLCommand < Command
  def initialize
    @query = ""
    @params = []
  end

  def query=(sql)
    @query = sql
  end

  def add_parameter(name, value)
    @params << [name, value]
  end

  def execute_non_query
    puts "MySQL executing: #{@query} with #{@params.size} params"
    1
  end
end

class MySQLFactory < DatabaseFactory
  def create_connection          = MySQLConnection.new
  def create_command(_connection) = MySQLCommand.new
  def database_name              = "MySQL"
end

# Client code — completely database-agnostic
class UserRepository
  def initialize(db_factory, connection_string)
    @db_factory = db_factory
    @connection_string = connection_string
  end

  def create_user(name, email)
    conn = @db_factory.create_connection
    conn.connect(@connection_string)

    cmd = @db_factory.create_command(conn)
    cmd.query = "INSERT INTO users (name, email) VALUES ($1, $2)"
    cmd.add_parameter("name", name)
    cmd.add_parameter("email", email)
    cmd.execute_non_query

    conn.disconnect
  end
end

# Switch databases by changing one line
pg_factory = PostgreSQLFactory.new
# mysql_factory = MySQLFactory.new
repo = UserRepository.new(pg_factory, "host=localhost dbname=myapp")
repo.create_user("Alice", "alice@example.com")
```

---


## 4.4 Builder Pattern

> The Builder pattern separates the construction of a complex object from its representation, allowing the same construction process to create different representations. It's ideal when an object requires many steps to create, or when you want to construct different variations of an object using the same building process.

---

### The Problem Builder Solves

```ruby
# BAD: Constructor with too many parameters — "telescoping constructor" anti-pattern
class Pizza
  def initialize(size, crust, sauce, cheese, pepperoni, mushrooms, onions,
                 olives, peppers, bacon, pineapple, extra_cheese, thin_crust)
    # Which boolean is which?! Easy to mix up parameters.
  end
end

# Calling this is error-prone and unreadable:
Pizza.new("large", "thin", "tomato", true, true, false, true, false, false, true, false, true, false)
# What does the 7th 'false' mean? Nobody knows without checking the constructor.
```

**Problems:**
1. Too many constructor parameters — hard to read, easy to mix up
2. Many parameters are optional — leads to complex default handling
3. Object might be in an inconsistent state during construction
4. No way to enforce construction order or validate intermediate state

**Note:** Ruby's keyword arguments mitigate this somewhat, but Builder still shines for complex multi-step construction with validation.

---

### Basic Builder Pattern

```ruby
# --- The Product (complex object to build) ---
class HttpRequest
  attr_accessor :method, :url, :headers, :body, :timeout,
                :follow_redirects, :max_retries, :auth_token

  def initialize
    @headers = {}
  end

  def send
    puts "#{@method} #{@url}"
    @headers.each { |key, value| puts "  #{key}: #{value}" }
    puts "  Body: #{@body}" unless @body.nil? || @body.empty?
    puts "  Timeout: #{@timeout}ms, Retries: #{@max_retries}"
  end
end

# --- The Builder ---
class HttpRequestBuilder
  def initialize
    @request = HttpRequest.new
    # Set defaults
    @request.method = "GET"
    @request.timeout = 30_000
    @request.follow_redirects = true
    @request.max_retries = 3
  end

  def set_method(method)
    @request.method = method
    self
  end

  def set_url(url)
    @request.url = url
    self
  end

  def add_header(key, value)
    @request.headers[key] = value
    self
  end

  def set_body(body)
    @request.body = body
    self
  end

  def set_timeout(ms)
    raise ArgumentError, "Timeout must be positive" unless ms > 0
    @request.timeout = ms
    self
  end

  def set_follow_redirects(follow)
    @request.follow_redirects = follow
    self
  end

  def set_max_retries(retries)
    raise ArgumentError, "Retries cannot be negative" if retries < 0
    @request.max_retries = retries
    self
  end

  def set_auth_token(token)
    @request.auth_token = token
    @request.headers["Authorization"] = "Bearer #{token}"
    self
  end

  # Build and validate
  def build
    raise "URL is required" if @request.url.nil? || @request.url.empty?

    if %w[POST PUT].include?(@request.method)
      @request.headers["Content-Type"] ||= "application/json"
    end

    @request
  end
end

# Usage — clear, readable, self-documenting
req = HttpRequestBuilder.new
  .set_method("POST")
  .set_url("https://api.example.com/users")
  .add_header("Content-Type", "application/json")
  .add_header("Accept", "application/json")
  .set_body('{"name": "Alice", "email": "alice@example.com"}')
  .set_auth_token("abc123xyz")
  .set_timeout(5000)
  .set_max_retries(2)
  .build

req.send
```

---

### Director Class

The **Director** defines the ORDER in which to call building steps. It encapsulates specific construction recipes, while the builder provides the implementation of each step.

```ruby
# --- Product ---
class House
  attr_accessor :foundation, :structure, :roof, :interior, :exterior,
                :floors, :has_garage, :has_swimming_pool, :has_garden,
                :heating_system

  def describe
    puts "House: #{@floors} floors"
    puts "  Foundation: #{@foundation}"
    puts "  Structure: #{@structure}"
    puts "  Roof: #{@roof}"
    puts "  Interior: #{@interior}"
    puts "  Exterior: #{@exterior}"
    puts "  Heating: #{@heating_system}"
    puts "  Garage: #{@has_garage ? 'Yes' : 'No'}"
    puts "  Pool: #{@has_swimming_pool ? 'Yes' : 'No'}"
    puts "  Garden: #{@has_garden ? 'Yes' : 'No'}"
  end
end

# --- Abstract Builder ---
class HouseBuilder
  def initialize
    reset
  end

  def reset
    @house = House.new
    @house.has_garage = false
    @house.has_swimming_pool = false
    @house.has_garden = false
    self
  end

  def build_foundation = raise(NotImplementedError)
  def build_structure  = raise(NotImplementedError)
  def build_roof       = raise(NotImplementedError)
  def build_interior   = raise(NotImplementedError)
  def build_exterior   = raise(NotImplementedError)

  def set_floors(n)
    @house.floors = n
    self
  end

  def add_garage
    @house.has_garage = true
    self
  end

  def add_swimming_pool
    @house.has_swimming_pool = true
    self
  end

  def add_garden
    @house.has_garden = true
    self
  end

  def set_heating(type)
    @house.heating_system = type
    self
  end

  def result
    @house
  end
end

# --- Concrete Builder: Wooden House ---
class WoodenHouseBuilder < HouseBuilder
  def build_foundation
    @house.foundation = "Concrete slab with wooden posts"
    self
  end

  def build_structure
    @house.structure = "Timber frame with wooden walls"
    self
  end

  def build_roof
    @house.roof = "Wooden shingles"
    self
  end

  def build_interior
    @house.interior = "Hardwood floors, wooden paneling"
    self
  end

  def build_exterior
    @house.exterior = "Cedar wood siding"
    self
  end
end

# --- Concrete Builder: Stone House ---
class StoneHouseBuilder < HouseBuilder
  def build_foundation
    @house.foundation = "Deep concrete foundation with rebar"
    self
  end

  def build_structure
    @house.structure = "Stone walls with steel reinforcement"
    self
  end

  def build_roof
    @house.roof = "Slate tiles"
    self
  end

  def build_interior
    @house.interior = "Marble floors, plastered walls"
    self
  end

  def build_exterior
    @house.exterior = "Natural stone facade"
    self
  end
end

# --- Director: defines construction recipes ---
class ConstructionDirector
  attr_writer :builder

  # Recipe: Simple house
  def construct_simple_house
    @builder.reset
      .build_foundation
      .build_structure
      .build_roof
      .build_interior
      .build_exterior
      .set_floors(1)
      .set_heating("Electric")
    @builder.result
  end

  # Recipe: Luxury house
  def construct_luxury_house
    @builder.reset
      .build_foundation
      .build_structure
      .build_roof
      .build_interior
      .build_exterior
      .set_floors(3)
      .add_garage
      .add_swimming_pool
      .add_garden
      .set_heating("Underfloor radiant")
    @builder.result
  end

  # Recipe: Minimal cabin
  def construct_cabin
    @builder.reset
      .build_foundation
      .build_structure
      .build_roof
      .set_floors(1)
      .set_heating("Wood stove")
    @builder.result
  end
end

# Usage
wood_builder = WoodenHouseBuilder.new
stone_builder = StoneHouseBuilder.new
director = ConstructionDirector.new

director.builder = wood_builder
wooden_cabin = director.construct_cabin
wooden_cabin.describe

director.builder = stone_builder
stone_mansion = director.construct_luxury_house
stone_mansion.describe
```

**Director benefits:**
- Encapsulates construction algorithms (recipes)
- Same director can work with different builders
- Client doesn't need to know the construction steps
- Easy to add new recipes without changing builders

---

### Fluent Builder (Method Chaining)

The most common modern usage — each setter returns `self`, enabling chained calls.

```ruby
class QueryBuilder
  def initialize
    @table = nil
    @columns = []
    @conditions = []
    @order_by = []
    @limit_val = nil
    @offset_val = nil
    @joins = []
    @distinct = false
  end

  def select(*cols)
    @columns = cols.flatten
    self
  end

  def select_all
    @columns = ["*"]
    self
  end

  def from(table_name)
    @table = table_name
    self
  end

  def where(condition)
    @conditions << condition
    self
  end

  def and_where(condition)
    @conditions << "AND #{condition}"
    self
  end

  def or_where(condition)
    @conditions << "OR #{condition}"
    self
  end

  def join(join_clause)
    @joins << "JOIN #{join_clause}"
    self
  end

  def left_join(join_clause)
    @joins << "LEFT JOIN #{join_clause}"
    self
  end

  def order_by_asc(column)
    @order_by << "#{column} ASC"
    self
  end

  def order_by_desc(column)
    @order_by << "#{column} DESC"
    self
  end

  def limit(n)
    @limit_val = n
    self
  end

  def offset(n)
    @offset_val = n
    self
  end

  def distinct(d = true)
    @distinct = d
    self
  end

  def build
    raise "Table name is required" if @table.nil? || @table.empty?

    query = "SELECT "
    query += "DISTINCT " if @distinct

    query += @columns.empty? ? "*" : @columns.join(", ")
    query += " FROM #{@table}"

    @joins.each { |j| query += " #{j}" }

    unless @conditions.empty?
      query += " WHERE #{@conditions.join(' ')}"
    end

    unless @order_by.empty?
      query += " ORDER BY #{@order_by.join(', ')}"
    end

    query += " LIMIT #{@limit_val}" if @limit_val
    query += " OFFSET #{@offset_val}" if @offset_val

    query
  end
end

# Usage — reads like a DSL (Domain-Specific Language)
query = QueryBuilder.new
  .select("u.name", "u.email", "o.total")
  .from("users u")
  .left_join("orders o ON u.id = o.user_id")
  .where("u.active = true")
  .and_where("o.total > 100")
  .order_by_desc("o.total")
  .limit(10)
  .offset(20)
  .build

puts query
# SELECT u.name, u.email, o.total FROM users u LEFT JOIN orders o ON u.id = o.user_id
# WHERE u.active = true AND o.total > 100 ORDER BY o.total DESC LIMIT 10 OFFSET 20
```

---

### Builder vs Constructor with Many Parameters

| Approach | Pros | Cons |
|----------|------|------|
| **Many positional params** | Simple, one line | Unreadable, error-prone, all params required |
| **Keyword arguments** | Clear names, optional with defaults | No validation at build time, mutable |
| **Setter methods** | Flexible, clear names | Object in invalid state between setters |
| **Builder** | Readable, validates, immutable product | More code, extra class |

**When Builder wins (even in Ruby with kwargs):**
```ruby
# Keyword args approach — OK for simple cases
Window.new(width: 800, height: 600, resizable: true, title: "Main")

# Builder approach — better for complex construction with validation & multi-step logic
window = WindowBuilder.new
  .set_width(800)
  .set_height(600)
  .set_resizable(true)
  .set_title("Main")
  .add_menu_bar(MenuBar.new)       # multi-step
  .add_toolbar(Toolbar.new)        # multi-step
  .set_theme(DarkTheme.new)        # complex dependency
  .build                           # validates everything together
```

---

### Builder for Immutable Objects (Frozen)

A common pattern: the builder is the ONLY way to create the object, and the resulting object is frozen (immutable).

```ruby
class ServerConfig
  attr_reader :host, :port, :max_connections, :timeout,
              :use_tls, :cert_path, :key_path, :allowed_origins

  # Private constructor — only Builder can create via .send(:new, ...)
  private_class_method :new

  def initialize(host:, port:, max_connections:, timeout:, use_tls:,
                 cert_path:, key_path:, allowed_origins:)
    @host = host
    @port = port
    @max_connections = max_connections
    @timeout = timeout
    @use_tls = use_tls
    @cert_path = cert_path
    @key_path = key_path
    @allowed_origins = allowed_origins.freeze
    freeze  # make this object immutable!
  end

  # Builder is the only way to create ServerConfig
  class Builder
    def initialize
      @host = "0.0.0.0"
      @port = 8080
      @max_connections = 100
      @timeout = 30_000
      @use_tls = false
      @cert_path = nil
      @key_path = nil
      @allowed_origins = []
    end

    def host(h)            = tap { @host = h }
    def port(p)            = tap { @port = p }
    def max_connections(mc) = tap { @max_connections = mc }
    def timeout(t)         = tap { @timeout = t }

    def enable_tls(cert:, key:)
      @use_tls = true
      @cert_path = cert
      @key_path = key
      self
    end

    def add_allowed_origin(origin)
      @allowed_origins << origin
      self
    end

    def build
      # Validation
      raise "Host is required" if @host.nil? || @host.empty?
      raise "Invalid port" unless (1..65535).include?(@port)
      raise "TLS requires cert and key paths" if @use_tls && (@cert_path.nil? || @key_path.nil?)
      raise "Max connections must be positive" unless @max_connections > 0

      ServerConfig.send(:new,
        host: @host,
        port: @port,
        max_connections: @max_connections,
        timeout: @timeout,
        use_tls: @use_tls,
        cert_path: @cert_path,
        key_path: @key_path,
        allowed_origins: @allowed_origins
      )
    end
  end
end

# Usage
config = ServerConfig::Builder.new
  .host("api.example.com")
  .port(443)
  .max_connections(1000)
  .timeout(5000)
  .enable_tls(cert: "/etc/ssl/cert.pem", key: "/etc/ssl/key.pem")
  .add_allowed_origin("https://app.example.com")
  .add_allowed_origin("https://admin.example.com")
  .build

# config is now frozen — truly immutable, thread-safe
puts config.host          # "api.example.com"
puts config.port          # 443
# config.instance_variable_set(:@port, 9999)  # => FrozenError!
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
- HTTP request builders (Faraday, HTTParty configuration)
- SQL query builders (Arel, Sequel)
- Configuration objects (Rails initializers)
- Test data builders (FactoryBot)
- HTML/XML builders (Nokogiri::Builder)
- Email builders (ActionMailer)

---


## 4.5 Prototype Pattern

> The Prototype pattern creates new objects by cloning an existing object (the prototype) rather than creating from scratch. It's useful when object creation is expensive, or when you want to create objects whose type is determined at runtime.

---

### The Problem Prototype Solves

```ruby
# Problem 1: Expensive object creation
# Creating a game character involves loading textures, animations, AI models...
# Creating 100 enemies from scratch = 100 expensive loads
# Cloning a prototype = 1 load + 99 cheap copies

# Problem 2: Unknown concrete type at runtime
def duplicate_shape(shape)
  # How do you create a copy? You don't know if it's Circle, Rectangle, etc.
  # copy = shape.class.new(???)  # Don't know the constructor args!

  # With Prototype pattern:
  copy = shape.clone  # Works regardless of actual type!
end
```

---

### Deep Copy vs Shallow Copy

Understanding the difference is critical for implementing Prototype correctly.

```ruby
class Document
  attr_accessor :title, :content, :tags, :metadata

  def initialize(title, content)
    @title = title
    @content = content
    @tags = []
    @metadata = {}
  end

  # Ruby's default .clone and .dup do SHALLOW copies
  # They copy the object but share references to mutable internal objects
end

original = Document.new("Report", "Hello World")
original.tags << "important"
original.metadata["author"] = "Alice"

# SHALLOW COPY (Ruby's default .dup)
shallow = original.dup
shallow.tags << "copy"
puts original.tags.inspect  # ["important", "copy"] — SHARED! Modified original!

# DEEP COPY — must implement manually
class Document
  def deep_clone
    copy = dup
    copy.title = @title.dup
    copy.content = @content.dup
    copy.tags = @tags.map(&:dup)
    copy.metadata = @metadata.transform_values(&:dup)
    copy
  end
end

deep = original.deep_clone
deep.tags << "deep_copy"
puts original.tags.inspect  # ["important", "copy"] — NOT affected!
```

```
SHALLOW COPY (Ruby's .dup / .clone):
┌──────────────┐          ┌─────────────────────┐
│ original     │          │ ["important"]       │
│ @tags ───────┼─────────→│                     │
└──────────────┘          └─────────────────────┘
                                    ↑
┌──────────────┐                    │
│ copy         │                    │
│ @tags ───────┼────────────────────┘  (SHARED! Modifying one affects both!)
└──────────────┘

DEEP COPY:
┌──────────────┐          ┌─────────────────────┐
│ original     │          │ ["important"]       │
│ @tags ───────┼─────────→│                     │
└──────────────┘          └─────────────────────┘

┌──────────────┐          ┌─────────────────────┐
│ copy         │          │ ["important"]       │  (INDEPENDENT copy)
│ @tags ───────┼─────────→│                     │
└──────────────┘          └─────────────────────┘
```

---

### Ruby's Clone vs Dup

Important distinction in Ruby:

| Aspect | `.dup` | `.clone` |
|--------|--------|----------|
| Copies instance variables | Yes (shallow) | Yes (shallow) |
| Copies frozen state | No (unfrozen copy) | Yes (frozen if original was frozen) |
| Copies singleton methods | No | Yes |
| Copies modules extended on instance | No | Yes |
| Calls `initialize_copy` | Yes | Yes |

```ruby
obj = "hello"
obj.freeze

duped = obj.dup
duped.frozen?   # false — dup does NOT preserve frozen state

cloned = obj.clone
cloned.frozen?  # true — clone DOES preserve frozen state
```

---

### Prototype Pattern with `initialize_copy`

Ruby's hook method `initialize_copy` is called by both `dup` and `clone`, making it the ideal place to implement deep copying.

```ruby
class Shape
  attr_accessor :x, :y, :color

  def initialize(x = 0, y = 0, color = "black")
    @x = x
    @y = y
    @color = color
  end

  def draw
    raise NotImplementedError
  end

  def area
    raise NotImplementedError
  end

  def move_to(new_x, new_y)
    @x = new_x
    @y = new_y
  end
end

class Circle < Shape
  attr_accessor :radius

  def initialize(x, y, radius, color)
    super(x, y, color)
    @radius = radius
  end

  def draw
    puts "Circle at (#{@x},#{@y}) r=#{@radius} color=#{@color}"
  end

  def area
    Math::PI * @radius ** 2
  end
end

class Rectangle < Shape
  attr_accessor :width, :height

  def initialize(x, y, width, height, color)
    super(x, y, color)
    @width = width
    @height = height
  end

  def draw
    puts "Rectangle at (#{@x},#{@y}) #{@width}x#{@height} color=#{@color}"
  end

  def area
    @width * @height
  end
end

class Triangle < Shape
  attr_accessor :base, :tri_height

  def initialize(x, y, base, tri_height, color)
    super(x, y, color)
    @base = base
    @tri_height = tri_height
  end

  def draw
    puts "Triangle at (#{@x},#{@y}) base=#{@base} height=#{@tri_height} color=#{@color}"
  end

  def area
    0.5 * @base * @tri_height
  end
end

# --- Using Prototype to duplicate unknown shapes ---
def duplicate_and_move(original, offset_x, offset_y)
  copy = original.dup  # don't need to know concrete type!
  copy.move_to(offset_x, offset_y)
  copy.draw
end

c = Circle.new(10, 20, 5.0, "red")
r = Rectangle.new(0, 0, 100, 50, "blue")

duplicate_and_move(c, 50, 50)   # clones the circle, moves copy
duplicate_and_move(r, 200, 100) # clones the rectangle, moves copy

# Polymorphic cloning
shapes = [
  Circle.new(0, 0, 10, "red"),
  Rectangle.new(0, 0, 20, 30, "green"),
  Triangle.new(0, 0, 15, 20, "blue")
]

# Clone all shapes — each clones correctly regardless of type!
copies = shapes.map(&:dup)
copies.each { |s| s.move_to(rand(100), rand(100)) }
copies.each(&:draw)
```

---

### Prototype Registry (Prototype Manager)

A registry that stores pre-configured prototypes by name, allowing clients to clone them by key.

```ruby
class ShapeRegistry
  def initialize
    @prototypes = {}
  end

  def register(name, prototype)
    @prototypes[name] = prototype
  end

  def create(name)
    prototype = @prototypes[name]
    raise ArgumentError, "Unknown prototype: #{name}" unless prototype
    prototype.dup
  end

  def has_prototype?(name)
    @prototypes.key?(name)
  end

  def available_prototypes
    @prototypes.keys
  end
end

# Setup registry with pre-configured prototypes
registry = ShapeRegistry.new
registry.register("small-red-circle", Circle.new(0, 0, 5, "red"))
registry.register("large-blue-circle", Circle.new(0, 0, 50, "blue"))
registry.register("standard-button", Rectangle.new(0, 0, 120, 40, "gray"))
registry.register("icon-frame", Rectangle.new(0, 0, 32, 32, "transparent"))

# Create shapes from registry — no need to know construction details
shape1 = registry.create("small-red-circle")
shape1.move_to(100, 200)
shape1.draw

shape2 = registry.create("standard-button")
shape2.move_to(50, 300)
shape2.draw

# Create many copies efficiently
100.times do |i|
  enemy = registry.create("small-red-circle")
  enemy.move_to(i * 20, 0)
end
```

---

### Deep Clone with Complex Object Graphs

When objects contain references to other mutable objects, cloning requires careful deep copying using `initialize_copy`.

```ruby
class Address
  attr_accessor :street, :city, :country

  def initialize(street, city, country)
    @street = street
    @city = city
    @country = country
  end

  protected

  def initialize_copy(source)
    super
    @street = source.street.dup
    @city = source.city.dup
    @country = source.country.dup
  end
end

class Department
  attr_accessor :name, :floor

  def initialize(name, floor)
    @name = name
    @floor = floor
  end
end

class Employee
  attr_accessor :name, :id, :salary, :address, :department

  def initialize(name, id, salary, address, department)
    @name = name
    @id = id
    @salary = salary
    @address = address       # owned, must deep copy
    @department = department # shared, shallow copy is OK
  end

  # Clone with new ID
  def clone_with_new_id(new_id)
    copy = dup
    copy.id = new_id
    copy
  end

  def print_info
    puts "#{@name} (ID: #{@id}) $#{@salary}"
    puts "  Address: #{@address.street}, #{@address.city}"
    puts "  Department: #{@department.name} (Floor #{@department.floor})"
  end

  protected

  # Deep copy hook — called by .dup and .clone
  def initialize_copy(source)
    super
    # Deep copy owned objects
    @address = source.address.dup
    # Shared objects stay shared (same department reference)
    # @department is intentionally NOT duped
  end
end

# Usage
dept = Department.new("Engineering", 3)
emp1 = Employee.new(
  "Alice", 101, 95_000,
  Address.new("123 Main St", "Seattle", "USA"),
  dept
)

# Clone and customize
emp2 = emp1.clone_with_new_id(102)
emp2.name = "Bob"
emp2.salary = 90_000
emp2.address = Address.new("456 Oak Ave", "Portland", "USA")

emp1.print_info
emp2.print_info
# emp1 and emp2 share the same Department but have independent Addresses

# Verify deep copy worked
emp3 = emp1.dup
emp3.address.street = "789 New St"
puts emp1.address.street  # "123 Main St" — NOT affected (deep copy worked!)
```

---

### Prototype with Marshal (Quick Deep Copy)

Ruby's `Marshal` module can serialize and deserialize objects, providing a quick (but not always appropriate) deep copy mechanism.

```ruby
class GameCharacter
  attr_accessor :name, :health, :attack, :defense, :inventory, :skills

  def initialize(name, health, attack, defense)
    @name = name
    @health = health
    @attack = attack
    @defense = defense
    @inventory = []
    @skills = {}
  end

  # Deep clone using Marshal — copies EVERYTHING deeply
  def deep_clone
    Marshal.load(Marshal.dump(self))
  end

  def add_item(item)
    @inventory << item
  end

  def add_skill(skill, level)
    @skills[skill] = level
  end

  def describe
    puts "#{@name} [HP:#{@health} ATK:#{@attack} DEF:#{@defense}]"
    puts "  Inventory: #{@inventory.join(', ')}"
    puts "  Skills: #{@skills.map { |s, l| "#{s}(#{l})" }.join(' ')}"
  end
end

# Create a template character (prototype)
warrior_template = GameCharacter.new("Warrior", 100, 15, 10)
warrior_template.add_item("Sword")
warrior_template.add_item("Shield")
warrior_template.add_skill("Slash", 1)
warrior_template.add_skill("Block", 1)

# Clone and customize for each instance
warrior1 = warrior_template.deep_clone
warrior1.name = "Aragorn"
warrior1.add_skill("Leadership", 3)

warrior2 = warrior_template.deep_clone
warrior2.name = "Boromir"
warrior2.add_item("Horn of Gondor")

warrior1.describe
warrior2.describe
# Both have base warrior stats/items, but with individual customizations

# Verify independence
puts warrior_template.inventory.inspect  # ["Sword", "Shield"] — unchanged!
```

**Caveats of Marshal-based deep copy:**
- Cannot marshal Procs, lambdas, IO objects, or singleton methods
- Slower than manual `initialize_copy` for simple objects
- Doesn't call `initialize` or `initialize_copy`
- Good for data-heavy objects without closures or IO

---

### When to Use Prototype Pattern

| Scenario | Why Prototype Helps |
|----------|---------------------|
| Object creation is expensive | Clone is cheaper than creating from scratch |
| Need to create objects whose type is unknown at runtime | `.dup` / `.clone` handles polymorphic copying |
| Want to avoid parallel class hierarchies of factories | One `clone` method replaces factory classes |
| Need many similar objects with slight variations | Clone template, modify differences |
| System should be independent of how products are created | Clone decouples creation from concrete types |
| Want to add/remove products at runtime | Registry of prototypes can be modified dynamically |

**Real-world examples:**
- Game development: spawning enemies from templates
- Document editors: copy-paste operations
- GUI: duplicating widgets/components
- Database: creating records from templates (FactoryBot uses this concept)
- Configuration: deriving configs from base templates
- Rails: `ActiveRecord#dup` for creating similar records

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

```ruby
# BAD: Creating and destroying expensive objects repeatedly
def handle_request(query)
  # Each request creates a new connection — EXPENSIVE!
  conn = DatabaseConnection.new("host=localhost")  # ~50ms
  conn.connect                                      # ~200ms
  result = conn.execute(query)
  conn.disconnect                                   # ~50ms
  # Total overhead: ~300ms per request just for connection management!
end

# With 1000 requests/second, that's 300 seconds of wasted time per second!
```

**Objects that are expensive to create:**
- Database connections (TCP handshake, authentication, SSL)
- Thread objects (OS kernel call, stack allocation)
- Network sockets
- Large memory buffers
- External API client sessions
- File handles

---

### Basic Object Pool Implementation

```ruby
require 'thread'

class ObjectPool
  def initialize(max_size:, &factory)
    @max_size = max_size
    @factory = factory
    @available = Queue.new
    @mutex = Mutex.new
    @current_size = 0
  end

  # Acquire an object from the pool
  def acquire
    # Try to get from available pool (non-blocking)
    begin
      return @available.pop(true)  # non_block = true
    rescue ThreadError
      # Queue is empty, fall through
    end

    # If pool not at max capacity, create a new object
    @mutex.synchronize do
      if @current_size < @max_size
        @current_size += 1
        return @factory.call
      end
    end

    # Pool exhausted — block until an object is returned
    @available.pop  # blocking
  end

  # Acquire with timeout — returns nil if timeout expires
  def acquire_with_timeout(timeout_seconds)
    begin
      return @available.pop(true)
    rescue ThreadError
      # empty
    end

    @mutex.synchronize do
      if @current_size < @max_size
        @current_size += 1
        return @factory.call
      end
    end

    # Wait with timeout
    deadline = Time.now + timeout_seconds
    loop do
      remaining = deadline - Time.now
      return nil if remaining <= 0

      begin
        return @available.pop(true)
      rescue ThreadError
        sleep(0.01)
      end
    end
  end

  # Return an object to the pool
  def release(obj)
    return if obj.nil?
    yield obj if block_given?  # optional reset block
    @available.push(obj)
  end

  # Pool statistics
  def available_count
    @available.size
  end

  def total_size
    @current_size
  end

  def max_size
    @max_size
  end
end
```

---

### Connection Pool Example

```ruby
require 'thread'

class DatabaseConnection
  attr_reader :query_count

  def initialize(connection_string)
    @connection_string = connection_string
    @connected = false
    @query_count = 0
    puts "  [DB] Creating connection (expensive!)..."
    sleep(0.1)  # Simulate expensive creation
  end

  def connect
    @connected = true
    puts "  [DB] Connected"
  end

  def disconnect
    @connected = false
    puts "  [DB] Disconnected"
  end

  def execute(query)
    raise "Not connected!" unless @connected
    @query_count += 1
    puts "  [DB] Executing: #{query} (query ##{@query_count})"
    "result"
  end

  def reset
    @query_count = 0
    # Don't disconnect — that's the whole point of pooling!
  end

  def connected? = @connected
  def healthy?   = @connected
end

# --- Connection Pool with RAII-style block usage ---
class ConnectionPool
  def initialize(connection_string, max_connections:)
    @pool = ObjectPool.new(max_size: max_connections) do
      conn = DatabaseConnection.new(connection_string)
      conn.connect
      conn
    end
  end

  # RAII-style: automatically returns connection when block completes
  def with_connection
    conn = @pool.acquire
    begin
      yield conn
    ensure
      @pool.release(conn) { |c| c.reset }
    end
  end

  # Manual acquire/release (less safe, prefer with_connection)
  def acquire
    @pool.acquire
  end

  def release(conn)
    @pool.release(conn) { |c| c.reset }
  end

  def available
    @pool.available_count
  end
end

# Usage — connections are automatically returned to pool
pool = ConnectionPool.new("host=localhost dbname=myapp", max_connections: 5)

puts "=== Handling requests ==="

pool.with_connection do |conn|
  conn.execute("SELECT * FROM users")
end
# Connection automatically returned here!

pool.with_connection do |conn|
  conn.execute("INSERT INTO logs VALUES(...)")
end
# Same connection reused — no expensive creation!

pool.with_connection do |conn|
  conn.execute("UPDATE users SET active=true")
end

puts "Available connections: #{pool.available}"
```

---

### Thread Pool Concept

A thread pool is a classic application of the Object Pool pattern — threads are expensive to create, so we maintain a pool of worker threads.

```ruby
require 'thread'

class ThreadPool
  def initialize(num_threads)
    @tasks = Queue.new
    @shutdown = false
    @workers = num_threads.times.map do |i|
      Thread.new do
        loop do
          task = @tasks.pop
          break if task == :shutdown
          begin
            task.call
          rescue => e
            puts "Worker error: #{e.message}"
          end
        end
      end
    end
    puts "ThreadPool created with #{num_threads} threads"
  end

  # Submit a task (block) to be executed by a worker
  def submit(&block)
    raise "ThreadPool is shut down" if @shutdown
    @tasks.push(block)
  end

  # Submit and get a future-like result
  def submit_with_result(&block)
    raise "ThreadPool is shut down" if @shutdown
    result_queue = Queue.new
    @tasks.push(-> {
      begin
        result_queue.push([:ok, block.call])
      rescue => e
        result_queue.push([:error, e])
      end
    })
    # Return a lambda that blocks until result is ready
    -> {
      status, value = result_queue.pop
      raise value if status == :error
      value
    }
  end

  def pending_tasks
    @tasks.size
  end

  def worker_count
    @workers.size
  end

  def shutdown
    @shutdown = true
    @workers.size.times { @tasks.push(:shutdown) }
    @workers.each(&:join)
    puts "ThreadPool destroyed"
  end
end

# Usage
pool = ThreadPool.new(4)  # 4 worker threads

# Submit tasks
future1 = pool.submit_with_result do
  sleep(0.1)
  42
end

future2 = pool.submit_with_result do
  10 + 20
end

puts "Result 1: #{future1.call}"  # 42
puts "Result 2: #{future2.call}"  # 30

pool.shutdown
```

---

### Generic Object Pool with Health Checks

A production-ready pool that validates objects before handing them out and handles broken objects.

```ruby
require 'thread'

class RobustObjectPool
  PoolConfig = Struct.new(
    :min_size,
    :max_size,
    :max_idle_time,
    :max_usage_count,
    keyword_init: true
  ) do
    def initialize(**)
      super
      self.min_size ||= 2
      self.max_size ||= 10
      self.max_idle_time ||= 300  # seconds
      self.max_usage_count ||= 1000
    end
  end

  PooledObject = Struct.new(:object, :last_used, :usage_count)

  def initialize(config:, factory:, health_check: nil, reset_func: nil)
    @config = config
    @factory = factory
    @health_check = health_check
    @reset_func = reset_func
    @available = []
    @mutex = Mutex.new
    @condition = ConditionVariable.new
    @active_count = 0
    @shutdown = false

    # Pre-populate with minimum objects
    @config.min_size.times do
      obj = PooledObject.new(@factory.call, Time.now, 0)
      @available << obj
    end
  end

  def acquire
    @mutex.synchronize do
      loop do
        # Try to find a healthy object
        while (po = @available.shift)
          # Validate object health
          if @health_check && !@health_check.call(po.object)
            puts "[Pool] Discarding unhealthy object"
            next
          end

          # Check if object has been used too many times
          if po.usage_count >= @config.max_usage_count
            puts "[Pool] Discarding overused object"
            next
          end

          # Check idle time
          if (Time.now - po.last_used) > @config.max_idle_time
            puts "[Pool] Discarding idle object"
            next
          end

          @active_count += 1
          po.usage_count += 1
          return po.object
        end

        # No valid objects available — create new if under max
        total = @active_count + @available.size
        if total < @config.max_size
          @active_count += 1
          return @factory.call
        end

        # Wait for return
        raise "Pool is shutting down" if @shutdown
        @condition.wait(@mutex)
      end
    end
  end

  def release(obj)
    return if obj.nil?

    @mutex.synchronize do
      @active_count -= 1

      # Reset object state
      @reset_func&.call(obj)

      # Return to pool
      po = PooledObject.new(obj, Time.now, 0)
      @available << po

      @condition.signal
    end
  end

  # RAII-style block usage
  def with_object
    obj = acquire
    begin
      yield obj
    ensure
      release(obj)
    end
  end

  def close
    @mutex.synchronize do
      @shutdown = true
      @available.clear
      @condition.broadcast
    end
  end

  # Statistics
  def active_count
    @mutex.synchronize { @active_count }
  end

  def idle_count
    @mutex.synchronize { @available.size }
  end

  def total_count
    @mutex.synchronize { @active_count + @available.size }
  end
end

# Usage
pool = RobustObjectPool.new(
  config: RobustObjectPool::PoolConfig.new(min_size: 2, max_size: 5),
  factory: -> {
    conn = DatabaseConnection.new("host=localhost")
    conn.connect
    conn
  },
  health_check: ->(conn) { conn.healthy? },
  reset_func: ->(conn) { conn.reset }
)

# Use with block (recommended — automatic release)
pool.with_object do |conn|
  conn.execute("SELECT * FROM users")
end

puts "Active: #{pool.active_count}, Idle: #{pool.idle_count}"
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
- Object creation is cheap (simple Ruby objects)
- Objects hold state that's hard to reset cleanly
- Pool management overhead exceeds creation cost
- Objects are rarely reused (long-lived, unique state)

**Real-world examples:**
- Database connection pools (ActiveRecord's connection pool, Sequel)
- Thread pools (Concurrent Ruby, Sidekiq)
- HTTP connection pools (Net::HTTP persistent, Faraday)
- Puma web server (thread pool for request handling)
- Redis connection pools (connection_pool gem)

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
| **Singleton** | Exactly one instance | `include Singleton` or private `.new` | Shared resources, global state |
| **Factory Method** | Defer creation to subclasses | Overridable creation method | Unknown types, extensibility |
| **Abstract Factory** | Create families of objects | Factory interface with multiple create methods | Consistent product families |
| **Builder** | Step-by-step construction | Separate builder class with fluent API | Complex objects, many parameters |
| **Prototype** | Clone existing objects | `.dup` / `.clone` with `initialize_copy` | Expensive creation, unknown types |
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
1. Start with **Simple Factory** (class method with case/when)
2. When you need extensibility → **Factory Method** (overridable method)
3. When you need product families → **Abstract Factory**
4. When construction is complex → **Builder**
5. When creation is expensive → **Prototype** or **Object Pool**
6. When you need exactly one → **Singleton** (use sparingly!)

---

## Ruby-Specific Patterns & Idioms

Ruby's dynamic nature provides alternatives to some classical patterns:

| Classical Pattern | Ruby Idiom |
|-------------------|------------|
| Singleton | `include Singleton` module |
| Factory Method | Blocks/Procs as factories, `Class.new` |
| Abstract Factory | Modules as mixins, duck typing |
| Builder | Block-based DSLs (`Foo.new { \|f\| f.bar = 1 }`) |
| Prototype | `.dup` / `.clone` with `initialize_copy` |
| Object Pool | `connection_pool` gem, `Queue`-based pools |

**Ruby-specific factory idiom using blocks:**
```ruby
# Ruby's blocks make lightweight factories natural
class ServiceContainer
  def initialize
    @factories = {}
  end

  def register(name, &factory)
    @factories[name] = factory
  end

  def resolve(name)
    factory = @factories[name]
    raise "Unknown service: #{name}" unless factory
    factory.call
  end
end

container = ServiceContainer.new
container.register(:logger) { Logger.instance }
container.register(:db) { PgConnection.new(ENV["DATABASE_URL"]) }
container.register(:cache) { RedisCache.new(ENV["REDIS_URL"]) }

db = container.resolve(:db)
```

**Ruby-specific builder idiom using block initialization:**
```ruby
class Config
  attr_accessor :host, :port, :timeout, :retries

  def initialize
    @host = "localhost"
    @port = 8080
    @timeout = 30
    @retries = 3
    yield self if block_given?
    freeze  # make immutable after configuration
  end
end

# Clean, idiomatic Ruby — no separate builder class needed for simple cases
config = Config.new do |c|
  c.host = "api.example.com"
  c.port = 443
  c.timeout = 5
end
```

---

## Interview Tips for Module 4

1. **Singleton in Ruby** — Know `include Singleton` (standard library), manual implementation with `private_class_method :new`, and thread safety with `Mutex`. Explain that Ruby's `Singleton` module handles thread safety, clone/dup prevention, and marshaling.

2. **Singleton criticism** — Be ready to explain why Singleton is controversial: hidden dependencies, hard to test, global state. Know the alternative (dependency injection). Mention that in Ruby/Rails, you can use `ActiveSupport::CurrentAttributes` or simply pass dependencies through constructors.

3. **Factory Method vs Abstract Factory** — Factory Method creates ONE product (single overridable method). Abstract Factory creates a FAMILY of products (multiple creation methods). In Ruby, duck typing means you don't always need formal abstract classes.

4. **Simple Factory vs Factory Method** — Simple Factory is NOT a GoF pattern. It's a class method with case/when. Factory Method uses inheritance and polymorphism for extensibility (OCP compliant). In Ruby, registered factories with blocks are very common.

5. **Builder vs keyword arguments** — Ruby's keyword args reduce the need for Builder in simple cases. Builder still wins when: construction is multi-step, validation happens at build time, you want immutable (frozen) products, or you need a Director pattern.

6. **Builder with Director** — Director encapsulates construction recipes. Same director + different builders = different representations. Same builder + different directors = different construction orders.

7. **Prototype: clone vs dup** — Always explain the difference. `.clone` preserves frozen state and singleton methods; `.dup` does not. Both do shallow copies by default. Override `initialize_copy` for deep copying. `Marshal.load(Marshal.dump(obj))` is a quick deep copy hack.

8. **Object Pool lifecycle** — Explain acquire/release/reset cycle. Know block-based usage (`pool.with_object { |obj| ... }`) for automatic release. Discuss health checks, max idle time, and pool sizing. Mention real examples (ActiveRecord connection pool, connection_pool gem).

9. **When NOT to use each pattern:**
   - Singleton: when testability matters, when you might need multiple instances later
   - Factory: when creation is simple and types are known
   - Builder: when keyword arguments suffice (simple objects)
   - Prototype: when objects are cheap to create
   - Pool: when creation is cheap or objects can't be cleanly reset

10. **Combining patterns** — Abstract Factory often uses Factory Methods internally. Object Pool is often a Singleton. Builder can use Factory for creating parts. Show you understand how patterns compose.

11. **Open/Closed Principle** — Factory Method and Abstract Factory are the primary patterns for achieving OCP in object creation. In Ruby, self-registering factories with blocks are particularly elegant for OCP compliance.

12. **Ruby's dynamic dispatch** — In Ruby, duck typing means you often don't need formal abstract classes or interfaces. A factory just needs to return objects that respond to the expected methods. This makes patterns lighter-weight than in statically-typed languages.
