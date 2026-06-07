This is one of the most important production topics because **read replicas alone do not give High Availability (HA)**.

---

# Question: What happens when the Master database crashes while replicas are still running?

Suppose we have:

```
           Master
              |
      ----------------
      |              |
   Replica1      Replica2
```

Normal operation:

```
Writes -> Master
Reads  -> Replicas
```

Now Master crashes:

```
           X Master (DOWN)
              |
      ----------------
      |              |
   Replica1      Replica2
```

What happens?

### Reads

Still work.

Replicas still contain data.

### Writes

Fail.

Because:

```
INSERT
UPDATE
DELETE
```

must go to Master.

Result:

```
Application becomes partially unavailable.
```

---

# Question: How do companies solve this?

Using:

```
Failover
```

Failover means:

```
Master dies
↓
One replica becomes new Master
↓
Application switches automatically
```

---

# Question: What is Replica Promotion?

Suppose:

```
Master
 |
 |---- Replica1
 |
 |---- Replica2
```

Master crashes.

System promotes:

```
Replica1
```

to become:

```
New Master
```

Now:

```
           Master (Replica1)
                |
         ----------------
         |
      Replica2
```

Writes start working again.

This process is called:

```
Replica Promotion
```

---

# Question: How does the system know the Master is dead?

A monitoring system continuously checks health.

Example:

```
Every few seconds
```

check:

```
Can I connect?
Can I execute query?
```

If not:

```
Master = unhealthy
```

Failover starts.

---

# Question: What is Leader Election?

Suppose:

```
Replica1
Replica2
Replica3
```

Master died.

Who becomes Master?

Need a decision.

This is called:

```
Leader Election
```

System chooses one replica.

Example criteria:

```
Most up-to-date replica
Lowest replication lag
Highest priority replica
```

---

# Question: Why can't all replicas become Master?

Imagine:

```
Replica1 becomes Master
Replica2 becomes Master
```

Now:

```
Write A -> Replica1
Write B -> Replica2
```

Data becomes inconsistent.

This is called:

```
Split Brain
```

Very dangerous.

Production systems must prevent this.

---

# Question: What is Split Brain?

Example:

Network issue.

Replica1 thinks:

```
Master is dead
```

Replica2 thinks:

```
Master is dead
```

Both become Master.

Now:

```
Two Masters
```

Data diverges.

Eventually:

```
User A data exists in DB1
User B data exists in DB2
```

Huge mess.

---

# Question: How do production systems prevent Split Brain?

Using:

```
Consensus Systems
```

Examples:

- ZooKeeper
- etcd
- Consul

These coordinate leader election safely.

---

# Question: How is failover done in PostgreSQL?

Several common solutions.

---

## Option 1: Manual Failover

Admin notices:

```
Master down
```

Runs commands:

```
Promote Replica1
```

Updates application config.

Slow.

Usually emergency only.

---

## Option 2: Automatic Failover

Much more common.

Tools:

```
Patroni
repmgr
pg_auto_failover
```

They monitor:

```
Master health
```

and automatically:

```
Promote replica
```

---

# Question: What does AWS RDS do?

AWS handles most of this automatically.

Example:

```
RDS Primary
 |
Standby
```

Primary crashes.

AWS:

```
Promotes Standby
Updates DNS
Reconnects clients
```

Often within:

```
30-120 seconds
```

---

# Question: What is Multi-AZ?

AWS terminology.

Instead of:

```
One database
```

you get:

```
Availability Zone A
      |
    Primary

Availability Zone B
      |
    Standby
```

If one AZ dies:

```
Standby promoted
```

Very common production setup.

---

# Question: Is a Standby the same as a Read Replica?

No.

Many developers confuse them.

### Read Replica

Purpose:

```
Scale Reads
```

Example:

```
Master
 |
Replica1
Replica2
```

Used for:

```
SELECT
```

---

### Standby

Purpose:

```
High Availability
```

Usually not used for reads.

Its only job:

```
Take over if Master dies
```

---

# Question: What is RPO?

Recovery Point Objective.

Question:

```
How much data can we lose?
```

Example:

```
5 seconds of data loss
```

means:

```
RPO = 5 seconds
```

---

# Question: What is RTO?

Recovery Time Objective.

Question:

```
How long can system stay down?
```

Example:

```
60 seconds
```

means:

```
RTO = 60 seconds
```

---

# Question: What does an enterprise database setup look like?

Common architecture:

```
               Load Balancer
                     |
                API Servers
                     |
                PostgreSQL
                     |
         -----------------------
         |                     |
      Primary              Standby
         |
   ----------------
   |              |
Read1         Read2
```

Purpose:

```
Primary -> Writes
Read Replicas -> Reads
Standby -> Failover
```

---

# Question: What happens to application code during failover?

Ideally:

```
Nothing
```

Applications connect using:

```
db.company.com
```

not:

```
10.1.1.10
```

During failover:

```
DNS / Proxy
```

is updated.

Application reconnects.

No code change needed.

---

# Question: In your Node.js + PostgreSQL world, what is most realistic?

Small/medium systems:

```
PostgreSQL
```

Large systems:

```
PostgreSQL
+
Read Replicas
```

Enterprise systems:

```
PostgreSQL
+
Read Replicas
+
Automatic Failover
+
Standby Database
```

---

# Notes Summary

At this point you've covered the major **Database Scaling & Availability** path:

```
1. Read Replicas
2. Replication Lag
3. Read-Your-Own-Write
4. Failover
5. Replica Promotion
6. Leader Election
7. High Availability
8. RPO/RTO
```

The next natural topic in system design is usually **Message Queues (RabbitMQ, Kafka, SQS) and Event-Driven Architecture**, because that's how large systems avoid making users wait for slow work like emails, notifications, payment reconciliation, report generation, and webhook processing.
