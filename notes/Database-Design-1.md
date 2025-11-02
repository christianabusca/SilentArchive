## Core Question: How Should We Organize and Manage Data?

Key considerations:
- **Schemas:** How should data be logically organized?
- **Normalization:** Should data have minimal dependency and redundancy?
- **Views:** What joins will be done most often?
- **Access control:** Should all users have the same level of access?
- **DBMS:** How to choose between SQL and NoSQL options?

**Answer:** It depends on the intended use of the data.

---

## Approaches to Processing Data

### OLTP vs OLAP Overview

| Aspect | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
|--------|--------------------------------------|-------------------------------------|
| **Purpose** | Support daily transactions | Report and analyze data |
| **Design** | Application-oriented | Subject-oriented |
| **Data** | Up-to-date, operational | Consolidated, historical |
| **Size** | Snapshot, gigabytes | Archive, terabytes |
| **Queries** | Simple transactions & frequent updates | Complex, aggregate queries & limited updates |
| **Users** | Thousands | Hundreds |

### OLTP Tasks (Examples)
- Find the price of a book
- Update latest customer transaction
- Keep track of employee hours

### OLAP Tasks (Examples)
- Calculate books with best profit margin
- Find most loyal customers
- Decide employee of the month

### How They Work Together
OLTP and OLAP systems complement each other - OLTP handles day-to-day operations while OLAP enables strategic analysis of historical data.

---

## Data Structures

### 1. Structured Data
- Follows a schema
- Defined data types & relationships
- **Examples:** SQL, tables in a relational database

### 2. Unstructured Data
- Schemaless
- Makes up most of data in the world
- **Examples:** Photos, chat logs, MP3 files

### 3. Semi-Structured Data
- Does not follow larger schema
- Self-describing structure
- **Examples:** NoSQL, XML, JSON

**JSON Example:**
```json
"user": { 
  "profile_use_background_image": true,  
  "statuses_count": 31,  
  "profile_background_color": "C0DEED",  
  "followers_count": 3066
}
```

---

## Data Storage Solutions

### Traditional Databases
- For storing real-time relational structured data
- **Use case:** OLTP

### Data Warehouses
- **Purpose:** For analyzing archived structured data (OLAP)
- Optimized for analytics
- Organized for reading/aggregating data
- Usually read-only
- Contains data from multiple sources
- Massively Parallel Processing (MPP)
- Typically uses denormalized schema and dimensional modeling

**Data Marts:** Subset of data warehouses dedicated to a specific topic

### Data Lakes
- Store all types of data at lower cost (raw, operational databases, IoT logs, real-time, relational and non-relational)
- Retains all data and can take up petabytes
- **Schema-on-read** (vs. schema-on-write)
- Need to catalog data to avoid becoming a "data swamp"
- Run big data analytics using Apache Spark and Hadoop
- Useful for deep learning and data discovery

### Data Integration Approaches
- **ETL:** Extract, Transform, Load
- **ELT:** Extract, Load, Transform

---

## Database Design Fundamentals

### What is Database Design?
Determines how data is logically stored based on:
- How data will be read and updated
- Database models (high-level specifications)
- Schemas (blueprint defining tables, fields, relationships, indexes, views)

**Most popular model:** Relational model

**Other options:** NoSQL models, object-oriented model, network model

---

## Data Modeling Process

### 1. Conceptual Data Model
- Describes entities, relationships, and attributes
- **Tools:** Entity-Relational (ER) diagrams, UML diagrams

### 2. Logical Data Model
- Defines tables, columns, relationships
- **Tools:** Database models and schemas (e.g., relational model, star schema)

### 3. Physical Data Model
- Describes physical storage
- **Tools:** Partitions, CPUs, indexes, backup systems, tablespaces

### From Conceptual to Logical
**Fastest conversion:** Entities become tables in the logical schema

---

## Dimensional Modeling

### Overview
- Adaptation of the relational model for data warehouse design
- Optimized for OLAP queries (aggregate data, not updating)
- Built using the **star schema**
- Easy to interpret and extend

### Key Elements

**Organization criteria:**
- What is being analyzed?
- How often do entities change?

**Fact Tables:**
- Decided by business use-case
- Holds records of metrics
- Changes regularly
- Connects to dimensions via foreign keys

**Dimension Tables:**
- Holds descriptions of attributes
- Does not change as often

---

## Key Takeaways

1. Always start by understanding business requirements
2. Understand the difference between OLAP and OLTP
3. Choose the appropriate approach: OLAP, OLTP, or something else
4. Select storage solutions based on data type and use case
5. Follow proper data modeling progression: Conceptual → Logical → Physical
