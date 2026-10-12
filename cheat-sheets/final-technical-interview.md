# Final-Round Technical Interview — Cheat Sheet

A short plan for a second, deeper technical interview with a senior engineer: open-ended, scenario-based, covering Java, Spring Boot, databases, concurrency, and system design. The first round asked about a TTL-bound resource lock, slow-query diagnosis, and deep-page pagination; expect the final round to push the same themes one level deeper and ask for real experience.

> **How to answer:** clarify the scenario → state the direct answer → explain the mechanism → name the trade-off and failure mode → tie it to something you actually did. Say what you would *measure* before what you would *change*.

## Four-day plan

| Day | Focus | Read | Practice aloud |
|---|---|---|---|
| 1 | Concurrency and locking | [Core Java](../01-java-core/README.md) Q43–57, Q81–82; [System Design](../07-system-design/README.md) Q65 | Write the lease lock from memory; explain why expiry does not stop the old holder |
| 2 | Database diagnosis and pagination | [Databases](../03-database/README.md) Q19, Q21–25, Q45–49, Q52, Q60, Q70–79 | Narrate the slow-query triage table; run `EXPLAIN ANALYZE` on a real table with and without an index |
| 3 | Spring Boot and transactions | [Spring](../02-spring/README.md) Q12–18, Q31–36, Q45–51; Databases Q30–35, Q53–54 | "This endpoint is slow in production" end to end; propagation and rollback traps |
| 4 | System design and stories | [System Design](../07-system-design/README.md) Q19–20, Q35–39, Q58, Q68, Q93; [Microservices](../04-microservices/README.md) Q15, Q21, Q24, Q29, Q33, Q42–43 | One 30-minute design aloud; rehearse five experience stories |
| Interview day | Light review only | This sheet and each chapter's thirty-second summary | — |

## Round-one topics: the sixty-second versions

### 1. Thread-safe lock on a file or network resource, held no longer than a TTL

- Ask the scope first: threads in one JVM, processes on one host, or several instances?
- It is a **lease**: acquire returns a token and a deadline; release and use are valid only for the current token.
- In one JVM: state guarded by a `ReentrantLock` and `Condition`, timed `tryAcquire`, time from `System.nanoTime()` (monotonic), ownership check on release.
- **The trap:** expiry frees the lock but does not stop the old holder. Bound the work (socket timeouts, `Future.get(timeout)` + `cancel(true)`) and reject stale tokens at the resource (fencing).
- Blocking `java.io` reads ignore interrupts — close the socket or set `SO_TIMEOUT`.
- File: `FileChannel.tryLock()` is per-JVM and often advisory; it excludes other processes, not other threads.
- Several instances: Redis `SET NX PX` with compare-and-delete, a database row with `lock_until`, or etcd/ZooKeeper — always with a fencing token, because a GC pause can outlive any TTL.
- TTL sizing: measured p99 work time plus margin. Too short means duplicate work; too long means slow crash recovery.
- If it means a pooled resource: `Semaphore`/pool for concurrency, borrow timeout, max lifetime and idle eviction (HikariCP `maxLifetime`).

### 2. "A query is problematic — how do you find the cause?"

Two questions first: **one slow statement or many fast ones? Working or waiting?**

| Cause | Tell-tale evidence |
|---|---|
| N+1 | High call count, low mean time; many identical queries in one trace |
| Index | Sequential scan; rows read far exceed rows returned; non-sargable predicate or implicit cast |
| Plan / statistics | Estimated rows far from actual; fast for one parameter, slow for another |
| Partitioning | Plan touches all partitions; partition key missing from, or wrapped in a function in, the predicate |
| Locks / pool | Fast alone, slow under load; lock wait events; connection-acquisition time |
| Sort / spill | Explicit sort or hash on disk in the plan |

Tools to name: slow-query log, `pg_stat_statements`, `EXPLAIN (ANALYZE, BUFFERS)`, `pg_stat_activity`, Hibernate statistics or SQL logging, APM traces. Change one thing, re-measure on production-like data.

### 3. Pagination that gets slow on higher pages

- **Cause: `OFFSET`.** `LIMIT 20 OFFSET 100000` produces 100,020 rows and discards 100,000 — cost grows with depth. A per-request `COUNT(*)` (Spring Data `Page`) adds a second scan.
- **Fix: keyset (cursor-based) pagination** — `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC LIMIT 20`, backed by a matching index with a unique tie-breaker. Cost is independent of depth and stable under concurrent inserts.
- Keyset is *still* slow when the index does not match filter plus order, the filter is sparse along the index, or the sort key is not unique.
- Keyset cannot jump to page N or support arbitrary sorts — then use a deferred join, cap the depth, or use `Slice` instead of `Page`.
- Do not confuse an **API cursor** (opaque keyset token, stateless) with a **database cursor** (server-side result set that pins a connection and transaction — suited to exports, not user paging).

## Likely deeper follow-ups

**Java and concurrency**
- `synchronized` versus `ReentrantLock` versus `StampedLock`/`ReadWriteLock`; when optimistic reads pay off.
- Thread-pool sizing, queue bounds, rejection policy; what happens when the queue is unbounded.
- Deadlock: the four conditions, lock ordering, `tryLock` with timeout, reading a thread dump.
- Virtual threads: what they fix (blocking I/O scale), what they do not (CPU work, pool-sized downstreams), pinning.
- `CompletableFuture` timeouts and cancellation; why cancelling does not interrupt the running task.

**Spring Boot**
- `@Transactional` through a proxy: self-invocation, private methods, checked exceptions not rolling back, `REQUIRES_NEW` needing a second connection.
- Holding a database connection across a remote call; pool exhaustion.
- Slow endpoint triage: traces → query count → pool wait → GC → downstream latency.
- Idempotent endpoints, `@Async` executor configuration, graceful shutdown.

**Databases**
- Composite index column order; covering indexes; why the optimizer ignores an index.
- Isolation levels in the engine you used; lost update versus write skew; optimistic version column versus `SELECT ... FOR UPDATE`.
- Deadlock handling and retry; long transactions and their cost.
- Partitioning versus sharding; replication lag and read-your-writes.
- Safe schema migration and backfill on a large live table.

**System design**
- Idempotency keys and retries; at-least-once delivery with idempotent consumers.
- Transactional outbox instead of dual writes; saga versus distributed transaction.
- Cache-aside, invalidation, stampede protection.
- Rate limiting, backpressure, timeouts, circuit breakers.
- Always state: requirements and scale → API and data model → simplest design → bottleneck → evolution trigger.

## Trade-off phrasing

- "The simplest thing that works is X; I would move to Y only when I measured Z."
- "That buys us A at the cost of B — acceptable here because …"
- "The failure mode I would worry about is …, and I would detect it with …"
- "This is engine-specific: in PostgreSQL …, in MySQL …"
- "I have not used that in production; here is how I would reason about it."

## Experience stories to prepare

Have one real story per row, two minutes each: situation, your decision, the alternative you rejected and why, the measured result, what you would do differently.

| Theme | Prompt |
|---|---|
| Performance | A slow query or endpoint you diagnosed — what was the evidence, what was the fix, by how much did it improve? |
| Concurrency | A race condition, deadlock, or duplicate-processing bug you found in production |
| Architecture | A design decision with a real trade-off (consistency versus availability, build versus buy, sync versus async) |
| Failure | An incident you handled: detection, mitigation, root cause, prevention |
| Scale | Something that worked at small volume and broke at large volume |
| Disagreement | A technical disagreement and how it was resolved with evidence |

## In the room

- Restate the scenario and ask one or two clarifying questions before answering.
- Think aloud; the interviewer is assessing reasoning, not recall.
- Start simple, then add depth when asked — do not open with the most complex design.
- If you realise an earlier statement was wrong, correct it plainly.
- End answers with the trade-off or how you would verify it.
