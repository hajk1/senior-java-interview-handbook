# Microservices — Senior Interview Guide

This chapter covers the microservices questions most often asked in senior Java interviews. Strong answers do not merely name patterns; they explain when distribution is justified, which failure modes it introduces, and how correctness and operability are preserved.

> **How to answer:** begin with the business or system requirement, describe the mechanism, and finish with failure behavior and trade-offs.

## Contents

1. [Architecture and service boundaries](#1-architecture-and-service-boundaries)
2. [Communication and API design](#2-communication-and-api-design)
3. [Resilience](#3-resilience)
4. [Data and distributed consistency](#4-data-and-distributed-consistency)
5. [Messaging and event-driven systems](#5-messaging-and-event-driven-systems)
6. [Discovery, gateways, and configuration](#6-discovery-gateways-and-configuration)
7. [Security](#7-security)
8. [Observability and operations](#8-observability-and-operations)
9. [Testing and delivery](#9-testing-and-delivery)
10. [Production scenarios](#10-production-scenarios)
11. [Rapid revision](#11-rapid-revision)

---

## 1. Architecture and service boundaries

### 1. What is a microservice?

A microservice is an independently deployable service aligned to a cohesive business capability, with explicit contracts and ownership of its data and operational lifecycle. “Small,” “REST,” or “runs in a container” is not sufficient.

Independence is the important property: a team should be able to change and release the service without coordinated deployment of the entire system. That independence has costs in networking, consistency, testing, observability, security, and operations.

### 2. Monolith versus modular monolith versus microservices?

| Style | Deployment | Internal boundaries | Typical advantage | Main risk |
|---|---|---|---|---|
| Monolith | One unit | Often informal | Simplicity | Tight coupling as it grows |
| Modular monolith | One unit | Explicit modules | Strong boundaries without distribution | Boundaries may erode |
| Microservices | Independent units | Network contracts | Independent scaling and delivery | Distributed complexity |

A modular monolith is often the best starting point. Choose microservices when independent team ownership, release cadence, scaling, fault isolation, or regulatory boundaries justify the operational cost—not because the organization expects future growth.

### 3. What are the main benefits and costs of microservices?

Benefits can include independent deployment, technology and scaling choices, clearer ownership, smaller change surfaces, and fault isolation. Costs include network failures, eventual consistency, duplicated platform work, contract evolution, cross-service debugging, security expansion, and higher infrastructure and cognitive load.

A senior answer quantifies the problem being solved. Splitting one deployment into ten services does not automatically create ten independent teams or reliable boundaries.

### 4. How do you identify service boundaries?

Start from business capabilities and domain language, not database tables or technical layers. Domain-driven design concepts such as bounded contexts help identify where models, rules, terminology, ownership, and rates of change differ.

Good boundaries minimize chatty cross-service workflows and keep invariants local. Signals of a poor boundary include shared tables, lockstep releases, constant synchronous calls, duplicated business decisions, and changes that touch several services every time.

### 5. What is a bounded context?

A bounded context defines the boundary within which a domain model and its language have a precise meaning. The same real-world concept can have different models in different contexts—for example, “customer” in sales, billing, and support.

A bounded context is a useful candidate for service ownership, not a rule that each context must become one service. Several contexts can live in a modular monolith, and one large context may later be decomposed around subdomains or workload boundaries.

### 6. Why should each service own its data?

Data ownership preserves autonomy: other services use the owner's contract rather than coupling to its schema and internal rules. Direct cross-service table access bypasses authorization, validation, evolution, and operational ownership.

“Database per service” means exclusive logical ownership, not necessarily a dedicated database server. Cross-service views require API composition, events, replicated read models, or analytical pipelines—with explicit consistency semantics.

### 7. When are microservices the wrong choice?

They are usually a poor choice when the domain and team boundaries are unclear, one small team owns everything, delivery is infrequent, operations are immature, strong cross-domain transactions dominate, or the application has not demonstrated independent scaling needs.

Distribution amplifies unclear design. A well-structured monolith can later be extracted; a distributed monolith combines monolithic coupling with network failure.

### 8. What is a distributed monolith?

It is a set of separately deployed services that cannot be changed, tested, or released independently. Common symptoms include shared databases, cyclic synchronous dependencies, coordinated releases, shared domain entities, and one request traversing many services.

Fix the coupling rather than adding more infrastructure: clarify ownership, stabilize contracts, make dependencies one-directional, move invariants together, introduce asynchronous flows where appropriate, or merge services whose separation has no value.

### 9. How do you decompose a monolith safely?

Use incremental extraction, commonly the strangler pattern:

1. Identify a cohesive capability with measurable value.
2. Establish an internal module boundary and tests.
3. Define the new service contract and data ownership.
4. Route selected traffic to the new implementation.
5. Migrate data with compatibility and reconciliation.
6. Observe, expand traffic, and remove the old path.

Avoid a big-bang rewrite. Start with a boundary that is valuable but not the most coupled mission-critical core.

### 10. Orchestration versus choreography?

Orchestration uses a coordinator that tells participants what action to perform and tracks workflow state. It offers visibility and centralized control but can become a logic-heavy central dependency.

Choreography has services react to events without a central coordinator. It reduces direct coupling but can hide the end-to-end workflow and produce event cycles or unclear ownership. Use orchestration for explicit multi-step business processes; use choreography for loosely coupled reactions and fact distribution.

---

## 2. Communication and API design

### 11. Synchronous versus asynchronous communication?

Synchronous communication gives an immediate response and simple request semantics, but couples caller availability and latency to the callee. Asynchronous messaging decouples time and absorbs bursts, but introduces eventual consistency, duplicates, ordering concerns, and harder debugging.

Choose per interaction. A user query may require synchronous response; fulfillment after order acceptance may be asynchronous. Avoid replacing every call with messaging or building long synchronous chains for workflows that do not require immediate completion.

### 12. REST versus gRPC?

REST over HTTP is widely interoperable, human-readable when JSON-based, cache-friendly, and convenient for public APIs. gRPC uses strongly typed contracts and efficient binary framing, supports streaming, and is effective for controlled service-to-service communication.

Trade-offs include browser/tooling compatibility, schema evolution, observability, payload size, latency, and organizational familiarity. Protocol choice does not fix a chatty service boundary.

### 13. What makes a good service API?

- Models a business capability rather than exposing database CRUD mechanically.
- Has explicit request, response, error, and authorization semantics.
- Uses stable identifiers and deterministic behavior.
- Defines idempotency, pagination, deadlines, and size limits.
- Supports backward-compatible evolution.
- Exposes enough information to diagnose and operate safely without leaking internals.

An API is a long-lived product contract, even when only internal clients use it.

### 14. How do you evolve an API without breaking clients?

Prefer additive changes: add optional fields, new endpoints, or new event fields with safe defaults. Readers should ignore unknown fields where the format allows; writers should not assume newly introduced fields are understood everywhere.

Use consumer telemetry, deprecation windows, contract tests, and explicit ownership. Version only for genuinely incompatible semantics. Supporting many permanent versions creates operational cost; migrate clients and retire old contracts deliberately.

### 15. What is idempotency?

An operation is idempotent when repeating the same request has the same intended effect as performing it once. Network timeouts make the caller uncertain whether an operation completed, so retryable writes need an idempotency strategy.

For a create/payment operation, accept a scoped idempotency key, atomically persist it with the outcome, and return the same outcome on retry. Define key lifetime, payload mismatch behavior, concurrent in-progress requests, and failure states. A check-then-insert without a unique constraint is racy.

### 16. How do deadlines and timeouts differ?

A timeout is a duration allowed for one operation. A deadline is an absolute point by which the overall request must complete. Propagating the remaining deadline prevents downstream calls from consuming time the caller no longer has.

Set connection and response timeouts explicitly. A chain of services each using the caller's full timeout can exceed the user request budget. Account for queueing, retries, network latency, and cleanup.

### 17. What is API composition?

An aggregator calls multiple services and combines their data for one client response. It centralizes client complexity but adds fan-out latency and partial-failure handling.

With parallel independent calls, total latency trends toward the slowest dependency rather than the sum, but the probability of at least one failure rises with fan-out. For frequent or high-scale reads, a precomputed read model may be more reliable than live composition.

### 18. What are backward and forward compatibility?

Backward compatibility means a newer provider or reader continues to support older clients/data. Forward compatibility means an older reader can tolerate newer data or messages, usually by ignoring unknown optional fields.

Compatibility depends on semantics, not only whether deserialization succeeds. Renaming meanings, changing defaults, narrowing valid values, or making an optional field required can break consumers without changing field types.

### 19. How should errors be represented across services?

Use a stable machine-readable error code, safe human message, correlation/trace identifier, and appropriate protocol status. Distinguish validation, authentication, authorization, conflict, throttling, dependency failure, and unexpected server failure.

Do not expose stack traces, SQL, internal topology, or secrets. A caller should retry only based on explicit semantics, not every `500`. Preserve detailed internal cause in correlated telemetry.

### 20. How do you prevent chatty service communication?

Align boundaries with use cases, design coarse business operations, batch requests, cache stable reference data, replicate read models through events, and avoid remote entity navigation.

Do not merely add an aggregator over a fundamentally fragmented invariant. If two services must synchronously coordinate for nearly every operation, reconsider whether they belong apart.

---

## 3. Resilience

### 21. Why are timeouts mandatory?

Without a timeout, a caller can hold threads, connections, memory, and user requests indefinitely while a dependency stalls. Enough stuck calls exhaust pools and spread failure upstream.

Choose timeouts from the end-to-end latency objective and measured dependency distribution, including connection establishment. Extremely tight timeouts cause false failures during ordinary tail latency; extremely long ones defeat containment.

### 22. When should a request be retried?

Retry only a transient failure, only when the operation is safe or idempotent, and only while the caller's deadline and retry budget remain. Examples may include connection establishment failure, throttling, or selected `5xx` responses, depending on contract.

Do not retry validation, authorization, or permanent conflicts. Retrying at several layers multiplies attempts: three retries across three layers can create up to 27 calls. Choose one responsible layer.

### 23. Why use exponential backoff and jitter?

Exponential backoff spaces repeated attempts, giving a dependency time to recover. Jitter randomizes schedules so many clients do not retry simultaneously and create synchronized retry waves.

Cap delay and total attempts, honor server retry hints when trustworthy, and use a retry budget. Backoff reduces pressure; it does not make an overloaded dependency healthy.

### 24. What is a circuit breaker?

A circuit breaker monitors failures or slow calls and temporarily rejects new requests when a threshold is exceeded. After a wait, it permits limited probes to see whether the dependency recovered.

States are commonly closed, open, and half-open. Configure thresholds from meaningful failures and traffic volume; low-volume endpoints need care. A breaker protects resources and speeds failure—it does not repair the dependency or guarantee a valid fallback.

### 25. What is a bulkhead?

A bulkhead partitions resources so one failing dependency or workload cannot consume all threads, connections, queue space, or concurrency. Examples include separate executor pools or per-dependency semaphores.

Isolation can reduce resource efficiency and adds sizing decisions. Monitor saturation and rejection. A shared unbounded queue defeats fault containment even if services are logically separate.

### 26. What is rate limiting?

Rate limiting controls admitted work by identity, tenant, endpoint, or resource. Token bucket permits bursts while enforcing an average rate; leaky bucket smooths output; fixed and sliding windows offer other accuracy/cost trade-offs.

Return a clear throttling response and retry guidance where appropriate. Distributed limits require consistency choices. Rate limiting protects capacity; quotas express longer-term entitlement, and concurrency limiting bounds in-flight work.

### 27. What are load shedding and backpressure?

Load shedding rejects lower-priority or excess work early when the system cannot serve it within useful latency. Backpressure signals producers to slow down when consumers cannot keep up.

Bound queues and define rejection semantics; an unbounded queue hides overload until latency and memory collapse. Prioritize critical work, propagate deadlines, and recover gradually to avoid another surge.

### 28. What is graceful degradation?

The system preserves its most important capability when optional dependencies fail—for example, serving product details without recommendations. A fallback must be semantically safe and observable.

Never fabricate authoritative data or silently bypass authorization. Define which features may be stale, absent, or rejected, and test degraded modes before incidents.

### 29. What is a retry storm?

An overloaded dependency becomes slow or fails; callers retry, increasing load, causing more failures and more retries. Multiple retrying layers and synchronized delays amplify the feedback loop.

Use bounded retries, jitter, retry budgets, admission control, circuit breakers, and load shedding. During recovery, ramp traffic rather than releasing every queued request at once.

### 30. How should resilience mechanisms be ordered?

There is no universal wrapper order, but the semantics must be intentional. An overall deadline should bound everything. Rate/concurrency limits protect admission. A retry may count each attempt through a breaker, and the breaker should observe meaningful attempt outcomes. Bulkheads isolate actual calls.

Test configuration as a system: timeout × attempts must fit the deadline, pools must align with dependencies, and metrics must distinguish logical requests from physical attempts.

---

## 4. Data and distributed consistency

### 31. Why is a distributed transaction difficult?

Atomic commitment across independent resources requires coordination despite process and network failure. Two-phase commit can provide strong guarantees with capable participants, but introduces blocking risk, coordinator complexity, longer-held resources, and operational coupling.

Many microservice workflows instead use local transactions, durable messages, idempotency, and compensation. This changes the model to explicit eventual consistency; it does not recreate one invisible ACID transaction.

### 32. What is a saga?

A saga is a sequence of local transactions where each step commits independently and later failure triggers compensating actions for completed steps. It can be orchestrated or choreographed.

Compensation is a new business action, not a database rollback. It may fail, may not exactly erase real-world effects, and must be idempotent and observable. Persist saga state so the workflow survives restarts.

### 33. What is the transactional outbox?

The service writes its domain change and an outbox message in the same local database transaction. A relay publishes the outbox record to the broker, often by polling or change-data capture.

This removes the database/broker dual-write gap. Publication can still occur more than once, so consumers must deduplicate or be naturally idempotent. Define ordering key, retries, retention, schema evolution, and monitoring of outbox age.

### 34. What is the inbox pattern?

A consumer stores the message identifier in an inbox/deduplication table in the same local transaction as its business effect. A unique constraint ensures redelivery does not repeat the effect.

Define identifier scope and retention. If deduplication expires before a delayed duplicate arrives, the operation must remain safe or the delivery contract must bound that delay.

### 35. What is eventual consistency?

Replicas or service views can temporarily disagree but converge after updates and retries complete. A useful design states which data can be stale, for how long, how conflicts resolve, and which invariants remain immediate.

“Eventually consistent” is not permission to lose updates or leave failures unobserved. Provide reconciliation, progress state, retry/dead-letter handling, and user-visible workflow semantics.

### 36. How do you maintain a cross-service invariant?

First try to place data participating in a strict invariant under one owner and transaction boundary. If it must span services, reserve/confirm resources, serialize through an owning service, use a saga, or redesign the guarantee.

Examples such as “never oversell inventory” need an authoritative reservation mechanism. A stale cached availability check cannot enforce the invariant.

### 37. What is CQRS?

Command Query Responsibility Segregation separates write and read models. It can range from distinct code paths over one store to independently persisted read projections fed by events.

CQRS is useful when reads and writes have substantially different models, scale, or authorization needs. It adds projection lag, duplication, rebuild logic, and operational complexity. Ordinary CRUD does not require it.

### 38. What is event sourcing?

Event sourcing stores an aggregate's accepted state changes as the source of truth and reconstructs state by replay. It supports audit history, temporal queries, and new projections.

It also introduces event-version evolution, snapshotting, replay cost, privacy/deletion challenges, debugging complexity, and strict aggregate ordering. An audit log or event-driven integration is not automatically event sourcing.

### 39. How do you reconcile inconsistent data?

Make inconsistency detectable: compare counts, hashes, versions, or business invariants; retain durable identifiers and provenance. Run idempotent repair jobs that can resume, throttle, and report progress.

Reconciliation is part of the design for eventually consistent systems, not an emergency afterthought. Define the authoritative source and avoid blindly overwriting newer correct state.

### 40. How do read models stay current?

Consume versioned domain events or change-data capture, update projections idempotently, track offsets/checkpoints, and expose lag. Preserve per-entity ordering where required and handle poison messages without blocking unrelated keys forever.

Support rebuild from a durable source or trusted snapshot. A read model should be disposable only if its complete source history and rebuild capacity truly exist.

---

## 5. Messaging and event-driven systems

### 41. Queue versus publish/subscribe?

A work queue distributes messages among competing consumers so one logical worker handles each item. Publish/subscribe delivers a copy to each interested subscription or consumer group.

Products often support both models through different constructs. Define the business delivery requirement rather than selecting by product label.

### 42. At-most-once, at-least-once, and exactly-once?

- **At-most-once:** messages may be lost but are not redelivered by the mechanism.
- **At-least-once:** messages are retried, so duplicates are possible.
- **Exactly-once:** one logical effect is guaranteed within a precisely defined boundary.

Exactly-once is never a universal end-to-end property merely because a broker offers transactions. External database, email, payment, and HTTP effects need their own atomicity or idempotency strategy.

### 43. How do you make a consumer idempotent?

Use a unique business operation key, conditional update, state-machine transition, or inbox record committed atomically with the effect. The message ID alone may be insufficient if the same business command can arrive with different transport IDs.

Avoid check-then-act races. Preserve enough result state to respond consistently to duplicates, and define deduplication retention.

### 44. How is message ordering achieved?

Ordering is usually guaranteed only within a partition, queue, session, or key—not across an entire distributed topic at arbitrary scale. Route events for the same aggregate to the same ordering key and include an aggregate sequence/version.

Consumers should detect gaps, stale events, and duplicates. Global ordering is expensive and often unnecessary; identify the smallest domain scope requiring order.

### 45. What is a consumer group?

In partitioned logs such as Kafka, consumers with the same group coordinate ownership of partitions so each partition is processed by one group member at a time. Different groups independently consume the topic.

Parallelism is bounded by partition count. Membership changes trigger reassignment, so processing, offset commits, and side effects must tolerate revocation and redelivery.

### 46. When should an offset be committed?

Commit only after the corresponding processing guarantee is satisfied. Committing before the side effect risks loss; committing after a non-idempotent effect risks duplication if the process fails before the commit.

Coordinate the side effect and progress atomically where possible, or make processing idempotent. Automatic commits can be acceptable only when their semantics match the workload.

### 47. What is a dead-letter queue?

A dead-letter queue or topic holds messages that exceed retry policy or cannot be processed. It prevents one poison message from blocking unrelated work, but it is not a disposal bin.

Include failure reason and original metadata, alert on growth, protect sensitive payloads, and provide a controlled replay workflow. Separate transient dependency failure from permanently invalid data before dead-lettering.

### 48. How do you handle poison messages?

Validate input, classify the failure, retry transient problems with bounded backoff, and quarantine permanent failures. Preserve the original message and diagnostic context while avoiding secret leakage.

For ordered partitions, decide whether later messages may proceed. Skipping one event can make subsequent state invalid, so repair or per-key isolation may be required.

### 49. How do event schemas evolve?

Use additive optional fields, stable meanings, tolerant readers, explicit defaults, compatibility validation, and a schema registry when appropriate. Never repurpose a field to mean something else.

Events may live longer than services. Plan for replay of old versions, consumer upgrade order, sensitive-data retention, and migration when a semantic break is unavoidable.

### 50. Event notification versus event-carried state transfer?

A notification says that something changed and requires the consumer to query the owner. It keeps payloads small but creates synchronous coupling and request bursts.

Event-carried state includes enough data for consumers to update their own views without calling back. It improves autonomy but duplicates data, expands privacy/schema obligations, and can send data unused by many consumers.

### 51. What is backpressure in messaging?

When consumers fall behind, the system must bound in-flight work and expose lag rather than continuously loading more into memory. Consumers can pause partitions, reduce prefetch, cap concurrency, or signal publishers where supported.

Scaling consumers helps only until partitions or downstream resources become the bottleneck. Protect the database and external APIs with concurrency limits even when the broker backlog is large.

### 52. How should message retries be designed?

Avoid hot-loop retries on the main partition. Use delayed retry topics/queues, broker redelivery delays, or scheduled retry state with increasing backoff and jitter. Retain attempt count and original ordering identity.

Set a maximum based on business deadline, then dead-letter or move to manual recovery. A message hours past its useful time may need expiration rather than execution.

---

## 6. Discovery, gateways, and configuration

### 53. What is service discovery?

Service discovery maps a logical service name to healthy instances. Client-side discovery selects an instance in the client; server-side discovery routes through a load balancer or proxy. Platforms such as Kubernetes provide DNS and service abstractions.

Discovery indicates reachability, not application correctness. Health signals, stale endpoints, load balancing, locality, and connection reuse still matter.

### 54. What is an API gateway?

An API gateway is an external entry point that can route requests, terminate TLS, authenticate, enforce rate limits, transform protocols, and provide edge observability.

Keep domain business logic out of the gateway. It can become a bottleneck and organization-wide coupling point. Use multiple gateways or backend-for-frontend patterns when client needs differ substantially.

### 55. API gateway versus service mesh?

An API gateway manages north-south traffic between clients and the system, including client-facing policies. A service mesh manages east-west service traffic through proxies and control-plane policy, often providing mTLS, traffic management, and telemetry.

They overlap but solve different primary boundaries. A mesh adds operational complexity and does not define application-level idempotency, authorization semantics, or resilience policy automatically.

### 56. How should configuration be managed?

Keep environment-specific configuration outside the artifact, validate it at startup, version changes where possible, and expose safe effective configuration for diagnosis. Separate ordinary configuration, secrets, and dynamic feature policy.

Dynamic changes need schema, rollout, rollback, audit, and failure behavior. A central configuration outage should not necessarily stop healthy services from using their last known valid configuration.

### 57. What are feature flags?

Feature flags separate deployment from release and enable gradual rollout, experiments, and rapid disablement. Evaluate flags consistently for the required scope and define safe defaults when the flag service fails.

Flags create branching complexity. Give each flag an owner, purpose, expiry date, observability, and removal plan. Do not use a flag to bypass authorization or permanently replace configuration management.

### 58. Client-side versus server-side load balancing?

Client-side balancing lets the caller select among discovered instances and can use rich locality/health information, but duplicates logic across clients. Server-side balancing centralizes policy behind a proxy/load balancer but adds a network hop and infrastructure dependency.

Connection pooling can defeat naive per-request balancing because traffic remains on established connections. Consider protocol, long-lived streams, zone locality, health, and slow-start behavior.

---

## 7. Security

### 59. How should authentication and authorization work across services?

Authenticate the external caller at the edge, propagate only trustworthy identity context, and authorize each sensitive action at the service that owns it. Service-to-service identity is distinct from end-user identity; preserve both when delegation is required.

Do not trust arbitrary identity headers from clients. Validate tokens or use a trusted gateway-to-service mechanism, enforce audience and issuer, and apply least privilege.

### 60. What is zero trust in a microservice environment?

Do not grant trust merely because traffic is inside a network. Authenticate workloads and users, authorize each request from explicit policy, encrypt traffic, minimize credentials and privileges, and continuously observe access.

Zero trust is an architecture principle, not simply enabling mTLS. Compromised authorized services, excessive permissions, insecure application logic, and data exfiltration remain threats.

### 61. What does mTLS provide?

Mutual TLS encrypts transport and lets both peers authenticate certificates. It supports workload identity and protects against network interception when issuance, validation, rotation, and private keys are managed correctly.

mTLS does not provide user-level authorization or prove that a permitted service is entitled to a particular record. Map workload identity to least-privilege policy and handle certificate rotation without outages.

### 62. How should secrets be managed?

Store secrets in a dedicated secrets system or platform facility, grant access through workload identity, rotate them, audit use, and avoid exposing them in source control, images, logs, URLs, or telemetry.

Prefer short-lived credentials. Applications must handle rotation and temporary secret-store unavailability. Encoding a value or placing it in an environment variable does not by itself make it secure.

### 63. OAuth 2.0 versus OpenID Connect?

OAuth 2.0 is an authorization framework for delegated access. OpenID Connect adds identity/authentication semantics on top of OAuth 2.0, including an ID token and user information.

An access token is for the resource server and must be validated for issuer, audience, signature, expiry, and policy. An ID token is for the client and should not be used as a general API access token.

### 64. How do you protect sensitive data in events and logs?

Minimize collection, classify fields, apply authorization at production and consumption, encrypt in transit and at rest, restrict retention, and audit access. Redact tokens, credentials, personal data, and payment information from logs.

Events are widely replicated and retained, making deletion and access revocation difficult. Publish identifiers or purpose-limited data rather than entire domain objects when consumers do not need them.

---

## 8. Observability and operations

### 65. Monitoring versus observability?

Monitoring tracks known conditions through predefined metrics and alerts. Observability is the ability to infer internal system state from emitted signals and investigate novel failures.

Logs, metrics, and traces are complementary: metrics reveal aggregate behavior, traces show request paths and latency, and logs capture discrete contextual events. Instrument business outcomes as well as infrastructure.

### 66. What are RED and USE metrics?

For request-driven services, RED means **Rate, Errors, Duration**. For resources, USE means **Utilization, Saturation, Errors**. These provide practical starting points, not a complete monitoring strategy.

Measure latency distributions rather than averages, distinguish logical requests from retries, and watch bounded pools and queues. Add domain signals such as orders accepted, payments pending, and message age.

### 67. What is distributed tracing?

A trace follows one request or workflow through services; spans represent operations and carry timing, status, and attributes. Context propagation connects child operations across process boundaries.

Propagate trace context through HTTP and messages, but do not blindly trust or log baggage. Sampling controls cost; head sampling can miss rare failures, while tail sampling requires backend coordination. Traces complement, not replace, metrics and logs.

### 68. What are SLIs, SLOs, and error budgets?

- **SLI:** a measured indicator of service behavior, such as successful eligible requests.
- **SLO:** a target for the SLI over a time window.
- **Error budget:** the allowed unreliability implied by the SLO.

Choose user-visible indicators and define eligibility precisely. Use burn-rate alerts to detect rapid and sustained budget consumption. A 100% target is usually unrealistic and discourages honest engineering trade-offs.

### 69. Liveness versus readiness?

Liveness answers whether the process is stuck and should be restarted. Readiness answers whether the instance should receive traffic. A temporary downstream failure should often make an instance unready only when it truly cannot serve its contract; making it fail liveness can cause restart storms.

Keep probes cheap and independent enough to avoid synchronized failure. Startup probes or sufficient initial delay protect slow initialization. Readiness should fail before graceful shutdown drains traffic.

### 70. How does graceful shutdown work?

Mark the instance unready, stop accepting new work, allow load-balancer propagation, drain in-flight requests within a deadline, stop consumers safely, and close resources. The platform's termination grace period must exceed the application's drain plan.

Message consumers must coordinate partition revocation, offset commits, and redelivery. Shutdown should be idempotent and bounded; exactly-once assumptions should be tested under forced termination.

### 71. What makes a useful log?

Use structured fields with timestamp, severity, service/version, stable event name, trace/span IDs, and relevant non-sensitive business identifiers. Log state transitions and decisions, not every method entry.

Avoid high-volume duplicate stack traces and secrets. Sampling routine success logs may be appropriate, but preserve errors and audit events according to policy. Ensure a trace can lead to related logs.

### 72. What is metric cardinality, and why is it dangerous?

Cardinality is the number of unique attribute combinations in a metric. Labels such as user ID, request ID, raw URL, or unbounded error text create many time series, consuming client and backend memory and increasing cost.

Use bounded dimensions for metrics and place high-cardinality details in traces or logs. Normalize routes rather than recording raw paths. Cardinality limits protect the system but can hide detailed groupings once overflow occurs.

---

## 9. Testing and delivery

### 73. What is the microservice testing pyramid?

Use many fast unit tests, focused component/service tests, integration tests with realistic infrastructure, contract tests at service boundaries, and a small number of end-to-end tests for critical journeys.

An end-to-end-heavy suite is slow and brittle; an all-mocked suite cannot prove integration semantics. Test failure paths—timeouts, duplicates, stale data, partial completion, and shutdown—not only the happy path.

### 74. What is consumer-driven contract testing?

Consumers publish examples or expectations of how they use a provider; the provider verifies those contracts before release. This detects incompatible changes without running every service together.

It does not replace provider functional tests, schema compatibility, security tests, or a small set of end-to-end journeys. Govern stale consumers and conflicting expectations so contracts do not freeze the provider permanently.

### 75. How do you test messaging workflows?

Use a real broker or faithful disposable environment for integration semantics. Verify serialization, headers, partition key, retries, redelivery, idempotency, offset behavior, dead-letter routing, and schema evolution.

Tests should force failure between the business effect and acknowledgement. Deterministic polling/assertion is preferable to arbitrary sleeps.

### 76. Rolling versus blue-green versus canary deployment?

- **Rolling:** gradually replaces instances; resource-efficient but versions coexist.
- **Blue-green:** prepares a complete parallel environment and switches traffic; fast rollback but higher resource and data-migration complexity.
- **Canary:** sends limited traffic to the new version and expands based on evidence; reduces blast radius but needs strong metrics and routing.

All require backward-compatible contracts and database changes during overlap.

### 77. What is expand-and-contract deployment?

Introduce a backward-compatible contract or schema first, deploy code that supports both forms, migrate traffic/data, verify, then remove the old form in a later release.

This supports rolling deployments where multiple versions coexist. A destructive rename in one release can break old instances, consumers, replays, and rollback.

### 78. How should a canary be evaluated?

Compare canary and baseline on error rate, latency distributions, saturation, business outcomes, and dependency behavior with enough representative traffic. Define automatic abort thresholds and observation duration before rollout.

A tiny canary may miss low-frequency failures; a biased tenant or region may mislead. Ensure schema and asynchronous side effects remain safe if the canary is rolled back.

### 79. How do you test resilience?

Inject controlled latency, errors, connection resets, dependency unavailability, broker redelivery, instance termination, and resource saturation in realistic environments. Verify user-visible behavior, bounded resource use, alerts, and recovery—not only that an exception occurs.

Begin with small blast radius and a hypothesis. Chaos testing without observability, abort conditions, or ownership is uncontrolled risk rather than engineering.

---

## 10. Production scenarios

### 80. One downstream service becomes slow. How can it take down callers?

Slow calls occupy caller threads, sockets, and connection-pool entries. Queues grow, request latency exceeds deadlines, retries multiply traffic, and upstream services exhaust their own resources, causing a cascading failure.

Contain it with deadlines, bounded pools and queues, concurrency limits, circuit breakers, controlled retries, load shedding, and valid degradation. Trace the critical path and monitor saturation before total failure.

### 81. Message-consumer lag is growing. What do you investigate?

Compare ingress and processing rates, partition distribution, consumer health, rebalance frequency, processing latency, downstream database/API saturation, errors/retries, and hot keys. Check whether added consumers can actually receive partitions.

Scale or optimize the bottleneck, not merely the consumer count. Pause intake or shed nonessential production if backlog age violates the business deadline. Preserve ordering and idempotency during catch-up.

### 82. Users see duplicate orders after a timeout. What went wrong?

The first request may have committed, but the response was lost; the client retried with no stable idempotency key, or deduplication used a racy check-then-insert.

Accept an idempotency key, atomically enforce uniqueness with the order creation, store the result, and return it on retry. Also align client retry policy and trace logical requests separately from physical attempts.

### 83. A deployment causes mixed-version failures. How do you respond?

Stop or roll back traffic expansion, identify the incompatible contract/schema, and protect data from further incompatible writes. Use traces and version-tagged metrics to isolate interactions.

The prevention is expand-and-contract, tolerant readers, compatibility tests, version telemetry, and rollback-safe migrations. “All services deploy together” is evidence of a distributed monolith.

### 84. A service is healthy but receives no traffic. What do you inspect?

Check readiness, discovery registration, service/endpoints, selector labels, routing rules, gateway/load-balancer health, DNS, network policy, TLS identity, ports, and zone topology. Compare from the caller's network context.

Liveness only proves the process has not been restarted. Follow traffic layer by layer rather than restarting healthy instances blindly.

### 85. How do you investigate an intermittent cross-service failure?

Start with correlation and timing: trace ID, affected tenant/request type, service versions, region/zone, retries, and dependency spans. Compare successful and failed traces and inspect saturation and rollout events around the same window.

Intermittency often comes from one bad instance, connection reuse, data-dependent behavior, race conditions, stale configuration, partial rollout, hot partition, or tail latency. Preserve evidence before restarting everything.

### 86. How do you prevent a noisy neighbor?

Attribute use by tenant or workload, apply per-tenant quotas and concurrency limits, isolate queues/pools where necessary, schedule fairly, and cap expensive requests. Shard or dedicate resources for exceptional tenants only when justified.

Monitor fairness and throttling outcomes. A global rate limit can still let one tenant consume the entire allowance.

### 87. How do you design multi-region services?

Begin with failure and consistency requirements: active-passive or active-active, RTO/RPO, data residency, write ownership, conflict resolution, routing, and dependency locality. Test complete regional evacuation and recovery.

Active-active writes add latency and conflict complexity. Keep strongly consistent invariants within the smallest feasible region or ownership scope, and expose degraded behavior when cross-region coordination is unavailable.

### 88. What distinguishes a senior microservices answer?

- Starts with why distribution is needed.
- Aligns service boundaries with ownership and invariants.
- Defines timeout, retry, idempotency, and overload behavior.
- Treats messaging as at-least-once unless a narrower guarantee is proven.
- Makes eventual consistency visible and repairable.
- Separates user identity, workload identity, and authorization.
- Designs telemetry, deployment compatibility, and rollback with the feature.
- Uses bounded resources and rehearses failure recovery.

---

## 11. Rapid revision

### Must-answer questions

Before an interview, answer these without notes:

1. When are microservices justified?
2. Modular monolith versus microservices?
3. What makes a good service boundary?
4. Why must a service own its data?
5. What is a distributed monolith?
6. Synchronous versus asynchronous communication?
7. REST versus gRPC?
8. How do you evolve an API safely?
9. How do you implement idempotency?
10. Timeout versus deadline?
11. When is retry safe?
12. Why use jitter?
13. Circuit breaker versus bulkhead?
14. Rate limiting versus load shedding?
15. What is a saga, and why is compensation not rollback?
16. How does the transactional outbox work?
17. How does an inbox deduplicate messages?
18. Lost cross-service invariant: how would you redesign it?
19. CQRS versus event sourcing?
20. At-most-once versus at-least-once versus exactly-once?
21. What ordering does a partitioned broker provide?
22. When should a consumer commit progress?
23. How do you handle poison messages?
24. How do event schemas evolve?
25. API gateway versus service mesh?
26. What does mTLS provide—and not provide?
27. Logs versus metrics versus traces?
28. SLI versus SLO versus error budget?
29. Liveness versus readiness?
30. How does graceful shutdown affect consumers?
31. How do contract tests help?
32. Rolling versus blue-green versus canary?
33. How do you stop a cascading failure?
34. How do you diagnose growing consumer lag?
35. How do you design for regional failure?

### Thirty-second summary

Microservices trade local simplicity for independent ownership, deployment, and scaling. Good boundaries keep business invariants and data under one owner; communication contracts explicitly define compatibility, deadlines, idempotency, and failure. Resilience depends on bounded resources, safe retries, isolation, and overload control. Distributed workflows use local transactions, durable messaging, compensation, deduplication, and reconciliation rather than pretending the network is one ACID transaction. Senior design includes security, telemetry, rollout compatibility, and recovery from the beginning.

## Official references

- [AWS Builders' Library](https://aws.amazon.com/builders-library/)
- [Apache Kafka documentation](https://kafka.apache.org/documentation/)
- [Kubernetes concepts](https://kubernetes.io/docs/concepts/)
- [OpenTelemetry concepts](https://opentelemetry.io/docs/concepts/)
- [gRPC documentation](https://grpc.io/docs/)
- [OAuth 2.0 security best current practice](https://www.rfc-editor.org/rfc/rfc9700.html)

