# 5.2 Application/API Scaling

## 5.2.1 First: What problem are we solving?

Our application currently looks like:

```text
                    Clients
                       |
                       v
                Load Balancer
                       |
          +------------+------------+
          |            |            |
          v            v            v
       Node.js      Node.js      Node.js
       Instance 1   Instance 2   Instance 3
          |            |            |
          +------------+------------+
                       |
                Redis / PostgreSQL
```

We established that our system has approximately:

* **1M searches/min** → ~16.7K requests/sec
* **100K reservations/min** → ~1.67K requests/sec
* **20K payments/min** → ~333 requests/sec

The first application-level problem is:

> **One Node.js instance cannot indefinitely handle increasing traffic.**

Eventually we hit a resource limit.

---

# 5.2.2 What can make a Node.js API instance a bottleneck?

There are several possibilities.

### CPU

For example:

```text
CPU → 95%
```

Possible causes:

* expensive calculations
* JSON serialization/deserialization
* large responses
* synchronous/blocking code
* CPU-heavy business logic

### Memory

```text
Memory → 90%+
```

Possible causes:

* memory leaks
* huge objects
* large caches in process memory
* unnecessarily large request/response payloads

### Network

The application may simply be processing too much network traffic.

### Connections

The Node.js instance may have limited:

* DB connections
* Redis connections
* HTTP connections to external services

### Event-loop blocking

This is particularly important for Node.js.

Even if the machine has multiple CPU cores, one Node.js process has a primary JavaScript execution thread.

If we do something like:

```js
while (veryLargeCalculation) {
   // CPU-heavy work
}
```

we can block the event loop.

Then unrelated requests start waiting.

So before scaling:

> **We need to know what resource is actually saturated.**

---

# 5.2.3 Vertical scaling vs Horizontal scaling

There are two basic approaches.

### Vertical scaling

Make the machine bigger:

```text
2 CPU / 4 GB
      ↓
8 CPU / 16 GB
```

Advantages:

* simple
* no application architecture change
* useful as an immediate step

Problems:

* hardware has limits
* larger machines become expensive
* still leaves a single-instance failure risk
* doesn't provide the same elasticity as horizontal scaling

---

### Horizontal scaling

Add more application instances:

```text
1 Node.js
   ↓
5 Node.js
   ↓
20 Node.js
   ↓
50 Node.js
```

This is generally the primary scaling strategy for stateless APIs.

---

# 5.2.4 Why does statelessness matter?

Suppose User A sends:

```text
Request 1 → Node 1
```

Then:

```text
Request 2 → Node 3
```

If Node 3 needs information that exists only inside Node 1's memory, we have a problem.

For example:

```js
const sessions = new Map();
```

Node 1:

```text
User A → session data
```

Node 3:

```text
User A → ??? 
```

This makes horizontal scaling difficult.

### Production approach

Keep shared state outside the application process.

For example:

```text
Node.js
   |
   +---- PostgreSQL
   |
   +---- Redis
   |
   +---- Object Storage
```

Then any instance can handle any request.

That's what we mean by a **stateless application tier**.

---

# 5.2.5 What should NOT be stored only in Node.js memory?

Be careful with things such as:

* user sessions
* authentication state
* reservation state
* distributed locks
* shared counters
* important cache data
* workflow state

These need an appropriate shared/durable mechanism.

However, **local memory isn't forbidden**.

We can still use process-local memory for things like:

```text
configuration
small immutable reference data
short-lived optimization caches
compiled objects
```

as long as correctness doesn't depend on it.

This is an important distinction:

> **Stateless doesn't mean the process cannot have memory. It means critical shared state doesn't depend on one particular instance.**

---

# 5.2.6 Load Balancer

Once we have multiple instances:

```text
                Clients
                   |
                   v
             Load Balancer
              /    |    \
             /     |     \
            v      v      v
          Node1  Node2  Node3
```

The load balancer distributes incoming requests.

Common strategies conceptually include:

### Round robin

```text
Request 1 → Node1
Request 2 → Node2
Request 3 → Node3
Request 4 → Node1
```

Simple, but doesn't consider actual instance load.

### Least connections

Send traffic toward the instance with fewer active connections.

Useful when request durations vary.

### Weighted routing

```text
Node1 → 50%
Node2 → 30%
Node3 → 20%
```

Useful when instances have different capacity.

### Hash-based routing

For example, based on some client/resource key.

Can provide affinity, but we generally don't want to depend on it unless there's a specific reason.

---

# 5.2.7 Do we need sticky sessions?

Sticky sessions mean:

> Once a user reaches Node 1, try to keep sending that user's requests to Node 1.

Example:

```text
User A → Node 1
User A → Node 1
User A → Node 1
```

This can help legacy stateful applications.

But for our architecture, we'd prefer:

```text
User A → Node 1
User A → Node 3
User A → Node 7
```

and everything still works.

Why?

Because shared state is already in:

```text
Redis / PostgreSQL / other shared systems
```

### Production preference

**Avoid sticky sessions when you don't need them.**

They can make:

* scaling less flexible
* failover more complicated
* load distribution less even

---

# 5.2.8 Health checks

Now suppose:

```text
Node 2 → crashed
```

The load balancer shouldn't continue sending traffic there.

So we have health checks.

Conceptually:

```text
Load Balancer
     |
     +---- Node1 → healthy
     |
     +---- Node2 → unhealthy ❌
     |
     +---- Node3 → healthy
```

The unhealthy instance is removed from traffic.

But there's an important production distinction:

### Liveness

> Is the process alive?

### Readiness

> Is the instance actually ready to receive traffic?

A Node.js process could technically be alive while:

* DB connection is unavailable
* required configuration isn't loaded
* application initialization isn't complete
* critical dependency is unavailable

So **readiness** is generally more useful for traffic routing.

---

# 5.2.9 Graceful shutdown

Now consider deployment.

We have:

```text
Node1
Node2
Node3
```

We're deploying a new version.

We don't want:

```text
Node1 receives request
      ↓
deployment kills Node1
      ↓
request abruptly terminates
```

Instead:

```text
Remove Node1 from load balancer
          ↓
Stop accepting new requests
          ↓
Allow in-flight requests to finish
          ↓
Close DB/Redis connections
          ↓
Terminate process
```

That's **graceful shutdown**.

For Node.js, this is particularly important because we may have:

* HTTP requests
* DB queries
* Redis operations
* external HTTP calls
* background work

in progress.

---

# 5.2.10 Autoscaling

Suppose normal traffic is:

```text
5K req/sec
```

and peak traffic becomes:

```text
20K req/sec
```

We don't necessarily want 50 instances running all day.

Instead:

```text
Normal:
5 instances

Traffic increases:
10 instances

Peak:
25 instances

Traffic decreases:
8 instances
```

That's autoscaling.

But here's an important senior-level point:

> **Don't autoscale purely on CPU.**

For a backend service, useful signals may include:

* CPU
* memory
* request rate
* request latency
* active requests
* queue depth
* event-loop utilization
* DB connection pool saturation

For example, a Node.js service could have:

```text
CPU = 40%
```

but:

```text
DB connection pool = 100% utilized
```

Adding more Node.js instances may make the DB problem worse.

So autoscaling must consider **downstream capacity**.

---

# 5.2.11 The scaling trap

This is one of the most important things to understand.

Imagine:

```text
5 Node.js instances
        ↓
PostgreSQL
```

DB can handle the workload.

Traffic increases.

We scale:

```text
5 → 20 Node.js instances
```

Great?

Not necessarily.

Now:

```text
20 Node.js instances
        ↓
PostgreSQL
```

Each instance has its own connection pool.

Suppose:

```text
pool = 50
```

Potential connections:

```text
20 × 50 = 1,000
```

Now PostgreSQL becomes saturated.

So:

> **Scaling the application layer can overload the database layer.**

This is why scaling must be viewed as a **system**, not as individual components.

---

# 5.2.12 Production approach for our Node.js services

For our parking system, I'd aim for:

```text
                    Load Balancer
                         |
              +----------+----------+
              |          |          |
              v          v          v
           Node.js    Node.js    Node.js
           Instance   Instance   Instance
              |          |          |
              +----------+----------+
                         |
                Shared infrastructure
                  /       |       \
                 v        v        v
              Redis   PostgreSQL  Kafka
```

Application instances should be:

* stateless
* horizontally scalable
* independently deployable
* health-checked
* gracefully shut down
* autoscaled based on meaningful metrics
* protected by timeouts
* protected from uncontrolled retries

And importantly:

> **Scaling the application tier must be coordinated with DB, Redis, Kafka, and external-provider capacity.**

---

# 5.2.13 One more production concern: deployment scaling

Suppose:

```text
10 old instances
```

and we deploy:

```text
10 new instances
```

For a short period, we may have:

```text
20 instances
```

That can temporarily double:

* DB connections
* Redis connections
* downstream requests

So deployments themselves can create load spikes.

This is why production systems use approaches such as:

* rolling deployments
* controlled instance replacement
* readiness checks
* connection draining
* graceful shutdown
* controlled autoscaling

---

# 5.2.14 What happens if one instance dies?

Because our application tier is stateless:

```text
Before:

LB
├── Node1
├── Node2
└── Node3
```

Node2 dies:

```text
LB
├── Node1
└── Node3
```

Requests continue through the remaining instances.

This is the beginning of **High Availability**, which we'll cover later in **5.7**.

But notice the connection:

> **Statelessness makes horizontal scaling easier and also improves failure handling.**

That's why these concepts belong together.

---

# What to remember for ANY system-design problem

When you reach the API/application layer, use this checklist:

```text
1. Is the service stateless?
       ↓
2. Can it scale horizontally?
       ↓
3. How is traffic distributed?
       ↓
4. How do we detect unhealthy instances?
       ↓
5. How do we deploy without dropping requests?
       ↓
6. What metrics trigger scaling?
       ↓
7. What downstream dependency limits scaling?
       ↓
8. What happens when one instance fails?
```

And the most important principle:

> **Horizontal scaling is useful only when the work and state can actually be distributed.**

---

## Status

We've covered the core of **Application/API Scaling**.

There are two deeper areas I want to cover:

1. **Node.js-specific scaling** — event loop, worker processes/cluster, CPU-bound work, PM2/container replicas, and why multiple processes matter.
2. **Capacity planning** — how to estimate how many instances we actually need instead of saying "add more servers."

---

# 5.2.15 Node.js-Specific Scaling

This is particularly important for you because you're preparing for senior Node.js interviews.

We need to distinguish between:

* **scaling Node.js across machines/containers**
* **scaling processes within one machine**

---

## Problem: One Node.js process doesn't use all CPU cores for JavaScript execution

Suppose we have a server with:

```text
8 CPU cores
32 GB RAM
```

and run:

```text
1 Node.js process
```

For normal I/O-heavy APIs, that's not necessarily a problem.

Node.js is very good at handling many concurrent I/O operations because the event loop doesn't wait synchronously for:

* DB queries
* Redis
* HTTP calls
* network I/O

But consider CPU-heavy work:

```text
Request
   ↓
Large computation
   ↓
CPU-intensive JavaScript
   ↓
Event loop blocked
   ↓
Other requests wait
```

Now our single process can become a bottleneck.

---

# 5.2.16 Multiple Node.js Processes

We can run multiple Node.js processes on the same machine:

```text
8-core machine

Process 1
Process 2
Process 3
Process 4
...
```

Each process has its own:

* event loop
* JavaScript heap
* memory
* DB connection pool
* Redis connections

The OS can distribute those processes across CPU cores.

Conceptually:

```text
                 Server
        +-----------------------+
        |                       |
        | Node Process 1        |
        | Node Process 2        |
        | Node Process 3        |
        | Node Process 4        |
        |                       |
        +-----------------------+
```

This can improve CPU utilization.

Tools such as **PM2 cluster mode** can help manage multiple Node.js processes, but the underlying concept is more important than the tool.

---

# 5.2.17 But there's a trap here

Remember our database connection discussion.

Suppose:

```text
8 Node processes
```

and each has:

```text
pool.max = 20
```

Potential DB connections:

```text
8 × 20 = 160
```

Now suppose we run:

```text
10 containers
× 8 processes
× 20 connections
```

That's:

```text
1,600 potential DB connections
```

So process-level scaling can multiply downstream resource usage.

This is why:

> **Connection pools must be designed at the system level, not independently for every process.**

---

# 5.2.18 Cluster vs Horizontal Containers

There are two possible deployment models.

### Model A — Multiple processes on one machine

```text
Server
├── Node process 1
├── Node process 2
├── Node process 3
└── Node process 4
```

### Model B — Multiple containers/VMs

```text
Load Balancer
    |
    +── Container 1
    +── Container 2
    +── Container 3
    +── Container 4
```

Modern production environments commonly favor **multiple independently scalable instances/containers**, because it gives better:

* isolation
* deployment flexibility
* failure isolation
* autoscaling
* resource management

But multiple processes per instance can still be useful depending on the deployment model.

---

# 5.2.19 What about Worker Threads?

Node.js also provides **Worker Threads**.

They're useful when we have CPU-intensive work that shouldn't block the main event loop.

For example:

```text
HTTP Request
     |
     v
Main Event Loop
     |
     +---- Worker Thread
             |
             +---- CPU-heavy calculation
```

The main thread can continue handling other I/O.

But don't confuse this with general API scaling.

Worker Threads are mainly useful for:

> **CPU-bound work**

They're not something we'd introduce simply because our API has high RPS.

For our parking system, most work is likely:

```text
HTTP
→ Redis
→ PostgreSQL
→ external APIs
```

which is primarily I/O-bound.

So horizontal application scaling is more important than Worker Threads.

---

# 5.2.20 Capacity Planning

Now let's move to the second important area.

Suppose someone asks you in an interview:

> "How many Node.js instances do you need?"

A weak answer is:

> "Maybe 20 instances."

A stronger answer is:

> **"We need to benchmark the actual workload and determine the sustainable capacity of one instance, then calculate the required capacity with headroom."**

Let's work through an example.

Suppose load testing tells us:

```text
One Node.js instance
→ sustainably handles 500 requests/sec
```

Our peak traffic is:

```text
20,000 requests/sec
```

Naively:

```text
20,000 / 500
= 40 instances
```

But running exactly 40 isn't a good production plan.

Why?

Because we need headroom for:

* traffic spikes
* instance failures
* deployments
* uneven traffic distribution
* GC/memory variation
* downstream latency changes

Suppose we target:

```text
70% utilization
```

Then our effective capacity per instance becomes:

```text
500 × 0.70 = 350 req/sec
```

Required instances:

```text
20,000 / 350
≈ 58
```

So we might provision around:

```text
~58–60 instances
```

depending on the actual scaling strategy and failure requirements.

The exact number isn't important here.

**The methodology is.**

---

# 5.2.21 Capacity isn't just RPS

This is another important senior-level point.

Suppose two endpoints both receive:

```text
1,000 requests/sec
```

Endpoint A:

```text
GET /health
```

Endpoint B:

```text
POST /reservation
→ 5 DB operations
→ Redis
→ transaction
→ outbox
```

They clearly don't have the same resource requirements.

So capacity planning should consider:

```text
Request rate
+
Request mix
+
CPU cost
+
Memory
+
I/O
+
DB operations
+
External calls
```

This is why **load testing with realistic traffic distribution** is important.

---

# 5.2.22 Average vs Peak Traffic

Suppose:

```text
Average = 5K req/sec
Peak    = 20K req/sec
```

Designing only for 5K is dangerous.

But permanently provisioning enough infrastructure for 20K may be expensive.

That's where:

```text
Autoscaling
+
Capacity headroom
+
Traffic forecasting
```

become important.

For predictable events, we can even **pre-scale** before the traffic arrives rather than waiting for reactive autoscaling.

---

# 5.2.23 What should we monitor?

For a Node.js service, production monitoring should include things like:

### Application

* RPS
* p50/p95/p99 latency
* error rate
* HTTP status codes

### Node.js

* CPU
* memory
* heap usage
* garbage collection
* event-loop lag/utilization

### Dependencies

* DB latency
* DB connection pool usage
* Redis latency
* Redis connection usage
* external API latency
* Kafka producer/consumer metrics

This is important because:

> **An application metric alone may not tell you the actual bottleneck.**

For example:

```text
Node CPU = 40%
API p95 = 4 seconds
```

Why?

Could be:

```text
DB latency = 3.5 sec
```

The Node.js server isn't CPU-bound.

---

# 5.2.24 Production Scaling Model

For our parking system, the conceptual production model becomes:

```text
                         Clients
                            |
                            v
                      Load Balancer
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
          Node.js       Node.js       Node.js
          Instance      Instance      Instance
              |             |             |
              +-------------+-------------+
                            |
                 +----------+----------+
                 |          |          |
                 v          v          v
               Redis   PostgreSQL    Kafka
```

And the application layer should support:

```text
Horizontal scaling
       +
Statelessness
       +
Health checks
       +
Graceful shutdown
       +
Autoscaling
       +
Capacity headroom
       +
Downstream-aware scaling
```

---

# 5.2.25 The Reusable Interview Framework

When an interviewer asks:

> **"How would you scale the API layer?"**

You can structure your answer:

### Step 1 — Make the service stateless

Shared state goes into appropriate external systems.

### Step 2 — Put instances behind a load balancer

Distribute traffic across healthy instances.

### Step 3 — Horizontally scale

Add/remove instances based on demand.

### Step 4 — Define scaling signals

Don't rely only on CPU.

Consider:

* RPS
* latency
* CPU
* memory
* event-loop utilization
* queue depth
* downstream saturation

### Step 5 — Protect downstream systems

Especially:

```text
DB connection pool
Redis
external APIs
Kafka
```

### Step 6 — Handle instance failures

Health checks + automatic replacement.

### Step 7 — Handle deployments

Graceful shutdown + connection draining + readiness.

### Step 8 — Capacity-plan

Benchmark one instance → determine sustainable capacity → add headroom → account for failure scenarios.

That's a reusable framework for practically every backend system-design interview.

---

We've now completed:

###  Application/API Scaling**

You should now understand not only **how to add more Node.js instances**, but also:

* why statelessness matters
* how load balancing works
* why sticky sessions are generally undesirable when avoidable
* health/readiness checks
* graceful shutdown
* autoscaling
* Node.js process-level scaling
* Worker Threads vs horizontal scaling
* capacity planning
* headroom
* downstream bottlenecks
* connection-pool multiplication
* failure/deployment considerations

---

## Next: Database Scaling

This is going to be a **larger and very important section** because the database is where scaling becomes substantially harder than the application tier.

We'll structure it as:

```text
5.3 Database Scaling
│
├── 5.3.1 Why DB is harder to scale
├── 5.3.2 Query optimization & indexing
├── 5.3.3 Connection pool & DB capacity
├── 5.3.4 Read scaling
│      └── Read replicas
├── 5.3.5 Write scaling
│      └── Why it's harder
├── 5.3.6 Hot rows / hotspots
├── 5.3.7 Partitioning
├── 5.3.8 Sharding
├── 5.3.9 When to use what
└── 5.3.10 Production architecture + interview framework
```

And we'll keep tying each concept back to the **100K reservations/minute** workload while making sure you understand the **general principle that applies to other system-design problems**.

