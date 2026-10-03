This is where the AWS mapping becomes useful for a senior interview. We won't just memorize services—we'll explain **why we chose them, what can fail, and what we do about it.**

# 10. AWS Failure Scenarios + Trade-offs

Our current mapping:

```text
Clients
   ↓
CloudFront / WAF
   ↓
ALB / API Gateway
   ↓
ECS Node.js
   ↓
┌───────────────┬──────────────┐
Redis           RDS           Secrets
   │              │
   │              ↓
   │           Outbox
   │              ↓
   │             MSK
   │              ↓
   └────── Consumers ──────────
```

---

# 10.1 What if an ECS/Node.js instance fails?

Suppose:

```text
ECS
 ├── Task 1 ✅
 ├── Task 2 ❌
 └── Task 3 ✅
```

The load balancer/health-check mechanism detects that Task 2 isn't healthy.

Traffic goes to:

```text
Task 1
Task 3
```

The service scheduler can replace the failed task.

```text
Task 2 ❌
   ↓
new Task 4
```

### Why our architecture handles this well

Because the Node.js API is **stateless**.

We don't lose critical business state when one application instance disappears.

Business state is in:

```text
PostgreSQL
Redis
Kafka
```

rather than only in:

```text
Node.js process memory
```

### Interview takeaway

> "Because the application layer is stateless, individual ECS tasks are disposable. Health checks and service scheduling replace unhealthy tasks, while durable state remains in shared infrastructure."

---

# 10.2 What if an entire Availability Zone fails?

Suppose:

```text
AZ-A ❌
```

If all our ECS tasks were in AZ-A:

```text
ECS
 ├── AZ-A
 │    ├── Task 1
 │    └── Task 2
 │
 └── AZ-B
      ├── Task 3
      └── Task 4
```

then Tasks 1 and 2 are lost, but AZ-B continues serving traffic.

So production workloads should be distributed across multiple AZs.

The same principle applies to important managed infrastructure where the service's HA configuration supports multi-AZ operation.

### Important distinction

```text
Multiple instances
        ≠
Multi-AZ
```

You want both.

---

# 10.3 What if RDS PostgreSQL fails?

This is much more serious because PostgreSQL is our source of truth.

With an HA configuration:

```text
          RDS
           │
      ┌────┴────┐
      ↓         ↓
   Primary    Standby
      ✅         🟢
```

If the primary fails:

```text
Primary ❌
   ↓
Failover
   ↓
Standby becomes primary
```

The application reconnects to the database endpoint.

---

## But what happens to an in-flight reservation?

Suppose:

```text
BEGIN
 ↓
lock inventory
 ↓
update inventory
 ↓
DB fails
```

The application cannot blindly assume:

```text
reservation succeeded
```

The transaction may have committed or may have been rolled back.

This is an **unknown outcome** problem.

For critical operations, the application should determine the final state through appropriate retry/reconciliation logic rather than creating a second booking blindly.

This connects directly to our earlier:

* transactions
* idempotency
* reconciliation

topics.

---

# 10.4 What if Redis fails?

This is a very important design decision.

Remember:

> **Redis is not our source of truth.**

So:

```text
Redis ❌
```

doesn't necessarily mean:

```text
Booking system ❌
```

We can potentially fall back to PostgreSQL.

But there's a danger.

Suppose we have:

```text
1M searches/min
```

and Redis suddenly disappears.

Then:

```text
1M searches/min
       ↓
PostgreSQL
```

could overload the database.

So our failure strategy should be:

```text
Redis failure
     ↓
Controlled fallback
     ↓
Protect PostgreSQL
     ↓
Recover Redis
```

Possible protections include:

* concurrency limits
* rate limiting
* request coalescing
* graceful degradation
* temporary rejection of low-priority requests

This is exactly why **HA and backpressure are connected**.

---

# 10.5 What if MSK/Kafka fails?

Suppose:

```text
Reservation DB transaction
        ↓
Outbox
        ↓
MSK ❌
```

The reservation itself should not necessarily fail.

Because:

```text
Reservation state
+
Outbox event
```

were committed in PostgreSQL.

The publisher can retry later:

```text
Outbox
  ↓
retry
  ↓
MSK
```

So:

> **Kafka availability does not need to determine whether the core reservation transaction succeeds.**

That's a very useful architectural principle:

> **Don't make asynchronous infrastructure a synchronous dependency for a critical transaction unless the business requirement genuinely requires it.**

---

# 10.6 What if a Kafka consumer fails?

We already discussed this.

Suppose:

```text
Partition 2
    ↓
Consumer B ❌
```

Kafka's consumer-group mechanism can rebalance the partition to another healthy consumer.

```text
Partition 2
     ↓
Consumer B ❌
     ↓
Rebalance
     ↓
Consumer A
```

The new consumer continues from the appropriate committed offset.

If all consumers are temporarily down:

```text
Kafka
  ↓
messages remain
  ↓
consumer returns
  ↓
processing resumes
```

The problem becomes:

```text
Consumer lag ↑
```

which should trigger monitoring/alerts if it persists.

---

# 10.7 Why MSK instead of SQS?

This is a classic AWS interview question.

### SQS is attractive when:

```text
Producer
   ↓
Queue
   ↓
Worker
```

You mainly need:

* task processing
* buffering
* retry
* decoupling

For example:

```text
Reservation
   ↓
SQS
   ↓
Send confirmation email
```

SQS can be a very good choice.

---

### Kafka/MSK becomes attractive when:

We need things such as:

```text
ReservationCreated
       ↓
 ┌─────┼─────┐
 ↓     ↓     ↓
Email Analytics Cache
```

with independent consumer groups.

And potentially:

* event replay
* partition-based ordering
* high-throughput event streams
* multiple independent consumers
* longer-lived event history

So the answer isn't:

> "Kafka is better."

It's:

> "The messaging requirement determines the choice."

---

# 10.8 Why RDS PostgreSQL instead of DynamoDB?

Another classic question.

Our booking problem has:

```text
Inventory
Reservation
Payment
Refund
```

with transactional relationships.

We need strong transactional behavior around inventory and reservation.

PostgreSQL gives us:

* relational model
* transactions
* constraints
* indexes
* row-level locking
* mature SQL querying

DynamoDB can also support transactions and strong consistency in specific patterns, but we'd need to design the data model around DynamoDB's access patterns.

So the architectural reasoning is:

> **The workload and consistency model make relational transactional storage a natural fit.**

Not:

> "SQL is always better."

---

# 10.9 Why Redis if PostgreSQL already exists?

Because they solve different problems.

PostgreSQL:

```text
Correctness
Durability
Transactions
```

Redis:

```text
Speed
High-volume reads
Reduced DB load
```

Without Redis:

```text
1M searches/min
       ↓
PostgreSQL
```

With Redis:

```text
1M searches/min
       ↓
Redis
       ↓
Most requests served cheaply
```

But booking still goes to PostgreSQL.

---

# 10.10 Why ECS instead of EKS?

This is another good senior interview trade-off.

### ECS

Pros:

* simpler operational model
* AWS-native
* less Kubernetes management
* good fit for containerized workloads

Cons:

* less Kubernetes portability/ecosystem
* AWS-specific abstractions

### EKS

Pros:

* Kubernetes ecosystem
* portability
* powerful orchestration capabilities
* useful if organization already standardizes on Kubernetes

Cons:

* more operational complexity
* more Kubernetes expertise required
* additional platform management

So our answer:

> "For a team that primarily needs managed container orchestration without Kubernetes-specific requirements, ECS reduces operational complexity. If the organization already operates Kubernetes or needs its ecosystem and portability, EKS may be justified."

---

# 10.11 Why not just use Lambda?

Lambda can work very well for:

* event-driven processing
* lightweight APIs
* bursty workloads
* short-lived tasks

But our system has:

```text
High-volume APIs
Long-running Node.js services
Database connection management
Complex booking workflows
Potentially sustained traffic
```

So containers may provide a more predictable operational model.

Again:

> **Not "Lambda is bad." The workload determines the choice.**

---

# 10.12 What if the entire AWS Region fails?

Now we reach our DR discussion.

Suppose:

```text
Region A ❌❌❌
```

Our options depend on RTO/RPO.

### Backup-based recovery

```text
Region A
   ↓
Backup
   ↓
Region B
   ↓
Restore
```

Lower cost, slower recovery.

### Warm standby

```text
Region A
 FULL
   ↓
Primary

Region B
 SMALL
   ↓
Standby
```

Faster recovery, higher cost.

### Active-active

```text
        Traffic
        /     \
       ↓       ↓
 Region A   Region B
 ACTIVE     ACTIVE
```

Fast failover, but much more difficult for our strongly consistent booking data.

---

# 10.13 The Multi-Region Booking Problem

This is worth emphasizing.

Suppose:

```text
Last parking slot = 1
```

Region A:

```text
User A → reserve
```

Region B:

```text
User B → reserve
```

If both regions independently accept:

```text
A → SUCCESS
B → SUCCESS
```

we've double-booked.

Therefore, multi-region active-active isn't just:

> "Run the application in two regions."

We need a strategy for **authoritative inventory ownership and consistency**.

Possible strategies include:

* single write region
* partitioning ownership by parking lot
* globally coordinated consistency
* carefully designed distributed transaction/consensus mechanisms

But each introduces complexity and/or latency.

This is why:

> **Multi-region architecture is a business requirement decision, not simply a scaling checkbox.**

---

# 10.14 AWS Cost vs Complexity

This is a major senior-level principle.

You could build:

```text
Multi-AZ
+
Multi-region
+
Active-active
+
Kafka
+
Redis Cluster
+
Multiple databases
+
Global routing
```

and end up with a very complicated system.

But if the actual business requirement is:

```text
RTO = 2 hours
RPO = 30 minutes
```

you may not need an extremely expensive active-active architecture.

Architecture should satisfy:

```text
Requirements
      ↓
Reliability target
      ↓
RPO / RTO
      ↓
Architecture
```

not:

```text
AWS service exists
      ↓
Let's use it
```

---

# 10.15 Final AWS Interview Framework

If an interviewer gives you a system and says:

> **"Design it on AWS."**

Think in this order:

### 1. Compute

```text
EC2 / ECS / EKS / Lambda
```

### 2. Traffic

```text
CloudFront
WAF
ALB
API Gateway
```

### 3. Database

```text
RDS / Aurora / DynamoDB
```

### 4. Cache

```text
ElastiCache
```

### 5. Async processing

```text
SQS / SNS / MSK / EventBridge
```

### 6. Storage

```text
S3
```

### 7. Security

```text
IAM
Secrets Manager
KMS
WAF
TLS
```

### 8. Observability

```text
CloudWatch
X-Ray / OpenTelemetry
```

### 9. HA

```text
Multi-AZ
Failover
Auto Scaling
```

### 10. DR

```text
Backup
Replication
Cross-region
RPO
RTO
Restore testing
```

But **don't start by listing AWS services**.

Start with:

> **Requirement → workload → bottleneck → consistency → failure model → then AWS service.**

That's the senior-level difference.

---

# Major Topic 10 — COMPLETE ✅

We now have:

```text
1. Transactions + Concurrency              ✅
2. Consistency + Cache                     ✅
3. Payment Failures + Reconciliation       ✅
4. Saga + Outbox + Kafka                   ✅
5. Scaling + Bottlenecks + HA              ✅
6. Security + Rate Limiting + Reliability  ✅
7. Observability + Disaster Recovery       ✅
8. Final Technology-Agnostic Architecture  ✅
9. AWS Mapping                             ✅
10. AWS Failure + Trade-offs               ✅
```

## Next: Major Topic 11 — Full End-to-End Walkthrough

This is an important step before the mock interview.

We'll take **one complete reservation request** and walk through it from:

```text
User searches
   ↓
Availability
   ↓
Reservation
   ↓
Concurrency
   ↓
Payment
   ↓
Outbox
   ↓
Kafka
   ↓
Confirmation
   ↓
Cache update
   ↓
Notification
   ↓
Observability
```

And we'll deliberately inject failures at different points to make sure you can explain the architecture **without relying on memorized topic-by-topic answers**.

