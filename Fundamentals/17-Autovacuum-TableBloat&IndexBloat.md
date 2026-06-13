Think of this entire topic as:

```
How PostgreSQL tables become fat over time
and how PostgreSQL cleans them.
```

---

# Quick Refresher: Why Bloat Exists

Remember MVCC?

When you run:

```
UPDATE users
SET name='Lucky'
WHERE id=1;
```

Postgres does NOT modify the row.

Instead:

```
Old Row -> Dead
New Row -> Created
```

Like:

```
Row Version 1
↓
UPDATE
↓
Row Version 2
```

Version 1 still occupies space.

---

Same for:

```
DELETEFROM users
WHERE id=1;
```

Postgres does:

```
Mark row dead
```

not:

```
Physically remove row
```

---

Over time:

```
Dead rows
Dead rows
Dead rows
Dead rows
```

accumulate.

That's where bloat comes from.

---

# Table Bloat

Suppose:

```
users table
```

contains:

```
1M live rows
```

---

During the month:

```
500k updates
```

happen.

Now:

```
1M live rows
500k dead rows
```

exist physically.

---

Table size grows:

```
1 GB
→
2 GB
→
3 GB
```

even though:

```
Still only 1M active rows
```

---

This is:

```
Table Bloat
```

---

# Why Table Bloat Is Bad

Suppose query:

```
SELECT*
FROM users
WHERE id=1;
```

needs:

```
100 pages
```

without bloat.

---

After bloat:

```
300 pages
```

need scanning.

---

Result:

```
More disk reads
More cache misses
Slower queries
```

---

# Index Bloat

Even worse.

Most engineers understand table bloat.

Few understand:

```
Index Bloat
```

---

Example:

Index:

```
CREATE INDEX idx_email
ON users(email);
```

---

Updates:

```
UPDATE users
SET email='new@test.com'
```

create new index entries.

Old index entries become dead.

---

Over time:

```
Index grows
```

like:

```
100 MB
→
500 MB
→
1 GB
```

---

even though active rows haven't grown much.

---

# Why Index Bloat Is Worse

Queries often use:

```
Indexes first
```

before touching tables.

---

Bloated index means:

```
More pages
More memory
More disk I/O
```

---

Many production issues are actually:

```
Index Bloat
```

not table bloat.

---

# Enter Autovacuum

Postgres created:

```
Autovacuum
```

to solve this.

---

Autovacuum continuously:

```
Find dead tuples
↓
Remove them
↓
Update statistics
```

---

Think:

```
Garbage Collector
```

for PostgreSQL.

---

# What Autovacuum Does

## Vacuum

Cleans:

```
Dead tuples
```

---

## Analyze

Updates:

```
Statistics
```

for planner.

---

Without ANALYZE:

Planner may think:

```
100 rows
```

when actually:

```
10 million rows
```

exist.

---

Then wrong plans happen.

---

# When Does Autovacuum Run?

Postgres tracks:

```
How many rows changed
```

---

Formula:

```
vacuum threshold
=
autovacuum_vacuum_threshold
+
(table_rows × scale_factor)
```

---

Default:

```
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.2
```

---

Example:

Table:

```
1,000,000 rows
```

---

Threshold:

```
50
+
(1,000,000 × 0.2)

=
200,050
```

---

Meaning:

```
200k dead rows
```

before vacuum starts.

---

# Why This Can Be Bad

Large table:

```
100M rows
```

---

Threshold:

```
20M dead rows
```

before cleanup.

Crazy.

---

Production teams often lower:

```
scale_factor
```

for big tables.

---

Example:

```
0.02
```

instead of:

```
0.2
```

---

Now vacuum runs much earlier.

---

# Autovacuum Tuning

Most common settings:

---

## autovacuum_vacuum_scale_factor

Default:

```
0.2
```

---

Production often:

```
0.01
0.02
0.05
```

for hot tables.

---

## autovacuum_analyze_scale_factor

Controls:

```
Statistics refresh
```

---

## autovacuum_vacuum_cost_limit

Controls:

```
How aggressive vacuum can be
```

---

Higher:

```
Faster cleanup
```

---

Lower:

```
Less CPU/Disk pressure
```

---

# How To Detect Bloat

You don't guess.

You measure.

---

Check dead tuples:

```
SELECT
relname,
n_live_tup,
n_dead_tup
FROM pg_stat_user_tables;
```

---

Look for:

```
Dead tuples huge
```

compared to:

```
Live tuples
```

---

Example:

```
Live = 1M
Dead = 900k
```

Bad sign.

---

# Real Production Rule

Watch:

```
n_dead_tup
```

for: high-update tables.

---

# VACUUM vs VACUUM FULL

Normal:

```
VACUUM users;
```

---

Removes dead tuples.

Does NOT shrink file.

---

Think:

```
Room becomes available
```

inside table.

---

File size:

```
Same
```

---

Suppose table has 10 rows.

```
Page 1

Row1
Row2
Row3
Row4
Row5
Row6
Row7
Row8
Row9
Row10
```

Now:

```
DELETEFROM users
WHERE idIN (2,4,6,8);
```

Physically table becomes:

```
Page 1

Row1
DEAD
Row3
DEAD
Row5
DEAD
Row7
DEAD
Row9
Row10
```

---

Now run:

```
VACUUM users;
```

Postgres marks those dead spaces as reusable:

```
Page 1

Row1
FREE
Row3
FREE
Row5
FREE
Row7
FREE
Row9
Row10
```

Future inserts can use:

```
FREE
FREE
FREE
FREE
```

---

But file size stays same.

If table was:

```
100 MB
```

before vacuum,

it is still:

```
100 MB
```

after vacuum.

---

# Why?

Because PostgreSQL does NOT move rows around.

It just says:

```
This space can be reused later.
```

Think of a parking lot.

Cars leave.

Spots become empty.

Parking lot size doesn't shrink.

---

# VACUUM FULL

```
VACUUMFULL users;
```

---

Rewrites entire table.

Actually shrinks file.

---

But:

```
Locks table
```

---

Dangerous in production.

---

When you run:

```
VACUUMFULL users;
```

Postgres creates an entirely new copy.

---

Old table:

```
100 MB

Row1
FREE
FREE
FREE
Row2
FREE
FREE
Row3
FREE
...
```

---

Postgres builds:

```
New table

Row1
Row2
Row3
...
```

packed tightly.

---

New size:

```
20 MB
```

---

Then:

```
Old table removed
New table renamed
```

This process is what I meant by:

```
Rewrites entire table
```

Postgres literally copies all live rows into a brand-new table file.

---

#### Why is VACUUM FULL dangerous?

Because while rewriting:

```
Old Table
 ↓
New Table
```

Postgres needs:

```
Exclusive Lock
```

on the table.

Meaning:

```
SELECT blocked
INSERT blocked
UPDATE blocked
DELETE blocked
```

until it finishes.

---

Example:

Table:

```
payments
```

Size:

```
200 GB
```

Running:

```
VACUUMFULL payments;
```

might take:

```
minutes
hours
```

depending on storage.

During that time:

```
Application waits
```

---

#### Senior Engineer Rule

Normally use:

```
VACUUM;
```

or let:

```
Autovacuum
```

handle it.

---

Use:

```
VACUUM FULL;
```

only when:

```
Table became massively bloated
AND
You have maintenance window
```

---

# Senior Engineer Mental Model

When table slow:

Ask:

```
Is planner wrong?
→ ANALYZE
```

---

Ask:

```
Dead tuples high?
→ VACUUM
```

---

Ask:

```
Table huge but row count small?
→ Bloat
```

---

Ask:

```
Index huge?
→ Index bloat
```

---

# Summary

```
MVCC
↓
Dead tuples created
↓
Table Bloat
↓
Index Bloat
↓
Slower queries

Autovacuum
↓
Removes dead tuples
↓
Updates statistics

Key Metrics:
-------------
n_live_tup
n_dead_tup

Key Settings:
-------------
autovacuum_vacuum_scale_factor
autovacuum_analyze_scale_factor
autovacuum_vacuum_cost_limit

Danger Signs:
-------------
Huge dead tuples
Huge indexes
Wrong query plans

Tools:
------
pg_stat_user_tables
VACUUM
ANALYZE
EXPLAIN ANALYZE
```
