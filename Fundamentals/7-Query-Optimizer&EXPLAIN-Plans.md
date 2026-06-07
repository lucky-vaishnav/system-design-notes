Most developers know:

```
CREATE INDEX ...
```

But when a query becomes slow, the real skill is understanding:

```
How did the database decide to execute this query?
```

That is the job of the:

```
Query Optimizer
```

and

```
EXPLAIN Plan
```

---

# Question: What is a Query Optimizer?

The Query Optimizer is the database component that decides:

```
What is the fastest way to execute this query?
```

You write:

```
SELECT*
FROM users
WHERE email='abc@test.com';
```

You don't tell PostgreSQL:

```
Use index X
Read table Y first
Use nested loop
```

The optimizer decides all that.

---

# Question: Why do we need an Optimizer?

Suppose:

```
SELECT*
FROM users
JOIN orders
ON users.id= orders.user_id;
```

Database can execute this in many ways.

Option 1:

```
Read Users first
Then Orders
```

Option 2:

```
Read Orders first
Then Users
```

Option 3:

```
Use Index
```

Option 4:

```
Use Full Scan
```

Different plans may differ by:

```
100x
1000x
10000x
```

performance.

---

# Question: What does the Optimizer actually do?

Think of it as:

```
Query
   ↓
Possible Plans
   ↓
Estimate Cost
   ↓
Choose Cheapest Plan
```

Example:

```
SELECT*
FROM users
WHERE email='abc@test.com';
```

Possible plans:

```
Plan A:
Table Scan

Plan B:
Use Email Index
```

Optimizer estimates cost and chooses.

---

# Question: What is EXPLAIN?

EXPLAIN shows:

```
What plan the optimizer selected
```

Example:

```
EXPLAIN
SELECT*
FROM users
WHERE email='abc@test.com';
```

PostgreSQL might show:

```
Index Scan using idx_email
```

Meaning:

```
Optimizer chose index
```

---

# Question: Why is EXPLAIN important?

Because when somebody says:

```
Query is slow
```

You need to know:

```
What path DB took
```

Not:

```
What SQL looks like
```

---

# Question: What is a Table Scan?

Also called:

```
Sequential Scan
```

Example:

```
SELECT*
FROM users
WHERE email='abc@test.com';
```

No index exists.

DB:

```
Read row 1
Read row 2
Read row 3
...
```

until end.

---

# EXPLAIN Example

```
EXPLAIN
SELECT*
FROM users
WHERE email='abc@test.com';
```

Output:

```
Seq Scan on users
```

Meaning:

```
Full table scan
```

---

# Question: What is an Index Scan?

Example:

```
CREATE INDEX idx_email
ON users(email);
```

Query:

```
SELECT*
FROM users
WHERE email='abc@test.com';
```

Plan:

```
Index Scan
```

Database:

```
Use B+ Tree
Find row
Return result
```

---

# Question: Why doesn't DB always use Index?

This surprises many developers.

Suppose:

```
SELECT*
FROM users
WHERE active=true;
```

Table:

```
10 million rows

9.5 million active=true
```

Index exists.

Optimizer thinks:

```
Index lookup
+
Fetch 9.5 million rows

More expensive
```

than:

```
Read table once
```

Result:

```
Seq Scan
```

---

# Question: What are Statistics?

Optimizer does not know actual data.

Instead it stores:

```
Statistics
```

Examples:

```
Row count

Distinct values

Null count

Distribution
```

PostgreSQL gathers this through:

```
ANALYZE
```

---

# Question: Why are Statistics important?

Suppose:

```
SELECT*
FROM users
WHERE country='India';
```

Optimizer estimates:

```
Rows matching = 100
```

Reality:

```
Rows matching = 5 million
```

Wrong estimate.

Wrong plan.

Slow query.

---

# Question: What is Cost?

Optimizer assigns:

```
Cost
```

to each plan.

Example:

```
Table Scan Cost = 5000

Index Scan Cost = 100
```

Optimizer chooses:

```
100
```

---

# Question: What does EXPLAIN output mean?

Example:

```
EXPLAIN
SELECT*
FROM users
WHERE email='abc@test.com';
```

Output:

```
Index Scan using idx_email

cost=0.43..8.45

rows=1
```

Meaning:

### cost=0.43

Estimated startup cost.

### cost=8.45

Total estimated cost.

### rows=1

Optimizer expects:

```
1 row
```

to match.

---

# Question: What is EXPLAIN ANALYZE?

Normal EXPLAIN:

```
Estimated Plan
```

EXPLAIN ANALYZE:

```
Actually executes query
```

and shows:

```
Actual rows
Actual time
Actual execution path
```

Example:

```
EXPLAIN ANALYZE
SELECT*
FROM users
WHERE email='abc@test.com';
```

---

# Question: Why is EXPLAIN ANALYZE more useful?

Because:

```
Estimate
```

may be wrong.

Example:

```
Estimated rows = 1

Actual rows = 50000
```

Huge problem.

Usually indicates:

```
Bad statistics
```

or

```
Bad index
```

---

# Question: What is a Bitmap Index Scan?

Common PostgreSQL interview topic.

Suppose:

```
WHERE age>20
```

returns:

```
100,000 rows
```

Too many for regular index scan.

Too few for full scan.

PostgreSQL may use:

```
Bitmap Index Scan
```

Process:

```
Find matching row locations

Build bitmap

Fetch rows efficiently
```

---

# Question: What is an Index Only Scan?

Suppose:

```
SELECT email
FROM users
WHERE email='abc@test.com';
```

Index contains:

```
email
```

Database already has everything needed.

No table lookup required.

Plan:

```
Index Only Scan
```

Very fast.

---

# Question: What is a Nested Loop Join?

Example:

```
Users
Orders
```

Process:

```
For each User
    Search Orders
```

Pseudo:

```
for each user
   for each order
```

Good when:

```
Small dataset
```

---

# Question: What is Hash Join?

Database builds:

```
Hash Table
```

from one table.

Then probes using second table.

Very common.

Example:

```
Users -> HashMap

Orders -> Lookup
```

Often faster than Nested Loop.

---

# Question: What is Merge Join?

Works when both datasets sorted.

Example:

```
Users sorted by ID

Orders sorted by UserID
```

Walk both together.

Very efficient for large datasets.

---

# Interview Question: Why is my query ignoring the index?

Most common reasons:

### Low Selectivity

```
WHERE active=true
```

Matches too many rows.

---

### Outdated Statistics

```
ANALYZE users;
```

needed.

---

### Function Usage

```
WHERE LOWER(email)=...
```

Index may not be used.

---

### Leading Wildcard

```
LIKE'%abc'
```

Index often unusable.

---

### Wrong Data Type

```
WHERE id='123'
```

column is integer.

Can hurt optimization.

---

# Question: What are the first things you check when a query is slow?

Production approach:

### Step 1

Run:

```
EXPLAIN ANALYZE
```

---

### Step 2

Look for:

```
Seq Scan
```

on large tables.

---

### Step 3

Check:

```
Estimated Rows
vs
Actual Rows
```

---

### Step 4

Verify indexes.

---

### Step 5

Check join strategy.

---

# Real Production Example

Query:

```
SELECT*
FROM trip
WHERE trip_date>='2025-01-01'
AND trip_date<='2025-01-31';
```

Slow.

EXPLAIN shows:

```
Seq Scan
```

Problem:

```
No index on trip_date
```

Add:

```
CREATE INDEX idx_trip_date
ON trip(trip_date);
```

Now:

```
Index Scan
```

Query drops from:

```
8 seconds
```

to

```
100 ms
```

# Question: Optimizer uses statistics, which can be outdated, why would they become outdated?

Yes, databases can update statistics automatically, but **not always immediately after every data change**.

Think about a table:

```
users
10 million rows
```

Suppose today:

```
country='India' = 10,000 rows
```

Optimizer statistics say:

```
India → 0.1% of table
```

Now imagine a bulk import inserts:

```
5 million new Indian users
```

Actual data becomes:

```
India → 50% of table
```

But optimizer may still believe:

```
India → 0.1%
```

until statistics are refreshed.

---

## Why doesn't DB update statistics after every INSERT/UPDATE?

Because statistics collection itself is expensive.

Imagine updating statistics after every write:

```
INSERT
→ update table
→ recalculate statistics

INSERT
→ update table
→ recalculate statistics
```

That would make writes much slower.

So databases use thresholds and background processes.

---

## PostgreSQL Example

PostgreSQL has:

```
autovacuum
autoanalyze
```

When enough rows change, PostgreSQL automatically runs:

```
ANALYZE table_name;
```

to refresh statistics.

Most of the time you never think about it.

---

## When do developers manually run ANALYZE?

Common situations:

### Large bulk import

```
COPY usersFROM ...
```

millions of rows inserted.

After import:

```
ANALYZE users;
```

---

### Huge delete

```
DELETEFROM trips
WHERE trip_date<'2020-01-01';
```

millions removed.

Run:

```
ANALYZE trips;
```

---

### Query suddenly became slow

You check:

```
EXPLAIN ANALYZE
```

and see:

```
Estimated rows = 10

Actual rows = 500000
```

Huge mismatch.

Often means stale statistics.

---

# Question: How can I detect stale statistics?

One of the biggest clues:

```
Estimated rows
≠
Actual rows
```

Example:

```
Estimated rows: 1

Actual rows: 100000
```

The optimizer made decisions using incorrect assumptions.

This is a classic sign.

---

# Question: Is EXPLAIN cost itself wrong when statistics are wrong?

Yes.

Remember:

```
Cost is not actual time.
Cost is optimizer estimation.
```

Example:

Optimizer thinks:

```
1 row will match
```

So it chooses:

```
Index Scan
```

But reality:

```
500,000 rows match
```

Now index scan becomes terrible.

The chosen plan can be completely wrong because the statistics were wrong.

---

# Question: Bitmap Scan — is the bitmap stored permanently or created temporarily?

Very important interview question.

The bitmap is:

```
Temporary
```

Created only for that query execution.

Not stored permanently.

---

Example:

```
SELECT*
FROM users
WHERE age>18;
```

Optimizer chooses:

```
Bitmap Index Scan
```

Process:

### Step 1

Read index.

Find matching rows:

```
Row 5
Row 10
Row 100
Row 120
...
```

---

### Step 2

Build temporary bitmap.

Conceptually:

```
000010000100000000100...
```

or internally similar structures.

---

### Step 3

Use bitmap to fetch table pages efficiently.

---

### Step 4

Query finishes.

Bitmap destroyed.

---

# Question: Why not just use normal Index Scan?

Suppose:

```
WHERE age>18
```

returns:

```
500,000 rows
```

---

Normal Index Scan:

```
Index lookup
→ fetch row

Index lookup
→ fetch row

Index lookup
→ fetch row
```

500,000 times.

This causes lots of random disk I/O.

---

Bitmap Scan:

```
Find all matching row locations

Sort/group page accesses

Read pages efficiently
```

Much faster.

---

# Question: When does PostgreSQL decide to use Bitmap Scan?

Optimizer compares estimated costs.

Generally:

### Very few rows match

```
1 row
10 rows
50 rows
```

Use:

```
Index Scan
```

---

### Medium-sized result

```
1000 rows
10000 rows
50000 rows
```

Use:

```
Bitmap Scan
```

---

### Huge result

```
50% of table
80% of table
```

Use:

```
Sequential Scan
```

---

Think of it like:

```
Tiny result
    ↓
Index Scan

Medium result
    ↓
Bitmap Scan

Huge result
    ↓
Seq Scan
```

This is exactly the type of decision the optimizer makes using statistics.

---

# Production Interview Tip

If you run:

```
EXPLAIN ANALYZE
```

and see:

```
Bitmap Index Scan
```

that usually means:

```
The optimizer expects enough rows
that a normal index scan would be inefficient,
but not so many rows that a full table scan is better.
```

That's often the sweet spot where Bitmap Scan shines.
