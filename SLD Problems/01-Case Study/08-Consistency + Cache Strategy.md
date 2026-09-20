#### My Thoughts That Time -
My thought will be that we can prevent stale Redis data from availability check. By keep updating cache on  reservation request for parking space when when we do the parking space holding . And also updating updating on cancellation and payment failure cases. And keep the DB as a source of truth at the time of the reservation as well. So doing these two things we can  still do the robust flow for the parking as as Redis can be stale. So the last uh DB will be the source of truth at the end of the reservation. And there is al- one point as well from my side. That if this Redis level approach for the checking availability is good or we can use the code level logic uh to check availability on on each availability request. So, k-se is the better one or or code level checking is the better one. So I know it will be slower but what do you think? that point also I want to raise  

Yes — these are exactly the two questions we should discuss in this section. Your overall approach is correct, but there is one important refinement.

## 1. Your Redis approach is correct, with one important distinction

You suggested updating Redis when:

* a parking space is **held**
* payment **fails**
* reservation is **cancelled**
* booking becomes **confirmed**

That's a reasonable strategy for keeping the availability cache reasonably fresh.

But I would **not make Redis the authority for the reservation decision**.

Think of it as:

```text
                    PostgreSQL
                  SOURCE OF TRUTH
                       │
              ┌────────┴────────┐
              │                 │
        Reservation         Inventory
              │                 │
              └────────┬────────┘
                       │
                  Redis cache
                       │
                 Fast availability
                     reads
```

So the responsibilities are different:

**Redis:**

> "What availability do we currently believe exists?"

**PostgreSQL:**

> "Can this reservation actually be created?"

That distinction is extremely important.

---

# 2. Example: Redis becomes stale

Suppose there is one space:

```text
DB:
available = 1

Redis:
available = 1
```

User A requests availability:

```text
Redis → 1 available
```

Then User B reserves it.

The DB transaction does:

```text
available: 1 → 0
reservation: PENDING_PAYMENT
```

And we update Redis:

```text
Redis:
1 → 0
```

So far, everything is good.

But imagine the Redis update fails:

```text
DB transaction → SUCCESS
Redis update   → FAILED
```

Now:

```text
DB    = 0
Redis = 1   ← stale
```

User C calls availability.

Redis incorrectly says:

```text
1 available
```

Should C be allowed to book?

**No.**

C can see stale availability, but when C actually attempts the reservation, PostgreSQL is the final authority.

```text
C → Availability API
       ↓
    Redis = 1
       ↓
C → Reservation API
       ↓
PostgreSQL transaction
       ↓
available = 0
       ↓
Reservation rejected
```

That's an important design principle:

> **A stale cache may cause an incorrect availability display, but it must never cause an incorrect reservation.**

---

# 3. This is why your "DB as source of truth" point is very important

The reservation flow should effectively be:

```text
Availability request
        ↓
      Redis
        ↓
  approximate/current view
```

but:

```text
Reservation request
        ↓
 PostgreSQL transaction
        ↓
 authoritative availability check
        ↓
 create PENDING_PAYMENT
```

So even if Redis says:

```text
available = 1
```

the database can still say:

```text
available = 0
```

and the reservation fails safely.

---

# 4. Now your second question: Redis vs code-level availability check

This is a very good system-design question.

When you say **"code-level checking"**, I'm assuming you mean:

```text
Availability API
       ↓
Node.js application
       ↓
PostgreSQL
       ↓
calculate availability
       ↓
return response
```

instead of:

```text
Availability API
       ↓
Redis
       ↓
return cached availability
```

### If we compare them:

|                       | Redis                          | Direct DB              |
| --------------------- | ------------------------------ | ---------------------- |
| Read latency          | Very low                       | Higher                 |
| DB load               | Low                            | High                   |
| Scalability for reads | Excellent                      | More limited           |
| Freshness             | Can be stale                   | Much fresher           |
| Complexity            | Higher                         | Simpler                |
| Infrastructure        | Redis required                 | No cache required      |
| Good for              | High-volume availability reads | Lower/moderate traffic |

So **neither is universally better**.

The decision depends heavily on the read/write ratio and scale.

---

# 5. For our parking system, I'd use Redis

Given the scale we're designing for, availability is likely to be a **very read-heavy operation**.

Imagine:

```text
100 users want to book
```

but:

```text
10,000 users are simply checking availability.
```

You don't want all 10,000 requests hitting PostgreSQL.

Instead:

```text
10,000 availability requests
          ↓
        Redis
          ↓
    fast responses

100 reservation requests
          ↓
     PostgreSQL
          ↓
 authoritative decision
```

This gives us a nice separation between:

### Read path

```text
Client
  ↓
Availability API
  ↓
Redis
  ↓
Response
```

### Write/booking path

```text
Client
  ↓
Reservation API
  ↓
PostgreSQL transaction
  ↓
Inventory + reservation
  ↓
Redis update/invalidation
```

---

# 6. But there's an even better refinement

I wouldn't necessarily cache a simple:

```text
parking_id → available_slots
```

for every scenario.

For a real parking reservation system, availability might depend on:

```text
parking lot
+
date
+
start time
+
end time
+
vehicle type
+
parking category
```

For example:

```text
parking: 101
date: 2026-09-21
start: 10:00
end: 12:00
type: EV
```

So the cache key could conceptually be something like:

```text
availability:{parkingId}:{date}:{timeRange}:{vehicleType}
```

The exact model depends on how our parking inventory is represented.

This is something we'll refine when we design the final architecture.

---

# 7. One more important issue: don't try to guarantee perfect Redis consistency

This is where I want to slightly challenge your original thought.

You said:

> "We can prevent stale Redis data by keeping it updated."

We can **reduce** staleness, but we cannot realistically guarantee that Redis will never be stale.

For example:

```text
DB transaction
     ↓
COMMIT SUCCESS
     ↓
Redis update
     ↓
Redis unavailable
```

or:

```text
DB updated
     ↓
application crashes
     ↓
Redis wasn't updated
```

or:

```text
Redis update succeeds
     ↓
response/network failure
     ↓
application doesn't know whether update completed
```

Therefore, our architecture should accept:

> **Redis is eventually consistent; PostgreSQL provides correctness.**

Then we design mechanisms to make Redis converge toward the DB state.

This leads directly into our later **Outbox + Kafka** topic.

---

# 8. The progression we're building

This is actually a very nice architecture progression:

```text
                 PostgreSQL
               Source of Truth
                     │
                     │
              Reservation
              transaction
                     │
                     ▼
              State changed
                     │
                     ▼
              Update cache
                     │
              ┌──────┴──────┐
              │             │
          immediate       async
           update         update
              │             │
            Redis       Outbox/Kafka
```

We'll eventually discuss why the **Outbox pattern** can make the DB → event → Redis synchronization much more reliable.

---

# 9. So for your interview answer

If an interviewer asks:

> "Why not simply query PostgreSQL for every availability request?"

You can say:

> "We could, and for low or moderate traffic I'd prefer the simpler direct-DB approach. But in a high-read parking system, availability checks can be much more frequent than actual reservations. To reduce database load and latency, I'd use Redis as a read cache. However, Redis is not the source of truth. The reservation transaction always validates availability against PostgreSQL and performs the atomic inventory update there. Therefore, stale Redis data can affect what availability the user sees temporarily, but it cannot result in double booking."

That's a **strong senior-level answer** because you're not blindly saying "Redis is faster, therefore Redis."

You're explaining the **trade-off and correctness boundary**.

---

## One thing I'd like you to think about next

We've established:

```text
Redis → fast availability
DB    → authoritative reservation
```

Now imagine this failure:

```text
DB transaction succeeds
        ↓
inventory becomes RESERVED
        ↓
application crashes
        ↓
Redis was NOT updated
```

Redis now says the space is available even though DB says it isn't.

**How should the system eventually correct Redis?**

That's the next piece of **Consistency + Cache Strategy** we'll tackle: **cache update/invalidation patterns, cache-aside vs write-through vs event-driven updates, and what happens when the cache update itself fails.**

### My Thoughts That Time -

According to me, the system eventually can correct the Redis when the user request came at the time. We can update the Redis, I think that is the smallest or the good to takes a way to eventually correct the Redis.

Yes, that's a valid mechanism, but I would make one important distinction:

> **Correcting Redis only when a user request arrives is a useful fallback/self-healing mechanism, but I would not use it as the primary consistency mechanism in a production system.**

Let's build this properly.

# Consistency + Cache Strategy — Part 2

We currently have:

```text
PostgreSQL
   ↓
Source of truth

Redis
   ↓
Fast availability read
```

Now consider:

```text
DB transaction
      ↓
inventory = RESERVED
      ↓
application crashes
      ↓
Redis still says AVAILABLE
```

We have a stale cache.

---

## 1. Your approach: repair on the next request

Suppose User C asks:

```text
GET /availability
```

Redis says:

```text
AVAILABLE
```

We could then check PostgreSQL and discover:

```text
DB → RESERVED
```

Then:

```text
DB result
   ↓
update Redis
   ↓
return correct availability
```

So the system self-heals.

That's a valid technique, often called **lazy/self-healing cache correction**.

### But there's a problem

If every availability request does:

```text
Redis
  ↓
DB
  ↓
compare
  ↓
possibly update Redis
```

then we're effectively hitting the DB again.

And we've lost much of the benefit of Redis.

For example:

```text
10,000 availability requests
        ↓
Redis
        ↓
10,000 DB checks
```

At that point, why have Redis?

So we need something better.

---

# 2. The usual production approach: cache + asynchronous correction

Instead, we can make the normal path:

```text
Availability request
        ↓
      Redis
        ↓
    fast response
```

And independently make sure Redis eventually converges with PostgreSQL.

Conceptually:

```text
                  PostgreSQL
                Source of Truth
                       │
                       │ state change
                       ▼
                    Event
                       │
                       ▼
                    Redis
                       │
                       ▼
               Updated availability
```

This is where **event-driven cache invalidation/update** comes in.

We'll eventually implement this using the **Outbox + Kafka** design in topic #4.

---

# 3. But before Kafka, let's understand the simpler options

There are several cache strategies.

### Option A — Cache-aside

Application controls the cache.

Read:

```text
Request
  ↓
Redis
  │
  ├── HIT → return
  │
  └── MISS
        ↓
       DB
        ↓
    update Redis
        ↓
      return
```

Write:

```text
Reservation
    ↓
DB transaction
    ↓
success
    ↓
invalidate/update Redis
```

This is probably the easiest strategy to start with.

---

# 4. The dangerous part of cache-aside

Suppose:

```text
DB = AVAILABLE
Redis = AVAILABLE
```

User A books it.

We do:

```text
BEGIN
 ↓
DB → RESERVED
 ↓
COMMIT
 ↓
Redis update
```

But Redis update fails.

Now:

```text
DB    = RESERVED
Redis = AVAILABLE
```

That's okay **if PostgreSQL remains authoritative for booking**.

But now another user sees:

```text
AVAILABLE
```

and tries to book.

The reservation transaction checks DB:

```text
DB → RESERVED
```

and rejects it.

So correctness remains intact.

---

# 5. What if we update Redis BEFORE DB?

This is where things become dangerous.

Suppose:

```text
Redis → RESERVED
DB     → update fails
```

Now:

```text
Redis = RESERVED
DB    = AVAILABLE
```

Users may unnecessarily see no availability even though the space is actually available.

That's a **false negative**.

It hurts availability/user experience, but doesn't cause double booking.

If instead we allow Redis to make the actual reservation decision, stale data can cause much worse problems.

Therefore:

> **Never make Redis availability the final authorization for a reservation.**

---

# 6. Update vs invalidate

There's another design decision.

After the DB changes, instead of:

```text
Redis:
available = 10
       ↓
available = 9
```

we could simply:

```text
DEL availability:key
```

Then the next request does:

```text
Redis MISS
   ↓
DB
   ↓
calculate fresh availability
   ↓
populate Redis
```

This is **cache invalidation**.

It can actually be safer and simpler than trying to keep every cached value perfectly synchronized.

So:

```text
DB changes
    ↓
invalidate Redis
```

rather than:

```text
DB changes
    ↓
calculate exact new cache value
    ↓
update Redis
```

---

# 7. Why invalidation can be attractive

Imagine availability depends on:

```text
parking lot
+
date
+
time range
+
vehicle type
+
multiple reservations
```

Trying to update every affected Redis key correctly can become complicated.

For example:

```text
Reservation created
       ↓
Which cache keys are affected?
       ├── 10:00–11:00
       ├── 10:30–11:30
       ├── 11:00–12:00
       └── ...
```

Instead, we can invalidate the relevant availability cache entries.

Then:

```text
next request
    ↓
cache miss
    ↓
DB
    ↓
fresh calculation
    ↓
Redis
```

This reduces the risk of incorrect cache calculations.

---

# 8. So our current design could be

### Availability read

```text
Client
  ↓
Availability API
  ↓
Redis
  │
  ├── HIT → return cached result
  │
  └── MISS
        ↓
       DB
        ↓
     calculate
        ↓
    populate Redis
        ↓
      return
```

### Reservation write

```text
Client
  ↓
Reservation API
  ↓
PostgreSQL transaction
  ↓
Reserve inventory
  ↓
COMMIT
  ↓
Invalidate/update Redis
```

And if Redis fails:

```text
DB = correct
Redis = temporarily stale
```

The system remains correct because:

```text
Reservation → DB validation
```

---

# 9. Where your "repair when user requests" idea fits

I wouldn't throw your idea away.

It's useful as an additional safety mechanism:

```text
Normal:
Redis → serve availability

Failure:
Redis stale
   ↓
TTL / invalidation / event correction
   ↓
Eventually consistent
```

And potentially:

```text
Rare inconsistency detected
        ↓
DB verification
        ↓
repair Redis
```

So your idea is good as a **self-healing fallback**, but not as the mechanism that verifies every normal availability request.

---

# 10. We now have an important architecture principle

For our system:

```text
              ┌─────────────┐
              │ PostgreSQL  │
              │   Truth     │
              └──────┬──────┘
                     │
               state changes
                     │
                     ▼
              cache invalidation
                     │
                     ▼
                ┌────────┐
                │ Redis  │
                │  Read  │
                │ Cache  │
                └────────┘
```

And:

```text
READ:
Redis first

WRITE:
PostgreSQL first

BOOKING:
PostgreSQL decides

CACHE:
Eventually consistent
```

That is the mental model I want you to retain.

---

## One more piece before we leave this topic

There's a race condition we should discuss next:

```text
Request A → Redis MISS
Request B → Redis MISS
Request C → Redis MISS
        ↓
all three hit PostgreSQL
        ↓
all three try to populate Redis
```

This is called a **cache stampede / thundering herd** problem.

Then we need to discuss:

* TTL
* cache stampede
* cache penetration
* cache avalanche
* distributed locking / request coalescing
* stale-while-revalidate
* what TTL makes sense for parking availability

**That should be the next part of Consistency + Cache Strategy.**

Absolutely. Let's continue with the **cache failure and high-concurrency side** of our Consistency + Cache Strategy.

# Consistency + Cache Strategy — Part 3

## Cache Stampede, TTL, Penetration & Avalanche

We currently have:

```text
                PostgreSQL
              Source of Truth
                    │
             DB state changes
                    │
                    ▼
             Redis cache
                    │
                    ▼
          Availability requests
```

The normal read path is:

```text
Request
   ↓
Redis
   │
   ├── HIT → return
   │
   └── MISS
        ↓
       DB
        ↓
    populate Redis
        ↓
      return
```

That looks fine.

But under high traffic, several problems appear.

---

# 1. Cache Stampede / Thundering Herd

Let's take a parking lot whose availability cache has expired.

Before expiry:

```text
Redis:
availability = 25
TTL = 5 minutes
```

Now TTL expires.

At almost exactly the same moment, 1,000 users request availability:

```text
R1 ─┐
R2 ─┤
R3 ─┤
... │
R1000 ┘
     ↓
   Redis
     ↓
   MISS
```

All 1,000 requests see:

```text
CACHE MISS
```

And if our code simply does:

```text
Redis MISS
   ↓
query PostgreSQL
```

we get:

```text
1,000 requests
      ↓
1,000 DB queries
```

The cache was supposed to **protect PostgreSQL**, but when it expired, everyone suddenly hit PostgreSQL.

That's the **cache stampede** / **thundering herd** problem.

---

# 2. How do we handle it?

There are several approaches.

### Approach 1 — Distributed lock

When the first request sees a cache miss:

```text
R1 → Redis MISS
      ↓
   acquire lock
      ↓
   query DB
      ↓
   populate Redis
      ↓
   release lock
```

Meanwhile:

```text
R2 → Redis MISS → waits
R3 → Redis MISS → waits
R4 → Redis MISS → waits
```

Once R1 populates Redis:

```text
Redis = fresh availability
```

The others can read it.

So instead of:

```text
1,000 DB queries
```

we get approximately:

```text
1 DB query
999 Redis reads
```

This is a big improvement.

---

# 3. But do we actually need a distributed lock?

Not necessarily.

For our availability API, another option is **request coalescing** at the application layer.

For example, within one Node.js instance:

```text
Map<cacheKey, Promise>
```

Conceptually:

```text
R1 → MISS → starts DB request
R2 → MISS → sees existing promise
R3 → MISS → waits for same promise
R4 → MISS → waits for same promise
```

Then:

```text
             DB
              ↑
              │
R1 ───────────┤
R2 ───────────┤ same request
R3 ───────────┤
R4 ───────────┘
```

But there's a limitation:

> This only protects requests handled by the **same Node.js process**.

If we have:

```text
Node 1
Node 2
Node 3
Node 4
```

then each process could independently hit PostgreSQL.

For a distributed system, a Redis-based distributed lock or another coordination mechanism can be considered.

---

# 4. Another approach: stale-while-revalidate

This is particularly interesting for availability.

Instead of immediately treating an expired cache as unusable, we can allow a **slightly stale value** while refreshing it in the background.

For example:

```text
Fresh TTL       = 30 seconds
Stale window    = 30 seconds
```

So:

```text
0–30 sec
    ↓
fresh → return

30–60 sec
    ↓
slightly stale
    ↓
return existing value
    +
background refresh

>60 sec
    ↓
hard expiration
    ↓
DB required
```

This can dramatically reduce stampedes.

However, remember our earlier principle:

> The availability response can be slightly stale; the reservation decision cannot.

So this technique is more acceptable for **display/search availability** than for the actual booking transaction.

---

# 5. TTL — how long should our availability cache live?

There's no universal number.

Suppose we choose:

```text
TTL = 5 minutes
```

That's potentially too stale for a high-demand parking lot.

Imagine:

```text
10:00 → Redis says 20 spaces
10:01 → 10 reservations happen
10:02 → Redis still says 20
```

Users see incorrect availability for several minutes.

If we choose:

```text
TTL = 5 seconds
```

we get fresher data, but:

```text
more cache misses
      ↓
more DB reads
```

So TTL is a **freshness vs database-load trade-off**.

For our system, I'd start with a relatively short TTL for availability—something like **tens of seconds**, not several minutes—and then tune it using actual traffic and staleness requirements.

But the important thing is:

**TTL is not our correctness mechanism.**

DB validation remains the correctness mechanism.

---

# 6. What about explicit invalidation?

We already discussed:

```text
Reservation created
       ↓
DB commit
       ↓
invalidate Redis
```

That means we don't have to wait for TTL.

For example:

```text
Redis:
available = 20

User books
   ↓
DB:
available = 19
   ↓
DEL Redis key
```

Next availability request:

```text
Redis MISS
   ↓
DB
   ↓
19
   ↓
Redis = 19
```

This gives us much fresher data than relying only on TTL.

So I would combine:

```text
Event/change → invalidate/update cache
                    +
                 TTL fallback
```

Why TTL?

Because invalidation itself can fail.

---

# 7. Cache penetration

Now another problem.

Suppose someone repeatedly asks for:

```text
parking_id = 999999999
```

which doesn't exist.

Our application does:

```text
Redis MISS
   ↓
DB
   ↓
not found
```

Then the next request does exactly the same thing.

An attacker or buggy client can generate:

```text
100,000 requests
      ↓
100,000 DB queries
```

even though the data doesn't exist.

This is **cache penetration**.

### Solution

We can cache negative results for a short period:

```text
parking:999999999
    ↓
NOT_FOUND
    ↓
TTL = short
```

Then repeated requests don't reach PostgreSQL.

Other defenses include:

* API validation
* rate limiting
* valid ID/key validation
* Bloom filters in very large datasets

For our system, **short-lived negative caching + rate limiting** is likely sufficient; we don't need to introduce Bloom filters without a concrete scale requirement.

---

# 8. Cache avalanche

Now imagine we have millions of keys:

```text
key1 → TTL 5 min
key2 → TTL 5 min
key3 → TTL 5 min
...
```

If they were populated around the same time, they can expire around the same time:

```text
          TTL expires
               ↓
┌───────────────────────────┐
│ key1 key2 key3 ... key1M  │
└───────────────────────────┘
               ↓
        huge DB traffic
```

That's a **cache avalanche**.

One simple mitigation is **TTL jitter**.

Instead of:

```text
TTL = 30 seconds
```

use something like:

```text
30 ± random few seconds
```

Conceptually:

```text
key1 → 28 sec
key2 → 33 sec
key3 → 31 sec
key4 → 36 sec
```

So expirations are spread out.

---

# 9. Putting these together for our parking system

I'd currently design the availability layer approximately like this:

```text
                    Client
                      │
                      ▼
              Availability API
                      │
                      ▼
                    Redis
                 /          \
             HIT              MISS
              │                 │
              │                 ▼
              │              DB query
              │                 │
              │                 ▼
              │            populate Redis
              │                 │
              └────────┬────────┘
                       ▼
                    Response
```

With:

```text
Cache invalidation
      ↑
      │
Reservation / cancellation /
payment failure / expiry
      │
      ▼
PostgreSQL
```

And protection mechanisms:

```text
Redis
 │
 ├── TTL
 ├── TTL jitter
 ├── negative caching
 ├── stampede protection
 └── optional stale-while-revalidate
```

---

# 10. But there's a critical question

Suppose we have:

```text
Redis = 10 available
DB    = 5 available
```

User calls:

```text
GET /availability
```

and Redis returns:

```text
10 available
```

Then the user immediately clicks **Reserve**.

Our reservation API does:

```text
DB transaction
   ↓
SELECT ... FOR UPDATE
   ↓
actual availability = 5
```

That's perfectly fine.

But suppose **all 5 spaces are already being reserved concurrently**.

The availability response may have said:

```text
5 available
```

yet the reservation can still return:

```text
NO AVAILABILITY
```

That's not necessarily an error in our system.

It's an inherent property of a highly concurrent reservation system.

We should communicate this correctly to the client:

> Availability is a point-in-time estimate; reservation is the authoritative operation.

This is an important distinction for our final API/design.

---

# 11. Our consistency model is now becoming clear

For this system:

| Operation            | Consistency                     |
| -------------------- | ------------------------------- |
| Search parking lots  | Eventual consistency acceptable |
| Availability display | Slightly stale acceptable       |
| Reservation creation | **Strong consistency required** |
| Inventory decrement  | **Strong consistency required** |
| Payment state        | **Strong consistency required** |
| Booking confirmation | **Strong consistency required** |
| Analytics            | Eventual consistency acceptable |

This is much better than saying:

> "Our entire system is strongly consistent."

We don't need strong consistency everywhere.

We need it **where correctness matters**.

---

## Where we are now

We've covered the major cache behavior:

```text
✅ Redis vs direct DB
✅ DB as source of truth
✅ Cache-aside
✅ Cache invalidation
✅ Stale cache
✅ Self-healing
✅ TTL
✅ Cache stampede
✅ Thundering herd
✅ Negative caching / penetration
✅ Cache avalanche
✅ TTL jitter
✅ Stale-while-revalidate
```

### One final piece before moving to next topic

I'd like to close this section with **one important race condition: cache update ordering**.

For example:

```text
Request A:
DB update
Redis update

Request B:
DB update
Redis update
```

What happens if the operations reach Redis in the wrong order?

You can end up with:

```text
DB = 8
Redis = 9   ← older state
```

even though both individual cache updates succeeded.

That leads directly into **event ordering, versioning, and why Outbox/Kafka becomes useful**.

Yes — this is the final piece we should cover before marking **Consistency + Cache Strategy complete**.

And your sequence is correct:

```text
Request A: DB update → Redis update
Request B: DB update → Redis update
```

The important part is that **DB updates and Redis updates are separate operations**. Even though our application code says them in that order, the Redis updates from concurrent requests can be observed in a different order.

### Simple example

Suppose initially:

```text
DB available_slots = 10
Redis available_slots = 10
```

Two users book at almost the same time.

**Request A** books one slot:

```text
A: DB 10 → 9
```

**Request B** books another slot:

```text
B: DB 9 → 8
```

So finally:

```text
DB = 8
```

Now imagine the Redis operations don't reach Redis in the same order:

```text
B: Redis → 8
A: Redis → 9
```

Final state:

```text
DB    = 8  ✅
Redis = 9  ❌ stale
```

So the issue isn't that **Request A's code intentionally did Redis before DB**.

It's that:

```text
A DB update
B DB update
B Redis update
A Redis update
```

can happen when requests are concurrent.

---

### How do we protect against this?

One common approach is to attach a **version** to the inventory state.

For example:

```text
DB:
available_slots = 8
version = 102
```

Request A may produce:

```text
available_slots = 9
version = 101
```

Request B produces:

```text
available_slots = 8
version = 102
```

When writing to Redis, we only accept a value if its version is newer than the version already stored.

So if Redis receives:

```text
version 102 → slots 8
```

and later:

```text
version 101 → slots 9
```

Redis rejects the older update.

Final:

```text
DB:
slots = 8
version = 102

Redis:
slots = 8
version = 102
```

This is **versioned cache updates / ordering protection**.

---

### But there is an even simpler approach

For our parking system, I'd actually prefer **cache invalidation** in many cases:

```text
DB transaction succeeds
        ↓
DEL Redis availability key
```

Then we don't have to worry about whether we're writing `8`, `9`, `7`, etc.

The next availability request does:

```text
Redis MISS
    ↓
Read DB
    ↓
Get current value = 8
    ↓
Populate Redis = 8
```

So the cache naturally repairs itself.

This is one reason I said earlier that **invalidation can be safer than trying to update the cache with calculated values**, especially when availability has multiple dimensions such as date, time range, vehicle type, etc.

---

### Where Outbox/Kafka comes in

There is still another problem:

```text
DB transaction succeeds
        ↓
Redis update/invalidation
        ↓
Redis is DOWN
```

Now the DB is correct but Redis wasn't updated.

That's okay for correctness because DB is authoritative, but we want the cache to eventually become correct.

Later, with **Outbox + Kafka**, we can do:

```text
DB Transaction
      │
      ├── Reservation updated
      │
      └── Outbox event created
                ↓
             Kafka
                ↓
       Cache update/invalidation
```

The event can carry the version, allowing consumers to ignore older events.

We'll cover the actual **Outbox + Kafka design** later in the agreed architecture section, so we don't need to go deep into it right now.

### Final mental model for this section

```text
                 READ
                  │
             Redis first
                  │
          ┌───────┴───────┐
         HIT             MISS
          │                │
       return             DB
                           │
                      populate Redis
                           │

                 WRITE / BOOK
                  │
             PostgreSQL
                  │
        source of truth
                  │
       invalidate/update Redis
                  │
          eventual consistency
```

And the interview line I'd remember is:

> **Redis improves read performance, but PostgreSQL determines booking correctness. Cache consistency is eventually maintained through invalidation/update, TTL, and eventually event-driven repair.**

So yes, **Consistency + Cache Strategy is now complete.**

Next in our sequence is **— Payment Failures + Reconciliation**, which is a very important part of this parking system because payment is an external dependency and introduces several failure scenarios.

Yes — your concern is valid, but there are **two different problems** getting mixed together: **cache invalidation causing a burst of DB reads** and **cache penetration**. They are not exactly the same.

### Your question

> **If we invalidate the availability cache after every reservation, and there are hundreds or thousands of reservations happening concurrently, could repeated invalidation cause a large number of Redis misses and therefore a sudden spike in database reads? Would it be better to maintain and update real-time availability in Redis instead of frequently invalidating cache keys, and how do we avoid stale Redis data?**

### Your reasoning

Suppose we have:

```text
Redis:
availability:lot1:10am-11am = 20
availability:lot1:10am-12pm = 35
availability:lot1:11am-12pm = 15
...
```

Now 1,000 reservations happen.

If every reservation does:

```text
DB transaction
   ↓
DELETE Redis key
```

then you could have many requests doing:

```text
Redis MISS
   ↓
DB query
   ↓
populate Redis
```

So yes, **if the invalidated keys are immediately requested by a large number of users, you can create a DB load spike.**

But I would **not call this cache penetration**.

* **Cache penetration:** requests repeatedly ask for data that does not exist, so every request bypasses the cache and hits the DB.
* **Cache invalidation storm / cache stampede:** valid cached data is invalidated or expires, and many requests simultaneously go to the DB.

Your scenario is closer to **cache stampede / DB load amplification after invalidation**.

---

## Should we instead update Redis in real time?

For this parking system, I would use a **hybrid approach**, rather than choosing only one.

### Option 1 — Invalidate

```text
Reservation
    ↓
PostgreSQL
    ↓
Invalidate Redis
```

Next request:

```text
Redis MISS
    ↓
PostgreSQL
    ↓
Redis
```

**Pros:**

* Very simple
* Less chance of writing an incorrect calculated value
* DB remains authoritative
* Naturally self-healing

**Problem:**

* Heavy invalidation can cause many DB reads.

---

### Option 2 — Update Redis directly

After successful DB reservation:

```text
DB: available = 100 → 99
        ↓
Redis: available = 100 → 99
```

Then the next user gets the new value directly from Redis.

This is better for **high-read/high-write availability data**, but now you have to deal with:

* concurrent Redis updates
* ordering
* failed Redis updates
* stale values
* multiple cache keys representing the same inventory

For example, one reservation might affect:

```text
lot1:today
lot1:today:10-11
lot1:today:10-12
lot1:today:EV
lot1:today:car
...
```

Updating every derived key correctly becomes complicated.

---

## What I'd choose for our case study

I'd design it like this:

```text
                  PostgreSQL
                Source of Truth
                     │
             Reservation succeeds
                     │
              Outbox/Event
                     │
                     ↓
                   Kafka
                     │
                     ↓
             Cache update/invalidation
                     │
                     ↓
                   Redis
```

And for the **hot availability data**, Redis can maintain a near-real-time availability representation.

So we aren't saying:

> "Every reservation must delete Redis and make the next request hit PostgreSQL."

Instead:

> **PostgreSQL establishes the authoritative inventory state, and Redis is asynchronously maintained as a high-performance read model.**

That gives us:

```text
PostgreSQL
   ↓
authoritative state

Redis
   ↓
fast availability read model
   ↓
may briefly be stale
```

And when Redis is temporarily wrong, **booking still goes to PostgreSQL and performs the authoritative availability check.**

### One important correction to your thinking

You said:

> "we just need to create real-time availability and then update all the Redis keys"

I'd slightly change that.

We **don't necessarily want to update every possible Redis key in real time**. That can become expensive and complicated because there may be many overlapping availability queries.

Instead, we can maintain a **canonical Redis availability representation**, and derive/cache specific query results as needed.

For example:

```text
Redis canonical inventory
        ↓
availability calculation
        ↓
query-specific cache
```

Or, depending on the actual inventory model, use Redis structures such as counters/sets for the hot inventory dimension.

So the senior-level principle is:

> **Don't use cache invalidation blindly when the cache is a high-throughput read model. For hot availability data, maintain Redis asynchronously from the authoritative DB state, while keeping PostgreSQL as the final booking authority.**

And yes, this leads very naturally into our next topic: **payment failures and reconciliation**, because there we'll see another case where the DB and an external system can temporarily disagree and we need to repair the state.

