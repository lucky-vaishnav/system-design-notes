(**i-dem-POH-tuhn-see** or **ee-dem-POH-tuhn-see**)

In real payment systems, idempotency is often **more important than distributed locks**.

Start your notes from here.

---

# What is Idempotency?

An operation is **idempotent** if:

```
Running it once
=
Running it multiple times
```

produces the same final result.

---

### Example

User clicks:

```
Pay Now
```

once.

Result:

```
Payment Created
```

Good.

---

Now user clicks:

```
Pay Now
```

twice.

Without idempotency:

```
Payment #1
Payment #2
```

Double charge.

Bad.

---

With idempotency:

```
Payment #1
```

Second request returns:

```
Already processed
```

No duplicate charge.

---

## Why do duplicate requests happen?

Many developers think:

```
User clicked twice
```

But there are many more causes.

---

#### Cause 1

User double-clicks button.

```
Request A
Request B
```

---

#### Cause 2

Mobile network retry.

```
Phone
 ↓
Request sent
 ↓
Response lost
 ↓
Phone retries
```

---

#### Cause 3

API Gateway retry.

```
Client
 ↓
Gateway
 ↓
Backend
```

Gateway retries automatically.

---

#### Cause 4

Webhook retries.

Uber.

Lyft.

Stripe.

Braintree.

All do this.

Example:

```
receipt.ready
receipt.ready
receipt.ready
```

Same event delivered multiple times.

---

## Real Example From Your Project

Suppose Lyft sends:

```
{
  "event_id":"abc123",
  "type":"ride.receipt.ready"
}
```

Then network timeout happens.

Lyft retries:

```
{
  "event_id":"abc123",
  "type":"ride.receipt.ready"
}
```

again.

Without idempotency:

```
Receipt generated twice
Email sent twice
Refund processed twice
```

---

# Idempotency Strategy

## Unique Request ID

Every request gets:

```
Idempotency-Key
```

Example:

```
POST /payment

Idempotency-Key: xyz123
```

---

Backend receives:

```
xyz123
```

and stores:

```
xyz123 → SUCCESS
```

---

Same request arrives again:

```
xyz123
```

Backend checks:

```
Already processed
```

Returns previous result.

---

#### Database Example

Table:

```
CREATE TABLE payment_requests (
    idempotency_key TEXTUNIQUE,
    payment_id BIGINT
);
```

---

First request:

```
xyz123
```

insert succeeds.

---

Second request:

```
xyz123
```

insert fails because:

```
UNIQUE constraint
```

Already processed.

---

### Why Idempotency is Powerful

Notice:

```
Leader Election failed
Redis Lock failed
Network retried
```

Still safe.

Because:

```
UNIQUE(idempotency_key)
```

becomes the final protection.

---

### Interview Question

Suppose:

```
Request A
Request B
```

arrive at the exact same millisecond.

Both use:

```
Idempotency-Key = xyz123
```

How can we prevent both from creating a payment?

Think about it before reading further.

---

### Answer

Use a database unique constraint.

```
ALTER TABLE payments
ADD CONSTRAINT unique_idempotency_key
UNIQUE(idempotency_key);
```

Now:

```
Request A → insert succeeds
Request B → unique violation
```

Only one payment exists.

---

### Notes Summary

```
Idempotency

Definition:
Performing an operation multiple times produces the same result.

Why needed:
- Double clicks
- Network retries
- Webhook retries
- Scheduler retries

Common implementation:
- Idempotency Key
- Database UNIQUE constraint

Benefits:
- Prevent duplicate payments
- Prevent duplicate refunds
- Prevent duplicate bookings
- Prevent duplicate webhook processing
```

## Is UNIQUE Constraint the only way?

No.

A **UNIQUE constraint is the strongest and most reliable implementation**, but there are several idempotency patterns.

---

### Pattern 1: UNIQUE Constraint (Most Common)

Example:

```
CREATE UNIQUE INDEX ux_payment_idempotency
ON payments(idempotency_key);
```

Request:

```
xyz123
```

First insert:

```
Success
```

Second insert:

```
Duplicate
```

Rejected.

---

#### Pros

```
Very reliable
Database guarantees correctness
Works across all servers
```

#### Cons

```
Need a database write
```

---

## Pattern 2: Processed Events Table

Very common for webhooks.

Table:

```
processed_events
-----------------
event_id
processed_at
```

Flow:

```
Receive webhook
↓
Check event_id
↓
Already processed?
↓
Skip
```

Example:

```
evt_123
```

arrives 10 times.

Only first gets processed.

---

## Pattern 3: State Machine

Example:

```
PENDING
PROCESSING
COMPLETED
FAILED
```

Refund flow:

```
Permit 123
↓
PROCESSING
```

Another worker sees:

```
Already PROCESSING
```

and skips.

---

## Pattern 4: Redis Idempotency

Store:

```
idempotency:xyz123
```

in Redis.

Example:

```
SETNX idempotency:xyz123
```

If exists:

```
Already processed
```

---

### Pros

```
Fast
```

### Cons

```
Redis loss
TTL expiry
Less reliable than DB
```

---

## Pattern 5: Natural Business Key

Sometimes you already have a unique business identifier.

Example:

```
Lyft Event ID
Uber Event ID
Braintree Transaction ID
```

Instead of creating:

```
idempotency_key
```

you use:

```
provider_event_id
```

as the idempotency key.

---

## Pattern 6: Upsert

Instead of:

```
INSERT
```

use:

```
INSERT ...
ON CONFLICTDO NOTHING
```

or

```
INSERT ...
ON CONFLICTUPDATE
```

PostgreSQL handles duplicates.

---

## Which one is best?

For critical operations:

```
Payments
Refunds
Bookings
```

Senior engineers usually trust:

```
Database Constraint
+
Idempotency Key
```

because databases are the source of truth.

### Note for interviews

If someone asks:

> "How would you implement idempotency?"
> 

A strong answer is:

```
Use an idempotency key and enforce uniqueness at the database level.
Additional layers like Redis or request caches can improve performance,
but the database should remain the final source of truth.
```

## Idempotency vs Concurrency vs Locking

This is where many engineers get confused.

For example:

```
SELECT FOR UPDATE
```

solves:

```
Concurrency
```

but not:

```
Duplicate retries
```

While:

```
Idempotency
```

solves:

```
Duplicate retries
```

but not:

```
Concurrent balance updates
```

## Idempotency for Webhooks

This is extremely relevant to your Uber/Lyft work.

---

### Problem

Lyft sends:

```
{
  "event_id":"evt_123",
  "type":"ride.receipt.ready"
}
```

Your API receives it.

You generate receipt.

Success.

---

Suddenly Lyft retries:

```
{
  "event_id":"evt_123",
  "type":"ride.receipt.ready"
}
```

again.

And again.

And again.

---

### Why do providers retry?

Because they don't know whether you processed it.

Example:

```
Lyft
 ↓
POST webhook
 ↓
Your server processes
 ↓
Network timeout
```

Lyft never receives:

```
200 OK
```

So Lyft assumes:

```
Webhook failed
```

and retries.

---

### Correct Solution

Create table:

```
processed_events
----------------
event_idUNIQUE
event_type
processed_at
```

---

Processing flow:

```
Receive event
↓
Insert event_id
↓
Success?
↓
Process event
```

---

If duplicate:

```
UNIQUE violation
```

Return:

```
200 OK
```

and skip.

---

### Important Interview Question

Should we use:

```
trip_id
```

or

```
event_id
```

as idempotency key?

---

### Wrong

```
trip_id
```

because:

```
ride.started
ride.completed
ride.receipt.ready
```

all have same trip.

---

### Correct

Usually:

```
event_id
```

because every event is unique.

---

## Out-of-Order Events

Another webhook problem.

Suppose:

```
ride.completed
```

arrives before:

```
ride.started
```

because of network delays.

This actually happens.

---

Good systems:

```
Do not assume order
```

Instead:

```
Store current status
```

and validate transitions.

---

## Idempotency for Payments

---

### Scenario

User clicks:

```
Pay Now
```

twice.

---

Request A:

```
payment_id = xyz
```

Request B:

```
payment_id = xyz
```

arrive together.

---

Without idempotency:

```
Charge $10
Charge $10
```

User pays:

```
$20
```

---

### Payment Idempotency Key

Client sends:

```
POST /payment

Idempotency-Key: abc123
```

---

Server stores:

```
abc123
```

along with:

```
payment result
```

---

Duplicate request:

```
abc123
```

returns:

```
same payment response
```

instead of creating another payment.

---

### Where should idempotency key be generated?

Interview favorite.

---

#### Option A

Frontend

```
Generate UUID
```

before calling payment API.

Most common.

---

#### Option B

Backend

Less common.

Useful when:

```
Backend owns workflow
```

---

### How long should we store it?

Depends.

---

Payment:

```
24 hours
7 days
30 days
```

common.

---

Webhook:

```
Forever
```

or very long.

#### Idempotency key Life Cycle(How long should we store it?)

#### Case 1: Database-based Idempotency

Example:

```
payment_requests
----------------
idempotency_key
payment_id
created_at
```

or

```
processed_events
----------------
event_id
processed_at
```

#### Do we keep these forever?

**Not necessarily.**

You decide based on business requirements.

Examples:

#### Payment APIs

Many companies keep:

```
24 hours
7 days
30 days
```

Then run cleanup:

```
DELETE FROM payment_requests
WHERE created_at< now()-interval'30 days';
```

because after 30 days nobody is retrying the same payment request.

---

#### Webhook Events

For Lyft/Uber:

```
event_id
```

might be kept:

```
30 days
90 days
180 days
```

depending on provider retry behavior and audit requirements.

---

#### Case 2: Database UNIQUE Constraint

Suppose:

```
CREATE TABLE refund_transactions (
  permit_id bigintUNIQUE
);
```

This is different.

Here the uniqueness itself represents business truth:

```
One refund per permit
```

In this case:

```
DO NOT DELETE
```

because removing the row removes the protection.

---

#### Case 3: Redis-based Idempotency

Example:

```
idempotency:abc123
```

stored in Redis.

Redis must have TTL:

```
24 hours
7 days
```

otherwise Redis grows forever.

Example:

```
await redis.set(
`idempotency:${key}`,
"processed",
"EX",
86400
);
```

---

### What should duplicate request return?

Bad:

```
409 Conflict
```

---

Better:

Return same response as first request.

Example:

Request 1:

```
{
  "payment_id":1001,
  "status":"SUCCESS"
}
```

---

Request 2:

Same idempotency key.

Return:

```
{
  "payment_id":1001,
  "status":"SUCCESS"
}
```

Again.

Client sees:

```
One payment
```

---

## Idempotency vs Locking

This is where people get confused.

---

### SELECT FOR UPDATE

Protects:

```
Concurrent updates
```

Example:

```
Wallet balance
Inventory
Account balance
```

---

### Idempotency

Protects:

```
Duplicate requests
Retries
Double clicks
Webhook retries
```

---

#### Example

User sends:

```
Pay Now
```

twice.

These are:

```
Duplicate requests
```

Use:

```
Idempotency
```

---

Two workers update same balance:

```
Worker A
Worker B
```

Use:

```
SELECT FOR UPDATE
```

---

## Senior Engineer Mental Model

Think:

#### Question 1

```
Can same request arrive multiple times?
```

Use:

```
Idempotency
```

---

#### Question 2

```
Can multiple workers update same data simultaneously?
```

Use:

```
Locking
```

---

#### Question 3

```
Can both happen?
```

Use:

```
Locking + Idempotency
```

Very common in:

```
Payments
Refunds
Bookings
```
