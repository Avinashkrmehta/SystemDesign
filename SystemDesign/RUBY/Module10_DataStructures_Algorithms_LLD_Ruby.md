# Module 10: Data Structures & Algorithms for LLD in Ruby

> In Low-Level Design, choosing the right data structure is as important as choosing the right design pattern. Data structures determine the time and space complexity of your operations, and algorithms built on top of them power features like caching, routing, rate limiting, and search. This module covers the essential data structures and algorithms you need to know for LLD interviews and real-world system component design in Ruby.

---

## 10.1 Essential Data Structures

> Every LLD problem maps to a set of operations — insert, delete, search, update, iterate. The data structure you choose determines how efficiently each operation runs. Ruby provides rich built-in data structures through its standard library, and its dynamic nature makes implementing custom structures concise and expressive. This section covers the core data structures, their Ruby implementations, time complexities, and when to use each in LLD contexts.

---

### Arrays

An **Array** is Ruby's fundamental ordered, integer-indexed collection. It is dynamically sized, can hold objects of any type, and provides a rich set of built-in methods. Ruby arrays are the equivalent of C++ `std::vector`.

**Array — dynamic array:**

```ruby
# Creation
a1 = []                        # empty array
a2 = Array.new(10, 0)          # 10 elements, all 0
a3 = [1, 2, 3, 4, 5]           # literal
a4 = Array.new(5) { |i| i * 2 } # [0, 2, 4, 6, 8] — block initialization

# Adding elements
a1.push(10)        # O(1) amortized — adds to the end (same as <<)
a1 << 20           # O(1) amortized — shovel operator
a1.unshift(5)      # O(n) — adds to the front (shifts all elements)
a1.insert(2, 99)   # O(n) — insert 99 at index 2

# Accessing elements
first = a3[0]       # O(1) — index access
safe  = a3.fetch(0) # O(1) — raises IndexError if invalid
last  = a3.last     # O(1) — last element (or a3[-1])
front = a3.first    # O(1) — first element

# Size
puts "Size: #{a3.size}"    # 5 (same as .length)
puts "Empty: #{a1.empty?}" # false

# Removing elements
a3.pop              # O(1) — remove and return last element
a3.shift            # O(n) — remove and return first element
a3.delete_at(1)     # O(n) — remove element at index 1
a3.delete(3)        # O(n) — remove ALL elements equal to 3

# Iterating
a3.each { |x| print "#{x} " }
puts

a3.each_with_index { |x, i| puts "#{i}: #{x}" }

# Mapping and filtering
doubled = a3.map { |x| x * 2 }
evens   = a3.select { |x| x.even? }
odds    = a3.reject { |x| x.even? }

# Sorting
a3.sort                          # ascending (returns new array)
a3.sort { |a, b| b <=> a }      # descending
a3.sort!                         # in-place sort
a3.sort_by { |x| -x }           # sort by transformed value (descending)

# Searching
idx = a3.index(3)                # O(n) — first index of value 3, or nil
found = a3.include?(3)           # O(n) — true/false
item = a3.find { |x| x > 2 }    # O(n) — first element matching condition

# Binary search (requires sorted array)
a3.sort!
idx = a3.bsearch_index { |x| x >= 3 }  # O(log n) — index of first element >= 3
val = a3.bsearch { |x| x >= 3 }         # O(log n) — value of first element >= 3
```

**How Ruby array resizing works internally:**

```
Ruby arrays (implemented in C as RArray) use a similar strategy to C++ vectors:

Initial:  [1, 2, 3] capacity=3, size=3

push(4):
  1. capacity == size → need to grow
  2. Allocate new internal buffer (typically 1.5x or 2x)
  3. Copy all elements to new buffer
  4. Add new element
  5. Free old buffer

Result:   [1, 2, 3, 4, _, _] capacity=6, size=4

This is why push/append is O(1) AMORTIZED — most calls are O(1),
but occasionally O(n) when reallocation happens.
```

**Array time complexity:**

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `push` / `<<` | O(1) amortized | O(n) when reallocation needed |
| `pop` | O(1) | |
| `[]` / `fetch` | O(1) | Random access |
| `unshift` | O(n) | Shifts all elements |
| `shift` | O(n) | Shifts all elements |
| `insert` (middle) | O(n) | Shifts elements |
| `delete_at` (middle) | O(n) | Shifts elements |
| `include?` / `index` | O(n) | Linear search |
| `bsearch` | O(log n) | Requires sorted array |
| `sort` | O(n log n) | |

**When to use arrays in LLD:**
- Default container for ordered collections (product lists, user lists)
- When you need random access by index (leaderboard positions)
- When most operations are at the end (stack-like behavior with `push`/`pop`)
- When you need rich enumeration methods (`map`, `select`, `reduce`)

---

### Linked Lists

Ruby does not have a built-in linked list class. However, linked lists are essential for LLD problems like LRU Cache where you need O(1) insertion/deletion at any position given a reference to the node. You implement them with plain Ruby classes.

**Custom doubly linked list:**

```ruby
class LinkedList
  class Node
    attr_accessor :data, :prev, :next_node

    def initialize(data)
      @data = data
      @prev = nil
      @next_node = nil
    end
  end

  attr_reader :size

  def initialize
    @head = nil
    @tail = nil
    @size = 0
  end

  # O(1) — push to front
  def push_front(data)
    node = Node.new(data)
    node.next_node = @head
    @head.prev = node if @head
    @head = node
    @tail = node unless @tail
    @size += 1
    node
  end

  # O(1) — push to back
  def push_back(data)
    node = Node.new(data)
    node.prev = @tail
    @tail.next_node = node if @tail
    @tail = node
    @head = node unless @head
    @size += 1
    node
  end

  # O(1) — remove a specific node (given reference)
  def remove(node)
    return unless node

    if node.prev
      node.prev.next_node = node.next_node
    else
      @head = node.next_node  # removing head
    end

    if node.next_node
      node.next_node.prev = node.prev
    else
      @tail = node.prev  # removing tail
    end

    @size -= 1
    node.data
  end

  # O(1) — move a node to the front (used in LRU Cache!)
  def move_to_front(node)
    return if node == @head  # already at front

    # Detach from current position
    node.prev.next_node = node.next_node if node.prev
    node.next_node.prev = node.prev if node.next_node
    @tail = node.prev if node == @tail

    # Attach at front
    node.prev = nil
    node.next_node = @head
    @head.prev = node if @head
    @head = node
  end

  def head
    @head
  end

  def tail
    @tail
  end

  def empty?
    @size == 0
  end

  def to_a
    result = []
    current = @head
    while current
      result << current.data
      current = current.next_node
    end
    result
  end
end

# Usage
list = LinkedList.new
n1 = list.push_back(10)
n2 = list.push_back(20)
n3 = list.push_back(30)
puts list.to_a.inspect  # [10, 20, 30]

list.move_to_front(n3)
puts list.to_a.inspect  # [30, 10, 20]

list.remove(n2)
puts list.to_a.inspect  # [30, 10]
```

**Linked list time complexity:**

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `push_front` / `push_back` | O(1) | |
| Remove node (with reference) | O(1) | Key advantage over array |
| Move to front (with reference) | O(1) | Essential for LRU Cache |
| Search by value | O(n) | No random access |
| Access by index | O(n) | Must traverse |

**When to use linked lists in LLD:**
- **LRU Cache** — O(1) move-to-front and remove-from-back
- When you need frequent insertions/deletions in the middle
- Implementing queues, deques, or custom ordered structures
- When you need stable references (inserting/deleting doesn't invalidate other node references)

---


### Stacks, Queues, and Deques

These are **abstract data types** that restrict how you interact with the underlying data to enforce specific access patterns. Ruby doesn't have dedicated stack or queue classes — you use `Array` with discipline, or build your own.

**Stack — LIFO (Last In, First Out) using Array:**

```ruby
# Ruby arrays naturally support stack operations
stack = []

# Push elements — O(1) amortized
stack.push(10)
stack.push(20)
stack.push(30)

# Peek at top element — O(1)
puts "Top: #{stack.last}"  # 30

# Pop (remove top) — O(1)
stack.pop
puts "Top after pop: #{stack.last}"  # 20

# Size and empty check
puts "Size: #{stack.size}"
puts "Empty: #{stack.empty?}"

# Process all elements
until stack.empty?
  print "#{stack.pop} "
end
puts
```

**LLD use cases for stack:**
- **Undo/Redo** in text editors (Command pattern + stack)
- **Expression evaluation** (postfix, infix to postfix conversion)
- **Bracket matching** (code editors, compilers)
- **DFS traversal** (iterative depth-first search)
- **Call stack simulation** (function call tracking)
- **Browser back button** (navigation history)

```ruby
# Example: Undo system using stack
class UndoManager
  def initialize
    @undo_stack = []
    @redo_stack = []
  end

  def execute(action)
    @undo_stack.push(action)
    @redo_stack.clear  # clear redo stack when a new action is performed
    puts "Executed: #{action}"
  end

  def undo
    if @undo_stack.empty?
      puts "Nothing to undo"
      return
    end
    action = @undo_stack.pop
    @redo_stack.push(action)
    puts "Undone: #{action}"
  end

  def redo
    if @redo_stack.empty?
      puts "Nothing to redo"
      return
    end
    action = @redo_stack.pop
    @undo_stack.push(action)
    puts "Redone: #{action}"
  end
end

manager = UndoManager.new
manager.execute("Type 'Hello'")
manager.execute("Type ' World'")
manager.undo   # Undone: Type ' World'
manager.redo   # Redone: Type ' World'
```

**Queue — FIFO (First In, First Out) using Array:**

```ruby
# Ruby arrays can act as queues, but shift is O(n)
queue = []

# Enqueue — O(1) amortized
queue.push(10)
queue.push(20)
queue.push(30)

# Peek at front and back — O(1)
puts "Front: #{queue.first}"  # 10
puts "Back: #{queue.last}"    # 30

# Dequeue (remove front) — O(n) with Array#shift!
queue.shift
puts "Front after shift: #{queue.first}"  # 20

# Process all elements
until queue.empty?
  print "#{queue.shift} "
end
puts
```

**Efficient Queue using Thread::Queue (thread-safe) or custom implementation:**

```ruby
# Thread::Queue — built-in thread-safe FIFO queue
# Great for producer-consumer patterns
require 'thread'

q = Queue.new

# Enqueue — O(1)
q.push(10)
q.push(20)
q.push(30)

# Dequeue — O(1), blocks if empty
puts q.pop  # 10
puts q.pop  # 20

puts "Size: #{q.size}"
puts "Empty: #{q.empty?}"
```

```ruby
# Custom efficient queue using two stacks (amortized O(1) dequeue)
class EfficientQueue
  def initialize
    @in_stack = []
    @out_stack = []
  end

  # O(1) amortized
  def enqueue(item)
    @in_stack.push(item)
  end

  # O(1) amortized
  def dequeue
    transfer if @out_stack.empty?
    @out_stack.pop
  end

  # O(1) amortized
  def front
    transfer if @out_stack.empty?
    @out_stack.last
  end

  def size
    @in_stack.size + @out_stack.size
  end

  def empty?
    @in_stack.empty? && @out_stack.empty?
  end

  private

  def transfer
    @out_stack.push(@in_stack.pop) until @in_stack.empty?
  end
end
```

**LLD use cases for queue:**
- **Task scheduling** (FCFS — first come, first served)
- **BFS traversal** (breadth-first search in graphs/trees)
- **Message queues** (producer-consumer pattern)
- **Print job queue** (printer spooler)
- **Request handling** (web server request queue)
- **Order processing** (e-commerce order queue)

```ruby
# Example: Simple task scheduler using queue
Task = Struct.new(:id, :description, :priority)

class TaskScheduler
  def initialize
    @task_queue = []
  end

  def submit(task)
    @task_queue.push(task)
    puts "Task #{task.id} submitted: #{task.description}"
  end

  def process_next
    if @task_queue.empty?
      puts "No tasks to process"
      return
    end
    task = @task_queue.shift
    puts "Processing task #{task.id}: #{task.description}"
  end

  def pending
    @task_queue.size
  end
end

scheduler = TaskScheduler.new
scheduler.submit(Task.new(1, "Send email", 3))
scheduler.submit(Task.new(2, "Generate report", 1))
scheduler.process_next  # Processing task 1: Send email
```

**Deque — Double-Ended Queue:**

Ruby's `Array` already supports efficient operations at both ends (except `shift`/`unshift` which are O(n)). For a true O(1) deque, you'd use a custom implementation or accept the O(n) cost for front operations on small datasets.

```ruby
# Ruby Array as a deque (simple approach)
dq = [2, 3, 4]

# Operations at both ends
dq.unshift(1)   # [1, 2, 3, 4] — O(n)
dq.push(5)      # [1, 2, 3, 4, 5] — O(1)
dq.shift         # [2, 3, 4, 5] — O(n)
dq.pop           # [2, 3, 4] — O(1)

# Random access — O(1)
puts "Element at index 1: #{dq[1]}"  # 3

dq.each { |x| print "#{x} " }
puts
```

**LLD use cases for deque:**
- **Sliding window** algorithms (add to back, remove from front)
- **Work stealing** queues (owner pops from back, thieves steal from front)
- **Palindrome checking** (compare front and back)
- **Undo with limited history** (push back, shift front when full)

**Comparison:**

| Feature | Stack (Array) | Queue (Array) | Queue (Thread::Queue) |
|---------|--------------|---------------|----------------------|
| Push back | O(1) | O(1) | O(1) |
| Pop back | O(1) | ❌ | ❌ |
| Pop front | ❌ | O(n)* | O(1) |
| Random access | O(1) | O(1) | ❌ |
| Thread-safe | ❌ | ❌ | ✅ |
| Access pattern | LIFO | FIFO | FIFO |

\* `Array#shift` is O(n); use `Thread::Queue` or a custom two-stack queue for O(1) amortized dequeue.

---

### Hashes and Sets

**Hash-based containers** provide average O(1) lookup, insertion, and deletion by mapping keys to array indices using a hash function. Ruby's `Hash` and `Set` are the most frequently used data structures in LLD.

**Hash — Ruby's hash map:**

```ruby
# Creation
scores = {}
scores = Hash.new(0)  # default value 0 for missing keys

# Insert — O(1) average
scores["Alice"] = 95
scores["Bob"] = 87
scores.store("Charlie", 92)

# Lookup — O(1) average
puts "Alice's score: #{scores["Alice"]}"

# Safe lookup (returns nil or default if key missing)
puts scores["Eve"]           # nil (or default value)
puts scores.fetch("Eve", 0)  # 0 (explicit default)
# scores.fetch("Eve")        # raises KeyError if no default

# Check existence — O(1) average
puts scores.key?("Bob")      # true (also: has_key?, include?)
puts scores.value?(87)       # true (O(n) — scans all values)

# Delete — O(1) average
scores.delete("Charlie")

# Iterate (insertion order is preserved in Ruby >= 1.9)
scores.each { |name, score| puts "#{name}: #{score}" }

# Keys and values
puts scores.keys.inspect    # ["Alice", "Bob"]
puts scores.values.inspect  # [95, 87]

# Merge hashes
extra = { "Diana" => 88, "Eve" => 91 }
all_scores = scores.merge(extra)  # returns new hash
scores.merge!(extra)              # modifies in place

# Transform
doubled = scores.transform_values { |v| v * 2 }
upcased = scores.transform_keys(&:upcase)

# Counting pattern (very common in LLD)
words = %w[apple banana apple cherry banana apple]
freq = Hash.new(0)
words.each { |w| freq[w] += 1 }
puts freq.inspect  # {"apple"=>3, "banana"=>2, "cherry"=>1}

# Group by pattern
people = [
  { name: "Alice", dept: "Engineering" },
  { name: "Bob", dept: "Marketing" },
  { name: "Charlie", dept: "Engineering" }
]
by_dept = people.group_by { |p| p[:dept] }
puts by_dept.inspect
```

**Set — Ruby's hash set:**

```ruby
require 'set'

tags = Set.new

# Insert — O(1) average
tags.add("ruby")
tags.add("design-patterns")
tags.add("lld")
tags.add("ruby")  # duplicate — ignored

# Lookup — O(1) average
puts tags.include?("ruby")  # true

# Delete — O(1) average
tags.delete("lld")

# Set operations
set_a = Set.new([1, 2, 3, 4])
set_b = Set.new([3, 4, 5, 6])

puts (set_a & set_b).inspect   # intersection: #<Set: {3, 4}>
puts (set_a | set_b).inspect   # union: #<Set: {1, 2, 3, 4, 5, 6}>
puts (set_a - set_b).inspect   # difference: #<Set: {1, 2}>
puts (set_a ^ set_b).inspect   # symmetric difference: #<Set: {1, 2, 5, 6}>
puts set_a.subset?(set_b)      # false
puts set_a.superset?(set_b)    # false

# Iterate
tags.each { |tag| print "#{tag} " }
puts
```

**Custom hash function (for user-defined types as hash keys):**

```ruby
# To use a custom object as a Hash key, define hash and eql?
class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
  end

  def eql?(other)
    other.is_a?(Point) && @x == other.x && @y == other.y
  end

  def hash
    [@x, @y].hash  # Ruby's Array#hash combines element hashes
  end
end

# Usage
point_labels = {}
point_labels[Point.new(0, 0)] = "origin"
point_labels[Point.new(1, 2)] = "point A"
puts point_labels[Point.new(0, 0)]  # "origin"
```

**How Ruby hashes work internally:**

```
Hash Function: key → hash value → bucket index

Ruby uses open addressing with linear probing (since Ruby 2.4):
  - Keys and values stored in an ordered array (preserves insertion order)
  - A separate hash table maps hash values to indices in the array
  - On collision, probes the next slot

Load factor management:
  When the table gets too full → rehash: allocate larger table,
  re-insert all entries — O(n)

Ruby's hash function:
  - Integers: identity hash (with SipHash mixing)
  - Strings: SipHash-1-3 (randomized per process for security)
  - Objects: Object#hash method (override for custom keys)
```

**Hash time complexity:**

| Operation | Average | Worst Case | Notes |
|-----------|---------|------------|-------|
| Insert (`[]=`) | O(1) | O(n) | Worst case during rehash |
| Lookup (`[]`) | O(1) | O(n) | Worst case: many collisions |
| Delete | O(1) | O(n) | Same as lookup |
| Iteration | O(n) | O(n) | Preserves insertion order |

**Hash vs SortedSet / sorted approaches:**

| Feature | `Hash` | `SortedSet` / sorted array |
|---------|--------|---------------------------|
| Underlying structure | Hash Table | Red-Black Tree / sorted array |
| Ordering | Insertion order | Sorted by value |
| Lookup | O(1) average | O(log n) |
| Insert | O(1) average | O(log n) or O(n) |
| Memory | More overhead (hash table) | Less overhead |
| Use when | Need fast lookup, no sorted order needed | Need sorted order, range queries |

**When to use hashes in LLD:**
- **Caching** — key-value store for cached data
- **Deduplication** — track seen items
- **Counting** — frequency maps (word count, vote tallying)
- **Indexing** — map IDs to objects (user ID → User object)
- **Configuration** — key-value settings
- **LRU Cache** — hash + doubly linked list for O(1) operations

---


### Trees (Binary Tree, BST, AVL, Red-Black)

Trees are hierarchical data structures where each node has a parent (except the root) and zero or more children. They are fundamental to databases (B-Trees), file systems (directory trees), expression parsing (AST), and many LLD problems.

**Binary Tree — basic structure:**

```ruby
class TreeNode
  attr_accessor :data, :left, :right

  def initialize(data)
    @data = data
    @left = nil
    @right = nil
  end
end

class BinaryTree
  # Inorder: Left → Root → Right (gives sorted order for BST)
  def inorder(node, result = [])
    return result unless node
    inorder(node.left, result)
    result << node.data
    inorder(node.right, result)
    result
  end

  # Preorder: Root → Left → Right (useful for copying/serializing trees)
  def preorder(node, result = [])
    return result unless node
    result << node.data
    preorder(node.left, result)
    preorder(node.right, result)
    result
  end

  # Postorder: Left → Right → Root (useful for deletion, expression evaluation)
  def postorder(node, result = [])
    return result unless node
    postorder(node.left, result)
    postorder(node.right, result)
    result << node.data
    result
  end

  # Level-order (BFS): level by level using a queue
  def level_order(root)
    return [] unless root
    result = []
    queue = [root]

    until queue.empty?
      level_size = queue.size
      level = []
      level_size.times do
        node = queue.shift
        level << node.data
        queue.push(node.left) if node.left
        queue.push(node.right) if node.right
      end
      result << level
    end
    result
  end
end

# Usage
root = TreeNode.new(1)
root.left = TreeNode.new(2)
root.right = TreeNode.new(3)
root.left.left = TreeNode.new(4)
root.left.right = TreeNode.new(5)

bt = BinaryTree.new
puts bt.inorder(root).inspect      # [4, 2, 5, 1, 3]
puts bt.level_order(root).inspect  # [[1], [2, 3], [4, 5]]
```

**Binary Search Tree (BST):**

A BST maintains the invariant: for every node, all values in the left subtree are smaller, and all values in the right subtree are larger. This enables O(log n) search, insert, and delete on average.

```ruby
class BST
  class Node
    attr_accessor :key, :left, :right

    def initialize(key)
      @key = key
      @left = nil
      @right = nil
    end
  end

  def initialize
    @root = nil
  end

  # Insert — O(log n) average, O(n) worst (skewed tree)
  def insert(key)
    @root = insert_node(@root, key)
  end

  # Search — O(log n) average, O(n) worst
  def search(key)
    search_node(@root, key)
  end

  # Delete — O(log n) average
  def delete(key)
    @root = delete_node(@root, key)
  end

  # Inorder traversal gives sorted output
  def sorted
    result = []
    inorder(@root, result)
    result
  end

  private

  def insert_node(node, key)
    return Node.new(key) unless node
    if key < node.key
      node.left = insert_node(node.left, key)
    elsif key > node.key
      node.right = insert_node(node.right, key)
    end
    # Duplicate keys ignored
    node
  end

  def search_node(node, key)
    return false unless node
    return true if node.key == key
    if key < node.key
      search_node(node.left, key)
    else
      search_node(node.right, key)
    end
  end

  def find_min(node)
    node = node.left while node.left
    node
  end

  def delete_node(node, key)
    return nil unless node

    if key < node.key
      node.left = delete_node(node.left, key)
    elsif key > node.key
      node.right = delete_node(node.right, key)
    else
      # Found the node to delete
      # Case 1: Leaf node or one child
      return node.right unless node.left
      return node.left unless node.right

      # Case 2: Two children — replace with inorder successor
      successor = find_min(node.right)
      node.key = successor.key
      node.right = delete_node(node.right, successor.key)
    end
    node
  end

  def inorder(node, result)
    return unless node
    inorder(node.left, result)
    result << node.key
    inorder(node.right, result)
  end
end

# Usage
bst = BST.new
[5, 3, 7, 1, 4, 6, 8].each { |k| bst.insert(k) }
puts bst.sorted.inspect    # [1, 3, 4, 5, 6, 7, 8]
puts bst.search(4)         # true
puts bst.search(9)         # false
bst.delete(3)
puts bst.sorted.inspect    # [1, 4, 5, 6, 7, 8]
```

**AVL Tree — self-balancing BST:**

An AVL tree maintains the property that for every node, the heights of the left and right subtrees differ by at most 1. This guarantees O(log n) for all operations.

```ruby
class AVLTree
  class Node
    attr_accessor :key, :left, :right, :height

    def initialize(key)
      @key = key
      @left = nil
      @right = nil
      @height = 1
    end
  end

  def initialize
    @root = nil
  end

  def insert(key)
    @root = insert_node(@root, key)
  end

  def search(key)
    search_node(@root, key)
  end

  private

  def height(node)
    node ? node.height : 0
  end

  def balance_factor(node)
    node ? height(node.left) - height(node.right) : 0
  end

  def update_height(node)
    node.height = 1 + [height(node.left), height(node.right)].max
  end

  # Right rotation (for left-heavy imbalance)
  #       y              x
  #      / \            / \
  #     x   C   →     A   y
  #    / \                / \
  #   A   B              B   C
  def rotate_right(y)
    x = y.left
    b = x.right

    x.right = y
    y.left = b

    update_height(y)
    update_height(x)
    x  # new root of this subtree
  end

  # Left rotation (for right-heavy imbalance)
  #     x                y
  #    / \              / \
  #   A   y     →     x   C
  #      / \         / \
  #     B   C       A   B
  def rotate_left(x)
    y = x.right
    b = y.left

    y.left = x
    x.right = b

    update_height(x)
    update_height(y)
    y  # new root of this subtree
  end

  # Rebalance after insertion
  def rebalance(node)
    update_height(node)
    bf = balance_factor(node)

    # Left-heavy
    if bf > 1
      if balance_factor(node.left) < 0
        node.left = rotate_left(node.left)  # Left-Right case
      end
      return rotate_right(node)  # Left-Left case
    end

    # Right-heavy
    if bf < -1
      if balance_factor(node.right) > 0
        node.right = rotate_right(node.right)  # Right-Left case
      end
      return rotate_left(node)  # Right-Right case
    end

    node  # balanced
  end

  def insert_node(node, key)
    return Node.new(key) unless node
    if key < node.key
      node.left = insert_node(node.left, key)
    elsif key > node.key
      node.right = insert_node(node.right, key)
    else
      return node  # duplicate
    end

    rebalance(node)
  end

  def search_node(node, key)
    return false unless node
    return true if node.key == key
    key < node.key ? search_node(node.left, key) : search_node(node.right, key)
  end
end
```

**Red-Black Tree:**

A Red-Black tree is another self-balancing BST. Ruby's `SortedSet` (from the `sorted_set` gem or older stdlib) and the `rbtree` gem provide Red-Black tree implementations. In practice, you rarely implement one from scratch in Ruby — you use the sorted containers available.

**Red-Black tree properties:**
1. Every node is either red or black
2. The root is always black
3. Every leaf (NIL) is black
4. If a node is red, both its children are black (no two consecutive reds)
5. Every path from a node to its descendant NIL nodes has the same number of black nodes

```
Red-Black Tree guarantees:
- Height ≤ 2 * log₂(n + 1)
- All operations: O(log n) guaranteed
- Less strict balancing than AVL → fewer rotations on insert/delete
- AVL is more strictly balanced → faster lookups but slower inserts/deletes
```

**When to use which tree in LLD:**

| Tree Type | Guarantee | Best For |
|-----------|-----------|----------|
| BST (unbalanced) | O(log n) avg, O(n) worst | Simple cases, educational |
| AVL Tree | O(log n) guaranteed | Read-heavy workloads (more balanced) |
| Red-Black Tree | O(log n) guaranteed | Write-heavy workloads (fewer rotations) |
| `SortedSet` / `rbtree` gem | O(log n) guaranteed | Default sorted container in Ruby |

**LLD use cases for trees:**
- **File system** — directory hierarchy (Composite pattern)
- **Expression parsing** — Abstract Syntax Tree (AST)
- **Database indexing** — B-Trees, B+ Trees
- **Autocomplete** — Trie (prefix tree)
- **Priority scheduling** — BST for ordered task management
- **Organizational hierarchy** — employee reporting structure

---

### Heaps (Min-Heap, Max-Heap)

A **heap** is a complete binary tree that satisfies the heap property: in a min-heap, every parent is smaller than its children; in a max-heap, every parent is larger. Heaps are the backbone of priority queues. Ruby does not have a built-in heap, so you implement one or use a gem.

**Custom MinHeap implementation:**

```ruby
class MinHeap
  def initialize
    @data = []
  end

  # O(log n) — add element and bubble up
  def push(val)
    @data << val
    heapify_up(@data.size - 1)
  end

  # O(1) — peek at minimum element
  def top
    @data.first
  end

  # O(log n) — remove minimum and bubble down
  def pop
    return nil if @data.empty?
    min = @data.first
    @data[0] = @data.last
    @data.pop
    heapify_down(0) unless @data.empty?
    min
  end

  def empty?
    @data.empty?
  end

  def size
    @data.size
  end

  private

  def heapify_up(index)
    while index > 0
      parent = (index - 1) / 2
      if @data[index] < @data[parent]
        @data[index], @data[parent] = @data[parent], @data[index]
        index = parent
      else
        break
      end
    end
  end

  def heapify_down(index)
    size = @data.size
    loop do
      smallest = index
      left = 2 * index + 1
      right = 2 * index + 2

      smallest = left if left < size && @data[left] < @data[smallest]
      smallest = right if right < size && @data[right] < @data[smallest]

      if smallest != index
        @data[index], @data[smallest] = @data[smallest], @data[index]
        index = smallest
      else
        break
      end
    end
  end
end

# Usage
heap = MinHeap.new
[30, 10, 50, 20, 40].each { |v| heap.push(v) }

puts "Min: #{heap.top}"  # 10
puts heap.pop             # 10
puts heap.pop             # 20
puts heap.pop             # 30
```

**MaxHeap — invert the comparison:**

```ruby
class MaxHeap
  def initialize
    @data = []
  end

  def push(val)
    @data << val
    heapify_up(@data.size - 1)
  end

  def top
    @data.first
  end

  def pop
    return nil if @data.empty?
    max = @data.first
    @data[0] = @data.last
    @data.pop
    heapify_down(0) unless @data.empty?
    max
  end

  def empty?
    @data.empty?
  end

  def size
    @data.size
  end

  private

  def heapify_up(index)
    while index > 0
      parent = (index - 1) / 2
      if @data[index] > @data[parent]
        @data[index], @data[parent] = @data[parent], @data[index]
        index = parent
      else
        break
      end
    end
  end

  def heapify_down(index)
    size = @data.size
    loop do
      largest = index
      left = 2 * index + 1
      right = 2 * index + 2

      largest = left if left < size && @data[left] > @data[largest]
      largest = right if right < size && @data[right] > @data[largest]

      if largest != index
        @data[index], @data[largest] = @data[largest], @data[index]
        index = largest
      else
        break
      end
    end
  end
end
```

**Priority Queue with custom comparator:**

```ruby
class PriorityQueue
  # compare_proc should return true if a has higher priority than b
  def initialize(&compare_proc)
    @data = []
    @compare = compare_proc || ->(a, b) { a < b }  # min-heap by default
  end

  def push(val)
    @data << val
    heapify_up(@data.size - 1)
  end

  def top
    @data.first
  end

  def pop
    return nil if @data.empty?
    top_val = @data.first
    @data[0] = @data.last
    @data.pop
    heapify_down(0) unless @data.empty?
    top_val
  end

  def empty?
    @data.empty?
  end

  def size
    @data.size
  end

  private

  def heapify_up(index)
    while index > 0
      parent = (index - 1) / 2
      if @compare.call(@data[index], @data[parent])
        @data[index], @data[parent] = @data[parent], @data[index]
        index = parent
      else
        break
      end
    end
  end

  def heapify_down(index)
    size = @data.size
    loop do
      target = index
      left = 2 * index + 1
      right = 2 * index + 2

      target = left if left < size && @compare.call(@data[left], @data[target])
      target = right if right < size && @compare.call(@data[right], @data[target])

      if target != index
        @data[index], @data[target] = @data[target], @data[index]
        index = target
      else
        break
      end
    end
  end
end

# Usage: Priority task queue (higher priority first)
Task = Struct.new(:id, :priority, :name)

task_queue = PriorityQueue.new { |a, b| a.priority > b.priority }
task_queue.push(Task.new(1, 5, "Low priority"))
task_queue.push(Task.new(2, 10, "High priority"))
task_queue.push(Task.new(3, 7, "Medium priority"))

until task_queue.empty?
  task = task_queue.pop
  puts "Processing: #{task.name} (priority #{task.priority})"
end
# Output: High priority (10), Medium priority (7), Low priority (5)
```

**How a heap works internally (array representation):**

```
Max-Heap as array: [50, 30, 40, 10, 20, 35, 25]

Tree representation:
           50
         /    \
       30      40
      /  \    /  \
    10   20  35   25

Parent of index i: (i - 1) / 2
Left child of i:   2 * i + 1
Right child of i:  2 * i + 2

Insert (push): Add at end, "bubble up" (swap with parent while larger)
Delete (pop):  Replace root with last element, "bubble down" (swap with larger child)
```

**Heap time complexity:**

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `push` | O(log n) | Bubble up |
| `pop` | O(log n) | Bubble down |
| `top` | O(1) | Peek at root |
| Build heap from array | O(n) | Heapify |

**When to use heaps in LLD:**
- **Task scheduling** — process highest priority task first
- **Leaderboards** — top-K players (min-heap of size K)
- **Median finding** — two heaps (max-heap for lower half, min-heap for upper half)
- **Dijkstra's algorithm** — shortest path with priority queue
- **Merge K sorted lists** — min-heap of K elements
- **Event-driven simulation** — process earliest event first
- **Rate limiting** — sliding window with timestamps in a heap

---

### Graphs (Adjacency List, Adjacency Matrix)

A **graph** consists of vertices (nodes) and edges (connections). Graphs model relationships — social networks, road maps, dependency chains, state machines, and more.

**Graph representations:**

```ruby
# Representation 1: Adjacency Matrix
# Good for dense graphs, O(1) edge lookup, O(V²) space
class GraphMatrix
  def initialize(vertices)
    @v = vertices
    @adj = Array.new(vertices) { Array.new(vertices, 0) }
  end

  def add_edge(u, v, directed: false)
    @adj[u][v] = 1
    @adj[v][u] = 1 unless directed
  end

  def has_edge?(u, v)
    @adj[u][v] == 1
  end

  def to_s
    @adj.map { |row| row.join(" ") }.join("\n")
  end
end

# Representation 2: Adjacency List (most common)
# Good for sparse graphs, O(V + E) space, efficient iteration over neighbors
class GraphList
  def initialize(vertices)
    @v = vertices
    @adj = Array.new(vertices) { [] }
  end

  def add_edge(u, v, directed: false)
    @adj[u] << v
    @adj[v] << u unless directed
  end

  def neighbors(u)
    @adj[u]
  end

  def to_s
    @adj.each_with_index.map { |neighbors, i| "#{i} → #{neighbors.join(' ')}" }.join("\n")
  end
end

# Representation 3: Adjacency List with string keys (for real-world LLD)
class SocialGraph
  def initialize
    @adj = Hash.new { |h, k| h[k] = [] }
  end

  def add_user(user)
    @adj[user] ||= []
  end

  def add_friendship(u1, u2)
    @adj[u1] << u2
    @adj[u2] << u1
  end

  def friends(user)
    @adj[user]
  end

  # BFS to find shortest path (degrees of separation)
  def degrees_of_separation(from, to)
    return 0 if from == to

    visited = Set.new([from])
    queue = [[from, 0]]

    until queue.empty?
      user, dist = queue.shift

      @adj[user].each do |friend|
        return dist + 1 if friend == to
        unless visited.include?(friend)
          visited.add(friend)
          queue.push([friend, dist + 1])
        end
      end
    end

    -1  # not connected
  end
end

# Usage
require 'set'

graph = SocialGraph.new
graph.add_friendship("Alice", "Bob")
graph.add_friendship("Bob", "Charlie")
graph.add_friendship("Charlie", "Diana")

puts graph.degrees_of_separation("Alice", "Diana")  # 3
puts graph.friends("Bob").inspect                     # ["Alice", "Charlie"]
```

**Weighted graph:**

```ruby
class WeightedGraph
  Edge = Struct.new(:to, :weight)

  def initialize(vertices)
    @v = vertices
    @adj = Array.new(vertices) { [] }
  end

  def add_edge(u, v, weight, directed: false)
    @adj[u] << Edge.new(v, weight)
    @adj[v] << Edge.new(u, weight) unless directed
  end

  def neighbors(u)
    @adj[u]
  end

  def vertices
    @v
  end
end
```

**Graph representation comparison:**

| Feature | Adjacency Matrix | Adjacency List |
|---------|-----------------|----------------|
| Space | O(V²) | O(V + E) |
| Check edge exists | O(1) | O(degree) |
| Get all neighbors | O(V) | O(degree) |
| Add edge | O(1) | O(1) |
| Best for | Dense graphs | Sparse graphs (most real-world) |

**When to use graphs in LLD:**
- **Social networks** — friend connections, followers
- **Maps/Navigation** — roads, routes, shortest path
- **Dependency resolution** — build systems, package managers
- **State machines** — game states, workflow engines
- **Recommendation engines** — user-item bipartite graphs
- **Network topology** — server connections, routing

---

### Tries (Prefix Trees)

A **trie** (pronounced "try") is a tree-like data structure used for efficient prefix-based operations on strings. Each node represents a character, and paths from root to nodes represent prefixes.

```ruby
class Trie
  class TrieNode
    attr_accessor :children, :end_of_word, :word_count

    def initialize
      @children = {}
      @end_of_word = false
      @word_count = 0  # how many words pass through this node
    end
  end

  def initialize
    @root = TrieNode.new
  end

  # Insert a word — O(L) where L is word length
  def insert(word)
    node = @root
    word.each_char do |ch|
      node.children[ch] ||= TrieNode.new
      node = node.children[ch]
      node.word_count += 1
    end
    node.end_of_word = true
  end

  # Search for exact word — O(L)
  def search(word)
    node = find_node(word)
    !node.nil? && node.end_of_word
  end

  # Check if any word starts with prefix — O(L)
  def starts_with?(prefix)
    !find_node(prefix).nil?
  end

  # Autocomplete: find all words with given prefix
  def autocomplete(prefix, limit = 10)
    node = find_node(prefix)
    return [] unless node

    results = []
    dfs_collect(node, prefix.dup, results, limit)
    results
  end

  private

  def find_node(prefix)
    node = @root
    prefix.each_char do |ch|
      return nil unless node.children.key?(ch)
      node = node.children[ch]
    end
    node
  end

  def dfs_collect(node, current, results, limit)
    return if results.size >= limit

    results << current.dup if node.end_of_word

    node.children.each do |ch, child|
      current << ch
      dfs_collect(child, current, results, limit)
      current.chop!  # remove last character (backtrack)
    end
  end
end

# Usage
trie = Trie.new
%w[apple app application apply banana].each { |w| trie.insert(w) }

puts trie.search("app")          # true
puts trie.search("apt")          # false
puts trie.starts_with?("app")    # true

suggestions = trie.autocomplete("app")
puts "Autocomplete 'app': #{suggestions.inspect}"
# Output: ["app", "apple", "application", "apply"]
```

**Trie time complexity:**

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| Insert | O(L) | L = length of word |
| Search | O(L) | |
| Prefix search | O(L) | |
| Autocomplete | O(L + K) | K = number of results |
| Space | O(N * L * A) | N words, L avg length, A alphabet size |

**When to use tries in LLD:**
- **Autocomplete / Typeahead** — search suggestions
- **Spell checker** — find similar words
- **IP routing** — longest prefix match
- **Dictionary** — word lookup and validation
- **Phone directory** — contact search by prefix
- **URL routing** — match URL patterns

---

### Essential Data Structures Summary

| Data Structure | Insert | Delete | Search | Access | Use Case |
|---------------|--------|--------|--------|--------|----------|
| Array | O(1)* / O(n) | O(n) | O(n) | O(1) | Default container, random access |
| Linked List | O(1)† | O(1)† | O(n) | O(n) | LRU cache, frequent mid-insertions |
| Stack (Array) | O(1) | O(1) | O(n) | O(1) top | Undo/redo, DFS, expression eval |
| Queue | O(1) | O(1)‡ | O(n) | O(1) front | BFS, task scheduling, FIFO |
| Hash | O(1) avg | O(1) avg | O(1) avg | O(1) avg | Caching, indexing, counting |
| BST | O(log n)§ | O(log n)§ | O(log n)§ | — | Sorted data, range queries |
| Heap | O(log n) | O(log n) | O(n) | O(1) top | Priority queues, top-K |
| Graph | O(1) | O(E) | O(V+E) | — | Networks, relationships, routing |
| Trie | O(L) | O(L) | O(L) | — | Prefix search, autocomplete |

\* amortized for `push`
† given a reference to the node
‡ O(1) with `Thread::Queue` or two-stack queue; O(n) with `Array#shift`
§ average case for unbalanced BST; O(log n) guaranteed for AVL/Red-Black

---


## 10.2 Algorithms Relevant to LLD

> Algorithms are the engines that power LLD components. A leaderboard needs sorting, a cache needs eviction algorithms, a map needs shortest-path, and a rate limiter needs token bucket or sliding window. This section covers the algorithms most relevant to Low-Level Design problems, with full Ruby implementations.

---

### Sorting (for Leaderboards, Rankings)

Sorting is fundamental to any system that displays ordered data — leaderboards, search results, timelines, and rankings.

**Array#sort — the default choice:**

```ruby
Player = Struct.new(:name, :score, :games_played)

players = [
  Player.new("Alice", 1500, 20),
  Player.new("Bob", 1800, 15),
  Player.new("Charlie", 1500, 25),
  Player.new("Diana", 2000, 10)
]

# Sort by score descending
by_score = players.sort_by { |p| -p.score }

# Multi-criteria sort: by score desc, then by games_played asc (tiebreaker)
ranked = players.sort_by { |p| [-p.score, p.games_played] }

puts "Leaderboard:"
ranked.each_with_index do |p, i|
  puts "#{i + 1}. #{p.name} - Score: #{p.score} - Games: #{p.games_played}"
end
# 1. Diana - Score: 2000 - Games: 10
# 2. Bob - Score: 1800 - Games: 15
# 3. Alice - Score: 1500 - Games: 20
# 4. Charlie - Score: 1500 - Games: 25
```

**sort vs sort_by:**

```ruby
# sort with spaceship operator — flexible but slower for complex comparisons
players.sort { |a, b| b.score <=> a.score }

# sort_by — Schwartzian transform, computes key once per element
# More efficient when the comparison key is expensive to compute
players.sort_by { |p| -p.score }

# sort_by with multiple keys (Ruby compares arrays element by element)
players.sort_by { |p| [-p.score, p.games_played] }
```

**Stable sort — preserves relative order of equal elements:**

```ruby
# Ruby's sort is NOT guaranteed to be stable (implementation-dependent).
# To get a stable sort, include the original index as a tiebreaker:
stable_sorted = players.each_with_index
  .sort_by { |p, i| [-p.score, i] }
  .map(&:first)
```

**Partial sort — when you only need top-K:**

```ruby
# Ruby doesn't have a built-in partial_sort, but you can use:

# Option 1: sort and take (O(n log n) + O(k))
top_10 = players.sort_by { |p| -p.score }.first(10)

# Option 2: max_by for top-K (O(n * k) — better when k << n)
top_3 = players.max_by(3) { |p| p.score }

# Option 3: min_by for bottom-K
bottom_3 = players.min_by(3) { |p| p.score }
```

**Finding the K-th element:**

```ruby
# Ruby doesn't have nth_element, but you can use:

# Quickselect-style approach (O(n) average)
# Or simply sort and index (O(n log n)):
scores = players.map(&:score).sort
median = scores[scores.size / 2]

# For min/max without full sort:
min_score = players.min_by(&:score)  # O(n)
max_score = players.max_by(&:score)  # O(n)
```

**Sorting algorithm comparison:**

| Method | Time (avg) | Stable | Notes |
|--------|-----------|--------|-------|
| `sort` | O(n log n) | No* | Uses quicksort (MRI Ruby) |
| `sort_by` | O(n log n) | No* | Schwartzian transform — faster for complex keys |
| `sort` with index tiebreaker | O(n log n) | Yes | Manual stable sort |
| `max_by(k)` | O(n * k) | — | Top-K without full sort |
| `min_by(k)` | O(n * k) | — | Bottom-K without full sort |

\* MRI Ruby uses quicksort which is not stable. JRuby uses merge sort which is stable.

**LLD use cases for sorting:**
- **Leaderboards** — sort players by score
- **Search results** — sort by relevance, date, price
- **Timeline/Feed** — sort posts by timestamp
- **Scheduling** — sort tasks by priority or deadline
- **Top-K queries** — `max_by(k)` for efficiency

---

### Searching (for Lookup Systems)

**Linear search — O(n):**

```ruby
# Simple but works on unsorted data
def linear_search(data, target)
  data.each_with_index do |item, i|
    return i if item == target
  end
  -1  # not found
end

# Ruby built-ins for linear search
arr = [10, 20, 30, 40, 50]
arr.index(30)        # 2 — first index of value
arr.include?(30)     # true
arr.find { |x| x > 25 }  # 30 — first element matching condition
```

**Binary search — O(log n) on sorted data:**

```ruby
# Manual binary search
def binary_search(sorted_data, target)
  left = 0
  right = sorted_data.size - 1

  while left <= right
    mid = left + (right - left) / 2  # avoid overflow (less relevant in Ruby)

    if sorted_data[mid] == target
      return mid
    elsif sorted_data[mid] < target
      left = mid + 1
    else
      right = mid - 1
    end
  end
  -1  # not found
end

# Ruby built-in binary search
data = [1, 3, 5, 7, 9, 11, 13, 15]

# bsearch — find element matching condition — O(log n)
# "Find minimum" mode: finds first element where block returns true
val = data.bsearch { |x| x >= 6 }   # 7 (first element >= 6)

# bsearch_index — returns index instead of value
idx = data.bsearch_index { |x| x >= 6 }  # 3

# "Find any" mode: block returns negative, 0, or positive (like <=>)
val = data.bsearch { |x| 7 <=> x }  # 7 (exact match)
```

**Binary search on custom objects:**

```ruby
Product = Struct.new(:id, :name, :price)

# Search products sorted by price
products = [
  Product.new(1, "Widget", 9.99),
  Product.new(2, "Gadget", 19.99),
  Product.new(3, "Doohickey", 29.99),
  Product.new(4, "Thingamajig", 39.99)
]

# Find first product with price >= 20.00
result = products.bsearch { |p| p.price >= 20.00 }
puts result.name if result  # "Doohickey"
```

**When to use which search:**

| Method | Time | Requirement | Use Case |
|--------|------|-------------|----------|
| Linear search (`find`, `index`) | O(n) | None | Small datasets, unsorted data |
| Binary search (`bsearch`) | O(log n) | Sorted data | Large sorted datasets |
| Hash lookup (`Hash#[]`) | O(1) avg | Hash built | Key-value lookups |
| Trie search | O(L) | Trie built | Prefix-based search |
| BFS/DFS | O(V+E) | Graph | Graph traversal, path finding |

---

### Graph Traversal — BFS/DFS (for Social Networks, Dependency Resolution)

**Breadth-First Search (BFS) — level by level:**

BFS explores all neighbors at the current depth before moving to the next level. It finds the shortest path in unweighted graphs.

```ruby
require 'set'

class Graph
  def initialize
    @adj = Hash.new { |h, k| h[k] = [] }
  end

  def add_edge(u, v)
    @adj[u] << v
    @adj[v] << u
  end

  # BFS — O(V + E)
  def bfs(start)
    order = []
    visited = Set.new([start])
    queue = [start]

    until queue.empty?
      node = queue.shift
      order << node

      @adj[node].each do |neighbor|
        unless visited.include?(neighbor)
          visited.add(neighbor)
          queue.push(neighbor)
        end
      end
    end
    order
  end

  # BFS shortest path (unweighted)
  def shortest_path(start, target)
    return 0 if start == target

    visited = Set.new([start])
    queue = [[start, 0]]  # [node, distance]

    until queue.empty?
      node, dist = queue.shift

      @adj[node].each do |neighbor|
        return dist + 1 if neighbor == target
        unless visited.include?(neighbor)
          visited.add(neighbor)
          queue.push([neighbor, dist + 1])
        end
      end
    end
    -1  # no path
  end

  # BFS level-order (useful for "friends of friends")
  def bfs_levels(start)
    levels = []
    visited = Set.new([start])
    queue = [start]

    until queue.empty?
      level_size = queue.size
      current_level = []

      level_size.times do
        node = queue.shift
        current_level << node

        @adj[node].each do |neighbor|
          unless visited.include?(neighbor)
            visited.add(neighbor)
            queue.push(neighbor)
          end
        end
      end
      levels << current_level
    end
    levels
  end
end

# Usage
g = Graph.new
g.add_edge(1, 2)
g.add_edge(1, 3)
g.add_edge(2, 4)
g.add_edge(3, 5)

puts g.bfs(1).inspect           # [1, 2, 3, 4, 5]
puts g.shortest_path(1, 5)      # 2
puts g.bfs_levels(1).inspect    # [[1], [2, 3], [4, 5]]
```

**Depth-First Search (DFS) — go deep first:**

DFS explores as far as possible along each branch before backtracking. It's used for cycle detection, topological sorting, and connected components.

```ruby
require 'set'

class DirectedGraph
  def initialize
    @adj = Hash.new { |h, k| h[k] = [] }
  end

  def add_edge(u, v)
    @adj[u] << v
  end

  def vertices
    @adj.keys
  end

  # DFS recursive — O(V + E)
  def dfs(start, visited = Set.new)
    visited.add(start)
    order = [start]

    @adj[start].each do |neighbor|
      unless visited.include?(neighbor)
        order.concat(dfs(neighbor, visited))
      end
    end
    order
  end

  # DFS iterative (using stack)
  def dfs_iterative(start)
    order = []
    visited = Set.new
    stack = [start]

    until stack.empty?
      node = stack.pop
      next if visited.include?(node)

      visited.add(node)
      order << node

      @adj[node].each do |neighbor|
        stack.push(neighbor) unless visited.include?(neighbor)
      end
    end
    order
  end

  # Cycle detection in directed graph using DFS
  def has_cycle?
    visited = Set.new
    rec_stack = Set.new  # nodes in current recursion path

    vertices.each do |node|
      unless visited.include?(node)
        return true if dfs_cycle?(node, visited, rec_stack)
      end
    end
    false
  end

  # Topological Sort (for dependency resolution) — only for DAGs
  # Kahn's algorithm (BFS-based)
  def topological_sort
    in_degree = Hash.new(0)
    vertices.each { |v| in_degree[v] ||= 0 }
    @adj.each do |_, neighbors|
      neighbors.each { |n| in_degree[n] += 1 }
    end

    queue = in_degree.select { |_, deg| deg == 0 }.keys
    order = []

    until queue.empty?
      node = queue.shift
      order << node

      @adj[node].each do |neighbor|
        in_degree[neighbor] -= 1
        queue.push(neighbor) if in_degree[neighbor] == 0
      end
    end

    # If order.size != number of nodes → cycle exists
    order
  end

  private

  def dfs_cycle?(node, visited, rec_stack)
    visited.add(node)
    rec_stack.add(node)

    @adj[node].each do |neighbor|
      return true if rec_stack.include?(neighbor)  # back edge = cycle
      unless visited.include?(neighbor)
        return true if dfs_cycle?(neighbor, visited, rec_stack)
      end
    end

    rec_stack.delete(node)
    false
  end
end

# Usage
g = DirectedGraph.new
g.add_edge(:a, :b)
g.add_edge(:a, :c)
g.add_edge(:b, :d)
g.add_edge(:c, :d)

puts g.topological_sort.inspect  # [:a, :b, :c, :d] or [:a, :c, :b, :d]
puts g.has_cycle?                # false

# Add a cycle
g.add_edge(:d, :a)
puts g.has_cycle?                # true
```

**BFS vs DFS comparison:**

| Feature | BFS | DFS |
|---------|-----|-----|
| Data structure | Queue (Array) | Stack (Array or recursion) |
| Explores | Level by level | Branch by branch |
| Shortest path (unweighted) | Yes | No |
| Space complexity | O(V) — width of graph | O(V) — depth of graph |
| Cycle detection | Yes | Yes (with recursion stack) |
| Topological sort | Yes (Kahn's algorithm) | Yes (reverse post-order) |
| Use case | Shortest path, level-order | Cycle detection, topological sort |

**LLD use cases:**
- **Social networks** — BFS for "people you may know" (friends of friends)
- **Dependency resolution** — topological sort for build systems, package managers (Bundler)
- **Maze solving** — BFS for shortest path
- **Spreadsheet** — DFS for dependency graph, cycle detection for circular references
- **State machines** — DFS/BFS for reachability analysis

---

### Shortest Path (for Maps, Routing)

Shortest path algorithms are essential for navigation systems, network routing, and any system that needs to find optimal paths.

**Dijkstra's Algorithm — shortest path from a single source (non-negative weights):**

```ruby
class ShortestPath
  Edge = Struct.new(:to, :weight)
  INFINITY = Float::INFINITY

  def initialize(vertices)
    @v = vertices
    @adj = Array.new(vertices) { [] }
  end

  def add_edge(u, v, weight, directed: false)
    @adj[u] << Edge.new(v, weight)
    @adj[v] << Edge.new(u, weight) unless directed
  end

  # Dijkstra's Algorithm — O((V + E) log V) with priority queue
  def dijkstra(source)
    dist = Array.new(@v, INFINITY)
    dist[source] = 0

    # Min-heap: [distance, node]
    # Ruby doesn't have a built-in priority queue, so we use our MinHeap
    # or a simple sorted approach. Here we use a basic array-based approach.
    visited = Array.new(@v, false)

    @v.times do
      # Find unvisited node with minimum distance — O(V) per iteration
      # Total: O(V²) — acceptable for small graphs
      u = -1
      (0...@v).each do |i|
        if !visited[i] && (u == -1 || dist[i] < dist[u])
          u = i
        end
      end

      break if dist[u] == INFINITY
      visited[u] = true

      @adj[u].each do |edge|
        new_dist = dist[u] + edge.weight
        dist[edge.to] = new_dist if new_dist < dist[edge.to]
      end
    end

    dist
  end

  # Dijkstra with priority queue (using our MinHeap) — O((V + E) log V)
  def dijkstra_pq(source)
    dist = Array.new(@v, INFINITY)
    dist[source] = 0

    # Priority queue: [distance, node]
    pq = PriorityQueue.new { |a, b| a[0] < b[0] }
    pq.push([0, source])

    until pq.empty?
      d, u = pq.pop
      next if d > dist[u]  # skip if we already found a shorter path

      @adj[u].each do |edge|
        new_dist = dist[u] + edge.weight
        if new_dist < dist[edge.to]
          dist[edge.to] = new_dist
          pq.push([new_dist, edge.to])
        end
      end
    end

    dist
  end

  # Dijkstra with path reconstruction
  def shortest_path_with_route(source, target)
    dist = Array.new(@v, INFINITY)
    parent = Array.new(@v, -1)
    dist[source] = 0

    pq = PriorityQueue.new { |a, b| a[0] < b[0] }
    pq.push([0, source])

    until pq.empty?
      d, u = pq.pop
      break if u == target  # found shortest path to target
      next if d > dist[u]

      @adj[u].each do |edge|
        new_dist = dist[u] + edge.weight
        if new_dist < dist[edge.to]
          dist[edge.to] = new_dist
          parent[edge.to] = u
          pq.push([new_dist, edge.to])
        end
      end
    end

    # Reconstruct path
    return [-1, []] if dist[target] == INFINITY

    path = []
    node = target
    while node != -1
      path.unshift(node)
      node = parent[node]
    end

    [dist[target], path]
  end
end

# Usage: Navigation system
# Cities: 0=NYC, 1=Boston, 2=DC, 3=Philadelphia, 4=Baltimore
graph = ShortestPath.new(5)
graph.add_edge(0, 1, 215)  # NYC → Boston: 215 miles
graph.add_edge(0, 3, 97)   # NYC → Philadelphia: 97 miles
graph.add_edge(1, 2, 440)  # Boston → DC: 440 miles
graph.add_edge(3, 4, 100)  # Philadelphia → Baltimore: 100 miles
graph.add_edge(4, 2, 40)   # Baltimore → DC: 40 miles

dist, path = graph.shortest_path_with_route(0, 2)
puts "NYC to DC: #{dist} miles"
puts "Route: #{path.join(' → ')}"
# NYC → Philadelphia → Baltimore → DC = 237 miles
```

**Bellman-Ford — handles negative weights:**

```ruby
BFEdge = Struct.new(:from, :to, :weight)

def bellman_ford(vertices, edges, source)
  dist = Array.new(vertices, Float::INFINITY)
  dist[source] = 0

  # Relax all edges V-1 times
  (vertices - 1).times do
    edges.each do |e|
      if dist[e.from] != Float::INFINITY && dist[e.from] + e.weight < dist[e.to]
        dist[e.to] = dist[e.from] + e.weight
      end
    end
  end

  # Check for negative weight cycles (one more iteration)
  edges.each do |e|
    if dist[e.from] != Float::INFINITY && dist[e.from] + e.weight < dist[e.to]
      puts "Negative weight cycle detected!"
      return []
    end
  end

  dist
end
```

**Shortest path algorithm comparison:**

| Algorithm | Time | Negative Weights | Use Case |
|-----------|------|-----------------|----------|
| Dijkstra | O((V+E) log V) | No | GPS navigation, network routing |
| Bellman-Ford | O(V * E) | Yes | Currency exchange, detecting negative cycles |
| BFS | O(V + E) | Unweighted only | Social network distance, maze solving |
| Floyd-Warshall | O(V³) | Yes | All-pairs shortest path (small graphs) |
| A* | O(E) with good heuristic | No | Game pathfinding, map navigation |

---

### Hashing (for Caching, Deduplication)

Hashing maps data of arbitrary size to fixed-size values. It's the foundation of hash maps, caches, deduplication systems, and data integrity checks.

**Hash function properties:**
- **Deterministic** — same input always produces same output
- **Uniform distribution** — outputs spread evenly across the range
- **Fast to compute** — O(1) for fixed-size inputs
- **Avalanche effect** — small input change → large output change (for cryptographic hashes)

```ruby
# Ruby's built-in hash method
puts 42.hash              # integer hash
puts "hello".hash         # string hash (randomized per process)
puts [1, 2, 3].hash       # array hash (combines element hashes)

# Custom hash for composite keys (define hash and eql?)
class UserSession
  attr_reader :user_id, :session_token

  def initialize(user_id, session_token)
    @user_id = user_id
    @session_token = session_token
  end

  def eql?(other)
    other.is_a?(UserSession) &&
      @user_id == other.user_id &&
      @session_token == other.session_token
  end

  def hash
    [@user_id, @session_token].hash
  end
end

# Now UserSession can be used as a Hash key
sessions = {}
sessions[UserSession.new(1, "abc")] = { logged_in: true }
```

**Deduplication using sets:**

```ruby
require 'set'

# Remove duplicate requests/messages
class Deduplicator
  def initialize
    @seen = Set.new
  end

  def duplicate?(message)
    hash_val = message.hash
    if @seen.include?(hash_val)
      true  # likely duplicate
    else
      @seen.add(hash_val)
      false
    end
  end

  def clear
    @seen.clear
  end

  def unique_count
    @seen.size
  end
end

dedup = Deduplicator.new
puts dedup.duplicate?("hello")  # false (first time)
puts dedup.duplicate?("world")  # false
puts dedup.duplicate?("hello")  # true (seen before)
```

**Bloom Filter — space-efficient probabilistic deduplication:**

A Bloom filter can tell you "definitely not in set" or "probably in set" using very little memory. It uses multiple hash functions and a bit array.

```ruby
require 'digest'

class BloomFilter
  def initialize(size, num_hashes = 3)
    @size = size
    @num_hashes = num_hashes
    @bits = Array.new(size, false)
  end

  def insert(key)
    get_hashes(key).each { |h| @bits[h] = true }
  end

  # Returns true if POSSIBLY in set, false if DEFINITELY NOT in set
  def possibly_contains?(key)
    get_hashes(key).all? { |h| @bits[h] }
  end

  private

  def get_hashes(key)
    hashes = []
    @num_hashes.times do |i|
      # Use MD5 with salt for multiple hash functions
      digest = Digest::MD5.hexdigest("#{i}:#{key}")
      hashes << (digest.to_i(16) % @size)
    end
    hashes
  end
end

# Usage
filter = BloomFilter.new(1000, 3)
filter.insert("user_123")
filter.insert("user_456")

puts filter.possibly_contains?("user_123")  # true (correct)
puts filter.possibly_contains?("user_789")  # false (correct) or true (false positive)
```

**LLD use cases for hashing:**
- **Caching** — hash keys for cache lookup
- **Deduplication** — detect duplicate messages, requests, or data
- **Load balancing** — hash-based request routing
- **Data partitioning** — hash-based sharding
- **Checksums** — data integrity verification
- **Bloom filters** — space-efficient membership testing

---


### LRU Cache Implementation (Hash + Doubly Linked List)

The **LRU (Least Recently Used) Cache** is one of the most frequently asked LLD problems. It evicts the least recently accessed item when the cache is full. The key insight is combining a hash (O(1) lookup) with a doubly linked list (O(1) insertion/deletion/reordering).

```ruby
class LRUCache
  class Node
    attr_accessor :key, :value, :prev, :next_node

    def initialize(key, value)
      @key = key
      @value = value
      @prev = nil
      @next_node = nil
    end
  end

  def initialize(capacity)
    @capacity = capacity
    @cache = {}  # key → Node

    # Dummy head and tail for easy boundary handling
    @head = Node.new(0, 0)  # most recently used (MRU)
    @tail = Node.new(0, 0)  # least recently used (LRU)
    @head.next_node = @tail
    @tail.prev = @head
  end

  # Get value by key — O(1)
  def get(key)
    node = @cache[key]
    return -1 unless node  # cache miss

    move_to_front(node)  # mark as recently used
    node.value
  end

  # Put key-value pair — O(1)
  def put(key, value)
    if @cache.key?(key)
      # Key exists — update value and move to front
      node = @cache[key]
      node.value = value
      move_to_front(node)
    else
      # Key doesn't exist — insert new
      if @cache.size >= @capacity
        # Evict LRU
        lru = remove_lru
        @cache.delete(lru.key)
      end

      new_node = Node.new(key, value)
      @cache[key] = new_node
      add_to_front(new_node)
    end
  end

  def print_cache
    items = []
    current = @head.next_node
    while current != @tail
      items << "[#{current.key}:#{current.value}]"
      current = current.next_node
    end
    puts "Cache (MRU → LRU): #{items.join(' ')}"
  end

  private

  # Remove a node from the doubly linked list — O(1)
  def remove_node(node)
    node.prev.next_node = node.next_node
    node.next_node.prev = node.prev
  end

  # Add a node right after head (most recently used position) — O(1)
  def add_to_front(node)
    node.next_node = @head.next_node
    node.prev = @head
    @head.next_node.prev = node
    @head.next_node = node
  end

  # Move an existing node to the front — O(1)
  def move_to_front(node)
    remove_node(node)
    add_to_front(node)
  end

  # Remove the LRU node (right before tail) — O(1)
  def remove_lru
    lru = @tail.prev
    remove_node(lru)
    lru
  end
end

# Usage
cache = LRUCache.new(3)  # capacity 3

cache.put(1, 10)
cache.put(2, 20)
cache.put(3, 30)
cache.print_cache  # [3:30] [2:20] [1:10]

cache.get(1)         # access key 1 → moves to front
cache.print_cache    # [1:10] [3:30] [2:20]

cache.put(4, 40)     # capacity full → evicts key 2 (LRU)
cache.print_cache    # [4:40] [1:10] [3:30]

puts "Get key 2: #{cache.get(2)}"  # -1 (evicted)
puts "Get key 3: #{cache.get(3)}"  # 30
```

**Why Hash + Doubly Linked List?**

```
Hash: key → Node reference
  Provides O(1) lookup to find any node by key

Doubly Linked List: head ↔ node1 ↔ node2 ↔ ... ↔ tail
  - Head side = Most Recently Used (MRU)
  - Tail side = Least Recently Used (LRU)
  - O(1) to move any node to front (given reference from Hash)
  - O(1) to remove from tail (eviction)

Combined: ALL operations are O(1)
  get(key):  Hash lookup → move node to front → return value
  put(key):  Hash lookup → if exists: update + move to front
                          → if new: add to front, evict from tail if full
```

---

### LFU Cache Implementation

The **LFU (Least Frequently Used) Cache** evicts the item that has been accessed the fewest times. If there's a tie, it evicts the least recently used among them.

```ruby
class LFUCache
  Node = Struct.new(:key, :value, :freq)

  def initialize(capacity)
    @capacity = capacity
    @min_freq = 0

    # key → node (stored as element in a frequency list)
    @key_map = {}

    # frequency → array of nodes with that frequency (ordered by recency)
    # front = most recent, back = least recent
    @freq_map = Hash.new { |h, k| h[k] = [] }

    # key → [freq, index_in_freq_list] for O(1) removal
    # (In production, you'd use a linked list for O(1) removal;
    #  here we use a hash-based approach for clarity)
    @key_to_node = {}
  end

  def get(key)
    return -1 unless @key_to_node.key?(key)

    node = @key_to_node[key]
    touch(node)
    node.value
  end

  def put(key, value)
    return if @capacity <= 0

    if @key_to_node.key?(key)
      # Update existing
      node = @key_to_node[key]
      node.value = value
      touch(node)
      return
    end

    # Evict if at capacity
    if @key_to_node.size >= @capacity
      # Remove LFU (and LRU among ties) — last element of min_freq list
      min_list = @freq_map[@min_freq]
      evict_node = min_list.pop
      @freq_map.delete(@min_freq) if min_list.empty?
      @key_to_node.delete(evict_node.key)
    end

    # Insert new node with frequency 1
    @min_freq = 1
    node = Node.new(key, value, 1)
    @freq_map[1].unshift(node)
    @key_to_node[key] = node
  end

  private

  def touch(node)
    freq = node.freq

    # Remove from current frequency list
    @freq_map[freq].delete(node)
    if @freq_map[freq].empty?
      @freq_map.delete(freq)
      @min_freq += 1 if @min_freq == freq
    end

    # Add to next frequency list (front = most recent)
    node.freq += 1
    @freq_map[node.freq].unshift(node)
  end
end

# Usage
cache = LFUCache.new(3)

cache.put(1, 10)  # freq: {1:1}
cache.put(2, 20)  # freq: {1:1, 2:1}
cache.put(3, 30)  # freq: {1:1, 2:1, 3:1}

cache.get(1)       # freq: {1:2, 2:1, 3:1}
cache.get(1)       # freq: {1:3, 2:1, 3:1}
cache.get(2)       # freq: {1:3, 2:2, 3:1}

cache.put(4, 40)   # evicts key 3 (freq=1, LFU)
puts cache.get(3)  # -1 (evicted)
puts cache.get(4)  # 40
```

**LRU vs LFU comparison:**

| Feature | LRU Cache | LFU Cache |
|---------|-----------|-----------|
| Eviction policy | Least recently used | Least frequently used |
| Data structures | Hash + Doubly Linked List | Hash + Frequency Map + Lists |
| Time complexity | O(1) all operations | O(1) all operations* |
| Best for | General caching, recency matters | Frequency matters (popular items stay) |
| Weakness | Scan pollution (one-time access evicts useful items) | Frequency accumulation (old popular items never evicted) |

\* O(1) with linked lists for frequency buckets; the simplified array-based version above has O(n) for `touch` due to `Array#delete`.

---

### Consistent Hashing

**Consistent hashing** is used in distributed systems to distribute data across multiple servers while minimizing redistribution when servers are added or removed.

**The problem with simple hashing:**

```
Simple approach: server = hash(key) % num_servers

If num_servers changes (server added/removed):
  - Almost ALL keys get remapped to different servers
  - Massive cache invalidation and data migration

Example: 3 servers → 4 servers
  hash("user_1") % 3 = 1  →  hash("user_1") % 4 = 2  (MOVED!)
  hash("user_2") % 3 = 0  →  hash("user_2") % 4 = 0  (stayed)
  hash("user_3") % 3 = 2  →  hash("user_3") % 4 = 1  (MOVED!)
  ~75% of keys are remapped!
```

**Consistent hashing solution:**

```ruby
require 'digest'
require 'set'

class ConsistentHash
  def initialize(virtual_nodes: 150)
    @ring = {}           # hash_position → server_name (sorted by position)
    @sorted_keys = []    # sorted array of hash positions for binary search
    @virtual_nodes = virtual_nodes
  end

  # Add a server to the ring
  def add_server(server)
    @virtual_nodes.times do |i|
      virtual_key = "#{server}##{i}"
      hash_val = hash_key(virtual_key)
      @ring[hash_val] = server
    end
    @sorted_keys = @ring.keys.sort
    puts "Added server: #{server} (#{@virtual_nodes} virtual nodes)"
  end

  # Remove a server from the ring
  def remove_server(server)
    @virtual_nodes.times do |i|
      virtual_key = "#{server}##{i}"
      hash_val = hash_key(virtual_key)
      @ring.delete(hash_val)
    end
    @sorted_keys = @ring.keys.sort
    puts "Removed server: #{server}"
  end

  # Find which server a key maps to
  def get_server(key)
    return nil if @ring.empty?

    hash_val = hash_key(key)

    # Find the first server position >= hash_val (clockwise on the ring)
    idx = @sorted_keys.bsearch_index { |k| k >= hash_val }
    idx = 0 if idx.nil?  # wrap around the ring

    @ring[@sorted_keys[idx]]
  end

  def server_count
    @ring.values.uniq.size
  end

  private

  def hash_key(key)
    # Use MD5 for consistent hashing (not for security)
    Digest::MD5.hexdigest(key).to_i(16)
  end
end

# Usage
ch = ConsistentHash.new(virtual_nodes: 100)

ch.add_server("server-1")
ch.add_server("server-2")
ch.add_server("server-3")

# Route keys to servers
keys = %w[user_1 user_2 user_3 user_4 user_5]
keys.each { |key| puts "#{key} → #{ch.get_server(key)}" }

puts "\n--- Adding server-4 ---\n"
ch.add_server("server-4")

# Most keys stay on the same server!
keys.each { |key| puts "#{key} → #{ch.get_server(key)}" }
# Only ~1/4 of keys move to the new server (instead of ~75% with simple hashing)
```

**How consistent hashing works:**

```
Ring (0 to 2^128 with MD5):

        S1          S2
         \         /
    ------●-------●------
   |                      |
   |     Hash Ring        |
   |                      |
    ------●-------●------
         /         \
        S3          S1 (virtual)

1. Servers are placed on the ring at positions determined by their hash
2. Each key is hashed and placed on the ring
3. The key is assigned to the first server found clockwise
4. Virtual nodes: each server gets multiple positions for better distribution

When a server is added/removed:
  - Only keys between the new/removed server and its predecessor are affected
  - ~1/N of keys are remapped (N = number of servers)
```

**When to use consistent hashing:**
- **Distributed caches** — Redis Cluster, Memcached
- **Load balancers** — route requests to servers
- **Database sharding** — distribute data across shards
- **CDN** — route content to edge servers
- **Distributed hash tables** — peer-to-peer systems

---

### Rate Limiting Algorithms (Token Bucket, Sliding Window)

Rate limiting controls how many requests a user or client can make in a given time period. It protects systems from abuse, ensures fair usage, and prevents overload.

**Token Bucket Algorithm:**

The token bucket is the most widely used rate limiting algorithm. Tokens are added to a bucket at a fixed rate. Each request consumes a token. If the bucket is empty, the request is rejected.

```ruby
class TokenBucket
  def initialize(max_tokens, refill_rate)
    @max_tokens = max_tokens          # bucket capacity
    @refill_rate = refill_rate        # tokens per second
    @current_tokens = max_tokens.to_f
    @last_refill = Time.now
    @mutex = Mutex.new
  end

  # Try to consume a token — returns true if allowed
  def allow_request?(tokens = 1)
    @mutex.synchronize do
      refill
      if @current_tokens >= tokens
        @current_tokens -= tokens
        true   # request allowed
      else
        false  # rate limited
      end
    end
  end

  def available_tokens
    @mutex.synchronize do
      refill
      @current_tokens
    end
  end

  private

  def refill
    now = Time.now
    elapsed = now - @last_refill
    @current_tokens = [@max_tokens.to_f, @current_tokens + elapsed * @refill_rate].min
    @last_refill = now
  end
end

# Usage
# 10 tokens max, refill at 2 tokens/second
limiter = TokenBucket.new(10, 2.0)

15.times do |i|
  if limiter.allow_request?
    puts "Request #{i}: ALLOWED"
  else
    puts "Request #{i}: RATE LIMITED"
  end
end
# First 10 requests allowed (bucket starts full)
# Requests 11-14 rate limited (bucket empty, refill is slow)

# Wait for tokens to refill
sleep(3)  # +6 tokens

8.times do |i|
  if limiter.allow_request?
    puts "After wait, request #{i}: ALLOWED"
  else
    puts "After wait, request #{i}: RATE LIMITED"
  end
end
```

**Token Bucket properties:**
- Allows **bursts** up to bucket capacity
- Smooth rate limiting over time
- Simple to implement and understand
- Used by: AWS API Gateway, Stripe, Rack::Attack

**Sliding Window Counter:**

Divides time into fixed windows and counts requests per window. The "sliding" part interpolates between the current and previous window for smoother limiting.

```ruby
class SlidingWindowCounter
  def initialize(max_requests, window_seconds)
    @max_requests = max_requests
    @window_size = window_seconds
    @current_count = 0
    @previous_count = 0
    @window_start = Time.now
    @mutex = Mutex.new
  end

  def allow_request?
    @mutex.synchronize do
      now = Time.now
      elapsed = now - @window_start

      # Check if we've moved to a new window
      if elapsed >= @window_size
        windows_passed = (elapsed / @window_size).to_i
        if windows_passed >= 2
          @previous_count = 0
          @current_count = 0
        else
          @previous_count = @current_count
          @current_count = 0
        end
        @window_start += @window_size * windows_passed
      end

      # Calculate weighted count (sliding window approximation)
      time_in_window = now - @window_start
      weight = time_in_window / @window_size
      estimated_count = @previous_count * (1.0 - weight) + @current_count

      if estimated_count < @max_requests
        @current_count += 1
        true
      else
        false
      end
    end
  end
end
```

**Sliding Window Log — exact counting:**

Stores the timestamp of every request. More accurate but uses more memory.

```ruby
class SlidingWindowLog
  def initialize(max_requests, window_seconds)
    @max_requests = max_requests
    @window_size = window_seconds
    @request_log = []  # timestamps
    @mutex = Mutex.new
  end

  def allow_request?
    @mutex.synchronize do
      now = Time.now
      window_start = now - @window_size

      # Remove expired entries
      @request_log.shift while !@request_log.empty? && @request_log.first <= window_start

      if @request_log.size < @max_requests
        @request_log.push(now)
        true
      else
        false
      end
    end
  end
end
```

**Fixed Window Counter — simplest approach:**

```ruby
class FixedWindowCounter
  def initialize(max_requests, window_seconds)
    @max_requests = max_requests
    @window_size = window_seconds
    @count = 0
    @window_start = Time.now
    @mutex = Mutex.new
  end

  def allow_request?
    @mutex.synchronize do
      now = Time.now

      if now - @window_start >= @window_size
        # New window
        @count = 0
        @window_start = now
      end

      if @count < @max_requests
        @count += 1
        true
      else
        false
      end
    end
  end
end
```

**Per-user rate limiter:**

```ruby
class PerUserRateLimiter
  def initialize(max_tokens, refill_rate)
    @max_tokens = max_tokens
    @refill_rate = refill_rate
    @user_buckets = {}
    @mutex = Mutex.new
  end

  def allow_request?(user_id)
    @mutex.synchronize do
      @user_buckets[user_id] ||= TokenBucket.new(@max_tokens, @refill_rate)
    end
    @user_buckets[user_id].allow_request?
  end
end

# Usage
limiter = PerUserRateLimiter.new(100, 10.0)  # 100 burst, 10 req/sec per user

limiter.allow_request?("user_123")  # each user has their own bucket
limiter.allow_request?("user_456")
```

**Rate limiting algorithm comparison:**

| Algorithm | Accuracy | Memory | Burst Handling | Complexity |
|-----------|----------|--------|---------------|------------|
| Token Bucket | Good | O(1) per user | Allows controlled bursts | Simple |
| Sliding Window Log | Exact | O(n) per user | No bursts | Medium |
| Sliding Window Counter | Approximate | O(1) per user | Smoothed | Medium |
| Fixed Window Counter | Approximate | O(1) per user | Burst at window edges | Simplest |
| Leaky Bucket | Good | O(1) per user | Smooths out bursts | Simple |

---


## Summary & Key Takeaways

| Topic | Core Concept | Key Data Structures / Algorithms |
|-------|-------------|----------------------------------|
| Arrays | Dynamic, ordered, O(1) random access | `Array`, `push`, `pop`, `sort_by`, `bsearch` |
| Linked Lists | O(1) insert/delete with reference | Custom doubly linked list |
| Stacks & Queues | LIFO / FIFO access patterns | `Array` (push/pop), `Thread::Queue`, custom queue |
| Hashes & Sets | O(1) average lookup by key | `Hash`, `Set` |
| Trees | Hierarchical data, O(log n) operations | BST, AVL, `SortedSet` / `rbtree` gem |
| Heaps | Priority-based access, O(1) top | Custom `MinHeap` / `MaxHeap` / `PriorityQueue` |
| Graphs | Relationships and connections | Adjacency list/hash, BFS, DFS |
| Tries | Prefix-based string operations | Custom trie for autocomplete |
| Sorting | Ordering data for display or search | `sort`, `sort_by`, `max_by(k)`, `min_by(k)` |
| Searching | Finding elements efficiently | `bsearch`, hash lookup, BFS/DFS |
| Shortest Path | Optimal routing | Dijkstra, Bellman-Ford, BFS |
| Hashing | Fixed-size mapping for caching/dedup | `hash`, `Digest::MD5`, Bloom filters |
| LRU Cache | Evict least recently used | Hash + Doubly Linked List |
| LFU Cache | Evict least frequently used | Hash + Frequency Map + Lists |
| Consistent Hashing | Distributed key routing | Hash ring with virtual nodes |
| Rate Limiting | Control request throughput | Token Bucket, Sliding Window |

---

## Interview Tips for Module 10

1. **"Implement an LRU Cache with O(1) get and put."** Use a `Hash` for O(1) key lookup and a Doubly Linked List for O(1) reordering. The head of the list is MRU, the tail is LRU. On `get()`, move the node to the front. On `put()`, add to front and evict from tail if full. This is the single most asked LLD data structure question. Ruby's `Hash` preserves insertion order, but you still need a linked list for O(1) move-to-front.

2. **"When would you use a Hash vs a sorted structure?"** `Hash` for O(1) average lookup when you don't need ordering. A sorted array with `bsearch` or a BST/`SortedSet` for O(log n) lookup when you need sorted keys, range queries, or ordered iteration. Example: use `Hash` for a cache, sorted structure for a leaderboard that needs range queries.

3. **"How does consistent hashing work?"** Servers and keys are placed on a hash ring. Each key is assigned to the first server found clockwise. Virtual nodes improve distribution. When a server is added/removed, only ~1/N of keys are remapped (vs ~100% with simple modulo hashing). Used in distributed caches, load balancers, and database sharding.

4. **"Implement a rate limiter."** Token Bucket is the most common answer: tokens refill at a fixed rate, each request consumes a token, requests are rejected when the bucket is empty. Allows controlled bursts. For per-user limiting, maintain a separate bucket per user ID in a `Hash`. Discuss trade-offs with sliding window (more accurate, more memory). In Ruby, use `Mutex` for thread safety. Gems like `Rack::Attack` implement this for Rails apps.

5. **"What data structure would you use for autocomplete?"** A Trie (prefix tree). Insert all words into the trie. For autocomplete, traverse to the prefix node, then DFS to collect all words below it. Time: O(L) to reach the prefix node + O(K) to collect K results. Alternative: sorted array with `bsearch` on prefix, but trie is more efficient for dynamic insertions.

6. **"How would you design a leaderboard?"** For top-K queries: use a min-heap of size K — O(n log K) to build, O(1) to get minimum. For dynamic updates: use a BST or sorted structure for O(log n) insert/delete/rank queries. For very large scale: use Redis ZSET (sorted set) which combines a skip list and hash map. In Ruby, `max_by(k)` is a quick way to get top-K from an array.

7. **"Explain BFS vs DFS and when to use each."** BFS explores level by level using a queue — finds shortest path in unweighted graphs, good for "nearest" queries (friends of friends). DFS explores depth-first using a stack/recursion — good for cycle detection, topological sorting, and exhaustive search. BFS uses more memory (stores entire level), DFS uses less (stores one path).

8. **"How does an LFU Cache differ from LRU?"** LRU evicts the least recently accessed item. LFU evicts the least frequently accessed item (with LRU as tiebreaker). LFU is better when access frequency matters (popular items should stay cached). LRU is simpler and better for general workloads. Both achieve O(1) for all operations with the right data structures.

9. **"What is a Bloom filter and when would you use it?"** A space-efficient probabilistic data structure that can tell you "definitely not in set" or "probably in set." Uses multiple hash functions and a bit array. False positives are possible, false negatives are not. Use for: checking if a username is taken (before hitting the database), spam filtering, cache lookup optimization, duplicate detection in streams.

10. **"How would you detect a cycle in a dependency graph?"** Use DFS with a recursion stack (`Set` of nodes in the current path). If you visit a node that's already in the recursion stack, there's a cycle. For directed graphs, maintain three states: unvisited, in-progress (in recursion stack), and completed. A back edge to an in-progress node indicates a cycle. Used in: build systems (Rake), spreadsheet formula evaluation, package managers (Bundler).

11. **"What is the time complexity of Dijkstra's algorithm?"** O((V + E) log V) with a binary heap priority queue. O(V²) with a simple array scan (better for dense graphs). Ruby doesn't have a built-in priority queue, so you'd implement a `MinHeap` or use a gem. Dijkstra doesn't work with negative edge weights — use Bellman-Ford O(V * E) for that.

12. **"How would you implement a priority task scheduler?"** Use a max-heap (priority queue) where each task has a priority value. The highest priority task is always at the top — O(1) to peek, O(log n) to insert or extract. For tasks with the same priority, use a secondary ordering (timestamp for FIFO among equal priorities). Implement a custom `PriorityQueue` class with a configurable comparator block.

13. **"Ruby doesn't have built-in heaps or priority queues. How do you handle that?"** Implement your own `MinHeap`/`MaxHeap` using an array with `heapify_up` and `heapify_down` operations. For production code, use gems like `algorithms` (provides `Containers::PriorityQueue`) or `rb_heap`. The array-based heap implementation is straightforward and commonly expected in interviews.