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

---

# 7.11 End-to-End Failure Scenarios

This is where we combine everything we've learned.

Rather than memorizing isolated patterns, we should reason through failures.

---

## Scenario 1: Node.js instance crashes

```text
Node instance ❌
```

What happens?

```text
Load Balancer
    ↓
Other healthy instances
```

Health/readiness checks remove the failed instance.

Requests continue.

### Patterns involved

* Horizontal scaling
* Load balancing
* Health checks
* Graceful recovery
* Observability

This is primarily an **HA problem**, not a DR problem.

---

# Scenario 2: Redis goes down

We already established:

> Redis is not the source of truth.

So:

```text
Redis ❌
   ↓
Fallback to PostgreSQL
```

But there is a danger:

```text
Redis failure
   ↓
100% cache misses
   ↓
PostgreSQL receives huge traffic
   ↓
DB overload
```

So we need:

* controlled fallback
* rate/concurrency limiting
* cache recovery
* protection against DB overload
* monitoring

Potentially temporarily degrade search functionality rather than allowing the DB to collapse.

---

# Scenario 3: PostgreSQL primary fails

If we have HA:

```text
Primary DB ❌
       ↓
Standby
       ↓
Failover
```

But there may be:

* failover delay
* in-flight transactions
* replication lag
* connection failures
* application reconnection

The system should retry/reconnect appropriately.

For correctness-critical booking operations, we should **not pretend writes succeeded** if the DB outcome is unknown.

---

# Scenario 4: Payment provider times out

We already solved this in Major Topic 3.

```text
Payment request
      ↓
Timeout
```

We do **not** immediately assume:

```text
FAILED
```

Instead:

```text
UNKNOWN
   ↓
persist state
   ↓
reconciliation
   ↓
query provider
```

If provider says:

```text
SUCCESS
```

we continue confirmation.

If reservation expired and payment succeeded:

```text
SUCCESS payment
      +
reservation unavailable
      ↓
compensation/refund
```

---

# Scenario 5: Kafka unavailable

Suppose:

```text
PostgreSQL transaction
       ↓
Outbox
       ↓
Kafka ❌
```

The reservation transaction should still succeed because:

```text
Business state + Outbox event
```

were committed to PostgreSQL.

The publisher can retry later:

```text
Outbox
  ↓
retry
  ↓
Kafka
```

This is exactly why we introduced the **Outbox pattern**.

---

# Scenario 6: Kafka consumer crashes

Suppose:

```text
Kafka
  ↓
Payment consumer
       ❌
```

Messages remain available according to Kafka's retention/consumer-offset strategy.

Another consumer can process them.

But duplicates are possible.

Therefore:

```text
event_id
   ↓
idempotent processing
```

Again, our earlier concepts connect.

---

# Scenario 7: Entire region fails

Now:

```text
Region A ❌❌❌
```

We use our DR strategy.

Depending on the chosen architecture:

```text
Backup-based
        OR
Warm standby
        OR
Active-passive
        OR
Active-active
```

The choice depends heavily on:

```text
RPO
RTO
Consistency requirements
Cost
Operational complexity
Business requirements
```

---

# Scenario 8: Database corruption

This is different from a normal DB instance failure.

Suppose bad data is written:

```text
Application bug
    ↓
incorrect updates
    ↓
DB corrupted
```

Replication may replicate the corruption.

So:

```text
HA alone ❌
```

We need:

```text
Backup
+
Point-in-time recovery
+
Restore testing
```

---

# Final Major Topic 7 mental model

This is what I want you to retain for interviews:

```text
                OBSERVABILITY
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
      Logs         Metrics      Traces
        │            │            │
        └─────── Correlation ─────┘
                     │
                   Alerts
                     │
                    SLO
                     │
              "Is system healthy?"
                     │
                     ↓
              FAILURE / DISASTER
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
         HA                    DR
    component failure      major disaster
          │                     │
    failover/redundancy    backup/replication
                                │
                         ┌──────┴──────┐
                         ↓             ↓
                        RPO           RTO
                     data loss      downtime
```

## The reusable interview framework

When asked:

> **"How would you make this system observable and recoverable?"**

Think:

1. **Logs** — What happened?
2. **Metrics** — How often/how badly?
3. **Traces** — Where did the request spend time?
4. **Correlation IDs** — How do I follow one request/workflow?
5. **Alerts** — When should humans act?
6. **SLI/SLO** — What does healthy mean?
7. **HA** — How do we survive normal infrastructure failures?
8. **Backups/replication** — How do we recover data?
9. **RPO** — How much data can we lose?
10. **RTO** — How quickly must we recover?
11. **Regional strategy** — What happens if the whole environment disappears?
12. **Restore testing** — Can we actually recover?

---
### Question - How we can do Restore testing? we can use a new env like staging env or prod copy env to test this?
Yes — exactly. **Restore testing means periodically taking an actual backup and proving that we can successfully restore it and use the restored data.**

It is not enough to say:

> “We have daily backups, so we're safe.”

We want to know:

> “If production is lost or corrupted today, can we actually restore a usable database within our RTO, and will the data be correct up to our RPO?”

### Should we replace the production DB with the backup?

**No, normally you should not overwrite the live production database just to test a backup.** That would create unnecessary risk.

Instead, you typically restore the backup into an **isolated environment**.

For example:

```text
PRODUCTION
    │
    ├── PostgreSQL
    │
    └── Backups
          │
          │ restore
          ↓
    DR TEST ENVIRONMENT
          │
          └── Restored PostgreSQL
```

That environment could be:

* a dedicated DR/test account or environment
* a temporary isolated database
* a separate restore instance
* sometimes a staging environment, **but preferably not the normal shared staging DB**

---

## Your STG idea is correct, with one adjustment

You could have something like:

```text
PROD
  ↓
Production Backup
  ↓
Restore
  ↓
Temporary DR/Restore Environment
  ↓
Validate
```

You *could* use a dedicated staging environment:

```text
PROD Backup
    ↓
STG Restore DB
    ↓
Run validation
```

But I wouldn't blindly restore over the normal STG database because other developers/tests may depend on it.

A better production setup is:

```text
Production
    ↓
Backup
    ↓
Temporary isolated restore environment
    ↓
Validation
    ↓
Destroy environment
```

---

# What exactly do we test?

Suppose we have a PostgreSQL backup from production.

We restore it:

```text
backup-2026-10-02
        ↓
restore
        ↓
postgres-restore-test
```

Then we verify things like:

### 1. Can the database actually start?

```text
DB starts successfully
```

### 2. Are expected tables present?

```text
users
reservations
payments
refunds
inventory
outbox
```

### 3. Is the data actually there?

For example:

```sql
SELECT COUNT(*) FROM reservations;
```

Compare expected characteristics against the source/backup metadata.

### 4. Can the application connect?

Point a **non-production application instance** to the restored DB:

```text
Test Node.js API
       ↓
Restored PostgreSQL
```

Then perform read-only or controlled functional tests:

```text
GET reservation
GET payment
GET booking history
```

### 5. Can critical workflows operate?

For example:

```text
Create test reservation
Read reservation
Cancel reservation
```

But be careful with external integrations. You don't want your restore test accidentally charging a real payment provider.

So external services should use:

```text
sandbox/mock/test provider
```

or be disabled.

---

# You can also test Point-in-Time Recovery

This is even more valuable.

Suppose:

```text
10:00 → valid database state
10:15 → application bug
10:20 → bad data written
```

You might test:

> Can we restore the database to 10:14?

Conceptually:

```text
Backup + WAL/logs
       ↓
Point-in-time restore
       ↓
10:14 database
```

Then verify the expected data exists and the corrupted state is absent.

---

# And there's another important thing: measure the restore time

This connects directly to **RTO**.

Suppose the requirement is:

```text
RTO = 30 minutes
```

You perform a restore test and discover:

```text
Backup restore      25 min
DB validation        10 min
Application startup   5 min
--------------------------------
Total                40 min
```

Then you have discovered something important:

> Your actual recovery process currently takes ~40 minutes, while the target is 30 minutes.

So the restore test isn't just checking whether the backup works.

It also validates whether your **DR process can meet the required RTO**.

---

# A realistic production process

For our parking system, I'd think about it like this:

```text
                 PRODUCTION
                     │
                     ↓
              Automated backups
                     │
                     ↓
          Separate backup storage
                     │
                     ↓
            Scheduled DR test
                     │
                     ↓
        Create isolated restore DB
                     │
                     ↓
              Restore backup
                     │
            ┌────────┴─────────┐
            ↓                  ↓
       Data validation     DB validation
            │                  │
            └────────┬─────────┘
                     ↓
             Application test
                     ↓
          Critical workflow tests
                     ↓
          Measure recovery time
                     ↓
              Record results
                     ↓
             Destroy test env
```

And importantly, the backup storage itself should be protected from the production environment as appropriate. Otherwise a production compromise could potentially affect both the primary data and backups.

---

## How often?

There's no universal interval.

For example, an organization might do:

* automated backups continuously/daily
* automated restore validation periodically
* full DR exercises periodically
* larger regional-failure exercises less frequently

The frequency should depend on the system's **RPO, RTO, business criticality, and operational requirements**.

---

## One more senior-level point

There are actually **two separate questions**:

### Backup test

> **Can I restore the data?**

### DR test

> **Can I restore the entire service and meet my RTO/RPO?**

For example:

```text
Backup restore
       ↓
Database works
       ↓
But...
       ↓
Application configuration missing
Secrets unavailable
DNS not configured
Kafka unavailable
Network rules wrong
Payment integration broken
```

The database backup may be perfectly good, while the **overall disaster recovery process fails**.

That's why mature systems test the entire recovery procedure, not just the backup file.

### Interview takeaway

If an interviewer asks:

> **“How do you know your backups actually work?”**

A strong answer is:

> “We periodically restore production backups into an isolated environment and validate database integrity, application connectivity, and critical workflows. We also measure the restoration time against our RTO and, where applicable, test point-in-time recovery against our RPO. We don't overwrite the production database during these tests.”

That is exactly what **restore testing** means.

---
# Major Topic 7 — COMPLETE ✅

Our overall case-study roadmap is now:

```text
1. Transactions + Concurrency              COMPLETE
2. Consistency + Cache                    COMPLETE
3. Payment Failures + Reconciliation      COMPLETE
4. Saga + Outbox + Kafka                  COMPLETE
5. Scaling + Bottlenecks + HA             COMPLETE
6. Security + Rate Limiting + Reliability COMPLETE
7. Observability + Disaster Recovery      COMPLETE
```

### Next major topic

**8. Final Technology-Agnostic Architecture**

This is an important transition.

We have now learned the individual pieces. Next we'll **assemble everything into one coherent production-level architecture**, without jumping into AWS yet.

We'll walk through:

```text
Client
 ↓
Traffic / Load Balancing
 ↓
API layer
 ↓
Services
 ↓
Redis
 ↓
PostgreSQL
 ↓
Outbox
 ↓
Kafka
 ↓
Async consumers
 ↓
Payment / Notification / Analytics
```

and, more importantly, explain **why every component exists, what responsibility it owns, and what happens during failures**.

After that we move to **AWS mapping**, where we'll map this technology-agnostic architecture to actual AWS services and discuss alternatives/trade-offs.

