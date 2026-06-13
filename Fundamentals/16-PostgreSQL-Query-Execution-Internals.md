Goal:

```
Understand WHY a query is fast or slow
Understand EXPLAIN ANALYZE
Predict which join PostgreSQL will choose
Understand memory vs disk execution
How PostgreSQL executes SQL
```

---

# Query Execution Pipeline

Suppose you write:

```
SELECT*
FROM users
WHERE email='abc@test.com';
```

---

## Step 1: Parser

PostgreSQL checks:

```
Syntax valid?
Table exists?
Columns exist?
```

Creates parse tree.

---

Example:

```
SELEC*FROM users
```

fails here.

---

## Step 2: Planner

Planner asks:

```
How can I get this data?
```

Possible plans:

```
Plan A:
Sequential Scan

Plan B:
Index Scan
```

---

## Step 3: Optimizer

Optimizer chooses cheapest plan.

Example:

```
Table has 100 rows
```

Maybe:

```
Sequential Scan cheaper
```

---

Example:

```
Table has 100 million rows
```

Maybe:

```
Index Scan cheaper
```

---

## Step 4: Executor

Actually runs query.

Reads pages.

Returns rows.

---

Pipeline:

```
SQL
 ↓
Parser
 ↓
Planner
 ↓
Optimizer
 ↓
Executor
 ↓
Result
```

---

# EXPLAIN

Shows plan.

```
EXPLAIN
SELECT*
FROM users
WHERE email='abc@test.com';
```

Output:

```
Index Scan
```

---

# EXPLAIN ANALYZE

Actually executes.

```
EXPLAIN ANALYZE
SELECT*
FROM users
WHERE email='abc@test.com';
```

Shows:

```
Plan chosen
Rows read
Time spent
```

---

# Join Algorithms

Interview favorite.

PostgreSQL mainly uses:

```
Nested Loop Join
Hash Join
Merge Join
```

---

## Nested Loop Join

Imagine:

```
Users = 10 rows
Orders = 1000 rows
```

Query:

```
SELECT*
FROM users u
JOIN orders o
ON u.id=o.user_id;
```

---

Nested Loop:

```
for each user
    scan orders
```

Like:

```
User1 -> search orders
User2 -> search orders
User3 -> search orders
```

---

Visualization:

```
Users
  ↓
Orders

repeat
repeat
repeat
```

---

Complexity:

```
N × M
```

Potentially expensive.

---

When is it good?

```
Small outer table
Index on inner table
```

Example:

```
SELECT*
FROM users
JOIN orders
ON users.id=orders.user_id
WHERE users.id=1;
```

Very efficient.

---

## Hash Join

Most common join.

---

Example:

```
Users = 1M rows
Orders = 5M rows
```

---

Postgres builds hash table.

```
Users
 ↓
Hash Table
```

Then:

```
Orders
 ↓
Hash Lookup
```

---

Visualization:

```
Users
 ↓
Hash Build

Orders
 ↓
Hash Probe
```

---

Complexity:

```
Almost O(N+M)
```

Much faster than nested loops.

---

Usually chosen when:

```
Large datasets
Equality joins
```

Example:

```
ON users.id=orders.user_id
```

---

## Merge Join

Requires:

```
Both inputs sorted
```

---

Example:

```
Users sorted by id
Orders sorted by user_id
```

---

Postgres walks both together.

Like:

```
Merge two sorted arrays
```

---

Visualization:

```
Users →→→→
Orders →→→→
```

---

Complexity:

```
O(N+M)
```

Very efficient.

---

Used when:

```
Data already sorted
Indexes help
```

---

### How PostgreSQL Chooses Join

Generally:

```
Tiny tables
→ Nested Loop

Large equality joins
→ Hash Join

Sorted datasets
→ Merge Join
```

---

### Real Example

Suppose:

```
SELECT*
FROM user_trips ut
JOIN trip_legs tl
ON ut.trip_id=tl.trip_id;
```

You may see:

```
Hash Join
```

because:

```
Many rows
Equality join
```

---

# Sorting

Query:

```
SELECT*
FROM users
ORDERBY created_at;
```

Needs sorting.

---

Two possibilities.

---

## In-Memory Sort

Data fits inside:

```
work_mem
```

Fast.

Example:

```
10 MB
```

Sort in RAM.

---

EXPLAIN:

```
Sort Method: quicksort
Memory: 12MB
```

Excellent.

---

## Disk Sort

Data exceeds:

```
work_mem
```

---

Example:

```
500 MB
```

work_mem:

```
4 MB
```

---

Postgres spills to disk.

---

EXPLAIN:

```
Sort Method:
external merge

Disk: 600MB
```

Danger sign.

---

# Memory vs Disk Execution

One of the biggest performance killers.

---

Example:

Hash Join.

Hash table fits memory.

```
Hash Join
Memory 20MB
```

Fast.

---

Now:

```
Hash Table 2GB
```

Memory:

```
64MB
```

---

Spills to disk.

---

EXPLAIN:

```
Batches: 64
```

Bad sign.

---

Same for sorting.

---

Memory:

```
quicksort
```

Good.

---

Disk:

```
external merge
```

Slow.

---

## work_mem

Controls memory available.

Example:

```
SHOW work_mem;
```

Maybe:

```
4MB
```

---

Increasing:

```
SET work_mem='128MB';
```

can make:

```
Disk Sort
```

become:

```
Memory Sort
```

---

But careful.

If:

```
100 queries
```

each use:

```
128MB
```

Server can run out of RAM.

---

## How To Read EXPLAIN ANALYZE

Look in this order:

---

## 1 Rows

```
Estimated rows
Actual rows
```

Huge mismatch?

Statistics problem.

---

## 2 Join Type

```
Nested Loop
Hash Join
Merge Join
```

Ask:

```
Why did planner choose this?
```

---

## 3 Sort

Look for:

```
quicksort
```

Good.

or

```
external merge
```

Bad.

---

## 4 Disk Spills

Look for:

```
Disk:
Batches:
```

These often explain slow queries.

---

# Senior Engineer Mental Model

When query is slow:

Don't immediately ask:

```
Need more indexes?
```

Ask:

```
What plan did PostgreSQL choose?
```

Then inspect:

```
Scan type
Join type
Sort type
Disk spills
Row estimates
```

---

## Summary

```
Query Execution Pipeline
------------------------
Parser
Planner
Optimizer
Executor

Join Algorithms
---------------
Nested Loop
Hash Join
Merge Join

Sorting
-------
In-memory sort
Disk sort

Memory vs Disk
--------------
work_mem controls memory

Good Signs
-----------
Hash Join
Merge Join
quicksort
Memory execution

Warning Signs
-------------
Nested Loop on huge tables
External Merge Sort
Hash batches
Disk spills

Tool
----
EXPLAIN ANALYZE
```

## Good signs are always achievable ?

```
No.
Good signs are not always achievable.
Sometimes the warning sign is actually the correct plan.
```

This is exactly how senior engineers think.

---

### Example 1: Nested Loop is BAD? Not always.

People often learn:

```
Nested Loop = bad
Hash Join = good
```

Not true.

Suppose:

```
SELECT*
FROM users u
JOIN orders o
ON u.id= o.user_id
WHERE u.id=123;
```

Users:

```
1 row
```

Orders:

```
100 million rows
```

with index:

```
orders(user_id)
```

---

Postgres chooses:

```
Nested Loop
```

Why?

```
1 user
↓
Index lookup orders
```

That's extremely fast.

A Hash Join here would actually be worse because it might need to scan huge amounts of data.

---

### Example 2: External Merge Sort

You see:

```
Sort Method: external merge
Disk: 500MB
```

Can we make it quicksort?

Maybe.

Increase:

```
work_mem
```

---

But what if:

```
Data to sort = 50GB
Server RAM = 8GB
```

Then:

```
External Merge Sort
```

is unavoidable.

You physically cannot fit 50GB in memory.

---

### Example 3: Hash Batches

You see:

```
Hash Join
Batches: 64
```

Meaning:

```
Hash table didn't fit memory
```

Can we fix it?

Maybe.

---

Option A:

```
Increase work_mem
```

---

Option B:

```
Reduce rows
```

via:

```
WHERE
```

or better joins.

---

But if:

```
Table = 1 billion rows
```

and

```
Server = 4 GB RAM
```

then some batching is unavoidable.

---

## Real Goal

Junior Engineer:

```
Make all warnings disappear
```

Senior Engineer:

```
Understand WHY PostgreSQL chose that plan.
```

## When should we investigate?

If you see:

```
Nested Loop
```

and:

```
10 million rows
```

then ask questions.

---

If you see:

```
Hash Join
Batches: 128
```

and query takes:

```
15 seconds
```

investigate.

---

If you see:

```
External Merge Sort
Disk: 20MB
```

and query takes:

```
50ms
```

ignore it.

Not worth optimizing.

---

## Golden Rule

Never optimize because:

```
Plan looks ugly
```

Optimize because:

```
Query is actually slow
```

---

A very common production mistake is:

```
Engineer sees Nested Loop
↓
Thinks it's bad
↓
Forces Hash Join
↓
Query becomes slower
```

The PostgreSQL planner is often smarter than us.

Always start with:

```
EXPLAIN ANALYZE
```

and ask:

```
What is consuming time?
```

not:

```
What looks suspicious?
```

That's the mindset that separates someone who knows PostgreSQL from someone who can truly tune PostgreSQL.
