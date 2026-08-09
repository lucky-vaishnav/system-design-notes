## Monitoring & Alerting —

This is the **final topic in our Operations phase**. We'll keep it separate from Observability: observability helps us understand *why* something is happening; monitoring continuously checks whether the system is behaving within expected boundaries.

### Operations

```
✅ Distributed Scheduling
✅ Observability
➡️ Monitoring & Alerting
```

We’ll cover **Monitoring & Alerting deeply**, but keep it distinct from Observability.

We’ll focus on:

1. **Monitoring vs Observability**
2. **What should be monitored in production**
3. **Golden Signals — Latency, Traffic, Errors, Saturation**
4. **RED and USE methods**
5. **Health checks & readiness/liveness**
6. **Metrics and dashboards**
7. **Alert design & thresholds**
8. **Alert fatigue and actionable alerts**
9. **SLI / SLO / SLA**
10. **Error budgets & burn rate**
11. **Incident detection and escalation**
12. **Monitoring Node.js + PostgreSQL + Redis + Kafka/SQS**
13. **AWS monitoring — CloudWatch and related services**
14. **A practical production monitoring architecture**
15. **Senior-level interview scenarios**

Let's start with the foundation.

---

## 1. What Is Monitoring?

**Monitoring = continuously measuring the health and performance of a system.**

For example:

```
Node.js API
   ↓
Monitor
   ↓
Requests: 5,000/min
Error rate: 0.5%
p95 latency: 350ms
CPU: 65%
Memory: 72%
```

The goal is to detect:

```
Normal
  ↓
Abnormal
  ↓
Problem detected
  ↓
Alert
```

---

# 2. Monitoring vs Observability

This distinction is important.

### Monitoring

Answers:

> **"Is something wrong?"**
> 

Example:

```
Error rate > 5%
```

### Observability

Answers:

> **"Why is it wrong?"**
> 

Example:

```
API
 ↓
Payment Service
 ↓
External Payment API
 ↓
Timeout
```

So:

```
Monitoring
→ Detection

Observability
→ Investigation
```

They work together.

---

# 3. What Should We Monitor?

For a production backend, think in layers.

```
Application
Infrastructure
Database
Queues
External Dependencies
Business Metrics
```

### Application

```
Request rate
Error rate
Latency
Throughput
Event-loop lag
Active requests
```

### Infrastructure

```
CPU
Memory
Disk
Network
Container health
EC2 health
```

### Database

```
Connections
Connection pool usage
Query latency
Slow queries
Locks
CPU
Storage
Replication lag
```

### Queue

For SQS/Kafka:

```
Queue depth
Consumer lag
Processing rate
Failed messages
Retries
DLQ messages
```

### External APIs

```
Latency
Error rate
Timeouts
Availability
Rate-limit responses
```

For example, your Uber/payment integrations.

---

# 4. The Four Golden Signals

This is one of the most important monitoring concepts.

```
1. Latency
2. Traffic
3. Errors
4. Saturation
```

### Latency

How long requests take.

```
p50 = 100ms
p95 = 400ms
p99 = 2s
```

### Traffic

How much traffic the system receives.

```
10,000 requests/min
```

### Errors

How many requests fail.

```
500 errors
timeouts
failed jobs
```

### Saturation

How close the system is to its capacity.

```
CPU       = 90%
Memory    = 85%
DB Pool   = 98%
Queue     = growing rapidly
```

---

# 5. Why Saturation Is Important

Imagine:

```
Current traffic = 5,000 req/min
```

Everything looks fine.

But:

```
DB connections = 98/100
```

You're close to the limit.

A traffic increase could cause:

```
DB pool exhausted
      ↓
Requests wait
      ↓
Timeouts
      ↓
Error rate increases
```

Monitoring saturation can therefore detect a problem **before users experience a failure**.

---

# 6. RED Method

For request-driven services, another very useful framework is **RED**.

```
R = Rate
E = Errors
D = Duration
```

Example:

```
Rate:
5,000 requests/sec

Errors:
1.2%

Duration:
p95 = 350ms
```

For a Node.js API, these three metrics give you a quick view of API health.

---

# 7. USE Method

For infrastructure/resources:

```
U = Utilization
S = Saturation
E = Errors
```

Example for a database:

```
Utilization  → CPU 75%
Saturation   → Connection pool 95%
Errors       → Connection failures
```

So a useful mental model is:

```
API
 ↓
RED

Infrastructure / Resources
 ↓
USE
```

---

# 8. Why Average Latency Is Not Enough

Suppose:

```
100 requests

95 → 100ms
5  → 10 seconds
```

Average may not look terrible.

But those 5 users had a terrible experience.

That's why we monitor:

```
p50
p95
p99
```

Especially:

> **p95/p99 are important for production APIs.**
> 

---

# 9. Health Checks

Monitoring isn't only about metrics.

Services should expose health endpoints.

For example:

```
GET /health
```

Basic response:

```json
{
  "status": "ok"
}
```

But in production, you often distinguish between:

### Liveness

> Is the process alive?
> 

```
Node.js process running?
```

### Readiness

> Can this instance actually serve traffic?
> 

```
Database available?
Required dependencies available?
Application initialized?
```

This distinction becomes especially important with Kubernetes/load balancers.

---

# 10. Example

Suppose Node.js is running:

```
Process → alive
Database → unavailable
```

Liveness:

```
OK
```

Readiness:

```
NOT READY
```

The load balancer should stop sending traffic to that instance.

This is better than restarting the application just because the database temporarily went down.

---

# 11. Metrics vs Logs

Don't use logs for everything.

### Logs

Good for:

```
Specific events
Errors
Business operations
Debugging details
```

### Metrics

Good for:

```
Rates
Counts
Latency
Resource usage
Trends
Alerts
```

For example:

Bad:

```
Log every request just to calculate error rate.
```

Better:

```
Metric:
http_requests_total
```

Then use logs when you need to investigate a specific failure.

---

# 12. Dashboards

A dashboard should answer:

> **"Is my service healthy right now?"**
> 

For an API dashboard, I'd want something like:

```
Requests/sec
Error rate
p50 latency
p95 latency
p99 latency
CPU
Memory
DB pool utilization
External API latency
```

Avoid creating dashboards with 100 meaningless graphs.

A good dashboard should help you make a decision quickly.

---

# 13. Alerting

Monitoring collects data.

**Alerting tells someone when action is required.**

Example:

```
Error rate > 5%
for 5 minutes
       ↓
Alert
       ↓
Engineer
```

---

# 14. Why "for 5 minutes"?

Consider a temporary spike:

```
12:00:00 → 6% errors
12:00:10 → 3%
12:00:20 → 1%
```

If you immediately alert at 6%, you may create unnecessary noise.

Instead:

```
Error rate > 5%
continuously for 5 minutes
```

is generally more meaningful.

The exact duration depends on the service and SLO.

---

# 15. Good vs Bad Alerts

### Bad

```
CPU > 80%
```

This may not mean users are affected.

### Better

```
API error rate > 5%
```

### Even better

```
API is violating its SLO
```

The goal is:

> **Alert on meaningful service impact, not every unusual metric.**
> 

---

# 16. Alert Fatigue

Suppose an engineer receives:

```
100 alerts/day
```

Most are irrelevant.

Eventually:

```
Alert
 ↓
Ignore
```

Then a real production problem occurs.

This is **alert fatigue**.

Good alerting should be:

```
Few
+
Relevant
+
Actionable
+
Prioritized
```

---

# 17. Alert Severity

A common approach:

```
P1 / Critical
→ Immediate response

P2 / High
→ Investigate soon

P3 / Warning
→ Can wait / investigate during working hours
```

Example:

```
Production completely unavailable
→ P1

Error rate slightly elevated
→ P2/P3 depending on impact
```

Don't treat every alert as an emergency.

---

# 18. SLI

Now we move to an important senior-level concept.

**SLI = Service Level Indicator**

It measures actual service behavior.

Example:

```
Successful requests
-------------------
Total requests
```

Suppose:

```
99,500 successful
500 failed
```

Then:

```
SLI = 99.5%
```

---

# 19. SLO

**SLO = Service Level Objective**

It's the target.

For example:

```
SLO:
99.9% successful requests
```

Current:

```
SLI = 99.95%
```

Therefore:

```
Current performance > target
```

---

# 20. SLA

**SLA = Service Level Agreement**

This is the contractual/business commitment.

Remember:

```
SLI
↓
What actually happened?

SLO
↓
What do we target?

SLA
↓
What do we promise customers?
```

---

# 21. Error Budget

Suppose:

```
SLO = 99.9%
```

Then allowed failure:

```
0.1%
```

That is the **error budget**.

For a 30-day month:

```
30 × 24 × 60
= 43,200 minutes
```

0.1%:

```
43.2 minutes
```

So approximately:

> You have 43 minutes of allowed unreliability during the month.
> 

---

# 22. Why Error Budget Matters

Imagine developers want to deploy a risky architecture change.

But:

```
Error budget
     ↓
Already almost exhausted
```

Then the team may decide:

```
Pause risky releases
       ↓
Improve reliability
```

If the system is performing well:

```
Healthy error budget
       ↓
More freedom for changes
```

This connects:

```
Reliability ↔ Development velocity
```

---

# 23. Burn Rate

Suppose your monthly error budget is:

```
43 minutes
```

But you consume:

```
20 minutes
in one hour
```

That's dangerous.

You're consuming the budget much faster than expected.

This is called:

> **Error-budget burn rate.**
> 

High burn rate can trigger an alert before the monthly SLO is completely violated.

---

# 24. Business Metrics

This is something developers sometimes forget.

You shouldn't monitor only technical metrics.

For your parking/payment system, examples might be:

```
Bookings/min
Payments/min
Refunds/min
Failed payments
Failed bookings
Wallet refills
Tap & Go purchases
Scheduler jobs
```

Imagine:

```
HTTP error rate = 0%
```

Everything looks healthy.

But:

```
Successful payments
↓
Suddenly dropped by 80%
```

Something is clearly wrong.

This is why **business-level monitoring** is valuable.

---

# 25. AWS Example

Since you work with AWS, a common production setup could be:

```
EC2
RDS
SQS
API Gateway
Lambda (if used)
      ↓
CloudWatch
      ↓
Metrics
Logs
Alarms
Dashboards
```

CloudWatch can monitor AWS infrastructure and application telemetry.

You can also integrate:

```
OpenTelemetry
     ↓
Observability backend
```

for deeper application-level observability.

---

# 26. Monitoring Architecture

A simplified production architecture:

```
                    Application
                         |
          +--------------+--------------+
          ↓              ↓              ↓
       Metrics         Logs          Traces
          |              |              |
          +--------------+--------------+
                         ↓
                   Observability
                      Backend
                         ↓
                    Dashboards
                         ↓
                     Alerting
                         ↓
                Engineer / On-call
```

Monitoring and observability therefore work together.

---

# 27. Example Production Incident

Imagine your users report:

> "Payment API is slow."
> 

Monitoring shows:

```
Traffic: normal
Error rate: normal
p95: 500ms → 4s
p99: 700ms → 8s
```

So:

```
Monitoring
   ↓
Detects latency problem
```

Then observability:

```
Trace
 ↓
Payment Service
 ↓
External Payment API
 ↓
7.5 seconds
```

Logs:

```
Payment API timeout
```

Now you know the likely root cause.

So:

```
Monitoring
→ Detect

Observability
→ Investigate
```

---

# 28. Senior-Level Monitoring Strategy

When designing monitoring for a new service, I would start with:

### Service-level

```
Rate
Error rate
p95/p99 latency
Availability
```

### Resource-level

```
CPU
Memory
Disk
Network
DB connections
Pool utilization
```

### Dependency-level

```
External API latency
External API errors
Kafka/SQS lag
Redis health
Database health
```

### Business-level

```
Successful bookings
Successful payments
Refund failures
Job failures
```

Then create alerts only for metrics where:

> **Someone can take meaningful action.**
> 

---

# 29. One Important Senior-Level Principle

Don't design monitoring as:

```
"What metrics can I collect?"
```

Design it as:

```
"What failures can happen?"
        ↓
"How will I detect them?"
        ↓
"Who needs to know?"
        ↓
"What action should they take?"
```

For example:

```
DB connection pool exhausted
        ↓
Metric
        ↓
Alert
        ↓
Engineer
        ↓
Check slow queries / pool config
```

That's much more mature monitoring design.

---

# 📌 Notes for Your Notebook

```
MONITORING & ALERTING

Monitoring:
Continuously measures system health/performance
and detects abnormal behavior.

Observability:
Helps investigate WHY the abnormal behavior happened.

Four Golden Signals:
1. Latency
2. Traffic
3. Errors
4. Saturation

RED:
Rate
Errors
Duration

USE:
Utilization
Saturation
Errors

Monitor:

Application:
- Request rate
- Error rate
- p95/p99 latency
- Throughput
- Event-loop lag

Infrastructure:
- CPU
- Memory
- Disk
- Network

Database:
- Connections
- Pool utilization
- Query latency
- Slow queries
- Locks
- Replication lag

Queues:
- Queue depth
- Consumer lag
- Processing rate
- Retries
- DLQ

Dependencies:
- Latency
- Errors
- Timeouts
- Availability

Health checks:
Liveness → Is process alive?
Readiness → Can it serve traffic?

SLI:
Actual measured service behavior.

SLO:
Target reliability/performance.

SLA:
Contractual commitment.

Error Budget:
Allowed unreliability based on SLO.

Burn Rate:
Speed at which error budget is being consumed.

Good alerts:
- Actionable
- Relevant
- Low noise
- Correct severity
- Based on meaningful service impact

Business monitoring:
Monitor business outcomes too:
- Payments
- Bookings
- Refunds
- Jobs
- Purchases

Senior mindset:

Failure scenario
     ↓
Detection metric
     ↓
Alert
     ↓
Owner
     ↓
Action / Runbook
```

### ⭐ One sentence to remember

> **Monitoring tells us when the system is unhealthy; alerting tells the right person when action is required, and good monitoring focuses on user impact, system capacity, and business health—not just CPU and memory.**
> 

With this, **Operations is now complete**:

```
Distributed Scheduling     ✅
Observability              ✅
Monitoring & Alerting      ✅
```

The next natural step is to move from **individual architecture topics → system design case studies**, where we combine these concepts into complete production systems.
