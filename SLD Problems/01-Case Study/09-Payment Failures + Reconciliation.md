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
