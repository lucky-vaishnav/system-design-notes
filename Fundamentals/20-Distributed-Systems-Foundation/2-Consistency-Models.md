This is the natural next topic after CAP.

Think of it this way:

```
CAP Theorem
answers:

"What happens during network failure?"

Consistency Models
answers:

"What consistency guarantee do users get?"
```

---

# Why Consistency Models Exist

Suppose:

```
Primary DB
Replica DB
```

User updates:

```
Balance = $200
```

Primary receives it.

Replication takes:

```
200ms
```

to reach replica.

---

Question:

```
If user immediately reads from replica,
what should happen?
```

Different systems give different guarantees.

That's exactly what Consistency Models define.

---

# Consistency Models Ladder

Strongest at top:

```
Strong Consistency

Linearizability

Sequential Consistency

Causal Consistency

Monotonic Reads

Read Your Writes

Eventual Consistency
```

As we go down:

```
Less consistency

More availability

More scalability
```

---

# 1. Strong Consistency

## Rule

After write succeeds:

```
Every future read
must return latest value
```

---

Example

Write:

```
Balance = $200
```

Immediately read:

```
$200
```

Always.

---

Never:

```
$100
```

---

Think:

```
Single database
```

behavior.

---

Examples

```
PostgreSQL (single primary)

etcd

ZooKeeper
```

---

Tradeoff

```
Slower

More coordination
```

---

# Interview Recognition

Banking

Payments

Inventory

Usually want:

```
Strong Consistency
```

---

# 2. Eventual Consistency

Most common distributed model.

---

Rule

```
Replica may be stale

Eventually becomes correct
```

---

Example

User changes profile picture.

Region A:

```
New photo
```

Region B:

```
Old photo
```

for 5 seconds.

Later:

```
All replicas synchronized
```

---

Examples

```
Cassandra
DynamoDB
DNS
CDNs
```

---

Interview Recognition

```
Social Media

Analytics

Metrics
```

---

# Problem With Eventual Consistency

User experience can be weird.

---

Example

Update profile:

```
Name = Lucky
```

Refresh page.

Still see:

```
Old Name
```

---

Users hate this.

Therefore weaker models introduce guarantees.

---

# 3. Read Your Writes (RYW)

Most important practical model.

---

Rule

```
After YOU write

YOU must see your write
```

---

Others may not.

---

Example

You update:

```
Name = Lucky
```

You refresh.

Must see:

```
Lucky
```

---

Another user may still see:

```
Old value
```

temporarily.

---

Very common.

---

Implementation

Often:

```
Read from primary
for same user

Read replicas for others
```

---

Examples

```
Facebook
Instagram
Twitter
```

---

Interview Question

```
How would you prevent user seeing stale data after updating profile?
```

Answer:

```
Read Your Writes
```

---

# 4. Monotonic Reads

Rule

```
Once you've seen newer data

Never see older data later
```

---

Example

You read:

```
Version 5
```

Next request should never return:

```
Version 4
```

---

Bad:

```
Read V5

Read V3

Read V6
```

---

Good:

```
V3

V4

V5

V5

V6
```

---

Used heavily in:

```
Feeds
Dashboards
```

---

# 5. Causal Consistency

Harder but important.

---

Rule

```
Cause must be seen
before effect
```

---

Example

Alice posts:

```
"I got promoted"
```

Bob comments:

```
"Congratulations!"
```

---

Bad system:

```
User sees:

Congratulations!

before

I got promoted
```

---

Good system:

```
Post first

Comment later
```

---

Cause before effect.

---

Examples

```
Collaborative editing

Social feeds
```

---

# 6. Sequential Consistency

Rule

```
Everyone sees same order

May not be real-time order
```

---

Example

Writes:

```
A
then
B
```

All users see:

```
A then B
```

---

Nobody sees:

```
B then A
```

---

But may arrive later.

---

Rare interview topic.

Know concept only.

---

# 7. Linearizability

Strongest practical consistency.

---

Rule

```
Operations appear
instantaneous
```

---

Example

Write:

```
Balance = $200
```

At:

```
10:00:01
```

Every read after:

```
10:00:01
```

returns:

```
$200
```

---

Think:

```
Perfect Strong Consistency
```

---

Examples

```
ZooKeeper
etcd
```

---

# Interview Scenarios

## Banking

Need:

```
Strong Consistency
Linearizability
```

---

## Inventory

Usually:

```
Strong Consistency
```

---

## Social Media Feed

Usually:

```
Eventual Consistency
Monotonic Reads
```

---

## Profile Update

Need:

```
Read Your Writes
```

---

## Comments

Need:

```
Causal Consistency
```

---

# Real Systems Cheat Sheet

| System | Consistency |
| --- | --- |
| PostgreSQL Primary | Strong |
| PostgreSQL Replica | Eventual-ish |
| ZooKeeper | Linearizable |
| etcd | Linearizable |
| Cassandra | Eventual |
| DynamoDB | Eventual by default |
| Redis Primary | Strong |
| Kafka | Strong ordering per partition |

---

# Most Important Interview Takeaways

If interviewer asks:

```
User updated profile but still sees old data.
How would you fix it?
```

Answer:

```
Read Your Writes
```

---

If interviewer asks:

```
How does Instagram survive replication lag?
```

Answer:

```
Eventual Consistency
+
Read Your Writes
```

---

If interviewer asks:

```
Why can Cassandra scale so much?
```

Answer:

```
Chooses Availability

Uses Eventual Consistency
```

---

# Notes Summary

```
Consistency Models

Strong Consistency
- Always latest value

Eventual Consistency
- Eventually correct

Read Your Writes
- Writer always sees own updates

Monotonic Reads
- Never go backwards

Causal Consistency
- Cause before effect

Sequential Consistency
- Same order for everyone

Linearizability
- Strongest practical guarantee

Banking:
Strong / Linearizable

Social Media:
Eventual + RYW + Monotonic

Comments:
Causal

Most Important Interview Model:
Read Your Writes
```

---

Next topic should be:

```
 Database Replication
```

because replication is exactly **what causes these consistency problems**, and you'll finally see how primary/replica, replication lag, failover, read replicas, and consistency models connect together.
