# System Design Trade-Off Handbook

## All 18 Trade-Offs

1. **SQL vs NoSQL** — When ACID matters vs when scale wins
2. **Consistency vs Availability (CAP)** — CP vs AP, partition behavior, PACELC
3. **Strong vs Eventual Consistency** — Synchronous vs async replication, read-your-writes
4. **Push vs Pull Architecture** — Fan-out on write vs read, celebrity problem, hybrid
5. **Synchronous vs Asynchronous Communication** — Coupling, latency, failure isolation
6. **Cache vs Database** — Redis strategies, invalidation, stampede prevention
7. **Monolith vs Microservices** — Team size, complexity, Strangler Fig, distributed monolith
8. **Replication vs Sharding** — Read vs write scaling, shard key selection, hotspots
9. **WebSocket vs Polling** — Real-time latency, connection scaling, SSE
10. **REST vs GraphQL** — Over/under-fetching, N+1, caching, DataLoader
11. **Batch vs Stream Processing** — Lambda/Kappa architecture, late data, watermarks
12. **Horizontal vs Vertical Scaling** — Stateless requirements, auto-scaling, cost curves
13. **Memory vs Latency** — Cache sizing, working set, memory hierarchy
14. **Throughput vs Latency** — Little's Law, batching, pipeline design
15. **Latency vs Accuracy** — HyperLogLog, Count-Min Sketch, Bloom Filters
16. **CDN vs No CDN** — Content-hash filenames, cache invalidation, DDoS protection
17. **Kafka vs RabbitMQ** — Event streams vs task queues, consumer groups, replay
18. **Normalization vs Denormalization** — JOIN performance, update anomalies, CQRS

---

## The Golden Interview Rule

> Never say "It depends" without immediately stating WHAT it depends on and giving a recommendation.
> Always say: "Given [specific constraints], I'd choose [option] because [specific reason]."

---

# System Design Trade-Off Handbook
## For Product-Based Company Interviews | Senior Engineer Level

---

> **Purpose:** Master the WHY behind every architectural decision so you can defend your choices, handle follow-ups, and think like a Senior Engineer under interview pressure.

---

# TRADE-OFF 1: SQL vs NoSQL

---

## 1. Concept Overview

**What it is:** The choice between relational databases (SQL — Postgres, MySQL, SQL Server) and non-relational databases (NoSQL — MongoDB, Cassandra, DynamoDB, Redis).

**Why it exists:** SQL was built for structured, transactional data with complex relationships. NoSQL emerged when internet-scale companies (Google, Amazon, Facebook) hit the limits of SQL's horizontal scalability and rigid schema requirements.

**Why interviewers ask:** It's the first architectural decision in almost every system design. Getting it wrong signals poor fundamentals. Interviewers want to see that you understand the implications, not just memorize "use NoSQL for scale."

---

## 2. Visual Diagram

```
SQL (Relational Model):
┌──────────┐    FK    ┌──────────┐    FK    ┌──────────────┐
│  Users    │────────▶│  Orders   │────────▶│  Order_Items  │
│  id, name │         │  id, user │         │  id, product  │
└──────────┘         └──────────┘         └──────────────┘
          JOIN at query time — normalized, consistent

NoSQL (Document Model):
┌──────────────────────────────────────────────┐
│  User Document                                 │
│  { id, name, orders: [{id, items:[...]}] }    │
│  Everything embedded — denormalized, fast read │
└──────────────────────────────────────────────┘
```

---

## 3. Option A — SQL (Relational Database)

**Definition:** Stores data in tables with predefined schemas, rows, and columns. Relationships enforced via foreign keys. Queried via SQL.

**Advantages:**
- ACID transactions — atomicity, consistency, isolation, durability guaranteed
- Rich query language — JOINs, aggregations, complex filtering
- Schema enforcement — data integrity guaranteed at the DB level
- Mature tooling — decades of optimization, monitoring, backup tooling
- Strong consistency — single primary always has the latest data

**Disadvantages:**
- Horizontal scaling is hard — sharding SQL breaks JOINs
- Schema migrations are painful at scale (ALTER TABLE on 500M rows)
- Fixed schema — can't store varying attributes per record
- Object-relational mismatch — app objects don't naturally map to tables

**Complexity:** Low initially (familiar SQL), high at scale (sharding, read replicas)

**Cost:** Vertical scaling is expensive. Large RDS instances cost significantly more than commodity horizontally-scaled NoSQL clusters.

**Scalability:** Read scaling via replicas is straightforward. Write scaling requires sharding — complex, and breaks JOIN and transaction semantics.

---

## 4. Option B — NoSQL

**Definition:** Non-relational databases optimized for specific access patterns. Key-value (Redis, DynamoDB), Document (MongoDB), Column-family (Cassandra), Graph (Neo4j).

**Advantages:**
- Horizontal scaling is native — designed for distributed deployment
- Flexible schema — different documents can have different fields
- High write throughput — Cassandra handles millions of writes/sec
- Access-pattern-optimized — model data for how it's queried, not normalized
- Eventual consistency by default — enables AP-leaning high-availability

**Disadvantages:**
- Weak or no multi-entity transactions (improving, but complex)
- No JOINs — must denormalize or handle in application code
- Eventual consistency — reads may return stale data
- Less mature — schema evolution, reporting, and tooling not as rich
- Developer familiarity — team must learn new query APIs and patterns

**Complexity:** Simple per-operation, complex overall data modelling (must design for access patterns upfront)

**Cost:** Commodity hardware at scale is cheaper. But over-provisioning for peak and complex operational management adds cost.

**Scalability:** Native horizontal scaling. Each NoSQL type scales differently — Cassandra for writes, Redis for reads from memory, DynamoDB for managed auto-scaling.

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬─────────────────────────────┬────────────────────────────────┐
│ Factor              │ SQL                          │ NoSQL                           │
├────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Scalability         │ Vertical first; sharding hard│ Horizontal native; built for it │
│ Availability        │ Primary-replica; failover OK │ Multi-master or AP-by-default   │
│ Consistency         │ Strong (ACID)                │ Eventual (tunable in Cassandra) │
│ Latency             │ Higher on complex queries    │ Lower for key-based lookups     │
│ Throughput          │ Limited by primary writes    │ Very high (Cassandra, DynamoDB) │
│ Complexity          │ Low schema, high at scale    │ High data modelling upfront      │
│ Cost                │ Expensive at scale           │ Cheaper commodity scaling        │
│ Operational Effort  │ Well-understood ops          │ Newer; operational complexity    │
│ Reliability         │ Very high; proven            │ High; depends on system choice   │
│ Transactions        │ Full ACID, multi-table       │ Limited, single-row in most      │
│ Query Flexibility   │ Very high (ad-hoc SQL)       │ Low (must design access patterns)│
└────────────────────┴─────────────────────────────┴────────────────────────────────┘
```

---

## 6. When To Choose SQL

- **Financial systems:** Bank transfers, payments, ledger entries — ACID is mandatory
- **Inventory management:** "Last unit" oversell prevention requires transactions
- **User account data:** Structured, relational data with complex queries
- **Order management systems:** Orders → line items → products → inventory
- **Companies:** Stripe (payments), GitHub (repository metadata), Shopify (orders)
- **Early-stage startups:** Ship fast, schema changes are manageable, team knows SQL

---

## 7. When To Choose NoSQL

- **Massive write throughput:** Uber location updates, IoT sensor data, Cassandra
- **Varying schemas:** Product catalogs (electronics vs clothing have different attributes)
- **Session/cache data:** Short-lived, key-lookup only — Redis
- **Social feeds/activity:** Write-heavy, eventual consistency fine — Cassandra/DynamoDB
- **Content management:** Articles with different structures — MongoDB
- **Companies:** Netflix (viewing history), Discord (messages), Instagram (activity feeds)

---

## 9. Strong Interview Answers

**Q: "Your SQL database is becoming a bottleneck. Walk me through what you'd do."**

> "First I'd identify WHERE the bottleneck is — reads or writes. For read bottlenecks, I'd add read replicas and direct 80% of traffic there, potentially with a caching layer (Redis) in front. For write bottlenecks, I'd first look at query optimization — missing indexes, N+1 problems, slow queries. If I genuinely exhaust vertical scaling options, I'd look at functional decomposition first: can I extract a less-critical domain to a separate database? As a last resort, sharding by user_id or tenant_id — but I'd accept losing cross-shard JOINs and I'd isolate my transactions to stay within one shard."

**Q: "Why not just use NoSQL for everything?"**

> "Because the problem NoSQL solves — horizontal write scaling and flexible schemas — is not the problem most applications have, especially early on. The problems NoSQL introduces — no transactions, no JOINs, eventual consistency, complex data modelling — are REAL problems that cause bugs. A $5M SaaS company with 50,000 users doesn't need Cassandra. They need reliable transactions for their billing system. Use the simplest tool that meets the requirement. Complexity has a real cost."

---

## 10. Common Mistakes

- **Choosing NoSQL because it "scales better"** without identifying the actual bottleneck
- **Choosing SQL without considering write scale** when the system is event-heavy
- **Treating NoSQL as a drop-in replacement** for SQL — schema design is completely different
- **Not considering the transaction boundary** — moving to NoSQL breaks transactions you were implicitly relying on
- **Over-normalizing in SQL** when denormalization for read performance is appropriate
- **Under-estimating operational cost** of self-managed NoSQL clusters

---

## 12. Real System Design Examples

| System | SQL/NoSQL | Why |
|--------|-----------|-----|
| WhatsApp | Cassandra for messages | Write-heavy, AP, billions of messages |
| Instagram | PostgreSQL (sharded) for core + Cassandra for activity | Transactional core, high-write feeds |
| Uber | MySQL/PostgreSQL for trips + Cassandra for location history | Trips need ACID, locations don't |
| YouTube | MySQL for video metadata + Bigtable for views | Metadata is relational; view counts are high-write |
| Dropbox | MySQL for file metadata | Highly relational, transactional |
| Netflix | Cassandra for viewing history | Write-heavy, globally distributed, eventual fine |
| Ticketmaster | SQL (Oracle/PostgreSQL) | Ticket inventory MUST be transactional |
| Twitter/X | Manhattan (custom) + MySQL | Scale forced custom solution; Tweets are eventually consistent |
| URL Shortener | Redis/DynamoDB | Simple key-value, extremely high reads |

---

## 13. Scaling Journey

```
Small Scale (< 100K users):
  → PostgreSQL single instance. Simple, reliable, ACID.
  → Add a read replica when reads slow down.

Medium Scale (100K - 10M users):
  → Add Redis caching layer for hot reads.
  → Add more read replicas, load balance reads.
  → Separate write-heavy features (events, logs) to DynamoDB/Cassandra.

Large Scale (10M - 500M users):
  → SQL sharded by user_id or tenant_id. Accept no cross-shard JOINs.
  → Introduce polyglot persistence: SQL for transactional, NoSQL for high-write.
  → Consider read-optimized stores (Elasticsearch) for search.

Hyperscale (500M+ users):
  → Custom distributed SQL (Google Spanner, CockroachDB) OR fully committed NoSQL.
  → Multi-region replication becomes a requirement, not an optimization.
  → Every data store optimized for its specific access pattern.
```

---

## 14. Decision Framework

```
Requirements
  → "Do I need multi-entity transactions?" → YES → SQL
  → "Do I need complex ad-hoc queries?"    → YES → SQL
  → "Do I need > 100K writes/sec?"         → YES → NoSQL
  → "Do I need flexible schema?"            → YES → NoSQL
  → "Is eventual consistency acceptable?"   → NO  → SQL

Constraints
  → Team SQL expertise? → Favor SQL
  → Budget for managed NoSQL (DynamoDB)? → Consider NoSQL
  → Is this a startup MVP? → Default to SQL (simplicity)

Trade-Off Analysis
  → SQL: correct by default, slower at write scale
  → NoSQL: fast by default, consistency burden on app

Decision
  → Default to SQL unless you have a SPECIFIC reason for NoSQL.
  → Never mix up "I might need NoSQL scale someday" with "I need NoSQL now."
```

---
---

# TRADE-OFF 2: Consistency vs Availability (CAP Theorem)

---

## 1. Concept Overview

**What it is:** Eric Brewer's CAP theorem states that in a distributed system, during a network partition, you can guarantee EITHER Consistency (all nodes see the same data at the same time) OR Availability (every request gets a response), but NOT both.

**Why it exists:** Networks fail. When two nodes can't communicate, one must choose: refuse requests (preserve consistency) or answer with potentially stale data (preserve availability). There is no third option.

**Why interviewers ask:** It reveals whether you understand the fundamental constraints of distributed systems. Any database that replicates data across nodes lives under CAP. Your choice of database IS a statement about which side you're on.

---

## 2. Visual Diagram

```
Network Partition:

Node A ─── X ─── Node B   (network link broken)
(has x=10)       (has x=5, stale)

CP System (choose Consistency):
Client → Node B → "I can't guarantee I'm up to date. ERROR 503."
✅ Never returns wrong data   ❌ Temporarily unavailable

AP System (choose Availability):
Client → Node B → returns x=5 (stale, but an answer)
✅ Always responds            ❌ Might return wrong data

No partition (normal operation — PACELC extension):
Every system can be both consistent AND available.
The choice only matters WHEN something breaks.
```

---

## 3. Option A — CP (Consistency + Partition Tolerance)

**Definition:** When a partition occurs, the system refuses requests it can't serve with guaranteed consistency. Prefers returning an error over returning stale data.

**Examples:** etcd, ZooKeeper, MongoDB (with default settings), HBase, traditional RDBMS with synchronous replication

**Advantages:**
- Data is always correct — no stale reads ever
- Safe for financial transactions, inventory counts
- Simplifies application logic (no "what if data is stale?" handling)

**Disadvantages:**
- Unavailable during partitions — causes errors to users
- Latency increases — must wait for quorum acknowledgment
- Write availability limited — if primary is unreachable, writes fail

**When partitions occur in practice:** AWS AZ failures, network switches failing, cloud provider networking issues — these happen multiple times per year in large systems.

---

## 4. Option B — AP (Availability + Partition Tolerance)

**Definition:** During a partition, the system continues serving requests even if the data returned might be stale. All nodes respond. Eventual consistency — data will agree "eventually."

**Examples:** Cassandra, DynamoDB (default), CouchDB, DNS, most CDNs

**Advantages:**
- Always responds — users always get an answer
- Lower latency — no need to wait for cross-node confirmation
- Higher write throughput — any node can accept writes
- Resilient — partial failures don't cascade

**Disadvantages:**
- Stale data possible — reads may not reflect the latest write
- Conflict resolution needed — two nodes may accept conflicting writes
- Application must handle "what if I read old data?" — more complex app logic

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬─────────────────────────────┬────────────────────────────────┐
│ Factor              │ CP (Consistency)              │ AP (Availability)               │
├────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Data Correctness    │ Always correct               │ Possibly stale during partition  │
│ Availability        │ Unavailable during partition │ Always available                 │
│ Latency             │ Higher (waits for quorum)    │ Lower (any node responds)        │
│ Throughput          │ Lower (coordination overhead)│ Higher (no coordination needed)  │
│ Complexity          │ Simpler app logic            │ App must handle stale data       │
│ Cost                │ Higher (strong consistency)  │ Lower (eventual cheaper)         │
│ Failure Behavior    │ Returns error on partition   │ Returns stale data on partition  │
│ Best For            │ Finance, inventory, auth     │ Social feeds, analytics, caches  │
└────────────────────┴─────────────────────────────┴────────────────────────────────┘
```

---

## 6. When To Choose CP

- **Financial transactions:** Bank balances MUST be correct. Wrong balance = overdrafts, regulatory violations
- **Inventory systems:** Selling the "last item" to 10 customers simultaneously destroys trust
- **Authentication systems:** Who is logged in MUST be consistent (revoked tokens must propagate)
- **Distributed locks:** Leader election, mutex locks — exactly one holder at any time
- **Configuration systems:** Feature flags, service discovery — all nodes must see same config (etcd, ZooKeeper)
- **Companies:** Stripe, traditional banks, Ticketmaster (ticket availability)

---

## 7. When To Choose AP

- **Social media feeds:** Seeing a "like count" that's 2 seconds behind is fine
- **Shopping carts:** Amazon's famous Dynamo paper: "always let customers add to cart, resolve conflicts at checkout"
- **DNS:** Slight propagation delay (eventual) is fine; DNS being down would be catastrophic
- **Analytics/metrics:** Dashboard showing metrics 5 seconds old is acceptable
- **Content delivery (CDN):** Cached content being slightly stale is acceptable; CDN being down is not
- **User preferences/settings:** Theme color being stale for a few seconds is fine
- **Companies:** Netflix (viewing data), Twitter (tweet counts), Amazon (cart)

---

## 9. Strong Interview Answers

**Q: "Your payment service is AP. A customer charges twice because of a stale read. How do you prevent this?"**

> "The fix isn't to change from AP to CP at the storage layer — it's to add application-level idempotency. Every payment request gets a unique idempotency key generated client-side. Before processing, we check Redis or DynamoDB for that key. If it exists, we return the cached result — no double charge. The storage layer can be AP, but the application layer enforces exactly-once semantics. This is how Stripe handles it. The payment operation itself can be serialized via the idempotency key even when the underlying storage is eventually consistent."

**Q: "What is PACELC and why does it matter more than CAP for day-to-day engineering?"**

> "CAP only describes behavior during partitions, which are rare. PACELC says: even WITHOUT a partition, there's a tradeoff. If there's no Partition (P), you choose between Latency (L) and Consistency (C). Every database that replicates data faces this: do you wait for all replicas to acknowledge (consistent, slower) or just write to one and propagate (faster, potentially stale)? For example, MongoDB configured for majority write concern is PC/EC — it's consistent even without partitions, at higher latency. DynamoDB by default is PA/EL — available during partitions, low latency normally. PACELC is more useful day-to-day because partitions are rare but the latency/consistency tradeoff is constant."

---

## 10. Common Mistakes

- **Thinking CAP means you always choose two of three** — Partition tolerance isn't optional for distributed systems. The real choice is CP vs AP.
- **Claiming "I'll have both"** — Not possible during an actual partition
- **Confusing CAP Consistency with ACID Consistency** — They are completely different concepts
- **Not considering how often partitions actually happen** — They're more common than people think (network blips, rolling restarts, AZ issues)
- **Applying CP/AP at a system level** — The same system can be CP for critical operations (payment processing) and AP for others (analytics)


---

## 12. Real System Design Examples

| System | CP or AP | Reasoning |
|--------|----------|-----------|
| WhatsApp | AP (messages) | Message delivery is eventually consistent; retries handle failures |
| Instagram | AP (likes/counts) | Like count can be stale; CP (login/auth) for account access |
| Uber | CP (pricing/trip state) | Surge pricing must be consistent; AP (driver location) is fine stale |
| YouTube | AP (view counts) | View counts update with seconds delay; CP for account billing |
| Dropbox | CP (file metadata) | Two devices must agree on file state; conflict resolution if AP |
| Netflix | AP (content delivery) | CDN cache is eventually consistent — never block on stale content |
| Ticketmaster | CP (seat inventory) | MUST prevent double-selling; consistency over availability |
| Twitter/X | AP (tweet counts/feeds) | Feed staleness is fine; CP for login/payments |
| URL Shortener | AP (redirect) | Stale redirect temporarily acceptable; CP for write of new short URL |

---

## 13. Scaling Journey

```
Small Scale:
→ Single-region, single DB. No real partition risk. Both C and A in practice.

Medium Scale:
→ Add read replicas (async replication = AP for reads).
→ First real CAP decision: do replicas serve reads immediately (AP)
  or must they wait for sync (CP)?

Large Scale:
→ Multi-AZ deployment. Real partition risk between AZs.
→ Must explicitly choose: ZooKeeper-style (CP) or Cassandra-style (AP)?
→ Different data stores for different data types based on CP/AP need.

Hyperscale:
→ Multi-region. Inter-region latency makes CP expensive (RTT ~150ms for sync writes).
→ Most systems become AP at this scale for write performance.
→ Application-level idempotency, conflict resolution, and CRDT-style data structures
  handle consistency needs that the storage layer can't guarantee.
```

---

## 14. Decision Framework

```
Requirements
  → "What is the cost of returning stale data?"
    Financial loss / regulatory violation → CP
    Slightly stale UI → AP

  → "What is the cost of being unavailable?"
    Revenue loss per second → AP
    Incorrect action worse than no action → CP

Constraints
  → Cross-region latency makes synchronous CP expensive
  → Team ability to implement app-level idempotency for AP safety

Trade-Off Analysis
  → CP: correct data, potential unavailability
  → AP: always responds, potential stale data

Decision Framework
  → For WRITES: Can I tolerate a write being lost? No → CP
  → For READS: Can I tolerate stale data? Yes → AP
  → Different answer for different operations → use both
```

---
---

# TRADE-OFF 3: Strong Consistency vs Eventual Consistency

---

## 1. Concept Overview

**What it is:** Strong consistency means every read after a write sees the latest write — globally, immediately. Eventual consistency means replicas will converge to the same value "eventually" — but reads during the propagation window may return stale data.

**Why it exists:** Strong consistency requires coordination between replicas — adds latency. Eventual consistency requires no coordination — higher throughput, lower latency, at the cost of temporary disagreement.

**Why interviewers ask:** This is the practical, day-to-day manifestation of CAP theorem. Almost every system with replication makes this choice. Interviewers want to see you understand the operational implications, not just the theory.

---

## 2. Visual Diagram

```
Strong Consistency (Synchronous Replication):
Client → Primary DB → waits for ✅ from ALL replicas → responds to Client
Result: Every read sees latest write. Latency: ~200ms (round trip to all replicas)

Write: Primary ──sync──▶ Replica A ──ACK──▶ Primary ──sync──▶ Replica B ──ACK──▶ Respond

Eventual Consistency (Asynchronous Replication):
Client → Primary DB → responds IMMEDIATELY → (background) propagates to replicas
Result: Replica may return old value for 10-500ms. Latency: ~10ms

Write: Primary → Respond (fast!) ──async later──▶ Replica A ──async later──▶ Replica B
Read from Replica at t+50ms: might still return old value ← stale read window
```

---

## 3. Option A — Strong Consistency

**Definition:** After a write completes, all subsequent reads from ANY node return the new value. Achieved via synchronous replication, quorum writes (W+R > N), or single-node writes.

**Advantages:**
- No stale reads ever — simplest mental model
- No application-level conflict resolution needed
- Safe for financial, transactional, inventory systems
- Simplifies debugging — state is always deterministic

**Disadvantages:**
- Higher write latency — must wait for slowest replica
- Reduced availability — if a replica is down, write may fail or block
- Lower write throughput — coordination overhead at every write
- Expensive in multi-region — synchronous replication across regions adds 150ms+ RTT

**Cost:** Higher infrastructure cost (synchronous replication needs low-latency, high-bandwidth links between nodes).

---

## 4. Option B — Eventual Consistency

**Definition:** Writes are acknowledged after reaching one node (or a quorum less than all). Replicas converge over time through background propagation. Reads during the window may be stale.

**Advantages:**
- Very low write latency — don't wait for all replicas
- High write throughput — no blocking on slow replicas
- High availability — a slow/unreachable replica doesn't block writes
- Works well across regions — don't need low-latency inter-region links

**Disadvantages:**
- Stale reads possible — must handle in application logic
- Conflict resolution needed — two replicas may receive different writes
- Harder to reason about — "what did a user actually see?" is unclear
- Requires application-level patterns (idempotency, read-your-writes session routing)

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬─────────────────────────────────┐
│ Factor              │ Strong Consistency             │ Eventual Consistency              │
├────────────────────┼──────────────────────────────┼─────────────────────────────────┤
│ Read correctness    │ Always current               │ Possibly stale (window: ms-secs) │
│ Write latency       │ Higher (waits for replicas)  │ Lower (immediate ack)            │
│ Write throughput    │ Lower (coordination)         │ Higher (no coordination)         │
│ Availability        │ Lower (replica dependency)   │ Higher (any node can respond)    │
│ App complexity      │ Lower (no stale handling)    │ Higher (must handle stale)       │
│ Multi-region cost   │ Very high (sync RTT)         │ Low (async background sync)      │
│ Conflict handling   │ Not needed (no conflicts)    │ Needed (LWW, CRDT, app merge)   │
│ Use case fit        │ Finance, auth, inventory      │ Social, analytics, CDN, caches   │
└────────────────────┴──────────────────────────────┴─────────────────────────────────┘
```

---

## 6. When To Choose Strong Consistency

- **Bank account balances:** Reading stale balance leads to overdrafts
- **Authentication token revocation:** Revoked tokens MUST not be accepted
- **Seat/ticket reservation:** One seat, one buyer
- **Distributed lock acquisition:** Exactly one service holds the lock at a time
- **Payment idempotency checks:** Must know if a payment was processed before re-processing
- **Leader election (ZooKeeper, etcd):** Exactly one leader at any time

---

## 7. When To Choose Eventual Consistency

- **Social media like/comment counts:** 10-second old count is fine
- **User profile updates:** Name change propagating in 1 second is acceptable
- **Shopping cart:** Items in cart can be eventually consistent
- **DNS propagation:** TTL-based eventual consistency is the foundation of DNS
- **CDN cache:** Cached content can be N minutes behind origin
- **Email read/unread status:** Cross-device sync eventually is fine
- **Analytics dashboards:** 30-second lag in metrics is acceptable for most businesses

---

## 9. Strong Interview Answers

**Q: "User changes password. You're using eventual consistency. Is this safe?"**

> "No, and here's the specific attack: user A's session is compromised, they change their password to revoke the attacker's access. With eventual consistency, the old password hash is still valid on replicas for N seconds. The attacker can authenticate against a stale replica during that window. The fix: password changes MUST be strongly consistent — invalidate all sessions synchronously. Even in an AP system like Cassandra, you can use QUORUM consistency level for specific high-stakes operations while using eventual consistency for everything else. This is the tunable consistency approach."

**Q: "How do you implement 'read your own writes' in an eventually consistent system?"**

> "Three approaches. One: route the same user's reads to the same replica using sticky session affinity — that replica will eventually get the write, and since all their reads hit the same replica, they see their own writes. Two: timestamp-based — after a write, store the write timestamp in the user's session. When reading, if the replica's replication lag is behind that timestamp, redirect the read to the primary. Three: use the primary for reads for a brief window after any write from that user. Most applications implement option two — it's explicit about what we're tracking and doesn't permanently sticky-bind users to replicas."

---

## 10. Common Mistakes

- **Thinking "eventual" means "unreliable"** — eventual consistency systems (Cassandra, DynamoDB) are very reliable; data is eventually consistent, not lost
- **Not specifying the staleness window** — "eventually" could mean 50ms or 50 hours; always quantify
- **Using strong consistency for EVERYTHING** — unnecessary performance cost for analytics data
- **Not handling conflict resolution** — just picking "eventual consistency" without designing how conflicts resolve leaves a dangerous gap
- **Ignoring "read your own writes"** — the most common user-visible bug in eventually consistent systems

---

## 12. Real System Design Examples

| System | Model | Specific Use |
|--------|-------|-------------|
| WhatsApp | Eventual (message delivery order) | Messages may arrive slightly out of order; retries ensure delivery |
| Instagram | Strong (auth) + Eventual (likes) | Your own post appears immediately; like counts lag by seconds |
| Uber | Strong (trip state machine) | MUST agree on trip status; driver can't both start and not start a trip |
| YouTube | Eventual (view count) | Famously, view counts are delayed; YouTube freezes counts at ~300 for bot detection |
| Dropbox | Strong (file version) | Two edits to same file create a conflict — not silently overwritten |
| Netflix | Eventual (recommendation) | Recommendations lag by hours — ML model batch update |
| Ticketmaster | Strong (seat reservation) | Two-phase hold → confirm prevents double-booking |
| Twitter/X | Eventual (timeline) | Home timeline is fan-out-on-write with eventual propagation |

---

## 13. Scaling Journey

```
Small Scale:
→ Single primary. No replicas. Everything is strongly consistent trivially.
→ No replication = no consistency choice to make.

Medium Scale:
→ Add a read replica (async). Now reads from replica are eventually consistent.
→ First decision: do ALL reads go to primary (strong, expensive) or replicas (eventual, cheap)?

Large Scale:
→ Multi-AZ, multi-replica. Replication lag becomes measurable (10-100ms).
→ Partition critical reads (auth, payments) to synchronous replicas.
→ Partition analytics reads to eventually consistent replicas.

Hyperscale:
→ Multi-region. Synchronous replication across regions = 150ms+ write latency.
→ Must accept eventual consistency for global writes.
→ Design entire system around eventual consistency with application-level patterns
  (idempotency, CRDT, LWW timestamps, conflict resolution) to maintain correctness.
```

---

## 14. Decision Framework

```
Question: "Is it acceptable for a user to read stale data for X milliseconds/seconds?"

YES → Eventual Consistency (optimize for throughput + availability)
NO  → Strong Consistency (pay the latency/availability cost)

Sub-question: "What is the WORST CASE if a user reads stale data?"
→ Minor UI inconsistency → Eventual
→ Financial loss / security breach / inventory oversell → Strong

Sub-question: "Is multi-region required?"
→ YES → Strong consistency is very expensive; design for eventual with app-level safety
→ NO  → Strong consistency is more feasible (single region round trips are fast)

Rule of Thumb:
  Strong Consistency = write once, read correctly everywhere, always
  Eventual Consistency = write fast, read fast, accept brief windows of disagreement
```

---
---

# TRADE-OFF 4: Push vs Pull Architecture

---

## 1. Concept Overview

**What it is:** The choice of whether the server PUSHES data to clients when it's available (push), or clients must POLL the server to ask "is there new data?" (pull).

**Why it exists:** Some data changes unpredictably and clients need it immediately (chat messages, live scores). Others change predictably or can tolerate a delay (email, analytics reports). The architecture for each is fundamentally different.

**Why interviewers ask:** This trade-off directly affects fan-out cost, storage requirements, latency, and scalability. Getting this wrong in a social feed or notification system design is a classic mistake.

---

## 2. Visual Diagram

```
PUSH MODEL (Fan-out on Write):
When User A posts:
  A → Post Service → Fan-out → Write to 1M follower inboxes
  Each follower's inbox is pre-populated.
  Reader: inbox lookup → O(1) read, instant

PULL MODEL (Fan-out on Read):
When User A posts:
  A → Post Service → Write to A's post table only
  Reader: "Give me feed" → look up all followed users → fetch posts → merge → sort
  Read is expensive (O(followed_count) × O(posts_per_user) lookups)

HYBRID (Twitter's actual approach):
  Regular users (< 1M followers) → PUSH to follower inboxes (fast reads)
  Celebrity users (> 1M followers) → PULL at read time (avoid fan-out to millions)
  Read = PUSH feed + just-in-time PULL from celebrity accounts you follow
```

---

## 3. Option A — Push (Fan-out on Write)

**Definition:** When new content is created, immediately write it to every consumer's "inbox" or cache. Readers always read from their own pre-populated feed.

**Advantages:**
- O(1) read — just fetch your pre-computed feed
- Low read latency — content is already where it needs to be
- Works for millions of concurrent readers — no coordination at read time

**Disadvantages:**
- Fan-out cost on write — 1 write from a user with 1M followers = 1M writes
- Storage explosion — same post stored in 1M places
- Hot celebrities break it — Katy Perry's tweet can't fan-out to 150M followers in real-time
- Write amplification — popular content causes write spikes

**Cost:** High storage, high write throughput capacity needed. Read infrastructure is simple.

---

## 4. Option B — Pull (Fan-out on Read)

**Definition:** Content is written once. When a user wants their feed, the system reads from all sources they follow, fetches content, merges and sorts it.

**Advantages:**
- O(1) write — just write the post once, anywhere
- No storage duplication — each post stored exactly once
- Works for celebrities — no fan-out regardless of follower count
- Simpler write path

**Disadvantages:**
- O(N) read — N = number of accounts followed; expensive per request
- Read latency is high — multiple queries, in-memory merge, sort
- Doesn't scale well at read tier — can't cache (feed is unique per user per moment)
- Under read load: database hit on every feed refresh

**Cost:** Read infrastructure is expensive. Storage is minimal. Write infrastructure is simple.

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ Push (Fan-out on Write)        │ Pull (Fan-out on Read)            │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Write complexity    │ High (fan-out to N places)    │ Low (write once)                  │
│ Read complexity     │ Low (O(1) inbox lookup)       │ High (O(N) followed-user lookups) │
│ Read latency        │ Very low (pre-computed)       │ Higher (computed at read time)    │
│ Write latency       │ Higher (fan-out cost)         │ Very low (write once)             │
│ Storage             │ High (N copies per post)      │ Low (1 copy per post)             │
│ Celebrity handling  │ Problem (millions of writes)  │ Natural (no fan-out)              │
│ Cache-ability       │ Cacheable (inbox doesn't change│ Hard (unique per user/time)       │
│                    │ until next write)              │                                   │
│ Scalability         │ Read scales well              │ Write scales well                 │
│ Best For            │ Read-heavy (most social feeds)│ Write-heavy, celebrities          │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose Push

- **Chat applications (WhatsApp, Slack):** Messages must be delivered instantly — push via WebSocket
- **Email notifications:** Email is pushed to inbox when received
- **Low-to-medium follower accounts:** Pushing to 1,000 followers is trivial
- **Real-time dashboards:** New metric → push to all dashboard viewers
- **Stock tickers, live scores:** Data changes must reach all subscribers immediately
- **Companies:** Twitter's regular user feeds, Instagram's feed for normal accounts

---

## 7. When To Choose Pull

- **High follower count accounts:** Katy Perry with 150M followers — pull from her feed at read time
- **Infrequently-read feeds:** Weekly newsletter digest — compute at read time
- **Low-write, high-staleness-tolerance data:** "Recently published" articles list
- **Search results:** Always computed at query time
- **Companies:** Twitter for celebrities; RSS feed readers; search engines

---

## 9. Strong Interview Answers

**Q: "You're designing Twitter. User A has 150M followers. She tweets. How do you handle fan-out?"**

> "Pure push would require 150M database writes synchronously — this would take minutes even with parallel writes, and the database can't sustain that write burst for a single tweet. Twitter's actual solution is the hybrid approach. For users below a threshold (say, 10,000 followers), push to all follower inboxes asynchronously via a message queue — their followers get it in their pre-computed timeline within seconds. For 'celebrity' accounts above the threshold, we DON'T pre-push. Instead, when a follower loads their feed, we fetch the push-based feed for regular accounts they follow, PLUS make an additional pull for any celebrity accounts they follow, merge them in real-time, and sort by timestamp. The cost is a few extra real-time lookups per feed load for users who follow celebrities. This is O(celebrity_count_followed) additional lookups, not O(celebrity_follower_count) fan-out."

---

## 10. Common Mistakes

- **Choosing pure push without considering high-follower-count accounts** — immediately breaks at scale
- **Choosing pure pull without accounting for high follow-counts** — user following 10,000 accounts = 10,000 DB reads per feed load
- **Not considering the asynchronous nature of push fan-out** — candidates often assume fan-out is synchronous (it's not; it's queued and eventually consistent)
- **Ignoring storage costs of push** — same post in a million places; calculate before committing


---

## 12. Real System Design Examples

| System | Model | Details |
|--------|-------|---------|
| WhatsApp | Push (WebSocket) | Messages pushed to connected client instantly |
| Instagram | Hybrid | Push for regular feeds; pull merged with celebrity posts at read time |
| Uber | Push (driver location) | Server pushes location updates to rider; driver status to dispatch |
| YouTube | Pull (recommendations) | Recommendations computed at page load based on viewing history |
| Dropbox | Push (file changes) | Changes pushed to all connected devices via long-polling |
| Netflix | Push (playback ready) | Video ready to play pushed to client buffer |
| Ticketmaster | Pull (seat availability) | Client polls seat map — too dynamic for per-seat push |
| Twitter/X | Hybrid | Push for regular users; pull for celebrities; pre-computed cached timeline |

---

## 13. Scaling Journey

```
Small Scale (< 100K users):
→ Pull model is fine. 100 followed accounts × 100K users = manageable DB reads.
→ Simple, no fan-out complexity.

Medium Scale (1M users):
→ Pull reads are now expensive. Each feed refresh = N × DB reads × M users.
→ Add caching layer for frequently-accessed user post feeds.
→ Consider push for inbox pre-computation.

Large Scale (100M users):
→ Push for regular users (up to 10K followers). Queue-based fan-out.
→ Identify celebrities. Pull at read time for celebrity content.
→ Redis for inbox storage (O(1) read for pre-computed timelines).

Hyperscale:
→ Dedicated "timeline service" with pre-computed feeds per user in Redis.
→ Sophisticated heuristics for push/pull boundary (follower count, posting frequency).
→ Feed stored as a Redis sorted set of post IDs (not full content) — pull content separately.
```

---

## 14. Decision Framework

```
Analyze your read:write ratio and entity cardinality:

Many readers, few writers (social feed, notifications):
  → PUSH: Optimize reads. Pre-compute. Writers pay one-time fan-out cost.

Few readers, many writers (event logs, metrics streams):
  → PULL: Readers compute what they need at query time. Don't pre-compute.

Celebrity/hotspot problem (any entity with N >> average followers):
  → HYBRID: Push for typical case, pull for outliers.

Real-time requirement (< 1 second delivery):
  → PUSH always: poll-based pull can't guarantee sub-second delivery.

Offline client handling (mobile apps):
  → PUSH with notification + lazy PULL: send notification,
    client pulls content when it reconnects.
```

---
---

# TRADE-OFF 5: Synchronous vs Asynchronous Communication

---

## 1. Concept Overview

**What it is:** Synchronous communication — the caller WAITS for the response before continuing. Asynchronous communication — the caller sends the message and CONTINUES IMMEDIATELY; the response (if any) comes later.

**Why it exists:** Not all operations need an immediate response. Sending an email confirmation, resizing an uploaded image, updating a search index — these can happen after the response is already sent to the user. Making users wait for these slow operations is a poor user experience.

**Why interviewers ask:** It's central to microservices design, API design, and resilience patterns. Understanding when to use async reveals whether you think about user experience, fault isolation, and system coupling.

---

## 2. Visual Diagram

```
Synchronous:
Client → [Service A] → [Service B] → [DB] → responds to A → responds to Client
Client is BLOCKED during the entire chain. Total latency = sum of all hops.
If Service B is slow: Client is slow. If Service B is down: Client gets error.

Asynchronous (via Message Queue):
Client → [Service A] → responds 200 OK immediately
                    ↓
                 [Queue]
                    ↓ (later, independently)
              [Service B] → [DB]
Client is NOT blocked. Service B failure doesn't affect the client response.
```

---

## 3. Option A — Synchronous Communication

**Definition:** Caller sends request and blocks, waiting for the complete response before continuing execution. REST HTTP calls, gRPC unary calls are synchronous.

**Advantages:**
- Simple to reason about — request-response is intuitive
- Immediate error feedback — caller knows instantly if something failed
- Easy to implement — no message broker, no queue
- Transactional — response confirms the operation completed

**Disadvantages:**
- Tight temporal coupling — both services must be available simultaneously
- Cascading failures — slow downstream = slow upstream = slow user
- Latency amplification — user latency = sum of all downstream calls
- Hard to scale independently — both sides must scale together
- Retry logic falls on the caller

---

## 4. Option B — Asynchronous Communication

**Definition:** Caller sends a message (to a queue, stream, or event bus) and continues immediately. Consumer processes the message independently, potentially at a different rate and time.

**Advantages:**
- Temporal decoupling — services don't need to be available simultaneously
- Failure isolation — consumer failure doesn't fail the producer
- Load leveling — queue absorbs traffic spikes; consumer processes at sustainable rate
- Independent scaling — producer and consumer scale separately
- Retry is the queue's responsibility, not the caller's

**Disadvantages:**
- No immediate confirmation of completion — user gets "your request is processing"
- More complex — requires message broker infrastructure (Kafka, SQS, RabbitMQ)
- Debugging harder — tracing async message flows across services is complex
- Ordering guarantees may be lost (unless explicitly preserved)
- At-least-once delivery requires idempotent consumers

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ Synchronous                   │ Asynchronous                      │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Response time       │ Immediate (complete result)   │ Deferred ("accepted")             │
│ Coupling            │ Tight (both must be up)       │ Loose (queue buffers failures)   │
│ Error handling      │ Immediate feedback            │ Consumer must handle+retry        │
│ Complexity          │ Low (direct call)             │ Higher (broker, consumer, retry)  │
│ Scalability         │ Must scale together           │ Scale independently               │
│ Latency to user     │ Full operation latency        │ Just enqueue latency (fast!)     │
│ Failure isolation   │ Low (cascade)                 │ High (failure doesn't propagate)  │
│ Throughput          │ Rate-limited by slowest link  │ Queue absorbs spikes              │
│ Debugging           │ Easy (call stack)             │ Hard (distributed trace needed)   │
│ Cost                │ Lower infra (no broker)       │ Higher (broker infra)             │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose Synchronous

- **User needs immediate result:** Login authentication — user can't wait; must know NOW
- **Simple, fast operations:** Checking if a username is available
- **Transactional requirement:** Payment authorization — must know if it succeeded before showing "order confirmed"
- **Low latency, low failure risk:** Intra-service calls within the same data center
- **Simple systems with limited services:** Early-stage startup with 2-3 services

---

## 7. When To Choose Asynchronous

- **Long-running operations:** Video encoding, ML model inference, PDF generation
- **Notifications:** Email confirmations, push notifications — user doesn't need to wait
- **Best-effort operations:** Updating analytics, search index, audit logs
- **Fan-out to many consumers:** One event → multiple independent consumers (email + analytics + recommendation update)
- **High-traffic spikes:** Payment spike at midnight → queue absorbs burst; payment processor handles at sustainable rate
- **Inter-service events in microservices:** Order placed → inventory update + email + recommendation update (all async)

---

## 9. Strong Interview Answers

**Q: "You have a synchronous call chain A → B → C → D. D becomes slow. Walk me through the impact."**

> "This is the cascading slow-down problem. D slows to 5 seconds per request. C is waiting for D — its threads are blocked, response times rise. C's thread pool fills up. New requests to C start queuing. B is waiting for C — same pattern. B's threads fill. A is waiting for B. Eventually A's thread pool fills. The entire chain becomes saturated. This is why synchronous call chains are dangerous. The fix is: timeout at each level (so A doesn't wait more than 2 seconds for B), circuit breaker at the C→D boundary (if D is consistently slow, stop trying and fail fast), and bulkhead isolation so D's slowness doesn't exhaust all threads at C. Longer term, the question is: does A actually need D's result synchronously? If not, make D async."

**Q: "How do you tell the user their async video upload is processing and notify them when it's done?"**

> "Three common patterns. One: polling — return a job ID immediately; client polls /jobs/{id}/status every 5 seconds. Simple but creates unnecessary requests. Two: WebSocket or SSE — maintain a persistent connection from client to notification service; when the job completes, push a notification over the connection. Good for web clients. Three: push notification (mobile/email) — after async job completes, send a push notification or email. Works even when the client isn't connected. YouTube uses the third approach — you upload, close the browser, get an email when it's processed. The right answer depends on the use case: real-time UI update → WebSocket; mobile background → push notification; background batch job → email."

---

## 10. Common Mistakes

- **Making everything async** — user clicks "Buy Now" and gets "your order is queued" — unacceptable; payment MUST be synchronously confirmed
- **Not designing for idempotency in async consumers** — at-least-once delivery means duplicate messages; consumers must be idempotent
- **No dead letter queue** — failed messages silently lost; always add a DLQ
- **No timeout on synchronous calls** — default is often infinite wait; always set explicit timeouts
- **Assuming async means eventually** — async doesn't mean slow; RabbitMQ/Kafka can process messages in milliseconds


---

## 12. Real System Design Examples

| System | Sync | Async | Reasoning |
|--------|------|-------|-----------|
| WhatsApp | Message delivery ACK (sync) | Multi-device sync (async) | User sees sent; other devices sync eventually |
| Instagram | Upload accepted (sync) | Resize, CDN distribution (async) | User gets immediate confirmation |
| Uber | Driver matched (sync) | ETA calculation updates (async) | Match is critical; ETA can update |
| YouTube | Upload ACK (sync) | Encoding pipeline (async) | Encoding takes minutes; user can close browser |
| Dropbox | File upload confirmed (sync) | Sync to other devices (async) | Confirm receipt; sync eventually |
| Netflix | Play request auth (sync) | Watch history update (async) | Must authorize to play; history can lag |
| Ticketmaster | Seat hold (sync) | Confirmation email (async) | Seat lock MUST be immediate |
| Twitter | Tweet posted (sync) | Timeline fan-out (async) | Post confirmed; followers see it in seconds |

---

## 13. Scaling Journey

```
Small Scale:
→ Synchronous everything. Simple, direct service calls. 
→ Works for low traffic; failures visible immediately.

Medium Scale:
→ Identify "slow path" operations (email, image processing).
→ Extract to async workers (SQS + Lambda, or Celery workers).
→ Keep "fast path" (auth, payment confirmation) synchronous.

Large Scale:
→ Event-driven architecture: domain events published to Kafka/SQS.
→ Multiple consumers per event (email service, analytics, search indexer).
→ Saga pattern for distributed transactions via async messages.

Hyperscale:
→ Nearly everything is async except the user-facing critical path.
→ Even the critical path may have async sub-steps with eventual guarantees.
→ CQRS: writes are async events; reads are from eventually-consistent projections.
```

---

## 14. Decision Framework

```
Ask: "Does the user need to wait for this operation's result?"

YES → Synchronous (payment, authentication, immediate data retrieval)
NO  → Asynchronous (email, image processing, notifications, analytics)

Ask: "What happens if the downstream service is down?"

User must know immediately → Synchronous (fail fast with error)
User can be notified later → Asynchronous (queue absorbs failure)

Ask: "What's the fan-out of this operation?"

Single consumer → Either works; sync is simpler
Multiple consumers → Asynchronous (event bus fans out to all consumers)

Rule: Synchronous for the critical path visible to the user.
      Asynchronous for everything else.
```

---
---

# TRADE-OFF 6: Cache vs Database

---

## 1. Concept Overview

**What it is:** The choice between serving data from a fast in-memory cache (Redis, Memcached) or going to the source-of-truth database on every request. Caching stores a copy of data in faster storage for repeated access.

**Why it exists:** Databases are durable but slow (disk I/O, query parsing, index traversal). Memory is fast but volatile and limited. When the same data is requested repeatedly, re-computing it from the database is wasteful. A cache keeps a copy in fast memory.

**Why interviewers ask:** Caching is present in almost every large system. Knowing WHAT to cache, HOW LONG, and how to handle invalidation is a critical senior engineering skill. Cache-related bugs (stale data, cache stampede, thundering herd) are common failure modes.

---

## 2. Visual Diagram

```
Without Cache:
Client → App Server → Database (50ms)
Every request hits the database. 10,000 req/sec = 10,000 DB queries/sec.

With Cache:
Client → App Server → [Cache MISS] → Database (50ms) → populate cache
                   → [Cache HIT ] → Cache (0.5ms)
Cache hit rate 95%: 10,000 req/sec = 500 DB queries/sec. 20x reduction.

Cache Invalidation Problem:
DB: price = ₹999  →  Update DB: price = ₹899  →  Cache still says ₹999 ← STALE!
                                                   User sees wrong price until TTL expires
```

---

## 3. Option A — Cache (Redis / Memcached)

**Definition:** In-memory key-value store that sits in front of the database. Serves frequently-accessed data in sub-millisecond time without touching the database.

**Advantages:**
- ~0.5ms latency vs ~50ms database latency — 100x faster
- Dramatically reduces database load (DB sees only cache misses)
- Can serve millions of requests per second per Redis instance
- Reduces cost — fewer database queries = smaller database needed
- Absorbs traffic spikes — cache can serve peaks without DB scaling

**Disadvantages:**
- Stale data — cache and DB can be out of sync
- Cache invalidation is hard — one of the hardest problems in CS
- Eviction under memory pressure — LRU eviction can cause cache stampedes
- Not durable by default — Redis restart loses cache (but it's a cache, so OK)
- Cache cold start — fresh deployments have empty caches; first wave hits DB

**Cost:** Redis cluster is cheaper than scaling the database. But adds operational complexity.

---

## 4. Option B — Database Direct

**Definition:** Every request goes to the database. No caching layer. Database is the single source of truth for every read.

**Advantages:**
- Always fresh data — no staleness possible
- No cache invalidation complexity
- Simpler system — fewer moving parts
- Consistency guaranteed — reads always reflect latest writes
- Suitable when data changes very frequently (cache would have very low hit rates)

**Disadvantages:**
- High latency — disk I/O on every read (~50ms vs ~0.5ms)
- Database becomes a bottleneck at scale
- Higher cost — need a powerful (and expensive) database to handle read load
- Doesn't absorb spikes — sudden traffic increase → sudden DB load increase

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ Cache                         │ Database Direct                   │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Read latency        │ 0.3-1ms (memory)              │ 10-100ms (disk + network)         │
│ Data freshness      │ Stale (TTL-dependent)         │ Always fresh                      │
│ Consistency         │ Eventual (cache lags DB)      │ Strong                            │
│ Throughput          │ 100K-1M ops/sec/node          │ 1K-100K ops/sec (varies)          │
│ Scalability         │ Horizontal (cluster easily)   │ Harder to scale writes            │
│ Cost                │ Lower at scale                │ Higher (needs bigger DB)          │
│ Complexity          │ High (invalidation, eviction) │ Low (just query the DB)           │
│ Availability        │ High (DB down = serve cache)  │ DB down = system down             │
│ Suitable data       │ Frequently read, rarely write │ Frequently written, must be fresh │
│ Failure mode        │ Stale data                    │ Slow/unavailable under load       │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose Cache

- **Product catalog pages:** Product descriptions change rarely; can be stale for 5 minutes
- **User profile data:** Changes infrequently; stale by seconds is acceptable
- **Session data:** Must be fast (every API call validates session); Redis is natural
- **Leaderboards/rankings:** Recomputing the top 100 users from DB every request is expensive
- **API rate limiting counters:** Must be fast and atomic — Redis INCR is perfect
- **Homepage content, trending topics:** Changes every few minutes; perfect cache TTL
- **DB query results that are expensive:** Aggregations, JOINs that take 500ms to compute

---

## 7. When to Go Database Direct (No Cache)

- **User's own balance/transaction history:** Stale balance is dangerous; always hit DB
- **Data changes very frequently:** Stock tick prices change every millisecond — cache would always miss or be stale
- **Highly personalized, uncacheable data:** Every request is unique (personalized recommendations)
- **Low-traffic systems:** Cache adds complexity without benefit
- **Real-time critical path:** Fraud detection must see the LATEST transaction data

---

## 9. Strong Interview Answers

**Q: "What is a cache stampede and how do you prevent it?"**

> "A cache stampede happens when a popular cache key expires (TTL runs out), and simultaneously hundreds or thousands of requests all get a cache MISS and all rush to the database to recompute the value. The database gets hit with hundreds of expensive queries simultaneously, potentially overwhelming it. Three prevention strategies. One: mutex/lock — when cache misses, only one request goes to the DB; others wait for the cache to be repopulated. Risk: lock contention. Two: probabilistic early expiration — instead of all entries expiring exactly at TTL, add random jitter (TTL = base + random(0, 10%)). Different requests expire at different times, smoothing the load. Three: background refresh — before the key expires, proactively kick off a background job to refresh it. The key always exists; the background job keeps it fresh. This is the best approach for predictably expensive, frequently-accessed keys."

**Q: "Explain cache-aside vs write-through caching."**

> "Cache-aside (lazy loading): the application checks the cache first. On a miss, it reads from the database and populates the cache. Writes go to the database only; the cache entry is invalidated or left to expire. Advantage: cache only contains what's actually been requested. Disadvantage: first request after expiry hits the DB (cache miss penalty). Write-through: every write goes to both the cache and the database simultaneously. Cache is always warm. Advantage: no cache misses for recently written data. Disadvantage: write latency doubles (must wait for both); cache fills with data that may never be read. Write-behind (write-back): writes go to cache only; a background process writes to the database asynchronously. Advantage: very low write latency. Disadvantage: data loss risk if cache fails before the DB write completes. For most systems, I'd recommend cache-aside with a reasonable TTL — it's simple, handles failures gracefully (cache down = reads go to DB directly), and caches exactly the data that's being accessed."

---

## 10. Common Mistakes

- **Caching things that don't benefit from caching** — highly-unique queries, frequently-changing data
- **No cache invalidation strategy** — cache updates only on TTL; users see stale data too long
- **Not planning for cache cold start** — fresh deployments cause a DB spike as cache warms
- **Assuming cache is always faster** — a cache miss is SLOWER than a direct DB read (two round trips)
- **Not monitoring cache hit rate** — if hit rate < 50%, your cache isn't helping
- **Caching sensitive data inappropriately** — shared cache serving user-specific private data to wrong users
- **Single Redis node with no replica** — Redis node failure = total cache loss = DB spike

---

## 12. Real System Design Examples

| System | What's Cached | TTL Strategy |
|--------|---------------|-------------|
| WhatsApp | User profile info | Short TTL; invalidate on profile update |
| Instagram | Post metadata, follower count | 60-second TTL; accept slight staleness |
| Uber | Driver current location | 5-second TTL (updates so frequently, cache barely helps) |
| YouTube | Video metadata (title, duration) | Long TTL (1 hour); rarely changes |
| Dropbox | File metadata listing | Invalidated on file change events |
| Netflix | Movie catalog, user's watch history | Catalog: 1 hour TTL. Watch history: short TTL or no cache (personal) |
| Ticketmaster | Event details | Long TTL; seat availability: NO CACHE (must be real-time) |
| Twitter | User's timeline | Short TTL or invalidate on new post in timeline |
| URL Shortener | Short URL → long URL mapping | Long TTL or no expiry (mappings don't change) |

---

## 13. Scaling Journey

```
Small Scale (< 10K req/sec):
→ No cache needed. Database handles the load.
→ Add caching only if a specific slow query is identified.

Medium Scale (10K-100K req/sec):
→ Add Redis in cache-aside mode for hot data.
→ Target pages, product data, user sessions.
→ Hit rates should be > 80% to justify cache.

Large Scale (100K-1M req/sec):
→ Redis Cluster (multiple shards) for capacity.
→ Replicas for read scaling within Redis.
→ Different TTLs per data type based on change frequency.
→ Cache warming strategy for deployments.

Hyperscale:
→ Tiered caching: L1 in-process cache (< 1ms), L2 Redis (1ms), L3 database.
→ CDN for static/quasi-static data.
→ Per-datacenter cache to avoid cross-region latency.
→ Cache invalidation via event streams (DB change → Kafka → cache invalidation service).
```

---

## 14. Decision Framework

```
Should I cache this data?

Step 1: Read frequency
→ Read < 100 times/day → probably not worth caching
→ Read > 1000 times/day → strong caching candidate

Step 2: Compute/retrieval cost
→ Simple DB lookup < 10ms → maybe not (cache miss penalty could exceed benefit)
→ Complex query, JOIN, aggregation > 100ms → definitely cache

Step 3: Change frequency
→ Changes every second → low TTL or no cache (hit rate will be terrible)
→ Changes hourly → good cache candidate (TTL = 30 minutes, accept slight staleness)

Step 4: Staleness tolerance
→ Must be fresh (balance, inventory last unit) → NO CACHE or very short TTL
→ Can be slightly stale (product description, user profile) → CACHE with TTL

Decision: Cache if (read_frequency HIGH) AND (compute_cost HIGH) AND (staleness OK)
```

---

**END OF PART 1 (Trade-offs 1-6)**
*Continues in Part 2: Monolith vs Microservices, Replication vs Sharding, WebSocket vs Polling, REST vs GraphQL, Batch vs Stream Processing, Horizontal vs Vertical Scaling*

---
---

# TRADE-OFF 7: Monolith vs Microservices

---

## 1. Concept Overview

**What it is:** A monolith is a single deployable unit where all components run in one process. Microservices decompose the application into independently deployable, independently scalable services that communicate over the network.

**Why it exists:** Monoliths are simple to start with but become harder to change at scale (large teams, complex domains). Microservices solve team autonomy and independent scaling — at the cost of distributed systems complexity.

**Why interviewers ask:** This is a fundamental architectural decision with deep implications for team structure, deployment, scaling, and operational complexity. Interviewers want to see that you understand the REAL costs of microservices, not just their benefits.

---

## 2. Visual Diagram

```
MONOLITH:
┌──────────────────────────────────────────────────────────┐
│  Single Process / Single Deployment                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  User     │  │  Order    │  │ Payment   │  │ Notif.   │  │
│  │  Module   │  │  Module   │  │  Module   │  │  Module  │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
│              ↓ all share ONE database                      │
│         ┌─────────────────────────────┐                   │
│         │      Single DB               │                   │
│         └─────────────────────────────┘                   │
└──────────────────────────────────────────────────────────┘

MICROSERVICES:
┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
│  User     │  │  Order    │  │ Payment   │  │  Notif.   │
│  Service  │  │  Service  │  │  Service  │  │  Service  │
│  + DB     │  │  + DB     │  │  + DB     │  │  + DB     │
└─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
      │               │               │               │
      └───────────────┴───────────────┴───────────────┘
                        API Gateway / Event Bus
```

---

## 3. Option A — Monolith

**Definition:** All application functionality compiled and deployed as a single unit. In-process calls between modules. Single shared database.

**Advantages:**
- Simple development — one repo, one IDE, one deployment
- Easy debugging — single call stack, no distributed tracing needed
- No network latency between modules — in-process calls are nanoseconds
- Transactions are free — ACID across all modules via one DB
- Simple operations — one thing to deploy, monitor, and scale
- Fast initial development — no service boundary negotiation

**Disadvantages:**
- Scaling is all-or-nothing — can't scale just the payment module
- Large team friction — 50+ engineers on one codebase causes merge conflicts, slow builds
- Technology lock-in — entire system must use the same language/framework
- Deployment risk — every deploy affects all features
- Long build and test times as codebase grows

**Cost:** Low initially. High at scale (must scale everything even if only one module needs it).

---

## 4. Option B — Microservices

**Definition:** Application decomposed into independently deployable services, each owning its own data store, communicating over the network via APIs or events.

**Advantages:**
- Independent scaling — scale only the payment service if it's the bottleneck
- Team autonomy — each team owns and deploys their service independently
- Technology flexibility — Python for ML service, Go for high-throughput API, Java for payments
- Fault isolation — payment service bug doesn't take down user profiles
- Independent deployments — 10 deploys per day without coordinating all teams

**Disadvantages:**
- Distributed systems complexity — network failures, partial failures, timeouts
- No ACID transactions — cross-service operations need Saga pattern
- Operational overhead — dozens of services to deploy, monitor, and debug
- Network latency — in-process call becomes network call (microseconds → milliseconds)
- Service discovery, load balancing, distributed tracing all needed
- Data consistency challenges — each service has its own database

**Cost:** High — more infrastructure, more operational tooling, more specialized expertise.

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ Monolith                       │ Microservices                     │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Scalability         │ Scale everything (wasteful)   │ Scale individual services          │
│ Deployment          │ One deploy, all features      │ Independent deploys per service   │
│ Team size           │ Works for small teams         │ Needed for large, distributed teams│
│ Development speed   │ Fast early, slow later        │ Slow to set up, fast at scale     │
│ Operational effort  │ Low                           │ Very high                         │
│ Debugging           │ Easy (single stack trace)     │ Hard (distributed tracing)        │
│ Transactions        │ Free (ACID)                   │ Complex (Saga pattern)            │
│ Network overhead    │ None (in-process)             │ High (service-to-service calls)   │
│ Fault isolation     │ Poor (one bug = all down)     │ Good (failure contained)          │
│ Tech flexibility    │ Low (one stack)               │ High (polyglot)                   │
│ Complexity          │ Low initially, high at scale  │ High from day one                 │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose Monolith

- **Early-stage startup:** Domain not understood yet; service boundaries will be wrong; pivot speed matters
- **Small team (< 20 engineers):** Microservices overhead consumes more time than the benefits provide
- **Simple domain:** CRUD application with limited complexity doesn't need distributed architecture
- **Well-structured monolith:** A "modular monolith" with clean internal boundaries can scale to 100+ engineers with discipline
- **Companies:** Shopify ran on a Rails monolith for years (now modular monolith + some extracted services); Stack Overflow still runs primarily on a monolith

---

## 7. When To Choose Microservices

- **Large teams with different domains:** 50+ engineers where team autonomy and independent deployment matter
- **Dramatically different scaling needs per component:** Payment service at 1,000 req/sec, recommendation engine at 100,000 req/sec
- **Different technology requirements:** ML model serving needs Python/GPUs; API gateway needs Go; existing legacy service needs Java
- **High-reliability requirement:** Need to deploy payment fixes without deploying unrelated features
- **Companies:** Amazon, Netflix, Uber (after hitting monolith limits at scale)

---

## 9. Strong Interview Answers

**Q: "You're building a new product from scratch. 5 engineers. Monolith or microservices?"**

> "Monolith, without hesitation. Microservices solve two specific problems: team autonomy at scale, and dramatically different scaling requirements per component. With 5 engineers, you don't have team autonomy problems — you ARE the team. You probably don't know your scaling profile yet. And microservices would consume 40% of those 5 engineers' time on infrastructure: service discovery, distributed tracing, API gateways, container orchestration. That's time not spent on the product. Start with a well-structured modular monolith — clean internal interfaces between Payment, Order, User modules. When a specific component needs different scaling (maybe the search feature takes off), extract THAT one service. Use the Strangler Fig pattern: route the new service's traffic through the API gateway while the monolith still handles everything else. You get microservices benefits exactly where you need them, without the overhead everywhere."

**Q: "What's a distributed monolith and why is it the worst of both worlds?"**

> "A distributed monolith is when you've split services by technical layer (frontend, backend, database, API) or just arbitrarily, without true domain isolation. The tell-tale signs: services must be deployed in a specific order because they're tightly coupled, services share a database directly (bypassing APIs), changing one service always requires changing several others simultaneously, and a failure in one service takes down multiple others. You have all the operational complexity of microservices (network calls, distributed tracing, deployment overhead) with none of the benefits (independent scaling, fault isolation, team autonomy). The root cause is usually extracting services before understanding your domain boundaries. The fix is to invest in Domain-Driven Design — identify your bounded contexts, ensure each service owns its data, and communicate only through well-defined APIs or events."

---

## 10. Common Mistakes

- **Starting with microservices** at day one of a startup — building distributed complexity before you understand your domain
- **Sharing a database between microservices** — this is the #1 anti-pattern; creates tight coupling
- **Not handling partial failures** — service A calls B synchronously; B is slow; A accumulates threads, cascades
- **Creating too many services** ("nanoservices") — one service per class/function creates absurd network overhead
- **Underestimating operational overhead** — each service needs CI/CD, monitoring, alerting, logging, on-call rotation


---

## 12. Real System Design Examples

| System | Architecture | Why |
|--------|-------------|-----|
| WhatsApp | Service-oriented (not fully micro) | 50 engineers, 1B users — value simplicity |
| Instagram | Monolith → service extraction | Started Django monolith; extracted services as team grew |
| Uber | Microservices | Different scaling for matching, pricing, dispatch, maps |
| YouTube | Mix | Monolith core + specialized services for video processing |
| Dropbox | Modular monolith → services | Extracted storage (Magic Pocket) and sync selectively |
| Netflix | Full microservices | 700+ services; pioneered resilience patterns (Chaos Monkey) |
| Ticketmaster | Moving to microservices | Legacy monolith; gradual extraction of high-traffic services |
| Twitter | Monolith → microservices (painful) | The "Fail Whale" era drove microservices adoption |
| URL Shortener | Monolith (ideally) | Simple domain; two endpoints; microservices overkill |

---

## 13. Scaling Journey

```
0-10 engineers: Monolith. Ship fast. Learn the domain.
10-30 engineers: Modular monolith. Clean internal boundaries. Shared DB still OK.
30-100 engineers: Extract 2-3 services where REAL scaling or team boundary pain exists.
100+ engineers: Full service-oriented architecture. Conway's Law demands it.
Hyperscale: Microservices as default with strong platform teams providing infra.
```

---

## 14. Decision Framework

```
Do you have clear domain boundaries? → NO → Monolith (you'll get them wrong)
Do you have > 30 engineers? → NO → Monolith (operational overhead isn't worth it)
Do components have DIFFERENT scaling needs? → NO → Monolith (scale all together)
Do teams need to deploy independently? → NO → Monolith (coordinate deploys)
Do you need polyglot technology per component? → NO → Monolith (one stack is fine)

ALL YES → Consider microservices. Start with 2-3 services, not 50.
ANY NO → Monolith or modular monolith first.

Rule: Default to monolith. Extract services ONLY when you feel specific pain.
```

---
---

# TRADE-OFF 8: Replication vs Sharding

---

## 1. Concept Overview

**What it is:** Two different strategies for scaling a database beyond a single node. Replication creates copies (replicas) of the entire database. Sharding splits the dataset across multiple nodes, each holding a portion.

**Why it exists:** A single database node has limits — disk size, memory, CPU, network, write throughput. When you exceed those limits, you must distribute the data. The question is how.

**Why interviewers ask:** Every large-scale system hits database scaling limits. Understanding when to replicate vs shard — and the tradeoffs of each — is fundamental to designing scalable data layers.

---

## 2. Visual Diagram

```
REPLICATION (same data, multiple copies):
                Primary (writes + reads)
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
      Replica 1  Replica 2  Replica 3
  (all have ALL data; reads distributed across them)

SHARDING (different data, different nodes):
   User IDs 1-10M   User IDs 10M-20M   User IDs 20M-30M
   ┌──────────┐     ┌──────────┐        ┌──────────┐
   │  Shard 1  │     │  Shard 2  │        │  Shard 3  │
   └──────────┘     └──────────┘        └──────────┘
   (each shard has 1/3 of data; combined = full dataset)

COMBINED (each shard is replicated):
   Shard 1 Primary + Shard 1 Replica
   Shard 2 Primary + Shard 2 Replica
   Shard 3 Primary + Shard 3 Replica
```

---

## 3. Option A — Replication

**Definition:** Create copies of the entire database. Primary handles writes, replicas handle reads. Increases read capacity and provides fault tolerance.

**Advantages:**
- Increases read throughput — spread reads across replicas
- Fault tolerance — primary fails → promote a replica
- Geographic distribution — replicas close to users globally
- Relatively simple — well-understood pattern, built into most databases
- Full dataset on every node — complex queries and JOINs still work

**Disadvantages:**
- Doesn't solve write bottleneck — all writes still go to one primary
- Replication lag — replicas may be milliseconds to seconds behind primary
- Storage cost — full dataset replicated N times
- Doesn't increase storage capacity — each replica has the full dataset
- Eventual consistency on reads from replicas (stale reads possible)

**Cost:** Storage cost = dataset_size × replica_count. Low operational overhead.

---

## 4. Option B — Sharding

**Definition:** Partition the dataset across multiple independent database instances. Each shard holds a disjoint subset of the data. All shards combined = complete dataset.

**Advantages:**
- Increases write throughput — writes distributed across shards
- Increases storage capacity — each shard holds partial data, combined = more
- Scales horizontally — add shards to handle more data/traffic
- Reduced individual node load — each shard handles 1/N of the data

**Disadvantages:**
- Cross-shard JOINs are expensive or impossible — data is on different machines
- Cross-shard transactions are complex (2PC or Saga) — no ACID across shards
- Hotspot problem — if one user is very active, their shard gets hammered
- Resharding is painful — rebalancing when adding new shards is expensive
- Operational complexity — routing layer needed, shard key choice is critical

**Cost:** Lower per-node cost (each node stores less data) but higher operational cost.

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ Replication                   │ Sharding                          │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Read scaling        │ Excellent (spread reads)      │ Moderate (each shard handles own) │
│ Write scaling       │ None (primary bottleneck)     │ Excellent (parallel writes)       │
│ Storage scaling     │ None (full copy on each)      │ Excellent (1/N per shard)         │
│ Query flexibility   │ Full (JOINs across all data)  │ Limited (no cross-shard JOINs)    │
│ Transactions        │ ACID on primary               │ Complex (cross-shard = Saga)      │
│ Complexity          │ Low (built-in to most DBs)    │ High (routing, resharding)        │
│ Fault tolerance     │ High (replicas = standby)     │ Partial (one shard fails = gap)  │
│ Latency             │ Low read latency (replicas)   │ Depends on shard key design       │
│ Consistency         │ Primary strong, replicas stale│ Per-shard strong; cross-shard eventual│
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose Replication

- **Read-heavy workloads:** Product catalog, article content, user profiles — 95% reads, 5% writes
- **Fault tolerance primary goal:** Primary fails → replica promoted; minimal downtime
- **Geographic distribution:** Replicate across regions; users read from nearest replica
- **Dataset fits on one machine:** Data < 10TB, writes < 50K/sec — replication alone sufficient
- **Complex queries needed:** JOINs, analytics queries require full dataset on one node

---

## 7. When To Choose Sharding

- **Write-heavy workloads:** Social feeds, financial transactions, IoT events — many simultaneous writers
- **Dataset too large for one node:** 100TB+; no single machine holds all the data
- **Strong write throughput requirement:** > 100K writes/sec sustained
- **Data naturally partitionable:** User data by user_id; geographic data by region
- **Companies:** Cassandra (consistent hashing sharding), MongoDB (auto-sharding), Vitess (MySQL sharding)

---

## 9. Strong Interview Answers

**Q: "How do you choose a shard key? What makes a bad shard key?"**

> "A shard key determines how data is distributed. A good shard key has three properties. First, high cardinality — enough unique values to distribute across many shards. Second, even distribution — no hot spots where one shard gets dramatically more traffic than others. Third, query locality — your most common queries should typically hit ONE shard, not all of them. The worst shard keys: timestamps or sequential IDs (every new write goes to the 'last' shard — a write hotspot). User creation date (same problem). The best shard keys: user_id (high cardinality, evenly distributed, most queries are per-user). A hash of the entity ID (guarantees even distribution). For geographic systems, geographic region. The classic mistake: sharding by a low-cardinality field like 'country' — only 200 countries, and US alone might get 40% of your traffic."

**Q: "Your read replicas are deployed. Reads are fast. But writes are now maxed at 10,000/sec. Replicas don't help. What next?"**

> "Read replicas solve read bottlenecks, not write bottlenecks — all writes still go to the single primary. At 10K writes/sec sustained, you're hitting the primary's CPU and I/O limits. Short term: vertical scale the primary — bigger machine, faster SSD, more RAM for buffer pool. That might get you to 30-50K writes/sec. Beyond that, you have three options. One: application-level write optimization — batch writes, reduce write frequency, eliminate unnecessary writes. Two: CQRS with a write-optimized store — direct writes to an append-only event log (Kafka), then materialize to the read DB asynchronously. Three: shard the database — partition by user_id using consistent hashing across 4 or 8 primary shards, each handling 1/N of the write volume. Accept the cross-shard complexity. I'd try options 1 and 2 before sharding, because sharding is a one-way door that makes many things harder."

---

## 10. Common Mistakes

- **Replicating to solve write bottlenecks** — replicas don't help writes
- **Choosing a monotonically increasing shard key** (timestamp, auto-increment ID) — all new writes go to one shard
- **Not planning for resharding** — adding a shard to an existing sharded system is painful; plan for it early
- **Forgetting to replicate shards** — a sharded but non-replicated system loses 1/N of data if any shard fails
- **Assuming cross-shard queries are free** — scatter-gather queries are much slower than single-shard queries

---

## 12. Real System Design Examples

| System | Strategy | Details |
|--------|----------|---------|
| WhatsApp | Sharding (by conversation_id) | Each conversation's messages co-located |
| Instagram | Sharding (PostgreSQL by user_id) + Replication | Custom PostgreSQL sharding with Vitess-like approach |
| Uber | Sharding (by geographic region) + Replication per shard | Geo-based sharding for proximity queries |
| YouTube | Replication (read-heavy) + Sharding for video metadata | Video metadata sharded by video_id |
| Dropbox | Replication (metadata DB) + Object Storage sharding | File metadata replicated; file data in distributed object store |
| Netflix | Cassandra (consistent hash sharding + 3x replication) | Each shard replicated 3 ways for fault tolerance |
| Ticketmaster | Replication (mostly) + Sharding by event_id for scale | Event inventory sharded; replication for read speed |
| Twitter | Custom sharding (Manhattan) + Replication | User timelines sharded by user_id |

---

## 13. Scaling Journey

```
Small (< 10GB, < 5K writes/sec):
→ Single node. No replication. No sharding.

Medium (10-100GB, < 20K writes/sec):
→ Add 2-3 read replicas. Reads distributed. Writes still on primary.
→ Vertical scale the primary as needed.

Large (100GB-10TB, 20K-100K writes/sec):
→ Replicas for reads (10+ replicas with load balancing).
→ Shard writes across 4-16 primary shards.
→ Each shard has its own replica set.

Hyperscale (10TB+, 100K+ writes/sec):
→ Hundreds of shards. Automated rebalancing (Cassandra, Vitess).
→ Each shard: primary + 2-3 replicas.
→ Consider multi-region replication of entire shard groups.
```

---

## 14. Decision Framework

```
Problem: What is your bottleneck?

Read bottleneck (reads are slow, writes are fine)?
→ REPLICATION: Add read replicas. Distribute read queries.

Write bottleneck (writes are slow, reads are fine)?
→ SHARDING: Split write load across multiple primaries.

Storage bottleneck (data doesn't fit on one machine)?
→ SHARDING: Each shard holds a portion of data.

Both read and write bottleneck?
→ SHARDING + REPLICATION: Shard for writes; replicate each shard for reads.

Need complex cross-entity queries?
→ Avoid sharding if possible. Use replication + caching.
→ If you must shard, accept scatter-gather for cross-shard queries.
```

---
---

# TRADE-OFF 9: WebSocket vs Polling

---

## 1. Concept Overview

**What it is:** Two approaches for getting real-time or near-real-time updates from a server to a client. Polling: client repeatedly asks "any new data?" on a schedule. WebSocket: a persistent bidirectional connection where server can push updates the instant they're available.

**Why it exists:** HTTP is request-response — the client asks, the server responds. For real-time updates (chat, live scores, notifications), clients traditionally had to constantly ask ("poll"). WebSockets inverted this — the server pushes data the instant it's available, over a persistent connection.

**Why interviewers ask:** It's central to any real-time system design. Choosing polling when you need real-time (or WebSockets when polling suffices) shows poor judgment. The trade-off between connection count, latency, server load, and simplicity must be understood.

---

## 2. Visual Diagram

```
SHORT POLLING (every 5 seconds):
Client ─[GET /messages?after=t1]─▶ Server ─▶ "nothing new" (empty response)
Client ─[GET /messages?after=t1]─▶ Server ─▶ "nothing new"
Client ─[GET /messages?after=t1]─▶ Server ─▶ "nothing new"
Client ─[GET /messages?after=t1]─▶ Server ─▶ "new message!" (3 seconds late)
Problem: Wastes bandwidth; 3-second latency; server hit N clients × polls/sec

LONG POLLING:
Client ─[GET /messages?after=t1]─▶ Server ─── holds connection open ───▶ "new message!"
Client immediately re-opens: ─[GET /messages?after=t2]─▶ Server ... (holds again)
Better: lower latency, fewer empty responses. Still creates new connections.

WEBSOCKET:
Client ──── WS handshake ────▶ Server
Client ◀──── push: new message ─── Server (instant, no polling)
Client ◀──── push: typing indicator Server (instant)
Client ──── user is typing ───▶ Server (bidirectional!)
One connection, two-way, zero latency, no repeated requests.
```

---

## 3. Option A — Polling (Short / Long)

**Definition:** Client initiates repeated HTTP requests to check for new data. Short polling: fixed interval (every N seconds). Long polling: server holds the connection open until data is available, then responds immediately.

**Advantages:**
- Simple to implement — just HTTP; no special server infrastructure
- Works everywhere — firewalls, proxies, CDNs all handle HTTP easily
- Stateless server — no persistent connections to manage
- Easy to scale horizontally — any server can handle any poll request
- Works over HTTP/1.1 and HTTP/2

**Disadvantages:**
- Latency (short poll): data available the instant after a poll → must wait for next poll cycle
- Server load: N clients × polls/sec = many requests, even when no new data
- Wasted bandwidth: most poll responses are empty ("nothing new")
- Long polling: server resources held during long waits; still one-way

**Cost:** Scales linearly with client count × poll frequency. Can be expensive at large scale.

---

## 4. Option B — WebSocket

**Definition:** Full-duplex, persistent TCP connection between client and server. After an initial HTTP upgrade handshake, either side can send data at any time without the overhead of HTTP request/response.

**Advantages:**
- True real-time: server pushes data the instant it's available — sub-10ms latency
- Bidirectional: both client and server can send at any time (chat, live collaboration)
- Efficient: one connection, no repeated HTTP headers, no empty responses
- Low server load for active data: push only when data exists

**Disadvantages:**
- Persistent connections: server must maintain connection state for each connected client
- Scale complexity: horizontal scaling requires connection affinity or a pub/sub broker (Redis pub/sub) to fan-out messages to the right server holding each client's connection
- Firewall/proxy issues: older infrastructure may not support WebSocket upgrades
- Reconnection logic needed: client must handle disconnections gracefully
- Resource usage: 1M concurrent connections = significant server memory

**Cost:** Higher per-server resource usage. Requires connection routing infrastructure for horizontal scale.

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ Polling                        │ WebSocket                         │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Latency             │ Up to poll interval delay      │ Sub-millisecond push              │
│ Server load         │ High (N clients × poll rate)  │ Low (push on event only)          │
│ Bidirectional       │ No (request-response)          │ Yes (both sides send anytime)     │
│ Implementation      │ Simple (HTTP client)           │ Moderate (WS client + server)     │
│ Scaling             │ Stateless, easy               │ Stateful, needs routing layer      │
│ Bandwidth           │ Wasteful (empty responses)    │ Efficient (only real data)        │
│ Firewall compat.    │ Universal                     │ Sometimes blocked                  │
│ Connection state    │ None (stateless)              │ Per-client state maintained        │
│ Battery usage       │ Higher (constant wakeup)      │ Lower (push when needed)           │
│ Infrastructure      │ Standard HTTP                 │ WS server, pub/sub for scale      │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose Polling

- **Low-frequency updates:** Check for new emails every 5 minutes — polling is fine
- **Simple infrastructure:** REST-only services; no WebSocket support in the client environment
- **Batch-style updates:** Dashboard that refreshes every 30 seconds
- **Behind strict firewalls/proxies:** Some enterprise environments block WebSocket upgrades
- **Serverless backends:** AWS Lambda doesn't maintain persistent connections — polling works; WebSocket via API Gateway is complex
- **Server-Sent Events (SSE)** as a middle ground: one-way push over HTTP/2 — simpler than WebSocket for server-only push

---

## 7. When To Choose WebSocket

- **Chat applications:** WhatsApp, Slack — message must arrive in under 100ms
- **Live collaboration:** Google Docs, Figma — cursor position, text changes must be real-time
- **Live sports scores / stock prices:** Latency of even 1 second is noticeable
- **Multiplayer games:** Player position updates dozens of times per second
- **Live notification systems:** "User liked your post" should appear within a second
- **IoT dashboards:** Sensor readings streaming in real-time to a web dashboard

---

## 9. Strong Interview Answers

**Q: "1 million users connected via WebSocket to your chat service. How do you scale?"**

> "The challenge with WebSocket at scale is that each connection is stateful — it's held open on a specific server. If you simply add more servers and load-balance, User A might be on Server 1, User B on Server 2 — when B sends A a message, Server 2 doesn't have A's connection. The solution: use a publish-subscribe broker (Redis pub/sub, or Kafka) as the message fan-out layer. When Server 2 receives B's message to A, it publishes it to a Redis channel for user A. Server 1 is subscribed to all channels for users connected to it. Server 1 receives the Redis message and pushes it over A's WebSocket. Each server now only manages its own connections; cross-server routing goes through Redis. Scaling: just add WebSocket servers behind a load balancer with sticky sessions (connection affinity) — the same user always hits the same server. Redis pub/sub fans out messages. With 1M connections and good memory management (each connection ~40KB RAM), you need about 40GB RAM for connections alone across a cluster — manageable with 10-20 nodes."

---

## 10. Common Mistakes

- **Using short polling for chat** — 2-second polling latency, thousands of empty responses/sec, terrible UX
- **Not handling WebSocket reconnection** — mobile apps constantly switch networks; must auto-reconnect
- **No heartbeat/ping-pong** — idle WebSocket connections dropped by firewalls/NATs; implement keepalive
- **Forgetting sticky sessions** — load balancer sends reconnect to different server; connection-level state lost
- **Building WebSocket for infrequent updates** — checking for new emails once per hour doesn't need WebSocket


---

## 12. Real System Design Examples

| System | Approach | Details |
|--------|---------|---------|
| WhatsApp | WebSocket (persistent connection) | Connection to WhatsApp servers; Redis pub/sub for cross-server delivery |
| Instagram | Polling (activity feed) + Push notifications | Feed: polling or SSE; Notifications: APNs/FCM push |
| Uber | WebSocket (driver location to rider) | Driver app pushes location; dispatched to rider's WebSocket |
| YouTube | Polling (comment load) + WebSocket (live chat) | Regular video: polling; Live streams: WebSocket for chat |
| Dropbox | Long polling (file change detection) | Client long-polls for change notifications; then syncs |
| Netflix | Polling (play state, queue) | Not real-time sensitive; 30-second refresh is fine |
| Ticketmaster | Polling (seat map) | Seat availability refreshed every few seconds via polling |
| Twitter/X | SSE / WebSocket (live timeline) | Streaming API delivers new tweets via persistent connection |

---

## 13. Scaling Journey

```
Small Scale (< 10K users):
→ Short polling works fine. Simple to implement. Server load manageable.

Medium Scale (10K-100K users):
→ Long polling to reduce empty responses. Reduce poll frequency.
→ Or: introduce WebSocket for high-engagement features (chat, notifications).

Large Scale (100K-1M users):
→ WebSocket with Redis pub/sub for cross-server message routing.
→ Sticky sessions at load balancer for connection affinity.
→ Separate WebSocket gateway tier from application logic.

Hyperscale (> 1M concurrent connections):
→ Dedicated connection management service (like Facebook's "Iris" or WhatsApp's connection server).
→ Connection load balanced across hundreds of nodes.
→ Persistent connection state replicated for failover.
→ Some companies (WhatsApp, Discord) use Erlang/Elixir specifically for connection management (designed for millions of lightweight concurrent connections).
```

---

## 14. Decision Framework

```
Is update latency critical (< 500ms)? YES → WebSocket
Is the connection bidirectional (client AND server send)? YES → WebSocket
Is the update frequency very high (> 1/sec)? YES → WebSocket (polling wastes bandwidth)
Is the infrastructure WebSocket-friendly? NO → Long polling or SSE

Is the update frequency low (< 1/min)? YES → Polling (WebSocket overhead isn't worth it)
Is the client behind strict firewalls? YES → Polling or SSE (over HTTP)
Is the backend stateless/serverless? YES → Polling (WebSocket requires persistent process)

SSE (Server-Sent Events): good middle ground for one-way server push
over HTTP/2 when bidirectional isn't needed. Simpler than WebSocket.
```

---
---

# TRADE-OFF 10: REST vs GraphQL

---

## 1. Concept Overview

**What it is:** Two API paradigms. REST: multiple endpoints, each returning a fixed data structure, designed around resources. GraphQL: a single endpoint where the client specifies EXACTLY what data it needs in the query — no more, no less.

**Why it exists:** REST's fixed endpoints cause over-fetching (getting more data than needed) and under-fetching (needing multiple requests to assemble a view). GraphQL solves this by letting clients specify their exact data requirements.

**Why interviewers ask:** API design is central to product quality. Understanding when GraphQL's flexibility outweighs its complexity, and when REST's simplicity beats GraphQL's power, shows product and engineering maturity.

---

## 2. Visual Diagram

```
REST (over-fetching example):
GET /users/123  → returns: {id, name, email, address, phone, createdAt, ...} (30 fields)
UI only needs: name, avatar_url (2 fields)  ← 28 fields wasted bandwidth

REST (under-fetching / N+1 example):
GET /posts             → [{id:1}, {id:2}, {id:3}]  (3 requests needed next)
GET /posts/1/comments  → [...]
GET /posts/2/comments  → [...]
GET /posts/3/comments  → [...]
4 total requests to render a page!

GraphQL (exact data):
POST /graphql  Query:
{
  user(id: "123") {
    name
    avatarUrl
    posts(first: 3) {
      title
      comments { text author { name } }
    }
  }
}
→ ONE request, exactly the fields needed, nested relationships resolved.
```

---

## 3. Option A — REST

**Definition:** Representational State Transfer. Resources exposed via URL endpoints. HTTP methods (GET/POST/PUT/DELETE) represent CRUD operations. Fixed response structure per endpoint.

**Advantages:**
- Universal — every HTTP client, framework, and tool understands REST
- Cacheable — GET requests are natively cacheable by CDN/browsers (URL-based cache key)
- Simple — easy to understand, implement, and debug with `curl` or browser
- Stateless — each request is self-contained
- Good tooling — OpenAPI/Swagger, Postman, native browser caching
- Well-understood performance — optimized queries per endpoint

**Disadvantages:**
- Over-fetching — endpoints return more data than the client needs
- Under-fetching — building a complex page requires multiple API calls
- Endpoint proliferation — need custom endpoints for every client's unique data needs
- Versioning — adding v2 endpoints for new client requirements is messy

**Cost:** Low — no special infrastructure or tooling needed.

---

## 4. Option B — GraphQL

**Definition:** A query language for APIs. Single endpoint (/graphql). Client specifies exactly what data it needs in the query. Server resolves the query against its schema.

**Advantages:**
- No over-fetching — client gets exactly what it asks for
- No under-fetching — nested relationships in one query
- Strongly typed schema — self-documenting; contract between frontend and backend
- Great for complex, relationship-heavy data (social graphs, e-commerce)
- Frontend teams can iterate without backend changes — just add fields to schema
- Introspection — clients can discover the schema at runtime

**Disadvantages:**
- Caching is hard — POST to single endpoint; URL-based caching doesn't apply
- Complex to implement — N+1 query problem (DataLoader needed to batch DB queries)
- Performance challenges — unbounded queries can cause expensive DB operations
- Learning curve — teams must learn GraphQL schema language, resolvers, etc.
- Not appropriate for all APIs — file uploads, streaming, binary data are awkward

**Cost:** Higher — infrastructure for schema, resolvers, DataLoader, query complexity limiting.

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ REST                           │ GraphQL                           │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Over-fetching       │ Common (fixed response shape) │ Eliminated (client specifies)    │
│ Under-fetching      │ Common (N+1 calls)            │ Eliminated (nested queries)      │
│ Caching             │ Excellent (native HTTP cache) │ Difficult (POST to one endpoint) │
│ Complexity          │ Low                           │ High (schema, resolvers, batching)│
│ Type safety         │ None by default               │ Strong (schema-enforced)          │
│ Flexibility         │ Low (fixed endpoints)         │ High (any valid query)            │
│ Performance         │ Predictable, optimizable      │ Risky (unbounded query depth)    │
│ File uploads        │ Native multipart              │ Awkward                           │
│ Learning curve      │ Minimal                       │ Significant                       │
│ Best clients        │ Mobile, public APIs           │ Complex web apps, multiple clients│
│ API versioning      │ Explicit (/v1, /v2)           │ Schema evolution, no versioning   │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose REST

- **Public APIs:** External developers need predictable, stable, cacheable endpoints
- **Mobile apps on limited bandwidth:** Predictable response sizes; optimized per endpoint
- **Simple CRUD services:** User registration, file upload, payment — simple operations
- **High-traffic cacheable resources:** Product pages, article content (CDN caches REST GET responses)
- **Microservices internal APIs:** Simple service-to-service communication
- **Teams without GraphQL experience:** Learning curve isn't worth it for simple APIs

---

## 7. When To Choose GraphQL

- **Multiple clients with different data needs:** Web, mobile, TV apps all need different subsets of the same data
- **Rapidly evolving frontend requirements:** Frontend can add fields without backend changes
- **Complex, interconnected data:** Social graph, e-commerce (products → variants → reviews → reviewers)
- **Developer experience priority:** Internal tools, developer portals where DX matters more than CDN caching
- **Companies:** GitHub API v4, Shopify Storefront API, Twitter API (internal), Facebook (invented it)

---

## 9. Strong Interview Answers

**Q: "What is the N+1 problem in GraphQL and how do you solve it?"**

> "The N+1 problem: suppose you query for 10 posts, and each post has an author. The naïve GraphQL resolver fetches posts (1 DB query), then for each post, calls the author resolver (1 DB query per post) — that's 1 + 10 = 11 DB queries to answer one GraphQL request. With 100 posts, it's 101 queries. DataLoader solves this by batching. Instead of immediately executing each author lookup, DataLoader collects all author_id values during a tick of the event loop, then issues a SINGLE batch query: `SELECT * FROM users WHERE id IN (1, 2, 3, ..., 10)`. Results are mapped back to individual resolvers. One query instead of N+1. DataLoader is an absolute requirement for production GraphQL — without it, your database gets hammered by every complex query."

---

## 10. Common Mistakes

- **Choosing GraphQL for a simple API** — adds enormous complexity where REST would be trivial
- **Not implementing query depth/complexity limiting** — allows clients to execute O(N^depth) database queries
- **Forgetting DataLoader** — results in catastrophic N+1 database queries under any real load
- **Treating GraphQL as a replacement for all REST endpoints** — file uploads, webhooks, streaming are REST-native
- **Ignoring caching** — public APIs built with GraphQL lose CDN caching benefits; must implement custom Apollo-based caching


---

## 12. Real System Design Examples

| System | REST or GraphQL | Reasoning |
|--------|----------------|-----------|
| WhatsApp | REST (internal) | Simple endpoints; predictable responses; no client flexibility needed |
| Instagram | REST (public API) | Stable endpoints; cacheable; simple mobile needs |
| Uber | REST (driver/rider APIs) | Simple, high-frequency endpoints; caching needed |
| YouTube | REST | Public API, predictable, CDN-cacheable |
| Dropbox | REST | Simple file operations; external developers |
| Netflix | REST + BFF pattern | Different BFFs per client type instead of GraphQL |
| Shopify | GraphQL (Storefront API) | Merchants need flexible product/variant/inventory queries |
| Twitter | REST (v1) + GraphQL (internal) | Public REST for stability; GraphQL internally for flexibility |
| GitHub | REST (v3) + GraphQL (v4) | Migrated to GraphQL to reduce API calls for complex queries |

---

## 13. Scaling Journey

```
Small Scale: REST. Simple, cacheable, everyone knows it. Ship fast.

Medium Scale: REST + BFF pattern (Backend for Frontend).
  Web BFF: aggregates N REST calls for web client's complex pages.
  Mobile BFF: aggregates differently for mobile.
  Better than GraphQL for most cases: predictable, cacheable.

Large Scale with many client types: Consider GraphQL.
  When different client teams need different data from same entities.
  When frontend teams are blocked waiting for new REST endpoints.
  When N+1 API calls from clients are a measurable performance problem.

Hyperscale with GraphQL: Apollo Federation.
  Each team owns a GraphQL subgraph (schema).
  Apollo Gateway federates them into one unified GraphQL API.
  Clients get one query, gateway routes to multiple backend services.
```

---

## 14. Decision Framework

```
Public API with external developers? → REST (stability, caching, universal support)
Simple CRUD with few clients? → REST (simplicity wins)
Many clients with different data needs? → Consider GraphQL
Complex nested data (social graph, catalog)? → GraphQL helps
Need HTTP caching (CDN, browser)? → REST
Need self-documenting schema? → GraphQL

When in doubt: REST + BFF pattern beats GraphQL for most applications.
GraphQL shines when you genuinely have multiple clients with very different
data needs that are changing rapidly and you have the team expertise.
```

---
---

# TRADE-OFF 11: Batch Processing vs Stream Processing

---

## 1. Concept Overview

**What it is:** Batch processing: collect data over a period (hours, days), then process it all at once. Stream processing: process each piece of data the instant it arrives — continuously, in real-time.

**Why it exists:** Some analyses need historical context (monthly revenue report). Others need immediate response (fraud detection). The tools and architectures are completely different — and mixing them up leads to wrong latency guarantees and unnecessary cost.

**Why interviewers ask:** Any data-intensive system requires this decision. Recommender systems, analytics pipelines, billing, fraud detection — each has different latency requirements and volume profiles that map to batch or stream.

---

## 2. Visual Diagram

```
BATCH PROCESSING:
Events accumulate    →  At midnight:    →  Report available
all day               process all         at 1am
[00000000000000000000000000000]
                              ↓ one run
                       [MapReduce / Spark Job]
                              ↓
                       [Report: ₹2.3M revenue today]

STREAM PROCESSING:
Event arrives → immediately processed → real-time output

[order ₹999] → [Fraud check] → [BLOCKED: unusual]  (< 100ms!)
[order ₹200] → [Fraud check] → [OK, proceed]       (< 100ms!)
[order ₹50K] → [Fraud check] → [BLOCKED: flag]     (< 100ms!)

LAMBDA ARCHITECTURE (both):
Raw events → [Batch Layer (Spark)] → accurate historical reports
           → [Speed Layer (Flink)] → approximate real-time views
           ↑
           merged in serving layer
```

---

## 3. Option A — Batch Processing

**Definition:** Data collected and stored, then processed in large runs at scheduled intervals (every hour, every night). Hadoop MapReduce, Apache Spark are the canonical tools.

**Advantages:**
- High throughput — can process petabytes with parallel compute
- Fault tolerant — retry failed batches; idempotent processing
- Correct and complete — processes full historical data, no missing records
- Resource efficient — run heavy computation at off-peak times
- Cheaper infrastructure — can use spot/preemptible instances
- Proven tools — Spark, Hadoop are mature and battle-tested

**Disadvantages:**
- High latency — results available only after batch completes (hours or day)
- Delay in insights — fraud detected 8 hours later is too late
- Resource spikes — batch jobs consume massive resources during execution
- Old data when complete — report represents yesterday, not right now
- Hard to iterate — reprocessing all historical data for a bug fix is slow

**Cost:** Low per-unit compute cost at scale; high infrastructure cost during job execution.

---

## 4. Option B — Stream Processing

**Definition:** Data processed event-by-event (or in micro-batches) as it arrives. Apache Kafka + Flink/Spark Streaming are canonical. Latency is milliseconds to seconds.

**Advantages:**
- Low latency — results available within milliseconds to seconds of event
- Real-time fraud detection, alerting, dashboards
- Continuous processing — no batch windows
- Can trigger immediate actions (block a payment, send a notification)
- Smaller, incremental resource usage — no spikes

**Disadvantages:**
- More complex — state management, exactly-once semantics, watermarking
- Harder to reprocess historical data — "what was the count 3 hours ago?"
- Approximate results — streaming windows may miss late-arriving events
- Higher infrastructure cost at scale — Kafka cluster always running
- Correctness challenges — out-of-order events, duplicate events

**Cost:** Higher operational cost (always-on infrastructure) but lower latency.

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ Batch Processing               │ Stream Processing                  │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Latency             │ Hours to days                 │ Milliseconds to seconds           │
│ Throughput          │ Very high (petabytes)         │ High but continuous               │
│ Correctness         │ High (complete data)          │ Approximate (windowed, late data) │
│ Complexity          │ Lower (batch jobs are simpler)│ Higher (state, ordering, at-least-once)│
│ Use for real-time?  │ No                            │ Yes                               │
│ Cost                │ Lower (batch spot instances)  │ Higher (always-on cluster)        │
│ Historical reprocess│ Natural (run job again)       │ Difficult (replay from Kafka)     │
│ Late data handling  │ Easy (just include in batch)  │ Hard (watermarks, out-of-order)   │
│ Tools               │ Spark, Hadoop, dbt            │ Flink, Kafka Streams, Spark Streaming│
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose Batch Processing

- **End-of-month billing:** Calculate billing totals — can wait until billing run
- **ML model training:** Train models on last month's data — batch overnight
- **ETL pipelines to data warehouse:** Nightly loads from operational DBs to BigQuery/Redshift
- **Large-scale analytics:** Revenue breakdowns, cohort analysis — computed daily
- **Recommendation model updates:** Collaborative filtering trained on batch data
- **Log aggregation for compliance:** Aggregate daily logs for audit reporting

---

## 7. When To Choose Stream Processing

- **Real-time fraud detection:** Block a payment within 200ms — batch is far too late
- **Live dashboards:** "Revenue in the last 5 minutes" — must process in real-time
- **Trending topics:** What's trending RIGHT NOW on Twitter — needs stream processing
- **Real-time alerting:** "Error rate spike in production" — must fire within 30 seconds
- **Live leaderboards:** Gaming leaderboard that updates as scores change
- **Event-driven microservices:** Order placed → inventory updated immediately


---

## 9. Strong Interview Answers

**Q: "What is the Lambda architecture and when would you use it?"**

> "Lambda architecture solves the problem of needing BOTH accurate historical data AND real-time current data. It has three layers. Batch layer: processes ALL historical data periodically (nightly Spark jobs); produces accurate but delayed views. Speed layer: processes the most recent data in real-time (Flink or Spark Streaming); produces approximate but fast current views. Serving layer: merges the two — for historical periods, use batch results (accurate); for recent data, use stream results (fast). The main criticism of Lambda is that you're maintaining two separate codebases for the same logic — batch and stream — which diverge over time and cause bugs. The Kappa architecture addresses this by eliminating the batch layer and processing everything through the stream layer, replaying historical data from Kafka when you need to reprocess. I'd choose Lambda when I need provably correct historical aggregates AND real-time views of recent data. I'd choose Kappa when the stream processing can handle the historical reprocessing volume and I want to maintain only one codebase."

---

## 10. Common Mistakes

- **Using batch processing for fraud detection** — by the time the batch runs, money is gone
- **Using streaming for historical analytics** — stream processing can't easily answer "what was the metric 6 months ago?"
- **Ignoring exactly-once semantics** — duplicate events in a payment stream cause double billing
- **Not handling late-arriving events** — events arriving 5 minutes late after the window closed are silently dropped
- **Underestimating stream processing complexity** — teams often switch to "simpler" batch for complex aggregations


---

## 12. Real System Design Examples

| System | Batch | Stream | Notes |
|--------|-------|--------|-------|
| WhatsApp | Message delivery stats (daily) | Message delivery confirmation (real-time) | Batch for analytics; stream for UX |
| Instagram | ML model training (nightly) | Story view counts (real-time) | Two separate pipelines |
| Uber | Earnings reports (weekly) | Surge pricing (real-time) | Fundamentally different latency needs |
| YouTube | Recommendation training (batch) | View count updates (stream) | Large-scale ML is batch |
| Dropbox | Storage usage reports (hourly) | Sync conflict detection (real-time) | Analytics batch; sync stream |
| Netflix | Recommendation models (batch) | "Continue Watching" updates (near-real-time) | Heavy ML is batch |
| Ticketmaster | Daily sales reports (batch) | Seat availability (stream/real-time) | Inventory must be real-time |
| Twitter | Trending topics (stream) | Follower count updates (batch) | Trending is real-time; counts OK to lag |

---

## 13. Scaling Journey

```
Small Scale:
→ Batch only. Nightly scripts that process yesterday's data. Simple.

Medium Scale:
→ Batch for analytics (daily Spark job on S3 data).
→ Introduce streaming for 1-2 specific real-time use cases (fraud, alerts).
→ Use Kafka as the ingestion layer for both.

Large Scale:
→ Lambda architecture: batch layer (Spark) + speed layer (Flink) + serving DB.
→ Stream for: fraud, real-time dashboards, recommendations trigger.
→ Batch for: billing, monthly reports, ML model training.

Hyperscale:
→ Kappa architecture (everything through the stream layer).
→ Historical reprocessing via Kafka log compaction and replay.
→ Unified compute platform (Flink handles both real-time and batch in unified API).
```

---

## 14. Decision Framework

```
Ask: "How stale can the result be?"

< 1 second: Stream processing required
< 1 minute: Stream processing (micro-batch acceptable)
< 1 hour: Either (hourly batch OR windowed stream)
< 1 day: Batch processing is simpler and sufficient
> 1 day: Definitely batch

Ask: "Does the result require complete historical data?"
YES + low staleness tolerance → Lambda architecture (both)
YES + high staleness tolerance → Batch only
NO (just recent data) → Stream only

Ask: "Is the volume suitable for real-time processing?"
> 1M events/sec sustained → Careful stream processing design or batch
< 100K events/sec → Stream processing is manageable
```

---
---

# TRADE-OFF 12: Horizontal vs Vertical Scaling

---

## 1. Concept Overview

**What it is:** Two approaches to handling increased load. Vertical scaling: make the existing machine bigger (more CPU, more RAM, faster SSD). Horizontal scaling: add more machines, distribute the load across them.

**Why it exists:** Both approaches work — up to a point. Vertical scaling has a hard ceiling (the biggest machine in existence). Horizontal scaling is theoretically unlimited but requires the application to be distributed (stateless, data partitioned). Most systems start vertical and eventually go horizontal.

**Why interviewers ask:** Every system eventually faces this decision. Interviewers want to see that you understand the real-world constraints of each approach and know when to transition from one to the other.

---

## 2. Visual Diagram

```
VERTICAL SCALING (Scale Up):
Before: [2 CPU, 8GB RAM]  →  After: [32 CPU, 256GB RAM]
Simple! Same machine, just bigger. One unit of complexity.
But: there's a ceiling. And downtime during upgrade.

HORIZONTAL SCALING (Scale Out):
Before: [Server 1]
After:  [Server 1] [Server 2] [Server 3] [Server 4]
Requires: Load balancer in front; application must be stateless;
          data must be partitioned or replicated.
Theoretically unlimited. But complex.

STATEFUL vs STATELESS:
Stateful app (holds sessions in memory): [User A → Server 2, only Server 2 knows session]
Adding Server 3 doesn't help User A — can't route to Server 3.
Must use sticky sessions or external session store (Redis).

Stateless app: [User A → any server, session in Redis]
Add Server 3: User A's next request can go to Server 3.
Horizontal scaling works perfectly.
```

---

## 3. Option A — Vertical Scaling (Scale Up)

**Definition:** Increase the resources of a single machine — more CPU cores, more RAM, faster storage, better network card.

**Advantages:**
- Zero application changes — app doesn't know it's on a bigger machine
- No distributed systems complexity — still one node
- Lower latency — in-process operations faster on more powerful hardware
- No data partitioning — full dataset on one machine; complex queries work
- Simple operations — one server to monitor, deploy to, back up

**Disadvantages:**
- Hard ceiling — the biggest machine available (currently ~224 vCPUs, ~24TB RAM on AWS)
- Single point of failure — one machine means one point of failure
- Expensive at high end — doubling CPU doesn't double throughput; price/performance ratio degrades
- Downtime during upgrade — resizing often requires a restart
- Not elastic — can't downscale as easily; you're paying for peak capacity always

**Cost:** Exponentially increasing cost as you approach the top of available instance types.

---

## 4. Option B — Horizontal Scaling (Scale Out)

**Definition:** Add more machines to a cluster. Distribute requests across multiple nodes using a load balancer. Increase capacity by adding nodes; reduce by removing them.

**Advantages:**
- Theoretically unlimited scale — just add more nodes
- No single point of failure — one node failing means 1/N of capacity, not total failure
- Elastic — scale out for peak, scale in at off-peak, save cost
- Cost-effective — many commodity machines cheaper than one enormous machine
- No downtime — add nodes without restarting existing ones

**Disadvantages:**
- Application must be stateless — session state must be external (Redis, DB)
- Data must be partitioned or replicated — can't have full dataset on all nodes
- Distributed systems complexity — load balancing, service discovery, distributed tracing
- Network overhead — cross-node communication adds latency
- More operational complexity — N servers to monitor, patch, and maintain

**Cost:** Lower per-unit compute cost; higher operational cost for management.

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ Vertical Scaling               │ Horizontal Scaling                │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Scalability ceiling │ Hard ceiling (biggest machine) │ Theoretically unlimited           │
│ Application changes │ None                          │ Must be stateless                 │
│ Complexity          │ Low                           │ High (distributed systems)        │
│ SPOF risk           │ High (one machine)            │ Low (failure = 1/N capacity)      │
│ Elasticity          │ Low (resize = downtime)       │ High (add/remove nodes online)   │
│ Cost curve          │ Exponential at top end        │ Linear (add nodes as needed)      │
│ Latency             │ Lower (no network between)    │ Higher (cross-node calls)         │
│ Fault tolerance     │ Poor                          │ Good                              │
│ Operational effort  │ Low                           │ High                              │
│ Data distribution   │ Not needed (one node)         │ Required (partition or replicate) │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose Vertical Scaling

- **Stateful applications:** Legacy apps that can't easily be made stateless
- **Databases:** Single-node databases are simpler; vertical scaling before sharding
- **Low traffic systems:** Don't add distributed complexity before you need it
- **Rapid prototypes:** Get to market; scale later
- **Specific workloads:** High-memory, single-threaded workloads benefit from bigger machines
- **When close to a vertical ceiling:** Before committing to horizontal, try one more size up — it might buy 6-12 more months

---

## 7. When To Choose Horizontal Scaling

- **Web servers / API tier:** Almost always horizontal — stateless by design
- **Cache clusters:** Redis Cluster, Memcached sharding
- **Message queues:** Kafka partitions distributed across brokers
- **Compute-heavy, embarrassingly parallel workloads:** Video encoding, ML training across GPUs
- **When vertical scaling ceiling is reached:** Must go horizontal
- **High availability requirement:** Can't have a SPOF in the critical path

---

## 9. Strong Interview Answers

**Q: "Your e-commerce app is seeing 10x traffic. Walk me through your scaling strategy."**

> "I'd take a tiered approach. First, identify the bottleneck — is it the web tier, the database, or a specific service? For the web tier: it's likely stateless already; add more instances behind the load balancer. If sessions are in memory, move them to Redis so all instances can serve any user. This is the easiest win. For the database: read-heavy? Add read replicas. Write-heavy? Look at query optimization first — often a missing index causes 10x database load. Then consider caching hot data in Redis (product pages, user sessions). If still bottlenecked, consider vertical scale up first before sharding — sharding adds tremendous complexity. For specific bottleneck services: identify via APM which service is slow and target it specifically. The key principle: scale horizontally at the stateless web tier (easy), scale vertically at the stateful database tier first (simpler), and use caching to reduce database load before resorting to sharding."

---

## 10. Common Mistakes

- **Sharding before trying read replicas and caching** — much simpler solutions should come first
- **Not making web tier stateless before horizontal scaling** — sticky sessions are a workaround, not a solution
- **Scaling everything instead of identifying the bottleneck** — find the bottleneck; scale THAT specifically
- **Not auto-scaling** — manually managing fleet size during traffic spikes is an on-call nightmare
- **Ignoring the cost model** — horizontal scaling can be more expensive if not coupled with auto-scaling to downscale


---

## 12. Real System Design Examples

| System | Horizontal | Vertical | Notes |
|--------|-----------|---------|-------|
| WhatsApp | Both (stateless messaging tier horizontal; DB vertical then sharded) | Maximized single server efficiency before scaling out |
| Instagram | Horizontal web tier; vertical DB then sharded | Stateless Django app servers; PostgreSQL vertically scaled for years |
| Uber | Horizontal matching/dispatch; vertical DB initially | Microservices all horizontally scaled; DBs per-service |
| YouTube | Horizontal web serving; vertical video DB initially | Video CDN massively horizontal; metadata DB vertical long time |
| Dropbox | Horizontal API tier; vertical metadata DB (now Edgestore) | File storage separate from metadata scaling |
| Netflix | Massively horizontal (microservices on auto-scaled EC2) | Every microservice independently auto-scaled |
| Ticketmaster | Horizontal web; vertical + replica DB | Inventory DB critical — vertical first; read replicas for event day |
| URL Shortener | Horizontal web (stateless); vertical or Redis for URL store | URL lookups cacheable; horizontal web tier straightforward |

---

## 13. Scaling Journey

```
Phase 1 (< 1K req/sec): Single server. Vertical as needed.
  → One web server + one DB. Cheapest and simplest.

Phase 2 (1K-10K req/sec): Read replicas + caching + vertical.
  → Multiple web servers (stateless) + load balancer.
  → Read replicas for read-heavy DB queries.
  → Redis cache for hot data.

Phase 3 (10K-100K req/sec): Full horizontal web tier.
  → Auto-scaling group of web servers.
  → Sessions in Redis (not local memory).
  → DB: vertical to largest available instance + read replicas.

Phase 4 (> 100K req/sec): Horizontal data tier.
  → DB sharding by user_id or entity_id.
  → Kafka for async processing.
  → CDN for static assets.
  → Microservices for independently scalable components.
```

---

## 14. Decision Framework

```
Start here:
"Is my application stateless?" YES → horizontal scaling is straightforward
                               NO  → make it stateless first (externalize state)

"Where is the bottleneck?"
→ Web/API tier: almost certainly horizontal scaling (stateless, easy)
→ Database reads: read replicas first, then caching, then horizontal
→ Database writes: vertical first, then sharding (last resort)
→ Compute (ML, video encoding): horizontal (embarrassingly parallel)

"What's the budget?"
→ Tight: vertical scaling is cheaper to operate (fewer machines)
→ Flexible: horizontal + auto-scaling optimizes cost at scale

Rule: Horizontal for stateless tiers, Vertical first for stateful tiers,
      then horizontal (sharding) only when vertical ceiling is hit.
```

---

**END OF PART 2 (Trade-offs 7-12)**
*Continues in Part 3: Memory vs Latency, Throughput vs Latency, Latency vs Accuracy, CDN vs No CDN, Kafka vs RabbitMQ, Normalization vs Denormalization + Relationships between trade-offs*

---
---

# TRADE-OFF 13: Memory vs Latency

---

## 1. Concept Overview

**What it is:** Trading memory (RAM) to achieve lower latency — keeping data in memory is orders of magnitude faster than reading from disk. Conversely, reducing memory usage typically means accepting higher latency from disk or network reads.

**Why it exists:** The memory hierarchy is fundamental to computing: registers (< 1ns) → L1/L2 cache (1-10ns) → RAM (100ns) → SSD (100µs) → HDD (10ms) → Network (1-100ms). Higher up the hierarchy = faster but more expensive per GB. Every caching and data storage decision is fundamentally a memory-latency tradeoff.

**Why interviewers ask:** System design is often about managing data locality. When you add Redis in front of a database, you're trading memory for latency. When you pre-compute aggregations, you're trading memory (and write cost) for read latency. Senior engineers must quantify these tradeoffs, not just intuit them.

---

## 2. Visual Diagram

```
THE MEMORY HIERARCHY:
  CPU Registers:  ~0.3ns  |  bytes
  L1 Cache:       ~1ns    |  32-64 KB
  L2 Cache:       ~4ns    |  256 KB-1MB
  L3 Cache:       ~10ns   |  4-32 MB
  RAM:            ~100ns  |  GBs (Redis lives here)
  NVMe SSD:       ~100µs  |  TBs (RocksDB, local files)
  Network (LAN):  ~0.5ms  |  another machine's RAM (Redis over network)
  Network (WAN):  ~150ms  |  remote database

THE TRADEOFF:
More Memory → Data stays higher in the hierarchy → Lower Latency
Less Memory → Data moves lower → Higher Latency → Lower Cost
```

---

## 3. Option A — More Memory (Optimize for Latency)

**Definition:** Keep as much data in RAM as possible — in-process caches, Redis, large database buffer pools — to minimize disk and network access.

**Advantages:**
- Dramatically lower latency (100ns RAM vs 10ms disk = 100,000x)
- Higher throughput (memory operations per second >> disk operations)
- Absorbs read traffic without touching disk
- Predictable latency (no disk seek jitter)

**Disadvantages:**
- RAM is expensive (~10-20x per GB vs SSD)
- Volatile — data lost on process restart (must persist to disk eventually)
- Limited capacity — you can't fit the full dataset in RAM at hyperscale
- Hot/cold data problem — you must decide what stays in memory

**Cost:** High — RAM is significantly more expensive than SSD per GB. A 1TB RAM machine costs far more than a 1TB SSD machine.

---

## 4. Option B — Less Memory (Optimize for Cost)

**Definition:** Accept higher latency from disk reads. Use SSDs (NVMe) as the primary storage. Only keep the hot working set in memory.

**Advantages:**
- Much cheaper per GB — 10-20x cheaper than RAM
- Larger dataset fits — 10TB SSD vs 1TB RAM (same machine)
- Data durable by default — disk survives restarts

**Disadvantages:**
- Higher latency — SSD reads ~100µs vs RAM ~100ns
- Lower throughput — disk IOPS limit vs memory throughput
- Latency variability — disk access patterns affect latency consistency
- More expensive at extreme IOPS — high-performance SSD still cheaper than RAM but must right-size

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ More Memory                    │ Less Memory (Disk-based)          │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Latency             │ 0.1-1ms (in-process/Redis)   │ 1-100ms (SSD/disk/network)        │
│ Cost per GB         │ ~$5-10/GB (RAM)               │ ~$0.10-0.50/GB (SSD)             │
│ Capacity            │ Limited (TBs per machine)     │ High (100s of TBs per machine)    │
│ Durability          │ Volatile (need to persist)    │ Durable by default                │
│ Throughput          │ Very high                     │ Limited by disk IOPS              │
│ Operational effort  │ Must manage eviction/TTL      │ Simpler (just disk)               │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When to Use More Memory

- **Session data:** Must respond in < 10ms; Redis is essential
- **Rate limiting counters:** Atomic increment needed at 100K+ ops/sec
- **Hot database rows:** MySQL buffer pool caches hot pages in RAM — tune this aggressively
- **Leaderboards/rankings:** Real-time sorted sets — Redis ZSets
- **ML model inference:** Model weights in RAM for sub-100ms inference latency
- **Time-series data (recent):** Last 1 hour of metrics in memory for real-time dashboards

---

## 7. When to Accept Higher Latency (Disk/Network)

- **Cold data (historical):** 6-month-old logs don't need sub-millisecond access
- **User-uploaded files:** Photos, videos stored on SSD/object storage; 100ms reads acceptable
- **Analytics queries on large datasets:** Batch queries on TBs; latency < accuracy
- **Low-traffic systems:** Can't justify cost of RAM for a startup with 100 users

---

## 9. Strong Interview Answers

**Q: "How do you decide the size of your Redis cache?"**

> "Start by identifying your 'working set' — the data actually accessed in a given time window. Pareto principle typically applies: 80% of reads hit 20% of the data. If your dataset is 100GB but 80% of reads hit only 20GB of 'hot' data, a 20GB Redis cluster would achieve ~80% cache hit rate, which is often sufficient. Calculate: cost of a cache miss (DB read latency × miss rate × QPS) vs cost of a larger Redis cluster. If 10% of reads are cache misses at 50ms each, and you have 10,000 QPS, that's 1,000 DB reads/sec × 50ms = 50 seconds of aggregate wait time per second — your DB is overloaded. Doubling the Redis size to hit 95% cache might cost $200/month in Redis but save $2,000/month in DB compute. Always monitor hit rate and adjust. Start with enough to cache the hot 20%; add more if hit rate is below 90%."

---

## 10. Common Mistakes

- **Not setting maxmemory on Redis** — Redis will use all available RAM and be killed by the OS OOM killer
- **Caching everything** — low-value cold data consumes memory better used for hot data
- **Not monitoring cache hit rate** — flying blind; you don't know if your cache is actually helping
- **Over-provisioning memory** — paying for RAM that holds cold data never accessed

---

## 11. Real System Design Examples

| System | Memory Strategy | Why |
|--------|----------------|-----|
| WhatsApp | Active sessions in RAM | Message delivery must be < 100ms |
| Instagram | Post metadata in Redis | 95% hit rate achievable on recent content |
| Uber | Geospatial index in RAM | Driver matching needs < 100ms latency |
| YouTube | CDN edge memory for popular videos | Video should start in < 1 second |
| Netflix | Model inference in GPU/CPU RAM | Recommendations generated in < 200ms |
| URL Shortener | Entire URL mapping in Redis | Read-only hot dataset; perfect cache fit |

---

## 12. Decision Framework

```
Latency requirement < 10ms? → Must use memory (RAM/Redis)
Latency requirement < 100ms? → Memory preferred; high-performance SSD acceptable
Latency requirement < 1s? → SSD is fine; memory for hot subset
Latency requirement > 1s? → Disk/object storage acceptable; memory not required

Cost vs Benefit:
  (cache_miss_rate × miss_latency × QPS × cost_per_ms) > monthly_memory_cost?
  YES → Add more memory/cache
  NO  → Accept disk reads; memory not cost-justified
```

---
---

# TRADE-OFF 14: Throughput vs Latency

---

## 1. Concept Overview

**What it is:** Throughput measures how much work a system completes per unit of time (requests/second, bytes/second). Latency measures how long a single operation takes (milliseconds per request). Optimizing for one often degrades the other.

**Why it exists:** A system can serve 1,000 requests/second with 100ms average latency. To increase throughput to 10,000 req/sec, you add batching — but batching increases latency (must wait for a batch to fill). Conversely, minimizing latency (process each request immediately) limits the throughput optimizations (batching, caching) you can use.

**Why interviewers ask:** Systems optimized for throughput (batch jobs, data pipelines) and systems optimized for latency (user-facing APIs) have completely different architectures. Confusing the two leads to wrong design choices.

---

## 2. Visual Diagram

```
THROUGHPUT-OPTIMIZED (Batching):
Requests arrive: [R1] [R2] [R3] ... [R100]  ← buffer 100 requests
Then: [Process batch of 100 at once]  → 100 responses
Latency: R1 waits 99ms for batch to fill, then 10ms to process = 109ms for R1
Throughput: 100 requests / 0.109 seconds = ~917 req/sec (HIGH!)

LATENCY-OPTIMIZED (Immediate processing):
Request arrives: [R1] → process immediately → respond (10ms)
                [R2] → process immediately → respond (10ms)
Latency: 10ms for every request (LOW!)
Throughput: 1 server thread, 1 req at a time = 100 req/sec (LOWER)

The tension: batching increases throughput but adds queuing latency.
             parallelism can increase throughput without adding latency (at a cost).
```

---

## 3. Option A — Optimize for Throughput

**Definition:** Maximize the amount of work done per unit of time. Techniques: batching, buffering, pipelining, compression, vectorization.

**Advantages:**
- More efficient resource utilization — amortize overhead across many operations
- Lower per-unit cost — batch processing is cheaper at scale
- Better for bulk operations — file exports, ML training, data migrations

**Disadvantages:**
- Higher latency per individual operation (must wait for batch)
- Poor user experience for interactive workloads
- Not suitable for real-time systems

**Best for:** ETL pipelines, ML training, bulk writes, analytics queries, video encoding.

---

## 4. Option B — Optimize for Latency

**Definition:** Minimize the time to complete a single operation. Techniques: pre-computation, caching, connection pooling, co-location of compute and data, non-blocking I/O.

**Advantages:**
- Fast user experience — critical for interactive applications
- Real-time responsiveness — fraud detection, payment auth, API responses
- P99 latency control — essential for SLA-bound services

**Disadvantages:**
- Less efficient at scale — small batches = higher overhead per operation
- More expensive — fewer optimizations means more compute per result
- Harder to sustain under high load — individual processing is expensive

**Best for:** User-facing APIs, authentication, payment authorization, real-time notifications.

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ High Throughput                │ Low Latency                       │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Processing model    │ Batched                       │ Individual, immediate             │
│ Latency             │ Higher (batch wait)           │ Lower (immediate)                 │
│ Throughput          │ Higher (amortized overhead)   │ Lower per server                  │
│ Cost per operation  │ Lower                         │ Higher                            │
│ User experience     │ Poor for interactive          │ Excellent                         │
│ Resource efficiency │ High                          │ Low                               │
│ Use case            │ ETL, ML, bulk jobs            │ APIs, auth, payment, UI           │
│ Failure model       │ Batch fails → retry batch     │ Single op fails → retry op        │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 7. Strong Interview Answers

**Q: "Explain Little's Law and how it relates to throughput and latency."**

> "Little's Law states: N = λ × W, where N is the average number of items in a system, λ is the arrival rate (throughput), and W is the average time an item spends in the system (latency). This means: if your system processes 1,000 req/sec (λ) and each request takes 200ms (W), you need 200 concurrent requests in-flight simultaneously (N). If you want to reduce latency (W) while maintaining throughput (λ), you must reduce N — fewer things in-flight means less queuing. If you want to increase throughput (λ) while keeping N constant, you must reduce latency (W). This is why optimizing for throughput AND latency simultaneously is hard — they're linked. Practical implication: if your service has 100 worker threads and requests take 50ms each, your theoretical throughput ceiling is 100 / 0.05 = 2,000 req/sec. To serve 10,000 req/sec, you either need more threads/workers (add servers) or reduce latency (optimize the code)."

---

## 8. Real System Design Examples

| System | Priority | Technique |
|--------|----------|-----------|
| WhatsApp message delivery | Latency | Immediate delivery; no batching of messages |
| YouTube video encoding | Throughput | Batch encodes; latency of hours acceptable |
| Uber driver matching | Latency | Match must complete in < 2 seconds |
| ML model training | Throughput | Process millions of examples per second via GPU batching |
| Payment authorization | Latency | < 200ms required; batch would be disastrous |
| Log aggregation pipeline | Throughput | Batch logs into 10MB chunks; send every 5 seconds |

---

## 9. Decision Framework

```
"Does the user (human or machine) need the result immediately?"
YES → Optimize for LATENCY (minimize time to complete one operation)
NO  → Optimize for THROUGHPUT (maximize operations per second)

"Is this user-facing?"
YES → Latency is critical (P99 SLA matters)
NO  → Throughput typically more important than latency

"Can operations be batched without user-visible impact?"
YES → Batch for throughput (DB write batching, Kafka consumer batching)
NO  → Process individually for latency (payments, auth)

Rule: User-facing = latency. Background processing = throughput.
      Hybrid = optimize critical path for latency, background for throughput.
```

---
---

# TRADE-OFF 15: Latency vs Accuracy

---

## 1. Concept Overview

**What it is:** In many systems — especially analytics, ML, and approximate computation — you can get a FAST approximate answer OR a SLOW exact answer. Examples: approximate distinct count (HyperLogLog: 1ms, 1% error vs exact count: 30 seconds, 0% error), approximate trending topics (Count-Min Sketch: milliseconds vs exact SQL aggregate: minutes).

**Why it exists:** Computing exact results often requires scanning all data or waiting for all distributed nodes to respond. Approximate algorithms sacrifice a small amount of accuracy for dramatic latency and resource improvements.

**Why interviewers ask:** At scale, exact answers become prohibitively expensive. Senior engineers know when approximate is good enough — and what algorithms provide controlled approximation. This is the foundation of probabilistic data structures.

---

## 2. Visual Diagram

```
EXACT COUNT OF UNIQUE VISITORS:
SELECT COUNT(DISTINCT user_id) FROM events WHERE date = today
→ Full table scan: 10 billion rows → 30 minutes → exact: 47,234,891

APPROXIMATE (HyperLogLog):
For each user_id: hash → update HLL data structure (12KB total!)
Query result: 47,311,245 (error: ~0.16%) → available in milliseconds!

EXACT TRENDING TOPICS:
SELECT hashtag, COUNT(*) FROM tweets GROUP BY hashtag ORDER BY COUNT(*) DESC LIMIT 10
→ Scan all tweets of the day → expensive query → minutes

APPROXIMATE (Count-Min Sketch):
Maintain sketch in memory; update on each tweet; query top-10 in microseconds.
Error: might show a hashtag that's slightly lower frequency, miss one near boundary.
Acceptable for "trending" display? Almost always yes.
```

---

## 3. Option A — Exact Results

**Advantages:**
- 100% accurate — trustworthy for decisions with real consequences
- Auditable — exact computation can be verified and explained
- Required for compliance — financial reports, legal counts must be exact

**Disadvantages:**
- High latency — must scan all data, wait for all nodes
- High resource cost — large aggregations are expensive
- May not be feasible in real-time — "exact trending right now" requires exact counts at update speed

---

## 4. Option B — Approximate Results

**Advantages:**
- Dramatically lower latency (milliseconds vs minutes)
- Much lower resource cost — probabilistic structures use kilobytes of memory
- Scales linearly with data volume

**Disadvantages:**
- Small error margin (tunable but never zero)
- Not suitable for financial or legal reporting
- Users must understand the approximation (rare for consumer products)

---

## 5. Key Approximate Algorithms

```
HyperLogLog (approximate cardinality — COUNT DISTINCT):
  Error: ~0.81%, Memory: 12KB regardless of cardinality
  Use: unique visitor counts, distinct error types
  Used by: Redis (PFADD/PFCOUNT), BigQuery, Postgres (approx_count_distinct)

Count-Min Sketch (approximate frequency — top-K, heavy hitters):
  Error: bounded by configurable ε
  Use: trending topics, heavy hitter IP detection, frequency estimation
  (Recall from Algorithms & Patterns notes)

Bloom Filter (approximate set membership — "have I seen this?"):
  False positive rate: configurable
  Use: cache layer, deduplication, "already crawled?" check
  (Recall from Algorithms & Patterns notes)

Reservoir Sampling (uniform random sample from stream):
  Use: "random 1% of events" without knowing total count
  Maintains exact random properties with bounded memory
```

---

## 6. When To Choose Exact

- **Financial reporting:** Revenue figures, tax calculations — must be exact
- **Legal/compliance:** Audit logs, GDPR data counts — exact required
- **Inventory:** Number of items in stock — can't be approximate (oversell)
- **Small datasets:** When exact computation is fast, no need to approximate
- **Critical thresholds:** "Do we have more than 1,000,000 pending messages?" — binary threshold

---

## 7. When To Choose Approximate

- **Unique visitor counts for dashboards:** 47.3M vs 47.2M — irrelevant for product decisions
- **Trending topics:** Being slightly off at the margin is fine
- **A/B test sample sizes:** 1% sample vs 1.02% — statistically equivalent
- **Cache deduplication:** Bloom filter for "already cached?" — false positive means rare unnecessary DB call
- **Rate limiting:** Approximate rate limiter with Count-Min Sketch — occasional false positive is acceptable


---

## 9. Strong Interview Answers

**Q: "Design a system to count unique visitors to a website in real-time. 1 billion events per day."**

> "Exact unique counting at this scale with SQL is impractical in real-time — a full COUNT DISTINCT on a 1B-row table takes hours. I'd use HyperLogLog. Each page view event goes to a stream (Kafka). A stream processor (Flink) maintains a HyperLogLog sketch per day per page. Each user_id is hashed and the HLL is updated. At any point, querying the HLL gives an approximate unique count in milliseconds with ~0.81% error. A 12KB data structure tracks hundreds of millions of uniques. For the exact daily count (for billing or reporting), I'd run a nightly Spark job with exact COUNT DISTINCT and store the result. Real-time dashboard shows HLL approximation; official report shows exact batch result. This is the Lambda Architecture applied to a specific metric."

---

## 10. Real System Design Examples

| System | Exact or Approx | Algorithm |
|--------|----------------|-----------|
| YouTube view counts | Approximate (then exact daily) | Counters in Redis; exact via batch |
| Twitter trending | Approximate | Count-Min Sketch over tweet stream |
| Google Analytics | Approximate | HyperLogLog for uniques; sampling for events |
| Cassandra compaction | Bloom Filter | "Is this key in this SSTable?" |
| Ad click deduplication | Bloom Filter | "Have I seen this click before?" |

---

## 11. Decision Framework

```
Is exact accuracy legally/financially required? YES → Exact (accept latency cost)
Is the result for human display/decision making? → Approximate likely fine (humans can't use 8 significant figures)
Is the dataset > 1B rows? → Approximate dramatically cheaper; consider seriously
Is latency < 100ms required? → Exact at this scale is usually impossible; approximate is the only option
Error tolerance > 1%? → HyperLogLog, Count-Min Sketch are excellent choices
Error tolerance < 0.1%? → Consider exact computation or very fine-tuned approximation
```

---
---

# TRADE-OFF 16: CDN vs No CDN

---

## 1. Concept Overview

**What it is:** A CDN (Content Delivery Network) is a globally distributed network of edge servers that cache content close to users. Without a CDN, all requests go to your origin servers — potentially thousands of miles away, adding significant latency.

**Why it exists:** The speed of light is a real constraint. A user in Mumbai requesting content from a server in Virginia experiences ~180ms round-trip time regardless of how fast your server is. A CDN edge node in Mumbai serves the same content in ~5ms.

**Why interviewers ask:** CDN decisions affect latency, cost, availability, and global scalability. Understanding when CDN is worth the operational complexity and cost signals mature architectural thinking.

---

## 2. Visual Diagram

```
WITHOUT CDN:
User (Mumbai) → [180ms RTT] → Origin Server (Virginia)
                                      ↕ every request

WITH CDN:
User (Mumbai) → [5ms] → CDN Edge (Mumbai) → [CACHE HIT] → response
                                          → [CACHE MISS] → [180ms] → Origin → cache → respond

Global CDN network:
[Mumbai PoP] [London PoP] [Tokyo PoP] [NYC PoP] [São Paulo PoP] ...
Each caches popular content; users always hit the nearest PoP.
```

---

## 3. Option A — CDN

**Advantages:**
- Dramatically lower latency for global users (5ms vs 180ms)
- Offloads origin servers — 95%+ of static traffic served from edge
- DDoS absorption — CDN scale (100+ Tbps capacity) absorbs most attacks
- Higher availability — CDN serves cached content even if origin is down
- HTTPS termination at the edge — reduces TLS handshake latency

**Disadvantages:**
- Cost — CDN providers charge per GB transferred
- Stale content — cache invalidation is complex (TTL, purge APIs)
- Configuration complexity — cache rules, TTL policies, origin routing
- Limited to cacheable content — dynamic, personalized content doesn't benefit
- Cold cache after purge — origin hit spike after invalidation

**Cost:** Cloudflare/CloudFront: ~$0.01-0.085/GB depending on region and tier.

---

## 4. Option B — No CDN (Origin Only)

**Advantages:**
- Simpler architecture — no CDN configuration to manage
- Always fresh content — no cache invalidation complexity
- Suitable for highly dynamic content — content changes every request

**Disadvantages:**
- High latency for global users — cross-continental RTT unavoidable
- Origin overloaded — all requests hit origin servers
- DDoS vulnerable — origin is directly exposed
- No global distribution — performance degrades for users far from origin

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ With CDN                       │ Without CDN                       │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Latency (global)    │ 5-50ms (edge PoP)             │ 10-300ms (origin distance)        │
│ Origin load         │ 5-10% of traffic (cache hit)  │ 100% of traffic                   │
│ DDoS protection     │ Built-in (CDN absorbs)        │ Must implement separately          │
│ Availability        │ High (serve from cache)       │ Depends entirely on origin        │
│ Stale content risk  │ Yes (TTL-based)               │ None (always origin)              │
│ Cost                │ Per-GB CDN cost               │ Higher origin compute cost        │
│ Complexity          │ Cache rules, TTL, invalidation│ Simple                            │
│ Dynamic content     │ Not effective                 │ Better (always fresh)             │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Use CDN

- **Static assets:** JS/CSS/images with content-hash filenames (infinite TTL, no invalidation needed)
- **Video streaming:** Netflix, YouTube — video files are CDN-native content
- **Global user base:** Any product with users on multiple continents
- **High-traffic events:** Product launches, news events — CDN absorbs the spike
- **Public API responses:** Read-heavy endpoints returning same data to many users

---

## 7. When to Skip CDN (or Minimize)

- **Fully dynamic, personalized content:** User's own bank statement, social feed — not cacheable
- **Internal tools:** Your internal HR system used by 50 employees in one office
- **Latency-insensitive backend services:** Microservice-to-microservice calls stay in VPC
- **Highly sensitive data:** Some compliance regimes limit which third parties can see data in transit

---

## 9. Strong Interview Answers

**Q: "You deploy new CSS. How do you ensure users get the new file immediately and not a cached version?"**

> "The standard solution is content-hash filenames. Instead of 'styles.css', the build process generates 'styles.a3f8b2c1.css' where the hash is derived from the file contents. The HTML references this hashed filename. The CDN caches 'styles.a3f8b2c1.css' with a very long TTL (1 year or even immutable). When you update the CSS, the build generates 'styles.b7d1e4f9.css' — a different filename. The updated HTML references the new filename. Users' browsers fetch the new file (it's a new URL, not in cache). No cache invalidation needed. The only file that needs a short TTL or validation is the HTML page itself — it changes with every deploy. This approach gives you both: instant deployment of new assets AND maximum CDN cache efficiency."

---

## 10. Real System Design Examples

| System | CDN Usage | Why |
|--------|-----------|-----|
| Netflix | Massive (Open Connect CDN) | Video delivery; Netflix built their own CDN |
| Instagram | Yes (images/videos) | User-uploaded media cached at CDN edges globally |
| YouTube | Yes | Video files are CDN-native; huge bandwidth savings |
| Dropbox | Yes (shared files) | Publicly shared Dropbox files served from CDN |
| Uber (app assets) | Yes (mobile app assets) | JS/CSS/images for app web views |
| Ticketmaster | Yes (static) + No (seat availability) | Static page assets CDN; seat inventory always fresh |
| URL Shortener | Yes | Same redirect URL → same destination; highly cacheable |
| WhatsApp (media) | Yes | Sent media (images, videos) delivered via CDN |

---

## 11. Decision Framework

```
Is the content the same for all users? YES → CDN candidate
Is the content cacheable for > 5 minutes? YES → CDN makes sense
Do you have global users? YES → CDN is high-value
Is the content static (changes on deploy only)? YES → CDN with content-hash filenames

Is the content personalized per user? → CDN ineffective (no cache hit)
Is it internal/low-traffic? → CDN overhead not justified
Is content security/compliance-sensitive? → Evaluate CDN data routing carefully

Rule: CDN for everything static and shared. Skip for personalized or real-time data.
```

---
---

# TRADE-OFF 17: Kafka vs RabbitMQ

---

## 1. Concept Overview

**What it is:** Two of the most popular message broker systems, each optimized for different use cases. RabbitMQ is a traditional message broker optimized for task queues and routing. Kafka is an event streaming platform optimized for high-throughput, persistent, replayable event logs.

**Why it exists:** "Message broker" covers fundamentally different use cases: task distribution (one job → one worker) vs event streaming (one event → many consumers, with replay). Using the wrong tool for the use case wastes engineering effort and can introduce bugs.

**Why interviewers ask:** Both are extremely common in distributed systems interviews. Candidates often blindly say "use Kafka" — a red flag. Understanding the SPECIFIC differences shows mature engineering judgment.

---

## 2. Visual Diagram

```
RABBITMQ (Message Queue / Task Distribution):
Producer → [Exchange] → [Queue A] → Consumer 1 (processes job, message DELETED)
                     ↘ [Queue B] → Consumer 2 (competing consumer)
Message lives until consumed. Consumer ACKs → message gone. Simple routing.

KAFKA (Event Stream / Log):
Producer → [Topic: orders] → [Partition 0] → Consumer Group A (payment-service)
                                           → Consumer Group B (analytics-service)
                                           → Consumer Group C (email-service)
Message PERSISTED on disk for days/weeks. Each consumer group reads independently.
Consumer Group A processes payment; Group B processes analytics — same events, independently.
Replay: new service can read ALL past events from offset 0.
```

---

## 3. Option A — RabbitMQ

**Definition:** Traditional message broker. AMQP protocol. Sophisticated routing via exchanges and bindings. Messages consumed and deleted. Designed for task queues and work distribution.

**Advantages:**
- Complex routing — topic exchanges, fanout, direct routing, header routing
- Per-message TTL and dead letter queue built-in
- Lower operational complexity — simpler to set up and manage
- Push-based delivery — broker pushes to consumers (vs Kafka's pull)
- Multiple protocols — AMQP, MQTT, STOMP
- Better for low-latency task execution (< 100ms delivery typical)

**Disadvantages:**
- Messages deleted after consumption — no replay capability
- Lower throughput ceiling — typically < 50K messages/sec
- Not designed for millions of concurrent consumers
- No built-in partitioning — one queue is one unit of parallelism
- Stateful consumers — broker tracks delivery; more complex under failure

**Cost:** Lightweight; small infrastructure footprint for moderate volumes.

---

## 4. Option B — Kafka

**Definition:** Distributed event streaming platform. Messages are appended to a replicated log (partition) and retained for a configurable period. Multiple consumer groups can read the same events independently.

**Advantages:**
- Very high throughput — millions of messages/sec
- Durable persistence — events stored on disk (days, weeks, forever)
- Replay capability — new consumers can read from the beginning
- Multiple independent consumers — same event to many services without re-publishing
- Exactly-once semantics (with idempotent producers + transactions)
- Built for stream processing — Kafka Streams, Flink integrate natively

**Disadvantages:**
- No complex message routing — routing is by topic and partition key
- Higher operational complexity — ZooKeeper (legacy) or KRaft, cluster management
- Message ordering within partition only — not global ordering
- Overkill for simple task queues — massive overhead for "process this job"
- Consumers must manage offsets — "where am I in the log?" is consumer responsibility
- Higher resource requirements — Kafka clusters need dedicated infrastructure

**Cost:** Higher — requires a cluster of brokers; significant operational investment.

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ RabbitMQ                       │ Kafka                             │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Throughput          │ Up to ~50K msg/sec             │ Millions of msg/sec               │
│ Message retention   │ Until consumed (deleted)       │ Configurable (days/weeks/forever) │
│ Replay              │ No                            │ Yes                               │
│ Multiple consumers  │ Each message → one consumer   │ Multiple consumer groups, same msg │
│ Routing flexibility │ Very high (exchanges, bindings)│ Low (topic + partition key)       │
│ Ordering guarantee  │ Per queue (FIFO)               │ Per partition                     │
│ Operational effort  │ Low-moderate                  │ High                              │
│ Stream processing   │ Not designed for              │ Native integration (Flink, Streams)│
│ Message size        │ Flexible                      │ Best for < 1MB per message        │
│ Use case            │ Task queues, RPC              │ Event streaming, data pipelines    │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose RabbitMQ

- **Background job processing:** Image resizing, email sending, report generation
- **Task queues with competing consumers:** Multiple workers process the same queue
- **Complex routing requirements:** Different message types route to different consumers based on content
- **RPC-style async:** Request-reply patterns where response routes back to specific caller
- **Low-to-medium volume:** < 50K messages/sec
- **Companies:** When you need work distribution without streaming complexity (most startups)

---

## 7. When To Choose Kafka

- **Event sourcing:** All system state changes as an immutable event log
- **Multiple independent consumers:** The same "order placed" event must go to payment, inventory, email, analytics — each independently
- **High-throughput event streams:** 1M+ events/sec (clickstream, IoT, logs)
- **Replay required:** New service needs to process all past events
- **Stream processing integration:** Flink/Spark Streaming consuming from Kafka
- **Audit log:** Immutable, ordered log of all events in the system
- **Companies:** LinkedIn (origin), Uber, Netflix, Airbnb — event-driven at scale

---

## 9. Strong Interview Answers

**Q: "You have 5 services that all react to 'order placed'. Kafka or RabbitMQ?"**

> "Kafka is the right choice here, and here's why. In RabbitMQ, once a message is consumed and ACK'd, it's gone. If 5 services need to react to an order, you'd need to either: publish the message 5 times to 5 separate queues (coupling the producer to all consumers), or use a fanout exchange (duplicates the message to 5 queues). With Kafka, the order-placed event is published ONCE to a Kafka topic. Each service creates its own consumer group. Payment service has consumer group 'payment-processors', inventory service has 'inventory-updaters', etc. Each consumer group maintains its own offset — where in the log it has read to. The Kafka broker doesn't care how many consumer groups there are. Payment service processes the event independently of whether inventory service has processed it. A new service can be added tomorrow and read from offset 0 to catch up on all historical orders without any change to the producer or other consumers. This is why Kafka is the natural choice for event-driven architectures with multiple independent consumers."

**Q: "When should you choose RabbitMQ over Kafka?"**

> "RabbitMQ is the better choice when you need: one — complex routing logic (route emails about payments to the email service, but emails about marketing to a different service, based on message headers); two — simple task distribution where exactly ONE worker should process each job (RabbitMQ's competing consumers pattern is elegant and battle-tested; Kafka can do this but it's not its primary design); three — when you don't need replay or long-term event retention (you just need to process jobs reliably); four — lower operational complexity — a small team can run RabbitMQ more easily than a Kafka cluster. For a startup building an email notification system or image processing pipeline, RabbitMQ is the right choice — Kafka would be overkill that consumes engineering time better spent on product."

---

## 10. Common Mistakes

- **"Always use Kafka"** — Kafka is overkill for simple task queues; the Kafka tax (operational complexity) is real
- **Not understanding message retention differences** — building replay on RabbitMQ is extremely complex; if replay is needed, use Kafka
- **Mixing up "consumer" and "consumer group"** — in Kafka, multiple consumers in one group share the load; across groups, same messages
- **Using Kafka for RPC-style request-reply** — Kafka is not designed for this; RabbitMQ handles it natively
- **Ignoring the "poison message" problem** — a message that always crashes consumers needs a dead letter queue strategy (both support DLQ but differently)

---

## 11. Real System Design Examples

| System | RabbitMQ | Kafka | Reasoning |
|--------|---------|-------|-----------|
| WhatsApp | Message delivery queues | Change data capture | Per-user task delivery; CDC pipeline |
| Instagram | Email/notification jobs | Event streaming (likes, follows) | Task queue for jobs; Kafka for activity streams |
| Uber | Driver job dispatch | Location event streaming | Task queue; high-volume location events |
| YouTube | Encoding job queue | View event stream | Job processing; clickstream analytics |
| Netflix | Background job dispatch | Activity event log, recommendations feed | Jobs; event-driven recommendations |
| Ticketmaster | Email confirmation jobs | Inventory change events | Simple task; event sourcing |

---

## 12. Decision Framework

```
Multiple services need the SAME event? → Kafka (multiple consumer groups)
Need to replay historical events? → Kafka (persistent log)
Need complex routing (type-based, header-based)? → RabbitMQ (exchanges)
Processing ONE job, exactly ONCE, by ONE worker? → RabbitMQ (simpler, built for this)
High throughput > 100K msg/sec? → Kafka
Simple to moderate volume, simple routing? → RabbitMQ
Need stream processing integration? → Kafka (Flink, Kafka Streams)
Small team, limited Kafka expertise? → RabbitMQ or AWS SQS

Quick rule: Task queue → RabbitMQ. Event streaming / multiple consumers → Kafka.
```

---
---

# TRADE-OFF 18: Normalization vs Denormalization

---

## 1. Concept Overview

**What it is:** Normalization organizes data to minimize redundancy — each fact stored once, related through foreign keys. Denormalization intentionally duplicates data to optimize read performance — pre-joining related data, storing derived values, embedding related entities.

**Why it exists:** Normalized data is correct and consistent but requires JOINs to assemble, which are expensive at scale. Denormalized data is fast to read but must be kept consistent when underlying data changes — harder to write, harder to maintain correctness.

**Why interviewers ask:** Data modeling decisions affect query performance, write complexity, and data consistency for the lifetime of the system. This trade-off is central to any database design discussion.

---

## 2. Visual Diagram

```
NORMALIZED (3NF):
users table:    (user_id, name, email)
orders table:   (order_id, user_id, total, created_at)
products table: (product_id, name, price, category)
order_items:    (order_id, product_id, quantity)

Query "get order with all details":
SELECT o.*, u.name, u.email, p.name, p.price, oi.quantity
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
WHERE o.order_id = 123
→ Correct, but requires 4-table JOIN

DENORMALIZED (for order history display):
orders_denormalized table:
(order_id, user_id, user_name, user_email, product_name, product_price, quantity, total)
→ All fields in one row
→ Query: SELECT * FROM orders_denormalized WHERE order_id = 123 ← one row, no JOIN!
But: if product name changes → must UPDATE all historical order rows too ← complexity
```

---

## 3. Option A — Normalization

**Definition:** Organize data into tables to eliminate redundant data. Each fact stored in exactly one place. Related data linked via foreign keys.

**Advantages:**
- Data consistency — change user's email in one place; all orders reflect it instantly
- No update anomalies — no risk of partially updated duplicate data
- Smaller storage footprint — each fact stored once
- Flexible querying — JOIN any combination of tables at query time

**Disadvantages:**
- JOIN overhead — complex queries across many tables are slow
- Not suitable for sharding — JOINs break when data is on different servers
- Recursive traversal expensive — deeply nested relationships need many JOINs
- Read performance degrades with table size and JOIN complexity

---

## 4. Option B — Denormalization

**Definition:** Intentionally duplicate data or pre-compute derived values to optimize read performance. Accept data redundancy for faster, simpler queries.

**Advantages:**
- Faster reads — single table scan, no JOINs
- Simpler queries — SELECT from one table, not 5
- Enables sharding — all data for an entity in one place; no cross-shard JOINs needed
- Pre-computed aggregates — store COUNT, SUM in a column instead of computing at read time

**Disadvantages:**
- Update anomalies — when the canonical data changes, all duplicates must be updated
- Write complexity — every write may update many records
- Consistency risk — bugs in update logic leave some copies stale
- Storage overhead — same data duplicated N times

---

## 5. Side-by-Side Comparison Table

```
┌────────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Factor              │ Normalization                  │ Denormalization                   │
├────────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Read performance    │ Slower (JOINs needed)         │ Faster (single table)             │
│ Write performance   │ Faster (write once)           │ Slower (update all duplicates)    │
│ Data consistency    │ Guaranteed                    │ Risk of inconsistency             │
│ Storage             │ Efficient (no duplication)    │ Larger (duplicated data)          │
│ Query flexibility   │ High (any combination of JOIN)│ Low (optimized for specific query)│
│ Update complexity   │ Simple (one place)            │ Complex (N places)                │
│ Shard-friendliness  │ Poor (cross-shard JOINs)     │ Good (all data per entity together)│
│ Schema evolution    │ Easier (change once)          │ Harder (change many places)       │
└────────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 6. When To Choose Normalization

- **OLTP systems:** Frequent writes where consistency is critical (banking, inventory)
- **Complex query requirements:** Ad-hoc queries with different JOIN combinations
- **Data that changes frequently:** User email, product prices — store once, always correct
- **Early-stage product:** Requirements change; normalized schema is easier to evolve
- **Small to medium scale:** JOINs on a well-indexed, properly sized DB are fast

---

## 7. When To Choose Denormalization

- **Read-heavy workloads:** 100:1 read:write ratio — optimize for the common case
- **Sharded databases:** Must avoid cross-shard JOINs; denormalize into the shard's data
- **Analytics/OLAP:** Data warehouses denormalize into star schema for fast aggregations
- **Document databases (NoSQL):** MongoDB's document model IS denormalization — embed related data
- **High-traffic read endpoints:** Product page, user profile — denormalize into a pre-joined view
- **Historical records:** Order receipt — embed product name/price at order time (they can change; receipt must be stable)


---

## 9. Strong Interview Answers

**Q: "You're building the order history page for an e-commerce site. The page shows order details including product names and prices. How do you model the data?"**

> "This is a classic snapshot vs live reference trade-off. The order history page must show what the customer ACTUALLY paid and what product they ACTUALLY bought at the time of purchase — not what the product is called today or what it costs today. If I normalize and just store product_id in the order_items table, when a product is renamed or its price changes, the order history would show the new name/price — incorrect for a receipt. I'd denormalize intentionally: at the time of order creation, I copy the product name and price INTO the order_items row: {order_id, product_id, product_name_at_purchase, product_price_at_purchase, quantity}. These historical fields are immutable — they're a snapshot of reality at purchase time. The product table still exists as the source of truth for current listings; the order_items table has a point-in-time denormalized snapshot. This is one of the few cases where denormalization is not just a performance optimization but a CORRECTNESS requirement."

---

## 10. Common Mistakes

- **Over-normalizing then complaining about JOIN performance** — know when to denormalize proactively
- **Denormalizing everything preemptively** — adds complexity before you know you need it
- **Not handling update propagation** — denormalize but forget to update all copies when canonical data changes
- **Normalizing historical records** — order receipts, audit logs should be snapshots, not live references
- **Confusing denormalization in SQL with NoSQL document modeling** — conceptually similar but implementation differs

---

## 11. Real System Design Examples

| System | Normalized | Denormalized | Why |
|--------|-----------|-------------|-----|
| Instagram | User table (normalized) | Feed items (denormalized with author info) | Feed reads must be fast; author info duplicated |
| Uber | Trip metadata (normalized) | Driver earnings report (denormalized) | Reports are read-only snapshots |
| Netflix | Movie catalog (normalized) | User recommendation data (denormalized) | Recommendations need flat, fast reads |
| Ticketmaster | Event/venue (normalized) | Search index (fully denormalized) | Search requires flat documents |
| Amazon | Product (normalized) | Order receipt (denormalized snapshot) | Historical accuracy required |
| Dropbox | File metadata (normalized) | Shared link preview (denormalized) | Preview must be fast |

---

## 12. Decision Framework

```
Read:write ratio > 10:1? → Consider denormalization for read-heavy tables
Data changes frequently? → Normalize (one update vs N updates)
Historical accuracy needed? → Denormalize as a snapshot (order receipt, audit log)
Using NoSQL? → Denormalize naturally (embed related data in document)
Using sharded DB? → Denormalize to avoid cross-shard JOINs
Building a data warehouse? → Star schema (denormalized) is standard
Building OLTP? → Normalize first; denormalize specific hot paths if needed

CQRS pattern:
Write model → Normalized (consistency, ACID)
Read model → Denormalized projection (optimized for specific queries, eventually consistent)
```

---
---

# RELATIONSHIPS BETWEEN TRADE-OFFS

---

## How Trade-Offs Interconnect

Understanding how these trade-offs relate to each other is what separates senior engineers from juniors. No architectural decision is made in isolation.

---

## 1. Cache vs Database → Affects Latency AND Consistency

```
Adding a cache (Trade-off 6):
  → Reduces latency dramatically (Trade-off 13: Memory vs Latency)
  → Introduces eventual consistency between cache and DB (Trade-off 3)
  → If cache is warm, reduces DB load → allows vertical scaling to last longer

CHAIN: Cache → Low Latency BUT Eventual Consistency
       No Cache → High Latency BUT Strong Consistency (always fresh from DB)
```

---

## 2. Replication → Affects Consistency AND Availability

```
Synchronous replication (Trade-off 8):
  → Strong consistency (Trade-off 3) — reads from replicas are always fresh
  → Lower availability — write fails if replica is unreachable (CP — Trade-off 2)
  → Higher write latency (Trade-off 14) — must wait for all replicas

Asynchronous replication:
  → Eventual consistency (Trade-off 3) — replica may be stale
  → Higher availability — write succeeds even if replica is slow (AP — Trade-off 2)
  → Lower write latency — don't wait for replicas

CHAIN: Replication Type → Determines CAP Position (Trade-off 2) AND Consistency Model (Trade-off 3)
```

---

## 3. Sharding → Affects Consistency AND Scalability AND Query Flexibility

```
Choosing to shard (Trade-off 8):
  → Eliminates cross-shard JOINs → forces denormalization (Trade-off 18)
  → Cross-shard transactions → need Saga pattern → eventual consistency (Trade-off 3)
  → Enables horizontal scaling (Trade-off 12)
  → BUT breaks strong consistency across entities on different shards (Trade-off 2)

CHAIN: Sharding → Denormalization → Eventual Consistency → Saga for Transactions
```

---

## 4. Microservices → Affects Consistency AND Communication Pattern

```
Choosing microservices (Trade-off 7):
  → Services need to communicate → sync or async decision (Trade-off 5)
  → No shared DB → must use events → eventual consistency (Trade-off 3)
  → Async events → need Kafka or RabbitMQ (Trade-off 17)
  → Independent deployments → REST or GraphQL API design needed (Trade-off 10)
  → Data ownership per service → must denormalize or use API composition (Trade-off 18)

CHAIN: Microservices → Async Communication → Kafka/RabbitMQ → Eventual Consistency → Denormalization
```

---

## 5. Push Architecture → Affects Storage AND Compute

```
Choosing push model (Trade-off 4):
  → Pre-computes and stores data in user inboxes → denormalization (Trade-off 18)
  → Requires high write throughput → horizontal scaling of write tier (Trade-off 12)
  → More storage per item (N copies) vs pull (1 copy) → memory vs latency (Trade-off 13)
  → Fan-out is async → asynchronous communication (Trade-off 5) → Kafka (Trade-off 17)

CHAIN: Push Architecture → Denormalized Storage → Async Fan-out via Kafka → Horizontal Write Scaling
```

---

## 6. Throughput vs Latency → Affects Architecture Fundamentally

```
Optimizing for THROUGHPUT (Trade-off 14):
  → Batch processing (Trade-off 11)
  → Can tolerate eventual consistency (Trade-off 3)
  → Horizontal scaling for parallel work (Trade-off 12)
  → Pull model more natural (Trade-off 4)

Optimizing for LATENCY (Trade-off 14):
  → Stream processing (Trade-off 11)
  → Often requires strong consistency or fast eventual consistency (Trade-off 3)
  → Memory-heavy architecture (Trade-off 13)
  → Push model or WebSocket (Trade-offs 4, 9)

CHAIN: Latency target → Stream/Batch choice → Consistency model → Memory architecture
```

---

## The Master Trade-Off Map

```
                    SQL vs NoSQL (1)
                         │
              ┌──────────┼──────────┐
              │           │           │
         CAP (2)    Strong/Eventual(3)  Normalization/Denorm(18)
              │           │           │
              │      Replication(8)   │
              │           │           │
              └───────────┴───────────┘
                          │
                    Consistency affects...
                    ↓                    ↓
             Microservices(7)      Horizontal Scaling(12)
                    │                    │
             ┌──────┘                    └──────┐
             │                                  │
        Sync/Async(5)                   Replication/Sharding(8)
             │
    ┌────────┼────────┐
    │        │        │
  Push/Pull Kafka/   WebSocket/
  (4)    RabbitMQ(17) Polling(9)
             │
         Throughput/
         Latency(14)
             │
    ┌────────┼────────┐
    │        │        │
  Batch/  REST/  Memory/
  Stream  GraphQL Latency
  (11)    (10)    (13)
             │
         Latency/
         Accuracy(15)
         CDN/NoCDN(16)
```

---

## Quick Reference: Trade-Off Decision Cheat Sheet

```
SCENARIO → TRADE-OFF CHAIN

"Need to scale writes" → Horizontal Scaling → Sharding → Denormalization → Eventual Consistency
"Need real-time feed" → Push Architecture → Async (Kafka) → Denormalized Inboxes
"Need global users" → CDN + Eventual Consistency → Approximate for analytics
"Need fraud detection" → Stream Processing → Low Latency → Memory-heavy
"Building microservices" → Async Communication (Kafka) → Eventual Consistency → Saga for Transactions
"Startup launching MVP" → Monolith → SQL → Normalized → Synchronous APIs → No CDN needed yet
"500M users" → Microservices → NoSQL + Sharded SQL → Denormalized → AP-leaning → CDN everywhere
```

---

## Final Interview Strategy

```
When asked ANY system design trade-off question:

STEP 1: CLARIFY THE REQUIREMENT
"What is the read:write ratio? What's the latency requirement? Global or regional?"

STEP 2: STATE THE TRADE-OFF EXPLICITLY
"The key trade-off here is between [Option A] and [Option B]. Let me explain both..."

STEP 3: GIVE THE DECISION WITH REASONING
"Given your requirements (high read volume, eventual consistency acceptable),
I'd choose [Option A] because..."

STEP 4: ACKNOWLEDGE THE DOWNSIDE
"The downside is [specific cost/complexity]. We'd mitigate it by [specific mitigation]."

STEP 5: CONNECT TO SCALE
"At 10x traffic, this decision [holds/breaks] because [reason]. At that point, we'd..."

NEVER say: "It depends." without immediately explaining WHAT it depends on and giving a recommendation.
ALWAYS say: "Given [specific constraints], I'd choose [specific option] because [specific reason]."
```
