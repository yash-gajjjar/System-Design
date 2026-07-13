# Consistency & Reliability — Complete Deep-Dive Revision Guide
## System Design Interview Preparation | Product-Based Companies

---

## Table of Contents

1. **Consistency Models** — Linearizability, sequential, causal, eventual consistency, vector clocks, CRDTs
2. **Availability vs Consistency** — PACELC, tunable consistency, read-your-own-writes, bounded staleness
3. **Fault Tolerance** — Failure categories, redundancy, timeouts, retries with jitter, graceful degradation, FMEA
4. **Distributed Transactions** — 2PC problems, Saga pattern, choreography vs orchestration, compensating transactions
5. **Idempotency** — Idempotency keys, Redis implementation, Stripe's pattern, DB-level idempotency, race conditions
6. **SLA / SLO / SLI** — Definitions, error budgets, golden signals, rolling vs calendar windows, burn rate alerts
7. **Chaos Engineering** — Principles, experiment process, Chaos Monkey, Gremlin, Game Days
8. **Appendix** — Cross-topic reference, complete reliability architecture, BFSI-specific tips, five resilience patterns

---

# TOPIC 1: Consistency Models

---

## 1. What Problem Do Consistency Models Solve?

When data is stored on a SINGLE machine, reading after writing is trivially correct — write a value, read it back, you get what you wrote. But in distributed systems, data is replicated across multiple nodes for availability and performance (recall Replication from Databases notes). Now a fundamental question arises: **after a write to one node, what does a read from another node see?**

```
THE DISTRIBUTED CONSISTENCY CHALLENGE:

Single machine (trivial):
Write: x = 10  →  Read: x = 10  ✅ Always correct

Distributed (multiple replicas):
Node A: Write x = 10
        (replication starts, takes some time...)

Node B: Read x = ???
  - Returns 10 (new value)? "Strong consistency"
  - Returns 5 (old value)? "Stale read" — "weak consistency"
  - Returns 7 (some intermediate?)

WHICH ANSWER IS CORRECT DEPENDS ON THE CONSISTENCY MODEL.
Different models make different guarantees — the right choice
depends on your use case.
```

**Analogy:** Imagine editing a Google Doc while someone else views it. Do they see your latest changes INSTANTLY (strong consistency) or only after some propagation delay (eventual consistency)? Both are "correct" behaviours — they just represent different consistency guarantees.

---

## 2. The Consistency Spectrum — From Strongest to Weakest

```
STRONGEST                                               WEAKEST
    │                                                       │
    ▼                                                       ▼
Linearizability → Sequential → Causal → Eventual
(Strict)         Consistency  Consistency Consistency
```

---

## 3. Linearizability (Strict Consistency) — The Gold Standard

### Core Intuition

```
DEFINITION: Every operation appears to take effect ATOMICALLY
at some point between its invocation and response — and all
operations appear to happen in a single global real-time order.

SIMPLER: The distributed system behaves as if there is EXACTLY
ONE COPY of the data, even though there are many replicas.
Every read sees the MOST RECENT completed write. Full stop.

REAL-TIME ORDER:
If Write(x=10) COMPLETES before Read(x) BEGINS:
  → Read MUST return 10 (or a value written even more recently)
  → Can NEVER return the old value

If Write(x=10) and Read(x) OVERLAP in time:
  → Read MAY return either old or new value (concurrent)
  → But once a read has returned new value, no subsequent read
    can return the old value
```

### Step-by-Step Diagram

```
TIME →

Client 1: ──[Write x=10]─────────────────────────────────────
Client 2: ──────────────────────[Read x]─────────────────────
Client 3: ──────────────────────────────────[Read x]─────────

If Write x=10 completes before Client 2's Read begins:
  Client 2 MUST see x=10 ✅
  Client 3 MUST see x=10 ✅

Linearizability guarantees: after the write completes, ALL
subsequent reads from ANY node return the new value.
This is what makes linearizability so strong — and so expensive.
```

### Cost of Linearizability

```
TO ACHIEVE LINEARIZABILITY:
Every write must be coordinated across all replicas BEFORE
being acknowledged. Every read must check with enough replicas
to guarantee seeing the latest write.

MECHANISMS:
- Synchronous replication (primary waits for ALL replicas)
- Quorum reads+writes (W+R > N in Dynamo-style systems)
- Consensus protocols (Raft, Paxos — one leader coordinates all writes)

PERFORMANCE COST:
  Each operation requires multiple network round trips.
  Latency = at least 1 RTT across all involved nodes.
  Throughput limited by the SLOWEST replica in the quorum.

AVAILABILITY COST (CAP theorem — Databases notes):
  If a network partition occurs, a linearizable system MUST
  refuse to respond (to avoid returning stale data) — it
  sacrifices Availability for Consistency (CP system).

WHO USES IT:
  - Financial transactions (bank balance must be exact)
  - Distributed locks (only ONE holder at any time)
  - Leader election (exactly ONE leader)
  - Configuration stores (ZooKeeper, etcd use Raft for linearizability)
```

---

## 4. Sequential Consistency — Slightly Relaxed

```
DEFINITION: All operations appear to execute in SOME sequential
order that is consistent with the order seen by each individual
process — but NOT necessarily real-time order.

KEY DIFFERENCE FROM LINEARIZABILITY:
Linearizability: respects real-time (wall-clock) ordering.
Sequential: only respects per-process ordering (if process P
sees A before B, everyone must see A before B — but two
CONCURRENT operations from DIFFERENT processes can be ordered
either way).

EXAMPLE:
Client 1 writes: x=1, then x=2
Client 2 reads twice

LINEARIZABLE requires: Client 2 eventually reads x=2.
SEQUENTIAL allows: Client 2 might always read x=1
(if the system executes all of Client 2's reads
BEFORE Client 1's writes in the sequential order)
— even if those reads happened AFTER the writes in wall time!

PRACTICAL USE: CPU memory models (x86, ARM) use sequential
consistency or weaker models internally — which is why
concurrent programming requires explicit memory barriers/fences.
Less commonly discussed in distributed system design interviews;
linearizability and eventual consistency are more relevant.
```

---

## 5. Causal Consistency — Preserving Cause and Effect

```
DEFINITION: Operations that are CAUSALLY RELATED must be seen
by all processes in the same causal order. Causally unrelated
(concurrent) operations can be seen in ANY order.

CAUSALITY: Operation B is causally dependent on A if:
  - B happened AFTER A on the SAME process (program order)
  - B happened AFTER A read A's value (read-write dependency)
  - Transitively: B causally depends on something that causally
    depends on A

INTUITION: "If you saw my question and then answered it,
everyone should see the question before your answer."

DIAGRAM:
Process P1: Write(x=1) ──────────────────────────────────────
Process P2: Read(x) → 1, Write(y = x+1 = 2) ────────────────
Process P3: ───────────────────────────────── Read(y) Read(x)

CAUSAL GUARANTEE:
P3 must see y=2 BEFORE it sees any value for x that would
make y=2 "impossible" (i.e., x must be at least 1 when y=2).
The causally related sequence is preserved.

P3 might not see x=1 (could still see old x=0 if the write
hasn't propagated yet) — but if it reads y=2, it MUST be
able to see the cause (x=1 write that P2 read before writing y).

STRONGER THAN EVENTUAL, WEAKER THAN LINEARIZABILITY:
Causally consistent systems CAN have different concurrent
ordering across nodes — but causally linked operations maintain
their order everywhere.

USED BY:
- Social media comment threads (reply must appear after original post)
- Collaborative editing (Google Docs uses a form of causal consistency)
- Chat applications (messages must appear in causal order)
- Amazon's internal systems for shopping cart operations
```

### Vector Clocks — Tracking Causality

```
To implement causal consistency, systems need to TRACK
which events causally precede others. Vector clocks do this.

VECTOR CLOCK: An array of counters, one per node.
Each node increments ITS OWN counter on each local event.
Before sending a message, a node includes its current vector.
On receiving a message, a node takes the element-wise MAX
of its own vector and the received vector, then increments
its own counter.

EXAMPLE (3 nodes: A, B, C — clocks start at [0,0,0]):

Node A writes x=1:   A's clock [1,0,0], value x=1 tagged [1,0,0]
A sends to B:        B receives, updates clock to max([0,0,0],[1,0,0])=[1,0,0]
B writes y=2:        B's clock [1,1,0], value y=2 tagged [1,1,0]

NOW: Is x=1 causally before y=2?
  x's tag [1,0,0] ≤ y's tag [1,1,0] (every element ≤)?
  YES → x causally precedes y ✅

Node C writes z=3 concurrently (no communication with A or B):
  C's clock [0,0,1], value z=3 tagged [0,0,1]

Is x=1 causally before z=3?
  x's tag [1,0,0] vs z's tag [0,0,1]:
  1 > 0 in first element → NOT causally ordered!
  These are CONCURRENT events — can be ordered either way.
```

---

## 6. Eventual Consistency — The Practical Default

```
DEFINITION: If no new updates are made to a given data item,
eventually all replicas will converge to the same value.

THE KEY WORD: "EVENTUALLY" — there is NO bound on how long
this takes. Could be milliseconds, could be seconds, could
be longer during network partitions.

WHAT IT DOES NOT GUARANTEE:
- A read immediately after a write returns the new value
- Two concurrent reads from different replicas return the same value
- Any specific time bound on convergence

WHAT IT DOES GUARANTEE:
- The system WILL eventually agree (no permanent divergence)
- Writes are not lost (with proper anti-entropy mechanisms)

WEAK FORMS OF EVENTUAL CONSISTENCY:
"Read Your Own Writes": after YOU write, YOUR subsequent reads
  see YOUR write. Others may not see it yet, but you do.
"Monotonic Reads": once you've read a value v, you'll never
  read an older value. Reads are non-decreasing in "freshness."
"Monotonic Writes": your writes are applied in the order you
  issued them (even if other clients can't see them yet).
```

### Convergence Mechanisms

```
HOW DO EVENTUALLY CONSISTENT SYSTEMS CONVERGE?

1. GOSSIP PROTOCOL (Anti-entropy):
   Nodes periodically "gossip" with random peers:
   "Hey, I have these versions of data, what do you have?"
   Each node RECONCILES differences.
   Amazon DynamoDB, Cassandra use gossip for convergence.

   NODE A ─── "I have x=10 at time T5" ───▶ NODE B
           ◀── "I have x=7 at time T3" ──── NODE B
   NODE A concludes: x=10 (T5 > T3 → newer) → updates B.

2. CONFLICT RESOLUTION (when both sides changed):
   Last-Write-Wins (LWW): compare timestamps, higher wins.
     Simple but can lose data (the "earlier" write is discarded).
   Vector clocks: determine causal order, keep causally later.
   CRDTs (Conflict-free Replicated Data Types): mathematically
     designed data structures that ALWAYS merge without conflict.
     Example: G-Counter (grow-only counter): each node has its
     own counter, total = sum of all node counters. Any two nodes
     can merge by taking element-wise MAX — no conflict possible!
     Used by: Riak, Redis CRDT module, collaborative text editors.
```

---

## 7. Consistency Models Comparison Table

```
┌───────────────────┬─────────────────────────────┬────────────────────┬───────────────────────┐
│ Model              │ Guarantee                    │ Performance        │ Real-World Use         │
├───────────────────┼─────────────────────────────┼────────────────────┼───────────────────────┤
│ Linearizability    │ Every read sees the latest   │ Slowest — requires │ etcd, ZooKeeper, bank  │
│ (Strict)           │ completed write globally     │ quorum/consensus   │ balances, locks, leader│
│                   │                              │                    │ election               │
├───────────────────┼─────────────────────────────┼────────────────────┼───────────────────────┤
│ Sequential         │ Global order exists, not     │ Fast but requires  │ CPU memory models,     │
│                   │ necessarily real-time         │ coordination       │ multi-threaded apps    │
├───────────────────┼─────────────────────────────┼────────────────────┼───────────────────────┤
│ Causal             │ Causally related ops in      │ Medium — only      │ Social feeds, chat,    │
│                   │ causal order everywhere       │ causal tracking    │ collaborative editing  │
├───────────────────┼─────────────────────────────┼────────────────────┼───────────────────────┤
│ Eventual           │ Will converge, no time bound │ Fastest — no       │ DNS, shopping cart,    │
│                   │                              │ coordination needed│ social like counts,    │
│                   │                              │                    │ CDN caches             │
└───────────────────┴─────────────────────────────┴────────────────────┴───────────────────────┘
```

---

## 8. Real-World Usage

**Amazon DynamoDB:** Offers BOTH eventual consistency (default) AND strong consistency (opt-in per read, at higher latency/cost). Default reads are eventually consistent — returned from any replica immediately. Strongly consistent reads always read from the leader and are guaranteed to reflect all prior writes. Most applications use eventual consistency for performance and accept the trade-off.

**Google Spanner:** The rare system that achieves linearizability GLOBALLY across data centers using TrueTime (GPS + atomic clock synchronization). The "external consistency" guarantee: if transaction T1 commits before T2 begins in real time, T2 always sees T1's writes. Achieved at significant engineering cost — most companies can't or don't need to replicate this.

**Apache Cassandra (Tunable Consistency):** Cassandra's elegance is making consistency TUNABLE per-operation. You specify: write to W replicas before ACK (W=1 fastest, W=ALL strongest), read from R replicas (R=1 fastest, R=QUORUM stronger). If W + R > N (total replicas), you get strong consistency. If not, eventual. Operators choose per query based on how much consistency that operation needs.

---

## 9. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Stale read causes incorrect      │ Eventual consistency — read      │ Use strong consistency for        │
│ business decision (e.g., user   │ from stale replica after write   │ critical reads (bank balance,     │
│ sees wrong balance after txn)    │ to another replica               │ inventory last unit); eventual    │
│                                  │                                  │ consistency for non-critical data │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Conflicting writes cause data    │ Two nodes accept concurrent      │ LWW timestamps (risk: data loss); │
│ loss in eventual consistent DB   │ writes to same key without       │ CRDTs (merge without loss); Saga  │
│                                  │ coordination                     │ pattern for complex transactions  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Linearizable system refuses all  │ Network partition + CP choice   │ Design for graceful degradation:   │
│ reads/writes during network      │ → system unavailable during      │ serve stale data during partition │
│ partition → full outage          │ partition rather than return     │ with clear "may be stale" warning; │
│                                  │ stale data                       │ or architect to not need strict   │
│                                  │                                  │ consistency for most ops          │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 10. Interview Quick-Fire Answers

**Q: What is the difference between linearizability and eventual consistency?**
A: Linearizability is the strongest consistency model — every read sees the most recent completed write, and operations appear to execute atomically in a single global real-time order. It's as if there's only one copy of the data. The cost is high latency (requires coordination across replicas) and reduced availability during partitions. Eventual consistency is the weakest — replicas will EVENTUALLY converge to the same value, but there's no time guarantee and reads may return stale values. It's very fast (no coordination) and highly available. Most systems offer something in between, or make consistency tunable per operation.

**Q: What is causal consistency and when would you use it?**
A: Causal consistency ensures that causally related operations are seen by all nodes in the same causal order — if you saw a question and wrote an answer, everyone sees the question before the answer. Causally unrelated (concurrent) operations can be seen in any order. It's stronger than eventual consistency (preserves cause-effect chains) but weaker than linearizability (doesn't require global real-time ordering). Use it for: social media comment threads (replies must appear after original posts), collaborative editing, chat applications where conversational context must be preserved, and any system where cause-and-effect relationships must be consistent across replicas.

**Q: What are CRDTs and what problem do they solve?**
A: CRDTs (Conflict-free Replicated Data Types) are data structures mathematically designed so that concurrent updates from different replicas can ALWAYS be merged without conflicts — no matter the order of operations or which replica receives which updates first. For example, a G-Counter (grow-only counter) stores a separate count per node; the total is the sum of all nodes; merging two replicas takes the element-wise MAX, which is always conflict-free. CRDTs solve the "write conflict" problem in eventually consistent systems by ensuring any two valid states can be merged into a correct result, making distributed coordination unnecessary for those operations.

---
---

# TOPIC 2: Availability vs Consistency

---

## 1. The Core Tension

This topic builds directly on the CAP Theorem (Databases notes) and Consistency Models (Topic 1), adding practical depth on HOW to make this trade-off in real system design.

```
THE FUNDAMENTAL QUESTION EVERY DISTRIBUTED SYSTEM MUST ANSWER:
"When something goes wrong (network partition, node failure,
high latency between replicas), what do we sacrifice?"

OPTION A: SACRIFICE AVAILABILITY
  During the problem period: REFUSE TO SERVE REQUESTS.
  Return errors until consistency can be guaranteed.
  "Better to show an error than to show wrong data."

OPTION B: SACRIFICE CONSISTENCY
  During the problem period: SERVE STALE/POTENTIALLY INCORRECT DATA.
  Return best-effort answers from whatever data is available.
  "Better to show something (possibly stale) than an error."

THERE IS NO "HAVE BOTH DURING A PARTITION" OPTION.
This is the mathematical guarantee of the CAP theorem.
```

---

## 2. Mapping to Real Business Requirements

```
THE RIGHT CHOICE DEPENDS ON YOUR DOMAIN:

FINANCIAL TRANSACTIONS (bank balance, payments, ledger):
  PREFER: Consistency over availability.
  Rationale: showing a user their balance as ₹50,000 when it's
  actually ₹0 (because a debit hasn't replicated yet) could
  lead to an overdraft. Wrong data causes REAL FINANCIAL HARM.
  A brief "service unavailable" is annoying but recoverable.
  Systems: PostgreSQL with synchronous replication, Google Spanner,
           etcd for distributed coordination.

INVENTORY "LAST UNIT" IN E-COMMERCE:
  PREFER: Consistency (strong enough to prevent overselling).
  Rationale: selling the last unit of an item to 10 customers
  simultaneously (because each read a stale count of "1")
  means 9 customers will be disappointed and you'll owe refunds.
  Solution: linearizable check on "units remaining" for the
  FINAL unit, even if earlier inventory checks are eventual.

SOCIAL MEDIA LIKE COUNTS:
  PREFER: Availability over consistency.
  Rationale: if a post shows "10,432 likes" when the exact count
  is "10,437" for a few seconds — NOBODY CARES. The slight
  inaccuracy has zero business impact. Being unavailable
  ("error loading likes") is actually more disruptive to UX.
  Systems: Redis counters with async replication, Cassandra counters.

DNS RESOLUTION:
  PREFER: Availability (high) with eventual consistency.
  Rationale: DNS returning a slightly-stale IP (from TTL caching)
  is far better than DNS being unavailable. Most DNS lookups can
  tolerate some staleness. DNS is AP by design.

PRODUCT CATALOG (descriptions, images, prices for most items):
  PREFER: Availability with eventual consistency.
  Rationale: if a product description is 10 seconds stale, a
  shopper sees information that was accurate moments ago. The
  catalog being DOWN is much worse than it being slightly stale.
  Exception: PRICE (especially for flash sales) may need stronger
  consistency to prevent customer service issues.
```

---

## 3. The PACELC Framework — Beyond CAP

CAP only addresses partition scenarios. Most of the time there IS no partition — PACELC extends the analysis to normal operation too (recall Databases notes — CAP Theorem topic introduced PACELC briefly; here's the full treatment):

```
PACELC:

P → PARTITION SCENARIO: choose A (Availability) or C (Consistency)
  (This is the original CAP theorem)

E → ELSE (no partition, normal operation): choose L (Latency) or C (Consistency)

WHY THE "ELSE" MATTERS:
Even without a partition, achieving STRONG CONSISTENCY requires
coordination (waiting for replicas to acknowledge). This adds LATENCY.
If you don't need strong consistency, you can serve reads faster from
any replica without waiting for confirmation.

PACELC CLASSIFICATIONS:

PA/EL — Cassandra, DynamoDB (default):
  During partition: Available (serve reads/writes, may be stale)
  Normal operation: Low Latency (don't wait for all replicas)
  → Maximum performance, eventual consistency

PC/EC — Spanner, etcd, ZooKeeper:
  During partition: Consistent (refuse unavailable data)
  Normal operation: Consistent (always coordinate)
  → Maximum correctness, higher latency

PA/EC — MongoDB (configurable):
  During partition: Available
  Normal operation: Consistent (reads from primary by default)
  → A middle ground: fast writes with stale-read protection

PC/EL — Rare but possible: refuse during partition,
  but relax consistency for low latency in normal ops.
```

---

## 4. Tunable Consistency — The Practical Middle Ground

```
Many production systems don't make a binary choice — they offer
TUNABLE CONSISTENCY so each operation chooses its trade-off:

CASSANDRA (W + R > N = strong consistency):

N = 3 (total replicas)

Operation: Write payment record
  W = ALL (must write to 3/3 before ACK) — strong, but slowest
  If any replica down: write FAILS (availability sacrificed)

Operation: Read "is this product in stock?" (approx OK)
  R = ONE (read from 1 replica, fastest)
  May return stale but user just wants to know "roughly available?"
  → Eventually consistent, but very fast

Operation: Read "can I buy the last unit?" (must be exact)
  R = QUORUM (read from 2/3) — strong if W=QUORUM also
  W + R = 2+2 = 4 > 3 = N → STRONG CONSISTENCY guaranteed
  → Correct but slightly slower

THIS IS PRACTICAL WISDOM:
Most applications have a MIX of operations with different
consistency requirements. Tunable consistency lets you apply
the RIGHT strength to each operation rather than accepting
a one-size-fits-all decision for the entire system.
```

---

## 5. Practical Patterns for Managing Consistency Trade-offs

### Pattern 1: Read-Your-Own-Writes

```
PROBLEM: User updates their profile picture. They're redirected
to their profile page. The page reads from a replica that
hasn't received the update yet → they see their old picture.
Confusing UX even if the system is "working correctly."

SOLUTION: After a write, ROUTE THE SAME USER'S reads to the
PRIMARY (or the replica that received the write) for a brief
window (e.g., 1 minute after any write).

Implementation:
  - Store a "last write timestamp" in the user's session
  - If current time - last write < 60 seconds: read from primary
  - Otherwise: read from any replica
  - The user always sees their own writes immediately.

Used by: Amazon (my orders list), Twitter/X (tweet just posted),
         Instagram (photo just uploaded appears immediately)
```

### Pattern 2: Monotonic Reads

```
PROBLEM: User reloads a page. First reload: sees new data (v=10).
Second reload (hits a DIFFERENT, more-lagged replica): sees old
data (v=7). The data appears to "go backwards" — very confusing.

SOLUTION: Sticky sessions for reads — route the same user to
the same replica for the duration of a session. Monotonic reads
guarantee: once you've seen version N, you'll never see < N.

Implementation:
  - Hash the user session ID to a specific replica
  - Or use a read-after-timestamp approach (send the timestamp
    of the last-read value; replica serves only if it has data
    AT LEAST as recent as that timestamp; otherwise, defer)
```

### Pattern 3: Bounded Staleness

```
Instead of "eventually (no time bound)" consistent, set a
MAXIMUM STALENESS WINDOW: "data may be stale, but NEVER more
than 5 seconds stale."

Implementation: Track replication lag actively. If a replica's
lag exceeds 5 seconds, stop sending reads to it (route to a
fresher replica or the primary).

Azure Cosmos DB "Bounded Staleness" consistency level:
Guarantees reads are at most K versions behind or T time behind
(user configures K and T based on their tolerance).
```

---

## 6. Real-World Usage

**Facebook's TAO (The Associations and Objects):** Facebook's social graph storage uses a consistency model they call "read-your-writes with eventual consistency for others." When you post a status update, YOU see it immediately (read-your-writes). Your friends may see it a few seconds later (eventual). This is a deliberate choice — their scale makes linearizability prohibitively expensive, but the user experience only strictly requires you see your own actions immediately.

**Amazon DynamoDB:** Uses eventual consistency by default for performance and availability. Engineers must explicitly opt-in to strongly consistent reads for operations where stale data would cause problems (like reading a counter before incrementing it in an atomic operation). The separation is explicit in the API — a good design choice that forces engineers to consciously decide.

**Stripe (payments):** Uses strong consistency for all payment operations. Every transaction goes through a linearizable primary database. The latency cost (~10-20ms extra) is acceptable because the alternative — a payment that appears successful but isn't, or a double charge — is catastrophic for a financial services company.

---

## 7. Interview Quick-Fire Answers

**Q: How do you decide between consistency and availability for a feature?**
A: Ask: "What's the cost of returning WRONG data vs returning NO data?" If wrong data causes financial loss, security violations, or severely broken UX (user charged twice, inventory oversold) — choose consistency. If wrong data just means slightly stale information that users won't notice or care about (social counts, product descriptions, recommendations) — choose availability. Most large systems use BOTH: strong consistency for critical operations (payments, inventory last-unit), eventual consistency for everything else, with tunable consistency APIs to express this per-operation.

**Q: What is PACELC and how does it extend CAP?**
A: CAP only addresses the partition scenario — when a network fault occurs, choose Availability or Consistency. PACELC extends this: "if Partition, choose A or C; Else (during normal operation), choose Latency or Consistency." Even without partitions, strong consistency requires coordination across replicas, adding latency. If you're willing to accept eventual consistency, you get lower latency (serve from any replica without waiting). PA/EL systems (Cassandra, DynamoDB) optimize for latency in normal operation and availability during partitions. PC/EC systems (Spanner, etcd) always coordinate for correctness.

---
---

# TOPIC 3: Fault Tolerance

---

## 1. What Problem Does Fault Tolerance Solve?

In distributed systems at scale, FAILURES ARE CERTAIN. Hardware breaks. Networks partition. Software has bugs. Data centers lose power. The question isn't IF failures will happen — it's designing systems that CONTINUE WORKING CORRECTLY when they do.

```
FAILURES AT SCALE — THE NUMBERS:

Google's data centers run ~1 MILLION servers.
Annual hard drive failure rate: ~2-5%
→ 20,000-50,000 drive failures per YEAR
→ ~55-137 drive failures PER DAY
→ ~2-6 drive failures PER HOUR

At AWS, any given large instance fleet (10,000+ EC2 instances):
→ Several instances fail EVERY HOUR (hardware, hypervisor issues)

LESSON: With enough machines, SOMETHING is ALWAYS broken.
Systems must be designed to treat hardware failure as a NORMAL
OPERATING CONDITION, not an exceptional event.

"A reliable distributed system is one that delivers correct
results despite some of its components failing."
— Reliable enough that users DON'T NOTICE the failures.
```

---

## 2. Categories of Failures

```
1. HARDWARE FAILURES:
   - Disk failure (most common): ~2-5% annual failure rate
   - Server crash: power supply, memory ECC errors, CPU failure
   - Network card failure, cable cut, switch failure
   MITIGATION: RAID, replication, redundant hardware, graceful failover

2. SOFTWARE FAILURES:
   - Application bugs (null pointer, off-by-one, race condition)
   - Memory leaks (service degrades over hours/days)
   - Deadlocks (service freezes, stops responding)
   - Bad deployments (new code breaks existing functionality)
   MITIGATION: Testing, canary deployments, health checks, automatic restart

3. NETWORK FAILURES:
   - Network partition: two parts of the cluster can't talk
   - Packet loss: elevated error rates without full partition
   - Increased latency: network congestion, routing changes
   - DNS failures: services can't resolve each other
   MITIGATION: Retries with backoff, circuit breakers, timeouts, redundant paths

4. EXTERNAL DEPENDENCY FAILURES:
   - Third-party API is down (payment processor, SMS gateway)
   - Cloud provider partial outage (one AWS region, one AZ)
   - Database cluster degraded (primary failed, failover in progress)
   MITIGATION: Bulkheads (Microservices notes), fallbacks, SLAs with vendors

5. BYZANTINE FAILURES (the hardest):
   - A component fails in an ARBITRARY, UNPREDICTABLE way
   - Could send wrong data, claim to be something it isn't
   - Relevant for: untrusted nodes (blockchain), adversarial scenarios
   MITIGATION: Byzantine Fault Tolerant (BFT) consensus — complex,
   rarely needed in trusted data center environments
```

---

## 3. Fault Tolerance Strategies — The Toolkit

### Redundancy — Eliminate Single Points of Failure

```
SPOF (Single Point of Failure): Any component whose failure
causes the ENTIRE SYSTEM to fail. Must be eliminated for
production systems serving real users.

COMMON SPOFs AND THEIR FIXES:

SPOF: Single database server
FIX: Primary + replica(s), automatic failover (Patroni, RDS Multi-AZ)

SPOF: Single load balancer
FIX: Two load balancers in active-passive (VRRP/keepalived) or
     active-active behind DNS/Anycast (cloud LBs handle this natively)

SPOF: Single availability zone (entire AZ goes down — REAL outage type!)
FIX: Deploy across 3 AZs minimum. Each AZ = a separate physical data
     center with independent power, cooling, networking.

SPOF: Single region (rare but happens — AWS us-east-1 incidents)
FIX: Multi-region active-active or active-passive deployment
     (Geo-distribution topic, Scalability notes)

SPOF: Single Kafka broker
FIX: Kafka cluster (minimum 3 brokers, replication factor 3)

N+1 REDUNDANCY: Have N copies you need, plus 1 extra spare.
  "I need 5 servers to handle peak load → run 6 normally so if
  one fails, the remaining 5 can handle the load."
  More expensive (always paying for idle capacity) but ensures
  failures don't immediately degrade service.
```

### Timeouts — Don't Wait Forever

```
EVERY NETWORK CALL MUST HAVE A TIMEOUT.
Without a timeout, a call to a slow/unresponsive service
blocks the calling thread FOREVER — consuming resources and
eventually cascading (recall Bulkhead pattern, Microservices notes).

SETTING THE RIGHT TIMEOUT:
Too SHORT: Frequent false timeouts → unnecessary errors for
           operations that would have succeeded with 1 more second.
Too LONG: Slow failures consume resources for a long time before
          giving up → cascades still happen, just slower.

RULE: Timeout = (p99 latency of the operation) × 1.5-2x
If payment-service's p99 latency is 500ms: timeout = 750ms-1s.
Monitor p99 actively (recall RED metrics, DevOps notes!) and
adjust timeout as the service's performance changes.

TIMEOUT HIERARCHY (layered timeouts):
  Client-facing request: 5 second total timeout (user expectation)
  → Service A call to B: 2 second timeout
  → B's call to DB: 500ms timeout
  → B's call to C: 1 second timeout

  If C takes 1.5s → B's call fails at 1s → A's call fails at 2s →
  User gets error at 2s (not 5s) — faster failure, fewer wasted resources.
```

### Retries — Transient Failure Recovery

```
MANY FAILURES ARE TRANSIENT: a brief network blip, a server
restart mid-request, a brief overload. Retrying after a brief
pause often succeeds without any manual intervention.

RETRY WITH EXPONENTIAL BACKOFF AND JITTER:

Attempt 1: fails → wait 100ms
Attempt 2: fails → wait 200ms
Attempt 3: fails → wait 400ms
Attempt 4: fails → wait 800ms (+ jitter)
Attempt 5: fail permanently → surface error to caller

WITHOUT JITTER: all clients retry at the SAME TIME → "retry storm"
amplifies the original overload → makes things WORSE.

WITH JITTER: random offset added to each wait time:
wait = base_delay × (2^attempt) + random(0, base_delay)
→ clients retry at staggered times → load is distributed.

ONLY RETRY IDEMPOTENT OPERATIONS:
Retrying GET, HEAD, PUT (idempotent) = safe.
Retrying POST without idempotency keys = risk of double-action!
(e.g., retrying "create order" without idempotency could create
2 orders — covered fully in Topic 5: Idempotency)

DO NOT RETRY:
- 4xx errors (client-side error — retrying won't help)
- Non-idempotent operations without idempotency keys
- When the circuit breaker is OPEN (service is down, retrying wastes time)
```

### Graceful Degradation — Partial Functionality Over None

```
PRINCIPLE: When a component fails, deliver REDUCED but STILL USEFUL
service rather than complete failure.

EXAMPLES:

Netflix recommendations service goes down:
  WRONG: Show blank home page → user leaves.
  RIGHT: Show "Top 10 in India" (a pre-cached static list, no
         personalization) → user can still browse and watch.

Payment service is degraded (high latency):
  WRONG: Fail the entire checkout page.
  RIGHT: Complete the order, QUEUE the payment (async processing),
         show user "Your order is confirmed, payment processing."
         Payment completes when the service recovers.

Search service is slow:
  WRONG: Search page returns error.
  RIGHT: Return cached results from 1 hour ago with a banner
         "showing recent results — search may not be current."

IMPLEMENTATION: Feature flags for each degradable feature.
  If payment_service.circuit_breaker.is_open():
      return async_payment_fallback()  # queue it
  If recommendation_service.failed():
      return cached_top_trending_list()  # static fallback
  If search_service.p99_latency > threshold:
      return stale_search_results()  # tolerable degradation
```

### Health Checks and Self-Healing

```
HEALTH CHECKS: Continuous verification that services are
functioning correctly (recall K8s liveness/readiness probes
from DevOps notes — same concept system-wide).

LIVENESS: "Is this process still running?"
  Check: can I reach it at all? HTTP 200 from /health?
  Failure action: RESTART the process.

READINESS: "Is this process ready to handle traffic?"
  Check: can it connect to its dependencies? DB connection alive?
  Failure action: REMOVE from load balancer rotation (but don't
  restart — the process is healthy but temporarily unavailable).

SELF-HEALING PATTERNS:
Auto-restart: systemd/K8s restarts crashed processes automatically.
Auto-replace: K8s reschedules pods to healthy nodes if a node fails.
Auto-scale-out: if remaining instances are overloaded after a failure,
  auto-scale (recall Auto-scaling, Scalability notes) adds capacity.
Auto-failover: database primary fails → replica automatically promoted
  (Patroni, AWS RDS Multi-AZ, MongoDB replica sets).

THE GOAL: MTTR (Mean Time To Recovery) measured in SECONDS, not hours.
Automated self-healing eliminates the need for an on-call engineer
to be woken at 3am for every individual component failure.
```

---

## 4. Failure Mode Analysis — FMEA for System Design

```
FMEA (Failure Mode and Effects Analysis): For each component,
ask: "What if THIS fails? What's the impact? What's the mitigation?"

EXAMPLE: Payment processing system

Component: Payment gateway API connection
Failure mode: Gateway API returns 5xx errors
Impact: ALL payments fail globally — revenue stops
Probability: Low (gateway has 99.9% SLA)
Mitigation:
  1. Retry with exponential backoff (catches transient 5xx)
  2. Circuit breaker (stops hammering a dead gateway)
  3. Fallback to secondary payment gateway if primary is down
  4. Queue payments for async retry if both gateways down
  5. Alert on-call engineer if queue exceeds 5 minutes depth

Component: Redis session cache
Failure mode: Redis cluster entirely unavailable
Impact: All users logged out (session lookup fails)
Probability: Medium (Redis is in-memory, can OOM)
Mitigation:
  1. Redis Sentinel / Redis Cluster for HA (3 nodes minimum)
  2. Fallback: validate session from JWT if Redis is down
     (JWT is self-contained — no Redis lookup needed if we allow it)
  3. Alert and auto-restart on OOM

THIS EXERCISE FORCES:
  - Identifying EVERY dependency
  - Rating each by probability × impact (risk matrix)
  - Defining clear mitigation strategies BEFORE the failure happens
  - Validating mitigations with failure injection (Chaos Engineering — Topic 7!)
```

---

## 5. Real-World Usage

**Google (Site Reliability Engineering):** Google's SRE book codified many fault tolerance principles. They use "error budgets" (recall SLO topic, Topic 6) — if a service is consuming its error budget too fast (too many failures), deployments are paused until reliability improves. This makes fault tolerance a quantified, team-managed property, not an afterthought.

**Amazon's "Cell-Based Architecture":** Amazon decomposes large services into CELLS — isolated, independent units (like bulkheads for an entire service). A failure in one cell affects only users in that cell, not globally. Cells use separate resources (databases, caches, compute) so a database issue in Cell 7 doesn't affect Cell 12. At Amazon's scale, this contains the blast radius of any failure to a small fraction of users.

**Netflix (Chaos Monkey and Resilience):** Netflix runs Chaos Monkey in PRODUCTION, killing random instances at random times during business hours. This forces every team to build services that survive instance failure — because if they don't handle it in dev/staging, they WILL have to handle it in production. The culture is: "instances will die, design for it."

---

## 6. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Retry storm amplifies outage     │ All clients retry simultaneously  │ Exponential backoff WITH jitter;  │
│ (retrying makes it worse)        │ on failures → traffic 10x the     │ circuit breaker stops retries     │
│                                  │ normal load hits already-         │ when service is clearly down;     │
│                                  │ stressed service                   │ retry budget (max N retries total)│
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Timeout too long → threads        │ Timeout set to 30s for an op      │ Set timeouts based on p99 latency │
│ accumulate waiting → OOM          │ that normally takes 200ms;        │ × 1.5-2x; monitor timeout rates;  │
│                                  │ slow service fills thread pool     │ combine with Bulkhead pattern     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Cascading failure: A→B→C chain    │ C is slow → B's threads blocked   │ Timeouts at every hop; circuit    │
│ — C slow takes down A and B too   │ waiting for C → A's threads       │ breaker between B and C;          │
│                                  │ blocked waiting for B → all fail  │ Bulkhead isolation per downstream;│
│                                  │                                  │ async where response not needed   │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ "Hidden SPOF": shared library     │ All services use a common         │ Audit shared dependencies;        │
│ or config service failure         │ library that has a bug, or all    │ feature flags stored locally too; │
│ takes down ALL services at once   │ poll the same config service      │ circuit break config service      │
│                                  │ that goes down                    │ reads with cached fallback        │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 7. Interview Quick-Fire Answers

**Q: What is fault tolerance and how is it different from high availability?**
A: Fault tolerance is the ability of a system to CONTINUE OPERATING CORRECTLY when some components fail — it's about correctness during failure. High availability (HA) is the ability of a system to REMAIN ACCESSIBLE during failures — it's about uptime percentage. A fault-tolerant system might degrade gracefully (serve fewer features, use stale data) while remaining available. A highly-available system minimizes the time it's completely unreachable. They're related but distinct: HA focuses on "can users reach the system?", fault tolerance on "does the system behave correctly when parts of it break?"

**Q: How do you prevent cascading failures in a microservices system?**
A: Layer multiple resilience patterns: (1) Timeouts on every inter-service call prevent indefinite blocking — a slow downstream fails fast after N seconds. (2) Retries with exponential backoff and jitter handle transient failures without amplifying load. (3) Circuit breakers stop calling a service that's clearly down, preventing wasted retries and resource exhaustion. (4) Bulkheads (separate thread pools per downstream) ensure one slow service only affects its own resource pool, not all other downstream calls. (5) Graceful degradation returns reduced but functional responses when a dependency fails. All five working together create defense in depth against cascades.

**Q: What is the difference between liveness and readiness probes (and why does it matter for fault tolerance)?**
A: A liveness probe answers "is this process alive and not deadlocked?" — failure triggers a RESTART (the process is broken and needs to be replaced). A readiness probe answers "is this process ready to handle traffic?" — failure removes it from the load balancer without restarting (the process is healthy but temporarily unable to serve, perhaps because a dependency is down or it's still warming up). For fault tolerance, using both correctly prevents two failure modes: liveness prevents zombie processes that are alive but deadlocked from receiving traffic, while readiness prevents healthy-but-temporarily-unavailable processes from being unnecessarily killed and causing additional disruption.


---
---

# TOPIC 4: Distributed Transactions

---

## 1. What Problem Do Distributed Transactions Solve?

In a monolith with a single database, a transaction that touches multiple tables either ALL succeeds or ALL rolls back — ACID guarantees this (recall ACID topic, Databases notes). But in microservices, each service owns its own database. A business operation that spans multiple services must update multiple independent databases — and there's no shared ACID transaction across them.

```
THE DISTRIBUTED TRANSACTION PROBLEM:

"Place Order" operation must:
  1. Order Service DB:    INSERT order (status=CONFIRMED)
  2. Inventory Service DB: UPDATE inventory SET stock=stock-1
  3. Payment Service DB:   INSERT payment (status=CHARGED)

WHAT IF step 2 succeeds but step 3 FAILS?
  → Order is confirmed ✅
  → Inventory is decremented ✅
  → Payment NOT charged ❌
  → DATA INCONSISTENCY: customer got the product for free!

WHAT IF step 1 succeeds but the network fails before step 2?
  → Order confirmed but inventory not reserved
  → Customer might receive an out-of-stock item

In a monolith: one DB transaction wraps all three → either all
commit or all roll back. Clean, automatic, guaranteed by the DB.

In microservices: THREE SEPARATE DB TRANSACTIONS on THREE SEPARATE
SERVERS. No shared transaction coordinator. This is the fundamental
challenge of distributed transactions.
```

---

## 2. Two-Phase Commit (2PC) — The Classic Solution

### How It Works

```
2PC introduces a COORDINATOR that orchestrates a two-phase protocol
across all participating services (PARTICIPANTS).

PHASE 1: PREPARE ("Can you commit?")

         ┌────────────────────────────────────────────┐
         │              COORDINATOR                    │
         └──────┬─────────────────┬───────────────────┘
                │ PREPARE         │ PREPARE
                ▼                 ▼
         ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
         │  Order DB     │  │ Inventory DB  │  │  Payment DB  │
         │ "Can commit?" │  │ "Can commit?" │  │ "Can commit?"│
         │ LOCKS the row │  │ LOCKS the row │  │ LOCKS the row│
         │ → VOTE: YES   │  │ → VOTE: YES   │  │ → VOTE: YES  │
         └──────────────┘  └──────────────┘  └──────────────┘
                │                 │                │
                └─────────────────┴────────────────┘
                              All voted YES

PHASE 2: COMMIT ("Now commit!")

         ┌────────────────────────────────────────────┐
         │  COORDINATOR: All YES → send COMMIT to all  │
         └──────┬─────────────────┬───────────────────┘
                │ COMMIT          │ COMMIT
                ▼                 ▼
         ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
         │  Order DB     │  │ Inventory DB  │  │  Payment DB  │
         │  COMMITS      │  │  COMMITS      │  │  COMMITS     │
         │  releases lock│  │  releases lock│  │  releases lock│
         └──────────────┘  └──────────────┘  └──────────────┘

IF ANY PARTICIPANT VOTES NO in Phase 1:
  Coordinator sends ROLLBACK to ALL participants instead of COMMIT.
  All participants roll back their changes. Clean state restored.
```

### Why 2PC Is Problematic at Scale

```
PROBLEM 1: COORDINATOR IS A SINGLE POINT OF FAILURE
If the coordinator crashes AFTER sending PREPARE but BEFORE
sending COMMIT:
  → All participants have LOCKED rows waiting for a commit decision
  → They wait... and wait... forever (or until timeout)
  → Rows remain LOCKED, blocking all other access
  → This is called "in-doubt" state — extremely difficult to recover
  
Recovery requires manual intervention or a sophisticated recovery
protocol — complex and error-prone.

PROBLEM 2: LOCKS HELD FOR ENTIRE PROTOCOL DURATION
During Phase 1 + Phase 2, each database holds LOCKS on the
rows being modified. Phase 1 alone requires a network round
trip to EVERY participant.
  3 participants × 100ms RTT = 300ms of locks held
  Under high concurrency: massive contention, throughput drops.

PROBLEM 3: BLOCKING DURING NETWORK PARTITIONS
If the coordinator can't reach all participants (network issue):
  Coordinator must ABORT (can't proceed without unanimous YES)
  OR wait until all are reachable.
  Either way: increased latency or failed transactions.

PROBLEM 4: DOESN'T SCALE TO MANY SERVICES
With 10 services: 10 network round trips in phase 1, 10 more in
phase 2 — all while holding locks on all 10 databases. The
coordination overhead becomes the bottleneck, not the actual work.

CONCLUSION: 2PC is appropriate for:
  - Small number of participants (2-3 databases tightly coupled)
  - Systems where strict ACID across services is truly required
  - Short-lived transactions where lock duration is bounded
  - NOT appropriate as the default pattern in microservices
```

---

## 3. The Saga Pattern — The Microservices Alternative

### Core Philosophy

```
INSTEAD OF: "Either everything succeeds atomically, or nothing does"

SAGA SAYS: "A long-running business transaction is a SEQUENCE of
local transactions. If any local transaction fails, we COMPENSATE
by executing UNDO operations for the previously completed steps."

SAGA is:
✅ Eventually consistent (not ACID-consistent)
✅ No distributed locks
✅ Each service uses its own local ACID transaction
✅ Scales well — services are only loosely coordinated via events
❌ NOT immediately consistent — brief windows of partial state exist
❌ Compensation (undo) logic must be explicitly implemented
❌ No easy "rollback" — each step needs a reverse operation designed

ANALOGY: Booking a vacation with separate hotel, flight, car rentals.
  Hotel booked ✅ → Flight booked ✅ → Car rental FAILED ❌
  You don't "rollback" your hotel and flight atomically —
  you CANCEL them individually (compensation transactions).
  For a brief moment, you had a hotel and flight but no car.
  The eventual state is either fully booked or fully cancelled.
```

### Choreography-Based Saga (Event-Driven)

```
No central coordinator. Each service listens to events and decides
its next action independently.

Order Service: Creates order (local txn) → Publishes "OrderCreated"
                                                      │
                                              ┌───────┘
                                              ▼
Inventory Service: Hears "OrderCreated"
  → Reserves inventory (local txn)
  → Publishes "InventoryReserved" ─────────────────────┐
  OR if out of stock:                                    │
  → Publishes "InventoryFailed" ──────────────────────┐ │
                                                       │ │
                                              ┌────────┘ │
                                              ▼           ▼
                                           FAILURE     SUCCESS
Order Service hears "InventoryFailed":  Payment Service hears "InventoryReserved":
  → COMPENSATE: cancel order (local txn)  → Charges payment (local txn)
  → Publishes "OrderCancelled"            → Publishes "PaymentProcessed"
                                                      │
                                              ┌───────┘
                                              ▼
                                     Order Service hears "PaymentProcessed":
                                       → Updates order to CONFIRMED (local txn)
                                       → Done!

PROS of Choreography:
✅ No central coordinator → no SPOF
✅ Loose coupling — services don't know about each other, only events
✅ Easy to add new participants (subscribe to existing events)

CONS of Choreography:
❌ Hard to trace the overall flow — which events triggered which actions?
❌ Testing is complex (must simulate entire event chain)
❌ Risk of "event spaghetti" — implicit dependencies through event chains
   that are hard to reason about as the system grows
```

### Orchestration-Based Saga (Central Coordinator)

```
A SAGA ORCHESTRATOR service explicitly coordinates all steps.
Similar to a workflow engine (connects to your LangGraph experience!).

              ┌──────────────────────────────────────────┐
              │          SAGA ORCHESTRATOR                │
              │                                          │
              │  Step 1: Tell OrderService: create order  │
              │  ← Success → Step 2                      │
              │                                          │
              │  Step 2: Tell InventoryService: reserve  │
              │  ← Success → Step 3                      │
              │  ← InventoryFailed → Compensate Step 1   │
              │                                          │
              │  Step 3: Tell PaymentService: charge     │
              │  ← Success → DONE                        │
              │  ← PaymentFailed → Compensate Steps 1+2  │
              └──────────────────────────────────────────┘

ORCHESTRATOR STATE MACHINE:
Each saga instance has a state stored in a DB:
{saga_id: "abc", step: "AWAITING_PAYMENT", order_id: 456,
 inventory_reserved: true, payment_attempted: false}

If the orchestrator crashes and restarts → reads state from DB →
resumes from where it left off → no work is lost.

PROS of Orchestration:
✅ Easy to visualize and trace the business flow
✅ Saga state is explicit and auditable
✅ Easier to test individual steps in isolation
✅ Handles compensation logic centrally (not scattered across services)

CONS of Orchestration:
❌ Orchestrator is a new service to build, deploy, and maintain
❌ Orchestrator knows about all participants → some coupling
❌ Can become a bottleneck if many sagas run concurrently
   (though each saga is a separate instance, so usually fine)

WHEN TO CHOOSE WHICH:
- Few services, simple flow → Choreography (simpler operationally)
- Many services, complex flow, need visibility → Orchestration
- Financial / regulatory processes needing audit trail → Orchestration
  (state machine provides complete audit trail per transaction)
```

---

## 4. Compensating Transactions — Designing Undos

```
Every Saga step needs a COMPENSATING TRANSACTION — the "undo."

STEP                      COMPENSATION
─────────────────────────────────────────────────────────────
Create order              Cancel order
Reserve inventory         Release inventory reservation
Charge payment            Issue refund
Send confirmation email   Send cancellation email (can't "unsend")
Create a database record  Delete the record (if no other dependents)

COMPENSATION DESIGN RULES:

1. COMPENSATIONS MUST BE IDEMPOTENT:
   If the compensating transaction is executed TWICE (due to retry),
   the result must be the same as executing it once.
   "Release reservation for order 456" → if already released, no-op.

2. SOME OPERATIONS CAN'T BE COMPENSATED:
   Sending an email is permanent — you can't unsend it.
   Compensation: send a DIFFERENT email ("correction: your order
   was cancelled") — a "semantic undo" that reverses the business
   effect, even if the technical action can't be undone.

3. COMPENSATIONS MAY ALSO FAIL:
   What if the compensation fails? ("Refund failed because payment
   processor is down.") → Need a retry mechanism + monitoring for
   "stuck sagas" → Manual intervention if compensation keeps failing.
   Production: saga state tracks compensation attempts; alerts on
   sagas stuck in compensating state for > N minutes.

4. ORDER MATTERS — COMPENSATE IN REVERSE:
   Completed: Step 1 → Step 2 → Step 3 (fails)
   Compensate: Step 2 first → then Step 1
   Always compensate in reverse order of completion.
```

---

## 5. Real-World Usage

**Uber (Trip Booking Saga):** A trip booking spans: driver matching, ETA computation, pricing calculation, and payment authorization. Each step runs in a separate service. Uber uses an orchestration-based saga for the overall trip lifecycle — if any step fails (e.g., payment authorization fails), compensating transactions release the driver reservation and cancel the trip, with the appropriate rollback notifications sent to both driver and rider.

**Stripe (payment workflows):** Stripe's payment processing is itself a saga — charge authorization → capture → settlement, each step potentially failing. Their "Radar" fraud detection can decline a charge mid-saga. Compensating transactions: releases on payment holds, refunds on already-captured charges. Stripe uses orchestration with explicit state machines, and every saga state transition is auditable in their system.

**Amazon Order Processing:** The classic Saga example. An Amazon order triggers dozens of steps: payment authorization, inventory reservation, fulfillment selection, shipping label creation. Each is a separate service. The entire flow is an orchestrated saga — if payment capture fails after inventory was reserved, the reservation is released. Amazon has handled these sagas at hundreds of millions of orders per year.

---

## 6. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Saga stuck: compensation fails   │ Payment service down when        │ Retry compensation with backoff;  │
│ → order confirmed but payment    │ trying to issue refund           │ alert on stuck sagas (> N min     │
│ not reversed → data inconsistent │                                  │ in compensating state); manual    │
│                                  │                                  │ intervention workflow for ops     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Duplicate saga execution: same   │ Orchestrator retried after a     │ Idempotency keys on each saga     │
│ order charged twice              │ timeout, but original had already│ step; exactly-once semantics      │
│                                  │ succeeded                         │ via idempotency (full topic next) │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Choreography "spaghetti":        │ Too many services publishing/     │ Limit choreography to simple,     │
│ impossible to trace why an       │ consuming events; implicit        │ shallow event chains; for complex │
│ order failed                     │ coupling through event chains     │ flows, switch to orchestration    │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ 2PC coordinator crash leaves     │ Coordinator down before           │ Avoid 2PC for microservices;      │
│ participants with locked rows     │ sending commit/rollback           │ use Saga instead; if 2PC needed,  │
│ indefinitely                     │                                  │ use timeout-based release of      │
│                                  │                                  │ in-doubt transactions             │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 7. Interview Quick-Fire Answers

**Q: Why is 2PC problematic in microservices and what's the alternative?**
A: 2PC requires all participants to hold database locks from the Prepare phase until the coordinator sends Commit — across multiple network round trips. Under high concurrency, this creates severe contention. More critically, if the coordinator crashes between phases, participants are left in "in-doubt" state with locks held indefinitely. The Saga pattern is the microservices alternative: each service executes its own local ACID transaction, and if any step fails, compensating transactions (explicitly designed "undo" operations) reverse the completed steps. Sagas are eventually consistent (brief windows of partial state exist) but scalable, lock-free, and resilient to individual service failures.

**Q: When would you choose choreography vs orchestration for a Saga?**
A: Choreography (services react to events independently, no central coordinator) works well for simple, shallow event chains with few participants — it's more loosely coupled and easier to add new subscribers. It becomes problematic as the number of services and steps grows, creating "event spaghetti" that's hard to trace and debug. Orchestration (central coordinator manages the saga state machine) is better for complex, multi-step business processes that need explicit visibility, auditability, and easier compensation logic management. For financial/regulatory use cases (your BFSI context), orchestration is typically preferred because the explicit state machine provides a complete audit trail.

---
---

# TOPIC 5: Idempotency

---

## 1. What Problem Does Idempotency Solve?

In distributed systems, networks are unreliable. A client sends a request, the server processes it, but the network drops the response. The client doesn't know if the request succeeded or not — so it RETRIES. If the server isn't idempotent, the retry causes a DUPLICATE OPERATION.

```
THE DUPLICATE REQUEST PROBLEM:

Client sends: POST /payments {amount: ₹5000, account: "ACC123"}
Server receives, processes payment ✅
Server sends response: 200 OK {payment_id: "PAY456"}
NETWORK DROPS THE RESPONSE ❌
Client receives: connection timeout

Client's perspective: "Did it succeed? I don't know! I'll retry."
Client sends: POST /payments {amount: ₹5000, account: "ACC123"}

NON-IDEMPOTENT SERVER: processes the payment AGAIN →
  Account charged TWICE: ₹10,000 total ❌
  Customer files dispute, bank issues chargeback → financial loss
  + regulatory issue

IDEMPOTENT SERVER: recognizes this is a DUPLICATE of a previous
  request → returns the SAME RESULT as before (payment_id: "PAY456")
  without processing again ✅ Account charged only ONCE.

IDEMPOTENCY DEFINITION: An operation is idempotent if performing
it MULTIPLE TIMES produces the SAME RESULT as performing it ONCE.
f(f(x)) = f(x) — applying the function repeatedly has the same
effect as applying it once.
```

---

## 2. HTTP Methods and Idempotency (Recap + Depth)

```
GET:    Idempotent ✅ Safe ✅ (reads don't change state)
PUT:    Idempotent ✅ (PUT user/123 {name:"Yash"} twice → same result)
DELETE: Idempotent ✅ (DELETE /users/123 twice → user 123 is gone;
        second call returns 404, not an error in idempotent design)
HEAD:   Idempotent ✅ Safe ✅
OPTIONS: Idempotent ✅ Safe ✅

POST:   NOT idempotent ❌ (by design — each POST creates a new resource)
PATCH:  Usually NOT idempotent ❌ (PATCH /counter with {increment: 1}
        applied twice → counter incremented by 2, not 1)

THE CRITICAL INSIGHT:
"Idempotent by default" HTTP methods (GET, PUT, DELETE) can be
safely retried without special handling.

POST and non-idempotent PATCH REQUIRE additional mechanisms to
make them safe to retry — this is what IDEMPOTENCY KEYS solve.
```

---

## 3. Idempotency Keys — Making POST Safe to Retry

### The Mechanism

```
CLIENT generates a UNIQUE ID (UUID) for each DISTINCT operation.
This ID (Idempotency Key) is sent in the request header.

CLIENT generates: idempotency_key = "IK-" + uuid4()
                = "IK-a3f8b2c1-5e89-4f23-b4d1-abc123xyz456"
(This key is generated ONCE for this operation, stored client-side.
On retry, the SAME key is sent — not a new one!)

REQUEST 1 (original):
POST /api/payments HTTP/1.1
Idempotency-Key: IK-a3f8b2c1-5e89-4f23-b4d1-abc123xyz456
Content-Type: application/json

{"amount": 5000, "currency": "INR", "account": "ACC123"}

SERVER SIDE — PROCESSING FLOW:
1. Extract Idempotency-Key from header
2. Check Redis/DB: has this key been seen before?
   NO → Process the payment
       → Store result: {key: "IK-a3f8b2c1...", result: {payment_id:"PAY456", status:"success"}}
       → Return 200 OK {payment_id: "PAY456"}

REQUEST 2 (retry, same key):
POST /api/payments HTTP/1.1
Idempotency-Key: IK-a3f8b2c1-5e89-4f23-b4d1-abc123xyz456

SERVER SIDE:
1. Extract Idempotency-Key
2. Check Redis: has "IK-a3f8b2c1..." been seen before?
   YES → Return CACHED result: 200 OK {payment_id: "PAY456"}
         WITHOUT processing payment again ✅

REQUEST 3 (NEW operation — client generates NEW key):
POST /api/payments HTTP/1.1
Idempotency-Key: IK-b7d1e4f9-6c12-4a87-c3e5-def789uvw012
                  ↑ DIFFERENT UUID = NEW distinct operation

SERVER: Not in cache → Process as new payment ✅
```

### Implementation Details

```
IDEMPOTENCY KEY STORAGE:
Redis is the natural store (fast, TTL-based expiry built-in):

SETNX idempotency_key:{key} "PROCESSING"   # atomic "set if not exists"
→ Returns 1 (key was set → first time, process)
→ Returns 0 (key already exists → duplicate, use cached result)

After processing:
SET idempotency_key:{key} {json_result} EX 86400  # store result, 24h TTL

On retry: GET idempotency_key:{key} → return cached result

HANDLING THE "PROCESSING" STATE:
What if the server crashes after setting "PROCESSING" but
before storing the result? Client retries → finds "PROCESSING"
→ Must wait (server is processing) or treat as a new attempt?

ROBUST APPROACH:
1. SETNX key "PROCESSING" with a short TTL (e.g., 30 seconds)
2. Process the request
3. Atomically update key to the actual result (longer TTL, e.g., 24h)
4. If key expired before processing finished → client retry safe to treat as new

WHAT TO INCLUDE IN THE KEY'S SCOPE:
Key should be scoped to: (user_id, operation_type, key)
NOT just the key alone — prevents key collisions across users.

Storage format: idempotency:{user_id}:{idempotency_key}
e.g.:          idempotency:user_123:IK-a3f8b2c1...
```

---

## 4. Stripe's Idempotency — The Gold Standard

```
Stripe's API is the best real-world example of idempotency done right.
Their idempotency implementation covers:

1. HEADER NAME: Idempotency-Key (capital I, capital K — standard)
2. KEY FORMAT: Any string, typically a UUID v4
3. KEY LIFESPAN: 24 hours (same key returns cached result for 24h)
4. SCOPE: Per secret API key + per idempotency key string
5. REQUEST BODY VALIDATION: If the same idempotency key is used with
   a DIFFERENT request body → return 422 Unprocessable Entity
   "This idempotency key was used with a different request payload."
   (Prevents accidentally reusing a key for a different amount!)

STRIPE RETRY LOGIC (what clients should do):
1. Generate idempotency key for this payment (once, store it)
2. POST to Stripe with the key
3. If response: use it
4. If timeout/503: wait (exponential backoff + jitter) → retry with SAME KEY
5. If 200 on retry: idempotency key caught the duplicate — don't charge again
6. If the result differs (shouldn't happen with proper key scope): alert!

THIS PATTERN IS EXACTLY WHAT ALL PAYMENT APIs SHOULD IMPLEMENT.
Relevant to your BFSI interview prep: any payment system design
answer should include idempotency key handling for POST operations.
```

---

## 5. Database-Level Idempotency

```
Beyond API-level idempotency keys, the DATABASE LAYER can enforce
idempotency via unique constraints:

EXAMPLE: Creating an order with a unique order reference number
INSERT INTO orders (reference_no, user_id, amount)
VALUES ('ORD-2026-ABC123', 123, 5000)
ON CONFLICT (reference_no) DO NOTHING;  -- PostgreSQL syntax

→ First insert: creates the order ✅
→ Retry insert with same reference_no: silently no-op ✅
→ No duplicate order, no exception thrown

EXAMPLE: Exactly-once event processing (Kafka consumer):
CREATE TABLE processed_events (
    event_id VARCHAR(64) PRIMARY KEY,   -- unique constraint
    processed_at TIMESTAMP
);

On receiving a Kafka event:
INSERT INTO processed_events (event_id, processed_at)
VALUES ('event_abc123', NOW())
ON CONFLICT (event_id) DO NOTHING;

IF inserted (rows affected = 1) → process the event → commit
IF not inserted (rows affected = 0) → this event was already processed → skip

This gives EXACTLY-ONCE processing semantics for Kafka consumers
without needing Kafka's own exactly-once transactions!
(Recall: Messaging notes — at-least-once delivery + idempotent
consumer = effectively exactly-once in terms of business impact)
```

---

## 6. Idempotency Patterns by Layer

```
┌──────────────────────┬────────────────────────────────────────────────────────────┐
│ Layer                 │ Idempotency Pattern                                          │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ HTTP API               │ Idempotency-Key header; server stores (key → result) in    │
│                       │ Redis with TTL; return cached result on duplicate key       │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ Database               │ Unique constraints (ON CONFLICT DO NOTHING); upsert        │
│                       │ patterns (INSERT ... ON CONFLICT UPDATE); natural keys       │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ Message consumers       │ Track processed event IDs in DB; before processing check  │
│ (Kafka, SQS)           │ "has this event_id been processed?"; skip if yes           │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ Saga steps              │ Each step checks "has this saga_id already completed this  │
│                       │ step?" before executing; stores step completion in state DB  │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ External API calls      │ Always send an operation_id / reference number; external   │
│ (calling third-party)  │ API stores it; on retry with same reference, same response │
└──────────────────────┴────────────────────────────────────────────────────────────┘
```

---

## 7. Real-World Usage

**Stripe:** Every mutating API endpoint accepts an Idempotency-Key. Their entire payment infrastructure is built on the assumption that clients WILL retry and MUST be safe to retry. The 24-hour key retention window is intentional — network issues sometimes last hours, and you need to be able to retry for that entire window safely.

**AWS SQS + Lambda:** SQS delivers messages at-least-once. Lambda functions consuming SQS must be idempotent — because SQS may deliver the same message more than once (on Lambda timeout, Lambda error, or SQS visibility timeout expiry). AWS recommends using a DynamoDB table to track processed message IDs, exactly the database-level idempotency pattern described above.

**UPI / RBI (BFSI relevance):** UPI transactions use a "transaction reference number" (RRN) as the idempotency key. If a payment appears to fail (network timeout at the user's end), the same RRN is used for the status check / retry, and the payment system knows not to process it twice. RBI guidelines explicitly require payment systems to handle duplicate transaction prevention — idempotency is a regulatory requirement, not just a best practice.

---

## 8. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Client generates a NEW key on    │ Bug in client code: key         │ Client MUST store the key before  │
│ each retry → duplicate charges   │ regenerated per attempt          │ sending; treat key generation as  │
│                                  │                                  │ part of the operation, not retry  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Idempotency key expires before   │ TTL too short (1 hour); client   │ TTL must exceed max retry window; │
│ final retry → duplicate processed│ retries at hour 2 after a long   │ Stripe uses 24h; for long-running │
│                                  │ network outage                   │ ops consider even longer windows  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ TOCTOU race: two concurrent       │ Two identical requests arrive    │ Use atomic "set if not exists"    │
│ requests both check "not         │ simultaneously; both check Redis  │ (SETNX in Redis) not GET-then-SET;│
│ processed" and both process       │ at same moment, both get "miss"  │ atomic DB upsert (ON CONFLICT);   │
│                                  │                                  │ distributed lock around processing │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Wrong key scope: user A uses      │ Key scoped globally, not per     │ ALWAYS scope key to user/API key: │
│ same key as user B → user B       │ user; user A's result returned   │ store as {user_id}:{idempotency_key}│
│ gets user A's payment result      │ for user B's request             │ never just {idempotency_key} alone │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 9. Interview Quick-Fire Answers

**Q: What is idempotency and why is it critical in distributed systems?**
A: An operation is idempotent if performing it multiple times produces the same result as performing it once. In distributed systems, networks are unreliable — responses get lost, timeouts occur, and clients MUST retry. Without idempotency, retrying a "charge payment" operation could charge the customer twice. With idempotency (implemented via an Idempotency-Key that clients generate once and reuse on retries, stored server-side with the result in Redis), the server recognizes duplicates and returns the cached result without re-processing. This makes at-least-once delivery safe — the system effectively achieves exactly-once business semantics.

**Q: How would you implement idempotency for a payment API?**
A: Client generates a UUID before sending the first request and includes it as `Idempotency-Key: {uuid}` in the header — same key on every retry, new key for a new distinct payment. Server-side: atomically set `idempotency:{user_id}:{key}` in Redis to "PROCESSING" (using SETNX). If set: process the payment, store the full response in Redis with a 24-hour TTL. If not set (key already exists): return the cached response immediately without re-processing. The Redis TTL must exceed the maximum retry window — Stripe uses 24 hours. The key must be scoped per user to prevent cross-user key collision.

---
---

# TOPIC 6: SLA / SLO / SLI

---

## 1. What Problem Do These Concepts Solve?

"The system should be reliable." This vague statement can't be measured, can't be enforced, and can't guide engineering decisions. SLI, SLO, and SLA provide a QUANTITATIVE FRAMEWORK for reliability — turning vague goals into measurable targets with clear consequences.

```
THE VAGUE VS SPECIFIC PROBLEM:

VAGUE: "The API should be fast and available."
  → What does "fast" mean? 100ms? 1 second?
  → What does "available" mean? 99%? 99.999%?
  → How do you know if you're meeting this?
  → When do you sound the alarm? When do you NOT sound it?
  → What's the consequence if you fail?

SPECIFIC (using SLI/SLO/SLA):
  "p99 API latency < 300ms (SLI: p99 latency, SLO: < 300ms),
   99.9% of the time measured over 30-day rolling windows (SLO),
   with SLA credit of 10% of monthly invoice if we miss by >0.5%."
  → Measurable, actionable, contractual.
```

---

## 2. The Three Definitions

```
SLI (Service Level INDICATOR):
A METRIC that measures some aspect of service behavior.
The "what we measure."

Examples:
  - Request latency: time from request received to response sent
  - Error rate: fraction of requests returning 5xx errors
  - Availability: fraction of successful health checks
  - Throughput: requests served per second
  - Durability: fraction of data correctly stored and retrievable
  - Freshness: time since data was last updated

SLO (Service Level OBJECTIVE):
A TARGET VALUE (or range) for an SLI over a time window.
The "what we aim for."

Examples:
  - "p99 request latency < 300ms measured over any 30-day window"
  - "Error rate < 0.1% over any 7-day rolling window"
  - "99.9% of health checks pass over any 30-day calendar month"

SLA (Service Level AGREEMENT):
A CONTRACT between a provider and a customer specifying minimum
acceptable SLOs and CONSEQUENCES for missing them.
The "what we promise and what happens if we fail."

Examples:
  - "99.9% monthly uptime. If actual uptime < 99.9%, customer
    receives a service credit of 10% of that month's invoice.
    If uptime < 99%, credit is 25% of invoice."
  - AWS's SLA for EC2: 99.99% availability. Below: credit up to 100%

RELATIONSHIP:
SLI is the measurement.
SLO is the internal target you set FOR YOURSELF.
SLA is the external commitment you make to CUSTOMERS.

Typically: SLO is STRICTER than SLA.
  SLA: "We'll maintain 99.9% uptime" (external promise)
  SLO: "Our internal target is 99.95% uptime"
  (buffer between SLO and SLA gives you room to detect
   problems and fix them BEFORE violating the customer SLA)
```

---

## 3. Error Budgets — The Most Important Derived Concept

```
ERROR BUDGET = 100% - SLO_target

If SLO = 99.9% availability over 30 days:
  Error budget = 0.1% of 30 days
  = 30 days × 24 hours × 60 min × 0.001
  = 43.2 minutes of ALLOWED downtime per month

If SLO = 99.99% availability:
  Error budget = 0.01%
  = 30 × 24 × 60 × 0.0001 = 4.32 minutes of allowed downtime

THE POWER OF ERROR BUDGETS:

1. QUANTIFIED RISK TOLERANCE:
   "Can we deploy this risky change?"
   → How much error budget have we burned this month?
   → Burned 30 minutes, budget is 43.2 minutes → 13 minutes left.
   → Risky deploy could burn 20 minutes if it fails → NO, too risky.
   → Budget is fresh (0 minutes burned) → deploy with careful monitoring.

2. RECONCILES PRODUCT VELOCITY AND RELIABILITY:
   Product team: "Ship features fast!"
   SRE team: "Protect reliability!"
   Error budget: neutral arbiter.
   "You want to ship 20 features this week. Your error budget is
   almost gone. Deploys are PAUSED until next month's budget resets.
   OR: Make the deploys more reliable so they don't burn budget."

3. REMOVES SUBJECTIVITY:
   Instead of "should we be more careful?" (subjective),
   it's "we have 5 minutes of error budget left" (objective).
   Decisions become data-driven, not political.
```

---

## 4. Choosing the Right SLIs

```
NOT EVERY METRIC MAKES A GOOD SLI.

GOOD SLI CHARACTERISTICS:
✅ Directly reflects USER EXPERIENCE
   (users notice when this degrades)
✅ Measurable without excessive noise
✅ Has a clear target that represents "working"

FOUR GOLDEN SIGNALS AS SLIs (from Google SRE book):
  1. LATENCY: How long requests take (p50, p95, p99)
     Reason: users notice slow responses immediately.
     SLO: "p99 latency < 300ms for 99.5% of requests"
  
  2. TRAFFIC: Request rate (queries per second)
     Reason: baseline — is load normal? Too high? Too low?
     SLO: Traffic itself rarely an SLO target, but used in
          context ("latency SLO at this traffic level")
  
  3. ERRORS: Fraction of requests returning errors
     Reason: errors are the clearest signal of user impact.
     SLO: "Error rate < 0.1%"
  
  4. SATURATION: How "full" the service is (CPU%, memory%, queue depth)
     Reason: predicts problems before they impact users.
     SLO: "CPU utilization < 70% (90th percentile over 5 min)"

AVOID SLIs THAT:
❌ Measure INTERNAL IMPLEMENTATION details users don't care about
   ("database query cache hit rate" — irrelevant if latency is fine)
❌ Are trivially always met
   ("average latency < 5 seconds" — anything < 5s passes even if p99 = 4.9s)
❌ Are impossible to attribute to your service
   ("end-to-end latency including user's browser" — too many variables)
```

---

## 5. SLO Windows — Rolling vs Calendar

```
CALENDAR WINDOW (Jan 1 - Jan 31, Feb 1 - Feb 28...):
  Simple to explain. Aligns with billing cycles.
  Problem: "sawtooth" effect — error budget fully resets at midnight
  on the 1st. A bad month can have terrible reliability in day 1-29
  but if everything is perfect on day 30, the month might still pass!
  A massive outage on Jan 31 rolls into Feb 1 budget cleanly —
  masking its impact.

30-DAY ROLLING WINDOW (continuously recalculated):
  "Availability measured over any 30-day period ending now."
  Smoother — an outage today reduces your budget for the next 30 days.
  No artificial "reset" that hides persistent problems.
  More complex to calculate but more representative.

RECOMMENDATION (Google SRE):
  Use rolling windows for SLOs that track continuous reliability.
  Calendar windows for billing/SLA purposes (customer credits).
  Report BOTH internally — rolling to catch trends, calendar for billing.

MULTI-WINDOW SLOs (modern approach):
  Burn rate alerts on BOTH long (1-hour) and short (5-minute) windows:
  "If 5% of error budget burned in last 1 hour (slow burn, major issue)
   OR 100% of hourly budget burned in 5 minutes (fast burn, acute crisis)
   → PAGE on-call engineer."
  This catches both gradual degradation and sudden outages.
```

---

## 6. Common SLO Examples by System Type

```
WEB APPLICATION:
  SLI: Fraction of HTTP requests with status < 500 and latency < 500ms
  SLO: 99.9% of requests meet this criteria over rolling 30-day window
  Error budget: 43.2 minutes/month of "bad" request window

STORAGE SYSTEM (durability focus):
  SLI: Fraction of write requests durably persisted and retrievable
  SLO: 99.999999% (9 nines) of writes durably stored
  (Very high target — even a tiny durability failure loses customer data)

PAYMENT API:
  SLI: Fraction of payment attempts succeeding within 3 seconds
  SLO: 99.95% success rate, 99.9% within 3 seconds
  (Slightly lower target acknowledges third-party card network failures)

MESSAGE QUEUE (delivery focus):
  SLI: Fraction of messages delivered within 30 seconds of enqueue
  SLO: 99.9% of messages delivered within 30 seconds

BATCH PROCESSING JOB:
  SLI: Fraction of daily reports produced on-time (before 6am)
  SLO: 99% of daily reports produced on-time each month
```

---

## 7. Real-World Usage

**Google SRE (the originator):** Google formalized SLI/SLO/SLA and error budgets in their SRE book (2016). Every Google service has defined SLOs, and the decision to allow or block deployments depends on error budget availability. Teams that consistently burn their error budget have deployment velocity reduced until they invest in reliability.

**AWS SLAs:** Amazon publishes SLAs for all major services. EC2: 99.99% for any region. S3: 99.9%. These are EXTERNAL SLAs with credit consequences. Internally, AWS's SRE teams maintain much stricter SLOs (often 99.999%) with more sophisticated monitoring.

**Stripe's SLAs:** Stripe publishes an uptime SLA for their payments API (99.99%) with specific credit tiers. Their internal SLOs are stricter — they aim for 99.999% (five nines = 5.3 minutes downtime per year) because any payment API downtime directly impacts their customers' revenue.

---

## 8. Interview Quick-Fire Answers

**Q: What is the difference between SLI, SLO, and SLA?**
A: An SLI (Service Level Indicator) is a specific metric measuring service behavior — e.g., p99 request latency, error rate, or availability (fraction of successful health checks). An SLO (Service Level Objective) is an internal target for an SLI — e.g., "p99 latency < 300ms for 99.9% of requests over any 30-day window." An SLA (Service Level Agreement) is an external contract with customers defining minimum acceptable SLOs and the consequences (credits, penalties) for missing them. SLOs are typically set stricter than SLAs to give teams a warning window before violating customer commitments.

**Q: What is an error budget and how does it help balance reliability and velocity?**
A: An error budget is the allowed amount of unreliability derived from an SLO — e.g., a 99.9% availability SLO gives 43.2 minutes/month of allowed downtime. It makes reliability quantitative and operational: when budget is plentiful, teams can take risks (risky deploys, experiments) and invest in features. When budget is nearly exhausted, risky deployments are paused and reliability work takes priority. This removes subjective arguments about "should we be more careful?" and replaces them with "we have X minutes of error budget left" — a data-driven, jointly agreed trade-off between product velocity and reliability.

---
---

# TOPIC 7: Chaos Engineering

---

## 1. What Problem Does Chaos Engineering Solve?

Most systems are designed to work in NORMAL CONDITIONS. But production has ABNORMAL CONDITIONS — server failures, network latency spikes, disk fills up, dependencies go down. These failures are often discovered by USERS before engineers know about them.

```
THE CONFIDENCE PROBLEM:

You've built a distributed system with:
  - Primary-replica database failover
  - Circuit breakers on all service calls
  - Retries with exponential backoff
  - Bulkhead pattern isolation
  - Health checks and auto-restart

"Great! Our system is resilient!"

But has any of this ACTUALLY BEEN TESTED WITH REAL FAILURES?

Reality: many "resilience" features don't work when you actually need them:
  - Failover takes 5 minutes (config error) → SLO violated
  - Circuit breaker configured incorrectly → never opens
  - Retry logic has a bug → retries synchronously, no backoff
  - Health check passes even when DB is down (wrong check)

These bugs only appear during actual failures — which is the WORST
time to discover them.

CHAOS ENGINEERING: Deliberately inject CONTROLLED FAILURES into
production (or production-like environments) to:
1. VERIFY that resilience mechanisms actually work
2. DISCOVER weaknesses BEFORE real failures expose them to users
3. BUILD CONFIDENCE that the system can handle real failure scenarios
```

**Analogy:** Fire drills. A building's fire suppression system, evacuation routes, and emergency procedures are only as reliable as they've been tested. A fire drill at 2pm on a Tuesday is uncomfortable but controlled. A real fire at 3am on a Saturday discovers every gap in the plan — with much worse consequences.

---

## 2. The Principles of Chaos Engineering

```
FIVE PRINCIPLES (from Principles of Chaos Engineering, Netflix):

1. BUILD A HYPOTHESIS AROUND STEADY STATE BEHAVIOR:
   Define "normal" before you break things.
   "Steady state: p99 latency < 200ms, error rate < 0.1%"
   This is your baseline to compare against.

2. VARY REAL-WORLD EVENTS:
   Test realistic failure scenarios:
   - Instance crashes (most common)
   - Network latency injection (add 200ms to all calls to service X)
   - Disk fills up
   - CPU spike (100% utilization on a node)
   - Dependency failure (take down service Y)
   - Data corruption (return malformed responses from service Z)

3. RUN EXPERIMENTS IN PRODUCTION:
   This is the controversial but important principle.
   Staging environments don't have production traffic, production
   data volumes, or production dependencies — they often don't
   CATCH failures that production will experience.
   START SMALL (1% of traffic / 1 instance), controlled blast radius.
   MONITOR actively during the experiment.

4. AUTOMATE EXPERIMENTS TO RUN CONTINUOUSLY:
   One-off experiments provide one-time insights.
   CONTINUOUS chaos testing (running on every deployment or weekly
   in production) ensures NEW CODE doesn't break resilience.
   "Did this week's deploy accidentally make us less resilient?"

5. MINIMIZE BLAST RADIUS:
   Never run experiments that could take down the whole system.
   Start with: 1 instance, 1% of traffic, canary environment.
   Gradually increase scope as confidence grows.
   ALWAYS have a kill switch to stop the experiment immediately.
```

---

## 3. The Chaos Engineering Process

```
STEP 1: DEFINE STEADY STATE
  What does "healthy" look like numerically?
  SLI baseline: p99 latency 180ms, error rate 0.05%, checkout
  success rate 99.2%.
  Monitor these BEFORE, DURING, and AFTER the experiment.

STEP 2: FORM A HYPOTHESIS
  "IF we terminate 1 of 5 order-service instances randomly,
   THEN the load balancer will detect the failure within 10 seconds,
   remaining instances will absorb the load, and steady-state
   metrics will return to baseline within 30 seconds."

  Be specific: what failure, what expected recovery, what time frame.

STEP 3: DESIGN THE EXPERIMENT
  - WHAT to break: one order-service pod via kubectl delete pod
  - WHEN: Tuesday 2pm (low-traffic, team available to respond)
  - BLAST RADIUS: one pod out of five (20% capacity removed)
  - MONITORING: all dashboards open, alert thresholds set
  - KILL SWITCH: immediately restore if error rate > 5% or latency > 2s

STEP 4: RUN AND OBSERVE
  Execute the failure injection.
  Record: what actually happened?
  - Load balancer detection time: was it 10 seconds or 2 minutes?
  - Traffic rerouting: smooth or dropped connections?
  - Latency during failover: stayed < 200ms or spiked?

STEP 5: VERIFY (OR FALSIFY) THE HYPOTHESIS
  Actual result: detection took 35 seconds (hypothesis: 10 seconds)
  Latency spiked to 850ms during failover (hypothesis: stays < 200ms)
  Hypothesis FALSIFIED → resilience is WEAKER than assumed.

STEP 6: IMPROVE AND REPEAT
  Root cause: health check interval was 30s (too long) + LB needed
  2 failed checks before removing instance = 60s total.
  Fix: reduce health check interval to 5s, unhealthy threshold to 2.
  Re-run experiment → now detection is 10 seconds ✅
  HYPOTHESIS VERIFIED after fix.

THIS LOOP CONTINUOUSLY IMPROVES RESILIENCE.
```

---

## 4. Chaos Engineering Tools

```
NETFLIX CHAOS MONKEY (original, open source):
  Randomly terminates EC2 instances in Auto Scaling Groups
  during business hours (9am-3pm Pacific).
  Why business hours? Engineers are awake and available to fix
  issues as they're discovered. Not 3am when everyone is asleep.
  Language: Java/Scala, runs as a service continuously.

NETFLIX SIMIAN ARMY (extended family, now mostly deprecated):
  Chaos Gorilla: terminates an entire AWS Availability Zone
  Chaos Kong: simulates an entire AWS Region failure (extreme!)
  Latency Monkey: injects artificial delays in RESTful calls
  Doctor Monkey: finds instances with poor health and disables them

GREMLIN (commercial SaaS, the production standard):
  GUI for designing and running chaos experiments.
  Failure types: resource attacks (CPU, memory, disk, I/O),
  network attacks (latency, packet loss, DNS failure, blackhole),
  state attacks (process kill, time travel, shutdown).
  Safety controls: blast radius limiting, automatic rollback.
  Used by: Expedia, Mailchimp, JP Morgan, many large enterprises.

CHAOS TOOLKIT (open source Python):
  CLI tool for designing experiments as JSON/YAML configuration.
  Integrates with Kubernetes, AWS, Azure, GCP.
  Example experiment: kill a Kubernetes pod, verify service recovers.

LITMUS CHAOS (Kubernetes-native, CNCF project):
  Chaos experiments defined as Kubernetes CRDs (ChaosEngines).
  Integrates with Prometheus for steady-state hypothesis testing.
  Built for cloud-native/K8s environments.

TOXIPROXY (network fault injection for testing):
  A proxy that simulates network conditions in test environments:
  - Latency (add 200ms to all connections)
  - Packet loss (drop 10% of packets)
  - Bandwidth throttling
  - Connection timeout simulation
  Use in INTEGRATION/STAGING environments before production chaos.
```

---

## 5. From Chaos Monkey to Game Days

```
GAME DAY: A planned, team-wide chaos engineering exercise where
engineers deliberately trigger large-scale failure scenarios and
practice the response.

FORMAT:
  - Chosen failure scenario announced to operations team
    (but NOT to the application teams — they respond as if real)
  - Incident commander coordinates
  - Metrics monitored in real-time
  - Recovery time measured
  - Post-incident review: what went well, what didn't

SCENARIOS TEAMS ACTUALLY RUN:
  "Regional failover": redirect all traffic from us-east-1 to us-west-2
  "Database primary failure": kill the database primary, observe failover
  "Dependency timeout injection": make payment service respond in 30s
  "Cascade prevention": verify circuit breakers actually open
  "Cache cold start": flush all Redis caches simultaneously

OUTCOMES:
  - Runbooks (recovery procedures) are tested and gaps identified
  - On-call engineers practice incident response in a safe environment
  - Institutional knowledge gaps are discovered ("only Bob knows how
    to do this failover — and Bob is on vacation")
  - Monitoring gaps identified ("we didn't have an alert for this!")

Google, Netflix, Amazon, and LinkedIn all run regular game days
for their most critical services. It's considered an SRE best
practice for services with strict availability SLOs.
```

---

## 6. Real-World Usage

**Netflix (Chaos Monkey in production since 2011):** The original chaos engineering company. Started Chaos Monkey to force engineers to build resilient services. Expanded to the full Simian Army. Netflix's famous "Chaos Kong" simulates an entire AWS region going down — a level of testing that most companies never attempt but that Netflix considers essential given their global scale and availability requirements.

**LinkedIn (Project Waterbear):** LinkedIn's chaos engineering program systematically tests all production systems. They discovered through chaos experiments that several of their "resilient" systems had subtle bugs that only appeared under specific failure conditions — bugs that would have caused hours-long outages if discovered during a real incident instead of a controlled experiment.

**AWS (Operational Readiness Reviews + GameDay):** AWS runs "Operational Readiness Reviews" (ORRs) before launching new services, which include chaos experiments. Their "GameDay" program runs across service teams, deliberately breaking infrastructure to test recovery procedures. AWS Route 53 was famously stress-tested with simulated failures before being offered as a production service.

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure in the Experiment       │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Chaos experiment causes a real   │ Blast radius miscalculated;     │ Start with single instance in non │
│ customer-impacting outage         │ no kill switch; ran during       │ peak hours; always have kill      │
│                                  │ peak traffic                     │ switch ready; monitor constantly;  │
│                                  │                                  │ never run during peak traffic     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Hypothesis proven but resilience │ System recovers during           │ Run experiments with REALISTIC    │
│ fails in actual outage           │ controlled experiment but not    │ traffic loads; test at peak hours  │
│                                  │ under real load conditions        │ (carefully); vary experiment      │
│                                  │                                  │ parameters (not just 1 failure     │
│                                  │                                  │ type at a time)                   │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Team stops running experiments   │ One experiment caused a bad       │ Culture shift: treat bad outcomes │
│ after a bad outcome               │ outcome; fear of repeating       │ as the POINT — you discovered a   │
│                                  │                                  │ weakness. Fix it, re-test.         │
│                                  │                                  │ Start smaller, lower blast radius  │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: What is chaos engineering and why do companies like Netflix run experiments in production?**
A: Chaos engineering is the practice of deliberately injecting controlled failures into production systems to verify that resilience mechanisms work as expected and to discover weaknesses before real failures expose them to users. Netflix runs chaos experiments in production because staging environments don't replicate production traffic patterns, data volumes, or dependency topology accurately — failures that matter often only appear at production scale. By running small, controlled failures during business hours (when engineers are available), Netflix discovers and fixes weaknesses proactively rather than reactively during 3am real incidents.

**Q: What is the difference between chaos testing in staging vs chaos engineering in production?**
A: Staging chaos testing verifies the code path exists (the circuit breaker CAN open) but doesn't guarantee it works correctly under production conditions (real traffic patterns, scale, data volumes, dependency chains). Production chaos engineering runs controlled failures against real traffic, verifying that the resilience mechanism actually protects users under real conditions. The blast radius is controlled (one instance, 1% of traffic) and there's always a kill switch, but the test is against the real system with real users — the only environment that fully reflects what actual failures will look like.

**Q: How does chaos engineering relate to SLOs?**
A: Chaos engineering is the practice that gives you CONFIDENCE that your SLOs are achievable when failures occur. You can set an SLO of 99.9% availability, but without testing failures, you don't know if your failover actually takes 10 seconds (as you assumed when calculating the error budget impact) or 10 minutes (which would violate the SLO for any single instance failure). Chaos experiments validate that the recovery time and error rate during failures are within your error budget assumptions. If an experiment shows your SLO would be violated by a common failure type, you have the opportunity to improve resilience before the failure occurs naturally.

---
---

# APPENDIX: Cross-Topic Quick Reference

## Master Comparison — All Consistency & Reliability Concepts

```
┌──────────────────────────┬───────────────────────────────────────────────────────────┐
│ Topic                     │ Core Question It Answers                                    │
├──────────────────────────┼───────────────────────────────────────────────────────────┤
│ Consistency Models         │ "After a write to one replica, what can another replica    │
│                           │ return — and what are the tradeoffs of each guarantee?"    │
│ Availability vs            │ "When something goes wrong, do we return wrong data or     │
│ Consistency                │ no data — and which is right for THIS use case?"           │
│ Fault Tolerance            │ "How do we keep working correctly when components fail,    │
│                           │ and how do we DESIGN for failure being a normal event?"    │
│ Distributed Transactions   │ "How do we maintain data consistency across multiple        │
│                           │ independent services with separate databases?"              │
│ Idempotency                │ "How do we make retries safe so a duplicated request       │
│                           │ never causes duplicate actions (double charges, etc.)?"    │
│ SLA / SLO / SLI            │ "How do we quantify reliability so we can measure it,       │
│                           │ target it, and make data-driven trade-off decisions?"      │
│ Chaos Engineering          │ "How do we VERIFY our resilience mechanisms actually        │
│                           │ work, by testing them with controlled real failures?"      │
└──────────────────────────┴───────────────────────────────────────────────────────────┘
```

## How All Seven Topics Interconnect

```
A COMPLETE RELIABILITY ARCHITECTURE:

1. DEFINE YOUR RELIABILITY TARGETS (Topic 6 — SLA/SLO/SLI):
   SLI: p99 latency < 300ms, error rate < 0.1%
   SLO: 99.9% of requests meet SLI over 30-day rolling window
   Error budget: 43.2 minutes/month of violations allowed
   SLA: 99.9% uptime commitment to customers with credit penalties

2. CHOOSE CONSISTENCY MODEL PER OPERATION (Topics 1 & 2):
   Payment balance reads: LINEARIZABLE (wrong data = financial harm)
   Product catalog reads: EVENTUAL (stale is fine, availability critical)
   User feed ordering: CAUSAL (replies must follow posts)
   Config service reads: SEQUENTIAL (consistent view per process)

3. BUILD FAULT-TOLERANT INFRASTRUCTURE (Topic 3):
   Redundancy: 3-AZ deployment, no SPOFs
   Timeouts: every service call has a deadline
   Retries: exponential backoff with jitter (Topic 5: idempotent!)
   Circuit breakers: prevent cascading failures
   Graceful degradation: partial service > complete failure
   Health checks: automatic self-healing via K8s liveness/readiness

4. MAKE ALL MUTATIONS IDEMPOTENT (Topic 5):
   Idempotency-Key on all POST endpoints
   Database unique constraints for natural idempotency
   Message consumer deduplication (processed_event_ids table)
   Saga steps idempotent (each checks "already completed?")

5. HANDLE CROSS-SERVICE DATA CONSISTENCY (Topic 4):
   Avoid 2PC — use Saga pattern instead
   Choreography for simple flows, orchestration for complex
   Design compensating transactions for each saga step
   Saga state machine provides audit trail (BFSI requirement!)

6. VERIFY EVERYTHING WORKS UNDER FAILURE (Topic 7):
   Define steady-state hypothesis (from SLOs in step 1)
   Run controlled chaos experiments:
     Kill one instance → verify recovery within SLO
     Inject network latency → verify circuit breakers open
     Kill downstream service → verify fallback works
   Automate and run continuously (especially after each deploy)

FEEDBACK LOOP:
Chaos experiments reveal gaps → improve fault tolerance →
re-verify against SLOs → adjust error budgets →
updated chaos experiments for new scenarios
```

## Final Study Tips

```
1. THE RELIABILITY VOCABULARY INTERVIEWERS TEST:
   Idempotency, Saga pattern, eventual consistency, error budget,
   linearizability, chaos engineering — these terms should roll
   off naturally with correct definitions AND examples.
   Practice saying "we use Saga with orchestration because our
   payment flow has 5 steps and we need auditability" not just
   "we use Saga" (the WHY matters as much as the WHAT).

2. CONNECT ACROSS ALL NOTES:
   - Consistency models → CAP theorem (Databases notes)
   - Saga pattern → EDA, choreography/orchestration (Messaging notes)
   - Idempotency → at-least-once delivery (Messaging notes)
   - Fault tolerance → Circuit breaker, Bulkhead (Microservices notes)
   - SLO/error budget → observability, alerting (DevOps notes)
   - Chaos engineering → health checks, K8s probes (DevOps notes)
   - Distributed transactions → ACID vs BASE (Databases notes)

3. THE BFSI/FINTECH ANGLE (relevant to your prep):
   CONSISTENCY: Always linearizable for account balances, payment
     status, and ledger entries. No eventual consistency for money.
   DISTRIBUTED TRANSACTIONS: Saga with orchestration is standard
     for multi-step payment workflows (authorization → capture →
     settlement). State machine provides RBI-required audit trail.
   IDEMPOTENCY: RBI/NPCI mandates transaction reference numbers
     (RRN) for UPI — this IS the idempotency key. Regulatory
     requirement, not just engineering best practice.
   SLOs: NPCI mandates UPI transaction processing within 30 seconds
     (or automatic reversal). This is a REGULATORY SLO — violation
     has regulatory consequences, not just customer credit.
   CHAOS ENGINEERING: RBI's "Business Continuity Planning" (BCP)
     circulars require banks to test disaster recovery scenarios.
     Chaos engineering is the engineering implementation of BCP
     testing. Banks must demonstrate their systems survive specific
     failure scenarios to maintain their banking license.

4. THE FIVE RESILIENCE PATTERNS — MEMORIZE AND CONNECT:
   Timeout → prevents indefinite hanging
   Retry (with idempotency!) → handles transient failures
   Circuit breaker → stops calling broken services
   Bulkhead → contains resource exhaustion
   Fallback → partial service > error
   These five, applied consistently, prevent cascading failures.
   ADDING: Saga + Idempotency + Chaos Engineering = complete
   reliability story that covers most interview questions.

5. FOR SYSTEM DESIGN INTERVIEWS:
   When designing ANY system, volunteer reliability info:
   "For consistency, payment status reads will use linearizable
   reads from the primary — we can't tolerate stale data here.
   Product catalog reads use eventual consistency from replicas —
   slightly stale data is acceptable for display, and the
   availability benefit is significant.
   All payment POST endpoints are idempotent via Idempotency-Key
   stored in Redis with 24-hour TTL. We'd set our payment API
   SLO at 99.95% with p99 latency < 500ms, leaving headroom
   above our 99.9% SLA commitment to customers."
   This level of detail in reliability design separates strong
   senior candidates from candidates who just describe the
   happy path architecture.
```
