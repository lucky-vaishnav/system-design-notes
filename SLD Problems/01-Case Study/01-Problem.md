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

