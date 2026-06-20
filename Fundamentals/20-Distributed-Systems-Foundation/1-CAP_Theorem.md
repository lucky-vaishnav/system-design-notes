This is one of the most important system design topics. Once you understand CAP properly, many things we've already learned start making sense:

```
Kafka
Redis
Replication
Sharding
Distributed Locks
Microservices
Consensus
```

all eventually connect back to CAP.

---

# CAP Theorem

Created by:

```
Eric Brewer
```

Later formally proved by:

```
Gilbert & Lynch
```

---

# What Problem Is CAP Trying To Solve?

Imagine:

```
Node A
Node B
```

Both store same data.

Example:

```
User Balance = $100
```

Now network cable breaks:

```
Node A  X  Node B
```

They cannot talk.

Question:

```
Now what should system do?
```

This is where CAP comes in.

---

# CAP Components

## C = Consistency

Every read gets:

```
Latest Write
```

Example:

User updates:

```
Balance = $200
```

Any server should return:

```
$200
```

Immediately.

Not:

```
$100
```

---

Think:

```
Single source of truth
```

---

## A = Availability

Every request receives:

```
Some response
```

No errors.

Example:

```
GET /balance
```

returns:

```
200 OK
```

even if data may be stale.

---

Think:

```
System always responds
```

---

## P = Partition Tolerance

Network failure exists.

Example:

```
US Datacenter
|
| Network Failure
|
Europe Datacenter
```

Cannot communicate.

System must continue operating.

---

Think:

```
Nodes cannot talk
```

---

# The Famous Rule

In distributed systems:

```
You can only fully guarantee

2 of 3
```

during a partition.

---

Most people memorize:

```
CP
AP
CA
```

but that's not the correct way to think.

---

Real rule:

```
When Partition happens

Choose:

Consistency

OR

Availability
```

---

# Why?

Imagine:

```
Node A
Node B
```

Network breaks.

---

User writes:

```
Balance = $200
```

to:

```
Node A
```

---

Immediately user reads from:

```
Node B
```

---

Node B doesn't know update yet.

Now system has two choices.

---

# Choice 1

## Consistency

Node B says:

```
I don't know latest value.

Request rejected.
```

Example:

```
503
Timeout
Unavailable
```

---

Result:

```
Consistency ✓

Availability ✗
```

This is:

```
CP
```

---

# Choice 2

## Availability

Node B says:

```
I'll return my value anyway.

Balance = $100
```

Response succeeds.

But value is stale.

---

Result:

```
Availability ✓

Consistency ✗
```

This is:

```
AP
```

---

# Why CA Is Mostly Fiction

Many tutorials say:

```
CA systems
```

Not really.

Because:

```
Real distributed systems
always have network failures
```

Therefore:

```
P is mandatory
```

---

Real world:

```
Choose CP

OR

Choose AP
```

when partition occurs.

---

# CP Systems

Prefer:

```
Correctness
```

over:

```
Availability
```

---

Examples:

```
ZooKeeper
etcd
Consul
PostgreSQL synchronous replication
```

---

Banking example:

```
Balance = $100
```

Would you rather:

Option A

```
Return wrong balance
```

or

Option B

```
Return error
```

Most banks choose:

```
Error
```

Hence:

```
CP
```

---

# AP Systems

Prefer:

```
Availability
```

over:

```
Perfect consistency
```

---

Examples:

```
Cassandra
DynamoDB
Riak
```

---

Social media example:

User changes profile picture.

Another region still shows old picture for:

```
5 seconds
```

Acceptable.

---

Therefore:

```
AP
```

---

# Eventual Consistency

Most AP systems rely on:

```
Eventual Consistency
```

Meaning:

```
Data may differ now

Eventually become same
```

---

Example:

```
Instagram profile photo
```

Old version visible briefly.

Later:

```
All replicas synchronized
```

---

# Interview Question

## Is PostgreSQL CP or AP?

Depends.

---

Single node:

```
CAP doesn't apply
```

No distributed system.

---

Synchronous replication:

```
CP
```

---

Asynchronous replication:

```
More AP-ish
```

because replica may lag.

---

# Interview Question

## Is Kafka CP or AP?

Interesting.

---

Producer writes.

Leader partition exists.

Followers replicate.

---

If leader dies:

Kafka may temporarily stop writes until new leader elected.

Meaning:

```
Consistency preferred
```

Generally:

```
Closer to CP
```

---

# Interview Question

## Is Redis CP or AP?

Depends.

---

Standalone Redis:

```
Not CAP relevant
```

---

Redis Cluster:

Usually:

```
Availability preferred
```

during failures.

Closer to:

```
AP
```

---

# Real World Tradeoffs

## Payments

Choose:

```
CP
```

Wrong payment data unacceptable.

---

## Banking

Choose:

```
CP
```

---

## Inventory

Usually:

```
CP
```

Overselling dangerous.

---

## Social Media Feed

Choose:

```
AP
```

Temporary inconsistency acceptable.

---

## Analytics

Choose:

```
AP
```

Few seconds delay acceptable.

---

# CAP Misconception

Wrong:

```
Choose any two
```

---

Correct:

```
Partition happens

Now choose

Consistency

OR

Availability
```

---

# Senior Engineer Takeaway

When designing a system ask:

```
If network splits,

Would I rather:

1. Return stale data?

OR

2. Return an error?
```

That answer determines:

```
AP

or

CP
```

---

# Interview Notes

```
CAP Theorem

C = Consistency
Latest write always visible

A = Availability
Every request gets response

P = Partition Tolerance
System survives network failures

Real distributed systems must tolerate P.

During partition:

Choose:

CP
or
AP

CP:
Correct data
Possible errors/timeouts

Examples:
ZooKeeper
etcd
Consul

AP:
Always respond
May return stale data

Examples:
Cassandra
DynamoDB

Payments:
CP

Social Media:
AP

Most important interview question:

What happens when network partition occurs?
```

---

Next topic after CAP should be:

```
Consistency Models
```

because CAP tells us **why** consistency is hard, and Consistency Models explain **how much consistency** different systems provide. This is where concepts like:

```
Strong Consistency
Eventual Consistency
Read Your Writes
Monotonic Reads
```

come in.
