The next phase is where we move from **architecture** to **large-scale systems** (millions of users, thousands of servers, TBs/PBs of data).

I intentionally recommend this order because:

```
Consistent Hashing
        ↓
Distributed Cache
        ↓
Sharding
```

Sharding becomes much easier once you understand Consistent Hashing.

---

# Consistent Hashing

This is one of the most frequently misunderstood scaling topics.

Many engineers memorize:

> "Consistent Hashing distributes keys."
> 

But they don't understand **why it was invented**.

---

# 1. The Problem It Solves

Imagine you have one cache server.

```
Application

↓

Redis
```

Everything works.

Later traffic grows.

Now you have:

```
Redis-1

Redis-2

Redis-3
```

Question:

> Which Redis should store a particular key?
> 

Example

```
user:100

user:101

user:102

user:103
```

Need to decide.

---

# 2. First Idea (Naive Hashing)

Most people write:

```
server = hash(key) % number_of_servers
```

Example

```
Servers = 4

user:100

↓

hash(user100)

↓

1242

↓

1242 % 4

↓

Server 2
```

Simple.

---

# 3. Works Great...

Until a server is added.

Before

```
4 servers

hash % 4
```

After

```
5 servers

hash % 5
```

Everything changes.

Example

```
Old

1242 % 4 = 2
```

New

```
1242 % 5 = 2
```

Maybe same.

Another key

```
1243

1243 % 4 = 3

1243 % 5 = 3
```

Another

```
1244

1244 % 4 = 0

1244 % 5 = 4
```

Almost every key gets reassigned.

---

# 4. Why Is This Bad?

Suppose Redis stores:

```
100 Million Keys
```

Add one server.

Now

```
80–90 Million Keys
```

must move.

Huge network traffic.

Cache misses.

Performance drops.

Exactly what we wanted to avoid.

---

# 5. The Core Idea

Instead of hashing to

```
Server Number
```

Hash everything onto a **ring**.

---

Imagine a circle.

```
0

↓

100

↓

200

↓

300

↓

...

↓

9999

↓

Back to 0
```

Both:

- Servers
- Keys

live on the same ring.

---

# 6. Placing Servers

Example

```
Server A

hash

↓

100
```

```
Server B

↓

350
```

```
Server C

↓

700
```

Ring

```
          A(100)

      /            \

 C(700)          B(350)

      \            /

          (0)
```

---

# 7. Placing Keys

Example

```
user100

↓

hash

↓

120
```

Find first server clockwise.

```
120

↓

Server B?
```

Let's see:

```
A=100

B=350
```

120 is after A.

First clockwise server

↓

B.

So

```
user100

↓

Server B
```

---

Another

```
user200

↓

720
```

Clockwise

Wrap around

↓

Server A

---

# Visual

```
          A

     key5

B

key1

key2

key3

C

key4
```

Every key belongs to the **next clockwise server**.

---

# 8. What Happens When a New Server Is Added?

Before

```
A

↓

B

↓

C
```

Add

```
D
```

between

```
B

↓

C
```

Only keys between

```
B

↓

D
```

move.

Everything else stays.

Instead of moving 90% of keys,

maybe

```
20%
```

move.

Huge improvement.

---

# 9. Why Is It Called "Consistent"?

Because

adding/removing servers

does **not**

change the mapping of every key.

Only nearby keys move.

The hash assignment stays "mostly consistent."

---

# 10. Removing a Server

Suppose

```
Redis-2

dies.
```

Only its keys move

to

next clockwise server.

Everything else untouched.

Excellent for fault tolerance.

---

# 11. Virtual Nodes (Very Important)

Real production systems almost always use

## Virtual Nodes

Interview favorite.

---

Problem

Suppose

```
3 servers
```

Hashes become

```
10

200

900
```

Distribution becomes

```
Huge Gap

Tiny Gap

Huge Gap
```

One server stores far more keys.

---

Solution

Each server gets many virtual positions.

Example

```
Redis A

↓

A1

A2

A3

A4

A5
```

Redis B

↓

B1

B2

B3

B4

---

Now

Ring

looks like

```
A1

B2

A4

C1

A2

B5

C2
```

Very even.

---

Real systems may use

```
100

200

500

1000

virtual nodes

per server.
```

---

# 12. Real Production Example

Redis Cluster

uses

Consistent Hashing concepts (though Redis Cluster specifically uses hash slots).

Memcached clients

commonly use Consistent Hashing.

CDNs

use Consistent Hashing.

Load balancers

sometimes use it.

Distributed storage

uses it.

---

# 13. Node.js Example

Simple (naive)

```jsx
const server = hash(userId) % servers.length;
```

Consistent Hashing

```jsx
ring.addServer(server);

ring.getServer(userId);
```

Usually you don't implement the algorithm yourself in production—you use a library or a service that already supports it.

---

# 14. Example Flow

```
user100

↓

Hash

↓

Ring

↓

Redis-2
```

Server removed

↓

Only Redis-2 keys move

↓

Redis-3

Everything else unchanged.

---

# 15. Real Use Cases

### Distributed Cache

```
Redis Cluster
```

---

### CDN

```
Image

↓

Edge Server
```

---

### Load Balancer

Sticky sessions.

---

### Distributed Storage

Example

```
Cassandra

Riak

Dynamo
```

---

# 16. Complexity

Lookup

```
O(log N)
```

(using sorted ring / tree)

Insertion

```
O(log N)
```

Memory

```
O(number of virtual nodes)
```

---

# 17. Common Interview Questions

### Why not use hash % N?

Because adding/removing servers causes nearly all keys to be remapped.

---

### What problem does Consistent Hashing solve?

Minimizes data movement when cluster size changes.

---

### Why virtual nodes?

To improve load balancing and distribute keys more evenly.

---

### Does Consistent Hashing move all keys?

No.

Only keys near the added/removed server.

---

### Is Redis Cluster exactly Consistent Hashing?

Not exactly.

Redis Cluster uses **16,384 hash slots**, which achieves a similar goal with a different implementation.

---

# 18. Senior Production Notes

- Never implement your own Consistent Hashing algorithm unless you're building infrastructure software.
- Use virtual nodes for even distribution.
- Monitor hotspots—hashing isn't enough if one key is much more popular than others.
- Combine with replication for fault tolerance.

---

# 19. Senior Developer Notes

```
Consistent Hashing

Purpose:
Distribute keys across servers while minimizing key movement.

Problem:
hash(key) % N remaps almost every key when N changes.

Solution:
Hash both servers and keys onto a ring.

Rule:
A key belongs to the first server found clockwise.

Benefits:

✓ Minimal key movement

✓ Easy horizontal scaling

✓ Fault tolerant

✓ Supports dynamic clusters

Virtual Nodes:

One physical server owns multiple positions on the ring.

Benefits:

✓ Better load balancing

✓ Fewer hotspots

Common Uses:

- Distributed Cache
- Redis Clients
- Memcached
- CDN
- Distributed Storage
- Load Balancers

Interview Keywords:

Hash Ring
Clockwise Lookup
Minimal Rebalancing
Virtual Nodes
```

---

# 🔗 Connection to Upcoming Topics

The next two topics build directly on this:

```
Consistent Hashing
        ↓
Distributed Cache
        ↓
Sharding
```

You'll notice that **Distributed Cache** often uses Consistent Hashing to distribute cache keys across multiple Redis nodes, and **Sharding** uses similar partitioning ideas for databases—but with additional concerns like cross-shard queries, rebalancing, and transactions. Once you understand today's topic, those become much easier.

---

# Question

**Q1. Is the hash ring (circular structure) the only way to implement Consistent Hashing, or is it just a visualization? Are there other algorithms or approaches that achieve the same goal of consistent hashing?**

---

# Answer

**Short Answer:**

**No.** The circular hash ring is **not the only implementation**. It is the **original and most common algorithm**, but there are other algorithms that achieve the same goal.

The important thing to remember is:

> **Consistent Hashing is a concept (or goal), not a specific implementation.**
> 

The goal is:

```
When servers are added or removed,

↓

Move as few keys as possible.
```

How you achieve that can vary.

---

# 1. Hash Ring (Classic Consistent Hashing)

This is the algorithm we learned.

```
Hash Servers

↓

Hash Keys

↓

Place both on a circle

↓

Move clockwise to find the owner.
```

Example

```
        Server A

Server C       Server B

      user100
```

Advantages

- Easy to understand
- Widely used
- Supports virtual nodes

Disadvantages

- Requires maintaining the ring
- Needs virtual nodes for good load balancing

---

# 2. Hash Slots (Redis Cluster)

Redis Cluster does **not** use the circular ring.

Instead, it creates fixed partitions.

```
16384 Hash Slots

↓

Slot 0

Slot 1

...

Slot 16383
```

Then

```
Slots

↓

Servers
```

Example

```
Redis A

↓

Slots

0-5000

Redis B

↓

5001-10000

Redis C

↓

10001-16383
```

When adding a server

Only some slots move.

Not every key.

Same goal.

Different implementation.

---

# 3. Rendezvous Hashing (Highest Random Weight Hashing)

Another popular algorithm.

Instead of using a ring,

For every key

Calculate

```
Hash(key, ServerA)

Hash(key, ServerB)

Hash(key, ServerC)
```

Whichever server gets the **highest score** owns the key.

Example

```
User100

ServerA = 78

ServerB = 22

ServerC = 95

↓

ServerC wins
```

Advantages

- No ring required
- No virtual nodes needed
- Simple implementation
- Very balanced

Many modern systems prefer this over the classic ring.

---

# 4. Jump Consistent Hash

Google introduced another algorithm.

Instead of a ring

or scoring every server,

it calculates directly

```
Key

↓

Server
```

in constant time.

Advantages

- Extremely fast
- Very little memory
- Great for large clusters

Used in several Google systems.

---

# 5. Comparison

| Algorithm | Uses Ring? | Virtual Nodes? | Used By |
| --- | --- | --- | --- |
| Classic Consistent Hashing | ✅ Yes | Usually Yes | Memcached, older systems |
| Redis Hash Slots | ❌ No | No | Redis Cluster |
| Rendezvous Hashing | ❌ No | No | Modern distributed systems |
| Jump Consistent Hash | ❌ No | No | Google and high-performance systems |

---

# So Why Do We Learn the Ring?

Because it explains the **core idea**.

Every consistent hashing algorithm is trying to achieve the same thing:

```
Server Added

↓

Move Few Keys

Server Removed

↓

Move Few Keys
```

The ring is simply the easiest way to understand that concept.

---

# Interview Tip ⭐

If an interviewer asks:

> **"How does Consistent Hashing work?"**
> 

Explain the **classic hash ring** first because it's the standard explanation.

If they ask:

> **"Are there alternatives?"**
> 

You can impress them by saying:

> "Yes. The classic ring is the original implementation, but modern systems also use approaches like Redis Hash Slots, Rendezvous Hashing, and Jump Consistent Hashing. They all aim to minimize key movement when the cluster changes, but they use different algorithms."
> 

That answer is typically what a senior engineer or architect would give.

---

### 📌 Add this to your notes

```
Q: Is the circular hash ring the only implementation of Consistent Hashing?

Answer:

No.

Consistent Hashing is a concept, not one specific algorithm.

Goal:
Minimize key movement when servers are added or removed.

Common implementations:

1. Classic Hash Ring (most common for interviews)
2. Redis Hash Slots (Redis Cluster)
3. Rendezvous Hashing (Highest Random Weight)
4. Jump Consistent Hash (Google)

For interviews:
Explain the classic hash ring first, then mention these alternatives if asked.
```
