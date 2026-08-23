# 8.4 — `shutdown()` vs `shutdownNow()`

> **Goal:** Understand executor shutdown semantics, graceful termination, interruption, task states, and how to shut down a thread pool safely in production.

---

## 1. Core Difference ⭐⭐⭐⭐⭐

```text
shutdown()
   ↓
Stop accepting NEW tasks
   ↓
Previously submitted tasks may continue
   ↓
Graceful shutdown

shutdownNow()
   ↓
Stop accepting NEW tasks
   ↓
Attempt to interrupt RUNNING tasks
   ↓
Return tasks that never started
   ↓
Forceful shutdown attempt
```

### One-line interview answer

> **`shutdown()` performs an orderly shutdown: no new tasks are accepted and previously submitted tasks are allowed to complete. `shutdownNow()` attempts a more immediate shutdown by interrupting running worker threads and returning tasks that never started; interruption is cooperative, so it does not guarantee immediate termination.**

---

# 2. `shutdown()`

```java
executor.shutdown();
```

After `shutdown()`:

- New task submissions are rejected.
- Previously submitted tasks can continue.
- The executor eventually terminates after eligible tasks complete.

## Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ShutdownExample {
    public static void main(String[] args) throws InterruptedException {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        executor.execute(() -> {
            try {
                Thread.sleep(1000);
                System.out.println("Task 1 completed");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        executor.execute(() -> {
            try {
                Thread.sleep(1000);
                System.out.println("Task 2 completed");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        executor.shutdown();

        System.out.println("Shutdown requested");

        // executor.execute(() -> System.out.println("Rejected"));
        // This would be rejected after shutdown().

        if (executor.awaitTermination(2,
                java.util.concurrent.TimeUnit.SECONDS)) {
            System.out.println("Executor terminated");
        }
    }
}
```

### Key observation

The call to `shutdown()` does **not** mean "kill the running tasks now".

It means:

> **Finish what has already been accepted, but don't accept anything new.**

---

# 3. `shutdownNow()`

```java
List<Runnable> notStarted = executor.shutdownNow();
```

`shutdownNow()`:

1. Stops accepting new tasks.
2. Attempts to interrupt worker threads.
3. Returns tasks that were awaiting execution and never started.

## Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ShutdownNowExample {
    public static void main(String[] args) throws InterruptedException {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        for (int i = 1; i <= 6; i++) {
            int taskId = i;

            executor.execute(() -> {
                try {
                    System.out.println("Task " + taskId + " started");
                    Thread.sleep(3000);
                    System.out.println("Task " + taskId + " completed");
                } catch (InterruptedException e) {
                    System.out.println("Task " + taskId + " interrupted");
                    Thread.currentThread().interrupt();
                }
            });
        }

        Thread.sleep(500);

        List<Runnable> notStarted = executor.shutdownNow();

        System.out.println("Tasks never started: "
                + notStarted.size());
    }
}
```

Because the pool has only two workers, some submitted tasks may still be waiting when `shutdownNow()` is called. Those not-started tasks can be returned by the method.

The exact number can vary because scheduling is concurrent.

---

# 4. Important: `shutdownNow()` Does NOT "Kill" Threads ❗

This is one of the most important interview points.

```java
executor.shutdownNow();
```

does **not** forcibly terminate arbitrary Java code.

It generally works by interrupting worker threads.

```text
shutdownNow()
      ↓
interrupt worker thread
      ↓
Task must respond to interruption
      ↓
Task exits / cleans up
```

If a task ignores interruption, it may continue running.

---

# 5. Interruption-Aware Task ⭐⭐⭐⭐⭐

```java
executor.execute(() -> {
    try {
        while (true) {
            System.out.println("Working...");
            Thread.sleep(200);
        }
    } catch (InterruptedException e) {
        System.out.println("Interrupted - stopping task");
        Thread.currentThread().interrupt();
    }
});
```

This task responds to interruption because `sleep()` throws `InterruptedException`.

### Important rule

When catching `InterruptedException`, code should normally either:

- propagate the exception, or
- restore the interrupted status with `Thread.currentThread().interrupt()` after performing appropriate cleanup.

---

# 6. Task That Ignores Interruption ❌

```java
executor.execute(() -> {
    while (true) {
        // Ignores interruption.
    }
});
```

Calling:

```java
executor.shutdownNow();
```

does not guarantee that this task stops.

This is why `shutdownNow()` should be described as a **shutdown attempt**, not a guaranteed kill operation.

---

# 7. `shutdown()` + `awaitTermination()` ⭐⭐⭐⭐⭐

A common graceful shutdown pattern is:

```java
executor.shutdown();

try {
    if (!executor.awaitTermination(10,
            java.util.concurrent.TimeUnit.SECONDS)) {

        executor.shutdownNow();
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

### Flow

```text
shutdown()
   ↓
Wait for normal completion
   ↓
Within timeout?
   ├── YES → terminated normally
   └── NO  → shutdownNow() attempt
```

This is a common pattern when an application wants to give tasks a reasonable grace period before attempting interruption.

---

# 8. Complete Graceful Shutdown Utility ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.TimeUnit;

public class ExecutorShutdownUtil {

    public static void shutdownAndAwaitTermination(
            ExecutorService executor) {

        executor.shutdown();

        try {
            if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                executor.shutdownNow();

                if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                    System.err.println("Executor did not terminate");
                }
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

### Why restore interruption?

The current thread's interrupted status is cleared when `InterruptedException` is thrown. Restoring it preserves the cancellation/interruption signal for higher-level code.

---

# 9. `isShutdown()` vs `isTerminated()`

These methods are frequently confused.

```java
executor.isShutdown();
executor.isTerminated();
```

### `isShutdown()`

Returns `true` once shutdown has been initiated.

### `isTerminated()`

Returns `true` only after shutdown has completed and all tasks have finished/terminated according to executor semantics.

Conceptually:

```text
Running
  ↓
shutdown()
  ↓
isShutdown() = true
  ↓
Tasks finish
  ↓
isTerminated() = true
```

---

# 10. Practice — Observe Both States

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorStatePractice {
    public static void main(String[] args) throws InterruptedException {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        executor.execute(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        executor.shutdown();

        System.out.println("isShutdown = "
                + executor.isShutdown());

        System.out.println("isTerminated = "
                + executor.isTerminated());

        executor.awaitTermination(2,
                java.util.concurrent.TimeUnit.SECONDS);

        System.out.println("isTerminated = "
                + executor.isTerminated());
    }
}
```

You should generally observe:

```text
isShutdown = true
isTerminated = false
...
isTerminated = true
```

The exact timing depends on scheduling and task duration.

---

# 11. Tasks Returned by `shutdownNow()`

Example:

```java
List<Runnable> pendingTasks = executor.shutdownNow();
```

These are tasks that were submitted but **never commenced execution** at the time of the shutdown attempt.

They are not automatically executed elsewhere.

You can inspect them if the application needs special handling:

```java
for (Runnable task : pendingTasks) {
    System.out.println("Never started: " + task);
}
```

Be careful about retrying them blindly because tasks may have side effects or may no longer be valid.

---

# 12. `shutdown()` vs `shutdownNow()`

| Feature | `shutdown()` | `shutdownNow()` |
|---|---|---|
| Accept new tasks | ❌ | ❌ |
| Previously submitted tasks | Allowed to complete | Running tasks are interrupted as an attempt |
| Attempts interruption | ❌ | ✅ |
| Returns not-started tasks | ❌ | ✅ |
| Graceful | ✅ | More forceful attempt |
| Guarantees immediate stop | ❌ | ❌ |
| Typical use | Normal application shutdown | Timeout/emergency escalation |

---

# 13. Real-World Scenario ⭐⭐⭐⭐⭐

Imagine a Spring Boot application is shutting down.

### Normal path

```text
Application shutdown signal
        ↓
Stop accepting new work
        ↓
executor.shutdown()
        ↓
Allow in-flight tasks to finish
        ↓
awaitTermination()
        ↓
Application exits
```

### Timeout path

```text
shutdown()
    ↓
Grace period expires
    ↓
shutdownNow()
    ↓
Interrupt workers
    ↓
Cleanup / exit
```

The exact lifecycle should be designed around the application's work and shutdown requirements.

---

# 14. Common Mistakes ❌

### Mistake 1 — Using `shutdownNow()` as a guaranteed kill

It is an interruption request, not a hard kill.

### Mistake 2 — Calling `shutdown()` and immediately assuming termination

```java
executor.shutdown();
// Executor may still be running tasks here.
```

Use `awaitTermination()` when you need to wait for termination.

### Mistake 3 — Swallowing interruption

Bad:

```java
catch (InterruptedException e) {
    // ignore
}
```

Better:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Mistake 4 — Re-submitting every task returned by `shutdownNow()` automatically

The task may be unsafe to retry or may have become stale.

---

# 15. Practice Challenge ⭐⭐⭐⭐⭐

Create a program with:

- Fixed pool of 3 workers
- 10 tasks
- Each task sleeps for 2 seconds
- Start all tasks
- After 500 ms call `shutdown()`
- Observe how many tasks finish
- Repeat using `shutdownNow()`
- Record the number of tasks returned by `shutdownNow()`
- Explain why the exact number may vary

### Extension

Create two task types:

1. Interruption-aware task
2. Task that ignores interruption

Call `shutdownNow()` and compare their behavior.

---

# 16. Interview Scenarios

### Scenario 1

> The application is shutting down normally. What should you use?

**Answer:** Usually `shutdown()` followed by an appropriate `awaitTermination()` strategy.

### Scenario 2

> Tasks did not finish within the graceful shutdown timeout.

**Answer:** Consider `shutdownNow()` as an escalation, while understanding that interruption is cooperative.

### Scenario 3

> Does `shutdownNow()` guarantee that every task stops immediately?

**Answer:** No.

### Scenario 4

> What happens to queued tasks after `shutdownNow()`?

**Answer:** Tasks that never started are returned as a `List<Runnable>` and are not automatically executed.

### Scenario 5

> Why restore the interrupt flag after catching `InterruptedException`?

**Answer:** To preserve the interruption signal for higher-level code when the method cannot propagate the checked exception.

---

# 17. Quick Revision

```text
shutdown()
   ↓
Graceful
   ↓
No new tasks
   ↓
Existing tasks finish
   ↓
awaitTermination()

shutdownNow()
   ↓
No new tasks
   ↓
Interrupt running workers
   ↓
Return tasks never started
   ↓
Task must cooperate with interruption
```

### Memory Trick

> **shutdown = finish accepted work**  
> **shutdownNow = attempt interruption now**

---

# 🎯 Interview Questions

1. What is the difference between `shutdown()` and `shutdownNow()`?
2. Does `shutdown()` stop currently running tasks?
3. Does `shutdownNow()` forcibly kill running tasks?
4. What does `shutdownNow()` return?
5. What is `awaitTermination()`?
6. Difference between `isShutdown()` and `isTerminated()`?
7. Why is interruption cooperative?
8. What happens when a task ignores interruption?
9. Why should `InterruptedException` generally not be swallowed?
10. Why restore interrupt status?
11. When would you use graceful shutdown?
12. When would you escalate to `shutdownNow()`?
13. What happens to tasks that never started?
14. Can tasks returned by `shutdownNow()` be safely retried?
15. Design a production shutdown sequence for an executor.

---

# 🏆 2-Minute Interview Answer

> **"`shutdown()` is the graceful option. It stops accepting new tasks but allows already submitted tasks to complete. We can use `awaitTermination()` to wait for a bounded period. If the executor doesn't terminate within that grace period, we can call `shutdownNow()`. `shutdownNow()` stops accepting new tasks, attempts to interrupt running worker threads and returns tasks that never started. It does not guarantee immediate termination because Java interruption is cooperative. Well-designed tasks should respond to interruption, clean up resources and preserve the interrupt status when appropriate."**

---

# 💻 Practice Checklist

- [ ] Run `shutdown()` example.
- [ ] Run `shutdownNow()` example.
- [ ] Observe queued tasks.
- [ ] Practice interruption-aware tasks.
- [ ] Practice `awaitTermination()`.
- [ ] Compare `isShutdown()` and `isTerminated()`.
- [ ] Inspect tasks returned by `shutdownNow()`.
- [ ] Practice graceful → forced shutdown escalation.
- [ ] Test a task that ignores interruption.
- [ ] Explain shutdown semantics in under 2 minutes.

---

## Navigation

[← 8.3 — `execute()` vs `submit()`](../03-execute-vs-submit/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.5 — `Future` and `Callable`**