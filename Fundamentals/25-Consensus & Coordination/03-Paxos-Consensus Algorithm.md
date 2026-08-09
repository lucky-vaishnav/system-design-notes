```
✅ Distributed Consensus
✅ Raft
➡️ Paxos — Overview
⬜ ZooKeeper / etcd Concepts
```

# Paxos — Consensus Algorithm

## 1. What Problem Does Paxos Solve?

Same fundamental problem as Raft:

> **How can multiple distributed nodes agree on one value when some nodes may fail or messages may be delayed/lost?**
> 

Example:

```
Node A
Node B
Node C
Node D
Node E
```

They need to agree:

```
"Operation #10 = X"
```

without ending up with:

```
A → X
B → Y
C → X
```

Paxos provides a protocol for reaching one safe decision.

---

# 2. Paxos vs Raft

The most important thing:

```
Consensus
   |
   +── Raft
   |
   +── Paxos
```

Both solve **distributed consensus**.

But their approach and terminology differ.

|  | Raft | Paxos |
| --- | --- | --- |
| Main goal | Consensus | Consensus |
| Leadership | Strong leader | More flexible roles |
| Understandability | Designed to be easier | More difficult |
| Common interview use | Explain deeply | Explain conceptually |
| Log replication | Central part | Can be built using Paxos |
| Typical modern example | etcd, Kafka KRaft | Historically important |

For your senior interviews, **Raft deserves deeper implementation knowledge; Paxos mainly needs conceptual understanding**.

---

# 3. Paxos Roles

Classic Paxos has three important roles:

```
Proposer
Acceptor
Learner
```

Think of them like this:

```
Client
  |
  v
Proposer
  |
  v
Acceptors
  |
  v
Learners
```

---

# 4. Proposer

The proposer says:

> "I propose this value."
> 

Example:

```
Proposer:

value = "X"
```

It asks acceptors to consider the proposal.

---

# 5. Acceptor

Acceptors are the nodes that actually participate in deciding whether a proposal can be accepted.

Example:

```
A
B
C
D
E
```

A majority must agree.

For 5 nodes:

```
3 / 5
```

---

# 6. Learner

Learners learn the final chosen value.

For example:

```
Chosen value = X

↓

Learner
Learner
Learner
```

The learner doesn't propose the value.

It learns what the consensus group decided.

---

# 7. The Two Main Phases

This is the most important Paxos concept.

Classic Paxos works in two major phases:

```
Phase 1 → Prepare
Phase 2 → Accept
```

---

# 8. Phase 1 — Prepare

Suppose proposer wants:

```
X
```

It generates a unique proposal number:

```
Proposal #10
```

and sends:

```
Prepare(10)
```

to acceptors.

```
       Proposer
          |
    Prepare(10)
      /  |  \
     v   v   v
    A    B    C
```

---

# 9. What Does an Acceptor Do?

The acceptor says:

> "I promise not to accept any proposal with a number lower than 10."
> 

For example:

```
Acceptor A

Promise:

I won't accept
proposal < 10
```

This is important because it prevents older proposals from interfering with newer ones.

---

# 10. Phase 2 — Accept

Once proposer gets promises from a majority:

```
A ✅
B ✅
C ❌
```

It has quorum.

Now it sends:

```
Accept(10, X)
```

to the acceptors.

```
Proposer
   |
Accept(10, X)
 /       \
A         B
```

If a majority accepts:

```
A ✅
B ✅
C ❌
```

then:

```
X = chosen
```

---

# 11. The Critical Rule

Here's the part that makes Paxos safe.

Suppose an acceptor had **already accepted an earlier value**.

Example:

```
Proposal 5 → Y
```

Now proposer comes with:

```
Proposal 10 → X
```

During Phase 1, the acceptor tells the proposer:

> "I previously accepted Y under proposal 5."
> 

The new proposer **cannot simply ignore that information**.

It must preserve the already-established value according to Paxos's rules.

This is how Paxos prevents two different values from being chosen.

---

# 12. Why Proposal Numbers Matter

Imagine:

```
Proposal 5
Proposal 10
Proposal 20
```

Higher numbers represent newer proposals.

An acceptor can say:

```
I already promised proposal 20.

Therefore:

Proposal 10 ❌
Proposal 5  ❌
```

This prevents an old proposer from coming back later and changing the decision.

---

# 13. Why Majority Again?

Suppose 5 acceptors:

```
A B C D E
```

Majority:

```
3
```

One proposal might get:

```
A B C
```

Another quorum might get:

```
C D E
```

Notice:

```
        C
        ↑
Both quorums overlap
```

That overlap is critical.

Because they share an acceptor, the system can preserve information about previously accepted proposals.

This is the same fundamental quorum idea you saw in Raft.

---

# 14. Network Partition

Suppose:

```
A B C | D E
```

Left side:

```
3 nodes
```

Right:

```
2 nodes
```

The 3-node side has quorum.

The 2-node side doesn't.

Therefore:

```
3 nodes → can make progress

2 nodes → cannot safely choose new values
```

Again:

**No quorum → no new consensus decision.**

---

# 15. Paxos in One Diagram

```
              Proposer
                  |
             Prepare(10)
                  |
          +-------+-------+
          ↓       ↓       ↓
          A       B       C
          |       |       |
       Promise  Promise  Promise
          \       |       /
           \      |      /
             Majority
                 |
                 ↓
          Accept(10, X)
             /    \
            A      B
             \    /
            Majority
                 |
                 ↓
             X chosen
                 |
                 ↓
              Learner
```

---

# 16. Paxos vs Raft — Important Difference

Raft essentially says:

```
Elect one leader

↓

Leader controls log

↓

Leader replicates log

↓

Majority commits
```

Paxos is more general and doesn't require exactly the same strong-leader structure.

Classic Paxos focuses on:

```
Proposer
Acceptor
Learner
```

and reaching agreement through proposals and quorums.

This is one reason Paxos can feel more complicated.

---

# 17. Multi-Paxos

Classic Paxos is mainly about agreeing on **one value**.

Real systems often need:

```
Operation 1
Operation 2
Operation 3
Operation 4
...
```

Doing a complete Paxos round for every operation is expensive.

**Multi-Paxos** extends the idea to efficiently agree on a sequence of values.

This starts to look conceptually similar to:

```
Replicated Log
```

which is why Raft is often easier to reason about for replicated state machines.

---

# 18. Paxos and Leader Election

A common misconception:

> "Paxos is just a leader election algorithm."
> 

No.

Paxos is a **consensus algorithm**.

It can be used as part of systems that need:

```
Leader election
Configuration agreement
Replicated logs
Distributed coordination
```

But consensus is the broader problem.

---

# 19. What About Failures?

Paxos is designed to remain safe despite failures such as:

```
Node crash
Message loss
Message delay
Network partition
```

But there's an important distinction:

### Safety

The system should **never choose conflicting values**.

### Liveness

The system should eventually make progress.

A network partition can prevent progress because quorum may be unavailable.

So:

```
Safety > Availability during partition
```

for a consensus system.

---

# 20. Paxos vs CAP

Don't confuse these concepts.

### CAP

A theorem describing a fundamental trade-off in distributed systems under partition.

### Paxos

An algorithm for reaching consensus.

Relationship:

```
Network Partition
       ↓
Need safe coordination
       ↓
Consensus algorithm
       ↓
Paxos / Raft
```

---

# 21. Where Is Paxos Used?

Paxos is historically very important and has influenced several distributed systems.

You don't need to memorize a long list of products for interviews.

The important point is:

> **Paxos is one of the foundational consensus algorithms on which many distributed-system designs are based.**
> 

For modern practical systems, you'll more commonly encounter **Raft-based systems such as etcd and Consul**, while ZooKeeper uses its own coordination protocol.

---

# ⭐ Senior Interview Questions

### Q1. What is Paxos?

> Paxos is a distributed consensus algorithm that allows a group of nodes to agree on a single value despite node failures and unreliable communication.
> 

---

### Q2. What are the three roles?

```
Proposer
Acceptor
Learner
```

---

### Q3. What are the two main phases?

```
1. Prepare / Promise
2. Accept
```

---

### Q4. Why do we need proposal numbers?

To establish ordering between proposals and prevent older proposals from overriding newer promises.

---

### Q5. Why majority?

Because two majorities must overlap, allowing information about previously accepted decisions to be preserved.

---

### Q6. Paxos vs Raft?

> Both solve consensus. Raft uses a more explicit leader-based replicated-log architecture and was designed for understandability, while Paxos uses proposer/acceptor/learner roles and is more abstract.
> 

---

### Q7. Can a minority partition commit a value?

**No.** It doesn't have quorum.

---

# 📌 Notes for Your Notebook

```
PAXOS

Paxos = Distributed Consensus Algorithm

Goal:
Allow distributed nodes to agree on one value
despite failures and unreliable communication.

Roles:
1. Proposer
2. Acceptor
3. Learner

Two main phases:

Phase 1:
Prepare
→ Promise

Phase 2:
Accept
→ Value can become chosen after majority acceptance.

Important:
Proposal numbers
Majority / Quorum
Previously accepted values
Safety

For 5 nodes:
Majority = 3

Network partition:
Majority → can make progress
Minority → cannot safely commit

Paxos ≠ Leader Election
Paxos = Consensus

Multi-Paxos:
Efficiently reaches consensus on a sequence of values.

Raft vs Paxos:
Both solve consensus.
Raft = leader-based + replicated log + easier to understand.
Paxos = proposer/acceptor/learner + more abstract.
```

## The one sentence to remember

> **Paxos uses numbered proposals, promises, acceptances, and quorum overlap to ensure that distributed nodes choose one consistent value despite failures.**
> 

### Next topic

Now we move from **algorithms** to the systems that actually provide coordination:

```
Distributed Consensus
        ↓
Raft
        ↓
Paxos
        ↓
➡️ ZooKeeper / etcd Concepts
```

That next topic is especially useful for a senior backend developer because we'll connect **consensus → leader election → distributed locks → service discovery → configuration → Kubernetes → Kafka**.
