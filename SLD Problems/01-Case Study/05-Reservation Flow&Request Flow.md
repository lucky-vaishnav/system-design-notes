### My Thoughts That Time-
So for our system design problem, so the next step is to about the reservation flow and the request flow. So I will start answering the questions which you asked. There's ten questions. So first question, first question answer will be that servicees called will be parking service for availability check, reservation service for reservation flow, and payment service will be called as well from reservation service mostly, and that will be a synchronous call.  
And for the second question, so the synchronous call will be parking availability check, reservation APIs, payment API calls, refund API calls. So these will be the synchronous communication and will be required.  
And for the question number three, the asynchronous communication calls will be for mostly majorly at higher level will be the notification service communication.  
Yeah. So when we will check the availability, so for the question number four. So when the user searches for the station for parking, that time we will show the parking availability, and also at the time of the reservation request from the user. So at code level, backend side also, code level, backend side also, we will check the availability again, so that the flow will be more robust.  
So for the question number five, so when will be the actual reservation will be created? So the actual reservation on payment success will be created, and there will be one reservation request also will be created temporarily, so that the station availability will be properly calculated while the payment is happening. So yeah, so reservation will be only after the successful payment, and before that, for the idempotency and for the station availability checking logic, we will create a reservation request for the user.  
And for question number six, when do we call the payment service? So after creating reservation request for ID dependency and station availability logic, which will be using this reservation request info, then after we will send the payment request. So the actual reservation will be after the payment request is successful.  
And for question number seven, what will happen if payment fails? So if payment fails, reservation request. So according to me, this will be totally depend upon the business requirement. So I am assuming that until the payment success is not received by the reservation service, we will consider this reservation not successful, and we will be canceling this reservation, and we will be removing that reservation request as well. And if there are some payments which are actually done, so we can use some backend functionality to refund those type of transaction for the user.So and for question number eight, so the answer will be the similar like I said for the question number seven. So if the payment is succeeded but response is lost, we will consider it as a failure and reservation will be cancelled. And yeah, if there are some actual transaction happened for the user, the backend process or some admin actions can trigger the refund for those successful transactions. Or it may totally depend on the business requirement, what they expect.  
And for question number nine, when do we update invalid Redis? So we treat it as, so on the reservation request created by our system, just the request, not the reservation. The Redis will be updated and this will be invalidated because we will need an updated availability for the station. And also whenever there is a payment failure or some cancellation of the reservation by the user, that time also we will be updating or invalidating the Redis. We will invalidate Redis that time also for the user cancellation and payment failure scenario for the reservation request.  
And for question number 10, so Kafka and outbox comes into the flow for the notification communication, and for that outbox approach will give us the fail-safe approach, so that the notification will not be missed. And if the notification service is not working, then the reservation can use this outbox pattern, so that we can re-trigger those events again, so that this notification will not be missed. So yeah, so for mostly the notification, because there we will be needing asynchronous communication. So there we will be needing asynchronous communication.  

Good — this is a much more complete answer, and you are thinking about **failure scenarios**, not just the happy path. That's exactly what I want at this stage.

There are, however, **two important architectural corrections** I want you to understand before we move forward:

1. **Don't create the final reservation only after payment succeeds.** Create a reservation/hold in `PENDING_PAYMENT` state before charging.
2. **Never treat "payment response lost" as payment failure.** A timeout means the payment result is **unknown**, and we must reconcile it before deciding whether to cancel/refund.

Those two points are extremely important in senior system design.

---

# Your Question — Rewritten for Notes

**How should the parking reservation request flow handle availability checking, reservation creation, payment processing, Redis updates, asynchronous notifications, and failure scenarios such as payment failure or a lost payment response?**

---

# Review of Your Flow

## 1. Services — ✅

Your flow is reasonable:

```text
Client
   ↓
Reservation Service
   ↓
Parking / Availability
   ↓
Payment Service
```

However, I would avoid making the client independently call Parking Service and then Reservation Service as two required steps.

The client can do:

```text
GET /availability
```

for displaying availability.

But the actual reservation flow should be controlled by Reservation Service.

---

# 2. Synchronous communication — ✅ Mostly

These are reasonable synchronous operations:

```text
Availability query
Create reservation
Payment authorization/charge
Cancellation request
Refund request
```

But **notification should not be synchronous** with reservation completion.

Good.

---

# 3. Asynchronous — ✅ but Kafka is not only for notifications

You correctly identified:

```text
Reservation
    ↓
Outbox
    ↓
Kafka
    ↓
Notification Service
```

But later we could have other consumers too:

```text
                  ┌→ Notification
Reservation Event ├→ Analytics
                  ├→ Reporting
                  └→ Audit
```

So Kafka is really our **event distribution mechanism**, while notification is one consumer.

---

# 4. Availability check — ✅ Very important

This part of your answer is correct.

We have:

### Search-time check

```text
GET /availability
        ↓
Redis / Read Model
        ↓
80 spaces available
```

This is primarily for **display**.

Then:

### Reservation-time check

```text
POST /reservations
        ↓
Authoritative availability check
        ↓
Reserve/hold capacity
```

This check is what protects correctness.

And here's the important point:

> **The second check cannot simply be another SELECT. It must participate in an atomic concurrency-control mechanism.**

We'll cover that next.

---

# 5. Reservation creation — ⚠️ Important correction

You said:

> Actual reservation is created only after payment succeeds.

I would change this.

We should create a **reservation/hold before payment**, but it should not be a confirmed reservation.

For example:

```text
Reservation
status = PENDING_PAYMENT
```

or conceptually:

```text
Reservation Hold
```

Flow:

```text
POST /reservations
        ↓
Check availability
        ↓
Create reservation
PENDING_PAYMENT
        ↓
Reduce/hold availability
        ↓
Call Payment
        ↓
SUCCESS
        ↓
CONFIRMED
```

Why?

Imagine capacity = 1.

```text
User A → availability = 1
User B → availability = 1
```

If neither creates a hold before payment:

```text
User A → Payment
User B → Payment

Both think they got the last space ❌
```

We need the reservation/hold to claim the capacity while payment is happening.

---

# 6. Payment — mostly correct

Your sequence should therefore become:

```text
Create reservation
PENDING_PAYMENT
       ↓
Call Payment
       ↓
SUCCESS
       ↓
CONFIRMED
```

And if payment takes too long:

```text
PENDING_PAYMENT
       ↓
expires after X minutes
       ↓
release hold
```

This introduces the concept of a **reservation/payment timeout**.

---

# 7. Payment failure — good thinking, but don't immediately delete

You said:

> cancel reservation request/remove it.

Conceptually correct, but I would **keep the record**.

Don't physically delete it.

Instead:

```text
Reservation
status = PAYMENT_FAILED
```

or:

```text
status = CANCELLED
reason = PAYMENT_FAILED
```

Why?

Because we need an audit trail.

For example:

```text
Reservation #123
User: 456
Created: 10:00
Payment: FAILED
Cancelled: 10:01
```

This is valuable for:

* customer support
* reporting
* fraud investigation
* reconciliation
* debugging

Then release the capacity.

---

# 8. Payment succeeds but response is lost — ❌ This is your biggest correction

You said:

> response is lost → consider failure → cancel reservation → refund if necessary.

**Don't do this.**

This is a classic distributed-systems problem.

Consider:

```text
Reservation Service
       ↓
Payment Service
       ↓
₹500 charged successfully
       ↓
Payment Service → SUCCESS
       X
Network response lost
```

Reservation Service sees:

```text
TIMEOUT / UNKNOWN
```

It does **not** know whether payment succeeded.

If we immediately cancel:

```text
Payment succeeded
Reservation cancelled
```

we could have money charged without a valid reservation.

Instead:

```text
Payment result = UNKNOWN
        ↓
Query Payment Service
        ↓
GET /payments/{paymentId}
        ↓
SUCCESS / FAILED
```

If still unknown, we need a **reconciliation mechanism**.

For example:

```text
Payment
status = UNKNOWN
        ↓
Retry/reconciliation worker
        ↓
Payment provider
        ↓
SUCCESS
        ↓
Confirm reservation
```

This is much safer.

### Senior interview statement

> **A payment timeout is not equivalent to payment failure; it creates an unknown state that requires idempotent status lookup and reconciliation.**

This is a very important point to remember.

---

# 9. Redis update/invalidation — ⚠️ Small correction

You said:

> create reservation request → update/invalidate Redis.

The idea is right, but I'd make the sequence:

```text
DB transaction
    ↓
Reservation hold created
    ↓
Outbox/event
    ↓
Availability cache update/invalidation
```

The **DB state is authoritative**.

Don't make Redis the thing that determines whether the reservation succeeded.

Also, if you're maintaining an availability count in Redis, you might do something like:

```text
500 capacity
↓
reservation hold
↓
499 available
```

But we need to make sure Redis operations themselves are concurrency-safe.

We'll design this carefully in the next phase.

---

# 10. Kafka + Outbox — ✅ Good

Your understanding here is correct.

The key reason for Outbox is not simply:

> "Kafka might fail."

It's the atomicity problem:

```text
DB transaction
    ↓
Reservation CONFIRMED
```

and:

```text
Kafka publish
```

could fail separately.

Without Outbox:

```text
DB updated ✅
Kafka publish ❌
```

Now notification/event is lost.

With Outbox:

```text
DB transaction
 ├── Reservation = CONFIRMED
 └── Outbox Event = ReservationConfirmed
              ↓
        Outbox Publisher
              ↓
            Kafka
              ↓
       Notification Service
```

So the event is persisted with the business transaction.

Excellent concept to use here.

---

# The reservation flow I want you to arrive at

Putting everything together:

```text
Client
  │
  │ POST /reservations
  ▼
Reservation Service
  │
  ├── Idempotency check
  │
  ├── Authoritative availability/concurrency check
  │
  ├── Create reservation
  │      status = PENDING_PAYMENT
  │
  ├── Create/hold capacity
  │
  ▼
Payment Service
  │
  ├── SUCCESS ──────────────┐
  │                         │
  ├── FAILED                │
  │                         ▼
  │                   Reservation
  │                   CONFIRMED
  │
  └── TIMEOUT/UNKNOWN
          │
          ▼
      Reconciliation
          │
          ▼
   Query payment status
```

And separately:

```text
Reservation DB transaction
        ↓
     Outbox
        ↓
      Kafka
        ↓
Notification Service
        ↓
Email / SMS / Push
```

---

# Your Current Level

Your reasoning is strong in one particular area: **you are naturally thinking about what happens when things fail.**

That's exactly what differentiates basic system design from senior-level system design.

The concepts I want you to tighten now are:

* **Reservation vs reservation hold**
* **PENDING_PAYMENT state**
* **Unknown payment state**
* **Reconciliation**
* **Atomic concurrency control**
* **DB as source of truth vs Redis as optimization**
* **Outbox vs Kafka responsibilities**

---

# Next Step: Concurrency Control

This is the **most important part of this parking system**.

Now I want you to answer one focused scenario rather than another 10-question list:

> **Parking Lot A has only 1 available space. Two users, User A and User B, send `POST /reservations` at exactly the same time. How would you guarantee that only ONE reservation succeeds?**

Think about:

* PostgreSQL
* transactions
* row locking / `SELECT FOR UPDATE`
* unique constraints
* optimistic vs pessimistic locking
* Redis
* whether a distributed lock is actually necessary

**Don't jump to Kafka or AWS yet. Give me your approach for this race condition first.**


