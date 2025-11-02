## Part 1: Database Roles and Access Control

### What are Database Roles?

**Definition:** A database role is an entity that contains information about:
1. **Privileges:** What the role can do (login, create databases, etc.)
2. **Authentication:** How the role interacts with the client (passwords, etc.)

**Key characteristics:**
- Roles are global across all databases in your cluster
- Roles can be assigned to one or more users
- Roles are separate from operating system users

---

### Creating Roles

#### Basic Syntax
```sql
CREATE ROLE role_name;
```

#### Examples

**1. Create a data analyst role:**
```sql
CREATE ROLE data_analyst;
```
*Currently empty - no privileges defined yet*

**2. Create an intern role with expiration:**
```sql
CREATE ROLE intern 
WITH PASSWORD 'secure_password' 
VALID UNTIL '2020-01-01';
```
*One second into 2020, the password becomes invalid*

**3. Create an admin role with database creation privilege:**
```sql
CREATE ROLE admin WITH CREATEDB;
```

---

### Modifying Roles

#### ALTER Syntax
```sql
ALTER ROLE role_name WITH attribute;
```

**Example - Add ability to create roles:**
```sql
ALTER ROLE admin WITH CREATEROLE;
```

---

### Common Role Attributes

| Attribute | Description |
|-----------|-------------|
| `LOGIN` | Allows role to log in |
| `SUPERUSER` | Has all privileges |
| `CREATEDB` | Can create databases |
| `CREATEROLE` | Can create other roles |
| `PASSWORD 'text'` | Sets password |
| `VALID UNTIL 'timestamp'` | Password expiration date |

---

### GRANT and REVOKE Privileges

#### Syntax
```sql
-- Grant privileges
GRANT privilege_type ON object_name TO role_name;

-- Revoke privileges
REVOKE privilege_type ON object_name FROM role_name;
```

#### Common Privileges (PostgreSQL)

| Privilege | Description |
|-----------|-------------|
| `SELECT` | Read data from table/view |
| `INSERT` | Add new rows |
| `UPDATE` | Modify existing data |
| `DELETE` | Remove rows |
| `TRUNCATE` | Empty table |
| `REFERENCES` | Create foreign keys |
| `TRIGGER` | Create triggers |
| `CREATE` | Create objects in schema |
| `CONNECT` | Connect to database |
| `TEMPORARY` | Create temporary tables |
| `EXECUTE` | Execute functions |
| `USAGE` | Use schema/sequences |

#### Examples

**Grant UPDATE privilege:**
```sql
GRANT UPDATE ON ratings TO data_analyst;
```

**Revoke UPDATE privilege:**
```sql
REVOKE UPDATE ON ratings FROM data_analyst;
```

---

### Users vs Groups (Both are Roles!)

**Important concept:** The role system encompasses both "users" and "groups"

#### User Roles
- Individual accounts
- Typically for one specific person
- Example: `alex`, `intern`

#### Group Roles
- Collections of privileges
- Multiple users can belong to a group
- Example: `data_analyst`, `admin`

#### Visual Representation
```
┌─────────────────────────┐
│   data_analyst (GROUP)  │
│                         │
│  - SELECT on tables     │
│  - UPDATE on ratings    │
└────────┬────────────────┘
         │
         ├──> alex (USER)
         │
         ├──> sarah (USER)
         │
         └──> john (USER)
```

---

### Adding Users to Groups

#### GRANT for Group Membership
```sql
GRANT group_role TO user_role;
```

**Example - Add Alex to data_analyst group:**
```sql
-- First, create the user
CREATE ROLE alex WITH LOGIN PASSWORD 'alex_password';

-- Then, add to group
GRANT data_analyst TO alex;
```

Now Alex has all privileges that `data_analyst` has!

#### REVOKE Group Membership
```sql
REVOKE group_role FROM user_role;
```

**Example:**
```sql
REVOKE data_analyst FROM alex;
```
Alex no longer has data_analyst privileges.

---

### Real-World Example

**Scenario:** Hiring data analysts and an intern

**Step 1: Create group role**
```sql
CREATE ROLE data_analyst;

-- Grant privileges to the group
GRANT SELECT ON ALL TABLES IN SCHEMA public TO data_analyst;
GRANT UPDATE ON ratings TO data_analyst;
GRANT INSERT ON user_feedback TO data_analyst;
```

**Step 2: Create user roles**
```sql
-- Create intern (temporary)
CREATE ROLE intern WITH LOGIN PASSWORD 'intern123' VALID UNTIL '2025-12-31';

-- Add intern to data_analyst group
GRANT data_analyst TO intern;

-- Create permanent analyst
CREATE ROLE sarah WITH LOGIN PASSWORD 'sarah_secure_pw';
GRANT data_analyst TO sarah;

-- Create admin
CREATE ROLE admin WITH LOGIN PASSWORD 'admin_pw' CREATEDB CREATEROLE;
```

**Step 3: When intern leaves**
```sql
REVOKE data_analyst FROM intern;
DROP ROLE intern;
```

**Step 4: When new analyst joins**
```sql
CREATE ROLE john WITH LOGIN PASSWORD 'john_pw';
GRANT data_analyst TO john;
```
*No need to reassign individual privileges!*

---

### Benefits of Roles

| Benefit | Description |
|---------|-------------|
| ✅ **Persistence** | Roles live on even when employees leave |
| ✅ **Efficiency** | Group common access levels, save time |
| ✅ **Pre-creation** | Create roles before employees get accounts |
| ✅ **Consistency** | Everyone in same role has same access |
| ✅ **Easy management** | Add/remove users from groups easily |

### ⚠️ Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Role gives too much access | Regularly audit role privileges |
| Forgotten to remove users | Document role memberships |
| Overlapping permissions | Use clear naming conventions |

---

## Part 2: Table Partitioning

### Why Partition?

**Problem:** When tables grow to hundreds of gigabytes or terabytes:
- Queries become slow
- Indices become too large to fit in memory
- Database performance degrades

**Solution:** Split tables into smaller parts = **Partitioning**

---

### Data Modeling Context

**Partitioning fits into the Physical Data Model:**
- **Logical:** Data structure remains the same
- **Physical:** Data is distributed over several physical entities

```
Conceptual Model (ER Diagram)
         ↓
Logical Model (Tables, Columns)
         ↓
Physical Model (Partitions, Storage) ← Partitioning happens here
```

---

### Types of Partitioning

## 1. Vertical Partitioning

**Definition:** Split table by columns (even when normalized)

### Example: Product Table

**Original Table (products):**
| product_id (PK) | product_name | price | long_description |
|-----------------|--------------|-------|------------------|
| 1               | Laptop       | 999   | [5000 chars...] |
| 2               | Mouse        | 25    | [4800 chars...] |
| 3               | Keyboard     | 75    | [4500 chars...] |

**After Vertical Partitioning:**

**products_main** (frequently accessed, fast storage)
| product_id (PK) | product_name | price |
|-----------------|--------------|-------|
| 1               | Laptop       | 999   |
| 2               | Mouse        | 25    |
| 3               | Keyboard     | 75    |

**products_details** (rarely accessed, slower storage)
| product_id (PK) | long_description |
|-----------------|------------------|
| 1               | [5000 chars...] |
| 2               | [4800 chars...] |
| 3               | [4500 chars...] |

**Benefits:**
- ✅ Faster queries on main table (less data to scan)
- ✅ Can store rarely-used data on slower, cheaper storage
- ✅ Reduces I/O for common queries

---

## 2. Horizontal Partitioning

**Definition:** Split table by rows (typically by date/timestamp)

### Example: Book Sales

**Original Table (book_sales):**
| sale_id | book_id | store_id | timestamp  | sales_amount | quantity |
|---------|---------|----------|------------|--------------|----------|
| 1       | 101     | 5        | 2018-01-15 | 250.00       | 10       |
| 2       | 205     | 3        | 2018-04-20 | 180.00       | 6        |
| 3       | 310     | 7        | 2018-07-10 | 125.00       | 5        |
| 4       | 101     | 5        | 2018-10-05 | 300.00       | 12       |
| 5       | 205     | 8        | 2019-01-20 | 200.00       | 8        |
| 6       | 310     | 3        | 2019-04-15 | 150.00       | 6        |

**After Horizontal Partitioning (by Quarter):**

**book_sales_2018_q1**
| sale_id | book_id | store_id | timestamp  | sales_amount | quantity |
|---------|---------|----------|------------|--------------|----------|
| 1       | 101     | 5        | 2018-01-15 | 250.00       | 10       |

**book_sales_2018_q2**
| sale_id | book_id | store_id | timestamp  | sales_amount | quantity |
|---------|---------|----------|------------|--------------|----------|
| 2       | 205     | 3        | 2018-04-20 | 180.00       | 6        |

**book_sales_2018_q3**
| sale_id | book_id | store_id | timestamp  | sales_amount | quantity |
|---------|---------|----------|------------|--------------|----------|
| 3       | 310     | 7        | 2018-07-10 | 125.00       | 5        |

**book_sales_2018_q4**
| sale_id | book_id | store_id | timestamp  | sales_amount | quantity |
|---------|---------|----------|------------|--------------|----------|
| 4       | 101     | 5        | 2018-10-05 | 300.00       | 12       |

---

### Creating Horizontal Partitions (PostgreSQL 10+)

#### Step 1: Create Parent Table
```sql
CREATE TABLE book_sales (
    sale_id INT,
    book_id INT,
    store_id INT,
    timestamp DATE,
    sales_amount DECIMAL,
    quantity INT
) PARTITION BY RANGE (timestamp);
```

#### Step 2: Create Partitions
```sql
-- Q1 2018 partition
CREATE TABLE book_sales_2018_q1 
PARTITION OF book_sales
FOR VALUES FROM ('2018-01-01') TO ('2018-04-01');

-- Q2 2018 partition
CREATE TABLE book_sales_2018_q2 
PARTITION OF book_sales
FOR VALUES FROM ('2018-04-01') TO ('2018-07-01');

-- Q3 2018 partition
CREATE TABLE book_sales_2018_q3 
PARTITION OF book_sales
FOR VALUES FROM ('2018-07-01') TO ('2018-10-01');

-- Q4 2018 partition
CREATE TABLE book_sales_2018_q4 
PARTITION OF book_sales
FOR VALUES FROM ('2018-10-01') TO ('2019-01-01');
```

#### Step 3: Create Index on Partition Column
```sql
CREATE INDEX ON book_sales (timestamp);
```

#### Querying Partitioned Tables
```sql
-- Query specific quarter (only scans one partition)
SELECT * FROM book_sales 
WHERE timestamp BETWEEN '2018-01-01' AND '2018-03-31';

-- Query entire table (scans all partitions)
SELECT * FROM book_sales;
```

---

### Pros and Cons of Horizontal Partitioning

#### ✅ Advantages

| Benefit | Description | Example |
|---------|-------------|---------|
| **Optimized indices** | Indices for each partition are smaller | Index for Q1 2018 fits in memory |
| **Faster queries** | Only relevant partitions are scanned | Query for Jan 2018 only scans Q1 partition |
| **Storage flexibility** | Move old data to slower storage | 2018 data on cheaper HDD, 2024 on SSD |
| **Easier maintenance** | Drop entire partitions | Drop all 2015 data at once |
| **Better for OLAP & OLTP** | Both benefit from partitioning | Analytics on recent data is faster |

#### ❌ Disadvantages

| Challenge | Description |
|-----------|-------------|
| **Migration complexity** | Must create new table and copy data |
| **Constraint limitations** | PRIMARY KEY constraints may not work |
| **Management overhead** | Must create and maintain multiple tables |
| **Query complexity** | Some queries need to scan all partitions |

---

### Sharding: Partitioning Across Machines

**Sharding = Horizontal partitioning across multiple servers**

**Example:**
```
Server 1: book_sales_2018_q1, book_sales_2018_q2
Server 2: book_sales_2018_q3, book_sales_2018_q4
Server 3: book_sales_2019_q1, book_sales_2019_q2
```

**Benefits:**
- Massively Parallel Processing (MPP)
- Each server processes its own data
- Scales horizontally

**Use cases:**
- Very large databases (petabytes)
- High-traffic applications
- Distributed systems

---

## Part 3: Data Integration

### What is Data Integration?

**Definition:** Combining data from different sources, formats, and technologies to provide a unified view.

```
Multiple Data Sources → Transformations → Unified Data Model
```

---

### Business Case Examples

#### 1. 360-Degree Customer View
**Scenario:** View all customer information from different departments

```
Sales DB (PostgreSQL)
    ↓
Marketing DB (MySQL)      →  Unified Customer View
    ↓
Support DB (MongoDB)
```

#### 2. Company Acquisition
**Challenge:** Merge databases from two companies

```
Company A Database  →
                      →  Combined Database
Company B Database  →
```

#### 3. Legacy Systems
**Scenario:** Insurance company with old and new claims systems

```
Legacy System (Old claims 1990-2010)
    ↓
Modern System (New claims 2010-2024)  →  Unified Claims Database
```

---

### Data Integration Planning

#### 1. Define Your Goal

**Unified data model purposes:**
- 📊 **Dashboards:** Daily sales graphs, KPI tracking
- 🤖 **Data Products:** Recommendation engines, ML models
- 📈 **Analytics:** Business intelligence, reporting

**Performance requirement:** Model must be fast enough for use case

---

#### 2. Identify Data Sources

**Example: DataCamp Skill Assessment Launch**

**Goal:** Target customers for new product

**Data needed:**
```
Sales Data (PostgreSQL)
    ↓
    └─> Which customers can afford new product?

Product Data (MongoDB)
    ↓
    └─> Who are potential early adopters?

         ↓
    
Unified Model (Redshift)
    ↓
Marketing Dashboard
```

---

#### 3. Determine Formats

**Data Source Formats:**
- PostgreSQL (relational)
- MongoDB (document store)
- CSV files
- APIs (JSON)
- Excel spreadsheets

**Unified Model Format:**
- Redshift (AWS data warehouse)
- Snowflake (cloud data platform)
- BigQuery (Google)
- Azure SQL Data Warehouse

---

#### 4. Update Cadence

**How often should data refresh?**

| Use Case | Update Frequency | Example |
|----------|-----------------|---------|
| **Sales reports** | Daily | `UPDATE DAILY at 2:00 AM` |
| **Air traffic control** | Real-time | `STREAM continuously` |
| **Customer analytics** | Hourly | `UPDATE HOURLY` |
| **Historical analysis** | Weekly/Monthly | `UPDATE WEEKLY on Sunday` |

⚠️ **Different sources can have different cadences!**

```
Sales DB → Daily updates
Product usage → Real-time  → Unified Model
Customer profiles → Weekly
```

---

### Transformations

**Problem:** Sources are in different formats - can't just plug them together!

**Solution:** Transformations extract and convert data to unified format

#### Example Transformation

**Source 1 (PostgreSQL):**
```
customers table:
| id | first_name | last_name | purchase_date |
```

**Source 2 (MongoDB):**
```
{
  "user_id": "123",
  "name": "John Doe",
  "bought_on": "2024-10-15"
}
```

**Unified Model:**
```
| customer_id | full_name | purchase_date |
```

**Transformation:**
```python
# PostgreSQL transformation
customer_id = id
full_name = first_name + " " + last_name
purchase_date = purchase_date

# MongoDB transformation
customer_id = user_id
full_name = name
purchase_date = bought_on
```

---

### Data Integration Tools

#### Hand-Coded Transformations
**Pros:**
- ✅ Full control
- ✅ Custom logic

**Cons:**
- ❌ Must create transformation for each source
- ❌ High maintenance burden
- ❌ Difficult to scale

#### Data Integration Tools (ETL)

**Popular Tools:**
- **Apache Airflow:** Workflow orchestration
- **Scriptella:** ETL framework
- **Talend:** Data integration platform
- **Apache NiFi:** Data flow automation
- **dbt:** Transform data in warehouse

**Example: Apache Airflow DAG**
```python
from airflow import DAG
from airflow.operators.python import PythonOperator

dag = DAG('data_integration', schedule_interval='@daily')

extract_sales = PythonOperator(
    task_id='extract_sales',
    python_callable=extract_sales_data
)

extract_product = PythonOperator(
    task_id='extract_product',
    python_callable=extract_product_data
)

transform = PythonOperator(
    task_id='transform',
    python_callable=transform_data
)

load = PythonOperator(
    task_id='load',
    python_callable=load_to_warehouse
)

[extract_sales, extract_product] >> transform >> load
```

---

### Choosing a Data Integration Tool

#### Key Criteria

| Criterion | Why Important | Example |
|-----------|---------------|---------|
| **Flexibility** | Connects to all your sources | PostgreSQL, MongoDB, APIs, CSV |
| **Reliability** | Maintainable long-term | Still supported in 5 years |
| **Scalability** | Handles growth | From GB to TB of data |
| **Monitoring** | Track data quality | Automated testing, alerts |
| **Security** | Protects sensitive data | Encryption, access control |

---

### Best Practices

#### 1. Automated Testing and Proactive Alerts

**Example checks:**
```sql
-- Check data counts match after transformation
SELECT COUNT(*) FROM source_sales;  -- Result: 10,000

SELECT COUNT(*) FROM unified_sales; -- Should be: 10,000

-- Check totals remain consistent
SELECT SUM(amount) FROM source_sales;  -- Result: $1,250,000

SELECT SUM(amount) FROM unified_sales; -- Should be: $1,250,000
```

**Alert if:**
- Row counts don't match
- Total amounts differ
- Required fields are NULL
- Data types are incorrect

---

#### 2. Security and Access Control

**Example: Credit Card Anonymization**

**Original Data (Restricted):**
| customer_id | credit_card_number | purchase_amount |
|-------------|-------------------|-----------------|
| 1           | 4532-1234-5678-9012 | 150.00 |
| 2           | 5411-8765-4321-0987 | 200.00 |

**Transformed Data (Business Analysts):**
| customer_id | card_type | purchase_amount |
|-------------|-----------|-----------------|
| 1           | Visa      | 150.00 |
| 2           | Mastercard| 200.00 |

**Transformation:**
```python
# Identify card type from first 4 digits
first_four = credit_card_number[:4]

if first_four.startswith('4'):
    card_type = 'Visa'
elif first_four.startswith('5'):
    card_type = 'Mastercard'

# Don't include full number in unified model
```

---

#### 3. Data Governance - Lineage

**Lineage = Tracking data's journey**

**Example:**
```
Source: sales_db.transactions
    ↓ (extracted 2024-10-29 02:00)
Staging: staging.raw_transactions
    ↓ (transformed 2024-10-29 02:15)
Unified: warehouse.sales
    ↓ (used by)
Dashboard: daily_sales_report
ML Model: customer_churn_prediction
```

**Why important:**
- ✅ Auditing compliance
- ✅ Debugging data issues
- ✅ Impact analysis (if source changes)
- ✅ Data quality tracking

---

## Part 4: Database Management Systems (DBMS)

### What is a DBMS?

**Definition:** System software for creating and maintaining databases

**DBMS manages three aspects:**
1. **Data:** The actual information stored
2. **Database Schema:** Logical structure
3. **Database Engine:** Allows data access, locking, modification

**Role:** Interface between database and users/applications

```
Users/Applications
        ↕
    [DBMS]
        ↕
   Database
```

---

### Two Main Types

## 1. SQL DBMS (Relational DBMS)

### Characteristics
- Based on relational data model
- Uses SQL for queries
- Structured data with predefined schema
- Tables with rows and columns
- ACID compliance (Atomicity, Consistency, Isolation, Durability)

### Popular SQL DBMS Options

| DBMS | Best For | Key Features |
|------|----------|--------------|
| **PostgreSQL** | General purpose, complex queries | Open source, extensible, JSON support |
| **MySQL** | Web applications | Fast reads, popular with WordPress |
| **Microsoft SQL Server** | Enterprise applications | Windows integration, .NET support |
| **Oracle Database** | Large enterprises | High performance, enterprise features |
| **SQLite** | Mobile apps, embedded systems | Lightweight, serverless |

### Example: PostgreSQL Table
```sql
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    hire_date DATE,
    salary DECIMAL(10, 2),
    department_id INT REFERENCES departments(id)
);

SELECT * FROM employees WHERE salary > 50000;
```

### ✅ Use SQL DBMS When:

| Scenario | Reason |
|----------|--------|
| **Fixed structure** | Schema doesn't change frequently |
| **Data consistency critical** | Banking, accounting, ERP systems |
| **Complex relationships** | Many foreign keys and joins |
| **ACID transactions needed** | Financial transactions, inventory |
| **Structured data** | Well-defined rows and columns |

---

## 2. NoSQL DBMS (Non-Relational DBMS)

### Characteristics
- Document-centered (not table-centered)
- No predefined schema required
- Flexible data structures
- Horizontal scaling
- Eventually consistent (often)

### 🔑 Key-Value Store

**Structure:** Stores key-value pairs (like a dictionary)

**Data model:**
```
Key          → Value
"user:1001"  → "{'name': 'Alice', 'age': 30}"
"session:abc"→ "{'user_id': 1001, 'cart': [...]}"
"cache:home" → "<!DOCTYPE html><html>..."
```

**Popular DBMS:** Redis, Amazon DynamoDB, Riak

**Example (Redis):**
```redis
SET user:1001 "{'name': 'Alice', 'age': 30}"
GET user:1001
DEL user:1001
```

**✅ Use for:**
- Session management (shopping carts)
- Caching (web pages, API responses)
- Real-time analytics
- Message queues

---

### 📄 Document Store

**Structure:** Keys with structured document values (JSON, XML, BSON)

**Data model:**
```json
{
  "_id": "post_001",
  "title": "Introduction to NoSQL",
  "author": "Alice Johnson",
  "content": "NoSQL databases are...",
  "tags": ["database", "nosql", "tutorial"],
  "comments": [
    {
      "user": "Bob",
      "text": "Great post!",
      "date": "2024-10-15"
    }
  ],
  "created_at": "2024-10-01"
}
```

**Popular DBMS:** MongoDB, CouchDB, Amazon DocumentDB

**Example (MongoDB):**
```javascript
// Insert document
db.posts.insertOne({
  title: "Introduction to NoSQL",
  author: "Alice Johnson",
  tags: ["database", "nosql"],
  created_at: new Date()
});

// Query documents
db.posts.find({ tags: "database" });

// Update document
db.posts.updateOne(
  { _id: "post_001" },
  { $set: { author: "Alice Smith" } }
);
```

**✅ Use for:**
- Content management systems (blogs, CMS)
- User profiles
- Product catalogs
- Mobile app backends

**Example Use Case: Blog Platform**
```
Each blog post = One document
{
  post_id, title, content, author, 
  comments[], tags[], metadata
}
```

---

### 🗂️ Columnar Database

**Structure:** Stores each column in separate files

**Traditional row-based:**
```
Row 1: [id=1, name=Alice, age=30, city=NYC]
Row 2: [id=2, name=Bob, age=25, city=LA]
Row 3: [id=3, name=Carol, age=35, city=NYC]
```

**Columnar:**
```
Column 'id': [1, 2, 3]
Column 'name': [Alice, Bob, Carol]
Column 'age': [30, 25, 35]
Column 'city': [NYC, LA, NYC]
```

**Popular DBMS:** Apache Cassandra, HBase, Amazon Redshift

**Example (Cassandra):**
```sql
CREATE TABLE user_events (
    user_id uuid,
    event_time timestamp,
    event_type text,
    value int,
    PRIMARY KEY (user_id, event_time)
);

-- Query billions of rows efficiently
SELECT event_type, SUM(value) 
FROM user_events 
WHERE event_time > '2024-01-01'
GROUP BY event_type;
```

**✅ Use for:**
- Big data analytics
- Data warehousing
- Time-series data
- When you query specific columns (not all columns)

**Why it's faster:**
```
Query: SELECT SUM(sales) FROM transactions;

Row-based: Reads ALL columns for EVERY row
Columnar: Reads ONLY the 'sales' column
```

---

### 🕸️ Graph Database

**Structure:** Nodes (entities) and edges (relationships)

**Data model:**
```
(Alice)-[:FRIENDS_WITH]->(Bob)
(Bob)-[:WORKS_AT]->(Company A)
(Alice)-[:LIKES]->(Product X)
(Bob)-[:BOUGHT]->(Product X)
(Product X)-[:CATEGORY]->(Electronics)
```

**Popular DBMS:** Neo4j, Amazon Neptune, ArangoDB

**Example (Neo4j - Cypher query language):**
```cypher
// Create nodes and relationships
CREATE (alice:Person {name: 'Alice', age: 30})
CREATE (bob:Person {name: 'Bob', age: 25})
CREATE (alice)-[:FRIENDS_WITH]->(bob)

// Find friends of friends
MATCH (person:Person {name: 'Alice'})-[:FRIENDS_WITH]->()-[:FRIENDS_WITH]->(fof)
RETURN fof.name

// Recommendation: Products liked by friends
MATCH (me:Person {name: 'Alice'})-[:FRIENDS_WITH]->(friend)-[:LIKES]->(product)
WHERE NOT (me)-[:LIKES]->(product)
RETURN product.name, COUNT(*) as friend_likes
ORDER BY friend_likes DESC
```

**✅ Use for:**
- Social networks (Facebook, LinkedIn)
- Recommendation engines (Netflix, Amazon)
- Fraud detection (transaction patterns)
- Knowledge graphs
- Network analysis

**Example Use Case: Social Network**
```
Nodes: Users, Posts, Groups
Edges: follows, likes, member_of, commented_on

Query: "Find all users who liked posts by people I follow"
```

---

## Choosing the Right DBMS

### Decision Tree

```
Do you have structured data with fixed schema?
    │
    ├─YES──> Do you need ACID transactions?
    │            │
    │            ├─YES──> Use SQL DBMS (PostgreSQL, MySQL)
    │            │
    │            └─NO───> Consider your query patterns
    │                        │
    │                        └──> Mostly analytical? → Columnar (Redshift)
    │
    └─NO──> What's your data structure?
                │
                ├─Simple key-value pairs? → Key-Value Store (Redis)
                │
                ├─Flexible documents/JSON? → Document Store (MongoDB)
                │
                ├─Highly connected data? → Graph Database (Neo4j)
                │
                └─Big data analytics? → Columnar (Cassandra)
```

---

### Comparison Table

| Requirement | SQL DBMS | NoSQL DBMS |
|-------------|----------|------------|
| **Fixed schema** | ✅ Required | ❌ Optional |
| **Scalability** | Vertical (bigger server) | Horizontal (more servers) |
| **Consistency** | Strong (ACID) | Eventual (BASE) |
| **Relationships** | Complex joins | Denormalized/embedded |
| **Query language** | SQL (standardized) | Varies by type |
| **Best for** | Structured, consistent data | Rapidly changing data |
| **Examples** | Banking, ERP, CRM | Social media, IoT, analytics |

---

### Real-World Examples

#### E-Commerce Platform

**SQL (PostgreSQL):**
- Orders, payments (need ACID)
- Customer accounts
- Inventory management

**NoSQL (MongoDB):**
- Product catalog (varying attributes)
- User reviews
- Shopping cart sessions

**NoSQL (Redis):**
- Session data
- Cache for popular products
- Real-time inventory counts

**NoSQL (Neo4j):**
- Product recommendations
- "Customers who bought this also bought..."

---

## Summary

### Database Roles
- **User roles:** Individual accounts
- **Group roles:** Collections of privileges
- Use GRANT/REVOKE for access control
- Roles are efficient and persistent

### Partitioning
- **Vertical:** Split by columns
- **Horizontal:** Split by rows (typically date)
- **Sharding:** Partition across servers
- Improves performance for large tables

### Data Integration
- Combines multiple sources into unified view
- Requires transformations (ETL)
- Consider: formats, update cadence, security
- Use tools like Airflow for orchestration

### DBMS Selection
- **SQL:** Structured data, ACID transactions, complex relationships
- **NoSQL:** Flexible schema, rapid growth, horizontal scaling
- Choose based on data structure and business needs
- Can use multiple DBMS types in same application (polyglot persistence)

---

## Quick Reference Cheat Sheet

### Role Management Commands

```sql
-- Create role
CREATE ROLE role_name [WITH options];

-- Modify role
ALTER ROLE role_name WITH attribute;

-- Delete role
DROP ROLE role_name;

-- Grant privileges
GRANT privilege ON object TO role;

-- Revoke privileges
REVOKE privilege ON object FROM role;

-- Add user to group
GRANT group_role TO user_role;

-- Remove user from group
REVOKE group_role FROM user_role;
```

---

### Partitioning Commands (PostgreSQL)

```sql
-- Create partitioned table
CREATE TABLE table_name (columns...)
PARTITION BY RANGE (column_name);

-- Create partition
CREATE TABLE partition_name
PARTITION OF table_name
FOR VALUES FROM (start) TO (end);

-- Create index
CREATE INDEX ON table_name (partition_column);
```

---

### DBMS Selection Quick Guide

| Your Situation | Recommended DBMS |
|----------------|------------------|
| Banking/financial system | PostgreSQL, Oracle |
| E-commerce product catalog | MongoDB |
| Real-time caching | Redis |
| Social network | Neo4j |
| Data warehouse/analytics | Redshift, Cassandra |
| Mobile app backend | MongoDB, Firebase |
| IoT sensor data | Cassandra, InfluxDB |
| Content management system | MongoDB, PostgreSQL |

---

## Practice Scenarios

### Scenario 1: New Data Team

**Situation:** You're setting up a data team with 5 analysts and 2 engineers.

**Solution:**
```sql
-- Create group roles
CREATE ROLE data_analyst;
CREATE ROLE data_engineer;

-- Grant appropriate privileges
GRANT SELECT ON ALL TABLES IN SCHEMA public TO data_analyst;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO data_engineer;

-- Create user roles
CREATE ROLE alice WITH LOGIN PASSWORD 'pw1';
CREATE ROLE bob WITH LOGIN PASSWORD 'pw2';
CREATE ROLE charlie WITH LOGIN PASSWORD 'pw3';

-- Assign to groups
GRANT data_analyst TO alice;
GRANT data_analyst TO bob;
GRANT data_engineer TO charlie;
```

---

### Scenario 2: Large Sales Table

**Situation:** Sales table with 500GB of data, queries are slow.

**Solution - Horizontal Partitioning:**
```sql
-- Create partitioned table
CREATE TABLE sales (
    sale_id BIGINT,
    sale_date DATE,
    amount DECIMAL,
    customer_id INT
) PARTITION BY RANGE (sale_date);

-- Create yearly partitions
CREATE TABLE sales_2022 PARTITION OF sales
FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

CREATE TABLE sales_2023 PARTITION OF sales
FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE sales_2024 PARTITION OF sales
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Queries now only scan relevant partitions
SELECT SUM(amount) FROM sales 
WHERE sale_date BETWEEN '2024-01-01' AND '2024-12-31';
-- Only scans sales_2024 partition!
```

---

### Scenario 3: Multi-Source Integration

**Situation:** Combine customer data from CRM (PostgreSQL) and website analytics (MongoDB).

**Planning:**

| Aspect | Decision |
|--------|----------|
| **Goal** | 360-degree customer view |
| **Sources** | CRM (PostgreSQL), Analytics (MongoDB) |
| **Target** | Data warehouse (Snowflake) |
| **Update** | Daily at 2 AM |
| **Tool** | Apache Airflow |

**Implementation:**
```python
from airflow import DAG
from datetime import datetime

dag = DAG(
    'customer_integration',
    schedule_interval='0 2 * * *',  # 2 AM daily
    start_date=datetime(2024, 1, 1)
)

# Define tasks
extract_crm = ExtractPostgreSQLTask(...)
extract_analytics = ExtractMongoDBTask(...)
transform_data = TransformTask(...)
load_warehouse = LoadSnowflakeTask(...)
validate_data = ValidateTask(...)

# Set dependencies
[extract_crm, extract_analytics] >> transform_data >> load_warehouse >> validate_data
```

---

### Scenario 4: Choosing DBMS for New Project

**Project:** Building a social media application

**Requirements Analysis:**

| Feature | Best DBMS | Reason |
|---------|-----------|--------|
| User profiles | MongoDB | Flexible schema (users have different fields) |
| Friend connections | Neo4j | Graph relationships |
| Session data | Redis | Fast key-value storage |
| Post history | PostgreSQL | Need transactions for consistency |
| Real-time feed | Cassandra | High write throughput |

**Architecture:**
```
User Registration → PostgreSQL (ACID for accounts)
    ↓
Profile Data → MongoDB (flexible attributes)
    ↓
Friendships → Neo4j (graph queries)
    ↓
Active Sessions → Redis (fast access)
    ↓
Post Feed → Cassandra (scalable writes)
```

**Polyglot Persistence:** Using multiple database types for different needs!

---

## Common Mistakes to Avoid

### ❌ Role Management

| Mistake | Why It's Bad | Solution |
|---------|--------------|----------|
| Not using groups | Repetitive privilege assignments | Create group roles for common access levels |
| Overly permissive roles | Security risk | Grant minimum necessary privileges |
| Forgetting to revoke | Former employees retain access | Regular access audits |
| No documentation | Can't track who has what access | Document all roles and their purposes |

---

### ❌ Partitioning

| Mistake | Why It's Bad | Solution |
|---------|--------------|----------|
| Partitioning small tables | Adds complexity without benefit | Only partition tables >100GB |
| Wrong partition key | Queries still scan all partitions | Choose key used in WHERE clauses |
| Too many partitions | Management overhead | Balance between size and quantity |
| Not maintaining | Old partitions accumulate | Regular cleanup/archival strategy |

---

### ❌ Data Integration

| Mistake | Why It's Bad | Solution |
|---------|--------------|----------|
| No data validation | Corrupt data propagates | Automated testing at each step |
| Ignoring security | Sensitive data exposed | Anonymize/encrypt during ETL |
| No monitoring | Failures go unnoticed | Proactive alerts and logging |
| Manual processes | Not scalable or reliable | Use ETL tools and automation |

---

### ❌ DBMS Selection

| Mistake | Why It's Bad | Solution |
|---------|--------------|----------|
| Using only SQL | Missing benefits of NoSQL | Evaluate needs objectively |
| Using only NoSQL | Losing ACID when needed | Use SQL for transactional data |
| Following trends | Tool doesn't fit needs | Choose based on requirements |
| Not considering scale | Can't handle growth | Plan for 10x data volume |

---

## Key Takeaways

### 🎯 Database Roles
1. Roles are the foundation of access control
2. Group roles save time and improve consistency
3. Use GRANT/REVOKE for fine-grained permissions
4. Audit roles regularly for security

### 🎯 Partitioning
1. Partition when tables exceed 100GB
2. Horizontal partitioning splits by rows (common: by date)
3. Vertical partitioning splits by columns (rare)
4. Sharding distributes partitions across servers
5. Both OLAP and OLTP benefit from partitioning

### 🎯 Data Integration
1. Plan: goal, sources, formats, update cadence
2. Transform data to unified format (ETL)
3. Use tools (Airflow, Talend) not manual code
4. Implement validation, monitoring, and security
5. Track data lineage for governance

### 🎯 DBMS Selection
1. SQL for structured data and ACID transactions
2. NoSQL for flexibility and horizontal scaling
3. Four NoSQL types: Key-Value, Document, Columnar, Graph
4. Choose based on data structure and use case
5. Polyglot persistence: use multiple DBMS types together

---

## Final Exam Practice Questions

### Question 1: Role Management
**Scenario:** You need to create a role for junior analysts who should only read from the `sales` and `customers` tables, but not modify them.

**Your task:** Write the SQL commands.

<details>
<summary>Click to see answer</summary>

```sql
-- Create the role
CREATE ROLE junior_analyst;

-- Grant SELECT privilege
GRANT SELECT ON sales TO junior_analyst;
GRANT SELECT ON customers TO junior_analyst;

-- Create a user and assign to group
CREATE ROLE jane WITH LOGIN PASSWORD 'secure_pw';
GRANT junior_analyst TO jane;
```
</details>

---

### Question 2: Partitioning Decision
**Scenario:** You have a 500GB `orders` table. Queries mostly filter by `order_date`. Should you partition? If yes, how?

<details>
<summary>Click to see answer</summary>

**Yes, partition by date:**
- Table is large (>100GB)
- Queries filter by `order_date`
- Partition by month or quarter

```sql
CREATE TABLE orders (
    order_id BIGINT,
    order_date DATE,
    customer_id INT,
    total DECIMAL
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024_q1 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
-- Create additional partitions for other quarters
```
</details>

---

### Question 3: DBMS Selection
**Scenario:** You're building:
1. A banking system (transactions, accounts)
2. A product recommendation engine
3. A real-time chat application cache

**Which DBMS for each?**

<details>
<summary>Click to see answer</summary>

1. **Banking:** PostgreSQL or Oracle (SQL)
   - Need ACID transactions
   - Structured data
   - Data consistency critical

2. **Recommendation engine:** Neo4j (Graph)
   - Relationships between users and products
   - "Users who liked X also liked Y"
   - Graph queries efficient

3. **Chat cache:** Redis (Key-Value)
   - Real-time access
   - Simple key-value pairs
   - Fast read/write
</details>

---

### Question 4: Data Integration
**Scenario:** Integrating customer data from:
- Sales DB (PostgreSQL) - daily updates needed
- Support tickets (MongoDB) - hourly updates needed
- Credit card data (must anonymize last 12 digits)

**What's your approach?**

<details>
<summary>Click to see answer</summary>

**Approach:**
1. **Different update schedules:**
   - Sales: Extract daily at 2 AM
   - Support: Extract hourly

2. **Security transformation:**
```python
def anonymize_credit_card(card_number):
    # Keep first 4 digits, mask rest
    return card_number[:4] + '-XXXX-XXXX-XXXX'
```

3. **ETL Pipeline (Airflow):**
```python
# Daily sales DAG
extract_sales >> transform_sales >> load_warehouse

# Hourly support DAG  
extract_support >> transform_support >> load_warehouse

# Add validation
load_warehouse >> validate_totals >> alert_on_failure
```

4. **Validation:**
   - Check record counts match
   - Verify no credit card numbers in output
   - Alert on failures
</details>

---

## Conclusion

Database management encompasses many critical areas:

- **Access Control:** Use roles to manage permissions efficiently
- **Performance:** Partition large tables for better query performance  
- **Integration:** Combine disparate sources into unified views
- **Technology:** Choose the right DBMS for your data and use case

The key is understanding your requirements and applying the right technique or technology to solve your specific problem. There's rarely a one-size-fits-all solution in database management!
