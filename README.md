# Senior Java Interview Handbook

A practical, interview-focused handbook for senior Java engineers. It concentrates on the questions that appear most often, explains what happens beneath framework annotations, and connects technical answers to production trade-offs.

The handbook currently contains **198 questions and model answers** across Core Java, Spring, and databases.

## Chapters

| Chapter | Coverage | Questions | Status |
|---|---|---:|---|
| [Core Java](01-java-core/README.md) | Object model, collections, generics, streams, concurrency, JVM, and modern Java | 72 | Available |
| [Spring and Spring Boot](02-spring/README.md) | DI, bean lifecycle, AOP, transactions, Boot, MVC, JPA, Security, and testing | 50 | Available |
| [Databases](03-database/README.md) | SQL, modeling, indexes, isolation, locking, optimization, replication, and scaling | 76 | Available |
| Microservices | Service boundaries, resilience, messaging, consistency, and observability | — | Planned |
| React | Components, hooks, state, rendering, performance, and testing | — | Planned |
| DevOps | Containers, Kubernetes, CI/CD, cloud infrastructure, and operations | — | Planned |
| System Design | Requirements, capacity, APIs, data, architecture, and trade-offs | — | Planned |
| Behavioral | Leadership, ownership, conflict, delivery, and STAR stories | — | Planned |

## What makes this handbook different?

The answers are designed for senior-level interviews rather than certification-style memorization. Each chapter emphasizes:

- A concise answer that can be delivered first.
- The mechanism behind the behavior.
- Common traps and misleading shortcuts.
- Production consequences and failure modes.
- Design alternatives and their trade-offs.
- A rapid-revision checklist before the interview.

## How to use it

### First pass: establish coverage

Read each question and try to answer it aloud before revealing the explanation. Mark topics where your answer was incomplete, imprecise, or based only on syntax.

### Second pass: practice senior follow-ups

For every topic, be ready to explain:

1. How does it work internally?
2. When would you use it?
3. What can go wrong in production?
4. What alternative would you consider?
5. How would you diagnose or measure it?

### Final pass: rapid revision

Use the checklist at the end of each chapter. A strong response usually follows this structure:

> **Rule → mechanism → example → trade-off**

For example, do not stop at “`@Transactional` starts a transaction.” Explain the proxy boundary, propagation choice, rollback rules, and what happens when a call uses self-invocation.

## Suggested study order

1. **Core Java** — language contracts, collections, concurrency, and JVM fundamentals.
2. **Spring** — dependency injection, proxies, transactions, web applications, and production behavior.
3. **Databases** — SQL execution, indexes, isolation, locking, and scaling.
4. **Microservices and system design** — distributed trade-offs built on the earlier foundations.
5. **DevOps and behavioral preparation** — delivery, operations, and leadership evidence.

## Interview principles

- Prefer precise contracts over folklore.
- Start with the simplest correct answer, then add depth.
- Distinguish general models from implementation-specific behavior.
- Connect technical choices to latency, throughput, consistency, and failure recovery.
- Use measurements, execution plans, traces, dumps, and profiles as evidence.
- State assumptions before solving an ambiguous design problem.
- Discuss boundaries: transactions, threads, services, ownership, and trust.

## Repository structure

```text
senior-java-interview-handbook/
├── 01-java-core/
├── 02-spring/
├── 03-database/
├── 04-microservices/
├── 05-react/
├── 06-devops/
├── 07-system-design/
├── 08-behavioral/
├── cheat-sheets/
├── templates/
├── INTERVIEW-CHECKLIST.md
├── INTERVIEW-LOG.md
├── QUICK-REVISION.md
└── ROADMAP.md
```

## Contributing

Contributions are welcome. Keep additions interview-focused, technically precise, and concise enough to revise quickly. When adding a question:

1. Lead with the direct answer.
2. Explain the underlying mechanism.
3. Include a realistic example or failure mode.
4. Call out behavior that varies by version, database, JVM, or framework implementation.
5. Prefer official documentation for version-sensitive claims.

## Project status

This handbook is under active development. Core Java, Spring, and Database chapters are available; the remaining chapters and supporting revision materials are planned.
