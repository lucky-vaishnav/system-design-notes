(Context Mapping, Anti-Corruption Layer, Shared Kernel, Open Host Service, etc.)

This is the **highest level of DDD** and one of the topics that differentiates a **Senior Developer** from a **Staff Engineer / Software Architect**.

Everything we've learned so far (Entities, Aggregates, Repositories, Domain Events) is called **Tactical DDD**.

Now we're moving to:

# Strategic DDD

This is about **how multiple systems and teams work together**, not how to write one class.

---

# DDD Learning Path

```
DDD

├── Tactical DDD ✅
│   ├── Entities
│   ├── Value Objects
│   ├── Aggregates
│   ├── Repositories
│   ├── Domain Events
│
└── Strategic DDD  ← Today
    ├── Bounded Context
    ├── Context Mapping
    ├── Anti-Corruption Layer
    ├── Shared Kernel
    ├── Open Host Service
    ├── Published Language
    ├── Conformist
    ├── Customer/Supplier
```

Notice:

Everything today is **system-to-system** design.

---

# Before We Start

One sentence explains Strategic DDD:

> **How should different business domains communicate without becoming tightly coupled?**
> 

---

# 1. Why Strategic DDD Exists

Imagine Amazon.

Do you think every team shares one huge codebase?

No.

There are teams for:

```
Orders

Inventory

Payments

Shipping

Recommendations

Notifications

Returns

Analytics
```

Each team builds software differently.

Problem:

How do they communicate?

---

# Without Strategic DDD

Every service imports every other service's model.

```
Order

↓

Inventory

↓

Payment

↓

Shipping

↓

Analytics
```

Eventually:

Everything depends on everything.

A nightmare.

---

Strategic DDD says:

Separate the business into independent **Bounded Contexts**.

---

# 2. Bounded Context (Deep Version)

You already know the basic definition.

Now let's learn the architect's definition.

## Definition

A Bounded Context is:

> A boundary inside which one business model has exactly one meaning.
> 

---

Example

Word:

```
Customer
```

In Billing

Customer means

```
Billing Account
```

In Marketing

Customer means

```
Email Subscriber
```

In Support

Customer means

```
Support Ticket Owner
```

Same English word.

Three models.

Three bounded contexts.

---

# Parking Example

Your project might have:

```
Parking Context

↓

ParkingSession

Permit

Vehicle
```

---

Payment Context

```
Wallet

Refund

Invoice
```

---

User Context

```
Profile

Role

Authentication
```

Each context owns its own language.

---

# 3. Why Not One Giant Model?

Suppose Payment needs:

```
Wallet

Balance
```

Parking needs:

```
ParkingSession
```

Don't create

```
UserEverything
```

Shared by everyone.

Eventually

```
User.js

6000 lines
```

DDD avoids this.

---

# 4. Context Map

Now we have multiple contexts.

How do they connect?

Answer:

Context Map.

---

Visual

```
Parking

↓

Billing

↓

Notification

↓

Analytics
```

This map is called the Context Map.

It shows:

- who owns data
- who depends on whom
- how communication happens

---

# 5. Example Context Map

```
          User

        /     \

Parking ---- Billing

      \

 Notifications

        \

     Analytics
```

Every arrow represents a relationship.

---

# 6. Context Relationships

This is the real heart of Strategic DDD.

There are several relationship types.

We'll learn the important ones.

---

# 7. Anti-Corruption Layer (ACL)

This is probably the most famous DDD concept.

## Definition

An Anti-Corruption Layer protects your model from another model.

---

Example

Old Legacy System

returns

```json
{
   "usr_nm":"Lucky",
   "tp":"A"
}
```

Your service expects

```json
{
   "name":"Lucky",
   "role":"Admin"
}
```

Don't spread legacy fields everywhere.

Instead

```
Legacy

↓

ACL

↓

Your Model
```

---

Example

```tsx
LegacyUser

↓

ACL

↓

User
```

The translator lives inside ACL.

---

Why?

If Legacy changes,

only ACL changes.

Business remains clean.

---

# 8. Parking Example

Suppose Uber sends

```json
{
  "pickup_lat":12,
  "pickup_lon":45
}
```

Your model

```tsx
Location

latitude

longitude
```

Don't use Uber model directly.

Create

```
Uber Adapter

↓

ACL

↓

Location
```

Exactly the same idea.

---

# 9. Shared Kernel

Sometimes two teams genuinely share business logic.

Example

Parking

Billing

Both need

```
Money
```

Instead of duplicating

Create

```
Shared Kernel

↓

Money

Currency
```

Shared by both.

---

Rule

Keep Shared Kernel very small.

Otherwise

everything becomes shared.

---

# 10. Open Host Service (OHS)

Suppose many systems need your API.

Instead of

Custom API

for each client

Create

One standard interface.

Example

```
Payment API

↓

REST

↓

Everyone
```

or

```
gRPC

↓

Everyone
```

This public interface is Open Host Service.

---

# 11. Published Language

Suppose your service publishes Kafka events.

Don't expose internal entities.

Instead

Publish

```json
{
   "event":"ORDER_CREATED"
}
```

Stable.

Versioned.

Documented.

That's Published Language.

---

# 12. Customer / Supplier

Suppose

Billing

depends on

Parking.

Parking Team changes API.

Billing Team breaks.

Here

Parking

is

Supplier.

Billing

is

Customer.

Both teams coordinate.

---

# 13. Conformist

Sometimes

One system is so powerful

everyone simply adapts.

Example

AWS

Stripe

Salesforce

You don't ask Stripe to change.

You conform.

This relationship is called

Conformist.

---

# 14. Separate Ways

Sometimes

Two contexts have nothing in common.

Don't integrate.

Keep separate.

Very common.

---

# 15. Partnership

Sometimes

Two teams own one feature together.

They coordinate equally.

Example

Parking

Billing

working together on

Subscriptions.

---

# 16. Parking Project Example

Bounded Contexts

```
Parking

↓

Payment

↓

Users

↓

Notifications

↓

Reporting
```

Relationships

Parking

↓

Billing

Customer/Supplier

---

Parking

↓

Notification

Kafka

Published Language

---

Parking

↓

Uber

ACL

---

Parking

↓

Money Library

Shared Kernel

---

# 17. Strategic DDD + Microservices

Notice something.

Microservice

does NOT equal

Bounded Context.

Usually

One Bounded Context

↓

One Microservice

But

Sometimes

Large contexts

↓

Multiple services.

---

# 18. Strategic DDD + Kafka

Parking

↓

ParkingStarted

↓

Published Language

↓

Kafka

↓

Billing

Billing never sees

ParkingSession Entity.

Only event.

Huge difference.

---

# 19. Strategic DDD + Outbox

Aggregate

↓

Domain Event

↓

Outbox

↓

Published Language

↓

Kafka

↓

Another Context

Everything now fits together.

---

# 20. Common Interview Questions

### What is Strategic DDD?

Designing relationships between business domains.

---

### What is Tactical DDD?

Designing classes inside one domain.

---

### What is an ACL?

A translation layer protecting your model from external models.

---

### Why use Shared Kernel?

To share truly common business concepts.

---

### What is Published Language?

Stable contract shared with other contexts.

---

### What is Open Host Service?

A standard public interface.

---

### Should services share Entities?

No.

Only contracts.

---

# 21. Real Production Flow

```
Parking Context

↓

ParkingStarted

↓

Domain Event

↓

Outbox

↓

Published Language

↓

Kafka

↓

Billing Context

↓

ACL

↓

Billing Model
```

Notice

Billing never imports Parking classes.

---

# 22. Senior Developer Notes

```
Strategic DDD

Goal:
Manage relationships between business domains.

Core Concepts:

1.
Bounded Context

One business meaning.

2.
Context Map

Relationship diagram.

3.
Anti-Corruption Layer

Translate external models.

4.
Shared Kernel

Small shared code.

5.
Open Host Service

Public standard interface.

6.
Published Language

Stable events/API.

7.
Customer/Supplier

Coordinated dependency.

8.
Conformist

Adapt to stronger upstream.

9.
Separate Ways

No integration.

10.
Partnership

Joint ownership.

Rule:

Share contracts,

Never share internal models.
```

---

# 🎯 The Entire DDD Picture (This Is What Senior Interviews Expect)

```
Business Problem
        │
        ▼
Bounded Contexts
        │
        ▼
Strategic DDD
(Context Map, ACL, Published Language)
        │
        ▼
Microservices
        │
        ▼
Tactical DDD
(Entities, Aggregates, Value Objects)
        │
        ▼
Domain Events
        │
        ▼
Outbox
        │
        ▼
Integration Events
        │
        ▼
Kafka / RabbitMQ / SQS
```

This is the complete flow used in many modern enterprise systems.

---

## 🎉 Congratulations!

With this lesson, you've completed the core DDD curriculum at a senior level:

- ✅ Tactical DDD
- ✅ Aggregates & Aggregate Design
- ✅ Domain Events vs Integration Events
- ✅ Strategic DDD

From here, you'll find that advanced system design case studies (Uber, Amazon, Banking, Parking, Ride Sharing, etc.) become much easier because you now understand **both the code-level and system-level modeling principles** behind them.
