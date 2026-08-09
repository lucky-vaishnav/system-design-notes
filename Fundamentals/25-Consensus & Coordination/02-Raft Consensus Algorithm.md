```
✅ Distributed Consensus
➡️ Raft
⬜ Paxos (overview)
⬜ ZooKeeper / etcd Concepts
```
# Raft Consensus Algorithm — Deep Dive

Raft is a **consensus algorithm** that allows a group of servers to maintain the same replicated state, even when some servers fail.

The easiest mental model is:

```
                Client
                  |
                  v
               Leader
             /   |   \
            v    v    v
         Node B Node C Node D
```

The **leader** receives operations and replicates them to followers.

---

## 1. Why Does Raft Exist?

Imagine 3 database/state-machine servers:

```
A     B     C
```

A client sends:

```
SET balance = 100
```

We want all healthy servers to eventually agree on the same sequence:

```
1. SET balance = 100
2. SET balance = 150
3. SET balance = 120
```

If A crashes, B and C should still be able to determine who leads and what the correct state is.

That's what Raft provides.

---

# 2. Three States

Every Raft server is always in one of three states:

```
Follower
   |
   | timeout
   v
Candidate
   |
   | wins election
   v
Leader
```

### Follower

Normally does nothing except:

- Receive messages from leader
- Respond to requests
- Follow the leader

### Candidate

A follower becomes a candidate when it doesn't hear from a leader for a while.

It starts an election.

### Leader

The elected leader:

- Receives client requests
- Replicates log entries
- Sends heartbeats
- Coordinates the cluster

---

# 3. Leader Election

Suppose:

```
A = Leader
B = Follower
C = Follower
```

A crashes:

```
A ❌

B       C
```

B and C stop receiving heartbeats.

Eventually one of them reaches its **election timeout** first.

Suppose B:

```
B
↓
Candidate
```

B increments its term and votes for itself.

Then sends:

```
RequestVote → C
```

If C votes for B:

```
B = 1 vote
C = 1 vote
```

B has a majority:

```
2 / 3
```

Therefore:

```
B → Leader
```

---

# 4. What Is a Term?

A **term** is basically a logical election period/epoch.

Example:

```
Term 1
   ↓
Leader A
```

A crashes.

```
Term 2
   ↓
Leader B
```

Then:

```
Term 3
   ↓
Leader C
```

Every election moves the cluster to a newer term.

Servers reject stale information from older terms.

This is extremely important for preventing an old leader from continuing to act as leader.

---

# 5. Heartbeats

Once B becomes leader:

```
B = Leader

↓ heartbeat
A

↓ heartbeat
C
```

The leader periodically sends **AppendEntries** messages, even when there is no new data.

These are called heartbeats.

They basically tell followers:

> "I'm still the leader."
> 

If a follower doesn't receive heartbeats within its election timeout:

```
Follower
   ↓
Candidate
```

and starts a new election.

---

# 6. Why Random Election Timeout?

Suppose B and C both become candidates at exactly the same time.

Both ask for votes.

```
B → Vote for B
C → Vote for C
```

Neither gets a majority.

This is called a **split vote**.

Raft uses randomized election timeouts so that one server is likely to start its election first.

For example:

```
B → 350 ms
C → 500 ms
D → 700 ms
```

B starts first.

This reduces election collisions.

---

# 7. Log Replication

Now the interesting part.

Suppose B is leader.

Client sends:

```
SET x = 10
```

Leader adds it to its log:

```
B:

[1] SET x = 10
```

Then B sends the entry to followers:

```
        B
        |
   -------------
   |           |
   v           v
   C           D

[1] SET x=10
[1] SET x=10
```

Once a **majority** has stored it:

```
B ✅
C ✅
D ❌
```

2 out of 3 have it.

The entry can be considered committed.

---

# 8. Why Majority Matters

Suppose 5 nodes:

```
A B C D E
```

Leader A replicates:

```
A ✅
B ✅
C ✅
D ❌
E ❌
```

That's:

```
3 / 5
```

Majority.

So the entry can be committed.

If A later crashes:

```
A ❌
```

B and C still have the committed entry.

The next leader can recover from that state.

---

# 9. Replicated Log

The key Raft idea is that servers maintain a log.

Example:

```
Leader:

Term 1
[1] SET x=10

Term 1
[2] SET y=20

Term 2
[3] SET x=30
```

Followers try to maintain the same log.

Eventually:

```
A:
1 2 3

B:
1 2 3

C:
1 2 3
```

The log is then applied to the actual state machine.

---

# 10. Raft Is Actually About a Replicated State Machine

This is an important senior-level concept.

Raft doesn't directly mean:

> "Replicate my database."
> 

Instead:

```
Client command
      ↓
Raft replicated log
      ↓
State Machine
      ↓
Current State
```

For example:

```
SET x=10
SET x=20
DELETE x
```

All nodes apply the same commands in the same order.

Therefore they reach the same state.

---

# 11. Commit vs Apply

This distinction is important.

An entry being **replicated** doesn't necessarily mean it has been applied to the state machine.

Conceptually:

```
Log Entry
   ↓
Replicated to majority
   ↓
Committed
   ↓
Applied to state machine
```

Example:

```
[SET x=10]

A ✅
B ✅
C ❌

↓

Committed

↓

Apply

↓

x = 10
```

---

# 12. What If a Follower Is Behind?

Suppose:

```
Leader:
1 2 3 4 5

Follower B:
1 2 3
```

Leader sends the missing entries.

```
4
5
```

Follower eventually becomes:

```
1 2 3 4 5
```

---

# 13. What If the Follower Has the Wrong Entry?

This is one of Raft's important safety mechanisms.

Leader:

```
1 2 3 4
```

Follower:

```
1 2 X
```

Raft checks the previous log entry.

If the previous entry doesn't match:

```
Leader: previous = 3
Follower: previous = X
```

the AppendEntries request fails.

The leader backs up until it finds a matching prefix.

Then it replaces the conflicting suffix.

So eventually:

```
Leader:
1 2 3 4

Follower:
1 2 3 4
```

---

# 14. Network Partition

Very important interview scenario.

5 nodes:

```
A B C | D E
```

Suppose A was leader.

The network splits.

Left side:

```
A B C
```

has 3 nodes.

Right side:

```
D E
```

has 2 nodes.

The majority side can continue making progress.

The minority side cannot safely commit new entries because:

```
2 < 3
```

This prevents both sides from independently committing conflicting decisions.

---

# 15. What If the Old Leader Is on the Minority Side?

Suppose:

```
A = Leader

A B | C D E
```

A is isolated with B.

A may still think:

```
"I'm leader."
```

But:

```
A + B = 2
```

No majority.

Therefore it cannot commit new entries.

Meanwhile C/D/E can elect a new leader.

When the network heals, the old leader discovers the newer term and steps down.

This is one of the most important safety properties of Raft.

---

# 16. Can There Be Two Leaders?

You may see two servers claiming to be leaders during a network partition.

But under Raft's term/majority rules, **two leaders cannot both safely commit conflicting entries for the same term**.

The key protection is:

```
Only majority-backed leadership
        +
Term numbers
        +
Log matching
```

---

# 17. Failure Tolerance

For:

```
N = 3
```

Majority:

```
2
```

Can tolerate:

```
1 failure
```

For:

```
N = 5
```

Majority:

```
3
```

Can tolerate:

```
2 failures
```

General rule:

```
Fault tolerance = floor((N - 1) / 2)
```

So:

| Nodes | Majority | Failures tolerated |
| --- | --- | --- |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

---

# 18. Raft Request Types You Should Know

For senior interviews, know these names.

### RequestVote

Used during leader election.

```
Candidate → Followers
```

Meaning:

> "Vote for me."
> 

### AppendEntries

Used for:

- Log replication
- Heartbeats

```
Leader → Followers
```

You don't need to memorize every field, but you should know these two RPCs.

---

# 19. Raft in One Diagram

```
                Client
                   |
                   v
                Leader
                   |
          AppendEntries
           /           \
          v             v
      Follower       Follower
          |             |
          +------log----+
                   |
              Majority
                   |
               Commit
                   |
             State Machine
```

---

# 20. Raft vs Normal Primary/Replica

This is an important distinction.

Traditional replication might look like:

```
Primary
  ↓
Replica
  ↓
Replica
```

But who decides the new primary if the primary dies?

You need some coordination mechanism.

Raft provides:

```
Leader Election
+
Replicated Log
+
Majority
+
Safety
```

So Raft can be used to build a reliable replicated state machine.

---

# 21. Where You'll See Raft

### etcd

```
Kubernetes
    ↓
   etcd
    ↓
   Raft
```

### Consul

Uses Raft among its server nodes.

### Kafka KRaft

Modern Kafka uses a Raft-based protocol for controller metadata.

This is especially relevant because you've already learned Kafka.

---

# 22. Raft vs Kafka

Don't confuse them.

Kafka:

> Distributed event streaming platform.
> 

Raft:

> Consensus algorithm.
> 

Kafka can **use a Raft-based consensus mechanism internally**, but Kafka itself is much larger than Raft.

---

# 23. Raft vs Paxos

Both solve consensus.

```
Consensus
   |
   +---- Raft
   |
   +---- Paxos
```

Raft emphasizes understandability.

Paxos is historically important and appears frequently in distributed-systems literature/interviews.

We'll cover Paxos at a high level after Raft.

---

# ⭐ Senior Interview Questions

### Q1. Why does Raft require a majority?

Because a majority guarantees overlap between successive quorums.

For example:

```
Cluster = 5

Majority = 3
```

Any two groups of 3 must share at least one node.

That overlap helps preserve committed decisions.

---

### Q2. Why can't the minority partition continue?

Because it doesn't have quorum and therefore cannot safely commit new entries.

---

### Q3. What happens if the leader crashes?

```
Leader ❌
   ↓
Followers stop receiving heartbeats
   ↓
Election timeout
   ↓
Candidate
   ↓
RequestVote
   ↓
Majority
   ↓
New Leader
```

---

### Q4. What prevents an old leader from continuing?

**Term numbers + quorum + leader election rules.**

A server that discovers a newer term updates its term and becomes a follower.

---

### Q5. Why is the log important?

Because all nodes apply the same ordered sequence of commands to their state machines.

Same commands + same order → same state.

---

# 📌 Notes for Your Interview Notebook

```
RAFT

Raft = consensus algorithm.

Goal:
Maintain a consistent replicated state across distributed nodes.

3 states:
1. Follower
2. Candidate
3. Leader

Leader:
- Handles client requests
- Replicates logs
- Sends heartbeats

Candidate:
- Starts election
- Requests votes

Follower:
- Follows leader
- Becomes candidate if leader disappears

Core concepts:
- Terms
- Majority / Quorum
- Leader Election
- Heartbeats
- Replicated Log
- Commit
- State Machine

RPCs:
RequestVote
→ Leader election

AppendEntries
→ Log replication + heartbeat

Majority:
3 nodes → 2
5 nodes → 3
7 nodes → 4

Network partition:
Majority partition can continue.
Minority partition cannot safely commit.

Leader failure:
Follower → Candidate → Election → New Leader

Key idea:
Same ordered log
        ↓
Same state machine operations
        ↓
Consistent state
```

### The one sentence to remember

> **Raft uses leader election and majority-based replicated logs so distributed nodes can agree on the same ordered sequence of operations, even when some nodes fail.**
> 

Next in this sequence is **Paxos (overview)**. After that we'll connect the concepts to **ZooKeeper, etcd, distributed locks, and real production systems**.
