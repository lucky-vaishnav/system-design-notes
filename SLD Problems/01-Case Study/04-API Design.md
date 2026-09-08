#### My Thoughts/Thinking-
Yeah, so for our next phase, next step, API design. So I will talk about the major API of this parking reservation flow. So yeah, for parking service, majorly, like the most important API will be first will be the GET API for fetching the station list. And for that it will be just a GET API which will be called, and this API will fetch the station list from the DB. And for that, we can use the cache as well, because station list will mostly not change, very rare. So caching will be faster. And then there will be second API in parking service will be search for searching. So it will be, I am thinking like that station-wise the user can search for parking. So the first API will be slash parking slash station list, which will be the GET API. Second will be parking slash station code slash date. Date will be the parameter and station code as well. So this will be the API for searching. So date will be used for the particular date to be used. So this will be the parameter in the API route and station code as well. And in the response, this API will send available parking and fee or charges of parking and cancellation fee and some other information if required. And we can use caching here as well, but if we are going in that way that data availability logic will be run each time to get the actual status of parking, then yeah we will not use the caching. But yeah, if not, then we can have some caching as well. And for reservation service, so majorly there will be two APIs. First will be POST API for the slash reservation, and this API will expect the reservation info in the request body, like date, station code, and charges, fee, etc. if we really want to change the fee, send from front end. But yeah, backend as well also we verify the fee, and yeah after successfully this reservation API first will create... First, so this slash reservation API will create reservation for the itempendency purpose, and then the payment flow will be triggered where the payment service API of making payment will be called. And from the response from the payment service, we will have the transaction ID on success, which will be stored in the reservation table, which will be used for our reports as well. And fee and those information we will also store at the reservation table level for our transaction history report or transaction history checking for the user. So this reservation service, it will also have this API call reservation slash transaction history, which will expect the pagination variables like limit and offset and variable like related to the sorting as well, sort by and sorting order. And this API will also use some default sort by and sorting order. And the API which I talked all the APIs, so the user ID will be fetched from the token itself. So yeah, that is why I have not included the user ID in the request body of this API. And yeah, this transaction history API for that we need some basic level of the indexing as well, just at the table level to improve the performance. And yeah, then this API will send reservation-related information like what was the fee, when it was paid, what is the date, so this reservation is active or expired. So expired will be calculated using the date. If the date is already in past, so we can show it in the UI as expired. And if we go to the payment service, so the most important API will be making payment and refund. So it will be like payment slash make payment, or maybe we can just use slash payment POST. POST API, which will be expecting the reservation ID and fee and user ID from the token and payment-related info like payment method and those information if we are storing those information first via the front end, then yeah there will be a payment method table for the user specific where we can use the payment method tokens for payment. If not, then yeah within the reservation API we also need to send some payment-related information. And in the response, so the payment will return the success response and transaction-related information like transaction ID if we want to store it in the reservation table, and refund will be payment slash refund, which will be a POST request, which will be expecting the transaction ID against which the refund will be triggered, and refund amount if we are supporting partial refund, then the refund amount also can be sent, and there will be validation like refund amount should be lower than or equal to the transaction amount, and this type of validation will be there, and transaction ID will be used for the refund for which we are refunding, and after response we will store information like refunded amount, last refund date, this type of information in our refund table. And there will be a notification-related API as well in the notification service, which can be called from the reservation itself. So if we want to make it specific notification type-wise API, then we can have multiple APIs like notifications slash SMS, notifications slash email, or if we just want to support one API, then request body will have those type of information by which we can understand that which type of notification is this, and required object will be shared, like if this is an email, then if we are supporting the email text to be sent via the body, if this is the plan of the body information, then we can send it, otherwise we can have the template within the notification service, and according to the request body, template ID, template information, we can trigger that template and we can replace the required information sent in the request, like some amount if we want to show in email, some dates we want to show, then this type of information we need to send this API. Yeah, so it will be the... these will be the major APIs, I think. And yeah, so I am not sure if this is the way you are expecting, but yeah, please review this and also tell me what is the next step after this.

Yes — this is the right level of thinking for the API-design phase. You’ve covered the major APIs and, importantly, you’re already thinking about **caching, authentication, pagination, idempotency, backend price validation, partial refunds, and notification templates**.

I would rate this around **8/10** for a senior system-design first attempt. The main improvements are around **API responsibility, resource-oriented naming, idempotency, and separating synchronous vs asynchronous operations**.

---

# 1. Parking APIs — Good, with one naming improvement

Your idea:

```text
GET /parking/station-list
GET /parking/{stationCode}/{date}
```

is reasonable, but I'd make the second API more explicit:

```text
GET /parking/stations/{stationId}/availability?date=2026-09-09
```

or, if we're searching across multiple stations:

```text
GET /parking/availability?stationId=123&date=2026-09-09
```

The second style is generally more flexible.

### Station list

```text
GET /parking/stations
```

Response could contain:

```text
stationId
stationCode
name
location
capacity
parkingFee
cancellationPolicy
status
```

Caching is a **very good choice** here because station metadata changes infrequently.

---

# 2. Availability API — Important distinction

Your thinking here is good.

```text
GET /parking/availability?stationId=123&date=2026-09-09
```

could return:

```text
{
  "capacity": 500,
  "reserved": 420,
  "available": 80,
  "fee": 25
}
```

But remember the principle we discussed:

> **Availability displayed to the user can be slightly stale; reservation creation cannot rely solely on this response.**

So:

```text
GET availability
       ↓
Redis/cache/read model
       ↓
"80 spaces available"
       ↓
User clicks Reserve
       ↓
POST /reservations
       ↓
Authoritative DB check
```

This distinction is important in an interview.

---

# 3. Reservation API — This needs the biggest improvement

You proposed:

```text
POST /reservation
```

That's correct conceptually.

I'd use:

```text
POST /reservations
Idempotency-Key: <unique-key>
```

Request:

```text
{
  "stationId": "123",
  "parkingDate": "2026-09-09"
}
```

### Don't trust these from the frontend

You mentioned sending:

> charges, fee, etc.

I'd strongly avoid accepting the final price from the client.

The backend should calculate:

```text
station
+ parking date
+ applicable pricing rules
+ discounts
+ taxes
= final amount
```

The client can send a displayed price for UI purposes, but the backend should **recalculate and verify the authoritative amount**.

Otherwise a malicious client could send:

```text
amount = ₹1
```

instead of:

```text
amount = ₹500
```

---

# 4. Reservation → Payment flow

There's an important architectural decision here.

You said:

> reservation is created → payment service is called

That's possible, but we need to define the reservation state.

I'd use something like:

```text
PENDING_PAYMENT
PAYMENT_PROCESSING
CONFIRMED
PAYMENT_FAILED
CANCELLED
EXPIRED
REFUND_PENDING
REFUNDED
```

For example:

```text
POST /reservations
        ↓
Create reservation
PENDING_PAYMENT
        ↓
Payment
        ↓
SUCCESS
        ↓
CONFIRMED
```

But if payment fails:

```text
PENDING_PAYMENT
       ↓
PAYMENT_FAILED
       ↓
release reservation/hold
```

This becomes very important when we discuss **Saga + state machine + failure scenarios**.

---

# 5. Payment API

Your idea:

```text
POST /payment
```

is fine.

I'd make it:

```text
POST /payments
Idempotency-Key: <key>
```

Request:

```text
{
  "reservationId": "R123",
  "paymentMethodToken": "..."
}
```

The backend gets:

```text
userId → authentication token
amount → reservation/database
```

It should **not trust the amount from the client**.

Payment service determines:

```text
reservation → amount
```

and charges that amount.

### Payment is definitely idempotent

This is critical.

Imagine:

```text
Client → Payment
        ↓
Payment succeeds
        ↓
Network timeout
        ↓
Client retries
```

Without idempotency:

```text
₹500 charged
₹500 charged again ❌
```

With:

```text
Idempotency-Key: abc123
```

the second request returns the existing result.

---

# 6. Refund API

Your:

```text
POST /payments/refund
```

is reasonable.

I'd prefer:

```text
POST /payments/{paymentId}/refunds
Idempotency-Key: <key>
```

Request:

```text
{
  "amount": 400,
  "reason": "USER_CANCELLED"
}
```

This makes the resource relationship clearer:

```text
Payment
   ↓
Refund
```

And yes, backend validates:

```text
refundAmount <= refundableAmount
```

rather than simply checking against the original transaction amount, because **multiple partial refunds** may exist.

---

# 7. History API — Good

Your pagination thinking is correct.

I'd use:

```text
GET /reservations?limit=20&offset=0&sortBy=createdAt&sortOrder=desc
```

Authentication gives us:

```text
userId
```

so the client doesn't need to send it.

### One senior-level improvement

For large datasets, prefer **cursor-based pagination** over offset pagination:

```text
GET /reservations?limit=20&cursor=abc123
```

Offset is simpler and perfectly acceptable initially.

But at very large scale:

```text
OFFSET 1000000
```

can become expensive.

---

# 8. "Expired" status

You said:

> expired can be calculated from date.

I'd be careful here.

If expiration is important business state, don't rely only on:

```text
parkingDate < today
```

For example, a reservation could have:

```text
status = CANCELLED
parkingDate = yesterday
```

Calling everything from the past "expired" can hide the actual lifecycle.

Better:

```text
status = CONFIRMED
parkingDate = past
```

UI can **display** it as "Completed/Past" without changing the stored business status.

---

# 9. Notification API — I would change this significantly

You proposed:

```text
POST /notifications/email
POST /notifications/sms
```

or:

```text
POST /notifications
{
   type: "EMAIL"
}
```

The second is better.

But more importantly:

> **Reservation Service should generally not synchronously call Notification Service during the reservation transaction.**

Instead:

```text
Reservation
    ↓
DB transaction
    ↓
Outbox
    ↓
Kafka
    ↓
Notification Service
    ↓
Email/SMS/Push
```

Example:

```text
Reservation CONFIRMED
        ↓
ReservationConfirmed event
        ↓
Kafka
        ↓
Notification Service
        ↓
Email + Push
```

This means the reservation isn't blocked because an email provider is slow.

That's a good place to apply what we've already learned about **Outbox + Kafka**.

---

# 10. Your API authentication approach is correct

You said:

> user ID will come from token.

Correct.

For example:

```text
Authorization: Bearer <JWT>
```

Backend extracts:

```text
userId
```

So don't do:

```text
POST /reservations

{
   "userId": "123"
}
```

because the client could attempt to impersonate another user.

The backend gets identity from the authenticated context.

---

# 11. APIs I'd finalize for our case study

At this point, I'd organize them approximately like this:

### Parking

```text
GET  /parking/stations
GET  /parking/availability
```

### Reservations

```text
POST /reservations
GET  /reservations/{reservationId}
GET  /reservations
POST /reservations/{reservationId}/cancel
```

### Payments

```text
POST /payments
GET  /payments/{paymentId}
POST /payments/{paymentId}/refunds
```

### Notifications

Internal/event-driven rather than public APIs wherever possible:

```text
ReservationConfirmed
PaymentSucceeded
RefundCompleted
ReservationCancelled
```

consumed asynchronously by Notification Service.

---

# 12. Idempotency summary

This is something I want you to remember for interviews:

| API                           | Idempotency                             |
| ----------------------------- | --------------------------------------- |
| GET availability              | Naturally idempotent                    |
| GET reservation               | Naturally idempotent                    |
| POST reservation              | **Yes**                                 |
| POST payment                  | **Yes — critical**                      |
| Cancel reservation            | **Yes**                                 |
| Refund                        | **Yes — critical**                      |
| Notification event processing | **Yes — consumer should be idempotent** |

---

# Where We Are Now

We've now completed:

* Requirements ✅
* Scale ✅
* High-level architecture ✅
* Core entities/data model ✅
* API design ✅

The **next step is Reservation Flow / Request Flow**.

And this is where the system gets much more interesting.

## Next task for you

Design the complete flow for:

> **User tries to reserve a parking space for a particular station/date.**

Think through this sequence yourself:

```text
Client
  ↓
API Gateway
  ↓
?
  ↓
?
  ↓
Reservation created
  ↓
Payment
  ↓
?
  ↓
Reservation confirmed
```

Tell me:

1. Which services are called?
2. Which calls are synchronous?
3. Which calls are asynchronous?
4. When do we check availability?
5. When do we create the reservation?
6. When do we call Payment Service?
7. What happens if payment fails?
8. What happens if payment succeeds but the response is lost?
9. When do we update/invalidate Redis?
10. Where does Kafka/Outbox come into the flow?

**Don't worry about getting it perfect. Design the flow first. I'll challenge it step-by-step like an interviewer.**



