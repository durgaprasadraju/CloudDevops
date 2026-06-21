# MASTER AWS DynamoDB Guide

> **Complete learning guide: Beginner to Advanced · SQL-to-NoSQL bridge · Partitioning deep dive · Exam & interview ready**

A comprehensive reference for **AWS Developer Associate (DVA-C02)**, **Solutions Architect Associate (SAA-C03)**, **Solutions Architect Professional**, **DevOps Engineers**, and **Backend Developers** who know SQL but are new to NoSQL.

**Related in this repo:** [AWS Lambda Guide](../lambda/README.md) · [Lambda Master Guide](../../AWS%20Lambda,%20Python(Boto3)%20%26%20Serverless-%20Beginner%20to%20Advanced/MASTER_LAMBDA_GUIDE.md)

---

## Table of Contents

1. [What is DynamoDB?](#section-1-what-is-dynamodb)
2. [SQL vs NoSQL](#section-2-sql-vs-nosql)
3. [DynamoDB Core Concepts](#section-3-dynamodb-core-concepts)
4. [Partitioning and Sharding](#section-4-partitioning-and-sharding)
5. [Horizontal Scaling](#section-5-horizontal-scaling)
6. [DynamoDB Storage Architecture](#section-6-dynamodb-storage-architecture)
7. [AWS Console Walkthrough](#section-7-aws-console-walkthrough)
8. [Creating Data in Console UI](#section-8-creating-data-in-console-ui)
9. [Reading Data](#section-9-reading-data)
10. [Secondary Indexes](#section-10-secondary-indexes)
11. [Capacity Modes](#section-11-capacity-modes)
12. [Advanced DynamoDB Features](#section-12-advanced-dynamodb-features)
13. [Data Modeling](#section-13-data-modeling)
14. [Real World Architecture](#section-14-real-world-architecture)
15. [Performance Optimization](#section-15-performance-optimization)
16. [Security](#section-16-security)
17. [DynamoDB Interview Questions](#section-17-dynamodb-interview-questions)
18. [System Design Interview Perspective](#section-18-system-design-interview-perspective)
19. [Common Mistakes](#section-19-common-mistakes)
20. [Cheat Sheet](#section-20-cheat-sheet)

---

# Section 1: What is DynamoDB?

## 1.1 Definition

**Amazon DynamoDB** is a fully managed, serverless, key-value and document NoSQL database service provided by AWS. It is designed to deliver single-digit millisecond performance at any scale — from a handful of requests per day to millions of requests per second — without you managing servers, clusters, or replication.

Think of DynamoDB as a **giant distributed hash map** that AWS operates for you. You give it a key (partition key, optionally with a sort key), and it returns or stores a JSON-like document (an **item**) in milliseconds.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Amazon DynamoDB (Managed)                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ Table A  │  │ Table B  │  │ Table C  │  │  ...     │        │
│  │ (Items)  │  │ (Items)  │  │ (Items)  │  │          │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
│         Auto-scaling · Replication · Encryption · Backups        │
└─────────────────────────────────────────────────────────────────┘
```

**Key properties:**

| Property | Description |
|----------|-------------|
| Type | NoSQL (key-value + document) |
| Model | Schema-less per item (flexible attributes) |
| Scaling | Horizontal, automatic |
| Consistency | Eventually consistent (default) or strongly consistent (optional) |
| Availability | Multi-AZ by default, 99.99% SLA |
| Billing | On-demand or provisioned capacity |

---

## 1.2 Why AWS Created DynamoDB

Before DynamoDB (launched 2012), Amazon ran massive e-commerce workloads on relational databases and custom distributed systems. Two forces drove DynamoDB's creation:

1. **Amazon's own scale** — During peak events (Prime Day, Black Friday), traditional databases could not scale fast enough without expensive over-provisioning.
2. **Lessons from Amazon Dynamo** — A 2007 research paper described a highly available key-value store used internally at Amazon. DynamoDB is the managed, cloud-native evolution of those ideas.

AWS wanted a database where:

- Developers never patch OS or tune replication
- Capacity scales in seconds, not weeks
- You pay for what you use
- Latency stays low regardless of table size (petabytes)

```
Timeline:

2004  → Amazon internal Dynamo (paper published 2007)
2012  → DynamoDB launched as managed service
2017  → On-demand capacity, global tables GA
2018  → Transactions, on-demand backup
2019  → DAX integration improvements, PartiQL
2023+ → Multi-Region strong consistency options evolve
```

---

## 1.3 Problems with Traditional Relational Databases

If you know MySQL or PostgreSQL, you understand tables, rows, joins, and ACID transactions. At web scale, relational databases hit walls:

### Problem 1: Vertical Scaling Limits

```
                    ┌─────────────────┐
                    │   MySQL Server  │
                    │   CPU: 64 cores │
                    │   RAM: 512 GB   │  ← You can only go so big
                    │   Disk: 10 TB   │
                    └─────────────────┘
                              │
                    Eventually hits ceiling
```

You buy a bigger machine. Eventually the biggest machine is not big enough.

### Problem 2: Joins Don't Scale Horizontally

```sql
SELECT o.*, c.name, p.title
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
WHERE o.customer_id = 101;
```

On one server, this is fine. Split `orders`, `customers`, and `products` across 50 servers — **joins become nightmares**. You need application-level joins, denormalization, or caching layers.

### Problem 3: Schema Rigidity

Adding a column to a 2-billion-row table can lock the table or take hours. Agile product teams need schema flexibility.

### Problem 4: Replication and Failover Complexity

You must configure:
- Primary/replica topology
- Failover scripts
- Connection pooling during failover
- Split-brain scenarios

### Problem 5: Spiky Traffic

```
Traffic
   │
   │     ╱╲
   │    ╱  ╲        ← Flash sale, viral post
   │───╱    ╲──────
   │          ╲___
   └────────────────── Time

Relational DB: Must provision for PEAK → expensive idle capacity
```

---

## 1.4 Distributed Database Challenges

When data spans many machines, new problems appear:

| Challenge | Description |
|-----------|-------------|
| **Partitioning** | Which server holds which data? |
| **Replication** | How do copies stay in sync? |
| **Consistency** | If two nodes disagree, who wins? |
| **Failure** | Nodes crash; network partitions happen |
| **Coordination** | Distributed transactions are slow and fragile |
| **Hot spots** | One key gets 90% of traffic |

```
         Client Request: GET user_id=42
                    │
                    ▼
            ┌───────────────┐
            │  Router /     │
            │  Coordinator  │
            └───────┬───────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Node A  │ │ Node B  │ │ Node C  │
   │ keys    │ │ keys    │ │ keys    │
   │ 1-1000  │ │1001-2000│ │2001-3000│
   └─────────┘ └─────────┘ └─────────┘

Question: Which node has user_id=42?
Answer:   Hash(user_id) → Node A
```

DynamoDB solves these internally so you focus on access patterns, not cluster operations.

---

## 1.5 CAP Theorem

The **CAP theorem** (Brewer) states that a distributed system can guarantee at most **two of three** properties simultaneously:

```
                    ┌─────────────┐
                    │ Consistency │
                    │ (C)         │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │                         │
    ┌─────────▼─────────┐   ┌─────────▼─────────┐
    │ Availability (A)  │   │ Partition         │
    │                     │   │ Tolerance (P)     │
    └─────────────────────┘   └───────────────────┘

Pick 2 (in practice, during a network partition):
  CP → Consistent but may be unavailable
  AP → Available but may return stale data
```

| System | Typical CAP bias | Notes |
|--------|------------------|-------|
| Traditional RDBMS (single node) | CA | Not distributed |
| DynamoDB | **AP** (with tunable consistency) | Highly available; eventual by default |
| MongoDB (replica set) | Configurable | Can tune read concern |
| Cassandra | AP | Tunable consistency per query |
| PostgreSQL (single) | CA | Cluster solutions add tradeoffs |

**DynamoDB's approach:**

- **Default reads:** Eventually consistent (AP-friendly)
- **Optional:** `ConsistentRead=true` for strongly consistent reads on the same partition key (trades some availability/latency for freshness)
- **Writes:** Always go to the leader partition; replicated synchronously within the region

```
Eventually Consistent Read:
  Write → Partition Leader → Replicas (async propagation)
  Read  → May hit replica BEFORE write replicated → stale possible

Strongly Consistent Read:
  Write → Partition Leader → Waits for quorum of replicas
  Read  → Reads from leader or confirmed replicas → always latest
```

---

## 1.6 Why NoSQL Databases Emerged

NoSQL ("Not Only SQL") databases trade some relational features for **scale, speed, and flexibility**:

```
Relational (SQL)                    NoSQL
─────────────────                   ─────────────────
Fixed schema                        Flexible schema
Normalize data                      Denormalize for access patterns
JOIN at query time                  JOIN at application time (or pre-join)
Scale up (bigger server)            Scale out (more servers)
ACID across tables (complex)        Per-item atomicity (simpler at scale)
```

**When NoSQL fits:**

- Massive read/write throughput
- Known access patterns upfront
- Horizontal scaling required
- Schema evolves frequently
- Geographic distribution needed

**When SQL still fits:**

- Complex ad-hoc analytics
- Heavy multi-table transactions
- Strict relational integrity
- Mature reporting ecosystem

---

## 1.7 Real-World Companies Using DynamoDB

| Company | Use Case |
|---------|----------|
| **Amazon** | Shopping cart, product catalog, session state |
| **Netflix** | User viewing history, personalization metadata |
| **Snap** | Messaging metadata, user state |
| **Lyft** | Real-time location and ride matching data |
| **Samsung** | IoT device state |
| **Capital One** | Customer-facing mobile app backends |
| **Airbnb** | Session and metadata stores (alongside other DBs) |
| **Duolingo** | User progress and gamification state |

These companies chose DynamoDB for **predictable low latency**, **serverless operations**, and **elastic scale** during traffic spikes.

---

## 1.8 DynamoDB Architecture Overview

A typical serverless application using DynamoDB:

```
┌──────────┐
│   User   │  (Browser / Mobile App)
└────┬─────┘
     │ HTTPS
     ▼
┌──────────────────┐
│   API Gateway    │  REST or HTTP API
│   - Auth         │  Rate limiting, API keys, Cognito authorizer
│   - Routing      │  Routes /orders → Lambda
└────┬─────────────┘
     │ Invoke
     ▼
┌──────────────────┐
│     Lambda       │  Business logic (Python, Node.js, etc.)
│   - Validation   │  Boto3 SDK calls to DynamoDB
│   - Transform    │  No persistent server — scales automatically
└────┬─────────────┘
     │ AWS SDK (HTTPS)
     ▼
┌──────────────────┐
│    DynamoDB      │  Tables, items, indexes
│   - Partitions   │  Auto-scaled storage and throughput
│   - Replicas     │  Multi-AZ durability
└──────────────────┘
```

### Component Explanations

| Component | Role |
|-----------|------|
| **User** | End consumer sending HTTP requests |
| **API Gateway** | Front door; handles TLS, authentication, throttling, request/response mapping |
| **Lambda** | Stateless compute; runs your code per request; uses IAM role to call DynamoDB |
| **DynamoDB** | Persistent data store; no connection pools to manage; HTTP-based API |

**Alternative integrations:**

```
DynamoDB ← App Runner / ECS / EC2 (direct SDK)
DynamoDB ← Step Functions (orchestration)
DynamoDB → DynamoDB Streams → Lambda (event-driven)
DynamoDB ← Amplify (GraphQL / DataStore)
```

---

## 1.9 DynamoDB vs Other AWS Databases

Choosing the right AWS database prevents costly re-architecture later.

| Service | Type | Best For |
|---------|------|----------|
| **DynamoDB** | NoSQL key-value/document | Serverless apps, session stores, gaming, IoT metadata |
| **RDS (MySQL/PG)** | Relational SQL | Traditional apps, complex queries, reporting |
| **Aurora** | Relational SQL (cloud-native) | High-performance SQL, MySQL/PostgreSQL compatibility |
| **DocumentDB** | MongoDB-compatible | Document workloads needing MongoDB API |
| **ElastiCache** | In-memory cache | Sub-ms caching (pairs with DynamoDB) |
| **Redshift** | Data warehouse | Analytics, BI, petabyte-scale SQL |
| **Timestream** | Time-series | IoT metrics, observability data |
| **Neptune** | Graph | Social networks, fraud detection graphs |

```
Decision shortcut:

Need SQL JOINs + complex reports?     → RDS / Aurora
Need key-based access at huge scale?  → DynamoDB
Need full-text search?                → OpenSearch (+ DynamoDB as source)
Need graph traversals?                → Neptune
Need analytics on historical data?    → Redshift / Athena + S3
```

**Common pattern:** DynamoDB as operational store + Streams → OpenSearch for search + S3/Redshift for analytics. Each database does what it does best.

**Cost mindset:** A common mistake is comparing DynamoDB on-demand costs to a small RDS instance. The fair comparison is DynamoDB versus **fully managed, Multi-AZ, auto-scaled relational infrastructure** plus the engineering time to operate it at scale. For spiky or unpredictable workloads, DynamoDB often wins on total cost of ownership even when per-request pricing looks higher on paper.

---

# Section 2: SQL vs NoSQL

## 2.1 Comparison Table

| Dimension | SQL (MySQL/PostgreSQL) | NoSQL (DynamoDB) |
|-----------|------------------------|------------------|
| **Schema** | Fixed schema; ALTER TABLE to change | Flexible per item; no ALTER needed |
| **Scaling** | Mostly vertical; read replicas for reads | Horizontal; partitions auto-distribute |
| **Performance** | Depends on indexes, query planner, joins | Predictable for key-based access; scans are slow |
| **Joins** | Native SQL JOINs | No joins; denormalize or query multiple times |
| **Transactions** | Full ACID multi-table | Single-item atomic; multi-item transactions (limited, same region) |
| **Availability** | Depends on setup (RDS Multi-AZ ~99.95%) | 99.99% SLA; Multi-AZ built-in |
| **Cost** | Pay for instance 24/7 | Pay per request + storage; can be cheaper at variable load |

---

## 2.2 Database Profiles

### MySQL

- Open-source relational database
- Great for web apps, CMS, traditional CRUD
- Vertical scaling primary; replication for reads
- **Example fit:** Blog with posts, comments, users — relational integrity matters

### PostgreSQL

- Advanced open-source RDBMS
- JSON support, extensions (PostGIS), powerful SQL
- **Example fit:** Financial reporting, geospatial apps, complex queries

### Oracle

- Enterprise commercial RDBMS
- Advanced partitioning, RAC clustering
- **Example fit:** Large enterprise ERP, legacy migrations

### MongoDB

- Document NoSQL (BSON documents)
- Flexible schema, rich query language, aggregation pipeline
- Sharding for horizontal scale (you manage shard keys)
- **Example fit:** Content management, catalogs with varying fields

### Cassandra

- Wide-column store, AP-oriented
- Linear scale-out; tunable consistency
- **Example fit:** Time-series, messaging, high-write IoT

### DynamoDB

- Managed key-value/document on AWS
- Zero ops sharding; integrated with AWS ecosystem
- **Example fit:** Serverless apps, gaming leaderboards, session stores, IoT at scale on AWS

```
Database Selection Flowchart:

Need complex SQL analytics? ──YES──→ PostgreSQL / Redshift
         │
         NO
         ▼
On AWS, serverless, key-based access? ──YES──→ DynamoDB
         │
         NO
         ▼
Flexible documents, self-managed? ──YES──→ MongoDB
         │
         NO
         ▼
Massive write throughput, multi-DC? ──YES──→ Cassandra
```

---

## 2.3 SQL Table vs DynamoDB Item

### SQL: Customers Table

```
┌─────────────────────────────────────────────────────┐
│                    customers                         │
├────┬──────────┬─────────────────────┬─────────────┤
│ id │ name     │ email               │ created_at  │
├────┼──────────┼─────────────────────┼─────────────┤
│101 │ John     │ john@gmail.com      │ 2024-01-15  │
│102 │ Jane     │ jane@yahoo.com      │ 2024-02-20  │
│103 │ Bob      │ bob@outlook.com     │ 2024-03-01  │
└────┴──────────┴─────────────────────┴─────────────┘

Primary Key: id (auto-increment or UUID)
Every row has SAME columns (schema enforced)
```

```sql
CREATE TABLE customers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO customers (name, email) VALUES ('John', 'john@gmail.com');
SELECT * FROM customers WHERE id = 101;
```

### DynamoDB: Customers Table

```
Table Name: Customers
Partition Key: CustomerID (String)

┌─────────────────────────────────────────────────────┐
│ Item 1 (JSON-like document)                          │
│ {                                                    │
│   "CustomerID": "101",        ← Partition Key      │
│   "Name": "John",                                    │
│   "Email": "john@gmail.com"                          │
│ }                                                    │
├─────────────────────────────────────────────────────┤
│ Item 2                                               │
│ {                                                    │
│   "CustomerID": "102",                               │
│   "Name": "Jane",                                    │
│   "Email": "jane@yahoo.com",                         │
│   "LoyaltyTier": "Gold"       ← Extra attribute OK  │
│ }                                                    │
├─────────────────────────────────────────────────────┤
│ Item 3                                               │
│ {                                                    │
│   "CustomerID": "103",                               │
│   "Name": "Bob",                                     │
│   "Email": "bob@outlook.com",                        │
│   "Preferences": {            ← Nested Map OK       │
│     "Newsletter": true,                              │
│     "Theme": "dark"                                  │
│   }                                                  │
│ }                                                    │
└─────────────────────────────────────────────────────┘

No fixed schema — items can have different attributes
```

**Critical difference:** In SQL you `SELECT WHERE id=101`. In DynamoDB you `GetItem` with the **exact primary key** — the partition key (and sort key if composite).

### Extended SQL vs DynamoDB Query Examples

**SQL — flexible WHERE clauses:**

```sql
-- Any column can filter (with index support)
SELECT * FROM customers WHERE email = 'john@gmail.com';
SELECT * FROM customers WHERE created_at > '2024-01-01';
SELECT * FROM customers WHERE name LIKE 'J%';

-- Join across tables
SELECT c.name, o.order_id, o.total
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.id = 101;
```

**DynamoDB — key-based access only (without Scan):**

```python
# GetItem — must have full primary key
table.get_item(Key={'CustomerID': '101'})

# Query — partition key required; optional sort key condition
table.query(
    KeyConditionExpression=Key('UserID').eq('101') &
                           Key('OrderDate').between('2024-01-01', '2024-12-31')
)

# GSI Query — lookup by email (requires EmailIndex GSI)
table.query(
    IndexName='EmailIndex',
    KeyConditionExpression=Key('Email').eq('john@gmail.com')
)

# Scan — NO key required but reads ENTIRE table (avoid)
table.scan(FilterExpression=Attr('Name').begins_with('J'))
```

```
SQL Query Flexibility:
┌────────────────────────────────────────────────────────┐
│  SELECT ... WHERE <any column>                         │
│  Optimizer picks index or full table scan              │
└────────────────────────────────────────────────────────┘

DynamoDB Query Model:
┌────────────────────────────────────────────────────────┐
│  Must know PARTITION KEY upfront                       │
│  Optional SORT KEY range                               │
│  OR use GSI with different key                       │
│  FilterExpression = post-filter (still reads all keys  │
│  in key condition first — costs RCU for read items)    │
└────────────────────────────────────────────────────────┘
```

### When SQL Wins vs When DynamoDB Wins

| Scenario | Winner | Why |
|----------|--------|-----|
| Report: revenue by region, last 12 months | PostgreSQL | Aggregations, GROUP BY, ad-hoc |
| Shopping cart for 50M users | DynamoDB | Key-based, elastic scale |
| Bank ledger with complex constraints | PostgreSQL | Multi-table ACID, constraints |
| Gaming session state, 1ms reads | DynamoDB | Predictable key access, DAX |
| Data warehouse analytics | Redshift/Snowflake | Columnar, SQL |
| Mobile app user profile | DynamoDB | Serverless, low ops |

---

# Section 3: DynamoDB Core Concepts

## 3.1 Tables

A **table** is a collection of items — analogous to an SQL table but without enforced column definitions.

```
Account
  └── Region (e.g., us-east-1)
        └── Table: Customers
        └── Table: Orders
        └── Table: Products
```

- Table names: 3–255 characters, alphanumeric, `-`, `_`, `.`
- No limit on number of tables per account (soft limits apply)
- Tables are **regional** resources (except Global Tables — multi-region)

---

## 3.2 Items

An **item** is a single record — like one row in SQL, but a flexible document.

- Maximum item size: **400 KB** (including attribute names and values)
- Items are uniquely identified by their **primary key**
- No two items in a table can have the same primary key

---

## 3.3 Attributes

**Attributes** are name-value pairs within an item — like columns, but optional and variable per item.

| Attribute Name | Value | DynamoDB Type |
|----------------|-------|---------------|
| CustomerID | "101" | S (String) |
| Age | 30 | N (Number) |
| Active | true | BOOL |
| Tags | ["vip", "premium"] | L (List) |
| Address | {"City": "NYC"} | M (Map) |

**Supported types:** String (S), Number (N), Binary (B), Boolean (BOOL), Null (NULL), String Set (SS), Number Set (NS), Binary Set (BS), List (L), Map (M).

---

## 3.4 Primary Key

The **primary key** uniquely identifies each item. Two types:

### Type A: Partition Key Only (Simple Primary Key)

```
Primary Key = Partition Key

┌─────────────────────────────────────┐
│ CustomerID (Partition Key)          │
├─────────────────────────────────────┤
│ "101"  →  {Name: John, ...}         │
│ "102"  →  {Name: Jane, ...}         │
│ "103"  →  {Name: Bob, ...}          │
└─────────────────────────────────────┘

One item per CustomerID value
```

### Type B: Partition Key + Sort Key (Composite Primary Key)

```
Primary Key = (Partition Key, Sort Key)

┌──────────────────────────────────────────────────────┐
│ UserID (PK)  │ OrderDate (SK)  │  Other attrs       │
├──────────────┼─────────────────┼────────────────────┤
│ user-1       │ 2024-01-01      │  {total: 50}       │
│ user-1       │ 2024-02-15      │  {total: 120}      │  ← Same PK, different SK
│ user-2       │ 2024-01-01      │  {total: 75}       │
└──────────────┴─────────────────┴────────────────────┘

Many items can share partition key; sort key differentiates them
```

---

## 3.5 Partition Key (HASH Key)

The **partition key** determines **which physical partition** stores the item.

```
hash("101") mod num_partitions → Partition 7
hash("102") mod num_partitions → Partition 3
hash("103") mod num_partitions → Partition 7  ← Collision possible (different keys, same partition)
```

**Design rule:** Choose a partition key with **high cardinality** (many unique values) to spread load evenly.

**Bad:** `Status` with values `active`/`inactive` → only 2 partitions get traffic  
**Good:** `CustomerID` UUID → millions of unique values → even distribution

---

## 3.6 Sort Key (RANGE Key)

The **sort key** enables:

1. **Multiple items per partition key** (one-to-many)
2. **Range queries** within a partition

```
Query: UserID = "user-1" AND OrderDate BETWEEN "2024-01-01" AND "2024-03-01"

Partition "user-1":
  ├── 2024-01-01  ✓
  ├── 2024-02-15  ✓
  └── 2024-04-01  ✗ (outside range)
```

Sort keys are stored **sorted** within each partition — efficient range scans on that partition only.

---

## 3.7 Customer Table Example

**Table:** `Customers`  
**Partition Key:** `CustomerID` (String)

```json
{
  "CustomerID": "101",
  "Name": "John",
  "Email": "john@gmail.com",
  "CreatedAt": "2024-06-01T10:00:00Z"
}
```

```json
{
  "CustomerID": "102",
  "Name": "Jane",
  "Email": "jane@yahoo.com",
  "LoyaltyPoints": 500
}
```

Note: Item 102 has `LoyaltyPoints`; Item 101 does not. **Valid.**

---

## 3.8 How DynamoDB Stores Data Internally

High-level internal model (simplified):

```
┌─────────────────────────────────────────────────────────────┐
│                        DynamoDB Table                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │ Partition 1 │ │ Partition 2 │ │ Partition 3 │  ...      │
│  │ (Hash Ring) │ │             │ │             │           │
│  │             │ │             │ │             │           │
│  │ Items where │ │ Items where │ │ Items where │           │
│  │ hash(PK) → 1│ │ hash(PK) → 2│ │ hash(PK) → 3│           │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘           │
│         │               │               │                   │
│         ▼               ▼               ▼                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │ Storage Node│ │ Storage Node│ │ Storage Node│  (SSD)    │
│  │ + Replicas  │ │ + Replicas  │ │ + Replicas  │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

**Steps when you write an item:**

1. SDK sends `PutItem` to DynamoDB endpoint
2. **Request router** hashes partition key → identifies partition
3. **Partition leader** receives write
4. Data written to **SSD storage** on that node
5. **Synchronously replicated** to 2 other AZs (within region)
6. Acknowledgment returned to client

**B-tree analogy:** Within each partition, sort keys are ordered in a B-tree-like structure for efficient range queries.

---

# Section 4: Partitioning and Sharding

> **This is the most important section for interviews and production success.**

## 4.1 What is a Partition?

A **partition** (formerly called a "shard" internally) is a **physical storage allocation** that holds a subset of your table's data. DynamoDB automatically creates and manages partitions.

```
Table: Orders (logical view — one table)
═══════════════════════════════════════

Physical storage (partitions):

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Partition A   │  │   Partition B   │  │   Partition C   │
│                 │  │                 │  │                 │
│ PK hash → A     │  │ PK hash → B     │  │ PK hash → C     │
│                 │  │                 │  │                 │
│ ~10 GB max *    │  │                 │  │                 │
│ 3000 RCU / WCU  │  │                 │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘

* Partition splits when size or throughput exceeds limits
```

Each partition:

- Has its own SSD storage
- Has throughput limits (3000 RCU or 1000 WCU per partition, or proportional share)
- Is replicated across 3 Availability Zones

---

## 4.2 What is Sharding?

**Sharding** is splitting data across multiple independent database nodes. **Partitioning in DynamoDB IS sharding** — but fully automated.

```
Before Sharding (single server):

┌──────────────────────────────────────┐
│           Single Database            │
│  All 10 million items here           │
│  Max throughput: limited             │
└──────────────────────────────────────┘


After Sharding (partitioned):

┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Shard 1  │ │ Shard 2  │ │ Shard 3  │ │ Shard 4  │
│ 2.5M     │ │ 2.5M     │ │ 2.5M     │ │ 2.5M     │
│ items    │ │ items    │ │ items    │ │ items    │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

In MongoDB/Cassandra, **you** choose the shard key. In DynamoDB, **AWS** manages partition count based on size and throughput.

---

## 4.3 Why Partitioning is Required

| Reason | Explanation |
|--------|-------------|
| **Storage limits** | One machine cannot hold petabytes efficiently |
| **Throughput limits** | One disk/controller has IOPS ceiling |
| **Parallelism** | Many partitions = many parallel read/write paths |
| **Fault isolation** | Failure affects one partition, not entire table |
| **Elastic growth** | Add partitions without application changes |

---

## 4.4 Hashing — How Partition Placement Works

DynamoDB uses an internal hash function on the **partition key value**:

```
                    Partition Key: "CustomerID" = "1001"
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │  Internal Hash Fn   │
                         │  hash("1001")       │
                         │  = 0x7A3F2B1C...    │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │  Partition Map      │
                         │  (consistent hash   │
                         │   ring)             │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Partition 1     Partition 2     Partition 3
              (not this)      (SELECTED)      (not this)
```

**Examples:**

```
CustomerID=1001  →  hash → Partition 2
CustomerID=1002  →  hash → Partition 5
CustomerID=1003  →  hash → Partition 2  (same partition as 1001 — OK, different keys)
```

**Important:** You cannot choose or see partition numbers in the console. You only design the **partition key** — AWS handles the rest.

---

## 4.5 Hot Partitions

A **hot partition** occurs when too many requests hit the **same partition key** (or too few keys dominate).

```
BAD DESIGN: Partition Key = "GameID" (only one game)

                    All writes
                        │
                        ▼
              ┌─────────────────┐
              │  Partition X    │  ← 10,000 WCU attempted
              │  (GameID=xyz)   │     Partition max ~1,000 WCU
              └─────────────────┘     → THROTTLED (429)

GOOD DESIGN: Partition Key = "GameID#PlayerID"

  Player A writes → Partition 3
  Player B writes → Partition 17
  Player C writes → Partition 8
  → Load spread evenly
```

**Symptoms:**
- `ProvisionedThroughputExceededException`
- CloudWatch: `ThrottledRequests` metric spikes
- Uneven `ConsumedReadCapacityUnits` per key (hard to see directly)

**Mitigations:**
- Add random suffix: `ORDER#<uuid>`
- Write sharding: `PK = shard-N` where N = random(0-9)
- Use sort key to distribute (e.g., timestamp per event)

---

## 4.6 Partition Limits

| Limit | Value |
|-------|-------|
| Max item size | 400 KB |
| Max partition throughput | 3,000 RCU + 1,000 WCU (provisioned model per partition) |
| Partition size trigger for split | ~10 GB of data per partition |
| Max partitions per table | Unlimited (grows with data/throughput) |

When limits are exceeded, DynamoDB **splits** the partition:

```
Before Split:
┌──────────────────────────────┐
│      Partition 5 (10+ GB)    │
│  hash range: 0x00 - 0xFF     │
└──────────────────────────────┘

After Split:
┌──────────────────┐  ┌──────────────────┐
│   Partition 5a   │  │   Partition 5b   │
│  hash: 0x00-0x7F │  │  hash: 0x80-0xFF │
└──────────────────┘  └──────────────────┘
```

Splits happen **automatically** and are **online** (no downtime). Brief latency jitter possible during split.

---

## 4.7 Partition Splitting Example

**Scenario:** E-commerce table with `ProductID` as partition key. One viral product gets 50 GB of review items.

```
1. Reviews accumulate under ProductID="VIRAL-123"
2. Partition exceeds 10 GB
3. DynamoDB splits partition
4. If STILL hot (all same PK), split doesn't help throughput!
   → Must redesign key to distribute writes
```

**Lesson:** Splits solve **size** problems. **Key design** solves **hot spot** problems.

---

## 4.8 Detailed Hot Partition Case Study

**Scenario:** Social media "likes" counter on a viral post.

```
BAD Design:
  PK = "POST#viral-999"
  SK = "LIKE"
  Attribute: LikeCount (incremented on every like)

Every like → same partition → throttled at ~1000 WCU

Traffic:
  Like ──┐
  Like ──┤
  Like ──┼──► Partition POST#viral-999  💥 HOT
  Like ──┤
  Like ──┘
```

**Solution A — Sharded counters:**

```
Write to random shard:
  PK = "POST#viral-999#SHARD#3"   (shard 0-9)
  SK = "COUNTER"
  UpdateItem ADD LikeCount :1

Read total:
  Query PK begins_with "POST#viral-999#SHARD"
  Sum all LikeCount values in application
```

```
Sharded Counter Layout:

POST#viral-999#SHARD#0  →  LikeCount: 10,231
POST#viral-999#SHARD#1  →  LikeCount: 10,105
POST#viral-999#SHARD#2  →  LikeCount: 9,998
...
POST#viral-999#SHARD#9  →  LikeCount: 10,402

Total = sum = ~102,000 likes (10 parallel write paths)
```

**Solution B — DynamoDB Streams + aggregate:**

```
Like event → Kinesis → Lambda batches → periodic counter update
(Sacrifices real-time accuracy for write scalability)
```

---

## 4.9 Partition Key Selection Decision Tree

```
Start: What uniquely identifies your most common access pattern?
         │
         ▼
┌─────────────────────────────────────┐
│ Single entity lookup by ID?         │
│ YES → PK = EntityID (UUID)          │
└─────────────────────────────────────┘
         │ NO
         ▼
┌─────────────────────────────────────┐
│ One-to-many within parent?          │
│ YES → PK = ParentID, SK = ChildID   │
└─────────────────────────────────────┘
         │ NO
         ▼
┌─────────────────────────────────────┐
│ Time-series per device/user?        │
│ YES → PK = DeviceID, SK = Timestamp │
│ (NEVER PK = Date alone)             │
└─────────────────────────────────────┘
         │ NO
         ▼
┌─────────────────────────────────────┐
│ Need query by secondary attribute?  │
│ YES → Add GSI with high-cardinality │
│       GSI partition key             │
└─────────────────────────────────────┘
```

---

## 4.10 Hash Collision and Distribution

Partition keys hash to partitions — **many keys share one partition** (this is normal):

```
Partition 7 contains:
  CustomerID=101
  CustomerID=5088
  CustomerID=92001
  ... (thousands of unrelated customer IDs)

This is GOOD — partition handles many keys.
This is BAD — one KEY gets all traffic (hot partition).
```

**Uniform distribution test:** If you have 1 million users and 100 partitions, ideally ~10,000 users per partition. Random UUIDs achieve this. Sequential IDs (1, 2, 3...) also hash well in DynamoDB's internal hash function — contrary to some myths about sequential IDs always causing hot spots. The hot spot problem is about **access frequency**, not **key distribution**.

```
Access frequency matters more than key sequence:

Sequential IDs, even distribution of READS:
  ID=1, ID=2, ID=3 ... each accessed equally → OK

Sequential IDs, only ID=1 accessed:
  Only one key hot → BAD (regardless of hash quality)
```

---

# Section 5: Horizontal Scaling

## 5.1 Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)

```
Before:                    After:
┌──────────┐              ┌──────────┐
│ Server   │              │ Server   │
│ 4 CPU    │   ──────►    │ 32 CPU   │
│ 16 GB RAM│              │ 256 GB   │
└──────────┘              └──────────┘

Pros: Simple
Cons: Hardware ceiling, downtime for upgrades, expensive
```

### Horizontal Scaling (Scale Out)

```
Before:                    After:
┌──────────┐              ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Server 1 │              │ Server 1 │ │ Server 2 │ │ Server 3 │ │ Server 4 │
└──────────┘              └──────────┘ └──────────┘ └──────────┘ └──────────┘

Pros: Near-unlimited scale, fault tolerance
Cons: Data distribution complexity (DynamoDB handles this)
```

---

## 5.2 Why DynamoDB Scales to Millions of RPS

```
Request Load: 5,000,000 reads/second
                    │
                    ▼
         ┌──────────────────────┐
         │  DynamoDB Request  │
         │  Router Fleet      │  (horizontally scaled)
         └──────────┬───────────┘
                    │
    ┌───────┬───────┼───────┬───────┬───────┐
    ▼       ▼       ▼       ▼       ▼       ▼
  Part  Part  Part  Part  Part  ... Part
   #1    #2    #3    #4    #5        #N

Each partition serves independent subset of keys in parallel
```

**Factors enabling scale:**

1. **Partition parallelism** — Thousands of partitions serve requests simultaneously
2. **SSD storage** — Low-latency reads/writes per partition
3. **Auto-partitioning** — New partitions added as data/throughput grows
4. **On-demand mode** — Instantly adapts to traffic without capacity planning
5. **No connection bottleneck** — HTTP API; no long-lived connection pools
6. **Multi-AZ replication** — Reads can be served from replicas (eventually consistent)

---

## 5.3 Internal AWS Infrastructure (Conceptual)

```
┌────────────────────────────────────────────────────────────────┐
│                     AWS Region (us-east-1)                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│  │    AZ-1a    │  │    AZ-1b    │  │    AZ-1c    │           │
│  │  Storage    │  │  Storage    │  │  Storage    │           │
│  │  Nodes      │  │  Nodes      │  │  Nodes      │           │
│  └─────────────┘  └─────────────┘  └─────────────┘           │
│         ▲                ▲                ▲                     │
│         └────────────────┼────────────────┘                     │
│                    Replication                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              DynamoDB Control Plane                      │   │
│  │  (partition management, splits, health, routing tables)  │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

You never SSH into these nodes. AWS manages patching, hardware failure, and rebalancing.

---

## 5.4 Scaling Scenarios with Numbers

### Scenario A: Startup MVP

```
Traffic: 50 reads/sec, 10 writes/sec
Item size: 2 KB reads, 1 KB writes
Capacity needed (provisioned, eventual reads):
  RCU: 50 × ceil(2/4) × 0.5 = 50 × 1 × 0.5 = 25 RCU
  WCU: 10 × ceil(1/1) = 10 WCU
Recommendation: On-demand (traffic unpredictable) or 25/10 provisioned
```

### Scenario B: Enterprise Peak Event

```
Traffic: 500,000 reads/sec during 2-hour sale
Partitions needed (rough): 500,000 × 0.5 RCU / 3000 ≈ 84+ partitions
DynamoDB on-demand: Handles automatically
Provisioned without auto-scaling: Throttling disaster
```

### Scenario C: Steady-State SaaS

```
Traffic: 24/7 2000 RPS reads, 500 WPS
Auto-scaling provisioned:
  Target utilization: 70%
  Min: 100 RCU / 50 WCU
  Max: 5000 RCU / 2000 WCU
Cost: Typically 30-50% less than on-demand at steady load
```

```
Scaling comparison for Scenario B:

Vertical (RDS):
  db.r6g.16xlarge → still may hit connection/CPU limits

Horizontal (DynamoDB):
  500K RPS → auto-partitions → no manual intervention
```

---

# Section 6: DynamoDB Storage Architecture

## 6.1 Request Flow

```
┌────────┐
│ Client │  (Lambda, EC2, local dev with AWS SDK)
└───┬────┘
    │ HTTPS (TLS 1.2+)
    │ SigV4 signed request
    ▼
┌────────────────────┐
│  DynamoDB Router   │  Resolves table → partition map
│  (Request Layer)   │  Authentication via IAM
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│  Partition Leader  │  Coordinates read/write for partition
└────────┬───────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌────────┐ ┌────────┐
│Storage │ │Replica │  (other AZs)
│ Node   │ │ Nodes  │
└────────┘ └────────┘
```

---

## 6.2 Replication and Fault Tolerance

Every partition maintains **3 copies** across 3 AZs in the region:

```
Write to Partition 7:

  AZ-1a (Leader)  ──sync──►  AZ-1b (Replica)
       │
       └────sync────►  AZ-1c (Replica)

If AZ-1a fails → replica promoted → no data loss (for acknowledged writes)
```

**Fault tolerance:** Survives single AZ failure without data loss for committed writes.

---

## 6.3 Multi-AZ Architecture

```
Region: eu-west-1
┌─────────────────────────────────────────────────────────┐
│  AZ-a          AZ-b          AZ-c                        │
│  ┌─────┐      ┌─────┐      ┌─────┐                      │
│  │Copy1│      │Copy2│      │Copy3│   ← Same partition  │
│  └─────┘      └─────┘      └─────┘                      │
└─────────────────────────────────────────────────────────┘
```

Clients connect to a **regional endpoint** — not a specific AZ.

---

## 6.4 Consistency Models

### Eventually Consistent Reads (Default)

- May return slightly stale data
- Costs **0.5 RCU** per 4 KB (half of strong)
- Higher read throughput

```
Timeline:
  T0: Write version=5 to leader
  T1: Read from replica (still version=4) → returns 4
  T2: Replication completes
  T3: Read from replica → returns 5
```

### Strongly Consistent Reads

- Returns latest write
- Costs **1 RCU** per 4 KB
- Use: `ConsistentRead=true` in GetItem/Query/BatchGetItem

```
Timeline:
  T0: Write version=5
  T1: Strong read → waits for replication/quorum → returns 5
```

**Limitation:** Strong consistency only guaranteed **within the same region**, for reads after writes to the same partition key.

---

## 6.5 Consistency Examples in Practice

### Example 1: Bank Balance Read

```
Requirement: User must see updated balance immediately after transfer

Write: UpdateItem on PK=ACCOUNT#101, ADD Balance :amount
Read:  GetItem with ConsistentRead=true

Why: Financial accuracy trumps extra RCU cost
```

### Example 2: Social Media Feed

```
Requirement: Show latest posts; 1-2 second delay acceptable

Write: PutItem new post
Read:  Query with ConsistentRead=false (default)

Why: Higher throughput, lower cost, UX tolerates brief delay
```

### Example 3: Read-Your-Own-Writes Pattern

```
User updates profile → immediately views profile

Option A: ConsistentRead=true on GetItem
Option B: Return updated data from Lambda response (skip read)
Option C: Read from base table (not GSI) with consistent read

Avoid: Write to base table → read from GSI immediately
```

---

## 6.6 Durability and the Write Path

```
Client sends PutItem
        │
        ▼
   Leader partition (AZ-a)
        │
        ├── Replicate to AZ-b (sync)
        ├── Replicate to AZ-c (sync)
        │
        ▼
   Acknowledge success to client

If leader fails before ack → client retries (idempotent with same key)
If leader fails after ack → data safe on replicas
```

**Durability:** Once write succeeds, data survives AZ failure.

---

# Section 7: AWS Console Walkthrough

## 7.1 Login to AWS Console

```
Step 1: Open https://console.aws.amazon.com
Step 2: Enter Account ID or alias, IAM username, password
Step 3: Complete MFA if required
Step 4: You land on AWS Management Console home

┌────────────────────────────────────────────────────────────┐
│  AWS Management Console                    [Region ▼]      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 🔍 Search: dynamodb                                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Recently visited services...                              │
└────────────────────────────────────────────────────────────┘
```

**Tip:** Select your target region (top-right). DynamoDB tables are regional.

---

## 7.2 Open DynamoDB Service

```
Step 1: Click search bar → type "DynamoDB"
Step 2: Click "DynamoDB" under Services

┌────────────────────────────────────────────────────────────┐
│  Amazon DynamoDB                          [us-east-1 ▼]    │
├────────────────────────────────────────────────────────────┤
│  [Tables]  [Explore items]  [PartiQL editor]  [Backups]    │
│                                                            │
│  Tables (0)                          [Create table]        │
│  ┌──────────────────────────────────────────────────────┐│
│  │ No tables                                            ││
│  └──────────────────────────────────────────────────────┘│
└────────────────────────────────────────────────────────────┘
```

---

## 7.3 Create Table — Step by Step

Click **Create table**.

```
┌────────────────────────────────────────────────────────────┐
│  Create DynamoDB table                                     │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Table name                                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Customers                                            │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Partition key                                             │
│  ┌────────────────────┐  ┌─────────────────────────────┐   │
│  │ CustomerID         │  │ String                  ▼  │   │
│  └────────────────────┘  └─────────────────────────────┘   │
│                                                            │
│  ☐ Sort key - optional                                     │
│     ┌────────────────┐  ┌─────────────┐                    │
│     │ (empty)        │  │ String   ▼  │                    │
│     └────────────────┘  └─────────────┘                    │
│                                                            │
│  Table settings                                            │
│  ○ Default settings                                        │
│  ● Custom settings                                         │
│                                                            │
│                              [Cancel]  [Create table]      │
└────────────────────────────────────────────────────────────┘
```

### Option Explanations

| Option | Description |
|--------|-------------|
| **Table name** | Unique per account per region |
| **Partition key** | Required; HASH key; determines data distribution |
| **Sort key** | Optional; RANGE key; enables composite key and range queries |
| **Default settings** | On-demand capacity, encryption on, deletion protection off |
| **Custom settings** | Choose provisioned capacity, secondary indexes, streams, etc. |

---

## 7.4 Configure Capacity

Expand **Custom settings**:

```
┌────────────────────────────────────────────────────────────┐
│  Table capacity                                            │
│  ○ On-demand                                               │
│     Pay per request; auto-scales; good for unknown traffic   │
│  ● Provisioned                                             │
│     Read capacity:  [  5  ] RCU                            │
│     Write capacity: [  5  ] WCU                            │
│     ☑ Auto scaling (recommended)                           │
│                                                            │
│  Secondary indexes                                         │
│  [Add index]                                               │
│                                                            │
│  Encryption                                                │
│  ● Owned by Amazon DynamoDB (default)                      │
│  ○ AWS owned key                                           │
│  ○ AWS managed key (KMS)                                   │
│                                                            │
│  ☑ Deletion protection (recommended for production)        │
└────────────────────────────────────────────────────────────┘
```

Click **Create table**. Status shows **Creating** → **Active** (usually seconds).

---

## 7.5 Post-Creation Console Overview

After table is **Active**:

```
┌────────────────────────────────────────────────────────────┐
│  Tables > Customers                          [Actions ▼]   │
├────────────────────────────────────────────────────────────┤
│  [Overview] [Indexes] [Metrics] [Access control] ...       │
├────────────────────────────────────────────────────────────┤
│  General information                                       │
│  Table name:     Customers                                 │
│  Partition key:  CustomerID (String)                       │
│  Sort key:       -                                         │
│  Capacity mode:  On-demand                                 │
│  Table status:   Active                                    │
│  Item count:     0                                         │
│  Table size:     0 bytes                                   │
│  ARN:            arn:aws:dynamodb:us-east-1:...:table/...  │
├────────────────────────────────────────────────────────────┤
│  [Explore table items]  [Create item]  [Delete table]      │
└────────────────────────────────────────────────────────────┘
```

### Tabs Explained

| Tab | Purpose |
|-----|---------|
| **Overview** | Key schema, capacity, size, stream ARN, encryption |
| **Indexes** | View/create GSIs, view LSIs |
| **Metrics** | CloudWatch graphs: capacity, throttles, latency |
| **Access control** | IAM policy examples, fine-grained access |
| **Exports and streams** | Enable/disable streams, export to S3 |
| **Backups** | On-demand backups, PITR settings |
| **Global tables** | Add/remove replica regions |
| **Permissions** | Resource-based policies (where applicable) |

---

## 7.6 Creating Table with Sort Key (Composite Key)

For an Orders table:

```
┌────────────────────────────────────────────────────────────┐
│  Create DynamoDB table                                     │
├────────────────────────────────────────────────────────────┤
│  Table name: Orders                                        │
│                                                            │
│  Partition key:  UserID          (String)                  │
│  ☑ Sort key      OrderDate       (String)                  │
│                                                            │
│  Capacity: ○ On-demand  ● Provisioned                      │
│    Read: 10 RCU    Write: 10 WCU                           │
│    ☑ Enable auto scaling                                   │
│                                                            │
│  [Create table]                                            │
└────────────────────────────────────────────────────────────┘
```

**Sort key type tip:** Use ISO 8601 strings (`2024-06-21T10:30:00Z`) for chronological sorting.

---

## 7.7 Adding a GSI from Console

```
Tables → Orders → Indexes tab → Create index

┌────────────────────────────────────────────────────────────┐
│  Create global secondary index                             │
├────────────────────────────────────────────────────────────┤
│  Index name:        OrderStatusIndex                       │
│  Partition key:     OrderStatus    (String)                │
│  Sort key:          OrderDate      (String)  [optional]    │
│                                                            │
│  Projected attributes:                                     │
│  ○ All                                                   │
│  ● Include:  [UserID] [Total]                            │
│  ○ Keys only                                               │
│                                                            │
│  Capacity: Inherit from table (on-demand) or custom       │
│                                                            │
│  [Create index]                                            │
└────────────────────────────────────────────────────────────┘

Status: Creating → Active (backfill existing data — takes time on large tables)
```

---

# Section 8: Creating Data in Console UI

## 8.1 Create Item Walkthrough

```
Step 1: Tables → Click "Customers"
Step 2: Click "Explore table items"
Step 3: Click "Create item"
Step 4: Choose "JSON" view

┌────────────────────────────────────────────────────────────┐
│  Create item — Customers                                   │
├────────────────────────────────────────────────────────────┤
│  [Form view]  [JSON view]                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ {                                                    │  │
│  │   "CustomerID": "101",                               │  │
│  │   "Name": "John",                                    │  │
│  │   "Email": "john@gmail.com"                          │  │
│  │ }                                                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                              [Cancel]  [Create item]       │
└────────────────────────────────────────────────────────────┘
```

**CustomerID is required** — it's the partition key. Omitting it causes an error.

---

## 8.2 Attribute Types in Console

| Type | Example | Console Notation |
|------|---------|------------------|
| String | `"John"` | S |
| Number | `42` or `3.14` | N (stored as string internally) |
| Boolean | `true` | BOOL |
| List | `["a", "b"]` | L |
| Map | `{"City": "NYC"}` | M |
| String Set | `{"a", "b"}` (unique strings) | SS |
| Null | `null` | NULL |

**Example with nested types:**

```json
{
  "CustomerID": "101",
  "Name": "John",
  "Email": "john@gmail.com",
  "Active": true,
  "OrderCount": 15,
  "Tags": ["vip", "early-adopter"],
  "Address": {
    "Street": "123 Main St",
    "City": "New York",
    "Zip": "10001"
  }
}
```

---

# Section 9: Reading Data

## 9.1 GetItem

**Purpose:** Retrieve **one item** by **exact primary key**.

```
GetItem(CustomerID = "101")

┌─────────────────────────────────────┐
│ Partition for "101"                 │
│   └── Direct lookup O(1)            │
│       Returns 1 item or nothing     │
└─────────────────────────────────────┘

Performance: Single-digit ms · Most efficient operation
```

```python
# Boto3 example
response = table.get_item(Key={'CustomerID': '101'})
item = response.get('Item')
```

---

## 9.2 Query

**Purpose:** Retrieve **multiple items** sharing the **same partition key**, optionally filtered by sort key condition.

```
Query: UserID = "user-1" AND OrderDate > "2024-01-01"

┌─────────────────────────────────────┐
│ Partition "user-1"                  │
│   ├── 2024-01-15  ✓                 │
│   ├── 2024-02-20  ✓                 │
│   └── 2023-12-01  ✗                 │
└─────────────────────────────────────┘

Performance: Efficient — touches ONE partition (unless cross-partition GSI)
```

**Requires:** Partition key equality condition. Sort key supports: `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `begins_with`.

---

## 9.3 Scan

**Purpose:** Read **every item** in the table (or secondary index).

```
Scan entire table:

┌──────────┐ ┌──────────┐ ┌──────────┐
│ Part 1   │ │ Part 2   │ │ Part 3   │  ... ALL partitions
│ READ ALL │ │ READ ALL │ │ READ ALL │
└──────────┘ └──────────┘ └──────────┘

Performance: O(table size) — SLOW and EXPENSIVE on large tables
```

**Use Scan only for:** Small tables, one-time migrations, admin tasks — **never** in hot production paths.

---

## 9.4 Comparison Table

| Operation | Key Required | Partitions Touched | Cost | Use Case |
|-----------|--------------|-------------------|------|----------|
| **GetItem** | Full primary key | 1 | Lowest | Fetch one known item |
| **Query** | Partition key (+ SK condition) | 1 (base table) | Low | Fetch related items |
| **Scan** | None | ALL | Highest | Full table export (avoid) |

---

## 9.5 Query Example: CustomerID = 101

Console → Explore items → Query:

```
┌────────────────────────────────────────────────────────────┐
│  Query or scan items                                       │
│  ● Query                                                   │
│                                                            │
│  Partition key: CustomerID                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 101                                                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  [Run]                                                     │
│                                                            │
│  Results: 1 item                                           │
│  CustomerID=101, Name=John, Email=john@gmail.com           │
└────────────────────────────────────────────────────────────┘
```

---

## 9.6 FilterExpression Deep Dive

`FilterExpression` applies **after** items are read from storage — it does **not** reduce RCU consumption.

```
Query PK=USER#101 (partition has 500 order items)
FilterExpression: Status = 'SHIPPED'

Process:
  1. Read all 500 items from partition (billed RCU for 500)
  2. Apply filter in memory
  3. Return 120 matching items

Client receives 120 items but paid for 500 reads
```

**When FilterExpression is OK:**
- Small result sets per partition key
- Filtering on non-key attributes where no GSI exists (temporary/dev)
- Reducing **returned** data over the wire (not RCU)

**When to avoid:**
- Large partitions with selective filters → create GSI

---

## 9.7 Parallel Scan for Data Migration

For **offline** bulk processing only:

```python
# Worker 0 of 4 parallel segments
table.scan(Segment=0, TotalSegments=4)
# Worker 1
table.scan(Segment=1, TotalSegments=4)
# ... etc
```

```
Table split across 4 parallel scanners:

Segment 0 ──► Partitions 1, 5, 9...
Segment 1 ──► Partitions 2, 6, 10...
Segment 2 ──► Partitions 3, 7, 11...
Segment 3 ──► Partitions 4, 8, 12...

4× faster migration BUT consumes 4× RCU — never on production table during peak
```

---

## 9.8 ProjectionExpression

Return only needed attributes to reduce bandwidth:

```python
table.get_item(
    Key={'CustomerID': '101'},
    ProjectionExpression='CustomerID, #n, Email',
    ExpressionAttributeNames={'#n': 'Name'}
)
# Returns only 3 attributes instead of full item
```

Note: RCU is still based on **full item size** on disk, not projected size.

---

# Section 10: Secondary Indexes

## 10.1 Why Secondary Indexes?

The primary key determines access patterns. If you need to query by **Email** but partition key is **CustomerID**, you cannot Query efficiently — you would Scan.

**Solution:** Create an index with a different key schema.

---

## 10.2 GSI — Global Secondary Index

A **GSI** has its **own partition key and optional sort key**, different from the base table. GSIs have **separate throughput** and span **all partitions**.

```
Base Table:                    GSI: EmailIndex
PK: CustomerID                 PK: Email
                               SK: (optional)

┌─────────────────────┐         ┌─────────────────────┐
│ CustomerID │ Email  │         │ Email      │ CustID │
├────────────┼────────┤  ───►   ├────────────┼────────┤
│ 101        │ j@...  │         │ j@...      │ 101    │
│ 102        │ ja@... │         │ ja@...     │ 102    │
└────────────┴────────┘         └─────────────────────┘

Query GSI: Email = "john@gmail.com" → CustomerID 101
```

**GSI properties:**
- Eventually consistent only (no strong consistency option)
- Separate provisioned/on-demand capacity
- Maximum 20 GSIs per table
- Projection: ALL, KEYS_ONLY, or INCLUDE specific attributes

```
Architecture:

        Query on GSI "EmailIndex"
                 │
                 ▼
        ┌─────────────────┐
        │  GSI Partitions │  (independent partition layout)
        └─────────────────┘
                 │
                 ▼ (async replication from base table)
        ┌─────────────────┐
        │  Base Table     │
        └─────────────────┘
```

---

## 10.3 LSI — Local Secondary Index

An **LSI** shares the **same partition key** as the base table but has a **different sort key**. Must be created **at table creation time**.

```
Base Table:
PK: UserID    SK: OrderDate

LSI: UserID (same PK)    SK: OrderTotal

Query: UserID="u1" sorted by OrderTotal instead of OrderDate
```

**LSI properties:**
- Same partition as base item → 10 GB per partition key limit (PK+LSI combined)
- Strongly consistent reads supported
- Max 5 LSIs per table
- Must be defined at table creation

```
                    Partition "user-1"
                    ┌─────────────────────────────┐
Base Table SK:      │ OrderDate ordering          │
                    ├─────────────────────────────┤
LSI SK:             │ OrderTotal ordering         │
                    │ (same items, different sort)│
                    └─────────────────────────────┘
```

---

## 10.4 GSI vs LSI Comparison

| Feature | GSI | LSI |
|---------|-----|-----|
| Partition key | Different from base | Same as base |
| Sort key | Different | Different |
| When to create | Anytime | At table creation only |
| Consistency | Eventually only | Strong available |
| Throughput | Separate | Shares table throughput |
| Per-partition size | Independent | Shared 10 GB limit |
| Max per table | 20 | 5 |

---

## 10.5 Use Cases

| Pattern | Index Type |
|---------|------------|
| Lookup user by email | GSI on Email |
| Orders by user, sort by date OR amount | LSI (same UserID) |
| Query orders by status across all users | GSI on Status (watch hot partitions!) |
| Multi-tenant: tenant_id + created_at | Composite keys on GSI |

---

## 10.6 Performance Considerations

1. **GSI writes cost extra WCU** — every base table write propagates to GSIs
2. **Sparse indexes** — only items with indexed attributes appear (saves space)
3. **Over-projection** — projecting ALL attributes doubles storage cost
4. **Hot GSI partitions** — low-cardinality GSI keys (e.g., `Status=ACTIVE`) cause throttling

---

# Section 11: Capacity Modes

## 11.1 Provisioned Capacity

You specify **read capacity units (RCU)** and **write capacity units (WCU)**. DynamoDB reserves throughput for your table.

```
┌────────────────────────────────────────────────────────────┐
│  Table: Orders (Provisioned)                               │
│  Read:  100 RCU  ──► Auto Scaling ──► 50-500 RCU          │
│  Write: 50 WCU   ──► Auto Scaling ──► 25-200 WCU          │
└────────────────────────────────────────────────────────────┘
```

**Best for:** Steady, predictable traffic; cost optimization when load is stable.

---

## 11.2 On-Demand Capacity

Pay per request; no capacity planning. DynamoDB instantly scales to handle any traffic.

```
Traffic spike:
  100 RPS ──► 50,000 RPS ──► 100 RPS
              (no throttling, no manual scaling)
```

**Best for:** Unknown or spiky workloads, new projects, dev/test.

---

## 11.3 RCU — Read Capacity Units

| Read Type | RCU Consumption |
|-----------|-----------------|
| Eventually consistent | **0.5 RCU** per 4 KB |
| Strongly consistent | **1 RCU** per 4 KB |

**Example:** Item size 10 KB, eventually consistent read:
- RCU = ceil(10/4) × 0.5 = ceil(2.5) × 0.5 = 3 × 0.5 = **1.5 RCU**

**Example:** 100 RCU provisioned (eventually consistent) = 100 / 0.5 = **200 reads/sec** for 4 KB items.

---

## 11.4 WCU — Write Capacity Units

| Write Type | WCU Consumption |
|------------|-----------------|
| Standard write | **1 WCU** per 1 KB |
| Transactional write | **2 WCU** per 1 KB |

**Example:** Item size 3.5 KB write = ceil(3.5/1) = **4 WCU**

**Example:** 50 WCU = **50 writes/sec** for items ≤ 1 KB.

---

## 11.5 Cost Calculation Examples

**Provisioned (us-east-1 approximate):**
- 1 RCU/month ≈ $0.00013 × 24 × 30
- 1 WCU/month ≈ $0.00065 × 24 × 30
- Storage: ~$0.25/GB-month

**On-demand:**
- Write: ~$1.25 per million write request units
- Read: ~$0.25 per million read request units

**Scenario:** 1M reads/day (4 KB, eventual), 100K writes/day (2 KB)

```
Provisioned estimate:
  Reads:  1M/day ÷ 86400 ≈ 12 RCU (with headroom → 25 RCU)
  Writes: 100K × 2 WCU each ÷ 86400 ≈ 3 WCU (→ 10 WCU)

On-demand estimate:
  Reads:  1M × 1 RRu (4KB) = 1M RRu/day × $0.25/M = $0.25/day
  Writes: 100K × 2 WRu = 200K WRu/day × $1.25/M = $0.25/day
```

Use **AWS Pricing Calculator** for production estimates.

---

# Section 12: Advanced DynamoDB Features

## 12.1 DynamoDB Streams

**Ordered stream** of item-level changes (INSERT, MODIFY, REMOVE).

```
DynamoDB Table
      │
      │ (change capture)
      ▼
┌─────────────────┐
│ DynamoDB Stream │
│ Shard 1         │──► Lambda (process events)
│ Shard 2         │──► Kinesis Data Streams (export)
└─────────────────┘
```

**Use cases:** Audit log, replicate to Elasticsearch, trigger notifications, CQRS projections.

**Stream record contains:** Keys, NewImage, OldImage, event type, timestamp.

```
Architecture:

  PutItem(Order)
       │
       ▼
  ┌─────────┐     ┌──────────┐     ┌─────────────┐
  │ Table   │────►│ Stream   │────►│ Lambda      │
  └─────────┘     └──────────┘     │ - Send email│
                                   │ - Update ES │
                                   └─────────────┘
```

---

## 12.2 TTL (Time To Live)

Automatically delete items when a **TTL attribute** (epoch timestamp) expires.

```
Item:
{
  "SessionID": "abc",
  "Data": {...},
  "ExpiresAt": 1719000000   ← TTL attribute (Number, epoch seconds)
}

DynamoDB background process deletes when now > ExpiresAt
(No WCU charged for TTL deletes)
```

**Use cases:** Sessions, temporary tokens, log retention, cache entries.

**Note:** Deletion is **eventually** consistent (may take up to 48 hours after expiry).

---

## 12.3 DAX — DynamoDB Accelerator

In-memory cache **sitting in front of** DynamoDB.

```
Client
  │
  ▼
┌─────────┐   cache hit    ┌───────────┐
│   DAX   │◄──────────────►│ DynamoDB  │
│ Cluster │   cache miss   │  Table    │
└─────────┘                └───────────┘

Microsecond read latency for cached items
```

**Use cases:** Read-heavy apps, repeated reads of same keys, gaming leaderboards.

**Not for:** Write-heavy, always-fresh data requirements, strong consistency needs.

---

## 12.4 Global Tables

Multi-region, multi-active replication.

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  us-east-1   │◄───────►│  eu-west-1   │◄───────►│  ap-south-1  │
│  Orders      │  repl   │  Orders      │  repl   │  Orders      │
└──────────────┘         └──────────────┘         └──────────────┘

Writes in any region → replicated to all (last writer wins for conflicts)
```

**Use cases:** Global apps, disaster recovery, local low-latency writes.

**Requirements:** DynamoDB Streams enabled, same table name, compatible keys.

---

## 12.5 Transactions

**TransactWriteItems** and **TransactGetItems** — ACID across **up to 100 items** in **same account/region**.

```
TransactWriteItems:
  - ConditionCheck: Order exists
  - Update: Decrement inventory
  - Put: Create order record

All succeed or all fail (atomic)
```

**Cost:** 2× WCU/RCU compared to non-transactional operations.

**Limitation:** Cannot span tables in different regions; 4 MB transaction payload limit.

---

## 12.6 Backup and Restore

### On-Demand Backup

```
┌─────────┐                    ┌─────────────┐
│ Table   │ ── CreateBackup ──►│ S3-backed   │
│ (live)  │                    │ Backup      │
└─────────┘                    └─────────────┘
```

Full table snapshot; no performance impact; retained until deleted.

### Point-in-Time Recovery (PITR)

```
Timeline: ────●────●────●────●────●────►
             Jan  Mar  May  Jul  Sep
                    ▲
              Restore to any second
              within last 35 days
```

Continuous backups; restore to new table; protects against accidental deletes.

---

## 12.7 PartiQL

**PartiQL** is a SQL-compatible query language for DynamoDB.

```sql
-- Select with key (efficient — uses GetItem/Query underneath)
SELECT * FROM "Customers" WHERE "CustomerID" = '101';

-- Insert
INSERT INTO "Customers" VALUE {'CustomerID': '102', 'Name': 'Jane'};

-- Update
UPDATE "Customers" SET "Name" = 'Johnny' WHERE "CustomerID" = '101';

-- Delete
DELETE FROM "Customers" WHERE "CustomerID" = '101';
```

**Warning:** `SELECT * FROM table WHERE nonKeyAttribute = 'x'` may execute a **Scan** — always verify access pattern.

---

## 12.8 DynamoDB Export and Import

### Export to S3

```
┌─────────┐    PITR-based     ┌─────┐
│ Table   │ ───export───────► │ S3  │
└─────────┘    (no RCU cost)  └─────┘
                                    │
                                    ▼
                              Athena / EMR / Redshift
```

### Import from S3

```
S3 (DynamoDB JSON) ──ImportTable──► New DynamoDB Table
Bulk load without consuming write capacity on existing table
```

---

# Section 13: Data Modeling

## 13.1 Access Pattern First Design

Unlike SQL (design schema, then query), DynamoDB requires:

```
1. List ALL access patterns
2. Design keys and indexes for EACH pattern
3. Denormalize where needed
```

```
Access Patterns for E-commerce:
  AP1: Get user by UserID
  AP2: Get orders for user (sorted by date)
  AP3: Get product by ProductID
  AP4: Get order by OrderID
  AP5: List products by category
```

---

## 13.2 Single Table Design

Store **multiple entity types** in **one table** using composite keys.

**Why AWS recommends it:**

- One request can fetch related data (no joins, no multiple round trips)
- Fewer tables to manage
- Cost-efficient (one billing entity)
- Enforces thinking in access patterns

```
Single Table: Ecommerce

PK              SK              Attributes
─────────────────────────────────────────────
USER#101        PROFILE         name, email
USER#101        ORDER#2024-01   orderTotal, status
PRODUCT#500     METADATA        title, price
CATEGORY#Elect  PRODUCT#500     (GSI alternative)
ORDER#999       METADATA        userId, items
```

---

## 13.3 E-Commerce Single Table Example

```
┌────────────────────────────────────────────────────────────────┐
│ Table: Ecommerce                                                │
│ PK: PK (String)    SK: SK (String)                             │
├────────────────────────────────────────────────────────────────┤
│ PK=USER#101    SK=PROFILE     → {name, email}                  │
│ PK=USER#101    SK=ORDER#001   → {total: 99, date: ...}       │
│ PK=USER#101    SK=ORDER#002   → {total: 50, date: ...}         │
│ PK=PRODUCT#55  SK=METADATA    → {title, price, stock}          │
│ PK=ORDER#001   SK=METADATA    → {userId, items, status}        │
└────────────────────────────────────────────────────────────────┘

Query AP2: PK=USER#101 AND begins_with(SK, "ORDER#")
GetItem AP3: PK=PRODUCT#55, SK=METADATA
```

**GSI for category browse:**

```
GSI1: GSI1PK=CATEGORY#Electronics, GSI1SK=PRODUCT#55
Query: All products in Electronics category
```

---

## 13.4 Adjacency List Pattern

Model one-to-many as multiple items sharing partition key:

```
PK=ORG#acme
  SK=DEPT#eng     → Engineering dept metadata
  SK=EMP#001      → Employee Alice
  SK=EMP#002      → Employee Bob
  SK=PROJECT#x    → Project X
```

One Query returns entire organizational subtree.

---

## 13.5 Data Modeling Patterns Reference

### Pattern 1: Single Table with Entity Type Prefix

```
PK              SK              Entity
──────────────────────────────────────
USER#101        METADATA        User profile
USER#101        ORDER#20240601  Order summary
PRODUCT#55      METADATA        Product details
```

### Pattern 2: GSI Overloading

One GSI serves multiple access patterns using typed keys:

```
Base item:
  PK=USER#101, SK=PROFILE, Email=john@..., GSI1PK=EMAIL#john@..., GSI1SK=USER#101

GSI1 query by email:
  Query GSI1 where GSI1PK = EMAIL#john@...

Base item (order):
  PK=USER#101, SK=ORDER#001, GSI1PK=STATUS#SHIPPED, GSI1SK=2024-06-01

GSI1 query by status:
  Query GSI1 where GSI1PK = STATUS#SHIPPED
```

### Pattern 3: Materialized Aggregates

```
PK=USER#101, SK=STATS
  { totalOrders: 42, totalSpent: 5000, lastOrderDate: "2024-06-01" }

Updated atomically on each order via TransactWrite or Stream consumer
```

### Pattern 4: Write Sharding for High-Volume Feeds

```
PK=FEED#USER#101#SHARD#4, SK=TS#2024-06-21T10:00:00Z
  { postId: "xyz", content: "..." }

Read feed: Query all shards 0-9 in parallel, merge sort by timestamp
```

### Pattern 5: S3 Hybrid for Large Objects

```
DynamoDB item (metadata):
{
  "VideoID": "v-123",
  "Title": "Tutorial",
  "S3Key": "videos/v-123.mp4",
  "Duration": 3600,
  "SizeBytes": 500000000
}

Actual video in S3 — never store 500 MB in DynamoDB (400 KB limit)
```

---

## 13.6 E-Commerce Access Pattern Matrix

| # | Access Pattern | Operation | Key Design |
|---|----------------|-----------|------------|
| AP1 | Get user profile | GetItem | PK=USER#id, SK=PROFILE |
| AP2 | List user orders | Query | PK=USER#id, SK begins_with ORDER# |
| AP3 | Get single order | GetItem | PK=ORDER#id, SK=METADATA |
| AP4 | Get product | GetItem | PK=PRODUCT#id, SK=METADATA |
| AP5 | Products by category | Query GSI | GSI1PK=CATEGORY#name |
| AP6 | Orders by status (admin) | Query GSI | GSI2PK=STATUS#shipped (watch cardinality) |
| AP7 | Check inventory | GetItem + Condition | PK=PRODUCT#id, SK=INVENTORY |

---

## 13.7 Anti-Patterns in Data Modeling

```
ANTI-PATTERN 1: Relational normalization
  users table, orders table, products table (3 tables, 3 GetItems per page load)
  FIX: Single table, one Query for user + orders

ANTI-PATTERN 2: Generic PK for all entities
  PK=DATA, SK=entity#id  → single hot partition
  FIX: Distribute PK across entity IDs

ANTI-PATTERN 3: GSI on low-cardinality attribute
  GSI PK = OrderStatus (5 values) → 5 hot partitions
  FIX: Composite GSI key: STATUS#shipped#DATE#2024-06-21

ANTI-PATTERN 4: Storing computed reports in DynamoDB
  FIX: Streams → S3/Redshift for analytics
```

---

# Section 14: Real World Architecture

## 14.1 E-Commerce System — Full Request Lifecycle

```
User clicks "Place Order"
         │
         ▼
┌─────────────────┐
│ CloudFront CDN  │  Static assets (optional)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  API Gateway    │  POST /orders
│  - Cognito JWT  │  Validates user token
│  - Rate limit   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Lambda          │  1. Validate cart
│ PlaceOrderFn    │  2. TransactWriteItems:
│                 │     - Check inventory
│                 │     - Create order
│                 │     - Decrement stock
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ DynamoDB        │  Ecommerce table
│ (TransactWrite) │
└────────┬────────┘
         │
         │ Stream event
         ▼
┌─────────────────┐
│ Lambda          │  Send confirmation email (SES)
│ OrderNotifyFn   │  Update search index
└─────────────────┘
```

### Step-by-Step Lifecycle

| Step | Component | Action |
|------|-----------|--------|
| 1 | User | HTTPS POST with JWT |
| 2 | API Gateway | Auth, route to Lambda |
| 3 | Lambda | Business validation |
| 4 | DynamoDB | Atomic transaction write |
| 5 | DynamoDB Streams | Emit change record |
| 6 | Lambda (async) | Side effects (email, analytics) |
| 7 | API Gateway | Return 201 + order ID to user |

**Latency:** Typically 50–200 ms end-to-end for serverless stack.

---

## 14.2 Complete Boto3 Request Examples

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr
from decimal import Decimal

dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('Ecommerce')

# 1. Place order with transaction
client = boto3.client('dynamodb')
client.transact_write_items(
    TransactItems=[
        {
            'ConditionCheck': {
                'TableName': 'Ecommerce',
                'Key': {'PK': {'S': 'PRODUCT#55'}, 'SK': {'S': 'INVENTORY'}},
                'ConditionExpression': 'Stock >= :qty',
                'ExpressionAttributeValues': {':qty': {'N': '1'}}
            }
        },
        {
            'Update': {
                'TableName': 'Ecommerce',
                'Key': {'PK': {'S': 'PRODUCT#55'}, 'SK': {'S': 'INVENTORY'}},
                'UpdateExpression': 'ADD Stock :neg1',
                'ExpressionAttributeValues': {':neg1': {'N': '-1'}}
            }
        },
        {
            'Put': {
                'TableName': 'Ecommerce',
                'Item': {
                    'PK': {'S': 'ORDER#001'},
                    'SK': {'S': 'METADATA'},
                    'UserID': {'S': 'USER#101'},
                    'Total': {'N': '99.99'},
                    'Status': {'S': 'PENDING'}
                }
            }
        }
    ]
)

# 2. Query user orders with pagination
response = table.query(
    KeyConditionExpression=Key('PK').eq('USER#101') &
                           Key('SK').begins_with('ORDER#'),
    ScanIndexForward=False,  # newest first
    Limit=20
)
items = response['Items']
while 'LastEvaluatedKey' in response:
    response = table.query(
        KeyConditionExpression=Key('PK').eq('USER#101') &
                               Key('SK').begins_with('ORDER#'),
        ExclusiveStartKey=response['LastEvaluatedKey'],
        Limit=20
    )
    items.extend(response['Items'])

# 3. Idempotent insert
table.put_item(
    Item={'PK': 'USER#101', 'SK': 'PROFILE', 'Name': 'John'},
    ConditionExpression='attribute_not_exists(PK)'
)
```

---

## 14.3 Error Handling and Retry Pattern

```
Lambda → DynamoDB throttled (429)
              │
              ▼
     Exponential backoff
     attempt 1: wait 50ms
     attempt 2: wait 100ms
     attempt 3: wait 200ms
              │
              ▼
     Success OR DLQ after max retries
```

```python
from botocore.exceptions import ClientError
import time
import random

def write_with_retry(table, item, max_retries=5):
    for attempt in range(max_retries):
        try:
            table.put_item(Item=item)
            return
        except ClientError as e:
            if e.response['Error']['Code'] == 'ProvisionedThroughputExceededException':
                sleep = (2 ** attempt) * 0.05 + random.uniform(0, 0.05)
                time.sleep(sleep)
            else:
                raise
    raise Exception('Max retries exceeded')
```

---

## 14.4 Alternative Architectures

### EC2 / ECS Monolith

```
ALB → EC2 Auto Scaling Group → DynamoDB
         (connection pooling not needed — HTTP API)
```

### Event-Driven CQRS

```
Write API → DynamoDB (command store)
                │
                ▼ Streams
            Lambda → Read-optimized GSI / OpenSearch / S3 data lake
```

### Multi-Region Active-Active

```
Route 53 Latency Routing
    ├── us-east-1: API GW → Lambda → Global Table
    └── eu-west-1:  API GW → Lambda → Global Table
```

---

# Section 15: Performance Optimization

## 15.1 Avoiding Hot Partitions

| Technique | Example |
|-----------|---------|
| High-cardinality PK | `user-{uuid}` not `country-US` |
| Write sharding | `PK = shard-{hash%10}#user-123` |
| Random suffix | `SESSION#{uuid}` per session |
| Scatter-gather | Query 10 shards, merge results |

---

## 15.2 Efficient Partition Keys

```
GOOD keys:
  userId (millions of users)
  orderId (UUID)
  deviceId (IoT)

BAD keys:
  status (3 values)
  date (365 values/year — maybe OK with sort key distribution)
  tenantId alone (if 1 tenant = 90% traffic)
```

---

## 15.3 Index Optimization

- Use **sparse GSIs** — index only items needing alternate access
- Project **KEYS_ONLY** + GetItem for full item (if read pattern allows)
- Avoid GSIs you don't query
- Monitor **ConsumedCapacity** per index

---

## 15.4 Caching Strategy

```
Read path:

  Request → DAX (μs) → miss → DynamoDB (ms) → populate DAX
                │
                └── hit → return (fast)
```

**ElastiCache** alternative for complex caching logic outside DAX.

---

## 15.5 Best Practices Summary

1. Design for access patterns, not entities
2. Prefer Query over Scan — always
3. Use pagination (`Limit`, `LastEvaluatedKey`)
4. Enable auto-scaling or on-demand for production
5. Use exponential backoff on throttling
6. Keep items under 100 KB when possible (faster, cheaper)
7. Use batch operations (`BatchGetItem`, `BatchWriteItem`) for bulk
8. Monitor CloudWatch: `ThrottledRequests`, `UserErrors`, latency
9. Enable PITR on production tables
10. Use VPC endpoints to reduce NAT costs and improve security

---

# Section 16: Security

## 16.1 IAM

DynamoDB access is controlled via **IAM policies**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:Query",
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/Customers"
    }
  ]
}
```

**Fine-grained access:** Condition keys on `dynamodb:LeadingKeys` for multi-tenant row-level security.

```
Lambda Role → IAM Policy → dynamodb:Query on Orders table only
```

---

## 16.2 Encryption at Rest

| Option | Key Management |
|--------|----------------|
| AWS owned key | Default, no cost |
| AWS managed key (aws/dynamodb) | KMS, automatic rotation |
| Customer managed CMK | Full KMS control, audit trail |

All options use **AES-256**.

---

## 16.3 Encryption in Transit

All DynamoDB API calls use **HTTPS/TLS**. No plaintext endpoint available.

---

## 16.4 KMS Integration

Customer managed keys enable:
- Key rotation policies
- Cross-account access control
- CloudTrail audit of key usage

```
Encrypt with CMK:
  Write → DynamoDB → KMS Encrypt → Storage

Read:
  Storage → KMS Decrypt → Client
```

---

## 16.5 VPC Endpoints

```
┌─────────────────────────────────────────────┐
│ VPC                                         │
│  ┌─────────┐      PrivateLink      ┌──────┐ │
│  │ Lambda  │ ─────────────────────► │ DDB  │ │
│  │ in VPC  │   (no internet route)  │      │ │
│  └─────────┘                        └──────┘ │
└─────────────────────────────────────────────┘
```

**Gateway endpoint** for DynamoDB — no hourly charge; traffic stays on AWS network.

---

# Section 17: DynamoDB Interview Questions

## 17.1 Beginner Questions (1–50)

**Q1. What is DynamoDB?**  
A: A fully managed NoSQL key-value and document database on AWS that delivers single-digit millisecond latency at any scale without server management.

**Q2. Is DynamoDB relational?**  
A: No. It is a NoSQL database without SQL joins or fixed schemas.

**Q3. What is a table in DynamoDB?**  
A: A collection of items, like a table in SQL but without enforced column definitions.

**Q4. What is an item?**  
A: A single record/document identified by a primary key, up to 400 KB.

**Q5. What is an attribute?**  
A: A name-value pair within an item (e.g., `Name: "John"`).

**Q6. What is a partition key?**  
A: The HASH key that uniquely identifies an item (simple key tables) or determines the partition (composite key tables).

**Q7. What is a sort key?**  
A: The RANGE key combined with partition key to uniquely identify items and enable range queries within a partition.

**Q8. Can two items have the same partition key?**  
A: Yes, if the table has a composite key (partition + sort). No, if only a partition key is defined.

**Q9. What is the maximum item size?**  
A: 400 KB including attribute names and values.

**Q10. Is DynamoDB schema-less?**  
A: Items are schemaless (flexible attributes), but the primary key schema is fixed at table creation.

**Q11. What AWS service category is DynamoDB?**  
A: Database — fully managed NoSQL.

**Q12. Does DynamoDB support SQL?**  
A: PartiQL provides SQL-like statements for SELECT/INSERT/UPDATE/DELETE, but it is not a relational engine.

**Q13. What is PartiQL?**  
A: A SQL-compatible query language supported by DynamoDB for item operations.

**Q14. What regions can DynamoDB run in?**  
A: Most AWS commercial regions; tables are regional resources.

**Q15. What is the default consistency for reads?**  
A: Eventually consistent.

**Q16. How do you get strongly consistent reads?**  
A: Set `ConsistentRead=true` on GetItem, Query, or BatchGetItem.

**Q17. What is GetItem?**  
A: API operation to fetch one item by its full primary key.

**Q18. What is Query?**  
A: API to fetch items sharing the same partition key with optional sort key conditions.

**Q19. What is Scan?**  
A: API that reads every item in a table or index — expensive on large tables.

**Q20. Which is faster: Query or Scan?**  
A: Query — it targets one partition; Scan reads the entire table.

**Q21. What is RCU?**  
A: Read Capacity Unit — throughput measure for reads (1 RCU = 1 strongly consistent 4 KB read per second).

**Q22. What is WCU?**  
A: Write Capacity Unit — 1 WCU = 1 write per second for items up to 1 KB.

**Q23. What are the two capacity modes?**  
A: Provisioned and On-Demand.

**Q24. When use On-Demand?**  
A: Unknown or unpredictable traffic, new apps, spiky workloads.

**Q25. When use Provisioned?**  
A: Predictable traffic with cost optimization via reserved capacity and auto-scaling.

**Q26. What is a GSI?**  
A: Global Secondary Index — alternate key schema with its own partitions and throughput.

**Q27. What is an LSI?**  
A: Local Secondary Index — same partition key, different sort key; created at table creation.

**Q28. How many GSIs per table?**  
A: Up to 20.

**Q29. How many LSIs per table?**  
A: Up to 5.

**Q30. Can you add an LSI after table creation?**  
A: No. LSIs must be defined at table creation.

**Q31. Can you add a GSI after table creation?**  
A: Yes.

**Q32. What SDK is used to call DynamoDB from Python?**  
A: Boto3 (`boto3.resource('dynamodb')` or `boto3.client('dynamodb')`).

**Q33. Does DynamoDB require a VPC?**  
A: No. It is a public AWS API endpoint (VPC endpoints optional for private access).

**Q34. What is DynamoDB Streams?**  
A: Time-ordered stream of item-level modifications for event-driven processing.

**Q35. What is TTL?**  
A: Automatic item expiration based on a numeric epoch timestamp attribute.

**Q36. What is DAX?**  
A: DynamoDB Accelerator — managed in-memory cache for microsecond read latency.

**Q37. What are Global Tables?**  
A: Multi-region active-active replication of DynamoDB tables.

**Q38. Does DynamoDB support backups?**  
A: Yes — on-demand backups and Point-in-Time Recovery (PITR).

**Q39. What is PITR?**  
A: Continuous backup allowing restore to any point within the last 35 days.

**Q40. Is DynamoDB serverless?**  
A: Yes — no servers to manage; you pay for usage.

**Q41. What happens if you exceed provisioned capacity?**  
A: Requests are throttled with `ProvisionedThroughputExceededException` unless auto-scaling adds capacity.

**Q42. What is auto-scaling for DynamoDB?**  
A: Application Auto Scaling adjusts provisioned RCU/WCU based on utilization targets.

**Q43. Can DynamoDB trigger Lambda?**  
A: Yes, via DynamoDB Streams as an event source.

**Q44. What is a hot partition?**  
A: A partition receiving disproportionate traffic due to poor key design.

**Q45. What is denormalization?**  
A: Storing redundant/copy data in items to avoid joins and enable efficient reads.

**Q46. Does DynamoDB support transactions?**  
A: Yes — TransactWriteItems and TransactGetItems (up to 100 items, same region).

**Q47. What is single-table design?**  
A: Storing multiple entity types in one table using composite PK/SK patterns.

**Q48. How is data encrypted at rest in DynamoDB?**  
A: AES-256 encryption by default; optional KMS keys.

**Q49. What is the free tier for DynamoDB?**  
A: 25 GB storage, 25 RCU, 25 WCU (provisioned) — always free tier allowances apply per account.

**Q50. What metric indicates throttling in CloudWatch?**  
A: `ThrottledRequests`, `ReadThrottleEvents`, `WriteThrottleEvents`.

---

## 17.2 Intermediate Questions (51–100)

**Q51. How does DynamoDB determine which partition stores an item?**  
A: It hashes the partition key value and maps it to a partition on an internal consistent hash ring.

**Q52. What triggers a partition split?**  
A: Partition exceeding ~10 GB storage or exceeding per-partition throughput limits.

**Q53. Does splitting fix hot partitions?**  
A: No. Splits address size/throughput per hash range; hot keys need better key design.

**Q54. What is the per-partition throughput limit?**  
A: Approximately 3,000 RCU and 1,000 WCU per partition (provisioned model).

**Q55. Explain eventually consistent vs strongly consistent reads.**  
A: Eventual may return stale data from replicas; strong waits for latest write acknowledgment, costs 2× RCU.

**Q56. Can GSIs support strongly consistent reads?**  
A: No. GSI queries are eventually consistent only.

**Q57. Can LSIs support strongly consistent reads?**  
A: Yes.

**Q58. What is index projection?**  
A: Which attributes are copied to the index: ALL, KEYS_ONLY, or INCLUDE specific attributes.

**Q59. What is a sparse index?**  
A: An index containing only items that have the indexed attribute — useful for subset queries.

**Q60. How do GSIs affect write costs?**  
A: Each GSI consumes additional WCU when base table items are written/updated/deleted.

**Q61. What is `LastEvaluatedKey`?**  
A: Pagination token returned when a Query/Scan result set is truncated; pass it to get the next page.

**Q62. What is the difference between BatchGetItem and Query?**  
A: BatchGetItem fetches up to 100 specific items by full primary key across partitions; Query fetches items sharing one partition key.

**Q63. What is BatchWriteItem limit?**  
A: Up to 25 put or delete requests per call.

**Q64. How does conditional write work?**  
A: `ConditionExpression` ensures write succeeds only if condition met (e.g., `attribute_not_exists(PK)` for idempotency).

**Q65. What is optimistic locking in DynamoDB?**  
A: Use a version attribute with condition `version = :expected` on update; fails if another writer updated first.

**Q66. Explain TransactWriteItems.**  
A: Atomic all-or-nothing write of up to 100 items across up to 100 tables in same account/region.

**Q67. Transaction cost compared to normal writes?**  
A: Transactional operations consume 2× WCU/RCU.

**Q68. What is idempotency with DynamoDB?**  
A: Use conditional puts with unique request tokens to prevent duplicate processing.

**Q69. How does DynamoDB Streams ordering work?**  
A: Records ordered per partition key (shard); parallel processing across different keys.

**Q70. What is stream view type NEW_AND_OLD_IMAGES?**  
A: Stream records include both before and after item images.

**Q71. When would you use DAX vs ElastiCache?**  
A: DAX for simple DynamoDB item caching with microsecond latency; ElastiCache for custom caching logic, non-DynamoDB data.

**Q72. Does DAX support strong consistency?**  
A: DAX is eventually consistent by default; DAX strongly consistent reads pass through to DynamoDB.

**Q73. How do Global Tables handle conflicts?**  
A: Last writer wins based on timestamp for concurrent updates to same item in different regions.

**Q74. What is required to enable Global Tables?**  
A: DynamoDB Streams enabled; table must have same name and compatible keys in each region.

**Q75. Explain TTL deletion behavior.**  
A: Items deleted asynchronously after expiry; no WCU charge; may lag up to 48 hours.

**Q76. Can TTL attribute be String?**  
A: No. Must be Number type (epoch seconds).

**Q77. What is the 10 GB limit with LSIs?**  
A: Combined size of all items sharing a partition key (base + LSI data) cannot exceed 10 GB.

**Q78. How to model many-to-many in DynamoDB?**  
A: Adjacency lists, GSI inversion, or duplicate items with different keys (denormalization).

**Q79. What is write sharding?**  
A: Distributing writes across synthetic partition keys (e.g., `shard-0` to `shard-9`) to avoid hot spots.

**Q80. What is scatter-gather query pattern?**  
A: Query multiple shards/partitions in parallel and merge results in application.

**Q81. How does on-demand pricing differ from provisioned?**  
A: On-demand charges per request unit; provisioned charges for reserved RCU/WCU per hour regardless of usage.

**Q82. Can you switch capacity modes?**  
A: Yes — provisioned ↔ on-demand with console/CLI (brief adjustment period).

**Q83. What is adaptive capacity?**  
A: DynamoDB feature that temporarily reassigns unused throughput to hot partitions (provisioned mode).

**Q84. Does Scan parallelize?**  
A: Yes — Scan uses segments (`TotalSegments`, `Segment`) for parallel workers.

**Q85. Why avoid Scan in production APIs?**  
A: Reads entire table, costly, latency grows with table size, competes with production traffic.

**Q86. What is fine-grained access control?**  
A: IAM conditions restricting access to specific partition key values for multi-tenant security.

**Q87. What is a VPC gateway endpoint for DynamoDB?**  
A: Route table entry allowing private VPC access to DynamoDB without internet/NAT.

**Q88. How does deletion protection work?**  
A: Table setting preventing accidental `DeleteTable` API calls.

**Q89. What is a composite primary key?**  
A: Partition key + sort key together uniquely identify an item.

**Q90. Explain `begins_with` on sort key.**  
A: Query condition matching sort keys with a given prefix — useful for hierarchical SK design.

**Q91. What is the difference between PutItem and UpdateItem?**  
A: PutItem replaces entire item; UpdateItem modifies specific attributes with optional atomic operators (`ADD`, `SET`, `REMOVE`).

**Q92. Can UpdateItem create an item if it doesn't exist?**  
A: Yes, with `upsert` behavior unless `attribute_exists` condition prevents it.

**Q93. What are DynamoDB metrics to monitor?**  
A: ConsumedReadCapacityUnits, ConsumedWriteCapacityUnits, ThrottledRequests, SuccessfulRequestLatency, UserErrors.

**Q94. How to export DynamoDB data to S3?**  
A: DynamoDB export to S3 (PITR required) or Scan-based custom export.

**Q95. What is import from S3?**  
A: Bulk import feature to create table and load data from S3 in DynamoDB JSON or ION format.

**Q96. Explain capacity unit rounding for reads.**  
A: Round up item size to next 4 KB boundary; multiply by 0.5 or 1 for consistency model.

**Q97. Explain capacity unit rounding for writes.**  
A: Round up item size to next 1 KB boundary per item.

**Q98. What is a DynamoDB local secondary index use case?**  
A: Same entity, multiple sort orders (e.g., orders by date vs by amount for same user).

**Q99. What is an inverted index pattern?**  
A: GSI that swaps PK and SK from base table to reverse access direction.

**Q100. How does DynamoDB integrate with AppSync?**  
A: AppSync can use DynamoDB as data source for GraphQL resolvers with direct mapping templates.

---

## 17.3 Advanced Questions (101–150)

**Q101. Explain DynamoDB's internal consistent hashing.**  
A: Partition keys are hashed onto a ring; partitions own hash ranges; when splits occur, ranges subdivide without changing the logical API.

**Q102. Why can't you query by non-key attributes efficiently?**  
A: Data is physically organized by partition key hash; without an index, only Scan can find arbitrary attributes.

**Q103. Design keys for a leaderboard with millions of players.**  
A: `PK=GAME#id, SK=SCORE#<inverted_score>#PLAYER#id` for sorted ranking; avoid single PK for all scores.

**Q104. How would you implement pagination for a Query returning 1M items?**  
A: Use `Limit`, loop on `LastEvaluatedKey`, optionally expose opaque cursor to clients; never Scan.

**Q105. Design multi-tenant SaaS isolation in DynamoDB.**  
A: Prefix PK with `TENANT#id`; IAM fine-grained access on `dynamodb:LeadingKeys`; avoid cross-tenant hot keys.

**Q106. Compare DynamoDB vs Aurora for serverless apps.**  
A: DynamoDB: key-value scale, no SQL joins; Aurora Serverless: SQL, connections, complex queries — choose by access pattern.

**Q107. How do you migrate from RDS to DynamoDB?**  
A: Dual-write, DMS export, batch import, redesign schema for access patterns, gradual cutover with validation.

**Q108. Explain the 10 GSIs limit workaround.**  
A: Combine access patterns via composite keys, overloaded GSIs (multiple entity types per index), or duplicate data streams to OpenSearch.

**Q109. What is an overloaded GSI?**  
A: One GSI serves multiple access patterns using typed PK/SK conventions like `GSI1PK=EMAIL#x`.

**Q110. How does DynamoDB handle node failure?**  
A: Replica promotion within AZ set; routing table updated; client retries with exponential backoff.

**Q111. Explain clock skew impact on Global Tables.**  
A: Conflict resolution uses timestamps; significant skew can cause unexpected last-writer-wins outcomes.

**Q112. Design session store for 10M concurrent users.**  
A: `PK=SESSION#uuid`, TTL attribute, on-demand capacity, optional DAX for read-heavy validation.

**Q113. How to implement counters at scale?**  
A: `UpdateItem ADD count :1` is atomic; for very hot counters, shard counter across keys and sum on read.

**Q114. Explain hot counter sharding.**  
A: `PK=COUNTER#shard-{0..N}, SK=views` — increment random shard; total = sum of all shards.

**Q115. What is the maximum transaction size?**  
A: 100 items, 4 MB total transaction payload, items must fit per-item 400 KB limit.

**Q116. Can transactions span GSI and base table atomically?**  
A: TransactWrite can include base table items; GSI updates are asynchronous side effects of those writes.

**Q117. Explain double-write problem with GSIs.**  
A: GSI propagation is eventual; reading GSI immediately after write may not show new item.

**Q118. How to achieve strongly consistent GSI-like reads?**  
A: Read from base table using primary key, or design LSI with same partition key; GSIs cannot be strong.

**Q119. Design IoT time-series in DynamoDB.**  
A: `PK=DEVICE#id, SK=TIMESTAMP#iso` with TTL for retention; avoid PK=DATE (hot partition).

**Q120. When would DynamoDB be wrong choice?**  
A: Complex ad-hoc analytics, heavy relational reporting, unconstrained query patterns, tiny fixed workloads cheaper on SQLite.

**Q121. Explain PartiQL `SELECT` vs Query API.**  
A: PartiQL SELECT without key condition may use Scan underneath — understand cost implications.

**Q122. How does export to S3 work internally?**  
A: PITR-based incremental export without consuming table RCU; writes to S3 as DynamoDB JSON.

**Q123. Design erasure (GDPR) in DynamoDB.**  
A: DeleteItem on known keys; GSI cleanup eventual; for Global Tables, delete in all regions; audit via Streams.

**Q124. Explain connection-less advantage for Lambda.**  
A: No connection pool exhaustion; each invocation uses HTTP API; scales with concurrency without RDS proxy.

**Q125. How to reduce item size for cost?**  
A: Short attribute names, compress large blobs to S3 with pointer, avoid redundant data.

**Q126. Design graph relationships in DynamoDB.**  
A: Adjacency lists, materialized paths, or integrate with Neptune for heavy graph traversals.

**Q127. What is the difference between DynamoDB and DocumentDB?**  
A: DynamoDB is key-value/document with fixed access patterns; DocumentDB is MongoDB-compatible with flexible queries and higher latency.

**Q128. Explain RCU consumption for BatchGetItem 100 items of 4 KB.**  
A: Each item = 1 RCU (strong) or 0.5 (eventual); total 50-100 RCU for one batch depending on consistency.

**Q129. How does DynamoDB Streams Lambda batching work?**  
A: Lambda polls shards, batches records (configurable batch size, parallelization factor), invokes function with retry and DLQ on failure.

**Q130. Design idempotent payment processing.**  
A: `PK=PAYMENT#id` conditional put `attribute_not_exists(PK)`; TransactWrite with balance check condition.

**Q131. Explain why monotonic sort keys help.**  
A: Time-UUID or timestamp SK enables efficient `Query` with `ScanIndexForward=false` for latest-first.

**Q132. How to benchmark DynamoDB?**  
A: Use distributed load test (Artillery, custom), ramp WCU/RCU, watch ThrottledRequests and p99 latency in CloudWatch.

**Q133. What is return capacity in API responses?**  
A: `ReturnConsumedCapacity` shows RCU/WCU used per operation for tuning.

**Q134. Design search functionality with DynamoDB.**  
A: DynamoDB for key-based lookups; Streams → OpenSearch for full-text search.

**Q135. Explain zonal isolation in DynamoDB storage.**  
A: Three replicas in separate AZs; write quorum before ack for durability.

**Q136. Can you change partition key after table creation?**  
A: No. Must create new table and migrate data.

**Q137. How does ImportTable differ from BatchWriteItem migration?**  
A: ImportTable is managed bulk load from S3 without consuming WCU; faster for initial loads.

**Q138. Design rate limiter with DynamoDB.**  
A: `PK=RATE#user#window` with TTL; atomic increment with condition on count < limit.

**Q139. Explain exponential backoff on throttling.**  
A: SDK retries with increasing delay (2^n × base); essential for burst traffic on provisioned tables.

**Q140. What is the impact of large items on performance?**  
A: More RCU/WCU per operation, slower transfers, higher cost — keep items lean.

**Q141. Design CQRS with DynamoDB.**  
A: Command writes to primary table; Streams project to read-optimized views/GSIs or separate store.

**Q142. How do you handle duplicate Stream events?**  
A: Design idempotent consumers using event ID or business key conditional writes.

**Q143. Compare on-demand vs provisioned at 70% steady utilization.**  
A: Provisioned with auto-scaling typically cheaper; on-demand premium for flexibility.

**Q144. Explain DynamoDB integration with Step Functions.**  
A: SDK integrations for GetItem, PutItem, UpdateItem, DeleteItem without Lambda wrapper.

**Q145. Design soft delete pattern.**  
A: `isDeleted=true` attribute; sparse GSI excluding deleted items; TTL for permanent purge.

**Q146. What happens during table deletion with PITR?**  
A: Table removed; PITR backups retained until retention expires; can restore deleted table within window.

**Q147. Explain attribute name expressions.**  
A: `ExpressionAttributeNames` (`#n`) escape reserved words like `Status`, `Name`, `Date`.

**Q148. How to test DynamoDB locally?**  
A: DynamoDB Local (JAR/Docker), LocalStack, or sam local with `--docker-network`.

**Q149. Design Amazon Shopping cart with DynamoDB concepts.**  
A: `PK=CUSTOMER#id, SK=CART#itemId`; conditional updates for inventory; TTL for abandoned carts; high cardinality PK.

**Q150. Summarize DynamoDB in one architectural sentence.**  
A: A horizontally sharded, multi-AZ replicated, key-addressed document store where performance is determined entirely by partition key and access pattern design, not query optimizer magic.

---

# Section 18: System Design Interview Perspective

## 18.1 Amazon Shopping

```
Components using DynamoDB:
  - Shopping cart (PK=customer, items as sort keys)
  - Session state
  - Product catalog (with ElastiCache/DAX)
  - Inventory counters (sharded writes)

Scale techniques:
  - Millions of partition keys (customers)
  - On-demand during Prime Day
  - Global Tables for regional catalogs
  - Streams → analytics pipeline
```

**Discussion points:** Hot inventory SKU → sharded counters; cart merge on login → TransactWrite; eventual GSI for product search elsewhere.

---

## 18.2 Netflix

```
Use cases:
  - Viewing history (PK=profileId, SK=timestamp)
  - Preferences, bookmarks
  - A/B test assignments

Why DynamoDB:
  - Massive read/write scale globally
  - Low latency for personalization API
  - Regional replication for latency
```

**Partitioning:** Profile ID ensures even distribution. **Replication:** Multi-region for disaster recovery. **Scalability:** On-demand handles premiere spikes.

---

## 18.3 Gaming Systems

```
Leaderboards:
  PK=GAME#id, SK=SCORE#<inverted>#PLAYER#id

Player state:
  PK=PLAYER#id, SK=STATE

Matchmaking:
  PK=QUEUE#rank-bracket, SK=timestamp
```

**Challenges:** Hot game leaderboard → write sharding or approximate ranking (Redis/DAX hybrid). **Latency:** DAX for frequent profile reads.

---

## 18.4 IoT Systems

```
Device telemetry:
  PK=DEVICE#uuid, SK=TS#2024-06-21T10:00:00Z
  TTL=90 days

Device shadow:
  PK=THING#id, SK=SHADOW
```

**Scale:** Millions of devices = millions of partition keys (good). **Pitfall:** Never use `PK=DATE#2024-06-21` — all devices write to one partition.

---

## 18.5 System Design Framework for DynamoDB

```
1. Requirements → read/write QPS, latency, consistency, regions
2. Access patterns → list every query type
3. Key design → PK/SK per pattern
4. Indexes → GSI/LSI only for proven patterns
5. Capacity → on-demand vs provisioned
6. Resilience → PITR, Global Tables, backups
7. Observability → CloudWatch alarms on throttles
8. Security → IAM least privilege, encryption, VPC endpoints
```

---

# Section 19: Common Mistakes

| # | Mistake | Consequence |
|---|---------|-------------|
| 1 | Using Scan in API hot path | High cost, slow responses, throttling |
| 2 | Low-cardinality partition key | Hot partitions, throttling |
| 3 | Ignoring item size (400 KB limit) | Write failures, high RCU/WCU |
| 4 | No pagination on large Queries | Lambda timeout, memory exhaustion |
| 5 | Treating GSI as strongly consistent | Stale reads, bugs after write |
| 6 | Too many GSIs | 2×+ write costs, complexity |
| 7 | LSI after table creation attempt | Must recreate table |
| 8 | Not enabling PITR | No point-in-time recovery |
| 9 | Not enabling deletion protection | Accidental table delete |
| 10 | Hardcoding RCU/WCU too low | Production throttling |
| 11 | No auto-scaling on provisioned | Manual capacity crises |
| 12 | Using SQL mental model (normalization) | Slow queries, multiple round trips |
| 13 | FilterExpression instead of key condition | Reads all items in partition, filters after — costly |
| 14 | Monolithic partition key for time-series | Hot partition per day |
| 15 | Not using conditional writes | Lost updates, duplicate records |
| 16 | Ignoring exponential backoff | Retry storms during throttling |
| 17 | Storing large blobs in items | Cost, latency; use S3 references |
| 18 | Wrong TTL attribute type (String) | TTL never deletes |
| 19 | Expecting immediate TTL deletion | Items linger post-expiry |
| 20 | Cross-region transactions | Not supported — design around it |
| 21 | Transaction > 100 items | API failure |
| 22 | Not monitoring ThrottledRequests | Silent user-facing errors |
| 23 | DAX for write-heavy fresh data | Cache pollution, stale reads |
| 24 | Global Tables without conflict strategy | Data overwrites |
| 25 | IAM policy too permissive (`dynamodb:*`) | Security breach risk |
| 26 | No VPC endpoint in private subnet + NAT costs | Higher cost, latency |
| 27 | Attribute names matching reserved words without `#` alias | Expression syntax errors |
| 28 | Composite key design without range queries | Wasted sort key |
| 29 | Single tenant key dominating traffic | One tenant throttles all |
| 30 | Changing partition key in place | Impossible — requires migration |

---

## 19.1 Deep Dive on Top 5 Mistakes

### Mistake #1: Scan in Production

Developers coming from SQL run `scan()` with `FilterExpression` thinking it is like `WHERE`.

```
Cost on 10 GB table, 1M items, 1% match filter:
  Scan reads ALL 1M items → ~250,000 RCU (4KB items)
  Query with GSI on indexed attribute → ~10,000 RCU

250× more expensive, 100× slower
```

### Mistake #2: FilterExpression Misunderstanding

```
Query PK=USER#101 returns 1000 orders (all read, all billed)
FilterExpression: Total > 100
  → Returns 50 items but you paid RCU for 1000 reads
```

**Fix:** Use sort key or GSI key condition to narrow at key level, not filter after.

### Mistake #3: Ignoring GSI Propagation Delay

```
1. PutItem to base table
2. Immediate Query on GSI
3. Item not found → user sees bug

Fix: Read from base table by primary key after write, or design for eventual consistency
```

### Mistake #4: Not Using TTL for Ephemeral Data

```
Developers manually delete expired sessions with scheduled Lambda Scan
  → Expensive, slow, error-prone

Fix: TTL attribute — free automatic deletion
```

### Mistake #5: Right-Sizing for Peak Without Auto-Scaling

```
Black Friday: 10× traffic → throttling → lost revenue
Fix: On-demand OR provisioned + auto-scaling with sensible min/max
```

---

# Section 20.5: DynamoDB Limits and Quotas Reference

| Resource | Limit |
|----------|-------|
| Max item size | 400 KB |
| Max attribute name length | 64 KB |
| Max partition key + sort key length | 2048 bytes combined |
| Max GSIs per table | 20 |
| Max LSIs per table | 5 |
| Max tables per region | 2500 (soft, increase via support) |
| BatchGetItem items | 100 per request |
| BatchWriteItem items | 25 per request |
| TransactWrite items | 100 per transaction |
| Transaction payload | 4 MB |
| Query/Scan result page | 1 MB per response |
| DynamoDB Streams shards | Scales with throughput |
| Global Tables regions | Up to 20 regions |
| PITR retention | 35 days |
| On-demand max table size | No practical limit |
| Account throughput (on-demand) | 40,000 RCU + 40,000 WCU per table (default soft) |

---

# Section 20: Cheat Sheet

## Core Concepts

```
Table → Items → Attributes
Primary Key = Partition Key [+ Sort Key]
Partition Key → hash → Partition (physical storage)
Max item: 400 KB | Max LSIs: 5 | Max GSIs: 20
```

## Operations

```
GetItem    → 1 item, full primary key, 1 partition
Query      → many items, same partition key
Scan       → entire table/index (AVOID in prod)
BatchGet   → up to 100 items by key
BatchWrite → up to 25 put/delete
Transact   → up to 100 items, atomic, 2× capacity cost
```

## Consistency

```
Default read:     Eventually consistent (0.5 RCU / 4KB)
Strong read:      ConsistentRead=true (1 RCU / 4KB)
GSI:              Eventually consistent ONLY
LSI:              Strong or eventual
```

## Capacity

```
RCU (eventual): 0.5 per 4 KB/sec
RCU (strong):   1 per 4 KB/sec
WCU:            1 per 1 KB/sec
Modes:          Provisioned | On-Demand
Per partition:  ~3000 RCU, ~1000 WCU
```

## Indexes

```
GSI: different PK, own throughput, add anytime, eventual only
LSI: same PK, different SK, table create only, strong OK
```

## Advanced Features

```
Streams     → change data capture → Lambda
TTL         → auto-delete (Number epoch attribute)
DAX         → microsecond read cache
Global Tables → multi-region active-active
PITR        → 35-day continuous backup
Transactions → ACID, 100 items max, same region
```

## Design Rules

```
1. Access patterns FIRST
2. High-cardinality partition keys
3. Query > Scan
4. Denormalize for reads
5. Single-table design for related entities
6. Avoid hot partitions (shard writes)
7. Project minimal GSI attributes
8. Enable PITR + deletion protection (prod)
9. Use conditional writes for idempotency
10. Monitor throttles and consumed capacity
```

## Request Flow

```
Client → IAM Auth → DynamoDB Router → Partition Leader → Storage + Replicas
```

## When to Use DynamoDB

```
YES: Serverless, massive scale, key-based access, AWS-native, variable traffic
NO:  Complex SQL analytics, ad-hoc queries, heavy joins, small static SQL workload
```

## Quick Architecture

```
User → API Gateway → Lambda → DynamoDB
                              ↓ Streams
                            Lambda → SNS/ES/audit
```

## Boto3 Snippets

```python
# GetItem
table.get_item(Key={'CustomerID': '101'}, ConsistentRead=True)

# Query
table.query(
    KeyConditionExpression=Key('UserID').eq('u1') & Key('OrderDate').gte('2024-01-01')
)

# Conditional Put
table.put_item(
    Item={'CustomerID': '101', 'Name': 'John'},
    ConditionExpression='attribute_not_exists(CustomerID)'
)

# TransactWrite
client.transact_write_items(TransactItems=[...])
```

## Local Development with DynamoDB Local

```
┌──────────────┐     ┌─────────────────┐
│ Your Laptop  │────►│ DynamoDB Local  │
│ Boto3 code   │     │ (Docker/JAR)    │
│ endpoint_url │     │ localhost:8000  │
└──────────────┘     └─────────────────┘
```

```bash
docker run -p 8000:8000 amazon/dynamodb-local
```

```python
dynamodb = boto3.resource(
    'dynamodb',
    endpoint_url='http://localhost:8000',
    region_name='us-east-1',
    aws_access_key_id='dummy',
    aws_secret_access_key='dummy'
)
```

**Limitations:** No TTL, Streams, DAX, or Global Tables. Use for unit tests and prototyping only.

## Exam Quick-Fire Facts (SAA/DVA)

| Fact | Answer |
|------|--------|
| Fastest read operation | GetItem by full primary key |
| Cheapest read | Eventually consistent (half RCU) |
| Index with strong reads | LSI only |
| Index with different partition key | GSI |
| Max item size | 400 KB |
| Serverless cache | DAX |
| Multi-region replication | Global Tables |
| Change data capture | DynamoDB Streams |
| Auto-delete expired items | TTL |
| Avoid in hot path | Scan |
| Hot partition fix | Better partition key / write sharding |

## Migration Checklist

```
□ Document all access patterns
□ Design PK/SK and GSIs
□ Estimate RCU/WCU or choose on-demand
□ Enable encryption, PITR, deletion protection
□ IAM least privilege
□ Load test with throttling alarms
□ Plan GSI eventual consistency in app logic
```

---

## Recommended Learning Path

```
Week 1: Sections 1-6  (concepts, partitioning, architecture)
Week 2: Sections 7-11 (console, CRUD, indexes, capacity)
Week 3: Sections 12-16 (advanced features, modeling, security)
Week 4: Sections 17-20 (interviews, mistakes, cheat sheet review)
```

---

*Guide version: 2026 · Covers DynamoDB fundamentals through production architecture, partitioning deep dive, and 150 interview Q&A.*

**Document stats:** 20 sections · 150 interview questions with answers · ASCII architecture diagrams · AWS Console UI walkthroughs · Boto3 code examples · Single-table design patterns · Production checklist.
