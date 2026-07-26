# System Design — Senior Interview Guide

This chapter covers the system-design questions and reasoning patterns most often used in senior engineering interviews. The goal is not to reproduce one “correct” architecture; it is to make assumptions explicit, protect the important invariants, and evolve a simple design in response to evidence.

> **How to answer:** clarify requirements, estimate scale, define contracts and data, draw the simplest viable design, then deepen the bottlenecks and trade-offs selected by the interviewer.

## Contents

1. [Interview method](#1-interview-method)
2. [Requirements and estimation](#2-requirements-and-estimation)
3. [APIs and data modeling](#3-apis-and-data-modeling)
4. [Scaling and traffic management](#4-scaling-and-traffic-management)
5. [Caching and content delivery](#5-caching-and-content-delivery)
6. [Storage, partitioning, and replication](#6-storage-partitioning-and-replication)
7. [Messaging and asynchronous processing](#7-messaging-and-asynchronous-processing)
8. [Consistency, coordination, and identifiers](#8-consistency-coordination-and-identifiers)
9. [Reliability and operations](#9-reliability-and-operations)
10. [Security and privacy](#10-security-and-privacy)
11. [Common design scenarios](#11-common-design-scenarios)
12. [Rapid revision](#12-rapid-revision)

---

## 1. Interview method

### 1. What should you do first in a system-design interview?

Clarify the problem before drawing components. Identify the users, core use cases, out-of-scope features, traffic shape, data volume, latency, availability, durability, consistency, security, and geographic requirements.

Restate the agreed scope. A design optimized for 100 internal users differs radically from one serving hundreds of millions, and a payment ledger has different correctness requirements from a social feed.

### 2. What is a strong interview structure?

1. Clarify functional and non-functional requirements.
2. Estimate traffic, storage, bandwidth, and growth.
3. Define the critical API and data model.
4. Draw the high-level request and data flows.
5. Identify the bottlenecks and failure domains.
6. Deep-dive into one or two important areas.
7. Summarize trade-offs, evolution, and open risks.

Keep a visible list of assumptions. Spend depth where the problem is distinctive rather than reciting generic infrastructure.

### 3. How much time should each part receive?

For a typical 45–60 minute session, spend roughly 5–10 minutes on requirements and estimates, 5–10 on contracts/data, 10–15 on the high-level design, and the remainder on selected deep dives and trade-offs.

This is guidance, not a script. If the interviewer redirects, follow the signal. Do not spend half the session multiplying numbers to false precision.

### 4. How should a design be communicated?

Name each component by responsibility, draw the critical arrows, and narrate one read and one write path. Label synchronous versus asynchronous edges, ownership, caches, queues, and data stores.

Explain why each component exists. If removing it does not violate a requirement, it may be premature. Keep the diagram readable and add detail incrementally.

### 5. Should you start with microservices?

No. Start with logical responsibilities and the simplest deployment that satisfies the requirements. Independent services are justified by ownership, release, scaling, fault, or security boundaries—not by the interview format.

You can say that components begin as modules and split when their workload or ownership diverges. This demonstrates evolutionary design and avoids accidental distributed complexity.

### 6. How should trade-offs be presented?

Tie every trade-off to a requirement: “I choose asynchronous replication because lower write latency is more important here, accepting bounded stale reads.” Compare alternatives and identify what evidence would cause a different choice.

Avoid absolute claims such as “NoSQL scales” or “Kafka guarantees exactly once.” State the guarantee boundary, workload, failure case, and operational cost.

### 7. What does “design for scale” actually mean?

Design for the stated scale and a plausible growth path. Separate stateless compute, keep state behind explicit interfaces, identify partition keys, bound queues, and avoid single resources that cannot be replicated or replaced.

Do not deploy maximum-scale complexity on day one. Explain the trigger for each evolution: measured database saturation, hot partitions, regional latency, or team ownership—not hypothetical fashion.

### 8. What signals a senior-level answer?

A senior answer covers correctness and operations, not only throughput. It identifies invariants, failure modes, overload behavior, security boundaries, data lifecycle, observability, migration, rollback, and cost.

It also says what is intentionally out of scope and where a simpler design is sufficient.

---

## 2. Requirements and estimation

### 9. Functional versus non-functional requirements?

Functional requirements describe what users can do: create a short URL, send a message, search files. Non-functional requirements describe qualities and constraints: latency, throughput, availability, durability, consistency, security, privacy, cost, and geography.

Prioritize them. “Highly available, strongly consistent, globally low-latency, and cheap” is not a useful unqualified requirement because the design must resolve conflicts among these goals.

### 10. Which scale estimates matter most?

- Average and peak requests per second, separated into reads and writes.
- Concurrent connections or active users where relevant.
- Payload and object sizes.
- New data per day and retention period.
- Read/write bandwidth and egress.
- Fan-out or amplification factors.
- Growth and burst assumptions.

Round numbers and state assumptions. The purpose is to expose the dominant resource and architecture, not to predict procurement exactly.

### 11. How do you estimate requests per second?

Divide daily operations by roughly 86,400 seconds, then apply a peak factor based on traffic shape. One billion requests/day is about 11,600 average requests/second; at a 5× peak, plan around 58,000 requests/second before retry and fan-out amplification.

Separate endpoints and regions. An average can hide a launch event, morning burst, or one tenant generating most traffic.

### 12. How do you estimate storage?

Estimate records per period × bytes per record × retention, then add indexes, metadata, replication, versioning, and safety margin. Distinguish logical payload from physical storage.

For example, 10 million 1 KB objects/day retained for one year is about 3.65 TB raw; three replicas, indexes, and metadata may multiply that substantially. State whether binary objects belong in object storage rather than the database.

### 13. How do you estimate bandwidth?

Multiply operations per second by average transferred bytes, separately for ingress and egress. Account for headers, compression, fan-out, replicas, cache miss rate, and cross-region traffic.

Large media downloads often make egress cost and CDN behavior more important than application CPU. A small write that fans out to millions of followers can invert the apparent workload.

### 14. Availability versus durability?

Availability is the ability to serve requests now. Durability is the probability that acknowledged data remains recoverable. A system may be available while returning stale data, or unavailable while safely retaining all data.

Define both. A temporary image-processing delay may be acceptable; losing an acknowledged payment is not. Replication improves both only when failure domains and restore behavior are correctly designed.

### 15. Latency versus throughput?

Latency is time per operation; throughput is completed work per unit time. Increasing concurrency can improve throughput until a resource saturates, after which queueing sharply increases latency.

Use percentile latency such as p95 or p99, not averages alone. Define end-to-end user latency and component budgets, including network, queue, retries, and dependency time.

### 16. What is a capacity envelope?

It is the tested range within which the system meets its SLOs: traffic rate, payload size, concurrency, data set, and dependency behavior. Beyond it, define graceful rejection or degradation rather than uncontrolled collapse.

Load tests should discover saturation points and recovery behavior. Autoscaling reacts after signals arrive and does not replace capacity headroom.

---

## 3. APIs and data modeling

### 17. How should an API be designed during an interview?

Define only the critical operations, identifiers, request/response fields, authentication context, errors, pagination, and idempotency. The API should reveal the business capability without exposing internal storage.

```http
POST /v1/short-links
Idempotency-Key: 8d70...

{"targetUrl":"https://example.com/article","expiresAt":"2027-01-01T00:00:00Z"}
```

Mention size limits, validation, rate limits, and compatibility where relevant.

### 18. REST versus RPC versus events?

REST is effective for resource-oriented public contracts and HTTP ecosystem benefits. RPC fits action-oriented internal calls and strong schemas. Events communicate facts asynchronously to multiple consumers.

Use the interaction semantics, not fashion: immediate response versus deferred work, one consumer versus many, compatibility needs, streaming, and failure behavior. A system commonly uses all three at different boundaries.

### 19. How do you make a write API idempotent?

Accept a client-scoped idempotency key, atomically associate it with request identity and result, and return the stored result on retry. Reject reuse with a different payload.

Define retention, concurrent requests, in-progress state, and retryable failures. Natural business keys or conditional writes may provide idempotency without a separate table.

### 20. Offset versus cursor/keyset pagination?

Offset pagination is easy and supports page jumps, but becomes expensive for deep pages and unstable under concurrent changes. Cursor/keyset pagination continues after a stable ordered key, scaling predictably and reducing duplicates/omissions.

Use a deterministic order with a unique tie-breaker. Protect opaque cursors from tampering and define how filters and data changes affect them.

### 21. How do you choose a data model?

Start from access patterns and invariants: what is read/written together, which queries dominate, what must be atomic, and which data can be duplicated. Then choose keys, relationships, indexes, and storage technology.

Do not choose a database first and force the domain into it. Normalize authoritative transactional data where useful; denormalize explicit read models with ownership and reconciliation.

### 22. Relational versus document versus key-value storage?

Relational stores excel at constraints, transactions, joins, and flexible querying. Document stores fit aggregate-shaped data and evolving nested documents. Key-value stores offer simple access and horizontal distribution for known key patterns.

The decision depends on consistency, query shape, size, transactions, indexing, operations, and team capability. “NoSQL for scale” is insufficient; relational databases also scale, and distributed NoSQL has its own limits.

### 23. When is a search engine appropriate?

Use a search engine for full-text relevance, tokenization, fuzzy matching, faceting, and complex inverted-index queries. Keep an authoritative source elsewhere and update the search index asynchronously.

Expose indexing lag and rebuild capability. Search engines are usually poor primary systems of record for strict transactional invariants.

### 24. Why store blobs in object storage?

Object storage is optimized for durable large objects, high capacity, multipart upload, lifecycle rules, and direct/CDN delivery. Keep metadata, permissions, and workflow state in a database.

Use pre-signed upload/download URLs where appropriate so application servers do not proxy every byte. Validate content, size, ownership, expiration, malware policy, and orphan cleanup.

### 25. How should audit history be designed?

Record actor, action, target, timestamp, outcome, and safe context in an append-oriented, tamper-resistant system with controlled access and retention. Audit is distinct from debug logging.

Avoid secrets and unnecessary personal data. Define clock/source identity, export, legal retention, deletion exceptions, and how privileged audit access is itself audited.

---

## 4. Scaling and traffic management

### 26. Vertical versus horizontal scaling?

Vertical scaling gives one node more CPU, memory, or storage and is simple but has a ceiling and larger failure unit. Horizontal scaling adds nodes and requires distribution, balancing, statelessness or state coordination.

Use vertical improvements before distribution when they are sufficient. Explain the future horizontal path for the resources likely to hit a ceiling.

### 27. Why are stateless application servers easier to scale?

Any instance can handle any request, so load balancing, replacement, and autoscaling are straightforward. Durable session or workflow state lives in a shared store or client token with appropriate security.

Stateless does not mean “no state”; it means instances do not own irreplaceable request-to-request state. Local caches and connection pools are fine when loss on restart is safe.

### 28. What load-balancing algorithms matter?

- Round robin for broadly equal instances and requests.
- Least connections/in-flight work for variable request duration.
- Weighted routing for unequal capacity or rollout.
- Consistent hashing for affinity with reduced remapping.
- Locality-aware routing to reduce latency and cross-zone cost.

Health, slow start, connection reuse, and long-lived streams can matter more than the headline algorithm.

### 29. Layer 4 versus Layer 7 load balancing?

Layer 4 routes using transport information such as IP and port and can handle arbitrary TCP/UDP efficiently. Layer 7 understands HTTP or other application protocols and can route by host, path, header, identity, or content.

Layer 7 enables richer policy but performs more work and terminates or inspects protocol connections. Many systems use both at different boundaries.

### 30. What is autoscaling based on?

Scale on a signal that predicts required capacity: CPU for CPU-bound work, concurrency or queue depth for request/work systems, and lag or message age for consumers. Account for startup time and downstream limits.

Use target utilization with headroom, scale-up quickly enough, and scale-down conservatively. Autoscaling cannot fix an overloaded database or a hot partition and may amplify pressure on dependencies.

### 31. How do you handle traffic spikes?

Use capacity headroom, CDN/cache absorption, bounded queues, rate/concurrency limits, load shedding, and prioritized work. Pre-scale for predictable events; smooth asynchronous work where latency allows.

Define what is rejected first and how clients retry. An unbounded queue converts a short burst into prolonged high latency or memory failure.

### 32. What is consistent hashing?

It maps keys and nodes onto a logical ring so adding/removing a node remaps only part of the keyspace rather than almost every key. Virtual nodes improve distribution and support unequal capacity.

It helps cache or shard routing but does not solve replication, hot keys, rebalancing safety, or membership consistency. Modern systems may use other deterministic partition maps with similar goals.

### 33. What causes a hot key or hot partition?

Skewed access sends disproportionate traffic to one cache key, shard, tenant, celebrity, or time bucket. Overall capacity appears sufficient while one owner saturates.

Mitigate with better partition keys, key salting and aggregation, replication of hot reads, local caching, tenant isolation, batching, or special treatment. Detect skew per key/partition, not only cluster averages.

### 34. What is a cell-based architecture?

A cell is a largely self-contained slice of compute and data serving a subset of users or tenants. Failures and overload remain within a cell, limiting blast radius and enabling incremental scaling.

Cells require routing, placement, capacity buffers, cross-cell operations, and migration. Shared global dependencies can defeat isolation, so minimize and harden them.

---

## 5. Caching and content delivery

### 35. Why cache?

Caching reduces latency, origin load, and cost by serving reused data closer to callers. It trades freshness and operational simplicity for performance.

Define key, value, TTL, invalidation, consistency, capacity, eviction, failure behavior, and observability. A cache without bounds or freshness semantics is a new source of outages.

### 36. Cache-aside versus read-through versus write-through?

- **Cache-aside:** application reads cache, loads on miss, then fills it; flexible but permits miss races.
- **Read-through:** cache abstraction loads missing data.
- **Write-through:** writes update cache and backing store synchronously.
- **Write-behind:** cache queues backing-store writes; faster but risks loss/ordering complexity.

Choose according to ownership and consistency. Cache-aside is common because it keeps the authoritative write explicit.

### 37. What is cache invalidation?

Remove or update cached data when the source changes, or accept staleness until TTL. Invalidation can be synchronous, event-driven, versioned, or namespace-based.

Distributed invalidation can be lost or reordered. Use TTL as a safety bound, version keys when useful, and define behavior when the invalidation channel is unavailable.

### 38. What is a cache stampede?

Many requests miss or observe an expired popular key and concurrently load the same data, overwhelming the origin. Prevent it with per-key request coalescing, stale-while-revalidate, randomized TTL, prewarming, or controlled refresh.

Ensure the lock/coalescing path has a timeout and failure behavior. Global locking can turn one cache miss into system-wide serialization.

### 39. What are cache penetration and cache pollution?

Penetration means repeated requests for absent keys bypass the cache and hit the origin; negative caching, validation, or probabilistic filters can help. Pollution means low-value or one-time entries evict useful data; admission policies and separate caches can help.

Negative-cache TTL must account for newly created data. Probabilistic filters can produce false positives but should not produce false negatives if used correctly.

### 40. What is a CDN?

A content delivery network caches and serves content from geographically distributed edge locations, reducing origin load and user latency. It is ideal for static and safely cacheable dynamic content.

Design cache-control headers, versioned URLs, purge behavior, authorization, signed URLs/cookies, origin protection, and regional failure. Do not cache private responses under incomplete keys.

### 41. Client, edge, service, or database caching?

Cache at the layer with the most reuse and clearest correctness boundary. Client/edge caches reduce network work; service caches reuse computed/domain values; database buffer caches accelerate storage access transparently.

Multiple cache layers multiply invalidation and observability complexity. State which layer is authoritative and how freshness propagates.

---

## 6. Storage, partitioning, and replication

### 42. Replication versus partitioning?

Replication copies data to improve availability, read scale, and durability. Partitioning divides data to increase write/storage capacity and isolate workloads. Large systems often use both: each partition has replicas.

Replication introduces lag and failover consistency; partitioning introduces routing, skew, rebalancing, and cross-partition operations.

### 43. How do you choose a shard key?

A good key distributes storage and traffic, keeps common operations within one shard, is stable, and supports routing. Tenant ID preserves locality but may create large-tenant hotspots; hashing distributes evenly but weakens range queries.

Estimate cardinality and skew. Resharding is inevitable at long time horizons, so keep routing indirect and avoid exposing physical shard identity in public contracts.

### 44. Range versus hash partitioning?

Range partitioning preserves ordered/range access and simplifies time-based lifecycle, but sequential keys can create a hot newest partition. Hash partitioning distributes point access evenly but scatters ranges.

Hybrid strategies—hash by tenant then range by time, or bucketing—can balance needs. More partitions increase metadata and operational cost.

### 45. Leader-follower replication?

One leader accepts writes and propagates changes to followers, which may serve reads or take over after failure. It simplifies write ordering but places leader availability and throughput on the critical path.

Define synchronous acknowledgement level, replica lag, failover detection, fencing, and what happens to acknowledged but not replicated writes. Prevent the old leader from accepting writes after failover.

### 46. Multi-leader replication?

Multiple leaders accept writes, often across regions, then replicate between them. It improves local write availability and latency but creates conflicts, ordering ambiguity, and more complex recovery.

Use domain-specific conflict resolution, ownership partitioning, or commutative operations. “Last write wins” can silently discard valid updates and depends on trustworthy ordering.

### 47. Quorum reads and writes?

With N replicas, a simplified quorum model chooses W write acknowledgements and R read responses; if `W + R > N`, read and write sets overlap. This does not by itself guarantee linearizability because versions, concurrent writes, sloppy quorums, repair, and failures matter.

State the database's actual consistency protocol instead of applying the formula as a universal proof.

### 48. What is replication lag?

Followers apply writes after the leader, so reads may be stale. This causes read-after-write anomalies, missing recent permissions, and inconsistent pagination.

Mitigate with leader reads for critical paths, session stickiness, version/position tokens, bounded-staleness routing, or user-visible pending state. Monitor time and log-position lag.

### 49. How do you rebalance shards?

Move bounded key ranges or virtual partitions while tracking ownership version. Copy a consistent snapshot, catch up changes, verify, switch routing atomically, then retire the old copy.

Throttle movement to protect foreground traffic and make it resumable. Dual reads/writes during transition need deduplication and clear authority to avoid divergence.

### 50. How do secondary indexes work in a sharded system?

A local secondary index covers data within one shard; a global index maps indexed values across shards. Local indexes make scatter-gather queries expensive. Global indexes improve lookup but add distributed writes and consistency lag.

Alternatives include search indexes, materialized views, or choosing a query-specific partition. Define uniqueness and failure semantics for global index updates.

### 51. What is data tiering?

Place hot recent data on low-latency expensive storage, warm data on cheaper online storage, and cold/archive data on high-latency low-cost storage. Lifecycle policies move data by age or access.

APIs must define retrieval latency, and compliance may constrain location/retention. Keep catalogs and restore procedures so archived data remains discoverable and testable.

---

## 7. Messaging and asynchronous processing

### 52. Why introduce a queue?

A queue decouples producer and consumer timing, absorbs bursts, enables retry, and isolates slow work from user latency. It adds backlog, duplicate delivery, eventual completion, and operational state.

Bound producer admission and define maximum useful message age. A durable queue moves overload in time; it does not create downstream capacity.

### 53. Queue versus event log?

A traditional queue distributes work and often removes/acknowledges completed messages. An append-only event log retains ordered partitioned history and lets independent consumer groups track positions and replay.

Use queues for task ownership and logs for durable streams of facts or multiple projections. Product implementations overlap; focus on retention, replay, ordering, fan-out, and acknowledgement semantics.

### 54. What delivery guarantee should be assumed?

Assume at-least-once unless a narrower end-to-end guarantee is demonstrated. Consumers must tolerate duplicates through idempotent effects, unique business keys, or transactional inbox state.

Broker-level exactly-once does not automatically cover email, payment APIs, or unrelated databases. Define the guarantee boundary explicitly.

### 55. How do you preserve message order?

Route events requiring order to the same partition/key and include entity version or sequence. Consumers detect duplicates, stale versions, and gaps.

Global order limits throughput and is rarely required. Define the smallest scope—order, account, conversation—whose events must be ordered.

### 56. What is fan-out on write versus fan-out on read?

Fan-out on write precomputes each follower's feed when content is created, making reads fast but writes expensive for high-fan-out authors. Fan-out on read merges followed authors' content at request time, making writes cheap but reads costly.

Large social systems often use a hybrid: precompute for ordinary accounts and merge celebrity content at read time.

### 57. How do you schedule background jobs?

Persist job definition and state durably, use a scheduler to enqueue due executions, lease work to workers, make handlers idempotent, and record attempts/results. Partition by due time or job ID as scale requires.

Handle clock skew, duplicate scheduling, missed execution, long jobs, cancellation, retries, and per-tenant fairness. “Exactly once at 09:00” is unrealistic without defining tolerance and effect idempotency.

### 58. What is backpressure?

Backpressure makes producers or upstream stages reduce work when consumers saturate. Use bounded queues, concurrency limits, demand signaling, admission rejection, or controlled polling/prefetch.

Monitor backlog age as well as count. Old work may violate business deadlines even when eventual throughput catches up.

### 59. How do dead-letter queues help?

They quarantine messages that cannot be processed after policy, allowing unrelated work to continue. Include failure reason, attempt history, and original identifiers, then alert and provide a controlled replay/repair workflow.

A dead-letter queue is not resolution. Sensitive payload handling, ordering impact, retention, and ownership must be defined.

---

## 8. Consistency, coordination, and identifiers

### 60. Strong versus eventual consistency?

Strong models provide a single up-to-date ordering or visibility guarantee within a defined scope. Eventual consistency permits temporary disagreement but converges when writes stop and repair succeeds.

Choose per invariant and user experience. Inventory reservation may require authoritative coordination; recommendations may tolerate minutes of lag. State session guarantees such as read-your-writes when relevant.

### 61. What does CAP theorem actually say?

During a network partition, a distributed system cannot guarantee both every request receives a non-error response and every read observes one single current value. Systems choose behavior per operation when communication is unavailable.

CAP does not describe normal operation, latency, durability, or isolation, and “CA system” is not a practical way to dismiss partitions in a distributed deployment.

### 62. What is linearizability?

Each operation appears to occur atomically at one point between invocation and response, respecting real-time ordering. After a successful write completes, later reads must observe it or a later value.

Linearizability is a single-operation consistency property, not the same as serializable multi-operation transactions. It often costs coordination and availability during partitions.

### 63. Serializability versus linearizability?

Serializability means concurrent transactions produce an outcome equivalent to some serial order, but that order need not respect real-time order across transactions. Linearizability respects real-time order for individual operations.

A system may offer one without the other. When stating “strong consistency,” name the required property.

### 64. What is consensus used for?

Consensus lets nodes agree on an ordered value or replicated log despite certain failures. It underpins leader election, configuration state, metadata, and strongly consistent replication.

Protocols such as Raft and Paxos require a quorum and cannot make progress without enough communicating members. Consensus does not solve arbitrary business transactions or eliminate application-level idempotency.

### 65. Why are distributed locks risky?

Clients can pause, lose connectivity, or outlive a lease while another client acquires the lock. The old holder may then continue writing, causing split ownership.

Use fencing tokens—monotonically increasing epochs rejected by the protected resource—or conditional/versioned writes. Keep lease semantics, clock assumptions, and failure handling explicit. Prefer data ownership or atomic storage operations when possible.

### 66. How do you generate globally unique IDs?

Options include database sequences, random UUIDs, time-ordered UUIDs, range allocation, and Snowflake-style timestamp/worker/sequence IDs. Trade-offs include coordination, sortability, size, privacy, clock behavior, and collision risk.

Do not use guess-resistant IDs as authorization. Time-based schemes need clock-regression and worker-identity handling; random IDs can reduce index locality.

### 67. What are logical clocks?

Lamport clocks establish a causal-compatible ordering but cannot identify concurrency by themselves. Vector clocks track causal history across actors and can detect concurrent updates, at higher metadata cost.

Physical timestamps are useful but clocks skew. Hybrid logical clocks combine physical time with logical ordering in some systems. Choose only the ordering guarantee the domain needs.

### 68. What is the transactional outbox?

Write the business update and an outbox event in one local database transaction, then publish outbox records asynchronously via polling or change-data capture. Consumers handle duplicate publication idempotently.

It closes the database/broker dual-write gap but still requires ordering, lag monitoring, retries, cleanup, schema evolution, and reconciliation.

### 69. How do sagas manage distributed workflows?

Each participant commits a local transaction; later failure triggers compensating business actions. Orchestration makes workflow state explicit, while choreography distributes reactions among events.

Persist progress, use idempotent steps, define timeouts and manual recovery, and remember compensation may not fully reverse external effects.

---

## 9. Reliability and operations

### 70. What are SLI, SLO, SLA, and error budget?

An SLI measures user-relevant service behavior. An SLO sets a target over a window. An SLA is a contract with consequences. The error budget is the unreliability permitted by the SLO.

Use SLOs to decide reliability investment and release risk. Define eligible events precisely and monitor percentiles/correctness, not only server uptime.

### 71. How many “nines” should a system have?

Enough to meet user and business needs at justified cost. Each additional nine sharply reduces allowed failure time and often requires independent redundancy, automation, testing, and operational maturity.

Calculate dependency composition. A request that requires several serial 99.9% services can have lower end-to-end availability unless redundancy or graceful degradation exists.

### 72. What is fault tolerance versus high availability?

High availability minimizes service interruption through redundancy and failover. Fault tolerance aims to continue correctly despite particular component failures, often with no visible interruption.

Define the failure model: process, node, zone, region, dependency, operator error, or data corruption. Replicas in one failure domain do not protect against that domain's failure.

### 73. What are RTO and RPO?

Recovery Time Objective is the target time to restore service after disaster. Recovery Point Objective is the acceptable data-loss window measured backward from failure.

These business requirements determine replication, backup frequency, topology, and operational investment. Test recovery; architecture diagrams do not prove either objective.

### 74. Backup versus replication?

Replication improves availability and can reduce data loss from hardware failure, but it also rapidly copies deletion, corruption, or malicious writes. Backups preserve recoverable historical states.

Use both where required, isolate backup credentials/failure domains, define retention, and regularly restore-test at representative scale.

### 75. How do you avoid cascading failure?

Use end-to-end deadlines, bounded pools/queues, per-dependency isolation, admission control, safe retries with jitter, circuit breakers, load shedding, and graceful degradation.

Monitor saturation and retry amplification. Recovery must be controlled so releasing accumulated traffic does not immediately overload the dependency again.

### 76. Active-active versus active-passive multi-region?

Active-passive routes to one region and fails over to a standby, simplifying write ownership but increasing failover time and idle cost. Active-active serves multiple regions, improving latency and capacity but complicating consistency, conflict resolution, and routing.

Choose from RTO/RPO, write semantics, residency, cost, and operational capability. Rehearse traffic and data failover, including return to normal.

### 77. What should be monitored?

Request rate, errors, latency distributions, saturation, dependency health, queue age, data correctness, replication lag, capacity, and domain outcomes. Correlate metrics with traces and structured logs.

Alert on user impact and actionable precursors. High-cardinality identifiers belong in traces/logs, not metric labels. Instrument the design before relying on automatic failover.

### 78. How should overload behavior be designed?

Set a capacity envelope, bound concurrency and queues, prioritize critical work, reject early with clear retry semantics, and degrade optional features. Preserve resources for health, control, and recovery operations.

Queueing theory means latency rises rapidly near saturation. Autoscaling alone responds too late to instantaneous bursts and cannot expand a fixed database.

### 79. How do you deploy safely?

Use backward-compatible contracts, expand-and-contract schema changes, automated tests, staged canary/rolling rollout, version-tagged telemetry, abort thresholds, and rapid rollback or roll-forward.

Mixed versions are normal during rollout. Asynchronous messages and stored data may outlive deployments, so compatibility extends beyond live HTTP clients.

---

## 10. Security and privacy

### 80. Where are the trust boundaries?

At every transition between identities, networks, services, tenants, and data classifications. Authenticate callers and workloads, authorize at the data-owning service, validate all external input, and apply least privilege.

Internal networks are not inherently trusted. Gateways can perform coarse checks, but downstream services still protect sensitive operations.

### 81. Authentication versus authorization?

Authentication establishes identity; authorization decides whether that identity may perform an action on a resource. Keep end-user identity distinct from service/workload identity when a service acts on a user's behalf.

Do not trust user-controlled identity headers. Validate token signature, issuer, audience, time, and policy; cache authorization only with safe invalidation semantics.

### 82. How should encryption be designed?

Use TLS in transit and managed encryption at rest, with keys separated from data and access controlled through workload identity. Rotate keys and certificates without downtime, audit use, and define backup encryption.

Encryption does not replace authorization or prevent an authorized compromised service from reading plaintext. Consider application-level or field-level encryption for specific threats.

### 83. How do you design tenant isolation?

Choose shared rows, separate schemas/databases, or dedicated infrastructure according to risk, scale, compliance, and cost. Enforce tenant identity in every access path and include it in keys, authorization, quotas, and audit.

Test for cross-tenant access and noisy-neighbor behavior. Operational tooling, caches, logs, analytics, and backups must preserve the same boundary.

### 84. How do privacy and retention affect architecture?

Minimize collected data, record purpose and consent where required, control regional location, restrict access, and define retention/deletion across primary stores, replicas, caches, search indexes, events, logs, and backups.

Event sourcing and broad event payloads make deletion difficult. Design data lineage and deletion propagation before accumulating irreversible copies.

### 85. How do you defend a public API?

Use authentication, authorization, TLS, schema/size validation, rate and concurrency limits, replay/idempotency protection, safe error responses, audit, and abuse monitoring. Place DDoS protection and CDN/WAF capabilities at the edge where appropriate.

Protect expensive query shapes and downstream resources, not just request count. Never expose internal stack traces or use unpredictable IDs as access control.

---

## 11. Common design scenarios

### 86. Design a URL shortener: what are the key decisions?

- Generate a unique short key with random or allocated IDs encoded compactly.
- Store `shortKey → targetUrl`, owner, timestamps, and policy.
- Serve redirects through a read-heavy, cacheable path.
- Make creation idempotent if required and validate malicious targets.
- Record analytics asynchronously so redirects do not wait.
- Handle custom aliases, expiration, abuse, and hot links.

The keyspace must make collisions manageable. Cache negative results briefly and protect popular redirects with edge caching.

### 87. Design a notification service: what are the key decisions?

Accept a durable notification command with idempotency key, recipient, template/version, channel policy, and schedule. Persist state, enqueue per channel, enforce preferences and rate limits, call providers with retries, and record delivery attempts.

Separate accepted, rendered, sent-to-provider, delivered, failed, and expired states. Provider acknowledgement is not end-user delivery. Protect PII and support unsubscribe/compliance rules.

### 88. Design a distributed job scheduler: what are the key decisions?

Persist schedules and job state, partition upcoming time ranges, use leader/lease ownership to enqueue due executions, and let workers claim jobs with leases. Handlers are idempotent and progress is durable.

Handle duplicate scheduling, clock skew, misfires, long execution, retries, cancellation, priority, fairness, and history retention. Use fencing/version checks so expired owners cannot complete stale leases.

### 89. Design a social feed: what are the key decisions?

Store posts and follow relationships, then choose fan-out on write, read, or hybrid based on follower distribution. Precompute ordinary users' feeds, merge celebrity posts at read time, and rank/paginate by stable cursor.

Define deletion, privacy changes, blocking, freshness, duplicate suppression, and ranking evolution. Media belongs behind object storage/CDN; counters may be eventually consistent.

### 90. Design chat: what are the key decisions?

Maintain long-lived connections through gateways, route by user/conversation, assign server message IDs and per-conversation sequence where needed, persist before acknowledgement, and deliver asynchronously to recipients/devices.

Handle reconnect cursors, offline delivery, duplicates, typing/presence as ephemeral state, multi-device sync, attachments, abuse, encryption, and retention. Global message order is unnecessary; conversation-level ordering is usually sufficient.

### 91. Design file storage and sharing: what are the key decisions?

Keep metadata and permissions in a transactional database and bytes in object storage. Use resumable multipart uploads with pre-signed URLs, checksum verification, malware scanning, versioning, and asynchronous processing.

Downloads use authorized short-lived URLs and CDN where safe. Define sharing inheritance, revocation, quotas, deduplication privacy, orphan cleanup, regional storage, and deletion across versions/backups.

### 92. Design a payment system: what are the key decisions?

Use an immutable double-entry ledger as financial source of truth, explicit payment state machine, idempotent commands, unique external references, and transactional outbox for downstream events. Integrate providers through durable attempts and reconciliation.

Never infer money from mutable status rows alone. Define authorization/capture/refund/chargeback, currency and decimal rules, audit, PCI boundaries, fraud, settlement mismatch, and manual recovery.

### 93. Design a rate limiter: what are the key decisions?

Select identity and policy scope, algorithm (token bucket, sliding window, etc.), local versus distributed enforcement, burst allowance, and failure behavior. Return remaining quota/retry hints where useful.

Local limits are fast but approximate globally; centralized counters improve consistency but add latency and failure. Shard by identity, use atomic scripts/operations, and protect the limiter itself from high-cardinality state.

### 94. Design metrics ingestion: what are the key decisions?

Accept batched compressed points through regional stateless collectors, validate and rate-limit tenants, buffer in a durable partitioned log, aggregate/downsample, then store in a time-series engine partitioned by tenant/metric/time.

Control label cardinality, support late/out-of-order data, retention tiers, backpressure, and query quotas. Separate ingestion availability from query availability and expose dropped/rejected measurements.

### 95. Design search autocomplete: what are the key decisions?

Build a normalized prefix index/trie or search-engine structure from candidate queries/entities with popularity and policy signals. Serve from memory/edge caches with tight latency, then rank by prefix, locale, freshness, personalization, and safety.

Update popular terms incrementally and rebuild offline. Protect privacy, suppress abusive/sensitive suggestions, handle typos, and bound personalization so cache reuse remains possible.

### 96. How should any design scenario conclude?

Restate how the design meets the highest-priority requirements, identify the principal trade-offs and remaining risks, and describe the next scaling trigger. Mention failure recovery, security, observability, cost, and what you deliberately deferred.

This summary demonstrates judgment and gives the interviewer a clear place to request another deep dive.

---

## 12. Rapid revision

### Must-answer questions

Before an interview, answer these without notes:

1. What do you clarify before drawing a design?
2. Which capacity estimates change architecture?
3. Availability versus durability?
4. Latency versus throughput?
5. How do you design an idempotent API?
6. Offset versus cursor pagination?
7. How do access patterns determine the data model?
8. When do blobs belong in object storage?
9. Vertical versus horizontal scaling?
10. How does a stateless tier help?
11. What creates a hot partition?
12. Why use cell-based architecture?
13. Cache-aside versus write-through?
14. How do you prevent a cache stampede?
15. What belongs behind a CDN?
16. Replication versus partitioning?
17. What makes a good shard key?
18. Leader-follower versus multi-leader?
19. How does replication lag affect UX?
20. Queue versus event log?
21. What does at-least-once require from consumers?
22. Fan-out on write versus read?
23. What is backpressure?
24. Strong versus eventual consistency?
25. What does CAP say during a partition?
26. Linearizability versus serializability?
27. Why do distributed locks need fencing?
28. How do you generate unique IDs?
29. How does the outbox close the dual-write gap?
30. How do sagas compensate failures?
31. SLI versus SLO versus SLA?
32. RTO versus RPO?
33. Backup versus replication?
34. How do you prevent cascading failure?
35. Active-active versus active-passive?
36. How should overload be handled?
37. What are the system's trust boundaries?
38. How do you isolate tenants?
39. What data must deletion propagate through?
40. How do you conclude a design interview?

### Thirty-second summary

System design begins with requirements, scale, and invariants—not a catalog of technologies. Define the API and data model, draw the simplest read/write paths, then evolve bottlenecks with caching, partitioning, replication, queues, and regional topology. Every choice has a consistency, failure, security, operational, and cost boundary. A senior design bounds resources, makes overload and recovery explicit, measures user-facing SLOs, and explains which evidence would justify the next level of complexity.

## Official references

- [Google Site Reliability Engineering](https://sre.google/sre-book/table-of-contents/)
- [Google Cloud Well-Architected Framework](https://docs.cloud.google.com/architecture/framework)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [IETF HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [Apache Kafka documentation](https://kafka.apache.org/documentation/)
- [Kubernetes concepts](https://kubernetes.io/docs/concepts/)

