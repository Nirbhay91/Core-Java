# 8.40 — Concurrency Utilities Interview Scenarios

> **Goal:** Convert Chapter 8 concepts into real interview problem-solving. The focus is not just knowing APIs, but choosing the right concurrency utility, explaining trade-offs, writing safe code, and debugging production scenarios.

## 1. Interview Decision Framework ⭐⭐⭐⭐⭐

When an interviewer gives a concurrency problem, think:

```text
What is the shared resource?
        ↓
Who owns it?
        ↓
Is the operation blocking or non-blocking?
        ↓
Do I need ordering?
        ↓
Do I need bounded concurrency?
        ↓
Do I need cancellation / timeout?
        ↓
Do I need task composition?
        ↓
Choose the smallest suitable primitive
```

---

# 2. Scenario — Limit Concurrent Requests with `Semaphore` ⭐⭐⭐⭐⭐

**Question:** Only 3 requests may access a downstream service at once. What would you use?

### Answer
Use `Semaphore`.

```java
import java.util.concurrent.Semaphore;

public class SemaphoreScenario {

    private static final Semaphore LIMITER = new Semaphore(3);

    static void callService(int requestId) {
        boolean acquired = false;

        try {
            LIMITER.acquire();
            acquired = true;

            System.out.println("Request " + requestId + " entered");
            Thread.sleep(1_000);
            System.out.println("Request " + requestId + " completed");

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (acquired) {
                LIMITER.release();
            }
        }
    }
}
```

**Interview line:**

> `Semaphore` limits concurrent access; it is useful when the resource has a fixed capacity.

---

# 3. Scenario — Wait for N Services to Start ⭐⭐⭐⭐⭐

**Question:** Application should continue only after 5 startup tasks finish.

Use `CountDownLatch`.

```java
import java.util.concurrent.CountDownLatch;

public class StartupScenario {

    public static void main(String[] args) throws Exception {

        CountDownLatch latch = new CountDownLatch(5);

        for (int i = 1; i <= 5; i++) {
            int service = i;

            new Thread(() -> {
                try {
                    Thread.sleep(service * 200L);
                    System.out.println("Service " + service + " started");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();
                }
            }).start();
        }

        latch.await();
        System.out.println("All services started");
    }
}
```

**Key:** `CountDownLatch` is one-shot.

---

# 4. Scenario — Repeated Phase Synchronization ⭐⭐⭐⭐⭐

**Question:** 4 workers must finish phase 1 before phase 2, and this happens repeatedly.

Use `CyclicBarrier`.

```java
import java.util.concurrent.CyclicBarrier;

public class BarrierScenario {

    public static void main(String[] args) {

        CyclicBarrier barrier = new CyclicBarrier(
                4,
                () -> System.out.println("Phase completed")
        );

        for (int i = 1; i <= 4; i++) {
            int worker = i;

            new Thread(() -> {
                try {
                    System.out.println("Worker " + worker + " phase 1");
                    barrier.await();

                    System.out.println("Worker " + worker + " phase 2");
                    barrier.await();

                } catch (Exception e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }
}
```

**Key:** Barrier is reusable; latch is not.

---

# 5. Scenario — Dynamic Number of Participants ⭐⭐⭐⭐

**Question:** Number of participating workers changes between phases.

Use `Phaser`.

```java
import java.util.concurrent.Phaser;

public class PhaserScenario {

    public static void main(String[] args) {

        Phaser phaser = new Phaser(1);

        for (int i = 1; i <= 3; i++) {
            phaser.register();
            int worker = i;

            new Thread(() -> {
                System.out.println("Worker " + worker + " phase 0");
                phaser.arriveAndAwaitAdvance();

                System.out.println("Worker " + worker + " phase 1");
                phaser.arriveAndDeregister();
            }).start();
        }

        phaser.arriveAndDeregister();
    }
}
```

---

# 6. Scenario — Producer/Consumer ⭐⭐⭐⭐⭐

**Question:** Build a safe producer-consumer system.

Use `BlockingQueue`.

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ProducerConsumerScenario {

    public static void main(String[] args) {

        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    queue.put(i);
                    System.out.println("Produced: " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    Integer value = queue.take();
                    System.out.println("Consumed: " + value);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

**Interview line:**

> `BlockingQueue` gives blocking coordination without manually implementing `wait()`/`notify()`.

---

# 7. Scenario — Read-Heavy Cache ⭐⭐⭐⭐⭐

**Question:** Many threads read a cache while writes are relatively rare.

Use `ReentrantReadWriteLock` when a read/write lock is appropriate.

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteCache {

    private final Map<String, String> cache = new HashMap<>();
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public String get(String key) {
        lock.readLock().lock();
        try {
            return cache.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }

    public void put(String key, String value) {
        lock.writeLock().lock();
        try {
            cache.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

---

# 8. Scenario — Avoid Waiting Forever for a Lock ⭐⭐⭐⭐⭐

**Question:** A thread should not wait indefinitely for a lock.

Use `tryLock()` with timeout.

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class TimedLockScenario {

    private final ReentrantLock lock = new ReentrantLock();

    public void process() {
        boolean acquired = false;

        try {
            acquired = lock.tryLock(500, TimeUnit.MILLISECONDS);

            if (!acquired) {
                System.out.println("Could not acquire lock");
                return;
            }

            System.out.println("Processing safely");

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (acquired) {
                lock.unlock();
            }
        }
    }
}
```

---

# 9. Scenario — Optimistic Read ⭐⭐⭐⭐⭐

**Question:** Reads are frequent and writes are occasional; minimize read locking where safe.

`StampedLock` can provide optimistic reads.

```java
import java.util.concurrent.locks.StampedLock;

public class StampedLockScenario {

    private double x;
    private double y;
    private final StampedLock lock = new StampedLock();

    public double distance() {
        long stamp = lock.tryOptimisticRead();

        double currentX = x;
        double currentY = y;

        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }

        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

**Important:** `StampedLock` is not reentrant.

---

# 10. Scenario — Atomic Counter ⭐⭐⭐⭐⭐

**Question:** Multiple threads increment a counter without using `synchronized`.

Use `AtomicInteger` for a simple atomic counter.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounterScenario {

    private final AtomicInteger counter = new AtomicInteger();

    public void increment() {
        counter.incrementAndGet();
    }

    public int get() {
        return counter.get();
    }
}
```

**Interview line:**

> Atomics use lock-free/CAS-based operations for supported atomic updates; they are not a universal replacement for locks.

---

# 11. Scenario — Very High-Frequency Counter ⭐⭐⭐⭐⭐

**Question:** Many threads update a statistics counter under high contention.

Consider `LongAdder`.

```java
import java.util.concurrent.atomic.LongAdder;

public class MetricsCounter {

    private final LongAdder requests = new LongAdder();

    public void recordRequest() {
        requests.increment();
    }

    public long totalRequests() {
        return requests.sum();
    }
}
```

**Trade-off:** `LongAdder` is excellent for high-contention statistics but does not provide the same atomic read-modify-write semantics as `AtomicLong` for every operation.

---

# 12. Scenario — Concurrent Cache ⭐⭐⭐⭐⭐

**Question:** Multiple threads access a shared map safely.

Use `ConcurrentHashMap`.

```java
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentCache {

    private final ConcurrentHashMap<String, Integer> cache =
            new ConcurrentHashMap<>();

    public void increment(String key) {
        cache.merge(key, 1, Integer::sum);
    }

    public Integer get(String key) {
        return cache.get(key);
    }
}
```

### Interview trap

Do not do this for a compound update:

```java
if (!cache.containsKey(key)) {
    cache.put(key, 1);
}
```

Prefer an atomic map operation such as `putIfAbsent`, `compute`, or `merge` when appropriate.

---

# 13. Scenario — Snapshot-Heavy List ⭐⭐⭐⭐

**Question:** Reads/iteration are extremely frequent and modifications are rare.

Consider `CopyOnWriteArrayList`.

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class ListenerRegistry {

    private final CopyOnWriteArrayList<String> listeners =
            new CopyOnWriteArrayList<>();

    public void register(String listener) {
        listeners.add(listener);
    }

    public void notifyListeners() {
        for (String listener : listeners) {
            System.out.println("Notify: " + listener);
        }
    }
}
```

**Trade-off:** Writes copy the underlying array, so it is unsuitable for write-heavy lists.

---

# 14. Scenario — Async Composition ⭐⭐⭐⭐⭐

**Question:** Call service A, then use its result to call service B.

Use `thenCompose()` when B depends on A.

```java
import java.util.concurrent.CompletableFuture;

public class AsyncCompositionScenario {

    static CompletableFuture<String> getUser() {
        return CompletableFuture.completedFuture("Nirbhay");
    }

    static CompletableFuture<String> getOrders(String user) {
        return CompletableFuture.completedFuture(
                "Orders for " + user
        );
    }

    public static void main(String[] args) {
        CompletableFuture<String> result =
                getUser().thenCompose(AsyncCompositionScenario::getOrders);

        System.out.println(result.join());
    }
}
```

**Remember:**

```text
thenApply   → transform value
thenCompose → chain async operation
thenCombine → combine independent futures
```

---

# 15. Scenario — Parallel Independent Calls ⭐⭐⭐⭐⭐

**Question:** Fetch profile and recommendations independently, then combine them.

```java
import java.util.concurrent.CompletableFuture;

public class CombineScenario {

    public static void main(String[] args) {

        CompletableFuture<String> profile =
                CompletableFuture.supplyAsync(() -> "Profile");

        CompletableFuture<String> recommendations =
                CompletableFuture.supplyAsync(() -> "Recommendations");

        CompletableFuture<String> result = profile.thenCombine(
                recommendations,
                (p, r) -> p + " + " + r
        );

        System.out.println(result.join());
    }
}
```

---

# 16. Scenario — Wait for Multiple Futures ⭐⭐⭐⭐⭐

```java
CompletableFuture<String> a = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> b = CompletableFuture.supplyAsync(() -> "B");
CompletableFuture<String> c = CompletableFuture.supplyAsync(() -> "C");

CompletableFuture<Void> all =
        CompletableFuture.allOf(a, b, c);

all.join();

System.out.println(a.join());
System.out.println(b.join());
System.out.println(c.join());
```

**Interview trap:** `allOf()` returns `CompletableFuture<Void>`, not a collection of results.

---

# 17. Scenario — First Completion Wins ⭐⭐⭐⭐⭐

```java
CompletableFuture<String> serverA =
        CompletableFuture.supplyAsync(() -> "A");

CompletableFuture<String> serverB =
        CompletableFuture.supplyAsync(() -> "B");

CompletableFuture<Object> fastest =
        CompletableFuture.anyOf(serverA, serverB);

System.out.println(fastest.join());
```

**Key:** `anyOf()` completes when the first supplied future completes, whether normally or exceptionally.

---

# 18. Scenario — Handle Async Exceptions ⭐⭐⭐⭐⭐

```java
CompletableFuture<Integer> result =
        CompletableFuture.supplyAsync(() -> 10 / 0)
                .exceptionally(ex -> {
                    System.out.println("Error: " + ex.getMessage());
                    return 0;
                });

System.out.println(result.join());
```

Know the difference between:

```text
exceptionally → recover
handle        → inspect both success/failure and return value
whenComplete  → observe side effect; does not transform normally
```

---

# 19. Scenario — Custom Executor for Blocking Work ⭐⭐⭐⭐⭐

**Question:** Do not let blocking tasks consume a shared/common pool.

```java
import java.util.concurrent.*;

public class CustomExecutorScenario {

    public static void main(String[] args) {

        ExecutorService ioPool = Executors.newFixedThreadPool(10);

        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    try {
                        Thread.sleep(1_000);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        throw new RuntimeException(e);
                    }
                    return "IO result";
                }, ioPool);

        System.out.println(future.join());
        ioPool.shutdown();
    }
}
```

**Interview line:**

> Choose executor configuration based on workload and isolation requirements; avoid blindly putting blocking work on an unrelated shared executor.

---

# 20. Scenario — Thread Pool Saturation ⭐⭐⭐⭐⭐

**Question:** What happens when all workers are busy and the queue is full?

Answer:

```text
worker threads busy
      ↓
queue fills
      ↓
new task arrives
      ↓
RejectedExecutionHandler
```

Example:

```java
import java.util.concurrent.*;

public class RejectionScenario {

    public static void main(String[] args) {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,
                2,
                0,
                TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(2),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        for (int i = 1; i <= 10; i++) {
            int task = i;
            executor.execute(() -> {
                System.out.println("Task " + task + " on " +
                        Thread.currentThread().getName());
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        executor.shutdown();
    }
}
```

**Why `CallerRunsPolicy`?** It can apply backpressure by making the submitting thread execute the task when the pool is saturated.

---

# 21. Scenario — Deadlock in Production ⭐⭐⭐⭐⭐

**Question:** Two services/threads need resources A and B and can acquire them in opposite order.

Bad pattern:

```text
Thread 1: A → B
Thread 2: B → A
```

Potential result:

```text
T1 owns A, waits for B
T2 owns B, waits for A
```

### Fix

Always acquire locks in a consistent global order.

```java
synchronized (lockA) {
    synchronized (lockB) {
        // work
    }
}
```

or use timed `tryLock()` where appropriate.

---

# 22. Scenario — `wait()` / `notify()` vs `BlockingQueue` ⭐⭐⭐⭐⭐

**Question:** Which would you choose for producer-consumer code?

Preferred in most application code:

```java
BlockingQueue<Task> queue;
```

rather than manually coordinating:

```java
wait();
notify();
```

Why?

```text
less error-prone
built-in blocking semantics
clearer ownership
capacity can be bounded
```

Know `wait/notify` for legacy/interview understanding, but prefer higher-level utilities when suitable.

---

# 23. Scenario — `AtomicInteger` vs `synchronized` ⭐⭐⭐⭐⭐

### Use AtomicInteger when

```text
simple independent atomic state
CAS-based update is enough
```

### Use synchronized/Lock when

```text
multiple variables
invariants across operations
compound critical section
complex coordination
```

**Interview line:**

> The choice is driven by the state invariant and operation complexity, not simply by which mechanism sounds faster.

---

# 24. Scenario — `LongAdder` vs `AtomicLong` ⭐⭐⭐⭐⭐

| Requirement | Prefer |
|---|---|
| Simple atomic counter semantics | `AtomicLong` |
| Very high contention statistics | `LongAdder` |
| Need exact atomic read-modify-write logic | `AtomicLong` often fits better |
| Metrics where occasional `sum()` is acceptable | `LongAdder` |

---

# 25. Scenario — `ConcurrentHashMap` Atomic Update ⭐⭐⭐⭐⭐

Bad:

```java
if (map.get(key) == null) {
    map.put(key, value);
}
```

Better:

```java
map.putIfAbsent(key, value);
```

For counters:

```java
map.merge(key, 1, Integer::sum);
```

For more complex initialization/update:

```java
map.computeIfAbsent(key, k -> createValue(k));
```

---

# 26. Scenario — Lock Acquisition Timeout ⭐⭐⭐⭐⭐

**Question:** A service must fail fast instead of waiting indefinitely for a lock.

```java
if (lock.tryLock(200, TimeUnit.MILLISECONDS)) {
    try {
        process();
    } finally {
        lock.unlock();
    }
} else {
    throw new IllegalStateException("Could not acquire lock");
}
```

Know why `finally` is mandatory for lock release.

---

# 27. Scenario — Thread Pool Sizing ⭐⭐⭐⭐⭐

**Question:** How many threads should you create?

Do not answer:

```text
CPU count × 2 always
```

Better:

```text
CPU-bound:
close to available CPU capacity, then benchmark

I/O-bound:
can need more concurrency because workers spend time waiting

But consider:
CPU
I/O latency
queueing
memory
DB connection pool
remote service capacity
timeouts
throughput
```

**Golden line:**

> Thread-pool sizing is workload- and system-dependent; measure under realistic load.

---

# 28. Scenario — Graceful Shutdown ⭐⭐⭐⭐⭐

```java
public static void shutdownGracefully(
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

This is a **must-write-from-memory** interview snippet.

---

# 29. Scenario — `ForkJoinPool` ⭐⭐⭐⭐⭐

**Question:** A large CPU-oriented recursive problem needs parallel decomposition.

Consider Fork/Join.

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

public class ForkJoinScenario {

    static class SumTask extends RecursiveTask<Long> {
        private final int[] array;
        private final int start;
        private final int end;
        private static final int THRESHOLD = 1_000;

        SumTask(int[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }

        @Override
        protected Long compute() {
            if (end - start <= THRESHOLD) {
                long sum = 0;
                for (int i = start; i < end; i++) {
                    sum += array[i];
                }
                return sum;
            }

            int mid = (start + end) / 2;
            SumTask left = new SumTask(array, start, mid);
            SumTask right = new SumTask(array, mid, end);

            left.fork();
            long rightResult = right.compute();
            long leftResult = left.join();

            return leftResult + rightResult;
        }
    }

    public static void main(String[] args) {
        int[] data = new int[10_000];

        ForkJoinPool pool = new ForkJoinPool();
        long result = pool.invoke(new SumTask(data, 0, data.length));

        System.out.println(result);
        pool.shutdown();
    }
}
```

**Key:** Work stealing helps idle workers obtain tasks from other workers' deques.

---

# 30. Scenario — Parallel Stream Risk ⭐⭐⭐⭐

Bad assumption:

```java
list.parallelStream()
    .map(this::blockingDatabaseCall)
    .toList();
```

Parallelism is not automatically beneficial for blocking external I/O.

Think about:

```text
common pool
DB capacity
connection pool
remote API limits
ordering
task granularity
```

---

# 31. Scenario — `CompletableFuture` Timeout ⭐⭐⭐⭐⭐

For modern Java versions, use appropriate timeout APIs where supported.

```java
CompletableFuture<String> result =
        CompletableFuture.supplyAsync(() -> callService())
                .orTimeout(2, TimeUnit.SECONDS);
```

Fallback:

```java
CompletableFuture<String> result =
        CompletableFuture.supplyAsync(() -> callService())
                .completeOnTimeout("fallback", 2, TimeUnit.SECONDS);
```

Know that a timeout on a `CompletableFuture` does not automatically mean the underlying external operation has been physically stopped.

---

# 32. Scenario — `CompletableFuture` Cancellation ⭐⭐⭐⭐

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> longOperation());

future.cancel(true);
```

Interview nuance:

> Cancellation changes the future's completion state, but you should not assume that every underlying operation is immediately interrupted or rolled back.

---

# 33. Scenario — Backpressure ⭐⭐⭐⭐⭐

**Question:** Producer is faster than consumers. What do you do?

Options:

```text
bounded queue
blocking put
CallerRunsPolicy
rate limiting
load shedding
batching
scaling consumers
```

The key principle:

> **Do not allow unlimited buffering to hide an overload problem.**

---

# 34. Scenario — Common Pool Problem ⭐⭐⭐⭐⭐

**Question:** Why can blocking work on the common pool be dangerous?

Because unrelated tasks may share the same pool and become delayed by blocking operations.

Better:

```java
ExecutorService ioExecutor =
        Executors.newFixedThreadPool(20);

CompletableFuture.supplyAsync(
        () -> blockingOperation(),
        ioExecutor
);
```

Size the pool based on the actual workload and downstream limits.

---

# 35. Scenario — Lock vs Semaphore ⭐⭐⭐⭐⭐

### Lock

Protects a critical section / shared state.

```text
one owner at a time
```

### Semaphore

Controls permits/capacity.

```text
N concurrent users allowed
```

Example:

```text
Lock      → protect account balance
Semaphore → allow 20 DB-like resources
```

---

# 36. Scenario — Latch vs Barrier vs Phaser ⭐⭐⭐⭐⭐

| Utility | Main idea |
|---|---|
| `CountDownLatch` | Wait until count reaches zero |
| `CyclicBarrier` | Fixed parties meet at a barrier repeatedly |
| `Phaser` | Dynamic parties + multiple phases |

This comparison is a frequent interview question.

---

# 37. Scenario — `ArrayBlockingQueue` vs `LinkedBlockingQueue` ⭐⭐⭐⭐⭐

### `ArrayBlockingQueue`

```text
bounded
array-backed
fixed capacity
```

### `LinkedBlockingQueue`

```text
linked-node based
optionally bounded
potentially different memory characteristics
```

Do not choose solely by class name; consider capacity and workload.

---

# 38. Scenario — `PriorityBlockingQueue` ⭐⭐⭐⭐

Use when consumers should receive elements according to priority.

Important:

> `PriorityBlockingQueue` is not a normal FIFO queue and is not bounded by a fixed capacity in the same way as `ArrayBlockingQueue`.

---

# 39. Scenario — `DelayQueue` ⭐⭐⭐⭐

Elements become available only after their delay expires.

Typical examples:

```text
retry scheduling
expiration processing
delayed jobs
```

---

# 40. Scenario — `ConcurrentLinkedQueue` ⭐⭐⭐⭐

Use for non-blocking concurrent FIFO-style queueing when blocking/backpressure semantics are not required.

```java
import java.util.concurrent.ConcurrentLinkedQueue;

ConcurrentLinkedQueue<String> queue =
        new ConcurrentLinkedQueue<>();

queue.offer("A");
queue.offer("B");

System.out.println(queue.poll());
```

**Important:** It is unbounded in nature and does not provide blocking `take()`/`put()` semantics.

---

# 41. Scenario — `Exchanger` ⭐⭐⭐

Two threads need to exchange objects directly.

```java
import java.util.concurrent.Exchanger;

public class ExchangerScenario {

    public static void main(String[] args) {

        Exchanger<String> exchanger = new Exchanger<>();

        Thread producer = new Thread(() -> {
            try {
                String received = exchanger.exchange("Data from producer");
                System.out.println("Producer received: " + received);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                String received = exchanger.exchange("Data from consumer");
                System.out.println("Consumer received: " + received);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

---

# 42. Scenario — `Condition` vs `wait/notify` ⭐⭐⭐⭐⭐

`Condition` provides explicit wait sets associated with a `Lock`.

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ConditionScenario {

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition available = lock.newCondition();
    private boolean ready;

    public void awaitReady() throws InterruptedException {
        lock.lock();
        try {
            while (!ready) {
                available.await();
            }
        } finally {
            lock.unlock();
        }
    }

    public void makeReady() {
        lock.lock();
        try {
            ready = true;
            available.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
```

**Important:** Use a `while` condition check, not a one-time `if`, around waits.

---

# 43. Scenario — `wait()` Rule ⭐⭐⭐⭐⭐

A thread calling `wait()` must own the object's monitor.

```java
synchronized (lock) {
    while (!ready) {
        lock.wait();
    }
}
```

Likewise:

```java
synchronized (lock) {
    lock.notifyAll();
}
```

Calling these outside the required monitor ownership results in `IllegalMonitorStateException`.

---

# 44. Scenario — `sleep()` vs `wait()` ⭐⭐⭐⭐⭐

| `sleep()` | `wait()` |
|---|---|
| `Thread` method | `Object` method |
| Does not release monitor | Releases object's monitor while waiting |
| Time-based delay | Communication/condition waiting |
| Does not require synchronized ownership | Must own monitor |

---

# 45. Scenario — `volatile` vs Atomic ⭐⭐⭐⭐⭐

```java
volatile boolean running;
```

Good for visibility of a simple state flag.

But:

```java
volatile int count;
count++; // NOT atomic
```

For atomic increment:

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```

---

# 46. Scenario — Why `volatile` Does Not Solve Compound Actions

```java
if (count < 100) {
    count++;
}
```

Multiple operations are involved:

```text
read
check
increment
write
```

Use a suitable synchronization/atomic mechanism when the whole operation must be atomic.

---

# 47. Scenario — `execute()` vs `submit()` ⭐⭐⭐⭐

```java
executor.execute(task);
```

returns no `Future`.

```java
Future<?> future = executor.submit(task);
```

returns a `Future` and supports result/cancellation/error observation through that future.

Interview trap:

> Exceptions from tasks submitted through `submit()` are captured by the `Future`; they are not simply handled the same way as exceptions escaping from `execute()`.

---

# 48. Scenario — Thread Factory ⭐⭐⭐⭐

**Question:** How do you give worker threads meaningful names?

```java
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

public class NamedThreadFactory implements ThreadFactory {

    private final AtomicInteger counter = new AtomicInteger(1);

    @Override
    public Thread newThread(Runnable task) {
        Thread thread = new Thread(task);
        thread.setName("payment-worker-" + counter.getAndIncrement());
        return thread;
    }
}
```

Useful for:

```text
logs
thread dumps
metrics
production debugging
```

---

# 49. Scenario — Thread Safety Strategy ⭐⭐⭐⭐⭐

When making a class thread-safe, consider:

```text
1. Immutable state
2. Thread confinement
3. Concurrent collections
4. Atomic variables
5. synchronized
6. Lock
7. Read/write locks
8. Message passing / queues
```

Prefer the simplest strategy that correctly protects the invariant.

---

# 50. Production Debugging Scenario ⭐⭐⭐⭐⭐

**Symptom:** API latency suddenly increases.

Check:

```text
Thread pool active count
Queue size
Rejected tasks
Task execution time
CPU
GC
DB connection pool
Remote dependency latency
Lock contention
Thread dumps
```

Do not immediately increase thread count.

More threads can increase:

```text
context switching
memory usage
queue pressure
DB contention
remote service overload
```

---

# 51. Production Debugging — Queue Growing ⭐⭐⭐⭐⭐

If:

```text
queue size ↑ continuously
```

likely:

```text
arrival rate > service rate
```

Possible responses:

```text
increase consumer capacity if downstream allows
reduce work
batch
shed load
apply backpressure
fix slow dependency
increase downstream capacity
```

---

# 52. Production Debugging — CPU 100% ⭐⭐⭐⭐⭐

Do not automatically add threads.

If workload is CPU-bound:

```text
more threads
   ↓
more context switching
   ↓
possibly worse throughput
```

First inspect CPU saturation and actual parallelism.

---

# 53. Production Debugging — DB Pool Exhaustion ⭐⭐⭐⭐⭐

Symptoms:

```text
threads waiting for DB connection
queue grows
latency increases
```

Increasing application threads may make the problem worse.

Think:

```text
executor concurrency
DB pool size
query latency
DB capacity
```

These capacities must be considered together.

---

# 54. Scenario — Safe Lock Ordering ⭐⭐⭐⭐⭐

Use a consistent order:

```java
void transfer(Account a, Account b) {
    Account first = a.getId() < b.getId() ? a : b;
    Account second = a.getId() < b.getId() ? b : a;

    synchronized (first) {
        synchronized (second) {
            // transfer
        }
    }
}
```

The goal is to avoid circular wait caused by inconsistent lock ordering.

---

# 55. Scenario — Avoid Holding Locks During Slow I/O ⭐⭐⭐⭐⭐

Bad:

```java
synchronized (this) {
    callRemoteService();
}
```

Potential problem:

```text
slow network
   ↓
lock held
   ↓
other threads blocked
```

Prefer minimizing critical sections when correctness allows.

---

# 56. Scenario — Complete Interview Design ⭐⭐⭐⭐⭐

**Question:** Design a bounded async task processor.

Requirements:

```text
• 4 workers
• max 100 queued tasks
• reject/backpressure when full
• named threads
• graceful shutdown
```

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class BoundedTaskProcessor {

    private final ThreadPoolExecutor executor;

    public BoundedTaskProcessor() {
        this.executor = new ThreadPoolExecutor(
                4,
                4,
                0,
                TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(100),
                new NamedFactory("processor-worker-"),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }

    public void submit(Runnable task) {
        executor.execute(task);
    }

    public void shutdown() {
        executor.shutdown();

        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }

    static class NamedFactory implements ThreadFactory {
        private final String prefix;
        private final AtomicInteger count = new AtomicInteger(1);

        NamedFactory(String prefix) {
            this.prefix = prefix;
        }

        @Override
        public Thread newThread(Runnable task) {
            Thread thread = new Thread(task);
            thread.setName(prefix + count.getAndIncrement());
            return thread;
        }
    }
}
```

This single example combines:

```text
ThreadPoolExecutor
Bounded queue
Rejection policy
ThreadFactory
Graceful shutdown
Interrupt handling
```

---

# 57. Scenario — Which Utility Would You Pick? 🧠

| Problem | First candidate |
|---|---|
| Limit concurrent access | `Semaphore` |
| Wait for one-time startup events | `CountDownLatch` |
| Repeated fixed-party synchronization | `CyclicBarrier` |
| Dynamic multi-phase synchronization | `Phaser` |
| Producer-consumer | `BlockingQueue` |
| Simple atomic counter | `AtomicInteger` / `AtomicLong` |
| High-contention metrics | `LongAdder` |
| Concurrent map | `ConcurrentHashMap` |
| Read-heavy list with rare writes | `CopyOnWriteArrayList` |
| Read/write locking | `ReentrantReadWriteLock` |
| Optimistic reads | `StampedLock` |
| Lock timeout | `tryLock()` |
| Async composition | `CompletableFuture` |
| Recursive parallel computation | Fork/Join |
| Executor lifecycle | `ExecutorService` |

---

# 58. Top 20 Interview Questions 🔥

1. Difference between `shutdown()` and `shutdownNow()`?
2. Why is `awaitTermination()` needed?
3. Is `shutdownNow()` guaranteed to stop tasks?
4. Why restore interrupt status?
5. `execute()` vs `submit()`?
6. `Future` vs `CompletableFuture`?
7. `thenApply` vs `thenCompose`?
8. `thenCompose` vs `thenCombine`?
9. `allOf()` vs `anyOf()`?
10. `AtomicInteger` vs `synchronized`?
11. `AtomicLong` vs `LongAdder`?
12. `CountDownLatch` vs `CyclicBarrier`?
13. `CyclicBarrier` vs `Phaser`?
14. `Semaphore` vs `Lock`?
15. `ConcurrentHashMap` vs `HashMap`?
16. `CopyOnWriteArrayList` trade-offs?
17. `ArrayBlockingQueue` vs `LinkedBlockingQueue`?
18. Why use bounded queues?
19. How do you size a thread pool?
20. How do you design graceful shutdown?

---

# 59. 2-Minute Interview Answer 🏆

> **"I choose Java concurrency utilities based on the problem rather than using one mechanism everywhere. For bounded concurrency I use `Semaphore`; for one-time coordination `CountDownLatch`; for repeated fixed-party phases `CyclicBarrier`; for dynamic multi-phase coordination `Phaser`; and for producer-consumer workflows `BlockingQueue`. For shared state I consider immutability, concurrent collections, atomics or locks depending on the invariant. For asynchronous workflows I use `CompletableFuture` with composition and explicit executors when workload isolation is needed. For production executors I prefer bounded queues where appropriate, meaningful rejection/backpressure behavior, monitoring and graceful shutdown. I also make cancellation and interruption cooperative and avoid assuming that more threads always mean more throughput."**

---

# 60. 30-Second Hinglish Answer

> **"Concurrency utility problem ke according choose karni chahiye. `Semaphore` concurrency limit karta hai, `CountDownLatch` one-time wait ke liye, `CyclicBarrier` repeated fixed parties ke liye, `Phaser` dynamic phases ke liye aur `BlockingQueue` producer-consumer ke liye useful hai. Shared state ke liye atomics, concurrent collections ya locks invariant ke according choose karunga. Async workflow ke liye `CompletableFuture`, aur production mein bounded queue, backpressure, monitoring aur graceful shutdown important hain."**

---

# 61. Practice Challenges 🔥

### Challenge 1 — Rate-limited service
Allow only 5 concurrent calls using `Semaphore`.

### Challenge 2 — Startup coordinator
Wait for 6 services using `CountDownLatch`.

### Challenge 3 — Multi-phase pipeline
Implement 3 phases with `CyclicBarrier`.

### Challenge 4 — Dynamic workers
Implement dynamic registration with `Phaser`.

### Challenge 5 — Bounded producer-consumer
Use `ArrayBlockingQueue` with multiple producers and consumers.

### Challenge 6 — Async aggregator
Fetch 3 independent values and combine them with `CompletableFuture.allOf()`.

### Challenge 7 — Timeout
Use `orTimeout()` and provide a fallback.

### Challenge 8 — Lock timeout
Use `tryLock()` to prevent indefinite waiting.

### Challenge 9 — Concurrent cache
Implement atomic updates with `ConcurrentHashMap.merge()`.

### Challenge 10 — Production executor
Build a `ThreadPoolExecutor` with:

```text
4 workers
bounded queue
CallerRunsPolicy
custom ThreadFactory
graceful shutdown
```

### Challenge 11 — Debug deadlock
Create a deadlock and fix it using consistent lock ordering.

### Challenge 12 — Metrics
Compare `AtomicLong` and `LongAdder` under concurrent updates.

---

# 62. Final Revision Map 🧠

```text
Coordination
 ├── CountDownLatch
 ├── CyclicBarrier
 ├── Phaser
 └── Exchanger

Capacity
 ├── Semaphore
 ├── BlockingQueue
 └── ThreadPoolExecutor

Shared State
 ├── Atomic Variables
 ├── ConcurrentHashMap
 ├── CopyOnWriteArrayList
 └── Locks

Async
 ├── Future
 ├── CompletableFuture
 ├── allOf / anyOf
 └── Custom Executors

Parallel Computation
 ├── Fork/Join
 ├── RecursiveTask
 └── Work Stealing

Lifecycle
 ├── shutdown
 ├── awaitTermination
 ├── shutdownNow
 └── Graceful Shutdown
```

---

# 63. Final Interview Checklist

- [ ] Choose utility based on problem
- [ ] `CountDownLatch`
- [ ] `CyclicBarrier`
- [ ] `Phaser`
- [ ] `Semaphore`
- [ ] `BlockingQueue`
- [ ] `ConcurrentHashMap`
- [ ] `CopyOnWriteArrayList`
- [ ] Atomics / CAS
- [ ] `LongAdder`
- [ ] `ReentrantLock`
- [ ] `tryLock()`
- [ ] `ReentrantReadWriteLock`
- [ ] `StampedLock`
- [ ] `Condition`
- [ ] `CompletableFuture`
- [ ] `thenApply` / `thenCompose` / `thenCombine`
- [ ] `allOf()` / `anyOf()`
- [ ] Custom executors
- [ ] Thread-pool sizing
- [ ] Backpressure
- [ ] Rejection policies
- [ ] Fork/Join
- [ ] Parallel stream risks
- [ ] Graceful shutdown
- [ ] Interruption/cancellation
- [ ] Deadlock prevention
- [ ] Production debugging
- [ ] 2-minute answer
- [ ] Rapid-fire questions
- [ ] Practice challenges

---

## Navigation

[← 8.39 — Graceful Shutdown & Production Patterns](../39-Graceful-Shutdown-and-Production-Patterns/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.41 — Concurrency Utilities Quick Revision**