# 8. Final Technology-Agnostic Architecture

We've now covered the individual production concerns:

```text
1. Transactions + Concurrency              ✅
2. Consistency + Cache                     ✅
3. Payment Failures + Reconciliation       ✅
4. Saga + Outbox + Kafka                   ✅
5. Scaling + Bottlenecks + HA              ✅
6. Security + Rate Limiting + Reliability  ✅
7. Observability + Disaster Recovery       ✅
```

Now we're going to **put everything together**.

The important goal here is not to draw a huge diagram and memorize it.

The goal is to be able to answer:

> **"Why does every component exist, what responsibility does it have, and what happens when something fails?"**

---

# 8.1 Start With Requirements

Before designing the architecture, an interview answer should start with the requirements.

### Functional

Our parking system needs to support:

```text
Search available parking
        ↓
Select parking
        ↓
Create reservation
        ↓
Make payment
        ↓
Confirm reservation
        ↓
Cancel reservation
        ↓
Refund
        ↓
View reservation history
```

External integrations:

```text
Payment Provider
Parking/Inventory Provider
Notification Provider
```

---

### Non-functional

We've already established the important ones:

```text
High availability
Scalability
No double booking
Strong consistency for booking/payment state
Eventual consistency for search/cache
Fault tolerance
Idempotency
Security
Observability
Disaster recovery
```

And our approximate scale:

```text
100M users
10M DAU

~1M searches/min
~100K reservations/min
~20K payments/min
```

These numbers are important because they influence architecture.

---

# 8.2 High-Level Architecture

Let's first forget specific technologies.

Conceptually:

```text
                         ┌───────────────┐
                         │    Clients    │
                         │ Web / Mobile  │
                         └───────┬───────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │ Traffic Layer │
                         │ LB / Gateway  │
                         └───────┬───────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   Stateless API Layer  │
                    │                         │
                    │ Search                  │
                    │ Reservation             │
                    │ Payment                 │
                    │ Cancellation            │
                    │ User                    │
                    └───────┬─────────┬───────┘
                            │         │
                     ┌──────┘         └──────┐
                     ▼                       ▼
              ┌────────────┐          ┌────────────┐
              │    Cache   │          │  Database  │
              │            │          │            │
              │ Availability│         │ PostgreSQL │
              └────────────┘          └─────┬──────┘
                                            │
                                            ▼
                                      ┌────────────┐
                                      │   Outbox   │
                                      └─────┬──────┘
                                            │
                                            ▼
                                      ┌────────────┐
                                      │ Event Bus  │
                                      │   Kafka    │
                                      └─────┬──────┘
                                            │
                         ┌──────────────────┼─────────────────┐
                         ▼                  ▼                 ▼
                    Notification         Payment          Analytics
                     Consumer            Consumer           Consumer
```

This is the basic architecture.

Now let's understand **why each layer exists**.

---

# 8.3 Client → Traffic Layer

Clients:

```text
Web
Mobile
```

send requests through a traffic-management layer.

For example:

```text
Client
   ↓
Load Balancer / API Gateway
```

Responsibilities can include:

* TLS termination
* routing
* authentication integration
* rate limiting
* WAF/security controls
* traffic distribution
* health checks

We don't want clients directly calling individual application instances.

---

# 8.4 Stateless API Layer

Behind the traffic layer:

```text
             ┌─────────────┐
             │ Node API #1 │
             ├─────────────┤
             │ Node API #2 │
             ├─────────────┤
             │ Node API #3 │
             ├─────────────┤
             │ Node API #N │
             └─────────────┘
```

The API servers are **stateless**.

Meaning:

> Any instance should be capable of handling any request.

We don't keep correctness-critical session/business state only in local memory.

Why?

Because then:

```text
Request 1 → Server A
Request 2 → Server B
```

should still work.

This enables horizontal scaling:

```text
10 instances
      ↓
20 instances
      ↓
50 instances
```

without changing business logic.

---

# 8.5 Where does business state live?

Shared systems:

```text
PostgreSQL
Redis
Kafka
```

rather than:

```text
Node.js instance memory
```

For example:

```text
Reservation state
      ↓
PostgreSQL

Availability read model
      ↓
Redis

Asynchronous events
      ↓
Kafka
```

This is a very important architecture principle:

> **Application instances can be disposable; durable business state cannot.**

---

# 8.6 Redis

Redis primarily serves our **read-heavy availability/search workload**.

Example:

```text
GET /availability
        ↓
      Redis
        ↓
      HIT
        ↓
    Return result
```

This prevents every search from hitting PostgreSQL.

But:

```text
Redis ≠ source of truth
```

For reservation:

```text
User wants slot
      ↓
PostgreSQL
      ↓
transaction + lock
      ↓
reservation
```

not:

```text
Redis says available
      ↓
Book immediately
```

That's one of the most important decisions in our architecture.

---

# 8.7 PostgreSQL

PostgreSQL is our **authoritative transactional system**.

It owns correctness-critical state:

```text
Users
Parking inventory
Reservations
Payments
Refunds
Idempotency records
Outbox events
```

Especially:

```text
Availability / inventory
Reservation state
Payment state
```

The database transaction handles concurrency.

For example:

```text
BEGIN
   ↓
Lock/check inventory
   ↓
Reserve slot
   ↓
Create PENDING_PAYMENT reservation
   ↓
Create Outbox event
   ↓
COMMIT
```

Notice that the external payment provider is **not inside the DB transaction**.

We already established why:

> Never hold a database transaction/row lock while waiting for an external network call.

---

# 8.8 Payment Flow

Let's zoom into one reservation.

```text
Client
  ↓
Create reservation
  ↓
PostgreSQL transaction
  ↓
PENDING_PAYMENT
  ↓
COMMIT
  ↓
Payment service
  ↓
External Payment Provider
```

Possible outcomes:

```text
SUCCESS
FAILED
UNKNOWN
```

### SUCCESS

```text
Payment SUCCESS
      ↓
Confirm reservation
```

### FAILED

```text
Payment FAILED
      ↓
Release inventory
      ↓
Reservation EXPIRED/RELEASED
```

### UNKNOWN

```text
Payment UNKNOWN
      ↓
Persist UNKNOWN
      ↓
Reconciliation
      ↓
Query provider
```

If the provider eventually says SUCCESS but the reservation cannot be fulfilled:

```text
Payment SUCCESS
      +
Reservation unavailable
      ↓
Compensation
      ↓
Refund
```

This is where our previous payment/reconciliation/Saga concepts connect.

---

# 8.9 Outbox

Now imagine:

```text
Reservation confirmed
```

and we need to publish:

```text
ReservationConfirmed
```

We don't want:

```text
DB COMMIT
    ↓
Kafka publish
```

as two unrelated operations.

Because:

```text
DB succeeds
Kafka fails
```

could leave the system inconsistent.

Instead:

```text
BEGIN
   ↓
Update reservation
   ↓
Insert Outbox event
   ↓
COMMIT
```

Then:

```text
Outbox Publisher
       ↓
Kafka
```

If Kafka is temporarily unavailable:

```text
Outbox remains
       ↓
Retry later
```

This gives us a durable DB → event handoff.

---

# 8.10 Kafka

Kafka is our asynchronous event backbone.

Events might include:

```text
ReservationCreated
ReservationConfirmed
ReservationCancelled
PaymentSucceeded
PaymentFailed
RefundCompleted
```

Different consumers can independently process them:

```text
                 Kafka
                   │
       ┌───────────┼────────────┐
       ↓           ↓            ↓
 Notification   Analytics    Cache Update
 Consumer        Consumer      Consumer
```

This gives us:

* decoupling
* asynchronous processing
* buffering
* independent consumer scaling
* replay capability where appropriate

But consumers must be **idempotent** because duplicate processing can occur.

---

# 8.11 Search vs Booking

This distinction should be crystal clear in your architecture explanation.

### Search

```text
Client
 ↓
API
 ↓
Redis
 ↓
Availability result
```

Can tolerate:

```text
eventual consistency
```

### Booking

```text
Client
 ↓
API
 ↓
PostgreSQL
 ↓
transaction
 ↓
lock/check inventory
 ↓
reservation
```

Requires:

```text
strong consistency
```

So:

> **The system deliberately uses different consistency models for different operations.**

We don't need strong consistency everywhere.

That would often make the system more expensive and less scalable than necessary.

---

# 8.12 Scaling the Architecture

Now apply Major Topic 5.

### API

```text
Horizontal scaling
```

### Redis

```text
Replication / clustering
Hot-key protection
```

### PostgreSQL

Start with:

```text
Indexes
Query optimization
Connection-pool control
```

Then potentially:

```text
Read replicas
Partitioning
Sharding
```

when justified.

### Kafka

```text
Partitions
Consumer groups
Horizontal consumers
```

### External providers

```text
Timeouts
Retries
Backoff
Circuit breaker
Concurrency limits
```

---

# 8.13 Failure Handling

This is where your architecture becomes genuinely production-level.

### Node instance fails

```text
LB
 ↓
healthy instances
```

### Redis fails

```text
Redis unavailable
 ↓
controlled fallback
 ↓
PostgreSQL
```

while protecting DB from a cache-miss storm.

### DB primary fails

```text
Primary
   ↓
Standby
   ↓
Failover
```

### Kafka fails

```text
PostgreSQL
 ↓
Outbox
 ↓
Kafka unavailable
 ↓
Retry later
```

### Payment provider fails

```text
Timeout
 ↓
UNKNOWN
 ↓
Reconciliation
```

### Kafka consumer fails

```text
Kafka
 ↓
consumer ❌
 ↓
another consumer/retry
```

#### Your question

> **If a Kafka consumer fails, how does Kafka ensure that the partition's messages are still processed? Do we need multiple consumers for the same partition, or does Kafka automatically reassign the partition to another consumer? What happens if all consumers assigned to that partition are down?**

#### Short answer

Suppose we have:

```text
Topic
 ├── Partition 0
 ├── Partition 1
 └── Partition 2

Consumer Group
 ├── Consumer A
 ├── Consumer B
 └── Consumer C
```

**Normally, one partition is assigned to only one consumer within the same consumer group.**

So you **don't have multiple active consumers processing the same partition simultaneously**.

If:

```text
Partition 1 → Consumer B ❌
```

Kafka detects that Consumer B is dead through the consumer group's heartbeat/rebalancing mechanism.

Then Kafka can **reassign Partition 1 to another healthy consumer in the same consumer group**:

```text
Partition 1
     ↓
Consumer B ❌
     ↓
Rebalance
     ↓
Consumer A
```

The new consumer continues from the **last committed offset**.

#### What if all consumers are down?

```text
Partition 1
     ↓
Consumer A ❌
Consumer B ❌
Consumer C ❌
```

The messages remain in Kafka according to retention.

When a consumer becomes available again, it can continue processing from the committed offset.

So **manual intervention is not normally required just because all consumers are temporarily down**.

However, if consumers are continuously crashing, Kafka lag will grow and monitoring/alerts should notify engineers. **Manual intervention may then be needed to investigate the underlying problem.**

#### One important limitation

If you have:

```text
3 partitions
3 consumers
```

and one consumer dies, another consumer can take its partition.

But if you have:

```text
1 partition
3 consumers
```

only **one consumer can actively consume that partition at a time**. The other two are idle, so they don't provide parallel processing for that single partition.

**Key note:**

> **Kafka provides automatic partition reassignment within a consumer group when a consumer fails. Multiple consumers provide failover and parallelism across partitions, but a single partition is processed by only one consumer in a consumer group at a time.**



### Entire region fails

```text
DR strategy
 ↓
secondary environment
 ↓
restore/failover
```

based on our RPO/RTO requirements.

---

# 8.14 Security Layer

We shouldn't treat security as one component.

It's cross-cutting:

```text
                Security
                   │
     ┌─────────────┼──────────────┐
     ↓             ↓              ↓
Authentication  Authorization  Validation
     ↓             ↓              ↓
TLS          Least privilege    Input safety
     ↓
Rate limiting
     ↓
Secrets
     ↓
Audit/security logs
```

For payments:

```text
Idempotency
Webhook verification
No sensitive payment data in logs
```

---

# 8.15 Observability Layer

Observability also cuts across the architecture.

We collect:

```text
Logs
Metrics
Traces
```

with:

```text
request_id
trace_id
reservation_id
payment_id
event_id
```

Then:

```text
Metrics
   ↓
Detect problem

Tracing
   ↓
Locate problem

Logs
   ↓
Understand problem
```

and:

```text
Alerts
   ↓
Human action
```

---

# 8.16 The Complete Mental Architecture

Now compress the whole design into one picture:

```text
                         CLIENTS
                    Web / Mobile
                           │
                           ▼
                ┌──────────────────┐
                │ Gateway / LB / WAF│
                │ Auth / Rate Limit │
                └─────────┬────────┘
                          │
                          ▼
             ┌─────────────────────────┐
             │   Stateless API Layer   │
             │                         │
             │ Search / Reservation    │
             │ Payment / Cancellation  │
             │ User / History          │
             └───────┬─────────┬───────┘
                     │         │
                     │         │
                     ▼         ▼
               ┌─────────┐  ┌─────────────┐
               │  Redis  │  │ PostgreSQL  │
               │         │  │             │
               │ Search/ │  │ Source of   │
               │ Cache   │  │ Truth       │
               └─────────┘  └──────┬──────┘
                                   │
                              Transaction
                                   │
                                   ▼
                              ┌─────────┐
                              │ Outbox  │
                              └────┬────┘
                                   │
                                   ▼
                              ┌─────────┐
                              │  Kafka  │
                              └────┬────┘
                                   │
                ┌──────────────────┼──────────────────┐
                ▼                  ▼                  ▼
          Notification         Analytics         Cache/read
           Consumer             Consumer            Consumer

                External Payment Provider
                         ▲
                         │
                  Payment Service
                         │
              timeout / retry / idempotency
                         │
                  reconciliation
                         │
                    compensation

        ──────────────────────────────────────────────
                  Cross-cutting concerns

       Security | Observability | HA | DR | Monitoring
```

---

# 8.17 The Most Important Architectural Principle

If an interviewer asks:

> "Explain your architecture."

Don't just list:

> "I use Redis, PostgreSQL, Kafka..."

Instead explain **responsibilities**:

> **PostgreSQL owns transactional correctness. Redis accelerates read-heavy availability queries but is not authoritative. Kafka decouples asynchronous processing. Outbox guarantees reliable handoff from database state changes to events. Stateless API instances provide horizontal scalability. External payment calls are protected with idempotency, timeouts, retries, reconciliation, and compensation. Observability and security are cross-cutting concerns.**

That explanation demonstrates architectural reasoning rather than technology memorization.

---

# 8.18 One Final Check: Does Every Component Have a Reason?

This is how I want you to think in interviews:

| Component             | Why do we need it?                    |
| --------------------- | ------------------------------------- |
| Load Balancer         | Distribute traffic / failover         |
| Stateless APIs        | Horizontal scaling                    |
| Redis                 | Fast read-heavy availability          |
| PostgreSQL            | Transactional source of truth         |
| DB transactions/locks | Prevent double booking                |
| Outbox                | Reliable DB → event handoff           |
| Kafka                 | Async decoupling/buffering            |
| Consumers             | Background processing                 |
| Idempotency           | Safe retries/duplicates               |
| Reconciliation        | Resolve unknown external outcomes     |
| Rate limiting         | Protect system from excessive traffic |
| Circuit breaker       | Protect against failing dependencies  |
| Observability         | Detect/investigate problems           |
| HA                    | Survive component failures            |
| DR                    | Recover from major disasters          |

If we cannot explain **why** a component exists, we should question whether we actually need it.

---

## 8.1 Final Architecture — COMPLETE

At this point, we've completed the **technology-agnostic architecture**.

### Next: 9. AWS Mapping

Now we'll take this exact architecture and ask:

> **"If I had to build this on AWS, which AWS services would I choose, what alternatives exist, and what are the trade-offs?"**

We'll map things like:

```text
Traffic layer       → ?
Node.js API         → ?
PostgreSQL          → ?
Redis               → ?
Kafka               → ?
Object storage      → ?
Secrets             → ?
Monitoring          → ?
DR                  → ?
```

And importantly, we won't just say **"use AWS X."** We'll discuss **why, alternatives, limitations, and when you'd choose something else**.

