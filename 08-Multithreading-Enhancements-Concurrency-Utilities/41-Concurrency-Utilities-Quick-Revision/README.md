# 8.41 — Concurrency Utilities Quick Revision

> **Goal:** Revise Chapter 8 quickly before an interview. Focus on choosing the right utility, remembering the core rule, and writing the minimum interview-ready code from memory.

## 1. One-Line Decision Map ⭐⭐⭐⭐⭐

| Problem | Remember |
|---|---|
| Run tasks asynchronously | `ExecutorService` |
| Return a result from a task | `Callable` + `Future` |
| Schedule tasks | `ScheduledExecutorService` |
| Limit concurrent access | `Semaphore` |
| Wait for one-time events | `CountDownLatch` |
| Repeated fixed-party synchronization | `CyclicBarrier` |
| Dynamic multi-phase coordination | `Phaser` |
| Exchange data between two threads | `Exchanger` |
| Producer-consumer | `BlockingQueue` |
| Simple atomic update | `AtomicInteger` / `AtomicLong` |
| High-contention metrics | `LongAdder` |
| Concurrent map | `ConcurrentHashMap` |
| Read-heavy rarely-written list | `CopyOnWriteArrayList` |
| Explicit lock | `ReentrantLock` |
| Lock timeout | `tryLock()` |
| Read/write locking | `ReentrantReadWriteLock` |
| Optimistic read | `StampedLock` |
| Lock-associated wait sets | `Condition` |
| Async composition | `CompletableFuture` |
| Recursive parallel computation | Fork/Join |

---

# 2. `ExecutorService` — Must Know ⭐⭐⭐⭐⭐

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

executor.execute(() -> System.out.println("Task"));

executor.shutdown();
```

### Interview line
> `ExecutorService` separates task submission from thread management and gives lifecycle control over a pool of worker threads.

### Remember

```text
execute() → no Future
submit()  → Future
shutdown() → stop accepting new tasks
shutdownNow() → request interruption of running tasks + return queued tasks
```

---

# 3. `execute()` vs `submit()` ⭐⭐⭐⭐⭐

```java
executor.execute(() -> doWork());

Future<?> future = executor.submit(() -> doWork());
```

| `execute()` | `submit()` |
|---|---|
| No `Future` | Returns `Future` |
| Runnable | Runnable / Callable |
| No result handle | Result/cancellation handle |

### Trap
Exceptions from a task submitted with `submit()` are captured by its `Future` rather than simply escaping to the caller.

---

# 4. `shutdown()` vs `shutdownNow()` ⭐⭐⭐⭐⭐

```java
executor.shutdown();
```

Means:

```text
No new tasks
Already submitted tasks can complete
```

```java
executor.shutdownNow();
```

Means:

```text
No new tasks
Attempt to interrupt running tasks
Return tasks that never started
```

### Golden line
> `shutdownNow()` is a request, not a guaranteed kill operation.

---

# 5. Graceful Shutdown — Memorize This Code 🏆

```java
static void shutdownGracefully(
        ExecutorService executor,
        long timeout,
        TimeUnit unit) {

    executor.shutdown();

    try {
        if (!executor.awaitTermination(timeout, unit)) {
            executor.shutdownNow();

            if (!executor.awaitTermination(timeout, unit)) {
                System.err.println("Executor did not terminate");
            }
        }
    } catch (InterruptedException e) {
        executor.shutdownNow();
        Thread.currentThread().interrupt();
    }
}
```

**Interview must-write.**

---

# 6. `Callable` + `Future` ⭐⭐⭐⭐⭐

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

Future<Integer> future = executor.submit(() -> 10 + 20);

System.out.println(future.get());

executor.shutdown();
```

Remember:

```text
Callable → returns value
Future   → represents pending/completed result
get()    → may block
cancel() → requests cancellation
```

---

# 7. `ScheduledExecutorService` ⭐⭐⭐⭐

```java
ScheduledExecutorService scheduler =
        Executors.newScheduledThreadPool(2);

scheduler.schedule(
        () -> System.out.println("Run once"),
        2,
        TimeUnit.SECONDS
);

scheduler.shutdown();
```

For repeated execution:

```java
scheduler.scheduleAtFixedRate(task, 0, 5, TimeUnit.SECONDS);
```

---

# 8. `CountDownLatch` ⭐⭐⭐⭐⭐

```java
CountDownLatch latch = new CountDownLatch(3);

// worker
latch.countDown();

// coordinator
latch.await();
```

### Remember
> One-shot coordination. Count only moves downward and does not reset.

---

# 9. `CyclicBarrier` ⭐⭐⭐⭐⭐

```java
CyclicBarrier barrier = new CyclicBarrier(3);

barrier.await();
```

### Remember
> Fixed number of parties meet at a common barrier and the barrier can be reused.

---

# 10. `Phaser` ⭐⭐⭐⭐

```java
Phaser phaser = new Phaser(1);

phaser.register();
phaser.arriveAndAwaitAdvance();
phaser.arriveAndDeregister();
```

### Remember
> Dynamic registration + multiple phases.

---

# 11. Latch vs Barrier vs Phaser 🔥

```text
CountDownLatch
→ one-time countdown

CyclicBarrier
→ fixed parties + reusable barrier

Phaser
→ dynamic parties + multiple phases
```

This comparison should be answered in **20–30 seconds**.

---

# 12. `Semaphore` ⭐⭐⭐⭐⭐

```java
Semaphore semaphore = new Semaphore(3);

boolean acquired = false;
try {
    semaphore.acquire();
    acquired = true;
    useResource();
} finally {
    if (acquired) {
        semaphore.release();
    }
}
```

### Remember
> Semaphore controls permits/concurrency; it is not primarily a mutual-exclusion lock.

---

# 13. `ReentrantLock` ⭐⭐⭐⭐⭐

```java
ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    criticalSection();
} finally {
    lock.unlock();
}
```

### Golden rule
> Every successful lock acquisition must have a matching unlock, normally in `finally`.

---

# 14. `tryLock()` ⭐⭐⭐⭐⭐

```java
if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
    try {
        criticalSection();
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("Could not acquire lock");
}
```

### Use when
> Waiting forever for a lock is undesirable.

---

# 15. `ReentrantReadWriteLock` ⭐⭐⭐⭐

```java
ReentrantReadWriteLock rw = new ReentrantReadWriteLock();

rw.readLock().lock();
try {
    readData();
} finally {
    rw.readLock().unlock();
}
```

### Remember

```text
Many readers can read together
Writer requires exclusive access
```

Useful for appropriate read-heavy workloads; it is not automatically faster than simpler locking.

---

# 16. `StampedLock` ⭐⭐⭐⭐

```java
long stamp = lock.tryOptimisticRead();

int value = data;

if (!lock.validate(stamp)) {
    stamp = lock.readLock();
    try {
        value = data;
    } finally {
        lock.unlockRead(stamp);
    }
}
```

### Must remember
> `StampedLock` supports optimistic reads but is **not reentrant**.

---

# 17. `Condition` ⭐⭐⭐⭐

```java
lock.lock();
try {
    while (!ready) {
        condition.await();
    }
} finally {
    lock.unlock();
}
```

Signal:

```java
lock.lock();
try {
    ready = true;
    condition.signalAll();
} finally {
    lock.unlock();
}
```

### Trap
Use a `while` loop to re-check the condition after waking.

---

# 18. Atomic Variables & CAS ⭐⭐⭐⭐⭐

```java
AtomicInteger counter = new AtomicInteger();

counter.incrementAndGet();
```

CAS idea:

```text
read current value
      ↓
compare with expected
      ↓
if equal → update
if changed → retry/fail
```

### Important
> Atomic variables are useful for supported atomic state transitions, but they do not replace locks for every multi-variable invariant.

---

# 19. `volatile` vs Atomic ⭐⭐⭐⭐⭐

```java
volatile boolean running = true;
```

Good for visibility of a simple state flag.

But:

```java
volatile int count;
count++; // NOT atomic
```

Use:

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```

### Interview line
> `volatile` gives visibility/order guarantees; it does not make compound operations atomic.

---

# 20. `LongAdder` vs `AtomicLong` ⭐⭐⭐⭐⭐

```java
LongAdder requests = new LongAdder();
requests.increment();
long total = requests.sum();
```

| `AtomicLong` | `LongAdder` |
|---|---|
| Strong atomic counter operations | Optimized for high-contention accumulation |
| Good for exact atomic state transitions | Excellent for metrics/statistics |
| Single value | Internally spreads contention |

### Golden line
> Use `LongAdder` for high-contention statistics when `sum()` semantics are sufficient.

---

# 21. `ConcurrentHashMap` ⭐⭐⭐⭐⭐

```java
ConcurrentHashMap<String, Integer> map =
        new ConcurrentHashMap<>();

map.putIfAbsent("A", 1);
map.merge("A", 1, Integer::sum);
map.computeIfAbsent("B", key -> 100);
```

### Trap
Avoid manually splitting an atomic operation:

```java
if (!map.containsKey(key)) {
    map.put(key, value);
}
```

Prefer the map's atomic methods where appropriate.

---

# 22. `CopyOnWriteArrayList` ⭐⭐⭐⭐

```java
CopyOnWriteArrayList<String> listeners =
        new CopyOnWriteArrayList<>();

listeners.add("A");

for (String listener : listeners) {
    System.out.println(listener);
}
```

### Remember
> Great for many reads/iterations and rare writes; writes are expensive because the backing array is copied.

---

# 23. `BlockingQueue` ⭐⭐⭐⭐⭐

```java
BlockingQueue<Integer> queue =
        new ArrayBlockingQueue<>(10);

queue.put(1);   // may block
Integer x = queue.take(); // may block
```

### Golden line
> `BlockingQueue` is a natural producer-consumer coordination primitive with built-in blocking semantics.

---

# 24. Queue Quick Comparison 🔥

| Queue | Remember |
|---|---|
| `ArrayBlockingQueue` | Bounded, array-backed |
| `LinkedBlockingQueue` | Linked-node based, optionally bounded |
| `PriorityBlockingQueue` | Priority ordering, not ordinary FIFO |
| `DelayQueue` | Element available after delay |
| `ConcurrentLinkedQueue` | Non-blocking concurrent queue |

---

# 25. `CompletableFuture` Fundamentals ⭐⭐⭐⭐⭐

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "Hello");

System.out.println(future.join());
```

Remember:

```text
Future → mainly represents async result
CompletableFuture → result + composition pipeline
```

---

# 26. `thenApply` / `thenCompose` / `thenCombine` 🔥

```java
thenApply   → A → transform result → B
thenCompose → A → async B depending on A
thenCombine → A + B independent → C
```

Examples:

```java
future.thenApply(String::toUpperCase);

future.thenCompose(this::fetchOrders);

futureA.thenCombine(futureB, (a, b) -> a + b);
```

### Memory trick

```text
Apply   = Transform
Compose = Chain
Combine = Join
```

---

# 27. `allOf()` / `anyOf()` ⭐⭐⭐⭐⭐

```java
CompletableFuture<Void> all =
        CompletableFuture.allOf(a, b, c);
```

All must complete.

```java
CompletableFuture<Object> any =
        CompletableFuture.anyOf(a, b, c);
```

First supplied future to complete wins.

### Trap
`allOf()` returns `CompletableFuture<Void>`, not a list of results.

---

# 28. `CompletableFuture` Exception Handling ⭐⭐⭐⭐⭐

```java
future.exceptionally(ex -> fallback());
```

```java
future.handle((result, ex) -> {
    if (ex != null) {
        return fallback();
    }
    return result;
});
```

```java
future.whenComplete((result, ex) -> log(result, ex));
```

### Remember

```text
exceptionally  → recover
handle         → inspect + transform
whenComplete   → observe/side effect
```

---

# 29. Async Execution & Custom Executor ⭐⭐⭐⭐⭐

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

CompletableFuture<String> future =
        CompletableFuture.supplyAsync(
                () -> blockingOperation(),
                executor
        );
```

### Production rule
> Executor choice should reflect workload, isolation and downstream capacity.

Do not blindly put blocking work on a shared common pool.

---

# 30. Thread Pool Sizing ⭐⭐⭐⭐⭐

Never memorize a universal formula such as:

```text
CPU × 2 = always correct
```

Think about:

```text
CPU-bound vs I/O-bound
CPU capacity
I/O wait
queue size
memory
DB connection pool
remote-service limits
latency
throughput
```

### Golden line
> Thread-pool sizing is workload-dependent and should be validated with realistic measurements.

---

# 31. Rejection & Backpressure ⭐⭐⭐⭐⭐

Flow:

```text
workers busy
   ↓
queue fills
   ↓
new task arrives
   ↓
RejectedExecutionHandler
```

Common policies:

```text
AbortPolicy
CallerRunsPolicy
DiscardPolicy
DiscardOldestPolicy
```

### Backpressure
> A bounded queue can prevent unlimited buffering and force the producer/system to slow down or reject work.

---

# 32. Custom `ThreadFactory` ⭐⭐⭐⭐

```java
ThreadFactory factory = task -> {
    Thread thread = new Thread(task);
    thread.setName("payment-worker-" + thread.getId());
    return thread;
};
```

Useful for:

```text
thread dumps
logs
monitoring
production debugging
```

---

# 33. Fork/Join & Work Stealing ⭐⭐⭐⭐

```text
large task
   ↓
split into subtasks
   ↓
fork
   ↓
compute
   ↓
join
```

### Remember
> Fork/Join is designed for tasks that can be recursively decomposed; work stealing lets idle workers take available work from other workers.

---

# 34. Parallel Streams — Risk ⭐⭐⭐⭐

```java
list.parallelStream()
    .map(this::blockingDatabaseCall)
    .toList();
```

Do not assume this is automatically better.

Consider:

```text
common pool
blocking I/O
DB capacity
remote-service limits
task granularity
ordering
```

---

# 35. `wait()` / `notify()` vs Modern Utilities ⭐⭐⭐⭐⭐

Remember the low-level rules:

```java
synchronized (lock) {
    while (!ready) {
        lock.wait();
    }
}
```

And:

```java
synchronized (lock) {
    ready = true;
    lock.notifyAll();
}
```

### Interview line
> For new producer-consumer code, prefer higher-level utilities such as `BlockingQueue` when they fit the problem.

---

# 36. Common Production Concurrency Problems 🔥

### Deadlock

```text
T1: A → B
T2: B → A
```

Fix with consistent lock ordering or suitable timed locking.

### Starvation

```text
A thread repeatedly fails to obtain required resources/CPU/lock access.
```

### Livelock

```text
Threads remain active but repeatedly react to each other without making progress.
```

### Queue overload

```text
arrival rate > service rate
→ queue grows
→ latency grows
→ memory/timeout pressure
```

---

# 37. Production Debugging Checklist ⭐⭐⭐⭐⭐

When concurrency causes latency problems, check:

```text
□ active worker count
□ queue size
□ rejected tasks
□ task execution time
□ CPU usage
□ GC
□ DB connection pool
□ remote dependency latency
□ lock contention
□ thread dumps
□ timeout/cancellation behavior
□ executor shutdown state
```

### Important
> More threads do not automatically mean more throughput.

---

# 38. 15 Must-Know Interview Comparisons 🔥

| Comparison | One-line answer |
|---|---|
| `execute()` vs `submit()` | `submit()` gives a `Future`; `execute()` does not |
| `shutdown()` vs `shutdownNow()` | Graceful stop vs interruption request |
| `Future` vs `CompletableFuture` | Result handle vs composable async pipeline |
| Latch vs Barrier | One-shot countdown vs reusable fixed-party barrier |
| Barrier vs Phaser | Fixed parties vs dynamic parties/phases |
| Semaphore vs Lock | Permits/capacity vs ownership of critical section |
| Atomic vs synchronized | Simple atomic state vs broader critical section/invariant |
| AtomicLong vs LongAdder | Exact atomic state transitions vs high-contention accumulation |
| `thenApply` vs `thenCompose` | Transform vs async chain |
| `thenCompose` vs `thenCombine` | Dependent async step vs independent futures |
| `allOf` vs `anyOf` | Wait for all vs first completion |
| `ArrayBlockingQueue` vs `LinkedBlockingQueue` | Array-backed bounded queue vs linked-node queue |
| `ConcurrentHashMap` vs `HashMap` | Concurrent access support vs non-concurrent map |
| `ReentrantLock` vs `synchronized` | Explicit lock API/features vs intrinsic monitor |
| `wait()` vs `sleep()` | Monitor-based coordination vs timed pause without releasing monitor |

---

# 39. 10 Code Snippets to Write From Memory 🏆

## 1. Executor

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(() -> doWork());
executor.shutdown();
```

## 2. Latch

```java
latch.countDown();
latch.await();
```

## 3. Semaphore

```java
semaphore.acquire();
try {
    use();
} finally {
    semaphore.release();
}
```

## 4. Lock

```java
lock.lock();
try {
    work();
} finally {
    lock.unlock();
}
```

## 5. Timed Lock

```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try { work(); }
    finally { lock.unlock(); }
}
```

## 6. Blocking Queue

```java
queue.put(task);
Task task = queue.take();
```

## 7. Atomic Counter

```java
counter.incrementAndGet();
```

## 8. Concurrent Map

```java
map.merge(key, 1, Integer::sum);
```

## 9. CompletableFuture

```java
CompletableFuture.supplyAsync(() -> fetch())
        .thenApply(this::transform)
        .exceptionally(ex -> fallback());
```

## 10. Graceful Shutdown

```java
executor.shutdown();
try {
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
        executor.shutdownNow();
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

---

# 40. Rapid-Fire Interview Round ⚡

**Q1. Does `shutdownNow()` guarantee task termination?**  
No. It requests interruption; tasks must cooperate.

**Q2. Is `volatile` enough for `count++`?**  
No. `count++` is a compound operation.

**Q3. Is `ConcurrentHashMap` enough for every compound business invariant?**  
No. You may need atomic map methods or external synchronization/locking.

**Q4. Can `LongAdder` replace every `AtomicLong`?**  
No. Their semantics and use cases differ.

**Q5. Is `StampedLock` reentrant?**  
No.

**Q6. Is `CountDownLatch` reusable?**  
No.

**Q7. Is `CyclicBarrier` reusable?**  
Yes, subject to its broken/reset behavior.

**Q8. Can `Semaphore` have multiple permits?**  
Yes.

**Q9. Does `sleep()` release a monitor?**  
No.

**Q10. Does `wait()` release the object's monitor while waiting?**  
Yes.

**Q11. What happens if `wait()` is called without owning the monitor?**  
`IllegalMonitorStateException`.

**Q12. What is `CallerRunsPolicy` useful for?**  
It can provide backpressure by making the submitting thread execute the task when the executor is saturated.

**Q13. Does `allOf()` return all values directly?**  
No. It returns `CompletableFuture<Void>`.

**Q14. Does `anyOf()` mean the fastest successful result only?**  
No. The first supplied future to complete, including exceptional completion, determines completion.

**Q15. Should blocking I/O automatically use `parallelStream()`?**  
No.

---

# 41. 60-Second Chapter 8 Answer 🧠

> **"Java concurrency utilities solve different classes of problems. `ExecutorService` manages asynchronous tasks and thread pools; `CountDownLatch`, `CyclicBarrier`, `Phaser` and `Semaphore` solve coordination and capacity problems; `BlockingQueue` supports producer-consumer workflows; concurrent collections and atomics protect shared state; locks provide explicit synchronization and timeout/read-write/optimistic locking options. `CompletableFuture` provides composable asynchronous workflows, while Fork/Join handles recursive parallel computation. In production I also care about bounded queues, backpressure, thread-pool sizing, interruption, cancellation, monitoring and graceful shutdown."**

---

# 42. Final Memory Map 🏆

```text
TASK EXECUTION
 └── ExecutorService

TASK RESULT
 ├── Callable
 └── Future

SCHEDULING
 └── ScheduledExecutorService

COORDINATION
 ├── CountDownLatch
 ├── CyclicBarrier
 ├── Phaser
 └── Exchanger

CAPACITY
 └── Semaphore

QUEUE / MESSAGING
 ├── BlockingQueue
 ├── ArrayBlockingQueue
 ├── LinkedBlockingQueue
 ├── PriorityBlockingQueue
 ├── DelayQueue
 └── ConcurrentLinkedQueue

SHARED STATE
 ├── Atomic variables
 ├── LongAdder
 ├── ConcurrentHashMap
 └── CopyOnWriteArrayList

LOCKING
 ├── ReentrantLock
 ├── tryLock
 ├── ReentrantReadWriteLock
 ├── StampedLock
 └── Condition

ASYNC
 └── CompletableFuture
     ├── thenApply
     ├── thenCompose
     ├── thenCombine
     ├── allOf
     ├── anyOf
     └── exception handling

PARALLEL COMPUTATION
 └── Fork/Join

PRODUCTION
 ├── Pool sizing
 ├── Backpressure
 ├── Rejection policies
 ├── Interruption
 ├── Cancellation
 ├── Monitoring
 └── Graceful shutdown
```

---

# 43. Practice Code Challenges 🔥

### Challenge 1
Write a 4-thread executor and submit 10 tasks.

### Challenge 2
Implement graceful shutdown from memory.

### Challenge 3
Build producer-consumer with `ArrayBlockingQueue`.

### Challenge 4
Limit API calls to 5 concurrent requests using `Semaphore`.

### Challenge 5
Wait for 5 startup tasks with `CountDownLatch`.

### Challenge 6
Synchronize 4 workers across two phases using `CyclicBarrier`.

### Challenge 7
Use `ConcurrentHashMap.merge()` to build a concurrent word counter.

### Challenge 8
Build an async pipeline using `thenCompose()`.

### Challenge 9
Combine two independent async calls with `thenCombine()`.

### Challenge 10
Implement a bounded `ThreadPoolExecutor` with `CallerRunsPolicy`.

---

# 44. Interview Readiness Checklist

- [ ] I can explain every utility in one sentence.
- [ ] I can choose between Latch / Barrier / Phaser.
- [ ] I can choose Lock vs Semaphore.
- [ ] I can explain Atomic vs `volatile`.
- [ ] I can explain `LongAdder` vs `AtomicLong`.
- [ ] I can explain concurrent collection trade-offs.
- [ ] I can write `CompletableFuture` composition.
- [ ] I can explain `allOf()` / `anyOf()`.
- [ ] I can explain rejection policies.
- [ ] I can explain backpressure.
- [ ] I can discuss thread-pool sizing.
- [ ] I can write graceful shutdown from memory.
- [ ] I can discuss cancellation/interruption.
- [ ] I can identify deadlock/starvation/livelock.
- [ ] I can debug queue/thread-pool/DB-pool saturation.
- [ ] I can answer Chapter 8 in 2 minutes.

---

## Navigation

[← 8.40 — Concurrency Utilities Interview Scenarios](../40-Concurrency-Utilities-Interview-Scenarios/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.42 — Concurrency Utilities Final Assessment**