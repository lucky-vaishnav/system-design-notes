```
➡️ Distributed Scheduling
⬜ Observability
⬜ Monitoring & Alerting
```

Since you already work with schedulers, we’ll go beyond basic cron jobs and focus on **senior-level distributed scheduling**: multiple instances, duplicate execution, leader election, retries, locks, failures, and production design.

# 1. What Is Distributed Scheduling?

A normal scheduler is simple:

```
Server
  ↓
Cron
  ↓
Run job
```

Example:

```
0 * * * * runPaymentReconciliation()
```

But in production, your service may have multiple replicas:

```
                 Load Balancer
                      |
          +-----------+-----------+
          ↓           ↓           ↓
       Server 1    Server 2    Server 3
          ↓           ↓           ↓
       Scheduler   Scheduler   Scheduler
```

Now the problem is:

> **Who should execute the scheduled job?**
> 

If all three execute it:

```
Server 1 → Job ✅
Server 2 → Job ✅
Server 3 → Job ✅
```

you may process the same task three times.

That's the core problem distributed scheduling solves.

---

# 2. The Main Problem: Duplicate Execution

Suppose you have:

```
Daily refund reconciliation
```

and 3 application replicas.

At 2:00 AM:

```
Server A → starts job
Server B → starts job
Server C → starts job
```

Potential consequences:

- Duplicate payments
- Duplicate emails
- Duplicate database updates
- Duplicate API calls
- Race conditions

Therefore, we need some form of **coordination**.

---

# 3. First Important Question

There are actually **two different distributed scheduling problems**.

### Problem A — Only one instance should run the job

```
Server A
Server B
Server C

        ↓

Only ONE executes
```

This is usually solved using:

- Leader election
- Distributed lock
- Scheduler with centralized coordination

### Problem B — Multiple workers can process tasks

```
Server A → Task 1
Server B → Task 2
Server C → Task 3
```

This is more like:

> **Distributed work distribution**
> 

Usually solved using:

- Queue
- Database task table
- Kafka
- SQS
- Redis Streams, etc.

This distinction is **very important for senior interviews**.

---

# 4. Approach 1 — Single Scheduler Instance

The simplest solution:

```
Scheduler
   |
   ↓
Only one server
```

For example:

```
EC2 Scheduler Server
        |
        ↓
Application servers
```

Only the dedicated scheduler executes cron jobs.

### Advantage

Very simple.

### Problem

Scheduler becomes a potential single point of failure.

If:

```
Scheduler ❌
```

jobs stop running.

---

# 5. Approach 2 — Leader Election

This connects directly to our previous topic.

Suppose:

```
Server A
Server B
Server C
```

They perform leader election.

```
Server B = Leader
```

Only B runs scheduled jobs.

```
A → standby
B → scheduler ✅
C → standby
```

If B dies:

```
B ❌
 ↓
Leader election
 ↓
A becomes leader
 ↓
Scheduler continues
```

This provides **high availability**.

---

# 6. etcd / ZooKeeper Example

We just learned these.

You can use:

```
Application replicas
        ↓
      etcd
        ↓
Leader election
```

Example:

```
/payment-reconciliation/leader
```

Server A acquires leadership.

```
A → leader
B → follower
C → follower
```

A periodically renews its lease.

If A dies:

```
Lease expires
      ↓
B/C compete
      ↓
New leader
```

---

# 7. Approach 3 — Distributed Lock

Instead of having a permanent leader, servers can compete for a lock whenever the job needs to run.

At 2 AM:

```
A ──┐
B ──┼──→ distributed lock
C ──┘
```

Suppose B gets it:

```
B → LOCK ✅
A → ❌
C → ❌
```

Only B runs the job.

After completion:

```
B → RELEASE LOCK
```

This is often called:

> **Lock-based scheduling**
> 

---

# 8. Leader Election vs Distributed Lock

Very important distinction.

### Leader election

The leader stays responsible for scheduling.

```
A
B ← Leader
C
```

B may run many scheduled jobs.

### Distributed lock

Leadership is acquired for a specific operation.

```
Job 1 → B gets lock
Job 2 → A gets lock
Job 3 → C gets lock
```

So:

```
Leader Election
→ long-lived ownership

Distributed Lock
→ temporary ownership
```

---

# 9. Approach 4 — Database-Based Lock

You can also use PostgreSQL.

For example, PostgreSQL provides advisory locks:

```sql
SELECT pg_try_advisory_lock(12345);
```

If it returns true:

```
This instance owns the lock.
```

Other instances:

```
false
```

and skip the job.

This can be a very practical solution when your application already depends heavily on PostgreSQL.

---

# 10. Node.js Example

Imagine:

```jsx
cron.schedule('0 2 * * *', async () => {
  // reconciliation
});
```

With 3 replicas, this executes on all three.

Instead:

```jsx
cron.schedule('0 2 * * *', async () => {
  const acquired = await acquireDistributedLock('refund-reconciliation');

  if (!acquired) {
    return;
  }

  try {
    await runRefundReconciliation();
  } finally {
    await releaseDistributedLock('refund-reconciliation');
  }
});
```

The important part isn't the Node.js code.

It's:

```
acquireDistributedLock()
```

The lock must be **shared between replicas**.

A local variable won't work.

---

# 11. Why a Local Lock Doesn't Work

This is a common mistake.

```jsx
let running = false;
```

Server A:

```
running = true
```

But Server B has its own memory:

```
running = false
```

because:

```
Server A memory ≠ Server B memory
```

Therefore:

```
❌ in-memory lock
```

doesn't provide distributed coordination.

You need something shared:

```
PostgreSQL
Redis
etcd
ZooKeeper
```

depending on the consistency and failure requirements.

---

# 12. The Hard Problem: What If the Scheduler Crashes?

Suppose:

```
Server A → acquired lock
```

Then:

```
Server A ❌
```

If the lock never expires:

```
Lock remains forever
```

Now nobody can run the job.

Therefore, distributed locks commonly need:

> **Lease / TTL**
> 

Example:

```
Lock
TTL = 30 seconds
```

The owner periodically renews it.

If it dies:

```
No renewal
    ↓
TTL expires
    ↓
Lock released
```

This is exactly where our previous **etcd lease** discussion becomes useful.

---

# 13. But TTL Creates Another Problem

Suppose:

```
Lock TTL = 30 sec
```

Job takes:

```
45 sec
```

At 30 seconds:

```
Lock expires
```

Server B acquires it.

Now:

```
Server A → still running ❗
Server B → also running ❗
```

Duplicate execution again.

Therefore, you need either:

### Lease renewal

```
A
 ↓
renew
 ↓
renew
 ↓
renew
```

or

### Fencing tokens

This is a more advanced senior-level technique.

---

# 14. Fencing Tokens

Imagine:

```
A gets token 10
```

Later:

```
A becomes slow
```

Lock expires.

B gets:

```
token 11
```

Now A wakes up and tries to modify the resource.

The resource says:

```
A's token = 10
B's token = 11
```

Since:

```
10 < 11
```

A's operation is rejected.

This prevents an old/expired worker from continuing to make changes.

This is a **very important distributed-systems concept**.

---

# 15. Scheduled Job vs Task Queue

Another major distinction.

Suppose at 2 AM you have:

```
10,000 payments to process
```

Don't necessarily do:

```
Scheduler
   ↓
process 10,000 sequentially
```

Instead:

```
Scheduler
   ↓
Queue
   ↓
+-------+-------+-------+
↓       ↓       ↓
Worker Worker Worker
```

Scheduler's responsibility becomes:

> **Create/enqueue the work.**
> 

Workers handle the actual processing.

This gives you:

- Parallelism
- Retry
- Backpressure
- Scaling
- Failure isolation

---

# 16. Scheduling vs Queueing

Remember this:

```
Scheduling
→ WHEN should work happen?

Queueing
→ WHO should process the work?
```

Example:

```
2:00 AM
   ↓
Scheduler
   ↓
Create 10,000 tasks
   ↓
SQS/Kafka
   ↓
Workers
```

This is usually much more scalable.

---

# 17. Retry Strategy

Distributed jobs fail.

Suppose:

```
Task
 ↓
API call
 ↓
500 error
```

Don't immediately retry forever.

Use:

```
Retry
 ↓
Exponential Backoff
 ↓
Retry
 ↓
Retry
 ↓
Dead Letter Queue
```

Example:

```
1 sec
2 sec
4 sec
8 sec
16 sec
```

with jitter.

---

# 18. Why Jitter?

Imagine 10,000 tasks fail at exactly 2:00:00.

If every worker retries after exactly 10 seconds:

```
2:00:10
    ↓
10,000 requests
```

You create another traffic spike.

With jitter:

```
9.3 sec
10.7 sec
11.2 sec
8.9 sec
...
```

the retries spread out.

This is called:

> **Retry jitter**
> 

---

# 19. Idempotency Is Essential

Even with excellent scheduling, duplicate execution can happen.

Therefore:

```
Scheduler
   +
Queue
   +
Retries
```

should generally be combined with:

```
Idempotent job processing
```

Example:

```
payment_id = 123
```

Before processing:

```
Has payment 123 already been processed?
```

or use a unique constraint:

```sql
UNIQUE(payment_id)
```

Then:

```
First execution → success
Duplicate execution → safely ignored
```

This is one of the biggest senior-level lessons:

> **Don't design distributed scheduling assuming exactly-once execution. Design the job so duplicate execution is safe.**
> 

---

# 20. Exactly Once vs At Least Once

In distributed systems, this is important.

### At-most-once

```
Run 0 or 1 times
```

May lose work.

### At-least-once

```
Run 1+ times
```

May duplicate work.

### Exactly-once

```
Run exactly once
```

Very difficult to guarantee end-to-end across distributed systems.

In practical systems, a common design is:

```
At-least-once delivery
        +
Idempotent processing
```

This gives you effectively-once business behavior in many cases.

---

# 21. What Happens If Scheduler Runs Late?

Suppose:

```
Job scheduled: 2:00 AM
Server was down until: 2:10 AM
```

What should happen?

There are different policies.

### Skip missed execution

```
2:00 missed
→ don't run
```

### Run immediately

```
Server starts 2:10
→ run missed job
```

### Catch up

If the job runs hourly:

```
1:00 ❌
2:00 ❌
3:00 current
```

You may need to process missed periods.

This is called **misfire handling / catch-up behavior**.

The correct choice depends on the business requirement.

---

# 22. Scheduler Time Zones

This becomes important in production.

Never casually assume:

```
2:00 AM
```

means the same thing everywhere.

Consider:

```
UTC
America/Los_Angeles
Asia/Kolkata
```

A robust scheduler should define:

```
timezone = UTC
```

or explicitly specify the business timezone.

Also be careful with:

- Daylight saving time
- Date boundaries
- DST transitions
- "Every day" semantics

---

# 23. Job State

A mature scheduler should track state.

For example:

```
job_runs

id
job_name
scheduled_at
started_at
completed_at
status
attempt
worker_id
error
```

Possible states:

```
scheduled
running
completed
failed
retrying
dead
```

This gives you observability and recovery.

---

# 24. What a Production Architecture Can Look Like

For your type of backend environment:

```
                  Scheduler
                     |
                     ↓
                Create Tasks
                     |
                     ↓
                   SQS
                     |
          +----------+----------+
          ↓          ↓          ↓
       Worker 1   Worker 2   Worker 3
          |          |          |
          +----------+----------+
                     |
                     ↓
                 PostgreSQL
```

And:

```
Scheduler
    |
    ↓
Distributed Lock / Leader Election
```

to make sure only one scheduler creates the tasks.

Then:

```
SQS
 ↓
Workers
```

handles parallel processing.

This is usually much better than having every application replica independently execute the entire scheduled workload.

---

# 25. How This Connects to Your Existing Knowledge

You've now seen almost every building block:

```
Leader Election
      ↓
etcd / ZooKeeper
      ↓
Distributed Scheduling
      ↓
Distributed Lock
      ↓
Lease / TTL
      ↓
Fencing Token
      ↓
Queue
      ↓
Retry + Backoff + Jitter
      ↓
Idempotency
```

That's the real senior-level picture.

---

# ⭐ Interview Questions

### Q1. How do you prevent multiple replicas from running the same cron job?

Use:

> Leader election or a distributed lock backed by a shared coordination system.
> 

---

### Q2. Why doesn't an in-memory lock work?

Because each replica has independent memory.

---

### Q3. What happens if the lock holder crashes?

Use a lease/TTL so the lock eventually expires and another instance can acquire it.

---

### Q4. Why isn't TTL alone enough?

The original worker may still be running after the TTL expires. Another worker could acquire the lock and execute concurrently.

Use lease renewal and, for stronger guarantees, **fencing tokens**.

---

### Q5. Should the scheduler process all tasks itself?

Usually no for large workloads.

Prefer:

```
Scheduler → Queue → Workers
```

---

### Q6. How do you handle duplicate execution?

Design jobs to be **idempotent** and use database constraints/idempotency keys where appropriate.

---

### Q7. What is the difference between scheduling and queueing?

> Scheduling determines **when work should happen**; queueing determines **how work is distributed to workers**.
> 

---

# 📌 Notes for Your Notebook

```
DISTRIBUTED SCHEDULING

Problem:
Multiple replicas may execute the same scheduled job.

Solutions:
1. Dedicated scheduler
2. Leader election
3. Distributed lock
4. Centralized scheduler

Leader Election:
One replica owns scheduling responsibility.

Distributed Lock:
Replica temporarily acquires ownership for a specific job.

Lock should generally use:
- Shared storage
- TTL / Lease
- Renewal
- Fencing tokens for stronger safety

Don't use:
❌ In-memory lock

Scheduling:
WHEN should work happen?

Queue:
WHO should process work?

Scalable architecture:

Scheduler
   ↓
Queue
   ↓
Workers
   ↓
Database/API

Reliability:
- Retry
- Exponential backoff
- Jitter
- Dead-letter queue
- Idempotency

Avoid assuming exactly-once execution.

Prefer:
At-least-once delivery
+
Idempotent processing

Also consider:
- Misfire handling
- Catch-up
- Time zones
- DST
- Job state
- Monitoring
```

### One sentence to remember

> **Distributed scheduling is not just about running cron jobs on multiple servers; it is about coordinating ownership, handling failures and duplicates, and reliably distributing scheduled work across a changing set of instances.**
> 

Next in Phase G is **Observability**, where we'll connect **logs + metrics + traces + correlation IDs + OpenTelemetry + distributed systems debugging**.

### Question —

> **For distributed scheduling, if we don't use a distributed lock directly, can we use ZooKeeper or etcd to coordinate schedulers? Is a distributed lock mandatory, or can ZooKeeper/etcd provide the coordination directly? When would we use ZooKeeper/etcd together with a distributed lock?**
> 

### Answer

Yes — **ZooKeeper/etcd can be used directly for scheduler coordination**. A distributed lock is not a separate mandatory component.

Think of it as:

```
ZooKeeper / etcd
       ↓
Coordination primitive
       ↓
Leader Election OR Distributed Lock
       ↓
Scheduler
```

For example, with **etcd**:

```
Scheduler A ─┐
Scheduler B ─┼──→ etcd ──→ Leader Election
Scheduler C ─┘
                       ↓
                  Scheduler B
                  runs jobs
```

Or you can use an etcd **lease/lock** for a specific job.

### So when do you use what?

- **ZooKeeper/etcd + Leader Election** → best when one scheduler should continuously own all scheduled jobs.
- **ZooKeeper/etcd + Distributed Lock** → best when different jobs need temporary ownership.
- **ZooKeeper/etcd alone** → technically yes, but you're really using its built-in coordination primitives such as election, leases, or locks.
- **ZooKeeper/etcd + another distributed-lock system** → usually unnecessary duplication unless there's a specific architectural reason.

The key point is:

> **ZooKeeper/etcd is the coordination system; leader election, leases, and distributed locks are mechanisms provided/implemented using that coordination system.**
> 

So you generally **choose the coordination mechanism you need**, rather than automatically using both.

### Question —

> **How do we actually deploy and use ZooKeeper or etcd in a production server environment? Do we install and configure ZooKeeper/etcd on servers, register the scheduler with it, and then let the scheduler handle the actual jobs?**
> 

### Short Answer

Yes, that's broadly correct.

```
Production servers
      ↓
Install / run ZooKeeper or etcd cluster
      ↓
Configure cluster + authentication/networking
      ↓
Scheduler instances connect to it
      ↓
Scheduler registers / participates in leader election
      ↓
ZooKeeper/etcd manages coordination
      ↓
Elected scheduler runs the jobs
```

You **don't put the actual scheduled-job logic inside ZooKeeper/etcd**. They only provide the coordination mechanism.

For production, you'd normally run **multiple ZooKeeper/etcd nodes** rather than one:

```
etcd-1
etcd-2  ← quorum
etcd-3
```

Your Node.js scheduler connects to the cluster using an appropriate client library and uses its **leader-election/lease/lock APIs**.

So the mental model is:

> **ZooKeeper/etcd = coordination infrastructure; Scheduler = business/job execution.**
>
