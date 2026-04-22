# Module 5: Design Patterns — Structural (Ruby)

> Structural patterns deal with object composition and relationships — they describe how classes and objects can be combined to form larger structures. These patterns focus on simplifying the design by identifying simple ways to realize relationships between entities. Instead of building monolithic classes, structural patterns help you compose objects to get new functionality, adapt incompatible interfaces, and control access to objects. Ruby's dynamic nature — duck typing, modules, `method_missing`, `SimpleDelegator`, and `Forwardable` — makes many structural patterns more concise and idiomatic than in statically typed languages.

---

## 5.1 Adapter Pattern

> The Adapter pattern converts the interface of a class into another interface that clients expect. It lets classes work together that couldn't otherwise because of incompatible interfaces. Think of it like a power plug adapter — it doesn't change the electrical current, it just makes the plug fit the socket.

---

### Why Adapter?

You often encounter situations where:
- You want to use an existing class, but its interface doesn't match what you need
- You want to create a reusable class that cooperates with unrelated or unforeseen classes
- You need to integrate third-party libraries with different interfaces
- You're migrating from one API to another and need backward compatibility

**The problem without Adapter:**
```ruby
# You have a legacy XML analytics library
class XMLAnalyticsLibrary
  def analyze_xml_data(xml_data)
    puts "Analyzing XML data: #{xml_data[0, 50]}..."
  end
end

# But your new system works with JSON
class ModernDashboard
  def display_analytics(# needs JSON interface)
    # Can't use XMLAnalyticsLibrary directly — wrong interface!
    # Do we modify XMLAnalyticsLibrary? NO — we might not own it.
    # Do we modify ModernDashboard? NO — it shouldn't know about XML.
  end
end
```

---

### Structure

```
┌──────────────────┐       ┌──────────────────┐
│     Client       │       │  Target Interface │
│                  │──────→│  (what client     │
│                  │       │   expects)        │
└──────────────────┘       └──────────────────┘
                                    ▲
                                    │ implements
                           ┌──────────────────┐       ┌──────────────────┐
                           │     Adapter      │       │    Adaptee       │
                           │  (converts       │──────→│  (existing class │
                           │   interface)     │ uses  │   with wrong     │
                           └──────────────────┘       │   interface)     │
                                                      └──────────────────┘
```

---

### Object Adapter (Composition — Preferred)

The adapter holds a reference to the adaptee and delegates calls to it. This is the **preferred** approach because it uses composition over inheritance.

```ruby
# ============================================================
# TARGET INTERFACE — what the client expects (duck type)
# ============================================================

# In Ruby, the "interface" is implicit — any object responding to
# play, pause, stop, format is a valid MediaPlayer.
# We define a base class for documentation purposes only.

class MediaPlayer
  def play(filename)  = raise(NotImplementedError)
  def pause           = raise(NotImplementedError)
  def stop            = raise(NotImplementedError)
  def format          = raise(NotImplementedError)
end

# ============================================================
# ADAPTEES — existing classes with incompatible interfaces
# ============================================================

# Legacy VLC library with different method names and signatures
class VLCPlayer
  def load_media(path)
    puts "[VLC] Loading media: #{path}"
  end

  def start_playback
    puts "[VLC] Playback started"
  end

  def pause_playback
    puts "[VLC] Playback paused"
  end

  def stop_playback
    puts "[VLC] Playback stopped"
  end

  def supported_format
    "VLC (avi, mkv, mp4, flv)"
  end
end

# Another legacy library — completely different interface
class FFmpegProcessor
  def initialize
    @current_file = nil
    @playing = false
  end

  def open_file(filepath, mode)
    @current_file = filepath
    puts "[FFmpeg] Opened file: #{filepath} mode=#{mode}"
    0  # success code
  end

  def decode
    @playing = true
    puts "[FFmpeg] Decoding and playing: #{@current_file}"
    0
  end

  def suspend
    @playing = false
    puts "[FFmpeg] Suspended"
    0
  end

  def close_file
    @playing = false
    @current_file = nil
    puts "[FFmpeg] File closed"
    0
  end
end

# ============================================================
# OBJECT ADAPTERS — adapt each library to MediaPlayer interface
# ============================================================

class VLCAdapter < MediaPlayer
  def initialize
    @vlc = VLCPlayer.new  # composition — holds the adaptee
  end

  def play(filename)
    @vlc.load_media(filename)
    @vlc.start_playback
  end

  def pause
    @vlc.pause_playback
  end

  def stop
    @vlc.stop_playback
  end

  def format
    @vlc.supported_format
  end
end

class FFmpegAdapter < MediaPlayer
  def initialize
    @ffmpeg = FFmpegProcessor.new  # composition
  end

  def play(filename)
    @ffmpeg.open_file(filename, 1)  # mode 1 = read
    @ffmpeg.decode
  end

  def pause
    @ffmpeg.suspend
  end

  def stop
    @ffmpeg.close_file
  end

  def format
    "FFmpeg (mp3, wav, ogg, flac, aac)"
  end
end

# ============================================================
# CLIENT CODE — works with MediaPlayer interface only
# ============================================================

class MusicApp
  def initialize(player)
    @player = player
  end

  def play_song(filename)
    puts "Using player: #{@player.format}"
    @player.play(filename)
  end

  def pause_song
    @player.pause
  end

  def stop_song
    @player.stop
  end
end

# Usage
app1 = MusicApp.new(VLCAdapter.new)
app1.play_song("song.mkv")
app1.pause_song
app1.stop_song

puts

app2 = MusicApp.new(FFmpegAdapter.new)
app2.play_song("song.mp3")
app2.pause_song
app2.stop_song
```

**Why Object Adapter is preferred:**
- Uses composition — more flexible than inheritance
- Can adapt multiple adaptees (hold references to several)
- Adaptee can be swapped at runtime
- Works even when adaptee is frozen or otherwise non-inheritable
- Follows "Composition over Inheritance" principle

---

### Adapter Using Ruby's `Forwardable` (Idiomatic Ruby)

Ruby's `Forwardable` module provides a clean way to delegate methods — perfect for adapters:

```ruby
require 'forwardable'

class VLCForwardableAdapter
  extend Forwardable

  def initialize
    @vlc = VLCPlayer.new
  end

  # Direct delegation for methods that map 1:1
  def_delegator :@vlc, :pause_playback, :pause
  def_delegator :@vlc, :stop_playback, :stop
  def_delegator :@vlc, :supported_format, :format

  # Custom adaptation for methods that need translation
  def play(filename)
    @vlc.load_media(filename)
    @vlc.start_playback
  end
end

adapter = VLCForwardableAdapter.new
adapter.play("song.mkv")
adapter.pause
adapter.stop
puts adapter.format
```

---

### Adapter Using `SimpleDelegator` (Another Ruby Idiom)

`SimpleDelegator` wraps an object and forwards all unknown methods to it. You override only the methods you need to adapt:

```ruby
require 'delegate'

class VLCDelegatorAdapter < SimpleDelegator
  def initialize
    super(VLCPlayer.new)
  end

  def play(filename)
    load_media(filename)      # delegated to VLCPlayer
    start_playback            # delegated to VLCPlayer
  end

  def pause  = pause_playback   # delegated to VLCPlayer
  def stop   = stop_playback    # delegated to VLCPlayer
  def format = supported_format # delegated to VLCPlayer
end
```

---

### Adapter Using `method_missing` (Dynamic Adapter)

For maximum flexibility, `method_missing` can dynamically translate method calls:

```ruby
class DynamicAdapter
  def initialize(adaptee, method_map = {})
    @adaptee = adaptee
    @method_map = method_map  # { target_method => adaptee_method }
  end

  def method_missing(method_name, *args, &block)
    adaptee_method = @method_map[method_name] || method_name

    if @adaptee.respond_to?(adaptee_method)
      @adaptee.send(adaptee_method, *args, &block)
    else
      super
    end
  end

  def respond_to_missing?(method_name, include_private = false)
    adaptee_method = @method_map[method_name] || method_name
    @adaptee.respond_to?(adaptee_method) || super
  end
end

# Usage — map target methods to adaptee methods
vlc = VLCPlayer.new
adapter = DynamicAdapter.new(vlc, {
  pause: :pause_playback,
  stop:  :stop_playback,
  format: :supported_format
})

adapter.pause   # calls vlc.pause_playback
adapter.stop    # calls vlc.stop_playback
adapter.format  # calls vlc.supported_format
```

---

### Two-Way Adapter

A two-way adapter implements BOTH interfaces, allowing it to be used as either type. Useful when two systems need to interoperate in both directions.

```ruby
# Interface A — used by System A
class NewLogger
  def log_info(message)  = raise(NotImplementedError)
  def log_error(message, error_code) = raise(NotImplementedError)
  def log_debug(message) = raise(NotImplementedError)
end

# Interface B — used by System B (legacy)
class LegacyLogger
  def write_log(level, msg) = raise(NotImplementedError)
  def flush                 = raise(NotImplementedError)
end

# Two-Way Adapter — implements BOTH interfaces
class LoggerAdapter
  # Internal implementation
  def log_info(message)
    write_to_file("INFO", message)
  end

  def log_error(message, error_code)
    write_to_file("ERROR", "#{message} (code: #{error_code})")
  end

  def log_debug(message)
    write_to_file("DEBUG", message)
  end

  def write_log(level, msg)
    case level
    when 0 then log_debug(msg)
    when 1 then log_info(msg)
    when 2 then log_error(msg, -1)
    else log_info(msg)
    end
  end

  def flush
    puts "[FLUSH] All logs written"
  end

  private

  def write_to_file(prefix, message)
    puts "[#{prefix}] #{message}"
  end
end

# System A uses NewLogger interface
def new_system(logger)
  logger.log_info("New system started")
  logger.log_error("Connection failed", 503)
end

# System B uses LegacyLogger interface
def legacy_system(logger)
  logger.write_log(1, "Legacy system running")
  logger.write_log(2, "Legacy error occurred")
  logger.flush
end

# Both systems can use the same adapter instance
adapter = LoggerAdapter.new
new_system(adapter)     # uses NewLogger interface
legacy_system(adapter)  # uses LegacyLogger interface
# Same object, two interfaces — seamless interop!
```

---

### Real-World Adapter Example — Payment Gateway Integration

```ruby
# Your application's payment interface (duck type)
# Any object responding to charge, refund, transaction_status is valid

# Third-party: Stripe-like API (different interface)
class StripeAPI
  ChargeResult = Struct.new(:charge_id, :success, :error_message)

  def create_charge(amount_in_cents, currency, customer_token)
    puts "[Stripe] Charging #{amount_in_cents} cents (#{currency}) to customer #{customer_token}"
    ChargeResult.new("ch_abc123", true, "")
  end

  def reverse_charge(charge_id, amount_in_cents)
    puts "[Stripe] Reversing charge #{charge_id} for #{amount_in_cents} cents"
    true
  end

  def retrieve_charge(charge_id)
    "succeeded"
  end
end

# Third-party: PayPal-like API (completely different interface)
class PayPalSDK
  PaymentResponse = Struct.new(:payment_id, :state)

  def execute_payment(payer_id, total, currency_code)
    puts "[PayPal] Executing payment of #{total} #{currency_code} for payer #{payer_id}"
    PaymentResponse.new("PAY-xyz789", "approved")
  end

  def refund_payment(sale_id, amount, currency_code)
    puts "[PayPal] Refunding #{amount} #{currency_code} for sale #{sale_id}"
    true
  end

  def get_payment_details(payment_id)
    "approved"
  end
end

# ============================================================
# ADAPTERS — make each gateway conform to our payment interface
# ============================================================

class StripeAdapter
  def initialize
    @stripe = StripeAPI.new
  end

  def charge(customer_id, amount, currency)
    # Stripe expects cents, not dollars
    amount_in_cents = (amount * 100).to_i
    result = @stripe.create_charge(amount_in_cents, currency, customer_id)
    result.success
  end

  def refund(transaction_id, amount)
    amount_in_cents = (amount * 100).to_i
    @stripe.reverse_charge(transaction_id, amount_in_cents)
  end

  def transaction_status(transaction_id)
    @stripe.retrieve_charge(transaction_id)
  end
end

class PayPalAdapter
  def initialize
    @paypal = PayPalSDK.new
  end

  def charge(customer_id, amount, currency)
    result = @paypal.execute_payment(customer_id, amount, currency)
    result.state == "approved"
  end

  def refund(transaction_id, amount)
    @paypal.refund_payment(transaction_id, amount, "USD")
  end

  def transaction_status(transaction_id)
    @paypal.get_payment_details(transaction_id)
  end
end

# ============================================================
# CLIENT — works with any payment processor (duck typing)
# ============================================================

class CheckoutService
  def initialize(processor)
    @processor = processor
  end

  def process_order(customer_id, total)
    puts "Processing order for $#{total}"
    success = @processor.charge(customer_id, total, "USD")
    puts success ? "Payment successful!" : "Payment failed!"
    success
  end
end

# Switch payment gateways by changing one line
stripe = StripeAdapter.new
paypal = PayPalAdapter.new

checkout = CheckoutService.new(stripe)  # or: CheckoutService.new(paypal)
checkout.process_order("cust_123", 99.99)
```

---

### Adapter Using Modules (Mixin Adapter)

Ruby modules provide another way to adapt interfaces — mix in an adapter module to add the expected interface:

```ruby
module MediaPlayerAdapter
  def play(filename)
    raise NotImplementedError, "#{self.class} must implement play"
  end

  def pause
    raise NotImplementedError, "#{self.class} must implement pause"
  end

  def stop
    raise NotImplementedError, "#{self.class} must implement stop"
  end
end

class VLCPlayer
  include MediaPlayerAdapter

  # Override adapter methods to delegate to existing VLC methods
  def play(filename)
    load_media(filename)
    start_playback
  end

  alias_method :pause, :pause_playback
  alias_method :stop, :stop_playback
end
```

---

### When to Use Adapter Pattern

| Scenario | Why Adapter Helps |
|----------|-------------------|
| Integrating third-party libraries | Adapt their interface to yours |
| Legacy system migration | Wrap old API with new interface |
| Multiple implementations with different APIs | Unify under one interface |
| Testing with mocks | Adapt mock objects to expected interface |
| API versioning | Adapt old API calls to new API |
| Cross-platform compatibility | Adapt platform-specific APIs |

**When NOT to use Adapter:**
- When you can modify the adaptee's interface directly
- When the interfaces are too different (adapter becomes too complex — consider redesign)
- When a simple method alias or delegation would suffice (don't over-engineer)

**Adapter vs other patterns:**
| Pattern | Difference from Adapter |
|---------|------------------------|
| Bridge | Designed up-front to separate abstraction from implementation; Adapter retrofits |
| Decorator | Adds new behavior; Adapter converts interface without adding behavior |
| Facade | Simplifies a complex subsystem; Adapter makes one interface match another |
| Proxy | Same interface as the real object; Adapter changes the interface |

**Ruby-specific adapter approaches:**

| Approach | When to Use |
|----------|-------------|
| Class with composition | Standard approach, most explicit |
| `Forwardable` | When many methods map 1:1 |
| `SimpleDelegator` | When most methods pass through unchanged |
| `method_missing` | Dynamic/generic adapters, method mapping at runtime |
| Module mixin | When you can modify the adaptee class |
| `alias_method` | Simple method renaming within the same class |

---


## 5.2 Bridge Pattern

> The Bridge pattern decouples an abstraction from its implementation so that the two can vary independently. It achieves this by placing the abstraction and the implementation in separate class hierarchies, connected by a "bridge" (a reference from the abstraction to the implementation).

---

### Why Bridge?

Without Bridge, combining abstractions and implementations leads to a **class explosion** problem:

```ruby
# BAD: Cartesian product of shapes × rendering APIs
# Shapes: Circle, Rectangle, Triangle
# Renderers: OpenGL, DirectX, Vulkan

# Without Bridge — need N × M classes:
class OpenGLCircle; end
class OpenGLRectangle; end
class OpenGLTriangle; end
class DirectXCircle; end
class DirectXRectangle; end
class DirectXTriangle; end
class VulkanCircle; end
class VulkanRectangle; end
class VulkanTriangle; end
# 3 shapes × 3 renderers = 9 classes!
# Add one shape → 3 more classes. Add one renderer → 3 more classes.

# WITH Bridge — need N + M classes:
# 3 shapes + 3 renderers = 6 classes. Much better!
```

**The core insight:** When you have two orthogonal dimensions of variation, use Bridge to separate them into independent hierarchies.

---

### Structure

```
┌─────────────────────┐           ┌─────────────────────┐
│    Abstraction      │           │   Implementation    │
│                     │ bridge    │   (duck type)       │
│ - impl  ────────────┼──────────→│                     │
│ + operation()       │           │ + operation_impl()  │
└─────────────────────┘           └─────────────────────┘
          ▲                                 ▲
          │                                 │
┌─────────┴──────────┐          ┌──────────┴──────────┐
│ RefinedAbstraction  │          │ ConcreteImplA       │
│ + operation()       │          │ + operation_impl()  │
└─────────────────────┘          ├─────────────────────┤
                                 │ ConcreteImplB       │
                                 │ + operation_impl()  │
                                 └─────────────────────┘
```

**Key idea:** The Abstraction holds a reference to the Implementation. The abstraction delegates work to the implementation object.

---

### Complete Implementation — Shapes × Renderers

```ruby
# ============================================================
# IMPLEMENTATION HIERARCHY — how to render
# ============================================================

class Renderer
  def render_circle(x, y, radius)       = raise(NotImplementedError)
  def render_rectangle(x, y, w, h)      = raise(NotImplementedError)
  def render_triangle(x1, y1, x2, y2, x3, y3) = raise(NotImplementedError)
  def set_color(r, g, b)                = raise(NotImplementedError)
  def name                              = raise(NotImplementedError)
end

class OpenGLRenderer < Renderer
  def render_circle(x, y, radius)
    puts "[OpenGL] glDrawCircle(#{x}, #{y}, #{radius})"
  end

  def render_rectangle(x, y, w, h)
    puts "[OpenGL] glDrawRect(#{x}, #{y}, #{w}, #{h})"
  end

  def render_triangle(x1, y1, x2, y2, x3, y3)
    puts "[OpenGL] glDrawTriangle(#{x1},#{y1} #{x2},#{y2} #{x3},#{y3})"
  end

  def set_color(r, g, b)
    puts "[OpenGL] glColor3f(#{r}, #{g}, #{b})"
  end

  def name = "OpenGL"
end

class DirectXRenderer < Renderer
  def render_circle(x, y, radius)
    puts "[DirectX] D3DDrawEllipse(#{x}, #{y}, #{radius}, #{radius})"
  end

  def render_rectangle(x, y, w, h)
    puts "[DirectX] D3DFillRect(#{x}, #{y}, #{x + w}, #{y + h})"
  end

  def render_triangle(x1, y1, x2, y2, x3, y3)
    puts "[DirectX] D3DDrawPrimitive(TRIANGLE, ...)"
  end

  def set_color(r, g, b)
    hex = format("0x%02x%02x%02x", r, g, b)
    puts "[DirectX] D3DSetColor(#{hex})"
  end

  def name = "DirectX"
end

class VulkanRenderer < Renderer
  def render_circle(x, y, radius)
    puts "[Vulkan] vkCmdDrawCircle(#{x}, #{y}, #{radius})"
  end

  def render_rectangle(x, y, w, h)
    puts "[Vulkan] vkCmdDrawRect(#{x}, #{y}, #{w}, #{h})"
  end

  def render_triangle(x1, y1, x2, y2, x3, y3)
    puts "[Vulkan] vkCmdDraw(TRIANGLE_LIST, ...)"
  end

  def set_color(r, g, b)
    puts "[Vulkan] vkSetPushConstant(color, #{r}, #{g}, #{b})"
  end

  def name = "Vulkan"
end

# ============================================================
# ABSTRACTION HIERARCHY — what to render
# ============================================================

class Shape
  attr_accessor :color_r, :color_g, :color_b

  def initialize(renderer, x, y)
    @renderer = renderer  # THE BRIDGE — reference to implementation
    @x = x
    @y = y
    @color_r = 0
    @color_g = 0
    @color_b = 0
  end

  def draw       = raise(NotImplementedError)
  def resize(factor) = raise(NotImplementedError)
  def describe   = raise(NotImplementedError)

  def set_color(r, g, b)
    @color_r = r
    @color_g = g
    @color_b = b
  end

  def move_to(new_x, new_y)
    @x = new_x
    @y = new_y
  end
end

# Refined Abstractions — specific shapes
class Circle < Shape
  def initialize(renderer, x, y, radius)
    super(renderer, x, y)
    @radius = radius
  end

  def draw
    @renderer.set_color(@color_r, @color_g, @color_b)
    @renderer.render_circle(@x, @y, @radius)
  end

  def resize(factor)
    @radius *= factor
  end

  def describe
    "Circle(r=#{@radius}) at (#{@x},#{@y})"
  end
end

class Rectangle < Shape
  def initialize(renderer, x, y, width, height)
    super(renderer, x, y)
    @width = width
    @height = height
  end

  def draw
    @renderer.set_color(@color_r, @color_g, @color_b)
    @renderer.render_rectangle(@x, @y, @width, @height)
  end

  def resize(factor)
    @width *= factor
    @height *= factor
  end

  def describe
    "Rectangle(#{@width}x#{@height}) at (#{@x},#{@y})"
  end
end

class Triangle < Shape
  def initialize(renderer, x1, y1, x2, y2, x3, y3)
    super(renderer, x1, y1)
    @x2 = x2
    @y2 = y2
    @x3 = x3
    @y3 = y3
  end

  def draw
    @renderer.set_color(@color_r, @color_g, @color_b)
    @renderer.render_triangle(@x, @y, @x2, @y2, @x3, @y3)
  end

  def resize(factor)
    @x2 = @x + (@x2 - @x) * factor
    @y2 = @y + (@y2 - @y) * factor
    @x3 = @x + (@x3 - @x) * factor
    @y3 = @y + (@y3 - @y) * factor
  end

  def describe
    "Triangle at (#{@x},#{@y})"
  end
end

# ============================================================
# CLIENT CODE — mix and match shapes with renderers
# ============================================================

opengl  = OpenGLRenderer.new
directx = DirectXRenderer.new
vulkan  = VulkanRenderer.new

# Same shape, different renderers
c1 = Circle.new(opengl, 100, 100, 50)
c2 = Circle.new(directx, 100, 100, 50)
c3 = Circle.new(vulkan, 100, 100, 50)

[c1, c2, c3].each { |c| c.set_color(255, 0, 0) }

puts "=== Same circle, three renderers ==="
c1.draw  # OpenGL rendering
c2.draw  # DirectX rendering
c3.draw  # Vulkan rendering

puts "\n=== Different shapes, same renderer ==="
rect = Rectangle.new(opengl, 50, 50, 200, 100)
tri  = Triangle.new(opengl, 0, 0, 100, 0, 50, 80)

rect.set_color(0, 255, 0)
tri.set_color(0, 0, 255)

rect.draw
tri.draw

# Adding a new shape or renderer requires only ONE new class!
```

---

### Bridge vs Adapter

This is a common interview question — they look similar but have fundamentally different intents:

| Aspect | Bridge | Adapter |
|--------|--------|---------|
| **Intent** | Designed up-front to separate two hierarchies | Retrofitted to make incompatible interfaces work |
| **When applied** | During design (before code exists) | After code exists (to fix incompatibility) |
| **Relationship** | Abstraction and implementation evolve independently | Adapts existing interface to expected one |
| **Hierarchies** | Two parallel hierarchies by design | One hierarchy adapts to another |
| **Flexibility** | Both sides can vary independently | Only adapts one side |
| **Example** | Shape × Renderer (designed to be separate) | Legacy XML lib → JSON interface (retrofit) |

**Key distinction:** Bridge is a design decision. Adapter is a fix for an existing problem.

---

### Another Example — Notification System with Bridge

```ruby
# Implementation hierarchy — HOW to send
class MessageSender
  def send_message(title, body, recipient) = raise(NotImplementedError)
  def sender_type                          = raise(NotImplementedError)
end

class EmailSender < MessageSender
  def send_message(title, body, recipient)
    puts "[Email] To: #{recipient}"
    puts "  Subject: #{title}"
    puts "  Body: #{body}"
  end

  def sender_type = "Email"
end

class SMSSender < MessageSender
  def send_message(title, body, recipient)
    puts "[SMS] To: #{recipient}"
    puts "  #{title}: #{body}"
  end

  def sender_type = "SMS"
end

class SlackSender < MessageSender
  def send_message(title, body, recipient)
    puts "[Slack] Channel: #{recipient}"
    puts "  *#{title}*"
    puts "  #{body}"
  end

  def sender_type = "Slack"
end

# Abstraction hierarchy — WHAT to send
class Notification
  def initialize(sender)
    @sender = sender  # bridge to implementation
  end

  def notify(recipient) = raise(NotImplementedError)
end

class UrgentNotification < Notification
  def initialize(sender, message)
    super(sender)
    @message = message
  end

  def notify(recipient)
    @sender.send_message("🚨 URGENT", @message, recipient)
    # Urgent: also send a follow-up
    @sender.send_message("🚨 REMINDER", "Please respond ASAP: #{@message}", recipient)
  end
end

class RegularNotification < Notification
  def initialize(sender, message)
    super(sender)
    @message = message
  end

  def notify(recipient)
    @sender.send_message("Notification", @message, recipient)
  end
end

class ScheduledNotification < Notification
  def initialize(sender, message, scheduled_time)
    super(sender)
    @message = message
    @scheduled_time = scheduled_time
  end

  def notify(recipient)
    puts "[Scheduled for #{@scheduled_time}]"
    @sender.send_message("Scheduled Alert", @message, recipient)
  end
end

# Usage — any notification type × any sender
email = EmailSender.new
sms   = SMSSender.new
slack = SlackSender.new

# Urgent via Email
UrgentNotification.new(email, "Server is down!").notify("admin@company.com")

puts

# Regular via Slack
RegularNotification.new(slack, "Deployment completed").notify("#deployments")

puts

# Scheduled via SMS
ScheduledNotification.new(sms, "Meeting in 15 minutes", "2:45 PM").notify("+1-555-0123")
```

---

### Bridge with Dependency Injection (Idiomatic Ruby)

In Ruby, the Bridge pattern often looks like simple dependency injection — pass the implementation to the constructor:

```ruby
class Report
  def initialize(formatter)
    @formatter = formatter  # bridge
  end

  def generate(data)
    @formatter.format(data)
  end
end

# Implementations as lambdas (lightweight bridge)
json_formatter = ->(data) { data.to_json }
csv_formatter  = ->(data) { data.map { |row| row.join(",") }.join("\n") }
html_formatter = ->(data) { "<table>#{data.map { |r| "<tr>#{r.map { |c| "<td>#{c}</td>" }.join}</tr>" }.join}</table>" }

# Or as simple objects
class XMLFormatter
  def format(data)
    data.map { |row| "<row>#{row.map { |c| "<col>#{c}</col>" }.join}</row>" }.join("\n")
  end
end

# Mix and match
report = Report.new(XMLFormatter.new)
report.generate([["Alice", 90], ["Bob", 85]])
```

---

### When to Use Bridge Pattern

| Scenario | Why Bridge Helps |
|----------|------------------|
| Two orthogonal dimensions of variation | Avoids class explosion (N × M → N + M) |
| Want to switch implementations at runtime | Implementation is a pluggable reference |
| Both abstraction and implementation should be extensible | Independent hierarchies |
| Want to hide implementation details from clients | Client sees only abstraction |
| Platform-independent code | Abstraction = logic, Implementation = platform |

**When NOT to use Bridge:**
- Only one dimension of variation (use simple inheritance or duck typing)
- Implementation won't change (over-engineering)
- The two hierarchies are tightly coupled by nature

**Real-world examples:**
- GUI frameworks: Window (abstraction) × WindowImpl (platform-specific)
- Database drivers: Query (abstraction) × DatabaseDriver (implementation)
- Device drivers: Device (abstraction) × HardwareInterface (implementation)
- Remote APIs: Service (abstraction) × Transport (HTTP, gRPC, WebSocket)

---


## 5.3 Composite Pattern

> The Composite pattern composes objects into tree structures to represent part-whole hierarchies. It lets clients treat individual objects (leaves) and compositions of objects (composites) uniformly through a common interface.

---

### Why Composite?

Many real-world structures are naturally hierarchical (trees):
- File systems: files and directories (directories contain files and other directories)
- UI components: buttons, panels, windows (panels contain buttons and other panels)
- Organization charts: employees and departments
- Menus: menu items and sub-menus
- Arithmetic expressions: numbers and operations (operations contain sub-expressions)

**The problem without Composite:**
```ruby
# BAD: Client must distinguish between files and directories
def calculate_size(item)
  if item.is_a?(MyFile)
    item.size
  elsif item.is_a?(MyDirectory)
    total = 0
    item.children.each do |child|
      if child.is_a?(MyFile)
        total += child.size
      else
        # Recursion with type checking — messy!
        total += calculate_size(child)
      end
    end
    total
  end
end
# Type checking everywhere, hard to extend, violates OCP
```

**With Composite:** Both files and directories implement the same interface. The client calls `size` on anything — files return their size, directories recursively sum their children's sizes.

---

### Structure

```
┌─────────────────────┐
│     Component       │ ← common interface
│                     │
│ + operation()       │
│ + add(Component)    │
│ + remove(Component) │
│ + child(int)        │
└─────────────────────┘
          ▲
          │
    ┌─────┴──────────┐
    │                │
┌───┴───┐     ┌─────┴──────┐
│ Leaf  │     │ Composite  │
│       │     │            │
│ +op() │     │ -children  │
└───────┘     │ +op()      │ ← delegates to children
              │ +add()     │
              │ +remove()  │
              └────────────┘
                    │
                    │ contains
                    ▼
              ┌───────────┐
              │ Component │ (children can be Leaf or Composite)
              └───────────┘
```

---

### Complete Implementation — File System

```ruby
# ============================================================
# COMPONENT — common interface for files and directories
# ============================================================

class FileSystemComponent
  attr_reader :name

  def initialize(name)
    @name = name
  end

  # Common operations
  def size             = raise(NotImplementedError)
  def display(indent = 0) = raise(NotImplementedError)
  def count_files      = raise(NotImplementedError)
  def count_directories = raise(NotImplementedError)

  def path(parent_path = "")
    parent_path.empty? ? name : "#{parent_path}/#{name}"
  end

  # Composite operations — default implementations for leaves
  def add(component)
    raise "Cannot add to a leaf node: #{name}"
  end

  def remove(component_name)
    raise "Cannot remove from a leaf node: #{name}"
  end

  def child(index)
    raise "No children in a leaf node: #{name}"
  end

  def find(search_name)
    nil
  end

  def composite?
    false
  end
end

# ============================================================
# LEAF — individual file
# ============================================================

class FileEntry < FileSystemComponent
  attr_reader :file_size, :extension

  def initialize(name, file_size)
    super(name)
    @file_size = file_size
    @extension = File.extname(name)
  end

  def size
    @file_size
  end

  def display(indent = 0)
    puts "#{' ' * indent}📄 #{name} (#{@file_size} bytes)"
  end

  def count_files      = 1
  def count_directories = 0
end

# ============================================================
# COMPOSITE — directory containing files and subdirectories
# ============================================================

class Directory < FileSystemComponent
  def initialize(name)
    super(name)
    @children = []
  end

  # Recursively calculate total size
  def size
    @children.sum(&:size)
  end

  def display(indent = 0)
    puts "#{' ' * indent}📁 #{name}/ (#{size} bytes, #{count_files} files)"
    @children.each { |child| child.display(indent + 4) }
  end

  def count_files
    @children.sum(&:count_files)
  end

  def count_directories
    1 + @children.sum(&:count_directories)  # count self
  end

  # Composite-specific operations
  def add(component)
    @children << component
    self  # return self for chaining
  end

  def <<(component)
    add(component)
  end

  def remove(component_name)
    @children.reject! { |c| c.name == component_name }
  end

  def child(index)
    @children.fetch(index)
  end

  def composite? = true

  # Search recursively
  def find(search_name)
    @children.each do |child|
      return child if child.name == search_name
      if child.composite?
        found = child.find(search_name)
        return found if found
      end
    end
    nil
  end

  def children
    @children.dup  # return a copy to prevent external mutation
  end

  # Make it Enumerable for Ruby idioms
  include Enumerable

  def each(&block)
    @children.each(&block)
  end
end

# ============================================================
# CLIENT CODE — treats files and directories uniformly
# ============================================================

# This method works with ANY FileSystemComponent — leaf or composite
def print_size(component)
  puts "#{component.name}: #{component.size} bytes"
end

# Build a file system tree
root = Directory.new("root")

home = Directory.new("home")
user = Directory.new("user")
docs = Directory.new("documents")
pics = Directory.new("pictures")

etc  = Directory.new("etc")
var  = Directory.new("var")
logs = Directory.new("logs")

# Add files (using << operator for convenience)
docs << FileEntry.new("resume.pdf", 250_000)
docs << FileEntry.new("notes.txt", 1_200)
docs << FileEntry.new("report.docx", 450_000)

pics << FileEntry.new("photo1.jpg", 3_500_000)
pics << FileEntry.new("photo2.png", 2_100_000)

user << docs
user << pics
user << FileEntry.new(".bashrc", 500)

home << user

etc << FileEntry.new("config.yaml", 800)
etc << FileEntry.new("hosts", 200)

logs << FileEntry.new("app.log", 5_000_000)
logs << FileEntry.new("error.log", 120_000)
var << logs

root << home
root << etc
root << var

# Display entire tree
root.display

puts

# Uniform treatment — works on both files and directories
print_size(root)           # total size of everything
print_size(docs)           # total size of documents folder
print_size(docs.child(0))  # size of resume.pdf

puts
puts "Total files: #{root.count_files}"
puts "Total directories: #{root.count_directories}"

# Search
found = root.find("error.log")
if found
  print "Found: "
  found.display
end
```

---

### Composite with Enumerable (Idiomatic Ruby)

Ruby's `Enumerable` module makes Composite trees feel natural:

```ruby
class Directory < FileSystemComponent
  include Enumerable

  def each(&block)
    @children.each(&block)
  end

  # Now you get all Enumerable methods for free:
  # select, reject, map, flat_map, any?, all?, count, min_by, max_by, etc.
end

# Ruby-idiomatic operations on the tree
root = Directory.new("root")
# ... build tree ...

# Find all files larger than 1MB
large_files = docs.select { |child| child.size > 1_000_000 }

# Find all .txt files
txt_files = docs.select { |child| child.name.end_with?(".txt") }

# Get total size using inject
total = docs.inject(0) { |sum, child| sum + child.size }

# Sort children by size
sorted = docs.sort_by(&:size)
```

---

### Leaf vs Composite Nodes

| Aspect | Leaf | Composite |
|--------|------|-----------|
| Children | None | Zero or more |
| `add`/`remove` | Raises error or no-op | Manages children |
| `operation` | Performs directly | Delegates to children |
| Examples | File, MenuItem, Employee | Directory, Menu, Department |

**Design choice — where to put child management methods:**

1. **In Component (transparency):** Both Leaf and Composite have `add`/`remove`. Leaf raises at runtime. Client treats everything uniformly but risks runtime errors.

2. **In Composite only (safety):** Only Composite has `add`/`remove`. Client must check type before adding. Safer but less uniform.

```ruby
# Approach 1: Transparency (shown above)
# FileEntry#add raises — runtime error if misused

# Approach 2: Safety — use respond_to? (Ruby duck typing)
if component.respond_to?(:add)
  component.add(child)
end

# Or check composite?
component.add(child) if component.composite?
```

The GoF book favors transparency (approach 1). Modern practice often favors safety (approach 2). Ruby's `respond_to?` makes the safety approach clean and idiomatic.

---

### Another Example — UI Component Tree

```ruby
class UIComponent
  attr_reader :id, :x, :y, :width, :height

  def initialize(id, x, y, width, height)
    @id = id
    @x = x
    @y = y
    @width = width
    @height = height
  end

  def render(offset_x = 0, offset_y = 0) = raise(NotImplementedError)
  def handle_click(click_x, click_y)      = raise(NotImplementedError)
  def component_count                      = raise(NotImplementedError)

  def add(component)
    raise "Cannot add children to #{id}"
  end

  def contains_point?(px, py, offset_x = 0, offset_y = 0)
    abs_x = x + offset_x
    abs_y = y + offset_y
    px >= abs_x && px < abs_x + width &&
      py >= abs_y && py < abs_y + height
  end
end

# Leaf components
class Button < UIComponent
  attr_reader :label

  def initialize(id, x, y, w, h, label, &on_click)
    super(id, x, y, w, h)
    @label = label
    @on_click = on_click
  end

  def render(offset_x = 0, offset_y = 0)
    puts "#{' ' * ((x + offset_x) / 10)}[#{label}]"
  end

  def handle_click(click_x, click_y)
    if contains_point?(click_x, click_y)
      puts "Button '#{label}' clicked!"
      @on_click&.call
    end
  end

  def component_count = 1
end

class TextLabel < UIComponent
  attr_reader :text

  def initialize(id, x, y, w, h, text)
    super(id, x, y, w, h)
    @text = text
  end

  def render(offset_x = 0, offset_y = 0)
    puts "#{' ' * ((x + offset_x) / 10)}#{text}"
  end

  def handle_click(click_x, click_y)
    # Labels don't respond to clicks
  end

  def component_count = 1
end

# Composite component
class Panel < UIComponent
  attr_reader :title

  def initialize(id, x, y, w, h, title = "")
    super(id, x, y, w, h)
    @title = title
    @children = []
  end

  def add(component)
    @children << component
    self
  end

  def <<(component)
    add(component)
  end

  def render(offset_x = 0, offset_y = 0)
    abs_x = x + offset_x
    abs_y = y + offset_y
    puts "#{' ' * (abs_x / 10)}┌── #{title} ──┐"
    @children.each { |child| child.render(abs_x, abs_y) }
    puts "#{' ' * (abs_x / 10)}└────────────────┘"
  end

  def handle_click(click_x, click_y)
    @children.each { |child| child.handle_click(click_x, click_y) }
  end

  def component_count
    1 + @children.sum(&:component_count)
  end
end

# Usage — build a UI tree
window = Panel.new("window", 0, 0, 800, 600, "Main Window")

toolbar = Panel.new("toolbar", 0, 0, 800, 40, "Toolbar")
toolbar << Button.new("btn-new", 10, 5, 60, 30, "New")
toolbar << Button.new("btn-open", 80, 5, 60, 30, "Open")
toolbar << Button.new("btn-save", 150, 5, 60, 30, "Save")

sidebar = Panel.new("sidebar", 0, 40, 200, 560, "Sidebar")
sidebar << TextLabel.new("lbl-files", 10, 10, 180, 20, "Files:")
sidebar << Button.new("btn-file1", 10, 35, 180, 25, "document.txt")
sidebar << Button.new("btn-file2", 10, 65, 180, 25, "image.png")

content = Panel.new("content", 200, 40, 600, 560, "Content")
content << TextLabel.new("lbl-welcome", 20, 20, 560, 30, "Welcome to the editor!")

window << toolbar
window << sidebar
window << content

# Render entire UI tree with one call
window.render

puts
puts "Total components: #{window.component_count}"

# Handle click — propagates through the tree
window.handle_click(100, 15)
```

---

### When to Use Composite Pattern

| Scenario | Why Composite Helps |
|----------|---------------------|
| Tree/hierarchical structures | Natural representation of part-whole |
| Uniform treatment of leaves and composites | Client doesn't need type checks |
| Recursive structures | Operations propagate through the tree |
| Need to add new component types easily | Just implement the Component interface |

**When NOT to use Composite:**
- Structure is flat (no hierarchy) — use a simple Array
- Leaf and composite behaviors are very different — uniform interface becomes awkward
- Performance-critical code where method dispatch overhead matters

**Real-world examples:**
- File systems (files and directories)
- GUI widget trees (buttons, panels, windows)
- HTML/XML DOM trees
- Organization hierarchies
- Menu systems (items and sub-menus)
- Arithmetic expression trees
- Scene graphs in game engines
- Rails nested forms and associations

---


## 5.4 Decorator Pattern

> The Decorator pattern attaches additional responsibilities to an object dynamically. It provides a flexible alternative to subclassing for extending functionality. Decorators wrap the original object and add behavior before or after delegating to it.

---

### Why Decorator?

Inheritance is static — you choose the behavior at compile time. But what if you need to add or remove behaviors at runtime, or combine them in arbitrary ways?

```ruby
# BAD: Inheritance explosion for combinations
# Base: Coffee
# Options: Milk, Sugar, Whip, Caramel, Vanilla, Soy

# Without Decorator — need a class for every combination:
class CoffeeWithMilk; end
class CoffeeWithSugar; end
class CoffeeWithMilkAndSugar; end
class CoffeeWithMilkAndWhip; end
class CoffeeWithMilkAndSugarAndWhip; end
class CoffeeWithMilkAndSugarAndWhipAndCaramel; end
# ... 2^6 = 64 possible combinations! Unmaintainable.

# WITH Decorator:
# 1 base + 6 decorators = 7 classes. Any combination at runtime.
```

**Key insight:** Decorators have the SAME interface as the object they wrap. This means you can stack them — a decorator wrapping a decorator wrapping the original object.

---

### Structure

```
┌─────────────────────┐
│     Component       │ ← common interface (duck type)
│ + operation()       │
└─────────────────────┘
          ▲
          │
    ┌─────┴──────────────┐
    │                    │
┌───┴────────┐   ┌──────┴──────────┐
│ Concrete   │   │   Decorator     │
│ Component  │   │                 │
│ +operation │   │ - wrapped: Comp │──→ Component
└────────────┘   │ + operation()   │
                 └─────────────────┘
                          ▲
                          │
                 ┌────────┴────────┐
                 │                 │
          ┌──────┴──────┐  ┌──────┴──────┐
          │ DecoratorA  │  │ DecoratorB  │
          │ +operation()│  │ +operation()│
          └─────────────┘  └─────────────┘
```

---

### Complete Implementation — Coffee Shop

```ruby
# ============================================================
# COMPONENT — base interface (duck type)
# ============================================================

class Beverage
  def description = raise(NotImplementedError)
  def cost        = raise(NotImplementedError)
end

# ============================================================
# CONCRETE COMPONENTS — base beverages
# ============================================================

class Espresso < Beverage
  def description = "Espresso"
  def cost        = 1.99
end

class HouseBlend < Beverage
  def description = "House Blend Coffee"
  def cost        = 0.89
end

class DarkRoast < Beverage
  def description = "Dark Roast Coffee"
  def cost        = 1.49
end

class Decaf < Beverage
  def description = "Decaf Coffee"
  def cost        = 1.29
end

# ============================================================
# BASE DECORATOR — wraps a Beverage
# ============================================================

class CondimentDecorator < Beverage
  def initialize(beverage)
    @beverage = beverage  # the wrapped component
  end
end

# ============================================================
# CONCRETE DECORATORS — each adds a condiment
# ============================================================

class Milk < CondimentDecorator
  def description = "#{@beverage.description}, Milk"
  def cost        = @beverage.cost + 0.30
end

class Sugar < CondimentDecorator
  def description = "#{@beverage.description}, Sugar"
  def cost        = @beverage.cost + 0.20
end

class WhipCream < CondimentDecorator
  def description = "#{@beverage.description}, Whip Cream"
  def cost        = @beverage.cost + 0.50
end

class Caramel < CondimentDecorator
  def description = "#{@beverage.description}, Caramel"
  def cost        = @beverage.cost + 0.60
end

class Vanilla < CondimentDecorator
  def description = "#{@beverage.description}, Vanilla"
  def cost        = @beverage.cost + 0.40
end

class SoyMilk < CondimentDecorator
  def description = "#{@beverage.description}, Soy Milk"
  def cost        = @beverage.cost + 0.50
end

# ============================================================
# CLIENT CODE — stack decorators freely
# ============================================================

def print_order(beverage)
  puts "#{beverage.description} — $#{'%.2f' % beverage.cost}"
end

# Simple espresso
order1 = Espresso.new
print_order(order1)
# Espresso — $1.99

# Dark roast with milk and sugar
order2 = Sugar.new(Milk.new(DarkRoast.new))
print_order(order2)
# Dark Roast Coffee, Milk, Sugar — $1.99

# House blend with double milk, whip, and caramel
order3 = Caramel.new(WhipCream.new(Milk.new(Milk.new(HouseBlend.new))))
print_order(order3)
# House Blend Coffee, Milk, Milk, Whip Cream, Caramel — $2.59

# Decaf with soy milk and vanilla
order4 = Vanilla.new(SoyMilk.new(Decaf.new))
print_order(order4)
# Decaf Coffee, Soy Milk, Vanilla — $2.19
```

**How stacking works:**
```
order3 call chain for cost:

Caramel#cost
  → WhipCream#cost + 0.60
    → Milk#cost + 0.50
      → Milk#cost + 0.30
        → HouseBlend#cost + 0.30
          → 0.89
        = 1.19
      = 1.49
    = 1.99
  = 2.59
```

---

### Decorator Using `SimpleDelegator` (Idiomatic Ruby)

Ruby's `SimpleDelegator` is a natural fit for the Decorator pattern — it forwards all methods to the wrapped object automatically:

```ruby
require 'delegate'

class MilkDecorator < SimpleDelegator
  def description
    "#{super}, Milk"
  end

  def cost
    super + 0.30
  end
end

class SugarDecorator < SimpleDelegator
  def description
    "#{super}, Sugar"
  end

  def cost
    super + 0.20
  end
end

# Usage — super clean!
order = SugarDecorator.new(MilkDecorator.new(Espresso.new))
puts order.description  # Espresso, Milk, Sugar
puts order.cost         # 2.49
```

**Why `SimpleDelegator` works well for Decorator:**
- Automatically forwards all methods to the wrapped object
- You only override the methods you want to enhance
- `super` calls the wrapped object's method (not the parent class)
- No need for a base decorator class

---

### Decorator Using Modules (Ruby-Specific Approach)

Ruby modules can be used as decorators via `extend` — adding behavior to individual objects at runtime:

```ruby
module Milk
  def description
    "#{super}, Milk"
  end

  def cost
    super + 0.30
  end
end

module Sugar
  def description
    "#{super}, Sugar"
  end

  def cost
    super + 0.20
  end
end

module WhipCream
  def description
    "#{super}, Whip Cream"
  end

  def cost
    super + 0.50
  end
end

module Caramel
  def description
    "#{super}, Caramel"
  end

  def cost
    super + 0.60
  end
end

# Usage — extend individual objects with decorator modules
order = Espresso.new
order.extend(Milk)
order.extend(Sugar)
order.extend(WhipCream)

puts order.description  # Espresso, Whip Cream, Sugar, Milk
puts order.cost         # 2.99

# Note: modules are applied in reverse order (last extended = first called)
# This is because Ruby's method lookup goes through the singleton class
# in reverse order of extension.
```

**Module decorators vs Class decorators:**

| Aspect | Class Decorators | Module Decorators (`extend`) |
|--------|-----------------|------------------------------|
| Wrapping | Creates new wrapper objects | Modifies the original object |
| Removal | Don't wrap = don't decorate | Cannot easily un-extend |
| Order | Explicit nesting order | Reverse extension order |
| Identity | `order.is_a?(Espresso)` → false | `order.is_a?(Espresso)` → true |
| Stacking same | Wrap multiple times | Can extend same module only once |
| Best for | Standard decorator pattern | Simple, one-off decoration |

---

### Decorator vs Inheritance

| Aspect | Decorator | Inheritance |
|--------|-----------|-------------|
| When decided | Runtime | Compile time |
| Combinations | Any combination, any order | One fixed combination per class |
| Number of classes | N base + M decorators | Up to 2^M combinations |
| Adding behavior | Wrap with new decorator | Create new subclass |
| Removing behavior | Don't wrap | Can't remove (without new class) |
| Stacking | Same decorator multiple times | Not possible |
| Complexity | More objects at runtime | More classes at compile time |

---

### Another Example — Data Stream Decorators

A classic use case: wrapping I/O streams with encryption, compression, and buffering.

```ruby
# Component interface
class DataStream
  def write(data) = raise(NotImplementedError)
  def read        = raise(NotImplementedError)
end

# Concrete component — writes to a "file"
class FileStream < DataStream
  def initialize(filename)
    @filename = filename
    @buffer = ""
  end

  def write(data)
    @buffer = data
    puts "[FileStream] Writing #{data.size} bytes to #{@filename}"
  end

  def read
    puts "[FileStream] Reading from #{@filename}"
    @buffer
  end
end

# Base decorator
class StreamDecorator < DataStream
  def initialize(stream)
    @wrapped_stream = stream
  end
end

# Encryption decorator
class EncryptionDecorator < StreamDecorator
  def initialize(stream, key = 42)
    super(stream)
    @key = key
  end

  def write(data)
    puts "[Encryption] Encrypting data..."
    encrypted = encrypt(data)
    @wrapped_stream.write(encrypted)
  end

  def read
    data = @wrapped_stream.read
    puts "[Encryption] Decrypting data..."
    decrypt(data)
  end

  private

  def encrypt(data)
    data.bytes.map { |b| b ^ @key }.pack("C*")
  end

  def decrypt(data)
    encrypt(data)  # XOR is its own inverse
  end
end

# Compression decorator
class CompressionDecorator < StreamDecorator
  def write(data)
    compressed = compress(data)
    @wrapped_stream.write(compressed)
  end

  def read
    data = @wrapped_stream.read
    decompress(data)
  end

  private

  def compress(data)
    puts "[Compression] Compressing #{data.size} bytes..."
    "COMPRESSED[#{data}]"
  end

  def decompress(data)
    puts "[Compression] Decompressing..."
    if data.start_with?("COMPRESSED[") && data.end_with?("]")
      data[11..-2]
    else
      data
    end
  end
end

# Logging decorator
class LoggingDecorator < StreamDecorator
  def write(data)
    puts "[LOG] Write operation: #{data.size} bytes"
    @wrapped_stream.write(data)
    puts "[LOG] Write complete"
  end

  def read
    puts "[LOG] Read operation started"
    data = @wrapped_stream.read
    puts "[LOG] Read complete: #{data.size} bytes"
    data
  end
end

# Usage — stack decorators in any order
puts "=== Plain write ==="
plain = FileStream.new("data.txt")
plain.write("Hello, World!")

puts "\n=== Encrypted + Compressed + Logged ==="
# Build decorated stream: Logging → Encryption → Compression → File
stream = FileStream.new("secure_data.bin")
stream = CompressionDecorator.new(stream)
stream = EncryptionDecorator.new(stream)
stream = LoggingDecorator.new(stream)

stream.write("Sensitive data that needs protection")

puts "\n=== Reading back ==="
result = stream.read
puts "Result: #{result}"
```

**Decorator order matters:**
```
Write path:  Client → Logging → Encryption → Compression → File
Read path:   File → Compression → Encryption → Logging → Client

Different order = different behavior:
- Encrypt then Compress: compressed encrypted data (less compressible)
- Compress then Encrypt: encrypted compressed data (more efficient)
```

---

### Decorator with `Proc` Chaining (Functional Ruby)

For simple decorators, Ruby's functional features provide a lightweight alternative:

```ruby
# Decorators as Procs that wrap other Procs
def with_logging(operation)
  ->(data) {
    puts "[LOG] Starting operation..."
    result = operation.call(data)
    puts "[LOG] Operation complete."
    result
  }
end

def with_timing(operation)
  ->(data) {
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    result = operation.call(data)
    elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
    puts "[TIMING] Took #{elapsed.round(4)}s"
    result
  }
end

def with_retry(operation, max_retries: 3)
  ->(data) {
    retries = 0
    begin
      operation.call(data)
    rescue => e
      retries += 1
      retry if retries < max_retries
      raise
    end
  }
end

# Base operation
save_to_db = ->(data) { puts "Saving: #{data}"; "saved" }

# Stack decorators
decorated = with_logging(with_timing(with_retry(save_to_db)))
decorated.call("important data")
```

---

### Decorator vs Proxy

This is another common interview comparison:

| Aspect | Decorator | Proxy |
|--------|-----------|-------|
| **Intent** | Add new behavior/responsibilities | Control access to the object |
| **Interface** | Same as wrapped object | Same as real object |
| **Stacking** | Multiple decorators can stack | Usually single proxy |
| **Lifecycle** | Doesn't control object lifecycle | May control creation/destruction |
| **Knowledge** | Knows it wraps a component | May create the real object lazily |
| **Example** | Add logging to a stream | Lazy-load an expensive image |

---

### When to Use Decorator Pattern

| Scenario | Why Decorator Helps |
|----------|---------------------|
| Add responsibilities dynamically | Wrap at runtime, not compile time |
| Avoid class explosion from combinations | N + M instead of N × M classes |
| Need to stack behaviors | Decorators compose naturally |
| Want to add/remove features independently | Each decorator is independent |
| Extension by subclassing is impractical | Too many combinations |

**When NOT to use Decorator:**
- Only one or two fixed combinations needed (just use subclasses)
- Order of decoration doesn't matter and you need all features (use a single class)
- Performance-critical path where indirection overhead matters
- Debugging is difficult (deep decorator chains are hard to trace)

**Real-world examples:**
- Rack middleware in Rails (logging, auth, compression, CORS)
- Ruby I/O wrappers (`StringIO`, `GzipWriter`)
- ActiveSupport extensions (decorating core classes)
- Draper gem (decorating Rails models for views)
- Game character power-ups and buffs
- HTTP request/response interceptors (Faraday middleware)

---


## 5.5 Facade Pattern

> The Facade pattern provides a simplified, unified interface to a complex subsystem. It doesn't add new functionality — it just makes the subsystem easier to use by hiding its complexity behind a single class.

---

### Why Facade?

Complex subsystems often have many classes with intricate interdependencies. Clients that need to use the subsystem must understand and coordinate all these classes — even for simple tasks.

```ruby
# BAD: Client must coordinate multiple subsystem classes directly
def play_movie(filename)
  dvd = DVDPlayer.new
  amp = Amplifier.new
  proj = Projector.new
  screen = Screen.new
  lights = Lights.new
  popcorn = PopcornMaker.new

  # Client must know the exact sequence and dependencies!
  screen.down
  lights.dim(10)
  proj.on
  proj.set_input("dvd")
  proj.wide_screen_mode
  amp.on
  amp.set_volume(7)
  amp.set_input("dvd")
  dvd.on
  dvd.play(filename)
  popcorn.on
  popcorn.pop
  # 12 steps just to watch a movie! And the order matters!
end
```

**With Facade:** One method call — `home_theater.watch_movie("Inception")`.

---

### Structure

```
┌──────────────┐
│    Client    │
└──────┬───────┘
       │ uses
       ▼
┌──────────────┐
│   Facade     │ ← simplified interface
│              │
│ + simple_op  │
└──────┬───────┘
       │ coordinates
       ▼
┌─────────────────────────────────┐
│         Complex Subsystem       │
│  ┌─────┐ ┌─────┐ ┌─────┐      │
│  │ A   │ │ B   │ │ C   │      │
│  └──┬──┘ └──┬──┘ └──┬──┘      │
│     │       │       │          │
│  ┌──┴──┐ ┌──┴──┐              │
│  │ D   │ │ E   │              │
│  └─────┘ └─────┘              │
└─────────────────────────────────┘
```

---

### Complete Implementation — Home Theater System

```ruby
# ============================================================
# SUBSYSTEM CLASSES — complex, interdependent components
# ============================================================

class DVDPlayer
  def on       = puts "  DVD Player: ON"
  def off      = puts "  DVD Player: OFF"
  def play(movie) = puts "  DVD Player: Playing \"#{movie}\""
  def pause    = puts "  DVD Player: Paused"
  def stop     = puts "  DVD Player: Stopped"
  def eject    = puts "  DVD Player: Disc ejected"
end

class Amplifier
  def on       = puts "  Amplifier: ON"
  def off      = puts "  Amplifier: OFF"
  def set_volume(level) = puts "  Amplifier: Volume set to #{level}"
  def set_input(source) = puts "  Amplifier: Input set to #{source}"
  def surround_sound    = puts "  Amplifier: Surround sound ON"
  def stereo            = puts "  Amplifier: Stereo mode"
end

class Projector
  def on       = puts "  Projector: ON"
  def off      = puts "  Projector: OFF"
  def set_input(source)  = puts "  Projector: Input set to #{source}"
  def wide_screen_mode   = puts "  Projector: Widescreen mode (16:9)"
  def standard_mode      = puts "  Projector: Standard mode (4:3)"
end

class Screen
  def down = puts "  Screen: Lowering..."
  def up   = puts "  Screen: Raising..."
end

class TheaterLights
  def on       = puts "  Lights: ON (full brightness)"
  def dim(level) = puts "  Lights: Dimmed to #{level}%"
  def off      = puts "  Lights: OFF"
end

class PopcornMaker
  def on  = puts "  Popcorn Maker: ON"
  def off = puts "  Popcorn Maker: OFF"
  def pop = puts "  Popcorn Maker: Popping corn!"
end

class StreamingPlayer
  def on       = puts "  Streaming Player: ON"
  def off      = puts "  Streaming Player: OFF"
  def stream(title) = puts "  Streaming Player: Streaming \"#{title}\""
  def pause    = puts "  Streaming Player: Paused"
  def stop     = puts "  Streaming Player: Stopped"
end

# ============================================================
# FACADE — simplified interface to the home theater subsystem
# ============================================================

class HomeTheaterFacade
  def initialize(dvd:, amp:, projector:, screen:, lights:, popcorn:, streamer:)
    @dvd = dvd
    @amp = amp
    @projector = projector
    @screen = screen
    @lights = lights
    @popcorn = popcorn
    @streamer = streamer
  end

  # Simple method that coordinates the entire subsystem
  def watch_movie(movie)
    puts "=== Getting ready to watch \"#{movie}\" ==="
    @popcorn.on
    @popcorn.pop
    @lights.dim(10)
    @screen.down
    @projector.on
    @projector.set_input("dvd")
    @projector.wide_screen_mode
    @amp.on
    @amp.set_input("dvd")
    @amp.surround_sound
    @amp.set_volume(7)
    @dvd.on
    @dvd.play(movie)
    puts "=== Movie started! Enjoy! ==="
  end

  def end_movie
    puts "=== Shutting down movie theater ==="
    @dvd.stop
    @dvd.eject
    @dvd.off
    @amp.off
    @projector.off
    @screen.up
    @lights.on
    @popcorn.off
    puts "=== Theater shut down ==="
  end

  def stream_show(title)
    puts "=== Getting ready to stream \"#{title}\" ==="
    @lights.dim(20)
    @screen.down
    @projector.on
    @projector.set_input("streaming")
    @projector.wide_screen_mode
    @amp.on
    @amp.set_input("streaming")
    @amp.stereo
    @amp.set_volume(5)
    @streamer.on
    @streamer.stream(title)
    puts "=== Streaming started! ==="
  end

  def end_stream
    puts "=== Ending stream ==="
    @streamer.stop
    @streamer.off
    @amp.off
    @projector.off
    @screen.up
    @lights.on
    puts "=== Stream ended ==="
  end

  def pause_all
    puts "=== Pausing ==="
    @dvd.pause
    @lights.dim(50)
  end
end

# ============================================================
# CLIENT CODE — simple!
# ============================================================

# Create subsystem components
dvd       = DVDPlayer.new
amp       = Amplifier.new
projector = Projector.new
screen    = Screen.new
lights    = TheaterLights.new
popcorn   = PopcornMaker.new
streamer  = StreamingPlayer.new

# Create facade with keyword arguments (idiomatic Ruby)
theater = HomeTheaterFacade.new(
  dvd: dvd, amp: amp, projector: projector, screen: screen,
  lights: lights, popcorn: popcorn, streamer: streamer
)

# Client uses simple facade methods
theater.watch_movie("Inception")
puts
theater.end_movie

puts

theater.stream_show("Breaking Bad S01E01")
puts
theater.end_stream
```

---

### Reducing Coupling

The Facade reduces the number of dependencies the client has:

```
WITHOUT Facade:
Client ──→ DVDPlayer
       ──→ Amplifier
       ──→ Projector
       ──→ Screen
       ──→ Lights
       ──→ PopcornMaker
       ──→ StreamingPlayer
(7 dependencies)

WITH Facade:
Client ──→ HomeTheaterFacade ──→ all subsystem classes
(1 dependency)
```

**Important:** The Facade doesn't prevent direct access to subsystem classes. Clients can still use them directly if they need fine-grained control. The Facade is an additional layer, not a replacement.

---

### Another Example — Computer Boot Sequence

```ruby
class CPU
  def freeze  = puts "  CPU: Freezing processor"
  def jump(address) = puts "  CPU: Jumping to address 0x#{address.to_s(16)}"
  def execute = puts "  CPU: Executing instructions"
end

class Memory
  def load(address, data)
    puts "  Memory: Loading \"#{data}\" at 0x#{address.to_s(16)}"
  end

  def clear = puts "  Memory: Clearing all memory"
end

class HardDrive
  def read(sector, size)
    puts "  HardDrive: Reading #{size} bytes from sector #{sector}"
    "boot_data"
  end
end

class BIOS
  def initialize     = puts "  BIOS: Initializing hardware"
  def check_hardware = puts "  BIOS: POST (Power-On Self-Test) passed"

  def find_boot_device
    puts "  BIOS: Boot device found: /dev/sda"
    "/dev/sda"
  end
end

class GPU
  def initialize_gpu   = puts "  GPU: Initializing graphics"
  def display_splash   = puts "  GPU: Displaying boot splash screen"
  def set_resolution(w, h) = puts "  GPU: Resolution set to #{w}x#{h}"
end

# Facade
class ComputerFacade
  def initialize
    @cpu = CPU.new
    @memory = Memory.new
    @hdd = HardDrive.new
    @bios = BIOS.new
    @gpu = GPU.new
  end

  def start
    puts "=== Computer starting up ==="
    @bios.check_hardware
    @gpu.initialize_gpu
    @gpu.display_splash
    @cpu.freeze
    boot_device = @bios.find_boot_device
    boot_data = @hdd.read(0, 512)
    @memory.load(0x00, boot_data)
    @cpu.jump(0x00)
    @cpu.execute
    @gpu.set_resolution(1920, 1080)
    puts "=== Computer ready! ==="
  end

  def shutdown
    puts "=== Computer shutting down ==="
    @cpu.freeze
    @memory.clear
    @gpu.set_resolution(0, 0)
    puts "=== Computer off ==="
  end
end

# Client
computer = ComputerFacade.new
computer.start
puts
computer.shutdown
```

---

### Facade with Module (Ruby Idiom)

In Ruby, a module with class methods can serve as a lightweight facade:

```ruby
module DeploymentFacade
  class << self
    def deploy(version)
      puts "=== Deploying version #{version} ==="
      run_tests
      build_assets
      push_to_registry(version)
      update_servers(version)
      run_health_checks
      notify_team(version)
      puts "=== Deployment complete ==="
    end

    private

    def run_tests
      puts "  Running test suite..."
    end

    def build_assets
      puts "  Building and minifying assets..."
    end

    def push_to_registry(version)
      puts "  Pushing Docker image v#{version} to registry..."
    end

    def update_servers(version)
      puts "  Rolling update to v#{version} on all servers..."
    end

    def run_health_checks
      puts "  Running health checks..."
    end

    def notify_team(version)
      puts "  Notifying team: v#{version} deployed successfully"
    end
  end
end

# One-liner deployment
DeploymentFacade.deploy("2.3.1")
```

---

### Facade vs Other Patterns

| Pattern | Relationship to Facade |
|---------|----------------------|
| **Adapter** | Changes one interface to another. Facade simplifies a complex interface. |
| **Mediator** | Mediator coordinates communication between objects (they know about mediator). Facade provides a one-way simplified interface (subsystem doesn't know about facade). |
| **Singleton** | Facade is often implemented as a Singleton since usually only one facade is needed. |
| **Abstract Factory** | Can be used as a Facade for creating subsystem objects. |

---

### When to Use Facade Pattern

| Scenario | Why Facade Helps |
|----------|------------------|
| Complex subsystem with many classes | Provides simple entry point |
| Want to reduce client-subsystem coupling | Client depends only on facade |
| Need to layer a system | Each layer has a facade |
| Want to provide a simple API for a library | Hide internal complexity |
| Legacy system integration | Wrap legacy complexity |

**When NOT to use Facade:**
- Subsystem is already simple (unnecessary layer)
- Clients need full control over subsystem (facade is too restrictive)
- Facade becomes a "God class" that does everything (split into multiple facades)

**Real-world examples:**
- Rails `ActiveRecord` (facade over SQL, connection pooling, schema management)
- Bundler (facade over gem resolution, installation, loading)
- Rake tasks (facade over build steps)
- `Net::HTTP` wrappers like HTTParty or Faraday (facade over HTTP complexity)
- jQuery (`$()` is a facade over DOM manipulation)
- Operating system API (facade over hardware)

---


## 5.6 Flyweight Pattern

> The Flyweight pattern uses sharing to support large numbers of fine-grained objects efficiently. It reduces memory usage by sharing common state (intrinsic state) among multiple objects, while keeping unique state (extrinsic state) external.

---

### Why Flyweight?

When your application creates millions of similar objects, memory becomes a bottleneck:

```ruby
# BAD: Each character in a text editor is a full object
class Character
  attr_accessor :symbol,       # 1 value — unique per character
                :font_family,  # string — often shared ("Arial")
                :font_size,    # integer — often shared (12)
                :bold,         # boolean — often shared
                :italic,       # boolean — often shared
                :color_r, :color_g, :color_b  # integers — often shared

  # In Ruby, each object has ~40 bytes overhead + instance variables
  # For a 1 million character document:
  # ~200 bytes × 1,000,000 = ~200 MB just for character objects!
end

# WITH Flyweight:
# Shared state (font, size, style, color) stored once per unique combination
# Unique state (symbol, position) stored per character
# If there are only 20 unique style combinations:
# 20 × 200 bytes (shared) + 1,000,000 × 40 bytes (unique) = ~40 MB
# 5x memory reduction!
```

---

### Intrinsic vs Extrinsic State

This is the core concept of Flyweight:

| State Type | Description | Where Stored | Example |
|-----------|-------------|--------------|---------|
| **Intrinsic** | Shared, immutable, context-independent | Inside the flyweight object | Font, color, texture |
| **Extrinsic** | Unique, varies per context | Passed by client at use time | Position, character symbol |

**The split:** Move all shared state INTO the flyweight. Keep all unique state OUTSIDE (passed as parameters).

---

### Structure

```
┌──────────────┐
│    Client    │
└──────┬───────┘
       │ uses
       ▼
┌──────────────────┐         ┌──────────────────┐
│ FlyweightFactory │────────→│   Flyweight      │
│                  │ creates │ (shared state)   │
│ + get_flyweight  │         │                  │
│ - cache: hash    │         │ + operation(     │
└──────────────────┘         │   extrinsic)     │
                             └──────────────────┘
                                      ▲
                                      │
                             ┌────────┴────────┐
                             │                 │
                      ConcreteFlyweightA  ConcreteFlyweightB
```

---

### Complete Implementation — Text Editor Character Rendering

```ruby
# ============================================================
# FLYWEIGHT — shared character style (intrinsic state)
# ============================================================

class CharacterStyle
  # INTRINSIC STATE — shared among many characters
  # Frozen to ensure immutability
  attr_reader :font_family, :font_size, :bold, :italic,
              :color_r, :color_g, :color_b

  def initialize(font_family, font_size, bold, italic, color_r, color_g, color_b)
    @font_family = font_family.freeze
    @font_size   = font_size
    @bold        = bold
    @italic      = italic
    @color_r     = color_r
    @color_g     = color_g
    @color_b     = color_b
    freeze  # make the entire flyweight immutable
  end

  # Operation takes EXTRINSIC state as parameters
  def render(symbol, x, y)
    style_desc = "#{@font_family} #{@font_size}pt"
    style_desc += " BOLD" if @bold
    style_desc += " ITALIC" if @italic
    puts "Rendering '#{symbol}' at (#{x},#{y}) " \
         "[#{style_desc} rgb(#{@color_r},#{@color_g},#{@color_b})]"
  end

  # Key for caching — unique identifier for this style combination
  def cache_key
    "#{@font_family}_#{@font_size}_#{@bold ? 'B' : ''}#{@italic ? 'I' : ''}" \
    "_#{@color_r}_#{@color_g}_#{@color_b}"
  end

  def describe
    desc = "#{@font_family} #{@font_size}pt"
    desc += " Bold" if @bold
    desc += " Italic" if @italic
    desc
  end
end

# ============================================================
# FLYWEIGHT FACTORY — creates and caches flyweight objects
# ============================================================

class CharacterStyleFactory
  def initialize
    @styles = {}
  end

  def get_style(font, size, bold, italic, r, g, b)
    # Create a temporary style to get the cache key
    key = "#{font}_#{size}_#{bold ? 'B' : ''}#{italic ? 'I' : ''}_#{r}_#{g}_#{b}"

    # Return existing flyweight if available
    return @styles[key] if @styles.key?(key)

    # Create new flyweight and cache it
    style = CharacterStyle.new(font, size, bold, italic, r, g, b)
    @styles[key] = style
    puts "[Factory] Created new style: #{style.describe} (total styles: #{@styles.size})"
    style
  end

  def style_count
    @styles.size
  end

  def print_stats
    puts "=== Flyweight Factory Stats ==="
    puts "Unique styles cached: #{@styles.size}"
    @styles.each_value { |style| puts "  #{style.describe}" }
  end
end

# ============================================================
# CONTEXT — stores extrinsic state + reference to flyweight
# ============================================================

CharacterContext = Struct.new(:symbol, :x, :y, :style) do
  def render
    style.render(symbol, x, y)
  end
end

# ============================================================
# CLIENT — text editor that uses flyweights
# ============================================================

class TextEditor
  def initialize(factory)
    @characters = []
    @factory = factory
    @cursor_x = 0
    @cursor_y = 0
    @char_width = 10
    @line_height = 20
  end

  def type(text, font:, size:, bold: false, italic: false, r: 0, g: 0, b: 0)
    style = @factory.get_style(font, size, bold, italic, r, g, b)

    text.each_char do |c|
      if c == "\n"
        @cursor_x = 0
        @cursor_y += @line_height
        next
      end

      @characters << CharacterContext.new(c, @cursor_x, @cursor_y, style)
      @cursor_x += @char_width
    end
  end

  def render
    puts "=== Rendering document ==="
    @characters.each(&:render)
  end

  def print_memory_stats
    # Estimate memory usage
    # Without flyweight: each character stores full style data (~200 bytes in Ruby)
    without_flyweight = @characters.size * 200

    # With flyweight: each character stores symbol + position + reference (~40 bytes)
    with_flyweight = @characters.size * 40 + @factory.style_count * 200

    puts "=== Memory Stats ==="
    puts "Total characters: #{@characters.size}"
    puts "Unique styles: #{@factory.style_count}"
    puts "Memory without Flyweight: ~#{without_flyweight} bytes"
    puts "Memory with Flyweight: ~#{with_flyweight} bytes"
    if without_flyweight > 0
      savings = 100 - (with_flyweight * 100 / without_flyweight)
      puts "Savings: ~#{savings}%"
    end
  end
end

# Usage
factory = CharacterStyleFactory.new
editor = TextEditor.new(factory)

# Type with different styles — factory reuses existing styles
editor.type("Hello ", font: "Arial", size: 12)
editor.type("World",  font: "Arial", size: 12, bold: true)
editor.type("! ",     font: "Arial", size: 12)              # reuses first style
editor.type("This is ", font: "Arial", size: 12)            # reuses again
editor.type("important", font: "Arial", size: 12, bold: true, italic: true, r: 255)
editor.type(" text.", font: "Arial", size: 12)              # reuses

puts
editor.render

puts
editor.print_memory_stats

puts
factory.print_stats
```

---

### Another Example — Game Forest with Tree Flyweights

A classic Flyweight example — rendering a forest with millions of trees where many share the same model/texture.

```ruby
# Flyweight — shared tree type data (intrinsic state)
class TreeType
  attr_reader :name, :texture, :mesh_data, :base_color

  def initialize(name, texture, mesh_data, base_color)
    @name = name.freeze
    @texture = texture.freeze      # large texture data — shared!
    @mesh_data = mesh_data.freeze  # 3D model data — shared!
    @base_color = base_color
    puts "[TreeType] Loading '#{name}' (texture: #{texture.size} bytes, " \
         "mesh: #{mesh_data.size} bytes)"
    freeze
  end

  def draw(x, y, scale, rotation)
    puts "Drawing #{@name} at (#{x},#{y}) scale=#{scale} rot=#{rotation}°"
  end

  def memory_size
    @texture.size + @mesh_data.size + @name.size + 4  # approximate
  end
end

# Flyweight Factory
class TreeFactory
  def initialize
    @tree_types = {}
  end

  def get_tree_type(name, texture, mesh, color)
    return @tree_types[name] if @tree_types.key?(name)

    type = TreeType.new(name, texture, mesh, color)
    @tree_types[name] = type
    type
  end

  def type_count     = @tree_types.size
  def total_shared_memory = @tree_types.values.sum(&:memory_size)
end

# Context — individual tree instance (extrinsic state)
Tree = Struct.new(:x, :y, :scale, :rotation, :type) do
  def draw
    type.draw(x, y, scale, rotation)
  end
end

# Forest — manages all trees
class Forest
  def initialize
    @trees = []
    @factory = TreeFactory.new
  end

  def plant_tree(x, y, scale, rotation, name, texture, mesh, color)
    type = @factory.get_tree_type(name, texture, mesh, color)
    @trees << Tree.new(x, y, scale, rotation, type)
  end

  def render
    @trees.each(&:draw)
  end

  def print_stats
    puts "=== Forest Stats ==="
    puts "Total trees: #{@trees.size}"
    puts "Unique tree types: #{@factory.type_count}"

    shared_memory = @factory.total_shared_memory
    per_tree_memory = @trees.size * 40  # x, y, scale, rotation, reference
    total_with_flyweight = shared_memory + per_tree_memory

    # Without flyweight: each tree stores its own copy of texture/mesh
    avg_type_memory = shared_memory / [@factory.type_count, 1].max
    total_without_flyweight = @trees.size * (avg_type_memory + 40)

    puts "Shared memory (textures/meshes): #{shared_memory} bytes"
    puts "Per-tree memory: #{per_tree_memory} bytes"
    puts "Total with Flyweight: #{total_with_flyweight} bytes"
    puts "Total without Flyweight: ~#{total_without_flyweight} bytes"
  end
end

# Usage
forest = Forest.new

# Simulate large texture/mesh data
oak_texture   = "O" * 100_000   # 100KB texture
oak_mesh      = "M" * 50_000    # 50KB mesh
pine_texture  = "P" * 80_000
pine_mesh     = "M" * 40_000
birch_texture = "B" * 90_000
birch_mesh    = "M" * 45_000

# Plant 10,000 trees — only 3 tree types loaded!
srand(42)
10_000.times do
  x = rand(1000)
  y = rand(1000)
  scale = 0.5 + rand(100) / 100.0
  rotation = rand(360)

  case rand(3)
  when 0 then forest.plant_tree(x, y, scale, rotation, "Oak", oak_texture, oak_mesh, 0x228B22)
  when 1 then forest.plant_tree(x, y, scale, rotation, "Pine", pine_texture, pine_mesh, 0x006400)
  when 2 then forest.plant_tree(x, y, scale, rotation, "Birch", birch_texture, birch_mesh, 0xF5F5DC)
  end
end

forest.print_stats
# 10,000 trees but only 3 texture/mesh sets loaded!
```

---

### Flyweight with Ruby Symbols and Frozen Strings (Ruby Idiom)

Ruby has built-in flyweight mechanisms. Symbols are interned (shared) by default, and frozen string literals (enabled with `# frozen_string_literal: true`) are shared automatically.

```ruby
# Symbols are flyweights — same symbol is always the same object
:hello.object_id == :hello.object_id  # true — same object!

# Frozen strings can be shared
a = "hello".freeze
b = "hello".freeze
a.object_id == b.object_id  # true in Ruby 2.5+ with frozen string literals

# Ruby's String#-@ (unary minus) creates/returns a frozen shared copy
a = -"hello"
b = -"hello"
a.object_id == b.object_id  # true — shared!

# Integer flyweights — small integers are cached
a = 42
b = 42
a.object_id == b.object_id  # true — Ruby caches small integers
```

**Using symbols as lightweight flyweight keys:**
```ruby
class StyleRegistry
  @styles = {}

  def self.register(name, **properties)
    @styles[name] = properties.freeze
  end

  def self.get(name)
    @styles[name]
  end
end

# Register styles once
StyleRegistry.register(:heading, font: "Arial", size: 24, bold: true)
StyleRegistry.register(:body, font: "Arial", size: 12, bold: false)
StyleRegistry.register(:code, font: "Courier", size: 11, bold: false)

# Use symbols as flyweight references — zero overhead
1_000_000.times do |i|
  style = StyleRegistry.get(:body)  # same frozen hash every time
  # render character with style...
end
```

---

### Flyweight Factory

The factory is essential — it ensures flyweight objects are shared, not duplicated:

```ruby
# The factory MUST:
# 1. Check if a flyweight with the given intrinsic state already exists
# 2. Return the existing one if found
# 3. Create a new one only if not found
# 4. Store the new one for future reuse

# Common implementation: Hash (Ruby's built-in hash map)
# Key = hash of intrinsic state
# Value = flyweight object

# Without the factory, clients might create duplicate flyweights,
# defeating the purpose.
```

---

### When to Use Flyweight Pattern

| Scenario | Why Flyweight Helps |
|----------|---------------------|
| Application creates millions of similar objects | Shares common state, reduces memory |
| Objects have significant shared state | Intrinsic state stored once |
| Object identity doesn't matter | Flyweights are interchangeable |
| Memory is the bottleneck | Trades CPU (lookup) for memory |

**When NOT to use Flyweight:**
- Few objects (overhead of factory exceeds savings)
- Objects have little shared state (nothing to share)
- Object identity matters (each must be unique)
- Extrinsic state is large (savings are minimal)

**Ruby-specific notes:**
- Symbols are built-in flyweights — use them for identifiers and keys
- `freeze` makes objects immutable, which is essential for safe sharing
- `String#-@` (dedup) creates shared frozen strings
- Ruby's `ObjectSpace` can help measure actual memory usage
- Consider using `Struct` for lightweight context objects (extrinsic state)

**Real-world examples:**
- Text editors (character formatting)
- Game engines (particle systems, tile maps, sprite rendering)
- Ruby symbols (interned strings)
- Integer caching in Ruby (small integers share object IDs)
- Icon/image caching in GUIs
- Database connection metadata
- CSS style sharing in browsers

**Flyweight vs other patterns:**
| Pattern | Difference from Flyweight |
|---------|--------------------------|
| Singleton | One instance total; Flyweight has one instance per unique state |
| Object Pool | Reuses mutable objects; Flyweight shares immutable state |
| Prototype | Clones objects; Flyweight shares objects (no cloning) |
| Composite | Often used together — Composite tree with Flyweight leaves |

---


## 5.7 Proxy Pattern

> The Proxy pattern provides a surrogate or placeholder for another object to control access to it. The proxy has the same interface as the real object, so the client doesn't know (or care) whether it's talking to the real object or the proxy.

---

### Why Proxy?

Sometimes you need to control how and when an object is accessed:
- **Lazy initialization:** Don't create an expensive object until it's actually needed
- **Access control:** Check permissions before allowing operations
- **Logging/Auditing:** Record all access to an object
- **Caching:** Return cached results instead of hitting the real object every time
- **Remote access:** Represent an object that lives on another server

```ruby
# BAD: Loading a 500MB image just to display its filename
class HugeImage
  def initialize(path)
    # Loads 500MB into memory immediately!
    load_from_disk(path)  # takes 5 seconds
  end
end

# User opens a folder with 1000 images
# Without proxy: loads 500GB into memory, takes 83 minutes
# With proxy: loads nothing until user actually views an image
```

---

### Types of Proxy

| Type | Purpose | Example |
|------|---------|---------|
| **Virtual Proxy** | Lazy initialization of expensive objects | Image proxy that loads on first display |
| **Protection Proxy** | Access control based on permissions | Admin-only operations |
| **Remote Proxy** | Represent an object in a different address space | RPC stub, network proxy |
| **Caching Proxy** | Cache results of expensive operations | Database query cache |
| **Logging Proxy** | Log all operations on the real object | Audit trail |
| **Smart Reference** | Additional actions on access (ref counting, locking) | Thread-safe wrappers |

---

### Structure

```
┌──────────────┐       ┌──────────────────┐
│    Client    │──────→│  Subject         │ ← common interface
└──────────────┘       │  (Interface)     │
                       │ + request()      │
                       └──────────────────┘
                                ▲
                                │
                       ┌────────┴────────┐
                       │                 │
               ┌───────┴──────┐  ┌───────┴──────┐
               │  RealSubject │  │    Proxy     │
               │              │  │              │
               │ + request()  │  │ - real: Real │──→ RealSubject
               └──────────────┘  │ + request()  │
                                 └──────────────┘
```

---

### Virtual Proxy — Lazy Loading

The most common type. Defers creation of an expensive object until it's actually needed.

```ruby
# Subject interface (duck typing — implicit in Ruby)
module Image
  def display
    raise NotImplementedError
  end

  def width
    raise NotImplementedError
  end

  def height
    raise NotImplementedError
  end

  def filename
    raise NotImplementedError
  end

  def memory_usage
    raise NotImplementedError
  end
end

# Real Subject — expensive to create
class HighResImage
  include Image

  attr_reader :filename, :width, :height

  def initialize(fname)
    @filename = fname
    puts "[HighResImage] Loading '#{fname}' from disk..."
    sleep(0.1)  # simulate expensive loading
    @width = 3840
    @height = 2160
    @pixel_data = Array.new(@width * @height * 4, 0)  # RGBA
    puts "[HighResImage] Loaded! (#{@pixel_data.size / (1024 * 1024)} MB)"
  end

  def display
    puts "[HighResImage] Displaying '#{@filename}' (#{@width}x#{@height})"
  end

  def memory_usage
    @pixel_data.size
  end
end

# Virtual Proxy — lazy loads the real image
class ImageProxy
  include Image

  def initialize(fname)
    @filename = fname
    @real_image = nil
    # Only store the filename — don't load the image!
    puts "[ImageProxy] Created proxy for '#{fname}' (no data loaded)"
  end

  def display
    ensure_loaded  # load only when actually displaying
    @real_image.display
  end

  def width
    ensure_loaded
    @real_image.width
  end

  def height
    ensure_loaded
    @real_image.height
  end

  def filename
    @filename  # no need to load for this!
  end

  def memory_usage
    return 0 unless loaded?
    @real_image.memory_usage
  end

  def loaded?
    !@real_image.nil?
  end

  private

  def ensure_loaded
    @real_image ||= HighResImage.new(@filename)
  end
end

# Client code — doesn't know it's using a proxy
def show_gallery(images)
  puts "=== Image Gallery ==="
  images.each do |img|
    puts "  #{img.filename} (memory: #{img.memory_usage} bytes)"
  end

  # Only load the image the user clicks on
  puts "\nUser clicks on image 2:"
  images[1].display  # only THIS image gets loaded!
end

# Create proxies for 5 images — none are loaded yet!
gallery = [
  ImageProxy.new("sunset.jpg"),
  ImageProxy.new("mountain.jpg"),
  ImageProxy.new("ocean.jpg"),
  ImageProxy.new("forest.jpg"),
  ImageProxy.new("city.jpg")
]

show_gallery(gallery)
# Only mountain.jpg is loaded — the other 4 are still just proxies!
```

---

### Virtual Proxy with method_missing (Ruby Idiom)

Ruby's `method_missing` creates transparent proxies that forward all method calls to the real object automatically:

```ruby
class LazyProxy
  def initialize(&creator)
    @creator = creator
    @real_object = nil
  end

  def method_missing(method_name, *args, &block)
    ensure_loaded
    @real_object.send(method_name, *args, &block)
  end

  def respond_to_missing?(method_name, include_private = false)
    ensure_loaded
    @real_object.respond_to?(method_name, include_private)
  end

  private

  def ensure_loaded
    @real_object ||= @creator.call
  end
end

# Usage — pass a block that creates the expensive object
image = LazyProxy.new { HighResImage.new("huge_photo.jpg") }

# No loading yet!
puts "Proxy created, nothing loaded"

# First method call triggers loading
image.display  # NOW it loads
image.width    # already loaded, no delay
```

---

### Protection Proxy — Access Control

Controls access to the real object based on permissions.

```ruby
# Real Subject
class Document
  attr_reader :name

  def initialize(name, content)
    @name = name
    @content = content
  end

  def read
    @content
  end

  def write(content)
    @content = content
    puts "[Document] '#{@name}' updated"
  end

  def delete
    @content = nil
    puts "[Document] '#{@name}' deleted"
  end
end

# User with roles
class User
  ROLES = { viewer: 0, editor: 1, admin: 2 }.freeze

  attr_reader :name, :role

  def initialize(name, role)
    @name = name
    @role = role
  end

  def role_level
    ROLES[@role] || 0
  end

  def role_name
    @role.to_s.capitalize
  end
end

# Protection Proxy — checks permissions before delegating
class DocumentProxy
  def initialize(document, current_user)
    @document = document
    @current_user = current_user
  end

  def name
    @document.name
  end

  def read
    check_permission!("read", :viewer)
    puts "[Proxy] #{@current_user.name} reading '#{@document.name}'"
    @document.read
  end

  def write(content)
    check_permission!("write", :editor)
    puts "[Proxy] #{@current_user.name} writing to '#{@document.name}'"
    @document.write(content)
  end

  def delete
    check_permission!("delete", :admin)
    puts "[Proxy] #{@current_user.name} deleting '#{@document.name}'"
    @document.delete
  end

  private

  def check_permission!(operation, min_role)
    min_level = User::ROLES[min_role] || 0
    if @current_user.role_level < min_level
      raise "ACCESS DENIED: User '#{@current_user.name}' " \
            "(#{@current_user.role_name}) cannot #{operation} " \
            "document '#{@document.name}'"
    end
  end
end

# Usage
viewer = User.new("Alice", :viewer)
editor = User.new("Bob", :editor)
admin  = User.new("Charlie", :admin)

doc = Document.new("Secret Plan", "Top secret content...")

# Viewer can read
viewer_proxy = DocumentProxy.new(doc, viewer)
puts viewer_proxy.read

# Viewer cannot write
begin
  viewer_proxy.write("Hacked!")
rescue => e
  puts e.message
end

# Editor can write
editor_proxy = DocumentProxy.new(doc, editor)
editor_proxy.write("Updated content")

# Editor cannot delete
begin
  editor_proxy.delete
rescue => e
  puts e.message
end

# Admin can do everything
admin_proxy = DocumentProxy.new(doc, admin)
admin_proxy.write("Final version")
admin_proxy.delete
```

---

### Caching Proxy

Caches results of expensive operations to avoid redundant computation.

```ruby
# Subject interface
module WeatherService
  def get_weather(city)
    raise NotImplementedError
  end

  def get_temperature(city)
    raise NotImplementedError
  end
end

# Real Subject — makes expensive API calls
class RealWeatherService
  include WeatherService

  def get_weather(city)
    puts "[API] Fetching weather for #{city} (slow network call)..."
    sleep(0.05)  # simulate latency
    "Sunny, 25°C"
  end

  def get_temperature(city)
    puts "[API] Fetching temperature for #{city} (slow network call)..."
    sleep(0.05)
    25.0
  end
end

# Caching Proxy
class CachingWeatherProxy
  include WeatherService

  CacheEntry = Struct.new(:data, :timestamp)

  def initialize(real_service, ttl: 300)
    @real_service = real_service
    @weather_cache = {}
    @temp_cache = {}
    @ttl = ttl  # time-to-live in seconds
    @cache_hits = 0
    @cache_misses = 0
  end

  def get_weather(city)
    entry = @weather_cache[city]
    if entry && cache_valid?(entry.timestamp)
      @cache_hits += 1
      puts "[Cache HIT] Weather for #{city}"
      return entry.data
    end

    @cache_misses += 1
    puts "[Cache MISS] Weather for #{city}"
    result = @real_service.get_weather(city)
    @weather_cache[city] = CacheEntry.new(result, Time.now)
    result
  end

  def get_temperature(city)
    entry = @temp_cache[city]
    if entry && cache_valid?(entry.timestamp)
      @cache_hits += 1
      puts "[Cache HIT] Temperature for #{city}"
      return entry.data
    end

    @cache_misses += 1
    puts "[Cache MISS] Temperature for #{city}"
    result = @real_service.get_temperature(city)
    @temp_cache[city] = CacheEntry.new(result, Time.now)
    result
  end

  def clear_cache
    @weather_cache.clear
    @temp_cache.clear
    puts "[Cache] Cleared"
  end

  def print_stats
    total = @cache_hits + @cache_misses
    puts "=== Cache Stats ==="
    puts "Hits: #{@cache_hits}, Misses: #{@cache_misses}"
    puts "Hit rate: #{total > 0 ? (@cache_hits * 100 / total) : 0}%" if total > 0
  end

  private

  def cache_valid?(timestamp)
    (Time.now - timestamp) < @ttl
  end
end

# Usage
real_service = RealWeatherService.new
weather = CachingWeatherProxy.new(real_service, ttl: 60)

# First call — cache miss, hits the real API
puts weather.get_weather("London")

# Second call — cache hit, instant!
puts weather.get_weather("London")

# Different city — cache miss
puts weather.get_weather("Tokyo")

# London again — still cached
puts weather.get_weather("London")

weather.print_stats
```

---

### Logging Proxy

Records all operations for auditing.

```ruby
module DatabaseService
  def query(sql)
    raise NotImplementedError
  end

  def execute(sql)
    raise NotImplementedError
  end
end

class RealDatabase
  include DatabaseService

  def query(sql)
    puts "[DB] Executing query: #{sql}"
    "result_data"
  end

  def execute(sql)
    puts "[DB] Executing: #{sql}"
    1  # rows affected
  end
end

class LoggingDatabaseProxy
  include DatabaseService

  def initialize(real_db)
    @real_db = real_db
    @audit_log = []
  end

  def query(sql)
    log_entry = "#{Time.now} | QUERY | #{sql}"
    @audit_log << log_entry
    puts "[LOG] #{log_entry}"

    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    result = @real_db.query(sql)
    duration = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time) * 1000).round(2)

    puts "[LOG] Query completed in #{duration}ms"
    result
  end

  def execute(sql)
    log_entry = "#{Time.now} | EXECUTE | #{sql}"
    @audit_log << log_entry
    puts "[LOG] #{log_entry}"

    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    result = @real_db.execute(sql)
    duration = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time) * 1000).round(2)

    puts "[LOG] Execution completed in #{duration}ms, #{result} rows affected"
    result
  end

  def print_audit_log
    puts "=== Audit Log ==="
    @audit_log.each { |entry| puts "  #{entry}" }
    puts "Total operations: #{@audit_log.size}"
  end
end

# Usage
db = RealDatabase.new
logged_db = LoggingDatabaseProxy.new(db)

logged_db.query("SELECT * FROM users WHERE active = true")
logged_db.execute("UPDATE users SET last_login = NOW() WHERE id = 42")
logged_db.query("SELECT COUNT(*) FROM orders")

puts
logged_db.print_audit_log
```

---

### Generic Proxy with method_missing (Ruby Idiom)

Ruby's dynamic nature allows building generic proxies that work with any object:

```ruby
class GenericLoggingProxy
  def initialize(target, logger: $stdout)
    @target = target
    @logger = logger
  end

  def method_missing(method_name, *args, &block)
    @logger.puts "[PROXY] Calling #{@target.class}##{method_name}(#{args.map(&:inspect).join(', ')})"

    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    result = @target.send(method_name, *args, &block)
    duration = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time) * 1000).round(2)

    @logger.puts "[PROXY] #{method_name} returned #{result.inspect} (#{duration}ms)"
    result
  end

  def respond_to_missing?(method_name, include_private = false)
    @target.respond_to?(method_name, include_private) || super
  end
end

# Works with ANY object — no interface needed!
array = GenericLoggingProxy.new([1, 2, 3])
array.push(4)
array.length
array.include?(2)

string = GenericLoggingProxy.new("hello world")
string.upcase
string.split(" ")
```

---

### Proxy with SimpleDelegator (Ruby Standard Library)

Ruby's `SimpleDelegator` is perfect for building proxies:

```ruby
require 'delegate'

class AccessControlProxy < SimpleDelegator
  def initialize(target, user)
    super(target)
    @user = user
  end

  def write(content)
    unless @user.role_level >= User::ROLES[:editor]
      raise "Access denied: #{@user.name} cannot write"
    end
    super
  end

  def delete
    unless @user.role_level >= User::ROLES[:admin]
      raise "Access denied: #{@user.name} cannot delete"
    end
    super
  end

  # read is automatically delegated — no restriction
end

doc = Document.new("Report", "Some content")
viewer = User.new("Alice", :viewer)

proxy = AccessControlProxy.new(doc, viewer)
puts proxy.read    # works — delegated automatically
puts proxy.name    # works — delegated automatically

begin
  proxy.write("new content")  # raises — access denied
rescue => e
  puts e.message
end
```

---

### Proxy vs Decorator

| Aspect | Proxy | Decorator |
|--------|-------|-----------|
| **Intent** | Control access to an object | Add responsibilities to an object |
| **Interface** | Same as real subject | Same as component |
| **Lifecycle** | May control creation of real object | Doesn't control wrapped object's lifecycle |
| **Stacking** | Usually single proxy | Multiple decorators stack |
| **Knowledge** | Knows the concrete real subject | Knows only the component interface |
| **Creation** | Proxy often creates the real object | Decorator receives the object to wrap |
| **Example** | Lazy-load image, access control | Add encryption to a stream |

**Key distinction:** Proxy controls access. Decorator adds behavior.

---

### Combining Proxy Types

In practice, a single proxy often combines multiple proxy behaviors:

```ruby
class SmartDocumentProxy
  def initialize(filename, user)
    @filename = filename
    @user = user
    @real_doc = nil       # virtual proxy (lazy)
    @cache = {}           # caching proxy
    @access_log = []      # logging proxy
  end

  def name
    @filename
  end

  def read
    check_access!("read")       # protection proxy
    ensure_loaded!              # virtual proxy

    if @cache.key?(:content)
      puts "[Proxy] Cache hit"
      return @cache[:content]   # caching proxy
    end

    content = @real_doc.read
    @cache[:content] = content
    content
  end

  def write(content)
    check_access!("write")
    ensure_loaded!
    @cache.clear  # invalidate cache
    @real_doc.write(content)
  end

  def delete
    check_access!("delete")
    ensure_loaded!
    @cache.clear
    @real_doc.delete
  end

  def access_log
    @access_log.dup
  end

  private

  def ensure_loaded!
    return if @real_doc
    puts "[Proxy] Lazy loading document..."
    @real_doc = Document.new(@filename, "loaded content")
  end

  def check_access!(operation)
    @access_log << "#{@user.name} #{operation} #{@filename} at #{Time.now}"
    if @user.role_level < User::ROLES[:viewer]
      raise "Access denied"
    end
  end
end
```

---

### When to Use Proxy Pattern

| Proxy Type | Scenario | Why It Helps |
|-----------|----------|--------------|
| Virtual | Expensive object creation | Defer creation until needed |
| Protection | Access control needed | Check permissions transparently |
| Remote | Object on another server | Hide network communication |
| Caching | Expensive repeated operations | Avoid redundant computation |
| Logging | Audit trail needed | Record all access transparently |
| Smart Reference | Resource management | Automatic cleanup |

**When NOT to use Proxy:**
- Object is cheap to create (virtual proxy adds unnecessary complexity)
- No access control needed (protection proxy is overhead)
- Direct access is fine (proxy adds indirection)
- Performance-critical path (proxy adds a layer of indirection)

**Ruby-specific notes:**
- `method_missing` enables transparent proxies that forward all calls
- `SimpleDelegator` and `Delegator` are built-in proxy/delegation support
- `BasicObject` is the ideal base class for transparent proxies (minimal methods)
- Ruby's `Forwardable` module provides selective delegation
- Blocks/Procs can serve as lightweight lazy-loading proxies

**Real-world examples:**
- ActiveRecord associations (lazy loading via proxy)
- DRb (Distributed Ruby) — remote proxy
- Rack middleware — logging/caching proxy
- Rails `ActionController::Parameters` — protection proxy for params
- Memoization gems — caching proxy

---


## Summary & Key Takeaways

| Pattern | Intent | Key Mechanism | When to Use |
|---------|--------|---------------|-------------|
| **Adapter** | Convert one interface to another | Wraps adaptee, translates calls | Integrating incompatible interfaces |
| **Bridge** | Decouple abstraction from implementation | Two separate hierarchies connected by reference | Two dimensions of variation |
| **Composite** | Tree structures with uniform treatment | Leaf and Composite share interface | Part-whole hierarchies |
| **Decorator** | Add responsibilities dynamically | Wraps component, adds behavior | Flexible combinations of features |
| **Facade** | Simplify complex subsystem | Single class coordinates subsystem | Reducing client-subsystem coupling |
| **Flyweight** | Share state to save memory | Intrinsic (shared) vs Extrinsic (unique) state | Millions of similar objects |
| **Proxy** | Control access to an object | Same interface, intercepts calls | Lazy loading, access control, caching |

---

## Ruby-Specific Structural Pattern Techniques

| Ruby Feature | Pattern Application | Example |
|-------------|---------------------|---------|
| **Duck typing** | Adapter becomes implicit — no formal interface needed | Any object with matching methods works |
| **`method_missing`** | Dynamic Adapter, transparent Proxy | Forward/translate calls at runtime |
| **Modules/Mixins** | Two-way Adapter, Decorator via `extend` | Include multiple interfaces, per-object decoration |
| **`SimpleDelegator`** | Decorator, Proxy | Built-in delegation with selective override |
| **Blocks/Procs** | Bridge implementation, Facade DSL | Lightweight implementations without classes |
| **`Enumerable`** | Composite iteration | Free `select`, `map`, `find` on tree structures |
| **`freeze`** | Flyweight immutability | Ensure shared state can't be mutated |
| **Symbols** | Built-in Flyweight | Interned, shared, immutable identifiers |
| **`Forwardable`** | Selective Proxy/Adapter | Delegate specific methods to another object |
| **Open classes** | Last-resort Adapter | Add methods directly to existing classes |

---

## Relationships Between Structural Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│                    STRUCTURAL PATTERNS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Adapter ──── similar to but different intent ──→ Bridge        │
│          └── similar interface to ──────────────→ Decorator     │
│          └── similar interface to ──────────────→ Proxy         │
│                                                                 │
│  Bridge ──── separates abstraction/impl ────────→ (design-time)│
│  Adapter ─── makes things work together ────────→ (after-the-  │
│                                                     fact)       │
│                                                                 │
│  Composite ── often used with ──────────────────→ Decorator    │
│           └── leaf nodes can be ────────────────→ Flyweight    │
│           └── traversal uses ───────────────────→ Iterator     │
│                                                                 │
│  Decorator ── wraps to add behavior ────────────→ (additive)   │
│  Proxy ────── wraps to control access ──────────→ (restrictive)│
│                                                                 │
│  Facade ──── simplifies ────────────────────────→ Subsystem    │
│          └── often is a ────────────────────────→ Singleton    │
│                                                                 │
│  Flyweight ── factory manages ──────────────────→ Sharing      │
│           └── often combined with ──────────────→ Composite    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to tell them apart (interview favorite):**
- **Adapter** makes things work that weren't designed to work together
- **Bridge** is designed from the start to let abstraction and implementation vary
- **Composite** lets you treat a group of objects as a single object
- **Decorator** adds new behavior without changing the interface
- **Facade** provides a simpler interface to a complex system
- **Flyweight** shares objects to save memory
- **Proxy** controls access to an object

---

## Interview Tips for Module 5

1. **Adapter: Duck Typing Advantage** — In Ruby, duck typing often eliminates the need for Adapter entirely. If two objects respond to the same methods, they're already compatible. Mention `method_missing` for dynamic adapters and modules for two-way adapters. Know the difference between Object Adapter (composition, preferred) and Class Adapter (inheritance).

2. **Adapter vs Bridge** — This is the #1 structural pattern comparison question. Adapter is a retrofit (makes existing incompatible things work). Bridge is a design decision (separates two dimensions from the start). Use the "designed up-front vs after-the-fact" distinction.

3. **Bridge: Class explosion** — Be ready to explain the N × M problem. Without Bridge, combining N abstractions with M implementations requires N × M classes. With Bridge, you need N + M. In Ruby, blocks/procs can serve as lightweight implementations for simple cases.

4. **Composite: Enumerable** — In Ruby, including `Enumerable` in your Composite gives you powerful iteration for free. Know the transparency vs safety trade-off for child management methods. Ruby's `respond_to?` makes the safety approach clean and idiomatic.

5. **Decorator: Module extend** — Ruby's `extend` with modules is the most idiomatic way to implement Decorator. It decorates individual objects without wrapper classes, and `super` naturally chains through the module hierarchy. Also know `SimpleDelegator` from the standard library.

6. **Decorator vs Inheritance** — Decorator is runtime, flexible, combinatorial. Inheritance is definition-time, fixed, leads to class explosion. Use Decorator when you need arbitrary combinations of features. In Ruby, `extend` with modules is the cleanest approach.

7. **Facade: Not a restriction** — Emphasize that Facade doesn't hide the subsystem — clients can still access subsystem classes directly. Facade is an additional convenience layer, not a wall. In Ruby, keyword arguments make facade constructors self-documenting.

8. **Flyweight: Intrinsic vs Extrinsic** — This is the core concept. Intrinsic state is shared and immutable (stored in flyweight). Extrinsic state is unique and varies (passed by client). The split determines memory savings. In Ruby, mention symbols as built-in flyweights and `freeze` for immutability.

9. **Proxy types** — Know all five types (Virtual, Protection, Remote, Caching, Logging). Be ready to implement at least Virtual and Protection proxies. In Ruby, `method_missing` enables transparent proxies, and `SimpleDelegator` provides built-in delegation.

10. **Proxy vs Decorator** — Another top comparison. Proxy controls access (same object, controlled). Decorator adds behavior (enhanced object). Proxy often creates the real object; Decorator receives it. Proxy is usually single; Decorators stack.

11. **Composite + Decorator** — These patterns combine naturally. A Composite tree where nodes can be decorated. Example: UI components (Composite tree) with borders and scrollbars (Decorators).

12. **Flyweight + Composite** — Another natural combination. Composite tree with Flyweight leaf nodes. Example: Document with shared character styles (Flyweight) in a paragraph tree (Composite).

13. **Real-world Ruby examples** — For each pattern, know at least 2-3 real-world examples:
    - Adapter: Payment gateways, legacy system wrappers, third-party gem integration
    - Bridge: Cross-platform UI, database drivers, notification systems
    - Composite: File systems, UI trees, HTML DOM, Rails nested forms
    - Decorator: Rack middleware, `SimpleDelegator`, ActiveSupport concerns
    - Facade: ActiveRecord, `Net::HTTP.get`, Bundler, Rails generators
    - Flyweight: Ruby symbols, frozen strings, integer caching, sprite sheets
    - Proxy: ActiveRecord lazy loading, DRb, `ActionController::Parameters`, memoization

14. **Ruby idioms** — Know when Ruby's dynamic features replace classical patterns:
    - Duck typing can replace Adapter
    - `method_missing` can replace Proxy and dynamic Adapter
    - `extend` with modules can replace Decorator
    - Blocks/Procs can replace Bridge implementations
    - Symbols and `freeze` are built-in Flyweight mechanisms
    - `SimpleDelegator` and `Forwardable` provide built-in delegation