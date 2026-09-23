## Payment Failures + Reconciliation

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
