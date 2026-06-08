This is usually the first topic after ACID because it answers:

```
What happens when multiple transactions run at the same time?
```

Without understanding Isolation Levels, it's difficult to understand locks, deadlocks, MVCC, and concurrency control later.

---

# Why Isolation Levels Exist

Suppose your parking system has:

```
Spot A
Available = 1
```

Two users try to book it simultaneously.

### Transaction A

```
SELECT availableFROM spotsWHERE id=1;
```

Returns:

```
1
```

---

### Transaction B

Runs at the same time:

```
SELECT availableFROM spotsWHERE id=1;
```

Returns:

```
1
```

---

Now both do:

```
UPDATE spots
SET available=0
WHERE id=1;
```

Both think they booked the spot.

Problem:

```
One parking spot
Two reservations
```

This is why databases need concurrency control.

---

# SQL Standard Isolation Levels

There are 4 standard levels:

```
1. Read Uncommitted
2. Read Committed
3. Repeatable Read
4. Serializable
```

Think of them as:

```
More Isolation
=
More Safety
=
Less Concurrency
```

---

# 1. Read Uncommitted

Lowest isolation.

Transaction B can read data that Transaction A has not committed yet.

Example:

### Transaction A

```
UPDATE accounts
SET balance= balance-100
WHERE id=1;
```

Balance becomes:

```
900
```

but A has NOT committed yet.

---

### Transaction B

```
SELECT balance
FROM accounts
WHERE id=1;
```

Sees:

```
900
```

---

Then Transaction A rolls back.

Actual balance:

```
1000
```

Transaction B saw data that never really existed.

This is called:

# Dirty Read

```
Reading uncommitted data
```

---

# 2. Read Committed

Most commonly used level.

(PostgreSQL default)

A transaction can only read committed data.

Dirty Reads are prevented.

---

Example

### Transaction A

```
UPDATE accounts
SET balance=900;
```

Not committed.

---

### Transaction B

```
SELECT balance
```

Still sees:

```
1000
```

until A commits.

Good.

---

But another problem exists.

# Non-Repeatable Read

---

Transaction A:

```
BEGIN;

SELECT balance;
```

Gets:

```
1000
```

---

Transaction B:

```
UPDATE balance=900;
COMMIT;
```

---

Transaction A:

```
SELECT balance;
```

Now gets:

```
900
```

Same query.

Same transaction.

Different result.

This is a:

```
Non-Repeatable Read
```

---

# 3. Repeatable Read

(PostgreSQL default behavior is even stronger than standard Repeatable Read because of MVCC.)

Guarantee:

```
If you read a row once,
you'll see the same version
for the whole transaction.
```

---

Transaction A:

```
BEGIN;

SELECT balance;
```

Gets:

```
1000
```

---

Transaction B:

```
UPDATE balance=900;
COMMIT;
```

---

Transaction A:

```
SELECT balance;
```

Still gets:

```
1000
```

because it uses the same snapshot.

Non-repeatable reads are prevented.

---

# 4. Serializable

Highest isolation.

Database behaves as if:

```
Transactions ran one after another
```

even when they actually run concurrently.

Safest.

Slowest.

Most locking/retries.

---

Used for things like:

```
Bank transfers
Financial ledgers
Critical inventory
```

---

# The Three Famous Problems

These are asked constantly in interviews.

## Dirty Read

```
Read uncommitted data
```

---

## Non-Repeatable Read

```
Same row
Different values
Inside one transaction
```

---

## Phantom Read

Example:

Transaction A:

```
SELECT*
FROM reservations
WHERE spot_id=1;
```

Returns:

```
2 rows
```

---

Transaction B inserts:

```
INSERT INTO reservations ...
```

Commits.

---

Transaction A runs the same query again.

Now gets:

```
3 rows
```

New row "appeared".

That's a:

```
Phantom Read
```

---

# Easy Interview Table

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| Read Uncommitted | ❌ Possible | ❌ Possible | ❌ Possible |
| Read Committed | ✅ Prevented | ❌ Possible | ❌ Possible |
| Repeatable Read | ✅ Prevented | ✅ Prevented | Depends on DB |
| Serializable | ✅ Prevented | ✅ Prevented | ✅ Prevented |

---

# Important PostgreSQL Interview Fact

Many engineers memorize the SQL standard table and stop there.

For PostgreSQL:

```
Default Isolation Level:
Read Committed
```

But PostgreSQL implements isolation using MVCC, which behaves differently from many other databases.

That's why the next topic I'd recommend after this is:

```
MVCC (Multi-Version Concurrency Control)
```

because it explains:

- Why reads don't block writes
- Why writes don't block reads
- How PostgreSQL snapshots work
- Why Repeatable Read behaves differently in PostgreSQL

MVCC is one of the most valuable database topics for backend engineers working on payments, reservations, bookings, refunds, and high-traffic systems.

## Question 1: Is Non-Repeatable Read really bad?

**Answer: Not always.**

Your thinking is actually reasonable:

> "If another transaction updated the data, shouldn't I see the latest value?"
> 

In many cases, **yes**, that's exactly what you want.

Example:

```
User opens Admin Portal
Checks wallet balance = $100

Another user adds $50

Admin refreshes
Sees $150
```

This is good.

Nobody wants stale data here.

---

The key thing is:

### Isolation Levels are not about "latest data"

They are about:

```
Consistency of a transaction
```

Imagine a bank transfer transaction.

Transaction A:

```
BEGIN;

SELECT balance;
-- returns 1000
```

At this point your business logic decides:

```
If balance >= 1000
allow withdrawal
```

Now another transaction changes the balance:

```
UPDATE balance=100;
COMMIT;
```

Then Transaction A reads again:

```
SELECT balance;
-- returns 100
```

Inside the SAME transaction:

```
First read = 1000
Second read = 100
```

Now your business logic becomes confusing because the transaction no longer has a consistent view of the world.

That's why Non-Repeatable Read is considered a "problem" in some scenarios.

---

## Question 2: Then why would Repeatable Read be useful?

Because some operations need a stable snapshot.

Example:

```
Generating monthly invoice
Calculating tax report
Generating settlement report
Computing parking revenue
```

Suppose your report starts at 10:00.

While it is running:

```
Thousands of payments arrive.
```

Should the report include them?

If data keeps changing during the report:

```
Total revenue = based on old values
Total refunds = based on new values
Total trips = based on mixed values
```

You can end up with inconsistent numbers.

Repeatable Read says:

```
Show me the database exactly as it looked
when my transaction started.
```

Now the entire report is internally consistent.

---

## Question 3: Which one is better?

Neither.

They solve different problems.

### Read Committed

Good for:

```
Web applications
Dashboards
Admin portals
Most APIs
```

Usually you want latest committed data.

---

### Repeatable Read

Good for:

```
Reports
Financial calculations
Batch jobs
Settlement jobs
```

Usually you want consistency more than freshness.

---

## Question 4: Does the table mean "prevented"?

Yes.

Table:

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| Read Uncommitted | ❌ Possible | ❌ Possible | ❌ Possible |
| Read Committed | ✅ Prevented | ❌ Possible | ❌ Possible |
| Repeatable Read | ✅ Prevented | ✅ Prevented | Depends |
| Serializable | ✅ Prevented | ✅ Prevented | ✅ Prevented |

Meaning:

```
✅ = database prevents it

❌ = database allows it
```

For example:

```
Read Committed
```

means:

```
Dirty Read = impossible

Non-Repeatable Read = possible

Phantom Read = possible
```

---

## Question 5: What do most databases use?

Most production applications use:

```
Read Committed
```

Why?

Because it gives the best balance:

```
Fresh data
Good performance
Good concurrency
```

Examples:

- PostgreSQL default = Read Committed
- Oracle default ≈ Read Committed
- SQL Server default = Read Committed

---

## Question 6: What do most applications use?

Most APIs never explicitly change isolation level.

For example, your Node.js service:

```
awaitsequelize.transaction(async (t) => {
   ...
});
```

Usually runs under:

```
Read Committed
```

because that's the database default.

---

## The mental model I use

Think of it like this:

### Read Committed

```
I want the latest committed truth.
```

### Repeatable Read

```
I want a frozen picture of the database.
```

### Serializable

```
I want the database to behave as if
transactions ran one by one.
```

# If PostgreSQL default is Read Committed, why learn Repeatable Read and MVCC?

Because:

```
MVCC is not only used by Repeatable Read.

MVCC is used by Read Committed too.
```

This is the important thing many tutorials don't explain clearly.

---

# PostgreSQL Architecture

PostgreSQL does NOT primarily use:

```
Read Lock
Write Lock
Read Lock
Write Lock
```

like some older database systems.

Instead it uses:

```
MVCC
```

for almost everything.

---

# Even Read Committed Uses MVCC

Suppose:

Transaction A:

```
UPDATE users
SET name='Lucky'
WHERE id=1;
```

Not committed yet.

---

Transaction B:

```
SELECT*
FROM users
WHERE id=1;
```

Question:

```
How does PostgreSQL prevent Dirty Reads?
```

Answer:

Using MVCC.

PostgreSQL doesn't wait for A to finish.

Instead it shows B:

```
the previous committed version
```

of the row.

---

Without MVCC:

```
Either:

1. Block the SELECT

or

2. Allow Dirty Read
```

PostgreSQL does neither.

That's why MVCC is so powerful.

---

# Then what changes between Read Committed and Repeatable Read?

Not MVCC itself.

The difference is:

```
Which snapshot is used?
```

---

# Read Committed

Every statement gets a fresh snapshot.

Example:

Transaction A:

```
BEGIN;

SELECT balance;
```

Gets:

```
100
```

---

Another transaction commits:

```
balance = 200
```

---

Transaction A:

```
SELECT balance;
```

now sees:

```
200
```

because PostgreSQL created a NEW snapshot.

---

# Repeatable Read

Transaction starts:

```
BEGIN;
```

PostgreSQL creates ONE snapshot.

---

Every query inside the transaction uses:

```
that same snapshot
```

until COMMIT.

---

Example:

```
BEGIN;

SELECT balance;
```

Gets:

```
100
```

---

Another transaction:

```
UPDATE balance=200;
COMMIT;
```

---

Transaction A:

```
SELECT balance;
```

Still sees:

```
100
```

because it keeps using the original snapshot.

---

# So MVCC is the engine

Think:

```
MVCC
│
├── Read Committed
│     Uses fresh snapshot per query
│
├── Repeatable Read
│     Uses same snapshot per transaction
│
└── Serializable
      Uses same snapshot +
      conflict detection
```

---

# Real Production Usage

In most backend APIs:

```
Read Committed
```

is enough.

Example:

- User profile APIs
- Trip APIs
- Wallet APIs
- Admin screens

You usually want:

```
latest committed data
```

---

# Where Repeatable Read is used

Example:

Settlement Job.

Imagine:

```
Calculate all transactions
between 11 PM and 12 PM
```

while payments are still arriving.

You don't want:

```
Query #1 sees 100 rows

Query #2 sees 105 rows

Query #3 sees 110 rows
```

inside the same settlement run.

You want:

```
One consistent view
```

of the database.

That's where Repeatable Read helps.

---

# Why learn MVCC if you mostly use Read Committed?

Because MVCC explains many production behaviors:

### Why reads don't block writes

```
SELECT doesn't block UPDATE
```

---

### Why writes don't block reads

```
UPDATE doesn't block SELECT
```

(most of the time)

---

### Why long-running transactions are dangerous

---

### Why VACUUM exists

---

### Why dead rows accumulate

---

### Why PostgreSQL performance degrades if VACUUM stops

---

All of these come from MVCC.

# Question 1: Does PostgreSQL automatically switch between Read Committed, Repeatable Read, and Serializable?

**No.**

PostgreSQL does **not** automatically decide:

```
This query should use Read Committed

This query should use Serializable
```

The isolation level is chosen by:

- Database default
- Session setting
- Transaction setting

---

## Default

By default PostgreSQL uses:

```
READ COMMITTED
```

for every transaction.

Most applications never change it.

---

## Change for one transaction

Example:

```
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT ...

COMMIT;
```

Only this transaction uses Repeatable Read.

---

Or:

```
BEGINTRANSACTIONISOLATIONLEVEL SERIALIZABLE;

...
```

Only this transaction uses Serializable.

---

## In Node.js / Sequelize

Example:

```
awaitsequelize.transaction(
  {
    isolationLevel:
Transaction.ISOLATION_LEVELS.REPEATABLE_READ
  },
async (t) => {
      ...
  }
);
```

---

# What do most companies use?

Usually:

```
95%+ of transactions

Read Committed
```

because:

```
Good performance

Good concurrency

Good enough for most APIs
```

---

Only special cases use:

```
Repeatable Read

Serializable
```

Examples:

- Settlement jobs
- Financial reconciliation
- Inventory systems
- Double-booking prevention
- Ledger calculations

---

# Question 2: Is MVCC PostgreSQL-only?

No.

MVCC is a database concept.

Many databases implement it.

---

## PostgreSQL

Uses MVCC heavily.

Actually PostgreSQL is one of the best examples of MVCC.

---

## MySQL (InnoDB)

Also uses MVCC.

But implementation differs.

---

## Oracle

Uses MVCC.

Oracle has used a version of MVCC for decades.

---

## SQL Server

Uses locking by default.

Can also support snapshot-based behavior.

Different implementation.

---

# NoSQL Databases?

Some do.

Some don't.

Depends on database.

---

## MongoDB

Uses concepts similar to MVCC internally.

Documents have versions.

Transactions use snapshots.

---

## Cassandra

Different model.

More focused on:

```
Eventual Consistency
```

than traditional MVCC.

---

## Redis

Generally no MVCC.

Because Redis is:

```
Single-threaded
In-memory
```

and solves concurrency differently.

---

# Is MVCC always called MVCC?

Not necessarily.

Vendors market things differently.

You may hear:

```
MVCC

Snapshot Isolation

Row Versioning

Snapshot Reads

Consistent Reads
```

Often these are related concepts.

---

# Why was MVCC invented?

Imagine old-school locking.

Transaction A:

```
UPDATE users
SET balance=100
```

Transaction B:

```
SELECT*
FROM users
```

Without MVCC:

```
SELECT waits

or

UPDATE waits
```

Lots of blocking.

Poor scalability.

---

MVCC idea:

```
Keep old versions of rows.

Readers can read old version.

Writers create new version.
```

Now:

```
Readers don't block writers.

Writers don't block readers.
```

This is why PostgreSQL handles heavy concurrent workloads so well.

---

# Real Interview Answer

If someone asks:

> Does PostgreSQL automatically choose isolation levels?
> 

Answer:

```
No. PostgreSQL uses Read Committed by default.
Applications can explicitly choose Repeatable Read or Serializable when stronger consistency guarantees are needed.
```

If someone asks:

> Is MVCC unique to PostgreSQL?
> 

Answer:

```
No. MVCC is a database concurrency-control technique used by PostgreSQL, MySQL InnoDB, Oracle, MongoDB and others, though each database implements it differently.
```

---

One thing to note:

You may never explicitly set:

```
REPEATABLE READ
```

in your entire career.

But you'll still benefit enormously from understanding MVCC because it explains:

```
VACUUM

Dead tuples

Long-running transactions

Read/write behavior

Replication lag behavior

Snapshot behavior
```

which are very common PostgreSQL production topics.

# **Serializable Level (More Info)**

## Quick Recap

### Read Committed

```
Show me latest committed data.
```

Good for:

```
Most APIs
Most CRUD operations
Admin screens
```

---

### Repeatable Read

```
Give me a consistent snapshot
for the whole transaction.
```

Good for:

```
Reports
Settlement jobs
Financial calculations
```

---

### Serializable

```
Make concurrent transactions behave
as if they ran one by one.
```

This is the strongest guarantee.

---

# Real Use Case: Last Parking Spot

Suppose:

```
Spot A
Available = 1
```

---

User A:

```
BEGIN;

SELECT available;
```

Gets:

```
1
```

---

At the same time:

User B:

```
BEGIN;

SELECT available;
```

Also gets:

```
1
```

---

Both create reservation.

Now:

```
2 reservations
1 parking spot
```

Bad.

---

# Why Repeatable Read Doesn't Fully Solve This

Repeatable Read gives:

```
Consistent snapshot
```

But:

```
Both transactions may still see
the same snapshot.
```

Both think:

```
Spot available = 1
```

Both proceed.

This is called:

```
Write Skew
```

One of the famous concurrency problems.

---

# What Serializable Does

PostgreSQL detects:

```
These two transactions
cannot both be correct.
```

At commit time:

```
Transaction A succeeds

Transaction B fails
```

Error:

```
could not serialize access
due to concurrent update
```

Application retries B.

Now only one booking succeeds.

---

# Real Use Case #1: Prevent Double Booking

Examples:

```
Parking Spot

Flight Seat

Hotel Room

Concert Ticket
```

Only one person can get it.

Serializable is perfect.

---

# Real Use Case #2: Bank Transfer System

Suppose rule:

```
Total balance across accounts
must never become negative.
```

Two transactions run together.

Each one sees:

```
Enough money exists
```

and withdraws.

After both commit:

```
System becomes invalid.
```

Serializable prevents this.

---

# Real Use Case #3: Inventory

Example:

```
Stock = 1
```

Two customers buy simultaneously.

Without Serializable:

```
Both orders succeed.
```

Now inventory:

```
-1
```

Impossible state.

---

# Why Not Always Use Serializable?

Because it is expensive.

Database must:

```
Track transaction dependencies

Detect conflicts

Abort transactions

Require retries
```

More CPU.

More memory.

More complexity.

---

# What Happens in Production?

Most systems use:

```
Read Committed
```

for:

```
95%+ of requests
```

---

Only critical workflows use:

```
Serializable
```

Examples:

```
Reserve Spot

Reserve Seat

Reserve Ticket

Transfer Money

Allocate Inventory
```

---

# Alternative Used More Often

Instead of Serializable, many companies use:

```
SELECT ...
FORUPDATE
```

or:

```
Distributed Locks

Redis Locks

Optimistic Locking
```

because Serializable can become costly.

We'll learn these when we reach:

```
Locks
Concurrency Control
```

---

# Real Backend Engineer View

If you ask:

```
How often will I use Read Committed?
```

Answer:

```
Every day.
```

---

If you ask:

```
How often will I use Repeatable Read?
```

Answer:

```
Sometimes.
```

---

If you ask:

```
How often will I use Serializable?
```

Answer:

```
Rarely.
```

But when you need it:

```
Nothing weaker is sufficient.
```

---

# Easy Memory Trick

```
Read Committed
=
Latest committed truth

Repeatable Read
=
Frozen picture

Serializable
=
One-at-a-time illusion
```

That "one-at-a-time illusion" is the key idea behind Serializable. The database allows concurrency, but guarantees the final result is the same as if transactions had executed sequentially.

PostgreSQL does **not automatically upgrade** transactions to Serializable.

You must explicitly start the transaction with Serializable isolation.

Example:

```
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- queries here

COMMIT;
```

or in Sequelize:

```
awaitsequelize.transaction(
  {
    isolationLevel:Transaction.ISOLATION_LEVELS.SERIALIZABLE
  },
async (t) => {
    ...
  }
);
```

---

## Do both User A and User B need Serializable?

**Yes.**

Serializable only works correctly when the competing transactions are running under Serializable isolation.

Imagine:

### User A

```
Serializable
```

### User B

```
Read Committed
```

Now PostgreSQL cannot fully guarantee serializable behavior between them.

Typically, if you decide:

```
Parking Reservation Flow
```

needs Serializable, then **every transaction participating in that business flow** should use Serializable.

---

## Real-world example

Reservation API:

```
POST/reserveParkingSpot
```

Inside:

```
awaitsequelize.transaction(
  {
    isolationLevel:Transaction.ISOLATION_LEVELS.SERIALIZABLE
  },
async (t) => {

// Check spot

// Create reservation

// Update availability
  }
);
```

Now:

```
User A reserve request
User B reserve request
```

both execute under Serializable.

PostgreSQL will detect conflicts and abort one if necessary.

---

## Important Interview Point

Serializable is usually applied to the **entire transaction**, not individual queries.

Not:

```
SELECT ... SERIALIZABLE
UPDATE ... SERIALIZABLE
```

Instead:

```
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT ...
UPDATE ...
INSERT ...

COMMIT;
```

Everything inside the transaction uses that isolation level.

---

## Production Reality

Most companies don't make the whole application Serializable.

Instead:

```
Normal APIs
→ Read Committed

Critical Reservation APIs
→ Serializable
```

For example:

```
Get User Profile
→ Read Committed

Get Trip History
→ Read Committed

Reserve Parking Spot
→ Serializable

Transfer Money
→ Serializable
```

because Serializable has higher overhead and can cause transaction retries.

One more important thing:

When PostgreSQL aborts a Serializable transaction due to a conflict, **your application must retry it**.

Example error:

```
could not serialize access due to concurrent update
```

A production-grade system typically catches this error and retries the transaction automatically 2–3 times before giving up. This retry requirement is one reason many teams prefer row locking (`SELECT ... FOR UPDATE`) over Serializable for specific workflows.

# Indexes  prevent concurrent updates ?

The answer is:

```
No, indexes do not prevent concurrent updates.

But UNIQUE indexes can prevent certain types of concurrency problems.
```

This is an important distinction.

---

# What SELECT FOR UPDATE does

It prevents another transaction from modifying the same row.

Example:

Transaction A:

```
BEGIN;

SELECT*
FROM parking_spots
WHERE id=1
FORUPDATE;
```

Row is locked.

---

Transaction B:

```
UPDATE parking_spots
SET available=false
WHERE id=1;
```

Must wait.

```
A finishes
↓
B continues
```

This is true concurrency control.

---

# What an Index does

An index helps find rows faster.

Example:

```
SELECT*
FROM users
WHERE email='abc@test.com';
```

Without index:

```
Full table scan
```

With index:

```
Direct lookup
```

No locking involved.

---

# Where Unique Indexes help

Imagine parking reservations.

Table:

```
spot_id
-------
101
```

Rule:

```
One active reservation per spot.
```

Create:

```
CREATE UNIQUE INDEX idx_spot
ON reservations(spot_id);
```

---

Now:

Transaction A:

```
INSERTINTO reservations
(spot_id)
VALUES (101);
```

Succeeds.

---

Transaction B:

```
INSERTINTO reservations
(spot_id)
VALUES (101);
```

Fails.

Error:

```
duplicate key value violates unique constraint
```

---

# Notice the Difference

The index did NOT:

```
Lock row

Block update

Prevent concurrent execution
```

Instead:

```
Database allowed both transactions.

Unique index rejected one.
```

---

# Production Example

Many senior engineers prefer:

```
Unique Constraints
```

over:

```
SELECT FOR UPDATE
```

when possible.

Example:

Instead of:

```
Check if reservation exists

Lock row

Insert reservation
```

They do:

```
Try insert

Unique constraint protects data
```

Much simpler.

---

# Real World Examples

### User Email

```
UNIQUE(email)
```

Prevents:

```
Two users registering same email
```

---

### Reservation

```
UNIQUE(spot_id)
```

Prevents:

```
Double booking
```

---

### Payment

```
UNIQUE(transaction_id)
```

Prevents:

```
Duplicate payment processing
```

---

# Interview Answer

If asked:

> Can indexing prevent concurrent updates?
> 

Good answer:

```
Normal indexes do not prevent concurrent updates.

They only improve lookup speed.

However, UNIQUE indexes can be used as a concurrency-control mechanism by allowing only one transaction to successfully insert or update a particular key value.
```

---

This is actually leading us toward the next topic we'll eventually cover:

```
Pessimistic Locking
   vs
Optimistic Locking
```

Where:

```
SELECT FOR UPDATE
= Pessimistic Locking

Version column / Unique Constraint
= Optimistic Locking
```

These are extremely common patterns in payment systems, booking systems, parking reservations, and ride platforms.
