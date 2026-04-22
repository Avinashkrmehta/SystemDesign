# Module 5: Design Patterns — Structural

> Structural patterns deal with object composition and relationships — they describe how classes and objects can be combined to form larger structures. These patterns focus on simplifying the design by identifying simple ways to realize relationships between entities. Instead of building monolithic classes, structural patterns help you compose objects to get new functionality, adapt incompatible interfaces, and control access to objects.

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
```cpp
// You have a legacy XML analytics library
class XMLAnalyticsLibrary {
public:
    void analyzeXMLData(const string& xmlData) {
        cout << "Analyzing XML data: " << xmlData.substr(0, 50) << "..." << endl;
    }
};

// But your new system works with JSON
class ModernDashboard {
public:
    void displayAnalytics(/* needs JSON interface */) {
        // Can't use XMLAnalyticsLibrary directly — wrong interface!
        // Do we modify XMLAnalyticsLibrary? NO — we might not own it.
        // Do we modify ModernDashboard? NO — it shouldn't know about XML.
    }
};
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

```cpp
// ============================================================
// TARGET INTERFACE — what the client expects
// ============================================================
class MediaPlayer {
public:
    virtual void play(const string& filename) = 0;
    virtual void pause() = 0;
    virtual void stop() = 0;
    virtual string getFormat() const = 0;
    virtual ~MediaPlayer() = default;
};

// ============================================================
// ADAPTEE — existing class with incompatible interface
// ============================================================

// Legacy VLC library with different method names and signatures
class VLCPlayer {
public:
    void loadMedia(const string& path) {
        cout << "[VLC] Loading media: " << path << endl;
    }
    void startPlayback() {
        cout << "[VLC] Playback started" << endl;
    }
    void pausePlayback() {
        cout << "[VLC] Playback paused" << endl;
    }
    void stopPlayback() {
        cout << "[VLC] Playback stopped" << endl;
    }
    string getSupportedFormat() const {
        return "VLC (avi, mkv, mp4, flv)";
    }
};

// Another legacy library — completely different interface
class FFmpegProcessor {
    string currentFile;
    bool playing = false;
public:
    int openFile(const char* filepath, int mode) {
        currentFile = filepath;
        cout << "[FFmpeg] Opened file: " << filepath << " mode=" << mode << endl;
        return 0;  // success code
    }
    int decode() {
        playing = true;
        cout << "[FFmpeg] Decoding and playing: " << currentFile << endl;
        return 0;
    }
    int suspend() {
        playing = false;
        cout << "[FFmpeg] Suspended" << endl;
        return 0;
    }
    int closeFile() {
        playing = false;
        currentFile.clear();
        cout << "[FFmpeg] File closed" << endl;
        return 0;
    }
};

// ============================================================
// OBJECT ADAPTERS — adapt each library to MediaPlayer interface
// ============================================================

class VLCAdapter : public MediaPlayer {
    VLCPlayer vlc;  // composition — holds the adaptee

public:
    void play(const string& filename) override {
        vlc.loadMedia(filename);
        vlc.startPlayback();
    }

    void pause() override {
        vlc.pausePlayback();
    }

    void stop() override {
        vlc.stopPlayback();
    }

    string getFormat() const override {
        return vlc.getSupportedFormat();
    }
};

class FFmpegAdapter : public MediaPlayer {
    FFmpegProcessor ffmpeg;  // composition

public:
    void play(const string& filename) override {
        ffmpeg.openFile(filename.c_str(), 1);  // mode 1 = read
        ffmpeg.decode();
    }

    void pause() override {
        ffmpeg.suspend();
    }

    void stop() override {
        ffmpeg.closeFile();
    }

    string getFormat() const override {
        return "FFmpeg (mp3, wav, ogg, flac, aac)";
    }
};

// ============================================================
// CLIENT CODE — works with MediaPlayer interface only
// ============================================================

class MusicApp {
    unique_ptr<MediaPlayer> player;

public:
    MusicApp(unique_ptr<MediaPlayer> p) : player(std::move(p)) {}

    void playSong(const string& filename) {
        cout << "Using player: " << player->getFormat() << endl;
        player->play(filename);
    }

    void pauseSong() {
        player->pause();
    }

    void stopSong() {
        player->stop();
    }
};

// Usage
int main() {
    // Client doesn't know about VLC or FFmpeg — only MediaPlayer
    MusicApp app1(make_unique<VLCAdapter>());
    app1.playSong("song.mkv");
    app1.pauseSong();
    app1.stopSong();

    cout << endl;

    MusicApp app2(make_unique<FFmpegAdapter>());
    app2.playSong("song.mp3");
    app2.pauseSong();
    app2.stopSong();
}
```

**Why Object Adapter is preferred:**
- Uses composition — more flexible than inheritance
- Can adapt multiple adaptees (hold references to several)
- Adaptee can be swapped at runtime
- Works even when adaptee is `final` (can't be inherited)
- Follows "Composition over Inheritance" principle

---

### Class Adapter (Multiple Inheritance)

The adapter inherits from BOTH the target interface and the adaptee. This is specific to C++ (languages like Java don't support multiple class inheritance).

```cpp
// Target interface
class Renderer {
public:
    virtual void renderCircle(int x, int y, int radius) = 0;
    virtual void renderRectangle(int x, int y, int width, int height) = 0;
    virtual void clear() = 0;
    virtual ~Renderer() = default;
};

// Adaptee — legacy graphics library with different interface
class LegacyGraphicsLib {
public:
    void drawEllipse(int cx, int cy, int rx, int ry) {
        cout << "[Legacy] Drawing ellipse at (" << cx << "," << cy
             << ") rx=" << rx << " ry=" << ry << endl;
    }
    void drawRect(int left, int top, int right, int bottom) {
        cout << "[Legacy] Drawing rect from (" << left << "," << top
             << ") to (" << right << "," << bottom << ")" << endl;
    }
    void clearScreen() {
        cout << "[Legacy] Screen cleared" << endl;
    }
};

// CLASS ADAPTER — inherits from both target and adaptee
class LegacyRendererAdapter : public Renderer, private LegacyGraphicsLib {
    //                                         ^^^^^^^ private inheritance
    // Private inheritance: we inherit implementation but don't expose
    // LegacyGraphicsLib's interface to clients
public:
    void renderCircle(int x, int y, int radius) override {
        // Adapt: circle is an ellipse with equal radii
        drawEllipse(x, y, radius, radius);
    }

    void renderRectangle(int x, int y, int width, int height) override {
        // Adapt: convert (x, y, w, h) to (left, top, right, bottom)
        drawRect(x, y, x + width, y + height);
    }

    void clear() override {
        clearScreen();
    }
};

// Client code
void renderScene(Renderer& renderer) {
    renderer.clear();
    renderer.renderCircle(100, 100, 50);
    renderer.renderRectangle(200, 200, 80, 60);
}

int main() {
    LegacyRendererAdapter adapter;
    renderScene(adapter);
}
```

**Class Adapter vs Object Adapter:**

| Aspect | Class Adapter | Object Adapter |
|--------|--------------|----------------|
| Mechanism | Multiple inheritance | Composition |
| Flexibility | Adapts one specific class | Can adapt any subclass of adaptee |
| Override | Can override adaptee behavior | Cannot override adaptee behavior |
| Language support | C++ only (multiple inheritance) | All OOP languages |
| Coupling | Tighter (inheritance) | Looser (composition) |
| Adaptee access | Direct (inherited members) | Through reference/pointer |
| Multiple adaptees | One at a time | Can hold multiple |

---

### Two-Way Adapter

A two-way adapter implements BOTH interfaces, allowing it to be used as either type. Useful when two systems need to interoperate in both directions.

```cpp
// Interface A — used by System A
class NewLogger {
public:
    virtual void logInfo(const string& message) = 0;
    virtual void logError(const string& message, int errorCode) = 0;
    virtual void logDebug(const string& message) = 0;
    virtual ~NewLogger() = default;
};

// Interface B — used by System B (legacy)
class LegacyLogger {
public:
    virtual void writeLog(int level, const string& msg) = 0;
    virtual void flush() = 0;
    virtual ~LegacyLogger() = default;
};

// Two-Way Adapter — implements BOTH interfaces
class LoggerAdapter : public NewLogger, public LegacyLogger {
    // Internal implementation
    void writeToFile(const string& prefix, const string& message) {
        cout << "[" << prefix << "] " << message << endl;
    }

public:
    // --- NewLogger interface ---
    void logInfo(const string& message) override {
        writeToFile("INFO", message);
    }

    void logError(const string& message, int errorCode) override {
        writeToFile("ERROR", message + " (code: " + to_string(errorCode) + ")");
    }

    void logDebug(const string& message) override {
        writeToFile("DEBUG", message);
    }

    // --- LegacyLogger interface ---
    void writeLog(int level, const string& msg) override {
        switch (level) {
            case 0: logDebug(msg); break;
            case 1: logInfo(msg); break;
            case 2: logError(msg, -1); break;
            default: logInfo(msg); break;
        }
    }

    void flush() override {
        cout << "[FLUSH] All logs written" << endl;
    }
};

// System A uses NewLogger
void newSystem(NewLogger& logger) {
    logger.logInfo("New system started");
    logger.logError("Connection failed", 503);
}

// System B uses LegacyLogger
void legacySystem(LegacyLogger& logger) {
    logger.writeLog(1, "Legacy system running");
    logger.writeLog(2, "Legacy error occurred");
    logger.flush();
}

// Both systems can use the same adapter instance
int main() {
    LoggerAdapter adapter;

    newSystem(adapter);     // uses NewLogger interface
    legacySystem(adapter);  // uses LegacyLogger interface
    // Same object, two interfaces — seamless interop!
}
```

---

### Real-World Adapter Example — Payment Gateway Integration

```cpp
// Your application's payment interface
class PaymentProcessor {
public:
    virtual bool charge(const string& customerId, double amount,
                        const string& currency) = 0;
    virtual bool refund(const string& transactionId, double amount) = 0;
    virtual string getTransactionStatus(const string& transactionId) = 0;
    virtual ~PaymentProcessor() = default;
};

// Third-party: Stripe-like API (different interface)
class StripeAPI {
public:
    struct ChargeResult {
        string chargeId;
        bool success;
        string errorMessage;
    };

    ChargeResult createCharge(int amountInCents, const string& currency,
                               const string& customerToken) {
        cout << "[Stripe] Charging " << amountInCents << " cents ("
             << currency << ") to customer " << customerToken << endl;
        return {"ch_abc123", true, ""};
    }

    bool reverseCharge(const string& chargeId, int amountInCents) {
        cout << "[Stripe] Reversing charge " << chargeId
             << " for " << amountInCents << " cents" << endl;
        return true;
    }

    string retrieveCharge(const string& chargeId) {
        return "succeeded";
    }
};

// Third-party: PayPal-like API (completely different interface)
class PayPalSDK {
public:
    struct PaymentResponse {
        string paymentId;
        string state;  // "approved", "failed"
    };

    PaymentResponse executePayment(const string& payerId,
                                    double total, const string& currencyCode) {
        cout << "[PayPal] Executing payment of " << total << " "
             << currencyCode << " for payer " << payerId << endl;
        return {"PAY-xyz789", "approved"};
    }

    bool refundPayment(const string& saleId, double amount,
                        const string& currencyCode) {
        cout << "[PayPal] Refunding " << amount << " " << currencyCode
             << " for sale " << saleId << endl;
        return true;
    }

    string getPaymentDetails(const string& paymentId) {
        return "approved";
    }
};

// ============================================================
// ADAPTERS — make each gateway conform to PaymentProcessor
// ============================================================

class StripeAdapter : public PaymentProcessor {
    StripeAPI stripe;

public:
    bool charge(const string& customerId, double amount,
                const string& currency) override {
        // Stripe expects cents, not dollars
        int amountInCents = static_cast<int>(amount * 100);
        auto result = stripe.createCharge(amountInCents, currency, customerId);
        return result.success;
    }

    bool refund(const string& transactionId, double amount) override {
        int amountInCents = static_cast<int>(amount * 100);
        return stripe.reverseCharge(transactionId, amountInCents);
    }

    string getTransactionStatus(const string& transactionId) override {
        return stripe.retrieveCharge(transactionId);
    }
};

class PayPalAdapter : public PaymentProcessor {
    PayPalSDK paypal;

public:
    bool charge(const string& customerId, double amount,
                const string& currency) override {
        auto result = paypal.executePayment(customerId, amount, currency);
        return result.state == "approved";
    }

    bool refund(const string& transactionId, double amount) override {
        return paypal.refundPayment(transactionId, amount, "USD");
    }

    string getTransactionStatus(const string& transactionId) override {
        return paypal.getPaymentDetails(transactionId);
    }
};

// ============================================================
// CLIENT — works with any payment processor
// ============================================================

class CheckoutService {
    PaymentProcessor& processor;

public:
    CheckoutService(PaymentProcessor& p) : processor(p) {}

    bool processOrder(const string& customerId, double total) {
        cout << "Processing order for $" << total << endl;
        bool success = processor.charge(customerId, total, "USD");
        if (success) {
            cout << "Payment successful!" << endl;
        } else {
            cout << "Payment failed!" << endl;
        }
        return success;
    }
};

// Switch payment gateways by changing one line
int main() {
    StripeAdapter stripe;
    PayPalAdapter paypal;

    CheckoutService checkout(stripe);  // or: checkout(paypal)
    checkout.processOrder("cust_123", 99.99);
}
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
- When a simple function call would suffice (don't over-engineer)

**Adapter vs other patterns:**
| Pattern | Difference from Adapter |
|---------|------------------------|
| Bridge | Designed up-front to separate abstraction from implementation; Adapter retrofits |
| Decorator | Adds new behavior; Adapter converts interface without adding behavior |
| Facade | Simplifies a complex subsystem; Adapter makes one interface match another |
| Proxy | Same interface as the real object; Adapter changes the interface |

---


## 5.2 Bridge Pattern

> The Bridge pattern decouples an abstraction from its implementation so that the two can vary independently. It achieves this by placing the abstraction and the implementation in separate class hierarchies, connected by a "bridge" (a reference from the abstraction to the implementation).

---

### Why Bridge?

Without Bridge, combining abstractions and implementations leads to a **class explosion** problem:

```cpp
// BAD: Cartesian product of shapes × rendering APIs
// Shapes: Circle, Rectangle, Triangle
// Renderers: OpenGL, DirectX, Vulkan

// Without Bridge — need N × M classes:
class OpenGLCircle {};
class OpenGLRectangle {};
class OpenGLTriangle {};
class DirectXCircle {};
class DirectXRectangle {};
class DirectXTriangle {};
class VulkanCircle {};
class VulkanRectangle {};
class VulkanTriangle {};
// 3 shapes × 3 renderers = 9 classes!
// Add one shape → 3 more classes. Add one renderer → 3 more classes.

// WITH Bridge — need N + M classes:
// 3 shapes + 3 renderers = 6 classes. Much better!
```

**The core insight:** When you have two orthogonal dimensions of variation, use Bridge to separate them into independent hierarchies.

---

### Structure

```
┌─────────────────────┐           ┌─────────────────────┐
│    Abstraction      │           │   Implementation    │
│                     │ bridge    │   (Interface)       │
│ - impl: Impl*  ────┼──────────→│                     │
│ + operation()       │           │ + operationImpl()   │
└─────────────────────┘           └─────────────────────┘
          ▲                                 ▲
          │                                 │
┌─────────┴──────────┐          ┌──────────┴──────────┐
│ RefinedAbstraction  │          │ ConcreteImplA       │
│ + operation()       │          │ + operationImpl()   │
└─────────────────────┘          ├─────────────────────┤
                                 │ ConcreteImplB       │
                                 │ + operationImpl()   │
                                 └─────────────────────┘
```

**Key idea:** The Abstraction holds a pointer/reference to the Implementation. The abstraction delegates work to the implementation object.

---

### Complete Implementation — Shapes × Renderers

```cpp
// ============================================================
// IMPLEMENTATION HIERARCHY — how to render
// ============================================================

class Renderer {
public:
    virtual void renderCircle(float x, float y, float radius) = 0;
    virtual void renderRectangle(float x, float y, float width, float height) = 0;
    virtual void renderTriangle(float x1, float y1, float x2, float y2,
                                 float x3, float y3) = 0;
    virtual void setColor(int r, int g, int b) = 0;
    virtual string getName() const = 0;
    virtual ~Renderer() = default;
};

class OpenGLRenderer : public Renderer {
public:
    void renderCircle(float x, float y, float radius) override {
        cout << "[OpenGL] glDrawCircle(" << x << ", " << y
             << ", " << radius << ")" << endl;
    }
    void renderRectangle(float x, float y, float width, float height) override {
        cout << "[OpenGL] glDrawRect(" << x << ", " << y
             << ", " << width << ", " << height << ")" << endl;
    }
    void renderTriangle(float x1, float y1, float x2, float y2,
                         float x3, float y3) override {
        cout << "[OpenGL] glDrawTriangle(" << x1 << "," << y1 << " "
             << x2 << "," << y2 << " " << x3 << "," << y3 << ")" << endl;
    }
    void setColor(int r, int g, int b) override {
        cout << "[OpenGL] glColor3f(" << r << ", " << g << ", " << b << ")" << endl;
    }
    string getName() const override { return "OpenGL"; }
};

class DirectXRenderer : public Renderer {
public:
    void renderCircle(float x, float y, float radius) override {
        cout << "[DirectX] D3DDrawEllipse(" << x << ", " << y
             << ", " << radius << ", " << radius << ")" << endl;
    }
    void renderRectangle(float x, float y, float width, float height) override {
        cout << "[DirectX] D3DFillRect(" << x << ", " << y
             << ", " << x + width << ", " << y + height << ")" << endl;
    }
    void renderTriangle(float x1, float y1, float x2, float y2,
                         float x3, float y3) override {
        cout << "[DirectX] D3DDrawPrimitive(TRIANGLE, ...)" << endl;
    }
    void setColor(int r, int g, int b) override {
        cout << "[DirectX] D3DSetColor(0x" << hex << (r << 16 | g << 8 | b)
             << dec << ")" << endl;
    }
    string getName() const override { return "DirectX"; }
};

class VulkanRenderer : public Renderer {
public:
    void renderCircle(float x, float y, float radius) override {
        cout << "[Vulkan] vkCmdDrawCircle(" << x << ", " << y
             << ", " << radius << ")" << endl;
    }
    void renderRectangle(float x, float y, float width, float height) override {
        cout << "[Vulkan] vkCmdDrawRect(" << x << ", " << y
             << ", " << width << ", " << height << ")" << endl;
    }
    void renderTriangle(float x1, float y1, float x2, float y2,
                         float x3, float y3) override {
        cout << "[Vulkan] vkCmdDraw(TRIANGLE_LIST, ...)" << endl;
    }
    void setColor(int r, int g, int b) override {
        cout << "[Vulkan] vkSetPushConstant(color, " << r << ", "
             << g << ", " << b << ")" << endl;
    }
    string getName() const override { return "Vulkan"; }
};

// ============================================================
// ABSTRACTION HIERARCHY — what to render
// ============================================================

class Shape {
protected:
    Renderer& renderer;  // THE BRIDGE — reference to implementation
    float x, y;
    int colorR = 0, colorG = 0, colorB = 0;

public:
    Shape(Renderer& r, float x, float y) : renderer(r), x(x), y(y) {}

    virtual void draw() = 0;
    virtual void resize(float factor) = 0;
    virtual string describe() const = 0;

    void setColor(int r, int g, int b) {
        colorR = r; colorG = g; colorB = b;
    }

    void moveTo(float newX, float newY) {
        x = newX; y = newY;
    }

    virtual ~Shape() = default;
};

// Refined Abstractions — specific shapes
class Circle : public Shape {
    float radius;

public:
    Circle(Renderer& r, float x, float y, float radius)
        : Shape(r, x, y), radius(radius) {}

    void draw() override {
        renderer.setColor(colorR, colorG, colorB);
        renderer.renderCircle(x, y, radius);
    }

    void resize(float factor) override {
        radius *= factor;
    }

    string describe() const override {
        return "Circle(r=" + to_string(radius) + ") at (" +
               to_string(x) + "," + to_string(y) + ")";
    }
};

class Rectangle : public Shape {
    float width, height;

public:
    Rectangle(Renderer& r, float x, float y, float w, float h)
        : Shape(r, x, y), width(w), height(h) {}

    void draw() override {
        renderer.setColor(colorR, colorG, colorB);
        renderer.renderRectangle(x, y, width, height);
    }

    void resize(float factor) override {
        width *= factor;
        height *= factor;
    }

    string describe() const override {
        return "Rectangle(" + to_string(width) + "x" + to_string(height) +
               ") at (" + to_string(x) + "," + to_string(y) + ")";
    }
};

class Triangle : public Shape {
    float x2, y2, x3, y3;

public:
    Triangle(Renderer& r, float x1, float y1, float x2, float y2,
             float x3, float y3)
        : Shape(r, x1, y1), x2(x2), y2(y2), x3(x3), y3(y3) {}

    void draw() override {
        renderer.setColor(colorR, colorG, colorB);
        renderer.renderTriangle(x, y, x2, y2, x3, y3);
    }

    void resize(float factor) override {
        // Scale relative to first vertex
        x2 = x + (x2 - x) * factor;
        y2 = y + (y2 - y) * factor;
        x3 = x + (x3 - x) * factor;
        y3 = y + (y3 - y) * factor;
    }

    string describe() const override {
        return "Triangle at (" + to_string(x) + "," + to_string(y) + ")";
    }
};

// ============================================================
// CLIENT CODE — mix and match shapes with renderers
// ============================================================

int main() {
    OpenGLRenderer opengl;
    DirectXRenderer directx;
    VulkanRenderer vulkan;

    // Same shape, different renderers
    Circle c1(opengl, 100, 100, 50);
    Circle c2(directx, 100, 100, 50);
    Circle c3(vulkan, 100, 100, 50);

    c1.setColor(255, 0, 0);
    c2.setColor(255, 0, 0);
    c3.setColor(255, 0, 0);

    cout << "=== Same circle, three renderers ===" << endl;
    c1.draw();  // OpenGL rendering
    c2.draw();  // DirectX rendering
    c3.draw();  // Vulkan rendering

    cout << endl << "=== Different shapes, same renderer ===" << endl;
    Rectangle rect(opengl, 50, 50, 200, 100);
    Triangle tri(opengl, 0, 0, 100, 0, 50, 80);

    rect.setColor(0, 255, 0);
    tri.setColor(0, 0, 255);

    rect.draw();
    tri.draw();

    // Adding a new shape or renderer requires only ONE new class!
}
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

```cpp
// Implementation hierarchy — HOW to send
class MessageSender {
public:
    virtual void sendMessage(const string& title, const string& body,
                              const string& recipient) = 0;
    virtual string getSenderType() const = 0;
    virtual ~MessageSender() = default;
};

class EmailSender : public MessageSender {
public:
    void sendMessage(const string& title, const string& body,
                      const string& recipient) override {
        cout << "[Email] To: " << recipient << endl;
        cout << "  Subject: " << title << endl;
        cout << "  Body: " << body << endl;
    }
    string getSenderType() const override { return "Email"; }
};

class SMSSender : public MessageSender {
public:
    void sendMessage(const string& title, const string& body,
                      const string& recipient) override {
        cout << "[SMS] To: " << recipient << endl;
        cout << "  " << title << ": " << body << endl;
    }
    string getSenderType() const override { return "SMS"; }
};

class SlackSender : public MessageSender {
public:
    void sendMessage(const string& title, const string& body,
                      const string& recipient) override {
        cout << "[Slack] Channel: " << recipient << endl;
        cout << "  *" << title << "*" << endl;
        cout << "  " << body << endl;
    }
    string getSenderType() const override { return "Slack"; }
};

// Abstraction hierarchy — WHAT to send
class Notification {
protected:
    MessageSender& sender;  // bridge to implementation

public:
    Notification(MessageSender& s) : sender(s) {}
    virtual void notify(const string& recipient) = 0;
    virtual ~Notification() = default;
};

class UrgentNotification : public Notification {
    string message;
public:
    UrgentNotification(MessageSender& s, const string& msg)
        : Notification(s), message(msg) {}

    void notify(const string& recipient) override {
        sender.sendMessage("🚨 URGENT", message, recipient);
        // Urgent: also send a follow-up
        sender.sendMessage("🚨 REMINDER", "Please respond ASAP: " + message, recipient);
    }
};

class RegularNotification : public Notification {
    string message;
public:
    RegularNotification(MessageSender& s, const string& msg)
        : Notification(s), message(msg) {}

    void notify(const string& recipient) override {
        sender.sendMessage("Notification", message, recipient);
    }
};

class ScheduledNotification : public Notification {
    string message;
    string scheduledTime;
public:
    ScheduledNotification(MessageSender& s, const string& msg, const string& time)
        : Notification(s), message(msg), scheduledTime(time) {}

    void notify(const string& recipient) override {
        cout << "[Scheduled for " << scheduledTime << "]" << endl;
        sender.sendMessage("Scheduled Alert", message, recipient);
    }
};

// Usage — any notification type × any sender
int main() {
    EmailSender email;
    SMSSender sms;
    SlackSender slack;

    // Urgent via Email
    UrgentNotification urgentEmail(email, "Server is down!");
    urgentEmail.notify("admin@company.com");

    cout << endl;

    // Regular via Slack
    RegularNotification regularSlack(slack, "Deployment completed");
    regularSlack.notify("#deployments");

    cout << endl;

    // Scheduled via SMS
    ScheduledNotification scheduledSMS(sms, "Meeting in 15 minutes", "2:45 PM");
    scheduledSMS.notify("+1-555-0123");
}
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
- Only one dimension of variation (use simple inheritance)
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
```cpp
// BAD: Client must distinguish between files and directories
void calculateSize(/* ??? */) {
    if (isFile) {
        return file.getSize();
    } else if (isDirectory) {
        int total = 0;
        for (auto& child : directory.getChildren()) {
            if (child.isFile()) {
                total += child.getSize();
            } else {
                // Recursion with type checking — messy!
                total += calculateSize(child);
            }
        }
        return total;
    }
}
// Type checking everywhere, hard to extend, violates OCP
```

**With Composite:** Both files and directories implement the same interface. The client calls `getSize()` on anything — files return their size, directories recursively sum their children's sizes.

---

### Structure

```
┌─────────────────────┐
│     Component       │ ← common interface
│                     │
│ + operation()       │
│ + add(Component)    │
│ + remove(Component) │
│ + getChild(int)     │
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

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <algorithm>
#include <iomanip>

using namespace std;

// ============================================================
// COMPONENT — common interface for files and directories
// ============================================================

class FileSystemComponent {
protected:
    string name;

public:
    FileSystemComponent(const string& name) : name(name) {}
    virtual ~FileSystemComponent() = default;

    // Common operations
    virtual int getSize() const = 0;
    virtual void display(int indent = 0) const = 0;
    virtual string getPath(const string& parentPath = "") const {
        return parentPath.empty() ? name : parentPath + "/" + name;
    }
    virtual int countFiles() const = 0;
    virtual int countDirectories() const = 0;

    // Composite operations — default implementations for leaves
    virtual void add(shared_ptr<FileSystemComponent> component) {
        throw runtime_error("Cannot add to a leaf node: " + name);
    }
    virtual void remove(const string& componentName) {
        throw runtime_error("Cannot remove from a leaf node: " + name);
    }
    virtual shared_ptr<FileSystemComponent> getChild(int index) const {
        throw runtime_error("No children in a leaf node: " + name);
    }
    virtual shared_ptr<FileSystemComponent> find(const string& searchName) const {
        if (name == searchName) return nullptr;  // found but can't return shared_ptr to this
        return nullptr;
    }

    string getName() const { return name; }
    virtual bool isComposite() const { return false; }
};

// ============================================================
// LEAF — individual file
// ============================================================

class File : public FileSystemComponent {
    int size;  // in bytes
    string extension;

public:
    File(const string& name, int size)
        : FileSystemComponent(name), size(size) {
        auto dotPos = name.rfind('.');
        extension = (dotPos != string::npos) ? name.substr(dotPos) : "";
    }

    int getSize() const override {
        return size;
    }

    void display(int indent = 0) const override {
        cout << string(indent, ' ') << "📄 " << name
             << " (" << size << " bytes)" << endl;
    }

    int countFiles() const override { return 1; }
    int countDirectories() const override { return 0; }

    string getExtension() const { return extension; }
};

// ============================================================
// COMPOSITE — directory containing files and subdirectories
// ============================================================

class Directory : public FileSystemComponent {
    vector<shared_ptr<FileSystemComponent>> children;

public:
    Directory(const string& name) : FileSystemComponent(name) {}

    // Recursively calculate total size
    int getSize() const override {
        int total = 0;
        for (const auto& child : children) {
            total += child->getSize();
        }
        return total;
    }

    void display(int indent = 0) const override {
        cout << string(indent, ' ') << "📁 " << name
             << "/ (" << getSize() << " bytes, "
             << countFiles() << " files)" << endl;
        for (const auto& child : children) {
            child->display(indent + 4);
        }
    }

    int countFiles() const override {
        int count = 0;
        for (const auto& child : children) {
            count += child->countFiles();
        }
        return count;
    }

    int countDirectories() const override {
        int count = 1;  // count self
        for (const auto& child : children) {
            count += child->countDirectories();
        }
        return count;
    }

    // Composite-specific operations
    void add(shared_ptr<FileSystemComponent> component) override {
        children.push_back(component);
    }

    void remove(const string& componentName) override {
        children.erase(
            std::remove_if(children.begin(), children.end(),
                [&componentName](const shared_ptr<FileSystemComponent>& c) {
                    return c->getName() == componentName;
                }),
            children.end()
        );
    }

    shared_ptr<FileSystemComponent> getChild(int index) const override {
        if (index < 0 || index >= static_cast<int>(children.size())) {
            throw out_of_range("Index out of range");
        }
        return children[index];
    }

    bool isComposite() const override { return true; }

    // Search recursively
    shared_ptr<FileSystemComponent> find(const string& searchName) const override {
        for (const auto& child : children) {
            if (child->getName() == searchName) return child;
            if (child->isComposite()) {
                auto found = child->find(searchName);
                if (found) return found;
            }
        }
        return nullptr;
    }

    const vector<shared_ptr<FileSystemComponent>>& getChildren() const {
        return children;
    }
};

// ============================================================
// CLIENT CODE — treats files and directories uniformly
// ============================================================

// This function works with ANY FileSystemComponent — leaf or composite
void printSize(const FileSystemComponent& component) {
    cout << component.getName() << ": " << component.getSize() << " bytes" << endl;
}

int main() {
    // Build a file system tree
    auto root = make_shared<Directory>("root");

    auto home = make_shared<Directory>("home");
    auto user = make_shared<Directory>("user");
    auto docs = make_shared<Directory>("documents");
    auto pics = make_shared<Directory>("pictures");

    auto etc = make_shared<Directory>("etc");
    auto var = make_shared<Directory>("var");
    auto logs = make_shared<Directory>("logs");

    // Add files
    docs->add(make_shared<File>("resume.pdf", 250000));
    docs->add(make_shared<File>("notes.txt", 1200));
    docs->add(make_shared<File>("report.docx", 450000));

    pics->add(make_shared<File>("photo1.jpg", 3500000));
    pics->add(make_shared<File>("photo2.png", 2100000));

    user->add(docs);
    user->add(pics);
    user->add(make_shared<File>(".bashrc", 500));

    home->add(user);

    etc->add(make_shared<File>("config.yaml", 800));
    etc->add(make_shared<File>("hosts", 200));

    logs->add(make_shared<File>("app.log", 5000000));
    logs->add(make_shared<File>("error.log", 120000));
    var->add(logs);

    root->add(home);
    root->add(etc);
    root->add(var);

    // Display entire tree
    root->display();

    cout << endl;

    // Uniform treatment — works on both files and directories
    printSize(*root);           // total size of everything
    printSize(*docs);           // total size of documents folder
    printSize(*docs->getChild(0));  // size of resume.pdf

    cout << endl;
    cout << "Total files: " << root->countFiles() << endl;
    cout << "Total directories: " << root->countDirectories() << endl;

    // Search
    auto found = root->find("error.log");
    if (found) {
        cout << "Found: ";
        found->display();
    }
}
```

---

### Leaf vs Composite Nodes

| Aspect | Leaf | Composite |
|--------|------|-----------|
| Children | None | Zero or more |
| `add()`/`remove()` | Throws or no-op | Manages children |
| `operation()` | Performs directly | Delegates to children |
| Examples | File, MenuItem, Employee | Directory, Menu, Department |

**Design choice — where to put child management methods:**

1. **In Component (transparency):** Both Leaf and Composite have `add()`/`remove()`. Leaf throws at runtime. Client treats everything uniformly but risks runtime errors.

2. **In Composite only (safety):** Only Composite has `add()`/`remove()`. Client must check type before adding. Safer but less uniform.

```cpp
// Approach 1: Transparency (shown above)
// Leaf::add() throws — runtime error if misused

// Approach 2: Safety
class Component {
public:
    virtual void operation() = 0;
    // NO add/remove here
};

class Composite : public Component {
public:
    void add(shared_ptr<Component> c);     // only in Composite
    void remove(shared_ptr<Component> c);  // only in Composite
    void operation() override { /* delegate */ }
};

// Client must downcast to add children:
auto composite = dynamic_cast<Composite*>(component);
if (composite) {
    composite->add(child);
}
```

The GoF book favors transparency (approach 1). Modern practice often favors safety (approach 2).

---

### Another Example — UI Component Tree

```cpp
class UIComponent {
protected:
    string id;
    int x, y, width, height;

public:
    UIComponent(const string& id, int x, int y, int w, int h)
        : id(id), x(x), y(y), width(w), height(h) {}

    virtual void render(int offsetX = 0, int offsetY = 0) const = 0;
    virtual void handleClick(int clickX, int clickY) = 0;
    virtual int getComponentCount() const = 0;

    virtual void add(shared_ptr<UIComponent> component) {
        throw runtime_error("Cannot add children to " + id);
    }

    bool containsPoint(int px, int py, int offsetX = 0, int offsetY = 0) const {
        int absX = x + offsetX, absY = y + offsetY;
        return px >= absX && px < absX + width &&
               py >= absY && py < absY + height;
    }

    string getId() const { return id; }
    virtual ~UIComponent() = default;
};

// Leaf components
class Button : public UIComponent {
    string label;
    function<void()> onClick;

public:
    Button(const string& id, int x, int y, int w, int h,
           const string& label, function<void()> handler = nullptr)
        : UIComponent(id, x, y, w, h), label(label), onClick(handler) {}

    void render(int offsetX = 0, int offsetY = 0) const override {
        cout << string((x + offsetX) / 10, ' ')
             << "[" << label << "]" << endl;
    }

    void handleClick(int clickX, int clickY) override {
        if (containsPoint(clickX, clickY)) {
            cout << "Button '" << label << "' clicked!" << endl;
            if (onClick) onClick();
        }
    }

    int getComponentCount() const override { return 1; }
};

class TextLabel : public UIComponent {
    string text;

public:
    TextLabel(const string& id, int x, int y, int w, int h, const string& text)
        : UIComponent(id, x, y, w, h), text(text) {}

    void render(int offsetX = 0, int offsetY = 0) const override {
        cout << string((x + offsetX) / 10, ' ') << text << endl;
    }

    void handleClick(int clickX, int clickY) override {
        // Labels don't respond to clicks
    }

    int getComponentCount() const override { return 1; }
};

// Composite component
class Panel : public UIComponent {
    vector<shared_ptr<UIComponent>> children;
    string title;

public:
    Panel(const string& id, int x, int y, int w, int h, const string& title = "")
        : UIComponent(id, x, y, w, h), title(title) {}

    void add(shared_ptr<UIComponent> component) override {
        children.push_back(component);
    }

    void render(int offsetX = 0, int offsetY = 0) const override {
        int absX = x + offsetX;
        int absY = y + offsetY;
        cout << string(absX / 10, ' ') << "┌── " << title << " ──┐" << endl;
        for (const auto& child : children) {
            child->render(absX, absY);
        }
        cout << string(absX / 10, ' ') << "└────────────────┘" << endl;
    }

    void handleClick(int clickX, int clickY) override {
        // Propagate click to children
        for (auto& child : children) {
            child->handleClick(clickX, clickY);
        }
    }

    int getComponentCount() const override {
        int count = 1;  // self
        for (const auto& child : children) {
            count += child->getComponentCount();
        }
        return count;
    }
};

// Usage — build a UI tree
int main() {
    auto window = make_shared<Panel>("window", 0, 0, 800, 600, "Main Window");

    auto toolbar = make_shared<Panel>("toolbar", 0, 0, 800, 40, "Toolbar");
    toolbar->add(make_shared<Button>("btn-new", 10, 5, 60, 30, "New"));
    toolbar->add(make_shared<Button>("btn-open", 80, 5, 60, 30, "Open"));
    toolbar->add(make_shared<Button>("btn-save", 150, 5, 60, 30, "Save"));

    auto sidebar = make_shared<Panel>("sidebar", 0, 40, 200, 560, "Sidebar");
    sidebar->add(make_shared<TextLabel>("lbl-files", 10, 10, 180, 20, "Files:"));
    sidebar->add(make_shared<Button>("btn-file1", 10, 35, 180, 25, "document.txt"));
    sidebar->add(make_shared<Button>("btn-file2", 10, 65, 180, 25, "image.png"));

    auto content = make_shared<Panel>("content", 200, 40, 600, 560, "Content");
    content->add(make_shared<TextLabel>("lbl-welcome", 20, 20, 560, 30, "Welcome to the editor!"));

    window->add(toolbar);
    window->add(sidebar);
    window->add(content);

    // Render entire UI tree with one call
    window->render();

    cout << endl;
    cout << "Total components: " << window->getComponentCount() << endl;

    // Handle click — propagates through the tree
    window->handleClick(100, 15);  // clicks somewhere in toolbar area
}
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
- Structure is flat (no hierarchy) — use a simple list
- Leaf and composite behaviors are very different — uniform interface becomes awkward
- Performance-critical code where virtual dispatch overhead matters

**Real-world examples:**
- File systems (files and directories)
- GUI widget trees (buttons, panels, windows)
- HTML/XML DOM trees
- Organization hierarchies
- Menu systems (items and sub-menus)
- Arithmetic expression trees
- Scene graphs in game engines

---


## 5.4 Decorator Pattern

> The Decorator pattern attaches additional responsibilities to an object dynamically. It provides a flexible alternative to subclassing for extending functionality. Decorators wrap the original object and add behavior before or after delegating to it.

---

### Why Decorator?

Inheritance is static — you choose the behavior at compile time. But what if you need to add or remove behaviors at runtime, or combine them in arbitrary ways?

```cpp
// BAD: Inheritance explosion for combinations
// Base: Coffee
// Options: Milk, Sugar, Whip, Caramel, Vanilla, Soy

// Without Decorator — need a class for every combination:
class CoffeeWithMilk {};
class CoffeeWithSugar {};
class CoffeeWithMilkAndSugar {};
class CoffeeWithMilkAndWhip {};
class CoffeeWithMilkAndSugarAndWhip {};
class CoffeeWithMilkAndSugarAndWhipAndCaramel {};
// ... 2^6 = 64 possible combinations! Unmaintainable.

// WITH Decorator:
// 1 base + 6 decorators = 7 classes. Any combination at runtime.
```

**Key insight:** Decorators have the SAME interface as the object they wrap. This means you can stack them — a decorator wrapping a decorator wrapping the original object.

---

### Structure

```
┌─────────────────────┐
│     Component       │ ← common interface
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

```cpp
#include <iostream>
#include <string>
#include <memory>

using namespace std;

// ============================================================
// COMPONENT — base interface
// ============================================================

class Beverage {
public:
    virtual string getDescription() const = 0;
    virtual double getCost() const = 0;
    virtual ~Beverage() = default;
};

// ============================================================
// CONCRETE COMPONENTS — base beverages
// ============================================================

class Espresso : public Beverage {
public:
    string getDescription() const override { return "Espresso"; }
    double getCost() const override { return 1.99; }
};

class HouseBlend : public Beverage {
public:
    string getDescription() const override { return "House Blend Coffee"; }
    double getCost() const override { return 0.89; }
};

class DarkRoast : public Beverage {
public:
    string getDescription() const override { return "Dark Roast Coffee"; }
    double getCost() const override { return 1.49; }
};

class Decaf : public Beverage {
public:
    string getDescription() const override { return "Decaf Coffee"; }
    double getCost() const override { return 1.29; }
};

// ============================================================
// BASE DECORATOR — wraps a Beverage
// ============================================================

class CondimentDecorator : public Beverage {
protected:
    unique_ptr<Beverage> beverage;  // the wrapped component

public:
    CondimentDecorator(unique_ptr<Beverage> bev) : beverage(std::move(bev)) {}
};

// ============================================================
// CONCRETE DECORATORS — each adds a condiment
// ============================================================

class Milk : public CondimentDecorator {
public:
    Milk(unique_ptr<Beverage> bev) : CondimentDecorator(std::move(bev)) {}

    string getDescription() const override {
        return beverage->getDescription() + ", Milk";
    }

    double getCost() const override {
        return beverage->getCost() + 0.30;
    }
};

class Sugar : public CondimentDecorator {
public:
    Sugar(unique_ptr<Beverage> bev) : CondimentDecorator(std::move(bev)) {}

    string getDescription() const override {
        return beverage->getDescription() + ", Sugar";
    }

    double getCost() const override {
        return beverage->getCost() + 0.20;
    }
};

class WhipCream : public CondimentDecorator {
public:
    WhipCream(unique_ptr<Beverage> bev) : CondimentDecorator(std::move(bev)) {}

    string getDescription() const override {
        return beverage->getDescription() + ", Whip Cream";
    }

    double getCost() const override {
        return beverage->getCost() + 0.50;
    }
};

class Caramel : public CondimentDecorator {
public:
    Caramel(unique_ptr<Beverage> bev) : CondimentDecorator(std::move(bev)) {}

    string getDescription() const override {
        return beverage->getDescription() + ", Caramel";
    }

    double getCost() const override {
        return beverage->getCost() + 0.60;
    }
};

class Vanilla : public CondimentDecorator {
public:
    Vanilla(unique_ptr<Beverage> bev) : CondimentDecorator(std::move(bev)) {}

    string getDescription() const override {
        return beverage->getDescription() + ", Vanilla";
    }

    double getCost() const override {
        return beverage->getCost() + 0.40;
    }
};

class SoyMilk : public CondimentDecorator {
public:
    SoyMilk(unique_ptr<Beverage> bev) : CondimentDecorator(std::move(bev)) {}

    string getDescription() const override {
        return beverage->getDescription() + ", Soy Milk";
    }

    double getCost() const override {
        return beverage->getCost() + 0.50;
    }
};

// ============================================================
// CLIENT CODE — stack decorators freely
// ============================================================

void printOrder(const Beverage& bev) {
    cout << bev.getDescription() << " — $" << bev.getCost() << endl;
}

int main() {
    // Simple espresso
    auto order1 = make_unique<Espresso>();
    printOrder(*order1);
    // Espresso — $1.99

    // Dark roast with milk and sugar
    unique_ptr<Beverage> order2 = make_unique<DarkRoast>();
    order2 = make_unique<Milk>(std::move(order2));
    order2 = make_unique<Sugar>(std::move(order2));
    printOrder(*order2);
    // Dark Roast Coffee, Milk, Sugar — $1.99

    // House blend with double milk, whip, and caramel
    unique_ptr<Beverage> order3 = make_unique<HouseBlend>();
    order3 = make_unique<Milk>(std::move(order3));
    order3 = make_unique<Milk>(std::move(order3));     // double milk!
    order3 = make_unique<WhipCream>(std::move(order3));
    order3 = make_unique<Caramel>(std::move(order3));
    printOrder(*order3);
    // House Blend Coffee, Milk, Milk, Whip Cream, Caramel — $2.59

    // Decaf with soy milk and vanilla
    unique_ptr<Beverage> order4 = make_unique<Decaf>();
    order4 = make_unique<SoyMilk>(std::move(order4));
    order4 = make_unique<Vanilla>(std::move(order4));
    printOrder(*order4);
    // Decaf Coffee, Soy Milk, Vanilla — $2.19
}
```

**How stacking works:**
```
order3 call chain for getCost():

Caramel::getCost()
  → WhipCream::getCost() + 0.60
    → Milk::getCost() + 0.50
      → Milk::getCost() + 0.30
        → HouseBlend::getCost() + 0.30
          → 0.89
        = 1.19
      = 1.49
    = 1.99
  = 2.59
```

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

```cpp
// Component interface
class DataStream {
public:
    virtual void write(const string& data) = 0;
    virtual string read() = 0;
    virtual ~DataStream() = default;
};

// Concrete component — writes to a file
class FileStream : public DataStream {
    string filename;
    string buffer;  // simulated file content

public:
    FileStream(const string& fname) : filename(fname) {}

    void write(const string& data) override {
        buffer = data;
        cout << "[FileStream] Writing " << data.size()
             << " bytes to " << filename << endl;
    }

    string read() override {
        cout << "[FileStream] Reading from " << filename << endl;
        return buffer;
    }
};

// Base decorator
class StreamDecorator : public DataStream {
protected:
    unique_ptr<DataStream> wrappedStream;

public:
    StreamDecorator(unique_ptr<DataStream> stream)
        : wrappedStream(std::move(stream)) {}
};

// Encryption decorator
class EncryptionDecorator : public StreamDecorator {
    int key;

    string encrypt(const string& data) const {
        string result = data;
        for (char& c : result) c = c ^ key;  // simple XOR cipher
        return result;
    }

    string decrypt(const string& data) const {
        return encrypt(data);  // XOR is its own inverse
    }

public:
    EncryptionDecorator(unique_ptr<DataStream> stream, int encryptionKey = 42)
        : StreamDecorator(std::move(stream)), key(encryptionKey) {}

    void write(const string& data) override {
        cout << "[Encryption] Encrypting data..." << endl;
        string encrypted = encrypt(data);
        wrappedStream->write(encrypted);
    }

    string read() override {
        string data = wrappedStream->read();
        cout << "[Encryption] Decrypting data..." << endl;
        return decrypt(data);
    }
};

// Compression decorator
class CompressionDecorator : public StreamDecorator {
    string compress(const string& data) const {
        // Simulated compression — in reality, use zlib etc.
        cout << "[Compression] Compressing " << data.size() << " bytes..." << endl;
        return "COMPRESSED[" + data + "]";
    }

    string decompress(const string& data) const {
        cout << "[Compression] Decompressing..." << endl;
        // Remove "COMPRESSED[" prefix and "]" suffix
        if (data.substr(0, 11) == "COMPRESSED[" && data.back() == ']') {
            return data.substr(11, data.size() - 12);
        }
        return data;
    }

public:
    CompressionDecorator(unique_ptr<DataStream> stream)
        : StreamDecorator(std::move(stream)) {}

    void write(const string& data) override {
        string compressed = compress(data);
        wrappedStream->write(compressed);
    }

    string read() override {
        string data = wrappedStream->read();
        return decompress(data);
    }
};

// Logging decorator
class LoggingDecorator : public StreamDecorator {
public:
    LoggingDecorator(unique_ptr<DataStream> stream)
        : StreamDecorator(std::move(stream)) {}

    void write(const string& data) override {
        cout << "[LOG] Write operation: " << data.size() << " bytes" << endl;
        wrappedStream->write(data);
        cout << "[LOG] Write complete" << endl;
    }

    string read() override {
        cout << "[LOG] Read operation started" << endl;
        string data = wrappedStream->read();
        cout << "[LOG] Read complete: " << data.size() << " bytes" << endl;
        return data;
    }
};

// Usage — stack decorators in any order
int main() {
    // Plain file stream
    cout << "=== Plain write ===" << endl;
    auto plain = make_unique<FileStream>("data.txt");
    plain->write("Hello, World!");

    cout << endl << "=== Encrypted + Compressed + Logged ===" << endl;
    // Build decorated stream: Logging → Encryption → Compression → File
    unique_ptr<DataStream> stream = make_unique<FileStream>("secure_data.bin");
    stream = make_unique<CompressionDecorator>(std::move(stream));
    stream = make_unique<EncryptionDecorator>(std::move(stream));
    stream = make_unique<LoggingDecorator>(std::move(stream));

    stream->write("Sensitive data that needs protection");

    cout << endl << "=== Reading back ===" << endl;
    string result = stream->read();
    cout << "Result: " << result << endl;
}
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

### Stacking Decorators

One of the most powerful features — the same decorator can be applied multiple times:

```cpp
// Double encryption with different keys
unique_ptr<DataStream> stream = make_unique<FileStream>("top_secret.bin");
stream = make_unique<EncryptionDecorator>(std::move(stream), 42);   // first layer
stream = make_unique<EncryptionDecorator>(std::move(stream), 137);  // second layer
// Data is encrypted twice with different keys!

// Triple logging (silly but demonstrates the concept)
unique_ptr<Beverage> coffee = make_unique<Espresso>();
coffee = make_unique<Milk>(std::move(coffee));
coffee = make_unique<Milk>(std::move(coffee));
coffee = make_unique<Milk>(std::move(coffee));
// Triple milk! getCost() adds 0.30 three times.
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
- Java I/O streams (`BufferedInputStream`, `DataInputStream`, `GZIPInputStream`)
- Middleware in web frameworks (logging, auth, compression, CORS)
- GUI component borders, scrollbars, shadows
- Game character power-ups and buffs
- HTTP request/response interceptors

---


## 5.5 Facade Pattern

> The Facade pattern provides a simplified, unified interface to a complex subsystem. It doesn't add new functionality — it just makes the subsystem easier to use by hiding its complexity behind a single class.

---

### Why Facade?

Complex subsystems often have many classes with intricate interdependencies. Clients that need to use the subsystem must understand and coordinate all these classes — even for simple tasks.

```cpp
// BAD: Client must coordinate multiple subsystem classes directly
void playMovie(const string& filename) {
    DVDPlayer dvd;
    Amplifier amp;
    Projector proj;
    Screen screen;
    Lights lights;
    PopcornMaker popcorn;

    // Client must know the exact sequence and dependencies!
    screen.down();
    lights.dim(10);
    projector.on();
    projector.setInput("dvd");
    projector.wideScreenMode();
    amp.on();
    amp.setVolume(7);
    amp.setInput("dvd");
    dvd.on();
    dvd.play(filename);
    popcorn.on();
    popcorn.pop();
    // 12 steps just to watch a movie! And the order matters!
}
```

**With Facade:** One method call — `homeTheater.watchMovie("Inception")`.

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
│ + simpleOp() │
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

```cpp
#include <iostream>
#include <string>

using namespace std;

// ============================================================
// SUBSYSTEM CLASSES — complex, interdependent components
// ============================================================

class DVDPlayer {
public:
    void on() { cout << "  DVD Player: ON" << endl; }
    void off() { cout << "  DVD Player: OFF" << endl; }
    void play(const string& movie) {
        cout << "  DVD Player: Playing \"" << movie << "\"" << endl;
    }
    void pause() { cout << "  DVD Player: Paused" << endl; }
    void stop() { cout << "  DVD Player: Stopped" << endl; }
    void eject() { cout << "  DVD Player: Disc ejected" << endl; }
};

class Amplifier {
public:
    void on() { cout << "  Amplifier: ON" << endl; }
    void off() { cout << "  Amplifier: OFF" << endl; }
    void setVolume(int level) {
        cout << "  Amplifier: Volume set to " << level << endl;
    }
    void setInput(const string& source) {
        cout << "  Amplifier: Input set to " << source << endl;
    }
    void setSurroundSound() {
        cout << "  Amplifier: Surround sound ON" << endl;
    }
    void setStereo() {
        cout << "  Amplifier: Stereo mode" << endl;
    }
};

class Projector {
public:
    void on() { cout << "  Projector: ON" << endl; }
    void off() { cout << "  Projector: OFF" << endl; }
    void setInput(const string& source) {
        cout << "  Projector: Input set to " << source << endl;
    }
    void wideScreenMode() {
        cout << "  Projector: Widescreen mode (16:9)" << endl;
    }
    void standardMode() {
        cout << "  Projector: Standard mode (4:3)" << endl;
    }
};

class Screen {
public:
    void down() { cout << "  Screen: Lowering..." << endl; }
    void up() { cout << "  Screen: Raising..." << endl; }
};

class TheaterLights {
public:
    void on() { cout << "  Lights: ON (full brightness)" << endl; }
    void dim(int level) {
        cout << "  Lights: Dimmed to " << level << "%" << endl;
    }
    void off() { cout << "  Lights: OFF" << endl; }
};

class PopcornMaker {
public:
    void on() { cout << "  Popcorn Maker: ON" << endl; }
    void off() { cout << "  Popcorn Maker: OFF" << endl; }
    void pop() { cout << "  Popcorn Maker: Popping corn!" << endl; }
};

class StreamingPlayer {
public:
    void on() { cout << "  Streaming Player: ON" << endl; }
    void off() { cout << "  Streaming Player: OFF" << endl; }
    void stream(const string& title) {
        cout << "  Streaming Player: Streaming \"" << title << "\"" << endl;
    }
    void pause() { cout << "  Streaming Player: Paused" << endl; }
    void stop() { cout << "  Streaming Player: Stopped" << endl; }
};

// ============================================================
// FACADE — simplified interface to the home theater subsystem
// ============================================================

class HomeTheaterFacade {
    DVDPlayer& dvd;
    Amplifier& amp;
    Projector& projector;
    Screen& screen;
    TheaterLights& lights;
    PopcornMaker& popcorn;
    StreamingPlayer& streamer;

public:
    HomeTheaterFacade(DVDPlayer& d, Amplifier& a, Projector& p,
                       Screen& s, TheaterLights& l, PopcornMaker& pc,
                       StreamingPlayer& st)
        : dvd(d), amp(a), projector(p), screen(s),
          lights(l), popcorn(pc), streamer(st) {}

    // Simple method that coordinates the entire subsystem
    void watchMovie(const string& movie) {
        cout << "=== Getting ready to watch \"" << movie << "\" ===" << endl;
        popcorn.on();
        popcorn.pop();
        lights.dim(10);
        screen.down();
        projector.on();
        projector.setInput("dvd");
        projector.wideScreenMode();
        amp.on();
        amp.setInput("dvd");
        amp.setSurroundSound();
        amp.setVolume(7);
        dvd.on();
        dvd.play(movie);
        cout << "=== Movie started! Enjoy! ===" << endl;
    }

    void endMovie() {
        cout << "=== Shutting down movie theater ===" << endl;
        dvd.stop();
        dvd.eject();
        dvd.off();
        amp.off();
        projector.off();
        screen.up();
        lights.on();
        popcorn.off();
        cout << "=== Theater shut down ===" << endl;
    }

    void streamShow(const string& title) {
        cout << "=== Getting ready to stream \"" << title << "\" ===" << endl;
        lights.dim(20);
        screen.down();
        projector.on();
        projector.setInput("streaming");
        projector.wideScreenMode();
        amp.on();
        amp.setInput("streaming");
        amp.setStereo();
        amp.setVolume(5);
        streamer.on();
        streamer.stream(title);
        cout << "=== Streaming started! ===" << endl;
    }

    void endStream() {
        cout << "=== Ending stream ===" << endl;
        streamer.stop();
        streamer.off();
        amp.off();
        projector.off();
        screen.up();
        lights.on();
        cout << "=== Stream ended ===" << endl;
    }

    void pauseAll() {
        cout << "=== Pausing ===" << endl;
        dvd.pause();
        lights.dim(50);
    }
};

// ============================================================
// CLIENT CODE — simple!
// ============================================================

int main() {
    // Create subsystem components
    DVDPlayer dvd;
    Amplifier amp;
    Projector projector;
    Screen screen;
    TheaterLights lights;
    PopcornMaker popcorn;
    StreamingPlayer streamer;

    // Create facade
    HomeTheaterFacade theater(dvd, amp, projector, screen,
                              lights, popcorn, streamer);

    // Client uses simple facade methods
    theater.watchMovie("Inception");
    cout << endl;
    theater.endMovie();

    cout << endl;

    theater.streamShow("Breaking Bad S01E01");
    cout << endl;
    theater.endStream();
}
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

```cpp
class CPU {
public:
    void freeze() { cout << "  CPU: Freezing processor" << endl; }
    void jump(long address) {
        cout << "  CPU: Jumping to address 0x" << hex << address << dec << endl;
    }
    void execute() { cout << "  CPU: Executing instructions" << endl; }
};

class Memory {
public:
    void load(long address, const string& data) {
        cout << "  Memory: Loading \"" << data << "\" at 0x"
             << hex << address << dec << endl;
    }
    void clear() { cout << "  Memory: Clearing all memory" << endl; }
};

class HardDrive {
public:
    string read(long sector, int size) {
        cout << "  HardDrive: Reading " << size << " bytes from sector "
             << sector << endl;
        return "boot_data";
    }
};

class BIOS {
public:
    void initialize() { cout << "  BIOS: Initializing hardware" << endl; }
    void checkHardware() { cout << "  BIOS: POST (Power-On Self-Test) passed" << endl; }
    string findBootDevice() {
        cout << "  BIOS: Boot device found: /dev/sda" << endl;
        return "/dev/sda";
    }
};

class GPU {
public:
    void initialize() { cout << "  GPU: Initializing graphics" << endl; }
    void displaySplash() { cout << "  GPU: Displaying boot splash screen" << endl; }
    void setResolution(int w, int h) {
        cout << "  GPU: Resolution set to " << w << "x" << h << endl;
    }
};

// Facade
class ComputerFacade {
    CPU cpu;
    Memory memory;
    HardDrive hdd;
    BIOS bios;
    GPU gpu;

public:
    void start() {
        cout << "=== Computer starting up ===" << endl;
        bios.initialize();
        bios.checkHardware();
        gpu.initialize();
        gpu.displaySplash();
        cpu.freeze();
        string bootDevice = bios.findBootDevice();
        string bootData = hdd.read(0, 512);  // read boot sector
        memory.load(0x00, bootData);
        cpu.jump(0x00);
        cpu.execute();
        gpu.setResolution(1920, 1080);
        cout << "=== Computer ready! ===" << endl;
    }

    void shutdown() {
        cout << "=== Computer shutting down ===" << endl;
        cpu.freeze();
        memory.clear();
        gpu.setResolution(0, 0);
        cout << "=== Computer off ===" << endl;
    }
};

// Client
int main() {
    ComputerFacade computer;
    computer.start();
    cout << endl;
    computer.shutdown();
}
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
- JDBC (facade over database communication)
- jQuery (`$()` is a facade over DOM manipulation)
- Spring Framework's `JdbcTemplate` (facade over JDBC)
- Compiler front-end (facade over lexer, parser, semantic analyzer)
- Operating system API (facade over hardware)

---


## 5.6 Flyweight Pattern

> The Flyweight pattern uses sharing to support large numbers of fine-grained objects efficiently. It reduces memory usage by sharing common state (intrinsic state) among multiple objects, while keeping unique state (extrinsic state) external.

---

### Why Flyweight?

When your application creates millions of similar objects, memory becomes a bottleneck:

```cpp
// BAD: Each character in a text editor is a full object
class Character {
    char symbol;           // 1 byte — unique per character
    string fontFamily;     // ~32 bytes — often shared ("Arial")
    int fontSize;          // 4 bytes — often shared (12)
    bool bold;             // 1 byte — often shared
    bool italic;           // 1 byte — often shared
    int colorR, colorG, colorB;  // 12 bytes — often shared
    // Total: ~51 bytes per character

    // For a 1 million character document:
    // 51 bytes × 1,000,000 = ~51 MB just for character objects!
};

// WITH Flyweight:
// Shared state (font, size, style, color) stored once per unique combination
// Unique state (symbol, position) stored per character
// If there are only 20 unique style combinations:
// 20 × 50 bytes (shared) + 1,000,000 × 5 bytes (unique) = ~5 MB
// 10x memory reduction!
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
│ + getFlyweight() │         │                  │
│ - cache: map     │         │ + operation(     │
└──────────────────┘         │   extrinsicState)│
                             └──────────────────┘
                                      ▲
                                      │
                             ┌────────┴────────┐
                             │                 │
                      ConcreteFlyweightA  ConcreteFlyweightB
```

---

### Complete Implementation — Text Editor Character Rendering

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <vector>
#include <memory>

using namespace std;

// ============================================================
// FLYWEIGHT — shared character style (intrinsic state)
// ============================================================

class CharacterStyle {
    // INTRINSIC STATE — shared among many characters
    string fontFamily;
    int fontSize;
    bool bold;
    bool italic;
    int colorR, colorG, colorB;

public:
    CharacterStyle(const string& font, int size, bool b, bool i,
                    int r, int g, int bColor)
        : fontFamily(font), fontSize(size), bold(b), italic(i),
          colorR(r), colorG(g), colorB(bColor) {}

    // Operation takes EXTRINSIC state as parameters
    void render(char symbol, int x, int y) const {
        cout << "Rendering '" << symbol << "' at (" << x << "," << y << ") "
             << "[" << fontFamily << " " << fontSize << "pt"
             << (bold ? " BOLD" : "") << (italic ? " ITALIC" : "")
             << " rgb(" << colorR << "," << colorG << "," << colorB << ")]"
             << endl;
    }

    // Key for caching — unique identifier for this style combination
    string getKey() const {
        return fontFamily + "_" + to_string(fontSize) + "_" +
               (bold ? "B" : "") + (italic ? "I" : "") + "_" +
               to_string(colorR) + "_" + to_string(colorG) + "_" + to_string(colorB);
    }

    // For debugging
    string describe() const {
        return fontFamily + " " + to_string(fontSize) + "pt" +
               (bold ? " Bold" : "") + (italic ? " Italic" : "");
    }
};

// ============================================================
// FLYWEIGHT FACTORY — creates and caches flyweight objects
// ============================================================

class CharacterStyleFactory {
    unordered_map<string, shared_ptr<CharacterStyle>> styles;

public:
    shared_ptr<CharacterStyle> getStyle(const string& font, int size,
                                         bool bold, bool italic,
                                         int r, int g, int b) {
        // Create key from parameters
        auto temp = CharacterStyle(font, size, bold, italic, r, g, b);
        string key = temp.getKey();

        // Return existing flyweight if available
        auto it = styles.find(key);
        if (it != styles.end()) {
            return it->second;
        }

        // Create new flyweight and cache it
        auto style = make_shared<CharacterStyle>(font, size, bold, italic, r, g, b);
        styles[key] = style;
        cout << "[Factory] Created new style: " << style->describe()
             << " (total styles: " << styles.size() << ")" << endl;
        return style;
    }

    int getStyleCount() const { return styles.size(); }

    void printStats() const {
        cout << "=== Flyweight Factory Stats ===" << endl;
        cout << "Unique styles cached: " << styles.size() << endl;
        for (const auto& [key, style] : styles) {
            cout << "  " << style->describe() << endl;
        }
    }
};

// ============================================================
// CONTEXT — stores extrinsic state + reference to flyweight
// ============================================================

struct CharacterContext {
    char symbol;                        // extrinsic: unique per character
    int x, y;                           // extrinsic: position
    shared_ptr<CharacterStyle> style;   // reference to shared flyweight
};

// ============================================================
// CLIENT — text editor that uses flyweights
// ============================================================

class TextEditor {
    vector<CharacterContext> characters;
    CharacterStyleFactory& factory;
    int cursorX = 0, cursorY = 0;
    int charWidth = 10;
    int lineHeight = 20;

public:
    TextEditor(CharacterStyleFactory& f) : factory(f) {}

    void type(const string& text, const string& font, int size,
              bool bold, bool italic, int r, int g, int b) {
        auto style = factory.getStyle(font, size, bold, italic, r, g, b);

        for (char c : text) {
            if (c == '\n') {
                cursorX = 0;
                cursorY += lineHeight;
                continue;
            }

            characters.push_back({c, cursorX, cursorY, style});
            cursorX += charWidth;
        }
    }

    void render() const {
        cout << "=== Rendering document ===" << endl;
        for (const auto& ch : characters) {
            ch.style->render(ch.symbol, ch.x, ch.y);
        }
    }

    void printMemoryStats() const {
        // Without flyweight: each character stores full style data (~50 bytes)
        size_t withoutFlyweight = characters.size() * (sizeof(char) + 2 * sizeof(int) + 50);

        // With flyweight: each character stores symbol + position + pointer (~21 bytes)
        size_t withFlyweight = characters.size() * (sizeof(char) + 2 * sizeof(int) + sizeof(shared_ptr<CharacterStyle>))
                              + factory.getStyleCount() * 50;

        cout << "=== Memory Stats ===" << endl;
        cout << "Total characters: " << characters.size() << endl;
        cout << "Unique styles: " << factory.getStyleCount() << endl;
        cout << "Memory without Flyweight: ~" << withoutFlyweight << " bytes" << endl;
        cout << "Memory with Flyweight: ~" << withFlyweight << " bytes" << endl;
        if (withoutFlyweight > 0) {
            cout << "Savings: ~" << (100 - (withFlyweight * 100 / withoutFlyweight)) << "%" << endl;
        }
    }
};

// Usage
int main() {
    CharacterStyleFactory factory;
    TextEditor editor(factory);

    // Type with different styles — factory reuses existing styles
    editor.type("Hello ", "Arial", 12, false, false, 0, 0, 0);
    editor.type("World", "Arial", 12, true, false, 0, 0, 0);
    editor.type("! ", "Arial", 12, false, false, 0, 0, 0);  // reuses first style
    editor.type("This is ", "Arial", 12, false, false, 0, 0, 0);  // reuses again
    editor.type("important", "Arial", 12, true, true, 255, 0, 0);  // new style
    editor.type(" text.", "Arial", 12, false, false, 0, 0, 0);  // reuses

    cout << endl;
    editor.render();

    cout << endl;
    editor.printMemoryStats();

    cout << endl;
    factory.printStats();
}
```

---

### Another Example — Game Forest with Tree Flyweights

A classic Flyweight example — rendering a forest with millions of trees where many share the same model/texture.

```cpp
// Flyweight — shared tree type data (intrinsic state)
class TreeType {
    string name;
    string texture;     // large texture data — shared!
    string meshData;    // 3D model data — shared!
    int baseColor;

public:
    TreeType(const string& name, const string& texture,
             const string& mesh, int color)
        : name(name), texture(texture), meshData(mesh), baseColor(color) {
        cout << "[TreeType] Loading '" << name << "' (texture: "
             << texture.size() << " bytes, mesh: " << mesh.size()
             << " bytes)" << endl;
    }

    void draw(int x, int y, float scale, float rotation) const {
        // In a real game, this would use the GPU
        cout << "Drawing " << name << " at (" << x << "," << y
             << ") scale=" << scale << " rot=" << rotation << "°" << endl;
    }

    string getName() const { return name; }
    size_t getMemorySize() const {
        return texture.size() + meshData.size() + name.size() + sizeof(int);
    }
};

// Flyweight Factory
class TreeFactory {
    unordered_map<string, shared_ptr<TreeType>> treeTypes;

public:
    shared_ptr<TreeType> getTreeType(const string& name, const string& texture,
                                      const string& mesh, int color) {
        auto it = treeTypes.find(name);
        if (it != treeTypes.end()) {
            return it->second;
        }

        auto type = make_shared<TreeType>(name, texture, mesh, color);
        treeTypes[name] = type;
        return type;
    }

    int getTypeCount() const { return treeTypes.size(); }

    size_t getTotalSharedMemory() const {
        size_t total = 0;
        for (const auto& [_, type] : treeTypes) {
            total += type->getMemorySize();
        }
        return total;
    }
};

// Context — individual tree instance (extrinsic state)
struct Tree {
    int x, y;                       // position — unique per tree
    float scale;                    // size variation — unique
    float rotation;                 // rotation — unique
    shared_ptr<TreeType> type;      // shared flyweight

    void draw() const {
        type->draw(x, y, scale, rotation);
    }
};

// Forest — manages all trees
class Forest {
    vector<Tree> trees;
    TreeFactory factory;

public:
    void plantTree(int x, int y, float scale, float rotation,
                    const string& name, const string& texture,
                    const string& mesh, int color) {
        auto type = factory.getTreeType(name, texture, mesh, color);
        trees.push_back({x, y, scale, rotation, type});
    }

    void render() const {
        for (const auto& tree : trees) {
            tree.draw();
        }
    }

    void printStats() const {
        cout << "=== Forest Stats ===" << endl;
        cout << "Total trees: " << trees.size() << endl;
        cout << "Unique tree types: " << factory.getTypeCount() << endl;

        size_t sharedMemory = factory.getTotalSharedMemory();
        size_t perTreeMemory = trees.size() * (sizeof(int) * 2 + sizeof(float) * 2 + sizeof(shared_ptr<TreeType>));
        size_t totalWithFlyweight = sharedMemory + perTreeMemory;

        // Without flyweight: each tree stores its own copy of texture/mesh
        size_t totalWithoutFlyweight = trees.size() * (sharedMemory / factory.getTypeCount()
                                       + sizeof(int) * 2 + sizeof(float) * 2);

        cout << "Shared memory (textures/meshes): " << sharedMemory << " bytes" << endl;
        cout << "Per-tree memory: " << perTreeMemory << " bytes" << endl;
        cout << "Total with Flyweight: " << totalWithFlyweight << " bytes" << endl;
        cout << "Total without Flyweight: ~" << totalWithoutFlyweight << " bytes" << endl;
    }
};

// Usage
int main() {
    Forest forest;

    // Simulate large texture/mesh data
    string oakTexture(100000, 'O');    // 100KB texture
    string oakMesh(50000, 'M');        // 50KB mesh
    string pineTexture(80000, 'P');
    string pineMesh(40000, 'M');
    string birchTexture(90000, 'B');
    string birchMesh(45000, 'M');

    // Plant 10,000 trees — only 3 tree types loaded!
    srand(42);
    for (int i = 0; i < 10000; i++) {
        int x = rand() % 1000;
        int y = rand() % 1000;
        float scale = 0.5f + (rand() % 100) / 100.0f;
        float rotation = rand() % 360;

        int type = rand() % 3;
        switch (type) {
            case 0: forest.plantTree(x, y, scale, rotation, "Oak", oakTexture, oakMesh, 0x228B22); break;
            case 1: forest.plantTree(x, y, scale, rotation, "Pine", pineTexture, pineMesh, 0x006400); break;
            case 2: forest.plantTree(x, y, scale, rotation, "Birch", birchTexture, birchMesh, 0xF5F5DC); break;
        }
    }

    forest.printStats();
    // 10,000 trees but only 3 texture/mesh sets loaded!
}
```

---

### Flyweight Factory

The factory is essential — it ensures flyweight objects are shared, not duplicated:

```cpp
// The factory MUST:
// 1. Check if a flyweight with the given intrinsic state already exists
// 2. Return the existing one if found
// 3. Create a new one only if not found
// 4. Store the new one for future reuse

// Common implementation: HashMap<Key, Flyweight>
// Key = hash of intrinsic state
// Value = shared_ptr to flyweight object
```

**Without the factory, clients might create duplicate flyweights, defeating the purpose.**

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

**Real-world examples:**
- Text editors (character formatting)
- Game engines (particle systems, tile maps, sprite rendering)
- String interning (`std::string` small string optimization, Java String pool)
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

```cpp
// BAD: Loading a 500MB image just to display its filename
class HugeImage {
public:
    HugeImage(const string& path) {
        // Loads 500MB into memory immediately!
        loadFromDisk(path);  // takes 5 seconds
    }
    void display() { /* render the image */ }
    string getFilename() { return filename; }
};

// User opens a folder with 1000 images
// Without proxy: loads 500GB into memory, takes 83 minutes
// With proxy: loads nothing until user actually views an image
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
| **Smart Reference** | Additional actions on access (ref counting, locking) | `shared_ptr`, `unique_ptr` |

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

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <chrono>
#include <thread>

using namespace std;

// Subject interface
class Image {
public:
    virtual void display() = 0;
    virtual int getWidth() const = 0;
    virtual int getHeight() const = 0;
    virtual string getFilename() const = 0;
    virtual size_t getMemoryUsage() const = 0;
    virtual ~Image() = default;
};

// Real Subject — expensive to create
class HighResImage : public Image {
    string filename;
    int width, height;
    vector<char> pixelData;  // large image data

public:
    HighResImage(const string& fname) : filename(fname) {
        cout << "[HighResImage] Loading '" << fname << "' from disk..." << endl;
        // Simulate expensive loading
        this_thread::sleep_for(chrono::milliseconds(500));
        width = 3840;
        height = 2160;
        pixelData.resize(width * height * 4);  // RGBA, ~33MB
        cout << "[HighResImage] Loaded! (" << pixelData.size() / (1024*1024)
             << " MB)" << endl;
    }

    void display() override {
        cout << "[HighResImage] Displaying '" << filename << "' ("
             << width << "x" << height << ")" << endl;
    }

    int getWidth() const override { return width; }
    int getHeight() const override { return height; }
    string getFilename() const override { return filename; }
    size_t getMemoryUsage() const override { return pixelData.size(); }
};

// Virtual Proxy — lazy loads the real image
class ImageProxy : public Image {
    string filename;
    mutable unique_ptr<HighResImage> realImage;  // created on demand

    // Lightweight metadata (loaded cheaply without full image)
    int cachedWidth = 0;
    int cachedHeight = 0;

    void ensureLoaded() const {
        if (!realImage) {
            realImage = make_unique<HighResImage>(filename);
        }
    }

public:
    ImageProxy(const string& fname) : filename(fname) {
        // Only store the filename — don't load the image!
        // Could load lightweight metadata here (dimensions from EXIF, etc.)
        cout << "[ImageProxy] Created proxy for '" << fname
             << "' (no data loaded)" << endl;
    }

    void display() override {
        ensureLoaded();  // load only when actually displaying
        realImage->display();
    }

    int getWidth() const override {
        ensureLoaded();
        return realImage->getWidth();
    }

    int getHeight() const override {
        ensureLoaded();
        return realImage->getHeight();
    }

    string getFilename() const override {
        return filename;  // no need to load for this!
    }

    size_t getMemoryUsage() const override {
        if (!realImage) return sizeof(ImageProxy);  // just the proxy
        return realImage->getMemoryUsage() + sizeof(ImageProxy);
    }

    bool isLoaded() const { return realImage != nullptr; }
};

// Client code — doesn't know it's using a proxy
void showGallery(vector<unique_ptr<Image>>& images) {
    cout << "=== Image Gallery ===" << endl;
    for (const auto& img : images) {
        cout << "  " << img->getFilename()
             << " (memory: " << img->getMemoryUsage() << " bytes)" << endl;
    }

    // Only load the image the user clicks on
    cout << endl << "User clicks on image 2:" << endl;
    images[1]->display();  // only THIS image gets loaded!
}

int main() {
    vector<unique_ptr<Image>> gallery;

    // Create proxies for 5 images — none are loaded yet!
    gallery.push_back(make_unique<ImageProxy>("sunset.jpg"));
    gallery.push_back(make_unique<ImageProxy>("mountain.jpg"));
    gallery.push_back(make_unique<ImageProxy>("ocean.jpg"));
    gallery.push_back(make_unique<ImageProxy>("forest.jpg"));
    gallery.push_back(make_unique<ImageProxy>("city.jpg"));

    showGallery(gallery);
    // Only mountain.jpg is loaded — the other 4 are still just proxies!
}
```

---

### Protection Proxy — Access Control

Controls access to the real object based on permissions.

```cpp
// Subject interface
class Document {
public:
    virtual string read() const = 0;
    virtual void write(const string& content) = 0;
    virtual void deleteDoc() = 0;
    virtual string getName() const = 0;
    virtual ~Document() = default;
};

// Real Subject
class RealDocument : public Document {
    string name;
    string content;

public:
    RealDocument(const string& n, const string& c) : name(n), content(c) {}

    string read() const override {
        return content;
    }

    void write(const string& c) override {
        content = c;
        cout << "[Document] '" << name << "' updated" << endl;
    }

    void deleteDoc() override {
        content.clear();
        cout << "[Document] '" << name << "' deleted" << endl;
    }

    string getName() const override { return name; }
};

// User with roles
class User {
public:
    enum class Role { VIEWER, EDITOR, ADMIN };

    string name;
    Role role;

    User(const string& n, Role r) : name(n), role(r) {}

    string getRoleName() const {
        switch (role) {
            case Role::VIEWER: return "Viewer";
            case Role::EDITOR: return "Editor";
            case Role::ADMIN:  return "Admin";
        }
        return "Unknown";
    }
};

// Protection Proxy — checks permissions before delegating
class DocumentProxy : public Document {
    unique_ptr<RealDocument> realDoc;
    User& currentUser;

    void checkPermission(const string& operation, User::Role minRole) const {
        if (currentUser.role < minRole) {
            throw runtime_error(
                "ACCESS DENIED: User '" + currentUser.name +
                "' (" + currentUser.getRoleName() +
                ") cannot " + operation + " document '" + realDoc->getName() + "'"
            );
        }
    }

public:
    DocumentProxy(unique_ptr<RealDocument> doc, User& user)
        : realDoc(std::move(doc)), currentUser(user) {}

    string read() const override {
        checkPermission("read", User::Role::VIEWER);
        cout << "[Proxy] " << currentUser.name << " reading '"
             << realDoc->getName() << "'" << endl;
        return realDoc->read();
    }

    void write(const string& content) override {
        checkPermission("write", User::Role::EDITOR);
        cout << "[Proxy] " << currentUser.name << " writing to '"
             << realDoc->getName() << "'" << endl;
        realDoc->write(content);
    }

    void deleteDoc() override {
        checkPermission("delete", User::Role::ADMIN);
        cout << "[Proxy] " << currentUser.name << " deleting '"
             << realDoc->getName() << "'" << endl;
        realDoc->deleteDoc();
    }

    string getName() const override { return realDoc->getName(); }
};

// Usage
int main() {
    User viewer("Alice", User::Role::VIEWER);
    User editor("Bob", User::Role::EDITOR);
    User admin("Charlie", User::Role::ADMIN);

    auto doc = make_unique<RealDocument>("Secret Plan", "Top secret content...");

    // Viewer can read
    DocumentProxy viewerProxy(make_unique<RealDocument>("Secret Plan", "Top secret content..."), viewer);
    cout << viewerProxy.read() << endl;

    // Viewer cannot write
    try {
        viewerProxy.write("Hacked!");
    } catch (const runtime_error& e) {
        cout << e.what() << endl;
    }

    // Editor can write
    DocumentProxy editorProxy(make_unique<RealDocument>("Secret Plan", "Top secret content..."), editor);
    editorProxy.write("Updated content");

    // Editor cannot delete
    try {
        editorProxy.deleteDoc();
    } catch (const runtime_error& e) {
        cout << e.what() << endl;
    }

    // Admin can do everything
    DocumentProxy adminProxy(make_unique<RealDocument>("Secret Plan", "Updated content"), admin);
    adminProxy.write("Final version");
    adminProxy.deleteDoc();
}
```

---

### Caching Proxy

Caches results of expensive operations to avoid redundant computation.

```cpp
#include <unordered_map>
#include <chrono>

// Subject interface
class WeatherService {
public:
    virtual string getWeather(const string& city) = 0;
    virtual double getTemperature(const string& city) = 0;
    virtual ~WeatherService() = default;
};

// Real Subject — makes expensive API calls
class RealWeatherService : public WeatherService {
public:
    string getWeather(const string& city) override {
        cout << "[API] Fetching weather for " << city << " (slow network call)..." << endl;
        this_thread::sleep_for(chrono::milliseconds(200));  // simulate latency
        return "Sunny, 25°C";
    }

    double getTemperature(const string& city) override {
        cout << "[API] Fetching temperature for " << city << " (slow network call)..." << endl;
        this_thread::sleep_for(chrono::milliseconds(200));
        return 25.0;
    }
};

// Caching Proxy
class CachingWeatherProxy : public WeatherService {
    unique_ptr<WeatherService> realService;

    struct CacheEntry {
        string data;
        chrono::steady_clock::time_point timestamp;
    };

    unordered_map<string, CacheEntry> weatherCache;
    unordered_map<string, pair<double, chrono::steady_clock::time_point>> tempCache;
    chrono::seconds cacheTTL;
    int cacheHits = 0;
    int cacheMisses = 0;

    bool isCacheValid(chrono::steady_clock::time_point timestamp) const {
        auto age = chrono::steady_clock::now() - timestamp;
        return age < cacheTTL;
    }

public:
    CachingWeatherProxy(unique_ptr<WeatherService> service,
                         chrono::seconds ttl = chrono::seconds(300))
        : realService(std::move(service)), cacheTTL(ttl) {}

    string getWeather(const string& city) override {
        auto it = weatherCache.find(city);
        if (it != weatherCache.end() && isCacheValid(it->second.timestamp)) {
            cacheHits++;
            cout << "[Cache HIT] Weather for " << city << endl;
            return it->second.data;
        }

        cacheMisses++;
        cout << "[Cache MISS] Weather for " << city << endl;
        string result = realService->getWeather(city);
        weatherCache[city] = {result, chrono::steady_clock::now()};
        return result;
    }

    double getTemperature(const string& city) override {
        auto it = tempCache.find(city);
        if (it != tempCache.end() && isCacheValid(it->second.second)) {
            cacheHits++;
            cout << "[Cache HIT] Temperature for " << city << endl;
            return it->second.first;
        }

        cacheMisses++;
        cout << "[Cache MISS] Temperature for " << city << endl;
        double result = realService->getTemperature(city);
        tempCache[city] = {result, chrono::steady_clock::now()};
        return result;
    }

    void clearCache() {
        weatherCache.clear();
        tempCache.clear();
        cout << "[Cache] Cleared" << endl;
    }

    void printStats() const {
        cout << "=== Cache Stats ===" << endl;
        cout << "Hits: " << cacheHits << ", Misses: " << cacheMisses << endl;
        int total = cacheHits + cacheMisses;
        if (total > 0) {
            cout << "Hit rate: " << (cacheHits * 100 / total) << "%" << endl;
        }
    }
};

// Usage
int main() {
    auto realService = make_unique<RealWeatherService>();
    CachingWeatherProxy weather(std::move(realService), chrono::seconds(60));

    // First call — cache miss, hits the real API
    cout << weather.getWeather("London") << endl;

    // Second call — cache hit, instant!
    cout << weather.getWeather("London") << endl;

    // Different city — cache miss
    cout << weather.getWeather("Tokyo") << endl;

    // London again — still cached
    cout << weather.getWeather("London") << endl;

    weather.printStats();
}
```

---

### Logging Proxy

Records all operations for auditing.

```cpp
class DatabaseService {
public:
    virtual string query(const string& sql) = 0;
    virtual int execute(const string& sql) = 0;
    virtual ~DatabaseService() = default;
};

class RealDatabase : public DatabaseService {
public:
    string query(const string& sql) override {
        cout << "[DB] Executing query: " << sql << endl;
        return "result_data";
    }

    int execute(const string& sql) override {
        cout << "[DB] Executing: " << sql << endl;
        return 1;  // rows affected
    }
};

class LoggingDatabaseProxy : public DatabaseService {
    unique_ptr<DatabaseService> realDb;
    vector<string> auditLog;

    string getTimestamp() const {
        auto now = chrono::system_clock::now();
        auto time = chrono::system_clock::to_time_t(now);
        string ts = ctime(&time);
        ts.pop_back();  // remove newline
        return ts;
    }

public:
    LoggingDatabaseProxy(unique_ptr<DatabaseService> db)
        : realDb(std::move(db)) {}

    string query(const string& sql) override {
        string logEntry = getTimestamp() + " | QUERY | " + sql;
        auditLog.push_back(logEntry);
        cout << "[LOG] " << logEntry << endl;

        auto start = chrono::steady_clock::now();
        string result = realDb->query(sql);
        auto end = chrono::steady_clock::now();

        auto duration = chrono::duration_cast<chrono::milliseconds>(end - start);
        cout << "[LOG] Query completed in " << duration.count() << "ms" << endl;

        return result;
    }

    int execute(const string& sql) override {
        string logEntry = getTimestamp() + " | EXECUTE | " + sql;
        auditLog.push_back(logEntry);
        cout << "[LOG] " << logEntry << endl;

        auto start = chrono::steady_clock::now();
        int result = realDb->execute(sql);
        auto end = chrono::steady_clock::now();

        auto duration = chrono::duration_cast<chrono::milliseconds>(end - start);
        cout << "[LOG] Execution completed in " << duration.count()
             << "ms, " << result << " rows affected" << endl;

        return result;
    }

    void printAuditLog() const {
        cout << "=== Audit Log ===" << endl;
        for (const auto& entry : auditLog) {
            cout << "  " << entry << endl;
        }
        cout << "Total operations: " << auditLog.size() << endl;
    }
};

// Usage
int main() {
    auto db = make_unique<RealDatabase>();
    LoggingDatabaseProxy loggedDb(std::move(db));

    loggedDb.query("SELECT * FROM users WHERE active = true");
    loggedDb.execute("UPDATE users SET last_login = NOW() WHERE id = 42");
    loggedDb.query("SELECT COUNT(*) FROM orders");

    cout << endl;
    loggedDb.printAuditLog();
}
```

---

### Smart Pointer as Proxy

C++ smart pointers (`shared_ptr`, `unique_ptr`) are a form of the Proxy pattern — they control access to raw pointers and manage their lifecycle.

```cpp
// shared_ptr is a proxy for raw pointers:
// - Same interface (operator*, operator->)
// - Controls access (reference counting)
// - Manages lifecycle (automatic deletion)

class Resource {
public:
    void use() { cout << "Using resource" << endl; }
    ~Resource() { cout << "Resource destroyed" << endl; }
};

// Raw pointer — no proxy, manual management
Resource* raw = new Resource();
raw->use();
delete raw;  // must remember to delete!

// shared_ptr — proxy with reference counting
shared_ptr<Resource> smart = make_shared<Resource>();
smart->use();  // same interface as raw pointer
// Automatically deleted when last shared_ptr goes out of scope

// Custom smart pointer (simplified proxy)
template <typename T>
class SmartProxy {
    T* ptr;
    int* refCount;

public:
    SmartProxy(T* p) : ptr(p), refCount(new int(1)) {}

    SmartProxy(const SmartProxy& other)
        : ptr(other.ptr), refCount(other.refCount) {
        (*refCount)++;
    }

    ~SmartProxy() {
        (*refCount)--;
        if (*refCount == 0) {
            delete ptr;
            delete refCount;
        }
    }

    T& operator*() { return *ptr; }
    T* operator->() { return ptr; }
    int getRefCount() const { return *refCount; }
};
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

```cpp
class SmartDocumentProxy : public Document {
    mutable unique_ptr<RealDocument> realDoc;  // virtual proxy (lazy)
    string filename;
    User& user;
    mutable unordered_map<string, string> cache;  // caching proxy
    mutable vector<string> accessLog;  // logging proxy

    void ensureLoaded() const {
        if (!realDoc) {
            cout << "[Proxy] Lazy loading document..." << endl;
            realDoc = make_unique<RealDocument>(filename, "loaded content");
        }
    }

    void checkAccess(const string& operation) const {
        accessLog.push_back(user.name + " " + operation + " " + filename);
        if (user.role < User::Role::VIEWER) {
            throw runtime_error("Access denied");
        }
    }

public:
    SmartDocumentProxy(const string& fname, User& u)
        : filename(fname), user(u) {}

    string read() const override {
        checkAccess("read");       // protection proxy
        ensureLoaded();            // virtual proxy

        auto it = cache.find("content");
        if (it != cache.end()) {
            cout << "[Proxy] Cache hit" << endl;
            return it->second;     // caching proxy
        }

        string content = realDoc->read();
        cache["content"] = content;
        return content;
    }

    // ... other methods combine all three proxy behaviors
};
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
| Smart Reference | Resource management | Automatic cleanup (RAII) |

**When NOT to use Proxy:**
- Object is cheap to create (virtual proxy adds unnecessary complexity)
- No access control needed (protection proxy is overhead)
- Direct access is fine (proxy adds indirection)
- Performance-critical path (proxy adds a layer of indirection)

**Real-world examples:**
- Smart pointers (`shared_ptr`, `unique_ptr`) — smart reference proxy
- ORM lazy loading (Hibernate proxies) — virtual proxy
- Spring AOP proxies — logging/transaction proxy
- Network stubs (gRPC, CORBA) — remote proxy
- CDN (Content Delivery Network) — caching proxy
- Firewall/API Gateway — protection proxy

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

1. **Adapter: Class vs Object** — Know both approaches. Object Adapter (composition) is preferred in most cases. Class Adapter (multiple inheritance) is C++-specific. Explain trade-offs: flexibility vs direct access.

2. **Adapter vs Bridge** — This is the #1 structural pattern comparison question. Adapter is a retrofit (makes existing incompatible things work). Bridge is a design decision (separates two dimensions from the start). Use the "designed up-front vs after-the-fact" distinction.

3. **Bridge: Class explosion** — Be ready to explain the N × M problem. Without Bridge, combining N abstractions with M implementations requires N × M classes. With Bridge, you need N + M. Draw the two hierarchies.

4. **Composite: Transparency vs Safety** — Know the trade-off. Transparency: `add()`/`remove()` in Component (uniform but risky). Safety: `add()`/`remove()` only in Composite (safe but requires type checking). GoF favors transparency; modern practice favors safety.

5. **Decorator: Stacking** — Explain how decorators compose. Each decorator wraps the previous one. Order matters (encrypt-then-compress ≠ compress-then-encrypt). Same decorator can be applied multiple times. Draw the call chain.

6. **Decorator vs Inheritance** — Decorator is runtime, flexible, combinatorial. Inheritance is compile-time, fixed, leads to class explosion. Use Decorator when you need arbitrary combinations of features.

7. **Facade: Not a restriction** — Emphasize that Facade doesn't hide the subsystem — clients can still access subsystem classes directly. Facade is an additional convenience layer, not a wall.

8. **Flyweight: Intrinsic vs Extrinsic** — This is the core concept. Intrinsic state is shared and immutable (stored in flyweight). Extrinsic state is unique and varies (passed by client). The split determines memory savings. Be ready to identify intrinsic vs extrinsic in a given scenario.

9. **Proxy types** — Know all five types (Virtual, Protection, Remote, Caching, Logging). Be ready to implement at least Virtual and Protection proxies. Mention `shared_ptr` as a real-world Smart Reference proxy.

10. **Proxy vs Decorator** — Another top comparison. Proxy controls access (same object, controlled). Decorator adds behavior (enhanced object). Proxy often creates the real object; Decorator receives it. Proxy is usually single; Decorators stack.

11. **Composite + Decorator** — These patterns combine naturally. A Composite tree where nodes can be decorated. Example: UI components (Composite tree) with borders and scrollbars (Decorators).

12. **Flyweight + Composite** — Another natural combination. Composite tree with Flyweight leaf nodes. Example: Document with shared character styles (Flyweight) in a paragraph tree (Composite).

13. **Real-world examples** — For each pattern, know at least 2-3 real-world examples:
    - Adapter: Payment gateways, legacy system wrappers, third-party library integration
    - Bridge: Cross-platform UI, database drivers, rendering engines
    - Composite: File systems, UI trees, HTML DOM, organization charts
    - Decorator: Java I/O streams, middleware pipelines, game power-ups
    - Facade: jQuery, Spring JdbcTemplate, compiler front-ends
    - Flyweight: Text editors, game particle systems, string interning
    - Proxy: Smart pointers, ORM lazy loading, CDN, API gateways

14. **Memory management** — In C++, use `unique_ptr` for exclusive ownership (Decorator wrapping, Proxy holding real subject). Use `shared_ptr` for shared ownership (Flyweight objects, Composite children). Always think about who owns what.
