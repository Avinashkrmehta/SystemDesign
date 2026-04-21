# Module 10: Data Structures & Algorithms for LLD

> In Low-Level Design, choosing the right data structure is as important as choosing the right design pattern. Data structures determine the time and space complexity of your operations, and algorithms built on top of them power features like caching, routing, rate limiting, and search. This module covers the essential data structures and algorithms you need to know for LLD interviews and real-world system component design in C++.

---

## 10.1 Essential Data Structures

> Every LLD problem maps to a set of operations — insert, delete, search, update, iterate. The data structure you choose determines how efficiently each operation runs. This section covers the core data structures, their C++ implementations, time complexities, and when to use each in LLD contexts.

---

### Arrays and Vectors

An **array** is a contiguous block of memory storing elements of the same type. In C++, `std::vector` is the go-to dynamic array — it handles resizing automatically and is the most commonly used container.

**std::vector — dynamic array:**

```cpp
#include <vector>
#include <iostream>
#include <algorithm>
using namespace std;

int main() {
    // Creation
    vector<int> v1;                    // empty vector
    vector<int> v2(10, 0);            // 10 elements, all 0
    vector<int> v3 = {1, 2, 3, 4, 5}; // initializer list

    // Adding elements
    v1.push_back(10);      // O(1) amortized — adds to the end
    v1.push_back(20);
    v1.emplace_back(30);   // constructs in-place — avoids copy

    // Accessing elements
    int first = v3[0];      // O(1) — no bounds checking
    int safe = v3.at(0);    // O(1) — throws std::out_of_range if invalid
    int last = v3.back();   // O(1) — last element
    int front = v3.front(); // O(1) — first element

    // Size and capacity
    cout << "Size: " << v3.size() << endl;         // 5 — number of elements
    cout << "Capacity: " << v3.capacity() << endl;  // >= 5 — allocated space
    v3.reserve(100);  // pre-allocate space for 100 elements — avoids reallocations
    v3.shrink_to_fit(); // release unused capacity

    // Inserting and erasing (middle operations are O(n))
    v3.insert(v3.begin() + 2, 99);  // insert 99 at index 2 — O(n) shift
    v3.erase(v3.begin() + 2);       // erase element at index 2 — O(n) shift

    // Iterating
    for (int x : v3) cout << x << " ";
    cout << endl;

    // Sorting
    sort(v3.begin(), v3.end());           // ascending
    sort(v3.begin(), v3.end(), greater<>()); // descending

    // Searching
    auto it = find(v3.begin(), v3.end(), 3);
    if (it != v3.end()) {
        cout << "Found 3 at index " << distance(v3.begin(), it) << endl;
    }

    // Binary search (requires sorted vector)
    sort(v3.begin(), v3.end());
    bool exists = binary_search(v3.begin(), v3.end(), 3);  // O(log n)
}
```

**How vector resizing works internally:**

```
Initial:  [1, 2, 3] capacity=3, size=3

push_back(4):
  1. capacity == size → need to grow
  2. Allocate new array of capacity * 2 = 6
  3. Copy/move all elements to new array
  4. Add new element
  5. Deallocate old array

Result:   [1, 2, 3, 4, _, _] capacity=6, size=4

This is why push_back is O(1) AMORTIZED — most calls are O(1),
but occasionally O(n) when reallocation happens.
```

**Vector time complexity:**

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `push_back` | O(1) amortized | O(n) when reallocation needed |
| `pop_back` | O(1) | |
| `operator[]` / `at()` | O(1) | Random access |
| `insert` (middle) | O(n) | Shifts elements |
| `erase` (middle) | O(n) | Shifts elements |
| `find` | O(n) | Linear search |
| `binary_search` | O(log n) | Requires sorted vector |
| `sort` | O(n log n) | |

**When to use vector in LLD:**
- Default container for ordered collections (product lists, user lists)
- When you need random access by index (leaderboard positions)
- When most operations are at the end (stack-like behavior)
- When you need cache-friendly iteration (contiguous memory)

---

### Linked Lists

A **linked list** stores elements in nodes, where each node contains data and a pointer to the next (and optionally previous) node. Unlike vectors, linked lists allow O(1) insertion/deletion at any position if you have a pointer to the node.

**std::list — doubly linked list:**

```cpp
#include <list>
#include <iostream>
using namespace std;

int main() {
    list<int> lst = {1, 2, 3, 4, 5};

    // O(1) insertion at front and back
    lst.push_front(0);
    lst.push_back(6);

    // O(1) insertion at any position (given an iterator)
    auto it = lst.begin();
    advance(it, 3);          // move iterator to position 3 — O(n)
    lst.insert(it, 99);      // insert 99 before position 3 — O(1)

    // O(1) deletion (given an iterator)
    lst.erase(it);           // remove element at iterator — O(1)

    // O(1) splicing — move elements between lists without copying
    list<int> other = {10, 20, 30};
    lst.splice(lst.end(), other);  // move all of 'other' to end of 'lst'
    // 'other' is now empty

    // Iterating (forward and backward)
    for (int x : lst) cout << x << " ";
    cout << endl;

    // Reverse iteration
    for (auto rit = lst.rbegin(); rit != lst.rend(); ++rit) {
        cout << *rit << " ";
    }
    cout << endl;

    // Remove by value
    lst.remove(99);  // removes ALL elements with value 99

    // Sort (list has its own sort — can't use std::sort because no random access)
    lst.sort();

    // Unique (remove consecutive duplicates — sort first)
    lst.unique();
}
```

**Custom linked list (commonly asked in interviews):**

```cpp
template <typename T>
class LinkedList {
    struct Node {
        T data;
        Node* next;
        Node* prev;
        Node(const T& d) : data(d), next(nullptr), prev(nullptr) {}
    };

    Node* head = nullptr;
    Node* tail = nullptr;
    size_t count = 0;

public:
    ~LinkedList() {
        while (head) {
            Node* temp = head;
            head = head->next;
            delete temp;
        }
    }

    // O(1) — push to front
    void push_front(const T& data) {
        Node* node = new Node(data);
        node->next = head;
        if (head) head->prev = node;
        head = node;
        if (!tail) tail = node;
        count++;
    }

    // O(1) — push to back
    void push_back(const T& data) {
        Node* node = new Node(data);
        node->prev = tail;
        if (tail) tail->next = node;
        tail = node;
        if (!head) head = node;
        count++;
    }

    // O(1) — remove a specific node (given pointer)
    void remove(Node* node) {
        if (!node) return;
        if (node->prev) node->prev->next = node->next;
        else head = node->next;  // removing head

        if (node->next) node->next->prev = node->prev;
        else tail = node->prev;  // removing tail

        delete node;
        count--;
    }

    // O(1) — move a node to the front (used in LRU Cache!)
    void move_to_front(Node* node) {
        if (node == head) return;  // already at front

        // Detach from current position
        if (node->prev) node->prev->next = node->next;
        if (node->next) node->next->prev = node->prev;
        if (node == tail) tail = node->prev;

        // Attach at front
        node->prev = nullptr;
        node->next = head;
        if (head) head->prev = node;
        head = node;
    }

    Node* get_head() { return head; }
    Node* get_tail() { return tail; }
    size_t size() const { return count; }
};
```

**Linked list time complexity:**

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `push_front` / `push_back` | O(1) | |
| `pop_front` / `pop_back` | O(1) | Doubly linked |
| Insert at position (with iterator) | O(1) | But finding the position is O(n) |
| Remove node (with pointer) | O(1) | Key advantage over vector |
| Search by value | O(n) | No random access |
| Access by index | O(n) | Must traverse |

**When to use linked lists in LLD:**
- **LRU Cache** — O(1) move-to-front and remove-from-back
- When you need frequent insertions/deletions in the middle
- When you need stable iterators (inserting/deleting doesn't invalidate other iterators)
- Implementing queues, deques, or custom ordered structures

---


### Stacks, Queues, and Deques

These are **adapter containers** — they restrict how you interact with the underlying data to enforce specific access patterns.

**std::stack — LIFO (Last In, First Out):**

```cpp
#include <stack>
#include <iostream>
using namespace std;

int main() {
    stack<int> s;

    // Push elements — O(1)
    s.push(10);
    s.push(20);
    s.push(30);

    // Access top element — O(1)
    cout << "Top: " << s.top() << endl;  // 30

    // Pop (remove top) — O(1)
    s.pop();
    cout << "Top after pop: " << s.top() << endl;  // 20

    // Size and empty check
    cout << "Size: " << s.size() << endl;
    cout << "Empty: " << s.empty() << endl;

    // Process all elements
    while (!s.empty()) {
        cout << s.top() << " ";
        s.pop();
    }
}
```

**LLD use cases for stack:**
- **Undo/Redo** in text editors (Command pattern + stack)
- **Expression evaluation** (postfix, infix to postfix conversion)
- **Bracket matching** (code editors, compilers)
- **DFS traversal** (iterative depth-first search)
- **Call stack simulation** (function call tracking)
- **Browser back button** (navigation history)

```cpp
// Example: Undo system using stack
class UndoManager {
    stack<string> undo_stack;
    stack<string> redo_stack;

public:
    void execute(const string& action) {
        undo_stack.push(action);
        // Clear redo stack when a new action is performed
        while (!redo_stack.empty()) redo_stack.pop();
        cout << "Executed: " << action << endl;
    }

    void undo() {
        if (undo_stack.empty()) {
            cout << "Nothing to undo" << endl;
            return;
        }
        string action = undo_stack.top();
        undo_stack.pop();
        redo_stack.push(action);
        cout << "Undone: " << action << endl;
    }

    void redo() {
        if (redo_stack.empty()) {
            cout << "Nothing to redo" << endl;
            return;
        }
        string action = redo_stack.top();
        redo_stack.pop();
        undo_stack.push(action);
        cout << "Redone: " << action << endl;
    }
};
```

**std::queue — FIFO (First In, First Out):**

```cpp
#include <queue>
#include <iostream>
using namespace std;

int main() {
    queue<int> q;

    // Enqueue — O(1)
    q.push(10);
    q.push(20);
    q.push(30);

    // Access front and back — O(1)
    cout << "Front: " << q.front() << endl;  // 10
    cout << "Back: " << q.back() << endl;    // 30

    // Dequeue (remove front) — O(1)
    q.pop();
    cout << "Front after pop: " << q.front() << endl;  // 20

    // Process all elements
    while (!q.empty()) {
        cout << q.front() << " ";
        q.pop();
    }
}
```

**LLD use cases for queue:**
- **Task scheduling** (FCFS — first come, first served)
- **BFS traversal** (breadth-first search in graphs/trees)
- **Message queues** (producer-consumer pattern)
- **Print job queue** (printer spooler)
- **Request handling** (web server request queue)
- **Order processing** (e-commerce order queue)

```cpp
// Example: Simple task scheduler using queue
struct Task {
    int id;
    string description;
    int priority;
};

class TaskScheduler {
    queue<Task> task_queue;

public:
    void submit(const Task& task) {
        task_queue.push(task);
        cout << "Task " << task.id << " submitted: " << task.description << endl;
    }

    void process_next() {
        if (task_queue.empty()) {
            cout << "No tasks to process" << endl;
            return;
        }
        Task task = task_queue.front();
        task_queue.pop();
        cout << "Processing task " << task.id << ": " << task.description << endl;
    }

    size_t pending() const { return task_queue.size(); }
};
```

**std::deque — Double-Ended Queue:**

A `deque` allows efficient insertion and removal at both ends. Unlike `vector`, it doesn't require contiguous memory — it uses a sequence of fixed-size arrays (chunks).

```cpp
#include <deque>
#include <iostream>
using namespace std;

int main() {
    deque<int> dq = {2, 3, 4};

    // O(1) operations at both ends
    dq.push_front(1);   // [1, 2, 3, 4]
    dq.push_back(5);    // [1, 2, 3, 4, 5]
    dq.pop_front();     // [2, 3, 4, 5]
    dq.pop_back();      // [2, 3, 4]

    // O(1) random access (like vector)
    cout << "Element at index 1: " << dq[1] << endl;  // 3

    // Iterating
    for (int x : dq) cout << x << " ";
    cout << endl;
}
```

**LLD use cases for deque:**
- **Sliding window** algorithms (add to back, remove from front)
- **Work stealing** queues (owner pops from back, thieves steal from front)
- **Palindrome checking** (compare front and back)
- **Undo with limited history** (push back, pop front when full)

**Comparison:**

| Feature | `stack` | `queue` | `deque` |
|---------|---------|---------|---------|
| Push front | ❌ | ❌ | O(1) |
| Push back | O(1) | O(1) | O(1) |
| Pop front | ❌ | O(1) | O(1) |
| Pop back | O(1) | ❌ | O(1) |
| Random access | ❌ | ❌ | O(1) |
| Access pattern | LIFO | FIFO | Both ends |
| Underlying default | `deque` | `deque` | Chunked array |

---

### Hash Maps and Hash Sets

**Hash-based containers** provide average O(1) lookup, insertion, and deletion by mapping keys to array indices using a hash function. They are the most frequently used data structures in LLD.

**std::unordered_map — hash map:**

```cpp
#include <unordered_map>
#include <iostream>
using namespace std;

int main() {
    unordered_map<string, int> scores;

    // Insert — O(1) average
    scores["Alice"] = 95;
    scores["Bob"] = 87;
    scores.insert({"Charlie", 92});
    scores.emplace("Diana", 88);

    // Lookup — O(1) average
    cout << "Alice's score: " << scores["Alice"] << endl;

    // Safe lookup (doesn't insert if key missing)
    auto it = scores.find("Eve");
    if (it != scores.end()) {
        cout << "Eve's score: " << it->second << endl;
    } else {
        cout << "Eve not found" << endl;
    }

    // Check existence — O(1) average
    if (scores.count("Bob") > 0) {
        cout << "Bob exists" << endl;
    }

    // C++20: contains()
    // if (scores.contains("Bob")) { ... }

    // Erase — O(1) average
    scores.erase("Charlie");

    // Iterate (order is NOT guaranteed)
    for (const auto& [name, score] : scores) {
        cout << name << ": " << score << endl;
    }

    // Size and bucket info
    cout << "Size: " << scores.size() << endl;
    cout << "Bucket count: " << scores.bucket_count() << endl;
    cout << "Load factor: " << scores.load_factor() << endl;
    cout << "Max load factor: " << scores.max_load_factor() << endl;
}
```

**std::unordered_set — hash set:**

```cpp
#include <unordered_set>
using namespace std;

int main() {
    unordered_set<string> tags;

    // Insert — O(1) average
    tags.insert("cpp");
    tags.insert("design-patterns");
    tags.insert("lld");
    tags.insert("cpp");  // duplicate — ignored

    // Lookup — O(1) average
    if (tags.count("cpp") > 0) {
        cout << "Tag 'cpp' exists" << endl;
    }

    // Erase — O(1) average
    tags.erase("lld");

    cout << "Tags: ";
    for (const auto& tag : tags) cout << tag << " ";
    cout << endl;
}
```

**Custom hash function (for user-defined types):**

```cpp
struct Point {
    int x, y;

    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
};

// Custom hash function
struct PointHash {
    size_t operator()(const Point& p) const {
        // Combine hashes of individual fields
        size_t h1 = hash<int>{}(p.x);
        size_t h2 = hash<int>{}(p.y);
        return h1 ^ (h2 << 1);  // simple hash combining
    }
};

// Usage
unordered_map<Point, string, PointHash> point_labels;
point_labels[{0, 0}] = "origin";
point_labels[{1, 2}] = "point A";
```

**How hash maps work internally:**

```
Hash Function: key → hash value → bucket index

Buckets (array of linked lists):
  [0] → nullptr
  [1] → ("Alice", 95) → nullptr
  [2] → ("Bob", 87) → ("Diana", 88) → nullptr   ← collision! (chaining)
  [3] → nullptr
  [4] → ("Charlie", 92) → nullptr
  ...

Collision resolution:
  - Chaining (linked list per bucket) — used by std::unordered_map
  - Open addressing (linear probing, quadratic probing, double hashing)

Load factor = size / bucket_count
When load factor exceeds max_load_factor (default 1.0):
  → Rehash: allocate more buckets, re-insert all elements — O(n)
```

**Hash map time complexity:**

| Operation | Average | Worst Case | Notes |
|-----------|---------|------------|-------|
| Insert | O(1) | O(n) | Worst case during rehash |
| Lookup | O(1) | O(n) | Worst case: all keys in one bucket |
| Delete | O(1) | O(n) | Same as lookup |
| Iteration | O(n) | O(n + bucket_count) | |

**std::map vs std::unordered_map:**

| Feature | `std::map` | `std::unordered_map` |
|---------|-----------|---------------------|
| Underlying structure | Red-Black Tree | Hash Table |
| Ordering | Sorted by key | No ordering |
| Lookup | O(log n) | O(1) average |
| Insert | O(log n) | O(1) average |
| Memory | Less overhead | More overhead (buckets) |
| Iterator invalidation | Stable | Invalidated on rehash |
| Use when | Need sorted order, range queries | Need fast lookup, no ordering needed |

**When to use hash maps in LLD:**
- **Caching** — key-value store for cached data
- **Deduplication** — track seen items
- **Counting** — frequency maps (word count, vote tallying)
- **Indexing** — map IDs to objects (user ID → User object)
- **Configuration** — key-value settings
- **LRU Cache** — hash map + doubly linked list for O(1) operations

---


### Trees (Binary Tree, BST, AVL, Red-Black)

Trees are hierarchical data structures where each node has a parent (except the root) and zero or more children. They are fundamental to databases (B-Trees), file systems (directory trees), expression parsing (AST), and many LLD problems.

**Binary Tree — basic structure:**

```cpp
struct TreeNode {
    int data;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int val) : data(val), left(nullptr), right(nullptr) {}
};

// Tree traversals — essential for interviews
class BinaryTree {
public:
    // Inorder: Left → Root → Right (gives sorted order for BST)
    void inorder(TreeNode* node) {
        if (!node) return;
        inorder(node->left);
        cout << node->data << " ";
        inorder(node->right);
    }

    // Preorder: Root → Left → Right (useful for copying/serializing trees)
    void preorder(TreeNode* node) {
        if (!node) return;
        cout << node->data << " ";
        preorder(node->left);
        preorder(node->right);
    }

    // Postorder: Left → Right → Root (useful for deletion, expression evaluation)
    void postorder(TreeNode* node) {
        if (!node) return;
        postorder(node->left);
        postorder(node->right);
        cout << node->data << " ";
    }

    // Level-order (BFS): level by level using a queue
    void level_order(TreeNode* root) {
        if (!root) return;
        queue<TreeNode*> q;
        q.push(root);

        while (!q.empty()) {
            int level_size = q.size();
            for (int i = 0; i < level_size; i++) {
                TreeNode* node = q.front();
                q.pop();
                cout << node->data << " ";
                if (node->left) q.push(node->left);
                if (node->right) q.push(node->right);
            }
            cout << endl;  // new line per level
        }
    }
};
```

**Binary Search Tree (BST):**

A BST maintains the invariant: for every node, all values in the left subtree are smaller, and all values in the right subtree are larger. This enables O(log n) search, insert, and delete on average.

```cpp
class BST {
    struct Node {
        int key;
        Node* left;
        Node* right;
        Node(int k) : key(k), left(nullptr), right(nullptr) {}
    };

    Node* root = nullptr;

    // Insert — O(log n) average, O(n) worst (skewed tree)
    Node* insert(Node* node, int key) {
        if (!node) return new Node(key);
        if (key < node->key)
            node->left = insert(node->left, key);
        else if (key > node->key)
            node->right = insert(node->right, key);
        // Duplicate keys ignored
        return node;
    }

    // Search — O(log n) average, O(n) worst
    Node* search(Node* node, int key) {
        if (!node || node->key == key) return node;
        if (key < node->key) return search(node->left, key);
        return search(node->right, key);
    }

    // Find minimum — leftmost node
    Node* find_min(Node* node) {
        while (node->left) node = node->left;
        return node;
    }

    // Delete — O(log n) average
    Node* remove(Node* node, int key) {
        if (!node) return nullptr;

        if (key < node->key) {
            node->left = remove(node->left, key);
        } else if (key > node->key) {
            node->right = remove(node->right, key);
        } else {
            // Found the node to delete
            // Case 1: Leaf node
            if (!node->left && !node->right) {
                delete node;
                return nullptr;
            }
            // Case 2: One child
            if (!node->left) {
                Node* temp = node->right;
                delete node;
                return temp;
            }
            if (!node->right) {
                Node* temp = node->left;
                delete node;
                return temp;
            }
            // Case 3: Two children — replace with inorder successor
            Node* successor = find_min(node->right);
            node->key = successor->key;
            node->right = remove(node->right, successor->key);
        }
        return node;
    }

public:
    void insert(int key) { root = insert(root, key); }
    bool search(int key) { return search(root, key) != nullptr; }
    void remove(int key) { root = remove(root, key); }

    // Inorder traversal gives sorted output
    void print_sorted(Node* node) {
        if (!node) return;
        print_sorted(node->left);
        cout << node->key << " ";
        print_sorted(node->right);
    }
    void print_sorted() { print_sorted(root); cout << endl; }
};
```

**AVL Tree — self-balancing BST:**

An AVL tree maintains the property that for every node, the heights of the left and right subtrees differ by at most 1. This guarantees O(log n) for all operations.

```cpp
class AVLTree {
    struct Node {
        int key;
        Node* left;
        Node* right;
        int height;
        Node(int k) : key(k), left(nullptr), right(nullptr), height(1) {}
    };

    Node* root = nullptr;

    int height(Node* n) { return n ? n->height : 0; }

    int balance_factor(Node* n) {
        return n ? height(n->left) - height(n->right) : 0;
    }

    void update_height(Node* n) {
        n->height = 1 + max(height(n->left), height(n->right));
    }

    // Right rotation (for left-heavy imbalance)
    //       y              x
    //      / \            / \
    //     x   C   →     A   y
    //    / \                / \
    //   A   B              B   C
    Node* rotate_right(Node* y) {
        Node* x = y->left;
        Node* B = x->right;

        x->right = y;
        y->left = B;

        update_height(y);
        update_height(x);
        return x;  // new root of this subtree
    }

    // Left rotation (for right-heavy imbalance)
    //     x                y
    //    / \              / \
    //   A   y     →     x   C
    //      / \         / \
    //     B   C       A   B
    Node* rotate_left(Node* x) {
        Node* y = x->right;
        Node* B = y->left;

        y->left = x;
        x->right = B;

        update_height(x);
        update_height(y);
        return y;
    }

    // Rebalance after insertion or deletion
    Node* rebalance(Node* node) {
        update_height(node);
        int bf = balance_factor(node);

        // Left-heavy
        if (bf > 1) {
            if (balance_factor(node->left) < 0) {
                node->left = rotate_left(node->left);  // Left-Right case
            }
            return rotate_right(node);  // Left-Left case
        }

        // Right-heavy
        if (bf < -1) {
            if (balance_factor(node->right) > 0) {
                node->right = rotate_right(node->right);  // Right-Left case
            }
            return rotate_left(node);  // Right-Right case
        }

        return node;  // balanced
    }

    Node* insert(Node* node, int key) {
        if (!node) return new Node(key);
        if (key < node->key)
            node->left = insert(node->left, key);
        else if (key > node->key)
            node->right = insert(node->right, key);
        else
            return node;  // duplicate

        return rebalance(node);
    }

public:
    void insert(int key) { root = insert(root, key); }
    // Search is identical to BST — O(log n) guaranteed
};
```

**Red-Black Tree:**

A Red-Black tree is another self-balancing BST used internally by `std::map`, `std::set`, `std::multimap`, and `std::multiset` in most C++ standard library implementations.

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
| `std::map` / `std::set` | O(log n) guaranteed | Default sorted container in C++ |

**LLD use cases for trees:**
- **File system** — directory hierarchy (Composite pattern)
- **Expression parsing** — Abstract Syntax Tree (AST)
- **Database indexing** — B-Trees, B+ Trees
- **Autocomplete** — Trie (prefix tree)
- **Priority scheduling** — BST for ordered task management
- **Organizational hierarchy** — employee reporting structure

---

### Heaps (Min-Heap, Max-Heap)

A **heap** is a complete binary tree that satisfies the heap property: in a min-heap, every parent is smaller than its children; in a max-heap, every parent is larger. Heaps are the backbone of priority queues.

**std::priority_queue — max-heap by default:**

```cpp
#include <queue>
#include <iostream>
#include <vector>
using namespace std;

int main() {
    // Max-heap (default)
    priority_queue<int> max_heap;
    max_heap.push(30);  // O(log n)
    max_heap.push(10);
    max_heap.push(50);
    max_heap.push(20);

    cout << "Max element: " << max_heap.top() << endl;  // 50 — O(1)
    max_heap.pop();  // removes 50 — O(log n)
    cout << "Next max: " << max_heap.top() << endl;  // 30

    // Min-heap
    priority_queue<int, vector<int>, greater<int>> min_heap;
    min_heap.push(30);
    min_heap.push(10);
    min_heap.push(50);
    min_heap.push(20);

    cout << "Min element: " << min_heap.top() << endl;  // 10

    // Custom comparator — e.g., priority queue of tasks
    struct Task {
        int id;
        int priority;
        string name;
    };

    auto cmp = [](const Task& a, const Task& b) {
        return a.priority < b.priority;  // higher priority first
    };

    priority_queue<Task, vector<Task>, decltype(cmp)> task_queue(cmp);
    task_queue.push({1, 5, "Low priority"});
    task_queue.push({2, 10, "High priority"});
    task_queue.push({3, 7, "Medium priority"});

    while (!task_queue.empty()) {
        auto task = task_queue.top();
        task_queue.pop();
        cout << "Processing: " << task.name << " (priority " << task.priority << ")" << endl;
    }
    // Output: High priority (10), Medium priority (7), Low priority (5)
}
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

**Custom min-heap implementation:**

```cpp
class MinHeap {
    vector<int> data;

    void heapify_up(int index) {
        while (index > 0) {
            int parent = (index - 1) / 2;
            if (data[index] < data[parent]) {
                swap(data[index], data[parent]);
                index = parent;
            } else break;
        }
    }

    void heapify_down(int index) {
        int size = data.size();
        while (true) {
            int smallest = index;
            int left = 2 * index + 1;
            int right = 2 * index + 2;

            if (left < size && data[left] < data[smallest]) smallest = left;
            if (right < size && data[right] < data[smallest]) smallest = right;

            if (smallest != index) {
                swap(data[index], data[smallest]);
                index = smallest;
            } else break;
        }
    }

public:
    void push(int val) {
        data.push_back(val);
        heapify_up(data.size() - 1);
    }

    int top() const { return data.front(); }

    void pop() {
        data[0] = data.back();
        data.pop_back();
        if (!data.empty()) heapify_down(0);
    }

    bool empty() const { return data.empty(); }
    size_t size() const { return data.size(); }
};
```

**Heap time complexity:**

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `push` | O(log n) | Bubble up |
| `pop` | O(log n) | Bubble down |
| `top` | O(1) | Peek at root |
| `heapify` (build heap) | O(n) | From unsorted array |

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

```cpp
#include <vector>
#include <list>
#include <unordered_map>
#include <iostream>
using namespace std;

// Representation 1: Adjacency Matrix
// Good for dense graphs, O(1) edge lookup, O(V²) space
class GraphMatrix {
    int V;  // number of vertices
    vector<vector<int>> adj;  // adj[i][j] = 1 if edge from i to j

public:
    GraphMatrix(int vertices) : V(vertices), adj(vertices, vector<int>(vertices, 0)) {}

    void add_edge(int u, int v, bool directed = false) {
        adj[u][v] = 1;
        if (!directed) adj[v][u] = 1;
    }

    bool has_edge(int u, int v) const { return adj[u][v] == 1; }

    void print() const {
        for (int i = 0; i < V; i++) {
            for (int j = 0; j < V; j++) {
                cout << adj[i][j] << " ";
            }
            cout << endl;
        }
    }
};

// Representation 2: Adjacency List
// Good for sparse graphs, O(V + E) space, efficient iteration over neighbors
class GraphList {
    int V;
    vector<list<int>> adj;

public:
    GraphList(int vertices) : V(vertices), adj(vertices) {}

    void add_edge(int u, int v, bool directed = false) {
        adj[u].push_back(v);
        if (!directed) adj[v].push_back(u);
    }

    const list<int>& neighbors(int u) const { return adj[u]; }

    void print() const {
        for (int i = 0; i < V; i++) {
            cout << i << " → ";
            for (int neighbor : adj[i]) {
                cout << neighbor << " ";
            }
            cout << endl;
        }
    }
};

// Representation 3: Adjacency List with string keys (for real-world LLD)
class SocialGraph {
    unordered_map<string, vector<string>> adj;

public:
    void add_user(const string& user) {
        if (adj.find(user) == adj.end()) {
            adj[user] = {};
        }
    }

    void add_friendship(const string& u1, const string& u2) {
        adj[u1].push_back(u2);
        adj[u2].push_back(u1);
    }

    vector<string> get_friends(const string& user) const {
        auto it = adj.find(user);
        if (it != adj.end()) return it->second;
        return {};
    }

    // BFS to find shortest path (degrees of separation)
    int degrees_of_separation(const string& from, const string& to) {
        if (from == to) return 0;

        unordered_set<string> visited;
        queue<pair<string, int>> q;
        q.push({from, 0});
        visited.insert(from);

        while (!q.empty()) {
            auto [user, dist] = q.front();
            q.pop();

            for (const auto& friend_name : adj[user]) {
                if (friend_name == to) return dist + 1;
                if (visited.find(friend_name) == visited.end()) {
                    visited.insert(friend_name);
                    q.push({friend_name, dist + 1});
                }
            }
        }
        return -1;  // not connected
    }
};
```

**Weighted graph:**

```cpp
class WeightedGraph {
    struct Edge {
        int to;
        int weight;
    };

    int V;
    vector<vector<Edge>> adj;

public:
    WeightedGraph(int vertices) : V(vertices), adj(vertices) {}

    void add_edge(int u, int v, int weight, bool directed = false) {
        adj[u].push_back({v, weight});
        if (!directed) adj[v].push_back({u, weight});
    }

    const vector<Edge>& neighbors(int u) const { return adj[u]; }
    int vertices() const { return V; }
};
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

```cpp
class Trie {
    struct TrieNode {
        unordered_map<char, TrieNode*> children;
        bool is_end_of_word = false;
        int word_count = 0;  // how many words pass through this node

        ~TrieNode() {
            for (auto& [ch, child] : children) {
                delete child;
            }
        }
    };

    TrieNode* root;

public:
    Trie() : root(new TrieNode()) {}
    ~Trie() { delete root; }

    // Insert a word — O(L) where L is word length
    void insert(const string& word) {
        TrieNode* node = root;
        for (char ch : word) {
            if (node->children.find(ch) == node->children.end()) {
                node->children[ch] = new TrieNode();
            }
            node = node->children[ch];
            node->word_count++;
        }
        node->is_end_of_word = true;
    }

    // Search for exact word — O(L)
    bool search(const string& word) const {
        TrieNode* node = find_node(word);
        return node && node->is_end_of_word;
    }

    // Check if any word starts with prefix — O(L)
    bool starts_with(const string& prefix) const {
        return find_node(prefix) != nullptr;
    }

    // Autocomplete: find all words with given prefix
    vector<string> autocomplete(const string& prefix, int limit = 10) const {
        TrieNode* node = find_node(prefix);
        if (!node) return {};

        vector<string> results;
        string current = prefix;
        dfs_collect(node, current, results, limit);
        return results;
    }

private:
    TrieNode* find_node(const string& prefix) const {
        TrieNode* node = root;
        for (char ch : prefix) {
            auto it = node->children.find(ch);
            if (it == node->children.end()) return nullptr;
            node = it->second;
        }
        return node;
    }

    void dfs_collect(TrieNode* node, string& current,
                     vector<string>& results, int limit) const {
        if (results.size() >= (size_t)limit) return;

        if (node->is_end_of_word) {
            results.push_back(current);
        }

        for (auto& [ch, child] : node->children) {
            current.push_back(ch);
            dfs_collect(child, current, results, limit);
            current.pop_back();
        }
    }
};

// Usage
int main() {
    Trie trie;
    trie.insert("apple");
    trie.insert("app");
    trie.insert("application");
    trie.insert("apply");
    trie.insert("banana");

    cout << "Search 'app': " << trie.search("app") << endl;        // true
    cout << "Search 'apt': " << trie.search("apt") << endl;        // false
    cout << "Starts with 'app': " << trie.starts_with("app") << endl; // true

    auto suggestions = trie.autocomplete("app");
    cout << "Autocomplete 'app': ";
    for (const auto& s : suggestions) cout << s << " ";
    cout << endl;
    // Output: app apple application apply
}
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
| Vector | O(1)* / O(n) | O(n) | O(n) | O(1) | Default container, random access |
| Linked List | O(1)† | O(1)† | O(n) | O(n) | LRU cache, frequent mid-insertions |
| Stack | O(1) | O(1) | O(n) | O(1) top | Undo/redo, DFS, expression eval |
| Queue | O(1) | O(1) | O(n) | O(1) front | BFS, task scheduling, FIFO |
| Hash Map | O(1) avg | O(1) avg | O(1) avg | O(1) avg | Caching, indexing, counting |
| BST | O(log n)‡ | O(log n)‡ | O(log n)‡ | — | Sorted data, range queries |
| Heap | O(log n) | O(log n) | O(n) | O(1) top | Priority queues, top-K |
| Graph | O(1) | O(E) | O(V+E) | — | Networks, relationships, routing |
| Trie | O(L) | O(L) | O(L) | — | Prefix search, autocomplete |

\* amortized for push_back  
† given a pointer/iterator to the node  
‡ average case for unbalanced BST; O(log n) guaranteed for AVL/Red-Black

---


## 10.2 Algorithms Relevant to LLD

> Algorithms are the engines that power LLD components. A leaderboard needs sorting, a cache needs eviction algorithms, a map needs shortest-path, and a rate limiter needs token bucket or sliding window. This section covers the algorithms most relevant to Low-Level Design problems, with full C++ implementations.

---

### Sorting (for Leaderboards, Rankings)

Sorting is fundamental to any system that displays ordered data — leaderboards, search results, timelines, and rankings.

**std::sort — the default choice:**

```cpp
#include <algorithm>
#include <vector>
#include <iostream>
using namespace std;

struct Player {
    string name;
    int score;
    int games_played;
};

int main() {
    vector<Player> players = {
        {"Alice", 1500, 20},
        {"Bob", 1800, 15},
        {"Charlie", 1500, 25},
        {"Diana", 2000, 10}
    };

    // Sort by score descending
    sort(players.begin(), players.end(), [](const Player& a, const Player& b) {
        return a.score > b.score;
    });

    // Multi-criteria sort: by score desc, then by games_played asc (tiebreaker)
    sort(players.begin(), players.end(), [](const Player& a, const Player& b) {
        if (a.score != b.score) return a.score > b.score;
        return a.games_played < b.games_played;  // fewer games = better
    });

    cout << "Leaderboard:" << endl;
    for (int i = 0; i < players.size(); i++) {
        cout << i + 1 << ". " << players[i].name
             << " - Score: " << players[i].score
             << " - Games: " << players[i].games_played << endl;
    }
}
```

**Stable sort — preserves relative order of equal elements:**

```cpp
// stable_sort guarantees that equal elements maintain their original order
// Important for multi-pass sorting (sort by secondary key first, then primary)
stable_sort(players.begin(), players.end(), [](const Player& a, const Player& b) {
    return a.score > b.score;
});
```

**Partial sort — when you only need top-K:**

```cpp
// Only sort the top 10 — O(n log k) instead of O(n log n)
partial_sort(players.begin(), players.begin() + 10, players.end(),
    [](const Player& a, const Player& b) {
        return a.score > b.score;
    });
// First 10 elements are the top 10 in sorted order
// Remaining elements are in unspecified order
```

**nth_element — find the K-th element in O(n):**

```cpp
// Find the median score — O(n) average
int n = players.size();
nth_element(players.begin(), players.begin() + n / 2, players.end(),
    [](const Player& a, const Player& b) {
        return a.score < b.score;
    });
// players[n/2] is now the median
// Elements before it are ≤ median, elements after are ≥ median
```

**Sorting algorithm comparison:**

| Algorithm | Time (avg) | Time (worst) | Stable | In-place | Notes |
|-----------|-----------|-------------|--------|----------|-------|
| `std::sort` | O(n log n) | O(n log n) | No | Yes | IntroSort (quicksort + heapsort + insertion) |
| `std::stable_sort` | O(n log n) | O(n log n) | Yes | No (O(n) extra) | Merge sort variant |
| `std::partial_sort` | O(n log k) | O(n log k) | No | Yes | Heap-based |
| `std::nth_element` | O(n) avg | O(n²) worst | No | Yes | QuickSelect |
| Counting Sort | O(n + k) | O(n + k) | Yes | No | When range of values is small |
| Radix Sort | O(d * n) | O(d * n) | Yes | No | For integers, d = number of digits |

**LLD use cases for sorting:**
- **Leaderboards** — sort players by score
- **Search results** — sort by relevance, date, price
- **Timeline/Feed** — sort posts by timestamp
- **Scheduling** — sort tasks by priority or deadline
- **Top-K queries** — partial_sort for efficiency

---

### Searching (for Lookup Systems)

**Linear search — O(n):**

```cpp
// Simple but works on unsorted data
template <typename T>
int linear_search(const vector<T>& data, const T& target) {
    for (int i = 0; i < data.size(); i++) {
        if (data[i] == target) return i;
    }
    return -1;  // not found
}
```

**Binary search — O(log n) on sorted data:**

```cpp
#include <algorithm>
using namespace std;

// Manual binary search
int binary_search_manual(const vector<int>& sorted_data, int target) {
    int left = 0, right = sorted_data.size() - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;  // avoid overflow

        if (sorted_data[mid] == target) return mid;
        else if (sorted_data[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;  // not found
}

// STL binary search utilities
void stl_binary_search_examples() {
    vector<int> data = {1, 3, 5, 7, 9, 11, 13, 15};

    // Check existence — O(log n)
    bool found = binary_search(data.begin(), data.end(), 7);  // true

    // Find first element >= target — O(log n)
    auto it = lower_bound(data.begin(), data.end(), 6);
    // *it == 7 (first element >= 6)

    // Find first element > target — O(log n)
    auto it2 = upper_bound(data.begin(), data.end(), 7);
    // *it2 == 9 (first element > 7)

    // Count elements equal to target — O(log n)
    auto range = equal_range(data.begin(), data.end(), 7);
    int count = distance(range.first, range.second);  // 1
}
```

**Binary search on custom objects:**

```cpp
struct Product {
    int id;
    string name;
    double price;
};

// Search products sorted by price
void search_by_price(const vector<Product>& products, double target_price) {
    auto it = lower_bound(products.begin(), products.end(), target_price,
        [](const Product& p, double price) {
            return p.price < price;
        });

    if (it != products.end() && it->price == target_price) {
        cout << "Found: " << it->name << endl;
    }
}
```

**When to use which search:**

| Method | Time | Requirement | Use Case |
|--------|------|-------------|----------|
| Linear search | O(n) | None | Small datasets, unsorted data |
| Binary search | O(log n) | Sorted data | Large sorted datasets |
| Hash lookup | O(1) avg | Hash map | Key-value lookups |
| Trie search | O(L) | Trie built | Prefix-based search |
| BFS/DFS | O(V+E) | Graph | Graph traversal, path finding |

---

### Graph Traversal — BFS/DFS (for Social Networks, Dependency Resolution)

**Breadth-First Search (BFS) — level by level:**

BFS explores all neighbors at the current depth before moving to the next level. It finds the shortest path in unweighted graphs.

```cpp
#include <queue>
#include <unordered_set>
#include <unordered_map>
#include <vector>
using namespace std;

class Graph {
    unordered_map<int, vector<int>> adj;

public:
    void add_edge(int u, int v) {
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    // BFS — O(V + E)
    vector<int> bfs(int start) {
        vector<int> order;
        unordered_set<int> visited;
        queue<int> q;

        visited.insert(start);
        q.push(start);

        while (!q.empty()) {
            int node = q.front();
            q.pop();
            order.push_back(node);

            for (int neighbor : adj[node]) {
                if (visited.find(neighbor) == visited.end()) {
                    visited.insert(neighbor);
                    q.push(neighbor);
                }
            }
        }
        return order;
    }

    // BFS shortest path (unweighted)
    int shortest_path(int start, int end) {
        if (start == end) return 0;

        unordered_set<int> visited;
        queue<pair<int, int>> q;  // {node, distance}

        visited.insert(start);
        q.push({start, 0});

        while (!q.empty()) {
            auto [node, dist] = q.front();
            q.pop();

            for (int neighbor : adj[node]) {
                if (neighbor == end) return dist + 1;
                if (visited.find(neighbor) == visited.end()) {
                    visited.insert(neighbor);
                    q.push({neighbor, dist + 1});
                }
            }
        }
        return -1;  // no path
    }

    // BFS level-order (useful for "friends of friends")
    vector<vector<int>> bfs_levels(int start) {
        vector<vector<int>> levels;
        unordered_set<int> visited;
        queue<int> q;

        visited.insert(start);
        q.push(start);

        while (!q.empty()) {
            int level_size = q.size();
            vector<int> current_level;

            for (int i = 0; i < level_size; i++) {
                int node = q.front();
                q.pop();
                current_level.push_back(node);

                for (int neighbor : adj[node]) {
                    if (visited.find(neighbor) == visited.end()) {
                        visited.insert(neighbor);
                        q.push(neighbor);
                    }
                }
            }
            levels.push_back(current_level);
        }
        return levels;
    }
};
```

**Depth-First Search (DFS) — go deep first:**

DFS explores as far as possible along each branch before backtracking. It's used for cycle detection, topological sorting, and connected components.

```cpp
class GraphDFS {
    unordered_map<int, vector<int>> adj;

public:
    void add_edge(int u, int v, bool directed = false) {
        adj[u].push_back(v);
        if (!directed) adj[v].push_back(u);
    }

    // DFS recursive — O(V + E)
    void dfs(int node, unordered_set<int>& visited) {
        visited.insert(node);
        cout << node << " ";

        for (int neighbor : adj[node]) {
            if (visited.find(neighbor) == visited.end()) {
                dfs(neighbor, visited);
            }
        }
    }

    // DFS iterative (using stack)
    vector<int> dfs_iterative(int start) {
        vector<int> order;
        unordered_set<int> visited;
        stack<int> s;

        s.push(start);

        while (!s.empty()) {
            int node = s.top();
            s.pop();

            if (visited.find(node) != visited.end()) continue;
            visited.insert(node);
            order.push_back(node);

            for (int neighbor : adj[node]) {
                if (visited.find(neighbor) == visited.end()) {
                    s.push(neighbor);
                }
            }
        }
        return order;
    }

    // Cycle detection in directed graph using DFS
    bool has_cycle_directed() {
        unordered_set<int> visited;
        unordered_set<int> rec_stack;  // nodes in current recursion path

        for (auto& [node, _] : adj) {
            if (visited.find(node) == visited.end()) {
                if (dfs_cycle(node, visited, rec_stack)) return true;
            }
        }
        return false;
    }

    // Topological Sort (for dependency resolution) — only for DAGs
    vector<int> topological_sort() {
        unordered_map<int, int> in_degree;
        for (auto& [node, _] : adj) in_degree[node] = 0;
        for (auto& [node, neighbors] : adj) {
            for (int n : neighbors) in_degree[n]++;
        }

        queue<int> q;
        for (auto& [node, deg] : in_degree) {
            if (deg == 0) q.push(node);
        }

        vector<int> order;
        while (!q.empty()) {
            int node = q.front();
            q.pop();
            order.push_back(node);

            for (int neighbor : adj[node]) {
                in_degree[neighbor]--;
                if (in_degree[neighbor] == 0) {
                    q.push(neighbor);
                }
            }
        }

        // If order.size() != number of nodes → cycle exists
        return order;
    }

private:
    bool dfs_cycle(int node, unordered_set<int>& visited, unordered_set<int>& rec_stack) {
        visited.insert(node);
        rec_stack.insert(node);

        for (int neighbor : adj[node]) {
            if (rec_stack.find(neighbor) != rec_stack.end()) return true;  // back edge = cycle
            if (visited.find(neighbor) == visited.end()) {
                if (dfs_cycle(neighbor, visited, rec_stack)) return true;
            }
        }

        rec_stack.erase(node);
        return false;
    }
};
```

**BFS vs DFS comparison:**

| Feature | BFS | DFS |
|---------|-----|-----|
| Data structure | Queue | Stack (or recursion) |
| Explores | Level by level | Branch by branch |
| Shortest path (unweighted) | Yes | No |
| Space complexity | O(V) — width of graph | O(V) — depth of graph |
| Cycle detection | Yes | Yes (with recursion stack) |
| Topological sort | Yes (Kahn's algorithm) | Yes (reverse post-order) |
| Use case | Shortest path, level-order | Cycle detection, topological sort |

**LLD use cases:**
- **Social networks** — BFS for "people you may know" (friends of friends)
- **Dependency resolution** — topological sort for build systems, package managers
- **Maze solving** — BFS for shortest path
- **Spreadsheet** — DFS for dependency graph, cycle detection for circular references
- **State machines** — DFS/BFS for reachability analysis

---

### Shortest Path (for Maps, Routing)

Shortest path algorithms are essential for navigation systems, network routing, and any system that needs to find optimal paths.

**Dijkstra's Algorithm — shortest path from a single source (non-negative weights):**

```cpp
#include <queue>
#include <vector>
#include <unordered_map>
#include <climits>
using namespace std;

class ShortestPath {
    struct Edge {
        int to;
        int weight;
    };

    int V;
    vector<vector<Edge>> adj;

public:
    ShortestPath(int vertices) : V(vertices), adj(vertices) {}

    void add_edge(int u, int v, int weight, bool directed = false) {
        adj[u].push_back({v, weight});
        if (!directed) adj[v].push_back({u, weight});
    }

    // Dijkstra's Algorithm — O((V + E) log V) with priority queue
    vector<int> dijkstra(int source) {
        vector<int> dist(V, INT_MAX);
        // Min-heap: {distance, node}
        priority_queue<pair<int, int>, vector<pair<int, int>>, greater<>> pq;

        dist[source] = 0;
        pq.push({0, source});

        while (!pq.empty()) {
            auto [d, u] = pq.top();
            pq.pop();

            // Skip if we already found a shorter path
            if (d > dist[u]) continue;

            for (const auto& edge : adj[u]) {
                int new_dist = dist[u] + edge.weight;
                if (new_dist < dist[edge.to]) {
                    dist[edge.to] = new_dist;
                    pq.push({new_dist, edge.to});
                }
            }
        }
        return dist;  // dist[i] = shortest distance from source to i
    }

    // Dijkstra with path reconstruction
    pair<int, vector<int>> shortest_path_with_route(int source, int target) {
        vector<int> dist(V, INT_MAX);
        vector<int> parent(V, -1);
        priority_queue<pair<int, int>, vector<pair<int, int>>, greater<>> pq;

        dist[source] = 0;
        pq.push({0, source});

        while (!pq.empty()) {
            auto [d, u] = pq.top();
            pq.pop();

            if (u == target) break;  // found shortest path to target
            if (d > dist[u]) continue;

            for (const auto& edge : adj[u]) {
                int new_dist = dist[u] + edge.weight;
                if (new_dist < dist[edge.to]) {
                    dist[edge.to] = new_dist;
                    parent[edge.to] = u;
                    pq.push({new_dist, edge.to});
                }
            }
        }

        // Reconstruct path
        vector<int> path;
        if (dist[target] == INT_MAX) return {-1, {}};  // no path

        for (int node = target; node != -1; node = parent[node]) {
            path.push_back(node);
        }
        reverse(path.begin(), path.end());
        return {dist[target], path};
    }
};

// Usage: Navigation system
int main() {
    // Cities: 0=NYC, 1=Boston, 2=DC, 3=Philadelphia, 4=Baltimore
    ShortestPath graph(5);
    graph.add_edge(0, 1, 215);  // NYC → Boston: 215 miles
    graph.add_edge(0, 3, 97);   // NYC → Philadelphia: 97 miles
    graph.add_edge(1, 2, 440);  // Boston → DC: 440 miles
    graph.add_edge(3, 4, 100);  // Philadelphia → Baltimore: 100 miles
    graph.add_edge(4, 2, 40);   // Baltimore → DC: 40 miles

    auto [dist, path] = graph.shortest_path_with_route(0, 2);
    cout << "NYC to DC: " << dist << " miles" << endl;
    cout << "Route: ";
    for (int city : path) cout << city << " → ";
    cout << endl;
    // NYC → Philadelphia → Baltimore → DC = 237 miles
}
```

**Bellman-Ford — handles negative weights:**

```cpp
// Bellman-Ford — O(V * E)
// Can detect negative weight cycles
struct BellmanFordEdge {
    int from, to, weight;
};

vector<int> bellman_ford(int V, const vector<BellmanFordEdge>& edges, int source) {
    vector<int> dist(V, INT_MAX);
    dist[source] = 0;

    // Relax all edges V-1 times
    for (int i = 0; i < V - 1; i++) {
        for (const auto& e : edges) {
            if (dist[e.from] != INT_MAX && dist[e.from] + e.weight < dist[e.to]) {
                dist[e.to] = dist[e.from] + e.weight;
            }
        }
    }

    // Check for negative weight cycles (one more iteration)
    for (const auto& e : edges) {
        if (dist[e.from] != INT_MAX && dist[e.from] + e.weight < dist[e.to]) {
            cout << "Negative weight cycle detected!" << endl;
            return {};
        }
    }

    return dist;
}
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

```cpp
#include <functional>
#include <string>
using namespace std;

// Built-in hash functions
void hash_examples() {
    hash<int> int_hash;
    hash<string> str_hash;

    cout << "Hash of 42: " << int_hash(42) << endl;
    cout << "Hash of 'hello': " << str_hash("hello") << endl;

    // Custom hash for composite keys
    struct UserSession {
        int user_id;
        string session_token;
    };

    auto session_hash = [](const UserSession& s) {
        size_t h1 = hash<int>{}(s.user_id);
        size_t h2 = hash<string>{}(s.session_token);
        // Combine hashes (boost::hash_combine approach)
        return h1 ^ (h2 * 0x9e3779b9 + (h1 << 6) + (h1 >> 2));
    };
}
```

**Deduplication using hash sets:**

```cpp
// Remove duplicate requests/messages
class Deduplicator {
    unordered_set<size_t> seen_hashes;
    hash<string> hasher;

public:
    bool is_duplicate(const string& message) {
        size_t h = hasher(message);
        if (seen_hashes.count(h) > 0) {
            return true;  // likely duplicate
        }
        seen_hashes.insert(h);
        return false;
    }

    void clear() { seen_hashes.clear(); }
    size_t unique_count() const { return seen_hashes.size(); }
};
```

**Bloom Filter — space-efficient probabilistic deduplication:**

A Bloom filter can tell you "definitely not in set" or "probably in set" using very little memory. It uses multiple hash functions and a bit array.

```cpp
#include <bitset>
#include <functional>
using namespace std;

template <size_t N>  // N = number of bits
class BloomFilter {
    bitset<N> bits;
    int num_hashes;

    // Generate multiple hash values from a single key
    vector<size_t> get_hashes(const string& key) const {
        vector<size_t> hashes;
        hash<string> h1;
        size_t base = h1(key);

        for (int i = 0; i < num_hashes; i++) {
            // Simple double hashing: h(i) = h1 + i * h2
            size_t h = (base + i * (base >> 16 | 1)) % N;
            hashes.push_back(h);
        }
        return hashes;
    }

public:
    BloomFilter(int num_hash_functions = 3) : num_hashes(num_hash_functions) {}

    void insert(const string& key) {
        for (size_t h : get_hashes(key)) {
            bits.set(h);
        }
    }

    // Returns true if POSSIBLY in set, false if DEFINITELY NOT in set
    bool possibly_contains(const string& key) const {
        for (size_t h : get_hashes(key)) {
            if (!bits.test(h)) return false;  // definitely not present
        }
        return true;  // might be present (false positive possible)
    }
};

// Usage
BloomFilter<1000> filter(3);
filter.insert("user_123");
filter.insert("user_456");

cout << filter.possibly_contains("user_123") << endl;  // true (correct)
cout << filter.possibly_contains("user_789") << endl;  // false (correct) or true (false positive)
```

**LLD use cases for hashing:**
- **Caching** — hash keys for cache lookup
- **Deduplication** — detect duplicate messages, requests, or data
- **Load balancing** — hash-based request routing
- **Data partitioning** — hash-based sharding
- **Checksums** — data integrity verification
- **Bloom filters** — space-efficient membership testing

---

### LRU Cache Implementation (HashMap + Doubly Linked List)

The **LRU (Least Recently Used) Cache** is one of the most frequently asked LLD problems. It evicts the least recently accessed item when the cache is full. The key insight is combining a hash map (O(1) lookup) with a doubly linked list (O(1) insertion/deletion/reordering).

```cpp
#include <unordered_map>
#include <iostream>
using namespace std;

class LRUCache {
    struct Node {
        int key;
        int value;
        Node* prev;
        Node* next;
        Node(int k, int v) : key(k), value(v), prev(nullptr), next(nullptr) {}
    };

    int capacity;
    unordered_map<int, Node*> cache;  // key → node pointer

    // Dummy head and tail for easy boundary handling
    Node* head;  // most recently used (MRU)
    Node* tail;  // least recently used (LRU)

    // Remove a node from the doubly linked list — O(1)
    void remove_node(Node* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
    }

    // Add a node right after head (most recently used position) — O(1)
    void add_to_front(Node* node) {
        node->next = head->next;
        node->prev = head;
        head->next->prev = node;
        head->next = node;
    }

    // Move an existing node to the front (mark as most recently used) — O(1)
    void move_to_front(Node* node) {
        remove_node(node);
        add_to_front(node);
    }

    // Remove the LRU node (right before tail) — O(1)
    Node* remove_lru() {
        Node* lru = tail->prev;
        remove_node(lru);
        return lru;
    }

public:
    LRUCache(int cap) : capacity(cap) {
        // Initialize dummy head and tail
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head->next = tail;
        tail->prev = head;
    }

    ~LRUCache() {
        Node* curr = head;
        while (curr) {
            Node* next = curr->next;
            delete curr;
            curr = next;
        }
    }

    // Get value by key — O(1)
    int get(int key) {
        auto it = cache.find(key);
        if (it == cache.end()) return -1;  // cache miss

        Node* node = it->second;
        move_to_front(node);  // mark as recently used
        return node->value;
    }

    // Put key-value pair — O(1)
    void put(int key, int value) {
        auto it = cache.find(key);

        if (it != cache.end()) {
            // Key exists — update value and move to front
            Node* node = it->second;
            node->value = value;
            move_to_front(node);
        } else {
            // Key doesn't exist — insert new
            if ((int)cache.size() >= capacity) {
                // Evict LRU
                Node* lru = remove_lru();
                cache.erase(lru->key);
                delete lru;
            }

            Node* new_node = new Node(key, value);
            cache[key] = new_node;
            add_to_front(new_node);
        }
    }

    void print_cache() {
        cout << "Cache (MRU → LRU): ";
        Node* curr = head->next;
        while (curr != tail) {
            cout << "[" << curr->key << ":" << curr->value << "] ";
            curr = curr->next;
        }
        cout << endl;
    }
};

// Usage
int main() {
    LRUCache cache(3);  // capacity 3

    cache.put(1, 10);
    cache.put(2, 20);
    cache.put(3, 30);
    cache.print_cache();  // [3:30] [2:20] [1:10]

    cache.get(1);         // access key 1 → moves to front
    cache.print_cache();  // [1:10] [3:30] [2:20]

    cache.put(4, 40);     // capacity full → evicts key 2 (LRU)
    cache.print_cache();  // [4:40] [1:10] [3:30]

    cout << "Get key 2: " << cache.get(2) << endl;  // -1 (evicted)
    cout << "Get key 3: " << cache.get(3) << endl;  // 30
}
```

**Why HashMap + Doubly Linked List?**

```
HashMap: key → Node*
  Provides O(1) lookup to find any node by key

Doubly Linked List: head ↔ node1 ↔ node2 ↔ ... ↔ tail
  - Head side = Most Recently Used (MRU)
  - Tail side = Least Recently Used (LRU)
  - O(1) to move any node to front (given pointer from HashMap)
  - O(1) to remove from tail (eviction)

Combined: ALL operations are O(1)
  get(key):  HashMap lookup → move node to front → return value
  put(key):  HashMap lookup → if exists: update + move to front
                             → if new: add to front, evict from tail if full
```

---

### LFU Cache Implementation

The **LFU (Least Frequently Used) Cache** evicts the item that has been accessed the fewest times. If there's a tie, it evicts the least recently used among them.

```cpp
#include <unordered_map>
#include <list>
#include <iostream>
using namespace std;

class LFUCache {
    struct Node {
        int key, value, freq;
        Node(int k, int v) : key(k), value(v), freq(1) {}
    };

    int capacity;
    int min_freq;  // track the minimum frequency for O(1) eviction

    // key → iterator to node in frequency list
    unordered_map<int, list<Node>::iterator> key_map;

    // frequency → list of nodes with that frequency (ordered by recency)
    unordered_map<int, list<Node>> freq_map;

    void touch(list<Node>::iterator it) {
        int key = it->key;
        int value = it->value;
        int freq = it->freq;

        // Remove from current frequency list
        freq_map[freq].erase(it);
        if (freq_map[freq].empty()) {
            freq_map.erase(freq);
            if (min_freq == freq) min_freq++;  // update min frequency
        }

        // Add to next frequency list (front = most recent)
        int new_freq = freq + 1;
        freq_map[new_freq].push_front(Node(key, value));
        freq_map[new_freq].front().freq = new_freq;
        key_map[key] = freq_map[new_freq].begin();
    }

public:
    LFUCache(int cap) : capacity(cap), min_freq(0) {}

    int get(int key) {
        auto it = key_map.find(key);
        if (it == key_map.end()) return -1;

        int value = it->second->value;
        touch(it->second);
        return value;
    }

    void put(int key, int value) {
        if (capacity <= 0) return;

        auto it = key_map.find(key);
        if (it != key_map.end()) {
            // Update existing
            it->second->value = value;
            touch(it->second);
            return;
        }

        // Evict if at capacity
        if ((int)key_map.size() >= capacity) {
            // Remove LFU (and LRU among ties) — back of min_freq list
            auto& min_list = freq_map[min_freq];
            int evict_key = min_list.back().key;
            min_list.pop_back();
            if (min_list.empty()) freq_map.erase(min_freq);
            key_map.erase(evict_key);
        }

        // Insert new node with frequency 1
        min_freq = 1;
        freq_map[1].push_front(Node(key, value));
        key_map[key] = freq_map[1].begin();
    }
};

// Usage
int main() {
    LFUCache cache(3);

    cache.put(1, 10);  // freq: {1:1}
    cache.put(2, 20);  // freq: {1:1, 2:1}
    cache.put(3, 30);  // freq: {1:1, 2:1, 3:1}

    cache.get(1);       // freq: {1:2, 2:1, 3:1}
    cache.get(1);       // freq: {1:3, 2:1, 3:1}
    cache.get(2);       // freq: {1:3, 2:2, 3:1}

    cache.put(4, 40);   // evicts key 3 (freq=1, LFU)
    cout << cache.get(3) << endl;  // -1 (evicted)
    cout << cache.get(4) << endl;  // 40
}
```

**LRU vs LFU comparison:**

| Feature | LRU Cache | LFU Cache |
|---------|-----------|-----------|
| Eviction policy | Least recently used | Least frequently used |
| Data structures | HashMap + Doubly Linked List | HashMap + Frequency Map + Lists |
| Time complexity | O(1) all operations | O(1) all operations |
| Best for | General caching, recency matters | Frequency matters (popular items stay) |
| Weakness | Scan pollution (one-time access evicts useful items) | Frequency accumulation (old popular items never evicted) |

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

```cpp
#include <map>
#include <string>
#include <functional>
#include <iostream>
#include <vector>
using namespace std;

class ConsistentHash {
    // Sorted map of hash positions on the ring → server name
    map<size_t, string> ring;
    int num_virtual_nodes;  // virtual nodes per server for better distribution
    hash<string> hasher;

public:
    ConsistentHash(int virtual_nodes = 150) : num_virtual_nodes(virtual_nodes) {}

    // Add a server to the ring
    void add_server(const string& server) {
        for (int i = 0; i < num_virtual_nodes; i++) {
            string virtual_key = server + "#" + to_string(i);
            size_t hash_val = hasher(virtual_key);
            ring[hash_val] = server;
        }
        cout << "Added server: " << server
             << " (" << num_virtual_nodes << " virtual nodes)" << endl;
    }

    // Remove a server from the ring
    void remove_server(const string& server) {
        for (int i = 0; i < num_virtual_nodes; i++) {
            string virtual_key = server + "#" + to_string(i);
            size_t hash_val = hasher(virtual_key);
            ring.erase(hash_val);
        }
        cout << "Removed server: " << server << endl;
    }

    // Find which server a key maps to
    string get_server(const string& key) const {
        if (ring.empty()) return "";

        size_t hash_val = hasher(key);

        // Find the first server position >= hash_val (clockwise on the ring)
        auto it = ring.lower_bound(hash_val);
        if (it == ring.end()) {
            it = ring.begin();  // wrap around the ring
        }
        return it->second;
    }

    size_t server_count() const {
        // Count unique servers
        unordered_set<string> servers;
        for (const auto& [_, server] : ring) {
            servers.insert(server);
        }
        return servers.size();
    }
};

// Usage
int main() {
    ConsistentHash ch(100);  // 100 virtual nodes per server

    ch.add_server("server-1");
    ch.add_server("server-2");
    ch.add_server("server-3");

    // Route keys to servers
    vector<string> keys = {"user_1", "user_2", "user_3", "user_4", "user_5"};
    for (const auto& key : keys) {
        cout << key << " → " << ch.get_server(key) << endl;
    }

    cout << "\n--- Adding server-4 ---\n" << endl;
    ch.add_server("server-4");

    // Most keys stay on the same server!
    for (const auto& key : keys) {
        cout << key << " → " << ch.get_server(key) << endl;
    }
    // Only ~1/4 of keys move to the new server (instead of ~75% with simple hashing)
}
```

**How consistent hashing works:**

```
Ring (0 to 2^64):

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

```cpp
#include <chrono>
#include <mutex>
using namespace std;

class TokenBucket {
    int max_tokens;          // bucket capacity
    double refill_rate;      // tokens per second
    double current_tokens;
    chrono::steady_clock::time_point last_refill;
    mutex mtx;

    void refill() {
        auto now = chrono::steady_clock::now();
        double elapsed = chrono::duration<double>(now - last_refill).count();
        current_tokens = min((double)max_tokens, current_tokens + elapsed * refill_rate);
        last_refill = now;
    }

public:
    TokenBucket(int capacity, double rate)
        : max_tokens(capacity), refill_rate(rate),
          current_tokens(capacity),
          last_refill(chrono::steady_clock::now()) {}

    // Try to consume a token — returns true if allowed
    bool allow_request(int tokens = 1) {
        lock_guard<mutex> lock(mtx);
        refill();

        if (current_tokens >= tokens) {
            current_tokens -= tokens;
            return true;   // request allowed
        }
        return false;      // rate limited
    }

    double available_tokens() {
        lock_guard<mutex> lock(mtx);
        refill();
        return current_tokens;
    }
};

// Usage
int main() {
    // 10 tokens max, refill at 2 tokens/second
    TokenBucket limiter(10, 2.0);

    for (int i = 0; i < 15; i++) {
        if (limiter.allow_request()) {
            cout << "Request " << i << ": ALLOWED" << endl;
        } else {
            cout << "Request " << i << ": RATE LIMITED" << endl;
        }
    }
    // First 10 requests allowed (bucket starts full)
    // Requests 11-14 rate limited (bucket empty, refill is slow)

    // Wait for tokens to refill
    this_thread::sleep_for(chrono::seconds(3));  // +6 tokens

    for (int i = 0; i < 8; i++) {
        if (limiter.allow_request()) {
            cout << "After wait, request " << i << ": ALLOWED" << endl;
        } else {
            cout << "After wait, request " << i << ": RATE LIMITED" << endl;
        }
    }
}
```

**Token Bucket properties:**
- Allows **bursts** up to bucket capacity
- Smooth rate limiting over time
- Simple to implement and understand
- Used by: AWS API Gateway, Stripe, many web APIs

**Sliding Window Counter:**

Divides time into fixed windows and counts requests per window. The "sliding" part interpolates between the current and previous window for smoother limiting.

```cpp
#include <chrono>
#include <mutex>
using namespace std;

class SlidingWindowCounter {
    int max_requests;           // max requests per window
    chrono::seconds window_size;
    int current_count = 0;
    int previous_count = 0;
    chrono::steady_clock::time_point window_start;
    mutex mtx;

public:
    SlidingWindowCounter(int max_req, chrono::seconds window)
        : max_requests(max_req), window_size(window),
          window_start(chrono::steady_clock::now()) {}

    bool allow_request() {
        lock_guard<mutex> lock(mtx);
        auto now = chrono::steady_clock::now();
        auto elapsed = chrono::duration_cast<chrono::seconds>(now - window_start);

        // Check if we've moved to a new window
        if (elapsed >= window_size) {
            int windows_passed = elapsed.count() / window_size.count();
            if (windows_passed >= 2) {
                previous_count = 0;
                current_count = 0;
            } else {
                previous_count = current_count;
                current_count = 0;
            }
            window_start += window_size * windows_passed;
        }

        // Calculate weighted count (sliding window approximation)
        auto time_in_window = chrono::duration<double>(now - window_start);
        double weight = time_in_window.count() / window_size.count();
        double estimated_count = previous_count * (1.0 - weight) + current_count;

        if (estimated_count < max_requests) {
            current_count++;
            return true;
        }
        return false;
    }
};
```

**Sliding Window Log — exact counting:**

Stores the timestamp of every request. More accurate but uses more memory.

```cpp
#include <deque>
#include <chrono>
#include <mutex>
using namespace std;

class SlidingWindowLog {
    int max_requests;
    chrono::seconds window_size;
    deque<chrono::steady_clock::time_point> request_log;
    mutex mtx;

public:
    SlidingWindowLog(int max_req, chrono::seconds window)
        : max_requests(max_req), window_size(window) {}

    bool allow_request() {
        lock_guard<mutex> lock(mtx);
        auto now = chrono::steady_clock::now();
        auto window_start = now - window_size;

        // Remove expired entries
        while (!request_log.empty() && request_log.front() <= window_start) {
            request_log.pop_front();
        }

        if ((int)request_log.size() < max_requests) {
            request_log.push_back(now);
            return true;
        }
        return false;
    }
};
```

**Fixed Window Counter — simplest approach:**

```cpp
class FixedWindowCounter {
    int max_requests;
    chrono::seconds window_size;
    int count = 0;
    chrono::steady_clock::time_point window_start;
    mutex mtx;

public:
    FixedWindowCounter(int max_req, chrono::seconds window)
        : max_requests(max_req), window_size(window),
          window_start(chrono::steady_clock::now()) {}

    bool allow_request() {
        lock_guard<mutex> lock(mtx);
        auto now = chrono::steady_clock::now();

        if (now - window_start >= window_size) {
            // New window
            count = 0;
            window_start = now;
        }

        if (count < max_requests) {
            count++;
            return true;
        }
        return false;
    }
};
```

**Per-user rate limiter:**

```cpp
class PerUserRateLimiter {
    unordered_map<string, TokenBucket*> user_buckets;
    int max_tokens;
    double refill_rate;
    mutex mtx;

public:
    PerUserRateLimiter(int capacity, double rate)
        : max_tokens(capacity), refill_rate(rate) {}

    ~PerUserRateLimiter() {
        for (auto& [_, bucket] : user_buckets) delete bucket;
    }

    bool allow_request(const string& user_id) {
        lock_guard<mutex> lock(mtx);

        if (user_buckets.find(user_id) == user_buckets.end()) {
            user_buckets[user_id] = new TokenBucket(max_tokens, refill_rate);
        }
        return user_buckets[user_id]->allow_request();
    }
};

// Usage
PerUserRateLimiter limiter(100, 10.0);  // 100 burst, 10 req/sec per user

limiter.allow_request("user_123");  // each user has their own bucket
limiter.allow_request("user_456");
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
| Arrays & Vectors | Contiguous memory, O(1) random access | `std::vector`, dynamic arrays |
| Linked Lists | O(1) insert/delete with pointer | `std::list`, custom doubly linked list |
| Stacks & Queues | LIFO / FIFO access patterns | `std::stack`, `std::queue`, `std::deque` |
| Hash Maps | O(1) average lookup by key | `std::unordered_map`, `std::unordered_set` |
| Trees | Hierarchical data, O(log n) operations | BST, AVL, Red-Black, `std::map`/`std::set` |
| Heaps | Priority-based access, O(1) top | `std::priority_queue`, min/max heap |
| Graphs | Relationships and connections | Adjacency list/matrix, BFS, DFS |
| Tries | Prefix-based string operations | Custom trie for autocomplete |
| Sorting | Ordering data for display or search | `std::sort`, `partial_sort`, `nth_element` |
| Searching | Finding elements efficiently | Binary search, hash lookup, BFS/DFS |
| Shortest Path | Optimal routing | Dijkstra, Bellman-Ford, BFS |
| Hashing | Fixed-size mapping for caching/dedup | Hash functions, Bloom filters |
| LRU Cache | Evict least recently used | HashMap + Doubly Linked List |
| LFU Cache | Evict least frequently used | HashMap + Frequency Map + Lists |
| Consistent Hashing | Distributed key routing | Hash ring with virtual nodes |
| Rate Limiting | Control request throughput | Token Bucket, Sliding Window |

---

## Interview Tips for Module 10

1. **"Implement an LRU Cache with O(1) get and put."** Use a HashMap for O(1) key lookup and a Doubly Linked List for O(1) reordering. The head of the list is MRU, the tail is LRU. On `get()`, move the node to the front. On `put()`, add to front and evict from tail if full. This is the single most asked LLD data structure question.

2. **"When would you use a hash map vs a tree map?"** Hash map (`unordered_map`) for O(1) average lookup when you don't need ordering. Tree map (`map`) for O(log n) lookup when you need sorted keys, range queries, or ordered iteration. Example: use hash map for a cache, tree map for a leaderboard that needs range queries.

3. **"How does consistent hashing work?"** Servers and keys are placed on a hash ring. Each key is assigned to the first server found clockwise. Virtual nodes improve distribution. When a server is added/removed, only ~1/N of keys are remapped (vs ~100% with simple modulo hashing). Used in distributed caches, load balancers, and database sharding.

4. **"Implement a rate limiter."** Token Bucket is the most common answer: tokens refill at a fixed rate, each request consumes a token, requests are rejected when the bucket is empty. Allows controlled bursts. For per-user limiting, maintain a separate bucket per user ID in a hash map. Discuss trade-offs with sliding window (more accurate, more memory).

5. **"What data structure would you use for autocomplete?"** A Trie (prefix tree). Insert all words into the trie. For autocomplete, traverse to the prefix node, then DFS to collect all words below it. Time: O(L) to reach the prefix node + O(K) to collect K results. Alternative: sorted array with binary search on prefix, but trie is more efficient for dynamic insertions.

6. **"How would you design a leaderboard?"** For top-K queries: use a min-heap of size K — O(n log K) to build, O(1) to get minimum. For dynamic updates: use a balanced BST (`std::set` or `std::map`) for O(log n) insert/delete/rank queries. For very large scale: use a sorted set (like Redis ZSET) which combines a skip list and hash map.

7. **"Explain BFS vs DFS and when to use each."** BFS explores level by level using a queue — finds shortest path in unweighted graphs, good for "nearest" queries (friends of friends). DFS explores depth-first using a stack/recursion — good for cycle detection, topological sorting, and exhaustive search. BFS uses more memory (stores entire level), DFS uses less (stores one path).

8. **"How does an LFU Cache differ from LRU?"** LRU evicts the least recently accessed item. LFU evicts the least frequently accessed item (with LRU as tiebreaker). LFU is better when access frequency matters (popular items should stay cached). LRU is simpler and better for general workloads. Both achieve O(1) for all operations with the right data structures.

9. **"What is a Bloom filter and when would you use it?"** A space-efficient probabilistic data structure that can tell you "definitely not in set" or "probably in set." Uses multiple hash functions and a bit array. False positives are possible, false negatives are not. Use for: checking if a username is taken (before hitting the database), spam filtering, cache lookup optimization, duplicate detection in streams.

10. **"How would you detect a cycle in a dependency graph?"** Use DFS with a recursion stack (set of nodes in the current path). If you visit a node that's already in the recursion stack, there's a cycle. For directed graphs, maintain three states: unvisited, in-progress (in recursion stack), and completed. A back edge to an in-progress node indicates a cycle. Used in: build systems, spreadsheet formula evaluation, package managers.

11. **"What is the time complexity of Dijkstra's algorithm?"** O((V + E) log V) with a binary heap priority queue. O(V² + E) with a simple array (better for dense graphs). O((V + E) log V) is optimal for sparse graphs. Dijkstra doesn't work with negative edge weights — use Bellman-Ford O(V * E) for that.

12. **"How would you implement a priority task scheduler?"** Use a max-heap (priority queue) where each task has a priority value. The highest priority task is always at the top — O(1) to peek, O(log n) to insert or extract. For tasks with the same priority, use a secondary ordering (timestamp for FIFO among equal priorities). For dynamic priority changes, use a combination of heap and hash map.

