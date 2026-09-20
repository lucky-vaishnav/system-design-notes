## Database Transaction + Concurrency Flow

**Parking Reservation System — simultaneous reservation requests**

We’ll focus on the most important correctness problem:

> **What happens when multiple users try to reserve the last available parking capacity at the same time?**

Today we’ll cover:

1. Inventory/capacity data model
2. Transaction boundary
3. `SELECT ... FOR UPDATE`
4. Two users racing for the last slot
5. `PENDING_PAYMENT` hold
6. Releasing capacity after payment failure/cancellation
7. Optimistic locking alternative
8. Unique constraints as a safety net
9. Lock contention, deadlocks, and timeouts
10. Why Redis/distributed locking isn't our primary authority
11. Serializable isolation — when it matters

### First, I want you to design it

Don't worry about being perfect.

Assume:

* Parking lot has **100 spaces**
* Currently **99 are reserved**
* User A and User B simultaneously call:

`POST /reservations`

Both want the same parking lot/date/time.

**Question:** How would you design the database transaction so that **only one user can successfully hold the last available space**, while also making sure we **never keep a DB lock open while waiting for payment**?

Give me your flow in your own words first. I'll review it like a senior system-design interviewer, then we'll go deeper into the actual transaction/SQL design.

#### My Thoughts That Time :
 So what we will do is that we will first turn section, you have the inventory table, and this section will be updating. The new session will, will be updated within that transaction, and then we will is the log by updating the inventory space, and then we require the status, and after payment service, successfully. Update the inventory table again. So since how the flat table has using the transaction. But there is one question from my side, is that if there is the log and the all print, scan, is used that, the parsing is good, but the only question is created by the status called payment pending. And payment is still in the process. So if we update, will be, and then I delete the log, content request will be assumed, status code, this is good, so if this is the main, the final we can do it. And if we do not update the inventory table and raising, and then after we just for payment service, then the pending request or concurrent request will be assumed that space is not. available to clear. So it will also. So according to we should not delete the log by payment service is called or if it is in the log, should update inventory table as per. So yeah, this is my input for yes point, later the transaction plus part I will. Say I am just review it. Yes. Give me first, for example, then we can move to the next topic. 

Yes — your approach is moving in the right direction. The key thing you have identified is **the difference between holding inventory during `PENDING_PAYMENT` and releasing it after the payment outcome**.

Let me review your approach first, without jumping into the full SQL/transaction deep dive yet.

### Your proposed flow

You are essentially thinking:

```text
Reservation Request
       ↓
Inventory check
       ↓
Lock inventory
       ↓
Reduce/hold available capacity
       ↓
Create Reservation = PENDING_PAYMENT
       ↓
Commit transaction
       ↓
Call Payment Service
       ↓
       ├── SUCCESS
       │      ↓
       │   PENDING_PAYMENT → CONFIRMED
       │
       └── FAILURE
              ↓
          release inventory
          PENDING_PAYMENT → PAYMENT_FAILED/CANCELLED
```

That is the **right overall direction**.

### One important correction

You said something like:

> After payment service is called, should we update the inventory again?

**Normally, no for successful payment.**

The inventory was already held when we created the `PENDING_PAYMENT` reservation.

For example:

```text
Total capacity = 100
Available = 1
```

User A starts reservation:

```text
Available: 1 → 0
Reservation: PENDING_PAYMENT
```

Now another user B comes in:

```text
Available = 0
```

So B cannot take that space while A's payment is pending.

If A's payment succeeds:

```text
PENDING_PAYMENT → CONFIRMED
```

**Inventory stays at 0.**

We don't decrement it again, otherwise we'd effectively reserve the same space twice.

---

### What happens when payment fails?

This is where you correctly identified that we need to update inventory.

```text
Available = 0
Reservation = PENDING_PAYMENT
```

Payment fails:

```text
Reservation:
PENDING_PAYMENT → PAYMENT_FAILED

Inventory:
reserved_count - 1
available_count + 1
```

Now another request can use that space.

So think of the inventory operation as:

| Event                           | Inventory                              |
| ------------------------------- | -------------------------------------- |
| Create `PENDING_PAYMENT`        | **Hold/decrease availability**         |
| Payment succeeds                | **No inventory change**                |
| Payment fails                   | **Release/increase availability**      |
| Payment becomes UNKNOWN         | **Keep held temporarily**              |
| Reservation expires             | **Release availability**               |
| Cancellation after confirmation | **Release according to business rule** |

### Your concern about concurrent requests is exactly correct

Suppose:

```text
Available = 1
```

A and B arrive simultaneously.

A:

```text
lock inventory
available = 1
→ hold it
available = 0
→ PENDING_PAYMENT
commit
```

B then gets the lock:

```text
available = 0
→ cannot create PENDING_PAYMENT
→ reservation rejected
```

That's why we **must create the hold before payment**.

If instead we did:

```text
check availability
      ↓
call payment
      ↓
create reservation
```

then both A and B could potentially see the same last space and both proceed to payment. That's a serious design problem.

---

### One more important point: don't delete the reservation

You mentioned deleting the request/log after payment.

For the actual **reservation record**, I would **not delete it**.

Keep:

```text
reservation_id
status = PAYMENT_FAILED
```

or:

```text
status = CANCELLED
```

This gives us:

* audit history
* troubleshooting
* payment reconciliation
* customer history
* financial traceability

The **inventory hold** is what gets released, not the business record itself.

So the mental model should be:

> **Reservation state changes; reservation record remains. Inventory is held or released based on that state.**

---

### And your `PENDING_PAYMENT` concern leads directly to an important production issue

What if:

```text
A → PENDING_PAYMENT
A → payment request sent
Payment provider → takes 30 seconds
```

During those 30 seconds, the space is held.

We therefore need a mechanism such as:

```text
PENDING_PAYMENT
      ↓
payment timeout / reservation expiry
      ↓
release inventory
```

But there is an important complication:

**What if the payment actually succeeds after we have expired the reservation?**

That's where **payment reconciliation and an UNKNOWN state** become important.

That's the next level of this discussion, and it's one of the most important production scenarios.

### So your current understanding

I'd summarize your approach as:

> **Create a short database transaction that atomically locks the inventory record, verifies availability, holds the required capacity, and creates a `PENDING_PAYMENT` reservation. Commit before calling the payment service. A successful payment only changes the reservation status to `CONFIRMED`; it does not consume inventory again. A failed or expired reservation releases the previously held inventory. The reservation record is retained for auditability rather than deleted.**

That is a solid foundation.


**Next deep dive:** I'll take this exact flow and show what happens with **2, 10, and 100 concurrent requests**, including the actual transaction boundaries, row locks, waiting behavior, lock timeout/deadlock scenarios, and then the optimistic-locking alternative.

Absolutely. Let’s continue from that exact point.

# Concurrent Booking: 2, 10, and 100 Requests

We'll use the same simplified booking scenario:

> A parking spot has **1 available slot**, and multiple users try to book it at nearly the same time.

Assume PostgreSQL and a transaction using row-level locking:

```sql
BEGIN;

SELECT available_slots
FROM parking_inventory
WHERE parking_id = 101
FOR UPDATE;

-- check available_slots > 0

UPDATE parking_inventory
SET available_slots = available_slots - 1
WHERE parking_id = 101;

INSERT INTO bookings (...);

COMMIT;
```

The important part is:

```sql
SELECT ... FOR UPDATE
```

This is where concurrency control happens.

---

## 1. Two concurrent requests

Suppose:

```text
Initial available_slots = 1
```

Two users arrive at almost exactly the same time:

```text
Request A → Book parking
Request B → Book parking
```

### What actually happens

```text
                    PostgreSQL
                       |
              parking_inventory
                 parking_id=101
                 available=1
                       |
              ┌────────┴────────┐
              │                 │
          Request A         Request B
          BEGIN             BEGIN
             │                 │
        SELECT ...             │
        FOR UPDATE             │
             │                 │
        🔒 Row locked           │
             │              SELECT ...
             │              FOR UPDATE
             │                 │
             │             WAITING...
             │                 │
        available = 1           │
             │
        UPDATE → 0              │
             │
        INSERT booking          │
             │
          COMMIT                │
             │
        🔓 lock released        │
                               │
                          lock acquired
                               │
                          available = 0
                               │
                         no availability
                               │
                            ROLLBACK
```

### Important point

Request B **does not read `available_slots = 1` while A is holding the lock**.

It waits for A.

After A commits, B gets the lock and sees:

```text
available_slots = 0
```

So B returns something like:

```http
409 Conflict
```

or:

```json
{
  "error": "PARKING_UNAVAILABLE"
}
```

### Result

```text
Request A → SUCCESS
Request B → NO AVAILABILITY
```

And importantly:

```text
Successful bookings = 1
```

No double booking.

---

# 2. What if B starts first?

The exact same principle applies.

```text
B → SELECT FOR UPDATE → gets lock
A → SELECT FOR UPDATE → waits

B → checks availability
B → update
B → insert booking
B → commit

A → gets lock
A → sees 0
A → rollback
```

So the result doesn't depend on which request arrived at your Node.js server first.

**The database lock determines which transaction gets access first.**

That's an important senior-level point.

---

# 3. What happens inside Node.js?

This is also important because people sometimes misunderstand the role of Node.js.

Suppose we have:

```text
Request A
Request B
Request C
```

Node.js can have all three requests executing concurrently from the application's perspective.

For example:

```js
await db.query('SELECT ... FOR UPDATE');
```

When Request A reaches the database:

```text
Node.js
   │
   ├── Request A
   │       │
   │       └── DB → lock acquired
   │
   ├── Request B
   │       │
   │       └── DB → waiting
   │
   └── Request C
           │
           └── DB → waiting
```

Node.js isn't blocking its entire event loop while B waits.

The database is handling the lock wait.

The Node.js process can continue handling other requests/events.

So:

> **Node.js provides application-level concurrency, while PostgreSQL provides the database-level serialization needed for this critical section.**

---

# 4. Now 10 concurrent requests

Assume:

```text
available_slots = 1
```

Ten users simultaneously attempt:

```text
R1
R2
R3
...
R10
```

Conceptually:

```text
R1 ──┐
R2 ──┤
R3 ──┤
R4 ──┤
R5 ──┤
R6 ──┤──→ PostgreSQL
R7 ──┤
R8 ──┤
R9 ──┤
R10 ─┘
```

One request acquires the row lock.

Let's say R4 gets it:

```text
R4 → 🔒 LOCK
```

Everyone else waits:

```text
R1 → WAIT
R2 → WAIT
R3 → WAIT
R5 → WAIT
...
R10 → WAIT
```

R4 performs:

```text
available 1 → 0
INSERT booking
COMMIT
```

Then the lock is released.

One of the waiting transactions gets the lock.

It sees:

```text
available = 0
```

It doesn't create a booking.

It rolls back.

The remaining transactions then proceed one by one, see `0`, and roll back.

### Final result

```text
R4 → SUCCESS
R1 → unavailable
R2 → unavailable
R3 → unavailable
...
R10 → unavailable
```

Only:

```text
1 booking
```

is created.

---

# 5. But there's an important performance problem

The database has protected correctness.

But now imagine:

```text
10,000 requests
        ↓
same parking inventory row
        ↓
FOR UPDATE
```

You get:

```text
             🔒
              │
R1 ───────────┤
R2 ───────────┤
R3 ───────────┤
R4 ───────────┤
R5 ───────────┤
...           │
R10000 ───────┘
```

This is called **lock contention**.

The database is correct, but the row becomes a bottleneck.

That's an important distinction:

> **Concurrency control solves correctness, but it doesn't automatically solve scalability.**

---

# 6. What about 100 concurrent requests?

Now let's make the example more interesting.

Suppose:

```text
Available slots = 50
```

and:

```text
100 concurrent booking requests
```

Then:

```text
100 requests
      ↓
same inventory row
      ↓
FOR UPDATE
```

Conceptually:

```text
R1  → 🔒
R2  → WAIT
R3  → WAIT
R4  → WAIT
...
R100 → WAIT
```

R1 completes:

```text
50 → 49
COMMIT
```

Then another waiting request gets the lock:

```text
49 → 48
COMMIT
```

This continues:

```text
50
 ↓
49
 ↓
48
 ↓
...
 ↓
1
 ↓
0
```

The first 50 successful transactions create bookings.

The remaining 50 eventually see:

```text
available_slots = 0
```

and fail.

Final state:

```text
Initial slots       = 50
Successful bookings = 50
Failed bookings     = 50
Remaining slots     = 0
```

Correctness is maintained.

---

# 7. Why does the transaction boundary matter?

Consider this:

```sql
BEGIN;

SELECT available_slots
FROM parking_inventory
WHERE parking_id = 101
FOR UPDATE;

UPDATE parking_inventory
SET available_slots = available_slots - 1
WHERE parking_id = 101;

INSERT INTO bookings (...);

COMMIT;
```

The lock is held approximately from:

```text
SELECT FOR UPDATE
       ↓
     UPDATE
       ↓
     INSERT
       ↓
    COMMIT
```

So the critical section should be **as small as reasonably possible**.

You don't want:

```text
BEGIN

lock row

call payment provider
   ↓
wait 2 seconds
   ↓
payment provider responds
   ↓
do something else
   ↓
COMMIT
```

That would mean the database row remains locked for seconds.

Under high concurrency, this can become painful.

---

# 8. What should NOT happen inside the transaction?

For example:

```text
BEGIN
 ↓
lock inventory
 ↓
call Stripe/CyberSource/payment provider
 ↓
wait for network
 ↓
send email
 ↓
call Uber
 ↓
update database
 ↓
COMMIT
```

This is generally a bad design.

The transaction should focus on the database consistency boundary.

External network calls should generally be outside that critical transaction where possible.

This leads us to the more advanced booking/payment architecture we discussed earlier:

```text
Reservation
     ↓
Inventory transaction
     ↓
Reservation = PENDING
     ↓
COMMIT
     ↓
Payment processing
     ↓
Payment success
     ↓
Reservation = CONFIRMED
```

We'll come back to this because it introduces **state machines, idempotency, and failure recovery**.

---

# 9. What happens if the transaction takes too long?

Suppose:

```text
R1 → acquires lock
```

but then something goes wrong:

```text
R1
 ↓
BEGIN
 ↓
FOR UPDATE
 ↓
lock acquired
 ↓
application hangs
```

Now:

```text
R2 → WAIT
R3 → WAIT
R4 → WAIT
...
```

This is why we need **timeouts**.

There are several different timeout concepts.

### Application/request timeout

For example:

```text
HTTP request timeout = 5 seconds
```

But this alone isn't enough.

The DB transaction could potentially remain open if the application doesn't properly clean it up.

### Database lock timeout

PostgreSQL supports:

```sql
SET lock_timeout = '2s';
```

Meaning:

> Don't wait indefinitely to acquire a lock.

If the lock isn't acquired within 2 seconds, PostgreSQL returns an error.

---

# 10. Lock timeout scenario

Suppose:

```text
R1 → holds lock
```

and:

```text
R2 → waiting
```

If:

```text
lock_timeout = 2 seconds
```

then:

```text
R2
 ↓
WAIT
 ↓
WAIT
 ↓
2 seconds
 ↓
LOCK TIMEOUT
```

R2 gets an error instead of waiting indefinitely.

Your API can translate that into an appropriate response, depending on your design:

```text
Retry
or
Temporary unavailable
```

The exact HTTP response should be based on the API contract rather than blindly mapping every database timeout to 409.

---

# 11. Deadlocks are different

This is a very important distinction.

### Lock contention

```text
A → waiting for B
```

Eventually B releases the lock.

That's normal contention.

### Deadlock

```text
A → holds Row 1
A → waits for Row 2

B → holds Row 2
B → waits for Row 1
```

Now:

```text
A ──holds──→ Row 1
│
└──waits──→ Row 2
             ↑
             │
          holds B
             │
             └──── waits for Row 1
```

Neither can proceed.

PostgreSQL detects the cycle and aborts one transaction.

For example:

```text
Transaction A → aborted
Transaction B → continues
```

The application receives a deadlock error for the aborted transaction.

---

# 12. How do we prevent deadlocks?

One major strategy:

## Always acquire locks in the same order.

Bad:

```text
Transaction A:
lock parking 1
lock parking 2

Transaction B:
lock parking 2
lock parking 1
```

Potential deadlock.

Better:

```text
Transaction A:
lock parking 1
lock parking 2

Transaction B:
lock parking 1
lock parking 2
```

Both follow the same ordering.

For example, if booking multiple resources:

```text
ORDER BY parking_id
```

and acquire locks consistently in that order.

This is a very useful senior interview point.

---

# 13. Now the optimistic-locking alternative

So far we've used:

```sql
SELECT ... FOR UPDATE
```

That's **pessimistic locking**.

We're essentially saying:

> "I'm going to modify this resource, so I'll lock it before doing the work."

Another approach is **optimistic concurrency control**.

Instead of locking first, we use a version number.

Example:

```text
parking_id = 101
available_slots = 1
version = 7
```

Request A reads:

```text
available = 1
version = 7
```

Request B also reads:

```text
available = 1
version = 7
```

Both think they can attempt the booking.

But then we perform an atomic conditional update:

```sql
UPDATE parking_inventory
SET available_slots = available_slots - 1,
    version = version + 1
WHERE parking_id = 101
  AND available_slots > 0
  AND version = 7;
```

Now suppose A executes first.

```text
version 7 → 8
available 1 → 0
```

A gets:

```text
rows_affected = 1
```

Success.

B then executes:

```text
WHERE version = 7
```

But the row is now:

```text
version = 8
```

Therefore:

```text
rows_affected = 0
```

B knows someone changed the record.

So:

```text
A → SUCCESS
B → CONFLICT / RETRY / UNAVAILABLE
```

---

# 14. Pessimistic vs Optimistic

|                   | Pessimistic                         | Optimistic                                   |
| ----------------- | ----------------------------------- | -------------------------------------------- |
| Example           | `FOR UPDATE`                        | version column                               |
| Lock acquired     | Before modification                 | No explicit application lock                 |
| Waiting           | Possible                            | Usually less waiting                         |
| Conflict handling | Database serializes                 | Application detects conflict                 |
| Good when         | High contention / critical resource | Lower contention                             |
| Main concern      | Lock contention                     | Retry/conflict handling                      |
| Deadlocks         | Possible                            | Much less relevant for this pattern          |
| Complexity        | Relatively straightforward          | Requires careful conditional updates/retries |

Neither is universally "better."

The choice depends on:

* contention
* transaction duration
* retry cost
* correctness requirements
* workload
* database behavior

---

# 15. One subtle but important point

For our parking inventory case, we don't necessarily need a separate `version` column.

We can often use an atomic conditional update:

```sql
UPDATE parking_inventory
SET available_slots = available_slots - 1
WHERE parking_id = 101
  AND available_slots > 0;
```

Then:

```text
rows_affected = 1
```

means:

```text
booking capacity successfully reserved
```

while:

```text
rows_affected = 0
```

means:

```text
no capacity available
```

This can be extremely efficient.

However, if we need to detect **which specific representation/version changed** or protect more complex state transitions, explicit optimistic versioning can be useful.

---

# 16. Senior-level mental model

For the interview, I would remember the whole thing like this:

```text
                 CONCURRENT BOOKINGS
                         │
             ┌───────────┴───────────┐
             │                       │
       Pessimistic               Optimistic
         locking                  locking
             │                       │
       SELECT FOR UPDATE        version / CAS
             │                       │
       serialize access         detect conflict
             │                       │
       possible waiting          possible retry
             │
       possible deadlock
```

And the most important distinction:

```text
Correctness
    ≠
Scalability
```

`FOR UPDATE` can guarantee that two transactions don't incorrectly consume the same inventory, but if **100,000 users contend for the same row**, that row can still become a bottleneck.

---

---

### Question:

If we commit the transaction before calling the payment service, the inventory will already be held for the `PENDING_PAYMENT` reservation. This means concurrent requests may see the parking as unavailable while the payment is still being processed.

If the payment later fails, we release the held inventory and make the parking available again.

Is this the correct production-level approach? In other words, is it acceptable to temporarily hold a parking space during `PENDING_PAYMENT`, causing concurrent requests to see no availability, or should we keep the space available until the payment is successfully completed?


This is a very good production-level question because it gets into the **reservation hold vs actual payment confirmation** trade-off.

---
Yes — your understanding is correct, and this is exactly the kind of trade-off we should discuss at senior level.

## Your question: Should `PENDING_PAYMENT` hold inventory?

**Yes. In a typical parking-reservation system, it is acceptable—and usually necessary—to temporarily hold the inventory while payment is being processed.**

The important distinction is:

> **Availability is not the same thing as confirmed booking.**

A parking space can be:

```text
AVAILABLE
   ↓
HELD / PENDING_PAYMENT
   ↓
CONFIRMED
```

While it is `PENDING_PAYMENT`, other users should generally **not be allowed to reserve the same inventory**.

### Why?

Imagine we don't hold it:

```text
10:00:00  User A → selects last parking space
10:00:01  User A → calls payment service

10:00:01  User B → sees same space AVAILABLE
10:00:02  User B → successfully reserves it
```

Now A's payment may succeed, but the inventory has already been given to B.

You have created a much harder problem.

---

# The production approach

A common flow is:

```text
                    DB Transaction
                         │
                         ▼
              Lock/check inventory
                         │
                         ▼
              Create reservation
              status=PENDING_PAYMENT
                         │
                         ▼
              Hold inventory
                         │
                       COMMIT
                         │
                         ▼
                  Payment Service
                    /          \
               SUCCESS         FAIL
                  │               │
                  ▼               ▼
             CONFIRMED       RELEASE HOLD
```

So yes:

```text
PENDING_PAYMENT
      ↓
inventory is temporarily unavailable
```

That is intentional.

---

## But there is one critical requirement

You **must not hold it indefinitely**.

For example:

```text
PENDING_PAYMENT
       │
       │ 10 minutes
       ▼
EXPIRED
       │
       ▼
release inventory
```

So your reservation could have:

```text
reservation_id
parking_id
user_id
status
expires_at
```

For example:

```text
status       = PENDING_PAYMENT
expires_at   = 10:15:00
```

If payment isn't completed by the expiry time, the reservation can be released.

---

# What should the availability API show?

This is where the domain semantics become important.

Suppose:

```text
Total spaces = 100

AVAILABLE       = 70
PENDING_PAYMENT = 5
CONFIRMED       = 25
```

For a new user:

```text
available = 70
```

not:

```text
available = 75
```

because the five pending spaces are already temporarily committed to other users.

So:

```text
Available inventory
=
Total
-
Active holds
-
Confirmed reservations
```

subject to your exact parking/time-slot model.

---

# Could we keep it available until payment succeeds?

Technically yes, but then you need a completely different concurrency model.

You would essentially be saying:

```text
Payment starts
       ↓
space still AVAILABLE
       ↓
another user can book it
       ↓
payment succeeds
       ↓
CONFLICT
```

Now you need to deal with:

* payment succeeded but inventory is gone
* refunds/voids
* compensating transactions
* confusing user experience
* race conditions between payment completion and booking
* more complicated recovery logic

For a reservation system, that is generally a much harder consistency problem.

So the simpler model is:

> **Reserve/hold first, then pay.**

---

# But there's another subtle issue

We should distinguish between:

### Inventory hold

```text
PENDING_PAYMENT
```

and:

### Final booking

```text
CONFIRMED
```

They are **not the same state**.

That's why I would model it as a state machine:

```text
                 ┌──────────────┐
                 │   AVAILABLE  │
                 └──────┬───────┘
                        │
                   create hold
                        │
                        ▼
              ┌──────────────────┐
              │ PENDING_PAYMENT  │
              └───────┬──────────┘
                      │
             ┌────────┴────────┐
             │                 │
        payment success    payment fails
             │                 │
             ▼                 ▼
       ┌───────────┐       ┌─────────┐
       │ CONFIRMED │       │ EXPIRED │
       └───────────┘       └─────────┘
```

And this gives us a very clean separation:

**Hold = temporary inventory reservation**

**Confirmed = successful booking/payment**

---

# One thing I would change in your wording

Instead of saying:

> "concurrent requests may see the parking as unavailable while payment is still being processed"

I'd phrase the design principle as:

> **Once a reservation enters `PENDING_PAYMENT`, its inventory is intentionally excluded from availability for other users until payment succeeds, fails, or the hold expires.**

That is more precise for your notes.

---
