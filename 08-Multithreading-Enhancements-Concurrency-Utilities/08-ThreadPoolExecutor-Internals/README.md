# 8.8 — `ThreadPoolExecutor` Internals

> **Goal:** Understand how `ThreadPoolExecutor` manages threads, queues, task submission, saturation, rejection and lifecycle — at interview and production level.

---

## 1. Why `ThreadPoolExecutor`? ⭐⭐⭐⭐⭐

Factory methods from `Executors` are convenient, but `ThreadPoolExecutor` gives explicit control over the most important resource-management parameters.

```text
                    ThreadPoolExecutor
                           │
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                  ↓
   Worker Threads        Queue          Rejection Policy
        │                  │                  │
        └────────────── Task Execution ───────┘
```

You can configure:

- Core pool size
- Maximum pool size
- Keep-alive time
- Work queue
- Thread factory
- Rejection policy

### Interview definition

> **`ThreadPoolExecutor` is a configurable implementation of `ExecutorService` that manages worker threads and queued tasks using core/max pool sizes, a work queue, keep-alive settings and a rejection policy.**

---

# 2. Constructor — Know Every Parameter ⭐⭐⭐⭐⭐

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        corePoolSize,
        maximumPoolSize,
        keepAliveTime,
        timeUnit,
        workQueue,
        threadFactory,
        rejectionHandler
);
```

Example:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        2,
        4,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(10),
        Executors.defaultThreadFactory(),
        new ThreadPoolExecutor.CallerRunsPolicy()
);
```

### Parameters

| Parameter | Meaning |
|---|---|
| `corePoolSize` | Baseline number of workers maintained by the pool |
| `maximumPoolSize` | Maximum worker count |
| `keepAliveTime` | Idle-time limit for eligible non-core workers |
| `unit` | Unit of keep-alive time |
| `workQueue` | Holds tasks waiting for workers |
| `threadFactory` | Creates worker threads |
| `handler` | Decides what happens when execution is rejected |

---

# 3. The Most Important Execution Algorithm ⭐⭐⭐⭐⭐

For a normal `execute(task)` submission, the high-level decision flow is:

```text
Task submitted
      ↓
worker count < corePoolSize ?
      │
   YES ─────────→ create worker for task
      │
     NO
      ↓
try to queue task
      │
   queued ───────→ wait for worker
      │
 queue full
      ↓
worker count < maximumPoolSize ?
      │
   YES ─────────→ create additional worker
      │
     NO
      ↓
REJECT TASK
```

### ⭐ Interview memory

> **Core → Queue → Max → Reject**

This is one of the most important `ThreadPoolExecutor` interview rules.

---

# 4. Core Pool Size ⭐⭐⭐⭐⭐

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        2,
        4,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(10)
);
```

`2` is the core pool size.

Normally, the executor tries to maintain this baseline of workers as tasks arrive.

### Practice Code

```java
import java.util.concurrent.*;

public class CorePoolExample {
    public static void main(String[] args) throws InterruptedException {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,
                4,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(10)
        );

        executor.execute(() -> sleepAndPrint("Task-1"));
        executor.execute(() -> sleepAndPrint("Task-2"));

        Thread.sleep(500);

        System.out.println("Pool size = " + executor.getPoolSize());
        System.out.println("Core size = " + executor.getCorePoolSize());
        System.out.println("Active = " + executor.getActiveCount());

        executor.shutdown();
    }

    private static void sleepAndPrint(String task) {
        try {
            Thread.sleep(1000);
            System.out.println(task + " -> "
                    + Thread.currentThread().getName());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

---

# 5. `maximumPoolSize` ⭐⭐⭐⭐⭐

The maximum pool size is an upper limit on worker threads.

```java
core = 2
max  = 4
```

The executor may grow beyond the core size when the queue cannot accept another task and workers are still below maximum.

> **Important:** A full queue is what normally creates the opportunity to grow from core workers toward maximum workers.

---

# 6. Work Queue ⭐⭐⭐⭐⭐

The queue stores tasks that cannot immediately be executed by available workers.

Common queues:

```text
ArrayBlockingQueue
LinkedBlockingQueue
SynchronousQueue
PriorityBlockingQueue
```

For production resource control, a bounded queue such as:

```java
new ArrayBlockingQueue<>(100)
```

is often useful because it puts an explicit limit on queued work.

---

# 7. Practice — Core → Queue → Max → Reject ⭐⭐⭐⭐⭐

This is the key experiment.

```java
import java.util.concurrent.*;

public class CoreQueueMaxRejectExample {
    public static void main(String[] args) throws InterruptedException {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,
                4,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(2),
                new ThreadPoolExecutor.AbortPolicy()
        );

        for (int i = 1; i <= 7; i++) {
            final int taskId = i;

            try {
                executor.execute(() -> {
                    System.out.println("Started Task " + taskId
                            + " by " + Thread.currentThread().getName());
                    sleep(3000);
                });

                System.out.println(
                        "Submitted " + taskId
                                + " | pool=" + executor.getPoolSize()
                                + " | active=" + executor.getActiveCount()
                                + " | queue=" + executor.getQueue().size());

            } catch (RejectedExecutionException e) {
                System.out.println("Rejected Task " + taskId);
            }
        }

        executor.shutdown();
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

With:

```text
core = 2
max  = 4
queue = 2
```

the important conceptual capacity path is:

```text
Task 1 → worker
Task 2 → worker
Task 3 → queue
Task 4 → queue
Task 5 → additional worker
Task 6 → additional worker
Task 7 → reject
```

Exact timing and observations can vary if tasks complete while submissions are happening.

---

# 8. `keepAliveTime` ⭐⭐⭐⭐

```java
30, TimeUnit.SECONDS
```

This controls how long eligible idle workers wait for new work before terminating.

By default, this primarily applies to workers beyond the core pool size.

### Allow core threads to time out

```java
executor.allowCoreThreadTimeOut(true);
```

When enabled, core workers can also terminate after being idle for at least the configured keep-alive time, subject to the executor's state and queue behavior.

### Practice Code

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        2,
        4,
        2,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(10)
);

executor.allowCoreThreadTimeOut(true);
```

Use this deliberately; aggressive timeout settings can cause unnecessary worker recreation.

---

# 9. Thread Factory ⭐⭐⭐⭐

A `ThreadFactory` controls how worker threads are created.

```java
ThreadFactory factory = runnable -> {
    Thread thread = new Thread(runnable);
    thread.setName("payment-worker-" + thread.getId());
    return thread;
};
```

### Practice Code

```java
import java.util.concurrent.*;

public class CustomThreadFactoryExample {
    public static void main(String[] args) {

        ThreadFactory factory = runnable -> {
            Thread thread = new Thread(runnable);
            thread.setName("worker-" + thread.getId());
            return thread;
        };

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,
                4,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(10),
                factory,
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        executor.execute(() ->
                System.out.println(Thread.currentThread().getName()));

        executor.shutdown();
    }
}
```

Useful for:

- Meaningful thread names
- Debugging
- Monitoring
- Uncaught exception handling
- Consistent thread configuration

---

# 10. Rejection Policies ⭐⭐⭐⭐⭐

When the pool is saturated:

```text
workers = maximumPoolSize
       +
queue = full
       ↓
RejectedExecutionHandler
```

Java provides four standard handlers.

| Policy | Behavior |
|---|---|
| `AbortPolicy` | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | Calling thread runs the task |
| `DiscardPolicy` | Silently discards the task |
| `DiscardOldestPolicy` | Drops the oldest queued task, then retries submission |

---

# 11. `AbortPolicy` ⭐⭐⭐⭐⭐

```java
new ThreadPoolExecutor.AbortPolicy()
```

It throws:

```java
RejectedExecutionException
```

### Practice Code

```java
import java.util.concurrent.*;

public class AbortPolicyExample {
    public static void main(String[] args) {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                1,
                1,
                0,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1),
                new ThreadPoolExecutor.AbortPolicy()
        );

        try {
            executor.execute(() -> sleep(3000));
            executor.execute(() -> System.out.println("Queued task"));
            executor.execute(() -> System.out.println("Rejected task"));
        } catch (RejectedExecutionException e) {
            System.out.println("Task rejected");
        } finally {
            executor.shutdown();
        }
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

---

# 12. `CallerRunsPolicy` ⭐⭐⭐⭐⭐

```java
new ThreadPoolExecutor.CallerRunsPolicy()
```

If the pool is saturated and the executor is still running, the submitting thread executes the task itself.

```text
Producer thread
      │
      ├── normal → executor
      │
      └── saturated → producer executes task
```

This can provide a simple form of backpressure because the producer is forced to spend time doing work.

### Practice Code

```java
import java.util.concurrent.*;

public class CallerRunsExample {
    public static void main(String[] args) {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                1,
                1,
                0,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        for (int i = 1; i <= 5; i++) {
            int taskId = i;

            executor.execute(() -> {
                System.out.println("Task " + taskId + " running on "
                        + Thread.currentThread().getName());
                sleep(1000);
            });
        }

        executor.shutdown();
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### Interview point

> `CallerRunsPolicy` can slow down a fast producer when the executor is saturated, but it should not be treated as a universal backpressure solution.

---

# 13. `DiscardPolicy` ⭐⭐⭐

```java
new ThreadPoolExecutor.DiscardPolicy()
```

Rejected tasks are silently discarded.

### Use carefully

This is appropriate only when losing work is explicitly acceptable.

For critical operations such as payments, order processing or audit records, silently dropping tasks is usually unacceptable unless there is another durable recovery mechanism.

---

# 14. `DiscardOldestPolicy` ⭐⭐⭐

```java
new ThreadPoolExecutor.DiscardOldestPolicy()
```

When saturated, the oldest queued task is discarded and submission is retried.

### Important

This is also workload-specific. Dropping the oldest queued task can violate business ordering or correctness requirements.

---

# 15. Rejection Policy Practice ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class RejectionPoliciesExample {
    public static void main(String[] args) {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                1,
                1,
                0,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1),
                new ThreadPoolExecutor.AbortPolicy()
        );

        executor.execute(() -> sleep(2000));
        executor.execute(() -> System.out.println("Queued"));

        try {
            executor.execute(() -> System.out.println("Third task"));
        } catch (RejectedExecutionException e) {
            System.out.println("Rejected by AbortPolicy");
        }

        executor.shutdown();
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

Repeat the experiment by replacing `AbortPolicy` with the other handlers and observe the behavior.

---

# 16. `execute()` vs `submit()` Internals ⭐⭐⭐⭐⭐

### `execute()`

```java
executor.execute(task);
```

- Accepts `Runnable`
- No Future result
- Rejection can be thrown directly

### `submit()`

```java
Future<?> future = executor.submit(task);
```

- Wraps the task in a Future-related mechanism
- Returns a `Future`
- Task exceptions are captured and become observable through `Future.get()` for submitted tasks

### Practice Code

```java
import java.util.concurrent.*;

public class ExecuteVsSubmitExample {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        executor.execute(() -> {
            throw new RuntimeException("execute failure");
        });

        Future<?> future = executor.submit(() -> {
            throw new RuntimeException("submit failure");
        });

        try {
            future.get();
        } catch (ExecutionException e) {
            System.out.println("Cause = " + e.getCause());
        }

        executor.shutdown();
    }
}
```

---

# 17. Runtime Monitoring APIs ⭐⭐⭐⭐⭐

`ThreadPoolExecutor` exposes useful metrics:

```java
executor.getPoolSize();
executor.getActiveCount();
executor.getCorePoolSize();
executor.getMaximumPoolSize();
executor.getQueue().size();
executor.getTaskCount();
executor.getCompletedTaskCount();
executor.getLargestPoolSize();
```

### Practice Code

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        2,
        4,
        30,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(10)
);

executor.execute(() -> sleep(1000));

System.out.println("Pool       = " + executor.getPoolSize());
System.out.println("Active     = " + executor.getActiveCount());
System.out.println("Queue      = " + executor.getQueue().size());
System.out.println("Completed  = " + executor.getCompletedTaskCount());
System.out.println("Largest    = " + executor.getLargestPoolSize());
```

These are useful for diagnostics and operational monitoring, while production monitoring should expose appropriate metrics rather than repeatedly logging raw values.

---

# 18. Prestarting Core Threads ⭐⭐⭐

Normally, core workers are created as work arrives.

You can proactively start them:

```java
executor.prestartAllCoreThreads();
```

Or:

```java
executor.prestartCoreThread();
```

### Practice

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        3,
        5,
        30,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(10)
);

System.out.println(executor.getPoolSize()); // often 0 initially

executor.prestartAllCoreThreads();

System.out.println(executor.getPoolSize()); // 3

executor.shutdown();
```

Useful when you deliberately want workers initialized before the first burst of work.

---

# 19. Changing Pool Sizes at Runtime ⭐⭐⭐⭐

Some executor parameters can be changed:

```java
executor.setCorePoolSize(4);
executor.setMaximumPoolSize(8);
executor.setKeepAliveTime(60, TimeUnit.SECONDS);
```

### Important

When changing core and maximum sizes, maintain valid relationships and understand the effect on existing workers and queued work.

---

# 20. Shutdown Lifecycle ⭐⭐⭐⭐⭐

```text
RUNNING
   ↓ shutdown()
SHUTDOWN
   ↓ queued tasks drain
   ↓ workers terminate
TERMINATED
```

Useful APIs:

```java
executor.shutdown();
executor.shutdownNow();
executor.awaitTermination(...);
executor.isShutdown();
executor.isTerminated();
```

### Practice Code

```java
import java.util.List;
import java.util.concurrent.*;

public class GracefulShutdownExample {
    public static void main(String[] args) throws InterruptedException {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,
                4,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(10)
        );

        for (int i = 0; i < 5; i++) {
            executor.execute(() -> sleep(1000));
        }

        executor.shutdown();

        if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
            List<Runnable> pending = executor.shutdownNow();
            System.out.println("Pending tasks = " + pending.size());
        }
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

---

# 21. Queue Choice Matters ⭐⭐⭐⭐⭐

### `ArrayBlockingQueue`

```java
new ArrayBlockingQueue<>(100)
```

- Bounded
- Explicit capacity
- Useful for resource control

### `LinkedBlockingQueue`

```java
new LinkedBlockingQueue<>(100)
```

- Can be bounded when capacity is supplied
- Without explicit capacity, its capacity is very large (`Integer.MAX_VALUE`)

### `SynchronousQueue`

```java
new SynchronousQueue<>()
```

- No normal internal task capacity
- Each handoff pairs a producer with a consumer/worker
- Commonly associated with cached executor behavior

### Interview point

> Queue choice directly affects when the executor queues work, creates additional workers, or rejects tasks.

---

# 22. `ThreadPoolExecutor` Mental Model ⭐⭐⭐⭐⭐

```text
                 submit task
                      ↓
             ┌─────────────────┐
             │ worker < core ? │
             └────────┬────────┘
                      │ YES
                      ↓
                create worker
                      │
                     NO
                      ↓
                 offer queue
                      │
             ┌────────┴────────┐
             │ accepted?       │
             └───────┬─────────┘
                 YES │ NO
                     │   ↓
                     │ worker < max ?
                     │   │
                     │ YES → create worker
                     │   │
                     │ NO  → reject
                     ↓
                  wait/execute
```

### Remember

> **Core → Queue → Max → Reject**

---

# 23. Real-World Example — Payment Processing ⭐⭐⭐⭐⭐

Suppose a payment service receives requests that trigger background processing.

```text
Incoming requests
       ↓
Bounded queue
       ↓
Payment worker pool
       ↓
External payment system
```

Possible design:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        4,
        8,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(200),
        new ThreadPoolExecutor.CallerRunsPolicy()
);
```

But pool sizing should be based on:

- Payment-system throughput
- DB connection pool
- HTTP connection pool
- Latency
- Rate limits
- Memory
- Retry behavior
- Business correctness

Never choose numbers only because they "look good."

---

# 24. Backpressure Connection ⭐⭐⭐⭐⭐

When producers are faster than consumers:

```text
Producer rate > Processing rate
             ↓
Queue grows
             ↓
Queue full
             ↓
Reject / slow producer
```

Possible strategies include:

- Bounded queue
- `CallerRunsPolicy`
- Explicit rejection handling
- Rate limiting
- Upstream throttling
- Load shedding
- Durable messaging systems where appropriate

Detailed backpressure and rejection patterns are covered in **8.9 — Queueing, Rejection Policies & Backpressure**.

---

# 25. Common Mistakes ❌

### Mistake 1

> "Maximum pool size means the executor immediately creates that many threads."

False. Workers generally grow beyond core only under the executor's queue/fullness conditions.

### Mistake 2

> "Increasing max threads always improves throughput."

False. Downstream resources and CPU can become bottlenecks.

### Mistake 3

> "The queue is just storage."

False. Queue choice changes executor behavior and overload handling.

### Mistake 4

> "Unbounded queue + high max pool means max threads will always be used."

Usually false: with a typical unbounded queue, tasks keep queuing instead of triggering growth beyond core under normal execution semantics.

### Mistake 5

> "CallerRunsPolicy guarantees backpressure."

It can provide a form of producer slowdown, but it is not a complete backpressure architecture.

### Mistake 6

> "I can ignore shutdown."

Executors own worker resources and should have a defined lifecycle.

---

# 26. Interview Scenarios ⭐⭐⭐⭐⭐

### Q1. Explain `ThreadPoolExecutor`.

**Answer:** It manages worker threads and queued tasks using configurable core/max workers, keep-alive, queue, thread factory and rejection policy.

### Q2. What happens when a task is submitted?

**Answer:** Remember **Core → Queue → Max → Reject**.

### Q3. Why use a bounded queue?

**Answer:** To limit queued work and make overload behavior explicit rather than allowing uncontrolled queue growth.

### Q4. Why doesn't maximum pool size immediately create maximum workers?

**Answer:** The executor normally creates core workers first and then queues tasks. Additional workers are created when the queue cannot accept work and the worker count is still below maximum.

### Q5. What is `CallerRunsPolicy`?

**Answer:** When saturated and still running, the submitting thread executes the task itself.

### Q6. What is `keepAliveTime`?

**Answer:** It controls how long eligible idle workers beyond the core size wait before termination.

### Q7. What is a `ThreadFactory`?

**Answer:** It creates the threads used by the executor, allowing custom naming and thread configuration.

### Q8. Why prefer explicit `ThreadPoolExecutor` configuration in production?

**Answer:** It gives direct control over resource limits, queue capacity and rejection behavior.

---

# 27. Quick Revision

```text
ThreadPoolExecutor
       │
       ├── corePoolSize
       ├── maximumPoolSize
       ├── keepAliveTime
       ├── BlockingQueue
       ├── ThreadFactory
       └── RejectedExecutionHandler
```

### Most Important Rule

```text
Task
 ↓
Core workers?
 ↓ no
Queue?
 ↓ full
Max workers?
 ↓ full
Reject
```

### Memory Trick

> **Core → Queue → Max → Reject**

---

# 🎯 Interview Questions

1. What is `ThreadPoolExecutor`?
2. Explain all seven constructor parameters.
3. Explain `corePoolSize`.
4. Explain `maximumPoolSize`.
5. Explain `keepAliveTime`.
6. What is the role of `BlockingQueue`?
7. Explain Core → Queue → Max → Reject.
8. Why can an unbounded queue prevent growth toward `maximumPoolSize`?
9. What are the four standard rejection policies?
10. Explain `CallerRunsPolicy`.
11. What is a `ThreadFactory`?
12. How do you name executor threads?
13. What is `allowCoreThreadTimeOut()`?
14. What is `prestartAllCoreThreads()`?
15. How can you monitor a thread pool?
16. `execute()` vs `submit()`?
17. How does queue choice affect thread creation?
18. How would you design a bounded executor?
19. How does `ThreadPoolExecutor` help with backpressure?
20. Why shouldn't you blindly increase pool size?
21. Explain graceful shutdown.
22. What happens to queued tasks during shutdown?
23. How would you handle rejected payment-processing tasks?
24. Why is `ThreadPoolExecutor` preferred over convenience factories in some production systems?
25. Explain `ThreadPoolExecutor` internals in 2 minutes.

---

# 🏆 2-Minute Interview Answer

> **"`ThreadPoolExecutor` is the configurable implementation behind many Java executor patterns. Its main parameters are core pool size, maximum pool size, keep-alive time, work queue, thread factory and rejection handler. The key task-submission rule is Core → Queue → Max → Reject: workers are created up to the core size, then tasks are normally queued; if the queue is full, additional workers can be created up to the maximum, and after that the rejection policy is applied. This means queue choice is extremely important. In production, I prefer explicit bounded queues and an appropriate rejection strategy when I need predictable resource usage and overload behavior. I also monitor pool and queue metrics and manage executor shutdown explicitly."**

---

# 💻 Practice Checklist

- [ ] Create a `ThreadPoolExecutor` manually.
- [ ] Experiment with `corePoolSize`.
- [ ] Experiment with `maximumPoolSize`.
- [ ] Observe `keepAliveTime`.
- [ ] Use a bounded `ArrayBlockingQueue`.
- [ ] Demonstrate Core → Queue → Max → Reject.
- [ ] Practice all four rejection policies.
- [ ] Practice `CallerRunsPolicy` as a producer-slowdown mechanism.
- [ ] Create a custom `ThreadFactory`.
- [ ] Monitor pool and queue metrics.
- [ ] Practice `prestartAllCoreThreads()`.
- [ ] Practice `allowCoreThreadTimeOut(true)`.
- [ ] Compare `ArrayBlockingQueue`, `LinkedBlockingQueue` and `SynchronousQueue`.
- [ ] Practice graceful shutdown.
- [ ] Design a bounded executor for a real-world service.
- [ ] Explain `ThreadPoolExecutor` internals in under 2 minutes.

---

## Navigation

[← 8.7 — Thread Pool Types](../07-Thread-Pool-Types/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.9 — Queueing, Rejection Policies & Backpressure**