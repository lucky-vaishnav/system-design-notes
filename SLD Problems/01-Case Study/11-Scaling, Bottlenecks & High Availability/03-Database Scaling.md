# 5.3 Database Scaling

The important mindset is:

> **Application servers are usually easier to scale horizontally because they can be stateless. Databases are harder because data has relationships, transactions, consistency requirements, locks, and writes that must be coordinated.**

For our parking system, this matters especially because **reservation correctness depends on PostgreSQL**.

---

## 5.3.1 Why the Database Becomes a Bottleneck

Imagine our system reaches:

* 1M searches/minute
* 100K reservations/minute
* ~20K payments/minute

We can add more Node.js instances:

```text
              Load Balancer
                    |
        +-----------+-----------+
        |           |           |
     Node 1      Node 2      Node 3
        |           |           |
        +-----------+-----------+
                    |
               PostgreSQL
```

Adding Node instances helps only until PostgreSQL becomes the bottleneck.

For example:

```text
100 Node instances
        |
        v
   PostgreSQL
     100% CPU
     high locks
     high I/O
     connection saturation
```

Adding another 50 Node instances doesn't solve the problem.

It can actually make it **worse**, because more application instances can generate more DB connections and more concurrent queries.

So the first principle is:

> **Never assume application scaling automatically scales the database.**

---

# 5.3.2 First Step: Optimize the Existing Database

Before introducing replicas, partitioning, or sharding, we should make the current database efficient.

Typical sequence:

```text
Bad query
   ↓
Query optimization
   ↓
Proper indexes
   ↓
Reduce unnecessary DB calls
   ↓
Connection-pool tuning
   ↓
Caching
   ↓
Read replicas
   ↓
Partitioning
   ↓
Sharding
```

The exact order depends on the bottleneck, but generally:

> **Don't introduce distributed database complexity before proving you need it.**

---

# 5.3.3 Query Optimization + Indexing

Suppose we have:

```sql
SELECT *
FROM reservation
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20;
```

If `user_id` isn't indexed, PostgreSQL may need to scan a large amount of data.

With an appropriate index:

```sql
CREATE INDEX idx_reservation_user_created
ON reservation(user_id, created_at DESC);
```

the database can find the relevant records much more efficiently.

### Important senior-level point

An index isn't automatically good just because it exists.

Indexes have costs:

* additional storage
* additional write work
* memory usage
* maintenance overhead
* potentially poor usefulness if the query doesn't match the index

So we design indexes based on **actual query patterns**.

---

# 5.3.4 Composite Indexes

For example:

```sql
SELECT *
FROM reservation
WHERE parking_lot_id = ?
  AND status = ?
  AND created_at > ?
ORDER BY created_at DESC;
```

A composite index could be considered:

```text
(parking_lot_id, status, created_at)
```

The important thing isn't memorizing a particular ordering.

The principle is:

> **Indexes should be designed around the filtering, joining, and ordering patterns of important queries.**

And in production, we verify with query plans such as:

```sql
EXPLAIN ANALYZE ...
```

rather than assuming the index is helping.

---

# 5.3.5 Reduce DB Work

Sometimes the biggest database optimization isn't an index.

It's **not making the query in the first place.**

For example, suppose one reservation request does:

```text
Request
  ↓
Query parking lot
  ↓
Query slot
  ↓
Query pricing
  ↓
Query reservation
  ↓
Query configuration
  ↓
INSERT reservation
```

If some information can safely come from:

* Redis
* in-memory immutable configuration
* a read model
* a single optimized query

we can reduce database work.

At high traffic:

```text
1 request → 6 DB queries

100K requests/min
→ 600K DB queries/min
```

Reducing it to:

```text
1 request → 2 DB queries
```

can have a much bigger impact than simply adding more application servers.

---

# 5.3.6 Connection Pooling

This is particularly important for our Node.js architecture.

Suppose:

```text
20 Node instances
×
pool.max = 50
=
1000 potential DB connections
```

If we scale to:

```text
100 Node instances
×
50
=
5000 connections
```

PostgreSQL may not be able to handle that effectively.

So database capacity isn't just:

> "How many queries per second can PostgreSQL handle?"

It also includes:

* connection capacity
* CPU
* memory
* disk I/O
* locks
* transaction duration
* query latency

### Important principle

> **Application horizontal scaling must be coordinated with database connection capacity.**

Connection pooling protects the DB from every request opening a new connection, but the **total pool capacity across all instances** still matters.

---

# 5.3.7 Read Scaling — Read Replicas

Now suppose our database workload looks like:

```text
90% reads
10% writes
```

Scaling reads is often easier than scaling writes.

We can introduce read replicas:

```text
                 PostgreSQL Primary
                 /       |       \
                /        |        \
               v         v         v
           Replica 1  Replica 2  Replica 3
```

Writes:

```text
Application → Primary
```

Read-heavy operations:

```text
Application → Replica
```

For example:

### Good candidates

* reservation history
* reports
* analytics
* non-critical search queries
* user history

### Bad candidates

For our parking system:

```text
"Is this slot definitely available?"
```

immediately before reservation.

Why?

Because replicas are generally **asynchronous**.

---

# 5.3.8 Replica Lag

Suppose:

```text
Primary:
slot A = RESERVED
```

but replication hasn't caught up.

Replica:

```text
slot A = AVAILABLE
```

If our booking service asks the replica:

```text
"Can I reserve slot A?"
```

it might incorrectly see:

```text
AVAILABLE
```

That creates a correctness problem.

Therefore:

> **Read replicas can improve read scalability, but they cannot automatically replace the primary for correctness-critical decisions.**

This is exactly the same principle we established earlier:

**PostgreSQL authoritative state decides booking correctness.**

---

# 5.3.9 Read/Write Splitting

We could have:

```text
                Application
                /         \
               /           \
          Read path       Write path
             |                |
             v                v
         Replica           Primary
```

But we must classify operations carefully.

### Example

```text
GET /reservations/history
        ↓
Replica
```

Potentially fine.

But:

```text
POST /reservations
        ↓
Primary
```

Definitely primary.

And something like:

```text
POST /reservation
        ↓
commit reservation
        ↓
immediately fetch authoritative reservation state
```

may also need primary/read-your-write handling.

This is where distributed systems reasoning matters.

---

# 5.3.10 Write Scaling Is Harder

Suppose reads are easy:

```text
Primary
   |
   +--- Replica
   +--- Replica
   +--- Replica
```

Now imagine we have too many writes.

Can we simply create:

```text
Primary 1
Primary 2
Primary 3
```

and let all of them modify the same inventory?

Not without additional coordination.

Consider:

```text
Slot A
available = 1
```

Two users:

```text
User A → Primary 1
User B → Primary 2
```

Both might believe:

```text
available = 1
```

and attempt to reserve it.

Now we need distributed coordination.

That's why:

> **Write scaling is fundamentally harder when multiple writers need to maintain shared consistency.**

---

# 5.3.11 Hot Rows

This is particularly important for our parking system.

Suppose a popular parking lot has:

```text
100 available slots
```

and thousands of users simultaneously try to reserve them.

They may all contend around the same inventory data.

Even if we have:

```text
20 Node servers
```

and:

```text
10 DB servers
```

the actual inventory operation may still need serialization around the relevant row/resource.

For example:

```sql
SELECT ...
FROM parking_inventory
WHERE parking_lot_id = ?
FOR UPDATE;
```

That lock is intentional because we need correctness.

But it creates a **hotspot**.

---

# 5.3.12 Why Sharding Doesn't Automatically Solve Hot Rows

Suppose we shard:

```text
Shard 1
Shard 2
Shard 3
Shard 4
```

based on:

```text
parking_lot_id
```

If one parking lot is extremely popular:

```text
Parking Lot A
     ↓
Shard 2
```

all requests for that lot still go to:

```text
Shard 2
```

So:

> **Sharding distributes data, but it does not automatically eliminate hotspots.**

This is a very important senior-level system-design point.

The **distribution key matters**.

---

# 5.3.13 Partitioning vs Sharding

These are often confused.

### Partitioning

Usually happens **inside one logical database**.

Example:

```text
reservation
   |
   +-- partition 2026-01
   +-- partition 2026-02
   +-- partition 2026-03
   +-- partition 2026-04
```

For example, partition by:

```text
created_at
```

Useful when:

* table becomes extremely large
* queries naturally filter by partition key
* old data needs archival/deletion
* maintenance becomes expensive

Partitioning can improve:

* query pruning
* maintenance
* storage management
* large-table operations

But it doesn't magically provide unlimited write capacity.

---

# 5.3.14 Sharding

Sharding distributes data across **multiple database instances/clusters**.

Example:

```text
                 Application
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Shard 1    Shard 2    Shard 3
       Users A-H  Users I-P  Users Q-Z
```

Now storage and database workload are distributed.

This can help with:

* huge datasets
* high write throughput
* high storage requirements
* workload isolation

But it introduces significant complexity.

---

# 5.3.15 Sharding Problems

Once data is distributed across databases, things become harder.

### Cross-shard queries

```text
Find all reservations for user X
```

If data isn't colocated properly, we may need multiple shards.

### Cross-shard transactions

Much harder than a normal DB transaction.

### Joins

Cross-database joins are expensive or unavailable in the normal relational sense.

### Rebalancing

Suppose:

```text
Shard 1 = 90% full
Shard 2 = 30%
Shard 3 = 20%
```

We may need to redistribute data.

### Hot shards

Bad shard key:

```text
shard = parking_lot_id
```

if one lot dominates traffic.

So:

> **Choosing a shard key is a major architectural decision.**

---

# 5.3.16 When Would We Actually Shard This Parking System?

Not simply because we have:

```text
100M users
```

That number alone doesn't justify sharding.

We first ask:

```text
Can one PostgreSQL cluster handle the workload?
```

If yes:

**Don't shard yet.**

We could use:

```text
PostgreSQL
+ proper indexes
+ query optimization
+ connection pooling
+ caching
+ read replicas
+ partitioning
```

If eventually the primary cannot handle the write/storage workload even after optimization and vertical scaling, then sharding becomes a candidate.

---

# 5.3.17 A Practical Database Scaling Evolution

A realistic architecture might evolve like this:

### Stage 1

```text
Node.js
   |
PostgreSQL
```

### Stage 2 — optimize

```text
Node.js
   |
PostgreSQL
   +
indexes/query optimization
```

### Stage 3 — cache

```text
Node.js
   |
Redis
   |
PostgreSQL
```

### Stage 4 — read scaling

```text
             +-- Replica 1
             |
Node.js ---- Primary
             |
             +-- Replica 2
```

### Stage 5 — partitioning

```text
PostgreSQL
   |
   +-- Reservation partitions
   +-- Payment partitions
   +-- Historical partitions
```

### Stage 6 — sharding, only if required

```text
              Application
             /     |     \
            /      |      \
       DB Cluster 1  DB Cluster 2  DB Cluster 3
```

Each step increases architectural complexity.

That's why we don't jump directly to sharding.

---

# 5.3.18 One Important Exception: Reservation Inventory

For our system, reservation inventory deserves special treatment.

The inventory path is:

```text
Search
  ↓
Redis
  ↓
User selects slot
  ↓
PostgreSQL
  ↓
Transaction
  ↓
Lock/check inventory
  ↓
Reserve
  ↓
PENDING_PAYMENT
```

This path prioritizes:

**correctness > raw scalability**

We can scale surrounding read traffic aggressively, but we cannot remove the consistency requirement simply to get more throughput.

This is a recurring system-design principle:

> **Not every part of a system needs the same consistency or scaling strategy.**

---

# 5.3.19 What Would I Monitor?

A senior engineer shouldn't just say:

> "We'll add replicas."

We should first identify the actual bottleneck.

### Database metrics

Monitor:

* CPU
* memory
* disk I/O
* storage
* query latency
* slow queries
* transactions/sec
* connections
* connection pool utilization
* lock waits
* deadlocks
* replication lag
* cache hit ratio
* rows scanned vs returned

For example:

```text
CPU:                 45%
Connections:         95%
Lock wait:           HIGH
Query latency:       HIGH
```

The solution might be **connection/pool or locking**, not a bigger database.

Another case:

```text
CPU:                 95%
Connections:         40%
Lock wait:           LOW
Slow queries:        HIGH
```

That points more toward:

* query optimization
* indexing
* reducing DB work
* scaling compute

The metrics tell us where the bottleneck actually is.

---

# 5.3.20 Production Decision Framework

In an interview, if asked:

> "How would you scale the database?"

Don't immediately say:

> "We'll use sharding."

Instead:

### Step 1 — Identify workload

```text
Read-heavy?
Write-heavy?
Mixed?
```

### Step 2 — Identify bottleneck

```text
CPU?
I/O?
Connections?
Locks?
Storage?
Slow queries?
```

### Step 3 — Optimize

```text
Queries
Indexes
DB calls
Transactions
Connection pools
```

### Step 4 — Scale reads

```text
Cache
Read replicas
Read models
```

### Step 5 — Scale large datasets

```text
Partitioning
Archival
```

### Step 6 — Scale writes

If the primary itself becomes the bottleneck:

```text
Partitioning
workload separation
specialized storage
sharding
```

depending on the actual problem.

### Step 7 — Re-evaluate correctness

Especially for:

```text
Inventory
Payments
Reservations
Financial state
```

Don't sacrifice correctness just to distribute load.

---

# 5.3.21 Common Interview Mistakes

### ❌ "We'll add read replicas for everything."

Replicas are generally asynchronous.

---

### ❌ "We'll shard because the system is huge."

Size alone doesn't justify sharding.

---

### ❌ "More Node servers means more DB capacity."

Not necessarily. They can actually increase DB pressure.

---

### ❌ "Redis solves database scaling."

Redis reduces some DB reads. It doesn't solve DB write contention or transactional correctness.

---

### ❌ "Partitioning and sharding are the same."

They're different architectural mechanisms.

---

### ❌ "A hot row can be solved by adding more database servers."

Not necessarily. If the same resource requires serialized updates, the contention remains.

---

# 5.3.22 The Reusable Mental Model

For **any** system-design problem:

```text
                Database
                   |
        +----------+----------+
        |                     |
      READS                  WRITES
        |                     |
     Optimize              Optimize
        |                     |
      Cache              Transactions
        |                     |
   Read replicas         Reduce contention
        |                     |
   Read models          Partition/workload
                              |
                           Sharding
```

But underneath everything:

```text
Correctness
    ↓
Actual bottleneck
    ↓
Least-complex solution
    ↓
Scale
```

---

# 5.3 COMPLETE ✅

The key things I want you to retain from this entire section are:

1. **Database scaling is harder than stateless application scaling.**
2. **Optimize queries and indexes before introducing distributed complexity.**
3. **Connection pools must be considered across all application instances.**
4. **Read replicas help read scaling but introduce replica lag.**
5. **Correctness-critical reads should use authoritative state.**
6. **Write scaling is harder because of consistency and coordination.**
7. **Hot rows/resources can remain bottlenecks even after horizontal scaling.**
8. **Partitioning and sharding solve different problems.**
9. **Sharding distributes data/workload but introduces significant complexity.**
10. **Always identify the actual bottleneck before choosing the scaling technique.**

### Current roadmap

* 5.1 Capacity & Bottleneck Identification — ✅
* 5.2 Application/API Scaling — ✅
* **5.3 Database Scaling — ✅**
* **5.4 Cache Scaling — NEXT**
* 5.5 Queue/Kafka Scaling
* 5.6 Traffic Spikes & Backpressure
* 5.7 High Availability
* 5.8 End-to-End Bottleneck Analysis

Next, **5.4 Cache Scaling** can also be covered as one combined section: **hot keys → Redis capacity → Redis HA → scaling strategies → failure scenarios → trade-offs**.
