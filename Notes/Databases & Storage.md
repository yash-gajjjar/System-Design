# Databases & Storage — Complete Deep-Dive Revision Guide
## System Design Interview Preparation | Product-Based Companies

---

**Prepared for:** Yash | AI/GenAI Engineer transitioning to Product Company System Design Interviews
**Coverage:** SQL vs NoSQL · CAP Theorem · Database Sharding · Replication · Indexing · ACID Properties · NoSQL Types · Database Normalization · Object Storage · Time-series DBs · Data Warehousing

---

## How This Guide Is Organized

Each topic follows the same structure as the Networking Fundamentals and
Scalability & Load Balancing guides:
1. **What Problem Does It Solve?** — the "why" before the "what"
2. **Core Intuition** — beginner-friendly explanations with analogies
3. **Step-by-step Diagrams** — ASCII diagrams of every concept and structure
4. **Deep Dive** — internals, algorithms, data structures, mechanisms
5. **Comparison Tables** — tradeoffs for every design decision
6. **Real-World Usage** — how Google, Netflix, Amazon, Stripe, Instagram etc. actually use these
7. **Failure Scenarios** — what breaks in production and how to fix it
8. **Interview Quick-Fire Answers** — ready-to-use answers for common questions

---

## Table of Contents

1. **SQL vs NoSQL** — relational model, JOINs, polyglot persistence, decision framework
2. **CAP Theorem** — consistency vs availability during partitions, CP vs AP, PACELC extension
3. **Database Sharding** — shard key selection, range/hash/directory strategies, cross-shard challenges
4. **Replication** — sync vs async, multi-leader, leaderless/quorum, failover
5. **Indexing** — B+ Trees, composite indexes, leftmost-prefix rule, covering indexes
6. **ACID Properties** — atomicity, consistency, isolation levels, MVCC, durability
7. **NoSQL Types** — key-value, document, column-family, graph databases
8. **Database Normalization** — 1NF/2NF/3NF, normalization vs denormalization tradeoffs
9. **Object Storage** — durability via erasure coding, consistency, storage tiering
10. **Time-series DBs** — column-oriented compression, time-based chunking, downsampling
11. **Data Warehousing** — OLAP vs OLTP, column-oriented storage, ETL/ELT, CDC, star schema
12. **Appendix** — cross-topic quick reference, complete data layer design flow, study tips

---

# Databases & Storage — Deep-Dive System Design Notes
### For Product-Based Company Interviews | Beginner → Advanced

---

> **How to use these notes:** Same structure as the Networking Fundamentals and
> Scalability & Load Balancing guides.
> What is it → Why does it exist → How it works step by step → Diagrams → Internals
> → Tradeoffs → Real-world → Failures → Interview tips.
> Every concept is explained from scratch — no prior database internals knowledge assumed.

---

# TOPIC 1: SQL vs NoSQL

---

## 1. What Problem Does This Choice Solve?

Every application needs to store data persistently. For decades, "database" essentially meant **relational database (SQL)** — Oracle, MySQL, PostgreSQL, SQL Server. From the early 2000s onward, as companies like Google, Amazon, and Facebook hit scale and flexibility limits with relational databases, a wave of alternative databases emerged, collectively called **NoSQL** ("Not Only SQL").

The question "SQL or NoSQL?" is really asking: **"What shape is my data, how will I query it, how consistent does it need to be, and how much will it grow?"** Getting this choice wrong early is one of the most expensive mistakes in system design — migrating from one data model to another after launch is enormously costly.

---

## 2. SQL (Relational Databases) — Deep Dive

### Core Intuition — Tables, Rows, and Relationships

A relational database organizes data into **tables** (like spreadsheets), where each row is a record and each column is an attribute. The "relational" part comes from **relationships between tables**, defined via **foreign keys**.

```
TABLE: users                          TABLE: orders
┌────┬─────────┬──────────────┐       ┌────┬─────────┬────────┬──────────┐
│ id │ name    │ email         │       │ id │ user_id │ total  │ status   │
├────┼─────────┼──────────────┤       ├────┼─────────┼────────┼──────────┤
│ 1  │ Yash    │ yash@mail.com │       │ 101│ 1       │ 4999   │ shipped  │
│ 2  │ Priya   │ priya@mail.com│       │ 102│ 1       │ 1299   │ pending  │
│ 3  │ Rahul   │ rahul@mail.com│       │ 103│ 2       │ 7500   │ delivered│
└────┴─────────┴──────────────┘       └────┴─────────┴────────┴──────────┘
       ▲                                          │
       └──────────────────────────────────────────┘
         orders.user_id is a FOREIGN KEY referencing users.id
         "Order 101 belongs to user 1 (Yash)"
```

### The Schema — Defined Upfront, Strictly Enforced

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    total DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

WHAT "STRICT SCHEMA" MEANS:
- Every row in `users` MUST have a name, email, id (NOT NULL constraints)
- email MUST be unique across the entire table (UNIQUE constraint)
- Every order MUST reference an EXISTING user (FOREIGN KEY constraint)
  → You CANNOT insert an order with user_id=999 if user 999 doesn't exist
- Trying to insert data that violates these rules → the DATABASE
  REJECTS the write with an error

THIS IS A FEATURE, NOT A LIMITATION: the database itself guarantees
data integrity — your application code doesn't have to remember to
check "does this user exist?" before every order insert. The
database enforces it, ALWAYS, for every write, from any source.
```

### SQL Query Language — Joins, the Core Superpower

```sql
-- "Get all orders for Yash, with his name"
SELECT users.name, orders.id, orders.total, orders.status
FROM users
JOIN orders ON orders.user_id = users.id
WHERE users.name = 'Yash';

RESULT:
┌──────┬─────┬────────┬──────────┐
│ name │ id  │ total  │ status   │
├──────┼─────┼────────┼──────────┤
│ Yash │ 101 │ 4999   │ shipped  │
│ Yash │ 102 │ 1299   │ pending  │
└──────┴─────┴────────┴──────────┘

JOINS let you combine data from MULTIPLE tables in a single query,
based on the relationships between them. This is the relational
model's core strength: data is normalized (stored once, no
duplication — covered in depth in the Normalization topic), and
JOINs reassemble it on demand.
```

### Why SQL Databases Are Hard to Scale Horizontally

```
THE FUNDAMENTAL TENSION:

JOINs require related data to be queryable TOGETHER. If `users`
lives on Server A and `orders` lives on Server B, a JOIN now
requires a NETWORK CALL between servers for EVERY query — slow,
and breaks the atomic guarantees the database normally provides.

If you SHARD a relational database (split `users` table itself
across multiple servers by user_id range, e.g., users 1-1M on
Server A, users 1M-2M on Server B):

- A query like "find user by email" now might need to check
  EVERY shard (unless email happens to align with your sharding key)
- A JOIN between `orders` (sharded by user_id) and `products`
  (not naturally shardable by user_id) becomes a CROSS-SHARD JOIN
  — extremely expensive, often requires the application to do the
  join itself (fetch from two shards, combine in app code)
- Transactions spanning multiple shards (e.g., "transfer money
  from user A's account to user B's account, where A and B are
  on different shards") require DISTRIBUTED TRANSACTIONS (2-phase
  commit) — slow and complex

THIS IS WHY: relational databases are traditionally scaled
VERTICALLY first (bigger machine — recall Topic 1 of Scalability
notes), then via READ REPLICAS (Replication topic) for read scaling,
and ONLY as a last resort via SHARDING (Database Sharding topic) —
each step trading away some of SQL's core conveniences.
```

---

## 3. NoSQL (Non-Relational Databases) — Deep Dive

NoSQL isn't ONE thing — it's an umbrella term for several DIFFERENT data models, each suited to different problems (deep dive on each in the "NoSQL Types" topic). But they share common philosophical threads:

```
CORE PHILOSOPHY SHIFT:

SQL:    "Define your schema rigorously upfront. Normalize data to
         avoid duplication. Use JOINs to assemble related data at
         query time. Prioritize CONSISTENCY and data integrity."

NoSQL:  "Design your data model around your ACCESS PATTERNS (how
         you'll QUERY the data), even if that means duplicating
         data. Avoid JOINs — store related data TOGETHER so a
         single lookup gets everything you need. Prioritize
         SCALABILITY and FLEXIBILITY, sometimes trading away strict
         consistency."
```

### Example: The Same Data, SQL vs NoSQL (Document Model)

```
SQL (normalized, 2 tables, requires a JOIN):

users table:                     orders table:
┌────┬───────┬───────────────┐   ┌─────┬─────────┬────────┐
│ id │ name  │ email          │   │ id  │ user_id │ total  │
├────┼───────┼───────────────┤   ├─────┼─────────┼────────┤
│ 1  │ Yash  │ yash@mail.com  │   │ 101 │ 1       │ 4999   │
└────┴───────┴───────────────┘   └─────┴─────────┴────────┘

NoSQL (document model — MongoDB-style, denormalized):

{
  "_id": 1,
  "name": "Yash",
  "email": "yash@mail.com",
  "orders": [
    { "id": 101, "total": 4999, "status": "shipped" },
    { "id": 102, "total": 1299, "status": "pending" }
  ]
}

ONE document contains EVERYTHING about Yash AND his orders.
Fetching "Yash's profile with his recent orders" = ONE lookup
by _id, ONE disk read (or cache hit) — NO JOIN NEEDED.

TRADEOFF: If Yash's email changes, it's stored in ONE place (fine).
But if "order status" needs to be queried independently across ALL
users (e.g., "find all pending orders across the entire system"),
this document structure is AWKWARD — you'd need to scan every
user's document. A separate `orders` collection might ALSO be
needed, duplicating order data — this is DENORMALIZATION, a
deliberate tradeoff (data duplication for query speed) covered
fully in the Normalization topic.
```

---

## 4. The Complete SQL vs NoSQL Comparison

```
┌──────────────────────────┬─────────────────────────────────┬─────────────────────────────────┐
│ Dimension                 │ SQL (Relational)                 │ NoSQL                            │
├──────────────────────────┼─────────────────────────────────┼─────────────────────────────────┤
│ Data model                │ Tables, rows, columns, fixed      │ Varies: documents (JSON), key-   │
│                          │ schema                            │ value, columnar, graph (see      │
│                          │                                   │ NoSQL Types topic)                │
│ Schema                    │ Rigid, enforced at write time     │ Flexible/dynamic — different      │
│                          │ (schema-on-write)                  │ documents can have different       │
│                          │                                   │ fields (schema-on-read)            │
│ Relationships             │ JOINs across normalized tables    │ Typically denormalized — related  │
│                          │                                   │ data embedded together             │
│ Query language            │ SQL (standardized, powerful,      │ Database-specific APIs/query       │
│                          │ declarative)                       │ languages (MongoDB query syntax,   │
│                          │                                   │ DynamoDB API, Cassandra CQL)       │
│ Consistency               │ Strong consistency, ACID          │ Often eventual consistency         │
│                          │ transactions (full topic ahead)   │ (tunable in many systems)          │
│ Scaling                   │ Vertical first; horizontal        │ Designed for horizontal scaling    │
│                          │ (sharding) is hard and complex     │ from the ground up                 │
│ Best for                  │ Complex relationships, multi-row   │ Massive scale, flexible/evolving   │
│                          │ transactions, reporting/analytics  │ schemas, simple access patterns   │
│                          │ (financial systems, inventory,     │ at huge volume (user sessions,    │
│                          │ order management)                  │ activity feeds, IoT data, catalogs)│
│ Examples                  │ PostgreSQL, MySQL, SQL Server,     │ MongoDB, Cassandra, DynamoDB,      │
│                          │ Oracle                            │ Redis, Neo4j, Elasticsearch        │
└──────────────────────────┴─────────────────────────────────┴─────────────────────────────────┘
```

---

## 5. The Decision Framework

```
ASK THESE QUESTIONS IN ORDER:

1. DO YOU NEED MULTI-ROW TRANSACTIONS WITH STRONG CONSISTENCY?
   "Transfer ₹5000 from Account A to Account B — both updates must
   succeed or both must fail, with no intermediate state visible
   to other transactions."
   → YES → Strong lean toward SQL (ACID transactions are
     first-class; NoSQL support varies and is often more limited
     or requires careful design)

2. IS YOUR DATA HIGHLY RELATIONAL with many many-to-many
   relationships you'll query in flexible, ad-hoc ways?
   "Find all users who bought Product X AND live in Mumbai AND
   have a 'premium' subscription."
   → YES → SQL (JOINs and ad-hoc querying are its strength)

3. WILL YOUR SCHEMA EVOLVE RAPIDLY, or do different records
   naturally have different shapes?
   "Different product types have wildly different attributes —
   a book has 'author/ISBN', a laptop has 'CPU/RAM', a t-shirt has
   'size/color'."
   → YES → NoSQL document model handles this naturally; SQL would
     need either a huge table with many NULL columns, or complex
     EAV (Entity-Attribute-Value) patterns

4. DO YOU NEED TO SCALE WRITES TO MASSIVE VOLUME ACROSS MANY
   SERVERS, with relatively SIMPLE access patterns (lookup by key)?
   "Store user session data for 500 million users, accessed by
   session ID."
   → YES → NoSQL (key-value/document stores designed for this
     from day one — see Consistent Hashing from Scalability notes)

5. DO YOU NEED POWERFUL ANALYTICS / REPORTING across your data
   ("how many orders per region per month, broken down by product
   category")?
   → SQL (and its descendant, data warehouses — covered later)
     are generally far better suited; many NoSQL systems require
     ETL-ing data OUT to an analytical system anyway
```

---

## 6. Real-World Usage — Hybrid Architectures (The Common Reality)

Most large systems use BOTH — different databases for different parts of the system. This is called **polyglot persistence**.

```
EXAMPLE: An E-commerce Platform

┌─────────────────────────────────────────────────────────────────┐
│ ORDERS / PAYMENTS / INVENTORY                                     │
│ → PostgreSQL/MySQL (SQL)                                          │
│ Why: Multi-row transactions (place order = decrement inventory   │
│ + create order + charge payment, all atomic), financial accuracy │
│ requires ACID guarantees                                          │
├─────────────────────────────────────────────────────────────────┤
│ PRODUCT CATALOG                                                   │
│ → MongoDB or similar document store (NoSQL)                      │
│ Why: Different product categories have wildly different attribute│
│ sets; catalog is read-heavy, eventually-consistent updates fine  │
├─────────────────────────────────────────────────────────────────┤
│ USER SESSIONS / SHOPPING CART                                     │
│ → Redis (NoSQL, key-value)                                        │
│ Why: Extremely high read/write volume, simple key lookups,        │
│ ephemeral data (sessions expire), low-latency requirement         │
├─────────────────────────────────────────────────────────────────┤
│ PRODUCT SEARCH                                                    │
│ → Elasticsearch (NoSQL, search-optimized)                         │
│ Why: Full-text search, faceted filtering — not a SQL strength     │
├─────────────────────────────────────────────────────────────────┤
│ ANALYTICS / REPORTING (sales trends, dashboards)                  │
│ → Data Warehouse (Snowflake/BigQuery/Redshift)                    │
│ Why: Complex aggregations across billions of rows, optimized      │
│ for read-heavy analytical queries (full topic ahead)              │
└─────────────────────────────────────────────────────────────────┘

NO SINGLE DATABASE IS "BEST" — the question is always "best FOR
WHAT PART of the system, given ITS access patterns."
```

**Amazon:** Famously moved away from a monolithic Oracle database to a polyglot architecture — DynamoDB (NoSQL) for the shopping cart and session data (massive scale, simple access patterns), Aurora/RDS (SQL) for order/financial systems requiring transactions, and various specialized stores for search, recommendations, etc.

**Uber:** Uses MySQL/PostgreSQL (via their Schemaless layer built on MySQL) for core trip/payment data requiring consistency, Cassandra for high-write-volume data like location updates and trip events, and Redis for caching and real-time data.

**Instagram:** Uses PostgreSQL (sharded) for core data (users, photos metadata, relationships), Cassandra for high-volume data like activity feeds and direct messages, and Redis extensively for caching.

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Choosing NoSQL for financial    │ Eventual consistency means a     │ Use SQL (or a NoSQL system with   │
│ transactions → inconsistent     │ balance check right after a      │ strong consistency options, e.g., │
│ account balances                │ transfer might show stale data   │ DynamoDB's strongly consistent     │
│                                  │                                  │ reads, or Spanner) for financial   │
│                                  │                                  │ data — consistency > raw scale     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Choosing SQL for a rapidly       │ Every new product attribute       │ Either use a NoSQL document store, │
│ evolving product catalog →       │ requires a schema migration       │ or a hybrid: SQL for core fields,  │
│ constant painful migrations      │ (ALTER TABLE) on a huge table     │ JSONB column (PostgreSQL) for      │
│                                  │                                  │ flexible/varying attributes        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Denormalized NoSQL data becomes  │ Same data duplicated across many │ Design update patterns carefully;  │
│ inconsistent after partial       │ documents; an update to one       │ accept eventual consistency for    │
│ updates                         │ "copy" doesn't propagate to all   │ non-critical duplicated fields, or │
│                                  │ others atomically                 │ use a single source of truth +      │
│                                  │                                  │ async propagation                  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "We'll just use MongoDB for      │ Application actually has highly  │ Re-evaluate data model — sometimes │
│ everything because it's          │ relational data with complex      │ this means SQL was the right       │
│ flexible" → ends up reimplementing│ multi-entity transactions, now    │ choice all along; sometimes it     │
│ JOINs and transactions in app code│ being painfully reimplemented     │ means redesigning the NoSQL schema │
│                                  │ in application code                │ around actual access patterns       │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: When would you choose SQL over NoSQL?**
A: When you need multi-row ACID transactions (e.g., financial transfers where two updates must succeed or fail together), when your data is highly relational and you need flexible ad-hoc querying with JOINs across entities, or when strong consistency is a hard requirement. SQL databases also offer mature tooling, standardized query languages, and decades of operational experience.

**Q: When would you choose NoSQL over SQL?**
A: When your schema needs to evolve rapidly or different records naturally have different shapes (product catalogs with varying attributes), when you need to scale writes to massive volume with relatively simple access patterns (key-based lookups — user sessions, activity feeds), or when you're optimizing for a specific access pattern (full-text search, time-series data, graph traversal) that a specialized NoSQL system handles better than general-purpose SQL.

**Q: Why is it hard to scale relational databases horizontally?**
A: JOINs require related data to be queried together efficiently. If you shard a relational database across multiple servers, JOINs that span shards become expensive cross-network operations, and multi-row transactions spanning shards require distributed transaction protocols (2-phase commit) which are slow and complex. This is why SQL databases are typically scaled vertically first, then via read replicas, with sharding as a last resort.

**Q: What does "polyglot persistence" mean and why do most large systems use it?**
A: Using different database technologies for different parts of a system, based on each part's specific access patterns and requirements — e.g., SQL for orders/payments (need ACID transactions), a document store for product catalogs (flexible schema), Redis for sessions (high-speed key lookups), Elasticsearch for search (full-text), and a data warehouse for analytics. No single database is optimal for every use case; large systems decompose by access pattern and choose the best-fit tool for each.

---
---

# TOPIC 2: CAP Theorem

---

## 1. What Problem Does CAP Theorem Address?

CAP Theorem is a fundamental result about **distributed systems** (any system with data spread across multiple machines connected by a network) — formulated by Eric Brewer in 2000, later formally proven. It states:

> In a distributed system, when a **network partition** occurs (some nodes can't communicate with others), you must choose between **Consistency** and **Availability**. You cannot have both.

This sounds abstract, but it has DIRECT, CONCRETE implications for every distributed database design decision — and it's one of the most commonly tested CONCEPTUAL topics in system design interviews, precisely because it's often MISUNDERSTOOD.

---

## 2. The Three Letters — Defined Precisely

```
C — CONSISTENCY
    Every read receives the MOST RECENT write (or an error).
    All nodes see the SAME data at the SAME time.
    
    NOTE: This is "consistency" in the CAP sense — related to but
    NOT IDENTICAL to the "C" in ACID (covered in the ACID topic).
    CAP consistency = "linearizability" — every client sees the
    same, most up-to-date view of data.

A — AVAILABILITY
    Every request to a non-failing node receives a response
    (not an error) — but the response might not reflect the
    most recent write.

P — PARTITION TOLERANCE
    The system continues to operate despite network partitions
    (some nodes can't communicate with others due to network
    failure — message loss, delay, or a network split).
```

---

## 3. Why You Can't Have All Three — The Core Argument

```
SETUP: A distributed database has 2 nodes, Node A and Node B,
each holding a replica of the same data. They're connected by
a network.

       ┌─────────┐         network        ┌─────────┐
       │  Node A  │◀──────────────────────▶│  Node B  │
       │ value=10 │                         │ value=10 │
       └─────────┘                         └─────────┘

SCENARIO: A NETWORK PARTITION occurs — Node A and Node B can no
longer communicate with each other (but both are still "up" and
reachable by CLIENTS).

       ┌─────────┐          ✗✗✗✗✗          ┌─────────┐
       │  Node A  │◀───── PARTITION ──────▶│  Node B  │
       │ value=10 │     (network broken)    │ value=10 │
       └─────────┘                         └─────────┘
            ▲                                    ▲
       Client 1                             Client 2
       writes value=20                      reads value

NOW: Client 1 writes value=20 to Node A. Node A CANNOT tell Node B
about this write (partition!). Client 2 then reads from Node B.

WHAT SHOULD NODE B DO?

OPTION 1 (Choose CONSISTENCY, sacrifice AVAILABILITY — "CP"):
Node B says: "I cannot guarantee I have the latest data (Node A
might have a newer value I don't know about) — I will REFUSE to
respond until the partition heals."
→ Client 2 gets an ERROR / TIMEOUT (Node B is "unavailable" for
  this request)
→ CONSISTENCY preserved (no stale data ever returned)
→ AVAILABILITY sacrificed (a request went unanswered)

OPTION 2 (Choose AVAILABILITY, sacrifice CONSISTENCY — "AP"):
Node B says: "I'll respond with what I have — value=10."
→ Client 2 gets a RESPONSE (value=10)
→ AVAILABILITY preserved (request was answered)
→ CONSISTENCY sacrificed (value=10 is STALE — the real current
  value, per Client 1's write, is 20)

THERE IS NO THIRD OPTION. As long as the partition exists, Node B
must either NOT RESPOND (sacrifice A) or RESPOND WITH POSSIBLY
STALE DATA (sacrifice C). This is the entire theorem.
```

### Why "P" Is Usually Not a Real Choice

```
A common point of confusion: "CAP theorem says pick 2 of 3 — so
can't I just pick CA and skip P (partition tolerance)?"

In a SINGLE-MACHINE system: yes, there are no network partitions
to worry about — C and A can both hold trivially.

In ANY DISTRIBUTED system (multiple machines over a network):
network partitions WILL happen. Cables get cut, switches fail,
cloud provider network hiccups occur, cross-region links degrade.
It's not a matter of IF, but WHEN.

THEREFORE: For distributed systems, P is NOT really a choice you
get to make — it's a FACT OF LIFE you must DESIGN FOR. The REAL
choice CAP theorem presents is:

"WHEN a partition happens (not if), do you choose C or A?"

This is why CAP theorem is often more usefully framed as a
choice between CP and AP systems — "CA" isn't a meaningful
category for genuinely distributed systems.
```

---

## 4. CP vs AP — Real Database Examples

```
┌─────────────────────────────────────────────────────────────────┐
│ CP SYSTEMS (Consistency + Partition Tolerance,                   │
│              sacrifice Availability during partitions)           │
│                                                                   │
│ Examples: HBase, MongoDB (with default settings), Zookeeper,     │
│           traditional RDBMS with synchronous replication,         │
│           Google Spanner (with caveats — uses TrueTime to        │
│           minimize the AP/CP tradeoff window)                     │
│                                                                   │
│ BEHAVIOR DURING A PARTITION: Some nodes become UNAVAILABLE        │
│ (return errors/timeouts) rather than risk returning stale data.  │
│                                                                   │
│ USE WHEN: Returning WRONG data is WORSE than returning NO data.   │
│ Example: Banking — would you rather your balance check FAILS      │
│ (you try again later) or SUCCEEDS but shows an OLD balance        │
│ (you might overdraw thinking you have more money than you do)?    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ AP SYSTEMS (Availability + Partition Tolerance,                  │
│              sacrifice Consistency during partitions)             │
│                                                                   │
│ Examples: Cassandra, DynamoDB (default), Riak, CouchDB,           │
│           most caches (Redis in cluster mode with async repl)     │
│                                                                   │
│ BEHAVIOR DURING A PARTITION: ALL nodes keep responding to reads/  │
│ writes, but different nodes might temporarily disagree about the │
│ current value (resolved later — "eventual consistency")           │
│                                                                   │
│ USE WHEN: Returning STALE data is BETTER than returning NO data.  │
│ Example: Social media "like count" — showing "1,234 likes" when   │
│ the real-time count is actually "1,236" is totally fine. But the  │
│ feature being COMPLETELY DOWN (can't even view the post) is bad.  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. PACELC — The More Practical Extension

CAP theorem only describes behavior DURING a partition. But partitions are (hopefully) RARE. What about NORMAL operation (no partition)? **PACELC** extends CAP to address this:

```
PACELC: "if Partition, then Availability or Consistency;
         Else (no partition), Latency or Consistency"

P-A-C: During a Partition, choose Availability or Consistency
       (this is the original CAP tradeoff)

E-L-C: Else (normal operation, no partition), choose Latency
       or Consistency

THE "ELC" PART IS OFTEN MORE RELEVANT DAY-TO-DAY:

Even with NO network partition, if you want STRONG consistency
(every read sees the latest write, from any replica), you need
to either:
  a) Read from the PRIMARY only (no replica reads — limits read
     scaling), or
  b) Have replicas SYNCHRONOUSLY confirm a write before
     acknowledging it (adds LATENCY to every write — must wait
     for the slowest replica to confirm)

If you're willing to accept EVENTUAL consistency (replicas might
lag by milliseconds), you can:
  a) Read from ANY replica (better read scaling), and
  b) Acknowledge writes as soon as the PRIMARY has them, without
     waiting for replicas (LOWER write latency)

EXAMPLES BY PACELC CLASSIFICATION:
- DynamoDB: PA/EL (Available during partitions, Low-latency 
  normally — both prioritizing latency/availability over
  consistency)
- MongoDB: PC/EC (Consistent — by default, reads go to primary;
  Consistent normally too, at some latency cost)
- Cassandra: PA/EL (tunable, but defaults toward availability/latency)
- PostgreSQL (single primary, sync replicas): PC/EC

PACELC IS WHY "EVENTUAL CONSISTENCY" SHOWS UP EVEN WHEN THERE'S
NO PARTITION — many systems CHOOSE lower latency over strict
consistency as a DEFAULT operating mode, not just a fallback
during failures.
```

---

## 6. Connecting CAP to Concepts You Already Know

```
GEO-DISTRIBUTION (from Scalability notes):
- Multi-region READ REPLICAS = an AP-leaning choice. Reads from
  a regional replica are FAST and AVAILABLE, but might be
  CONSISTENCY-stale (the "read-your-writes" problem discussed
  there is a DIRECT manifestation of choosing A/L over C).
- Active-active multi-region with conflict resolution = explicitly
  AP — both regions stay available and accept writes during a
  partition, accepting that conflicts need resolving afterward.

REPLICATION (upcoming topic):
- SYNCHRONOUS replication (primary waits for replica ACK before
  confirming write) = leans CP/EC (consistency, at a latency cost)
- ASYNCHRONOUS replication (primary confirms write immediately,
  replicates in background) = leans AP/EL (availability/latency,
  at a consistency cost — replica might be behind)

DATABASE SHARDING (upcoming topic):
- Each shard might individually be CP, but the OVERALL system's
  behavior during a partition between shards/regions still follows
  CAP reasoning for cross-shard operations.
```

---

## 7. Real-World Usage

**Amazon DynamoDB:** Explicitly AP by default — prioritizes availability (a request to a Dynamo table will almost always succeed, even during network issues), with EVENTUAL consistency for reads. DynamoDB DOES offer "strongly consistent reads" as an OPTION (at higher latency/cost) — but the default and most common usage pattern is eventually-consistent, embracing the AP side of CAP. This directly traces back to the original Amazon Dynamo paper (2007), which explicitly chose availability for the shopping cart use case — Amazon decided "always let customers add to cart, resolve conflicts later" was better than "sometimes refuse to let customers shop."

**Google Spanner:** A famous attempt to get CONSISTENCY AND high availability across a GLOBALLY distributed database — achieved using "TrueTime" (GPS + atomic clocks in Google's data centers to get extremely tightly synchronized clocks across the globe). Spanner is technically still CP (it will sacrifice availability during a severe enough partition), but the WINDOW where this matters is engineered to be extremely small. This represents an enormous engineering investment most companies can't replicate — most teams choose AP (Cassandra/DynamoDB-style) or accept CP's availability tradeoffs (traditional RDBMS) rather than building Spanner-like infrastructure.

**Banking systems (relevant to your BFSI prep):** Core ledger/balance systems are almost universally CP — a temporarily UNAVAILABLE balance-check API is an acceptable (if annoying) failure mode; a balance check that returns WRONG information (leading to a double-spend or incorrect overdraft decision) is a much more serious failure with real financial and regulatory consequences.

---

## 8. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "Split brain" — both sides of a │ Two nodes both think they're     │ Use a quorum-based approach        │
│ partition think THEY are the     │ the "primary" and both accept    │ (require a majority of nodes to    │
│ primary, both accept writes      │ writes during a partition        │ agree before accepting writes —    │
│                                  │                                  │ e.g., Raft/Paxos consensus          │
│                                  │                                  │ protocols) so only ONE side of a   │
│                                  │                                  │ partition can have a majority       │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Choosing an AP database for a    │ "Eventual consistency" allows    │ Re-evaluate: this is a CP use case;│
│ use case that NEEDS CP (e.g.,     │ a brief window where stale       │ either choose a CP database, or     │
│ inventory count for the LAST     │ inventory data allows overselling│ add application-level safeguards   │
│ unit of a product)                │                                  │ (e.g., a final strongly-consistent │
│                                  │                                  │ check at checkout time)             │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Application code assumes strong  │ Developer reads from a replica   │ Be explicit about consistency       │
│ consistency from an AP database, │ expecting it to reflect a write   │ requirements per-operation; some    │
│ causing subtle bugs               │ that just happened on the primary │ databases offer "read your own      │
│                                  │ (replication lag)                 │ writes" or tunable consistency      │
│                                  │                                  │ levels (e.g., Cassandra's            │
│                                  │                                  │ QUORUM reads/writes)                │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Conflict resolution after a       │ AP system allowed writes on both │ Design conflict resolution UP       │
│ partition heals produces           │ sides of a partition; when it    │ FRONT (LWW, CRDTs, application      │
│ unexpected/lost data               │ heals, conflicting writes must   │ merge logic) — don't discover this  │
│                                  │ be reconciled                     │ need only after an incident          │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 9. Interview Quick-Fire Answers

**Q: Explain CAP theorem in your own words.**
A: In a distributed system, when a network partition occurs (nodes can't communicate), each node must choose between responding with possibly-stale data (favoring Availability) or refusing to respond until it can guarantee correctness (favoring Consistency). You can't have both simultaneously during a partition. Partition tolerance isn't really an optional choice for distributed systems — partitions happen regardless — so the real-world framing is "CP vs AP": when a partition occurs, do you prioritize consistency or availability?

**Q: Give a real example of a system that should be CP, and one that should be AP.**
A: A banking balance/ledger system should be CP — if there's any doubt about getting the correct, up-to-date balance, it's better to return an error (the user retries) than to return a stale balance that could lead to an incorrect financial decision (e.g., overdrafting). A social media "like count" or "view count" should be AP — it's far better for the feature to remain available (showing a slightly-stale count) than to go down entirely just because one node can't immediately confirm it has the absolute latest number.

**Q: What is PACELC and why is it often more useful than CAP alone?**
A: PACELC extends CAP by addressing NORMAL operation (no partition), not just partition scenarios. It states that even without a partition, there's a tradeoff between Latency and Consistency — achieving strong consistency (every read sees the latest write) typically requires either reading only from a primary or having writes wait for synchronous replica confirmation, both of which add latency. Many systems (like DynamoDB) choose lower latency / eventual consistency as their DEFAULT behavior, not just as a fallback during failures — which is why "eventual consistency" is so commonly encountered even in systems that rarely experience actual network partitions.

**Q: How does CAP theorem relate to choosing between synchronous and asynchronous replication?**
A: Synchronous replication (the primary waits for replicas to acknowledge a write before confirming it to the client) leans toward consistency at the cost of latency (and potentially availability if a replica is slow/down — the write might block or fail). Asynchronous replication (the primary confirms the write immediately, replicates in the background) leans toward availability/low-latency at the cost of consistency — replicas may briefly have stale data, and if the primary fails before replicating, that data could be lost. This is essentially CAP's C-vs-A tradeoff applied to the replication mechanism specifically — a topic covered in depth in the Replication notes.


---
---

# TOPIC 3: Database Sharding

---

## 1. What Problem Does Sharding Solve?

A single database server has limits — limits on storage capacity (disk size), limits on write throughput (a single machine can only process so many writes/sec, even with vertical scaling), and limits on memory for caching the "working set" of data.

**Sharding** (also called **horizontal partitioning**) is the practice of splitting a database's data across MULTIPLE separate database servers ("shards"), where EACH shard holds a SUBSET of the total data — and EACH shard is a complete, independent database.

```
WITHOUT SHARDING:
┌─────────────────────────────────────────────────┐
│  ONE Database Server                              │
│  users table: ALL 500 million users               │
│  Storage: 10 TB (approaching disk limits)          │
│  Writes: 50,000 writes/sec (approaching CPU/IO     │
│          limits of even the biggest instance)      │
└─────────────────────────────────────────────────┘

WITH SHARDING (4 shards, split by user_id range):
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Shard 1     │ │   Shard 2     │ │   Shard 3     │ │   Shard 4     │
│ users 0-125M  │ │users 125-250M │ │users 250-375M │ │users 375-500M │
│ 2.5 TB        │ │ 2.5 TB        │ │ 2.5 TB        │ │ 2.5 TB        │
│ 12,500 wr/sec │ │ 12,500 wr/sec │ │ 12,500 wr/sec │ │ 12,500 wr/sec │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘

Each shard is now 1/4 the size and 1/4 the write load — and you
can ADD MORE SHARDS as you grow further. This is the relational-
database equivalent of the horizontal scaling discussed in the
Scalability notes — but applied to the DATA LAYER specifically,
which is fundamentally harder (as established in the SQL vs NoSQL
topic — JOINs and transactions don't cross shard boundaries cleanly).
```

**Analogy:** A library with 10 million books in ONE building becomes physically unmanageable — too much floor space needed, too many people trying to access it at once, too slow to find anything. Sharding is like building 10 separate library branches, each holding a SUBSET of the books (e.g., by subject area, or by first letter of the author's last name) — each branch is independently manageable, but finding a specific book now requires knowing WHICH BRANCH to go to.

---

## 2. Sharding Key Selection — The Most Important Decision

The **shard key** (or **partition key**) is the field used to decide WHICH shard a given row belongs to. This single decision determines almost everything about how well your sharded system performs.

```
EXAMPLE: An e-commerce `orders` table. Candidate shard keys:

OPTION A: shard by user_id
  Pro: "Get all orders for user X" → single shard, fast
  Con: A user with millions of orders (a large business account
       on a B2B platform) creates a "hot shard" — all their data
       on one shard while others are lighter

OPTION B: shard by order_id (e.g., using consistent hashing on
          a randomly-generated order ID)
  Pro: Extremely even distribution — order IDs are uniformly
       random, so shards stay balanced
  Con: "Get all orders for user X" now requires querying ALL
       shards (since a user's orders are scattered randomly) —
       a "scatter-gather" query, much slower

OPTION C: shard by region (e.g., geographic region of the user)
  Pro: Aligns with geo-distribution (Scalability notes) — each
       shard can live in its OWN data center, close to its users
       Also helps with DATA RESIDENCY compliance (also covered
       in Geo-distribution)
  Con: Uneven if one region has vastly more users than others
       (e.g., if 80% of users are in one region, that shard is
       overloaded while others are underutilized)
```

### The Core Tradeoff: Query Patterns vs Even Distribution

```
THIS IS THE CENTRAL TENSION IN SHARD KEY SELECTION:

A shard key that makes your MOST COMMON QUERIES fast (by keeping
related data together on one shard) often CONFLICTS with a shard
key that distributes load EVENLY (which tends to favor random/
hashed distribution that scatters related data).

GOLDEN RULE: Choose the shard key based on your DOMINANT ACCESS
PATTERN. If 95% of your queries are "get all data for user X,"
shard by user_id (or a hash of user_id) even if it means the
remaining 5% of queries (cross-user analytics, say) are slower
or need to go through a different system entirely (e.g., a data
warehouse — covered later).
```

---

## 3. Sharding Strategies — Deep Dive

### Strategy 1: Range-Based Sharding

```
HOW IT WORKS: Each shard owns a CONTIGUOUS RANGE of the shard
key's possible values.

Shard 1: user_id 0       to 124,999,999
Shard 2: user_id 125,000,000 to 249,999,999
Shard 3: user_id 250,000,000 to 374,999,999
Shard 4: user_id 375,000,000 to 499,999,999

ROUTING: 
shard = lookup_range_map(user_id)
"user_id = 130,000,001 falls in range [125M, 250M) → Shard 2"

PROS:
✅ RANGE QUERIES are efficient — "find all users with id between
   X and Y" might only touch 1-2 shards
✅ Easy to understand and reason about
✅ Easy to add a NEW shard for a NEW range (e.g., user_ids
   500M-625M go to a new Shard 5) — doesn't require moving
   EXISTING data (unlike hash-based resharding)

CONS:
❌ HOT SPOTS — if user_ids are assigned sequentially (auto-
   increment), ALL NEW USERS go to the LAST shard (the one with
   the highest range) — that shard gets ALL the write traffic
   for new signups, while older shards become read-mostly/idle
❌ Uneven distribution if the underlying data isn't uniformly
   distributed across the range (e.g., if certain ID ranges
   correspond to a single bulk-imported dataset that's much
   larger than organic data)

USE WHEN: Range queries on the shard key are common, AND the
key isn't monotonically increasing in a way that creates a
"last shard is hot" problem (or you've mitigated this — e.g.,
by sharding on a DIFFERENT, non-sequential key).
```

### Strategy 2: Hash-Based Sharding

```
HOW IT WORKS: Apply a hash function to the shard key, then use
the hash to determine the shard (often via consistent hashing —
see the Scalability notes' Consistent Hashing topic for the
full algorithm).

shard = hash(user_id) % number_of_shards
   (or, better: consistent hashing, so adding/removing shards
    doesn't require remapping nearly all keys)

EXAMPLE:
hash(user_12345) → maps to Shard 3
hash(user_67890) → maps to Shard 1
hash(user_11111) → maps to Shard 4

PROS:
✅ EXCELLENT distribution — hash functions produce near-uniform
   output, so shards stay evenly loaded regardless of access
   patterns (no "last shard is hot" problem)
✅ Naturally avoids hot spots from sequential IDs

CONS:
❌ RANGE QUERIES become impossible (or require scatter-gather
   across ALL shards) — "find all users with id between X and Y"
   has NO relationship to which shard each falls into, since
   hashing scrambles the order
❌ Resharding (adding shards) requires moving data — mitigated
   significantly by consistent hashing (only ~1/N of data moves,
   as covered in detail in the Scalability notes), but still
   non-trivial compared to range-based "just add a new range"

USE WHEN: Your queries are predominantly "exact match on shard
key" (e.g., "get data for THIS specific user_id") rather than
range scans — this describes the VAST MAJORITY of OLTP
(transactional) workloads, which is why hash-based sharding
(often via consistent hashing) is the most common approach in
practice.
```

### Strategy 3: Directory-Based (Lookup Service) Sharding

```
HOW IT WORKS: Maintain an explicit MAPPING TABLE (often in a
separate, small, highly-available database) that says
"this specific key → this specific shard" — rather than
COMPUTING the shard via range or hash.

┌──────────────────────────────────┐
│  Shard Lookup Service (e.g., a    │
│  small, replicated key-value      │
│  store or config database)         │
│                                    │
│  user_12345 → Shard 2              │
│  user_67890 → Shard 1              │
│  user_99999 → Shard 4              │
│  ... (one entry per key, or per    │
│  range of keys)                    │
└──────────────────────────────────┘

PROS:
✅ MAXIMUM FLEXIBILITY — can place ANY key on ANY shard,
   independent of any formula. Useful for:
   - Manually rebalancing "hot" individual keys to less-loaded
     shards
   - Migrating specific high-value customers to dedicated/
     premium infrastructure
   - Gradual migration during resharding (move keys one at a
     time, updating the lookup table as you go — no "big bang"
     cutover)

CONS:
❌ The lookup service itself becomes critical infrastructure —
   EVERY query needs a lookup first (extra hop/latency), and if
   the lookup service is down, you can't find ANYTHING (though
   it's typically small enough to be heavily cached/replicated)
❌ More operational complexity than a pure formula (hash/range)

USE WHEN: You need fine-grained control over data placement —
common in large multi-tenant SaaS platforms where specific
high-value tenants might need dedicated resources, or during
complex migrations.
```

### Comparison Table

```
┌──────────────────────────┬───────────────────────┬───────────────────────┬──────────────────────────┐
│ Strategy                  │ Range Queries          │ Distribution Evenness  │ Resharding Complexity     │
├──────────────────────────┼───────────────────────┼───────────────────────┼──────────────────────────┤
│ Range-based                │ Fast (often 1-2       │ Can be uneven (hot     │ Easy to ADD a new range   │
│                           │ shards)                │ "last shard" problem)  │ (no data movement)         │
│ Hash-based                 │ Scatter-gather (all   │ Excellent (near-uniform│ Moderate with consistent   │
│ (esp. consistent hashing)  │ shards)                │ via hashing)           │ hashing (~1/N moves)       │
│ Directory/lookup-based      │ Depends on            │ Fully controllable      │ Most flexible — move keys  │
│                           │ implementation         │ (manual placement)     │ individually, gradually    │
└──────────────────────────┴───────────────────────┴───────────────────────┴──────────────────────────┘
```

---

## 4. The Hard Problems Sharding Introduces

### Problem 1: Cross-Shard Joins

```
Shard 1 has: users 1-1000, their orders
Shard 2 has: users 1001-2000, their orders

QUERY: "Find all orders over ₹10,000, joined with user details,
        across ALL users"

This query needs data from BOTH shards. The database CANNOT do
this with a single SQL JOIN anymore — the application (or a
middleware layer) must:
1. Query Shard 1: "orders > 10000" → get results + user_ids
2. Query Shard 2: "orders > 10000" → get results + user_ids
3. COMBINE results in application code
4. For user details, ANOTHER round of per-shard lookups (or
   denormalize user details into the orders record itself —
   recall denormalization from SQL vs NoSQL!)

This is SLOWER, MORE COMPLEX application code, and doesn't
support all the optimizations a single database's query planner
would normally provide.
```

### Problem 2: Cross-Shard Transactions

```
SCENARIO: User A (on Shard 1) sends money to User B (on Shard 2).
This requires:
1. Decrement User A's balance (Shard 1)
2. Increment User B's balance (Shard 2)
BOTH must succeed, or BOTH must be rolled back (ACID, covered next
topic) — but they're on DIFFERENT DATABASE SERVERS!

SOLUTION: Distributed transactions (2-Phase Commit / 2PC)

┌─────────────────────────────────────────────────────────────┐
│ PHASE 1 (PREPARE):                                            │
│ Coordinator → Shard 1: "Can you decrement A's balance? Don't  │
│                          commit yet, just PREPARE."           │
│ Coordinator → Shard 2: "Can you increment B's balance? Don't  │
│                          commit yet, just PREPARE."           │
│ Both shards LOCK the relevant rows and respond "YES, ready"   │
│ (or "NO" if something's wrong — e.g., insufficient balance)   │
│                                                                │
│ PHASE 2 (COMMIT):                                             │
│ If BOTH said "YES" → Coordinator tells BOTH to COMMIT          │
│ If EITHER said "NO" → Coordinator tells BOTH to ROLLBACK       │
└─────────────────────────────────────────────────────────────┘

PROBLEMS WITH 2PC:
- SLOW — multiple round trips, rows are LOCKED during the entire
  process (blocking other operations on those rows)
- The COORDINATOR is a potential single point of failure — if it
  crashes between Phase 1 and Phase 2, shards are left in a
  "prepared but uncommitted" LIMBO state, holding locks
  indefinitely until manual intervention or timeout-based recovery

THIS IS WHY: Application architects go to great lengths to AVOID
cross-shard transactions — by choosing a shard key that keeps
"transactionally related" data TOGETHER (e.g., shard by account_id
such that a transfer between two accounts owned by the SAME user
stays on one shard), or by using SAGA patterns (a sequence of
local transactions with compensating actions for rollback,
common in microservices — connects to your LangGraph/multi-agent
orchestration background, conceptually similar to compensation
logic in agent workflows!) instead of true distributed transactions.
```

### Problem 3: Rebalancing

```
As data grows unevenly (some shards fill up faster than others)
or you add shards to handle overall growth, you need to MOVE
DATA between shards — while the system is LIVE, serving traffic.

CHALLENGES:
- Data being moved must remain READABLE/WRITABLE during the move
  (often via a "dual write" period — writes go to both old and
  new location temporarily)
- The shard lookup/routing layer must be updated ATOMICALLY (or
  near-atomically) once the move completes — a brief window of
  "where is this data NOW?" inconsistency must be handled
  gracefully (retry logic, or a directory-based lookup that can
  be updated instantly — connects back to Strategy 3 above)

THIS IS WHY consistent hashing (Scalability notes) is so valuable
for hash-based sharding — it MINIMIZES (to ~1/N) how much data
needs this complex live-migration treatment when shards are added.
```

---

## 5. Real-World Usage

**Instagram:** Famously shards their PostgreSQL database by a custom scheme: each ID (for photos, etc.) encodes a SHARD ID directly within it (using bit-packing — part of the ID's bits identify the shard, part is a sequence number, part is a timestamp). This means "which shard is this data on?" can be computed FROM THE ID ITSELF, with zero lookups — an elegant directory-free approach for a hash/ID-based scheme.

**MongoDB (built-in sharding):** Offers sharding as a core feature — you choose a "shard key" for a collection, and MongoDB's "config servers" (a directory-based lookup layer) and "mongos" routers handle distributing data across "shard" replica sets automatically, including automated rebalancing ("chunk migration") as data grows unevenly.

**Vitess (used by YouTube, Slack, GitHub):** A sharding middleware layer FOR MySQL — lets applications continue to "speak MySQL" while Vitess handles routing queries to the correct shard(s), including support for resharding with minimal downtime. This is a great example of solving sharding's complexity with a DEDICATED MIDDLEWARE LAYER rather than baking sharding logic into every application.

**Discord:** Shards their message storage by a combination of GUILD ID (server) — since most queries are "get messages for THIS server" — using a Cassandra-based architecture (which itself uses consistent hashing internally, as covered in Scalability notes) for the underlying distribution.

---

## 6. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Hot shard (one shard gets        │ Range-based sharding with        │ Switch to hash-based sharding for │
│ disproportionate load)           │ sequential IDs → all new writes  │ write-heavy keys; or use a         │
│                                  │ go to the "last" shard            │ composite/randomized prefix on     │
│                                  │                                  │ sequential IDs                     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "Celebrity" key creates a hot    │ Shard key (e.g., user_id) for a  │ Special-case extremely large       │
│ shard (e.g., one massive tenant  │ huge account puts disproportionate│ "tenants"/keys — directory-based   │
│ on a B2B SaaS platform)          │ load on its one shard             │ sharding lets you isolate them on  │
│                                  │                                  │ dedicated, appropriately-sized     │
│                                  │                                  │ infrastructure                     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Distributed transaction (2PC)    │ Coordinator crashes between       │ Avoid cross-shard transactions by  │
│ leaves shards in "prepared"      │ Prepare and Commit phases         │ design (shard key chosen so related│
│ limbo, holding locks indefinitely│                                  │ data stays together); use Sagas     │
│                                  │                                  │ for cases that genuinely need       │
│                                  │                                  │ cross-shard coordination            │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Application queries don't        │ Shard key wasn't included in     │ Carefully design APIs/queries to   │
│ include the shard key, requiring│ the query (e.g., "find order by  │ ALWAYS include the shard key when  │
│ a scatter-gather across ALL      │ order_id" when sharded by         │ possible; for unavoidable cases,    │
│ shards for every request          │ user_id) — must check every shard│ maintain a secondary index/lookup  │
│                                  │                                  │ table mapping order_id → user_id   │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 7. Interview Quick-Fire Answers

**Q: How do you choose a shard key?**
A: Base it on your DOMINANT access pattern — the queries that happen most frequently and need to be fastest. If most queries are "get everything for user X," shard by `user_id` (or its hash) so that data stays together on one shard. The core tradeoff is that a shard key optimized for your common query pattern often conflicts with one optimized for even load distribution — you generally prioritize the access pattern and accept that some less-common query types (cross-shard analytics) will be slower or handled by a separate system.

**Q: Range-based vs hash-based sharding — what's the tradeoff?**
A: Range-based sharding makes range queries efficient (a query for IDs 100-200 might only touch one or two shards) and makes adding new shards easy (just add a new range), but can create hot spots if IDs are sequential (all new writes go to the "last" shard). Hash-based sharding (especially with consistent hashing) distributes load very evenly and avoids hot spots, but range queries become scatter-gather operations across all shards, since hashing destroys any ordering relationship between keys.

**Q: Why are cross-shard transactions a problem, and how do systems avoid them?**
A: A transaction spanning two shards (different physical database servers) can't use a single database's normal ACID transaction mechanism. It requires a distributed transaction protocol like Two-Phase Commit (2PC), which is slow (multiple network round trips), holds locks across the whole process (blocking other operations), and has failure modes where a crashed coordinator leaves shards in a "prepared but uncommitted" limbo state. Systems avoid this primarily by choosing a shard key that keeps transactionally-related data together (so transactions stay within one shard), or by using the Saga pattern — a sequence of local transactions with compensating "undo" actions, accepting eventual consistency instead of true atomicity.

**Q: What happens when you need to add more shards to an existing sharded system?**
A: This is "resharding" — data must be redistributed across the new total number of shards while the system remains live. With naive hash sharding (`hash(key) % N`), changing N remaps almost all keys — extremely disruptive. Consistent hashing minimizes this to roughly 1/N of keys needing to move. Regardless of the hashing scheme, the actual data movement must happen without downtime — often via a "dual write" period where writes go to both old and new locations until migration completes, then an atomic (or near-atomic) update to the routing/lookup layer.

---
---

# TOPIC 4: Replication

---

## 1. What Problem Does Replication Solve?

**Replication** means maintaining MULTIPLE COPIES of the same data on different servers. It solves THREE distinct problems (note the overlap with the Geo-distribution topic from Scalability notes — replication is the underlying MECHANISM that geo-distribution architectures rely on):

```
PROBLEM 1: AVAILABILITY / DURABILITY
If your ONLY copy of the data is on ONE server, and that server's
disk fails, you've LOST THE DATA PERMANENTLY. Replication means a
copy exists elsewhere.

PROBLEM 2: READ SCALING
A single database server can only handle so many reads/second.
If you have 10 IDENTICAL COPIES of the data, you can spread reads
across all 10 — 10x the read capacity (recall the 100:1 read:write
ratio common in many systems, from the Back-of-Envelope topic in
Scalability notes — this is HUGE).

PROBLEM 3: LATENCY (GEO-DISTRIBUTION)
Copies of data placed CLOSER to users (different regions) let
those users read with lower latency — this is the database-layer
foundation of the Geo-distribution patterns from Scalability notes.
```

---

## 2. Primary-Replica (Master-Slave) Replication — The Foundation

```
                  ┌──────────────────┐
   WRITES  ──────▶│  PRIMARY (Master)  │
                  │  (single source of │
                  │   truth for writes)│
                  └─────────┬──────────┘
                             │
              ┌──────────────┼──────────────┐
              │ replication  │ replication   │
              ▼ stream        ▼ stream        ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │  Replica 1     │ │  Replica 2     │ │  Replica 3     │
     │  (read-only)   │ │  (read-only)   │ │  (read-only)   │
     └──────────────┘ └──────────────┘ └──────────────┘
              ▲                ▲                ▲
              └────────────────┴────────────────┘
                        READS (distributed
                        across replicas)

RULE: ALL WRITES go to the PRIMARY. The primary then streams its
changes (via a "replication log" — e.g., MySQL's binlog,
PostgreSQL's WAL — Write-Ahead Log) to each REPLICA, which applies
the same changes to its own copy of the data.

READS can go to EITHER the primary OR any replica — this is where
the read-scaling benefit comes from.
```

### How the Replication Log Works (The Mechanism)

```
EVERY write operation (INSERT, UPDATE, DELETE) is first recorded
in a sequential, append-only LOG before (or as) it's applied to
the actual data files. This log is the SOURCE OF TRUTH for
replication.

PRIMARY's WAL (Write-Ahead Log) — a sequential stream of changes:
┌─────────────────────────────────────────────────────────────┐
│ LSN 1001: INSERT INTO orders (id=501, user_id=10, total=999) │
│ LSN 1002: UPDATE users SET balance=balance-999 WHERE id=10    │
│ LSN 1003: INSERT INTO orders (id=502, user_id=22, total=4999) │
│ LSN 1004: UPDATE inventory SET stock=stock-1 WHERE sku='ABC' │
│ ...                                                            │
└─────────────────────────────────────────────────────────────┘
(LSN = Log Sequence Number — a monotonically increasing identifier)

Each REPLICA maintains a "cursor" — the LSN up to which it has
applied changes. The replica continuously:
1. Asks the primary: "send me everything after LSN 1002"
   (the replica's current cursor position)
2. Receives LSN 1003, 1004, ... 
3. Applies each change to its own copy of the data
4. Advances its cursor

REPLICATION LAG = (primary's latest LSN) - (replica's current LSN),
typically measured in TIME (e.g., "this replica is 200ms behind")
by comparing timestamps embedded in the log entries.
```

---

## 3. Synchronous vs Asynchronous Replication — The Critical Tradeoff

This is THE most important concept in this topic, and connects DIRECTLY to CAP theorem and PACELC from Topic 2.

### Asynchronous Replication

```
TIMELINE:
1. Client sends WRITE to Primary
2. Primary applies the write to its OWN data, appends to its WAL
3. Primary responds to client: "WRITE SUCCESSFUL" ✅
   (this happens IMMEDIATELY — does NOT wait for replicas!)
4. (separately, in the background) Primary streams the WAL entry
   to Replica 1, Replica 2, Replica 3
5. Each replica applies the change WHENEVER it receives it
   (could be milliseconds later, could be seconds if a replica
   is lagging)

PROS:
✅ LOW WRITE LATENCY — client gets confirmation immediately,
   without waiting for any network round trip to replicas
✅ Primary remains available even if ALL replicas are down/slow
   (writes still succeed)

CONS:
❌ DATA LOSS RISK — if the PRIMARY crashes AFTER step 3 (client
   told "success") but BEFORE step 4 completes (replicas haven't
   received the change yet), and the primary's disk is
   UNRECOVERABLE, that write is LOST FOREVER — even though the
   client was told it succeeded!
❌ REPLICAS CAN SERVE STALE DATA — a read from a replica might
   not reflect the most recent write (the "read-your-writes"
   problem from Geo-distribution, Scalability notes)

THIS IS "AP" / "EL" LEANING (PACELC) — favors availability and
low latency over strict consistency/durability guarantees.
```

### Synchronous Replication

```
TIMELINE:
1. Client sends WRITE to Primary
2. Primary applies the write to its OWN data, appends to its WAL
3. Primary sends the WAL entry to Replica(s) and WAITS for
   acknowledgment that they've received (and often, applied) it
4. ONLY AFTER receiving ACK from replica(s) → Primary responds
   to client: "WRITE SUCCESSFUL" ✅

PROS:
✅ NO DATA LOSS — by the time the client is told "success", the
   data exists on MULTIPLE servers. If the primary crashes
   immediately after, a replica has the data and can become the
   new primary with ZERO data loss.
✅ REPLICAS ARE GUARANTEED UP TO DATE (at least the synchronous
   ones) — reading from them reflects the latest write

CONS:
❌ HIGHER WRITE LATENCY — every write must wait for a network
   round trip to the replica(s), PLUS the replica's own write time
❌ AVAILABILITY RISK — if the synchronous replica is DOWN or SLOW,
   what happens to writes on the primary?
   - If the primary BLOCKS writes until the replica responds →
     primary becomes UNAVAILABLE for writes (CP-leaning: sacrifice
     availability for consistency)
   - If the primary proceeds anyway (falls back to async) →
     you've LOST the durability guarantee you wanted (some systems
     do this as a "best effort" — e.g., PostgreSQL's
     synchronous_commit can be configured with fallback behavior)

THIS IS "CP" / "EC" LEANING (PACELC) — favors consistency/
durability over availability/low latency.
```

### Semi-Synchronous Replication (The Common Middle Ground)

```
HOW IT WORKS: The primary waits for ACK from AT LEAST ONE replica
(not ALL replicas) before confirming the write to the client.
The remaining replicas update asynchronously.

PRIMARY ──sync──▶ Replica 1 (must ACK before write confirmed)
        ──async─▶ Replica 2 (updates whenever, no blocking)
        ──async─▶ Replica 3 (updates whenever, no blocking)

BENEFIT: Guarantees the write exists on AT LEAST 2 servers
(primary + 1 sync replica) before confirming — strong durability
— WITHOUT requiring ALL replicas to be available/fast (only ONE
needs to ACK).

THIS IS THE DEFAULT/RECOMMENDED CONFIGURATION FOR MANY PRODUCTION
SYSTEMS (e.g., MySQL semi-synchronous replication, PostgreSQL
with synchronous_standby_names set to ANY 1).
```

---

## 4. Multi-Leader (Multi-Primary) Replication

```
Instead of ONE primary accepting all writes, MULTIPLE nodes can
each accept writes — and replicate to each other.

           ┌──────────────┐         ┌──────────────┐
WRITES ───▶│  Primary A     │◀───────▶│  Primary B     │◀─── WRITES
(Region 1) │ (Region 1)     │  bi-dir  │ (Region 2)     │ (Region 2)
           └──────────────┘ replicate └──────────────┘

This is EXACTLY the "Active-Active Multi-Region" pattern from the
Geo-distribution topic (Scalability notes) — replication is the
underlying mechanism, and the CONFLICT RESOLUTION problem discussed
there (Last-Write-Wins, CRDTs, application merge logic) is the
direct consequence of allowing writes on multiple "primaries"
that can conflict.

WHEN TO USE: When you need LOW-LATENCY LOCAL WRITES in multiple
regions (recall: with single-primary, non-primary regions have
high write latency since writes must travel to the primary).
ACCEPT the conflict-resolution complexity as the cost.
```

---

## 5. Leaderless Replication (Quorum-Based)

```
USED BY: Cassandra, DynamoDB, Riak — NO single "primary" at all.
ANY node can accept reads AND writes. Consistency is achieved
through QUORUMS — requiring a certain NUMBER of nodes to
participate in each read/write.

NOTATION: N = total replicas, W = write quorum, R = read quorum

EXAMPLE: N=3, W=2, R=2
- A WRITE must be acknowledged by AT LEAST 2 of the 3 replicas
  before being considered successful
- A READ must query AT LEAST 2 of the 3 replicas and use the
  MOST RECENT version among their responses (each replica
  includes a timestamp/version)

WHY W=2, R=2 (with N=3) GUARANTEES CONSISTENCY (mostly):
If a write succeeded on 2 of 3 replicas, and a read queries 2 of
3 replicas, by the PIGEONHOLE PRINCIPLE, AT LEAST ONE of the
replicas the read contacts MUST OVERLAP with the replicas the
write reached — so the read will see (at least one copy of) the
latest write, and can identify it as "most recent" via the
timestamp/version.

TUNABLE CONSISTENCY:
- W=1, R=1: Fastest, but NO overlap guarantee — might read stale
  data (AP-leaning)
- W=N, R=1: Writes must reach ALL replicas (slow writes, but any
  single replica read is guaranteed fresh)
- W=1, R=N: Fast writes, but reads must check ALL replicas
  (slow reads, but guaranteed fresh)
- W + R > N: General rule for "strong" quorum consistency

This TUNABILITY is exactly the "tunable consistency" mentioned
in the CAP theorem topic for Cassandra/DynamoDB — the application
can choose, PER OPERATION, where on the consistency/latency/
availability spectrum it wants to operate.
```

---

## 6. Replication Comparison Table

```
┌──────────────────────────┬─────────────────────────┬─────────────────────────┬──────────────────────────┐
│ Replication Type           │ Write Path               │ Consistency              │ Best For                  │
├──────────────────────────┼─────────────────────────┼─────────────────────────┼──────────────────────────┤
│ Async Primary-Replica       │ Single primary, fast      │ Eventual (replicas may   │ Read scaling, general      │
│                           │ writes, background repl   │ lag); risk of data loss   │ purpose, most common       │
│                           │                          │ on primary failure        │ default                    │
│ Sync Primary-Replica        │ Single primary, waits for │ Strong (no data loss)    │ Critical data requiring    │
│                           │ replica ACK                │                          │ durability guarantees       │
│ Semi-Sync Primary-Replica   │ Waits for ≥1 replica ACK   │ Strong durability (≥2     │ Production default for     │
│                           │                          │ copies guaranteed)        │ most "important" data       │
│ Multi-Leader                │ Multiple primaries,        │ Eventual + conflicts      │ Multi-region active-active,│
│                           │ local writes               │ (needs resolution)        │ low write latency globally │
│ Leaderless / Quorum         │ Any node, quorum-based     │ Tunable per-operation      │ Massive scale, flexible     │
│                           │                          │ (W+R vs N)                │ consistency needs           │
└──────────────────────────┴─────────────────────────┴─────────────────────────┴──────────────────────────┘
```

---

## 7. Failover — What Happens When the Primary Dies?

```
DETECTION:
A monitoring/orchestration system (e.g., Patroni for PostgreSQL,
MySQL Group Replication, or a cloud provider's managed failover)
detects the primary is unresponsive (failed health checks —
connects back to the Health Checks concept from the Load Balancing
topic in Scalability notes!)

PROMOTION:
One of the replicas is PROMOTED to become the new primary.

WHICH REPLICA? Ideally, the one with the LEAST replication lag
(most up-to-date) — promoting a lagging replica means the writes
that were "in flight" but not yet replicated to THAT replica are
LOST.

┌─────────────────────────────────────────────────────────────┐
│ Before failure:                                              │
│   Primary (LSN 1004) ──▶ Replica A (LSN 1004, in sync)        │
│                      ──▶ Replica B (LSN 1001, lagging)         │
│                                                                │
│ Primary crashes at LSN 1004.                                  │
│                                                                │
│ Promote Replica A (LSN 1004) → new primary. ZERO data loss    │
│ (assuming synchronous replication to A, or A happened to be   │
│ caught up).                                                    │
│                                                                │
│ If Replica B were promoted instead → LSN 1002-1004 (3 writes)│
│ are LOST — they existed on the old primary but never reached  │
│ B before the crash.                                            │
└─────────────────────────────────────────────────────────────┘

RECONFIGURATION:
- The OTHER replicas must be told to replicate from the NEW
  primary instead of the old one
- The application/connection layer must be redirected to the new
  primary (often via DNS update, or a "virtual IP" that moves to
  whichever node is currently primary, or application-level
  service discovery)

THIS ENTIRE PROCESS TAKES TIME (seconds to tens of seconds,
depending on automation) — during which WRITES ARE UNAVAILABLE
(no primary to accept them). This is a real, measurable
AVAILABILITY cost of the primary-replica model, distinct from
the CONSISTENCY tradeoffs discussed above.
```

---

## 8. Real-World Usage

**PostgreSQL / MySQL (most common setup):** Primary-replica with asynchronous or semi-synchronous streaming replication. Read replicas serve reporting queries and read-heavy application traffic, while all writes go to the primary. Tools like Patroni (PostgreSQL) or MySQL's built-in Group Replication / Orchestrator handle automated failover.

**MongoDB:** Uses "replica sets" — one primary, multiple secondaries, with automatic election of a new primary if the current one fails (using a Raft-like consensus protocol). Read preference is configurable per-query (read from primary for strong consistency, or from secondaries for lower latency at the cost of potential staleness).

**Cassandra:** Leaderless, quorum-based replication as described above — every write specifies a "replication factor" (N) and consistency level (effectively setting W and R), giving applications fine-grained control over the consistency/availability/latency tradeoff PER QUERY.

**Redis:** Primary-replica (asynchronous by default) for general use; Redis Sentinel handles automated failover; Redis Cluster combines sharding (Topic 3 concepts) WITH per-shard replication for both scale AND redundancy simultaneously.

---

## 9. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Data loss on failover            │ Asynchronous replication;        │ Use semi-synchronous replication  │
│                                  │ primary crashed before recent     │ for critical data — guarantees at │
│                                  │ writes reached any replica        │ least one replica has every        │
│                                  │                                  │ acknowledged write                 │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "Split brain" — old primary       │ Network partition isolates the    │ Use consensus protocols (Raft) for │
│ comes back online and ALSO        │ primary; a replica is promoted,   │ leader election; the OLD primary    │
│ thinks it's still primary,        │ but the OLD primary doesn't        │ must be explicitly "fenced" (told   │
│ accepting conflicting writes      │ realize it's been replaced         │ to stop accepting writes / shut down)│
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Replication lag causes stale      │ Async replication; replica is      │ For latency-sensitive reads right  │
│ reads right after a write          │ behind the primary at the moment   │ after a write, read from the       │
│ ("read your own writes" issue)    │ of read                            │ primary or use "read your own       │
│                                  │                                  │ writes" session routing             │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Replication storm overwhelms      │ A replica falls far behind (e.g., │ Monitor replication lag with        │
│ network/replicas after a long      │ was offline for maintenance),     │ alerting; consider rebuilding a     │
│ outage of one replica             │ then tries to "catch up" all at    │ severely lagging replica from a     │
│                                  │ once, consuming huge bandwidth     │ fresh snapshot instead of catch-up   │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Multi-leader write conflicts       │ Same record updated on two         │ Conflict resolution strategy        │
│ (active-active)                   │ different "leaders" near-          │ (LWW, CRDTs, application logic) —   │
│                                  │ simultaneously                    │ as discussed in Geo-distribution     │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 10. Interview Quick-Fire Answers

**Q: What's the difference between synchronous and asynchronous replication, and what's the tradeoff?**
A: In asynchronous replication, the primary confirms a write to the client immediately, then replicates to replicas in the background — giving low write latency, but risking data loss if the primary crashes before replication completes (the client was told "success" for data that may never reach a replica). In synchronous replication, the primary waits for replica acknowledgment before confirming the write — guaranteeing durability (the data exists on multiple servers before "success" is returned), at the cost of higher write latency and potential availability issues if the synchronous replica is slow or down. Semi-synchronous (wait for at least ONE replica, not all) is a common middle ground.

**Q: How does a quorum-based (leaderless) replication system like Cassandra achieve tunable consistency?**
A: With N total replicas, each write requires acknowledgment from W replicas, and each read queries R replicas, using the most recent version found (via timestamps). If W + R > N, there's guaranteed overlap between the replicas a write reached and the replicas a read checks — guaranteeing the read sees the latest write (strong consistency). Applications can tune W and R per-operation: lower values (e.g., W=1, R=1) favor speed/availability over consistency, higher values favor consistency over latency — directly implementing the PACELC tradeoff on a per-query basis.

**Q: What happens during primary failover, and what's the risk?**
A: A monitoring system detects the primary is down, and one of the replicas is promoted to be the new primary. The risk is data loss: if the promoted replica was LAGGING behind the old primary (common with async replication), any writes that existed on the old primary but hadn't yet replicated to this replica are permanently lost. There's also an availability gap — writes are unavailable during the detection + promotion + reconfiguration process, which can take seconds. Using synchronous or semi-synchronous replication for at least one replica minimizes the data-loss risk by ensuring at least one replica is always caught up.

**Q: How does replication relate to the CAP theorem / PACELC?**
A: Replication is the underlying mechanism for most of the CAP/PACELC tradeoffs. Asynchronous replication is an "AP/EL" choice — favoring availability and low latency, accepting eventual consistency and potential data loss. Synchronous replication is "CP/EC" — favoring consistency/durability at the cost of latency and availability (if the sync replica is unreachable). Multi-leader replication is the mechanism behind "active-active" geo-distribution, and directly creates the conflict-resolution challenges discussed under CAP's AP side. Quorum-based leaderless replication makes this tradeoff explicitly tunable per-operation via the W/R/N parameters.


---
---

# TOPIC 5: Indexing

---

## 1. What Problem Does Indexing Solve?

Without an index, finding a specific row in a table requires the database to check EVERY ROW, one by one — a **full table scan**.

```
TABLE: users (10 million rows)

QUERY: SELECT * FROM users WHERE email = 'yash@example.com';

WITHOUT AN INDEX:
The database starts at row 1, checks if email matches. No.
Row 2, checks. No. Row 3... ... Row 10,000,000, checks. 
If the matching row happens to be near the END of the table
(or doesn't exist at all), the database checked ALL 10 million
rows — an O(n) operation.

On a fast server, scanning 10 million rows might take hundreds
of milliseconds to seconds — WAY too slow for a query that should
feel instant (e.g., a login lookup).

WITH AN INDEX on the email column:
The database uses a separate, SORTED data structure (typically a
B-Tree, explained below) that maps email values directly to row
locations. Finding 'yash@example.com' becomes a SEARCH in this
structure — O(log n) — roughly 23 comparisons for 10 million rows,
INSTEAD OF up to 10 million comparisons.
```

**Analogy:** A book with 1,000 pages and no index. To find every mention of "consistent hashing," you'd have to read EVERY page (full scan). A book WITH an index at the back lets you jump directly to the relevant pages — the index itself takes up a few extra pages (storage cost), and someone had to BUILD it when the book was written (write cost) — but lookups become nearly instant.

---

## 2. The B-Tree — The Workhorse Data Structure

The vast majority of database indexes (in PostgreSQL, MySQL/InnoDB, SQL Server, Oracle) use a variant of the **B-Tree** (specifically, often a **B+ Tree**).

### Core Intuition — Why Not a Simple Binary Search Tree?

```
A regular Binary Search Tree (BST) — each node has at most 2
children — works great IN MEMORY. But databases store data on
DISK, and disk reads happen in fixed-size BLOCKS (e.g., 4KB,
8KB, 16KB pages) — reading 1 byte costs the SAME as reading the
ENTIRE BLOCK (this connects to the "disk seek" latency numbers
from the Back-of-Envelope Estimation topic — a single disk seek
~10ms is EXPENSIVE; you want to MINIMIZE the NUMBER of seeks,
even if each seek reads more data).

A BST with millions of entries would be very TALL (many levels
deep) — and EACH LEVEL might require a SEPARATE DISK READ
(seek). 10 million entries → ~23 levels in a binary tree → up
to 23 disk seeks per lookup → 23 × 10ms = 230ms. Too slow!

A B-TREE solves this by making each NODE WIDE — each node holds
MANY keys (often hundreds), sized to fit exactly in ONE DISK
BLOCK. This makes the tree very SHALLOW (few levels) even with
millions of entries — fewer disk seeks needed.
```

### B+ Tree Structure — Visualized

```
                    ┌─────────────────────────────┐
                    │   ROOT NODE (1 disk block)    │
                    │   [ 30 | 60 | 90 ]              │
                    └──┬────────┬────────┬────────┬──┘
                       │        │        │        │
            ┌──────────▼──┐ ┌───▼────────▼──┐ ┌───▼──────────┐
            │ [5|10|15|20]│ │ [35|40|50|55]   │ │ [65|75|85|100]│
            │  (< 30)     │ │  (30-60)        │ │  (> 90)       │
            └──┬───┬───┬──┘ └────┬────┬───────┘ └──┬────┬──────┘
               │   │   │         │    │             │    │
          ┌────▼┐ ┌▼─┐┌▼──┐  ┌──▼┐ ┌─▼──┐      ┌──▼┐ ┌─▼──┐
          │LEAF │ │..││.. │  │.. │ │ .. │      │.. │ │ .. │
          │NODES│ │  ││   │  │   │ │    │      │   │ │    │
          │(actual│ │  ││   │  │   │ │    │      │   │ │    │
          │data    │ │  ││   │  │   │ │    │      │   │ │    │
          │pointers)│ │  ││   │  │   │ │    │      │   │ │    │
          └────────┘ └──┘└───┘  └───┘ └────┘      └───┘ └────┘

EACH NODE = ONE DISK BLOCK (e.g., 8KB), containing MANY keys
(could be hundreds for small key types like integers).

LEAF NODES contain the actual ROW LOCATIONS (pointers to where
the full row data is stored on disk), AND leaf nodes are LINKED
TOGETHER in sorted order (the "+" in B+ Tree) — making RANGE
SCANS efficient (once you find the start of a range, just follow
the linked leaves).

WHY THIS IS FAST:
With a "fan-out" (number of children per node) of, say, 200:
- 1 level: 200 entries
- 2 levels: 200 × 200 = 40,000 entries
- 3 levels: 200³ = 8,000,000 entries
- 4 levels: 200⁴ = 1.6 BILLION entries

A B+ Tree indexing 10 million rows is typically only 3-4 LEVELS
DEEP — meaning a lookup requires only 3-4 disk block reads
(and in practice, the TOP levels are almost always cached in
memory — connects to the memory vs disk latency numbers from
Back-of-Envelope Estimation — so often just 1 ACTUAL disk read
is needed).
```

### How a Lookup Works

```
QUERY: Find row where indexed_column = 52

Step 1: Read ROOT node [30 | 60 | 90]
        52 is between 30 and 60 → go to that child

Step 2: Read child node [35|40|50|55]
        52 is between 50 and 55 → go to that child's leaf

Step 3: Read LEAF node, find entry for key=52, get the pointer
        to the actual row's location on disk

Step 4: Read the actual ROW from disk using that pointer

TOTAL: ~3-4 disk reads (often fewer in practice due to caching
of upper levels), regardless of whether the table has 10,000
rows or 10,000,000,000 rows (logarithmic growth!) — this is the
O(log n) guarantee.
```

---

## 3. The Write Cost of Indexes — There's No Free Lunch

```
EVERY INDEX ON A TABLE MUST BE UPDATED ON EVERY WRITE TO THAT
TABLE.

INSERT INTO users (id, name, email) VALUES (123, 'Yash', 'yash@x.com');

If `users` has indexes on: id (primary key), email (unique),
AND name (for search) — THREE indexes — this single INSERT must:
1. Insert the row into the main table storage
2. Insert an entry into the `id` index's B+ Tree (find correct
   leaf, possibly SPLIT a node if it's full — "page split")
3. Insert an entry into the `email` index's B+ Tree (same)
4. Insert an entry into the `name` index's B+ Tree (same)

= 4 STRUCTURES TO UPDATE for ONE logical write!

THIS IS WHY:
- "Just add an index for every column you might query" is BAD
  ADVICE — each additional index SLOWS DOWN every INSERT/UPDATE/
  DELETE on that table, and consumes additional disk space
  (indexes can collectively be LARGER than the table data itself!)
- Index design is a TRADEOFF: read speed vs write speed vs
  storage. The right answer depends on your read:write ratio
  (recall this concept from Back-of-Envelope Estimation) — a
  table that's read 1000x more than it's written can easily
  justify several indexes; a write-heavy table (e.g., a raw
  event log) might intentionally have FEW or NO secondary indexes
```

---

## 4. Types of Indexes — Deep Dive

### Primary Index (Clustered Index)

```
The PRIMARY KEY index is special in many databases (notably
MySQL's InnoDB engine, and SQL Server with clustered indexes):
the TABLE DATA ITSELF is physically stored IN THE ORDER of the
primary key — the B+ Tree's LEAF NODES contain the ACTUAL ROW
DATA, not just a pointer to it.

BENEFIT: Looking up by primary key requires NO extra step to
fetch the row — the leaf node IS the row.

ONLY ONE clustered index can exist per table (data can only be
PHYSICALLY sorted one way).
```

### Secondary Index (Non-Clustered Index)

```
ANY OTHER index (e.g., on `email`, `created_at`) is a SEPARATE
B+ Tree whose leaf nodes contain a POINTER (e.g., the primary key
value, or a physical row address) back to the main table.

LOOKUP PROCESS for a secondary index:
1. Search the secondary index's B+ Tree for the value (e.g., email)
2. Find the LEAF entry → get the primary key (or row pointer)
3. Use that to look up the ACTUAL ROW in the primary/clustered index
   (this second step is sometimes called a "bookmark lookup" or
   "table access by row ID")

This is why a query that can be ANSWERED ENTIRELY from a
secondary index — WITHOUT needing step 3 — is much faster
("index-only scan" / "covering index" — see below).
```

### Composite (Multi-Column) Index

```
CREATE INDEX idx_user_status ON orders (user_id, status);

This creates ONE B+ Tree where entries are sorted FIRST by
user_id, THEN by status (within each user_id).

┌─────────────────────────────────────────────────────────────┐
│ (user_id=1, status='delivered') → row pointer                │
│ (user_id=1, status='pending')   → row pointer                │
│ (user_id=1, status='shipped')   → row pointer                │
│ (user_id=2, status='delivered') → row pointer                │
│ (user_id=2, status='pending')   → row pointer                │
│ ...                                                            │
└─────────────────────────────────────────────────────────────┘

THE "LEFTMOST PREFIX" RULE — extremely important:

✅ WHERE user_id = 1                          → uses index (prefix)
✅ WHERE user_id = 1 AND status = 'pending'   → uses index (full match)
❌ WHERE status = 'pending'                    → CANNOT use this
   index efficiently! The index is sorted by user_id FIRST — 
   entries with status='pending' are SCATTERED throughout the
   tree (one for each user_id), not grouped together.
   (A separate index on `status` alone would be needed for this
   query pattern.)

INTUITION: Think of a phone book sorted by (Last Name, First
Name). You can quickly find "Sharma, Yash" or all "Sharma, *".
But finding everyone with First Name = "Yash" (regardless of
last name) requires scanning the ENTIRE phone book — the sort
order doesn't help.

ORDER MATTERS: An index on (user_id, status) is DIFFERENT from
an index on (status, user_id) — choose the order based on which
column is used in EQUALITY filters most often / first, and
consider which queries need to use a PREFIX of the index.
```

### Covering Index

```
A covering index is one that contains ALL the columns a query
needs — so the database can answer the ENTIRE query using JUST
the index, without ever touching the main table (no "step 3"
bookmark lookup from above).

CREATE INDEX idx_covering ON orders (user_id, status, total);

QUERY: SELECT status, total FROM orders WHERE user_id = 123;

Since `status` and `total` are BOTH part of the index (alongside
the filter column `user_id`), the database can read EVERYTHING
it needs directly from the index's B+ Tree leaves — called an
"index-only scan" — significantly faster, as it avoids the extra
disk reads to fetch full rows.

TRADEOFF: Covering indexes are LARGER (more columns stored in the
index itself) — another instance of the read-speed vs storage/
write-cost tradeoff.
```

### Hash Index

```
Some databases (and many NoSQL key-value stores — connects to the
SQL vs NoSQL topic) offer HASH INDEXES instead of B-Trees for
certain use cases.

HOW: hash(key) → bucket containing pointer(s) to row(s)

PROS: O(1) lookup for EXACT MATCHES — even faster than B-Tree's
O(log n) for simple "find by exact key" queries (this is exactly
how Redis, DynamoDB, etc. achieve their speed for key-based lookups)

CONS: 
❌ NO RANGE QUERIES (hashing destroys ordering — same limitation
   as hash-based sharding from the Sharding topic!)
❌ NO SORTING support
❌ Hash collisions need handling (multiple keys → same bucket)

USE WHEN: You ONLY ever do exact-match lookups on this column,
NEVER ranges or sorting (e.g., a session_id → session_data lookup).
```

---

## 5. EXPLAIN — Reading a Query Plan (Practical Skill)

```
EVERY relational database lets you ask: "HOW will you execute
this query?" — via EXPLAIN (or EXPLAIN ANALYZE for actual
runtime stats).

EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';

POSSIBLE OUTPUT (simplified, conceptually):

Without an index on (user_id, status):
  → "Seq Scan on orders" (sequential/full table scan)
  → cost estimate: HIGH, scales with table size

With an index on (user_id, status):
  → "Index Scan using idx_user_status on orders"
  → cost estimate: LOW, roughly constant regardless of table size

THIS IS THE #1 PRACTICAL TOOL for diagnosing "why is this query
slow?" — if EXPLAIN shows "Seq Scan" on a large table for a query
that filters/sorts, that's almost always a sign a useful index is
MISSING (or the existing index can't be used due to how the query
is written — e.g., violating the leftmost-prefix rule, or applying
a function to the indexed column like `WHERE LOWER(email) = ...`
which prevents using a plain index on `email`).
```

---

## 6. Real-World Usage

**PostgreSQL/MySQL:** B+ Tree indexes are the default and most common. PostgreSQL additionally offers GiST/GIN indexes for full-text search and array/JSON containment queries, and BRIN indexes for very large, naturally-ordered tables (e.g., time-series data ordered by timestamp — connects to the Time-series DBs topic) where a full B-Tree would be overkill.

**Elasticsearch:** Built around an "inverted index" (conceptually different from B-Trees) — optimized specifically for full-text search: maps EACH WORD to the list of documents containing it, enabling fast "find all documents containing X" queries that would be painfully slow with B-Tree indexes on text columns.

**DynamoDB:** Every table has a mandatory PRIMARY KEY (hash-based, for O(1) lookups), and optionally "Global Secondary Indexes" (GSIs) and "Local Secondary Indexes" (LSIs) for querying by other attributes — but these come with their OWN write costs and capacity considerations, explicitly surfaced to the developer (unlike traditional RDBMS where indexes feel more "automatic").

**Stripe/payment systems (relevant to your BFSI prep):** Heavy use of composite indexes on (merchant_id, created_at) or (merchant_id, status, created_at) patterns — since the dominant query pattern is "show me THIS merchant's transactions, filtered by status, sorted by time" — directly applying the leftmost-prefix and covering-index concepts above.

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Query suddenly becomes slow      │ Index exists but isn't being     │ Run EXPLAIN — check for "Seq Scan";│
│ ("it was fast yesterday!")       │ used — e.g., a function was       │ avoid wrapping indexed columns in  │
│                                  │ applied to the column            │ functions in WHERE clauses, or use │
│                                  │ (WHERE LOWER(email)=...)          │ a functional/expression index       │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Writes (INSERT/UPDATE) become     │ Too many indexes on a write-      │ Audit indexes — remove unused ones │
│ slow over time                   │ heavy table; each write updates  │ (most DBs can report index usage   │
│                                  │ every index                       │ statistics); consider whether some │
│                                  │                                  │ indexes are truly needed            │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Composite index not used for a   │ Query filters on the SECOND       │ Either reorder the index columns   │
│ query                           │ column of the index without the   │ (if this query pattern is more     │
│                                  │ first (leftmost-prefix violation)│ common), or add a separate index    │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Index bloat / fragmentation       │ Heavy UPDATE/DELETE activity      │ Periodic maintenance (VACUUM in     │
│                                  │ leaves "holes" in B+ Tree pages,   │ PostgreSQL, OPTIMIZE TABLE in       │
│                                  │ index grows larger than needed,   │ MySQL) to reclaim space and         │
│                                  │ degrading cache efficiency         │ defragment indexes                  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Adding an index on a huge table   │ Building a new index on a large   │ Use online/concurrent index          │
│ locks the table for a long time   │ table requires scanning and        │ creation (e.g., PostgreSQL's        │
│                                  │ sorting ALL existing data           │ CREATE INDEX CONCURRENTLY) to avoid │
│                                  │                                  │ blocking writes during creation     │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: Why do databases use B-Trees (specifically B+ Trees) for indexes instead of binary search trees?**
A: Database data lives on disk, where reads happen in fixed-size blocks and each disk seek is relatively expensive (~10ms for HDDs, less but still significant for SSDs). A binary search tree with millions of entries would be very tall, requiring many disk seeks per lookup. B+ Trees pack many keys into each node (sized to match a disk block), making the tree very shallow — typically 3-4 levels even for millions of rows — minimizing the number of disk reads per lookup to O(log n) with a very large base.

**Q: What's the cost of adding an index, and how do you decide whether to add one?**
A: Every index must be updated on every INSERT/UPDATE/DELETE to the table — more indexes mean slower writes and more storage. The decision should be based on the read:write ratio for that table and column: if a column is queried frequently (especially in WHERE/JOIN/ORDER BY clauses) relative to how often the table is written, an index is usually worth the write-cost. For write-heavy tables (e.g., raw event logs), minimizing indexes is often the right call.

**Q: Explain the "leftmost prefix" rule for composite indexes.**
A: A composite index on (A, B, C) is a single B+ Tree sorted first by A, then by B (within each A), then by C (within each A,B). This index can efficiently serve queries that filter on A alone, or A+B, or A+B+C — any "prefix" of the column list, in order. But a query that filters ONLY on B (or only on C), without A, can't use this index efficiently, because entries matching that filter are scattered throughout the tree rather than grouped together.

**Q: What's a covering index and why is it faster?**
A: A covering index includes ALL the columns a specific query needs — not just the filter/sort columns, but also any columns in the SELECT clause. This lets the database satisfy the entire query directly from the index's leaf nodes (an "index-only scan"), without the extra step of looking up the full row in the main table. The tradeoff is a larger index (more storage, slightly more write overhead) in exchange for avoiding that extra lookup on every read.

---
---

# TOPIC 6: ACID Properties

---

## 1. What Problem Do ACID Properties Solve?

A **transaction** is a group of one or more database operations that should be treated as a SINGLE UNIT — either ALL of them happen, or NONE of them do. ACID is a set of four guarantees that databases provide for transactions, ensuring they behave PREDICTABLY even in the presence of concurrent access, system crashes, and errors.

```
CLASSIC EXAMPLE: "Transfer ₹5,000 from Account A to Account B"

This is logically ONE operation, but physically requires TWO writes:
1. UPDATE accounts SET balance = balance - 5000 WHERE id = 'A';
2. UPDATE accounts SET balance = balance + 5000 WHERE id = 'B';

WHAT COULD GO WRONG WITHOUT ACID GUARANTEES?
- The server crashes AFTER step 1 but BEFORE step 2 → ₹5,000
  VANISHES (deducted from A, never added to B)
- Another transaction reads Account A's balance BETWEEN step 1
  and step 2 → sees a "transient" state where money has left A
  but hasn't arrived at B yet (the ₹5,000 appears to not exist
  anywhere, momentarily)
- Two CONCURRENT transfers from Account A happen simultaneously,
  both reading the "old" balance, both deducting — resulting in
  an INCORRECT final balance (a classic "race condition")

ACID GUARANTEES THESE PROBLEMS DON'T HAPPEN.
```

---

## 2. Atomicity — "All or Nothing"

```
DEFINITION: A transaction's operations either ALL succeed and
are PERMANENTLY applied (COMMIT), or if ANY part fails, ALL
operations are UNDONE as if none of them ever happened (ROLLBACK)
— there is NO "partially applied" state.

BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 5000 WHERE id = 'A';
  -- Suppose the SERVER CRASHES right here, after step 1
  UPDATE accounts SET balance = balance + 5000 WHERE id = 'B';
COMMIT;

ON RESTART: The database's RECOVERY PROCESS looks at the
transaction log (the same WAL discussed in Replication!) and sees
this transaction never reached COMMIT — so it ROLLS BACK the
partial change. Account A's balance is RESTORED to its original
value. It's as if the transaction NEVER STARTED.

HOW IT'S IMPLEMENTED: The Write-Ahead Log records EVERY change
BEFORE it's applied to the actual data, along with markers for
transaction BEGIN/COMMIT. On crash recovery, the database REPLAYS
the log: any transaction with a COMMIT marker is "redone" (ensuring
its effects persist); any transaction WITHOUT a COMMIT marker is
"undone" (its partial effects are reversed).
```

---

## 3. Consistency — "Valid State to Valid State"

```
DEFINITION: A transaction can only bring the database from one
VALID state to another VALID state — it cannot violate defined
CONSTRAINTS (foreign keys, unique constraints, check constraints
— recall these from the SQL vs NoSQL topic!).

NOTE: This "C" is DIFFERENT from CAP theorem's "C" (which is
about REPLICAS agreeing with each other). ACID's "C" is about
a SINGLE database's data always satisfying its declared rules.

EXAMPLE CONSTRAINT: CHECK (balance >= 0) — "an account balance
can never go negative"

BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 5000 WHERE id = 'A';
  -- If A's balance was only 3000, this would make it -2000
COMMIT;
-- DATABASE REJECTS this transaction: "CHECK constraint violated"
-- Transaction is automatically ROLLED BACK (connects to Atomicity!)
-- Account A's balance remains 3000, unchanged.

CONSISTENCY ENSURES: No matter what transactions run, the
database NEVER ends up in a state that violates its declared
integrity rules (foreign keys pointing to non-existent rows,
unique constraints duplicated, check constraints violated, etc.)
— this is the SAME "consistency enforcement" discussed in the
SQL vs NoSQL topic as a core SQL strength.
```

---

## 4. Isolation — "Concurrent Transactions Don't Interfere"

This is the MOST COMPLEX of the four — and the one most commonly tested in depth in interviews, because of ISOLATION LEVELS (a tunable spectrum, similar in spirit to the tunable consistency from Replication's quorum discussion).

### The Problem Isolation Solves

```
TWO TRANSACTIONS RUNNING CONCURRENTLY:

Transaction 1 (T1): Transfer ₹1000 from A to B
Transaction 2 (T2): Read total balance of A + B (for a report)

WITHOUT ISOLATION, T2 might run AT THE SAME TIME as T1, in
between T1's two updates:

T1: UPDATE accounts SET balance = balance - 1000 WHERE id='A';
                                    (A now has 1000 less)
T2: SELECT SUM(balance) FROM accounts WHERE id IN ('A','B');
                                    (sees A's NEW balance, but
                                     B's OLD balance — the ₹1000
                                     appears to have VANISHED!)
T1: UPDATE accounts SET balance = balance + 1000 WHERE id='B';
                                    (B now has 1000 more)

T2's report shows an INCORRECT total — even though BOTH
transactions, individually, are perfectly correct and will
each maintain atomicity/consistency on their OWN.

ISOLATION ensures T2 either sees the state BEFORE T1 started
(A and B's old balances, total correct) OR AFTER T1 fully
completed (A and B's new balances, total still correct) —
NEVER the "in-between" state.
```

### The Four Standard Isolation Levels

```
┌──────────────────────┬──────────────┬────────────────┬─────────────────┬───────────────────────┐
│ Isolation Level        │ Dirty Read    │ Non-Repeatable │ Phantom Read     │ Performance            │
│                       │ Prevented?    │ Read Prevented?│ Prevented?       │                        │
├──────────────────────┼──────────────┼────────────────┼─────────────────┼───────────────────────┤
│ Read Uncommitted       │ NO            │ NO              │ NO               │ Fastest (rarely used)  │
│ Read Committed         │ YES           │ NO              │ NO               │ Fast (common default,  │
│                       │              │                │                 │ e.g., PostgreSQL,      │
│                       │              │                │                 │ Oracle default)        │
│ Repeatable Read        │ YES           │ YES             │ NO (varies)      │ Moderate (MySQL        │
│                       │              │                │                 │ InnoDB default)        │
│ Serializable           │ YES           │ YES             │ YES              │ Slowest (strongest      │
│                       │              │                │                 │ guarantee)             │
└──────────────────────┴──────────────┴────────────────┴─────────────────┴───────────────────────┘
```

### The Anomalies Explained

```
DIRTY READ: Reading data that another transaction has written
but NOT YET COMMITTED (might be ROLLED BACK later!)

T1: UPDATE accounts SET balance = 5000 WHERE id='A';  (uncommitted)
T2: SELECT balance FROM accounts WHERE id='A';  → reads 5000
T1: ROLLBACK;  (T1's change is undone — balance reverts!)
T2 read a value (5000) that NEVER OFFICIALLY EXISTED.

NON-REPEATABLE READ: Reading the SAME row TWICE within one
transaction, getting DIFFERENT values, because another
transaction COMMITTED a change in between.

T2: SELECT balance FROM accounts WHERE id='A';  → reads 1000
T1: UPDATE accounts SET balance=2000 WHERE id='A'; COMMIT;
T2: SELECT balance FROM accounts WHERE id='A';  → reads 2000
(Same query, same transaction, DIFFERENT results!)

PHANTOM READ: Running the SAME RANGE QUERY TWICE within one
transaction, getting a DIFFERENT SET OF ROWS, because another
transaction INSERTED/DELETED rows matching the condition.

T2: SELECT COUNT(*) FROM orders WHERE status='pending';  → 50
T1: INSERT INTO orders (status) VALUES ('pending'); COMMIT;
T2: SELECT COUNT(*) FROM orders WHERE status='pending';  → 51
(A NEW row "phantom" appeared that matches T2's condition)
```

### How Isolation Is Implemented — MVCC (Multi-Version Concurrency Control)

```
Most modern databases (PostgreSQL, MySQL/InnoDB) implement
isolation using MVCC rather than simply LOCKING everything.

CORE IDEA: Instead of one "current value" per row, the database
keeps MULTIPLE VERSIONS of each row, each tagged with the
transaction ID that created it.

Row for Account A:
┌─────────────────────────────────────────────────────────┐
│ Version 1: balance=1000, created_by=T0 (committed)        │
│ Version 2: balance=2000, created_by=T1 (committed at t=5) │
└─────────────────────────────────────────────────────────┘

When T2 STARTS at t=3 (before T1 committed), T2 sees a
"snapshot" of the database AS OF t=3 — so T2's reads consistently
return Version 1 (balance=1000), EVEN AFTER T1 commits at t=5,
for the DURATION of T2's transaction (in Repeatable Read /
Serializable levels).

BENEFIT: READERS DON'T BLOCK WRITERS, and WRITERS DON'T BLOCK
READERS — readers just see an OLDER VERSION, while a writer
creates a NEW VERSION. This is why MVCC-based databases (Postgres,
MySQL InnoDB) generally have much better READ CONCURRENCY than
purely lock-based approaches.

COST: Old row versions must eventually be CLEANED UP (PostgreSQL's
VACUUM process, mentioned in the Indexing failure scenarios) once
no transaction needs them anymore — if this falls behind, "table
bloat" results.
```

---

## 5. Durability — "Once Committed, It Survives"

```
DEFINITION: Once a transaction is COMMITTED (the database says
"success"), its effects are PERMANENT — they will survive a power
loss, OS crash, or database restart.

HOW: The Write-Ahead Log (WAL) entries for the transaction are
FLUSHED TO DISK (fsync) BEFORE the COMMIT is acknowledged to the
client. Even if the database process crashes immediately after
(before the actual data files are updated), on restart the
recovery process REPLAYS the WAL entries and re-applies the
committed changes.

THIS CONNECTS DIRECTLY TO REPLICATION (Topic 4):
"Durability" for a SINGLE node means "survives THAT node's
crash/restart." But if the node's DISK is destroyed entirely
(not just a crash — physical failure), durability requires the
data to ALSO exist on ANOTHER node — which is exactly what
SYNCHRONOUS REPLICATION provides. "True" durability at scale =
local WAL fsync (survives process crash) + replication to
other nodes/data centers (survives hardware/site failure).
```

---

## 6. Real-World Usage

**Banking core systems (relevant to BFSI prep):** The canonical ACID use case — every transaction (transfer, deposit, withdrawal) MUST be atomic (no money created/destroyed), consistent (balances never negative, double-entry bookkeeping invariants always hold), isolated (concurrent transfers don't corrupt balances), and durable (a committed transaction survives any failure). Most core banking systems run on traditional RDBMS (Oracle, DB2, PostgreSQL) specifically because of these guarantees — this is WHY the SQL vs NoSQL decision framework so strongly favors SQL for financial data.

**E-commerce checkout:** "Place order" typically wraps: decrement inventory, create order record, create payment record, possibly award loyalty points — all in ONE transaction. If payment fails, the ENTIRE transaction rolls back (inventory is NOT decremented, no "ghost order" exists) — atomicity in action.

**NoSQL and ACID:** Historically, many NoSQL databases offered ACID only WITHIN a single document/row (e.g., MongoDB pre-4.0 had no multi-document transactions). Modern versions (MongoDB 4.0+, DynamoDB with TransactWriteItems, Cassandra's lightweight transactions) have added MULTI-document/row transaction support — but often with caveats (performance cost, limited scope) reflecting the inherent tension between ACID guarantees and horizontal scale (recall the cross-shard transaction problem from the Sharding topic — 2PC-like mechanisms underlie many of these).

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "Lost update" — two concurrent   │ Both transactions read the same │ Use appropriate isolation level    │
│ transactions both read, modify,  │ value, both write based on that  │ (Repeatable Read or Serializable), │
│ and write back, one overwrites    │ stale value, second write         │ or use explicit row-level locking  │
│ the other's change                │ overwrites the first's effect    │ (SELECT ... FOR UPDATE)            │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Deadlock                         │ T1 locks Row A then wants Row B;  │ Database detects deadlock (cycle  │
│                                  │ T2 locks Row B then wants Row A   │ in lock wait graph), automatically │
│                                  │ → each waits for the other          │ aborts ONE transaction (app must   │
│                                  │ forever                           │ retry); minimize by always locking │
│                                  │                                  │ rows in a CONSISTENT ORDER          │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Serializable isolation causes     │ Strongest isolation level         │ Use Serializable selectively only   │
│ frequent transaction failures/    │ requires the database to detect   │ for transactions that truly need it;│
│ retries under high concurrency    │ and abort transactions that       │ Read Committed/Repeatable Read for  │
│                                  │ would violate serializability      │ the rest (most apps)                │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Long-running transaction holds    │ A transaction left OPEN (e.g.,    │ Keep transactions SHORT; avoid      │
│ locks/old MVCC versions for a     │ app bug, or interactive debug     │ holding transactions open across   │
│ long time, blocking others/        │ session) for minutes               │ external calls (API requests,      │
│ causing table bloat               │                                  │ LLM inference calls!)               │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "Torn write" appears durable but  │ Database confirmed COMMIT, but     │ Ensure synchronous replication or  │
│ isn't, after a disk failure        │ the disk holding the WAL/data       │ proper fsync configuration; for     │
│                                  │ files is physically destroyed       │ true durability beyond single-node │
│                                  │ before any replication              │ failures, replicate (Topic 4)       │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: Explain ACID with a real-world example.**
A: Consider transferring money between two bank accounts (two UPDATE statements). Atomicity ensures both updates happen or neither does — no "money disappears" if the system crashes mid-transfer. Consistency ensures the result never violates rules like "balance can't be negative." Isolation ensures another transaction reading balances concurrently never sees the "in-between" state where money has left one account but not yet arrived at the other. Durability ensures that once the transfer is confirmed, it survives a crash immediately afterward — it won't silently revert.

**Q: What's the difference between Read Committed and Serializable isolation levels?**
A: Read Committed prevents "dirty reads" (you'll never read uncommitted data from another transaction) but allows "non-repeatable reads" (the same query run twice in one transaction can return different results if another transaction commits in between) and "phantom reads" (a range query can return different rows on repeat). Serializable is the strongest level — transactions behave as if they ran one at a time, sequentially, even though they're actually concurrent — preventing all these anomalies, but at the cost of more transaction aborts/retries under contention.

**Q: How does MVCC enable high concurrency without heavy locking?**
A: Instead of one "current value" per row, MVCC keeps multiple VERSIONS of each row, tagged with the transaction that created them. A reading transaction sees a consistent "snapshot" of the database as of when it started, regardless of writes happening concurrently by other transactions — readers see an older version rather than being blocked. This means readers don't block writers and writers don't block readers, dramatically improving concurrency compared to lock-everything approaches — at the cost of needing to periodically clean up old row versions (e.g., PostgreSQL's VACUUM).

**Q: Why is ACID harder to achieve in a sharded/distributed database?**
A: ACID transactions rely on mechanisms (WAL, MVCC, row locks) that work within a SINGLE database instance's storage engine. When data spans multiple shards (different physical servers), a transaction touching multiple shards needs distributed coordination — typically Two-Phase Commit — which is slow, holds locks across network round trips, and has its own failure modes (coordinator crash leaves shards in limbo, as discussed in the Sharding topic). This is why systems requiring strong ACID guarantees across large datasets either avoid sharding for that data, design shard keys to keep transactionally-related data together, or accept the performance/complexity cost of distributed transactions.


---
---

# TOPIC 7: NoSQL Types

---

## 1. What Problem Does This Topic Solve?

The "SQL vs NoSQL" topic established WHEN to consider NoSQL. But "NoSQL" is an umbrella covering at least FOUR fundamentally different data models, each with a different internal structure, query capability, and ideal use case. Choosing "NoSQL" without choosing WHICH TYPE is like choosing "vehicle" without choosing between a motorcycle, a truck, and a bus — they're all "vehicles" but solve completely different problems.

This topic goes one level deeper than SQL vs NoSQL: given that you've decided NoSQL fits, **which of the four major NoSQL types fits your access pattern?**

---

## 2. Key-Value Stores

### Core Intuition

The simplest possible data model: every piece of data is a `(key, value)` pair. The VALUE is OPAQUE to the database — it's just bytes (could be a string, JSON, binary blob). The database's ONLY job is: given a key, return the value, as fast as possible.

```
┌──────────────────────────────────────────────────────────┐
│ KEY                          │ VALUE                       │
├──────────────────────────────┼─────────────────────────────┤
│ "session:a3f8b2c1"            │ {"user_id": 123, "exp": ...} │
│ "user:123:cart"               │ {"items": [...], "total":...}│
│ "rate_limit:user_123:minute"  │ "47"                         │
│ "cache:product:789"           │ {"name": "Shoe", "price":...}│
└──────────────────────────────────────────────────────────┘

OPERATIONS: GET(key), SET(key, value), DELETE(key) — that's
essentially it (plus some extras like TTL/expiry, atomic
increment, etc.)

NO QUERY LANGUAGE for "find all keys where value.price > 1000" —
if you need that, a key-value store is the WRONG TOOL (you'd
need a secondary index, which moves you toward a document or
search database).
```

### Internal Architecture — Why So Fast

```
Because the data model is so simple, key-value stores can use
HASH TABLES (recall hash indexes from the Indexing topic) for
O(1) lookups — no B+ Tree traversal needed for exact-match
lookups.

REDIS (the most common example) is often ENTIRELY IN-MEMORY:
- All data lives in RAM → memory access (~100ns, per the
  Back-of-Envelope latency numbers) instead of disk (~10ms)
  → ~100,000x faster for reads
- Optional persistence (RDB snapshots, AOF logs) for durability,
  but the PRIMARY copy is in-memory

This is WHY Redis can handle 100,000+ operations/second on a
single instance — it's leveraging the memory-vs-disk latency
gap directly.
```

### Use Cases and Real-World Examples

```
✅ Session storage (every web request: "is this session valid?")
✅ Caching layer (in front of a slower database — recall the
   CDN/caching concepts from Networking Fundamentals, applied
   at the application/database layer)
✅ Rate limiting counters (recall the Token Bucket / Sliding
   Window Counter implementations from Scalability notes —
   these are typically implemented AS key-value operations
   in Redis!)
✅ Distributed locks (SET key value NX EX — "set if not exists,
   with expiry" — a common pattern for leader election /
   mutual exclusion in distributed systems)
✅ Leaderboards (Redis Sorted Sets — a value type that maintains
   sorted order, used for "top 10 players by score")

EXAMPLES: Redis, Memcached, Amazon DynamoDB (at its core — though
DynamoDB adds secondary indexes, making it a richer "key-value +
document" hybrid), etcd (used for Kubernetes configuration —
relevant to your container orchestration background)
```

---

## 3. Document Stores

### Core Intuition

Each record is a **document** — typically JSON (or BSON, a binary JSON variant used by MongoDB) — which can have a NESTED, FLEXIBLE structure. Unlike key-value stores, the database UNDERSTANDS the structure of the document and can QUERY/INDEX based on FIELDS WITHIN it.

```
COLLECTION: products

Document 1 (a book):
{
  "_id": "p001",
  "type": "book",
  "title": "Designing Data-Intensive Applications",
  "author": "Martin Kleppmann",
  "isbn": "978-1449373320",
  "price": 1899,
  "pages": 590
}

Document 2 (a laptop):
{
  "_id": "p002",
  "type": "laptop",
  "title": "ThinkPad X1 Carbon",
  "brand": "Lenovo",
  "price": 145000,
  "specs": {
    "cpu": "Intel i7-1365U",
    "ram_gb": 16,
    "storage_gb": 512
  },
  "available_colors": ["black", "silver"]
}

THESE TWO DOCUMENTS ARE IN THE SAME COLLECTION but have
COMPLETELY DIFFERENT FIELDS — this is "schema-on-read" / dynamic
schema, directly addressing Decision Point #3 from the SQL vs
NoSQL topic (rapidly evolving / varying-shape data).

QUERYING:
db.products.find({ price: { $lt: 2000 } })
  → returns Document 1 (price 1899 < 2000)

db.products.find({ "specs.ram_gb": { $gte: 16 } })
  → returns Document 2 (queries INSIDE nested fields!)

This is FUNDAMENTALLY different from a key-value store — the
database can index and query on ARBITRARY FIELDS within
documents, even nested ones, NOT just a single opaque key.
```

### Embedding vs Referencing — The Core Design Decision

```
This connects DIRECTLY to the denormalization discussion in the
SQL vs NoSQL topic, and is THE central design decision for
document databases.

EMBEDDING (denormalize — store related data together):
{
  "_id": "user_1",
  "name": "Yash",
  "addresses": [
    {"type": "home", "city": "Pune", "pincode": "411001"},
    {"type": "work", "city": "Pune", "pincode": "411006"}
  ]
}
→ ONE document, ONE read gets user + all their addresses.
→ GOOD WHEN: addresses are ALWAYS accessed together with the
  user, rarely queried INDEPENDENTLY, and the list doesn't grow
  unbounded (a user has 2-3 addresses, not 10,000)

REFERENCING (normalize — store IDs, separate collection):
{
  "_id": "user_1",
  "name": "Yash",
  "address_ids": ["addr_101", "addr_102"]
}
// separate "addresses" collection holds the actual address docs

→ GOOD WHEN: the related data is LARGE, grows UNBOUNDED (a user's
  "orders" could be thousands — embedding ALL of them in the user
  document would make it huge and slow to load), or needs to be
  QUERIED INDEPENDENTLY (e.g., "find all orders with status=pending
  across ALL users" — much easier with a separate collection)

RULE OF THUMB: Embed data that's "owned by" and always accessed
WITH the parent, and bounded in size. Reference data that's
large, unbounded, shared across multiple parents, or queried
independently.
```

### Real-World Examples

```
MongoDB: The most popular general-purpose document database —
flexible schema, rich querying (including aggregation pipelines
for analytics-lite operations), used heavily for content
management, catalogs, user profiles.

Couchbase, Amazon DocumentDB (MongoDB-compatible API),
Firebase Firestore (popular for mobile/web apps needing
real-time sync — connects to the WebSockets topic from
Networking Fundamentals, as Firestore uses persistent
connections for real-time updates).

USE CASES: Product catalogs (varying attributes per category —
exactly the example used in SQL vs NoSQL), user profiles, content
management systems, mobile app backends (flexible schema evolves
with app versions without painful migrations).
```

---

## 4. Column-Family (Wide-Column) Stores

### Core Intuition

This is the MOST CONFUSING NoSQL type for beginners because the name "column-family" sounds like it's about COLUMNS in the SQL sense, but the actual model is quite different. Think of it as: **rows can have a HUGE, VARIABLE number of columns, and columns are GROUPED into "families" that are stored together on disk.**

```
CONCEPTUALLY (Cassandra/HBase-style):

Row Key: "user_123"
┌─────────────────────────────────────────────────────────────┐
│ Column Family: "profile"                                      │
│   name="Yash"  email="yash@x.com"  city="Pune"                │
├─────────────────────────────────────────────────────────────┤
│ Column Family: "activity_log"  (could have THOUSANDS of      │
│   columns, one per event, with TIMESTAMPS as column names!)  │
│   "2026-06-01T10:00:00"="login"                                │
│   "2026-06-01T10:05:00"="viewed_product_456"                  │
│   "2026-06-01T10:07:00"="added_to_cart"                        │
│   ... (could be thousands more for an active user)            │
└─────────────────────────────────────────────────────────────┘

Row Key: "user_456"
┌─────────────────────────────────────────────────────────────┐
│ Column Family: "profile"                                      │
│   name="Priya"  email="priya@x.com"                            │
│   (NOTE: no "city" — this row simply doesn't have that column!)│
├─────────────────────────────────────────────────────────────┤
│ Column Family: "activity_log"                                  │
│   "2026-06-02T09:00:00"="login"                                │
│   (only ONE entry — very different "width" from user_123!)    │
└─────────────────────────────────────────────────────────────┘

KEY INSIGHT: Each ROW can have a COMPLETELY DIFFERENT SET AND
NUMBER of columns. This is EXTREMELY well suited for data where
the "columns" are actually more like a TIME-ORDERED LOG or a
SPARSE SET OF ATTRIBUTES per entity.
```

### Why This Model Scales So Well — Storage Layout

```
Within a column family, data is stored SORTED BY ROW KEY, then
BY COLUMN NAME — and PHYSICALLY GROUPED on disk by column family.

This means: "give me ALL the activity_log entries for user_123,
in chronological order" is a SINGLE SEQUENTIAL DISK READ (the
data is ALREADY stored in that order) — extremely fast for
TIME-SERIES-LIKE access patterns (this connects directly to the
upcoming Time-series DBs topic — column-family stores are a
common UNDERLYING technology for time-series workloads).

COMBINED WITH the consistent-hashing-based distribution (Topic 3
of Scalability notes) across nodes by ROW KEY, this gives:
- Horizontally scalable writes (each node handles a slice of
  row keys)
- Very fast "get everything for THIS row key" reads (data is
  co-located and sorted)
- Poor support for "query by column value across all rows"
  (e.g., "find all users with city='Pune'" requires scanning —
  unless you build a SEPARATE table/index for this access
  pattern — "denormalize for each query pattern," a VERY common
  Cassandra design principle)
```

### Real-World Examples

```
Apache Cassandra: The canonical example — used by Netflix
(viewing history, recommendations data), Discord (message
storage, as referenced in the Sharding topic), Instagram
(activity feeds). Uses the leaderless/quorum replication model
from the Replication topic, combined with consistent hashing
(Scalability notes) for distribution.

Apache HBase: Built on top of HDFS (Hadoop), used heavily in
big-data/analytics contexts at companies running Hadoop
ecosystems.

Google Bigtable: The ORIGINAL inspiration for this model (2006
paper) — powers many Google internal systems (search index,
Google Maps, Gmail) at massive scale.

USE CASES: Time-series data (sensor readings, metrics, logs —
each "row" = a device/metric, "columns" = timestamped readings),
activity feeds/event logs, write-heavy applications with simple
"by row key" access patterns at MASSIVE scale (billions of rows).
```

---

## 5. Graph Databases

### Core Intuition

Some data is FUNDAMENTALLY about RELATIONSHIPS — "who knows whom," "what depends on what," "which products are frequently bought together." In a relational database, representing and QUERYING deep relationships requires MANY JOINs — and the COST of a JOIN typically GROWS with each additional "hop" in the relationship chain.

Graph databases store data as **NODES** (entities) and **EDGES** (relationships between entities) NATIVELY — and are optimized specifically for TRAVERSING these relationships, regardless of how many "hops" deep.

```
GRAPH MODEL:

        ┌────────┐  FOLLOWS   ┌────────┐
        │  Yash   │ ─────────▶│ Priya   │
        └───┬────┘            └───┬────┘
            │ FOLLOWS              │ FOLLOWS
            ▼                      ▼
        ┌────────┐            ┌────────┐
        │  Rahul  │◀───────────│  Aman   │
        └────────┘   FOLLOWS   └────────┘

QUERY: "Find friends-of-friends of Yash who Yash doesn't
already follow" (a classic "people you may know" feature)

IN A RELATIONAL DATABASE (a `follows` table with columns
follower_id, followee_id):

SELECT DISTINCT f2.followee_id
FROM follows f1
JOIN follows f2 ON f1.followee_id = f2.follower_id
WHERE f1.follower_id = 'yash'
  AND f2.followee_id != 'yash'
  AND f2.followee_id NOT IN (
    SELECT followee_id FROM follows WHERE follower_id = 'yash'
  );

This is a 2-HOP query (already needs a self-JOIN). For a 3-HOP
or 4-HOP query ("friends of friends of friends..."), you'd need
ADDITIONAL self-joins — each one MULTIPLYING the rows being
processed, and performance DEGRADES RAPIDLY with each hop in a
relational database, especially as the graph grows large.

IN A GRAPH DATABASE (Cypher query language, Neo4j-style):

MATCH (yash:Person {name: 'Yash'})-[:FOLLOWS]->(:Person)
      -[:FOLLOWS]->(fof:Person)
WHERE NOT (yash)-[:FOLLOWS]->(fof) AND fof <> yash
RETURN DISTINCT fof

The graph database TRAVERSES the relationships DIRECTLY — each
node stores POINTERS to its connected edges, so "follow this
edge" is an O(1) operation (like following a pointer), regardless
of how many total nodes exist in the graph. A 2-hop or 10-hop
traversal has roughly LINEAR cost in the number of hops, NOT
the kind of multiplicative blowup relational JOINs experience.
```

### Use Cases and Real-World Examples

```
✅ Social networks (friend/follower graphs, "people you may know")
✅ Recommendation engines ("customers who bought X also bought Y" —
   a graph traversal: Product → Customers who bought it → Other
   products THEY bought)
✅ Fraud detection (relevant to BFSI!) — "is this new account
   connected, through a chain of shared devices/IPs/payment
   methods, to KNOWN FRAUDULENT accounts?" — a graph traversal
   problem that's extremely hard to express efficiently in SQL
✅ Knowledge graphs / dependency graphs (e.g., "what services
   depend on this service, transitively?" — relevant to
   understanding blast radius in microservices)
✅ AI/LLM context — relevant to your background: knowledge
   graphs are increasingly used alongside vector databases for
   RAG systems where relationships between entities matter (e.g.,
   "GraphRAG" approaches)

EXAMPLES: Neo4j (the most well-known dedicated graph database),
Amazon Neptune, ArangoDB (multi-model), and graph capabilities
increasingly built INTO other databases (PostgreSQL with
recursive CTEs can do SOME graph-like queries; Cosmos DB's
Gremlin API)
```

---

## 6. The Four Types — Side-by-Side Comparison

```
┌──────────────────┬────────────────┬────────────────┬────────────────────┬────────────────────┐
│ Type               │ Data Unit       │ Query By        │ Best For             │ Examples            │
├──────────────────┼────────────────┼────────────────┼────────────────────┼────────────────────┤
│ Key-Value          │ Opaque blob     │ Exact key only   │ Caching, sessions,   │ Redis, Memcached,   │
│                   │ per key         │ (O(1) lookup)    │ rate limiting,       │ etcd                │
│                   │                │                 │ distributed locks    │                     │
├──────────────────┼────────────────┼────────────────┼────────────────────┼────────────────────┤
│ Document           │ JSON/BSON       │ Any field,       │ Catalogs, profiles,  │ MongoDB, Couchbase, │
│                   │ document        │ including nested │ CMS, flexible-schema │ Firestore           │
│                   │                │ fields           │ app data             │                     │
├──────────────────┼────────────────┼────────────────┼────────────────────┼────────────────────┤
│ Column-Family       │ Sparse row with │ Row key (fast);  │ Time-series, activity│ Cassandra, HBase,   │
│ (Wide-Column)      │ many columns,   │ column-value     │ feeds, write-heavy   │ Bigtable            │
│                   │ grouped         │ scans are        │ massive-scale data   │                     │
│                   │ families        │ expensive        │                     │                     │
├──────────────────┼────────────────┼────────────────┼────────────────────┼────────────────────┤
│ Graph               │ Nodes + edges   │ Relationship     │ Social graphs,       │ Neo4j, Amazon       │
│                   │ (relationships) │ traversals (any  │ recommendations,      │ Neptune, ArangoDB   │
│                   │                │ depth)           │ fraud detection      │                     │
└──────────────────┴────────────────┴────────────────┴────────────────────┴────────────────────┘
```

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Using a key-value store for      │ Trying to query "by value" /     │ Re-evaluate — needs a document     │
│ data that needs querying by       │ secondary attributes that the    │ store (queryable fields) or a       │
│ non-key fields                    │ key-value model fundamentally     │ secondary index alongside the      │
│                                  │ can't support                     │ key-value store                    │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Unbounded document growth in a    │ Embedding a list that grows        │ Switch to referencing (separate    │
│ document store ("the document     │ without bound (e.g., embedding ALL │ collection) for unbounded lists,    │
│ that ate the database")           │ of a user's order history inside   │ as discussed in embedding vs        │
│                                  │ their user document)               │ referencing                         │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Column-family "query by column     │ Trying to run "find all rows       │ Design a SEPARATE table/column     │
│ value" scans the entire cluster    │ where column X = Y" — this model   │ family specifically for this query │
│                                  │ has no efficient secondary index   │ pattern (denormalize per access    │
│                                  │ by default                          │ pattern — common Cassandra practice)│
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Graph database used for           │ Graph traversal-optimized           │ Re-evaluate — bulk analytical       │
│ high-volume bulk analytical         │ databases aren't optimized for     │ workloads usually belong in a       │
│ queries (full scans, aggregations) │ scanning/aggregating ALL nodes      │ data warehouse (upcoming topic)     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "Hot partition" in Cassandra-like  │ A row key (e.g., a very active     │ Design row keys to spread load —   │
│ systems for a popular row key       │ user, or a time-bucketed key       │ bucket time-series data into        │
│                                  │ for "today")                       │ smaller time windows or composite   │
│                                  │                                  │ keys (connects to consistent        │
│                                  │                                  │ hashing hot-spot mitigation)        │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: What are the four main types of NoSQL databases and when would you use each?**
A: Key-value stores (Redis, DynamoDB) for simple, extremely fast exact-key lookups — caching, sessions, rate limiting. Document stores (MongoDB) for semi-structured data with varying shapes that need querying by fields, including nested ones — catalogs, user profiles, CMS. Column-family/wide-column stores (Cassandra, HBase) for massive-scale, write-heavy workloads where the dominant access pattern is "get everything for this row key," especially time-ordered data. Graph databases (Neo4j) for data that's fundamentally about relationships and requires multi-hop traversals — social networks, recommendations, fraud detection.

**Q: In a document database, when should you embed related data versus reference it in a separate collection?**
A: Embed when the related data is always accessed together with the parent, is "owned by" the parent (doesn't need to be shared or queried independently), and is bounded in size (won't grow unboundedly). Reference (store IDs, separate collection) when the data is large or grows unbounded (e.g., a user's complete order history), is shared across multiple parents, or needs to be queried independently of its parent.

**Q: Why are graph databases better than relational databases for multi-hop relationship queries?**
A: In a relational database, each "hop" in a relationship traversal requires an additional JOIN, and the cost of joins tends to grow significantly as the dataset and number of hops increase. Graph databases store relationships as direct pointers between nodes — traversing from one node to its connected nodes is an O(1) operation (follow the pointer), so a multi-hop traversal has roughly linear cost in the number of hops, regardless of the total size of the graph. This makes graph databases dramatically more efficient for queries like "friends of friends" or "find chains of connected fraudulent accounts."

**Q: Why is "design for your access pattern" especially emphasized for column-family stores like Cassandra?**
A: Column-family stores are extremely fast for "get all data for this row key" (data is co-located and sorted on disk by row key, then by column), but have no efficient way to query "find rows where some column equals X" — there's no secondary index by default. The standard practice (often called "query-driven design") is to create SEPARATE tables, each denormalized/structured specifically to serve ONE query pattern efficiently — accepting data duplication across these tables as the cost of making every important query a fast, single-row-key lookup.

---
---

# TOPIC 8: Database Normalization

---

## 1. What Problem Does Normalization Solve?

**Normalization** is a set of rules for organizing data in a RELATIONAL database to minimize REDUNDANCY (the same piece of information stored in multiple places) and avoid certain classes of DATA INTEGRITY problems that arise from that redundancy.

```
THE PROBLEM — AN UNNORMALIZED TABLE:

orders table:
┌────┬──────────┬───────────────┬─────────────┬──────────────┬─────────────┐
│ id │ user_name│ user_email     │ user_city   │ product_name │ product_price│
├────┼──────────┼───────────────┼─────────────┼──────────────┼─────────────┤
│ 1  │ Yash     │ yash@x.com     │ Pune        │ Laptop       │ 80000        │
│ 2  │ Yash     │ yash@x.com     │ Pune        │ Mouse        │ 500          │
│ 3  │ Priya    │ priya@x.com    │ Mumbai      │ Laptop       │ 80000        │
└────┴──────────┴───────────────┴─────────────┴──────────────┴─────────────┘

PROBLEMS WITH THIS DESIGN:

1. UPDATE ANOMALY: Yash changes his email. You must update BOTH
   row 1 AND row 2 (every row mentioning Yash). If you only
   update ONE row, the data becomes INCONSISTENT — "Yash" now
   has TWO different emails in the same table!

2. INSERT ANOMALY: You can't add a new product ("Keyboard",
   price 1500) to your catalog UNLESS someone places an order
   for it — the product's existence is ENTANGLED with order
   records. Want to pre-populate your catalog before any sales?
   You'd need to insert a "fake order" just to record the
   product's price.

3. DELETE ANOMALY: If Priya CANCELS her only order (row 3) and
   that row is DELETED, you've also LOST the information that
   "Laptop costs 80000" IF that was the only row mentioning it
   (in this tiny example it's duplicated, but in general,
   deleting one logical fact can accidentally delete OTHER
   facts bundled into the same row).

4. STORAGE WASTE: "Yash", "yash@x.com", "Pune", "Laptop", "80000"
   are repeated across multiple rows — at scale (millions of
   orders), this is a LOT of duplicated data.

NORMALIZATION SOLVES ALL FOUR by splitting this into MULTIPLE
TABLES, each representing ONE "entity" or "fact," linked by
foreign keys (recall the SQL vs NoSQL topic's introduction to
this).
```

---

## 2. The Normal Forms — Step by Step

### First Normal Form (1NF) — Atomic Values, No Repeating Groups

```
VIOLATES 1NF:
┌────┬───────┬─────────────────────────────┐
│ id │ name  │ phone_numbers                 │
├────┼───────┼─────────────────────────────┤
│ 1  │ Yash  │ "9876543210, 9876500000"       │  ← MULTIPLE VALUES
│ 2  │ Priya │ "9123456789"                   │     in ONE cell!
└────┴───────┴─────────────────────────────┘

PROBLEM: How do you query "find the user with phone number
9876500000"? You'd need string-parsing/LIKE queries — fragile
and slow (can't use a normal index efficiently).

1NF FIX: Each cell holds ONE atomic value. Repeating groups
become a SEPARATE TABLE (one row per phone number):

users:                    phone_numbers:
┌────┬───────┐            ┌────┬─────────┬────────────┐
│ id │ name  │            │ id │ user_id │ phone       │
├────┼───────┤            ├────┼─────────┼────────────┤
│ 1  │ Yash  │            │ 1  │ 1       │ 9876543210  │
│ 2  │ Priya │            │ 2  │ 1       │ 9876500000  │
└────┴───────┘            │ 3  │ 2       │ 9123456789  │
                           └────┴─────────┴────────────┘

Now "find user with phone 9876500000" is a simple, indexable
equality lookup on phone_numbers.phone.
```

### Second Normal Form (2NF) — No Partial Dependencies (on composite keys)

```
2NF applies to tables with a COMPOSITE PRIMARY KEY (made of
multiple columns). It requires that EVERY non-key column depends
on the ENTIRE composite key, not just PART of it.

VIOLATES 2NF:
order_items table, PRIMARY KEY = (order_id, product_id)
┌──────────┬────────────┬──────────┬───────────────┬───────────────┐
│ order_id │ product_id │ quantity │ product_name   │ product_price │
├──────────┼────────────┼──────────┼───────────────┼───────────────┤
│ 101      │ P1         │ 2        │ Laptop          │ 80000          │
│ 101      │ P2         │ 1        │ Mouse           │ 500            │
│ 102      │ P1         │ 1        │ Laptop          │ 80000          │
└──────────┴────────────┴──────────┴───────────────┴───────────────┘

PROBLEM: `product_name` and `product_price` depend ONLY on
`product_id` (PART of the composite key), NOT on the FULL
(order_id, product_id) combination. This is a "PARTIAL
DEPENDENCY" — and it causes the SAME update/insert/delete
anomalies as before (product price duplicated across every order
that includes it).

2NF FIX: Move product_name/product_price to their OWN `products`
table, keyed by product_id alone:

order_items:                    products:
┌──────────┬────────────┬──────────┐  ┌────────────┬───────────────┬───────────────┐
│ order_id │ product_id │ quantity │  │ product_id │ product_name   │ product_price │
├──────────┼────────────┼──────────┤  ├────────────┼───────────────┼───────────────┤
│ 101      │ P1         │ 2        │  │ P1         │ Laptop          │ 80000          │
│ 101      │ P2         │ 1        │  │ P2         │ Mouse           │ 500            │
│ 102      │ P1         │ 1        │  └────────────┴───────────────┴───────────────┘
└──────────┴────────────┴──────────┘
```

### Third Normal Form (3NF) — No Transitive Dependencies

```
3NF requires that non-key columns depend ONLY on the primary key
— NOT on OTHER non-key columns ("transitive dependency").

VIOLATES 3NF:
employees table:
┌────┬───────┬──────────────┬─────────────────┐
│ id │ name  │ department_id │ department_name │
├────┼───────┼──────────────┼─────────────────┤
│ 1  │ Yash  │ D1            │ Engineering      │
│ 2  │ Priya │ D1            │ Engineering      │
│ 3  │ Rahul │ D2            │ Sales            │
└────┴───────┴──────────────┴─────────────────┘

PROBLEM: `department_name` depends on `department_id`, which
itself is just a non-key column (not the primary key `id`). This
is a TRANSITIVE dependency: id → department_id → department_name.

If "D1" is renamed from "Engineering" to "Product Engineering",
you must update EVERY employee row with department_id='D1' — the
SAME update anomaly as before, one level removed.

3NF FIX: Separate `departments` into its OWN table:

employees:                  departments:
┌────┬───────┬──────────────┐  ┌──────────────┬─────────────────┐
│ id │ name  │ department_id │  │ department_id │ department_name │
├────┼───────┼──────────────┤  ├──────────────┼─────────────────┤
│ 1  │ Yash  │ D1            │  │ D1            │ Engineering      │
│ 2  │ Priya │ D1            │  │ D2            │ Sales            │
│ 3  │ Rahul │ D2            │  └──────────────┴─────────────────┘
└────┴───────┴──────────────┘

Now renaming "Engineering" → "Product Engineering" requires
changing exactly ONE row.

MOST PRODUCTION SCHEMAS AIM FOR 3NF — it eliminates the most
common/practical sources of redundancy and anomalies. Higher
normal forms (BCNF, 4NF, 5NF) exist but address increasingly
rare/theoretical edge cases and are less commonly discussed in
interviews.
```

---

## 3. The Tradeoff — Normalization vs Denormalization

This is where this topic CONNECTS BACK to NoSQL and is the crux of "when should I normalize vs denormalize?"

```
NORMALIZATION (more tables, JOINs to reassemble):

✅ NO DATA DUPLICATION — each fact stored ONCE, update it ONCE
✅ STRONG CONSISTENCY GUARANTEES — impossible for "Yash's email"
   to disagree with itself across rows, because there's only
   ONE ROW with Yash's email
✅ SMALLER STORAGE FOOTPRINT

❌ READS REQUIRE JOINS — "get order with product details" needs
   to JOIN orders + order_items + products = MULTIPLE TABLE
   LOOKUPS (even if all on one server, this has overhead; recall
   from the Sharding topic, JOINs become VERY expensive or
   impossible across shards)

DENORMALIZATION (duplicate data deliberately, avoid JOINs):

✅ READS ARE FAST — a single lookup (one table, or one document
   in NoSQL) returns EVERYTHING needed
✅ Enables horizontal scaling patterns (no cross-shard JOINs
   needed) — this is WHY NoSQL document stores often look
   "denormalized" compared to SQL schemas (recall the
   embedding example from the NoSQL Types topic)

❌ DATA DUPLICATION — the same fact (e.g., a product's name)
   might be copied into THOUSANDS of order documents
❌ UPDATE COMPLEXITY — if a product's name changes, do you need
   to update it everywhere it's duplicated? (Sometimes the
   answer is "no, historical orders should show the name AS IT
   WAS AT THE TIME" — which is actually a FEATURE of
   denormalization for certain use cases! An order should show
   "Laptop - ₹80,000" even if the product is later renamed or
   re-priced — this is "point-in-time" data, not "live reference"
   data.)
❌ RISK OF INCONSISTENCY — if an update SHOULD propagate
   everywhere but a bug/partial-failure means it only updates
   SOME copies
```

---

## 4. The Practical Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│ NORMALIZE WHEN:                                                   │
│ - Data changes FREQUENTLY and must stay CONSISTENT everywhere     │
│   (e.g., a user's profile information — name, email)              │
│ - The system is primarily WRITE-validating / transactional        │
│   (recall ACID topic — normalization + foreign keys + ACID work   │
│   together to guarantee integrity)                                 │
│ - Storage efficiency matters (less relevant with modern storage    │
│   costs, but still relevant at extreme scale)                      │
│                                                                     │
│ DENORMALIZE WHEN:                                                   │
│ - READ PERFORMANCE is the priority and JOINs are the bottleneck    │
│   (recall the 100:1 read:write ratios from Back-of-Envelope         │
│   Estimation — optimizing the 100 is often more impactful)         │
│ - The data represents a POINT-IN-TIME SNAPSHOT (order details       │
│   AS THEY WERE when the order was placed — "Laptop, ₹80,000",      │
│   even if the product is later discontinued/repriced)               │
│ - You're sharding (Topic 3) and need to avoid cross-shard JOINs     │
│ - Using a NoSQL document/column-family model (Topic 7) where        │
│   the data model itself encourages "one document per access         │
│   pattern"                                                          │
│                                                                     │
│ COMMON HYBRID APPROACH: Normalize the "source of truth" (e.g., a    │
│ SQL database with `products` table), but maintain DENORMALIZED      │
│ READ-OPTIMIZED COPIES (in a cache, search index, or NoSQL store)    │
│ that are kept "eventually consistent" via async updates — this is  │
│ effectively CQRS (Command Query Responsibility Segregation):        │
│ writes go to the normalized source of truth; reads are served        │
│ from denormalized projections optimized for each access pattern.    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Real-World Usage

**Traditional ERP/banking systems:** Heavily normalized (often 3NF or beyond) — financial data integrity (recall ACID) depends on each fact existing in exactly one place, with foreign keys enforcing relationships. A customer's address exists in ONE table; every system that needs it JOINS to that table.

**E-commerce order records (a famous example of intentional denormalization):** When an order is placed, the product's name, price, and other details are typically COPIED into the order record AT THAT MOMENT — even though `products` is a normalized table elsewhere. This is denormalization used CORRECTLY: the order is a historical record of "what was purchased, at what price, at that time" — it should NOT change if the product is later renamed or repriced. Normalizing this (just storing a product_id and JOINing) would mean old orders show CURRENT product info, which is WRONG for a receipt/invoice.

**Read-heavy social media feeds:** The underlying "source of truth" for a post might be normalized (post content in one table, author info in `users`, like counts computed from a `likes` table). But the FEED a user sees is often a heavily DENORMALIZED, pre-computed view — each feed entry might include a COPY of the author's name/avatar at the time of caching, refreshed periodically, rather than JOINing live on every feed load (recall the "fan-out" discussion from Back-of-Envelope Estimation in Scalability notes).

---

## 6. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "Update anomaly" bug — same       │ Under-normalized schema; a       │ Normalize to 3NF for data that     │
│ logical entity's data disagrees   │ fact is duplicated and only       │ must stay consistent; if            │
│ across rows                       │ SOME copies were updated          │ denormalized intentionally,         │
│                                  │                                  │ ensure ALL update paths are aware  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Over-normalized schema causes      │ Every query requires many JOINs   │ Selectively denormalize for the    │
│ slow reads at scale                │ across many small tables           │ specific read-heavy queries        │
│                                  │                                  │ (covering indexes from Indexing    │
│                                  │                                  │ topic, materialized views, or       │
│                                  │                                  │ CQRS-style read projections)        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Denormalized historical record     │ A "live reference" was used        │ For point-in-time records (orders, │
│ incorrectly shows CURRENT data     │ instead of a "snapshot copy" —     │ invoices, contracts), explicitly    │
│ instead of data-as-it-was          │ e.g., an invoice JOINs to current  │ COPY the relevant data at creation │
│                                  │ product price instead of storing   │ time rather than referencing live   │
│                                  │ the price AT PURCHASE TIME          │ data                                │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Denormalized cache/projection      │ Async propagation from the         │ Monitor staleness; for critical    │
│ becomes stale/inconsistent with     │ normalized source of truth has     │ paths, allow reading from the       │
│ source of truth                    │ lag or a failure that wasn't        │ source of truth as a fallback, or   │
│                                  │ retried                            │ use change-data-capture (CDC) with  │
│                                  │                                  │ guaranteed delivery for propagation│
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 7. Interview Quick-Fire Answers

**Q: What problems does normalization solve, with an example?**
A: Normalization eliminates redundant data storage and the anomalies that come with it. If a customer's email is duplicated across every order row, updating it requires changing ALL those rows (update anomaly) — miss one and you have inconsistent data. Normalization splits this into a `customers` table (email stored once) and an `orders` table referencing it via foreign key — updating the email means changing exactly one row, and all orders automatically reflect the current email via JOIN.

**Q: What's the difference between 2NF and 3NF?**
A: 2NF addresses "partial dependencies" in tables with composite primary keys — every non-key column must depend on the ENTIRE composite key, not just part of it (e.g., in an order_items table keyed by (order_id, product_id), product_name depends only on product_id, so it should move to a separate products table). 3NF addresses "transitive dependencies" — non-key columns shouldn't depend on OTHER non-key columns (e.g., in an employees table, department_name depends on department_id, which is itself just a non-key attribute, so department_name should move to a separate departments table).

**Q: When is denormalization the RIGHT choice, even though it duplicates data?**
A: Two main cases: (1) Read performance — when JOINs become a bottleneck, especially at scale or across shards where cross-shard JOINs are expensive or impossible, denormalizing so a single lookup returns everything needed can be essential. (2) Point-in-time/historical records — an order or invoice SHOULD capture product details "as they were" at the time of the transaction, not reflect current (possibly changed) data; this is denormalization used correctly, not a shortcut.

**Q: How do normalization and NoSQL document modeling relate to each other?**
A: They represent opposite ends of a spectrum, and the "right" choice depends on access patterns (as covered in SQL vs NoSQL and NoSQL Types). Normalized relational schemas optimize for write consistency and avoiding redundancy, at the cost of requiring JOINs to assemble related data for reads. NoSQL document models (and the "embedding" pattern specifically) often look like deliberately DENORMALIZED relational schemas — embedding related data so a single document read avoids JOINs entirely. A common hybrid (CQRS) keeps a normalized source of truth (often SQL) for writes/consistency, while maintaining denormalized read-optimized projections (often NoSQL/cache) for fast reads — propagated asynchronously.


---
---

# TOPIC 9: Object Storage

---

## 1. What Problem Does Object Storage Solve?

Everything covered so far (SQL, NoSQL types) deals with **structured or semi-structured data** — rows, documents, columns — designed to be QUERIED. But applications also need to store **unstructured BLOBS**: images, videos, PDFs, audio files, application logs, ML model files, backups, data lake files.

```
WHY NOT STORE THESE IN A DATABASE?

A 50MB video file stored as a BLOB column in PostgreSQL:
- Every database BACKUP now includes ALL video data (huge backups)
- The database's WAL/replication (Topic 4!) must now replicate
  this 50MB blob alongside every other write — massively
  increasing replication traffic for unrelated transactions
- Database connection pools, memory caches, and indexes are
  optimized for small, structured rows — not multi-MB/GB blobs
- The database's STORAGE ENGINE (B+ Trees, MVCC versioning from
  the ACID/Indexing topics) is OVERKILL and INEFFICIENT for
  "store this 2GB file, retrieve it later, rarely if ever update
  it in place"

WHY NOT STORE THESE AS FILES ON A SERVER'S DISK?

- A single server's disk has a SIZE LIMIT (recall vertical
  scaling's ceiling from Scalability notes)
- If that server dies, the files are GONE (no built-in
  replication/durability)
- Doesn't scale horizontally — "which server has this file?"
  becomes its own routing/sharding problem (recall Sharding topic)
- No built-in CDN integration, access control, versioning
```

**Object storage** (S3, Google Cloud Storage, Azure Blob Storage) is purpose-built for this: a flat namespace of "objects" (files), each identified by a unique KEY (like a path/filename), stored durably across many machines/data centers automatically, accessible via simple HTTP APIs (GET/PUT/DELETE — recall the HTTP methods from Networking Fundamentals!), at MASSIVE scale (petabytes+, as seen in the Back-of-Envelope Estimation media storage example).

---

## 2. Core Intuition — The "Object" Model

```
TRADITIONAL FILESYSTEM (hierarchical):
/photos/2026/06/vacation/IMG_001.jpg
/photos/2026/06/vacation/IMG_002.jpg
/photos/2026/07/birthday/IMG_001.jpg
   └── directories are REAL structures; "moving" a file between
       directories is a metadata operation; deeply nested paths
       have real filesystem implications (inode lookups, etc.)

OBJECT STORAGE (flat namespace, "folders" are an ILLUSION):
Bucket: "my-photos"
  Object key: "2026/06/vacation/IMG_001.jpg"
  Object key: "2026/06/vacation/IMG_002.jpg"
  Object key: "2026/07/birthday/IMG_001.jpg"

THERE ARE NO REAL "DIRECTORIES" — each object is stored
independently, keyed by its FULL PATH STRING. The "/" characters
are just PART OF THE KEY STRING. The UI/API might DISPLAY these
as folders (grouping by common prefixes), but internally, it's
a flat key-value-like store where:
- KEY = the object's path string (e.g., "2026/06/vacation/IMG_001.jpg")
- VALUE = the file's bytes (the "object")
- + METADATA (content-type, size, last-modified, custom tags,
  access permissions)

THIS IS WHY: object storage is sometimes described as "a
key-value store for large blobs" (recall Key-Value Stores from
the NoSQL Types topic — conceptually related, but object storage
is optimized for LARGE values — MBs to TBs — and DURABILITY at
extreme scale, not low-latency small-value access).
```

---

## 3. The API — Simple HTTP Operations

```
PUT /my-bucket/2026/06/vacation/IMG_001.jpg HTTP/1.1
Content-Type: image/jpeg
Content-Length: 2458624

[binary image data]

→ Stores the object. The object storage system handles:
  - Splitting the data and replicating it across multiple
    physical disks/servers/data centers (durability)
  - Computing a checksum for integrity verification
  - Storing metadata (Content-Type, size, timestamp)


GET /my-bucket/2026/06/vacation/IMG_001.jpg HTTP/1.1

→ Returns the object's bytes + metadata as HTTP headers
  (Content-Type: image/jpeg, Content-Length: 2458624, ETag: ...)

NOTICE: This is the SAME HTTP vocabulary from Networking
Fundamentals — GET, PUT, Content-Type, ETag (for caching/
conditional requests), Content-Length. Object storage is
literally "a giant, distributed, durable web server for files,"
accessible directly over HTTP/HTTPS.

THIS IS WHY object storage integrates SO NATURALLY with CDNs
(Networking Fundamentals topic) — a CDN can cache objects
directly from an S3 bucket as its "origin," using the exact
same caching mechanisms (Cache-Control, ETag, conditional GETs)
covered there.
```

---

## 4. Durability — How "11 Nines" Is Achieved

```
AWS S3 advertises "99.999999999% durability" (11 nines) — meaning
if you store 10,000,000 objects, you'd statistically expect to
lose ONE object roughly every 10,000 YEARS.

HOW THIS IS ACHIEVED — ERASURE CODING (a more storage-efficient
alternative to simple full replication):

SIMPLE REPLICATION (recall Replication topic):
Store 3 FULL COPIES of each object across 3 different
disks/racks/data centers. Storage overhead: 3x (200% extra).

ERASURE CODING (used by most large object stores):
Split the object into N data fragments, and compute M additional
PARITY fragments (using algorithms like Reed-Solomon — similar
mathematical family to RAID 6, error-correcting codes used in
CDs/DVDs, and network error correction).

Store all (N + M) fragments across DIFFERENT disks/racks/data
centers. The object can be RECONSTRUCTED from ANY N of the
(N + M) fragments — meaning up to M fragments can be LOST
(disk failures, rack failures) and the data is still recoverable.

EXAMPLE: N=10, M=4 (a common ratio)
Storage overhead: (10+4)/10 = 1.4x (40% extra) — MUCH better
than 3x replication's 200% overhead, while tolerating UP TO 4
simultaneous fragment failures.

THIS IS WHY object storage can offer extreme durability at a
LOWER cost-per-GB than naive triple-replication — the math of
erasure coding provides similar (often better) fault tolerance
with significantly less storage overhead.
```

---

## 5. Consistency Model

```
HISTORICALLY: S3 (and similar) offered "eventual consistency"
for certain operations — e.g., after uploading a NEW object, a
LIST operation might not immediately show it (this connects
DIRECTLY to the CAP theorem / AP-leaning systems discussed
earlier — object storage at this scale faces the same
distributed-systems tradeoffs).

MODERN OBJECT STORES (AWS S3 since Dec 2020, GCS, Azure Blob)
now offer STRONG READ-AFTER-WRITE CONSISTENCY for most
operations — a PUT followed immediately by a GET for the same
key will return the new data. This was a SIGNIFICANT engineering
achievement (AWS published details on the internal
re-architecture required) — directly illustrating that the
AP-vs-CP tradeoff ISN'T FIXED FOREVER; with enough engineering
investment (recall Google Spanner from the CAP topic), systems
can move toward stronger guarantees over time.

HOWEVER: For OVERWRITES and DELETES, there can still be brief
windows where different requests might see old vs new versions
during the propagation — application design should generally
avoid relying on IMMEDIATE global consistency for HIGH-FREQUENCY
overwrites of the same key (use versioned keys / new objects
instead — connects to the "immutable, content-hashed filenames"
pattern from the CDN topic in Networking Fundamentals).
```

---

## 6. Storage Classes / Tiering

```
Object storage systems offer multiple STORAGE TIERS, trading
ACCESS LATENCY/COST for STORAGE COST — a direct application of
the memory/disk/network latency-vs-cost tradeoffs from the
Back-of-Envelope Estimation topic, but at the "cold storage"
end of the spectrum.

┌──────────────────────┬──────────────────┬─────────────────────┬──────────────────────┐
│ Tier                   │ Access Latency     │ Storage Cost          │ Use Case               │
├──────────────────────┼──────────────────┼─────────────────────┼──────────────────────┤
│ Standard               │ Milliseconds       │ Highest                │ Actively-accessed       │
│                       │ (immediate)        │                       │ content (product images,│
│                       │                   │                       │ user uploads)           │
│ Infrequent Access       │ Milliseconds       │ Lower (but per-GET     │ Backups, older content   │
│                       │ (immediate)        │ retrieval fee)         │ accessed occasionally    │
│ Archive (Glacier-like)  │ Minutes to HOURS   │ Lowest                 │ Compliance archives,     │
│                       │ (must "restore"     │                       │ data you legally must    │
│                       │ before access)      │                       │ retain but rarely read   │
└──────────────────────┴──────────────────┴─────────────────────┴──────────────────────┘

LIFECYCLE POLICIES automate moving objects between tiers:
"After 30 days, move to Infrequent Access. After 1 year, move
to Archive. After 7 years, delete." — common for compliance/
audit log retention (relevant to BFSI: RBI/regulatory retention
requirements for transaction records often specify multi-year
retention, where Archive-tier storage makes the cost
manageable for data that's "must keep, rarely access").
```

---

## 7. Real-World Usage

**Netflix:** Stores ALL their video content (the master files, before CDN distribution — recall the Netflix Open Connect discussion from the CDN topic) in S3. Encoding pipelines READ from S3, produce multiple bitrate/format versions, WRITE results back to S3, which are then distributed to Open Connect appliances.

**Data Lakes (analytics):** Companies dump raw data (logs, events, exports from various databases) into object storage in formats like Parquet/ORC, then run analytical query engines (Presto, Spark, Athena) DIRECTLY against this data without first loading it into a traditional database — object storage acts as the foundation for modern "data lake" and "data lakehouse" architectures (connects to the Data Warehousing topic).

**ML/AI model storage (relevant to your GenAI background):** Trained model weights (often GBs to 100s of GBs for large models) are stored in object storage, downloaded by inference servers at startup or on-demand. Training pipelines read datasets FROM and write checkpoints TO object storage — a clean separation between COMPUTE (ephemeral, auto-scaled — recall Auto-scaling topic) and STORAGE (durable, persists independent of compute lifecycle).

**Static website/CDN origin:** As discussed in the CDN topic — object storage commonly serves as the ORIGIN for CDN-distributed static assets (images, JS/CSS bundles, downloadable files) — S3 + CloudFront is one of the most common patterns in cloud architecture.

---

## 8. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Application reads STALE data      │ Relying on immediate              │ Use versioned/content-hashed keys │
│ after an overwrite                 │ consistency for OVERWRITES of      │ for frequently-updated content     │
│                                  │ the same key, hitting a propagation│ (new key per version, never        │
│                                  │ window edge case                   │ overwrite — recall CDN topic)       │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Unexpectedly high costs            │ Frequent small reads/writes        │ Understand the pricing model       │
│                                  │ against Infrequent Access/Archive   │ (per-request AND per-GB costs);    │
│                                  │ tiers — designed for INFREQUENT     │ choose tiers based on ACTUAL        │
│                                  │ access, penalized for frequent use  │ access patterns, not just "cheapest│
│                                  │                                  │ per-GB" in isolation                │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "Hot prefix" throttling             │ Many requests to keys sharing a    │ Use varied/randomized key prefixes │
│                                  │ common PREFIX (e.g., all keys       │ (e.g., hash-prefix the key) to     │
│                                  │ starting "2026-06-13-...") can hit  │ distribute load across the          │
│                                  │ per-prefix request-rate limits      │ underlying partitions               │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Accidental public exposure of      │ Misconfigured bucket/object         │ Default-deny access policies;      │
│ sensitive data                     │ permissions (a very common, high-  │ automated scanning for public      │
│                                  │ profile real-world incident type)   │ buckets; encryption at rest         │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Large object upload fails           │ Network interruption during a       │ Use MULTIPART UPLOAD — split large │
│ partway through                    │ single huge PUT request             │ objects into chunks, upload         │
│                                  │                                  │ independently (parallel, retryable │
│                                  │                                  │ per-chunk), then combine            │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 9. Interview Quick-Fire Answers

**Q: Why wouldn't you store large files (images, videos) directly in a relational database?**
A: Storing large blobs in a database means every backup, replication stream, and the database's storage engine (designed for small structured rows, B+ Tree indexes, MVCC) has to handle these large objects — bloating backups, increasing replication traffic, and being generally inefficient. Object storage is purpose-built for this: a flat namespace of objects accessible via simple HTTP, with automatic distributed replication/erasure coding for durability, separate from your transactional database entirely.

**Q: How does object storage achieve extreme durability (e.g., "11 nines") cost-effectively?**
A: Primarily through erasure coding — splitting an object into N data fragments plus M parity fragments (computed via Reed-Solomon-like algorithms), then storing all N+M fragments across different disks/racks/data centers. The object can be reconstructed from any N of the N+M fragments, tolerating up to M simultaneous failures, with much lower storage overhead (e.g., 40% extra for N=10,M=4) compared to simple triple-replication (200% extra).

**Q: What's the relationship between object storage and CDNs?**
A: Object storage commonly serves as the ORIGIN for CDN-distributed content. Both speak the same HTTP vocabulary (GET/PUT, Content-Type, ETag, Cache-Control), so a CDN can cache objects directly from a bucket as if it were any other web origin. This is one of the most common cloud architecture patterns — object storage for durable origin storage, CDN for global edge caching/delivery (as covered in the Networking Fundamentals CDN topic).

**Q: How would you handle storing years of compliance/audit data cost-effectively?**
A: Use lifecycle policies to automatically tier data based on age and access patterns — recent data in Standard tier (frequent access), older data moved to Infrequent Access after some period, and data that must be retained for regulatory reasons but is essentially never accessed moved to Archive tier (lowest cost, but retrieval takes minutes-to-hours). This directly maps storage cost to actual access patterns rather than paying premium rates for data that's "must keep, rarely touch" — relevant for RBI/regulatory retention requirements in fintech.

---
---

# TOPIC 10: Time-series Databases

---

## 1. What Problem Does a Time-series Database Solve?

**Time-series data** is data points indexed by TIME — server metrics (CPU usage every 10 seconds), IoT sensor readings, stock prices, application logs, user activity events. This data has VERY DISTINCTIVE characteristics that general-purpose databases handle poorly at scale:

```
CHARACTERISTICS OF TIME-SERIES DATA:

1. EXTREMELY HIGH WRITE VOLUME, APPEND-ONLY
   A fleet of 10,000 servers, each reporting 50 metrics every
   10 seconds = 10,000 × 50 / 10 = 50,000 writes/second, 24/7,
   FOREVER. Data is essentially NEVER updated after being
   written (recall the "write-heavy, no secondary indexes
   needed" column-family discussion from NoSQL Types).

2. QUERIES ARE ALMOST ALWAYS TIME-RANGE-BASED
   "Show CPU usage for server X over the last 24 hours."
   "What was the average response time per minute, for the
   last hour, across all servers in region Y?"
   Range-based, time-ordered queries are THE dominant pattern
   (recall range queries from the Indexing and Sharding topics
   — this STRONGLY favors range-friendly storage layouts).

3. DATA HAS NATURAL "AGING" — RECENT DATA IS QUERIED OFTEN,
   OLD DATA RARELY (BUT MIGHT NEED RETENTION FOR COMPLIANCE)
   "What's CPU usage RIGHT NOW / in the last hour" is queried
   constantly (dashboards, alerting). "What was CPU usage on
   March 3rd, 2024 at 2:17am" is queried RARELY — but might
   need to be RETAINED for a year for trend analysis or
   compliance (recall the Object Storage tiering discussion —
   the SAME aging principle applies here).

4. DOWNSAMPLING / AGGREGATION IS NATURAL AND EXPECTED
   Nobody needs PER-SECOND CPU data from 6 months ago — a
   per-hour AVERAGE is sufficient. Time-series systems often
   AUTOMATICALLY downsample old data (keep raw 10-second data
   for 7 days, 1-minute aggregates for 30 days, 1-hour
   aggregates for 1 year) — trading PRECISION for STORAGE
   SAVINGS on data that's unlikely to be queried at full
   resolution again.
```

---

## 2. Why General-Purpose Databases Struggle With This Workload

```
USING A REGULAR RELATIONAL DATABASE FOR METRICS:

CREATE TABLE metrics (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    server_id INT,
    metric_name VARCHAR(50),
    value DOUBLE,
    timestamp TIMESTAMP
);
CREATE INDEX idx_server_time ON metrics (server_id, timestamp);

PROBLEM 1: WRITE THROUGHPUT
50,000 inserts/second means 50,000 B+ Tree updates/second
(recall Indexing topic — EVERY index must be updated on every
write). Even with ONE composite index, this is an enormous
sustained write load — B+ Trees experience "page splits" under
heavy sequential-key insertion, and the auto-increment primary
key index itself becomes a write bottleneck (all new rows have
ever-increasing IDs — similar to the "hot last shard" problem
from the Sharding topic, but now it's a "hot last B+ Tree page").

PROBLEM 2: STORAGE OVERHEAD
Each row stores: id (8 bytes) + server_id (4) + metric_name
(string, maybe 20+ bytes) + value (8) + timestamp (8) + ROW
OVERHEAD (20-50 bytes per the Back-of-Envelope numbers) ≈ 70-100
bytes per data point. At 50,000 points/second:
50,000 × 86,400 × 100 bytes ≈ 432 GB/day — and this grows
WITHOUT BOUND.

PROBLEM 3: NO BUILT-IN DOWNSAMPLING/RETENTION
The database has no concept of "automatically aggregate data
older than 30 days into hourly averages and discard the raw
points" — this must be built manually (scheduled jobs that
delete/aggregate — themselves consuming significant resources
on a huge, write-heavy table).

TIME-SERIES DATABASES (TSDBs) ARE PURPOSE-BUILT TO SOLVE ALL
THREE OF THESE PROBLEMS NATIVELY.
```

---

## 3. Core TSDB Techniques

### Technique 1: Column-Oriented Storage + Compression

```
Most TSDBs store data COLUMN-BY-COLUMN (not row-by-row like
traditional RDBMS) for each TIME SERIES (a unique combination
of metric name + "tags"/labels, e.g.,
"cpu_usage{server=web-01, region=us-east}").

ROW-ORIENTED (traditional):
(t1, server=web-01, cpu=45.2)
(t2, server=web-01, cpu=46.1)
(t3, server=web-01, cpu=45.8)

COLUMN-ORIENTED (TSDB):
timestamps: [t1, t2, t3, ...]
values:     [45.2, 46.1, 45.8, ...]

WHY THIS ENABLES MASSIVE COMPRESSION:
- TIMESTAMPS in a series are often EVENLY SPACED (every 10
  seconds) — can be stored as a SINGLE START TIME + INTERVAL,
  or DELTA-ENCODED (store the DIFFERENCE between consecutive
  values, which is often a CONSTANT — compresses to almost
  nothing)
- VALUES often change GRADUALLY (CPU usage doesn't jump from
  10% to 95% between consecutive 10-second samples) — DELTA
  encoding + specialized floating-point compression (e.g.,
  Facebook's "Gorilla" compression algorithm, used in many
  TSDBs) can achieve 10-20x compression compared to storing
  raw doubles

REAL IMPACT: The "432 GB/day" estimate above can often be
reduced to 20-40 GB/day with TSDB-specific compression — a
10-20x reduction, directly addressing Problem 2 from above.
```

### Technique 2: Time-Based Partitioning

```
Data is automatically partitioned into CHUNKS based on TIME
RANGES (e.g., one chunk per hour, or per day) — conceptually
similar to RANGE-BASED SHARDING (Topic 3) but the "shard key"
is TIME itself.

┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Chunk:       │ │ Chunk:       │ │ Chunk:       │ │ Chunk:       │
│ June 11      │ │ June 12      │ │ June 13      │ │ (current,    │
│ (immutable,   │ │ (immutable,   │ │ (immutable,   │ │  being       │
│ compressed)  │ │ compressed)  │ │ compressed)  │ │  written)     │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘

BENEFITS:
✅ A query for "the last 24 hours" only needs to touch the
   LAST 1-2 CHUNKS — older chunks aren't even opened (this is
   the time-series equivalent of the BRIN index mentioned in
   the Indexing topic for naturally-ordered data)
✅ RETENTION POLICIES become trivial: "delete data older than
   90 days" = just DELETE entire old CHUNK FILES — an O(1)
   filesystem operation, NOT a row-by-row DELETE (which would
   be extremely slow on a huge table, and cause the "table
   bloat"/MVCC cleanup issues from the ACID topic)
✅ The CURRENT (most recent) chunk is the ONLY one being
   actively written to — older chunks are IMMUTABLE and can be
   AGGRESSIVELY COMPRESSED once "closed"
```

### Technique 3: Automatic Downsampling / Rollups

```
CONFIGURATION EXAMPLE (conceptual):

RAW DATA (10-second resolution): retain for 7 days
1-MINUTE AGGREGATES (avg/min/max/p99): retain for 30 days
1-HOUR AGGREGATES: retain for 1 year

A BACKGROUND PROCESS continuously computes these aggregates from
raw data and stores them as SEPARATE, SMALLER time series. After
the retention window, the FINER-GRAINED data is deleted (via the
chunk-deletion mechanism above), but the AGGREGATED version
remains.

QUERY ROUTING: A query for "CPU usage over the last 6 hours" uses
RAW data (still within the 7-day raw retention). A query for "CPU
usage over the last 6 months" automatically uses the 1-HOUR
AGGREGATES (raw data for that period no longer exists) — the TSDB
handles this transparently, the query just specifies a time range.

THIS DIRECTLY ADDRESSES Problem 3 and CONNECTS to the Object
Storage tiering discussion — same underlying principle (match
storage/precision to actual access patterns over time), applied
to time-series data specifically.
```

---

## 4. Real-World Usage

**Prometheus (the dominant open-source monitoring TSDB, heavily used in Kubernetes ecosystems — relevant to your container orchestration background):** Uses a custom storage engine with the time-based chunking and Gorilla-style compression described above. The data model is "metric name + key-value labels" (e.g., `http_requests_total{method="GET", status="200", service="user-api"}`) — each unique label combination is its OWN time series. PromQL (its query language) is built around time-range and aggregation operations (`rate()`, `avg_over_time()`, etc.) — NOT general-purpose SQL JOINs.

**InfluxDB:** Another widely-used TSDB, with a SQL-like query language (InfluxQL/Flux), commonly used for IoT sensor data, application metrics, and financial tick data — explicitly supporting retention policies and continuous queries (automatic downsampling) as first-class features.

**Time-series workloads on general-purpose databases:** PostgreSQL with the TimescaleDB extension adds time-based partitioning ("hypertables") and compression on top of standard PostgreSQL — a good example of a hybrid approach: get TSDB-style storage efficiency while retaining full SQL/JOIN capability for combining time-series data with relational metadata (e.g., joining sensor readings with a `devices` table containing device metadata).

**Financial tick data (relevant to BFSI):** Stock exchanges and trading platforms generate enormous volumes of time-stamped price/volume data (every trade, every quote update) — TSDBs (or TSDB-like custom systems) are used for both real-time analysis (recent data, low latency) and historical backtesting (years of compressed historical data).

---

## 5. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "Cardinality explosion" —         │ Each UNIQUE combination of        │ Be deliberate about which          │
│ TSDB runs out of memory/becomes   │ labels/tags creates a NEW time     │ attributes become LABELS (low       │
│ extremely slow                    │ series; if a label has HIGH        │ cardinality: server_id, region,    │
│                                  │ CARDINALITY (e.g., user_id, or     │ status_code) vs stored as VALUES   │
│                                  │ a UUID per request) → MILLIONS of  │ in the metric itself (high          │
│                                  │ distinct time series are created   │ cardinality data shouldn't be a    │
│                                  │                                  │ label — use logs/traces instead)    │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Out-of-order writes (data          │ Network delays/buffering mean      │ Most TSDBs accept SOME out-of-order│
│ arrives late, out of timestamp     │ a data point with an OLDER          │ writes within a bounded window;     │
│ order)                            │ timestamp arrives AFTER newer       │ design ingestion pipelines to       │
│                                  │ ones — can complicate the           │ minimize extreme delays              │
│                                  │ "immutable closed chunk" model      │                                     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Query for a long time range        │ Querying RAW (high-resolution)     │ Use the appropriate downsampled     │
│ ("last 1 year") is extremely slow │ data over a huge time range,        │ aggregate for long ranges (the      │
│ or runs out of memory              │ scanning/decompressing huge         │ TSDB or query layer should route    │
│                                  │ amounts of data                    │ this automatically, or app/dashboard│
│                                  │                                  │ explicitly queries the rollup)      │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Retention policy deletes data      │ Misconfigured retention removes    │ Carefully align retention policies  │
│ needed for compliance/audit         │ raw or aggregated data that's       │ with regulatory requirements         │
│                                  │ legally required to be retained    │ (relevant for BFSI); consider        │
│                                  │                                  │ exporting to Object Storage          │
│                                  │                                  │ (Topic 9) Archive tier for long-term│
│                                  │                                  │ compliance retention beyond the     │
│                                  │                                  │ TSDB's practical limits              │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 6. Interview Quick-Fire Answers

**Q: Why are general-purpose relational databases poorly suited for time-series workloads at scale?**
A: Time-series workloads have extremely high, continuous write volume (every index update costs on every write, per the Indexing topic), generate unbounded storage growth with no built-in concept of downsampling/retention, and the "hot" recent data combined with rarely-needed historical data doesn't map well to a single table's storage layout. Time-series databases solve this with column-oriented storage and specialized compression (10-20x smaller than raw row storage), time-based chunking/partitioning (so retention = deleting whole chunk files, and recent-data queries only touch recent chunks), and automatic downsampling (keeping full resolution only for recent data, aggregates for older data).

**Q: What is "cardinality" in the context of time-series databases, and why does high cardinality cause problems?**
A: Cardinality refers to the number of UNIQUE time series, where each unique combination of metric name + label/tag values constitutes one series. If a label has high cardinality (e.g., a unique request ID or user ID used as a label), every distinct value creates a brand new time series — potentially millions of them — overwhelming the TSDB's indexing of "which series exist" and degrading performance dramatically. High-cardinality data (per-request IDs, etc.) should go into logs or traces, not time-series metric labels.

**Q: How does time-based chunking make retention policies efficient?**
A: Data is partitioned into immutable chunks based on time ranges (e.g., one chunk per day). Deleting old data is then just deleting entire chunk files — an O(1) filesystem operation — rather than running row-by-row DELETE statements on a massive table, which would be slow and (in MVCC-based databases, per the ACID/Indexing topics) leave significant cleanup overhead. This also means queries for recent time ranges only need to access the most recent (small number of) chunks, ignoring the bulk of historical data entirely.

**Q: How would you design a metrics pipeline for a fleet of 10,000 servers reporting 50 metrics every 10 seconds?**
A: That's 50,000 writes/second sustained — a clear case for a purpose-built TSDB (Prometheus, InfluxDB, or TimescaleDB) rather than a general-purpose database. Keep label cardinality low (server_id, region, metric_name as labels — avoid high-cardinality labels like request IDs). Configure tiered retention: raw 10-second data for a short window (e.g., 7 days) for detailed debugging/alerting, with automatic downsampling to 1-minute and 1-hour aggregates retained for longer (30 days, 1 year) for trend analysis and dashboards — matching storage cost and query performance to actual access patterns over time.


---
---

# TOPIC 11: Data Warehousing

---

## 1. What Problem Does Data Warehousing Solve?

Every topic so far has focused on **OLTP** (Online Transaction Processing) — databases optimized for the operational, day-to-day workload of an application: "create this order," "look up this user," "update this balance." These queries touch FEW ROWS, run FREQUENTLY, and need to be FAST and CONSISTENT (recall ACID).

But businesses ALSO need a completely different kind of query — **OLAP** (Online Analytical Processing): "What was total revenue per region per month for the last 2 years?" "How does customer retention vary by acquisition channel?" "Which product categories have the highest return rate?"

```
THE FUNDAMENTAL CONFLICT:

OLTP QUERY:
SELECT * FROM orders WHERE id = 12345;
→ Touches 1 ROW. Uses an INDEX (Topic 5). Returns instantly.
→ Runs THOUSANDS of times per second.

OLAP QUERY:
SELECT region, DATE_TRUNC('month', created_at) AS month,
       SUM(total) AS revenue
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY region, month
ORDER BY region, month;
→ Touches MILLIONS/BILLIONS of ROWS (potentially the ENTIRE
  orders table, going back YEARS).
→ Runs occasionally (a dashboard refresh, a scheduled report).
→ Even WITH indexes, "scan and aggregate millions of rows" is
  fundamentally a DIFFERENT KIND of operation than "find this
  one row by ID."

RUNNING THIS OLAP QUERY DIRECTLY ON YOUR PRODUCTION OLTP DATABASE:
- Consumes massive CPU/IO resources for MINUTES
- COMPETES for resources with the live application's OLTP
  queries — could cause your production app to SLOW DOWN or
  even become UNAVAILABLE while a report runs!
- Even with READ REPLICAS (Topic 4), a single heavy analytical
  query can still degrade that replica's performance for OTHER
  reads happening on it at the same time
```

**A Data Warehouse** is a SEPARATE database system, specifically architected for OLAP-style queries — large scans, aggregations, and joins across massive historical datasets — completely ISOLATED from the OLTP systems that run the live application.

---

## 2. Core Intuition — Row-Oriented vs Column-Oriented Storage

This is THE fundamental architectural difference between OLTP databases and Data Warehouses, and the single most important concept in this topic.

### Row-Oriented Storage (OLTP databases — PostgreSQL, MySQL)

```
TABLE: orders (id, user_id, total, status, created_at, region)

PHYSICAL STORAGE ON DISK (row-oriented):
┌─────────────────────────────────────────────────────────────┐
│ Row 1: [id=1, user_id=10, total=999, status='shipped',         │
│         created_at='2024-01-01', region='IN']                  │
├─────────────────────────────────────────────────────────────┤
│ Row 2: [id=2, user_id=22, total=4999, status='pending',         │
│         created_at='2024-01-01', region='US']                  │
├─────────────────────────────────────────────────────────────┤
│ Row 3: [id=3, user_id=15, total=1299, status='delivered',       │
│         created_at='2024-01-02', region='IN']                  │
└─────────────────────────────────────────────────────────────┘
ALL columns of ONE row are stored TOGETHER, contiguously.

WHY THIS IS GREAT FOR OLTP:
"Get order #2, ALL its details" → ONE disk read gets the ENTIRE
row (id, user_id, total, status, created_at, region) — exactly
the OLTP pattern (recall: "find by primary key, return the full
record" from the Indexing topic).

WHY THIS IS TERRIBLE FOR OLAP:
SUM(total) across 10 million rows → the database must READ
EVERY ROW (each containing id, user_id, total, status,
created_at, region — 6 columns), even though the query ONLY
NEEDS the `total` column! You're reading 6x more data than
necessary from disk, just to throw away 5/6 of it.
```

### Column-Oriented Storage (Data Warehouses — Snowflake, BigQuery, Redshift, ClickHouse)

```
PHYSICAL STORAGE ON DISK (column-oriented):

id column:          [1, 2, 3, 4, 5, ...]
user_id column:     [10, 22, 15, 8, 30, ...]
total column:       [999, 4999, 1299, 599, 2499, ...]
status column:      ['shipped', 'pending', 'delivered', ...]
created_at column:  ['2024-01-01', '2024-01-01', '2024-01-02', ...]
region column:      ['IN', 'US', 'IN', 'IN', 'EU', ...]

EACH COLUMN is stored SEPARATELY, contiguously, on disk.

WHY THIS IS GREAT FOR OLAP:
SUM(total) across 10 million rows → the database reads ONLY the
`total` column's data — a SINGLE, SEQUENTIAL, contiguous block
of disk, containing JUST the numbers needed. It NEVER touches
id, user_id, status, created_at, region at all. 

If `total` is 8 bytes and there are 10 million rows: reading
just this column = 80 MB. Reading the WHOLE ROW-ORIENTED table
(6 columns, similar sizes) = potentially 480 MB+ — a 6x
reduction just from NOT reading unnecessary columns!

ADDITIONALLY: storing each column SEPARATELY enables MUCH BETTER
COMPRESSION — a `region` column with only 3-4 distinct values
("IN", "US", "EU"...) repeated millions of times compresses
EXTREMELY well (similar principle to the Gorilla compression
discussed in the Time-series DBs topic — VALUES with low
variability/repetition compress dramatically). Real-world
column stores often achieve 10x or greater compression on
typical analytical data.

WHY THIS IS TERRIBLE FOR OLTP:
"Get order #2, ALL its details" now requires reading from
SIX DIFFERENT PLACES ON DISK (one seek per column) and
RECONSTRUCTING the row — much slower than row-oriented storage
for this "fetch one full record" pattern. This is EXACTLY why
column stores are NOT used for OLTP — the access pattern is
INVERTED.
```

---

## 3. ETL / ELT — Getting Data INTO the Warehouse

```
Since the warehouse is a SEPARATE system from your OLTP databases
(and possibly NoSQL stores, object storage, third-party APIs,
etc. — recall the "polyglot persistence" discussion from SQL vs
NoSQL), data must be MOVED from these SOURCE SYSTEMS into the
warehouse — this process is called ETL or ELT.

ETL (Extract, Transform, Load) — the traditional approach:

┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│  EXTRACT      │ ──▶│  TRANSFORM         │──▶│  LOAD          │
│  Pull data    │    │  Clean, reshape,    │    │  Write the     │
│  from OLTP    │    │  join, aggregate    │    │  transformed   │
│  databases,    │    │  data in a          │    │  data into the │
│  APIs, files   │    │  SEPARATE PROCESSING│    │  warehouse      │
│               │    │  step (before it     │    │  (already in   │
│               │    │  reaches the         │    │  final shape)  │
│               │    │  warehouse)          │    │                │
└──────────────┘    └──────────────────┘    └──────────────┘

ELT (Extract, Load, Transform) — the modern approach:

┌──────────────┐    ┌──────────────┐    ┌──────────────────┐
│  EXTRACT      │ ──▶│  LOAD          │──▶│  TRANSFORM         │
│  Pull RAW     │    │  Write RAW DATA │    │  Use the          │
│  data from     │    │  directly into  │    │  warehouse's OWN  │
│  sources       │    │  the warehouse  │    │  (massive) compute│
│               │    │  AS-IS           │    │  power to         │
│               │    │                │    │  transform/reshape │
│               │    │                │    │  data WITHIN it    │
│               │    │                │    │  (e.g., using SQL  │
│               │    │                │    │  via tools like     │
│               │    │                │    │  dbt)              │
└──────────────┘    └──────────────┘    └──────────────────┘

WHY ELT BECAME DOMINANT: Modern cloud data warehouses (Snowflake,
BigQuery) have SEPARATED compute and storage (you pay for storage
cheaply, and SCALE COMPUTE ON DEMAND for transformations) —
making it cost-effective to load raw data FIRST and use the
warehouse's own elastic compute (recall Auto-scaling concepts) for
transformations, rather than maintaining a separate transformation
infrastructure. Tools like dbt ("data build tool") have become
the standard for managing these in-warehouse SQL transformations
as version-controlled, testable code.
```

---

## 4. CDC (Change Data Capture) — Keeping the Warehouse Fresh

```
PROBLEM: Running a nightly "dump the entire orders table" ETL job
worked when data was smaller, but:
- Re-extracting the ENTIRE table every night is wasteful
  (most rows haven't changed)
- Data in the warehouse is up to 24 HOURS STALE — unacceptable
  for many modern "real-time analytics" use cases

SOLUTION: Change Data Capture — instead of periodically QUERYING
the source database for "what's changed," TAP DIRECTLY into the
database's REPLICATION LOG (the WAL/binlog from the Replication
topic!) — the SAME mechanism used for primary-replica replication
can ALSO stream changes to a data warehouse pipeline.

┌──────────────┐  WAL/binlog   ┌──────────────────┐  stream  ┌──────────────┐
│  OLTP Primary │ ─────────────▶│  CDC Tool          │─────────▶│  Data          │
│  Database     │  (same         │  (Debezium, etc.)  │  events  │  Warehouse     │
│              │  mechanism as  │  reads the WAL,     │          │  (near-real-   │
│              │  Replication!) │  emits change events│          │  time updates) │
└──────────────┘                └──────────────────┘          └──────────────┘

BENEFIT: Near-real-time data warehouse updates (seconds/minutes
of lag instead of hours), with MINIMAL load on the source
database (reading a replication log is far cheaper than running
large extraction queries against it).

THIS DIRECTLY REUSES the Write-Ahead Log concept from BOTH the
Replication topic AND the ACID/Durability topic — the SAME
underlying mechanism (a sequential log of all changes) serves
THREE different purposes: crash recovery (ACID durability),
replica synchronization (Replication), and analytical data
pipelines (CDC) — a great example of one well-designed primitive
enabling multiple capabilities.
```

---

## 5. The Star Schema — Organizing Data for Analytics

```
Data warehouses commonly organize data using a STAR SCHEMA — a
DELIBERATE, ANALYTICS-OPTIMIZED structure, distinct from the
normalized (3NF) OLTP schemas discussed in the Normalization
topic.

                    ┌──────────────────┐
                    │  dim_date          │
                    │  date_id (PK)      │
                    │  date, month, year, │
                    │  quarter, weekday   │
                    └─────────┬─────────┘
                              │
┌──────────────────┐  ┌──────▼─────────────┐  ┌──────────────────┐
│  dim_customer      │  │   FACT TABLE:        │  │  dim_product       │
│  customer_id (PK)  │◀─┤   fact_orders        │─▶│  product_id (PK)   │
│  name, region,      │  │   order_id           │  │  name, category,    │
│  segment, ...       │  │   customer_id (FK)   │  │  brand, price, ...  │
└──────────────────┘  │   product_id (FK)    │  └──────────────────┘
                       │   date_id (FK)       │
                       │   quantity            │
                       │   revenue              │
                       │   discount             │
                       └─────────────────────┘

FACT TABLE: The "events" — one row per order/transaction.
Contains MEASURES (numeric values to aggregate: revenue,
quantity) and FOREIGN KEYS to DIMENSION TABLES.

DIMENSION TABLES: The "context" — descriptive attributes about
customers, products, time, etc. RELATIVELY SMALL compared to
the fact table (e.g., thousands of products vs billions of order
line items).

WHY THIS WORKS WELL WITH COLUMN-ORIENTED STORAGE:

"Total revenue by product category, by quarter":
SELECT dim_product.category, dim_date.quarter, SUM(fact_orders.revenue)
FROM fact_orders
JOIN dim_product ON fact_orders.product_id = dim_product.product_id
JOIN dim_date ON fact_orders.date_id = dim_date.date_id
GROUP BY dim_product.category, dim_date.quarter;

The column store reads ONLY: fact_orders.revenue,
fact_orders.product_id, fact_orders.date_id (from the huge fact
table), plus the SMALL dimension tables (category, quarter) —
extremely efficient, even though it LOOKS like a normalized
schema (recall Normalization topic) with JOINs.

NOTE: This is DIFFERENT from OLTP normalization — DIMENSION
tables are often DELIBERATELY DENORMALIZED themselves (e.g.,
dim_customer might include flattened address fields rather than
a separate normalized addresses table) because the GOAL is
QUERY SIMPLICITY/SPEED for analysts, not write-consistency
(dimension tables are updated relatively infrequently via ETL,
not by live application writes).
```

---

## 6. Real-World Usage

**Snowflake/BigQuery/Redshift (cloud data warehouses):** All implement column-oriented storage with MASSIVE PARALLEL PROCESSING (MPP) — a single query is automatically split across MANY compute nodes, each processing a portion of the data in parallel (recall horizontal scaling concepts from Scalability notes, applied to QUERY EXECUTION itself, not just request handling). BigQuery in particular separates storage and compute so completely that you're billed per-query based on bytes scanned — directly incentivizing column-pruning (only select the columns you need) and partition-pruning (filter on date columns to avoid scanning irrelevant time ranges — recall time-based chunking from the Time-series DBs topic).

**Modern "Data Lakehouse" architecture (Databricks, etc.):** Combines OBJECT STORAGE (Topic 9 — cheap, durable, for raw data in formats like Parquet — itself a COLUMN-ORIENTED file format) with a QUERY ENGINE layered on top, blurring the line between "data lake" (raw files in object storage) and "data warehouse" (structured, queryable) — getting object storage's cost benefits with warehouse-like query performance via column-oriented file formats and metadata layers (Delta Lake, Iceberg).

**E-commerce analytics (typical end-to-end):** OLTP PostgreSQL (orders, users — Topic 1/6) → CDC via Debezium reads the WAL (Topic 4 mechanism) → streams to Kafka → loaded into Snowflake's fact/dimension tables (star schema) → BI tools (Looker, Tableau) query the warehouse for dashboards — completely isolated from the production OLTP system's load.

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Production database slows down   │ Analysts/BI tools running heavy  │ NEVER run OLAP queries directly    │
│ during business hours             │ analytical queries DIRECTLY      │ against OLTP primaries; use read   │
│                                  │ against the OLTP primary/replicas│ replicas ONLY for light reporting,│
│                                  │                                  │ heavy analytics ALWAYS go to a      │
│                                  │                                  │ dedicated data warehouse           │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Data warehouse query scans far     │ Query doesn't filter/select       │ Always filter on partition columns │
│ more data (and costs much more)    │ specific columns/partitions —     │ (e.g., date ranges) and SELECT     │
│ than expected                     │ a column store's efficiency        │ only needed columns — "SELECT *"   │
│                                  │ depends on PRUNING what's read    │ is an anti-pattern in column stores│
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ ETL/CDC pipeline failure causes    │ Source schema change (column      │ Schema validation/contracts in the │
│ silent data corruption/loss in     │ added/renamed) breaks downstream  │ pipeline; monitoring for row-count │
│ warehouse                          │ transformation logic without an   │ anomalies and pipeline failures     │
│                                  │ explicit error                     │                                     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Dashboards show stale data          │ Batch ETL runs nightly; business  │ Move to CDC-based streaming         │
│                                  │ needs near-real-time numbers       │ pipelines for freshness-critical    │
│                                  │                                  │ dashboards; clearly communicate     │
│                                  │                                  │ data freshness SLAs to stakeholders│
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Star schema dimension table         │ A dimension attribute (e.g.,       │ Use "Slowly Changing Dimension"     │
│ doesn't reflect HISTORICAL          │ customer's region) CHANGES over    │ (SCD) techniques — e.g., keep        │
│ context (e.g., a customer's         │ time, but the fact table's         │ historical versions of dimension    │
│ region at the time of an OLD        │ foreign key just points to the      │ rows so historical facts can be     │
│ order)                              │ CURRENT dimension row              │ correctly attributed to the         │
│                                  │                                  │ dimension values AS THEY WERE THEN  │
│                                  │                                  │ (connects to the "point-in-time"     │
│                                  │                                  │ denormalization discussion!)        │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: Why can't you just run analytical queries on your production database?**
A: Analytical (OLAP) queries scan and aggregate across millions/billions of rows — fundamentally different from operational (OLTP) queries that touch a few rows via an index. Running heavy analytical queries against the production database (even a read replica) consumes resources and can degrade performance for the live application's normal operations. Data warehouses are separate systems specifically architected (column-oriented storage, massive parallel processing) for these large-scan workloads, completely isolated from production OLTP load.

**Q: Explain row-oriented vs column-oriented storage and why it matters for analytics.**
A: Row-oriented storage (used by OLTP databases) keeps all columns of a row together on disk — great for "fetch this one record with all its fields" (the OLTP pattern). Column-oriented storage (used by data warehouses) stores each column separately and contiguously — so a query like `SUM(revenue)` reads ONLY the revenue column, ignoring all other columns entirely, dramatically reducing I/O for analytical queries that aggregate over specific columns across huge numbers of rows. Column storage also compresses much better, since each column tends to have low value variability (e.g., a "region" column with only a few distinct values repeated millions of times).

**Q: What's the difference between ETL and ELT, and why has ELT become more common?**
A: ETL transforms data in a separate processing step BEFORE loading it into the warehouse. ELT loads raw data into the warehouse first, then uses the warehouse's own (elastic, on-demand) compute power to perform transformations via SQL — often managed with tools like dbt. ELT became dominant because modern cloud data warehouses separate storage and compute, making it cheap to store raw data and cost-effective to scale compute on-demand for transformations, rather than maintaining separate transformation infrastructure.

**Q: How does Change Data Capture (CDC) work, and what's its advantage over batch ETL?**
A: CDC taps directly into a database's replication log (the same Write-Ahead Log used for crash recovery and primary-replica replication) to stream every change (insert/update/delete) as an event, in near-real-time, to downstream systems like a data warehouse. This avoids the overhead of periodically re-querying the source database for "what changed" (batch extraction), reduces load on the production database, and dramatically reduces data staleness — from hours (nightly batch ETL) to seconds/minutes — enabling near-real-time analytics.

**Q: What is a star schema and why is it used in data warehouses instead of a normalized (3NF) schema?**
A: A star schema organizes data into a central FACT table (one row per event/transaction, containing numeric measures and foreign keys) surrounded by DIMENSION tables (descriptive context — customers, products, dates). Unlike OLTP's 3NF normalization (optimized to minimize redundancy for consistent writes), star schemas are optimized for ANALYTICAL QUERY SIMPLICITY AND SPEED — dimension tables are often deliberately denormalized themselves, and the simple fact-plus-dimensions structure maps naturally onto column-oriented storage's strengths (the fact table contributes only a few columns per query, dimension tables are small).

---
---

# APPENDIX: Cross-Topic Quick Reference

## Master Comparison — All Concepts at a Glance

```
┌───────────────────────────┬───────────────────────────────────────────────────────────┐
│ Topic                     │ Core Question It Answers                                    │
├───────────────────────────┼───────────────────────────────────────────────────────────┤
│ SQL vs NoSQL               │ "What SHAPE is my data, and how will I QUERY it?"           │
│ CAP Theorem                │ "When a network partition happens, do I prioritize          │
│                           │ correctness (C) or responsiveness (A)?"                      │
│ Database Sharding          │ "How do I split my data across multiple servers when one    │
│                           │ server can no longer hold/serve it all?"                     │
│ Replication                │ "How do I keep multiple copies of data in sync, and what    │
│                           │ tradeoffs come with HOW they're synced?"                     │
│ Indexing                   │ "How do I find specific rows quickly without scanning        │
│                           │ everything, and what does that cost on writes?"              │
│ ACID Properties            │ "How do I guarantee a group of operations behaves as ONE     │
│                           │ atomic, consistent, isolated, durable unit?"                 │
│ NoSQL Types                │ "Given that NoSQL fits, WHICH data model (key-value,         │
│                           │ document, column-family, graph) fits my access pattern?"     │
│ Database Normalization     │ "How do I structure tables to avoid redundancy and the       │
│                           │ anomalies that come with it — and when should I NOT?"        │
│ Object Storage              │ "How do I durably store large unstructured files at         │
│                           │ massive scale, separate from my queryable databases?"        │
│ Time-series DBs             │ "How do I handle extremely high-volume, time-ordered,       │
│                           │ append-only data with natural aging/downsampling needs?"     │
│ Data Warehousing            │ "How do I run heavy analytical queries across historical    │
│                           │ data WITHOUT impacting my live production systems?"          │
└───────────────────────────┴───────────────────────────────────────────────────────────┘
```

## A Complete System — All Topics in One Flow

```
DESIGNING A DATA LAYER FOR A PRODUCT — putting it all together:

1. CHOOSE THE RIGHT DATA MODEL PER COMPONENT (Topic 1 + Topic 7):
   - Orders/payments/inventory → SQL (need ACID — Topic 6)
   - Product catalog → Document store (varying schemas)
   - Sessions/cache/rate-limit counters → Key-value (Redis)
   - Activity feed/event log → Column-family (Cassandra)
   - "People you may know" / fraud detection → Graph database
   - User-uploaded files/images → Object storage (Topic 9)
   - Server/app metrics → Time-series DB (Topic 10)

2. APPLY NORMALIZATION PRINCIPLES (Topic 8) where consistency
   matters (SQL schemas); DENORMALIZE deliberately where read
   speed matters or for point-in-time records (orders/invoices),
   and naturally in NoSQL document/embedding designs.

3. INDEX THOUGHTFULLY (Topic 5) based on actual query patterns —
   composite indexes following the leftmost-prefix rule, covering
   indexes for hot read paths, being mindful of write-cost on
   high-volume tables.

4. SCALE READS WITH REPLICATION (Topic 4):
   - Async replicas for general read scaling
   - Synchronous/semi-sync for data requiring durability guarantees
   - Multi-region replicas for geo-distribution (connects to
     Scalability notes' Geo-distribution topic)
   - Understand the CAP/PACELC (Topic 2) tradeoff each choice makes

5. SCALE WRITES WITH SHARDING (Topic 3) when a single primary
   can no longer handle write volume — choose shard keys based
   on dominant access patterns, understanding the cross-shard
   JOIN/transaction costs this introduces.

6. SEPARATE OPERATIONAL FROM ANALYTICAL WORKLOADS (Topic 11):
   - CDC pipelines (reusing the WAL mechanism from Topic 4/6)
     stream changes from OLTP systems into a data warehouse
   - Star schema design for analytical query simplicity
   - Column-oriented storage for efficient large-scan aggregations
   - Dashboards/reporting NEVER touch production OLTP systems

7. EVERYTHING CONNECTS BACK to the Scalability & Load Balancing
   notes: consistent hashing underlies sharding AND NoSQL
   distribution; back-of-envelope estimation determines WHICH
   of these patterns you actually need at your scale; rate
   limiting and caching (key-value stores) protect the data
   layer from overload.
```

---

## Final Study Tips

```
1. DRAW the B+ Tree structure (Indexing), the CAP theorem
   partition scenario (CAP Theorem), and the star schema
   (Data Warehousing) from memory. These three diagrams anchor
   a huge fraction of "databases" interview questions.

2. ALWAYS connect a database choice back to ACCESS PATTERNS and
   SCALE (Back-of-Envelope Estimation from Scalability notes).
   "We'll use Cassandra" is incomplete. "We need ~50,000
   writes/second with a 'get everything for this user' access
   pattern, so Cassandra's column-family model with user_id as
   the partition/row key fits — but we'd need a separate table
   if we also need to query by a different attribute" shows
   depth.

3. PRACTICE explaining the SAME underlying mechanism (the
   Write-Ahead Log) showing up in THREE different topics —
   Durability (ACID), Replication, and CDC (Data Warehousing).
   Recognizing these connections is exactly the "systems
   thinking" interviewers are probing for.

4. FOR EVERY DESIGN DECISION, STATE THE TRADEOFF EXPLICITLY.
   "We chose asynchronous replication" is incomplete.
   "We chose asynchronous replication for read-scaling, accepting
   eventual consistency and a small data-loss risk on primary
   failure — for THIS use case (product catalog reads), that's
   an acceptable tradeoff; for the payments table, we'd use
   semi-synchronous replication instead" demonstrates the kind
   of judgment that separates strong candidates.

5. For BFSI/fintech interviews (relevant to your prep): expect
   deep questions on ACID guarantees for transactions, CP-leaning
   choices (CAP theorem) for balance/ledger data, normalized
   schemas with strict foreign-key integrity for core banking
   data, time-series databases for market/tick data, and data
   warehousing for regulatory reporting — with CDC pipelines
   feeding from core banking OLTP systems into warehouses for
   compliance/audit analytics without touching live transaction
   processing.
```
