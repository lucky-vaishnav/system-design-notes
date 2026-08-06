Excellent. This is one of my favorite topics because **almost every backend system uses caching**, but very few developers truly understand it beyond "we use Redis."

A senior engineer is expected to know **when to cache, what to cache, cache consistency, invalidation strategies, and common cache failures**.

---

# 1. Why Do We Need a Cache?

Imagine your application.

```
Client

↓

API

↓

Database
```

Every request hits the database.

Example

```
GET /users/100
```

Database query

↓

20 ms

One request?

Fine.

Now

```
100,000 requests/second
```

Database becomes overloaded.

---

Question:

Do we really need to query the database every time?

Usually

No.

User profile rarely changes.

Store it in memory.

That's caching.

---

# 2. What is a Cache?

## Definition

A cache is a **fast temporary storage** used to avoid repeatedly reading the same data from a slower system.

Usually

```
Memory

↓

Much Faster

Than

↓

Disk
```

---

Example

Without cache

```
API

↓

Database

↓

20 ms
```

With cache

```
API

↓

Redis

↓

1 ms
```

Huge improvement.

---

# 3. Local Cache vs Distributed Cache

## Local Cache

Stored inside one application instance.

```
App

↓

Memory
```

Example

```jsx
const cache = new Map();
```

Advantages

- Extremely fast
- No network

Problem

Multiple servers.

---

Example

```
App1

Cache

App2

Cache

App3

Cache
```

All different.

If App1 updates data,

App2 still has stale data.

---

## Distributed Cache

One shared cache.

```
App1

App2

App3

↓

Redis Cluster
```

Everyone sees same data.

This is what most production systems use.

---

# 4. Popular Distributed Caches

- Redis ⭐ (most popular)
- Memcached
- Hazelcast
- Apache Ignite

In interviews,

assume Redis unless stated otherwise.

---

# 5. Cache-Aside Pattern (Most Common)

This is the pattern used in most Node.js applications.

Flow

```
Request

↓

Redis

↓

Miss

↓

Database

↓

Redis

↓

Return
```

Example

```tsx
let user = await redis.get(`user:${id}`);

if (!user) {
    user = await User.findByPk(id);

    await redis.set(`user:${id}`, JSON.stringify(user));
}

return user;
```

Advantages

- Simple
- Flexible
- Most common

---

# 6. Read-Through Cache

Application

never reads DB directly.

```
App

↓

Cache

↓

DB
```

Cache itself loads missing data.

Less common with Redis.

---

# 7. Write-Through Cache

Write flow

```
App

↓

Cache

↓

Database
```

Every write updates cache immediately.

Good consistency.

Higher write latency.

---

# 8. Write-Back Cache

Flow

```
App

↓

Cache

↓

Return Immediately

↓

Database Later
```

Very fast.

Risk

Cache crash

↓

Data loss.

Used less frequently.

---

# 9. Write-Around Cache

Write

goes directly

to DB.

Cache updated only on reads.

Useful when

writes are frequent,

reads are rare.

---

# 10. Cache Invalidation

The hardest problem in computer science.

😊

Suppose

User changes profile.

DB updated.

Redis

still contains

old profile.

Now

everyone gets stale data.

---

Solutions

### Option 1

Update cache

after DB.

```
DB

↓

Redis
```

---

### Option 2

Delete cache.

```
Update DB

↓

Delete Redis Key
```

Next request

rebuilds cache.

Most common.

---

# 11. TTL (Time To Live)

Every cache item can expire.

Example

```
User

↓

10 minutes
```

Redis

```
SET user:100 value EX 600
```

600 seconds.

Even if invalidation fails,

cache eventually expires.

---

# 12. Cache Hit

Data found.

```
Redis

↓

Return

↓

Done
```

---

# 13. Cache Miss

Data absent.

```
Redis

↓

Database

↓

Redis

↓

Return
```

---

# 14. Cache Hit Ratio

Very common interview metric.

Formula

```
Hits

/

(Hits + Misses)
```

Example

```
900 Hits

100 Misses

↓

90%
```

Higher

is better.

---

# 15. Cache Stampede

Interview favorite.

Imagine

Key expires.

1000 requests arrive simultaneously.

All miss cache.

All hit database.

```
1000

↓

DB
```

Database crashes.

Solutions

- Distributed Lock
- Request Coalescing
- Refresh Before Expiry

---

# 16. Cache Penetration

User requests

```
user999999999
```

Doesn't exist.

Redis

Miss

↓

DB

Miss

↓

Every request

hits DB.

Solution

Cache

Null

Result.

Or

Bloom Filter.

---

# 17. Cache Avalanche

Thousands of keys

expire together.

Suddenly

DB gets flooded.

Solution

Random TTL.

Instead of

```
10 min
```

Use

```
10-15 min
```

Different expiry times.

---

# 18. Cache Consistency

Question

Which is source of truth?

Always

```
Database
```

Cache

is

temporary.

Never

source of truth.

---

# 19. Distributed Cache Scaling

Redis Cluster

uses

multiple nodes.

Keys

distributed.

How?

Exactly what we learned yesterday.

Consistent Hashing (or Redis hash slots).

Everything connects.

---

# 20. Parking Project Example

Current

```
Profile

↓

PostgreSQL
```

Better

```
Profile

↓

Redis

↓

Miss

↓

PostgreSQL
```

Update

```
DB

↓

Delete Redis
```

Next request

rebuilds cache.

Very common.

---

# 21. What Should Be Cached?

Good candidates

```
User Profile

Station List

Configuration

Permissions

Product Catalog

Reference Data
```

Bad candidates

```
Wallet Balance

Current Payment

Bank Transfer

Financial Ledger
```

Frequently changing,

strong consistency needed.

---

# 22. Common Interview Questions

### Why cache?

Reduce database load.

Improve response time.

---

### Source of truth?

Database.

---

### Most common cache pattern?

Cache-Aside.

---

### Cache Stampede?

Many requests rebuilding same key simultaneously.

---

### Cache Penetration?

Repeated requests for non-existent data.

---

### Cache Avalanche?

Large number of keys expire together.

---

### Cache Consistency?

Update DB first,

then update/delete cache.

---

# 23. Production Architecture

```
Client

↓

API

↓

Redis

↓

Hit

↓

Return

Miss

↓

PostgreSQL

↓

Redis

↓

Return
```

---

# 24. Senior Production Practices

- Cache only data with a good read/write ratio.
- Always define a TTL unless there's a strong reason not to.
- Prefer **delete-on-update** over trying to update cached objects in many places.
- Monitor:
    - Cache hit ratio
    - Redis memory usage
    - Evictions
    - Latency
- Design for cache failures—your application should still work if Redis is temporarily unavailable.

---

# 25. Senior Developer Notes

```
Distributed Cache

Purpose:

✓ Reduce DB Load

✓ Faster Response

✓ Scale Reads

Types:

1.
Local Cache

2.
Distributed Cache

Popular:

Redis

Memcached

Patterns:

1.
Cache Aside ⭐

2.
Read Through

3.
Write Through

4.
Write Back

5.
Write Around

Key Concepts:

Cache Hit

Cache Miss

TTL

Hit Ratio

Problems:

Cache Stampede

Cache Penetration

Cache Avalanche

Source of Truth:

Always Database

Invalidate:

Update DB

↓

Delete Cache

Most Common:

Redis + Cache Aside
```

---

### Question

**Q: In the Write-Around Cache pattern, do we update the cache on every read? If not, when is the cache updated? If we don't refresh it on every read, won't users sometimes see stale data? Then what is the benefit of using Write-Around Cache?**

No. **The cache is updated only on a cache miss**, not on every read. Once the data is cached, subsequent reads come directly from the cache until it expires (TTL) or is explicitly invalidated. Yes, this means users **may temporarily see stale data**, which is acceptable for use cases like logs and analytics where perfect real-time consistency isn't required. Write-Around trades a little consistency for lower write overhead and better cache efficiency.

> **Rule to remember:
Write → Database only
First Read (Cache Miss) → Database → Cache
Later Reads → Cache until TTL/Invalidation**
>
