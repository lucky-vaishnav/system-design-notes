You've already learned:

- ✅ REST
- ✅ gRPC
- ✅ Kafka

Now the obvious question becomes:

> **What if one of those services is down?**
> 

That's where **Circuit Breaker** comes in.

This is one of the most important microservices resilience patterns and is asked frequently in senior interviews.

---

# Circuit Breaker

---

# 1. The Problem It Solves

Imagine this architecture:

```
User

↓

Order Service

↓

Payment Service

↓

Bank API
```

Customer places an order.

Order Service calls Payment Service.

Payment Service calls Bank.

Everything works.

---

Suddenly...

Bank API becomes slow.

Instead of taking

```
100 ms
```

it starts taking

```
30 seconds
```

Now every request waits.

---

After a few users:

```
Request 1 → waiting

Request 2 → waiting

Request 3 → waiting

Request 4 → waiting

...
```

Soon:

```
Thread Pool Full

↓

CPU High

↓

Memory High

↓

Timeouts

↓

Order Service also crashes
```

This is called:

## Cascading Failure

One service failed.

Now the whole system fails.

---

# 2. Real Production Example

Uber

```
Ride Service

↓

Payment Service

↓

Stripe
```

Stripe is down.

Without protection:

Every ride booking still tries Stripe.

Thousands of requests pile up.

Ride Service becomes slow.

Eventually Ride Service also crashes.

---

The actual problem isn't Stripe anymore.

It's that we **keep trying** to call Stripe.

---

# 3. Circuit Breaker Idea

Think about your home's electrical circuit breaker.

Normal:

```
Electricity

↓

Bulb
```

Short circuit.

Instead of burning the house...

Breaker trips.

```
Electricity

×

Stopped
```

House survives.

Same idea in software.

---

# 4. Software Circuit Breaker

Normally

```
Order

↓

Payment
```

If Payment starts failing...

Circuit Breaker says:

```
Stop calling Payment.
```

Immediately.

No waiting.

No timeout.

---

# 5. Three States

Every interview asks this.

Circuit Breaker has **3 states**.

```
Closed

↓

Open

↓

Half Open
```

---

# State 1 — Closed

Normal state.

Everything works.

```
Order

↓

Payment

↓

Success
```

Requests pass through.

---

# State 2 — Open

Too many failures detected.

Breaker opens.

Now:

```
Order

↓

Circuit Breaker

×

Payment NOT called
```

Instead of waiting 30 seconds...

Return immediately.

Example

```
503

Payment Service Unavailable
```

System remains healthy.

---

# State 3 — Half Open

After some time

Breaker asks:

```
Maybe service recovered?
```

Allow one request.

If successful

↓

Close breaker.

If failed

↓

Open again.

---

Complete Flow

```
Closed

↓

Failures

↓

Open

↓

Wait 30 sec

↓

Half Open

↓

Success?

↓

Closed

↓

or

↓

Open
```

---

# 6. Visual Example

Closed

```
✓

✓

✓

✓
```

Failure

```
×

×

×

×

×
```

Breaker opens

```
×

×

×

No Requests
```

Wait

↓

Half Open

```
One Test Request
```

Success

↓

Closed

---

# 7. Why Is It Better?

Without Circuit Breaker

```
1000 requests

↓

1000 failures

↓

1000 timeouts

↓

Server dies
```

---

With Circuit Breaker

```
1000 requests

↓

5 failures

↓

Breaker Opens

↓

995 fast failures
```

Huge difference.

---

# 8. Fast Failure

Important interview phrase.

Instead of

```
Wait 30 seconds
```

Return

```
Service Unavailable
```

in

```
5 ms
```

Fast failure is better than slow failure.

---

# 9. Node.js Example

Popular library

```
opossum
```

Example

```jsx
const CircuitBreaker = require("opossum");

const breaker = new CircuitBreaker(makePayment,{
    timeout:3000,
    errorThresholdPercentage:50,
    resetTimeout:10000
});
```

---

Call

```jsx
breaker.fire(paymentData);
```

Instead of

```jsx
makePayment();
```

---

# 10. Configuration

Most important parameters.

---

## Timeout

How long before considering request failed.

Example

```
3 seconds
```

---

## Failure Threshold

Suppose

```
50%
```

10 requests.

6 failed.

Breaker opens.

---

## Reset Timeout

How long before Half Open.

Example

```
30 seconds
```

---

# 11. Fallback

Very important.

If Payment unavailable.

Return

```jsx
{
 paymentStatus:"Pending"
}
```

Instead of crashing.

Or

Use cached data.

Example

Maps Service down.

Return

```
Last Known Route
```

---

# 12. Real Example

Product Details

↓

Recommendation Service

Recommendation fails.

Do you fail whole page?

No.

Fallback

```
Show page

Without recommendations
```

User happy.

---

# 13. Netflix Example

Movie Page

↓

Recommendation

↓

Rating

↓

Reviews

Reviews fail.

Still show movie.

That's fallback.

---

# 14. Amazon Example

Checkout

↓

Inventory

↓

Recommendation

↓

Ads

Ads fail.

Ignore.

Checkout still works.

---

# 15. Circuit Breaker + Retry

Question:

Retry first?

Or breaker first?

Production:

```
Retry

↓

Still Failing

↓

Circuit Breaker Opens
```

Usually:

Few retries.

Then breaker.

---

# 16. Circuit Breaker + Timeout

Always together.

Never wait forever.

```
Timeout

↓

Retry

↓

Circuit Breaker
```

---

# 17. Circuit Breaker + Load Balancer

Example

```
Payment

Server1

Server2

Server3
```

Server2 failing.

Breaker opens only for Server2.

Traffic continues to others.

---

# 18. Common Mistakes

Open breaker forever.

Wrong.

Need Half Open.

---

Retry forever.

Wrong.

Need limits.

---

No timeout.

Wrong.

Always set timeout.

---

# 19. When To Use

Use for

```
REST

gRPC

External APIs

Database

Redis

Payment Gateway

Maps API
```

Anything remote.

---

# Don't Use

Simple function call

```jsx
calculateTotal()
```

No network.

No breaker.

---

# 20. Real Architecture

```
Browser

↓

Gateway

↓

Order Service

↓

Circuit Breaker

↓

Payment

↓

Stripe
```

Stripe fails.

Breaker protects Order Service.

---

# 21. Interview Questions

---

## Why Circuit Breaker?

Prevent cascading failures.

---

## Three States?

```
Closed

Open

Half Open
```

---

## Difference Between Retry and Circuit Breaker?

Retry

```
Try again.
```

Circuit Breaker

```
Stop trying.
```

---

## Can They Work Together?

Yes.

Retry first.

Breaker after repeated failures.

---

## What Is Fast Failure?

Fail immediately.

Don't wait.

---

# 22. Real Company Examples

Netflix

Uses Circuit Breaker heavily.

Historically with **Hystrix** (now discontinued), newer resilience libraries replaced it.

---

Uber

Payment

Maps

Driver

All protected.

---

Amazon

External services

↓

Protected by breakers.

---

# Complete Flow

```
REST/gRPC

↓

Timeout

↓

Retry

↓

Circuit Breaker

↓

Fallback

↓

User
```

This sequence is extremely common in production.

---

# Interview Notes

```
Circuit Breaker

Purpose:
Prevent cascading failures.

States:
1. Closed
2. Open
3. Half Open

Configuration:
- Timeout
- Failure Threshold
- Reset Timeout

Works With:
- Retry
- Timeout
- Fallback

Advantages:
- Fast failure
- Protects services
- Prevents overload
- Better user experience

Use For:
REST
gRPC
External APIs
Databases
Redis
Payment Gateway

Don't Use:
Local function calls

Flow:
Request
↓
Timeout
↓
Retry
↓
Circuit Breaker
↓
Fallback
```

---

# Today's Progress

You have now completed:

```
Phase C

✅ Microservices Communication

✅ gRPC

✅ Circuit Breaker
```

---

## Next Topic

The next topic is **Bulkhead**.

Many people confuse it with Circuit Breaker.

A simple way to remember:

- **Circuit Breaker** → *Stop calling a failing service.*
- **Bulkhead** → *Prevent one failing part of your application from consuming all resources and taking down the rest.*

By the end of Bulkhead, you'll understand why both patterns are often used together in production microservices.

---

# Q1. Is a Circuit Breaker used only for handling failures, or can it also be used when there are too many parallel requests and the API response time is increasing?

## Answer

Excellent question.

The short answer is:

**Yes, a Circuit Breaker can open because of increased response time**, but **its primary purpose is still failure detection**, not traffic control.

Let's understand the difference.

### Scenario 1 — Service Starts Returning Errors

```
Payment API

↓

500 Error

500 Error

500 Error
```

Circuit Breaker counts failures.

Eventually:

```
Circuit Opens
```

This is the most common case.

---

### Scenario 2 — Service Becomes Very Slow

Normally:

```
Response Time = 100 ms
```

Now suddenly:

```
Response Time = 15 sec
```

Even if the API eventually returns **200 OK**, your application may have a timeout configured:

```
Timeout = 3 sec
```

After 3 seconds:

```
Timeout Exception

↓

Circuit Breaker counts it as a failure
```

So yes, **high latency can indirectly open the Circuit Breaker** because timeouts are treated as failures.

---

### Scenario 3 — Too Many Parallel Requests

Suppose:

```
1000 requests/sec

↓

Payment Service
```

The service is overloaded.

As load increases:

```
Response Time

100 ms

↓

500 ms

↓

2 sec

↓

5 sec

↓

Timeouts
```

Now requests start timing out.

Circuit Breaker sees many failures:

```
Timeout

↓

Timeout

↓

Timeout

↓

Open Circuit
```

So the Circuit Breaker doesn't open because **there are many requests**.

It opens because **those requests start failing or timing out**.

---

### Important Difference

Traffic control is usually handled by:

- Rate Limiter
- Load Balancer
- Autoscaling
- Queue
- Bulkhead (our next topic)

Circuit Breaker handles:

```
Unhealthy dependency
```

not

```
High traffic
```

Although high traffic may eventually make the dependency unhealthy.

---

# Interview Answer

> A Circuit Breaker opens when requests continuously fail or time out. High traffic alone does not open the breaker, but if that traffic causes increased latency and timeouts, those timeouts are treated as failures and can cause the circuit to open.
> 

---

# Q2. If there are multiple Payment Service instances behind a Load Balancer, is the Circuit Breaker specific to each service instance? If the circuit opens for one instance, will requests to that instance fail while the other instances continue working?

## Answer

Excellent question.

This is exactly how production systems work.

Suppose:

```
               Load Balancer

             /      |      \

      Payment-1  Payment-2  Payment-3
```

Now imagine:

```
Payment-2
```

starts failing.

---

### Without Circuit Breaker

Load Balancer still sends traffic:

```
Request

↓

Payment-2

↓

Timeout

↓

Timeout

↓

Timeout
```

Users continue getting failures.

---

### With Circuit Breaker

If the Circuit Breaker is associated with **Payment-2**, then:

```
Payment-2

↓

Circuit Open
```

Requests are immediately rejected (or rerouted, depending on the architecture).

Meanwhile:

```
Payment-1

✓ Healthy

Payment-3

✓ Healthy
```

continue serving requests normally.

---

### Is the Circuit Breaker Centralized?

Usually **no**.

It is commonly implemented in the **client making the call**.

For example:

```
Order Service

↓

Circuit Breaker

↓

Payment Service
```

The Order Service keeps track of failures when calling Payment Service.

---

### How Does This Work with Multiple Instances?

There are two common approaches.

### Option 1 — One Circuit Per Service (Most Common)

The client only knows:

```
Payment Service
```

The Load Balancer decides which instance to call.

If many requests fail regardless of the instance selected, the client opens the breaker for the whole Payment Service.

---

### Option 2 — One Circuit Per Instance (More Advanced)

Service discovery knows:

```
Payment-1

Payment-2

Payment-3
```

Each instance has its own Circuit Breaker.

Example:

```
Payment-1

Closed

✓

Payment-2

Open

×

Payment-3

Closed

✓
```

The client skips only:

```
Payment-2
```

and continues calling the healthy instances.

This approach is used by systems like:

- Istio
- Envoy
- Linkerd
- Netflix Ribbon (historically)
- Some advanced service discovery/load-balancing libraries

---

### Which One Is More Common?

For most Node.js backend applications:

```
Client

↓

Load Balancer

↓

Payment Service
```

The Circuit Breaker is typically applied to the **service as a whole**.

Per-instance Circuit Breakers are more common in large-scale service mesh or advanced service discovery environments.

---

# Interview Answer

> A Circuit Breaker is typically implemented in the client calling another service. In simple architectures, it protects the entire target service. In more advanced systems with service discovery or a service mesh, each service instance can have its own Circuit Breaker, allowing traffic to continue flowing only to healthy instances while avoiding unhealthy ones.
> 

---

Both of these questions are the kind of follow-up questions interviewers ask after "Explain Circuit Breaker," so they're excellent questions to explore.
