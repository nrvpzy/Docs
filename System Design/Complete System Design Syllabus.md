# Complete System Design Syllabus — From Foundations to Mastery

---

## How to Use This Syllabus

Before anything else, understand what this syllabus is trying to do. Most system design resources teach you answers. This syllabus teaches you reasoning. The difference is everything.

When you finish this syllabus, you should be able to sit in front of a system you have never seen before and reason through it — not because you memorized something similar, but because you understand the underlying forces well enough to derive the answer from first principles.

Each section builds on the previous one. Do not skip ahead. The foundations are not optional prerequisites you can skim — they are the source of the intuition that makes everything else make sense.

---

## Phase 0: The Mindset Before You Begin

Internalize these before studying anything else.

Every system design problem is a negotiation between competing forces. Speed vs consistency. Simplicity vs scalability. Cost vs reliability. An architect's job is not to eliminate these tensions — it is to understand them clearly enough to make conscious tradeoffs.

There is no universally correct architecture. There is only the architecture that is correct for a specific set of requirements, constraints, team skills, and business context. When you see something in a real system that looks wrong, your first instinct should be "what constraint led to this decision?" not "this is wrong."

Start simple and scale only when you can measure the need. Over-engineering is as dangerous as under-engineering. The graveyard of failed software projects is full of beautifully designed systems that solved problems the product never actually had.

Understand the problem before proposing the solution. This sounds obvious. Almost nobody does it.

---

## Phase 1: Foundations — The Ground Everything Else Stands On

These are not "nice to know." They are the physical constraints of computing that drive every architectural decision you will ever make. Skip these and your system design reasoning will always be shallow.

---

### 1.1 Networking Fundamentals

**What you need to understand:**

The OSI model at a conceptual level — not to memorize all seven layers, but to understand why they exist. Each layer abstracts the one below it. When you understand this, you understand why HTTP is layered on TCP, why TCP is layered on IP, and what each layer guarantees and what it doesn't.

TCP vs UDP is one of the most important architectural decisions in real-time systems. TCP guarantees delivery and ordering but at the cost of latency and overhead. UDP sacrifices both guarantees for speed. Video streaming uses UDP because a dropped frame is better than a frozen screen. Your bank's transaction system uses TCP because a dropped transaction is catastrophic. Know when each is correct.

HTTP — understand deeply how it actually works. A request goes out, a response comes back, but what happens in between? DNS resolution, TCP handshake, TLS handshake, request serialization, server processing, response deserialization. Each step has a cost. Understanding these costs tells you where latency comes from and what caching actually saves.

HTTP/1.1 vs HTTP/2 vs HTTP/3. The evolution of HTTP is a story of solving real performance problems. HTTP/1.1's head-of-line blocking problem led to HTTP/2's multiplexing. HTTP/2's TCP head-of-line blocking led to HTTP/3's QUIC protocol. Understanding the problem each version solved tells you which to use when.

DNS and how caching works within it. A DNS resolution takes 20-120ms the first time and near-zero after caching. This is why CDNs work and why DNS propagation takes time when you change records.

WebSockets vs long polling vs server-sent events. When you need the server to push data to the client, these are your options. Long polling is a hack that works when WebSockets aren't available. Server-sent events are for one-directional server-to-client streaming. WebSockets are for bidirectional real-time communication. Know the tradeoffs.

TLS and where its cost actually comes from. The TLS handshake adds 1-2 round trips before any data flows. Session resumption eliminates most of this cost. Understanding this explains why HTTPS has become essentially free in modern infrastructure.

Load balancers and proxies at the network level. The difference between Layer 4 (transport) and Layer 7 (application) load balancing. L4 is faster but knows nothing about the request. L7 can route based on URL, headers, or content but adds processing overhead.

**Numbers to internalize — these drive architecture:**

A same-datacenter network round trip takes approximately 0.5 milliseconds. A cross-continent round trip takes 150 milliseconds. This is physics — you cannot optimize it away. Any architecture that requires multiple cross-continent round trips for a single user action will feel slow.

**Suggested exercises:**

Use `curl -v` on a URL and trace every step that happens. Then use `curl --http2` and compare. Use `dig` to trace DNS resolution. Set up Wireshark and watch a TCP handshake happen. These are not optional — watching the actual packets trains intuition that reading never can.

---

### 1.2 Databases — The Most Critical Foundation

Databases are where most architectural decisions live, and most developers have a surface-level understanding that breaks under examination. Go deep here.

**SQL Databases — what you actually need to understand:**

How indexes work at the data structure level. A B-tree index is a sorted tree structure. Queries that can use this sorted structure are fast. Queries that cannot are slow. Understanding this — not just "add an index" but why it works — means you can predict which queries will be slow before you run them.

What a query plan is and how to read it. Every SQL query gets compiled into an execution plan before it runs. The query planner decides whether to use indexes, how to join tables, and in what order to execute operations. Learn to run EXPLAIN ANALYZE in PostgreSQL or EXPLAIN in MySQL. Read the output. Find the table scans. This skill alone will save you from more production incidents than any other.

ACID properties — not as an acronym but as a mechanism. Atomicity means all-or-nothing, implemented through write-ahead logging. Consistency means data constraints are always satisfied. Isolation means concurrent transactions don't interfere, implemented through locks or MVCC (Multi-Version Concurrency Control). Durability means committed data survives crashes, implemented through flushing the write-ahead log to disk. Know how each property is implemented, not just what it means.

Transactions and isolation levels. Read Uncommitted, Read Committed, Repeatable Read, Serializable — these are not arbitrary settings. Each one trades concurrency for correctness. Understanding the anomalies each level prevents (dirty reads, non-repeatable reads, phantom reads) tells you which isolation level a business requirement demands.

Normalization and when to denormalize. Third normal form eliminates redundancy and ensures consistency. But normalized schemas often require multiple joins, which are expensive at scale. Denormalization trades storage and consistency complexity for read performance. The right choice depends on whether you're read-heavy or write-heavy, and how much consistency complexity you can manage.

Database replication — how it works and what it guarantees. A primary receives writes and replicates them to one or more replicas. Replicas can serve reads. But replication has lag — there's always some window during which a replica doesn't have the latest data. Your architecture must account for this. "Read your own writes" consistency is violated when you read from a replica immediately after writing to the primary.

**NoSQL Databases — understand the categories and why each exists:**

The key insight about NoSQL: each NoSQL database was built to solve a specific limitation of relational databases for specific access patterns. There is no single "NoSQL." There are document stores, wide-column stores, key-value stores, graph databases, and time-series databases. Each optimizes for different things.

Document databases like MongoDB store data as documents (JSON-like structures). They excel when your data is hierarchical and you always access it together. An order with its line items, shipping address, and payment info can be one document — one read instead of four joins. They struggle when you need to query across documents in complex ways or when your data model changes frequently.

Key-value stores like Redis store simple key-value pairs in memory. Access is O(1) and extremely fast. They're perfect for caching, session storage, and any access pattern that is lookup-by-key. They're useless for anything that requires querying by value.

Wide-column stores like Cassandra organize data in tables with rows and dynamic columns. Designed for write-heavy workloads at massive scale. The data model is query-driven — you design your schema based on how you'll query it, not on the relationships between entities. This makes it extremely fast for the queries it's designed for and impossible for queries it's not.

Graph databases like Neo4j store data as nodes and edges. For data that is inherently relational — social networks, recommendation engines, fraud detection — graphs are dramatically more natural and performant than SQL joins. For most other use cases, they're unnecessary complexity.

Time-series databases like InfluxDB or TimescaleDB optimize for data that has a time dimension — metrics, IoT sensor data, financial prices. They compress this data extremely efficiently and provide time-based query primitives that general-purpose databases don't have.

**The most important database question in any design:**

What are my access patterns? What queries will run most frequently? What are the consistency requirements? The answers to these questions determine the database choice, the schema design, and the indexing strategy. Without answering these first, all database decisions are guesses.

**Suggested exercises:**

Take any schema from a project you've worked on. Run EXPLAIN ANALYZE on five of its most common queries. Find the table scans. Add indexes and measure the difference. Then intentionally design a bad schema — highly normalized — and compare its query performance to a denormalized version for the same access patterns.

---

### 1.3 Caching — The Most Misunderstood Topic

Caching is not "put things in Redis." Caching is a set of strategies for trading consistency complexity for performance. Understanding this reframe is essential.

**What caching actually is:**

A cache is a faster, smaller, more expensive storage layer that stores a subset of data from a slower, larger, cheaper storage layer. The bet you make with caching is that the performance gain outweighs the consistency complexity you introduce. Sometimes this bet is correct. Sometimes it isn't.

**Caching layers — from fastest to slowest:**

In-process memory (L1/L2/L3 CPU cache, JVM heap) is the fastest cache available. Data here is accessed in nanoseconds. It's also the smallest and lives only in one process on one machine. When you scale to multiple servers, in-process caches become inconsistent with each other unless you explicitly synchronize them.

Distributed cache like Redis sits outside the application but is shared across all application instances. It's slower than in-process cache (network round trip: 0.5ms) but solves the consistency problem across multiple servers. It's the right choice when you need cache sharing or when your data doesn't fit in a single process's memory.

CDN (Content Delivery Network) caches static and semi-static content geographically close to users. Serving an image from a CDN node 10ms away is dramatically faster than serving it from your origin server 150ms away. CDNs are a specific form of caching with geographic distribution built in.

Database query cache — most databases have internal caches that cache the results of frequently run queries. These are transparent to the application and generally not something you design explicitly, but understanding they exist helps you understand why repeated identical queries can be faster than the first run.

**Cache strategies — the four patterns:**

Cache-aside (lazy loading) means the application checks the cache first. If the data is there (cache hit), it returns it. If not (cache miss), it queries the database, stores the result in the cache, and returns it. This is the most common strategy. Its weakness is that the first request for any data always hits the database (cold cache problem) and that the cache can become stale if the database is updated by something other than your application.

Write-through means every write to the database also writes to the cache simultaneously. The cache is always consistent with the database. Its weakness is that writes are slower (you wait for both) and you cache data that might never be read.

Write-behind (write-back) means writes go to the cache first and are asynchronously flushed to the database. Writes feel fast. The weakness is that data can be lost if the cache fails before flushing, making it inappropriate for anything that requires durability.

Read-through means the cache sits in front of the database and the application only ever talks to the cache. Cache misses are handled by the cache itself (it fetches from the database). This simplifies application code but is less flexible than cache-aside.

**Cache invalidation — the hardest problem:**

Phil Karlton famously said there are only two hard things in computer science: cache invalidation and naming things. He was right.

When the underlying data changes, the cache must be updated or invalidated. Strategies include: time-to-live (TTL) expiration (accept some staleness, simple to implement), event-driven invalidation (write to cache when database changes, complex but accurate), and versioning (include a version in the cache key, never invalidate — let old versions expire). Each has tradeoffs in consistency, complexity, and performance.

**The thundering herd problem:**

When a cached item expires and thousands of simultaneous requests all get a cache miss at the same moment, they all hit the database simultaneously. This can overwhelm the database. Solutions include probabilistic early expiration (proactively refresh before TTL expires), mutex/locking on cache miss (only one request fetches from database, others wait), and staggered TTL (add random jitter to TTL to spread expiration).

**Suggested exercises:**

Implement a cache-aside pattern with Redis in a Spring Boot application. Add TTL. Then deliberately cause a cache invalidation problem — update the database directly and observe stale data in the cache. Then implement proper invalidation. This experience teaches more than any amount of reading.

---

### 1.4 Concurrency

Concurrency is the source of the most subtle and catastrophic bugs in systems. Architects need to understand it at a level that goes beyond "use synchronized."

**The fundamental problem:**

Multiple threads or processes accessing shared mutable state simultaneously. Without coordination, reads and writes can interleave in ways that produce incorrect results. The classic example is two bank account threads both reading a balance of ₹1000, both adding ₹500, and both writing ₹1500 — when the correct result is ₹2000.

**Race conditions and how they manifest:**

A race condition exists whenever the correctness of a program depends on the relative timing of events. They're notoriously difficult to reproduce and debug because they depend on timing, which is non-deterministic. Understanding what conditions create race conditions is the first step to avoiding them.

**Synchronization mechanisms and their costs:**

Locks (mutexes) are the simplest synchronization mechanism. One thread holds the lock; all others wait. They work but create contention — threads waiting on locks aren't doing useful work. Lock granularity matters — a single global lock serializes everything, while fine-grained locks per record allow more concurrency.

Read-write locks distinguish between reads (which can happen concurrently without problem) and writes (which require exclusive access). This allows multiple concurrent readers with exclusive writers, which dramatically improves throughput in read-heavy systems.

Atomic operations are operations that execute as a single, indivisible unit. Compare-and-swap (CAS) is the fundamental atomic operation that most lock-free data structures are built on. Understanding CAS means understanding how Java's AtomicInteger, AtomicReference, and concurrent collections work.

Optimistic concurrency control — instead of locking before modification, you read the current state, make changes, and then check before writing whether the state changed since you read. If it did, retry. This works well when conflicts are rare (most read-heavy systems) and is terrible when conflicts are frequent. Database row versioning is optimistic concurrency in practice.

**The database concurrency implications:**

Transactions at different isolation levels use different concurrency mechanisms. Read Committed uses MVCC to give each transaction a consistent snapshot without locking reads. Serializable uses predicate locks to prevent phantom reads. Understanding these mechanisms helps you choose the right isolation level and predict performance under concurrent load.

**Distributed concurrency — the harder problem:**

Distributed locks (implemented via Redis or ZooKeeper) allow coordination across multiple processes on multiple machines. They introduce network latency and failure modes that don't exist with in-process locks. The Redis SETNX command (set if not exists) is the basis of distributed locking. Understanding its edge cases — what happens if the lock holder crashes before releasing? — teaches you why distributed concurrency is fundamentally harder than single-process concurrency.

**Suggested exercises:**

Write a program that deliberately causes a race condition. Then fix it with different mechanisms — synchronized, ReentrantLock, AtomicInteger — and measure the performance difference. Then implement a distributed lock with Redis and deliberately test the failure case where the lock holder crashes.

---

### 1.5 Distributed Systems Basics

This is the conceptual foundation that makes everything in HLD make sense. Understanding distributed systems means understanding why the solutions we use are the way they are.

**Why distribution exists:**

No single machine can handle the storage or processing requirements of large systems. Distribution is the solution. But distribution introduces problems that don't exist when everything is on one machine: network failures, partial failures, latency, consistency across replicas, and coordination overhead.

**The CAP Theorem:**

In the presence of a network partition (and partitions always eventually happen), a distributed system must choose between consistency (every read receives the most recent write) and availability (every request receives a response, even if it might be stale).

The critical nuance most people miss: you don't make this choice once at system design time. You make it for each operation, and different parts of your system can make different choices. Your user authentication might choose consistency (a stale session token is a security risk) while your product catalog might choose availability (a slightly outdated product description is acceptable).

**The PACELC extension:**

CAP only talks about behavior during a partition. PACELC extends it: even in the absence of a partition (P), you must choose between latency (L) and consistency (C). A system that chooses consistency during both partitions and normal operation will always be slower because it must coordinate writes across replicas. This is why "choose between CA, CP, and AP" is an oversimplification — you also choose between low-latency eventual consistency and higher-latency strong consistency in normal operation.

**Consistency models — a spectrum:**

Strong consistency means every read sees the most recent write. The database is always in a consistent state from the reader's perspective. This requires coordination on every write, which creates latency and reduces availability. It's what traditional SQL databases provide within a single node.

Eventual consistency means given enough time without new writes, all replicas will converge to the same value. Reads might see stale data in the interim. This allows writes to be fast and available because you don't wait for all replicas to acknowledge before returning success.

Read-your-own-writes consistency means you always see the results of your own writes, even if other users might not see them immediately. This is the minimum acceptable consistency for most user-facing applications — it's jarring when you post something and immediately refresh and it's gone.

Causal consistency means operations that are causally related (A happened before B) are seen in that order. Operations that are causally unrelated might be seen in any order. This is stronger than eventual consistency but weaker than strong consistency.

**Replication:**

Data replication means storing copies of data on multiple machines. This provides fault tolerance (if one machine fails, others have the data), read scalability (reads can be distributed across replicas), and geographic distribution (replicas closer to users reduce latency).

Single-leader replication has one node that accepts writes and replicates them to followers. This is simple but creates a single point of failure for writes and a bottleneck for write throughput.

Multi-leader replication allows multiple nodes to accept writes. This solves the write bottleneck and provides geographic write locality. It introduces write conflict resolution — what happens when two leaders accept conflicting writes? The answer (last-write-wins, merge, user resolution) is a business decision, not a technical one.

Leaderless replication (used by Cassandra) allows any node to accept writes. The write is sent to multiple nodes simultaneously and is considered successful when a quorum of nodes acknowledge. This provides maximum write availability but requires careful quorum configuration to ensure consistency.

**Consensus:**

When multiple nodes must agree on a value (who is the current leader? which value was committed?), you need consensus. Consensus algorithms like Paxos and Raft implement this agreement correctly even when some nodes fail or messages are delayed. Raft is the more understandable of the two — understanding Raft means understanding how Kafka's controller election, etcd, and many distributed databases maintain consistency.

**Suggested exercises:**

Set up a PostgreSQL primary with one replica. Write a script that writes to the primary and immediately reads from the replica. Observe replication lag. Then simulate a replica failure and observe the behavior. Then simulate a primary failure and practice promoting a replica to primary.

---

## Phase 2: Core System Design Concepts

These are the building blocks you assemble when designing large systems. Know each one deeply — not just what it is but when to use it, when not to, and what it costs.

---

### 2.1 Load Balancing

Load balancing is distributing incoming traffic across multiple servers. It solves two problems: scalability (one server can only handle so much) and availability (if one server fails, others continue serving traffic).

**Algorithms and when each is appropriate:**

Round robin distributes requests sequentially across servers. It's simple and works well when all requests have similar resource requirements and all servers have similar capacity.

Weighted round robin assigns different weights to servers, routing proportionally more traffic to higher-capacity servers. Use this when your server fleet is heterogeneous.

Least connections routes each new request to the server with the fewest active connections. This is better than round robin when requests have wildly varying processing times — a long-running request shouldn't be counted the same as a fast one.

IP hash routes requests from the same client IP to the same server consistently. This is useful for stateful applications that store session state on the server, though modern applications should avoid server-side session state for exactly this reason.

Resource-based (adaptive) load balancing routes based on actual server resource utilization — CPU, memory, request queue length. This is the most sophisticated and most accurate but requires agents on each server reporting metrics.

**Layer 4 vs Layer 7:**

A Layer 4 load balancer operates at the TCP level — it distributes connections without understanding what's inside them. It's extremely fast but can only route based on IP address and port.

A Layer 7 load balancer understands HTTP and can route based on URL path, headers, cookies, and request body. It's slower than L4 but enables sophisticated routing — sending API requests to one server farm, static assets to another, and websocket connections to a third.

**Health checks:**

A load balancer continuously checks whether backend servers are healthy. When a server fails its health check, the load balancer stops routing traffic to it. This is how load balancers provide high availability. Health checks can be active (the load balancer probes each server) or passive (the load balancer monitors response codes from real traffic).

**Session persistence and why to avoid it:**

Routing the same user to the same server every time (sticky sessions) maintains session state without a shared session store. But it creates uneven load distribution and means user sessions are lost when that server goes down. The better solution is externalizing session state to Redis, making any server equally capable of handling any user.

---

### 2.2 Horizontal vs Vertical Scaling

**Vertical scaling** means making individual servers more powerful — more CPU cores, more RAM, faster storage. It's simple (no code changes required, no distribution problems) but has a hard ceiling (the most powerful single machine available) and creates a single point of failure. It's the right first response to growing load when the current architecture isn't the bottleneck.

**Horizontal scaling** means adding more servers. It has no hard ceiling — you can add servers indefinitely. But it requires your application to be stateless (or to externalize state), your data to be distributable, and your architecture to handle the coordination between servers. It introduces all the distributed systems problems from Phase 1.

The decision between them is not ideological. It's economic and technical. Vertical scaling is simpler and sufficient until you hit either the ceiling or the cost curve where horizontal becomes cheaper. Most systems should vertically scale before introducing the complexity of horizontal scaling.

**What doesn't scale horizontally without work:**

Stateful application servers (with in-memory session state) require sticky sessions or session externalization. Databases require sharding or read replicas. File storage requires distributed storage. Long-running background jobs require distributed task queues. Understanding what requires work to scale horizontally is the architect's job.

---

### 2.3 Caching Strategies at System Level

Beyond the basics from Phase 1, system-level caching decisions include:

Where to cache: at the client (browser cache), at the edge (CDN), at the application tier (in-process or distributed cache), or at the database tier (query cache). Each layer has different performance characteristics, different consistency controls, and different failure modes.

What to cache: hot data (frequently accessed, slow to generate), computed results (expensive aggregations or calculations), and static assets (images, JavaScript, CSS). Not everything benefits from caching — data that is rarely accessed or trivially fast to generate doesn't benefit enough to justify the consistency complexity.

Cache hit ratio as the key metric: if your cache has a 99% hit rate, it's doing its job. If it has a 30% hit rate, you've introduced consistency complexity without meaningful performance benefit. Monitor this.

Cache warm-up strategies: cold starts (cache is empty on startup) can cause thundering herds. Strategies to address this include pre-warming from recent logs, gradual rollouts that keep old servers running until the cache warms, and background prefetching.

---

### 2.4 Messaging Queues and Async Processing

Message queues decouple producers of work from consumers of that work. The producer sends a message to the queue and immediately continues. The consumer reads from the queue and processes it independently. This decoupling provides several benefits.

**Why message queues:**

Load leveling — a system that processes 100 requests/second average but 10,000 requests/second at peak can handle the peak by queuing the excess and processing it at a sustainable rate. Without a queue, the peak overwhelms the system. With a queue, the system processes at its maximum sustainable rate and the queue absorbs the excess.

Fault tolerance — if the consumer fails, messages remain in the queue and can be reprocessed when the consumer recovers. Without a queue, a synchronous call to a failing service results in a failed request.

Decoupling — producers and consumers can evolve independently. Adding a new consumer (adding a new behavior when something happens) doesn't require modifying the producer.

**Kafka vs traditional message queues (RabbitMQ, SQS):**

Traditional message queues (RabbitMQ, SQS) are designed for task queues — each message is consumed by exactly one consumer and deleted after acknowledgment. They're great for work distribution but not for event streaming.

Kafka is a distributed log, not a traditional queue. Messages are retained for a configurable period (days or weeks) and can be consumed by multiple consumer groups independently. Each consumer group tracks its own position (offset) in the log. This enables use cases that traditional queues can't support: replaying events, adding new consumers to process historical events, and multiple independent downstream systems all reading the same event stream.

Use RabbitMQ/SQS for task distribution. Use Kafka for event streaming, audit logs, and event-driven architectures where multiple consumers need the same events.

**Message delivery guarantees:**

At-most-once: messages might be lost but never duplicated. Simple but unacceptable for most business events.

At-least-once: messages might be duplicated but never lost. The consumer must handle duplicates (be idempotent). This is the default for most systems because it's simpler to handle duplicates than to lose events.

Exactly-once: messages are delivered and processed exactly once. This requires coordination between the producer and consumer and is expensive to implement. Kafka supports it with transactions but it has significant performance overhead. Most systems achieve this by combining at-least-once delivery with idempotent consumers.

---

### 2.5 Event-Driven Architecture

Event-driven architecture (EDA) is a pattern where components communicate by producing and consuming events rather than calling each other directly. When something happens in the system (an order is placed, a payment is processed, a user registers), an event is published to a message bus and all interested parties consume it.

**Benefits:**

Loose coupling — the order service doesn't know or care what the inventory service, notification service, and analytics service do when an order is placed. It just publishes the event. New consumers can be added without modifying producers.

Scalability — consumers can scale independently based on their own load. The order service's traffic spike doesn't create a spike in the notification service.

Auditability — the event log is a complete history of everything that happened in the system. This is invaluable for debugging and compliance.

**Challenges:**

Eventual consistency — because processing is asynchronous, different parts of the system are in different states at any moment. This is often acceptable but must be designed for explicitly.

Debugging complexity — tracing a business transaction across multiple services and events is harder than tracing a synchronous call chain. Distributed tracing (Zipkin, Jaeger) is essential.

Event schema evolution — when the structure of an event changes, old consumers must still be able to process old events and handle new events. Schema registries and Avro or Protobuf with backward-compatible evolution are the solutions.

Choreography vs orchestration — in choreography, each service reacts to events and produces its own events; there's no central coordinator. In orchestration, a central service (saga orchestrator) explicitly tells each service what to do. Choreography is more decoupled; orchestration is easier to understand and debug.

---

### 2.6 Microservices vs Monolith

This is the decision that creates the most cargo-culting in software engineering. Teams choose microservices because they're "modern" without understanding what they're trading for what.

**The monolith's advantages:**

Simple deployment — one deployable artifact. Simple debugging — everything is in one process, one log stream, one trace. Simple development — no distributed system complexity. Simple data management — one database, real transactions, no eventual consistency.

The monolith is not a legacy architecture. It's often the correct architecture, especially for teams under 50 engineers and products with fewer than a million daily active users. Many successful products run as monoliths at impressive scale. Shopify was a Rails monolith handling significant scale before they started selectively extracting services.

**When microservices make sense:**

Independent deployment requirements — when different parts of the system need to be deployed at different cadences by different teams. When the deployment coordination cost of a monolith exceeds the operational cost of microservices.

Independent scaling requirements — when different parts of the system have dramatically different resource needs. The image processing service might need GPU instances while the user profile service needs memory.

Team independence — Conway's Law says your system architecture will mirror your communication structure. If you have multiple teams who need to work independently, microservices allow them to deploy independently. But if you're a small team, microservices create coordination overhead without this benefit.

**The microservices tax:**

Every microservice is a distributed system problem. Network calls fail. Services go down. Data must be eventually consistent across services. Debugging requires distributed tracing. Testing requires service mocks or a full service mesh. Operations requires container orchestration (Kubernetes). Each of these is a real cost that must be justified by real benefit.

The architecture decision should be: what is the minimum distribution that achieves our actual requirements? Start as a modular monolith. Extract services when you can measure a specific benefit that justifies the cost.

---

### 2.7 API Design

APIs are contracts between systems. Changing an API breaks its consumers. Designing APIs well means designing them to evolve.

**REST principles:**

Resources, not actions. URLs identify nouns (users, orders, products), not verbs (getUser, createOrder). HTTP methods encode the action (GET, POST, PUT, PATCH, DELETE).

Statelessness — each request contains all information needed to process it. The server doesn't store client state between requests. This is what makes REST APIs horizontally scalable.

Versioning — APIs need to evolve. The options are URL versioning (/v1/users vs /v2/users), header versioning (Accept: application/vnd.company.v1+json), and query parameter versioning (?version=1). URL versioning is the most explicit and cacheable. Always version your APIs from day one — retrofitting versioning is painful.

Pagination — APIs that return collections must paginate. Cursor-based pagination (return a cursor pointing to the last item, use it to get the next page) is more robust than offset-based pagination for live data because it isn't affected by insertions or deletions.

Error handling — errors should be informative and consistent. An error response should include a machine-readable error code, a human-readable message, and enough context to diagnose the problem. HTTP status codes should be used correctly — 404 for not found, 422 for validation errors, 500 for server errors, 429 for rate limiting.

**GraphQL vs REST:**

GraphQL lets clients specify exactly what data they want. This eliminates over-fetching (REST returns all fields even if you want two) and under-fetching (REST requires multiple requests for related data that GraphQL gets in one). It's excellent for complex, hierarchical data with diverse clients. It's unnecessary complexity for simple CRUD APIs with a single client.

**gRPC:**

gRPC uses Protocol Buffers for serialization (much more efficient than JSON) and HTTP/2 for transport (enabling bidirectional streaming). It's ideal for internal service-to-service communication where performance matters. The contract-first approach (defining your API in a .proto file before implementing it) ensures both sides agree on the schema. It's not appropriate for public APIs because the tooling is less accessible than REST.

---

### 2.8 Rate Limiting

Rate limiting controls how many requests a client can make in a given time window. It protects services from overload, prevents abuse, and enables fair resource sharing.

**Algorithms:**

Fixed window counts requests in fixed time windows (0-60 seconds, 60-120 seconds). Simple to implement but has an edge case: a burst of requests at the end of one window and beginning of the next can exceed the intended limit.

Sliding window log maintains a log of request timestamps. When a request comes in, remove all timestamps older than the window, add the new timestamp, and count. If the count exceeds the limit, reject. Accurate but memory-intensive for high-traffic systems.

Sliding window counter approximates the sliding window using a fixed window count plus a weighted count from the previous window. Accurate enough for most purposes and much more memory-efficient.

Token bucket allows a burst of requests up to the bucket size, then throttles to the refill rate. The burst allowance is important — users naturally interact in bursts, and rate limiting that doesn't allow any bursting feels broken.

Leaky bucket processes requests at a fixed output rate regardless of input rate. Requests that can't be immediately processed queue up. This ensures a smooth output rate but can create long queues under sustained high load.

**Implementation:**

Redis is the standard implementation for distributed rate limiting. The key is typically a combination of the user identifier and the time window. Redis's atomic operations (INCR with EXPIRE, or Lua scripts for complex algorithms) make it naturally suited for this.

Rate limit headers should inform the client of their current limits (X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset) so they can self-regulate.

---

### 2.9 CDN (Content Delivery Networks)

A CDN is a geographically distributed network of servers that cache content close to users. Requesting an image from a CDN node 10ms away is dramatically faster than requesting it from your origin server 150ms away.

**What to put on a CDN:**

Static assets — images, CSS, JavaScript, fonts. These change rarely and can be cached aggressively. The cache invalidation strategy is deploying assets with content-hash filenames (bundle.a3f4c.js) so new versions have new URLs and don't conflict with cached old versions.

Semi-dynamic content — pages that are generated from data but change infrequently. A product page that changes when the product information changes can be cached for 60 seconds, significantly reducing origin load.

APIs are increasingly served through CDNs — not for caching (most API responses are user-specific) but for DDoS protection, SSL termination at the edge, and geographic routing.

**Cache invalidation at the CDN level:**

CDN cache invalidation is slow and expensive (you're purging caches across dozens of geographic nodes). The correct strategy for assets is content-hash filenames so you never need to invalidate — you just publish a new version at a new URL. For semi-dynamic content, use short TTLs (30-300 seconds) rather than relying on invalidation.

**CDN and origin protection:**

Your origin server should only receive traffic from the CDN, not directly from users. This protects it from DDoS attacks (the CDN absorbs the volume) and means your origin is sized for CDN-to-origin traffic (much smaller than user-to-CDN traffic).

---

### 2.10 Storage Design

Understanding storage options and their tradeoffs is foundational to system design.

**Object storage (S3, GCS, Azure Blob):**

Object storage is for large, unstructured binary data — images, videos, documents, backups. It's optimized for durability (eleven nines), not performance. Read and write throughput is high but latency is high (tens to hundreds of milliseconds). It's not appropriate for data that must be accessed with low latency — that's what databases and caches are for. It is appropriate for almost everything else. User uploads, application assets, log archives, database backups — all go to object storage.

**Block storage (EBS, persistent disks):**

Block storage is what your database uses. It presents as a raw disk — the filesystem is layered on top. It provides low-latency, high-throughput access to data. It's attached to a single server at a time (with some exceptions). It's what you use when a database's data must be persistent and survive server restarts.

**File storage (NFS, EFS):**

Shared file storage is a network filesystem that multiple servers can mount simultaneously. It's useful when multiple servers need to read and write the same files. It's slower than local block storage because every operation goes over the network. Use it when you genuinely need shared file access; prefer object storage when you just need to store files durably.

**Databases as storage:**

The choice of database (SQL vs various NoSQL) is itself a storage design decision. The key is understanding the access patterns and consistency requirements as covered in section 1.2.

---

## Phase 3: Design Patterns for Low Level Design

Design patterns are named solutions to commonly recurring problems. They matter not because you should use them everywhere, but because they give you a shared vocabulary and a toolkit of proven solutions.

The most important thing about design patterns is understanding the problem each one solves — not memorizing the implementation. If you know the problem, you can derive the pattern. If you only know the implementation, you'll apply it in the wrong places.

---

### 3.1 Creational Patterns

Creational patterns are about how objects are created. They separate the creation logic from the usage, making systems more flexible and easier to change.

**Singleton**

The problem: you need exactly one instance of a class and a global point of access to it. Database connection pools, configuration managers, and logger instances are classic examples.

The solution: a class that controls its own instantiation, ensuring only one instance is ever created.

Where it's misused: Singleton is frequently overused as a glorified global variable, creating hidden dependencies and making testing difficult. Use it only when the single-instance constraint is genuinely important, not just convenient.

The thread-safe implementation with double-checked locking in Java requires the instance variable to be declared volatile — a subtlety that many developers miss.

**Factory Method**

The problem: you need to create objects but the exact class to create isn't known until runtime, or you want to delegate object creation to subclasses without specifying the exact class.

The solution: a method that returns an object of a certain type, but the actual class of the returned object is determined by subclasses or by a parameter.

Real system example: A notification system that creates SMS, email, or push notification objects based on user preference. The client code creates notifications without knowing which concrete class it's getting. Adding a new notification type (in-app notification) requires only adding a new class and a factory entry, not modifying any existing code.

**Abstract Factory**

The problem: you need to create families of related objects that must be used together, without specifying their concrete classes.

The solution: a factory that creates families of objects. The client uses the factory interface, and swapping the concrete factory switches the entire family of objects.

Real system example: A payment system that supports Stripe and Razorpay. Each has its own concrete factory that creates the corresponding PaymentIntent, Customer, and WebhookHandler objects. Switching payment providers means switching factories, not rewriting business logic.

**Builder**

The problem: constructing a complex object step-by-step where different combinations of steps produce different objects, and not all steps are always required.

The solution: a Builder class that accumulates configuration through method calls and creates the final object when build() is called.

Real system example: Constructing an HTTP request with optional headers, query parameters, body, timeout, and authentication. A builder makes this far more readable than a constructor with a dozen parameters.

This is one of the most practically useful patterns in Java backend development.

**Prototype**

The problem: creating new objects by copying an existing object when construction is expensive.

The solution: a clone() method that creates a copy of the object.

Real system example: Caching expensive-to-create objects (parsing a large configuration file, establishing a complex connection) and cloning them rather than recreating from scratch.

---

### 3.2 Structural Patterns

Structural patterns are about how classes and objects are composed to form larger structures.

**Adapter**

The problem: you need to use a class with an incompatible interface. You can't modify the class (it's from a library, or changing it would break other things).

The solution: a wrapper class that translates between the interface you need and the interface the class provides.

Real system example: Integrating a third-party payment gateway with an incompatible API. The adapter wraps the third-party client and exposes the interface your application expects. When you switch payment providers, you write a new adapter — the business logic is untouched.

**Decorator**

The problem: you need to add behavior to individual objects dynamically without affecting other objects of the same class, and without using inheritance (which applies behavior to all instances).

The solution: wrap the object in a decorator that adds behavior, then wraps that in another decorator if needed. Each decorator delegates to the wrapped object while adding its own behavior.

Real system example: Adding authentication, rate limiting, logging, and caching to an API endpoint handler. Each is a decorator that wraps the core handler. They can be composed in any order and combination. This is exactly how HTTP middleware chains work in frameworks like Spring.

Java's IO classes (BufferedReader wrapping InputStreamReader wrapping FileInputStream) are the canonical example.

**Facade**

The problem: a complex subsystem with many classes and a complicated interface. You need to provide a simpler interface to the most commonly used features.

The solution: a Facade class that provides a simplified interface to the subsystem.

Real system example: A HomeTheaterFacade that provides a watchMovie() method, which internally coordinates the projector, speaker system, streaming client, and lighting. The client calls one method instead of orchestrating ten.

In backend systems, service classes often function as facades — coordinating multiple repositories, caches, and external clients behind a simple interface.

**Proxy**

The problem: you need to control access to an object — adding security checks, lazy initialization, caching, logging, or remote access.

The solution: a proxy object with the same interface as the real object. Clients use the proxy; the proxy delegates to the real object while adding its own logic.

Real system example: Spring AOP uses dynamic proxies to add transaction management, security, and caching around method calls without modifying the methods themselves. When you annotate a method with @Transactional, Spring wraps it in a proxy that begins a transaction before the call and commits or rolls back after.

**Composite**

The problem: you need to treat individual objects and compositions of objects uniformly.

The solution: define a component interface, leaf objects implement it directly, composite objects implement it by delegating to their children.

Real system example: A file system (files and directories), an organizational chart (individual contributors and managers with reports), a UI component tree (leaf components and container components). Operations like size() or render() work the same way on leaves and composites.

---

### 3.3 Behavioral Patterns

Behavioral patterns are about communication between objects and responsibility assignment.

**Observer (Event Listener)**

The problem: when one object changes state, other objects need to be notified and updated automatically, without tight coupling between them.

The solution: the subject maintains a list of observers and notifies them when its state changes. Observers register themselves with the subject.

Real system example: An e-commerce order changes state (placed → payment confirmed → shipped → delivered). Every state change triggers notifications to the customer (email, SMS), updates inventory, creates fulfillment tasks, and records analytics events. Each of these is an observer. Adding a new reaction (loyalty points) means adding a new observer, not modifying the order class.

This is the foundation of event-driven architecture at the code level.

**Strategy**

The problem: a class has multiple behaviors that vary by context, and you want to switch between them at runtime.

The solution: extract each behavior into a separate strategy class with a common interface. The client holds a reference to a strategy and delegates to it.

Real system example: A payment processor that supports multiple payment methods (credit card, UPI, net banking, wallet). Each payment method is a strategy. The client code that processes payment is identical regardless of which strategy is in use. Adding a new payment method means adding a new strategy, not modifying the processor.

This pattern is how you achieve the Open/Closed Principle in practice — open for extension (add a new strategy), closed for modification (don't change existing code).

**Command**

The problem: you want to encapsulate a request as an object, enabling queuing, logging, undoing, and retrying of requests.

The solution: a Command interface with an execute() method. Each concrete command encapsulates a receiver and parameters. An invoker calls execute() without knowing what it does.

Real system example: A task queue that persists commands to a database, retries them on failure, and can replay them for debugging. Each task is a command object with execute(). The queue doesn't know what each command does — it just calls execute() and handles success or failure.

Also the basis of undo/redo in any editor — each action is a command with execute() and undo() methods, stored in a history stack.

**Chain of Responsibility**

The problem: you need to pass a request through a chain of handlers, where each handler either processes the request or passes it to the next handler.

The solution: a chain of handler objects, each with a reference to the next. Each handler decides whether to handle the request or delegate to the next.

Real system example: HTTP request processing in Spring — authentication filter, authorization filter, rate limiting filter, logging filter, and finally the controller. Each filter either handles the request (returns a response) or passes it to the next filter.

**Template Method**

The problem: several algorithms share the same overall structure but differ in specific steps. You don't want to duplicate the shared structure.

The solution: a base class defines the algorithm's skeleton (the template method), calling abstract methods for the variable steps. Subclasses implement the variable steps.

Real system example: A report generator that always: fetches data, formats it, adds headers, and exports. The steps are the same but the data source, formatting, and export format vary. The template method defines the steps; subclasses implement each step for specific report types.

**State**

The problem: an object's behavior changes dramatically based on its internal state. Lots of conditional logic based on current state.

The solution: extract each state into a separate state class that implements a common interface. The context delegates to its current state object.

Real system example: An order with states: Pending, PaymentConfirmed, Processing, Shipped, Delivered, Cancelled. Each state determines which operations are valid (you can't ship a cancelled order) and what happens on each event. Without the State pattern, this is a massive switch statement that grows with every new state. With it, each state is self-contained.

**Iterator**

The problem: you need to traverse a collection without exposing its underlying implementation.

The solution: a separate iterator object that knows how to traverse the collection.

Real system example: Paginating through database results without loading everything into memory. The iterator knows how to fetch the next page when the current one is exhausted. Java's for-each loop uses this pattern — any class implementing Iterable works.

**Mediator**

The problem: many objects communicate in complex ways, creating a tangled dependency web. Every object knows about many others.

The solution: a mediator object that all others communicate through. Objects know about the mediator; they don't know about each other.

Real system example: An air traffic control system where planes communicate through the control tower rather than directly with each other. In software: a chat room (mediator) where users send messages to the room, which distributes them to all participants, rather than users knowing about each other directly.

---

### SOLID Principles — The Foundation of LLD

Design patterns implement these principles. Understanding the principles explains why the patterns exist.

**Single Responsibility Principle:** A class should have one reason to change. A class that handles user authentication, profile management, and email sending will change when any of those three things changes — creating unrelated changes in the same place. Split it.

**Open/Closed Principle:** Open for extension, closed for modification. Adding new behavior should require adding new code, not modifying existing code. Strategy pattern is the canonical implementation.

**Liskov Substitution Principle:** Subtypes must be substitutable for their base types. If code works correctly with the base class, it should work correctly with any subclass. Violations typically mean the inheritance relationship is wrong.

**Interface Segregation Principle:** Clients should not be forced to depend on interfaces they don't use. A large interface that few implementors fully satisfy should be split into smaller, more specific interfaces.

**Dependency Inversion Principle:** Depend on abstractions, not concretions. High-level modules should not depend on low-level modules — both should depend on abstractions. This is why you inject interfaces rather than concrete implementations in Spring Boot — it makes the system flexible and testable.

---

## Phase 4: Real-World System Design Case Studies

For each system, apply the seven-step design process from Part 5. The goal is not to memorize these designs but to practice the reasoning process until it becomes automatic.

---

### Case Study 1: URL Shortener (Start Here — Foundational)

**Why start here:** URL shortener is the simplest system that still involves all the core concepts — database design, caching, scaling, and a non-trivial algorithmic decision. It's the perfect first practice system.

**Requirements clarification:**

The functional requirements are: given a long URL, generate a short URL. Given a short URL, redirect to the original long URL. Optional: custom aliases, expiration dates, analytics.

The non-functional requirements drive all the interesting design decisions: how many URLs are created per day (write load)? How many redirects per day (read load)? What's acceptable latency for the redirect (speed is critical — users notice slow redirects)? How long are shortened URLs valid?

Assume: 100 million URLs created per day, 10 billion redirects per day (100:1 read-write ratio). This is extreme read dominance — the design must optimize for reads.

**High-level architecture:**

There are two critical services: a creation service that takes a long URL and generates a short code, and a redirect service that takes a short code and returns the original URL. These should be separated because they have completely different load patterns.

The database stores the mapping: short code → long URL. A relational database works perfectly here — the data model is simple, and the access pattern is a point lookup by primary key.

**The key algorithmic decision:**

How do you generate the short code? Three approaches:

Random generation creates a random 6-character alphanumeric string. Simple but requires checking for collisions (two URLs could get the same code). At high scale, collisions become frequent.

Hash-based generation takes the MD5 or SHA256 hash of the long URL and uses the first 6 characters. Deterministic (same URL always gets the same code, enabling deduplication) but still has collision risk.

Auto-increment with Base62 encoding converts an auto-incrementing database ID to Base62 (A-Z, a-z, 0-9). 6 Base62 characters gives 62^6 = 56 billion unique codes. No collisions. The database ID is the source of truth. This is the cleanest approach.

**Scaling challenges and solutions:**

The redirect service handles 10 billion requests per day (about 115,000 per second). Hitting the database on every redirect would require an enormous database cluster. The solution is aggressive caching — most redirects are for popular URLs, which follow a power law distribution (a few URLs account for most traffic). Redis caching of the short code → long URL mapping, with a 24-hour TTL and LRU eviction, will achieve 95%+ hit rates for popular URLs.

The creation service's auto-increment ID generation becomes a bottleneck at scale because you need to serialize ID generation to ensure uniqueness. Solutions include: a dedicated counter service (using Redis INCR for atomic increment), pre-allocating ranges of IDs to application servers (each server takes 1,000 IDs at a time and uses them locally), or using distributed ID generation (Twitter Snowflake pattern, which generates sortable unique IDs without coordination).

**Tradeoffs:**

Base62 with auto-increment vs random generation: Base62 gives no collisions and natural ordering but exposes business information (sequential IDs reveal how many URLs have been created). Random generation hides this but requires collision detection.

Redirect architecture: returning a 301 (permanent redirect) means browsers cache it and future requests don't hit your service. This reduces load but means you can never change where a URL redirects (browsers never ask you again). 302 (temporary redirect) means every redirect goes through your service, enabling analytics and future changes, at the cost of higher load.

---

### Case Study 2: Notification System

**Requirements clarification:**

Types of notifications: email, SMS, push notifications, in-app notifications. Events that trigger notifications: various business events (order placed, payment received, friend request). Volume: hundreds of millions of notifications per day. Latency: most notifications can tolerate seconds of delay; some (OTP, payment confirmation) require near-instant delivery. Reliability: notifications must not be lost, but occasionally duplicate notifications are acceptable.

**High-level architecture:**

The system has three main components: the event ingestion layer that receives events from other services (order service, payment service, user service), the notification service that transforms events into notifications and determines which users to notify and how, and the delivery layer that sends notifications through the appropriate channels (email via SendGrid, SMS via Twilio, push via APNs/FCM).

**Key design decisions:**

Use a message queue (Kafka) between the event ingestion layer and the notification service, and between the notification service and the delivery layer. This provides: fault tolerance (messages aren't lost if a service fails), load leveling (high-volume events don't overwhelm the delivery layer), and at-least-once delivery guarantee.

Separate notification workers by channel. Email workers, SMS workers, and push notification workers operate independently. SMS and email workers can be scaled independently when SMS volumes spike.

User notification preferences must be checked before sending. Storing preferences in a database with a cache layer (preferences change rarely) prevents sending unwanted notifications.

**Scaling challenges:**

At hundreds of millions of notifications per day, the delivery layer is the bottleneck — third-party providers have rate limits and SLAs. Solutions include multiple provider accounts with round-robin distribution, provider fallback when one is degraded, and careful rate limiting to avoid provider throttling.

Deduplication is essential at this scale — the at-least-once delivery guarantee means the same notification might be generated twice. A deduplication key (hash of the event ID + user ID + notification type) with a 24-hour TTL in Redis prevents duplicate sends.

**Tradeoffs:**

Priority queues — time-sensitive notifications (OTP for login) should not wait behind batch notification campaigns. Separate high-priority and low-priority queues ensure OTPs deliver in seconds even when a marketing campaign is sending millions of messages.

---

### Case Study 3: Ride-Sharing System (Uber/Ola)

**Requirements clarification:**

Core functionality: a rider requests a ride, the system matches them with a nearby driver, the driver accepts, both parties track location in real-time, the ride is completed and payment is processed.

Non-functional: location updates every 4 seconds from every active driver, matching must complete in under 5 seconds, the system handles hundreds of thousands of simultaneous rides.

**High-level architecture:**

Location service: receives GPS coordinates from all active drivers every 4 seconds. At 100,000 active drivers, this is 25,000 location updates per second — a significant write load. Uses a write-optimized data store. Critically, this data is highly transient (you need current location, not history) and geospatial (you need to find nearby drivers).

Matching service: when a rider requests a ride, it queries the location service for drivers within a geographic area, filters by availability and type, ranks by proximity and estimated arrival time, and sends a match request to the top candidate.

Trip management service: tracks the state of active trips (requested → driver assigned → driver en route → trip in progress → completed). Uses a state machine.

**The geospatial challenge:**

Finding all drivers within 2km of a rider's location is a geospatial query. Options include Redis GEOADD and GEORADIUS commands (Redis natively supports geospatial indexing), a quadtree data structure (hierarchically divides geographic area into quadrants for efficient range queries), or a geohash (encodes lat/long as a string where prefix similarity indicates geographic proximity, enabling standard database prefix queries).

Redis GEOADD with GEORADIUS is the practical choice — mature, performant, and operationally simple. Quadtrees are appropriate when Redis can't handle the volume.

**The matching challenge:**

The system must avoid "matching storms" where the same driver receives simultaneous requests from multiple riders. The solution is an optimistic lock on the driver's status — the matching service sets the driver status from "available" to "pending" atomically (using Redis SETNX) before sending the match request. If the atomic operation fails (another request already claimed the driver), it tries the next candidate.

**Real-time communication:**

Riders and drivers need real-time location updates during a trip. WebSocket connections (or long polling as a fallback) from mobile clients to a connection service. The connection service subscribes to location updates from Kafka and pushes them to connected clients. At scale, connection services form a cluster; a driver's location update is published to Kafka and all connection service instances that have interested clients receive it.

**Tradeoffs:**

Consistency vs availability in matching: the matching system must be correct (a driver can't be matched to two riders simultaneously). This requires coordination, which creates latency. The atomic Redis operation is the right balance — fast enough for user experience, correct enough to prevent double booking.

Location data consistency: with 25,000 updates per second, you cannot afford strong consistency on every update. The location store uses eventual consistency — a driver's location might be 4-8 seconds stale in the worst case. This is acceptable because at 60 km/h, 8 seconds of staleness means 130 meters of uncertainty, which doesn't significantly affect matching quality.

---

### Case Study 4: Video Streaming Platform (Netflix/YouTube/Hotstar)

**Requirements clarification:**

Upload: content creators or the platform itself uploads video. This is infrequent, can tolerate minutes of latency, but videos are large (1GB to hundreds of GB).

Playback: users watch video. Must be fast to start, smooth (no buffering), and adapt to available network bandwidth. This is the dominant use case — 99% of traffic.

Scale: billions of hours of video watched per day.

**High-level architecture:**

Video processing pipeline: raw uploaded video is stored in object storage, then processed by a video encoding cluster that transcodes it into multiple formats and resolutions (240p, 360p, 480p, 720p, 1080p, 4K) and multiple codecs (H.264, H.265, VP9). Output files are stored back in object storage. This is a batch processing pipeline — it's expensive computation but doesn't require real-time performance.

Metadata service: stores video metadata (title, description, duration, available resolutions, URL templates). This is a read-heavy workload with predictable access patterns — ideal for a relational database with aggressive caching.

Streaming delivery: this is where CDN is critical. Video files are distributed to CDN nodes globally. When a user plays a video, they're served from the nearest CDN node. Without CDN, video traffic would overwhelm origin servers and latency would make streaming impossible.

**Adaptive bitrate streaming (ABR):**

Instead of sending one fixed-quality video, the server sends a manifest file (HLS playlist or MPEG-DASH manifest) that lists all available quality levels. The client's video player monitors available bandwidth and buffer fullness, and dynamically requests segments at the appropriate quality level. When bandwidth drops, it steps down to a lower quality. When bandwidth recovers, it steps up.

This is why Netflix starts at low quality and improves, and why it rarely buffers — the ABR algorithm keeps quality just below available bandwidth.

**The CDN is the entire playback architecture:**

At billions of hours watched per day, origin servers only exist to handle CDN cache misses. Popular content (new releases) lives in CDN edge caches and is served without ever touching origin. Long-tail content (obscure documentaries) might hit origin, but that traffic is low volume.

CDN performance is the primary determinant of playback quality. Netflix and YouTube operate their own CDN infrastructure (Netflix Open Connect) rather than relying entirely on commercial CDNs, because at their scale the economics favor ownership and they need the control.

**Tradeoffs:**

Storage cost vs quality: storing a 2-hour movie at twelve quality levels in three codecs requires significant storage. The tradeoff is clear — storage is cheap, but buffering is catastrophic for user experience.

Processing latency vs availability: after upload, video must be processed before it's available. Processing a full movie might take hours. Live streaming requires near-real-time processing (latency of seconds). These are completely different architectures — live streaming can't use the same batch pipeline as on-demand.

---

### Case Study 5: Social Media Feed (Twitter/Instagram)

**Requirements clarification:**

Publishing: users create posts (tweets, photos, short videos). Reading: users see a feed of posts from accounts they follow, ranked by relevance or recency.

Scale: millions of posts per day. Billions of feed views per day. A celebrity with 10 million followers creates a single post that needs to appear in 10 million feeds.

**The fundamental challenge — fan-out:**

When a user with 10 million followers posts, that post must appear in 10 million feeds. Two approaches:

Fan-out on write (push model): when someone posts, immediately write that post to every follower's feed. Reading a feed is fast (your feed is pre-computed). Writing is slow and resource-intensive for users with many followers. A celebrity's post triggers 10 million writes.

Fan-out on read (pull model): when someone reads their feed, query all accounts they follow and merge their recent posts. Writing is instant. Reading is slow because you're aggregating from many sources.

**The hybrid solution:**

Fan-out on write for regular users (fewer than some follower threshold, say 1 million). Fan-out on read for celebrities (above the threshold). Regular users' posts are pushed immediately. Celebrity posts are fetched on demand when someone loads their feed.

This combines the read performance of fan-out on write for the common case (most posts are from non-celebrities) with the write efficiency of fan-out on read for celebrities (where fan-out on write is prohibitively expensive).

**Feed storage:**

The pre-computed feed is stored in Redis as a sorted set (post ID as score for recency ordering, or a composite score for relevance). Reading a feed is one Redis ZRANGE operation — extremely fast.

The raw post content is stored in a separate database (Cassandra for high write throughput and horizontal scalability). The feed only stores post IDs — the rendering layer fetches post content by ID for the posts in the feed.

**Tradeoffs:**

Relevance vs recency: a chronological feed is easy to implement but surfaces less engaging content. A ranked feed improves engagement but requires a machine learning ranking model — significant added complexity and a potential point of failure.

Consistency: with fan-out on write, there's a window after posting where some followers see the post and others don't, depending on where in the fan-out process they are. This is acceptable eventual consistency.

---

### Case Study 6: Payment System

**Requirements clarification:**

Core flow: user initiates payment → validate payment method → deduct from source → credit to destination → notify both parties.

Critical non-functional requirements: correctness is paramount (money cannot be lost or created), atomicity (partial transfers are never acceptable), idempotency (retrying a payment must never result in double charge), auditability (every operation must be logged for compliance and debugging).

**The fundamental architecture constraint:**

Payment systems are one of the few places where strong consistency is non-negotiable. Using eventual consistency for a payment system means real money can be in an inconsistent state — unacceptable. The architecture must choose consistency over availability.

**Idempotency — the most critical concept:**

Every payment operation must be idempotent. If a client sends a payment request, the response is lost in transit, and the client retries — the payment must not be processed twice.

The implementation: every payment request includes a client-generated idempotency key (a UUID). The server stores processed idempotency keys with their results. When a request arrives with an existing idempotency key, the server returns the stored result without reprocessing. The idempotency key storage must be atomic with the payment processing (typically done in the same database transaction).

**Double-entry bookkeeping:**

Every financial system uses double-entry bookkeeping — every transaction has a debit entry and a credit entry, and the sum of all entries must equal zero. This provides automatic consistency checking (if debits don't equal credits, something is wrong) and a complete audit trail.

In a database: a transaction table with debit and credit entries. A balance is always computed from the transaction history, never stored directly. This prevents balance inconsistency from concurrent updates.

**The saga pattern for distributed payments:**

When a payment involves multiple services (payment processing, fraud detection, ledger update, notification), each is a separate step with its own transaction. A saga coordinates these steps with compensating transactions — if any step fails, the already-completed steps are reversed.

Example: payment authorization succeeds, ledger update fails. The compensating transaction reverses the authorization. The end state is consistent, but achieved through a series of coordinated steps rather than a single database transaction.

**Tradeoffs:**

Consistency vs performance: strong consistency (synchronous replication, serializable isolation) makes writes slower. But in a payment system, this is the correct tradeoff — correctness is worth latency.

Optimistic vs pessimistic locking: pessimistic locking (lock the account record before updating the balance) prevents concurrent modifications but reduces throughput. Optimistic locking (check that the version hasn't changed before committing) allows higher concurrency but requires retry logic on conflict. Pessimistic locking is typically correct for high-value operations; optimistic locking for high-throughput, low-conflict operations.

---

### Case Study 7: Search System (Google-scale concept, applied to product search)

**Requirements clarification:**

Input: a text query. Output: relevant documents (products, articles, users) ranked by relevance. Scale: millions of queries per day. Latency: under 100ms for query results. Content: potentially hundreds of millions of documents to search.

**Why traditional databases can't solve this:**

A SQL LIKE '%search term%' query does a full table scan. On 100 million products, this takes minutes, not milliseconds. You need a specialized data structure: an inverted index.

**The inverted index:**

An inverted index is the core data structure of any search engine. For each word in your corpus, store a list of all documents that contain that word. A query is processed by looking up each query term in the index, finding the intersection of document lists, and ranking the results.

Example: "running shoes" query → look up "running" (documents: [1, 5, 7, 23, ...]) → look up "shoes" (documents: [1, 3, 7, 8, ...]) → intersection ([1, 7, ...]) → rank by relevance.

Elasticsearch implements this — it's essentially a distributed inverted index with a REST API. For most product search use cases, Elasticsearch is the right choice. Understanding what it does under the hood is essential for using it effectively.

**Ranking:**

The raw intersection gives you documents that match — not documents that are most relevant. Ranking determines which matches are best.

TF-IDF (term frequency-inverse document frequency) is the classic relevance scoring algorithm. A term's relevance to a document increases with how frequently it appears in that document (TF) and decreases with how common it is across all documents (IDF). "The" appears everywhere and has low IDF — it contributes little to relevance. "Orthopedic" appears rarely and has high IDF — documents containing it are likely highly relevant to queries containing it.

Modern search adds signals beyond text: product rating, sales velocity, inventory availability, personalization (user purchase history), and business logic (promoted products, margin optimization).

**Scaling challenges:**

A single inverted index for 100 million documents is too large for one machine. Elasticsearch shards the index across multiple nodes. Queries are distributed to all shards simultaneously, and results are merged and re-ranked. This horizontal scaling of search is how Elasticsearch handles large corpora.

**Autocomplete:**

The search box suggests completions as the user types. This is a different data structure problem — the inverted index is designed for full-word queries. Autocomplete requires prefix queries: "run" → "running shoes", "running jacket", "running socks".

A trie (prefix tree) or a sorted index of popular queries enables prefix queries efficiently. Redis's sorted sets with lexicographic scoring support prefix queries and are a common implementation.

**Tradeoffs:**

Freshness vs performance: the inverted index is built from a snapshot of documents. Adding new documents or updating existing ones requires re-indexing. Real-time indexing (index as soon as a document changes) is complex and can impact query performance. Near-real-time indexing (Elasticsearch refreshes indexes every second by default) is the practical balance.

---

### Case Study 8: Messaging System (WhatsApp/Slack)

**Requirements clarification:**

One-to-one messaging and group messaging. Messages must be delivered reliably (at-least-once) and in order within a conversation. Delivery receipts (sent, delivered, read). Message history (searchable). Online presence (who is currently online).

**Real-time delivery:**

The client maintains a persistent WebSocket connection to a chat server. When user A sends a message to user B:

If B is online and connected to the same chat server as A, the message is delivered immediately over B's WebSocket. If B is connected to a different chat server, the message is published to a message bus (Kafka/Redis pub-sub), and B's server receives it and delivers it over B's WebSocket. If B is offline, the message is stored and delivered when B reconnects.

This combination of WebSocket for online delivery and store-and-forward for offline delivery is how every messaging system works.

**Message ordering:**

Within a conversation, messages must be delivered in the order sent. This is not trivially guaranteed in a distributed system — two messages sent milliseconds apart might arrive at the server in a different order.

The solution: each conversation has a sequence number. Messages are assigned sequence numbers when stored. Clients display messages in sequence number order and request any gaps they detect.

**Group messaging — the scaling challenge:**

In a one-to-one conversation, delivering a message requires finding two connections. In a group with 1000 members, delivering a message requires fan-out to 1000 connections across potentially hundreds of chat servers.

WhatsApp solves this by sending messages to a group message queue. Each group member's chat server polls the queue for their members. Slack uses a similar approach but with per-channel event streams.

**Message storage:**

Conversation history must be stored durably and be retrievable. Cassandra is a natural fit — its data model supports writing messages efficiently, the partition key is the conversation ID, and messages within a partition are sorted by timestamp. This makes loading recent messages extremely fast (one Cassandra partition read).

**Tradeoffs:**

End-to-end encryption complicates server-side search — if messages are encrypted client-side, the server cannot search them. WhatsApp offers E2E encryption and therefore no server-side message search. Slack prioritizes searchability and stores messages in a searchable form, though it encrypts data at rest and in transit.

---

## Phase 5: How to Approach a New System Design Problem

This is the framework that makes everything else portable. Learn this process and you can approach any system you've never seen before.

---

### The Seven-Step Framework

**Step 1: Understand the problem before proposing anything (5-10 minutes)**

This is the step most developers rush or skip. The problem statement is never complete. Always ask clarifying questions:

What are the core use cases? (You need to know what you're designing before designing it.) What scale is expected — users, requests per second, data volume? What are the consistency requirements? What are the latency requirements? What already exists that this must integrate with? What is the team size that will build and maintain this? What are the business priorities — is latency more important than cost? Is this a greenfield design or replacing an existing system?

The questions you ask reveal how you think. Good questions impress more than fast answers.

**Step 2: Define and constrain the problem**

After clarifying, explicitly state the scope. "For this design, I'll focus on: the core read path, the write path, and data storage. I'll defer: authentication, admin tools, and analytics. Does that scope make sense?"

This demonstrates architectural judgment — knowing what to include and what to defer. Real systems are scoped; infinite scope is a design anti-pattern.

**Step 3: Estimate the scale**

Back-of-envelope calculations drive every design decision. Work through them out loud.

Daily active users × requests per user per day = requests per day. Divide by 86,400 (seconds per day) = average requests per second. Peak is typically 2-5x average. Each request generates X database queries (calculate this). Each piece of user data is Y bytes (estimate realistically). At Z users, total data = X terabytes.

These estimates tell you whether you need one server or one thousand, whether you need caching or whether the database can handle the load, and whether a single database instance can hold all your data.

**Step 4: Design the data model**

Before drawing boxes, understand the data. What entities exist? What are their relationships? What are the access patterns?

The data model determines almost everything else — the database choice, the schema, the indexing strategy, and where the performance bottlenecks will be. Getting this wrong makes everything downstream harder.

**Step 5: Design the API / interface**

Define the contracts between the system and its clients before implementing internals. What requests come in? What responses go out? What are the error cases?

Good API design often reveals overlooked requirements ("what happens if the user isn't found?" "how does pagination work?") before you've committed to an implementation.

**Step 6: Design the high-level architecture**

Now draw the boxes. Start with the simplest architecture that satisfies the requirements. One server, one database. Trace a request through it. Understand why this works and where it fails at scale.

Then systematically identify failure points and bottlenecks and add complexity only to solve specific identified problems. "At 10,000 requests/second, the database becomes the bottleneck, so we add a read replica and cache." Not "let's add Redis because we should have caching."

**Step 7: Deep dive on the hardest part**

Every system has one or two genuinely hard problems. Identify them and dig in.

For a URL shortener: the ID generation strategy at scale. For a social feed: the fan-out problem for celebrities. For a ride-sharing system: the geospatial matching and real-time location handling. For a payment system: idempotency and consistency guarantees.

In an interview, your interviewer will guide you here. In a real design, you should identify these proactively.

**After the design: challenge yourself**

What happens if the most critical dependency fails? What happens at 10x the estimated load? What's the simplest change that would make this fail? What did I assume that might be wrong?

---

## Phase 6: Practice Roadmap

---

### Order of Study

Follow this sequence. Each phase builds on the previous.

**Month 1-2: Foundations**
Study the foundational topics from Phase 1 in this order: databases first (most impactful, most applicable to current work), then networking, then caching, then concurrency, then distributed systems. For each topic, study the concept, find it in your current work, and practice the suggested exercises.

**Month 3: Core concepts**
Work through Phase 2 topics in order. For each concept, implement a small example (implement a token bucket rate limiter, set up Kafka locally and build a producer/consumer, implement a simple consistent hash ring) before studying the theory of the next.

**Month 4: Design patterns**
Study LLD patterns from Phase 3. For each pattern, identify one place in your current codebase where it is or should be used. Refactor one system to apply a pattern you identified. This contextualizes the patterns.

**Month 5-8: Case studies**
One case study per week. For each: derive the design yourself (30-60 minutes before any reading), then read existing designs, then identify where your derivation differed and why.

**Month 9-12: Original designs**
Start designing systems you haven't studied. Apply the seven-step framework to novel problems. Get feedback from engineers more experienced than you.

---

### Exercises After Each Topic

**After databases:** Profile five queries in a real system. Find the slowest. Add an index. Measure the improvement. Document what you learned.

**After caching:** Implement a cache-aside layer in front of a database you control. Measure hit rate after 24 hours. Identify what to pre-warm for better cold-start performance.

**After distributed systems:** Set up a three-node Kafka cluster locally. Produce messages from one process and consume from three parallel consumers. Observe partition assignment and rebalancing.

**After each design pattern:** Find this pattern in a real codebase (Spring Framework source code is excellent for this — almost every pattern appears there). Read how it's implemented in production code.

**After each case study:** Write a one-page document titled "How I would build [system] with a team of three engineers and a $10,000/month infrastructure budget." The constraints force creative thinking about what matters.

---

### Increasing Difficulty

**Beginner systems (start here):**
URL shortener, rate limiter, key-value store, task scheduler, parking lot system, library management system.

**Intermediate systems:**
Notification system, social media feed (no recommendations), image storage service, leaderboard system, real-time collaborative document (simplified), search autocomplete.

**Advanced systems:**
Video streaming, ride-sharing, payment processing, distributed message queue, web crawler, recommendation engine, distributed cache.

**Expert systems (for senior/staff-level thinking):**
Global distributed database, multi-region active-active systems, distributed tracing infrastructure, large-scale ML feature store, real-time fraud detection at payment scale.

---

## Phase 7: Common Mistakes Beginners Make

**Jumping to solutions before understanding requirements.**
The most common and most damaging mistake. Designing a system without understanding the constraints produces a design for the wrong problem. The solution: spend 20-30% of your time on clarification.

**Treating microservices as the default architecture.**
Starting with microservices at the design stage creates enormous accidental complexity for small teams. The solution: start with a modular monolith. Extract services when you have a specific, measurable benefit that justifies the cost.

**Ignoring the data model.**
Drawing service boxes without understanding the data those services manage leads to interfaces that make the right thing hard and the wrong thing easy. The solution: design the data model before the service architecture.

**Over-specifying implementation details, under-specifying tradeoffs.**
Saying "I'll use Kafka" without explaining why (what problem does Kafka solve here that a simpler approach doesn't?) signals memorization rather than understanding. The solution: for every technology choice, explain the problem it solves and the alternative you rejected.

**Treating CAP theorem as a database property rather than an operation property.**
"PostgreSQL is CP" is a common but oversimplified claim. A database can be configured for different consistency levels per operation. The solution: discuss consistency requirements at the level of specific operations, not entire systems.

**Adding caching as an afterthought.**
"And then we add Redis for caching" without explaining what's cached, with what TTL, what the invalidation strategy is, and what happens when the cache is cold is not a design — it's a wish. The solution: whenever you add caching, describe the full strategy.

**Ignoring operational concerns.**
A design that doesn't address monitoring, deployment, failure recovery, and debugging will create enormous operational pain when built. The solution: spend time on the operational questions in Phase 2's step 7.

**Designing for day-ten scale instead of day-one scale.**
Building for millions of users when you have a thousand creates premature complexity and delays the launch that would get you to millions. The solution: design for 10x current scale, with a clear path to 100x, and identify the specific trigger points that require architectural changes.

**Treating performance and cost as separate concerns.**
Aggressive caching, multiple replicas, and global CDN distribution are expensive. A design that optimizes purely for performance without considering cost will be rejected in a real engineering organization. The solution: include rough cost estimates for major architectural decisions.

**Never practicing out loud.**
System design is a communication skill as much as a technical skill. Reading about systems and thinking about them silently is very different from explaining them clearly while someone challenges your decisions. The solution: practice explaining designs out loud, record yourself, and get feedback from others.

---

## Phase 8: Suggested Projects and Exercises

These are the exercises that build real intuition, ordered by what they teach.

**Project 1: Build a key-value store from scratch.**
Implement a simple in-memory key-value store with get, set, delete, and TTL expiration. Then add persistence (write-ahead log). Then add a simple replication protocol (primary sends operations to replicas). This teaches database internals, persistence, and replication — all foundational.

**Project 2: Implement a consistent hash ring.**
Build the data structure, add nodes, remove nodes, and observe which keys move. Then implement virtual nodes and observe improved distribution. This teaches the data structure underlying distributed caching and sharding.

**Project 3: Build a simple task queue.**
A producer adds tasks to a queue stored in Redis. Multiple worker processes consume and execute tasks. Handle worker failure (tasks that are claimed but not acknowledged). Implement priority queues. This teaches async processing, distributed coordination, and fault tolerance.

**Project 4: Implement a rate limiter.**
Build the token bucket algorithm backed by Redis. Test it under concurrent load. Observe the edge cases (what happens at exactly the rate limit?). Add the rate limit headers. This teaches both the algorithm and concurrent distributed state management.

**Project 5: Build a URL shortener end-to-end.**
The complete system: creation API, redirect service, database, caching layer. Then load test it. Find where it breaks. Fix it. Document what you learned. This is the foundational system design exercise.

**Project 6: Implement a notification system.**
Multiple notification channels (email via a service like Mailgun, push notification simulation). Event-driven — publish events, consume them, send notifications. Deduplication. Priority queues. This teaches event-driven architecture in a practical context.

**Project 7: Build a simple search engine.**
Build an inverted index from scratch over a corpus of documents. Implement basic TF-IDF ranking. Add Elasticsearch and compare your implementation's quality and performance to it. This demystifies search.

**Project 8: Implement a distributed lock.**
Use Redis SETNX with an expiry to implement a distributed lock. Write a test that deliberately causes the lock to expire while the holder is still working — observe the behavior. Implement lock extension. This teaches distributed coordination and its failure modes.

**Project 9: Design, document, and present your work system's architecture.**
Take the most complex system you work on daily. Draw its complete architecture. Write an architecture document in the ADR format. Identify three things you'd change and why. Present it to a colleague and defend your analysis. This is the exercise with the most immediate professional impact.

**Project 10: Design a system from a real interview prompt.**
Take a novel system design prompt (design a hotel booking system, design a document collaboration tool, design a distributed job scheduler). Apply the seven-step framework. Write it up. Get feedback from an experienced engineer. Repeat monthly.

---

## Final Note: The Mindset That Ties Everything Together

Everything in this syllabus is in service of one capability: the ability to reason clearly about systems you have never seen before.

That capability is not built by memorizing designs. It is built by understanding the forces — the physics of networks, the economics of storage, the mathematics of distributed consensus, the psychology of users — well enough that the right design emerges naturally from the problem.

When you deeply understand why Kafka retains messages while RabbitMQ deletes them, you can reason about any event streaming requirement. When you deeply understand why consistent hashing exists, you can apply it to any sharding problem. When you deeply understand the fan-out problem in social feeds, you can recognize the same problem in any system where one event must be propagated to many consumers.

The designs are examples. The reasoning is the skill.

Study the examples to build the reasoning. Apply the reasoning to everything else. There is no shortcut, but there is a path — and you are already on it.