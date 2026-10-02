
Yes. Let's finish **all of 5.x today** and then move to **Major Topic 6: Security + Rate Limiting + Reliability**.

We are at:

* 5.1 Capacity & Bottleneck Identification — ✅
* 5.2 Application/API Scaling — ✅
* 5.3 Database Scaling — ✅
* 5.4 Cache Scaling — ✅
* 5.5 Queue/Kafka Scaling — ✅
* **5.6 Traffic Spikes & Backpressure — ⬅️**
* 5.7 High Availability
* 5.8 End-to-End Bottleneck Analysis

# 5.6 Traffic Spikes & Backpressure

So far we've mostly discussed **normal scaling**.

Now consider an abnormal situation.

Our parking system normally receives:

```text
Search:       16,700 req/sec
Reservations:  1,667 req/sec
Payments:        333 req/sec
```

Then suddenly there's a major event nearby and traffic becomes:

```text
Search:       100,000 req/sec
Reservations:  10,000 req/sec
```

Our system has finite capacity.

The important question isn't:

> "Can we handle unlimited traffic?"

It's:

> **"What does the system do when demand temporarily exceeds capacity?"**

---

## 5.6.1 The Basic Problem

Suppose an API can sustainably process:

```text
5,000 requests/sec
```

but incoming traffic is:

```text
10,000 requests/sec
```

Then:

```text
Incoming
10,000/sec
    ↓
+----------------+
| Application    |
| Capacity 5K/s  |
+----------------+
    ↓
5,000/sec
```

The remaining requests have to:

* wait
* queue
* be rejected
* be shed
* or eventually time out

If we simply allow everything in:

```text
traffic spike
    ↓
more concurrent requests
    ↓
DB connections increase
    ↓
DB overload
    ↓
latency increases
    ↓
timeouts
    ↓
clients retry
    ↓
even more traffic
```

This is how a traffic spike can become a **cascading failure**.

---

# 5.6.2 Backpressure

**Backpressure** means:

> When a downstream component cannot safely process work at the incoming rate, the upstream component must slow down, limit, buffer, or reject work.

Example:

```text
Client
  ↓
API
  ↓
Database
```

If DB capacity is:

```text
1,000 concurrent operations
```

we shouldn't allow:

```text
10,000 concurrent DB operations
```

Instead:

```text
10,000 requests
       ↓
Concurrency control
       ↓
1,000 DB operations
```

The remaining requests may wait briefly or receive a controlled response.

---

# 5.6.3 Backpressure vs Queueing

They're related but not identical.

### Queueing

We accept work and put it somewhere to process later:

```text
Request
   ↓
Queue
   ↓
Worker
```

Useful for asynchronous work.

### Backpressure

We prevent the upstream from overwhelming downstream:

```text
Producer
   ↓
[controlled rate]
   ↓
Consumer
```

Backpressure can involve:

* limiting concurrency
* slowing producers
* bounded queues
* rejecting work
* load shedding

---

# 5.6.4 Why We Can't Queue Everything

Suppose a user wants to reserve a parking slot.

If the slot is held for:

```text
5 minutes
```

and we put 20,000 reservation requests into a queue:

```text
Request 1
Request 2
...
Request 20,000
```

By the time request #15,000 is processed, the user's reservation hold may no longer make sense.

This is why:

> **Queues are excellent for asynchronous work but aren't automatically appropriate for latency-sensitive interactive operations.**

For booking, controlled admission/rejection may be preferable to an unlimited queue.

---

# 5.6.5 Bounded Queues

A queue should generally have a capacity.

Bad:

```text
Queue
∞
```

because eventually memory/storage and latency become the problem.

Better:

```text
Queue
Capacity = 10,000
```

Once full:

```text
new work
   ↓
reject / shed / fallback
```

This creates a predictable failure mode.

---

# 5.6.6 Load Shedding

**Load shedding** means deliberately rejecting lower-priority or excess work to keep the core system alive.

Imagine:

```text
System overloaded
      |
      +---- Search
      +---- Reservation
      +---- Payment
      +---- Analytics
      +---- Recommendations
```

We may protect:

```text
Reservation
Payment
```

while reducing:

```text
Analytics
Recommendations
```

The objective is:

> **Preserve critical functionality instead of allowing overload to take down everything.**

---

# 5.6.7 Graceful Degradation

Another strategy is to provide a reduced version of functionality.

For example:

Normally:

```text
Search
 ↓
availability
 ↓
pricing
 ↓
recommendations
 ↓
distance calculation
 ↓
personalization
```

During overload:

```text
Search
 ↓
basic availability
 ↓
basic pricing
```

We temporarily remove non-essential work.

This is **graceful degradation**.

For our system:

> Showing basic parking availability is more important than showing an advanced recommendation or analytics-derived ranking.

---

# 5.6.8 Rate Limiting vs Concurrency Limiting

These are different.

### Rate limiting

Controls:

> **How many requests are allowed over time.**

Example:

```text
100 requests/minute/user
```

### Concurrency limiting

Controls:

> **How many requests can execute simultaneously.**

Example:

```text
maximum 100 concurrent DB operations
```

You can have:

```text
10,000 requests/minute
```

but only:

```text
100 concurrent operations
```

depending on the system.

Both can be useful.

---

# 5.6.9 Protecting the Database

This connects everything we've learned.

Suppose:

```text
Node.js
   ↓
PostgreSQL
```

DB is already at 90% capacity.

If we continue accepting unlimited requests, the application can overwhelm it.

Instead:

```text
Request
   ↓
API admission control
   ↓
bounded concurrency
   ↓
PostgreSQL
```

This is **downstream-aware scaling**.

The application should not say:

> "I have 100 servers, therefore I can process 100× more DB operations."

The real capacity is determined by the downstream system.

---

# 5.6.10 Retry Storm During Traffic Spike

This deserves special attention.

Suppose DB temporarily returns errors.

Clients retry:

```text
Request
 ↓
failure
 ↓
retry
 ↓
failure
 ↓
retry
```

If 10,000 clients do this simultaneously:

```text
10,000 original requests
+
10,000 retries
+
10,000 retries
...
```

Traffic can multiply.

So retries need:

* exponential backoff
* jitter
* bounded retry count
* idempotency
* circuit breaking where appropriate

And not every error should be retried.

For example:

```text
400 Bad Request
```

usually shouldn't be retried.

A temporary:

```text
503 Service Unavailable
```

may be retryable.

---

# 5.6.11 Circuit Breaker

Suppose our application calls an external payment provider.

If the provider is failing:

```text
Node
 ↓
Payment Provider ❌
```

We shouldn't continuously hammer it.

A circuit breaker can transition roughly:

```text
CLOSED
   ↓ failures
OPEN
   ↓
stop sending requests
   ↓
after cooldown
   ↓
HALF-OPEN
   ↓
test requests
   ↓
healthy → CLOSED
```

This protects both systems.

---

# 5.6.12 Admission Control

Admission control answers:

> **Should this request be allowed into the expensive part of the system?**

Example:

```text
10,000 requests
       ↓
Admission control
       |
       +---- 1,000 accepted
       |
       +---- 9,000 rejected/deferred
```

This sounds harsh, but:

> **Controlled rejection is often better than accepting everything and causing total system failure.**

---

# 5.6.13 Parking-System Example

Suppose a highly popular parking lot has:

```text
2 available slots
```

and:

```text
5,000 users
```

attempt to reserve simultaneously.

We shouldn't necessarily allow all 5,000 requests to hammer the same DB row.

We could combine:

```text
rate limiting
+
concurrency control
+
DB row locking
+
short transaction
+
inventory hold
```

The DB still decides correctness.

The upper layers reduce unnecessary pressure.

---

# 5.6 COMPLETE ✅

### What to remember for ANY system-design problem

When traffic exceeds capacity:

```text
Detect overload
     ↓
Control admission
     ↓
Apply backpressure
     ↓
Bound concurrency/queues
     ↓
Protect dependencies
     ↓
Retry carefully
     ↓
Degrade non-critical features
     ↓
Recover gradually
```

The most important principle:

> **It is better to serve fewer requests correctly than to accept everything and bring down the entire system.**
