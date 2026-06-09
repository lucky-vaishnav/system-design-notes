# 🔥 1. Why do we need Locks?

Even with MVCC:

```
Multiple users can read/write at same time
BUT conflicts still happen when same data is modified
```

👉 Locks are used to **protect critical sections of data**

---

# 📦 Simple real-world example

Imagine:

```
Seat booking system
Seat = A1 (only 1 available)
```

Two users try:

- User A books A1
- User B books A1 at same time

👉 Without locks → DOUBLE BOOKING ❌

---

# 🧠 What Lock does

```
Lock = "I am working on this row, others wait"
```

---

# 🔐 2. Types of Locks (VERY IMPORTANT)

## 🟢 1. Row-Level Lock

Locks only one row.

```
SELECT*FROM bookingsWHERE id=1FORUPDATE;
```

👉 Only that row is locked

---

## 🔵 2. Table-Level Lock

Locks entire table.

```
Used for schema changes or bulk operations
```

Example:

```
ALTERTABLE bookingsADDCOLUMN status;
```

---

## 🟡 3. Shared Lock (Read Lock)

```
Multiple users can read, but cannot write
```

---

## 🔴 4. Exclusive Lock (Write Lock)

```
Only one transaction can write/read-modify
Others are blocked
```

---

# 🧠 3. SELECT FOR UPDATE (VERY IMPORTANT)

This is how backend engineers prevent race conditions.

```
BEGIN;

SELECT*FROM bookings
WHERE seat_id='A1'
FORUPDATE;
```

👉 This locks the row

Now:

- User A locks row
- User B must wait

---

# 🚀 4. Concurrency Control (big idea)

Concurrency control = how DB handles multiple users safely

There are 2 main strategies:

---

## 🟢 A. Pessimistic Locking

```
Lock first → then work
```

Used when conflicts are expected.

Example:

```
booking seats, payments
```

---

## 🔵 B. Optimistic Locking

```
Work freely → check conflict before commit
```

Uses version column:

```
UPDATE bookings
SET status='BOOKED', version= version+1
WHERE id=1AND version=5;
```

👉 If version changed → update fails

---

# 🔥 5. What is Deadlock?

Deadlock = **two transactions waiting forever**

---

# 📦 Simple example

### Transaction A:

```
LOCKrow1
WAITforrow2
```

### Transaction B:

```
LOCKrow2
WAITforrow1
```

👉 Both wait forever = DEADLOCK ❌

---

# ⚠️ DB behavior

PostgreSQL detects deadlock and:

```
❌ aborts one transaction automatically
```

---

# 🧠 6. How DB prevents Deadlocks

DB uses:

## ✔ Wait-for graph detection

It checks:

```
who is waiting for whom
```

If cycle found → kill one transaction

---

# 🔥 7. Real production deadlock example

In your domain (booking/payment):

```
1. update booking + payment
2. update payment + booking
```

If order is different → deadlock risk

---

# 💡 Golden rule to avoid deadlocks

```
Always lock rows in SAME ORDER
```

---

# 🧠 8. Lock waiting vs blocking

| Term | Meaning |
| --- | --- |
| Blocking | one query holds lock |
| Waiting | other query waiting for it |

---

# 🚀 9. MVCC + Locks together (VERY IMPORTANT)

```
MVCC → handles reads
Locks → handle write conflicts
```

So PostgreSQL uses BOTH:

- MVCC = performance
- Locks = correctness

---

# 🧠 One-line interview answer

```
Locks are mechanisms used in databases to control concurrent access to data, ensuring consistency during write conflicts, while deadlocks occur when two or more transactions wait indefinitely for each other’s locked resources, which the database resolves using detection and rollback mechanisms.
```

---

# 🔥 Key takeaway

```
MVCC = no blocking reads
Locks = prevent conflicting writes
Deadlocks = circular lock dependency
```

# Question

> When we use `SELECT ... FOR UPDATE`, does it lock reads too?
> 
> 
> Or are reads handled separately by MVCC?
> 

---

# 🔥 Short answer

```
MVCC handles normal reads (no locks needed)
SELECT FOR UPDATE locks ONLY write conflicts (rows being modified)
```

👉 So both work together, but they do different jobs.

---

# 🧠 1. Normal SELECT (no lock)

```
SELECT*FROM bookingsWHERE id=1;
```

### What happens:

- Uses MVCC snapshot
- No locking
- No blocking

👉 Even if someone is updating the row, you still get a consistent old version

---

# 📦 Example

User A:

```
SELECT balance → 100
```

User B:

```
UPDATE balance = 200
```

👉 User A still sees 100 (MVCC snapshot)

---

# 🔥 2. SELECT FOR UPDATE (locks row)

```
SELECT*FROM bookingsWHERE id=1FORUPDATE;
```

### What happens:

- Row is LOCKED
- Other transactions CANNOT update it
- Others must WAIT

---

# ⚠️ Important clarification

👉 It does NOT block normal reads

So:

| Operation | Allowed? |
| --- | --- |
| SELECT | ✅ Yes |
| UPDATE | ❌ Blocked |
| DELETE | ❌ Blocked |
| SELECT FOR UPDATE | waits |

---

# 🧠 3. So how do MVCC + Locks work together?

Think like this:

```
MVCC = reading engine (snapshot system)
Locks = writing safety system
```

---

# 🚀 Combined flow

## Case:

Two users accessing same row

---

### Step 1: User A reads

```
SELECT balanceFROM account;
```

👉 MVCC gives snapshot → no lock

---

### Step 2: User B tries update

```
UPDATE accountSET balance=200;
```

👉 DB checks lock rules + MVCC visibility

---

### Step 3: If User A uses FOR UPDATE

```
SELECT*FROM accountFORUPDATE;
```

👉 Now row is locked → User B must wait

---

# 🧠 Key insight (VERY IMPORTANT)

```
✔ Reads = MVCC (no locks)
✔ Writes = locks + MVCC version control
```

---

# 🔥 Common misconception

> “Does SELECT FOR UPDATE lock read also?”
> 

👉 NO

It does NOT stop:

```
✔ normal SELECT queries
```

It only blocks:

```
✔ UPDATE
✔ DELETE
✔ other FOR UPDATE
```

# 🧠 One-line interview answer

```
MVCC handles consistent non-blocking reads using snapshots, while SELECT FOR UPDATE applies row-level locks to prevent concurrent write conflicts; both systems work together where reads are lock-free and writes are controlled using locks plus MVCC versioning.
```

---

# 🚀 Final mental model (VERY IMPORTANT)

```
READ → MVCC (no lock)
WRITE → Lock + MVCC check
```
