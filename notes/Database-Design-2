# Star Schema vs Snowflake Schema

## Introduction to Dimensional Modeling

The **star schema** is the simplest form of the dimensional model. Many use the terms "star schema" and "dimensional model" interchangeably.

### Key Components
- **Fact Tables:** Hold records of metrics
- **Dimension Tables:** Describe the metrics in fact tables

---

## Star Schema

### Structure
The star schema extends in **one dimension** - directly connecting fact tables to dimension tables.

### Business Example
A company sells books in bulk to bookstores across the US and Canada. They track book sales in a database.

### Star Schema Diagram

```
         ┌─────────────┐
         │   book      │
         │   dimension │
         └──────┬──────┘
                │
         ┌──────▼──────┐
         │   time      │◄───┐
         │   dimension │    │
         └──────┬──────┘    │
                │           │
         ┌──────▼──────────▼┐
         │                  │
         │   FACT TABLE     │
         │   (book_sales)   │
         │                  │
         └──────┬───────────┘
                │
         ┌──────▼──────┐
         │   store     │
         │   dimension │
         └─────────────┘
```

### Fact Table Example

**fact_sales**
| sale_id (PK) | book_id (FK) | time_id (FK) | store_id (FK) | sales_amount | quantity |
|--------------|--------------|--------------|---------------|--------------|----------|
| 1            | 101          | 20181201     | 5             | 250.00       | 10       |
| 2            | 205          | 20181205     | 3             | 180.00       | 6        |
| 3            | 101          | 20181210     | 5             | 125.00       | 5        |

### Dimension Tables Examples

**dim_book** (Denormalized)
| book_id (PK) | title                    | author           | publisher      | genre   |
|--------------|--------------------------|------------------|----------------|---------|
| 101          | Kindred                  | Octavia E Butler | Beacon Press   | Sci-Fi  |
| 205          | The Left Hand of Darkness| Ursula K Le Guin | Ace Books      | Sci-Fi  |
| 310          | Dune                     | Frank Herbert    | Chilton Books  | Sci-Fi  |

**dim_time**
| time_id (PK) | day | month | quarter | year |
|--------------|-----|-------|---------|------|
| 20181201     | 1   | 12    | Q4      | 2018 |
| 20181205     | 5   | 12    | Q4      | 2018 |
| 20181210     | 10  | 12    | Q4      | 2018 |

**dim_store** (Denormalized)
| store_id (PK) | store_name        | city      | state      | country |
|---------------|-------------------|-----------|------------|---------|
| 5             | BookWorld         | Vancouver | BC         | Canada  |
| 3             | Page Turner       | Brooklyn  | New York   | USA     |
| 7             | Novel Ideas       | Brooklyn  | New York   | USA     |
| 9             | Reading Corner    | Brooklyn  | New York   | USA     |

**Notice the redundancy:** "Brooklyn", "New York", and "USA" are repeated multiple times!

### Relationships
- **One-to-Many:** A store can be part of many book sales, but one sale belongs to only one store
- The schema resembles a **star** with extension points

---

## Snowflake Schema

### Structure
The snowflake schema is an **extension of the star schema** that extends over **more than one dimension**. This happens because dimension tables are **normalized**.

### What is Normalization?
**Definition:** A technique that divides tables into smaller tables and connects them via relationships.

**Goals:**
- Reduce redundancy
- Increase data integrity

**Method:** Identify repeating groups of data and create new tables for them.

---

## Snowflake Schema Examples

### Book Dimension - Normalized

In the star schema, we identified that:
- Authors often publish more than one book
- Publishers definitely publish many books
- Many books share genres

**Star Schema (Denormalized):**
```
dim_book: book_id, title, author, publisher, genre
```

**Snowflake Schema (Normalized):**

**dim_book**
| book_id (PK) | title                     | author_id (FK) | publisher_id (FK) | genre_id (FK) |
|--------------|---------------------------|----------------|-------------------|---------------|
| 101          | Kindred                   | 1              | 10                | 5             |
| 205          | The Left Hand of Darkness | 2              | 11                | 5             |
| 310          | Dune                      | 3              | 12                | 5             |

**dim_author**
| author_id (PK) | author_name      |
|----------------|------------------|
| 1              | Octavia E Butler |
| 2              | Ursula K Le Guin |
| 3              | Frank Herbert    |

**dim_publisher**
| publisher_id (PK) | publisher_name |
|-------------------|----------------|
| 10                | Beacon Press   |
| 11                | Ace Books      |
| 12                | Chilton Books  |

**dim_genre**
| genre_id (PK) | genre_name |
|---------------|------------|
| 5             | Sci-Fi     |
| 6             | Fantasy    |
| 7             | Mystery    |

---

### Store Dimension - Normalized

In the star schema, we identified that:
- Cities, states, and countries can have multiple bookstores
- A city stays in the same state and country (hierarchical relationship)

**Star Schema (Denormalized):**
```
dim_store: store_id, store_name, city, state, country
```

**Snowflake Schema (Normalized):**

**dim_store**
| store_id (PK) | store_name     | city_id (FK) |
|---------------|----------------|--------------|
| 5             | BookWorld      | 100          |
| 3             | Page Turner    | 200          |
| 7             | Novel Ideas    | 200          |
| 9             | Reading Corner | 200          |

**dim_city**
| city_id (PK) | city_name | state_id (FK) |
|--------------|-----------|---------------|
| 100          | Vancouver | 10            |
| 200          | Brooklyn  | 20            |

**dim_state**
| state_id (PK) | state_name | country_id (FK) |
|---------------|------------|-----------------|
| 10            | BC         | 1               |
| 20            | New York   | 2               |

**dim_country**
| country_id (PK) | country_name |
|-----------------|--------------|
| 1               | Canada       |
| 2               | USA          |

**Notice:** "Brooklyn" is now stored only ONCE instead of three times!

---

### Time Dimension - Normalized

**dim_day**
| day_id (PK) | day | month_id (FK) |
|-------------|-----|---------------|
| 20181201    | 1   | 201812        |
| 20181205    | 5   | 201812        |

**dim_month**
| month_id (PK) | month | quarter_id (FK) |
|---------------|-------|-----------------|
| 201812        | 12    | 20184           |

**dim_quarter**
| quarter_id (PK) | quarter | year_id (FK) |
|-----------------|---------|--------------|
| 20184           | Q4      | 2018         |

**dim_year**
| year_id (PK) | year |
|--------------|------|
| 2018         | 2018 |

A day is part of a month, which is part of a quarter, which is part of a year!

---

## Query Comparison

### Scenario
Get the quantity of all books by Octavia E. Butler sold in Vancouver in Q4 of 2018.

### Star Schema Query (Denormalized)
```sql
SELECT SUM(fs.quantity)
FROM fact_sales fs
INNER JOIN dim_book db ON fs.book_id = db.book_id
INNER JOIN dim_time dt ON fs.time_id = dt.time_id
INNER JOIN dim_store ds ON fs.store_id = ds.store_id
WHERE db.author = 'Octavia E Butler'
  AND ds.city = 'Vancouver'
  AND dt.quarter = 'Q4'
  AND dt.year = 2018;
```
**3 joins** - Simple and fast!

### Snowflake Schema Query (Normalized)
```sql
SELECT SUM(fs.quantity)
FROM fact_sales fs
INNER JOIN dim_book db ON fs.book_id = db.book_id
INNER JOIN dim_author da ON db.author_id = da.author_id
INNER JOIN dim_time_day dd ON fs.time_id = dd.day_id
INNER JOIN dim_time_month dm ON dd.month_id = dm.month_id
INNER JOIN dim_time_quarter dq ON dm.quarter_id = dq.quarter_id
INNER JOIN dim_time_year dy ON dq.year_id = dy.year_id
INNER JOIN dim_store ds ON fs.store_id = ds.store_id
INNER JOIN dim_city dc ON ds.city_id = dc.city_id
WHERE da.author_name = 'Octavia E Butler'
  AND dc.city_name = 'Vancouver'
  AND dq.quarter = 'Q4'
  AND dy.year = 2018;
```
**8 joins** - Much more complex and slower!

---

## Normalization: Pros and Cons

### ✅ Advantages

#### 1. Saves Space
**Example - Store Table Redundancy:**

**Denormalized (Star Schema):**
| store_id | store_name     | city     | state    | country |
|----------|----------------|----------|----------|---------|
| 3        | Page Turner    | Brooklyn | New York | USA     |
| 7        | Novel Ideas    | Brooklyn | New York | USA     |
| 9        | Reading Corner | Brooklyn | New York | USA     |

Strings like "Brooklyn", "New York", and "USA" are stored **multiple times**.

**Normalized (Snowflake Schema):**
- "Brooklyn" stored ONCE in dim_city
- "New York" stored ONCE in dim_state
- "USA" stored ONCE in dim_country

**No data redundancy!**

#### 2. Better Data Integrity

**a) Enforces Data Consistency**
- Prevents variations like "CA" vs "California" vs "calif"
- Referential integrity ensures naming conventions
- One source of truth for each value

**b) Safer Modifications**
- Update state spelling in ONE place (dim_state table)
- Change automatically applies to all stores in that state
- No risk of missing records or creating inconsistencies

**c) Easier Schema Changes**
- Smaller, object-organized tables
- Can extend tables without altering large central tables
- More modular database structure

---

### ❌ Disadvantages

#### 1. More Complex Queries
- Requires many more joins
- Harder to write and understand
- More prone to errors

#### 2. Slower Query Performance
- Each join adds processing time
- More tables to scan
- Particularly problematic for analytics

#### 3. More Tables to Manage
- Increased database complexity
- More foreign key relationships to maintain
- Harder for new users to navigate

---

## When to Use Each Schema

### Star Schema (Denormalized) ⭐
**Best for:** OLAP (Online Analytical Processing)

**Characteristics:**
- Read-intensive operations
- Running analytics on data
- Need fast query performance
- Prioritize query speed over storage space

**Use when:**
- Building data warehouses
- Creating business intelligence dashboards
- Running complex analytical queries frequently
- Query performance is critical

---

### Snowflake Schema (Normalized) ❄️
**Best for:** OLTP (Online Transaction Processing)

**Characteristics:**
- Write-intensive operations
- Frequent updates and inserts
- Need data consistency
- Want to minimize storage space

**Use when:**
- Adding data quickly and consistently
- Storage space is limited
- Data integrity is paramount
- Update operations are common

---

## Key Decision Factors

| Factor | Star Schema | Snowflake Schema |
|--------|-------------|------------------|
| **Query Speed** | Fast (fewer joins) | Slower (more joins) |
| **Storage Space** | More (redundancy) | Less (no redundancy) |
| **Data Integrity** | Good | Excellent |
| **Query Complexity** | Simple | Complex |
| **Maintenance** | Harder (redundancy) | Easier (normalized) |
| **Best for** | OLAP (Analytics) | OLTP (Transactions) |

---

## Summary

- **Star Schema** = Denormalized dimensions extending in one level
- **Snowflake Schema** = Normalized dimensions extending over multiple levels
- **Trade-off:** Query performance vs. storage space and data integrity
- **Choice depends on:** Whether your database is read-intensive (OLAP) or write-intensive (OLTP)
