# 8.6 — `ScheduledExecutorService`

> **Goal:** Understand how Java schedules tasks for delayed or periodic execution using `ScheduledExecutorService`.

---

## 1. Core Idea ⭐⭐⭐⭐⭐

`ScheduledExecutorService` is an `ExecutorService` designed for:

- Running a task after a delay
- Running a task repeatedly with a fixed rate
- Running a task repeatedly with a fixed delay between executions

```text
ScheduledExecutorService
        │
        ├── schedule()
        │      └── run once after delay
        │
        ├── scheduleAtFixedRate()
        │      └── periodic execution based on a fixed rate
        │
        └── scheduleWithFixedDelay()
               └── delay starts after previous execution finishes
```

### Interview definition

> **`ScheduledExecutorService` is an executor service that supports delayed and periodic task execution. It provides `schedule()`, `scheduleAtFixedRate()`, and `scheduleWithFixedDelay()` for different scheduling requirements.**

---

# 2. Creating a Scheduler

```java
ScheduledExecutorService scheduler =
        Executors.newScheduledThreadPool(2);
```

The pool can contain multiple worker threads, so different scheduled tasks can execute concurrently when needed.

---

# 3. `schedule()` — Run Once After a Delay ⭐⭐⭐⭐⭐

Use `schedule()` when a task should execute **one time** after a specified delay.

```java
import java.util.concurrent.*;

public class ScheduleOnceExample {
    public static void main(String[] args) {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(1);

        scheduler.schedule(
                () -> System.out.println("Task executed"),
                2,
                TimeUnit.SECONDS
        );

        scheduler.shutdown();
    }
}
```

### Timeline

```text
submit at t=0
     ↓
wait 2 seconds
     ↓
execute once
```

---

# 4. `schedule()` Returns a `ScheduledFuture`

```java
ScheduledFuture<?> future = scheduler.schedule(
        () -> System.out.println("Hello"),
        5,
        TimeUnit.SECONDS
);
```

Because the scheduled task has a Future-like handle, you can use operations such as:

```java
future.cancel(false);
future.isDone();
future.isCancelled();
```

---

# 5. Cancel a Delayed Task ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class CancelScheduledTaskExample {
    public static void main(String[] args) throws InterruptedException {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(1);

        ScheduledFuture<?> future = scheduler.schedule(
                () -> System.out.println("Should not run"),
                5,
                TimeUnit.SECONDS
        );

        Thread.sleep(1000);

        boolean cancelled = future.cancel(false);

        System.out.println("Cancelled = " + cancelled);

        scheduler.shutdown();
    }
}
```

Since the task was cancelled before its delay elapsed, it should not execute.

---

# 6. `scheduleAtFixedRate()` ⭐⭐⭐⭐⭐

Use it for periodic execution based on a **fixed rate**.

```java
import java.util.concurrent.*;

public class FixedRateExample {
    public static void main(String[] args) throws InterruptedException {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(1);

        ScheduledFuture<?> future = scheduler.scheduleAtFixedRate(
                () -> System.out.println(
                        "Run at " + System.currentTimeMillis()),
                0,
                2,
                TimeUnit.SECONDS
        );

        Thread.sleep(7000);

        future.cancel(false);
        scheduler.shutdown();
    }
}
```

### Meaning

```java
scheduleAtFixedRate(task, initialDelay, period, unit)
```

- `initialDelay` → time before first execution
- `period` → target interval between scheduled starts
- `unit` → time unit

---

# 7. Fixed Rate — Important Timing Concept ⭐⭐⭐⭐⭐

Suppose:

```java
scheduleAtFixedRate(task, 0, 5, TimeUnit.SECONDS);
```

The scheduler tries to maintain a schedule approximately like:

```text
0s       5s       10s       15s
│        │         │         │
Task     Task      Task      Task
```

The next execution is based on the configured periodic rate, not simply "wait 5 seconds after the previous task finishes."

### Important

For a single scheduled task, executions do not overlap with themselves. If one execution takes longer than the period, later execution is delayed rather than creating overlapping executions of that same periodic task.

---

# 8. `scheduleWithFixedDelay()` ⭐⭐⭐⭐⭐

Use it when you want a fixed delay **after the previous execution completes**.

```java
import java.util.concurrent.*;

public class FixedDelayExample {
    public static void main(String[] args) throws InterruptedException {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(1);

        ScheduledFuture<?> future = scheduler.scheduleWithFixedDelay(
                () -> {
                    System.out.println("Task started");
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                    System.out.println("Task finished");
                },
                0,
                3,
                TimeUnit.SECONDS
        );

        Thread.sleep(10000);

        future.cancel(false);
        scheduler.shutdown();
    }
}
```

### Timeline

If task execution takes 2 seconds and fixed delay is 3 seconds:

```text
Task starts
   │
   └── 2 sec execution
          ↓
       3 sec delay
          ↓
       Task starts
```

So the start-to-start interval is approximately:

```text
execution time + configured delay
```

---

# 9. `scheduleAtFixedRate()` vs `scheduleWithFixedDelay()` ⭐⭐⭐⭐⭐

| Feature | `scheduleAtFixedRate()` | `scheduleWithFixedDelay()` |
|---|---|---|
| Scheduling basis | Fixed rate | Delay after completion |
| Delay measured from | Scheduled execution times | Previous task completion |
| Useful for | Regular periodic work | Work that should pause between runs |
| Self-overlap | Same periodic task does not overlap with itself | Same periodic task does not overlap with itself |
| Long task effect | Can make later executions start late | Adds execution time + delay to start-to-start interval |

### Memory Trick

> **Fixed Rate = "every N units" target schedule**  
> **Fixed Delay = "wait N units after finishing"**

---

# 10. Practice — Compare Both ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class RateVsDelayExample {
    public static void main(String[] args) throws InterruptedException {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(2);

        ScheduledFuture<?> rateFuture = scheduler.scheduleAtFixedRate(
                () -> {
                    System.out.println("RATE start: "
                            + System.currentTimeMillis());
                    sleep(2000);
                    System.out.println("RATE end");
                },
                0,
                3,
                TimeUnit.SECONDS
        );

        ScheduledFuture<?> delayFuture = scheduler.scheduleWithFixedDelay(
                () -> {
                    System.out.println("DELAY start: "
                            + System.currentTimeMillis());
                    sleep(2000);
                    System.out.println("DELAY end");
                },
                0,
                3,
                TimeUnit.SECONDS
        );

        Thread.sleep(12000);

        rateFuture.cancel(false);
        delayFuture.cancel(false);
        scheduler.shutdown();
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

Observe the difference between the two scheduling strategies.

---

# 11. Exception Handling in Periodic Tasks ⭐⭐⭐⭐⭐

This is an important production/interview trap.

Consider:

```java
scheduler.scheduleAtFixedRate(() -> {
    throw new RuntimeException("Something failed");
}, 0, 1, TimeUnit.SECONDS);
```

If a periodic task execution terminates abruptly because of an exception, subsequent executions of that periodic task are suppressed.

Therefore, periodic task logic should handle expected exceptions appropriately.

### Safer pattern

```java
scheduler.scheduleAtFixedRate(() -> {
    try {
        performWork();
    } catch (Exception e) {
        System.err.println("Scheduled task failed: " + e.getMessage());
    }
}, 0, 1, TimeUnit.SECONDS);
```

Be deliberate about which exceptions should be caught; do not blindly hide programming errors.

---

# 12. Practice — Periodic Task with Error Handling

```java
import java.util.concurrent.*;

public class PeriodicExceptionHandlingExample {
    public static void main(String[] args) throws InterruptedException {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(1);

        ScheduledFuture<?> future = scheduler.scheduleAtFixedRate(() -> {
            try {
                System.out.println("Running scheduled work");
                riskyOperation();
            } catch (Exception e) {
                System.err.println("Handled: " + e.getMessage());
            }
        }, 0, 1, TimeUnit.SECONDS);

        Thread.sleep(5000);

        future.cancel(false);
        scheduler.shutdown();
    }

    private static void riskyOperation() {
        if (System.currentTimeMillis() % 2 == 0) {
            throw new IllegalStateException("Temporary failure");
        }
    }
}
```

---

# 13. `ScheduledFuture`

`ScheduledFuture<V>` combines scheduled-task behavior with `Future` behavior.

Useful methods include:

```java
future.cancel(false);
future.isCancelled();
future.isDone();
future.get();
```

For periodic tasks, the Future normally remains incomplete while the periodic task continues running. Cancelling the Future stops future executions.

---

# 14. `schedule()` with `Callable` ⭐⭐⭐⭐⭐

Unlike periodic scheduling methods, `schedule()` can schedule a `Callable` and return its result through a `ScheduledFuture<V>`.

```java
import java.util.concurrent.*;

public class ScheduledCallableExample {
    public static void main(String[] args) throws Exception {

        ScheduledExecutorService scheduler =
                Executors.newScheduledThreadPool(1);

        ScheduledFuture<Integer> future = scheduler.schedule(
                () -> 40 + 2,
                2,
                TimeUnit.SECONDS
        );

        System.out.println("Waiting...");
        System.out.println("Result = " + future.get());

        scheduler.shutdown();
    }
}
```

---

# 15. Scheduler Thread Pool Size ⭐⭐⭐⭐⭐

```java
Executors.newScheduledThreadPool(1);
```

A single-thread scheduler executes scheduled tasks using one worker.

With:

```java
Executors.newScheduledThreadPool(4);
```

up to four tasks can execute concurrently, subject to the executor's scheduling and worker availability.

### Important

The pool size does **not** mean one periodic task will execute concurrently with itself. A given periodic execution does not overlap with its own next execution.

---

# 16. Delayed Task vs Periodic Task

### One-time delay

```java
scheduler.schedule(task, 5, TimeUnit.SECONDS);
```

### Fixed rate

```java
scheduler.scheduleAtFixedRate(task, 5, 10, TimeUnit.SECONDS);
```

### Fixed delay

```java
scheduler.scheduleWithFixedDelay(task, 5, 10, TimeUnit.SECONDS);
```

Think:

```text
schedule()
    → once

scheduleAtFixedRate()
    → repeatedly at a target rate

scheduleWithFixedDelay()
    → repeatedly after completion + delay
```

---

# 17. Cancellation and Shutdown ⭐⭐⭐⭐⭐

A scheduled executor should be shut down when its lifecycle is complete.

```java
ScheduledExecutorService scheduler =
        Executors.newScheduledThreadPool(2);

try {
    // submit scheduled work
} finally {
    scheduler.shutdown();
}
```

For application lifecycle management, use a deliberate graceful-shutdown strategy and consider the previous topic:

[8.4 — `shutdown()` vs `shutdownNow()`](../04-shutdown-vs-shutdownNow/README.md)

---

# 18. Real-World Scenarios ⭐⭐⭐⭐⭐

### Scenario 1 — Delayed retry

```text
Request fails
    ↓
Schedule retry after delay
    ↓
Retry
```

For production retries, also consider maximum attempts, backoff, jitter and cancellation.

### Scenario 2 — Cache refresh

```text
Every N seconds
       ↓
Refresh cache
```

### Scenario 3 — Background cleanup

```text
Every N minutes
       ↓
Remove expired data
```

### Scenario 4 — Periodic health check

```text
Every N seconds
       ↓
Check dependency
       ↓
Record status / alert
```

### Scenario 5 — Delayed notification

```text
Event occurs
    ↓
Schedule notification
    ↓
Send after delay
```

---

# 19. `Timer` vs `ScheduledExecutorService` ⭐⭐⭐⭐

Legacy code may use:

```java
java.util.Timer
```

Modern concurrent applications generally prefer `ScheduledExecutorService` because it integrates with the Executor framework and supports multiple worker threads and Future-based task control.

### Interview answer

> **For modern Java applications, I would generally prefer `ScheduledExecutorService` over `Timer` because it provides executor-based scheduling, better concurrency control and Future-based cancellation.**

---

# 20. Common Mistakes ❌

### Mistake 1 — Confusing fixed rate with fixed delay

Remember:

```text
Fixed rate   → target schedule
Fixed delay  → delay after completion
```

### Mistake 2 — Forgetting to cancel periodic work

A periodic task can continue until its Future is cancelled or the executor is shut down.

### Mistake 3 — Assuming exceptions automatically allow future executions

An uncaught exception can suppress subsequent executions of that periodic task.

### Mistake 4 — Creating a scheduler and never shutting it down

The executor owns worker threads and should have an appropriate lifecycle.

### Mistake 5 — Assuming `shutdown()` immediately stops periodic work

Shutdown prevents new task submissions, while already-scheduled work follows executor shutdown semantics. Use explicit cancellation when you need to stop a particular periodic task.

---

# 21. Interview Scenarios

### Scenario 1

> Run a task once after 10 seconds.

**Answer:** `schedule()`.

### Scenario 2

> Run a metrics collection task every 30 seconds based on a fixed schedule.

**Answer:** Consider `scheduleAtFixedRate()`.

### Scenario 3

> Run cleanup, then wait 30 seconds before starting cleanup again.

**Answer:** `scheduleWithFixedDelay()`.

### Scenario 4

> What is the difference between fixed rate and fixed delay?

**Answer:** Fixed rate targets scheduled execution times; fixed delay waits the configured delay after the previous execution completes.

### Scenario 5

> What happens if a periodic task throws an exception?

**Answer:** That periodic task's subsequent executions are suppressed unless the failure is handled appropriately.

### Scenario 6

> Can scheduled tasks overlap with themselves?

**Answer:** For a given periodic task, executions do not overlap; if an execution takes longer than its period, later execution is delayed. Different scheduled tasks can execute concurrently when the scheduler has multiple workers.

---

# 22. Quick Revision

```text
ScheduledExecutorService
        │
        ├── schedule()
        │      → once after delay
        │
        ├── scheduleAtFixedRate()
        │      → fixed-rate periodic execution
        │
        └── scheduleWithFixedDelay()
               → delay after previous completion
```

### Memory Trick

> **schedule = once**  
> **fixedRate = calendar/rate**  
> **fixedDelay = finish → wait → repeat**

---

# 🎯 Interview Questions

1. What is `ScheduledExecutorService`?
2. `ExecutorService` vs `ScheduledExecutorService`?
3. What does `schedule()` do?
4. What does `scheduleAtFixedRate()` do?
5. What does `scheduleWithFixedDelay()` do?
6. Fixed rate vs fixed delay?
7. Can periodic executions overlap with themselves?
8. What happens when a periodic task throws an exception?
9. What is `ScheduledFuture`?
10. How do you cancel a scheduled task?
11. How do you stop a periodic task?
12. Why should a scheduler be shut down?
13. `Timer` vs `ScheduledExecutorService`?
14. How would you implement a periodic health check?
15. How would you implement delayed retry?
16. How does scheduler pool size affect concurrent scheduled tasks?
17. What happens when a periodic task takes longer than its period?
18. How would you gracefully shut down a scheduler?
19. How do you prevent a scheduled task failure from stopping future executions?
20. Explain all three scheduling methods in 2 minutes.

---

# 🏆 2-Minute Interview Answer

> **"ScheduledExecutorService extends the ExecutorService model with delayed and periodic execution. `schedule()` runs a task once after a delay. `scheduleAtFixedRate()` tries to execute a periodic task according to a fixed rate, while `scheduleWithFixedDelay()` waits for the previous execution to finish and then waits the configured delay before the next execution. Scheduled methods return a ScheduledFuture, which can be used for cancellation and status checks. A periodic task should handle expected exceptions because an uncaught exception can suppress future executions. In production I also make sure the scheduler has a clear lifecycle and is shut down appropriately."**

---

# 💻 Practice Checklist

- [ ] Create a `ScheduledExecutorService`.
- [ ] Practice `schedule()`.
- [ ] Practice `scheduleAtFixedRate()`.
- [ ] Practice `scheduleWithFixedDelay()`.
- [ ] Cancel a delayed task.
- [ ] Cancel a periodic task.
- [ ] Practice `ScheduledFuture`.
- [ ] Compare fixed rate vs fixed delay with timestamps.
- [ ] Handle exceptions inside periodic tasks.
- [ ] Practice scheduled `Callable`.
- [ ] Practice scheduler shutdown.
- [ ] Build a delayed retry example.
- [ ] Build a periodic health-check example.
- [ ] Explain the three scheduling APIs in under 2 minutes.

---

## Navigation

[← 8.5 — `Future` and `Callable`](../05-Future-and-Callable/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.7 — Thread Pool Types**