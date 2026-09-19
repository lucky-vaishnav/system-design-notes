### My Thoughts That Time -
Yeah, so this topic is a very, very important topic, and I think this concurrency control and the idempotency are the major pillar of the system design, and this should be handled properly. So, like you said that how this isolation system should be handled, the concurrency or the concurrent request. And so yeah, for that we can go from database level to the distributed locking for the distributed system. So yeah, first we should do the database level of locking, like select for update or skip the locked row. So this we can use, and so it will be for the database level, and database will be handling these things. So it will be row-level locking to avoid the same row to be updated.  

And when it comes to the type of locking, like this optimistic or pessimistic. So according to me, for the reservation, as this is a very synchronous process, and so it is an important flow, so we should use the pessimistic locking, I think, so that we can avoid the failure or the code crash or these type of things. So we should pessimistically think that this concurrency issue will be there. So I think we should use the pessimistic locking. And so yeah. And then it comes about the distributed locking. So there are two ways of distributed locking. One is that we can use Redis as well, but I think for the reservation or the banking projects, we should not use the Redis, because it is not a robust way to handle the concurrency. If the system goes down, then we need to handle those type of cases as well. So we can use the distributed lock for the distributed system. Yeah, I think that will also help. And yeah, and I would like from your side is that please explain these concurrency control approaches according to production right now used by the companies, and properly explain these things, so that this will also help me for the next system design problem. So I would like to properly explain me the production-level approaches right now used, what should be the best approaches, when to use for the concurrency controlling, and the same for the idempotency as well. So do the same for the idempotency, so that any doubt if I have, so that will be clear, and I will be more confident with these two topics, idempotency and concurrency control. And whenever these things come for a system design, so I can confidently answer these things and also explain in that way that how I will explain this thing to the interview whenever this type of question will come.  

### Rewritten for Notes

**How should we implement production-grade concurrency control and idempotency in a distributed parking reservation system? When should we use database locking, optimistic/pessimistic concurrency, unique constraints, distributed locks, and idempotency keys, and how should I explain these choices in a senior system-design interview?**

You are thinking about the right two pillars. The main thing I want to correct is this:

> **Concurrency control and idempotency solve different problems.**

Once that distinction is very clear, system-design questions become much easier.

---

# 1. First: Concurrency vs Idempotency

### Concurrency control

Answers:

> **"What if two different requests try to modify the same resource at the same time?"**

Example:

```text
User A ──┐
         ├──> Last parking spot
User B ──┘
```

Only one should succeed.

### Idempotency

Answers:

> **"What if the same request is accidentally sent multiple times?"**

Example:

```text
User A
  │
  ├── POST /reservations
  │
  └── network timeout
          ↓
      retry same request
```

We must not create two reservations.

So:

```text
Concurrency control → competing requests
Idempotency          → duplicate/retried request
```

A production system often needs **both**.

---

# 2. Concurrency Control — Production Hierarchy

Think about concurrency control in layers.

```text
Application
     ↓
Database transaction
     ↓
Row locking / optimistic locking
     ↓
Database constraints
     ↓
Distributed coordination (only when actually required)
```

You should **prefer the lowest layer that can correctly enforce the invariant**.

For our parking system, the database is usually the natural place to enforce reservation correctness.

---

# 3. Pessimistic Locking

Your understanding of `SELECT FOR UPDATE` is correct.

Suppose we model capacity with a row:

```text
ParkingInventory

station_id | date       | capacity | reserved
-----------+------------+----------+---------
101        | 2026-09-20 | 100      | 99
```

Two requests arrive:

```text
User A
User B
```

Both want the last available space.

Transaction A:

```sql
BEGIN;

SELECT *
FROM parking_inventory
WHERE station_id = 101
  AND date = '2026-09-20'
FOR UPDATE;
```

PostgreSQL locks that row. A second transaction trying to lock the same row has to wait until the first transaction commits or rolls back. ([PostgreSQL][1])

Then A checks:

```text
reserved < capacity
99 < 100
```

and does:

```sql
UPDATE parking_inventory
SET reserved = reserved + 1
WHERE station_id = 101
  AND date = '2026-09-20';
```

Commit.

Then B obtains the lock and sees:

```text
reserved = 100
capacity = 100
```

So B fails with:

```text
NO_AVAILABILITY
```

That's a strong design.

---

# 4. But `SELECT FOR UPDATE` Isn't Magic

This is an important senior-level detail.

`SELECT FOR UPDATE` locks the **existing row**. It doesn't magically lock a row that doesn't exist.

For example, this can be problematic:

```sql
SELECT *
FROM reservations
WHERE station_id = 101
AND parking_date = '2026-09-20'
FOR UPDATE;
```

If there are currently **zero reservations**, there may be no row to lock.

That's why an inventory/capacity model can be useful:

```text
ParkingInventory
----------------
station_id
date
capacity
reserved_count
```

Now there is a deterministic row to lock.

This is one reason I prefer an explicit **inventory/capacity row** for our case rather than trying to lock a query result of reservations.

---

# 5. `SKIP LOCKED` — Don't automatically use it

You mentioned:

> `SELECT FOR UPDATE SKIP LOCKED`

Good that you know it, but don't use it by default for reservations.

`SKIP LOCKED` means:

> "If another transaction has locked the row, skip it instead of waiting."

PostgreSQL specifically notes that this gives an inconsistent view and is primarily useful for queue-like workloads with multiple consumers. ([PostgreSQL][2])

For parking reservation:

```text
User A → locks inventory
User B → SKIP LOCKED
```

B might incorrectly conclude:

```text
"No availability"
```

when the resource is actually available but temporarily locked.

For a reservation system, ordinary:

```text
SELECT ... FOR UPDATE
```

is usually more appropriate.

---

# 6. Optimistic Locking

Optimistic locking takes a different approach.

Suppose:

```text
ParkingInventory
----------------
station_id
capacity
reserved_count
version
```

Initially:

```text
reserved_count = 99
version = 10
```

Both requests read version 10.

User A executes:

```sql
UPDATE parking_inventory
SET reserved_count = reserved_count + 1,
    version = version + 1
WHERE station_id = 101
AND version = 10
AND reserved_count < capacity;
```

A succeeds:

```text
reserved_count = 100
version = 11
```

User B then executes the same operation expecting:

```text
version = 10
```

but the row is now version 11.

Therefore:

```text
UPDATE affected rows = 0
```

B knows it lost the race.

### Optimistic locking is good when:

* contention is relatively low
* conflicts are relatively rare
* you don't want transactions waiting on locks
* retrying is acceptable

### Pessimistic locking is attractive when:

* contention is expected
* the resource is scarce
* conflicts are expensive
* you want serialization around a critical resource

For our **last parking space** scenario, pessimistic locking is a very reasonable choice.

---

# 7. But Don't Say "Pessimistic Is Always Better"

This is the one part of your original answer I'd change.

You said:

> "Reservation is important, so we should use pessimistic locking."

Importance alone doesn't determine the strategy.

The better reasoning is:

> "Reservation has a strong correctness requirement and potentially high contention on scarce inventory, so pessimistic locking on the inventory row is a reasonable choice. For lower-contention resources, optimistic concurrency may provide better throughput."

That's a **senior-level explanation**.

---

# 8. Database Unique Constraints — Extremely Important

Locks aren't your only protection.

Suppose your business rule is:

> A user can have at most one active reservation for a station/date.

The database can enforce that.

For example, conceptually:

```text
UNIQUE(
    user_id,
    station_id,
    parking_date
)
```

possibly with a **partial unique index** if only active reservations count.

This gives you defense at the database level even if application code has a bug.

PostgreSQL supports unique indexes/constraints as part of conflict handling such as `ON CONFLICT`. ([PostgreSQL][3])

So I like this architecture:

```text
Application logic
      +
Transaction
      +
Lock / optimistic check
      +
DB constraint
```

Not:

```text
Application if-check only ❌
```

---

# 9. Do We Need Distributed Locking?

This is where I want you to adjust your thinking.

You said:

> "For distributed systems we can use distributed locking."

Technically yes.

But:

> **A distributed system does not automatically require a distributed lock.**

Suppose we have:

```text
Reservation Service
   ├── Instance 1
   ├── Instance 2
   ├── Instance 3
   └── Instance 4
          ↓
       PostgreSQL
```

All four instances can safely use the **same database transaction and row lock**.

The database coordinates them.

You don't need:

```text
Node 1
   ↓
Redis lock
   ↓
Postgres
```

just because there are multiple application servers.

---

# 10. Redis Distributed Lock

Redis locks can be useful for some distributed coordination problems.

For example:

```text
Scheduler Instance A
Scheduler Instance B
Scheduler Instance C
        ↓
     Redis lock
        ↓
Only one runs the job
```

That's a reasonable use case.

But for **financial/reservation correctness**, I would not make Redis the only authority when PostgreSQL can enforce the invariant transactionally.

Your instinct here is good:

> Don't replace a durable database constraint/transaction with a fragile application-level lock unnecessarily.

---

# 11. What if we have multiple databases?

Now distributed locking becomes more interesting.

Suppose:

```text
Reservation Service
      ↓
Reservation DB

Inventory Service
      ↓
Inventory DB
```

and you need one atomic operation across both databases.

Now you have a distributed transaction problem.

Possible approaches include:

* redesigning ownership so one DB owns the invariant
* Saga
* event-driven coordination
* distributed locks where appropriate
* serializable/transactional mechanisms within one authoritative store

Usually the **best answer isn't immediately "add Redis lock."**

First ask:

> **Can I redesign the ownership so the critical invariant lives in one authoritative database?**

That's often much cleaner.

---

# 12. Our Recommended Reservation Concurrency Design

For this case study, I'd recommend:

```text
                 POST /reservations
                         │
                         ▼
                Reservation Service
                         │
                         ▼
                BEGIN TRANSACTION
                         │
                         ▼
              Lock inventory row
                SELECT FOR UPDATE
                         │
                         ▼
             Check reserved < capacity
                    /          \
                  NO            YES
                  │              │
                  ▼              ▼
             Rollback       Create PENDING_PAYMENT
                               reservation
                                  │
                                  ▼
                              COMMIT
```

Then payment happens **outside that short DB transaction**.

That's important.

Don't hold the PostgreSQL row lock while calling:

```text
Payment Provider
```

because payment could take seconds.

Bad:

```text
BEGIN
 ↓
LOCK ROW
 ↓
CALL PAYMENT PROVIDER
 ↓
wait 2 seconds
 ↓
COMMIT
```

That creates unnecessary lock contention.

Instead:

```text
BEGIN
 ↓
LOCK inventory
 ↓
reserve/hold capacity
 ↓
CREATE PENDING_PAYMENT
 ↓
COMMIT
 ↓
CALL PAYMENT
```

Then:

```text
Payment SUCCESS
       ↓
Reservation CONFIRMED
```

or:

```text
Payment FAILED
       ↓
Release reservation hold
```

This is a **very important production design point**.

---

# 13. Now Idempotency

Let's move to the second pillar.

Suppose:

```text
POST /reservations
Idempotency-Key: abc-123
```

The client sends it.

Server creates:

```text
Reservation R100
```

but response gets lost.

Client retries:

```text
POST /reservations
Idempotency-Key: abc-123
```

We should return:

```text
R100
```

not create:

```text
R101
```

AWS explicitly recommends idempotency tokens for mutating APIs so retries don't create duplicate records or side effects. ([AWS Documentation][4])

---

# 14. How to Implement Idempotency

A simple production model:

```text
IdempotencyRecord
-----------------
idempotency_key
user_id
operation
request_hash
status
resource_id
response
created_at
expires_at
```

And unique constraint:

```text
UNIQUE(user_id, operation, idempotency_key)
```

Flow:

```text
Request
   ↓
Check idempotency key
   ↓
Already exists?
  /       \
YES       NO
 |         |
Return     Create idempotency record
existing   + perform operation
result
```

But there's a critical detail:

### The idempotency record and business operation must be coordinated atomically where appropriate.

AWS's Builders' Library specifically calls out the importance of making the idempotency token recording and the relevant mutating operation ACID-consistent, so you don't record the token but fail to create the resource—or create the resource but fail to record the token. ([Amazon Web Services, Inc.][5])

---

# 15. Idempotency ≠ Unique Business Constraint

This is another important distinction.

Suppose:

```text
User A
POST /reservations
Idempotency-Key: X
```

That protects against:

```text
same request repeated
```

But what if:

```text
User A → Idempotency-Key X
User B → Idempotency-Key Y
```

both try to reserve the last spot?

Idempotency doesn't help.

That's why we need:

```text
Idempotency
       +
Concurrency control
       +
Business constraints
```

---

# 16. Which APIs Need Idempotency?

For our system:

| API                             | Idempotency        |
| ------------------------------- | ------------------ |
| `GET /availability`             | Naturally safe     |
| `GET /reservations/:id`         | Naturally safe     |
| `POST /reservations`            | **Yes**            |
| `POST /payments`                | **Absolutely yes** |
| `POST /reservations/:id/cancel` | Yes                |
| `POST /payments/:id/refunds`    | **Absolutely yes** |
| Notification event consumer     | **Yes**            |

Payment and refund are especially important because duplicate execution can have direct financial consequences.

---

# 17. Idempotency at the Payment Provider Boundary

There's another layer.

Our system:

```text
Reservation Service
       ↓
Payment Service
       ↓
External Payment Provider
```

You need idempotency **at both levels**.

For example:

```text
Our API
Idempotency-Key = RES-123
```

Payment Service might generate/use:

```text
Provider idempotency key = PAY-456
```

and send that to the payment provider if the provider supports idempotent requests.

Otherwise:

```text
Payment Service
    ↓
₹500 charge
    ↓
timeout
    ↓
retry
    ↓
₹500 charge AGAIN ❌
```

That's exactly the kind of failure we were discussing earlier.

---

# 18. Don't Say "Exactly Once"

Be careful in interviews.

You'll sometimes hear:

> "Idempotency guarantees exactly-once processing."

That's not quite the right mental model.

A better statement is:

> **Idempotency makes repeated execution of the same logical request produce the same business result without creating additional side effects.**

Distributed systems can still have retries, duplicate messages, crashes, and unknown outcomes.

So we build systems that are **safe under retries and duplicates**.

---

# 19. Production Mental Model

I want you to remember this:

```text
                    CONCURRENCY
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   DB transaction    Row locking     DB constraints
        │                │                │
        └────────────────┼────────────────┘
                         │
                  Protect invariant
                         │
                    "one spot"


                    IDEMPOTENCY
                         │
        ┌────────────────┼────────────────┐
        │                │                │
 Idempotency key    Unique record    Stored result
        │                │                │
        └────────────────┼────────────────┘
                         │
                  Protect retries
                         │
              "same request ≠ duplicate"
```

---

# 20. What I'd Say in Your Interview

If the interviewer asks:

> **"How do you prevent two users from booking the same parking space?"**

Don't give them ten patterns.

Give them a clear answer:

> **"The database will be the source of truth for reservation capacity. During reservation creation, I'll use a short database transaction and pessimistic row-level locking on the inventory/capacity record. I'll verify that capacity is still available and create a PENDING_PAYMENT reservation while incrementing/holding the inventory atomically. I'll also use database constraints as a final correctness guard. I would not hold the DB lock while calling the payment provider. Redis can be used to accelerate availability reads, but it won't be the source of truth. A distributed lock is only necessary if the invariant cannot be safely owned and enforced by a single authoritative datastore."**

That's a **strong senior-level answer**.

If they then ask:

> **"What about duplicate requests?"**

Answer:

> **"That's a separate idempotency problem. Mutating APIs such as reservation, payment, cancellation, and refund will accept an idempotency key. We'll persist the key with the operation/result under a unique constraint, so retries return the original result rather than creating another resource or financial side effect. At the payment-provider boundary, I'll also use the provider's idempotency mechanism where supported."** ([Amazon Web Services, Inc.][5])

---

## The key takeaway for our Parking case

Our final reservation architecture is becoming:

```text
                    Client
                       │
                       ▼
              Reservation API
                       │
                       ▼
                Idempotency
                       │
                       ▼
              DB Transaction
                       │
                SELECT FOR UPDATE
                       │
                       ▼
              Inventory / Capacity
                       │
                 available?
                  /       \
                NO         YES
                │           │
                ▼           ▼
             Reject    PENDING_PAYMENT
                            │
                         COMMIT
                            │
                            ▼
                      Payment Service
                       /          \
                  SUCCESS        FAILED
                     │              │
                     ▼              ▼
                 CONFIRMED      Release hold
```

Redis:

```text
GET availability → Redis
```

Kafka/Outbox:

```text
Reservation state change
        ↓
      Outbox
        ↓
      Kafka
        ↓
Notification / Reporting / other consumers
```

This is the model I want you to carry into future system-design problems.

---
This one is deliberately deep because we're using it to build a **reusable system-design way of thinking**, not just finish one architecture.

Next rough plan:

|  | Focus                                            |
| --- | ------------------------------------------------ |
| 1   | Database transaction + concurrency deep dive     |
| 2   | Consistency + cache strategy                     |
| 3   | Payment failures + reconciliation                |
| 4   | Saga + Outbox + Kafka in this system             |
| 5   | Scaling, bottlenecks & high availability         |
| 6   | Security + rate limiting + reliability           |
| 7   | Observability + DR + disaster scenarios          |
| 8   | Final technology-agnostic architecture           |
| 9   | **AWS mapping** — services + alternatives        |
| 10  | AWS failure/scaling/trade-off discussion         |
| 11  | Full end-to-end interview walkthrough            |
| 12  | Mock senior system-design interview / refinement |

---

**Next step after this:** we should go deeper into the **actual database transaction and reservation concurrency flow**, including what happens with 2, 10, or 100 concurrent requests for the last few spaces. After that we'll cover **consistency, cache strategy, and failure/reconciliation scenarios**, and only then finalize the technology-agnostic architecture before mapping it to AWS.

[1]: https://www.postgresql.org/docs/17/explicit-locking.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 17: 13.3. Explicit Locking"
[2]: https://www.postgresql.org/docs/10/sql-select.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 10: SELECT"
[3]: https://www.postgresql.org/docs/19/sql-insert.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: INSERT"
[4]: https://docs.aws.amazon.com/wellarchitected/2024-06-27/framework/rel_prevent_interaction_failure_idempotent.html?utm_source=chatgpt.com "REL04-BP04 Make all responses idempotent - AWS Well-Architected Framework"
[5]: https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/?utm_source=chatgpt.com "references-details-empty"
