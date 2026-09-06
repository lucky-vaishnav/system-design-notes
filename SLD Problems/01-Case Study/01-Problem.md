Let's start **Case Study #1**. I’ll treat this like a real senior-level system-design interview rather than teaching the solution upfront.

# Case Study 1 — Parking Reservation & Payment System

### 🎯 Problem

Design a **large-scale parking reservation system** where users can:

* Search for available parking locations.
* See available parking slots.
* Reserve a parking slot.
* Pay for the reservation.
* Cancel a reservation and receive a refund.
* View their parking history.

The system should support both **mobile/web clients** and potentially integrations with external parking providers.

---

## 1. Initial Requirements

Assume the following functional requirements:

### User

```text
Search parking
      ↓
Select parking location
      ↓
Check availability
      ↓
Reserve slot
      ↓
Make payment
      ↓
Reservation confirmed
```

Users should also be able to:

```text
View reservation
Cancel reservation
Receive refund
View parking history
```

### Important requirement

**Two users must not be able to successfully reserve the same parking slot for the same time period.**

This is going to be one of our important distributed-systems problems.

---

## 2. Initial Scale

Let's assume:

```text
100 million registered users

10 million daily active users

1 million parking searches/minute (peak)

100,000 reservation attempts/minute (peak)

20,000 payments/minute (peak)
```

We can modify these numbers later if our architecture requires it.

---

## 3. Non-Functional Requirements

The system should be:

* Highly available
* Horizontally scalable
* Fault tolerant
* Consistent where required
* Eventually consistent where acceptable
* Idempotent for payment/reservation operations
* Observable
* Secure

---

# Your First Task

Don't design everything at once.

### Step 1 — High-Level Architecture

Imagine you're in an interview and I ask:

> **"Design the high-level architecture for this parking reservation system."**

Start by telling me what major components you would use.

For example, think about:

```text
Client
   ↓
?
   ↓
?
   ↓
?
```

You can decide whether we need:

* API Gateway
* Load Balancer
* Parking Service
* Reservation Service
* Payment Service
* User Service
* Database
* Redis
* Kafka/SQS
* Distributed Lock
* External Parking Provider
* etc.

**Don't worry about being perfect.** Based on everything we've learned, give me your first architecture.

Then I'll review it like a senior interviewer and challenge your decisions.

### My Thoughts/Thinking-
So this is the first question I am answering about system design. So maybe there will be mistakes, or I am not sure what is expected when the same thing will happen in the interview. So what should be the process to explain the SDL? So you can walk me through that as well, after reviewing my answer, so that you can give me the steps as well: what to do first, and then second, and third, like that. But yeah, for the system design document, system design for this parking reservation system, first I will share or reconfirm or talk about the three things. First is the functional requirements. For this, functional requirement will be search parking, reserve parking, pay for parking, cancellation and the refund, view parking history, and integration with the external parking provider, and mobiles and web clients, right? So and the non-functional will be the system should be highly available, system should be scalable, system should be fault tolerant, system should be having, wherever required, there should be consistency in the system, and there should be eventually consistency wherever acceptable or the required, and the idempotency for the payments and the reservation should be there, and yeah, and it should be secure as well. Scaling will totally depend upon how the problem start with some limited number of users, and then if we want to scale it. So these were the three sections I will talk about. And then for system design, so according to me, there will be API gateway plus load balancer, like Nginx in the front of the parking service, or you can name it as a reservation as well. So we will, I will make it the one only, parking or the reservation service, where the client will be sending the request. So there will be a load balancer, and then this parking reservation service will talk to the related microservices, like payment service and the user service. So payment service will be responsible for payment flow and the refund flow, and the user service will be responsible for the user-related information processing, like user information, user profile, user preference. So these will be there. And as this is a reservation and payment related functionality, so we should consider for this particular part, there should be consistency. So we will use the REST API for the communication between parking service and the payment service, and there will be also a notification service. For that we can use the asynchronous communication, like Kafka, and we can send those type of event to the asynchronous services, like the notification service. And yeah, so there will be two types of communication: for payment will be REST APIs, and for notification-like services it will be asynchronous. There will be asynchronous communication using the Kafka, or maybe same as Kafka-related techniques. And yeah, and for the consistency level, maybe for the... So that will be related to the communication between the microservices and the load balancer, and as we know that number of requests may be increased in the future for payments, so we should use the load balancer in front of this payment service as well. And if we come to the idempotency, so it should be handled because reservation should be... reservation booking flow should be idempotence supported. So for that we can use database-level locks using the unique constraint, so that whenever the request will come with this unique constraint, so once the reservation is created, so the same reservation should not be created again and again. So by this we can achieve the idempotency as well, or if we want to make it in the code, then yeah, we can use the queries like select for update as well. So this is how we can achieve the idempotency as well. And for our system, because it is payment and parking, which should be. So if we talk about the CAP theorem, so the CAP theorem, we should support CP for the payment flow at least, and should be following the CP, CNP. And for notification part or the same operation like the notification, we can use first will be AP. So AP means availability and partition tolerance. And for database-level scaling, so if we really want to scale the database and we do not want some type of complexity, we can use the read-your-write approach, so that the recently updated will be read from the master DB and read operation will be performed using the read replica. And yeah, so majorly I am thinking like that, and this is my first time I am explaining system design for an app problem. So yeah, and for fault tolerance, fault tolerance, we can use some approaches, like circuit breaker will be good to have to avoid the failures from the dependent services, and we can use also Saga plus Outbox, so that if the service itself having some issue and goes down, we can use Saga plus Outbox to safely send those related events to the other services. And what things we should share or think about when we talk about fault tolerance, you can add that as well. So yeah, you can review my thought process or the way I am explaining, and share your thoughts on what should be the actual answer, and then we can go to the next phase, whatever you think, like making that system design, which you will share now with the AWS services.
