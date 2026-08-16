# Dynamo — Amazon’s Highly Available Key-Value Store

---

## 1. Executive Summary & Core Engineering Challenge

### 1.1 What problem was Dynamo solving?

The core problem was building a storage system that remains highly available even when failures are the normal operating condition.

Amazon's e-commerce platform consisted of hundreds of loosely coupled services running across tens of thousands of servers and many data centers. At this scale, components continuously experience failures. Amazon needed storage where the business requirement was effectively:

> “Keep accepting reads and writes even when parts of the infrastructure are broken.”

Shopping carts are the clearest example: a customer must be able to add/remove items even when disks fail, network routes flap, or a data center becomes unavailable. The system deliberately prioritized availability over strong consistency.

$$\text{High availability} + \text{Partition tolerance} \longrightarrow \text{Accept eventual consistency}$$

Dynamo became an always-writeable, eventually consistent, primary-key-only distributed key-value store.

### 1.2 Why not just use a relational database?

Many Amazon workloads only needed simple primary-key operations:

* `get(key)`
* `put(key, value)`

They did not require joins, relational schemas, multi-item transactions, or complex queries. Using a relational database for those workloads introduced unnecessary functionality, operational cost, and scaling limitations. Existing replication mechanisms also favored consistency over availability. Dynamo specifically targets relatively small objects, usually below 1 MB, identified by a single key.

### 1.3 Scale and operational constraints

| Dimension | Paper's Figure |
| --- | --- |
| **Amazon servers** | Tens of thousands |
| **Data centers** | Multiple worldwide |
| **Amazon services** | Hundreds |
| **Page request dependencies** | Often $>150$ services |
| **Example SLA** | 300 ms for 99.9% at 500 QPS |
| **Dynamo object size** | Usually $<1\text{ MB}$ |
| **Initial Dynamo instance scale target** | Hundreds of storage hosts |
| **Shopping-cart traffic** | Tens of millions of requests/day |
| **Shopping-cart checkouts** | $>3\text{ million/day}$ |
| **Session service** | Hundreds of thousands concurrent sessions |
| **Dynamo observed 99.9th percentile latency** | Around 200 ms |
| **Production successful responses** | 99.9995% without timeout |
| **Reported data loss** | None during observation period |
| **Typical $N$** | $3$ |

> **Interview Nuance:** The paper explicitly states Amazon did not publish absolute request rates, outage durations, and detailed latency measurements in some sections due to business sensitivity. Do not invent exact QPS numbers when referencing this paper.

### 1.4 The deepest engineering challenge

The problem reduces to a continuous operational chain:

$$\text{Failures are unavoidable} \longrightarrow \text{Waiting for certainty reduces availability} \longrightarrow \text{Allow operations during failures}$$

$$\longrightarrow \text{Replicas diverge} \longrightarrow \text{Need versioning \& conflict detection} \longrightarrow \text{Need reconciliation}$$

$$\longrightarrow \text{Need background replica repair} \longrightarrow \text{Maintain 99.9th percentile SLA}$$

---

## 2. System Architecture & High-Level Design

### 2.1 Architecture classification

This is a Software / Distributed Systems System Design for a highly available key-value storage system. The implementation breaks a storage node into three major components written in Java:

1. **Request coordination**
2. **Membership + failure detection**
3. **Local persistence engine**

| Problem | Dynamo Technique | Purpose |
| --- | --- | --- |
| **Partitioning** | Consistent hashing | Incremental scalability |
| **Versions** | Vector clocks + read reconciliation | Detect causal relationships |
| **Temporary failures** | Sloppy quorum + hinted handoff | Preserve availability |
| **Permanent failures** | Anti-entropy + Merkle trees | Background synchronization |
| **Membership** | Gossip protocol | Decentralized membership/failure information |

### 2.2 Request interface

Dynamo exposes two main operations:

* `get(key)`
* `put(key, context, object)`

The `key` and `object` are opaque byte arrays. The key is hashed using MD5, producing a 128-bit identifier that determines placement on the ring. The `context` contains version metadata (including vector-clock information) and is required when performing updates.

### 2.3 Partitioning — Consistent Hashing

Dynamo represents the hash space as a circular ring. A key is hashed into a position on the ring; walking clockwise finds the node responsible for that key range. Adding or removing a node affects primarily neighboring portions of the ring rather than requiring complete redistribution.

#### Virtual nodes

Basic consistent hashing can cause uneven load distribution and doesn't account for heterogeneous host capacities. Dynamo assigns each physical node multiple virtual nodes (tokens):

```
Physical Node A
 ├── Token 1
 ├── Token 8
 ├── Token 15
 └── Token 22

```

This improves load balancing during failures, redistribution when nodes return, incremental scaling, and hardware utilization.

### 2.4 Replication

Each object is replicated at $N$ hosts (typically $N = 3$).

```
Key K ──► Coordinator ──┬──► Replica A
                        ├──► Replica B
                        └──► Replica C

```

The coordinator stores the object locally and replicates it to the next $N-1$ successor nodes on the ring. The **preference list** identifies the distinct physical nodes responsible for a given key.

### 2.5 Eventual consistency & Vector clocks

A write may return successfully before every replica receives the update. Instead of rejecting writes during failures, Dynamo allows multiple versions to coexist.

Vector clocks capture causal relationships between versions:

```
          D1 [(Sx,1)]
               │
               ▼
          D2 [(Sx,2)]
             /   \
            ▼     ▼
D3 [(Sx,2),(Sy,1)]   D4 [(Sx,2),(Sz,1)]
            \     /
             ▼   ▼
     [Conflict: Semantic Reconciliation Needed]

```

* **Syntactic reconciliation:** The system automatically determines that an older version is an ancestor and discards it.
* **Semantic reconciliation:** Concurrent versions divergence ($D_3$ vs $D_4$) requires the application (e.g., shopping cart) to merge state upon reading.

> **Vector Clock Truncation:** To prevent metadata growth, Dynamo truncates $(node, counter)$ pairs after a threshold (e.g., 10). This introduces a theoretical risk of bound metadata degrading causal tracking, though paper authors noted it never surfaced in production.

### 2.6 Read/Write Quorum configuration

Dynamo configures quorums using:

* $N$ = Number of replicas
* $R$ = Minimum replicas required for successful read
* $W$ = Minimum replicas required for successful write

When $R + W > N$, the system achieves quorum-like consistency. However, Dynamo allows flexible configurations like $W = 1$ (yielding $R + W \le N$) when maximum write availability is preferred over immediate durability.

### 2.7 Sloppy quorum & Hinted handoff

If nodes in a key's primary preference list are down during a write:

1. The coordinator routes the write to healthy alternate nodes outside the top $N$ list.
2. The alternate node accepts the write with a **hint** in its local metadata indicating the primary intended node.
3. Upon detecting that the primary node has recovered, the alternate node transfers the hinted replica back (**hinted handoff**).

### 2.8 Anti-Entropy with Merkle Trees

To recover from permanent failures or missed hinted handoffs, Dynamo uses background anti-entropy synchronization using **Merkle trees**.

Each node maintains a Merkle tree for each key range it holds. Nodes compare the root hashes of their Merkle trees:

* **Root Hashes Match:** Replicas are synchronized; no data is transferred.
* **Root Hashes Differ:** Nodes traverse child branches to isolate and transfer only the divergent keys, minimizing disk reads and network usage.

### 2.9 Implementation details

* **Local Persistence:** Pluggable storage engine (typically Berkeley DB Transactional Data Store, BDB Java Edition, or MySQL).
* **Execution Architecture:** Event-driven, staged processing design (similar to SEDA) using Java NIO channels with request state machines.

### 2.10 Routing architecture comparison

```
Server-Driven Routing:
Client ──► Load Balancer ──► Random Node ──► Coordinator Node

Client-Driven Routing:
Client (with local ring state) ──────► Coordinator Node directly

```

| Metric | Server-Driven | Client-Driven |
| --- | --- | --- |
| **99.9% Read Latency** | 68.9 ms | 30.4 ms |
| **99.9% Write Latency** | 68.5 ms | 30.4 ms |
| **Average Read Latency** | 3.9 ms | 1.55 ms |
| **Average Write Latency** | 4.02 ms | 1.9 ms |

Clients using the client-driven library poll a Dynamo host every 10 seconds for membership updates to maintain accurate local ring state.

### 2.11 Background workload isolation

Background tasks (anti-entropy, hinted handoff, data movement) can degrade customer-facing foreground operations. Dynamo implements an **admission controller**:

```
Background Admission Controller
  │
  ├─► Monitor: Disk latency, lock contention, queue lengths
  ├─► Evaluate: e.g., Keep 99th percentile read latency <= 50ms
  └─► Throttle: Dynamically scale background task resource budget

```

---

## 3. Critical Engineering Trade-offs & Design Choices

```
                        Dynamo Design Choices
                                  │
    ┌─────────────────────────────┼─────────────────────────────┐
    ▼                             ▼                             ▼
Availability Over             Read-Side Conflict            Decoupled Logical
Consistency                  Resolution                    Partitioning
• Sloppy quorums             • Always-writeable path       • Fixed partitions
• Hinted handoff             • Application merges state      • Virtual nodes on physical
• Eventual reconciliation    • Vector clock tracking         hardware

```

1. **Availability vs. Consistency:** Push conflict resolution to the read path so writes are never rejected during network partitions or node failures.
2. **Database vs. Application Conflict Resolution:** Generic database policies like *Last-Write-Wins (LWW)* can drop updates. Application-aware resolution allows business logic (e.g., set union on cart items) to recover data accurately.
3. **Partition Strategy Evolution:** Decoupled token assignment from physical placement to prevent slow node bootstraps and costly Merkle tree recalculations.
4. **Tail Latency Optimization:** Optimized specifically for the 99.9th percentile SLA rather than average performance using memory buffering, single-hop routing, and feedback-driven throttling of background tasks.

---

## 4. System Design Interview Reference

### 4.1 System Framing

When designing an availability-first key-value store, frame the architectural progression in order:

$$\text{High Availability} \longrightarrow \text{Partitioning} \longrightarrow \text{Replication} \longrightarrow \text{Eventual Consistency}$$

$$\longrightarrow \text{Versioning (Vector Clocks)} \longrightarrow \text{Conflict Detection} \longrightarrow \text{Reconciliation \& Repair}$$

### 4.2 Key Architectural Concepts

* **Vector Clocks vs. Timestamps:** Timestamps cannot definitively establish causality between concurrent updates across partitioned nodes. Vector clocks explicitly trace lineage to distinguish causal successor writes from concurrent updates requiring reconciliation.
* **Failure Mode Layering:**
* *Temporary Unavailability:* Handled via **Sloppy Quorums** and **Hinted Handoff**.
* *Permanent/Long-lived Divergence:* Handled via **Anti-Entropy Synchronization using Merkle Trees**.


* **SLA Protection:** Prevent background maintenance operations from violating latency guarantees using **feedback-based admission control**.

### 4.3 Summary Architecture Chain

$$\text{Consistent Hashing} \longrightarrow \text{Virtual Nodes} \longrightarrow N\text{-Replica Preference Lists}$$

$$\longrightarrow \text{Tunable } R/W \text{ Quorums} \longrightarrow \text{Sloppy Quorum + Hinted Handoff}$$

$$\longrightarrow \text{Vector Clocks} \longrightarrow \text{Read Repair + Merkle Anti-Entropy}$$

$$\longrightarrow \text{Gossip Membership} \longrightarrow \text{Background Task Admission Control}$$
