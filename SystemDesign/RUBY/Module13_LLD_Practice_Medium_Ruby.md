# Module 13: LLD Practice Problems (Medium) — Ruby

> This module raises the bar from Module 12 with more complex, real-world systems. Each problem involves multiple interacting entities, non-trivial business rules, concurrency considerations, and requires combining several design patterns. These are the problems most commonly asked in LLD interviews at mid-level companies (and are the bread-and-butter of FAANG LLD rounds). Every problem follows the same structured approach: requirements, key entities, class design, full Ruby implementation, design patterns used, and key design decisions.

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

```ruby
require 'monitor'

# ─── Enums ───────────────────────────────────────────────

module BookCopyStatus
  AVAILABLE = :available
  ISSUED    = :issued
  RESERVED  = :reserved
  LOST      = :lost
end

module AccountStatus
  ACTIVE    = :active
  SUSPENDED = :suspended
  CLOSED    = :closed
end

# ─── Fine Strategy (Strategy Pattern) ────────────────────

class FineStrategy
  def calculate(_overdue_days)
    raise NotImplementedError
  end

  def name
    raise NotImplementedError
  end
end

class FlatFineStrategy < FineStrategy
  def initialize(rate_per_day = 1.0)
    @rate_per_day = rate_per_day
  end

  def calculate(overdue_days)
    return 0.0 if overdue_days <= 0
    overdue_days * @rate_per_day
  end

  def name
    "Flat ($#{'%.2f' % @rate_per_day}/day)"
  end
end

class TieredFineStrategy < FineStrategy
  def calculate(overdue_days)
    return 0.0 if overdue_days <= 0
    # First 7 days: $1/day, after that: $2/day
    if overdue_days <= 7
      overdue_days * 1.0
    else
      7.0 + (overdue_days - 7) * 2.0
    end
  end

  def name
    "Tiered ($1/day first week, $2/day after)"
  end
end

# ─── Book (catalog entry) ────────────────────────────────

class Book
  attr_reader :isbn, :title, :author, :publisher, :year

  def initialize(isbn, title, author, publisher, year)
    @isbn = isbn
    @title = title
    @author = author
    @publisher = publisher
    @year = year
  end

  def to_s
    "\"#{@title}\" by #{@author} (ISBN: #{@isbn}, #{@year})"
  end
end

# ─── BookCopy (physical copy) ────────────────────────────

class BookCopy
  attr_reader :barcode, :isbn
  attr_accessor :status

  def initialize(barcode, isbn)
    @barcode = barcode
    @isbn = isbn
    @status = BookCopyStatus::AVAILABLE
  end

  def available?
    @status == BookCopyStatus::AVAILABLE
  end
end

# ─── Loan (tracks an issued book) ────────────────────────

class Loan
  attr_reader :barcode, :member_id, :issue_date, :due_date, :return_date

  def initialize(barcode, member_id, issue_date, due_date)
    @barcode = barcode
    @member_id = member_id
    @issue_date = issue_date
    @due_date = due_date
    @return_date = nil
  end

  def returned?
    !@return_date.nil?
  end

  def mark_returned
    @return_date = Time.now
  end

  def overdue_days
    end_time = @return_date || Time.now
    diff = ((end_time - @due_date) / 86_400).to_i  # seconds → days
    [0, diff].max
  end
end

# ─── Reservation ─────────────────────────────────────────

class Reservation
  attr_reader :isbn, :member_id, :reserved_date

  def initialize(isbn, member_id)
    @isbn = isbn
    @member_id = member_id
    @reserved_date = Time.now
  end
end

# ─── Observer Interface (for notifications) ──────────────

class LibraryObserver
  def on_book_available(_isbn, _member_id)
    raise NotImplementedError
  end
end

class ConsoleNotifier < LibraryObserver
  def on_book_available(isbn, member_id)
    puts "📬 NOTIFICATION: Member #{member_id}" \
         " — your reserved book (ISBN: #{isbn}) is now available!"
  end
end

# ─── Member ──────────────────────────────────────────────

class Member
  MAX_BOOKS = 5

  attr_reader :member_id, :name, :email, :status, :total_fines

  def initialize(id, name, email)
    @member_id = id
    @name = name
    @email = email
    @status = AccountStatus::ACTIVE
    @active_barcodes = Set.new
    @total_fines = 0.0
  end

  def books_held
    @active_barcodes.size
  end

  def can_issue?
    @status == AccountStatus::ACTIVE && @active_barcodes.size < MAX_BOOKS
  end

  def add_book(barcode)
    @active_barcodes.add(barcode)
  end

  def remove_book(barcode)
    @active_barcodes.delete(barcode)
  end

  def has_book?(barcode)
    @active_barcodes.include?(barcode)
  end

  def add_fine(amount)
    @total_fines += amount
  end

  def set_status(status)
    @status = status
  end

  def to_s
    "Member: #{@name} (ID: #{@member_id})" \
      " | Books held: #{@active_barcodes.size}/#{MAX_BOOKS}" \
      " | Fines: $#{'%.2f' % @total_fines}"
  end
end

# ─── Library (Singleton, Facade) ─────────────────────────

class Library
  include MonitorMixin

  @instance = nil
  @instance_mutex = Mutex.new

  def self.instance
    @instance_mutex.synchronize do
      @instance ||= new
    end
  end

  private_class_method :new

  def initialize
    super  # initializes MonitorMixin
    @name = ""
    @catalog = {}            # isbn => Book
    @copies = {}             # barcode => BookCopy
    @isbn_to_barcodes = {}   # isbn => [barcodes]
    @members = {}            # member_id => Member
    @active_loans = {}       # barcode => Loan
    @loan_history = []
    @reservations = {}       # isbn => Queue of member_ids
    @fine_strategy = FlatFineStrategy.new(1.0)
    @observers = []
  end

  def set_name(name)
    @name = name
  end

  def add_observer(observer)
    @observers << observer
  end

  def set_fine_strategy(strategy)
    synchronize { @fine_strategy = strategy }
  end

  # ─── Catalog Management ────────────────────────────────

  def add_book(book, num_copies)
    synchronize do
      isbn = book.isbn
      @catalog[isbn] = book
      @isbn_to_barcodes[isbn] ||= []

      num_copies.times do
        barcode = "#{isbn}-#{@isbn_to_barcodes[isbn].size + 1}"
        @copies[barcode] = BookCopy.new(barcode, isbn)
        @isbn_to_barcodes[isbn] << barcode
      end

      puts "Added #{num_copies} copies of \"#{book.title}\" (ISBN: #{isbn})"
    end
  end

  def add_member(member)
    synchronize do
      @members[member.member_id] = member
      puts "Registered member: #{member.name} (ID: #{member.member_id})"
    end
  end

  # ─── Search ────────────────────────────────────────────

  def search_by_title(query)
    synchronize do
      lower_query = query.downcase
      @catalog.values.select { |book| book.title.downcase.include?(lower_query) }
    end
  end

  def search_by_author(query)
    synchronize do
      lower_query = query.downcase
      @catalog.values.select { |book| book.author.downcase.include?(lower_query) }
    end
  end

  def search_by_isbn(isbn)
    synchronize { @catalog[isbn] }
  end

  # ─── Issue Book ────────────────────────────────────────

  def issue_book(member_id, isbn)
    synchronize do
      # Validate member
      member = @members[member_id]
      unless member
        puts "Error: Member #{member_id} not found."
        return false
      end

      unless member.can_issue?
        puts "Error: Member #{member.name}" \
             " cannot issue more books (limit reached or account suspended)."
        return false
      end

      # Find an available copy
      barcodes = @isbn_to_barcodes[isbn]
      unless barcodes
        puts "Error: Book with ISBN #{isbn} not in catalog."
        return false
      end

      available_copy = barcodes
        .map { |bc| @copies[bc] }
        .find(&:available?)

      unless available_copy
        puts "No available copies of ISBN #{isbn}. Consider reserving."
        return false
      end

      # Issue the book
      available_copy.status = BookCopyStatus::ISSUED
      member.add_book(available_copy.barcode)

      loan = Loan.new(
        available_copy.barcode, member_id,
        Time.now, Time.now + (14 * 86_400)  # 14 days
      )
      @active_loans[available_copy.barcode] = loan

      book = @catalog[isbn]
      puts "✅ Issued \"#{book.title}\" (barcode: #{available_copy.barcode})" \
           " to #{member.name}. Due in 14 days."
      true
    end
  end

  # ─── Return Book ───────────────────────────────────────

  def return_book(member_id, barcode)
    synchronize do
      # Validate member
      member = @members[member_id]
      unless member
        puts "Error: Member #{member_id} not found."
        return -1
      end

      unless member.has_book?(barcode)
        puts "Error: Member #{member.name} does not hold book #{barcode}"
        return -1
      end

      # Find the loan
      loan = @active_loans[barcode]
      unless loan
        puts "Error: No active loan for barcode #{barcode}"
        return -1
      end

      loan.mark_returned

      # Calculate fine
      overdue = loan.overdue_days
      fine = @fine_strategy.calculate(overdue)

      # Update copy status
      copy = @copies[barcode]
      copy.status = BookCopyStatus::AVAILABLE

      # Update member
      member.remove_book(barcode)
      member.add_fine(fine) if fine > 0

      # Move loan to history
      @loan_history << loan
      @active_loans.delete(barcode)

      book = @catalog[copy.isbn]
      msg = "✅ Returned \"#{book.title}\" (barcode: #{barcode}) by #{member.name}"
      msg += " | Overdue by #{overdue} days | Fine: $#{'%.2f' % fine}" if overdue > 0
      puts msg

      # Check reservations
      check_reservations(copy.isbn)

      fine
    end
  end

  # ─── Reserve Book ──────────────────────────────────────

  def reserve_book(member_id, isbn)
    synchronize do
      member = @members[member_id]
      unless member
        puts "Error: Member #{member_id} not found."
        return false
      end

      unless @catalog.key?(isbn)
        puts "Error: Book with ISBN #{isbn} not in catalog."
        return false
      end

      # Check if any copy is available (no need to reserve)
      has_available = @isbn_to_barcodes[isbn].any? { |bc| @copies[bc].available? }

      if has_available
        puts "Copies are available — no need to reserve. Use issue_book() instead."
        return false
      end

      @reservations[isbn] ||= []
      @reservations[isbn] << member_id

      book = @catalog[isbn]
      puts "📋 Reserved \"#{book.title}\" for member #{member.name}." \
           " Position in queue: #{@reservations[isbn].size}"
      true
    end
  end

  # ─── Display ───────────────────────────────────────────

  def display_catalog
    synchronize do
      puts "\n=== #{@name} Catalog ==="
      @catalog.each do |isbn, book|
        total = @isbn_to_barcodes[isbn].size
        available = @isbn_to_barcodes[isbn].count { |bc| @copies[bc].available? }
        puts "  #{book}"
        puts "    Copies: #{available}/#{total} available"
      end
      puts "==============================\n"
    end
  end

  def display_member(member_id)
    synchronize do
      member = @members[member_id]
      puts member if member
    end
  end

  private

  def notify_book_available(isbn, member_id)
    @observers.each { |obs| obs.on_book_available(isbn, member_id) }
  end

  def check_reservations(isbn)
    queue = @reservations[isbn]
    return if queue.nil? || queue.empty?

    next_member = queue.shift
    notify_book_available(isbn, next_member)
  end
end

# ─── Usage ───────────────────────────────────────────────

lib = Library.instance
lib.set_name("City Central Library")
lib.add_observer(ConsoleNotifier.new)

# Add books
lib.add_book(Book.new("978-0-13-468599-1", "The C++ Programming Language",
                      "Bjarne Stroustrup", "Addison-Wesley", 2013), 3)
lib.add_book(Book.new("978-0-201-63361-0", "Design Patterns",
                      "Gang of Four", "Addison-Wesley", 1994), 2)
lib.add_book(Book.new("978-0-13-235088-4", "Clean Code",
                      "Robert C. Martin", "Prentice Hall", 2008), 1)

# Add members
lib.add_member(Member.new("M001", "Alice", "alice@example.com"))
lib.add_member(Member.new("M002", "Bob", "bob@example.com"))
lib.add_member(Member.new("M003", "Charlie", "charlie@example.com"))

lib.display_catalog

# Search
puts "--- Search by author: 'Stroustrup' ---"
results = lib.search_by_author("Stroustrup")
results.each { |book| puts book }

# Issue books
puts "\n--- Issue books ---"
lib.issue_book("M001", "978-0-13-235088-4")  # Alice gets Clean Code (only 1 copy)
lib.issue_book("M002", "978-0-13-235088-4")  # Bob tries — no copies available

# Bob reserves
puts "\n--- Reserve ---"
lib.reserve_book("M002", "978-0-13-235088-4")

# Alice returns — Bob gets notified
puts "\n--- Return ---"
lib.return_book("M001", "978-0-13-235088-4-1")

lib.display_catalog
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Singleton** | `Library` | One library instance, centralized management |
| **Strategy** | `FineStrategy` | Swap fine calculation algorithms (flat, tiered) without changing library logic |
| **Observer** | `LibraryObserver`, `ConsoleNotifier` | Notify members when reserved books become available |
| **Factory** | Barcode generation in `add_book` | Automatically creates unique barcodes for each copy |
| **Facade** | `Library` class | Provides a simple interface (`issue_book`, `return_book`) hiding complex internal interactions |

### Key Design Decisions

1. **Book vs BookCopy separation** — a `Book` is a catalog entry (ISBN, title, author); a `BookCopy` is a physical copy (barcode, status). Multiple copies share the same `Book` metadata
2. **Reservation queue** — array used as FIFO queue per ISBN ensures fairness; first to reserve is first to be notified
3. **Fine strategy** — pluggable fine calculation allows the library to change policies without modifying core logic
4. **Thread safety** — `MonitorMixin` with `synchronize` protects all shared state for concurrent issue/return operations. Ruby's `MonitorMixin` provides reentrant mutual exclusion, similar to C++ `std::mutex` with `lock_guard`
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

```ruby
require 'set'
require 'monitor'

# ─── Enums ───────────────────────────────────────────────

module Direction
  UP   = :up
  DOWN = :down
  IDLE = :idle
end

module DoorState
  OPEN   = :open
  CLOSED = :closed
end

# ─── Request ─────────────────────────────────────────────

ExternalRequest = Struct.new(:floor, :direction)
InternalRequest = Struct.new(:destination_floor, :elevator_id)

# ─── Elevator ────────────────────────────────────────────

class Elevator
  attr_reader :id, :current_floor, :direction, :door, :capacity, :current_load

  def initialize(id, min_floor, max_floor, capacity = 10)
    @id = id
    @current_floor = min_floor
    @direction = Direction::IDLE
    @door = DoorState::CLOSED
    @capacity = capacity
    @current_load = 0
    @min_floor = min_floor
    @max_floor = max_floor

    @up_stops = SortedSet.new    # floors to stop at while going up
    @down_stops = SortedSet.new  # floors to stop at while going down
  end

  def idle?
    @direction == Direction::IDLE
  end

  def has_space?
    @current_load < @capacity
  end

  def total_stops
    @up_stops.size + @down_stops.size
  end

  # Add a destination floor
  def add_stop(floor)
    if floor > @current_floor ||
       (floor == @current_floor && @direction == Direction::UP)
      @up_stops.add(floor)
    elsif floor < @current_floor ||
          (floor == @current_floor && @direction == Direction::DOWN)
      @down_stops.add(floor)
    else
      # Same floor and idle — just open doors
      if @direction == Direction::UP || @direction == Direction::IDLE
        @up_stops.add(floor)
      else
        @down_stops.add(floor)
      end
    end

    # Set direction if idle
    if @direction == Direction::IDLE
      if floor > @current_floor
        @direction = Direction::UP
      elsif floor < @current_floor
        @direction = Direction::DOWN
      end
    end
  end

  # Calculate distance to a floor (for scheduling)
  def distance_to(floor, requested_dir)
    return (@current_floor - floor).abs if @direction == Direction::IDLE

    # Moving in the same direction as the request
    if @direction == Direction::UP && requested_dir == Direction::UP &&
       floor >= @current_floor
      return floor - @current_floor
    end
    if @direction == Direction::DOWN && requested_dir == Direction::DOWN &&
       floor <= @current_floor
      return @current_floor - floor
    end

    # Moving in opposite direction or past the floor — need to reverse
    if @direction == Direction::UP
      top = @up_stops.empty? ? @current_floor : @up_stops.max
      (top - @current_floor) + (top - floor)
    else
      bottom = @down_stops.empty? ? @current_floor : @down_stops.min
      (@current_floor - bottom) + (floor - bottom)
    end
  end

  # Move one step (simulate one time unit)
  def step
    return if @direction == Direction::IDLE

    # Check if current floor is a stop
    if @direction == Direction::UP && @up_stops.include?(@current_floor)
      open_doors
      @up_stops.delete(@current_floor)
      close_doors
    elsif @direction == Direction::DOWN && @down_stops.include?(@current_floor)
      open_doors
      @down_stops.delete(@current_floor)
      close_doors
    end

    # Move
    if @direction == Direction::UP
      if !@up_stops.empty?
        @current_floor += 1
        puts "  Elevator #{@id} ▲ Floor #{@current_floor}"
      elsif !@down_stops.empty?
        @direction = Direction::DOWN
        @current_floor -= 1
        puts "  Elevator #{@id} ▼ Floor #{@current_floor}"
      else
        @direction = Direction::IDLE
        puts "  Elevator #{@id} ■ Idle at Floor #{@current_floor}"
      end
    elsif @direction == Direction::DOWN
      if !@down_stops.empty?
        @current_floor -= 1
        puts "  Elevator #{@id} ▼ Floor #{@current_floor}"
      elsif !@up_stops.empty?
        @direction = Direction::UP
        @current_floor += 1
        puts "  Elevator #{@id} ▲ Floor #{@current_floor}"
      else
        @direction = Direction::IDLE
        puts "  Elevator #{@id} ■ Idle at Floor #{@current_floor}"
      end
    end

    # Check if arrived at a stop after moving
    if @direction == Direction::UP && @up_stops.include?(@current_floor)
      open_doors
      @up_stops.delete(@current_floor)
      close_doors
      @direction = Direction::IDLE if @up_stops.empty? && @down_stops.empty?
    elsif @direction == Direction::DOWN && @down_stops.include?(@current_floor)
      open_doors
      @down_stops.delete(@current_floor)
      close_doors
      @direction = Direction::IDLE if @up_stops.empty? && @down_stops.empty?
    end
  end

  def print_status
    puts "Elevator #{@id}: Floor #{@current_floor}" \
         " | Dir: #{@direction}" \
         " | Load: #{@current_load}/#{@capacity}" \
         " | Up stops: #{@up_stops.size}" \
         " | Down stops: #{@down_stops.size}"
  end

  private

  def open_doors
    @door = DoorState::OPEN
    puts "  Elevator #{@id} 🔔 Doors OPEN at Floor #{@current_floor}"
  end

  def close_doors
    @door = DoorState::CLOSED
    puts "  Elevator #{@id} Doors CLOSED at Floor #{@current_floor}"
  end
end

# ─── Scheduling Strategy (Strategy Pattern) ──────────────

class SchedulingStrategy
  def select_elevator(_elevators, _request)
    raise NotImplementedError
  end

  def name
    raise NotImplementedError
  end
end

# LOOK algorithm: assign to the nearest elevator moving in the same direction
# or the nearest idle elevator
class LOOKStrategy < SchedulingStrategy
  def select_elevator(elevators, request)
    best = nil
    best_distance = Float::INFINITY

    elevators.each do |elev|
      next unless elev.has_space?

      dist = elev.distance_to(request.floor, request.direction)
      if dist < best_distance
        best_distance = dist
        best = elev
      end
    end

    best
  end

  def name
    "LOOK"
  end
end

# FCFS: assign to the elevator with the fewest pending stops
class FCFSStrategy < SchedulingStrategy
  def select_elevator(elevators, _request)
    best = nil
    min_stops = Float::INFINITY

    elevators.each do |elev|
      next unless elev.has_space?

      # Prefer idle elevators, then least busy
      stops = elev.idle? ? -1 : elev.total_stops

      if stops < min_stops
        min_stops = stops
        best = elev
      end
    end

    best
  end

  def name
    "FCFS"
  end
end

# ─── Elevator System (Controller) ────────────────────────

class ElevatorSystem
  include MonitorMixin

  def initialize(floors, num_elevators, capacity = 10)
    super()  # initializes MonitorMixin
    @num_floors = floors
    @elevators = (1..num_elevators).map do |i|
      Elevator.new(i, 0, floors - 1, capacity)
    end
    @strategy = LOOKStrategy.new

    puts "Elevator System initialized: #{floors} floors," \
         " #{num_elevators} elevators (strategy: #{@strategy.name})"
  end

  def set_strategy(strategy)
    synchronize do
      @strategy = strategy
      puts "Scheduling strategy changed to: #{@strategy.name}"
    end
  end

  # External request: user on a floor presses UP or DOWN
  def request_elevator(floor, direction)
    synchronize do
      if floor < 0 || floor >= @num_floors
        puts "Invalid floor: #{floor}"
        return
      end

      request = ExternalRequest.new(floor, direction)
      assigned = @strategy.select_elevator(@elevators, request)

      unless assigned
        puts "No elevator available for floor #{floor}"
        return
      end

      assigned.add_stop(floor)
      puts "📞 External request: Floor #{floor} #{direction}" \
           " → Assigned to Elevator #{assigned.id}"
    end
  end

  # Internal request: user inside elevator presses a floor button
  def select_floor(elevator_id, floor)
    synchronize do
      if elevator_id < 1 || elevator_id > @elevators.size
        puts "Invalid elevator ID: #{elevator_id}"
        return
      end

      if floor < 0 || floor >= @num_floors
        puts "Invalid floor: #{floor}"
        return
      end

      @elevators[elevator_id - 1].add_stop(floor)
      puts "🔘 Internal request: Elevator #{elevator_id} → Floor #{floor}"
    end
  end

  # Simulate one time step — all elevators move
  def step
    synchronize do
      puts "--- Step ---"
      @elevators.each(&:step)
    end
  end

  # Simulate multiple steps
  def run(steps)
    steps.times { step }
  end

  def display_status
    synchronize do
      puts "\n=== Elevator System Status ==="
      @elevators.each(&:print_status)
      puts "==============================\n"
    end
  end
end

# ─── Usage ───────────────────────────────────────────────

system = ElevatorSystem.new(10, 3)  # 10 floors, 3 elevators

system.display_status

# External requests
system.request_elevator(5, Direction::UP)    # someone on floor 5 wants to go up
system.request_elevator(3, Direction::DOWN)  # someone on floor 3 wants to go down
system.request_elevator(7, Direction::DOWN)  # someone on floor 7 wants to go down

# Simulate movement
system.run(3)

# Internal requests (passengers select destination)
system.select_floor(1, 8)  # elevator 1 passenger wants floor 8
system.select_floor(2, 1)  # elevator 2 passenger wants floor 1

system.run(5)

system.display_status

# Switch strategy
system.set_strategy(FCFSStrategy.new)
system.request_elevator(0, Direction::UP)
system.run(3)

system.display_status
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
| **State** | `Direction` + movement logic | Elevator behavior changes based on direction state |
| **Observer** | Console output on state changes | Log elevator movements (extensible to real notification system) |

### Key Design Decisions

1. **Separate up/down stop sets** — the elevator maintains two sorted sets of stops, one for each direction. This naturally implements the LOOK algorithm. Ruby's `SortedSet` keeps elements ordered automatically
2. **Distance calculation** — accounts for current direction; an elevator moving away from a request has a higher effective distance
3. **Strategy pattern for scheduling** — the system can switch algorithms at runtime without modifying elevator logic
4. **Step-based simulation** — each `step` call moves all elevators one floor, making the system easy to test and reason about
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

```ruby
# ─── Dice (Strategy Pattern) ─────────────────────────────

class Dice
  def roll
    raise NotImplementedError
  end

  def name
    raise NotImplementedError
  end
end

class SingleDice < Dice
  def roll
    rand(1..6)
  end

  def name
    "Single Die (1-6)"
  end
end

class DoubleDice < Dice
  def roll
    rand(1..6) + rand(1..6)
  end

  def name
    "Double Dice (2-12)"
  end
end

# Biased dice — for testing (always returns a fixed value)
class FixedDice < Dice
  def initialize(value)
    @value = value
  end

  def roll
    @value
  end

  def name
    "Fixed (#{@value})"
  end
end

# ─── Board Entity (Snake or Ladder) ──────────────────────

module EntityType
  SNAKE  = :snake
  LADDER = :ladder
end

class BoardEntity
  attr_reader :start_pos, :end_pos, :type

  def initialize(start_pos, end_pos, type)
    @start_pos = start_pos
    @end_pos = end_pos
    @type = type
  end

  def to_s
    if @type == EntityType::SNAKE
      "🐍 Snake: #{@start_pos} → #{@end_pos}"
    else
      "🪜 Ladder: #{@start_pos} → #{@end_pos}"
    end
  end
end

# ─── Board ───────────────────────────────────────────────

class Board
  attr_reader :size

  def initialize(size = 100)
    @size = size
    @entities = {}  # position => BoardEntity
  end

  def add_snake(head, tail)
    if head <= tail
      puts "Invalid snake: head must be above tail."
      return
    end
    if head > @size || tail < 1
      puts "Invalid snake: positions out of bounds."
      return
    end
    @entities[head] = BoardEntity.new(head, tail, EntityType::SNAKE)
  end

  def add_ladder(bottom, top)
    if bottom >= top
      puts "Invalid ladder: bottom must be below top."
      return
    end
    if top > @size || bottom < 1
      puts "Invalid ladder: positions out of bounds."
      return
    end
    @entities[bottom] = BoardEntity.new(bottom, top, EntityType::LADDER)
  end

  # Check if a position has a snake or ladder, return final position
  def final_position(position)
    entity = @entities[position]
    if entity
      puts "    #{entity}"
      return entity.end_pos
    end
    position
  end

  def print_board
    puts "\n=== Board (#{@size} cells) ==="
    puts "Snakes:"
    @entities.each_value do |entity|
      puts "  #{entity}" if entity.type == EntityType::SNAKE
    end
    puts "Ladders:"
    @entities.each_value do |entity|
      puts "  #{entity}" if entity.type == EntityType::LADDER
    end
    puts "============================\n"
  end
end

# ─── Player ──────────────────────────────────────────────

class SnakePlayer
  attr_reader :name
  attr_accessor :position

  def initialize(name)
    @name = name
    @position = 0
  end

  def to_s
    "#{@name} at position #{@position}"
  end
end

# ─── Game Observer ───────────────────────────────────────

class GameObserver
  def on_turn(_player, _dice_value, _old_pos, _new_pos); end
  def on_snake(_player, _from, _to); end
  def on_ladder(_player, _from, _to); end
  def on_win(_player, _turns); end
  def on_bounce(_player, _dice_value); end
end

class ConsoleGameObserver < GameObserver
  def on_turn(player, dice_value, old_pos, new_pos)
    puts "  #{player.name} rolled #{dice_value}: #{old_pos} → #{new_pos}"
  end

  def on_snake(player, from, to)
    puts "  😱 #{player.name} bitten by snake! #{from} → #{to}"
  end

  def on_ladder(player, from, to)
    puts "  🎉 #{player.name} climbed a ladder! #{from} → #{to}"
  end

  def on_win(player, turns)
    puts "\n🏆 #{player.name} WINS in #{turns} turns!"
  end

  def on_bounce(player, dice_value)
    puts "  #{player.name} rolled #{dice_value}" \
         " but can't move (would exceed 100). Stays at #{player.position}"
  end
end

# ─── Game ────────────────────────────────────────────────

class SnakeAndLadderGame
  def initialize(dice = SingleDice.new)
    @board = Board.new
    @players = []
    @dice = dice
    @observers = []
    @current_player_index = 0
    @total_turns = 0
    @game_over = false
  end

  def set_board(board)
    @board = board
  end

  def add_player(name)
    if @players.size >= 4
      puts "Maximum 4 players allowed."
      return
    end
    @players << SnakePlayer.new(name)
    puts "Added player: #{name}"
  end

  def add_observer(observer)
    @observers << observer
  end

  # Play one turn
  def play_turn
    return false if @game_over || @players.empty?

    current = @players[@current_player_index]
    dice_value = @dice.roll
    old_pos = current.position
    new_pos = old_pos + dice_value

    @total_turns += 1

    # Check if move is valid (can't exceed board size)
    if new_pos > @board.size
      notify_bounce(current, dice_value)
      @current_player_index = (@current_player_index + 1) % @players.size
      return true
    end

    # Move to new position
    current.position = new_pos
    notify_turn(current, dice_value, old_pos, new_pos)

    # Check for snake or ladder
    final_pos = @board.final_position(new_pos)
    if final_pos != new_pos
      if final_pos < new_pos
        notify_snake(current, new_pos, final_pos)
      else
        notify_ladder(current, new_pos, final_pos)
      end
      current.position = final_pos
    end

    # Check for win
    if current.position == @board.size
      @game_over = true
      notify_win(current, @total_turns)
      return false
    end

    # Next player
    @current_player_index = (@current_player_index + 1) % @players.size
    true
  end

  # Play the entire game
  def play
    if @players.size < 2
      puts "Need at least 2 players to start."
      return
    end

    puts "\n=== Snake and Ladder Game ==="
    puts "Players: #{@players.map(&:name).join(' ')}"
    puts "Dice: #{@dice.name}"
    @board.print_board

    play_turn while play_turn  # keep playing until game over

    # Final positions
    puts "\n--- Final Positions ---"
    @players.each { |p| puts "  #{p}" }
  end

  private

  def notify_turn(player, dv, op, np)
    @observers.each { |obs| obs.on_turn(player, dv, op, np) }
  end

  def notify_snake(player, from, to)
    @observers.each { |obs| obs.on_snake(player, from, to) }
  end

  def notify_ladder(player, from, to)
    @observers.each { |obs| obs.on_ladder(player, from, to) }
  end

  def notify_win(player, turns)
    @observers.each { |obs| obs.on_win(player, turns) }
  end

  def notify_bounce(player, dv)
    @observers.each { |obs| obs.on_bounce(player, dv) }
  end
end

# ─── Usage ───────────────────────────────────────────────

# Setup board
board = Board.new(100)

# Add snakes
board.add_snake(99, 54)
board.add_snake(70, 55)
board.add_snake(52, 42)
board.add_snake(25, 2)
board.add_snake(95, 72)

# Add ladders
board.add_ladder(6, 25)
board.add_ladder(11, 40)
board.add_ladder(60, 85)
board.add_ladder(46, 90)
board.add_ladder(17, 69)

# Setup game
game = SnakeAndLadderGame.new
game.set_board(board)
game.add_player("Alice")
game.add_player("Bob")
game.add_player("Charlie")
game.add_observer(ConsoleGameObserver.new)

# Play
game.play
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `Dice` (SingleDice, DoubleDice, FixedDice) | Swap dice behavior without changing game logic; FixedDice enables deterministic testing |
| **Observer** | `GameObserver`, `ConsoleGameObserver` | Decouple game events from presentation; extensible for GUI, network, or logging observers |
| **Composite** | `BoardEntity` on `Board` | Board composes snakes and ladders uniformly |

### Key Design Decisions

1. **Board entity abstraction** — snakes and ladders are both `BoardEntity` with a start and end position, differing only in direction. The board maps positions to entities for O(1) lookup via a Hash
2. **Dice as strategy** — pluggable dice allows single die, double dice, or fixed dice for testing. The game doesn't know or care how the dice value is generated. Ruby's duck typing means any object with a `roll` method works
3. **Bounce rule** — if a roll would take a player past 100, they stay in place. This is the standard rule and prevents overshooting
4. **Observer for events** — game events (turns, snake bites, ladder climbs, wins) are broadcast to observers, keeping the game logic clean and the presentation separate
5. **Turn-based architecture** — `play_turn` processes one turn at a time, making the game easy to integrate with a UI or network layer

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

```ruby
require 'monitor'
require 'set'

# ─── Split Type ──────────────────────────────────────────

module SplitType
  EQUAL      = :equal
  EXACT      = :exact
  PERCENTAGE = :percentage
end

# ─── User ────────────────────────────────────────────────

class SplitwiseUser
  attr_reader :user_id, :name, :email

  def initialize(id, name, email)
    @user_id = id
    @name = name
    @email = email
  end
end

# ─── Split (how much each person owes) ───────────────────

Split = Struct.new(:user_id, :amount)

# ─── Split Strategy (Strategy Pattern) ───────────────────

class SplitStrategy
  def calculate(_total_amount, _paid_by, _participants, _split_details)
    raise NotImplementedError
  end

  def validate(_total_amount, _participants, _split_details)
    raise NotImplementedError
  end

  def type
    raise NotImplementedError
  end
end

class EqualSplitStrategy < SplitStrategy
  def calculate(total_amount, paid_by, participants, _split_details = {})
    per_person = (total_amount / participants.size).round(2)

    participants
      .reject { |uid| uid == paid_by }
      .map { |uid| Split.new(uid, per_person) }
  end

  def validate(_total, participants, _details = {})
    participants.size >= 2
  end

  def type
    SplitType::EQUAL
  end
end

class ExactSplitStrategy < SplitStrategy
  def calculate(_total_amount, paid_by, participants, split_details)
    participants
      .reject { |uid| uid == paid_by }
      .filter_map do |uid|
        amount = split_details[uid]
        Split.new(uid, amount) if amount && amount > 0
      end
  end

  def validate(total_amount, _participants, split_details)
    return false if split_details.values.any?(&:negative?)

    sum = split_details.values.sum
    (sum - total_amount).abs < 0.01
  end

  def type
    SplitType::EXACT
  end
end

class PercentageSplitStrategy < SplitStrategy
  def calculate(total_amount, paid_by, participants, split_details)
    participants
      .reject { |uid| uid == paid_by }
      .filter_map do |uid|
        pct = split_details[uid]
        if pct && pct > 0
          amount = (total_amount * pct / 100.0).round(2)
          Split.new(uid, amount)
        end
      end
  end

  def validate(_total, _participants, split_details)
    return false if split_details.values.any? { |v| v < 0 || v > 100 }

    sum = split_details.values.sum
    (sum - 100.0).abs < 0.01
  end

  def type
    SplitType::PERCENTAGE
  end
end

# ─── Expense ─────────────────────────────────────────────

class Expense
  @@next_id = 1

  attr_reader :expense_id, :description, :amount, :paid_by, :split_type, :splits

  def initialize(description, amount, paid_by, split_type, splits)
    @expense_id = @@next_id
    @@next_id += 1
    @description = description
    @amount = amount
    @paid_by = paid_by
    @split_type = split_type
    @splits = splits
  end

  def to_s
    lines = ["Expense ##{@expense_id}: #{@description}" \
             " ($#{'%.2f' % @amount}) paid by #{@paid_by} [#{@split_type}]"]
    @splits.each do |s|
      lines << "  #{s.user_id} owes $#{'%.2f' % s.amount}"
    end
    lines.join("\n")
  end
end

# ─── Balance Sheet ───────────────────────────────────────

class BalanceSheet
  def initialize
    # balances[A][B] > 0 means A owes B that amount
    # balances[A][B] < 0 means B owes A that amount
    @balances = Hash.new { |h, k| h[k] = Hash.new(0.0) }
  end

  def add_debt(debtor, creditor, amount)
    @balances[debtor][creditor] += amount
    @balances[creditor][debtor] -= amount
  end

  def settle(from, to, amount)
    owed = @balances[from][to]
    if owed <= 0
      puts "#{from} does not owe #{to} anything."
      return
    end
    settle_amount = [amount, owed].min
    @balances[from][to] -= settle_amount
    @balances[to][from] += settle_amount
    puts "💰 #{from} paid $#{'%.2f' % settle_amount} to #{to}"
  end

  def get_balance(user1, user2)
    @balances[user1][user2]
  end

  def print_balances(user_id, users)
    user_balances = @balances[user_id]
    if user_balances.empty?
      puts "No balances for #{user_id}"
      return
    end

    puts "Balances for #{users[user_id].name}:"
    user_balances.each do |other_id, amount|
      next if amount.abs < 0.01

      if amount > 0
        puts "  Owes #{users[other_id].name}: $#{'%.2f' % amount}"
      else
        puts "  Owed by #{users[other_id].name}: $#{'%.2f' % -amount}"
      end
    end
  end

  # Simplify debts — minimize number of transactions
  def simplify
    # Calculate net balance for each user
    net = Hash.new(0.0)
    @balances.each do |user, others|
      others.each { |_other, amount| net[user] += amount }
    end

    # Separate into debtors (positive net = owes money) and creditors
    debtors = []
    creditors = []

    net.each do |user, balance|
      if balance > 0.01
        debtors << [user, balance]
      elsif balance < -0.01
        creditors << [user, -balance]
      end
    end

    # Greedy matching: match largest debtor with largest creditor
    transactions = []
    i = 0
    j = 0

    while i < debtors.size && j < creditors.size
      amount = [debtors[i][1], creditors[j][1]].min
      transactions << [debtors[i][0], creditors[j][0], amount]

      debtors[i][1] -= amount
      creditors[j][1] -= amount

      i += 1 if debtors[i][1] < 0.01
      j += 1 if creditors[j][1] < 0.01
    end

    transactions
  end
end

# ─── Group ───────────────────────────────────────────────

class SplitwiseGroup
  attr_reader :group_id, :name, :member_ids, :expenses

  def initialize(id, name)
    @group_id = id
    @name = name
    @member_ids = Set.new
    @expenses = []
  end

  def add_member(user_id)
    @member_ids.add(user_id)
  end

  def add_expense(expense)
    @expenses << expense
  end
end

# ─── Expense Manager (Facade) ────────────────────────────

class ExpenseManager
  include MonitorMixin

  def initialize
    super
    @users = {}
    @groups = {}
    @balance_sheet = BalanceSheet.new

    # Strategy registry
    @strategies = {
      SplitType::EQUAL      => EqualSplitStrategy.new,
      SplitType::EXACT      => ExactSplitStrategy.new,
      SplitType::PERCENTAGE => PercentageSplitStrategy.new
    }
  end

  # ─── User Management ───────────────────────────────────

  def add_user(id, name, email)
    synchronize do
      @users[id] = SplitwiseUser.new(id, name, email)
      puts "User registered: #{name} (#{id})"
    end
  end

  # ─── Group Management ──────────────────────────────────

  def create_group(group_id, name, member_ids)
    synchronize do
      group = SplitwiseGroup.new(group_id, name)
      member_ids.each do |mid|
        group.add_member(mid) if @users.key?(mid)
      end
      @groups[group_id] = group
      puts "Group created: #{name} with #{member_ids.size} members"
    end
  end

  # ─── Add Expense ───────────────────────────────────────

  def add_expense(group_id, description, amount, paid_by,
                  split_type, split_details = {})
    synchronize do
      group = @groups[group_id]
      unless group
        puts "Error: Group #{group_id} not found."
        return false
      end

      unless group.member_ids.include?(paid_by)
        puts "Error: Payer #{paid_by} is not in the group."
        return false
      end

      # Get participants
      participants = group.member_ids.to_a

      # Validate split
      strategy = @strategies[split_type]
      unless strategy.validate(amount, participants, split_details)
        puts "Error: Invalid split details for #{split_type} split."
        return false
      end

      # Calculate splits
      splits = strategy.calculate(amount, paid_by, participants, split_details)

      # Create expense
      expense = Expense.new(description, amount, paid_by, split_type, splits)
      group.add_expense(expense)

      # Update balances
      splits.each do |split|
        @balance_sheet.add_debt(split.user_id, paid_by, split.amount)
      end

      puts "✅ #{expense}"
      true
    end
  end

  # ─── Settle Up ─────────────────────────────────────────

  def settle_up(from, to, amount)
    synchronize { @balance_sheet.settle(from, to, amount) }
  end

  # ─── View Balances ─────────────────────────────────────

  def show_balances(user_id)
    synchronize { @balance_sheet.print_balances(user_id, @users) }
  end

  def show_all_balances
    synchronize do
      puts "\n=== All Balances ==="
      @users.each_key { |uid| @balance_sheet.print_balances(uid, @users) }
      puts "====================\n"
    end
  end

  # ─── Simplify Debts ────────────────────────────────────

  def simplify_debts
    synchronize do
      transactions = @balance_sheet.simplify

      puts "\n=== Simplified Debts ==="
      if transactions.empty?
        puts "All settled up!"
      else
        transactions.each do |from, to, amount|
          puts "  #{@users[from].name} pays #{@users[to].name}:" \
               " $#{'%.2f' % amount}"
        end
      end
      puts "========================\n"
    end
  end
end

# ─── Usage ───────────────────────────────────────────────

manager = ExpenseManager.new

# Register users
manager.add_user("U1", "Alice", "alice@example.com")
manager.add_user("U2", "Bob", "bob@example.com")
manager.add_user("U3", "Charlie", "charlie@example.com")
manager.add_user("U4", "Diana", "diana@example.com")

# Create a group
manager.create_group("G1", "Trip to Paris", %w[U1 U2 U3 U4])

# Add expenses
puts "\n--- Adding Expenses ---"

# Equal split: Alice pays $100 for dinner, split equally among 4
manager.add_expense("G1", "Dinner", 100.0, "U1", SplitType::EQUAL)

# Exact split: Bob pays $60 for taxi
manager.add_expense("G1", "Taxi", 60.0, "U2", SplitType::EXACT,
                    { "U1" => 10.0, "U2" => 20.0, "U3" => 15.0, "U4" => 15.0 })

# Percentage split: Charlie pays $200 for hotel
manager.add_expense("G1", "Hotel", 200.0, "U3", SplitType::PERCENTAGE,
                    { "U1" => 30.0, "U2" => 30.0, "U3" => 20.0, "U4" => 20.0 })

# View balances
puts "\n--- Individual Balances ---"
manager.show_balances("U1")
manager.show_balances("U2")

# Simplify debts
manager.simplify_debts

# Settle up
puts "--- Settling Up ---"
manager.settle_up("U1", "U3", 50.0)

manager.show_balances("U1")
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `SplitStrategy` (Equal, Exact, Percentage) | Different splitting algorithms without changing expense logic |
| **Facade** | `ExpenseManager` | Simple interface hiding complex balance tracking and split calculations |
| **Observer** | Console output on expense/settlement events | Extensible for push notifications, email alerts |

### Key Design Decisions

1. **Bidirectional balance tracking** — `balances[A][B]` and `balances[B][A]` are always negatives of each other, ensuring consistency. If A owes B $50, then `balances[A][B] = 50` and `balances[B][A] = -50`. Ruby's `Hash.new(0.0)` provides clean default values
2. **Strategy registry** — split strategies are registered by type in a hash, making it easy to add new split types (e.g., share-based, weight-based) without modifying the manager
3. **Debt simplification** — the greedy algorithm calculates net balances and matches debtors with creditors, minimizing the number of transactions needed to settle all debts
4. **Validation before calculation** — each strategy validates its inputs (exact amounts sum to total, percentages sum to 100) before computing splits
5. **Rounding** — amounts are rounded to 2 decimal places using Ruby's `round(2)` to avoid floating-point accumulation errors in currency calculations

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

```ruby
require 'monitor'

# ─── Enums ───────────────────────────────────────────────

module SeatCategory
  SILVER   = :silver
  GOLD     = :gold
  PLATINUM = :platinum
end

module SeatStatus
  AVAILABLE = :available
  LOCKED    = :locked
  BOOKED    = :booked
end

module BookingStatus
  PENDING   = :pending
  CONFIRMED = :confirmed
  CANCELLED = :cancelled
end

module PaymentStatus
  PENDING = :pending
  SUCCESS = :success
  FAILED  = :failed
end

# ─── Movie ───────────────────────────────────────────────

class Movie
  attr_reader :movie_id, :title, :genre, :duration_minutes, :language

  def initialize(id, title, genre, duration, language)
    @movie_id = id
    @title = title
    @genre = genre
    @duration_minutes = duration
    @language = language
  end
end

# ─── Seat ────────────────────────────────────────────────

class Seat
  attr_reader :seat_id, :row, :col, :category

  def initialize(id, row, col, category)
    @seat_id = id
    @row = row
    @col = col
    @category = category
  end
end

# ─── ShowSeat (seat status for a specific show) ──────────

class ShowSeat
  attr_reader :seat_id, :category, :status, :price, :locked_by

  def initialize(seat_id, category, price)
    @seat_id = seat_id
    @category = category
    @status = SeatStatus::AVAILABLE
    @price = price
    @locked_by = nil
    @lock_expiry = nil
  end

  def available?
    return true if @status == SeatStatus::AVAILABLE
    # Check if lock has expired
    if @status == SeatStatus::LOCKED && @lock_expiry
      return Time.now > @lock_expiry
    end
    false
  end

  def lock(user_id, lock_seconds = 300)
    return false unless available?

    @status = SeatStatus::LOCKED
    @locked_by = user_id
    @lock_expiry = Time.now + lock_seconds
    true
  end

  def book(user_id)
    return false unless @status == SeatStatus::LOCKED && @locked_by == user_id

    @status = SeatStatus::BOOKED
    true
  end

  def release
    @status = SeatStatus::AVAILABLE
    @locked_by = nil
    @lock_expiry = nil
  end
end

# ─── Screen ──────────────────────────────────────────────

class Screen
  attr_reader :screen_id, :name, :seats, :total_rows, :total_cols

  def initialize(id, name, rows, cols, row_categories)
    @screen_id = id
    @name = name
    @total_rows = rows
    @total_cols = cols
    @seats = []

    # row_categories: array of [num_rows, category]
    current_row = 0
    row_categories.each do |num_rows, category|
      num_rows.times do
        row_letter = ('A'.ord + current_row).chr
        (1..cols).each do |c|
          seat_id = "#{row_letter}#{c}"
          @seats << Seat.new(seat_id, current_row, c, category)
        end
        current_row += 1
      end
    end
  end
end

# ─── Pricing Strategy (Strategy Pattern) ─────────────────

class PricingStrategy
  def get_price(_category)
    raise NotImplementedError
  end

  def name
    raise NotImplementedError
  end
end

class StandardMoviePricing < PricingStrategy
  PRICES = {
    SeatCategory::SILVER   => 150.0,
    SeatCategory::GOLD     => 250.0,
    SeatCategory::PLATINUM => 400.0
  }.freeze

  def get_price(category)
    PRICES[category] || 0
  end

  def name
    "Standard"
  end
end

class WeekendMoviePricing < PricingStrategy
  PRICES = {
    SeatCategory::SILVER   => 200.0,
    SeatCategory::GOLD     => 350.0,
    SeatCategory::PLATINUM => 500.0
  }.freeze

  def get_price(category)
    PRICES[category] || 0
  end

  def name
    "Weekend"
  end
end

# ─── Show ────────────────────────────────────────────────

class Show
  include MonitorMixin

  attr_reader :show_id, :movie_id, :screen_id, :start_time

  def initialize(id, movie_id, screen_id, start_time, screen, pricing)
    super()  # initializes MonitorMixin
    @show_id = id
    @movie_id = movie_id
    @screen_id = screen_id
    @start_time = start_time
    @show_seats = {}  # seat_id => ShowSeat

    screen.seats.each do |seat|
      price = pricing.get_price(seat.category)
      @show_seats[seat.seat_id] = ShowSeat.new(seat.seat_id, seat.category, price)
    end
  end

  # Get available seats
  def available_seats
    synchronize do
      @show_seats.values.select(&:available?)
    end
  end

  # Lock seats for a user (returns true if all seats locked successfully)
  def lock_seats(seat_ids, user_id)
    synchronize do
      # First check all seats are available
      return false unless seat_ids.all? { |sid| @show_seats[sid]&.available? }

      # Lock all seats atomically
      seat_ids.each { |sid| @show_seats[sid].lock(user_id) }
      true
    end
  end

  # Book locked seats (after payment)
  def book_seats(seat_ids, user_id)
    synchronize do
      seat_ids.each do |sid|
        show_seat = @show_seats[sid]
        unless show_seat&.book(user_id)
          # Rollback: release all seats locked by this user
          seat_ids.each do |s|
            ss = @show_seats[s]
            ss.release if ss && ss.locked_by == user_id
          end
          return false
        end
      end
      true
    end
  end

  # Release locked seats (on payment failure or timeout)
  def release_seats(seat_ids, user_id)
    synchronize do
      seat_ids.each do |sid|
        show_seat = @show_seats[sid]
        show_seat.release if show_seat && show_seat.locked_by == user_id
      end
    end
  end

  def calculate_total(seat_ids)
    synchronize do
      seat_ids.sum { |sid| @show_seats[sid]&.price || 0 }
    end
  end

  def display_seats
    synchronize do
      puts "Show #{@show_id} at #{@start_time}:"
      avail = @show_seats.values.count(&:available?)
      locked = @show_seats.values.count { |s| s.status == SeatStatus::LOCKED }
      booked = @show_seats.values.count { |s| s.status == SeatStatus::BOOKED }
      puts "  Available: #{avail} | Locked: #{locked} | Booked: #{booked}"
    end
  end
end

# ─── Booking ─────────────────────────────────────────────

class MovieBooking
  @@next_id = 1

  attr_reader :booking_id, :user_id, :show_id, :seat_ids, :total_amount, :status

  def initialize(user_id, show_id, seats, amount)
    @booking_id = "BK#{@@next_id}"
    @@next_id += 1
    @user_id = user_id
    @show_id = show_id
    @seat_ids = seats
    @total_amount = amount
    @status = BookingStatus::PENDING
    @payment_status = PaymentStatus::PENDING
  end

  def confirm
    @status = BookingStatus::CONFIRMED
    @payment_status = PaymentStatus::SUCCESS
  end

  def cancel
    @status = BookingStatus::CANCELLED
  end

  def payment_failed
    @payment_status = PaymentStatus::FAILED
  end

  def to_s
    "Booking #{@booking_id}: Show #{@show_id}" \
      " | Seats: #{@seat_ids.join(' ')}" \
      " | Total: $#{'%.2f' % @total_amount}" \
      " | Status: #{@status}"
  end
end

# ─── Theater ─────────────────────────────────────────────

class Theater
  attr_reader :theater_id, :name, :city

  def initialize(id, name, city)
    @theater_id = id
    @name = name
    @city = city
    @screens = {}
  end

  def add_screen(screen)
    @screens[screen.screen_id] = screen
  end

  def get_screen(id)
    @screens[id]
  end
end

# ─── Booking System (Facade + Singleton) ─────────────────

class BookingSystem
  include MonitorMixin

  @instance = nil
  @instance_mutex = Mutex.new

  def self.instance
    @instance_mutex.synchronize do
      @instance ||= new
    end
  end

  private_class_method :new

  def initialize
    super
    @movies = {}
    @theaters = {}
    @shows = {}
    @bookings = {}
  end

  # ─── Setup ─────────────────────────────────────────────

  def add_movie(movie)
    synchronize do
      @movies[movie.movie_id] = movie
      puts "Movie added: #{movie.title}"
    end
  end

  def add_theater(theater)
    synchronize do
      @theaters[theater.theater_id] = theater
      puts "Theater added: #{theater.name} (#{theater.city})"
    end
  end

  def add_show(show_id, movie_id, theater_id, screen_id, start_time, pricing)
    synchronize do
      theater = @theaters[theater_id]
      unless theater
        puts "Error: Theater not found."
        return
      end

      screen = theater.get_screen(screen_id)
      unless screen
        puts "Error: Screen not found."
        return
      end

      @shows[show_id] = Show.new(show_id, movie_id, screen_id,
                                 start_time, screen, pricing)
      puts "Show added: #{@movies[movie_id].title}" \
           " at #{start_time} on #{screen.name}"
    end
  end

  # ─── Search ────────────────────────────────────────────

  def search_shows_by_movie(movie_id)
    synchronize do
      @shows.values.select { |show| show.movie_id == movie_id }
    end
  end

  # ─── Booking Flow ──────────────────────────────────────

  # Step 1: View available seats
  def view_available_seats(show_id)
    synchronize do
      show = @shows[show_id]
      unless show
        puts "Show not found."
        return
      end

      available = show.available_seats
      puts "\nAvailable seats for show #{show_id}:"
      available.each do |seat|
        puts "  #{seat.seat_id} [#{seat.category}] $#{'%.2f' % seat.price}"
      end
      puts "Total available: #{available.size}"
    end
  end

  # Step 2: Select and lock seats
  def select_seats(show_id, seat_ids, user_id)
    synchronize do
      show = @shows[show_id]
      unless show
        puts "Show not found."
        return nil
      end

      # Try to lock seats
      unless show.lock_seats(seat_ids, user_id)
        puts "❌ Some seats are not available. Please try again."
        return nil
      end

      # Calculate total
      total = show.calculate_total(seat_ids)

      # Create pending booking
      booking = MovieBooking.new(user_id, show_id, seat_ids, total)
      @bookings[booking.booking_id] = booking

      puts "🔒 Seats locked for #{user_id}. Booking: #{booking.booking_id}" \
           " | Total: $#{'%.2f' % total}" \
           " | Complete payment within 5 minutes."

      booking.booking_id
    end
  end

  # Step 3: Process payment and confirm
  def confirm_booking(booking_id, payment_success = true)
    synchronize do
      booking = @bookings[booking_id]
      unless booking
        puts "Booking not found."
        return false
      end

      show = @shows[booking.show_id]

      if payment_success
        if show.book_seats(booking.seat_ids, booking.user_id)
          booking.confirm
          puts "✅ Booking confirmed! #{booking}"
          return true
        else
          puts "❌ Failed to book seats (lock may have expired)."
          booking.payment_failed
          return false
        end
      else
        # Payment failed — release seats
        show.release_seats(booking.seat_ids, booking.user_id)
        booking.payment_failed
        puts "❌ Payment failed. Seats released."
        false
      end
    end
  end

  # Cancel booking
  def cancel_booking(booking_id)
    synchronize do
      booking = @bookings[booking_id]
      unless booking
        puts "Booking not found."
        return false
      end

      unless booking.status == BookingStatus::CONFIRMED
        puts "Can only cancel confirmed bookings."
        return false
      end

      show = @shows[booking.show_id]
      show.release_seats(booking.seat_ids, booking.user_id)
      booking.cancel

      puts "🚫 Booking #{booking_id} cancelled. Seats released."
      true
    end
  end

  def display_show_status(show_id)
    synchronize do
      show = @shows[show_id]
      show&.display_seats
    end
  end
end

# ─── Usage ───────────────────────────────────────────────

system = BookingSystem.instance

# Add movie
system.add_movie(Movie.new("MOV1", "Inception", "Sci-Fi", 148, "English"))

# Add theater with screen
theater = Theater.new("TH1", "PVR Cinemas", "Mumbai")
screen = Screen.new("SCR1", "Screen 1", 8, 10, [
  [2, SeatCategory::PLATINUM],  # rows A-B: Platinum
  [3, SeatCategory::GOLD],      # rows C-E: Gold
  [3, SeatCategory::SILVER]     # rows F-H: Silver
])
theater.add_screen(screen)
system.add_theater(theater)

# Add show
pricing = StandardMoviePricing.new
system.add_show("SH1", "MOV1", "TH1", "SCR1", "7:00 PM", pricing)

# User 1: View and book seats
puts "\n--- User 1: Alice ---"
system.view_available_seats("SH1")

booking1 = system.select_seats("SH1", %w[A1 A2], "alice")
system.confirm_booking(booking1, true) if booking1  # payment success

# User 2: Try to book same seats (should fail)
puts "\n--- User 2: Bob (tries same seats) ---"
booking2 = system.select_seats("SH1", %w[A1 A3], "bob")

# User 2: Book different seats
puts "\n--- User 2: Bob (different seats) ---"
booking3 = system.select_seats("SH1", %w[C1 C2 C3], "bob")
system.confirm_booking(booking3, true) if booking3

# User 3: Payment fails
puts "\n--- User 3: Charlie (payment fails) ---"
booking4 = system.select_seats("SH1", %w[F1 F2], "charlie")
system.confirm_booking(booking4, false) if booking4  # payment fails

# Show status
puts "\n--- Show Status ---"
system.display_show_status("SH1")

# Cancel booking
puts "\n--- Cancel Alice's booking ---"
system.cancel_booking(booking1) if booking1

system.display_show_status("SH1")
```

---

### Booking Flow (Sequence)

```
User                    System                  Show
 │                        │                       │
 │  view_available_seats  │                       │
 │───────────────────────▶│  available_seats      │
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
2. **Atomic seat locking** — all requested seats are checked for availability before any are locked. If any seat is unavailable, none are locked (all-or-nothing). Ruby's `MonitorMixin` ensures thread safety
3. **Lock expiry** — locked seats automatically become available after 5 minutes, preventing indefinite holds from abandoned sessions. The `available?` method checks expiry time
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

```ruby
require 'monitor'

# ─── Enums ───────────────────────────────────────────────

module OrderStatus
  PLACED    = :placed
  CONFIRMED = :confirmed
  SHIPPED   = :shipped
  DELIVERED = :delivered
  CANCELLED = :cancelled
end

module PaymentMethod
  CREDIT_CARD = :credit_card
  DEBIT_CARD  = :debit_card
  UPI         = :upi
end

# ─── Product ─────────────────────────────────────────────

class ShopProduct
  include MonitorMixin

  attr_reader :product_id, :name, :description, :category, :price

  def initialize(id, name, description, category, price, stock)
    super()  # initializes MonitorMixin
    @product_id = id
    @name = name
    @description = description
    @category = category
    @price = price
    @stock = stock
  end

  def stock
    synchronize { @stock }
  end

  def in_stock?(quantity = 1)
    synchronize { @stock >= quantity }
  end

  def reserve(quantity)
    synchronize do
      return false if @stock < quantity
      @stock -= quantity
      true
    end
  end

  def restock(quantity)
    synchronize { @stock += quantity }
  end

  def to_s
    "#{@product_id}: #{@name} - $#{'%.2f' % @price}" \
      " [#{@category}] (Stock: #{stock})"
  end
end

# ─── Cart Item ───────────────────────────────────────────

CartItem = Struct.new(:product_id, :product_name, :unit_price, :quantity) do
  def total
    unit_price * quantity
  end
end

# ─── Shopping Cart ───────────────────────────────────────

class ShoppingCart
  include MonitorMixin

  def initialize
    super
    @items = {}  # product_id => CartItem
  end

  def add_item(product, quantity = 1)
    synchronize do
      existing = @items[product.product_id]
      if existing
        existing.quantity += quantity
      else
        @items[product.product_id] = CartItem.new(
          product.product_id, product.name, product.price, quantity
        )
      end
      puts "  Added #{quantity}x #{product.name} to cart"
    end
  end

  def remove_item(product_id)
    synchronize { @items.delete(product_id) }
  end

  def update_quantity(product_id, quantity)
    synchronize do
      if quantity <= 0
        @items.delete(product_id)
      else
        item = @items[product_id]
        item.quantity = quantity if item
      end
    end
  end

  def get_total
    synchronize { @items.values.sum(&:total) }
  end

  def get_items
    synchronize { @items.values.dup }
  end

  def empty?
    synchronize { @items.empty? }
  end

  def clear
    synchronize { @items.clear }
  end

  def print_cart
    synchronize do
      puts "Shopping Cart:"
      if @items.empty?
        puts "  (empty)"
        return
      end
      @items.each_value do |item|
        puts "  #{item.product_name} x#{item.quantity}" \
             " @ $#{'%.2f' % item.unit_price} = $#{'%.2f' % item.total}"
      end
      puts "  Total: $#{'%.2f' % @items.values.sum(&:total)}"
    end
  end
end

# ─── Coupon / Discount (Strategy Pattern) ────────────────

class DiscountStrategy
  def apply(_total)
    raise NotImplementedError
  end

  def description
    raise NotImplementedError
  end

  def valid?
    true
  end
end

class PercentageDiscount < DiscountStrategy
  def initialize(code, percentage, max_discount = Float::INFINITY)
    @code = code
    @percentage = percentage
    @max_discount = max_discount
  end

  def apply(total)
    discount = total * @percentage / 100.0
    [discount, @max_discount].min
  end

  def description
    desc = "#{@code}: #{@percentage.to_i}% off"
    desc += " (max $#{@max_discount.to_i})" if @max_discount < Float::INFINITY
    desc
  end
end

class FlatDiscount < DiscountStrategy
  def initialize(code, amount, min_order = 0)
    @code = code
    @amount = amount
    @min_order = min_order
  end

  def apply(total)
    return 0 if total < @min_order
    [total, @amount].min
  end

  def description
    desc = "#{@code}: $#{@amount.to_i} off"
    desc += " (min order $#{@min_order.to_i})" if @min_order > 0
    desc
  end
end

# ─── Payment Strategy ────────────────────────────────────

class PaymentProcessor
  def process(_amount)
    raise NotImplementedError
  end

  def name
    raise NotImplementedError
  end
end

class CreditCardPayment < PaymentProcessor
  def initialize(card_number)
    @card_number = card_number
  end

  def process(amount)
    puts "  💳 Processing Credit Card payment of $#{'%.2f' % amount}" \
         " (card: ****#{@card_number[-4..]})"
    true  # simulate success
  end

  def name
    "Credit Card"
  end
end

class UPIPayment < PaymentProcessor
  def initialize(upi_id)
    @upi_id = upi_id
  end

  def process(amount)
    puts "  📱 Processing UPI payment of $#{'%.2f' % amount}" \
         " (UPI: #{@upi_id})"
    true
  end

  def name
    "UPI"
  end
end

# ─── Order ───────────────────────────────────────────────

OrderItem = Struct.new(:product_id, :product_name, :unit_price, :quantity) do
  def total
    unit_price * quantity
  end
end

class Order
  @@next_id = 1

  attr_reader :order_id, :user_id, :items, :subtotal, :discount,
              :total, :status, :payment_method

  def initialize(user_id, cart_items, subtotal, discount, payment)
    @order_id = "ORD#{@@next_id}"
    @@next_id += 1
    @user_id = user_id
    @subtotal = subtotal
    @discount = discount
    @total = subtotal - discount
    @status = OrderStatus::PLACED
    @payment_method = payment

    @items = cart_items.map do |ci|
      OrderItem.new(ci.product_id, ci.product_name, ci.unit_price, ci.quantity)
    end
  end

  def set_status(status)
    @status = status
  end

  def to_s
    lines = ["Order #{@order_id} [#{@status}]"]
    @items.each do |item|
      lines << "  #{item.product_name} x#{item.quantity}" \
               " @ $#{'%.2f' % item.unit_price} = $#{'%.2f' % item.total}"
    end
    lines << "  Subtotal: $#{'%.2f' % @subtotal}"
    lines << "  Discount: -$#{'%.2f' % @discount}" if @discount > 0
    lines << "  Total: $#{'%.2f' % @total}"
    lines << "  Payment: #{@payment_method}"
    lines.join("\n")
  end
end

# ─── Shopping System (Facade) ────────────────────────────

class ShoppingSystem
  include MonitorMixin

  def initialize
    super
    @products = {}
    @carts = {}       # user_id => ShoppingCart
    @orders = {}      # user_id => [Order]
    @coupons = {}     # code => DiscountStrategy
  end

  # ─── Product Management ────────────────────────────────

  def add_product(product)
    synchronize do
      @products[product.product_id] = product
      puts "Product added: #{product}"
    end
  end

  def search_by_name(query)
    synchronize do
      lower_query = query.downcase
      @products.values.select { |p| p.name.downcase.include?(lower_query) }
    end
  end

  def search_by_category(category)
    synchronize do
      @products.values.select { |p| p.category == category }
    end
  end

  # ─── Cart Operations ──────────────────────────────────

  def add_to_cart(user_id, product_id, qty = 1)
    synchronize do
      product = @products[product_id]
      unless product
        puts "Product not found."
        return
      end

      unless product.in_stock?(qty)
        puts "Insufficient stock for #{product.name}"
        return
      end

      @carts[user_id] ||= ShoppingCart.new
      @carts[user_id].add_item(product, qty)
    end
  end

  def remove_from_cart(user_id, product_id)
    synchronize do
      @carts[user_id]&.remove_item(product_id)
    end
  end

  def view_cart(user_id)
    synchronize do
      cart = @carts[user_id]
      if cart.nil? || cart.empty?
        puts "Cart is empty."
        return
      end
      cart.print_cart
    end
  end

  # ─── Coupon Management ─────────────────────────────────

  def add_coupon(code, discount)
    synchronize do
      puts "Coupon added: #{discount.description}"
      @coupons[code] = discount
    end
  end

  # ─── Place Order ───────────────────────────────────────

  def place_order(user_id, payment, coupon_code = "")
    synchronize do
      cart = @carts[user_id]
      if cart.nil? || cart.empty?
        puts "Cart is empty. Cannot place order."
        return nil
      end

      cart_items = cart.get_items
      subtotal = cart.get_total

      # Validate stock for all items
      cart_items.each do |item|
        product = @products[item.product_id]
        unless product.in_stock?(item.quantity)
          puts "❌ #{item.product_name} is out of stock."
          return nil
        end
      end

      # Apply coupon
      discount = 0.0
      unless coupon_code.empty?
        coupon = @coupons[coupon_code]
        if coupon&.valid?
          discount = coupon.apply(subtotal)
          puts "  🎟️ Coupon applied: #{coupon.description}" \
               " (-$#{'%.2f' % discount})"
        else
          puts "  Invalid coupon code: #{coupon_code}"
        end
      end

      total = subtotal - discount

      # Process payment
      unless payment.process(total)
        puts "❌ Payment failed."
        return nil
      end

      # Reserve inventory
      cart_items.each do |item|
        @products[item.product_id].reserve(item.quantity)
      end

      # Create order
      order = Order.new(user_id, cart_items, subtotal, discount, payment.name)
      order.set_status(OrderStatus::CONFIRMED)

      puts "✅ Order placed!"
      puts order

      @orders[user_id] ||= []
      @orders[user_id] << order
      cart.clear

      order.order_id
    end
  end

  # ─── Cancel Order ──────────────────────────────────────

  def cancel_order(user_id, order_id)
    synchronize do
      user_orders = @orders[user_id]
      unless user_orders
        puts "No orders found."
        return false
      end

      order = user_orders.find { |o| o.order_id == order_id }
      unless order
        puts "Order not found."
        return false
      end

      if order.status == OrderStatus::DELIVERED
        puts "Cannot cancel a delivered order."
        return false
      end

      if order.status == OrderStatus::CANCELLED
        puts "Order already cancelled."
        return false
      end

      # Restore inventory
      order.items.each do |item|
        @products[item.product_id].restock(item.quantity)
      end

      order.set_status(OrderStatus::CANCELLED)
      puts "🚫 Order #{order_id} cancelled." \
           " Inventory restored. Refund of $#{'%.2f' % order.total} initiated."
      true
    end
  end

  # ─── Order History ─────────────────────────────────────

  def view_orders(user_id)
    synchronize do
      user_orders = @orders[user_id]
      if user_orders.nil? || user_orders.empty?
        puts "No orders found for #{user_id}"
        return
      end

      puts "\n=== Order History for #{user_id} ==="
      user_orders.each do |order|
        puts order
        puts "---"
      end
    end
  end

  # ─── Update Order Status ───────────────────────────────

  def update_order_status(user_id, order_id, new_status)
    synchronize do
      user_orders = @orders[user_id]
      return unless user_orders

      order = user_orders.find { |o| o.order_id == order_id }
      if order
        order.set_status(new_status)
        puts "📦 Order #{order_id} status updated to: #{new_status}"
      end
    end
  end
end

# ─── Usage ───────────────────────────────────────────────

shop = ShoppingSystem.new

# Add products
shop.add_product(ShopProduct.new("P1", "Laptop", "High-performance laptop",
                                 "Electronics", 999.99, 10))
shop.add_product(ShopProduct.new("P2", "Headphones", "Noise-cancelling headphones",
                                 "Electronics", 199.99, 50))
shop.add_product(ShopProduct.new("P3", "C++ Book", "The C++ Programming Language",
                                 "Books", 49.99, 30))
shop.add_product(ShopProduct.new("P4", "Mouse", "Wireless ergonomic mouse",
                                 "Electronics", 29.99, 100))

# Add coupons
shop.add_coupon("SAVE10", PercentageDiscount.new("SAVE10", 10, 100))
shop.add_coupon("FLAT50", FlatDiscount.new("FLAT50", 50, 200))

# User shopping
puts "\n--- Alice Shopping ---"

# Search
puts "Search 'laptop':"
results = shop.search_by_name("laptop")
results.each { |p| puts p }

# Add to cart
shop.add_to_cart("alice", "P1", 1)
shop.add_to_cart("alice", "P2", 2)
shop.add_to_cart("alice", "P3", 1)

shop.view_cart("alice")

# Place order with coupon
puts "\n--- Place Order ---"
order_id = shop.place_order(
  "alice",
  CreditCardPayment.new("4111111111111111"),
  "SAVE10"
)

# View order history
shop.view_orders("alice")

# Update order status
if order_id
  shop.update_order_status("alice", order_id, OrderStatus::SHIPPED)
  shop.update_order_status("alice", order_id, OrderStatus::DELIVERED)
end

# Bob tries to order
puts "\n--- Bob Shopping ---"
shop.add_to_cart("bob", "P4", 3)
shop.place_order("bob", UPIPayment.new("bob@upi"), "FLAT50")
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

1. **Cart per user** — each user has an independent shopping cart, stored in a hash. Cart operations are thread-safe via `MonitorMixin`
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

```ruby
require 'monitor'

# ─── Enums ───────────────────────────────────────────────

module VehicleType
  HATCHBACK = :hatchback
  SEDAN     = :sedan
  SUV       = :suv
  LUXURY    = :luxury
end

module ReservationStatus
  ACTIVE    = :active
  COMPLETED = :completed
  CANCELLED = :cancelled
end

# ─── Date (simplified) ──────────────────────────────────

class RentalDate
  attr_reader :year, :month, :day

  def initialize(year, month, day)
    @year = year
    @month = month
    @day = day
  end

  def <=>(other)
    [year, month, day] <=> [other.year, other.month, other.day]
  end

  include Comparable

  def days_until(other)
    # Simplified: assume 30 days per month
    (other.year - year) * 365 + (other.month - month) * 30 + (other.day - day)
  end

  def to_s
    format('%04d-%02d-%02d', year, month, day)
  end

  def self.overlaps?(s1, e1, s2, e2)
    s1 < e2 && s2 < e1
  end
end

# ─── Vehicle ─────────────────────────────────────────────

class RentalVehicle
  attr_reader :vehicle_id, :make, :model, :year, :type, :license_plate
  attr_accessor :location

  def initialize(id, make, model, year, type, plate, location)
    @vehicle_id = id
    @make = make
    @model = model
    @year = year
    @type = type
    @license_plate = plate
    @location = location
  end

  def to_s
    "#{@vehicle_id}: #{@year} #{@make} #{@model}" \
      " [#{@type}] Plate: #{@license_plate} @ #{@location}"
  end
end

# ─── Customer ────────────────────────────────────────────

class RentalCustomer
  attr_reader :customer_id, :name, :email, :license_number

  def initialize(id, name, email, license)
    @customer_id = id
    @name = name
    @email = email
    @license_number = license
  end
end

# ─── Add-On Service (Builder Pattern) ────────────────────

AddOn = Struct.new(:name, :daily_rate)

class AddOnBuilder
  def initialize
    @add_ons = []
  end

  def with_insurance(rate = 15.0)
    @add_ons << AddOn.new("Insurance", rate)
    self
  end

  def with_gps(rate = 5.0)
    @add_ons << AddOn.new("GPS Navigation", rate)
    self
  end

  def with_child_seat(rate = 8.0)
    @add_ons << AddOn.new("Child Seat", rate)
    self
  end

  def with_roadside_assistance(rate = 10.0)
    @add_ons << AddOn.new("Roadside Assistance", rate)
    self
  end

  def build
    @add_ons.dup
  end

  def daily_total
    @add_ons.sum(&:daily_rate)
  end
end

# ─── Pricing Strategy (Strategy Pattern) ─────────────────

class RentalPricingStrategy
  def calculate(_type, _days)
    raise NotImplementedError
  end

  def name
    raise NotImplementedError
  end
end

class DailyPricing < RentalPricingStrategy
  RATES = {
    VehicleType::HATCHBACK => 30.0,
    VehicleType::SEDAN     => 50.0,
    VehicleType::SUV       => 75.0,
    VehicleType::LUXURY    => 150.0
  }.freeze

  def calculate(type, days)
    RATES[type] * days
  end

  def name
    "Daily"
  end
end

class WeeklyPricing < RentalPricingStrategy
  WEEKLY_RATES = {
    VehicleType::HATCHBACK => 180.0,
    VehicleType::SEDAN     => 300.0,
    VehicleType::SUV       => 450.0,
    VehicleType::LUXURY    => 900.0
  }.freeze

  DAILY_RATES = {
    VehicleType::HATCHBACK => 30.0,
    VehicleType::SEDAN     => 50.0,
    VehicleType::SUV       => 75.0,
    VehicleType::LUXURY    => 150.0
  }.freeze

  def calculate(type, days)
    weeks = days / 7
    remaining = days % 7
    weeks * WEEKLY_RATES[type] + remaining * DAILY_RATES[type]
  end

  def name
    "Weekly"
  end
end

# ─── Reservation ─────────────────────────────────────────

class RentalReservation
  @@next_id = 1

  attr_reader :reservation_id, :customer_id, :vehicle_id,
              :start_date, :end_date, :pickup_location, :return_location,
              :add_ons, :vehicle_cost, :addon_cost, :total_cost, :status

  def initialize(customer_id, vehicle_id, start_date, end_date,
                 pickup, return_loc, addons, vehicle_cost, addon_cost)
    @reservation_id = "RES#{@@next_id}"
    @@next_id += 1
    @customer_id = customer_id
    @vehicle_id = vehicle_id
    @start_date = start_date
    @end_date = end_date
    @pickup_location = pickup
    @return_location = return_loc
    @add_ons = addons
    @vehicle_cost = vehicle_cost
    @addon_cost = addon_cost
    @total_cost = vehicle_cost + addon_cost
    @status = ReservationStatus::ACTIVE
  end

  def set_status(status)
    @status = status
  end

  def overlaps_with?(s, e)
    @status == ReservationStatus::ACTIVE &&
      RentalDate.overlaps?(@start_date, @end_date, s, e)
  end

  def to_s
    days = @start_date.days_until(@end_date)
    lines = [
      "Reservation #{@reservation_id} [#{@status}]",
      "  Vehicle: #{@vehicle_id}",
      "  Period: #{@start_date} to #{@end_date} (#{days} days)",
      "  Pickup: #{@pickup_location} | Return: #{@return_location}",
      "  Vehicle cost: $#{'%.2f' % @vehicle_cost}"
    ]
    unless @add_ons.empty?
      lines << "  Add-ons:"
      @add_ons.each { |a| lines << "    #{a.name}: $#{a.daily_rate}/day" }
      lines << "  Add-on cost: $#{'%.2f' % @addon_cost}"
    end
    lines << "  Total: $#{'%.2f' % @total_cost}"
    lines.join("\n")
  end
end

# ─── Rental System (Facade) ─────────────────────────────

class RentalSystem
  include MonitorMixin

  def initialize
    super
    @vehicles = {}
    @customers = {}
    @reservations = []
    @pricing = DailyPricing.new
  end

  def set_pricing(strategy)
    synchronize do
      @pricing = strategy
      puts "Pricing strategy set to: #{@pricing.name}"
    end
  end

  def add_vehicle(vehicle)
    synchronize do
      @vehicles[vehicle.vehicle_id] = vehicle
      puts "Vehicle added: #{vehicle}"
    end
  end

  def add_customer(customer)
    synchronize do
      @customers[customer.customer_id] = customer
      puts "Customer registered: #{customer.name}"
    end
  end

  # ─── Search Available Vehicles ─────────────────────────

  def search_available(type, start_date, end_date, location = "")
    synchronize do
      @vehicles.values.select do |vehicle|
        next false unless vehicle.type == type
        next false if !location.empty? && vehicle.location != location

        # Check if vehicle is available for the date range
        @reservations.none? do |res|
          res.vehicle_id == vehicle.vehicle_id &&
            res.overlaps_with?(start_date, end_date)
        end
      end
    end
  end

  # ─── Make Reservation ──────────────────────────────────

  def make_reservation(customer_id, vehicle_id, start_date, end_date,
                       pickup_location, return_location,
                       addons = AddOnBuilder.new)
    synchronize do
      unless @customers.key?(customer_id)
        puts "Customer not found."
        return nil
      end

      vehicle = @vehicles[vehicle_id]
      unless vehicle
        puts "Vehicle not found."
        return nil
      end

      # Check availability
      conflict = @reservations.any? do |res|
        res.vehicle_id == vehicle_id && res.overlaps_with?(start_date, end_date)
      end

      if conflict
        puts "❌ Vehicle is not available for the requested dates."
        return nil
      end

      # Calculate pricing
      days = start_date.days_until(end_date)
      if days <= 0
        puts "Invalid date range."
        return nil
      end

      vehicle_cost = @pricing.calculate(vehicle.type, days)
      addon_cost = addons.daily_total * days

      # Create reservation
      reservation = RentalReservation.new(
        customer_id, vehicle_id, start_date, end_date,
        pickup_location, return_location,
        addons.build, vehicle_cost, addon_cost
      )

      puts "✅ Reservation created!"
      puts reservation

      @reservations << reservation
      reservation.reservation_id
    end
  end

  # ─── Cancel Reservation ────────────────────────────────

  def cancel_reservation(reservation_id)
    synchronize do
      res = @reservations.find { |r| r.reservation_id == reservation_id }
      unless res
        puts "Reservation not found."
        return false
      end

      unless res.status == ReservationStatus::ACTIVE
        puts "Reservation is not active."
        return false
      end

      res.set_status(ReservationStatus::CANCELLED)
      puts "🚫 Reservation #{reservation_id} cancelled."
      true
    end
  end

  # ─── Complete Reservation (Return Vehicle) ─────────────

  def complete_reservation(reservation_id, actual_return_date)
    synchronize do
      res = @reservations.find { |r| r.reservation_id == reservation_id }
      unless res
        puts "Reservation not found."
        return -1
      end

      unless res.status == ReservationStatus::ACTIVE
        puts "Reservation is not active."
        return -1
      end

      total = res.total_cost

      # Check for late return
      if res.end_date < actual_return_date
        late_days = res.end_date.days_until(actual_return_date)
        vehicle = @vehicles[res.vehicle_id]
        late_fee = @pricing.calculate(vehicle.type, late_days) * 1.5
        total += late_fee
        puts "⚠️ Late return by #{late_days} days." \
             " Late fee: $#{'%.2f' % late_fee}"
      end

      res.set_status(ReservationStatus::COMPLETED)
      puts "✅ Reservation #{reservation_id} completed." \
           " Total charge: $#{'%.2f' % total}"
      total
    end
  end

  # ─── Display ───────────────────────────────────────────

  def display_fleet
    synchronize do
      puts "\n=== Vehicle Fleet ==="
      @vehicles.each_value { |v| puts "  #{v}" }
      puts "=====================\n"
    end
  end
end

# ─── Usage ───────────────────────────────────────────────

system = RentalSystem.new

# Add vehicles
system.add_vehicle(RentalVehicle.new("V1", "Toyota", "Camry", 2024,
                                     VehicleType::SEDAN, "ABC-1234", "Downtown"))
system.add_vehicle(RentalVehicle.new("V2", "Honda", "CR-V", 2024,
                                     VehicleType::SUV, "DEF-5678", "Downtown"))
system.add_vehicle(RentalVehicle.new("V3", "BMW", "7 Series", 2024,
                                     VehicleType::LUXURY, "GHI-9012", "Airport"))
system.add_vehicle(RentalVehicle.new("V4", "Hyundai", "i20", 2024,
                                     VehicleType::HATCHBACK, "JKL-3456", "Downtown"))

# Add customers
system.add_customer(RentalCustomer.new("C1", "Alice", "alice@example.com", "DL-001"))
system.add_customer(RentalCustomer.new("C2", "Bob", "bob@example.com", "DL-002"))

system.display_fleet

# Search available SUVs
puts "--- Search: SUVs available Dec 20-25 ---"
start_date = RentalDate.new(2025, 12, 20)
end_date = RentalDate.new(2025, 12, 25)
available = system.search_available(VehicleType::SUV, start_date, end_date, "Downtown")
available.each { |v| puts "  #{v}" }

# Make reservation with add-ons
puts "\n--- Alice rents SUV ---"
res1 = system.make_reservation(
  "C1", "V2", start_date, end_date, "Downtown", "Airport",
  AddOnBuilder.new.with_insurance.with_gps
)

# Try to book same vehicle for overlapping dates (should fail)
puts "\n--- Bob tries same SUV ---"
bob_start = RentalDate.new(2025, 12, 23)
bob_end = RentalDate.new(2025, 12, 28)
system.make_reservation("C2", "V2", bob_start, bob_end, "Downtown", "Downtown")

# Bob books a different vehicle
puts "\n--- Bob rents Luxury ---"
system.set_pricing(WeeklyPricing.new)
res2 = system.make_reservation(
  "C2", "V3", bob_start, bob_end, "Airport", "Airport",
  AddOnBuilder.new.with_insurance.with_roadside_assistance
)

# Complete reservation (on time)
puts "\n--- Alice returns vehicle ---"
system.complete_reservation(res1, end_date) if res1

# Complete reservation (late)
puts "\n--- Bob returns late ---"
late_return = RentalDate.new(2025, 12, 30)
system.complete_reservation(res2, late_return) if res2
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `RentalPricingStrategy` (Daily, Weekly) | Swap pricing models without changing reservation logic |
| **Builder** | `AddOnBuilder` | Fluent construction of optional add-on services |
| **Factory** | Reservation ID generation | Automatic unique ID creation |
| **Facade** | `RentalSystem` | Simple interface hiding complex vehicle, customer, reservation, and pricing interactions |

### Key Design Decisions

1. **Date-range availability** — availability is checked by scanning existing reservations for overlapping date ranges, preventing double-booking. Ruby's `Comparable` module makes date comparisons clean
2. **Builder for add-ons** — the fluent builder pattern makes it easy to compose optional services without constructor explosion. Ruby's method chaining (`with_insurance.with_gps`) reads naturally
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

```ruby
require 'monitor'

# ─── Enums ───────────────────────────────────────────────

module RoomType
  STANDARD      = :standard
  DELUXE        = :deluxe
  SUITE         = :suite
  PRESIDENTIAL  = :presidential
end

module RoomStatus
  AVAILABLE   = :available
  RESERVED    = :reserved
  OCCUPIED    = :occupied
  MAINTENANCE = :maintenance
end

module HotelReservationStatus
  CONFIRMED   = :confirmed
  CHECKED_IN  = :checked_in
  CHECKED_OUT = :checked_out
  CANCELLED   = :cancelled
end

# ─── Date (simplified) ──────────────────────────────────

class BookingDate
  attr_reader :year, :month, :day

  def initialize(year, month, day)
    @year = year
    @month = month
    @day = day
  end

  def <=>(other)
    [year, month, day] <=> [other.year, other.month, other.day]
  end

  include Comparable

  def days_until(other)
    (other.year - year) * 365 + (other.month - month) * 30 + (other.day - day)
  end

  def to_s
    format('%04d-%02d-%02d', year, month, day)
  end

  def self.overlaps?(s1, e1, s2, e2)
    s1 < e2 && s2 < e1
  end
end

# ─── Room ────────────────────────────────────────────────

class HotelRoom
  attr_reader :room_number, :type, :floor, :capacity, :amenities

  def initialize(number, type, floor, capacity, amenities)
    @room_number = number
    @type = type
    @floor = floor
    @capacity = capacity
    @amenities = amenities
  end

  def to_s
    "Room #{@room_number} [#{@type}] Floor #{@floor}" \
      " | Capacity: #{@capacity}" \
      " | Amenities: #{@amenities.join(', ')}"
  end
end

# ─── Guest ───────────────────────────────────────────────

class HotelGuest
  attr_reader :guest_id, :name, :email, :phone

  def initialize(id, name, email, phone)
    @guest_id = id
    @name = name
    @email = email
    @phone = phone
  end
end

# ─── Charge (for invoice) ────────────────────────────────

Charge = Struct.new(:description, :amount)

# ─── Pricing Strategy (Strategy Pattern) ─────────────────

class HotelPricingStrategy
  def get_nightly_rate(_type)
    raise NotImplementedError
  end

  def name
    raise NotImplementedError
  end
end

class StandardHotelPricing < HotelPricingStrategy
  RATES = {
    RoomType::STANDARD     => 100.0,
    RoomType::DELUXE       => 180.0,
    RoomType::SUITE        => 350.0,
    RoomType::PRESIDENTIAL => 800.0
  }.freeze

  def get_nightly_rate(type)
    RATES[type] || 0
  end

  def name
    "Standard"
  end
end

class SeasonalHotelPricing < HotelPricingStrategy
  def initialize(multiplier = 1.5)
    @multiplier = multiplier
    @base = StandardHotelPricing.new
  end

  def get_nightly_rate(type)
    @base.get_nightly_rate(type) * @multiplier
  end

  def name
    "Seasonal (#{@multiplier}x)"
  end
end

# ─── Reservation ─────────────────────────────────────────

class HotelReservation
  @@next_id = 1

  attr_reader :reservation_id, :guest_id, :room_numbers,
              :check_in_date, :check_out_date, :num_guests,
              :room_charge, :status

  def initialize(guest_id, room_numbers, check_in, check_out,
                 num_guests, room_charge)
    @reservation_id = "HRS#{@@next_id}"
    @@next_id += 1
    @guest_id = guest_id
    @room_numbers = room_numbers
    @check_in_date = check_in
    @check_out_date = check_out
    @num_guests = num_guests
    @room_charge = room_charge
    @additional_charges = []
    @status = HotelReservationStatus::CONFIRMED
  end

  def set_status(status)
    @status = status
  end

  def add_charge(description, amount)
    @additional_charges << Charge.new(description, amount)
  end

  def total
    @room_charge + @additional_charges.sum(&:amount)
  end

  def overlaps_with?(s, e)
    (@status == HotelReservationStatus::CONFIRMED ||
     @status == HotelReservationStatus::CHECKED_IN) &&
      BookingDate.overlaps?(@check_in_date, @check_out_date, s, e)
  end

  def print_invoice
    nights = @check_in_date.days_until(@check_out_date)
    puts
    puts "╔══════════════════════════════════════╗"
    puts "║          HOTEL INVOICE               ║"
    puts "╠══════════════════════════════════════╣"
    puts "  Reservation: #{@reservation_id}"
    puts "  Guest: #{@guest_id}"
    puts "  Rooms: #{@room_numbers.join(' ')}"
    puts "  Check-in:  #{@check_in_date}"
    puts "  Check-out: #{@check_out_date}"
    puts "  Nights: #{nights}"
    puts "╠══════════════════════════════════════╣"
    puts "  Room charges: $#{'%.2f' % @room_charge}"
    unless @additional_charges.empty?
      puts "  Additional charges:"
      @additional_charges.each do |c|
        puts "    #{c.description}: $#{'%.2f' % c.amount}"
      end
    end
    puts "╠══════════════════════════════════════╣"
    puts "  TOTAL: $#{'%.2f' % total}"
    puts "╚══════════════════════════════════════╝"
    puts
  end
end

# ─── Hotel (Facade) ─────────────────────────────────────

class Hotel
  include MonitorMixin

  def initialize(name)
    super()
    @name = name
    @rooms = {}
    @guests = {}
    @reservations = []
    @pricing = StandardHotelPricing.new
  end

  def set_pricing(strategy)
    synchronize do
      @pricing = strategy
      puts "Pricing set to: #{@pricing.name}"
    end
  end

  def add_room(room)
    synchronize { @rooms[room.room_number] = room }
  end

  def add_guest(guest)
    synchronize { @guests[guest.guest_id] = guest }
  end

  # ─── Search Available Rooms ────────────────────────────

  def search_available(type, check_in, check_out, num_guests = 1)
    synchronize do
      @rooms.values.select do |room|
        next false unless room.type == type
        next false if room.capacity < num_guests

        @reservations.none? do |res|
          res.room_numbers.include?(room.room_number) &&
            res.overlaps_with?(check_in, check_out)
        end
      end
    end
  end

  # ─── Make Reservation ──────────────────────────────────

  def make_reservation(guest_id, room_numbers, check_in, check_out, num_guests)
    synchronize do
      unless @guests.key?(guest_id)
        puts "Guest not found."
        return nil
      end

      nights = check_in.days_until(check_out)
      if nights <= 0
        puts "Invalid date range."
        return nil
      end

      # Validate all rooms are available
      room_numbers.each do |rn|
        unless @rooms.key?(rn)
          puts "Room #{rn} not found."
          return nil
        end

        conflict = @reservations.any? do |res|
          res.room_numbers.include?(rn) && res.overlaps_with?(check_in, check_out)
        end

        if conflict
          puts "❌ Room #{rn} is not available for the requested dates."
          return nil
        end
      end

      # Calculate room charge
      total_room_charge = room_numbers.sum do |rn|
        room = @rooms[rn]
        @pricing.get_nightly_rate(room.type) * nights
      end

      reservation = HotelReservation.new(
        guest_id, room_numbers, check_in, check_out,
        num_guests, total_room_charge
      )

      puts "✅ Reservation #{reservation.reservation_id} confirmed!"
      puts "  Guest: #{@guests[guest_id].name}"
      print "  Rooms: "
      room_numbers.each { |rn| print "#{rn} [#{@rooms[rn].type}] " }
      puts
      puts "  #{check_in} to #{check_out} (#{nights} nights)"
      puts "  Room charge: $#{'%.2f' % total_room_charge}"

      @reservations << reservation
      reservation.reservation_id
    end
  end

  # ─── Check In ──────────────────────────────────────────

  def check_in(reservation_id)
    synchronize do
      res = @reservations.find { |r| r.reservation_id == reservation_id }
      unless res
        puts "Reservation not found."
        return false
      end

      unless res.status == HotelReservationStatus::CONFIRMED
        puts "Cannot check in: reservation is #{res.status}"
        return false
      end

      res.set_status(HotelReservationStatus::CHECKED_IN)
      puts "🔑 Checked in! Reservation #{reservation_id}"
      puts "  Rooms: #{res.room_numbers.join(' ')}"
      true
    end
  end

  # ─── Add Service Charge ────────────────────────────────

  def add_service_charge(reservation_id, service, amount)
    synchronize do
      res = @reservations.find { |r| r.reservation_id == reservation_id }
      unless res
        return false
      end

      unless res.status == HotelReservationStatus::CHECKED_IN
        puts "Can only add charges to checked-in reservations."
        return false
      end

      res.add_charge(service, amount)
      puts "  🛎️ Added charge: #{service} ($#{'%.2f' % amount})"
      true
    end
  end

  # ─── Check Out ─────────────────────────────────────────

  def check_out(reservation_id)
    synchronize do
      res = @reservations.find { |r| r.reservation_id == reservation_id }
      unless res
        puts "Reservation not found."
        return -1
      end

      unless res.status == HotelReservationStatus::CHECKED_IN
        puts "Cannot check out: reservation is #{res.status}"
        return -1
      end

      res.set_status(HotelReservationStatus::CHECKED_OUT)
      puts "👋 Checked out! Reservation #{reservation_id}"
      res.print_invoice
      res.total
    end
  end

  # ─── Cancel Reservation ────────────────────────────────

  def cancel_reservation(reservation_id)
    synchronize do
      res = @reservations.find { |r| r.reservation_id == reservation_id }
      unless res
        return false
      end

      unless res.status == HotelReservationStatus::CONFIRMED
        puts "Can only cancel confirmed reservations."
        return false
      end

      res.set_status(HotelReservationStatus::CANCELLED)
      puts "🚫 Reservation #{reservation_id} cancelled."
      true
    end
  end

  # ─── Display ───────────────────────────────────────────

  def display_rooms
    synchronize do
      puts "\n=== #{@name} — Rooms ==="
      @rooms.each_value { |room| puts "  #{room}" }
      puts "==============================\n"
    end
  end
end

# ─── Usage ───────────────────────────────────────────────

hotel = Hotel.new("Grand Palace Hotel")

# Add rooms
hotel.add_room(HotelRoom.new("101", RoomType::STANDARD, 1, 2,
                              %w[WiFi TV AC]))
hotel.add_room(HotelRoom.new("102", RoomType::STANDARD, 1, 2,
                              %w[WiFi TV AC]))
hotel.add_room(HotelRoom.new("201", RoomType::DELUXE, 2, 3,
                              %w[WiFi TV AC Minibar Balcony]))
hotel.add_room(HotelRoom.new("301", RoomType::SUITE, 3, 4,
                              ["WiFi", "TV", "AC", "Minibar", "Balcony",
                               "Jacuzzi", "Living Room"]))
hotel.add_room(HotelRoom.new("401", RoomType::PRESIDENTIAL, 4, 6,
                              ["WiFi", "TV", "AC", "Minibar", "Balcony",
                               "Jacuzzi", "Living Room", "Dining Room",
                               "Butler Service"]))

# Add guests
hotel.add_guest(HotelGuest.new("G1", "Alice Johnson", "alice@example.com", "+1-555-0101"))
hotel.add_guest(HotelGuest.new("G2", "Bob Smith", "bob@example.com", "+1-555-0202"))

hotel.display_rooms

# Search available suites
check_in = BookingDate.new(2025, 7, 15)
check_out = BookingDate.new(2025, 7, 18)

puts "--- Search: Deluxe rooms, July 15-18 ---"
available = hotel.search_available(RoomType::DELUXE, check_in, check_out, 2)
available.each { |room| puts "  #{room}" }

# Alice books a Deluxe room
puts "\n--- Alice books Deluxe ---"
res1 = hotel.make_reservation("G1", ["201"], check_in, check_out, 2)

# Bob tries same room (should fail)
puts "\n--- Bob tries same room ---"
hotel.make_reservation("G2", ["201"],
                       BookingDate.new(2025, 7, 16), BookingDate.new(2025, 7, 20), 2)

# Bob books Standard room
puts "\n--- Bob books Standard ---"
res2 = hotel.make_reservation("G2", ["101"],
                              BookingDate.new(2025, 7, 16),
                              BookingDate.new(2025, 7, 20), 2)

# Alice checks in
puts "\n--- Alice checks in ---"
hotel.check_in(res1) if res1

# Alice orders room service
puts "\n--- Alice orders services ---"
if res1
  hotel.add_service_charge(res1, "Room Service - Dinner", 45.00)
  hotel.add_service_charge(res1, "Laundry", 25.00)
  hotel.add_service_charge(res1, "Minibar", 30.00)
end

# Alice checks out
puts "\n--- Alice checks out ---"
hotel.check_out(res1) if res1

# Switch to seasonal pricing
puts "\n--- Switch to seasonal pricing ---"
hotel.set_pricing(SeasonalHotelPricing.new(1.5))

# New booking with seasonal pricing
res3 = hotel.make_reservation("G1", ["301"],
                              BookingDate.new(2025, 12, 24),
                              BookingDate.new(2025, 12, 27), 3)
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

1. **Room vs Reservation separation** — `HotelRoom` defines the physical room (number, type, amenities); `HotelReservation` tracks the booking (dates, guest, charges). A room can have multiple non-overlapping reservations
2. **Date-range overlap checking** — reservations are checked for date overlap to prevent double-booking. Only CONFIRMED and CHECKED_IN reservations block availability. Ruby's `Comparable` module makes date comparisons clean and readable
3. **Incremental charges** — additional services (room service, laundry, minibar) are added as charges during the stay and included in the final invoice at checkout. Ruby's `Struct` makes `Charge` lightweight
4. **Seasonal pricing** — the pricing strategy can be swapped at runtime, allowing the hotel to adjust rates for peak seasons, holidays, or special events. The `SeasonalHotelPricing` delegates to `StandardHotelPricing` and applies a multiplier
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

2. **"How do you handle concurrent booking/reservation?"** Use a two-phase approach: (a) **Lock** — temporarily reserve the resource (seat, room, vehicle) with a timeout. (b) **Confirm** — permanently book after payment succeeds. If payment fails or times out, release the lock. Use `MonitorMixin` with `synchronize` to protect shared state. Ensure atomic operations — either all seats are locked or none are (all-or-nothing).

3. **"How do you choose which design patterns to use?"** Match the pattern to the problem: **Strategy** when you have multiple interchangeable algorithms (pricing, scheduling, splitting). **Observer** when one event should notify multiple listeners (book available, order status change). **State** when behavior changes based on internal state (elevator direction, vending machine). **Builder** when constructing complex objects with many optional parts (add-ons, configurations). **Facade** when you need a simple interface over a complex subsystem.

4. **"How do you handle pricing that changes?"** Use the Strategy pattern. Define a pricing strategy class with a `calculate` method. Implement concrete strategies (daily, weekly, seasonal, promotional). The system holds a reference to the current strategy and can swap it at runtime. This follows the Open/Closed Principle — new pricing models are added without modifying existing code.

5. **"How do you prevent double-booking?"** For time-based resources (rooms, vehicles), check for date-range overlap before confirming a reservation. Two ranges `[s1, e1)` and `[s2, e2)` overlap if `s1 < e2 AND s2 < e1`. For seat-based resources (movie tickets), use a lock-then-book approach with expiring locks. Always use `MonitorMixin#synchronize` to ensure atomicity of the check-and-reserve operation.

6. **"How do you simplify debts in a Splitwise-like system?"** Calculate the net balance for each user (total owed minus total owed to them). Separate users into debtors (positive net) and creditors (negative net). Use a greedy algorithm to match the largest debtor with the largest creditor, settling the minimum of their balances. This minimizes the number of transactions needed.

7. **"How do you design an elevator scheduling algorithm?"** The LOOK algorithm is the most common: the elevator continues in its current direction, stopping at requested floors, until there are no more requests in that direction, then reverses. Maintain two sorted sets of stops (up and down). For multi-elevator systems, assign requests to the elevator with the shortest estimated travel distance, considering current direction and pending stops.

8. **"How do you handle inventory in an e-commerce system?"** Decrement inventory when an order is confirmed (not when added to cart). If an order is cancelled, restore the inventory. Validate stock for all items before processing any — if one item is out of stock, reject the entire order. Use `MonitorMixin` to prevent race conditions where two users order the last item simultaneously.

9. **"How do you design a notification system?"** Use the Observer pattern. Define an observer class with methods for each event type. Components that generate events (library, booking system) maintain a list of observers and notify them when events occur. This decouples the event source from the notification mechanism — you can add email, SMS, push, or webhook observers without changing the source. Ruby's duck typing makes this especially clean — any object responding to the right method works as an observer.

10. **"What's the difference between easy and medium LLD problems?"** Easy problems (parking lot, tic-tac-toe, vending machine) typically have a single main entity with straightforward state transitions. Medium problems (booking systems, e-commerce, expense sharing) involve multiple interacting entities, complex business rules, time-based logic, concurrent access, and require combining 3-5 design patterns. The key skill at the medium level is managing complexity through clean separation of concerns.

11. **"How do you handle search functionality in LLD?"** For in-memory systems, iterate over the collection and filter by criteria. Use case-insensitive string matching for text search (`downcase` + `include?`). For extensibility, consider the Strategy pattern for different search criteria. In a real system, you'd use database indexes or a search engine (Elasticsearch), but in an LLD interview, the focus is on the interface design and how search integrates with the rest of the system.

12. **"How do you generate invoices or bills?"** Track all charges incrementally during the lifecycle (room charges, service charges, late fees, discounts). Store charges as a list of `Struct` pairs (description, amount). At checkout/completion, sum all charges to produce the total. This approach is extensible — new charge types can be added without modifying the invoice logic. Use the Decorator pattern if discounts or taxes need to be applied on top of the base charges.

### Ruby-Specific Interview Notes

1. **Thread safety** — Ruby's `MonitorMixin` provides reentrant mutual exclusion via `synchronize`, equivalent to C++ `std::mutex` with `lock_guard`. For simpler cases, `Mutex` works too, but `MonitorMixin` integrates cleanly as a mixin
2. **Enums** — Ruby uses modules with symbol constants instead of C++ `enum class`. Symbols are immutable, interned strings that are perfect for representing fixed sets of values
3. **Duck typing** — Ruby doesn't need explicit interfaces. Any object that responds to the expected methods works. This makes Strategy and Observer patterns lighter — no need for abstract base classes, though they're useful for documentation
4. **Struct** — Ruby's `Struct` creates lightweight value objects with named attributes, replacing simple C++ structs. `Struct.new(:name, :amount)` generates a class with constructor, accessors, equality, and more
5. **Comparable** — including `Comparable` and defining `<=>` gives you `<`, `<=`, `>`, `>=`, and `between?` for free, making date comparisons clean
6. **Hash defaults** — `Hash.new(0.0)` and `Hash.new { |h, k| h[k] = {} }` provide clean default values, eliminating nil checks that would be needed with C++ `unordered_map`
7. **Blocks and iterators** — Ruby's `select`, `reject`, `map`, `find`, `any?`, `none?`, and `sum` replace verbose C++ loops with expressive one-liners, making search and filtering code concise
