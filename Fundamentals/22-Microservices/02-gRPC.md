one of the most common interview questions:

> **"When should we use REST vs gRPC?"**
> 

Many candidates answer:

> "gRPC is faster."
> 

That answer is incomplete. As a senior developer, you should know **why it's faster, how it works internally, where it's used, and when NOT to use it.**

---

# gRPC (Google Remote Procedure Call)

---

# 1. Why Did Google Create gRPC?

Imagine you have 100 microservices.

```
User Service

Order Service

Payment Service

Inventory Service

Notification Service

Analytics Service

...
```

Services constantly communicate.

Example:

```
Order
   ↓
Payment
   ↓
Inventory
   ↓
Fraud
   ↓
Shipping
```

If every call uses REST:

- JSON serialization
- HTTP parsing
- Large payloads

This becomes expensive.

Google had thousands of services communicating millions of times per second.

REST became a bottleneck.

So Google built **gRPC**.

---

# 2. What is gRPC?

Think of it as:

```
REST

↓

Optimized for Service-to-Service communication
```

Instead of

```
HTTP + JSON
```

it uses

```
HTTP/2

+

Protocol Buffers (Binary)
```

---

# 3. REST vs gRPC

REST

```
Client

↓

HTTP

↓

JSON

↓

Server
```

---

gRPC

```
Client

↓

HTTP/2

↓

Protocol Buffers

↓

Server
```

Much smaller payload.

---

# 4. JSON vs Protocol Buffers

REST sends

```
{
    "id":101,
    "name":"Lucky",
    "email":"abc@gmail.com"
}
```

This is text.

---

gRPC sends

```
(Binary bytes)
```

You can't even read it.

Much smaller.

---

# Example Size

JSON

```
{
"id":1,
"name":"Lucky"
}
```

≈ 40 bytes

---

Protocol Buffer

```
(Binary)

≈ 8 bytes
```

Huge difference.

---

# 5. Why Faster?

Three reasons.

---

## A. Binary Serialization

Instead of text

↓

Binary

Smaller payload.

---

## B. HTTP/2

REST usually

```
HTTP/1.1
```

gRPC

```
HTTP/2
```

Supports:

- Multiplexing
- Header compression
- One connection

---

## C. Generated Code

No reflection.

No JSON parsing.

Everything pre-generated.

---

# 6. Protocol Buffers (.proto)

This is the heart of gRPC.

Instead of writing APIs manually

You define them.

Example

```
syntax = "proto3";

service PaymentService {

    rpc MakePayment(PaymentRequest)
        returns (PaymentResponse);

}

message PaymentRequest {

    int32 amount = 1;

    string userId = 2;

}

message PaymentResponse {

    bool success = 1;

}
```

This file becomes

Node

Java

Go

Python

C#

Automatically.

Amazing.

---

# 7. Generated Code

You never manually create

```
classPaymentRequest{}
```

gRPC generates it.

Also generates

Server

Client

Serialization

Deserialization

Everything.

---

# 8. Node.js Example

Server

```
constgrpc=require('@grpc/grpc-js');
constprotoLoader=require('@grpc/proto-loader');

functionmakePayment(call,callback) {

callback(null,{
        success:true
    });

}
```

---

Client

```
client.makePayment({

    amount:500,

    userId:"123"

},(err,res)=>{

});
```

Looks similar to function calls.

---

# 9. Why "Remote Procedure Call"?

Because

Instead of

```
paymentService.makePayment()
```

inside same application

You're calling

```
paymentService.makePayment()
```

on another server.

Looks local.

Runs remotely.

Hence

Remote Procedure Call.

---

# 10. Streaming

REST

```
One Request

↓

One Response
```

Done.

---

gRPC supports

Four communication styles.

---

## Unary

Exactly like REST.

```
Request

↓

Response
```

---

## Server Streaming

Client asks once.

Server keeps sending.

Example

Stock Prices.

```
Client

↓

Give prices

↓

100

↓

101

↓

102

↓

103

...
```

One request.

Many responses.

---

## Client Streaming

Opposite.

Client keeps uploading.

Server responds once.

Example

Large file upload.

```
Chunk1

Chunk2

Chunk3

↓

Done
```

---

## Bidirectional Streaming

Both sides keep talking.

Example

Uber Driver Location.

```
Driver

↓

Location

↓

Server

↓

ETA

↓

Driver

↓

Location

↓

Server
```

Perfect for realtime.

---

# 11. Where Is gRPC Used?

Internal communication.

Example

Order

↓

Payment

↓

Inventory

↓

Fraud

↓

Shipping

Perfect.

---

# NOT for Browsers

Browser cannot directly call gRPC easily.

Usually

```
Browser

↓

REST

↓

API Gateway

↓

gRPC

↓

Microservices
```

Very common.

---

# 12. Why Not Browser?

Browsers understand

```
HTTP

JSON
```

Not Protocol Buffers directly.

Need

```
gRPC-Web
```

or Gateway.

---

# 13. REST vs gRPC

REST

Pros

```
Simple

Readable

Browser Friendly

Public APIs
```

---

Cons

```
Large payload

Slower

JSON parsing
```

---

gRPC

Pros

```
Fast

Small payload

Streaming

Code generation

Strong typing
```

---

Cons

```
Hard to debug

Binary

Not browser friendly

Learning curve
```

---

# 14. Production Example

Amazon

Browser

↓

REST

↓

API Gateway

↓

gRPC

↓

Order

↓

Payment

↓

Inventory

↓

Shipping

Exactly how many companies work.

---

# 15. Uber Example

Passenger App

↓

REST

↓

Ride Service

↓

gRPC

↓

Pricing

↓

Driver

↓

Payment

↓

Maps

Internal traffic is gRPC.

---

# 16. Netflix

TV

↓

REST

↓

Gateway

↓

gRPC

↓

Recommendation

↓

Streaming

↓

History

---

# 17. When Should I Use gRPC?

Ask

Is communication

```
Service

↓

Service
```

?

Yes

↓

Consider gRPC.

---

Need

```
Streaming
```

Use gRPC.

---

Need

```
Millions of calls
```

Use gRPC.

---

Need

```
Browser
```

REST.

---

Need

```
Public API
```

REST.

---

# 18. REST + gRPC Together

Very common.

```
Browser

↓

REST

↓

Gateway

↓

gRPC

↓

Microservices
```

Best of both.

---

# 19. Interview Questions

---

## Why is gRPC faster?

Answer

```
Binary serialization

HTTP/2

Smaller payload

Multiplexing

Generated code
```

---

## Does gRPC replace REST?

No.

Different use cases.

---

## Can browser call gRPC?

Normally no.

Need

```
gRPC-Web

or

Gateway
```

---

## Does gRPC use HTTP?

Yes.

HTTP/2.

---

## Difference Between REST and gRPC?

| REST | gRPC |
| --- | --- |
| JSON | Protocol Buffers |
| HTTP/1.1 (typically) | HTTP/2 |
| Human readable | Binary |
| Browser friendly | Service friendly |
| CRUD | High-performance communication |

---

# 20. Real Architecture

```
Browser

↓

REST

↓

API Gateway

↓

gRPC

↓

User

↓

Payment

↓

Inventory

↓

Notification

↓

Kafka Events
```

Notice how everything you've learned connects.

---

# 21. Common Mistakes

❌

Use gRPC for public APIs.

Usually wrong.

---

❌

Use REST between 500 internal services.

May become slow.

---

❌

Think gRPC replaces Kafka.

Wrong.

gRPC

```
Synchronous
```

Kafka

```
Asynchronous
```

Different purposes.

---

# 22. REST vs gRPC vs Kafka

| REST | gRPC | Kafka |
| --- | --- | --- |
| Sync | Sync | Async |
| Browser | Internal | Events |
| JSON | Binary | Messages |
| CRUD | High Performance | Event Driven |

---

# Senior Developer Decision Tree

```
Need immediate response?

        Yes
         │
         ├── Browser?
         │      │
         │      ├── Yes → REST
         │      │
         │      └── No → gRPC
         │
         No
         │
         └── Kafka / RabbitMQ / SQS
```

This decision tree is excellent for interviews.

---

# Complete Interview Notes

```
gRPC

Created by:
Google

Uses:
HTTP/2
Protocol Buffers

Advantages
- Fast
- Binary serialization
- Streaming
- Small payload
- Strong typing
- Code generation

Disadvantages
- Hard debugging
- Binary format
- Not browser friendly

Streaming Types
1. Unary
2. Server Streaming
3. Client Streaming
4. Bidirectional Streaming

Use Cases
- Internal microservices
- High throughput
- Low latency
- Realtime communication

Avoid
- Public APIs
- Browser-first applications

REST vs gRPC

REST
- Browser
- JSON
- Public APIs

gRPC
- Internal
- Binary
- HTTP/2
- High performance
```

---

You now have a strong foundation for **how microservices communicate**. The next topics—**Circuit Breaker**, **Bulkhead**, and **Service Mesh**—will answer the next question:

> **What happens when one of those services becomes slow, unavailable, or starts failing?**
>
