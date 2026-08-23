# 8.10 — Custom `ThreadFactory`

> **Goal:** Learn how to create threads with controlled names, daemon status, priority, uncaught-exception handling and consistent thread configuration when using executors.

---

## 1. What is `ThreadFactory`? ⭐⭐⭐⭐⭐

`ThreadFactory` is a functional interface used to create new threads.

```java
public interface ThreadFactory {
    Thread newThread(Runnable r);
}
```

Instead of letting an executor create threads with its default settings, we can provide a custom factory:

```text
Executor
   ↓
ThreadFactory
   ↓
new Thread(...)
   ↓
Configured worker thread
```

### Memory Trick

> **ThreadFactory = centralized thread creation policy.**

---

# 2. Why Use a Custom `ThreadFactory`? ⭐⭐⭐⭐⭐

A custom factory can standardize:

- Thread names
- Thread group
- Daemon status
- Priority
- `UncaughtExceptionHandler`
- Thread numbering
- Consistent thread creation rules

This is especially useful for **production observability and debugging**.

---

# 3. Basic Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class BasicThreadFactoryExample {

    public static void main(String[] args) {

        ThreadFactory factory = runnable -> {
            Thread thread = new Thread(runnable);
            thread.setName("payment-worker");
            return thread;
        };

        ExecutorService executor = Executors.newFixedThreadPool(2, factory);

        executor.submit(() ->
                System.out.println("Running on " + Thread.currentThread().getName()));

        executor.shutdown();
    }
}
```

Output will contain a custom name such as:

```text
Running on payment-worker
```

---

# 4. Why Thread Naming Matters ⭐⭐⭐⭐⭐

Default names can be difficult to identify in logs:

```text
pool-1-thread-1
pool-1-thread-2
```

Custom names provide context:

```text
payment-worker-1
payment-worker-2
notification-worker-1
```

This makes thread dumps and production logs much easier to understand.

---

# 5. Production-Style Numbered Factory ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class NumberedThreadFactory implements ThreadFactory {

    private final AtomicInteger counter = new AtomicInteger(1);
    private final String prefix;

    public NumberedThreadFactory(String prefix) {
        this.prefix = prefix;
    }

    @Override
    public Thread newThread(Runnable runnable) {
        Thread thread = new Thread(runnable);
        thread.setName(prefix + "-" + counter.getAndIncrement());
        return thread;
    }
}
```

Usage:

```java
ExecutorService executor = Executors.newFixedThreadPool(
        3,
        new NumberedThreadFactory("payment-worker")
);

for (int i = 1; i <= 5; i++) {
    int taskId = i;
    executor.execute(() ->
            System.out.println("Task " + taskId + " -> "
                    + Thread.currentThread().getName()));
}

executor.shutdown();
```

Possible output:

```text
Task 1 -> payment-worker-1
Task 2 -> payment-worker-2
Task 3 -> payment-worker-3
Task 4 -> payment-worker-1
Task 5 -> payment-worker-2
```

> The exact task-to-thread ordering is not guaranteed.

---

# 6. Why `AtomicInteger`? ⭐⭐⭐⭐

Thread factories may create threads concurrently.

Using:

```java
AtomicInteger
```

makes the thread-number generation safe when multiple calls to `newThread()` happen concurrently.

Do not use a plain shared `int` counter without considering concurrent access.

---

# 7. Custom `UncaughtExceptionHandler` ⭐⭐⭐⭐⭐

A custom factory can install an exception handler for every created thread.

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ExceptionHandlingThreadFactory implements ThreadFactory {

    private final AtomicInteger counter = new AtomicInteger(1);

    @Override
    public Thread newThread(Runnable runnable) {
        Thread thread = new Thread(runnable);

        thread.setName("worker-" + counter.getAndIncrement());

        thread.setUncaughtExceptionHandler((t, e) ->
                System.out.println(
                        "Uncaught exception in "
                                + t.getName()
                                + ": "
                                + e.getMessage()));

        return thread;
    }
}
```

### Important executor nuance ⭐⭐⭐⭐⭐

If you use:

```java
executor.execute(task);
```

an uncaught exception from the task can reach the thread's uncaught-exception handler.

But with:

```java
executor.submit(task);
```

exceptions are captured by the returned `Future` and should be inspected using `Future.get()`.

Therefore, do not assume a thread-level uncaught handler is the complete exception-handling strategy for executor tasks.

---

# 8. `execute()` vs `submit()` Practice ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class ExecuteVsSubmitExceptionExample {

    public static void main(String[] args) throws Exception {

        ThreadFactory factory = runnable -> {
            Thread thread = new Thread(runnable);
            thread.setName("custom-worker");
            thread.setUncaughtExceptionHandler((t, e) ->
                    System.out.println("Handler saw: " + e.getMessage()));
            return thread;
        };

        ExecutorService executor = Executors.newSingleThreadExecutor(factory);

        executor.execute(() -> {
            throw new RuntimeException("execute failure");
        });

        Future<?> future = executor.submit(() -> {
            throw new RuntimeException("submit failure");
        });

        try {
            future.get();
        } catch (ExecutionException e) {
            System.out.println("Future captured: "
                    + e.getCause().getMessage());
        }

        executor.shutdown();
    }
}
```

### Interview point

> `ThreadFactory` configures how the thread is created; it does not replace proper task-level error handling.

---

# 9. Daemon Threads ⭐⭐⭐⭐

A factory can create daemon workers:

```java
Thread thread = new Thread(runnable);
thread.setDaemon(true);
return thread;
```

### Important

The JVM does not keep running solely because daemon threads exist.

```text
Non-daemon threads finish
        ↓
No non-daemon threads remain
        ↓
JVM may terminate
        ↓
Daemon work may be stopped
```

### Practice Code

```java
ThreadFactory daemonFactory = runnable -> {
    Thread thread = new Thread(runnable);
    thread.setName("background-worker");
    thread.setDaemon(true);
    return thread;
};
```

### Warning ⚠️

Do not use daemon threads for work that must complete, such as critical persistence or payment processing.

---

# 10. Thread Priority ⭐⭐⭐

A factory can configure priority:

```java
Thread thread = new Thread(runnable);
thread.setPriority(Thread.NORM_PRIORITY);
```

Available constants include:

```java
Thread.MIN_PRIORITY
Thread.NORM_PRIORITY
Thread.MAX_PRIORITY
```

### Interview nuance

Thread priority is a scheduler hint and should not be treated as a reliable business-priority mechanism.

Do not design correctness around priority ordering.

---

# 11. Thread Group ⭐⭐

A thread can also belong to a `ThreadGroup`:

```java
ThreadGroup group = new ThreadGroup("payment-workers");

ThreadFactory factory = runnable ->
        new Thread(group, runnable);
```

For modern application-level concurrency, thread names and structured observability are generally more useful than relying heavily on `ThreadGroup`.

---

# 12. Full Custom Factory ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

public class ProductionThreadFactory implements ThreadFactory {

    private final AtomicInteger counter = new AtomicInteger(1);
    private final String prefix;
    private final boolean daemon;
    private final Thread.UncaughtExceptionHandler exceptionHandler;

    public ProductionThreadFactory(
            String prefix,
            boolean daemon,
            Thread.UncaughtExceptionHandler exceptionHandler) {

        this.prefix = prefix;
        this.daemon = daemon;
        this.exceptionHandler = exceptionHandler;
    }

    @Override
    public Thread newThread(Runnable runnable) {

        Thread thread = new Thread(runnable);

        thread.setName(prefix + "-" + counter.getAndIncrement());
        thread.setDaemon(daemon);
        thread.setPriority(Thread.NORM_PRIORITY);

        if (exceptionHandler != null) {
            thread.setUncaughtExceptionHandler(exceptionHandler);
        }

        return thread;
    }
}
```

Usage:

```java
Thread.UncaughtExceptionHandler handler = (thread, exception) ->
        System.out.println("Unhandled error in "
                + thread.getName()
                + ": "
                + exception.getMessage());

ThreadFactory factory = new ProductionThreadFactory(
        "order-worker",
        false,
        handler
);

ExecutorService executor = Executors.newFixedThreadPool(3, factory);

executor.execute(() ->
        System.out.println(Thread.currentThread().getName()));

executor.shutdown();
```

---

# 13. `ThreadFactory` with `ThreadPoolExecutor` ⭐⭐⭐⭐⭐

`ThreadFactory` is not limited to `Executors` factory methods.

```java
import java.util.concurrent.*;

public class ThreadPoolExecutorFactoryExample {

    public static void main(String[] args) {

        ThreadFactory factory = runnable -> {
            Thread thread = new Thread(runnable);
            thread.setName("api-worker-" + thread.getId());
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

        for (int i = 1; i <= 5; i++) {
            int taskId = i;
            executor.execute(() ->
                    System.out.println("Task " + taskId
                            + " on "
                            + Thread.currentThread().getName()));
        }

        executor.shutdown();
    }
}
```

### Remember the architecture

```text
ThreadPoolExecutor
       ↓
 ThreadFactory
       ↓
Configured Thread
       ↓
 Executes Runnable
```

---

# 14. Default vs Custom ThreadFactory ⭐⭐⭐⭐⭐

| Feature | Default factory | Custom factory |
|---|---|---|
| Thread creation | ✅ | ✅ |
| Custom names | Limited/default | ✅ |
| Naming convention | Generic | Application-specific |
| Daemon policy | Default | Configurable |
| Priority | Default | Configurable |
| Exception handler | Default behavior | Configurable |
| Centralized policy | ❌ | ✅ |
| Observability | Basic | Better |

---

# 15. ThreadFactory vs Executor ⭐⭐⭐⭐⭐

These solve different problems.

### `ThreadFactory`

Answers:

> **How should a new thread be created?**

### `ExecutorService`

Answers:

> **How should tasks be scheduled and executed?**

```text
ThreadFactory
     ↓
creates/configures threads

ExecutorService
     ↓
manages task execution
```

---

# 16. ThreadFactory vs Runnable vs Callable ⭐⭐⭐⭐

| Component | Responsibility |
|---|---|
| `Runnable` | Defines task |
| `Callable` | Defines task + return value/exception |
| `ThreadFactory` | Defines how worker thread is created |
| `ExecutorService` | Manages task execution |

### Memory Trick

```text
Runnable/Callable → WHAT work?
ThreadFactory     → WHAT thread?
ExecutorService   → HOW/WHERE schedule work?
```

---

# 17. Production Scenario — Payment Workers ⭐⭐⭐⭐⭐

Suppose an application has:

```text
Payment API
    ↓
Payment Executor
    ↓
Payment Worker Threads
```

Use a custom factory:

```text
payment-worker-1
payment-worker-2
payment-worker-3
```

Now logs and thread dumps immediately reveal which worker pool is involved.

Example:

```text
2026-08-23 ERROR [payment-worker-2]
Payment processing timeout
```

This is much more useful than:

```text
pool-3-thread-2
```

---

# 18. Common Mistakes ❌

### Mistake 1 — Creating a new thread for every task

`ThreadFactory` does not mean one permanent thread per task.

The executor controls worker reuse.

### Mistake 2 — Making all workers daemon threads

Critical work can be terminated when the JVM exits.

### Mistake 3 — Depending on thread priority for correctness

Priority is not a reliable synchronization mechanism.

### Mistake 4 — Assuming uncaught handler catches every executor exception

`submit()` captures task exceptions in `Future`.

### Mistake 5 — Sharing an unsafe counter

Thread creation can be concurrent; use `AtomicInteger` or another safe mechanism for numbering.

### Mistake 6 — Over-customizing thread properties

Keep the factory focused on useful operational policies such as naming and exception handling.

---

# 19. Practice — Factory + Monitoring ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class CustomFactoryPractice {

    static class NamedFactory implements ThreadFactory {
        private final AtomicInteger count = new AtomicInteger(1);

        @Override
        public Thread newThread(Runnable task) {
            Thread thread = new Thread(task);
            thread.setName("report-worker-" + count.getAndIncrement());
            thread.setUncaughtExceptionHandler((t, e) ->
                    System.out.println("ERROR [" + t.getName() + "] "
                            + e.getMessage()));
            return thread;
        }
    }

    public static void main(String[] args) throws InterruptedException {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,
                4,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(5),
                new NamedFactory(),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        for (int i = 1; i <= 10; i++) {
            int taskId = i;

            executor.execute(() -> {
                System.out.println("Task " + taskId
                        + " -> "
                        + Thread.currentThread().getName());
                sleep(300);
            });
        }

        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
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

# 20. Interview Scenarios ⭐⭐⭐⭐⭐

### Q1. What is `ThreadFactory`?

> `ThreadFactory` is an interface used by executors to create new threads according to a custom policy.

### Q2. Why use a custom `ThreadFactory`?

> Mainly to standardize thread creation, especially naming, daemon status, priority and uncaught-exception handling.

### Q3. Why are custom thread names useful?

> They improve observability, debugging, log analysis and thread-dump investigation.

### Q4. Can `ThreadFactory` control the thread pool size?

> No. Pool sizing belongs to the executor, such as `ThreadPoolExecutor`. The factory controls thread creation properties.

### Q5. What is the relationship between `ThreadFactory` and `ThreadPoolExecutor`?

> When the executor needs a new worker thread, it uses the configured `ThreadFactory` to create that thread.

### Q6. Why use `AtomicInteger` for thread names?

> Because thread creation can happen concurrently and the counter must be safely incremented.

### Q7. Does `ThreadFactory` handle task exceptions?

> It can configure an uncaught exception handler for created threads, but executor APIs can capture task exceptions differently; for example, `submit()` exposes them through `Future`.

### Q8. Should production workers be daemon threads?

> Usually not when they perform work that must complete reliably. Daemon threads do not keep the JVM alive.

### Q9. Is thread priority a reliable way to prioritize business tasks?

> No. Priority is scheduler-related and should not be used as a business correctness mechanism.

### Q10. `Runnable` vs `ThreadFactory`?

> `Runnable` defines the work; `ThreadFactory` defines how the worker thread executing that work is created.

---

# 21. Quick Revision

```text
ThreadFactory
      ↓
newThread(Runnable)
      ↓
Configured Thread
```

### Common customizations

```text
Name
Daemon
Priority
Exception Handler
Thread Group
Counter
```

### Core interview distinction

```text
Runnable / Callable
       ↓
      TASK

ThreadFactory
       ↓
     THREAD

ExecutorService
       ↓
  TASK EXECUTION
```

---

# 🎯 Interview Questions

1. What is `ThreadFactory`?
2. Why would you create a custom `ThreadFactory`?
3. How do you assign custom thread names?
4. Why use `AtomicInteger` in a thread factory?
5. How do you configure an `UncaughtExceptionHandler`?
6. `execute()` vs `submit()` with exceptions?
7. Can `ThreadFactory` control pool size?
8. What is a daemon thread?
9. Why can daemon threads be dangerous for critical work?
10. Is thread priority reliable?
11. `ThreadFactory` vs `ExecutorService`?
12. `ThreadFactory` vs `Runnable`?
13. How does `ThreadPoolExecutor` use `ThreadFactory`?
14. How would you name payment-service worker threads?
15. How would you debug a production thread dump using custom names?
16. Design a production-ready custom thread factory.
17. Why should thread configuration be centralized?
18. What happens if `ThreadFactory.newThread()` returns `null`?
19. How would you handle thread creation failure?
20. Explain `ThreadFactory` in 2 minutes.

---

# 🏆 2-Minute Interview Answer

> **"`ThreadFactory` is an interface used by executor implementations to create worker threads. A custom factory lets us centralize thread-creation policies such as meaningful names, daemon status, priority and uncaught-exception handling. In production, custom naming is especially useful for logs and thread dumps—for example, `payment-worker-1`. I commonly use an `AtomicInteger` to generate unique names safely. The important distinction is that `Runnable` or `Callable` defines the task, `ThreadFactory` defines how a thread is created, and `ExecutorService` manages task execution. I also need to remember that exceptions submitted through `submit()` are captured by the returned `Future`, so a thread-level uncaught exception handler is not a replacement for proper task error handling."**

---

# 💻 Practice Checklist

- [ ] Create a basic `ThreadFactory`.
- [ ] Create numbered thread names.
- [ ] Use `AtomicInteger` for thread numbering.
- [ ] Configure daemon status.
- [ ] Configure thread priority.
- [ ] Configure `UncaughtExceptionHandler`.
- [ ] Compare `execute()` and `submit()` exception behavior.
- [ ] Use a custom factory with `ExecutorService`.
- [ ] Use a custom factory with `ThreadPoolExecutor`.
- [ ] Build a production-style named worker factory.
- [ ] Simulate a payment-worker pool.
- [ ] Inspect logs using thread names.
- [ ] Explain `Runnable` vs `ThreadFactory` vs `ExecutorService`.
- [ ] Explain the topic in under 2 minutes.

---

## Navigation

[← 8.9 — Queueing, Rejection Policies & Backpressure](../09-Queueing-Rejection-Policies-and-Backpressure/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.11 — `CountDownLatch`**