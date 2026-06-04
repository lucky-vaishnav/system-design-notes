What is an API Gateway?

Think of an API Gateway as the single entry point for all client requests.

Without API Gateway:

Mobile App
    ├── User Service
    ├── Payment Service
    ├── Trip Service
    └── Notification Service

The client must know every service URL.

With API Gateway:

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

Client only talks to the API Gateway.

Why do we need it?

Imagine Uber:

The mobile app may need:

GET /user/profile
GET /trips
POST /payment
POST /ride/request

Without a gateway:

api-user.company.com
api-trip.company.com
api-payment.company.com

The app must manage:

URLs
Authentication
Rate limits
Versioning

for every service.

An API Gateway centralizes these concerns.

Main Responsibilities
1. Request Routing

Gateway decides:

/api/users/*   -> User Service
/api/trips/*   -> Trip Service
/api/payments/* -> Payment Service

Example:

GET /api/users/123

Gateway forwards:

http://user-service/users/123
2. Authentication

Instead of every service validating JWTs:

Client
  ↓
API Gateway
  ↓ Validate JWT
  ↓
Services

Example:

Authorization: Bearer xxx

Gateway verifies:

Auth0 token
OAuth token
SSO token

before forwarding.

3. Authorization

Example:

Admin can access:
  /admin/*

Users cannot.

Gateway can block requests before they reach services.

4. Rate Limiting

Prevents abuse.

Example:

100 requests/minute/user

If user sends:

1000 requests/minute

Gateway returns:

429 Too Many Requests
5. Load Balancing

Gateway distributes traffic.

Trip Service

Instance 1
Instance 2
Instance 3

Gateway chooses where to send requests.

6. SSL Termination

Instead of HTTPS everywhere:

Client
  HTTPS
    ↓
Gateway
  HTTP
    ↓
Services

Gateway handles TLS certificates.

7. Logging

Every request can be logged.

User: auth0|123
Endpoint: /trips
Status: 200
Duration: 120ms

Useful for debugging.

8. Monitoring

Collect metrics:

Requests/sec
Error rate
Latency

Used in dashboards.

9. Request Aggregation

One client request can call multiple services.

Example:

GET /dashboard

Gateway calls:

User Service
Trip Service
Payment Service

Returns combined response.

API Gateway vs Load Balancer

Many developers confuse these.

Load Balancer

Job:

Send traffic to servers

Example:

Request
  ↓
Load Balancer
  ↓
Server1
Server2
Server3
API Gateway

Job:

Authentication
Authorization
Routing
Rate limiting
Caching
Aggregation
Monitoring

Gateway is much smarter.

Popular API Gateways
Cloud Managed
Amazon Web Services API Gateway
Microsoft API Management
Google API Gateway
Open Source
Kong
Traefik
NGINX
Apache APISIX
Real Example (Uber-like)

Request:

GET /api/trips/123

Flow:

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
Problems with API Gateway
Single Point of Failure

If gateway dies:

Everything dies

Solution:

Multiple Gateway Instances
Bottleneck

All traffic goes through it.

Need:

Horizontal Scaling
Increased Latency

Extra hop:

Client
 ↓
Gateway
 ↓
Service

adds some milliseconds.

Interview Questions
Why use API Gateway?

Answer:

Single entry point,
authentication,
routing,
rate limiting,
monitoring,
request aggregation.
API Gateway vs Load Balancer?

Load Balancer distributes traffic.

API Gateway manages APIs and cross-cutting concerns.

How does rate limiting work?

Common algorithms:

Token Bucket
Leaky Bucket
Fixed Window
Sliding Window

(learn these later)

How does authentication happen?

Usually:

JWT
OAuth
Auth0
SSO

validated at the gateway.
