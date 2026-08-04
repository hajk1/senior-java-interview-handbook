# Handbook Roadmap

This roadmap tracks the handbook from its current interview-question chapters toward a complete senior Java interview preparation system. Priorities are ordered by relevance to typical backend and full-stack senior roles.

## Progress

| Area | Status | Current scope | Next improvement |
|---|---|---|---|
| Core Java | Complete | 75 questions | Add focused coding exercises and diagrams |
| Spring and Spring Boot | Complete | 50 questions | Add configuration and debugging exercises |
| Databases | Complete | 76 questions | Add SQL exercises and execution-plan examples |
| Microservices | Complete | 88 questions | Add worked resilience and messaging exercises |
| System Design | Complete | 96 questions | Add more worked case studies and diagrams |
| DevOps | Complete | 73 questions | Add worked runbook and IaC exercises |
| React | Complete | 58 questions | Add worked hook and performance-debugging exercises |
| Behavioral | Planned | Empty scaffold | Add leadership questions and STAR preparation |
| Revision materials | Planned | Empty scaffold | Populate checklist, quick revision, and cheat sheets |

## Phase 1 — Backend foundations

### Completed

- [x] Create the Core Java senior interview chapter.
- [x] Cover the Java object model, equality, generics, and collections.
- [x] Cover streams, concurrency, the Java Memory Model, JVM, and modern Java.
- [x] Create the Spring and Spring Boot senior interview chapter.
- [x] Cover dependency injection, proxies, transactions, MVC, JPA, Security, and testing.
- [x] Create the Database senior interview chapter.
- [x] Cover SQL, indexes, isolation, locking, query optimization, replication, and scaling.
- [x] Add a navigable repository landing page.

### Quality improvements

- [ ] Add short coding exercises for collections, streams, and concurrency.
- [ ] Add JVM memory, happens-before, and Spring proxy diagrams.
- [ ] Add SQL exercises with schemas, expected answers, and alternatives.
- [ ] Add representative execution plans and a plan-reading walkthrough.
- [ ] Review chapter overlap and replace duplication with cross-links where useful.
- [ ] Add a consistent difficulty marker: foundational, senior, or deep dive.

## Phase 2 — Microservices (complete)

Built `04-microservices/README.md` around the most frequent senior backend questions.

### Architecture

- [x] Monolith versus modular monolith versus microservices.
- [x] Service boundaries, bounded contexts, and data ownership.
- [x] Synchronous versus asynchronous communication.
- [x] API contracts, compatibility, and versioning.
- [x] Service discovery, gateways, and load balancing.

### Reliability

- [x] Timeouts, retries, exponential backoff, and jitter.
- [x] Circuit breakers, bulkheads, rate limiting, and load shedding.
- [x] Idempotency, duplicate delivery, and retry safety.
- [x] Graceful degradation and fallback semantics.
- [x] Health checks, readiness, liveness, and graceful shutdown.

### Data and messaging

- [x] Database per service and cross-service queries.
- [x] Saga orchestration versus choreography.
- [x] Transactional outbox, change-data capture, and inbox patterns.
- [x] Delivery guarantees, ordering, partitioning, and consumer groups.
- [x] Event schema evolution and replay.

### Operations and security

- [x] Logs, metrics, traces, correlation, and service-level objectives.
- [x] Authentication, authorization, service identity, and secret management.
- [x] Deployment compatibility and safe rollouts.
- [x] Incident scenarios: retry storms, cascading failure, and partial outage.

## Phase 3 — System design (complete)

Built `07-system-design/README.md` as both an interview method and a collection of worked designs.

### Interview framework

- [x] Clarify functional and non-functional requirements.
- [x] Estimate traffic, storage, bandwidth, and growth.
- [x] Define APIs, data model, and consistency requirements.
- [x] Draw the high-level architecture and critical request flows.
- [x] Identify bottlenecks, failure modes, security boundaries, and trade-offs.
- [x] Explain scaling evolution rather than jumping to maximum complexity.

### Core concepts

- [x] Caching, invalidation, eviction, and stampede prevention.
- [x] Partitioning, replication, quorum, and consistency.
- [x] Queues, streams, backpressure, and asynchronous workflows.
- [x] Unique ID generation, distributed locks, and leader election.
- [x] Rate limiting, API gateways, CDNs, and search systems.
- [x] Multi-region design, disaster recovery, RPO, and RTO.

### Worked case studies

- [x] URL shortener.
- [x] Notification service.
- [x] Payment or order-processing system.
- [x] Distributed job scheduler.
- [x] Real-time chat or activity feed.
- [x] File-storage and sharing service.
- [x] Metrics and logging platform.

## Phase 4 — Delivery and operations (complete)

Built `06-devops/README.md` for the operational knowledge expected from a senior engineer.

- [x] Container images, layers, build security, and runtime limits.
- [x] Kubernetes pods, deployments, services, ingress, and probes.
- [x] Requests, limits, autoscaling, disruption budgets, and scheduling.
- [x] Configuration, secrets, workload identity, and network policy.
- [x] CI/CD pipelines, artifact promotion, and supply-chain security.
- [x] Rolling, blue-green, and canary deployments.
- [x] Infrastructure as code and environment drift.
- [x] Cloud networking, DNS, load balancers, and managed data services.
- [x] Observability, alerting, incident response, and postmortems.
- [x] Cost, capacity, availability, and disaster-recovery trade-offs.

## Phase 5 — Full-stack and behavioral preparation

### React (complete)

Built `05-react/README.md` for full-stack senior Java roles.

- [x] Rendering, reconciliation, component identity, and keys.
- [x] State, props, hooks, closures, and effect lifecycles.
- [x] Context, reducers, server state, and state-management trade-offs.
- [x] Performance profiling, memoization, code splitting, and virtualization.
- [x] Forms, accessibility, security, and error handling.
- [x] Component, integration, and end-to-end testing.

### Behavioral

Build `08-behavioral/README.md` around evidence-based senior-level stories.

- [ ] Create a reusable STAR story bank.
- [ ] Leadership without authority.
- [ ] Technical disagreement and conflict resolution.
- [ ] Production incidents and ownership.
- [ ] Failed decisions and lessons learned.
- [ ] Mentoring, feedback, and team growth.
- [ ] Prioritization, ambiguity, and stakeholder management.
- [ ] Architecture decisions and measurable business impact.

## Phase 6 — Revision system

- [ ] Populate `QUICK-REVISION.md` with one-line prompts from every chapter.
- [ ] Populate `INTERVIEW-CHECKLIST.md` with preparation and interview-day checks.
- [ ] Define a useful format in `INTERVIEW-LOG.md` for recording real questions and gaps.
- [ ] Add one-page cheat sheets for Java, Spring, SQL, microservices, and system design.
- [ ] Add spaced-repetition prompts or flashcard-ready exports.
- [ ] Add a two-week and four-week study plan.
- [ ] Cross-link related questions across chapters.

## Chapter definition of done

A chapter is considered complete when it:

- Covers the highest-frequency questions for its topic.
- Leads with concise answers suitable for an interview.
- Explains mechanisms, trade-offs, and production failure modes.
- Separates general principles from version- or vendor-specific behavior.
- Includes realistic examples where they materially improve understanding.
- Ends with a rapid-revision checklist and thirty-second summary.
- Uses current official sources for version-sensitive claims.
- Has working navigation and passes Markdown/whitespace validation.

## Future ideas

These are valuable but intentionally lower priority than completing the main chapters:

- [ ] Runnable Java coding examples with automated tests.
- [ ] Interactive diagrams for JVM, transactions, and distributed workflows.
- [ ] Mock interview question sets grouped into 30-, 60-, and 90-minute sessions.
- [ ] Company-style interview tracks: backend, platform, full-stack, and lead engineer.
- [ ] Community contribution templates and automated Markdown link checking.
- [ ] Release tags for stable handbook milestones.
