# Module 13: LLD Practice Problems (Medium)

> This module raises the bar from Module 12 with more complex, real-world systems. Each problem involves multiple interacting entities, non-trivial business rules, concurrency considerations, and requires combining several design patterns. These are the problems most commonly asked in LLD interviews at mid-level companies (and are the bread-and-butter of FAANG LLD rounds). Every problem follows the same structured approach: requirements, key entities, class design, full C++ implementation, design patterns used, and key design decisions.

---

## 13.1 Library Management System

> Design a library management system that supports books, members, librarians, issuing/returning/reserving books, fine calculation, and search functionality. This problem tests your ability to model real-world entities with multiple relationships and business rules.

---

### Requirements

**Functional Requirements:**
1. The library has a catalog of books, each identified by ISBN
2. Multiple copies of the same book can exist (each copy has a unique barcode)
3. Members can search for books by title, author, or ISBN
4. Members can issue (check out) a book copy for a fixed period (14 days)
5. Members can return a book copy
6. Members can reserve a book if all copies are currently issued
7. When a reserved book is returned, the member who reserved it is notified
8. A member can hold at most 5 books at a time
9. Overdue books incur a fine ($1 per day)
10. Librarians can add/remove books and manage members

**Non-Functional Requirements:**
- Thread-safe for concurrent issue/return operations
- Extensible for new search criteria or fine calculation strategies

---

### Key Entities & Class Design

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   Library     │──────▶│    Book      │──────▶│  BookCopy    │
│  (Singleton)  │       │  (catalog)   │       │  (physical)  │
└──────────────┘       └──────────────┘       └──────────────┘
       │                                             │
       │                                             │ issued to
       ▼                                             ▼
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   Member     │◀─────▶│   Loan       │       │ Reservation  │
│              │       │              │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
       │                      │
       │                      ▼
┌──────────────┐       ┌──────────────┐
│  Librarian   │       │FineStrategy  │
└──────────────┘       └──────────────┘
```

---

### Implementation

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <unordered_set>
#include <memory>
#include <optional>
#include <algorithm>
#include <chrono>
#include <mutex>
#include <queue>
#include <functional>
using namespace std;

// ─── Enums ───────────────────────────────────────────────

enum class BookCopyStatus { AVAILABLE, ISSUED, RESERVED, LOST };
enum class AccountStatus { ACTIVE, SUSPENDED, CLOSED };

string to_string(BookCopyStatus s) {
    switch (s) {
        case BookCopyStatus::AVAILABLE: return "Available";
        case BookCopyStatus::ISSUED:    return "Issued";
        case BookCopyStatus::RESERVED:  return "Reserved";
        case BookCopyStatus::LOST:      return "Lost";
    }
    return "Unknown";
}

// ─── Date Helper ─────────────────────────────────────────

using Date = chrono::system_clock::time_point;

Date now() { return chrono::system_clock::now(); }

Date days_from_now(int days) {
    return now() + chrono::hours(24 * days);
}

int days_between(Date from, Date to) {
    auto diff = chrono::duration_cast<chrono::hours>(to - from);
    return diff.count() / 24;
}

// ─── Fine Strategy (Strategy Pattern) ────────────────────

class FineStrategy {
public:
    virtual ~FineStrategy() = default;
    virtual double calculate(int overdue_days) const = 0;
    virtual string name() const = 0;
};

class FlatFineStrategy : public FineStrategy {
    double rate_per_day;
public:
    FlatFineStrategy(double rate = 1.0) : rate_per_day(rate) {}

    double calculate(int overdue_days) const override {
        if (overdue_days <= 0) return 0.0;
        return overdue_days * rate_per_day;
    }

    string name() const override { return "Flat ($" + to_string(rate_per_day) + "/day)"; }
};

class TieredFineStrategy : public FineStrategy {
public:
    double calculate(int overdue_days) const override {
        if (overdue_days <= 0) return 0.0;
        // First 7 days: $1/day, after that: $2/day
        if (overdue_days <= 7) return overdue_days * 1.0;
        return 7.0 + (overdue_days - 7) * 2.0;
    }

    string name() const override { return "Tiered ($1/day first week, $2/day after)"; }
};

// ─── Book (catalog entry) ────────────────────────────────

class Book {
    string isbn;
    string title;
    string author;
    string publisher;
    int year;

public:
    Book(const string& isbn, const string& title, const string& author,
         const string& publisher, int year)
        : isbn(isbn), title(title), author(author),
          publisher(publisher), year(year) {}

    const string& get_isbn() const { return isbn; }
    const string& get_title() const { return title; }
    const string& get_author() const { return author; }
    const string& get_publisher() const { return publisher; }
    int get_year() const { return year; }

    void print() const {
        cout << "\"" << title << "\" by " << author
             << " (ISBN: " << isbn << ", " << year << ")" << endl;
    }
};

// ─── BookCopy (physical copy) ────────────────────────────

class BookCopy {
    string barcode;
    string isbn;  // references the Book catalog entry
    BookCopyStatus status;

public:
    BookCopy(const string& barcode, const string& isbn)
        : barcode(barcode), isbn(isbn), status(BookCopyStatus::AVAILABLE) {}

    const string& get_barcode() const { return barcode; }
    const string& get_isbn() const { return isbn; }
    BookCopyStatus get_status() const { return status; }
    bool is_available() const { return status == BookCopyStatus::AVAILABLE; }

    void set_status(BookCopyStatus s) { status = s; }
};

// ─── Loan (tracks an issued book) ────────────────────────

class Loan {
    string barcode;       // which copy
    string member_id;     // who borrowed it
    Date issue_date;
    Date due_date;
    optional<Date> return_date;

public:
    Loan(const string& barcode, const string& member_id,
         Date issued, Date due)
        : barcode(barcode), member_id(member_id),
          issue_date(issued), due_date(due) {}

    const string& get_barcode() const { return barcode; }
    const string& get_member_id() const { return member_id; }
    Date get_issue_date() const { return issue_date; }
    Date get_due_date() const { return due_date; }
    optional<Date> get_return_date() const { return return_date; }

    bool is_returned() const { return return_date.has_value(); }

    void mark_returned() { return_date = now(); }

    int overdue_days() const {
        Date end = return_date.value_or(now());
        int diff = days_between(due_date, end);
        return max(0, diff);
    }
};

// ─── Reservation ─────────────────────────────────────────

class Reservation {
    string isbn;
    string member_id;
    Date reserved_date;

public:
    Reservation(const string& isbn, const string& member_id)
        : isbn(isbn), member_id(member_id), reserved_date(now()) {}

    const string& get_isbn() const { return isbn; }
    const string& get_member_id() const { return member_id; }
    Date get_reserved_date() const { return reserved_date; }
};

// ─── Observer Interface (for notifications) ──────────────

class LibraryObserver {
public:
    virtual ~LibraryObserver() = default;
    virtual void on_book_available(const string& isbn, const string& member_id) = 0;
};

class ConsoleNotifier : public LibraryObserver {
public:
    void on_book_available(const string& isbn, const string& member_id) override {
        cout << "📬 NOTIFICATION: Member " << member_id
             << " — your reserved book (ISBN: " << isbn
             << ") is now available!" << endl;
    }
};

// ─── Member ──────────────────────────────────────────────

class Member {
    string member_id;
    string name;
    string email;
    AccountStatus status;
    unordered_set<string> active_barcodes;  // currently held book barcodes
    double total_fines = 0.0;

    static const int MAX_BOOKS = 5;

public:
    Member(const string& id, const string& name, const string& email)
        : member_id(id), name(name), email(email), status(AccountStatus::ACTIVE) {}

    const string& get_id() const { return member_id; }
    const string& get_name() const { return name; }
    const string& get_email() const { return email; }
    AccountStatus get_status() const { return status; }
    int books_held() const { return active_barcodes.size(); }
    double get_total_fines() const { return total_fines; }

    bool can_issue() const {
        return status == AccountStatus::ACTIVE
               && (int)active_barcodes.size() < MAX_BOOKS;
    }

    void add_book(const string& barcode) {
        active_barcodes.insert(barcode);
    }

    void remove_book(const string& barcode) {
        active_barcodes.erase(barcode);
    }

    bool has_book(const string& barcode) const {
        return active_barcodes.count(barcode) > 0;
    }

    void add_fine(double amount) { total_fines += amount; }

    void set_status(AccountStatus s) { status = s; }

    void print() const {
        cout << "Member: " << name << " (ID: " << member_id
             << ") | Books held: " << active_barcodes.size()
             << "/" << MAX_BOOKS
             << " | Fines: $" << total_fines << endl;
    }
};

// ─── Library (Singleton, Facade) ─────────────────────────

class Library {
    string name;

    // Catalog: ISBN → Book
    unordered_map<string, Book> catalog;

    // Physical copies: barcode → BookCopy
    unordered_map<string, BookCopy> copies;

    // ISBN → list of barcodes
    unordered_map<string, vector<string>> isbn_to_barcodes;

    // Members: member_id → Member
    unordered_map<string, Member> members;

    // Active loans: barcode → Loan
    unordered_map<string, Loan> active_loans;

    // Loan history (for records)
    vector<Loan> loan_history;

    // Reservations: ISBN → queue of member_ids
    unordered_map<string, queue<string>> reservations;

    // Fine strategy
    unique_ptr<FineStrategy> fine_strategy;

    // Observers
    vector<shared_ptr<LibraryObserver>> observers;

    mutable mutex mtx;

    Library() : fine_strategy(make_unique<FlatFineStrategy>(1.0)) {}

    void notify_book_available(const string& isbn, const string& member_id) {
        for (auto& obs : observers) {
            obs->on_book_available(isbn, member_id);
        }
    }

public:
    static Library& instance() {
        static Library lib;
        return lib;
    }

    void set_name(const string& n) { name = n; }

    void add_observer(shared_ptr<LibraryObserver> obs) {
        observers.push_back(obs);
    }

    void set_fine_strategy(unique_ptr<FineStrategy> strategy) {
        lock_guard<mutex> lock(mtx);
        fine_strategy = std::move(strategy);
    }

    // ─── Catalog Management ──────────────────────────────

    void add_book(const Book& book, int num_copies) {
        lock_guard<mutex> lock(mtx);
        string isbn = book.get_isbn();
        catalog.emplace(isbn, book);

        for (int i = 0; i < num_copies; i++) {
            string barcode = isbn + "-" + to_string(isbn_to_barcodes[isbn].size() + 1);
            copies.emplace(barcode, BookCopy(barcode, isbn));
            isbn_to_barcodes[isbn].push_back(barcode);
        }

        cout << "Added " << num_copies << " copies of \"" << book.get_title()
             << "\" (ISBN: " << isbn << ")" << endl;
    }

    void add_member(const Member& member) {
        lock_guard<mutex> lock(mtx);
        members.emplace(member.get_id(), member);
        cout << "Registered member: " << member.get_name()
             << " (ID: " << member.get_id() << ")" << endl;
    }

    // ─── Search ──────────────────────────────────────────

    vector<const Book*> search_by_title(const string& query) const {
        lock_guard<mutex> lock(mtx);
        vector<const Book*> results;
        string lower_query = query;
        transform(lower_query.begin(), lower_query.end(),
                  lower_query.begin(), ::tolower);

        for (const auto& [isbn, book] : catalog) {
            string lower_title = book.get_title();
            transform(lower_title.begin(), lower_title.end(),
                      lower_title.begin(), ::tolower);
            if (lower_title.find(lower_query) != string::npos) {
                results.push_back(&book);
            }
        }
        return results;
    }

    vector<const Book*> search_by_author(const string& query) const {
        lock_guard<mutex> lock(mtx);
        vector<const Book*> results;
        string lower_query = query;
        transform(lower_query.begin(), lower_query.end(),
                  lower_query.begin(), ::tolower);

        for (const auto& [isbn, book] : catalog) {
            string lower_author = book.get_author();
            transform(lower_author.begin(), lower_author.end(),
                      lower_author.begin(), ::tolower);
            if (lower_author.find(lower_query) != string::npos) {
                results.push_back(&book);
            }
        }
        return results;
    }

    optional<const Book*> search_by_isbn(const string& isbn) const {
        lock_guard<mutex> lock(mtx);
        auto it = catalog.find(isbn);
        if (it != catalog.end()) return &it->second;
        return nullopt;
    }

    // ─── Issue Book ──────────────────────────────────────

    bool issue_book(const string& member_id, const string& isbn) {
        lock_guard<mutex> lock(mtx);

        // Validate member
        auto mem_it = members.find(member_id);
        if (mem_it == members.end()) {
            cout << "Error: Member " << member_id << " not found." << endl;
            return false;
        }
        Member& member = mem_it->second;

        if (!member.can_issue()) {
            cout << "Error: Member " << member.get_name()
                 << " cannot issue more books (limit reached or account suspended)." << endl;
            return false;
        }

        // Find an available copy
        auto barcode_it = isbn_to_barcodes.find(isbn);
        if (barcode_it == isbn_to_barcodes.end()) {
            cout << "Error: Book with ISBN " << isbn << " not in catalog." << endl;
            return false;
        }

        BookCopy* available_copy = nullptr;
        for (const auto& barcode : barcode_it->second) {
            auto& copy = copies.at(barcode);
            if (copy.is_available()) {
                available_copy = &copy;
                break;
            }
        }

        if (!available_copy) {
            cout << "No available copies of ISBN " << isbn
                 << ". Consider reserving." << endl;
            return false;
        }

        // Issue the book
        available_copy->set_status(BookCopyStatus::ISSUED);
        member.add_book(available_copy->get_barcode());

        Loan loan(available_copy->get_barcode(), member_id,
                  now(), days_from_now(14));
        active_loans.emplace(available_copy->get_barcode(), loan);

        const Book& book = catalog.at(isbn);
        cout << "✅ Issued \"" << book.get_title() << "\" (barcode: "
             << available_copy->get_barcode() << ") to " << member.get_name()
             << ". Due in 14 days." << endl;

        return true;
    }

    // ─── Return Book ─────────────────────────────────────

    double return_book(const string& member_id, const string& barcode) {
        lock_guard<mutex> lock(mtx);

        // Validate member
        auto mem_it = members.find(member_id);
        if (mem_it == members.end()) {
            cout << "Error: Member " << member_id << " not found." << endl;
            return -1;
        }
        Member& member = mem_it->second;

        if (!member.has_book(barcode)) {
            cout << "Error: Member " << member.get_name()
                 << " does not hold book " << barcode << endl;
            return -1;
        }

        // Find the loan
        auto loan_it = active_loans.find(barcode);
        if (loan_it == active_loans.end()) {
            cout << "Error: No active loan for barcode " << barcode << endl;
            return -1;
        }

        Loan& loan = loan_it->second;
        loan.mark_returned();

        // Calculate fine
        int overdue = loan.overdue_days();
        double fine = fine_strategy->calculate(overdue);

        // Update copy status
        auto& copy = copies.at(barcode);
        copy.set_status(BookCopyStatus::AVAILABLE);

        // Update member
        member.remove_book(barcode);
        if (fine > 0) {
            member.add_fine(fine);
        }

        // Move loan to history
        loan_history.push_back(loan);
        active_loans.erase(loan_it);

        string isbn = copy.get_isbn();
        const Book& book = catalog.at(isbn);

        cout << "✅ Returned \"" << book.get_title() << "\" (barcode: "
             << barcode << ") by " << member.get_name();
        if (overdue > 0) {
            cout << " | Overdue by " << overdue << " days | Fine: $" << fine;
        }
        cout << endl;

        // Check reservations
        check_reservations(isbn);

        return fine;
    }

    // ─── Reserve Book ────────────────────────────────────

    bool reserve_book(const string& member_id, const string& isbn) {
        lock_guard<mutex> lock(mtx);

        auto mem_it = members.find(member_id);
        if (mem_it == members.end()) {
            cout << "Error: Member " << member_id << " not found." << endl;
            return false;
        }

        if (catalog.find(isbn) == catalog.end()) {
            cout << "Error: Book with ISBN " << isbn << " not in catalog." << endl;
            return false;
        }

        // Check if any copy is available (no need to reserve)
        bool has_available = false;
        for (const auto& barcode : isbn_to_barcodes[isbn]) {
            if (copies.at(barcode).is_available()) {
                has_available = true;
                break;
            }
        }

        if (has_available) {
            cout << "Copies are available — no need to reserve. "
                 << "Use issue_book() instead." << endl;
            return false;
        }

        reservations[isbn].push(member_id);
        const Book& book = catalog.at(isbn);
        cout << "📋 Reserved \"" << book.get_title() << "\" for member "
             << mem_it->second.get_name() << ". Position in queue: "
             << reservations[isbn].size() << endl;

        return true;
    }

    // ─── Display ─────────────────────────────────────────

    void display_catalog() const {
        lock_guard<mutex> lock(mtx);
        cout << "\n=== " << name << " Catalog ===" << endl;
        for (const auto& [isbn, book] : catalog) {
            int total = isbn_to_barcodes.at(isbn).size();
            int available = 0;
            for (const auto& barcode : isbn_to_barcodes.at(isbn)) {
                if (copies.at(barcode).is_available()) available++;
            }
            cout << "  ";
            book.print();
            cout << "    Copies: " << available << "/" << total << " available" << endl;
        }
        cout << "==============================\n" << endl;
    }

    void display_member(const string& member_id) const {
        lock_guard<mutex> lock(mtx);
        auto it = members.find(member_id);
        if (it != members.end()) {
            it->second.print();
        }
    }

    // Delete copy/move
    Library(const Library&) = delete;
    Library& operator=(const Library&) = delete;

private:
    void check_reservations(const string& isbn) {
        auto res_it = reservations.find(isbn);
        if (res_it == reservations.end() || res_it->second.empty()) return;

        string next_member = res_it->second.front();
        res_it->second.pop();

        notify_book_available(isbn, next_member);
    }
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    auto& lib = Library::instance();
    lib.set_name("City Central Library");
    lib.add_observer(make_shared<ConsoleNotifier>());

    // Add books
    lib.add_book(Book("978-0-13-468599-1", "The C++ Programming Language",
                      "Bjarne Stroustrup", "Addison-Wesley", 2013), 3);
    lib.add_book(Book("978-0-201-63361-0", "Design Patterns",
                      "Gang of Four", "Addison-Wesley", 1994), 2);
    lib.add_book(Book("978-0-13-235088-4", "Clean Code",
                      "Robert C. Martin", "Prentice Hall", 2008), 1);

    // Add members
    lib.add_member(Member("M001", "Alice", "alice@example.com"));
    lib.add_member(Member("M002", "Bob", "bob@example.com"));
    lib.add_member(Member("M003", "Charlie", "charlie@example.com"));

    lib.display_catalog();

    // Search
    cout << "--- Search by author: 'Stroustrup' ---" << endl;
    auto results = lib.search_by_author("Stroustrup");
    for (auto* book : results) book->print();

    // Issue books
    cout << "\n--- Issue books ---" << endl;
    lib.issue_book("M001", "978-0-13-235088-4");  // Alice gets Clean Code (only 1 copy)
    lib.issue_book("M002", "978-0-13-235088-4");  // Bob tries — no copies available

    // Bob reserves
    cout << "\n--- Reserve ---" << endl;
    lib.reserve_book("M002", "978-0-13-235088-4");

    // Alice returns — Bob gets notified
    cout << "\n--- Return ---" << endl;
    lib.return_book("M001", "978-0-13-235088-4-1");

    lib.display_catalog();

    return 0;
}
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Singleton** | `Library` | One library instance, centralized management |
| **Strategy** | `FineStrategy` | Swap fine calculation algorithms (flat, tiered) without changing library logic |
| **Observer** | `LibraryObserver`, `ConsoleNotifier` | Notify members when reserved books become available |
| **Factory** | Barcode generation in `add_book()` | Automatically creates unique barcodes for each copy |
| **Facade** | `Library` class | Provides a simple interface (`issue_book`, `return_book`) hiding complex internal interactions |

### Key Design Decisions

1. **Book vs BookCopy separation** — a `Book` is a catalog entry (ISBN, title, author); a `BookCopy` is a physical copy (barcode, status). Multiple copies share the same `Book` metadata
2. **Reservation queue** — FIFO queue per ISBN ensures fairness; first to reserve is first to be notified
3. **Fine strategy** — pluggable fine calculation allows the library to change policies without modifying core logic
4. **Thread safety** — mutex protects all shared state for concurrent issue/return operations
5. **Loan history** — completed loans are moved to history for auditing while active loans are tracked separately for quick lookup

---


## 13.2 Elevator System

> Design an elevator system for a building with multiple elevators and multiple floors. This problem tests scheduling algorithms, state management, and concurrent request handling. It's one of the most frequently asked medium-difficulty LLD problems.

---

### Requirements

**Functional Requirements:**
1. The building has N floors and M elevators
2. Each elevator can move up, down, or be idle
3. Users can request an elevator from any floor (external request with direction)
4. Users inside an elevator can select a destination floor (internal request)
5. The system assigns the optimal elevator to each external request
6. Each elevator has a maximum capacity (number of people)
7. Elevators process requests efficiently using a scheduling algorithm
8. The system supports multiple scheduling strategies (SCAN, LOOK, FCFS)
9. Elevator doors open/close at each stop

**Non-Functional Requirements:**
- Thread-safe for concurrent requests from multiple floors
- Extensible for new scheduling algorithms
- Observable — log elevator movements and state changes

---

### Key Entities & Class Design

```
┌──────────────────┐       ┌──────────────────┐
│ ElevatorSystem   │──────▶│    Elevator       │
│ (Controller)     │       │                   │
└──────────────────┘       └──────────────────┘
       │                          │
       │                          │ uses
       ▼                          ▼
┌──────────────────┐       ┌──────────────────┐
│    Request       │       │  ElevatorState    │
│ (Floor + Dir)    │       │  (IDLE/UP/DOWN)   │
└──────────────────┘       └──────────────────┘
       │
       ▼
┌──────────────────┐
│SchedulingStrategy│
│ (SCAN/LOOK/FCFS) │
└──────────────────┘
```

---

### Implementation

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <set>
#include <queue>
#include <memory>
#include <mutex>
#include <algorithm>
#include <functional>
#include <unordered_map>
using namespace std;

// ─── Enums ───────────────────────────────────────────────

enum class Direction { UP, DOWN, IDLE };
enum class DoorState { OPEN, CLOSED };

string to_string(Direction d) {
    switch (d) {
        case Direction::UP:   return "UP";
        case Direction::DOWN: return "DOWN";
        case Direction::IDLE: return "IDLE";
    }
    return "UNKNOWN";
}

// ─── Request ─────────────────────────────────────────────

struct ExternalRequest {
    int floor;
    Direction direction;

    ExternalRequest(int f, Direction d) : floor(f), direction(d) {}
};

struct InternalRequest {
    int destination_floor;
    int elevator_id;

    InternalRequest(int floor, int eid) : destination_floor(floor), elevator_id(eid) {}
};

// ─── Elevator ────────────────────────────────────────────

class Elevator {
    int id;
    int current_floor;
    Direction direction;
    DoorState door;
    int capacity;
    int current_load;

    set<int> up_stops;    // floors to stop at while going up (sorted ascending)
    set<int> down_stops;  // floors to stop at while going down (sorted descending)

    int min_floor;
    int max_floor;

public:
    Elevator(int id, int min_floor, int max_floor, int capacity = 10)
        : id(id), current_floor(min_floor), direction(Direction::IDLE),
          door(DoorState::CLOSED), capacity(capacity), current_load(0),
          min_floor(min_floor), max_floor(max_floor) {}

    int get_id() const { return id; }
    int get_current_floor() const { return current_floor; }
    Direction get_direction() const { return direction; }
    DoorState get_door_state() const { return door; }
    int get_capacity() const { return capacity; }
    int get_current_load() const { return current_load; }
    bool is_idle() const { return direction == Direction::IDLE; }

    bool has_space() const { return current_load < capacity; }

    int total_stops() const { return up_stops.size() + down_stops.size(); }

    // Add a destination floor
    void add_stop(int floor) {
        if (floor > current_floor || (floor == current_floor && direction == Direction::UP)) {
            up_stops.insert(floor);
        } else if (floor < current_floor || (floor == current_floor && direction == Direction::DOWN)) {
            down_stops.insert(floor);
        } else {
            // Same floor and idle — just open doors
            if (direction == Direction::UP || direction == Direction::IDLE) {
                up_stops.insert(floor);
            } else {
                down_stops.insert(floor);
            }
        }

        // Set direction if idle
        if (direction == Direction::IDLE) {
            if (floor > current_floor) direction = Direction::UP;
            else if (floor < current_floor) direction = Direction::DOWN;
        }
    }

    // Calculate distance to a floor (for scheduling)
    int distance_to(int floor, Direction requested_dir) const {
        if (direction == Direction::IDLE) {
            return abs(current_floor - floor);
        }

        // Moving in the same direction as the request
        if (direction == Direction::UP && requested_dir == Direction::UP
            && floor >= current_floor) {
            return floor - current_floor;
        }
        if (direction == Direction::DOWN && requested_dir == Direction::DOWN
            && floor <= current_floor) {
            return current_floor - floor;
        }

        // Moving in opposite direction or past the floor — need to reverse
        // Estimate: go to end, come back
        if (direction == Direction::UP) {
            int top = up_stops.empty() ? current_floor : *up_stops.rbegin();
            return (top - current_floor) + (top - floor);
        } else {
            int bottom = down_stops.empty() ? current_floor : *down_stops.begin();
            return (current_floor - bottom) + (floor - bottom);
        }
    }

    // Move one step (simulate one time unit)
    void step() {
        if (direction == Direction::IDLE) return;

        // Check if current floor is a stop
        if (direction == Direction::UP && up_stops.count(current_floor)) {
            open_doors();
            up_stops.erase(current_floor);
            close_doors();
        } else if (direction == Direction::DOWN && down_stops.count(current_floor)) {
            open_doors();
            down_stops.erase(current_floor);
            close_doors();
        }

        // Move
        if (direction == Direction::UP) {
            if (!up_stops.empty()) {
                current_floor++;
                cout << "  Elevator " << id << " ▲ Floor " << current_floor << endl;
            } else if (!down_stops.empty()) {
                // Reverse direction
                direction = Direction::DOWN;
                current_floor--;
                cout << "  Elevator " << id << " ▼ Floor " << current_floor << endl;
            } else {
                direction = Direction::IDLE;
                cout << "  Elevator " << id << " ■ Idle at Floor " << current_floor << endl;
            }
        } else if (direction == Direction::DOWN) {
            if (!down_stops.empty()) {
                current_floor--;
                cout << "  Elevator " << id << " ▼ Floor " << current_floor << endl;
            } else if (!up_stops.empty()) {
                // Reverse direction
                direction = Direction::UP;
                current_floor++;
                cout << "  Elevator " << id << " ▲ Floor " << current_floor << endl;
            } else {
                direction = Direction::IDLE;
                cout << "  Elevator " << id << " ■ Idle at Floor " << current_floor << endl;
            }
        }

        // Check if arrived at a stop after moving
        if (direction == Direction::UP && up_stops.count(current_floor)) {
            open_doors();
            up_stops.erase(current_floor);
            close_doors();
            if (up_stops.empty() && down_stops.empty()) {
                direction = Direction::IDLE;
            }
        } else if (direction == Direction::DOWN && down_stops.count(current_floor)) {
            open_doors();
            down_stops.erase(current_floor);
            close_doors();
            if (up_stops.empty() && down_stops.empty()) {
                direction = Direction::IDLE;
            }
        }
    }

    void print_status() const {
        cout << "Elevator " << id << ": Floor " << current_floor
             << " | Dir: " << to_string(direction)
             << " | Load: " << current_load << "/" << capacity
             << " | Up stops: " << up_stops.size()
             << " | Down stops: " << down_stops.size() << endl;
    }

private:
    void open_doors() {
        door = DoorState::OPEN;
        cout << "  Elevator " << id << " 🔔 Doors OPEN at Floor "
             << current_floor << endl;
    }

    void close_doors() {
        door = DoorState::CLOSED;
        cout << "  Elevator " << id << " Doors CLOSED at Floor "
             << current_floor << endl;
    }
};

// ─── Scheduling Strategy (Strategy Pattern) ──────────────

class SchedulingStrategy {
public:
    virtual ~SchedulingStrategy() = default;
    virtual Elevator* select_elevator(
        vector<unique_ptr<Elevator>>& elevators,
        const ExternalRequest& request) = 0;
    virtual string name() const = 0;
};

// LOOK algorithm: assign to the nearest elevator moving in the same direction
// or the nearest idle elevator
class LOOKStrategy : public SchedulingStrategy {
public:
    Elevator* select_elevator(
        vector<unique_ptr<Elevator>>& elevators,
        const ExternalRequest& request) override {

        Elevator* best = nullptr;
        int best_distance = INT_MAX;

        for (auto& elev : elevators) {
            if (!elev->has_space()) continue;

            int dist = elev->distance_to(request.floor, request.direction);
            if (dist < best_distance) {
                best_distance = dist;
                best = elev.get();
            }
        }

        return best;
    }

    string name() const override { return "LOOK"; }
};

// FCFS: assign to the elevator with the fewest pending stops
class FCFSStrategy : public SchedulingStrategy {
public:
    Elevator* select_elevator(
        vector<unique_ptr<Elevator>>& elevators,
        const ExternalRequest& request) override {

        Elevator* best = nullptr;
        int min_stops = INT_MAX;

        for (auto& elev : elevators) {
            if (!elev->has_space()) continue;

            // Prefer idle elevators, then least busy
            int stops = elev->total_stops();
            if (elev->is_idle()) stops = -1;  // prioritize idle

            if (stops < min_stops) {
                min_stops = stops;
                best = elev.get();
            }
        }

        return best;
    }

    string name() const override { return "FCFS"; }
};

// ─── Elevator System (Controller) ────────────────────────

class ElevatorSystem {
    int num_floors;
    vector<unique_ptr<Elevator>> elevators;
    unique_ptr<SchedulingStrategy> strategy;
    mutable mutex mtx;

public:
    ElevatorSystem(int floors, int num_elevators, int capacity = 10)
        : num_floors(floors),
          strategy(make_unique<LOOKStrategy>()) {
        for (int i = 0; i < num_elevators; i++) {
            elevators.push_back(
                make_unique<Elevator>(i + 1, 0, floors - 1, capacity));
        }
        cout << "Elevator System initialized: " << floors << " floors, "
             << num_elevators << " elevators (strategy: "
             << strategy->name() << ")" << endl;
    }

    void set_strategy(unique_ptr<SchedulingStrategy> s) {
        lock_guard<mutex> lock(mtx);
        strategy = std::move(s);
        cout << "Scheduling strategy changed to: " << strategy->name() << endl;
    }

    // External request: user on a floor presses UP or DOWN
    void request_elevator(int floor, Direction direction) {
        lock_guard<mutex> lock(mtx);

        if (floor < 0 || floor >= num_floors) {
            cout << "Invalid floor: " << floor << endl;
            return;
        }

        ExternalRequest req(floor, direction);
        Elevator* assigned = strategy->select_elevator(elevators, req);

        if (!assigned) {
            cout << "No elevator available for floor " << floor << endl;
            return;
        }

        assigned->add_stop(floor);
        cout << "📞 External request: Floor " << floor << " "
             << to_string(direction) << " → Assigned to Elevator "
             << assigned->get_id() << endl;
    }

    // Internal request: user inside elevator presses a floor button
    void select_floor(int elevator_id, int floor) {
        lock_guard<mutex> lock(mtx);

        if (elevator_id < 1 || elevator_id > (int)elevators.size()) {
            cout << "Invalid elevator ID: " << elevator_id << endl;
            return;
        }

        if (floor < 0 || floor >= num_floors) {
            cout << "Invalid floor: " << floor << endl;
            return;
        }

        elevators[elevator_id - 1]->add_stop(floor);
        cout << "🔘 Internal request: Elevator " << elevator_id
             << " → Floor " << floor << endl;
    }

    // Simulate one time step — all elevators move
    void step() {
        lock_guard<mutex> lock(mtx);
        cout << "--- Step ---" << endl;
        for (auto& elev : elevators) {
            elev->step();
        }
    }

    // Simulate multiple steps
    void run(int steps) {
        for (int i = 0; i < steps; i++) {
            step();
        }
    }

    void display_status() const {
        lock_guard<mutex> lock(mtx);
        cout << "\n=== Elevator System Status ===" << endl;
        for (const auto& elev : elevators) {
            elev->print_status();
        }
        cout << "==============================\n" << endl;
    }
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    ElevatorSystem system(10, 3);  // 10 floors, 3 elevators

    system.display_status();

    // External requests
    system.request_elevator(5, Direction::UP);    // someone on floor 5 wants to go up
    system.request_elevator(3, Direction::DOWN);  // someone on floor 3 wants to go down
    system.request_elevator(7, Direction::DOWN);  // someone on floor 7 wants to go down

    // Simulate movement
    system.run(3);

    // Internal requests (passengers select destination)
    system.select_floor(1, 8);  // elevator 1 passenger wants floor 8
    system.select_floor(2, 1);  // elevator 2 passenger wants floor 1

    system.run(5);

    system.display_status();

    // Switch strategy
    system.set_strategy(make_unique<FCFSStrategy>());
    system.request_elevator(0, Direction::UP);
    system.run(3);

    system.display_status();

    return 0;
}
```

---

### Scheduling Algorithms Explained

| Algorithm | How It Works | Pros | Cons |
|-----------|-------------|------|------|
| **FCFS** | Assign to least busy elevator | Simple, fair | Not optimal for distance |
| **SCAN** | Elevator goes to one end, then reverses (like disk arm) | Predictable, no starvation | May travel to extremes unnecessarily |
| **LOOK** | Like SCAN but reverses at the last request, not the end | More efficient than SCAN | Slightly more complex |
| **Nearest** | Assign to closest elevator | Minimizes wait time | Can cause starvation for far requests |

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `SchedulingStrategy` | Swap scheduling algorithms (LOOK, FCFS) without changing system logic |
| **State** | `Direction` enum + movement logic | Elevator behavior changes based on direction state |
| **Observer** | Console output on state changes | Log elevator movements (extensible to real notification system) |

### Key Design Decisions

1. **Separate up/down stop sets** — the elevator maintains two sorted sets of stops, one for each direction. This naturally implements the LOOK algorithm
2. **Distance calculation** — accounts for current direction; an elevator moving away from a request has a higher effective distance
3. **Strategy pattern for scheduling** — the system can switch algorithms at runtime without modifying elevator logic
4. **Step-based simulation** — each `step()` call moves all elevators one floor, making the system easy to test and reason about
5. **Capacity tracking** — elevators with no space are skipped during assignment

---


## 13.3 Snake and Ladder Game

> Design a Snake and Ladder board game that supports multiple players, configurable board with snakes and ladders, dice rolling, and win detection. This problem tests your ability to model game mechanics with clean separation of concerns.

---

### Requirements

**Functional Requirements:**
1. The board is a 10×10 grid (positions 1–100)
2. The board has configurable snakes (head → tail, moves player down) and ladders (bottom → top, moves player up)
3. Players take turns rolling a dice (1–6)
4. A player moves forward by the dice value
5. If a player lands on a snake's head, they slide down to the tail
6. If a player lands on a ladder's bottom, they climb to the top
7. The first player to reach exactly position 100 wins
8. If a roll would take a player beyond 100, they stay in place
9. Support 2–4 players
10. Dice can be configurable (single die, double dice, biased dice)

**Non-Functional Requirements:**
- Extensible for different board sizes and dice types
- Observable — notify on game events (snake bite, ladder climb, win)

---

### Key Entities & Class Design

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│    Game      │──────▶│    Board     │──────▶│ Snake/Ladder │
│              │       │              │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
       │                                             
       │                                             
       ▼                                             
┌──────────────┐       ┌──────────────┐
│   Player     │       │    Dice      │
│              │       │ (Strategy)   │
└──────────────┘       └──────────────┘
```

---

### Implementation

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <memory>
#include <random>
#include <algorithm>
#include <functional>
using namespace std;

// ─── Dice (Strategy Pattern) ─────────────────────────────

class Dice {
public:
    virtual ~Dice() = default;
    virtual int roll() = 0;
    virtual string name() const = 0;
};

class SingleDice : public Dice {
    mt19937 rng;
    uniform_int_distribution<int> dist;

public:
    SingleDice() : rng(random_device{}()), dist(1, 6) {}

    int roll() override { return dist(rng); }
    string name() const override { return "Single Die (1-6)"; }
};

class DoubleDice : public Dice {
    mt19937 rng;
    uniform_int_distribution<int> dist;

public:
    DoubleDice() : rng(random_device{}()), dist(1, 6) {}

    int roll() override { return dist(rng) + dist(rng); }
    string name() const override { return "Double Dice (2-12)"; }
};

// Biased dice — for testing (always returns a fixed value)
class FixedDice : public Dice {
    int value;
public:
    FixedDice(int v) : value(v) {}
    int roll() override { return value; }
    string name() const override { return "Fixed (" + to_string(value) + ")"; }
};

// ─── Board Entity (Snake or Ladder) ──────────────────────

enum class EntityType { SNAKE, LADDER };

class BoardEntity {
    int start;
    int end;
    EntityType type;

public:
    BoardEntity(int start, int end, EntityType type)
        : start(start), end(end), type(type) {}

    int get_start() const { return start; }
    int get_end() const { return end; }
    EntityType get_type() const { return type; }

    string to_string() const {
        if (type == EntityType::SNAKE) {
            return "🐍 Snake: " + std::to_string(start) + " → " + std::to_string(end);
        }
        return "🪜 Ladder: " + std::to_string(start) + " → " + std::to_string(end);
    }
};

// ─── Board ───────────────────────────────────────────────

class Board {
    int size;  // total cells (default 100)
    unordered_map<int, BoardEntity> entities;  // position → snake or ladder

public:
    Board(int size = 100) : size(size) {}

    int get_size() const { return size; }

    void add_snake(int head, int tail) {
        if (head <= tail) {
            cout << "Invalid snake: head must be above tail." << endl;
            return;
        }
        if (head > size || tail < 1) {
            cout << "Invalid snake: positions out of bounds." << endl;
            return;
        }
        entities.emplace(head, BoardEntity(head, tail, EntityType::SNAKE));
    }

    void add_ladder(int bottom, int top) {
        if (bottom >= top) {
            cout << "Invalid ladder: bottom must be below top." << endl;
            return;
        }
        if (top > size || bottom < 1) {
            cout << "Invalid ladder: positions out of bounds." << endl;
            return;
        }
        entities.emplace(bottom, BoardEntity(bottom, top, EntityType::LADDER));
    }

    // Check if a position has a snake or ladder, return final position
    int get_final_position(int position) const {
        auto it = entities.find(position);
        if (it != entities.end()) {
            const BoardEntity& entity = it->second;
            cout << "    " << entity.to_string() << endl;
            return entity.get_end();
        }
        return position;
    }

    void print() const {
        cout << "\n=== Board (" << size << " cells) ===" << endl;
        cout << "Snakes:" << endl;
        for (const auto& [pos, entity] : entities) {
            if (entity.get_type() == EntityType::SNAKE) {
                cout << "  " << entity.to_string() << endl;
            }
        }
        cout << "Ladders:" << endl;
        for (const auto& [pos, entity] : entities) {
            if (entity.get_type() == EntityType::LADDER) {
                cout << "  " << entity.to_string() << endl;
            }
        }
        cout << "============================\n" << endl;
    }
};

// ─── Player ──────────────────────────────────────────────

class Player {
    string name;
    int position;

public:
    Player(const string& name) : name(name), position(0) {}

    const string& get_name() const { return name; }
    int get_position() const { return position; }

    void set_position(int pos) { position = pos; }

    void print() const {
        cout << name << " at position " << position;
    }
};

// ─── Game Observer ───────────────────────────────────────

class GameObserver {
public:
    virtual ~GameObserver() = default;
    virtual void on_turn(const Player& player, int dice_value, int old_pos, int new_pos) = 0;
    virtual void on_snake(const Player& player, int from, int to) = 0;
    virtual void on_ladder(const Player& player, int from, int to) = 0;
    virtual void on_win(const Player& player, int turns) = 0;
    virtual void on_bounce(const Player& player, int dice_value) = 0;
};

class ConsoleGameObserver : public GameObserver {
public:
    void on_turn(const Player& player, int dice_value, int old_pos, int new_pos) override {
        cout << "  " << player.get_name() << " rolled " << dice_value
             << ": " << old_pos << " → " << new_pos << endl;
    }

    void on_snake(const Player& player, int from, int to) override {
        cout << "  😱 " << player.get_name() << " bitten by snake! "
             << from << " → " << to << endl;
    }

    void on_ladder(const Player& player, int from, int to) override {
        cout << "  🎉 " << player.get_name() << " climbed a ladder! "
             << from << " → " << to << endl;
    }

    void on_win(const Player& player, int turns) override {
        cout << "\n🏆 " << player.get_name() << " WINS in " << turns
             << " turns!" << endl;
    }

    void on_bounce(const Player& player, int dice_value) override {
        cout << "  " << player.get_name() << " rolled " << dice_value
             << " but can't move (would exceed 100). Stays at "
             << player.get_position() << endl;
    }
};

// ─── Game ────────────────────────────────────────────────

class SnakeAndLadderGame {
    Board board;
    vector<Player> players;
    unique_ptr<Dice> dice;
    vector<shared_ptr<GameObserver>> observers;
    int current_player_index = 0;
    int total_turns = 0;
    bool game_over = false;

    void notify_turn(const Player& p, int dv, int op, int np) {
        for (auto& obs : observers) obs->on_turn(p, dv, op, np);
    }
    void notify_snake(const Player& p, int f, int t) {
        for (auto& obs : observers) obs->on_snake(p, f, t);
    }
    void notify_ladder(const Player& p, int f, int t) {
        for (auto& obs : observers) obs->on_ladder(p, f, t);
    }
    void notify_win(const Player& p, int turns) {
        for (auto& obs : observers) obs->on_win(p, turns);
    }
    void notify_bounce(const Player& p, int dv) {
        for (auto& obs : observers) obs->on_bounce(p, dv);
    }

public:
    SnakeAndLadderGame(unique_ptr<Dice> dice = make_unique<SingleDice>())
        : dice(std::move(dice)) {}

    void set_board(const Board& b) { board = b; }

    void add_player(const string& name) {
        if (players.size() >= 4) {
            cout << "Maximum 4 players allowed." << endl;
            return;
        }
        players.emplace_back(name);
        cout << "Added player: " << name << endl;
    }

    void add_observer(shared_ptr<GameObserver> obs) {
        observers.push_back(obs);
    }

    // Play one turn
    bool play_turn() {
        if (game_over || players.empty()) return false;

        Player& current = players[current_player_index];
        int dice_value = dice->roll();
        int old_pos = current.get_position();
        int new_pos = old_pos + dice_value;

        total_turns++;

        // Check if move is valid (can't exceed board size)
        if (new_pos > board.get_size()) {
            notify_bounce(current, dice_value);
            current_player_index = (current_player_index + 1) % players.size();
            return true;
        }

        // Move to new position
        current.set_position(new_pos);
        notify_turn(current, dice_value, old_pos, new_pos);

        // Check for snake or ladder
        int final_pos = board.get_final_position(new_pos);
        if (final_pos != new_pos) {
            if (final_pos < new_pos) {
                notify_snake(current, new_pos, final_pos);
            } else {
                notify_ladder(current, new_pos, final_pos);
            }
            current.set_position(final_pos);
        }

        // Check for win
        if (current.get_position() == board.get_size()) {
            game_over = true;
            notify_win(current, total_turns);
            return false;
        }

        // Next player
        current_player_index = (current_player_index + 1) % players.size();
        return true;
    }

    // Play the entire game
    void play() {
        if (players.size() < 2) {
            cout << "Need at least 2 players to start." << endl;
            return;
        }

        cout << "\n=== Snake and Ladder Game ===" << endl;
        cout << "Players: ";
        for (const auto& p : players) cout << p.get_name() << " ";
        cout << "\nDice: " << dice->name() << endl;
        board.print();

        while (play_turn()) {
            // Game continues
        }

        // Final positions
        cout << "\n--- Final Positions ---" << endl;
        for (const auto& p : players) {
            cout << "  ";
            p.print();
            cout << endl;
        }
    }
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    // Setup board
    Board board(100);

    // Add snakes
    board.add_snake(99, 54);
    board.add_snake(70, 55);
    board.add_snake(52, 42);
    board.add_snake(25, 2);
    board.add_snake(95, 72);

    // Add ladders
    board.add_ladder(6, 25);
    board.add_ladder(11, 40);
    board.add_ladder(60, 85);
    board.add_ladder(46, 90);
    board.add_ladder(17, 69);

    // Setup game
    SnakeAndLadderGame game;
    game.set_board(board);
    game.add_player("Alice");
    game.add_player("Bob");
    game.add_player("Charlie");
    game.add_observer(make_shared<ConsoleGameObserver>());

    // Play
    game.play();

    return 0;
}
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `Dice` (SingleDice, DoubleDice, FixedDice) | Swap dice behavior without changing game logic; FixedDice enables deterministic testing |
| **Observer** | `GameObserver`, `ConsoleGameObserver` | Decouple game events from presentation; extensible for GUI, network, or logging observers |
| **Composite** | `BoardEntity` on `Board` | Board composes snakes and ladders uniformly |

### Key Design Decisions

1. **Board entity abstraction** — snakes and ladders are both `BoardEntity` with a start and end position, differing only in direction. The board maps positions to entities for O(1) lookup
2. **Dice as strategy** — pluggable dice allows single die, double dice, or fixed dice for testing. The game doesn't know or care how the dice value is generated
3. **Bounce rule** — if a roll would take a player past 100, they stay in place. This is the standard rule and prevents overshooting
4. **Observer for events** — game events (turns, snake bites, ladder climbs, wins) are broadcast to observers, keeping the game logic clean and the presentation separate
5. **Turn-based architecture** — `play_turn()` processes one turn at a time, making the game easy to integrate with a UI or network layer

---


## 13.4 Splitwise / Expense Sharing

> Design an expense sharing system like Splitwise that supports users, groups, multiple split types (equal, exact, percentage), balance tracking, and debt simplification. This problem tests your ability to handle complex financial calculations and multiple splitting strategies.

---

### Requirements

**Functional Requirements:**
1. Users can create accounts with name and email
2. Users can create groups and add members
3. Any user can add an expense to a group
4. Expenses can be split in three ways:
   - **Equal**: split equally among all participants
   - **Exact**: each participant owes a specific amount
   - **Percentage**: each participant owes a percentage of the total
5. The system tracks balances between every pair of users
6. Users can view how much they owe or are owed
7. The system can simplify debts (minimize the number of transactions)
8. Users can settle up (record a payment)

**Non-Functional Requirements:**
- Accurate floating-point handling for currency
- Extensible for new split types
- Thread-safe for concurrent expense additions

---

### Key Entities & Class Design

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│    User      │◀─────▶│    Group     │──────▶│   Expense    │
│              │       │              │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
                                                     │
                                                     │ uses
                                                     ▼
                                              ┌──────────────┐
                                              │SplitStrategy │
                                              │(Equal/Exact/ │
                                              │ Percentage)  │
                                              └──────────────┘
       ┌──────────────┐
       │BalanceSheet  │  (tracks who owes whom)
       └──────────────┘
```

---

### Implementation

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <unordered_set>
#include <memory>
#include <cmath>
#include <algorithm>
#include <mutex>
#include <iomanip>
#include <numeric>
using namespace std;

// ─── Split Type ──────────────────────────────────────────

enum class SplitType { EQUAL, EXACT, PERCENTAGE };

string to_string(SplitType t) {
    switch (t) {
        case SplitType::EQUAL:      return "Equal";
        case SplitType::EXACT:      return "Exact";
        case SplitType::PERCENTAGE: return "Percentage";
    }
    return "Unknown";
}

// ─── User ────────────────────────────────────────────────

class User {
    string user_id;
    string name;
    string email;

public:
    User(const string& id, const string& name, const string& email)
        : user_id(id), name(name), email(email) {}

    const string& get_id() const { return user_id; }
    const string& get_name() const { return name; }
    const string& get_email() const { return email; }
};

// ─── Split (how much each person owes) ───────────────────

struct Split {
    string user_id;
    double amount;  // how much this user owes

    Split(const string& uid, double amt) : user_id(uid), amount(amt) {}
};

// ─── Split Strategy (Strategy Pattern) ───────────────────

class SplitStrategy {
public:
    virtual ~SplitStrategy() = default;
    virtual vector<Split> calculate(
        double total_amount,
        const string& paid_by,
        const vector<string>& participants,
        const unordered_map<string, double>& split_details) const = 0;
    virtual bool validate(
        double total_amount,
        const vector<string>& participants,
        const unordered_map<string, double>& split_details) const = 0;
    virtual SplitType type() const = 0;
};

class EqualSplitStrategy : public SplitStrategy {
public:
    vector<Split> calculate(
        double total_amount,
        const string& paid_by,
        const vector<string>& participants,
        const unordered_map<string, double>& /*split_details*/) const override {

        vector<Split> splits;
        double per_person = total_amount / participants.size();

        // Round to 2 decimal places
        per_person = round(per_person * 100.0) / 100.0;

        for (const auto& uid : participants) {
            if (uid != paid_by) {
                splits.emplace_back(uid, per_person);
            }
        }
        return splits;
    }

    bool validate(double /*total*/, const vector<string>& participants,
                  const unordered_map<string, double>& /*details*/) const override {
        return participants.size() >= 2;
    }

    SplitType type() const override { return SplitType::EQUAL; }
};

class ExactSplitStrategy : public SplitStrategy {
public:
    vector<Split> calculate(
        double total_amount,
        const string& paid_by,
        const vector<string>& participants,
        const unordered_map<string, double>& split_details) const override {

        vector<Split> splits;
        for (const auto& uid : participants) {
            if (uid != paid_by) {
                auto it = split_details.find(uid);
                if (it != split_details.end() && it->second > 0) {
                    splits.emplace_back(uid, it->second);
                }
            }
        }
        return splits;
    }

    bool validate(double total_amount, const vector<string>& participants,
                  const unordered_map<string, double>& split_details) const override {
        double sum = 0;
        for (const auto& [uid, amount] : split_details) {
            if (amount < 0) return false;
            sum += amount;
        }
        // Sum of exact amounts should equal total
        return abs(sum - total_amount) < 0.01;
    }

    SplitType type() const override { return SplitType::EXACT; }
};

class PercentageSplitStrategy : public SplitStrategy {
public:
    vector<Split> calculate(
        double total_amount,
        const string& paid_by,
        const vector<string>& participants,
        const unordered_map<string, double>& split_details) const override {

        vector<Split> splits;
        for (const auto& uid : participants) {
            if (uid != paid_by) {
                auto it = split_details.find(uid);
                if (it != split_details.end() && it->second > 0) {
                    double amount = round(total_amount * it->second / 100.0 * 100.0) / 100.0;
                    splits.emplace_back(uid, amount);
                }
            }
        }
        return splits;
    }

    bool validate(double /*total*/, const vector<string>& participants,
                  const unordered_map<string, double>& split_details) const override {
        double sum = 0;
        for (const auto& [uid, pct] : split_details) {
            if (pct < 0 || pct > 100) return false;
            sum += pct;
        }
        // Percentages should sum to 100
        return abs(sum - 100.0) < 0.01;
    }

    SplitType type() const override { return SplitType::PERCENTAGE; }
};

// ─── Expense ─────────────────────────────────────────────

class Expense {
    static int next_id;

    int expense_id;
    string description;
    double amount;
    string paid_by;
    SplitType split_type;
    vector<Split> splits;

public:
    Expense(const string& desc, double amount, const string& paid_by,
            SplitType type, const vector<Split>& splits)
        : expense_id(next_id++), description(desc), amount(amount),
          paid_by(paid_by), split_type(type), splits(splits) {}

    int get_id() const { return expense_id; }
    const string& get_description() const { return description; }
    double get_amount() const { return amount; }
    const string& get_paid_by() const { return paid_by; }
    SplitType get_split_type() const { return split_type; }
    const vector<Split>& get_splits() const { return splits; }

    void print() const {
        cout << "Expense #" << expense_id << ": " << description
             << " ($" << fixed << setprecision(2) << amount << ")"
             << " paid by " << paid_by
             << " [" << to_string(split_type) << "]" << endl;
        for (const auto& s : splits) {
            cout << "  " << s.user_id << " owes $"
                 << fixed << setprecision(2) << s.amount << endl;
        }
    }
};

int Expense::next_id = 1;

// ─── Balance Sheet ───────────────────────────────────────

class BalanceSheet {
    // balances[A][B] > 0 means A owes B that amount
    // balances[A][B] < 0 means B owes A that amount
    unordered_map<string, unordered_map<string, double>> balances;

public:
    void add_debt(const string& debtor, const string& creditor, double amount) {
        balances[debtor][creditor] += amount;
        balances[creditor][debtor] -= amount;
    }

    void settle(const string& from, const string& to, double amount) {
        double owed = balances[from][to];
        if (owed <= 0) {
            cout << from << " does not owe " << to << " anything." << endl;
            return;
        }
        double settle_amount = min(amount, owed);
        balances[from][to] -= settle_amount;
        balances[to][from] += settle_amount;
        cout << "💰 " << from << " paid $" << fixed << setprecision(2)
             << settle_amount << " to " << to << endl;
    }

    double get_balance(const string& user1, const string& user2) const {
        auto it1 = balances.find(user1);
        if (it1 == balances.end()) return 0;
        auto it2 = it1->second.find(user2);
        if (it2 == it1->second.end()) return 0;
        return it2->second;
    }

    // Get net balance for a user (positive = owed money, negative = owes money)
    double get_net_balance(const string& user_id) const {
        double net = 0;
        auto it = balances.find(user_id);
        if (it == balances.end()) return 0;
        for (const auto& [other, amount] : it->second) {
            // Negative balance means others owe us
            net += amount;
        }
        return net;  // positive = we owe others, negative = others owe us
    }

    void print_balances(const string& user_id,
                        const unordered_map<string, User>& users) const {
        auto it = balances.find(user_id);
        if (it == balances.end()) {
            cout << "No balances for " << user_id << endl;
            return;
        }

        cout << "Balances for " << users.at(user_id).get_name() << ":" << endl;
        for (const auto& [other_id, amount] : it->second) {
            if (abs(amount) < 0.01) continue;
            if (amount > 0) {
                cout << "  Owes " << users.at(other_id).get_name()
                     << ": $" << fixed << setprecision(2) << amount << endl;
            } else {
                cout << "  Owed by " << users.at(other_id).get_name()
                     << ": $" << fixed << setprecision(2) << -amount << endl;
            }
        }
    }

    // Simplify debts — minimize number of transactions
    vector<tuple<string, string, double>> simplify() const {
        // Calculate net balance for each user
        unordered_map<string, double> net;
        for (const auto& [user, others] : balances) {
            for (const auto& [other, amount] : others) {
                net[user] += amount;
            }
        }

        // Separate into debtors (positive net = owes money) and creditors (negative net)
        vector<pair<string, double>> debtors;   // owe money
        vector<pair<string, double>> creditors;  // are owed money

        for (const auto& [user, balance] : net) {
            if (balance > 0.01) {
                debtors.push_back({user, balance});
            } else if (balance < -0.01) {
                creditors.push_back({user, -balance});
            }
        }

        // Greedy matching: match largest debtor with largest creditor
        vector<tuple<string, string, double>> transactions;
        int i = 0, j = 0;

        while (i < (int)debtors.size() && j < (int)creditors.size()) {
            double amount = min(debtors[i].second, creditors[j].second);
            transactions.emplace_back(debtors[i].first, creditors[j].first, amount);

            debtors[i].second -= amount;
            creditors[j].second -= amount;

            if (debtors[i].second < 0.01) i++;
            if (creditors[j].second < 0.01) j++;
        }

        return transactions;
    }
};

// ─── Group ───────────────────────────────────────────────

class Group {
    string group_id;
    string name;
    unordered_set<string> member_ids;
    vector<Expense> expenses;

public:
    Group(const string& id, const string& name) : group_id(id), name(name) {}

    const string& get_id() const { return group_id; }
    const string& get_name() const { return name; }
    const unordered_set<string>& get_members() const { return member_ids; }

    void add_member(const string& user_id) {
        member_ids.insert(user_id);
    }

    void add_expense(const Expense& expense) {
        expenses.push_back(expense);
    }

    const vector<Expense>& get_expenses() const { return expenses; }
};

// ─── Expense Manager (Facade) ────────────────────────────

class ExpenseManager {
    unordered_map<string, User> users;
    unordered_map<string, Group> groups;
    BalanceSheet balance_sheet;

    // Strategy registry
    unordered_map<SplitType, unique_ptr<SplitStrategy>> strategies;

    mutable mutex mtx;

public:
    ExpenseManager() {
        strategies[SplitType::EQUAL] = make_unique<EqualSplitStrategy>();
        strategies[SplitType::EXACT] = make_unique<ExactSplitStrategy>();
        strategies[SplitType::PERCENTAGE] = make_unique<PercentageSplitStrategy>();
    }

    // ─── User Management ─────────────────────────────────

    void add_user(const string& id, const string& name, const string& email) {
        lock_guard<mutex> lock(mtx);
        users.emplace(id, User(id, name, email));
        cout << "User registered: " << name << " (" << id << ")" << endl;
    }

    // ─── Group Management ────────────────────────────────

    void create_group(const string& group_id, const string& name,
                      const vector<string>& member_ids) {
        lock_guard<mutex> lock(mtx);
        Group group(group_id, name);
        for (const auto& mid : member_ids) {
            if (users.count(mid)) {
                group.add_member(mid);
            }
        }
        groups.emplace(group_id, group);
        cout << "Group created: " << name << " with " << member_ids.size()
             << " members" << endl;
    }

    // ─── Add Expense ─────────────────────────────────────

    bool add_expense(const string& group_id, const string& description,
                     double amount, const string& paid_by,
                     SplitType split_type,
                     const unordered_map<string, double>& split_details = {}) {
        lock_guard<mutex> lock(mtx);

        auto grp_it = groups.find(group_id);
        if (grp_it == groups.end()) {
            cout << "Error: Group " << group_id << " not found." << endl;
            return false;
        }
        Group& group = grp_it->second;

        if (!group.get_members().count(paid_by)) {
            cout << "Error: Payer " << paid_by << " is not in the group." << endl;
            return false;
        }

        // Get participants
        vector<string> participants(group.get_members().begin(),
                                    group.get_members().end());

        // Validate split
        auto& strategy = strategies.at(split_type);
        if (!strategy->validate(amount, participants, split_details)) {
            cout << "Error: Invalid split details for "
                 << to_string(split_type) << " split." << endl;
            return false;
        }

        // Calculate splits
        vector<Split> splits = strategy->calculate(
            amount, paid_by, participants, split_details);

        // Create expense
        Expense expense(description, amount, paid_by, split_type, splits);
        group.add_expense(expense);

        // Update balances
        for (const auto& split : splits) {
            balance_sheet.add_debt(split.user_id, paid_by, split.amount);
        }

        cout << "✅ ";
        expense.print();

        return true;
    }

    // ─── Settle Up ───────────────────────────────────────

    void settle_up(const string& from, const string& to, double amount) {
        lock_guard<mutex> lock(mtx);
        balance_sheet.settle(from, to, amount);
    }

    // ─── View Balances ───────────────────────────────────

    void show_balances(const string& user_id) const {
        lock_guard<mutex> lock(mtx);
        balance_sheet.print_balances(user_id, users);
    }

    void show_all_balances() const {
        lock_guard<mutex> lock(mtx);
        cout << "\n=== All Balances ===" << endl;
        for (const auto& [uid, user] : users) {
            balance_sheet.print_balances(uid, users);
        }
        cout << "====================\n" << endl;
    }

    // ─── Simplify Debts ──────────────────────────────────

    void simplify_debts() const {
        lock_guard<mutex> lock(mtx);
        auto transactions = balance_sheet.simplify();

        cout << "\n=== Simplified Debts ===" << endl;
        if (transactions.empty()) {
            cout << "All settled up!" << endl;
        }
        for (const auto& [from, to, amount] : transactions) {
            cout << "  " << users.at(from).get_name() << " pays "
                 << users.at(to).get_name() << ": $"
                 << fixed << setprecision(2) << amount << endl;
        }
        cout << "========================\n" << endl;
    }
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    ExpenseManager manager;

    // Register users
    manager.add_user("U1", "Alice", "alice@example.com");
    manager.add_user("U2", "Bob", "bob@example.com");
    manager.add_user("U3", "Charlie", "charlie@example.com");
    manager.add_user("U4", "Diana", "diana@example.com");

    // Create a group
    manager.create_group("G1", "Trip to Paris", {"U1", "U2", "U3", "U4"});

    // Add expenses
    cout << "\n--- Adding Expenses ---" << endl;

    // Equal split: Alice pays $100 for dinner, split equally among 4
    manager.add_expense("G1", "Dinner", 100.0, "U1", SplitType::EQUAL);

    // Exact split: Bob pays $60 for taxi
    manager.add_expense("G1", "Taxi", 60.0, "U2", SplitType::EXACT,
                        {{"U1", 10.0}, {"U2", 20.0}, {"U3", 15.0}, {"U4", 15.0}});

    // Percentage split: Charlie pays $200 for hotel
    manager.add_expense("G1", "Hotel", 200.0, "U3", SplitType::PERCENTAGE,
                        {{"U1", 30.0}, {"U2", 30.0}, {"U3", 20.0}, {"U4", 20.0}});

    // View balances
    cout << "\n--- Individual Balances ---" << endl;
    manager.show_balances("U1");
    manager.show_balances("U2");

    // Simplify debts
    manager.simplify_debts();

    // Settle up
    cout << "--- Settling Up ---" << endl;
    manager.settle_up("U1", "U3", 50.0);

    manager.show_balances("U1");

    return 0;
}
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `SplitStrategy` (Equal, Exact, Percentage) | Different splitting algorithms without changing expense logic |
| **Facade** | `ExpenseManager` | Simple interface hiding complex balance tracking and split calculations |
| **Observer** | Console output on expense/settlement events | Extensible for push notifications, email alerts |

### Key Design Decisions

1. **Bidirectional balance tracking** — `balances[A][B]` and `balances[B][A]` are always negatives of each other, ensuring consistency. If A owes B $50, then `balances[A][B] = 50` and `balances[B][A] = -50`
2. **Strategy registry** — split strategies are registered by type in a map, making it easy to add new split types (e.g., share-based, weight-based) without modifying the manager
3. **Debt simplification** — the greedy algorithm calculates net balances and matches debtors with creditors, minimizing the number of transactions needed to settle all debts
4. **Validation before calculation** — each strategy validates its inputs (exact amounts sum to total, percentages sum to 100) before computing splits
5. **Rounding** — amounts are rounded to 2 decimal places to avoid floating-point accumulation errors in currency calculations

---


## 13.5 Movie Ticket Booking (BookMyShow)

> Design a movie ticket booking system like BookMyShow that supports theaters, screens, shows, seat selection, concurrent booking handling, and payment. This problem tests your ability to handle concurrency (seat locking), complex entity relationships, and real-world booking workflows.

---

### Requirements

**Functional Requirements:**
1. The system manages multiple theaters, each with multiple screens
2. Each screen has a seating arrangement (rows and columns) with seat categories (Silver, Gold, Platinum)
3. Movies are scheduled as shows on specific screens at specific times
4. Users can search for movies by name, city, or theater
5. Users can view available seats for a show
6. Users can select seats and book tickets
7. Selected seats are temporarily locked (5 minutes) to prevent double booking
8. Payment is processed after seat selection
9. Booking is confirmed after successful payment; seats are released if payment fails or times out
10. Users can cancel bookings (with cancellation policy)

**Non-Functional Requirements:**
- Thread-safe for concurrent seat selection and booking
- No double booking — a seat can only be booked by one user per show
- Extensible for different pricing strategies and payment methods

---

### Key Entities & Class Design

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   Theater    │──────▶│   Screen     │──────▶│    Seat      │
│              │       │              │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
                              │                      │
                              ▼                      │
                       ┌──────────────┐              │
                       │    Show      │              │
                       │(Movie+Time) │              │
                       └──────────────┘              │
                              │                      │
                              ▼                      ▼
                       ┌──────────────┐       ┌──────────────┐
                       │  ShowSeat    │──────▶│   Booking    │
                       │(seat status) │       │              │
                       └──────────────┘       └──────────────┘
                                                     │
                                                     ▼
                                              ┌──────────────┐
                                              │   Payment    │
                                              └──────────────┘
```

---

### Implementation

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <memory>
#include <mutex>
#include <optional>
#include <chrono>
#include <algorithm>
#include <iomanip>
using namespace std;

// ─── Enums ───────────────────────────────────────────────

enum class SeatCategory { SILVER, GOLD, PLATINUM };
enum class SeatStatus { AVAILABLE, LOCKED, BOOKED };
enum class BookingStatus { PENDING, CONFIRMED, CANCELLED };
enum class PaymentStatus { PENDING, SUCCESS, FAILED };

string to_string(SeatCategory c) {
    switch (c) {
        case SeatCategory::SILVER:   return "Silver";
        case SeatCategory::GOLD:     return "Gold";
        case SeatCategory::PLATINUM: return "Platinum";
    }
    return "Unknown";
}

string to_string(SeatStatus s) {
    switch (s) {
        case SeatStatus::AVAILABLE: return "Available";
        case SeatStatus::LOCKED:    return "Locked";
        case SeatStatus::BOOKED:    return "Booked";
    }
    return "Unknown";
}

string to_string(BookingStatus s) {
    switch (s) {
        case BookingStatus::PENDING:   return "Pending";
        case BookingStatus::CONFIRMED: return "Confirmed";
        case BookingStatus::CANCELLED: return "Cancelled";
    }
    return "Unknown";
}

// ─── Movie ───────────────────────────────────────────────

class Movie {
    string movie_id;
    string title;
    string genre;
    int duration_minutes;
    string language;

public:
    Movie(const string& id, const string& title, const string& genre,
          int duration, const string& lang)
        : movie_id(id), title(title), genre(genre),
          duration_minutes(duration), language(lang) {}

    const string& get_id() const { return movie_id; }
    const string& get_title() const { return title; }
    const string& get_genre() const { return genre; }
    int get_duration() const { return duration_minutes; }
    const string& get_language() const { return language; }
};

// ─── Seat ────────────────────────────────────────────────

class Seat {
    string seat_id;  // e.g., "A1", "B5"
    int row;
    int col;
    SeatCategory category;

public:
    Seat(const string& id, int row, int col, SeatCategory cat)
        : seat_id(id), row(row), col(col), category(cat) {}

    const string& get_id() const { return seat_id; }
    int get_row() const { return row; }
    int get_col() const { return col; }
    SeatCategory get_category() const { return category; }
};

// ─── ShowSeat (seat status for a specific show) ──────────

class ShowSeat {
    string seat_id;
    SeatCategory category;
    SeatStatus status;
    double price;
    string locked_by;  // user who locked this seat
    chrono::steady_clock::time_point lock_expiry;

public:
    ShowSeat(const string& seat_id, SeatCategory cat, double price)
        : seat_id(seat_id), category(cat), status(SeatStatus::AVAILABLE),
          price(price) {}

    const string& get_seat_id() const { return seat_id; }
    SeatCategory get_category() const { return category; }
    SeatStatus get_status() const { return status; }
    double get_price() const { return price; }
    const string& get_locked_by() const { return locked_by; }

    bool is_available() const {
        if (status == SeatStatus::AVAILABLE) return true;
        // Check if lock has expired
        if (status == SeatStatus::LOCKED) {
            return chrono::steady_clock::now() > lock_expiry;
        }
        return false;
    }

    bool lock(const string& user_id, int lock_seconds = 300) {
        if (!is_available()) return false;
        status = SeatStatus::LOCKED;
        locked_by = user_id;
        lock_expiry = chrono::steady_clock::now() + chrono::seconds(lock_seconds);
        return true;
    }

    bool book(const string& user_id) {
        if (status != SeatStatus::LOCKED || locked_by != user_id) return false;
        status = SeatStatus::BOOKED;
        return true;
    }

    void release() {
        status = SeatStatus::AVAILABLE;
        locked_by.clear();
    }
};

// ─── Screen ──────────────────────────────────────────────

class Screen {
    string screen_id;
    string name;
    vector<Seat> seats;
    int total_rows;
    int total_cols;

public:
    Screen(const string& id, const string& name, int rows, int cols,
           const vector<pair<int, SeatCategory>>& row_categories)
        : screen_id(id), name(name), total_rows(rows), total_cols(cols) {

        // row_categories: list of (num_rows, category)
        int current_row = 0;
        for (const auto& [num_rows, category] : row_categories) {
            for (int r = 0; r < num_rows; r++) {
                char row_letter = 'A' + current_row;
                for (int c = 1; c <= cols; c++) {
                    string seat_id = string(1, row_letter) + to_string(c);
                    seats.emplace_back(seat_id, current_row, c, category);
                }
                current_row++;
            }
        }
    }

    const string& get_id() const { return screen_id; }
    const string& get_name() const { return name; }
    const vector<Seat>& get_seats() const { return seats; }
    int get_total_rows() const { return total_rows; }
    int get_total_cols() const { return total_cols; }
};

// ─── Pricing Strategy (Strategy Pattern) ─────────────────

class PricingStrategy {
public:
    virtual ~PricingStrategy() = default;
    virtual double get_price(SeatCategory category) const = 0;
    virtual string name() const = 0;
};

class StandardPricing : public PricingStrategy {
public:
    double get_price(SeatCategory category) const override {
        switch (category) {
            case SeatCategory::SILVER:   return 150.0;
            case SeatCategory::GOLD:     return 250.0;
            case SeatCategory::PLATINUM: return 400.0;
        }
        return 0;
    }
    string name() const override { return "Standard"; }
};

class WeekendPricing : public PricingStrategy {
public:
    double get_price(SeatCategory category) const override {
        switch (category) {
            case SeatCategory::SILVER:   return 200.0;
            case SeatCategory::GOLD:     return 350.0;
            case SeatCategory::PLATINUM: return 500.0;
        }
        return 0;
    }
    string name() const override { return "Weekend"; }
};

// ─── Show ────────────────────────────────────────────────

class Show {
    string show_id;
    string movie_id;
    string screen_id;
    string start_time;  // simplified as string for demo
    unordered_map<string, ShowSeat> show_seats;  // seat_id → ShowSeat
    mutable mutex mtx;

public:
    Show(const string& id, const string& movie_id, const string& screen_id,
         const string& start_time, const Screen& screen,
         const PricingStrategy& pricing)
        : show_id(id), movie_id(movie_id), screen_id(screen_id),
          start_time(start_time) {

        for (const auto& seat : screen.get_seats()) {
            double price = pricing.get_price(seat.get_category());
            show_seats.emplace(seat.get_id(),
                ShowSeat(seat.get_id(), seat.get_category(), price));
        }
    }

    const string& get_id() const { return show_id; }
    const string& get_movie_id() const { return movie_id; }
    const string& get_screen_id() const { return screen_id; }
    const string& get_start_time() const { return start_time; }

    // Get available seats
    vector<const ShowSeat*> get_available_seats() const {
        lock_guard<mutex> lock(mtx);
        vector<const ShowSeat*> available;
        for (const auto& [id, seat] : show_seats) {
            if (seat.is_available()) {
                available.push_back(&seat);
            }
        }
        return available;
    }

    // Lock seats for a user (returns true if all seats locked successfully)
    bool lock_seats(const vector<string>& seat_ids, const string& user_id) {
        lock_guard<mutex> lock(mtx);

        // First check all seats are available
        for (const auto& sid : seat_ids) {
            auto it = show_seats.find(sid);
            if (it == show_seats.end() || !it->second.is_available()) {
                return false;
            }
        }

        // Lock all seats atomically
        for (const auto& sid : seat_ids) {
            show_seats.at(sid).lock(user_id);
        }
        return true;
    }

    // Book locked seats (after payment)
    bool book_seats(const vector<string>& seat_ids, const string& user_id) {
        lock_guard<mutex> lock(mtx);
        for (const auto& sid : seat_ids) {
            auto it = show_seats.find(sid);
            if (it == show_seats.end() || !it->second.book(user_id)) {
                // Rollback: release all seats locked by this user
                for (const auto& s : seat_ids) {
                    auto sit = show_seats.find(s);
                    if (sit != show_seats.end() &&
                        sit->second.get_locked_by() == user_id) {
                        sit->second.release();
                    }
                }
                return false;
            }
        }
        return true;
    }

    // Release locked seats (on payment failure or timeout)
    void release_seats(const vector<string>& seat_ids, const string& user_id) {
        lock_guard<mutex> lock(mtx);
        for (const auto& sid : seat_ids) {
            auto it = show_seats.find(sid);
            if (it != show_seats.end() && it->second.get_locked_by() == user_id) {
                it->second.release();
            }
        }
    }

    double calculate_total(const vector<string>& seat_ids) const {
        lock_guard<mutex> lock(mtx);
        double total = 0;
        for (const auto& sid : seat_ids) {
            auto it = show_seats.find(sid);
            if (it != show_seats.end()) {
                total += it->second.get_price();
            }
        }
        return total;
    }

    void display_seats() const {
        lock_guard<mutex> lock(mtx);
        cout << "Show " << show_id << " at " << start_time << ":" << endl;
        int avail = 0, locked = 0, booked = 0;
        for (const auto& [id, seat] : show_seats) {
            if (seat.is_available()) avail++;
            else if (seat.get_status() == SeatStatus::LOCKED) locked++;
            else booked++;
        }
        cout << "  Available: " << avail << " | Locked: " << locked
             << " | Booked: " << booked << endl;
    }
};

// ─── Booking ─────────────────────────────────────────────

class Booking {
    static int next_id;

    string booking_id;
    string user_id;
    string show_id;
    vector<string> seat_ids;
    double total_amount;
    BookingStatus status;
    PaymentStatus payment_status;

public:
    Booking(const string& user_id, const string& show_id,
            const vector<string>& seats, double amount)
        : booking_id("BK" + to_string(next_id++)),
          user_id(user_id), show_id(show_id), seat_ids(seats),
          total_amount(amount), status(BookingStatus::PENDING),
          payment_status(PaymentStatus::PENDING) {}

    const string& get_id() const { return booking_id; }
    const string& get_user_id() const { return user_id; }
    const string& get_show_id() const { return show_id; }
    const vector<string>& get_seat_ids() const { return seat_ids; }
    double get_total() const { return total_amount; }
    BookingStatus get_status() const { return status; }

    void confirm() {
        status = BookingStatus::CONFIRMED;
        payment_status = PaymentStatus::SUCCESS;
    }

    void cancel() {
        status = BookingStatus::CANCELLED;
    }

    void payment_failed() {
        payment_status = PaymentStatus::FAILED;
    }

    void print() const {
        cout << "Booking " << booking_id << ": Show " << show_id
             << " | Seats: ";
        for (const auto& s : seat_ids) cout << s << " ";
        cout << "| Total: $" << fixed << setprecision(2) << total_amount
             << " | Status: " << to_string(status) << endl;
    }
};

int Booking::next_id = 1;

// ─── Theater ─────────────────────────────────────────────

class Theater {
    string theater_id;
    string name;
    string city;
    unordered_map<string, Screen> screens;

public:
    Theater(const string& id, const string& name, const string& city)
        : theater_id(id), name(name), city(city) {}

    const string& get_id() const { return theater_id; }
    const string& get_name() const { return name; }
    const string& get_city() const { return city; }

    void add_screen(const Screen& screen) {
        screens.emplace(screen.get_id(), screen);
    }

    const Screen* get_screen(const string& id) const {
        auto it = screens.find(id);
        return (it != screens.end()) ? &it->second : nullptr;
    }

    const unordered_map<string, Screen>& get_screens() const { return screens; }
};

// ─── Booking System (Facade + Singleton) ─────────────────

class BookingSystem {
    unordered_map<string, Movie> movies;
    unordered_map<string, Theater> theaters;
    unordered_map<string, Show> shows;
    unordered_map<string, Booking> bookings;
    mutable mutex mtx;

    BookingSystem() {}

public:
    static BookingSystem& instance() {
        static BookingSystem sys;
        return sys;
    }

    // ─── Setup ───────────────────────────────────────────

    void add_movie(const Movie& movie) {
        lock_guard<mutex> lock(mtx);
        movies.emplace(movie.get_id(), movie);
        cout << "Movie added: " << movie.get_title() << endl;
    }

    void add_theater(const Theater& theater) {
        lock_guard<mutex> lock(mtx);
        theaters.emplace(theater.get_id(), theater);
        cout << "Theater added: " << theater.get_name()
             << " (" << theater.get_city() << ")" << endl;
    }

    void add_show(const string& show_id, const string& movie_id,
                  const string& theater_id, const string& screen_id,
                  const string& start_time, const PricingStrategy& pricing) {
        lock_guard<mutex> lock(mtx);

        auto theater_it = theaters.find(theater_id);
        if (theater_it == theaters.end()) {
            cout << "Error: Theater not found." << endl;
            return;
        }

        const Screen* screen = theater_it->second.get_screen(screen_id);
        if (!screen) {
            cout << "Error: Screen not found." << endl;
            return;
        }

        shows.emplace(show_id,
            Show(show_id, movie_id, screen_id, start_time, *screen, pricing));

        cout << "Show added: " << movies.at(movie_id).get_title()
             << " at " << start_time << " on " << screen->get_name() << endl;
    }

    // ─── Search ──────────────────────────────────────────

    vector<const Show*> search_shows_by_movie(const string& movie_id) const {
        lock_guard<mutex> lock(mtx);
        vector<const Show*> results;
        for (const auto& [id, show] : shows) {
            if (show.get_movie_id() == movie_id) {
                results.push_back(&show);
            }
        }
        return results;
    }

    // ─── Booking Flow ────────────────────────────────────

    // Step 1: View available seats
    void view_available_seats(const string& show_id) const {
        lock_guard<mutex> lock(mtx);
        auto it = shows.find(show_id);
        if (it == shows.end()) {
            cout << "Show not found." << endl;
            return;
        }

        auto available = it->second.get_available_seats();
        cout << "\nAvailable seats for show " << show_id << ":" << endl;
        for (const auto* seat : available) {
            cout << "  " << seat->get_seat_id()
                 << " [" << to_string(seat->get_category()) << "]"
                 << " $" << fixed << setprecision(2) << seat->get_price() << endl;
        }
        cout << "Total available: " << available.size() << endl;
    }

    // Step 2: Select and lock seats
    optional<string> select_seats(const string& show_id,
                                  const vector<string>& seat_ids,
                                  const string& user_id) {
        lock_guard<mutex> lock(mtx);

        auto show_it = shows.find(show_id);
        if (show_it == shows.end()) {
            cout << "Show not found." << endl;
            return nullopt;
        }

        Show& show = show_it->second;

        // Try to lock seats
        if (!show.lock_seats(seat_ids, user_id)) {
            cout << "❌ Some seats are not available. Please try again." << endl;
            return nullopt;
        }

        // Calculate total
        double total = show.calculate_total(seat_ids);

        // Create pending booking
        Booking booking(user_id, show_id, seat_ids, total);
        string booking_id = booking.get_id();
        bookings.emplace(booking_id, booking);

        cout << "🔒 Seats locked for " << user_id << ". Booking: " << booking_id
             << " | Total: $" << fixed << setprecision(2) << total
             << " | Complete payment within 5 minutes." << endl;

        return booking_id;
    }

    // Step 3: Process payment and confirm
    bool confirm_booking(const string& booking_id, bool payment_success = true) {
        lock_guard<mutex> lock(mtx);

        auto bk_it = bookings.find(booking_id);
        if (bk_it == bookings.end()) {
            cout << "Booking not found." << endl;
            return false;
        }

        Booking& booking = bk_it->second;
        Show& show = shows.at(booking.get_show_id());

        if (payment_success) {
            // Book the seats
            if (show.book_seats(booking.get_seat_ids(), booking.get_user_id())) {
                booking.confirm();
                cout << "✅ Booking confirmed! ";
                booking.print();
                return true;
            } else {
                cout << "❌ Failed to book seats (lock may have expired)." << endl;
                booking.payment_failed();
                return false;
            }
        } else {
            // Payment failed — release seats
            show.release_seats(booking.get_seat_ids(), booking.get_user_id());
            booking.payment_failed();
            cout << "❌ Payment failed. Seats released." << endl;
            return false;
        }
    }

    // Cancel booking
    bool cancel_booking(const string& booking_id) {
        lock_guard<mutex> lock(mtx);

        auto bk_it = bookings.find(booking_id);
        if (bk_it == bookings.end()) {
            cout << "Booking not found." << endl;
            return false;
        }

        Booking& booking = bk_it->second;
        if (booking.get_status() != BookingStatus::CONFIRMED) {
            cout << "Can only cancel confirmed bookings." << endl;
            return false;
        }

        Show& show = shows.at(booking.get_show_id());
        show.release_seats(booking.get_seat_ids(), booking.get_user_id());
        booking.cancel();

        cout << "🚫 Booking " << booking_id << " cancelled. Seats released." << endl;
        return true;
    }

    void display_show_status(const string& show_id) const {
        lock_guard<mutex> lock(mtx);
        auto it = shows.find(show_id);
        if (it != shows.end()) {
            it->second.display_seats();
        }
    }

    // Delete copy/move
    BookingSystem(const BookingSystem&) = delete;
    BookingSystem& operator=(const BookingSystem&) = delete;
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    auto& system = BookingSystem::instance();

    // Add movie
    system.add_movie(Movie("MOV1", "Inception", "Sci-Fi", 148, "English"));

    // Add theater with screen
    Theater theater("TH1", "PVR Cinemas", "Mumbai");
    Screen screen("SCR1", "Screen 1", 8, 10, {
        {2, SeatCategory::PLATINUM},  // rows A-B: Platinum
        {3, SeatCategory::GOLD},      // rows C-E: Gold
        {3, SeatCategory::SILVER}     // rows F-H: Silver
    });
    theater.add_screen(screen);
    system.add_theater(theater);

    // Add show
    StandardPricing pricing;
    system.add_show("SH1", "MOV1", "TH1", "SCR1", "7:00 PM", pricing);

    // User 1: View and book seats
    cout << "\n--- User 1: Alice ---" << endl;
    system.view_available_seats("SH1");

    auto booking1 = system.select_seats("SH1", {"A1", "A2"}, "alice");
    if (booking1) {
        system.confirm_booking(*booking1, true);  // payment success
    }

    // User 2: Try to book same seats (should fail)
    cout << "\n--- User 2: Bob (tries same seats) ---" << endl;
    auto booking2 = system.select_seats("SH1", {"A1", "A3"}, "bob");

    // User 2: Book different seats
    cout << "\n--- User 2: Bob (different seats) ---" << endl;
    auto booking3 = system.select_seats("SH1", {"C1", "C2", "C3"}, "bob");
    if (booking3) {
        system.confirm_booking(*booking3, true);
    }

    // User 3: Payment fails
    cout << "\n--- User 3: Charlie (payment fails) ---" << endl;
    auto booking4 = system.select_seats("SH1", {"F1", "F2"}, "charlie");
    if (booking4) {
        system.confirm_booking(*booking4, false);  // payment fails
    }

    // Show status
    cout << "\n--- Show Status ---" << endl;
    system.display_show_status("SH1");

    // Cancel booking
    cout << "\n--- Cancel Alice's booking ---" << endl;
    if (booking1) system.cancel_booking(*booking1);

    system.display_show_status("SH1");

    return 0;
}
```

---

### Booking Flow (Sequence)

```
User                    System                  Show
 │                        │                       │
 │  view_available_seats  │                       │
 │───────────────────────▶│  get_available_seats  │
 │                        │──────────────────────▶│
 │  available seats list  │                       │
 │◀───────────────────────│◀──────────────────────│
 │                        │                       │
 │  select_seats(A1, A2)  │                       │
 │───────────────────────▶│  lock_seats(A1, A2)   │
 │                        │──────────────────────▶│
 │  booking_id + total    │  seats locked (5 min) │
 │◀───────────────────────│◀──────────────────────│
 │                        │                       │
 │  confirm(payment=true) │                       │
 │───────────────────────▶│  book_seats(A1, A2)   │
 │                        │──────────────────────▶│
 │  booking confirmed ✅  │  seats booked         │
 │◀───────────────────────│◀──────────────────────│
```

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Singleton** | `BookingSystem` | One centralized booking system |
| **Strategy** | `PricingStrategy` (Standard, Weekend) | Different pricing for different days/events without changing booking logic |
| **Observer** | Console notifications on booking events | Extensible for email/SMS notifications |
| **State** | `SeatStatus`, `BookingStatus` | Seat and booking behavior changes based on state (available → locked → booked) |

### Key Design Decisions

1. **Two-phase booking (lock then book)** — seats are locked temporarily when selected, then permanently booked after payment. This prevents double booking while allowing payment processing time
2. **Atomic seat locking** — all requested seats are checked for availability before any are locked. If any seat is unavailable, none are locked (all-or-nothing)
3. **Lock expiry** — locked seats automatically become available after 5 minutes, preventing indefinite holds from abandoned sessions
4. **ShowSeat separation** — `Seat` defines the physical seat (row, col, category); `ShowSeat` tracks the status for a specific show. The same physical seat can be available for one show and booked for another
5. **Pricing strategy** — decoupled from seat/show logic, allowing dynamic pricing (weekday, weekend, holiday, special event) without modifying the booking flow

---


## 13.6 Online Shopping (Amazon)

> Design an online shopping system like Amazon that supports products, shopping cart, orders, inventory management, payment processing, order tracking, and a discount/coupon system. This is one of the most comprehensive LLD problems, testing your ability to model a complex e-commerce workflow with multiple interacting subsystems.

---

### Requirements

**Functional Requirements:**
1. Products have name, description, price, category, and inventory count
2. Users can search products by name or category
3. Users can add/remove products to/from a shopping cart
4. Users can place an order from their cart
5. Inventory is decremented when an order is placed and restored on cancellation
6. Multiple payment methods are supported (Credit Card, Debit Card, UPI)
7. Orders go through states: Placed → Confirmed → Shipped → Delivered (or Cancelled)
8. Users can apply discount coupons (percentage off, flat off, buy-one-get-one)
9. Users can view order history and track order status
10. The system prevents ordering out-of-stock items

**Non-Functional Requirements:**
- Thread-safe for concurrent cart operations and order placement
- Extensible for new payment methods and discount types
- Inventory consistency under concurrent orders

---

### Key Entities & Class Design

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│    User      │──────▶│    Cart      │──────▶│  CartItem    │
│              │       │              │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
       │                                             │
       │                                             ▼
       │                                      ┌──────────────┐
       │                                      │   Product    │
       │                                      │              │
       │                                      └──────────────┘
       │                                             ▲
       ▼                                             │
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│    Order     │──────▶│  OrderItem   │───────│  Inventory   │
│              │       │              │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
       │
       ▼
┌──────────────┐       ┌──────────────┐
│   Payment    │       │   Coupon     │
│  (Strategy)  │       │ (Decorator)  │
└──────────────┘       └──────────────┘
```

---

### Implementation

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <memory>
#include <mutex>
#include <optional>
#include <algorithm>
#include <iomanip>
#include <chrono>
using namespace std;

// ─── Enums ───────────────────────────────────────────────

enum class OrderStatus { PLACED, CONFIRMED, SHIPPED, DELIVERED, CANCELLED };
enum class PaymentMethod { CREDIT_CARD, DEBIT_CARD, UPI };

string to_string(OrderStatus s) {
    switch (s) {
        case OrderStatus::PLACED:    return "Placed";
        case OrderStatus::CONFIRMED: return "Confirmed";
        case OrderStatus::SHIPPED:   return "Shipped";
        case OrderStatus::DELIVERED: return "Delivered";
        case OrderStatus::CANCELLED: return "Cancelled";
    }
    return "Unknown";
}

string to_string(PaymentMethod m) {
    switch (m) {
        case PaymentMethod::CREDIT_CARD: return "Credit Card";
        case PaymentMethod::DEBIT_CARD:  return "Debit Card";
        case PaymentMethod::UPI:         return "UPI";
    }
    return "Unknown";
}

// ─── Product ─────────────────────────────────────────────

class Product {
    string product_id;
    string name;
    string description;
    string category;
    double price;
    int stock;
    mutable mutex mtx;

public:
    Product(const string& id, const string& name, const string& desc,
            const string& category, double price, int stock)
        : product_id(id), name(name), description(desc),
          category(category), price(price), stock(stock) {}

    // Copy constructor (mutex is not copyable)
    Product(const Product& other)
        : product_id(other.product_id), name(other.name),
          description(other.description), category(other.category),
          price(other.price), stock(other.stock) {}

    const string& get_id() const { return product_id; }
    const string& get_name() const { return name; }
    const string& get_description() const { return description; }
    const string& get_category() const { return category; }
    double get_price() const { return price; }

    int get_stock() const {
        lock_guard<mutex> lock(mtx);
        return stock;
    }

    bool is_in_stock(int quantity = 1) const {
        lock_guard<mutex> lock(mtx);
        return stock >= quantity;
    }

    bool reserve(int quantity) {
        lock_guard<mutex> lock(mtx);
        if (stock < quantity) return false;
        stock -= quantity;
        return true;
    }

    void restock(int quantity) {
        lock_guard<mutex> lock(mtx);
        stock += quantity;
    }

    void print() const {
        cout << product_id << ": " << name << " - $"
             << fixed << setprecision(2) << price
             << " [" << category << "] (Stock: " << get_stock() << ")" << endl;
    }
};

// ─── Cart Item ───────────────────────────────────────────

struct CartItem {
    string product_id;
    string product_name;
    double unit_price;
    int quantity;

    double total() const { return unit_price * quantity; }
};

// ─── Shopping Cart ───────────────────────────────────────

class ShoppingCart {
    unordered_map<string, CartItem> items;  // product_id → CartItem
    mutable mutex mtx;

public:
    void add_item(const Product& product, int quantity = 1) {
        lock_guard<mutex> lock(mtx);
        auto it = items.find(product.get_id());
        if (it != items.end()) {
            it->second.quantity += quantity;
        } else {
            items[product.get_id()] = {
                product.get_id(), product.get_name(),
                product.get_price(), quantity
            };
        }
        cout << "  Added " << quantity << "x " << product.get_name()
             << " to cart" << endl;
    }

    void remove_item(const string& product_id) {
        lock_guard<mutex> lock(mtx);
        items.erase(product_id);
    }

    void update_quantity(const string& product_id, int quantity) {
        lock_guard<mutex> lock(mtx);
        if (quantity <= 0) {
            items.erase(product_id);
        } else {
            auto it = items.find(product_id);
            if (it != items.end()) {
                it->second.quantity = quantity;
            }
        }
    }

    double get_total() const {
        lock_guard<mutex> lock(mtx);
        double total = 0;
        for (const auto& [id, item] : items) {
            total += item.total();
        }
        return total;
    }

    vector<CartItem> get_items() const {
        lock_guard<mutex> lock(mtx);
        vector<CartItem> result;
        for (const auto& [id, item] : items) {
            result.push_back(item);
        }
        return result;
    }

    bool is_empty() const {
        lock_guard<mutex> lock(mtx);
        return items.empty();
    }

    void clear() {
        lock_guard<mutex> lock(mtx);
        items.clear();
    }

    void print() const {
        lock_guard<mutex> lock(mtx);
        cout << "Shopping Cart:" << endl;
        if (items.empty()) {
            cout << "  (empty)" << endl;
            return;
        }
        for (const auto& [id, item] : items) {
            cout << "  " << item.product_name << " x" << item.quantity
                 << " @ $" << fixed << setprecision(2) << item.unit_price
                 << " = $" << item.total() << endl;
        }
        double total = 0;
        for (const auto& [id, item] : items) total += item.total();
        cout << "  Total: $" << fixed << setprecision(2) << total << endl;
    }
};

// ─── Coupon / Discount (Decorator Pattern) ───────────────

class DiscountStrategy {
public:
    virtual ~DiscountStrategy() = default;
    virtual double apply(double total) const = 0;
    virtual string description() const = 0;
    virtual bool is_valid() const = 0;
};

class PercentageDiscount : public DiscountStrategy {
    string code;
    double percentage;
    double max_discount;

public:
    PercentageDiscount(const string& code, double pct, double max_disc = 1e9)
        : code(code), percentage(pct), max_discount(max_disc) {}

    double apply(double total) const override {
        double discount = total * percentage / 100.0;
        return min(discount, max_discount);
    }

    string description() const override {
        return code + ": " + to_string((int)percentage) + "% off"
               + (max_discount < 1e9 ? " (max $" + to_string((int)max_discount) + ")" : "");
    }

    bool is_valid() const override { return true; }
};

class FlatDiscount : public DiscountStrategy {
    string code;
    double amount;
    double min_order;

public:
    FlatDiscount(const string& code, double amount, double min_order = 0)
        : code(code), amount(amount), min_order(min_order) {}

    double apply(double total) const override {
        if (total < min_order) return 0;
        return min(amount, total);
    }

    string description() const override {
        return code + ": $" + to_string((int)amount) + " off"
               + (min_order > 0 ? " (min order $" + to_string((int)min_order) + ")" : "");
    }

    bool is_valid() const override { return true; }
};

// ─── Payment Strategy ────────────────────────────────────

class PaymentProcessor {
public:
    virtual ~PaymentProcessor() = default;
    virtual bool process(double amount) = 0;
    virtual string name() const = 0;
};

class CreditCardPayment : public PaymentProcessor {
    string card_number;
public:
    CreditCardPayment(const string& card) : card_number(card) {}

    bool process(double amount) override {
        cout << "  💳 Processing Credit Card payment of $"
             << fixed << setprecision(2) << amount
             << " (card: ****" << card_number.substr(card_number.size() - 4)
             << ")" << endl;
        return true;  // simulate success
    }

    string name() const override { return "Credit Card"; }
};

class UPIPayment : public PaymentProcessor {
    string upi_id;
public:
    UPIPayment(const string& upi) : upi_id(upi) {}

    bool process(double amount) override {
        cout << "  📱 Processing UPI payment of $"
             << fixed << setprecision(2) << amount
             << " (UPI: " << upi_id << ")" << endl;
        return true;
    }

    string name() const override { return "UPI"; }
};

// ─── Order ───────────────────────────────────────────────

struct OrderItem {
    string product_id;
    string product_name;
    double unit_price;
    int quantity;
    double total() const { return unit_price * quantity; }
};

class Order {
    static int next_id;

    string order_id;
    string user_id;
    vector<OrderItem> items;
    double subtotal;
    double discount;
    double total;
    OrderStatus status;
    string payment_method;

public:
    Order(const string& user_id, const vector<CartItem>& cart_items,
          double subtotal, double discount, const string& payment)
        : order_id("ORD" + to_string(next_id++)),
          user_id(user_id), subtotal(subtotal), discount(discount),
          total(subtotal - discount), status(OrderStatus::PLACED),
          payment_method(payment) {

        for (const auto& ci : cart_items) {
            items.push_back({ci.product_id, ci.product_name,
                             ci.unit_price, ci.quantity});
        }
    }

    const string& get_id() const { return order_id; }
    const string& get_user_id() const { return user_id; }
    const vector<OrderItem>& get_items() const { return items; }
    double get_total() const { return total; }
    OrderStatus get_status() const { return status; }

    void set_status(OrderStatus s) { status = s; }

    void print() const {
        cout << "Order " << order_id << " [" << to_string(status) << "]" << endl;
        for (const auto& item : items) {
            cout << "  " << item.product_name << " x" << item.quantity
                 << " @ $" << fixed << setprecision(2) << item.unit_price
                 << " = $" << item.total() << endl;
        }
        cout << "  Subtotal: $" << fixed << setprecision(2) << subtotal << endl;
        if (discount > 0) {
            cout << "  Discount: -$" << fixed << setprecision(2) << discount << endl;
        }
        cout << "  Total: $" << fixed << setprecision(2) << total << endl;
        cout << "  Payment: " << payment_method << endl;
    }
};

int Order::next_id = 1;

// ─── Shopping System (Facade) ────────────────────────────

class ShoppingSystem {
    unordered_map<string, Product> products;
    unordered_map<string, ShoppingCart> carts;       // user_id → cart
    unordered_map<string, vector<Order>> orders;     // user_id → order history
    unordered_map<string, unique_ptr<DiscountStrategy>> coupons;  // code → discount
    mutable mutex mtx;

public:
    // ─── Product Management ──────────────────────────────

    void add_product(const Product& product) {
        lock_guard<mutex> lock(mtx);
        products.emplace(product.get_id(), product);
        cout << "Product added: ";
        product.print();
    }

    vector<const Product*> search_by_name(const string& query) const {
        lock_guard<mutex> lock(mtx);
        vector<const Product*> results;
        string lower_query = query;
        transform(lower_query.begin(), lower_query.end(),
                  lower_query.begin(), ::tolower);

        for (const auto& [id, product] : products) {
            string lower_name = product.get_name();
            transform(lower_name.begin(), lower_name.end(),
                      lower_name.begin(), ::tolower);
            if (lower_name.find(lower_query) != string::npos) {
                results.push_back(&product);
            }
        }
        return results;
    }

    vector<const Product*> search_by_category(const string& category) const {
        lock_guard<mutex> lock(mtx);
        vector<const Product*> results;
        for (const auto& [id, product] : products) {
            if (product.get_category() == category) {
                results.push_back(&product);
            }
        }
        return results;
    }

    // ─── Cart Operations ─────────────────────────────────

    void add_to_cart(const string& user_id, const string& product_id, int qty = 1) {
        lock_guard<mutex> lock(mtx);
        auto prod_it = products.find(product_id);
        if (prod_it == products.end()) {
            cout << "Product not found." << endl;
            return;
        }

        if (!prod_it->second.is_in_stock(qty)) {
            cout << "Insufficient stock for " << prod_it->second.get_name() << endl;
            return;
        }

        carts[user_id].add_item(prod_it->second, qty);
    }

    void remove_from_cart(const string& user_id, const string& product_id) {
        lock_guard<mutex> lock(mtx);
        carts[user_id].remove_item(product_id);
    }

    void view_cart(const string& user_id) const {
        lock_guard<mutex> lock(mtx);
        auto it = carts.find(user_id);
        if (it == carts.end() || it->second.is_empty()) {
            cout << "Cart is empty." << endl;
            return;
        }
        it->second.print();
    }

    // ─── Coupon Management ───────────────────────────────

    void add_coupon(const string& code, unique_ptr<DiscountStrategy> discount) {
        lock_guard<mutex> lock(mtx);
        cout << "Coupon added: " << discount->description() << endl;
        coupons[code] = std::move(discount);
    }

    // ─── Place Order ─────────────────────────────────────

    optional<string> place_order(const string& user_id,
                                 unique_ptr<PaymentProcessor> payment,
                                 const string& coupon_code = "") {
        lock_guard<mutex> lock(mtx);

        auto cart_it = carts.find(user_id);
        if (cart_it == carts.end() || cart_it->second.is_empty()) {
            cout << "Cart is empty. Cannot place order." << endl;
            return nullopt;
        }

        ShoppingCart& cart = cart_it->second;
        auto cart_items = cart.get_items();
        double subtotal = cart.get_total();

        // Validate stock for all items
        for (const auto& item : cart_items) {
            auto& product = products.at(item.product_id);
            if (!product.is_in_stock(item.quantity)) {
                cout << "❌ " << item.product_name << " is out of stock." << endl;
                return nullopt;
            }
        }

        // Apply coupon
        double discount = 0;
        if (!coupon_code.empty()) {
            auto coupon_it = coupons.find(coupon_code);
            if (coupon_it != coupons.end() && coupon_it->second->is_valid()) {
                discount = coupon_it->second->apply(subtotal);
                cout << "  🎟️ Coupon applied: " << coupon_it->second->description()
                     << " (-$" << fixed << setprecision(2) << discount << ")" << endl;
            } else {
                cout << "  Invalid coupon code: " << coupon_code << endl;
            }
        }

        double total = subtotal - discount;

        // Process payment
        if (!payment->process(total)) {
            cout << "❌ Payment failed." << endl;
            return nullopt;
        }

        // Reserve inventory
        for (const auto& item : cart_items) {
            products.at(item.product_id).reserve(item.quantity);
        }

        // Create order
        Order order(user_id, cart_items, subtotal, discount, payment->name());
        order.set_status(OrderStatus::CONFIRMED);
        string order_id = order.get_id();

        cout << "✅ Order placed!" << endl;
        order.print();

        orders[user_id].push_back(order);
        cart.clear();

        return order_id;
    }

    // ─── Cancel Order ────────────────────────────────────

    bool cancel_order(const string& user_id, const string& order_id) {
        lock_guard<mutex> lock(mtx);

        auto orders_it = orders.find(user_id);
        if (orders_it == orders.end()) {
            cout << "No orders found." << endl;
            return false;
        }

        for (auto& order : orders_it->second) {
            if (order.get_id() == order_id) {
                if (order.get_status() == OrderStatus::DELIVERED) {
                    cout << "Cannot cancel a delivered order." << endl;
                    return false;
                }
                if (order.get_status() == OrderStatus::CANCELLED) {
                    cout << "Order already cancelled." << endl;
                    return false;
                }

                // Restore inventory
                for (const auto& item : order.get_items()) {
                    products.at(item.product_id).restock(item.quantity);
                }

                order.set_status(OrderStatus::CANCELLED);
                cout << "🚫 Order " << order_id << " cancelled. "
                     << "Inventory restored. Refund of $"
                     << fixed << setprecision(2) << order.get_total()
                     << " initiated." << endl;
                return true;
            }
        }

        cout << "Order not found." << endl;
        return false;
    }

    // ─── Order History ───────────────────────────────────

    void view_orders(const string& user_id) const {
        lock_guard<mutex> lock(mtx);
        auto it = orders.find(user_id);
        if (it == orders.end() || it->second.empty()) {
            cout << "No orders found for " << user_id << endl;
            return;
        }

        cout << "\n=== Order History for " << user_id << " ===" << endl;
        for (const auto& order : it->second) {
            order.print();
            cout << "---" << endl;
        }
    }

    // ─── Update Order Status ─────────────────────────────

    void update_order_status(const string& user_id, const string& order_id,
                             OrderStatus new_status) {
        lock_guard<mutex> lock(mtx);
        auto orders_it = orders.find(user_id);
        if (orders_it == orders.end()) return;

        for (auto& order : orders_it->second) {
            if (order.get_id() == order_id) {
                order.set_status(new_status);
                cout << "📦 Order " << order_id << " status updated to: "
                     << to_string(new_status) << endl;
                return;
            }
        }
    }
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    ShoppingSystem shop;

    // Add products
    shop.add_product(Product("P1", "Laptop", "High-performance laptop",
                             "Electronics", 999.99, 10));
    shop.add_product(Product("P2", "Headphones", "Noise-cancelling headphones",
                             "Electronics", 199.99, 50));
    shop.add_product(Product("P3", "C++ Book", "The C++ Programming Language",
                             "Books", 49.99, 30));
    shop.add_product(Product("P4", "Mouse", "Wireless ergonomic mouse",
                             "Electronics", 29.99, 100));

    // Add coupons
    shop.add_coupon("SAVE10", make_unique<PercentageDiscount>("SAVE10", 10, 100));
    shop.add_coupon("FLAT50", make_unique<FlatDiscount>("FLAT50", 50, 200));

    // User shopping
    cout << "\n--- Alice Shopping ---" << endl;

    // Search
    cout << "Search 'laptop':" << endl;
    auto results = shop.search_by_name("laptop");
    for (auto* p : results) p->print();

    // Add to cart
    shop.add_to_cart("alice", "P1", 1);
    shop.add_to_cart("alice", "P2", 2);
    shop.add_to_cart("alice", "P3", 1);

    shop.view_cart("alice");

    // Place order with coupon
    cout << "\n--- Place Order ---" << endl;
    auto order_id = shop.place_order(
        "alice",
        make_unique<CreditCardPayment>("4111111111111111"),
        "SAVE10"
    );

    // View order history
    shop.view_orders("alice");

    // Update order status
    if (order_id) {
        shop.update_order_status("alice", *order_id, OrderStatus::SHIPPED);
        shop.update_order_status("alice", *order_id, OrderStatus::DELIVERED);
    }

    // Bob tries to order
    cout << "\n--- Bob Shopping ---" << endl;
    shop.add_to_cart("bob", "P4", 3);
    shop.place_order("bob", make_unique<UPIPayment>("bob@upi"), "FLAT50");

    return 0;
}
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `PaymentProcessor` (CreditCard, UPI) | Swap payment methods without changing order logic |
| **Strategy** | `DiscountStrategy` (Percentage, Flat) | Different discount types applied uniformly |
| **Observer** | Console notifications on order events | Extensible for email/push notifications |
| **Factory** | Order ID generation | Automatic unique ID creation |
| **Facade** | `ShoppingSystem` | Simple interface hiding complex interactions between products, cart, inventory, payment, and orders |
| **Decorator** | Coupon applied on top of base price | Discounts modify the total without changing the order structure |

### Key Design Decisions

1. **Cart per user** — each user has an independent shopping cart, stored in a map. Cart operations are thread-safe
2. **Inventory reservation** — stock is decremented atomically when an order is placed, not when items are added to cart. This prevents phantom inventory issues
3. **All-or-nothing ordering** — if any item is out of stock, the entire order fails. No partial orders
4. **Coupon validation** — coupons are validated and applied before payment. Invalid coupons are silently ignored (order proceeds without discount)
5. **Order state machine** — orders follow a strict state progression (Placed → Confirmed → Shipped → Delivered). Cancellation is only allowed before delivery
6. **Inventory restoration on cancel** — cancelled orders restore the reserved inventory, making items available for other customers

---


## 13.7 Car Rental System

> Design a car rental system that supports multiple vehicle types, reservations, pricing strategies, availability checking, and add-on services like insurance. This problem tests your ability to model a reservation-based system with time-based availability and configurable pricing.

---

### Requirements

**Functional Requirements:**
1. The system manages a fleet of vehicles of different types (Sedan, SUV, Hatchback, Luxury)
2. Customers can search for available vehicles by type, date range, and location
3. Customers can make a reservation for a specific vehicle and date range
4. Pricing varies by vehicle type and rental duration (daily rate, weekly rate)
5. Customers can add optional services (insurance, GPS, child seat, roadside assistance)
6. The system prevents double-booking of the same vehicle
7. Customers can cancel reservations (with cancellation policy)
8. Vehicles can be picked up and returned at different locations
9. Late returns incur additional charges

**Non-Functional Requirements:**
- Thread-safe for concurrent reservation requests
- Extensible for new vehicle types, pricing models, and add-on services
- Accurate date-based availability checking

---

### Key Entities & Class Design

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ RentalSystem │──────▶│   Vehicle    │       │  Customer    │
│   (Facade)   │       │              │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
       │                      │                      │
       │                      │                      │
       ▼                      ▼                      ▼
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ Reservation  │──────▶│PricingStrategy│      │   Add-On     │
│              │       │(Daily/Weekly) │       │  (Builder)   │
└──────────────┘       └──────────────┘       └──────────────┘
```

---

### Implementation

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <unordered_set>
#include <memory>
#include <mutex>
#include <optional>
#include <algorithm>
#include <iomanip>
#include <chrono>
#include <numeric>
using namespace std;

// ─── Enums ───────────────────────────────────────────────

enum class VehicleType { HATCHBACK, SEDAN, SUV, LUXURY };
enum class VehicleStatus { AVAILABLE, RESERVED, RENTED, MAINTENANCE };
enum class ReservationStatus { ACTIVE, COMPLETED, CANCELLED };

string to_string(VehicleType t) {
    switch (t) {
        case VehicleType::HATCHBACK: return "Hatchback";
        case VehicleType::SEDAN:     return "Sedan";
        case VehicleType::SUV:       return "SUV";
        case VehicleType::LUXURY:    return "Luxury";
    }
    return "Unknown";
}

string to_string(ReservationStatus s) {
    switch (s) {
        case ReservationStatus::ACTIVE:    return "Active";
        case ReservationStatus::COMPLETED: return "Completed";
        case ReservationStatus::CANCELLED: return "Cancelled";
    }
    return "Unknown";
}

// ─── Date (simplified) ──────────────────────────────────

struct RentalDate {
    int year, month, day;

    bool operator<(const RentalDate& other) const {
        if (year != other.year) return year < other.year;
        if (month != other.month) return month < other.month;
        return day < other.day;
    }

    bool operator<=(const RentalDate& other) const {
        return !(other < *this);
    }

    bool operator==(const RentalDate& other) const {
        return year == other.year && month == other.month && day == other.day;
    }

    int days_until(const RentalDate& other) const {
        // Simplified: assume 30 days per month
        return (other.year - year) * 365 + (other.month - month) * 30
               + (other.day - day);
    }

    string to_string() const {
        return std::to_string(year) + "-"
               + (month < 10 ? "0" : "") + std::to_string(month) + "-"
               + (day < 10 ? "0" : "") + std::to_string(day);
    }

    // Check if two date ranges overlap
    static bool overlaps(const RentalDate& s1, const RentalDate& e1,
                         const RentalDate& s2, const RentalDate& e2) {
        return s1 < e2 && s2 < e1;
    }
};

// ─── Vehicle ─────────────────────────────────────────────

class Vehicle {
    string vehicle_id;
    string make;
    string model;
    int year;
    VehicleType type;
    string license_plate;
    string location;
    int mileage;

public:
    Vehicle(const string& id, const string& make, const string& model,
            int year, VehicleType type, const string& plate,
            const string& location)
        : vehicle_id(id), make(make), model(model), year(year),
          type(type), license_plate(plate), location(location), mileage(0) {}

    const string& get_id() const { return vehicle_id; }
    const string& get_make() const { return make; }
    const string& get_model() const { return model; }
    int get_year() const { return year; }
    VehicleType get_type() const { return type; }
    const string& get_plate() const { return license_plate; }
    const string& get_location() const { return location; }

    void set_location(const string& loc) { location = loc; }

    void print() const {
        cout << vehicle_id << ": " << year << " " << make << " " << model
             << " [" << to_string(type) << "] Plate: " << license_plate
             << " @ " << location << endl;
    }
};

// ─── Customer ────────────────────────────────────────────

class Customer {
    string customer_id;
    string name;
    string email;
    string license_number;

public:
    Customer(const string& id, const string& name, const string& email,
             const string& license)
        : customer_id(id), name(name), email(email), license_number(license) {}

    const string& get_id() const { return customer_id; }
    const string& get_name() const { return name; }
    const string& get_email() const { return email; }
};

// ─── Add-On Service (Builder Pattern) ────────────────────

struct AddOn {
    string name;
    double daily_rate;

    AddOn(const string& n, double rate) : name(n), daily_rate(rate) {}
};

class AddOnBuilder {
    vector<AddOn> add_ons;

public:
    AddOnBuilder& with_insurance(double rate = 15.0) {
        add_ons.emplace_back("Insurance", rate);
        return *this;
    }

    AddOnBuilder& with_gps(double rate = 5.0) {
        add_ons.emplace_back("GPS Navigation", rate);
        return *this;
    }

    AddOnBuilder& with_child_seat(double rate = 8.0) {
        add_ons.emplace_back("Child Seat", rate);
        return *this;
    }

    AddOnBuilder& with_roadside_assistance(double rate = 10.0) {
        add_ons.emplace_back("Roadside Assistance", rate);
        return *this;
    }

    vector<AddOn> build() const { return add_ons; }

    double daily_total() const {
        double total = 0;
        for (const auto& a : add_ons) total += a.daily_rate;
        return total;
    }
};

// ─── Pricing Strategy (Strategy Pattern) ─────────────────

class PricingStrategy {
public:
    virtual ~PricingStrategy() = default;
    virtual double calculate(VehicleType type, int days) const = 0;
    virtual string name() const = 0;
};

class DailyPricing : public PricingStrategy {
    unordered_map<int, double> rates;  // VehicleType → daily rate

public:
    DailyPricing() {
        rates[static_cast<int>(VehicleType::HATCHBACK)] = 30.0;
        rates[static_cast<int>(VehicleType::SEDAN)]     = 50.0;
        rates[static_cast<int>(VehicleType::SUV)]       = 75.0;
        rates[static_cast<int>(VehicleType::LUXURY)]    = 150.0;
    }

    double calculate(VehicleType type, int days) const override {
        return rates.at(static_cast<int>(type)) * days;
    }

    string name() const override { return "Daily"; }
};

class WeeklyPricing : public PricingStrategy {
    unordered_map<int, double> weekly_rates;
    unordered_map<int, double> daily_rates;

public:
    WeeklyPricing() {
        weekly_rates[static_cast<int>(VehicleType::HATCHBACK)] = 180.0;
        weekly_rates[static_cast<int>(VehicleType::SEDAN)]     = 300.0;
        weekly_rates[static_cast<int>(VehicleType::SUV)]       = 450.0;
        weekly_rates[static_cast<int>(VehicleType::LUXURY)]    = 900.0;

        daily_rates[static_cast<int>(VehicleType::HATCHBACK)] = 30.0;
        daily_rates[static_cast<int>(VehicleType::SEDAN)]     = 50.0;
        daily_rates[static_cast<int>(VehicleType::SUV)]       = 75.0;
        daily_rates[static_cast<int>(VehicleType::LUXURY)]    = 150.0;
    }

    double calculate(VehicleType type, int days) const override {
        int weeks = days / 7;
        int remaining_days = days % 7;
        int key = static_cast<int>(type);
        return weeks * weekly_rates.at(key) + remaining_days * daily_rates.at(key);
    }

    string name() const override { return "Weekly"; }
};

// ─── Reservation ─────────────────────────────────────────

class Reservation {
    static int next_id;

    string reservation_id;
    string customer_id;
    string vehicle_id;
    RentalDate start_date;
    RentalDate end_date;
    string pickup_location;
    string return_location;
    vector<AddOn> add_ons;
    double vehicle_cost;
    double addon_cost;
    double total_cost;
    ReservationStatus status;

public:
    Reservation(const string& customer_id, const string& vehicle_id,
                const RentalDate& start, const RentalDate& end,
                const string& pickup, const string& return_loc,
                const vector<AddOn>& addons,
                double vehicle_cost, double addon_cost)
        : reservation_id("RES" + std::to_string(next_id++)),
          customer_id(customer_id), vehicle_id(vehicle_id),
          start_date(start), end_date(end),
          pickup_location(pickup), return_location(return_loc),
          add_ons(addons), vehicle_cost(vehicle_cost), addon_cost(addon_cost),
          total_cost(vehicle_cost + addon_cost),
          status(ReservationStatus::ACTIVE) {}

    const string& get_id() const { return reservation_id; }
    const string& get_customer_id() const { return customer_id; }
    const string& get_vehicle_id() const { return vehicle_id; }
    const RentalDate& get_start() const { return start_date; }
    const RentalDate& get_end() const { return end_date; }
    double get_total() const { return total_cost; }
    ReservationStatus get_status() const { return status; }

    void set_status(ReservationStatus s) { status = s; }

    bool overlaps_with(const RentalDate& s, const RentalDate& e) const {
        return status == ReservationStatus::ACTIVE
               && RentalDate::overlaps(start_date, end_date, s, e);
    }

    void print() const {
        cout << "Reservation " << reservation_id << " ["
             << to_string(status) << "]" << endl;
        cout << "  Vehicle: " << vehicle_id << endl;
        cout << "  Period: " << start_date.to_string() << " to "
             << end_date.to_string() << " ("
             << start_date.days_until(end_date) << " days)" << endl;
        cout << "  Pickup: " << pickup_location
             << " | Return: " << return_location << endl;
        cout << "  Vehicle cost: $" << fixed << setprecision(2) << vehicle_cost << endl;
        if (!add_ons.empty()) {
            cout << "  Add-ons:" << endl;
            for (const auto& a : add_ons) {
                cout << "    " << a.name << ": $" << a.daily_rate << "/day" << endl;
            }
            cout << "  Add-on cost: $" << fixed << setprecision(2) << addon_cost << endl;
        }
        cout << "  Total: $" << fixed << setprecision(2) << total_cost << endl;
    }
};

int Reservation::next_id = 1;

// ─── Rental System (Facade) ─────────────────────────────

class RentalSystem {
    unordered_map<string, Vehicle> vehicles;
    unordered_map<string, Customer> customers;
    vector<Reservation> reservations;
    unique_ptr<PricingStrategy> pricing;
    mutable mutex mtx;

public:
    RentalSystem() : pricing(make_unique<DailyPricing>()) {}

    void set_pricing(unique_ptr<PricingStrategy> strategy) {
        lock_guard<mutex> lock(mtx);
        pricing = std::move(strategy);
        cout << "Pricing strategy set to: " << pricing->name() << endl;
    }

    void add_vehicle(const Vehicle& vehicle) {
        lock_guard<mutex> lock(mtx);
        vehicles.emplace(vehicle.get_id(), vehicle);
        cout << "Vehicle added: ";
        vehicle.print();
    }

    void add_customer(const Customer& customer) {
        lock_guard<mutex> lock(mtx);
        customers.emplace(customer.get_id(), customer);
        cout << "Customer registered: " << customer.get_name() << endl;
    }

    // ─── Search Available Vehicles ───────────────────────

    vector<const Vehicle*> search_available(
        VehicleType type, const RentalDate& start, const RentalDate& end,
        const string& location = "") const {

        lock_guard<mutex> lock(mtx);
        vector<const Vehicle*> results;

        for (const auto& [id, vehicle] : vehicles) {
            if (vehicle.get_type() != type) continue;
            if (!location.empty() && vehicle.get_location() != location) continue;

            // Check if vehicle is available for the date range
            bool available = true;
            for (const auto& res : reservations) {
                if (res.get_vehicle_id() == id && res.overlaps_with(start, end)) {
                    available = false;
                    break;
                }
            }

            if (available) {
                results.push_back(&vehicle);
            }
        }

        return results;
    }

    // ─── Make Reservation ────────────────────────────────

    optional<string> make_reservation(
        const string& customer_id, const string& vehicle_id,
        const RentalDate& start, const RentalDate& end,
        const string& pickup_location, const string& return_location,
        const AddOnBuilder& addons = AddOnBuilder()) {

        lock_guard<mutex> lock(mtx);

        // Validate customer
        if (customers.find(customer_id) == customers.end()) {
            cout << "Customer not found." << endl;
            return nullopt;
        }

        // Validate vehicle
        auto veh_it = vehicles.find(vehicle_id);
        if (veh_it == vehicles.end()) {
            cout << "Vehicle not found." << endl;
            return nullopt;
        }

        // Check availability
        for (const auto& res : reservations) {
            if (res.get_vehicle_id() == vehicle_id && res.overlaps_with(start, end)) {
                cout << "❌ Vehicle is not available for the requested dates." << endl;
                return nullopt;
            }
        }

        // Calculate pricing
        int days = start.days_until(end);
        if (days <= 0) {
            cout << "Invalid date range." << endl;
            return nullopt;
        }

        double vehicle_cost = pricing->calculate(veh_it->second.get_type(), days);
        double addon_cost = addons.daily_total() * days;

        // Create reservation
        Reservation reservation(customer_id, vehicle_id, start, end,
                                pickup_location, return_location,
                                addons.build(), vehicle_cost, addon_cost);
        string res_id = reservation.get_id();

        cout << "✅ Reservation created!" << endl;
        reservation.print();

        reservations.push_back(reservation);
        return res_id;
    }

    // ─── Cancel Reservation ──────────────────────────────

    bool cancel_reservation(const string& reservation_id) {
        lock_guard<mutex> lock(mtx);

        for (auto& res : reservations) {
            if (res.get_id() == reservation_id) {
                if (res.get_status() != ReservationStatus::ACTIVE) {
                    cout << "Reservation is not active." << endl;
                    return false;
                }
                res.set_status(ReservationStatus::CANCELLED);
                cout << "🚫 Reservation " << reservation_id << " cancelled." << endl;
                return true;
            }
        }

        cout << "Reservation not found." << endl;
        return false;
    }

    // ─── Complete Reservation (Return Vehicle) ───────────

    double complete_reservation(const string& reservation_id,
                                const RentalDate& actual_return_date) {
        lock_guard<mutex> lock(mtx);

        for (auto& res : reservations) {
            if (res.get_id() == reservation_id) {
                if (res.get_status() != ReservationStatus::ACTIVE) {
                    cout << "Reservation is not active." << endl;
                    return -1;
                }

                double total = res.get_total();

                // Check for late return
                if (res.get_end() < actual_return_date) {
                    int late_days = res.get_end().days_until(actual_return_date);
                    auto& vehicle = vehicles.at(res.get_vehicle_id());
                    double late_fee = pricing->calculate(vehicle.get_type(), late_days) * 1.5;
                    total += late_fee;
                    cout << "⚠️ Late return by " << late_days << " days. "
                         << "Late fee: $" << fixed << setprecision(2) << late_fee << endl;
                }

                res.set_status(ReservationStatus::COMPLETED);
                cout << "✅ Reservation " << reservation_id << " completed. "
                     << "Total charge: $" << fixed << setprecision(2) << total << endl;
                return total;
            }
        }

        cout << "Reservation not found." << endl;
        return -1;
    }

    // ─── Display ─────────────────────────────────────────

    void display_fleet() const {
        lock_guard<mutex> lock(mtx);
        cout << "\n=== Vehicle Fleet ===" << endl;
        for (const auto& [id, vehicle] : vehicles) {
            cout << "  ";
            vehicle.print();
        }
        cout << "=====================\n" << endl;
    }
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    RentalSystem system;

    // Add vehicles
    system.add_vehicle(Vehicle("V1", "Toyota", "Camry", 2024,
                               VehicleType::SEDAN, "ABC-1234", "Downtown"));
    system.add_vehicle(Vehicle("V2", "Honda", "CR-V", 2024,
                               VehicleType::SUV, "DEF-5678", "Downtown"));
    system.add_vehicle(Vehicle("V3", "BMW", "7 Series", 2024,
                               VehicleType::LUXURY, "GHI-9012", "Airport"));
    system.add_vehicle(Vehicle("V4", "Hyundai", "i20", 2024,
                               VehicleType::HATCHBACK, "JKL-3456", "Downtown"));

    // Add customers
    system.add_customer(Customer("C1", "Alice", "alice@example.com", "DL-001"));
    system.add_customer(Customer("C2", "Bob", "bob@example.com", "DL-002"));

    system.display_fleet();

    // Search available SUVs
    cout << "--- Search: SUVs available Dec 20-25 ---" << endl;
    RentalDate start{2025, 12, 20};
    RentalDate end{2025, 12, 25};
    auto available = system.search_available(VehicleType::SUV, start, end, "Downtown");
    for (auto* v : available) {
        cout << "  ";
        v->print();
    }

    // Make reservation with add-ons
    cout << "\n--- Alice rents SUV ---" << endl;
    auto res1 = system.make_reservation(
        "C1", "V2", start, end, "Downtown", "Airport",
        AddOnBuilder().with_insurance().with_gps()
    );

    // Try to book same vehicle for overlapping dates (should fail)
    cout << "\n--- Bob tries same SUV ---" << endl;
    RentalDate bob_start{2025, 12, 23};
    RentalDate bob_end{2025, 12, 28};
    system.make_reservation("C2", "V2", bob_start, bob_end, "Downtown", "Downtown");

    // Bob books a different vehicle
    cout << "\n--- Bob rents Luxury ---" << endl;
    system.set_pricing(make_unique<WeeklyPricing>());
    auto res2 = system.make_reservation(
        "C2", "V3", bob_start, bob_end, "Airport", "Airport",
        AddOnBuilder().with_insurance().with_roadside_assistance()
    );

    // Complete reservation (on time)
    cout << "\n--- Alice returns vehicle ---" << endl;
    if (res1) system.complete_reservation(*res1, end);

    // Complete reservation (late)
    cout << "\n--- Bob returns late ---" << endl;
    RentalDate late_return{2025, 12, 30};
    if (res2) system.complete_reservation(*res2, late_return);

    return 0;
}
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `PricingStrategy` (Daily, Weekly) | Swap pricing models without changing reservation logic |
| **Builder** | `AddOnBuilder` | Fluent construction of optional add-on services |
| **Factory** | Reservation ID generation | Automatic unique ID creation |
| **Facade** | `RentalSystem` | Simple interface hiding complex vehicle, customer, reservation, and pricing interactions |

### Key Design Decisions

1. **Date-range availability** — availability is checked by scanning existing reservations for overlapping date ranges, preventing double-booking
2. **Builder for add-ons** — the fluent builder pattern makes it easy to compose optional services without constructor explosion
3. **Late return penalty** — late fees are calculated at 1.5x the normal daily rate, incentivizing on-time returns
4. **Pricing strategy** — daily and weekly pricing strategies allow the system to offer discounts for longer rentals. The strategy can be swapped at runtime
5. **Location tracking** — vehicles have a location, enabling location-based search and supporting one-way rentals (pick up at one location, return at another)

---


## 13.8 Hotel Booking System

> Design a hotel booking system that supports rooms of different types, reservations, guest management, check-in/check-out, amenities, and pricing. This problem tests your ability to model a hospitality domain with time-based room availability, guest lifecycle, and configurable pricing.

---

### Requirements

**Functional Requirements:**
1. The hotel has rooms of different types (Standard, Deluxe, Suite, Presidential)
2. Each room type has a base price, capacity, and set of amenities
3. Guests can search for available rooms by type, date range, and number of guests
4. Guests can make a reservation for one or more rooms
5. The system prevents double-booking of rooms
6. Guests can check in (room status changes to Occupied)
7. Guests can check out (room status changes to Available, bill is generated)
8. Pricing varies by room type and can include seasonal rates
9. Guests can request additional amenities (room service, laundry, minibar)
10. The system tracks guest history and generates invoices

**Non-Functional Requirements:**
- Thread-safe for concurrent booking requests
- Extensible for new room types, amenities, and pricing strategies
- Accurate date-based availability management

---

### Key Entities & Class Design

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│    Hotel     │──────▶│    Room      │──────▶│  RoomType    │
│  (Facade)    │       │              │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
       │                      │
       │                      │
       ▼                      ▼
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│    Guest     │◀─────▶│ Reservation  │──────▶│PricingStrategy│
│              │       │              │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
       │
       ▼
┌──────────────┐
│   Invoice    │
│  (charges)   │
└──────────────┘
```

---

### Implementation

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <memory>
#include <mutex>
#include <optional>
#include <algorithm>
#include <iomanip>
#include <numeric>
using namespace std;

// ─── Enums ───────────────────────────────────────────────

enum class RoomType { STANDARD, DELUXE, SUITE, PRESIDENTIAL };
enum class RoomStatus { AVAILABLE, RESERVED, OCCUPIED, MAINTENANCE };
enum class ReservationStatus { CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED };

string to_string(RoomType t) {
    switch (t) {
        case RoomType::STANDARD:      return "Standard";
        case RoomType::DELUXE:        return "Deluxe";
        case RoomType::SUITE:         return "Suite";
        case RoomType::PRESIDENTIAL:  return "Presidential";
    }
    return "Unknown";
}

string to_string(RoomStatus s) {
    switch (s) {
        case RoomStatus::AVAILABLE:   return "Available";
        case RoomStatus::RESERVED:    return "Reserved";
        case RoomStatus::OCCUPIED:    return "Occupied";
        case RoomStatus::MAINTENANCE: return "Maintenance";
    }
    return "Unknown";
}

string to_string(ReservationStatus s) {
    switch (s) {
        case ReservationStatus::CONFIRMED:   return "Confirmed";
        case ReservationStatus::CHECKED_IN:  return "Checked In";
        case ReservationStatus::CHECKED_OUT: return "Checked Out";
        case ReservationStatus::CANCELLED:   return "Cancelled";
    }
    return "Unknown";
}

// ─── Date (simplified) ──────────────────────────────────

struct BookingDate {
    int year, month, day;

    bool operator<(const BookingDate& o) const {
        if (year != o.year) return year < o.year;
        if (month != o.month) return month < o.month;
        return day < o.day;
    }

    bool operator<=(const BookingDate& o) const { return !(o < *this); }
    bool operator==(const BookingDate& o) const {
        return year == o.year && month == o.month && day == o.day;
    }

    int days_until(const BookingDate& o) const {
        return (o.year - year) * 365 + (o.month - month) * 30 + (o.day - day);
    }

    string to_string() const {
        return std::to_string(year) + "-"
               + (month < 10 ? "0" : "") + std::to_string(month) + "-"
               + (day < 10 ? "0" : "") + std::to_string(day);
    }

    static bool overlaps(const BookingDate& s1, const BookingDate& e1,
                         const BookingDate& s2, const BookingDate& e2) {
        return s1 < e2 && s2 < e1;
    }
};

// ─── Room ────────────────────────────────────────────────

class Room {
    string room_number;
    RoomType type;
    int floor;
    int capacity;
    vector<string> amenities;

public:
    Room(const string& number, RoomType type, int floor, int capacity,
         const vector<string>& amenities)
        : room_number(number), type(type), floor(floor),
          capacity(capacity), amenities(amenities) {}

    const string& get_number() const { return room_number; }
    RoomType get_type() const { return type; }
    int get_floor() const { return floor; }
    int get_capacity() const { return capacity; }
    const vector<string>& get_amenities() const { return amenities; }

    void print() const {
        cout << "Room " << room_number << " [" << to_string(type)
             << "] Floor " << floor << " | Capacity: " << capacity
             << " | Amenities: ";
        for (size_t i = 0; i < amenities.size(); i++) {
            cout << amenities[i];
            if (i < amenities.size() - 1) cout << ", ";
        }
        cout << endl;
    }
};

// ─── Guest ───────────────────────────────────────────────

class Guest {
    string guest_id;
    string name;
    string email;
    string phone;

public:
    Guest(const string& id, const string& name, const string& email,
          const string& phone)
        : guest_id(id), name(name), email(email), phone(phone) {}

    const string& get_id() const { return guest_id; }
    const string& get_name() const { return name; }
    const string& get_email() const { return email; }
};

// ─── Charge (for invoice) ────────────────────────────────

struct Charge {
    string description;
    double amount;

    Charge(const string& desc, double amt) : description(desc), amount(amt) {}
};

// ─── Pricing Strategy (Strategy Pattern) ─────────────────

class HotelPricingStrategy {
public:
    virtual ~HotelPricingStrategy() = default;
    virtual double get_nightly_rate(RoomType type) const = 0;
    virtual string name() const = 0;
};

class StandardPricing : public HotelPricingStrategy {
public:
    double get_nightly_rate(RoomType type) const override {
        switch (type) {
            case RoomType::STANDARD:     return 100.0;
            case RoomType::DELUXE:       return 180.0;
            case RoomType::SUITE:        return 350.0;
            case RoomType::PRESIDENTIAL: return 800.0;
        }
        return 0;
    }
    string name() const override { return "Standard"; }
};

class SeasonalPricing : public HotelPricingStrategy {
    double multiplier;
public:
    SeasonalPricing(double mult = 1.5) : multiplier(mult) {}

    double get_nightly_rate(RoomType type) const override {
        StandardPricing base;
        return base.get_nightly_rate(type) * multiplier;
    }
    string name() const override {
        return "Seasonal (" + std::to_string(multiplier) + "x)";
    }
};

// ─── Reservation ─────────────────────────────────────────

class HotelReservation {
    static int next_id;

    string reservation_id;
    string guest_id;
    vector<string> room_numbers;
    BookingDate check_in_date;
    BookingDate check_out_date;
    int num_guests;
    double room_charge;
    vector<Charge> additional_charges;
    ReservationStatus status;

public:
    HotelReservation(const string& guest_id, const vector<string>& rooms,
                     const BookingDate& check_in, const BookingDate& check_out,
                     int guests, double room_charge)
        : reservation_id("HRS" + std::to_string(next_id++)),
          guest_id(guest_id), room_numbers(rooms),
          check_in_date(check_in), check_out_date(check_out),
          num_guests(guests), room_charge(room_charge),
          status(ReservationStatus::CONFIRMED) {}

    const string& get_id() const { return reservation_id; }
    const string& get_guest_id() const { return guest_id; }
    const vector<string>& get_rooms() const { return room_numbers; }
    const BookingDate& get_check_in() const { return check_in_date; }
    const BookingDate& get_check_out() const { return check_out_date; }
    ReservationStatus get_status() const { return status; }
    double get_room_charge() const { return room_charge; }

    void set_status(ReservationStatus s) { status = s; }

    void add_charge(const string& desc, double amount) {
        additional_charges.emplace_back(desc, amount);
    }

    double get_total() const {
        double total = room_charge;
        for (const auto& c : additional_charges) total += c.amount;
        return total;
    }

    bool overlaps_with(const BookingDate& s, const BookingDate& e) const {
        return (status == ReservationStatus::CONFIRMED
                || status == ReservationStatus::CHECKED_IN)
               && BookingDate::overlaps(check_in_date, check_out_date, s, e);
    }

    void print_invoice() const {
        cout << "\n╔══════════════════════════════════════╗" << endl;
        cout << "║          HOTEL INVOICE               ║" << endl;
        cout << "╠══════════════════════════════════════╣" << endl;
        cout << "  Reservation: " << reservation_id << endl;
        cout << "  Guest: " << guest_id << endl;
        cout << "  Rooms: ";
        for (const auto& r : room_numbers) cout << r << " ";
        cout << endl;
        cout << "  Check-in:  " << check_in_date.to_string() << endl;
        cout << "  Check-out: " << check_out_date.to_string() << endl;
        cout << "  Nights: " << check_in_date.days_until(check_out_date) << endl;
        cout << "╠══════════════════════════════════════╣" << endl;
        cout << "  Room charges: $" << fixed << setprecision(2)
             << room_charge << endl;
        if (!additional_charges.empty()) {
            cout << "  Additional charges:" << endl;
            for (const auto& c : additional_charges) {
                cout << "    " << c.description << ": $"
                     << fixed << setprecision(2) << c.amount << endl;
            }
        }
        cout << "╠══════════════════════════════════════╣" << endl;
        cout << "  TOTAL: $" << fixed << setprecision(2) << get_total() << endl;
        cout << "╚══════════════════════════════════════╝\n" << endl;
    }
};

int HotelReservation::next_id = 1;

// ─── Hotel (Facade) ─────────────────────────────────────

class Hotel {
    string name;
    unordered_map<string, Room> rooms;
    unordered_map<string, Guest> guests;
    vector<HotelReservation> reservations;
    unique_ptr<HotelPricingStrategy> pricing;
    mutable mutex mtx;

public:
    Hotel(const string& name)
        : name(name), pricing(make_unique<StandardPricing>()) {}

    void set_pricing(unique_ptr<HotelPricingStrategy> strategy) {
        lock_guard<mutex> lock(mtx);
        pricing = std::move(strategy);
        cout << "Pricing set to: " << pricing->name() << endl;
    }

    void add_room(const Room& room) {
        lock_guard<mutex> lock(mtx);
        rooms.emplace(room.get_number(), room);
    }

    void add_guest(const Guest& guest) {
        lock_guard<mutex> lock(mtx);
        guests.emplace(guest.get_id(), guest);
    }

    // ─── Search Available Rooms ──────────────────────────

    vector<const Room*> search_available(
        RoomType type, const BookingDate& check_in,
        const BookingDate& check_out, int num_guests = 1) const {

        lock_guard<mutex> lock(mtx);
        vector<const Room*> results;

        for (const auto& [number, room] : rooms) {
            if (room.get_type() != type) continue;
            if (room.get_capacity() < num_guests) continue;

            bool available = true;
            for (const auto& res : reservations) {
                for (const auto& rn : res.get_rooms()) {
                    if (rn == number && res.overlaps_with(check_in, check_out)) {
                        available = false;
                        break;
                    }
                }
                if (!available) break;
            }

            if (available) results.push_back(&room);
        }

        return results;
    }

    // ─── Make Reservation ────────────────────────────────

    optional<string> make_reservation(
        const string& guest_id, const vector<string>& room_numbers,
        const BookingDate& check_in, const BookingDate& check_out,
        int num_guests) {

        lock_guard<mutex> lock(mtx);

        if (guests.find(guest_id) == guests.end()) {
            cout << "Guest not found." << endl;
            return nullopt;
        }

        int nights = check_in.days_until(check_out);
        if (nights <= 0) {
            cout << "Invalid date range." << endl;
            return nullopt;
        }

        // Validate all rooms are available
        for (const auto& rn : room_numbers) {
            auto room_it = rooms.find(rn);
            if (room_it == rooms.end()) {
                cout << "Room " << rn << " not found." << endl;
                return nullopt;
            }

            for (const auto& res : reservations) {
                for (const auto& res_rn : res.get_rooms()) {
                    if (res_rn == rn && res.overlaps_with(check_in, check_out)) {
                        cout << "❌ Room " << rn << " is not available for the requested dates." << endl;
                        return nullopt;
                    }
                }
            }
        }

        // Calculate room charge
        double total_room_charge = 0;
        for (const auto& rn : room_numbers) {
            const Room& room = rooms.at(rn);
            total_room_charge += pricing->get_nightly_rate(room.get_type()) * nights;
        }

        HotelReservation reservation(guest_id, room_numbers, check_in, check_out,
                                     num_guests, total_room_charge);
        string res_id = reservation.get_id();

        cout << "✅ Reservation " << res_id << " confirmed!" << endl;
        cout << "  Guest: " << guests.at(guest_id).get_name() << endl;
        cout << "  Rooms: ";
        for (const auto& rn : room_numbers) {
            cout << rn << " [" << to_string(rooms.at(rn).get_type()) << "] ";
        }
        cout << endl;
        cout << "  " << check_in.to_string() << " to " << check_out.to_string()
             << " (" << nights << " nights)" << endl;
        cout << "  Room charge: $" << fixed << setprecision(2)
             << total_room_charge << endl;

        reservations.push_back(reservation);
        return res_id;
    }

    // ─── Check In ────────────────────────────────────────

    bool check_in(const string& reservation_id) {
        lock_guard<mutex> lock(mtx);

        for (auto& res : reservations) {
            if (res.get_id() == reservation_id) {
                if (res.get_status() != ReservationStatus::CONFIRMED) {
                    cout << "Cannot check in: reservation is "
                         << to_string(res.get_status()) << endl;
                    return false;
                }
                res.set_status(ReservationStatus::CHECKED_IN);
                cout << "🔑 Checked in! Reservation " << reservation_id << endl;
                cout << "  Rooms: ";
                for (const auto& rn : res.get_rooms()) cout << rn << " ";
                cout << endl;
                return true;
            }
        }

        cout << "Reservation not found." << endl;
        return false;
    }

    // ─── Add Service Charge ──────────────────────────────

    bool add_service_charge(const string& reservation_id,
                            const string& service, double amount) {
        lock_guard<mutex> lock(mtx);

        for (auto& res : reservations) {
            if (res.get_id() == reservation_id) {
                if (res.get_status() != ReservationStatus::CHECKED_IN) {
                    cout << "Can only add charges to checked-in reservations." << endl;
                    return false;
                }
                res.add_charge(service, amount);
                cout << "  🛎️ Added charge: " << service << " ($"
                     << fixed << setprecision(2) << amount << ")" << endl;
                return true;
            }
        }
        return false;
    }

    // ─── Check Out ───────────────────────────────────────

    double check_out(const string& reservation_id) {
        lock_guard<mutex> lock(mtx);

        for (auto& res : reservations) {
            if (res.get_id() == reservation_id) {
                if (res.get_status() != ReservationStatus::CHECKED_IN) {
                    cout << "Cannot check out: reservation is "
                         << to_string(res.get_status()) << endl;
                    return -1;
                }

                res.set_status(ReservationStatus::CHECKED_OUT);
                cout << "👋 Checked out! Reservation " << reservation_id << endl;
                res.print_invoice();
                return res.get_total();
            }
        }

        cout << "Reservation not found." << endl;
        return -1;
    }

    // ─── Cancel Reservation ──────────────────────────────

    bool cancel_reservation(const string& reservation_id) {
        lock_guard<mutex> lock(mtx);

        for (auto& res : reservations) {
            if (res.get_id() == reservation_id) {
                if (res.get_status() != ReservationStatus::CONFIRMED) {
                    cout << "Can only cancel confirmed reservations." << endl;
                    return false;
                }
                res.set_status(ReservationStatus::CANCELLED);
                cout << "🚫 Reservation " << reservation_id << " cancelled." << endl;
                return true;
            }
        }
        return false;
    }

    // ─── Display ─────────────────────────────────────────

    void display_rooms() const {
        lock_guard<mutex> lock(mtx);
        cout << "\n=== " << name << " — Rooms ===" << endl;
        for (const auto& [number, room] : rooms) {
            cout << "  ";
            room.print();
        }
        cout << "==============================\n" << endl;
    }
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    Hotel hotel("Grand Palace Hotel");

    // Add rooms
    hotel.add_room(Room("101", RoomType::STANDARD, 1, 2,
                        {"WiFi", "TV", "AC"}));
    hotel.add_room(Room("102", RoomType::STANDARD, 1, 2,
                        {"WiFi", "TV", "AC"}));
    hotel.add_room(Room("201", RoomType::DELUXE, 2, 3,
                        {"WiFi", "TV", "AC", "Minibar", "Balcony"}));
    hotel.add_room(Room("301", RoomType::SUITE, 3, 4,
                        {"WiFi", "TV", "AC", "Minibar", "Balcony", "Jacuzzi", "Living Room"}));
    hotel.add_room(Room("401", RoomType::PRESIDENTIAL, 4, 6,
                        {"WiFi", "TV", "AC", "Minibar", "Balcony", "Jacuzzi",
                         "Living Room", "Dining Room", "Butler Service"}));

    // Add guests
    hotel.add_guest(Guest("G1", "Alice Johnson", "alice@example.com", "+1-555-0101"));
    hotel.add_guest(Guest("G2", "Bob Smith", "bob@example.com", "+1-555-0202"));

    hotel.display_rooms();

    // Search available suites
    BookingDate check_in{2025, 7, 15};
    BookingDate check_out{2025, 7, 18};

    cout << "--- Search: Deluxe rooms, July 15-18 ---" << endl;
    auto available = hotel.search_available(RoomType::DELUXE, check_in, check_out, 2);
    for (auto* room : available) {
        cout << "  ";
        room->print();
    }

    // Alice books a Deluxe room
    cout << "\n--- Alice books Deluxe ---" << endl;
    auto res1 = hotel.make_reservation("G1", {"201"}, check_in, check_out, 2);

    // Bob tries same room (should fail)
    cout << "\n--- Bob tries same room ---" << endl;
    hotel.make_reservation("G2", {"201"}, {2025, 7, 16}, {2025, 7, 20}, 2);

    // Bob books Standard room
    cout << "\n--- Bob books Standard ---" << endl;
    auto res2 = hotel.make_reservation("G2", {"101"}, {2025, 7, 16}, {2025, 7, 20}, 2);

    // Alice checks in
    cout << "\n--- Alice checks in ---" << endl;
    if (res1) hotel.check_in(*res1);

    // Alice orders room service
    cout << "\n--- Alice orders services ---" << endl;
    if (res1) {
        hotel.add_service_charge(*res1, "Room Service - Dinner", 45.00);
        hotel.add_service_charge(*res1, "Laundry", 25.00);
        hotel.add_service_charge(*res1, "Minibar", 30.00);
    }

    // Alice checks out
    cout << "\n--- Alice checks out ---" << endl;
    if (res1) hotel.check_out(*res1);

    // Switch to seasonal pricing
    cout << "\n--- Switch to seasonal pricing ---" << endl;
    hotel.set_pricing(make_unique<SeasonalPricing>(1.5));

    // New booking with seasonal pricing
    auto res3 = hotel.make_reservation("G1", {"301"},
                                       {2025, 12, 24}, {2025, 12, 27}, 3);

    return 0;
}
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `HotelPricingStrategy` (Standard, Seasonal) | Swap pricing models for different seasons/events without changing booking logic |
| **Observer** | Console notifications on check-in/check-out events | Extensible for email confirmations, SMS alerts |
| **Factory** | Reservation ID generation | Automatic unique ID creation |
| **Facade** | `Hotel` class | Simple interface hiding complex room, guest, reservation, and pricing interactions |

### Key Design Decisions

1. **Room vs Reservation separation** — `Room` defines the physical room (number, type, amenities); `HotelReservation` tracks the booking (dates, guest, charges). A room can have multiple non-overlapping reservations
2. **Date-range overlap checking** — reservations are checked for date overlap to prevent double-booking. Only CONFIRMED and CHECKED_IN reservations block availability
3. **Incremental charges** — additional services (room service, laundry, minibar) are added as charges during the stay and included in the final invoice at checkout
4. **Seasonal pricing** — the pricing strategy can be swapped at runtime, allowing the hotel to adjust rates for peak seasons, holidays, or special events
5. **Multi-room booking** — a single reservation can include multiple rooms, supporting group bookings and families needing adjacent rooms

---


## Summary & Key Takeaways

| Problem | Core Challenge | Key Patterns | Difficulty Focus |
|---------|---------------|-------------|-----------------|
| Library Management | Multi-entity relationships, reservation queues | Singleton, Strategy, Observer, Facade | Entity modeling, notification system |
| Elevator System | Scheduling algorithms, concurrent requests | Strategy, State, Observer | Algorithm selection, state management |
| Snake and Ladder | Game mechanics, configurable board | Strategy, Observer | Turn-based logic, event broadcasting |
| Splitwise | Financial calculations, debt simplification | Strategy, Facade | Split algorithms, balance tracking |
| Movie Ticket Booking | Concurrent seat locking, two-phase booking | Singleton, Strategy, State | Concurrency, atomic operations |
| Online Shopping | End-to-end e-commerce workflow | Strategy, Facade, Decorator | Complex workflow, inventory management |
| Car Rental System | Date-based availability, add-on services | Strategy, Builder, Facade | Time-range queries, fluent configuration |
| Hotel Booking | Room lifecycle, incremental charges | Strategy, Observer, Facade | Check-in/out flow, invoice generation |

---

## Interview Tips for Module 13

1. **"How do you approach a medium-difficulty LLD problem?"** Start by clarifying requirements (functional and non-functional). Identify the key entities and their relationships. Draw a class diagram. Choose appropriate design patterns. Implement incrementally — start with the core entities, then add business logic, then handle edge cases. Always consider thread safety and extensibility.

2. **"How do you handle concurrent booking/reservation?"** Use a two-phase approach: (a) **Lock** — temporarily reserve the resource (seat, room, vehicle) with a timeout. (b) **Confirm** — permanently book after payment succeeds. If payment fails or times out, release the lock. Use mutexes to protect shared state. Ensure atomic operations — either all seats are locked or none are (all-or-nothing).

3. **"How do you choose which design patterns to use?"** Match the pattern to the problem: **Strategy** when you have multiple interchangeable algorithms (pricing, scheduling, splitting). **Observer** when one event should notify multiple listeners (book available, order status change). **State** when behavior changes based on internal state (elevator direction, vending machine). **Builder** when constructing complex objects with many optional parts (add-ons, configurations). **Facade** when you need a simple interface over a complex subsystem.

4. **"How do you handle pricing that changes?"** Use the Strategy pattern. Define a `PricingStrategy` interface with a `calculate()` method. Implement concrete strategies (daily, weekly, seasonal, promotional). The system holds a reference to the current strategy and can swap it at runtime. This follows the Open/Closed Principle — new pricing models are added without modifying existing code.

5. **"How do you prevent double-booking?"** For time-based resources (rooms, vehicles), check for date-range overlap before confirming a reservation. Two ranges `[s1, e1)` and `[s2, e2)` overlap if `s1 < e2 AND s2 < e1`. For seat-based resources (movie tickets), use a lock-then-book approach with expiring locks. Always use mutexes to ensure atomicity of the check-and-reserve operation.

6. **"How do you simplify debts in a Splitwise-like system?"** Calculate the net balance for each user (total owed minus total owed to them). Separate users into debtors (positive net) and creditors (negative net). Use a greedy algorithm to match the largest debtor with the largest creditor, settling the minimum of their balances. This minimizes the number of transactions needed.

7. **"How do you design an elevator scheduling algorithm?"** The LOOK algorithm is the most common: the elevator continues in its current direction, stopping at requested floors, until there are no more requests in that direction, then reverses. Maintain two sorted sets of stops (up and down). For multi-elevator systems, assign requests to the elevator with the shortest estimated travel distance, considering current direction and pending stops.

8. **"How do you handle inventory in an e-commerce system?"** Decrement inventory when an order is confirmed (not when added to cart). If an order is cancelled, restore the inventory. Validate stock for all items before processing any — if one item is out of stock, reject the entire order. Use mutexes to prevent race conditions where two users order the last item simultaneously.

9. **"How do you design a notification system?"** Use the Observer pattern. Define an observer interface with methods for each event type. Components that generate events (library, booking system) maintain a list of observers and notify them when events occur. This decouples the event source from the notification mechanism — you can add email, SMS, push, or webhook observers without changing the source.

10. **"What's the difference between easy and medium LLD problems?"** Easy problems (parking lot, tic-tac-toe, vending machine) typically have a single main entity with straightforward state transitions. Medium problems (booking systems, e-commerce, expense sharing) involve multiple interacting entities, complex business rules, time-based logic, concurrent access, and require combining 3-5 design patterns. The key skill at the medium level is managing complexity through clean separation of concerns.

11. **"How do you handle search functionality in LLD?"** For in-memory systems, iterate over the collection and filter by criteria. Use case-insensitive string matching for text search. For extensibility, consider the Strategy pattern for different search criteria. In a real system, you'd use database indexes or a search engine (Elasticsearch), but in an LLD interview, the focus is on the interface design and how search integrates with the rest of the system.

12. **"How do you generate invoices or bills?"** Track all charges incrementally during the lifecycle (room charges, service charges, late fees, discounts). Store charges as a list of `(description, amount)` pairs. At checkout/completion, sum all charges to produce the total. This approach is extensible — new charge types can be added without modifying the invoice logic. Use the Decorator pattern if discounts or taxes need to be applied on top of the base charges.
