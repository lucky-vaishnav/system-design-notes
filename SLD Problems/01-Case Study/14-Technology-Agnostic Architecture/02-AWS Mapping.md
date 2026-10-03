We've already designed the system technology-agnostically. Now we'll map each responsibility to AWS services **without changing the architecture just because AWS has a service for it**.

# 9. AWS Mapping

The mental model is:

> **Technology-agnostic responsibility → AWS service → alternatives → trade-offs**

---
f
## 9.1 Overall AWS Architecture

Our previous architecture becomes roughly:

```text
                    Clients
                       │
                       ▼
                CloudFront / WAF
                       │
                       ▼
              API Gateway / ALB
                       │
                       ▼
                Node.js API
                       │
            ┌──────────┴──────────┐
            ▼                     ▼
         ElastiCache             RDS
          (Redis)             PostgreSQL
                                  │
                                  ▼
                               Outbox
                                  │
                                  ▼
                       Amazon MSK / Kafka
                                  │
                  ┌───────────────┼───────────────┐
                  ▼               ▼               ▼
             Notification      Payment         Analytics
              Consumer        Consumer          Consumer

        CloudWatch + X-Ray/OpenTelemetry
        Secrets Manager
        IAM
        S3
        Backup / DR
```

We don't need to memorize this exact diagram yet. We'll map it component by component.

---

# 9.2 Traffic / API Layer

We need:

```text
Client
   ↓
Traffic management
   ↓
Node.js APIs
```

AWS gives us several relevant options.

### Option 1: Application Load Balancer — ALB

```text
Client
   ↓
ALB
   ↓
Node.js instances/containers
```

ALB is useful for HTTP/HTTPS traffic and distributing requests across healthy application instances.

For our backend APIs, this is a natural option.

---

### Option 2: API Gateway

```text
Client
   ↓
API Gateway
   ↓
Backend
```

API Gateway provides additional API-management capabilities such as:

* API routing
* authentication integration
* throttling
* request controls
* API lifecycle/versioning features

---

### Important trade-off

Don't say:

> "API Gateway is always better than ALB."

Instead:

> **ALB is primarily a load-balancing layer, while API Gateway provides broader API-management capabilities. The choice depends on the API requirements and architecture.**

You can also use both in some architectures.

For example:

```text
Client
 ↓
API Gateway
 ↓
ALB
 ↓
Node.js
```

But every additional layer adds:

* latency
* cost
* operational complexity

So we shouldn't add it without a reason.

---

# 9.3 Node.js Application

Our requirement is:

> Stateless Node.js API servers that can scale horizontally.

AWS options include:

### ECS

```text
ALB
 ↓
ECS
 ├── Node.js container
 ├── Node.js container
 └── Node.js container
```

This is a strong fit when we want containerized applications without managing servers directly.

### EKS

Kubernetes-based architecture.

Useful when an organization already has Kubernetes expertise/requirements, but introduces substantially more operational complexity.

### EC2

We can run:

```text
ALB
 ↓
EC2
 ├── Node.js
 ├── Node.js
 └── Node.js
```

More infrastructure control, but more server management.

### Lambda

Possible for some API workloads, but our long-running/high-throughput Node.js backend doesn't automatically need to become serverless.

---

## Our conceptual choice

For this case study, a reasonable AWS mapping is:

> **Node.js APIs → ECS containers behind an ALB**

But remember:

**This is a mapping choice, not a universal rule.**

---

# 9.4 PostgreSQL

Our source of truth is PostgreSQL.

AWS has:

## Amazon RDS for PostgreSQL

Conceptually:

```text
Node.js
   ↓
RDS PostgreSQL
```

RDS handles much of the infrastructure management around the database.

For HA, we can have:

```text
Primary
   │
   └── Standby
```

with managed failover capabilities.

---

## Why not DynamoDB?

This is an important interview discussion.

We need:

* transactions
* relational constraints
* inventory correctness
* row-level locking
* complex transactional workflows

PostgreSQL is a natural fit for this particular model.

DynamoDB can absolutely support high-scale transactional systems, but it requires designing around its access patterns and consistency model.

So:

> **We choose PostgreSQL because our core booking problem is strongly transactional and relational, not simply because PostgreSQL is familiar.**

---

# 9.5 Redis

Our conceptual requirement:

```text
Fast distributed cache
```

AWS mapping:

## Amazon ElastiCache for Redis

```text
Node.js
   ↓
ElastiCache Redis
```

Used for:

* availability cache
* frequently accessed data
* distributed rate limiting
* potentially distributed coordination where appropriate

Remember:

```text
Redis
  ≠
source of truth
```

PostgreSQL remains authoritative.

---

# 9.6 Kafka

Our architecture requires:

```text
PostgreSQL
     ↓
Outbox
     ↓
Kafka
```

AWS gives us:

## Amazon MSK

Managed Apache Kafka.

Conceptually:

```text
Outbox Publisher
       ↓
      MSK
       ↓
Consumer Groups
```

We still retain Kafka concepts:

* topics
* partitions
* offsets
* consumer groups
* replication

AWS manages much of the underlying Kafka infrastructure.

---

## Alternative: Amazon SQS

This is an important distinction.

SQS is a queue.

Kafka is an event streaming platform.

For a simple background task:

```text
Reservation
    ↓
Queue
    ↓
Email Worker
```

SQS may be perfectly appropriate.

But if we need:

```text
Multiple consumers
Event replay
Partition-based ordering
Longer-lived event streams
Independent consumer groups
```

Kafka/MSK becomes more attractive.

So don't say:

> "Kafka is always better than SQS."

Instead:

> **Choose based on messaging requirements, ordering, replay, consumer model, throughput, and operational needs.**

**
The key differences between SQS and Kafka stem from their contrasting messaging models, scalability, and performance. While SQS utilizes a pull-based model with messages typically consumed by a single consumer, Kafka follows a publish-subscribe model, enabling multiple consumers to read from the same stream of messages.  

Moreover, Kafka outshines SQS in terms of scalability and performance. It exhibits high throughput capabilities, making it suitable for handling large volumes of data and messages and allows for seamless scalability by adding more nodes to a Kafka cluster.
**

---

# 9.7 Object Storage

For things such as:

```text
Reports
Exports
Documents
Large files
Backup artifacts
```

AWS mapping:

> **Amazon S3**

For example:

```text
Application
    ↓
S3
    ↓
Report/file
```

We don't want to store large files directly inside PostgreSQL unnecessarily.

---

# 9.8 Secrets

Our previous requirement:

> Never hard-code credentials.

AWS mapping:

## AWS Secrets Manager

For:

```text
DB credentials
Payment provider secrets
API credentials
Other sensitive configuration
```

Application retrieves secrets through appropriate IAM permissions.

Another AWS service:

## Systems Manager Parameter Store

Can be useful for configuration and some secret-management scenarios.

Again, choice depends on requirements.

---

# 9.9 IAM

IAM handles AWS-level authorization.

For example:

```text
Node.js ECS task
      ↓
IAM Role
      ↓
Allowed:
   S3 read
   Secrets Manager read
   CloudWatch write

Not allowed:
   unrelated resources
```

This implements the least-privilege principle we discussed earlier.

We don't want every application component having:

```text
AdministratorAccess
```

---

# 9.10 Observability

Our conceptual requirement:

```text
Logs
Metrics
Traces
Alerts
```

AWS mapping can include:

### CloudWatch

For:

* logs
* metrics
* dashboards
* alarms

Conceptually:

```text
Node.js
   ↓
CloudWatch Logs

Infrastructure
   ↓
CloudWatch Metrics
```

For distributed tracing, AWS environments can use tracing through services such as **AWS X-Ray**, often alongside OpenTelemetry-based instrumentation.

The important architectural idea remains:

```text
Metrics → detect
Trace → locate
Logs → investigate
```

The AWS service is secondary to that principle.

---

# 9.11 WAF

We previously discussed:

* malicious traffic
* bots
* abusive requests
* common web attacks

AWS mapping:

> **AWS WAF**

Conceptually:

```text
Internet
   ↓
WAF
   ↓
ALB / API Gateway
   ↓
Application
```

WAF can help filter/block unwanted HTTP traffic before it reaches our application.

This complements, rather than replaces:

* authentication
* authorization
* application-level validation
* rate limiting

---

# 9.12 CDN

For static content:

```text
Images
CSS
JavaScript
Static files
```

we can use:

> **Amazon CloudFront**

Conceptually:

```text
User
 ↓
CloudFront
 ↓
S3 / origin
```

For our backend APIs, CDN caching is a separate decision from our Redis availability cache.

Don't confuse:

```text
CloudFront
```

with:

```text
Redis
```

They solve different caching problems.

---

# 9.13 Backup / Disaster Recovery

Our DR requirements were:

```text
Backup
Replication
RPO
RTO
Restore testing
```

AWS provides managed backup capabilities around many services.

For PostgreSQL/RDS, we can use:

* automated backups
* snapshots
* point-in-time recovery
* cross-region backup/replication strategies depending on requirements

And for other data/services, we need to design corresponding recovery mechanisms.

The key point:

> **AWS providing a backup feature doesn't automatically mean our application's DR requirements are satisfied.**

We still need to validate:

```text
Can we restore?
How much data can we lose?
How quickly can we recover?
Can the application reconnect?
Can traffic be redirected?
Are secrets/configuration available?
Do dependent services recover?
```

---

# 9.14 Putting the AWS Pieces Together

A reasonable architecture could therefore look like:

```text
                         Users
                           │
                           ▼
                     CloudFront
                           │
                           ▼
                         WAF
                           │
                           ▼
                  API Gateway / ALB
                           │
                           ▼
                    ECS Node.js
                  ┌────────┼────────┐
                  │        │        │
                  ▼        ▼        ▼
              Redis     PostgreSQL  Secrets
            ElastiCache    RDS      Manager
                           │
                           ▼
                         Outbox
                           │
                           ▼
                          MSK
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          Payment      Notification  Analytics
          Consumer       Consumer      Consumer

                 ┌─────────────────┐
                 │   CloudWatch    │
                 │ Logs/Metrics    │
                 │ Alarms          │
                 └─────────────────┘

                 IAM + TLS + WAF
                 Security controls
```

Again, **this is one reasonable mapping**, not the only correct AWS architecture.

---

# The important interview skill

If an interviewer asks:

> "Why did you choose ECS?"

Don't answer:

> "Because ECS is an AWS container service."

Instead:

> "Our application is a stateless Node.js backend, so we need horizontally scalable compute with container support. ECS provides managed container orchestration without requiring us to operate a Kubernetes control plane. If the organization already standardized on Kubernetes, EKS could be considered, while EC2 provides more infrastructure control at the cost of more operational management."

**That is the level of reasoning we're aiming for.**

---

### 9.1–9.11 Status

We've now mapped the major architecture components:

```text
Traffic/API        → ALB / API Gateway
Compute             → ECS / EKS / EC2
Database            → RDS PostgreSQL
Cache               → ElastiCache Redis
Events              → MSK / Kafka
Object storage      → S3
Secrets             → Secrets Manager
Authorization       → IAM
Observability       → CloudWatch + tracing
Edge security       → WAF
CDN                 → CloudFront
DR                  → Backups / replication / recovery strategy
```

**Next we should do the part that is more valuable for senior interviews: AWS failure scenarios and trade-offs.**

For example:

> What happens if RDS fails?
> What happens if Redis fails?
> Why ECS instead of EKS?
> Why MSK instead of SQS?
> Why RDS instead of DynamoDB?
> What happens if an entire AWS region goes down?

That will turn the AWS mapping from a **service-name list into actual architecture reasoning**.

