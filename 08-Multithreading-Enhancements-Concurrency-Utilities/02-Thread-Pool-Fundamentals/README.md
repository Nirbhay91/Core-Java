# 8.2 — Thread Pool Fundamentals

> **Goal:** Understand how a thread pool reuses worker threads, limits concurrency, queues tasks, and manages worker lifecycle.

---

## 1. What is a Thread Pool?

A **thread pool** is a managed collection of worker threads that execute submitted tasks.

Instead of:

```java
new Thread(task).start();
```

for every task, we submit tasks to an executor backed by a pool.

```text
Tasks
  ↓
ExecutorService
  ↓
Task Queue
  ↓
Worker Threads
  ↓
Execute + Reuse
```

### Main benefits

- Thread reuse
- Controlled concurrency
- Less thread-creation overhead
- Centralized lifecycle management
- Better resource control
- Queueing of pending tasks

---

# 2. Why Not One Thread Per Task?

```java
for (int i = 1; i <= 10_000; i++) {
    new Thread(task).start();
}
```

This can create an excessive number of threads and cause memory pressure, scheduling overhead and poor application stability.

A pool lets us put a bound on the number of actively executing worker threads.

---

# 3. Basic Fixed Thread Pool ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class FixedThreadPoolExample {
    public static void main(String[] args) {

        ExecutorService executor = Executors.newFixedThreadPool(3);

        for (int i = 1; i <= 8; i++) {
            int taskId = i;

            executor.execute(() -> {
                System.out.println("Task " + taskId
                        + " started by "
                        + Thread.currentThread().getName());

                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }

                System.out.println("Task " + taskId + " completed");
            });
        }

        executor.shutdown();
    }
}
```

### Observe

There are 8 tasks but only 3 worker threads are configured.

```text
Task 1 ─┐
Task 2 ─┼─→ 3 workers
Task 3 ─┘

Task 4–8 wait in the executor's work queue as needed.
```

The same worker threads can execute later tasks after completing earlier tasks.

---

# 4. Thread Reuse ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadReuseExample {
    public static void main(String[] args) {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        for (int i = 1; i <= 6; i++) {
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

### Expected idea

You should see a small set of worker-thread names repeatedly handling multiple tasks.

> **Thread pool = worker reuse, not one permanent thread per task.**

---

# 5. Pool Size vs Number of Tasks

Suppose:

```java
ExecutorService executor = Executors.newFixedThreadPool(2);
```

and there are 10 tasks.

Conceptually:

```text
Pool size = 2
Tasks    = 10

Worker-1 → Task 1 → Task 3 → Task 5 → ...
Worker-2 → Task 2 → Task 4 → Task 6 → ...

Remaining tasks wait until workers become available.
```

The exact execution order is **not guaranteed**.

---

# 6. CPU-Bound vs I/O-Bound Work

Thread-pool sizing depends on workload.

### CPU-bound

Examples:

- Calculations
- Data transformation
- CPU-heavy algorithms

Too many active threads can increase context switching and reduce efficiency.

### I/O-bound

Examples:

- HTTP calls
- Database calls
- File/network operations

Tasks may spend significant time waiting, so an appropriately larger concurrency level can sometimes improve throughput.

> There is no universal "pool size = X" rule. Measure the workload and consider CPU, latency, blocking behavior, memory, downstream limits and application requirements.

---

# 7. Practice — CPU vs I/O Simulation

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class WorkloadSimulation {
    public static void main(String[] args) {

        ExecutorService executor = Executors.newFixedThreadPool(4);

        for (int i = 1; i <= 8; i++) {
            int taskId = i;

            executor.submit(() -> {
                try {
                    // Simulated blocking work
                    Thread.sleep(300);
                    System.out.println("I/O-like task " + taskId
                            + " completed by "
                            + Thread.currentThread().getName());
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        executor.shutdown();
    }
}
```

Change the pool size and compare completion behavior.

---

# 8. Fixed Thread Pool Characteristics

`Executors.newFixedThreadPool(n)` provides a pool whose core and maximum worker count are fixed at `n` in the standard implementation.

Important conceptual model:

```text
                ┌──────────────┐
Tasks ────────→ │ Work Queue   │
                └──────┬───────┘
                       ↓
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Worker-1     Worker-2     Worker-3
          ↓            ↓            ↓
        Task         Task         Task
```

Pending tasks are queued until a worker becomes available.

---

# 9. Thread Pool Lifecycle

```text
Created
   ↓
Accept tasks
   ↓
Workers execute tasks
   ↓
shutdown()
   ↓
No new tasks accepted
   ↓
Previously submitted tasks finish
   ↓
Terminated
```

Practice:

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class PoolLifecycleExample {
    public static void main(String[] args) throws InterruptedException {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        executor.execute(() -> System.out.println("Task 1"));
        executor.execute(() -> System.out.println("Task 2"));

        System.out.println("Shutdown requested: "
                + executor.isShutdown());

        executor.shutdown();

        System.out.println("Shutdown requested: "
                + executor.isShutdown());

        while (!executor.isTerminated()) {
            Thread.sleep(50);
        }

        System.out.println("Pool terminated");
    }
}
```

---

# 10. `shutdown()` Does Not Mean Immediate Stop

```java
executor.shutdown();
```

Means:

- Stop accepting new tasks.
- Previously submitted tasks are allowed to complete under normal executor shutdown semantics.

For forceful interruption attempts, there is:

```java
executor.shutdownNow();
```

Detailed comparison is covered in **8.4**.

---

# 11. Queueing Concept ⭐⭐⭐⭐⭐

With a bounded worker count, tasks that cannot immediately be assigned to a worker may wait in the executor's work queue.

```text
submit tasks
     ↓
[worker available?]
     ↓ yes             ↓ no
execute             queue task
                       ↓
                worker completes
                       ↓
                take next task
```

Queue capacity and rejection behavior become especially important when using `ThreadPoolExecutor` directly.

---

# 12. Custom `ThreadPoolExecutor` Preview

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ThreadPoolExecutorPreview {
    public static void main(String[] args) {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,                  // corePoolSize
                4,                  // maximumPoolSize
                10, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(5)
        );

        for (int i = 1; i <= 10; i++) {
            int taskId = i;
            executor.execute(() ->
                    System.out.println("Task " + taskId
                            + " -> "
                            + Thread.currentThread().getName()));
        }

        executor.shutdown();
    }
}
```

This previews the more detailed topic **8.8 — `ThreadPoolExecutor` Internals**.

---

# 13. Monitoring a Pool ⭐⭐⭐⭐

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        2,
        4,
        10,
        java.util.concurrent.TimeUnit.SECONDS,
        new java.util.concurrent.ArrayBlockingQueue<>(5)
);

System.out.println("Pool size: " + executor.getPoolSize());
System.out.println("Active count: " + executor.getActiveCount());
System.out.println("Completed: " + executor.getCompletedTaskCount());
System.out.println("Queue size: " + executor.getQueue().size());
```

These metrics can help understand executor behavior during diagnostics. Production monitoring should use an appropriate observability approach rather than relying only on ad-hoc logging.

---

# 14. Wrong Assumption ❌

### "A pool of 10 means exactly 10 threads always run."

Not necessarily.

A pool configuration controls how workers can be created/used according to its implementation and workload. Threads may be created lazily, and pool lifecycle policies affect worker retention.

For a fixed pool, the configured size bounds the worker count used for normal task execution, but it does not mean all workers are actively executing all the time.

---

# 15. Practice Challenge ⭐⭐⭐⭐⭐

Create a program with:

- 3 worker threads
- 12 tasks
- Each task sleeps for 1 second
- Print task ID and worker name
- Measure total execution time

Then change the pool size to:

```text
1
2
3
6
12
```

Compare the runtime and explain why the results differ.

### Important

The measured time will vary by machine and scheduling. The goal is to understand **concurrency and throughput**, not memorize a fixed timing.

---

# 16. Real-World Example

Imagine an order service receives 1,000 independent background jobs.

Bad approach:

```text
1 request
   ↓
1 new Thread
   ↓
1,000 requests
   ↓
1,000 threads
```

Better conceptual design:

```text
1,000 tasks
    ↓
ExecutorService
    ↓
Controlled worker pool
    ↓
Queue + workers
    ↓
Controlled downstream load
```

The exact pool size should consider:

- CPU capacity
- Blocking time
- Database connection pool size
- Remote service rate limits
- Memory
- Latency requirements
- Queue capacity
- Rejection strategy

---

# 17. Common Mistakes

### ❌ Mistake 1 — Unlimited concurrency

More threads do not automatically mean more performance.

### ❌ Mistake 2 — Ignoring downstream limits

A large pool can overwhelm a database or remote API.

### ❌ Mistake 3 — Never shutting down a short-lived executor

Executor lifecycle must have a clear owner.

### ❌ Mistake 4 — Assuming task order

Thread pools do not automatically guarantee that tasks complete in submission order.

### ❌ Mistake 5 — Choosing pool size without considering blocking

CPU-bound and I/O-heavy workloads have different characteristics.

---

# 18. Interview Questions

1. What is a thread pool?
2. Why do we use thread pools?
3. What is thread reuse?
4. What happens when tasks exceed the number of worker threads?
5. What is the difference between task count and pool size?
6. How does `newFixedThreadPool()` behave conceptually?
7. Why is pool sizing workload-dependent?
8. CPU-bound vs I/O-bound pool sizing?
9. What is the role of a work queue?
10. What happens after `shutdown()`?
11. Does a fixed pool execute tasks in submission order?
12. Why can too many threads hurt performance?
13. What downstream resources should be considered when sizing a pool?
14. Why might `ThreadPoolExecutor` be preferred when precise queue/rejection behavior is required?
15. How would you monitor a production executor?

---

# 🏆 2-Minute Interview Answer

> **"A thread pool is a managed set of worker threads that repeatedly execute submitted tasks. Its main purpose is to reuse threads and control concurrency instead of creating an uncontrolled number of threads. With a fixed pool, tasks beyond the currently available workers wait in the executor's work queue until workers become available. Pool sizing is workload-dependent: CPU-heavy work is constrained by CPU capacity, while blocking I/O can require a different concurrency level. In production I also consider queue capacity, database connections, downstream rate limits, memory, rejection behavior, monitoring and graceful shutdown. More threads do not automatically mean better performance."**

---

# 💻 Practice Checklist

- [ ] Create a fixed pool of 3 workers.
- [ ] Submit 8 tasks.
- [ ] Observe thread reuse.
- [ ] Compare pool sizes 1, 2, 3, 6 and 12.
- [ ] Measure execution time.
- [ ] Explain task queueing.
- [ ] Practice executor lifecycle.
- [ ] Run the `ThreadPoolExecutor` preview.
- [ ] Explain CPU-bound vs I/O-bound workloads.
- [ ] Explain thread-pool sizing in under 2 minutes.

---

## Navigation

[← 8.1 — Executor and ExecutorService Fundamentals](../01-Executor-and-ExecutorService-Fundamentals/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.3 — `execute()` vs `submit()`**