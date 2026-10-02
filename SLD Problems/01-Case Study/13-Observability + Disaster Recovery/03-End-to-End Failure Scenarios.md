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
