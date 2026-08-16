# Dynamo: Amazon’s Highly Available Key-value Store

---

## 1. Executive Summary & Core Engineering Challenge

### 1.1 What real-world problem was Dynamo solving?

Dynamo was designed for Amazon services where availability was more important than strong consistency.
Amazon's platform consisted of hundreds of loosely coupled services running across tens of thousands of servers and many datacenters. At this scale, failures were considered normal rather than exceptional: servers, disks, network routes, and even entire datacenters could fail continuously.

The central requirement was therefore:

> **Build a storage system that remains writable and readable even when infrastructure is failing, while still meeting extremely strict latency requirements.**

The shopping-cart example captures the requirement perfectly:

```
Customer adds item
       │
       ▼
     Dynamo
       │
       ├── disk failure?       → still write
       ├── network failure?    → still write
       ├── server failure?     → still write
       └── datacenter failure? → still write

```

The paper explicitly says that a shopping cart must continue accepting additions/removals even during server and network failures.

---

### 1.2 Why not a traditional relational database?

The target Amazon services generally needed only:

* `get(key)`
* `put(key, value)`

They did **not** require:

* Joins
* Complex queries
* Relational schemas
* Multi-item operations
* General transactions

Objects were typically relatively small—usually less than **1 MB**.

The paper argues that using a relational database for these workloads introduced unnecessary functionality, operational cost, and limitations around scale and availability.

Examples of target services included:

* Shopping carts
* Best-seller lists
* Customer preferences
* Session management
* Sales rank
* Product catalog

---

### 1.3 The fundamental architectural decision

Dynamo deliberately chooses:

```
                  Availability
                       ▲
                       │
                       │
                       │
Consistency ◄──────────┼──────────► Performance
                       │
                       │
                       ▼
                    Cost

```

Rather than trying to provide maximum consistency, Dynamo gives application owners configurable control over:

* Availability
* Consistency
* Durability
* Performance
* Cost

> "Dynamo sacrifices consistency under certain failure scenarios to achieve high availability."

---

### 1.4 Scale and production constraints

The paper intentionally withholds some absolute production numbers for business reasons. It explicitly says that absolute request rates, datacenter latencies, outage lengths, and workloads are presented only as aggregate measures.

Key system parameters and scale metrics from the paper:

| Metric | Paper / Amazon Infrastructure Value |
| --- | --- |
| **Physical Scale** | Tens of thousands of servers, multiple worldwide datacenters |
| **Peak Customers** | Tens of millions |
| **Shopping Cart Service** | Tens of millions of requests/day (>3 million checkouts/day) |
| **Session Service** | Hundreds of thousands of concurrent sessions |
| **Target Object Size** | Usually < 1 MB |
| **Initial Instance Scale** | Hundreds of storage hosts |
| **Typical Replication Factor ($N$)** | 3 |
| **SLA Focus** | **99.9th percentile** (e.g., 300 ms at 500 QPS) |
| **Observed Latency** | ~200 ms at 99.9th percentile |
| **Overall Availability** | **99.9995%** successful requests |
| **Data-Loss Events** | None reported during production period |

---

### 1.5 Why the 99.9th percentile matters so much

This is one of the most important lessons in the paper. Amazon does not optimize for average latency or median latency; it focuses heavily on **99.9th percentile latency**.

An average can hide terrible experiences for a small but critical population of users. Because Amazon user-facing pages could depend on 150+ underlying services, every dependency needs a sufficiently tight SLA for the overall page to meet its own target latency.

> **Interview Lesson:**
> "I would optimize the tail, not just the average, because a slow dependency can violate the application's user-facing SLA even when its average latency looks excellent."

---

### 1.6 Core design philosophy

The system can essentially be summarized as:

$$\text{Consistent Hashing} + \text{Replication} + \text{Vector Clocks} + \text{Quorum R/W} + \text{Sloppy Quorum} + \text{Hinted Handoff} + \text{Merkle Trees} + \text{Gossip} + \text{Failure Detection} = \mathbf{Dynamo}$$

This combination—not one individual algorithm—is the paper's central contribution.

---

## 2. System Architecture & High-Level Design

### 2.1 High-level architecture

```
                         Client
                           │
                           ▼
                    Load Balancer
                           │
                           ▼
                 Any Dynamo Storage Node
                           │
                      Coordinator
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
          Replica 1    Replica 2    Replica 3
              │            │            │
              ▼            ▼            ▼
         Local Store   Local Store   Local Store
              │            │            │
              └────────────┼────────────┘
                           │
                    Read / Write result
                           │
                           ▼
                         Client

```

There is **no centralized master**. Every node participates as a peer. Dynamo is explicitly designed around symmetry, decentralization, and peer-to-peer coordination.

---

### 2.2 Node component architecture

Each Dynamo storage node contains three main layers implemented in Java:

```
┌──────────────────────────────────────┐
│             Dynamo Node              │
│                                      │
│  1. Request Coordination             │
│  2. Membership & Failure Detection   │
│  3. Local Persistence Engine         │
└──────────────────────────────────────┘

```

---

### 2.3 API

Dynamo intentionally exposes a very small API:

* `get(key)`: Returns one object, or multiple conflicting versions, plus a context containing metadata such as version information.
* `put(key, context, object)`: Stores a new version of the object. The context allows Dynamo to understand which previous version the client is updating.

---

### 2.4 Key hashing & consistent hashing

Dynamo treats the key and value as opaque byte arrays. The key is hashed using MD5 to produce a 128-bit identifier on a circular ring space.

```
                 A
             /       \
           G           B
           |           |
           F           C
             \       /
                E-D

```

Each node gets assigned positions on this ring. To route a key:

1. Hash the key into the ring.
2. Walk clockwise until reaching the first node.
3. That node becomes responsible for the key's range.

If a node (e.g., node $C$) disappears, only nearby ranges need to move. You don't need to redistribute the entire dataset.

---

### 2.5 Virtual nodes

Basic consistent hashing suffers from uneven key distribution and non-uniform server hardware capacities. Dynamo solves this using **virtual nodes (tokens)**.

```
Physical Node A
 ├── Token 1
 ├── Token 2
 ├── Token 3
 └── ...

```

**Benefits:**

* **Failure Redistribution:** If a node fails, its workload is split across many small ranges owned by multiple peers rather than dumped on a single successor.
* **Faster Bootstrapping:** A new node takes small pieces from many existing nodes.
* **Heterogeneous Hardware:** More powerful machines are assigned a higher number of virtual nodes.

---

### 2.6 Replication

Each key is replicated across $N$ physical nodes. The coordinator node stores the object locally and replicates it to the next $N-1$ physical nodes walking clockwise on the ring.

```
Ring: A → B → C → D → E → F
Key K → Coordinator B (N=3)
Replicas: B (Primary), C (Replica), D (Replica)

```

The set of nodes responsible for a key is called the **preference list**. The preference list contains more than $N$ nodes to bypass healthy nodes experiencing transient failures.

To survive catastrophic site failures, the preference list is constructed so replicas are spread across multiple physical datacenters.

---

### 2.7 Eventual consistency & vector clocks

Dynamo allows writes to complete asynchronously without synchronous confirmation across all $N$ replicas. Therefore, concurrent updates can lead to multiple divergent versions during partitions.

To capture causality, Dynamo uses **Vector Clocks**: a list of `(node, counter)` pairs associated with every version of an object.

```
Initial: D1 [(Sx,1)]
  │
  ▼
Update on same node: D2 [(Sx,2)]
  │
  ├──────────────────────────┐
  ▼                          ▼
Update by Sy: D3           Update by Sz: D4
[(Sx,2),(Sy,1)]            [(Sx,2),(Sz,1)]

```

In this case, $D_3$ and $D_4$ are concurrent—neither clock dominates the other. Dynamo retains both versions and pushes conflict resolution to the application level.

* **Syntactic Reconciliation:** Handled automatically by Dynamo when vector clocks show one version strictly dominates another (ancestor relationship).
* **Semantic Reconciliation:** Handled by application logic (e.g., merging shopping carts) when concurrent versions exist.
* **Truncation:** To prevent unbounded clock growth, Dynamo truncates vector clocks when they reach a size threshold (e.g., 10 entries).

---

### 2.8 Request routing & execution

Clients can route requests via two strategies:

1. **Generic Load Balancer:** Client $\rightarrow$ Load Balancer $\rightarrow$ Random Dynamo Node $\rightarrow$ Coordinator Node.
2. **Partition-Aware Client:** Client $\rightarrow$ Local Client Library $\rightarrow$ Direct Coordinator Node (Lower latency).

---

### 2.9 Quorum configurations ($N, R, W$)

Dynamo uses configurable quorum parameters:

* $N$: Number of replicas.
* $R$: Minimum required read responses.
* $W$: Minimum required write acknowledgements.

Setting $R + W > N$ yields a quorum-like property. However, Dynamo uses **Sloppy Quorums** to prioritize availability during failures:

```
Preference List: A (Down), B, C
N = 3, Required = 3

Actual Path:
  ├── B (Healthy)
  ├── C (Healthy)
  └── D (Temporary Substitute - Stores Hinted Replica)

```

#### Hinted Handoff

When Node $D$ accepts data meant for Node $A$, it holds a "hint" in its local store. Once Node $D$ detects Node $A$ is back online, it hands off the updated data and purges its local copy.

---

### 2.10 Anti-Entropy via Merkle Trees

To handle permanent failures or cases where hinted handoffs fail, Dynamo uses **Merkle trees** for replica synchronization:

```
             Root Hash
             /       \
          Hash       Hash
          / \        / \
         H1 H2      H3 H4

```

Replicas compare root hashes of key ranges. If the hashes match, no data is transferred. If they differ, nodes traverse down the tree to isolate and synchronize only the differing keys without scanning the entire dataset.

---

### 2.11 Partitioning strategy evolution

```
Strategy 1: Random tokens + Token-defined partitions
  └─ Caused expensive bootstrapping scans and Merkle tree recalculations.

Strategy 2: Fixed equal-sized partitions + Random tokens
  └─ Decoupled partition boundaries from replica placement.

Strategy 3: Equal-sized partitions + Controlled token assignment
  └─ Smooth token redistribution on joins/leaves, simplifying operations like archival.

```

---

### 2.12 Production optimizations

* **Pluggable Storage Engines:** Berkeley DB (BDB) TDS was used for most production instances (small objects), while MySQL was used for larger items.
* **Read Repair:** Coordinators asynchronously update stale replicas discovered during read operations.
* **Write Buffering:** An optional in-memory write buffer reduced 99.9th-percentile write latencies by $5\times$ during traffic spikes.
* **Background Admission Control:** Dynamo monitors disk latency, queue times, and lock contention to dynamically throttle background processes (anti-entropy, hinted handoff) so foreground SLA operations are protected.

---

## 3. Critical Engineering Trade-offs & Design Choices

| Trade-off | Option A | Option B | Dynamo Choice & Rationale |
| --- | --- | --- | --- |
| **Availability vs Consistency** | Reject operations during network partitions (PSSR/Strong) | Accept writes during partitions, handle divergent state later | **Option B:** Availability is paramount; dropping writes destroys revenue (e.g., shopping carts). |
| **Conflict Resolution Timing** | On Write (reject writes if conflicts can't be resolved) | On Read (allow writes, reconcile during read phase) | **Option B:** Keeps the write path extremely fast and always available. |
| **Conflict Resolution Layer** | System-Level (e.g., Last-Write-Wins based on timestamps) | Application-Level (Business logic semantic merges) | **Option B:** Prevents data loss (e.g., un-deleting items) by letting applications apply domain context. |
| **Quorum Type** | Strict Quorum (Must access exact $N$ primary nodes) | Sloppy Quorum + Hinted Handoff | **Option B:** Accepts writes on temporary substitute nodes to survive cluster faults. |
| **Vector Clock Metadata** | Unlimited Clock History (100% causal accuracy) | Truncated Clocks (Bounded metadata) | **Option B:** Bounds storage overhead with minimal real-world impacts on causal accuracy. |
| **Request Routing** | Server-side Load Balancing (Fewer client dependencies) | Partition-Aware Clients | **Option B (where possible):** Eliminates network hops, significantly cutting tail latency. |

---

### Client-Driven vs. Server-Driven Performance Data

According to production metrics presented in the paper, client-side routing dramatically reduces tail latency:

| Approach | P99.9 Read | P99.9 Write | Avg Read | Avg Write |
| --- | --- | --- | --- | --- |
| **Server-driven** | 68.9 ms | 68.5 ms | 3.9 ms | 4.02 ms |
| **Client-driven** | **30.4 ms** | **30.4 ms** | **1.55 ms** | **1.9 ms** |

---

## 4. High-Impact Interview Takeaways

1. **Design for the actual availability requirement:**
> *"If the business requirement says the system must remain writable during network partitions, I would not start with strong consistency. Dynamo shows that for workloads like shopping carts, it can be better to accept temporary divergent versions and reconcile them later than to reject customer writes."*


2. **Separate partitioning from replication:**
> *"I would first partition the key space so that ownership is evenly distributed, then independently decide how replicas are placed. Dynamo's production experience showed that coupling partition boundaries with replica placement created operational problems, so later designs explicitly decoupled the two."*


3. **Tail latency drives architecture:**
> *"I would optimize distributed storage against P99 or P99.9 rather than average latency. Dynamo explicitly observed that the 99.9th percentile was roughly an order of magnitude worse than average, so design choices such as coordinator selection and write buffering were made specifically to control tail latency."*


4. **Layered failure recovery:**
> *"I wouldn't use one recovery mechanism for every failure. Dynamo separates temporary failure handling through hinted handoff from longer-term replica divergence through anti-entropy with Merkle trees."*


5. **Protect the foreground SLA from background maintenance:**
> *"Replica synchronization and data movement are necessary, but they should be treated as background work with resource budgets. Dynamo uses feedback-based admission control so background operations are throttled when foreground latency approaches its threshold."*



---

## 5. Follow-up Deep-Dive Questions

### Question 1: "Why doesn't Dynamo simply use strong consistency with synchronous replication?"

**Ideal Answer:**
Because Dynamo's target applications prioritize availability during failures. With synchronous replication, if the system cannot reach enough replicas due to a network partition or node outage, it must reject the write.

For a shopping cart, rejecting an item addition direct costs revenue:

```
Customer adds item ──► Replica unavailable ──► Strong consistency ──► Reject write ❌

```

Dynamo instead writes to available nodes, returns success, and reconciles divergence later via vector clocks and application logic.

---

### Question 2: "Suppose a node fails while handling writes. How does Dynamo prevent data loss?"

**Ideal Answer:**
Dynamo guards against write loss through a four-tiered strategy:

1. **$N$-wise Replication:** Writes are sent to $N$ physical nodes.
2. **Sloppy Quorum:** If a designated node is down, the write routes to an alternate healthy node in the preference list.
3. **Hinted Handoff:** The substitute node stores a temporary hint and transfers the payload once the primary recovers.
4. **Anti-Entropy Synchronization:** If the substitute node crashes before handoff, background Merkle-tree reconciliation detects missing keys across nodes and repairs them.

---

### Question 3: "How does Dynamo know whether two conflicting versions can be merged or whether one supersedes the other?"

**Ideal Answer:**
Through **Vector Clocks**.

If Version $A$'s vector clock is less than or equal to Version $B$'s clock for all nodes (e.g., $V_1 = [ (A, 1) ]$ vs $V_2 = [ (A, 2) ]$), $V_2$ causally succeeds $V_1$, and $V_1$ can be safely garbage collected.

If neither vector clock dominates the other (e.g., $V_3 = [ (A, 2), (B, 1) ]$ vs $V_4 = [ (A, 2), (C, 1) ]$), the versions are concurrent. Dynamo preserves both versions and hands them to the client during the next `get()` request for application-level reconciliation.

---

## Key Architecture Diagram to Memorize

```
                         CLIENT
                           │
                           ▼
                    ┌─────────────┐
                    │Load Balancer│
                    └──────┬──────┘
                           │
                           ▼
                  Any Dynamo Node
                           │
                           ▼
                      COORDINATOR
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
          Node A         Node B         Node C
        Replica 1      Replica 2      Replica 3
      (Local Store)  (Local Store)  (Local Store)

```
