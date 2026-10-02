We’ll keep the same approach: apply it to our parking system, focus on senior-level reasoning, and avoid re-teaching fundamentals unnecessarily.

# 7. Observability + Disaster Recovery

### Roadmap

```text
7. Observability + Disaster Recovery
├── 7.1 Logging
├── 7.2 Metrics
├── 7.3 Distributed Tracing
├── 7.4 Correlation / Request IDs
├── 7.5 Alerting
├── 7.6 SLI / SLO / SLA
├── 7.7 Disaster Recovery
├── 7.8 RPO / RTO
├── 7.9 Backup & Restore
├── 7.10 Regional Failure
└── 7.11 End-to-End Failure Scenarios
```

---

# 7.1 Logging

We've already discussed failures, retries, reconciliation, Kafka, payment providers, etc.

Now the question is:

> **When something fails in production, how do we actually understand what happened?**

That's the role of observability.

Logging is one part of observability, but **good logging is not simply “put logs everywhere.”**

The goal is:

> **Produce enough structured information to reconstruct what happened to a request/business operation without overwhelming the system or exposing sensitive data.**

---

## 1. What should we log?

For our parking system, consider:

```text
User
  ↓
API
  ↓
Reservation
  ↓
Payment
  ↓
Database
  ↓
Kafka
  ↓
Notification
```

A single booking can touch many components.

We need logs that allow us to follow that operation.

For example:

```text
request_id = req-123
user_id = 456
reservation_id = res-789
payment_id = pay-111
```

Then logs might look conceptually like:

```text
INFO Reservation request received
reservation_id=res-789
user_id=456

INFO Inventory successfully reserved
reservation_id=res-789
parking_lot_id=PL-10

INFO Payment initiated
reservation_id=res-789
payment_id=pay-111

WARN Payment provider timeout
reservation_id=res-789
payment_id=pay-111

INFO Payment marked UNKNOWN
reservation_id=res-789
payment_id=pay-111

INFO Reconciliation started
payment_id=pay-111

INFO Payment provider status=SUCCESS
payment_id=pay-111

INFO Reservation confirmed
reservation_id=res-789
```

Now an engineer can reconstruct the entire business flow.

---

# 2. Structured logging

For a production system, prefer structured logs rather than only free-form strings.

For example:

```json
{
  "timestamp": "2026-10-02T12:30:45Z",
  "level": "ERROR",
  "service": "reservation-service",
  "event": "payment_provider_timeout",
  "request_id": "req-123",
  "reservation_id": "res-789",
  "payment_id": "pay-111",
  "provider": "payment-provider",
  "duration_ms": 3000
}
```

This is much easier to search and aggregate.

Instead of searching:

```text
"something went wrong"
```

we can query:

```text
service = reservation-service
event = payment_provider_timeout
```

or:

```text
reservation_id = res-789
```

---

# 3. Log levels

Typical levels:

```text
DEBUG
INFO
WARN
ERROR
```

### DEBUG

Detailed troubleshooting information.

Useful during development or temporarily during production investigation.

Don't blindly enable huge DEBUG volumes in production.

### INFO

Normal important business/system events.

Example:

```text
Reservation created
Payment initiated
Reservation confirmed
```

### WARN

Something unusual happened but the system handled it.

Example:

```text
Payment provider timeout
Retry scheduled
Cache unavailable → DB fallback
```

### ERROR

An operation failed and requires attention.

Example:

```text
Database transaction failed
Payment reconciliation failed
Kafka consumer processing failed
```

The exact severity policy should be consistent across services.

---

# 4. What should NOT be logged?

This is extremely important.

Never casually log:

```text
password
credit card number
CVV
access token
refresh token
API secret
payment credentials
```

Even user information may need masking depending on sensitivity.

For example, don't do:

```text
payment_token=abc123...
```

if that token could be used for authentication or payment operations.

Instead:

```text
payment_token_present=true
```

or use a safe identifier.

This connects directly to our **Security** topic.

---

# 5. Logging business events vs technical events

This is a useful senior-level distinction.

### Technical event

```text
Database connection timeout
Redis connection failed
HTTP 504 from provider
```

### Business event

```text
Reservation created
Reservation expired
Payment succeeded
Refund initiated
Refund completed
```

Both are important.

Suppose a customer says:

> "I paid, but my reservation isn't confirmed."

We don't only want:

```text
HTTP 500
```

We want to understand:

```text
Reservation created
→ Payment initiated
→ Payment succeeded
→ Confirmation failed
→ Reconciliation
→ Reservation confirmed/refund initiated
```

Business-level events make troubleshooting much easier.

---

# 6. Logs should have correlation identifiers

This becomes particularly important in distributed systems.

Imagine:

```text
API Service
   ↓
Reservation Service
   ↓
Payment Service
   ↓
Kafka
   ↓
Notification Service
```

A request could generate dozens of logs across multiple services.

We need to connect them.

That's why we use things like:

```text
request_id
trace_id
reservation_id
payment_id
event_id
```

We'll cover this properly in **7.4 Correlation / Request IDs**.

---

# 7.7? Why I'm stopping here

I want to separate **logging** from the other observability mechanisms instead of mixing everything together.

For now:

### 7.1 Logging — COMPLETE

**Remember for ANY system-design problem:**

> Logging should be structured, searchable, correlated across services, useful for reconstructing business flows, and safe from sensitive-data leakage.

---

# 7.2 Metrics

Logs tell us:

> **What happened?**

Metrics tell us:

> **How is the system behaving over time?**

This distinction is very important.

Suppose our reservation API starts becoming slow.

Logs might show individual failures.

Metrics can show:

```text
Requests/sec
p50 latency
p95 latency
p99 latency
Error rate
CPU
Memory
DB connections
DB latency
Redis latency
Kafka lag
```

Now we can identify trends.

---

## 1. The four important metric categories

### Traffic

How much work is coming in?

```text
requests/sec
reservations/sec
payments/sec
Kafka messages/sec
```

### Latency

How long does it take?

```text
p50
p95
p99
```

### Errors

How often does it fail?

```text
HTTP 5xx
payment failures
DB errors
timeouts
```

### Saturation

How close are resources to their limits?

```text
CPU
memory
DB connection pool
Redis connections
Kafka consumer lag
queue depth
```

A useful mental model:

> **Traffic + Latency + Errors + Saturation**

---

# 2. Why percentiles instead of only average?

Suppose 1,000 requests happen.

999 requests take:

```text
100 ms
```

One request takes:

```text
10 seconds
```

The average might still look reasonably good.

But that one slow request represents a real user experience.

That's why production systems commonly monitor:

```text
p50 → typical experience
p95 → slower users
p99 → worst tail behavior
```

For our reservation API, we might care about:

```text
p50 = 150 ms
p95 = 500 ms
p99 = 1.2 sec
```

Then suddenly:

```text
p99 = 8 sec
```

That is a strong signal that something has changed even if average latency looks acceptable.

---

# 3. Business metrics are also important

This is something people sometimes miss in system design.

Technical metrics:

```text
CPU
latency
error rate
DB connections
```

But business metrics:

```text
reservation success rate
payment success rate
reservation expiration rate
refund rate
payment UNKNOWN count
booking conversion rate
```

can reveal problems that infrastructure metrics don't.

For example:

```text
CPU = normal
DB = normal
API latency = normal
```

but:

```text
payment success rate dropped from 98% → 70%
```

That's still a serious production incident.

---

# 4. Example: detecting our bottleneck

Suppose reservation traffic increases.

Metrics show:

```text
API CPU          → 55%
API latency      → 400 ms
DB CPU           → 90%
DB connections   → 95%
DB lock wait     → high
Redis            → normal
Kafka            → normal
```

We shouldn't automatically add Node.js instances.

The metrics indicate:

> **The database is becoming the bottleneck.**

This connects directly to our Major Topic 5:

**Scale the actual bottleneck, not the component receiving the traffic.**

---

### 7.2 Metrics — COMPLETE

**Remember for ANY system-design problem:**

> Monitor traffic, latency, errors, and saturation—and include business-level metrics, not just infrastructure metrics.

---

# 7.3 Distributed Tracing

Now we have:

* Logs → what happened
* Metrics → how the system behaves

Tracing answers:

> **Where did the time go across the distributed system?**

Example:

```text
POST /reservations
        |
        | 50 ms
        ↓
Reservation Service
        |
        | 20 ms
        ↓
Redis
        |
        | 300 ms
        ↓
PostgreSQL
        |
        | 2 sec
        ↓
Payment Provider
```

We immediately see that the payment provider consumed most of the request time.

---

## Trace

A trace represents one end-to-end request/workflow.

Example:

```text
Trace ID: trace-123
```

It contains multiple **spans**:

```text
Span 1: API request
Span 2: DB query
Span 3: payment API call
Span 4: Kafka publish
```

Each span has:

```text
start time
end time
duration
service
operation
status
```

---

# Example

User calls:

```http
POST /reservations
```

Trace:

```text
trace-123

API
 ├── authentication       5 ms
 ├── Redis availability   8 ms
 ├── PostgreSQL           120 ms
 ├── Payment Provider     2800 ms
 └── response             10 ms
```

Now we know exactly where the latency is coming from.

---

# Logs + Metrics + Traces together

This is the important mental model:

```text
Metrics
   ↓
Something is wrong

Traces
   ↓
Where is it happening?

Logs
   ↓
What exactly happened?
```

Example:

```text
Metric:
p99 reservation latency increased

        ↓

Trace:
payment provider taking 3 seconds

        ↓

Logs:
provider timeout/retry events
```

That's the real value of observability.

---

### 7.3 Distributed Tracing — COMPLETE

**Remember for ANY system-design problem:**

> Metrics detect the problem, tracing localizes the problem, and logs explain the detailed event history.

---

# 7.4 Correlation / Request IDs

This is closely related, so we'll complete it now.

Imagine:

```text
Client
 ↓
API Gateway
 ↓
Reservation Service
 ↓
Payment Service
 ↓
Kafka
 ↓
Notification Service
```

We need a way to connect activity across services.

A request can have:

```text
request_id = req-123
```

A distributed trace can have:

```text
trace_id = trace-456
```

A business operation has:

```text
reservation_id = res-789
payment_id = pay-111
```

An event has:

```text
event_id = evt-222
```

These identifiers serve different purposes.

### Request ID

Identifies an individual request.

### Trace ID

Connects distributed spans across services.

### Business ID

Identifies the actual business entity/workflow.

### Event ID

Identifies an individual event and helps with deduplication/debugging.

---

## Why business IDs matter

Suppose the customer says:

> "My reservation is res-789."

We can search:

```text
reservation_id=res-789
```

and find logs across services.

That's much more useful than searching by timestamp.

---

### 7.4 Correlation / Request IDs — COMPLETE

**Remember:**

> Every distributed operation should be traceable using correlation identifiers, while business IDs allow engineers to follow a specific business workflow.

---

# 7.5 Alerting

Monitoring is not enough.

Someone needs to know when something is wrong.

But:

> **Don't alert on every error.**

Otherwise engineers get alert fatigue.

---

## Good alert examples

```text
Reservation API p99 latency > threshold
```

```text
5xx error rate > threshold
```

```text
Payment UNKNOWN count increasing
```

```text
Kafka consumer lag continuously increasing
```

```text
Database connection pool near exhaustion
```

```text
Redis unavailable for sustained period
```

These represent conditions requiring attention.

---

## Bad alert

```text
One request failed
```

That's usually noise.

Instead:

```text
5%+ requests failing for 5 minutes
```

might be meaningful.

The exact threshold depends on the system's SLO and traffic pattern.

---

### Alert principle

> Alert on symptoms that require action, not every individual event.

---

### 7.5 Alerting — COMPLETE

---

# 7.6 SLI / SLO / SLA

These are important senior interview concepts.

### SLI

**Service Level Indicator**

What we actually measure.

Example:

```text
99.95% of reservation requests succeeded
```

or:

```text
p95 latency = 400 ms
```

---

### SLO

**Service Level Objective**

The target.

Example:

```text
99.9% successful reservation requests
```

or:

```text
95% of reservation requests complete within 500 ms
```

---

### SLA

**Service Level Agreement**

A contractual commitment, usually to customers.

Example:

```text
99.9% availability guaranteed
```

An SLA may have business consequences if violated.

---

### Simple hierarchy

```text
SLI
 ↓
What are we measuring?

SLO
 ↓
What target do we want?

SLA
 ↓
What contractual commitment do we make?
```

---

## Why SLO matters for architecture

Suppose we say:

```text
Reservation API SLO:
99.9% successful requests
```

Now architectural decisions have context.

We can ask:

* How much downtime/error budget do we have?
* How aggressive should failover be?
* What latency should we target?
* Which failures can we tolerate?
* How much redundancy is justified?

---

### 7.6 SLI / SLO / SLA — COMPLETE

---

# Major Topic 7 status

```text
7.1 Logging                  COMPLETE
7.2 Metrics                 COMPLETE
7.3 Distributed Tracing      COMPLETE
7.4 Correlation IDs          COMPLETE
7.5 Alerting                COMPLETE
7.6 SLI / SLO / SLA          COMPLETE
7.7 Disaster Recovery        NEXT
7.8 RPO / RTO                NEXT
7.9 Backup & Restore         NEXT
7.10 Regional Failure        NEXT
7.11 End-to-End Failures    NEXT
```

The important observability mental model to keep in your notes is:

> **Metrics tell us that something is wrong. Traces tell us where. Logs tell us what happened. Business identifiers let us follow the actual workflow. Alerts tell us when humans need to act. SLOs define what “healthy enough” means.**

Next we'll move into the second half of Topic 7: **Disaster Recovery**, which is different from the high availability we already covered.

