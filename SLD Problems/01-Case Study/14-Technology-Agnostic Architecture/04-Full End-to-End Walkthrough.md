Yes. We'll keep this as a **short synthesis**, not another deep dive. The goal is to connect the pieces and show how you'd narrate the flow in an interview.

## Topic 12 — Full End-to-End Walkthrough

### The one flow we'll follow

> **Search → Reserve → Pay → Confirm**

---

## 1. User searches for parking

**Interview explanation:**

> "The user first searches for available parking. Since this is read-heavy, I would primarily serve availability from Redis. The data can be eventually consistent because this is only a view for the user."

Important point:

> **Redis helps answer "what appears available?" — it does not decide whether the reservation is allowed.**

---

## 2. User clicks Reserve

Now correctness becomes important.

> "When the user actually reserves a slot, I don't trust the Redis result. I go to PostgreSQL, which is the source of truth."

Conceptually:

```text
BEGIN
  ↓
Lock/check inventory
  ↓
Reserve inventory
  ↓
Create PENDING_PAYMENT
  ↓
Set expires_at
  ↓
COMMIT
```

Interview explanation:

> "The transaction ensures that concurrent users cannot successfully reserve the same inventory. I also create a temporary `PENDING_PAYMENT` hold so the inventory isn't offered to another user while payment is in progress."

**Key distinction:**

> Search availability ≠ reservation authorization.

---

## 3. Call the payment provider

After the DB transaction commits:

> "I call the external payment provider outside the database transaction. I don't want to hold database locks while waiting for an external network call."

Then:

```text
Payment
 ├── SUCCESS
 ├── FAILED
 └── UNKNOWN
```

### SUCCESS

Continue toward confirmation.

### FAILED

Release the reservation/inventory.

### UNKNOWN

This is important:

> "A timeout doesn't necessarily mean the payment failed. The request could have succeeded at the provider while the response was lost."

So:

```text
UNKNOWN
   ↓
Reconciliation
   ↓
Provider status
   ↓
SUCCESS / FAILED
```

And idempotency prevents a retry from accidentally creating a second charge.

---

## 4. Confirm the reservation

Once payment is successfully established:

> "We update the reservation to `CONFIRMED`."

The transactional state change and event publication are handled through the mechanism we've already designed:

```text
PostgreSQL transaction
       ↓
Reservation = CONFIRMED
       +
Outbox event
       ↓
     Kafka
       ↓
 ┌─────┼─────────┐
 ↓     ↓         ↓
Email  Cache   Analytics
```

The important interview point:

> "Notification or analytics should not block the critical reservation path."

---

# 5. What if something fails?

You don't need to explain every failure. Just show that the architecture has an answer.

| Failure                                 | What happens                                                                           |
| --------------------------------------- | -------------------------------------------------------------------------------------- |
| API instance crashes                    | Load balancer sends traffic to another instance                                        |
| Redis unavailable                       | Controlled fallback/protection; DB remains authoritative                               |
| Concurrent reservation                  | DB transaction/locking determines the winner                                           |
| Payment fails                           | Release the temporary hold                                                             |
| Payment times out                       | Mark UNKNOWN and reconcile                                                             |
| Kafka unavailable                       | Outbox retains the event                                                               |
| Consumer crashes                        | Kafka reassigns the partition to another consumer                                      |
| Payment succeeds but confirmation fails | Retry/reconcile first; compensate/refund if reservation ultimately cannot be fulfilled |

---

# 6. The 2–3 minute interview version

If the interviewer says:

> **"Walk me through a reservation."**

You could answer approximately like this:

> "The user first searches for parking, and availability is primarily served from Redis because it's read-heavy and can tolerate some eventual consistency. When the user actually reserves a slot, Redis is not authoritative. The API starts a PostgreSQL transaction, checks and locks the relevant inventory, reserves it, and creates a PENDING_PAYMENT reservation with an expiry time.
>
> Once that transaction commits, we call the external payment provider outside the DB transaction. We use an idempotency key because a payment timeout doesn't necessarily mean the payment failed. If the result is unknown, we reconcile with the provider rather than blindly retrying.
>
> Once payment is confirmed, we mark the reservation confirmed. The transactional state change can generate an Outbox event, which is published to Kafka for asynchronous work such as notifications, cache updates, and analytics. This keeps those operations from blocking the critical booking path.
>
> Throughout the flow, PostgreSQL remains the source of truth for reservation correctness, while Redis and Kafka are used for performance and asynchronous processing."

**That is enough for the initial walkthrough.**

Then let the interviewer ask:

> "How do you prevent double booking?"

→ concurrency deep dive.

> "What happens if payment times out?"

→ UNKNOWN + reconciliation.

> "How does this scale?"

→ application/DB/Redis/Kafka scaling.

> "What happens if Redis goes down?"

→ failure handling.

You already know the deep answers.

---

# The biggest lesson from this walkthrough

You don't need to **remember 25 topics as 25 separate things**.

You should now see them as layers around the main request:

```text
                  USER REQUEST
                       │
                 ┌─────▼─────┐
                 │ Main Flow │
                 └─────┬─────┘
                       │
       ┌───────────────┼────────────────┐
       ↓               ↓                ↓
  Correctness       Performance       Reliability
       │               │                │
 Transactions       Redis            Retries
 Locks              Scaling          Idempotency
 Consistency        Kafka            Reconciliation
       │                                │
       └──────────────┬─────────────────┘
                      ↓
              Security / Observability
                      ↓
                  DR / HA
```

That's the **top-level mental model** I want you to carry into your next system-design problem.

### Topic 12 — Full End-to-End Walkthrough: **COMPLETE ✅**

### Parking Reservation & Payment System: **COMPLETE ✅**

We have now covered this problem from **fundamentals → deep concepts → architecture → AWS → failures → end-to-end flow** without turning the final stage into another giant set of notes.

For the **next system-design problem**, we'll start applying this knowledge more independently and use the process more like an actual interview.
