# Live Problem-Solving — Senior Interview Guide

This chapter prepares you for the screen-share problem-solving round: a small, open-ended exercise where engineers watch how you think, structure a solution, and react to feedback. They care less about a perfect final answer than about your reasoning, communication, and engineering judgment. All code below was compiled and run on a current JDK.

> **How to answer:** clarify the requirements, state a simple approach and its complexity, code the smallest correct version while narrating, test it with examples and edge cases, then discuss trade-offs, concurrency, and how it would change in production.

## Contents

1. [Method and communication](#1-method-and-communication)
2. [Worked exercises](#2-worked-exercises)
3. [Rapid revision](#3-rapid-revision)

---

## 1. Method and communication

### 1. What is a reliable structure for a live problem-solving exercise?

Use the same sequence every time so you never freeze:

1. **Clarify.** Restate the problem. Ask about inputs and sizes, null or empty values, duplicates, ordering, thread safety, memory limits, and what to do on invalid input.
2. **Examples.** Write one normal and one edge example before coding. This exposes misunderstandings cheaply.
3. **Approach.** Say the brute-force idea first, give its complexity, then improve it. Name the data structure and why.
4. **Code.** Write small, named methods. Narrate decisions, not keystrokes. Prefer clarity over cleverness.
5. **Test.** Walk through your examples by hand, then the edge cases. Run the code if you can.
6. **Discuss.** State complexity, limits, and what you would change for concurrency, scale, or failure.

Trap: starting to type immediately. A silent first five minutes of coding hides the part the interviewers are evaluating.

### 2. How do you think aloud without rambling?

Narrate *decisions and reasons* in short sentences: "I'll use a `HashMap` for O(1) lookup, because we need repeated membership checks." Name trade-offs explicitly ("this is O(n log n) because of the sort; if the input were already sorted I could do it in one pass"). Pause to check in: "Does this match what you had in mind?" Do not narrate syntax, and do not go quiet while debugging; say what you suspect and how you will confirm it.

### 3. What do you do when you get stuck or make a mistake?

Say so calmly, then use a systematic recovery path: re-read the requirements, shrink the problem to a tiny example, trace it by hand, and state the invariant your code should maintain. Offer a working simpler solution first and improve it. Treat hints as collaboration, not failure: restate the hint and show how it changes your approach. Interviewers frequently rate recovery behavior above getting the answer instantly.

### 4. What do interviewers evaluate beyond correctness?

- Clarifying questions and requirement handling.
- Sensible data-structure choice with stated complexity.
- Readable code: naming, small methods, no premature abstraction.
- Edge cases and testing discipline.
- Awareness of production concerns: thread safety, memory growth, error handling, observability.
- Communication and how you respond to feedback.
- Honesty about what you do not know.

---

## 2. Worked exercises

### 5. Implement an LRU cache.

Clarify capacity, null handling, and thread safety. The direct answer: a hash map for O(1) lookup plus an ordering structure (doubly linked list) for O(1) eviction. In Java, `LinkedHashMap` in access-order mode provides both; say you can hand-roll the map and list if asked.

```java
final class LruCache<K, V> {
    private final int capacity;
    private final LinkedHashMap<K, V> map;

    LruCache(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("capacity must be positive");
        this.capacity = capacity;
        this.map = new LinkedHashMap<>(16, 0.75f, true) { // true = access order
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > LruCache.this.capacity;
            }
        };
    }

    synchronized V get(K key) { return map.get(key); }          // get() reorders, so it needs the lock too
    synchronized void put(K key, V value) { map.put(key, value); }
}
```

Discussion points: `get` mutates order, so even reads need synchronization; a single lock limits throughput, so mention striping or a library such as Caffeine for production; add TTL, metrics, and stampede protection if the cache fronts a slow source. Expect "now implement the linked list yourself" as a follow-up.

### 6. Design a rate limiter (token bucket).

Clarify per-user or global, burst behavior, and single node versus distributed. A token bucket allows bursts up to `capacity` and refills at a steady rate. Inject the clock so the logic is testable.

```java
final class TokenBucket {
    private final long capacity;
    private final double refillPerNano;
    private final LongSupplier clock;
    private double tokens;
    private long last;

    TokenBucket(long capacity, double perSecond, LongSupplier nanoClock) {
        this.capacity = capacity;
        this.refillPerNano = perSecond / 1_000_000_000.0;
        this.clock = nanoClock;
        this.tokens = capacity;
        this.last = nanoClock.getAsLong();
    }

    synchronized boolean tryAcquire() {
        long now = clock.getAsLong();
        tokens = Math.min(capacity, tokens + (now - last) * refillPerNano);
        last = now;
        if (tokens >= 1) { tokens -= 1; return true; }
        return false;
    }
}
```

Discussion points: use `System.nanoTime` (monotonic) rather than wall-clock time; keep one bucket per key in a bounded or expiring map to avoid memory growth; a distributed limiter needs shared state (for example an atomic script in Redis) and accepts trade-offs between accuracy and latency. Compare with fixed window (boundary bursts), sliding window (more precise, more state), and leaky bucket (smooths output).

### 7. Return the top K most frequent words.

Count with a `HashMap` (O(n)), then keep a min-heap of size K (O(m log K) for m distinct words) instead of sorting everything (O(m log m)). Define tie-breaking up front.

```java
static List<String> topK(Collection<String> words, int k) {
    Map<String, Long> freq = words.stream()
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

    // min-heap: the weakest candidate (lowest count, then alphabetically last) is evicted first
    PriorityQueue<Map.Entry<String, Long>> heap = new PriorityQueue<>(
        Map.Entry.<String, Long>comparingByValue()
            .thenComparing(Map.Entry.<String, Long>comparingByKey().reversed()));

    for (var e : freq.entrySet()) {
        heap.offer(e);
        if (heap.size() > k) heap.poll();
    }
    List<String> out = new ArrayList<>();
    while (!heap.isEmpty()) out.add(heap.poll().getKey());
    Collections.reverse(out);
    return out;
}
```

Discussion points: for a stream too large for memory, discuss approximate counting (count-min sketch) or partitioned counting and merging; mention that `k >= distinct` and `k <= 0` need defined behavior.

### 8. Merge overlapping intervals.

Sort by start, then sweep, extending the last interval while the next one overlaps. Complexity is O(n log n) for the sort and O(n) for the sweep.

```java
record Interval(int start, int end) {}

static List<Interval> merge(List<Interval> in) {
    List<Interval> sorted = new ArrayList<>(in);          // do not mutate the caller's list
    sorted.sort(Comparator.comparingInt(Interval::start));
    List<Interval> out = new ArrayList<>();
    for (Interval i : sorted) {
        if (out.isEmpty() || out.get(out.size() - 1).end() < i.start()) {
            out.add(i);
        } else {
            Interval last = out.remove(out.size() - 1);
            out.add(new Interval(last.start(), Math.max(last.end(), i.end())));
        }
    }
    return out;
}
```

Edge cases to say aloud: empty input, touching intervals (`[1,3]` and `[3,5]` — merge or not? ask), fully nested intervals (this is why `Math.max` is needed), and invalid intervals where `start > end`.

### 9. Implement a bounded blocking queue.

Clarify blocking versus failing when full or empty, fairness, and interruption. Use a circular buffer with one lock and two conditions. Always wait in a `while` loop, because of spurious wakeups and because another thread may consume the state change first.

```java
final class BoundedQueue<T> {
    private final Object[] items;
    private int head, tail, count;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    BoundedQueue(int capacity) { items = new Object[capacity]; }

    void put(T t) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == items.length) notFull.await();
            items[tail] = t;
            tail = (tail + 1) % items.length;
            count++;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    T take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == 0) notEmpty.await();
            T t = (T) items[head];
            items[head] = null;                     // avoid retaining the reference
            head = (head + 1) % items.length;
            count--;
            notFull.signal();
            return t;
        } finally {
            lock.unlock();
        }
    }
}
```

Discussion points: `signal` is enough because each condition has a single kind of waiter; `await` releases the lock and re-acquires it before returning; in production, use `ArrayBlockingQueue`. Mention timeouts (`awaitNanos`) and `close` or poison-pill semantics if asked.

### 10. Write a retry helper with exponential backoff and jitter.

State the policy first: retry only transient, idempotent failures; cap attempts and delay; add jitter so many clients do not retry in lockstep; never retry non-idempotent operations without an idempotency key.

```java
static <T> T retry(Callable<T> call, int maxAttempts, long baseMillis,
                   Predicate<Exception> retryable) throws Exception {
    long delay = baseMillis;
    for (int attempt = 1; ; attempt++) {
        try {
            return call.call();
        } catch (Exception e) {
            if (attempt >= maxAttempts || !retryable.test(e)) throw e;
            Thread.sleep(ThreadLocalRandom.current().nextLong(delay + 1)); // full jitter
            delay = Math.min(delay * 2, 5_000);                           // capped exponential
        }
    }
}
```

Discussion points: propagate `InterruptedException` instead of swallowing it; add an overall deadline, not only an attempt count; pair with a circuit breaker so retries do not amplify an outage; in Spring use Resilience4j or Spring Retry rather than hand-rolled code.

### 11. You are given buggy code and asked to find the problem. How do you approach it?

Do not scan randomly. Reproduce or reason about the failing input, form a hypothesis, then confirm it. Typical Java traps to check in order: wrong `equals`/`hashCode` on a map key, mutation of a collection during iteration, integer overflow or boxing comparison with `==`, shared mutable state without synchronization, swallowed exceptions, resource leaks, off-by-one in boundaries, and time or locale assumptions. Explain what you checked and ruled out, fix the root cause rather than the symptom, and add the failing case as a test.

---

## 3. Rapid revision

### Must-answer questions

1. What are the steps you follow in a live exercise before writing code?
2. How do you communicate trade-offs while coding?
3. What do you do when you are stuck?
4. How do you implement an LRU cache, and why does `get` need a lock?
5. Token bucket versus fixed window versus sliding window?
6. Why use a min-heap of size K for top-K?
7. Why does interval merging need `Math.max`?
8. Why do condition waits use `while` rather than `if`?
9. Why add jitter to retries?
10. How do you debug unfamiliar code under time pressure?

### Thirty-second summary

A live exercise tests how you think, not whether you memorized a solution. Clarify, write examples, state a simple approach with its complexity, code small readable pieces while narrating decisions, test by hand with edge cases, and finish by discussing concurrency, scale, and failure. Recover calmly from mistakes, treat hints as collaboration, and connect even a small exercise to production concerns such as bounded memory, thread safety, and retries that do not amplify outages.

## Official references

- [Java Collections Framework](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/doc-files/coll-index.html)
- [`java.util.concurrent` package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
- [`LinkedHashMap` API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedHashMap.html)
- [AWS Builders' Library: Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
