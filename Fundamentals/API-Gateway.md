# What is an API Gateway?

Think of an API Gateway as the **single entry point** for all client requests.

Without API Gateway:

```
Mobile App
    ├── User Service
    ├── Payment Service
    ├── Trip Service
    └── Notification Service
```

The client must know every service URL.

With API Gateway:

```
Mobile App
      │
      ▼
+-------------+
| API Gateway |
+-------------+
      │
 ┌────┼────┬────┐
 ▼    ▼    ▼    ▼
User Payment Trip Notification
Svc   Svc   Svc      Svc
```

Client only talks to the API Gateway.

---

# Why do we need it?

Imagine Uber:

The mobile app may need:

```
GET /user/profile
GET /trips
POST /payment
POST /ride/request
```

Without a gateway:

```
api-user.company.com
api-trip.company.com
api-payment.company.com
```

The app must manage:

- URLs
- Authentication
- Rate limits
- Versioning

for every service.

An API Gateway centralizes these concerns.

---

# Main Responsibilities

## 1. Request Routing

Gateway decides:

```
/api/users/*   -> User Service
/api/trips/*   -> Trip Service
/api/payments/* -> Payment Service
```

Example:

```
GET /api/users/123
```

Gateway forwards:

```
http://user-service/users/123
```

---

## 2. Authentication

Instead of every service validating JWTs:

```
Client
  ↓
API Gateway
  ↓ Validate JWT
  ↓
Services
```

Example:

```
Authorization: Bearer xxx
```

Gateway verifies:

- Auth0 token
- OAuth token
- SSO token

before forwarding.

---

## 3. Authorization

Example:

```
Admin can access:
  /admin/*

Users cannot.
```

Gateway can block requests before they reach services.

---

## 4. Rate Limiting

Prevents abuse.

Example:

```
100 requests/minute/user
```

If user sends:

```
1000 requests/minute
```

Gateway returns:

```
429 Too Many Requests
```

---

## 5. Load Balancing

Gateway distributes traffic.

```
Trip Service

Instance 1
Instance 2
Instance 3
```

Gateway chooses where to send requests.

---

## 6. SSL Termination

Instead of HTTPS everywhere:

```
Client
  HTTPS
    ↓
Gateway
  HTTP
    ↓
Services
```

Gateway handles TLS certificates.

---

## 7. Logging

Every request can be logged.

```
User: auth0|123
Endpoint: /trips
Status: 200
Duration: 120ms
```

Useful for debugging.

---

## 8. Monitoring

Collect metrics:

```
Requests/sec
Error rate
Latency
```

Used in dashboards.

---

## 9. Request Aggregation

One client request can call multiple services.

Example:

```
GET /dashboard
```

Gateway calls:

```
User Service
Trip Service
Payment Service
```

Returns combined response.

---

# API Gateway vs Load Balancer

Many developers confuse these.

## Load Balancer

Job:

```
Send traffic to servers
```

Example:

```
Request
  ↓
Load Balancer
  ↓
Server1
Server2
Server3
```

---

## API Gateway

Job:

```
Authentication
Authorization
Routing
Rate limiting
Caching
Aggregation
Monitoring
```

Gateway is much smarter.

---

# Popular API Gateways

### Cloud Managed

- Amazon Web Services API Gateway
- Microsoft API Management
- Google API Gateway

### Open Source

- Kong
- Traefik
- NGINX
- Apache APISIX

---

# Real Example (Uber-like)

Request:

```
GET /api/trips/123
```

Flow:

```
Mobile App
    ↓
API Gateway
    ↓ Verify JWT
    ↓ Rate Limit Check
    ↓ Logging
    ↓
Trip Service
    ↓
Database
    ↓
Trip Service
    ↓
API Gateway
    ↓
Mobile App
```

---

# Problems with API Gateway

## Single Point of Failure

If gateway dies:

```
Everything dies
```

Solution:

```
Multiple Gateway Instances
```

---

## Bottleneck

All traffic goes through it.

Need:

```
Horizontal Scaling
```

---

## Increased Latency

Extra hop:

```
Client
 ↓
Gateway
 ↓
Service
```

adds some milliseconds.

---

# Interview Questions

### Why use API Gateway?

Answer:

```
Single entry point,
authentication,
routing,
rate limiting,
monitoring,
request aggregation.
```

---

### API Gateway vs Load Balancer?

Load Balancer distributes traffic.

API Gateway manages APIs and cross-cutting concerns.

---

### How does rate limiting work?

Common algorithms:

- Token Bucket
- Leaky Bucket
- Fixed Window
- Sliding Window

(learn these later)

---

### How does authentication happen?

Usually:

```
JWT
OAuth
Auth0
SSO
```

validated at the gateway.

**API Gateway can perform load balancing.**

However:

```
API Gateway ≠ Load Balancer
```

A load balancer's primary job is traffic distribution.

An API Gateway's primary job is API management (routing, auth, rate limiting, etc.).

Load balancing is often just one of its features.

---

## Scenario 1: Small System

You may have:

```
Client
  ↓
API Gateway
  ↓
User Service Instance 1
User Service Instance 2
User Service Instance 3
```

The gateway itself load balances among service instances.

No separate load balancer needed.

This is common in small systems.

---

## Scenario 2: Large Production System

Much more common:

```
Client
   ↓
Load Balancer
   ↓
API Gateway (multiple instances)
   ↓
Load Balancer
   ↓
User Service (multiple instances)
```

Example:

```
                ┌─────────────┐
Client ───────► │ LoadBalancer│
                └──────┬──────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
      Gateway1     Gateway2      Gateway3
         │             │             │
         └──────┬──────┴──────┬──────┘
                ▼             ▼
           User Service Load Balancer
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
    User1    User2    User3
```

---

## Why put a Load Balancer before API Gateway?

Suppose:

```
Gateway1
Gateway2
Gateway3
```

exist.

If Gateway1 crashes:

```
Gateway1 ❌
Gateway2 ✅
Gateway3 ✅
```

You need something to distribute traffic among gateway instances.

That's the job of the load balancer.

---

## Why put another Load Balancer after API Gateway?

Suppose User Service runs on:

```
User1
User2
User3
User4
```

The gateway may route to:

```
user-service.company.internal
```

and then a service load balancer distributes traffic.

Benefits:

- Independent scaling
- Health checks
- Failover
- Service discovery

---

## Real AWS Example

A common architecture:

```
Internet
   ↓
ALB (Load Balancer)
   ↓
API Gateway / Nginx / Kong
   ↓
ECS/Kubernetes Services
   ↓
Service Load Balancing
```

or

```
Internet
   ↓
AWS API Gateway
   ↓
Lambda Functions
```

where AWS API Gateway handles much of the load balancing internally.

---

## Think of it like a Hotel

### API Gateway = Reception Desk

Responsible for:

```
Authentication
Authorization
Routing
Rate Limiting
Monitoring
```

Example:

```
Room Service?
→ Floor 2

Laundry?
→ Basement

Restaurant?
→ Floor 1
```

---

### Load Balancer = Queue Manager

Responsible for:

```
Guest 1 → Counter 1
Guest 2 → Counter 2
Guest 3 → Counter 3
```

Its only goal is to spread traffic efficiently.

---

## Interview Answer

If an interviewer asks:

> Can API Gateway act as a Load Balancer?
> 

A strong answer is:

> Yes. Many API Gateways such as Kong, Nginx, and AWS API Gateway support load balancing. However, load balancing is only one of their capabilities. API Gateways focus on routing, authentication, rate limiting, monitoring, and API management, while dedicated load balancers focus on efficient traffic distribution and high availability. In large-scale systems, it is common to have load balancers both in front of API Gateway instances and behind them for service instances.
> 

This answer usually scores well because it shows you understand both the overlap and the distinction.

---

### Rule of Thumb

```
Load Balancer:
"Which server should receive this request?"

API Gateway:
"Which service should receive this request,
and is the request allowed?"
```

That's the key difference to remember.

when we say **multiple API Gateway instances**, we usually mean **multiple running copies of the API Gateway application**, typically on different servers, containers, or pods.

### Example: Single API Gateway Instance

```
Client
   ↓
API Gateway
   ↓
Services
```

Problem:

```
API Gateway server crashes
↓
Entire system becomes unavailable
```

---

### Multiple API Gateway Instances

```
                Load Balancer
                       ↓
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
   API Gateway 1  API Gateway 2  API Gateway 3
        ↓              ↓              ↓
                Services
```

Here:

- Gateway 1 may run on Server A
- Gateway 2 may run on Server B
- Gateway 3 may run on Server C

or

- Container 1
- Container 2
- Container 3

inside Kubernetes/ECS/Docker Swarm.

---

## Real Example with Nginx/Kong

Suppose your API Gateway is Kong or Nginx.

You deploy:

```
EC2-1 → Kong
EC2-2 → Kong
EC2-3 → Kong
```

Then put an AWS Load Balancer in front:

```
Internet
   ↓
ALB
   ↓
Kong-1
Kong-2
Kong-3
```

The load balancer distributes traffic among them.

---

## Why Multiple Instances?

### High Availability

If one gateway dies:

```
Gateway-1 ❌
Gateway-2 ✅
Gateway-3 ✅
```

Users are still served.

---

### Scalability

Suppose:

```
1 Gateway
```

handles:

```
5,000 requests/sec
```

But traffic grows to:

```
20,000 requests/sec
```

You can run:

```
4 Gateway instances
```

instead of upgrading a single machine.

This is called **horizontal scaling**.

---

## How Do They Share Configuration?

This is an important interview question.

If Gateway 1 has a route:

```
/api/users → User Service
```

then Gateway 2 and 3 must have the same route.

Usually they share:

### Option 1: Database

```
Gateway-1
Gateway-2
Gateway-3
      ↓
 Shared Config DB
```

Kong works like this.

---

### Option 2: Config Files

```
gateway-config.yaml
```

deployed to every instance.

---

### Option 3: Service Discovery

In Kubernetes:

```
Gateway Pods
      ↓
Kubernetes Service Registry
```

Routes and service locations are discovered dynamically.

---

## Relating to Your Node.js Experience

Imagine you have:

```
pm2 start api.js -i max
```

PM2 creates multiple Node.js processes:

```
Process 1
Process 2
Process 3
Process 4
```

Multiple API Gateway instances are conceptually similar, except:

- They often run on different servers/containers.
- A load balancer sits in front.
- Configuration is shared.

---

## Interview Note

When someone says:

> "We have 3 API Gateway instances"
> 

they typically mean:

```
3 running copies of the gateway application
behind a load balancer
sharing the same configuration
```

Those copies may be:

- 3 EC2 instances
- 3 Docker containers
- 3 Kubernetes pods
- 3 ECS tasks

The idea is the same: **remove the single point of failure and handle more traffic.**

**This concept connects directly to the next topics you should learn: Load Balancers, Reverse Proxies, and Service Discovery.**
