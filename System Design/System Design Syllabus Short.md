# Complete System Design Syllabus — From Foundations to Mastery (Concise + Fun)

## How to Use This
Most resources teach *templates*. This teaches *thinking*.  
Goal: walk into any unfamiliar system and derive a good design from first principles—without “I saw something like this once.”

Rule of thumb: **don’t skip foundations**. They’re the reason your later tradeoffs make sense.

---

## Phase 0: Mindset (Before You Draw Boxes)
System design is a series of tradeoffs:

- **Speed vs Consistency**
- **Simplicity vs Scale**
- **Cost vs Reliability**

Key habits:
- **There’s no universally correct architecture**—only “correct for these requirements + constraints.”
- **Start simple, scale with evidence.** Over-engineering kills more projects than under-engineering.
- **Clarify before you propose.** Your questions are part of the solution.

---

## Phase 1: Foundations (The “Physics” of Systems)

### 1.1 Networking (Where Latency Comes From)
Know:
- **Layering (OSI conceptually):** what each layer guarantees—and what it doesn’t.
- **TCP vs UDP:** reliability/ordering vs low-latency (bank transfer vs video call).
- **HTTP request cost stack:** DNS → TCP handshake → TLS → request/response.
- **HTTP/1.1 vs HTTP/2 vs HTTP/3:** which bottleneck each fixed (HOL blocking → multiplexing → QUIC).
- **DNS caching:** why first request is slower and why CDNs work.
- **Realtime choices:** WebSockets (2-way), SSE (1-way), long polling (fallback).
- **L4 vs L7 load balancing:** fast + dumb vs slower + smart routing.

Numbers to internalize:
- Same datacenter RTT: ~**0.5 ms**
- Cross-continent RTT: ~**150 ms** (physics wins)

Exercises:
- `curl -v`, `curl --http2`, `dig`, Wireshark a TCP/TLS handshake.

---

### 1.2 Databases (Where Most Designs Win or Die)
SQL essentials:
- **Indexes (B-tree intuition):** predict fast vs slow queries.
- **Query plans:** `EXPLAIN (ANALYZE)`—learn to spot scans and bad joins.
- **ACID (mechanisms, not slogans):** WAL, constraints, MVCC/locks, durability.
- **Isolation levels + anomalies:** choose correctness vs throughput intentionally.
- **Normalize vs denormalize:** join cost vs duplication/consistency complexity.
- **Replication + lag:** read replicas break “read-your-writes” unless designed for it.

NoSQL map (pick by access pattern):
- **Document:** read stuff together.
- **Key-value:** lookup-by-key, caching/sessions.
- **Wide-column:** write-heavy, query-driven schema.
- **Graph:** relationships-first workloads.
- **Time-series:** time-indexed, compressed metrics.

The most important DB question:
- **What are the top access patterns + consistency needs?**

Exercises:
- Take 5 real queries → run `EXPLAIN ANALYZE` → index + measure.
- Compare a normalized vs denormalized version for the same workload.

---

### 1.3 Caching (Performance With a Side of Staleness)
Mental model: cache = **speed purchased with complexity**.

Layers:
- In-process (fastest, inconsistent across servers)
- Redis/memcached (shared cache)
- CDN/edge (geo-close content)
- DB internal caches (helpful but not your primary plan)

Patterns:
- **Cache-aside** (common; cold-start + staleness risk)
- **Write-through** (consistent; slower writes)
- **Write-behind** (fast writes; durability risk)
- **Read-through** (cache acts as proxy)

Hard parts:
- **Invalidation:** TTL, events, versioned keys
- **Thundering herd:** jitter TTLs, locks-on-miss, early refresh

Exercise:
- Build cache-aside + TTL → intentionally create stale reads → fix invalidation.

---

### 1.4 Concurrency (How Bugs Get Invented)
Know:
- **Race conditions** and why they’re “works in dev, explodes in prod.”
- **Locks:** granularity and contention costs.
- **Read-write locks:** useful for read-heavy workloads.
- **Atomics/CAS:** how “lock-free-ish” structures work.
- **Optimistic concurrency:** retry on conflict (great when conflicts are rare).
- **Distributed locks:** Redis/ZooKeeper basics + failure cases (crash while holding lock).

Exercise:
- Write a race → fix via lock vs atomics → compare throughput.
- Implement Redis lock + test lock expiry mid-work.

---

### 1.5 Distributed Systems (Where Certainty Goes to Die)
Why distributed: one box can’t do it all. But distribution adds:
- latency, partial failure, coordination, consistency problems.

Core concepts:
- **CAP:** during partitions, choose C or A (you always have partitions eventually).
- **PACELC:** even without partitions, you trade latency vs consistency.
- **Consistency spectrum:** strong → eventual → read-your-writes → causal.
- **Replication modes:** single leader, multi-leader (conflicts), leaderless/quorums.
- **Consensus:** Raft/Paxos concepts (leader election, committed log).

Exercise:
- Primary + replica Postgres → observe replication lag → simulate failover.

---

## Phase 2: Core System Design Building Blocks (Your LEGO Set)

- **Load balancing:** RR, weighted, least-conns, adaptive; health checks; avoid sticky sessions.
- **Scaling:** vertical first, horizontal when forced; understand what breaks (state, DB, files, jobs).
- **System-level caching:** where (client/edge/app/DB), what (hot data), hit-rate monitoring, warmups.
- **Queues & async:** buffering peaks, decoupling, retries.
  - RabbitMQ/SQS = tasks
  - Kafka = event log + replay + many consumers
  - Delivery semantics: at-most-once / at-least-once (idempotency!) / exactly-once (expensive)
- **Event-driven architecture:** auditability + decoupling; debugging/tracing; schema evolution.
- **Monolith vs microservices:** start modular monolith; pay the “microservices tax” only for real benefits.
- **API design:** REST basics, versioning, pagination, errors; GraphQL for complex fetching; gRPC for internal performance.
- **Rate limiting:** token bucket, sliding windows; Redis atomic ops; return headers.
- **CDN:** cache static and semi-dynamic; content-hash filenames; protect origin.
- **Storage:** object vs block vs file; pick based on latency + sharing + durability.

---

## Phase 3: LLD Patterns + SOLID (Code-Level Architecture)
Treat patterns as **named tools for recurring problems**, not collectibles.

- **Creational:** Singleton (careful), Factory/Abstract Factory, Builder, Prototype
- **Structural:** Adapter, Decorator (middleware), Facade, Proxy (AOP), Composite
- **Behavioral:** Observer, Strategy, Command, Chain of Responsibility, Template Method, State, Iterator, Mediator
- **SOLID:** SRP, OCP, LSP, ISP, DIP (use these to explain *why* patterns exist)

---

## Phase 4: Case Studies (Practice the Reasoning)
Do each with the Phase 5 framework; focus on the “hard part.”

1. **URL shortener:** ID generation + cache-heavy read path  
2. **Notification system:** queues, dedupe, priorities, provider limits  
3. **Ride-sharing:** geo queries + match contention + real-time updates  
4. **Video streaming:** CDN + ABR + encoding pipeline  
5. **Social feed:** fan-out (push/pull hybrid) + feed storage  
6. **Payments:** strong consistency + idempotency + ledger + sagas  
7. **Search:** inverted index + ranking + sharding + autocomplete  
8. **Messaging:** WebSockets + ordering + offline store-and-forward + group fan-out

---

## Phase 5: The 7-Step System Design Playbook
1. **Clarify requirements** (use cases, scale, latency, consistency, constraints)
2. **Define scope** (what you’ll do vs defer)
3. **Estimate** (RPS, storage, bandwidth, peaks)
4. **Data model first**
5. **API contracts**
6. **High-level architecture** (start simple; add components to fix specific bottlenecks)
7. **Deep dive the hardest problem** (and failure modes)

After: stress test assumptions (dependency failure, 10x load, cost).

---

## Phase 6: Practice Roadmap (12 Months, Realistic)
- **Months 1–2:** Foundations (DB → networking → caching → concurrency → distributed)
- **Month 3:** Core building blocks (implement small versions)
- **Month 4:** LLD patterns (refactor real code)
- **Months 5–8:** 1 case study/week (design first, then compare)
- **Months 9–12:** Novel designs + feedback loops

---

## Phase 7: Common Traps (Avoid These for Free)
- Designing before clarifying requirements
- Defaulting to microservices
- Drawing boxes before the data model
- Name-dropping tech instead of explaining tradeoffs
- “Add Redis” without TTL + invalidation + cold-start plan
- Ignoring ops: monitoring, deploys, tracing, on-call reality
- Building for “millions” when you have “hundreds”
- Never practicing out loud

---

## Phase 8: Projects (Build Muscle, Not Just Notes)
1. Key-value store (TTL → WAL → replication)
2. Consistent hash ring (with virtual nodes)
3. Task queue (retries, visibility timeouts, priorities)
4. Redis rate limiter (token bucket)
5. URL shortener (end-to-end + load test)
6. Notification system (EDA + dedupe + priorities)
7. Tiny search engine (inverted index + TF-IDF) then compare to Elasticsearch
8. Distributed lock (and break it on purpose)
9. Document your current system as an ADR + propose 3 improvements
10. Monthly “new prompt” full design write-up
