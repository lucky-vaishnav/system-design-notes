(especially with Kafka and Outbox)

This is one of the most important DDD topics because it connects **DDD → Kafka → Outbox → Saga → Event-Driven Architecture**, all of which you've already learned.

Many senior interviewers ask this in different ways:

- "What is a Domain Event?"
- "How is it different from a Kafka event?"
- "Does every Domain Event become a Kafka message?"
- "Where does the Outbox Pattern fit?"

After this lesson, these questions should all make sense.

---

# DDD Advanced Series

```
Part 1 ✅ DDD Basics
Part 2 ✅ Aggregate Design

➡️ Part 3 — Domain Events vs Integration Events (Today)

Part 4 — Strategic DDD
```

---

# 1. Why Do Events Exist?

Suppose a customer places an order.

Without events:

```
Order Service

↓

Save Order

↓

Call Payment

↓

Call Inventory

↓

Call Email

↓

Call Shipping
```

Everything is tightly coupled.

If Email Service is down...

Order creation fails.

Bad design.

---

Instead:

```
Order Created

↓

Publish Event

↓

Payment Service

Inventory Service

Email Service

Analytics Service
```

Order Service doesn't know who is listening.

This is Event-Driven Architecture.

---

# 2. The Big Confusion

Most developers think:

```
Domain Event

=

Kafka Event
```

❌ Wrong.

These are **different concepts**.

---

# 3. Domain Event

## Definition

A **Domain Event** represents something important that happened **inside the business domain**.

Examples:

```
OrderPlaced

PaymentAuthorized

ParkingStarted

ParkingEnded

WalletRefilled

TripCompleted
```

Notice:

These are **business events**.

Not technical events.

---

Example

User buys Tap & Go parking.

Business says:

```
Parking Started
```

That is a Domain Event.

---

# 4. Where Does a Domain Event Live?

Inside the same service.

Example

```
Parking Service

↓

ParkingSession.start()

↓

ParkingStarted (Domain Event)
```

Nothing has left the service yet.

---

Example code

```tsx
class ParkingStarted {

    constructor(
        public sessionId: string,
        public userId: string
    ) {}

}
```

Very simple.

---

# 5. Why Create Domain Events?

Suppose your Aggregate finishes work.

```
ParkingSession

↓

Started Successfully
```

Now something important happened.

Instead of calling everything directly,

raise an event.

```
ParkingStarted
```

---

# 6. What Can Listen?

Inside the same service.

Examples

```
Audit Logger

↓

Notification

↓

Statistics

↓

Cache Update
```

All inside Parking Service.

---

Notice:

No Kafka required.

---

# 7. Integration Event

Now suppose another service needs to know.

Example

Billing Service.

Analytics Service.

Notification Service.

Inventory Service.

Now we must leave our service.

That becomes:

## Integration Event

---

Example

```
ParkingStarted

↓

Publish

↓

Kafka
```

This Kafka message

is an Integration Event.

---

# 8. Visual

```
Parking Aggregate

↓

ParkingStarted

(Domain Event)

↓

Outbox

↓

Kafka

↓

ParkingStartedIntegrationEvent

↓

Billing Service

↓

Analytics
```

This is the flow used in many production systems.

---

# 9. Why Two Events?

Because their responsibilities differ.

Domain Event

```
Inside Business
```

Integration Event

```
Between Services
```

---

# 10. Domain Event Example

```tsx
class OrderPlaced {

    constructor(
        public orderId: string,
        public customerId: string
    ) {}

}
```

Business meaning only.

---

# 11. Integration Event Example

```json
{
  "event": "ORDER_CREATED",
  "orderId": 100,
  "customerId": 50,
  "createdAt": "..."
}
```

Notice:

Different format.

Different purpose.

May contain version numbers.

Metadata.

Headers.

Tracing IDs.

---

# 12. Do All Domain Events Become Integration Events?

Interview Question.

Answer:

**No.**

---

Example

Customer changes password.

Business event

```
PasswordChanged
```

Does Inventory Service care?

No.

No Kafka.

No Integration Event.

---

Example

OrderPlaced

Payment cares.

Inventory cares.

Shipping cares.

Yes.

Publish Integration Event.

---

# 13. One Domain Event → Many Integration Events

Sometimes.

Example

```
OrderPlaced
```

Produces

```
OrderCreated

↓

InventoryReservationRequested

↓

RecommendationEngineUpdated
```

Possible.

---

# 14. One Integration Event from Multiple Domain Events

Also possible.

Example

Domain

```
OrderConfirmed

↓

PaymentAuthorized

↓

InventoryReserved
```

Only after all three

publish

```
OrderReadyForShipping
```

---

# 15. Where Does Outbox Fit?

This is why we learned Outbox.

Flow

```
Business

↓

Domain Event

↓

Save Aggregate

↓

Save Outbox

↓

Commit Transaction

↓

Outbox Worker

↓

Kafka

↓

Integration Event
```

Notice

Kafka happens later.

Business already committed.

---

# 16. Why Not Publish Kafka Immediately?

You've already learned this.

Dual Write Problem.

Example

```
Save Order

✓

Kafka Publish

✗
```

Other services never know.

Outbox solves this.

---

# 17. Domain Event Lifecycle

```
Aggregate

↓

Business Rule Complete

↓

Raise Domain Event

↓

Application Layer

↓

Outbox

↓

Kafka

↓

Consumer

↓

Other Service
```

---

# 18. Parking Project Example

Parking Aggregate

```
ParkingSession.start()
```

Raises

```
ParkingStarted
```

Application Layer

Stores

```
Outbox
```

Worker

Publishes

```
PARKING_STARTED
```

Billing Service

Consumes

Starts charging.

---

# 19. Another Example

Wallet

Aggregate

```
Wallet.refill()
```

Raises

```
WalletRefilled
```

Inside

Audit

History

Notification

Outside

Kafka

```
WALLET_REFILLED
```

Analytics Service updates reports.

---

# 20. Domain Event Characteristics

Usually

- Rich business meaning
- Inside same service
- Strongly typed classes
- No Kafka dependency

---

# 21. Integration Event Characteristics

Usually

- JSON
- Versioned
- Sent via Kafka/SQS/RabbitMQ
- Contains metadata
- Cross-service communication

---

# 22. Comparison Table

| Domain Event | Integration Event |
| --- | --- |
| Inside one service | Between services |
| Business concept | Communication contract |
| Usually class/object | Usually JSON message |
| No Kafka required | Usually Kafka/RabbitMQ/SQS |
| Immediate | Often via Outbox |
| Not all are published | Published only when other services need them |

---

# 23. Common Interview Questions

### Are Domain Events and Kafka events the same?

No.

Domain Events are internal business events.

Kafka events are Integration Events used for inter-service communication.

---

### Should every Domain Event be published?

No.

Only when another bounded context needs the information.

---

### Why use Outbox?

To reliably convert committed business events into Integration Events without dual-write problems.

---

### Can one Domain Event produce multiple Integration Events?

Yes.

---

### Can Integration Events be versioned?

Yes.

Very common.

---

# 24. Senior Production Flow

```
Aggregate

↓

Raise Domain Event

↓

Application Service

↓

Persist Aggregate

↓

Persist Outbox

↓

Commit

↓

Outbox Worker

↓

Kafka

↓

Integration Event

↓

Consumer

↓

Another Service
```

---

# 25. Senior Developer Notes

```
Domain Event

Purpose:
Represent something important in the business.

Lives:
Inside one bounded context.

Examples:

OrderPlaced

ParkingStarted

WalletRefilled

TripCompleted

Characteristics:

✓ Business language

✓ Internal

✓ No messaging dependency

-----------------------------------

Integration Event

Purpose:
Notify other services.

Lives:
Between bounded contexts.

Examples:

ORDER_CREATED

PARKING_STARTED

PAYMENT_COMPLETED

Characteristics:

✓ Cross-service

✓ Kafka/RabbitMQ/SQS

✓ JSON

✓ Versioned

-----------------------------------

Flow

Aggregate

↓

Domain Event

↓

Outbox

↓

Kafka

↓

Integration Event

↓

Other Services

Rule:

Not every Domain Event becomes an Integration Event.
```

---

# ⭐ Senior Interview Tip (One of the Most Valuable Distinctions)

Imagine you're inside the **Parking Service**.

When a parking session starts:

- The **Domain Event** answers: **"What happened in my business?"**
    - `ParkingStarted`

When that information needs to reach **Billing**, **Analytics**, or **Notifications**:

- The **Integration Event** answers: **"What information do other services need?"**
    - `PARKING_STARTED`

This distinction is subtle but extremely important. It keeps your **domain model independent of your messaging infrastructure**. If you later switch from Kafka to RabbitMQ or SQS, your **Domain Events don't change**—only the mechanism that publishes the **Integration Events** changes.

This separation is a hallmark of well-designed DDD-based microservices.
