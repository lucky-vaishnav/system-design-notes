 **top-level interview approach** from everything we learned.

You don't need another 20 sections of notes. The purpose should be: **“How do I actually approach a system-design interview using everything I have learned?”**

# End-to-End System Design Interview Approach

### Goal

You have now learned the individual pieces:

> concurrency → consistency → cache → payments → Saga/Outbox/Kafka → scaling → security → reliability → observability → DR → AWS

In an actual interview, **you should NOT explain all of these one by one**.

Instead, you should build the design progressively and bring a topic in **only when it becomes relevant**.

---

# 11.1 The Overall Interview Flow

Think of your interview as roughly these phases:

```text
1. Clarify requirements
        ↓
2. Define scale / assumptions
        ↓
3. Identify core entities
        ↓
4. Draw simple high-level architecture
        ↓
5. Explain the main user flow
        ↓
6. Deep dive into the hardest problems
        ↓
7. Discuss scalability
        ↓
8. Discuss failures / reliability
        ↓
9. Security + observability
        ↓
10. Summarize trade-offs
```

That's your **mental framework**.

You don't start by saying:

> "I'll use Redis, Kafka, PostgreSQL, Outbox, Saga..."

Instead, start with the **problem**.

---

# 11.2 Phase 1 — Clarify Requirements

Before designing, ask questions.

For our parking system:

### Functional

You could ask:

> "Should users only search and reserve parking, or do we also need cancellation, refunds and reservation history?"

> "Do we need to support external payment providers?"

> "Is payment required immediately during reservation?"

### Non-functional

Then ask:

> "What scale are we expecting?"

> "What availability and latency requirements do we have?"

> "Is preventing double booking a strict requirement?"

> "Do we need multi-region support?"

You don't need to ask 20 questions.

Ask the **few questions that can materially change the architecture**.

### Interview principle

> **Ask questions that change your design, not questions just to show that you know questions.**

---

# 11.3 Phase 2 — Establish Scale

You don't need exact numbers if the interviewer doesn't provide them.

For example:

> "I'll assume roughly 100M registered users, 10M DAU, around 1M searches per minute, 100K reservations per minute and 20K payments per minute."

Then immediately identify:

> "The important observation is that searches are much more read-heavy, while reservations and payments are correctness-sensitive writes."

That observation already starts guiding your architecture.

---

# 11.4 Phase 3 — Identify the Core Data

Keep this short.

For our system:

```text
User
ParkingLot
ParkingSlot / Inventory
Reservation
Payment
Refund
IdempotencyRecord
```

Then identify the most important one:

> "Reservation and inventory are the critical transactional data because we must prevent double booking."

That's enough.

Don't explain every table unless asked.

---

# 11.5 Phase 4 — Draw the First Architecture

Initially keep it **simple**.

```text
              Clients
                 |
             Load Balancer
                 |
        Stateless API Servers
           /             \
       Redis          PostgreSQL
                          |
                       Outbox
                          |
                        Kafka
                     /    |    \
              Payment  Notification Analytics
```

Then say:

> "At a high level, I'll keep the API layer stateless so we can horizontally scale it. Redis will handle read-heavy availability/search traffic, while PostgreSQL will remain the source of truth for reservations and inventory."

Stop there.

**Don't immediately explain Kafka, Outbox, retries, DR, security, etc.**

You have created the skeleton.

---

# 11.6 Phase 5 — Explain One Main Flow

Now choose the most important flow.

For this problem:

> **Search → Reserve → Pay → Confirm**

This is where you demonstrate that your architecture actually works.

### Search

```text
Client
  ↓
API
  ↓
Redis
  ↓
Availability
```

Briefly:

> "Search is read-heavy, so I'll serve availability primarily from Redis. It can be eventually consistent because it's only a view of availability."

---

### Reservation

Then say:

> "However, Redis cannot authorize the reservation because stale cache could cause double booking."

Then:

```text
API
 ↓
PostgreSQL transaction
 ↓
Lock/check inventory
 ↓
Reserve inventory
 ↓
Create PENDING_PAYMENT reservation
 ↓
Commit
```

And explain the key decision:

> "PostgreSQL is the authority for the reservation decision. I hold the inventory temporarily using PENDING_PAYMENT with an expiry."

That's one of the **most important statements in the whole design**.

---

### Payment

Then:

```text
Reservation committed
        ↓
Payment Provider
        ↓
SUCCESS / FAILED / UNKNOWN
```

Say:

> "I don't keep the database transaction open while calling the external payment provider because that would hold database locks during network latency."

Excellent senior-level point.

Then:

> "If the provider times out, I treat the result as UNKNOWN rather than assuming failure, and reconcile with the provider using an idempotency key."

That's enough.

---

### Confirmation

Then:

```text
DB transaction
     ↓
Outbox
     ↓
Kafka
     ↓
Consumers
```

Say:

> "Once the transactional state changes, the Outbox guarantees that the corresponding event isn't lost if Kafka is temporarily unavailable."

Again, don't give the whole Outbox lecture unless asked.

---

# 11.7 Phase 6 — Now Discuss the Hard Parts

This is where the interviewer will usually start asking:

> "What happens if two users reserve the same slot?"

You go into concurrency.

> "The inventory row is protected using a database transaction and row-level locking. Concurrent requests targeting the same inventory wait or fail according to our timeout policy, so only one transaction can successfully reserve it."

Then perhaps:

> "For very high contention on a hot resource, the database row becomes the serialization point, so adding more Node.js instances doesn't eliminate that bottleneck."

That demonstrates deeper understanding.

---

Then the interviewer may ask:

> "What if payment fails?"

You discuss:

```text
PENDING_PAYMENT
      ↓
Payment failed
      ↓
Release inventory
      ↓
EXPIRED / RELEASED
```

If payment is unknown:

```text
UNKNOWN
   ↓
Reconciliation
   ↓
SUCCESS / FAILED
```

You already know all this. **You don't need to reteach it.**

---

# 11.8 Phase 7 — Scaling

Now ask yourself:

> **"Where will this system break first?"**

Don't randomly say "use Kafka and sharding."

Walk through the bottlenecks.

### Search

Redis → cache hit ratio → hot keys → DB fallback.

### Reservation

PostgreSQL → connections → locks → hot inventory rows.

### Payment

External provider → latency → timeout → retries.

### Kafka

Partitions → consumer throughput → lag → downstream capacity.

Then explain:

> "Scaling is iterative. After solving one bottleneck, another component may become the bottleneck."

This is a very good senior-level statement.

---

# 11.9 Phase 8 — Failure Scenarios

Don't list every failure you've learned.

Pick the **important failures**.

For example:

| Failure            | Response                                                     |
| ------------------ | ------------------------------------------------------------ |
| Node instance dies | LB routes to healthy instances                               |
| Redis unavailable  | Controlled DB fallback + protection against cache-miss storm |
| PostgreSQL failure | HA/failover + reconcile uncertain operations                 |
| Payment timeout    | UNKNOWN + reconciliation                                     |
| Kafka unavailable  | Outbox retains events                                        |
| Consumer crashes   | Kafka reassigns partition                                    |
| Region failure     | DR strategy based on RPO/RTO                                 |

You can expand any row if the interviewer asks.

---

# 11.10 Phase 9 — Security & Observability

Keep this very short initially.

Say:

> "For security, I'd use authentication, resource-level authorization, input validation, rate limiting, TLS, secrets management and least-privilege service credentials."

Then:

> "For observability, I'd use structured logs, metrics and distributed tracing, with correlation IDs such as request ID, reservation ID and payment ID."

That's enough unless they ask for details.

---

# 11.11 Phase 10 — Trade-offs

This is where you demonstrate seniority.

Don't just say:

> "I use Redis."

Say:

> "I'm using Redis because availability is read-heavy, but I'm deliberately not making Redis the source of truth because stale cache cannot be allowed to determine whether a reservation is valid."

Similarly:

> "I'm using Kafka for asynchronous event processing because notification, analytics and other consumers shouldn't block the transactional reservation path."

And:

> "I'm not using sharding initially because PostgreSQL optimization, replicas, partitioning and proper indexing may be sufficient. I'd introduce sharding only if a single database becomes the actual bottleneck."

This is much stronger than simply listing technologies.

---

# 11.12 How Much Should You Actually Speak?

This is important for you.

You **cannot and should not explain all 20–25 sections** in an interview.

Think:

### Initial explanation

**5–10 minutes**

Give:

```text
Requirements
   ↓
Scale
   ↓
High-level architecture
   ↓
Main flow
   ↓
2–3 critical design decisions
```

Then let the interviewer drive deeper.

### Deep dive

If they ask about concurrency:

→ discuss locks, optimistic locking, hot rows.

If they ask about payment:

→ discuss idempotency, UNKNOWN, reconciliation.

If they ask about scaling:

→ discuss DB, Redis, Kafka, bottlenecks.

If they ask about failures:

→ discuss HA, retries, circuit breakers, Outbox, DR.

**You don't dump your entire knowledge at once.**

---

# 11.13 The Most Important Mental Model

For your next system-design problem, remember this:

```text
             REQUIREMENTS
                  ↓
               SCALE
                  ↓
         HIGH-LEVEL ARCHITECTURE
                  ↓
             MAIN USER FLOW
                  ↓
       ┌──────────┼──────────┐
       ↓          ↓          ↓
   Correctness  Scale     Reliability
       ↓          ↓          ↓
  Consistency   Bottlenecks Failures
  Concurrency   Caching     Retries
  Transactions  DB scale    Idempotency
                             Recovery
                  ↓
          Security / Observability
                  ↓
              Trade-offs
```

You don't need to memorize 25 topics.

You need to know **where each topic belongs when a problem appears**.

---

# 11.14 What We Learned From This Parking Problem

This is probably the most valuable takeaway from this first problem:

> **A system-design interview is not a checklist of technologies. It is a sequence of decisions driven by requirements, scale, correctness, bottlenecks and failure scenarios.**

For example:

```text
Need fast search
      ↓
Redis

Need no double booking
      ↓
PostgreSQL + transaction + locking

Need external payment
      ↓
Idempotency + UNKNOWN + reconciliation

Need reliable async events
      ↓
Outbox + Kafka

Need horizontal API scaling
      ↓
Stateless APIs + Load Balancer

Need protection during overload
      ↓
Rate limiting + backpressure + load shedding

Need failure recovery
      ↓
HA + retries + circuit breakers + DR
```

**That is the reusable system-design skill.**
