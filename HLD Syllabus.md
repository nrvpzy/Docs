# SDE-2 HLD — Step-by-Step Learning Order

| Step | Topic | Resource |
|---:|---|---|
| 1 | APIs — API design fundamentals | https://lnkd.in/ezwnCGqS |
| 2 | REST | https://lnkd.in/eY2ACHFC |
| 3 | REST vs GraphQL vs gRPC vs tRPC | https://lnkd.in/eydTuVj3 |
| 4 | Stateful vs Stateless Architecture | https://lnkd.in/egXhAmY4 |
| 5 | Proxy vs Reverse Proxy | https://lnkd.in/enEy9QYD |
| 6 | Load Balancing & Algorithms | https://lnkd.in/ewTeu-58 |
| 7 | API Gateway | https://lnkd.in/eqNrc77q |
| 8 | CDN | https://lnkd.in/eCSccEkz |
| 9 | Sync vs Async Communication | https://lnkd.in/ekrADFHy |
| 10 | Services / Microservices | https://lnkd.in/exyDGmSe |
| 11 | Databases — Fundamentals & Choosing a Database | https://lnkd.in/eifbKsr6 |
| 12 | SQL vs NoSQL | https://lnkd.in/entah3zc |
| 13 | ACID Transactions | https://lnkd.in/etXk_wa4 |
| 14 | Database Indexing | — |
| 15 | Database Replication — Primary/Replica, Read Replicas, Sync vs Async Replication, Failover | — |
| 16 | Data Sharding & Partitioning | https://lnkd.in/eVhzCnW5 |
| 17 | Database Sharding Strategies | https://lnkd.in/g5_pUPyD |
| 18 | CAP Theorem | https://lnkd.in/eePkq2kJ |
| 19 | Consistency Models & Trade-offs | — |
| 20 | Caching Fundamentals & Caching Strategies | https://lnkd.in/gq97n6UR |
| 21 | Caching in System Design Interviews | https://www.youtube.com/watch?v=1NngTUYPdpI |
| 22 | Consistent Hashing | https://lnkd.in/eYgXNHz4 |
| 23 | Bloom Filters | https://lnkd.in/eq6hN3n |
| 24 | Message Queues — Fundamentals | https://lnkd.in/eKQWVxqw |
| 25 | Kafka / Distributed Messaging — Partitions, Consumer Groups, Offsets, Ordering, Rebalancing | — |
| 26 | Message Delivery Semantics — At-most-once, At-least-once, Effectively-once | — |
| 27 | Short Polling vs Long Polling vs WebSockets | https://lnkd.in/eYZnk-93 |
| 28 | Idempotency & Idempotent APIs | https://lnkd.in/e-sB7a3w |
| 29 | Concurrency vs Parallelism | https://lnkd.in/eRpCq8KQ |
| 30 | Distributed Locks | — |
| 31 | Distributed Transactions — 2PC | — |
| 32 | SAGA Pattern & Compensation | — |
| 33 | Outbox Pattern & Dual-Write Problem | — |
| 34 | Event-Driven Architecture | — |
| 35 | Rate Limiting Algorithms | https://lnkd.in/etby2w5C |
| 36 | Timeout & Retry Patterns | — |
| 37 | Exponential Backoff & Jitter | — |
| 38 | Circuit Breaker | — |
| 39 | Bulkhead Pattern | — |
| 40 | Backpressure & Load Shedding | — |
| 41 | Thundering Herd / Cache Stampede | — |
| 42 | High Availability — Active-Passive & Active-Active | — |
| 43 | Service Discovery | — |
| 44 | Fault Tolerance & Graceful Degradation | — |
| 45 | Observability — Logs, Metrics & Monitoring | — |
| 46 | Distributed Tracing & Correlation IDs | — |
| 47 | Disaster Recovery — Backup, Restore, RPO & RTO | — |
| 48 | Multi-Region Architecture | — |
| 49 | OAuth 2.0 | — |
| 50 | JWT | https://lnkd.in/eAnfnzm7 |
| 51 | Authentication vs Authorization | — |
| 52 | HTTPS / TLS & Basic Encryption Concepts | — |
| 53 | CORS / CSRF / XSS / SQL Injection — Security Awareness | — |
| 54 | Back-of-the-Envelope Estimation | — |
| 55 | System Design: URL Shortener | — |
| 56 | System Design: Rate Limiter | — |
| 57 | System Design: WhatsApp / Chat System | — |
| 58 | System Design: Notification System | — |
| 59 | System Design: Ticket Booking System | — |
| 60 | System Design: Payment System | — |
| 61 | System Design: E-commerce / Order System | — |
| 62 | System Design: News Feed | — |
| 63 | System Design: YouTube / Video Streaming | — |
| 64 | System Design: Uber / Ride Booking | — |
| 65 | System Design: Distributed File Storage | — |
| 66 | System Design: Job Scheduler | — |
| 67 | System Design: Search / Autocomplete | — |

---

### Why this order?

The progression is intentional:

**1–10 → Request/traffic layer**

You first understand how a request travels:

`Client → CDN → Load Balancer → API Gateway → Service`

**11–19 → Database layer**

Then understand where data lives and how it scales:

`DB → Transactions → Indexes → Replication → Sharding → Consistency`

**20–27 → Performance + async communication**

Then:

`Cache → Consistent Hashing → Queue → Kafka → WebSockets`

**28–34 → Distributed correctness**

This is where SDE-2 interviews become much more interesting:

`Idempotency → Concurrency → Locks → Transactions → SAGA → Outbox → Events`

**35–48 → Production reliability**

Now learn what happens when things go wrong:

`Rate Limit → Retry → Circuit Breaker → Backpressure → HA → Observability → DR → Multi-region`

**49–53 → Security**

You don't need to become a security engineer, but you should be able to make sensible choices in a system-design interview.

**54 → Estimation**

I'd actually practice estimation throughout the course, but have one dedicated session for it before starting full designs.

**55–67 → Actual interview practice**

This is the most important section.

Don't watch a solution first.

For each problem, give yourself **45–60 minutes** and independently go through:

```text
Requirements
     ↓
Scale / Estimation
     ↓
APIs
     ↓
High-Level Architecture
     ↓
Database + Data Model
     ↓
Caching
     ↓
Async Processing / Queues
     ↓
Consistency
     ↓
Failure Handling
     ↓
Scaling
     ↓
Bottlenecks
     ↓
Trade-offs
```

### One important thing

You **don't need to study all 67 steps with equal depth**.

For SDE-2, I'd make **1–48 your theory foundation**, then spend a lot of time on **55–67**.

If you're short on time, the highest-ROI topics are:

**APIs → LB → Caching → DB → Replication → Sharding → CAP/Consistency → Kafka → Idempotency → SAGA/Outbox → Rate Limiting → Retry/Circuit Breaker → HA → Observability → Estimation → System-design practice.**

That combination is more valuable than going extremely deep into every individual topic.

