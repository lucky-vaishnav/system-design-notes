This is probably the **most asked microservices interview topic**.

A senior developer should know:

- Why services communicate
- Different communication styles
- When to use each
- Trade-offs
- Production examples
- Failure handling

---

# 1. Why Do Microservices Need Communication?

Suppose we have an e-commerce system.

```
User Service

Order Service

Payment Service

Inventory Service

Notification Service
```

Customer places an order.

Question:

Can Order Service complete everything alone?

No.

It needs:

```
Inventory

↓

Payment

↓

Notification
```

Therefore services must communicate.

---

# 2. Two Types of Communication

Everything falls into two categories.

```
Microservice Communication

├── Synchronous

└── Asynchronous
```

This is the first thing interviewers expect.

---

# 3. Synchronous Communication

Means:

```
I send request.

I wait.

I get response.
```

Like calling someone.

```
"Are you home?"

(wait...)

"Yes."
```

---

Example

Order Service

↓

calls

↓

Payment Service

↓

gets response

↓

continues

---

Typical technologies

```
REST API

gRPC

GraphQL
```

---

# Flow

```
Order Service

↓

HTTP

↓

Payment Service

↓

Response

↓

Continue
```

Everything waits.

---

# Pros

```
Simple

Easy debugging

Immediate response
```

---

# Cons

```
Tight coupling

Timeouts

Retries

Service dependency
```

---

# 4. Asynchronous Communication

Instead of waiting...

Publish message.

Leave.

Someone else handles it later.

Example

Order Service

↓

Kafka

↓

Email Service

↓

Analytics

↓

Inventory

↓

Notification

Order Service never waits.

---

# Flow

```
Order Service

↓

Kafka

↓

Consumer

↓

Done
```

---

Technologies

```
Kafka

RabbitMQ

Amazon SQS

SNS

EventBridge
```

---

# Pros

```
Loose coupling

Scalable

Independent

Fault tolerant
```

---

# Cons

```
Eventually consistent

Harder debugging

Need retries

Need DLQ

Need idempotency
```

You've already learned all these.

---

# 5. REST Communication

The most common.

Example

```
POST /payments
```

Order Service

↓

HTTP

↓

Payment Service

↓

JSON Response

---

Node.js

```
awaitaxios.post(
"/payments",
paymentData
);
```

Simple.

---

Good for

```
CRUD

Internal APIs

External APIs
```

---

# 6. gRPC

We'll learn deeply next.

Idea:

Instead of JSON

↓

Binary Protocol

Much faster.

Google created it.

---

# 7. Event Communication

You've already learned this.

Order Service

↓

Publish

```
OrderPlaced
```

↓

Consumers react.

No waiting.

---

# 8. Request/Response vs Event

Very important.

Need immediate answer?

```
REST

gRPC
```

Need notification?

```
Kafka

RabbitMQ
```

---

# Example

User Login

Need response immediately.

Use

```
REST
```

---

User Registered

Need

```
Email

Analytics

CRM

Recommendations
```

Use

```
Events
```

---

# 9. Fan-Out

One request

↓

Many services.

```
OrderPlaced

↓

Email

↓

Analytics

↓

Warehouse

↓

Fraud

↓

Billing
```

Perfect for events.

---

# 10. Chaining

Bad

```
A

↓

B

↓

C

↓

D

↓

E
```

Every service waits.

Latency increases.

---

Better

```
A

↓

Kafka

↓

B

C

D
```

Parallel.

---

# 11. Which Style Should I Choose?

| Situation | Best Choice |
| --- | --- |
| Login | REST |
| Payment Authorization | REST / gRPC |
| Search | REST |
| Dashboard | REST |
| Notifications | Kafka |
| Analytics | Kafka |
| Audit Logs | Kafka |
| Emails | Kafka |

---

# 12. Production Example

Uber

Book Ride

Need response immediately.

```
Passenger

↓

Ride Service

↓

Driver Assigned
```

REST.

---

After booking

Need

```
Analytics

Email

Billing

Notifications
```

Events.

Both together.

---

# 13. Netflix Example

Watch movie.

Need

```
Start streaming
```

Immediately.

REST/gRPC.

---

After

```
Recommendation

Analytics

Continue Watching

History
```

Events.

---

# 14. Amazon Example

Checkout

↓

REST

↓

Payment

↓

Inventory

↓

Success

↓

Publish

```
OrderPlaced
```

↓

Shipping

↓

Email

↓

Warehouse

---

# 15. Common Mistake

Everything REST.

```
Order

↓

Inventory

↓

Payment

↓

Notification

↓

Analytics
```

Every service waits.

Terrible scaling.

---

Everything Kafka.

Also bad.

Sometimes you need immediate answer.

Balance is important.

---

# 16. Decision Tree (Interview Gold)

Ask:

### Do I need immediate response?

Yes

↓

REST / gRPC

---

No

↓

Event

---

Can failure be delayed?

Yes

↓

Kafka

---

No

↓

REST

---

# 17. Senior Design Principle

A common pattern in production is:

```
Critical Path

↓

REST / gRPC

Everything Else

↓

Events
```

Example

Checkout

Critical

```
Payment

Inventory
```

Need success now.

---

Non-critical

```
Email

Analytics

Recommendation

Audit

CRM
```

Can happen later.

---

# 18. How Everything Fits Together

```
REST
        ↓
Fast synchronous calls

gRPC
        ↓
High-performance synchronous calls

Kafka
        ↓
Asynchronous events

Saga
        ↓
Distributed business transaction

Outbox
        ↓
Reliable event publishing

Circuit Breaker
        ↓
Protect failed services

Bulkhead
        ↓
Prevent cascading failures

Service Mesh
        ↓
Manage communication automatically
```

Notice how the remaining topics naturally build on this foundation.

---

# Interview Notes

```
Microservices Communication

Two Types

1. Synchronous
   - REST
   - gRPC

2. Asynchronous
   - Kafka
   - RabbitMQ
   - SQS

REST
- Immediate response
- CRUD
- Tight coupling

gRPC
- Immediate response
- Binary
- High performance

Events
- Loose coupling
- Eventually consistent
- Scalable

Use REST/gRPC when:
- User is waiting
- Need immediate result

Use Events when:
- Notification
- Analytics
- Email
- Audit
- Background work

Production Pattern:
Critical path → REST/gRPC
Non-critical → Events
```

---

## Next Topic

The best next topic is **gRPC**.
