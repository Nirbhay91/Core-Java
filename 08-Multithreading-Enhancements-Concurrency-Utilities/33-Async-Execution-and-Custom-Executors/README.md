# 8.33 — Async Execution & Custom Executors

> **Goal:** Understand how `CompletableFuture` chooses threads for async stages, how to provide a custom `Executor`, and why executor selection matters in production.

---

## 1. Core Mental Model ⭐⭐⭐⭐⭐

`CompletableFuture` has two broad styles of continuation:

```text
thenApply / thenCompose / thenAccept
→ synchronous continuation
→ may run in the thread that completes the previous stage

thenApplyAsync / thenComposeAsync / thenAcceptAsync
→ asynchronous continuation
→ uses the default async executor unless a custom Executor is supplied
```

With a custom executor:

```java
.thenApplyAsync(fn, executor)
```

you explicitly control where that async stage is submitted.

---

# 2. `thenApply()` vs `thenApplyAsync()` ⭐⭐⭐⭐⭐

```java
CompletableFuture.supplyAsync(() -> "data")
        .thenApply(value -> transform(value));
```

versus:

```java
CompletableFuture.supplyAsync(() -> "data")
        .thenApplyAsync(value -> transform(value));
```

The important point is **not** simply:

```text
thenApply = same thread always
thenApplyAsync = new thread always
```

That is too simplistic.

The correct mental model is:

```text
thenApply
→ non-async continuation
→ execution may occur in the thread completing the stage

thenApplyAsync
→ async continuation
→ submitted to an async executor
```

---

# 3. Practice — Compare Thread Names ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ApplyVsApplyAsyncDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    System.out.println("supplyAsync: " +
                            Thread.currentThread().getName());
                    return "java";
                })
                .thenApply(value -> {
                    System.out.println("thenApply: " +
                            Thread.currentThread().getName());
                    return value.toUpperCase();
                })
                .thenApplyAsync(value -> {
                    System.out.println("thenApplyAsync: " +
                            Thread.currentThread().getName());
                    return value + "!";
                });

        System.out.println(future.join());
    }
}
```

### Practice question

Run it multiple times and ask:

> Which thread executed each stage, and what guarantee does the API actually provide?

---

# 4. Default Async Executor ⭐⭐⭐⭐⭐

For async methods without an explicit executor:

```java
thenApplyAsync(...)
thenComposeAsync(...)
thenAcceptAsync(...)
runAfterBothAsync(...)
```

the stage uses the `CompletableFuture` async execution mechanism, which by default is based on the common `ForkJoinPool` when parallelism is available.

Do not memorize it as:

```text
Async = always creates a new thread
```

It does not necessarily create a brand-new thread per task.

---

# 5. Custom `Executor` ⭐⭐⭐⭐⭐

You can supply an executor explicitly:

```java
ExecutorService executor =
        Executors.newFixedThreadPool(4);

CompletableFuture.supplyAsync(() -> loadData(), executor)
        .thenApplyAsync(this::transform, executor);
```

Benefits:

```text
✓ Explicit thread ownership
✓ Isolation between workloads
✓ Easier capacity control
✓ Better production tuning
✓ Prevent accidental use of shared common pool
```

---

# 6. Practice — Custom Executor ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CustomExecutorDemo {

    public static void main(String[] args) {
        ExecutorService executor =
                Executors.newFixedThreadPool(2);

        try {
            CompletableFuture<String> future =
                    CompletableFuture.supplyAsync(() -> {
                        System.out.println("Load: " +
                                Thread.currentThread().getName());
                        return "data";
                    }, executor)
                    .thenApplyAsync(value -> {
                        System.out.println("Transform: " +
                                Thread.currentThread().getName());
                        return value.toUpperCase();
                    }, executor);

            System.out.println(future.join());
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 7. `supplyAsync()` With Custom Executor ⭐⭐⭐⭐⭐

You can control the executor of the initial async computation:

```java
CompletableFuture.supplyAsync(task, executor);
```

This is different from supplying an executor only to a later continuation.

Example:

```text
Executor A
   ↓
supplyAsync
   ↓
result
   ↓
Executor B
   ↓
thenApplyAsync(..., executorB)
```

Different stages can intentionally use different executors.

---

# 8. Practice — Two Executors ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class TwoExecutorsDemo {

    public static void main(String[] args) {
        ExecutorService ioExecutor =
                Executors.newFixedThreadPool(2);
        ExecutorService cpuExecutor =
                Executors.newFixedThreadPool(2);

        try {
            CompletableFuture<String> future =
                    CompletableFuture.supplyAsync(() -> {
                        System.out.println("I/O: " +
                                Thread.currentThread().getName());
                        return "raw-data";
                    }, ioExecutor)
                    .thenApplyAsync(value -> {
                        System.out.println("CPU: " +
                                Thread.currentThread().getName());
                        return value.toUpperCase();
                    }, cpuExecutor);

            System.out.println(future.join());
        } finally {
            ioExecutor.shutdown();
            cpuExecutor.shutdown();
        }
    }
}
```

This demonstrates workload isolation:

```text
I/O executor → blocking/remote work
CPU executor → CPU-bound transformation
```

The exact executor split should be based on workload and capacity, not a rigid rule.

---

# 9. `thenComposeAsync()` ⭐⭐⭐⭐⭐

For dependent asynchronous operations:

```java
loadUser()
    .thenComposeAsync(user -> loadOrders(user), executor);
```

The supplied executor controls the async continuation stage.

---

# 10. Practice — Async Composition ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ComposeAsyncDemo {

    static CompletableFuture<String> loadUser(ExecutorService executor) {
        return CompletableFuture.supplyAsync(() -> "User-101", executor);
    }

    static CompletableFuture<String> loadOrders(
            String user, ExecutorService executor) {
        return CompletableFuture.supplyAsync(
                () -> user + " -> Orders", executor);
    }

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(4);

        try {
            CompletableFuture<String> result =
                    loadUser(executor)
                            .thenComposeAsync(
                                    user -> loadOrders(user, executor),
                                    executor);

            System.out.println(result.join());
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 11. `thenCombineAsync()` ⭐⭐⭐⭐⭐

For independent operations:

```text
User service       ─┐
                    ├→ combine
Recommendation      ─┘
```

You can control the combination stage with:

```java
thenCombineAsync(other, combiner, executor)
```

---

# 12. Practice — Async Combination ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CombineAsyncDemo {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(4);

        try {
            CompletableFuture<String> user =
                    CompletableFuture.supplyAsync(() -> "Nirbhay", executor);

            CompletableFuture<String> orders =
                    CompletableFuture.supplyAsync(() -> "5 orders", executor);

            CompletableFuture<String> result =
                    user.thenCombineAsync(
                            orders,
                            (u, o) -> u + " has " + o,
                            executor);

            System.out.println(result.join());
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 13. `runAsync()` With Custom Executor ⭐⭐⭐⭐

When there is no result:

```java
CompletableFuture.runAsync(task, executor);
```

Use it for asynchronous side-effect work where no value is returned.

---

# 14. Practice — `runAsync()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class RunAsyncDemo {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        try {
            CompletableFuture<Void> future =
                    CompletableFuture.runAsync(() -> {
                        System.out.println("Sending notification on: " +
                                Thread.currentThread().getName());
                    }, executor);

            future.join();
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 15. `thenAcceptAsync()` and `thenRunAsync()` ⭐⭐⭐⭐

Use:

```java
thenAcceptAsync(value -> ..., executor)
```

when the result is consumed.

Use:

```java
thenRunAsync(() -> ..., executor)
```

when the previous result is not needed.

---

# 16. Practice — Consumer and Runnable Stages ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ConsumerAndRunnableDemo {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        try {
            CompletableFuture.completedFuture("ORDER-101")
                    .thenAcceptAsync(order ->
                            System.out.println("Email for " + order),
                            executor)
                    .join();

            CompletableFuture.completedFuture("ORDER-102")
                    .thenRunAsync(() ->
                            System.out.println("Audit completed"),
                            executor)
                    .join();
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 17. Why Custom Executors Matter ⭐⭐⭐⭐⭐

Imagine one JVM has:

```text
Payment calls
Search calls
Email notifications
Report generation
Image processing
```

If all async work shares one executor, a slow workload can consume capacity needed by another workload.

A custom executor can isolate workloads:

```text
paymentExecutor
searchExecutor
notificationExecutor
```

This is a **capacity-isolation** decision, not merely a syntax preference.

---

# 18. Practice — Workload Isolation ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class WorkloadIsolationDemo {

    public static void main(String[] args) {
        ExecutorService paymentExecutor =
                Executors.newFixedThreadPool(4);
        ExecutorService notificationExecutor =
                Executors.newFixedThreadPool(2);

        try {
            CompletableFuture<String> payment =
                    CompletableFuture.supplyAsync(() ->
                            "PAYMENT_DONE", paymentExecutor);

            CompletableFuture<String> notification =
                    CompletableFuture.supplyAsync(() ->
                            "EMAIL_DONE", notificationExecutor);

            System.out.println(payment.join());
            System.out.println(notification.join());
        } finally {
            paymentExecutor.shutdown();
            notificationExecutor.shutdown();
        }
    }
}
```

---

# 19. Common Pool vs Custom Executor ⭐⭐⭐⭐⭐

| Situation | Typical choice |
|---|---|
| Small CPU-oriented async transformation | Default async executor may be sufficient |
| Need explicit capacity control | Custom executor |
| Need workload isolation | Custom executor |
| Blocking external I/O | Carefully designed dedicated executor or appropriate async I/O model |
| Application-specific thread naming/monitoring | Custom executor |
| Need bounded queue/rejection policy | `ThreadPoolExecutor` |

Avoid a blanket rule such as:

```text
"Always use a custom executor."
```

The correct choice depends on workload, ownership, capacity and operational requirements.

---

# 20. Blocking Work Inside `CompletableFuture` 🚨

A major production risk is running blocking work on an executor intended for CPU-oriented async tasks.

Example:

```java
CompletableFuture.supplyAsync(() -> {
    blockingDatabaseCall();
    return result;
});
```

If many tasks block simultaneously, available workers can be exhausted.

Think:

```text
Blocking task
   ↓
worker occupied
   ↓
more tasks queued
   ↓
latency increases
```

---

# 21. Practice — Simulate Blocking Work ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class BlockingWorkDemo {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        try {
            CompletableFuture.allOf(
                    task(executor, 1),
                    task(executor, 2),
                    task(executor, 3),
                    task(executor, 4)
            ).join();
        } finally {
            executor.shutdown();
        }
    }

    static CompletableFuture<Void> task(ExecutorService executor, int id) {
        return CompletableFuture.runAsync(() -> {
            try {
                System.out.println("Start " + id);
                TimeUnit.SECONDS.sleep(1);
                System.out.println("End " + id);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            }
        }, executor);
    }
}
```

Observe how only two tasks can occupy the two workers simultaneously.

---

# 22. `ThreadPoolExecutor` for Explicit Control ⭐⭐⭐⭐⭐

For production-sensitive workloads, you may want:

```text
core pool size
maximum pool size
keep-alive time
work queue
thread factory
rejection policy
```

Example:

```java
ExecutorService executor = new ThreadPoolExecutor(
        4,
        8,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(100)
);
```

This connects directly to Chapter 8 topics on `ThreadPoolExecutor`, queueing, rejection policies and backpressure.

---

# 23. Practice — Bounded Custom Executor ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class BoundedExecutorDemo {

    public static void main(String[] args) {
        ExecutorService executor = new ThreadPoolExecutor(
                2,
                4,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(10),
                new ThreadPoolExecutor.CallerRunsPolicy());

        try {
            for (int i = 1; i <= 20; i++) {
                int taskId = i;
                executor.execute(() ->
                        System.out.println("Task " + taskId +
                                " on " + Thread.currentThread().getName()));
            }
        } finally {
            executor.shutdown();
        }
    }
}
```

`CallerRunsPolicy` can provide a form of producer-side throttling by making the submitting thread execute rejected tasks when the pool is saturated.

---

# 24. Custom `ThreadFactory` ⭐⭐⭐⭐⭐

A custom executor becomes easier to operate when worker threads have meaningful names.

```java
ThreadFactory factory = runnable -> {
    Thread thread = new Thread(runnable);
    thread.setName("payment-worker");
    return thread;
};
```

For serious applications, prefer a naming strategy that generates unique names such as:

```text
payment-worker-1
payment-worker-2
```

---

# 25. Practice — Named Workers ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

public class NamedThreadFactoryDemo {

    public static void main(String[] args) {
        AtomicInteger counter = new AtomicInteger(1);

        ThreadFactory factory = runnable -> {
            Thread thread = new Thread(runnable);
            thread.setName("payment-worker-" + counter.getAndIncrement());
            return thread;
        };

        ExecutorService executor =
                Executors.newFixedThreadPool(2, factory);

        try {
            CompletableFuture.allOf(
                    CompletableFuture.runAsync(() ->
                            System.out.println(Thread.currentThread().getName()), executor),
                    CompletableFuture.runAsync(() ->
                            System.out.println(Thread.currentThread().getName()), executor)
            ).join();
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 26. Executor Lifecycle ⭐⭐⭐⭐⭐

Custom executors must have a lifecycle.

```java
executor.shutdown();
```

means:

```text
stop accepting new tasks
allow submitted tasks to finish
```

In an application-managed executor, shutting it down at the wrong time can break other requests. Therefore ownership should be clear.

---

# 27. Practice — Graceful Executor Shutdown ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ExecutorShutdownDemo {

    public static void main(String[] args)
            throws InterruptedException {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        executor.submit(() -> System.out.println("Task running"));
        executor.shutdown();

        if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
            executor.shutdownNow();
        }
    }
}
```

---

# 28. Do Not Create an Executor Per Request 🚨

Bad pattern:

```java
public void handleRequest() {
    ExecutorService executor =
            Executors.newFixedThreadPool(10);
}
```

Repeated creation can cause:

```text
thread explosion
memory pressure
context-switching overhead
poor shutdown management
```

Prefer controlled, application-owned executor lifecycles when custom executors are needed.

---

# 29. Practice — Identify the Bad Design ⭐⭐⭐⭐⭐

### Bad

```java
CompletableFuture<String> process() {
    ExecutorService executor =
            Executors.newFixedThreadPool(10);

    return CompletableFuture.supplyAsync(
            () -> "done", executor);
}
```

### Better concept

```text
Application / component owns executor
        ↓
requests reuse executor
        ↓
application lifecycle shuts executor down
```

---

# 30. Async Stage Placement ⭐⭐⭐⭐⭐

Suppose:

```java
future
    .thenApply(a)
    .thenApplyAsync(b, executor)
    .thenApply(c);
```

Think in terms of **stage execution**, not a single thread for the whole chain.

```text
Stage A → non-async continuation
Stage B → async continuation on supplied executor
Stage C → another non-async continuation
```

The thread executing the chain can change between stages.

---

# 31. Practice — Trace the Chain ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class AsyncStageTraceDemo {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        try {
            CompletableFuture<String> future =
                    CompletableFuture.supplyAsync(() -> {
                        print("supplyAsync");
                        return "java";
                    })
                    .thenApply(value -> {
                        print("thenApply");
                        return value.toUpperCase();
                    })
                    .thenApplyAsync(value -> {
                        print("thenApplyAsync");
                        return value + "-8";
                    }, executor)
                    .thenApply(value -> {
                        print("final thenApply");
                        return value + "-CONCURRENCY";
                    });

            System.out.println(future.join());
        } finally {
            executor.shutdown();
        }
    }

    static void print(String stage) {
        System.out.println(stage + " -> " +
                Thread.currentThread().getName());
    }
}
```

---

# 32. Exception Handling + Custom Executor ⭐⭐⭐⭐⭐

Executor choice and exception handling are separate concerns.

You can combine them:

```java
CompletableFuture.supplyAsync(task, executor)
        .thenApplyAsync(transform, executor)
        .exceptionally(ex -> fallback());
```

The executor controls async execution; the recovery stage controls failure semantics.

---

# 33. Practice — Custom Executor + Recovery ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorRecoveryDemo {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        try {
            CompletableFuture<String> future =
                    CompletableFuture.supplyAsync(() -> {
                        throw new RuntimeException("Service failed");
                    }, executor)
                    .thenApplyAsync(String::toUpperCase, executor)
                    .exceptionally(ex -> "FALLBACK");

            System.out.println(future.join());
        } finally {
            executor.shutdown();
        }
    }
}
```

The `thenApplyAsync()` stage is skipped because the previous stage failed; `exceptionally()` handles the failure.

---

# 34. Executor Choice for CPU vs Blocking Work ⭐⭐⭐⭐⭐

A useful interview-level distinction:

### CPU-bound

```text
computation
parsing
transformation
aggregation
```

Often needs a pool sized around available CPU capacity.

### Blocking I/O

```text
database
HTTP call
file/network wait
```

Can occupy workers while they wait, so executor sizing and isolation require different reasoning.

Do not use one universal pool-size formula without measuring workload and understanding the actual blocking behavior.

---

# 35. Production Anti-Patterns 🚨

```text
❌ Create executor for every request
❌ Never shut down manually owned executors
❌ Put blocking work on an unsuitable shared pool
❌ Assume Async means brand-new thread
❌ Assume thenApply always runs on the same thread
❌ Use unlimited queues without understanding memory/backpressure
❌ Use one giant executor for unrelated latency-sensitive workloads
❌ Choose pool size without measuring workload
```

---

# 36. Senior-Level Scenario 🏆

### Question

> Your Spring Boot service calls three external APIs. Some calls are blocking, while CPU-heavy JSON transformation happens afterward. How would you think about executor selection?

### Strong answer

```text
1. Identify blocking vs CPU-bound work.
2. Avoid accidentally consuming a shared CPU-oriented executor with long blocking calls.
3. Consider dedicated/bounded executors for blocking workloads.
4. Use async continuations with explicit executors where isolation is required.
5. Bound queues and define rejection/backpressure behavior where appropriate.
6. Give workers meaningful names and expose metrics.
7. Own and shut down the executors according to application lifecycle.
8. Measure queue depth, active threads, latency, saturation and failures before tuning.
```

The goal is **controlled concurrency**, not simply "more threads".

---

# 37. Interview Questions ⭐⭐⭐⭐⭐

### Q1. `thenApply()` vs `thenApplyAsync()`?

> `thenApply()` is a non-async continuation whose execution may occur in the thread completing the previous stage. `thenApplyAsync()` schedules the continuation through an async executor, using the default async executor unless one is supplied.

### Q2. Why pass a custom executor?

> To control execution resources, isolate workloads, provide capacity limits, improve observability, and avoid inappropriate sharing of a common executor.

### Q3. Does `thenApplyAsync()` always create a new thread?

> No. It submits work to an executor. The executor may reuse existing worker threads.

### Q4. Can different stages use different executors?

> Yes. For example, an I/O stage can use one executor and a CPU-heavy transformation can use another, if that separation is justified.

### Q5. Why is blocking work dangerous on a small pool?

> Blocking occupies worker threads while they wait, reducing available capacity and potentially causing queue growth and latency.

### Q6. Should every service create its own executor?

> Not automatically. Executor ownership should be based on isolation and operational requirements. Too many pools can also create resource waste and complexity.

### Q7. What is the difference between executor and thread?

> A thread is an execution resource. An executor manages how tasks are submitted and executed using worker threads.

---

# 38. 2-Minute Interview Answer 🏆

> **"In `CompletableFuture`, I use the non-async methods such as `thenApply` when a continuation can run as part of completion processing, while `thenApplyAsync` schedules the continuation through an async executor. If I don't provide an executor, the default async execution mechanism is used. In production, I may provide a custom executor to control capacity, isolate workloads, name and monitor threads, and prevent unrelated workloads from competing for the same execution resources. I also distinguish CPU-bound work from blocking I/O and avoid putting large amounts of blocking work on an unsuitable shared pool. Custom executors need clear ownership, bounded capacity where appropriate, and graceful shutdown. The important idea is controlled execution and workload isolation, not simply creating more threads."**

---

# 39. Quick Revision 🧠

```text
thenApply
→ non-async continuation

thenApplyAsync
→ async continuation

thenApplyAsync(..., executor)
→ async continuation on supplied executor

supplyAsync(task, executor)
→ initial async task on supplied executor

thenComposeAsync
→ async dependent composition

thenCombineAsync
→ async combination stage

runAsync
→ async task without result

Custom Executor
→ control + isolation + observability + capacity
```

### Golden Rule

> **Async does not mean "new thread". It means asynchronous execution through an executor. If execution resources matter, provide the appropriate executor explicitly.**

---

# 40. 💻 Practice Checklist

- [ ] Compare `thenApply()` and `thenApplyAsync()` thread names
- [ ] Use default async execution
- [ ] Supply a custom executor
- [ ] Use custom executor with `supplyAsync()`
- [ ] Use two executors for separate workloads
- [ ] Practice `thenComposeAsync()`
- [ ] Practice `thenCombineAsync()`
- [ ] Practice `runAsync()` with executor
- [ ] Practice `thenAcceptAsync()`
- [ ] Practice `thenRunAsync()`
- [ ] Simulate blocking work
- [ ] Build a bounded `ThreadPoolExecutor`
- [ ] Use `CallerRunsPolicy`
- [ ] Create named worker threads
- [ ] Gracefully shut down an executor
- [ ] Identify executor-per-request anti-pattern
- [ ] Trace threads across an async chain
- [ ] Combine custom executor with exception recovery
- [ ] Design CPU vs blocking workload isolation
- [ ] Answer senior-level executor scenario aloud
- [ ] Give the 2-minute interview answer

---

## Navigation

[← 8.32 — Exception Handling in `CompletableFuture`](../32-Exception-Handling-in-CompletableFuture/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.34 — `allOf()` / `anyOf()`**