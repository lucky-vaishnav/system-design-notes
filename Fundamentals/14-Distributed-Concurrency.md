## What is Distributed Concurrency?

Until now, everything we learned assumed:

```
One Database
One Lock Manager
```

Example:

```
App
 ↓
PostgreSQL
```

When we use:

```
SELECT ...FORUPDATE
```

PostgreSQL handles concurrency for us.

---

## The Problem

Modern systems usually have:

```
            ┌──────────┐
            │ Server A │
            └────┬─────┘
                 │
                 │
            ┌────▼─────┐
            │ Database │
            └────▲─────┘
                 │
                 │
            ┌────┴─────┐
            │ Server B │
            └──────────┘
```

Multiple servers can process the same request at the same time.

---

## Real Example

Suppose you have:

```
parking-scheduler
```

running on:

```
Server A
Server B
Server C
```

All three execute every minute.

At 10:00 AM:

```
Permit #123 needs refund
```

All three servers find it.

Without protection:

```
Server A → Refund
Server B → Refund
Server C → Refund
```

User gets refunded 3 times.

---

## Why is this called "Distributed" Concurrency?

Because the concurrent work is happening across:

```
Multiple Processes
Multiple Servers
Multiple Containers
```

instead of a single database transaction.

---

## Key Difference

### Database Concurrency

Handled using:

```
SELECTFORUPDATE
NOWAIT
SKIP LOCKED
```

---

### Distributed Concurrency

Handled using:

```
Distributed Locks
Leader Election
Idempotency
Message Queues
```

---

## Interview Question

A very common senior backend question:

> You have 5 scheduler instances running the same cron job. How do you ensure that a job is processed only once?
> 

This is exactly the problem Distributed Concurrency solves.

---

### 

## Part 1 — Distributed Locks

---

## Why do we need Distributed Locks?

Suppose you have:

```
Server A
Server B
Server C
```

All running:

```
parking-scheduler
```

Every minute.

A permit needs refund:

```
Permit #123
```

All servers discover it simultaneously.

Without protection:

```
Server A → Refund
Server B → Refund
Server C → Refund
```

Result:

```
3 refunds
```

---

## Solution: Lock before processing

Instead of directly processing:

```
Find permit
↓
Refund
```

Do:

```
Find permit
↓
Acquire Lock
↓
Refund
↓
Release Lock
```

Only one server gets the lock.

---

# Approach 1: Database Lock

Using PostgreSQL.

Example:

```
BEGIN;

SELECT*
FROM permits
WHERE permit_id=123
FORUPDATE;

-- refund logic

COMMIT;
```

What happens:

```
Server A acquires lock
Server B waits
Server C waits
```

Only one processes.

---

## Pros

```
Simple
Already have PostgreSQL
No extra infrastructure
```

---

## Cons

```
Not great for long-running jobs
Can hold DB connections
Can increase contention
```

---

# Approach 2: SKIP LOCKED

We already learned this.

Example:

```
SELECT*
FROM permits
WHERE status='PENDING'
FORUPDATE SKIP LOCKED
LIMIT10;
```

Result:

```
Worker A gets permits 1-10
Worker B gets permits 11-20
Worker C gets permits 21-30
```

No duplicates.

This is extremely common in:

```
Schedulers
Background workers
Queue processors
```

---

# Real Example From Your Project

Suppose:

```
Auto refund scheduler
```

runs every minute.

Instead of:

```
SELECT*
FROM permits
WHERE status='REFUND_PENDING';
```

you'd do:

```
SELECT*
FROM permits
WHERE status='REFUND_PENDING'
FORUPDATE SKIP LOCKED
LIMIT100;
```

Now multiple scheduler instances can safely work together.

---

## Part 2- Redis Distributed Lock

### Question

Suppose refund processing takes:

```
30 seconds
```

because it calls:

```
Braintree
Uber
Email service
```

Would you still use a PostgreSQL lock for the entire 30 seconds?

Or does that sound risky?

Think about it before reading further, because this leads directly into **Redis Distributed Locks**, which is the next concept.

### The concern is different

The concern is not:

```
Can other workers read/process other rows?
```

They absolutely can.

The concern is:

```
Should a database transaction stay open for 30 seconds?
```

Example:

```
BEGIN;

SELECT*
FROM permits
WHERE permit_id=123
FORUPDATE SKIP LOCKED;
```

Now:

```
Call Braintree
↓
Call Uber
↓
Send Email
↓
Wait for responses
↓
30 seconds pass
```

Then:

```
COMMIT;
```

---

### Problems with long-held DB locks

During those 30 seconds:

```
Connection remains occupied
Transaction remains open
Lock remains held
```

If many workers do this:

```
50 jobs × 30 sec
```

you can end up consuming many DB connections.

---

### Better pattern

Many production systems do:

```
BEGIN;

SELECT ...
FORUPDATE SKIP LOCKED;

UPDATE permits
SET status='PROCESSING';

COMMIT;
```

Lock exists for only milliseconds.

Then:

```
Call Braintree
Call Uber
Send Email
```

outside the transaction.

**Question:** After marking a row as `PROCESSING` and committing, what happens if the server crashes before the refund is completed? How would you prevent that job from being lost forever? That's the next real-world problem distributed systems have to solve.

# Redis Distributed Lock

## Problem

Suppose you have:

```
Server A
Server B
Server C
```

All running:

```
refund scheduler
```

You want:

```
Only one server processes Permit 123
```

---

## Instead of locking in PostgreSQL

You lock in Redis.

Worker A:

```
SET permit:123:lock serverA NX EX 60
```

Meaning:

```
Create key if not exists
Expire after 60 seconds
```

---

## What happens?

Worker A:

```
SET permit:123:lock serverA NX EX 60
```

Success.

Worker B:

```
SET permit:123:lock serverB NX EX 60
```

Fails.

Worker C:

```
SET permit:123:lock serverC NX EX 60
```

Fails.

Only A processes.

---

# Why Redis?

No database transaction.

No DB connection held.

No row lock.

Just:

```
Redis key
```

---

# Example

### Database Lock

```
Acquire DB lock
↓
Call Uber
↓
Call Braintree
↓
Send Email
↓
Commit
```

DB lock exists entire time.

---

### Redis Lock

```
Acquire Redis lock
↓
Call Uber
↓
Call Braintree
↓
Send Email
↓
Delete Redis lock
```

No database lock involved.

---

# Major Benefit

Suppose payment takes:

```
45 seconds
```

With PostgreSQL:

```
DB connection occupied for 45 seconds
```

With Redis:

```
No DB connection occupied
```

Huge scalability difference.

---

# But there is a new problem

Imagine:

```
Worker A gets lock
```

Then:

```
Server crashes
```

before:

```
DEL permit:123:lock
```

Now lock remains forever.

Nobody can process Permit 123.

---

# Solution

Redis locks usually have TTL.

Example:

```
SET permit:123:lock serverA NX EX 60
```

Meaning:

```
Auto-delete after 60 seconds
```

Even if server dies.

---

# Real World Example

Suppose in your project:

```
Auto Refund Scheduler
```

running on:

```
3 EC2 servers
```

You could do:

```
lock:refund:permit:123
```

before processing.

Only one scheduler instance gets the lock.

---

# When to use DB Lock vs Redis Lock

### PostgreSQL Lock

Use when:

```
Short transaction
Data update
Few seconds max
```

Examples:

```
Wallet balance update
Permit purchase
Inventory update
```

---

### Redis Lock

Use when:

```
Long-running work
External APIs
Schedulers
Background jobs
```

Examples:

```
Refund processing
Email sending
File generation
Uber API calls
Webhook processing
```

---

# Senior Engineer Rule

A good heuristic:

```
If the work can be completed entirely inside a DB transaction,
use database locking.

If the work involves external systems and may take a long time,
consider a distributed lock (Redis).
```

---

The next important question is:

> What if Worker A acquires the Redis lock, the lock expires after 60 seconds, but Worker A is still processing and Worker B acquires the lock?
> 

That's the famous **Redis lock expiration problem**, and understanding it separates basic Redis knowledge from senior-level distributed systems knowledge.

# Your idea: Re-acquire / Extend Lock

Yes.

This is usually called:

```
Lock Renewal
Heartbeat
Lease Extension
```

Worker A periodically says:

```
I'm still alive
Extend my lock
```

For example every:

```
20 seconds
```

it extends the TTL:

```
EXPIRE permit:123:lock 60
```

or:

```
PEXPIRE permit:123:lock 60000
```

---

# But there is another trap

Suppose:

```
Worker A gets lock
```

Then lock expires.

Worker B acquires lock.

Now:

```
Worker A wakes up
```

and tries:

```
DEL permit:123:lock
```

Oops.

That lock now belongs to:

```
Worker B
```

Worker A accidentally deletes B's lock.

---

# Solution: Lock Ownership

Instead of:

```
lock = workerA
```

Store:

```
lock = 8f7c9a2e
```

(random UUID)

When releasing:

```
Only delete if value == my UUID
```

This prevents:

```
Worker A deleting Worker B's lock
```

---

# Real-world systems

Libraries such as:

```
Redlock
BullMQ
Sidekiq
Celery
```

implement:

```
TTL
Renewal
Ownership verification
```

for you.

---

# Interview Question

Suppose Redis completely crashes.

```
Redis down
```

What happens to your distributed lock?

Can you still guarantee:

```
Only one refund?
```

Think about that one.

It's the reason many payment systems still combine:

```
Redis Lock
+
Database Constraint
+
Idempotency
```

instead of trusting Redis alone.

That's also why **Idempotency** is the next topic after Distributed Concurrency.

Excellent questions. These are exactly the questions that come up in real systems.

# Question -

### 1. What if processing keeps taking longer than expected?

Suppose:

```
TTL = 60 sec
```

Worker A starts:

```
Refund permit
```

Normally takes:

```
30 sec
```

But today:

```
Uber API slow
Braintree slow
```

and it takes:

```
5 minutes
```

That's okay **if the worker keeps renewing the lock**.

Example:

```
0 sec   → acquire lock (60 sec TTL)
20 sec  → extend lock
40 sec  → extend lock
60 sec  → extend lock
80 sec  → extend lock
...
```

As long as renewal succeeds:

```
lock never expires
```

and no other worker gets it.

This is called:

```
Lease Renewal
Heartbeat
Lock Extension
```

Very common.

---

## What if worker freezes?

Suppose:

```
Node process crashes
EC2 dies
Container killed
```

Then:

```
heartbeat stops
```

Eventually:

```
TTL expires
```

Another worker can safely continue.

That's exactly why TTL exists.

---

# 2. Do we write all this manually?

Almost never.

In production people usually use libraries.

Examples:

```
Node.js → redlock
BullMQ
NestJS queues

Java → ShedLock

Python → Celery
```

These libraries handle:

```
✓ acquire lock
✓ renewal
✓ ownership token
✓ release
✓ expiration
```

for you.

---

# Simple Node.js Example

Without library:

```
awaitredis.set(
"refund:123",
"workerA",
"NX",
"EX",
60
);
```

Lots of work after that.

---

# With Redlock

Install:

```
npm install redlock
```

Code:

```
constRedlock=require('redlock');

constredlock=newRedlock([redisClient]);

constlock=awaitredlock.acquire(
  ['refund:permit:123'],
60000
);

try {
awaitprocessRefund();
}
finally {
awaitlock.release();
}
```

Much cleaner.

---

# What happens internally?

Library does something like:

Acquire:

```
refund:permit:123
value=8f7c9a2e
TTL=60s
```

Store unique token:

```
8f7c9a2e
```

When releasing:

```
Only delete if token matches
```

So another worker's lock can't be deleted accidentally.

## What if refund takes longer than 60 sec?

Then you need lock extension.

Redlock provides:

```
awaitlock.extend(60000);
```

---

## How is it usually done?

You normally don't manually call:

```
lock.extend()
```

everywhere.

Instead people run a heartbeat timer:

```
constlock=awaitredlock.acquire(
  ['refund:permit:123'],
60000
);

constrenewer=setInterval(async () => {
awaitlock.extend(60000);
},20000);

try {
awaitprocessRefund();
}
finally {
clearInterval(renewer);
awaitlock.release();
}
```

Meaning:

```
TTL = 60 sec
Renew every 20 sec
```

So the lock never expires while the worker is healthy.
