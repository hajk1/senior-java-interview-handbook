# Spring & Spring Boot — Senior Interview Guide

This chapter focuses on the Spring questions that appear most often in senior Java interviews. The goal is not to memorize annotations; it is to explain how Spring behaves, recognize production pitfalls, and defend design choices.

> **How to answer:** start with the one-sentence answer, explain the mechanism, then give a production example or failure mode.

## Contents

1. [Core container and dependency injection](#1-core-container-and-dependency-injection)
2. [AOP and proxies](#2-aop-and-proxies)
3. [Transactions](#3-transactions)
4. [Spring Boot](#4-spring-boot)
5. [Web and REST](#5-web-and-rest)
6. [Data access and JPA](#6-data-access-and-jpa)
7. [Security](#7-security)
8. [Testing](#8-testing)
9. [Production and architecture scenarios](#9-production-and-architecture-scenarios)
10. [Rapid revision](#10-rapid-revision)

---

## 1. Core container and dependency injection

### 1. What are IoC and dependency injection?

**Short answer:** Inversion of Control means the framework controls object creation and lifecycle. Dependency injection is the technique Spring uses to supply an object's collaborators instead of letting the object construct or locate them.

```java
@Service
class OrderService {
    private final PaymentGateway paymentGateway;

    OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

Constructor injection is normally preferred because dependencies are explicit, required fields can be `final`, and the class is easy to instantiate in a unit test. Field injection hides dependencies and requires framework or reflection support in tests. Setter injection is useful for genuinely optional or reconfigurable dependencies.

**Senior follow-up:** DI does not automatically produce good design. A constructor with many parameters often exposes a class with too many responsibilities.

### 2. What is the difference between `BeanFactory` and `ApplicationContext`?

`BeanFactory` is the basic bean container. `ApplicationContext` builds on it with features normally needed by applications: event publication, internationalization, resource loading, environment abstraction, and automatic registration of bean post-processors.

In practice, application code uses an `ApplicationContext`. A common trap is saying that the difference is simply “lazy versus eager initialization.” Singleton pre-instantiation is an `ApplicationContext` default, not the fundamental distinction, and can be changed with `@Lazy`.

### 3. Describe the lifecycle of a Spring bean.

A useful interview-level sequence is:

1. Spring reads configuration and creates a bean definition.
2. It instantiates the bean.
3. It injects dependencies.
4. `*Aware` callbacks run, when implemented.
5. `BeanPostProcessor#postProcessBeforeInitialization` runs.
6. Initialization callbacks run: `@PostConstruct`, `InitializingBean`, then a configured init method.
7. `BeanPostProcessor#postProcessAfterInitialization` runs; this is where a proxy may be returned.
8. The bean is ready for use.
9. On context shutdown, destruction callbacks run: `@PreDestroy`, `DisposableBean`, then a configured destroy method.

The exact internals contain more steps. The important senior-level point is that callers may receive a post-processed proxy rather than the original object.

### 4. What are the common bean scopes?

| Scope | Meaning | Typical use |
|---|---|---|
| `singleton` | One bean instance per application context; the default | Stateless services |
| `prototype` | A new instance for each container lookup/injection request | Stateful short-lived objects |
| `request` | One instance per HTTP request | Request-specific state |
| `session` | One instance per HTTP session | Session-specific state |
| `application` | One instance per `ServletContext` | Web-application-wide state |

Spring singleton does **not** mean one instance per JVM. Singleton beans are not automatically thread-safe; shared mutable state must be avoided or synchronized.

Spring creates prototype beans but does not manage their complete destruction lifecycle. Injecting a prototype directly into a singleton resolves it only once. Use `ObjectProvider<T>`, a scoped proxy, or an explicit factory when a fresh instance is required repeatedly.

### 5. How does Spring resolve multiple beans of the same type?

Use `@Qualifier` to select by semantic identity and `@Primary` to define a default candidate. Injection by a collection such as `List<PaymentGateway>` intentionally supplies all matching beans, with `@Order` available for ordering.

```java
OrderService(@Qualifier("stripeGateway") PaymentGateway gateway) {
    this.gateway = gateway;
}
```

Prefer meaningful qualifiers over relying on field or parameter names. If no unique candidate can be selected, startup fails, which is safer than choosing arbitrarily.

### 6. What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?

They are stereotype annotations discovered by component scanning:

- `@Component` is generic.
- `@Service` communicates application or domain service intent.
- `@Repository` communicates persistence intent and participates in persistence-exception translation.
- `@Controller` is an MVC controller; `@RestController` combines `@Controller` and `@ResponseBody`.

The specialized stereotypes are valuable mainly for semantics and for behavior attached by the framework, not because each creates a different kind of object.

### 7. What is the difference between `@Bean` and component scanning?

Component scanning registers classes you own and can annotate. An `@Bean` method gives explicit construction control and is ideal for third-party types or configuration requiring parameters and conditional logic.

With a full proxied `@Configuration`, calls between `@Bean` methods are intercepted so singleton semantics are preserved. With `@Configuration(proxyBeanMethods = false)`—or “lite” configuration—direct Java calls are not intercepted. Prefer method-parameter injection and avoid calling one bean method from another when proxying is disabled.

### 8. How do circular dependencies happen, and how should they be fixed?

A circular dependency exists when A requires B and B requires A. Constructor cycles fail because neither object can be completed first. Some setter/field cycles may be resolvable through early references, but relying on that produces fragile design and may fail when proxies are involved.

The best fix is normally architectural: extract the shared responsibility, introduce an event, or reverse the dependency through an interface. `@Lazy` or `ObjectProvider` can break the creation cycle but should be treated as deliberate indirection, not the default cure.

### 9. How do Spring application events work?

Publish with `ApplicationEventPublisher` and consume with `@EventListener`. By default, listeners execute synchronously in the publisher's thread, so their latency and exceptions affect the publisher.

`@Async` can move work to an executor, but in-memory async events are not durable. For work that must survive crashes or coordinate with other services, use a durable broker or the transactional outbox pattern. `@TransactionalEventListener` can bind handling to a transaction phase such as `AFTER_COMMIT`.

---

## 2. AOP and proxies

### 10. How does Spring AOP work?

Spring normally creates a proxy around a bean. Calls entering through that proxy are matched against pointcuts and passed through advice before or after reaching the target. Transactions, caching, async execution, method security, and custom aspects commonly use this mechanism.

Spring AOP supports method-execution join points on Spring-managed beans. It is not the same as full AspectJ weaving, which can advise broader join points such as constructors or field access.

### 11. JDK dynamic proxy versus class-based proxy?

- A JDK dynamic proxy implements interfaces exposed by the target.
- A class-based proxy subclasses the target class.

Class-based proxies cannot override `final` methods, and cannot proxy a `final` class. Private methods cannot be overridden either. Code should not depend on the proxy's concrete implementation; program to the service contract where practical.

### 12. Why does self-invocation break `@Transactional`, `@Async`, or `@Cacheable`?

```java
@Service
class BillingService {
    void checkout() {
        charge(); // direct call on this; it does not enter through the proxy
    }

    @Transactional
    public void charge() { }
}
```

The external caller invokes the proxy, but one method calling another on `this` bypasses it, so proxy advice is not applied. Move the advised operation to another bean, restructure the transaction boundary, or use AspectJ weaving when justified. Self-injecting the proxy or looking it up from the context couples business code to infrastructure and is usually a last resort.

---

## 3. Transactions

### 13. How does `@Transactional` work?

An interceptor around a proxied method obtains or joins a transaction through a `PlatformTransactionManager`, invokes the target, then commits or rolls back. This implies three recurring interview traps:

- The object must normally be Spring-managed.
- The call must normally cross the proxy boundary.
- The selected transaction manager must match the resource being used.

Place transaction boundaries around a complete business use case, usually in the service layer, rather than around every repository call.

### 14. When does Spring roll back a transaction?

By default, Spring rolls back for unchecked `RuntimeException` and `Error`, but not for checked exceptions. This can be customized:

```java
@Transactional(rollbackFor = PaymentException.class)
public void placeOrder() throws PaymentException { }
```

Catching an exception inside the transactional method and not rethrowing it can allow a commit. If an inner operation marks the transaction rollback-only and the outer code tries to commit, the caller may receive `UnexpectedRollbackException`.

### 15. Explain transaction propagation.

| Propagation | Behavior |
|---|---|
| `REQUIRED` | Join the current transaction or create one; default |
| `REQUIRES_NEW` | Suspend the current transaction and create an independent one |
| `NESTED` | Use a savepoint inside the existing physical transaction when supported |
| `SUPPORTS` | Join if one exists; otherwise run without one |
| `MANDATORY` | Require an existing transaction or fail |
| `NOT_SUPPORTED` | Suspend an existing transaction and run without one |
| `NEVER` | Fail if a transaction exists |

`REQUIRES_NEW` consumes a separate database connection while the outer transaction may retain its connection. Under concurrency, a small pool can therefore be exhausted. `NESTED` is not equivalent to `REQUIRES_NEW`; its availability and behavior depend on the transaction manager and savepoint support.

### 16. Explain isolation levels and anomalies.

| Isolation | Prevents | May still allow |
|---|---|---|
| Read uncommitted | — | Dirty, non-repeatable, phantom reads |
| Read committed | Dirty reads | Non-repeatable and phantom reads |
| Repeatable read | Dirty and non-repeatable reads | Phantom reads in the SQL model |
| Serializable | All three anomalies | Lower concurrency / serialization failures |

Actual behavior is database-specific. Isolation is not a substitute for understanding the invariant. For conflicting updates, consider optimistic locking with a version column, pessimistic locking, atomic SQL, or a constraint—not merely a stronger isolation level.

### 17. What does `readOnly = true` do?

It expresses intent and may allow the transaction manager, ORM, or database driver to optimize behavior. It is generally a hint, not a universal enforcement mechanism. It does not guarantee that writes are impossible on every database and configuration.

### 18. Can one Spring transaction atomically update a database and publish to Kafka?

Not safely in the general case without distributed transaction coordination, and even then complexity is high. A local database commit and a broker publish have a dual-write failure window.

A common solution is the transactional outbox: save the business change and an outbox record in the same database transaction; a separate publisher reliably sends the record; consumers are idempotent. This gives eventual consistency rather than pretending the network call is part of one local ACID transaction.

---

## 4. Spring Boot

### 19. What does `@SpringBootApplication` include?

It combines:

- `@SpringBootConfiguration` (a specialized `@Configuration`)
- `@EnableAutoConfiguration`
- `@ComponentScan`

Place the application class in a root package so default component scanning and auto-configuration package discovery cover the intended classes. Avoid the default package.

### 20. How does auto-configuration work?

Spring Boot loads auto-configuration candidates and activates them based on conditions such as classes, properties, resources, the web application type, or missing beans. Typical conditions include `@ConditionalOnClass` and `@ConditionalOnMissingBean`.

Auto-configuration is designed to back off when the application provides its own bean. It is not runtime “magic”; it is conditional configuration evaluated while the context starts. Use the condition evaluation report or start with `--debug` to understand why a configuration matched or did not match.

### 21. What is a Spring Boot starter?

A starter is a dependency descriptor that brings in a coherent set of libraries for a capability, such as web or data access. Auto-configuration is the code that conditionally configures those libraries. The two concepts work together but are not the same.

A company starter can standardize dependencies and include custom auto-configuration. Modern auto-configuration classes are listed in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

### 22. How does externalized configuration precedence work?

Spring Boot can read properties from configuration files, environment variables, system properties, command-line arguments, test sources, and other locations. Later/higher-priority sources override earlier ones; command-line arguments override ordinary file-based configuration.

Use validated, type-safe `@ConfigurationProperties` for a related group of settings. Use `@Value` sparingly for isolated values. Never store production secrets in source control; inject references or values from the deployment platform or a secrets manager.

```java
@ConfigurationProperties("payment")
@Validated
public record PaymentProperties(
        @NotBlank String baseUrl,
        @Positive Duration timeout) { }
```

### 23. What are profiles, and what is their common misuse?

Profiles conditionally activate beans or configuration for an environment or mode. They are useful for a small number of coherent variants.

Do not encode an uncontrolled matrix of environment and feature combinations into profiles. Prefer ordinary configuration for values and a dedicated feature-flag system for runtime rollout. Production code should not be selected merely because a profile name was accidentally absent.

### 24. What is Spring Boot Actuator?

Actuator exposes operational information and management capabilities such as health, metrics, loggers, environment, mappings, and thread dumps, depending on configuration. Micrometer provides vendor-neutral metrics instrumentation and observation integration.

Only expose endpoints that operators need. Put sensitive management endpoints behind authentication and authorization, consider a separate management port or network, and sanitize sensitive values. Liveness should answer whether the process must restart; readiness should answer whether it should receive traffic.

### 25. What happens during Spring Boot startup?

At a high level, `SpringApplication` determines the application type, prepares the environment, creates and refreshes the appropriate application context, registers and instantiates beans, starts the embedded server when relevant, and then runs `ApplicationRunner` and `CommandLineRunner` beans.

Slow startup is commonly investigated through startup metrics, condition reports, bean initialization, classpath size, database migrations, remote calls during initialization, and excessive component scanning. Avoid remote network calls in constructors or `@PostConstruct` unless startup truly depends on them.

---

## 5. Web and REST

### 26. How does Spring MVC process a request?

1. The servlet container passes the request through filters.
2. `DispatcherServlet` receives it.
3. A `HandlerMapping` finds the controller method.
4. A `HandlerAdapter` resolves arguments and invokes it.
5. The return value is processed—often by an `HttpMessageConverter` into JSON.
6. Exceptions are resolved by the exception-resolution chain.

Interceptors sit inside MVC around handler execution. Filters are servlet-level and can wrap requests and responses; they are appropriate for concerns that must occur before MVC, including security filter chains.

### 27. `@Controller` versus `@RestController`?

`@Controller` commonly returns a logical view name. `@ResponseBody` tells Spring to serialize a method's return value into the response. `@RestController` applies `@ResponseBody` to every handler method in the class.

Response serialization uses content negotiation and configured message converters. Returning an object does not itself imply JSON if no compatible converter or media type exists.

### 28. How should validation and exception handling be implemented?

Use Jakarta Bean Validation annotations on boundary DTOs and trigger validation with `@Valid` or `@Validated`. Use `@RestControllerAdvice` and targeted `@ExceptionHandler` methods to map domain and validation failures into a consistent error contract, preferably Problem Details (`ProblemDetail`) where appropriate.

Do not expose stack traces, SQL details, or internal exception messages to clients. Log unexpected failures with a correlation identifier, but avoid logging the same exception at every layer.

### 29. Filter versus interceptor versus aspect?

| Mechanism | Boundary | Good use cases |
|---|---|---|
| Servlet filter | Raw HTTP request/response, outside MVC | Security, CORS, request wrapping, correlation IDs |
| MVC interceptor | Around controller handler execution | Controller-aware logging, locale, lightweight request policy |
| AOP aspect | Spring bean method execution | Transactions, service metrics, cross-cutting method policy |

Choose the layer that actually owns the concern. For example, authentication normally belongs in Spring Security's filter chain, not a controller interceptor.

### 30. Spring MVC versus WebFlux?

Spring MVC is servlet-based and commonly uses one request thread during blocking work. WebFlux is reactive and non-blocking, supports backpressure through Reactive Streams, and can handle many concurrent I/O-bound operations with fewer threads.

WebFlux is beneficial only when the end-to-end path is non-blocking. Calling JDBC or another blocking client on an event-loop thread can destroy throughput. Reactive programming adds debugging and cognitive cost; it is not automatically faster for CPU-bound work or ordinary CRUD services.

### 31. How do you design an idempotent REST endpoint?

HTTP semantics make `GET`, `PUT`, and `DELETE` idempotent by definition, though implementations must honor that contract. For a retryable operation such as payment creation via `POST`, accept an idempotency key, store it with a unique constraint and the outcome, and return the same result for the same request.

Do not implement “check then insert” without a uniqueness constraint; concurrent requests can both pass the check. Define key scope, request-payload mismatch behavior, retention, and treatment of in-progress requests.

---

## 6. Data access and JPA

### 32. What does Spring Data JPA provide?

It generates repository implementations, derives queries from method names, supports declared queries, pagination, specifications, auditing, projections, and integration with Spring transactions. It reduces persistence boilerplate; it does not remove the need to understand JPA, SQL, indexes, fetching, and transaction boundaries.

Repository interfaces should represent useful aggregate access patterns. A generic repository method for every possible operation can leak persistence concerns into the domain and encourage inefficient queries.

### 33. What is the N+1 query problem, and how do you fix it?

One query loads N parent rows; accessing a lazy association then runs one additional query per parent. Detect it through SQL logs, tracing, query counts, and production metrics—not merely by inspecting annotations.

Fix it per use case with a fetch join, `@EntityGraph`, DTO projection, batch fetching, or a purpose-built query. Making every association eager often replaces N+1 with over-fetching, large joins, duplicate rows, or pagination problems.

### 34. Explain lazy loading and `LazyInitializationException`.

Lazy associations may be represented by proxies or persistent collections and require an open persistence context to initialize. Accessing them after the context is closed causes `LazyInitializationException`.

The robust fix is to fetch exactly what the use case needs inside the transaction and map to a DTO. Open Session in View keeps the persistence context open through web rendering, but it can hide accidental queries in controllers or serializers and blur transaction boundaries.

### 35. Optimistic versus pessimistic locking?

Optimistic locking uses `@Version` and detects a conflicting update at flush/commit; the loser retries or reports a conflict. It suits low-to-moderate contention and avoids holding database locks while business logic runs.

Pessimistic locking asks the database to lock rows, blocking or failing competing operations. It may fit high contention or cases where retry is expensive, but increases deadlock, latency, and timeout risk. Keep locked transactions short and use a deterministic lock order.

### 36. What are common JPA transaction pitfalls?

- Assuming `save()` always executes an immediate SQL statement; writes may wait until flush.
- Performing remote calls while a database transaction and connection remain open.
- Serializing managed entities directly, triggering lazy queries or recursion.
- Using bulk update/delete queries without clearing affected persistence-context state.
- Paginating over a collection fetch join and expecting correct SQL-level pagination.
- Treating `equals`/`hashCode` for generated-ID entities as trivial.
- Assuming `@Transactional` on a repository makes an entire multi-step service operation atomic.

---

## 7. Security

### 37. Explain Spring Security's servlet architecture.

The servlet container delegates to Spring Security's `FilterChainProxy`, which selects a `SecurityFilterChain`. Ordered security filters perform tasks such as exploit protection, authentication, security-context handling, and authorization.

An `AuthenticationManager`, commonly `ProviderManager`, delegates to one or more `AuthenticationProvider`s. A successful authentication is represented by an `Authentication` stored in the `SecurityContext`. Authorization then evaluates that identity and its authorities against request or method rules.

### 38. Authentication versus authorization?

Authentication establishes who the caller is. Authorization decides whether that identity may perform an action. Returning `401 Unauthorized` normally means authentication is absent or invalid; `403 Forbidden` means the authenticated caller lacks permission.

Enforce coarse rules at the HTTP boundary and business-sensitive rules at the service method or domain boundary. Hiding a UI button is not authorization.

### 39. How would you secure a stateless JWT API?

- Validate signature, issuer, audience, time claims, and allowed algorithms.
- Map claims to minimal authorities explicitly.
- Do not trust a token merely because it can be decoded.
- Keep access tokens short-lived; design refresh-token rotation and revocation according to risk.
- Avoid putting secrets or mutable authorization state in tokens.
- Configure the API as stateless and return consistent `401`/`403` responses.

JWT does not automatically make a system secure or truly session-free—refresh tokens, revocation, key rotation, and identity-provider state still require design.

### 40. When should CSRF protection be enabled?

CSRF matters when a browser automatically attaches credentials, especially session cookies. A state-changing cookie-authenticated browser application normally needs CSRF protection.

An API that accepts a bearer token only from an `Authorization` header and never relies on automatically sent credentials is generally not vulnerable in the same way. Disabling CSRF simply because an endpoint returns JSON is incorrect. CORS is a separate browser policy and is not a replacement for CSRF protection.

### 41. What are common Spring Security mistakes?

- Broadly permitting endpoints because matchers are ordered incorrectly.
- Logging tokens or credentials.
- Using plain text or fast general-purpose hashes for passwords instead of an adaptive password encoder.
- Confusing roles and authorities or depending on accidental prefix behavior.
- Trusting client-supplied identity fields instead of the authenticated principal.
- Omitting method-level authorization for sensitive operations reachable through multiple entry points.
- Exposing Actuator endpoints or error details publicly.

---

## 8. Testing

### 42. Unit test versus slice test versus `@SpringBootTest`?

| Test | Starts Spring? | Purpose |
|---|---:|---|
| Plain unit test | No | Business logic in isolation; fastest |
| `@WebMvcTest` | MVC slice | Controllers, validation, serialization, security behavior |
| `@DataJpaTest` | JPA slice | Mappings, repositories, and real query behavior |
| `@SpringBootTest` | Full context | Application wiring and end-to-end integration |

Use the smallest test that proves the behavior. Full-context tests are valuable but should not replace fast unit and focused integration tests.

### 43. MockMvc versus WebTestClient versus a real server?

`MockMvc` exercises Spring MVC without starting a real server. `WebTestClient` is native for WebFlux and can also test other server arrangements. A full test with `webEnvironment = RANDOM_PORT` crosses a real HTTP boundary and is useful when server configuration, filters, serialization, or networking behavior matters.

Do not mock the component whose integration you intend to verify. For persistence and infrastructure behavior, a real supported database or service in a disposable container is often more truthful than an in-memory substitute.

### 44. What commonly makes Spring tests slow or flaky?

Spring caches application contexts, but a different context configuration creates another cache entry. Excessive `@MockBean`/bean overrides, varying profiles, and unnecessary full-context annotations can produce many unique contexts.

Flakiness commonly comes from shared mutable data, fixed ports, uncontrolled clocks, asynchronous work, transaction assumptions, test order dependence, and external services. Prefer isolated data, random ports, injected `Clock`, deterministic async coordination, and disposable infrastructure.

---

## 9. Production and architecture scenarios

### 45. A Spring service is slow in production. How do you investigate?

Start with evidence and narrow the bottleneck:

1. Check latency percentiles, throughput, error rate, and saturation—not averages alone.
2. Follow a distributed trace to database, cache, messaging, and downstream calls.
3. Inspect thread pools, connection pools, HTTP clients, GC, CPU, and memory.
4. Compare application time with dependency time.
5. Look for N+1 queries, missing indexes, lock waits, retries, timeouts, and serialized work.
6. Reproduce with a representative load and validate the change against the same measurements.

Adding threads or caching before identifying the constrained resource can amplify the failure.

### 46. How do you make outbound HTTP calls resilient?

Set connection, response, and overall deadlines. Bound concurrency and connection pools. Retry only transient failures and only when the operation is safe or idempotent; use exponential backoff with jitter and a retry budget. Use a circuit breaker to stop repeated calls to an unhealthy dependency, and bulkheads to contain resource exhaustion.

Fallbacks must be semantically valid. Returning stale or fabricated data can be worse than failing clearly. Propagate correlation context, collect per-dependency metrics, and align timeouts with the caller's remaining deadline.

### 47. `@Async`: what should a senior engineer know?

`@Async` submits proxy-intercepted method calls to a `TaskExecutor`. Self-invocation does not work. The original thread's transaction does not automatically continue in the worker thread, and context such as security or logging correlation may require explicit propagation.

Configure a bounded executor, queue capacity, rejection policy, and metrics; do not rely blindly on defaults. Exceptions from `void` methods cannot be returned to the caller. Use `CompletableFuture` or another explicit result when failures matter. For durable business work, use a message broker rather than an in-memory executor.

### 48. How should caching be used with Spring?

`@Cacheable` reads through the cache, `@CachePut` updates it while executing the method, and `@CacheEvict` removes entries. Proxy limitations apply, including self-invocation.

The difficult part is policy: key design, TTL, invalidation, consistency, serialization compatibility, cache stampede protection, negative caching, and observability. Cache only data whose staleness contract is understood. Avoid caching mutable entities or user-specific responses under incomplete keys.

### 49. How do you shut down a Spring service gracefully?

Stop accepting new traffic, fail readiness, allow in-flight work to finish within a deadline, stop consumers carefully, flush or complete safe work, and close resources. Kubernetes termination grace periods, load-balancer propagation, application shutdown timeout, and message-consumer semantics must be aligned.

Shutdown hooks and `@PreDestroy` should be bounded and should not start new work. Exactly-once claims during shutdown deserve scrutiny; design message processing to tolerate redelivery.

### 50. What signals distinguish a senior Spring answer?

A strong answer consistently connects framework behavior to system behavior:

- Explains the proxy boundary, not just the annotation.
- Places transaction boundaries around business invariants.
- Discusses thread safety, connection pools, and failure modes.
- Separates local ACID guarantees from distributed consistency.
- Chooses blocking or reactive programming based on the complete call path.
- Uses metrics and traces to diagnose before tuning.
- Treats security, retries, caching, and async work as policies with trade-offs.
- Tests with realistic infrastructure where implementation differences matter.

---

## 10. Rapid revision

### Must-answer questions

Before an interview, be able to answer these without notes:

1. Why is constructor injection preferred?
2. What is the lifecycle of a Spring bean?
3. Are singleton beans thread-safe?
4. How do JDK and class-based proxies differ?
5. Why does self-invocation bypass `@Transactional`?
6. Which exceptions cause rollback by default?
7. `REQUIRED` versus `REQUIRES_NEW` versus `NESTED`?
8. How does Spring Boot auto-configuration back off?
9. Starter versus auto-configuration?
10. `@ConfigurationProperties` versus `@Value`?
11. Filter versus interceptor versus aspect?
12. MVC versus WebFlux?
13. How do you prevent duplicate processing on retries?
14. What causes N+1 queries?
15. Optimistic versus pessimistic locking?
16. Authentication versus authorization?
17. When is CSRF relevant?
18. Slice test versus full-context test?
19. Why can `@Async` exhaust a system?
20. How do you solve a database-and-broker dual write?

### Thirty-second summary

Spring is a container that creates and connects application objects. Much of its cross-cutting behavior is implemented with proxies, so calls must cross a proxy boundary. Transactions protect local resource operations, not arbitrary distributed work. Spring Boot adds opinionated dependency management and conditional auto-configuration that backs off when the application supplies its own configuration. Senior-level usage means understanding the boundaries: threads, transactions, persistence contexts, HTTP calls, security filters, caches, and failure recovery.

## Official references

- [Spring Framework reference](https://docs.spring.io/spring-framework/reference/)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)
- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Security reference](https://docs.spring.io/spring-security/reference/)

