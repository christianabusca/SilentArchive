# Database Views - Complete Guide

## What are Database Views?

### Definition
**A view is a virtual table** that is not part of the physical schema. The view itself isn't stored in physical memory; instead, **the query to create the view** is stored.

### Key Characteristics
- Virtual tables (not physically stored)
- Data comes from actual tables in the same database
- Can be queried like a regular table
- Don't require retyping common queries
- Add virtual tables without altering the database schema

---

## Creating Views

### Basic Syntax
```sql
CREATE VIEW view_name AS
  SELECT column1, column2, ...
  FROM table_name
  WHERE condition;
```

### Example: Science Fiction Books View

**Scenario:** Analysts frequently run analytics on science fiction books. Create a view to simplify their workflow.

**Database Schema (Snowflake):**
```
dim_book ──> dim_genre
    │
    └──> dim_author
```

**Tables:**

**dim_book**
| book_id | title                     | genre_id | author_id |
|---------|---------------------------|----------|-----------|
| 101     | Kindred                   | 5        | 1         |
| 205     | The Left Hand of Darkness | 5        | 2         |
| 310     | Dune                      | 5        | 3         |
| 401     | Pride and Prejudice       | 8        | 4         |

**dim_genre**
| genre_id | genre_name |
|----------|------------|
| 5        | Sci-Fi     |
| 8        | Classic    |

**dim_author**
| author_id | author_name      |
|-----------|------------------|
| 1         | Octavia E Butler |
| 2         | Ursula K Le Guin |
| 3         | Frank Herbert    |
| 4         | Jane Austen      |

**Original Query (Without View):**
```sql
SELECT b.title, a.author_name
FROM dim_book b
INNER JOIN dim_genre g ON b.genre_id = g.genre_id
INNER JOIN dim_author a ON b.author_id = a.author_id
WHERE g.genre_name = 'Sci-Fi';
```

**Creating the View:**
```sql
CREATE VIEW scifi_books AS
  SELECT b.title, a.author_name
  FROM dim_book b
  INNER JOIN dim_genre g ON b.genre_id = g.genre_id
  INNER JOIN dim_author a ON b.author_id = a.author_id
  WHERE g.genre_name = 'Sci-Fi';
```

**Querying the View:**
```sql
SELECT * FROM scifi_books;
```

**Result:**
| title                     | author_name      |
|---------------------------|------------------|
| Kindred                   | Octavia E Butler |
| The Left Hand of Darkness | Ursula K Le Guin |
| Dune                      | Frank Herbert    |

---

## Behind the Scenes

When you run:
```sql
SELECT * FROM scifi_books;
```

**What actually happens:**
The database executes the original query stored in the view definition:
```sql
SELECT b.title, a.author_name
FROM dim_book b
INNER JOIN dim_genre g ON b.genre_id = g.genre_id
INNER JOIN dim_author a ON b.author_id = a.author_id
WHERE g.genre_name = 'Sci-Fi';
```

**Important:** `scifi_books` is NOT a real table with physical memory. It's a virtual table created on-the-fly each time you query it.

---

## Viewing Existing Views

### List All Views (PostgreSQL)
```sql
SELECT * FROM INFORMATION_SCHEMA.views;
```
⚠️ This returns ALL views, including system views

### List Only User-Created Views
```sql
SELECT * FROM INFORMATION_SCHEMA.views
WHERE table_schema NOT IN ('pg_catalog', 'information_schema');
```
This excludes built-in DBMS views

---

## Benefits of Views

### 1. ✅ Minimal Storage
- Only stores the query statement
- Doesn't duplicate data
- No physical storage overhead

### 2. ✅ Access Control
**Example:** Sensitive employee data

**Original Table:**
| employee_id | name          | salary | ssn         | department |
|-------------|---------------|--------|-------------|------------|
| 1           | Alice Johnson | 85000  | 123-45-6789 | Engineering|
| 2           | Bob Smith     | 72000  | 987-65-4321 | Marketing  |

**Create view WITHOUT sensitive columns:**
```sql
CREATE VIEW employee_directory AS
  SELECT employee_id, name, department
  FROM employees;
```

**Result for users:**
| employee_id | name          | department  |
|-------------|---------------|-------------|
| 1           | Alice Johnson | Engineering |
| 2           | Bob Smith     | Marketing   |

Users can access employee information without seeing salary or SSN!

### 3. ✅ Masks Query Complexity
**Example:** Aggregating data from normalized snowflake schema

Without views, users must remember complex joins:
```sql
SELECT d.day, m.month, q.quarter, y.year, SUM(fs.sales_amount)
FROM fact_sales fs
JOIN dim_day d ON fs.time_id = d.day_id
JOIN dim_month m ON d.month_id = m.month_id
JOIN dim_quarter q ON m.quarter_id = q.quarter_id
JOIN dim_year y ON q.year_id = y.year_id
GROUP BY d.day, m.month, q.quarter, y.year;
```

With a view:
```sql
CREATE VIEW sales_by_date AS
  SELECT d.day, m.month, q.quarter, y.year, SUM(fs.sales_amount) as total_sales
  FROM fact_sales fs
  JOIN dim_day d ON fs.time_id = d.day_id
  JOIN dim_month m ON d.month_id = m.month_id
  JOIN dim_quarter q ON m.quarter_id = q.quarter_id
  JOIN dim_year y ON q.year_id = y.year_id
  GROUP BY d.day, m.month, q.quarter, y.year;

-- Now users simply query:
SELECT * FROM sales_by_date WHERE year = 2018;
```

---

## Managing Views

### Complex Views
Views can be as complex as needed:
- Aggregation functions (SUM, AVG, COUNT)
- Multiple joins
- Subqueries
- Conditional logic (CASE statements)

⚠️ **Caution:** Complex queries still need to execute, so be aware of execution time.

---

## Granting and Revoking Access

### Syntax
```sql
-- Grant privileges
GRANT privilege_type ON object_name TO user_or_role;

-- Revoke privileges
REVOKE privilege_type ON object_name FROM user_or_role;
```

### Common Privileges
| Privilege | Description |
|-----------|-------------|
| `SELECT` | Read data |
| `INSERT` | Add new rows |
| `UPDATE` | Modify existing data |
| `DELETE` | Remove rows |

### Examples

**Grant UPDATE to all users:**
```sql
GRANT UPDATE ON ratings TO PUBLIC;
```
(`PUBLIC` = all users)

**Revoke INSERT from specific user:**
```sql
REVOKE INSERT ON films FROM db_user;
```

---

## Updating Views

### UPDATE Command
```sql
UPDATE view_name
SET column1 = value1
WHERE condition;
```

**Important:** When you update a view, you're actually updating the **tables behind the view**.

### Updatable View Criteria
A view is updatable if:
- ✅ Made up of **one table only**
- ✅ Doesn't use **window functions**
- ✅ Doesn't use **aggregate functions** (SUM, AVG, COUNT, etc.)
- ✅ Doesn't use GROUP BY or HAVING

**Example of updatable view:**
```sql
CREATE VIEW active_employees AS
  SELECT employee_id, name, department
  FROM employees
  WHERE status = 'active';

-- This works (single table, no aggregates):
UPDATE active_employees
SET department = 'Sales'
WHERE employee_id = 5;
```

**Example of non-updatable view:**
```sql
CREATE VIEW department_counts AS
  SELECT department, COUNT(*) as employee_count
  FROM employees
  GROUP BY department;

-- This FAILS (uses aggregate function):
UPDATE department_counts
SET employee_count = 10
WHERE department = 'Sales';
```

---

## Inserting into Views

### INSERT Command
```sql
INSERT INTO view_name (column1, column2)
VALUES (value1, value2);
```

Similar to UPDATE, when you insert into a view, you're inserting into the **underlying table**.

### Criteria
Same as updatable views - typically requires:
- Single table
- No aggregates or window functions

### ⚠️ Best Practice
**Avoid modifying data through views.** Use views for **read-only purposes only**.

---

## Dropping Views

### Basic Syntax
```sql
DROP VIEW view_name;
```

### Parameters

**RESTRICT (default):**
```sql
DROP VIEW view_name RESTRICT;
```
- Returns an error if other objects depend on this view
- Safe option - prevents accidental data loss

**CASCADE:**
```sql
DROP VIEW view_name CASCADE;
```
- Drops the view AND any objects that depend on it
- ⚠️ Use with caution!

**Example:**
```
View A ──> View B ──> View C
```

If you run:
```sql
DROP VIEW view_a CASCADE;
```
Result: Views A, B, and C are all dropped!

---

## Redefining Views

### CREATE OR REPLACE Syntax
```sql
CREATE OR REPLACE VIEW view_name AS
  new_query;
```

### Requirements
The new query must generate:
- ✅ Same column names
- ✅ Same column order
- ✅ Same column data types
- ✅ Can add new columns at the end

**Example - This works:**
```sql
-- Original view
CREATE VIEW employee_summary AS
  SELECT employee_id, name, department
  FROM employees;

-- Redefine with added column
CREATE OR REPLACE VIEW employee_summary AS
  SELECT employee_id, name, department, hire_date
  FROM employees;
```

**Example - This fails:**
```sql
-- Original view
CREATE VIEW employee_summary AS
  SELECT employee_id, name, department
  FROM employees;

-- This FAILS (different column order)
CREATE OR REPLACE VIEW employee_summary AS
  SELECT name, employee_id, department
  FROM employees;
```

**Solution if criteria can't be met:**
```sql
DROP VIEW employee_summary;
CREATE VIEW employee_summary AS
  SELECT name, employee_id, department
  FROM employees;
```

---

## Altering View Properties

### ALTER VIEW Syntax
```sql
ALTER VIEW view_name [action];
```

### Common Actions
```sql
-- Rename view
ALTER VIEW old_name RENAME TO new_name;

-- Change owner
ALTER VIEW view_name OWNER TO new_owner;

-- Change schema
ALTER VIEW view_name SET SCHEMA new_schema;
```

---

## Materialized Views

### Two Types of Views

| Type | Storage | Performance | Freshness |
|------|---------|-------------|-----------|
| **Non-Materialized** (Regular) | Stores query only | Slower (runs query each time) | Always current |
| **Materialized** | Stores query RESULTS | Faster (precomputed) | Requires refresh |

### What are Materialized Views?

**Non-materialized view:**
- Stores the query
- Runs the query each time you access the view
- Creates a virtual table

**Materialized view:**
- Stores the query **results** on disk
- Query is precomputed
- Accesses stored results (no need to rerun query)
- Results must be refreshed/rematerialized

### Creating Materialized Views

```sql
CREATE MATERIALIZED VIEW view_name AS
  SELECT ...
  FROM ...;
```

### Refreshing Materialized Views

```sql
REFRESH MATERIALIZED VIEW view_name;
```

Refreshing means:
- Query is rerun
- Stored results are updated
- Can be scheduled (e.g., daily, hourly)

---

## When to Use Materialized Views

### ✅ Use When:

**1. Long execution time queries**
- Queries that take hours to complete
- Complex joins
- Processing large amounts of data

**Example:**
```sql
-- This query takes 2 hours to run
CREATE MATERIALIZED VIEW yearly_sales_analysis AS
  SELECT 
    p.product_name,
    c.category,
    d.year,
    SUM(s.sales_amount) as total_sales,
    AVG(s.quantity) as avg_quantity,
    COUNT(DISTINCT s.customer_id) as unique_customers
  FROM sales s
  JOIN products p ON s.product_id = p.product_id
  JOIN categories c ON p.category_id = c.category_id
  JOIN dates d ON s.date_id = d.date_id
  WHERE d.year BETWEEN 2015 AND 2024
  GROUP BY p.product_name, c.category, d.year;

-- Now queries run instantly:
SELECT * FROM yearly_sales_analysis WHERE year = 2024;
```

**2. Data warehouses (OLAP)**
- Used for analysis, not frequent updates
- Same queries run repeatedly
- Computational cost adds up

**3. Infrequently updated data**
- Historical data
- Reference data
- Archived information

### ❌ Don't Use When:

**1. Data updates frequently**
- Real-time data
- OLTP systems
- Live dashboards needing current data

**2. Storage is limited**
- Materialized views take up disk space
- Regular views use minimal storage

---

## Managing Materialized View Dependencies

### The Dependency Problem

**Example:**
```
Materialized View X ──> Materialized View Y
(Y depends on X)
```

**Scenario:**
- View X has a time-consuming query (takes 1 hour)
- View Y uses data from View X
- If Y refreshes before X finishes refreshing
- Y now has **out-of-date data**

### Dependency Chain

```
View A ──> View B ──> View C ──> View D
```

**Refreshing order matters!**
- Must refresh A before B
- Must refresh B before C
- Must refresh C before D

### Tools for Managing Dependencies

**1. Directed Acyclic Graphs (DAGs)**
- Track dependencies visually
- "Acyclic" = no circular dependencies
- View A can depend on View B, but View B cannot also depend on View A

**Example DAG:**
```
        View A
       /      \
   View B    View C
       \      /
        View D
```

**2. Pipeline Scheduler Tools**
- **Apache Airflow**
- **Luigi**
- Schedule REFRESH statements
- Handle dependencies automatically
- Optimize refresh timing

**3. Cron Jobs**
- UNIX-based job scheduler
- Schedule regular refresh operations

**Example scheduling strategy:**
```
Non-working hours (2 AM - 6 AM):
- 2:00 AM: Refresh View A (1 hour)
- 3:00 AM: Refresh View B (30 min)
- 3:30 AM: Refresh View C (30 min)
- 4:00 AM: Refresh View D (1 hour)
```

---

## Views Summary

### Regular (Non-Materialized) Views

**Best for:**
- Simplifying complex queries
- Access control
- Data that changes frequently
- Minimal storage requirements

**Characteristics:**
- Virtual table
- Runs query each time
- Always current data
- Slower performance for complex queries

---

### Materialized Views

**Best for:**
- Long-running queries
- Data warehouses (OLAP)
- Infrequently changing data
- Repeated analysis queries

**Characteristics:**
- Physical storage of results
- Fast query performance
- Requires scheduled refresh
- Data may be out-of-date

---

## Key Takeaways

1. **Views simplify complex queries** - Users don't need to remember complicated joins
2. **Views provide access control** - Hide sensitive data while sharing other information
3. **Regular views = minimal storage** - Only the query is stored
4. **Materialized views = speed** - Great for expensive queries, but need refresh management
5. **Use views for read-only** - Avoid INSERT/UPDATE operations through views
6. **Manage dependencies carefully** - Especially critical for materialized views
7. **Choose the right type** - Regular for current data, materialized for performance
