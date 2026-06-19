# PostgreSQL — Complete Notes (Basics → Advanced + Interview Prep)

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [PostgreSQL vs MySQL vs Other Databases](#2-postgresql-vs-mysql-vs-other-databases)
3. [PostgreSQL Architecture](#3-postgresql-architecture)
4. [Installation & Setup](#4-installation--setup)
5. [Databases, Schemas & Tables](#5-databases-schemas--tables)
6. [Data Types](#6-data-types)
7. [Constraints](#7-constraints)
8. [CRUD — SQL Basics](#8-crud--sql-basics)
9. [Filtering, Sorting & Aggregation](#9-filtering-sorting--aggregation)
10. [Joins](#10-joins)
11. [Subqueries & CTEs](#11-subqueries--ctes)
12. [Recursive Queries](#12-recursive-queries)
13. [Indexes](#13-indexes)
14. [Keys](#14-keys)
15. [Normalization](#15-normalization)
16. [Transactions & ACID](#16-transactions--acid)
17. [MVCC — Multi-Version Concurrency Control](#17-mvcc)
18. [VACUUM & Autovacuum](#18-vacuum--autovacuum)
19. [Isolation Levels](#19-isolation-levels)
20. [Locking](#20-locking)
21. [Views & Materialized Views](#21-views--materialized-views)
22. [Functions, Procedures & Triggers](#22-functions-procedures--triggers)
23. [JSON & JSONB](#23-json--jsonb)
24. [Arrays & Other Rich Types](#24-arrays--other-rich-types)
25. [Window Functions](#25-window-functions)
26. [Query Optimization & EXPLAIN](#26-query-optimization--explain)
27. [Partitioning](#27-partitioning)
28. [Replication](#28-replication)
29. [Backup & Recovery](#29-backup--recovery)
30. [Extensions](#30-extensions)
31. [Security & Roles](#31-security--roles)
32. [Troubleshooting & Performance](#32-troubleshooting--performance)
33. [Best Practices Summary](#33-best-practices-summary)
34. [Cheat Sheet](#34-cheat-sheet)
35. [Interview Questions & Answers](#35-interview-questions--answers)

---

## 1. Introduction

**PostgreSQL** ("Postgres") is a powerful, open-source **object-relational database management system (ORDBMS)** known for standards compliance, extensibility, and advanced features beyond what traditional RDBMSs offer — strong support for complex queries, JSON, full-text search, custom types, and a robust extension ecosystem.

**Key characteristics:**
- Fully ACID-compliant
- Highly extensible — custom data types, operators, functions, even custom index types
- Rich native data types: arrays, JSON/JSONB, hstore, ranges, UUID, geometric types
- MVCC-based concurrency (no read locks blocking writers)
- Strong standards (SQL) compliance
- Advanced indexing: B-Tree, Hash, GIN, GiST, BRIN, SP-GiST
- Native support for table inheritance, partitioning, foreign data wrappers (querying external data sources via SQL)

---

## 2. PostgreSQL vs MySQL vs Other Databases

| Aspect | PostgreSQL | MySQL | MongoDB (NoSQL) |
|---|---|---|---|
| Type | Object-relational | Relational | Document-based |
| Standards compliance | Very high | Moderate | N/A |
| JSON support | Native `JSONB` (indexable, binary) | `JSON` type (less indexable) | Native (it's the core model) |
| Extensibility | Very high (custom types, extensions, e.g. PostGIS) | Limited | N/A |
| Concurrency model | MVCC, no read locks | MVCC (InnoDB), but historically more locking-prone | Varies |
| Full-text search | Built-in, advanced | Basic built-in | Built-in (Atlas Search separate) |
| Replication | Streaming + logical | Binlog-based | Replica sets |
| Best for | Complex queries, data integrity, analytics, geospatial (PostGIS), feature richness | Simplicity, read-heavy web apps, fast simple lookups | Flexible/evolving schema, horizontal scale |

**Key interview point:** MySQL has historically been favored for simplicity and raw read speed in simple web workloads; PostgreSQL is favored when you need advanced SQL features, strict standards compliance, complex queries/analytics, JSON with real indexing, or geospatial capability (PostGIS). The performance gap has narrowed significantly in recent years — the choice today is often driven more by feature needs and ecosystem than raw speed.

---

## 3. PostgreSQL Architecture

```
            Client (psql, app, pgAdmin)
                       │
                       ▼
            ┌─────────────────────┐
            │   Postmaster (main)    │   listens for connections
            └──────────┬──────────┘
                       │ forks one process per connection
        ┌──────────────┼──────────────┐
        ▼               ▼               ▼
  ┌───────────┐   ┌───────────┐   ┌───────────┐
  │ Backend     │   │ Backend     │   │ Backend     │   (one OS process per client connection)
  │ Process 1    │   │ Process 2    │   │ Process 3    │
  └───────────┘   └───────────┘   └───────────┘
        │               │               │
        └───────────────┼───────────────┘
                        ▼
            ┌─────────────────────┐
            │   Shared Buffers        │   (in-memory page cache)
            │   WAL Buffers             │   (write-ahead log buffer)
            └──────────┬──────────┘
                       ▼
            ┌─────────────────────┐
            │   Disk: Data files,     │
            │   WAL files, etc.          │
            └─────────────────────┘
```

**Key processes (background workers):**
- **Postmaster** — the main supervisor process; listens for connections and forks a new backend process per client.
- **Backend processes** — one per client connection (unlike MySQL's thread-per-connection model, PostgreSQL traditionally uses a **process**-per-connection model — heavier per connection, which is why connection pooling, e.g., PgBouncer, is commonly used).
- **WAL Writer** — flushes Write-Ahead Log records to disk.
- **Checkpointer** — periodically flushes dirty shared buffer pages to disk, creating a checkpoint.
- **Autovacuum launcher/workers** — automatically run `VACUUM`/`ANALYZE` to reclaim dead row space and update planner statistics.
- **Background writer** — incrementally writes dirty pages to reduce I/O spikes at checkpoint time.

**Write-Ahead Logging (WAL):** Before any data change is written to the actual data files, it's first written to the WAL — this is the foundation of crash recovery, durability, and physical replication.

---

## 4. Installation & Setup

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install postgresql postgresql-contrib

# Docker
docker run -d --name postgres -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres:16

# Connect
sudo -u postgres psql
psql -h localhost -U myuser -d mydb
```

```sql
\l              -- list databases
\c mydb         -- connect to a database
\dt             -- list tables
\d table_name   -- describe a table
\du             -- list roles/users
\q              -- quit
SELECT version();
```

---

## 5. Databases, Schemas & Tables

PostgreSQL has an extra organizational layer compared to MySQL: **Database → Schema → Table** (MySQL effectively treats "database" and "schema" as the same thing).

```sql
CREATE DATABASE company;
\c company

CREATE SCHEMA hr;
CREATE SCHEMA sales;

CREATE TABLE hr.employees (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    department_id INT,
    salary NUMERIC(10,2),
    hire_date DATE DEFAULT CURRENT_DATE
);

SET search_path TO hr, public;    -- controls unqualified table name resolution order
SELECT * FROM employees;           -- resolves to hr.employees given the search_path above

ALTER TABLE hr.employees ADD COLUMN phone VARCHAR(20);
ALTER TABLE hr.employees ALTER COLUMN salary TYPE NUMERIC(12,2);
ALTER TABLE hr.employees DROP COLUMN phone;
ALTER TABLE hr.employees RENAME TO staff;
DROP TABLE hr.staff;
TRUNCATE TABLE hr.employees RESTART IDENTITY CASCADE;
```

**Why schemas matter (interview point):** Schemas let you namespace tables within a single database — useful for multi-tenant designs, logically separating modules (e.g., `hr`, `sales`, `audit`) without needing separate databases, and applying different permission sets per schema.

---

## 6. Data Types

| Category | Types | Notes |
|---|---|---|
| **Numeric** | `INTEGER`, `SMALLINT`, `BIGINT`, `NUMERIC(p,s)`, `REAL`, `DOUBLE PRECISION`, `SERIAL`/`BIGSERIAL` (auto-increment) | `NUMERIC` for exact precision; `SERIAL` creates an implicit sequence |
| **String** | `VARCHAR(n)`, `CHAR(n)`, `TEXT` | `TEXT` has no length limit and performs the same as `VARCHAR` internally — often preferred over arbitrary `VARCHAR(n)` limits |
| **Date/Time** | `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL` | `TIMESTAMPTZ` (timestamp with time zone) is generally recommended — stores in UTC internally, converts on display |
| **Boolean** | `BOOLEAN` | True native boolean type (`TRUE`/`FALSE`/`NULL`) — unlike MySQL |
| **JSON** | `JSON`, `JSONB` | `JSONB` is binary, indexable, and generally preferred |
| **Array** | Any type can be an array, e.g., `INTEGER[]`, `TEXT[]` | Native array support, unlike MySQL |
| **UUID** | `UUID` | Native type, often used as primary keys |
| **Range types** | `INT4RANGE`, `TSRANGE`, `DATERANGE`, etc. | Represent a range of values with built-in overlap/containment operators |
| **Geometric** | `POINT`, `LINE`, `POLYGON`, `CIRCLE` | Built-in; PostGIS extension adds full GIS capability |
| **Network** | `INET`, `CIDR`, `MACADDR` | Native IP/network address types |
| **Other** | `HSTORE` (key-value pairs), `XML`, `MONEY`, `ENUM` (custom) | |

```sql
CREATE TYPE mood AS ENUM ('happy', 'sad', 'neutral');
CREATE TABLE people (name TEXT, current_mood mood);
```

**`TIMESTAMP` vs `TIMESTAMPTZ` (interview point):** `TIMESTAMP` stores a naive date/time with no timezone awareness. `TIMESTAMPTZ` stores the instant in UTC internally and converts to the client's session timezone on display — generally the recommended choice to avoid timezone bugs in distributed/multi-region applications.

---

## 7. Constraints

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    total NUMERIC(10,2) CHECK (total >= 0),
    status VARCHAR(20) DEFAULT 'pending',
    email VARCHAR(100) UNIQUE,
    CONSTRAINT valid_status CHECK (status IN ('pending', 'shipped', 'cancelled'))
);

-- Named constraints can be added/dropped independently
ALTER TABLE orders ADD CONSTRAINT chk_total_positive CHECK (total > 0);
ALTER TABLE orders DROP CONSTRAINT chk_total_positive;

-- EXCLUSION constraint (Postgres-specific — prevents overlapping ranges, etc.)
CREATE TABLE bookings (
    room_id INT,
    during TSRANGE,
    EXCLUDE USING GIST (room_id WITH =, during WITH &&)   -- no two bookings for same room can overlap
);
```

**`EXCLUDE` constraints** are a PostgreSQL-specific feature (not in standard SQL/MySQL) — they generalize uniqueness to arbitrary operators, commonly used to prevent overlapping time ranges (e.g., room booking conflicts) directly at the database level.

---

## 8. CRUD — SQL Basics

```sql
-- CREATE
INSERT INTO employees (first_name, last_name, email, salary)
VALUES ('Jane', 'Doe', 'jane@example.com', 75000)
RETURNING id;                            -- Postgres-specific: get back generated values immediately

-- READ
SELECT * FROM employees;
SELECT first_name, salary FROM employees WHERE department_id = 3;

-- UPDATE
UPDATE employees SET salary = salary * 1.1 WHERE department_id = 3
RETURNING id, salary;

-- DELETE
DELETE FROM employees WHERE id = 5 RETURNING *;

-- UPSERT (insert or update on conflict)
INSERT INTO employees (id, email, salary)
VALUES (1, 'jane@example.com', 80000)
ON CONFLICT (id) DO UPDATE SET salary = EXCLUDED.salary;
```

**`RETURNING` clause (Postgres-specific):** Lets `INSERT`/`UPDATE`/`DELETE` return the affected rows' values directly, avoiding a separate `SELECT` round-trip — very useful for getting auto-generated IDs or confirming exactly what changed.

**`ON CONFLICT` (UPSERT):** PostgreSQL's equivalent of MySQL's `ON DUPLICATE KEY UPDATE`, but more expressive — can target a specific constraint, do nothing (`DO NOTHING`), or update using `EXCLUDED` to reference the row that would have been inserted.

---

## 9. Filtering, Sorting & Aggregation

```sql
SELECT * FROM employees WHERE salary > 50000 AND department_id = ANY(ARRAY[1,2,3]);
SELECT * FROM employees WHERE last_name ILIKE 's%';     -- case-insensitive LIKE (Postgres-specific)
SELECT * FROM employees WHERE hire_date BETWEEN '2023-01-01' AND '2023-12-31';
SELECT * FROM employees WHERE email IS NULL;
SELECT DISTINCT ON (department_id) * FROM employees ORDER BY department_id, salary DESC;  -- Postgres-specific
SELECT * FROM employees ORDER BY salary DESC LIMIT 10 OFFSET 20;

SELECT department_id, COUNT(*) AS emp_count, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5
ORDER BY avg_salary DESC;
```

**`DISTINCT ON` (Postgres-specific):** Returns the first row (per `ORDER BY`) for each distinct value of the specified expression — a concise way to get, e.g., "the highest-paid employee per department" without a subquery or window function.

**`ILIKE`:** Case-insensitive pattern match — `LIKE` is case-sensitive in Postgres by default (unlike MySQL, where default collations are often case-insensitive).

---

## 10. Joins

```sql
SELECT e.first_name, d.name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;

SELECT e.first_name, d.name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;

SELECT e.first_name, d.name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;

-- FULL OUTER JOIN — natively supported in PostgreSQL (unlike MySQL!)
SELECT e.first_name, d.name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.id;

-- LATERAL join (Postgres feature) — allows a subquery to reference columns from preceding FROM items
SELECT e.name, top_sales.amount
FROM employees e,
LATERAL (
    SELECT amount FROM sales s WHERE s.employee_id = e.id ORDER BY amount DESC LIMIT 1
) top_sales;
```

**`FULL OUTER JOIN` (key difference from MySQL):** PostgreSQL natively supports `FULL OUTER JOIN` — returns all rows from both tables, with NULLs where there's no match on either side. MySQL has no native `FULL JOIN` and requires a `UNION` of `LEFT` and `RIGHT` joins to emulate it.

**`LATERAL` joins:** Allow a subquery in the `FROM` clause to reference columns from earlier tables in the same `FROM` list — useful for "top N per group" queries without window functions.

---

## 11. Subqueries & CTEs

```sql
-- Scalar / IN / correlated subqueries work the same as standard SQL
SELECT name FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);

-- CTE (Common Table Expression)
WITH dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT e.name, e.salary, d.avg_salary
FROM employees e
JOIN dept_avg d ON e.department_id = d.department_id
WHERE e.salary > d.avg_salary;

-- Multiple CTEs
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 80000
),
dept_counts AS (
    SELECT department_id, COUNT(*) FROM high_earners GROUP BY department_id
)
SELECT * FROM dept_counts;

-- Writable CTEs — Postgres allows INSERT/UPDATE/DELETE inside a CTE
WITH deleted_rows AS (
    DELETE FROM employees WHERE department_id = 5 RETURNING *
)
INSERT INTO archived_employees SELECT * FROM deleted_rows;
```

**Writable CTEs (Postgres-specific):** Unlike MySQL, PostgreSQL allows `INSERT`/`UPDATE`/`DELETE` statements inside a CTE (using `RETURNING`), enabling powerful "move data from one table to another" patterns in a single statement.

---

## 12. Recursive Queries

```sql
-- Classic example: organizational hierarchy traversal
WITH RECURSIVE org_chart AS (
    -- Anchor member: top-level employees (no manager)
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive member: join back to org_chart
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    INNER JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY level, name;
```

Used for hierarchical/tree-structured data: org charts, category trees, bill-of-materials, graph traversal. The query has an **anchor** (base case) and a **recursive** term that references the CTE itself, repeated until no new rows are produced.

---

## 13. Indexes

```sql
CREATE INDEX idx_lastname ON employees(last_name);
CREATE UNIQUE INDEX idx_email ON employees(email);
CREATE INDEX idx_dept_salary ON employees(department_id, salary);

-- Partial index — only indexes rows matching a condition (smaller, faster)
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Expression index — index on a computed expression, not a raw column
CREATE INDEX idx_lower_email ON users(LOWER(email));

-- Covering index (INCLUDE) — adds extra columns to avoid heap lookups
CREATE INDEX idx_covering ON employees(department_id) INCLUDE (salary, name);

DROP INDEX idx_lastname;
\d employees    -- shows indexes on a table (psql)
```

**Index types in PostgreSQL:**
| Type | Best for |
|---|---|
| **B-Tree** (default) | Equality and range queries (`=`, `<`, `>`, `BETWEEN`, sorting) — the general-purpose default |
| **Hash** | Equality only (`=`); rarely needed since B-Tree handles equality well too |
| **GIN** (Generalized Inverted Index) | Composite values with many keys: `JSONB`, arrays, full-text search (`tsvector`) |
| **GiST** (Generalized Search Tree) | Geometric data, range types, full-text search, nearest-neighbor search |
| **BRIN** (Block Range Index) | Very large tables with naturally ordered data (e.g., timestamps in an append-only log) — tiny index size |
| **SP-GiST** | Non-balanced data structures (e.g., quad-trees, certain text patterns) |

**Important interview point:** Unlike MySQL/InnoDB, PostgreSQL tables are **heap-organized by default** — there is no automatic clustered index on the primary key. All indexes (including the primary key's) are secondary indexes pointing to a row's physical location (a "TID" — tuple ID) in the heap. You *can* manually run `CLUSTER table USING index` to physically reorder the heap to match an index, but this is a one-time operation, not maintained automatically afterward.

---

## 14. Keys

Same concepts as standard relational theory (Primary, Foreign, Unique, Composite, Candidate, Surrogate keys) — see the constraints section for syntax. PostgreSQL-specific notes:
- `SERIAL`/`BIGSERIAL` auto-creates a backing sequence for surrogate primary keys (modern Postgres also supports SQL-standard `GENERATED ALWAYS AS IDENTITY`, which is now generally preferred).
```sql
CREATE TABLE t (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,   -- modern, SQL-standard approach
    name TEXT
);
```
- `UUID` primary keys are common in PostgreSQL (`gen_random_uuid()` via the `pgcrypto` extension, or `uuid-ossp`), useful for distributed systems where IDs must be generated without coordination.

---

## 15. Normalization

Same normalization theory as any relational database (1NF, 2NF, 3NF, BCNF — see general relational database principles). PostgreSQL's rich native types (arrays, JSONB, ranges) sometimes let you intentionally store semi-structured/denormalized data cleanly within a single column where a strictly normalized design would otherwise require extra join tables — a pragmatic middle ground unique to Postgres's flexibility.

---

## 16. Transactions & ACID

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- or ROLLBACK;

BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SAVEPOINT sp1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
ROLLBACK TO SAVEPOINT sp1;
COMMIT;
```

PostgreSQL is fully ACID-compliant for all standard table types (unlike MySQL, where this depends on storage engine choice — Postgres has only one core storage format, so this isn't a per-table decision).

**Important Postgres-specific quirk:** If any statement within a transaction errors, the **entire transaction is aborted** and every subsequent statement will fail with `"current transaction is aborted, commands ignored until end of transaction block"` until you `ROLLBACK` (or `ROLLBACK TO` a savepoint) — unlike MySQL, which often lets you continue after a single failed statement within a transaction.

---

## 17. MVCC

PostgreSQL implements concurrency using **Multi-Version Concurrency Control (MVCC)**: instead of locking rows for reads, every row version carries hidden transaction-ID metadata (`xmin`/`xmax`). When a row is updated, Postgres doesn't overwrite it in place — it writes a **new row version** and marks the old version as expired (rather than immediately deleting it). Each transaction sees a consistent snapshot of the data as of when it started, determined by which row versions are visible to its transaction ID.

**Benefits:**
- **Readers never block writers, and writers never block readers** (a major practical difference from many lock-based systems)
- Each transaction gets a consistent, repeatable view of the data without needing to hold long read locks

**Cost:** Old row versions ("dead tuples") accumulate and must be cleaned up — this is exactly what **VACUUM** is for (next section). Excessive dead tuples bloat tables/indexes and slow down queries if vacuuming falls behind.

---

## 18. VACUUM & Autovacuum

Because MVCC leaves behind dead row versions after updates/deletes, PostgreSQL needs a maintenance process to reclaim that space and keep the query planner's statistics fresh.

```sql
VACUUM employees;              -- reclaims dead tuple space for reuse (doesn't shrink file size on disk)
VACUUM FULL employees;         -- reclaims space AND shrinks the file on disk, but takes an exclusive lock (use cautiously, causes downtime on large tables)
VACUUM ANALYZE employees;      -- vacuum + update planner statistics
ANALYZE employees;             -- update planner statistics only

SELECT * FROM pg_stat_user_tables WHERE relname = 'employees';   -- check dead tuple counts, last vacuum time
```

**Autovacuum** is a background process that automatically runs `VACUUM`/`ANALYZE` on tables once they cross configured dead-tuple thresholds — enabled by default and should almost always be left on in production. Tuning autovacuum (more aggressive thresholds on high-churn tables) is a common real-world DBA task.

**Why this matters (very common interview/real-world topic):** A table with autovacuum falling behind (e.g., due to a long-running transaction holding back cleanup, or a very high write rate) will accumulate "bloat" — dead tuples that waste space and slow scans — and in extreme cases risk **transaction ID wraparound**, a serious failure mode Postgres actively guards against by forcing aggressive vacuuming as IDs approach exhaustion.

---

## 19. Isolation Levels

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SHOW transaction_isolation;
```

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Serialization Anomaly |
|---|---|---|---|---|
| **READ UNCOMMITTED** | — (Postgres treats this the same as READ COMMITTED; true dirty reads are never possible due to MVCC) | Possible | Possible | Possible |
| **READ COMMITTED** (default) | Prevented | Possible | Possible | Possible |
| **REPEATABLE READ** | Prevented | Prevented | Prevented | Possible |
| **SERIALIZABLE** | Prevented | Prevented | Prevented | Prevented |

**Important Postgres-specific note:** Because of MVCC, PostgreSQL **never actually performs a true dirty read** even at the `READ UNCOMMITTED` level — it's silently treated identically to `READ COMMITTED`. PostgreSQL's `SERIALIZABLE` is implemented via **Serializable Snapshot Isolation (SSI)**, which can abort transactions with a serialization failure error that the application must be prepared to retry.

---

## 20. Locking

```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;       -- row-level exclusive lock
SELECT * FROM accounts WHERE id = 1 FOR SHARE;          -- row-level shared lock
SELECT * FROM accounts WHERE id = 1 FOR UPDATE SKIP LOCKED;   -- skip already-locked rows (great for job queues!)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;          -- fail immediately instead of waiting

LOCK TABLE accounts IN EXCLUSIVE MODE;     -- explicit table-level lock (rarely needed)

SELECT * FROM pg_locks;       -- inspect current locks
SELECT * FROM pg_stat_activity;   -- inspect active queries/connections, find blocking sessions
```

**`FOR UPDATE SKIP LOCKED` (Postgres pattern worth knowing for interviews):** A common technique for implementing a reliable job/task queue — multiple workers can each grab a different unlocked row without blocking each other or accidentally processing the same row twice.

---

## 21. Views & Materialized Views

```sql
CREATE VIEW high_earners AS
SELECT first_name, last_name, salary FROM employees WHERE salary > 80000;

SELECT * FROM high_earners;

-- Materialized view — actually stores the result physically, unlike a regular view
CREATE MATERIALIZED VIEW dept_summary AS
SELECT department_id, COUNT(*) AS emp_count, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id;

REFRESH MATERIALIZED VIEW dept_summary;                    -- recompute and replace stored data (locks reads during refresh by default)
REFRESH MATERIALIZED VIEW CONCURRENTLY dept_summary;        -- refresh without blocking reads (requires a unique index on the view)
```

**View vs Materialized View (interview point):** A regular `VIEW` is just a saved query — re-executed every time it's referenced, always reflecting current data, no extra storage. A `MATERIALIZED VIEW` physically stores its result set on disk like a table, making reads much faster for expensive aggregations, but the data goes stale until explicitly `REFRESH`ed — a trade-off between query speed and data freshness, common for dashboards/reports.

---

## 22. Functions, Procedures & Triggers

```sql
-- Function (PL/pgSQL)
CREATE OR REPLACE FUNCTION get_full_name(fname TEXT, lname TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN fname || ' ' || lname;
END;
$$ LANGUAGE plpgsql;

SELECT get_full_name(first_name, last_name) FROM employees;

-- Procedure (Postgres 11+) — can manage its own transactions (COMMIT/ROLLBACK inside it), unlike functions
CREATE OR REPLACE PROCEDURE give_raise(emp_id INT, amount NUMERIC)
LANGUAGE plpgsql AS $$
BEGIN
    UPDATE employees SET salary = salary + amount WHERE id = emp_id;
    COMMIT;
END;
$$;

CALL give_raise(5, 1000);

-- Trigger
CREATE OR REPLACE FUNCTION audit_salary_change()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO employee_audit(employee_id, old_salary, new_salary, changed_at)
    VALUES (OLD.id, OLD.salary, NEW.salary, NOW());
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER before_salary_update
BEFORE UPDATE ON employees
FOR EACH ROW
WHEN (OLD.salary IS DISTINCT FROM NEW.salary)
EXECUTE FUNCTION audit_salary_change();
```

**Function vs Procedure in Postgres (interview point, version-specific):** Before PostgreSQL 11, only functions existed and couldn't manage transactions internally. Since 11, `PROCEDURE`s were added — they're invoked with `CALL` (not embeddable in a `SELECT`), and unlike functions, they **can** issue `COMMIT`/`ROLLBACK` inside their own body, useful for long-running batch/admin tasks.

**Procedural languages:** PL/pgSQL is the default and most common, but Postgres also supports PL/Python, PL/Perl, PL/Tcl, and others via extensions.

---

## 23. JSON & JSONB

PostgreSQL's JSON support is one of its standout features versus MySQL.

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    attributes JSONB
);

INSERT INTO products (name, attributes) VALUES
('Laptop', '{"brand": "Dell", "specs": {"ram": "16GB", "cpu": "i7"}, "tags": ["electronics", "computers"]}');

-- Query JSON fields
SELECT name, attributes->>'brand' AS brand FROM products;                -- ->> returns text
SELECT name, attributes->'specs'->>'ram' AS ram FROM products;            -- nested access
SELECT * FROM products WHERE attributes->>'brand' = 'Dell';
SELECT * FROM products WHERE attributes @> '{"brand": "Dell"}';            -- containment operator
SELECT * FROM products WHERE attributes ? 'tags';                          -- key existence check
SELECT * FROM products WHERE attributes->'tags' ? 'electronics';            -- array element exists

-- Index JSONB for fast lookups
CREATE INDEX idx_attributes_gin ON products USING GIN (attributes);

-- Update a nested JSON field
UPDATE products SET attributes = jsonb_set(attributes, '{specs,ram}', '"32GB"') WHERE id = 1;
```

**`JSON` vs `JSONB` (very common interview question):** `JSON` stores an exact text copy of the input (preserves whitespace/key order, re-parsed on every access — slower). `JSONB` stores a decomposed **binary** format (no whitespace/duplicate-key/order preservation, but faster to process and — critically — **indexable** via GIN indexes). **`JSONB` is recommended for almost all use cases** unless you specifically need to preserve the exact original text representation.

---

## 24. Arrays & Other Rich Types

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title TEXT,
    tags TEXT[]
);

INSERT INTO posts (title, tags) VALUES ('My Post', ARRAY['postgres', 'database', 'sql']);

SELECT * FROM posts WHERE 'postgres' = ANY(tags);
SELECT * FROM posts WHERE tags @> ARRAY['postgres'];     -- contains
SELECT unnest(tags) FROM posts;                            -- expand array into rows
CREATE INDEX idx_tags ON posts USING GIN (tags);

-- Range types
CREATE TABLE reservations (
    room_id INT,
    during TSRANGE
);
SELECT * FROM reservations WHERE during && '[2024-01-01, 2024-01-05)'::tsrange;   -- overlap operator
```

Native array and range types let you model certain data (tags, multi-valued attributes, time ranges/booking conflicts) cleanly within a single column, something MySQL lacks built-in equivalents for.

---

## 25. Window Functions

PostgreSQL has full, mature window function support (this was a Postgres strength well before MySQL added it in 8.0).

```sql
SELECT name, department_id, salary,
       ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rank_in_dept,
       RANK() OVER (ORDER BY salary DESC) AS overall_rank,
       SUM(salary) OVER (PARTITION BY department_id) AS dept_total,
       AVG(salary) OVER () AS company_avg
FROM employees;

-- Running total with frame clause
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders;
```

(See the MySQL notes' Window Functions section for `RANK` vs `DENSE_RANK` vs `ROW_NUMBER` and `LAG`/`LEAD` explanations — identical syntax/behavior in PostgreSQL.)

---

## 26. Query Optimization & EXPLAIN

```sql
EXPLAIN SELECT * FROM employees WHERE department_id = 3;
EXPLAIN ANALYZE SELECT * FROM employees WHERE department_id = 3;   -- actually runs the query, shows real timings
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;                  -- detailed, machine-readable output
```

**Key things to look for in `EXPLAIN ANALYZE` output:**
| Concept | What to check |
|---|---|
| **Seq Scan vs Index Scan** | `Seq Scan` (full table scan) on a large table is often a red flag — check if a useful index is missing |
| **Estimated vs Actual rows** | A large mismatch between planner estimates and actual rows suggests stale statistics — run `ANALYZE` |
| **Nested Loop / Hash Join / Merge Join** | Different join strategies the planner chose — usually fine to trust the planner, but worth understanding for tuning |
| **Cost** | `cost=startup..total` — relative planner cost estimate, not actual time |
| **Actual Time** | Real measured execution time per node (only with `ANALYZE`) |

**Common optimization techniques:**
- Index columns used in `WHERE`/`JOIN`/`ORDER BY` — choose the right index type (GIN for JSONB/arrays, B-Tree for most else)
- Run `ANALYZE` (or rely on autovacuum) so the planner has fresh statistics
- Use partial indexes for queries that always filter on a specific condition
- Watch for `Seq Scan` on large tables in `EXPLAIN` output
- Avoid `SELECT *` when only specific columns are needed
- Use connection pooling (PgBouncer) since each connection is a full OS process — high connection counts are expensive
- Consider `VACUUM`/bloat status on frequently updated tables affecting performance

---

## 27. Partitioning

```sql
CREATE TABLE sales (
    id SERIAL,
    sale_date DATE NOT NULL,
    amount NUMERIC(10,2)
) PARTITION BY RANGE (sale_date);

CREATE TABLE sales_2023 PARTITION OF sales
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE sales_2024 PARTITION OF sales
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- List partitioning
CREATE TABLE orders (
    id SERIAL,
    region TEXT
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('US');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('EU');

-- Hash partitioning
CREATE TABLE events (
    id SERIAL,
    user_id INT
) PARTITION BY HASH (user_id);

CREATE TABLE events_p0 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
```

Modern PostgreSQL (10+) has **declarative partitioning** built in (`PARTITION BY RANGE/LIST/HASH`) — queries against the parent table automatically route to the right partition(s) ("partition pruning"), and maintenance (e.g., dropping an old partition) is far cheaper than deleting matching rows from a single giant table.

---

## 28. Replication

```
Primary (read/write)
      │
      │ WAL streaming
      ▼
┌─────────────┐    ┌─────────────┐
│ Streaming      │    │ Streaming      │
│ Replica 1       │    │ Replica 2       │   (read-only / hot standby)
└─────────────┘    └─────────────┘
```

**Streaming replication (physical):** Replicas continuously receive and replay the primary's WAL records, producing an exact byte-for-byte copy of the database. Used for high availability, failover, and read scaling (replicas can serve read-only queries — "hot standby").

```sql
-- Check replication status on primary
SELECT * FROM pg_stat_replication;
-- Check status on a replica
SELECT pg_is_in_recovery();
```

**Logical replication:** Replicates data changes at the logical (row/table) level rather than byte-for-byte WAL streaming — allows replicating specific tables, between different PostgreSQL major versions, or even into different schemas, and supports more flexible multi-directional setups. Configured via `PUBLICATION` (on the source) and `SUBSCRIPTION` (on the target).

```sql
-- On source
CREATE PUBLICATION my_pub FOR TABLE employees;
-- On target
CREATE SUBSCRIPTION my_sub CONNECTION 'host=source_host dbname=mydb' PUBLICATION my_pub;
```

**Streaming (physical) vs Logical replication (interview point):** Streaming replication copies the entire database byte-for-byte via WAL — simple, robust, used for full HA/failover, but the replica must match the primary's major version and replicates everything. Logical replication works at the row/table level — more flexible (selective tables, cross-version, multi-master-ish patterns) but with more setup overhead and some feature limitations (e.g., DDL isn't automatically replicated).

---

## 29. Backup & Recovery

```bash
# Logical backup
pg_dump -U postgres -d mydb -F c -f backup.dump      # custom format (compressed, supports parallel restore)
pg_dump -U postgres -d mydb > backup.sql               # plain SQL format
pg_dumpall -U postgres > full_cluster_backup.sql        # all databases + roles

# Restore
pg_restore -U postgres -d mydb backup.dump
psql -U postgres -d mydb < backup.sql

# Physical backup (base backup + WAL archiving) — for large databases / PITR
pg_basebackup -U replicator -D /backup/base -Fp -Xs -P
```

**Point-in-time recovery (PITR):** Combine a periodic physical base backup with continuous **WAL archiving** — you can then restore the base backup and replay WAL records up to any specific point in time (e.g., "1 minute before someone ran a bad `DELETE`"), a powerful disaster-recovery capability.

---

## 30. Extensions

One of PostgreSQL's biggest strengths — functionality can be added via extensions without modifying core code.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;     -- cryptographic functions, gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS postgis;       -- full geographic/GIS support
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;   -- query performance tracking across all queries
CREATE EXTENSION IF NOT EXISTS uuid-ossp;      -- UUID generation functions
CREATE EXTENSION IF NOT EXISTS hstore;          -- key-value store within a column
CREATE EXTENSION IF NOT EXISTS pg_trgm;          -- trigram-based fuzzy text search/similarity

\dx     -- list installed extensions (psql)
```

**`pg_stat_statements` (worth knowing for interviews):** Tracks execution statistics for every distinct query run on the server (calls, total time, rows) — the standard tool for finding your slowest/most expensive queries in production.

---

## 31. Security & Roles

PostgreSQL unifies "users" and "groups" into a single concept: **roles**.

```sql
CREATE ROLE app_user WITH LOGIN PASSWORD 'StrongPassword123!';
CREATE ROLE readonly_group;

GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA hr TO app_user;
GRANT SELECT, INSERT, UPDATE ON hr.employees TO app_user;
GRANT SELECT ON ALL TABLES IN SCHEMA hr TO readonly_group;
GRANT readonly_group TO app_user;     -- role membership/inheritance

REVOKE INSERT ON hr.employees FROM app_user;

ALTER ROLE app_user WITH PASSWORD 'NewPassword!';
DROP ROLE app_user;

\du     -- list roles (psql)
```

**Row-Level Security (RLS) — Postgres-specific feature:**
```sql
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

CREATE POLICY employee_self_access ON employees
    USING (id = current_setting('app.current_employee_id')::INT);
```
RLS lets you define **per-row** access policies enforced directly by the database — e.g., a user can only see their own rows, regardless of what query they run — a powerful multi-tenant security feature MySQL doesn't natively have.

**Other security practices:** require SSL/TLS connections (`sslmode=require`), use `pg_hba.conf` to control which hosts/methods can authenticate, parameterized queries to prevent SQL injection (same principle as any RDBMS), encrypt sensitive columns with `pgcrypto` where appropriate.

---

## 32. Troubleshooting & Performance

```sql
SELECT * FROM pg_stat_activity;                      -- active connections/queries
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE pid = 1234;   -- kill a query/connection
SELECT * FROM pg_locks WHERE NOT granted;              -- find blocked lock requests
SELECT * FROM pg_stat_user_tables;                       -- table-level stats (seq scans, dead tuples, etc.)
SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;   -- top slow queries (needs extension)

-- Find blocking queries
SELECT blocked_locks.pid AS blocked_pid, blocking_locks.pid AS blocking_pid
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
 AND blocking_locks.pid != blocked_locks.pid
WHERE NOT blocked_locks.granted;
```

**Common performance issues:**
| Symptom | Likely cause |
|---|---|
| Slow queries despite indexes | Stale planner statistics — run `ANALYZE`; or wrong index type for the data (e.g., B-Tree on JSONB instead of GIN) |
| Table bloat / growing disk usage | Autovacuum falling behind — check `pg_stat_user_tables`, tune autovacuum settings |
| Connection exhaustion | No connection pooling — each connection is a full process; use PgBouncer/PgPool |
| Lock waits | Long-running transactions holding row/table locks — check `pg_locks`/`pg_stat_activity` |
| Sudden planner regressions | Outdated statistics after large data changes — run `ANALYZE` |

---

## 33. Best Practices Summary

- Prefer `TIMESTAMPTZ` over `TIMESTAMP` for time data
- Prefer `JSONB` over `JSON` unless you specifically need exact text preservation
- Use `TEXT` instead of arbitrary `VARCHAR(n)` limits unless a real constraint exists
- Always have a primary key (use `GENERATED ALWAYS AS IDENTITY` or `UUID` as appropriate)
- Use schemas to logically organize large databases
- Don't disable autovacuum; tune it for high-churn tables instead
- Use connection pooling (PgBouncer) — connections are expensive (process-per-connection model)
- Use `EXPLAIN ANALYZE` before deploying complex/slow queries
- Choose the right index type for the data (GIN for JSONB/arrays/full-text, B-Tree for general use, BRIN for huge naturally-ordered tables)
- Use partial/expression indexes to keep indexes small and targeted
- Use Row-Level Security for multi-tenant data isolation where appropriate
- Regularly back up (logical + physical/WAL for PITR) and test restores
- Monitor with `pg_stat_statements` to find expensive queries proactively

---

## 34. Cheat Sheet

```sql
-- psql meta-commands
\l \c db \dt \d table \du \dx \q

-- Structure
CREATE SCHEMA / CREATE TABLE / ALTER TABLE / DROP TABLE / TRUNCATE TABLE

-- CRUD with RETURNING
INSERT INTO t (...) VALUES (...) RETURNING id;
UPDATE t SET col = val WHERE cond RETURNING *;
DELETE FROM t WHERE cond RETURNING *;
INSERT INTO t (...) VALUES (...) ON CONFLICT (col) DO UPDATE SET ...;

-- Joins
SELECT * FROM a INNER/LEFT/RIGHT/FULL OUTER JOIN b ON a.id = b.a_id;

-- CTEs
WITH x AS (SELECT ...) SELECT * FROM x;
WITH RECURSIVE x AS (...) SELECT * FROM x;

-- Indexes
CREATE INDEX idx ON t(col);
CREATE INDEX idx ON t USING GIN (jsonb_col);
EXPLAIN ANALYZE SELECT ...;

-- Transactions
BEGIN; ... COMMIT; / ROLLBACK;

-- JSONB
col->>'key'   -- text
col->'key'    -- json
col @> '{...}'  -- contains
col ? 'key'      -- key exists

-- Roles
CREATE ROLE r WITH LOGIN PASSWORD 'pwd';
GRANT privs ON obj TO r;

-- Backup
pg_dump -U user -d db -F c -f backup.dump
pg_restore -U user -d db backup.dump
```

---

## 35. Interview Questions & Answers

**Q1: What is PostgreSQL and how does it differ from MySQL?**
A: PostgreSQL is an open-source object-relational database known for standards compliance and extensibility — native JSONB with indexing, arrays, range types, full `FULL OUTER JOIN` support, writable CTEs, and a rich extension ecosystem (e.g., PostGIS). MySQL has historically focused more on simplicity and raw speed for straightforward read-heavy workloads. The performance gap has narrowed; the choice today often comes down to feature needs (advanced SQL, JSON, geospatial) versus simplicity/ecosystem familiarity.

**Q2: Explain MVCC and why PostgreSQL uses it.**
A: Multi-Version Concurrency Control means every row update creates a new row version rather than overwriting in place, with old versions marked expired (not deleted immediately). Each transaction sees a consistent snapshot based on which row versions are visible to it. This means **readers never block writers and writers never block readers**, unlike pure lock-based concurrency models — at the cost of needing periodic cleanup (`VACUUM`) of old dead row versions.

**Q3: What is VACUUM and why is it necessary?**
A: Because MVCC leaves "dead tuples" (old row versions) behind after updates/deletes, `VACUUM` reclaims that space for reuse and updates planner statistics. PostgreSQL runs this automatically via **autovacuum** in the background. If vacuuming falls behind, tables bloat (wasting space, slowing scans), and in extreme cases the database risks transaction ID wraparound — a critical failure mode Postgres actively guards against.

**Q4: What's the difference between `JSON` and `JSONB`?**
A: `JSON` stores the input as an exact text copy (preserves formatting/key order, but is re-parsed on every access). `JSONB` stores a decomposed binary representation — faster to query, supports GIN indexing for fast containment/key-existence lookups, but doesn't preserve exact original formatting or duplicate keys. `JSONB` is recommended for almost all practical use cases.

**Q5: Does PostgreSQL have a clustered index like MySQL's InnoDB?**
A: No — by default, PostgreSQL tables are heap-organized with no automatic clustered index; every index (including the primary key's) is a secondary structure pointing to a row's physical location via a tuple ID. You can manually run `CLUSTER table USING index` to physically reorder the table to match an index once, but it isn't automatically maintained afterward like InnoDB's clustered primary key.

**Q6: What is a CTE, and what's a "writable CTE"?**
A: A Common Table Expression (`WITH ... AS (...)`) defines a temporary named result set usable within a larger query — improves readability over nested subqueries and enables recursive queries. A "writable CTE" is a PostgreSQL-specific capability allowing `INSERT`/`UPDATE`/`DELETE` statements (with `RETURNING`) inside a CTE, enabling patterns like moving deleted rows directly into an archive table in one statement.

**Q7: How would you write a recursive query to traverse a hierarchy (e.g., an org chart)?**
A: Use `WITH RECURSIVE`, with an anchor member selecting the base case (e.g., top-level employees with no manager) `UNION ALL`'d with a recursive member that joins the employees table back to the CTE itself on the manager relationship — repeated until no new matching rows are produced.

**Q8: What index types does PostgreSQL support, and when would you use GIN vs B-Tree?**
A: PostgreSQL supports B-Tree (default — equality/range/sorting), Hash (equality only), GIN (composite values like JSONB, arrays, full-text search — fast lookups within complex values), GiST (geometric/range data, nearest-neighbor search), BRIN (huge naturally-ordered tables, very small index size), and SP-GiST (non-balanced structures). Use B-Tree for typical scalar column filtering/sorting; use GIN when indexing JSONB fields, arrays, or full-text search vectors where you need to search "within" a composite value.

**Q9: What's the difference between a view and a materialized view?**
A: A `VIEW` is just a stored query, re-executed every time it's referenced — always current, no extra storage. A `MATERIALIZED VIEW` physically stores the computed result set like a table — much faster to read for expensive aggregations, but the data becomes stale until explicitly `REFRESH`ed, trading freshness for query speed.

**Q10: Explain PostgreSQL's process-per-connection model and why connection pooling matters.**
A: Unlike MySQL's lightweight thread-per-connection model, PostgreSQL forks a full OS process per client connection — more memory/overhead per connection, and a practical ceiling on how many concurrent connections a server can comfortably handle. This is why connection pooling tools like **PgBouncer** are standard in production PostgreSQL deployments — they multiplex many client connections onto a smaller pool of actual database connections.

**Q11: What's the difference between streaming replication and logical replication?**
A: Streaming (physical) replication continuously ships and replays the primary's WAL to replicas, producing an exact byte-for-byte copy — simple, robust, used for full HA/failover/read scaling, but requires matching major versions and replicates the entire cluster. Logical replication works at the row/table level via publications/subscriptions — more flexible (specific tables, cross-version, multi-directional setups) but doesn't automatically replicate DDL and has more operational complexity.

**Q12: What is `FOR UPDATE SKIP LOCKED` used for?**
A: A locking read that skips any rows already locked by another transaction rather than waiting — a common pattern for implementing reliable concurrent job/task queues, where multiple worker processes can each grab a different available row without blocking each other or double-processing the same row.

**Q13: How does PostgreSQL's `SERIALIZABLE` isolation level work?**
A: It's implemented via **Serializable Snapshot Isolation (SSI)** — transactions run concurrently under snapshot isolation, but PostgreSQL detects when their interleaving would have produced a result impossible under any serial (one-at-a-time) execution order, and aborts one of the conflicting transactions with a serialization failure error, which the application must catch and retry.

**Q14: What is Row-Level Security (RLS) and when would you use it?**
A: A PostgreSQL feature letting you define per-row access policies enforced directly by the database engine (`CREATE POLICY ... USING (condition)`), so different users/roles only see rows matching the policy regardless of the query they run. Commonly used for multi-tenant applications to guarantee tenant data isolation at the database layer itself, rather than relying solely on application-level filtering.

**Q15: What's the difference between `RETURNING` and a separate `SELECT` after an `INSERT`?**
A: `RETURNING` lets an `INSERT`/`UPDATE`/`DELETE` statement directly return the affected rows' values (e.g., a generated ID) in the same round-trip — avoiding a second network round-trip and avoiding any race condition where another transaction could modify the row between the write and a separate follow-up `SELECT`.

**Q16: How would you implement table partitioning in PostgreSQL, and why?**
A: Use declarative partitioning (`PARTITION BY RANGE/LIST/HASH` on the parent table, with child tables created `PARTITION OF` it). Queries automatically prune to only the relevant partition(s), and maintenance operations (like dropping old data) become a cheap `DROP TABLE` on a partition instead of an expensive `DELETE` — used for very large tables, especially time-series data partitioned by date.

**Q17: What is `pg_stat_statements` and why is it useful?**
A: An extension that tracks execution statistics (call count, total/average execution time, rows returned) for every distinct query pattern run on the server — the standard tool for identifying the most expensive or frequently-run queries in a production PostgreSQL database, guiding where to focus optimization effort.

**Q18: How does PostgreSQL handle a failed statement inside a transaction differently from MySQL?**
A: In PostgreSQL, if any statement within a transaction errors, the **entire transaction is aborted** — every subsequent statement fails immediately until you issue a `ROLLBACK` (or roll back to a `SAVEPOINT`). This is stricter than MySQL's typical default behavior, where a single failed statement doesn't necessarily abort the whole transaction — an important gotcha for engineers moving between the two systems.

**Q19: What are `EXCLUDE` constraints used for?**
A: A PostgreSQL-specific constraint type that generalizes uniqueness to arbitrary operators — most commonly used with GiST indexes to prevent overlapping ranges, such as ensuring no two bookings for the same room have overlapping time ranges, enforced directly at the database level without needing application-side conflict checks.

**Q20: How would you debug a slow query in PostgreSQL?**
A: Run `EXPLAIN ANALYZE` to see both the planner's estimated cost and actual execution time/row counts per step; look for `Seq Scan` on large tables (possible missing index), large mismatches between estimated and actual rows (stale statistics — run `ANALYZE`), and check `pg_stat_statements` to confirm whether this query is a frequent/expensive offender overall versus a one-off. Also check for table bloat (`pg_stat_user_tables`) and lock contention (`pg_locks`, `pg_stat_activity`) as other common root causes.

---

### Final interview tip
Be ready to explain **MVCC and why it means readers don't block writers**, the difference between **JSON vs JSONB**, why **PostgreSQL has no automatic clustered index** (unlike MySQL/InnoDB), and to write a **recursive CTE** for a hierarchy example from memory. Also expect at least one question contrasting PostgreSQL with MySQL directly — interviewers often want to see you understand the trade-offs, not just memorized syntax.
