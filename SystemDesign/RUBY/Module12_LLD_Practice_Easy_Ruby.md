# Module 12: LLD Practice Problems (Easy) — Ruby

> This module puts everything from Modules 1–11 into practice. Each problem follows a structured approach: understand the requirements, identify the key entities, define relationships, choose appropriate design patterns, and implement a clean, extensible solution in Ruby. These are the problems most commonly asked in LLD interviews at the entry level. For each problem, we provide a complete, working implementation with detailed explanations.

---

## 12.1 Parking Lot System

> Design a parking lot system that supports multiple vehicle types, different parking spot sizes, entry/exit gates, ticketing, and payment calculation. This is the single most asked LLD interview question.

---

### Requirements

**Functional Requirements:**
1. The parking lot has multiple floors, each with multiple parking spots
2. Parking spots come in three sizes: Small, Medium, Large
3. Vehicles come in three types: Bike, Car, Truck
4. A Bike can park in Small, Medium, or Large spots
5. A Car can park in Medium or Large spots
6. A Truck can only park in Large spots
7. The system should assign the smallest available spot that fits the vehicle
8. Entry gates issue tickets with entry time
9. Exit gates calculate payment based on duration
10. The system tracks which spots are occupied

**Non-Functional Requirements:**
- Thread-safe for concurrent entry/exit
- Extensible for new vehicle types or spot sizes

---

### Key Entities & Class Design

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  ParkingLot  │────▶│    Floor     │────▶│ ParkingSpot  │
│  (Singleton) │     │              │     │              │
└─────────────┘     └──────────────┘     └─────────────┘
       │                                        ▲
       │                                        │ parks in
       ▼                                        │
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ EntryGate   │────▶│   Ticket     │◀────│   Vehicle    │
└─────────────┘     └──────────────┘     └─────────────┘
       │                    │
       │                    ▼
┌─────────────┐     ┌──────────────┐
│  ExitGate   │────▶│  Payment     │
└─────────────┘     │  Strategy    │
                    └──────────────┘
```

---

### Implementation

```ruby
require 'monitor'

# ─── Enums (Ruby uses symbols or constants) ───────────────

module VehicleType
  BIKE  = :bike
  CAR   = :car
  TRUCK = :truck
end

module SpotSize
  SMALL  = :small
  MEDIUM = :medium
  LARGE  = :large

  # Ordering for comparison — smaller index = smaller spot
  ORDER = { SMALL => 0, MEDIUM => 1, LARGE => 2 }.freeze

  def self.fits?(spot_size, min_size)
    ORDER[spot_size] >= ORDER[min_size]
  end
end

# ─── Vehicle ─────────────────────────────────────────────

class Vehicle
  attr_reader :license_plate, :type

  def initialize(plate, type)
    @license_plate = plate
    @type = type
  end

  # Determine the minimum spot size this vehicle needs
  def min_spot_size
    case @type
    when VehicleType::BIKE  then SpotSize::SMALL
    when VehicleType::CAR   then SpotSize::MEDIUM
    when VehicleType::TRUCK then SpotSize::LARGE
    end
  end

  def fits_in?(spot_size)
    SpotSize.fits?(spot_size, min_spot_size)
  end
end

# ─── Parking Spot ────────────────────────────────────────

class ParkingSpot
  attr_reader :spot_id, :floor_number, :size, :parked_vehicle

  def initialize(id, floor, size)
    @spot_id = id
    @floor_number = floor
    @size = size
    @parked_vehicle = nil
  end

  def available?
    @parked_vehicle.nil?
  end

  def can_fit?(vehicle)
    available? && vehicle.fits_in?(@size)
  end

  def park(vehicle)
    @parked_vehicle = vehicle
  end

  def vacate
    @parked_vehicle = nil
  end
end

# ─── Ticket ──────────────────────────────────────────────

class Ticket
  @@next_id = 1

  attr_reader :ticket_id, :vehicle_plate, :spot_id, :floor_number, :entry_time

  def initialize(plate, spot_id, floor)
    @ticket_id = @@next_id
    @@next_id += 1
    @vehicle_plate = plate
    @spot_id = spot_id
    @floor_number = floor
    @entry_time = Time.now
  end

  # Duration in hours (for payment calculation)
  def duration_hours
    elapsed_seconds = Time.now - @entry_time
    elapsed_seconds / 3600.0
  end

  def to_s
    "Ticket ##{@ticket_id} | Vehicle: #{@vehicle_plate} " \
      "| Floor: #{@floor_number} | Spot: #{@spot_id}"
  end
end

# ─── Payment Strategy (Strategy Pattern) ─────────────────

class PaymentStrategy
  def calculate(_vehicle_type, _hours)
    raise NotImplementedError, "Subclasses must implement #calculate"
  end

  def name
    raise NotImplementedError, "Subclasses must implement #name"
  end
end

# Flat hourly rate based on vehicle type
class HourlyPayment < PaymentStrategy
  RATES = {
    VehicleType::BIKE  => 10.0,
    VehicleType::CAR   => 20.0,
    VehicleType::TRUCK => 30.0
  }.freeze

  def calculate(vehicle_type, hours)
    rate = RATES[vehicle_type]
    billable_hours = [1.0, hours.ceil].max  # minimum 1 hour
    rate * billable_hours
  end

  def name
    "Hourly"
  end
end

# Tiered pricing: first 2 hours flat, then per-hour after
class TieredPayment < PaymentStrategy
  TIERS = {
    VehicleType::BIKE  => { base: 15, hourly: 5 },
    VehicleType::CAR   => { base: 30, hourly: 10 },
    VehicleType::TRUCK => { base: 50, hourly: 20 }
  }.freeze

  def calculate(vehicle_type, hours)
    tier = TIERS[vehicle_type]
    return tier[:base] if hours <= 2.0

    tier[:base] + tier[:hourly] * (hours - 2.0).ceil
  end

  def name
    "Tiered"
  end
end

# ─── Floor ───────────────────────────────────────────────

class ParkingFloor
  attr_reader :floor_number

  def initialize(number, small_count, medium_count, large_count)
    @floor_number = number
    @spots = []
    id = 1

    small_count.times  { @spots << ParkingSpot.new(id, number, SpotSize::SMALL);  id += 1 }
    medium_count.times { @spots << ParkingSpot.new(id, number, SpotSize::MEDIUM); id += 1 }
    large_count.times  { @spots << ParkingSpot.new(id, number, SpotSize::LARGE);  id += 1 }
  end

  # Find the smallest available spot that fits the vehicle
  def find_spot(vehicle)
    best = nil
    @spots.each do |spot|
      if spot.can_fit?(vehicle)
        if best.nil? || SpotSize::ORDER[spot.size] < SpotSize::ORDER[best.size]
          best = spot
        end
      end
    end
    best
  end

  def find_spot_by_id(id)
    @spots.find { |spot| spot.spot_id == id }
  end

  def available_count(size)
    @spots.count { |spot| spot.size == size && spot.available? }
  end

  def total_count(size)
    @spots.count { |spot| spot.size == size }
  end
end

# ─── Parking Lot (Singleton) ─────────────────────────────

class ParkingLot
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
    @floors = []
    @active_tickets = {}  # plate => ticket
    @payment_strategy = HourlyPayment.new
  end

  def initialize_lot(lot_name, floor_configs)
    synchronize do
      @name = lot_name
      @floors = []
      @active_tickets = {}

      floor_configs.each_with_index do |config, index|
        small, medium, large = config
        @floors << ParkingFloor.new(index + 1, small, medium, large)
      end
    end
  end

  def set_payment_strategy(strategy)
    synchronize do
      @payment_strategy = strategy
    end
  end

  # ─── Entry Gate Logic ────────────────────────────────

  def enter(vehicle)
    synchronize do
      # Check if vehicle is already parked
      if @active_tickets.key?(vehicle.license_plate)
        puts "Vehicle #{vehicle.license_plate} is already parked!"
        return nil
      end

      # Find the best spot across all floors
      @floors.each do |floor|
        spot = floor.find_spot(vehicle)
        if spot
          spot.park(vehicle)
          ticket = Ticket.new(vehicle.license_plate, spot.spot_id, floor.floor_number)
          @active_tickets[vehicle.license_plate] = ticket

          puts "Vehicle #{vehicle.license_plate} (#{vehicle.type})" \
               " parked at Floor #{floor.floor_number}," \
               " Spot #{spot.spot_id} (#{spot.size})"

          return ticket
        end
      end

      puts "No available spot for #{vehicle.type} #{vehicle.license_plate}"
      nil
    end
  end

  # ─── Exit Gate Logic ─────────────────────────────────

  def exit_vehicle(plate)
    synchronize do
      ticket = @active_tickets[plate]
      unless ticket
        puts "No active ticket for vehicle #{plate}"
        return nil
      end

      hours = ticket.duration_hours

      # Find and vacate the spot
      @floors.each do |floor|
        next unless floor.floor_number == ticket.floor_number

        spot = floor.find_spot_by_id(ticket.spot_id)
        if spot
          vtype = spot.parked_vehicle.type
          amount = @payment_strategy.calculate(vtype, hours)

          spot.vacate
          @active_tickets.delete(plate)

          puts "Vehicle #{plate} exited." \
               " Duration: #{'%.2f' % hours} hrs" \
               " | Payment (#{@payment_strategy.name}): $#{'%.2f' % amount}"

          return amount
        end
      end

      nil
    end
  end

  # ─── Display Status ──────────────────────────────────

  def display_status
    synchronize do
      puts "\n=== #{@name} Status ==="
      @floors.each do |floor|
        puts "Floor #{floor.floor_number}: " \
             "Small(#{floor.available_count(SpotSize::SMALL)}" \
             "/#{floor.total_count(SpotSize::SMALL)}) " \
             "Medium(#{floor.available_count(SpotSize::MEDIUM)}" \
             "/#{floor.total_count(SpotSize::MEDIUM)}) " \
             "Large(#{floor.available_count(SpotSize::LARGE)}" \
             "/#{floor.total_count(SpotSize::LARGE)})"
      end
      puts "Active vehicles: #{@active_tickets.size}"
      puts "========================\n"
    end
  end
end

# ─── Usage ───────────────────────────────────────────────

lot = ParkingLot.instance

# 2 floors: Floor 1 (5 small, 5 medium, 2 large), Floor 2 (3 small, 5 medium, 3 large)
lot.initialize_lot("Downtown Parking", [[5, 5, 2], [3, 5, 3]])
lot.display_status

# Vehicles arrive
bike1  = Vehicle.new("BIKE-001",  VehicleType::BIKE)
car1   = Vehicle.new("CAR-001",   VehicleType::CAR)
car2   = Vehicle.new("CAR-002",   VehicleType::CAR)
truck1 = Vehicle.new("TRUCK-001", VehicleType::TRUCK)

lot.enter(bike1)    # parks in Small spot
lot.enter(car1)     # parks in Medium spot
lot.enter(car2)     # parks in Medium spot
lot.enter(truck1)   # parks in Large spot

lot.display_status

# Vehicles exit
lot.exit_vehicle("BIKE-001")
lot.exit_vehicle("CAR-001")

lot.display_status

# Switch to tiered payment
lot.set_payment_strategy(TieredPayment.new)
lot.exit_vehicle("TRUCK-001")
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Singleton** | `ParkingLot` | One parking lot instance, global access |
| **Strategy** | `PaymentStrategy` | Swap payment algorithms (hourly, tiered) without changing lot logic |
| **Factory** | `Vehicle#min_spot_size` | Maps vehicle type to required spot size |

### Key Design Decisions

1. **Smallest-fit allocation** — a Bike gets a Small spot before a Medium, reducing waste
2. **Thread safety** — `MonitorMixin` protects all shared state for concurrent entry/exit. Ruby's `MonitorMixin` provides reentrant mutual exclusion via `synchronize`, similar to C++ `std::mutex` with `lock_guard`
3. **Separation of concerns** — Vehicle doesn't know about spots; ParkingSpot doesn't know about payment
4. **Open/Closed** — new vehicle types or payment strategies can be added without modifying existing code
5. **Symbols as enums** — Ruby idiomatically uses symbols (`:bike`, `:small`) instead of C++ enum classes. A module with constants groups them for clarity

---


## 12.2 Tic-Tac-Toe

> Design a Tic-Tac-Toe game that supports two players, validates moves, detects win/draw conditions, and is extensible to an NxN board.

---

### Requirements

**Functional Requirements:**
1. Two players take turns placing their symbol (X or O) on a grid
2. Default board is 3x3, but should support NxN
3. A player wins by completing a full row, column, or diagonal
4. The game detects draws (board full, no winner)
5. Invalid moves (occupied cell, out of bounds) are rejected
6. Players can be human or computer (extensible)

---

### Implementation

```ruby
# ─── Symbols ─────────────────────────────────────────────

module Symbol
  NONE = :none
  X    = :x
  O    = :o

  def self.to_char(sym)
    case sym
    when X then 'X'
    when O then 'O'
    else '.'
    end
  end
end

module GameState
  IN_PROGRESS = :in_progress
  X_WINS      = :x_wins
  O_WINS      = :o_wins
  DRAW        = :draw
end

# ─── Board ───────────────────────────────────────────────

class Board
  attr_reader :size

  def initialize(n = 3)
    @size = n
    @grid = Array.new(n) { Array.new(n, Symbol::NONE) }
    @moves_made = 0

    # Optimization: track row/col/diagonal sums for O(1) win check
    # +1 for X, -1 for O. If any sum reaches +N or -N, that player wins.
    @row_sums = Array.new(n, 0)
    @col_sums = Array.new(n, 0)
    @diag_sum = 0       # main diagonal (top-left to bottom-right)
    @anti_diag_sum = 0  # anti-diagonal (top-right to bottom-left)
  end

  def valid_move?(row, col)
    row.between?(0, @size - 1) &&
      col.between?(0, @size - 1) &&
      @grid[row][col] == Symbol::NONE
  end

  # Place a symbol and check if this move wins — O(1)
  def place(row, col, symbol)
    raise ArgumentError, "Invalid move: (#{row}, #{col})" unless valid_move?(row, col)

    @grid[row][col] = symbol
    @moves_made += 1

    val = symbol == Symbol::X ? 1 : -1
    @row_sums[row] += val
    @col_sums[col] += val
    @diag_sum += val if row == col
    @anti_diag_sum += val if row + col == @size - 1

    # Check if this move wins
    @row_sums[row].abs == @size ||
      @col_sums[col].abs == @size ||
      @diag_sum.abs == @size ||
      @anti_diag_sum.abs == @size
  end

  def full?
    @moves_made == @size * @size
  end

  def cell(row, col)
    @grid[row][col]
  end

  def print_board
    # Column headers
    print "  "
    @size.times { |c| print "#{c} " }
    puts

    @size.times do |r|
      print "#{r} "
      @size.times do |c|
        print Symbol.to_char(@grid[r][c])
        print "|" if c < @size - 1
      end
      puts
      if r < @size - 1
        print "  "
        @size.times do |c|
          print "-"
          print "+" if c < @size - 1
        end
        puts
      end
    end
    puts
  end
end

# ─── Player ──────────────────────────────────────────────

class Player
  attr_reader :name, :symbol

  def initialize(name, symbol)
    @name = name
    @symbol = symbol
  end

  # Get move from player — can be overridden for AI
  def get_move(_board)
    print "#{@name} (#{Symbol.to_char(@symbol)}), enter row col: "
    input = gets.chomp.split.map(&:to_i)
    [input[0], input[1]]
  end
end

# Simple AI player — picks first available spot
class SimpleAIPlayer < Player
  def get_move(board)
    board.size.times do |r|
      board.size.times do |c|
        if board.valid_move?(r, c)
          puts "#{name} plays: #{r} #{c}"
          return [r, c]
        end
      end
    end
    raise "No valid moves"
  end
end

# ─── Game ────────────────────────────────────────────────

class TicTacToeGame
  def initialize(board_size = 3)
    @board = Board.new(board_size)
    @players = []
    @current_player_index = 0
    @state = GameState::IN_PROGRESS
  end

  def add_player(player)
    raise "Maximum 2 players" if @players.size >= 2

    @players << player
  end

  attr_reader :state

  def play
    raise "Need exactly 2 players to start" unless @players.size == 2

    puts "=== Tic-Tac-Toe (#{@board.size}x#{@board.size}) ==="
    @board.print_board

    while @state == GameState::IN_PROGRESS
      current = @players[@current_player_index]

      # Get a valid move
      valid_move = false
      until valid_move
        begin
          row, col = current.get_move(@board)
          wins = @board.place(row, col, current.symbol)
          valid_move = true

          @board.print_board

          if wins
            @state = current.symbol == Symbol::X ? GameState::X_WINS : GameState::O_WINS
            puts "#{current.name} wins!"
          elsif @board.full?
            @state = GameState::DRAW
            puts "It's a draw!"
          end
        rescue ArgumentError
          puts "Invalid move. Try again."
        end
      end

      # Switch player
      @current_player_index = 1 - @current_player_index
    end
  end
end

# ─── Usage ───────────────────────────────────────────────

game = TicTacToeGame.new(3)  # 3x3 board

game.add_player(Player.new("Alice", Symbol::X))
game.add_player(SimpleAIPlayer.new("Bot", Symbol::O))

game.play
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `Player#get_move` (polymorphic) | Different player types (human, AI) provide moves differently |
| **State** | `GameState` + game loop | Game behavior changes based on state (in progress, won, draw) |

### Key Design Decisions

1. **O(1) win checking** — instead of scanning the entire board after each move, we track row/column/diagonal sums. A sum of +N or -N means a player completed that line
2. **NxN extensible** — board size is a constructor parameter, win logic works for any N
3. **Polymorphic players** — `Player` base class with overridable `get_move` allows human, AI, or network players. Ruby's duck typing makes this natural — any object with a `get_move(board)` method works
4. **Separation** — Board handles grid state, Game handles turn logic, Player handles input

---


## 12.3 Vending Machine

> Design a vending machine that manages product inventory, accepts coins/notes, handles state transitions (Idle → HasMoney → Dispensing), and returns change. This is a classic State pattern problem.

---

### Requirements

**Functional Requirements:**
1. The machine holds multiple products, each with a name, price, and quantity
2. Users insert coins/notes (accepted denominations: 1, 5, 10, 25, 50, 100)
3. Users select a product to purchase
4. The machine dispenses the product if sufficient money is inserted and the product is in stock
5. The machine returns change after dispensing
6. Users can cancel and get a full refund at any time
7. The machine transitions between states: Idle, HasMoney, Dispensing

---

### Implementation

```ruby
# ─── Product ─────────────────────────────────────────────

class Product
  attr_reader :name, :price, :quantity

  def initialize(name, price, quantity)
    @name = name
    @price = price
    @quantity = quantity
  end

  def available?
    @quantity > 0
  end

  def decrement
    @quantity -= 1 if @quantity > 0
  end

  def restock(amount)
    @quantity += amount
  end
end

# ─── State Interface (State Pattern) ─────────────────────

class VendingState
  def insert_money(_machine, _amount)
    raise NotImplementedError
  end

  def select_product(_machine, _code)
    raise NotImplementedError
  end

  def dispense(_machine)
    raise NotImplementedError
  end

  def cancel(_machine)
    raise NotImplementedError
  end

  def name
    raise NotImplementedError
  end
end

# ─── Vending Machine ─────────────────────────────────────

class VendingMachine
  attr_reader :inserted_money, :selected_product_code

  def initialize
    @products = {}          # code => Product
    @inserted_money = 0.0
    @selected_product_code = nil
    @current_state = IdleState.new
  end

  def add_product(code, name, price, quantity)
    @products[code] = Product.new(name, price, quantity)
  end

  # Public interface — delegates to current state
  def insert_money(amount)
    print "[#{@current_state.name}] "
    @current_state.insert_money(self, amount)
  end

  def select_product(code)
    print "[#{@current_state.name}] "
    @current_state.select_product(self, code)
  end

  def dispense
    print "[#{@current_state.name}] "
    @current_state.dispense(self)
  end

  def cancel
    print "[#{@current_state.name}] "
    @current_state.cancel(self)
  end

  def display_products
    puts "\n=== Available Products ==="
    @products.each do |code, product|
      puts "#{code}: #{product.name} - $#{'%.2f' % product.price} (#{product.quantity} left)"
    end
    puts "Inserted: $#{'%.2f' % @inserted_money}"
    puts "State: #{@current_state.name}"
    puts "==========================\n"
  end

  # ─── Internal helpers (used by state classes) ────────

  def set_state(state)
    @current_state = state
  end

  def add_money(amount)
    @inserted_money += amount
  end

  def return_money
    if @inserted_money > 0
      puts "Returning $#{'%.2f' % @inserted_money}"
      @inserted_money = 0
    end
  end

  def get_product(code)
    @products[code]
  end

  def set_selected(code)
    @selected_product_code = code
  end

  def reset
    @inserted_money = 0
    @selected_product_code = nil
  end
end

# ─── Concrete States ─────────────────────────────────────

class IdleState < VendingState
  def name
    "Idle"
  end

  def insert_money(machine, amount)
    machine.add_money(amount)
    puts "Inserted $#{'%.2f' % amount}. Total: $#{'%.2f' % machine.inserted_money}"
    machine.set_state(HasMoneyState.new)
  end

  def select_product(_machine, _code)
    puts "Please insert money first."
  end

  def dispense(_machine)
    puts "Please insert money and select a product."
  end

  def cancel(_machine)
    puts "Nothing to cancel."
  end
end

class HasMoneyState < VendingState
  def name
    "HasMoney"
  end

  def insert_money(machine, amount)
    machine.add_money(amount)
    puts "Inserted $#{'%.2f' % amount}. Total: $#{'%.2f' % machine.inserted_money}"
  end

  def select_product(machine, code)
    product = machine.get_product(code)

    unless product
      puts "Invalid product code: #{code}"
      return
    end

    unless product.available?
      puts "#{product.name} is out of stock."
      return
    end

    if machine.inserted_money < product.price
      puts "Insufficient funds. Need $#{'%.2f' % product.price}, " \
           "have $#{'%.2f' % machine.inserted_money}"
      return
    end

    machine.set_selected(code)
    puts "Selected: #{product.name}"
    machine.set_state(DispensingState.new)
    machine.dispense  # auto-dispense after selection
  end

  def dispense(_machine)
    puts "Please select a product first."
  end

  def cancel(machine)
    print "Transaction cancelled. "
    machine.return_money
    machine.reset
    machine.set_state(IdleState.new)
  end
end

class DispensingState < VendingState
  def name
    "Dispensing"
  end

  def insert_money(_machine, _amount)
    puts "Please wait, dispensing in progress."
  end

  def select_product(_machine, _code)
    puts "Please wait, dispensing in progress."
  end

  def dispense(machine)
    product = machine.get_product(machine.selected_product_code)
    unless product
      puts "Error: product not found."
      machine.return_money
      machine.reset
      machine.set_state(IdleState.new)
      return
    end

    # Dispense product
    product.decrement
    change = machine.inserted_money - product.price

    puts "Dispensing: #{product.name}"
    puts "Returning change: $#{'%.2f' % change}" if change > 0

    machine.reset
    machine.set_state(IdleState.new)
  end

  def cancel(_machine)
    puts "Cannot cancel during dispensing."
  end
end

# ─── Usage ───────────────────────────────────────────────

vm = VendingMachine.new

# Stock the machine
vm.add_product("A1", "Cola",  1.50, 10)
vm.add_product("A2", "Chips", 2.00, 5)
vm.add_product("B1", "Water", 1.00, 15)
vm.add_product("B2", "Candy", 0.75, 20)

vm.display_products

# Scenario 1: Successful purchase
puts "--- Scenario 1: Buy Cola ---"
vm.insert_money(1.00)
vm.insert_money(1.00)
vm.select_product("A1")  # Cola costs $1.50, inserted $2.00 → change $0.50

# Scenario 2: Insufficient funds
puts "\n--- Scenario 2: Insufficient funds ---"
vm.insert_money(0.50)
vm.select_product("A2")  # Chips costs $2.00, only $0.50

# Scenario 3: Cancel transaction
puts "\n--- Scenario 3: Cancel ---"
vm.cancel  # returns $0.50

# Scenario 4: Normal purchase
puts "\n--- Scenario 4: Normal purchase ---"
vm.insert_money(1.00)
vm.select_product("B1")  # Water costs $1.00

vm.display_products
```

---

### State Transition Diagram

```
                insert_money()
    ┌──────┐ ──────────────────▶ ┌───────────┐
    │ Idle │                     │ HasMoney  │◀─┐
    └──────┘ ◀────────────────── └───────────┘  │ insert_money()
                  cancel()             │
                                       │ select_product()
                                       │ (valid + sufficient funds)
                                       ▼
                                ┌─────────────┐
                                │ Dispensing   │
                                └─────────────┘
                                       │
                                       │ dispense() complete
                                       ▼
                                ┌──────┐
                                │ Idle │
                                └──────┘
```

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **State** | `VendingState`, `IdleState`, `HasMoneyState`, `DispensingState` | Behavior changes based on machine state; avoids complex if/else chains |
| **Strategy** | Could add `ChangeStrategy` for different change-making algorithms | Extensible change calculation |

### Key Design Decisions

1. **State pattern eliminates conditionals** — each state class handles only its valid operations
2. **Auto-dispense** — selecting a valid product with sufficient funds triggers dispensing automatically
3. **Open internal API** — Ruby doesn't have C++'s `friend` keyword. Instead, the machine exposes internal helper methods that states call. In Ruby, you could use `protected` or convention-based privacy, but for LLD clarity, explicit methods are preferred
4. **Immutable product codes** — products are identified by code strings, making the system extensible

---


## 12.4 Stack Overflow (Simplified)

> Design a simplified Stack Overflow system with users, questions, answers, comments, a voting system, and tag-based search.

---

### Requirements

**Functional Requirements:**
1. Users can register and have a reputation score
2. Users can post questions with a title, body, and tags
3. Users can post answers to questions
4. Users can comment on questions and answers
5. Users can upvote or downvote questions and answers
6. Upvotes increase reputation (+10 for question, +10 for answer), downvotes decrease (-2)
7. Questions can be searched by tags
8. A question can be marked as having an accepted answer

---

### Implementation

```ruby
# ─── Comment ─────────────────────────────────────────────

class Comment
  @@next_id = 1

  attr_reader :id, :body, :author, :created_at

  def initialize(body, author)
    @id = @@next_id
    @@next_id += 1
    @body = body
    @author = author
    @created_at = Time.now
  end
end

# ─── Votable (mixin for Question and Answer) ─────────────

module Votable
  def self.included(base)
    base.instance_variable_set(:@votes, {})
  end

  def init_votable
    @votes = {}  # user_id => vote (+1 or -1)
  end

  def upvote(user_id)
    @votes[user_id] = 1
  end

  def downvote(user_id)
    @votes[user_id] = -1
  end

  def remove_vote(user_id)
    @votes.delete(user_id)
  end

  def score
    @votes.values.sum
  end

  def vote_by(user_id)
    @votes.fetch(user_id, 0)
  end
end

# ─── User ────────────────────────────────────────────────

class User
  @@next_id = 1

  attr_reader :id, :username, :email, :reputation

  def initialize(username, email)
    @id = @@next_id
    @@next_id += 1
    @username = username
    @email = email
    @reputation = 0
  end

  def add_reputation(points)
    @reputation += points
    @reputation = 0 if @reputation < 0  # floor at 0
  end

  def to_s
    "#{@username} (rep: #{@reputation})"
  end
end

# ─── Answer ──────────────────────────────────────────────

class Answer
  include Votable

  @@next_id = 1

  attr_reader :id, :body, :author, :comments
  attr_accessor :accepted

  def initialize(body, author)
    @id = @@next_id
    @@next_id += 1
    @body = body
    @author = author
    @comments = []
    @accepted = false
    init_votable
  end

  def add_comment(body, author)
    @comments << Comment.new(body, author)
  end

  def print_answer(indent = 2)
    pad = " " * indent
    marker = @accepted ? "✓ " : "  "
    puts "#{pad}#{marker}[Score: #{score}] Answer by #{author}"
    puts "#{pad}  #{@body}"
    @comments.each do |comment|
      puts "#{pad}    💬 #{comment.author.username}: #{comment.body}"
    end
  end
end

# ─── Question ────────────────────────────────────────────

class Question
  include Votable

  @@next_id = 1

  attr_reader :id, :title, :body, :author, :tags, :answers, :comments

  def initialize(title, body, author, tags)
    @id = @@next_id
    @@next_id += 1
    @title = title
    @body = body
    @author = author
    @tags = tags
    @answers = []
    @comments = []
    @accepted_answer_id = nil
    init_votable
  end

  def has_tag?(tag)
    @tags.include?(tag)
  end

  def add_answer(body, author)
    @answers << Answer.new(body, author)
  end

  def add_comment(body, author)
    @comments << Comment.new(body, author)
  end

  def get_answer(answer_id)
    @answers.find { |a| a.id == answer_id }
  end

  # Only the question author can accept an answer
  def accept_answer(answer_id, requester)
    unless requester.id == @author.id
      puts "Only the question author can accept an answer."
      return false
    end

    answer = get_answer(answer_id)
    return false unless answer

    answer.accepted = true
    @accepted_answer_id = answer_id
    answer.author.add_reputation(15)  # +15 for accepted answer
    true
  end

  def print_question
    puts "\n━━━ Question ##{@id} [Score: #{score}] ━━━"
    puts "  #{@title}"
    puts "  #{@body}"
    puts "  Tags: #{@tags.map { |t| "[#{t}]" }.join(' ')}"
    puts "  Asked by: #{@author}"

    @comments.each do |comment|
      puts "  💬 #{comment.author.username}: #{comment.body}"
    end

    puts "  --- #{@answers.size} Answer(s) ---"
    @answers.each(&:print_answer)
    puts
  end
end

# ─── StackOverflow System ────────────────────────────────

class StackOverflow
  def initialize
    @users = {}       # id => User
    @questions = []
  end

  # ─── User Management ─────────────────────────────────

  def register_user(username, email)
    user = User.new(username, email)
    @users[user.id] = user
    puts "Registered user: #{username} (ID: #{user.id})"
    user
  end

  # ─── Question Management ─────────────────────────────

  def ask_question(author, title, body, tags)
    question = Question.new(title, body, author, tags)
    @questions << question
    puts "#{author.username} asked: #{title}"
    question
  end

  # ─── Voting ──────────────────────────────────────────

  def upvote_question(voter, question)
    prev = question.vote_by(voter.id)
    question.upvote(voter.id)
    question.author.add_reputation(10) if prev != 1
  end

  def downvote_question(voter, question)
    prev = question.vote_by(voter.id)
    question.downvote(voter.id)
    question.author.add_reputation(-2) if prev != -1
  end

  def upvote_answer(voter, answer)
    prev = answer.vote_by(voter.id)
    answer.upvote(voter.id)
    answer.author.add_reputation(10) if prev != 1
  end

  def downvote_answer(voter, answer)
    prev = answer.vote_by(voter.id)
    answer.downvote(voter.id)
    answer.author.add_reputation(-2) if prev != -1
  end

  # ─── Search ──────────────────────────────────────────

  def search_by_tag(tag)
    @questions.select { |q| q.has_tag?(tag) }
  end

  def search_by_keyword(keyword)
    @questions.select { |q| q.title.include?(keyword) }
  end
end

# ─── Usage ───────────────────────────────────────────────

so = StackOverflow.new

# Register users
alice   = so.register_user("alice", "alice@example.com")
bob     = so.register_user("bob", "bob@example.com")
charlie = so.register_user("charlie", "charlie@example.com")

# Alice asks a question
q1 = so.ask_question(alice,
  "How to implement Singleton in Ruby?",
  "I want a thread-safe singleton. What's the best approach?",
  ["ruby", "design-patterns", "singleton"])

# Bob answers
q1.add_answer(
  "Use Ruby's built-in Singleton module: `include Singleton`. " \
  "It's thread-safe and lazy-loaded.",
  bob)

# Charlie answers
q1.add_answer(
  "Use a class-level instance variable with a Mutex for explicit control.",
  charlie)

# Alice comments on the question
q1.add_comment("I'm using Ruby 3.2, if that matters.", alice)

# Voting
so.upvote_question(bob, q1)
so.upvote_question(charlie, q1)

bob_answer = q1.get_answer(1)
if bob_answer
  so.upvote_answer(alice, bob_answer)
  so.upvote_answer(charlie, bob_answer)
  bob_answer.add_comment("Great answer! The Singleton module is the cleanest.", alice)
end

# Alice accepts Bob's answer
q1.accept_answer(1, alice)

# Display
q1.print_question

# Search by tag
puts "\n--- Search: 'design-patterns' ---"
results = so.search_by_tag("design-patterns")
results.each { |q| puts "  Found: #{q.title}" }

# Reputation check
puts "\n--- Reputation ---"
puts alice
puts bob
puts charlie
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Observer** | Voting → reputation update | Votes on content notify the author's reputation |
| **Composite** | Question contains Answers, both contain Comments | Uniform treatment of votable/commentable content |

### Key Design Decisions

1. **Votable mixin** — shared voting logic between Question and Answer via Ruby's `include`. This is Ruby's idiomatic alternative to C++ base class inheritance for shared behavior (DRY)
2. **One vote per user** — `Hash` of `user_id => vote` prevents duplicate votes and allows vote changes
3. **Tag-based search** — simple linear scan; in production, use an inverted index
4. **Reputation system** — +10 for upvote, -2 for downvote, +15 for accepted answer
5. **Duck typing** — both `Question` and `Answer` include `Votable`, so any code that calls `score`, `upvote`, or `downvote` works on either without type checking

---


## 12.5 Logger System

> Design a logging framework with multiple log levels, multiple output targets (console, file, network), chain-of-responsibility filtering, and singleton access. This combines several design patterns into one cohesive system.

---

### Requirements

**Functional Requirements:**
1. Support log levels: DEBUG, INFO, WARN, ERROR, FATAL
2. Support multiple output targets (sinks): Console, File, Network
3. Each sink can have its own minimum log level
4. Log messages include timestamp, level, source, and message
5. The logger is a singleton — one instance used across the application
6. Sinks can be added/removed at runtime
7. Chain of Responsibility: log messages pass through a chain of handlers

---

### Implementation

```ruby
require 'monitor'

# ─── Log Level ───────────────────────────────────────────

module LogLevel
  DEBUG = 0
  INFO  = 1
  WARN  = 2
  ERROR = 3
  FATAL = 4

  NAMES = { DEBUG => "DEBUG", INFO => "INFO", WARN => "WARN",
            ERROR => "ERROR", FATAL => "FATAL" }.freeze

  def self.to_s(level)
    NAMES.fetch(level, "UNKNOWN")
  end
end

# ─── Log Message ─────────────────────────────────────────

class LogMessage
  attr_reader :level, :text, :source, :timestamp

  def initialize(level, text, source = "")
    @level = level
    @text = text
    @source = source
    @timestamp = Time.now
  end

  def format
    ts = @timestamp.strftime("%Y-%m-%d %H:%M:%S.%L")
    level_str = LogLevel.to_s(@level).ljust(5)
    msg = "#{ts} [#{level_str}]"
    msg += " [#{@source}]" unless @source.empty?
    msg += " #{@text}"
    msg
  end
end

# ─── Log Handler — Chain of Responsibility ───────────────

class LogHandler
  def initialize(min_level)
    @min_level = min_level
    @next_handler = nil
  end

  def set_next(handler)
    if @next_handler
      @next_handler.set_next(handler)
    else
      @next_handler = handler
    end
  end

  def handle(msg)
    write(msg) if msg.level >= @min_level

    # Pass to next handler in chain regardless
    @next_handler&.handle(msg)
  end

  def write(_msg)
    raise NotImplementedError, "Subclasses must implement #write"
  end

  def handler_name
    raise NotImplementedError, "Subclasses must implement #handler_name"
  end
end

# ─── Console Handler ─────────────────────────────────────

class ConsoleHandler < LogHandler
  def initialize(min_level = LogLevel::DEBUG)
    super(min_level)
  end

  def write(msg)
    if msg.level >= LogLevel::ERROR
      $stderr.puts msg.format
    else
      $stdout.puts msg.format
    end
  end

  def handler_name
    "Console"
  end
end

# ─── File Handler ────────────────────────────────────────

class FileHandler < LogHandler
  def initialize(path, min_level = LogLevel::DEBUG)
    super(min_level)
    @file = File.open(path, "a")
    @mutex = Mutex.new
  end

  def write(msg)
    @mutex.synchronize do
      @file.puts msg.format
      @file.flush
    end
  end

  def handler_name
    "File"
  end
end

# ─── Network Handler (simulated) ─────────────────────────

class NetworkHandler < LogHandler
  def initialize(endpoint, min_level = LogLevel::ERROR)
    super(min_level)
    @endpoint = endpoint
  end

  def write(msg)
    # In production, this would send to a log aggregation service
    puts "[→ #{@endpoint}] #{msg.format}"
  end

  def handler_name
    "Network(#{@endpoint})"
  end
end

# ─── Logger (Singleton) ──────────────────────────────────

class Logger
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
    @handler_chain = nil
    @global_level = LogLevel::DEBUG
  end

  def set_level(level)
    synchronize { @global_level = level }
  end

  def add_handler(handler)
    synchronize do
      if @handler_chain.nil?
        @handler_chain = handler
      else
        @handler_chain.set_next(handler)
      end
    end
  end

  def log(level, message, source = "")
    return if level < @global_level

    msg = LogMessage.new(level, message, source)

    synchronize do
      @handler_chain&.handle(msg)
    end
  end

  # Convenience methods
  def debug(msg, src = "") = log(LogLevel::DEBUG, msg, src)
  def info(msg, src = "")  = log(LogLevel::INFO, msg, src)
  def warn(msg, src = "")  = log(LogLevel::WARN, msg, src)
  def error(msg, src = "") = log(LogLevel::ERROR, msg, src)
  def fatal(msg, src = "") = log(LogLevel::FATAL, msg, src)
end

# ─── Usage ───────────────────────────────────────────────

logger = Logger.instance

# Build the handler chain:
# Console (DEBUG+) → File (INFO+) → Network (ERROR+)
logger.add_handler(ConsoleHandler.new(LogLevel::DEBUG))
logger.add_handler(FileHandler.new("app.log", LogLevel::INFO))
logger.add_handler(NetworkHandler.new("https://logs.example.com", LogLevel::ERROR))

# Log messages at various levels
logger.debug("Initializing database connection", "DBService")
logger.info("Server started on port 8080", "Main")
logger.warn("Cache miss rate above 50%", "CacheService")
logger.error("Failed to connect to payment gateway", "PaymentService")
logger.fatal("Out of memory — shutting down", "System")

# DEBUG → Console only
# INFO  → Console + File
# WARN  → Console + File
# ERROR → Console + File + Network
# FATAL → Console + File + Network
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Singleton** | `Logger` | One global logger instance |
| **Chain of Responsibility** | `LogHandler` chain | Each handler decides whether to process the message based on level, then passes to the next |
| **Observer** | Handlers observe log events | Multiple sinks react to the same log message |

### Key Design Decisions

1. **Chain of Responsibility** — handlers are linked; each processes if level matches, then passes along. This allows each handler to have its own filter level
2. **Singleton** — ensures all components use the same logger configuration. Ruby's `private_class_method :new` + class-level `Mutex` is the idiomatic thread-safe singleton (alternatively, `require 'singleton'` and `include Singleton`)
3. **Open/Closed** — new handler types (database, Slack, email) can be added without modifying existing code
4. **Thread safety** — `MonitorMixin` protects the handler chain; `FileHandler` has its own `Mutex` for file writes
5. **Endless method syntax** — Ruby 3.0+ supports `def method = expression` for one-liner convenience methods, keeping the API clean

---


## 12.6 ATM Machine

> Design an ATM system that supports card authentication, balance inquiry, withdrawal, deposit, and transaction state management. This is another classic State pattern problem with Chain of Responsibility for transaction processing.

---

### Requirements

**Functional Requirements:**
1. User inserts a card and enters a PIN for authentication
2. After authentication, user can: check balance, withdraw cash, deposit cash
3. The ATM has a finite amount of cash in different denominations
4. Withdrawals are rejected if insufficient balance or insufficient ATM cash
5. The ATM transitions between states: Idle → CardInserted → Authenticated → Transaction
6. User can cancel at any point and get their card back
7. Cash dispenser uses Chain of Responsibility for denomination selection

---

### Implementation

```ruby
# ─── Bank Account (simulated) ────────────────────────────

class BankAccount
  attr_reader :card_number, :balance

  def initialize(card, pin, balance)
    @card_number = card
    @pin = pin
    @balance = balance
  end

  def verify_pin(entered_pin)
    @pin == entered_pin
  end

  def withdraw(amount)
    return false if amount > @balance

    @balance -= amount
    true
  end

  def deposit(amount)
    @balance += amount
  end
end

# ─── Cash Dispenser — Chain of Responsibility ────────────

class CashHandler
  attr_reader :denomination

  def initialize(denom, count)
    @denomination = denom
    @count = count
    @next = nil
  end

  def set_next(handler)
    if @next
      @next.set_next(handler)
    else
      @next = handler
    end
  end

  # Try to dispense the amount using this denomination, pass remainder to next
  def dispense(amount, dispensed = [])
    return true if amount <= 0

    notes_needed = amount / @denomination
    notes_used = [notes_needed, @count].min

    if notes_used > 0
      dispensed << [@denomination, notes_used]
      @count -= notes_used
      amount -= notes_used * @denomination
    end

    return true if amount == 0

    if @next
      return @next.dispense(amount, dispensed)
    end

    # Cannot dispense remaining amount — rollback
    if notes_used > 0
      @count += notes_used
      dispensed.pop
    end
    false
  end

  def display
    puts "  $#{@denomination} x #{@count}"
    @next&.display
  end
end

# ─── ATM State Interface ─────────────────────────────────

class ATMState
  def insert_card(_atm, _card_number)  = raise(NotImplementedError)
  def enter_pin(_atm, _pin)            = raise(NotImplementedError)
  def check_balance(_atm)              = raise(NotImplementedError)
  def withdraw(_atm, _amount)          = raise(NotImplementedError)
  def deposit(_atm, _amount)           = raise(NotImplementedError)
  def eject_card(_atm)                 = raise(NotImplementedError)
  def name                             = raise(NotImplementedError)
end

# ─── ATM Machine ─────────────────────────────────────────

class ATM
  attr_reader :current_account

  def initialize
    @accounts = {}          # card => BankAccount
    @cash_chain = nil
    @state = IdleATMState.new
    @current_account = nil
  end

  def add_account(card, pin, balance)
    @accounts[card] = BankAccount.new(card, pin, balance)
  end

  def setup_cash(hundreds, fifties, twenties, tens)
    @cash_chain = CashHandler.new(100, hundreds)
    @cash_chain.set_next(CashHandler.new(50, fifties))
    @cash_chain.set_next(CashHandler.new(20, twenties))
    @cash_chain.set_next(CashHandler.new(10, tens))
  end

  # Public interface — delegates to state
  def insert_card(card)
    print "[#{@state.name}] "
    @state.insert_card(self, card)
  end

  def enter_pin(pin)
    print "[#{@state.name}] "
    @state.enter_pin(self, pin)
  end

  def check_balance
    print "[#{@state.name}] "
    @state.check_balance(self)
  end

  def withdraw(amount)
    print "[#{@state.name}] "
    @state.withdraw(self, amount)
  end

  def deposit(amount)
    print "[#{@state.name}] "
    @state.deposit(self, amount)
  end

  def eject_card
    print "[#{@state.name}] "
    @state.eject_card(self)
  end

  def display_cash
    puts "ATM Cash:"
    @cash_chain&.display
  end

  # ─── Internal helpers (used by state classes) ────────

  def set_state(new_state)
    @state = new_state
  end

  def find_account(card)
    @accounts[card]
  end

  def set_current_account(account)
    @current_account = account
  end

  def dispense_cash(amount)
    dispensed = []
    if @cash_chain&.dispense(amount, dispensed)
      puts "Dispensing:"
      dispensed.each { |denom, count| puts "  $#{denom} x #{count}" }
      true
    else
      puts "ATM cannot dispense this amount."
      false
    end
  end
end

# ─── Concrete ATM States ─────────────────────────────────

class IdleATMState < ATMState
  def name = "Idle"

  def insert_card(atm, card)
    account = atm.find_account(card)
    unless account
      puts "Card not recognized."
      return
    end
    atm.set_current_account(account)
    puts "Card inserted. Please enter your PIN."
    atm.set_state(CardInsertedState.new)
  end

  def enter_pin(_atm, _pin)    = puts("Please insert your card first.")
  def check_balance(_atm)      = puts("Please insert your card first.")
  def withdraw(_atm, _amount)  = puts("Please insert your card first.")
  def deposit(_atm, _amount)   = puts("Please insert your card first.")
  def eject_card(_atm)         = puts("No card inserted.")
end

class CardInsertedState < ATMState
  MAX_ATTEMPTS = 3

  def initialize
    @attempts = 0
  end

  def name = "CardInserted"

  def insert_card(_atm, _card) = puts("Card already inserted.")

  def enter_pin(atm, pin)
    account = atm.current_account
    if account&.verify_pin(pin)
      puts "PIN verified. Welcome!"
      atm.set_state(AuthenticatedState.new)
    else
      @attempts += 1
      if @attempts >= MAX_ATTEMPTS
        puts "Too many failed attempts. Ejecting card."
        atm.set_current_account(nil)
        atm.set_state(IdleATMState.new)
      else
        puts "Incorrect PIN. #{MAX_ATTEMPTS - @attempts} attempts remaining."
      end
    end
  end

  def check_balance(_atm)      = puts("Please enter your PIN first.")
  def withdraw(_atm, _amount)  = puts("Please enter your PIN first.")
  def deposit(_atm, _amount)   = puts("Please enter your PIN first.")

  def eject_card(atm)
    puts "Card ejected."
    atm.set_current_account(nil)
    atm.set_state(IdleATMState.new)
  end
end

class AuthenticatedState < ATMState
  def name = "Authenticated"

  def insert_card(_atm, _card) = puts("Card already inserted.")
  def enter_pin(_atm, _pin)    = puts("Already authenticated.")

  def check_balance(atm)
    account = atm.current_account
    puts "Balance: $#{'%.2f' % account.balance}" if account
  end

  def withdraw(atm, amount)
    account = atm.current_account
    return unless account

    if amount <= 0 || amount.to_i % 10 != 0
      puts "Amount must be a positive multiple of $10."
      return
    end

    if amount > account.balance
      puts "Insufficient balance. Available: $#{'%.2f' % account.balance}"
      return
    end

    # Try to dispense cash from ATM
    if atm.dispense_cash(amount.to_i)
      account.withdraw(amount)
      puts "Withdrawal successful. New balance: $#{'%.2f' % account.balance}"
    else
      puts "ATM cannot dispense this amount. Try a different amount."
    end
  end

  def deposit(atm, amount)
    account = atm.current_account
    return unless account

    if amount <= 0
      puts "Amount must be positive."
      return
    end

    account.deposit(amount)
    puts "Deposited $#{'%.2f' % amount}. New balance: $#{'%.2f' % account.balance}"
  end

  def eject_card(atm)
    puts "Card ejected. Thank you!"
    atm.set_current_account(nil)
    atm.set_state(IdleATMState.new)
  end
end

# ─── Usage ───────────────────────────────────────────────

atm = ATM.new

# Setup accounts
atm.add_account("4111-1111-1111-1111", "1234", 5000.00)
atm.add_account("4222-2222-2222-2222", "5678", 1200.00)

# Setup cash: 20x$100, 30x$50, 50x$20, 100x$10
atm.setup_cash(20, 30, 50, 100)

puts "=== ATM Session 1 ==="

# Normal flow
atm.insert_card("4111-1111-1111-1111")
atm.enter_pin("1234")
atm.check_balance
atm.withdraw(270)  # $100x2 + $50x1 + $20x1
atm.check_balance
atm.deposit(100)
atm.check_balance
atm.eject_card

puts "\n=== ATM Session 2: Wrong PIN ==="

atm.insert_card("4222-2222-2222-2222")
atm.enter_pin("0000")  # wrong
atm.enter_pin("0000")  # wrong
atm.enter_pin("0000")  # wrong — card ejected

puts "\n=== ATM Session 3: Insufficient funds ==="

atm.insert_card("4222-2222-2222-2222")
atm.enter_pin("5678")
atm.withdraw(5000)  # more than balance
atm.eject_card

puts
atm.display_cash
```

---

### State Transition Diagram

```
                insert_card()
    ┌──────┐ ──────────────────▶ ┌────────────────┐
    │ Idle │                     │ CardInserted   │
    └──────┘ ◀────────────────── └────────────────┘
                eject_card()            │
                or 3 failed PINs        │ enter_pin() (correct)
                                        ▼
                                ┌────────────────┐
                                │ Authenticated  │
                                │                │
                                │ • check_balance│
                                │ • withdraw     │
                                │ • deposit      │
                                └────────────────┘
                                        │
                                        │ eject_card()
                                        ▼
                                ┌──────┐
                                │ Idle │
                                └──────┘
```

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **State** | `ATMState`, `IdleATMState`, `CardInsertedState`, `AuthenticatedState` | ATM behavior depends entirely on current state |
| **Chain of Responsibility** | `CashHandler` chain ($100 → $50 → $20 → $10) | Each denomination handler tries to fulfill the amount, passes remainder to next |

### Key Design Decisions

1. **State pattern** — each state only handles operations valid for that state; invalid operations get clear error messages
2. **Chain of Responsibility for cash** — dispenser tries largest denomination first, cascades to smaller ones. Easy to add new denominations
3. **PIN attempt limiting** — 3 failed attempts ejects the card (security)
4. **Denomination constraint** — withdrawals must be multiples of $10 (smallest denomination)
5. **Rollback on dispense failure** — if the chain can't fulfill the amount, notes are returned to the handlers
6. **Endless methods for one-liners** — Ruby 3.0+ `def method = expression` keeps invalid-state responses concise without sacrificing readability

---


## Summary & Key Takeaways

| Problem | Core Patterns | Key Data Structures | Difficulty |
|---------|--------------|-------------------|------------|
| Parking Lot | Singleton, Strategy, Factory | Hash, Array | Easy |
| Tic-Tac-Toe | Strategy, State | 2D Array, row/col/diag sums | Easy |
| Vending Machine | State | Hash (products), State objects | Easy |
| Stack Overflow | Observer, Composite | Hash, Array, Votable mixin | Easy-Medium |
| Logger System | Singleton, Chain of Responsibility, Observer | Linked handler chain | Easy |
| ATM Machine | State, Chain of Responsibility | State objects, denomination chain | Easy |

### Common Themes Across All Problems

1. **State Pattern** appears in 4 of 6 problems — whenever behavior changes based on an object's internal state (game state, machine state, transaction state)
2. **Strategy Pattern** appears when algorithms need to be swappable (payment calculation, player input, sorting)
3. **Singleton** is used for system-wide resources (parking lot, logger)
4. **Chain of Responsibility** handles sequential processing (log handlers, cash denomination dispensing)
5. **Separation of Concerns** — entities don't know about each other's internals; they communicate through well-defined interfaces

### Ruby-Specific Patterns & Idioms

| C++ Concept | Ruby Equivalent | Notes |
|-------------|----------------|-------|
| `enum class` | Symbols (`:bike`) or module constants | Symbols are lightweight, immutable identifiers |
| `std::mutex` + `lock_guard` | `Mutex#synchronize` or `MonitorMixin` | `MonitorMixin` is reentrant; `Mutex` is not |
| Abstract base class | `raise NotImplementedError` | Ruby has no `abstract` keyword; convention-based |
| `friend class` | Public helper methods | Ruby has no `friend`; use `protected` or open API |
| `unique_ptr<T>` | Regular references (GC handles memory) | No manual memory management needed |
| Multiple inheritance | Mixins (`include`/`extend`) | `Votable` module replaces C++ `Votable` base class |
| Template/Generic | Duck typing | Any object with the right methods works |
| `static` local singleton | `private_class_method :new` + class `Mutex` | Or `require 'singleton'; include Singleton` |

### Interview Approach for LLD Problems

1. **Clarify requirements** — ask about scope, constraints, and edge cases
2. **Identify entities** — nouns in the requirements become classes
3. **Define relationships** — has-a, is-a, uses
4. **Choose patterns** — match the problem's behavioral needs to known patterns
5. **Start with the core** — implement the happy path first, then add error handling
6. **Consider extensibility** — can new types/behaviors be added without modifying existing code?
7. **Discuss trade-offs** — explain why you chose a pattern and what alternatives exist