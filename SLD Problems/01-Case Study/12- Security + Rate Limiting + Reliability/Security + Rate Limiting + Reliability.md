We already covered some reliability concepts indirectly—idempotency, retries, reconciliation, circuit breakers, failure handling—so I’ll **connect those concepts rather than teach them from zero**.

# 6. Security + Rate Limiting + Reliability

For our Parking Reservation & Payment System, security isn't one component sitting in front of the application.

It's a concern across the entire flow:

```text
Client
  ↓
API Gateway / Load Balancer
  ↓
Authentication
  ↓
Authorization
  ↓
Application
  ↓
Redis / PostgreSQL / Kafka
  ↓
External Payment Provider
```

And reliability is similarly end-to-end:

```text
Request
  ↓
Timeouts
  ↓
Retries
  ↓
Idempotency
  ↓
Circuit breakers
  ↓
Fallbacks
  ↓
Reconciliation
```

We'll start with the most fundamental distinction:

> **Authentication answers "Who are you?" Authorization answers "What are you allowed to do?"**

---

# 6.1 Authentication & Authorization

## 6.1.1 Authentication

Authentication verifies identity.

Example:

```text
POST /login
       ↓
credentials
       ↓
Identity Provider
       ↓
authenticated user
```

The system may issue:

* session/cookie
* access token
* refresh token

For a large distributed system, an external Identity Provider is often useful because authentication becomes centralized.

Conceptually:

```text
                    Identity Provider
                           |
                         Token
                           ↓
Client ----------------> API
                           |
                     verify token
                           ↓
                        User ID
```

The API doesn't need to independently authenticate the user against the database on every request.

---

# 6.1.2 JWT

A common approach for APIs is a JWT access token.

Conceptually:

```text
Header.Payload.Signature
```

The token can contain claims such as:

```text
user_id
role
issuer
audience
expiration
```

The server verifies the signature and validates important claims.

But one important point:

> **A JWT being cryptographically valid does not automatically mean the request is authorized.**

We still need authorization.

---

# 6.1.3 Authorization

Suppose:

```text
User A
```

is authenticated.

That doesn't mean User A can:

```text
GET /admin/users
```

or:

```text
DELETE /parking-lots/123
```

Authorization checks permissions.

Example:

```text
User
 ├── authenticated? ✅
 ├── role = ADMIN? ❌
 └── resource permission? ❌
```

Return:

```text
403 Forbidden
```

Authentication failure is generally:

```text
401 Unauthorized
```

Authorization failure:

```text
403 Forbidden
```

This distinction is commonly asked in interviews.

---

# 6.1.4 RBAC

A simple authorization model is **Role-Based Access Control**.

Example:

```text
ADMIN
  → manage parking lots
  → view reports
  → manage users

OPERATOR
  → manage parking availability

USER
  → search
  → reserve
  → cancel own reservation
```

Conceptually:

```text
User
 ↓
Role
 ↓
Permissions
```

But we should be careful with coarse roles.

Suppose:

```text
USER
```

can cancel:

```text
reservationId = R123
```

We shouldn't only check:

```text
role == USER
```

We also need:

```text
R123 belongs to this user
```

That's **resource-level authorization**.

---

# 6.1.5 Object-Level Authorization

This is particularly important for APIs.

Bad:

```text
GET /reservations/123
```

and the backend does:

```text
if authenticated:
    return reservation 123
```

A malicious user could try:

```text
GET /reservations/124
GET /reservations/125
```

and access someone else's reservations.

Correct approach:

```text
authenticated?
      ↓
does reservation belong to user?
      ↓
allowed
```

For admin access:

```text
authenticated?
      ↓
authorized admin?
      ↓
allowed
```

This is why:

> **Authorization must be enforced server-side, not trusted from the client.**

---

# 6.1.6 Never Trust the Client

Suppose frontend sends:

```json
{
  "userId": "U123",
  "role": "ADMIN",
  "amount": 10
}
```

The backend should not trust these values.

For example, reservation price should be:

```text
parking configuration
+
duration
+
vehicle type
+
other business rules
        ↓
backend calculation
```

not:

```text
client-provided amount
```

Similarly, identity should come from:

```text
verified token/session
```

not from:

```text
req.body.userId
```

when the API is acting on behalf of the authenticated user.

---

# 6.1.7 Authentication Flow

A conceptual flow:

```text
Client
  |
  | credentials
  v
Identity Provider
  |
  | access token
  v
Client
  |
  | Authorization: Bearer <token>
  v
API
  |
  | verify token
  v
Authenticated identity
  |
  | authorization check
  v
Business logic
```

The application validates things such as:

* signature
* issuer
* audience
* expiration
* required claims

Then authorization happens.

---

# 6.1.8 Access Token vs Refresh Token

A common design:

```text
Access token
  → short-lived
  → used for APIs

Refresh token
  → longer-lived
  → used to obtain a new access token
```

Why short-lived access tokens?

If an access token leaks, limiting its lifetime reduces the window of misuse.

But refresh-token management becomes important:

* secure storage
* rotation
* revocation strategy
* expiration

The exact mechanism depends on the authentication architecture.

---

# 6.1.9 Service-to-Service Authentication

Security isn't only:

```text
User → API
```

We also have:

```text
Reservation Service → Payment Service
Reservation Service → Notification Service
Consumer → Database
```

Services need identity too.

For example:

```text
Service A
   ↓
service credential/token
   ↓
Service B
```

Service B should verify:

```text
Who is calling?
What is this service allowed to do?
```

Don't rely purely on:

```text
"It's inside our private network."
```

Network location alone shouldn't be treated as sufficient authorization.

---

# 6.1.10 Least Privilege

A very important security principle:

> **Give every user/service only the permissions it actually needs.**

For example:

```text
Notification Service
```

shouldn't have:

```text
DROP DATABASE
```

permissions.

It might only need:

```text
INSERT notification_log
SELECT user_notification_preferences
```

Similarly, a reporting service may need read access but not reservation-write permissions.

If credentials are compromised, least privilege limits the blast radius.

---

# 6.1 COMPLETE ✅

### What to remember for ANY system-design problem

When designing authentication/authorization:

```text
Authentication
     ↓
Who are you?

Authorization
     ↓
What can you do?

Resource authorization
     ↓
Can you access THIS specific resource?

Least privilege
     ↓
What is the minimum permission required?
```

And:

> **Never trust identity, role, ownership, price, or permission information simply because it came from the client.**

---

# 6.2 API Security

Now let's protect the actual API surface.

For our system:

```text
GET  /parking/search
POST /reservations
POST /payments
POST /refunds
GET  /reservations/:id
```

Each endpoint should have explicit security requirements.

---

## 6.2.1 Input Validation

Suppose:

```text
POST /reservations
```

expects:

```json
{
  "parkingLotId": "123",
  "startTime": "...",
  "endTime": "..."
}
```

The backend validates:

* required fields
* data types
* allowed values
* length
* date/time validity
* business constraints

Don't rely on frontend validation.

Frontend validation is for user experience.

Backend validation is for security and correctness.

---

# 6.2.2 SQL Injection

Never construct SQL like:

```text
"SELECT * FROM users WHERE id = '" + userId + "'"
```

Use parameterized queries / prepared statements.

Conceptually:

```text
SQL
+
parameters
```

rather than dynamically concatenating untrusted input.

ORMs/query builders can help, but developers still need to avoid unsafe raw-query construction.

---

# 6.2.3 NoSQL Injection

The same principle applies to MongoDB-style queries.

Don't directly allow arbitrary client objects to become query operators.

Validate and construct the query server-side.

---

# 6.2.4 Mass Assignment

Suppose an API accepts:

```json
{
  "name": "Lucky",
  "email": "...",
  "role": "ADMIN"
}
```

If the backend blindly updates every supplied field, a user may modify fields they shouldn't control.

Instead:

```text
Allowed fields:
name
email
```

and explicitly ignore/reject:

```text
role
permissions
accountStatus
```

This is often called **mass-assignment vulnerability**.

---

# 6.2.5 Sensitive Data Exposure

Don't return unnecessary information.

Bad:

```json
{
  "userId": "...",
  "email": "...",
  "passwordHash": "...",
  "internalNotes": "...",
  "paymentProviderData": "..."
}
```

Return only what the client needs.

Also:

> **Never log passwords, access tokens, full payment credentials, or other sensitive secrets.**

---

# 6.2.6 HTTPS/TLS

All client and service communication containing sensitive information should use encrypted transport where applicable.

```text
Client
  |
 HTTPS/TLS
  ↓
API
```

Similarly, service-to-service communication may use TLS depending on architecture.

TLS protects data in transit, but doesn't solve:

* authorization
* compromised credentials
* malicious authenticated users
* bad application logic

Security is layered.

---

# 6.2.7 Payment Security

For our payment flow:

```text
Client
  ↓
Payment API
  ↓
Payment Provider
```

The backend should avoid unnecessarily handling sensitive payment-card data if the payment provider offers tokenized/hosted payment mechanisms.

We also:

* don't trust client amount
* use payment-provider IDs
* store payment status
* use idempotency
* protect webhook endpoints
* verify webhook authenticity
* avoid logging sensitive payment information

---

# 6.2.8 Webhook Security

Our payment provider may call:

```text
POST /webhooks/payment
```

We cannot simply trust:

```text
"paymentStatus": "SUCCESS"
```

because anyone could call the endpoint.

We need to verify the webhook using the provider's supported mechanism, such as:

* signature verification
* secret
* certificate-based mechanism
* timestamp/replay protection where supported

Then:

```text
verified webhook
      ↓
process
```

And processing should be idempotent because webhooks can be duplicated.

---

# 6.2 COMPLETE ✅

### Reusable API security checklist

```text
Authentication
Authorization
Input validation
Parameterized queries
Output filtering
HTTPS/TLS
Secret protection
Webhook verification
Idempotency
Audit/security logging
```

---

# 6.3 Rate Limiting

Now let's get into rate limiting properly.

Rate limiting answers:

> **How much traffic is this client allowed to generate within a given period?**

For example:

```text
User:
100 requests/minute
```

But a senior design shouldn't use one global number for every endpoint.

---

# 6.3.1 Why We Need Rate Limiting

Without limits:

```text
Client
  ↓
100,000 requests/sec
  ↓
API
  ↓
DB ❌
```

Rate limiting can protect:

* application servers
* databases
* Redis
* external APIs
* payment providers
* expensive endpoints

It also helps mitigate abusive traffic.

---

# 6.3.2 Different Limits for Different APIs

For example:

```text
Search:
1000 requests/min/user

Reservation:
30 requests/min/user

Payment:
10 requests/min/user

Admin APIs:
lower volume but stricter authentication
```

The exact values must come from capacity testing and business requirements.

There shouldn't be one arbitrary limit for everything.

---

# 6.3.3 Rate Limit Key

We can limit by:

### User

```text
userId
```

### IP

```text
IP address
```

### API key

```text
apiKey
```

### Device

```text
deviceId
```

### Endpoint

```text
userId + endpoint
```

Often a combination is useful.

Why?

An attacker could otherwise distribute requests across multiple identities/IPs.

---

# 6.3.4 Rate Limiting in a Distributed System

This is important for our Node.js system.

Suppose we have:

```text
Node 1
Node 2
Node 3
```

If each process has:

```text
local counter = 0
```

then each server independently allows:

```text
100 requests
```

A client could potentially send:

```text
100 → Node 1
100 → Node 2
100 → Node 3
```

and get:

```text
300 requests
```

instead of 100.

Therefore:

> **Distributed rate limiting requires shared coordination when the limit is intended to be global.**

Redis is a common choice.

---

# 6.3.5 Token Bucket

One common rate-limiting algorithm is **Token Bucket**.

Imagine a bucket:

```text
Bucket capacity = 100 tokens
```

Tokens are added at:

```text
10 tokens/sec
```

Each request consumes:

```text
1 token
```

If tokens are available:

```text
request → allowed
```

If not:

```text
request → rejected/throttled
```

This allows controlled bursts.

---

# 6.3.6 Leaky Bucket

Another approach is Leaky Bucket.

Conceptually:

```text
Incoming requests
       ↓
    Queue/bucket
       ↓
fixed processing rate
```

It smooths traffic.

The exact algorithm depends on requirements.

You don't need to memorize every implementation detail.

The interview principle is:

> **Choose a rate-limiting strategy based on whether you need burst tolerance, strict smoothing, fairness, or simple request-per-window limits.**

---

# 6.3.7 Rate Limiting vs Concurrency Limiting

We covered this in 5.6, but it is worth connecting.

Rate limit:

```text
100 requests/sec
```

Concurrency limit:

```text
20 requests executing simultaneously
```

A system may need both.

For example:

```text
Payment API
Rate limit: 100/sec
Concurrency: 20
```

because payment-provider capacity might be constrained by concurrent requests rather than just request rate.

---

# 6.3.8 What Response?

When rate limited, APIs commonly return:

```text
HTTP 429 Too Many Requests
```

The response can communicate retry timing where appropriate, such as via:

```text
Retry-After
```

Clients should respect it rather than immediately retrying.

---

# 6.3.9 Rate Limiting Isn't Security by Itself

Rate limiting helps security, but it isn't authentication or authorization.

An authenticated attacker is still authenticated.

So we use multiple layers:

```text
Authentication
      +
Authorization
      +
Rate limiting
      +
Input validation
      +
Monitoring
```

---

# 6.3 COMPLETE ✅

## What to remember for ANY system-design problem

When designing rate limiting, ask:

```text
Who?
  ↓
User/IP/API key/device?

What?
  ↓
Which endpoint?

How much?
  ↓
Requests/sec/min?

Where?
  ↓
Per-instance or global?

Algorithm?
  ↓
Token bucket / leaky bucket / other

What happens at limit?
  ↓
429 / queue / degrade / reject
```

---

# 6.4 Abuse Protection

Rate limiting alone isn't enough for a high-value system.

Suppose an attacker creates:

```text
10,000 accounts
```

and stays below:

```text
100 requests/user
```

They can still generate huge aggregate traffic.

So we need additional controls.

---

## 6.4.1 Account-Level Controls

Possible mechanisms:

* email/phone verification
* suspicious activity detection
* account limits
* device/IP reputation
* progressive throttling

The exact controls depend on threat model and business requirements.

---

## 6.4.2 Expensive Endpoint Protection

Not all APIs cost the same.

For example:

```text
GET /health
```

is cheap.

But:

```text
POST /reservation
```

may involve:

```text
DB transaction
inventory lock
payment workflow
events
notifications
```

Therefore expensive operations should have stricter protection.

---

## 6.4.3 Bot/Automation Protection

For suspicious automated traffic, systems may use:

* CAPTCHA/challenges
* behavioral detection
* WAF rules
* IP/device reputation
* request-pattern analysis

Again, these are layers rather than a single solution.

---

# 6.4 COMPLETE ✅

---

# 6.5 Data Security

Now let's protect stored data.

Our system contains:

```text
User information
Reservation information
Payment information
Operational data
Audit data
```

We should classify data by sensitivity.

Example:

```text
Public
Internal
Confidential
Highly sensitive
```

Then apply appropriate controls.

---

## 6.5.1 Encryption at Rest

Sensitive stored data can be encrypted at rest.

Examples:

```text
Database
Backups
Object storage
Logs where appropriate
```

But encryption doesn't replace authorization.

If an application has unrestricted DB access, encryption at rest doesn't stop that application from reading decrypted data through normal authorized operations.

---

## 6.5.2 Encryption in Transit

As discussed:

```text
Client
  ↓ TLS
API
  ↓ TLS where applicable
Service
```

---

## 6.5.3 Passwords

If our application manages passwords directly:

> **Never store plaintext passwords.**

Use a strong password hashing algorithm designed for passwords, such as:

* Argon2
* bcrypt
* scrypt

with appropriate configuration.

---

## 6.5.4 Sensitive Logs

Avoid:

```text
password=...
token=...
cardNumber=...
secret=...
```

Logs have a wide blast radius because many engineers/systems may have access.

Use masking/redaction.

---

# 6.5 COMPLETE ✅

---

# 6.6 Secrets & Credential Management

Don't put:

```text
PAYMENT_API_KEY=abc123
```

directly into source code.

Bad:

```text
config.js
```

committed to Git.

Instead use a secure secrets-management mechanism.

Conceptually:

```text
Application
    ↓
Secret Manager
    ↓
credential
```

Examples include:

* cloud secret managers
* Vault-style systems
* environment injection from secure deployment systems

Also:

> **Rotate credentials periodically and after suspected compromise.**

And give each service its own credentials rather than sharing one master credential everywhere.

---

# 6.6 COMPLETE ✅

---

# 6.7 Reliability Patterns

Now we're moving deeper into reliability.

We've already covered several of these during payment and scaling.

The major toolbox is:

```text
Timeouts
Retries
Backoff
Jitter
Circuit breakers
Bulkheads
Idempotency
Reconciliation
Graceful degradation
Health checks
Failover
```

Let's connect them.

---

# 6.7.1 Timeout

Never allow an external call to wait indefinitely.

Example:

```text
Reservation Service
      ↓
Payment Provider
```

Without timeout:

```text
request
  ↓
waiting...
  ↓
waiting...
  ↓
waiting...
```

Threads/connections/resources remain occupied.

With timeout:

```text
request
  ↓
wait 5 sec
  ↓
timeout
```

Now the system can decide what to do next.

Important:

> **A timeout means the result may be unknown.**

It doesn't necessarily mean the operation failed.

---

# 6.7.2 Retry

Retry only when the operation is likely to succeed if attempted again.

Potentially retryable:

```text
temporary network failure
temporary 503
transient DB connection failure
```

Usually not useful:

```text
invalid request
authentication failure
business-rule rejection
```

---

# 6.7.3 Exponential Backoff

Instead of:

```text
retry immediately
retry immediately
retry immediately
```

we use increasing delays:

```text
1 sec
2 sec
4 sec
8 sec
```

with limits.

---

# 6.7.4 Jitter

If 10,000 clients all retry after exactly:

```text
4 seconds
```

we create another traffic spike.

Jitter randomizes the retry timing.

Conceptually:

```text
4 sec + random variation
```

This spreads load.

---

# 6.7.5 Circuit Breaker

Already discussed:

```text
Healthy
 ↓ failures
OPEN
 ↓
stop calling dependency
 ↓ cooldown
HALF-OPEN
 ↓
test
 ↓
healthy
CLOSED
```

This protects failing dependencies.

---

# 6.7.6 Bulkhead

Another useful reliability pattern is **bulkheading**.

Imagine one shared resource:

```text
100 connections
```

and payment calls consume all 100.

Now:

```text
Search
Reservation
History
```

can't use the resource.

Instead, we can isolate capacity:

```text
Payment → dedicated concurrency pool
Search  → separate pool
```

Conceptually:

```text
Application
 |
 +-- Payment pool
 |
 +-- Reservation pool
 |
 +-- Search pool
```

Now a failure/traffic spike in one workload is less likely to consume all resources.

This is analogous to compartments in a ship: one compartment can flood without sinking the whole ship.

---

# 6.7 COMPLETE ✅

---

# 6.8 Idempotency & Duplicate Requests

We've already covered this deeply during payment/reconciliation, so here we'll just consolidate the security/reliability perspective.

Suppose:

```text
POST /reservations
```

Client sends:

```text
Idempotency-Key: abc123
```

Request succeeds:

```text
Reservation R123 created
```

Response is lost.

Client retries with:

```text
Idempotency-Key: abc123
```

Server recognizes:

```text
abc123 → R123
```

and returns the existing result rather than creating another reservation.

---

## Where We Need It

Particularly for:

```text
Create reservation
Payment
Refund
Cancel operations
Webhook processing
Kafka consumers
```

Not every GET needs idempotency because GET should generally already be safe/repeatable.

---

# 6.8 COMPLETE ✅

---

# 6.9 Timeouts, Retries & Circuit Breakers

Let's connect these three because they are often asked together.

Suppose:

```text
Reservation Service
       ↓
Payment Provider
```

### Step 1: Timeout

Don't wait forever.

```text
5 sec → timeout
```

### Step 2: Determine whether retry is safe

If operation has idempotency:

```text
retry may be safe
```

But the result can still be unknown.

### Step 3: Backoff

```text
1 sec
2 sec
4 sec
```

plus jitter.

### Step 4: Circuit breaker

If provider keeps failing:

```text
OPEN
```

Stop continuously calling it.

### Step 5: Reconciliation

If payment status remains unknown:

```text
background reconciliation
```

queries provider status later.

This gives us:

```text
Timeout
   ↓
Retry carefully
   ↓
Backoff + jitter
   ↓
Circuit breaker if dependency unhealthy
   ↓
Reconciliation for unknown outcome
```

This is a **complete reliability strategy**, not a collection of isolated patterns.

---

# 6.9 COMPLETE ✅

---

# 6.10 End-to-End Security + Reliability

Now let's combine everything we've learned.

A request might flow:

```text id="qz2d11"
                 Client
                   |
                 HTTPS
                   |
            API Gateway / LB
                   |
            Rate Limiting
                   |
            Authentication
                   |
            Authorization
                   |
            Input Validation
                   |
             Node.js API
                   |
          +--------+--------+
          |                 |
        Redis           PostgreSQL
          |                 |
          |              Transaction
          |                 |
          |               Outbox
          |                 |
          |               Kafka
          |                 |
          |             Consumers
          |
       Cache layer
```

External payment:

```text id="k9k6s9"
Node.js
   |
 Timeout
   |
 Payment Provider
   |
 Idempotency
   |
 Webhook verification
   |
 Reconciliation
```

And around the entire system:

```text id="91f1qz"
Rate limiting
Timeouts
Retries
Backoff
Jitter
Circuit breakers
Bulkheads
Monitoring
Audit logging
Secrets management
```

---

# The Most Important Senior-Level Mental Model

When you're designing any production system, think about **four layers**:

### 1. Prevent

```text
Authentication
Authorization
Input validation
Rate limiting
Least privilege
```

### 2. Limit

```text
Concurrency limits
Connection pools
Bounded queues
Backpressure
Load shedding
```

### 3. Recover

```text
Timeouts
Retries
Backoff
Circuit breakers
Failover
Reconciliation
```

### 4. Contain

```text
Bulkheads
Failure isolation
Graceful degradation
Idempotency
```

So instead of memorizing 15 independent patterns, think:

```text
                 FAILURE
                    |
          +---------+---------+
          |                   |
       PREVENT              LIMIT
          |                   |
       security           backpressure
       validation         rate limits
                           concurrency
          |                   |
          +---------+---------+
                    |
                 RECOVER
                    |
        retry / failover / reconcile
                    |
                 CONTAIN
                    |
        bulkhead / degradation
```

---

# 6.10 COMPLETE ✅

We've now covered the entire **Major Topic 6** planned scope:

| Section                                  | Status |
| ---------------------------------------- | ------ |
| 6.1 Authentication & Authorization       | ✅      |
| 6.2 API Security                         | ✅      |
| 6.3 Rate Limiting                        | ✅      |
| 6.4 Abuse Protection                     | ✅      |
| 6.5 Data Security                        | ✅      |
| 6.6 Secrets & Credential Management      | ✅      |
| 6.7 Reliability Patterns                 | ✅      |
| 6.8 Idempotency & Duplicate Requests     | ✅      |
| 6.9 Timeouts, Retries & Circuit Breakers | ✅      |
| 6.10 End-to-End Security + Reliability   | ✅      |

# 🎯 Major Topic 6 — COMPLETE

And importantly, we didn't treat these as isolated concepts. They connect directly to what we've already learned:

**Concurrency → Scaling → Backpressure → Rate limiting → Idempotency → Payment failures → Reconciliation → Reliability.**

### Next Major Topic: **7. Observability + Disaster Recovery**

This will cover:

```text
7. Observability + DR
│
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

After that, we'll be getting much closer to the **final architecture + AWS mapping + interview walkthrough** stages.
