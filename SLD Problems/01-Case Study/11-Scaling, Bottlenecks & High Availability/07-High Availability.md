# 5.7 High Availability

Now we move from:

> "Can the system handle more traffic?"

to:

> **"What happens when something fails?"**

At senior level, scalability without availability isn't enough.

---

## 5.7.1 Single Point of Failure

Imagine:

```text
Load Balancer
      |
   Node.js
      |
 PostgreSQL
```

If the only PostgreSQL instance fails:

```text
PostgreSQL ❌
     ↓
entire booking system ❌
```

That's a single point of failure.

HA means designing so that failure of one component doesn't necessarily take down the entire system.

---

# 5.7.2 Application HA

We already covered this.

Instead of:

```text
Node 1
```

use:

```text
        Load Balancer
          /       \
       Node 1    Node 2
```

If Node 1 fails:

```text
Node 1 ❌

Load Balancer
     ↓
Node 2
```

No single application instance is required for the system to function.

---

# 5.7.3 Availability Zones

For production systems, putting everything on one physical failure domain isn't enough.

Conceptually:

```text
                Load Balancer
                 /         \
                /           \
             AZ-A          AZ-B
              |              |
           Node 1          Node 2
           Node 3          Node 4
```

If AZ-A has a major failure:

```text
AZ-A ❌
```

AZ-B can continue serving traffic, assuming it has sufficient capacity.

This is why production HA often means:

> **Multiple failure domains, not merely multiple instances.**

---

# 5.7.4 Database HA

For PostgreSQL:

```text
              Primary
             /       \
            /         \
      Standby A     Standby B
```

If the primary fails:

```text
Primary ❌
    ↓
Standby promoted
```

This provides database availability.

But there's an important distinction:

> **Replication is not the same as backup.**

Replication helps with availability.

Backups help with recovery from:

* accidental deletion
* corruption
* bad deployments/data changes
* certain disaster scenarios

We need both.

---

# 5.7.5 Database Failure

Suppose PostgreSQL primary becomes unavailable.

The application may see:

```text
connection failures
timeouts
```

A production system should have:

```text
DB HA/failover
+
connection retry/recovery
+
health detection
+
transaction retry where safe
```

But retries must be careful.

For example, if:

```text
INSERT reservation
```

timed out, we cannot automatically assume:

```text
INSERT failed
```

It may have committed successfully.

This connects directly to our earlier **idempotency/reconciliation** discussion.

---

# 5.7.6 Redis Failure

We already covered this:

```text
Redis ❌
```

should ideally not cause:

```text
whole application ❌
```

because Redis isn't our source of truth.

But we must protect PostgreSQL from the resulting cache misses.

So:

```text
Redis failure
   ↓
fallback
   ↓
controlled DB traffic
```

rather than:

```text
Redis failure
   ↓
every request → DB
   ↓
DB overload
```

---

# 5.7.7 Kafka Failure

Suppose Kafka becomes temporarily unavailable.

Because we use:

```text
DB transaction
+
Outbox
```

the reservation itself can still be committed.

The event remains in:

```text
Outbox
```

until Kafka becomes available.

So:

```text
Reservation
    ↓
PostgreSQL ✅
    ↓
Outbox ✅
    ↓
Kafka ❌
```

Later:

```text
Kafka recovers
    ↓
Outbox publisher
    ↓
Kafka
```

This is exactly why we introduced the Outbox pattern earlier.

---

# 5.7.8 External Payment Provider Failure

This is another important failure domain.

```text
Parking System
      ↓
Payment Provider ❌
```

We don't want to mark every transaction as permanently failed just because the network/provider is unavailable.

We use the state machine we already designed:

```text
PENDING
   ↓
UNKNOWN
   ↓
reconciliation
```

Then:

```text
Provider status
   ↓
SUCCESS / FAILED
```

Again:

> **Unknown is a valid distributed-systems state.**

---

# 5.7.9 Availability vs Consistency Trade-off

Here's an important architectural question.

Suppose PostgreSQL is unavailable.

Should we allow reservations using Redis?

No.

Because that would violate our correctness requirement.

We would rather:

```text
Booking temporarily unavailable
```

than:

```text
Two users successfully book the same slot
```

So for this particular operation:

> **Correctness is more important than availability.**

But for a search operation, we may potentially serve cached/stale data.

This demonstrates:

> **Availability requirements can differ by operation within the same system.**

---

# 5.7.10 Failure Isolation

We also want one subsystem's failure not to cascade everywhere.

For example:

```text
Analytics ❌
```

should not cause:

```text
Reservation ❌
Payment ❌
```

Kafka helps here by decoupling asynchronous workloads.

Similarly:

```text
Notification service ❌
```

shouldn't cause:

```text
reservation transaction rollback
```

if notification isn't part of the critical transaction.

This is **failure isolation**.

---

# 5.7.11 Graceful Shutdown

HA isn't only about crashes.

Deployments also cause instances to disappear.

Correct sequence:

```text
Remove instance from LB
       ↓
Stop accepting new requests
       ↓
Finish in-flight requests
       ↓
Commit/rollback active work
       ↓
Close connections
       ↓
Terminate
```

This prevents unnecessary failed requests during deployment.

---

# 5.7.12 Health Checks

We previously distinguished:

### Liveness

> Is the process alive?

### Readiness

> Can this instance safely receive traffic?

Suppose Node.js is running but:

```text
DB connection pool completely exhausted
```

The process is alive.

But it may not be ready to handle requests.

Therefore:

```text
Liveness ≠ Readiness
```

This is important for load balancers and orchestrators.

---

# 5.7 COMPLETE ✅

### What to remember for ANY system-design problem

Think in terms of failure domains:

```text
Application
Database
Cache
Queue
External providers
Network
Availability Zone
Region
```

For every major dependency ask:

1. What happens if it fails?
2. Is there a redundant instance?
3. Can we fail over?
4. Can the application degrade?
5. Can we recover?
6. Can failure cascade into another dependency?
