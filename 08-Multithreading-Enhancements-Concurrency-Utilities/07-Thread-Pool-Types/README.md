# 8.7 — Thread Pool Types

> **Goal:** Understand the different executor/thread-pool factories in Java, when to use each, their internal behavior, risks, and production/interview trade-offs.

---

## 1. Why Thread Pools? ⭐⭐⭐⭐⭐

Creating a new `Thread` for every task is expensive and difficult to control.

A thread pool:

```text
Tasks
  ↓
Queue
  ↓
Worker Threads
  ↓
Execute tasks
  ↓
Reuse threads
```

### Main benefits

- Reuse worker threads
- Reduce thread-creation overhead
- Control concurrency
- Queue pending tasks
- Manage lifecycle centrally
- Improve application stability

---

# 2. Common Java Thread-Pool Types ⭐⭐⭐⭐⭐

Java provides convenient factory methods through `Executors`:

| Type | Factory | Typical Use |
|---|---|---|
| Fixed | `newFixedThreadPool(n)` | Controlled number of concurrent tasks |
| Cached | `newCachedThreadPool()` | Short-lived asynchronous tasks with varying load |
| Single | `newSingleThreadExecutor()` | Sequential background processing |
| Scheduled | `newScheduledThreadPool(n)` | Delayed/periodic tasks |
| Work-stealing | `newWorkStealingPool()` | Parallel/work-stealing workloads |
| Single Scheduled | `newSingleThreadScheduledExecutor()` | Sequential scheduled tasks |

> **Interview note:** These are convenience factory methods. For production systems with strict resource limits, `ThreadPoolExecutor` often gives more explicit control over queue capacity, thread counts and rejection policy.

---

# 3. Fixed Thread Pool ⭐⭐⭐⭐⭐

```java
ExecutorService executor = Executors.newFixedThreadPool(3);
```

A fixed pool maintains up to the specified number of worker threads.

```text
             ┌── Worker 1
Tasks ──Queue─┼── Worker 2
             └── Worker 3
```

### Practice Code

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class FixedThreadPoolExample {
    public static void main(String[] args) {

        ExecutorService executor = Executors.newFixedThreadPool(3);

        for (int i = 1; i <= 6; i++) {
            int taskId = i;

            executor.execute(() -> {
                System.out.println(
                        "Task " + taskId + " executed by "
                                + Thread.currentThread().getName());
            });
        }

        executor.shutdown();
    }
}
```

### What happens?

With 6 tasks and 3 workers:

```text
Worker-1 → Task 1 → Task 4 ...
Worker-2 → Task 2 → Task 5 ...
Worker-3 → Task 3 → Task 6 ...
```

Exact task/thread ordering is not guaranteed.

### Best fit

- Controlled concurrency
- CPU-bound work with carefully chosen pool size
- Bounded application worker concurrency when paired with an appropriately bounded executor configuration

---

# 4. Cached Thread Pool ⭐⭐⭐⭐

```java
ExecutorService executor = Executors.newCachedThreadPool();
```

Conceptually:

```text
Incoming task
     ↓
Reuse idle worker if available
     ↓
Otherwise create worker
     ↓
Worker becomes idle
     ↓
May be reused
```

### Practice Code

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CachedThreadPoolExample {
    public static void main(String[] args) {

        ExecutorService executor = Executors.newCachedThreadPool();

        for (int i = 1; i <= 10; i++) {
            int taskId = i;

            executor.execute(() -> {
                System.out.println(
                        "Task " + taskId + " -> "
                                + Thread.currentThread().getName());
            });
        }

        executor.shutdown();
    }
}
```

### Important risk ⚠️

A cached pool can create many threads under sustained load because its implementation uses a `SynchronousQueue` and permits a very large maximum thread count.

Therefore:

> Do **not** interpret `newCachedThreadPool()` as a universally safe high-performance pool. Understand the workload and resource limits first.

### Best fit

- Many short-lived asynchronous tasks
- Bursty workloads where thread reuse is beneficial
- Situations where the concurrency level is acceptable and resource growth is controlled by the application environment

---

# 5. Single Thread Executor ⭐⭐⭐⭐⭐

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
```

Only one worker executes tasks sequentially.

```text
Task 1
  ↓
Task 2
  ↓
Task 3
  ↓
Task 4
```

### Practice Code

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class SingleThreadExecutorExample {
    public static void main(String[] args) {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        for (int i = 1; i <= 5; i++) {
            int taskId = i;

            executor.execute(() ->
                    System.out.println(
                            "Task " + taskId + " -> "
                                    + Thread.currentThread().getName()));
        }

        executor.shutdown();
    }
}
```

### Key property

Tasks execute sequentially through one worker. The executor also replaces the worker if it terminates unexpectedly, subject to executor lifecycle behavior.

### Best fit

- Sequential background processing
- Ordered task pipelines where executor submission order is used and tasks are executed one at a time
- Single-writer style processing

---

# 6. Scheduled Thread Pool ⭐⭐⭐⭐⭐

```java
ScheduledExecutorService scheduler =
        Executors.newScheduledThreadPool(2);
```

It supports:

- Delayed execution
- Fixed-rate execution
- Fixed-delay execution

### Practice Code

```java
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ScheduledThreadPoolExample {
    public static void main(String[] args) throws InterruptedException {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(2);

        scheduler.schedule(() ->
                System.out.println("Runs after 2 seconds"),
                2,
                TimeUnit.SECONDS);

        scheduler.scheduleAtFixedRate(() ->
                System.out.println("Periodic task"),
                0,
                3,
                TimeUnit.SECONDS);

        Thread.sleep(7000);
        scheduler.shutdown();
    }
}
```

For detailed scheduling semantics, see:

[8.6 — `ScheduledExecutorService`](../06-ScheduledExecutorService/README.md)

---

# 7. Single Thread Scheduled Executor ⭐⭐⭐⭐

```java
ScheduledExecutorService scheduler =
        Executors.newSingleThreadScheduledExecutor();
```

This combines:

```text
One worker
   +
Scheduling capability
```

### Practice Code

```java
import java.util.concurrent.*;

public class SingleThreadScheduledExample {
    public static void main(String[] args) throws InterruptedException {

        ScheduledExecutorService scheduler =
                Executors.newSingleThreadScheduledExecutor();

        scheduler.scheduleAtFixedRate(() ->
                System.out.println(
                        "Running on "
                                + Thread.currentThread().getName()),
                0,
                1,
                TimeUnit.SECONDS);

        Thread.sleep(3500);
        scheduler.shutdown();
    }
}
```

### Best fit

- Sequential scheduled work
- One periodic background worker
- Lightweight application maintenance tasks

---

# 8. Work-Stealing Pool ⭐⭐⭐⭐

```java
ExecutorService executor = Executors.newWorkStealingPool();
```

It uses a work-stealing executor, backed by the Fork/Join framework.

Conceptually:

```text
Worker A queue → tasks
Worker B queue → tasks
Worker C queue → empty
                    ↑
                 steal work
```

An idle worker can steal tasks from another worker's deque to improve utilization.

### Practice Code

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class WorkStealingPoolExample {
    public static void main(String[] args) throws InterruptedException {

        ExecutorService executor =
                Executors.newWorkStealingPool(4);

        for (int i = 1; i <= 12; i++) {
            int taskId = i;

            executor.submit(() -> {
                System.out.println(
                        "Task " + taskId + " -> "
                                + Thread.currentThread().getName());
            });
        }

        Thread.sleep(1000);
        executor.shutdown();
    }
}
```

### Important

`newWorkStealingPool()` is not simply a replacement for every fixed thread pool. It is particularly relevant to workloads that benefit from Fork/Join-style work stealing and parallel decomposition.

---

# 9. Fixed vs Cached vs Single ⭐⭐⭐⭐⭐

| | Fixed | Cached | Single |
|---|---|---|---|
| Worker concurrency | Fixed maximum | Can grow substantially | One |
| Reuses threads | Yes | Yes | Yes |
| Main idea | Controlled parallelism | Dynamic worker creation/reuse | Sequential execution |
| Main concern | Queue/resource sizing | Potential thread growth | Throughput limited to one worker |
| Typical use | Controlled workloads | Short-lived bursty tasks | Sequential background work |

### Memory Trick

```text
Fixed  → fixed number
Cached → create/reuse as needed
Single → one worker
Scheduled → time-based execution
Stealing → workers steal work
```

---

# 10. CPU-Bound vs I/O-Bound Pool Sizing ⭐⭐⭐⭐⭐

Pool type alone is not enough. Workload matters.

### CPU-bound

Examples:

- Compression
- CPU-heavy calculations
- Image processing

Too many concurrent CPU tasks can increase context switching and reduce throughput.

A common starting point is around the number of available processors, followed by measurement and tuning.

### I/O-bound

Examples:

- HTTP calls
- Database operations
- File I/O

Tasks often spend time waiting, so useful concurrency may be higher than CPU count, but the actual limit must account for downstream capacity, memory, connection pools and latency.

> There is no universal formula that guarantees the optimal pool size.

---

# 11. Practice — CPU-Bound Pool

```java
import java.util.concurrent.*;

public class CpuBoundPoolExample {
    public static void main(String[] args) {

        int processors = Runtime.getRuntime().availableProcessors();

        ExecutorService executor =
                Executors.newFixedThreadPool(processors);

        for (int i = 0; i < processors * 2; i++) {
            executor.submit(() -> calculate());
        }

        executor.shutdown();
    }

    private static void calculate() {
        long result = 0;
        for (int i = 0; i < 1_000_000; i++) {
            result += i;
        }
        System.out.println(result);
    }
}
```

Treat the processor-count choice as a starting experiment, not a guaranteed optimum.

---

# 12. Practice — I/O-Bound Simulation

```java
import java.util.concurrent.*;

public class IoBoundPoolExample {
    public static void main(String[] args) {

        int poolSize = 10;
        ExecutorService executor =
                Executors.newFixedThreadPool(poolSize);

        for (int i = 1; i <= 20; i++) {
            int taskId = i;

            executor.submit(() -> {
                try {
                    System.out.println("I/O task " + taskId + " started");
                    Thread.sleep(1000); // simulate waiting
                    System.out.println("I/O task " + taskId + " finished");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        executor.shutdown();
    }
}
```

In real systems, choose concurrency based on measurements and downstream limits rather than blindly increasing pool size.

---

# 13. Hidden Problem with `Executors` Factory Methods ⚠️⭐⭐⭐⭐⭐

Convenience factories are easy to use, but some have resource-management characteristics that may be unsafe for unbounded production workloads.

For example:

```java
Executors.newFixedThreadPool(10);
```

uses an unbounded `LinkedBlockingQueue` internally in the standard implementation.

Similarly:

```java
Executors.newCachedThreadPool();
```

can create a very large number of threads under sustained pressure.

### Production lesson

For strict resource control, consider configuring `ThreadPoolExecutor` explicitly:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        4,
        8,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(100),
        new ThreadPoolExecutor.CallerRunsPolicy()
);
```

This gives explicit control over:

- Core threads
- Maximum threads
- Keep-alive time
- Queue capacity
- Rejection policy

Detailed internals come in **8.8 — `ThreadPoolExecutor` Internals**.

---

# 14. Thread Pool Factory Selection Guide ⭐⭐⭐⭐⭐

```text
Need one worker?
      ↓
newSingleThreadExecutor()

Need fixed concurrency?
      ↓
newFixedThreadPool(n)

Need bursty short-lived tasks?
      ↓
Consider newCachedThreadPool()
      ↓
First verify resource limits

Need delayed/periodic tasks?
      ↓
newScheduledThreadPool(n)

Need sequential scheduled tasks?
      ↓
newSingleThreadScheduledExecutor()

Need Fork/Join-style work stealing?
      ↓
newWorkStealingPool()

Need strict queue/thread/rejection control?
      ↓
ThreadPoolExecutor
```

---

# 15. Real-World Scenarios ⭐⭐⭐⭐⭐

### Scenario 1 — Email processing

A service receives email jobs and wants controlled concurrency.

**Possible choice:** Fixed pool, with explicit capacity/backpressure requirements.

### Scenario 2 — Sequential audit writer

Only one task should write to a particular ordered stream at a time.

**Possible choice:** Single-thread executor, if the design and throughput requirements fit.

### Scenario 3 — Periodic cache refresh

**Possible choice:** Scheduled thread pool.

### Scenario 4 — CPU-heavy parallel computation

**Possible choice:** Fixed pool or Fork/Join/work-stealing approach depending on task decomposition.

### Scenario 5 — High-volume external API calls

Do not simply choose a larger pool. Consider:

```text
Pool size
+ Queue capacity
+ HTTP connection pool
+ Remote rate limits
+ Timeout
+ Retry
+ Backpressure
```

---

# 16. Common Mistakes ❌

### Mistake 1 — "More threads always means faster"

False. Excess threads can cause:

- Context switching
- Memory pressure
- CPU contention
- Downstream overload

### Mistake 2 — Using cached pool blindly

It can create many threads under load.

### Mistake 3 — Ignoring queue capacity

An unbounded queue can hide overload until memory pressure becomes severe.

### Mistake 4 — Not shutting down the executor

```java
executor.shutdown();
```

### Mistake 5 — Assuming a pool guarantees task ordering

Task execution order depends on the executor and workload. A fixed pool does not guarantee global completion ordering.

### Mistake 6 — Using one pool for every workload

Different workloads may require different concurrency, queueing and isolation policies.

---

# 17. Interview Scenarios

### Scenario 1

> I want exactly 5 worker threads.

**Answer:** `newFixedThreadPool(5)` as a simple factory, or explicit `ThreadPoolExecutor` when queue/rejection behavior must also be controlled.

### Scenario 2

> I want tasks to execute one at a time.

**Answer:** `newSingleThreadExecutor()`.

### Scenario 3

> I need periodic execution.

**Answer:** `ScheduledExecutorService`, commonly `newScheduledThreadPool(n)`.

### Scenario 4

> What is the biggest concern with `newCachedThreadPool()`?

**Answer:** It can create many threads under sustained load; resource growth must be considered.

### Scenario 5

> What is the major concern with `newFixedThreadPool()` in the convenience factory?

**Answer:** Its standard implementation uses an unbounded `LinkedBlockingQueue`, so queued work can grow substantially if producers outrun consumers.

### Scenario 6

> What is work stealing?

**Answer:** Idle workers can take tasks from another worker's deque, improving utilization for suitable parallel workloads.

### Scenario 7

> Which pool should I use for CPU-bound work?

**Answer:** Start with controlled parallelism near available processor count and benchmark; Fork/Join/work stealing can also fit recursively decomposable workloads.

### Scenario 8

> Which pool should I use for I/O-bound work?

**Answer:** Often a higher concurrency than CPU count may be useful, but size it based on measured latency and downstream limits rather than a fixed universal formula.

---

# 18. Quick Revision

```text
Fixed
  → controlled worker count

Cached
  → reuse idle / potentially create many workers

Single
  → one worker

Scheduled
  → delayed + periodic

Single Scheduled
  → one worker + scheduling

Work Stealing
  → Fork/Join + stealing

ThreadPoolExecutor
  → explicit production control
```

### One-Line Memory

> **Fixed = controlled, Cached = dynamic, Single = sequential, Scheduled = time-based, Work-Stealing = parallel load balancing.**

---

# 🎯 Interview Questions

1. What is a thread pool?
2. Why do we use thread pools?
3. Explain `newFixedThreadPool()`.
4. Explain `newCachedThreadPool()`.
5. Explain `newSingleThreadExecutor()`.
6. Explain `newScheduledThreadPool()`.
7. Explain `newSingleThreadScheduledExecutor()`.
8. What is `newWorkStealingPool()`?
9. Fixed vs cached thread pool?
10. Fixed vs single thread executor?
11. What is the danger of an unbounded executor queue?
12. Why can cached thread pools be dangerous under load?
13. How do you choose pool size for CPU-bound work?
14. How do you choose pool size for I/O-bound work?
15. What is work stealing?
16. When would you use `ThreadPoolExecutor` directly?
17. Why doesn't more concurrency always improve throughput?
18. What happens if producers are faster than consumers?
19. How do queue capacity and rejection policy affect a thread pool?
20. Explain thread pool types in 2 minutes.

---

# 🏆 2-Minute Interview Answer

> **"Java provides several executor pool types for different workload patterns. A fixed thread pool limits worker concurrency to a configured number and is useful when I need controlled parallelism. A single-thread executor runs tasks sequentially. A cached thread pool reuses idle workers and can create additional threads for demand, so I would be careful with it under sustained load. Scheduled thread pools support delayed and periodic tasks. A work-stealing pool uses the Fork/Join framework and can suit parallel, decomposable workloads. For production systems where I need explicit queue capacity, maximum threads, keep-alive and rejection behavior, I would usually configure `ThreadPoolExecutor` directly rather than relying only on convenience factories."**

---

# 💻 Practice Checklist

- [ ] Create a fixed thread pool.
- [ ] Observe worker reuse.
- [ ] Create a cached thread pool.
- [ ] Observe thread reuse/creation behavior.
- [ ] Create a single-thread executor.
- [ ] Verify sequential execution.
- [ ] Create a scheduled thread pool.
- [ ] Practice delayed execution.
- [ ] Practice periodic execution.
- [ ] Create a single-thread scheduled executor.
- [ ] Practice a work-stealing pool.
- [ ] Compare CPU-bound vs I/O-bound workload behavior.
- [ ] Identify unbounded-queue risk.
- [ ] Identify cached-pool thread-growth risk.
- [ ] Explain when to use `ThreadPoolExecutor` directly.
- [ ] Explain all thread-pool types in under 2 minutes.

---

## Navigation

[← 8.6 — `ScheduledExecutorService`](../06-ScheduledExecutorService/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.8 — `ThreadPoolExecutor` Internals**