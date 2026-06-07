Once you've learned **API Gateway → Load Balancer → Cache**, the next bottleneck is almost always the **Database**.

# 1. Why Do We Need Database Scaling?

Imagine your app starts small:

```
Users
  |
API
  |
PostgreSQL
```

Works fine.

But traffic grows:

```
1,000 users
10,000 users
100,000 users
1 million users
```

Problems start appearing:

- Slow queries
- High CPU usage
- High memory usage
- Connection limits reached
- Database becomes bottleneck

Unlike API servers, databases are harder to scale.

---

# 2. Vertical Scaling (Scale Up)

Simplest solution.

Before:

```
DB Server
4 CPU
8 GB RAM
```

Upgrade to:

```
DB Server
32 CPU
128 GB RAM
```

Benefits:

- Easy
- No application changes

Problems:

- Expensive
- Hardware limit exists

Eventually you can't keep upgrading forever.

---

# 3. Horizontal Scaling (Scale Out)

Instead of:

```
1 database
```

Use:

```
DB1
DB2
DB3
```

This is harder but much more scalable.

---

# 4. Read vs Write Problem

Most systems have far more reads than writes.

Example:

```
Read user profile
Read trip history
Read station list
Read permit details
```

versus

```
Create user
Book ride
Make payment
```

Typical ratio:

```
90% Reads
10% Writes
```

---

# 5. Read Replicas

Most common scaling approach.

Architecture:

```
                Master
                  |
        --------------------
        |        |         |
      Replica1 Replica2 Replica3
```

---

### Writes

Always go to master:

```
INSERT
UPDATE
DELETE
```

---

### Reads

Can go to replicas:

```
SELECT
```

---

Flow:

```
User
 |
API
 |
Read Query
 |
Replica
```

```
User
 |
API
 |
Write Query
 |
Master
```

---

# 6. Why Read Replicas Help

Without replicas:

```
Master
100,000 reads
10,000 writes
```

Total:

```
110,000 operations
```

With replicas:

```
Master
10,000 writes

Replica1
33,000 reads

Replica2
33,000 reads

Replica3
34,000 reads
```

Huge reduction.

---

# 7. Replication Lag

Important interview topic.

Suppose:

```
UPDATE users
SET name='Lucky'
```

Written to master.

Immediately after:

```
SELECT*FROM users
```

from replica.

Replica may still contain:

```
Old value
```

because replication isn't instantaneous.

Called:

```
Replication Lag
```

---

# 8. Real Example

User changes profile name.

```
Write -> Master
```

Immediately refreshes page.

```
Read -> Replica
```

Might see old name.

A few seconds later:

```
Replica updated
```

New name appears.

---

# 9. How Big Companies Handle This

Sometimes:

```
Recent write?
Read from Master
```

Otherwise:

```
Read from Replica
```

Called:

```
Read Your Own Write
```

---

# 10. Database Partitioning

Suppose one table becomes huge.

Users table:

```
500 million users
```

Single server struggles.

Split data.

---

Example:

```
User IDs

1-100M     -> DB1
100M-200M  -> DB2
200M-300M  -> DB3
```

Called:

```
Partitioning
```

---

# 11. Sharding

Most important scaling concept.

Instead of:

```
Users Table
```

on one database.

Split into shards.

```
Shard1
Shard2
Shard3
```

---

Example:

```
userId % 3

0 -> Shard1
1 -> Shard2
2 -> Shard3
```

---

User:

```
ID = 11

11 % 3 = 2

Shard3
```

---

User:

```
ID = 25

25 % 3 = 1

Shard2
```

---

# 12. Why Sharding Helps

Without sharding:

```
1 server
500M rows
```

With sharding:

```
5 servers
100M rows each
```

Much faster.

---

# 13. Sharding Challenges

Big interview topic.

Suppose:

```
User data -> Shard1
Trip data -> Shard2
```

Need:

```
JOIN users and trips
```

Now difficult.

Cross-shard queries become expensive.

---

# 14. Partitioning vs Sharding

People often confuse these.

### Partitioning

Same database.

```
Postgres
  |
---------------
|     |       |
P1    P2      P3
```

Managed internally.

---

### Sharding

Different databases.

```
DB1
DB2
DB3
```

Application must know where data lives.

---

# 15. Connection Pooling

Very important.

Bad:

Every request creates DB connection.

```
Request
 |
Connect DB
 |
Query
 |
Disconnect
```

Expensive.

---

Use pool:

```
API
 |
Connection Pool
 |
Database
```

Connections reused.

Node.js ORMs like Sequelize already do this.

---

# 16. Database Indexing

Most common performance optimization.

Without index:

```
SELECT*
FROM users
WHERE email='abc@test.com'
```

Database scans entire table.

```
1 million rows
```

---

Add index:

```
CREATE INDEX idx_email
ON users(email);
```

Now lookup is much faster.

---

# 17. Why Too Many Indexes Are Bad

Every write must update indexes.

Example:

```
INSERTuser
```

Need to update:

```
Table
Index1
Index2
Index3
```

Writes become slower.

---

# 18. Common Database Scaling Architecture

Typical production setup:

```
Users
  |
Load Balancer
  |
API Servers
  |
Redis
  |
Master DB
  |
-------------
|     |     |
R1    R2    R3
```

---

Flow:

```
Writes -> Master
Reads -> Replicas
Cache -> Redis
```

Very common.

---

# 19. When Sharding Appears

Usually after:

```
API Gateway
Load Balancer
Cache
Read Replicas
```

still not enough.

Then:

```
Sharding
```

Large companies:

- Google
- Uber
- Amazon
- Netflix

all use sharding extensively.

---

# 20. How This Relates To Your Work

In your Node.js + PostgreSQL projects:

Currently you likely use:

```
Application
 |
PostgreSQL
```

As system grows, common evolution:

```
Application
 |
Redis
 |
Master DB
 |
Read Replicas
```

Most companies never reach sharding.

Many systems handle millions of users using:

- Good indexes
- Connection pooling
- Read replicas
- Caching

without sharding.

---

# Common Interview Questions

### Why use Read Replicas?

Scale reads.

---

### Why writes go to Master?

To avoid data inconsistency.

---

### What is Replication Lag?

Replica temporarily behind master.

---

### What is Sharding?

Split data across multiple databases.

---

### What is Partitioning?

Split data inside same database.

---

### What is Connection Pooling?

Reuse database connections.

---

### What is an Index?

Data structure that speeds lookups.

---

# Question: What techniques are used for updating Read Replicas?

There are mainly two replication methods:

## 1. Asynchronous Replication (Most Common)

Flow:

```
Write -> Master
       -> Success returned to user

Later
Master -> Replica
```

Example:

```
12:00:00 Write completed on Master
12:00:01 Replica updated
```

Pros:

- Fast writes
- Most scalable
- Used by most production systems

Cons:

- Replication lag exists

---

## 2. Synchronous Replication

Flow:

```
Write -> Master
      -> Replica
      -> Success returned
```

Master waits for replica confirmation.

Pros:

- No replication lag
- Strong consistency

Cons:

- Slower writes
- If replica is slow, write is slow

---

## 3. Semi-Synchronous Replication

Middle ground.

```
Write -> Master
      -> At least one replica confirms
      -> Success returned
```

Used by many enterprise systems.

---

# Question: How much replication lag is expected?

Depends on workload.

Typical values:

```
Normal system:
10 ms - 500 ms

Busy system:
1 - 5 seconds

Very overloaded:
10+ seconds
```

In a healthy PostgreSQL setup:

```
Usually under 1 second
```

---

# Question: Is there any way to have zero lag?

Practically:

```
No
```

unless using synchronous replication.

But then:

```
Write performance decreases
```

This is a classic tradeoff:

```
Consistency vs Performance
```

---

# Question: How do companies handle users reading stale data after an update?

This is called:

```
Read Your Own Write
```

Example:

```
User updates profile name
Immediately refreshes page
```

You want:

```
Updated value
```

not the old value from replica.

---

# Question: How is "Read Your Own Write" implemented in code?

One common approach:

After update:

```
awaitUser.update(
  { name:"Lucky" },
  { where: { id:userId } }
);
```

For the next few seconds:

```
constuser=awaitUser.findByPk(
userId,
  {
    useMaster:true
  }
);
```

Sequelize actually supports:

```
useMaster:true
```

which forces query to master.

---

Example:

```
awaitUser.update(...);

constfreshUser=awaitUser.findByPk(
userId,
  {
    useMaster:true
  }
);
```

Now you're guaranteed to see latest data.

---

# Question: How does the system know whether to read from Master or Replica?

Several approaches exist.

---

## Approach 1: Read Master Immediately After Write

Example:

```
Update Profile
↓
Redirect to Profile Page
↓
Read from Master
```

Only that read goes to Master.

All others:

```
Replica
```

---

## Approach 2: Store Recent Write Timestamp

Store:

```
req.session.lastWriteTime
```

or

```
redis.set(
`last-write:${userId}`,
Date.now()
);
```

Then:

```
if (Date.now()-lastWriteTime<5000) {
readMaster();
}else {
readReplica();
}
```

Meaning:

```
For 5 seconds after write:
Read Master

After 5 seconds:
Read Replica
```

Very common.

---

# Question: Will this slow down response time?

A little.

Example:

```
Master Query = 20ms
Replica Query = 15ms
```

Difference:

```
5ms
```

Usually negligible.

More importantly:

```
Correctness > Speed
```

for recently updated records.

Users expect to see their changes immediately.

---

# Question: Does every request need this Master-vs-Replica logic?

No.

Only for data that was recently modified.

Example:

### Read from Replica

```
Station List
Trip History
Reports
Configurations
```

These can tolerate slight lag.

---

### Read from Master

```
Profile just updated
Password changed
Payment status changed
Order just placed
```

Need fresh data.

---

# Question: How is this configured in Sequelize?

Example:

```
constsequelize=newSequelize({
  dialect:'postgres',

  replication: {
    read: [
      {
        host:'replica1'
      },
      {
        host:'replica2'
      }
    ],

    write: {
      host:'master'
    }
  }
});
```

Now Sequelize automatically:

```
SELECT -> Replica

INSERT -> Master
UPDATE -> Master
DELETE -> Master
```

And when needed:

```
User.findByPk(id, {
  useMaster:true
});
```

forces master.

# Question: Can we set up read replicas so they automatically update whenever the master is updated?

Yes, absolutely.

In fact, **that's how read replicas normally work**.

You do **not** write code like:

```
updateMaster();
updateReplica1();
updateReplica2();
```

That would be a bad design.

Instead:

```
Application
     |
     v
 Master DB
     |
     v
Replication Process
     |
----------------------
|         |          |
Replica1 Replica2 Replica3
```

When data changes on the Master:

```
UPDATE users
SET name='Lucky'
WHERE id=123;
```

PostgreSQL automatically records the change in its transaction log (WAL - Write Ahead Log).

Replicas continuously read that log and apply the same changes.

So:

```
Master Updated
      ↓
Replica1 Updated
Replica2 Updated
Replica3 Updated
```

No application code required.

---

# Question: Do cloud platforms provide this automatically?

Yes.

Most cloud providers make this very easy.

### AWS RDS PostgreSQL

You create:

```
Primary DB
```

Then click:

```
Create Read Replica
```

AWS automatically handles:

- Replication setup
- Data synchronization
- Monitoring
- Failover options

---

### AWS Aurora

Even better.

```
Writer Node
   |
------------------
|       |        |
Reader1 Reader2 Reader3
```

Aurora manages replication internally.

---

### Google Cloud SQL

Supports:

```
Primary
  |
Read Replica
```

with minimal configuration.

---

### Azure Database for PostgreSQL

Also supports:

```
Primary
  |
Read Replicas
```

out of the box.

---

# Question: Then why is there replication lag if updates are automatic?

Because:

```
Master Write
```

and

```
Replica Update
```

are still two separate operations.

Example:

```
12:00:00.000
Write arrives at Master

12:00:00.005
Master commits transaction

12:00:00.020
Replica receives WAL record

12:00:00.030
Replica applies change
```

There is always some tiny delay.

Usually:

```
milliseconds
```

but under heavy load it can become:

```
seconds
```

---

# Question: In my Node.js application, do I need to write replica synchronization code?

No.

Your application only needs to know:

```
Read -> Replica
Write -> Master
```

Database replication itself is handled by:

- PostgreSQL
- MySQL
- Aurora
- RDS
- Cloud SQL

depending on your setup.

---

# Question: Then what code changes are required in the application?

Mostly just connection configuration.

Example Sequelize:

```
constsequelize=newSequelize({
  replication: {
    write: {
      host:'master-db'
    },
    read: [
      { host:'replica1' },
      { host:'replica2' }
    ]
  }
});
```

Then Sequelize routes:

```
SELECT -> Replica
INSERT -> Master
UPDATE -> Master
DELETE -> Master
```

automatically.

# Question: If only one user's record is updated, will all users' reads go to Master?

No.

That's the key point many people miss.

A production system would **not** do:

```
User A updated profile
↓
All reads from Master for next 5 seconds
```

That would defeat the purpose of replicas.

Instead:

```
User A updated profile
↓
Only User A's relevant reads go to Master
```

Other users continue reading from replicas.

---

# Question: What exactly is being tracked? User or Table?

Usually one of these:

```
User ID
Session ID
Request Context
Business Entity
```

Not the entire database.

Example:

```
User A updates profile
```

Store:

```
lastWrite:userA = current_timestamp
```

But:

```
User B
User C
User D
```

have no recent writes.

Their reads still go to replicas.

---

# Question: How is this implemented in production?

## Approach 1: Read Master Immediately After Write (Most Common)

Example:

```
PUT /profile
```

updates:

```
User 123
```

Immediately after:

```
GET /profile
```

Application knows:

```
This request follows a write
```

So:

```
awaitUser.findByPk(id, {
  useMaster:true
});
```

Only that request.

---

Other requests:

```
awaitUser.findByPk(id);
```

go to replicas.

---

# Question: Is useMaster:true dynamically added?

Yes.

Very commonly.

Example:

```
constoptions= {};

if (mustReadFreshData) {
options.useMaster=true;
}

constuser=awaitUser.findByPk(
userId,
options
);
```

or

```
constuser=awaitUser.findByPk(
userId,
  {
    useMaster:recentlyUpdated
  }
);
```

Very normal production code.

---

# Question: How does the application know recentlyUpdated?

Several ways.

---

## Option 1: Request Context

Example:

```
POST /profile/update
```

returns:

```
Profile updated
```

Immediately after:

```
GET /profile
```

Application knows:

```
This user just updated profile
```

Use Master.

---

## Option 2: Redis Timestamp

Store:

```
awaitredis.set(
`last-write:${userId}`,
Date.now(),
"EX",
5
);
```

TTL:

```
5 seconds
```

---

Read:

```
constlastWrite=
awaitredis.get(
`last-write:${userId}`
  );

constrecentlyUpdated=
!!lastWrite;
```

If true:

```
useMaster:true
```

Otherwise:

```
useMaster:false
```

---

# Question: Is this done per user or globally?

Per user.

Example:

```
last-write:123
last-write:456
last-write:789
```

Not:

```
databaseRecentlyUpdated=true
```

That would be terrible.

---

# Question: What if a user updates one record but reads another table?

Depends on business rules.

Example:

```
User updated profile
```

Then reading:

```
Profile
```

Read Master.

---

Reading:

```
Station List
Permit Types
Configurations
```

Can still read Replica.

Because those tables were not affected.

---

# Question: What do large companies actually do?

Usually they classify data.

---

## Eventually Consistent Data

Can tolerate lag.

Examples:

```
Reports
Analytics
Trip History
Station List
Configurations
```

Always Replica.

---

## Strongly Consistent Data

Must be fresh.

Examples:

```
Profile Updates
Account Status
Payment Status
Order Status
Inventory
```

Read Master after write.

---

# Question: Does every company use timestamp tracking?

No.

Many systems simply use business logic.

Example:

```
Update Profile
↓
Return Updated Profile
```

Instead of:

```
Update Profile
↓
Read Replica
```

They skip the second query entirely.

---

# Question: What is the most common enterprise approach?

Usually:

### Normal Reads

```
Replica
```

### Critical Reads

```
Master
```

### Immediately After Write

```
Master
```

### Everything Else

```
Replica
```

Simple and effective.

---

# Question: In my Node.js + Sequelize project, what would I likely do?

Something like:

```
constforceMaster=
req.forceFreshData===true;

constuser=
awaitUser.findByPk(
userId,
    {
      useMaster:forceMaster
    }
  );
```

or

```
constrecentlyUpdated=
awaitredis.exists(
`last-write:${userId}`
  );

constuser=
awaitUser.findByPk(
userId,
    {
      useMaster:recentlyUpdated
    }
  );
```

Only that user's reads get routed to Master.

Everyone else still uses replicas.

The next question often discussed is:

**"What happens when the Master database crashes while replicas are still running?"**

That leads into **failover, primary promotion, leader election, and high availability**, which is the next step after understanding read replicas.
