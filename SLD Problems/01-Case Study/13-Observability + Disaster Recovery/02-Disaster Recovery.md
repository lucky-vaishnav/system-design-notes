We've already covered **High Availability (HA)** in Topic 5. The important distinction now is:

> **HA handles failures we expect during normal operation. Disaster Recovery handles large-scale failures that may take down an entire environment or region.**

---

# 7.7 Disaster Recovery

Consider our parking system running normally:

```text
Users
  ↓
Load Balancer
  ↓
Node.js instances
  ↓
Redis + PostgreSQL
  ↓
Kafka
  ↓
External Payment Provider
```

Suppose one Node.js instance crashes.

That's an **HA problem**.

We replace it with another instance.

But suppose:

```text
Entire AWS region unavailable
```

or:

```text
Database is corrupted
```

or:

```text
Accidental deletion of critical data
```

or:

```text
Major infrastructure failure
```

Now simply having multiple instances isn't enough.

That's where **Disaster Recovery (DR)** comes in.

---

# HA vs DR

This distinction is very important in interviews.

| High Availability                         | Disaster Recovery                        |
| ----------------------------------------- | ---------------------------------------- |
| Handles component/failure-domain failures | Handles major disasters                  |
| Usually automatic                         | May be automatic or manual               |
| Multiple instances/AZs                    | Backup/replication/secondary environment |
| Goal: minimal interruption                | Goal: recover service/data               |
| Example: EC2/container dies               | Example: entire region fails             |

### Simple mental model

> **HA keeps the system running. DR gets the system back when the primary environment cannot continue.**

---

# 7.8 RPO and RTO

These are probably the two most important DR concepts.

## RPO — Recovery Point Objective

RPO answers:

> **How much data can we afford to lose?**

Suppose our database backup/replication strategy means that after a disaster we can recover data only up to:

```text
10:00 AM
```

but the disaster occurs at:

```text
10:15 AM
```

Potential data loss:

```text
15 minutes
```

Therefore:

```text
RPO = 15 minutes
```

---

## RTO — Recovery Time Objective

RTO answers:

> **How long can the system be unavailable?**

Suppose the disaster happens at:

```text
10:15 AM
```

and service is restored at:

```text
10:45 AM
```

Then:

```text
RTO = 30 minutes
```

---

## The easiest way to remember

> **RPO = data loss**

> **RTO = downtime**

---

# Example

Suppose the business requirements are:

```text
RPO = 5 minutes
RTO = 30 minutes
```

That means:

* We should lose no more than approximately 5 minutes of data.
* We should restore service within approximately 30 minutes.

These requirements directly influence architecture.

For example, an RPO of a few seconds requires a very different replication/backup strategy than an RPO of 24 hours.

---

# 7.9 Backup & Restore

A common interview mistake is saying:

> "We have database replication, so we're safe."

Replication is **not a replacement for backups**.

Why?

Imagine someone accidentally executes:

```sql
DELETE FROM reservations;
```

If the deletion is replicated immediately to the standby:

```text
Primary DB
   ↓
DELETE
   ↓
Replica
   ↓
DELETE
```

The replica may contain the same bad state.

Backups provide historical recovery points.

---

## Types of protection

Conceptually:

```text
Database
 ├── Replication
 ├── Backups
 └── Point-in-time recovery
```

### Replication

Useful for:

* HA
* failover
* reducing recovery time

### Backup

Useful for:

* accidental deletion
* corruption
* ransomware/security incidents
* historical recovery

### Point-in-time recovery

Allows us to recover to a particular point before the problem occurred, depending on the database/backup architecture.

---

# Backup strategy

For our system, important data includes:

```text
Users
Reservations
Payments
Refunds
Inventory
Idempotency records
Outbox events
```

We need to decide:

* How frequently are backups taken?
* How long are they retained?
* Are backups encrypted?
* Are they stored separately from the primary environment?
* Can we restore them?
* How often do we test restoration?

And the last one is particularly important.

> **A backup that has never been restored/tested is not something we should blindly trust.**

---

# Backup vs replication

Keep this distinction very clear:

```text
Replication
→ Helps keep another copy current
→ Mainly HA/failover

Backup
→ Preserves recoverable historical state
→ Mainly DR/data recovery
```

You generally want both.

---

### 7.9 Backup & Restore — COMPLETE

---

# 7.10 Regional Failure

Now let's consider the serious scenario:

```text
Entire Region A
      ❌
```

Our architecture currently lives there:

```text
Region A
 ├── Node.js
 ├── PostgreSQL
 ├── Redis
 └── Kafka
```

If the entire region disappears, having multiple AZs **inside Region A** doesn't solve the problem.

We need some form of cross-region DR strategy.

---

# Cross-region strategies

There are several possibilities.

## 1. Backup-based recovery

Primary:

```text
Region A
```

Backups:

```text
Region B / separate storage
```

When Region A fails:

```text
Restore infrastructure/data
       ↓
Region B
       ↓
Start services
       ↓
Recover traffic
```

### Advantage

Simpler and cheaper.

### Disadvantage

Recovery takes longer.

RTO may be relatively high.

---

# 2. Warm standby

Region B already has infrastructure running at reduced capacity.

```text
Region A
FULL
  ↓
Primary

Region B
SMALL
  ↓
Standby
```

During disaster:

```text
Region A ❌

Region B
scale up
   ↓
receive traffic
```

Faster than rebuilding everything from scratch.

More expensive than backup-only recovery.

---

# 3. Active-passive

Region A:

```text
ACTIVE
```

Region B:

```text
PASSIVE
```

Traffic normally goes to Region A.

If Region A fails:

```text
Traffic
   ↓
Region B
```

Data replication needs to support the required RPO.

---

# 4. Active-active

Both regions serve traffic:

```text
          Traffic
          /     \
         ↓       ↓
    Region A   Region B
      ACTIVE     ACTIVE
```

This can provide very fast failover.

But now things become significantly more complicated.

---

# The difficult part: data consistency

Imagine a reservation:

```text
Parking Lot A
10 available slots
```

User 1 goes to Region A.

User 2 goes to Region B.

Both attempt to reserve the **last available slot**.

If both regions independently accept the booking:

```text
Region A → reservation SUCCESS
Region B → reservation SUCCESS
```

We have:

> **Double booking.**

This is why multi-region active-active systems are much harder for strongly consistent workloads.

---

# This is a key architecture principle

Not every workload should automatically become active-active.

For our parking system:

### Search

Could potentially be:

```text
active-active
```

because search can tolerate eventual consistency.

### Analytics

Definitely can be:

```text
active-active / asynchronous
```

depending on architecture.

### Reservation inventory

Much harder.

We need a clear authoritative ownership/consistency strategy.

### Payment state

Also requires careful consistency and reconciliation.

---

# Possible approach

One conceptual design is to assign ownership:

```text
Parking Lot A → Region A
Parking Lot B → Region B
Parking Lot C → Region A
```

Then reservation writes for a parking lot go to its authoritative region.

But this introduces:

* routing complexity
* failover complexity
* ownership transfer
* cross-region latency
* recovery coordination

So again:

> **Don't introduce multi-region active-active simply because it sounds scalable. Choose it based on RPO/RTO and consistency requirements.**

---

# 7.10 Regional Failure — COMPLETE
