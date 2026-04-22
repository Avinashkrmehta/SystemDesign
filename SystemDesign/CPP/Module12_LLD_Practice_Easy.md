# Module 12: LLD Practice Problems (Easy)

> This module puts everything from Modules 1–11 into practice. Each problem follows a structured approach: understand the requirements, identify the key entities, define relationships, choose appropriate design patterns, and implement a clean, extensible solution in C++. These are the problems most commonly asked in LLD interviews at the entry level. For each problem, we provide a complete, working implementation with detailed explanations.

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

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <memory>
#include <chrono>
#include <mutex>
#include <optional>
#include <algorithm>
using namespace std;

// ─── Enums ───────────────────────────────────────────────

enum class VehicleType { BIKE, CAR, TRUCK };
enum class SpotSize { SMALL, MEDIUM, LARGE };

string to_string(VehicleType type) {
    switch (type) {
        case VehicleType::BIKE:  return "Bike";
        case VehicleType::CAR:   return "Car";
        case VehicleType::TRUCK: return "Truck";
    }
    return "Unknown";
}

string to_string(SpotSize size) {
    switch (size) {
        case SpotSize::SMALL:  return "Small";
        case SpotSize::MEDIUM: return "Medium";
        case SpotSize::LARGE:  return "Large";
    }
    return "Unknown";
}

// ─── Vehicle ─────────────────────────────────────────────

class Vehicle {
    string license_plate;
    VehicleType type;

public:
    Vehicle(const string& plate, VehicleType type)
        : license_plate(plate), type(type) {}

    const string& get_plate() const { return license_plate; }
    VehicleType get_type() const { return type; }

    // Determine the minimum spot size this vehicle needs
    SpotSize min_spot_size() const {
        switch (type) {
            case VehicleType::BIKE:  return SpotSize::SMALL;
            case VehicleType::CAR:   return SpotSize::MEDIUM;
            case VehicleType::TRUCK: return SpotSize::LARGE;
        }
        return SpotSize::LARGE;
    }

    bool fits_in(SpotSize spot_size) const {
        return spot_size >= min_spot_size();
    }
};

// ─── Parking Spot ────────────────────────────────────────

class ParkingSpot {
    int spot_id;
    int floor_number;
    SpotSize size;
    Vehicle* parked_vehicle = nullptr;  // non-owning pointer

public:
    ParkingSpot(int id, int floor, SpotSize size)
        : spot_id(id), floor_number(floor), size(size) {}

    int get_id() const { return spot_id; }
    int get_floor() const { return floor_number; }
    SpotSize get_size() const { return size; }
    bool is_available() const { return parked_vehicle == nullptr; }
    Vehicle* get_vehicle() const { return parked_vehicle; }

    bool can_fit(const Vehicle& vehicle) const {
        return is_available() && vehicle.fits_in(size);
    }

    void park(Vehicle* vehicle) {
        parked_vehicle = vehicle;
    }

    void vacate() {
        parked_vehicle = nullptr;
    }
};

// ─── Ticket ──────────────────────────────────────────────

class Ticket {
    static int next_id;

    int ticket_id;
    string vehicle_plate;
    int spot_id;
    int floor_number;
    chrono::steady_clock::time_point entry_time;

public:
    Ticket(const string& plate, int spot_id, int floor)
        : ticket_id(next_id++), vehicle_plate(plate),
          spot_id(spot_id), floor_number(floor),
          entry_time(chrono::steady_clock::now()) {}

    int get_id() const { return ticket_id; }
    const string& get_vehicle_plate() const { return vehicle_plate; }
    int get_spot_id() const { return spot_id; }
    int get_floor() const { return floor_number; }
    chrono::steady_clock::time_point get_entry_time() const { return entry_time; }

    // Duration in hours (for payment calculation)
    double get_duration_hours() const {
        auto now = chrono::steady_clock::now();
        auto duration = chrono::duration_cast<chrono::minutes>(now - entry_time);
        return duration.count() / 60.0;
    }

    void print() const {
        cout << "Ticket #" << ticket_id
             << " | Vehicle: " << vehicle_plate
             << " | Floor: " << floor_number
             << " | Spot: " << spot_id << endl;
    }
};

int Ticket::next_id = 1;

// ─── Payment Strategy (Strategy Pattern) ─────────────────

class PaymentStrategy {
public:
    virtual ~PaymentStrategy() = default;
    virtual double calculate(VehicleType type, double hours) const = 0;
    virtual string name() const = 0;
};

// Flat hourly rate based on vehicle type
class HourlyPayment : public PaymentStrategy {
    unordered_map<int, double> rates;  // VehicleType → rate per hour

public:
    HourlyPayment() {
        rates[static_cast<int>(VehicleType::BIKE)]  = 10.0;
        rates[static_cast<int>(VehicleType::CAR)]   = 20.0;
        rates[static_cast<int>(VehicleType::TRUCK)] = 30.0;
    }

    double calculate(VehicleType type, double hours) const override {
        double rate = rates.at(static_cast<int>(type));
        double billable_hours = max(1.0, ceil(hours));  // minimum 1 hour
        return rate * billable_hours;
    }

    string name() const override { return "Hourly"; }
};

// Tiered pricing: first 2 hours flat, then per-hour after
class TieredPayment : public PaymentStrategy {
public:
    double calculate(VehicleType type, double hours) const override {
        double base_rate = 0;
        double hourly_rate = 0;

        switch (type) {
            case VehicleType::BIKE:  base_rate = 15; hourly_rate = 5;  break;
            case VehicleType::CAR:   base_rate = 30; hourly_rate = 10; break;
            case VehicleType::TRUCK: base_rate = 50; hourly_rate = 20; break;
        }

        if (hours <= 2.0) return base_rate;
        return base_rate + hourly_rate * ceil(hours - 2.0);
    }

    string name() const override { return "Tiered"; }
};

// ─── Floor ───────────────────────────────────────────────

class Floor {
    int floor_number;
    vector<ParkingSpot> spots;

public:
    Floor(int number, int small_count, int medium_count, int large_count)
        : floor_number(number) {
        int id = 1;
        for (int i = 0; i < small_count; i++)
            spots.emplace_back(id++, number, SpotSize::SMALL);
        for (int i = 0; i < medium_count; i++)
            spots.emplace_back(id++, number, SpotSize::MEDIUM);
        for (int i = 0; i < large_count; i++)
            spots.emplace_back(id++, number, SpotSize::LARGE);
    }

    int get_floor_number() const { return floor_number; }

    // Find the smallest available spot that fits the vehicle
    ParkingSpot* find_spot(const Vehicle& vehicle) {
        ParkingSpot* best = nullptr;
        for (auto& spot : spots) {
            if (spot.can_fit(vehicle)) {
                if (!best || spot.get_size() < best->get_size()) {
                    best = &spot;
                }
            }
        }
        return best;
    }

    ParkingSpot* find_spot_by_id(int id) {
        for (auto& spot : spots) {
            if (spot.get_id() == id) return &spot;
        }
        return nullptr;
    }

    int available_count(SpotSize size) const {
        int count = 0;
        for (const auto& spot : spots) {
            if (spot.get_size() == size && spot.is_available()) count++;
        }
        return count;
    }

    int total_count(SpotSize size) const {
        int count = 0;
        for (const auto& spot : spots) {
            if (spot.get_size() == size) count++;
        }
        return count;
    }
};

// ─── Parking Lot (Singleton) ─────────────────────────────

class ParkingLot {
    string name;
    vector<Floor> floors;
    unordered_map<string, Ticket> active_tickets;  // plate → ticket
    unique_ptr<PaymentStrategy> payment_strategy;
    mutable mutex mtx;

    ParkingLot() : payment_strategy(make_unique<HourlyPayment>()) {}

public:
    static ParkingLot& instance() {
        static ParkingLot lot;
        return lot;
    }

    void initialize(const string& lot_name,
                    const vector<tuple<int, int, int>>& floor_configs) {
        lock_guard<mutex> lock(mtx);
        name = lot_name;
        floors.clear();
        active_tickets.clear();

        int floor_num = 1;
        for (auto& [small, medium, large] : floor_configs) {
            floors.emplace_back(floor_num++, small, medium, large);
        }
    }

    void set_payment_strategy(unique_ptr<PaymentStrategy> strategy) {
        lock_guard<mutex> lock(mtx);
        payment_strategy = std::move(strategy);
    }

    // ─── Entry Gate Logic ────────────────────────────────

    optional<Ticket> enter(Vehicle& vehicle) {
        lock_guard<mutex> lock(mtx);

        // Check if vehicle is already parked
        if (active_tickets.count(vehicle.get_plate())) {
            cout << "Vehicle " << vehicle.get_plate()
                 << " is already parked!" << endl;
            return nullopt;
        }

        // Find the best spot across all floors
        for (auto& floor : floors) {
            ParkingSpot* spot = floor.find_spot(vehicle);
            if (spot) {
                spot->park(&vehicle);
                Ticket ticket(vehicle.get_plate(), spot->get_id(),
                              floor.get_floor_number());
                active_tickets.emplace(vehicle.get_plate(), ticket);

                cout << "Vehicle " << vehicle.get_plate()
                     << " (" << to_string(vehicle.get_type()) << ")"
                     << " parked at Floor " << floor.get_floor_number()
                     << ", Spot " << spot->get_id()
                     << " (" << to_string(spot->get_size()) << ")" << endl;

                return ticket;
            }
        }

        cout << "No available spot for " << to_string(vehicle.get_type())
             << " " << vehicle.get_plate() << endl;
        return nullopt;
    }

    // ─── Exit Gate Logic ─────────────────────────────────

    optional<double> exit(const string& plate) {
        lock_guard<mutex> lock(mtx);

        auto it = active_tickets.find(plate);
        if (it == active_tickets.end()) {
            cout << "No active ticket for vehicle " << plate << endl;
            return nullopt;
        }

        Ticket& ticket = it->second;
        double hours = ticket.get_duration_hours();

        // Find and vacate the spot
        for (auto& floor : floors) {
            if (floor.get_floor_number() == ticket.get_floor()) {
                ParkingSpot* spot = floor.find_spot_by_id(ticket.get_spot_id());
                if (spot) {
                    VehicleType vtype = spot->get_vehicle()->get_type();
                    double amount = payment_strategy->calculate(vtype, hours);

                    spot->vacate();
                    active_tickets.erase(it);

                    cout << "Vehicle " << plate << " exited."
                         << " Duration: " << hours << " hrs"
                         << " | Payment (" << payment_strategy->name()
                         << "): $" << amount << endl;

                    return amount;
                }
            }
        }

        return nullopt;
    }

    // ─── Display Status ──────────────────────────────────

    void display_status() const {
        lock_guard<mutex> lock(mtx);
        cout << "\n=== " << name << " Status ===" << endl;
        for (const auto& floor : floors) {
            cout << "Floor " << floor.get_floor_number() << ": "
                 << "Small(" << floor.available_count(SpotSize::SMALL)
                 << "/" << floor.total_count(SpotSize::SMALL) << ") "
                 << "Medium(" << floor.available_count(SpotSize::MEDIUM)
                 << "/" << floor.total_count(SpotSize::MEDIUM) << ") "
                 << "Large(" << floor.available_count(SpotSize::LARGE)
                 << "/" << floor.total_count(SpotSize::LARGE) << ")"
                 << endl;
        }
        cout << "Active vehicles: " << active_tickets.size() << endl;
        cout << "========================\n" << endl;
    }

    // Delete copy/move
    ParkingLot(const ParkingLot&) = delete;
    ParkingLot& operator=(const ParkingLot&) = delete;
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    auto& lot = ParkingLot::instance();

    // 2 floors: Floor 1 (5 small, 5 medium, 2 large), Floor 2 (3 small, 5 medium, 3 large)
    lot.initialize("Downtown Parking", {{5, 5, 2}, {3, 5, 3}});
    lot.display_status();

    // Vehicles arrive
    Vehicle bike1("BIKE-001", VehicleType::BIKE);
    Vehicle car1("CAR-001", VehicleType::CAR);
    Vehicle car2("CAR-002", VehicleType::CAR);
    Vehicle truck1("TRUCK-001", VehicleType::TRUCK);

    auto t1 = lot.enter(bike1);    // parks in Small spot
    auto t2 = lot.enter(car1);     // parks in Medium spot
    auto t3 = lot.enter(car2);     // parks in Medium spot
    auto t4 = lot.enter(truck1);   // parks in Large spot

    lot.display_status();

    // Vehicles exit
    lot.exit("BIKE-001");
    lot.exit("CAR-001");

    lot.display_status();

    // Switch to tiered payment
    lot.set_payment_strategy(make_unique<TieredPayment>());
    lot.exit("TRUCK-001");

    return 0;
}
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Singleton** | `ParkingLot` | One parking lot instance, global access |
| **Strategy** | `PaymentStrategy` | Swap payment algorithms (hourly, tiered) without changing lot logic |
| **Factory** | `Vehicle::min_spot_size()` | Maps vehicle type to required spot size |

### Key Design Decisions

1. **Smallest-fit allocation** — a Bike gets a Small spot before a Medium, reducing waste
2. **Thread safety** — mutex protects all shared state for concurrent entry/exit
3. **Separation of concerns** — Vehicle doesn't know about spots; ParkingSpot doesn't know about payment
4. **Open/Closed** — new vehicle types or payment strategies can be added without modifying existing code

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

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <stdexcept>
using namespace std;

// ─── Enums ───────────────────────────────────────────────

enum class Symbol { NONE, X, O };

char to_char(Symbol s) {
    switch (s) {
        case Symbol::X: return 'X';
        case Symbol::O: return 'O';
        default: return '.';
    }
}

enum class GameState { IN_PROGRESS, X_WINS, O_WINS, DRAW };

// ─── Board ───────────────────────────────────────────────

class Board {
    int size;
    vector<vector<Symbol>> grid;
    int moves_made = 0;

    // Optimization: track row/col/diagonal sums for O(1) win check
    // +1 for X, -1 for O. If any sum reaches +N or -N, that player wins.
    vector<int> row_sums;
    vector<int> col_sums;
    int diag_sum = 0;       // main diagonal (top-left to bottom-right)
    int anti_diag_sum = 0;  // anti-diagonal (top-right to bottom-left)

    int symbol_value(Symbol s) const {
        return (s == Symbol::X) ? 1 : -1;
    }

public:
    Board(int n = 3) : size(n), grid(n, vector<Symbol>(n, Symbol::NONE)),
                       row_sums(n, 0), col_sums(n, 0) {}

    int get_size() const { return size; }

    bool is_valid_move(int row, int col) const {
        return row >= 0 && row < size && col >= 0 && col < size
               && grid[row][col] == Symbol::NONE;
    }

    // Place a symbol and check if this move wins — O(1)
    bool place(int row, int col, Symbol symbol) {
        if (!is_valid_move(row, col)) {
            throw invalid_argument("Invalid move: (" + to_string(row)
                                   + ", " + to_string(col) + ")");
        }

        grid[row][col] = symbol;
        moves_made++;

        int val = symbol_value(symbol);
        row_sums[row] += val;
        col_sums[col] += val;
        if (row == col) diag_sum += val;
        if (row + col == size - 1) anti_diag_sum += val;

        // Check if this move wins
        return abs(row_sums[row]) == size
            || abs(col_sums[col]) == size
            || abs(diag_sum) == size
            || abs(anti_diag_sum) == size;
    }

    bool is_full() const { return moves_made == size * size; }

    Symbol get_cell(int row, int col) const { return grid[row][col]; }

    void print() const {
        // Column headers
        cout << "  ";
        for (int c = 0; c < size; c++) cout << c << " ";
        cout << endl;

        for (int r = 0; r < size; r++) {
            cout << r << " ";
            for (int c = 0; c < size; c++) {
                cout << to_char(grid[r][c]);
                if (c < size - 1) cout << "|";
            }
            cout << endl;
            if (r < size - 1) {
                cout << "  ";
                for (int c = 0; c < size; c++) {
                    cout << "-";
                    if (c < size - 1) cout << "+";
                }
                cout << endl;
            }
        }
        cout << endl;
    }
};

// ─── Player ──────────────────────────────────────────────

class Player {
    string name;
    Symbol symbol;

public:
    Player(const string& name, Symbol symbol) : name(name), symbol(symbol) {}

    const string& get_name() const { return name; }
    Symbol get_symbol() const { return symbol; }

    // Get move from player — can be overridden for AI
    virtual pair<int, int> get_move(const Board& board) {
        int row, col;
        cout << get_name() << " (" << to_char(symbol) << "), enter row col: ";
        cin >> row >> col;
        return {row, col};
    }

    virtual ~Player() = default;
};

// Simple AI player — picks first available spot
class SimpleAIPlayer : public Player {
public:
    SimpleAIPlayer(const string& name, Symbol symbol) : Player(name, symbol) {}

    pair<int, int> get_move(const Board& board) override {
        int size = board.get_size();
        for (int r = 0; r < size; r++) {
            for (int c = 0; c < size; c++) {
                if (board.is_valid_move(r, c)) {
                    cout << get_name() << " plays: " << r << " " << c << endl;
                    return {r, c};
                }
            }
        }
        throw runtime_error("No valid moves");
    }
};

// ─── Game ────────────────────────────────────────────────

class TicTacToeGame {
    Board board;
    vector<unique_ptr<Player>> players;
    int current_player_index = 0;
    GameState state = GameState::IN_PROGRESS;

public:
    TicTacToeGame(int board_size = 3) : board(board_size) {}

    void add_player(unique_ptr<Player> player) {
        if (players.size() >= 2) {
            throw runtime_error("Maximum 2 players");
        }
        players.push_back(std::move(player));
    }

    GameState get_state() const { return state; }

    void play() {
        if (players.size() != 2) {
            throw runtime_error("Need exactly 2 players to start");
        }

        cout << "=== Tic-Tac-Toe (" << board.get_size() << "x"
             << board.get_size() << ") ===" << endl;
        board.print();

        while (state == GameState::IN_PROGRESS) {
            Player& current = *players[current_player_index];

            // Get a valid move
            bool valid_move = false;
            while (!valid_move) {
                try {
                    auto [row, col] = current.get_move(board);
                    bool wins = board.place(row, col, current.get_symbol());
                    valid_move = true;

                    board.print();

                    if (wins) {
                        state = (current.get_symbol() == Symbol::X)
                                ? GameState::X_WINS : GameState::O_WINS;
                        cout << current.get_name() << " wins!" << endl;
                    } else if (board.is_full()) {
                        state = GameState::DRAW;
                        cout << "It's a draw!" << endl;
                    }
                }
                catch (const invalid_argument& e) {
                    cout << "Invalid move. Try again." << endl;
                }
            }

            // Switch player
            current_player_index = 1 - current_player_index;
        }
    }
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    TicTacToeGame game(3);  // 3x3 board

    game.add_player(make_unique<Player>("Alice", Symbol::X));
    game.add_player(make_unique<SimpleAIPlayer>("Bot", Symbol::O));

    game.play();

    return 0;
}
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | `Player::get_move()` (virtual) | Different player types (human, AI) provide moves differently |
| **State** | `GameState` enum + game loop | Game behavior changes based on state (in progress, won, draw) |

### Key Design Decisions

1. **O(1) win checking** — instead of scanning the entire board after each move, we track row/column/diagonal sums. A sum of +N or -N means a player completed that line
2. **NxN extensible** — board size is a constructor parameter, win logic works for any N
3. **Polymorphic players** — `Player` base class with virtual `get_move()` allows human, AI, or network players
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

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <vector>
#include <memory>
#include <stdexcept>
using namespace std;

// ─── Product ─────────────────────────────────────────────

class Product {
    string name;
    double price;
    int quantity;

public:
    Product(const string& name, double price, int qty)
        : name(name), price(price), quantity(qty) {}

    const string& get_name() const { return name; }
    double get_price() const { return price; }
    int get_quantity() const { return quantity; }
    bool is_available() const { return quantity > 0; }

    void decrement() {
        if (quantity > 0) quantity--;
    }

    void restock(int amount) { quantity += amount; }
};

// ─── Forward Declarations ────────────────────────────────

class VendingMachine;

// ─── State Interface (State Pattern) ─────────────────────

class VendingState {
public:
    virtual ~VendingState() = default;
    virtual void insert_money(VendingMachine& machine, double amount) = 0;
    virtual void select_product(VendingMachine& machine, const string& code) = 0;
    virtual void dispense(VendingMachine& machine) = 0;
    virtual void cancel(VendingMachine& machine) = 0;
    virtual string name() const = 0;
};

// ─── Vending Machine ─────────────────────────────────────

class VendingMachine {
    unordered_map<string, Product> products;  // code → product
    double inserted_money = 0.0;
    string selected_product_code;
    unique_ptr<VendingState> current_state;

    // Allow states to access internals
    friend class IdleState;
    friend class HasMoneyState;
    friend class DispensingState;

public:
    VendingMachine();  // defined after state classes

    void add_product(const string& code, const string& name,
                     double price, int quantity) {
        products.emplace(code, Product(name, price, quantity));
    }

    // Public interface — delegates to current state
    void insert_money(double amount) {
        cout << "[" << current_state->name() << "] ";
        current_state->insert_money(*this, amount);
    }

    void select_product(const string& code) {
        cout << "[" << current_state->name() << "] ";
        current_state->select_product(*this, code);
    }

    void dispense() {
        cout << "[" << current_state->name() << "] ";
        current_state->dispense(*this);
    }

    void cancel() {
        cout << "[" << current_state->name() << "] ";
        current_state->cancel(*this);
    }

    void display_products() const {
        cout << "\n=== Available Products ===" << endl;
        for (const auto& [code, product] : products) {
            cout << code << ": " << product.get_name()
                 << " - $" << product.get_price()
                 << " (" << product.get_quantity() << " left)" << endl;
        }
        cout << "Inserted: $" << inserted_money << endl;
        cout << "State: " << current_state->name() << endl;
        cout << "==========================\n" << endl;
    }

private:
    void set_state(unique_ptr<VendingState> state) {
        current_state = std::move(state);
    }

    void add_money(double amount) { inserted_money += amount; }

    void return_money() {
        if (inserted_money > 0) {
            cout << "Returning $" << inserted_money << endl;
            inserted_money = 0;
        }
    }

    double get_inserted_money() const { return inserted_money; }

    Product* get_product(const string& code) {
        auto it = products.find(code);
        return (it != products.end()) ? &it->second : nullptr;
    }

    void set_selected(const string& code) { selected_product_code = code; }
    const string& get_selected() const { return selected_product_code; }

    void reset() {
        inserted_money = 0;
        selected_product_code.clear();
    }
};

// ─── Concrete States ─────────────────────────────────────

class IdleState : public VendingState {
public:
    string name() const override { return "Idle"; }

    void insert_money(VendingMachine& m, double amount) override {
        m.add_money(amount);
        cout << "Inserted $" << amount
             << ". Total: $" << m.get_inserted_money() << endl;
        m.set_state(make_unique<HasMoneyState>());
    }

    void select_product(VendingMachine& m, const string& code) override {
        cout << "Please insert money first." << endl;
    }

    void dispense(VendingMachine& m) override {
        cout << "Please insert money and select a product." << endl;
    }

    void cancel(VendingMachine& m) override {
        cout << "Nothing to cancel." << endl;
    }
};

class HasMoneyState : public VendingState {
public:
    string name() const override { return "HasMoney"; }

    void insert_money(VendingMachine& m, double amount) override {
        m.add_money(amount);
        cout << "Inserted $" << amount
             << ". Total: $" << m.get_inserted_money() << endl;
    }

    void select_product(VendingMachine& m, const string& code) override {
        Product* product = m.get_product(code);

        if (!product) {
            cout << "Invalid product code: " << code << endl;
            return;
        }

        if (!product->is_available()) {
            cout << product->get_name() << " is out of stock." << endl;
            return;
        }

        if (m.get_inserted_money() < product->get_price()) {
            cout << "Insufficient funds. Need $" << product->get_price()
                 << ", have $" << m.get_inserted_money() << endl;
            return;
        }

        m.set_selected(code);
        cout << "Selected: " << product->get_name() << endl;
        m.set_state(make_unique<DispensingState>());
        m.dispense();  // auto-dispense after selection
    }

    void dispense(VendingMachine& m) override {
        cout << "Please select a product first." << endl;
    }

    void cancel(VendingMachine& m) override {
        cout << "Transaction cancelled. ";
        m.return_money();
        m.reset();
        m.set_state(make_unique<IdleState>());
    }
};

class DispensingState : public VendingState {
public:
    string name() const override { return "Dispensing"; }

    void insert_money(VendingMachine& m, double amount) override {
        cout << "Please wait, dispensing in progress." << endl;
    }

    void select_product(VendingMachine& m, const string& code) override {
        cout << "Please wait, dispensing in progress." << endl;
    }

    void dispense(VendingMachine& m) override {
        Product* product = m.get_product(m.get_selected());
        if (!product) {
            cout << "Error: product not found." << endl;
            m.return_money();
            m.reset();
            m.set_state(make_unique<IdleState>());
            return;
        }

        // Dispense product
        product->decrement();
        double change = m.get_inserted_money() - product->get_price();

        cout << "Dispensing: " << product->get_name() << endl;
        if (change > 0) {
            cout << "Returning change: $" << change << endl;
        }

        m.reset();
        m.set_state(make_unique<IdleState>());
    }

    void cancel(VendingMachine& m) override {
        cout << "Cannot cancel during dispensing." << endl;
    }
};

// ─── VendingMachine Constructor ──────────────────────────

VendingMachine::VendingMachine()
    : current_state(make_unique<IdleState>()) {}

// ─── Usage ───────────────────────────────────────────────

int main() {
    VendingMachine vm;

    // Stock the machine
    vm.add_product("A1", "Cola", 1.50, 10);
    vm.add_product("A2", "Chips", 2.00, 5);
    vm.add_product("B1", "Water", 1.00, 15);
    vm.add_product("B2", "Candy", 0.75, 20);

    vm.display_products();

    // Scenario 1: Successful purchase
    cout << "--- Scenario 1: Buy Cola ---" << endl;
    vm.insert_money(1.00);
    vm.insert_money(1.00);
    vm.select_product("A1");  // Cola costs $1.50, inserted $2.00 → change $0.50

    // Scenario 2: Insufficient funds
    cout << "\n--- Scenario 2: Insufficient funds ---" << endl;
    vm.insert_money(0.50);
    vm.select_product("A2");  // Chips costs $2.00, only $0.50

    // Scenario 3: Cancel transaction
    cout << "\n--- Scenario 3: Cancel ---" << endl;
    vm.cancel();  // returns $0.50

    // Scenario 4: Out of stock
    cout << "\n--- Scenario 4: Normal purchase ---" << endl;
    vm.insert_money(1.00);
    vm.select_product("B1");  // Water costs $1.00

    vm.display_products();

    return 0;
}
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
3. **Friend access** — states are friends of VendingMachine to access private helpers, keeping the public API clean
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

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <unordered_set>
#include <memory>
#include <algorithm>
#include <chrono>
using namespace std;

// ─── Forward Declarations ────────────────────────────────

class User;
class Question;
class Answer;

// ─── Comment ─────────────────────────────────────────────

class Comment {
    static int next_id;
    int id;
    string body;
    User* author;
    chrono::system_clock::time_point created_at;

public:
    Comment(const string& body, User* author)
        : id(next_id++), body(body), author(author),
          created_at(chrono::system_clock::now()) {}

    int get_id() const { return id; }
    const string& get_body() const { return body; }
    User* get_author() const { return author; }
};

int Comment::next_id = 1;

// ─── Votable (base for Question and Answer) ──────────────

class Votable {
protected:
    unordered_map<int, int> votes;  // user_id → vote (+1 or -1)

public:
    virtual ~Votable() = default;

    void upvote(int user_id) { votes[user_id] = 1; }
    void downvote(int user_id) { votes[user_id] = -1; }

    void remove_vote(int user_id) { votes.erase(user_id); }

    int get_score() const {
        int score = 0;
        for (auto& [_, vote] : votes) score += vote;
        return score;
    }

    int get_vote_by(int user_id) const {
        auto it = votes.find(user_id);
        return (it != votes.end()) ? it->second : 0;
    }
};

// ─── User ────────────────────────────────────────────────

class User {
    static int next_id;
    int id;
    string username;
    string email;
    int reputation = 0;

public:
    User(const string& username, const string& email)
        : id(next_id++), username(username), email(email) {}

    int get_id() const { return id; }
    const string& get_username() const { return username; }
    int get_reputation() const { return reputation; }

    void add_reputation(int points) {
        reputation += points;
        if (reputation < 0) reputation = 0;  // floor at 0
    }

    void print() const {
        cout << username << " (rep: " << reputation << ")";
    }
};

int User::next_id = 1;

// ─── Answer ──────────────────────────────────────────────

class Answer : public Votable {
    static int next_id;
    int id;
    string body;
    User* author;
    vector<Comment> comments;
    bool accepted = false;

public:
    Answer(const string& body, User* author)
        : id(next_id++), body(body), author(author) {}

    int get_id() const { return id; }
    const string& get_body() const { return body; }
    User* get_author() const { return author; }
    bool is_accepted() const { return accepted; }
    void accept() { accepted = true; }

    void add_comment(const string& body, User* author) {
        comments.emplace_back(body, author);
    }

    void print(int indent = 2) const {
        string pad(indent, ' ');
        cout << pad << (accepted ? "✓ " : "  ")
             << "[Score: " << get_score() << "] "
             << "Answer by ";
        author->print();
        cout << endl;
        cout << pad << "  " << body << endl;

        for (const auto& comment : comments) {
            cout << pad << "    💬 " << comment.get_author()->get_username()
                 << ": " << comment.get_body() << endl;
        }
    }
};

int Answer::next_id = 1;

// ─── Question ────────────────────────────────────────────

class Question : public Votable {
    static int next_id;
    int id;
    string title;
    string body;
    User* author;
    vector<string> tags;
    vector<Answer> answers;
    vector<Comment> comments;
    int accepted_answer_id = -1;

public:
    Question(const string& title, const string& body,
             User* author, const vector<string>& tags)
        : id(next_id++), title(title), body(body),
          author(author), tags(tags) {}

    int get_id() const { return id; }
    const string& get_title() const { return title; }
    User* get_author() const { return author; }
    const vector<string>& get_tags() const { return tags; }

    bool has_tag(const string& tag) const {
        return find(tags.begin(), tags.end(), tag) != tags.end();
    }

    void add_answer(const string& body, User* author) {
        answers.emplace_back(body, author);
    }

    void add_comment(const string& body, User* author) {
        comments.emplace_back(body, author);
    }

    Answer* get_answer(int answer_id) {
        for (auto& a : answers) {
            if (a.get_id() == answer_id) return &a;
        }
        return nullptr;
    }

    // Only the question author can accept an answer
    bool accept_answer(int answer_id, User* requester) {
        if (requester->get_id() != author->get_id()) {
            cout << "Only the question author can accept an answer." << endl;
            return false;
        }
        Answer* answer = get_answer(answer_id);
        if (!answer) return false;

        // Unaccept previous if any
        for (auto& a : answers) {
            if (a.is_accepted()) {
                // Remove previous acceptance reputation
            }
        }

        answer->accept();
        accepted_answer_id = answer_id;
        answer->get_author()->add_reputation(15);  // +15 for accepted answer
        return true;
    }

    void print() const {
        cout << "\n━━━ Question #" << id << " [Score: " << get_score() << "] ━━━" << endl;
        cout << "  " << title << endl;
        cout << "  " << body << endl;
        cout << "  Tags: ";
        for (const auto& tag : tags) cout << "[" << tag << "] ";
        cout << endl;
        cout << "  Asked by: ";
        author->print();
        cout << endl;

        for (const auto& comment : comments) {
            cout << "  💬 " << comment.get_author()->get_username()
                 << ": " << comment.get_body() << endl;
        }

        cout << "  --- " << answers.size() << " Answer(s) ---" << endl;
        for (const auto& answer : answers) {
            answer.print();
        }
        cout << endl;
    }
};

int Question::next_id = 1;

// ─── StackOverflow System ────────────────────────────────

class StackOverflow {
    unordered_map<int, User> users;
    vector<Question> questions;

public:
    // ─── User Management ─────────────────────────────────

    User& register_user(const string& username, const string& email) {
        User user(username, email);
        int id = user.get_id();
        users.emplace(id, std::move(user));
        cout << "Registered user: " << username << " (ID: " << id << ")" << endl;
        return users.at(id);
    }

    // ─── Question Management ─────────────────────────────

    Question& ask_question(User& author, const string& title,
                           const string& body, const vector<string>& tags) {
        questions.emplace_back(title, body, &author, tags);
        cout << author.get_username() << " asked: " << title << endl;
        return questions.back();
    }

    // ─── Voting ──────────────────────────────────────────

    void upvote_question(User& voter, Question& question) {
        int prev = question.get_vote_by(voter.get_id());
        question.upvote(voter.get_id());
        if (prev != 1) {  // new upvote or changed from downvote
            question.get_author()->add_reputation(10);
        }
    }

    void downvote_question(User& voter, Question& question) {
        int prev = question.get_vote_by(voter.get_id());
        question.downvote(voter.get_id());
        if (prev != -1) {
            question.get_author()->add_reputation(-2);
        }
    }

    void upvote_answer(User& voter, Answer& answer) {
        int prev = answer.get_vote_by(voter.get_id());
        answer.upvote(voter.get_id());
        if (prev != 1) {
            answer.get_author()->add_reputation(10);
        }
    }

    void downvote_answer(User& voter, Answer& answer) {
        int prev = answer.get_vote_by(voter.get_id());
        answer.downvote(voter.get_id());
        if (prev != -1) {
            answer.get_author()->add_reputation(-2);
        }
    }

    // ─── Search ──────────────────────────────────────────

    vector<Question*> search_by_tag(const string& tag) {
        vector<Question*> results;
        for (auto& q : questions) {
            if (q.has_tag(tag)) results.push_back(&q);
        }
        return results;
    }

    vector<Question*> search_by_keyword(const string& keyword) {
        vector<Question*> results;
        for (auto& q : questions) {
            if (q.get_title().find(keyword) != string::npos) {
                results.push_back(&q);
            }
        }
        return results;
    }
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    StackOverflow so;

    // Register users
    auto& alice = so.register_user("alice", "alice@example.com");
    auto& bob = so.register_user("bob", "bob@example.com");
    auto& charlie = so.register_user("charlie", "charlie@example.com");

    // Alice asks a question
    auto& q1 = so.ask_question(alice,
        "How to implement Singleton in C++?",
        "I want a thread-safe singleton. What's the best approach?",
        {"cpp", "design-patterns", "singleton"});

    // Bob answers
    q1.add_answer(
        "Use Meyer's Singleton: static local variable in a function. "
        "C++11 guarantees thread-safe initialization.",
        &bob);

    // Charlie answers
    q1.add_answer(
        "Use std::call_once with a static pointer for explicit control.",
        &charlie);

    // Alice comments on the question
    q1.add_comment("I'm using C++17, if that matters.", &alice);

    // Voting
    so.upvote_question(bob, q1);
    so.upvote_question(charlie, q1);

    Answer* bob_answer = q1.get_answer(1);
    if (bob_answer) {
        so.upvote_answer(alice, *bob_answer);
        so.upvote_answer(charlie, *bob_answer);
        bob_answer->add_comment("Great answer! Meyer's is the cleanest.", &alice);
    }

    // Alice accepts Bob's answer
    q1.accept_answer(1, &alice);

    // Display
    q1.print();

    // Search by tag
    cout << "\n--- Search: 'design-patterns' ---" << endl;
    auto results = so.search_by_tag("design-patterns");
    for (auto* q : results) {
        cout << "  Found: " << q->get_title() << endl;
    }

    // Reputation check
    cout << "\n--- Reputation ---" << endl;
    alice.print(); cout << endl;
    bob.print(); cout << endl;
    charlie.print(); cout << endl;

    return 0;
}
```

---

### Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Observer** | Voting → reputation update | Votes on content notify the author's reputation |
| **Composite** | Question contains Answers, both contain Comments | Uniform treatment of votable/commentable content |

### Key Design Decisions

1. **Votable base class** — shared voting logic between Question and Answer (DRY)
2. **One vote per user** — `unordered_map<user_id, vote>` prevents duplicate votes and allows vote changes
3. **Tag-based search** — simple linear scan; in production, use an inverted index
4. **Reputation system** — +10 for upvote, -2 for downvote, +15 for accepted answer

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

```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <vector>
#include <memory>
#include <mutex>
#include <chrono>
#include <iomanip>
#include <sstream>
using namespace std;

// ─── Log Level ───────────────────────────────────────────

enum class LogLevel { DEBUG = 0, INFO = 1, WARN = 2, ERROR = 3, FATAL = 4 };

string level_to_string(LogLevel level) {
    switch (level) {
        case LogLevel::DEBUG: return "DEBUG";
        case LogLevel::INFO:  return "INFO";
        case LogLevel::WARN:  return "WARN";
        case LogLevel::ERROR: return "ERROR";
        case LogLevel::FATAL: return "FATAL";
    }
    return "UNKNOWN";
}

// ─── Log Message ─────────────────────────────────────────

struct LogMessage {
    LogLevel level;
    string text;
    string source;
    chrono::system_clock::time_point timestamp;

    LogMessage(LogLevel lvl, const string& msg, const string& src = "")
        : level(lvl), text(msg), source(src),
          timestamp(chrono::system_clock::now()) {}

    string format() const {
        auto time = chrono::system_clock::to_time_t(timestamp);
        auto ms = chrono::duration_cast<chrono::milliseconds>(
            timestamp.time_since_epoch()) % 1000;

        ostringstream oss;
        oss << put_time(localtime(&time), "%Y-%m-%d %H:%M:%S")
            << "." << setfill('0') << setw(3) << ms.count()
            << " [" << setw(5) << level_to_string(level) << "]";

        if (!source.empty()) {
            oss << " [" << source << "]";
        }
        oss << " " << text;
        return oss.str();
    }
};

// ─── Log Handler — Chain of Responsibility ───────────────

class LogHandler {
protected:
    LogLevel min_level;
    unique_ptr<LogHandler> next_handler;

public:
    LogHandler(LogLevel level) : min_level(level) {}
    virtual ~LogHandler() = default;

    void set_next(unique_ptr<LogHandler> next) {
        if (next_handler) {
            next_handler->set_next(std::move(next));
        } else {
            next_handler = std::move(next);
        }
    }

    void handle(const LogMessage& msg) {
        if (msg.level >= min_level) {
            write(msg);
        }
        // Pass to next handler in chain regardless
        if (next_handler) {
            next_handler->handle(msg);
        }
    }

    virtual void write(const LogMessage& msg) = 0;
    virtual string handler_name() const = 0;
};

// ─── Console Handler ─────────────────────────────────────

class ConsoleHandler : public LogHandler {
public:
    ConsoleHandler(LogLevel level = LogLevel::DEBUG)
        : LogHandler(level) {}

    void write(const LogMessage& msg) override {
        if (msg.level >= LogLevel::ERROR) {
            cerr << msg.format() << endl;
        } else {
            cout << msg.format() << endl;
        }
    }

    string handler_name() const override { return "Console"; }
};

// ─── File Handler ────────────────────────────────────────

class FileHandler : public LogHandler {
    ofstream file;
    mutex file_mtx;

public:
    FileHandler(const string& path, LogLevel level = LogLevel::DEBUG)
        : LogHandler(level), file(path, ios::app) {
        if (!file.is_open()) {
            throw runtime_error("Cannot open log file: " + path);
        }
    }

    void write(const LogMessage& msg) override {
        lock_guard<mutex> lock(file_mtx);
        file << msg.format() << endl;
        file.flush();
    }

    string handler_name() const override { return "File"; }
};

// ─── Network Handler (simulated) ─────────────────────────

class NetworkHandler : public LogHandler {
    string endpoint;

public:
    NetworkHandler(const string& endpoint, LogLevel level = LogLevel::ERROR)
        : LogHandler(level), endpoint(endpoint) {}

    void write(const LogMessage& msg) override {
        // In production, this would send to a log aggregation service
        cout << "[→ " << endpoint << "] " << msg.format() << endl;
    }

    string handler_name() const override { return "Network(" + endpoint + ")"; }
};

// ─── Logger (Singleton) ──────────────────────────────────

class Logger {
    unique_ptr<LogHandler> handler_chain;
    LogLevel global_level = LogLevel::DEBUG;
    mutable mutex mtx;

    Logger() = default;

public:
    static Logger& instance() {
        static Logger logger;
        return logger;
    }

    void set_level(LogLevel level) {
        lock_guard<mutex> lock(mtx);
        global_level = level;
    }

    void add_handler(unique_ptr<LogHandler> handler) {
        lock_guard<mutex> lock(mtx);
        if (!handler_chain) {
            handler_chain = std::move(handler);
        } else {
            handler_chain->set_next(std::move(handler));
        }
    }

    void log(LogLevel level, const string& message, const string& source = "") {
        if (level < global_level) return;

        LogMessage msg(level, message, source);

        lock_guard<mutex> lock(mtx);
        if (handler_chain) {
            handler_chain->handle(msg);
        }
    }

    // Convenience methods
    void debug(const string& msg, const string& src = "") {
        log(LogLevel::DEBUG, msg, src);
    }
    void info(const string& msg, const string& src = "") {
        log(LogLevel::INFO, msg, src);
    }
    void warn(const string& msg, const string& src = "") {
        log(LogLevel::WARN, msg, src);
    }
    void error(const string& msg, const string& src = "") {
        log(LogLevel::ERROR, msg, src);
    }
    void fatal(const string& msg, const string& src = "") {
        log(LogLevel::FATAL, msg, src);
    }

    // Delete copy/move
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;
};

// ─── Usage ───────────────────────────────────────────────

int main() {
    auto& logger = Logger::instance();

    // Build the handler chain:
    // Console (DEBUG+) → File (INFO+) → Network (ERROR+)
    logger.add_handler(make_unique<ConsoleHandler>(LogLevel::DEBUG));
    logger.add_handler(make_unique<FileHandler>("app.log", LogLevel::INFO));
    logger.add_handler(make_unique<NetworkHandler>(
        "https://logs.example.com", LogLevel::ERROR));

    // Log messages at various levels
    logger.debug("Initializing database connection", "DBService");
    logger.info("Server started on port 8080", "Main");
    logger.warn("Cache miss rate above 50%", "CacheService");
    logger.error("Failed to connect to payment gateway", "PaymentService");
    logger.fatal("Out of memory — shutting down", "System");

    // DEBUG → Console only
    // INFO  → Console + File
    // WARN  → Console + File
    // ERROR → Console + File + Network
    // FATAL → Console + File + Network

    return 0;
}
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
2. **Singleton** — ensures all components use the same logger configuration
3. **Open/Closed** — new handler types (database, Slack, email) can be added without modifying existing code
4. **Thread safety** — mutex protects the handler chain and file writes

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

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <memory>
#include <vector>
using namespace std;

// ─── Bank Account (simulated) ────────────────────────────

class BankAccount {
    string card_number;
    string pin;
    double balance;

public:
    BankAccount(const string& card, const string& pin, double balance)
        : card_number(card), pin(pin), balance(balance) {}

    const string& get_card() const { return card_number; }
    bool verify_pin(const string& entered_pin) const { return pin == entered_pin; }
    double get_balance() const { return balance; }

    bool withdraw(double amount) {
        if (amount > balance) return false;
        balance -= amount;
        return true;
    }

    void deposit(double amount) {
        balance += amount;
    }
};

// ─── Cash Dispenser — Chain of Responsibility ────────────

class CashHandler {
protected:
    int denomination;
    int count;  // number of notes available
    unique_ptr<CashHandler> next;

public:
    CashHandler(int denom, int count) : denomination(denom), count(count) {}
    virtual ~CashHandler() = default;

    void set_next(unique_ptr<CashHandler> handler) {
        if (next) {
            next->set_next(std::move(handler));
        } else {
            next = std::move(handler);
        }
    }

    // Try to dispense the amount using this denomination, pass remainder to next
    bool dispense(int amount, vector<pair<int, int>>& dispensed) {
        if (amount <= 0) return true;

        int notes_needed = amount / denomination;
        int notes_used = min(notes_needed, count);

        if (notes_used > 0) {
            dispensed.push_back({denomination, notes_used});
            count -= notes_used;
            amount -= notes_used * denomination;
        }

        if (amount == 0) return true;

        if (next) {
            return next->dispense(amount, dispensed);
        }

        // Cannot dispense remaining amount — rollback
        if (notes_used > 0) {
            count += notes_used;
            dispensed.pop_back();
        }
        return false;
    }

    void display() const {
        cout << "  $" << denomination << " x " << count << endl;
        if (next) next->display();
    }
};

// ─── Forward Declaration ─────────────────────────────────

class ATM;

// ─── ATM State Interface ─────────────────────────────────

class ATMState {
public:
    virtual ~ATMState() = default;
    virtual void insert_card(ATM& atm, const string& card_number) = 0;
    virtual void enter_pin(ATM& atm, const string& pin) = 0;
    virtual void check_balance(ATM& atm) = 0;
    virtual void withdraw(ATM& atm, double amount) = 0;
    virtual void deposit(ATM& atm, double amount) = 0;
    virtual void eject_card(ATM& atm) = 0;
    virtual string name() const = 0;
};

// ─── ATM Machine ─────────────────────────────────────────

class ATM {
    unordered_map<string, BankAccount> accounts;  // card → account
    unique_ptr<CashHandler> cash_chain;
    unique_ptr<ATMState> state;
    BankAccount* current_account = nullptr;

    friend class IdleATMState;
    friend class CardInsertedState;
    friend class AuthenticatedState;

public:
    ATM();  // defined after state classes

    void add_account(const string& card, const string& pin, double balance) {
        accounts.emplace(card, BankAccount(card, pin, balance));
    }

    void setup_cash(int hundreds, int fifties, int twenties, int tens) {
        cash_chain = make_unique<CashHandler>(100, hundreds);
        cash_chain->set_next(make_unique<CashHandler>(50, fifties));
        cash_chain->set_next(make_unique<CashHandler>(20, twenties));
        cash_chain->set_next(make_unique<CashHandler>(10, tens));
    }

    // Public interface — delegates to state
    void insert_card(const string& card) {
        cout << "[" << state->name() << "] ";
        state->insert_card(*this, card);
    }

    void enter_pin(const string& pin) {
        cout << "[" << state->name() << "] ";
        state->enter_pin(*this, pin);
    }

    void check_balance() {
        cout << "[" << state->name() << "] ";
        state->check_balance(*this);
    }

    void withdraw(double amount) {
        cout << "[" << state->name() << "] ";
        state->withdraw(*this, amount);
    }

    void deposit(double amount) {
        cout << "[" << state->name() << "] ";
        state->deposit(*this, amount);
    }

    void eject_card() {
        cout << "[" << state->name() << "] ";
        state->eject_card(*this);
    }

    void display_cash() const {
        cout << "ATM Cash:" << endl;
        if (cash_chain) cash_chain->display();
    }

private:
    void set_state(unique_ptr<ATMState> new_state) {
        state = std::move(new_state);
    }

    BankAccount* find_account(const string& card) {
        auto it = accounts.find(card);
        return (it != accounts.end()) ? &it->second : nullptr;
    }

    void set_current_account(BankAccount* account) {
        current_account = account;
    }

    BankAccount* get_current_account() { return current_account; }

    bool dispense_cash(int amount) {
        vector<pair<int, int>> dispensed;
        if (cash_chain && cash_chain->dispense(amount, dispensed)) {
            cout << "Dispensing:" << endl;
            for (auto& [denom, count] : dispensed) {
                cout << "  $" << denom << " x " << count << endl;
            }
            return true;
        }
        cout << "ATM cannot dispense this amount." << endl;
        return false;
    }
};

// ─── Concrete ATM States ─────────────────────────────────

class IdleATMState : public ATMState {
public:
    string name() const override { return "Idle"; }

    void insert_card(ATM& atm, const string& card) override {
        BankAccount* account = atm.find_account(card);
        if (!account) {
            cout << "Card not recognized." << endl;
            return;
        }
        atm.set_current_account(account);
        cout << "Card inserted. Please enter your PIN." << endl;
        atm.set_state(make_unique<CardInsertedState>());
    }

    void enter_pin(ATM& atm, const string& pin) override {
        cout << "Please insert your card first." << endl;
    }
    void check_balance(ATM& atm) override {
        cout << "Please insert your card first." << endl;
    }
    void withdraw(ATM& atm, double amount) override {
        cout << "Please insert your card first." << endl;
    }
    void deposit(ATM& atm, double amount) override {
        cout << "Please insert your card first." << endl;
    }
    void eject_card(ATM& atm) override {
        cout << "No card inserted." << endl;
    }
};

class CardInsertedState : public ATMState {
    int attempts = 0;
    static const int MAX_ATTEMPTS = 3;

public:
    string name() const override { return "CardInserted"; }

    void insert_card(ATM& atm, const string& card) override {
        cout << "Card already inserted." << endl;
    }

    void enter_pin(ATM& atm, const string& pin) override {
        BankAccount* account = atm.get_current_account();
        if (account && account->verify_pin(pin)) {
            cout << "PIN verified. Welcome!" << endl;
            atm.set_state(make_unique<AuthenticatedState>());
        } else {
            attempts++;
            if (attempts >= MAX_ATTEMPTS) {
                cout << "Too many failed attempts. Ejecting card." << endl;
                atm.set_current_account(nullptr);
                atm.set_state(make_unique<IdleATMState>());
            } else {
                cout << "Incorrect PIN. " << (MAX_ATTEMPTS - attempts)
                     << " attempts remaining." << endl;
            }
        }
    }

    void check_balance(ATM& atm) override {
        cout << "Please enter your PIN first." << endl;
    }
    void withdraw(ATM& atm, double amount) override {
        cout << "Please enter your PIN first." << endl;
    }
    void deposit(ATM& atm, double amount) override {
        cout << "Please enter your PIN first." << endl;
    }

    void eject_card(ATM& atm) override {
        cout << "Card ejected." << endl;
        atm.set_current_account(nullptr);
        atm.set_state(make_unique<IdleATMState>());
    }
};

class AuthenticatedState : public ATMState {
public:
    string name() const override { return "Authenticated"; }

    void insert_card(ATM& atm, const string& card) override {
        cout << "Card already inserted." << endl;
    }

    void enter_pin(ATM& atm, const string& pin) override {
        cout << "Already authenticated." << endl;
    }

    void check_balance(ATM& atm) override {
        BankAccount* account = atm.get_current_account();
        if (account) {
            cout << "Balance: $" << account->get_balance() << endl;
        }
    }

    void withdraw(ATM& atm, double amount) override {
        BankAccount* account = atm.get_current_account();
        if (!account) return;

        if (amount <= 0 || (int)amount % 10 != 0) {
            cout << "Amount must be a positive multiple of $10." << endl;
            return;
        }

        if (amount > account->get_balance()) {
            cout << "Insufficient balance. Available: $"
                 << account->get_balance() << endl;
            return;
        }

        // Try to dispense cash from ATM
        if (atm.dispense_cash((int)amount)) {
            account->withdraw(amount);
            cout << "Withdrawal successful. New balance: $"
                 << account->get_balance() << endl;
        } else {
            cout << "ATM cannot dispense this amount. Try a different amount." << endl;
        }
    }

    void deposit(ATM& atm, double amount) override {
        BankAccount* account = atm.get_current_account();
        if (!account) return;

        if (amount <= 0) {
            cout << "Amount must be positive." << endl;
            return;
        }

        account->deposit(amount);
        cout << "Deposited $" << amount
             << ". New balance: $" << account->get_balance() << endl;
    }

    void eject_card(ATM& atm) override {
        cout << "Card ejected. Thank you!" << endl;
        atm.set_current_account(nullptr);
        atm.set_state(make_unique<IdleATMState>());
    }
};

// ─── ATM Constructor ─────────────────────────────────────

ATM::ATM() : state(make_unique<IdleATMState>()) {}

// ─── Usage ───────────────────────────────────────────────

int main() {
    ATM atm;

    // Setup accounts
    atm.add_account("4111-1111-1111-1111", "1234", 5000.00);
    atm.add_account("4222-2222-2222-2222", "5678", 1200.00);

    // Setup cash: 20x$100, 30x$50, 50x$20, 100x$10
    atm.setup_cash(20, 30, 50, 100);

    cout << "=== ATM Session 1 ===" << endl;

    // Normal flow
    atm.insert_card("4111-1111-1111-1111");
    atm.enter_pin("1234");
    atm.check_balance();
    atm.withdraw(270);  // $100x2 + $50x1 + $20x1
    atm.check_balance();
    atm.deposit(100);
    atm.check_balance();
    atm.eject_card();

    cout << "\n=== ATM Session 2: Wrong PIN ===" << endl;

    atm.insert_card("4222-2222-2222-2222");
    atm.enter_pin("0000");  // wrong
    atm.enter_pin("0000");  // wrong
    atm.enter_pin("0000");  // wrong — card ejected

    cout << "\n=== ATM Session 3: Insufficient funds ===" << endl;

    atm.insert_card("4222-2222-2222-2222");
    atm.enter_pin("5678");
    atm.withdraw(5000);  // more than balance
    atm.eject_card();

    cout << endl;
    atm.display_cash();

    return 0;
}
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

---


## Summary & Key Takeaways

| Problem | Core Patterns | Key Data Structures | Difficulty |
|---------|--------------|-------------------|------------|
| Parking Lot | Singleton, Strategy, Factory | HashMap, Vector | Easy |
| Tic-Tac-Toe | Strategy, State | 2D Array, row/col/diag sums | Easy |
| Vending Machine | State | HashMap (products), State objects | Easy |
| Stack Overflow | Observer, Composite | HashMap, Vector, Votable hierarchy | Easy-Medium |
| Logger System | Singleton, Chain of Responsibility, Observer | Linked handler chain | Easy |
| ATM Machine | State, Chain of Responsibility | State objects, denomination chain | Easy |

### Common Themes Across All Problems

1. **State Pattern** appears in 4 of 6 problems — whenever behavior changes based on an object's internal state (game state, machine state, transaction state)
2. **Strategy Pattern** appears when algorithms need to be swappable (payment calculation, player input, sorting)
3. **Singleton** is used for system-wide resources (parking lot, logger)
4. **Chain of Responsibility** handles sequential processing (log handlers, cash denomination dispensing)
5. **Separation of Concerns** — entities don't know about each other's internals; they communicate through well-defined interfaces

### Interview Approach for LLD Problems

1. **Clarify requirements** — ask about scope, constraints, and edge cases
2. **Identify entities** — nouns in the requirements become classes
3. **Define relationships** — has-a, is-a, uses
4. **Choose patterns** — match the problem's behavioral needs to known patterns
5. **Start with the core** — implement the happy path first, then add error handling
6. **Consider extensibility** — can new types/behaviors be added without modifying existing code?
7. **Discuss trade-offs** — explain why you chose a pattern and what alternatives exist

