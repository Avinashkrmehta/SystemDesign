# Module 18: Databases — Relational (SQL)

> Databases are the backbone of every system. Choosing the right database, designing the right schema, writing efficient queries, and understanding how transactions and indexes work under the hood are skills that separate good engineers from great ones. This module covers relational databases in depth — from fundamentals and SQL to normalization, indexing internals, transaction isolation, locking strategies, and the characteristics of popular SQL databases.

> **Ruby Context:** Ruby has a rich ecosystem for working with relational databases. **ActiveRecord** (part of Rails) is the dominant ORM, providing migrations, validations, associations, and query building. For lighter-weight alternatives, the **Sequel** gem offers a powerful dataset-oriented approach. For raw database access, the **pg** gem (PostgreSQL) and **mysql2** gem (MySQL) provide direct connections. Understanding the SQL fundamentals below is essential even when using an ORM — it helps you write efficient queries, debug performance issues, and design proper schemas.

---

## 18.1 Relational Database Fundamentals

> A **relational database** organizes data into **tables** (also called relations) consisting of **rows** (records/tuples) and **columns** (fields/attributes). Tables are related to each other through keys. The relational model was proposed by Edgar F. Codd in 1970 and remains the dominant data model for structured data.

---

### Tables, Rows, Columns

A table is a two-dimensional structure that stores data about a specific entity:

```
Table: users
+----+----------+---------------------+------+--------+---------------------+
| id | name     | email               | age  | status | created_at          |
+----+----------+---------------------+------+--------+---------------------+
|  1 | Alice    | alice@example.com   |   30 | active | 2024-01-15 10:30:00 |
|  2 | Bob      | bob@example.com     |   25 | active | 2024-02-20 14:45:00 |
|  3 | Charlie  | charlie@example.com |   35 | inactive| 2024-03-10 09:15:00 |
+----+----------+---------------------+------+--------+---------------------+

- Each ROW is a single record (one user)
- Each COLUMN is an attribute (name, email, age)
- Each CELL holds a single atomic value
- The table has a defined SCHEMA (column names, types, constraints)
```

**Key Terminology:**

| Term | Formal Name | Description |
|------|-------------|-------------|
| Table | Relation | A collection of related data organized in rows and columns |
| Row | Tuple / Record | A single entry in the table (one user, one order) |
| Column | Attribute / Field | A property of the entity (name, email, age) |
| Schema | — | The structure definition: column names, types, constraints |
| Degree | — | Number of columns in a table |
| Cardinality | — | Number of rows in a table |

> **Ruby Context:** In ActiveRecord, tables map to model classes, rows map to model instances, and columns map to attributes. A `users` table corresponds to a `User` model class. Each `User.find(1)` returns an object representing one row.

---

### Primary Key, Foreign Key, Composite Key

**Primary Key (PK):**
A column (or set of columns) that **uniquely identifies** each row in a table. Every table must have a primary key.

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,  -- surrogate key (auto-generated)
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

> **Ruby Context — ActiveRecord Migration:**
```ruby
class CreateUsers < ActiveRecord::Migration[7.1]
  def change
    create_table :users do |t|  # id BIGINT PRIMARY KEY is added automatically
      t.string :email, null: false, index: { unique: true }
      t.string :name, null: false, limit: 100
      t.timestamps  # adds created_at and updated_at automatically
    end
  end
end
```

**Primary Key Properties:**
- **Unique:** No two rows can have the same primary key value
- **Not NULL:** Primary key columns cannot contain NULL values
- **Immutable:** Primary key values should never change (in practice)
- **Single per table:** Each table has exactly one primary key (but it can span multiple columns)

**Natural Key vs Surrogate Key:**

| Type | Description | Example | Pros | Cons |
|------|-------------|---------|------|------|
| **Natural Key** | A real-world attribute that is naturally unique | Email, SSN, ISBN | Meaningful, no extra column | May change, may not be truly unique, can be large |
| **Surrogate Key** | An artificial identifier with no business meaning | Auto-increment ID, UUID | Stable, small, fast joins | No business meaning, extra column |

**Recommendation:** Use **surrogate keys** (auto-increment BIGINT or UUID) as primary keys. Use natural keys as unique constraints.

**Auto-Increment vs UUID:**

| Feature | Auto-Increment (BIGINT) | UUID (v4) |
|---------|------------------------|-----------|
| Size | 8 bytes | 16 bytes |
| Ordering | Sequential (good for B-Tree indexes) | Random (causes index fragmentation) |
| Generation | Database generates (single point) | Application generates (distributed-friendly) |
| Guessability | Predictable (id=1, 2, 3...) | Unpredictable (good for public APIs) |
| Distributed systems | Requires coordination | No coordination needed |
| Index performance | Excellent (sequential inserts at end of B-Tree) | Poor (random inserts cause page splits) |

**Tip for distributed systems:** Use **UUID v7** (time-ordered UUID) or **Snowflake IDs** — they combine the distributed generation of UUIDs with the sequential ordering of auto-increment.

> **Ruby Context:** Rails 6+ supports UUID primary keys natively. Enable with `create_table :users, id: :uuid`. The `securerandom` stdlib provides `SecureRandom.uuid`. For Snowflake IDs, gems like `snowflake-id` are available.

---

**Foreign Key (FK):**
A column that references the primary key of another table, establishing a **relationship** between the two tables.

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE RESTRICT        -- prevent deleting user if they have orders
        ON UPDATE CASCADE         -- if user id changes, update orders too
);
```

> **Ruby Context — ActiveRecord Migration with Foreign Key:**
```ruby
class CreateOrders < ActiveRecord::Migration[7.1]
  def change
    create_table :orders do |t|
      t.references :user, null: false, foreign_key: { on_delete: :restrict }
      t.decimal :total_amount, precision: 10, scale: 2, null: false
      t.string :status, default: 'pending'
      t.timestamps
    end
  end
end

# In the model:
class Order < ApplicationRecord
  belongs_to :user
end

class User < ApplicationRecord
  has_many :orders, dependent: :restrict_with_exception
end
```

**Referential Integrity Actions:**

| Action | ON DELETE | ON UPDATE | Description |
|--------|-----------|-----------|-------------|
| `RESTRICT` | Block delete if child rows exist | Block update if child rows reference old value | Safest — prevents orphaned data |
| `CASCADE` | Delete all child rows | Update all child rows | Dangerous for deletes — can cascade widely |
| `SET NULL` | Set FK column to NULL | Set FK column to NULL | Child rows remain but lose the reference |
| `SET DEFAULT` | Set FK column to default value | Set FK column to default value | Rarely used |
| `NO ACTION` | Same as RESTRICT (checked at end of statement) | Same as RESTRICT | Default in most databases |

**Foreign Key Trade-offs in System Design:**

| Aspect | With Foreign Keys | Without Foreign Keys |
|--------|------------------|---------------------|
| Data integrity | Guaranteed by database | Must be enforced by application |
| Write performance | Slower (FK checks on every insert/update/delete) | Faster (no constraint checks) |
| Schema flexibility | Rigid (must define relationships upfront) | Flexible (can evolve independently) |
| Debugging | Easier (database prevents invalid data) | Harder (orphaned data can accumulate) |
| Microservices | Not possible across databases | Application-level consistency required |

**Key Point:** In monolithic applications with a single database, use foreign keys. In microservices with separate databases, enforce referential integrity at the application level (eventual consistency, saga pattern).

---

**Composite Key:**
A primary key that consists of **two or more columns** together. The combination must be unique, though individual columns may have duplicates.

```sql
-- Many-to-many relationship: a user can enroll in many courses, a course has many users
CREATE TABLE enrollments (
    user_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,
    enrolled_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    grade VARCHAR(2),

    PRIMARY KEY (user_id, course_id),  -- composite key
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (course_id) REFERENCES courses(id)
);

-- user_id=1, course_id=101 → valid
-- user_id=1, course_id=102 → valid (same user, different course)
-- user_id=2, course_id=101 → valid (different user, same course)
-- user_id=1, course_id=101 → REJECTED (duplicate combination)
```

> **Ruby Context — Composite Key Migration:**
```ruby
class CreateEnrollments < ActiveRecord::Migration[7.1]
  def change
    create_table :enrollments, primary_key: [:user_id, :course_id] do |t|
      t.bigint :user_id, null: false
      t.bigint :course_id, null: false
      t.datetime :enrolled_at, default: -> { 'CURRENT_TIMESTAMP' }
      t.string :grade, limit: 2
    end
    add_foreign_key :enrollments, :users
    add_foreign_key :enrollments, :courses
  end
end
# Note: Rails 7.1+ supports composite primary keys natively with `query_constraints`.
```

---

### Constraints

Constraints enforce **data integrity rules** at the database level:

| Constraint | Purpose | Example |
|-----------|---------|---------|
| `PRIMARY KEY` | Uniquely identifies each row | `id BIGINT PRIMARY KEY` |
| `FOREIGN KEY` | References another table's primary key | `FOREIGN KEY (user_id) REFERENCES users(id)` |
| `NOT NULL` | Column cannot contain NULL | `name VARCHAR(100) NOT NULL` |
| `UNIQUE` | All values in the column must be distinct | `email VARCHAR(255) UNIQUE` |
| `CHECK` | Values must satisfy a condition | `CHECK (age >= 0 AND age <= 150)` |
| `DEFAULT` | Provides a default value if none is specified | `status VARCHAR(20) DEFAULT 'active'` |
| `UNIQUE INDEX` | Unique constraint with an index for fast lookups | `CREATE UNIQUE INDEX idx_email ON users(email)` |

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    sku VARCHAR(50) NOT NULL UNIQUE,                    -- unique product code
    price DECIMAL(10, 2) NOT NULL CHECK (price > 0),   -- must be positive
    stock_quantity INT NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    category_id BIGINT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL
);
```

> **Ruby Context — ActiveRecord Validations + DB Constraints:**
```ruby
# Migration (database-level constraints):
class CreateProducts < ActiveRecord::Migration[7.1]
  def change
    create_table :products do |t|
      t.string :name, null: false, limit: 200
      t.string :sku, null: false, limit: 50, index: { unique: true }
      t.decimal :price, precision: 10, scale: 2, null: false
      t.integer :stock_quantity, null: false, default: 0
      t.references :category, foreign_key: { on_delete: :nullify }
      t.boolean :is_active, default: true
      t.timestamps
    end
    # CHECK constraints via raw SQL (Rails doesn't have a DSL for CHECK):
    execute "ALTER TABLE products ADD CONSTRAINT price_positive CHECK (price > 0)"
    execute "ALTER TABLE products ADD CONSTRAINT stock_non_negative CHECK (stock_quantity >= 0)"
  end
end

# Model (application-level validations — always use BOTH):
class Product < ApplicationRecord
  belongs_to :category, optional: true
  validates :name, presence: true, length: { maximum: 200 }
  validates :sku, presence: true, uniqueness: true, length: { maximum: 50 }
  validates :price, presence: true, numericality: { greater_than: 0 }
  validates :stock_quantity, presence: true, numericality: { greater_than_or_equal_to: 0 }
end
```

---

### Data Types

Choosing the right data type affects storage, performance, and data integrity:

**Numeric Types:**

| Type | Size | Range | Use Case |
|------|------|-------|----------|
| `TINYINT` | 1 byte | -128 to 127 (0 to 255 unsigned) | Flags, small enums |
| `SMALLINT` | 2 bytes | -32,768 to 32,767 | Small counters |
| `INT` / `INTEGER` | 4 bytes | -2.1B to 2.1B | Most integer columns |
| `BIGINT` | 8 bytes | -9.2×10^18 to 9.2×10^18 | Primary keys, large counters |
| `DECIMAL(p,s)` | Variable | Exact precision | Money, financial data (NEVER use FLOAT for money) |
| `FLOAT` | 4 bytes | ~7 decimal digits | Scientific data (approximate) |
| `DOUBLE` | 8 bytes | ~15 decimal digits | Scientific data (approximate) |

**String Types:**

| Type | Max Size | Storage | Use Case |
|------|----------|---------|----------|
| `CHAR(n)` | 255 bytes | Fixed-length (padded) | Fixed-length codes (country codes, status codes) |
| `VARCHAR(n)` | 65,535 bytes | Variable-length | Names, emails, short text |
| `TEXT` | 65,535 bytes | Variable-length | Long text (descriptions, comments) |
| `MEDIUMTEXT` | 16 MB | Variable-length | Articles, blog posts |
| `LONGTEXT` | 4 GB | Variable-length | Large documents (rarely used — use object storage) |

**Date/Time Types:**

| Type | Format | Range | Use Case |
|------|--------|-------|----------|
| `DATE` | YYYY-MM-DD | 1000-01-01 to 9999-12-31 | Birthdays, dates without time |
| `TIME` | HH:MM:SS | -838:59:59 to 838:59:59 | Duration, time of day |
| `DATETIME` | YYYY-MM-DD HH:MM:SS | 1000-01-01 to 9999-12-31 | Timestamps without timezone |
| `TIMESTAMP` | YYYY-MM-DD HH:MM:SS | 1970-01-01 to 2038-01-19 | Timestamps with timezone (stored as UTC) |

**Other Types:**

| Type | Use Case |
|------|----------|
| `BOOLEAN` | True/false flags |
| `ENUM('a','b','c')` | Fixed set of values (use sparingly — hard to modify) |
| `JSON` | Semi-structured data (PostgreSQL has excellent JSON support) |
| `UUID` | Universally unique identifiers |
| `BLOB` | Binary data (avoid — use object storage instead) |
| `ARRAY` | Arrays (PostgreSQL only) |
| `INET` | IP addresses (PostgreSQL only) |

**Key Rules:**
- **Never use FLOAT/DOUBLE for money** — use `DECIMAL(10,2)` for exact precision
- **Use TIMESTAMP over DATETIME** — TIMESTAMP stores in UTC and converts to local timezone
- **Use VARCHAR over CHAR** — unless the length is truly fixed (like country codes)
- **Use the smallest type that fits** — `TINYINT` for status flags, not `INT`
- **Store dates as dates, not strings** — enables date arithmetic and proper sorting

> **Ruby Context:** ActiveRecord maps SQL types to Ruby types automatically. `BIGINT` → `Integer`, `DECIMAL` → `BigDecimal`, `VARCHAR/TEXT` → `String`, `TIMESTAMP` → `Time`/`DateTime`, `BOOLEAN` → `true`/`false`, `JSON` → `Hash`/`Array`. For money, consider the `money-rails` gem which wraps `BigDecimal` with currency support.

---


## 18.2 SQL Deep Dive

> **SQL (Structured Query Language)** is the standard language for interacting with relational databases. It's divided into sub-languages based on the type of operation. Mastering SQL — especially JOINs, subqueries, window functions, and CTEs — is essential for both system design and coding interviews.

> **Ruby Context:** While ActiveRecord generates SQL for you, understanding raw SQL is critical for debugging (check `ActiveRecord::Base.logger`), writing complex queries, using `find_by_sql`, or working with the **Sequel** gem which exposes a more SQL-like DSL. You can always drop to raw SQL with `ActiveRecord::Base.connection.execute(sql)` or the **pg** / **mysql2** gems directly.

---

### DDL, DML, DCL, TCL

SQL is divided into four sub-languages:

| Category | Full Name | Purpose | Key Commands |
|----------|-----------|---------|-------------|
| **DDL** | Data Definition Language | Define/modify database structure | `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME` |
| **DML** | Data Manipulation Language | Query and modify data | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| **DCL** | Data Control Language | Manage permissions | `GRANT`, `REVOKE` |
| **TCL** | Transaction Control Language | Manage transactions | `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

```sql
-- DDL: Define structure
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    total DECIMAL(10,2) NOT NULL
);

ALTER TABLE orders ADD COLUMN status VARCHAR(20) DEFAULT 'pending';
ALTER TABLE orders DROP COLUMN status;
ALTER TABLE orders MODIFY COLUMN total DECIMAL(12,2);

DROP TABLE orders;          -- deletes table and all data (irreversible!)
TRUNCATE TABLE orders;      -- deletes all rows but keeps table structure (faster than DELETE)

-- DML: Manipulate data
INSERT INTO orders (user_id, total) VALUES (1, 99.99);
INSERT INTO orders (user_id, total) VALUES (1, 49.99), (2, 149.99), (3, 29.99);  -- batch insert

UPDATE orders SET total = 109.99 WHERE id = 1;
UPDATE orders SET status = 'cancelled' WHERE created_at < '2024-01-01';  -- bulk update

DELETE FROM orders WHERE id = 1;
DELETE FROM orders WHERE status = 'cancelled' AND created_at < '2023-01-01';

-- DCL: Control access
GRANT SELECT, INSERT ON orders TO 'app_user'@'localhost';
REVOKE DELETE ON orders FROM 'app_user'@'localhost';

-- TCL: Transaction control
BEGIN;                      -- or START TRANSACTION
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                     -- make changes permanent

-- or ROLLBACK;             -- undo all changes since BEGIN
```

> **Ruby Context — ActiveRecord equivalents:**
```ruby
# DDL via migrations (Rails manages schema changes):
# rails generate migration CreateOrders user_id:bigint total:decimal

# DML via ActiveRecord:
Order.create!(user_id: 1, total: 99.99)                          # INSERT
Order.insert_all([{ user_id: 1, total: 49.99 },                  # batch INSERT (Rails 6+)
                  { user_id: 2, total: 149.99 }])
Order.find(1).update!(total: 109.99)                              # UPDATE
Order.where("created_at < ?", '2024-01-01').update_all(status: 'cancelled')  # bulk UPDATE
Order.find(1).destroy                                             # DELETE
Order.where(status: 'cancelled').where("created_at < ?", '2023-01-01').delete_all

# TCL via ActiveRecord:
ActiveRecord::Base.transaction do
  account1.update!(balance: account1.balance - 100)
  account2.update!(balance: account2.balance + 100)
  # Automatically commits; raises → automatic rollback
end
```

---

### JOINs

JOINs combine rows from two or more tables based on a related column. Understanding JOINs is critical for writing efficient queries.

**Setup for examples:**
```sql
-- users table                    -- orders table
-- +----+-------+                 -- +----+---------+--------+
-- | id | name  |                 -- | id | user_id | total  |
-- +----+-------+                 -- +----+---------+--------+
-- |  1 | Alice |                 -- |  1 |       1 |  99.99 |
-- |  2 | Bob   |                 -- |  2 |       1 |  49.99 |
-- |  3 | Charlie|                -- |  3 |       2 | 149.99 |
-- +----+-------+                 -- |  4 |       4 |  29.99 |  ← user_id=4 doesn't exist!
--                                -- +----+---------+--------+
-- Note: Charlie (id=3) has no orders. Order 4 references non-existent user 4.
```

**INNER JOIN** — returns only rows that have matching values in both tables:

```sql
SELECT u.name, o.id AS order_id, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- Result:
-- +-------+----------+--------+
-- | name  | order_id | total  |
-- +-------+----------+--------+
-- | Alice |        1 |  99.99 |
-- | Alice |        2 |  49.99 |
-- | Bob   |        3 | 149.99 |
-- +-------+----------+--------+
-- Charlie is excluded (no orders). Order 4 is excluded (no matching user).
```

> **Ruby Context:** `User.joins(:orders).select("users.name, orders.id AS order_id, orders.total")`

**LEFT JOIN (LEFT OUTER JOIN)** — returns all rows from the left table, with matching rows from the right table (NULL if no match):

```sql
SELECT u.name, o.id AS order_id, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- Result:
-- +---------+----------+--------+
-- | name    | order_id | total  |
-- +---------+----------+--------+
-- | Alice   |        1 |  99.99 |
-- | Alice   |        2 |  49.99 |
-- | Bob     |        3 | 149.99 |
-- | Charlie |     NULL |   NULL |  ← Charlie included with NULLs
-- +---------+----------+--------+
```

> **Ruby Context:** `User.left_joins(:orders).select("users.name, orders.id AS order_id, orders.total")`

**RIGHT JOIN (RIGHT OUTER JOIN)** — returns all rows from the right table, with matching rows from the left table:

```sql
SELECT u.name, o.id AS order_id, o.total
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- Result:
-- +-------+----------+--------+
-- | name  | order_id | total  |
-- +-------+----------+--------+
-- | Alice |        1 |  99.99 |
-- | Alice |        2 |  49.99 |
-- | Bob   |        3 | 149.99 |
-- | NULL  |        4 |  29.99 |  ← Order 4 included, no matching user
-- +-------+----------+--------+
```

**FULL OUTER JOIN** — returns all rows from both tables (NULL where no match):

```sql
SELECT u.name, o.id AS order_id, o.total
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- Result:
-- +---------+----------+--------+
-- | name    | order_id | total  |
-- +---------+----------+--------+
-- | Alice   |        1 |  99.99 |
-- | Alice   |        2 |  49.99 |
-- | Bob     |        3 | 149.99 |
-- | Charlie |     NULL |   NULL |  ← no orders
-- | NULL    |        4 |  29.99 |  ← no matching user
-- +---------+----------+--------+
-- Note: MySQL doesn't support FULL OUTER JOIN directly — use UNION of LEFT and RIGHT JOINs.
```

**CROSS JOIN** — returns the Cartesian product (every row from table A paired with every row from table B):

```sql
SELECT u.name, p.name AS product
FROM users u
CROSS JOIN products p;

-- If users has 3 rows and products has 4 rows → result has 3 × 4 = 12 rows
-- Use case: generating all combinations (e.g., all user-product pairs for a recommendation matrix)
```

**SELF JOIN** — a table joined with itself:

```sql
-- Find employees and their managers (both stored in the same table)
-- employees: id, name, manager_id (references employees.id)

SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Result:
-- +----------+---------+
-- | employee | manager |
-- +----------+---------+
-- | Alice    | NULL    |  ← Alice is the CEO (no manager)
-- | Bob      | Alice   |
-- | Charlie  | Alice   |
-- | Dave     | Bob     |
-- +----------+---------+
```

> **Ruby Context — Self-referential association:**
```ruby
class Employee < ApplicationRecord
  belongs_to :manager, class_name: 'Employee', optional: true
  has_many :reports, class_name: 'Employee', foreign_key: :manager_id

  # Self join query:
  scope :with_managers, -> {
    joins("LEFT JOIN employees managers ON employees.manager_id = managers.id")
      .select("employees.name AS employee, managers.name AS manager")
  }
end
```

**JOIN Performance Tips:**
- Always JOIN on **indexed columns** (primary keys and foreign keys are indexed by default)
- Use **INNER JOIN** when you only need matching rows (faster than OUTER JOINs)
- Avoid joining on **computed expressions** or **functions** — they prevent index usage
- Be careful with **many-to-many JOINs** — they can produce a Cartesian explosion
- Use **EXPLAIN** to verify the query plan uses indexes

---

### Subqueries and CTEs

**Subquery** — a query nested inside another query:

```sql
-- Scalar subquery (returns a single value)
SELECT name, email
FROM users
WHERE id = (SELECT user_id FROM orders WHERE id = 1);

-- IN subquery (returns a list of values)
SELECT name, email
FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders WHERE total > 100);

-- EXISTS subquery (checks for existence — often faster than IN)
SELECT name, email
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.total > 100
);

-- Correlated subquery (references the outer query — runs once per outer row)
SELECT u.name,
       (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u;
```

> **Ruby Context — ActiveRecord subqueries:**
```ruby
# IN subquery
User.where(id: Order.where("total > 100").select(:user_id).distinct)

# EXISTS subquery (Rails 7+)
User.where(Order.where("orders.user_id = users.id").where("total > 100").arel.exists)

# Correlated subquery via select
User.select("users.name, (SELECT COUNT(*) FROM orders WHERE orders.user_id = users.id) AS order_count")
```

**CTE (Common Table Expression)** — a named temporary result set defined with `WITH`:

```sql
-- Simple CTE
WITH active_users AS (
    SELECT id, name, email
    FROM users
    WHERE status = 'active'
)
SELECT au.name, COUNT(o.id) AS order_count
FROM active_users au
LEFT JOIN orders o ON au.id = o.user_id
GROUP BY au.name;

-- Multiple CTEs
WITH
user_totals AS (
    SELECT user_id, SUM(total) AS total_spent, COUNT(*) AS order_count
    FROM orders
    GROUP BY user_id
),
high_spenders AS (
    SELECT user_id, total_spent, order_count
    FROM user_totals
    WHERE total_spent > 1000
)
SELECT u.name, hs.total_spent, hs.order_count
FROM high_spenders hs
JOIN users u ON hs.user_id = u.id
ORDER BY hs.total_spent DESC;

-- Recursive CTE (for hierarchical data like org charts, categories, file trees)
WITH RECURSIVE org_chart AS (
    -- Base case: top-level employees (no manager)
    SELECT id, name, manager_id, 0 AS depth
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case: employees who report to someone in the previous level
    SELECT e.id, e.name, e.manager_id, oc.depth + 1
    FROM employees e
    INNER JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT REPEAT('  ', depth) || name AS org_tree, depth
FROM org_chart
ORDER BY depth, name;

-- Result:
-- +------------------+-------+
-- | org_tree         | depth |
-- +------------------+-------+
-- | Alice            |     0 |
-- |   Bob            |     1 |
-- |   Charlie        |     1 |
-- |     Dave         |     2 |
-- |     Eve          |     2 |
-- +------------------+-------+
```

> **Ruby Context:** ActiveRecord doesn't have built-in CTE support, but you can use `find_by_sql` for complex CTEs, or the **Sequel** gem which has first-class CTE support: `DB[:users].with(:active_users, DB[:users].where(status: 'active'))`. The `activerecord-cte` gem also adds `.with()` to ActiveRecord. For recursive tree queries, the `ancestry` or `closure_tree` gems handle hierarchical data elegantly.

**CTE vs Subquery:**

| Feature | CTE | Subquery |
|---------|-----|----------|
| Readability | Better (named, top-down) | Worse (nested, inside-out) |
| Reusability | Can reference multiple times in the same query | Must repeat the subquery |
| Recursion | Supported (`WITH RECURSIVE`) | Not supported |
| Performance | Usually same as subquery (optimizer inlines it) | Same |
| Materialization | Some DBs materialize CTEs (PostgreSQL < 12) | Never materialized |

---

### Window Functions

**Window functions** perform calculations across a set of rows that are related to the current row — without collapsing the rows into groups (unlike `GROUP BY`).

```sql
-- Syntax:
-- function_name() OVER (
--     [PARTITION BY column]    -- divide rows into groups
--     [ORDER BY column]        -- order within each group
--     [ROWS/RANGE frame]       -- define the window frame
-- )
```

**ROW_NUMBER, RANK, DENSE_RANK:**

```sql
SELECT
    name,
    department,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    RANK()       OVER (ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;

-- Result:
-- +-------+------------+--------+---------+------+------------+
-- | name  | department | salary | row_num | rank | dense_rank |
-- +-------+------------+--------+---------+------+------------+
-- | Alice | Eng        | 150000 |       1 |    1 |          1 |
-- | Bob   | Eng        | 130000 |       2 |    2 |          2 |
-- | Carol | Sales      | 130000 |       3 |    2 |          2 |  ← tie!
-- | Dave  | Sales      | 120000 |       4 |    4 |          3 |
-- | Eve   | Eng        | 110000 |       5 |    5 |          4 |
-- +-------+------------+--------+---------+------+------------+
--
-- ROW_NUMBER: always unique (1, 2, 3, 4, 5) — breaks ties arbitrarily
-- RANK:       ties get same rank, next rank is skipped (1, 2, 2, 4, 5)
-- DENSE_RANK: ties get same rank, next rank is NOT skipped (1, 2, 2, 3, 4)
```

**PARTITION BY — window functions per group:**

```sql
-- Rank employees within each department
SELECT
    name,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;

-- Result:
-- +-------+------------+--------+-----------+
-- | name  | department | salary | dept_rank |
-- +-------+------------+--------+-----------+
-- | Alice | Eng        | 150000 |         1 |
-- | Bob   | Eng        | 130000 |         2 |
-- | Eve   | Eng        | 110000 |         3 |
-- | Carol | Sales      | 130000 |         1 |  ← rank resets per department
-- | Dave  | Sales      | 120000 |         2 |
-- +-------+------------+--------+-----------+
```

**LAG and LEAD — access previous/next rows:**

```sql
SELECT
    date,
    revenue,
    LAG(revenue, 1)  OVER (ORDER BY date) AS prev_day_revenue,
    LEAD(revenue, 1) OVER (ORDER BY date) AS next_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY date) AS daily_change
FROM daily_sales;

-- Result:
-- +------------+---------+------------------+------------------+--------------+
-- | date       | revenue | prev_day_revenue | next_day_revenue | daily_change |
-- +------------+---------+------------------+------------------+--------------+
-- | 2025-01-01 |    1000 |             NULL |             1200 |         NULL |
-- | 2025-01-02 |    1200 |             1000 |              900 |          200 |
-- | 2025-01-03 |     900 |             1200 |             1500 |         -300 |
-- | 2025-01-04 |    1500 |              900 |             NULL |          600 |
-- +------------+---------+------------------+------------------+--------------+
```

**Other Useful Window Functions:**

```sql
-- Running total
SELECT date, revenue,
       SUM(revenue) OVER (ORDER BY date) AS running_total
FROM daily_sales;

-- Moving average (last 7 days)
SELECT date, revenue,
       AVG(revenue) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg_7d
FROM daily_sales;

-- Percentage of total
SELECT department, salary,
       salary * 100.0 / SUM(salary) OVER () AS pct_of_total
FROM employees;

-- First/Last value in partition
SELECT department, name, salary,
       FIRST_VALUE(name) OVER (PARTITION BY department ORDER BY salary DESC) AS highest_paid
FROM employees;

-- NTH_VALUE
SELECT name, salary,
       NTH_VALUE(name, 2) OVER (ORDER BY salary DESC) AS second_highest
FROM employees;

-- NTILE (divide into N equal groups)
SELECT name, salary,
       NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees;
```

> **Ruby Context:** Window functions are best used via raw SQL in ActiveRecord: `Employee.find_by_sql("SELECT name, department, salary, RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank FROM employees")`. The **Sequel** gem has built-in window function support: `DB[:employees].select(:name, :salary).select_append { rank.function.over(order: Sequel.desc(:salary)).as(:rank) }`.

---

### Aggregations (GROUP BY, HAVING)

```sql
-- GROUP BY: collapse rows into groups and apply aggregate functions
SELECT department, COUNT(*) AS emp_count, AVG(salary) AS avg_salary, MAX(salary) AS max_salary
FROM employees
GROUP BY department;

-- HAVING: filter groups (WHERE filters rows BEFORE grouping, HAVING filters AFTER)
SELECT department, COUNT(*) AS emp_count, AVG(salary) AS avg_salary
FROM employees
WHERE status = 'active'           -- filter rows first
GROUP BY department
HAVING COUNT(*) >= 5              -- then filter groups
   AND AVG(salary) > 100000
ORDER BY avg_salary DESC;

-- Common aggregates: COUNT, SUM, AVG, MIN, MAX, COUNT(DISTINCT), GROUP_CONCAT/STRING_AGG
```

> **Ruby Context:**
```ruby
# GROUP BY with aggregates
Employee.group(:department).count                    # => { "Eng" => 10, "Sales" => 5 }
Employee.group(:department).average(:salary)         # => { "Eng" => 130000, "Sales" => 125000 }
Employee.group(:department).maximum(:salary)

# HAVING
Employee.where(status: 'active')
        .group(:department)
        .having("COUNT(*) >= 5 AND AVG(salary) > 100000")
        .select("department, COUNT(*) AS emp_count, AVG(salary) AS avg_salary")
        .order("avg_salary DESC")
```

**Query Execution Order:**
Understanding the logical execution order of SQL clauses is essential for writing correct queries:

```
1. FROM / JOIN      — identify the tables and join them
2. WHERE            — filter individual rows
3. GROUP BY         — group rows
4. HAVING           — filter groups
5. SELECT           — compute output columns (including window functions)
6. DISTINCT         — remove duplicate rows
7. ORDER BY         — sort the result
8. LIMIT / OFFSET   — restrict the number of rows returned
```

This is why you can't use a column alias from SELECT in WHERE (WHERE runs before SELECT), but you can use it in ORDER BY (ORDER BY runs after SELECT).

---

### Views and Materialized Views

**View** — a saved query that acts like a virtual table:

```sql
CREATE VIEW active_user_orders AS
SELECT u.id AS user_id, u.name, u.email, o.id AS order_id, o.total, o.created_at
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active';

-- Use it like a table
SELECT * FROM active_user_orders WHERE total > 100;

-- The view doesn't store data — it runs the underlying query every time
```

**Materialized View** — a view that stores the query result physically (like a cached table):

```sql
-- PostgreSQL syntax
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
    DATE_TRUNC('month', created_at) AS month,
    COUNT(*) AS order_count,
    SUM(total) AS total_revenue,
    AVG(total) AS avg_order_value
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;

-- Query it (reads from stored data — fast!)
SELECT * FROM monthly_revenue WHERE month >= '2025-01-01';

-- Refresh when underlying data changes
REFRESH MATERIALIZED VIEW monthly_revenue;
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;  -- non-blocking refresh (requires unique index)
```

> **Ruby Context:** Rails doesn't have built-in support for database views, but the **scenic** gem provides a clean migration-based workflow for creating and managing views and materialized views:
```ruby
# Using the scenic gem:
# rails generate scenic:view active_user_orders
# Then define SQL in db/views/active_user_orders_v01.sql

# For materialized views:
# rails generate scenic:view monthly_revenue --materialized
class RefreshMonthlyRevenue
  def self.call
    Scenic.database.refresh_materialized_view(:monthly_revenue, concurrently: true, cascade: false)
  end
end
```

**View vs Materialized View:**

| Feature | View | Materialized View |
|---------|------|-------------------|
| Data storage | No (virtual — runs query each time) | Yes (physically stored) |
| Read performance | Same as underlying query | Fast (reads stored data) |
| Data freshness | Always current | Stale until refreshed |
| Write overhead | None | Must refresh after data changes |
| Use case | Simplify complex queries, access control | Expensive aggregations, reporting, dashboards |
| Supported | All databases | PostgreSQL, Oracle, SQL Server (not MySQL natively) |

---


## 18.3 Normalization & Denormalization

> **Normalization** is the process of organizing data to reduce redundancy and improve data integrity. **Denormalization** is the deliberate introduction of redundancy to improve read performance. Understanding when to normalize and when to denormalize is a key system design skill.

---

### Normal Forms

**1NF (First Normal Form):**
- Each column contains **atomic** (indivisible) values
- No repeating groups or arrays in a single column
- Each row is unique (has a primary key)

```
❌ Violates 1NF:
+----+-------+---------------------------+
| id | name  | phone_numbers             |
+----+-------+---------------------------+
|  1 | Alice | 555-0101, 555-0102        |  ← multiple values in one cell
+----+-------+---------------------------+

✅ 1NF:
+----+-------+--------------+
| id | name  | phone_number |
+----+-------+--------------+
|  1 | Alice | 555-0101     |
|  1 | Alice | 555-0102     |  ← separate rows for each value
+----+-------+--------------+
(or better: separate phone_numbers table with foreign key)
```

**2NF (Second Normal Form):**
- Must be in 1NF
- Every non-key column must depend on the **entire** primary key (no partial dependencies)
- Only relevant for tables with composite primary keys

```
❌ Violates 2NF (composite key: student_id + course_id):
+------------+-----------+-------------+------------------+
| student_id | course_id | course_name | student_grade    |
+------------+-----------+-------------+------------------+
|          1 |       101 | Math        | A                |
|          2 |       101 | Math        | B                |  ← course_name depends only on course_id
+------------+-----------+-------------+------------------+
course_name depends on course_id alone, not on (student_id, course_id)

✅ 2NF: Split into two tables
courses: (course_id PK, course_name)
enrollments: (student_id, course_id, student_grade)  -- composite PK
```

**3NF (Third Normal Form):**
- Must be in 2NF
- No **transitive dependencies** — non-key columns must depend directly on the primary key, not on other non-key columns

```
❌ Violates 3NF:
+----+-------+---------------+-------------------+
| id | name  | department_id | department_name   |
+----+-------+---------------+-------------------+
|  1 | Alice |            10 | Engineering       |
|  2 | Bob   |            10 | Engineering       |  ← department_name depends on department_id, not on id
+----+-------+---------------+-------------------+
department_name → depends on department_id → depends on id (transitive!)

✅ 3NF: Split into two tables
employees: (id PK, name, department_id FK)
departments: (department_id PK, department_name)
```

> **Ruby Context:** ActiveRecord naturally encourages 3NF through associations. Instead of storing `department_name` on the `employees` table, you create a `Department` model and use `belongs_to :department`. Rails conventions (separate models with foreign keys) align well with normalized design.

**BCNF (Boyce-Codd Normal Form):**
- Must be in 3NF
- Every **determinant** must be a candidate key
- Stricter than 3NF — handles edge cases where a non-candidate-key column determines part of a candidate key

```
❌ Violates BCNF:
A professor can teach only one subject, but a subject can be taught by multiple professors.
+------------+-----------+-----------+
| student_id | subject   | professor |
+------------+-----------+-----------+
|          1 | Math      | Dr. Smith |
|          2 | Math      | Dr. Smith |  ← professor → subject (professor determines subject)
|          3 | Physics   | Dr. Jones |     but professor is not a candidate key
+------------+-----------+-----------+

✅ BCNF: Split
professor_subjects: (professor PK, subject)
enrollments: (student_id, professor)
```

**Quick Reference:**

| Normal Form | Rule | Eliminates |
|-------------|------|-----------|
| 1NF | Atomic values, no repeating groups | Repeating groups, multi-valued cells |
| 2NF | No partial dependencies on composite key | Partial dependencies |
| 3NF | No transitive dependencies | Transitive dependencies |
| BCNF | Every determinant is a candidate key | Remaining anomalies from 3NF |

**Practical Rule:** Most production databases aim for **3NF**. BCNF is used when 3NF still has anomalies. Going beyond BCNF (4NF, 5NF) is rarely needed in practice.

---

### When to Denormalize

Denormalization intentionally adds redundancy to improve **read performance** at the cost of **write complexity** and **storage**.

**Common Denormalization Techniques:**

| Technique | Description | Example |
|-----------|-------------|---------|
| **Duplicating columns** | Copy a frequently accessed column into another table | Store `user_name` in the `orders` table to avoid joining `users` |
| **Pre-computed aggregates** | Store calculated values instead of computing on the fly | Store `order_count` and `total_spent` in the `users` table |
| **Materialized views** | Physically store the result of a complex query | Monthly revenue summary table |
| **Denormalized tables** | Combine multiple normalized tables into one wide table | Combine `orders`, `order_items`, `products` into a flat reporting table |
| **JSON columns** | Store related data as JSON in a single column | Store `address` as JSON instead of a separate `addresses` table |
| **Summary tables** | Pre-aggregate data for reporting | Daily/weekly/monthly sales summaries |

> **Ruby Context:** ActiveRecord supports counter caches (a form of denormalization) natively: `belongs_to :user, counter_cache: true` automatically maintains an `orders_count` column on the `users` table. For JSON columns, ActiveRecord serializes/deserializes automatically: `attribute :metadata, :json`. The `store` accessor provides a convenient interface for JSON/hstore columns.

**When to Denormalize:**

| Scenario | Why Denormalize |
|----------|----------------|
| Read-heavy workloads (100:1 read:write ratio) | Avoid expensive JOINs on every read |
| Reporting and analytics | Pre-compute aggregates instead of scanning millions of rows |
| High-traffic dashboards | Materialized views or summary tables for sub-second response |
| Search results | Denormalized search index (Elasticsearch) alongside normalized DB |
| Caching layer | Redis stores denormalized data for fast access |
| Event sourcing / CQRS | Write model is normalized; read model is denormalized |

**When NOT to Denormalize:**

| Scenario | Why Keep Normalized |
|----------|-------------------|
| Write-heavy workloads | Denormalized data must be updated in multiple places |
| Data integrity is critical | Redundant data can become inconsistent |
| Small datasets | JOINs are fast on small tables — denormalization adds unnecessary complexity |
| Frequently changing schema | Denormalized structures are harder to evolve |

---

### Trade-offs (Read Performance vs Write Complexity)

| Aspect | Normalized | Denormalized |
|--------|-----------|-------------|
| **Read performance** | Slower (requires JOINs) | Faster (data is pre-joined) |
| **Write performance** | Faster (update one place) | Slower (update multiple places) |
| **Storage** | Less (no redundancy) | More (duplicated data) |
| **Data integrity** | High (single source of truth) | Risk of inconsistency |
| **Schema flexibility** | Easy to modify | Harder to modify (changes ripple) |
| **Query complexity** | Complex (many JOINs) | Simple (flat tables) |
| **Consistency** | Strong (one copy of data) | Eventual (must sync copies) |

**Real-World Pattern — Normalize for Writes, Denormalize for Reads (CQRS):**

```
Write Path (normalized):
  Client → API → Normalized DB (users, orders, products — 3NF)
                  ↓ (async event)
Read Path (denormalized):
  Client → API → Denormalized Read Store (flat tables, materialized views, Elasticsearch)
```

This is the **CQRS (Command Query Responsibility Segregation)** pattern — separate the write model (optimized for consistency) from the read model (optimized for performance).

> **Ruby Context:** In Rails, CQRS can be implemented using ActiveJob or Sidekiq to asynchronously update denormalized read stores after writes. The `rails_event_store` gem provides event sourcing primitives. For search, the `searchkick` gem (backed by Elasticsearch) maintains a denormalized search index alongside your normalized ActiveRecord models.

---


## 18.4 Indexing

> Indexes are the single most important tool for database performance. An index is a data structure that allows the database to find rows quickly without scanning the entire table. Understanding how indexes work internally — especially B-Trees — is essential for writing performant queries and designing efficient schemas.

---

### How Indexes Work — The Basics

Without an index, the database must perform a **full table scan** — reading every row to find matches:

```
Query: SELECT * FROM users WHERE email = 'alice@example.com'

Without index (full table scan):
  Read row 1: email = 'bob@...'       → no match
  Read row 2: email = 'charlie@...'   → no match
  Read row 3: email = 'alice@...'     → MATCH!
  Read row 4: email = 'dave@...'      → no match
  ... (must read ALL rows)
  Time: O(n) — 1 million rows = 1 million reads

With index on email:
  Look up 'alice@example.com' in the index → points to row 3
  Read row 3 directly
  Time: O(log n) — 1 million rows = ~20 reads (B-Tree depth)
```

> **Ruby Context — Adding indexes in Rails migrations:**
```ruby
class AddIndexToUsersEmail < ActiveRecord::Migration[7.1]
  def change
    add_index :users, :email, unique: true
    # Generates: CREATE UNIQUE INDEX index_users_on_email ON users (email)
  end
end
```

---

### B-Tree Index and B+ Tree Index

**B-Tree (Balanced Tree)** is the default index structure in virtually all relational databases.

**B-Tree Properties:**
- Self-balancing tree where all leaf nodes are at the same depth
- Each node can have multiple keys and children (high fan-out)
- Keys are stored in sorted order within each node
- Typical fan-out: 100-500 keys per node (each node = one disk page, typically 4-16 KB)
- Depth is very shallow: a B-Tree with fan-out 500 and 3 levels can index 500^3 = 125 million rows

```
B+ Tree Structure (used by most databases):

                        [50 | 100]                    ← Root node (level 0)
                       /     |     \
              [10|20|30]  [60|70|80]  [110|120|130]   ← Internal nodes (level 1)
              / | | \     / | | \     / | | \
            [1-9][10-19][20-29][30-49] ...            ← Leaf nodes (level 2)
             ↔     ↔      ↔     ↔                    ← Leaf nodes are linked (range scans!)

Each leaf node contains:
  - The indexed column value(s)
  - A pointer to the actual row on disk (row ID / page + offset)
```

**B-Tree vs B+ Tree:**

| Feature | B-Tree | B+ Tree |
|---------|--------|---------|
| Data storage | Keys + data in all nodes | Keys in all nodes, data only in leaf nodes |
| Leaf node linking | No | Yes (doubly linked list) |
| Range queries | Slower (must traverse tree) | Fast (follow leaf node links) |
| Point queries | Slightly faster (data in internal nodes) | Slightly slower (must reach leaf) |
| Used by | — | PostgreSQL, MySQL/InnoDB, Oracle, SQL Server |

**Why B+ Trees are preferred:**
- Internal nodes store only keys → higher fan-out → shallower tree → fewer disk reads
- Leaf nodes are linked → range scans (`WHERE age BETWEEN 20 AND 30`) are sequential reads
- All data is at the same level → predictable performance

**Index Lookup Example:**

```
Query: SELECT * FROM users WHERE age = 25

B+ Tree index on age:
  Level 0 (root):     [50]           → 25 < 50, go left
  Level 1 (internal): [20 | 30]      → 20 ≤ 25 < 30, go to middle child
  Level 2 (leaf):     [21, 22, 23, 24, 25, 26, 27, 28, 29]
                                      → Found 25! → pointer to row on disk
  Read the actual row from disk

Total: 3 disk reads (one per level) instead of scanning millions of rows
```

---

### Hash Index

A **hash index** uses a hash function to map keys directly to their location. It provides O(1) lookups for **exact match** queries but cannot support range queries.

```
Hash function: hash('alice@example.com') → bucket 42
Bucket 42 contains: pointer to row with email='alice@example.com'

✅ Fast for: WHERE email = 'alice@example.com'  (O(1) lookup)
❌ Cannot do: WHERE email LIKE 'alice%'          (no ordering)
❌ Cannot do: WHERE age > 25                     (no range support)
❌ Cannot do: ORDER BY email                     (no sorting)
```

| Feature | B-Tree Index | Hash Index |
|---------|-------------|-----------|
| Exact match (`=`) | O(log n) | O(1) |
| Range queries (`>`, `<`, `BETWEEN`) | Yes | No |
| Prefix matching (`LIKE 'abc%'`) | Yes | No |
| Sorting (`ORDER BY`) | Yes | No |
| Use case | Default — works for everything | Memory-only exact lookups |
| Supported | All databases | PostgreSQL (explicit), MySQL Memory engine |

**Key Point:** B-Tree indexes are the default and work for almost all use cases. Hash indexes are niche — only use them when you exclusively need exact-match lookups and the data fits in memory.

---

### Composite Index (Multi-Column Index)

A **composite index** indexes multiple columns together. The column order matters significantly.

```sql
CREATE INDEX idx_user_status_date ON orders (user_id, status, created_at);
```

> **Ruby Context:**
```ruby
# Rails migration for composite index:
add_index :orders, [:user_id, :status, :created_at], name: 'idx_user_status_date'
```

**The Leftmost Prefix Rule:**
A composite index can be used for queries that filter on a **leftmost prefix** of the indexed columns:

```sql
-- Index: (user_id, status, created_at)

-- ✅ Uses index (full match)
SELECT * FROM orders WHERE user_id = 1 AND status = 'active' AND created_at > '2025-01-01';

-- ✅ Uses index (leftmost prefix: user_id, status)
SELECT * FROM orders WHERE user_id = 1 AND status = 'active';

-- ✅ Uses index (leftmost prefix: user_id)
SELECT * FROM orders WHERE user_id = 1;

-- ❌ Cannot use index (skips user_id — not a leftmost prefix)
SELECT * FROM orders WHERE status = 'active';

-- ❌ Cannot use index (skips user_id and status)
SELECT * FROM orders WHERE created_at > '2025-01-01';

-- ⚠️ Partially uses index (uses user_id, skips status, cannot use created_at for range)
SELECT * FROM orders WHERE user_id = 1 AND created_at > '2025-01-01';
```

**Column Order Strategy:**
1. Put **equality** columns first (`=`)
2. Then **range** columns (`>`, `<`, `BETWEEN`, `LIKE 'prefix%'`)
3. Put the most **selective** (highest cardinality) equality column first

```sql
-- Query: WHERE user_id = ? AND status = ? AND created_at > ?
-- Best index: (user_id, status, created_at)
--              equality  equality  range (must be last)
```

---

### Covering Index

A **covering index** contains all the columns needed by a query, so the database can answer the query entirely from the index without reading the actual table rows (called an **index-only scan**).

```sql
-- Query:
SELECT user_id, status, created_at FROM orders WHERE user_id = 1 AND status = 'active';

-- Regular index: (user_id, status)
-- Database: look up index → find matching rows → go to table to get created_at → return
-- (requires table lookup — slower)

-- Covering index: (user_id, status, created_at)
-- Database: look up index → all needed columns are in the index → return directly
-- (no table lookup — faster!)

-- PostgreSQL INCLUDE syntax (index non-key columns without affecting sort order):
CREATE INDEX idx_orders_covering ON orders (user_id, status) INCLUDE (created_at, total);
```

**Key Point:** Covering indexes can dramatically speed up read-heavy queries by eliminating table lookups. But they increase index size and slow down writes (more data to maintain in the index).

---

### Partial Index (Filtered Index)

A **partial index** only indexes rows that match a condition. This reduces index size and improves performance for queries that filter on that condition.

```sql
-- Only index active orders (90% of queries filter on status = 'active')
CREATE INDEX idx_active_orders ON orders (user_id, created_at)
WHERE status = 'active';

-- This index is smaller (only active orders) and faster to maintain
-- It will be used for: SELECT * FROM orders WHERE status = 'active' AND user_id = 1
-- It will NOT be used for: SELECT * FROM orders WHERE status = 'cancelled' AND user_id = 1
```

> **Ruby Context — Partial index in Rails:**
```ruby
add_index :orders, [:user_id, :created_at],
          where: "status = 'active'",
          name: 'idx_active_orders'
```

**Use Cases:**
- Index only non-NULL values: `WHERE column IS NOT NULL`
- Index only recent data: `WHERE created_at > '2024-01-01'`
- Index only active/pending records: `WHERE status = 'active'`
- Soft deletes: `WHERE deleted_at IS NULL`

---

### Full-Text Index

A **full-text index** enables efficient text search across large text columns — far faster than `LIKE '%keyword%'` which cannot use regular B-Tree indexes.

```sql
-- MySQL
ALTER TABLE articles ADD FULLTEXT INDEX idx_ft_content (title, body);
SELECT * FROM articles WHERE MATCH(title, body) AGAINST('database indexing' IN NATURAL LANGUAGE MODE);

-- PostgreSQL (using tsvector)
CREATE INDEX idx_ft_articles ON articles USING GIN (to_tsvector('english', title || ' ' || body));
SELECT * FROM articles WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('database & indexing');
```

**Full-Text Index Features:**
- **Tokenization:** Breaks text into individual words (tokens)
- **Stemming:** Reduces words to their root form ("running" → "run")
- **Stop words:** Ignores common words ("the", "is", "and")
- **Ranking:** Scores results by relevance
- **Boolean operators:** AND, OR, NOT, phrase matching

**For serious full-text search at scale, use a dedicated search engine (Elasticsearch, Solr) rather than database full-text indexes.**

> **Ruby Context:** For PostgreSQL full-text search in Rails, the **pg_search** gem provides a clean interface. For Elasticsearch integration, use **searchkick** or **elasticsearch-rails**. Example with pg_search:
```ruby
class Article < ApplicationRecord
  include PgSearch::Model
  pg_search_scope :search_content,
                  against: { title: 'A', body: 'B' },
                  using: { tsearch: { prefix: true, dictionary: 'english' } }
end

Article.search_content('database indexing')
```

---

### Index Selectivity

**Selectivity** measures how well an index narrows down the result set. High selectivity = fewer matching rows = more useful index.

```
Selectivity = Number of distinct values / Total number of rows

Column: email (unique)     → Selectivity = 1,000,000 / 1,000,000 = 1.0  (perfect — every value is unique)
Column: status (3 values)  → Selectivity = 3 / 1,000,000 = 0.000003     (terrible — each value matches 333K rows)
Column: country (200 values) → Selectivity = 200 / 1,000,000 = 0.0002   (low — each value matches ~5K rows)
Column: created_at (dates) → Selectivity = 365 / 1,000,000 = 0.000365   (moderate)
```

**Rules:**
- **High selectivity** (close to 1.0): Index is very useful — use it
- **Low selectivity** (close to 0): Index may not help — the optimizer might prefer a full table scan
- **Boolean/status columns** with 2-3 values: Usually not worth indexing alone (but useful in composite indexes)
- The query optimizer decides whether to use an index based on selectivity and estimated row count

---

### When NOT to Index

Indexes are not free — they have costs:

| Cost | Description |
|------|-------------|
| **Write overhead** | Every INSERT, UPDATE, DELETE must also update all relevant indexes |
| **Storage** | Indexes consume disk space (sometimes as much as the table itself) |
| **Maintenance** | Indexes can become fragmented and need periodic rebuilding |
| **Optimizer confusion** | Too many indexes can confuse the query optimizer |

**Don't index when:**

| Scenario | Why |
|----------|-----|
| Small tables (< 1000 rows) | Full table scan is fast enough — index overhead isn't worth it |
| Columns with very low selectivity | Index on `gender` (M/F) scans half the table anyway |
| Columns that are rarely queried in WHERE/JOIN/ORDER BY | Index is never used but still maintained on writes |
| Tables with very high write:read ratio | Index maintenance overhead outweighs read benefits |
| Columns that are frequently updated | Every update must also update the index |
| Expressions or functions in WHERE | `WHERE YEAR(created_at) = 2025` can't use index on `created_at` (use `WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'` instead) |

**Index Monitoring:**

```sql
-- PostgreSQL: find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- MySQL: check if a query uses an index
EXPLAIN SELECT * FROM orders WHERE user_id = 1;
-- Look for: type=ref (index used), type=ALL (full table scan — bad!)

-- PostgreSQL: detailed query plan
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1;
```

> **Ruby Context:** Use `ActiveRecord::Base.connection.execute("EXPLAIN ANALYZE ...")` to check query plans. The **bullet** gem detects N+1 queries and missing indexes in development. The **pghero** gem provides a dashboard for PostgreSQL performance monitoring, including unused indexes and slow queries.

**The EXPLAIN Command:**

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1 AND status = 'active';

-- Output (PostgreSQL):
-- Index Scan using idx_orders_user_status on orders  (cost=0.43..8.45 rows=1 width=64) (actual time=0.023..0.025 rows=3 loops=1)
--   Index Cond: (user_id = 1 AND status = 'active')
-- Planning Time: 0.152 ms
-- Execution Time: 0.048 ms

-- Key things to look for:
-- Seq Scan         → full table scan (usually bad for large tables)
-- Index Scan       → using an index (good)
-- Index Only Scan  → covering index, no table lookup (best)
-- Bitmap Index Scan → combines multiple indexes (good for OR conditions)
-- Nested Loop      → for each row in outer, scan inner (OK for small inner)
-- Hash Join        → build hash table from smaller table, probe with larger (good for large joins)
-- Merge Join       → both inputs sorted, merge them (good for sorted data)
```

---


## 18.5 Transactions & ACID

> A **transaction** is a sequence of database operations that are treated as a single, indivisible unit of work. Either all operations succeed (commit) or all fail (rollback). Transactions are the foundation of data integrity in relational databases. Understanding ACID properties, isolation levels, and locking strategies is critical for building correct, concurrent systems.

---

### ACID Properties

| Property | Meaning | Example |
|----------|---------|---------|
| **Atomicity** | All operations in a transaction succeed or all fail — no partial execution | Transferring $100: debit AND credit both happen, or neither happens |
| **Consistency** | A transaction brings the database from one valid state to another valid state | Account balances can never be negative (if a constraint exists) |
| **Isolation** | Concurrent transactions don't interfere with each other | Two users buying the last item — only one succeeds |
| **Durability** | Once committed, the data survives crashes, power failures, etc. | After COMMIT, the data is written to disk (WAL) and won't be lost |

**Atomicity Example:**

```sql
-- Transfer $100 from account 1 to account 2
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- debit
-- What if the server crashes HERE?
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- credit
COMMIT;

-- With atomicity: if the crash happens between the two UPDATEs,
-- the entire transaction is rolled back — account 1 is NOT debited.
-- Without atomicity: account 1 loses $100, account 2 never receives it.
```

> **Ruby Context — ActiveRecord Transactions:**
```ruby
# ActiveRecord wraps this in BEGIN/COMMIT/ROLLBACK automatically:
ActiveRecord::Base.transaction do
  account1 = Account.lock.find(1)  # SELECT ... FOR UPDATE
  account2 = Account.lock.find(2)

  account1.update!(balance: account1.balance - 100)
  account2.update!(balance: account2.balance + 100)
  # If any exception is raised, the entire transaction is rolled back
end

# Nested transactions use SAVEPOINTs:
ActiveRecord::Base.transaction do
  user.save!
  ActiveRecord::Base.transaction(requires_new: true) do
    order.save!  # Uses SAVEPOINT — can roll back independently
  end
end
```

**Durability — Write-Ahead Log (WAL):**

Databases ensure durability using a **Write-Ahead Log (WAL)**:

```
1. Transaction begins
2. Changes are written to the WAL (sequential writes — fast)
3. COMMIT → WAL entry is flushed to disk (fsync)
4. Response sent to client ("committed!")
5. Later: changes are applied to the actual data pages (background process)

If crash happens after step 3:
  → On recovery, replay the WAL to reconstruct committed changes
  → Data is not lost

If crash happens before step 3:
  → Transaction was never committed
  → WAL entries are discarded on recovery
  → No partial changes
```

**Why WAL is fast:** Writing to the WAL is a **sequential append** (fast on both HDD and SSD). Writing to random data pages is **random I/O** (slow on HDD). The WAL lets the database acknowledge commits quickly and apply changes to data pages lazily.

---

### Isolation Levels

**Isolation** determines how much one transaction can see the changes made by other concurrent transactions. Higher isolation = more correctness but less concurrency (slower).

**The Three Read Phenomena:**

| Phenomenon | Description | Example |
|-----------|-------------|---------|
| **Dirty Read** | Transaction reads data written by another **uncommitted** transaction | T1 updates a row but hasn't committed. T2 reads the updated value. T1 rolls back. T2 has read data that never existed. |
| **Non-Repeatable Read** | Transaction reads the same row twice and gets **different values** | T1 reads a row. T2 updates and commits that row. T1 reads the same row again — different value. |
| **Phantom Read** | Transaction re-executes a query and gets **different rows** | T1 queries `WHERE age > 25` and gets 10 rows. T2 inserts a new row with age=30 and commits. T1 re-queries — now gets 11 rows. |

**Isolation Levels:**

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|----------------|-----------|-------------------|-------------|-------------|
| **Read Uncommitted** | Possible | Possible | Possible | Fastest |
| **Read Committed** | Prevented | Possible | Possible | Fast |
| **Repeatable Read** | Prevented | Prevented | Possible* | Moderate |
| **Serializable** | Prevented | Prevented | Prevented | Slowest |

*MySQL/InnoDB prevents phantom reads at Repeatable Read using gap locks. PostgreSQL's Repeatable Read also prevents phantoms via MVCC snapshots.

**Detailed Examples:**

```
--- Dirty Read (Read Uncommitted) ---

Time  Transaction 1                    Transaction 2
 T1   BEGIN;
 T2   UPDATE accounts SET balance=900
      WHERE id=1;  (was 1000)
 T3                                    BEGIN;
 T4                                    SELECT balance FROM accounts WHERE id=1;
                                       → Returns 900 (DIRTY READ — T1 hasn't committed!)
 T5   ROLLBACK;  (balance reverts to 1000)
 T6                                    -- T2 used balance=900, but it was never committed!
                                       -- This is a dirty read.

--- Non-Repeatable Read (Read Committed) ---

Time  Transaction 1                    Transaction 2
 T1   BEGIN;
 T2   SELECT balance FROM accounts
      WHERE id=1;  → Returns 1000
 T3                                    BEGIN;
 T4                                    UPDATE accounts SET balance=900 WHERE id=1;
 T5                                    COMMIT;
 T6   SELECT balance FROM accounts
      WHERE id=1;  → Returns 900
      -- Same query, different result! (Non-repeatable read)
 T7   COMMIT;

--- Phantom Read (Repeatable Read) ---

Time  Transaction 1                    Transaction 2
 T1   BEGIN;
 T2   SELECT COUNT(*) FROM orders
      WHERE user_id=1;  → Returns 5
 T3                                    BEGIN;
 T4                                    INSERT INTO orders (user_id, total) VALUES (1, 99.99);
 T5                                    COMMIT;
 T6   SELECT COUNT(*) FROM orders
      WHERE user_id=1;  → Returns 6
      -- Same query, different row count! (Phantom read)
 T7   COMMIT;
```

**Which Isolation Level to Use:**

| Level | Use Case |
|-------|----------|
| **Read Uncommitted** | Almost never — only for rough analytics where accuracy doesn't matter |
| **Read Committed** | Default for PostgreSQL, Oracle, SQL Server. Good for most OLTP workloads |
| **Repeatable Read** | Default for MySQL/InnoDB. Good when you need consistent reads within a transaction |
| **Serializable** | Financial transactions, inventory management — when correctness is critical and you can tolerate lower throughput |

> **Ruby Context — Setting isolation level in ActiveRecord:**
```ruby
# Per-transaction isolation level:
ActiveRecord::Base.transaction(isolation: :serializable) do
  # All queries in this block use SERIALIZABLE isolation
  balance = Account.find(1).balance
  Account.find(1).update!(balance: balance - 100)
end

# Supported values: :read_uncommitted, :read_committed, :repeatable_read, :serializable
```

---

### Optimistic vs Pessimistic Locking

**Pessimistic Locking** — assume conflicts will happen, lock the data upfront:

```sql
-- Lock the row when reading — no other transaction can modify it until we commit
BEGIN;
SELECT * FROM products WHERE id = 1 FOR UPDATE;  -- acquires exclusive lock
-- ... check stock, calculate price ...
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;  -- lock released

-- Any other transaction trying to SELECT FOR UPDATE on the same row will BLOCK until this commits
```

> **Ruby Context — Pessimistic Locking:**
```ruby
ActiveRecord::Base.transaction do
  product = Product.lock("FOR UPDATE").find(1)  # SELECT ... FOR UPDATE
  # Shorthand:
  product = Product.lock.find(1)

  if product.stock > 0
    product.update!(stock: product.stock - 1)
  else
    raise "Out of stock!"
  end
end
# Lock is released when the transaction commits/rolls back
```

**Optimistic Locking** — assume conflicts are rare, detect them at write time:

```sql
-- Read the row with its version number
SELECT id, name, stock, version FROM products WHERE id = 1;
-- Returns: id=1, name='Widget', stock=10, version=5

-- ... application logic (may take seconds) ...

-- Update only if the version hasn't changed
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;  -- version check!

-- If affected_rows = 1 → success (no one else modified it)
-- If affected_rows = 0 → conflict! Someone else updated it. Retry the entire operation.
```

> **Ruby Context — Optimistic Locking (built into ActiveRecord!):**
```ruby
# Just add a `lock_version` integer column (default: 0) to your table:
class AddLockVersionToProducts < ActiveRecord::Migration[7.1]
  def change
    add_column :products, :lock_version, :integer, default: 0, null: false
  end
end

# ActiveRecord handles optimistic locking automatically:
product = Product.find(1)          # lock_version = 5
product.stock -= 1
product.save!                      # UPDATE ... SET stock=9, lock_version=6 WHERE id=1 AND lock_version=5

# If another process updated the row first:
# Raises ActiveRecord::StaleObjectError — catch and retry!
begin
  product.save!
rescue ActiveRecord::StaleObjectError
  product.reload
  retry  # or handle the conflict
end
```

**Comparison:**

| Feature | Pessimistic Locking | Optimistic Locking |
|---------|--------------------|--------------------|
| Approach | Lock before reading | Check at write time |
| Conflict handling | Block other transactions | Detect and retry |
| Best for | High contention (many writers on same data) | Low contention (rare conflicts) |
| Throughput | Lower (transactions wait for locks) | Higher (no waiting, but retries on conflict) |
| Deadlock risk | Yes (two transactions waiting for each other's locks) | No (no locks held) |
| Implementation | `SELECT ... FOR UPDATE` | Version column + conditional UPDATE |
| Starvation risk | Yes (long-running transactions block others) | Yes (high contention → constant retries) |

---

### MVCC (Multi-Version Concurrency Control)

**MVCC** is the mechanism used by PostgreSQL, MySQL/InnoDB, Oracle, and most modern databases to provide isolation without excessive locking. Instead of locking rows, MVCC keeps **multiple versions** of each row and shows each transaction the version that was current when the transaction started.

```
Row: id=1, balance=1000

Time  Transaction 1 (snapshot at T1)    Transaction 2 (snapshot at T3)
 T1   BEGIN; (sees version at T1)
 T2   SELECT balance → 1000
 T3                                      BEGIN; (sees version at T3)
 T4                                      UPDATE balance = 900 WHERE id=1;
                                         (creates NEW version: balance=900)
                                         (old version: balance=1000 still exists)
 T5                                      COMMIT;
 T6   SELECT balance → 1000             
      (still sees the OLD version — snapshot isolation!)
 T7   COMMIT;

Database internally:
  Version 1: balance=1000, created_at=T0, expired_at=T5  (visible to T1)
  Version 2: balance=900,  created_at=T5, expired_at=∞   (visible to new transactions)
```

**How MVCC Works:**
- Each row has hidden columns: `created_transaction_id` and `expired_transaction_id` (or `xmin`/`xmax` in PostgreSQL)
- When a row is updated, the old version is marked as expired and a new version is created
- Each transaction has a snapshot — it can only see versions created before its snapshot and not yet expired
- **Readers don't block writers, writers don't block readers** — this is the key benefit

**MVCC Garbage Collection (Vacuuming):**
- Old row versions accumulate over time (dead tuples)
- A background process (VACUUM in PostgreSQL, purge thread in MySQL) cleans up old versions that are no longer visible to any active transaction
- If vacuuming falls behind, the table bloats (more disk space, slower scans)
- PostgreSQL's `autovacuum` handles this automatically, but high-write tables may need tuning

---

### Two-Phase Locking (2PL)

**Two-Phase Locking** is a concurrency control protocol that guarantees **serializability** (the strongest isolation level). It has two phases:

```
Phase 1: Growing Phase
  Transaction acquires locks as needed
  Can acquire new locks
  Cannot release any locks

Phase 2: Shrinking Phase
  Transaction releases locks
  Cannot acquire any new locks

         Locks held
           ^
           |     /\
           |    /  \
           |   /    \
           |  /      \
           | /        \
           |/          \
           +------------+--→ Time
           Growing  Shrinking
           Phase    Phase
```

**Lock Types:**

| Lock | Also Called | Allows | Blocks |
|------|-----------|--------|--------|
| **Shared Lock (S)** | Read lock | Other shared locks | Exclusive locks |
| **Exclusive Lock (X)** | Write lock | Nothing | All other locks |

**Lock Compatibility Matrix:**

|  | Shared (S) | Exclusive (X) |
|--|-----------|--------------|
| **Shared (S)** | ✅ Compatible | ❌ Conflict |
| **Exclusive (X)** | ❌ Conflict | ❌ Conflict |

- Multiple transactions can hold shared locks on the same row (concurrent reads)
- Only one transaction can hold an exclusive lock (exclusive write access)
- A shared lock blocks exclusive locks (readers block writers)
- An exclusive lock blocks everything (writer blocks readers and other writers)

**Strict 2PL:** All locks are held until the transaction commits or rolls back (not released during the shrinking phase). This is what most databases actually implement — it prevents cascading rollbacks.

---

### Deadlock Detection and Prevention

A **deadlock** occurs when two or more transactions are each waiting for a lock held by the other, creating a circular dependency.

```
Transaction 1:                    Transaction 2:
  BEGIN;                            BEGIN;
  LOCK row A (exclusive)            LOCK row B (exclusive)
  ...                               ...
  LOCK row B → BLOCKED!             LOCK row A → BLOCKED!
  (waiting for T2 to release B)     (waiting for T1 to release A)
  
  → DEADLOCK! Neither can proceed.
```

**Deadlock Detection:**
- Most databases run a **deadlock detector** periodically (or on every lock wait)
- It builds a **wait-for graph** — if a cycle is detected, one transaction is chosen as the **victim** and rolled back
- The victim is typically the transaction that has done the least work (cheapest to retry)
- PostgreSQL and MySQL detect deadlocks automatically and return an error to the victim

```sql
-- MySQL deadlock error:
-- ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction

-- Application must catch this error and RETRY the transaction
```

> **Ruby Context — Handling deadlocks:**
```ruby
# ActiveRecord raises ActiveRecord::Deadlocked on deadlock detection
MAX_RETRIES = 3

def transfer_funds(from_id, to_id, amount)
  retries = 0
  begin
    ActiveRecord::Base.transaction do
      from = Account.lock.find(from_id)
      to = Account.lock.find(to_id)
      from.update!(balance: from.balance - amount)
      to.update!(balance: to.balance + amount)
    end
  rescue ActiveRecord::Deadlocked => e
    retries += 1
    raise if retries >= MAX_RETRIES
    sleep(rand * 0.1 * retries)  # exponential backoff with jitter
    retry
  end
end

# Prevention: always lock in consistent order (e.g., by ID)
def transfer_funds_safe(from_id, to_id, amount)
  first_id, second_id = [from_id, to_id].sort
  ActiveRecord::Base.transaction do
    accounts = Account.lock.where(id: [first_id, second_id]).order(:id).to_a
    # Now locked in consistent order — no deadlock possible
    from = accounts.find { |a| a.id == from_id }
    to = accounts.find { |a| a.id == to_id }
    from.update!(balance: from.balance - amount)
    to.update!(balance: to.balance + amount)
  end
end
```

**Deadlock Prevention Strategies:**

| Strategy | Description |
|----------|-------------|
| **Lock ordering** | Always acquire locks in the same order (e.g., by primary key ascending) |
| **Lock timeout** | Set a maximum wait time for locks — give up and retry if exceeded |
| **Reduce lock scope** | Lock only the rows you need, not entire tables |
| **Reduce lock duration** | Keep transactions short — commit as quickly as possible |
| **Optimistic locking** | Avoid locks entirely — use version-based conflict detection |
| **Avoid user interaction in transactions** | Never wait for user input while holding locks |

**Application-Level Best Practices:**

```
1. Keep transactions SHORT — do all computation outside the transaction
2. Access tables and rows in a CONSISTENT ORDER across all transactions
3. Use the LOWEST isolation level that meets your requirements
4. RETRY transactions that fail due to deadlocks (with exponential backoff)
5. Use SELECT ... FOR UPDATE only when you actually need to modify the row
6. Avoid long-running transactions — they hold locks and block others
7. Monitor for deadlocks and slow queries in production
```

---


## 18.6 Popular SQL Databases

> Choosing the right SQL database depends on your workload, scale, team expertise, and ecosystem. Each database has different strengths, storage engines, replication models, and operational characteristics. This section covers the most popular relational databases and when to use each.

---

### PostgreSQL

**PostgreSQL** is the most advanced open-source relational database. It's known for correctness, extensibility, and rich feature set.

**Key Features:**

| Feature | Details |
|---------|---------|
| **MVCC** | True MVCC with snapshot isolation; readers never block writers |
| **Data types** | JSON/JSONB, arrays, hstore, range types, geometric types, inet, UUID, custom types |
| **Indexing** | B-Tree, Hash, GIN (full-text, JSON), GiST (geometric, range), BRIN (large sequential data), SP-GiST |
| **Full-text search** | Built-in with tsvector/tsquery, ranking, stemming |
| **Partitioning** | Declarative partitioning (range, list, hash) since v10 |
| **Replication** | Streaming replication (async/sync), logical replication |
| **Extensions** | PostGIS (geospatial), TimescaleDB (time-series), pg_trgm (fuzzy search), Citus (distributed) |
| **ACID compliance** | Full ACID with all isolation levels including true Serializable (SSI) |
| **JSON support** | JSONB with indexing (GIN), JSON path queries, JSON aggregation |
| **CTEs** | Full support including recursive CTEs, writable CTEs |
| **Window functions** | Full support |
| **Materialized views** | Supported with concurrent refresh |
| **Default isolation** | Read Committed |

**When to Use PostgreSQL:**
- Complex queries with many JOINs, CTEs, window functions
- Need advanced data types (JSON, arrays, geospatial)
- Data integrity is critical (strongest ACID compliance)
- Full-text search without a separate search engine
- Geospatial applications (PostGIS)
- You need extensibility (custom types, functions, operators)

> **Ruby Context:** PostgreSQL is the **most popular database in the Rails ecosystem**. The `pg` gem provides the driver, and ActiveRecord has excellent PostgreSQL-specific support: JSONB columns, array columns, range types, UUID primary keys, and full-text search via `pg_search`. Rails generators default to PostgreSQL on many platforms. Heroku, Render, and most Rails hosting providers use PostgreSQL.

---

### MySQL / MariaDB

**MySQL** is the most popular open-source database by deployment count. **MariaDB** is a community fork of MySQL that maintains compatibility while adding features.

**Key Features:**

| Feature | Details |
|---------|---------|
| **Storage engines** | InnoDB (default, ACID), MyISAM (legacy, no transactions), Memory, Archive |
| **Replication** | Master-slave, group replication, MySQL Cluster (NDB) |
| **Partitioning** | Range, list, hash, key partitioning |
| **JSON support** | JSON data type with functions (less powerful than PostgreSQL's JSONB) |
| **Full-text search** | InnoDB full-text indexes (basic) |
| **Default isolation** | Repeatable Read (InnoDB) |
| **MVCC** | InnoDB uses MVCC with gap locking for phantom prevention |
| **Clustering** | MySQL Group Replication, MySQL InnoDB Cluster, Vitess (sharding) |

**InnoDB vs MyISAM:**

| Feature | InnoDB | MyISAM |
|---------|--------|--------|
| Transactions | Yes (ACID) | No |
| Row-level locking | Yes | Table-level locking only |
| Foreign keys | Yes | No |
| Crash recovery | Yes (WAL) | No (corrupt on crash) |
| Full-text search | Yes (since 5.6) | Yes |
| Use case | Everything (default) | Legacy, read-only archives |

**When to Use MySQL:**
- Simple CRUD applications with high read throughput
- Web applications (LAMP stack — Linux, Apache, MySQL, PHP)
- When you need a large ecosystem of tools and hosting options
- Read-heavy workloads with read replicas
- When team expertise is in MySQL

> **Ruby Context:** MySQL is supported via the `mysql2` gem. ActiveRecord works well with MySQL, though some PostgreSQL-specific features (arrays, JSONB indexing) aren't available. Shopify, one of the largest Rails applications, runs on MySQL. Use `rails new myapp -d mysql` to scaffold a MySQL-backed Rails app.

---

### Oracle Database

**Oracle** is the enterprise-grade commercial database with the most features and the highest price tag.

**Key Features:**

| Feature | Details |
|---------|---------|
| **RAC (Real Application Clusters)** | Multiple servers share a single database — active-active clustering |
| **Partitioning** | Range, list, hash, composite, interval, reference partitioning |
| **Advanced compression** | Table, index, and backup compression |
| **Flashback** | Query data as it existed at a past point in time |
| **Data Guard** | Standby databases for disaster recovery |
| **PL/SQL** | Powerful procedural language for stored procedures |
| **Materialized views** | With automatic query rewrite |
| **Advanced security** | Transparent Data Encryption (TDE), Virtual Private Database, Label Security |

**When to Use Oracle:**
- Enterprise environments with existing Oracle investments
- Need RAC for high availability without application changes
- Complex stored procedures and PL/SQL logic
- Regulatory requirements that mandate Oracle-specific features
- Budget is not a constraint

> **Ruby Context:** Oracle is supported via the `ruby-oci8` gem and the `activerecord-oracle_enhanced-adapter`. It's less common in the Ruby ecosystem but used in enterprise Rails deployments that integrate with existing Oracle infrastructure.

---

### SQL Server (Microsoft)

**SQL Server** is Microsoft's enterprise database, tightly integrated with the Windows/.NET ecosystem.

**Key Features:**

| Feature | Details |
|---------|---------|
| **T-SQL** | Microsoft's SQL dialect with procedural extensions |
| **Always On** | High availability with automatic failover |
| **Columnstore indexes** | Columnar storage for analytics (OLAP) |
| **In-memory OLTP** | Memory-optimized tables for extreme performance |
| **Integration Services (SSIS)** | ETL tool for data integration |
| **Reporting Services (SSRS)** | Built-in reporting |
| **Analysis Services (SSAS)** | OLAP cubes and data mining |
| **Azure SQL** | Fully managed cloud version |

**When to Use SQL Server:**
- .NET / Windows ecosystem
- Need integrated BI tools (SSIS, SSRS, SSAS)
- Enterprise environments with Microsoft licensing
- Hybrid cloud with Azure SQL

> **Ruby Context:** SQL Server is supported via the `activerecord-sqlserver-adapter` gem with the `tiny_tds` driver. It's uncommon in Ruby projects but available when integrating with Microsoft-centric environments.

---

### SQLite

**SQLite** is a serverless, file-based database embedded directly into the application. It's the most deployed database in the world (every smartphone, browser, and many applications use it).

**Key Features:**

| Feature | Details |
|---------|---------|
| **Serverless** | No separate server process — database is a single file |
| **Zero configuration** | No setup, no administration |
| **Self-contained** | Single C library, no dependencies |
| **File-based** | Entire database is one file (easy to copy, backup, share) |
| **ACID compliant** | Full ACID with WAL mode |
| **Size limit** | Up to 281 TB (practical limit: a few GB for best performance) |
| **Concurrency** | Single writer at a time (readers don't block) |

**When to Use SQLite:**
- Mobile applications (iOS, Android)
- Desktop applications (embedded database)
- Testing and prototyping (in-memory mode)
- Small websites with low traffic
- Edge computing and IoT devices
- Configuration storage
- **NOT for:** High-concurrency web servers, multi-user applications, distributed systems

> **Ruby Context:** SQLite is the **default database for new Rails apps** (`rails new myapp` uses SQLite). The `sqlite3` gem provides the driver. It's perfect for development and testing. Rails 8 has improved SQLite support significantly, making it viable for small production deployments. For production at scale, switch to PostgreSQL or MySQL. The **Litestack** gem bundles SQLite-based solutions for caching, jobs, and more — a full Rails stack on SQLite.

---

### Database Comparison Summary

| Feature | PostgreSQL | MySQL | Oracle | SQL Server | SQLite |
|---------|-----------|-------|--------|-----------|--------|
| **License** | Open source (PostgreSQL License) | Open source (GPL) / Commercial | Commercial | Commercial | Public domain |
| **ACID** | Full | Full (InnoDB) | Full | Full | Full |
| **MVCC** | Yes | Yes (InnoDB) | Yes | Yes | WAL mode |
| **JSON support** | Excellent (JSONB) | Good | Good | Good | Basic |
| **Full-text search** | Good (built-in) | Basic | Good | Good | Basic (FTS5) |
| **Replication** | Streaming + Logical | Master-Slave + Group | Data Guard + RAC | Always On | None |
| **Partitioning** | Declarative | Range/List/Hash | Advanced | Yes | None |
| **Max DB size** | Unlimited | Unlimited | Unlimited | 524 PB | 281 TB |
| **Concurrency** | Excellent | Good | Excellent | Excellent | Limited (single writer) |
| **Cloud managed** | AWS RDS/Aurora, GCP Cloud SQL, Azure | AWS RDS/Aurora, GCP, Azure | Oracle Cloud, AWS RDS | Azure SQL, AWS RDS | N/A (embedded) |
| **Best for** | Complex queries, data integrity, extensibility | Web apps, read-heavy, simplicity | Enterprise, HA, complex workloads | .NET ecosystem, BI | Embedded, mobile, testing |
| **Ruby gem** | `pg` | `mysql2` | `ruby-oci8` | `tiny_tds` | `sqlite3` |

**Decision Framework:**

| Question | Recommendation |
|----------|---------------|
| Need advanced data types and complex queries? | PostgreSQL |
| Simple web app, LAMP stack, read-heavy? | MySQL |
| Enterprise with existing Oracle/Microsoft investment? | Oracle / SQL Server |
| Embedded in mobile/desktop app? | SQLite |
| Need the best open-source option? | PostgreSQL |
| Need the largest community and hosting options? | MySQL |
| Budget is unlimited, need maximum features? | Oracle |
| .NET ecosystem with BI needs? | SQL Server |
| **Building a Rails application?** | **PostgreSQL (strongly recommended)** |

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Primary Key | Use surrogate keys (BIGINT auto-increment or UUID v7); natural keys as unique constraints |
| Foreign Key | Enforces referential integrity; use in monoliths, skip in microservices (app-level consistency) |
| Constraints | Enforce data integrity at the database level — NOT NULL, UNIQUE, CHECK, DEFAULT |
| Data Types | Use DECIMAL for money (never FLOAT), TIMESTAMP for dates, smallest type that fits |
| JOINs | INNER (matching only), LEFT (all left + matching right), know when to use each |
| CTEs | Named temporary result sets — more readable than subqueries, support recursion |
| Window Functions | ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD — calculations across related rows without GROUP BY |
| Query Execution Order | FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT |
| Normalization | 1NF (atomic), 2NF (no partial deps), 3NF (no transitive deps) — aim for 3NF |
| Denormalization | Trade write complexity for read performance — use for read-heavy workloads, reporting |
| B+ Tree Index | Default index; O(log n) lookups; supports range queries and sorting |
| Composite Index | Column order matters — leftmost prefix rule; equality columns first, then range |
| Covering Index | Contains all query columns — eliminates table lookups (index-only scan) |
| EXPLAIN | Always check query plans — look for Seq Scan (bad) vs Index Scan (good) |
| ACID | Atomicity (all or nothing), Consistency (valid states), Isolation (no interference), Durability (survives crashes) |
| Isolation Levels | Read Committed (PostgreSQL default), Repeatable Read (MySQL default), Serializable (strongest) |
| MVCC | Multiple row versions — readers don't block writers; foundation of modern database concurrency |
| Optimistic Locking | Version column + conditional UPDATE — best for low contention |
| Pessimistic Locking | SELECT FOR UPDATE — best for high contention |
| Deadlocks | Circular lock dependencies — prevented by consistent lock ordering, short transactions |
| PostgreSQL | Best open-source option — advanced types, strong ACID, extensible |
| MySQL | Most popular — simple, fast reads, huge ecosystem |

> **Ruby Ecosystem Summary:**
> | Tool | Purpose |
> |------|---------|
> | **ActiveRecord** | Default ORM — migrations, validations, associations, query building |
> | **Sequel** | Alternative ORM — more SQL-like, powerful dataset API |
> | **pg** / **mysql2** / **sqlite3** | Database drivers for direct connections |
> | **scenic** | Database views and materialized views in Rails |
> | **pg_search** | PostgreSQL full-text search integration |
> | **searchkick** | Elasticsearch integration for Rails |
> | **bullet** | N+1 query detection in development |
> | **pghero** | PostgreSQL performance dashboard |
> | **strong_migrations** | Catch unsafe migrations before they run |
> | **money-rails** | Money/currency handling with BigDecimal |
> | **ancestry** / **closure_tree** | Hierarchical data (tree structures) in ActiveRecord |

---

## Interview Tips for Module 18

1. **Design a schema** — be ready to design tables, keys, and relationships for any system (e-commerce, social network, booking system)
2. **Write SQL** — JOINs, GROUP BY, window functions, CTEs — practice writing queries by hand
3. **Indexing strategy** — given a set of queries, design the right indexes; explain the leftmost prefix rule
4. **EXPLAIN** — know how to read a query plan and identify performance problems
5. **Normalization** — explain 1NF through 3NF with examples; know when to denormalize
6. **ACID** — explain each property with a concrete example (bank transfer is the classic)
7. **Isolation levels** — know the three read phenomena and which isolation level prevents each
8. **Optimistic vs Pessimistic locking** — explain both with code examples; know when to use each
9. **MVCC** — explain how it works and why readers don't block writers
10. **Deadlocks** — explain how they happen and how to prevent them
11. **PostgreSQL vs MySQL** — know the key differences and when to recommend each
12. **Denormalization** — explain CQRS pattern: normalize for writes, denormalize for reads

> **Ruby-Specific Interview Tips:**
> - Know how ActiveRecord maps to SQL — be able to explain what SQL a given ActiveRecord query generates
> - Understand `lock_version` (optimistic) vs `.lock` / `FOR UPDATE` (pessimistic) in ActiveRecord
> - Be familiar with Rails migrations and how they map to DDL statements
> - Know how to use `includes` (eager loading) vs `joins` vs `left_joins` and when each is appropriate
> - Understand N+1 queries and how `includes`, `preload`, and `eager_load` solve them
> - Be able to drop to raw SQL with `find_by_sql` or `connection.execute` when ActiveRecord isn't enough
