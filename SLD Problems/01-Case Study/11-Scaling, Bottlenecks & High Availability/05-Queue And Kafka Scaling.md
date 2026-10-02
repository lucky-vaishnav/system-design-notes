We already covered Kafka conceptually during **Saga + Outbox + Kafka**, so I won't re-teach Kafka from scratch. Here we're focusing specifically on **how the event/queue layer scales under high traffic and what happens when consumers can't keep up.**

# 5.5 Queue / Kafka Scaling

For our parking system, we might have events such as:

```text
ReservationCreated
PaymentSucceeded
PaymentFailed
ReservationExpired
RefundRequested
ReservationCancelled
```

A simplified flow:

```text
                    PostgreSQL
                        |
                      Outbox
                        |
                        v
                      Kafka
                 /       |       \
                /        |        \
               v         v         v
          Payment      Cache     Notification
          Consumer    Consumer    Consumer
```

The important question is:

> **What happens when event production is faster than event consumption?**

That's where Kafka scaling becomes important.

---

# 5.5.1 The Core Kafka Scaling Model

The key concept is:

> **Kafka scales primarily through partitions.**

Suppose we have:

```text
Topic: reservation-events

Partition 0
Partition 1
Partition 2
Partition 3
```

Kafka can distribute events across these partitions.

Then consumers can process those partitions in parallel.

```text
                 reservation-events
                  /   |   |   \
                 v    v   v    v
                P0   P1  P2   P3
                 |    |   |    |
                 v    v   v    v
                C1   C2  C3   C4
```

This gives us parallel processing.

---

# 5.5.2 Partitions Are the Unit of Parallelism

This is a very important interview concept.

Suppose:

```text
Topic = 4 partitions
Consumer group = 8 consumers
```

You don't get 8 consumers processing Kafka partitions simultaneously.

You effectively have:

```text
4 partitions
4 active consumers
4 idle consumers
```

because one partition is assigned to only one consumer within the same consumer group at a time.

Conceptually:

```text
P0 → C1
P1 → C2
P2 → C3
P3 → C4

C5 → idle
C6 → idle
C7 → idle
C8 → idle
```

Therefore:

> **Adding consumers beyond the number of partitions doesn't increase parallelism for that consumer group.**

---

# 5.5.3 More Partitions → More Potential Parallelism

Suppose we start with:

```text
4 partitions
4 consumers
```

and consumers can't keep up.

We could have:

```text
8 partitions
8 consumers
```

Now more events can be processed concurrently.

But this isn't free.

More partitions mean:

* more metadata
* more files/state
* more network activity
* more consumer coordination
* more operational complexity

So we shouldn't blindly create thousands of partitions.

---

# 5.5.4 Consumer Groups

A consumer group gives us independent processing of a topic.

Example:

```text
reservation-events

        |
   +----+----+
   |         |
   v         v
Payment    Notification
Group        Group
```

Each group receives the events independently.

So:

```text
ReservationCreated
       |
       +----> Payment consumer group
       |
       +----> Notification consumer group
       |
       +----> Analytics consumer group
```

This is useful because one consumer workload doesn't have to directly block another.

---

# 5.5.5 Example With Our Parking System

Suppose:

```text
reservation-events
```

has:

```text
8 partitions
```

Payment consumer group:

```text
8 consumers
```

Notification group:

```text
3 consumers
```

Analytics group:

```text
2 consumers
```

Kafka can independently maintain those consumer groups.

Conceptually:

```text
                 Kafka
                   |
       +-----------+-----------+
       |           |           |
       v           v           v
    Payment    Notification  Analytics
     Group        Group        Group
```

This is one reason Kafka is useful for decoupling workloads.

---

# 5.5.6 Ordering

There's an important limitation.

Kafka guarantees ordering **within a partition**, not globally across all partitions.

For example:

```text
Partition 0:

ReservationCreated
PaymentSucceeded
ReservationConfirmed
```

These can maintain their order.

But across:

```text
Partition 0
Partition 1
```

we shouldn't assume a global ordering.

---

# 5.5.7 Why Partition Key Matters

Suppose we have:

```text
ReservationCreated
PaymentSucceeded
ReservationCancelled
```

For reservation `R123`, we may want these events to remain ordered.

We can use:

```text
partition key = reservationId
```

Then all events for:

```text
R123
```

go to the same partition.

Conceptually:

```text
R123 → Partition 2

ReservationCreated
PaymentSucceeded
ReservationConfirmed
ReservationCancelled
```

This helps preserve ordering for that reservation.

But there's a trade-off.

---

# 5.5.8 Bad Partition Keys Can Create Hot Partitions

Suppose we use:

```text
parkingLotId
```

as the partition key.

And one parking lot is extremely popular.

Then:

```text
ParkingLot A
      ↓
Partition 2
      ↓
huge traffic
```

while other partitions are relatively idle.

We have recreated the same hotspot problem we saw with:

* PostgreSQL hot rows
* Redis hot keys

This gives us another reusable principle:

> **Distributed systems can move a bottleneck rather than eliminate it.**

The partition key must balance:

1. distribution
2. ordering requirements

---

# 5.5.9 Consumer Lag

This is probably the most important operational metric for Kafka.

Suppose producers generate:

```text
10,000 events/sec
```

but consumers process:

```text
7,000 events/sec
```

Then:

```text
incoming > processing
```

The backlog grows.

Kafka's **consumer lag** increases.

Conceptually:

```text
Events produced
████████████████████

Events consumed
██████████████

Remaining
██████
```

Lag tells us that consumers are falling behind.

---

# 5.5.10 What Happens When Consumer Lag Increases?

Suppose:

```text
PaymentSucceeded
```

is waiting in Kafka.

The payment event may not immediately trigger downstream processing.

Therefore:

```text
Kafka lag
   ↓
processing delay
   ↓
business workflow delay
```

For example:

```text
Payment succeeds
   ↓
event enters Kafka
   ↓
consumer is overloaded
   ↓
event waits 30 seconds
   ↓
reservation confirmation delayed
```

This might be acceptable for some workloads but not others.

So we need **business-specific lag thresholds**.

---

# 5.5.11 How Do We Scale Consumers?

Suppose:

```text
8 partitions
8 consumers
```

and lag is increasing.

We can scale the consumer group:

```text
8 consumers
      ↓
16 consumers
```

But if there are only 8 partitions:

```text
8 active
8 idle
```

So consumer scaling works only up to the available partition parallelism.

Therefore:

```text
Partitions
    ↓
maximum parallelism
    ↓
Consumers
```

This relationship is very important.

---

# 5.5.12 Slow Consumer vs Slow Dependency

Suppose Payment Consumer is slow.

Why?

It might not actually be Kafka's fault.

Maybe:

```text
Kafka
  ↓
Payment Consumer
  ↓
PostgreSQL
  ↓
slow query
```

or:

```text
Kafka
  ↓
Consumer
  ↓
Payment Provider
  ↓
slow external API
```

Adding more consumers might make the downstream problem worse.

For example:

```text
10 consumers
    ↓
10 concurrent DB operations
```

becomes:

```text
50 consumers
    ↓
50 concurrent DB operations
    ↓
DB overloaded
```

So:

> **Consumer scaling must be downstream-aware.**

This connects directly to what we learned in **5.2 Application Scaling**.

---

# 5.5.13 Consumer Concurrency

There are several levels of parallelism:

```text
Kafka partitions
      ↓
Consumer instances
      ↓
Consumer concurrency
      ↓
Downstream operations
```

We need to control all of them.

For example:

```text
8 partitions
8 consumers
each consumer processes 5 messages concurrently
```

Potential downstream concurrency:

```text
8 × 5 = 40
```

If each message performs a DB operation, that could mean 40 concurrent DB operations.

Therefore, blindly increasing consumer concurrency can overload PostgreSQL.

---

# 5.5.14 Backpressure

Suppose:

```text
Kafka → Consumer → PostgreSQL
```

PostgreSQL is becoming saturated.

We shouldn't necessarily keep increasing consumer concurrency.

Instead:

```text
Kafka
  ↓
Consumer
  ↓
controlled concurrency
  ↓
PostgreSQL
```

The consumer slows processing to match downstream capacity.

That's **backpressure**.

The broader principle:

> **The fastest component should not overwhelm the slowest component.**

---

# 5.5.15 What If a Consumer Crashes?

Kafka keeps the event because it is durable.

Another consumer can take over the partition after rebalancing.

Conceptually:

```text
P2 → Consumer 3 ❌

P2 → Consumer 4
```

This provides fault tolerance.

But there are important delivery semantics.

---

# 5.5.16 Duplicate Processing

Suppose consumer does:

```text
1. Read event
2. Update PostgreSQL
3. Crash before acknowledging/committing Kafka progress
```

Kafka may deliver the same event again.

So:

```text
Event X
  ↓
Consumer
  ↓
DB update ✅
  ↓
Crash ❌
  ↓
Event X delivered again
```

Therefore consumers should be **idempotent**.

We already covered this during Saga/Outbox.

For example:

```text
event_id = E123
```

and consumer records:

```text
processed_event(E123)
```

or performs an idempotent state transition.

---

# 5.5.17 Poison Messages

Suppose one event repeatedly causes a consumer failure.

```text
Event X
 ↓
Consumer fails
 ↓
retry
 ↓
fails
 ↓
retry
 ↓
fails
```

This can block progress or waste resources depending on the processing model.

A common solution is a:

> **Dead-letter queue/topic (DLQ/DLT)**

Conceptually:

```text
Kafka
  ↓
Consumer
  ↓
processing fails repeatedly
  ↓
Dead Letter Topic
```

Then the problematic event can be investigated separately.

But we shouldn't send an event to DLQ immediately for every transient error.

For example:

```text
temporary DB timeout
```

should generally be retried.

Whereas:

```text
invalid event schema
```

may be a candidate for DLQ after appropriate retry policy.

---

# 5.5.18 Retry Storms

Imagine a downstream payment service is temporarily unavailable.

Every consumer retries immediately:

```text
Consumer 1 → retry
Consumer 2 → retry
Consumer 3 → retry
...
```

This can create:

```text
failure
  ↓
retry
  ↓
more traffic
  ↓
more failure
  ↓
more retry
```

So retries should use things like:

* exponential backoff
* jitter
* bounded retries
* circuit breakers where appropriate
* DLQ for non-recoverable failures

This is the same failure-amplification problem we discussed earlier.

---

# 5.5.19 Kafka vs Synchronous API Calls

Why do we even need Kafka?

Suppose reservation service directly calls:

```text
Reservation → Notification
Reservation → Analytics
Reservation → Cache
```

Now the reservation flow depends on all those services.

Kafka allows:

```text
Reservation
    ↓
Kafka
    ↓
+----------+----------+----------+
|          |          |          |
Cache   Notification Analytics  ...
```

The reservation service doesn't need to synchronously wait for every downstream consumer.

This provides:

* decoupling
* asynchronous processing
* buffering
* independent scaling
* fault isolation

But it also introduces:

* eventual consistency
* processing delay
* duplicate handling
* operational complexity
* more difficult debugging

So Kafka isn't automatically better than synchronous calls.

Use it where asynchronous decoupling provides real value.

---

# 5.5.20 What Goes Through Kafka in Our System?

Good candidates:

```text
ReservationCreated
ReservationConfirmed
ReservationCancelled
ReservationExpired
PaymentSucceeded
PaymentFailed
RefundRequested
```

Potential consumers:

```text
Kafka
 |
 +-- Cache updater
 +-- Notification service
 +-- Analytics
 +-- Reconciliation
 +-- Audit/event processing
```

But the core reservation transaction itself should not become:

```text
Reserve slot
   ↓
Kafka
   ↓
eventually decide whether slot is reserved
```

That would weaken the correctness model we've already established.

Instead:

```text
Reservation request
      ↓
PostgreSQL transaction
      ↓
reservation committed
      ↓
Outbox
      ↓
Kafka
      ↓
async downstream processing
```

---

# 5.5.21 Scaling Strategy for Our Parking System

A reasonable approach:

### Step 1 — Identify event volume

```text
events/sec
```

### Step 2 — Identify processing requirements

For example:

```text
Payment reconciliation → high importance
Analytics → can tolerate delay
Notifications → moderate delay acceptable
Cache update → should be reasonably fast
```

### Step 3 — Choose partitioning strategy

For events requiring per-reservation ordering:

```text
key = reservationId
```

subject to workload distribution.

### Step 4 — Monitor consumer lag

```text
lag
lag growth rate
processing latency
```

### Step 5 — Scale consumers

But only when:

```text
partitions available
+
downstream capacity available
```

### Step 6 — Handle failures

```text
retry
backoff
idempotency
DLQ
reconciliation
```

---

# 5.5.22 Senior Interview Question

### Interviewer:

> "Kafka consumer lag is continuously increasing. What would you do?"

A strong answer isn't simply:

> "Add more consumers."

I'd say:

> "First I'd determine why lag is increasing: whether producer throughput has increased, consumers are processing slowly, partitions are unevenly distributed, or a downstream dependency such as the database or an external provider is limiting throughput. I'd check partition-level lag and consumer processing latency. If partitions have sufficient parallelism and downstream capacity is available, I'd scale the consumer group. Otherwise I'd optimize processing or increase partition parallelism where appropriate. I'd also verify retries aren't creating a retry storm and ensure consumers are idempotent."

That's the level of reasoning we're aiming for.

---

# 5.5.23 What to Remember for ANY System-Design Problem

The reusable mental model:

```text
Producer
   ↓
Queue / Kafka
   ↓
Partitions
   ↓
Consumers
   ↓
Downstream dependency
```

At every stage ask:

```text
How much traffic?
      ↓
How much parallelism?
      ↓
Where is the bottleneck?
      ↓
Can we scale it?
      ↓
Will scaling overload downstream?
      ↓
What happens during failure?
```

And remember these five points:

1. **Partitions provide Kafka parallelism.**
2. **Consumers within one group cannot exceed useful partition-level parallelism.**
3. **Consumer lag means processing capacity is behind incoming workload.**
4. **Scaling consumers can overload downstream systems.**
5. **Partition-key choice must balance ordering requirements and workload distribution.**

---

# 5.5 COMPLETE ✅

Current roadmap:

* 5.1 Capacity & Bottleneck Identification — ✅
* 5.2 Application/API Scaling — ✅
* 5.3 Database Scaling — ✅
* 5.4 Cache Scaling — ✅
* **5.5 Queue/Kafka Scaling — ✅**
* **5.6 Traffic Spikes & Backpressure — NEXT**
* 5.7 High Availability
* 5.8 End-to-End Bottleneck Analysis

Next we'll connect several things we've already learned—**traffic spikes, queues, backpressure, admission control, rate limiting, load shedding, retries, and graceful degradation**—into one production-level failure/scaling strategy.

