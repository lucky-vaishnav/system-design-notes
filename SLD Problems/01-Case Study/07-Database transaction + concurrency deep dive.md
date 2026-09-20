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

---

### Question:

If we commit the transaction before calling the payment service, the inventory will already be held for the `PENDING_PAYMENT` reservation. This means concurrent requests may see the parking as unavailable while the payment is still being processed.

If the payment later fails, we release the held inventory and make the parking available again.

Is this the correct production-level approach? In other words, is it acceptable to temporarily hold a parking space during `PENDING_PAYMENT`, causing concurrent requests to see no availability, or should we keep the space available until the payment is successfully completed?


This is a very good production-level question because it gets into the **reservation hold vs actual payment confirmation** trade-off.


---

**Next deep dive:** I'll take this exact flow and show what happens with **2, 10, and 100 concurrent requests**, including the actual transaction boundaries, row locks, waiting behavior, lock timeout/deadlock scenarios, and then the optimistic-locking alternative.




