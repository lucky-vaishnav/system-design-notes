### Scaling, Bottlenecks & High Availability 

We’ll keep this **senior-level and interview-oriented**, applying everything we've already designed rather than re-learning basic scaling concepts.

Our established peak load is:

* **1M searches/min**
* **100K reservations/min**
* **20K payments/min**
* PostgreSQL = source of truth
* Redis = availability/read-performance layer
* Kafka = asynchronous event transport
* Node.js services are stateless

### How we'll approach this topic

Before saying *"add more servers"*, we'll identify **where the system actually becomes the bottleneck**.

We'll examine:

1. **Application/API layer**

   * Horizontal scaling
   * Load balancing
   * Stateless Node.js services
   * Autoscaling

2. **PostgreSQL**

   * Connection pool limits
   * CPU/IO bottlenecks
   * Lock contention
   * Hot inventory rows
   * Read replicas
   * Partitioning
   * When sharding becomes relevant

3. **Redis**

   * Redis throughput
   * Hot keys
   * Redis HA
   * Scaling/cache topology

4. **Kafka**

   * Partitioning
   * Consumer scaling
   * Consumer lag
   * Backpressure

5. **Traffic spikes**

   * Queueing
   * Rate limiting
   * Load shedding
   * Backpressure

6. **High Availability**

   * AZ failure
   * DB failure
   * Redis failure
   * Kafka failure
   * Service failure
   * Regional failure

7. **Final bottleneck analysis**

   * What actually limits our system?
   * What should we scale first?
   * What should we *not* scale?
   * How do we preserve booking correctness while scaling?

---

### Let's start with the most important senior-level question

Imagine our system is receiving the established peak traffic:

**1M searches/min + 100K reservations/min + 20K payments/min.**

We already have multiple Node.js instances behind a load balancer.

Now traffic increases **5×**.

Instead of immediately proposing solutions, let's reason through the request path:

**Client → Load Balancer → Node.js → Redis/PostgreSQL → Kafka/Payment Provider**

**Question:**
Which layer do you think is likely to become the **first major bottleneck**, and why?

Think through **searches vs reservations vs payments separately**.

#### My Thoughts That Time-
Certain According to the first major bottleneck will be the searching, the searches. For there are one million searches per minute. So if searches are there into this time, there are two more searches, then they will be put in, but the rest of the group will never be featured. And they will be missed. The searches will search within the first one minute, which will be the first. And I mean to say the how it the searches.

Yes, I understand what you're trying to say. Your **direction is correct**, but there is one important correction.

At **1 million searches/minute**, search is a very high-volume workload, so it can become a major bottleneck. But requests don't automatically get "missed" just because they exceed one minute. The actual problem is **capacity**: if our infrastructure can process fewer requests than the incoming rate, requests will start **queuing, timing out, or being rejected**.

For example:

* Incoming: **1M searches/min ≈ 16,667 searches/sec**
* If our system can handle only **10,000 searches/sec**
* Then roughly **6,667 requests/sec** are arriving faster than we can process them.
* The backlog grows, latency increases, and eventually we may need **load shedding/rate limiting** or requests will time out.

### But there's another important point

For our system, searches should **not normally hit PostgreSQL for every request**.

Our architecture already has:

**Client → Load Balancer → Search Service → Redis → PostgreSQL (cache miss)**

So the first question is actually:

> **Can Redis/Search Service handle ~16.7K requests/sec efficiently?**

If most searches are served from Redis, PostgreSQL doesn't necessarily see 1M requests/minute.

That's why, in a senior interview, I would phrase your answer like this:

> **"Search traffic is likely to be one of the first scaling concerns because we have around 1 million searches per minute, or roughly 16.7K requests per second. Since search is predominantly a read workload, we should keep it horizontally scalable and serve frequently requested availability data from Redis rather than hitting PostgreSQL for every request. We then need to evaluate Redis throughput, cache hit ratio, hot keys, application-server capacity, and database load from cache misses."**

That's a much stronger senior-level answer.

### Now let's go one step deeper

Suppose we scale our Node.js search service from **10 instances → 50 instances**, but suddenly PostgreSQL CPU goes to 95%.

That tells us:

**The Node.js layer wasn't the bottleneck. PostgreSQL became the bottleneck.**

So scaling one layer can simply **move the bottleneck downstream**.

That's the key mindset for this topic:

> **Don't ask "How do I scale the system?" first. Ask "Which component is limiting throughput right now?"**

---

The goal is that after we finish, you should be able to look at almost any system-design problem and systematically answer:

> **Where will it bottleneck? How do I prove it? How do I scale that component? What new bottleneck does scaling introduce? How do I maintain correctness and availability while scaling?**

---

## The mental model I want you to build

For any system, think in this order:

**Traffic → Capacity → Bottleneck → Scale → New Bottleneck → Failure → Recovery**

Not:

> "We have high traffic, so let's add more servers."

That's usually too shallow for a senior interview.

---

## 1. First principle: Every component has a capacity

Imagine our request path:

```text
Client
   ↓
Load Balancer
   ↓
Node.js Application
   ↓
Redis
   ↓
PostgreSQL
   ↓
Kafka / External Services
```

Each component has some maximum sustainable throughput.

For example, purely as an illustration:

```text
Node.js       → 30K req/sec
Redis         → 100K req/sec
PostgreSQL    → 8K writes/sec
Kafka         → 100K events/sec
```

These aren't universal numbers. The important concept is:

> **The system's throughput is constrained by its bottleneck.**

If PostgreSQL can sustainably process only 8K writes/sec, adding 100 Node.js servers doesn't magically give us 100K writes/sec.

It may actually make things worse by sending more concurrent work toward PostgreSQL.

---

# 2. Throughput vs Latency vs Capacity

This distinction is extremely important and applies to almost every design.

### Throughput

How much work can the system process?

Example:

> 10,000 requests/sec

### Latency

How long does one request take?

Example:

> p95 = 200 ms

### Capacity

How much load can a component sustainably handle before its performance degrades?

Example:

> PostgreSQL can sustainably handle 8K writes/sec under our workload.

These are related but **not the same thing**.

For example, a system might process:

```text
10,000 req/sec
```

but each request takes:

```text
2 seconds
```

That isn't necessarily good performance.

And a system might have:

```text
500 ms average latency
```

while p99 is:

```text
10 seconds
```

which means a small but important percentage of users are having a terrible experience.

So when we discuss scaling, we'll look at:

* throughput
* latency
* concurrency
* CPU
* memory
* I/O
* connection utilization
* queue depth
* error rate
* lock contention

---

# 3. Now apply this to our PostgreSQL

Our established reservation load is:

**100K reservations/minute**

That's approximately:

**1,667 reservation requests/sec**

And payment:

**20K payments/minute**

≈ **333 payments/sec**

The important thing is that **100K HTTP requests/sec doesn't necessarily mean 100K DB writes/sec**.

A single reservation request may perform several DB operations.

For example:

```text
BEGIN

Check inventory
       ↓
Lock inventory
       ↓
Update inventory
       ↓
Create reservation
       ↓
Create payment record
       ↓
Create outbox event

COMMIT
```

One API request could therefore generate multiple database operations.

This is where senior-level capacity analysis starts.

---

# 4. The first PostgreSQL bottleneck: Connection Pool

This is one of the most important concepts for Node.js backend engineers.

Suppose we have:

```text
20 Node.js instances
```

and each instance has:

```text
pool.max = 50
```

Potentially:

```text
20 × 50 = 1,000 DB connections
```

Now imagine PostgreSQL can comfortably handle only a few hundred active connections for our workload.

We have a problem.

More application servers can actually **make the database less stable**.

This leads to a fundamental principle:

> **Application-layer scaling and database-layer scaling must be planned together.**

---

# 5. Why can't we just increase the DB connection pool?

Because a connection isn't free.

Every connection consumes resources:

* memory
* CPU
* connection management overhead
* query execution resources

And if thousands of requests are waiting for DB connections, increasing the pool can simply move the queue from:

```text
Application
```

to:

```text
PostgreSQL
```

Instead of solving the bottleneck.

This is why **connection pooling is a capacity-control mechanism**, not simply a performance optimization.

---

# 6. Now the interesting part: reservation concurrency

Suppose 1,000 users simultaneously try to reserve the **same parking lot / inventory resource**.

Our PostgreSQL design says:

```sql
SELECT ...
FOR UPDATE;
```

The first transaction obtains the lock.

The others wait.

Conceptually:

```text
User A ──→ LOCK ──→ inventory row
                    │
User B ────────────┤ waiting
User C ────────────┤ waiting
User D ────────────┤ waiting
User E ────────────┘
```

This is important:

### We can horizontally scale Node.js

```text
10 → 50 → 100 instances
```

But if all those requests ultimately need to modify **the same database row**, we haven't eliminated the contention.

We've simply created more concurrent requests waiting for the same lock.

This is called a **hotspot / hot row**.

---

# 7. This is a very important distinction

There are two different problems:

### Problem A — insufficient compute

```text
Node.js CPU = 95%
```

Solution might be:

```text
Add more application instances
```

### Problem B — serialization bottleneck

```text
1000 requests
       ↓
same inventory row
       ↓
one lock
       ↓
requests wait
```

Adding more Node.js instances doesn't fundamentally solve it.

The database intentionally serializes those operations because **correctness requires it**.

This is one of the biggest lessons from our concurrency discussion:

> **Some bottlenecks cannot simply be horizontally scaled because the underlying business operation must remain serialized.**

---

# 8. Now let's connect this with our parking system

Imagine:

```text
Parking Lot A
Available spaces = 1
```

And:

```text
User A
User B
User C
...
User Z
```

all try to reserve it simultaneously.

We cannot allow:

```text
A → success
B → success
C → success
```

because there is only one space.

So some serialization is unavoidable.

The goal isn't:

> "Remove all locking."

The goal is:

> **Keep the correctness-critical serialization as small and efficient as possible.**

That means:

* short transactions
* proper indexes
* minimal work while holding locks
* no external API calls inside DB transactions
* consistent lock ordering
* appropriate isolation level
* avoid unnecessary rows being locked

We've already established why we don't call the payment provider while holding the DB lock.

That's also a **scaling decision**, not only a transaction decision.

---

# 9. Read scaling vs Write scaling

This distinction will be useful across almost every future system-design problem.

### Reads

Usually easier to scale.

We can use:

```text
PostgreSQL
    ↓
Read Replicas
    ↓
Application
```

or:

```text
Redis
```

or:

```text
Search index
```

depending on the workload.

### Writes

Much harder.

Because writes often involve:

* consistency
* ordering
* locking
* transactions
* unique constraints
* conflicts
* durability

That's why our:

```text
1M searches/min
```

may actually be easier to scale than:

```text
100K reservations/min
```

even though the reservation number is much smaller.

---

# 10. Read replicas — important caveat

Suppose we add:

```text
Primary DB
   ↓
Read Replica 1
Read Replica 2
Read Replica 3
```

Now we can distribute read traffic.

But replicas are generally **asynchronous**.

So:

```text
Primary:
reservation = CONFIRMED
```

while a replica might temporarily have:

```text
reservation = PENDING_PAYMENT
```

Therefore:

> **Don't send correctness-critical reads to a potentially stale replica.**

For our system:

### Can use replica

```text
Reservation history
Analytics
Reporting
Some search/read workloads
```

### Must use authoritative state

```text
Can I reserve this slot?
Did this reservation succeed?
What's the authoritative payment state?
```

This connects directly to the consistency topic we already completed.

---

# 11. This gives us a general scaling hierarchy

When a system becomes slow, don't immediately jump to sharding.

Think progressively:

```text
1. Optimize inefficient work
        ↓
2. Remove unnecessary DB calls
        ↓
3. Add proper indexes
        ↓
4. Cache appropriate reads
        ↓
5. Horizontal application scaling
        ↓
6. Read replicas
        ↓
7. Partitioning
        ↓
8. Specialized storage/read models
        ↓
9. Sharding
```

The exact order can change by system, but the principle is:

> **Use the least complex solution that removes the actual bottleneck.**

Sharding is powerful, but it introduces significant complexity:

* routing
* cross-shard queries
* transactions
* rebalancing
* operational complexity
* hot shards

So we shouldn't introduce it simply because the system is "large."

---

# 12. One more fundamental concept: Queueing

Suppose:

```text
Incoming = 20K req/sec
Capacity = 15K req/sec
```

Then:

```text
Backlog = +5K/sec
```

The system may initially appear fine.

Then latency starts increasing:

```text
100 ms
↓
500 ms
↓
2 sec
↓
5 sec
↓
timeouts
```

This is why **latency degradation is often an early symptom of capacity saturation**.

Eventually:

```text
timeouts
→ retries
→ more requests
→ more load
→ more saturation
→ more timeouts
```

This creates a **retry storm**.

That's an extremely important production failure mode.

---

# 13. The scaling feedback loop

You should remember this pattern:

```text
Traffic increases
      ↓
Component approaches capacity
      ↓
Latency increases
      ↓
Requests timeout
      ↓
Clients retry
      ↓
Traffic increases further
      ↓
Component becomes even more overloaded
```

This is why production systems often need:

* timeouts
* bounded retries
* exponential backoff
* jitter
* circuit breakers
* rate limiting
* backpressure
* load shedding

We'll cover these more deeply in the **Security + Rate Limiting + Reliability** section.

---

# The foundation I want you to take from this

For **any future system-design problem**, when you reach scaling, ask these questions:

### 1. What is the traffic?

```text
RPS
read/write ratio
peak vs average
burstiness
```

### 2. What does one request actually do?

```text
DB queries
cache operations
external calls
events
CPU work
```

### 3. What is the capacity of each layer?

```text
API
Cache
DB
Queue
External dependency
```

### 4. Where is the bottleneck?

```text
CPU?
Memory?
Network?
DB I/O?
Connections?
Locks?
Hot keys?
External API?
```

### 5. Can it scale horizontally?

If yes:

```text
Add instances / replicas / partitions
```

If no:

Ask **why**.

Often the reason is:

```text
shared state
serialization
consistency
ordering
```

### 6. What happens after we scale it?

This is the senior-level question.

> **Scaling one component frequently moves the bottleneck somewhere else.**

---

## Now let's continue interactively

For our parking system, let's focus specifically on this scenario:

> **100K reservation requests/minute, but a large percentage of them are concentrated on a small number of extremely popular parking lots.**

We already know Node.js can horizontally scale.

**What would you do about the database hot-row / hot-inventory problem?**

Don't worry about giving a perfect answer. Think through the trade-off yourself first.

---
Yes — I understand your idea. You're essentially saying:

> If a particular parking lot becomes extremely hot, instead of allowing unlimited concurrent reservation attempts against the same inventory row, we should **control/constrain the concurrency** for that specific resource.

That is the right direction. 👍

But there's an important refinement: **we should not simply limit requests at the global API level**, because that would unnecessarily block reservations for other parking lots.

## The problem

Suppose:

```text
Parking Lot A → 1 slot available
Parking Lot B → 500 slots available
Parking Lot C → 300 slots available
```

And suddenly:

```text
10,000 users → Parking Lot A
500 users    → Parking Lot B
300 users    → Parking Lot C
```

The problem is specifically **Lot A's inventory**, not the whole reservation system.

If all 10,000 requests reach PostgreSQL:

```text
10,000 requests
       ↓
same inventory row
       ↓
lock contention
       ↓
9,999 requests waiting/rejected
```

Adding more Node.js instances doesn't solve the fundamental serialization.

---

# Resource-level concurrency control

One approach is to introduce a **queue / admission control mechanism per hot resource**.

Conceptually:

```text
             Reservation Requests
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
     Lot A Queue           Lot B Queue
     10,000 requests          500
          │
          ↓
   Controlled processing
          │
          ↓
     PostgreSQL
```

Instead of allowing thousands of requests to simultaneously fight for the same DB lock, we control how many are admitted for that particular lot.

For example, conceptually:

```text
Lot A → only N concurrent reservation attempts
```

The remaining requests wait in a queue or receive a controlled response such as:

> "High demand. Please try again."

The exact mechanism depends on the product requirements.

---

# But here's the important trade-off

We **cannot blindly put every reservation into a queue**.

Why?

Because reservation is usually an interactive operation.

Imagine:

```text
User clicks "Reserve"
        ↓
Request enters queue
        ↓
User waits 30 seconds
        ↓
Inventory no longer available
```

That's a poor user experience.

So we'd generally use this selectively for **extreme hot spots**, rather than making the entire reservation system asynchronous.

---

# Another important optimization

Before introducing a queue, we should ask:

> **Do we actually need to lock a single shared row for the entire inventory?**

For example, instead of:

```text
parking_lot
------------
available_slots = 100
```

and every request updating the same row:

```sql
UPDATE parking_lot
SET available_slots = available_slots - 1
WHERE id = ?
AND available_slots > 0;
```

we could model individual inventory units or use an appropriate inventory allocation strategy.

Then requests might contend less depending on the business model.

But there's a trade-off:

* More granular inventory → potentially less contention
* More rows/state → more complexity
* Aggregate counters → simpler but potentially hotter

So **data modeling itself can become a scaling decision**.

---

# The bigger lesson

You just touched an important senior-level concept:

> **Scale based on the distribution of traffic, not just the total traffic.**

This is extremely important.

Our system might have:

```text
100K reservations/min
```

but the distribution could be:

```text
95% → normal lots
5%  → extremely popular lots
```

The 5% could create disproportionate contention.

This is called a **hotspot**.

And hotspots can appear in:

* database rows
* partitions
* Redis keys
* Kafka partitions
* shards
* geographic regions

---

## One correction to your approach

You mentioned something like limiting requests so they don't all come at once.

That's good, but remember:

**Rate limiting ≠ concurrency control ≠ queueing.**

They solve different problems.

| Mechanism            | Purpose                                          |
| -------------------- | ------------------------------------------------ |
| Rate limiting        | Limit how many requests can enter                |
| Concurrency limiting | Limit how many operations execute simultaneously |
| Queue                | Temporarily hold work for later processing       |
| Load shedding        | Reject work when the system is overloaded        |
| Lock                 | Protect shared state/correctness                 |

For our hot parking lot, **resource-level concurrency control or admission control** is closer to what you're describing.

---

### And there's an even deeper question

Suppose we introduce a queue for Lot A.

Now we have:

```text
10,000 users
      ↓
Lot A Queue
      ↓
Reservation workers
      ↓
PostgreSQL
```

**What happens if we have 10 application instances?**

If every instance maintains its own:

```text
Lot A Queue
```

we could still have:

```text
Instance 1 → 100 requests
Instance 2 → 100 requests
Instance 3 → 100 requests
...
```

and we haven't really created **one global concurrency limit for Lot A**.

This leads us to another important scaling concept:

> **Distributed concurrency control.**

That's where things like distributed locks, centralized queues, partitioned work, and resource ownership become relevant.

We'll tackle that next, because it generalizes extremely well to other system-design problems.

---

# Distributed Concurrency Control

Let's continue from exactly where we stopped.

Imagine we have:

```text
             Load Balancer
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
    Node 1     Node 2     Node 3
       │          │          │
       └──────┬───┴──────────┘
              ↓
        PostgreSQL
```

10,000 requests target the same hot parking lot.

If we try to maintain a concurrency limit like:

```text
"Only 10 requests for Lot A at a time"
```

inside each Node.js process:

```text
Node 1 → 10
Node 2 → 10
Node 3 → 10
...
```

we don't actually have a global limit of 10.

We might have:

```text
10 × number_of_instances
```

requests executing.

So we need **shared coordination** if we truly want a system-wide limit.

Possible approaches include:

### A. Distributed lock

A shared coordination mechanism determines which worker can process a resource.

### B. Centralized queue

Requests for the same resource enter a shared queue.

### C. Partitioned ownership

We assign a particular resource/key to a particular worker/partition.

For example:

```text
hash(parkingLotId)
       ↓
Partition 7
       ↓
Worker responsible for Lot A
```

This is particularly interesting because it connects directly to **Kafka partitioning**, which we'll study later.

### D. Database itself

Sometimes the best answer is:

> **Don't add another distributed coordination layer. Let PostgreSQL handle the concurrency because it already provides the required correctness guarantees.**

This is an important senior-level trade-off.

---

# The principle I want you to remember

**Distributed systems have a cost for coordination.**

If PostgreSQL can safely handle:

```text
hot row → 1,000 concurrent attempts
```

then introducing Redis locks + queues + workers may make the architecture unnecessarily complicated.

But if:

```text
hot row
   ↓
massive contention
   ↓
lock waits
   ↓
timeouts
   ↓
retry storm
```

then we have to consider another strategy.

So the question isn't:

> "Which scaling technique is best?"

It's:

> **"What is the actual bottleneck, and what is the simplest mechanism that removes it without compromising correctness?"**

That is the mindset I want you to carry into **every system-design problem**.

---

## Our track from here

We'll now finish **5.1** by understanding:

**hotspots → distributed concurrency → capacity limits → queueing/backpressure**

Then I'll explicitly say:

> **"5.1 complete."**

Only then we'll move to **5.2 Application/API Scaling**.

And at the end of each major subsection, I'll give you a short **"What to remember for any system design"** summary so you don't lose the bigger picture again.

