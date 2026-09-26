
Our overall case-study next roadmap is:

| #     | Topic                                        | Status             |
| ----- | -------------------------------------------- | ------------------ |
| 1     | Transactions + Concurrency                   | ✅ Complete         |
| 2     | Consistency + Cache Strategy                 | ✅ Complete         |
| 3     | Payment Failures + Reconciliation            | ✅ Complete         |
| 4     | Saga + Outbox + Kafka                        | ✅ Complete         |
| **5** | **Scaling, Bottlenecks & High Availability** | 🟡 **In progress** |
| 6     | Security + Rate Limiting + Reliability       | ⏳                  |
| 7     | Observability + DR                           | ⏳                  |
| 8     | Final Technology-Agnostic Architecture       | ⏳                  |
| 9     | AWS Mapping                                  | ⏳                  |
| 10    | AWS Failures + Trade-offs                    | ⏳                  |
| 11    | Full Walkthrough                             | ⏳                  |
| 12    | Mock Interview                               | ⏳                  |

And **#5 itself** has this structure:

```text
5. Scaling, Bottlenecks & High Availability
│
├── 5.1 Capacity & Bottleneck Identification   ← WE ARE HERE
│
├── 5.2 Application/API Scaling
│
├── 5.3 Database Scaling
│     ├── Connection pools
│     ├── Read scaling
│     ├── Write scaling
│     ├── Hot rows
│     ├── Partitioning
│     └── Sharding
│
├── 5.4 Cache Scaling
│     ├── Hot keys
│     └── Redis HA/scaling
│
├── 5.5 Queue/Kafka Scaling
│     ├── Partitions
│     ├── Consumers
│     └── Consumer lag
│
├── 5.6 Traffic Spikes & Backpressure
│
├── 5.7 High Availability
│     ├── AZ failures
│     ├── DB failure
│     ├── Redis failure
│     └── Service failure
│
└── 5.8 End-to-End Bottleneck Analysis
```
