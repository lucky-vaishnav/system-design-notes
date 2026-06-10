# First concept: SELECT FOR UPDATE

Suppose we have:

```
parking_slots

id| available
---+----------
1|true
```

Two users try to reserve the same slot.

Without locking:

```
User A reads available=true
User B reads available=true

Both update to reserved
```

Double booking ❌

---

With:

```
BEGIN;

SELECT*
FROM parking_slots
WHERE id=1
FORUPDATE;
```

PostgreSQL places a row lock.

Now:

```
User A gets lock
User B waits
```

No double booking ✅

---

# Problem with FOR UPDATE

Imagine 100 workers processing jobs.

Worker 1:

```
SELECT*
FROM jobs
WHERE status='PENDING'
LIMIT1
FORUPDATE;
```

Worker 2 runs same query.

Worker 2 waits.

Worker 3 waits.

Worker 4 waits.

Not efficient.

---

# NOWAIT

```
SELECT*
FROM jobs
WHERE id=1
FORUPDATE NOWAIT;
```

Meaning:

```
If row locked
→ don't wait
→ throw error immediately
```

Example:

```
Row already locked
```

Postgres returns:

```
could not obtain lock on row
```

---

# When is NOWAIT useful?

Example:

```
Payment processing
```

If another transaction is already handling the payment:

```
Don't wait
Return error immediately
```

and let the application decide.

---

# SKIP LOCKED

This is extremely important.

```
SELECT*
FROM jobs
WHERE status='PENDING'
FORUPDATE SKIP LOCKED
LIMIT1;
```

Meaning:

```
If row locked
→ ignore it
→ pick another row
```

---

# Real-world example

Jobs table:

```
Job 1
Job 2
Job 3
Job 4
```

Worker A locks:

```
Job 1
```

Worker B executes:

```
FORUPDATE SKIP LOCKED
```

Instead of waiting:

```
Job 2
```

is returned.

Worker C:

```
Job 3
```

Worker D:

```
Job 4
```

All workers stay busy.

---

# Why SKIP LOCKED is popular

Used in:

- Job queues
- Notification processing
- Email sending
- Report generation
- Background schedulers

Many companies build internal queues using PostgreSQL and:

```
FORUPDATE SKIP LOCKED
```

instead of using Kafka or RabbitMQ.

---

# Comparison

| Feature | FOR UPDATE | NOWAIT | SKIP LOCKED |
| --- | --- | --- | --- |
| Lock row | ✅ | ✅ | ✅ |
| Wait if locked | ✅ | ❌ | ❌ |
| Error if locked | ❌ | ✅ | ❌ |
| Skip locked row | ❌ | ❌ | ✅ |

---

# Real interview question

**How would you build multiple workers processing jobs without processing the same job twice?**

Expected answer:

```
Use transactions with
SELECT ... FOR UPDATE SKIP LOCKED
```

This allows multiple workers to safely process different rows concurrently.

### Question -Can NOWAIT , SELECT for UPDATE  also can be used to avoid processing the same job twice?

```
YES, FOR UPDATE and NOWAIT can also prevent duplicate processing.
The difference is efficiency and behavior when a row is already locked.
```

Let's compare.

---

### Option 1: FOR UPDATE

Worker A:

```
SELECT*FROM jobs
WHERE status='PENDING'
LIMIT1
FORUPDATE;
```

Gets Job 1 and locks it.

Worker B runs same query.

```
Worker B waits
```

until Worker A commits.

### Result

```
✔ Job not processed twice
✔ Safe
❌ Worker B sits idle waiting
```

---

### Option 2: FOR UPDATE NOWAIT

Worker A locks Job 1.

Worker B:

```
SELECT*FROM jobs
FORUPDATE NOWAIT;
```

Immediately gets:

```
ERROR: could not obtain lock
```

### Result

```
✔ Job not processed twice
✔ No waiting
❌ Application must handle error and retry
```

So NOWAIT works, but your code must do something like:

```
try lock
if fail
pick another job
or retry later
```

---

### Option 3: FOR UPDATE SKIP LOCKED

Worker A locks Job 1.

Worker B:

```
SELECT ...
FORUPDATE SKIP LOCKED;
```

Postgres automatically skips Job 1 and returns Job 2.

### Result

```
✔ Job not processed twice
✔ No waiting
✔ No error
✔ Workers stay busy
```

---

### Why SKIP LOCKED is preferred for job queues

Imagine:

```
100 pending jobs
20 workers
```

With FOR UPDATE:

```
19 workers waiting
```

With NOWAIT:

```
19 workers getting errors and retrying
```

With SKIP LOCKED:

```
20 workers process 20 different jobs
```

Much more efficient.

---

### Your second question

> or select for update wait then remaining transaction can still be executed?
> 

Yes.

Only the transaction trying to access the locked row waits.

If asked:

> Can FOR UPDATE and NOWAIT also prevent duplicate job processing?
> 

Answer:

```
Yes. FOR UPDATE, NOWAIT, and SKIP LOCKED all prevent duplicate processing. The difference is that FOR UPDATE waits for locks, NOWAIT fails immediately if a row is locked, and SKIP LOCKED skips locked rows and continues processing other available work, making it the preferred choice for multi-worker job queues.
```

### Note -

When I said:

```
Job1
Job2
Job3
```

I was referring to **rows in a jobs table**, not SQL queries.

For example:

```
jobs

id| status
---+---------
1| PENDING
2| PENDING
3| PENDING
```

Each row represents a background task (a "job").

Examples:

```
Send email
Generate report
Process refund
Sync Uber receipt
Export CSV
```

---

`SKIP LOCKED` is indeed written in the SQL query:

```
SELECT*
FROM jobs
WHERE status='PENDING'
FORUPDATE SKIP LOCKED
LIMIT1;
```

But the thing being skipped is:

```
locked ROWS
```

not the SQL query itself.

### Question -

If a row is locked by an unrelated process and the email scheduler uses `SKIP LOCKED`, will the email job be skipped for that run? How can we ensure the email is eventually sent and not missed permanently?

In that case, **yes, the email can be skipped for that run.**

Example:

Suppose table:

```
id=1
status=PENDING
send_email=true
```

Worker A:

```
SELECT*FROM jobs
WHERE id=1
FORUPDATE;
```

But Worker A is doing:

```
Refund calculation
```

not email sending.

---

At the same time, Email Scheduler runs:

```
SELECT*
FROM jobs
WHERE send_email=true
FORUPDATE SKIP LOCKED;
```

Row is locked by Worker A.

So:

```
Email Scheduler skips row
```

No email sent during that execution.

---

# Is that a problem?

Usually no.

Because schedulers run repeatedly:

```
Every 1 minute
Every 5 minutes
Every 15 minutes
```

Next run:

```
Worker A committed
Lock released
```

Email Scheduler sees row again:

```
send email
```

So:

```
Delayed
≠
Lost
```

---

# When can it become a problem?

If:

```
Scheduler runs only once
```

or

```
Application assumes skipped rows will never need reprocessing
```

Then yes, work can be missed.

That's why good systems track state:

```
EMAIL_PENDING
EMAIL_SENT
EMAIL_FAILED
```

and keep rechecking pending items.

---

# Real-world design lesson

If two unrelated processes need the same row:

```
Refund processor
Email sender
```

then locking the entire row may be too broad.

A better design might be:

```
refund_status
email_status
```

handled independently.

Or even separate tables:

```
refund_jobs
email_jobs
```

so email processing is not blocked by refund processing.

---

This is actually one of the tradeoffs of `SKIP LOCKED`:

```
It prevents waiting,
but it can delay processing of locked rows until a later run.
```

And that's usually acceptable for background jobs like emails, reports, notifications, etc. where a small delay is better than workers sitting idle.
