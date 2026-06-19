# MySQL — Complete Notes (Basics → Advanced + Interview Prep)

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [MySQL vs Other Databases](#2-mysql-vs-other-databases)
3. [MySQL Architecture](#3-mysql-architecture)
4. [Installation & Setup](#4-installation--setup)
5. [Databases & Tables Basics](#5-databases--tables-basics)
6. [Data Types](#6-data-types)
7. [Constraints](#7-constraints)
8. [CRUD — SQL Basics](#8-crud--sql-basics)
9. [Filtering, Sorting & Aggregation](#9-filtering-sorting--aggregation)
10. [Joins](#10-joins)
11. [Subqueries](#11-subqueries)
12. [Indexes](#12-indexes)
13. [Keys (Primary, Foreign, Unique, Composite)](#13-keys)
14. [Normalization](#14-normalization)
15. [Transactions & ACID](#15-transactions--acid)
16. [Storage Engines (InnoDB vs MyISAM)](#16-storage-engines)
17. [Locking & Concurrency](#17-locking--concurrency)
18. [Isolation Levels](#18-isolation-levels)
19. [Views](#19-views)
20. [Stored Procedures, Functions & Triggers](#20-stored-procedures-functions--triggers)
21. [Query Optimization & EXPLAIN](#21-query-optimization--explain)
22. [Replication](#22-replication)
23. [Backup & Recovery](#23-backup--recovery)
24. [Partitioning](#24-partitioning)
25. [Security](#25-security)
26. [User Management & Privileges](#26-user-management--privileges)
27. [Common Functions](#27-common-functions)
28. [Window Functions](#28-window-functions)
29. [Troubleshooting & Performance](#29-troubleshooting--performance)
30. [Best Practices Summary](#30-best-practices-summary)
31. [Cheat Sheet](#31-cheat-sheet)
32. [Interview Questions & Answers](#32-interview-questions--answers)

---

## 1. Introduction

**MySQL** is one of the world's most popular open-source **Relational Database Management Systems (RDBMS)**, using **SQL (Structured Query Language)** to define, query, and manage structured data organized into tables with rows and columns.

**Key characteristics:**
- Relational — data organized in tables with defined relationships (via foreign keys)
- ACID-compliant (with InnoDB, the default storage engine)
- Supports transactions, indexes, views, stored procedures, triggers, replication
- Client-server architecture — a server process (`mysqld`) handles connections from clients
- Widely used in web applications (the "M" in the classic LAMP stack)

---

## 2. MySQL vs Other Databases

| Aspect | MySQL | PostgreSQL | MongoDB (NoSQL) |
|---|---|---|---|
| Type | Relational (RDBMS) | Relational (RDBMS, more feature-rich) | Document-based (NoSQL) |
| Schema | Fixed schema | Fixed schema, more advanced types | Schema-less/flexible |
| ACID | Yes (InnoDB) | Yes | Limited/eventual (varies by config) |
| Joins | Yes | Yes (more advanced) | No native joins (`$lookup` aggregation only) |
| Best for | Web apps, read-heavy workloads | Complex queries, data integrity, analytics | Unstructured/rapidly evolving data, horizontal scale |
| Extensibility | Moderate | Highly extensible (custom types, extensions) | N/A — different paradigm |

**Key interview point:** SQL/relational databases enforce a fixed schema and strong consistency, ideal when data integrity and relationships matter (financial systems, transactional apps). NoSQL databases trade strict schema/relations for flexibility and horizontal scalability, suited to rapidly changing or massive-scale unstructured data.

---

## 3. MySQL Architecture

```
                Client (app, CLI, MySQL Workbench)
                            │
                            ▼
                ┌────────────────────────┐
                │     Connection Layer      │   (auth, thread handling)
                └───────────┬────────────┘
                            ▼
                ┌────────────────────────┐
                │     SQL Layer             │
                │  - Parser                  │
                │  - Optimizer                 │
                │  - Query Cache (removed in 8.0)│
                └───────────┬────────────┘
                            ▼
                ┌────────────────────────┐
                │   Storage Engine Layer     │   (InnoDB, MyISAM, Memory...)
                └───────────┬────────────┘
                            ▼
                ┌────────────────────────┐
                │      Disk / Files          │
                └────────────────────────┘
```

- **Connection Layer** — handles client authentication and connection pooling/threading.
- **SQL Layer** — parses SQL, the **optimizer** decides the most efficient execution plan (which indexes to use, join order, etc.).
- **Storage Engine Layer** — pluggable layer that actually stores/retrieves data; different tables can even use different engines. **InnoDB** is the default and most widely used (ACID-compliant, row-level locking, foreign keys). **MyISAM** is an older engine (table-level locking, no transactions, no foreign keys — largely legacy now).

---

## 4. Installation & Setup

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install mysql-server
sudo mysql_secure_installation     # set root password, remove anonymous users, disable remote root login

# Docker
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=secret -p 3306:3306 mysql:8.0

# Connect
mysql -u root -p
mysql -h <host> -u <user> -p<password> <database>
```

```sql
SHOW DATABASES;
USE mydb;
SHOW TABLES;
SELECT VERSION();
```

---

## 5. Databases & Tables Basics

```sql
CREATE DATABASE company;
DROP DATABASE company;
USE company;

CREATE TABLE employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    department_id INT,
    salary DECIMAL(10,2),
    hire_date DATE DEFAULT (CURRENT_DATE),
    FOREIGN KEY (department_id) REFERENCES departments(id)
);

DESCRIBE employees;          -- or: SHOW COLUMNS FROM employees;
ALTER TABLE employees ADD COLUMN phone VARCHAR(20);
ALTER TABLE employees MODIFY COLUMN salary DECIMAL(12,2);
ALTER TABLE employees DROP COLUMN phone;
RENAME TABLE employees TO staff;
DROP TABLE staff;
TRUNCATE TABLE employees;     -- removes all rows, resets AUTO_INCREMENT, faster than DELETE, can't be rolled back in some engines
```

---

## 6. Data Types

| Category | Types | Notes |
|---|---|---|
| **Numeric** | `INT`, `TINYINT`, `SMALLINT`, `BIGINT`, `DECIMAL(p,s)`, `FLOAT`, `DOUBLE` | `DECIMAL` for exact precision (money); `FLOAT`/`DOUBLE` for approximate values |
| **String** | `CHAR(n)`, `VARCHAR(n)`, `TEXT`, `BLOB`, `ENUM`, `SET` | `CHAR` fixed-length (padded), `VARCHAR` variable-length; `TEXT`/`BLOB` for large data |
| **Date/Time** | `DATE`, `DATETIME`, `TIMESTAMP`, `TIME`, `YEAR` | `TIMESTAMP` is timezone-aware & auto-converts, `DATETIME` is not; `TIMESTAMP` range is smaller (1970–2038) |
| **Boolean** | `BOOLEAN` (alias for `TINYINT(1)`) | MySQL has no true native boolean type |
| **JSON** | `JSON` | Native JSON type (MySQL 5.7+), supports JSON functions/indexing |
| **Spatial** | `GEOMETRY`, `POINT`, `POLYGON` | For geographic/spatial data |

**`CHAR` vs `VARCHAR` (interview point):** `CHAR(n)` always uses `n` bytes (padded with spaces), faster for fixed-length data (e.g., country codes). `VARCHAR(n)` uses only as much space as needed plus 1-2 length bytes, better for variable-length text.

---

## 7. Constraints

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    total DECIMAL(10,2) CHECK (total >= 0),
    status VARCHAR(20) DEFAULT 'pending',
    email VARCHAR(100) UNIQUE,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

| Constraint | Purpose |
|---|---|
| `PRIMARY KEY` | Uniquely identifies each row; implies `NOT NULL` + `UNIQUE` |
| `FOREIGN KEY` | Enforces referential integrity to another table |
| `UNIQUE` | No duplicate values allowed in the column |
| `NOT NULL` | Column cannot be NULL |
| `CHECK` | Validates a condition on the column value (MySQL 8.0.16+) |
| `DEFAULT` | Default value if none provided |

**`ON DELETE`/`ON UPDATE` actions:** `CASCADE` (propagate change/delete to child rows), `SET NULL`, `RESTRICT`/`NO ACTION` (block the operation if dependent rows exist).

---

## 8. CRUD — SQL Basics

```sql
-- CREATE
INSERT INTO employees (first_name, last_name, email, salary)
VALUES ('Jane', 'Doe', 'jane@example.com', 75000);

INSERT INTO employees (first_name, last_name) VALUES
('Tom', 'Lee'), ('Amy', 'Singh');     -- multi-row insert

-- READ
SELECT * FROM employees;
SELECT first_name, salary FROM employees WHERE department_id = 3;

-- UPDATE
UPDATE employees SET salary = salary * 1.1 WHERE department_id = 3;

-- DELETE
DELETE FROM employees WHERE id = 5;
```

**Important:** Always use `WHERE` with `UPDATE`/`DELETE` — omitting it affects **every row** in the table.

---

## 9. Filtering, Sorting & Aggregation

```sql
SELECT * FROM employees WHERE salary > 50000 AND department_id IN (1,2,3);
SELECT * FROM employees WHERE last_name LIKE 'S%';     -- starts with S
SELECT * FROM employees WHERE hire_date BETWEEN '2023-01-01' AND '2023-12-31';
SELECT * FROM employees WHERE email IS NULL;
SELECT * FROM employees ORDER BY salary DESC LIMIT 10;
SELECT * FROM employees ORDER BY salary DESC LIMIT 10 OFFSET 20;   -- pagination

-- Aggregation
SELECT department_id, COUNT(*) AS emp_count, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5
ORDER BY avg_salary DESC;
```

**`WHERE` vs `HAVING` (common interview question):** `WHERE` filters rows **before** grouping/aggregation; `HAVING` filters groups **after** aggregation (e.g., `HAVING COUNT(*) > 5` — can't be expressed in `WHERE` since `COUNT()` doesn't exist per-row yet).

**Common aggregate functions:** `COUNT()`, `SUM()`, `AVG()`, `MIN()`, `MAX()`, `GROUP_CONCAT()`.

---

## 10. Joins

```sql
-- INNER JOIN — only matching rows in both tables
SELECT e.first_name, d.name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;

-- LEFT JOIN — all rows from left table, NULLs for unmatched right
SELECT e.first_name, d.name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;

-- RIGHT JOIN — all rows from right table, NULLs for unmatched left
SELECT e.first_name, d.name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;

-- FULL OUTER JOIN — MySQL has no native FULL JOIN; emulate with UNION
SELECT e.first_name, d.name FROM employees e LEFT JOIN departments d ON e.department_id = d.id
UNION
SELECT e.first_name, d.name FROM employees e RIGHT JOIN departments d ON e.department_id = d.id;

-- SELF JOIN — table joined to itself (e.g., employee-manager relationship)
SELECT e.first_name AS employee, m.first_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- CROSS JOIN — Cartesian product (every row × every row)
SELECT * FROM colors CROSS JOIN sizes;
```

**Visual reference:**
```
INNER JOIN:  A ∩ B (only matches)
LEFT JOIN:   all of A + matches from B (NULL if no match)
RIGHT JOIN:  all of B + matches from A (NULL if no match)
FULL JOIN:   all of A + all of B (NULLs where no match) — emulated via UNION in MySQL
```

---

## 11. Subqueries

```sql
-- Scalar subquery
SELECT name FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);

-- IN subquery
SELECT name FROM employees WHERE department_id IN (SELECT id FROM departments WHERE location = 'NY');

-- Correlated subquery — references outer query, runs once per outer row
SELECT e.name FROM employees e
WHERE salary > (SELECT AVG(salary) FROM employees WHERE department_id = e.department_id);

-- EXISTS — efficient existence check
SELECT name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- Subquery in FROM (derived table)
SELECT dept_avg.department_id, dept_avg.avg_salary
FROM (SELECT department_id, AVG(salary) AS avg_salary FROM employees GROUP BY department_id) AS dept_avg
WHERE dept_avg.avg_salary > 60000;

-- CTE (Common Table Expression) — MySQL 8.0+, more readable alternative to derived tables
WITH dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees GROUP BY department_id
)
SELECT * FROM dept_avg WHERE avg_salary > 60000;
```

**Subquery vs JOIN (interview point):** Joins are usually more efficient for combining data from multiple tables since the optimizer can better plan execution; subqueries (especially correlated ones, which re-run per outer row) can be slower but are sometimes clearer for existence checks or aggregations. `EXISTS` is typically faster than `IN` with large subquery result sets since it can short-circuit on the first match.

---

## 12. Indexes

An **index** is a separate data structure (typically a B-Tree in InnoDB) that speeds up row lookups, at the cost of extra storage and slower writes (every insert/update/delete must also update the index).

```sql
CREATE INDEX idx_lastname ON employees(last_name);
CREATE UNIQUE INDEX idx_email ON employees(email);
CREATE INDEX idx_dept_salary ON employees(department_id, salary);   -- composite index
DROP INDEX idx_lastname ON employees;
SHOW INDEX FROM employees;
```

**Types of indexes:**
| Type | Description |
|---|---|
| **Primary key index** | Automatically created on the primary key; in InnoDB this *is* the table's physical row storage (clustered index) |
| **Unique index** | Like a regular index but enforces uniqueness |
| **Composite (multi-column) index** | Index on multiple columns — order matters (leftmost-prefix rule) |
| **Full-text index** | Optimized for text search (`MATCH() AGAINST()`) |
| **Spatial index** | For geometric data types |

**Clustered vs Non-clustered index (interview point):** In InnoDB, the **primary key is the clustered index** — actual row data is physically stored in primary key order. All other (secondary/non-clustered) indexes store the indexed column(s) plus the primary key value, requiring an extra lookup ("bookmark lookup") back to the clustered index to fetch full row data.

**Leftmost-prefix rule:** A composite index on `(a, b, c)` can be used for queries filtering on `a`, `(a,b)`, or `(a,b,c)`, but **not** for a query filtering on just `b` or `c` alone — the index is only useful starting from its leftmost column(s).

**When NOT to over-index:** Every index adds overhead to writes (`INSERT`/`UPDATE`/`DELETE`) and consumes disk space — index only columns actually used in `WHERE`, `JOIN`, `ORDER BY`, or `GROUP BY` clauses.

---

## 13. Keys

| Key Type | Description |
|---|---|
| **Primary Key** | Uniquely identifies each row; one per table; cannot be NULL |
| **Foreign Key** | References a primary/unique key in another table, enforcing referential integrity |
| **Unique Key** | Ensures column values are unique; unlike primary key, can allow one NULL (MySQL) |
| **Composite Key** | A primary/unique key made of multiple columns together |
| **Candidate Key** | Any column(s) that could qualify as the primary key |
| **Surrogate Key** | An artificial key (e.g., auto-increment `id`) with no business meaning, used instead of a natural key |

---

## 14. Normalization

The process of organizing tables/columns to minimize data redundancy and avoid update/insert/delete anomalies.

| Normal Form | Rule |
|---|---|
| **1NF** | Each column holds atomic (indivisible) values; no repeating groups |
| **2NF** | 1NF + every non-key column fully depends on the **entire** primary key (no partial dependency on part of a composite key) |
| **3NF** | 2NF + no transitive dependencies (non-key columns don't depend on other non-key columns) |
| **BCNF** | Stricter version of 3NF — every determinant is a candidate key |

**Denormalization:** Intentionally introducing some redundancy (e.g., duplicating a column to avoid a join) to improve **read performance** in specific cases — a deliberate trade-off, common in reporting/analytics tables, often paired with caching or materialized views.

---

## 15. Transactions & ACID

A **transaction** is a sequence of operations executed as a single logical unit — either all succeed (commit) or all fail (rollback).

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- or
ROLLBACK;

-- Savepoints — partial rollback within a transaction
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SAVEPOINT sp1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
ROLLBACK TO sp1;     -- undoes only the second update
COMMIT;
```

**ACID properties:**
| Property | Meaning |
|---|---|
| **Atomicity** | All operations in a transaction succeed, or none do |
| **Consistency** | A transaction brings the database from one valid state to another, respecting all constraints |
| **Isolation** | Concurrent transactions don't interfere with each other's intermediate state |
| **Durability** | Once committed, changes survive even a crash (written to durable storage) |

Only the **InnoDB** storage engine supports full ACID transactions in MySQL — **MyISAM does not**.

---

## 16. Storage Engines

| Engine | Transactions | Locking | Foreign Keys | Crash Recovery | Use Case |
|---|---|---|---|---|---|
| **InnoDB** (default) | Yes | Row-level | Yes | Yes (via redo log) | General purpose — virtually always the right default choice today |
| **MyISAM** | No | Table-level | No | Limited | Legacy; read-heavy, no transaction needs (rare today) |
| **Memory** | No | Table-level | No | No (data lost on restart) | Temporary, very fast lookup tables |
| **Archive** | No | — | No | — | Compressed, append-only historical/log data |

```sql
SHOW ENGINES;
ALTER TABLE my_table ENGINE = InnoDB;
CREATE TABLE t (...) ENGINE = InnoDB;
```

**InnoDB vs MyISAM (very common interview question):** InnoDB supports transactions (ACID), row-level locking (better concurrency), and foreign key constraints, and uses a clustered index structure with crash recovery via its redo/undo logs. MyISAM is simpler/historically faster for pure read-heavy, no-transaction workloads but lacks transactions, uses table-level locking (poor write concurrency), and has no foreign key support — InnoDB is the modern default for virtually all use cases.

---

## 17. Locking & Concurrency

```sql
-- Row-level locking examples (InnoDB)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;     -- exclusive lock, blocks other writers/lockers
SELECT * FROM accounts WHERE id = 1 LOCK IN SHARE MODE;   -- (or FOR SHARE in 8.0+) shared lock, blocks writers but allows other readers
```

| Lock Type | Description |
|---|---|
| **Shared lock (S)** | Allows other transactions to read but not write the locked row |
| **Exclusive lock (X)** | Blocks other transactions from reading (with locking reads) or writing the row |
| **Row-level locking** | InnoDB locks individual rows, allowing high concurrency |
| **Table-level locking** | Locks the entire table (MyISAM, or explicit `LOCK TABLES`) |
| **Deadlock** | Two transactions waiting on each other's locks indefinitely — InnoDB automatically detects and resolves by rolling back one transaction |

```sql
SHOW ENGINE INNODB STATUS;     -- inspect locks, deadlocks, transaction info
```

---

## 18. Isolation Levels

Controls how/when changes made by one transaction become visible to others — a trade-off between consistency and concurrency performance.

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| **READ UNCOMMITTED** | Possible | Possible | Possible |
| **READ COMMITTED** | Prevented | Possible | Possible |
| **REPEATABLE READ** (MySQL default) | Prevented | Prevented | Prevented* |
| **SERIALIZABLE** | Prevented | Prevented | Prevented |

*MySQL's InnoDB REPEATABLE READ uses **MVCC (Multi-Version Concurrency Control)** and gap locking, which also prevents most phantom reads — stricter than the SQL standard technically requires for this level.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT @@transaction_isolation;
```

**Read phenomena explained:**
- **Dirty read** — reading uncommitted changes from another transaction (which might later roll back)
- **Non-repeatable read** — re-reading the same row within a transaction gives different results because another transaction modified/committed it in between
- **Phantom read** — re-running the same query within a transaction returns a different *set* of rows because another transaction inserted/deleted rows matching the criteria

---

## 19. Views

A **view** is a saved, virtual table defined by a query — doesn't store data itself (except materialized views, which MySQL doesn't natively support), just re-runs the underlying query each time it's referenced.

```sql
CREATE VIEW high_earners AS
SELECT first_name, last_name, salary
FROM employees
WHERE salary > 80000;

SELECT * FROM high_earners;
DROP VIEW high_earners;
CREATE OR REPLACE VIEW high_earners AS SELECT ...;
```

**Benefits:** Simplifies complex/repeated queries, provides a security layer (expose only certain columns/rows to certain users without granting access to the base table), and creates a stable interface even if underlying table structure changes.

---

## 20. Stored Procedures, Functions & Triggers

```sql
-- Stored Procedure
DELIMITER //
CREATE PROCEDURE GetEmployeesByDept(IN dept_id INT)
BEGIN
    SELECT * FROM employees WHERE department_id = dept_id;
END //
DELIMITER ;

CALL GetEmployeesByDept(3);

-- Function
DELIMITER //
CREATE FUNCTION GetFullName(fname VARCHAR(50), lname VARCHAR(50))
RETURNS VARCHAR(101)
DETERMINISTIC
BEGIN
    RETURN CONCAT(fname, ' ', lname);
END //
DELIMITER ;

SELECT GetFullName(first_name, last_name) FROM employees;

-- Trigger
DELIMITER //
CREATE TRIGGER before_employee_update
BEFORE UPDATE ON employees
FOR EACH ROW
BEGIN
    INSERT INTO employee_audit (employee_id, old_salary, new_salary, changed_at)
    VALUES (OLD.id, OLD.salary, NEW.salary, NOW());
END //
DELIMITER ;
```

**Procedure vs Function (interview point):** A function must return exactly one value and can be used inline within SQL statements (e.g., in a `SELECT`); a procedure may return zero, one, or multiple result sets/output parameters, and is called separately via `CALL` — not embeddable directly inside a query expression.

**Trigger timing/events:** `BEFORE`/`AFTER` × `INSERT`/`UPDATE`/`DELETE` — six possible trigger combinations per table. Common uses: auditing, enforcing complex business rules, maintaining denormalized/derived columns.

---

## 21. Query Optimization & EXPLAIN

```sql
EXPLAIN SELECT * FROM employees WHERE department_id = 3;
EXPLAIN ANALYZE SELECT * FROM employees WHERE department_id = 3;   -- MySQL 8.0.18+, shows actual execution stats
```

**Key `EXPLAIN` columns to know:**
| Column | Meaning |
|---|---|
| `type` | Join/access type — order from best to worst: `system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL` (full table scan — usually bad) |
| `key` | Which index MySQL actually decided to use (NULL = no index used) |
| `rows` | Estimated rows MySQL must examine |
| `Extra` | Notes like `Using index` (good — covering index), `Using filesort` (bad — extra sort step), `Using temporary` (bad — temp table needed) |

**Common optimization techniques:**
- Add indexes on columns used in `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`
- Avoid `SELECT *` — fetch only needed columns (also enables covering indexes)
- Avoid functions on indexed columns in `WHERE` (e.g., `WHERE YEAR(hire_date) = 2023` can't use an index on `hire_date` efficiently — rewrite as a range condition instead)
- Use `LIMIT` for pagination, but beware large `OFFSET` values are still slow (MySQL scans and discards the skipped rows)
- Batch large `INSERT`/`UPDATE` operations rather than row-by-row
- Use composite indexes matching common multi-column query patterns (leftmost-prefix rule)
- Periodically run `ANALYZE TABLE` so the optimizer has accurate statistics

---

## 22. Replication

Copies data from one MySQL server (**source/master**) to one or more other servers (**replica/slave**), used for read scaling, high availability, and backups.

```
       ┌─────────────┐
       │   Source      │  (writes happen here)
       │  (Primary)     │
       └──────┬──────┘
              │ binary log (binlog) stream
     ┌────────┼────────┐
     ▼                  ▼
┌──────────┐      ┌──────────┐
│ Replica 1  │      │ Replica 2  │  (read-only, serve read traffic)
└──────────┘      └──────────┘
```

**How it works:** The source records all data-changing statements/row events to its **binary log (binlog)**. Each replica's I/O thread pulls binlog events from the source, writes them to its own **relay log**, and the replica's SQL thread applies them locally.

**Replication types:**
- **Asynchronous** (default) — source doesn't wait for replicas to acknowledge; fastest but replicas can lag and data loss is possible on source failure before replication catches up
- **Semi-synchronous** — source waits for at least one replica to acknowledge receipt before committing; reduces (but doesn't eliminate) data-loss risk
- **Group Replication / InnoDB Cluster** — multi-primary, fully synchronous consensus-based replication for high availability

```sql
-- On source
SHOW MASTER STATUS;
-- On replica
CHANGE REPLICATION SOURCE TO SOURCE_HOST='...', SOURCE_USER='...', SOURCE_LOG_FILE='...', SOURCE_LOG_POS=...;
START REPLICA;
SHOW REPLICA STATUS\G
```

**Use cases:** Scaling read traffic (route reads to replicas, writes to source), disaster recovery, zero-downtime backups (backup from a replica instead of the live source), and analytics workloads isolated from production traffic.

---

## 23. Backup & Recovery

```bash
# Logical backup (SQL dump) — portable, human-readable, slower for large DBs
mysqldump -u root -p mydb > backup.sql
mysqldump -u root -p --all-databases > full_backup.sql
mysql -u root -p mydb < backup.sql        # restore

# Physical backup (binary files) — much faster for large databases, requires same MySQL version/engine compatibility
# Common tool: Percona XtraBackup (hot backup, no downtime)
xtrabackup --backup --target-dir=/backup/

# Point-in-time recovery (PITR) — restore a full backup, then replay binlogs up to a specific point
mysqlbinlog binlog.000123 | mysql -u root -p
```

**`mysqldump` vs physical backup (interview point):** `mysqldump` produces a logical SQL script (portable across versions/platforms, but slow to restore for large databases since it re-executes every statement). Physical backups (e.g., XtraBackup) copy the actual data files directly — much faster backup/restore for large datasets, but tied to compatible MySQL versions/configurations.

---

## 24. Partitioning

Splits a large table into smaller physical pieces while still being queried as a single logical table — improves performance and manageability for very large tables.

```sql
CREATE TABLE sales (
    id INT,
    sale_date DATE,
    amount DECIMAL(10,2)
)
PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);
```

**Partition types:** `RANGE`, `LIST`, `HASH`, `KEY`. Benefits: queries that filter on the partition key can skip irrelevant partitions entirely ("partition pruning"), and maintenance operations (e.g., dropping old data) can target a single partition instead of large `DELETE` operations.

---

## 25. Security

1. **Use strong authentication** — `mysql_secure_installation`, disable anonymous accounts, disable remote root login.
2. **Principle of least privilege** — grant only the specific privileges each user/application actually needs.
3. **Use parameterized queries / prepared statements** in application code — never concatenate user input into SQL strings (prevents **SQL injection**).
4. **Encrypt connections** — require SSL/TLS for client connections, especially over untrusted networks.
5. **Encrypt data at rest** — InnoDB tablespace encryption for sensitive data.
6. **Regularly patch/update** MySQL to get security fixes.
7. **Restrict network exposure** — bind to internal interfaces only, use firewalls/security groups, avoid exposing port 3306 publicly.
8. **Audit logging** — track sensitive operations (MySQL Enterprise Audit, or open-source alternatives).
9. **Don't store secrets/passwords in plaintext** in the database — hash passwords (bcrypt/argon2) at the application layer.

```sql
-- SQL injection example (vulnerable — never do this)
-- query = "SELECT * FROM users WHERE username = '" + input + "'"
-- If input = "' OR '1'='1", the query becomes:
-- SELECT * FROM users WHERE username = '' OR '1'='1'   -- returns all rows!

-- Safe: use parameterized queries (example shown conceptually, actual syntax depends on language/driver)
-- PreparedStatement: SELECT * FROM users WHERE username = ?
```

---

## 26. User Management & Privileges

```sql
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
CREATE USER 'app_user'@'%' IDENTIFIED BY 'StrongPassword123!';   -- '%' = any host (use cautiously)

GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'app_user'@'localhost';
GRANT ALL PRIVILEGES ON mydb.* TO 'admin_user'@'localhost';
GRANT SELECT ON mydb.employees TO 'readonly_user'@'%';

REVOKE INSERT ON mydb.* FROM 'app_user'@'localhost';

SHOW GRANTS FOR 'app_user'@'localhost';
DROP USER 'app_user'@'localhost';

FLUSH PRIVILEGES;     -- reload grant tables (needed after some direct edits to mysql.user)
```

**Common privilege levels:** Global (`*.*`), Database (`db.*`), Table (`db.table`), Column-level (`GRANT SELECT (col1, col2) ON db.table`).

---

## 27. Common Functions

```sql
-- String functions
CONCAT(a, b), SUBSTRING(str, pos, len), LENGTH(str), UPPER(str), LOWER(str), TRIM(str), REPLACE(str, from, to)

-- Numeric functions
ROUND(x, d), CEIL(x), FLOOR(x), ABS(x), MOD(a, b)

-- Date functions
NOW(), CURDATE(), DATEDIFF(d1, d2), DATE_ADD(date, INTERVAL 1 DAY), DATE_FORMAT(date, '%Y-%m-%d')

-- Conditional
CASE WHEN salary > 80000 THEN 'High' WHEN salary > 50000 THEN 'Medium' ELSE 'Low' END
IFNULL(value, default_value)        -- like COALESCE but only 2 args
COALESCE(val1, val2, val3)          -- first non-null value
IF(condition, true_val, false_val)
```

---

## 28. Window Functions

MySQL 8.0+ supports window functions — perform calculations across a set of rows related to the current row, **without** collapsing them into a single group (unlike `GROUP BY`).

```sql
-- Row number within each department, ordered by salary
SELECT name, department_id, salary,
       ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rank_in_dept
FROM employees;

-- Running total
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;

-- RANK vs DENSE_RANK vs ROW_NUMBER
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS rnk,           -- gaps after ties (1,2,2,4)
       DENSE_RANK() OVER (ORDER BY salary DESC) AS d_rnk,    -- no gaps after ties (1,2,2,3)
       ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num   -- always unique (1,2,3,4) even with ties
FROM employees;

-- LAG/LEAD — access previous/next row's value
SELECT order_date, amount,
       LAG(amount, 1) OVER (ORDER BY order_date) AS prev_amount,
       LEAD(amount, 1) OVER (ORDER BY order_date) AS next_amount
FROM orders;
```

---

## 29. Troubleshooting & Performance

```sql
SHOW PROCESSLIST;                      -- view active connections/queries
SHOW FULL PROCESSLIST;
KILL <process_id>;                      -- terminate a stuck query/connection

SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';

SHOW ENGINE INNODB STATUS;              -- deep InnoDB diagnostics (locks, deadlocks, buffer pool)

-- Slow query log
SHOW VARIABLES LIKE 'slow_query_log%';
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;         -- log queries taking > 1 second
```

**Common performance issues:**
| Symptom | Likely cause |
|---|---|
| Slow `SELECT` queries | Missing index, full table scan (`type: ALL` in EXPLAIN) |
| Slow writes | Too many indexes on the table, or large transactions |
| Lock waits/timeouts | Long-running transactions holding locks, or deadlocks |
| High connection count | Connection leaks in application code, no connection pooling |
| `Using filesort` / `Using temporary` in EXPLAIN | Query needs better indexing for ORDER BY/GROUP BY |

---

## 30. Best Practices Summary

- Always define a primary key on every table
- Use InnoDB for virtually all tables (transactions, row locking, FK support)
- Use the appropriate, smallest data type that fits the data (saves space, improves performance)
- Index columns used in `WHERE`/`JOIN`/`ORDER BY`/`GROUP BY` — but don't over-index
- Use `EXPLAIN` before deploying complex/slow queries
- Avoid `SELECT *` in application code
- Use prepared statements/parameterized queries to prevent SQL injection
- Wrap related multi-statement writes in transactions
- Normalize for data integrity, denormalize selectively for read performance when justified
- Regularly back up databases and test the restore process
- Monitor slow query logs and act on them
- Use replication for read scaling and HA, not as a substitute for backups

---

## 31. Cheat Sheet

```sql
-- Structure
SHOW DATABASES; USE db; SHOW TABLES; DESCRIBE table;
CREATE TABLE / ALTER TABLE / DROP TABLE / TRUNCATE TABLE

-- CRUD
SELECT col FROM table WHERE cond ORDER BY col LIMIT n;
INSERT INTO table (col) VALUES (val);
UPDATE table SET col = val WHERE cond;
DELETE FROM table WHERE cond;

-- Joins
SELECT * FROM a INNER/LEFT/RIGHT JOIN b ON a.id = b.a_id;

-- Aggregation
SELECT col, COUNT(*)/SUM()/AVG() FROM table GROUP BY col HAVING cond;

-- Indexes
CREATE INDEX idx ON table(col);
EXPLAIN SELECT ...;

-- Transactions
START TRANSACTION; ... COMMIT; / ROLLBACK;

-- Users
CREATE USER 'u'@'host' IDENTIFIED BY 'pwd';
GRANT privs ON db.* TO 'u'@'host';

-- Backup
mysqldump -u root -p db > backup.sql
mysql -u root -p db < backup.sql
```

---

## 32. Interview Questions & Answers

**Q1: What is the difference between `WHERE` and `HAVING`?**
A: `WHERE` filters individual rows **before** any grouping/aggregation occurs. `HAVING` filters **groups** after `GROUP BY` aggregation has been computed — used when the filter condition depends on an aggregate function (e.g., `HAVING COUNT(*) > 5`), which isn't available at the per-row stage `WHERE` operates on.

**Q2: Difference between `DELETE`, `TRUNCATE`, and `DROP`?**
A: `DELETE` removes rows one at a time (can be filtered with `WHERE`, is transactional/can be rolled back, fires triggers, slower for large tables). `TRUNCATE` removes all rows at once, resets `AUTO_INCREMENT`, is faster, but generally can't be rolled back and doesn't fire row-level triggers. `DROP` removes the entire table structure along with its data.

**Q3: What's the difference between `InnoDB` and `MyISAM`?**
A: InnoDB supports transactions (ACID-compliant), uses row-level locking for better write concurrency, supports foreign keys, and has crash recovery via redo/undo logs — it's the modern default. MyISAM has no transaction support, uses table-level locking (poor concurrency under writes), no foreign keys, but was historically slightly faster for simple read-heavy workloads — largely legacy today.

**Q4: Explain ACID properties.**
A: **Atomicity** — a transaction's operations all succeed or all fail together. **Consistency** — a transaction moves the database from one valid state to another, respecting constraints. **Isolation** — concurrent transactions don't see each other's uncommitted intermediate state (governed by isolation levels). **Durability** — once committed, changes persist even through a crash.

**Q5: What is a primary key vs a foreign key?**
A: A **primary key** uniquely identifies each row in its own table and cannot be NULL. A **foreign key** is a column (or set of columns) in one table that references the primary key (or a unique key) of another table, enforcing referential integrity between the two.

**Q6: What is an index, and what's the trade-off of adding one?**
A: An index is an auxiliary data structure (typically a B-Tree) that speeds up lookups on a column by avoiding full table scans. The trade-off: it consumes additional storage and slows down writes (`INSERT`/`UPDATE`/`DELETE`), since the index must be updated alongside the underlying data — so indexes should be added deliberately for columns actually used in filtering/sorting/joining, not indiscriminately.

**Q7: What is the difference between a clustered and a non-clustered index?**
A: A clustered index determines the **physical storage order** of the table's rows — in InnoDB, this is always the primary key, so there's exactly one clustered index per table. A non-clustered (secondary) index stores the indexed column(s) alongside a pointer back to the primary key, requiring an extra lookup into the clustered index to retrieve the full row — there can be many secondary indexes per table.

**Q8: What's the difference between `INNER JOIN` and `LEFT JOIN`?**
A: `INNER JOIN` returns only rows that have matching values in both tables. `LEFT JOIN` returns all rows from the left table, plus matching rows from the right table where they exist — rows from the left table with no match get `NULL` for the right table's columns.

**Q9: What is normalization, and what are the first three normal forms?**
A: Normalization organizes a schema to reduce data redundancy and avoid update/insert/delete anomalies. **1NF**: atomic column values, no repeating groups. **2NF**: 1NF plus every non-key column fully depends on the whole primary key (no partial dependency on part of a composite key). **3NF**: 2NF plus no transitive dependencies (a non-key column shouldn't depend on another non-key column).

**Q10: When would you denormalize a database?**
A: When read performance matters more than strict redundancy avoidance — e.g., duplicating a frequently-joined column directly into a table to avoid expensive joins in a high-traffic read path, common in reporting/analytics tables or heavily cached read models. It's a deliberate trade-off accepting some redundancy/update complexity for query speed.

**Q11: What is a transaction, and how do you use `COMMIT`/`ROLLBACK`?**
A: A transaction groups multiple SQL statements into a single atomic unit of work, started with `START TRANSACTION`. `COMMIT` permanently applies all the changes; `ROLLBACK` undoes all changes made since the transaction began (or since a `SAVEPOINT`), ensuring partial failures don't leave the database in an inconsistent state.

**Q12: What are isolation levels, and what does MySQL use by default?**
A: Isolation levels control how visible one transaction's in-progress changes are to other concurrent transactions, trading off consistency against concurrency: `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`. MySQL's InnoDB defaults to **REPEATABLE READ**, using MVCC to give each transaction a consistent snapshot of the data while still allowing high concurrency.

**Q13: What is a deadlock, and how does MySQL handle it?**
A: A deadlock occurs when two or more transactions each hold a lock the other needs, waiting on each other indefinitely. InnoDB automatically detects deadlocks and resolves them by rolling back one of the involved transactions (typically the one that has done less work), returning an error to that transaction's client so the application can retry.

**Q14: What's the difference between a stored procedure and a function in MySQL?**
A: A function must return exactly one value and can be called inline within a SQL expression (e.g., inside a `SELECT`). A stored procedure can return zero, one, or multiple result sets/output parameters but must be invoked separately via `CALL` — it cannot be embedded directly inside another query's expression.

**Q15: How would you prevent SQL injection?**
A: Use parameterized queries / prepared statements in application code instead of concatenating user input directly into SQL strings — this ensures user input is always treated as data, never as executable SQL syntax. Additional defenses include input validation, using an ORM that parameterizes by default, and applying least-privilege database permissions to limit blast radius even if injection occurs.

**Q16: What is MySQL replication, and what's the difference between asynchronous and semi-synchronous replication?**
A: Replication copies data changes from a source server to one or more replicas via the binary log, used for read scaling, high availability, and offloading backups/analytics from the primary. In **asynchronous** replication (default), the source commits without waiting for any replica acknowledgment — fastest, but risks losing recent transactions if the source fails before replicas catch up. **Semi-synchronous** replication requires at least one replica to acknowledge receipt before the source commits, reducing (but not eliminating) that data-loss window.

**Q17: How do you use `EXPLAIN` to optimize a slow query?**
A: `EXPLAIN <query>` shows MySQL's query execution plan — check the `type` column (avoid `ALL`, a full table scan), the `key` column (confirm an appropriate index is actually being used, not `NULL`), the `rows` estimate (lower is better), and the `Extra` column for red flags like `Using filesort` or `Using temporary`, which indicate the query needs better indexing to avoid expensive extra sort/temp-table steps.

**Q18: What is the difference between `COUNT(*)`, `COUNT(column)`, and `COUNT(DISTINCT column)`?**
A: `COUNT(*)` counts all rows regardless of NULLs. `COUNT(column)` counts only rows where that specific column is non-NULL. `COUNT(DISTINCT column)` counts the number of unique non-NULL values in that column.

**Q19: What's the difference between `UNION` and `UNION ALL`?**
A: `UNION` combines result sets from multiple `SELECT` statements and removes duplicate rows (requires an extra sort/dedup step, so it's slower). `UNION ALL` combines them without removing duplicates — faster, and should be preferred whenever you know there won't be duplicates or duplicates are acceptable/expected.

**Q20: How would you design a database schema for a many-to-many relationship?**
A: Use a **junction (bridge/association) table** containing foreign keys referencing the primary keys of both related tables (e.g., `student_courses` with `student_id` and `course_id`, often combined as a composite primary key) — this resolves the many-to-many relationship into two one-to-many relationships, which is how relational databases natively support it.

**Q21: What is the N+1 query problem, and how do you avoid it?**
A: It occurs when code runs one query to fetch a list of N records, then runs N additional queries (one per record) to fetch related data — instead of a single efficient join or batched query. It's avoided by using `JOIN`s, `IN (...)` batched lookups, or ORM eager-loading features to fetch related data in one (or few) round trips instead of N+1.

**Q22: What's the difference between `CHAR` and `VARCHAR`?**
A: `CHAR(n)` is fixed-length — MySQL always allocates `n` bytes, padding shorter values with spaces, which can be slightly faster for fixed-size data. `VARCHAR(n)` is variable-length — it only uses as much storage as the actual string requires (plus 1-2 bytes to store the length), making it more space-efficient for variable-length text.

---

### Final interview tip
Be ready to explain **ACID properties**, **InnoDB vs MyISAM**, **clustered vs non-clustered indexes**, and to **write joins and `GROUP BY`/`HAVING` queries on a whiteboard** from a simple schema (e.g., employees/departments) — these fundamentals come up in nearly every backend/data engineering interview. Also expect at least one **query optimization** question involving `EXPLAIN` output.
