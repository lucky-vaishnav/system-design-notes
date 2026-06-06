
**Caching** is one of the highest-value system design concepts because almost every large-scale system uses it.

# 1. What Problem Does Caching Solve?

Without cache:

```
User
  |
API
  |
Database
```

Every request hits the database.

Problems:

- Database becomes slow.
- Expensive queries run repeatedly.
- More CPU usage.
- Higher latency.

Example:

```
SELECT*FROM usersWHERE id=123;
```

Executed 100,000 times daily.

Same result every time.

Wasteful.

---

# 2. What is a Cache?

A cache stores frequently used data in fast memory.

Instead of:

```
User
 |
API
 |
Database
```

You get:

```
User
 |
API
 |
Cache (Redis)
 |
Database
```

Redis is RAM-based, so it's much faster than a database.

Typical latency:

```
Redis      : < 1 ms
Postgres   : 5-50 ms
```

Sometimes much more.

---

# 3. Cache Hit vs Cache Miss

### Cache Hit

Data exists in cache.

```
User
 |
API
 |
Redis
 |
Data Found
```

Response immediately.

---

### Cache Miss

Data not in cache.

```
User
 |
API
 |
Redis
 |
Not Found
 |
Database
 |
Store in Redis
 |
Return Data
```

---

# 4. Most Common Pattern: Cache Aside

This is the one you should know best.

### First Request

```
User
 |
API
 |
Redis
 |
MISS
 |
Database
 |
Redis SET
 |
Response
```

### Second Request

```
User
 |
API
 |
Redis
 |
HIT
 |
Response
```

Node.js example:

```
constkey=`user:${userId}`;

letuser=awaitredis.get(key);

if (!user) {
user=awaitUser.findByPk(userId);

awaitredis.set(
key,
JSON.stringify(user),
"EX",
3600
    );
}

returnJSON.parse(user);
```

---

# 5. TTL (Time To Live)

Cache shouldn't live forever.

Example:

```
redis.set(
"user:123",
value,
"EX",
3600
);
```

Means:

```
1 hour
```

After that:

```
Auto deleted
```

---

# 6. Why Not Cache Forever?

Suppose:

```
User Name = John
```

Stored in cache.

Later:

```
User Name = John Smith
```

Database updated.

Cache still contains:

```
John
```

Now data is stale.

---

# 7. Cache Invalidation

One of the hardest problems.

Common interview joke:

> There are only two hard things:
> 
> - Cache invalidation
> - Naming things

### Example

Update user:

```
awaitUser.update(...);
```

Then:

```
awaitredis.del(`user:${userId}`);
```

Next request reloads fresh data.

---

# 8. Write Through Cache

Normal flow:

```
Update DB
Delete Cache
```

Write Through:

```
Update Cache
Update DB
```

Both updated together.

Rare compared to Cache Aside.

---

# 9. Write Back Cache

Very fast.

```
Write Cache
Return Success
Database updated later
```

Risk:

```
Cache crash
Data lost
```

Used in specialized systems.

---

# 10. What Data Should Be Cached?

Good candidates:

### User Profiles

```
User Info
Preferences
Settings
```

---

### Product Details

```
Product Name
Description
Price
```

---

### Configuration

```
Agency Config
Feature Flags
App Settings
```

---

### Frequently Read Reports

Expensive queries.

---

# 11. What Should NOT Be Cached?

### Payment Transactions

Need latest value.

### Bank Balance

Need consistency.

### Critical Financial Data

Need accuracy.

---

# 12. Redis Is Not Only For Cache

Redis can also store:

### Sessions

```
session:123
```

### OTPs

```
otp:user123
```

### Rate Limits

```
rate:user123
```

### Queues

```
BullMQ
```

### Temporary State

```
Ride booking state
```

---

# 13. Cache Eviction

Redis memory is limited.

Suppose:

```
RAM = 10 GB
```

Redis full.

Must remove old data.

Common policies:

### LRU

Least Recently Used

Remove data not accessed recently.

Most common.

---

### LFU

Least Frequently Used

Remove data rarely accessed.

---

# 14. Single Redis Problem

Bad:

```
API
 |
Redis
```

Redis dies.

Everything slower.

---

Production:

```
API Servers
      |
   Redis
      |
Redis Replica
```

or

```
Redis Cluster
```

for high availability.

---

# 15. Local Cache vs Distributed Cache

### Local Cache

Stored inside Node process.

Example:

```
constcache= {};
```

or

```
node-cache
```

Problem:

```
API1 cache != API2 cache
```

---

### Distributed Cache

Redis.

```
API1
API2
API3
  |
Redis
```

Everyone sees same data.

Preferred.

---

# 16. Where Cache Sits

Typical architecture:

```
User
 |
Load Balancer
 |
API Servers
 |
Redis
 |
Database
```

---

# 17. Example: Uber Ride Lookup

Without cache:

```
100k requests
100k DB queries
```

With cache:

```
100k requests
5k DB queries
95k Redis queries
```

Huge reduction.

---

# 18. Common Interview Questions

### Why use Redis?

Fast in-memory storage.

---

### Difference between Redis and PostgreSQL?

Redis:

```
Fast
RAM
Temporary
```

PostgreSQL:

```
Slower
Disk
Permanent
```

---

### What is Cache Hit?

Data found in cache.

---

### What is Cache Miss?

Data not found in cache.

---

### What is TTL?

Automatic expiration time.

---

### What is Cache Aside?

Read cache first.

If missing:

- Read DB
- Save cache
- Return response

Most common pattern.

---

# How This Relates to Your Work

In your Node.js projects, common Redis usages would be:

```
Auth0 user details
Agency configurations
Permit settings
Payment configurations
Feature flags
Session data
Rate limiting
```

instead of repeatedly querying:

```
PostgreSQL
Auth0
External APIs
```
### Recommended Next Topic

After API Gateway → Load Balancer → Caching, the next natural topic is **Database Scaling (Replication, Read Replicas, Sharding, Partitioning)** because caching and database scaling are usually discussed together in system design interviews.
