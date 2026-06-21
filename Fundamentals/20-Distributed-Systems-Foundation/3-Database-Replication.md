This topic is one of the most important database/system design concepts because:

```
Replication
↓
Read Replicas
↓
Replication Lag
↓
Consistency Models
↓
Failover
↓
High Availability
```

all come from this.

---

# What Problem Does Replication Solve?

Suppose we have:

```
PostgreSQL

CPU 90%
RAM 80%
5000 req/sec
```

Users growing.

---

Option A

```
Bigger server
```

Expensive.

---

Option B

```
Create copies
```

of database.

This is:

```
Replication
```

---

# Definition

Replication means:

```
Copying data

from one database

to one or more databases
```

---

Example

```
Primary

      |
      |
      v

Replica 1

Replica 2

Replica 3
```

---

All contain same data.

---

# Why Use Replication?

## 1. Read Scaling

Instead of:

```
All reads
All writes

→ Primary
```

Do:

```
Writes → Primary

Reads → Replicas
```

---

Example

```
1000 writes/sec

10000 reads/sec
```

Primary handles writes.

Replicas handle reads.

---

Huge scaling benefit.

---

## 2. High Availability

Primary crashes.

Use replica.

---

Without replication:

```
Database Down
```

---

With replication:

```
Promote Replica
```

---

Service survives.

---

## 3. Disaster Recovery

If:

```
Server
Disk
AZ
Datacenter
```

fails

Data still exists elsewhere.

---

# Basic Architecture

## Leader-Follower Replication

Most common.

Also called:

```
Primary Replica

Master Slave
```

(old terminology)

---

Architecture

```
        Primary

       /   |   \

 Replica Replica Replica
```

---

Writes:

```
INSERT
UPDATE
DELETE
```

go to:

```
Primary
```

only.

---

Reads:

```
SELECT
```

may go to:

```
Replicas
```

---

# How Replication Works

User writes:

```
UPDATE users
SET name='Lucky'
WHERE id=1;
```

---

Primary:

```
Executes update
```

---

Then sends change to replicas.

---

Replica:

```
Applies same change
```

---

Now all databases same.

---

# Two Types

---

# 1. Synchronous Replication

Primary waits.

---

Flow

```
Write

↓

Primary

↓

Replica ACK

↓

Success Returned
```

---

Meaning:

```
Replica must confirm
```

before success.

---

Example

```
Update Balance = 200
```

---

Primary waits.

Replica receives.

Replica confirms.

Only then:

```
200 OK
```

returned.

---

Advantage

```
Strong Consistency
```

---

Disadvantage

```
Slower
```

because waiting.

---

# 2. Asynchronous Replication

Most common.

---

Flow

```
Write

↓

Primary

↓

Success Returned

↓

Replica Updated Later
```

---

Advantage

```
Fast
```

---

Disadvantage

```
Replication Lag
```

---

# Replication Lag

Most important interview topic.

---

Write:

```
Balance = 200
```

Primary updated.

---

Replica still has:

```
Balance = 100
```

for:

```
50ms
500ms
5s
```

---

User reads from replica.

Sees:

```
100
```

---

This is:

```
Replication Lag
```

---

# Example You've Probably Seen

Update profile.

Refresh page.

Old data visible.

---

Usually caused by:

```
Replication Lag
```

---

# Fixing Replication Lag

---

## Option 1

Read from Primary

---

After write:

```
Read Primary
```

---

Used for:

```
Profile Update
Payments
Orders
```

---

This gives:

```
Read Your Writes
```

which we learned yesterday.

---

## Option 2

Wait

---

Not ideal.

---

## Option 3

Sticky Session

---

User who writes:

```
Continue reading primary
```

for:

```
5-10 seconds
```

---

Then switch back to replica.

---

Very common.

---

# Failover

Primary crashes.

---

Before

```
Primary

Replica1

Replica2
```

---

Primary dies.

---

Promote:

```
Replica1
```

to:

```
New Primary
```

---

System continues.

---

This is:

```
Failover
```

---

# Problem During Failover

Suppose:

```
Primary received write
```

but replica never got it.

---

Crash happens.

---

Data lost.

---

This is why:

```
Asynchronous Replication
```

can lose recent writes.

---

# PostgreSQL Replication

Most common interview example.

---

Primary

writes WAL.

---

WAL means:

```
Write Ahead Log
```

---

Every change recorded.

---

Replica continuously reads:

```
WAL stream
```

and replays it.

---

Architecture

```
Primary

WAL

↓

Replica
```

---

# Interview Question

How does PostgreSQL replicate data?

Answer:

```
WAL Streaming Replication
```

---

# Multi-Primary Replication

Less common.

---

Instead of:

```
One Primary
```

we have:

```
Primary A

Primary B
```

---

Both accept writes.

---

Problem:

```
Conflicts
```

---

Example

Primary A:

```
Balance = 100
```

Primary B:

```
Balance = 200
```

same time.

---

Which wins?

Hard problem.

---

Most systems avoid this.

---

# Read Replicas

Very common.

---

Example

```
Primary

Replica1

Replica2

Replica3
```

---

Writes:

```
Primary
```

---

Reads:

```
Any Replica
```

---

Often load balancer decides.

---

# Real World Examples

## Instagram

```
Writes → Primary

Reads → Replicas
```

---

## Amazon

```
Orders → Primary

Product browsing → Replicas
```

---

## Banking

Often:

```
Strong Consistency

Synchronous Replication
```

for critical operations.

---

# Common Interview Questions

---

## Why Use Replication?

Answer:

```
Read Scaling

High Availability

Disaster Recovery
```

---

## What Is Replication Lag?

Answer:

```
Replica behind primary
```

---

## How To Fix User Seeing Stale Data?

Answer:

```
Read Your Writes

Read Primary

Sticky Sessions
```

---

## Why Is Async Replication Popular?

Answer:

```
Much faster
```

than synchronous.

---

# Notes Summary

```
Database Replication

Purpose:
- Read Scaling
- High Availability
- Disaster Recovery

Leader-Follower Replication

Writes:
Primary

Reads:
Replicas

Types:

1. Synchronous
- Wait for replica ACK
- Strong consistency
- Slower

2. Asynchronous
- Immediate success
- Faster
- Replication lag possible

Replication Lag:
Replica behind primary

Solutions:
- Read primary
- Sticky sessions
- Read Your Writes

Failover:
Promote replica to primary

PostgreSQL:
WAL Streaming Replication

Most Common Architecture:

Primary
|
|--> Replica
|--> Replica
|--> Replica
```

### Question - How does application know where to read/write?

In most real systems, you actually have **multiple database connections** configured in code.

Example:

```
Primary DB
postgres-primary.company.com

Replica 1
postgres-replica1.company.com

Replica 2
postgres-replica2.company.com
```

Application configuration:

```
constprimaryDb=newSequelize(PRIMARY_CONFIG);

constreplicaDb=newSequelize(REPLICA_CONFIG);
```

Then:

```
// Write
awaitprimaryDb.query(`
  UPDATE users
  SET name='Lucky'
  WHERE id=1
`);

// Read
constuser=awaitreplicaDb.query(`
  SELECT *
  FROM users
  WHERE id=1
`);
```

---

#### How ORMs Usually Handle It

Most mature ORMs support replication.

Example with Sequelize:

```
constsequelize=newSequelize({
  dialect:'postgres',

  replication: {
    read: [
      {
        host:'replica1'
      },
      {
        host:'replica2'
      }
    ],

    write: {
      host:'primary'
    }
  }
});
```

Now Sequelize automatically does:

```
SELECT
→ Replica

INSERT
UPDATE
DELETE
→ Primary
```

without you manually choosing every time.

---

#### Real Production Example

Suppose:

```
GET /user/123
```

Application:

```
User.findByPk(123);
```

ORM sends it to:

```
Replica
```

---

Suppose:

```
POST /user/update
```

Application:

```
User.update(...)
```

ORM sends it to:

```
Primary
```

---

#### Problem You Learned Yesterday

User updates profile:

```
Name = Lucky
```

Write goes:

```
Primary
```

Immediately refreshes page.

Read goes:

```
Replica
```

Replica lagging.

User sees:

```
Old Name
```

This is exactly:

```
Replication Lag
```

---

#### How Big Companies Handle This

After update:

```
Read from Primary
```

for a few seconds.

Example:

```
awaitUser.update(...);

// force read from primary
constuser=awaitUser.findByPk(id, {
  useMaster:true
});
```

In Sequelize:

```
useMaster:true
```

forces primary.

---

#### Interview Point

If interviewer asks:

```
How does application know where to read/write?
```

Answer:

```
Usually application has separate primary and replica connections.

ORM or DB router routes:

SELECT -> Replica

INSERT/UPDATE/DELETE -> Primary
```

This is exactly how systems like:

```
Instagram
Amazon
Uber
```

handle read scaling.

And yes, as a senior developer, you should assume there are **multiple DB connections/pools behind the scenes**, not a single database connection string.

---

### Next Topic

The next logical topic is:

```
CQRS (Command Query Responsibility Segregation)
```

Because CQRS takes everything we learned today:

```
Primary
Replicas
Read Scaling
Consistency
```

and turns it into a full architecture pattern used in large systems.
