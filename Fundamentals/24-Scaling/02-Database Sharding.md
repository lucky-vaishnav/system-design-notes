Perfect. Now we'll move to the next topic in the scaling phase.

This is one of the most important topics for senior backend interviews.

Almost every large company asks about it:

- Amazon
- Uber
- Google
- Microsoft
- Stripe
- Airbnb

---

# Database Sharding

---

# 1. What Problem Does Sharding Solve?

Imagine your application starts with one database.

```
Application

↓

PostgreSQL
```

Initially

```
1 Million Users

Works Fine
```

After some years

```
200 Million Users

10 Billion Rows

20 TB Database
```

Now problems start.

---

Problems

```
Queries become slower

Indexes become huge

Storage limit reached

CPU becomes bottleneck

Replication lag

Maintenance becomes difficult
```

One database server cannot grow forever.

This is called

**Vertical Scaling Limit**

---

# 2. First Solution — Vertical Scaling

Buy a bigger server.

```
16 CPU

↓

32 CPU

↓

64 CPU

↓

128 CPU
```

Eventually

No bigger machine exists.

Or

Too expensive.

---

Need another solution.

---

# 3. Horizontal Scaling

Instead of one DB

Use many DBs.

```
App

↓

DB1

DB2

DB3

DB4
```

Now

Each DB stores only part of the data.

This is

## Sharding

---

# 4. Definition

Sharding means:

> Splitting one logical database into multiple physical databases.
> 

Users still think there is

```
One Database
```

Actually

```
Shard1

Shard2

Shard3

Shard4
```

---

# 5. Example

Users

```
User1

User2

...

User100 Million
```

Instead of

```
One Table
```

Split

```
Shard1

Users

1-25M

Shard2

25-50M

Shard3

50-75M

Shard4

75-100M
```

---

# 6. Sharding Key

The most important concept.

Question:

How do we know

which shard stores a user?

Need a

## Shard Key

Example

```
userId
```

---

# 7. Simple Formula

```
Shard

=

userId % numberOfShards
```

Example

```
4 shards

User 13

13 % 4

=

1

↓

Shard1
```

User 15

```
15 % 4

=

3

↓

Shard3
```

Simple.

---

# 8. Real Node.js Example

```tsx
const shard = userId % 4;

const db = shards[shard];

await db.User.findByPk(userId);
```

Notice

Application decides

which DB to query.

---

# 9. Types of Sharding

There are several strategies.

---

## A. Hash-Based Sharding

Most common.

```
userId

↓

Hash

↓

Shard
```

Very even distribution.

Hard to query ranges.

---

## B. Range-Based Sharding

Example

```
1-100000

↓

Shard1

100001-200000

↓

Shard2
```

Advantages

Easy range queries.

Disadvantages

Hotspots.

---

Example

Newest users

Always

Shard4.

Traffic imbalance.

---

## C. Geographic Sharding

Example

```
India

↓

Mumbai DB

US

↓

Virginia DB

Europe

↓

Frankfurt DB
```

Very common.

---

## D. Directory-Based Sharding

Maintain lookup table.

Example

```
User123

↓

Shard7
```

Very flexible.

More complex.

---

# 10. Real Example

Instagram

Stores

Users

Posts

Comments

Across many shards.

User

123

Always

goes to same shard.

---

# 11. Cross-Shard Queries

Big interview topic.

Suppose

Need

```sql
SELECT COUNT(*)
FROM Users
```

Impossible on one shard.

Must ask

```
Shard1

↓

Shard2

↓

Shard3

↓

Shard4
```

Then combine.

Expensive.

---

# 12. Joins

Very important.

Suppose

Orders

on

Shard1

Customers

on

Shard4

Now

JOIN

becomes difficult.

Large companies

avoid

cross-shard joins.

---

# 13. Transactions

Suppose

Money transfer

Between

Shard1

and

Shard2.

Cannot use

simple database transaction.

Need

```
Saga

or

Distributed Transaction
```

Exactly what we've already learned.

Everything connects.

---

# 14. Resharding

Another interview favorite.

Suppose

```
4 shards
```

Need

```
8 shards
```

Problem

```
userId % 4

↓

userId % 8
```

Everything changes.

Huge migration.

This is why

Consistent Hashing

exists.

---

# 15. Sharding vs Partitioning

Most confusing interview question.

## Partitioning

One database.

Multiple partitions.

```
One PostgreSQL

↓

Partition1

Partition2

Partition3
```

Still

one DB server.

---

## Sharding

Multiple database servers.

```
DB1

DB2

DB3

DB4
```

Different machines.

---

# 16. Sharding vs Replication

Replication

Copies data.

```
Primary

↓

Replica

Replica
```

Same data.

---

Sharding

Splits data.

```
DB1

User1-25M

DB2

User25-50M
```

Different data.

---

# 17. Combining Both

Real production

Uses both.

```
Shard1

↓

Replica

Replica

Shard2

↓

Replica

Replica
```

Every shard

has replicas.

---

# 18. Parking Project Example

Imagine

BART

expands.

Need

100 Million Users.

Possible

Shard Key

```
userId
```

Queries

```tsx
const shard = getShard(userId);

return shard.query(...);
```

Parking Sessions

for one user

stay together.

---

# 19. Challenges

Hotspots

Cross-shard joins

Cross-shard transactions

Rebalancing

Backups

Monitoring

Migration

Routing

These are why sharding is considered difficult.

---

# 20. Interview Questions

### Why shard?

Database too large.

Need horizontal scaling.

---

### What is a shard key?

Field used to decide which database stores data.

---

### Good shard key?

High cardinality.

Even distribution.

Rarely changes.

Frequently queried.

Example

```
userId
```

---

### Bad shard key?

Country.

Because

```
India

↓

90% Users
```

Hotspot.

---

### Can shard key change?

Should almost never change.

Migration becomes painful.

---

### Can we JOIN across shards?

Technically yes.

Practically avoided.

---

### Can transactions span shards?

Possible but expensive.

Use Saga instead.

---

# 21. Senior Production Notes

- Design the shard key carefully—it is one of the hardest decisions and difficult to change later.
- Prefer colocating related data on the same shard when possible (e.g., all orders for a user if userId is the shard key).
- Expect operational complexity: routing, monitoring, backups, migrations, and rebalancing.
- Sharding is usually introduced only when vertical scaling and replication are no longer sufficient.

---

# 22. Senior Developer Notes

```
Sharding

Definition:
Split one logical database into multiple physical databases.

Purpose:

✓ Horizontal Scaling

✓ Store More Data

✓ Increase Throughput

Need:

Shard Key

Good Shard Key:

✓ High Cardinality

✓ Even Distribution

✓ Frequently Queried

✓ Rarely Changes

Types:

1. Hash
2. Range
3. Geographic
4. Directory

Avoid:

Cross-Shard JOINs

Cross-Shard Transactions

Hotspots

Difference:

Replication

Copies Data

Sharding

Splits Data

Partitioning

One DB Server

Sharding

Multiple DB Servers
```

---

# 🎯 Next Topic

The next topic is **Distributed Cache**.

You'll see how:

- Redis clusters are distributed,
- Consistent Hashing is used for cache distribution,
- Cache invalidation works,
- Cache-aside, Read-through, Write-through, Write-back patterns,
- Cache stampede, cache penetration, cache avalanche,
- Production caching strategies used by companies like Amazon, Uber, and Netflix.

This topic ties together everything you've learned about scaling so far.
