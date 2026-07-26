# Databases — Senior Interview Guide

This chapter covers the database questions most often asked in senior Java interviews. It emphasizes relational databases and SQL while connecting them to application design, JDBC, distributed systems, and production troubleshooting.

> **How to answer:** state the general principle, explain the database mechanism, then acknowledge where behavior is vendor-specific.

## Contents

1. [Relational foundations and modeling](#1-relational-foundations-and-modeling)
2. [SQL and query semantics](#2-sql-and-query-semantics)
3. [Indexes](#3-indexes)
4. [Transactions and isolation](#4-transactions-and-isolation)
5. [Locking and concurrency](#5-locking-and-concurrency)
6. [Query optimization](#6-query-optimization)
7. [Java and database integration](#7-java-and-database-integration)
8. [Scaling, replication, and distributed data](#8-scaling-replication-and-distributed-data)
9. [Production scenarios](#9-production-scenarios)
10. [Rapid revision](#10-rapid-revision)

---

## 1. Relational foundations and modeling

### 1. What does ACID mean?

- **Atomicity:** a transaction's changes commit together or have no effect.
- **Consistency:** a committed transaction preserves declared constraints and application invariants.
- **Isolation:** concurrent transactions behave according to the selected isolation guarantees.
- **Durability:** committed changes survive the failures covered by the database's durability configuration.

ACID does not mean “all business data is automatically correct.” The database can enforce constraints it knows; the application must still define valid invariants. Durability also depends on storage, replication, and acknowledgement settings.

### 2. Primary key versus unique constraint?

A primary key uniquely identifies each row and is non-null. A table has one primary key, which may contain multiple columns. A unique constraint enforces another candidate key, and a table may have several.

Null treatment in unique constraints varies by database and configuration. Foreign keys can reference a primary key or another eligible unique key. Choose stable identifiers; changing a primary key can be operationally expensive because it propagates to indexes and references.

### 3. Natural key versus surrogate key?

A natural key comes from the business domain, such as a country code. A surrogate key is generated solely as an identifier, such as a sequence value or UUID.

Surrogate keys are stable and compact in relationships, but they do not remove the need for a unique constraint on the real business identity. Natural keys avoid duplicate identity but can be wide, mutable, or governed externally. A common design uses a surrogate primary key plus a unique business key.

### 4. What is normalization?

Normalization decomposes relations to reduce redundancy and update anomalies.

- **1NF:** attributes contain atomic values for the chosen model; no repeating groups.
- **2NF:** non-key attributes depend on the whole candidate key, not part of a composite key.
- **3NF:** non-key attributes do not depend transitively on a key through another non-key attribute.
- **BCNF:** every determinant is a candidate key.

Normalization is about dependencies and correctness, not blindly creating many tables.

### 5. When should a schema be denormalized?

Denormalize only for a measured access pattern or operational requirement—for example, a read model, precomputed aggregate, or avoiding an expensive distributed join. Define the source of truth, consistency model, update path, repair process, and acceptable staleness.

Denormalization trades read simplicity for write complexity. Copying columns without ownership and reconciliation rules creates silent drift.

### 6. What are foreign keys for?

A foreign key enforces referential integrity: referenced parent values must exist, subject to nullability and configured update/delete actions. It protects data across every writer, not only one application version.

Index foreign-key columns when parent deletion/update or child lookup performance requires it; some databases do not create this index automatically. Choose `CASCADE`, `RESTRICT`, `SET NULL`, or other actions from domain semantics, not convenience.

### 7. UUID versus numeric sequence as a primary key?

Numeric sequence keys are compact, ordered, index-friendly, and easy to debug, but require coordination or allocation strategies in distributed writers. Random UUIDs can be generated independently and are hard to guess, but consume more space and randomize B-tree insertion.

Time-ordered UUID variants improve locality but expose some timing/order information and still require collision and representation decisions. Avoid using public unpredictability as an authorization control. The right choice depends on write topology, index size, exposure, and migration needs.

### 8. How should money be stored?

Use an exact decimal type with intentional precision and scale, or an integer representing the smallest currency unit when its range and currency rules fit. Never use binary floating-point for exact monetary equality or accounting.

Store or otherwise know the currency, define rounding mode at business boundaries, and do not assume every currency has two fractional digits. Java should normally use `BigDecimal` with explicit scale and rounding policy.

### 9. How should date and time be stored?

Store an instant for an absolute event and preserve a zone identifier when future civil-time rules or the user's original zone matter. A local date/time without zone is appropriate for concepts such as “every day at 09:00 in Dubai,” not an already-occurring global event.

Understand the database type: “timestamp with time zone” does not necessarily retain the original zone name. Use UTC consistently for instants, but do not erase business time-zone semantics.

### 10. Soft delete versus hard delete?

Soft delete retains a row with a deletion marker; it supports recovery and audit-like use cases but complicates every query, uniqueness rule, foreign key, index, aggregate, and retention policy. It is not a substitute for a proper audit log.

Hard delete reduces live data and query complexity but may conflict with recovery or regulatory needs. Decide from domain retention requirements. For soft deletion, centralize filtering, index the live-row access pattern, and define eventual physical purging.

---

## 2. SQL and query semantics

### 11. What is SQL's logical query-processing order?

A useful conceptual order is:

1. `FROM` and `JOIN`
2. `WHERE`
3. `GROUP BY`
4. aggregate calculation
5. `HAVING`
6. window functions
7. `SELECT`
8. `DISTINCT`
9. `ORDER BY`
10. limit/offset

This explains why a select-list alias is often unavailable in `WHERE`, why `WHERE` cannot filter an aggregate, and why window-function results usually need an outer query to filter. The optimizer may physically execute an equivalent plan in a different order.

### 12. `WHERE` versus `HAVING`?

`WHERE` filters input rows before grouping; `HAVING` filters groups after aggregation. Put ordinary row predicates in `WHERE` so fewer rows reach grouping and indexes remain usable.

```sql
SELECT customer_id, COUNT(*)
FROM orders
WHERE created_at >= :start
GROUP BY customer_id
HAVING COUNT(*) >= 10;
```

### 13. Explain inner, outer, and cross joins.

- `INNER JOIN` returns matching combinations.
- `LEFT JOIN` preserves every left row and fills unmatched right columns with null.
- `RIGHT JOIN` is the mirrored form.
- `FULL OUTER JOIN` preserves unmatched rows from both sides where supported.
- `CROSS JOIN` returns the Cartesian product.

For an outer join, moving a predicate on the nullable side from `ON` to `WHERE` can eliminate null-extended rows and effectively turn it into an inner join.

### 14. `EXISTS` versus `IN` versus a join?

Use `EXISTS` when asking whether a related row exists; the optimizer can often implement it as a semi-join and stop after a match. Use a join when columns or multiplicity from both relations are needed. `IN` can be clear for a set of values and may optimize similarly.

`NOT IN` has a notorious null trap: if the subquery contains null, comparisons become unknown and may return no rows. Prefer `NOT EXISTS` with a correlated equality for anti-join semantics.

### 15. How does SQL null behave?

Null represents missing or unknown information and uses three-valued logic: true, false, and unknown. `column = NULL` is not true; use `IS NULL`. Most comparisons with null yield unknown, which `WHERE` filters out.

`COUNT(column)` ignores nulls, while `COUNT(*)` counts rows. Null ordering, concatenation, unique constraints, and null-safe comparison operators vary by database.

### 16. `UNION` versus `UNION ALL`?

`UNION ALL` concatenates results and preserves duplicates. `UNION` removes duplicates, usually requiring a sort or hash operation. Use `UNION ALL` unless duplicate elimination is part of the requirement.

The number and compatible types of output columns must align. A final `ORDER BY` applies to the combined result.

### 17. What are aggregate and window functions?

An aggregate with `GROUP BY` collapses rows into one row per group. A window function calculates across related rows while preserving each input row.

```sql
SELECT employee_id,
       department_id,
       salary,
       RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rank
FROM employee;
```

Window frames matter. The default frame with an ordered window may not mean the entire partition, so cumulative sums and `LAST_VALUE` deserve explicit frame definitions.

### 18. CTE versus subquery?

A common table expression gives a named intermediate result and improves readability; recursive CTEs express hierarchical or graph traversal. A subquery is often sufficient for a small local expression.

Do not assume a CTE is always materialized or always inlined. Optimizer behavior and explicit controls differ across databases and versions. Inspect the plan when performance matters.

### 19. Offset versus keyset pagination?

Offset pagination is simple and supports arbitrary page jumps, but the database still processes skipped rows, and concurrent inserts/deletes can cause duplicates or omissions between pages.

Keyset pagination seeks after the last stable sort key:

```sql
SELECT id, created_at, total
FROM orders
WHERE (created_at, id) < (:lastCreatedAt, :lastId)
ORDER BY created_at DESC, id DESC
LIMIT :size;
```

It scales better for sequential navigation. The ordering must be deterministic and backed by a suitable index; include a unique tie-breaker.

### 20. Delete duplicate rows while retaining one: how would you reason about it?

First define “duplicate” and which row survives. A common approach assigns `ROW_NUMBER()` over the duplicate key and deletes rows with a number above one.

Before deletion, preview the exact rows, take appropriate backup/recovery precautions, and then add a unique constraint so duplicates cannot return. Cleaning symptoms without enforcing the invariant is incomplete.

---

## 3. Indexes

### 21. How does a B-tree index work?

A balanced search tree stores ordered keys in pages, enabling logarithmic equality and range lookup and ordered traversal. Leaf entries point to rows or contain the storage engine's row locator; exact layout varies by database.

B-trees support equality, ranges, prefixes of compatible composite keys, ordering, and often min/max efficiently. They cost storage, memory, write amplification, and maintenance.

### 22. What is a composite index, and why does column order matter?

An index on `(a, b, c)` is ordered first by `a`, then `b` within equal `a`, then `c`. It commonly supports lookups using a leading prefix such as `(a)` or `(a, b)`. Once a broad range is used, later columns may be less useful for narrowing the scan, though they may still help filtering or covering.

Choose order from actual predicates, selectivity, range conditions, joins, and ordering—not simply “most selective first.” One composite index is not equivalent to several single-column indexes.

### 23. What is a covering index?

An index covers a query when it contains all values needed for filtering and output, allowing the engine to avoid or reduce table/heap access. Some databases support included columns that do not participate in key ordering.

Coverage is query-specific and visibility rules can still require table access. Wide covering indexes increase size and write cost, so use them for measured hot paths.

### 24. Why might a database ignore an index?

- The predicate matches a large portion of the table.
- The table is small, making a sequential scan cheaper.
- Statistics are stale or distributions are misestimated.
- A function or incompatible cast is applied to the indexed column.
- The predicate does not match the index's leading structure or operator class.
- Collation, type, or parameter estimates differ.
- Random table lookups cost more than one sequential pass.

“An index exists” does not mean using it is cheaper. Verify with the actual plan and runtime evidence.

### 25. What makes a predicate non-sargable?

A sargable predicate lets the engine use an index search condition. Wrapping an indexed column in a function or calculation can prevent a direct seek.

```sql
-- Often non-sargable
WHERE DATE(created_at) = :day

-- Searchable range
WHERE created_at >= :start AND created_at < :nextDay
```

Expression/function indexes can help when the expression itself is the stable access pattern. Leading-wildcard searches such as `LIKE '%term'` typically need a specialized index or search system.

### 26. Clustered versus non-clustered index?

A clustered organization determines or closely controls how table rows are stored by key, so there can be only one physical clustering order. A secondary/non-clustered index is a separate structure pointing to the row or primary-key entry.

Terminology and implementation vary. InnoDB organizes table data by primary key; PostgreSQL heaps are separate from indexes and `CLUSTER` is not continuously maintained. Always describe the engine rather than treating the terms as universal internals.

### 27. Hash index versus B-tree index?

Hash indexes are designed for equality lookup and do not provide key ordering or ordinary range scans. B-trees handle equality plus ranges and ordering, making them the general default.

Support, persistence, concurrency, and optimizer usage differ by engine. Other index families—bitmap, inverted, full-text, spatial, BRIN, GiST, or GIN—fit different data and operators.

### 28. What are partial and expression indexes?

A partial/filtered index contains rows satisfying a predicate, such as only active records. It can be much smaller and enforce conditional uniqueness. The query predicate must be compatible enough for the optimizer to use it.

An expression index stores a computed expression such as `lower(email)`, enabling efficient lookup and uniqueness on that normalized expression. Both trade write cost for a targeted access pattern.

### 29. Why can too many indexes hurt?

Every insert, delete, and relevant update must maintain each index. Indexes consume storage and cache, increase WAL/redo and replication traffic, lengthen maintenance, and give the optimizer more alternatives to estimate.

Find redundant and unused indexes with workload-aware statistics, but account for rare critical queries, constraint enforcement, and observation resets before removal.

---

## 4. Transactions and isolation

### 30. What anomalies can concurrent transactions produce?

- **Dirty read:** reading another transaction's uncommitted change.
- **Non-repeatable read:** rereading a row and seeing a committed change.
- **Phantom:** repeating a predicate and seeing a changed set of matching rows.
- **Lost update:** one update overwrites another without detecting it.
- **Write skew:** transactions read a shared condition and update different rows, jointly violating an invariant.

The first three are the classic SQL phenomena; real concurrency correctness also requires considering lost updates, write skew, and application invariants.

### 31. Explain the standard isolation levels.

| Level | Dirty read | Non-repeatable read | Phantom |
|---|---:|---:|---:|
| Read uncommitted | Possible | Possible | Possible |
| Read committed | Prevented | Possible | Possible |
| Repeatable read | Prevented | Prevented | Possible by the SQL model |
| Serializable | Prevented | Prevented | Prevented |

This table is a minimum standards model, not a full prediction of an engine. PostgreSQL's Read Uncommitted behaves like Read Committed, and its Repeatable Read prevents phantom reads but can still require retry handling. InnoDB uses MVCC and next-key locking with behavior influenced by statement type and isolation.

### 32. What is MVCC?

Multi-Version Concurrency Control retains row versions so readers can use a consistent snapshot while writers create new versions. It reduces reader/writer blocking but does not eliminate write conflicts, locks, deadlocks, or storage maintenance.

Long transactions can retain old versions, increase cleanup pressure, keep stale snapshots, and cause replication or vacuum-related issues depending on the engine. MVCC behavior is database-specific.

### 33. Optimistic versus pessimistic concurrency control?

Optimistic control assumes conflicts are uncommon and detects them at write time, often with a version column:

```sql
UPDATE account
SET balance = :newBalance, version = version + 1
WHERE id = :id AND version = :expectedVersion;
```

Zero updated rows means conflict. Pessimistic control locks data before modification, for example with `SELECT ... FOR UPDATE`. Optimistic control favors concurrency and retry; pessimistic control may suit hot contested rows but increases waiting and deadlock risk.

### 34. What is a lost update, and how do you prevent it?

Two transactions read the same value, calculate independently, and the later write overwrites the earlier one. Prevent it with an atomic SQL update, optimistic version check, row lock, or an isolation level/engine behavior that detects the conflict.

```sql
UPDATE counter SET value = value + 1 WHERE id = :id;
```

This is safer than read-increment-write in application code. Define retry behavior when the chosen mechanism reports a conflict.

### 35. What is write skew?

Two transactions read a shared invariant, then update different rows. Neither sees a direct write conflict, but together they violate the invariant—for example, both doctors independently going off-call after observing the other on-call.

Row-level optimistic versions may not detect it because different rows are updated. Solutions include serializable isolation with retry, locking a common invariant row/range, redesigning the data model, or expressing the invariant as an enforceable constraint.

### 36. What is autocommit?

With autocommit enabled, each standalone statement forms its own transaction unless an explicit transaction is started. This is safe for a single atomic statement but cannot make several application steps one unit.

Connection pools reuse sessions, so frameworks must reliably reset transaction state. Never assume two repository calls are atomic merely because both individually execute in transactions.

### 37. What are savepoints?

A savepoint marks a point inside a transaction to which later work can be rolled back without discarding earlier work. It does not create an independently durable nested transaction; the outer commit still decides durability.

Support and interaction with locks or framework “nested transactions” vary. Savepoints are useful for controlled partial recovery, but complicated transactional control may signal that the business operation should be decomposed.

### 38. Why are long-running transactions dangerous?

They hold locks and connections, increase contention and deadlock opportunities, retain MVCC versions, delay cleanup, enlarge recovery work, and may cause replication or log retention pressure.

Keep database transactions focused. Do not wait for user input, sleep, or perform slow remote calls while holding one. For workflows spanning time or services, use explicit state transitions and compensation rather than one open transaction.

---

## 5. Locking and concurrency

### 39. Shared versus exclusive locks?

A shared lock generally permits other shared holders but conflicts with an exclusive lock. An exclusive lock protects modification and conflicts with other incompatible access. Databases also use intention, key, gap, predicate, metadata, and advisory locks.

The exact compatibility matrix and whether ordinary reads take locks depend on the engine, isolation level, and MVCC implementation.

### 40. What does `SELECT ... FOR UPDATE` do?

It performs a locking read for rows selected by the query, preventing incompatible concurrent modifications until transaction end. Locks may cover scanned index entries, gaps, or predicates depending on the database and plan—not merely the final rows imagined by the application.

Use it within an explicit short transaction and with an appropriate index. Options such as `NOWAIT` or `SKIP LOCKED` are useful for particular workflows but change semantics.

### 41. What is a database deadlock?

Transactions form a cycle of waits—for example, T1 holds row A and waits for B while T2 holds B and waits for A. The database detects the cycle and aborts one transaction as the victim.

Prevent frequent deadlocks by accessing rows/tables in consistent order, keeping transactions short, using selective indexes, and locking only what is needed. Still treat deadlock as a normal transient failure and retry the complete transaction with bounded backoff when safe.

### 42. Blocking versus deadlock?

Blocking is a wait for a lock holder and may resolve when that transaction commits or rolls back. Deadlock is a cycle that cannot resolve without aborting a participant.

Investigate both the waiter and blocker. The query reported as waiting may be innocent; a forgotten idle-in-transaction session can be the real cause.

### 43. How can an index reduce locking?

A selective index allows an update or locking read to scan fewer index entries and rows, often reducing the lock footprint and transaction duration. Without a suitable index, some engines scan and lock many more records or ranges than the final result suggests.

This is a performance and concurrency concern. Validate the plan and the engine's locking behavior; do not assume the `WHERE` clause alone limits acquired locks.

### 44. What is an advisory lock?

An advisory lock is an application-defined lock managed by the database, usually identified by a key rather than automatically tied to row access. It can coordinate work for which no single natural row exists.

The database generally does not enforce that all writers honor it. Define scope, ownership, timeout, connection/transaction lifetime, failure behavior, and a stable ordering when acquiring multiple advisory locks.

---

## 6. Query optimization

### 45. How do you analyze a slow query?

1. Capture the actual SQL, bound values or value distribution, frequency, latency, and rows returned.
2. Inspect the execution plan and, safely, actual runtime statistics.
3. Compare estimated and actual row counts.
4. Find large scans, expensive joins/sorts, spills, repeated loops, lock waits, and I/O.
5. Check schema, indexes, statistics, data skew, and query shape.
6. Change one cause and validate under representative data and concurrency.

A fast isolated query can still overload production when executed thousands of times per request or under contention.

### 46. What does `EXPLAIN ANALYZE` do, and what is the risk?

`EXPLAIN` shows the optimizer's chosen plan and estimated costs/cardinality. The analyze option actually executes the statement and adds runtime measurements such as actual rows and timing.

Because it executes the query, use care with writes and expensive statements in production. A transaction rollback can protect data changes in some workflows but does not undo every external effect or eliminate resource impact.

### 47. Why are cardinality estimates important?

The optimizer estimates how many rows each operation will produce. Those estimates drive join order, join algorithm, scan type, parallelism, and memory decisions. A large estimate error early in a plan compounds downstream.

Causes include stale statistics, correlated columns, skew, expressions, parameter sensitivity, and rapidly changing data. Fix evidence and statistics/modeling before forcing a plan with hints.

### 48. Nested-loop, hash, and merge joins?

- **Nested loop:** for each outer row, find inner matches; excellent with a small outer input and indexed inner lookup.
- **Hash join:** build a hash table for one input and probe with the other; strong for large equality joins, but memory/spill matters.
- **Merge join:** consume inputs ordered by join key; effective for sorted/indexed large equality or range-compatible work.

There is no universally best join. Cardinality, ordering, memory, indexes, and data distribution decide.

### 49. What is parameter-sensitive plan behavior?

A prepared statement may reuse a plan generated from generic estimates or earlier parameter values. When data is skewed, a plan ideal for a rare value can be terrible for a common value, or vice versa. SQL Server often calls a related phenomenon parameter sniffing; other systems make different generic/custom plan choices.

Confirm through plans and distributions. Possible remedies include better statistics, query restructuring, partitioning, selective recompilation or engine-specific plan controls—chosen cautiously.

### 50. Why is `SELECT *` discouraged?

It transfers and materializes unnecessary data, increases network and memory use, may prevent covering-index access, couples callers to schema changes, and risks exposing new sensitive columns.

It is acceptable for deliberate exploration, but application queries should name their contract. Fewer selected columns do not automatically mean fewer database pages unless the plan can exploit that fact.

### 51. What are the risks of database functions and triggers?

They can enforce rules close to data and reduce round trips, but can hide behavior from application readers, complicate deployment/versioning, increase vendor coupling, and make latency or side effects surprising.

Triggers are appropriate for carefully owned invariants or auditing in some systems, but must handle bulk operations, recursion, ordering, failure, observability, and replication semantics. Use them intentionally, not as invisible application logic.

### 52. Partitioning versus indexing?

Partitioning divides a logical table into physical partitions, often by time, range, list, or hash. It helps lifecycle management and can prune irrelevant data, but it is not a replacement for indexes within relevant partitions.

A poor partition key can create hotspots, prevent pruning, complicate uniqueness and foreign keys, and produce too many partitions. Partition for a proven size, maintenance, or access requirement.

---

## 7. Java and database integration

### 53. Why use a connection pool?

Opening a database connection is expensive. A pool reuses a bounded set and caps concurrent database work. It must be sized with the database capacity, application replica count, workload, and transaction duration in mind—not the number of incoming requests alone.

Configure acquisition timeout, validation/lifetime, leak detection where useful, and metrics. More connections can increase database contention and reduce throughput.

### 54. What commonly exhausts a connection pool?

- Leaked connections or result sets.
- Slow queries or lock waits.
- Remote calls inside database transactions.
- Too much application concurrency.
- Long streaming results.
- Database overload or network failure.
- Pool size larger than the database can effectively serve.

Measure active, idle, pending, acquisition time, and hold time. Raising the maximum may merely move the queue into the database.

### 55. How does JDBC transaction management work?

With autocommit disabled, statements on the same `Connection` participate until `commit` or `rollback`. Always return a pooled connection in a clean state and use try-with-resources for statements and result sets.

Framework transaction managers bind a connection/resource to the execution context. Crossing threads normally loses that context. Multiple connections do not form one local transaction merely because they are used by the same method.

### 56. `Statement` versus `PreparedStatement`?

`PreparedStatement` separates SQL structure from bound data, prevents ordinary SQL injection through parameters, handles types correctly, and can enable statement/plan reuse. It cannot parameterize identifiers such as table names or sort direction; whitelist those choices and construct only the controlled fragment.

Prepared statements do not fix an unsafe SQL fragment concatenated before preparation.

### 57. What are JDBC batching and bulk loading?

Batching reduces network round trips by sending many parameter sets together. Correct batch size balances memory, packet limits, transaction size, generated-key needs, driver behavior, and lock duration.

For very large ingestion, database-native bulk loading is often faster. Preserve validation, failure reporting, deduplication, and restartability rather than making one enormous transaction.

### 58. ORM versus SQL?

An ORM maps object state and relationships to relational operations and is productive for transactional aggregate work. SQL-first access offers explicit control for reporting, bulk operations, complex joins, and database-specific features.

These are complementary. A senior engineer understands generated SQL, fetching, flush timing, transaction scope, batching, indexes, and query plans even when using JPA. Do not force every read model through managed entities.

### 59. How do schema migrations work safely?

Treat migrations as versioned, reviewed production code. In rolling deployments, use expand-and-contract:

1. Add backward-compatible schema.
2. Deploy code that can work across versions and backfill safely.
3. Switch reads/writes and verify.
4. Remove obsolete code and schema later.

Avoid combining a long table rewrite with a latency-sensitive deployment. Know the engine's locking behavior, test on production-scale data, make backfills resumable, and plan rollback or roll-forward.

### 60. What is the N+1 query problem?

One query loads N parent objects and then N additional queries load related data. It causes round-trip amplification and often passes small tests unnoticed.

Detect with query counts, tracing, logs, and production metrics. Fix per use case with joins, batch fetching, entity graphs, projections, or a dedicated query. Making all relationships eager usually introduces different performance problems.

---

## 8. Scaling, replication, and distributed data

### 61. Vertical versus horizontal database scaling?

Vertical scaling adds CPU, memory, faster storage, or capacity to one node and is operationally simple but has a ceiling. Horizontal scaling distributes work or data across replicas/shards and adds routing, consistency, failure, and rebalancing complexity.

Before sharding, optimize queries and indexes, archive unnecessary data, use appropriate caching, scale reads where semantics allow, and understand the actual bottleneck.

### 62. Synchronous versus asynchronous replication?

Synchronous replication waits for configured replicas before acknowledging commit, reducing data-loss risk but increasing latency and sensitivity to replica/network health. Asynchronous replication acknowledges earlier but allows lag and potential loss during failover.

“Synchronous” itself has levels: receipt, durable log flush, replay, or quorum. State the exact acknowledgement guarantee.

### 63. What is replication lag, and why does it matter?

A replica applies changes after the primary, so it may serve stale data. This causes read-after-write failures, apparently missing records, stale authorization, and inconsistent pagination.

Mitigations include reading critical paths from the primary, session stickiness, waiting for a replication position, bounded-staleness routing, or designing the UI/workflow to expose eventual consistency. Monitor lag in time and log position, including during incidents.

### 64. What is sharding?

Sharding partitions rows across independent database nodes using a shard key. The key should distribute load and storage while keeping common transactions and queries within one shard.

Sharding complicates cross-shard joins, uniqueness, transactions, secondary indexes, resharding, hot tenants, backup/restore, and operational tooling. Hashing distributes well but weakens range locality; range or tenant sharding preserves locality but can create hotspots.

### 65. CAP theorem: what is the interview-safe explanation?

When a network partition prevents nodes from communicating, a distributed system cannot simultaneously guarantee every request receives a non-error response and that every read observes a single up-to-date value. It must choose behavior for affected operations during the partition.

CAP is not a database product label or a choice to ignore partitions. Real systems choose per operation, and latency, durability, isolation, and behavior outside partitions require additional models.

### 66. What is eventual consistency?

Replicas or denormalized views may temporarily disagree but converge when updates stop and delivery/reconciliation succeeds. This is not a complete design until the system defines conflict resolution, ordering, retry, deduplication, convergence, and user-visible staleness.

Use domain language: which read can be stale, by how much, and what invariant must never be violated?

### 67. What is the transactional outbox?

Write the business change and an outbox event in the same local database transaction. A separate relay publishes outbox rows to a broker; consumers handle duplicates idempotently.

It closes the database-and-broker dual-write gap without a distributed transaction. It still requires reliable polling/CDC, ordering policy, retry, cleanup, observability, schema evolution, and idempotent consumers.

### 68. What are read replicas good for, and what are they not good for?

They can offload suitable read traffic, reporting, and backups while improving geographic access. They do not automatically increase write capacity and may be stale.

Replica reads are dangerous for read-after-write workflows, uniqueness checks, locks, or authorization decisions unless the consistency mechanism explicitly supports them. Also budget connection counts across every application replica and database replica.

### 69. SQL versus NoSQL?

Choose from the data model and access/consistency requirements, not fashion. Relational databases excel at constraints, joins, flexible querying, and multi-row transactions. Key-value, document, wide-column, graph, and search systems optimize different models and distribution patterns.

“NoSQL” is not one consistency model. Polyglot persistence adds operational and consistency cost; use another store when its capability materially justifies that cost.

---

## 9. Production scenarios

### 70. The database CPU is high. What do you investigate?

Identify which statements consume total time, CPU, calls, and rows—not only the single slowest query. Inspect plans, estimate errors, scans, sorts, spills, parallelism, compilation, and application call amplification.

Also check workload change, connection count, missing indexes, stale statistics, maintenance, replication, and host/storage metrics. Reducing one query from 50 ms to 5 ms matters more if it runs a million times than optimizing a rare one-second query.

### 71. The database has many active connections but low CPU. Why?

Connections may be waiting on locks, I/O, remote storage, pool/session state, or client consumption; they may also be idle in transaction. Low CPU does not mean spare query capacity.

Inspect database wait events, blockers, transaction ages, query state, pool acquisition/hold metrics, network, and storage latency. Categorize waits before changing connection limits.

### 72. How do you handle a hot row?

A single frequently updated row serializes writers—for example, one global counter or balance. Options include atomic updates, sharded counters with later aggregation, append-only events, per-entity partitioning, batching, queues, optimistic retry, or redesigning the invariant.

The correct solution depends on whether every read needs an exact current value. Caching cannot safely remove the serialization point when the database must enforce a strict global invariant.

### 73. How do you run a large backfill safely?

Make it resumable and idempotent. Process bounded key ranges or keyset pages, commit small batches, throttle against database health, avoid long snapshots, record progress, and expose metrics and cancellation.

Test lock and log/replication impact. Do not use offset pagination over a changing table. Validate counts and invariants afterward, and keep new writes compatible during the migration.

### 74. How do you design database backups?

Define recovery point objective (acceptable data loss) and recovery time objective (acceptable restoration time). Use a combination of full/incremental backups and transaction-log archiving as supported, encrypt and control access, store copies across failure domains, and define retention.

A backup is not proven until restoration is regularly tested. Test point-in-time recovery, credentials, application consistency, large-dataset timing, and operational runbooks.

### 75. What should be monitored?

- Query latency, throughput, errors, and top statements.
- CPU, memory/cache, storage latency, IOPS, and capacity.
- Connections, transactions, wait events, locks, and deadlocks.
- Long/idle transactions and vacuum/cleanup health.
- Replication lag and log growth.
- Buffer/cache hit indicators interpreted with workload context.
- Pool acquisition, active count, timeouts, and hold duration from applications.
- Backup freshness and restore-test results.

Alert on user impact and resource saturation with actionable context, not every fluctuating counter.

### 76. What distinguishes a senior database answer?

- Starts from invariants and access patterns.
- Treats constraints as correctness tools, not inconveniences.
- Separates the SQL standard model from engine-specific behavior.
- Reads execution plans and validates estimates with actual data.
- Explains MVCC without claiming it eliminates locks.
- Makes transactions short and retries transient conflicts safely.
- Treats connections, indexes, logs, and replicas as finite resources.
- Plans migrations, backups, failover, and observability before incidents.

---

## 10. Rapid revision

### Must-answer questions

Before an interview, answer these without notes:

1. What does each ACID property guarantee?
2. Primary key versus unique constraint?
3. Natural versus surrogate key?
4. Why normalize, and when denormalize?
5. Why keep foreign keys?
6. How does null affect comparisons and `NOT IN`?
7. `WHERE` versus `HAVING`?
8. `EXISTS` versus a join?
9. Offset versus keyset pagination?
10. How does a B-tree index work?
11. Why does composite-index order matter?
12. What is a covering index?
13. Why might the optimizer ignore an index?
14. What makes a predicate non-sargable?
15. What anomalies exist beyond dirty/non-repeatable/phantom reads?
16. What is MVCC?
17. Lost update versus write skew?
18. Optimistic versus pessimistic locking?
19. Why are long transactions harmful?
20. Blocking versus deadlock?
21. How do you diagnose a slow query?
22. What do estimated versus actual rows tell you?
23. Nested-loop versus hash versus merge join?
24. How should a connection pool be sized?
25. Why use `PreparedStatement`?
26. How do you deploy a breaking schema change safely?
27. What causes N+1 queries?
28. How does replication lag affect application behavior?
29. What makes a good shard key?
30. How does the transactional outbox solve dual writes?
31. How do you perform a safe backfill?
32. Why must backups be restore-tested?

### Thirty-second summary

A relational database protects declared invariants through constraints and transactions. Indexes trade storage and write cost for particular access paths; their usefulness depends on ordering, selectivity, and query shape. Isolation and MVCC control concurrency but remain engine-specific and do not eliminate conflicts, locks, or retries. Efficient systems keep transactions short, pools bounded, SQL observable, and plans validated with real cardinalities. Scaling through replicas, partitioning, or sharding adds consistency and operational trade-offs that application design must expose explicitly.

## Official references

- [PostgreSQL SQL language documentation](https://www.postgresql.org/docs/current/sql.html)
- [PostgreSQL concurrency control](https://www.postgresql.org/docs/current/mvcc.html)
- [PostgreSQL indexes](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL query planning](https://www.postgresql.org/docs/current/performance-tips.html)
- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/)
- [MySQL InnoDB transaction model](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-model.html)

