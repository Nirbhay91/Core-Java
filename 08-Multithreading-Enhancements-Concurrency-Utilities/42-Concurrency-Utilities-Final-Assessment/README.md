# 8.42 — Concurrency Utilities Final Assessment

> **Goal:** Final interview-level assessment for Chapter 8. Do this **without looking at notes first**. Write the code, explain your design, identify risks, and justify the concurrency utility you selected.

## 🎯 Assessment Rules

**Recommended time:** 90–120 minutes  
**Level:** 5-year Java developer / interview level  
**Rule:** First attempt from memory. Open previous chapter notes only after completing an attempt.

### Scoring

| Section | Marks |
|---|---:|
| Executor / Thread Pools | 15 |
| Synchronization Utilities | 15 |
| Locks / Atomics / Concurrent Collections | 20 |
| `CompletableFuture` | 20 |
| Fork/Join / Parallelism | 10 |
| Production / Design Scenarios | 20 |
| **Total** | **100** |

### Readiness

```text
90–100  → Interview Ready ⭐⭐⭐⭐⭐
80–89   → Strong; revise weak areas
70–79   → Good foundation; practice code again
60–69   → Needs another revision cycle
<60     → Revisit Chapter 8 fundamentals
```

---

# Part A — Executor & Thread Pool Coding

## Q1. Fixed Thread Pool — 5 marks

Create an `ExecutorService` with 4 workers and submit 10 independent tasks.

Requirements:

- Print task number and executing thread name.
- Do not create one thread per task.
- Shut down the executor correctly.

### Expected skeleton

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

for (int i = 1; i <= 10; i++) {
    final int taskId = i;
    executor.submit(() -> {
        // your code
    });
}

// graceful shutdown
```

**Interview follow-up:** Why is a thread pool preferable to creating 10 raw threads?

---

## Q2. `execute()` vs `submit()` — 3 marks

Write two versions of a task that throws `RuntimeException`:

1. using `execute()`
2. using `submit()` and `Future`

Explain where/how the exception becomes observable.

---

## Q3. Bounded Thread Pool + Backpressure — 7 marks ⭐⭐⭐⭐⭐

Create a `ThreadPoolExecutor` with:

```text
corePoolSize = 2
maximumPoolSize = 4
queueCapacity = 2
RejectedExecutionHandler = CallerRunsPolicy
```

Submit 20 tasks.

### Interview questions

- What happens after workers and queue are saturated?
- Why can `CallerRunsPolicy` provide backpressure?
- Why is an unbounded queue potentially dangerous?

---

# Part B — Coordination Utilities

## Q4. `CountDownLatch` — 5 marks

Simulate application startup with 4 independent services:

```text
Database
Cache
Messaging
Configuration
```

The main thread must continue only after all four initialization tasks finish.

Use `CountDownLatch`.

### Follow-up
Can the same latch be reset and reused for another startup cycle?

---

## Q5. `CyclicBarrier` — 5 marks

Create 4 worker threads. Each worker must:

```text
Phase 1 work
   ↓
wait at barrier
   ↓
Phase 2 work
   ↓
wait at barrier
```

Use `CyclicBarrier`.

### Follow-up
Why is this different from `CountDownLatch`?

---

## Q6. `Phaser` — 5 marks

Create workers that can dynamically register/deregister and coordinate across two phases.

Use:

```java
register()
arriveAndAwaitAdvance()
arriveAndDeregister()
```

### Explain
Why is `Phaser` more flexible than `CyclicBarrier` for dynamic participants?

---

## Q7. `Semaphore` — 5 marks ⭐⭐⭐⭐⭐

You have only **3 database connections** available for 10 concurrent tasks.

Implement access control with:

```java
Semaphore semaphore = new Semaphore(3);
```

### Mandatory
Use `try/finally` so permits are always released after successful acquisition.

### Follow-up
What happens if a task acquires a permit and throws an exception before releasing it?

---

# Part C — Locks & Shared State

## Q8. `ReentrantLock` + `tryLock()` — 5 marks

Two threads need the same critical section, but neither should wait indefinitely.

Implement a timed lock acquisition:

```java
lock.tryLock(500, TimeUnit.MILLISECONDS)
```

Requirements:

- Handle `InterruptedException`.
- Unlock only after successful acquisition.
- Put `unlock()` in `finally`.

---

## Q9. `ReentrantReadWriteLock` — 5 marks

Implement a small in-memory cache where:

- many threads can read concurrently
- writes require exclusive access

Explain when this design can help and when a simple lock may be preferable.

---

## Q10. `StampedLock` — 5 marks

Implement an optimistic-read pattern:

```java
long stamp = lock.tryOptimisticRead();
// read state
if (!lock.validate(stamp)) {
    // fall back to read lock
}
```

### Mandatory interview point
Explain why `StampedLock` is **not reentrant** and why optimistic reads must be validated.

---

## Q11. Atomic Counter — 5 marks ⭐⭐⭐⭐⭐

Create 10 threads, each incrementing a counter 10,000 times.

Implement three versions:

1. unsafe `int`
2. `AtomicInteger`
3. `LongAdder`

Compare the result and explain why the unsafe version can lose updates.

---

## Q12. `volatile` Trap — 5 marks

Consider:

```java
volatile int count = 0;

count++;
```

Explain why `volatile` does **not** make `count++` atomic.

Then replace it with an appropriate atomic solution.

---

# Part D — Concurrent Collections

## Q13. Concurrent Word Counter — 5 marks ⭐⭐⭐⭐⭐

Multiple threads process text and update word frequencies.

Use:

```java
ConcurrentHashMap<String, Integer>
```

Do not write a non-atomic pattern like:

```java
if (!map.containsKey(word)) {
    map.put(word, 1);
}
```

Use an atomic map operation such as:

```java
merge()
```

### Follow-up
Why is `merge()` safer for this compound update?

---

## Q14. `CopyOnWriteArrayList` — 3 marks

Design a listener registry where reads/iterations happen very frequently and listener registration/removal is rare.

Explain why `CopyOnWriteArrayList` can fit this workload.

Also explain why it is a poor choice for write-heavy workloads.

---

## Q15. Producer-Consumer — 7 marks ⭐⭐⭐⭐⭐

Build a producer-consumer system using:

```java
BlockingQueue<Integer>
```

Requirements:

- bounded queue
- 2 producers
- 3 consumers
- producers use `put()`
- consumers use `take()`
- graceful termination

### Follow-up
Why is `BlockingQueue` generally preferable to manually implementing `wait()` / `notify()` for this problem?

---

# Part E — `CompletableFuture`

## Q16. Basic Async Pipeline — 5 marks

Create an asynchronous pipeline:

```text
fetchUser()
    ↓
transformUser()
    ↓
return DTO
```

Use:

```java
supplyAsync()
thenApply()
```

---

## Q17. `thenCompose()` — 5 marks ⭐⭐⭐⭐⭐

Implement:

```text
fetchUser()
      ↓
fetchOrders(userId)
```

Both operations return `CompletableFuture`.

Explain why `thenCompose()` is preferred over `thenApply()` here.

### Memory line

```text
thenApply   → A → B
thenCompose → A → Future<B>
```

---

## Q18. `thenCombine()` — 5 marks ⭐⭐⭐⭐⭐

Two independent APIs run concurrently:

```text
getUserProfile()
getAccountBalance()
```

Combine their results into a single response.

Use `thenCombine()`.

---

## Q19. `allOf()` — 5 marks

Run three independent asynchronous operations:

```text
fetchOrders()
fetchPayments()
fetchNotifications()
```

Wait until all complete using `CompletableFuture.allOf()`.

### Trap
Explain why `allOf()` returns `CompletableFuture<Void>` and how you would collect the individual results.

---

## Q20. `anyOf()` — 5 marks

Call three equivalent services and use the first completed result:

```text
Service A
Service B
Service C
```

Use `anyOf()`.

### Follow-up
Does `anyOf()` mean “first successful result”? Explain the distinction between completion and successful completion.

---

## Q21. Exception Handling — 5 marks ⭐⭐⭐⭐⭐

Create a `CompletableFuture` pipeline that can fail.

Implement recovery with:

```java
exceptionally()
```

Then implement an alternative using:

```java
handle()
```

Explain:

```text
exceptionally → recovery
handle        → inspect + transform
whenComplete  → observe/side effect
```

---

## Q22. Custom Executor — 5 marks

Run a blocking operation using a dedicated executor rather than blindly using the default/common pool.

```java
ExecutorService executor =
        Executors.newFixedThreadPool(10);

CompletableFuture.supplyAsync(
        () -> blockingOperation(),
        executor
);
```

Explain why executor choice matters for blocking I/O.

---

# Part F — Fork/Join & Parallelism

## Q23. Recursive Task — 5 marks

Implement a `RecursiveTask<Long>` that recursively sums a large array.

Concept:

```text
large problem
     ↓
split
  ↙     ↘
small   small
  ↓       ↓
compute + compute
      ↓
     join
```

Explain work stealing.

---

## Q24. Parallel Stream Risk Analysis — 5 marks

Review:

```java
orders.parallelStream()
      .map(this::callPaymentService)
      .toList();
```

Identify at least five production risks.

Expected areas:

```text
common pool
blocking I/O
DB capacity
remote-service limits
task granularity
contention
latency
ordering
```

---

# Part G — Production Scenarios 🔥

## Q25. Thread Pool Saturation — 5 marks

Production symptoms:

```text
CPU = 40%
queue size = constantly increasing
request latency = increasing
rejections = increasing
```

### Questions

1. What does this suggest?
2. Would increasing the thread count automatically solve it?
3. What downstream dependency might be saturated?
4. What metrics would you inspect?
5. How can backpressure help?

---

## Q26. DB Connection Pool Exhaustion — 5 marks

You have:

```text
HTTP worker threads = 100
DB connections = 10
```

100 requests simultaneously require DB access.

### Explain

- Why creating more application threads may not help.
- How excessive concurrency can increase waiting.
- How a semaphore or bounded executor can protect the DB.
- Why concurrency limits should align with downstream capacity.

---

## Q27. Deadlock Diagnosis — 5 marks ⭐⭐⭐⭐⭐

Two threads:

```text
Thread A: lock(A) → lock(B)
Thread B: lock(B) → lock(A)
```

### Tasks

1. Identify the deadlock.
2. Explain circular wait.
3. Provide a consistent lock-ordering solution.
4. Explain how `tryLock()` can help avoid indefinite waiting.

---

## Q28. Graceful Shutdown — 5 marks ⭐⭐⭐⭐⭐

Write production-style shutdown code for an executor.

It must:

- stop accepting new tasks
- wait for existing tasks
- force interruption after timeout
- preserve interrupt status
- report failure if termination still does not occur

### Expected pattern

```java
executor.shutdown();
try {
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
        executor.shutdownNow();
        if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
            System.err.println("Executor did not terminate");
        }
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

---

# Part H — Design Challenge 🏆

## Q29. Production API Aggregator — 10 marks

Design an API aggregator that receives one request and needs:

```text
User Profile API
Order API
Payment API
Recommendation API
```

### Requirements

- independent calls should execute concurrently
- each call should have timeout/failure handling
- combine successful results
- do not block unnecessarily
- use a dedicated executor for blocking operations where appropriate
- cleanly shut down owned executors

### Suggested architecture

```text
                 Request
                    |
              Aggregator
                    |
       +------------+------------+
       |            |            |
     User         Orders      Payments   Recommendations
       |            |            |            |
       +------------+------------+------------+
                    |
             CompletableFuture
                    |
               Combined DTO
```

### Interview discussion
Be ready to explain:

- `thenCombine()` vs `allOf()`
- timeout strategy
- fallback strategy
- cancellation
- executor sizing
- downstream rate limits
- thread-pool saturation
- observability

---

# Part I — Rapid-Fire Final Round ⚡

Answer each in **10–20 seconds**.

1. What problem does `ExecutorService` solve?
2. `execute()` vs `submit()`?
3. `shutdown()` vs `shutdownNow()`?
4. Why is `shutdownNow()` not a guaranteed kill?
5. What does `Future.get()` do?
6. Why can `get()` be dangerous on a request thread?
7. Latch vs Barrier?
8. Barrier vs Phaser?
9. Why use `Semaphore`?
10. Does `Semaphore` provide ownership semantics like a lock?
11. Why use `tryLock()`?
12. Is `StampedLock` reentrant?
13. Why validate optimistic reads?
14. Why does `volatile` not make `count++` atomic?
15. `AtomicLong` vs `LongAdder`?
16. Why use `ConcurrentHashMap`?
17. Why can `CopyOnWriteArrayList` be expensive on writes?
18. Why use `BlockingQueue`?
19. `thenApply()` vs `thenCompose()`?
20. `thenCompose()` vs `thenCombine()`?
21. `allOf()` vs `anyOf()`?
22. How do you handle `CompletableFuture` exceptions?
23. Why provide a custom executor?
24. What is work stealing?
25. What are parallel-stream risks?
26. What is backpressure?
27. What happens when a bounded executor queue is full?
28. How do you size a thread pool?
29. How do you detect deadlock?
30. Why is graceful shutdown important?

---

# Part J — Final Coding Challenge Set 🔥

Do these **without copying previous solutions**.

### Challenge 1 — Concurrent Counter

10 threads × 100,000 increments.

Implement:

```text
synchronized
AtomicInteger
LongAdder
```

Compare correctness and discuss contention.

### Challenge 2 — Rate-Limited Resource

10 tasks, maximum 3 concurrent resource users.

Use `Semaphore`.

### Challenge 3 — Startup Coordinator

5 services initialize concurrently.

Main thread proceeds only after all complete.

Use `CountDownLatch`.

### Challenge 4 — Producer Consumer

Build a bounded producer-consumer system using `BlockingQueue`.

### Challenge 5 — Concurrent Cache

Build a read-heavy cache with:

```text
ReentrantReadWriteLock
```

Then discuss whether `ConcurrentHashMap` could simplify it.

### Challenge 6 — Async Aggregator

Run three independent async calls and combine them.

Use:

```text
CompletableFuture
thenCombine
exceptionally
custom Executor
```

### Challenge 7 — Service Race

Call three equivalent services and use the first completed response.

Use `anyOf()` and discuss cancellation of remaining work.

### Challenge 8 — Recursive Sum

Use Fork/Join to sum one million integers.

### Challenge 9 — Bounded Executor

Build:

```text
2 core threads
4 max threads
queue size 5
CallerRunsPolicy
```

Submit 30 tasks and observe backpressure.

### Challenge 10 — Graceful Shutdown

Write the shutdown handler entirely from memory.

---

# Part K — Interview Scenario Answers Framework 🧠

When asked **“Which concurrency utility would you use?”**, answer using this structure:

```text
1. Identify the problem.
2. Identify whether the workload is CPU-bound or I/O-bound.
3. Identify shared state / coordination requirement.
4. Identify capacity/downstream limits.
5. Choose the smallest suitable utility.
6. Explain failure/interruption behavior.
7. Explain shutdown/cancellation.
8. Mention monitoring/backpressure.
```

Example:

> “For a producer-consumer workflow, I would start with a bounded `BlockingQueue` because it gives me thread-safe handoff plus blocking semantics. I would size the consumers based on the workload and downstream capacity, and the bounded queue provides natural backpressure. I would also define a clean termination strategy rather than leaving consumers blocked forever.”

---

# Part L — Final Cheat Sheet ⭐⭐⭐⭐⭐

```text
ExecutorService
→ task execution + lifecycle

ScheduledExecutorService
→ delayed/repeated tasks

Callable + Future
→ result of async task

CountDownLatch
→ one-time waiting

CyclicBarrier
→ reusable fixed-party synchronization

Phaser
→ dynamic multi-phase synchronization

Semaphore
→ limit concurrent access

ReentrantLock
→ explicit lock + advanced features

tryLock
→ bounded lock waiting

ReadWriteLock
→ separate read/write access

StampedLock
→ optimistic reads

Condition
→ lock-based condition waiting

Atomic
→ lock-free atomic state operations

LongAdder
→ high-contention counters/metrics

ConcurrentHashMap
→ concurrent map operations

CopyOnWriteArrayList
→ read-heavy / write-rare list

BlockingQueue
→ producer-consumer + backpressure

CompletableFuture
→ composable async pipeline

thenApply
→ transform

thenCompose
→ dependent async chain

thenCombine
→ combine independent futures

allOf
→ wait for all

anyOf
→ first completion

Fork/Join
→ recursive parallel computation

ThreadPoolExecutor
→ explicit pool + queue + rejection policy

Graceful shutdown
→ shutdown → await → shutdownNow → await
```

---

# 🏆 Final Assessment Checklist

Before moving to Chapter 9, you should be able to:

- [ ] Write a `ThreadPoolExecutor` from memory.
- [ ] Explain queueing and rejection policies.
- [ ] Implement graceful shutdown.
- [ ] Choose Latch vs Barrier vs Phaser.
- [ ] Use `Semaphore` for concurrency limits.
- [ ] Write safe `ReentrantLock` code.
- [ ] Use `tryLock()` correctly.
- [ ] Explain `StampedLock` optimistic reads.
- [ ] Explain Atomic vs `volatile`.
- [ ] Choose `LongAdder` vs `AtomicLong`.
- [ ] Use `ConcurrentHashMap.merge()`.
- [ ] Implement producer-consumer with `BlockingQueue`.
- [ ] Write `CompletableFuture` pipelines.
- [ ] Explain `thenApply` / `thenCompose` / `thenCombine`.
- [ ] Use `allOf()` / `anyOf()` correctly.
- [ ] Handle async exceptions.
- [ ] Choose a custom executor appropriately.
- [ ] Explain Fork/Join and work stealing.
- [ ] Identify parallel-stream risks.
- [ ] Diagnose deadlock/starvation/livelock.
- [ ] Explain backpressure.
- [ ] Discuss thread-pool sizing.
- [ ] Design concurrency around downstream capacity.
- [ ] Answer Chapter 8 concurrency scenarios in 2 minutes.

---

# 🎓 Final Interview Rule

> **Do not memorize concurrency APIs in isolation. Identify the problem first, then choose the utility.**

```text
Problem
  ↓
Coordination? → Latch / Barrier / Phaser
Capacity?     → Semaphore / bounded executor / queue
Shared state? → Atomic / concurrent collection / lock
Async flow?   → CompletableFuture
Parallel work?→ Fork/Join
Task execution?→ ExecutorService
Production?   → sizing + backpressure + interruption + shutdown + monitoring
```

---

## Navigation

[← 8.41 — Concurrency Utilities Quick Revision](../41-Concurrency-Utilities-Quick-Revision/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Chapter 8 — Multithreading Enhancements / Concurrency Utilities → ✅ COMPLETE**