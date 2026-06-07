# Question: What is a Transaction?

A transaction is:

```
A group of database operations
that succeed or fail together.
```

Example:

```
Transfer ₹1000 from User A to User B
```

Seems simple, but actually involves:

```
1. Deduct ₹1000 from A
2. Add ₹1000 to B
3. Save transaction record
```

Without a transaction:

```
Step 1 succeeds

Server crashes

Step 2 never runs
```

Now:

```
A lost money
B never received money
```

Data corruption.

---

# Question: What is BEGIN, COMMIT, and ROLLBACK?

Transaction lifecycle:

```
BEGIN;

SQL1
SQL2
SQL3

COMMIT;
```

Meaning:

```
Start transaction

Run queries

Make changes permanent
```

---

If something fails:

```
BEGIN;

SQL1

SQL2 fails

ROLLBACK;
```

Meaning:

```
Undo everything
```

---

# Example

Wallet:

```
Balance = 1000
```

Transaction:

```
BEGIN;

UPDATE wallet
SET balance= balance-500;

INSERTINTO payments(...);

COMMIT;
```

After commit:

```
Balance = 500
Payment record exists
```

---

If insert fails:

```
BEGIN;

UPDATE wallet
SET balance= balance-500;

INSERTINTO payments(...)-- fails

ROLLBACK;
```

Result:

```
Balance remains 1000
```

---

# Question: What is ACID?

Every proper database transaction follows ACID.

---

# A = Atomicity

Atomic means:

```
All or Nothing
```

Example:

```
Deduct money

Create payment record
```

Either:

```
Both happen
```

or

```
Neither happens
```

Never:

```
Half completed
```

---

# Why Atomicity Matters

Without it:

```
Money deducted

Payment record missing
```

Production incident.

---

# C = Consistency

# What Consistency actually means

The more accurate definition is:

```
A transaction takes the database
from one valid state
to another valid state.
```

The important phrase is:

```
Valid State
```

---

# What is a Valid State?

Suppose our business rules are:

```
Account balance >= 0

Reservation must belong to a valid user

Payment must belong to a valid reservation
```

Before transaction:

```
Database satisfies all rules
```

After transaction:

```
Database must still satisfy all rules
```

---

# Example: Money Transfer

Before:

```
Account A = 1000
Account B = 500

Total = 1500
```

Transaction:

```
Transfer 100 from A to B
```

After:

```
Account A = 900
Account B = 600

Total = 1500
```

Database remains valid.

---

# Inconsistent State

Suppose transaction does:

```
Deduct 100 from A
```

but crashes before:

```
Add 100 to B
```

Now:

```
Account A = 900
Account B = 500

Total = 1400
```

Money disappeared.

This violates business rules.

Database is inconsistent.

---

# Important Senior-Level Point

Consistency is **not something the database guarantees by itself**.

Many developers misunderstand this.

Database guarantees:

```
Atomicity
Isolation
Durability
```

But Consistency often depends on:

```
Your schema
Your constraints
Your application logic
```

---

Example:

Database allows:

```
UPDATE users
SET balance=-1000;
```

if no constraint exists.

PostgreSQL doesn't magically know:

```
Balance cannot be negative
```

unless you tell it.

Example:

```
CHECK (balance>=0)
```

Now the database can enforce it.

---

# Interview Trap

Many interviewers ask:

```
Which ACID property prevents invalid business data?
```

Most people answer:

```
Consistency
```

Partially correct.

Better answer:

```
Consistency means the transaction preserves all defined constraints and business rules, taking the database from one valid state to another.
```

---

# Why Consistency is the Hardest ACID Property

Because:

```
Atomicity
→ Database feature

Durability
→ Database feature

Isolation
→ Database feature
```

But:

```
Consistency
→ Combination of

Database constraints
+
Application logic
+
Transaction design
```

# I = Isolation

Most misunderstood ACID property.

Isolation means:

```
Concurrent transactions
should not interfere incorrectly.
```

Example:

User A:

```
Withdraw 500
```

User B:

```
Withdraw 500
```

same account.

Isolation controls what each transaction can see.

---

We'll spend a whole lesson on this because this leads directly into:

```
Dirty Reads
Non-repeatable Reads
Phantom Reads
```

---

# D = Durability

Once committed:

```
Data survives crash
```

Example:

```
COMMIT;
```

Server crashes immediately.

Still:

```
Data remains saved
```

after restart.

---

# Question: How does Durability work internally?

Databases don't immediately write every row to disk.

Instead:

```
Write transaction log
```

first.

PostgreSQL uses:

```
WAL
(Write Ahead Log)
```

Concept:

```
1. Write log
2. Commit
3. Update actual pages later
```

---

# Interview Question

Why is WAL called "Write Ahead Log"?

Because:

```
Log is written
before actual data page
```

---

# Question: Why not make every query its own transaction?

Actually PostgreSQL does.

Example:

```
UPDATE users
SET name='Lucky'
WHERE id=1;
```

Internally:

```
BEGIN

UPDATE

COMMIT
```

automatic transaction.

---

# Then Why Use Explicit Transactions?

Because many operations must succeed together.

Example:

```
Create reservation

Create payment

Create trip
```

These belong together.

---

# Question: What Happens if Server Crashes During Transaction?

Example:

```
BEGIN;

UPDATE wallet;

INSERT payment;

-- crash here

COMMIT;
```

Since commit never happened:

```
Transaction rolled back
```

after recovery.

---

# Question: Can Transactions Span Multiple Tables?

Absolutely.

Example:

```
BEGIN;

UPDATE wallet;

INSERT payment;

INSERT audit_log;

UPDATE reservation;

COMMIT;
```

All become one unit.

---

# Question: Can Transactions Span Multiple Databases?

Depends.

Single PostgreSQL instance:

```
Easy
```

Multiple databases:

```
Much harder
```

This becomes:

```
Distributed Transactions
```

which we'll learn later.

---

# Real Production Example: Parking Reservation

User books parking.

Steps:

```
1. Check availability
2. Reserve spot
3. Charge card
4. Create reservation
5. Create audit record
```

Without transaction:

```
Spot reserved

Payment failed
```

or

```
Payment charged

Reservation missing
```

Disaster.

---

With transaction:

```
Everything succeeds

or

Everything rolls back
```

---

# Important Production Rule

Many senior developers make this mistake.

Never keep a DB transaction open while waiting for slow external services.

Bad:

```
BEGIN

Call Stripe

Wait 5 seconds

COMMIT
```

Transaction stays open.

Locks stay held.

Other requests block.

---

Better:

```
Call Stripe

Receive response

Start DB transaction

Save result

COMMIT
```

Much safer.

---

# Question: Why do long transactions cause problems?

Because:

```
Locks remain active
```

which causes:

```
Blocking

Deadlocks

Slow queries
```

A more accurate ACID summary is:

```
A = Atomicity
    All operations succeed or fail together.

C = Consistency
    Transaction moves database from one valid state
    to another valid state while preserving all
    constraints and business rules.

I = Isolation
    Concurrent transactions do not interfere improperly.

D = Durability
    Once committed, data survives crashes.
```
