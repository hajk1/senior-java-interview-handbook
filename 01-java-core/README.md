# Core Java — Senior Interview Guide

This chapter covers the Core Java questions most often used to assess senior engineers. Strong answers explain the language rule, its runtime consequence, and the design trade-off—not merely the syntax.

> **How to answer:** give the rule in one sentence, explain why it matters, and finish with an example, edge case, or production consequence.

## Contents

1. [Language and object model](#1-language-and-object-model)
2. [Equality, immutability, and object contracts](#2-equality-immutability-and-object-contracts)
3. [Generics and collections](#3-generics-and-collections)
4. [Exceptions and resource management](#4-exceptions-and-resource-management)
5. [Functional Java and streams](#5-functional-java-and-streams)
6. [Concurrency](#6-concurrency)
7. [JVM and memory](#7-jvm-and-memory)
8. [Modern Java](#8-modern-java)
9. [Design and production scenarios](#9-design-and-production-scenarios)
10. [Rapid revision](#10-rapid-revision)

---

## 1. Language and object model

### 1. Is Java pass-by-value or pass-by-reference?

Java is always pass-by-value. For an object, the copied value is a reference to that object.

```java
static void update(Person person) {
    person.setName("Sam");     // mutates the shared object
    person = new Person("Jo"); // changes only the local copy of the reference
}
```

The caller can observe mutation of the referenced object, but reassignment of the method parameter cannot replace the caller's variable. Saying “objects are passed by reference” predicts the second operation incorrectly.

### 2. What is the difference between `==` and `equals()`?

For primitives, `==` compares values. For references, `==` compares identity—whether both references point to the same object. `equals()` expresses logical equality when the class overrides it; otherwise `Object.equals()` is identity equality.

Use `Objects.equals(a, b)` for a null-safe logical comparison. Do not use `==` for strings or boxed values; interning and wrapper caches can make it appear to work for some values and fail for others.

### 3. Overloading versus overriding?

Overloading chooses among methods with the same name using compile-time argument types. Overriding supplies a subtype implementation selected at runtime through dynamic dispatch.

An overriding method cannot reduce visibility, may use a covariant return type, and may not declare broader checked exceptions. `static` methods are hidden, not overridden. `private` methods are not inherited and therefore cannot be overridden.

```java
class Parent { static void staticCall() {} void call() {} }
class Child extends Parent { static void staticCall() {} @Override void call() {} }
```

### 4. Interface versus abstract class?

An interface defines a capability or contract and allows a class to implement multiple interfaces. It can have abstract, default, static, and private methods, but its instance fields are implicitly `public static final`.

An abstract class can hold instance state, constructors, protected members, and partial implementation. Use an interface for substitutable roles across a type hierarchy; use an abstract class when closely related types genuinely share state or invariant-preserving implementation. Prefer composition when inheritance exists mainly for code reuse.

### 5. What is the diamond problem with default methods?

If two unrelated interfaces provide the same default method, the implementing class must resolve the conflict explicitly. A concrete class method wins over an interface default, and a more specific interface wins over a parent interface.

```java
class Report implements PdfView, HtmlView {
    @Override
    public String render() {
        return PdfView.super.render();
    }
}
```

Default methods support interface evolution, but adding one can create a source-level conflict for existing implementors.

### 6. Explain `final`, `finally`, and `finalize()`.

- `final` prevents reassignment of a variable, overriding of a method, or extension of a class.
- `finally` is a cleanup block associated with `try`; try-with-resources is preferred for closeable resources.
- `finalize()` was an unreliable cleanup mechanism and is deprecated for removal. It offered no prompt-execution guarantee and could harm performance and correctness.

A `final` reference does not make the referenced object immutable.

### 7. What are Java access modifiers?

| Modifier | Same class | Same package | Subclass in another package | Unrelated class elsewhere |
|---|---:|---:|---:|---:|
| `private` | Yes | No | No | No |
| package-private | Yes | Yes | No | No |
| `protected` | Yes | Yes | Yes, through inheritance rules | No |
| `public` | Yes | Yes | Yes | Yes |

The subtle case is `protected` across packages: a subclass can access the member through inheritance, subject to restrictions on the qualifying reference; it is not simply “package plus any object from a subclass.”

### 8. Composition versus inheritance?

Inheritance models an “is-a” relationship and enables subtype polymorphism, but couples the child to the parent's implementation and invariants. Composition models “has-a,” delegates behavior, and allows collaborators to vary independently.

Prefer composition unless clients should genuinely be able to substitute the child wherever the parent is expected. A classic warning sign is extending a concrete collection only to reuse code while changing its behavioral contract.

### 9. What does `static` mean?

A static member belongs to the class, not to an instance. Static methods have no `this`, cannot directly access instance state, and use compile-time method selection rather than runtime overriding.

Class initialization occurs when the JVM first actively uses the class, and is synchronized by the JVM. The initialization-on-demand holder idiom uses this guarantee for lazy, thread-safe initialization without explicit locking.

### 10. In what order are fields and constructors initialized?

For object construction: memory receives default values, the superclass constructor chain runs, then instance field initializers and instance initializer blocks execute in source order, followed by the remainder of the constructor body. Each class's initialization occurs after its superclass portion.

Calling overridable methods from a constructor is dangerous because dispatch can reach a subclass before its fields have been initialized.

---

## 2. Equality, immutability, and object contracts

### 11. What is the `equals()` contract?

For non-null references, equality must be reflexive, symmetric, transitive, consistent, and false against `null`. Whenever `equals()` is overridden, `hashCode()` must normally be overridden too.

Inheritance makes value equality difficult: adding a field in a subclass can break symmetry or transitivity. Value objects are often safer as final classes or records, with equality based on stable value components.

### 12. What is the `hashCode()` contract?

Objects equal according to `equals()` must have the same hash code. Unequal objects may collide. The hash code should remain consistent while fields used by equality remain unchanged.

Mutating a key after inserting it into a `HashMap` can place it in a bucket inconsistent with its new hash, making lookup or removal appear to lose the entry. Use immutable keys.

### 13. How do you design an immutable class?

- Prevent extension or otherwise control subclass behavior.
- Make state private and final.
- Validate and defensively copy mutable inputs.
- Do not expose mutable internals; return copies or immutable views.
- Ensure methods do not mutate observable state.
- Avoid leaking `this` during construction.

```java
public final class Schedule {
    private final List<Instant> times;

    public Schedule(List<Instant> times) {
        this.times = List.copyOf(times);
    }

    public List<Instant> times() {
        return times;
    }
}
```

Immutability simplifies sharing and concurrency, but an immutable container is not deeply immutable when its elements are mutable.

### 14. `String`, `StringBuilder`, and `StringBuffer`?

`String` is immutable and suitable for values, keys, and safe sharing. `StringBuilder` is mutable and normally best for local string construction. `StringBuffer` synchronizes its operations but is rarely needed; external coordination is still required for a multi-operation invariant.

The compiler can optimize simple concatenation. In a loop or dynamic construction path, use `StringBuilder` rather than repeatedly creating intermediate strings.

### 15. Why is `String` immutable?

Immutability allows safe sharing and interning, stable hash codes, predictable use as map keys, and safer handling of security-sensitive values such as class or resource names. It also makes `String` naturally thread-safe.

Immutability does not make `String` suitable for secrets: its contents cannot be cleared deterministically. A mutable character array may be preferable for short-lived sensitive data, although copies can still exist elsewhere.

### 16. Shallow copy versus deep copy?

A shallow copy duplicates the outer object but shares references to nested mutable objects. A deep copy recursively duplicates the relevant mutable object graph.

`Cloneable` and `Object.clone()` are often awkward: constructors are bypassed, copying is shallow by default, and invariants are easy to violate. Prefer a copy constructor, static factory, or explicit mapping that states which parts are shared and which are copied.

---

## 3. Generics and collections

### 17. Why do Java generics use type erasure?

Most generic type information is enforced by the compiler and erased to bounds or `Object` in bytecode, with casts inserted where necessary. This preserved broad binary compatibility when generics were introduced.

Consequences include:

- No `new T()` or `new T[]` directly.
- No `instanceof List<String>`.
- `List<String>` and `List<Integer>` have the same runtime class.
- Overloads cannot differ only by generic type arguments if erasure is identical.
- The compiler may generate bridge methods to preserve polymorphism.

Some generic information remains in class-file signatures and can be inspected through reflection when declared in a reifiable location.

### 18. What is PECS?

**Producer Extends, Consumer Super.** Use `? extends T` when reading values as `T`; use `? super T` when writing `T` values.

```java
static <T> void copy(List<? extends T> source, List<? super T> target) {
    target.addAll(source);
}
```

From `List<? extends Number>` you can safely read `Number`, but cannot add a specific number because the actual list might be `List<Integer>`. Into `List<? super Integer>` you can add `Integer`, but reads are only guaranteed as `Object`.

### 19. Why is `List<String>` not a subtype of `List<Object>`?

Generics are invariant. If the subtype relationship existed, code could add an `Integer` through the `List<Object>` view and corrupt a list promised to contain only strings. Wildcards express safe covariance or contravariance where needed.

Arrays, unlike generics, are covariant and reified. Therefore `Object[] values = new String[1]` compiles but storing an integer throws `ArrayStoreException` at runtime.

### 20. `ArrayList` versus `LinkedList`?

`ArrayList` provides constant-time indexed access and amortized constant-time append, with contiguous references and good cache locality. Inserting or removing in the middle shifts elements.

`LinkedList` has linear indexed access and allocates a node per element. Insertion is constant-time only after the node position is already known. In real applications, `ArrayList` usually wins even for many workloads casually described as insertion-heavy. Use a purpose-built deque such as `ArrayDeque` for stack or queue behavior.

### 21. How does `HashMap` work?

`HashMap` spreads the key's hash to select a table bucket. It compares the hash and then `equals()` to find the key. Collisions share a bucket; sufficiently large collision chains may become balanced trees when capacity and threshold conditions are met.

Average lookup is constant-time with a good hash distribution, but it is not a concurrency guarantee or a strict worst-case promise. One null key and multiple null values are supported. Capacity grows when size exceeds `capacity × loadFactor`, requiring redistribution.

### 22. How does `ConcurrentHashMap` differ from `HashMap`?

`ConcurrentHashMap` supports thread-safe concurrent access without one global lock for all ordinary operations. Reads are generally non-blocking, while updates coordinate at finer granularity. It rejects null keys and values so `null` cannot ambiguously mean “absent” during concurrent operations.

Compound actions must use atomic methods such as `compute`, `merge`, `putIfAbsent`, or `replace`; `if (!map.containsKey(k)) map.put(k, v)` is still a race. Mapping functions should be short and should avoid recursive updates or blocking work.

### 23. `HashMap`, `LinkedHashMap`, and `TreeMap`?

| Map | Ordering | Typical operation cost | Best use |
|---|---|---:|---|
| `HashMap` | None guaranteed | Average O(1) | General lookup |
| `LinkedHashMap` | Insertion or access order | Average O(1) | Predictable iteration, simple LRU policy |
| `TreeMap` | Sorted by natural order or comparator | O(log n) | Range and ordered operations |

A comparator used by a sorted map should normally be consistent with `equals`; otherwise the map's idea of duplicate keys may surprise callers.

### 24. `Comparable` versus `Comparator`?

`Comparable<T>` defines the type's natural ordering through `compareTo`. `Comparator<T>` defines an external, reusable ordering and supports multiple orderings for the same type.

Never subtract integers to implement comparison because overflow can reverse the result. Use `Integer.compare`, `Comparator.comparing`, and then-comparators. A comparator must be antisymmetric and transitive, and should return zero precisely when values are equivalent for the sorted collection's purpose.

### 25. How do fail-fast iterators work?

Many ordinary collections track structural modification and an iterator checks an expected modification count. Unexpected modification can cause `ConcurrentModificationException`.

This is a best-effort bug detector, not a thread-safety mechanism or guaranteed behavior. Use the iterator's own `remove`, a bulk operation such as `removeIf`, or an appropriate concurrent collection. Weakly consistent concurrent iterators may reflect some updates without throwing.

### 26. Immutable collection versus unmodifiable view?

`Collections.unmodifiableList(source)` returns a read-only view; changes through another reference to `source` remain visible. `List.copyOf(source)` creates an unmodifiable snapshot of the element references and rejects null elements.

Neither approach deep-copies mutable elements. Also distinguish a fixed-size list such as `Arrays.asList`, which permits `set` but not structural addition or removal.

### 27. When should `CopyOnWriteArrayList` be used?

It is useful when reads and iteration vastly outnumber writes and snapshots are desirable, such as a small listener registry. Every mutation copies the backing array, making frequent writes or large lists expensive. Its iterator observes a stable snapshot and does not reflect later changes.

### 28. What should you know about `Optional`?

`Optional` models a possibly absent return value. It is not intended as a universal replacement for `null`, and is usually inappropriate for entity fields, DTO fields, method parameters, or collections of optionals.

Use `orElseGet` when the fallback is expensive because `orElse` evaluates its argument eagerly. Avoid `isPresent()` followed by `get()` when `map`, `flatMap`, `filter`, or `orElseThrow` expresses the flow directly.

---

## 4. Exceptions and resource management

### 29. Checked versus unchecked exceptions?

Checked exceptions must be caught or declared. Unchecked exceptions extend `RuntimeException`; compiler handling is not required.

Use checked exceptions when callers can reasonably and immediately recover as part of the contract. Use unchecked exceptions for programming errors, violated invariants, or failures that most layers cannot meaningfully handle. The key is a stable abstraction: do not leak low-level SQL or transport exceptions through a domain API.

### 30. How should exceptions be handled and translated?

Catch an exception only when you can recover, add meaningful context, translate it at an abstraction boundary, or perform cleanup that cannot use structured resource management. Preserve the cause when translating:

```java
throw new OrderLoadException("Cannot load order " + orderId, cause);
```

Do not log and rethrow the same failure at every layer; that creates duplicate noise. Do not catch `Exception` merely to continue with corrupted or unknown state.

### 31. How does try-with-resources work?

Resources implementing `AutoCloseable` are closed in reverse declaration order, even when the body fails. If both the body and `close()` throw, the body exception remains primary and close failures are attached as suppressed exceptions.

```java
try (var input = Files.newInputStream(path);
     var reader = new BufferedReader(new InputStreamReader(input))) {
    return reader.readLine();
}
```

Inspect `getSuppressed()` when cleanup failures matter. Avoid returning resources tied to an already-closed owner.

### 32. Does `finally` always execute?

Normally it runs when control leaves `try` or `catch`, including through `return` or an exception. It may not run if the process or JVM terminates abruptly, the machine fails, or execution never leaves the block.

A `return` or thrown exception inside `finally` can replace the original result or exception and should be avoided.

---

## 5. Functional Java and streams

### 33. What is a functional interface?

A functional interface has one abstract method and can be the target of a lambda or method reference. Default, static, private, and compatible `Object` methods do not count toward that single abstract method.

`@FunctionalInterface` is optional but lets the compiler protect the design. Common types include `Function`, `Predicate`, `Consumer`, `Supplier`, and their primitive specializations, which avoid boxing.

### 34. Lambda versus anonymous class?

A lambda does not introduce a new `this`; `this` refers to the enclosing instance. An anonymous class creates its own `this` and class scope. Lambda implementation details are deliberately unspecified and should not be treated as ordinary anonymous-class instances.

Captured local variables must be final or effectively final. The restriction avoids mutable local-variable capture semantics after the stack frame returns; mutable object state referenced by such a variable can still change.

### 35. How are streams evaluated?

A stream pipeline has a source, lazy intermediate operations, and a terminal operation. Traversal begins only when the terminal operation requests values. Stateless operations can be fused, and short-circuiting operations may avoid processing the entire source.

A stream is single-use and does not store data. Avoid side effects in pipeline functions; they make ordering, parallelism, testing, and reasoning difficult.

### 36. `map` versus `flatMap`?

`map` transforms each element into one result. `flatMap` transforms each element into a nested stream-like result and flattens one level.

```java
List<String> lines = files.stream()
        .flatMap(file -> readLines(file).stream())
        .toList();
```

The same concept appears with `Optional.flatMap` and `CompletableFuture.thenCompose`: use it when the mapping function already returns the container type and nesting is unwanted.

### 37. `reduce` versus `collect`?

`reduce` combines values into an immutable-style result, such as a sum. `collect` performs mutable reduction into a container, such as a list, map, or grouped result.

For parallel correctness, the identity must be neutral and the accumulator/combiner must obey the required associativity and compatibility rules. Do not mutate and return the same collection from `reduce`; use `collect`.

### 38. What are common stream mistakes?

- Reusing a consumed stream.
- Mutating external state in `map`, `filter`, or `forEach`.
- Using `peek` for required business behavior.
- Ignoring duplicate keys in `Collectors.toMap`.
- Assuming encounter order in an unordered or parallel pipeline.
- Boxing large primitive workloads instead of using primitive streams.
- Using streams for complex branching where a loop is clearer.
- Running blocking work in a parallel stream.

### 39. When should parallel streams be used?

Only after measurement, for sufficiently large, splittable, CPU-bound work with independent operations and an associative reduction. Parallel streams normally use the common fork-join pool, a process-wide resource also used by other facilities.

They are a poor default for blocking I/O, small collections, ordered operations, shared mutation, request-per-thread server code, or work requiring a dedicated concurrency limit. Parallelism can make a single request faster while reducing total service throughput.

---

## 6. Concurrency

### 40. What are race conditions, visibility, and atomicity?

- A race condition means correctness depends on uncontrolled timing.
- Visibility means one thread can observe another thread's writes.
- Atomicity means an operation appears indivisible relative to other threads.

`count++` is a read-modify-write sequence and is not atomic. Thread safety requires protecting the entire invariant, not merely making individual fields visible.

### 41. What does `synchronized` guarantee?

It provides mutual exclusion for code locking the same monitor and establishes happens-before relationships: an unlock happens-before a later lock of that monitor. This provides both atomicity for the critical section and visibility across it.

Instance synchronized methods lock `this`; static synchronized methods lock the `Class` object. Keep critical sections small, avoid blocking network calls while holding a lock, and use a consistent lock order to prevent deadlock.

### 42. What does `volatile` guarantee?

A write to a volatile variable happens-before a later read of that variable, providing visibility and ordering constraints. Individual volatile reads and writes are atomic, including `long` and `double`, but compound operations such as `count++` are not.

`volatile` works well for an independent state flag or safely publishing an immutable object. It cannot protect an invariant spanning multiple variables. Use locking or an appropriate atomic abstraction for compound state transitions.

### 43. Explain the Java Memory Model and happens-before.

The Java Memory Model defines which writes one thread is guaranteed to observe and which reorderings are legal. In the absence of a happens-before relationship, a data race can produce stale or surprising observations even if the code appears ordered in each thread.

Important happens-before edges include monitor unlock-to-lock, volatile write-to-read, actions before `Thread.start()` to the new thread, and all actions in a thread to another thread successfully returning from `join()`.

### 44. `synchronized` versus `ReentrantLock`?

Both provide mutual exclusion and visibility. `ReentrantLock` additionally offers interruptible lock acquisition, timed `tryLock`, optional fairness, multiple `Condition`s, and explicit lock management.

```java
lock.lock();
try {
    updateState();
} finally {
    lock.unlock();
}
```

Prefer `synchronized` when its structured simplicity is enough. Use explicit locks for a specific capability, not because they are assumed to be universally faster.

### 45. Atomic variables versus `LongAdder`?

`AtomicInteger` and `AtomicLong` use atomic compare-and-set style operations and are appropriate when updates and exact reads must form one linearizable value. Compound multi-variable invariants still need broader coordination.

`LongAdder` spreads updates across cells to reduce contention and is excellent for high-update statistics. `sum()` is not an atomic snapshot relative to concurrent updates, so it is inappropriate for identifiers or exact coordination.

### 46. How should a thread pool be sized and configured?

There is no universal number. CPU-bound work often starts near the available processor count. I/O-bound work can use more threads according to wait time, service limits, memory, and measured throughput.

Always consider:

- Maximum threads and queue capacity.
- Per-task memory and `ThreadLocal` state.
- Rejection/backpressure policy.
- Task deadlines, cancellation, and shutdown.
- Dependency connection-pool capacity.
- Metrics for active threads, queueing, rejection, and latency.

An unbounded queue can convert overload into high latency and memory exhaustion.

### 47. `Runnable`, `Callable`, `Future`, and `CompletableFuture`?

`Runnable` returns no result and cannot declare checked exceptions. `Callable<T>` returns a value and may throw. `Future<T>` represents a submitted result but offers limited composition. `CompletableFuture<T>` supports non-blocking completion stages and composition.

Use `thenCompose` for an asynchronous function that already returns a future; `thenApply` would create a nested future. Know which executor executes each stage. Non-`Async` continuations may run in the thread that completes the previous stage; `*Async` methods without an executor typically use the common pool.

### 48. How do interruption and cancellation work?

Interruption is a cooperative request, not forced thread termination. Blocking methods may throw `InterruptedException` and clear the status. Code that cannot handle it should usually restore the status and exit or propagate:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return;
}
```

`Future.cancel(true)` requests interruption if the task is running; it cannot guarantee that a task ignoring interruption will stop. Never swallow interruption silently.

### 49. What causes deadlock, and how do you prevent it?

Deadlock typically requires mutual exclusion, hold-and-wait, no forced preemption, and circular wait. Prevent it by imposing a global lock order, avoiding nested locks, keeping locked sections small, using timed acquisition where appropriate, and avoiding unknown external code while holding locks.

Diagnose with thread dumps: look for threads waiting on locks held in a cycle. Random delays may hide the issue but do not fix it.

### 50. `CountDownLatch`, `CyclicBarrier`, `Semaphore`, and `Phaser`?

| Utility | Purpose |
|---|---|
| `CountDownLatch` | One-shot wait until N events complete |
| `CyclicBarrier` | Reusable meeting point for a fixed number of parties |
| `Semaphore` | Limit concurrent access with permits |
| `Phaser` | Multi-phase coordination with dynamic registration |

Use higher-level concurrency utilities before hand-writing wait/notify protocols. A semaphore bounds concurrency; it does not itself guarantee fairness or protect a complex invariant.

### 51. How does `ConcurrentHashMap.computeIfAbsent` help, and what are its traps?

It atomically computes and installs a value when a key is absent, avoiding a check-then-act race. The mapping function should be short, side-effect-aware, and must not recursively update the same map in a way that violates its contract.

It can be useful for memoization, but unbounded keys create an unbounded cache. For production caching, consider eviction, expiry, failure behavior, and stampede handling.

### 52. Platform threads versus virtual threads?

Platform threads are typically mapped to operating-system threads and are relatively expensive in large numbers. Virtual threads are lightweight JVM-managed threads designed to make thread-per-task style scale for high-concurrency blocking I/O.

Virtual threads improve scale, not the speed of CPU work. They do not remove database connection limits, rate limits, memory pressure, race conditions, or the need for timeouts. Avoid pooling virtual threads merely to limit their number; use semaphores or resource pools to limit access to scarce dependencies. Monitor pinning and long-running CPU work where relevant.

---

## 7. JVM and memory

### 53. What are the main JVM runtime memory areas?

- **Heap:** objects and arrays, shared across threads and managed by GC.
- **Java stacks:** per-thread frames containing local variables, operand stacks, and call state.
- **Metaspace:** native memory used for class metadata.
- **PC register:** current instruction position per thread.
- **Native method stack:** supports native execution.
- **Code cache and direct/native memory:** important HotSpot/process consumers outside the Java heap.

An `OutOfMemoryError` may refer to heap, metaspace, direct buffer memory, native thread creation, or another native allocation—not only “too many objects on heap.”

### 54. Stack versus heap?

Method invocation frames are placed on a thread's stack; objects are conceptually allocated on the heap. References can exist in either place. The JIT may eliminate allocations or scalar-replace objects, so the source-level model is not a guarantee of physical placement.

Deep or infinite recursion can cause `StackOverflowError`. Retaining an ever-growing reachable object graph can exhaust the heap.

### 55. How does garbage collection determine that an object is collectible?

The JVM traces from GC roots—such as live thread stacks, static references, and JNI references. Objects not reachable from those roots are eligible for collection, including isolated cycles.

Eligibility does not imply immediate reclamation. A Java memory leak is usually an object that is no longer useful but remains strongly reachable through caches, listeners, static collections, queues, class loaders, or `ThreadLocal`s.

### 56. Strong, soft, weak, and phantom references?

- **Strong:** ordinary reference; keeps the object alive.
- **Soft:** may be cleared under memory pressure; unsuitable for predictable cache policy.
- **Weak:** may be cleared once no strong/soft reachability remains; useful for canonical mappings or metadata with careful design.
- **Phantom:** enqueued after the object becomes phantom reachable; useful for post-mortem resource tracking with `ReferenceQueue`.

Reference types do not replace explicit resource management. Close files, sockets, and native resources deterministically.

### 57. What does a generational collector optimize for?

Most objects die young. Generational collectors place newly allocated objects into a young generation and collect it frequently; survivors may be promoted. Older regions are collected less often or concurrently depending on the collector.

The names and mechanics vary by collector. Senior analysis should focus on allocation rate, live-set size, pause and throughput goals, promotion, humongous objects, and evidence from GC logs rather than memorized folklore.

### 58. How do you investigate an `OutOfMemoryError`?

1. Identify the exact OOME message and whether memory is heap or native.
2. Preserve evidence: heap dump near failure, GC logs, native-memory data, container metrics, thread count, and application metrics.
3. Compare dominator/retained-size paths to GC roots.
4. Determine whether the problem is a leak, insufficient sizing, excessive allocation, traffic growth, or native resource exhaustion.
5. Fix the retention/allocation cause and validate under representative load.

Increasing the heap can delay a leak and lengthen collection pauses without fixing it.

### 59. What are class loading and parent delegation?

Class loaders load class bytes and define runtime classes. A class's identity is its binary name plus its defining class loader. Parent-first delegation normally asks the parent before attempting local loading, protecting platform classes and encouraging consistency.

Application servers, plugin systems, and hot reload may use multiple loaders. A static field is singleton only within one loaded class identity, and class-loader leaks can retain entire application graphs.

### 60. What does the JIT compiler do?

The JVM begins with interpreted or lightly compiled execution, profiles hot code, and compiles frequently executed paths with optimizations such as inlining, escape analysis, lock elimination, and speculative optimization. Invalidated assumptions can cause deoptimization.

This makes naive microbenchmarks misleading. Use JMH, allow warmup, consume results, isolate setup, consider constant folding and dead-code elimination, and measure the actual production objective.

### 61. What is safe publication?

Safe publication ensures another thread sees a fully initialized object. It can be achieved through static initialization, storing into a volatile field, publishing under the same lock used by readers, thread-safe collections, or correctly constructed objects with final-field guarantees.

Letting `this` escape from a constructor—such as registering a listener before construction finishes—can expose default or partially initialized state.

---

## 8. Modern Java

### 62. What are records, and are they immutable?

A record is a concise nominal data carrier with final component fields, accessors, a canonical constructor, and generated `equals`, `hashCode`, and `toString`. Records are implicitly final and can implement interfaces.

They are shallowly immutable: component references cannot be reassigned, but referenced objects may still mutate. Validate and defensively copy mutable components in the canonical or compact constructor.

```java
record Order(String id, List<String> items) {
    Order {
        Objects.requireNonNull(id);
        items = List.copyOf(items);
    }
}
```

### 63. What are sealed classes?

A sealed class or interface restricts direct permitted subtypes. Each permitted subtype must be `final`, `sealed`, or `non-sealed`. This is useful for closed domain alternatives and enables exhaustive pattern matching.

Use sealing when the set of variants is intentionally controlled. Do not seal an extension API that third parties are expected to implement freely.

### 64. What does pattern matching improve?

Pattern matching combines a type test with safe extraction, reducing casts and making algebraic-style domain handling clearer. Java supports patterns for `instanceof`, record patterns, and pattern matching in `switch` in modern releases.

```java
static BigDecimal total(Payment payment) {
    return switch (payment) {
        case CardPayment(var amount, var ignored) -> amount;
        case CashPayment(var amount) -> amount;
    };
}
```

With a sealed hierarchy, the compiler can check exhaustiveness. Keep domain behavior on objects when polymorphism is the better design; pattern matching is especially useful when operations vary independently of a closed data hierarchy.

### 65. What are modules, and why are they different from packages?

The Java Platform Module System groups packages into named modules with explicit dependencies and exported packages. A package controls source-level names and access; a module controls readability and strong encapsulation across package boundaries.

`requires` declares dependencies, `exports` exposes API packages, `opens` permits deep reflection, and `uses`/`provides` support services. The unnamed module preserves classpath compatibility. Framework migrations can require targeted `opens`, but broadly opening everything loses encapsulation benefits.

### 66. What is `var`?

`var` requests local-variable type inference; Java remains statically typed and the compiler fixes the type from the initializer. It is limited to local contexts and does not make Java dynamically typed.

Use it when the type is obvious or repeated and the variable name communicates intent. Avoid it when the inferred type is surprising or important to understanding the code. It cannot infer from `null` alone.

---

## 9. Design and production scenarios

### 67. How would you design a thread-safe in-memory cache?

Start by defining semantics: maximum size, expiry, eviction, concurrent loading, failure caching, null handling, consistency, and metrics. `ConcurrentHashMap` alone provides thread-safe storage but not bounded memory or complete cache policy.

Avoid a cache stampede by coordinating one load per key, but ensure failed or cancelled loads do not poison the entry forever. In production, prefer a proven cache library unless the requirement is deliberately minimal.

### 68. How do you avoid resource leaks in Java services?

Use try-with-resources for deterministic lifetime, bound executors and queues, shut down owned thread pools, remove `ThreadLocal` values in pooled threads, unregister listeners, close HTTP responses/streams, and give caches eviction policies.

Ownership must be explicit: the component that creates or acquires a resource should know whether it owns closing it. GC reclaims Java memory; it does not provide timely release of file descriptors, sockets, or database connections.

### 69. How do you choose a collection?

Choose from behavior, not habit:

1. Sequence, uniqueness, key lookup, priority, or double-ended queue?
2. Is ordering required— insertion, access, sorted, or none?
3. What are the dominant operations and data size?
4. Is concurrent access required, and what consistency does iteration need?
5. Are nulls, duplicates, or mutable keys possible?
6. Must memory be bounded?

Then measure if performance matters. Big-O omits allocation, cache locality, contention, and actual distributions.

### 70. How would you diagnose high CPU in a Java process?

Correlate process/container CPU with thread-level evidence. Take multiple thread dumps or a profile, identify repeatedly runnable hot threads and stacks, then connect them to request traces, workload, lock contention, GC activity, compilation, or a tight loop.

Possible causes include inefficient algorithms, retry storms, serialization, regex backtracking, excessive allocation/GC, busy waiting, lock spinning, and too much parallelism. Optimize the measured hot path and retest; do not infer it from a single snapshot.

### 71. How would you diagnose a hanging Java service?

Check whether it is deadlocked, blocked on external I/O, waiting for a depleted connection or thread pool, overloaded behind an unbounded queue, paused by GC, or unable to make scheduler progress. Gather thread dumps over time, pool metrics, dependency latency, traces, socket state, and GC logs.

A service can appear idle while all request threads wait on one downstream call. Always configure deadlines and expose saturation metrics for each bounded resource.

### 72. What distinguishes a senior Core Java answer?

- States contracts precisely instead of relying on folklore.
- Connects `equals` and immutability to collection correctness.
- Chooses collections from access patterns and concurrency semantics.
- Explains concurrency with happens-before, not “it usually works.”
- Treats pools, queues, caches, and memory as bounded resources.
- Distinguishes source-level models from JVM optimizations.
- Uses profiling, dumps, logs, and benchmarks as evidence.
- Understands that modern features improve expression but do not replace sound design.

---

## 10. Rapid revision

### Must-answer questions

Before an interview, answer these without notes:

1. Why is Java always pass-by-value?
2. `==` versus `equals()`?
3. Overloading versus overriding?
4. Interface versus abstract class?
5. What contracts connect `equals()` and `hashCode()`?
6. How do you make a class immutable?
7. What does type erasure prevent?
8. Explain PECS.
9. How does `HashMap` find a key?
10. Why are mutable map keys dangerous?
11. `ArrayList` versus `LinkedList`?
12. Unmodifiable view versus immutable snapshot?
13. Checked versus unchecked exceptions?
14. How are suppressed exceptions produced?
15. `map` versus `flatMap`?
16. Why can parallel streams hurt server throughput?
17. Atomicity versus visibility?
18. `synchronized` versus `volatile`?
19. What is a happens-before relationship?
20. How do you prevent deadlock?
21. How should a thread pool be bounded?
22. How should interruption be handled?
23. Platform thread versus virtual thread?
24. How does the JVM find collectible objects?
25. What causes a Java memory leak?
26. Stack versus heap versus metaspace?
27. How do you investigate OOME or high CPU?
28. Why are naive microbenchmarks unreliable?
29. Are records deeply immutable?
30. When are sealed classes useful?

### Thirty-second summary

Java is pass-by-value, dynamically dispatches overridden instance methods, and relies on explicit object contracts for equality and ordering. Generics provide compile-time safety mainly through erasure; collections differ in ordering, complexity, memory, and concurrency semantics. Correct concurrent code requires a happens-before relationship and protection of complete invariants. The JVM manages reachable memory, optimizes hot code, and can still leak resources or strongly reachable objects. Senior Java engineering means knowing these contracts, bounding resources, and validating performance claims with evidence.

## Official references

- [Java Language Specification](https://docs.oracle.com/javase/specs/)
- [Java SE API](https://docs.oracle.com/en/java/javase/21/docs/api/)
- [Java Collections Framework](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/doc-files/coll-index.html)
- [Java concurrency guide](https://docs.oracle.com/en/java/javase/21/core/concurrency.html)
- [Java language changes](https://docs.oracle.com/en/java/javase/21/language/java-language-changes-release.html)
- [OpenJDK JEP index](https://openjdk.org/jeps/0)

