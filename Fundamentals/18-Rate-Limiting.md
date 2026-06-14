## What Problem Does Rate Limiting Solve?

Without rate limiting:

```
Client
 ↓
API
 ↓
Database
```

User can send:

```
1000 requests/sec
10000 requests/sec
100000 requests/sec
```

Possible consequences:

```
Server overload
Database overload
DoS attack
Brute-force attack
Excessive API cost
Abuse by a single customer
```

---

## Simple Example

Login API:

```
POST /login
```

Attacker:

```
1 million password attempts
```

Without rate limit:

```
Server processes all requests
```

With rate limit:

```
5 attempts/minute
```

Server blocks further requests.

---

# What Is Rate Limiting?

Definition:

```
Control how many requests
can be performed
during a time period.
```

Examples:

```
100 requests/minute
10 requests/second
5000 requests/day
```

---

# What Can We Rate Limit?

---

## By User

```
user_id
```

Example:

```
100 requests/min
per user
```

---

## By IP

```
192.168.1.1
```

Example:

```
50 requests/min
per IP
```

---

## By API Key

```
api_key
```

Common for public APIs.

---

## By Organization

Example:

```
Uber Org
Lyft Org
```

---

## By Endpoint

Example:

```
GET /stations
1000/min

POST /payment
10/min
```

---

# Where Can Rate Limiting Be Applied?

---

## API Gateway

Examples:

```
Kong
NGINX
AWS API Gateway
Apigee
```

Most common.

---

## Application Layer

Node.js:

```
express-rate-limit
```

---

## Redis Layer

Distributed rate limiting.

Most common for scalable systems.

---

# Core Algorithms

There are 5 major algorithms.

Interview favorite topic.

---

# Algorithm 1: Fixed Window

Example:

```
Limit:
100 requests/minute
```

Current minute:

```
10:00 - 10:01
```

Counter:

```
User A = 87
```

Request arrives:

```
88
89
90
...
100
```

Allowed.

Request:

```
101
```

Rejected.

---

Window resets:

```
10:01
```

Counter:

```
0
```

---

Implementation:

```
Redis Key:

user:123:10:00
```

Value:

```
87
```

---

Advantages:

```
Simple
Fast
Easy
```

---

Problem:

Boundary Burst.

Example:

```
100 requests
at 10:00:59

100 requests
at 10:01:00
```

Actual:

```
200 requests
within 2 seconds
```

But limiter allows it.

---

# Algorithm 2: Sliding Window Log

Stores every request timestamp.

Example:

```
10:00:01
10:00:03
10:00:05
10:00:10
```

New request:

```
Remove timestamps older than 1 minute
Count remaining
```

---

Advantages:

```
Accurate
No burst issue
```

---

Disadvantages:

```
High memory usage
```

Every request stored.

---

# Algorithm 3: Sliding Window Counter

Most practical.

Combines:

```
Fixed Window
+
Sliding Logic
```

---

Example:

```
Current Window:
70 requests

Previous Window:
90 requests
```

Uses weighted calculation.

---

Advantages:

```
Accurate
Low memory
```

---

Used by many production systems.

---

# Algorithm 4: Token Bucket

Most important interview algorithm.

---

Imagine bucket:

```
Capacity = 100 tokens
```

---

Tokens refill:

```
10/sec
```

---

Request:

```
Consumes 1 token
```

---

Example:

```
Bucket = 100
```

Request:

```
20 requests
```

Remaining:

```
80
```

---

If bucket empty:

```
Request rejected
```

---

Advantages:

```
Allows bursts
Simple
Widely used
```

---

Examples:

```
AWS
NGINX
Networking devices
```

---

# Algorithm 5: Leaky Bucket

Think:

```
Water enters quickly
```

but exits at:

```
constant rate
```

---

Example:

```
100 requests arrive instantly
```

Processed:

```
10/sec
```

Smooth output.

---

Used when:

```
Need constant flow
```

---

# Fixed Window vs Token Bucket

Interview question.

---

Fixed Window:

```
100/minute
```

No burst handling.

---

Token Bucket:

```
100 capacity
10/sec refill
```

Allows short bursts.

---

Most production systems prefer:

```
Token Bucket
```

---

# Distributed Rate Limiting

Single server:

```
Node App
```

Easy.

---

Multiple servers:

```
Server A
Server B
Server C
```

Need shared state.

---

Wrong:

```
In-memory counter
```

Each server has own count.

---

Correct:

```
Redis
```

Shared counter.

---

# Redis Rate Limiter

Key:

```
rate:user123
```

---

Request:

```
INCR rate:user123
```

---

Result:

```
1
2
3
...
```

---

Expire:

```
EXPIRE 60
```

---

Simple Fixed Window.

---

# Atomicity Problem

Wrong:

```
GET counter
counter++
SET counter
```

Race condition.

---

Correct:

```
INCR
```

Atomic operation.

---

# Rate Limiting Headers

Common APIs return:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 23
X-RateLimit-Reset: 1718131200
```

---

When blocked:

```
429 Too Many Requests
```

---

Response:

```
{
  "message":"Rate limit exceeded"
}
```

---

# Retry-After Header

Example:

```
Retry-After: 60
```

Means:

```
Try again after 60 seconds
```

---

# Real Production Examples

---

Login API

```
5 attempts/minute
```

---

OTP API

```
3 requests/5 minutes
```

---

Password Reset

```
10/hour
```

---

Public API

```
1000/day
```

---

Payment Endpoint

```
10/minute
```

Prevents abuse.

---

# Common Interview Questions

---

### Why not store counters in memory?

Because:

```
Multiple servers
Counter inconsistency
```

---

### Why Redis?

```
Fast
Atomic
Distributed
TTL support
```

---

### Most common algorithm?

```
Token Bucket
```

---

### Most accurate algorithm?

```
Sliding Window Log
```

---

### Most scalable?

```
Token Bucket
Sliding Window Counter
```

## Code Examples For Better Understadning

### 1. Fixed Window

Store:

```
{
count:3,
windowStart:"10:00:00"
}
```

Pseudo Code:

```
function allowRequest(user) {
const key=`user:${user}:10:00`;

let count=redis.get(key)||0;

if (count === 1) {
    await redis.expire(key, 60);
}

if (count>=5) {
return false;
    }

redis.incr(key);

return true;
}
```

Timeline:

```
10:00:50 -> Request #1
10:00:51 -> Request #2
10:00:52 -> Request #3
10:00:53 -> Request #4
10:00:54 -> Request #5
```

Allowed.

Now:

```
10:01:00
```

Counter resets.

Again:

```
10:01:00 -> Request #1
10:01:01 -> Request #2
10:01:02 -> Request #3
10:01:03 -> Request #4
10:01:04 -> Request #5
```

Allowed.

Actual traffic:

```
10 requests in 14 seconds
```

But limiter thinks:

```
5 in previous minute
5 in current minute
```

This is the boundary problem.

---

# 2. Sliding Window Log

Store every request timestamp.

Data:

```
[
"10:00:10",
"10:00:20",
"10:00:30",
"10:00:40"
]
```

Pseudo:

```
function allowRequest(user) {

const now=Date.now();

removeAllRequestsOlderThan(now-60 seconds);

if (requests.length>=5) {
return false;
   }

requests.push(now);

return true;
}
```

Request:

```
10:00:50
```

Stored:

```
10
20
30
40
50
```

Now count:

```
5
```

Next request:

```
10:00:51
```

Rejected.

---

At:

```
10:01:11
```

Request at:

```
10:00:10
```

falls outside window.

Now count becomes:

```
4
```

Request allowed.

---

Most accurate.

Problem:

```
Need to store every request.
```

For:

```
1 million users
```

Memory becomes huge.

---

# 3. Sliding Window Counter

Optimization of Sliding Log.

Instead of storing every request:

Store:

```
Current Minute=3
Previous Minute=5
```

Example:

```
Current time = 10:00:30
```

We are:

```
50% into current minute
```

Weighted count:

```
Current Counter
+
Previous Counter * Remaining Percentage
```

Example:

```
Current = 3
Previous = 5

3 + (5 * 0.5)

= 5.5
```

---

Pseudo:

```
weighted=
currentCount+
(previousCount*remainingWindowPercent);
```

Much less memory.

Most cloud systems use some variation of this.

---

# 4. Token Bucket

My favorite.

Think:

```
Bucket Size = 5
```

Initial:

```
Tokens = 5
```

Each request consumes:

```
1 token
```

Code:

```
function allowRequest() {

refillTokens();

if (tokens<1) {
return false;
   }

tokens--;

return true;
}
```

---

Timeline:

Initial:

```
5 tokens
```

Requests:

```
R1 -> 4
R2 -> 3
R3 -> 2
R4 -> 1
R5 -> 0
```

Now:

```
R6
```

Rejected.

---

Refill rule:

```
1 token every 12 seconds
```

After:

```
12 sec
```

Tokens:

```
1
```

One request allowed.

---

Key advantage:

Allows bursts.

Example:

```
User idle for 1 hour.
```

Bucket stays:

```
5 tokens
```

User suddenly sends:

```
5 requests immediately
```

Allowed.

---

# 5. Leaky Bucket

Think queue.

Bucket:

```
Capacity = 100
```

Processing speed:

```
1 request/sec
```

Requests arrive:

```
100 requests instantly
```

Instead of processing all:

Queue:

```
Req1
Req2
Req3
...
Req100
```

Worker processes:

```
1/sec
```

Pseudo:

```
queue.push(request);

worker() {
everysecond:
processOneRequest();
}
```

---

Output:

```
Smooth
```

Input:

```
Burst
```

---

Common use:

```
Network Routers
Traffic Shaping
Message Brokers
```

---

# Real Life Analogy

Fixed Window

```
Nightclub
Counter resets every hour
```

---

Sliding Log

```
Security guard remembers exact entry time of every person
```

---

Sliding Counter

```
Security guard remembers only recent statistics
```

---

Token Bucket

```
Coupons

You have 5 coupons.
Each entry uses one coupon.
Coupons regenerate slowly.
```

---

Leaky Bucket

```
Queue outside movie theater

100 people arrive together.

Only 1 person enters every second.
```

---

# Senior Engineer Interview Answer

If interviewer asks:

```
Which rate limiting algorithm would you use?
```

My answer:

```
Fixed Window
→ Simple but burst issue.

Sliding Window Log
→ Most accurate but memory heavy.

Sliding Window Counter
→ Good balance.

Token Bucket
→ Most common production choice.

Leaky Bucket
→ Useful when traffic smoothing is important.
```
