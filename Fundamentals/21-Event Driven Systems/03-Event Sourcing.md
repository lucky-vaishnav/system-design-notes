# 1. The Problem Event Sourcing Solves

Let's take a banking example.

Current database:

```
Account

AccountId
Balance
```

Suppose:

Initially:

```
Balance = 1000
```

Customer deposits:

```
+500
```

Database becomes:

```
Balance = 1500
```

Customer withdraws:

```
-300
```

Database becomes:

```
Balance = 1200
```

---

Question:

**How did balance become 1200?**

Database only stores:

```
1200
```

Everything else is lost.

---

Questions like:

```
Who deposited?

When?

How much?

Who reversed payment?

What happened yesterday?

Can we replay history?
```

Impossible unless you've built audit tables.

---

Traditional databases only store:

```
Current State
```

---

# 2. Event Sourcing Says

Instead of storing:

```
Current State
```

Store:

```
Every Event
```

---

Instead of:

```
Balance = 1200
```

Store:

```
AccountCreated

Deposit500

Withdraw300
```

---

Current balance becomes:

```
1000

+500

-300

------

1200
```

State is **calculated**.

---

# 3. Traditional Database

```
Account

ID

Balance = 1200
```

---

# Event Sourcing Database

```
Events

1 AccountCreated

2 MoneyDeposited +1000

3 MoneyDeposited +500

4 MoneyWithdrawn -300
```

Notice:

No balance column.

---

# 4. Core Idea

Traditional:

```
Store State
```

Event Sourcing:

```
Store State Changes
```

Huge difference.

---

# 5. Real Example

Uber Ride

Traditional

```
Ride

Status

Completed
```

---

What happened?

Unknown.

---

Event Sourcing

```
RideRequested

DriverAssigned

DriverArrived

RideStarted

RideCompleted
```

Now entire ride history exists forever.

---

# 6. State Is Rebuilt

Example

Events

```
AccountCreated

Deposit100

Deposit50

Withdraw20
```

To calculate balance:

```
0

+100

+50

-20

=130
```

Database never stored:

```
130
```

Application calculated it.

---

# 7. Real Production Code

Event Store

```
Event

Id

AggregateId

EventType

Payload

CreatedAt
```

---

Example

```
{
  "aggregateId":101,
  "eventType":"MoneyDeposited",
  "amount":500
}
```

---

Another event

```
{
  "aggregateId":101,
  "eventType":"MoneyWithdrawn",
  "amount":300
}
```

---

# 8. Aggregate

Very common interview word.

Aggregate = Business Object.

Example:

```
Account

Order

Ride

Invoice
```

Each aggregate has its own event stream.

Example:

```
Ride123

↓

RideRequested

↓

DriverAssigned

↓

RideStarted

↓

RideCompleted
```

---

# 9. Rebuilding State

Suppose Account 101.

Load:

```
Deposit100

Deposit50

Withdraw20
```

Replay:

```
letbalance=0;

for(eventofevents){

if(event.type==="Deposit")
balance+=event.amount;

if(event.type==="Withdraw")
balance-=event.amount;
}
```

Result:

```
130
```

---

# 10. Replay

One of Event Sourcing's superpowers.

Example:

Bug fixed today.

Need to regenerate analytics.

Instead of:

```
Restore Backup
```

Replay all events.

```
2019

↓

2020

↓

2021

↓

2022

↓

2023

↓

2024
```

Everything rebuilt.

Amazing.

---

# 11. Time Travel

Question:

"What was balance yesterday?"

Traditional DB

Impossible.

---

Event Store

Replay until:

```
Yesterday 5PM
```

Done.

---

This is why banks love Event Sourcing.

---

# 12. Audit Trail

Every change exists.

Forever.

Example

```
PaymentCreated

PaymentApproved

PaymentCaptured

PaymentRefunded
```

Nothing lost.

---

# 13. CQRS + Event Sourcing

Perfect combination.

Flow

```
Command

↓

Event Store

↓

Publish Event

↓

Read Model

↓

Queries
```

Notice:

Read Model still exists.

Exactly what you learned yesterday.

---

# 14. Snapshot

Problem.

Imagine:

```
Bank Account

5 million events
```

Need balance.

Replaying:

```
5 million events
```

Too slow.

---

Solution:

Snapshot.

Store:

```
Balance = 1,000,000

after Event #4,000,000
```

Later replay only:

```
Last 1 million
```

Much faster.

---

Interview Question:

Why snapshots?

Answer:

```
Avoid replaying huge event streams.
```

---

# 15. Event Versioning

Production issue.

Old event

```
{
"name":"Lucky"
}
```

Later

Need

```
{
"name":"Lucky",
"email":"..."
}
```

Cannot break old events.

Need versioning.

```
{
"version":2
}
```

Very important.

---

# 16. Event Immutability

Events are NEVER updated.

Never.

Wrong

```
Update Event
```

Correct

```
Append New Event
```

Example

Wrong amount?

Don't edit.

Append

```
RefundIssued
```

History preserved.

---

# 17. Real Stripe Example

Payment

```
PaymentCreated

↓

PaymentAuthorized

↓

PaymentCaptured

↓

PaymentRefunded
```

Exactly Event Sourcing.

---

# 18. Real Uber Example

Ride

```
RideRequested

↓

DriverAssigned

↓

RideAccepted

↓

DriverArrived

↓

RideStarted

↓

RideCompleted
```

Entire ride history.

---

# 19. Advantages

```
Perfect Audit

Replay

Time Travel

Debugging

Event Driven

Easy CQRS integration

History preserved
```

---

# 20. Disadvantages

Much more complex.

Need:

```
Snapshots

Replay

Versioning

Event Store

Projection

Read Models
```

Learning curve high.

---

# 21. When NOT To Use

Simple CRUD.

Inventory app.

School management.

Employee Portal.

Overkill.

---

# 22. When To Use

Banking

Payments

Trading

Accounting

Ledger

Ride History

Financial Systems

Insurance

Healthcare

---

# 23. Interview Questions

---

## Difference Between Audit Table and Event Sourcing?

Audit

```
Extra logging.
```

---

Event Sourcing

```
Events ARE the database.
```

Huge difference.

---

## Why Snapshot?

Performance.

---

## Can Event Be Changed?

Never.

Append only.

---

## How Is Current State Built?

Replay events.

---

## Why CQRS With Event Sourcing?

Read model becomes projection of events.

Perfect match.

---

# 24. Complete Flow

```
User

↓

Command

↓

Command Handler

↓

Validate

↓

Save Event

↓

Event Store

↓

Kafka Publish

↓

Consumers

↓

Read Models

↓

Dashboard

↓

Reports

↓

Search
```

Almost every topic we've learned appears in this architecture.

---

# 25. Real Node.js Example

Command

```
awaiteventStore.save({
    aggregateId:101,
    type:"MoneyDeposited",
    amount:500
});

awaitkafka.publish("MoneyDeposited");
```

Read Model

```
consumer.on("MoneyDeposited",async(event)=>{

awaitaccountProjection.updateBalance(...);

});
```

Notice:

Nobody updates balance directly.

Balance comes from events.

---

# Event Sourcing vs Traditional DB

| Traditional | Event Sourcing |
| --- | --- |
| Stores current state | Stores all events |
| Update rows | Append events |
| Limited history | Complete history |
| Hard to replay | Replay anytime |
| Easy | Complex |
| Good for CRUD | Good for audit-heavy systems |

---

# CQRS vs Event Sourcing

Many candidates confuse these.

| CQRS | Event Sourcing |
| --- | --- |
| Separates read/write models | Stores every event |
| About architecture | About persistence |
| Can exist without Event Sourcing | Often paired with CQRS |

They are independent, but they work extremely well together.

---

# Complete Interview Notes

```
Event Sourcing

Core Idea:
Store events instead of current state.

Traditional:
Current state only.

Event Sourcing:
Every state change stored.

Advantages:
- Audit trail
- Replay
- Time travel
- Debugging
- Event history
- CQRS friendly

Disadvantages:
- Complex
- Replay cost
- Versioning
- Snapshots required

Important Concepts:
- Aggregate
- Event Store
- Replay
- Snapshot
- Append Only
- Event Versioning
- Immutable Events

Use Cases:
Banking
Payments
Trading
Accounting
Insurance
Healthcare
Ride History

Avoid:
Simple CRUD systems
```
