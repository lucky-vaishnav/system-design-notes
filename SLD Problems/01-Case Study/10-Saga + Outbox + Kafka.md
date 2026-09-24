# Saga + Outbox + Kafka

## Why do we need Saga?

Our reservation workflow now crosses multiple components:

```text
                    ┌───────────────┐
                    │ Reservation   │
                    │   Service     │
                    └───────┬───────┘
                            │
                            ↓
                    ┌───────────────┐
                    │    Payment    │
                    │    Service    │
                    └───────┬───────┘
                            │
                            ↓
                    External Payment
                       Provider
```

And potentially:

```text
Reservation
     ↓
Payment
     ↓
Notification
     ↓
Parking / Inventory
     ↓
Analytics
```

The problem is that these are **separate systems with separate databases**.

---

## Why can't we just use one DB transaction?

Imagine:

```text
BEGIN TRANSACTION

Reserve parking slot
     ↓
Charge payment
     ↓
Confirm reservation

COMMIT
```

This looks attractive, but the payment provider is an external system.

Your PostgreSQL transaction cannot do:

```text
BEGIN
  PostgreSQL operation
  External payment API operation
COMMIT
```

and guarantee atomicity across both.

For example:

```text
DB transaction:
SUCCESS

Payment provider:
SUCCESS

But our application crashes before COMMIT
```

Or:

```text
DB:
COMMIT SUCCESS

Payment:
SUCCESS

Notification:
FAILED
```

There is no single transaction manager that can roll everything back safely.

---

# What Saga gives us

A **Saga** breaks a distributed business transaction into a sequence of **local transactions**.

Each step commits independently.

If a later step fails, we perform a **compensating action** for the earlier successful steps.

For our parking system:

```text
Step 1:
Create reservation / hold inventory
        ↓
Step 2:
Process payment
        ↓
Step 3:
Confirm reservation
```

If Step 3 cannot complete:

```text
Step 3 FAILED
        ↓
Compensate Step 2
        ↓
Refund payment
        ↓
Compensate Step 1
        ↓
Release parking inventory
```

So instead of:

> "Everything must commit atomically"

we use:

> **"Each step commits locally, and failures are resolved through retries or compensating actions."**

---

# Important distinction

Saga does **not** give us ACID transactions across services.

It gives us a way to manage **business consistency across distributed operations**.

For example:

```text
Reservation DB
      ↓
COMMITTED

Payment DB
      ↓
COMMITTED

Refund DB
      ↓
COMMITTED
```

These aren't one transaction.

They're coordinated business steps.

---

# Our parking example

Let's make it concrete.

User wants:

```text
Parking Lot A
10:00–11:00
₹500
```

### Step 1 — Reserve inventory

PostgreSQL:

```text
Reservation:
R1
status = PENDING_PAYMENT

Inventory:
available = 9
```

Commit.

---

### Step 2 — Payment

Payment service:

```text
Payment:
P1
status = SUCCESS
amount = ₹500
```

Commit.

---

### Step 3 — Confirm reservation

Reservation service:

```text
R1:
PENDING_PAYMENT → CONFIRMED
```

Success.

Saga completes.

```text
Reservation = CONFIRMED
Payment = SUCCESS
Inventory = held
```

---

# Now introduce failure

Suppose:

```text
Step 1 → SUCCESS
Step 2 → SUCCESS
Step 3 → FAILURE
```

We can't simply undo Step 1 and Step 2 with a database rollback.

Instead:

```text
Step 3 failed
      ↓
Refund Payment P1
      ↓
Release Inventory for R1
      ↓
Reservation → CANCELLED/FAILED
```

That's compensation.

---

# But who coordinates all of this?

This brings us to the two Saga styles.

## 1. Choreography

There is no central coordinator.

Services react to events.

```text
Reservation Service
       │
       │ ReservationCreated
       ↓
     Kafka
       │
       ↓
Payment Service
       │
       │ PaymentSucceeded
       ↓
     Kafka
       │
       ↓
Reservation Service
       │
       │ ReservationConfirmed
       ↓
     Kafka
```

Each service knows:

> "When I receive event X, I perform action Y and publish event Z."

This is **Saga choreography**.

---

## 2. Orchestration

A central **Saga Orchestrator** controls the workflow.

```text
             Saga Orchestrator
              /      |       \
             ↓       ↓        ↓
      Reservation  Payment   Refund
        Service    Service   Service
```

The orchestrator says:

```text
1. Reserve inventory
2. Process payment
3. Confirm reservation
```

If payment succeeds but confirmation fails:

```text
Orchestrator:
    → Refund payment
    → Release inventory
```

For **our parking reservation system**, orchestration is generally easier to reason about because the workflow has explicit business steps and compensation.

I'm deliberately not calling it "better"; the choice depends on workflow complexity, coupling, event volume, and team/service boundaries.

---

# Where does Outbox come in?

This is the next critical piece.

Suppose Reservation Service does:

```text
DB transaction:
Reservation → PENDING_PAYMENT
```

Then:

```text
Publish Kafka event:
ReservationCreated
```

What happens if:

```text
DB COMMIT → SUCCESS

Kafka publish → FAILED
```

Now:

```text
Database:
Reservation exists

Kafka:
No event
```

Our workflow is stuck.

This is the **dual-write problem**.

---

# Outbox solves the dual-write problem

Instead of:

```text
DB
 ↓
commit

then

Kafka
 ↓
publish
```

we put the event into the **same PostgreSQL transaction**:

```text
BEGIN

Reservation:
PENDING_PAYMENT

Outbox:
ReservationCreated

COMMIT
```

Now both are committed together.

```text
PostgreSQL
 ├── Reservation
 └── Outbox Event
```

Then a separate publisher reads:

```text
Outbox
   ↓
Kafka
```

If Kafka is temporarily unavailable:

```text
Outbox:
Event still exists
```

So when Kafka comes back:

```text
Outbox → Kafka
```

The event isn't lost.

---

# The complete architecture starts looking like this

```text
                    ┌──────────────────────┐
                    │   Saga Orchestrator  │
                    └──────────┬───────────┘
                               │
             ┌─────────────────┼─────────────────┐
             ↓                 ↓                 ↓
       Reservation          Payment            Refund
          Service           Service            Service
             │                 │                 │
             ↓                 ↓                 ↓
        PostgreSQL         PostgreSQL        PostgreSQL
             │                 │                 │
             ↓                 ↓                 ↓
          Outbox            Outbox             Outbox
             │                 │                 │
             └─────────────────┼─────────────────┘
                               ↓
                             Kafka
```

And external payment:

```text
Payment Service
      ↓
External Payment Provider
```

---

## One very important interview point

Even with Kafka + Outbox + Saga, **duplicate events can happen**.

For example:

```text
Outbox → Kafka
       ↓
Consumer processes event
       ↓
DB update succeeds
       ↓
Consumer crashes before acknowledging Kafka
       ↓
Kafka delivers event again
```

So:

```text
PaymentSucceeded
```

may be processed twice.

Therefore every important consumer should be **idempotent**.

For example:

```text
event_id = EVT123
```

Consumer records:

```text
ProcessedEvents:
EVT123
```

If it receives `EVT123` again:

```text
Already processed
      ↓
Ignore / safely return
```

This connects directly with what we already learned about **payment idempotency**.

---

### The mental model I want you to build

Don't think:

> **Saga = Kafka**

They solve different problems.

```text
Saga
 ↓
Business workflow / compensation

Outbox
 ↓
Reliable DB → event handoff

Kafka
 ↓
Durable asynchronous event transport

Idempotency
 ↓
Safe duplicate processing

Reconciliation
 ↓
Repair unknown/inconsistent external state
```

These patterns work **together**, but they are not interchangeable.
