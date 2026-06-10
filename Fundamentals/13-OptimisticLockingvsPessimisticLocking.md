This builds directly on everything we've learned:

```
Transactions
→ Isolation Levels
→ MVCC
→ Locks
→ Deadlocks
→ SELECT FOR UPDATE
→ SKIP LOCKED
→ Optimistic vs Pessimistic Locking
```

---

# First Question

Imagine:

```
Bank Account Balance = ₹1000
```

User A and User B both open the screen.

Both see:

```
₹1000
```

Now:

```
User A withdraws ₹100
User B withdraws ₹200
```

How do we prevent data corruption?

There are two major approaches.

---

# 1. Pessimistic Locking

"Pessimistic" means:

```
Assume conflicts WILL happen.
```

So lock first.

Example:

```
BEGIN;

SELECT*
FROM accounts
WHERE id=1
FOR UPDATE;
```

Now:

```
User A gets lock
User B waits
```

After A commits:

```
User B proceeds
```

---

## Pros

```
✔ Very safe
✔ Easy to reason about
✔ Prevents concurrent modifications
```

---

## Cons

```
❌ Waiting
❌ Lock contention
❌ Possible deadlocks
```

---

# 2. Optimistic Locking

"Optimistic" means:

```
Assume conflicts are rare.
```

Don't lock.

Instead:

```
Detect conflict during update.
```

---

Example table:

```
id | balance | version
---+---------+--------
1  | 1000    | 1
```

User A reads:

```
balance=1000
version=1
```

User B reads:

```
balance=1000
version=1
```

---

User A updates:

```
UPDATE accounts
SET balance=900,
    version=2
WHERE id=1
AND version=1;
```

Success.

---

User B tries:

```
UPDATE accounts
SET balance=800,
    version=2
WHERE id=1
AND version=1;
```

But database now has:

```
version=2
```

So:

```
0 rows updated
```

Conflict detected.

---

# What happens then?

Application says:

```
Data changed.
Please refresh and try again.
```

or retries automatically.

---

# Pros

```
✔ No waiting
✔ No locks
✔ High scalability
✔ Great for web applications
```

---

# Cons

```
❌ Update may fail
❌ Retry logic needed
❌ More application code
```

---

# Real-world examples

### Pessimistic Locking

```
Parking reservation
Seat booking
Payment processing
Inventory deduction
```

You usually don't want two users buying the last seat.

---

### Optimistic Locking

```
User profile updates
Settings pages
Admin panels
Product information
```

Conflicts are rare, so locking would be wasteful.

---

# In PostgreSQL

Pessimistic:

```
SELECT ...FORUPDATE
```

---

Optimistic:

```
UPDATEtable
SET version= version+1
WHERE id= ?
AND version= ?
```

or using:

```
updated_at timestamp
```

as the version check.

#### Using `updated_at` for Optimistic Locking

Instead of maintaining a separate:

```
version = 1
version = 2
version = 3
```

column, some systems use:

```
updated_at
```

to detect whether data changed.

---

Example

Table:

```
users

id | name  | updated_at
---+-------+--------------------
1  | Lucky | 2025-08-01 10:00
```

---

User A loads page.

Gets:

```
name = Lucky
updated_at = 2025-08-01 10:00
```

---

User B loads page.

Gets same values.

---

User A updates name.

```
Lucky
→ Lucky V
```

Database becomes:

```
name = Lucky V
updated_at = 2025-08-01 10:05
```

---

Now User B submits old form.

Instead of:

```
UPDATE users
SET name='Lucky Kumar'
WHERE id=1;
```

we do:

```
UPDATE users
SET name='Lucky Kumar'
WHERE id=1
AND updated_at='2025-08-01 10:00';
```

---

What happens?

Database currently contains:

```
updated_at = 2025-08-01 10:05
```

So:

```
0 rows updated
```

Meaning:

```
Someone modified the record after you loaded it.
```

Conflict detected ✅

---

#### Why do this?

Avoids adding:

```
version INTEGER
```

column.

Many tables already have:

```
created_at
updated_at
```

so developers reuse them.

---

### Version vs updated_at

#### Version Column

```
version = 1
version = 2
version = 3
```

Pros:

```
✔ Explicit
✔ Very common
✔ Easy to understand
```

---

#### updated_at

```
2025-08-01 10:00
2025-08-01 10:05
```

Pros:

```
✔ No extra column needed
```

Cons:

```
❌ Depends on timestamp precision
❌ Slightly less explicit
```

---

### Which is used more?

In enterprise systems:

```
Version column
```

is generally preferred.

In smaller CRUD applications:

```
updated_at
```

is quite common.

---

# Interview Question

> Which is more scalable?
> 

Answer:

```
Optimistic locking is usually more scalable because it avoids locks and waiting. However, pessimistic locking is safer when conflicts are common or data consistency is critical.
```

---

Think about:

```
Refund processing
Trip payment
Permit purchase
Parking reservation
```

Those are typically:

```
Pessimistic locking
```

using:

```
SELECT ...FORUPDATE
```

because money and reservations are involved.

# One interview-level takeaway

Optimistic locking does **not** prevent two users from reading the same data.

Instead it says:

```
"Go ahead and edit.
We'll check at update time whether someone changed it."
```

That's the core idea behind optimistic locking.
