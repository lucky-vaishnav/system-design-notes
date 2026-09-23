## Payment Failures + Reconciliation
### Part A — Payment timeout / unknown state
This is the next important piece because our reservation flow currently looks like:

```text
Client
  ↓
Reservation API
  ↓
PostgreSQL
  ↓
PENDING_PAYMENT
  ↓
Payment Provider
  ↓
CONFIRMED
```

The tricky part is that **PostgreSQL and the external payment provider are two independent systems**.

There is no single transaction that can atomically commit both.

### Let's start with the failure scenarios

Consider this:

```text
1. Create reservation
2. DB → PENDING_PAYMENT
3. Call payment provider
4. Payment succeeds
5. Our application crashes
6. We never update reservation → CONFIRMED
```

Now we have:

```text
Our DB:
PENDING_PAYMENT

Payment Provider:
SUCCESS
```

The user may have been charged, but our system thinks payment is still pending.

---

Another scenario:

```text
1. Create reservation
2. DB → PENDING_PAYMENT
3. Call payment provider
4. Payment succeeds
5. Response is lost because of network failure
```

Again:

```text
DB       = PENDING_PAYMENT
Provider = SUCCESS
```

The payment actually happened, but we don't know it from the API response.

---

And the opposite:

```text
DB       = PENDING_PAYMENT
Provider = FAILED
```

This one is easier: we can release the reservation.

But there is another important case:

```text
DB       = PENDING_PAYMENT
Payment Provider = UNKNOWN
```

"UNKNOWN" is actually very important.

We cannot safely assume:

```text
UNKNOWN → FAILED
```

because the payment might have succeeded but the response was lost.

---

## The key principle

For payment systems:

> **A timeout or network error does not necessarily mean payment failed.**

It means:

> **We don't know the payment result yet.**

That distinction is extremely important in a senior system-design interview.

---

### Before we design reconciliation

I want you to think through this scenario yourself:

```text
Reservation R1

DB:
PENDING_PAYMENT

Payment request → external provider

Our API gets a timeout.

We don't know whether the provider:
- received the request and failed it
- received it and succeeded
- never received it
```

**Question:**

What should our system do with `R1`?

Should we immediately mark the reservation as `FAILED/EXPIRED`, retry the payment request, wait and check the provider, or use some combination?

Give me your design first. Then we'll build the production-grade **payment failure + reconciliation flow** from your answer.

#### My Thoughts That Time -
So for this R1 scenario, our API get timeout. So reason for the timeout can be, I think, our payment API got timeout, so that our reservation service do not get any response. There is some network error, because of that our reservation service do not get any response. Or maybe the external provider API gets timeout. So the payment service is not aware of the final status. So for these type of cases, I think whenever this happens, we can use the retry logic, and retry logic can be first at that if there is a timeout error, then we will know. So we can try it again and do some type of reconciliation within the existing transaction-related DB. And if our system do not have the status, then we can check with the external provider again. So this type of reconciliation we can do. And second, if let's say our service gets down, services get down. So before that, we should store some information, like the status with unknown or pending payments. This type of status we can fetch, and we can use some type of job, maybe cron job, and we can try the same reconciliation for those pending payments when the service got restarted again. For saving the data, we can similar, like outbox pattern, we can use, I think. So yeah, mostly the reconciliation part via some jobs, or at the time of the timeout error, so network error, this type of thing we can do, I think, according to me.  

Yes — your overall direction is **correct**, and you identified the two important recovery paths:

1. **Immediate recovery** when the payment call times out.
2. **Background reconciliation** for payments that remain unresolved because the service crashed or the response was lost.

There are a few important refinements I'd make for a senior-level design.

### 1. Don't immediately retry the payment blindly

Your retry idea is correct, but there's a dangerous case:

```text
Request 1 → Payment Provider
           ↓
       Provider charges user
           ↓
       Response lost
           ↓
Our service → TIMEOUT
```

If we simply retry:

```text
Request 2 → Payment Provider
```

we could potentially **charge the user twice**.

So the payment request needs an **idempotency key**, for example:

```text
payment_id = PAY123
idempotency_key = RESERVATION_R1_PAYMENT
```

Then:

```text
First request:
PAY123 → SUCCESS

Retry:
PAY123 → same result
```

The provider should recognize the same idempotency key and avoid creating another charge, assuming the provider supports idempotency.

This is one of the most important pieces of the payment design.

---

### 2. Your "UNKNOWN" idea is important

I'd model the state more explicitly:

```text
PENDING_PAYMENT
       ↓
   Payment call
       ↓
 ┌─────┼─────────┐
 ↓     ↓         ↓
SUCCESS FAILED   UNKNOWN
```

`UNKNOWN` means:

> We don't know what happened at the payment provider.

It does **not** mean payment failed.

Then:

```text
UNKNOWN
   ↓
Query payment provider
   ↓
 ┌───────┼────────┐
 ↓       ↓        ↓
SUCCESS FAILED   STILL UNKNOWN
```

If still unknown, reconciliation retries later.

---

### 3. Your background job idea is exactly what we need

Suppose the service crashes here:

```text
DB:
Reservation = PENDING_PAYMENT

        ↓

Payment Provider:
SUCCESS

        ↓

Our service crashes
```

When the service comes back, we shouldn't depend on someone making another API request.

We have a reconciliation worker:

```text
                 PostgreSQL
                     │
                     ↓
          Find unresolved payments
                     │
                     ↓
             Reconciliation Job
                     │
                     ↓
             Payment Provider
                     │
              ┌──────┴──────┐
              ↓             ↓
           SUCCESS        FAILED
              ↓             ↓
        CONFIRMATION      RELEASE
```

This is much more reliable than saying:

> "We'll reconcile when the service restarts."

Because the service might not restart immediately, or we might have many instances.

I'd say:

> **A scheduled/continuous reconciliation worker periodically finds unresolved payments and reconciles them with the provider.**

---

## 4. Where Outbox fits

You mentioned Outbox, which is a good instinct, but let's distinguish its purpose.

The Outbox pattern primarily solves:

> **"Our DB transaction succeeded, but publishing an event/message failed."**

For example:

```text
BEGIN
  Reservation → PENDING_PAYMENT
  Outbox → PaymentInitiated
COMMIT
```

Then:

```text
Outbox Worker
     ↓
Kafka
     ↓
Payment processing / reconciliation
```

So Outbox gives us a **durable record that the payment workflow needs to continue**.

But the Outbox itself doesn't magically reconcile the payment provider. We still need the **reconciliation worker**.

---

# The production flow

Putting your ideas together:

```text
                    Reservation
                         │
                         ↓
                 PostgreSQL Transaction
                         │
                         ├── Reservation = PENDING_PAYMENT
                         │
                         └── Outbox Event
                              │
                              ↓
                           Kafka
                              │
                              ↓
                       Payment Worker
                              │
                              ↓
                    Payment Provider
                              │
                ┌─────────────┼──────────────┐
                ↓             ↓              ↓
             SUCCESS        FAILED         TIMEOUT
                ↓             ↓              ↓
            CONFIRMED       RELEASE       UNKNOWN
                                              │
                                              ↓
                                      Reconciliation
                                           Worker
                                              │
                                              ↓
                                     Query Provider
                                              │
                              ┌───────────────┼──────────────┐
                              ↓               ↓              ↓
                           SUCCESS          FAILED        UNKNOWN
                              ↓               ↓              ↓
                          CONFIRMED         RELEASE        Retry later
```

And throughout the flow:

```text
             idempotency_key
                   ↓
             Payment Provider
```

prevents duplicate payment attempts from creating duplicate charges.

---

## One more important production issue: reservation expiry

Remember our earlier reservation state:

```text
AVAILABLE
    ↓
PENDING_PAYMENT
    ↓
CONFIRMED
```

Suppose payment remains `UNKNOWN` for 20 minutes.

We cannot hold the parking space forever.

So we need something like:

```text
PENDING_PAYMENT
expires_at = 10:15 AM
```

If reconciliation hasn't resolved it by the expiry policy, we need a carefully defined business rule.

For example:

```text
PENDING_PAYMENT
       │
       ├── Provider says SUCCESS → CONFIRMED
       │
       ├── Provider says FAILED → RELEASED
       │
       └── Still UNKNOWN after policy → special recovery handling
```

The last case is deliberately **not simply "mark failed"**, because we could have charged the customer.

We may need to continue provider reconciliation and potentially initiate a refund/compensation workflow if the reservation can no longer be honored.

That leads directly into our next layer of complexity: **payment reconciliation + refunds + Saga/compensation**.

### Your answer, refined for an interview

You could say:

> "If the payment API times out, I would not immediately assume that the payment failed because the timeout only tells us that our service didn't receive a definitive response. I would use an idempotency key for the payment so retries don't create duplicate charges. The payment would remain in an unresolved or unknown state, and we'd query the provider for the authoritative payment status. I'd also persist the payment workflow state so that if our service crashes after the provider succeeds, a background reconciliation worker can find unresolved payments and reconcile them. An Outbox can make the workflow trigger durable, while the reconciliation worker handles provider-state recovery."

That's a **strong senior-level answer**.

Next, we should go deeper into **what happens when payment succeeds but the reservation cannot be confirmed**, because that's where **compensation/refunds and eventually Saga** become necessary.

---

### Part B — Next: Payment succeeds, but reservation confirmation fails

This is one of the most important failure scenarios in our parking system because **money and inventory are in two different systems**.

### Start with the scenario

Suppose:

```text
Reservation R1
Parking Slot S1
Amount = ₹500
```

Our flow:

```text
1. Create reservation
        ↓
2. DB → PENDING_PAYMENT
        ↓
3. Call Payment Provider
        ↓
4. Payment Provider → SUCCESS
        ↓
5. User is charged ₹500
        ↓
6. Our service tries to update:
       PENDING_PAYMENT → CONFIRMED
        ↓
7. DB update fails
```

Now we have:

```text
Payment Provider:
SUCCESS / ₹500 charged

Our DB:
PENDING_PAYMENT
```

This is recoverable.

We **should not immediately refund**, because the first thing we should do is determine whether the reservation confirmation can simply be retried.

---

## First principle: Payment success does not automatically mean refund

Our first action should be:

```text
Payment SUCCESS
      ↓
Try to confirm reservation
      ↓
Did confirmation succeed?
```

If yes:

```text
PENDING_PAYMENT → CONFIRMED
```

Done.

If the DB update failed because of a temporary problem:

```text
DB unavailable
transaction timeout
network issue
deadlock
temporary service failure
```

then a retry may successfully confirm the reservation.

So:

```text
Payment SUCCESS
      ↓
Confirmation failed
      ↓
Retry confirmation
      ↓
SUCCESS
```

No refund is required.

---

# But what if confirmation cannot happen?

Here's where it becomes more interesting.

Suppose:

```text
Payment = SUCCESS
Reservation = PENDING_PAYMENT
```

Then the reservation expires:

```text
PENDING_PAYMENT
      ↓
EXPIRED
      ↓
Parking inventory released
```

Now the customer has paid, but the parking space is no longer held.

We cannot simply change:

```text
EXPIRED → CONFIRMED
```

because another customer might already have booked that slot.

So now we have:

```text
Customer paid ₹500
        ↓
No reservation available
        ↓
Need compensation
```

Usually that means:

```text
Refund ₹500
```

---

# The important thing: Don't refund immediately on every failure

Imagine:

```text
Payment SUCCESS
       ↓
Confirmation DB update fails
       ↓
Refund immediately
```

That can be dangerous.

The DB update might actually have succeeded, but our application received a timeout.

For example:

```text
Our service → DB: CONFIRM
                    ↓
                 SUCCESS
                    ↓
              Response lost
                    ↓
Our service thinks: "confirmation failed"
                    ↓
Refund
```

Now we could end up with:

```text
Reservation = CONFIRMED
Payment = SUCCESS
Refund = SUCCESS
```

That's an inconsistent financial state.

So we need **reconciliation before compensation**.

---

# A safer decision flow

```text
Payment SUCCESS
      ↓
Confirm reservation
      ↓
 ┌────┴──────────┐
 ↓               ↓
SUCCESS        UNKNOWN/FAILURE
 ↓               ↓
CONFIRMED      Reconcile
                 ↓
          Is reservation actually
             confirmed?
            /          \
          YES           NO
           ↓             ↓
        Done          Can we still
                      confirm it?
                       /     \
                     YES      NO
                      ↓        ↓
                  Confirm     Refund
```

The key idea is:

> **Before issuing a refund, establish that the reservation cannot be fulfilled.**

---

# Now let's introduce the Refund state

We shouldn't have:

```text
Payment = SUCCESS
Refund = boolean
```

Instead, model refund as a state machine.

For example:

```text
NOT_REQUIRED
     ↓
REFUND_PENDING
     ↓
REFUND_PROCESSING
     ↓
REFUNDED
```

And failures:

```text
REFUND_PROCESSING
        ↓
     FAILED
        ↓
 retry
```

This becomes important because **refunds can fail too**.

---

## Example: Refund API times out

Suppose:

```text
Payment = SUCCESS
Reservation = EXPIRED

        ↓

Refund request → Payment Provider

        ↓

TIMEOUT
```

What do we know?

We don't know whether:

```text
Refund failed
```

or:

```text
Refund succeeded but response was lost
```

Exactly the same problem we had with payment.

Therefore:

> **Refunds also require idempotency and reconciliation.**

For example:

```text
refund_id = REF123
idempotency_key = REFUND_RESERVATION_R1
```

If we retry:

```text
Refund request REF123
```

the provider should not create two refunds.

---

# So now we have two reconciliation processes

### Payment reconciliation

```text
PENDING_PAYMENT / UNKNOWN
        ↓
Check provider
        ↓
SUCCESS / FAILED / UNKNOWN
```

### Refund reconciliation

```text
REFUND_PENDING / UNKNOWN
        ↓
Check provider
        ↓
REFUNDED / FAILED / UNKNOWN
```

Both need:

* durable state
* idempotency
* retry
* exponential backoff
* background reconciliation
* monitoring/alerting

---

# One more important scenario

Suppose payment succeeds:

```text
Payment = SUCCESS
```

But reservation confirmation fails because the parking inventory is no longer available.

For example:

```text
Reservation R1
Parking Slot S1

Payment → SUCCESS

But meanwhile:
S1 → booked by another reservation
```

We now **cannot fulfill R1**.

Our business workflow becomes:

```text
Payment SUCCESS
       ↓
Reservation cannot be fulfilled
       ↓
Refund required
       ↓
Refund SUCCESS
       ↓
Reservation → PAYMENT_FAILED/REFUNDED
```

This is what we call **compensation**.

We're effectively undoing the financial effect because we couldn't complete the business operation.

And this is exactly where we're heading toward **Saga**.

---

# The big picture

Our parking workflow is now:

```text
                 Reservation
                     │
                     ↓
              PENDING_PAYMENT
                     │
                     ↓
                Pay Provider
                     │
            ┌────────┼────────┐
            ↓        ↓        ↓
         SUCCESS   FAILED   UNKNOWN
            │        │        │
            │        │        └── Reconciliation
            │        │
            ↓        ↓
       Confirm    Release
       Booking
            │
       ┌────┴─────┐
       ↓          ↓
    SUCCESS    Cannot confirm
       │          │
       ↓          ↓
   CONFIRMED    REFUND
                  │
             ┌────┴────┐
             ↓         ↓
          SUCCESS    UNKNOWN
             │         │
             ↓         ↓
          CLOSED   Reconcile
```

### The senior-level mental model

There are **three different concerns**:

**1. Retry**

> Try the operation again when failure is likely transient.

**2. Reconciliation**

> Ask the authoritative external system what actually happened when the result is unknown.

**3. Compensation**

> If one business step succeeded but the overall business operation cannot complete, perform a compensating action such as a refund.

That's the distinction I want you to remember.
