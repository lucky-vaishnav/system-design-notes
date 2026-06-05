Load Balancer is one of the most important System Design topics because almost every scalable system uses it.

# 1. What Problem Does a Load Balancer Solve?

Imagine you have:

```
Users
  |
  v
API Server
```

One server handles everything.

Problems:

- Server crashes → application down.
- Traffic increases → server becomes slow.
- Maintenance → downtime.

Solution:

```
           +----------+
Users ---> | Load     |
           | Balancer |
           +----------+
            /   |   \
           /    |    \
          v     v     v
      Server1 Server2 Server3
```

The Load Balancer distributes requests among multiple servers.

Benefits:

- High availability
- Scalability
- Fault tolerance
- Better performance

---

# 2. Real-Life Example

Suppose Uber receives:

```
100,000 requests per minute
```

One server cannot handle this.

Instead:

```
Load Balancer
    |
--------------------------------
|              |              |
API-1         API-2         API-3
```

The load balancer decides which API server receives each request.

---

# 3. Where Does It Sit?

Typical architecture:

```
Client
  |
  v
API Gateway
  |
  v
Load Balancer
  |
------------------
|       |        |
S1      S2       S3
```

API Gateway and Load Balancer serve different purposes.

### API Gateway

Focuses on:

- Authentication
- Routing
- Rate limiting
- Request transformation

### Load Balancer

Focuses on:

- Traffic distribution
- Health checks
- Failover

---

# 4. Types of Load Balancers

## Layer 4 (Transport Layer)

Works using:

```
IP + Port
```

Example:

```
TCP
UDP
```

Fast.

Does not understand URLs.

Example:

```
AWS Network Load Balancer
```

---

## Layer 7 (Application Layer)

Understands:

```
HTTP
HTTPS
Headers
URLs
Cookies
```

Example:

```
/api/users
/api/payments
```

Can route differently.

Example:

```
AWS Application Load Balancer
Nginx
HAProxy
```

Most interview questions use Layer 7.

---

# 5. Load Balancing Algorithms

## Round Robin

```
Req1 -> Server1
Req2 -> Server2
Req3 -> Server3
Req4 -> Server1
```

Most common.

---

## Weighted Round Robin

Some servers are stronger.

```
Server1 = Weight 3
Server2 = Weight 1
```

Traffic:

```
S1
S1
S1
S2
```

---

## Least Connections

Send request to server with fewer active connections.

```
Server1 = 100 connections
Server2 = 20 connections

Choose Server2
```

Good for long-running requests.

---

## Least Response Time

Send traffic to fastest server.

Used in modern cloud environments.

---

## IP Hash

```
User A -> Server1
User B -> Server2
```

Based on client IP.

Useful for session stickiness.

---

# 6. Health Checks

Load balancer constantly checks servers.

Example:

```
GET /health
```

Response:

```
{
  "status":"ok"
}
```

If server fails:

```
Server1 ❌
Server2 ✅
Server3 ✅
```

Traffic automatically stops going to Server1.

---

# 7. Active vs Passive Health Checks

## Active

Load balancer periodically pings:

```
/health
```

Most common.

---

## Passive

Observe actual traffic failures.

If too many failures:

```
Mark server unhealthy
```

---

# 8. Sticky Sessions

Without sticky sessions:

```
Login -> Server1
Next request -> Server2
```

Problem:

```
Session missing
```

because session stored on Server1.

---

### Solution 1

Sticky sessions.

```
User always goes to Server1
```

---

### Solution 2 (Better)

Store session externally.

```
Redis
Database
```

Then:

```
User -> Any server
```

Modern systems prefer this.

---

# 9. Internal vs External Load Balancer

## External

Internet-facing.

```
Users
  |
External LB
  |
Backend Servers
```

Example:

- AWS ALB
- Nginx

---

## Internal

Only inside company network.

```
Service A
   |
Internal LB
   |
Service B instances
```

Used in microservices.

---

# 10. Single Point of Failure Problem

Bad:

```
Users
  |
LB
```

If LB dies:

```
Everything down
```

---

Solution:

```
         DNS
          |
    ----------------
    |              |
   LB1            LB2
```

High availability.

Cloud providers manage this automatically.

---

# 11. Nginx as Load Balancer

Example:

```
upstream api_servers {
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000;
}

server {
    location / {
        proxy_pass http://api_servers;
    }
}
```

Nginx distributes traffic.

---

# 12. AWS Load Balancers

### ALB

Application Load Balancer

Layer 7

Supports:

- URLs
- Headers
- Hostnames

Most common.

---

### NLB

Network Load Balancer

Layer 4

Very fast.

Supports:

- TCP
- UDP

---

### Classic LB

Legacy.

Rarely used today.

---

# 13. Load Balancer in Microservices

```
Client
  |
API Gateway
  |
-----------------------
|          |          |
User LB   Ride LB   Payment LB
|          |          |
User Svc  Ride Svc Payment Svc
```

Each service may have its own load balancer.

---

# 14. Common Interview Questions

### Why use a Load Balancer?

Distribute traffic and improve availability.

---

### Difference between API Gateway and Load Balancer?

API Gateway:

- Authentication
- Routing
- Rate limiting

Load Balancer:

- Traffic distribution
- Health checks

---

### What happens if one server crashes?

Health check fails.

Load balancer removes it from rotation.

---

### What is sticky session?

User consistently routed to the same server.

---

### What is weighted round robin?

Stronger servers receive more traffic.

# Q&A

## 1. Weighted Round Robin: Does weight=3 mean first 3 requests go there?

Not necessarily, but conceptually yes.

Suppose:

```
Server A weight = 3
Server B weight = 1
```

Traffic distribution will be approximately:

```
A
A
A
B
A
A
A
B
...
```

So out of every 4 requests:

```
75% -> A
25% -> B
```

Why?

Maybe:

```
A = 8 CPU cores
B = 2 CPU cores
```

So A can handle more traffic.

Real load balancers use more sophisticated algorithms, but this is the basic idea.

---

# 2. Does every service need a /health endpoint?

Usually yes.

Example:

```
Load Balancer
     |
 -----------------
 |       |       |
API1    API2    API3
```

Every few seconds the load balancer calls:

```
GET /health
```

Response:

```
{
  "status":"ok"
}
```

If API2 returns:

```
500 Internal Server Error
```

or doesn't respond:

```
API2 = unhealthy
```

Load balancer stops sending traffic.

---

## Does it have to be publicly accessible?

No.

Very common setup:

```
Internet
   |
Load Balancer
   |
Private Network
   |
API Servers
```

Only the load balancer can call:

```
GET /health
```

Users cannot.

---

# 3. What exactly is a Session?

This is where many developers get confused.

### Example: Old School Login

User logs in.

Server creates:

```
{
userId:123,
role:"admin"
}
```

Stores it in memory:

```
sessions= {
  abc123: {
    userId:123,
    role:"admin"
  }
}
```

Browser gets:

```
sessionId = abc123
```

stored in cookie.

---

Next request:

```
Cookie: sessionId=abc123
```

Server finds:

```
sessions["abc123"]
```

and identifies the user.

---

# 4. Why does this break with multiple servers?

Suppose:

```
Load Balancer
      |
 -------------
 |           |
API1       API2
```

Login request:

```
User -> API1
```

Session stored:

```
API1memory:
{
abc123:user
}
```

Next request:

```
User -> API2
```

API2 has:

```
{}
```

No session.

User appears logged out.

---

# 5. Solution 1: Sticky Sessions

Load balancer remembers:

```
User A -> API1
```

Always sends them there.

Works but not ideal.

---

# 6. Solution 2: Redis Session Store (Better)

Instead of storing session in server memory:

```
API1
API2
API3
   |
 Redis
```

Login:

```
Redis:
{
abc123: {
userId:123,
role:"admin"
  }
}
```

Request hits any server:

```
API1
```

or

```
API2
```

Both can read:

```
redis.get("abc123")
```

No sticky sessions needed.

---

# 7. Is JWT a Session?

Modern systems often avoid server-side sessions.

For example, Auth0.

Login returns:

```
JWT Token
```

Contains:

```
{
  "sub":"auth0|123",
  "role":"admin"
}
```

Each request:

```
Authorization: Bearer xyz...
```

Server verifies token.

No session lookup.

No Redis required.

This is called:

```
Stateless Authentication
```

---

# 8. Then why do people still use Redis?

Even with JWT.

Because JWT should not store everything.

Bad:

```
{
  "cart":500 items,
  "theme":"dark",
  "preferences": ...
}
```

Token becomes huge.

Instead:

```
{
  "userId":123
}
```

Store additional data:

```
Redis
Database
```

---

# 9. In your Node.js/Auth0 project

You are mostly using:

```
Auth0 + JWT
```

which means:

```
No traditional session storage
```

Request flow:

```
User
  |
JWT
  |
API
```

Backend extracts:

```
req.sso_id
```

from token.

No Redis session required.

---

# Interview Question

Suppose interviewer asks:

> Why do we need Redis if we already have JWT?
> 

Good answer:

> JWT is used for authentication and contains limited user identity information. Redis is often used to store frequently accessed user data, shopping carts, preferences, temporary state, rate limiting counters, or server-side sessions that should not be stored inside the token.
> 

# 10. Redis is a shared service ?

Exactly. In most real-world systems, **Redis is a shared service**, not something tied to a specific API server.

Instead of:

```
Load Balancer
      |
 ----------------
 |              |
API-1         API-2

API-1 Memory -> Session A
API-2 Memory -> Session B
```

You have:

```
           Redis
             |
 -------------------------
 |           |           |
API-1      API-2      API-3
```

All servers connect to the same Redis instance (or Redis cluster).

### Example

User logs in through API-1:

```
awaitredis.set(
"session:abc123",
JSON.stringify({
    userId:100,
    role:"admin"
  })
);
```

Later, the next request reaches API-3:

```
constsession=awaitredis.get("session:abc123");
```

API-3 gets the same data because Redis is shared.

---

## Why not keep Redis inside API-1?

Bad design:

```
API-1 + Redis-1
API-2 + Redis-2
```

Now:

```
User -> API-1
Session saved in Redis-1

Next request -> API-2
Looks in Redis-2
Session not found
```

Problem returns.

---

## In Production

You'll usually see:

```
Internet
   |
Load Balancer
   |
--------------------
|        |         |
API1    API2     API3
   \      |      /
    \     |     /
      Redis
         |
     PostgreSQL
```

Think of:

- **API servers** → many instances
- **Redis** → shared cache/session store
- **Database** → shared permanent storage

---

## Is Redis always a separate server?

Usually yes.

Examples:

```
AWS ElastiCache (Redis)
Azure Redis Cache
Redis Cloud
Dedicated Redis VM
Docker Container
```

For local development:

```
Laptop
 ├─ Node.js
 ├─ PostgreSQL
 └─ Redis
```

all may run on the same machine.

For production:

```
Server A -> API
Server B -> API
Server C -> Redis
Server D -> PostgreSQL
```

or managed cloud services.

---

## Interview Note

When someone says:

> "Store sessions in Redis"
> 

they almost always mean:

> "Store sessions in a shared Redis instance that all application servers can access."
> 

That's what allows requests to hit **any API server behind the load balancer** and still find the user's session data.

### Recommended next topic

After Load Balancer, learn **Caching** (Redis, Cache-Aside, Write-Through, TTL, Cache Invalidation). It's one of the highest-frequency topics in system design interviews and appears in almost every large-scale architecture discussion.
