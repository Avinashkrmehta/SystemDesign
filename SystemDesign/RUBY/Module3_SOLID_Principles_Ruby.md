# Module 3: SOLID Principles (Ruby)

> SOLID is an acronym for five design principles that make software designs more understandable, flexible, and maintainable. These principles were introduced by Robert C. Martin (Uncle Bob) and form the foundation of good object-oriented design. Each principle addresses a specific aspect of class design and dependency management, and together they guide developers toward code that is easy to extend, test, and refactor.

---

## 3.1 Single Responsibility Principle (SRP)

> "A class should have one, and only one, reason to change." — Robert C. Martin

The Single Responsibility Principle states that every class should have exactly one responsibility — one axis of change. If a class has more than one reason to change, it has more than one responsibility, and those responsibilities are coupled. A change to one responsibility may impair or inhibit the class's ability to meet the others.

---

### One Class, One Reason to Change

**What is a "reason to change"?**

A "reason to change" corresponds to a **stakeholder** or **actor** who might request a change. If two different actors can request changes to the same class, that class has two responsibilities.

```ruby
# BAD: This class has THREE reasons to change:
# 1. Business logic changes (how salary is calculated)
# 2. Persistence changes (how employee is saved to DB)
# 3. Presentation changes (how employee is displayed/reported)

class Employee
  attr_reader :name, :department, :base_salary, :bonus

  def initialize(name, department, base_salary, bonus)
    @name = name
    @department = department
    @base_salary = base_salary
    @bonus = bonus
  end

  # Responsibility 1: Business logic (HR department cares about this)
  def calculate_pay
    base_salary + bonus
  end

  def calculate_tax
    pay = calculate_pay
    if pay > 100_000
      pay * 0.30
    elsif pay > 50_000
      pay * 0.20
    else
      pay * 0.10
    end
  end

  # Responsibility 2: Persistence (DBA/Infrastructure team cares about this)
  def save_to_database
    # SQL query to save employee
    sql = "INSERT INTO employees (name, dept, salary, bonus) VALUES " \
          "('#{name}', '#{department}', #{base_salary}, #{bonus})"
    # execute(sql)
  end

  def load_from_database(id)
    # SQL query to load employee
    # ... parse results and set fields ...
  end

  # Responsibility 3: Presentation (UI/Report team cares about this)
  def generate_report
    <<~REPORT
      Employee Report
      Name: #{name}
      Department: #{department}
      Salary: $#{calculate_pay}
      Tax: $#{calculate_tax}
    REPORT
  end

  def print_to_console
    puts generate_report
  end

  def to_json
    { name: name, salary: calculate_pay }.to_json
  end
end
```

**Why is this problematic?**
- If the report format changes, you modify the same class that handles business logic
- If the database schema changes, you risk breaking salary calculations
- If tax rules change, you might accidentally affect the persistence layer
- Testing one responsibility requires setting up infrastructure for all three
- Multiple developers working on different features will create merge conflicts

---

**GOOD: Each class has exactly one responsibility:**

```ruby
# Responsibility 1: Business logic — salary and tax calculations
class Employee
  attr_reader :name, :department, :base_salary, :bonus

  def initialize(name, department, base_salary, bonus)
    @name = name
    @department = department
    @base_salary = base_salary
    @bonus = bonus
  end

  def calculate_pay
    base_salary + bonus
  end

  def calculate_tax
    pay = calculate_pay
    if pay > 100_000
      pay * 0.30
    elsif pay > 50_000
      pay * 0.20
    else
      pay * 0.10
    end
  end
end

# Responsibility 2: Persistence — saving and loading from database
class EmployeeRepository
  def save(employee)
    # Parameterized query for safety
    # stmt = db.prepare("INSERT INTO employees (name, dept, salary, bonus) VALUES (?, ?, ?, ?)")
    # stmt.execute(employee.name, employee.department, employee.base_salary, employee.bonus)
    puts "Saved #{employee.name} to database"
  end

  def find_by_id(id)
    # Query database and construct Employee
    Employee.new("loaded", "dept", 0, 0)
  end

  def find_by_department(dept)
    # ...
    []
  end
end

# Responsibility 3: Presentation — formatting and display
class EmployeeReportGenerator
  def generate_text_report(employee)
    <<~REPORT
      Employee Report
      Name: #{employee.name}
      Department: #{employee.department}
      Salary: $#{employee.calculate_pay}
      Tax: $#{employee.calculate_tax}
    REPORT
  end

  def generate_json(employee)
    {
      name: employee.name,
      department: employee.department,
      salary: employee.calculate_pay,
      tax: employee.calculate_tax
    }.to_json
  end

  def generate_csv(employee)
    "#{employee.name},#{employee.department},#{employee.calculate_pay},#{employee.calculate_tax}"
  end
end

# Usage — each class is independent and focused
emp = Employee.new("Alice", "Engineering", 85_000, 15_000)

repo = EmployeeRepository.new
repo.save(emp)

reporter = EmployeeReportGenerator.new
puts reporter.generate_text_report(emp)
puts reporter.generate_json(emp)
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

```ruby
# Identifying responsibilities through cohesion analysis

class UserManager
  def initialize(username, password_hash, salt, display_name, email, avatar_url, bio,
                 email_notifications, push_notifications, subscribed_topics)
    # Group A: Authentication data
    @username = username
    @password_hash = password_hash
    @salt = salt
    @failed_attempts = 0
    @lockout_until = nil

    # Group B: Profile data
    @display_name = display_name
    @email = email
    @avatar_url = avatar_url
    @bio = bio

    # Group C: Notification preferences
    @email_notifications = email_notifications
    @push_notifications = push_notifications
    @subscribed_topics = subscribed_topics
  end

  # Methods using Group A (Authentication)
  def authenticate(password)
    # uses @password_hash, @salt, @failed_attempts
  end

  def reset_password(new_password)
    # uses @password_hash, @salt
  end

  def locked_out?
    # uses @failed_attempts, @lockout_until
  end

  # Methods using Group B (Profile)
  def update_profile(name, bio)
    # uses @display_name, @bio
  end

  def change_email(new_email)
    # uses @email
  end

  def set_avatar(url)
    # uses @avatar_url
  end

  # Methods using Group C (Notifications)
  def subscribe(topic)
    # uses @subscribed_topics
  end

  def set_email_notifications(enabled)
    # uses @email_notifications
  end

  def subscriptions
    # uses @subscribed_topics
  end
end

# REFACTORED: Three cohesive classes

class UserAuthentication
  def initialize(username, password_hash, salt)
    @username = username
    @password_hash = password_hash
    @salt = salt
    @failed_attempts = 0
    @lockout_until = nil
  end

  def authenticate(password)
    # verify password against hash
  end

  def reset_password(new_password)
    # update hash and salt
  end

  def locked_out?
    @failed_attempts >= 5 && @lockout_until && Time.now < @lockout_until
  end

  def record_failed_attempt
    @failed_attempts += 1
    @lockout_until = Time.now + 900 if @failed_attempts >= 5  # 15 min lockout
  end

  def unlock
    @failed_attempts = 0
    @lockout_until = nil
  end
end

class UserProfile
  attr_reader :user_id, :display_name, :email, :avatar_url, :bio

  def initialize(user_id, display_name, email)
    @user_id = user_id
    @display_name = display_name
    @email = email
    @avatar_url = nil
    @bio = nil
  end

  def update_display_name(name)
    @display_name = name
  end

  def change_email(new_email)
    @email = new_email
  end

  def set_avatar(url)
    @avatar_url = url
  end

  def update_bio(new_bio)
    @bio = new_bio
  end
end

class NotificationPreferences
  attr_reader :user_id

  def initialize(user_id)
    @user_id = user_id
    @email_enabled = true
    @push_enabled = true
    @subscribed_topics = []
  end

  def enable_email(enabled)
    @email_enabled = enabled
  end

  def enable_push(enabled)
    @push_enabled = enabled
  end

  def subscribe(topic)
    @subscribed_topics << topic unless @subscribed_topics.include?(topic)
  end

  def unsubscribe(topic)
    @subscribed_topics.delete(topic)
  end

  def should_notify_via_email?
    @email_enabled
  end

  def should_notify_via_push?
    @push_enabled
  end
end
```

---

### Refactoring God Classes

A **God Class** (or Blob) is a class that knows too much or does too much. It violates SRP by accumulating responsibilities over time. Refactoring a God Class involves extracting cohesive groups of data and behavior into separate classes.

**Step-by-step refactoring process:**

```ruby
# BEFORE: God Class — an online store "does everything" class
class OnlineStore
  def initialize
    @products = []
    @inventory = {}  # product_id => quantity
    @users = []
    @user_orders = {}  # user_id => [order_ids]
    @orders = []
    @next_order_id = 1
    @stripe_api_key = nil
    @paypal_client_id = nil
    @smtp_server = nil
    @from_address = nil
  end

  # Product management
  def add_product(product) = @products << product
  def update_inventory(product_id, qty) = @inventory[product_id] = qty
  def in_stock?(product_id) = (@inventory[product_id] || 0) > 0

  # User management
  def register_user(user) = @users << user
  def find_user(user_id) = @users.find { |u| u.id == user_id }
  def authenticate_user(email, password) = # ...

  # Order processing
  def create_order(user_id, items) = # ...
  def cancel_order(order_id) = # ...
  def calculate_order_total(order_id) = # ...

  # Payment processing
  def charge_stripe(order_id, card_token) = # ...
  def charge_paypal(order_id, paypal_email) = # ...
  def process_refund(order_id) = # ...

  # Email notifications
  def send_order_confirmation(order_id) = # ...
  def send_shipping_notification(order_id, tracking_num) = # ...
  def send_password_reset(user_id) = # ...

  # Reporting
  def total_revenue = # ...
  def top_selling_products(n) = # ...
  def sales_by_category = # ...
end
```

**AFTER: Refactored into focused classes:**

```ruby
# Each class has ONE responsibility and ONE reason to change

class ProductCatalog
  def initialize
    @products = []
  end

  def add_product(product)
    @products << product
  end

  def remove_product(product_id)
    @products.reject! { |p| p.id == product_id }
  end

  def find_product(product_id)
    @products.find { |p| p.id == product_id }
  end

  def search(query)
    @products.select { |p| p.name.downcase.include?(query.downcase) }
  end
end

class InventoryManager
  def initialize
    @stock = {}  # product_id => quantity
  end

  def set_stock(product_id, quantity)
    @stock[product_id] = quantity
  end

  def available?(product_id, quantity = 1)
    (@stock[product_id] || 0) >= quantity
  end

  def reserve(product_id, quantity)
    raise "Insufficient stock" unless available?(product_id, quantity)
    @stock[product_id] -= quantity
  end

  def release(product_id, quantity)
    @stock[product_id] = (@stock[product_id] || 0) + quantity
  end

  def get_stock(product_id)
    @stock[product_id] || 0
  end
end

class UserService
  def initialize
    @users = []
  end

  def register(user)
    @users << user
  end

  def find_by_id(user_id)
    @users.find { |u| u.id == user_id }
  end

  def find_by_email(email)
    @users.find { |u| u.email == email }
  end

  def authenticate(email, password)
    user = find_by_email(email)
    user&.authenticate(password)
  end
end

class OrderService
  def initialize(inventory)
    @orders = []
    @next_order_id = 1
    @inventory = inventory  # dependency injection
  end

  def create_order(user_id, items)
    items.each { |item| @inventory.reserve(item.product_id, item.quantity) }
    order = Order.new(@next_order_id, user_id, items)
    @orders << order
    @next_order_id += 1
    order
  end

  def cancel_order(order_id)
    order = find_order(order_id)
    order.items.each { |item| @inventory.release(item.product_id, item.quantity) }
    order.cancel!
  end

  def find_order(order_id)
    @orders.find { |o| o.id == order_id }
  end

  def calculate_total(order_id)
    order = find_order(order_id)
    order.items.sum { |item| item.price * item.quantity }
  end

  def orders_by_user(user_id)
    @orders.select { |o| o.user_id == user_id }
  end
end

class PaymentService
  def initialize(stripe_api_key, paypal_client_id)
    @stripe_api_key = stripe_api_key
    @paypal_client_id = paypal_client_id
  end

  def charge_card(amount, card_token)
    puts "Stripe: charging $#{amount}"
    true
  end

  def charge_paypal(amount, paypal_email)
    puts "PayPal: charging $#{amount} to #{paypal_email}"
    true
  end

  def refund(transaction_id, amount)
    puts "Refunding $#{amount} for transaction #{transaction_id}"
    true
  end
end

class NotificationService
  def initialize(smtp_server, from_address)
    @smtp_server = smtp_server
    @from_address = from_address
  end

  def send_order_confirmation(order, email)
    puts "Sending order confirmation for ##{order.id} to #{email}"
  end

  def send_shipping_update(order, tracking_num)
    puts "Sending shipping update: #{tracking_num}"
  end

  def send_password_reset(email, reset_token)
    puts "Sending password reset to #{email}"
  end
end

class SalesReportService
  def initialize(orders, catalog)
    @orders = orders
    @catalog = catalog
  end

  def total_revenue
    @orders.sum { |o| o.total }
  end

  def top_selling(n)
    # ... aggregate and sort by quantity sold
  end

  def revenue_by_category
    # ... group by category and sum
  end
end
```

**Benefits of the refactoring:**
- Each class can be tested independently
- Changes to payment processing don't risk breaking order logic
- Different teams can work on different classes without conflicts
- New notification channels (SMS, Push) only affect NotificationService
- Database changes only affect the relevant repository/service

---

### Ruby Examples with Before/After

**Example 1: Logger with too many responsibilities**

```ruby
# BEFORE: Logger handles formatting, filtering, AND output
class Logger
  def initialize(filename, min_level)
    @file = File.open(filename, 'a')
    @min_level = min_level  # 0=DEBUG, 1=INFO, 2=WARN, 3=ERROR
  end

  def log(level, message)
    return if level < @min_level  # filtering

    # formatting
    level_str = %w[DEBUG INFO WARN ERROR][level]
    timestamp = Time.now.strftime('%Y-%m-%d %H:%M:%S')
    formatted = "[#{timestamp}] [#{level_str}] #{message}"

    # output to file
    @file.puts(formatted)
    @file.flush

    # output to console (hardcoded!)
    $stderr.puts(formatted) if level >= 2
  end

  def close
    @file.close
  end
end
```

```ruby
# AFTER: Separated into focused components

# Responsibility: Format log messages
class LogFormatter
  def format(level, message)
    raise NotImplementedError
  end
end

class StandardFormatter < LogFormatter
  LEVELS = %w[DEBUG INFO WARN ERROR].freeze

  def format(level, message)
    timestamp = Time.now.strftime('%Y-%m-%d %H:%M:%S')
    "[#{timestamp}] [#{LEVELS[level]}] #{message}"
  end
end

class JSONFormatter < LogFormatter
  def format(level, message)
    { level: level, message: message, timestamp: Time.now.iso8601 }.to_json
  end
end

# Responsibility: Filter log messages by level
class LogFilter
  attr_accessor :min_level

  def initialize(min_level)
    @min_level = min_level
  end

  def should_log?(level)
    level >= @min_level
  end
end

# Responsibility: Write log messages to a destination
class LogOutput
  def write(formatted_message)
    raise NotImplementedError
  end
end

class FileOutput < LogOutput
  def initialize(filename)
    @file = File.open(filename, 'a')
  end

  def write(msg)
    @file.puts(msg)
    @file.flush
  end

  def close
    @file.close
  end
end

class ConsoleOutput < LogOutput
  def write(msg)
    $stderr.puts(msg)
  end
end

class NetworkOutput < LogOutput
  def initialize(server_url)
    @server_url = server_url
  end

  def write(msg)
    # Send to log aggregation service
    puts "[NETWORK -> #{@server_url}] #{msg}"
  end
end

# Orchestrator: Composes the above components
class Logger
  def initialize(formatter, min_level)
    @formatter = formatter
    @filter = LogFilter.new(min_level)
    @outputs = []
  end

  def add_output(output)
    @outputs << output
  end

  def log(level, message)
    return unless @filter.should_log?(level)

    formatted = @formatter.format(level, message)

    @outputs.each { |output| output.write(formatted) }
  end
end

# Usage — flexible composition
logger = Logger.new(StandardFormatter.new, 1)  # INFO and above
logger.add_output(FileOutput.new("app.log"))
logger.add_output(ConsoleOutput.new)
logger.add_output(NetworkOutput.new("https://logs.example.com"))

logger.log(1, "Application started")
logger.log(3, "Database connection failed")
```

**Example 2: Invoice class refactoring**

```ruby
# BEFORE: Invoice handles calculation, persistence, AND printing
class Invoice
  def initialize(items, tax_rate, customer_name, customer_email)
    @items = items
    @tax_rate = tax_rate
    @customer_name = customer_name
    @customer_email = customer_email
  end

  # Calculation (business logic)
  def subtotal
    @items.sum { |item| item.price * item.quantity }
  end

  def tax
    subtotal * @tax_rate
  end

  def total
    subtotal + tax
  end

  # Persistence (infrastructure concern)
  def save_to_file(filename)
    File.open(filename, 'w') do |f|
      f.puts "Customer: #{@customer_name}"
      @items.each { |item| f.puts "#{item.name},#{item.price},#{item.quantity}" }
      f.puts "Total: #{total}"
    end
  end

  # Printing (presentation concern)
  def print
    puts "=== INVOICE ==="
    puts "Customer: #{@customer_name}"
    @items.each do |item|
      puts "  #{item.name} x#{item.quantity} @ $#{item.price}"
    end
    puts "Subtotal: $#{subtotal}"
    puts "Tax: $#{tax}"
    puts "TOTAL: $#{total}"
  end

  # Email sending (notification concern)
  def email_to_customer
    # SMTP logic here...
  end
end
```

```ruby
# AFTER: Clean separation

class Invoice
  attr_reader :items, :tax_rate, :customer_id

  def initialize(customer_id, tax_rate)
    @customer_id = customer_id
    @tax_rate = tax_rate
    @items = []
  end

  def add_item(item)
    @items << item
  end

  def subtotal
    @items.sum { |item| item.price * item.quantity }
  end

  def tax
    subtotal * @tax_rate
  end

  def total
    subtotal + tax
  end
end

class InvoiceRepository
  def save(invoice)
    raise NotImplementedError
  end

  def load(invoice_id)
    raise NotImplementedError
  end
end

class FileInvoiceRepository < InvoiceRepository
  def initialize(directory)
    @directory = directory
  end

  def save(invoice)
    # write to file
  end

  def load(invoice_id)
    # read from file
  end
end

class InvoicePrinter
  def print(invoice)
    raise NotImplementedError
  end
end

class ConsoleInvoicePrinter < InvoicePrinter
  def print(invoice)
    puts "=== INVOICE ==="
    invoice.items.each do |item|
      puts "  #{item.name} x#{item.quantity} @ $#{item.price}"
    end
    puts "TOTAL: $#{invoice.total}"
  end
end

class PDFInvoicePrinter < InvoicePrinter
  def print(invoice)
    # Generate PDF...
  end
end
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

**Ruby-specific SRP patterns:**
- Use modules (mixins) to extract shared behavior, but be careful — a module included in a class still adds responsibility to that class
- Use service objects (plain Ruby classes with a single `call` method) for operations
- Use form objects to separate validation from models
- Use presenter/decorator objects to separate display logic from domain models

```ruby
# Ruby idiom: Service objects for single-responsibility operations
class CalculateTax
  def initialize(employee)
    @employee = employee
  end

  def call
    pay = @employee.calculate_pay
    if pay > 100_000
      pay * 0.30
    elsif pay > 50_000
      pay * 0.20
    else
      pay * 0.10
    end
  end
end

# Usage
tax = CalculateTax.new(employee).call
```

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

```ruby
# BAD: Every new shape requires modifying this method
class AreaCalculator
  def calculate_area(shapes)
    total = 0
    shapes.each do |shape|
      case shape[:type]
      when :circle
        total += Math::PI * shape[:radius]**2
      when :rectangle
        total += shape[:width] * shape[:height]
      when :triangle
        total += 0.5 * shape[:base] * shape[:height]
      # Adding a new shape (e.g., Pentagon) requires MODIFYING this method!
      # This violates OCP.
      end
    end
    total
  end
end
```

**Problems with the above:**
- Adding a new shape means modifying `calculate_area` — violates "closed for modification"
- The method grows unboundedly with each new shape
- Risk of breaking existing shape calculations when adding new ones
- Cannot add shapes from external libraries without modifying this code

**GOOD: Open for extension, closed for modification:**

```ruby
# Abstract base — the stable, closed part
class Shape
  def area
    raise NotImplementedError, "#{self.class} must implement #area"
  end

  def name
    raise NotImplementedError, "#{self.class} must implement #name"
  end
end

# Extensions — new shapes added WITHOUT modifying existing code
class Circle < Shape
  def initialize(radius)
    @radius = radius
  end

  def area
    Math::PI * @radius**2
  end

  def name
    "Circle"
  end
end

class Rectangle < Shape
  def initialize(width, height)
    @width = width
    @height = height
  end

  def area
    @width * @height
  end

  def name
    "Rectangle"
  end
end

class Triangle < Shape
  def initialize(base, height)
    @base = base
    @height = height
  end

  def area
    0.5 * @base * @height
  end

  def name
    "Triangle"
  end
end

# This class is CLOSED for modification — never needs to change
class AreaCalculator
  def total_area(shapes)
    shapes.sum(&:area)
  end

  def print_report(shapes)
    shapes.each do |shape|
      puts "#{shape.name}: area = #{shape.area}"
    end
    puts "Total area: #{total_area(shapes)}"
  end
end

# Adding a new shape — NO modification to AreaCalculator or any existing shape!
class Pentagon < Shape
  def initialize(side)
    @side = side
  end

  def area
    0.25 * Math.sqrt(5 * (5 + 2 * Math.sqrt(5))) * @side**2
  end

  def name
    "Pentagon"
  end
end

class Ellipse < Shape
  def initialize(a, b)
    @a = a  # semi-major axis
    @b = b  # semi-minor axis
  end

  def area
    Math::PI * @a * @b
  end

  def name
    "Ellipse"
  end
end

# Usage — AreaCalculator works with ALL shapes, past and future
shapes = [
  Circle.new(5.0),
  Rectangle.new(3.0, 4.0),
  Pentagon.new(2.0),
  Ellipse.new(3.0, 2.0)
]

calc = AreaCalculator.new
calc.print_report(shapes)  # works without any modification to AreaCalculator
```

---

### Using Inheritance and Polymorphism for OCP

The most common way to achieve OCP in Ruby is through **inheritance** and **duck typing**. The base class (or implicit interface) defines the contract (closed), and subclasses or new objects provide new implementations (open for extension).

**Example: Payment processing system**

```ruby
# Closed for modification — this interface is stable
class PaymentMethod
  def pay(amount)
    raise NotImplementedError
  end

  def refund(amount)
    raise NotImplementedError
  end

  def method_name
    raise NotImplementedError
  end

  def transaction_fee(amount)
    raise NotImplementedError
  end
end

# Existing implementations — already tested and deployed
class CreditCardPayment < PaymentMethod
  def initialize(card_number, expiry_date)
    @card_number = card_number
    @expiry_date = expiry_date
  end

  def pay(amount)
    fee = transaction_fee(amount)
    puts "Charging $#{amount + fee} to card ending in #{@card_number[-4..]}"
    true
  end

  def refund(amount)
    puts "Refunding $#{amount} to card"
    true
  end

  def method_name
    "Credit Card"
  end

  def transaction_fee(amount)
    amount * 0.029 + 0.30
  end
end

class PayPalPayment < PaymentMethod
  def initialize(email)
    @email = email
  end

  def pay(amount)
    puts "PayPal payment of $#{amount} from #{@email}"
    true
  end

  def refund(amount)
    puts "PayPal refund of $#{amount} to #{@email}"
    true
  end

  def method_name
    "PayPal"
  end

  def transaction_fee(amount)
    amount * 0.034 + 0.30
  end
end

# Payment processor — CLOSED for modification
class PaymentProcessor
  def process_payment(method, amount)
    puts "Processing #{method.method_name} payment..."
    fee = method.transaction_fee(amount)
    puts "Transaction fee: $#{fee}"

    success = method.pay(amount)
    if success
      puts "Payment successful!"
    else
      puts "Payment failed!"
    end
    success
  end
end

# EXTENSION: Adding cryptocurrency payment — NO modification to PaymentProcessor!
class CryptoPayment < PaymentMethod
  def initialize(wallet_address, currency)
    @wallet_address = wallet_address
    @currency = currency  # BTC, ETH, etc.
  end

  def pay(amount)
    puts "Sending #{amount} USD equivalent in #{@currency} to #{@wallet_address}"
    true
  end

  def refund(amount)
    puts "Crypto refund of $#{amount} in #{@currency}"
    true
  end

  def method_name
    "Crypto (#{@currency})"
  end

  def transaction_fee(amount)
    amount * 0.01
  end
end

# EXTENSION: Adding bank transfer — NO modification needed anywhere!
class BankTransferPayment < PaymentMethod
  def initialize(account_number, routing_number)
    @account_number = account_number
    @routing_number = routing_number
  end

  def pay(amount)
    puts "Bank transfer of $#{amount} from account #{@account_number}"
    true
  end

  def refund(amount)
    puts "Bank refund of $#{amount} to account #{@account_number}"
    true
  end

  def method_name
    "Bank Transfer"
  end

  def transaction_fee(_amount)
    0.25  # flat fee
  end
end

# Usage — PaymentProcessor handles ALL payment types without modification
processor = PaymentProcessor.new

card = CreditCardPayment.new("4111111111111111", "12/25")
processor.process_payment(card, 99.99)

crypto = CryptoPayment.new("0xABC123...", "ETH")
processor.process_payment(crypto, 50.00)

bank = BankTransferPayment.new("123456789", "021000021")
processor.process_payment(bank, 1000.00)
```

---

### Strategy Pattern as OCP Example

The **Strategy Pattern** is one of the most natural implementations of OCP. It allows you to define a family of algorithms, encapsulate each one, and make them interchangeable — all without modifying the context class.

```ruby
# Strategy interface — closed for modification
class CompressionStrategy
  def compress(data)
    raise NotImplementedError
  end

  def decompress(data)
    raise NotImplementedError
  end

  def name
    raise NotImplementedError
  end

  def compression_ratio
    raise NotImplementedError
  end
end

# Concrete strategies — each is an extension
class ZipCompression < CompressionStrategy
  def compress(data)
    puts "Compressing with ZIP algorithm..."
    data  # simplified
  end

  def decompress(data)
    puts "Decompressing ZIP..."
    data
  end

  def name
    "ZIP"
  end

  def compression_ratio
    0.6
  end
end

class GzipCompression < CompressionStrategy
  def compress(data)
    puts "Compressing with GZIP algorithm..."
    data
  end

  def decompress(data)
    puts "Decompressing GZIP..."
    data
  end

  def name
    "GZIP"
  end

  def compression_ratio
    0.55
  end
end

class LZ4Compression < CompressionStrategy
  def compress(data)
    puts "Compressing with LZ4 (fast)..."
    data
  end

  def decompress(data)
    puts "Decompressing LZ4..."
    data
  end

  def name
    "LZ4"
  end

  def compression_ratio
    0.75  # less compression, more speed
  end
end

# Context class — CLOSED for modification
class FileArchiver
  def initialize(strategy)
    @strategy = strategy
  end

  def strategy=(new_strategy)
    @strategy = new_strategy
  end

  def archive_file(filename)
    puts "Archiving #{filename} using #{@strategy.name}"
    puts "Expected compression ratio: #{@strategy.compression_ratio}"

    # Read file data (simplified)
    data = File.exist?(filename) ? File.read(filename) : ""

    # Compress using current strategy
    compressed = @strategy.compress(data)

    puts "Archive complete."
    compressed
  end

  def extract_file(archive_name)
    puts "Extracting #{archive_name} using #{@strategy.name}"
    data = ""  # archive contents
    decompressed = @strategy.decompress(data)
    puts "Extraction complete."
    decompressed
  end
end

# Adding a new compression algorithm — ZERO modification to FileArchiver
class BrotliCompression < CompressionStrategy
  def compress(data)
    puts "Compressing with Brotli (web-optimized)..."
    data
  end

  def decompress(data)
    puts "Decompressing Brotli..."
    data
  end

  def name
    "Brotli"
  end

  def compression_ratio
    0.50
  end
end

# Usage
archiver = FileArchiver.new(GzipCompression.new)
archiver.archive_file("document.pdf")

# Switch strategy at runtime — no modification to FileArchiver
archiver.strategy = BrotliCompression.new
archiver.archive_file("image.png")
```

---

### Template Method as OCP Example

The **Template Method Pattern** defines the skeleton of an algorithm in a base class, letting subclasses override specific steps without changing the algorithm's structure. The base class is closed for modification (the algorithm structure is fixed), but open for extension (individual steps can be customized).

```ruby
# Template Method — the algorithm skeleton is CLOSED for modification
class DataMiner
  # Template method — defines the algorithm structure
  def mine(path)
    raw_data = extract_data(path)
    data = parse_data(raw_data)
    analysis = analyze_data(data)
    report = generate_report(analysis)
    send_report(report)
  end

  protected

  # Steps that subclasses MUST implement (open for extension)
  def extract_data(path)
    raise NotImplementedError, "#{self.class} must implement #extract_data"
  end

  def parse_data(raw_data)
    raise NotImplementedError, "#{self.class} must implement #parse_data"
  end

  # Steps with default implementation that CAN be overridden (hook methods)
  def analyze_data(data)
    # Default: word frequency analysis
    data.each_with_object(Hash.new(0)) { |word, freq| freq[word] += 1 }
  end

  def generate_report(analysis)
    report = "Analysis Report:\n"
    analysis.each { |key, value| report += "  #{key}: #{value}\n" }
    report
  end

  def send_report(report)
    puts report  # default: print to console
  end
end

# Extension: CSV data mining — only implements what's different
class CSVDataMiner < DataMiner
  protected

  def extract_data(path)
    puts "Reading CSV file: #{path}"
    # Read CSV file contents
    "name,age,city\nAlice,30,NYC\nBob,25,LA"
  end

  def parse_data(raw_data)
    raw_data.split("\n")
  end
end

# Extension: PDF data mining — different extraction and parsing
class PDFDataMiner < DataMiner
  protected

  def extract_data(path)
    puts "Extracting text from PDF: #{path}"
    # Use PDF library to extract text
    "extracted pdf text content here"
  end

  def parse_data(raw_data)
    raw_data.split(/\s+/)
  end

  # Override hook: send report via email instead of console
  def send_report(report)
    puts "[EMAIL] Sending PDF analysis report..."
    puts report
  end
end

# Extension: Database data mining
class DatabaseDataMiner < DataMiner
  protected

  def extract_data(path)
    puts "Querying database: #{path}"
    # Execute SQL query
    "row1|row2|row3"
  end

  def parse_data(raw_data)
    raw_data.split("|")
  end
end

# Usage — the mine() algorithm never changes, but behavior varies
CSVDataMiner.new.mine("data.csv")
PDFDataMiner.new.mine("report.pdf")
DatabaseDataMiner.new.mine("SELECT * FROM analytics")
```

---

### OCP with Blocks, Procs, and Lambdas

In Ruby, OCP can be achieved elegantly without inheritance using **blocks**, **Procs**, and **lambdas** — Ruby's first-class function objects:

```ruby
# Closed for modification — accepts any validation logic
class InputValidator
  def initialize
    @rules = []
  end

  def add_rule(name, &block)
    @rules << { name: name, check: block }
  end

  def validate(input)
    @rules.each do |rule|
      unless rule[:check].call(input)
        puts "Validation failed: #{rule[:name]}"
        return false
      end
    end
    true
  end
end

# Open for extension — add new rules without modifying InputValidator
email_validator = InputValidator.new
email_validator.add_rule("not_empty") { |s| !s.empty? }
email_validator.add_rule("has_at_sign") { |s| s.include?('@') }
email_validator.add_rule("has_dot") { |s| s.include?('.') }
email_validator.add_rule("min_length") { |s| s.length >= 5 }

# Later, add more rules without touching InputValidator:
email_validator.add_rule("no_spaces") { |s| !s.include?(' ') }
email_validator.add_rule("valid_domain") do |s|
  at_pos = s.index('@')
  return false unless at_pos
  domain = s[(at_pos + 1)..]
  domain.include?('.') && domain.length > 3
end

puts email_validator.validate("user@example.com")  # true
puts email_validator.validate("invalid")            # false
```

**Ruby-specific OCP with modules (mixins):**

```ruby
# Base class is closed for modification
class Report
  def generate
    header + body + footer
  end

  private

  def header
    "=== Report ===\n"
  end

  def footer
    "\n=== End ===\n"
  end

  def body
    raise NotImplementedError
  end
end

# Extend behavior via modules without modifying Report
module Timestamped
  def header
    "#{super}Generated at: #{Time.now}\n"
  end
end

module Paginated
  def body
    pages = super.chars.each_slice(80).map(&:join)
    pages.map.with_index(1) { |page, i| "--- Page #{i} ---\n#{page}" }.join("\n")
  end
end

# Compose behaviors
class SalesReport < Report
  include Timestamped
  include Paginated

  private

  def body
    "Sales data goes here..."
  end
end
```

---

### Key Takeaways for OCP

| Technique | How it achieves OCP | When to use |
|-----------|---|---|
| Inheritance + Method Overriding | New behavior via new subclasses | When you have a clear type hierarchy |
| Strategy Pattern | Swap algorithms without modifying context | When behavior varies independently of the object |
| Template Method | Override steps without changing algorithm structure | When algorithm skeleton is fixed but steps vary |
| Blocks / Procs / Lambdas | Inject behavior as callable objects | When you want lightweight, flexible extension |
| Modules (Mixins) | Add responsibilities without modifying original | When you need to layer behaviors dynamically |
| Decorator Pattern | Wrap objects to add behavior | When you need to compose behaviors at runtime |

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

If code works correctly with a base class object, it must continue to work correctly when that object is replaced with any subclass object. The subclass must honor the contract established by the base class.

```ruby
# A method that works with the base type
def process_shape(shape)
  area = shape.area
  raise "Area must be non-negative" if area < 0
  puts "Area: #{area}"
end

# LSP says: ANY subclass of Shape passed to process_shape() must work correctly
# - Circle, Rectangle, Triangle — all should return valid non-negative areas
# - No subclass should throw unexpected exceptions
# - No subclass should violate the contract (e.g., return negative area)
```

**A correct LSP-compliant hierarchy:**

```ruby
class Bird
  def name
    raise NotImplementedError
  end

  def habitat
    raise NotImplementedError
  end

  def eat
    raise NotImplementedError
  end
end

class FlyingBird < Bird
  def fly
    raise NotImplementedError
  end

  def altitude
    raise NotImplementedError
  end
end

class Sparrow < FlyingBird
  def name = "Sparrow"
  def habitat = "Urban areas"
  def eat = puts "Sparrow eating seeds"
  def fly = puts "Sparrow flying"
  def altitude = 50.0
end

class Penguin < Bird  # NOT FlyingBird — penguins can't fly!
  def name = "Penguin"
  def habitat = "Antarctic"
  def eat = puts "Penguin eating fish"
  def swim = puts "Penguin swimming"
end

# LSP-compliant usage:
def describe_bird(bird)
  puts "#{bird.name} lives in #{bird.habitat}"
  # This works for ALL birds — Sparrow, Penguin, Eagle, etc.
end

def make_fly(bird)
  bird.fly
  puts "Altitude: #{bird.altitude}m"
  # This works for ALL flying birds — Sparrow, Eagle, etc.
  # Penguin is NOT a FlyingBird, so it shouldn't be passed here
end
```

**LSP violation — the classic bad design:**

```ruby
# BAD: Penguin inherits from a Bird class that promises fly()
class BirdBad
  def fly
    raise NotImplementedError
  end
end

class PenguinBad < BirdBad
  def fly
    raise RuntimeError, "Penguins can't fly!"  # VIOLATES LSP!
    # Any code expecting BirdBad#fly to work will break
  end
end

def migrate_birds(flock)
  flock.each { |bird| bird.fly }  # CRASH if any bird is a Penguin!
end
```

---

### The Rectangle-Square Problem

This is the most famous example of an LSP violation. Mathematically, a square IS a rectangle (a rectangle with equal sides). But in OOP, making `Square` inherit from `Rectangle` violates LSP.

**The violation:**

```ruby
class Rectangle
  attr_accessor :width, :height

  def initialize(width, height)
    @width = width
    @height = height
  end

  def area
    @width * @height
  end
end

class Square < Rectangle
  def initialize(side)
    super(side, side)
  end

  # Must keep width == height to maintain square invariant
  def width=(w)
    @width = w
    @height = w  # forced to change both!
  end

  def height=(h)
    @width = h   # forced to change both!
    @height = h
  end
end

# This method works correctly for Rectangle but BREAKS for Square
def test_rectangle(r)
  r.width = 5
  r.height = 4

  # For a Rectangle, we expect:
  raise "Width should be 5" unless r.width == 5    # FAILS for Square! (width became 4)
  raise "Height should be 4" unless r.height == 4  # passes
  raise "Area should be 20" unless r.area == 20    # FAILS for Square! (area is 16)

  puts "Expected area 20, got #{r.area}"
end

rect = Rectangle.new(3, 3)
test_rectangle(rect)  # works: area = 20

sq = Square.new(3)
test_rectangle(sq)    # FAILS: area = 16, not 20!
# Square violates the contract that width= and height= are independent
```

**Why this violates LSP:**
- `Rectangle` establishes a contract: `width=` changes only width, `height=` changes only height
- `Square` breaks this contract: setting one dimension changes both
- Code written for `Rectangle` produces incorrect results when given a `Square`
- The subtype is NOT substitutable for the base type

**Solutions to the Rectangle-Square problem:**

**Solution 1: Make shapes immutable (no setters)**
```ruby
class Shape
  def area
    raise NotImplementedError
  end

  def perimeter
    raise NotImplementedError
  end
end

class Rectangle < Shape
  attr_reader :width, :height

  def initialize(width, height)
    @width = width
    @height = height
  end

  def area
    @width * @height
  end

  def perimeter
    2 * (@width + @height)
  end

  # "Modification" returns a new object
  def with_width(w)
    Rectangle.new(w, @height)
  end

  def with_height(h)
    Rectangle.new(@width, h)
  end
end

class Square < Shape
  attr_reader :side

  def initialize(side)
    @side = side
  end

  def area
    @side * @side
  end

  def perimeter
    4 * @side
  end

  def with_side(s)
    Square.new(s)
  end
end

# No LSP violation — no mutable state to create inconsistency
# Square is NOT a subtype of Rectangle — they're siblings under Shape
```

**Solution 2: Separate hierarchies (no inheritance between Rectangle and Square)**
```ruby
class Shape
  def area
    raise NotImplementedError
  end
end

class Rectangle < Shape
  attr_accessor :width, :height

  def initialize(width, height)
    @width = width
    @height = height
  end

  def area
    @width * @height
  end
end

class Square < Shape
  attr_accessor :side

  def initialize(side)
    @side = side
  end

  def area
    @side * @side
  end
end

# Square and Rectangle are independent — no LSP issue
# Both are Shapes, but Square is NOT a Rectangle
```

**Solution 3: Use a factory method if you need both**
```ruby
class Rectangle
  attr_accessor :width, :height

  def initialize(width, height)
    @width = width
    @height = height
  end

  # Factory method for creating a square (a rectangle with equal sides)
  def self.create_square(side)
    new(side, side)
  end

  def area
    @width * @height
  end

  def square?
    @width == @height
  end
end

# No inheritance, no LSP violation
# A "square" is just a rectangle that happens to have equal sides
sq = Rectangle.create_square(5)
sq.width = 10  # perfectly fine — it's just a rectangle now
```

---

### Preconditions and Postconditions

LSP can be formally defined in terms of **preconditions** (what must be true before a method is called) and **postconditions** (what must be true after a method returns).

**LSP Rules:**
1. **Preconditions cannot be strengthened in a subtype** — the derived class cannot demand MORE from the caller than the base class does
2. **Postconditions cannot be weakened in a subtype** — the derived class must guarantee at least as much as the base class promises

```ruby
# Base class contract:
class Account
  attr_reader :balance

  def initialize(initial_balance)
    @balance = initial_balance
  end

  # Precondition: amount > 0
  # Postcondition: balance increases by exactly 'amount'
  def deposit(amount)
    raise ArgumentError, "Amount must be positive" unless amount > 0
    @balance += amount
  end

  # Precondition: amount > 0 AND amount <= balance
  # Postcondition: balance decreases by exactly 'amount', returns true
  #                OR balance unchanged, returns false (insufficient funds)
  def withdraw(amount)
    raise ArgumentError, "Amount must be positive" unless amount > 0
    return false if amount > @balance
    @balance -= amount
    true
  end
end
```

**LSP-COMPLIANT subclass (weaker precondition, stronger postcondition):**
```ruby
class PremiumAccount < Account
  def initialize(initial_balance, overdraft_limit)
    super(initial_balance)
    @overdraft_limit = overdraft_limit
  end

  # WEAKER precondition: allows withdrawal even when amount > balance
  # (up to overdraft limit) — this is FINE, it accepts MORE inputs
  def withdraw(amount)
    raise ArgumentError, "Amount must be positive" unless amount > 0
    return false if amount > @balance + @overdraft_limit
    @balance -= amount  # balance can go negative (up to -overdraft_limit)
    true
  end

  # STRONGER postcondition: deposit also earns 1% bonus
  # Guarantees MORE than base class — this is FINE
  def deposit(amount)
    raise ArgumentError, "Amount must be positive" unless amount > 0
    @balance += amount * 1.01  # 1% bonus — gives MORE than promised
  end
end
```

**LSP-VIOLATING subclass (stronger precondition):**
```ruby
class RestrictedAccount < Account
  def initialize(initial_balance, max_withdrawal)
    super(initial_balance)
    @max_withdrawal = max_withdrawal
  end

  # STRONGER precondition: amount must also be <= max_withdrawal
  # This VIOLATES LSP — demands MORE from the caller!
  def withdraw(amount)
    raise ArgumentError, "Amount must be positive" unless amount > 0
    raise RuntimeError, "Exceeds maximum withdrawal limit" if amount > @max_withdrawal  # NEW restriction!
    return false if amount > @balance
    @balance -= amount
    true
  end
end

# Code that works with Account will BREAK with RestrictedAccount:
def withdraw_all(account)
  balance = account.balance
  success = account.withdraw(balance)  # should work per Account's contract
  # But RestrictedAccount raises if balance > max_withdrawal!
  # LSP VIOLATED
end
```

**LSP-VIOLATING subclass (weaker postcondition):**
```ruby
class BrokenAccount < Account
  # WEAKER postcondition: doesn't guarantee balance increases by 'amount'
  def deposit(amount)
    raise ArgumentError, "Amount must be positive" unless amount > 0
    # "Processing fee" — balance increases by LESS than amount
    @balance += amount * 0.95  # only 95% deposited — VIOLATES postcondition!
  end
end

# Code expecting Account's contract:
def double_balance(account)
  current = account.balance
  account.deposit(current)
  expected = 2 * current
  raise "Expected #{expected}, got #{account.balance}" unless account.balance == expected
  # FAILS with BrokenAccount!
end
```

**Summary of precondition/postcondition rules:**

| Rule | Allowed | Violation |
|------|---------|-----------|
| Precondition | Same or WEAKER (accept more) | STRONGER (reject what base accepts) |
| Postcondition | Same or STRONGER (guarantee more) | WEAKER (guarantee less than base) |
| Invariant | Must be maintained | Breaking class invariant |
| Exceptions | Same or fewer exceptions | New unexpected exceptions |

---

### Duck Typing and LSP in Ruby

Ruby's duck typing means LSP applies not just to inheritance hierarchies but to any object that responds to the expected methods. If code expects an object with certain methods, any object providing those methods must honor the behavioral contract.

```ruby
# In Ruby, LSP applies through duck typing — no inheritance required
# Any object that responds to #area and returns a non-negative number is valid

def total_area(shapes)
  shapes.sum do |shape|
    area = shape.area
    raise "#{shape.class}#area returned negative value" if area < 0
    area
  end
end

# These all work — they honor the contract (respond to #area, return non-negative)
class Circle
  def initialize(radius) = @radius = radius
  def area = Math::PI * @radius**2
end

class Rectangle
  def initialize(w, h) = (@width, @height = w, h)
  def area = @width * @height
end

# Even a Struct works — duck typing!
TriangleShape = Struct.new(:base, :height) do
  def area
    0.5 * base * height
  end
end

shapes = [Circle.new(5), Rectangle.new(3, 4), TriangleShape.new(6, 3)]
puts total_area(shapes)  # works perfectly — all honor the contract
```

---

### Behavioral Subtyping

LSP is ultimately about **behavioral subtyping** — the subclass must behave in a way that is consistent with the expectations set by the base class. This goes beyond just matching method signatures.

**Behavioral contract includes:**
1. Method signatures (enforced by duck typing in Ruby)
2. Preconditions and postconditions (logical contracts)
3. Invariants (conditions that must always hold)
4. History constraint (how the object's state evolves over time)

```ruby
# Base class with a behavioral contract
class Stack
  # Invariant: size >= 0
  # Invariant: after push(x), top == x
  # Invariant: after push(x), size increases by 1
  # Invariant: after pop, size decreases by 1

  def push(value)
    raise NotImplementedError
  end

  def pop
    raise NotImplementedError
  end

  def top
    raise NotImplementedError
  end

  def size
    raise NotImplementedError
  end

  def empty?
    raise NotImplementedError
  end
end

# LSP-COMPLIANT: Bounded stack (adds a limit but doesn't break existing behavior)
class BoundedStack < Stack
  def initialize(max_size)
    @data = []
    @max_size = max_size
  end

  def push(value)
    raise "Stack is full" if @data.size >= @max_size
    @data.push(value)
  end

  def pop
    raise "Stack is empty" if @data.empty?
    @data.pop
  end

  def top
    raise "Stack is empty" if @data.empty?
    @data.last
  end

  def size
    @data.size
  end

  def empty?
    @data.empty?
  end
end

# LSP-VIOLATING: A "stack" that doesn't maintain LIFO order
class RandomStack < Stack
  def initialize
    @data = []
  end

  def push(value)
    @data.push(value)
  end

  def pop
    raise "Empty" if @data.empty?
    # Returns a RANDOM element instead of the last one pushed!
    idx = rand(@data.size)
    @data.delete_at(idx)  # VIOLATES: after push(x), pop should return x (if no other push)
  end

  def top
    raise "Empty" if @data.empty?
    @data.last  # inconsistent with pop!
  end

  def size
    @data.size
  end

  def empty?
    @data.empty?
  end
end

# Code that relies on Stack's behavioral contract:
def undo_last_action(stack)
  unless stack.empty?
    last_action = stack.pop  # expects LIFO behavior
    puts "Undoing action: #{last_action}"
  end
end
# RandomStack breaks this — it might "undo" a random action, not the last one!
```

**The History Constraint:**

The history constraint says that a subtype should not allow state transitions that the base type would not allow.

```ruby
# Base: Immutable point (once created, coordinates don't change)
class ImmutablePoint
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
    freeze  # Ruby's way to enforce immutability
  end
end

# VIOLATION: Mutable subclass breaks the immutability contract
class MutablePoint < ImmutablePoint
  def initialize(x, y)
    @x = x
    @y = y
    # NOT calling freeze — breaks the immutability guarantee
  end

  def x=(new_x)
    @x = new_x  # allows mutation!
  end

  def y=(new_y)
    @y = new_y
  end
end

# If code stores an ImmutablePoint reference and expects it to never change,
# a MutablePoint subclass violates that expectation.
```

---

### Signs of LSP Violations

Watch for these code smells that indicate LSP violations:

1. **Type checking in client code:**
```ruby
# If you see this, LSP is likely violated
def process(animal)
  if animal.is_a?(Penguin)
    # special handling because Penguin doesn't fit the Animal contract
  else
    animal.fly  # assumes all animals can fly
  end
end
```

2. **Empty or raising method implementations:**
```ruby
class ReadOnlyCollection < Collection
  def add(item)
    raise NotImplementedError, "Cannot add to read-only collection"  # LSP violation!
  end
end
```

3. **Conditional behavior based on subtype:**
```ruby
def calculate_area(shape)
  case shape
  when Square
    # special formula for square
  when Rectangle
    # different formula for rectangle
  end
end
```

4. **Documentation saying "don't call X on this subtype"**

**How to fix LSP violations:**
- Redesign the hierarchy (separate interfaces for different capabilities)
- Use composition instead of inheritance
- Make the base class contract more precise
- Split into smaller, more focused interfaces (ISP helps here)
- In Ruby, use modules to define capabilities rather than deep inheritance

---


## 3.4 Interface Segregation Principle (ISP)

> "No client should be forced to depend on methods it does not use." — Robert C. Martin

The Interface Segregation Principle states that large, monolithic interfaces should be split into smaller, more specific ones so that clients only need to know about the methods that are relevant to them. A class should not be forced to implement interfaces it doesn't use.

**Note on Ruby:** Ruby doesn't have formal interfaces like Java or C++. Instead, ISP in Ruby is achieved through **modules (mixins)**, **duck typing**, and **small, focused classes**. The principle still applies — objects shouldn't be burdened with methods they don't need.

---

### No Client Should Depend on Methods It Doesn't Use

**The problem with fat interfaces:**

When a class depends on an interface with methods it doesn't need, it becomes coupled to those methods. Implementors are burdened with providing meaningless implementations.

```ruby
# BAD: Fat interface — forces ALL implementations to handle everything
module Machine
  def print_document(doc)
    raise NotImplementedError
  end

  def scan_document(doc)
    raise NotImplementedError
  end

  def fax_document(doc)
    raise NotImplementedError
  end

  def staple_document(doc)
    raise NotImplementedError
  end

  def copy_document(doc, copies)
    raise NotImplementedError
  end
end

# A simple printer is FORCED to implement scan, fax, staple, copy!
class SimplePrinter
  include Machine

  def print_document(doc)
    puts "Printing: #{doc.title}"
  end

  # Forced to provide meaningless implementations:
  def scan_document(doc)
    raise RuntimeError, "SimplePrinter cannot scan!"  # BAD
  end

  def fax_document(doc)
    raise RuntimeError, "SimplePrinter cannot fax!"   # BAD
  end

  def staple_document(doc)
    # do nothing — silent failure is even worse!
  end

  def copy_document(doc, copies)
    raise RuntimeError, "SimplePrinter cannot copy!"  # BAD
  end
end

# A scanner is FORCED to implement print, fax, staple!
class SimpleScanner
  include Machine

  def scan_document(doc)
    puts "Scanning: #{doc.title}"
  end

  def print_document(doc)
    raise RuntimeError, "SimpleScanner cannot print!"
  end

  def fax_document(doc)
    raise RuntimeError, "SimpleScanner cannot fax!"
  end

  def staple_document(doc)
    raise RuntimeError, "SimpleScanner cannot staple!"
  end

  def copy_document(doc, copies)
    raise RuntimeError, "SimpleScanner cannot copy!"
  end
end
```

**Problems:**
- `SimplePrinter` is polluted with methods it can never meaningfully implement
- Client code that only needs printing still depends on the scan/fax/staple interface
- Adding a new method to `Machine` forces ALL implementations to change
- Violates LSP: calling `scan_document` on a `SimplePrinter` raises unexpectedly

---

### Fat Interfaces vs Thin Interfaces

**Fat Interface:** A single large module with many methods covering multiple concerns. Forces implementors to deal with all methods even if they only need a subset.

**Thin Interface:** A small, focused module with a few cohesive methods representing a single capability. Implementors only include what they actually support.

```ruby
# GOOD: Thin, focused modules — each represents ONE capability

module Printable
  def print_document(doc)
    raise NotImplementedError, "#{self.class} must implement #print_document"
  end

  def print_multiple(doc, copies)
    raise NotImplementedError, "#{self.class} must implement #print_multiple"
  end

  def paper_loaded?
    raise NotImplementedError, "#{self.class} must implement #paper_loaded?"
  end
end

module Scannable
  def scan
    raise NotImplementedError, "#{self.class} must implement #scan"
  end

  def scan_with_resolution(dpi)
    raise NotImplementedError, "#{self.class} must implement #scan_with_resolution"
  end

  def ready?
    raise NotImplementedError, "#{self.class} must implement #ready?"
  end
end

module Faxable
  def send_fax(doc, number)
    raise NotImplementedError, "#{self.class} must implement #send_fax"
  end

  def receive_fax
    raise NotImplementedError, "#{self.class} must implement #receive_fax"
  end
end

module Stapleable
  def staple(doc)
    raise NotImplementedError, "#{self.class} must implement #staple"
  end

  def staples_remaining
    raise NotImplementedError, "#{self.class} must implement #staples_remaining"
  end
end

# Now each device implements ONLY what it supports:

class SimplePrinter
  include Printable

  def print_document(doc)
    puts "Printing: #{doc.title}"
  end

  def print_multiple(doc, copies)
    copies.times { print_document(doc) }
  end

  def paper_loaded?
    true
  end
end

class SimpleScanner
  include Scannable

  def scan
    puts "Scanning at default resolution..."
    Document.new("Scanned Document")
  end

  def scan_with_resolution(dpi)
    puts "Scanning at #{dpi} DPI..."
    Document.new("High-res Scan")
  end

  def ready?
    true
  end
end

# Multi-function device includes multiple modules
class MultiFunctionPrinter
  include Printable
  include Scannable
  include Faxable
  include Stapleable

  # Printable
  def print_document(doc)
    puts "MFP printing: #{doc.title}"
  end

  def print_multiple(doc, copies)
    copies.times { print_document(doc) }
  end

  def paper_loaded?
    true
  end

  # Scannable
  def scan
    puts "MFP scanning..."
    Document.new("MFP Scan")
  end

  def scan_with_resolution(dpi)
    puts "MFP scanning at #{dpi} DPI..."
    Document.new("MFP High-res Scan")
  end

  def ready?
    true
  end

  # Faxable
  def send_fax(doc, number)
    puts "MFP faxing to #{number}"
    true
  end

  def receive_fax
    Document.new("Received Fax")
  end

  # Stapleable
  def staple(doc)
    puts "MFP stapling: #{doc.title}"
  end

  def staples_remaining
    100
  end
end
```

**Client code depends only on what it needs:**

```ruby
# This method only needs printing — doesn't know or care about scanning/faxing
def print_report(printer, report)
  if printer.paper_loaded?
    printer.print_document(report)
  else
    puts "Please load paper!"
  end
end

# This method only needs scanning
def digitize_document(scanner)
  if scanner.ready?
    scanned = scanner.scan_with_resolution(300)
    # ... process scanned document ...
  end
end

# Both SimplePrinter and MultiFunctionPrinter work with print_report()
# Both SimpleScanner and MultiFunctionPrinter work with digitize_document()
# Each method depends on the MINIMUM interface it needs
```

---

### Splitting Interfaces in Ruby (Modules as Interfaces)

Ruby achieves ISP through **modules (mixins)**. Since Ruby supports including multiple modules, classes can compose exactly the capabilities they need.

**Real-world example: Plugin system**

```ruby
# Segregated modules for a plugin system

module Initializable
  def initialize_plugin(config)
    raise NotImplementedError
  end

  def shutdown
    raise NotImplementedError
  end

  def initialized?
    raise NotImplementedError
  end
end

module Updatable
  def update(delta_time)
    raise NotImplementedError
  end
end

module Renderable
  def render
    raise NotImplementedError
  end

  def visible=(visible)
    raise NotImplementedError
  end

  def visible?
    raise NotImplementedError
  end
end

module Serializable
  def serialize
    raise NotImplementedError
  end

  def deserialize(data)
    raise NotImplementedError
  end
end

module EventListening
  def on_event(event)
    raise NotImplementedError
  end

  def subscribed_events
    raise NotImplementedError
  end
end

module Configurable
  def configure(settings)
    raise NotImplementedError
  end

  def configuration
    raise NotImplementedError
  end
end

# A physics plugin: needs initialization, updating, and event handling
# Does NOT need rendering or serialization
class PhysicsPlugin
  include Initializable
  include Updatable
  include EventListening

  def initialize
    @initialized = false
  end

  def initialize_plugin(config)
    puts "Physics engine initialized"
    @initialized = true
  end

  def shutdown
    @initialized = false
  end

  def initialized?
    @initialized
  end

  def update(dt)
    puts "Physics step: #{dt}s"
  end

  def on_event(event)
    # Handle collision events, etc.
  end

  def subscribed_events
    ["collision", "force_applied"]
  end
end

# A UI widget: needs rendering, events, and configuration
# Does NOT need physics updates or serialization
class UIWidget
  include Renderable
  include EventListening
  include Configurable

  def initialize
    @visible = true
    @config = {}
  end

  def render
    puts "Rendering widget" if visible?
  end

  def visible=(v)
    @visible = v
  end

  def visible?
    @visible
  end

  def on_event(event)
    # Handle click, hover, etc.
  end

  def subscribed_events
    ["click", "hover", "key_press"]
  end

  def configure(settings)
    @config = settings
  end

  def configuration
    @config
  end
end

# A save-game system: needs serialization and initialization only
class SaveSystem
  include Initializable
  include Serializable

  def initialize
    @initialized = false
    @game_state = ""
  end

  def initialize_plugin(config)
    @initialized = true
  end

  def shutdown
    @initialized = false
  end

  def initialized?
    @initialized
  end

  def serialize
    @game_state
  end

  def deserialize(data)
    @game_state = data
  end
end

# The engine processes each interface independently:
class GameEngine
  def initialize
    @initializables = []
    @updatables = []
    @renderables = []
    @listeners = []
  end

  def register(plugin)
    @initializables << plugin if plugin.is_a?(Initializable)
    @updatables << plugin if plugin.is_a?(Updatable)
    @renderables << plugin if plugin.is_a?(Renderable)
    @listeners << plugin if plugin.is_a?(EventListening)
  end

  def initialize_all(config)
    @initializables.each { |obj| obj.initialize_plugin(config) }
  end

  def update_all(dt)
    @updatables.each { |obj| obj.update(dt) }
  end

  def render_all
    @renderables.select(&:visible?).each(&:render)
  end

  def broadcast_event(event)
    @listeners.each { |listener| listener.on_event(event) }
  end
end
```

---

### Role Interfaces

A **role interface** defines a specific role that an object can play in a particular context. Instead of defining interfaces based on what an object IS, define them based on what role the object PLAYS.

```ruby
# Role-based modules — defined by what the object DOES in a specific context

# Role: Something that can be persisted to storage
module Persistable
  def entity_id
    raise NotImplementedError
  end

  def to_record
    raise NotImplementedError
  end

  def from_record(record)
    raise NotImplementedError
  end
end

# Role: Something that can be validated
module Validatable
  def valid?
    raise NotImplementedError
  end

  def validation_errors
    raise NotImplementedError
  end
end

# Role: Something that can be audited (track changes)
module Auditable
  def last_modified_by
    raise NotImplementedError
  end

  def last_modified_at
    raise NotImplementedError
  end

  def change_history
    raise NotImplementedError
  end
end

# Role: Something that can be displayed to users
module Displayable
  def display_name
    raise NotImplementedError
  end

  def description
    raise NotImplementedError
  end

  def icon_path
    raise NotImplementedError
  end
end

# Role: Something that can be searched/indexed
module Searchable
  def searchable_fields
    raise NotImplementedError
  end

  def search_summary
    raise NotImplementedError
  end
end

# A Product plays multiple roles depending on context
class Product
  include Persistable
  include Validatable
  include Auditable
  include Displayable
  include Searchable

  attr_reader :id, :name, :price, :stock
  attr_accessor :description_text

  def initialize(id, name, price)
    @id = id
    @name = name
    @price = price
    @stock = 0
    @description_text = ""
    @last_modifier = nil
    @last_modified = Time.now
    @history = []
  end

  # Persistable role
  def entity_id
    @id
  end

  def to_record
    { id: @id, name: @name, price: @price, stock: @stock }
  end

  def from_record(record)
    @name = record[:name]
    @price = record[:price]
    @stock = record[:stock]
  end

  # Validatable role
  def valid?
    validation_errors.empty?
  end

  def validation_errors
    errors = []
    errors << "Name is required" if @name.nil? || @name.empty?
    errors << "Price must be positive" unless @price.positive?
    errors << "Stock cannot be negative" if @stock.negative?
    errors
  end

  # Auditable role
  def last_modified_by
    @last_modifier
  end

  def last_modified_at
    @last_modified
  end

  def change_history
    @history
  end

  # Displayable role
  def display_name
    @name
  end

  def description
    @description_text
  end

  def icon_path
    "/icons/product.png"
  end

  # Searchable role
  def searchable_fields
    [@name, @description_text, @id]
  end

  def search_summary
    "#{@name} - $#{@price}"
  end
end

# Each subsystem only depends on the role it cares about:

class Repository
  def save(entity)
    # Only cares about Persistable
    record = entity.to_record
    puts "Saving entity #{entity.entity_id}"
    # ... save to database ...
  end
end

class ValidationService
  def validate(entity)
    # Only cares about Validatable
    unless entity.valid?
      entity.validation_errors.each { |error| $stderr.puts "Validation error: #{error}" }
      return false
    end
    true
  end
end

class SearchEngine
  def index(entity)
    # Only cares about Searchable
    fields = entity.searchable_fields
    puts "Indexing: #{entity.search_summary}"
    # ... add to search index ...
  end
end

class AuditLogger
  def log_change(entity)
    # Only cares about Auditable
    puts "Change by #{entity.last_modified_by} at #{entity.last_modified_at}"
  end
end
```

---

### Duck Typing as Natural ISP

Ruby's duck typing naturally supports ISP — methods only care about the specific messages they send, not the full interface of the object:

```ruby
# Ruby's duck typing means methods naturally depend on minimal interfaces

# This method only needs #each — works with Array, Hash, Set, custom collections
def process_items(collection)
  collection.each { |item| puts item }
end

# This method only needs #read and #close — works with File, StringIO, Socket
def consume_stream(io)
  data = io.read
  io.close
  data
end

# This method only needs #call — works with Proc, Lambda, Method, any callable
def execute_action(action)
  action.call
end

# You can formalize duck type expectations with respond_to? checks:
def print_report(printer)
  unless printer.respond_to?(:print_document)
    raise ArgumentError, "Expected an object that responds to #print_document"
  end
  printer.print_document(report)
end
```

---

### Guidelines for Interface Segregation

**How to identify interfaces that need splitting:**

1. **Look for "raise NotImplementedError" patterns** — if implementors raise for methods they can't support, the interface is too fat

2. **Look for empty method bodies** — if implementors leave methods empty, they don't need those methods

3. **Look for groups of methods used together** — methods that are always called together belong in the same module; methods called independently should be in separate modules

4. **Apply the "Interface Pollution" test** — if adding a new implementor requires implementing methods that make no sense for it, the interface needs splitting

**Granularity balance:**

```ruby
# TOO GRANULAR — one method per module is excessive
module Addable
  def add(x) = raise NotImplementedError
end
module Removable
  def remove(x) = raise NotImplementedError
end
module Findable
  def find(x) = raise NotImplementedError
end
# This creates too many tiny modules — hard to work with

# TOO COARSE — everything in one module
module Everything
  def add(x) = raise NotImplementedError
  def remove(x) = raise NotImplementedError
  def find(x) = raise NotImplementedError
  def sort = raise NotImplementedError
  def serialize = raise NotImplementedError
  def render = raise NotImplementedError
  # ... 20 more methods
end

# JUST RIGHT — cohesive groups of related methods
module Collection
  def add(item) = raise NotImplementedError
  def remove(item) = raise NotImplementedError
  def include?(item) = raise NotImplementedError
  def size = raise NotImplementedError
  def empty? = raise NotImplementedError
end

module Sortable
  def sort = raise NotImplementedError
  def sort_by(&block) = raise NotImplementedError
end

module Serializable
  def serialize = raise NotImplementedError
  def deserialize(data) = raise NotImplementedError
end
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

```ruby
# LOW-LEVEL: Concrete database implementation
class MySQLDatabase
  def connect(connection_string)
    puts "Connected to MySQL: #{connection_string}"
  end

  def query(sql)
    puts "MySQL executing: #{sql}"
    []  # results
  end

  def execute(sql)
    puts "MySQL executing: #{sql}"
  end
end

# LOW-LEVEL: Concrete email service
class SendGridEmailService
  def initialize(api_key)
    @api_key = api_key
  end

  def send_email(to, subject, body)
    puts "SendGrid: sending to #{to}"
  end
end

# HIGH-LEVEL: Business logic — DIRECTLY depends on low-level modules!
class OrderService
  def initialize
    @database = MySQLDatabase.new              # DIRECT dependency on MySQL!
    @email_service = SendGridEmailService.new("api-key-123")  # DIRECT dependency on SendGrid!
    @database.connect("mysql://localhost/orders")
  end

  def place_order(user_id, items)
    # Business logic mixed with infrastructure details
    @database.execute("INSERT INTO orders ...")  # tied to MySQL
    @email_service.send_email(user_id, "Order Confirmed", "...")  # tied to SendGrid

    # What if we want to:
    # - Switch to PostgreSQL? Must modify OrderService!
    # - Switch to Mailgun? Must modify OrderService!
    # - Test without a real database? Impossible!
    # - Test without sending real emails? Impossible!
  end
end
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

```ruby
# ABSTRACTIONS (interfaces via modules/duck typing) — owned by the high-level module

# In Ruby, we define the expected interface through documentation and duck typing.
# Optionally, we can use modules to formalize the contract:

module OrderRepository
  def save(order)
    raise NotImplementedError
  end

  def find_by_id(order_id)
    raise NotImplementedError
  end

  def find_by_user(user_id)
    raise NotImplementedError
  end

  def update_status(order_id, status)
    raise NotImplementedError
  end
end

module NotificationService
  def send_order_confirmation(user_id, order)
    raise NotImplementedError
  end

  def send_shipping_update(user_id, tracking_num)
    raise NotImplementedError
  end

  def send_cancellation_notice(user_id, order)
    raise NotImplementedError
  end
end

module PaymentGateway
  def charge(user_id, amount)
    raise NotImplementedError
  end

  def refund(transaction_id, amount)
    raise NotImplementedError
  end
end

module InventoryService
  def available?(product_id, quantity)
    raise NotImplementedError
  end

  def reserve(product_id, quantity)
    raise NotImplementedError
  end

  def release(product_id, quantity)
    raise NotImplementedError
  end
end

# HIGH-LEVEL: Business logic depends ONLY on abstractions
class OrderService
  def initialize(repository:, notifications:, payments:, inventory:)
    @repository = repository
    @notifications = notifications
    @payments = payments
    @inventory = inventory
  end

  def place_order(user_id, items)
    # Pure business logic — no infrastructure details!

    # 1. Check inventory
    items.each do |item|
      unless @inventory.available?(item.product_id, item.quantity)
        raise OutOfStockError, item.product_id
      end
    end

    # 2. Calculate total
    total = items.sum { |item| item.price * item.quantity }

    # 3. Process payment
    payment_result = @payments.charge(user_id, total)
    raise PaymentFailedError, payment_result.error_message unless payment_result.success?

    # 4. Reserve inventory
    items.each { |item| @inventory.reserve(item.product_id, item.quantity) }

    # 5. Create and save order
    order = Order.new(user_id: user_id, items: items, total: total,
                      transaction_id: payment_result.transaction_id)
    @repository.save(order)

    # 6. Send confirmation
    @notifications.send_order_confirmation(user_id, order)

    order
  end

  def cancel_order(order_id)
    order = @repository.find_by_id(order_id)

    # Refund payment
    @payments.refund(order.transaction_id, order.total)

    # Release inventory
    order.items.each { |item| @inventory.release(item.product_id, item.quantity) }

    # Update status
    @repository.update_status(order_id, :cancelled)

    # Notify customer
    @notifications.send_cancellation_notice(order.user_id, order)
  end
end

# LOW-LEVEL: Concrete implementations depend on the same abstractions

class MySQLOrderRepository
  include OrderRepository

  def save(order)
    puts "MySQL: INSERT INTO orders ..."
  end

  def find_by_id(order_id)
    puts "MySQL: SELECT * FROM orders WHERE id = ..."
    Order.new
  end

  def find_by_user(user_id)
    []
  end

  def update_status(order_id, status)
    puts "MySQL: UPDATE orders SET status = ..."
  end
end

class PostgresOrderRepository
  include OrderRepository

  def save(order)
    puts "PostgreSQL: INSERT INTO orders ..."
  end

  def find_by_id(order_id)
    Order.new
  end

  def find_by_user(user_id)
    []
  end

  def update_status(order_id, status)
    puts "PostgreSQL: UPDATE orders SET status = ..."
  end
end

class EmailNotificationService
  include NotificationService

  def send_order_confirmation(user_id, order)
    puts "Email: Order confirmation to #{user_id}"
  end

  def send_shipping_update(user_id, tracking_num)
    puts "Email: Shipping update to #{user_id}"
  end

  def send_cancellation_notice(user_id, order)
    puts "Email: Cancellation notice to #{user_id}"
  end
end

class SMSNotificationService
  include NotificationService

  def send_order_confirmation(user_id, order)
    puts "SMS: Order confirmation to #{user_id}"
  end

  def send_shipping_update(user_id, tracking_num)
    puts "SMS: Shipping update to #{user_id}"
  end

  def send_cancellation_notice(user_id, order)
    puts "SMS: Cancellation notice to #{user_id}"
  end
end

class StripePaymentGateway
  include PaymentGateway

  def initialize(api_key)
    @api_key = api_key
  end

  def charge(user_id, amount)
    puts "Stripe: charging $#{amount}"
    PaymentResult.new(success: true, transaction_id: "txn_123")
  end

  def refund(transaction_id, amount)
    puts "Stripe: refunding $#{amount}"
    PaymentResult.new(success: true, transaction_id: transaction_id)
  end
end
```

**The dependency direction is now INVERTED:**

```
BEFORE (traditional):
  OrderService ──depends on──→ MySQLDatabase
  OrderService ──depends on──→ SendGridEmailService

AFTER (inverted):
  OrderService ──depends on──→ OrderRepository ←──implements── MySQLOrderRepository
  OrderService ──depends on──→ NotificationService ←──implements── EmailNotificationService

Both high-level and low-level depend on the ABSTRACTION (module/interface).
The abstraction is owned by the high-level module.
```

---

### Dependency Injection in Ruby (Constructor, Setter, Default with Override)

Dependency Injection (DI) is the practical mechanism for implementing DIP. It's how you actually provide the concrete implementations to the high-level module at runtime.

#### Constructor Injection (Preferred)

Dependencies are provided through the constructor (`initialize`). The object is always fully initialized.

```ruby
class ReportGenerator
  def initialize(data_source:, formatter:, output:)
    @data_source = data_source
    @formatter = formatter
    @output = output
  end

  def generate_report(report_type)
    data = @data_source.fetch_data(report_type)
    formatted = @formatter.format(data)
    @output.write(formatted)
  end
end

# Production configuration
prod_data = MySQLDataSource.new("mysql://prod-server/analytics")
html_fmt = HTMLFormatter.new
file_out = FileOutputTarget.new("/reports/daily.html")
prod_report = ReportGenerator.new(data_source: prod_data, formatter: html_fmt, output: file_out)
prod_report.generate_report("daily_sales")

# Test configuration — completely different implementations, same ReportGenerator
test_data = MockDataSource.new
text_fmt = PlainTextFormatter.new
string_out = StringOutputTarget.new
test_report = ReportGenerator.new(data_source: test_data, formatter: text_fmt, output: string_out)
test_report.generate_report("test")
# Verify: string_out.content contains expected output
```

**Advantages of constructor injection:**
- Object is always in a valid state (all dependencies present from creation)
- Dependencies are clearly visible in the constructor signature
- Easy to see when a class has too many dependencies (parameter count)
- Works naturally with Ruby's keyword arguments for clarity

#### Setter Injection

Dependencies are provided through setter methods after construction. Useful for optional dependencies or when dependencies need to change at runtime.

```ruby
class NotificationManager
  attr_writer :email_sender, :sms_sender, :push_notifier

  def initialize
    @email_sender = nil
    @sms_sender = nil
    @push_notifier = nil
  end

  def notify_user(user_id, message)
    @email_sender&.send(user_id, message)
    @sms_sender&.send(user_id, message)
    @push_notifier&.notify(user_id, message)
  end

  def any_channel?
    !@email_sender.nil? || !@sms_sender.nil? || !@push_notifier.nil?
  end
end

# Configure with available channels
manager = NotificationManager.new
manager.email_sender = SendGridSender.new("api-key")
manager.sms_sender = TwilioSMSSender.new("account-sid", "auth-token")
# Push notifications not available in this environment — that's fine

manager.notify_user("user123", "Your order has shipped!")
```

**When to use setter injection:**
- Optional dependencies (not all implementations need all dependencies)
- Dependencies that change at runtime (strategy pattern)
- Framework-managed lifecycle where construction and configuration are separate steps

#### Default with Override (Ruby Idiom)

A common Ruby pattern is to provide sensible defaults that can be overridden for testing or configuration:

```ruby
class UserService
  def initialize(repository: PostgresUserRepository.new,
                 logger: Rails.logger,
                 cache: RedisCache.new)
    @repository = repository
    @logger = logger
    @cache = cache
  end

  def find_user(id)
    # Check cache first
    cached = @cache.get("user:#{id}")
    if cached
      @logger.info("Cache hit for user #{id}")
      return cached
    end

    # Query database
    @logger.info("Cache miss, querying DB for user #{id}")
    user = @repository.find(id)

    # Store in cache
    @cache.set("user:#{id}", user)
    user
  end
end

# Production: uses defaults
service = UserService.new

# Testing: override with mocks
mock_repo = double("repository")
mock_cache = double("cache", get: nil, set: nil)
test_service = UserService.new(repository: mock_repo, logger: NullLogger.new, cache: mock_cache)
```

#### Interface Injection via Modules

```ruby
# Injection interfaces as modules
module DatabaseAware
  def database=(db)
    @database = db
  end
end

module LoggerAware
  def logger=(logger)
    @logger = logger
  end
end

module CacheAware
  def cache=(cache)
    @cache = cache
  end
end

# Service declares its dependencies by including modules
class UserService
  include DatabaseAware
  include LoggerAware
  include CacheAware

  def find_user(id)
    cached = @cache&.get("user:#{id}")
    if cached
      @logger&.info("Cache hit for user #{id}")
      return cached
    end

    @logger&.info("Cache miss, querying DB for user #{id}")
    user = @database.query("SELECT * FROM users WHERE id = ?", id)
    @cache&.set("user:#{id}", user)
    user
  end
end

# A simple DI container that wires dependencies
class DIContainer
  def initialize
    @services = {}
  end

  def register(name, instance)
    @services[name] = instance
  end

  def inject(service)
    service.database = @services[:database] if service.is_a?(DatabaseAware) && @services[:database]
    service.logger = @services[:logger] if service.is_a?(LoggerAware) && @services[:logger]
    service.cache = @services[:cache] if service.is_a?(CacheAware) && @services[:cache]
    service
  end
end

# Usage
container = DIContainer.new
container.register(:database, PostgresDatabase.new("postgres://localhost/app"))
container.register(:logger, FileLogger.new("app.log"))
container.register(:cache, RedisCache.new("redis://localhost"))

user_service = UserService.new
container.inject(user_service)  # automatically wires all dependencies

user = user_service.find_user("user-42")
```

---

### Inversion of Control (IoC) Concept

**Inversion of Control** is the broader principle that DIP and DI implement. In traditional programming, your code calls library code. With IoC, the framework calls your code. You give up control of the flow to the framework.

**Traditional control flow (you call the library):**
```ruby
# YOU decide when to create objects and call methods
db = MySQLDatabase.new
db.connect("localhost")

email = EmailService.new
email.configure("smtp.gmail.com")

orders = OrderService.new(db, email)
orders.process_order("order-1")
```

**Inversion of Control (the framework calls you):**
```ruby
# Framework/container manages object creation and lifecycle

class Application
  def configure(registry)
    raise NotImplementedError
  end

  def on_start
    raise NotImplementedError
  end

  def on_shutdown
    raise NotImplementedError
  end
end

class MyApp < Application
  def configure(registry)
    # DECLARE what you need — the framework creates and wires them
    registry.singleton(:database) { PostgresDatabase.new(ENV['DATABASE_URL']) }
    registry.singleton(:notifications) { EmailNotificationService.new }
    registry.transient(:order_repository) { PostgresOrderRepository.new }
    registry.singleton(:order_service) do |container|
      OrderService.new(
        repository: container.resolve(:order_repository),
        notifications: container.resolve(:notifications),
        payments: container.resolve(:payments),
        inventory: container.resolve(:inventory)
      )
    end
  end

  def on_start
    # Framework has already created and wired everything
    puts "Application started, OrderService ready"
  end

  def on_shutdown
    puts "Application shutting down"
  end
end

# The framework controls the lifecycle:
# 1. Creates MyApp
# 2. Calls configure() to learn about dependencies
# 3. Creates all registered services in correct order
# 4. Injects dependencies automatically
# 5. Calls on_start()
# 6. ... application runs ...
# 7. Calls on_shutdown()
# 8. Destroys services in reverse order
```

**A simple IoC container in Ruby:**

```ruby
class ServiceContainer
  def initialize
    @factories = {}
    @singletons = {}
  end

  # Register a factory for creating instances (new instance each time)
  def transient(name, &block)
    @factories[name] = block
  end

  # Register a singleton (created once, shared)
  def singleton(name, &block)
    @factories[name] = -> {
      @singletons[name] ||= block.call(self)
    }
  end

  # Resolve a dependency
  def resolve(name)
    factory = @factories[name]
    raise "Service not registered: #{name}" unless factory
    factory.call
  end

  # Shorthand
  def [](name)
    resolve(name)
  end
end

# Usage
container = ServiceContainer.new
container.singleton(:logger) { FileLogger.new("app.log") }
container.singleton(:database) { PostgresDatabase.new(ENV['DATABASE_URL']) }
container.transient(:order_repository) { PostgresOrderRepository.new }

logger = container.resolve(:logger)
db = container.resolve(:database)
repo = container.resolve(:order_repository)

# Same logger instance every time (singleton)
puts container[:logger].equal?(container[:logger])  # true

# Different repo instance every time (transient)
puts container[:order_repository].equal?(container[:order_repository])  # false
```

---

### DIP in Practice — Complete Example

```ruby
# A complete example showing DIP applied to a notification system

# --- Abstractions (owned by high-level module) ---

module MessageFormatter
  def format(template_name, variables)
    raise NotImplementedError
  end
end

module DeliveryChannel
  def deliver(recipient, subject, body)
    raise NotImplementedError
  end

  def channel_name
    raise NotImplementedError
  end

  def available?
    raise NotImplementedError
  end
end

module RecipientResolver
  def resolve_address(user_id, channel_type)
    raise NotImplementedError
  end

  def has_channel?(user_id, channel_type)
    raise NotImplementedError
  end
end

module NotificationLogger
  def log_sent(user_id, channel, subject)
    raise NotImplementedError
  end

  def log_failed(user_id, channel, reason)
    raise NotImplementedError
  end
end

# --- High-level module: Notification orchestration ---

class NotificationEngine
  def initialize(formatter:, resolver:, logger:)
    @formatter = formatter
    @resolver = resolver
    @logger = logger
    @channels = []
  end

  def add_channel(channel)
    @channels << channel
  end

  def notify(user_id, template_name, variables)
    body = @formatter.format(template_name, variables)
    subject = variables[:subject] || "Notification"

    any_sent = false

    @channels.each do |channel|
      next unless channel.available?

      channel_type = channel.channel_name
      next unless @resolver.has_channel?(user_id, channel_type)

      address = @resolver.resolve_address(user_id, channel_type)

      if channel.deliver(address, subject, body)
        @logger.log_sent(user_id, channel_type, subject)
        any_sent = true
      else
        @logger.log_failed(user_id, channel_type, "Delivery failed")
      end
    end

    @logger.log_failed(user_id, "all", "No channel delivered successfully") unless any_sent

    any_sent
  end
end

# --- Low-level modules: Concrete implementations ---

class HandlebarsFormatter
  include MessageFormatter

  def initialize
    @templates = {}
  end

  def load_template(name, template)
    @templates[name] = template
  end

  def format(template_name, variables)
    result = @templates.fetch(template_name)
    variables.each do |key, value|
      result = result.gsub("{{#{key}}}", value.to_s)
    end
    result
  end
end

class SMTPEmailChannel
  include DeliveryChannel

  def initialize(smtp_server, port)
    @smtp_server = smtp_server
    @port = port
  end

  def deliver(recipient, subject, body)
    puts "SMTP: Sending to #{recipient} via #{@smtp_server}"
    true
  end

  def channel_name
    "email"
  end

  def available?
    true
  end
end

class TwilioSMSChannel
  include DeliveryChannel

  def initialize(account_sid, auth_token)
    @account_sid = account_sid
    @auth_token = auth_token
  end

  def deliver(recipient, subject, body)
    puts "Twilio SMS: Sending to #{recipient}"
    true
  end

  def channel_name
    "sms"
  end

  def available?
    true
  end
end

class SlackChannel
  include DeliveryChannel

  def initialize(webhook_url)
    @webhook_url = webhook_url
  end

  def deliver(recipient, subject, body)
    puts "Slack: Posting to #{recipient}"
    true
  end

  def channel_name
    "slack"
  end

  def available?
    true
  end
end

class DatabaseRecipientResolver
  include RecipientResolver

  def resolve_address(user_id, channel_type)
    case channel_type
    when "email" then "#{user_id}@example.com"
    when "sms" then "+1234567890"
    when "slack" then "@#{user_id}"
    else ""
    end
  end

  def has_channel?(user_id, channel_type)
    true  # simplified
  end
end

class FileNotificationLogger
  include NotificationLogger

  def initialize(path)
    @file = File.open(path, 'a')
  end

  def log_sent(user_id, channel, subject)
    @file.puts "[SENT] #{user_id} via #{channel}: #{subject}"
    @file.flush
  end

  def log_failed(user_id, channel, reason)
    @file.puts "[FAILED] #{user_id} via #{channel}: #{reason}"
    @file.flush
  end
end

# --- Wiring it all together ---

def setup_notification_system
  # Create low-level implementations
  formatter = HandlebarsFormatter.new
  formatter.load_template("order_confirm",
    "Hi {{name}}, your order #{{order_id}} has been confirmed!")
  formatter.load_template("shipping_update",
    "{{name}}, your order #{{order_id}} is on its way! Tracking: {{tracking}}")

  resolver = DatabaseRecipientResolver.new
  logger = FileNotificationLogger.new("notifications.log")

  email = SMTPEmailChannel.new("smtp.example.com", 587)
  sms = TwilioSMSChannel.new("AC123", "token456")
  slack = SlackChannel.new("https://hooks.slack.com/...")

  # Wire high-level module with low-level implementations
  engine = NotificationEngine.new(formatter: formatter, resolver: resolver, logger: logger)
  engine.add_channel(email)
  engine.add_channel(sms)
  engine.add_channel(slack)

  # Use the system — high-level module doesn't know about SMTP, Twilio, or Slack
  engine.notify("alice", "order_confirm", {
    subject: "Order Confirmed",
    name: "Alice",
    order_id: "ORD-789"
  })
end
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

```ruby
# BAD: Duplicated validation logic
class UserRegistration
  def register_user(email, password)
    # Email validation duplicated here
    return false if email.empty? || !email.include?('@') || !email.include?('.')

    # Password validation duplicated here
    return false if password.length < 8 || password !~ /[A-Z]/ || password !~ /\d/

    # ... register ...
    true
  end
end

class UserProfileUpdate
  def update_email(new_email)
    # SAME email validation duplicated!
    return false if new_email.empty? || !new_email.include?('@') || !new_email.include?('.')

    # ... update ...
    true
  end
end

class PasswordReset
  def reset_password(new_password)
    # SAME password validation duplicated!
    return false if new_password.length < 8 || new_password !~ /[A-Z]/ || new_password !~ /\d/

    # ... reset ...
    true
  end
end
```

```ruby
# GOOD: Single source of truth for validation rules
class EmailValidator
  def self.valid?(email)
    return false if email.nil? || email.empty?
    return false unless email.include?('@')
    domain = email.split('@').last
    domain.include?('.') && domain.length > 3
  end

  def self.errors(email)
    errors = []
    errors << "Email is required" if email.nil? || email.empty?
    errors << "Missing @ symbol" if email && !email.include?('@')
    errors
  end
end

class PasswordValidator
  MIN_LENGTH = 8

  def self.valid?(password)
    password.length >= MIN_LENGTH &&
      password.match?(/[A-Z]/) &&
      password.match?(/[a-z]/) &&
      password.match?(/\d/)
  end

  def self.errors(password)
    errors = []
    errors << "Must be at least #{MIN_LENGTH} characters" if password.length < MIN_LENGTH
    errors << "Must contain uppercase letter" unless password.match?(/[A-Z]/)
    errors << "Must contain lowercase letter" unless password.match?(/[a-z]/)
    errors << "Must contain a digit" unless password.match?(/\d/)
    errors
  end
end

# Now all classes use the single source of truth:
class UserRegistration
  def register_user(email, password)
    return false unless EmailValidator.valid?(email)
    return false unless PasswordValidator.valid?(password)
    # ... register ...
    true
  end
end

class UserProfileUpdate
  def update_email(new_email)
    return false unless EmailValidator.valid?(new_email)
    # ... update ...
    true
  end
end

class PasswordReset
  def reset_password(new_password)
    return false unless PasswordValidator.valid?(new_password)
    # ... reset ...
    true
  end
end
```

**DRY is NOT just about code duplication:**
- Two identical code blocks that represent DIFFERENT concepts should NOT be merged
- If they change for different reasons, they are not true duplicates
- "Accidental duplication" (same code, different purposes) is fine to keep separate

---

### KISS (Keep It Simple, Stupid)

> "The simplest solution that works is usually the best."

KISS advocates for simplicity in design. Don't over-engineer, don't add unnecessary abstractions, and don't use complex patterns when a straightforward solution works.

```ruby
# BAD: Over-engineered for a simple task
# Just need to check if a number is even...

class NumberClassificationStrategyFactory
  def self.get_strategy(type)
    case type
    when :even then EvenNumberStrategy.new
    when :odd then OddNumberStrategy.new
    end
  end
end

class EvenNumberStrategy
  def classify(n)
    n.even?
  end
end

# ... 30 lines of code to check if a number is even

# GOOD: Simple and clear
def even?(n)
  n.even?
end

# Or just use Ruby's built-in: 42.even?
```

**When complexity IS justified:**
- When requirements genuinely demand it (multiple payment methods → Strategy pattern)
- When you have evidence of future extension needs (not speculation)
- When the complexity reduces overall system complexity (a well-designed abstraction)

**KISS in practice:**
```ruby
# KISS: Use the simplest data structure that works
# Need to store unique items and check membership? Use Set.
require 'set'
active_users = Set.new
active_users.add("alice")
is_active = active_users.include?("alice")

# KISS: Use standard library methods instead of reimplementing
nums = [5, 2, 8, 1, 9]
nums.sort!                    # don't write your own sort!
nums.include?(8)              # don't write your own search!
nums.select { |n| n > 3 }    # use Enumerable methods!

# KISS: Use Ruby's built-in features
# Instead of a complex builder pattern for simple objects:
User = Struct.new(:name, :email, :role, keyword_init: true)
user = User.new(name: "Alice", email: "alice@example.com", role: :admin)
```

---

### YAGNI (You Aren't Gonna Need It)

> "Don't implement something until you actually need it."

YAGNI warns against adding functionality based on speculation about future requirements. Premature features add complexity, maintenance burden, and often turn out to be wrong when the actual requirement arrives.

```ruby
# BAD: Adding "just in case" features that aren't needed yet

class UserService
  def initialize(db:, cache:, queue:, analytics:, audit_log:, encryption:)
    @db = db
    @cache = cache
    @queue = queue               # "We might need async processing someday"
    @analytics = analytics       # "We might want to track this later"
    @audit_log = audit_log       # "Compliance might require this eventually"
    @encryption = encryption     # "We might need to encrypt data at rest"
  end

  # Constructor with 6 dependencies for a service that currently just does CRUD!

  def create_user(name, email)
    encrypted = @encryption.encrypt(email)       # not needed yet!
    user = User.new(name, encrypted)
    @db.save(user)
    @cache.invalidate("users")
    @queue.publish("user.created", user.to_json) # no consumer exists yet!
    @analytics.track("user_created", name: name) # no dashboard yet!
    @audit_log.log("CREATE", "user", user.id)    # no compliance requirement yet!
    user
  end
end
```

```ruby
# GOOD: Only what's needed NOW

class UserService
  def initialize(db:)
    @db = db
  end

  def create_user(name, email)
    user = User.new(name, email)
    @db.save(user)
    user
  end

  # Add caching WHEN performance becomes an issue
  # Add analytics WHEN the business requests it
  # Add encryption WHEN compliance requires it
  # Add async processing WHEN load demands it
end
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
1. Its own object (`self`)
2. Objects passed as parameters
3. Objects it creates
4. Its direct component objects (instance variables)

It should NOT call methods on objects returned by other methods (no "train wrecks").

```ruby
# BAD: Violating Law of Demeter — reaching through objects
class Address
  attr_reader :city, :zip_code

  def initialize(city, zip_code)
    @city = city
    @zip_code = zip_code
  end
end

class Customer
  attr_reader :address

  def initialize(address)
    @address = address
  end
end

class Order
  attr_reader :customer

  def initialize(customer)
    @customer = customer
  end
end

# "Train wreck" — reaching deep into the object graph
def print_shipping_label(order)
  # BAD: order → customer → address → city (3 levels deep!)
  city = order.customer.address.city
  zip = order.customer.address.zip_code
  puts "#{city}, #{zip}"
end
```

```ruby
# GOOD: Each object provides what's needed directly

class Order
  def initialize(customer)
    @customer = customer
  end

  # Order provides shipping info directly — hides internal structure
  def shipping_city
    @customer.city
  end

  def shipping_zip
    @customer.zip_code
  end

  def shipping_label
    "#{@customer.city}, #{@customer.zip_code}"
  end
end

class Customer
  def initialize(address)
    @address = address
  end

  # Customer delegates to Address — hides Address from callers
  def city
    @address.city
  end

  def zip_code
    @address.zip_code
  end
end

def print_shipping_label(order)
  # GOOD: only talks to the Order object directly
  puts order.shipping_label
end
```

**Ruby's `delegate` and `Forwardable` for Law of Demeter:**

```ruby
require 'forwardable'

class Customer
  extend Forwardable

  def_delegators :@address, :city, :zip_code, :state

  def initialize(address)
    @address = address
  end
end

# Or with Rails:
# class Customer
#   delegate :city, :zip_code, :state, to: :address
# end
```

**Benefits:**
- Reduces coupling (caller doesn't know about Address class)
- If Address structure changes, only Customer needs updating (not all callers)
- Easier to test (mock only the immediate dependency)

**When it's OK to "violate" Law of Demeter:**
- Fluent interfaces / method chaining (builder pattern): `query.where(age: 25).order(:name).limit(10)`
- Data structures (DTOs, value objects): accessing nested data is their purpose
- Standard library: `array.first.upcase` is idiomatic Ruby

---

### Composition over Inheritance

> "Favor object composition over class inheritance." — Gang of Four

This principle advises using composition (has-a) to achieve code reuse and polymorphism rather than inheritance (is-a), because composition is more flexible and creates less coupling.

```ruby
# BAD: Deep inheritance hierarchy for behavior combinations
class Animal; end
class FlyingAnimal < Animal; end
class SwimmingAnimal < Animal; end
# What about a flying-swimming animal? Ruby doesn't have multiple inheritance!
# Mixins help but can get messy with deep hierarchies

# GOOD: Compose behaviors
class FlyBehavior
  def move
    puts "Flying through the air"
  end
end

class SwimBehavior
  def move
    puts "Swimming in water"
  end
end

class WalkBehavior
  def move
    puts "Walking on land"
  end
end

class Animal
  attr_reader :name

  def initialize(name)
    @name = name
    @movements = []
  end

  def add_movement(behavior)
    @movements << behavior
  end

  def move_all
    print "#{@name}: "
    @movements.each(&:move)
  end
end

# Any combination without class explosion!
duck = Animal.new("Duck")
duck.add_movement(FlyBehavior.new)
duck.add_movement(SwimBehavior.new)
duck.add_movement(WalkBehavior.new)
duck.move_all
```

---

### Program to an Interface, Not an Implementation

> "Depend on abstractions, not on concretions."

This principle (closely related to DIP) means that variables, parameters, and return types should reference abstract types (interfaces/modules) rather than concrete classes whenever possible.

```ruby
# BAD: Programming to implementation
def process_data(array)  # tied to Array specifically
  # What if we want to use a Set or custom collection?
end

db = MySQLDatabase.new  # tied to MySQL

# GOOD: Programming to interface (duck typing in Ruby)
def process_data(collection)  # works with any Enumerable
  collection.each { |item| puts item }
end

# In Ruby, duck typing naturally encourages this:
def save_user(repository, user)
  repository.save(user)  # works with ANY repository implementation
end

# Any object that responds to #save works:
save_user(PostgresRepo.new, user)
save_user(InMemoryRepo.new, user)
save_user(FileRepo.new, user)
```

---

### Encapsulate What Varies

> "Identify the aspects of your application that vary and separate them from what stays the same."

Find the parts of your code that are likely to change and encapsulate them behind stable interfaces. This way, changes are isolated and don't ripple through the system.

```ruby
# What varies: tax calculation rules (different per country, change frequently)
# What stays the same: the order processing flow

class USTaxCalculator
  def calculate_tax(amount, category)
    case category
    when "food" then 0.0
    when "luxury" then amount * 0.12
    else amount * 0.08
    end
  end
end

class UKTaxCalculator
  def calculate_tax(amount, category)
    case category
    when "food" then 0.0
    else amount * 0.20  # VAT
    end
  end
end

# Order processing is STABLE — doesn't change when tax rules change
class OrderProcessor
  def initialize(tax_calculator)
    @tax_calculator = tax_calculator  # the varying part is encapsulated
  end

  def calculate_total(items)
    subtotal = items.sum { |item| item.price * item.quantity }
    tax = items.sum { |item| @tax_calculator.calculate_tax(item.price * item.quantity, item.category) }
    subtotal + tax
  end
end
```

---

### Favor Object Composition over Class Inheritance

This is the Gang of Four's formulation of "Composition over Inheritance." The key insight is that inheritance creates a compile-time, static relationship that cannot change, while composition creates a runtime, dynamic relationship that can be reconfigured.

```ruby
# Inheritance: behavior is fixed at class definition time
class TimestampLogger
  def log(msg)
    puts "[#{Time.now}] #{msg}"
  end
end

class EncryptedTimestampLogger < TimestampLogger
  # What if we want encryption WITHOUT timestamp?
  # What if we want timestamp + encryption + compression?
  # Inheritance doesn't compose well!
end

# Composition: behavior can be combined freely at runtime (Decorator pattern)
class ConsoleLogger
  def log(msg)
    puts msg
  end
end

class TimestampDecorator
  def initialize(logger)
    @logger = logger
  end

  def log(msg)
    @logger.log("[#{Time.now}] #{msg}")
  end
end

class EncryptionDecorator
  def initialize(logger)
    @logger = logger
  end

  def log(msg)
    @logger.log(encrypt(msg))
  end

  private

  def encrypt(msg)
    "ENC(#{msg})"
  end
end

class CompressionDecorator
  def initialize(logger)
    @logger = logger
  end

  def log(msg)
    @logger.log(compress(msg))
  end

  private

  def compress(msg)
    "ZIP(#{msg})"
  end
end

# Compose ANY combination at runtime!
logger = TimestampDecorator.new(
  EncryptionDecorator.new(
    ConsoleLogger.new
  )
)
logger.log("Hello")  # outputs: [2024-01-01 12:00:00] ENC(Hello)

# Different combination — no new classes needed:
logger2 = CompressionDecorator.new(
  TimestampDecorator.new(
    ConsoleLogger.new
  )
)
logger2.log("Hello")  # outputs: ZIP([2024-01-01 12:00:00] Hello)
```

**Ruby-specific composition with `SimpleDelegator`:**

```ruby
require 'delegate'

class LoggerDecorator < SimpleDelegator
  # SimpleDelegator forwards all methods to the wrapped object
end

class TimestampDecorator < LoggerDecorator
  def log(msg)
    super("[#{Time.now}] #{msg}")
  end
end

class UppercaseDecorator < LoggerDecorator
  def log(msg)
    super(msg.upcase)
  end
end

base = ConsoleLogger.new
decorated = TimestampDecorator.new(UppercaseDecorator.new(base))
decorated.log("hello")  # outputs: [2024-01-01 12:00:00] HELLO
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

| Principle | Core Idea | Key Technique in Ruby |
|-----------|-----------|---------------|
| SRP | One class, one reason to change | Service objects, extract classes by responsibility |
| OCP | Open for extension, closed for modification | Inheritance, Strategy, Template Method, Blocks/Procs |
| LSP | Subtypes must be substitutable for base types | Honor contracts, duck typing discipline |
| ISP | No client depends on methods it doesn't use | Modules (mixins), duck typing, focused classes |
| DIP | Depend on abstractions, not concretions | Dependency Injection, keyword args, IoC containers |
| DRY | Single source of truth for every piece of knowledge | Extract shared logic into reusable components |
| KISS | Simplest solution that works | Avoid over-engineering, use Ruby idioms |
| YAGNI | Don't build what you don't need yet | Implement on demand, not speculation |
| Law of Demeter | Only talk to immediate friends | Delegate, Forwardable, don't chain through objects |
| Composition over Inheritance | Has-a over is-a for flexibility | Strategy, Decorator, Delegation, SimpleDelegator |

---

## Interview Tips for Module 3

1. **SRP — "What is a reason to change?"** Explain that a "reason to change" maps to a stakeholder/actor. If HR and IT both request changes to the same class, it has two responsibilities. Give the Employee example (business logic vs persistence vs presentation). In Ruby, mention service objects as a common SRP pattern.

2. **OCP — "How do you add features without modifying existing code?"** Explain using polymorphism and duck typing. The Strategy pattern is the cleanest example. Show how a new payment method or compression algorithm can be added by creating a new class without touching existing ones. In Ruby, also mention blocks/procs and modules as extension mechanisms.

3. **LSP — "What's the Rectangle-Square problem?"** Explain that Square inheriting from Rectangle violates LSP because `width=`/`height=` have different semantics. Code expecting independent width/height breaks with Square. Solutions: immutable shapes, separate hierarchies, or factory methods.

4. **LSP — "What are preconditions and postconditions?"** Preconditions cannot be strengthened (subclass can't demand more), postconditions cannot be weakened (subclass can't guarantee less). Give the Account example where a restricted account adds new restrictions that break callers.

5. **ISP — "What's wrong with a fat interface?"** Implementors are forced to provide meaningless implementations (raise or no-op). Clients depend on methods they don't use, creating unnecessary coupling. Show the Printer/Scanner/Fax example and how splitting into Printable, Scannable, Faxable modules solves it. In Ruby, emphasize that modules naturally support ISP.

6. **DIP — "What's the difference between DIP, DI, and IoC?"** DIP is the principle (depend on abstractions). DI is the technique (inject dependencies from outside). IoC is the broader pattern (framework controls flow). DIP tells you WHAT to do, DI tells you HOW to do it.

7. **DIP — "How do you test code that uses a database?"** Define a repository interface (module or duck type). In production, inject a real database implementation. In tests, inject a mock/stub that returns predefined data. The business logic doesn't know or care which one it's using. In Ruby, show keyword arguments with defaults as a clean DI pattern.

8. **Composition over Inheritance — "When would you use inheritance vs composition?"** Use inheritance for genuine "is-a" relationships where LSP holds (Dog is-a Animal). Use composition for "has-a" relationships, when behavior needs to change at runtime, or when you'd need multiple inheritance to combine features. In Ruby, modules (mixins) offer a middle ground but still add responsibility to the including class.

9. **SOLID violations in practice — "How do you identify violations?"** SRP: class has methods serving different actors. OCP: adding a feature requires modifying existing case/when chains. LSP: type checks (`is_a?`) or "unsupported operation" exceptions. ISP: empty method implementations or `raise NotImplementedError`. DIP: `ClassName.new` inside business logic instead of injection.

10. **Trade-offs — "Can you over-apply SOLID?"** Yes! Over-applying SRP creates too many tiny classes. Over-applying OCP creates premature abstractions. Over-applying DIP adds unnecessary indirection. Apply SOLID where you see or expect change, not everywhere. KISS and YAGNI balance SOLID.

11. **Real-world application — "How do SOLID principles help in system design?"** They make code testable (DIP + DI), extensible (OCP), maintainable (SRP), and flexible (LSP + ISP). In interviews, show how you'd structure a class hierarchy for a parking lot, elevator system, or payment processor using these principles.

12. **Law of Demeter — "What's a train wreck?"** `order.customer.address.city` — reaching through multiple objects. Fix by having Order expose `shipping_city` directly (or use `delegate`/`Forwardable`). Reduces coupling and makes refactoring easier.

13. **DRY vs Premature Abstraction** — Not all code that looks similar IS duplicate. Two methods with identical code that serve different purposes and change for different reasons should remain separate. DRY applies to knowledge/concepts, not just syntax.

14. **Ruby-specific patterns for SOLID:**
    - **Service Objects** (SRP): Single-purpose classes with a `call` method
    - **Form Objects** (SRP): Separate validation from persistence
    - **Presenters/Decorators** (SRP, OCP): Separate display logic from domain
    - **Policy Objects** (OCP): Encapsulate business rules
    - **Query Objects** (SRP): Encapsulate complex database queries
    - **Value Objects** (LSP): Immutable objects that are always substitutable

15. **Applying SOLID to a design problem** — In an interview, when designing a system:
    - Identify responsibilities → SRP (separate classes/service objects)
    - Identify extension points → OCP (use duck typing, modules, strategy)
    - Verify substitutability → LSP (can any subclass replace the base?)
    - Check interface size → ISP (are clients forced to depend on unused methods?)
    - Check dependency direction → DIP (does business logic depend on infrastructure?)
