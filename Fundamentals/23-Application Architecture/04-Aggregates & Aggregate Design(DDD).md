This is **widely considered the hardest topic in DDD**, and it's one of the most common reasons developers fail senior system design interviews.

Why?

Because almost everyone understands:

- Entity ✅
- Value Object ✅
- Repository ✅

Repositories are objects that provide an interface for domain objects to access persistent storage (such as databases or file systems). They are responsible for retrieving and storing domain objects like entities, value objects, and aggregates.

A key characteristic of repositories is the **abstraction of persistence**. This means that the users of the repository shouldn’t need to know whether the data is stored in a CSV, a database, or a file. The repository handles the details, providing a consistent way to interact with the domain objects regardless of the underlying storage mechanism.

But when asked:

> **"How do you design Aggregates?"**
> 

many developers struggle.

---

# DDD Advanced Series

```
Part 1 ✅ DDD Basics
Part 2 → Aggregate & Aggregate Design (Today)
Part 3 → Domain Events vs Integration Events
Part 4 → Strategic DDD
```

---

# Before We Start

## One Important Rule

Most people define an Aggregate like this:

> "A group of related entities."
> 

That definition is **incomplete**.

The correct definition is:

> **An Aggregate is a consistency boundary.**
> 

Everything else follows from this.

---

# 1. Why Do Aggregates Exist?

Imagine you're building an e-commerce system.

Database

```
Orders

OrderItems

Payments

Customers

Inventory
```

Looks normal.

Now imagine two users update the same order at the same time.

User A

```
Change Quantity
```

User B

```
Delete Item
```

Without proper rules,

your order can become inconsistent.

Example

```
Order Total = $100

Items Total = $120
```

Impossible.

DDD says:

We need something that guarantees consistency.

That is an Aggregate.

---

# 2. What is an Aggregate?

## Definition

An Aggregate is:

> A cluster of related objects that are treated as **one unit for consistency and transactions.**
> 

Notice the important words:

- One Unit
- Consistency
- Transaction

---

# Example

Order

contains

```
Order

↓

OrderItems

↓

ShippingAddress
```

Business Rule

```
Order Total

=

Sum(Order Items)
```

This rule must **always** be true.

Therefore

Order + OrderItems

belong in one Aggregate.

---

# 3. Aggregate Root

Every Aggregate has exactly **one entry point**.

Called:

## Aggregate Root

Example

```
Order
```

is the Aggregate Root.

OrderItems

are inside.

Outside code should never directly change OrderItems.

---

Wrong

```jsx
orderItem.quantity = 5;
```

Correct

```jsx
order.changeItemQuantity(itemId, 5);
```

Why?

Because Order must recalculate:

```
Total

Tax

Discount

Validation
```

The root protects business rules.

---

# Visual

```
                Aggregate

          +------------------+

          |      Order       |   ← Root

          |------------------|

          | OrderItem        |

          | OrderItem        |

          | Address          |

          +------------------+
```

Outside world

↓

Only talks to

```
Order
```

---

# 4. Why Can't We Modify Child Entities Directly?

Suppose

```
Order

Total = $200
```

Someone does

```jsx
orderItem.price = 50;
```

Now

Order Total

is wrong.

Instead

```jsx
order.changeItemPrice(...)
```

Order immediately updates:

```
Total

Tax

Discount

Coupons
```

Consistency maintained.

---

# 5. Aggregate = Transaction Boundary

This is the biggest interview point.

Everything inside an Aggregate should be updated in **one database transaction**.

Example

```
BEGIN

↓

Order

↓

OrderItems

↓

COMMIT
```

One transaction.

---

But

Inventory

should NOT be inside.

Payment

should NOT be inside.

Those are different Aggregates.

---

# 6. Bad Aggregate Design

Some developers make one giant Aggregate.

Example

```
Order

↓

OrderItems

↓

Inventory

↓

Customer

↓

Payment

↓

Shipment

↓

Notification
```

Huge mistake.

Every change locks everything.

Performance dies.

---

# Good Design

```
Order Aggregate

Order

OrderItems
```

Separate

```
Inventory Aggregate

Stock
```

Separate

```
Payment Aggregate

Payment
```

Each has its own transaction.

---

# 7. Rule

**One Aggregate = One Transaction**

Never try to update multiple Aggregates in one database transaction.

If multiple Aggregates must work together,

use

```
Domain Events

or

Saga
```

---

# 8. Aggregate Size

Common mistake

Bigger Aggregate

↓

Safer?

No.

Actually

Bigger Aggregate

↓

More Locking

↓

Less Concurrency

↓

Poor Performance

DDD recommends

**Small Aggregates**

---

# 9. Example

Parking Project

Question

Should User and Parking Session belong together?

No.

Because

Parking Session changes frequently.

User rarely changes.

Separate Aggregates.

---

Parking Aggregate

```
ParkingSession

↓

ParkingLeg
```

User Aggregate

```
User

↓

Profile
```

Payment Aggregate

```
Wallet

↓

Transactions
```

---

# 10. Aggregate Reference

Another common mistake.

Wrong

```
Order

↓

Customer Object
```

Huge object graph.

Correct

```
Order

↓

customerId
```

Just store IDs.

Not objects.

---

Example

Wrong

```jsx
order.customer.address.city
```

Correct

```jsx
order.customerId
```

Need Customer?

Load separately.

---

# 11. Aggregate Invariant

Interview keyword.

An Invariant is:

A business rule that must always remain true.

Example

```
Order Total

=

Sum(Order Items)
```

Invariant.

Another

```
Wallet Balance

>= 0
```

Invariant.

Aggregate Root guarantees these.

---

# 12. Order Aggregate Example

Root

```
Order
```

Methods

```tsx
addItem()

removeItem()

changeQuantity()

cancel()

confirm()
```

Notice

No one edits OrderItem directly.

---

# 13. Parking Example

Aggregate Root

```
ParkingSession
```

Methods

```
start()

end()

extend()

cancel()
```

Not

```
ParkingLeg.setEndTime()
```

The session controls it.

---

# 14. Aggregate Design Rules

Rule 1

Small Aggregates.

---

Rule 2

Only one Root.

---

Rule 3

Only Root is public.

---

Rule 4

Everything inside updated together.

---

Rule 5

Other Aggregates referenced by ID.

---

Rule 6

Never cross Aggregate transactions.

---

# 15. Aggregate vs Entity

Entity

Identity.

Aggregate

Consistency boundary.

Example

```
OrderItem

↓

Entity
```

Order

↓

Aggregate Root

---

# 16. Aggregate vs Repository

Repository stores

Aggregate Roots.

Not child entities.

Example

```tsx
OrderRepository
```

Never

```
OrderItemRepository
```

usually.

---

# 17. Aggregate vs Microservice

Huge misconception.

One Aggregate

≠

One Microservice.

A Microservice may contain several related Aggregates.

Or a very large bounded context may later be split into multiple services.

---

# 18. Aggregate Design Process (Senior Approach)

When designing an Aggregate, ask these questions in order:

### Step 1

What business rule must always remain true?

↓

Invariant.

---

### Step 2

Which entities participate in that rule?

↓

Candidates for the Aggregate.

---

### Step 3

Can they be updated independently?

If yes

↓

Separate Aggregate.

---

### Step 4

Who owns the consistency?

↓

Aggregate Root.

---

### Step 5

What methods should the Root expose?

↓

Only business operations.

---

# 19. Real Amazon Example

Order

Aggregate

```
Order

↓

Items

↓

Discount
```

Payment

Aggregate

```
Payment
```

Inventory

Aggregate

```
Inventory
```

Shipment

Aggregate

```
Shipment
```

Communication

↓

Kafka

↓

Saga

---

# 20. Common Interview Questions

### Why not create one huge Aggregate?

Because large Aggregates reduce concurrency, increase locking, and become difficult to maintain.

---

### Why only one Aggregate Root?

To ensure all business rules are enforced from a single entry point.

---

### Why reference other Aggregates by ID?

To avoid loading large object graphs and to keep Aggregates independent.

---

### Can two Aggregates be updated in one transaction?

Generally, **no**.

Use:

- Domain Events
- Saga
- Eventual Consistency

---

### Why are Aggregates important?

They define consistency boundaries and transactional boundaries.

---

# 21. Senior Developer Notes

```
Aggregate

Definition:
Consistency Boundary.

Aggregate Root

Single public entry point.

Purpose:

✓ Maintain invariants

✓ Control modifications

✓ Transaction boundary

Rules:

1.
One Root

2.
Small Aggregate

3.
Reference other Aggregates by ID

4.
One Transaction per Aggregate

5.
Root enforces all business rules

Use:

Domain Events

Saga

for communication between Aggregates.

Never:

Update child entities directly.

Never:

Create giant Aggregates.
```

---

# ⭐ The Biggest Mistake Developers Make

Many developers design Aggregates based on **database relationships**.

For example:

```
Order
   |
OrderItem
   |
Customer
   |
Payment
   |
Inventory
```

They think:

> "These tables have foreign keys, so they should all be one Aggregate."
> 

❌ That's wrong.

**Aggregates are designed around business consistency, not database relationships.**

A foreign key tells you how data is related.

An Aggregate tells you **what must be kept consistent in a single transaction**.

---

## 📌 Before the Next Topic

The next topic,

**Domain Events vs Integration Events**,

becomes much easier now because you'll see how **Aggregates communicate without directly updating each other**.

That's the missing piece that connects DDD to Kafka, Outbox, and Saga—the topics you've already mastered.
