# Caching — Complete Deep-Dive Revision Guide
## System Design Interview Preparation | Product-Based Companies

---

**Prepared for:** Yash | AI/GenAI Engineer transitioning to Product Company System Design Interviews
**Coverage:** Cache Strategies · Cache Eviction Policies · Redis · Cache Invalidation · Distributed Cache · Cache Warming · CPU & Browser Cache

---

## Table of Contents

1. **Cache Strategies** — Cache-Aside, Read-Through, Write-Through, Write-Behind, Write-Around
2. **Cache Eviction Policies** — LRU, LFU, FIFO, ARC, TTL
3. **Redis** — Architecture, data structures, persistence, replication, patterns
4. **Cache Invalidation** — TTL, active invalidation, CDC, stampede prevention
5. **Distributed Cache** — Consistent hashing, multi-layer, Redlock, hot keys
6. **Cache Warming** — Pre-warming, gradual traffic shift, lazy + protection
7. **CPU & Browser Cache** — Cache lines, spatial locality, HTTP headers, content hashing
8. **Appendix** — Cross-topic reference, complete caching architecture, decision framework

---

# Caching — Deep-Dive System Design Notes
### For Product-Based Company Interviews | Beginner → Advanced

---

> **How to use these notes:** Same structure as the Networking Fundamentals,
> Scalability & Load Balancing, and Databases & Storage guides.
> What is it → Why does it exist → How it works step by step → Diagrams → Internals
> → Tradeoffs → Real-world → Failures → Interview tips.
> Every concept is explained from scratch — no prior caching knowledge assumed.

---

# TOPIC 1: Cache Strategies (Cache-Aside, Read-Through, Write-Through, Write-Behind, Write-Around)

---

## 1. What Problem Does Caching Solve?

Recall the latency numbers from the Back-of-Envelope Estimation topic (Scalability notes):

```
L1/L2 cache reference     ≈ 1-7 ns
Main memory (RAM)          ≈ 100 ns
Disk seek (HDD)             ≈ 10,000,000 ns (10 ms)
Cross-continent network     ≈ 150,000,000 ns (150 ms)

RAM is ROUGHLY 100,000x FASTER than a disk seek.
```

A database query (especially one involving JOINs, aggregations, or even just reading from disk-backed storage) is FUNDAMENTALLY SLOWER than reading the same data from MEMORY. **Caching** means storing a COPY of frequently-accessed data in a FAST storage layer (usually RAM, sometimes a faster disk tier) so that REPEAT requests for the same data can be served WITHOUT repeating the expensive original computation/lookup.

```
WITHOUT CACHE:
Request 1: "Get product 789" → Query database (50ms) → Return
Request 2: "Get product 789" → Query database (50ms) → Return
Request 3: "Get product 789" → Query database (50ms) → Return
... (1000 identical requests = 1000 × 50ms = 50 SECONDS of
     database work, for data that hasn't changed at all!)

WITH CACHE:
Request 1: "Get product 789" → Cache MISS → Query database (50ms)
                              → Store result in cache → Return
Request 2: "Get product 789" → Cache HIT → Return from memory (0.5ms)
Request 3: "Get product 789" → Cache HIT → Return from memory (0.5ms)
... (999 subsequent requests = 999 × 0.5ms ≈ 0.5 SECONDS)

~100x reduction in TOTAL work, and each individual request is
~100x FASTER after the first.
```

**Analogy:** A chef keeps frequently-used ingredients (chopped onions, pre-made sauce) on the counter next to the stove (cache — fast access), rather than walking to the walk-in freezer (database — slow access) every single time a dish needs them. The freezer is still the SOURCE OF TRUTH (has everything, in bulk), but the counter holds COPIES of the "hot" items for instant access.

---

## 2. Cache-Aside (Lazy Loading) — The Most Common Pattern

### How It Works

```
                  ┌─────────────────┐
                  │   APPLICATION     │
                  └────────┬─────────┘
                           │
              ┌────────────┴────────────┐
              │                          │
              ▼                          ▼
       ┌─────────────┐           ┌─────────────┐
       │    CACHE      │           │  DATABASE     │
       │   (Redis)     │           │ (PostgreSQL)  │
       └─────────────┘           └─────────────┘

THE APPLICATION (not the cache, not the database) is responsible
for managing the cache. The cache is "alongside" (aside) the
application's normal data access path — hence "cache-aside."

READ PATH:
1. Application checks: "Is this data in the cache?"
2. CACHE HIT → return cached data immediately. DONE.
3. CACHE MISS → Application queries the DATABASE directly
4. Application WRITES the result INTO the cache (for next time)
5. Application returns the data to the caller

WRITE PATH:
1. Application writes DIRECTLY to the DATABASE
2. Application INVALIDATES (deletes) the corresponding cache
   entry (or, less commonly, updates it) — so the NEXT read
   will be a cache miss and fetch the FRESH data
```

### Step-by-Step Flow Diagram

```
READ FLOW (cache-aside):

  App                Cache               Database
   │── GET key ──────▶│
   │◀── MISS ─────────│
   │
   │── SELECT * FROM products WHERE id=789 ─────────────────▶│
   │◀──────────────────────────────────── row data ──────────│
   │
   │── SET key=value, TTL=300 ────▶│
   │◀── OK ────────────────────────│
   │
   │ (return data to caller)


WRITE FLOW (cache-aside):

  App                Cache               Database
   │── UPDATE products SET price=999 WHERE id=789 ───────────▶│
   │◀──────────────────────────────────── OK ─────────────────│
   │
   │── DEL key ───────▶│   (invalidate — remove stale entry)
   │◀── OK ────────────│
```

### Pros and Cons

```
✅ SIMPLE — the cache is "just a key-value store"; no special
   logic needed in the cache itself
✅ RESILIENT TO CACHE FAILURES — if the cache is down, the
   application can fall back to querying the database directly
   (slower, but FUNCTIONAL — a "fail open" pattern)
✅ ONLY CACHES DATA THAT'S ACTUALLY REQUESTED — "lazy loading"
   means the cache naturally fills with the "hot" data that's
   actually being accessed, not data nobody cares about

❌ FIRST REQUEST FOR ANY KEY IS ALWAYS A CACHE MISS (the "cold
   start" / "cache stampede" problem — covered in depth in
   Cache Warming and Cache Invalidation topics)
❌ DATA CAN BECOME STALE between writes and the cache
   invalidation (a brief window of inconsistency — more on this
   in Cache Invalidation)
❌ EXTRA CODE in the application for every read/write path —
   every query site must remember to check cache, populate
   cache, and invalidate on write

THIS IS THE DEFAULT, MOST WIDELY USED PATTERN — used by
virtually every application that puts Redis/Memcached "in front
of" a database, exactly as discussed in the SQL vs NoSQL "hybrid
architecture" example (Databases notes).
```

---

## 3. Read-Through Cache

### How It Works

```
                  ┌─────────────────┐
                  │   APPLICATION     │
                  └────────┬─────────┘
                           │ (ONLY talks to cache!)
                           ▼
                  ┌─────────────────┐
                  │   CACHE LIBRARY    │──────▶ DATABASE
                  │  (handles misses   │ (cache library
                  │   internally)      │  fetches on miss)
                  └─────────────────┘

THE KEY DIFFERENCE FROM CACHE-ASIDE: the APPLICATION ONLY EVER
TALKS TO THE CACHE. The cache (or a cache-provider LIBRARY sitting
in front of it) is responsible for fetching from the database on
a miss, populating itself, and returning the data — TRANSPARENTLY
to the application.

READ FLOW (read-through):

  App              Cache/Library          Database
   │── GET key ──────────▶│
   │                       │── (on MISS) SELECT ... ──▶│
   │                       │◀──────── row data ─────────│
   │                       │ (cache stores it internally)
   │◀──── value ───────────│

From the APPLICATION's perspective, it's ALWAYS just "GET key" —
it has NO IDEA whether this was a hit or a miss; the cache layer
abstracts that entirely.
```

### Pros and Cons

```
✅ CLEANER APPLICATION CODE — no "if cache miss, query DB,
   populate cache" boilerplate scattered across the codebase;
   this logic lives in ONE place (the cache library/layer)
✅ CONSISTENT CACHING LOGIC — every access goes through the
   same code path, reducing the risk of "forgot to cache this
   one query site" bugs common with cache-aside

❌ REQUIRES the cache layer/library to KNOW HOW TO QUERY the
   database (tighter coupling between cache and data source —
   e.g., needs a defined "loader function" per cache/data type)
❌ If the cache is DOWN, the application typically CANNOT fall
   back easily (it doesn't know how to query the DB itself) —
   less resilient than cache-aside unless explicitly designed
   with a fallback
❌ FIRST ACCESS still a "miss" with the same latency penalty
   (same cold-start issue as cache-aside)

USE WHEN: You want centralized, consistent caching logic and
are willing to accept tighter coupling — common in caching
FRAMEWORKS/LIBRARIES (e.g., Spring's @Cacheable, or ORM-level
caching) where the "loader" pattern is built into the framework.
```

---

## 4. Write-Through Cache

### How It Works

```
WRITE FLOW (write-through):

  App                Cache               Database
   │── SET key=value ─▶│
   │                    │── INSERT/UPDATE ──────────────▶│
   │                    │◀──────────── OK ─────────────────│
   │◀──── OK ───────────│
   (write is considered "done" only after BOTH cache AND
    database have been updated — SYNCHRONOUSLY)

THE CACHE IS UPDATED *AS PART OF* THE WRITE OPERATION ITSELF —
not invalidated/deleted (as in cache-aside), but UPDATED WITH
THE NEW VALUE, in the SAME logical operation as the database write.
```

### Pros and Cons

```
✅ CACHE IS ALWAYS CONSISTENT WITH THE DATABASE — there's no
   "stale cache" window after a write (unlike cache-aside's
   invalidate-then-repopulate cycle, which has a brief gap)
✅ READS ARE ALWAYS FAST — since every write also updates the
   cache, subsequent reads are cache HITS by definition (no
   "cold" entries for recently-written data)

❌ EVERY WRITE IS SLOWER — the write must complete on BOTH the
   cache AND the database before being considered successful
   (recall the synchronous replication tradeoff from the
   Replication topic — same latency-vs-consistency tension)
❌ CACHES DATA THAT MIGHT NEVER BE READ — if you write 1 million
   records but only 1,000 are ever read again, you've still
   populated the cache with all 1 million — WASTING CACHE
   MEMORY on "cold" data (recall this is the OPPOSITE of
   cache-aside's "lazy" approach, which only caches what's
   actually requested)

USE WHEN: Data that's written is ALMOST ALWAYS read again soon
after (write-then-read patterns), and consistency between cache
and database is important enough to accept slower writes.
```

---

## 5. Write-Behind (Write-Back) Cache

### How It Works

```
WRITE FLOW (write-behind):

  App                Cache               Database
   │── SET key=value ─▶│
   │◀──── OK ───────────│   (confirmed IMMEDIATELY —
                              database write happens LATER,
                              ASYNCHRONOUSLY, in the background)
                          │
                          │ (after some delay / batching)
                          │── INSERT/UPDATE (batched) ────▶│
                          │◀──────────── OK ─────────────────│

THE CACHE CONFIRMS THE WRITE IMMEDIATELY, then asynchronously
(often BATCHING multiple writes together) persists to the
database LATER. This is the cache-layer EQUIVALENT of
asynchronous replication (Replication topic) — the "primary"
here is the CACHE, and the database is like a lagging "replica."
```

### Pros and Cons

```
✅ EXTREMELY FAST WRITES — the client gets confirmation as soon
   as the (in-memory) cache is updated, without waiting for disk
   I/O on the database
✅ CAN BATCH/COALESCE WRITES — if the SAME key is written 100
   times in a second, write-behind can collapse this into ONE
   database write (the FINAL value) instead of 100 — reducing
   database load dramatically for write-heavy "hot keys"

❌ DATA LOSS RISK — if the CACHE crashes BEFORE the batched
   writes reach the database, those writes are LOST (the SAME
   fundamental risk as asynchronous replication's data-loss
   scenario — recall the Replication topic's "primary crashes
   before replicating" discussion)
❌ COMPLEXITY — requires careful handling of write ORDERING,
   batching logic, retry-on-failure, and reconciliation if the
   cache and database diverge

USE WHEN: Extremely high write throughput is required, and SOME
risk of data loss on cache failure is ACCEPTABLE (e.g.,
analytics counters, view counts, non-critical metrics) — NOT
appropriate for financial transactions or anything requiring
ACID durability guarantees (recall the ACID topic — write-behind
explicitly TRADES AWAY durability for speed).
```

---

## 6. Write-Around Cache

### How It Works

```
WRITE FLOW (write-around):

  App                Cache               Database
   │── UPDATE ──────────────────────────────────────▶│
   │◀──────────────────── OK ──────────────────────────│
   (cache is NOT touched at all during the write)

The cache is BYPASSED entirely on writes — data goes DIRECTLY
to the database. The cache is ONLY populated later, on a
subsequent READ (cache-aside style miss-then-populate).
```

### Pros and Cons

```
✅ AVOIDS CACHE POLLUTION from write-heavy data that's RARELY
   READ AGAIN — e.g., bulk data imports, logs, write-once
   records. Write-through would needlessly cache ALL of this;
   write-around keeps the cache focused on actually-read data.

❌ FIRST READ AFTER A WRITE IS ALWAYS A CACHE MISS (potentially
   slower for write-then-immediately-read patterns) — same
   cold-start issue, but now GUARANTEED on every write rather
   than just the first-ever read

USE WHEN: Data is written much more often than it's read, OR
written data and read "hot" data are largely DIFFERENT sets
(e.g., a system ingesting large volumes of raw events, where
only AGGREGATED/PROCESSED views are actually read frequently).
```

---

## 7. Strategy Comparison Table

```
┌──────────────────┬───────────────┬────────────────┬──────────────────┬─────────────────────────┐
│ Strategy           │ Write Latency  │ Read Latency    │ Consistency        │ Best For                  │
│                   │ (vs DB direct) │ (after write)   │ Risk                │                          │
├──────────────────┼───────────────┼────────────────┼──────────────────┼─────────────────────────┤
│ Cache-Aside        │ Same as DB     │ Miss on first    │ Brief staleness     │ General purpose, most     │
│ (Lazy Loading)     │ (cache not in  │ access, then     │ between write &     │ common default — read-     │
│                   │ write path)    │ fast              │ invalidation        │ heavy workloads            │
│ Read-Through       │ Same as DB     │ Miss on first    │ Same as cache-aside │ Centralized caching         │
│                   │                │ access, then     │                     │ logic via libraries/        │
│                   │                │ fast              │                     │ frameworks                  │
│ Write-Through       │ SLOWER (waits  │ Always fast      │ Always consistent   │ Write-then-read patterns,  │
│                   │ for both)       │ (pre-populated)  │ (no staleness)      │ consistency-critical data   │
│ Write-Behind        │ FASTEST        │ Always fast      │ DATA LOSS RISK on   │ Extreme write throughput,  │
│ (Write-Back)        │ (async to DB)  │ (pre-populated)  │ cache crash         │ non-critical data           │
│ Write-Around        │ Same as DB     │ ALWAYS miss on   │ Brief staleness     │ Write-heavy, rarely-read-   │
│                   │ (cache not in  │ first read after │                     │ again data (avoids cache    │
│                   │ write path)     │ a write          │                     │ pollution)                  │
└──────────────────┴───────────────┴────────────────┴──────────────────┴─────────────────────────┘
```

---

## 8. Combining Strategies — The Real-World Default

```
MOST PRODUCTION SYSTEMS USE: Cache-Aside for READS + Write-Around
or "invalidate on write" for WRITES.

This combination is so common it's often just called "the cache
pattern" without further qualification:

READ: Cache-aside (check cache → miss → query DB → populate cache)
WRITE: Update DB directly → DELETE (invalidate) the cache key
       (NOT write-through's "update cache with new value" — simply
       REMOVING the stale entry is simpler and avoids subtle bugs
       around partial/concurrent updates — covered more in Cache
       Invalidation topic)

WHY DELETE INSTEAD OF UPDATE ON WRITE?
If TWO concurrent writes happen to update the SAME cache key with
DIFFERENT new values (recall concurrency issues from the ACID
topic), updating the cache directly risks a RACE CONDITION where
the cache ends up with an OLDER value than the database (the
"slower" write's cache-update happens AFTER the "faster" write's
cache-update, but the database has the FASTER write's value as
final). DELETING simply means "the next read will fetch
WHATEVER IS CURRENT in the database" — avoiding this race
entirely. This is a widely-recommended best practice.
```

---

## 9. Real-World Usage

**Facebook's "Memcache" architecture (famous 2013 paper):** Uses cache-aside extensively at massive scale — billions of cache lookups per second across thousands of Memcached servers, with application-level logic handling cache population on misses and invalidation on writes (using a system called "McSqueal" to propagate invalidations from database changes — connecting to the CDC/replication-log concepts from the Databases notes).

**Spring Framework (`@Cacheable`, `@CacheEvict`):** A read-through-style abstraction — developers annotate methods, and the framework handles checking the cache, calling the method on a miss, and storing the result — abstracting cache-aside boilerplate into declarative annotations.

**Analytics/counters (write-behind in practice):** Systems tracking "view counts" or "like counts" often use write-behind-style patterns — increment a counter in Redis immediately (fast), and periodically flush aggregated counts to the database (e.g., every few seconds or minutes) — accepting that a crash might lose a FEW seconds of counts, which is an acceptable tradeoff for this kind of approximate data.

---

## 10. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Cache and database disagree       │ Write-through/cache-aside        │ Prefer "invalidate" (delete) over │
│ ("cache incoherence")             │ update race condition — two       │ "update" on write; for stricter   │
│                                  │ concurrent writes update cache    │ guarantees, use distributed locks │
│                                  │ in the wrong order vs DB           │ around the cache-update step       │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Cache failure causes total         │ Read-through pattern with no      │ Design read-through layers with    │
│ outage (app can't function          │ fallback; application has NO       │ an explicit DB-fallback path, OR    │
│ without cache)                     │ direct DB-query code path          │ prefer cache-aside for resilience  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Data loss after cache restart       │ Write-behind cache crashed         │ Use write-behind only for           │
│                                  │ before flushing batched writes     │ non-critical data; for critical    │
│                                  │ to the database                    │ data, write-through or              │
│                                  │                                  │ synchronous DB writes               │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Cache filled with "cold" data,      │ Write-through used for write-      │ Switch to write-around or           │
│ evicting genuinely "hot" data       │ heavy data that's rarely read       │ cache-aside for this data — only   │
│ (poor hit rate)                    │ again — wastes cache memory         │ cache what's actually read          │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 11. Interview Quick-Fire Answers

**Q: Explain cache-aside and why it's the most commonly used pattern.**
A: In cache-aside, the application checks the cache first; on a miss, it queries the database directly, then populates the cache for next time. On writes, the application updates the database and invalidates (deletes) the corresponding cache entry. It's popular because it's simple (the cache is "just a key-value store" with no special logic), resilient (if the cache is down, the app can fall back to querying the database directly), and naturally caches only data that's actually being requested ("lazy loading").

**Q: What's the difference between write-through and write-behind caching?**
A: Write-through updates BOTH the cache and the database synchronously as part of the write — the write isn't considered complete until both succeed, giving strong consistency but slower writes. Write-behind (write-back) confirms the write as soon as the cache is updated, then asynchronously (often batched) persists to the database later — giving very fast writes and the ability to coalesce repeated writes to the same key, but with a data-loss risk if the cache crashes before the database write completes.

**Q: Why might you choose write-around instead of write-through?**
A: Write-through caches EVERY write, even data that might never be read again — wasting cache memory on "cold" data and potentially evicting genuinely "hot" entries. Write-around bypasses the cache on writes entirely (writing directly to the database), and only populates the cache later via a normal read (cache-aside style) if and when that data is actually requested. This is better for write-heavy workloads where written data and frequently-read data are largely different sets.

**Q: On a write, why is it generally better to invalidate (delete) a cache entry rather than update it with the new value?**
A: If two concurrent writes update the same key with different values, directly updating the cache risks a race condition — the cache could end up holding an OLDER value than what's now in the database, if the "slower" write's cache update happens after the "faster" write's. Simply deleting the cache entry avoids this entirely: the next read will be a cache miss that fetches whatever value is CURRENTLY in the database, which is always correct regardless of write ordering.

---
---

# TOPIC 2: Cache Eviction Policies

---

## 1. What Problem Do Eviction Policies Solve?

Cache memory is FINITE — typically RAM, which is far more expensive and limited than disk storage (recall the storage cost discussions from the Object Storage topic). You CANNOT cache "everything" — at some point, the cache fills up, and to store a NEW item, an OLD item must be REMOVED.

**Eviction policy** is the algorithm that decides WHICH item to remove when the cache is full and a new item needs to be added. The goal: remove items that are LEAST LIKELY to be needed again soon, keeping items that ARE likely to be requested again — maximizing the **cache hit rate** (the percentage of requests served from cache vs requiring the slower underlying source).

```
CACHE IS FULL (capacity = 4 items):

┌─────────┬─────────┬─────────┬─────────┐
│ Item A   │ Item B   │ Item C   │ Item D   │
└─────────┴─────────┴─────────┴─────────┘

NEW REQUEST: "Cache Item E" (cache miss, need to add it)

WHICH ITEM GETS EVICTED to make room for E?
→ This is the question eviction policies answer.

A GOOD eviction policy: evicts the item LEAST LIKELY to be
requested again soon — maximizing future hit rate.
A BAD eviction policy: might evict an item that's about to be
requested again immediately — causing a "miss, fetch, cache,
evict again" THRASHING pattern.
```

---

## 2. LRU (Least Recently Used) — The Most Common Policy

### Core Intuition

```
LRU's ASSUMPTION: "If an item HASN'T been accessed in a long
time, it's UNLIKELY to be accessed again soon — recently
accessed items are more likely to be accessed again
('temporal locality')."

EVICT: the item that was accessed LONGEST AGO.

EXAMPLE TIMELINE (cache capacity = 3):

Access A → Cache: [A]
Access B → Cache: [A, B]
Access C → Cache: [A, B, C]   (cache now full)
Access A → Cache: [B, C, A]   (A moved to "most recently used" end)
Access D → Cache MISS, evict LEAST recently used (B)
           Cache: [C, A, D]
```

### Implementation — Doubly Linked List + Hash Map

```
LRU is typically implemented with TWO data structures working
together:

1. A HASH MAP: key → pointer to a node in the linked list
   (gives O(1) lookup — "is this key in the cache, and where?")

2. A DOUBLY LINKED LIST: nodes ordered by RECENCY OF ACCESS
   - HEAD = most recently used
   - TAIL = least recently used

┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  D (MRU) │◀──▶│  A       │◀──▶│  C       │◀──▶│  B (LRU) │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
   HEAD                                          TAIL

ON ACCESS (GET key):
1. Hash map lookup → find the node (O(1))
2. REMOVE the node from its current position in the list (O(1)
   — doubly linked list allows removal without traversal, given
   a direct pointer to the node)
3. INSERT the node at the HEAD (most recently used) (O(1))

ON INSERT (cache miss, adding new item):
1. If cache is FULL: remove the TAIL node (least recently used)
   — O(1) — and remove its entry from the hash map
2. Insert the new item at the HEAD, add to hash map

EVERY OPERATION IS O(1) — this is WHY LRU is the most common
eviction policy: it provides good real-world hit rates AND is
CHEAP to implement and execute on every single cache access
(critical, since eviction-policy bookkeeping happens on EVERY
read/write — any O(log n) or worse overhead here would be a
significant tax on overall cache performance).
```

### LRU's Weakness — Scans Ruin Everything

```
SCENARIO: A cache with capacity 1000, holding "hot" frequently-
accessed items (A1-A1000, each accessed thousands of times/day).

A BACKGROUND JOB runs a "SELECT * FROM all_products" query that
touches 5,000 DIFFERENT products, ONCE EACH (a full table scan
for, say, a nightly export).

WHAT HAPPENS WITH PURE LRU:
Each of these 5,000 one-time accesses becomes the "most recently
used" — pushing the genuinely HOT items (A1-A1000) toward the
TAIL (least recently used), and EVENTUALLY EVICTING ALL OF THEM!

After the scan completes: the cache is now FULL of items that
were each accessed EXACTLY ONCE and will likely NEVER be accessed
again — while the TRULY hot items (A1-A1000) have all been
evicted. The cache's hit rate COLLAPSES until A1-A1000 are
re-fetched and re-cached (recall the "cache stampede"
terminology from the CDN topic in Networking Fundamentals —
similar concept).

THIS IS WHY: more sophisticated policies (LFU, ARC — below)
exist specifically to address this "scan resistance" problem.
```

---

## 3. LFU (Least Frequently Used)

### Core Intuition

```
LFU's ASSUMPTION: "Items accessed MANY TIMES over their lifetime
in the cache are more valuable than items accessed RECENTLY but
only ONCE."

EVICT: the item with the LOWEST ACCESS COUNT (frequency).

THIS DIRECTLY ADDRESSES LRU's "ONE-TIME SCAN" WEAKNESS: a
one-time-accessed item from a background scan has a frequency
of 1 — among the LOWEST possible — so it gets evicted FIRST,
protecting genuinely hot items (frequency in the thousands) from
being displaced by a single scan.

EXAMPLE:
Item A: accessed 500 times (frequency=500)
Item B: accessed 3 times (frequency=3)
Item C: accessed 1 time (frequency=1) ← just added by a scan

Cache full, need to evict → evict Item C (lowest frequency)
```

### LFU's Weakness — "Frequency Inertia" / Cold Start

```
PROBLEM 1: An item that was VERY popular LAST WEEK (frequency=
10,000) but hasn't been accessed AT ALL today still has a HIGH
frequency count — it WON'T be evicted, even though it's no
longer relevant ("frequency inertia" — past popularity protects
an item indefinitely, or for a very long time).

PROBLEM 2: A BRAND NEW item that just became popular (e.g.,
breaking news, a flash sale item) starts with frequency=1 —
it looks identical to a "one-time scan" item, and might be
evicted IMMEDIATELY even though it's about to be accessed
thousands of times in the next few minutes ("new items are
unfairly penalized").

MITIGATION: Many real-world LFU implementations use AGING —
periodically DECAYING all frequency counts (e.g., halve every
counter every hour) so that OLD popularity fades over time,
giving NEW items a fairer chance while still protecting
GENUINELY CONSISTENTLY popular items.
```

---

## 4. FIFO (First In, First Out)

```
EVICT: the item that was ADDED to the cache LONGEST AGO —
regardless of how often or recently it's been ACCESSED.

EXAMPLE (capacity = 3):
Insert A → [A]
Insert B → [A, B]
Insert C → [A, B, C]
ACCESS A (doesn't matter for FIFO!)
Insert D → cache full, evict A (oldest INSERTION, even though
            it was JUST accessed!) → [B, C, D]

WEAKNESS: Completely ignores ACCESS PATTERNS — an item that's
accessed CONSTANTLY can still be evicted just because it was
inserted first. Rarely used alone in serious caching systems,
but SIMPLE to implement (just a queue) and sometimes used as a
baseline or component of more sophisticated policies.
```

---

## 5. ARC (Adaptive Replacement Cache) — The Sophisticated Hybrid

```
ARC (used by ZFS filesystem, and conceptually by some advanced
caching systems) maintains TWO LRU lists:

LIST 1 (T1): Items accessed ONCE recently (recency-focused, like
              pure LRU)
LIST 2 (T2): Items accessed MULTIPLE TIMES (frequency-focused,
              like LFU)

ARC DYNAMICALLY ADJUSTS the SIZE ALLOCATED to each list based on
OBSERVED WORKLOAD PATTERNS:
- If "ghost hits" on T1 (items recently evicted from T1 that
  would have been hits if T1 were larger) are HIGH → workload
  is "recency-dominated" → GROW T1's allocation
- If "ghost hits" on T2 are HIGH → workload is "frequency-
  dominated" → GROW T2's allocation

ESSENTIALLY: ARC tries to get the BEST OF BOTH LRU and LFU,
adapting automatically to whether the CURRENT workload behaves
more like "recency matters" or "frequency matters" — without
requiring manual tuning.

TRADEOFF: More complex to implement and reason about than LRU;
the EXTRA bookkeeping (tracking "ghost" entries for recently-
evicted items) has its own memory/CPU overhead. Most general-
purpose caches (Redis, Memcached) default to LRU-family
algorithms; ARC-style approaches appear in specialized systems
(filesystem caches, database buffer pools) where the complexity
investment pays off at that scale.
```

---

## 6. TTL (Time To Live) — Eviction by Expiry, Not Capacity

```
A DIFFERENT DIMENSION entirely: TTL-based expiry removes items
based on AGE (how long since they were cached), REGARDLESS of
memory pressure or access patterns.

SET key value EX 300   ← expires in 300 seconds, no matter what

WHY THIS MATTERS SEPARATELY FROM LRU/LFU/etc.:
- LRU/LFU/etc. are about WHAT TO EVICT WHEN THE CACHE IS FULL
  (a CAPACITY-driven concern)
- TTL is about WHEN DATA BECOMES STALE/INVALID — a CORRECTNESS
  concern, directly connecting to the Cache Invalidation topic
  (next topic!) — even if the cache has PLENTY of free memory,
  an item with an EXPIRED TTL should NOT be served (it might be
  WRONG/OUTDATED data)

MOST PRODUCTION CACHES COMBINE BOTH:
- TTL ensures data doesn't become "too old" regardless of access
  patterns (correctness)
- LRU/LFU/etc. ensures the cache stays within MEMORY LIMITS,
  evicting the "least valuable" items when full (capacity)

An item can be evicted EITHER because its TTL expired OR because
it was chosen by the eviction policy due to memory pressure —
TWO INDEPENDENT REASONS an item might not be in the cache anymore.
```

---

## 7. Eviction Policy Comparison Table

```
┌──────────────────┬─────────────────────────┬──────────────────────────────┬───────────────────────────┐
│ Policy             │ Eviction Criterion        │ Strength                       │ Weakness                    │
├──────────────────┼─────────────────────────┼──────────────────────────────┤───────────────────────────┤
│ LRU                │ Longest time since LAST    │ Simple, O(1), good for         │ Vulnerable to "scan"        │
│                   │ access                     │ "temporal locality" workloads  │ pollution (one-time         │
│                   │                           │ (most common workloads)        │ accesses evict hot data)    │
│ LFU                │ Lowest access COUNT        │ Resistant to scan pollution;    │ "Frequency inertia" — old   │
│                   │ (frequency)                │ protects consistently-hot items│ popularity lingers; new      │
│                   │                           │                                │ items unfairly penalized     │
│ FIFO               │ Oldest INSERTION time       │ Trivially simple (a queue)     │ Ignores access patterns      │
│                   │                           │                                │ entirely — often poor hit    │
│                   │                           │                                │ rates                         │
│ ARC                │ Adaptive (recency +        │ Best-of-both LRU/LFU,           │ Implementation complexity,   │
│                   │ frequency, self-tuning)    │ adapts to workload shifts      │ extra bookkeeping overhead   │
│ TTL                │ Item AGE (absolute time)    │ Ensures CORRECTNESS/freshness, │ Doesn't address CAPACITY —   │
│ (orthogonal to     │                           │ independent of capacity         │ used ALONGSIDE, not INSTEAD  │
│ the above)         │                           │ pressure                       │ of, a capacity-based policy   │
└──────────────────┴─────────────────────────┴──────────────────────────────┴───────────────────────────┘
```

---

## 8. Real-World Usage

**Redis:** Supports multiple eviction policies, configurable via `maxmemory-policy` — including `allkeys-lru`, `allkeys-lfu`, `volatile-lru` (only evict keys WITH a TTL set, never evict "permanent" keys), `volatile-ttl` (evict the key with the NEAREST expiry first), and `noeviction` (return errors on writes once full, rather than evicting anything — used when the application MUST know if the cache is full rather than silently losing data).

**CDNs (recall Networking Fundamentals):** CDN edge caches typically use LRU-family policies combined AGGRESSIVELY with TTL — content with `Cache-Control: max-age=86400` will be evicted EITHER when its TTL expires OR when the edge node's storage fills up and LRU kicks in, whichever happens first.

**CPU Caches (preview of Topic 7):** Modern CPU caches (L1/L2/L3) use variations of LRU (often "pseudo-LRU" — a cheaper-to-implement approximation, since TRUE LRU bookkeeping at CPU-cache speeds/sizes would itself be too costly) — illustrating that even at the HARDWARE level, the LRU-vs-implementation-cost tradeoff applies.

**Database buffer pools (recall Indexing topic):** A database's in-memory cache of disk PAGES (containing B+ Tree nodes, row data) uses eviction policies very similar to ARC — PostgreSQL's buffer manager and many database systems use "clock sweep" algorithms (an efficient APPROXIMATION of LRU, avoiding the overhead of maintaining an exact recency-ordered list for potentially millions of pages).

---

## 9. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Cache hit rate suddenly drops      │ A large one-time scan/batch job   │ Switch to LFU or ARC-style policy; │
│ after a batch job runs             │ pollutes an LRU cache, evicting    │ OR run batch jobs against a        │
│                                  │ genuinely hot data                 │ SEPARATE cache instance / bypass   │
│                                  │                                  │ the cache for batch reads (recall  │
│                                  │                                  │ write-around's "avoid pollution"   │
│                                  │                                  │ principle, applied to reads)        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Stale data served despite           │ Capacity-based eviction (LRU/LFU)│ ALWAYS set appropriate TTLs —       │
│ plenty of free cache memory         │ alone doesn't address              │ capacity-based eviction and TTL    │
│                                  │ CORRECTNESS — an item can sit     │ solve DIFFERENT problems and       │
│                                  │ in cache indefinitely if never     │ should be used TOGETHER             │
│                                  │ evicted for capacity reasons        │                                     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Writes fail unexpectedly when       │ `noeviction` policy configured,    │ Choose `noeviction` deliberately   │
│ cache is full                     │ cache at maxmemory, and a NEW      │ only when "fail loudly rather than │
│                                  │ key is being written                │ silently evict" is the desired      │
│                                  │                                  │ behavior; otherwise use an LRU/LFU │
│                                  │                                  │ variant                             │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ A "celebrity" cached item is        │ LFU's frequency-based protection   │ Implement frequency AGING/decay     │
│ never evicted despite being         │ keeps high-frequency-but-stale     │ so historical popularity fades      │
│ irrelevant now                     │ items indefinitely                  │ over time                           │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 10. Interview Quick-Fire Answers

**Q: How is LRU typically implemented to achieve O(1) operations?**
A: A combination of a hash map (key → pointer to a node, for O(1) lookup) and a doubly linked list ordered by recency (head = most recently used, tail = least recently used). On access, the node is removed from its current position (O(1) given a direct pointer in a doubly linked list) and moved to the head. On eviction, the tail node is removed. Both operations are O(1), which is essential since this bookkeeping happens on every cache access.

**Q: What's a major weakness of LRU, and how does LFU address it?**
A: LRU is vulnerable to "scan pollution" — if a one-time batch job accesses many different items just once, each becomes "most recently used," pushing genuinely hot, frequently-accessed items toward eviction, even though they'll be needed again soon. LFU evicts based on access FREQUENCY rather than recency — a one-time-accessed item has a frequency of 1 (among the lowest possible) and gets evicted first, protecting items with consistently high access counts.

**Q: What's the weakness of pure LFU, and how is it typically mitigated?**
A: LFU has "frequency inertia" — an item that was very popular in the past retains a high frequency count even if it's no longer being accessed, protecting it from eviction long after it's relevant. It also unfairly penalizes brand-new items that are JUST becoming popular (they start at frequency=1, indistinguishable from a one-time scan access). Mitigation: periodically decay/age all frequency counters (e.g., halve them every hour), so old popularity fades and new items get a fairer chance.

**Q: How do TTL-based expiry and capacity-based eviction policies (LRU/LFU) relate to each other?**
A: They address different concerns and are typically used TOGETHER. TTL ensures CORRECTNESS/freshness — an item is removed after a defined time regardless of how often it's accessed or whether the cache has free memory (connects to Cache Invalidation). Capacity-based eviction policies (LRU/LFU/etc.) ensure the cache stays within its MEMORY LIMIT, choosing which item to remove when a new item needs space. An item can be evicted for either reason independently — both are needed for a correct AND memory-bounded cache.


---
---

# TOPIC 3: Redis

---

## 1. What Problem Does Redis Solve?

We established in Topic 1 that caching requires a fast, in-memory key-value store. **Redis** (Remote Dictionary Server) is the dominant, industry-standard implementation of this — but calling Redis "just a cache" massively undersells it. Redis is an in-memory **data structure server** — it understands and natively operates on STRINGS, LISTS, SETS, SORTED SETS, HASH MAPS, STREAMS, and more, not just opaque byte blobs.

```
MEMCACHED (the simpler predecessor):
GET key → opaque bytes
SET key value → opaque bytes

REDIS:
GET key → string
LPUSH list value → prepend to a list
ZADD leaderboard 1500 "Yash" → add to a sorted set with a score
HSET user:123 name "Yash" email "y@x.com" → set hash fields
INCR page_views → atomically increment a counter
XADD stream * field value → append to a stream

This NATIVE DATA STRUCTURE support means many algorithms that
would normally require fetching data from the cache, modifying
it in application code, and writing it back can be done in Redis
WITH A SINGLE ATOMIC COMMAND — avoiding race conditions and
reducing round trips.
```

---

## 2. Core Architecture — Why Redis Is Fast

```
REASON 1: ENTIRELY IN-MEMORY (primary storage)
All data lives in RAM by default. RAM access ~100ns vs disk
~10ms (from Back-of-Envelope latency numbers, Scalability notes)
= ~100,000x faster. Redis serves requests WITHOUT ANY DISK I/O
on the critical path (persistence is async, discussed below).

REASON 2: SINGLE-THREADED EVENT LOOP (for command processing)
Redis processes commands on a SINGLE THREAD using an event loop
(similar to Node.js — recall WebSockets topic's C10K discussion
from Networking Fundamentals):

┌─────────────────────────────────────────────────────────────┐
│  Event Loop (single thread)                                   │
│                                                               │
│  while true:                                                  │
│    events = epoll_wait(all_client_sockets)                    │
│    for each readable event:                                   │
│      command = read_command(socket)                           │
│      result = execute_command(command)  ← PURE CPU, no I/O   │
│      write_response(socket, result)                           │
└─────────────────────────────────────────────────────────────┘

WHY SINGLE-THREADED IS NOT A BOTTLENECK HERE:
Commands are executed in MEMORY — no disk I/O, no network I/O
on the data path. Each command completes in MICROSECONDS.
A single thread can execute ~100,000-1,000,000 simple commands
per second because each command is so fast that thread-switching
overhead would DOMINATE if multi-threading were used for
commands. The event loop handles I/O multiplexing (many
clients) without blocking on any one client.

NOTE: Redis 6.0+ added "threaded I/O" for READING requests
and WRITING responses (the network I/O), while STILL processing
commands on a single thread — addressing the case where network
I/O itself became the bottleneck at very high connection counts,
without introducing the complexity of concurrent command
execution.

REASON 3: OPTIMIZED DATA STRUCTURES
Redis's internal implementations use highly optimized C data
structures — listpack (compact encoding for small lists/hashes/
sets), skiplist (for sorted sets — O(log n) rank operations),
intset (compact set of integers) — automatically switching
between compact and full-featured representations based on
collection size.
```

---

## 3. Redis Data Types — Deep Dive with Use Cases

### String

```
COMMANDS: GET, SET, INCR, INCRBY, APPEND, STRLEN, MGET, MSET
INTERNAL: raw bytes (max 512 MB per value)

USE CASES:
- Simple key-value caching: SET product:789 "{name:Laptop,...}" EX 300
- Counters: INCR page_views:homepage (atomic — no race condition
  even with many concurrent writers, unlike app-level read-modify-
  write operations from the ACID topic)
- Distributed locks: SET lock:resource "owner_id" NX EX 30
  (NX = "set only if Not eXists" — this is how distributed
  mutex locks are implemented in Redis, discussed further in
  Distributed Cache topic)
- Session storage: SET session:a3f8b2c1 "{user_id:123,...}" EX 3600

ATOMIC INCREMENT EXAMPLE:
Without Redis:
  1. App reads counter from DB: SELECT count FROM page_views
  2. App increments: count + 1
  3. App writes back: UPDATE page_views SET count = {count+1}
  RACE CONDITION: Two concurrent requests BOTH read "100", both
  write "101" — one increment is lost!

With Redis:
  INCR page_views  → atomically returns 101, then 102, etc.
  No race condition possible — Redis processes commands serially.
```

### List

```
COMMANDS: LPUSH, RPUSH, LPOP, RPOP, LRANGE, LLEN, LINDEX
INTERNAL: Doubly linked list (for large lists) or listpack
          (compact array for small lists, ≤128 elements)

USE CASES:
- Message queues (simple): RPUSH queue:emails "email_task_1"
                            LPOP queue:emails (blocks until item available
                            with BLPOP — "blocking left pop")
- Recent activity log: LPUSH user:123:activity "login"
                       LRANGE user:123:activity 0 9 (last 10 events)
- Rate limiting sliding window: track timestamps of recent requests
  as a list, prune old entries (though Redis Sorted Set is more
  efficient for this — see below)

DIAGRAM:
LPUSH adds to LEFT (head)  →  RPUSH adds to RIGHT (tail)
LPOP removes from LEFT     →  RPOP removes from RIGHT

[new_item] ← LPUSH               RPUSH → [new_item]
[C] [B] [A]                               [A] [B] [C]
LPOP → removes C             RPOP → removes C
```

### Hash

```
COMMANDS: HSET, HGET, HGETALL, HMSET, HMGET, HDEL, HINCRBY
INTERNAL: listpack (≤64 fields, ≤64 bytes/value) or hash table

USE CASES:
- Object storage (avoids serializing/deserializing entire JSON
  just to update one field):
  HSET user:123 name "Yash" email "y@x.com" city "Pune"
  HGET user:123 email  → "y@x.com"
  HSET user:123 city "Mumbai"  → update just the city field,
                                   without touching name/email
  HGETALL user:123 → {name:Yash, email:y@x.com, city:Mumbai}

- Shopping cart (each item is a hash field):
  HSET cart:user_123 product_789 2   (qty 2 of product 789)
  HSET cart:user_123 product_456 1   (qty 1 of product 456)
  HINCRBY cart:user_123 product_789 1  (add 1 more of 789)
```

### Set

```
COMMANDS: SADD, SREM, SISMEMBER, SMEMBERS, SUNION, SINTER, SDIFF
INTERNAL: intset (for small sets of integers) or hash table

USE CASES:
- Unique visitors tracking: SADD visitors:page_789:2026-06-14 user_123
  SCARD visitors:page_789:2026-06-14  → unique visitor count
  (a user visiting MULTIPLE TIMES only counts ONCE — sets have
  no duplicates by definition)
- Tags/relationships: SADD product:789:tags "electronics" "laptop"
  SINTER product:789:tags product:456:tags  → common tags
- Social graph (who follows whom): SADD follows:user_123 user_456
  SMEMBERS follows:user_123  → all users that user_123 follows
```

### Sorted Set (ZSet)

```
COMMANDS: ZADD, ZRANK, ZRANGE, ZRANGEBYSCORE, ZREVRANK, ZREM
INTERNAL: listpack (small) or a combination of hash table (for
          O(1) member→score lookup) + skiplist (for O(log n)
          rank/range operations)

EVERY MEMBER has a FLOATING-POINT SCORE. Members are ordered
by score (ascending). When scores are equal, members are ordered
LEXICOGRAPHICALLY.

USE CASES:
- Leaderboards:
  ZADD leaderboard 1500 "Yash"
  ZADD leaderboard 2300 "Priya"
  ZADD leaderboard 1800 "Rahul"
  ZREVRANGE leaderboard 0 2 WITHSCORES  → ["Priya",2300,"Rahul",1800,"Yash",1500]
  ZRANK leaderboard "Yash"  → 2 (0-indexed rank, ascending = rank 2)
  ZREVRANK leaderboard "Yash"  → 0 (reversed, highest score = rank 0)

- Rate limiting (sliding window counter from the Rate Limiting
  topic, Scalability notes):
  ZADD rate:user_123 1718000001 "req_1"
  ZADD rate:user_123 1718000002 "req_2"
  ZREMRANGEBYSCORE rate:user_123 0 (now-60)  ← remove entries older than 60s
  ZCARD rate:user_123  → count of requests in last 60 seconds

- Delayed task queue:
  ZADD delayed_tasks <unix_timestamp> "task_id"
  ZRANGEBYSCORE delayed_tasks 0 <now>  → tasks ready to execute
```

### Stream

```
COMMANDS: XADD, XREAD, XREADGROUP, XACK, XLEN
Redis Streams (added in Redis 5.0) — a persistent, append-only
log of messages, similar to Kafka but MUCH simpler.

USE CASES:
- Event sourcing (lightweight): record events as they happen,
  multiple consumers can read independently
- Activity feeds
- Inter-service messaging (for small-scale systems that don't
  need Kafka's throughput/retention guarantees)

XADD events * user_id 123 action "login" timestamp 1718000000
         ↑
         * = auto-generate message ID (timestamp-based)

Consumer reads: XREAD COUNT 10 STREAMS events 0  ← read up to 10 messages
Consumer group: XREADGROUP GROUP workers consumer1 COUNT 10 STREAMS events >
               ← ">" = only undelivered messages; each message delivered
                 to ONE consumer in the group (like Kafka consumer groups)
```

---

## 4. Redis Persistence — Durability in an In-Memory Store

```
Redis's PRIMARY storage is RAM. What happens when Redis RESTARTS?
By default, data is LOST. But Redis offers TWO OPTIONAL
persistence mechanisms (connecting to the Durability concept
from ACID, and the WAL concept from Replication — same ideas!):

OPTION 1: RDB (Redis Database Snapshot)
Periodically takes a FULL SNAPSHOT of all data and writes to disk.

save 900 1      ← save if ≥1 key changed in last 15 minutes
save 300 10     ← save if ≥10 keys changed in last 5 minutes
save 60 10000   ← save if ≥10,000 keys changed in last 1 minute

HOW: Redis forks the process (copy-on-write, so the fork is fast
and doesn't block commands). The forked child writes the snapshot
to a new .rdb file while the parent continues serving commands.

PROS: Compact single-file snapshot; fast restart (load one file)
CONS: Can LOSE UP TO the last snapshot interval of writes if
Redis crashes (if the crash happens just before a scheduled save)

OPTION 2: AOF (Append-Only File) — like a WAL!
EVERY WRITE COMMAND is appended to an AOF file as it happens.
On restart, Redis REPLAYS the entire AOF file to reconstruct state.

appendfsync always    ← fsync on every command (slowest, safest)
appendfsync everysec  ← fsync every second (lose max ~1s of data)
appendfsync no        ← let OS decide when to flush (fastest, least safe)

PROS: Durability much better than RDB (lose at most 1 second of
writes with everysec, or nothing with always)
CONS: AOF file GROWS CONTINUOUSLY — Redis periodically
REWRITES/COMPRESSES the AOF (a "rewrite" removes commands that
are superseded by later ones, e.g., 1000 INCR commands on a
key collapse to one SET command with the final value)

OPTION 3: Hybrid (RDB + AOF) — recommended for most productions:
Use RDB for fast restarts (load the snapshot, then replay only
the AOF SINCE the snapshot) — combining fast recovery with
minimal data loss.

CONNECTION TO REPLICATION (Databases notes):
Redis's RDB snapshot is ALSO used for INITIAL REPLICATION —
when a new Redis replica connects to the primary, the primary
takes an RDB snapshot and sends it to the replica as the
"initial state," then sends the AOF stream of changes since
the snapshot. EXACTLY the same mechanism as database replication
(snapshot + WAL stream) — a recurring pattern!
```

---

## 5. Redis Replication and High Availability

### Primary-Replica Replication

```
┌──────────────────┐    async replication    ┌──────────────────┐
│  PRIMARY (master)  │ ─────────────────────▶│  REPLICA (slave)   │
│  (reads + writes) │    (AOF stream)         │  (reads only)      │
└──────────────────┘                         └──────────────────┘

Same async/sync tradeoffs as database replication (Databases
Topic 4). Redis replication is ASYNCHRONOUS by default — primary
confirms writes immediately, replicates in background.
WAIT command can enforce synchronous-like behavior:
WAIT 1 100  ← wait until 1 replica has replicated, timeout 100ms

REDIS SENTINEL:
Automated failover for Redis primary-replica setups.

┌──────────────────────────────────────────────────────────────┐
│                    SENTINEL CLUSTER (3 nodes)                  │
│  Sentinel 1     Sentinel 2     Sentinel 3                      │
└──────┬─────────────────┬──────────────────┬──────────────────┘
       │ monitors         │ monitors          │ monitors
       ▼                  ▼                   ▼
  ┌──────────┐       ┌──────────┐       ┌──────────┐
  │ PRIMARY   │       │ REPLICA 1│       │ REPLICA 2│
  └──────────┘       └──────────┘       └──────────┘

If PRIMARY is unreachable by QUORUM (majority) of Sentinels:
→ Sentinels VOTE on which replica to PROMOTE
→ Promoted replica becomes new primary
→ Other replicas reconfigure to follow new primary
→ Clients are notified of the new primary address

Sentinel provides: automatic failover, monitoring, notifications,
and acts as a "service discovery" for clients (clients ask a
Sentinel "who is the current primary?" instead of hardcoding
the primary's address — if failover occurs, Sentinel gives the
new address).
```

### Redis Cluster — Sharding + Replication

```
For HORIZONTAL SCALING of Redis beyond a single node (recall
Sharding and Consistent Hashing from Scalability notes):

Redis Cluster automatically SHARDS data across MULTIPLE REDIS
PRIMARIES using a FIXED 16,384 "hash slots" (slightly different
from pure consistent hashing but achieves similar goals):

hash_slot = CRC16(key) % 16384

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Primary A     │  │ Primary B     │  │ Primary C     │
│ Slots 0-5460  │  │ Slots 5461-   │  │ Slots 10923-  │
│               │  │ 10922         │  │ 16383         │
│ + Replica A   │  │ + Replica B   │  │ + Replica C   │
└──────────────┘  └──────────────┘  └──────────────┘

EACH shard is a primary-replica pair (or multiple replicas) for
HA within that shard.

CLIENT ROUTING: Clients know the slot mapping and route directly
to the correct shard. If they hit the wrong shard, they get a
MOVED redirect: "MOVED 5000 host:port" — telling the client
where to redirect. Smart clients cache this mapping to avoid
redirects on subsequent requests for the same key range.

THIS IS THE "Distributed Cache" pattern — full deep dive in
Topic 5.
```

---

## 6. Common Redis Patterns for System Design Interviews

### Pattern 1: Distributed Rate Limiting

```
(Connecting directly to the Rate Limiting topic, Scalability notes)

Using Redis INCR + EXPIRE (fixed window):
key = "rate:{user_id}:{window_start_minute}"
count = INCR key
if count == 1: EXPIRE key 60   (set TTL on first request)
if count > limit: REJECT

Using Redis Sorted Set (sliding window — more accurate):
ZADD rate:{user_id} {now_ms} {request_uuid}
ZREMRANGEBYSCORE rate:{user_id} 0 {now_ms - 60000}
count = ZCARD rate:{user_id}
EXPIRE rate:{user_id} 60
if count > limit: REJECT
```

### Pattern 2: Distributed Lock (Mutex)

```
ACQUIRING A LOCK:
SET lock:{resource} {owner_token} NX EX 30
↑                                  ↑   ↑
key                    NX=only if Not eXists  EX=expire in 30s

If the SET returns OK → lock ACQUIRED (we own it)
If the SET returns nil → lock already held by someone else

RELEASING A LOCK (must be atomic — use Lua script!):
The owner must verify they STILL OWN the lock before releasing
(prevents releasing a lock that expired and was re-acquired by
someone else):

EVAL "if redis.call('get',KEYS[1]) == ARGV[1] then
        return redis.call('del',KEYS[1])
      else
        return 0
      end" 1 lock:{resource} {owner_token}

WHY THE LUA SCRIPT: A GET then DEL as separate commands has a
RACE CONDITION (another process could acquire the lock BETWEEN
the GET confirming ownership and the DEL releasing it). Lua
scripts in Redis execute ATOMICALLY (single-threaded command
execution ensures no other command runs between Lua statements).

For multi-node high-availability distributed locks: Redlock
algorithm (extends this across 5+ Redis masters — more complex,
discussed in Distributed Cache topic).
```

### Pattern 3: Pub/Sub for Real-Time Messaging

```
(Connecting to WebSockets scaling pattern from Networking Fundamentals)

Publisher (Server A, handling User A's message):
PUBLISH channel:room:general "{type:message, text:Hello, from:user_a}"

Subscribers (all WS servers subscribed to this channel):
SUBSCRIBE channel:room:general
→ All subscribers receive the message IMMEDIATELY

USE CASES:
- Real-time chat fan-out (recall WebSocket scaling)
- Live notifications
- Cache invalidation broadcasting (covered in Cache Invalidation
  topic!) — a server that updates the database publishes an
  invalidation event; all OTHER servers subscribed to that
  channel know to invalidate their LOCAL cache entries

LIMITATION: Redis Pub/Sub is "fire and forget" — if a subscriber
is OFFLINE when a message is published, it MISSES that message.
No persistence, no replay. For reliable messaging with delivery
guarantees, Redis Streams (above) or a real message broker
(Kafka, RabbitMQ) is needed.
```

---

## 7. Real-World Usage

**Twitter/X:** Uses Redis extensively for the "home timeline" feature — each user's pre-computed timeline (the fan-out of tweets from followed users) is stored in a Redis List per user. This is the "fan-out on write" design mentioned in the Back-of-Envelope Estimation worked example (Scalability notes) — Redis's List makes this efficient (LPUSH new tweet to up to ~1000-2000 followers' timeline Lists, LRANGE to render a user's timeline).

**GitHub:** Uses Redis for distributed locking (preventing concurrent background jobs from colliding), rate limiting API requests, caching rendered Markdown (expensive to render, frequently viewed), and Sidekiq job queues (Redis Lists as job queues — RPUSH to enqueue, BLPOP to dequeue workers).

**Snapchat:** Uses Redis for ephemeral storage (snaps are temporary — Redis's TTL mechanism is a natural fit), leaderboard features (Sorted Sets), and friend graph caching.

**Stack Overflow:** Uses Redis for caching highly-repeated page fragments (question pages, user profiles), vote counting (atomic INCR), and session storage — a great example from the "smart vertical scaling + targeted caching" philosophy discussed in the Horizontal vs Vertical Scaling topic.

---

## 8. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Redis primary crashes, replica    │ Asynchronous replication — the    │ Semi-synchronous WAIT for critical │
│ promoted but missing recent data  │ promoted replica hadn't received  │ writes; AOF persistence for        │
│                                  │ the last few writes               │ durability; accept small data loss │
│                                  │                                  │ for cache data (it's a cache —     │
│                                  │                                  │ next read is a miss, DB is truth)  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Redis OOM (Out of Memory) kills   │ No maxmemory limit set; Redis     │ ALWAYS set maxmemory and an        │
│ the process unexpectedly          │ grows until OS kills it           │ appropriate eviction policy        │
│                                  │                                  │ (allkeys-lru for pure cache use)   │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Thundering herd on Redis restart  │ All keys lost; every request is   │ Cache warming (Topic 6); Sentinel  │
│ (cache cold after failover)       │ a cache miss simultaneously        │ for fast failover; persistent      │
│                                  │                                  │ RDB for fast restart               │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Lock not released (deadlock)      │ Lock holder crashes before        │ ALWAYS set a TTL on locks (the    │
│                                  │ releasing; lock has no TTL         │ EX in SET NX EX ensures automatic │
│                                  │                                  │ expiry even on holder crash)        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Hot key bottleneck                │ One key (e.g., a viral product)   │ Local in-process caching for the   │
│ (single Redis shard overloaded    │ receives millions of requests/sec  │ hottest keys; key sharding ("split │
│ for one key)                      │ — all going to ONE Redis shard     │ a hot key across N keys with a     │
│                                  │                                  │ random suffix, round-robin read")  │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 9. Interview Quick-Fire Answers

**Q: Why is Redis single-threaded but still extremely fast?**
A: Redis processes all commands on a single thread using a non-blocking event loop. This is fast because all commands execute entirely in MEMORY — no disk I/O on the critical path — so each command completes in microseconds. At that speed, thread-switching overhead would actually HURT performance. A single thread executing simple in-memory operations can handle 100K-1M commands/second. Redis 6.0+ added threaded network I/O for reading/writing to many clients simultaneously, while keeping command EXECUTION single-threaded.

**Q: What's the difference between Redis RDB and AOF persistence?**
A: RDB takes periodic SNAPSHOTS of all data to disk — fast to restart (load one file), but can lose up to the last snapshot interval of writes on crash. AOF logs every WRITE COMMAND to a file as it happens (like a WAL — same mechanism as database replication!), allowing near-zero data loss at the cost of a larger file and slower restart (must replay all commands). Most production setups use BOTH: RDB for fast restart + AOF for minimal data loss since the last snapshot.

**Q: How would you implement a leaderboard using Redis?**
A: Use a Redis Sorted Set (ZSET) — add each player with their score using `ZADD leaderboard {score} {player_id}`. Retrieve the top N players with `ZREVRANGE leaderboard 0 N-1 WITHSCORES` (descending by score). Get a player's rank with `ZREVRANK leaderboard {player_id}`. Update a score with `ZADD leaderboard {new_score} {player_id}` (overwrites the existing score). All operations are O(log n) via the internal skiplist structure.

**Q: How does Redis implement atomic operations, and why does that matter?**
A: Redis processes commands one-at-a-time on a single thread, so by definition no two commands can interleave — each command is inherently atomic. For operations requiring MULTIPLE commands to be atomic as a unit (e.g., check-and-set for distributed locks), Redis supports Lua scripts that execute atomically as a single command block and MULTI/EXEC transactions. This matters because application-level "read, modify, write" sequences have race conditions under concurrent access — the same problem ACID transactions solve for databases (Databases notes) — Redis's single-threaded model solves it at the cache level.

---
---

# TOPIC 4: Cache Invalidation

---

## 1. What Problem Does Cache Invalidation Solve?

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

A cache stores a **COPY** of data. The ORIGINAL data (in the database or source system) can CHANGE. When it does, the cached COPY becomes **STALE** — it no longer reflects reality.

**Cache invalidation** is the process of ensuring that when the source data changes, the cached copy is either REMOVED or UPDATED so that subsequent reads don't serve incorrect data.

```
THE STALENESS PROBLEM:

t=0: Product 789 price = ₹80,000
     Cache:    product:789 = {price: 80000, ...}  ← matches DB

t=1: Business updates price: DB now says product:789 price = ₹75,000
     Cache:    product:789 = {price: 80000, ...}  ← STALE! wrong price

t=2: User checks product 789 price
     → Cache HIT → served ₹80,000 ← WRONG! User shown old price
     → If user buys based on this, there's a price discrepancy!

HOW LONG DOES THIS STALE STATE LAST?
Without explicit invalidation: until the TTL EXPIRES (if one
was set — could be hours/days for product data)
With TTL only: up to TTL seconds of staleness after every update
With explicit invalidation: near-zero staleness (invalidated the
moment the DB update happens)

THE DIFFICULTY: At scale, caches are DISTRIBUTED across many
servers (more on this in Topic 5: Distributed Cache). When
product 789 is updated in the database, which cache SERVERS
have a copy of this entry? How do you efficiently find and
invalidate ALL of them WITHOUT broadcasting to every cache
server for every single update?
```

---

## 2. Invalidation Strategy 1: TTL-Based Expiry

```
APPROACH: Set a Time-To-Live on every cache entry. Accept that
cached data might be stale for UP TO TTL seconds.

SET product:789 "{price:80000,...}" EX 300  ← expires in 5 minutes

AFTER THE DB UPDATE: Don't do anything special. Wait for TTL to
expire. The NEXT read after expiry will be a cache miss, fetching
fresh data.

TIMELINE:
t=0:   Cache set with TTL=300s
t=100: DB updated (price changed)
t=100-300: Cache serves STALE data (up to 200 more seconds!)
t=300: TTL expires → next read is cache miss → DB queried → fresh data

PROS:
✅ EXTREMELY SIMPLE — no invalidation code needed anywhere
✅ ROBUST — works even if the invalidation mechanism fails or
   is forgotten somewhere (the TTL provides a "fallback
   correctness guarantee" — staleness is bounded)
✅ NO COUPLING between the write path and the cache layer

CONS:
❌ GUARANTEED STALENESS for up to TTL seconds after every update
❌ IF TTL IS SHORT (e.g., 1 second): low staleness but HIGH
   cache miss rate → defeats the purpose of caching
❌ IF TTL IS LONG (e.g., 1 day): good hit rate but potentially
   very stale data after updates

CHOOSING THE RIGHT TTL:
The TTL choice encodes YOUR BUSINESS TOLERANCE for stale data:
- Product PRICE: maybe 5 minutes (pricing errors are costly,
  but not worth the complexity of instant invalidation)
- User PROFILE: maybe 1 hour (names/emails change rarely)
- Stock QUANTITY (for the last 10 units): maybe 30 seconds
  or even no caching (overselling has real consequences)
- Static CONTENT (CSS, JS bundles): 1 year + immutable URLs
  (never stale — the URL changes when content changes, recall
  the CDN topic from Networking Fundamentals)

TTL IS THE BASELINE — always set one. Other strategies REDUCE
staleness beyond what TTL alone provides, but TTL is the safety
net that ensures correctness is EVENTUALLY restored even if
all other invalidation mechanisms fail.
```

---

## 3. Invalidation Strategy 2: Event-Driven Invalidation (Active Invalidation)

```
APPROACH: When the database is updated, IMMEDIATELY (or very
soon after) invalidate the corresponding cache entries.

THE CACHE-ASIDE WRITE PATH (recall Topic 1):
1. App updates DB:    UPDATE products SET price=75000 WHERE id=789
2. App invalidates:   DEL product:789  (remove from cache)
3. Next read:         Cache MISS → fetch fresh data from DB → re-cache

TIMELINE:
t=0:   Cache contains {price: 80000}
t=100: DB updated; cache entry DELETED (t=100, not t=300!)
t=101: First read → cache MISS → DB queried → cached again with new price
t=102+: Cache serves correct {price: 75000}

Staleness window: ~1-100ms (the time between DB write and cache
delete) vs up to 300 seconds with pure TTL!

WHY DELETE RATHER THAN UPDATE?
Recall from Topic 1: directly writing the new value to the cache
creates a RACE CONDITION under concurrent writes. DELETING is
safer — the next read will fetch WHATEVER IS NOW CURRENT in the
database, regardless of update ordering. This is the standard
recommendation.
```

### The Distributed Invalidation Challenge

```
PROBLEM IN A DISTRIBUTED SYSTEM:
Multiple APP SERVERS each have the "invalidate on write" pattern.
But your cache is DISTRIBUTED across multiple Redis nodes (recall
Redis Cluster, or simply multiple independent cache servers in
different regions — Geo-distribution, Scalability notes).

                    APP SERVER A
                    (handles write)
                         │
                         ├──── DEL product:789 ──▶ Cache Server 1 ✓
                         │
                         │    Cache Server 2 still has stale entry!
                         │    Cache Server 3 still has stale entry!
                         │

App Server A's write only directly deleted from the cache server
IT was talking to. Other cache servers might hold stale copies.

SOLUTION: Publish an invalidation EVENT via a message channel
(e.g., Redis Pub/Sub — recall from Topic 3!):

App Server A:
1. Updates DB
2. PUBLISH cache_invalidation "product:789"

Cache invalidation listener (running on ALL app servers or all
cache servers):
1. Receives "product:789" event
2. Executes DEL product:789 on ITS OWN LOCAL CACHE

RESULT: ALL servers receive the invalidation event and delete
their copies — near-simultaneous invalidation across the entire
distributed cache.

This is exactly the pattern Memcache used at Facebook scale —
"McSqueal" listened to MySQL's binlog (recall CDC and WAL from
Databases notes!) to detect DB changes and broadcast cache
invalidations, rather than relying on each application write
path to trigger invalidation.
```

---

## 4. Invalidation Strategy 3: Write-Through (Cache Always Matches DB)

```
Recall from Topic 1: in the write-through strategy, EVERY WRITE
updates BOTH the cache AND the database synchronously. This means
the cache NEVER becomes stale for written data — staleness is
eliminated at the cost of slower writes.

The cache is ALWAYS consistent with the database after a write.
NO explicit invalidation needed — because the cache is updated
IN THE SAME OPERATION as the DB write, there's no window where
they disagree.

TRADEOFF (already covered in Topic 1): slower writes, and
potentially caches data that won't be read again — so this
"solves" invalidation at the cost of write performance and
potential cache pollution.
```

---

## 5. Invalidation Strategy 4: Event Streaming via CDC

```
For complex systems where many different services write to the
SAME database, and each service can't reliably "remember" to
invalidate the cache on every write path:

CDC (Change Data Capture — recall from Databases notes,
Data Warehousing topic) reads the DATABASE'S OWN REPLICATION
LOG (WAL/binlog) to detect EVERY change, regardless of which
service caused it.

┌─────────────────────────────────────────────────────────────┐
│   Service A  Service B  Service C  (all can write to DB)     │
│       │          │          │                                  │
│       └──────────┴──────────┘                                  │
│                  │  writes                                     │
│                  ▼                                             │
│           ┌───────────┐                                        │
│           │  Database   │                                       │
│           │  (primary)  │                                       │
│           └──────┬──────┘                                       │
│                  │  WAL / binlog                                │
│                  ▼                                             │
│           ┌───────────┐                                        │
│           │  CDC Tool    │ (Debezium, Maxwell, etc.)            │
│           │  (reads WAL) │                                      │
│           └──────┬──────┘                                       │
│                  │  change events                               │
│                  ▼                                             │
│           ┌───────────┐                                        │
│           │  Kafka /    │                                       │
│           │  Redis Pub  │                                       │
│           │  Sub         │                                      │
│           └──────┬──────┘                                       │
│                  │                                             │
│                  ▼                                             │
│           ┌───────────┐                                        │
│           │  Cache       │                                      │
│           │  Invalidation│ DEL the affected cache keys          │
│           │  Service     │                                      │
│           └───────────┘                                        │
└─────────────────────────────────────────────────────────────┘

BENEFIT: Cache invalidation is GUARANTEED regardless of which
service made the DB change — no service "forgets" to invalidate.
The CDC pipeline is the SINGLE, AUTHORITATIVE source of
"something changed, go invalidate."

TRADEOFF: A few seconds of staleness (CDC latency) and
operational complexity of the CDC pipeline — worth it for large
distributed systems where service-level coordination for cache
invalidation is impractical.
```

---

## 6. The Cache Stampede / Thundering Herd Problem

```
SCENARIO: A VERY POPULAR cached item (e.g., the homepage of a
major site) has its TTL EXPIRE — or is INVALIDATED — at the
SAME MOMENT that thousands of requests are coming in for it.

TIMELINE:
t=0:    product:789 cache entry expires (TTL=0)
t=1ms:  Request #1 arrives → cache MISS → starts DB query
t=1ms:  Request #2 arrives → cache MISS → starts DB query
t=1ms:  Request #3 arrives → cache MISS → starts DB query
...
t=1ms:  Request #10,000 arrives → cache MISS → starts DB query

ALL 10,000 requests simultaneously hit the database with the
SAME query ("get product 789") — the database is suddenly
handling 10,000 concurrent queries for data that JUST ONE query
would be sufficient to get. This can OVERWHELM and potentially
CRASH the database — a "thundering herd" or "cache stampede."

SOLUTION 1: MUTEX LOCK / PROBABILISTIC EARLY EXPIRATION

MUTEX LOCK approach:
When a cache miss is detected, ONLY ONE REQUEST is allowed to
fetch from the DB and repopulate the cache:

1. Request detects miss
2. Try to acquire a distributed lock: SET lock:product:789 "1" NX EX 5
3. If lock ACQUIRED → fetch from DB, populate cache, release lock
4. If lock NOT acquired (another request is already fetching):
   Wait briefly and retry cache → probably a HIT by then

OTHER REQUESTS wait (with brief retries/backoff) for the "winner"
to populate the cache, rather than all hammering the DB.

SOLUTION 2: PROBABILISTIC EARLY EXPIRATION (PER algorithm):
Don't wait for TTL to FULLY expire — start stochastically
refreshing items BEFORE they expire, based on the remaining TTL
and the expected re-computation time:

if (now - last_updated + beta * log(random())) > TTL:
    # probabilistically refresh before expiry
    # only a fraction of requests trigger early refresh

This ensures the cache entry is REFRESHED before it expires,
eliminating the "all hit at once" window entirely.

SOLUTION 3: STALE-WHILE-REVALIDATE (recall CDN topic!)
Serve the STALE cached item while ONE background request
refreshes it asynchronously:

Cache entry structure: {value: ..., expire_at: T, stale_until: T+30}
If now > expire_at but now < stale_until:
  → Serve stale value immediately (fast, correct enough)
  → Asynchronously trigger ONE background re-fetch and re-cache
If now > stale_until:
  → Must block and fetch fresh (now truly expired)

Users ALWAYS get a fast response (the stale value), and the
cache gets refreshed before the stale window closes — nobody
experiences a cache-miss latency spike.
```

---

## 7. The Dual Write Problem — Why Invalidation Is Hard

```
ATOMIC CHALLENGE:
A write involves TWO operations that can't ATOMICALLY succeed
or fail together:
  1. Write to the DATABASE
  2. Invalidate the CACHE

FAILURE SCENARIO A (Write DB succeeds, Cache DEL fails):
→ DB has new price (₹75,000)
→ Cache still has old price (₹80,000)
→ Users see STALE price until TTL expires (which is why TTL
   as a FALLBACK is always essential)

FAILURE SCENARIO B (Cache DEL succeeds, Write DB fails):
→ Cache has NO entry (just deleted)
→ DB still has old price (₹80,000)
→ Next read: cache MISS → fetches ₹80,000 from DB → re-caches ₹80,000
→ Actually FINE — the DB is the source of truth, the cache
   re-populated from it correctly. The DB write failing means
   the update itself didn't happen, so ₹80,000 IS currently
   correct.

→ LESSON: DO THE DB WRITE FIRST, THEN INVALIDATE THE CACHE.
   Never invalidate the cache BEFORE the DB write — that creates
   a window where the cache is empty BUT the DB still has stale
   data, and a concurrent read will REPOPULATE THE CACHE WITH
   THE OLD DATA (because the DB write hasn't happened yet),
   THEN the DB write happens, and now the cache has the OLD
   value again even after the update. This is a subtle but real
   ordering bug.

CORRECT ORDER ALWAYS:
1. Write to database (source of truth updated)
2. THEN invalidate cache (cache entry removed/updated)

Even if step 2 fails, TTL guarantees eventual correctness.
Even if steps 1 and 2 both fail, the system is in the same
state as before the attempted update (consistent).
```

---

## 8. Real-World Usage

**Amazon DynamoDB Accelerator (DAX):** A write-through cache purpose-built for DynamoDB — writes go to BOTH DAX (cache) and DynamoDB simultaneously. The cache is never stale for items that were written through it. Invalidation is automatic (tightly coupled write-through design). Used heavily for DynamoDB workloads with read-heavy patterns that don't need the full flexibility of Redis.

**Varnish Cache (used by many media/CDN setups):** Uses a "surrogate keys" (cache tags) invalidation system — exactly the CDN-level surrogate key pattern from the CDN topic of Networking Fundamentals. A single `PURGE` request with a surrogate key tag invalidates ALL cached responses tagged with that key across ALL Varnish nodes — a production-grade distributed invalidation mechanism.

**Instagram (relevant cache invalidation at scale):** Uses a system called "Polaris" — when a database write occurs, the primary's replication log (binlog) is tailed via CDC, and invalidation events are published. Cache servers subscribed to these events invalidate their local copies. This ensures that even across hundreds of cache servers in multiple data centers, cache invalidation propagates correctly without requiring every write path to explicitly manage it — the SAME "CDC for cache invalidation" pattern described above.

---

## 9. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Cache stampede / thundering herd │ Popular cache entry expires;     │ Mutex lock (only one DB fetch at  │
│                                  │ many concurrent requests all      │ a time); stale-while-revalidate;  │
│                                  │ become cache misses simultaneously│ probabilistic early expiration    │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Cache invalidation NOT triggered  │ A code path that updates the DB   │ CDC-based invalidation as a       │
│ after a write (code forgot to    │ forgot to invalidate; or an        │ safety net — catches ALL DB        │
│ invalidate)                     │ external system updated the DB     │ changes regardless of which code  │
│                                  │ without going through the app      │ path caused them                   │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Cache repopulated with OLD data   │ Cache invalidated BEFORE the DB   │ ALWAYS write DB first, then        │
│ after an update (subtle ordering  │ write; a concurrent read happened  │ invalidate cache; never the        │
│ bug)                            │ in between and re-populated the    │ reverse                             │
│                                  │ cache from the not-yet-updated DB  │                                     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Partial distributed invalidation  │ Invalidation message lost or       │ CDC + reliable message broker      │
│ (some nodes invalidated, others   │ subscriber not running on some     │ (Kafka with at-least-once          │
│ still have stale data)           │ cache nodes at the time of event   │ delivery); TTL as fallback          │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 10. Interview Quick-Fire Answers

**Q: Why is cache invalidation considered one of the hardest problems in computer science?**
A: Because correctness requires ATOMICALLY keeping two separate systems (cache and database) in sync — but distributed systems don't offer true cross-system atomicity. The write to DB and the invalidation of the cache are TWO SEPARATE OPERATIONS that can each independently succeed or fail. Getting the ordering right (always write DB first, then invalidate), handling failures of either step gracefully (TTL as a safety net), and propagating invalidations to ALL nodes in a distributed cache (without broadcasting to every cache for every write) all require careful design.

**Q: What is a cache stampede and how do you prevent it?**
A: A cache stampede occurs when a popular cached item expires (or is invalidated) at the same moment many concurrent requests arrive for it. All of them experience a cache miss simultaneously and all race to the database — potentially overwhelming it with identical queries. Prevention strategies: mutex locking (only ONE request fetches from DB, others wait and retry the cache); stale-while-revalidate (serve the stale value while asynchronously refreshing in the background); and probabilistic early expiration (proactively refresh entries slightly BEFORE they expire, eliminating the expiry window entirely).

**Q: Should you invalidate the cache before or after updating the database?**
A: Always update the DATABASE FIRST, then invalidate the cache. Invalidating the cache first creates a dangerous window: the cache is empty, but the DB still has old data. A concurrent read during this window causes a cache miss, fetches the (still-old) data from the DB, and re-populates the cache with the OLD value — which then stays until TTL, even after the DB write succeeds. Writing DB first means any concurrent read during the brief post-write/pre-invalidation window gets a slightly stale cache hit, which is far better — and if the invalidation step fails, TTL bounds the staleness.

**Q: How does CDC help with cache invalidation at scale?**
A: In large distributed systems, many different services and code paths write to the same database. Relying on each write path to "remember" to invalidate the cache is fragile — one forgotten invalidation on any code path causes staleness. CDC (Change Data Capture) tails the database's own replication log (WAL/binlog) and emits an event for EVERY change, regardless of which service caused it. A dedicated cache invalidation service subscribes to these events and invalidates the corresponding cache keys — a centralized, reliable mechanism that catches all DB changes with no coupling to individual service write paths.


---
---

# TOPIC 5: Distributed Cache

---

## 1. What Problem Does a Distributed Cache Solve?

A single-node cache (one Redis server) has hard limits: fixed RAM (one machine's worth), a single point of failure (if it dies, the cache is gone and the stampede hits your database), and a physical ceiling on throughput (one machine can only serve so many requests/second).

**Distributed caching** solves all three by spreading cache data AND the serving of cache requests across MULTIPLE machines, giving you: larger total cache capacity, no single point of failure, and higher aggregate throughput.

```
SINGLE-NODE CACHE LIMITS:
┌─────────────────────────────────────────────────────────────┐
│  Redis Single Node                                            │
│  Max RAM: ~256-512GB (one machine)                            │
│  Max throughput: ~1M ops/sec                                  │
│  Availability: if this dies → total cache outage              │
└─────────────────────────────────────────────────────────────┘

DISTRIBUTED CACHE (e.g., 6-node Redis Cluster):
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Node 1    │ │ Node 2    │ │ Node 3    │
│ 256GB RAM │ │ 256GB RAM │ │ 256GB RAM │
│ ~333K rps │ │ ~333K rps │ │ ~333K rps │
└──────────┘ └──────────┘ └──────────┘
    Total: 768GB RAM, ~1M rps, N-1 fault tolerance

EACH NODE HOLDS A SUBSET OF THE DATA (sharding — exactly the
same concept as Database Sharding, Databases notes Topic 3,
applied to the cache layer).
```

---

## 2. Data Distribution — How Keys Are Mapped to Nodes

### Consistent Hashing (Recall from Scalability Notes)

```
The SAME consistent hashing algorithm from the Scalability &
Load Balancing notes (Consistent Hashing topic) applies directly
to distributed caches.

For Memcached (client-side consistent hashing via libketama):
node = consistent_hash_ring.find_node(key)
→ Client computes which node to talk to LOCALLY
→ No central coordinator needed
→ When a node is added/removed: only ~1/N of keys "move"
  (minimizing cache-miss spike — recall the 1/N property)

For Redis Cluster (server-side with 16,384 hash slots):
slot = CRC16(key) % 16384
→ Each Redis node owns a range of slots
→ Client can be told (via MOVED redirect) which node handles
  a given slot
→ Adding/removing nodes involves migrating slot RANGES between
  nodes — conceptually similar to range-based sharding
```

### Client-Side vs Server-Side Distribution

```
CLIENT-SIDE DISTRIBUTION (Memcached + consistent hashing):

  App (contains consistent hash ring) ──▶ Node A (if key hashes there)
                                      ──▶ Node B (if key hashes there)
                                      ──▶ Node C (if key hashes there)

Pros: No extra hop to a routing layer; client routes directly
Cons: ALL clients must have the SAME view of the ring (cluster
      membership — adding/removing a node must be coordinated
      across all client processes simultaneously)

SERVER-SIDE DISTRIBUTION (Redis Cluster):

  App ──▶ Any Redis Node ──MOVED──▶ Correct Node
  (smart clients cache slot map; dumb clients just redirect)

Pros: Clients don't need to know the cluster topology in detail
Cons: Extra hop on first request (MOVED redirect); slightly more
      network overhead
```

---

## 3. Replication Within Distributed Caches

```
A distributed cache with ONLY sharding (no replication) is still
vulnerable: if Node 2 dies, all cache entries whose keys hash to
Node 2 are LOST.

SOLUTION: Each shard has a PRIMARY and at least ONE REPLICA:

┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Primary A     │    │ Primary B     │    │ Primary C     │
│ Slots 0-5460  │    │ Slots 5461-   │    │ Slots 10923-  │
│               │    │ 10922         │    │ 16383         │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │ replicates         │ replicates         │ replicates
       ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Replica A     │    │ Replica B     │    │ Replica C     │
└──────────────┘    └──────────────┘    └──────────────┘

If Primary B fails → Replica B is automatically promoted
→ Only Shard B's data is lost (if async replication and any
  lag) → ~1/N of total data potentially affected, NOT all data
→ For a cache (not a source of truth), this is acceptable:
  the next reads for those keys are cache misses → DB queries
  → re-cache on the promoted replica

REDIS CLUSTER DOES THIS AUTOMATICALLY — same Sentinel-like
automatic failover, but built into the cluster protocol.
```

---

## 4. The Multi-Layer Cache Architecture

```
Production systems often have MULTIPLE LEVELS of caching, each
closer to the computation and faster (connecting to the CPU
cache discussion in Topic 7):

┌─────────────────────────────────────────────────────────────┐
│ LAYER 1: IN-PROCESS / LOCAL MEMORY CACHE                      │
│  Location: Inside the application process's own RAM           │
│  Latency: Sub-microsecond (no network — just a hash table     │
│           in the app's own memory address space)              │
│  Size: Small (bounded by app's heap — typically MBs, not GBs)│
│  Example: Guava Cache (Java), functools.lru_cache (Python),   │
│           a simple Dictionary/HashMap in any language         │
│  LIMITATION: Each app server has its OWN cache —             │
│              NOT SHARED across multiple app server instances! │
│              A cache hit on Server 1 doesn't help Server 2.  │
│              (Called "L1" or "local" cache in this context)  │
└─────────────────────────────────────────────────────────────┘
                              │ miss
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ LAYER 2: DISTRIBUTED SHARED CACHE                             │
│  Location: Separate Redis/Memcached cluster (over network)    │
│  Latency: ~0.5-2ms (a local network round trip)              │
│  Size: Large (GBs to TBs across the cluster)                  │
│  Example: Redis Cluster, Memcached cluster                    │
│  SHARED: ALL app servers read from / write to the SAME cache  │
│          → a cache hit from one server's request benefits all │
└─────────────────────────────────────────────────────────────┘
                              │ miss
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ LAYER 3: DATABASE                                             │
│  Location: Separate DB servers (over network)                 │
│  Latency: 5-50ms (query execution + disk I/O)                │
│  This is the source of truth                                  │
└─────────────────────────────────────────────────────────────┘

REAL-WORLD PATTERN: For a HOT KEY (e.g., homepage data for a
viral product, accessed 100K times/sec):

- With ONLY Redis (shared cache): 100K requests/sec × 1ms = high
  load on Redis, 100K network round trips to Redis per second
- Add LOCAL IN-PROCESS cache (L1): most requests served from
  the app's own memory in <1 microsecond; Redis only hit when
  L1 misses (e.g., first request from THIS app server, or after
  a brief local TTL expires). Redis load drops dramatically.

TRADEOFF: Local caches introduce INCOHERENCE between app servers
— Server 1's local cache might have an OLDER version than
Server 2's local cache (they each independently cached at
different times, and might independently invalidate at different
times). For FREQUENTLY CHANGING data, use ONLY shared cache
(always consistent across servers). For STABLE data (config,
feature flags, reference data), local caches with short TTLs
(seconds to minutes) are very effective.
```

---

## 5. Redlock — Distributed Locking Across Multiple Redis Masters

```
The single-node Redis lock (SET NX EX from Topic 3) fails if
that Redis node goes down. For HIGH-AVAILABILITY distributed
locking, the Redlock algorithm (proposed by Redis's creator
Salvatore Sanfilippo) acquires a lock on MULTIPLE Redis masters:

SETUP: 5 independent Redis master nodes (no replication between
them — they are completely independent for Redlock's purposes)

ACQUIRING A LOCK:
1. Record start time
2. Try to acquire the lock on ALL 5 masters:
   SET lock:resource {owner_token} NX PX 30000
   (PX = expire in 30,000 milliseconds = 30 seconds)
3. The lock is ACQUIRED if AND ONLY IF:
   a. The lock was acquired on a MAJORITY (≥3 of 5) masters
   b. The TOTAL TIME taken to acquire was less than the lock's
      TTL (so the lock hasn't already expired on early masters)
4. The lock's EFFECTIVE duration = TTL - (time elapsed to acquire)
   (the remaining safe window before the earliest-set lock expires)

RELEASING:
Del the lock key on ALL 5 masters simultaneously (even the
ones where acquisition "failed" — just in case).

WHY 5 MASTERS?
With 3 of 5 needed (majority), even if 2 masters FAIL entirely:
- The remaining 3 can still form a quorum
- No single master's failure prevents lock acquisition
- No two clients can BOTH hold the lock simultaneously (they'd
  each need 3 of 5, but 3+3=6 > 5 — impossible)

CONTROVERSY: Redlock is still debated — distributed systems
researchers (e.g., Martin Kleppmann of DDIA fame) have argued
it has subtle safety issues under certain clock skew + network
delay combinations. For truly critical mutual exclusion in
financial systems, purpose-built consensus-based systems
(ZooKeeper, etcd using Raft — same Raft mentioned in the
Replication failover discussion) are more formally correct.
For most practical applications (preventing double-processing
of a background job, rate limiting, etc.), Redlock is
sufficient and widely used.
```

---

## 6. Real-World Usage

**Netflix (EVCache):** A globally-distributed cache built ON TOP OF Memcached — one of the largest cache deployments in the world. EVCache replicates cache entries ACROSS REGIONS (not just across nodes within a region) — so a user served by the Mumbai region has their cached data available in Mumbai's EVCache, even if the "source" write happened in the Virginia region. This is "cross-region cache replication" — a distributed cache layer that supports the multi-region architecture from the Geo-distribution topic.

**Facebook (Memcache at scale):** The famous 2013 "Scaling Memcache at Facebook" paper describes a hierarchical distributed cache with local clusters (for each data center) and regional pools (shared across multiple clusters in a region), with a custom UDP-based protocol for read requests (lower overhead than TCP for small cache GET operations — recall TCP vs UDP from Networking Fundamentals!), and a system called "lease tokens" to handle cache stampedes — an early version of the mutex-lock-on-cache-miss pattern described above.

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Cache node failure causes         │ Sharding with no replication —    │ Each shard should have ≥1         │
│ 1/N of all cache traffic to hit   │ all keys for that shard are        │ replica; automatic failover        │
│ the database suddenly             │ unavailable                        │ (Redis Cluster) + cache warming    │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Client-side ring inconsistency     │ Different app servers have          │ Use a centralized configuration    │
│ (different servers route same key  │ different views of which nodes     │ service (ZooKeeper, etcd, or        │
│ to different cache nodes)          │ exist (one updated its ring,        │ Redis Cluster's built-in slot       │
│                                  │ another hasn't yet)                │ mapping) for authoritative         │
│                                  │                                  │ cluster membership                  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Hot key overwhelms a single shard  │ A "celebrity key" (viral product,  │ Key sharding: "product:789#0",     │
│                                  │ homepage data) generates requests  │ "product:789#1", ... "#N-1" —      │
│                                  │ far exceeding one node's capacity  │ split the hot key across N shards, │
│                                  │                                  │ read from a RANDOM shard; or use   │
│                                  │                                  │ local in-process (L1) caching to   │
│                                  │                                  │ absorb the load before Redis        │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: How does a distributed cache handle node failures without losing all cached data?**
A: By combining SHARDING (data distributed across nodes) with REPLICATION (each shard has a primary + at least one replica). If a primary node fails, its replica is automatically promoted — only that shard's data is affected. For a cache (not a source of truth), the worst-case impact is a cache-miss spike for keys that were on the failed shard — the database absorbs these temporarily until the promoted replica re-warms. With proper replication, no cache data is permanently lost unless both primary AND replica fail simultaneously.

**Q: What is the "hot key" problem and how do you solve it in a distributed cache?**
A: A hot key is a single cache key that receives an extremely disproportionate share of requests (e.g., a viral product's data). In a distributed cache, all requests for that key go to ONE shard — potentially overwhelming that node even if the rest of the cluster is lightly loaded. Solutions: local in-process (L1) caching to absorb repeated reads within each app server before hitting Redis; or key sharding — split the hot key into N copies ("product:789#0" through "product:789#N-1"), read from a randomly chosen copy, so requests are distributed across N Redis nodes.

**Q: What's the difference between client-side and server-side cache distribution?**
A: In client-side distribution (e.g., Memcached with libketama), the application maintains the consistent hash ring and routes each request DIRECTLY to the correct cache node without any intermediary — lower latency, no extra hop, but requires all clients to agree on the cluster topology simultaneously. In server-side distribution (Redis Cluster), the client can connect to any node and gets a MOVED redirect to the correct node if needed — clients don't need to maintain the full topology, but there's an extra redirect on cache misses until clients learn the slot map. Smart clients cache the slot map locally, making subsequent requests direct.

---
---

# TOPIC 6: Cache Warming

---

## 1. What Problem Does Cache Warming Solve?

A COLD cache — one with no data in it — serves EVERY request as a cache miss, sending ALL traffic to the database simultaneously. This happens at:
- System startup (fresh deployment, Redis restart)
- After a cache node failure and failover
- After a new feature launches (new cache key namespace)
- After a full cache flush (forced invalidation)

```
THE COLD START PROBLEM:

At t=0 (cold cache), all requests are misses:
100,000 req/sec → 100,000 DB queries/sec!

Your database (designed for maybe 5,000-10,000 queries/sec under
normal load, with 95% cache hit rate) suddenly receives 20-30x
its normal load → database overwhelmed → timeouts → cascade
failure → site down.

EVEN WITH NO BUGS OR CONFIGURATION ERRORS, a cold start event
can take down an otherwise well-designed system.
```

---

## 2. Cache Warming Strategies

### Strategy 1: Proactive Pre-Warming (Before Going Live)

```
Before switching traffic to a new deployment (or restored cache),
proactively POPULATE the cache with data you KNOW will be needed:

APPROACH A: Replay recent access logs
  Read yesterday's/last hour's HTTP access log
  → Extract frequently-requested keys
  → Pre-fetch those items from the database
  → Populate the cache

  access_log | awk '{print $7}' | sort | uniq -c | sort -rn | head -10000
  → Get the top 10,000 most-requested URLs from yesterday
  → For each, pre-fetch and cache the corresponding data

APPROACH B: Programmatic "warm-up" job
  Before traffic switch, run a script that fetches known-hot items:

  HOT_PRODUCTS = [top 1000 products by last 7 days' sales]
  for product_id in HOT_PRODUCTS:
      data = db.query("SELECT * FROM products WHERE id=?", product_id)
      cache.set(f"product:{product_id}", serialize(data), ttl=3600)

  HOT_USERS = [users with active sessions in last hour]
  for user_id in HOT_USERS:
      data = db.query("SELECT * FROM users WHERE id=?", user_id)
      cache.set(f"user:{user_id}", serialize(data), ttl=1800)

BENEFIT: When traffic actually hits the new deployment, the cache
ALREADY has the hot data → near-normal hit rate from the first
request, not after minutes of gradual warming.
```

### Strategy 2: Gradual Traffic Shifting (Blue-Green / Canary)

```
Instead of switching 100% of traffic to the new (cold) deployment
at once, gradually shift a SMALL PERCENTAGE:

t=0:   5% of traffic → new deployment (cold cache)
       95% → old deployment (warm cache)
       → New deployment's cache starts warming with 5% of traffic

t=5min: 20% → new, 80% → old
       → Cache hit rate on new deployment improves as it warms

t=15min: 50% → new, 50% → old
t=30min: 100% → new (cache now reasonably warm from 30 min of 50% traffic)

BENEFIT: The new deployment's database load during the cold-
warming period is PROPORTIONAL to its traffic share — at 5%
traffic, it has 5% of normal load going to the DB (misses),
well within capacity.

This DIRECTLY CONNECTS to the Auto-scaling topic's "gradual
traffic ramp-up" pattern (Scalability notes) — the same
principle (avoid sudden 100% load transitions) applied to
cache warming.
```

### Strategy 3: Lazy Warming (Organic, with Protection)

```
Accept that the cache will warm UP GRADUALLY as real traffic
comes in — but PROTECT the database during this warm-up:

1. RATE LIMITING on cache misses: limit how many concurrent
   "miss → DB fetch" operations can happen simultaneously
   (e.g., semaphore with max 100 concurrent DB fetches at any
   time — more misses queue up or get a "retry later" response)

2. CIRCUIT BREAKER: if the database's response time suddenly
   spikes (due to cold-start miss overload), trip a circuit
   breaker that temporarily REJECTS cache-miss DB fetches
   (returns a 503 or serves stale data) rather than allowing
   continued DB overload

3. MUTEX LOCK ON MISSES (staggered re-fetch): the stampede
   protection from Cache Invalidation topic — only ONE request
   per key does the DB fetch, others wait for the cache to
   populate. As the cache warms, fewer and fewer concurrent
   fetches are needed.

BENEFIT: Simplest operationally (no pre-warming job to build
and maintain). DOWNSIDE: there IS a warm-up period during
which performance degrades (higher latency, more DB load) —
acceptable for some systems, not for others.
```

### Strategy 4: Persistent Cache (Survive Restarts)

```
If the warming problem is specifically about RESTARTS (not new
deployments), configure Redis with PERSISTENCE (RDB snapshots
and/or AOF — recall from Topic 3) so that on restart, Redis
RELOADS its previous state from disk:

RESTART TIMELINE WITH PERSISTENCE:
t=0: Redis restart initiated
t=0 → t=30s: Redis loads RDB snapshot from disk
              (during this time, Redis serves no requests —
               client connections queue up)
t=30s: Redis fully loaded with ~all previous cache data
t=30s+: Cache HIT RATE returns to normal immediately

vs WITHOUT PERSISTENCE:
t=0: Redis restart
t=0+: Redis is EMPTY — cold cache, every request a miss
t=30min+: Cache slowly warms to normal hit rate

Persistence-based warm-up works best for PURE CACHE USE CASES
where "recently hot data" is a good proxy for "immediately
future hot data" — which is true for most stable workloads.
Less effective for workloads where traffic patterns SHIFT
significantly (the cached data from before the restart might
not be relevant after a long downtime).
```

---

## 3. Real-World Usage

**Netflix:** Has dedicated "warm-up" procedures as part of regional failover runbooks. When traffic is rerouted to a previously cold region (e.g., during a disaster recovery scenario), EVCache (their distributed cache) is pre-warmed using "cache fill" jobs that replay recent activity patterns — preventing a cache-cold failover region from collapsing under the full load of an entire region's traffic.

**Google:** For services that run "stateless" compute in containers (relevant to Kubernetes deployments), session/request-level caches warm up naturally per-container on startup. But SHARED caches (Memcache/Bigtable-backed caches) are pre-warmed via "cache warming" traffic before new container versions receive production traffic in their gradual rollout — a specific type of canary deployment.

---

## 4. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Database overwhelmed at          │ Deployment switches 100% traffic  │ Gradual traffic shifting (canary/ │
│ deployment time (cold start)     │ to new cold cache instantly        │ blue-green); proactive pre-warming │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Cache warmed with WRONG data      │ Pre-warming job fetches outdated  │ Warm from the SAME data source     │
│ (stale from pre-warming)          │ data (e.g., uses a stale export   │ as the live system; include        │
│                                  │ file rather than live DB)          │ short TTLs even on pre-warmed      │
│                                  │                                  │ entries so they refresh quickly     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Pre-warming job itself overloads  │ Pre-warming fetches too many items │ Rate-limit the pre-warming job;    │
│ the database (ironic!)            │ simultaneously from the database   │ stagger the fetches (batches with  │
│                                  │                                  │ sleeps); schedule during low-       │
│                                  │                                  │ traffic hours                       │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 5. Interview Quick-Fire Answers

**Q: What is cache warming and why does it matter?**
A: Cache warming is the process of proactively populating a cache BEFORE it receives production traffic, or managing the gradual population of a cache during its first exposure to traffic after a cold start. It matters because a cold cache sends ALL requests to the database as cache misses — potentially overwhelming the database 10-30x its normal load if the cache normally absorbs most traffic. The resulting database overload can cascade into a full outage.

**Q: How would you handle a cache cold start for a high-traffic system?**
A: Multiple complementary approaches: (1) Proactive pre-warming — before shifting traffic, run a job that fetches the top N most-accessed items (based on recent access logs) and populates the cache. (2) Gradual traffic shifting — route only 5% of traffic to the new (cold) deployment initially, letting it warm up while 95% continues hitting the warm old deployment, then gradually increase the percentage. (3) Database protection — implement a mutex lock on cache misses (only one in-flight DB fetch per cache key at a time) and circuit breakers to prevent database overload during the warming period.

---
---

# TOPIC 7: CPU & Browser Cache

---

## 1. CPU Cache — The Fastest Cache in the System

### Core Intuition

Everything discussed so far — Redis, distributed caches — operates at MILLISECOND timescales (or sub-millisecond for local Redis). But modern CPUs operate at NANOSECOND timescales, and MAIN MEMORY (RAM), while much faster than disk, is STILL too slow for the CPU to access without waiting. This is why CPUs have their own MULTI-LEVEL cache hierarchy built directly into the chip.

```
LATENCY COMPARISON (FULL PICTURE):

Level         │ Location            │ Size             │ Latency
──────────────┼─────────────────────┼──────────────────┼─────────
L1 Cache      │ On CPU core          │ 32-64 KB          │ ~1 ns
L2 Cache      │ On CPU (near core)   │ 256 KB - 1 MB     │ ~4 ns
L3 Cache      │ On CPU (shared)      │ 8-64 MB           │ ~10 ns
Main Memory    │ DRAM (on motherboard)│ 16-256 GB         │ ~100 ns
Redis (local)  │ Separate process,RAM │ GBs-TBs           │ ~500,000 ns (0.5ms)
Database (SSD) │ Separate server,disk │ TBs               │ ~5,000,000 ns (5ms)

L1 vs RAM:    100x faster
L1 vs Redis:  500,000x faster
L1 vs DB:     5,000,000x faster

A CPU stall waiting for RAM is ~100 cycles — an ETERNITY at
modern clock speeds. This is why CPU cache design directly
determines the performance ceiling of SOFTWARE running on that
hardware.
```

### How CPU Caches Work

```
THE CPU'S PERSPECTIVE (when it needs data at memory address 0x4F8A20):

1. Check L1 cache (32KB): Is address 0x4F8A20's data here?
   HIT: return in ~1 cycle. Done.
   MISS: check L2...

2. Check L2 cache (256KB): Is 0x4F8A20's data here?
   HIT: load to L1, return in ~4 cycles. Done.
   MISS: check L3...

3. Check L3 cache (32MB): Is 0x4F8A20's data here?
   HIT: load to L2 and L1, return in ~10 cycles. Done.
   MISS: go to main memory...

4. Main memory: DRAM lookup, load cache LINE (typically 64 bytes,
   not just the 4-8 bytes requested) into L3→L2→L1, return in
   ~100 cycles. CPU STALLS for those 100 cycles.

KEY CONCEPT: CACHE LINE
The CPU doesn't cache individual bytes — it caches CACHE LINES
(typically 64 bytes). When ANY byte at an address is needed, the
CPU loads the ENTIRE 64-byte block containing that address into
its cache.

WHY THIS MATTERS FOR SOFTWARE PERFORMANCE:
If your data is SPATIALLY CLOSE IN MEMORY (array elements,
struct fields — stored contiguously), accessing element [0] loads
elements [0] through [7] (for 8-byte integers) into cache
SIMULTANEOUSLY. Accessing [1], [2], ... [7] are then all L1
HITS — extremely fast.

If your data is SCATTERED IN MEMORY (a linked list where each
node can be anywhere in RAM), accessing node[0] loads its 64
bytes into cache. But node[1] might be 50MB away in RAM — that's
a CACHE MISS, requiring another ~100ns main memory access.
Traversing a 1000-element linked list = 1000 potential cache
misses. Traversing a 1000-element array = maybe 15-20 cache
line loads for the whole thing (10,000x better cache utilization).

THIS IS WHY: Arrays outperform linked lists for sequential
access patterns in modern systems, DESPITE both being O(n) —
the CONSTANT FACTOR from cache behavior dominates at scale.
And why column-oriented storage in data warehouses (Databases
notes, Data Warehousing topic) achieves better performance — the
same column's values are stored CONTIGUOUSLY, making CPU cache
lines extremely efficient when aggregating across millions of rows.
```

### CPU Cache Eviction

```
CPU caches use eviction policies similar to those discussed in
Topic 2 — primarily PSEUDO-LRU (an approximation of LRU that's
cheaper to implement in hardware):

A TRUE LRU implementation in hardware would require tracking the
exact access time/order for every cache line in the cache —
with 32KB L1 caches and 64-byte lines, that's 512 entries to
track, which sounds manageable, but at CPU speeds (~1 billion
accesses per second), the bookkeeping overhead of maintaining
an exact LRU ORDER would consume significant die area and power.

PSEUDO-LRU: A tree-based approximation where each group of N
cache lines (a "set") has a few bits of state tracking
"approximately which is oldest" — accurate enough to make good
eviction decisions in practice, with negligible hardware overhead.

WRITE POLICIES in CPU caches:
- Write-through: every write immediately propagates to the
  next cache level (maintains consistency, uses more bandwidth)
- Write-back: writes only propagate when the cache line is
  evicted (better performance; requires "dirty bit" tracking)
  → Same fundamental write-through vs write-back tradeoff as
  cache strategies in Topic 1!
```

---

## 2. Browser Cache — Caching at the Client

### Core Intuition

Every time you visit a website, your browser downloads resources: HTML, CSS, JavaScript files, images, fonts. Without caching, EVERY PAGE LOAD re-downloads ALL of these — even if they haven't changed since your last visit 5 seconds ago.

Browser caching stores these resources LOCALLY on the user's device, serving them from the LOCAL DISK (or memory) on repeat visits — eliminating the network round trip entirely for unchanged resources.

```
WITHOUT BROWSER CACHE:
User opens gmail.com:
  → Downloads: HTML (30KB), main CSS (120KB), JS bundle (800KB),
    logo (15KB), 12 UI icons (48KB total) = ~1MB per page load
  
  Every navigation within Gmail:
  → Re-downloads everything → ~1MB per click!
  Slow, wastes bandwidth, poor UX

WITH BROWSER CACHE (first visit):
  → Same downloads: ~1MB (must fetch everything first time)

  Subsequent navigations / revisits:
  → HTML: fetched fresh (frequently changes) → ~30KB
  → CSS, JS, logo, icons: SERVED FROM BROWSER CACHE → 0 bytes
    downloaded!
  Page loads in ~100ms instead of ~1-3 seconds
```

### HTTP Headers That Control Browser Caching

```
These connect DIRECTLY to the HTTP headers section of the
Networking Fundamentals (HTTP/HTTPS) topic — browser caching
IS the HTTP caching model in action.

RESPONSE HEADER 1: Cache-Control (the primary control mechanism)

Cache-Control: max-age=31536000, immutable
                  ↑                    ↑
                  "cache for 1 year"   "content NEVER changes —
                                        don't even re-validate"
USE FOR: Content-hashed CSS/JS bundles (main.a3f8b2c1.js),
         images, fonts — content with VERSIONED URLs that change
         when content changes.

Cache-Control: no-cache
"Cache this response, but RE-VALIDATE with the server on EVERY
 subsequent request before serving it" (misleading name — it
 does NOT mean "don't cache at all"!) → server returns 304
 Not Modified if unchanged (browser uses its cached copy without
 re-downloading the body) → good for HTML pages.

Cache-Control: no-store
"Don't cache this AT ALL — don't even write it to disk."
USE FOR: Sensitive data (banking session pages, personal data,
         OTP pages) — should never be cached anywhere.

Cache-Control: private
"Only the USER'S browser can cache this — CDN/proxy caches
 MUST NOT cache it."
USE FOR: User-specific pages (your profile, your cart).
         CDNs that serve to ALL users would be wrong to cache this.

Cache-Control: public
"Both browser AND intermediate caches (CDN, proxies) may cache."
USE FOR: Public content (product pages, blog posts, public APIs).

RESPONSE HEADER 2: ETag (entity tag — a "version fingerprint")

ETag: "33a64df551425fcc55e"
          ↑ hash/fingerprint of this specific version of the content

Next visit, browser sends:
If-None-Match: "33a64df551425fcc55e"
→ Server checks: "Is current content still hash 33a64df...?"
→ YES → 304 Not Modified (no body sent — saves bandwidth)
→ NO  → 200 OK with NEW content (and new ETag)

RESPONSE HEADER 3: Last-Modified

Last-Modified: Thu, 12 Jun 2026 10:30:00 GMT
Next visit, browser sends:
If-Modified-Since: Thu, 12 Jun 2026 10:30:00 GMT
→ Server checks: "Was this modified after that time?"
→ NO  → 304 Not Modified
→ YES → 200 OK with fresh content + new Last-Modified

ETag is PREFERRED over Last-Modified for accuracy: Last-Modified
has 1-second granularity (two changes within the same second
look identical), while ETag fingerprints ACTUAL CONTENT changes
regardless of timing.
```

### The Versioned URL Pattern (Cache-Busting)

```
PROBLEM WITH LONG TTLs: If you set CSS to cache for 1 year:
  Cache-Control: max-age=31536000
  
  Browser caches styles.css for 1 year. You deploy new CSS.
  Browser STILL serves the year-old cached version for the
  next 365 days — users see broken/old styling!

SOLUTION: CONTENT-HASHED FILENAMES (build-time cache busting)

Build tool (Webpack, Vite, etc.) generates:
styles.a3f8b2c1.css  ← the hash (a3f8b2c1) is derived from
                         the FILE CONTENT. Same content = same
                         hash. Different content = different hash.

Your HTML references:
<link rel="stylesheet" href="/styles.a3f8b2c1.css">

Cache-Control: max-age=31536000, immutable
                                   ↑ NEVER re-validate — the hash
                                     guarantees this URL's content
                                     will NEVER change (it's in the name!)

When you update CSS → build tool generates styles.b7d1e4f9.css
HTML references the NEW filename → browser treats it as a
BRAND NEW URL → cache miss → downloads new file

OLD URL (styles.a3f8b2c1.css) stays cached in browser (for a
year or until evicted) but is NO LONGER REFERENCED by any HTML
— harmlessly sitting there until evicted.

THIS IS THE "immutable, content-hashed filename" pattern
discussed in BOTH the CDN topic (Networking Fundamentals) and
the Cache Invalidation topic above — the SAME approach works
at BOTH the CDN level and the browser cache level!
```

### Service Workers — Browser-Side Programmatic Caching

```
Service Workers are JavaScript that run in the BACKGROUND of
the browser (separate from the page itself), intercepting
EVERY network request the page makes and deciding:

"Should I serve this from my own cache? From the network?
  Show a custom offline page? Serve stale + update in background?"

Service workers implement THE SAME PATTERNS as server-side caches:
- Cache-First (like cache-aside but check own cache first)
- Network-First (try network, fall back to cache if offline)
- Stale-While-Revalidate (serve stale immediately, refresh async)
- Cache-Only (fully offline, only serve from cache)

USE CASES:
- Progressive Web Apps (PWAs) that work OFFLINE
- Background sync (queue offline actions, replay when reconnected)
- Push notifications (browser notifications via background script)

KEY DIFFERENCE FROM HTTP CACHE: HTTP cache headers are CONTROLLED
BY THE SERVER. Service Worker caching is controlled by CLIENT-
SIDE JavaScript — giving the developer FULL PROGRAMMATIC CONTROL
over exactly what gets cached, for how long, and with what
strategy — but also requiring careful management (clearing stale
service worker caches on deployments is its own problem).
```

---

## 3. Comparison Table — All Caching Levels

```
┌──────────────────┬───────────┬──────────┬──────────────┬──────────────────────────┐
│ Level              │ Location   │ Latency   │ Controlled By│ Best For                  │
├──────────────────┼───────────┼──────────┼──────────────┼──────────────────────────┤
│ CPU L1/L2 Cache    │ CPU chip   │ 1-4 ns    │ Hardware      │ Instruction/data reuse    │
│                   │           │           │              │ within microseconds         │
│ CPU L3 Cache       │ CPU chip   │ ~10 ns    │ Hardware      │ Cross-core data sharing    │
│ In-process cache   │ App RAM    │ ~100 ns   │ Application   │ Per-server hot data         │
│ (local hashmap)    │           │           │ code          │ (not shared across servers)│
│ Redis/Memcached    │ Separate   │ 0.5-2 ms  │ Application   │ Shared hot data across all │
│ (distributed)      │ server, RAM│           │ code +        │ app servers, rate limiting,│
│                   │           │           │ configuration │ sessions, distributed locks│
│ CDN Edge Cache     │ Global PoP │ 5-50ms    │ HTTP headers  │ Static assets, public API  │
│                   │ disk/RAM   │           │ + CDN config  │ responses, global delivery │
│ Browser Cache      │ User device│ 0 ms      │ HTTP headers  │ CSS/JS/images, reducing    │
│                   │ disk/RAM   │           │ + Service     │ repeat network requests    │
│                   │           │           │ Worker        │ for the same user           │
│ Database Buffer    │ DB server  │ ~100 ns   │ Database      │ Frequently accessed data    │
│ Pool              │ RAM        │ (RAM hit) │ engine        │ pages within the database  │
│                   │           │ ~10 ms    │               │                            │
│                   │           │ (disk miss)│              │                            │
└──────────────────┴───────────┴──────────┴──────────────┴──────────────────────────┘
```

---

## 4. Real-World Usage

**Google Chrome (browser cache):** Implements a disk-backed HTTP cache with its own eviction policy — a hybrid of LRU and size-aware eviction (larger files are less likely to be kept than many small files, since a single large file could evict many small frequently-used ones). Chrome also implements "preloading" (fetching resources BEFORE the user navigates, based on link prefetch hints) — essentially PROACTIVE CACHE WARMING for the browser.

**V8 JavaScript Engine (Chrome/Node.js):** The JavaScript JIT (Just-In-Time) compiler benefits enormously from CPU cache behavior. Hot JavaScript functions (called frequently) will have their compiled machine code in CPU L1/L2 cache — subsequent calls execute without any cache miss penalty. V8's object model is deliberately designed for "hidden classes" (objects of the same "shape" share a class descriptor) partly to improve CPU cache line efficiency for property access.

**Next.js / Vite / Webpack (build tools):** All implement content-hash-based filename generation as their DEFAULT mode for production builds — setting `Cache-Control: max-age=31536000, immutable` for hashed assets and short-lived or no-cache for the HTML entry point.

---

## 5. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Users see stale CSS/JS after a   │ Long-lived browser cache holds   │ Content-hashed filenames with      │
│ deployment                       │ old file; no cache-bust          │ max-age=31536000+immutable; short │
│                                  │ mechanism in place               │ TTL or no-cache for HTML entry    │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ User's private data cached by    │ Response lacks Cache-Control:    │ ALWAYS add Cache-Control: private,│
│ a shared CDN or proxy            │ private, served for all users    │ no-store to authenticated/personal│
│                                  │ who request the same URL          │ responses                          │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ CPU "cache thrashing" —          │ Working set of data EXCEEDS       │ Reduce working set size;           │
│ application 10x slower than      │ available CPU cache; every        │ optimize data structure layout     │
│ expected                        │ access is a cache miss (main       │ for spatial locality (arrays vs    │
│                                  │ memory access every time)          │ linked lists; struct of arrays     │
│                                  │                                  │ vs array of structs for hot fields)│
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Service worker serves stale       │ Old service worker still running  │ Implement update-on-reload logic   │
│ offline content after a new       │ after deployment; browser keeps   │ in service worker; use skipWaiting │
│ deployment                       │ old version until page closed     │ + clients.claim(); versioned cache │
│                                  │                                  │ names to force cache invalidation  │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 6. Interview Quick-Fire Answers

**Q: What are CPU cache levels and why do they exist?**
A: CPUs have multi-level caches (L1: ~1ns, 32-64KB; L2: ~4ns, 256KB-1MB; L3: ~10ns, 8-64MB) because main memory (~100ns) is too slow for the CPU to wait for on every access. The hierarchy exists because faster cache (SRAM) is far more expensive per byte than DRAM — so only small amounts of ultra-fast cache can fit economically on the chip, with progressively larger (but slower and cheaper) levels further out.

**Q: What HTTP headers control browser caching, and what does each do?**
A: `Cache-Control: max-age=N` tells the browser to serve the cached version for N seconds without revalidating. `immutable` signals the content will never change (used with content-hashed URLs). `no-cache` means always revalidate before serving (still caches, but checks with the server every time via 304 Not Modified). `no-store` means don't cache at all (for sensitive data). `private` means only the user's browser may cache (not CDNs/proxies). `ETag` provides a content fingerprint — the browser sends it back as `If-None-Match` and receives 304 if unchanged, saving bandwidth.

**Q: How do you ensure users always get the latest JavaScript/CSS after a deployment, while also maximizing caching efficiency?**
A: Content-hash-based filenames — the build tool generates `main.a3f8b2c1.js` (where the hash comes from the file's content). Set `Cache-Control: max-age=31536000, immutable` on these hashed files — they can be cached for a year, forever, because the URL itself changes when the content changes. The HTML entry point (which references these hashed filenames) should have a SHORT TTL or `no-cache` — so the browser always re-fetches the HTML (small, fast) and from it learns the current hashed filenames. This gives both maximum caching efficiency AND instant deployment visibility.

**Q: What is spatial locality in the context of CPU caches, and why does it matter for software design?**
A: Spatial locality means that when a CPU accesses a memory address, it's likely to access NEARBY addresses soon after. CPUs exploit this by loading 64-byte CACHE LINES — when you access one byte, the surrounding 63 bytes come into cache "for free." Data structures that store related data CONTIGUOUSLY in memory (arrays, packed structs) benefit hugely from this — traversing an array hits very few cache misses. Pointer-chasing data structures (linked lists, trees with scattered allocations) suffer constant cache misses since each "next" pointer might point anywhere in RAM. This is why arrays beat linked lists for sequential access despite identical big-O complexity, and why column-oriented databases outperform row-oriented databases for analytical queries.

---
---

# APPENDIX: Cross-Topic Quick Reference

## Master Comparison — All Caching Concepts at a Glance

```
┌──────────────────────────┬───────────────────────────────────────────────────────────┐
│ Topic                     │ Core Question It Answers                                    │
├──────────────────────────┼───────────────────────────────────────────────────────────┤
│ Cache Strategies           │ "HOW does the application interact with the cache on reads │
│                           │ and writes? Who is responsible for population/invalidation?"│
│ Cache Eviction Policies    │ "When the cache is FULL, WHICH item is removed to make room│
│                           │ for new data — LRU, LFU, FIFO, TTL?"                       │
│ Redis                      │ "What IS the cache — internals, data structures, persistence│
│                           │ persistence, replication, and advanced patterns?"           │
│ Cache Invalidation         │ "How do I ensure the cache STOPS serving stale data after  │
│                           │ the source changes — TTL, active invalidation, CDC?"        │
│ Distributed Cache          │ "How do I SCALE the cache itself — sharding, replication,  │
│                           │ multi-layer, distributed locking?"                          │
│ Cache Warming              │ "How do I handle a COLD CACHE at startup, failover, or     │
│                           │ deployment — pre-warm, gradual shift, lazy + protection?"   │
│ CPU & Browser Cache        │ "How do caching principles apply at the HARDWARE LEVEL      │
│                           │ (CPU cache lines, spatial locality) and at the CLIENT LEVEL │
│                           │ (HTTP caching headers, content-hash filenames)?"            │
└──────────────────────────┴───────────────────────────────────────────────────────────┘
```

## A Complete Caching Architecture — All Topics in One Flow

```
USER REQUEST → web.company.com/product/789

1. BROWSER CACHE (Topic 7):
   HTML: Cache-Control: no-cache → browser revalidates (fast 304)
   JS/CSS: Cache-Control: max-age=31536000, immutable → served
           from browser's local disk, ZERO network bytes!

2. CDN EDGE (Networking Fundamentals — CDN topic):
   Product page HTML (if public): served from edge PoP, ~5ms
   On CDN MISS: request hits origin infrastructure below

3. LOAD BALANCER → APP SERVER (Scalability — LB topic)

4. IN-PROCESS / L1 CACHE on app server (Topic 5: Distributed Cache):
   Very hot config/feature flags: served from app's own memory
   in ~100ns. No network round trip to Redis.

5. REDIS DISTRIBUTED CACHE (Topic 3):
   Cache-Aside strategy (Topic 1): GET product:789
   CACHE HIT (e.g., 95% of requests): return in ~0.5ms
   CACHE MISS: proceed to step 6, then populate cache
   LRU eviction policy (Topic 2): older entries evicted when
   Redis hits maxmemory limit

6. DATABASE (Databases notes):
   On cache miss: SELECT * FROM products WHERE id=789
   Result cached in Redis with TTL=300 seconds (Topic 1+4)

7. CACHE INVALIDATION (Topic 4):
   When product 789's price changes in DB:
   → App invalidates cache: DEL product:789 (or CDC pipeline does it)
   → Next request is a cache miss → fetches fresh data → re-caches

8. CACHE WARMING (Topic 6):
   Before a new deployment or after a Redis failover:
   → Pre-warming job populates top 10,000 products
   → Gradual traffic shift (5% → 100%) to avoid cold-start spike
```

## The One Caching Decision Framework

```
DESIGNING A CACHING LAYER — ask these questions in order:

1. WHAT DATA TO CACHE?
   → High read:write ratio (read 100x more than written)
   → Expensive to compute/fetch (DB queries, external API calls)
   → Tolerable staleness (seconds to hours, depending on data)
   → NEVER cache: user-specific sensitive data in shared caches;
     data requiring always-fresh reads; single-access data

2. WHERE TO CACHE?
   → Browser: for assets that change between DEPLOYMENTS, not
     between REQUESTS (CSS, JS, images)
   → CDN: for PUBLIC content served globally
   → App-local (in-process): for per-server hot config/reference
     data with acceptable cross-server incoherence
   → Distributed Redis: for data that MUST be consistent across
     ALL app servers (sessions, counters, locks)
   → Database buffer pool: handled automatically by DB engine

3. WHAT STRATEGY?
   → Cache-Aside: the default for most cases
   → Write-Through: when writes are immediately re-read
   → Write-Behind: for high-write non-critical data (counters)
   → Write-Around: for write-heavy rarely-read data

4. WHAT EVICTION POLICY?
   → LRU: the default for most workloads
   → LFU: when batch/scan workloads risk polluting the cache
   → TTL: ALWAYS set one as the safety net for correctness

5. HOW TO INVALIDATE?
   → TTL expiry: always, as the baseline
   → Active invalidation (DEL on write): for data freshness
   → CDC-based: for distributed systems with many write paths
   → DB-first then invalidate: always this order

6. WHAT HAPPENS ON COLD START?
   → Plan pre-warming + gradual traffic shift for every deployment
   → Protect the database with miss-rate limiting + circuit breakers
```

## Final Study Tips

```
1. DRAW the cache-aside read/write flow (Topic 1), the LRU
   doubly-linked-list + hashmap diagram (Topic 2), and the
   multi-layer cache architecture (Topic 5) from memory. These
   three diagrams cover most "how does caching work" interview
   questions.

2. For EVERY caching design decision, state TWO things:
   a) The TRADEOFF you're making (latency vs consistency,
      hit rate vs memory, speed vs durability)
   b) The FAILURE MODE you're accepting and how you mitigate it

3. CONNECT caching to what you know:
   - Rate limiting (Scalability notes) IS implemented as Redis
     cache operations (INCR, ZADD, ZCARD)
   - CDN caching (Networking Fundamentals) IS the same HTTP
     caching model as browser caching, at an edge server
   - Write-ahead logs (Databases notes — ACID/Replication) are
     Redis's own AOF persistence mechanism
   - Cache stampedes (Topic 4) are the cache-layer version of
     thundering herd (Scalability — Auto-scaling topic)
   - Consistent hashing (Scalability notes) IS how distributed
     caches shard their keyspace across nodes

4. For BFSI/fintech interviews (relevant to your prep):
   - Bank balances, account data: very SHORT TTL or no caching
     (correctness critical; use write-through for anything cached)
   - Reference data (interest rate tables, currency rates):
     longer TTL (minutes to hours), active invalidation when
     rates change — a controlled, admin-driven operation
   - Session tokens: Redis key-value with TTL = session duration,
     secure key hashing (never cache raw auth tokens as keys in
     monitoring/logs)
   - Transaction idempotency keys (Stripe pattern): Redis SET NX
     — store the idempotency key with TTL = 24 hours, return the
     cached result if the same key is submitted again (prevents
     double-charging on network retry)
```
