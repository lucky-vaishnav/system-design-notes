We've completed the major foundational areas we planned:

```text
Distributed Systems Foundation   ✅
Event-Driven Systems              ✅
Microservices                     ✅
Architecture / DDD                ✅
Scaling                           ✅
Consensus & Coordination          ✅
Operations                        ✅
Observability                     ✅
Monitoring & Alerting             ✅
```

Now we move from **learning individual concepts → applying them together**.

# Phase — System Design Case Studies

We'll use the approach we agreed on:

```text
                    Case Study
                        │
              ┌─────────┴─────────┐
              ↓                   ↓
     Technology-Agnostic       AWS Design
          Design                Design
              │                   │
              └─────────┬─────────┘
                        ↓
                Trade-offs &
                 Failure Cases
```

### How we'll do the first case study

I **won't give you the solution immediately**.

I'll give you the system-design problem and requirements. You'll first try to design it based on everything we've learned.

Then I'll act like an interviewer and challenge your design:

* What happens at 10× traffic?
* Why this database?
* Where is the cache?
* Why Kafka vs SQS?
* Do we need a distributed lock?
* What happens if a service fails?
* How do we handle duplicate messages?
* Where do we use Saga/Outbox?
* How do we make it highly available?
* What are the bottlenecks?
* How do we monitor it?
* How would you implement it on AWS?

Then we'll refine your design into a **senior-level production architecture**.

### First Case Study

I'd recommend starting with a **Parking/Trip Booking System** rather than jumping immediately into something generic like URL Shortener.

It will let us naturally use many concepts you've already learned:

```text
Users
 ↓
API Gateway / Load Balancer
 ↓
Booking Service
 ↓
Parking Service
 ↓
Payment Service
 ↓
PostgreSQL
 ↓
Redis
 ↓
Kafka/SQS
 ↓
Workers
 ↓
External APIs
```

And we can introduce realistic problems such as:

**booking + payment consistency, concurrent parking-slot allocation, distributed locking, idempotency, Saga, Outbox, caching, async processing, external API failures, scaling, and observability.**

That would make it a very good **first senior-level case study** rather than just a diagram exercise.
