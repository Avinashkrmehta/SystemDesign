# Module 14: LLD Practice Problems (Hard)

> Hard LLD problems test your ability to design complex, real-world systems with intricate rules, multiple interacting components, and non-trivial algorithms. These problems require combining multiple design patterns, handling concurrency, managing complex state transitions, and building extensible architectures. In interviews, you're expected to identify the right abstractions, define clean interfaces, handle edge cases, and justify your design decisions. This module covers 10 hard problems in depth with complete C++ implementations.

---

## 14.1 Chess Game

> Chess is one of the most classic LLD interview problems. It tests your ability to model complex rules, validate moves per piece type, detect game states (check, checkmate, stalemate), and handle special moves (castling, en passant, pawn promotion). The key challenge is designing a clean class hierarchy for pieces while keeping move validation extensible and the game state manageable.

---

### Requirements

1. **Board**: 8×8 grid with alternating colors
2. **Pieces**: King, Queen, Rook, Bishop, Knight, Pawn — each with unique movement rules
3. **Players**: Two players (White and Black), alternating turns
4. **Move Validation**: Each piece type has specific legal moves; moves cannot leave own king in check
5. **Game States**: Active, Check, Checkmate, Stalemate, Resigned, Draw
6. **Special Moves**: Castling (king-side and queen-side), En Passant, Pawn Promotion
7. **Move History**: Track all moves for undo and game record

---

### Key Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Piece hierarchy | Abstract `Piece` base class with virtual `canMove()` | Each piece has unique movement — polymorphism is natural |
| Board representation | 8×8 2D array of `Piece*` | Direct access by position, simple and efficient |
| Move validation | Two-phase: piece-specific + global (king safety) | Separates concerns — piece logic vs game rules |
| Game state | State pattern or enum with transition logic | Game has clear states with defined transitions |
| Special moves | Handled in `Game` class, not in individual pieces | Castling/en passant involve multiple pieces and game state |

---

### Class Design

```
+------------------+       +------------------+       +------------------+
|      Game        |       |      Board       |       |      Player      |
+------------------+       +------------------+       +------------------+
| - board          |<>---->| - grid[8][8]     |       | - name           |
| - players[2]     |<>---->| + getPiece()     |       | - color          |
| - currentTurn    |       | + setPiece()     |       | - capturedPieces |
| - status         |       | + movePiece()    |       +------------------+
| - moveHistory    |       | + isInside()     |
| + makeMove()     |       +------------------+
| + isCheck()      |
| + isCheckmate()  |              ^
| + isStalemate()  |              |
+------------------+       +------+------+
                           |    Piece    |
                           +-------------+
                           | - color     |
                           | - position  |
                           | - hasMoved  |
                           | + canMove() | (pure virtual)
                           | + getType() | (pure virtual)
                           +-------------+
                                 ^
          +----------+-----------+----------+-----------+----------+
          |          |           |          |           |          |
        King      Queen       Rook      Bishop      Knight      Pawn
```

---

### Implementation

**Enums and Position:**

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <string>
#include <cmath>
#include <optional>
#include <algorithm>
using namespace std;

enum class Color { WHITE, BLACK };
enum class PieceType { KING, QUEEN, ROOK, BISHOP, KNIGHT, PAWN };
enum class GameStatus { ACTIVE, CHECK, CHECKMATE, STALEMATE, RESIGNED, DRAW };

string color_to_string(Color c) {
    return c == Color::WHITE ? "White" : "Black";
}

Color opposite(Color c) {
    return c == Color::WHITE ? Color::BLACK : Color::WHITE;
}

struct Position {
    int row, col;

    Position(int r, int c) : row(r), col(c) {}

    bool is_valid() const {
        return row >= 0 && row < 8 && col >= 0 && col < 8;
    }

    bool operator==(const Position& other) const {
        return row == other.row && col == other.col;
    }

    bool operator!=(const Position& other) const {
        return !(*this == other);
    }

    string to_string() const {
        return string(1, 'a' + col) + std::to_string(row + 1);
    }
};
```

**Abstract Piece class:**

```cpp
class Board;  // forward declaration

class Piece {
protected:
    Color color;
    Position position;
    bool has_moved = false;

public:
    Piece(Color color, Position pos) : color(color), position(pos) {}
    virtual ~Piece() = default;

    Color get_color() const { return color; }
    Position get_position() const { return position; }
    bool get_has_moved() const { return has_moved; }

    void set_position(Position pos) {
        position = pos;
        has_moved = true;
    }

    // Returns true if the piece can move from its current position to 'target'
    // Does NOT check if the move leaves own king in check (that's the Game's job)
    virtual bool canMove(const Board& board, Position target) const = 0;

    virtual PieceType getType() const = 0;

    virtual char getSymbol() const = 0;

    // Get all positions this piece can potentially move to (for check/checkmate detection)
    virtual vector<Position> getPossibleMoves(const Board& board) const {
        vector<Position> moves;
        for (int r = 0; r < 8; r++) {
            for (int c = 0; c < 8; c++) {
                Position target(r, c);
                if (target != position && canMove(board, target)) {
                    moves.push_back(target);
                }
            }
        }
        return moves;
    }
};
```

**Board class:**

```cpp
class Board {
    Piece* grid[8][8] = {};  // nullptr means empty square

public:
    Board() = default;

    Piece* getPiece(Position pos) const {
        if (!pos.is_valid()) return nullptr;
        return grid[pos.row][pos.col];
    }

    void setPiece(Position pos, Piece* piece) {
        if (pos.is_valid()) {
            grid[pos.row][pos.col] = piece;
        }
    }

    void removePiece(Position pos) {
        if (pos.is_valid()) {
            grid[pos.row][pos.col] = nullptr;
        }
    }

    void movePiece(Position from, Position to) {
        Piece* piece = getPiece(from);
        if (piece) {
            grid[to.row][to.col] = piece;
            grid[from.row][from.col] = nullptr;
            piece->set_position(to);
        }
    }

    // Check if path between two positions is clear (for sliding pieces: rook, bishop, queen)
    bool isPathClear(Position from, Position to) const {
        int dr = 0, dc = 0;
        if (to.row > from.row) dr = 1;
        else if (to.row < from.row) dr = -1;
        if (to.col > from.col) dc = 1;
        else if (to.col < from.col) dc = -1;

        int r = from.row + dr;
        int c = from.col + dc;

        while (r != to.row || c != to.col) {
            if (grid[r][c] != nullptr) return false;
            r += dr;
            c += dc;
        }
        return true;
    }

    // Find the king of a given color
    Position findKing(Color color) const {
        for (int r = 0; r < 8; r++) {
            for (int c = 0; c < 8; c++) {
                Piece* p = grid[r][c];
                if (p && p->getType() == PieceType::KING && p->get_color() == color) {
                    return Position(r, c);
                }
            }
        }
        throw runtime_error("King not found — invalid board state");
    }

    // Get all pieces of a given color
    vector<Piece*> getPieces(Color color) const {
        vector<Piece*> pieces;
        for (int r = 0; r < 8; r++) {
            for (int c = 0; c < 8; c++) {
                if (grid[r][c] && grid[r][c]->get_color() == color) {
                    pieces.push_back(grid[r][c]);
                }
            }
        }
        return pieces;
    }

    // Check if a square is attacked by any piece of the given color
    bool isSquareAttacked(Position pos, Color by_color) const {
        for (auto* piece : getPieces(by_color)) {
            if (piece->canMove(*this, pos)) {
                return true;
            }
        }
        return false;
    }

    void display() const {
        cout << "  a b c d e f g h" << endl;
        for (int r = 7; r >= 0; r--) {
            cout << (r + 1) << " ";
            for (int c = 0; c < 8; c++) {
                if (grid[r][c]) {
                    cout << grid[r][c]->getSymbol() << " ";
                } else {
                    cout << ". ";
                }
            }
            cout << (r + 1) << endl;
        }
        cout << "  a b c d e f g h" << endl;
    }
};
```

**Concrete Piece classes:**

```cpp
class King : public Piece {
public:
    using Piece::Piece;

    PieceType getType() const override { return PieceType::KING; }
    char getSymbol() const override { return color == Color::WHITE ? 'K' : 'k'; }

    bool canMove(const Board& board, Position target) const override {
        if (!target.is_valid()) return false;

        int dr = abs(target.row - position.row);
        int dc = abs(target.col - position.col);

        // King moves one square in any direction
        if (dr > 1 || dc > 1) return false;
        if (dr == 0 && dc == 0) return false;

        // Can't capture own piece
        Piece* dest = board.getPiece(target);
        if (dest && dest->get_color() == color) return false;

        return true;
    }
};

class Queen : public Piece {
public:
    using Piece::Piece;

    PieceType getType() const override { return PieceType::QUEEN; }
    char getSymbol() const override { return color == Color::WHITE ? 'Q' : 'q'; }

    bool canMove(const Board& board, Position target) const override {
        if (!target.is_valid()) return false;

        int dr = abs(target.row - position.row);
        int dc = abs(target.col - position.col);

        // Queen moves like rook (straight) or bishop (diagonal)
        bool straight = (dr == 0 || dc == 0);
        bool diagonal = (dr == dc);

        if (!straight && !diagonal) return false;

        // Path must be clear
        if (!board.isPathClear(position, target)) return false;

        // Can't capture own piece
        Piece* dest = board.getPiece(target);
        if (dest && dest->get_color() == color) return false;

        return true;
    }
};

class Rook : public Piece {
public:
    using Piece::Piece;

    PieceType getType() const override { return PieceType::ROOK; }
    char getSymbol() const override { return color == Color::WHITE ? 'R' : 'r'; }

    bool canMove(const Board& board, Position target) const override {
        if (!target.is_valid()) return false;

        int dr = abs(target.row - position.row);
        int dc = abs(target.col - position.col);

        // Rook moves in straight lines (horizontal or vertical)
        if (dr != 0 && dc != 0) return false;

        if (!board.isPathClear(position, target)) return false;

        Piece* dest = board.getPiece(target);
        if (dest && dest->get_color() == color) return false;

        return true;
    }
};

class Bishop : public Piece {
public:
    using Piece::Piece;

    PieceType getType() const override { return PieceType::BISHOP; }
    char getSymbol() const override { return color == Color::WHITE ? 'B' : 'b'; }

    bool canMove(const Board& board, Position target) const override {
        if (!target.is_valid()) return false;

        int dr = abs(target.row - position.row);
        int dc = abs(target.col - position.col);

        // Bishop moves diagonally
        if (dr != dc || dr == 0) return false;

        if (!board.isPathClear(position, target)) return false;

        Piece* dest = board.getPiece(target);
        if (dest && dest->get_color() == color) return false;

        return true;
    }
};

class Knight : public Piece {
public:
    using Piece::Piece;

    PieceType getType() const override { return PieceType::KNIGHT; }
    char getSymbol() const override { return color == Color::WHITE ? 'N' : 'n'; }

    bool canMove(const Board& board, Position target) const override {
        if (!target.is_valid()) return false;

        int dr = abs(target.row - position.row);
        int dc = abs(target.col - position.col);

        // Knight moves in L-shape: 2+1 or 1+2
        if (!((dr == 2 && dc == 1) || (dr == 1 && dc == 2))) return false;

        // Knight can jump over pieces — no path check needed

        Piece* dest = board.getPiece(target);
        if (dest && dest->get_color() == color) return false;

        return true;
    }
};

class Pawn : public Piece {
public:
    using Piece::Piece;

    PieceType getType() const override { return PieceType::PAWN; }
    char getSymbol() const override { return color == Color::WHITE ? 'P' : 'p'; }

    bool canMove(const Board& board, Position target) const override {
        if (!target.is_valid()) return false;

        int direction = (color == Color::WHITE) ? 1 : -1;
        int start_row = (color == Color::WHITE) ? 1 : 6;

        int dr = target.row - position.row;
        int dc = target.col - position.col;

        // Forward move (1 square)
        if (dc == 0 && dr == direction) {
            return board.getPiece(target) == nullptr;  // must be empty
        }

        // Forward move (2 squares from starting position)
        if (dc == 0 && dr == 2 * direction && position.row == start_row) {
            Position intermediate(position.row + direction, position.col);
            return board.getPiece(intermediate) == nullptr &&
                   board.getPiece(target) == nullptr;
        }

        // Diagonal capture
        if (abs(dc) == 1 && dr == direction) {
            Piece* dest = board.getPiece(target);
            if (dest && dest->get_color() != color) return true;  // normal capture
            // En passant is handled separately in the Game class
            return false;
        }

        return false;
    }
};
```

**Move record and Game class:**

```cpp
struct Move {
    Position from;
    Position to;
    Piece* piece;
    Piece* captured;       // nullptr if no capture
    bool is_castling;
    bool is_en_passant;
    bool is_promotion;
    PieceType promotion_type;

    Move(Position f, Position t, Piece* p, Piece* cap = nullptr)
        : from(f), to(t), piece(p), captured(cap),
          is_castling(false), is_en_passant(false),
          is_promotion(false), promotion_type(PieceType::QUEEN) {}
};

class Game {
    Board board;
    vector<unique_ptr<Piece>> all_pieces;  // owns all piece objects
    Color current_turn = Color::WHITE;
    GameStatus status = GameStatus::ACTIVE;
    vector<Move> move_history;

public:
    Game() {
        setupBoard();
    }

    void setupBoard() {
        // Place white pieces
        placePiece<Rook>(Color::WHITE, Position(0, 0));
        placePiece<Knight>(Color::WHITE, Position(0, 1));
        placePiece<Bishop>(Color::WHITE, Position(0, 2));
        placePiece<Queen>(Color::WHITE, Position(0, 3));
        placePiece<King>(Color::WHITE, Position(0, 4));
        placePiece<Bishop>(Color::WHITE, Position(0, 5));
        placePiece<Knight>(Color::WHITE, Position(0, 6));
        placePiece<Rook>(Color::WHITE, Position(0, 7));
        for (int c = 0; c < 8; c++) {
            placePiece<Pawn>(Color::WHITE, Position(1, c));
        }

        // Place black pieces
        placePiece<Rook>(Color::BLACK, Position(7, 0));
        placePiece<Knight>(Color::BLACK, Position(7, 1));
        placePiece<Bishop>(Color::BLACK, Position(7, 2));
        placePiece<Queen>(Color::BLACK, Position(7, 3));
        placePiece<King>(Color::BLACK, Position(7, 4));
        placePiece<Bishop>(Color::BLACK, Position(7, 5));
        placePiece<Knight>(Color::BLACK, Position(7, 6));
        placePiece<Rook>(Color::BLACK, Position(7, 7));
        for (int c = 0; c < 8; c++) {
            placePiece<Pawn>(Color::BLACK, Position(6, c));
        }
    }

    template <typename T>
    void placePiece(Color color, Position pos) {
        auto piece = make_unique<T>(color, pos);
        board.setPiece(pos, piece.get());
        all_pieces.push_back(std::move(piece));
    }

    bool makeMove(Position from, Position to) {
        if (status == GameStatus::CHECKMATE || status == GameStatus::STALEMATE ||
            status == GameStatus::RESIGNED) {
            cout << "Game is over!" << endl;
            return false;
        }

        Piece* piece = board.getPiece(from);
        if (!piece) {
            cout << "No piece at " << from.to_string() << endl;
            return false;
        }

        if (piece->get_color() != current_turn) {
            cout << "Not your turn!" << endl;
            return false;
        }

        // Check for castling
        if (piece->getType() == PieceType::KING && abs(to.col - from.col) == 2) {
            return tryCastling(from, to);
        }

        // Check for en passant
        if (piece->getType() == PieceType::PAWN && abs(to.col - from.col) == 1 &&
            board.getPiece(to) == nullptr) {
            return tryEnPassant(from, to);
        }

        // Normal move validation
        if (!piece->canMove(board, to)) {
            cout << "Invalid move for " << piece->getSymbol() << endl;
            return false;
        }

        // Simulate the move and check if it leaves own king in check
        Piece* captured = board.getPiece(to);
        board.movePiece(from, to);
        if (captured) board.removePiece(to);  // already overwritten by movePiece

        if (isInCheck(current_turn)) {
            // Undo the move — can't leave own king in check
            board.movePiece(to, from);
            piece->set_position(from);  // reset position
            if (captured) board.setPiece(to, captured);
            cout << "Move leaves king in check!" << endl;
            return false;
        }

        // Record the move
        Move move(from, to, piece, captured);

        // Check for pawn promotion
        int promotion_row = (current_turn == Color::WHITE) ? 7 : 0;
        if (piece->getType() == PieceType::PAWN && to.row == promotion_row) {
            promotePawn(to, PieceType::QUEEN);  // auto-promote to queen
            move.is_promotion = true;
        }

        move_history.push_back(move);

        // Switch turns
        current_turn = opposite(current_turn);

        // Update game status
        updateGameStatus();

        return true;
    }

    bool isInCheck(Color color) const {
        Position king_pos = board.findKing(color);
        return board.isSquareAttacked(king_pos, opposite(color));
    }

    bool isCheckmate(Color color) {
        if (!isInCheck(color)) return false;
        return !hasAnyLegalMove(color);
    }

    bool isStalemate(Color color) {
        if (isInCheck(color)) return false;
        return !hasAnyLegalMove(color);
    }

    GameStatus getStatus() const { return status; }
    Color getCurrentTurn() const { return current_turn; }
    void display() const { board.display(); }

private:
    bool hasAnyLegalMove(Color color) {
        for (auto* piece : board.getPieces(color)) {
            auto moves = piece->getPossibleMoves(board);
            for (auto& target : moves) {
                // Simulate each move
                Position from = piece->get_position();
                Piece* captured = board.getPiece(target);
                board.movePiece(from, target);

                bool still_in_check = isInCheck(color);

                // Undo
                board.movePiece(target, from);
                piece->set_position(from);
                if (captured) board.setPiece(target, captured);

                if (!still_in_check) return true;  // found at least one legal move
            }
        }
        return false;
    }

    bool tryCastling(Position from, Position to) {
        Piece* king = board.getPiece(from);
        if (king->get_has_moved()) return false;
        if (isInCheck(current_turn)) return false;  // can't castle out of check

        bool king_side = (to.col > from.col);
        int rook_col = king_side ? 7 : 0;
        Position rook_pos(from.row, rook_col);
        Piece* rook = board.getPiece(rook_pos);

        if (!rook || rook->getType() != PieceType::ROOK || rook->get_has_moved()) {
            return false;
        }

        // Check that path between king and rook is clear
        int start_col = min(from.col, rook_col) + 1;
        int end_col = max(from.col, rook_col);
        for (int c = start_col; c < end_col; c++) {
            if (board.getPiece(Position(from.row, c)) != nullptr) return false;
        }

        // King cannot pass through or land on attacked squares
        int direction = king_side ? 1 : -1;
        for (int i = 1; i <= 2; i++) {
            Position passing(from.row, from.col + i * direction);
            if (board.isSquareAttacked(passing, opposite(current_turn))) return false;
        }

        // Execute castling
        int new_rook_col = king_side ? 5 : 3;
        board.movePiece(from, to);
        board.movePiece(rook_pos, Position(from.row, new_rook_col));

        Move move(from, to, king);
        move.is_castling = true;
        move_history.push_back(move);

        current_turn = opposite(current_turn);
        updateGameStatus();
        return true;
    }

    bool tryEnPassant(Position from, Position to) {
        if (move_history.empty()) return false;

        const Move& last_move = move_history.back();
        // En passant: last move was a pawn moving 2 squares, landing beside our pawn
        if (last_move.piece->getType() != PieceType::PAWN) return false;
        if (abs(last_move.to.row - last_move.from.row) != 2) return false;
        if (last_move.to.row != from.row) return false;
        if (last_move.to.col != to.col) return false;

        // Capture the pawn
        Piece* captured = board.getPiece(last_move.to);
        board.removePiece(last_move.to);
        board.movePiece(from, to);

        // Verify king is not in check after en passant
        if (isInCheck(current_turn)) {
            // Undo
            board.movePiece(to, from);
            board.getPiece(from)->set_position(from);
            board.setPiece(last_move.to, captured);
            cout << "En passant leaves king in check!" << endl;
            return false;
        }

        Move move(from, to, board.getPiece(to), captured);
        move.is_en_passant = true;
        move_history.push_back(move);

        current_turn = opposite(current_turn);
        updateGameStatus();
        return true;
    }

    void promotePawn(Position pos, PieceType type) {
        Piece* pawn = board.getPiece(pos);
        Color color = pawn->get_color();
        board.removePiece(pos);

        // Create the promoted piece
        switch (type) {
            case PieceType::QUEEN:
                placePiece<Queen>(color, pos); break;
            case PieceType::ROOK:
                placePiece<Rook>(color, pos); break;
            case PieceType::BISHOP:
                placePiece<Bishop>(color, pos); break;
            case PieceType::KNIGHT:
                placePiece<Knight>(color, pos); break;
            default:
                placePiece<Queen>(color, pos); break;
        }
    }

    void updateGameStatus() {
        if (isCheckmate(current_turn)) {
            status = GameStatus::CHECKMATE;
            cout << color_to_string(opposite(current_turn)) << " wins by checkmate!" << endl;
        } else if (isStalemate(current_turn)) {
            status = GameStatus::STALEMATE;
            cout << "Game drawn by stalemate!" << endl;
        } else if (isInCheck(current_turn)) {
            status = GameStatus::CHECK;
            cout << color_to_string(current_turn) << " is in check!" << endl;
        } else {
            status = GameStatus::ACTIVE;
        }
    }
};
```

**Usage example:**

```cpp
int main() {
    Game game;
    game.display();

    // Scholar's Mate (4-move checkmate)
    game.makeMove(Position(1, 4), Position(3, 4));  // e2-e4
    game.makeMove(Position(6, 4), Position(4, 4));  // e7-e5
    game.makeMove(Position(0, 5), Position(3, 2));  // Bf1-c4
    game.makeMove(Position(7, 1), Position(5, 2));  // Nb8-c6
    game.makeMove(Position(0, 3), Position(4, 7));  // Qd1-h5
    game.makeMove(Position(7, 6), Position(5, 5));  // Ng8-f6
    game.makeMove(Position(4, 7), Position(6, 5));  // Qh5-f7# (checkmate)

    game.display();
    // Output: White wins by checkmate!
    return 0;
}
```

**Design patterns used:**
- **Strategy**: Each piece type encapsulates its own movement strategy via `canMove()`
- **Command**: `Move` struct records each move for history/undo
- **Observer**: Game status updates after each move (could be extended to notify UI)
- **Template Method**: `getPossibleMoves()` uses `canMove()` as a hook method

---


## 14.2 Design a File System (In-Memory)

> An in-memory file system is a classic problem that tests your understanding of tree data structures, the Composite pattern, path resolution, and permission management. The key insight is that files and directories share a common interface (both are "file system entries") but directories can contain children — a textbook Composite pattern. This problem also tests your ability to handle path parsing, recursive operations, and access control.

---

### Requirements

1. **Files**: Store content (text), have a name and size
2. **Directories**: Contain files and other directories (tree structure)
3. **CRUD Operations**: Create, read, update, delete files and directories
4. **Path Resolution**: Navigate using absolute paths like `/home/user/docs/file.txt`
5. **Permissions**: Read, write, execute permissions for owner
6. **Search**: Find files by name or extension
7. **Metadata**: Creation time, modification time, size

---

### Key Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| File/Directory hierarchy | Composite pattern — common `FSEntry` base | Uniform treatment of files and directories |
| Path resolution | Split path by `/`, traverse tree | Standard approach, handles absolute paths |
| Storage | `unordered_map<string, unique_ptr<FSEntry>>` in each directory | O(1) lookup by name within a directory |
| Permissions | Bitmask (rwx) | Simple, efficient, mirrors Unix permissions |

---

### Implementation

**Enums and base class:**

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <memory>
#include <vector>
#include <sstream>
#include <chrono>
#include <algorithm>
#include <functional>
using namespace std;

enum class EntryType { FILE, DIRECTORY };

// Permission bitmask
enum Permission : uint8_t {
    NONE    = 0,
    EXECUTE = 1,  // 001
    WRITE   = 2,  // 010
    READ    = 4,  // 100
    RW      = READ | WRITE,
    RWX     = READ | WRITE | EXECUTE
};

using TimePoint = chrono::system_clock::time_point;

// Abstract base class for file system entries (Composite pattern)
class FSEntry {
protected:
    string name;
    uint8_t permissions;
    TimePoint created_at;
    TimePoint modified_at;
    FSEntry* parent = nullptr;

public:
    FSEntry(const string& name, uint8_t perms = RWX)
        : name(name), permissions(perms),
          created_at(chrono::system_clock::now()),
          modified_at(created_at) {}

    virtual ~FSEntry() = default;

    const string& getName() const { return name; }
    void setName(const string& n) { name = n; touch(); }
    uint8_t getPermissions() const { return permissions; }
    void setPermissions(uint8_t p) { permissions = p; }
    FSEntry* getParent() const { return parent; }
    void setParent(FSEntry* p) { parent = p; }
    TimePoint getCreatedAt() const { return created_at; }
    TimePoint getModifiedAt() const { return modified_at; }

    void touch() { modified_at = chrono::system_clock::now(); }

    bool hasPermission(Permission p) const {
        return (permissions & p) != 0;
    }

    // Get the full path from root to this entry
    string getPath() const {
        if (!parent) return "/";
        string parent_path = parent->getPath();
        if (parent_path == "/") return "/" + name;
        return parent_path + "/" + name;
    }

    virtual EntryType getType() const = 0;
    virtual size_t getSize() const = 0;
    virtual void display(int indent = 0) const = 0;
};
```

**File class:**

```cpp
class File : public FSEntry {
    string content;

public:
    File(const string& name, uint8_t perms = RW)
        : FSEntry(name, perms) {}

    EntryType getType() const override { return EntryType::FILE; }

    size_t getSize() const override { return content.size(); }

    const string& getContent() const {
        if (!hasPermission(READ)) {
            throw runtime_error("Permission denied: cannot read " + getPath());
        }
        return content;
    }

    void setContent(const string& data) {
        if (!hasPermission(WRITE)) {
            throw runtime_error("Permission denied: cannot write " + getPath());
        }
        content = data;
        touch();
    }

    void appendContent(const string& data) {
        if (!hasPermission(WRITE)) {
            throw runtime_error("Permission denied: cannot write " + getPath());
        }
        content += data;
        touch();
    }

    void display(int indent = 0) const override {
        string pad(indent * 2, ' ');
        cout << pad << name << " (" << getSize() << " bytes)" << endl;
    }
};
```

**Directory class:**

```cpp
class Directory : public FSEntry {
    unordered_map<string, unique_ptr<FSEntry>> children;

public:
    Directory(const string& name, uint8_t perms = RWX)
        : FSEntry(name, perms) {}

    EntryType getType() const override { return EntryType::DIRECTORY; }

    // Directory size = sum of all children sizes (recursive)
    size_t getSize() const override {
        size_t total = 0;
        for (const auto& [name, entry] : children) {
            total += entry->getSize();
        }
        return total;
    }

    // Add a child entry
    FSEntry* addChild(unique_ptr<FSEntry> entry) {
        if (!hasPermission(WRITE)) {
            throw runtime_error("Permission denied: cannot write to " + getPath());
        }
        if (children.count(entry->getName())) {
            throw runtime_error("Entry already exists: " + entry->getName());
        }
        entry->setParent(this);
        FSEntry* raw = entry.get();
        children[entry->getName()] = std::move(entry);
        touch();
        return raw;
    }

    // Get a child by name
    FSEntry* getChild(const string& name) const {
        if (!hasPermission(READ)) {
            throw runtime_error("Permission denied: cannot read " + getPath());
        }
        auto it = children.find(name);
        if (it == children.end()) return nullptr;
        return it->second.get();
    }

    // Remove a child by name
    bool removeChild(const string& name) {
        if (!hasPermission(WRITE)) {
            throw runtime_error("Permission denied: cannot write to " + getPath());
        }
        auto it = children.find(name);
        if (it == children.end()) return false;
        children.erase(it);
        touch();
        return true;
    }

    // List all children
    vector<FSEntry*> listChildren() const {
        if (!hasPermission(READ)) {
            throw runtime_error("Permission denied: cannot read " + getPath());
        }
        vector<FSEntry*> result;
        for (const auto& [name, entry] : children) {
            result.push_back(entry.get());
        }
        return result;
    }

    bool hasChild(const string& name) const {
        return children.count(name) > 0;
    }

    void display(int indent = 0) const override {
        string pad(indent * 2, ' ');
        cout << pad << name << "/" << endl;
        for (const auto& [name, entry] : children) {
            entry->display(indent + 1);
        }
    }
};
```

**FileSystem class (facade):**

```cpp
class FileSystem {
    unique_ptr<Directory> root;

public:
    FileSystem() : root(make_unique<Directory>("")) {}

    // Parse a path into components: "/home/user/file.txt" → ["home", "user", "file.txt"]
    static vector<string> parsePath(const string& path) {
        vector<string> parts;
        istringstream iss(path);
        string part;
        while (getline(iss, part, '/')) {
            if (!part.empty() && part != ".") {
                if (part == "..") {
                    if (!parts.empty()) parts.pop_back();
                } else {
                    parts.push_back(part);
                }
            }
        }
        return parts;
    }

    // Resolve a path to an FSEntry
    FSEntry* resolve(const string& path) const {
        auto parts = parsePath(path);
        FSEntry* current = root.get();

        for (const auto& part : parts) {
            if (current->getType() != EntryType::DIRECTORY) {
                return nullptr;  // can't traverse into a file
            }
            auto* dir = static_cast<Directory*>(current);
            current = dir->getChild(part);
            if (!current) return nullptr;
        }
        return current;
    }

    // Resolve parent directory of a path
    Directory* resolveParent(const string& path) const {
        auto parts = parsePath(path);
        if (parts.empty()) return root.get();

        parts.pop_back();  // remove the last component (the entry itself)

        FSEntry* current = root.get();
        for (const auto& part : parts) {
            if (current->getType() != EntryType::DIRECTORY) return nullptr;
            auto* dir = static_cast<Directory*>(current);
            current = dir->getChild(part);
            if (!current) return nullptr;
        }

        if (current->getType() != EntryType::DIRECTORY) return nullptr;
        return static_cast<Directory*>(current);
    }

    // Create a file at the given path
    File* createFile(const string& path, const string& content = "") {
        auto parts = parsePath(path);
        if (parts.empty()) throw runtime_error("Invalid file path");

        string filename = parts.back();
        Directory* parent = resolveParent(path);
        if (!parent) throw runtime_error("Parent directory not found: " + path);

        auto file = make_unique<File>(filename);
        if (!content.empty()) file->setContent(content);
        File* raw = static_cast<File*>(parent->addChild(std::move(file)));
        return raw;
    }

    // Create a directory at the given path
    Directory* createDirectory(const string& path) {
        auto parts = parsePath(path);
        if (parts.empty()) throw runtime_error("Invalid directory path");

        string dirname = parts.back();
        Directory* parent = resolveParent(path);
        if (!parent) throw runtime_error("Parent directory not found: " + path);

        auto dir = make_unique<Directory>(dirname);
        Directory* raw = static_cast<Directory*>(parent->addChild(std::move(dir)));
        return raw;
    }

    // Create directories recursively (like mkdir -p)
    Directory* mkdirp(const string& path) {
        auto parts = parsePath(path);
        Directory* current = root.get();

        for (const auto& part : parts) {
            FSEntry* child = current->getChild(part);
            if (!child) {
                auto dir = make_unique<Directory>(part);
                child = current->addChild(std::move(dir));
            }
            if (child->getType() != EntryType::DIRECTORY) {
                throw runtime_error(part + " exists and is not a directory");
            }
            current = static_cast<Directory*>(child);
        }
        return current;
    }

    // Read file content
    string readFile(const string& path) const {
        FSEntry* entry = resolve(path);
        if (!entry) throw runtime_error("File not found: " + path);
        if (entry->getType() != EntryType::FILE) {
            throw runtime_error("Not a file: " + path);
        }
        return static_cast<File*>(entry)->getContent();
    }

    // Write to a file
    void writeFile(const string& path, const string& content) {
        FSEntry* entry = resolve(path);
        if (!entry) throw runtime_error("File not found: " + path);
        if (entry->getType() != EntryType::FILE) {
            throw runtime_error("Not a file: " + path);
        }
        static_cast<File*>(entry)->setContent(content);
    }

    // Delete a file or directory
    bool remove(const string& path) {
        auto parts = parsePath(path);
        if (parts.empty()) throw runtime_error("Cannot delete root");

        Directory* parent = resolveParent(path);
        if (!parent) return false;

        return parent->removeChild(parts.back());
    }

    // List directory contents
    vector<FSEntry*> ls(const string& path) const {
        FSEntry* entry = resolve(path);
        if (!entry) throw runtime_error("Path not found: " + path);
        if (entry->getType() != EntryType::DIRECTORY) {
            throw runtime_error("Not a directory: " + path);
        }
        return static_cast<Directory*>(entry)->listChildren();
    }

    // Search for files by name (recursive)
    vector<string> find(const string& path, const string& name) const {
        vector<string> results;
        FSEntry* start = resolve(path);
        if (!start) return results;

        findRecursive(start, name, results);
        return results;
    }

    // Get total size of a path
    size_t getSize(const string& path) const {
        FSEntry* entry = resolve(path);
        if (!entry) throw runtime_error("Path not found: " + path);
        return entry->getSize();
    }

    // Display the entire file system tree
    void display() const {
        root->display();
    }

private:
    void findRecursive(FSEntry* entry, const string& name,
                       vector<string>& results) const {
        if (entry->getName() == name || matchWildcard(entry->getName(), name)) {
            results.push_back(entry->getPath());
        }
        if (entry->getType() == EntryType::DIRECTORY) {
            auto* dir = static_cast<Directory*>(entry);
            for (auto* child : dir->listChildren()) {
                findRecursive(child, name, results);
            }
        }
    }

    // Simple wildcard matching: "*.txt" matches "file.txt"
    bool matchWildcard(const string& str, const string& pattern) const {
        if (pattern.empty()) return str.empty();
        if (pattern[0] == '*') {
            string suffix = pattern.substr(1);
            if (suffix.empty()) return true;
            return str.size() >= suffix.size() &&
                   str.substr(str.size() - suffix.size()) == suffix;
        }
        return str == pattern;
    }
};
```

**Usage example:**

```cpp
int main() {
    FileSystem fs;

    // Create directory structure
    fs.mkdirp("/home/user/documents");
    fs.mkdirp("/home/user/pictures");
    fs.mkdirp("/etc/config");

    // Create files
    fs.createFile("/home/user/documents/readme.txt", "Hello, World!");
    fs.createFile("/home/user/documents/notes.txt", "Some notes here");
    fs.createFile("/home/user/pictures/photo.jpg", "<binary data>");
    fs.createFile("/etc/config/app.conf", "port=8080\nhost=localhost");

    // Read a file
    cout << fs.readFile("/home/user/documents/readme.txt") << endl;
    // Output: Hello, World!

    // List directory
    for (auto* entry : fs.ls("/home/user")) {
        cout << entry->getName() << " (" 
             << (entry->getType() == EntryType::DIRECTORY ? "dir" : "file")
             << ")" << endl;
    }
    // Output: documents (dir)
    //         pictures (dir)

    // Search for .txt files
    auto results = fs.find("/", "*.txt");
    for (const auto& path : results) {
        cout << "Found: " << path << endl;
    }
    // Output: Found: /home/user/documents/readme.txt
    //         Found: /home/user/documents/notes.txt

    // Get directory size
    cout << "Size: " << fs.getSize("/home/user") << " bytes" << endl;

    // Display tree
    fs.display();

    return 0;
}
```

**Design patterns used:**
- **Composite**: `FSEntry` base with `File` (leaf) and `Directory` (composite) — uniform treatment
- **Iterator**: `listChildren()` and recursive `find()` traverse the tree
- **Visitor**: Search functionality visits each node in the tree
- **Facade**: `FileSystem` class provides a simplified interface to the tree structure

---


## 14.3 Design a Spreadsheet (Excel)

> A spreadsheet engine is one of the most intellectually challenging LLD problems. It requires modeling cells that can contain raw values or formulas, building a dependency graph between cells, detecting circular dependencies, and efficiently recalculating values when a cell changes. This problem tests your knowledge of graph algorithms (topological sort, cycle detection), the Observer pattern, and expression evaluation.

---

### Requirements

1. **Cells**: Each cell can hold a raw value (number, string) or a formula (e.g., `=A1+B2`)
2. **Formulas**: Support basic arithmetic (`+`, `-`, `*`, `/`) and cell references
3. **Dependency Graph**: Track which cells depend on which other cells
4. **Circular Dependency Detection**: Detect and reject formulas that create cycles
5. **Recalculation**: When a cell changes, all dependent cells are recalculated in correct order
6. **Cell Addressing**: Standard spreadsheet notation (A1, B2, AA1, etc.)

---

### Key Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Cell value storage | `variant<double, string>` | Cells can hold numbers or text |
| Formula parsing | Simple recursive descent parser | Handles arithmetic and cell references |
| Dependency tracking | Adjacency list (DAG) | Efficient for topological sort and cycle detection |
| Recalculation order | Topological sort of dependency graph | Ensures dependencies are calculated before dependents |
| Cycle detection | DFS with coloring (white/gray/black) | Standard algorithm for cycle detection in directed graphs |

---

### Implementation

**Cell addressing utilities:**

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <unordered_set>
#include <vector>
#include <variant>
#include <memory>
#include <sstream>
#include <stdexcept>
#include <stack>
#include <queue>
#include <functional>
#include <cctype>
#include <algorithm>
using namespace std;

// Convert column letter(s) to index: A→0, B→1, ..., Z→25, AA→26
int colToIndex(const string& col) {
    int result = 0;
    for (char c : col) {
        result = result * 26 + (toupper(c) - 'A' + 1);
    }
    return result - 1;
}

// Convert column index to letter(s): 0→A, 1→B, ..., 25→Z, 26→AA
string indexToCol(int index) {
    string result;
    index++;
    while (index > 0) {
        index--;
        result = char('A' + index % 26) + result;
        index /= 26;
    }
    return result;
}

// Cell ID: "A1" → {row=0, col=0}, "B3" → {row=2, col=1}
struct CellId {
    int row, col;

    CellId() : row(0), col(0) {}
    CellId(int r, int c) : row(r), col(c) {}

    // Parse "A1", "B3", "AA10" etc.
    static CellId parse(const string& ref) {
        int i = 0;
        while (i < (int)ref.size() && isalpha(ref[i])) i++;
        string col_str = ref.substr(0, i);
        int row_num = stoi(ref.substr(i)) - 1;  // 1-indexed to 0-indexed
        return CellId(row_num, colToIndex(col_str));
    }

    string toString() const {
        return indexToCol(col) + to_string(row + 1);
    }

    bool operator==(const CellId& other) const {
        return row == other.row && col == other.col;
    }
};

// Hash for CellId (needed for unordered_map/set)
struct CellIdHash {
    size_t operator()(const CellId& id) const {
        return hash<int>()(id.row) ^ (hash<int>()(id.col) << 16);
    }
};
```

**Expression AST and parser:**

```cpp
// AST node for formula expressions
struct Expr {
    virtual ~Expr() = default;
    virtual double evaluate(const function<double(CellId)>& getCellValue) const = 0;
    virtual vector<CellId> getDependencies() const = 0;
};

struct NumberExpr : public Expr {
    double value;
    NumberExpr(double v) : value(v) {}

    double evaluate(const function<double(CellId)>&) const override {
        return value;
    }
    vector<CellId> getDependencies() const override { return {}; }
};

struct CellRefExpr : public Expr {
    CellId cell;
    CellRefExpr(CellId c) : cell(c) {}

    double evaluate(const function<double(CellId)>& getCellValue) const override {
        return getCellValue(cell);
    }
    vector<CellId> getDependencies() const override { return {cell}; }
};

struct BinaryExpr : public Expr {
    char op;
    unique_ptr<Expr> left, right;

    BinaryExpr(char op, unique_ptr<Expr> l, unique_ptr<Expr> r)
        : op(op), left(std::move(l)), right(std::move(r)) {}

    double evaluate(const function<double(CellId)>& getCellValue) const override {
        double l = left->evaluate(getCellValue);
        double r = right->evaluate(getCellValue);
        switch (op) {
            case '+': return l + r;
            case '-': return l - r;
            case '*': return l * r;
            case '/':
                if (r == 0.0) throw runtime_error("Division by zero");
                return l / r;
            default: throw runtime_error("Unknown operator");
        }
    }

    vector<CellId> getDependencies() const override {
        auto deps = left->getDependencies();
        auto right_deps = right->getDependencies();
        deps.insert(deps.end(), right_deps.begin(), right_deps.end());
        return deps;
    }
};

// Simple recursive descent parser for formulas
// Grammar:
//   expr     → term (('+' | '-') term)*
//   term     → factor (('*' | '/') factor)*
//   factor   → NUMBER | CELL_REF | '(' expr ')'
class FormulaParser {
    string input;
    int pos = 0;

    char peek() const { return pos < (int)input.size() ? input[pos] : '\0'; }
    char advance() { return input[pos++]; }

    void skipSpaces() {
        while (pos < (int)input.size() && input[pos] == ' ') pos++;
    }

    unique_ptr<Expr> parseExpr() {
        auto left = parseTerm();
        skipSpaces();
        while (peek() == '+' || peek() == '-') {
            char op = advance();
            auto right = parseTerm();
            left = make_unique<BinaryExpr>(op, std::move(left), std::move(right));
            skipSpaces();
        }
        return left;
    }

    unique_ptr<Expr> parseTerm() {
        auto left = parseFactor();
        skipSpaces();
        while (peek() == '*' || peek() == '/') {
            char op = advance();
            auto right = parseFactor();
            left = make_unique<BinaryExpr>(op, std::move(left), std::move(right));
            skipSpaces();
        }
        return left;
    }

    unique_ptr<Expr> parseFactor() {
        skipSpaces();

        // Parenthesized expression
        if (peek() == '(') {
            advance();  // consume '('
            auto expr = parseExpr();
            skipSpaces();
            if (peek() != ')') throw runtime_error("Expected ')'");
            advance();  // consume ')'
            return expr;
        }

        // Negative number or negation
        if (peek() == '-') {
            advance();
            auto expr = parseFactor();
            return make_unique<BinaryExpr>('-', make_unique<NumberExpr>(0), std::move(expr));
        }

        // Cell reference (starts with letter)
        if (isalpha(peek())) {
            string ref;
            while (isalpha(peek())) ref += advance();
            while (isdigit(peek())) ref += advance();
            return make_unique<CellRefExpr>(CellId::parse(ref));
        }

        // Number
        if (isdigit(peek()) || peek() == '.') {
            string num;
            while (isdigit(peek()) || peek() == '.') num += advance();
            return make_unique<NumberExpr>(stod(num));
        }

        throw runtime_error("Unexpected character: " + string(1, peek()));
    }

public:
    unique_ptr<Expr> parse(const string& formula) {
        input = formula;
        pos = 0;
        auto result = parseExpr();
        skipSpaces();
        if (pos != (int)input.size()) {
            throw runtime_error("Unexpected trailing characters in formula");
        }
        return result;
    }
};
```

**Cell and Spreadsheet classes:**

```cpp
using CellValue = variant<double, string>;

class Cell {
    CellId id;
    string raw_input;                  // what the user typed
    CellValue cached_value;            // computed value
    unique_ptr<Expr> formula;          // parsed formula (nullptr if raw value)
    vector<CellId> dependencies;       // cells this cell depends on

public:
    Cell(CellId id) : id(id), cached_value(0.0) {}

    const CellId& getId() const { return id; }
    const string& getRawInput() const { return raw_input; }
    const CellValue& getValue() const { return cached_value; }

    double getNumericValue() const {
        if (holds_alternative<double>(cached_value)) {
            return get<double>(cached_value);
        }
        return 0.0;  // non-numeric cells evaluate to 0 in formulas
    }

    const vector<CellId>& getDependencies() const { return dependencies; }
    bool hasFormula() const { return formula != nullptr; }

    // Set a raw value (number or string)
    void setRawValue(const string& input) {
        raw_input = input;
        formula.reset();
        dependencies.clear();

        // Try to parse as number
        try {
            size_t pos;
            double val = stod(input, &pos);
            if (pos == input.size()) {
                cached_value = val;
                return;
            }
        } catch (...) {}

        // Otherwise store as string
        cached_value = input;
    }

    // Set a formula (starts with '=')
    void setFormula(const string& input, FormulaParser& parser) {
        raw_input = input;
        string formula_str = input.substr(1);  // remove '='
        formula = parser.parse(formula_str);
        dependencies = formula->getDependencies();
    }

    // Evaluate the formula and update cached value
    void evaluate(const function<double(CellId)>& getCellValue) {
        if (formula) {
            cached_value = formula->evaluate(getCellValue);
        }
    }

    string getDisplayValue() const {
        if (holds_alternative<double>(cached_value)) {
            double val = get<double>(cached_value);
            if (val == (int)val) return to_string((int)val);
            return to_string(val);
        }
        return get<string>(cached_value);
    }
};

class Spreadsheet {
    unordered_map<CellId, unique_ptr<Cell>, CellIdHash> cells;
    // Reverse dependency map: cell → set of cells that depend on it
    unordered_map<CellId, unordered_set<CellId, CellIdHash>, CellIdHash> dependents;
    FormulaParser parser;

public:
    // Set a cell's value or formula
    void setCell(const string& ref, const string& input) {
        CellId id = CellId::parse(ref);

        // Get or create the cell
        if (!cells.count(id)) {
            cells[id] = make_unique<Cell>(id);
        }
        Cell* cell = cells[id].get();

        // Remove old dependencies
        for (const auto& dep : cell->getDependencies()) {
            dependents[dep].erase(id);
        }

        // Parse new input
        if (!input.empty() && input[0] == '=') {
            cell->setFormula(input, parser);
        } else {
            cell->setRawValue(input);
        }

        // Check for circular dependencies
        if (cell->hasFormula() && hasCircularDependency(id)) {
            // Rollback
            cell->setRawValue("0");
            throw runtime_error("Circular dependency detected for " + ref);
        }

        // Register new dependencies
        for (const auto& dep : cell->getDependencies()) {
            dependents[dep].insert(id);
        }

        // Recalculate this cell and all dependents
        recalculate(id);
    }

    // Get a cell's display value
    string getCell(const string& ref) const {
        CellId id = CellId::parse(ref);
        auto it = cells.find(id);
        if (it == cells.end()) return "";
        return it->second->getDisplayValue();
    }

    // Get a cell's numeric value (for formula evaluation)
    double getCellNumericValue(CellId id) const {
        auto it = cells.find(id);
        if (it == cells.end()) return 0.0;
        return it->second->getNumericValue();
    }

    // Display the spreadsheet
    void display(int rows = 5, int cols = 5) const {
        // Header
        cout << "     ";
        for (int c = 0; c < cols; c++) {
            cout << indexToCol(c) << "      ";
        }
        cout << endl;

        for (int r = 0; r < rows; r++) {
            cout << (r + 1) << "    ";
            for (int c = 0; c < cols; c++) {
                CellId id(r, c);
                auto it = cells.find(id);
                string val = (it != cells.end()) ? it->second->getDisplayValue() : "";
                // Pad to 7 chars
                if (val.size() > 7) val = val.substr(0, 7);
                cout << val;
                for (int p = val.size(); p < 7; p++) cout << " ";
            }
            cout << endl;
        }
    }

private:
    // Detect circular dependencies using DFS with coloring
    bool hasCircularDependency(CellId start) const {
        enum class Color { WHITE, GRAY, BLACK };
        unordered_map<CellId, Color, CellIdHash> color;

        // Initialize all cells as WHITE
        function<bool(CellId)> dfs = [&](CellId id) -> bool {
            color[id] = Color::GRAY;

            auto it = cells.find(id);
            if (it != cells.end()) {
                for (const auto& dep : it->second->getDependencies()) {
                    auto dep_color = color.count(dep) ? color[dep] : Color::WHITE;
                    if (dep_color == Color::GRAY) return true;   // back edge → cycle
                    if (dep_color == Color::WHITE && dfs(dep)) return true;
                }
            }

            color[id] = Color::BLACK;
            return false;
        };

        return dfs(start);
    }

    // Recalculate a cell and all its dependents in topological order
    void recalculate(CellId start) {
        // BFS to find all affected cells
        vector<CellId> order;
        unordered_set<CellId, CellIdHash> visited;
        queue<CellId> q;

        q.push(start);
        visited.insert(start);

        while (!q.empty()) {
            CellId current = q.front();
            q.pop();
            order.push_back(current);

            if (dependents.count(current)) {
                for (const auto& dep : dependents.at(current)) {
                    if (!visited.count(dep)) {
                        visited.insert(dep);
                        q.push(dep);
                    }
                }
            }
        }

        // Evaluate in order (start first, then its dependents)
        auto getCellValue = [this](CellId id) -> double {
            return getCellNumericValue(id);
        };

        for (const auto& id : order) {
            auto it = cells.find(id);
            if (it != cells.end() && it->second->hasFormula()) {
                it->second->evaluate(getCellValue);
            }
        }
    }
};
```

**Usage example:**

```cpp
int main() {
    Spreadsheet sheet;

    // Set raw values
    sheet.setCell("A1", "10");
    sheet.setCell("A2", "20");
    sheet.setCell("A3", "30");

    // Set formulas
    sheet.setCell("B1", "=A1+A2");       // B1 = 10 + 20 = 30
    sheet.setCell("B2", "=A1*A3");       // B2 = 10 * 30 = 300
    sheet.setCell("C1", "=B1+B2");       // C1 = 30 + 300 = 330

    cout << "B1 = " << sheet.getCell("B1") << endl;  // 30
    cout << "B2 = " << sheet.getCell("B2") << endl;  // 300
    cout << "C1 = " << sheet.getCell("C1") << endl;  // 330

    // Change A1 — triggers recalculation of B1, B2, and C1
    sheet.setCell("A1", "100");
    cout << "\nAfter changing A1 to 100:" << endl;
    cout << "B1 = " << sheet.getCell("B1") << endl;  // 120
    cout << "B2 = " << sheet.getCell("B2") << endl;  // 3000
    cout << "C1 = " << sheet.getCell("C1") << endl;  // 3120

    // Circular dependency detection
    try {
        sheet.setCell("A1", "=C1");  // A1 depends on C1, which depends on B1, which depends on A1
    } catch (const runtime_error& e) {
        cout << "\nError: " << e.what() << endl;
        // Output: Error: Circular dependency detected for A1
    }

    sheet.display();
    return 0;
}
```

**Design patterns used:**
- **Observer**: When a cell changes, all dependent cells are notified and recalculated
- **Composite**: Expression AST — `BinaryExpr` contains child `Expr` nodes
- **Memento**: Could be extended to support undo by saving cell states before changes
- **Strategy**: Different expression types evaluate differently (number, cell ref, binary op)

---


## 14.4 Design a Text Editor

> A text editor is a rich LLD problem that combines the Command pattern (for undo/redo), the Memento pattern (for state snapshots), cursor management, clipboard operations, and text search. The key challenge is designing an efficient undo/redo system that can handle arbitrary sequences of operations while keeping memory usage reasonable. This problem also tests your ability to separate concerns between the document model, the command system, and the user interface.

---

### Requirements

1. **Text Operations**: Insert, delete, and replace text at cursor position
2. **Cursor**: Move left, right, to start, to end; track current position
3. **Undo/Redo**: Unlimited undo/redo using Command pattern
4. **Clipboard**: Cut, copy, paste operations
5. **Find and Replace**: Search for text, replace occurrences
6. **Selection**: Select a range of text for operations

---

### Key Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Text storage | `std::string` (simple) or rope (advanced) | String is simple for interviews; rope for large files |
| Undo/Redo | Command pattern with inverse operations | Each command knows how to undo itself |
| Clipboard | Singleton clipboard shared across operations | Standard editor behavior |
| Cursor | Integer offset into the text buffer | Simple, efficient for string operations |

---

### Implementation

**Document (the text buffer):**

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <stack>
#include <memory>
#include <algorithm>
#include <optional>
using namespace std;

class Document {
    string content;
    int cursor_pos = 0;
    int selection_start = -1;  // -1 means no selection
    int selection_end = -1;

public:
    // --- Content access ---
    const string& getContent() const { return content; }
    int length() const { return content.size(); }

    // --- Cursor operations ---
    int getCursorPos() const { return cursor_pos; }

    void setCursorPos(int pos) {
        cursor_pos = max(0, min(pos, (int)content.size()));
        clearSelection();
    }

    void moveCursorLeft(int n = 1) { setCursorPos(cursor_pos - n); }
    void moveCursorRight(int n = 1) { setCursorPos(cursor_pos + n); }
    void moveCursorToStart() { setCursorPos(0); }
    void moveCursorToEnd() { setCursorPos(content.size()); }

    // --- Selection ---
    void setSelection(int start, int end) {
        selection_start = max(0, min(start, (int)content.size()));
        selection_end = max(0, min(end, (int)content.size()));
        if (selection_start > selection_end) swap(selection_start, selection_end);
    }

    void clearSelection() {
        selection_start = selection_end = -1;
    }

    bool hasSelection() const {
        return selection_start >= 0 && selection_end >= 0 && selection_start != selection_end;
    }

    pair<int, int> getSelection() const {
        return {selection_start, selection_end};
    }

    string getSelectedText() const {
        if (!hasSelection()) return "";
        return content.substr(selection_start, selection_end - selection_start);
    }

    // --- Text modification (low-level, used by commands) ---
    void insertAt(int pos, const string& text) {
        pos = max(0, min(pos, (int)content.size()));
        content.insert(pos, text);
    }

    string deleteRange(int start, int length) {
        start = max(0, min(start, (int)content.size()));
        length = min(length, (int)content.size() - start);
        string deleted = content.substr(start, length);
        content.erase(start, length);
        return deleted;
    }

    // --- Search ---
    int find(const string& query, int from_pos = 0) const {
        size_t pos = content.find(query, from_pos);
        return (pos == string::npos) ? -1 : (int)pos;
    }

    vector<int> findAll(const string& query) const {
        vector<int> positions;
        size_t pos = 0;
        while ((pos = content.find(query, pos)) != string::npos) {
            positions.push_back((int)pos);
            pos += query.size();
        }
        return positions;
    }

    void display() const {
        cout << content << endl;
        // Show cursor position
        string cursor_line(cursor_pos, ' ');
        cursor_line += '^';
        cout << cursor_line << " (pos " << cursor_pos << ")" << endl;
    }
};
```

**Command interface and concrete commands:**

```cpp
// Command interface — each command can execute and undo
class Command {
public:
    virtual ~Command() = default;
    virtual void execute() = 0;
    virtual void undo() = 0;
    virtual string describe() const = 0;
};

// Insert text at a position
class InsertCommand : public Command {
    Document& doc;
    int position;
    string text;

public:
    InsertCommand(Document& doc, int pos, const string& text)
        : doc(doc), position(pos), text(text) {}

    void execute() override {
        doc.insertAt(position, text);
        doc.setCursorPos(position + text.size());
    }

    void undo() override {
        doc.deleteRange(position, text.size());
        doc.setCursorPos(position);
    }

    string describe() const override {
        return "Insert \"" + text + "\" at " + to_string(position);
    }
};

// Delete text at a position
class DeleteCommand : public Command {
    Document& doc;
    int position;
    int length;
    string deleted_text;  // saved for undo

public:
    DeleteCommand(Document& doc, int pos, int len)
        : doc(doc), position(pos), length(len) {}

    void execute() override {
        deleted_text = doc.deleteRange(position, length);
        doc.setCursorPos(position);
    }

    void undo() override {
        doc.insertAt(position, deleted_text);
        doc.setCursorPos(position + deleted_text.size());
    }

    string describe() const override {
        return "Delete " + to_string(length) + " chars at " + to_string(position)
               + " (\"" + deleted_text + "\")";
    }
};

// Replace text (delete + insert as a single undoable operation)
class ReplaceCommand : public Command {
    Document& doc;
    int position;
    string old_text;
    string new_text;

public:
    ReplaceCommand(Document& doc, int pos, const string& old_t, const string& new_t)
        : doc(doc), position(pos), old_text(old_t), new_text(new_t) {}

    void execute() override {
        doc.deleteRange(position, old_text.size());
        doc.insertAt(position, new_text);
        doc.setCursorPos(position + new_text.size());
    }

    void undo() override {
        doc.deleteRange(position, new_text.size());
        doc.insertAt(position, old_text);
        doc.setCursorPos(position + old_text.size());
    }

    string describe() const override {
        return "Replace \"" + old_text + "\" with \"" + new_text + "\" at "
               + to_string(position);
    }
};
```

**Clipboard:**

```cpp
class Clipboard {
    string content;

    Clipboard() = default;

public:
    static Clipboard& instance() {
        static Clipboard clip;
        return clip;
    }

    void set(const string& text) { content = text; }
    const string& get() const { return content; }
    bool empty() const { return content.empty(); }

    Clipboard(const Clipboard&) = delete;
    Clipboard& operator=(const Clipboard&) = delete;
};
```

**TextEditor (orchestrator with undo/redo):**

```cpp
class TextEditor {
    Document doc;
    vector<unique_ptr<Command>> undo_stack;
    vector<unique_ptr<Command>> redo_stack;

    void executeCommand(unique_ptr<Command> cmd) {
        cmd->execute();
        undo_stack.push_back(std::move(cmd));
        redo_stack.clear();  // new action invalidates redo history
    }

public:
    // --- Basic text operations ---
    void insertText(const string& text) {
        executeCommand(make_unique<InsertCommand>(doc, doc.getCursorPos(), text));
    }

    void deleteBackward(int n = 1) {
        int pos = doc.getCursorPos();
        int actual = min(n, pos);
        if (actual > 0) {
            executeCommand(make_unique<DeleteCommand>(doc, pos - actual, actual));
        }
    }

    void deleteForward(int n = 1) {
        int pos = doc.getCursorPos();
        int actual = min(n, doc.length() - pos);
        if (actual > 0) {
            executeCommand(make_unique<DeleteCommand>(doc, pos, actual));
        }
    }

    void deleteSelection() {
        if (doc.hasSelection()) {
            auto [start, end] = doc.getSelection();
            executeCommand(make_unique<DeleteCommand>(doc, start, end - start));
        }
    }

    // --- Cursor movement ---
    void moveCursorLeft(int n = 1) { doc.moveCursorLeft(n); }
    void moveCursorRight(int n = 1) { doc.moveCursorRight(n); }
    void moveCursorToStart() { doc.moveCursorToStart(); }
    void moveCursorToEnd() { doc.moveCursorToEnd(); }

    // --- Selection ---
    void select(int start, int end) { doc.setSelection(start, end); }
    void selectAll() { doc.setSelection(0, doc.length()); }

    // --- Clipboard operations ---
    void copy() {
        if (doc.hasSelection()) {
            Clipboard::instance().set(doc.getSelectedText());
        }
    }

    void cut() {
        if (doc.hasSelection()) {
            Clipboard::instance().set(doc.getSelectedText());
            deleteSelection();
        }
    }

    void paste() {
        const string& clip = Clipboard::instance().get();
        if (!clip.empty()) {
            if (doc.hasSelection()) {
                auto [start, end] = doc.getSelection();
                string old_text = doc.getSelectedText();
                executeCommand(make_unique<ReplaceCommand>(doc, start, old_text, clip));
            } else {
                insertText(clip);
            }
        }
    }

    // --- Undo / Redo ---
    void undo() {
        if (undo_stack.empty()) {
            cout << "Nothing to undo" << endl;
            return;
        }
        auto cmd = std::move(undo_stack.back());
        undo_stack.pop_back();
        cmd->undo();
        redo_stack.push_back(std::move(cmd));
    }

    void redo() {
        if (redo_stack.empty()) {
            cout << "Nothing to redo" << endl;
            return;
        }
        auto cmd = std::move(redo_stack.back());
        redo_stack.pop_back();
        cmd->execute();
        undo_stack.push_back(std::move(cmd));
    }

    // --- Find and Replace ---
    int find(const string& query) {
        int pos = doc.find(query, doc.getCursorPos());
        if (pos >= 0) {
            doc.setCursorPos(pos);
            doc.setSelection(pos, pos + query.size());
            cout << "Found at position " << pos << endl;
        } else {
            cout << "Not found: \"" << query << "\"" << endl;
        }
        return pos;
    }

    int replaceNext(const string& query, const string& replacement) {
        int pos = doc.find(query, doc.getCursorPos());
        if (pos >= 0) {
            executeCommand(make_unique<ReplaceCommand>(doc, pos, query, replacement));
            return pos;
        }
        return -1;
    }

    int replaceAll(const string& query, const string& replacement) {
        auto positions = doc.findAll(query);
        if (positions.empty()) return 0;

        // Replace from end to start to preserve positions
        int count = 0;
        for (int i = positions.size() - 1; i >= 0; i--) {
            executeCommand(make_unique<ReplaceCommand>(
                doc, positions[i], query, replacement));
            count++;
        }
        return count;
    }

    // --- Display ---
    void display() const {
        doc.display();
    }

    string getContent() const { return doc.getContent(); }
    int getCursorPos() const { return doc.getCursorPos(); }
};
```

**Usage example:**

```cpp
int main() {
    TextEditor editor;

    // Type some text
    editor.insertText("Hello World");
    editor.display();
    // Output: Hello World
    //                    ^ (pos 11)

    // Move cursor and insert
    editor.moveCursorLeft(6);
    editor.insertText("Beautiful ");
    editor.display();
    // Output: Hello Beautiful World
    //                         ^ (pos 15)

    // Undo the insert
    editor.undo();
    editor.display();
    // Output: Hello World
    //              ^ (pos 5)

    // Redo
    editor.redo();
    editor.display();
    // Output: Hello Beautiful World
    //                         ^ (pos 15)

    // Select and copy
    editor.select(6, 15);
    editor.copy();

    // Move to end and paste
    editor.moveCursorToEnd();
    editor.insertText(" - ");
    editor.paste();
    editor.display();
    // Output: Hello Beautiful World - Beautiful
    //                                          ^ (pos 34)

    // Find and replace
    int count = editor.replaceAll("Beautiful", "Amazing");
    cout << "Replaced " << count << " occurrences" << endl;
    editor.display();
    // Output: Hello Amazing World - Amazing

    // Multiple undos
    editor.undo();  // undo second replace
    editor.undo();  // undo first replace
    editor.display();
    // Output: Hello Beautiful World - Beautiful

    return 0;
}
```

**Design patterns used:**
- **Command**: Each text operation is encapsulated as a command with `execute()` and `undo()`
- **Memento**: Commands save the state needed to undo (deleted text, old text for replace)
- **Singleton**: Clipboard is a singleton shared across operations
- **Iterator**: Find operations iterate through the text buffer

---


## 14.5 Design a Card Game Framework

> A card game framework tests your ability to design extensible, reusable systems. The challenge is creating a generic framework that can support multiple card games (Poker, Blackjack, UNO) without modification. This requires the Template Method pattern (define the game loop skeleton, let subclasses fill in rules), the Strategy pattern (interchangeable scoring and dealing strategies), and the Factory pattern (create game-specific components). The key is finding the right level of abstraction — too specific and it only works for one game, too generic and it's useless.

---

### Requirements

1. **Deck**: Standard 52-card deck (extensible for UNO, custom decks)
2. **Cards**: Suit + Rank (extensible for special cards)
3. **Players**: Hand of cards, name, score
4. **Game Loop**: Deal → Play rounds → Determine winner (Template Method)
5. **Turn Management**: Players take turns in order
6. **Extensibility**: Easy to add new games (Poker, Blackjack, UNO) without modifying the framework

---

### Key Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Game loop | Template Method pattern | Skeleton is same for all games; rules differ |
| Card creation | Factory pattern | Different games need different decks |
| Scoring | Strategy pattern | Each game has unique scoring rules |
| Hand evaluation | Polymorphism | Poker hand vs Blackjack hand vs UNO hand |

---

### Implementation

**Card and Deck:**

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <algorithm>
#include <random>
#include <functional>
#include <unordered_map>
using namespace std;

enum class Suit { HEARTS, DIAMONDS, CLUBS, SPADES, NONE };
enum class Rank {
    ACE = 1, TWO, THREE, FOUR, FIVE, SIX, SEVEN,
    EIGHT, NINE, TEN, JACK, QUEEN, KING,
    // Special ranks for games like UNO
    SKIP, REVERSE, DRAW_TWO, WILD, WILD_DRAW_FOUR
};

string suit_to_string(Suit s) {
    switch (s) {
        case Suit::HEARTS:   return "Hearts";
        case Suit::DIAMONDS: return "Diamonds";
        case Suit::CLUBS:    return "Clubs";
        case Suit::SPADES:   return "Spades";
        default:             return "";
    }
}

string rank_to_string(Rank r) {
    switch (r) {
        case Rank::ACE:   return "A";
        case Rank::TWO:   return "2";
        case Rank::THREE: return "3";
        case Rank::FOUR:  return "4";
        case Rank::FIVE:  return "5";
        case Rank::SIX:   return "6";
        case Rank::SEVEN: return "7";
        case Rank::EIGHT: return "8";
        case Rank::NINE:  return "9";
        case Rank::TEN:   return "10";
        case Rank::JACK:  return "J";
        case Rank::QUEEN: return "Q";
        case Rank::KING:  return "K";
        default:          return "?";
    }
}

class Card {
    Suit suit;
    Rank rank;
    bool face_up = false;

public:
    Card(Suit s, Rank r) : suit(s), rank(r) {}
    virtual ~Card() = default;

    Suit getSuit() const { return suit; }
    Rank getRank() const { return rank; }
    bool isFaceUp() const { return face_up; }
    void flip() { face_up = !face_up; }
    void setFaceUp(bool up) { face_up = up; }

    virtual int getValue() const { return static_cast<int>(rank); }

    virtual string toString() const {
        if (!face_up) return "[??]";
        return "[" + rank_to_string(rank) + " " + suit_to_string(suit) + "]";
    }
};

class Deck {
    vector<unique_ptr<Card>> cards;
    mt19937 rng;

public:
    Deck() : rng(random_device{}()) {}

    void addCard(unique_ptr<Card> card) {
        cards.push_back(std::move(card));
    }

    void shuffle() {
        std::shuffle(cards.begin(), cards.end(), rng);
    }

    unique_ptr<Card> drawCard() {
        if (cards.empty()) throw runtime_error("Deck is empty");
        auto card = std::move(cards.back());
        cards.pop_back();
        return card;
    }

    bool isEmpty() const { return cards.empty(); }
    int remaining() const { return cards.size(); }

    // Factory method for standard 52-card deck
    static unique_ptr<Deck> createStandardDeck() {
        auto deck = make_unique<Deck>();
        for (int s = 0; s < 4; s++) {
            for (int r = 1; r <= 13; r++) {
                deck->addCard(make_unique<Card>(
                    static_cast<Suit>(s), static_cast<Rank>(r)));
            }
        }
        return deck;
    }
};
```

**Player and Hand:**

```cpp
class Hand {
    vector<unique_ptr<Card>> cards;

public:
    void addCard(unique_ptr<Card> card) {
        cards.push_back(std::move(card));
    }

    unique_ptr<Card> removeCard(int index) {
        if (index < 0 || index >= (int)cards.size()) {
            throw runtime_error("Invalid card index");
        }
        auto card = std::move(cards[index]);
        cards.erase(cards.begin() + index);
        return card;
    }

    const vector<unique_ptr<Card>>& getCards() const { return cards; }
    int size() const { return cards.size(); }
    bool isEmpty() const { return cards.empty(); }

    void clear() { cards.clear(); }

    void showAll() {
        for (auto& card : cards) card->setFaceUp(true);
    }

    string toString() const {
        string result;
        for (const auto& card : cards) {
            result += card->toString() + " ";
        }
        return result;
    }
};

class Player {
    string name;
    Hand hand;
    int score = 0;

public:
    Player(const string& name) : name(name) {}

    const string& getName() const { return name; }
    Hand& getHand() { return hand; }
    const Hand& getHand() const { return hand; }
    int getScore() const { return score; }
    void setScore(int s) { score = s; }
    void addScore(int s) { score += s; }

    void receiveCard(unique_ptr<Card> card) {
        hand.addCard(std::move(card));
    }

    unique_ptr<Card> playCard(int index) {
        return hand.removeCard(index);
    }
};
```

**Abstract Game class (Template Method):**

```cpp
class CardGame {
protected:
    vector<unique_ptr<Player>> players;
    unique_ptr<Deck> deck;
    int current_player_index = 0;
    bool game_over = false;

public:
    virtual ~CardGame() = default;

    void addPlayer(const string& name) {
        players.push_back(make_unique<Player>(name));
    }

    // Template Method — defines the game loop skeleton
    void play() {
        cout << "=== Starting " << getGameName() << " ===" << endl;

        // Step 1: Initialize the deck
        initializeDeck();

        // Step 2: Shuffle
        deck->shuffle();

        // Step 3: Deal cards to players
        dealCards();

        // Step 4: Play rounds until game is over
        while (!isGameOver()) {
            Player& current = *players[current_player_index];
            cout << "\n--- " << current.getName() << "'s turn ---" << endl;
            displayGameState(current);

            // Each player takes a turn
            playTurn(current);

            // Check win condition after each turn
            if (checkWinCondition()) {
                game_over = true;
                break;
            }

            // Move to next player
            nextPlayer();
        }

        // Step 5: Determine and announce winner
        announceWinner();
    }

protected:
    // Hook methods — subclasses override these
    virtual string getGameName() const = 0;
    virtual void initializeDeck() = 0;
    virtual void dealCards() = 0;
    virtual void playTurn(Player& player) = 0;
    virtual bool checkWinCondition() = 0;
    virtual void announceWinner() = 0;
    virtual void displayGameState(const Player& player) = 0;

    virtual bool isGameOver() const { return game_over; }

    virtual void nextPlayer() {
        current_player_index = (current_player_index + 1) % players.size();
    }

    // Helper: deal n cards to each player
    void dealToAll(int n) {
        for (int i = 0; i < n; i++) {
            for (auto& player : players) {
                if (!deck->isEmpty()) {
                    auto card = deck->drawCard();
                    card->setFaceUp(true);
                    player->receiveCard(std::move(card));
                }
            }
        }
    }
};
```

**Blackjack implementation:**

```cpp
class BlackjackGame : public CardGame {
    static const int TARGET = 21;

    int handValue(const Hand& hand) const {
        int total = 0;
        int aces = 0;

        for (const auto& card : hand.getCards()) {
            int val = card->getValue();
            if (val == 1) {
                aces++;
                total += 11;
            } else if (val >= 10) {
                total += 10;
            } else {
                total += val;
            }
        }

        // Reduce aces from 11 to 1 if over 21
        while (total > TARGET && aces > 0) {
            total -= 10;
            aces--;
        }
        return total;
    }

    bool isBust(const Hand& hand) const {
        return handValue(hand) > TARGET;
    }

protected:
    string getGameName() const override { return "Blackjack"; }

    void initializeDeck() override {
        deck = Deck::createStandardDeck();
    }

    void dealCards() override {
        dealToAll(2);  // 2 cards each
    }

    void playTurn(Player& player) override {
        // Simple AI: hit if under 17, stand otherwise
        while (handValue(player.getHand()) < 17 && !deck->isEmpty()) {
            cout << player.getName() << " hits!" << endl;
            auto card = deck->drawCard();
            card->setFaceUp(true);
            cout << "  Drew: " << card->toString() << endl;
            player.receiveCard(std::move(card));

            if (isBust(player.getHand())) {
                cout << player.getName() << " BUSTS with "
                     << handValue(player.getHand()) << "!" << endl;
                return;
            }
        }
        cout << player.getName() << " stands with "
             << handValue(player.getHand()) << endl;
    }

    bool checkWinCondition() override {
        // Game ends after all players have played one round
        return current_player_index == (int)players.size() - 1;
    }

    void displayGameState(const Player& player) override {
        cout << "Hand: " << player.getHand().toString()
             << " (Value: " << handValue(player.getHand()) << ")" << endl;
    }

    void announceWinner() override {
        cout << "\n=== Final Results ===" << endl;

        int best_score = -1;
        string winner;

        for (auto& player : players) {
            int val = handValue(player->getHand());
            cout << player->getName() << ": " << player->getHand().toString()
                 << " = " << val;

            if (isBust(player->getHand())) {
                cout << " (BUST)" << endl;
            } else {
                cout << endl;
                if (val > best_score) {
                    best_score = val;
                    winner = player->getName();
                }
            }
        }

        if (best_score > 0) {
            cout << "\nWinner: " << winner << " with " << best_score << "!" << endl;
        } else {
            cout << "\nEveryone busted! No winner." << endl;
        }
    }
};
```

**Usage example:**

```cpp
int main() {
    // Play Blackjack
    BlackjackGame blackjack;
    blackjack.addPlayer("Alice");
    blackjack.addPlayer("Bob");
    blackjack.addPlayer("Charlie");
    blackjack.play();

    // To add a new game (e.g., Poker), just create a new subclass:
    // class PokerGame : public CardGame { ... };
    // The game loop (play()) stays the same — only the hook methods change.

    return 0;
}
```

**Design patterns used:**
- **Template Method**: `CardGame::play()` defines the game loop; subclasses override hook methods
- **Strategy**: Scoring logic varies per game (Blackjack hand value vs Poker hand ranking)
- **Factory**: `Deck::createStandardDeck()` creates game-specific decks
- **Iterator**: Iterating through cards in hand, players in game

---


## 14.6 Design an LRU Cache

> An LRU (Least Recently Used) Cache is a data structure that evicts the least recently accessed item when the cache is full. It's one of the most frequently asked LLD problems because it tests your knowledge of data structures (HashMap + Doubly Linked List), algorithmic complexity (O(1) for both get and put), and concurrency (thread-safe version). The key insight is combining a hash map for O(1) lookup with a doubly linked list for O(1) insertion/removal to maintain access order.

---

### Requirements

1. **O(1) Get**: Retrieve a value by key in constant time
2. **O(1) Put**: Insert or update a key-value pair in constant time
3. **Eviction**: When capacity is exceeded, evict the least recently used item
4. **Access Order**: Any get or put operation marks the item as most recently used
5. **Thread Safety**: Support concurrent access (bonus)

---

### Key Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Data structure | HashMap + Doubly Linked List | O(1) lookup + O(1) order maintenance |
| List ordering | Most recent at head, least recent at tail | Evict from tail in O(1) |
| Thread safety | `std::mutex` with `lock_guard` | Simple, correct; can optimize with read-write lock |

---

### Implementation

**Doubly Linked List Node:**

```cpp
#include <iostream>
#include <unordered_map>
#include <mutex>
#include <optional>
#include <stdexcept>
using namespace std;

template <typename K, typename V>
struct Node {
    K key;
    V value;
    Node* prev = nullptr;
    Node* next = nullptr;

    Node(K k, V v) : key(k), value(v) {}
};
```

**LRU Cache (non-thread-safe version):**

```cpp
template <typename K, typename V>
class LRUCache {
    int capacity;
    unordered_map<K, Node<K, V>*> cache;  // key → node pointer

    // Doubly linked list with sentinel nodes
    Node<K, V>* head;  // dummy head (most recent side)
    Node<K, V>* tail;  // dummy tail (least recent side)

public:
    LRUCache(int capacity) : capacity(capacity) {
        if (capacity <= 0) throw invalid_argument("Capacity must be positive");

        // Sentinel nodes simplify edge cases (no null checks)
        head = new Node<K, V>(K{}, V{});
        tail = new Node<K, V>(K{}, V{});
        head->next = tail;
        tail->prev = head;
    }

    ~LRUCache() {
        Node<K, V>* current = head;
        while (current) {
            Node<K, V>* next = current->next;
            delete current;
            current = next;
        }
    }

    // Get value by key — O(1)
    optional<V> get(const K& key) {
        auto it = cache.find(key);
        if (it == cache.end()) return nullopt;  // cache miss

        Node<K, V>* node = it->second;
        // Move to front (most recently used)
        removeNode(node);
        addToFront(node);

        return node->value;
    }

    // Put key-value pair — O(1)
    void put(const K& key, const V& value) {
        auto it = cache.find(key);

        if (it != cache.end()) {
            // Key exists — update value and move to front
            Node<K, V>* node = it->second;
            node->value = value;
            removeNode(node);
            addToFront(node);
        } else {
            // New key — check capacity
            if ((int)cache.size() >= capacity) {
                evict();
            }

            // Create new node and add to front
            Node<K, V>* node = new Node<K, V>(key, value);
            cache[key] = node;
            addToFront(node);
        }
    }

    // Check if key exists — O(1) (does NOT update access order)
    bool contains(const K& key) const {
        return cache.count(key) > 0;
    }

    // Remove a specific key — O(1)
    bool remove(const K& key) {
        auto it = cache.find(key);
        if (it == cache.end()) return false;

        Node<K, V>* node = it->second;
        removeNode(node);
        cache.erase(it);
        delete node;
        return true;
    }

    int size() const { return cache.size(); }
    bool empty() const { return cache.empty(); }

    // Display cache contents (most recent first)
    void display() const {
        Node<K, V>* current = head->next;
        cout << "Cache [" << cache.size() << "/" << capacity << "]: ";
        while (current != tail) {
            cout << "(" << current->key << ":" << current->value << ") ";
            current = current->next;
        }
        cout << endl;
    }

private:
    // Remove a node from the linked list (does NOT delete it)
    void removeNode(Node<K, V>* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
    }

    // Add a node right after head (most recent position)
    void addToFront(Node<K, V>* node) {
        node->next = head->next;
        node->prev = head;
        head->next->prev = node;
        head->next = node;
    }

    // Evict the least recently used item (node before tail)
    void evict() {
        Node<K, V>* lru = tail->prev;
        if (lru == head) return;  // empty cache

        removeNode(lru);
        cache.erase(lru->key);
        delete lru;
    }
};
```

**Thread-safe version:**

```cpp
template <typename K, typename V>
class ThreadSafeLRUCache {
    LRUCache<K, V> cache;
    mutable mutex mtx;

public:
    ThreadSafeLRUCache(int capacity) : cache(capacity) {}

    optional<V> get(const K& key) {
        lock_guard<mutex> lock(mtx);
        return cache.get(key);
    }

    void put(const K& key, const V& value) {
        lock_guard<mutex> lock(mtx);
        cache.put(key, value);
    }

    bool contains(const K& key) const {
        lock_guard<mutex> lock(mtx);
        return cache.contains(key);
    }

    bool remove(const K& key) {
        lock_guard<mutex> lock(mtx);
        return cache.remove(key);
    }

    int size() const {
        lock_guard<mutex> lock(mtx);
        return cache.size();
    }
};
```

**Usage example:**

```cpp
int main() {
    LRUCache<int, string> cache(3);

    cache.put(1, "one");
    cache.put(2, "two");
    cache.put(3, "three");
    cache.display();
    // Cache [3/3]: (3:three) (2:two) (1:one)

    // Access key 1 — moves it to front
    auto val = cache.get(1);
    if (val) cout << "Got: " << *val << endl;  // "one"
    cache.display();
    // Cache [3/3]: (1:one) (3:three) (2:two)

    // Add key 4 — evicts key 2 (least recently used)
    cache.put(4, "four");
    cache.display();
    // Cache [3/3]: (4:four) (1:one) (3:three)

    // Key 2 was evicted
    auto missing = cache.get(2);
    cout << "Key 2: " << (missing ? *missing : "NOT FOUND") << endl;
    // Output: Key 2: NOT FOUND

    // Update existing key
    cache.put(3, "THREE");
    cache.display();
    // Cache [3/3]: (3:THREE) (4:four) (1:one)

    return 0;
}
```

**Complexity analysis:**

| Operation | Time | Space |
|-----------|------|-------|
| `get(key)` | O(1) | — |
| `put(key, value)` | O(1) | O(1) per entry |
| `remove(key)` | O(1) | — |
| `evict()` | O(1) | — |
| Total space | — | O(capacity) |

**Why HashMap + Doubly Linked List?**
- **HashMap** gives O(1) lookup by key
- **Doubly Linked List** gives O(1) insertion and removal (given a pointer to the node)
- The HashMap stores pointers to list nodes, so after finding a node by key, we can move it to the front in O(1)
- Sentinel nodes (dummy head/tail) eliminate null checks for edge cases

---


## 14.7 Design a Pub-Sub Messaging System

> A Publish-Subscribe (Pub-Sub) messaging system decouples message producers (publishers) from consumers (subscribers) through topics. Publishers send messages to topics without knowing who will receive them, and subscribers receive messages from topics they're interested in without knowing who sent them. This problem tests your understanding of the Observer pattern, message ordering, acknowledgment mechanisms, and concurrent message delivery.

---

### Requirements

1. **Topics**: Named channels that messages are published to
2. **Publishers**: Can publish messages to any topic
3. **Subscribers**: Can subscribe to one or more topics and receive messages
4. **Message Ordering**: Messages within a topic are delivered in order (FIFO)
5. **Acknowledgment**: Subscribers acknowledge message receipt; unacknowledged messages can be redelivered
6. **Filtering**: Subscribers can filter messages within a topic
7. **Thread Safety**: Support concurrent publishers and subscribers

---

### Key Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Message delivery | Push model (topic pushes to subscribers) | Lower latency, simpler subscriber logic |
| Message storage | Per-topic queue with offset tracking | Enables replay and acknowledgment |
| Subscriber tracking | Per-subscriber offset per topic | Each subscriber progresses independently |
| Thread safety | Mutex per topic | Reduces contention vs single global lock |

---

### Implementation

**Message class:**

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <unordered_set>
#include <queue>
#include <mutex>
#include <functional>
#include <memory>
#include <chrono>
#include <thread>
#include <condition_variable>
#include <atomic>
using namespace std;

using MessageId = uint64_t;
using Timestamp = chrono::system_clock::time_point;

class Message {
    MessageId id;
    string topic;
    string key;       // optional partition key
    string payload;
    unordered_map<string, string> headers;
    Timestamp timestamp;

    static atomic<MessageId> next_id;

public:
    Message(const string& topic, const string& payload,
            const string& key = "")
        : id(next_id++), topic(topic), key(key), payload(payload),
          timestamp(chrono::system_clock::now()) {}

    MessageId getId() const { return id; }
    const string& getTopic() const { return topic; }
    const string& getKey() const { return key; }
    const string& getPayload() const { return payload; }
    Timestamp getTimestamp() const { return timestamp; }

    void setHeader(const string& k, const string& v) { headers[k] = v; }

    string getHeader(const string& k) const {
        auto it = headers.find(k);
        return it != headers.end() ? it->second : "";
    }

    string toString() const {
        return "[" + to_string(id) + "] " + topic + ": " + payload;
    }
};

atomic<MessageId> Message::next_id{1};
```

**Subscriber interface:**

```cpp
// Message handler callback type
using MessageHandler = function<void(const Message&)>;

// Message filter predicate
using MessageFilter = function<bool(const Message&)>;

class ISubscriber {
public:
    virtual ~ISubscriber() = default;
    virtual string getId() const = 0;
    virtual void onMessage(const Message& msg) = 0;
};

// Concrete subscriber with callback
class Subscriber : public ISubscriber {
    string id;
    MessageHandler handler;
    MessageFilter filter;

public:
    Subscriber(const string& id, MessageHandler handler,
               MessageFilter filter = nullptr)
        : id(id), handler(handler), filter(filter) {}

    string getId() const override { return id; }

    void onMessage(const Message& msg) override {
        if (!filter || filter(msg)) {
            handler(msg);
        }
    }
};
```

**Topic class:**

```cpp
class Topic {
    string name;
    vector<Message> messages;           // message log (append-only)
    vector<shared_ptr<ISubscriber>> subscribers;
    unordered_map<string, size_t> subscriber_offsets;  // subscriber_id → last acked offset
    mutable mutex mtx;

public:
    Topic(const string& name) : name(name) {}

    const string& getName() const { return name; }

    // Add a subscriber
    void subscribe(shared_ptr<ISubscriber> subscriber) {
        lock_guard<mutex> lock(mtx);
        // Check if already subscribed
        for (const auto& sub : subscribers) {
            if (sub->getId() == subscriber->getId()) return;
        }
        subscribers.push_back(subscriber);
        subscriber_offsets[subscriber->getId()] = messages.size();  // start from current position
        cout << subscriber->getId() << " subscribed to " << name << endl;
    }

    // Remove a subscriber
    void unsubscribe(const string& subscriber_id) {
        lock_guard<mutex> lock(mtx);
        subscribers.erase(
            remove_if(subscribers.begin(), subscribers.end(),
                [&](const shared_ptr<ISubscriber>& sub) {
                    return sub->getId() == subscriber_id;
                }),
            subscribers.end());
        subscriber_offsets.erase(subscriber_id);
        cout << subscriber_id << " unsubscribed from " << name << endl;
    }

    // Publish a message to this topic
    void publish(const Message& msg) {
        lock_guard<mutex> lock(mtx);
        messages.push_back(msg);

        // Deliver to all subscribers
        for (auto& sub : subscribers) {
            try {
                sub->onMessage(msg);
            } catch (const exception& e) {
                cerr << "Error delivering to " << sub->getId()
                     << ": " << e.what() << endl;
            }
        }
    }

    // Acknowledge a message (advance subscriber's offset)
    void acknowledge(const string& subscriber_id, MessageId msg_id) {
        lock_guard<mutex> lock(mtx);
        auto it = subscriber_offsets.find(subscriber_id);
        if (it == subscriber_offsets.end()) return;

        // Find the message index and advance offset
        for (size_t i = it->second; i < messages.size(); i++) {
            if (messages[i].getId() == msg_id) {
                it->second = i + 1;
                break;
            }
        }
    }

    // Get unacknowledged messages for a subscriber
    vector<Message> getUnacknowledged(const string& subscriber_id) const {
        lock_guard<mutex> lock(mtx);
        vector<Message> result;
        auto it = subscriber_offsets.find(subscriber_id);
        if (it == subscriber_offsets.end()) return result;

        for (size_t i = it->second; i < messages.size(); i++) {
            result.push_back(messages[i]);
        }
        return result;
    }

    // Replay messages from a specific offset
    vector<Message> replay(size_t from_offset, size_t count) const {
        lock_guard<mutex> lock(mtx);
        vector<Message> result;
        for (size_t i = from_offset; i < min(from_offset + count, messages.size()); i++) {
            result.push_back(messages[i]);
        }
        return result;
    }

    size_t messageCount() const {
        lock_guard<mutex> lock(mtx);
        return messages.size();
    }

    int subscriberCount() const {
        lock_guard<mutex> lock(mtx);
        return subscribers.size();
    }
};
```

**MessageBroker (facade):**

```cpp
class MessageBroker {
    unordered_map<string, unique_ptr<Topic>> topics;
    mutable mutex mtx;

public:
    // Create a topic
    Topic* createTopic(const string& name) {
        lock_guard<mutex> lock(mtx);
        if (topics.count(name)) {
            return topics[name].get();
        }
        topics[name] = make_unique<Topic>(name);
        cout << "Topic created: " << name << endl;
        return topics[name].get();
    }

    // Get or create a topic
    Topic* getTopic(const string& name) {
        lock_guard<mutex> lock(mtx);
        auto it = topics.find(name);
        if (it == topics.end()) return nullptr;
        return it->second.get();
    }

    // Publish a message to a topic
    void publish(const string& topic_name, const string& payload,
                 const string& key = "") {
        Topic* topic = nullptr;
        {
            lock_guard<mutex> lock(mtx);
            auto it = topics.find(topic_name);
            if (it == topics.end()) {
                // Auto-create topic
                topics[topic_name] = make_unique<Topic>(topic_name);
                topic = topics[topic_name].get();
            } else {
                topic = it->second.get();
            }
        }

        Message msg(topic_name, payload, key);
        topic->publish(msg);
    }

    // Subscribe to a topic
    void subscribe(const string& topic_name, shared_ptr<ISubscriber> subscriber) {
        Topic* topic = nullptr;
        {
            lock_guard<mutex> lock(mtx);
            auto it = topics.find(topic_name);
            if (it == topics.end()) {
                topics[topic_name] = make_unique<Topic>(topic_name);
            }
            topic = topics[topic_name].get();
        }
        topic->subscribe(subscriber);
    }

    // Unsubscribe from a topic
    void unsubscribe(const string& topic_name, const string& subscriber_id) {
        lock_guard<mutex> lock(mtx);
        auto it = topics.find(topic_name);
        if (it != topics.end()) {
            it->second->unsubscribe(subscriber_id);
        }
    }

    // List all topics
    vector<string> listTopics() const {
        lock_guard<mutex> lock(mtx);
        vector<string> names;
        for (const auto& [name, _] : topics) {
            names.push_back(name);
        }
        return names;
    }
};
```

**Usage example:**

```cpp
int main() {
    MessageBroker broker;

    // Create subscribers
    auto logger = make_shared<Subscriber>("logger",
        [](const Message& msg) {
            cout << "[LOG] " << msg.toString() << endl;
        });

    auto alert_handler = make_shared<Subscriber>("alert-handler",
        [](const Message& msg) {
            cout << "[ALERT] Processing: " << msg.getPayload() << endl;
        },
        // Filter: only messages containing "ERROR"
        [](const Message& msg) {
            return msg.getPayload().find("ERROR") != string::npos;
        });

    auto analytics = make_shared<Subscriber>("analytics",
        [](const Message& msg) {
            cout << "[ANALYTICS] Recorded: " << msg.getPayload() << endl;
        });

    // Subscribe to topics
    broker.subscribe("orders", logger);
    broker.subscribe("orders", analytics);
    broker.subscribe("errors", logger);
    broker.subscribe("errors", alert_handler);

    // Publish messages
    cout << "\n--- Publishing messages ---" << endl;
    broker.publish("orders", "New order #1001 placed");
    broker.publish("orders", "Order #1001 shipped");
    broker.publish("errors", "ERROR: Payment gateway timeout");
    broker.publish("errors", "WARN: High latency detected");

    // Output:
    // [LOG] [1] orders: New order #1001 placed
    // [ANALYTICS] Recorded: New order #1001 placed
    // [LOG] [2] orders: Order #1001 shipped
    // [ANALYTICS] Recorded: Order #1001 shipped
    // [LOG] [3] errors: ERROR: Payment gateway timeout
    // [ALERT] Processing: ERROR: Payment gateway timeout
    // [LOG] [4] errors: WARN: High latency detected
    // (alert_handler filters out the WARN message — no ERROR keyword)

    return 0;
}
```

**Design patterns used:**
- **Observer**: Topics notify subscribers when new messages arrive
- **Mediator**: `MessageBroker` mediates between publishers and subscribers
- **Strategy**: Message filters allow subscribers to customize which messages they receive
- **Facade**: `MessageBroker` provides a simplified interface to the topic/subscriber system

---


## 14.8 Design a Task Scheduler

> A task scheduler manages the execution of tasks based on priority, dependencies, and timing constraints. This problem combines graph algorithms (DAG for dependencies, topological sort for execution order), the Strategy pattern (different scheduling policies), concurrency (parallel task execution), and error handling (retry, timeout). It's a hard problem because it requires balancing multiple concerns: correctness (dependency order), efficiency (parallel execution), and reliability (retry on failure).

---

### Requirements

1. **Tasks**: Each task has a name, priority, and optional dependencies on other tasks
2. **Dependency Graph**: Tasks form a DAG (Directed Acyclic Graph); a task runs only after all its dependencies complete
3. **Priority**: Higher priority tasks execute first among tasks with satisfied dependencies
4. **Concurrent Execution**: Independent tasks can run in parallel
5. **Retry**: Failed tasks can be retried up to a configurable limit
6. **Timeout**: Tasks that exceed a time limit are cancelled
7. **Status Tracking**: Track task status (pending, running, completed, failed)

---

### Key Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Dependency resolution | Topological sort (Kahn's algorithm) | Handles DAG ordering, detects cycles |
| Task queue | Priority queue of ready tasks | Higher priority tasks execute first |
| Execution | Thread pool with configurable concurrency | Parallel execution of independent tasks |
| Retry policy | Configurable max retries with backoff | Handles transient failures |

---

### Implementation

**Task and TaskStatus:**

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <unordered_set>
#include <queue>
#include <functional>
#include <memory>
#include <mutex>
#include <thread>
#include <condition_variable>
#include <chrono>
#include <atomic>
#include <future>
#include <algorithm>
using namespace std;

enum class TaskStatus {
    PENDING,     // waiting for dependencies
    READY,       // dependencies satisfied, waiting to execute
    RUNNING,     // currently executing
    COMPLETED,   // finished successfully
    FAILED,      // failed after all retries
    CANCELLED    // cancelled (timeout or manual)
};

string status_to_string(TaskStatus s) {
    switch (s) {
        case TaskStatus::PENDING:   return "PENDING";
        case TaskStatus::READY:     return "READY";
        case TaskStatus::RUNNING:   return "RUNNING";
        case TaskStatus::COMPLETED: return "COMPLETED";
        case TaskStatus::FAILED:    return "FAILED";
        case TaskStatus::CANCELLED: return "CANCELLED";
        default: return "UNKNOWN";
    }
}

using TaskFunction = function<bool()>;  // returns true on success

class Task {
    string id;
    string name;
    int priority;                    // higher = more important
    TaskFunction work;               // the actual work to do
    vector<string> dependencies;     // IDs of tasks this depends on
    TaskStatus status = TaskStatus::PENDING;
    int max_retries;
    int retry_count = 0;
    chrono::milliseconds timeout;
    string error_message;

public:
    Task(const string& id, const string& name, TaskFunction work,
         int priority = 0, int max_retries = 0,
         chrono::milliseconds timeout = chrono::milliseconds(0))
        : id(id), name(name), priority(priority), work(work),
          max_retries(max_retries), timeout(timeout) {}

    const string& getId() const { return id; }
    const string& getName() const { return name; }
    int getPriority() const { return priority; }
    TaskStatus getStatus() const { return status; }
    void setStatus(TaskStatus s) { status = s; }
    const vector<string>& getDependencies() const { return dependencies; }
    const string& getError() const { return error_message; }
    chrono::milliseconds getTimeout() const { return timeout; }

    void addDependency(const string& dep_id) {
        dependencies.push_back(dep_id);
    }

    bool canRetry() const {
        return retry_count < max_retries;
    }

    // Execute the task
    bool execute() {
        status = TaskStatus::RUNNING;
        try {
            bool success = work();
            if (success) {
                status = TaskStatus::COMPLETED;
                return true;
            } else {
                retry_count++;
                if (canRetry()) {
                    status = TaskStatus::READY;  // will be retried
                } else {
                    status = TaskStatus::FAILED;
                    error_message = "Task returned false after " +
                                    to_string(retry_count) + " attempts";
                }
                return false;
            }
        } catch (const exception& e) {
            retry_count++;
            error_message = e.what();
            if (canRetry()) {
                status = TaskStatus::READY;
            } else {
                status = TaskStatus::FAILED;
            }
            return false;
        }
    }
};

// Comparator for priority queue (higher priority first)
struct TaskPriorityCompare {
    bool operator()(const Task* a, const Task* b) const {
        return a->getPriority() < b->getPriority();  // min-heap → flip for max
    }
};
```

**TaskScheduler:**

```cpp
class TaskScheduler {
    unordered_map<string, unique_ptr<Task>> tasks;
    unordered_map<string, unordered_set<string>> dependents;  // task → tasks that depend on it
    unordered_map<string, int> in_degree;  // task → number of unsatisfied dependencies
    priority_queue<Task*, vector<Task*>, TaskPriorityCompare> ready_queue;
    mutex mtx;
    int max_concurrency;

public:
    TaskScheduler(int concurrency = 4) : max_concurrency(concurrency) {}

    // Add a task to the scheduler
    void addTask(unique_ptr<Task> task) {
        string id = task->getId();

        // Calculate in-degree (number of dependencies)
        int deg = 0;
        for (const auto& dep : task->getDependencies()) {
            dependents[dep].insert(id);
            deg++;
        }
        in_degree[id] = deg;

        tasks[id] = std::move(task);
    }

    // Validate the task graph (check for cycles)
    bool validate() const {
        // Kahn's algorithm for cycle detection
        unordered_map<string, int> deg = in_degree;
        queue<string> q;

        for (const auto& [id, d] : deg) {
            if (d == 0) q.push(id);
        }

        int processed = 0;
        while (!q.empty()) {
            string current = q.front();
            q.pop();
            processed++;

            if (dependents.count(current)) {
                for (const auto& dep : dependents.at(current)) {
                    deg[dep]--;
                    if (deg[dep] == 0) q.push(dep);
                }
            }
        }

        if (processed != (int)tasks.size()) {
            cerr << "Circular dependency detected! Processed " << processed
                 << " of " << tasks.size() << " tasks." << endl;
            return false;
        }
        return true;
    }

    // Execute all tasks respecting dependencies and priority
    void execute() {
        if (!validate()) {
            throw runtime_error("Cannot execute: circular dependencies detected");
        }

        cout << "=== Starting Task Scheduler ===" << endl;
        cout << "Tasks: " << tasks.size()
             << ", Max concurrency: " << max_concurrency << endl;

        // Initialize ready queue with tasks that have no dependencies
        for (auto& [id, task] : tasks) {
            if (in_degree[id] == 0) {
                task->setStatus(TaskStatus::READY);
                ready_queue.push(task.get());
            }
        }

        // Track running tasks
        vector<future<void>> running;
        unordered_set<string> running_ids;
        mutex result_mtx;
        vector<string> completed_tasks;

        while (!ready_queue.empty() || !running_ids.empty()) {
            // Launch tasks up to concurrency limit
            while (!ready_queue.empty() &&
                   (int)running_ids.size() < max_concurrency) {
                Task* task = ready_queue.top();
                ready_queue.pop();

                if (task->getStatus() != TaskStatus::READY) continue;

                running_ids.insert(task->getId());
                cout << "[START] " << task->getName()
                     << " (priority: " << task->getPriority() << ")" << endl;

                // Launch task asynchronously
                running.push_back(async(launch::async, [&, task]() {
                    bool success = false;

                    if (task->getTimeout().count() > 0) {
                        // Execute with timeout
                        auto future = async(launch::async, [task]() {
                            return task->execute();
                        });

                        auto status = future.wait_for(task->getTimeout());
                        if (status == future_status::timeout) {
                            task->setStatus(TaskStatus::CANCELLED);
                            cout << "[TIMEOUT] " << task->getName() << endl;
                        } else {
                            success = future.get();
                        }
                    } else {
                        success = task->execute();
                    }

                    lock_guard<mutex> lock(result_mtx);
                    running_ids.erase(task->getId());

                    if (task->getStatus() == TaskStatus::COMPLETED) {
                        cout << "[DONE] " << task->getName() << endl;
                        completed_tasks.push_back(task->getId());
                    } else if (task->getStatus() == TaskStatus::READY) {
                        // Retry
                        cout << "[RETRY] " << task->getName()
                             << " - " << task->getError() << endl;
                        ready_queue.push(task);
                    } else {
                        cout << "[FAILED] " << task->getName()
                             << " - " << task->getError() << endl;
                    }
                }));
            }

            // Process completed tasks and unlock dependents
            {
                lock_guard<mutex> lock(result_mtx);
                for (const auto& id : completed_tasks) {
                    if (dependents.count(id)) {
                        for (const auto& dep_id : dependents[id]) {
                            in_degree[dep_id]--;
                            if (in_degree[dep_id] == 0) {
                                tasks[dep_id]->setStatus(TaskStatus::READY);
                                ready_queue.push(tasks[dep_id].get());
                            }
                        }
                    }
                }
                completed_tasks.clear();
            }

            // Brief sleep to avoid busy waiting
            this_thread::sleep_for(chrono::milliseconds(10));
        }

        // Wait for all futures to complete
        for (auto& f : running) {
            if (f.valid()) f.wait();
        }

        cout << "\n=== Execution Complete ===" << endl;
        printSummary();
    }

    void printSummary() const {
        cout << "\nTask Summary:" << endl;
        for (const auto& [id, task] : tasks) {
            cout << "  " << task->getName() << ": "
                 << status_to_string(task->getStatus());
            if (task->getStatus() == TaskStatus::FAILED) {
                cout << " (" << task->getError() << ")";
            }
            cout << endl;
        }
    }
};
```

**Usage example:**

```cpp
int main() {
    TaskScheduler scheduler(2);  // max 2 concurrent tasks

    // Create tasks with dependencies
    // Build pipeline: compile → link → test → deploy
    auto compile = make_unique<Task>("compile", "Compile Source", []() {
        cout << "  Compiling..." << endl;
        this_thread::sleep_for(chrono::milliseconds(100));
        return true;
    }, 10);  // priority 10

    auto lint = make_unique<Task>("lint", "Run Linter", []() {
        cout << "  Linting..." << endl;
        this_thread::sleep_for(chrono::milliseconds(50));
        return true;
    }, 8);  // priority 8

    auto link_task = make_unique<Task>("link", "Link Objects", []() {
        cout << "  Linking..." << endl;
        this_thread::sleep_for(chrono::milliseconds(80));
        return true;
    }, 7);
    link_task->addDependency("compile");

    auto test = make_unique<Task>("test", "Run Tests", []() {
        cout << "  Testing..." << endl;
        this_thread::sleep_for(chrono::milliseconds(120));
        return true;
    }, 5);
    test->addDependency("link");
    test->addDependency("lint");

    auto deploy = make_unique<Task>("deploy", "Deploy", []() {
        cout << "  Deploying..." << endl;
        this_thread::sleep_for(chrono::milliseconds(60));
        return true;
    }, 3, 2);  // max 2 retries
    deploy->addDependency("test");

    scheduler.addTask(std::move(compile));
    scheduler.addTask(std::move(lint));
    scheduler.addTask(std::move(link_task));
    scheduler.addTask(std::move(test));
    scheduler.addTask(std::move(deploy));

    scheduler.execute();

    // Execution order respects dependencies:
    // compile and lint run in parallel (no dependencies)
    // link runs after compile
    // test runs after link AND lint
    // deploy runs after test

    return 0;
}
```

**Design patterns used:**
- **Strategy**: Different scheduling policies (priority-based, FIFO, round-robin) can be swapped
- **Observer**: Task completion notifies the scheduler to unlock dependent tasks
- **Command**: Each `Task` encapsulates work to be executed, with retry capability
- **Template Method**: The execution pipeline (validate → schedule → execute → report) follows a fixed structure

---


## 14.9 Design a Rate Limiter

> A rate limiter controls the rate of requests a client can make to a system within a given time window. It's essential for protecting APIs from abuse, ensuring fair usage, and preventing system overload. This problem tests your understanding of different rate limiting algorithms (Token Bucket, Sliding Window, Fixed Window), time-based data structures, and thread-safe implementations. In interviews, you're expected to compare algorithms, discuss trade-offs, and implement at least one in detail.

---

### Requirements

1. **Per-Client Limiting**: Each client (identified by ID, IP, or API key) has its own rate limit
2. **Configurable Limits**: Different limits for different APIs or client tiers
3. **Multiple Algorithms**: Token Bucket, Sliding Window, Fixed Window Counter
4. **Thread Safety**: Support concurrent requests from multiple clients
5. **Accurate Timing**: Handle edge cases around window boundaries
6. **Informative Responses**: Tell clients how many requests remain and when to retry

---

### Key Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Algorithm interface | Strategy pattern — `RateLimiter` base class | Swap algorithms without changing client code |
| Per-client state | `unordered_map<string, State>` | Each client tracked independently |
| Thread safety | Mutex per client (or per limiter) | Reduces contention for concurrent requests |
| Time source | `chrono::steady_clock` | Monotonic, not affected by system clock changes |

---

### Algorithm Comparison

| Algorithm | Pros | Cons | Best For |
|-----------|------|------|----------|
| **Token Bucket** | Smooth rate, allows bursts | Slightly complex | API rate limiting |
| **Fixed Window** | Simple, low memory | Boundary spike (2x burst) | Simple counters |
| **Sliding Window Log** | Most accurate | High memory (stores timestamps) | Strict rate limiting |
| **Sliding Window Counter** | Good accuracy, low memory | Approximate | Production APIs |

---

### Implementation

**Rate Limiter interface and result:**

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <mutex>
#include <chrono>
#include <deque>
#include <algorithm>
#include <memory>
#include <thread>
using namespace std;

using Clock = chrono::steady_clock;
using TimePoint = Clock::time_point;
using Duration = chrono::milliseconds;

struct RateLimitResult {
    bool allowed;
    int remaining;          // requests remaining in current window
    Duration retry_after;   // how long to wait before retrying (if denied)

    static RateLimitResult allow(int remaining) {
        return {true, remaining, Duration(0)};
    }

    static RateLimitResult deny(Duration retry_after) {
        return {false, 0, retry_after};
    }
};

// Abstract rate limiter interface (Strategy pattern)
class RateLimiter {
public:
    virtual ~RateLimiter() = default;
    virtual RateLimitResult tryAcquire(const string& client_id) = 0;
    virtual string getAlgorithmName() const = 0;
};
```

**Token Bucket Algorithm:**

```cpp
// Token Bucket: tokens are added at a fixed rate; each request consumes one token.
// Allows bursts up to bucket capacity, then limits to the refill rate.
class TokenBucketLimiter : public RateLimiter {
    struct Bucket {
        double tokens;
        TimePoint last_refill;
    };

    int max_tokens;           // bucket capacity (max burst size)
    double refill_rate;       // tokens per second
    unordered_map<string, Bucket> buckets;
    mutex mtx;

    void refill(Bucket& bucket) {
        auto now = Clock::now();
        double elapsed = chrono::duration<double>(now - bucket.last_refill).count();
        bucket.tokens = min((double)max_tokens, bucket.tokens + elapsed * refill_rate);
        bucket.last_refill = now;
    }

public:
    // max_tokens: burst capacity, refill_rate: sustained rate (tokens/sec)
    TokenBucketLimiter(int max_tokens, double refill_rate)
        : max_tokens(max_tokens), refill_rate(refill_rate) {}

    string getAlgorithmName() const override { return "Token Bucket"; }

    RateLimitResult tryAcquire(const string& client_id) override {
        lock_guard<mutex> lock(mtx);

        auto it = buckets.find(client_id);
        if (it == buckets.end()) {
            // New client — start with full bucket
            buckets[client_id] = {(double)max_tokens, Clock::now()};
            it = buckets.find(client_id);
        }

        Bucket& bucket = it->second;
        refill(bucket);

        if (bucket.tokens >= 1.0) {
            bucket.tokens -= 1.0;
            return RateLimitResult::allow((int)bucket.tokens);
        }

        // Calculate wait time for next token
        double deficit = 1.0 - bucket.tokens;
        auto wait_ms = (int)(deficit / refill_rate * 1000);
        return RateLimitResult::deny(Duration(wait_ms));
    }
};
```

**Fixed Window Counter:**

```cpp
// Fixed Window: divide time into fixed windows (e.g., 1 minute).
// Count requests per window. Reset counter at window boundary.
class FixedWindowLimiter : public RateLimiter {
    struct Window {
        int count;
        TimePoint window_start;
    };

    int max_requests;
    Duration window_size;
    unordered_map<string, Window> windows;
    mutex mtx;

public:
    FixedWindowLimiter(int max_requests, Duration window_size)
        : max_requests(max_requests), window_size(window_size) {}

    string getAlgorithmName() const override { return "Fixed Window"; }

    RateLimitResult tryAcquire(const string& client_id) override {
        lock_guard<mutex> lock(mtx);
        auto now = Clock::now();

        auto it = windows.find(client_id);
        if (it == windows.end()) {
            windows[client_id] = {0, now};
            it = windows.find(client_id);
        }

        Window& window = it->second;

        // Check if we've moved to a new window
        if (now - window.window_start >= window_size) {
            window.count = 0;
            window.window_start = now;
        }

        if (window.count < max_requests) {
            window.count++;
            return RateLimitResult::allow(max_requests - window.count);
        }

        // Calculate time until window resets
        auto elapsed = chrono::duration_cast<Duration>(now - window.window_start);
        auto retry = window_size - elapsed;
        return RateLimitResult::deny(retry);
    }
};
```

**Sliding Window Log:**

```cpp
// Sliding Window Log: store timestamp of each request.
// Count requests in the last N seconds. Most accurate but uses more memory.
class SlidingWindowLogLimiter : public RateLimiter {
    int max_requests;
    Duration window_size;
    unordered_map<string, deque<TimePoint>> request_logs;
    mutex mtx;

public:
    SlidingWindowLogLimiter(int max_requests, Duration window_size)
        : max_requests(max_requests), window_size(window_size) {}

    string getAlgorithmName() const override { return "Sliding Window Log"; }

    RateLimitResult tryAcquire(const string& client_id) override {
        lock_guard<mutex> lock(mtx);
        auto now = Clock::now();

        auto& log = request_logs[client_id];

        // Remove expired entries
        auto cutoff = now - window_size;
        while (!log.empty() && log.front() < cutoff) {
            log.pop_front();
        }

        if ((int)log.size() < max_requests) {
            log.push_back(now);
            return RateLimitResult::allow(max_requests - (int)log.size());
        }

        // Calculate when the oldest request will expire
        auto oldest = log.front();
        auto retry = chrono::duration_cast<Duration>(oldest + window_size - now);
        return RateLimitResult::deny(retry);
    }
};
```

**Sliding Window Counter (approximate but efficient):**

```cpp
// Sliding Window Counter: combines fixed window simplicity with sliding window accuracy.
// Uses weighted average of current and previous window counts.
class SlidingWindowCounterLimiter : public RateLimiter {
    struct WindowState {
        int current_count = 0;
        int previous_count = 0;
        TimePoint current_window_start;
    };

    int max_requests;
    Duration window_size;
    unordered_map<string, WindowState> states;
    mutex mtx;

public:
    SlidingWindowCounterLimiter(int max_requests, Duration window_size)
        : max_requests(max_requests), window_size(window_size) {}

    string getAlgorithmName() const override { return "Sliding Window Counter"; }

    RateLimitResult tryAcquire(const string& client_id) override {
        lock_guard<mutex> lock(mtx);
        auto now = Clock::now();

        auto it = states.find(client_id);
        if (it == states.end()) {
            states[client_id] = {0, 0, now};
            it = states.find(client_id);
        }

        WindowState& state = it->second;

        // Check if we need to advance windows
        auto elapsed = chrono::duration_cast<Duration>(now - state.current_window_start);
        if (elapsed >= window_size * 2) {
            // Skipped a full window
            state.previous_count = 0;
            state.current_count = 0;
            state.current_window_start = now;
        } else if (elapsed >= window_size) {
            // Moved to next window
            state.previous_count = state.current_count;
            state.current_count = 0;
            state.current_window_start = state.current_window_start + window_size;
        }

        // Calculate weighted count
        auto current_elapsed = chrono::duration_cast<Duration>(
            now - state.current_window_start);
        double weight = 1.0 - (double)current_elapsed.count() / window_size.count();
        double estimated = state.previous_count * weight + state.current_count;

        if (estimated < max_requests) {
            state.current_count++;
            return RateLimitResult::allow(max_requests - (int)estimated - 1);
        }

        // Estimate retry time
        auto retry = Duration((int)(window_size.count() * weight));
        return RateLimitResult::deny(retry);
    }
};
```

**Usage example:**

```cpp
void test_limiter(RateLimiter& limiter, const string& client_id, int requests) {
    cout << "\n--- Testing " << limiter.getAlgorithmName()
         << " (" << requests << " requests) ---" << endl;

    int allowed = 0, denied = 0;
    for (int i = 0; i < requests; i++) {
        auto result = limiter.tryAcquire(client_id);
        if (result.allowed) {
            allowed++;
        } else {
            denied++;
            if (denied == 1) {
                cout << "First denial at request " << (i + 1)
                     << ", retry after " << result.retry_after.count() << "ms" << endl;
            }
        }
    }
    cout << "Allowed: " << allowed << ", Denied: " << denied << endl;
}

int main() {
    // Token Bucket: 5 tokens max, refill 2 per second
    TokenBucketLimiter token_bucket(5, 2.0);
    test_limiter(token_bucket, "user1", 10);

    // Fixed Window: 5 requests per 1 second
    FixedWindowLimiter fixed_window(5, Duration(1000));
    test_limiter(fixed_window, "user1", 10);

    // Sliding Window Log: 5 requests per 1 second
    SlidingWindowLogLimiter sliding_log(5, Duration(1000));
    test_limiter(sliding_log, "user1", 10);

    // Sliding Window Counter: 5 requests per 1 second
    SlidingWindowCounterLimiter sliding_counter(5, Duration(1000));
    test_limiter(sliding_counter, "user1", 10);

    // Wait and try again (tokens should refill)
    cout << "\n--- After 1 second wait ---" << endl;
    this_thread::sleep_for(chrono::seconds(1));
    auto result = token_bucket.tryAcquire("user1");
    cout << "Token Bucket: " << (result.allowed ? "ALLOWED" : "DENIED")
         << ", remaining: " << result.remaining << endl;

    return 0;
}
```

**Design patterns used:**
- **Strategy**: `RateLimiter` interface with interchangeable algorithm implementations
- **Factory**: Could add a factory to create limiters based on configuration
- **Singleton**: Rate limiter is typically a singleton per service
- **Proxy**: Rate limiter acts as a proxy — checks rate before forwarding the request

---


## 14.10 Design a Multi-Level Cache

> A multi-level cache system uses multiple cache layers (L1 in-memory, L2 distributed) to optimize for both speed and capacity. Requests check L1 first (fastest, smallest), then L2 (slower, larger), and finally the data source. This problem tests your understanding of the Chain of Responsibility pattern (request flows through cache layers), caching strategies (cache-aside, write-through, write-back), eviction policies (LRU, LFU, FIFO), and the Proxy pattern (cache sits between client and data source).

---

### Requirements

1. **Multiple Levels**: L1 (in-process, fast, small) and L2 (distributed, slower, larger)
2. **Cache Policies**: Cache-aside, write-through, write-back
3. **Eviction Strategies**: LRU, LFU, FIFO — configurable per level
4. **TTL**: Time-to-live for cache entries
5. **Cache Coherence**: When L1 is updated, L2 should be consistent
6. **Metrics**: Track hit/miss rates per level
7. **Extensibility**: Easy to add more levels (L3, etc.)

---

### Key Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Cache layer interface | Chain of Responsibility | Each layer tries to serve, then delegates to next |
| Eviction strategy | Strategy pattern | Different levels can use different eviction policies |
| Write policy | Strategy pattern | Configurable per deployment |
| TTL | Per-entry expiration timestamp | Flexible, entries can have different TTLs |

---

### Implementation

**Cache entry with TTL:**

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <list>
#include <mutex>
#include <chrono>
#include <optional>
#include <memory>
#include <functional>
#include <atomic>
using namespace std;

using Clock = chrono::steady_clock;
using TimePoint = Clock::time_point;
using Duration = chrono::milliseconds;

template <typename V>
struct CacheEntry {
    V value;
    TimePoint created_at;
    TimePoint expires_at;
    int access_count = 0;

    CacheEntry(const V& val, Duration ttl)
        : value(val), created_at(Clock::now()),
          expires_at(Clock::now() + ttl) {}

    bool isExpired() const {
        return Clock::now() >= expires_at;
    }

    void touch() { access_count++; }
};
```

**Eviction Strategy interface:**

```cpp
// Eviction strategy interface
template <typename K>
class EvictionStrategy {
public:
    virtual ~EvictionStrategy() = default;
    virtual void onAccess(const K& key) = 0;
    virtual void onInsert(const K& key) = 0;
    virtual void onRemove(const K& key) = 0;
    virtual K evict() = 0;  // returns key to evict
    virtual string getName() const = 0;
};

// LRU Eviction — evict least recently used
template <typename K>
class LRUEviction : public EvictionStrategy<K> {
    list<K> order;  // front = most recent, back = least recent
    unordered_map<K, typename list<K>::iterator> positions;

public:
    string getName() const override { return "LRU"; }

    void onAccess(const K& key) override {
        auto it = positions.find(key);
        if (it != positions.end()) {
            order.erase(it->second);
            order.push_front(key);
            it->second = order.begin();
        }
    }

    void onInsert(const K& key) override {
        order.push_front(key);
        positions[key] = order.begin();
    }

    void onRemove(const K& key) override {
        auto it = positions.find(key);
        if (it != positions.end()) {
            order.erase(it->second);
            positions.erase(it);
        }
    }

    K evict() override {
        if (order.empty()) throw runtime_error("Nothing to evict");
        K key = order.back();
        order.pop_back();
        positions.erase(key);
        return key;
    }
};

// LFU Eviction — evict least frequently used
template <typename K>
class LFUEviction : public EvictionStrategy<K> {
    unordered_map<K, int> frequency;
    // frequency → list of keys with that frequency (for tie-breaking by insertion order)
    map<int, list<K>> freq_lists;
    unordered_map<K, typename list<K>::iterator> positions;
    int min_freq = 0;

public:
    string getName() const override { return "LFU"; }

    void onAccess(const K& key) override {
        auto it = frequency.find(key);
        if (it == frequency.end()) return;

        int old_freq = it->second;
        int new_freq = old_freq + 1;
        it->second = new_freq;

        // Remove from old frequency list
        freq_lists[old_freq].erase(positions[key]);
        if (freq_lists[old_freq].empty()) {
            freq_lists.erase(old_freq);
            if (min_freq == old_freq) min_freq = new_freq;
        }

        // Add to new frequency list
        freq_lists[new_freq].push_front(key);
        positions[key] = freq_lists[new_freq].begin();
    }

    void onInsert(const K& key) override {
        frequency[key] = 1;
        freq_lists[1].push_front(key);
        positions[key] = freq_lists[1].begin();
        min_freq = 1;
    }

    void onRemove(const K& key) override {
        int freq = frequency[key];
        freq_lists[freq].erase(positions[key]);
        if (freq_lists[freq].empty()) {
            freq_lists.erase(freq);
        }
        frequency.erase(key);
        positions.erase(key);
    }

    K evict() override {
        if (freq_lists.empty()) throw runtime_error("Nothing to evict");
        auto& lfu_list = freq_lists[min_freq];
        K key = lfu_list.back();
        lfu_list.pop_back();
        if (lfu_list.empty()) freq_lists.erase(min_freq);
        frequency.erase(key);
        positions.erase(key);
        return key;
    }
};

// FIFO Eviction — evict oldest inserted
template <typename K>
class FIFOEviction : public EvictionStrategy<K> {
    list<K> order;
    unordered_map<K, typename list<K>::iterator> positions;

public:
    string getName() const override { return "FIFO"; }

    void onAccess(const K& key) override {
        // FIFO doesn't change order on access
    }

    void onInsert(const K& key) override {
        order.push_back(key);
        positions[key] = prev(order.end());
    }

    void onRemove(const K& key) override {
        auto it = positions.find(key);
        if (it != positions.end()) {
            order.erase(it->second);
            positions.erase(it);
        }
    }

    K evict() override {
        if (order.empty()) throw runtime_error("Nothing to evict");
        K key = order.front();
        order.pop_front();
        positions.erase(key);
        return key;
    }
};
```

**Cache Layer (single level):**

```cpp
// Cache metrics
struct CacheMetrics {
    atomic<uint64_t> hits{0};
    atomic<uint64_t> misses{0};
    atomic<uint64_t> evictions{0};

    double hitRate() const {
        uint64_t total = hits + misses;
        return total > 0 ? (double)hits / total * 100.0 : 0.0;
    }

    void reset() { hits = 0; misses = 0; evictions = 0; }
};

// Abstract cache layer interface (Chain of Responsibility)
template <typename K, typename V>
class CacheLayer {
public:
    virtual ~CacheLayer() = default;
    virtual optional<V> get(const K& key) = 0;
    virtual void put(const K& key, const V& value, Duration ttl) = 0;
    virtual bool remove(const K& key) = 0;
    virtual void clear() = 0;
    virtual string getName() const = 0;
    virtual CacheMetrics& getMetrics() = 0;
};

// Concrete in-memory cache layer
template <typename K, typename V>
class InMemoryCache : public CacheLayer<K, V> {
    string name;
    int capacity;
    Duration default_ttl;
    unordered_map<K, CacheEntry<V>> store;
    unique_ptr<EvictionStrategy<K>> eviction;
    CacheMetrics metrics;
    mutable mutex mtx;

public:
    InMemoryCache(const string& name, int capacity, Duration default_ttl,
                  unique_ptr<EvictionStrategy<K>> eviction)
        : name(name), capacity(capacity), default_ttl(default_ttl),
          eviction(std::move(eviction)) {}

    string getName() const override { return name; }
    CacheMetrics& getMetrics() override { return metrics; }

    optional<V> get(const K& key) override {
        lock_guard<mutex> lock(mtx);

        auto it = store.find(key);
        if (it == store.end()) {
            metrics.misses++;
            return nullopt;
        }

        // Check TTL
        if (it->second.isExpired()) {
            eviction->onRemove(key);
            store.erase(it);
            metrics.misses++;
            return nullopt;
        }

        it->second.touch();
        eviction->onAccess(key);
        metrics.hits++;
        return it->second.value;
    }

    void put(const K& key, const V& value, Duration ttl = Duration(0)) override {
        lock_guard<mutex> lock(mtx);

        Duration actual_ttl = ttl.count() > 0 ? ttl : default_ttl;

        auto it = store.find(key);
        if (it != store.end()) {
            // Update existing
            it->second = CacheEntry<V>(value, actual_ttl);
            eviction->onAccess(key);
        } else {
            // Evict if at capacity
            while ((int)store.size() >= capacity) {
                K evicted_key = eviction->evict();
                store.erase(evicted_key);
                metrics.evictions++;
            }

            store.emplace(key, CacheEntry<V>(value, actual_ttl));
            eviction->onInsert(key);
        }
    }

    bool remove(const K& key) override {
        lock_guard<mutex> lock(mtx);
        auto it = store.find(key);
        if (it == store.end()) return false;
        eviction->onRemove(key);
        store.erase(it);
        return true;
    }

    void clear() override {
        lock_guard<mutex> lock(mtx);
        store.clear();
    }
};
```

**Multi-Level Cache (Chain of Responsibility):**

```cpp
// Write policy
enum class WritePolicy {
    WRITE_THROUGH,  // write to all levels immediately
    WRITE_BACK,     // write to L1 only, sync to L2 later
    CACHE_ASIDE     // caller manages cache and data source separately
};

template <typename K, typename V>
class MultiLevelCache {
    vector<unique_ptr<CacheLayer<K, V>>> levels;
    WritePolicy write_policy;
    // Data source (e.g., database) — called on cache miss at all levels
    function<optional<V>(const K&)> data_source;

public:
    MultiLevelCache(WritePolicy policy,
                    function<optional<V>(const K&)> source = nullptr)
        : write_policy(policy), data_source(source) {}

    void addLevel(unique_ptr<CacheLayer<K, V>> level) {
        levels.push_back(std::move(level));
    }

    // Get: check each level in order; on hit, populate higher levels
    optional<V> get(const K& key) {
        // Check each cache level
        for (int i = 0; i < (int)levels.size(); i++) {
            auto value = levels[i]->get(key);
            if (value) {
                // Cache hit at level i — populate all higher levels (L1, L2, ...)
                for (int j = 0; j < i; j++) {
                    levels[j]->put(key, *value, Duration(0));
                }
                return value;
            }
        }

        // Cache miss at all levels — fetch from data source
        if (data_source) {
            auto value = data_source(key);
            if (value) {
                // Populate all cache levels
                for (auto& level : levels) {
                    level->put(key, *value, Duration(0));
                }
                return value;
            }
        }

        return nullopt;
    }

    // Put: write according to policy
    void put(const K& key, const V& value, Duration ttl = Duration(0)) {
        switch (write_policy) {
            case WritePolicy::WRITE_THROUGH:
                // Write to all levels
                for (auto& level : levels) {
                    level->put(key, value, ttl);
                }
                break;

            case WritePolicy::WRITE_BACK:
                // Write to L1 only (sync to L2 happens asynchronously)
                if (!levels.empty()) {
                    levels[0]->put(key, value, ttl);
                }
                break;

            case WritePolicy::CACHE_ASIDE:
                // Write to all levels (caller also writes to data source)
                for (auto& level : levels) {
                    level->put(key, value, ttl);
                }
                break;
        }
    }

    // Remove from all levels (invalidation)
    void invalidate(const K& key) {
        for (auto& level : levels) {
            level->remove(key);
        }
    }

    // Clear all levels
    void clear() {
        for (auto& level : levels) {
            level->clear();
        }
    }

    // Print metrics for all levels
    void printMetrics() const {
        cout << "\n=== Cache Metrics ===" << endl;
        for (const auto& level : levels) {
            auto& m = level->getMetrics();
            cout << level->getName() << ":"
                 << " hits=" << m.hits
                 << " misses=" << m.misses
                 << " evictions=" << m.evictions
                 << " hit_rate=" << m.hitRate() << "%"
                 << endl;
        }
    }
};
```

**Usage example:**

```cpp
int main() {
    // Simulated database
    unordered_map<string, string> database = {
        {"user:1", "Alice"},
        {"user:2", "Bob"},
        {"user:3", "Charlie"},
        {"user:4", "Diana"},
        {"user:5", "Eve"}
    };

    auto db_lookup = [&](const string& key) -> optional<string> {
        cout << "  [DB] Fetching " << key << endl;
        auto it = database.find(key);
        if (it != database.end()) return it->second;
        return nullopt;
    };

    // Create multi-level cache
    MultiLevelCache<string, string> cache(WritePolicy::WRITE_THROUGH, db_lookup);

    // L1: Small, fast, LRU, 5-second TTL
    cache.addLevel(make_unique<InMemoryCache<string, string>>(
        "L1 (In-Memory)", 3, Duration(5000),
        make_unique<LRUEviction<string>>()));

    // L2: Larger, LFU, 30-second TTL
    cache.addLevel(make_unique<InMemoryCache<string, string>>(
        "L2 (Distributed)", 10, Duration(30000),
        make_unique<LFUEviction<string>>()));

    // First access — cache miss, fetches from DB
    cout << "--- First access ---" << endl;
    auto val = cache.get("user:1");
    cout << "user:1 = " << (val ? *val : "NOT FOUND") << endl;
    // Output: [DB] Fetching user:1
    //         user:1 = Alice

    // Second access — cache hit at L1
    cout << "\n--- Second access (cached) ---" << endl;
    val = cache.get("user:1");
    cout << "user:1 = " << (val ? *val : "NOT FOUND") << endl;
    // No DB fetch — served from L1

    // Access more keys to fill L1 (capacity 3)
    cout << "\n--- Filling L1 cache ---" << endl;
    cache.get("user:2");
    cache.get("user:3");
    cache.get("user:4");  // L1 is full — evicts LRU (user:1)

    // user:1 evicted from L1, but still in L2
    cout << "\n--- After L1 eviction ---" << endl;
    val = cache.get("user:1");
    cout << "user:1 = " << (val ? *val : "NOT FOUND") << endl;
    // L1 miss, L2 hit — no DB fetch, repopulates L1

    // Write-through: update propagates to all levels
    cout << "\n--- Write-through update ---" << endl;
    cache.put("user:1", "Alice (Updated)");
    val = cache.get("user:1");
    cout << "user:1 = " << (val ? *val : "NOT FOUND") << endl;
    // Output: user:1 = Alice (Updated)

    // Invalidation
    cout << "\n--- Invalidation ---" << endl;
    cache.invalidate("user:1");
    val = cache.get("user:1");
    cout << "user:1 = " << (val ? *val : "NOT FOUND") << endl;
    // Fetches from DB again (original value, not updated)

    cache.printMetrics();

    return 0;
}
```

**Design patterns used:**
- **Chain of Responsibility**: Request flows through L1 → L2 → data source until served
- **Strategy**: Eviction policies (LRU, LFU, FIFO) are interchangeable per cache level
- **Proxy**: Cache acts as a proxy between the client and the data source
- **Template Method**: The get flow (check level → populate higher levels → fetch from source) follows a fixed structure

---


## Summary & Key Takeaways

| Problem | Core Challenge | Key Data Structures | Key Design Patterns |
|---------|---------------|--------------------|--------------------|
| Chess Game | Complex rules, move validation, game state detection | 2D array, piece hierarchy | Strategy, Command, Observer |
| In-Memory File System | Tree structure, path resolution, permissions | Tree (Composite), HashMap | Composite, Iterator, Visitor, Facade |
| Spreadsheet | Dependency graph, circular detection, recalculation | DAG, AST, HashMap | Observer, Composite, Memento |
| Text Editor | Undo/redo, cursor management, clipboard | String buffer, command stack | Command, Memento, Singleton |
| Card Game Framework | Extensible game loop, multiple game types | Deck, Hand, Player | Template Method, Strategy, Factory |
| LRU Cache | O(1) get/put with eviction | HashMap + Doubly Linked List | — (pure data structure) |
| Pub-Sub Messaging | Decoupled communication, message ordering | Topic queues, subscriber map | Observer, Mediator, Strategy |
| Task Scheduler | DAG dependencies, priority, concurrency | DAG, Priority Queue, Thread Pool | Strategy, Observer, Command |
| Rate Limiter | Algorithm comparison, time-based tracking | Token bucket, sliding window | Strategy, Proxy, Singleton |
| Multi-Level Cache | Layered caching, eviction, write policies | Multiple cache layers, eviction structures | Chain of Responsibility, Strategy, Proxy |

---

## Interview Tips for Module 14

1. **"Design a Chess Game."** Start with the class hierarchy: abstract `Piece` base with `canMove()`, concrete pieces (King, Queen, Rook, Bishop, Knight, Pawn). Board is an 8×8 grid. Move validation is two-phase: piece-specific rules + global check (does the move leave own king in check?). Handle special moves (castling, en passant, pawn promotion) in the `Game` class, not in individual pieces. Use the Command pattern for move history. Checkmate detection: king is in check AND no legal move can escape it.

2. **"Design an In-Memory File System."** Use the Composite pattern: `FSEntry` base class, `File` (leaf) and `Directory` (composite). Directories store children in a `HashMap<string, FSEntry>` for O(1) lookup. Path resolution: split by `/`, traverse the tree. Support `mkdir -p` (recursive directory creation). Add permissions as a bitmask (rwx). The `FileSystem` class is a Facade that provides path-based operations.

3. **"Design a Spreadsheet."** The hard part is the dependency graph. Each cell with a formula depends on other cells. Use a DAG to track dependencies. When a cell changes, recalculate all dependents in topological order (BFS from the changed cell). Detect circular dependencies using DFS with coloring (white/gray/black — a gray node visited again means a cycle). Parse formulas with a recursive descent parser into an AST.

4. **"Design a Text Editor with Undo/Redo."** Use the Command pattern: each operation (insert, delete, replace) is a command object with `execute()` and `undo()`. Maintain two stacks: undo stack and redo stack. On new action: push to undo stack, clear redo stack. On undo: pop from undo stack, call `undo()`, push to redo stack. On redo: pop from redo stack, call `execute()`, push to undo stack. Each command stores the data needed to reverse itself (deleted text, old text for replace).

5. **"Design a Card Game Framework."** Use the Template Method pattern: `CardGame::play()` defines the game loop skeleton (initialize → shuffle → deal → play rounds → announce winner). Subclasses override hook methods (`dealCards()`, `playTurn()`, `checkWinCondition()`, `announceWinner()`). To add a new game, create a new subclass — the framework code doesn't change. Use Factory for deck creation (standard 52-card vs UNO deck).

6. **"Design an LRU Cache."** HashMap + Doubly Linked List. HashMap maps key → list node pointer (O(1) lookup). Doubly linked list maintains access order (most recent at head, least recent at tail). On `get()`: move node to head. On `put()`: if key exists, update and move to head; if new, add to head and evict from tail if at capacity. Use sentinel nodes (dummy head/tail) to eliminate null checks. For thread safety, wrap with `mutex` and `lock_guard`.

7. **"Design a Pub-Sub System."** Three main components: `MessageBroker` (facade), `Topic` (message channel), `Subscriber` (message consumer). Publishers send messages to topics via the broker. Topics maintain a message log and a list of subscribers. Each subscriber has an independent offset for acknowledgment. Use the Observer pattern for message delivery. Support message filtering with predicates. Thread safety: mutex per topic to reduce contention.

8. **"Design a Task Scheduler."** Model tasks as nodes in a DAG where edges represent dependencies. Use Kahn's algorithm (BFS-based topological sort) to determine execution order and detect cycles. Maintain an in-degree count per task; when a task completes, decrement in-degree of its dependents. Tasks with in-degree 0 are ready to execute. Use a priority queue for ready tasks. Execute independent tasks in parallel using a thread pool. Support retry with configurable max attempts.

9. **"Design a Rate Limiter."** Know the four main algorithms: (a) **Token Bucket** — tokens refill at a fixed rate, each request consumes one token; allows bursts up to bucket capacity. (b) **Fixed Window** — count requests per time window; simple but allows 2x burst at window boundaries. (c) **Sliding Window Log** — store timestamps of all requests; most accurate but high memory. (d) **Sliding Window Counter** — weighted average of current and previous window; good balance of accuracy and efficiency. Use the Strategy pattern to make algorithms interchangeable.

10. **"Design a Multi-Level Cache."** Use Chain of Responsibility: request flows through L1 → L2 → data source. On cache hit at level N, populate all higher levels (L1 through L(N-1)). Use the Strategy pattern for eviction policies (LRU, LFU, FIFO) — different levels can use different strategies. Support write policies: write-through (write to all levels), write-back (write to L1, sync later), cache-aside (caller manages). Track metrics (hit rate, miss rate, evictions) per level for monitoring.

11. **"How do you handle concurrency in these systems?"** Use `mutex` with `lock_guard` or `unique_lock` for thread safety. Prefer fine-grained locking (per-object or per-topic) over a single global lock to reduce contention. For read-heavy workloads, use `shared_mutex` (read-write lock). For lock-free structures, use `atomic` operations. Always consider deadlock prevention: consistent lock ordering, avoid holding multiple locks, use `std::lock` for multiple mutexes.

12. **"What design patterns appear most frequently in hard LLD problems?"** (a) **Strategy** — appears in almost every problem (scoring, eviction, rate limiting, scheduling). (b) **Observer** — for reactive updates (spreadsheet recalculation, pub-sub, game state). (c) **Command** — for undo/redo and task encapsulation. (d) **Composite** — for tree structures (file system, expression AST). (e) **Chain of Responsibility** — for layered processing (multi-level cache, middleware). (f) **Template Method** — for extensible frameworks (card game, game loop). (g) **Facade** — for simplifying complex subsystems.
