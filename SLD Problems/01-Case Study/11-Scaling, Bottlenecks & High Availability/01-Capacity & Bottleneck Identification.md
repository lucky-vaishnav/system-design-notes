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


