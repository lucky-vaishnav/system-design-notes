# 5.4 Cache Scaling

We already established the role of Redis in this system:

> **Redis is a performance/read layer; PostgreSQL remains the source of truth for reservation correctness.**

Now the question is: **what happens when Redis itself becomes a bottleneck or fails?**

---

## 5.4.1 Why Redis Can Become a Bottleneck

Suppose our system receives:

* ~1M availability searches/minute
* ≈ **16,700 searches/sec**

If most searches hit Redis, that's a significant Redis workload.

Redis can become constrained by:

* CPU
* memory
* network bandwidth
* number of operations/sec
* large values
* connection count
* hot keys

So our architecture can evolve from:

```text
Node.js
   |
 Redis
   |
PostgreSQL
```

to:

```text
             Load Balancer
                   |
             Node.js instances
                   |
              Redis Cluster
                   |
              PostgreSQL
```

But there's an important problem we need to understand first.

---

# 5.4.2 Hot Keys

A **hot key** is one Redis key receiving a disproportionately large amount of traffic.

Imagine one parking lot is extremely popular.

Our key might be:

```text
availability:parkingLot123:today
```

Suppose thousands of users search that parking lot simultaneously.

Instead of traffic being distributed evenly:

```text
Key A → 1,000 requests
Key B → 1,000 requests
Key C → 1,000 requests
```

we might have:

```text
Key A → 50,000 requests
Key B → 1,000
Key C → 1,000
```

Now one Redis shard can become the bottleneck.

This is similar to the **hot-row** problem we saw in PostgreSQL.

> **Total traffic isn't enough. You must also understand traffic distribution.**

---

# 5.4.3 Why Redis Cluster Doesn't Automatically Solve Hot Keys

Suppose we have:

```text
Redis Cluster

Shard 1
Shard 2
Shard 3
Shard 4
```

Redis distributes keys across shards.

But a particular key maps to one shard:

```text
availability:parkingLot123:today
                ↓
             Shard 2
```

All requests for that key still go to Shard 2.

So:

```text
4 Redis shards
```

doesn't necessarily mean that a single hot key gets 4× the capacity.

This is an important interview point:

> **Sharding distributes keys, not necessarily requests for the same key.**

---

# 5.4.4 What Can We Do About Hot Keys?

There are several approaches.

### 1. Reduce unnecessary requests

For example:

* client-side caching where appropriate
* sensible polling intervals
* avoid repeatedly requesting unchanged data
* cache at API/gateway/CDN layers when applicable

If users are polling availability every few seconds, reducing unnecessary polling can significantly reduce load.

---

### 2. Local application cache

For data that is:

* small
* non-critical
* relatively static

we could keep a short-lived local cache:

```text
Node 1 → local cache
Node 2 → local cache
Node 3 → local cache
```

But this creates separate copies.

So it is **not appropriate for authoritative availability state**.

For example, static parking-lot metadata might be reasonable.

Current reservation availability is different.

---

### 3. Replicate extremely hot read data

For certain read-heavy workloads, a hot value can potentially be replicated across multiple cache nodes/keys so traffic is distributed.

Conceptually:

```text
                 Hot availability
                       |
          +------------+------------+
          |            |            |
       Cache A      Cache B      Cache C
```

But this increases:

* memory usage
* invalidation complexity
* update complexity

So we wouldn't automatically do this.

---

# 5.4.5 Redis Scaling: Vertical vs Horizontal

Just like application/database scaling:

### Vertical scaling

Give Redis more:

* CPU
* RAM
* network capacity

Simple, but eventually limited.

### Horizontal scaling

Use multiple Redis nodes/shards:

```text
             Redis Cluster
          /       |       \
       Node 1   Node 2   Node 3
```

Data is distributed across nodes.

This provides more aggregate capacity.

But now we have distributed-cache concerns:

* key distribution
* resharding
* node failure
* replication
* failover
* hot keys

---

# 5.4.6 Redis Replication and High Availability

We don't want a single Redis node:

```text
Node.js
   |
Redis
```

because if Redis goes down, the cache disappears.

A common HA architecture is:

```text
             Redis Primary
              /        \
             /          \
       Replica 1      Replica 2
```

If the primary fails, a replica can be promoted.

The exact mechanism depends on the Redis deployment architecture.

The important system-design concept is:

> **Replication provides redundancy; clustering provides distributed capacity.**

They're related but not the same thing.

---

# 5.4.7 What If Redis Completely Fails?

This is a very important question for our parking system.

Suppose:

```text
Node.js → Redis ❌
```

Do we stop the entire system?

Not necessarily.

Remember:

```text
PostgreSQL = source of truth
Redis = performance layer
```

So we can degrade to:

```text
Node.js
   |
Redis ❌
   |
PostgreSQL
```

The system can still function, but PostgreSQL suddenly receives much more read traffic.

That creates a **secondary failure risk**.

---

# 5.4.8 Cache Failure Can Cause Database Overload

Suppose normally:

```text
1,000,000 searches/min

Redis handles 950,000
PostgreSQL handles 50,000
```

Redis goes down.

Now potentially:

```text
PostgreSQL
↑
1,000,000 searches/min
```

PostgreSQL may become overloaded.

Then:

```text
Redis failure
    ↓
cache misses
    ↓
DB traffic increases
    ↓
DB overload
    ↓
DB latency increases
    ↓
API latency increases
    ↓
timeouts
    ↓
retries
    ↓
even more DB traffic
```

This is a **cascading failure**.

This is one of the most important production lessons from cache scaling:

> **A cache isn't just an optimization; at scale, downstream systems must be protected when the cache disappears.**

---

# 5.4.9 Protecting PostgreSQL During Cache Failure

Several techniques can help.

### Rate limiting

Limit how much traffic reaches the DB.

### Concurrency limiting

Instead of allowing thousands of simultaneous DB requests:

```text
1000 requests
     ↓
only 100 DB operations concurrently
```

The rest wait or are rejected.

#### rate limiting and concurrency limiting solve different problems
Yes. And this is an important distinction because **rate limiting and concurrency limiting solve different problems**.

#### 1. Rate limit

Controls **how many requests can start over a period of time**.

Example:

```text
Reserve API
→ 100 requests/sec per user
→ 1,000 requests/sec globally
```

Common algorithms:

* Token Bucket
* Leaky Bucket
* Fixed Window
* Sliding Window

For a distributed Node.js system, a shared store such as Redis is commonly used so all API instances enforce the same limit.

---

#### 2. Concurrency limit

Controls **how many requests can be executing at the same time**.

For example:

```text
Reserve API
        ↓
Maximum 500 concurrent reservation operations
        ↓
If 500 are already running
        ↓
Reject / wait / queue
```

This is different from:

> "Allow 1,000 requests per second."

You could receive 1,000 requests/sec, but if each request takes 2 seconds, you could have roughly 2,000 requests in flight.

---

#### How would we implement it?

For our parking system, I would think in **layers**:

```text
                 Incoming Request
                        │
                        ▼
              ┌─────────────────┐
              │   Rate Limiter  │
              │  e.g. Redis     │
              └────────┬────────┘
                       │
                  allowed?
                       │
                       ▼
              ┌─────────────────┐
              │ Concurrency     │
              │ Limiter         │
              └────────┬────────┘
                       │
                  capacity?
                       │
                       ▼
                Reservation API
                       │
                       ▼
                 PostgreSQL
```

#### Rate limit

For example:

> **100 reservation requests/sec per user and 10,000/sec globally.**

This protects the API from excessive request volume.

#### Concurrency limit

Then:

> **At most 500 reservation operations executing concurrently across the system.**

This protects the expensive downstream operation, particularly PostgreSQL.

---

#### But there's an important distributed-systems issue

If you simply do this inside each Node.js process:

```js
let activeRequests = 0;
```

it's **not a global limit**.

If you have:

```text
10 Node instances
×
100 concurrent requests each
=
1,000 concurrent requests
```

Your supposed "100 concurrent request limit" is actually 1,000.

So for a **global concurrency limit**, you need distributed coordination, such as:

* Redis-based semaphore/counter
* A centralized admission-control service
* A queue/worker model
* Or sometimes the database itself acts as the final concurrency control

---

#### For our parking system, I wouldn't automatically add a global concurrency limiter

This is the senior-level nuance.

We already have:

```text
Rate limiting
      ↓
API protection

DB transactions + locks
      ↓
Correctness

DB connection pool
      ↓
Limits DB concurrency

Timeouts / backpressure
      ↓
Prevents uncontrolled waiting
```

If PostgreSQL can comfortably handle the reservation workload, adding a distributed semaphore just adds another distributed dependency and another potential bottleneck.

I'd introduce a **global concurrency limit when measurements show that reservation requests are overwhelming the downstream capacity**.

And for a very hot parking lot, we might need **resource-specific concurrency/admission control**, rather than one global limit.

#### Interview answer

If asked:

> **"How would you control concurrency?"**

A strong concise answer would be:

> "I'd distinguish rate limiting from concurrency limiting. Rate limiting controls how many requests can enter over time, typically using a distributed token-bucket implementation such as Redis. Concurrency limiting controls how many expensive operations are in flight at once. For a distributed system, a global concurrency limit requires shared coordination such as a Redis semaphore or centralized admission control. However, I wouldn't add it by default; I'd first use database connection limits, transactions, and backpressure, and introduce explicit concurrency control when downstream saturation or a hot resource requires it."

That's the important takeaway.



### Request coalescing

Suppose 500 requests all miss the same key.

Instead of:

```text
500 requests
   ↓
500 DB queries
```

we can make:

```text
500 requests
     ↓
1 DB query
     ↓
populate Redis
     ↓
all 500 receive result
```

This is particularly useful for hot keys.

### Stale data

If business rules allow it, temporarily serve slightly stale cached data rather than immediately hitting PostgreSQL.

For availability display, this may be acceptable.

For actual reservation authorization, it isn't.

---

# 5.4.10 Cache Stampede

We've already discussed this earlier, but now we can connect it to Redis scaling.

Suppose:

```text
availability:lot123
```

expires.

10,000 users request it at almost the same time.

All see:

```text
CACHE MISS
```

Without protection:

```text
10,000 requests
      ↓
10,000 DB queries
```

That's a cache stampede.

A better approach:

```text
10,000 requests
      ↓
one request refreshes cache
      ↓
other requests wait/coalesce
      ↓
Redis populated
      ↓
all receive result
```

In a distributed Node.js environment, a local `Map<key, Promise>` can coalesce requests **inside one process**, but it doesn't coordinate across all instances.

For cross-instance coordination, we may need distributed mechanisms—but those themselves add complexity.

---

# 5.4.11 TTL Strategy

We should also think carefully about TTL.

For example:

```text
availability cache TTL = 10 seconds
```

If every key expires at exactly predictable intervals, many keys can expire together.

That can create:

```text
Many keys expire
      ↓
many cache misses
      ↓
DB spike
```

So TTL **jitter** can help:

```text
TTL = 10 sec + random small variation
```

This spreads refreshes over time.

But remember:

> **TTL is not a correctness mechanism.**

Even a 1-second-old cache entry might be wrong for booking.

Therefore:

```text
Search/display → cache okay
Booking decision → PostgreSQL
```

---

# 5.4.12 Cache Invalidation at Scale

Earlier we discussed:

```text
DB update
   ↓
invalidate Redis
```

At high scale, we might use:

```text
PostgreSQL
    ↓
Outbox
    ↓
Kafka
    ↓
Cache consumers
    ↓
Redis
```

Conceptually:

```text
Reservation committed
        ↓
DB transaction
        ↓
Outbox event
        ↓
Kafka
        ↓
Redis update/invalidation
```

This gives us a durable way to propagate changes.

But remember the trade-off:

**Event-driven cache updates are eventually consistent.**

That's okay because Redis isn't the authority.

---

# 5.4.13 Cache vs Database Responsibilities

This distinction should be very clear in an interview.

| Operation            | Redis             | PostgreSQL                                  |
| -------------------- | ----------------- | ------------------------------------------- |
| Search parking lots  | Primary read path | Fallback                                    |
| Availability display | Primary read path | Fallback/source                             |
| Reservation creation | ❌ Not authority   | ✅ Authority                                 |
| Inventory decrement  | ❌                 | ✅                                           |
| Payment state        | ❌                 | ✅                                           |
| Reservation history  | Could cache       | ✅ Source                                    |
| Analytics            | Possible          | Source/read model depending on architecture |

So if the interviewer asks:

> "What if Redis says a slot is available but PostgreSQL says it isn't?"

Answer:

> **PostgreSQL wins. Redis is eventually consistent and is never used as the final authorization for reservation.**

---

# 5.4.14 A Production Cache Architecture

For our parking system, a reasonable conceptual architecture is:

```text
                     Load Balancer
                           |
                    Node.js instances
                           |
                     +-----+-----+
                     |           |
                     v           v
                  Redis       PostgreSQL
                Cluster        Primary
                  /   \           |
             replicas  ...     Replicas
                               
                           |
                         Outbox
                           |
                         Kafka
                           |
                    Cache consumers
                           |
                         Redis
```

The important flow is:

### Read

```text
Request
  ↓
Redis
  ↓ HIT
Response
```

On miss:

```text
Redis MISS
   ↓
PostgreSQL
   ↓
populate Redis
   ↓
Response
```

### Write

```text
Reservation
   ↓
PostgreSQL transaction
   ↓
Outbox event
   ↓
Kafka
   ↓
Redis update/invalidation
```

And critically:

```text
Booking correctness
       ↓
PostgreSQL
```

---

# 5.4.15 Common Mistakes

### ❌ "We'll just add more Redis nodes."

Not necessarily useful for one hot key.

### ❌ "Redis failure means the application must stop."

Not necessarily. It depends on the cache's role, but we need DB protection.

### ❌ "Cache is always eventually consistent, so it's fine."

Not for every operation. Reservation authorization requires strong correctness.

### ❌ "TTL keeps availability correct."

No. TTL only limits how long stale data remains cached.

### ❌ "Redis cluster means every request is distributed."

A hot key can still concentrate traffic on one shard.

### ❌ "We'll update Redis before PostgreSQL."

Dangerous. DB should establish authoritative state first.

---

# 5.4.16 Interview Framework

If asked:

> **"How would you scale Redis?"**

A strong senior answer is:

> "First I'd determine whether the bottleneck is CPU, memory, network, operations per second, or a hot key. I'd use vertical scaling where appropriate, then Redis clustering for horizontal capacity and replication/failover for availability. For hot keys, I'd reduce unnecessary reads, use request coalescing or appropriate replication strategies, and protect the database from cache-miss storms. Cache updates can be driven asynchronously through events, but PostgreSQL remains the source of truth for correctness-critical operations."

That answer demonstrates that you're thinking beyond:

> "Just add Redis nodes."

---

# 5.4 COMPLETE ✅

### What to remember for ANY system-design problem

The reusable mental model is:

```text
Cache scaling
     |
     +── Capacity
     |     ├── CPU
     |     ├── Memory
     |     └── Network
     |
     +── Distribution
     |     └── Cluster/sharding
     |
     +── Hotspots
     |     └── Hot keys
     |
     +── Availability
     |     └── Replication/failover
     |
     +── Failure handling
     |     ├── Cache miss storm
     |     ├── DB protection
     |     └── Graceful degradation
     |
     +── Consistency
           └── Cache ≠ source of truth
```

**Key principle:**

> **A cache should improve performance without becoming a single point of correctness or a trigger for cascading failure.**

---

### Current progress

* 5.1 Capacity & Bottleneck Identification — ✅
* 5.2 Application/API Scaling — ✅
* 5.3 Database Scaling — ✅
* **5.4 Cache Scaling — ✅**
* **5.5 Queue/Kafka Scaling — NEXT**
* 5.6 Traffic Spikes & Backpressure
* 5.7 High Availability
* 5.8 End-to-End Bottleneck Analysis

