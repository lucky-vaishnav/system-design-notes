## What is Caching?

Caching means:

```
Store frequently accessed data in a faster storage layer
to avoid expensive operations.
```

Example:

```
Without Cache

Client
 ↓
API
 ↓
PostgreSQL
 ↓
Result
```

Every request hits DB.

---

With Cache:

```
Client
 ↓
API
 ↓
Redis
 ↓
Result
```

DB often isn't touched.

---

# Why Cache?

Benefits:

```
Faster Response Time
Reduced Database Load
Reduced API Calls
Reduced Cost
Improved Scalability
```

---

Example:

```
1000 requests/min
```

Without cache:

```
1000 DB queries/min
```

With cache:

```
1 DB query
999 Redis reads
```

---

# Cache Candidates

Good:

```
Station Data
Configurations
Pricing Tables
Feature Flags
Reference Data
```

Bad:

```
Live Driver Location
Real-time Balance
Rapidly Changing Data
```

---

# Pattern 1: Cache Aside

Most common pattern.

---

Flow:

```
Request
 ↓
Check Cache
 ↓
Found?
```

YES:

```
Return Cache
```

NO:

```
Read DB
 ↓
Populate Cache
 ↓
Return Result
```

---

Example:

```
conststation=awaitredis.get(key);

if (!station) {
station=awaitdb.query(...);
awaitredis.set(key,station);
}
```

---

Advantages:

```
Simple
Flexible
Widely Used
```

---

Disadvantages:

```
First request slow
Needs cache invalidation
```

---

# Cache Invalidation

Hardest part of caching.

---

Question:

```
User updates station name
```

DB updated.

Cache still contains:

```
Old value
```

Now:

```
Stale Data
```

---

Solutions:

---

## Delete Cache

After update:

```
awaitdb.update(...);

awaitredis.del(cacheKey);
```

Next request:

```
Cache Miss
→ Read DB
→ Repopulate Cache
```

Most common.

---

## Update Cache

```
awaitdb.update(...);

awaitredis.set(cacheKey,newValue);
```

---

Risk:

```
DB success
Cache update fails
```

Inconsistency.

---

# Pattern 2: Read Through

Application does NOT read DB.

Instead:

```
App
 ↓
Cache
 ↓
DB
```

Cache handles DB access.

---

Example:

```
App asks cache
Cache misses
Cache loads DB
Cache returns result
```

---

Advantage:

```
Application simpler
```

---

Disadvantage:

```
More complex cache infrastructure
```

Less common.

---

# Pattern 3: Write Through

Write both:

```
App
 ↓
Cache
 ↓
DB
```

---

Example:

```
Update Station
 ↓
Cache Updated
 ↓
DB Updated
```

---

Advantage:

```
Cache always fresh
```

---

Disadvantage:

```
Every write slower
```

---

# Pattern 4: Write Behind

Write only cache first.

---

Flow:

```
App
 ↓
Cache
 ↓
Immediate Success
```

Later:

```
Cache
 ↓
DB
```

Background process.

---

Advantage:

```
Very fast writes
```

---

Disadvantage:

```
Cache crash
↓
Data loss
```

Used carefully.

---

# Pattern 5: Refresh Ahead

Before cache expires:

```
Background Job
 ↓
Refresh Cache
```

---

Without:

```
Cache Expires
 ↓
User waits
```

---

With:

```
User always gets warm cache
```

---

Example:

```
Weather
Stock Prices
Frequently Read Configs
```

---

# Cache TTL

TTL:

```
Time To Live
```

Example:

```
EX 3600
```

means:

```
Expire after 1 hour
```

---

Question:

How long?

Depends.

---

Example:

Station Data:

```
24 hours
```

---

User Profile:

```
15 minutes
```

---

Live Data:

```
30 seconds
```

---

# Cache Stampede

Interview favorite.

---

Suppose:

```
Key expires
```

Then:

```
1000 users arrive
```

All see:

```
Cache Miss
```

All hit DB.

---

Result:

```
DB overwhelmed
```

---

Solution 1:

```
Distributed Lock
```

One request:

```
Rebuild Cache
```

Others wait.

---

Solution 2:

```
Refresh Ahead
```

Never let cache expire.

---

# Cache Avalanche

Many keys expire together.

---

Example:

```
1 million keys
```

TTL:

```
3600 seconds
```

All expire:

```
At same time
```

---

Result:

```
Massive DB traffic
```

---

Solution:

Add randomness.

Instead of:

```
3600
```

Use:

```
3600 + random(300)
```

---

# Cache Penetration

Requests for data that doesn't exist.

Example:

```
station:INVALID123
station:INVALID123
station:INVALID123
```

Every request:

```
Cache Miss
DB Miss
```

---

DB gets hammered.

---

Solution:

Cache NULL.

Example:

```
station:INVALID123 = NULL
TTL 5 minutes
```

---

# Redis Eviction Policies

What if Redis becomes full?

---

Policy 1:

```
noeviction
```

Reject new writes.

---

Policy 2:

```
allkeys-lru
```

Least Recently Used removed.

Most common.

---

Policy 3:

```
allkeys-random
```

Random key removed.

---

Policy 4:

```
volatile-lru
```

Only TTL keys eligible.

---

Most systems:

```
allkeys-lru
```

---

# Distributed Cache

Application Memory:

```
Server A
Server B
Server C
```

Each has own cache.

Problem:

```
Inconsistent data
```

---

Redis:

```
Server A
Server B
Server C
      ↓
     Redis
```

Shared cache.

Most modern systems use this.

---

# Real Interview Questions

### Why cache?

```
Reduce latency
Reduce DB load
Improve scalability
```

---

### Most common cache pattern?

```
Cache Aside
```

---

### Hardest part of caching?

```
Cache Invalidation
```

---

### How to prevent cache stampede?

```
Distributed Lock
Refresh Ahead
```

---

### Difference between Cache Aside and Write Through?

```
Cache Aside:
Read DB on miss

Write Through:
Update cache and DB together
```

---

# Topic Summary

```
Caching
│
├── Why Cache
├── Cache Aside
├── Cache Invalidation
├── Read Through
├── Write Through
├── Write Behind
├── Refresh Ahead
├── TTL
├── Cache Stampede
├── Cache Avalanche
├── Cache Penetration
├── Redis Eviction Policies
└── Distributed Cache
```
