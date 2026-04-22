# Module 14: LLD Practice Problems (Hard) — Ruby

> Hard LLD problems test your ability to design complex, real-world systems with intricate rules, multiple interacting components, and non-trivial algorithms. These problems require combining multiple design patterns, handling concurrency, managing complex state transitions, and building extensible architectures. In interviews, you're expected to identify the right abstractions, define clean interfaces, handle edge cases, and justify your design decisions. This module covers 10 hard problems in depth with complete Ruby implementations.

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
| Piece hierarchy | Abstract `Piece` base class with `can_move?` | Each piece has unique movement — polymorphism is natural |
| Board representation | 8×8 2D array of `Piece` references | Direct access by position, simple and efficient |
| Move validation | Two-phase: piece-specific + global (king safety) | Separates concerns — piece logic vs game rules |
| Game state | Enum-style module with transition logic | Game has clear states with defined transitions |
| Special moves | Handled in `Game` class, not in individual pieces | Castling/en passant involve multiple pieces and game state |

---

### Class Design

```
+------------------+       +------------------+       +------------------+
|      Game        |       |      Board       |       |      Player      |
+------------------+       +------------------+       +------------------+
| - board          |<>---->| - grid[8][8]     |       | - name           |
| - players[2]     |<>---->| + get_piece()    |       | - color          |
| - current_turn   |       | + set_piece()    |       | - captured_pieces|
| - status         |       | + move_piece()   |       +------------------+
| - move_history   |       | + inside?()      |
| + make_move()    |       +------------------+
| + in_check?()    |
| + checkmate?()   |              ^
| + stalemate?()   |              |
+------------------+       +------+------+
                           |    Piece    |
                           +-------------+
                           | - color     |
                           | - position  |
                           | - has_moved |
                           | + can_move? | (abstract)
                           | + piece_type| (abstract)
                           +-------------+
                                 ^
          +----------+-----------+----------+-----------+----------+
          |          |           |          |           |          |
        King      Queen       Rook      Bishop      Knight      Pawn
```

---

### Implementation

```ruby
# ─── Enums and Position ──────────────────────────────────

module Color
  WHITE = :white
  BLACK = :black

  def self.opposite(color)
    color == WHITE ? BLACK : WHITE
  end

  def self.to_s(color)
    color == WHITE ? "White" : "Black"
  end
end

module PieceType
  KING   = :king
  QUEEN  = :queen
  ROOK   = :rook
  BISHOP = :bishop
  KNIGHT = :knight
  PAWN   = :pawn
end

module GameStatus
  ACTIVE    = :active
  CHECK     = :check
  CHECKMATE = :checkmate
  STALEMATE = :stalemate
  RESIGNED  = :resigned
  DRAW      = :draw
end

class Position
  attr_reader :row, :col

  def initialize(row, col)
    @row = row
    @col = col
  end

  def valid?
    @row.between?(0, 7) && @col.between?(0, 7)
  end

  def ==(other)
    other.is_a?(Position) && @row == other.row && @col == other.col
  end

  def !=(other)
    !(self == other)
  end

  def to_s
    "#{('a'.ord + @col).chr}#{@row + 1}"
  end

  # For use as hash key
  def eql?(other)
    self == other
  end

  def hash
    [@row, @col].hash
  end
end
```

**Abstract Piece class:**

```ruby
class Piece
  attr_reader :color, :position
  attr_accessor :has_moved

  def initialize(color, position)
    @color = color
    @position = position
    @has_moved = false
  end

  def set_position(pos)
    @position = pos
    @has_moved = true
  end

  # Returns true if the piece can move from its current position to 'target'
  # Does NOT check if the move leaves own king in check (that's the Game's job)
  def can_move?(board, target)
    raise NotImplementedError, "#{self.class} must implement #can_move?"
  end

  def piece_type
    raise NotImplementedError, "#{self.class} must implement #piece_type"
  end

  def symbol
    raise NotImplementedError, "#{self.class} must implement #symbol"
  end

  # Get all positions this piece can potentially move to (for check/checkmate detection)
  def possible_moves(board)
    moves = []
    8.times do |r|
      8.times do |c|
        target = Position.new(r, c)
        moves << target if target != @position && can_move?(board, target)
      end
    end
    moves
  end
end
```

**Board class:**

```ruby
class Board
  def initialize
    @grid = Array.new(8) { Array.new(8, nil) }
  end

  def get_piece(pos)
    return nil unless pos.valid?
    @grid[pos.row][pos.col]
  end

  def set_piece(pos, piece)
    @grid[pos.row][pos.col] = piece if pos.valid?
  end

  def remove_piece(pos)
    @grid[pos.row][pos.col] = nil if pos.valid?
  end

  def move_piece(from, to)
    piece = get_piece(from)
    if piece
      @grid[to.row][to.col] = piece
      @grid[from.row][from.col] = nil
      piece.set_position(to)
    end
  end

  # Check if path between two positions is clear (for sliding pieces: rook, bishop, queen)
  def path_clear?(from, to)
    dr = to.row <=> from.row   # -1, 0, or 1
    dc = to.col <=> from.col

    r = from.row + dr
    c = from.col + dc

    while r != to.row || c != to.col
      return false unless @grid[r][c].nil?
      r += dr
      c += dc
    end
    true
  end

  # Find the king of a given color
  def find_king(color)
    8.times do |r|
      8.times do |c|
        p = @grid[r][c]
        if p && p.piece_type == PieceType::KING && p.color == color
          return Position.new(r, c)
        end
      end
    end
    raise "King not found — invalid board state"
  end

  # Get all pieces of a given color
  def get_pieces(color)
    pieces = []
    8.times do |r|
      8.times do |c|
        p = @grid[r][c]
        pieces << p if p && p.color == color
      end
    end
    pieces
  end

  # Check if a square is attacked by any piece of the given color
  def square_attacked?(pos, by_color)
    get_pieces(by_color).any? { |piece| piece.can_move?(self, pos) }
  end

  def display
    puts "  a b c d e f g h"
    7.downto(0) do |r|
      print "#{r + 1} "
      8.times do |c|
        piece = @grid[r][c]
        print piece ? "#{piece.symbol} " : ". "
      end
      puts r + 1
    end
    puts "  a b c d e f g h"
  end
end
```

**Concrete Piece classes:**

```ruby
class King < Piece
  def piece_type = PieceType::KING
  def symbol = color == Color::WHITE ? 'K' : 'k'

  def can_move?(board, target)
    return false unless target.valid?

    dr = (target.row - position.row).abs
    dc = (target.col - position.col).abs

    # King moves one square in any direction
    return false if dr > 1 || dc > 1
    return false if dr == 0 && dc == 0

    # Can't capture own piece
    dest = board.get_piece(target)
    return false if dest && dest.color == color

    true
  end
end

class Queen < Piece
  def piece_type = PieceType::QUEEN
  def symbol = color == Color::WHITE ? 'Q' : 'q'

  def can_move?(board, target)
    return false unless target.valid?

    dr = (target.row - position.row).abs
    dc = (target.col - position.col).abs

    straight = (dr == 0 || dc == 0)
    diagonal = (dr == dc)
    return false unless straight || diagonal

    return false unless board.path_clear?(position, target)

    dest = board.get_piece(target)
    return false if dest && dest.color == color

    true
  end
end

class Rook < Piece
  def piece_type = PieceType::ROOK
  def symbol = color == Color::WHITE ? 'R' : 'r'

  def can_move?(board, target)
    return false unless target.valid?

    dr = (target.row - position.row).abs
    dc = (target.col - position.col).abs

    # Rook moves in straight lines
    return false if dr != 0 && dc != 0

    return false unless board.path_clear?(position, target)

    dest = board.get_piece(target)
    return false if dest && dest.color == color

    true
  end
end

class Bishop < Piece
  def piece_type = PieceType::BISHOP
  def symbol = color == Color::WHITE ? 'B' : 'b'

  def can_move?(board, target)
    return false unless target.valid?

    dr = (target.row - position.row).abs
    dc = (target.col - position.col).abs

    # Bishop moves diagonally
    return false if dr != dc || dr == 0

    return false unless board.path_clear?(position, target)

    dest = board.get_piece(target)
    return false if dest && dest.color == color

    true
  end
end

class Knight < Piece
  def piece_type = PieceType::KNIGHT
  def symbol = color == Color::WHITE ? 'N' : 'n'

  def can_move?(board, target)
    return false unless target.valid?

    dr = (target.row - position.row).abs
    dc = (target.col - position.col).abs

    # Knight moves in L-shape: 2+1 or 1+2
    return false unless (dr == 2 && dc == 1) || (dr == 1 && dc == 2)

    # Knight can jump over pieces — no path check needed

    dest = board.get_piece(target)
    return false if dest && dest.color == color

    true
  end
end

class Pawn < Piece
  def piece_type = PieceType::PAWN
  def symbol = color == Color::WHITE ? 'P' : 'p'

  def can_move?(board, target)
    return false unless target.valid?

    direction = color == Color::WHITE ? 1 : -1
    start_row = color == Color::WHITE ? 1 : 6

    dr = target.row - position.row
    dc = target.col - position.col

    # Forward move (1 square)
    if dc == 0 && dr == direction
      return board.get_piece(target).nil?
    end

    # Forward move (2 squares from starting position)
    if dc == 0 && dr == 2 * direction && position.row == start_row
      intermediate = Position.new(position.row + direction, position.col)
      return board.get_piece(intermediate).nil? && board.get_piece(target).nil?
    end

    # Diagonal capture
    if dc.abs == 1 && dr == direction
      dest = board.get_piece(target)
      return true if dest && dest.color != color
      # En passant is handled separately in the Game class
      return false
    end

    false
  end
end
```

**Move record and Game class:**

```ruby
class Move
  attr_reader :from, :to, :piece, :captured
  attr_accessor :is_castling, :is_en_passant, :is_promotion, :promotion_type

  def initialize(from, to, piece, captured = nil)
    @from = from
    @to = to
    @piece = piece
    @captured = captured
    @is_castling = false
    @is_en_passant = false
    @is_promotion = false
    @promotion_type = PieceType::QUEEN
  end
end

class Game
  attr_reader :status, :current_turn

  def initialize
    @board = Board.new
    @all_pieces = []  # holds references to all piece objects
    @current_turn = Color::WHITE
    @status = GameStatus::ACTIVE
    @move_history = []
    setup_board
  end

  def setup_board
    # Place white pieces
    place_piece(Rook,   Color::WHITE, Position.new(0, 0))
    place_piece(Knight, Color::WHITE, Position.new(0, 1))
    place_piece(Bishop, Color::WHITE, Position.new(0, 2))
    place_piece(Queen,  Color::WHITE, Position.new(0, 3))
    place_piece(King,   Color::WHITE, Position.new(0, 4))
    place_piece(Bishop, Color::WHITE, Position.new(0, 5))
    place_piece(Knight, Color::WHITE, Position.new(0, 6))
    place_piece(Rook,   Color::WHITE, Position.new(0, 7))
    8.times { |c| place_piece(Pawn, Color::WHITE, Position.new(1, c)) }

    # Place black pieces
    place_piece(Rook,   Color::BLACK, Position.new(7, 0))
    place_piece(Knight, Color::BLACK, Position.new(7, 1))
    place_piece(Bishop, Color::BLACK, Position.new(7, 2))
    place_piece(Queen,  Color::BLACK, Position.new(7, 3))
    place_piece(King,   Color::BLACK, Position.new(7, 4))
    place_piece(Bishop, Color::BLACK, Position.new(7, 5))
    place_piece(Knight, Color::BLACK, Position.new(7, 6))
    place_piece(Rook,   Color::BLACK, Position.new(7, 7))
    8.times { |c| place_piece(Pawn, Color::BLACK, Position.new(6, c)) }
  end

  def make_move(from, to)
    if [GameStatus::CHECKMATE, GameStatus::STALEMATE, GameStatus::RESIGNED].include?(@status)
      puts "Game is over!"
      return false
    end

    piece = @board.get_piece(from)
    unless piece
      puts "No piece at #{from}"
      return false
    end

    if piece.color != @current_turn
      puts "Not your turn!"
      return false
    end

    # Check for castling
    if piece.piece_type == PieceType::KING && (to.col - from.col).abs == 2
      return try_castling(from, to)
    end

    # Check for en passant
    if piece.piece_type == PieceType::PAWN && (to.col - from.col).abs == 1 &&
       @board.get_piece(to).nil?
      return try_en_passant(from, to)
    end

    # Normal move validation
    unless piece.can_move?(@board, to)
      puts "Invalid move for #{piece.symbol}"
      return false
    end

    # Simulate the move and check if it leaves own king in check
    captured = @board.get_piece(to)
    @board.move_piece(from, to)

    if in_check?(@current_turn)
      # Undo the move — can't leave own king in check
      @board.move_piece(to, from)
      piece.instance_variable_set(:@position, from)
      @board.set_piece(to, captured) if captured
      puts "Move leaves king in check!"
      return false
    end

    # Record the move
    move = Move.new(from, to, piece, captured)

    # Check for pawn promotion
    promotion_row = @current_turn == Color::WHITE ? 7 : 0
    if piece.piece_type == PieceType::PAWN && to.row == promotion_row
      promote_pawn(to, PieceType::QUEEN)  # auto-promote to queen
      move.is_promotion = true
    end

    @move_history << move

    # Switch turns
    @current_turn = Color.opposite(@current_turn)

    # Update game status
    update_game_status

    true
  end

  def in_check?(color)
    king_pos = @board.find_king(color)
    @board.square_attacked?(king_pos, Color.opposite(color))
  end

  def checkmate?(color)
    return false unless in_check?(color)
    !has_any_legal_move?(color)
  end

  def stalemate?(color)
    return false if in_check?(color)
    !has_any_legal_move?(color)
  end

  def display
    @board.display
  end

  private

  def place_piece(klass, color, pos)
    piece = klass.new(color, pos)
    @board.set_piece(pos, piece)
    @all_pieces << piece
  end

  def has_any_legal_move?(color)
    @board.get_pieces(color).each do |piece|
      piece.possible_moves(@board).each do |target|
        from = piece.position

        # Simulate each move
        captured = @board.get_piece(target)
        @board.move_piece(from, target)

        still_in_check = in_check?(color)

        # Undo
        @board.move_piece(target, from)
        piece.instance_variable_set(:@position, from)
        @board.set_piece(target, captured) if captured

        return true unless still_in_check
      end
    end
    false
  end

  def try_castling(from, to)
    king = @board.get_piece(from)
    return false if king.has_moved
    return false if in_check?(@current_turn)

    king_side = to.col > from.col
    rook_col = king_side ? 7 : 0
    rook_pos = Position.new(from.row, rook_col)
    rook = @board.get_piece(rook_pos)

    return false unless rook && rook.piece_type == PieceType::ROOK && !rook.has_moved

    # Check that path between king and rook is clear
    start_col = [from.col, rook_col].min + 1
    end_col = [from.col, rook_col].max
    (start_col...end_col).each do |c|
      return false unless @board.get_piece(Position.new(from.row, c)).nil?
    end

    # King cannot pass through or land on attacked squares
    direction = king_side ? 1 : -1
    (1..2).each do |i|
      passing = Position.new(from.row, from.col + i * direction)
      return false if @board.square_attacked?(passing, Color.opposite(@current_turn))
    end

    # Execute castling
    new_rook_col = king_side ? 5 : 3
    @board.move_piece(from, to)
    @board.move_piece(rook_pos, Position.new(from.row, new_rook_col))

    move = Move.new(from, to, king)
    move.is_castling = true
    @move_history << move

    @current_turn = Color.opposite(@current_turn)
    update_game_status
    true
  end

  def try_en_passant(from, to)
    return false if @move_history.empty?

    last_move = @move_history.last
    # En passant: last move was a pawn moving 2 squares, landing beside our pawn
    return false unless last_move.piece.piece_type == PieceType::PAWN
    return false unless (last_move.to.row - last_move.from.row).abs == 2
    return false unless last_move.to.row == from.row
    return false unless last_move.to.col == to.col

    # Capture the pawn
    captured = @board.get_piece(last_move.to)
    @board.remove_piece(last_move.to)
    @board.move_piece(from, to)

    # Verify king is not in check after en passant
    if in_check?(@current_turn)
      # Undo
      @board.move_piece(to, from)
      @board.get_piece(from)&.instance_variable_set(:@position, from)
      @board.set_piece(last_move.to, captured)
      puts "En passant leaves king in check!"
      return false
    end

    move = Move.new(from, to, @board.get_piece(to), captured)
    move.is_en_passant = true
    @move_history << move

    @current_turn = Color.opposite(@current_turn)
    update_game_status
    true
  end

  def promote_pawn(pos, type)
    pawn = @board.get_piece(pos)
    color = pawn.color
    @board.remove_piece(pos)

    klass = case type
            when PieceType::QUEEN  then Queen
            when PieceType::ROOK   then Rook
            when PieceType::BISHOP then Bishop
            when PieceType::KNIGHT then Knight
            else Queen
            end

    place_piece(klass, color, pos)
  end

  def update_game_status
    if checkmate?(@current_turn)
      @status = GameStatus::CHECKMATE
      puts "#{Color.to_s(Color.opposite(@current_turn))} wins by checkmate!"
    elsif stalemate?(@current_turn)
      @status = GameStatus::STALEMATE
      puts "Game drawn by stalemate!"
    elsif in_check?(@current_turn)
      @status = GameStatus::CHECK
      puts "#{Color.to_s(@current_turn)} is in check!"
    else
      @status = GameStatus::ACTIVE
    end
  end
end
```

**Usage example:**

```ruby
game = Game.new
game.display

# Scholar's Mate (4-move checkmate)
game.make_move(Position.new(1, 4), Position.new(3, 4))  # e2-e4
game.make_move(Position.new(6, 4), Position.new(4, 4))  # e7-e5
game.make_move(Position.new(0, 5), Position.new(3, 2))  # Bf1-c4
game.make_move(Position.new(7, 1), Position.new(5, 2))  # Nb8-c6
game.make_move(Position.new(0, 3), Position.new(4, 7))  # Qd1-h5
game.make_move(Position.new(7, 6), Position.new(5, 5))  # Ng8-f6
game.make_move(Position.new(4, 7), Position.new(6, 5))  # Qh5-f7# (checkmate)

game.display
# Output: White wins by checkmate!
```

**Design patterns used:**
- **Strategy**: Each piece type encapsulates its own movement strategy via `can_move?`
- **Command**: `Move` struct records each move for history/undo
- **Observer**: Game status updates after each move (could be extended to notify UI)
- **Template Method**: `possible_moves` uses `can_move?` as a hook method

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
| Storage | `Hash` in each directory | O(1) lookup by name within a directory |
| Permissions | Bitmask (rwx) | Simple, efficient, mirrors Unix permissions |

---

### Implementation

```ruby
# ─── Enums and Base Class ────────────────────────────────

module EntryType
  FILE      = :file
  DIRECTORY = :directory
end

# Permission bitmask
module Permission
  NONE    = 0
  EXECUTE = 1  # 001
  WRITE   = 2  # 010
  READ    = 4  # 100
  RW      = READ | WRITE
  RWX     = READ | WRITE | EXECUTE
end

# Abstract base class for file system entries (Composite pattern)
class FSEntry
  attr_reader :name, :created_at, :modified_at, :permissions
  attr_accessor :parent

  def initialize(name, perms = Permission::RWX)
    @name = name
    @permissions = perms
    @created_at = Time.now
    @modified_at = @created_at
    @parent = nil
  end

  def set_name(n)
    @name = n
    touch
  end

  def set_permissions(p)
    @permissions = p
  end

  def touch
    @modified_at = Time.now
  end

  def has_permission?(perm)
    (@permissions & perm) != 0
  end

  # Get the full path from root to this entry
  def path
    return "/" unless @parent
    parent_path = @parent.path
    parent_path == "/" ? "/#{@name}" : "#{parent_path}/#{@name}"
  end

  def entry_type
    raise NotImplementedError
  end

  def size
    raise NotImplementedError
  end

  def display(indent = 0)
    raise NotImplementedError
  end
end
```

**File class:**

```ruby
class FSFile < FSEntry
  def initialize(name, perms = Permission::RW)
    super(name, perms)
    @content = ""
  end

  def entry_type = EntryType::FILE

  def size = @content.size

  def content
    unless has_permission?(Permission::READ)
      raise "Permission denied: cannot read #{path}"
    end
    @content
  end

  def set_content(data)
    unless has_permission?(Permission::WRITE)
      raise "Permission denied: cannot write #{path}"
    end
    @content = data
    touch
  end

  def append_content(data)
    unless has_permission?(Permission::WRITE)
      raise "Permission denied: cannot write #{path}"
    end
    @content += data
    touch
  end

  def display(indent = 0)
    puts "#{' ' * (indent * 2)}#{@name} (#{size} bytes)"
  end
end
```

**Directory class:**

```ruby
class FSDirectory < FSEntry
  def initialize(name, perms = Permission::RWX)
    super(name, perms)
    @children = {}  # name => FSEntry
  end

  def entry_type = EntryType::DIRECTORY

  # Directory size = sum of all children sizes (recursive)
  def size
    @children.values.sum(&:size)
  end

  def add_child(entry)
    unless has_permission?(Permission::WRITE)
      raise "Permission denied: cannot write to #{path}"
    end
    if @children.key?(entry.name)
      raise "Entry already exists: #{entry.name}"
    end
    entry.parent = self
    @children[entry.name] = entry
    touch
    entry
  end

  def get_child(name)
    unless has_permission?(Permission::READ)
      raise "Permission denied: cannot read #{path}"
    end
    @children[name]
  end

  def remove_child(name)
    unless has_permission?(Permission::WRITE)
      raise "Permission denied: cannot write to #{path}"
    end
    return false unless @children.key?(name)
    @children.delete(name)
    touch
    true
  end

  def list_children
    unless has_permission?(Permission::READ)
      raise "Permission denied: cannot read #{path}"
    end
    @children.values
  end

  def has_child?(name)
    @children.key?(name)
  end

  def display(indent = 0)
    puts "#{' ' * (indent * 2)}#{@name}/"
    @children.each_value { |entry| entry.display(indent + 1) }
  end
end
```

**FileSystem class (facade):**

```ruby
class FileSystem
  def initialize
    @root = FSDirectory.new("")
  end

  # Parse a path into components: "/home/user/file.txt" → ["home", "user", "file.txt"]
  def self.parse_path(path)
    parts = []
    path.split("/").each do |part|
      next if part.empty? || part == "."
      if part == ".."
        parts.pop unless parts.empty?
      else
        parts << part
      end
    end
    parts
  end

  # Resolve a path to an FSEntry
  def resolve(path)
    parts = self.class.parse_path(path)
    current = @root

    parts.each do |part|
      return nil unless current.entry_type == EntryType::DIRECTORY
      current = current.get_child(part)
      return nil unless current
    end
    current
  end

  # Resolve parent directory of a path
  def resolve_parent(path)
    parts = self.class.parse_path(path)
    return @root if parts.empty?

    parts.pop  # remove the last component (the entry itself)

    current = @root
    parts.each do |part|
      return nil unless current.entry_type == EntryType::DIRECTORY
      current = current.get_child(part)
      return nil unless current
    end

    return nil unless current.entry_type == EntryType::DIRECTORY
    current
  end

  # Create a file at the given path
  def create_file(path, content = "")
    parts = self.class.parse_path(path)
    raise "Invalid file path" if parts.empty?

    filename = parts.last
    parent = resolve_parent(path)
    raise "Parent directory not found: #{path}" unless parent

    file = FSFile.new(filename)
    file.set_content(content) unless content.empty?
    parent.add_child(file)
  end

  # Create a directory at the given path
  def create_directory(path)
    parts = self.class.parse_path(path)
    raise "Invalid directory path" if parts.empty?

    dirname = parts.last
    parent = resolve_parent(path)
    raise "Parent directory not found: #{path}" unless parent

    parent.add_child(FSDirectory.new(dirname))
  end

  # Create directories recursively (like mkdir -p)
  def mkdirp(path)
    parts = self.class.parse_path(path)
    current = @root

    parts.each do |part|
      child = current.get_child(part)
      unless child
        child = current.add_child(FSDirectory.new(part))
      end
      raise "#{part} exists and is not a directory" unless child.entry_type == EntryType::DIRECTORY
      current = child
    end
    current
  end

  # Read file content
  def read_file(path)
    entry = resolve(path)
    raise "File not found: #{path}" unless entry
    raise "Not a file: #{path}" unless entry.entry_type == EntryType::FILE
    entry.content
  end

  # Write to a file
  def write_file(path, content)
    entry = resolve(path)
    raise "File not found: #{path}" unless entry
    raise "Not a file: #{path}" unless entry.entry_type == EntryType::FILE
    entry.set_content(content)
  end

  # Delete a file or directory
  def remove(path)
    parts = self.class.parse_path(path)
    raise "Cannot delete root" if parts.empty?

    parent = resolve_parent(path)
    return false unless parent

    parent.remove_child(parts.last)
  end

  # List directory contents
  def ls(path)
    entry = resolve(path)
    raise "Path not found: #{path}" unless entry
    raise "Not a directory: #{path}" unless entry.entry_type == EntryType::DIRECTORY
    entry.list_children
  end

  # Search for files by name (recursive)
  def find(path, name)
    results = []
    start = resolve(path)
    return results unless start

    find_recursive(start, name, results)
    results
  end

  # Get total size of a path
  def get_size(path)
    entry = resolve(path)
    raise "Path not found: #{path}" unless entry
    entry.size
  end

  # Display the entire file system tree
  def display
    @root.display
  end

  private

  def find_recursive(entry, name, results)
    if entry.name == name || match_wildcard?(entry.name, name)
      results << entry.path
    end
    if entry.entry_type == EntryType::DIRECTORY
      entry.list_children.each { |child| find_recursive(child, name, results) }
    end
  end

  # Simple wildcard matching: "*.txt" matches "file.txt"
  def match_wildcard?(str, pattern)
    return str.empty? if pattern.empty?
    if pattern[0] == '*'
      suffix = pattern[1..]
      return true if suffix.empty?
      str.size >= suffix.size && str.end_with?(suffix)
    else
      str == pattern
    end
  end
end
```

**Usage example:**

```ruby
fs = FileSystem.new

# Create directory structure
fs.mkdirp("/home/user/documents")
fs.mkdirp("/home/user/pictures")
fs.mkdirp("/etc/config")

# Create files
fs.create_file("/home/user/documents/readme.txt", "Hello, World!")
fs.create_file("/home/user/documents/notes.txt", "Some notes here")
fs.create_file("/home/user/pictures/photo.jpg", "<binary data>")
fs.create_file("/etc/config/app.conf", "port=8080\nhost=localhost")

# Read a file
puts fs.read_file("/home/user/documents/readme.txt")
# Output: Hello, World!

# List directory
fs.ls("/home/user").each do |entry|
  type = entry.entry_type == EntryType::DIRECTORY ? "dir" : "file"
  puts "#{entry.name} (#{type})"
end
# Output: documents (dir)
#         pictures (dir)

# Search for .txt files
results = fs.find("/", "*.txt")
results.each { |path| puts "Found: #{path}" }
# Output: Found: /home/user/documents/readme.txt
#         Found: /home/user/documents/notes.txt

# Get directory size
puts "Size: #{fs.get_size('/home/user')} bytes"

# Display tree
fs.display
```

**Design patterns used:**
- **Composite**: `FSEntry` base with `FSFile` (leaf) and `FSDirectory` (composite) — uniform treatment
- **Iterator**: `list_children` and recursive `find` traverse the tree
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
| Cell value storage | Number or String | Cells can hold numbers or text |
| Formula parsing | Simple recursive descent parser | Handles arithmetic and cell references |
| Dependency tracking | Adjacency list (DAG) | Efficient for topological sort and cycle detection |
| Recalculation order | BFS from changed cell through dependents | Ensures dependencies are calculated before dependents |
| Cycle detection | DFS with coloring (white/gray/black) | Standard algorithm for cycle detection in directed graphs |

---

### Implementation

**Cell addressing utilities:**

```ruby
# ─── Cell Addressing ─────────────────────────────────────

module CellAddress
  # Convert column letter(s) to index: A→0, B→1, ..., Z→25, AA→26
  def self.col_to_index(col)
    result = 0
    col.each_char { |c| result = result * 26 + (c.upcase.ord - 'A'.ord + 1) }
    result - 1
  end

  # Convert column index to letter(s): 0→A, 1→B, ..., 25→Z, 26→AA
  def self.index_to_col(index)
    result = ""
    index += 1
    while index > 0
      index -= 1
      result = (('A'.ord + index % 26).chr) + result
      index /= 26
    end
    result
  end
end

class CellId
  attr_reader :row, :col

  def initialize(row, col)
    @row = row
    @col = col
  end

  # Parse "A1", "B3", "AA10" etc.
  def self.parse(ref)
    i = 0
    i += 1 while i < ref.size && ref[i].match?(/[A-Za-z]/)
    col_str = ref[0...i]
    row_num = ref[i..].to_i - 1  # 1-indexed to 0-indexed
    CellId.new(row_num, CellAddress.col_to_index(col_str))
  end

  def to_s
    "#{CellAddress.index_to_col(@col)}#{@row + 1}"
  end

  def ==(other)
    other.is_a?(CellId) && @row == other.row && @col == other.col
  end

  def eql?(other)
    self == other
  end

  def hash
    [@row, @col].hash
  end
end
```

**Expression AST and parser:**

```ruby
# ─── AST Nodes for Formula Expressions ──────────────────

class Expr
  def evaluate(_get_cell_value)
    raise NotImplementedError
  end

  def dependencies
    raise NotImplementedError
  end
end

class NumberExpr < Expr
  def initialize(value)
    @value = value
  end

  def evaluate(_get_cell_value) = @value
  def dependencies = []
end

class CellRefExpr < Expr
  def initialize(cell_id)
    @cell_id = cell_id
  end

  def evaluate(get_cell_value)
    get_cell_value.call(@cell_id)
  end

  def dependencies = [@cell_id]
end

class BinaryExpr < Expr
  def initialize(op, left, right)
    @op = op
    @left = left
    @right = right
  end

  def evaluate(get_cell_value)
    l = @left.evaluate(get_cell_value)
    r = @right.evaluate(get_cell_value)
    case @op
    when '+' then l + r
    when '-' then l - r
    when '*' then l * r
    when '/'
      raise "Division by zero" if r == 0.0
      l / r
    else
      raise "Unknown operator: #{@op}"
    end
  end

  def dependencies
    @left.dependencies + @right.dependencies
  end
end

# ─── Recursive Descent Parser ────────────────────────────
# Grammar:
#   expr     → term (('+' | '-') term)*
#   term     → factor (('*' | '/') factor)*
#   factor   → NUMBER | CELL_REF | '(' expr ')' | '-' factor

class FormulaParser
  def parse(formula)
    @input = formula
    @pos = 0
    result = parse_expr
    skip_spaces
    if @pos != @input.size
      raise "Unexpected trailing characters in formula"
    end
    result
  end

  private

  def peek
    @pos < @input.size ? @input[@pos] : nil
  end

  def advance
    ch = @input[@pos]
    @pos += 1
    ch
  end

  def skip_spaces
    @pos += 1 while @pos < @input.size && @input[@pos] == ' '
  end

  def parse_expr
    left = parse_term
    skip_spaces
    while peek == '+' || peek == '-'
      op = advance
      right = parse_term
      left = BinaryExpr.new(op, left, right)
      skip_spaces
    end
    left
  end

  def parse_term
    left = parse_factor
    skip_spaces
    while peek == '*' || peek == '/'
      op = advance
      right = parse_factor
      left = BinaryExpr.new(op, left, right)
      skip_spaces
    end
    left
  end

  def parse_factor
    skip_spaces

    # Parenthesized expression
    if peek == '('
      advance  # consume '('
      expr = parse_expr
      skip_spaces
      raise "Expected ')'" unless peek == ')'
      advance  # consume ')'
      return expr
    end

    # Negative number or negation
    if peek == '-'
      advance
      expr = parse_factor
      return BinaryExpr.new('-', NumberExpr.new(0), expr)
    end

    # Cell reference (starts with letter)
    if peek&.match?(/[A-Za-z]/)
      ref = ""
      ref += advance while peek&.match?(/[A-Za-z]/)
      ref += advance while peek&.match?(/[0-9]/)
      return CellRefExpr.new(CellId.parse(ref))
    end

    # Number
    if peek&.match?(/[0-9.]/)
      num = ""
      num += advance while peek&.match?(/[0-9.]/)
      return NumberExpr.new(num.to_f)
    end

    raise "Unexpected character: #{peek}"
  end
end
```

**Cell and Spreadsheet classes:**

```ruby
# ─── Cell ────────────────────────────────────────────────

class Cell
  attr_reader :id, :raw_input, :cached_value, :dependencies

  def initialize(id)
    @id = id
    @raw_input = ""
    @cached_value = 0.0
    @formula = nil        # parsed formula (nil if raw value)
    @dependencies = []    # cells this cell depends on
  end

  def numeric_value
    @cached_value.is_a?(Numeric) ? @cached_value : 0.0
  end

  def has_formula?
    !@formula.nil?
  end

  # Set a raw value (number or string)
  def set_raw_value(input)
    @raw_input = input
    @formula = nil
    @dependencies = []

    # Try to parse as number
    if input.match?(/\A-?\d+(\.\d+)?\z/)
      @cached_value = input.to_f
    else
      @cached_value = input
    end
  end

  # Set a formula (starts with '=')
  def set_formula(input, parser)
    @raw_input = input
    formula_str = input[1..]  # remove '='
    @formula = parser.parse(formula_str)
    @dependencies = @formula.dependencies
  end

  # Evaluate the formula and update cached value
  def evaluate(get_cell_value)
    @cached_value = @formula.evaluate(get_cell_value) if @formula
  end

  def display_value
    if @cached_value.is_a?(Float)
      @cached_value == @cached_value.to_i ? @cached_value.to_i.to_s : @cached_value.to_s
    else
      @cached_value.to_s
    end
  end
end

# ─── Spreadsheet ─────────────────────────────────────────

class Spreadsheet
  def initialize
    @cells = {}        # CellId => Cell
    @dependents = {}   # CellId => Set of CellIds that depend on it
    @parser = FormulaParser.new
  end

  # Set a cell's value or formula
  def set_cell(ref, input)
    id = CellId.parse(ref)

    # Get or create the cell
    @cells[id] ||= Cell.new(id)
    cell = @cells[id]

    # Remove old dependencies
    cell.dependencies.each do |dep|
      @dependents[dep]&.delete(id)
    end

    # Parse new input
    if !input.empty? && input[0] == '='
      cell.set_formula(input, @parser)
    else
      cell.set_raw_value(input)
    end

    # Check for circular dependencies
    if cell.has_formula? && has_circular_dependency?(id)
      # Rollback
      cell.set_raw_value("0")
      raise "Circular dependency detected for #{ref}"
    end

    # Register new dependencies
    cell.dependencies.each do |dep|
      @dependents[dep] ||= Set.new
      @dependents[dep].add(id)
    end

    # Recalculate this cell and all dependents
    recalculate(id)
  end

  # Get a cell's display value
  def get_cell(ref)
    id = CellId.parse(ref)
    cell = @cells[id]
    cell ? cell.display_value : ""
  end

  # Get a cell's numeric value (for formula evaluation)
  def get_cell_numeric_value(id)
    cell = @cells[id]
    cell ? cell.numeric_value : 0.0
  end

  # Display the spreadsheet
  def display(rows = 5, cols = 5)
    # Header
    print "     "
    cols.times { |c| print "%-7s" % CellAddress.index_to_col(c) }
    puts

    rows.times do |r|
      print "%-5s" % (r + 1)
      cols.times do |c|
        id = CellId.new(r, c)
        cell = @cells[id]
        val = cell ? cell.display_value : ""
        val = val[0...7] if val.size > 7
        print "%-7s" % val
      end
      puts
    end
  end

  private

  # Detect circular dependencies using DFS with coloring
  def has_circular_dependency?(start)
    color = {}  # CellId => :white | :gray | :black

    dfs = lambda do |id|
      color[id] = :gray

      cell = @cells[id]
      if cell
        cell.dependencies.each do |dep|
          dep_color = color[dep] || :white
          return true if dep_color == :gray   # back edge → cycle
          return true if dep_color == :white && dfs.call(dep)
        end
      end

      color[id] = :black
      false
    end

    dfs.call(start)
  end

  # Recalculate a cell and all its dependents in BFS order
  def recalculate(start)
    order = []
    visited = Set.new

    queue = [start]
    visited.add(start)

    until queue.empty?
      current = queue.shift
      order << current

      if @dependents[current]
        @dependents[current].each do |dep|
          unless visited.include?(dep)
            visited.add(dep)
            queue << dep
          end
        end
      end
    end

    # Evaluate in order (start first, then its dependents)
    get_cell_value = ->(id) { get_cell_numeric_value(id) }

    order.each do |id|
      cell = @cells[id]
      cell.evaluate(get_cell_value) if cell&.has_formula?
    end
  end
end
```

**Usage example:**

```ruby
sheet = Spreadsheet.new

# Set raw values
sheet.set_cell("A1", "10")
sheet.set_cell("A2", "20")
sheet.set_cell("A3", "30")

# Set formulas
sheet.set_cell("B1", "=A1+A2")       # B1 = 10 + 20 = 30
sheet.set_cell("B2", "=A1*A3")       # B2 = 10 * 30 = 300
sheet.set_cell("C1", "=B1+B2")       # C1 = 30 + 300 = 330

puts "B1 = #{sheet.get_cell('B1')}"  # 30
puts "B2 = #{sheet.get_cell('B2')}"  # 300
puts "C1 = #{sheet.get_cell('C1')}"  # 330

# Change A1 — triggers recalculation of B1, B2, and C1
sheet.set_cell("A1", "100")
puts "\nAfter changing A1 to 100:"
puts "B1 = #{sheet.get_cell('B1')}"  # 120
puts "B2 = #{sheet.get_cell('B2')}"  # 3000
puts "C1 = #{sheet.get_cell('C1')}"  # 3120

# Circular dependency detection
begin
  sheet.set_cell("A1", "=C1")  # A1 depends on C1, which depends on B1, which depends on A1
rescue => e
  puts "\nError: #{e.message}"
  # Output: Error: Circular dependency detected for A1
end

sheet.display
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
| Text storage | `String` (simple) | String is simple for interviews; rope for large files |
| Undo/Redo | Command pattern with inverse operations | Each command knows how to undo itself |
| Clipboard | Singleton clipboard shared across operations | Standard editor behavior |
| Cursor | Integer offset into the text buffer | Simple, efficient for string operations |

---

### Implementation

**Document (the text buffer):**

```ruby
class Document
  attr_reader :content, :cursor_pos

  def initialize
    @content = ""
    @cursor_pos = 0
    @selection_start = -1
    @selection_end = -1
  end

  # ─── Cursor operations ──────────────────────────────

  def set_cursor_pos(pos)
    @cursor_pos = [[0, pos].max, @content.size].min
    clear_selection
  end

  def move_cursor_left(n = 1)  = set_cursor_pos(@cursor_pos - n)
  def move_cursor_right(n = 1) = set_cursor_pos(@cursor_pos + n)
  def move_cursor_to_start     = set_cursor_pos(0)
  def move_cursor_to_end       = set_cursor_pos(@content.size)

  # ─── Selection ──────────────────────────────────────

  def set_selection(start_pos, end_pos)
    @selection_start = [[0, start_pos].max, @content.size].min
    @selection_end = [[0, end_pos].max, @content.size].min
    @selection_start, @selection_end = @selection_end, @selection_start if @selection_start > @selection_end
  end

  def clear_selection
    @selection_start = @selection_end = -1
  end

  def has_selection?
    @selection_start >= 0 && @selection_end >= 0 && @selection_start != @selection_end
  end

  def selection
    [@selection_start, @selection_end]
  end

  def selected_text
    return "" unless has_selection?
    @content[@selection_start...@selection_end]
  end

  # ─── Text modification (low-level, used by commands) ─

  def insert_at(pos, text)
    pos = [[0, pos].max, @content.size].min
    @content.insert(pos, text)
  end

  def delete_range(start, length)
    start = [[0, start].max, @content.size].min
    length = [length, @content.size - start].min
    deleted = @content[start, length]
    @content.slice!(start, length)
    deleted
  end

  # ─── Search ─────────────────────────────────────────

  def find(query, from_pos = 0)
    pos = @content.index(query, from_pos)
    pos.nil? ? -1 : pos
  end

  def find_all(query)
    positions = []
    pos = 0
    while (idx = @content.index(query, pos))
      positions << idx
      pos = idx + query.size
    end
    positions
  end

  def display
    puts @content
    cursor_line = ' ' * @cursor_pos + '^'
    puts "#{cursor_line} (pos #{@cursor_pos})"
  end
end
```

**Command interface and concrete commands:**

```ruby
# ─── Command Interface ───────────────────────────────────

class Command
  def execute
    raise NotImplementedError
  end

  def undo
    raise NotImplementedError
  end

  def describe
    raise NotImplementedError
  end
end

# Insert text at a position
class InsertCommand < Command
  def initialize(doc, position, text)
    @doc = doc
    @position = position
    @text = text
  end

  def execute
    @doc.insert_at(@position, @text)
    @doc.set_cursor_pos(@position + @text.size)
  end

  def undo
    @doc.delete_range(@position, @text.size)
    @doc.set_cursor_pos(@position)
  end

  def describe
    "Insert \"#{@text}\" at #{@position}"
  end
end

# Delete text at a position
class DeleteCommand < Command
  def initialize(doc, position, length)
    @doc = doc
    @position = position
    @length = length
    @deleted_text = ""  # saved for undo
  end

  def execute
    @deleted_text = @doc.delete_range(@position, @length)
    @doc.set_cursor_pos(@position)
  end

  def undo
    @doc.insert_at(@position, @deleted_text)
    @doc.set_cursor_pos(@position + @deleted_text.size)
  end

  def describe
    "Delete #{@length} chars at #{@position} (\"#{@deleted_text}\")"
  end
end

# Replace text (delete + insert as a single undoable operation)
class ReplaceCommand < Command
  def initialize(doc, position, old_text, new_text)
    @doc = doc
    @position = position
    @old_text = old_text
    @new_text = new_text
  end

  def execute
    @doc.delete_range(@position, @old_text.size)
    @doc.insert_at(@position, @new_text)
    @doc.set_cursor_pos(@position + @new_text.size)
  end

  def undo
    @doc.delete_range(@position, @new_text.size)
    @doc.insert_at(@position, @old_text)
    @doc.set_cursor_pos(@position + @old_text.size)
  end

  def describe
    "Replace \"#{@old_text}\" with \"#{@new_text}\" at #{@position}"
  end
end
```

**Clipboard:**

```ruby
class Clipboard
  @instance = nil

  def self.instance
    @instance ||= new
  end

  private_class_method :new

  def initialize
    @content = ""
  end

  def set(text)
    @content = text
  end

  def get
    @content
  end

  def empty?
    @content.empty?
  end
end
```

**TextEditor (orchestrator with undo/redo):**

```ruby
class TextEditor
  def initialize
    @doc = Document.new
    @undo_stack = []
    @redo_stack = []
  end

  # ─── Basic text operations ──────────────────────────

  def insert_text(text)
    execute_command(InsertCommand.new(@doc, @doc.cursor_pos, text))
  end

  def delete_backward(n = 1)
    pos = @doc.cursor_pos
    actual = [n, pos].min
    execute_command(DeleteCommand.new(@doc, pos - actual, actual)) if actual > 0
  end

  def delete_forward(n = 1)
    pos = @doc.cursor_pos
    actual = [n, @doc.content.size - pos].min
    execute_command(DeleteCommand.new(@doc, pos, actual)) if actual > 0
  end

  def delete_selection
    if @doc.has_selection?
      start_pos, end_pos = @doc.selection
      execute_command(DeleteCommand.new(@doc, start_pos, end_pos - start_pos))
    end
  end

  # ─── Cursor movement ────────────────────────────────

  def move_cursor_left(n = 1)  = @doc.move_cursor_left(n)
  def move_cursor_right(n = 1) = @doc.move_cursor_right(n)
  def move_cursor_to_start     = @doc.move_cursor_to_start
  def move_cursor_to_end       = @doc.move_cursor_to_end

  # ─── Selection ──────────────────────────────────────

  def select(start_pos, end_pos) = @doc.set_selection(start_pos, end_pos)
  def select_all = @doc.set_selection(0, @doc.content.size)

  # ─── Clipboard operations ───────────────────────────

  def copy
    Clipboard.instance.set(@doc.selected_text) if @doc.has_selection?
  end

  def cut
    if @doc.has_selection?
      Clipboard.instance.set(@doc.selected_text)
      delete_selection
    end
  end

  def paste
    clip = Clipboard.instance.get
    unless clip.empty?
      if @doc.has_selection?
        start_pos, end_pos = @doc.selection
        old_text = @doc.selected_text
        execute_command(ReplaceCommand.new(@doc, start_pos, old_text, clip))
      else
        insert_text(clip)
      end
    end
  end

  # ─── Undo / Redo ────────────────────────────────────

  def undo
    if @undo_stack.empty?
      puts "Nothing to undo"
      return
    end
    cmd = @undo_stack.pop
    cmd.undo
    @redo_stack.push(cmd)
  end

  def redo
    if @redo_stack.empty?
      puts "Nothing to redo"
      return
    end
    cmd = @redo_stack.pop
    cmd.execute
    @undo_stack.push(cmd)
  end

  # ─── Find and Replace ──────────────────────────────

  def find(query)
    pos = @doc.find(query, @doc.cursor_pos)
    if pos >= 0
      @doc.set_cursor_pos(pos)
      @doc.set_selection(pos, pos + query.size)
      puts "Found at position #{pos}"
    else
      puts "Not found: \"#{query}\""
    end
    pos
  end

  def replace_next(query, replacement)
    pos = @doc.find(query, @doc.cursor_pos)
    if pos >= 0
      execute_command(ReplaceCommand.new(@doc, pos, query, replacement))
      return pos
    end
    -1
  end

  def replace_all(query, replacement)
    positions = @doc.find_all(query)
    return 0 if positions.empty?

    # Replace from end to start to preserve positions
    count = 0
    positions.reverse_each do |pos|
      execute_command(ReplaceCommand.new(@doc, pos, query, replacement))
      count += 1
    end
    count
  end

  # ─── Display ────────────────────────────────────────

  def display     = @doc.display
  def get_content = @doc.content
  def cursor_pos  = @doc.cursor_pos

  private

  def execute_command(cmd)
    cmd.execute
    @undo_stack.push(cmd)
    @redo_stack.clear  # new action invalidates redo history
  end
end
```

**Usage example:**

```ruby
editor = TextEditor.new

# Type some text
editor.insert_text("Hello World")
editor.display
# Output: Hello World
#                    ^ (pos 11)

# Move cursor and insert
editor.move_cursor_left(6)
editor.insert_text("Beautiful ")
editor.display
# Output: Hello Beautiful World
#                         ^ (pos 15)

# Undo the insert
editor.undo
editor.display
# Output: Hello World
#              ^ (pos 5)

# Redo
editor.redo
editor.display
# Output: Hello Beautiful World
#                         ^ (pos 15)

# Select and copy
editor.select(6, 15)
editor.copy

# Move to end and paste
editor.move_cursor_to_end
editor.insert_text(" - ")
editor.paste
editor.display
# Output: Hello Beautiful World - Beautiful
#                                          ^ (pos 34)

# Find and replace
count = editor.replace_all("Beautiful", "Amazing")
puts "Replaced #{count} occurrences"
editor.display
# Output: Hello Amazing World - Amazing

# Multiple undos
editor.undo  # undo second replace
editor.undo  # undo first replace
editor.display
# Output: Hello Beautiful World - Beautiful
```

**Design patterns used:**
- **Command**: Each text operation is encapsulated as a command with `execute` and `undo`
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

```ruby
# ─── Enums ───────────────────────────────────────────────

module Suit
  HEARTS   = :hearts
  DIAMONDS = :diamonds
  CLUBS    = :clubs
  SPADES   = :spades
  NONE     = :none

  def self.to_s(suit)
    { HEARTS => "Hearts", DIAMONDS => "Diamonds",
      CLUBS => "Clubs", SPADES => "Spades", NONE => "" }[suit]
  end
end

module Rank
  ACE = 1; TWO = 2; THREE = 3; FOUR = 4; FIVE = 5; SIX = 6; SEVEN = 7
  EIGHT = 8; NINE = 9; TEN = 10; JACK = 11; QUEEN = 12; KING = 13
  # Special ranks for games like UNO
  SKIP = 14; REVERSE = 15; DRAW_TWO = 16; WILD = 17; WILD_DRAW_FOUR = 18

  NAMES = {
    ACE => "A", TWO => "2", THREE => "3", FOUR => "4", FIVE => "5",
    SIX => "6", SEVEN => "7", EIGHT => "8", NINE => "9", TEN => "10",
    JACK => "J", QUEEN => "Q", KING => "K"
  }.freeze

  def self.to_s(rank)
    NAMES[rank] || "?"
  end
end

# ─── Card ────────────────────────────────────────────────

class Card
  attr_reader :suit, :rank
  attr_accessor :face_up

  def initialize(suit, rank)
    @suit = suit
    @rank = rank
    @face_up = false
  end

  def value = @rank

  def flip
    @face_up = !@face_up
  end

  def to_s
    return "[??]" unless @face_up
    "[#{Rank.to_s(@rank)} #{Suit.to_s(@suit)}]"
  end
end

# ─── Deck ────────────────────────────────────────────────

class Deck
  def initialize
    @cards = []
  end

  def add_card(card)
    @cards << card
  end

  def shuffle!
    @cards.shuffle!
  end

  def draw_card
    raise "Deck is empty" if @cards.empty?
    @cards.pop
  end

  def empty?     = @cards.empty?
  def remaining  = @cards.size

  # Factory method for standard 52-card deck
  def self.create_standard_deck
    deck = Deck.new
    [Suit::HEARTS, Suit::DIAMONDS, Suit::CLUBS, Suit::SPADES].each do |suit|
      (1..13).each do |rank|
        deck.add_card(Card.new(suit, rank))
      end
    end
    deck
  end
end
```

**Player and Hand:**

```ruby
# ─── Hand ────────────────────────────────────────────────

class Hand
  def initialize
    @cards = []
  end

  def add_card(card)
    @cards << card
  end

  def remove_card(index)
    raise "Invalid card index" if index < 0 || index >= @cards.size
    @cards.delete_at(index)
  end

  def cards     = @cards
  def size      = @cards.size
  def empty?    = @cards.empty?
  def clear     = @cards.clear

  def show_all
    @cards.each { |card| card.face_up = true }
  end

  def to_s
    @cards.map(&:to_s).join(" ")
  end
end

# ─── Player ──────────────────────────────────────────────

class Player
  attr_reader :name, :hand
  attr_accessor :score

  def initialize(name)
    @name = name
    @hand = Hand.new
    @score = 0
  end

  def receive_card(card)
    @hand.add_card(card)
  end

  def play_card(index)
    @hand.remove_card(index)
  end
end
```

**Abstract Game class (Template Method):**

```ruby
# ─── Abstract Card Game (Template Method) ────────────────

class CardGame
  def initialize
    @players = []
    @deck = nil
    @current_player_index = 0
    @game_over = false
  end

  def add_player(name)
    @players << Player.new(name)
  end

  # Template Method — defines the game loop skeleton
  def play
    puts "=== Starting #{game_name} ==="

    # Step 1: Initialize the deck
    initialize_deck

    # Step 2: Shuffle
    @deck.shuffle!

    # Step 3: Deal cards to players
    deal_cards

    # Step 4: Play rounds until game is over
    until game_over?
      current = @players[@current_player_index]
      puts "\n--- #{current.name}'s turn ---"
      display_game_state(current)

      play_turn(current)

      if check_win_condition
        @game_over = true
        break
      end

      next_player
    end

    # Step 5: Determine and announce winner
    announce_winner
  end

  protected

  # Hook methods — subclasses override these
  def game_name          = raise(NotImplementedError)
  def initialize_deck    = raise(NotImplementedError)
  def deal_cards         = raise(NotImplementedError)
  def play_turn(_player) = raise(NotImplementedError)
  def check_win_condition = raise(NotImplementedError)
  def announce_winner    = raise(NotImplementedError)
  def display_game_state(_player) = raise(NotImplementedError)

  def game_over? = @game_over

  def next_player
    @current_player_index = (@current_player_index + 1) % @players.size
  end

  # Helper: deal n cards to each player
  def deal_to_all(n)
    n.times do
      @players.each do |player|
        unless @deck.empty?
          card = @deck.draw_card
          card.face_up = true
          player.receive_card(card)
        end
      end
    end
  end
end
```

**Blackjack implementation:**

```ruby
class BlackjackGame < CardGame
  TARGET = 21

  protected

  def game_name = "Blackjack"

  def initialize_deck
    @deck = Deck.create_standard_deck
  end

  def deal_cards
    deal_to_all(2)  # 2 cards each
  end

  def play_turn(player)
    # Simple AI: hit if under 17, stand otherwise
    while hand_value(player.hand) < 17 && !@deck.empty?
      puts "#{player.name} hits!"
      card = @deck.draw_card
      card.face_up = true
      puts "  Drew: #{card}"
      player.receive_card(card)

      if bust?(player.hand)
        puts "#{player.name} BUSTS with #{hand_value(player.hand)}!"
        return
      end
    end
    puts "#{player.name} stands with #{hand_value(player.hand)}"
  end

  def check_win_condition
    # Game ends after all players have played one round
    @current_player_index == @players.size - 1
  end

  def display_game_state(player)
    puts "Hand: #{player.hand} (Value: #{hand_value(player.hand)})"
  end

  def announce_winner
    puts "\n=== Final Results ==="

    best_score = -1
    winner = nil

    @players.each do |player|
      val = hand_value(player.hand)
      print "#{player.name}: #{player.hand} = #{val}"

      if bust?(player.hand)
        puts " (BUST)"
      else
        puts
        if val > best_score
          best_score = val
          winner = player.name
        end
      end
    end

    if best_score > 0
      puts "\nWinner: #{winner} with #{best_score}!"
    else
      puts "\nEveryone busted! No winner."
    end
  end

  private

  def hand_value(hand)
    total = 0
    aces = 0

    hand.cards.each do |card|
      val = card.value
      if val == 1
        aces += 1
        total += 11
      elsif val >= 10
        total += 10
      else
        total += val
      end
    end

    # Reduce aces from 11 to 1 if over 21
    while total > TARGET && aces > 0
      total -= 10
      aces -= 1
    end
    total
  end

  def bust?(hand)
    hand_value(hand) > TARGET
  end
end
```

**Usage example:**

```ruby
# Play Blackjack
blackjack = BlackjackGame.new
blackjack.add_player("Alice")
blackjack.add_player("Bob")
blackjack.add_player("Charlie")
blackjack.play

# To add a new game (e.g., Poker), just create a new subclass:
# class PokerGame < CardGame ... end
# The game loop (play) stays the same — only the hook methods change.
```

**Design patterns used:**
- **Template Method**: `CardGame#play` defines the game loop; subclasses override hook methods
- **Strategy**: Scoring logic varies per game (Blackjack hand value vs Poker hand ranking)
- **Factory**: `Deck.create_standard_deck` creates game-specific decks
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
| Thread safety | `Mutex` with `synchronize` | Simple, correct; Ruby's GIL helps but explicit locking is still needed for correctness |

---

### Implementation

**Doubly Linked List Node:**

```ruby
class Node
  attr_accessor :key, :value, :prev, :next

  def initialize(key, value)
    @key = key
    @value = value
    @prev = nil
    @next = nil
  end
end
```

**LRU Cache (non-thread-safe version):**

```ruby
class LRUCache
  def initialize(capacity)
    raise ArgumentError, "Capacity must be positive" if capacity <= 0

    @capacity = capacity
    @cache = {}  # key => Node

    # Sentinel nodes simplify edge cases (no nil checks)
    @head = Node.new(nil, nil)  # dummy head (most recent side)
    @tail = Node.new(nil, nil)  # dummy tail (least recent side)
    @head.next = @tail
    @tail.prev = @head
  end

  # Get value by key — O(1)
  def get(key)
    node = @cache[key]
    return nil unless node  # cache miss

    # Move to front (most recently used)
    remove_node(node)
    add_to_front(node)

    node.value
  end

  # Put key-value pair — O(1)
  def put(key, value)
    if @cache.key?(key)
      # Key exists — update value and move to front
      node = @cache[key]
      node.value = value
      remove_node(node)
      add_to_front(node)
    else
      # New key — check capacity
      evict if @cache.size >= @capacity

      # Create new node and add to front
      node = Node.new(key, value)
      @cache[key] = node
      add_to_front(node)
    end
  end

  # Check if key exists — O(1) (does NOT update access order)
  def contains?(key)
    @cache.key?(key)
  end

  # Remove a specific key — O(1)
  def remove(key)
    node = @cache[key]
    return false unless node

    remove_node(node)
    @cache.delete(key)
    true
  end

  def size  = @cache.size
  def empty? = @cache.empty?

  # Display cache contents (most recent first)
  def display
    current = @head.next
    print "Cache [#{@cache.size}/#{@capacity}]: "
    while current != @tail
      print "(#{current.key}:#{current.value}) "
      current = current.next
    end
    puts
  end

  private

  # Remove a node from the linked list (does NOT delete it)
  def remove_node(node)
    node.prev.next = node.next
    node.next.prev = node.prev
  end

  # Add a node right after head (most recent position)
  def add_to_front(node)
    node.next = @head.next
    node.prev = @head
    @head.next.prev = node
    @head.next = node
  end

  # Evict the least recently used item (node before tail)
  def evict
    lru = @tail.prev
    return if lru == @head  # empty cache

    remove_node(lru)
    @cache.delete(lru.key)
  end
end
```

**Thread-safe version:**

```ruby
class ThreadSafeLRUCache
  def initialize(capacity)
    @cache = LRUCache.new(capacity)
    @mutex = Mutex.new
  end

  def get(key)
    @mutex.synchronize { @cache.get(key) }
  end

  def put(key, value)
    @mutex.synchronize { @cache.put(key, value) }
  end

  def contains?(key)
    @mutex.synchronize { @cache.contains?(key) }
  end

  def remove(key)
    @mutex.synchronize { @cache.remove(key) }
  end

  def size
    @mutex.synchronize { @cache.size }
  end
end
```

**Usage example:**

```ruby
cache = LRUCache.new(3)

cache.put(1, "one")
cache.put(2, "two")
cache.put(3, "three")
cache.display
# Cache [3/3]: (3:three) (2:two) (1:one)

# Access key 1 — moves it to front
val = cache.get(1)
puts "Got: #{val}"  # "one"
cache.display
# Cache [3/3]: (1:one) (3:three) (2:two)

# Add key 4 — evicts key 2 (least recently used)
cache.put(4, "four")
cache.display
# Cache [3/3]: (4:four) (1:one) (3:three)

# Key 2 was evicted
missing = cache.get(2)
puts "Key 2: #{missing || 'NOT FOUND'}"
# Output: Key 2: NOT FOUND

# Update existing key
cache.put(3, "THREE")
cache.display
# Cache [3/3]: (3:THREE) (4:four) (1:one)
```

**Complexity analysis:**

| Operation | Time | Space |
|-----------|------|-------|
| `get(key)` | O(1) | — |
| `put(key, value)` | O(1) | O(1) per entry |
| `remove(key)` | O(1) | — |
| `evict` | O(1) | — |
| Total space | — | O(capacity) |

**Why HashMap + Doubly Linked List?**
- **HashMap** gives O(1) lookup by key
- **Doubly Linked List** gives O(1) insertion and removal (given a reference to the node)
- The HashMap stores references to list nodes, so after finding a node by key, we can move it to the front in O(1)
- Sentinel nodes (dummy head/tail) eliminate nil checks for edge cases

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
| Message storage | Per-topic array with offset tracking | Enables replay and acknowledgment |
| Subscriber tracking | Per-subscriber offset per topic | Each subscriber progresses independently |
| Thread safety | Mutex per topic | Reduces contention vs single global lock |

---

### Implementation

**Message class:**

```ruby
require 'set'

class Message
  @@next_id = 0

  attr_reader :id, :topic, :key, :payload, :timestamp, :headers

  def initialize(topic, payload, key = "")
    @@next_id += 1
    @id = @@next_id
    @topic = topic
    @key = key
    @payload = payload
    @headers = {}
    @timestamp = Time.now
  end

  def set_header(k, v)
    @headers[k] = v
  end

  def get_header(k)
    @headers.fetch(k, "")
  end

  def to_s
    "[#{@id}] #{@topic}: #{@payload}"
  end
end
```

**Subscriber interface:**

```ruby
# ─── Subscriber Interface ────────────────────────────────

class ISubscriber
  def subscriber_id = raise(NotImplementedError)
  def on_message(_msg) = raise(NotImplementedError)
end

# Concrete subscriber with callback and optional filter
class Subscriber < ISubscriber
  def initialize(id, handler, filter = nil)
    @id = id
    @handler = handler   # Proc/lambda that receives a Message
    @filter = filter     # Proc/lambda that returns true/false for a Message
  end

  def subscriber_id = @id

  def on_message(msg)
    return if @filter && !@filter.call(msg)
    @handler.call(msg)
  end
end
```

**Topic class:**

```ruby
class Topic
  attr_reader :name

  def initialize(name)
    @name = name
    @messages = []           # message log (append-only)
    @subscribers = []
    @subscriber_offsets = {} # subscriber_id => last acked offset
    @mutex = Mutex.new
  end

  # Add a subscriber
  def subscribe(subscriber)
    @mutex.synchronize do
      return if @subscribers.any? { |s| s.subscriber_id == subscriber.subscriber_id }
      @subscribers << subscriber
      @subscriber_offsets[subscriber.subscriber_id] = @messages.size
      puts "#{subscriber.subscriber_id} subscribed to #{@name}"
    end
  end

  # Remove a subscriber
  def unsubscribe(subscriber_id)
    @mutex.synchronize do
      @subscribers.reject! { |s| s.subscriber_id == subscriber_id }
      @subscriber_offsets.delete(subscriber_id)
      puts "#{subscriber_id} unsubscribed from #{@name}"
    end
  end

  # Publish a message to this topic
  def publish(msg)
    @mutex.synchronize do
      @messages << msg

      @subscribers.each do |sub|
        begin
          sub.on_message(msg)
        rescue => e
          $stderr.puts "Error delivering to #{sub.subscriber_id}: #{e.message}"
        end
      end
    end
  end

  # Acknowledge a message (advance subscriber's offset)
  def acknowledge(subscriber_id, msg_id)
    @mutex.synchronize do
      offset = @subscriber_offsets[subscriber_id]
      return unless offset

      (offset...@messages.size).each do |i|
        if @messages[i].id == msg_id
          @subscriber_offsets[subscriber_id] = i + 1
          break
        end
      end
    end
  end

  # Get unacknowledged messages for a subscriber
  def get_unacknowledged(subscriber_id)
    @mutex.synchronize do
      offset = @subscriber_offsets[subscriber_id]
      return [] unless offset
      @messages[offset..]
    end
  end

  # Replay messages from a specific offset
  def replay(from_offset, count)
    @mutex.synchronize do
      @messages[from_offset, count] || []
    end
  end

  def message_count
    @mutex.synchronize { @messages.size }
  end

  def subscriber_count
    @mutex.synchronize { @subscribers.size }
  end
end
```

**MessageBroker (facade):**

```ruby
class MessageBroker
  def initialize
    @topics = {}
    @mutex = Mutex.new
  end

  # Create a topic
  def create_topic(name)
    @mutex.synchronize do
      return @topics[name] if @topics.key?(name)
      @topics[name] = Topic.new(name)
      puts "Topic created: #{name}"
      @topics[name]
    end
  end

  # Get a topic
  def get_topic(name)
    @mutex.synchronize { @topics[name] }
  end

  # Publish a message to a topic
  def publish(topic_name, payload, key = "")
    topic = nil
    @mutex.synchronize do
      @topics[topic_name] ||= Topic.new(topic_name)
      topic = @topics[topic_name]
    end

    msg = Message.new(topic_name, payload, key)
    topic.publish(msg)
  end

  # Subscribe to a topic
  def subscribe(topic_name, subscriber)
    topic = nil
    @mutex.synchronize do
      @topics[topic_name] ||= Topic.new(topic_name)
      topic = @topics[topic_name]
    end
    topic.subscribe(subscriber)
  end

  # Unsubscribe from a topic
  def unsubscribe(topic_name, subscriber_id)
    @mutex.synchronize do
      topic = @topics[topic_name]
      topic&.unsubscribe(subscriber_id)
    end
  end

  # List all topics
  def list_topics
    @mutex.synchronize { @topics.keys }
  end
end
```

**Usage example:**

```ruby
broker = MessageBroker.new

# Create subscribers
logger = Subscriber.new("logger",
  ->(msg) { puts "[LOG] #{msg}" }
)

alert_handler = Subscriber.new("alert-handler",
  ->(msg) { puts "[ALERT] Processing: #{msg.payload}" },
  # Filter: only messages containing "ERROR"
  ->(msg) { msg.payload.include?("ERROR") }
)

analytics = Subscriber.new("analytics",
  ->(msg) { puts "[ANALYTICS] Recorded: #{msg.payload}" }
)

# Subscribe to topics
broker.subscribe("orders", logger)
broker.subscribe("orders", analytics)
broker.subscribe("errors", logger)
broker.subscribe("errors", alert_handler)

# Publish messages
puts "\n--- Publishing messages ---"
broker.publish("orders", "New order #1001 placed")
broker.publish("orders", "Order #1001 shipped")
broker.publish("errors", "ERROR: Payment gateway timeout")
broker.publish("errors", "WARN: High latency detected")

# Output:
# [LOG] [1] orders: New order #1001 placed
# [ANALYTICS] Recorded: New order #1001 placed
# [LOG] [2] orders: Order #1001 shipped
# [ANALYTICS] Recorded: Order #1001 shipped
# [LOG] [3] errors: ERROR: Payment gateway timeout
# [ALERT] Processing: ERROR: Payment gateway timeout
# [LOG] [4] errors: WARN: High latency detected
# (alert_handler filters out the WARN message — no ERROR keyword)
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
| Task queue | Sort ready tasks by priority | Higher priority tasks execute first |
| Execution | Thread pool with configurable concurrency | Parallel execution of independent tasks |
| Retry policy | Configurable max retries | Handles transient failures |

---

### Implementation

**Task and TaskStatus:**

```ruby
require 'set'
require 'thread'

module TaskStatus
  PENDING   = :pending     # waiting for dependencies
  READY     = :ready       # dependencies satisfied, waiting to execute
  RUNNING   = :running     # currently executing
  COMPLETED = :completed   # finished successfully
  FAILED    = :failed      # failed after all retries
  CANCELLED = :cancelled   # cancelled (timeout or manual)
end

class Task
  attr_reader :id, :name, :priority, :dependencies, :error_message, :timeout
  attr_accessor :status

  def initialize(id, name, work:, priority: 0, max_retries: 0, timeout: nil)
    @id = id
    @name = name
    @priority = priority       # higher = more important
    @work = work               # Proc that returns true on success
    @dependencies = []         # IDs of tasks this depends on
    @status = TaskStatus::PENDING
    @max_retries = max_retries
    @retry_count = 0
    @timeout = timeout         # seconds, nil means no timeout
    @error_message = nil
  end

  def add_dependency(dep_id)
    @dependencies << dep_id
  end

  def can_retry?
    @retry_count < @max_retries
  end

  # Execute the task
  def execute
    @status = TaskStatus::RUNNING
    begin
      success = @work.call
      if success
        @status = TaskStatus::COMPLETED
        true
      else
        @retry_count += 1
        if can_retry?
          @status = TaskStatus::READY  # will be retried
        else
          @status = TaskStatus::FAILED
          @error_message = "Task returned false after #{@retry_count} attempts"
        end
        false
      end
    rescue => e
      @retry_count += 1
      @error_message = e.message
      if can_retry?
        @status = TaskStatus::READY
      else
        @status = TaskStatus::FAILED
      end
      false
    end
  end
end
```

**TaskScheduler:**

```ruby
class TaskScheduler
  def initialize(concurrency = 4)
    @tasks = {}            # id => Task
    @dependents = {}       # task_id => Set of task_ids that depend on it
    @in_degree = {}        # task_id => number of unsatisfied dependencies
    @max_concurrency = concurrency
    @mutex = Mutex.new
  end

  # Add a task to the scheduler
  def add_task(task)
    id = task.id

    deg = 0
    task.dependencies.each do |dep|
      @dependents[dep] ||= Set.new
      @dependents[dep].add(id)
      deg += 1
    end
    @in_degree[id] = deg

    @tasks[id] = task
  end

  # Validate the task graph (check for cycles)
  def validate
    # Kahn's algorithm for cycle detection
    deg = @in_degree.dup
    queue = []

    deg.each { |id, d| queue << id if d == 0 }

    processed = 0
    until queue.empty?
      current = queue.shift
      processed += 1

      if @dependents[current]
        @dependents[current].each do |dep|
          deg[dep] -= 1
          queue << dep if deg[dep] == 0
        end
      end
    end

    if processed != @tasks.size
      $stderr.puts "Circular dependency detected! Processed #{processed}" \
                   " of #{@tasks.size} tasks."
      return false
    end
    true
  end

  # Execute all tasks respecting dependencies and priority
  def execute
    unless validate
      raise "Cannot execute: circular dependencies detected"
    end

    puts "=== Starting Task Scheduler ==="
    puts "Tasks: #{@tasks.size}, Max concurrency: #{@max_concurrency}"

    # Initialize ready queue with tasks that have no dependencies
    ready_queue = []
    @tasks.each do |id, task|
      if @in_degree[id] == 0
        task.status = TaskStatus::READY
        ready_queue << task
      end
    end

    running_threads = []
    completed_tasks = []

    until ready_queue.empty? && running_threads.empty?
      # Sort ready queue by priority (highest first)
      ready_queue.sort_by! { |t| -t.priority }

      # Launch tasks up to concurrency limit
      while !ready_queue.empty? && running_threads.size < @max_concurrency
        task = ready_queue.shift
        next unless task.status == TaskStatus::READY

        puts "[START] #{task.name} (priority: #{task.priority})"

        thread = Thread.new(task) do |t|
          if t.timeout
            # Execute with timeout
            timed_out = false
            worker = Thread.new { t.execute }
            unless worker.join(t.timeout)
              worker.kill
              t.status = TaskStatus::CANCELLED
              timed_out = true
              puts "[TIMEOUT] #{t.name}"
            end
          else
            t.execute
          end

          @mutex.synchronize do
            case t.status
            when TaskStatus::COMPLETED
              puts "[DONE] #{t.name}"
              completed_tasks << t.id
            when TaskStatus::READY
              puts "[RETRY] #{t.name} - #{t.error_message}"
              ready_queue << t
            when TaskStatus::FAILED
              puts "[FAILED] #{t.name} - #{t.error_message}"
            end
          end
        end

        running_threads << thread
      end

      # Wait for at least one thread to finish
      running_threads.each(&:join)
      running_threads.clear

      # Process completed tasks and unlock dependents
      @mutex.synchronize do
        completed_tasks.each do |id|
          if @dependents[id]
            @dependents[id].each do |dep_id|
              @in_degree[dep_id] -= 1
              if @in_degree[dep_id] == 0
                @tasks[dep_id].status = TaskStatus::READY
                ready_queue << @tasks[dep_id]
              end
            end
          end
        end
        completed_tasks.clear
      end
    end

    puts "\n=== Execution Complete ==="
    print_summary
  end

  def print_summary
    puts "\nTask Summary:"
    @tasks.each do |_id, task|
      msg = "  #{task.name}: #{task.status}"
      msg += " (#{task.error_message})" if task.status == TaskStatus::FAILED
      puts msg
    end
  end
end
```

**Usage example:**

```ruby
scheduler = TaskScheduler.new(2)  # max 2 concurrent tasks

# Create tasks with dependencies
# Build pipeline: compile → link → test → deploy
compile = Task.new("compile", "Compile Source",
  work: -> { puts "  Compiling..."; sleep(0.1); true },
  priority: 10)

lint = Task.new("lint", "Run Linter",
  work: -> { puts "  Linting..."; sleep(0.05); true },
  priority: 8)

link_task = Task.new("link", "Link Objects",
  work: -> { puts "  Linking..."; sleep(0.08); true },
  priority: 7)
link_task.add_dependency("compile")

test = Task.new("test", "Run Tests",
  work: -> { puts "  Testing..."; sleep(0.12); true },
  priority: 5)
test.add_dependency("link")
test.add_dependency("lint")

deploy = Task.new("deploy", "Deploy",
  work: -> { puts "  Deploying..."; sleep(0.06); true },
  priority: 3, max_retries: 2)
deploy.add_dependency("test")

scheduler.add_task(compile)
scheduler.add_task(lint)
scheduler.add_task(link_task)
scheduler.add_task(test)
scheduler.add_task(deploy)

scheduler.execute

# Execution order respects dependencies:
# compile and lint run in parallel (no dependencies)
# link runs after compile
# test runs after link AND lint
# deploy runs after test
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
| Per-client state | `Hash` per client | Each client tracked independently |
| Thread safety | Mutex per limiter | Reduces contention for concurrent requests |
| Time source | `Process.clock_gettime(Process::CLOCK_MONOTONIC)` | Monotonic, not affected by system clock changes |

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

```ruby
RateLimitResult = Struct.new(:allowed, :remaining, :retry_after_ms) do
  def self.allow(remaining)
    new(true, remaining, 0)
  end

  def self.deny(retry_after_ms)
    new(false, 0, retry_after_ms)
  end
end

# Abstract rate limiter interface (Strategy pattern)
class RateLimiter
  def try_acquire(_client_id)
    raise NotImplementedError
  end

  def algorithm_name
    raise NotImplementedError
  end

  private

  # Monotonic time in seconds (float)
  def now
    Process.clock_gettime(Process::CLOCK_MONOTONIC)
  end
end
```

**Token Bucket Algorithm:**

```ruby
# Token Bucket: tokens are added at a fixed rate; each request consumes one token.
# Allows bursts up to bucket capacity, then limits to the refill rate.
class TokenBucketLimiter < RateLimiter
  def initialize(max_tokens, refill_rate)
    @max_tokens = max_tokens       # bucket capacity (max burst size)
    @refill_rate = refill_rate     # tokens per second
    @buckets = {}                  # client_id => { tokens:, last_refill: }
    @mutex = Mutex.new
  end

  def algorithm_name = "Token Bucket"

  def try_acquire(client_id)
    @mutex.synchronize do
      @buckets[client_id] ||= { tokens: @max_tokens.to_f, last_refill: now }
      bucket = @buckets[client_id]

      # Refill tokens
      elapsed = now - bucket[:last_refill]
      bucket[:tokens] = [@max_tokens.to_f, bucket[:tokens] + elapsed * @refill_rate].min
      bucket[:last_refill] = now

      if bucket[:tokens] >= 1.0
        bucket[:tokens] -= 1.0
        RateLimitResult.allow(bucket[:tokens].to_i)
      else
        # Calculate wait time for next token
        deficit = 1.0 - bucket[:tokens]
        wait_ms = (deficit / @refill_rate * 1000).to_i
        RateLimitResult.deny(wait_ms)
      end
    end
  end
end
```

**Fixed Window Counter:**

```ruby
# Fixed Window: divide time into fixed windows (e.g., 1 second).
# Count requests per window. Reset counter at window boundary.
class FixedWindowLimiter < RateLimiter
  def initialize(max_requests, window_seconds)
    @max_requests = max_requests
    @window_seconds = window_seconds
    @windows = {}  # client_id => { count:, window_start: }
    @mutex = Mutex.new
  end

  def algorithm_name = "Fixed Window"

  def try_acquire(client_id)
    @mutex.synchronize do
      current = now
      @windows[client_id] ||= { count: 0, window_start: current }
      window = @windows[client_id]

      # Check if we've moved to a new window
      if current - window[:window_start] >= @window_seconds
        window[:count] = 0
        window[:window_start] = current
      end

      if window[:count] < @max_requests
        window[:count] += 1
        RateLimitResult.allow(@max_requests - window[:count])
      else
        elapsed = current - window[:window_start]
        retry_ms = ((@window_seconds - elapsed) * 1000).to_i
        RateLimitResult.deny(retry_ms)
      end
    end
  end
end
```

**Sliding Window Log:**

```ruby
# Sliding Window Log: store timestamp of each request.
# Count requests in the last N seconds. Most accurate but uses more memory.
class SlidingWindowLogLimiter < RateLimiter
  def initialize(max_requests, window_seconds)
    @max_requests = max_requests
    @window_seconds = window_seconds
    @request_logs = {}  # client_id => Array of timestamps
    @mutex = Mutex.new
  end

  def algorithm_name = "Sliding Window Log"

  def try_acquire(client_id)
    @mutex.synchronize do
      current = now
      @request_logs[client_id] ||= []
      log = @request_logs[client_id]

      # Remove expired entries
      cutoff = current - @window_seconds
      log.shift while !log.empty? && log.first < cutoff

      if log.size < @max_requests
        log << current
        RateLimitResult.allow(@max_requests - log.size)
      else
        # Calculate when the oldest request will expire
        oldest = log.first
        retry_ms = ((oldest + @window_seconds - current) * 1000).to_i
        RateLimitResult.deny(retry_ms)
      end
    end
  end
end
```

**Sliding Window Counter (approximate but efficient):**

```ruby
# Sliding Window Counter: combines fixed window simplicity with sliding window accuracy.
# Uses weighted average of current and previous window counts.
class SlidingWindowCounterLimiter < RateLimiter
  def initialize(max_requests, window_seconds)
    @max_requests = max_requests
    @window_seconds = window_seconds
    @states = {}  # client_id => { current_count:, previous_count:, window_start: }
    @mutex = Mutex.new
  end

  def algorithm_name = "Sliding Window Counter"

  def try_acquire(client_id)
    @mutex.synchronize do
      current = now
      @states[client_id] ||= { current_count: 0, previous_count: 0, window_start: current }
      state = @states[client_id]

      elapsed = current - state[:window_start]

      if elapsed >= @window_seconds * 2
        # Skipped a full window
        state[:previous_count] = 0
        state[:current_count] = 0
        state[:window_start] = current
      elsif elapsed >= @window_seconds
        # Moved to next window
        state[:previous_count] = state[:current_count]
        state[:current_count] = 0
        state[:window_start] += @window_seconds
      end

      # Calculate weighted count
      current_elapsed = current - state[:window_start]
      weight = 1.0 - current_elapsed / @window_seconds
      estimated = state[:previous_count] * weight + state[:current_count]

      if estimated < @max_requests
        state[:current_count] += 1
        RateLimitResult.allow(@max_requests - estimated.to_i - 1)
      else
        retry_ms = (@window_seconds * weight * 1000).to_i
        RateLimitResult.deny(retry_ms)
      end
    end
  end
end
```

**Usage example:**

```ruby
def test_limiter(limiter, client_id, requests)
  puts "\n--- Testing #{limiter.algorithm_name} (#{requests} requests) ---"

  allowed = 0
  denied = 0
  first_denial_shown = false

  requests.times do |i|
    result = limiter.try_acquire(client_id)
    if result.allowed
      allowed += 1
    else
      denied += 1
      unless first_denial_shown
        puts "First denial at request #{i + 1}," \
             " retry after #{result.retry_after_ms}ms"
        first_denial_shown = true
      end
    end
  end
  puts "Allowed: #{allowed}, Denied: #{denied}"
end

# Token Bucket: 5 tokens max, refill 2 per second
token_bucket = TokenBucketLimiter.new(5, 2.0)
test_limiter(token_bucket, "user1", 10)

# Fixed Window: 5 requests per 1 second
fixed_window = FixedWindowLimiter.new(5, 1.0)
test_limiter(fixed_window, "user1", 10)

# Sliding Window Log: 5 requests per 1 second
sliding_log = SlidingWindowLogLimiter.new(5, 1.0)
test_limiter(sliding_log, "user1", 10)

# Sliding Window Counter: 5 requests per 1 second
sliding_counter = SlidingWindowCounterLimiter.new(5, 1.0)
test_limiter(sliding_counter, "user1", 10)

# Wait and try again (tokens should refill)
puts "\n--- After 1 second wait ---"
sleep(1)
result = token_bucket.try_acquire("user1")
puts "Token Bucket: #{result.allowed ? 'ALLOWED' : 'DENIED'}," \
     " remaining: #{result.remaining}"
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

```ruby
require 'set'

class CacheEntry
  attr_reader :value, :created_at, :expires_at
  attr_accessor :access_count

  def initialize(value, ttl_seconds)
    @value = value
    @created_at = monotonic_now
    @expires_at = @created_at + ttl_seconds
    @access_count = 0
  end

  def expired?
    monotonic_now >= @expires_at
  end

  def touch
    @access_count += 1
  end

  private

  def monotonic_now
    Process.clock_gettime(Process::CLOCK_MONOTONIC)
  end
end
```

**Eviction Strategy interface:**

```ruby
# ─── Eviction Strategies ─────────────────────────────────

class EvictionStrategy
  def on_access(_key) = raise(NotImplementedError)
  def on_insert(_key) = raise(NotImplementedError)
  def on_remove(_key) = raise(NotImplementedError)
  def evict           = raise(NotImplementedError)  # returns key to evict
  def name            = raise(NotImplementedError)
end

# LRU Eviction — evict least recently used
class LRUEviction < EvictionStrategy
  def initialize
    @order = []       # front = most recent, back = least recent
    @positions = {}   # key => index (for fast lookup)
  end

  def name = "LRU"

  def on_access(key)
    return unless @positions.key?(key)
    @order.delete(key)
    @order.unshift(key)
    rebuild_positions
  end

  def on_insert(key)
    @order.unshift(key)
    rebuild_positions
  end

  def on_remove(key)
    @order.delete(key)
    @positions.delete(key)
    rebuild_positions
  end

  def evict
    raise "Nothing to evict" if @order.empty?
    key = @order.pop
    @positions.delete(key)
    key
  end

  private

  def rebuild_positions
    @positions.clear
    @order.each_with_index { |k, i| @positions[k] = i }
  end
end

# LFU Eviction — evict least frequently used
class LFUEviction < EvictionStrategy
  def initialize
    @frequency = {}    # key => count
    @insertion_order = [] # for tie-breaking
  end

  def name = "LFU"

  def on_access(key)
    @frequency[key] = (@frequency[key] || 0) + 1
  end

  def on_insert(key)
    @frequency[key] = 1
    @insertion_order << key
  end

  def on_remove(key)
    @frequency.delete(key)
    @insertion_order.delete(key)
  end

  def evict
    raise "Nothing to evict" if @frequency.empty?

    min_freq = @frequency.values.min
    # Among keys with min frequency, evict the oldest inserted
    key = @insertion_order.find { |k| @frequency[k] == min_freq }
    @frequency.delete(key)
    @insertion_order.delete(key)
    key
  end
end

# FIFO Eviction — evict oldest inserted
class FIFOEviction < EvictionStrategy
  def initialize
    @order = []
  end

  def name = "FIFO"

  def on_access(_key)
    # FIFO doesn't change order on access
  end

  def on_insert(key)
    @order << key
  end

  def on_remove(key)
    @order.delete(key)
  end

  def evict
    raise "Nothing to evict" if @order.empty?
    @order.shift
  end
end
```

**Cache Layer (single level):**

```ruby
# ─── Cache Metrics ───────────────────────────────────────

class CacheMetrics
  attr_accessor :hits, :misses, :evictions

  def initialize
    @hits = 0
    @misses = 0
    @evictions = 0
  end

  def hit_rate
    total = @hits + @misses
    total > 0 ? (@hits.to_f / total * 100.0) : 0.0
  end

  def reset
    @hits = @misses = @evictions = 0
  end
end

# ─── Abstract Cache Layer (Chain of Responsibility) ──────

class CacheLayer
  def get(_key)         = raise(NotImplementedError)
  def put(_key, _value, _ttl = nil) = raise(NotImplementedError)
  def remove(_key)      = raise(NotImplementedError)
  def clear             = raise(NotImplementedError)
  def cache_name        = raise(NotImplementedError)
  def metrics           = raise(NotImplementedError)
end

# ─── In-Memory Cache Layer ───────────────────────────────

class InMemoryCache < CacheLayer
  attr_reader :metrics

  def initialize(name, capacity, default_ttl_seconds, eviction)
    @name = name
    @capacity = capacity
    @default_ttl = default_ttl_seconds
    @store = {}          # key => CacheEntry
    @eviction = eviction
    @metrics = CacheMetrics.new
    @mutex = Mutex.new
  end

  def cache_name = @name

  def get(key)
    @mutex.synchronize do
      entry = @store[key]
      unless entry
        @metrics.misses += 1
        return nil
      end

      # Check TTL
      if entry.expired?
        @eviction.on_remove(key)
        @store.delete(key)
        @metrics.misses += 1
        return nil
      end

      entry.touch
      @eviction.on_access(key)
      @metrics.hits += 1
      entry.value
    end
  end

  def put(key, value, ttl = nil)
    @mutex.synchronize do
      actual_ttl = ttl || @default_ttl

      if @store.key?(key)
        # Update existing
        @store[key] = CacheEntry.new(value, actual_ttl)
        @eviction.on_access(key)
      else
        # Evict if at capacity
        while @store.size >= @capacity
          evicted_key = @eviction.evict
          @store.delete(evicted_key)
          @metrics.evictions += 1
        end

        @store[key] = CacheEntry.new(value, actual_ttl)
        @eviction.on_insert(key)
      end
    end
  end

  def remove(key)
    @mutex.synchronize do
      return false unless @store.key?(key)
      @eviction.on_remove(key)
      @store.delete(key)
      true
    end
  end

  def clear
    @mutex.synchronize { @store.clear }
  end
end
```

**Multi-Level Cache (Chain of Responsibility):**

```ruby
# ─── Write Policy ────────────────────────────────────────

module WritePolicy
  WRITE_THROUGH = :write_through  # write to all levels immediately
  WRITE_BACK    = :write_back     # write to L1 only, sync to L2 later
  CACHE_ASIDE   = :cache_aside    # caller manages cache and data source separately
end

# ─── Multi-Level Cache ───────────────────────────────────

class MultiLevelCache
  def initialize(policy, data_source = nil)
    @levels = []
    @write_policy = policy
    @data_source = data_source  # Proc that takes a key, returns value or nil
  end

  def add_level(level)
    @levels << level
  end

  # Get: check each level in order; on hit, populate higher levels
  def get(key)
    # Check each cache level
    @levels.each_with_index do |level, i|
      value = level.get(key)
      if value
        # Cache hit at level i — populate all higher levels (L1, L2, ...)
        (0...i).each { |j| @levels[j].put(key, value) }
        return value
      end
    end

    # Cache miss at all levels — fetch from data source
    if @data_source
      value = @data_source.call(key)
      if value
        # Populate all cache levels
        @levels.each { |level| level.put(key, value) }
        return value
      end
    end

    nil
  end

  # Put: write according to policy
  def put(key, value, ttl = nil)
    case @write_policy
    when WritePolicy::WRITE_THROUGH
      # Write to all levels
      @levels.each { |level| level.put(key, value, ttl) }
    when WritePolicy::WRITE_BACK
      # Write to L1 only (sync to L2 happens asynchronously)
      @levels.first&.put(key, value, ttl)
    when WritePolicy::CACHE_ASIDE
      # Write to all levels (caller also writes to data source)
      @levels.each { |level| level.put(key, value, ttl) }
    end
  end

  # Remove from all levels (invalidation)
  def invalidate(key)
    @levels.each { |level| level.remove(key) }
  end

  # Clear all levels
  def clear
    @levels.each(&:clear)
  end

  # Print metrics for all levels
  def print_metrics
    puts "\n=== Cache Metrics ==="
    @levels.each do |level|
      m = level.metrics
      puts "#{level.cache_name}:" \
           " hits=#{m.hits}" \
           " misses=#{m.misses}" \
           " evictions=#{m.evictions}" \
           " hit_rate=#{'%.1f' % m.hit_rate}%"
    end
  end
end
```

**Usage example:**

```ruby
# Simulated database
database = {
  "user:1" => "Alice",
  "user:2" => "Bob",
  "user:3" => "Charlie",
  "user:4" => "Diana",
  "user:5" => "Eve"
}

db_lookup = ->(key) {
  puts "  [DB] Fetching #{key}"
  database[key]
}

# Create multi-level cache
cache = MultiLevelCache.new(WritePolicy::WRITE_THROUGH, db_lookup)

# L1: Small, fast, LRU, 5-second TTL
cache.add_level(InMemoryCache.new(
  "L1 (In-Memory)", 3, 5.0, LRUEviction.new))

# L2: Larger, LFU, 30-second TTL
cache.add_level(InMemoryCache.new(
  "L2 (Distributed)", 10, 30.0, LFUEviction.new))

# First access — cache miss, fetches from DB
puts "--- First access ---"
val = cache.get("user:1")
puts "user:1 = #{val || 'NOT FOUND'}"
# Output: [DB] Fetching user:1
#         user:1 = Alice

# Second access — cache hit at L1
puts "\n--- Second access (cached) ---"
val = cache.get("user:1")
puts "user:1 = #{val || 'NOT FOUND'}"
# No DB fetch — served from L1

# Access more keys to fill L1 (capacity 3)
puts "\n--- Filling L1 cache ---"
cache.get("user:2")
cache.get("user:3")
cache.get("user:4")  # L1 is full — evicts LRU (user:1)

# user:1 evicted from L1, but still in L2
puts "\n--- After L1 eviction ---"
val = cache.get("user:1")
puts "user:1 = #{val || 'NOT FOUND'}"
# L1 miss, L2 hit — no DB fetch, repopulates L1

# Write-through: update propagates to all levels
puts "\n--- Write-through update ---"
cache.put("user:1", "Alice (Updated)")
val = cache.get("user:1")
puts "user:1 = #{val || 'NOT FOUND'}"
# Output: user:1 = Alice (Updated)

# Invalidation
puts "\n--- Invalidation ---"
cache.invalidate("user:1")
val = cache.get("user:1")
puts "user:1 = #{val || 'NOT FOUND'}"
# Fetches from DB again (original value, not updated)

cache.print_metrics
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
| In-Memory File System | Tree structure, path resolution, permissions | Tree (Composite), Hash | Composite, Iterator, Visitor, Facade |
| Spreadsheet | Dependency graph, circular detection, recalculation | DAG, AST, Hash | Observer, Composite, Memento |
| Text Editor | Undo/redo, cursor management, clipboard | String buffer, command stack | Command, Memento, Singleton |
| Card Game Framework | Extensible game loop, multiple game types | Deck, Hand, Player | Template Method, Strategy, Factory |
| LRU Cache | O(1) get/put with eviction | Hash + Doubly Linked List | — (pure data structure) |
| Pub-Sub Messaging | Decoupled communication, message ordering | Topic queues, subscriber map | Observer, Mediator, Strategy |
| Task Scheduler | DAG dependencies, priority, concurrency | DAG, Priority Queue, Thread Pool | Strategy, Observer, Command |
| Rate Limiter | Algorithm comparison, time-based tracking | Token bucket, sliding window | Strategy, Proxy, Singleton |
| Multi-Level Cache | Layered caching, eviction, write policies | Multiple cache layers, eviction structures | Chain of Responsibility, Strategy, Proxy |

---

## Interview Tips for Module 14

1. **"Design a Chess Game."** Start with the class hierarchy: abstract `Piece` base with `can_move?`, concrete pieces (King, Queen, Rook, Bishop, Knight, Pawn). Board is an 8×8 grid. Move validation is two-phase: piece-specific rules + global check (does the move leave own king in check?). Handle special moves (castling, en passant, pawn promotion) in the `Game` class, not in individual pieces. Use the Command pattern for move history. Checkmate detection: king is in check AND no legal move can escape it.

2. **"Design an In-Memory File System."** Use the Composite pattern: `FSEntry` base class, `FSFile` (leaf) and `FSDirectory` (composite). Directories store children in a `Hash` for O(1) lookup. Path resolution: split by `/`, traverse the tree. Support `mkdirp` (recursive directory creation). Add permissions as a bitmask (rwx). The `FileSystem` class is a Facade that provides path-based operations.

3. **"Design a Spreadsheet."** The hard part is the dependency graph. Each cell with a formula depends on other cells. Use a DAG to track dependencies. When a cell changes, recalculate all dependents in topological order (BFS from the changed cell). Detect circular dependencies using DFS with coloring (white/gray/black — a gray node visited again means a cycle). Parse formulas with a recursive descent parser into an AST.

4. **"Design a Text Editor with Undo/Redo."** Use the Command pattern: each operation (insert, delete, replace) is a command object with `execute` and `undo`. Maintain two stacks: undo stack and redo stack. On new action: push to undo stack, clear redo stack. On undo: pop from undo stack, call `undo`, push to redo stack. On redo: pop from redo stack, call `execute`, push to undo stack. Each command stores the data needed to reverse itself (deleted text, old text for replace).

5. **"Design a Card Game Framework."** Use the Template Method pattern: `CardGame#play` defines the game loop skeleton (initialize → shuffle → deal → play rounds → announce winner). Subclasses override hook methods (`deal_cards`, `play_turn`, `check_win_condition`, `announce_winner`). To add a new game, create a new subclass — the framework code doesn't change. Use Factory for deck creation (standard 52-card vs UNO deck).

6. **"Design an LRU Cache."** Hash + Doubly Linked List. Hash maps key → list node reference (O(1) lookup). Doubly linked list maintains access order (most recent at head, least recent at tail). On `get`: move node to head. On `put`: if key exists, update and move to head; if new, add to head and evict from tail if at capacity. Use sentinel nodes (dummy head/tail) to eliminate nil checks. For thread safety, wrap with `Mutex` and `synchronize`.

7. **"Design a Pub-Sub System."** Three main components: `MessageBroker` (facade), `Topic` (message channel), `Subscriber` (message consumer). Publishers send messages to topics via the broker. Topics maintain a message log and a list of subscribers. Each subscriber has an independent offset for acknowledgment. Use the Observer pattern for message delivery. Support message filtering with lambdas/procs. Thread safety: mutex per topic to reduce contention.

8. **"Design a Task Scheduler."** Model tasks as nodes in a DAG where edges represent dependencies. Use Kahn's algorithm (BFS-based topological sort) to determine execution order and detect cycles. Maintain an in-degree count per task; when a task completes, decrement in-degree of its dependents. Tasks with in-degree 0 are ready to execute. Use a priority sort for ready tasks. Execute independent tasks in parallel using threads. Support retry with configurable max attempts.

9. **"Design a Rate Limiter."** Know the four main algorithms: (a) **Token Bucket** — tokens refill at a fixed rate, each request consumes one token; allows bursts up to bucket capacity. (b) **Fixed Window** — count requests per time window; simple but allows 2x burst at window boundaries. (c) **Sliding Window Log** — store timestamps of all requests; most accurate but high memory. (d) **Sliding Window Counter** — weighted average of current and previous window; good balance of accuracy and efficiency. Use the Strategy pattern to make algorithms interchangeable.

10. **"Design a Multi-Level Cache."** Use Chain of Responsibility: request flows through L1 → L2 → data source. On cache hit at level N, populate all higher levels (L1 through L(N-1)). Use the Strategy pattern for eviction policies (LRU, LFU, FIFO) — different levels can use different strategies. Support write policies: write-through (write to all levels), write-back (write to L1, sync later), cache-aside (caller manages). Track metrics (hit rate, miss rate, evictions) per level for monitoring.

11. **"How do you handle concurrency in these systems?"** Use `Mutex` with `synchronize` for thread safety. Prefer fine-grained locking (per-object or per-topic) over a single global lock to reduce contention. Ruby's `MonitorMixin` provides reentrant locking when needed. For read-heavy workloads, consider `Concurrent::ReadWriteLock` from the `concurrent-ruby` gem. Ruby's GIL (Global Interpreter Lock) prevents true parallel execution of Ruby code in MRI, but `Mutex` is still needed for correctness when threads interleave at I/O boundaries or between Ruby instructions.

12. **"What design patterns appear most frequently in hard LLD problems?"** (a) **Strategy** — appears in almost every problem (scoring, eviction, rate limiting, scheduling). (b) **Observer** — for reactive updates (spreadsheet recalculation, pub-sub, game state). (c) **Command** — for undo/redo and task encapsulation. (d) **Composite** — for tree structures (file system, expression AST). (e) **Chain of Responsibility** — for layered processing (multi-level cache, middleware). (f) **Template Method** — for extensible frameworks (card game, game loop). (g) **Facade** — for simplifying complex subsystems.

### Ruby-Specific Notes

- **Duck typing over inheritance**: Ruby's duck typing means you don't always need abstract base classes. Any object that responds to the right methods works. However, for LLD interviews, explicit base classes with `raise NotImplementedError` document the contract clearly.
- **Blocks, Procs, and Lambdas**: Ruby's first-class closures replace the need for single-method interfaces. The Pub-Sub subscriber filter and Task scheduler work functions use lambdas instead of separate strategy classes.
- **Symbols as enums**: Ruby idiomatically uses symbols (`:white`, `:active`) grouped in modules with constants, rather than C++ enum classes.
- **MonitorMixin vs Mutex**: `MonitorMixin` provides reentrant locking (a thread can re-enter a synchronized block it already holds), which is useful when public methods call other public methods. Plain `Mutex` is simpler but will deadlock on re-entry.
- **No destructors**: Ruby uses garbage collection, so there's no need for explicit memory management. The LRU cache doesn't need a destructor to clean up nodes — the GC handles it.
- **`Struct` for value objects**: Ruby's `Struct` is ideal for simple data containers like `RateLimitResult`, `Move`, and `Position`. It auto-generates `==`, `hash`, and accessors.