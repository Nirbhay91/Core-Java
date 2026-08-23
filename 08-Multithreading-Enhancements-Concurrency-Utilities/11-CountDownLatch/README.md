# 8.11 — `CountDownLatch`

> **Goal:** Understand how `CountDownLatch` lets one or more threads wait until a fixed number of operations are completed.

---

## 1. What is `CountDownLatch`? ⭐⭐⭐⭐⭐

`CountDownLatch` is a synchronization utility from `java.util.concurrent`.

It maintains a counter:

```text
Initial count = N

countDown() → N - 1
countDown() → N - 1
...

count = 0
   ↓
waiting threads are released
```

A thread calling:

```java
latch.await();
```

waits until the count reaches zero, unless it is interrupted or a timed await expires.

### Memory Trick

> **Latch = gate that opens once the count reaches zero.**

---

# 2. Basic Example ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CountDownLatch;

public class BasicCountDownLatchExample {

    public static void main(String[] args) throws InterruptedException {

        CountDownLatch latch = new CountDownLatch(3);

        for (int i = 1; i <= 3; i++) {
            int workerId = i;

            new Thread(() -> {
                System.out.println("Worker " + workerId + " started");

                sleep(1000 * workerId);

                System.out.println("Worker " + workerId + " completed");
                latch.countDown();
            }).start();
        }

        System.out.println("Main waiting...");

        latch.await();

        System.out.println("All workers completed");
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

### Flow

```text
Main Thread
    |
    | await()
    ↓
┌───────────────┐
│ Count = 3     │
└───────────────┘
   ↑    ↑    ↑
 W1   W2   W3
   |    |    |
 countDown()
   ↓    ↓    ↓
┌───────────────┐
│ Count = 0     │
└───────────────┘
       ↓
Main continues
```

---

# 3. Constructor

```java
CountDownLatch latch = new CountDownLatch(3);
```

The count must be non-negative.

```java
new CountDownLatch(-1); // IllegalArgumentException
```

The count cannot be changed after construction.

---

# 4. `await()` ⭐⭐⭐⭐⭐

```java
latch.await();
```

The calling thread waits until:

```text
count == 0
```

or the waiting thread is interrupted.

It can also be timed:

```java
boolean completed = latch.await(5, TimeUnit.SECONDS);
```

The timed version returns:

```text
true  → count reached zero
false → timeout occurred first
```

### Practice Code — Timed Await

```java
import java.util.concurrent.*;

public class TimedAwaitExample {

    public static void main(String[] args) throws InterruptedException {

        CountDownLatch latch = new CountDownLatch(1);

        new Thread(() -> {
            sleep(3000);
            latch.countDown();
        }).start();

        boolean completed = latch.await(1, TimeUnit.SECONDS);

        System.out.println("Completed within timeout: " + completed);
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

Expected result:

```text
Completed within timeout: false
```

---

# 5. `countDown()` ⭐⭐⭐⭐⭐

```java
latch.countDown();
```

It decreases the count by one.

Important:

```java
Count = 0
countDown()
countDown()
countDown()
```

The count does **not** become negative.

Additional calls after zero have no effect.

---

# 6. `getCount()` ⭐⭐⭐

You can inspect the current count:

```java
long count = latch.getCount();
```

Example:

```java
CountDownLatch latch = new CountDownLatch(3);

System.out.println(latch.getCount()); // 3
latch.countDown();
System.out.println(latch.getCount()); // 2
```

### Important

`getCount()` is useful for observation/debugging, but it should not normally be used as the correctness mechanism for synchronization.

---

# 7. One Latch, Multiple Waiting Threads ⭐⭐⭐⭐⭐

Multiple threads can wait on the same latch.

```java
import java.util.concurrent.*;

public class MultipleWaitersExample {

    public static void main(String[] args) throws InterruptedException {

        CountDownLatch latch = new CountDownLatch(2);

        Runnable waiter = () -> {
            try {
                System.out.println(Thread.currentThread().getName()
                        + " waiting");

                latch.await();

                System.out.println(Thread.currentThread().getName()
                        + " released");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };

        new Thread(waiter, "waiter-1").start();
        new Thread(waiter, "waiter-2").start();

        Thread.sleep(500);

        latch.countDown();
        latch.countDown();
    }
}
```

When the count reaches zero, all currently waiting threads are released.

---

# 8. `CountDownLatch` Is One-Shot ⭐⭐⭐⭐⭐

This is one of the most important interview points.

```text
Count = 3
   ↓
2 → 1 → 0
   ↓
OPEN
```

Once it reaches zero:

```text
0 → cannot be reset
```

If you need a reusable synchronization point, consider `CyclicBarrier` or another appropriate primitive.

### Interview Question

**Can we reset a `CountDownLatch`?**

> No. `CountDownLatch` is one-shot. Once its count reaches zero, it remains open. Create a new latch for another cycle.

---

# 9. `CountDownLatch` vs `join()` ⭐⭐⭐⭐⭐

### `join()`

Usually waits for a **specific thread** to terminate.

```java
thread1.join();
thread2.join();
thread3.join();
```

### `CountDownLatch`

Waits for a **logical number of completion signals**.

```java
CountDownLatch latch = new CountDownLatch(3);
```

Workers can be executor tasks, not necessarily threads directly created by the caller.

| Feature | `join()` | `CountDownLatch` |
|---|---|---|
| Wait for | Thread termination | Count reaching zero |
| Reusable | Thread can be joined once | Latch is one-shot |
| Works naturally with executor tasks | Less convenient | Yes |
| Logical completion model | Limited | Excellent |

---

# 10. Worker Completion Pattern ⭐⭐⭐⭐⭐

A common pattern:

```text
Main / Coordinator
       ↓
Submit N tasks
       ↓
┌──────┬──────┬──────┐
│ W1   │ W2   │ W3   │
└──────┴──────┴──────┘
  ↓      ↓      ↓
countDown()
       ↓
     count=0
       ↓
Coordinator continues
```

### Practice Code — ExecutorService

```java
import java.util.concurrent.*;

public class ExecutorCountDownLatchExample {

    public static void main(String[] args) throws InterruptedException {

        int workers = 4;
        CountDownLatch latch = new CountDownLatch(workers);

        ExecutorService executor = Executors.newFixedThreadPool(workers);

        for (int i = 1; i <= workers; i++) {
            int workerId = i;

            executor.submit(() -> {
                try {
                    System.out.println("Worker " + workerId + " started");
                    sleep(1000);
                    System.out.println("Worker " + workerId + " completed");
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();

        System.out.println("All tasks completed");

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

### Why `finally`?

Because the completion signal should happen even when the task encounters an exception.

Without careful handling, a missing `countDown()` can leave another thread waiting indefinitely.

---

# 11. Failure Scenario — Missing `countDown()` ⚠️

Bad:

```java
executor.submit(() -> {
    if (somethingFails()) {
        return;
    }

    latch.countDown();
});
```

If the method returns before `countDown()`, the latch may never reach zero.

Better:

```java
executor.submit(() -> {
    try {
        doWork();
    } finally {
        latch.countDown();
    }
});
```

### Important

`finally` ensures the signal is sent when the task exits normally or because of an exception. It does not magically solve cancellation or process termination scenarios; those need their own design.

---

# 12. Start Gate Pattern ⭐⭐⭐⭐⭐

`CountDownLatch` can also be used to make workers wait for a signal before starting.

Use a latch with count `1`:

```text
Workers
  ↓
await()
  ↓
WAITING

Coordinator
  ↓
countDown()
  ↓
All workers may proceed
```

### Practice Code

```java
import java.util.concurrent.*;

public class StartGateExample {

    public static void main(String[] args) throws InterruptedException {

        CountDownLatch startGate = new CountDownLatch(1);
        ExecutorService executor = Executors.newFixedThreadPool(3);

        for (int i = 1; i <= 3; i++) {
            int id = i;

            executor.execute(() -> {
                try {
                    System.out.println("Worker " + id + " ready");
                    startGate.await();
                    System.out.println("Worker " + id + " started work");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        Thread.sleep(1000);
        System.out.println("Coordinator opens start gate");
        startGate.countDown();

        executor.shutdown();
    }
}
```

---

# 13. Start Gate vs Completion Latch ⭐⭐⭐⭐⭐

There are two common patterns.

### Completion latch

```java
new CountDownLatch(numberOfWorkers)
```

Workers call:

```java
countDown();
```

Coordinator calls:

```java
await();
```

### Start gate

```java
new CountDownLatch(1)
```

Workers call:

```java
await();
```

Coordinator calls:

```java
countDown();
```

### Memory Trick

```text
N-count latch → wait for N completions
1-count latch → release workers together
```

---

# 14. `CountDownLatch` with Success/Failure Tracking ⭐⭐⭐⭐⭐

A latch only tracks completion count. It does **not** tell you whether tasks succeeded.

You need separate result/error handling.

```java
import java.util.*;
import java.util.concurrent.*;

public class LatchWithResultsExample {

    public static void main(String[] args) throws InterruptedException {

        int workers = 3;
        CountDownLatch latch = new CountDownLatch(workers);
        List<String> results = Collections.synchronizedList(new ArrayList<>());

        ExecutorService executor = Executors.newFixedThreadPool(workers);

        for (int i = 1; i <= workers; i++) {
            int id = i;

            executor.execute(() -> {
                try {
                    if (id == 2) {
                        throw new RuntimeException("Worker failed");
                    }

                    results.add("Worker " + id + " success");
                } catch (RuntimeException e) {
                    results.add("Worker " + id + " failed");
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();

        results.forEach(System.out::println);
        executor.shutdown();
    }
}
```

### Important

> **Latch answers: "Are all required completion signals received?"**

It does not answer:

> "Did all tasks succeed?"

---

# 15. Timed Coordination ⭐⭐⭐⭐⭐

A coordinator should often avoid waiting forever.

```java
boolean completed = latch.await(10, TimeUnit.SECONDS);

if (!completed) {
    System.out.println("Timed out waiting for workers");
}
```

This is useful when dealing with:

- Slow services
- External systems
- Startup dependencies
- Batch operations
- Resource initialization

---

# 16. Startup Initialization Scenario ⭐⭐⭐⭐⭐

Imagine a service depends on three initialization tasks:

```text
Load configuration
       ↓
Connect database
       ↓
Warm cache
       ↓
All ready
```

```java
CountDownLatch startupLatch = new CountDownLatch(3);
```

Each initializer signals completion:

```java
try {
    initialize();
} finally {
    startupLatch.countDown();
}
```

Application startup waits:

```java
if (startupLatch.await(30, TimeUnit.SECONDS)) {
    startAcceptingRequests();
} else {
    failStartup();
}
```

This creates a clear startup barrier with a timeout.

---

# 17. Real-World Scenario — Parallel API Calls ⭐⭐⭐⭐⭐

Suppose an API needs data from multiple independent sources:

```text
Request
  ↓
┌────────┬────────┬────────┐
│ User   │ Order  │ Wallet │
│ API    │ API    │ API    │
└────────┴────────┴────────┘
     ↓       ↓       ↓
 countDown() for each
          ↓
      count = 0
          ↓
   Combine response
```

A latch can coordinate completion, but a production implementation also needs:

- Timeouts
- Cancellation strategy
- Error handling
- Executor management
- Result collection
- Partial-result policy

For asynchronous composition, `CompletableFuture` may be a better abstraction depending on the use case.

---

# 18. `CountDownLatch` vs `CyclicBarrier` ⭐⭐⭐⭐⭐

| Feature | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| One-shot | Yes | No, reusable |
| Count reaches zero | Opens latch | Trips barrier |
| Who signals? | Any participating thread can `countDown()` | Threads call `await()` |
| Common use | Wait for tasks to complete | Wait for parties to reach a phase |
| Reusable phases | ❌ | ✅ |
| Barrier action | ❌ | ✅ |

### Example intuition

**Latch:**

> "Wait until these 5 jobs finish."

**Barrier:**

> "All 5 workers must reach this point before any continue to the next phase."

---

# 19. `CountDownLatch` vs `Semaphore` ⭐⭐⭐⭐

These solve different problems.

### Latch

Coordinates completion/signaling.

```text
Wait until count reaches zero
```

### Semaphore

Controls access to a limited number of permits.

```text
Only N threads may access resource concurrently
```

### Memory Trick

```text
Latch      → WAIT FOR EVENT
Semaphore  → LIMIT CONCURRENCY
```

---

# 20. `CountDownLatch` vs `Future` ⭐⭐⭐⭐

A `Future` represents the result of an asynchronous computation.

A latch represents a completion signal/count.

```text
Future
  → What is the result?

Latch
  → Are all required signals complete?
```

For result-oriented coordination, `Future`/`CompletableFuture` is often more expressive.

---

# 21. Thread Interruption ⭐⭐⭐⭐⭐

`await()` is interruptible.

```java
try {
    latch.await();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Important

When catching `InterruptedException`, code should normally either:

1. propagate the exception, or
2. restore the interrupt status with:

```java
Thread.currentThread().interrupt();
```

Do not silently swallow interruption.

---

# 22. Practice — Interrupting a Waiting Thread ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class LatchInterruptExample {

    public static void main(String[] args) throws InterruptedException {

        CountDownLatch latch = new CountDownLatch(1);

        Thread waiter = new Thread(() -> {
            try {
                System.out.println("Waiting on latch...");
                latch.await();
                System.out.println("Latch opened");
            } catch (InterruptedException e) {
                System.out.println("Waiter interrupted");
                Thread.currentThread().interrupt();
            }
        });

        waiter.start();

        Thread.sleep(500);
        waiter.interrupt();
    }
}
```

---

# 23. Common Mistakes ❌

### Mistake 1 — Forgetting `countDown()`

The waiter may block indefinitely.

### Mistake 2 — Using the wrong count

```java
new CountDownLatch(5)
```

but only four tasks ever call `countDown()`.

The latch never opens.

### Mistake 3 — Assuming zero means success

Zero only means all expected signals were received.

### Mistake 4 — Trying to reset the latch

It cannot be reset.

### Mistake 5 — Waiting forever

Use timed `await()` when indefinite waiting is unacceptable.

### Mistake 6 — Swallowing interruption

Restore the interrupt status or propagate the exception.

### Mistake 7 — Using a latch when a reusable barrier is needed

Consider `CyclicBarrier` for repeated phase coordination.

---

# 24. Production Pattern ⭐⭐⭐⭐⭐

```java
CountDownLatch latch = new CountDownLatch(taskCount);

for (Task task : tasks) {
    executor.execute(() -> {
        try {
            process(task);
        } finally {
            latch.countDown();
        }
    });
}

if (!latch.await(30, TimeUnit.SECONDS)) {
    handleTimeout();
}
```

### Production checklist

```text
✓ Correct count
✓ countDown() in finally
✓ Timeout
✓ Interrupt handling
✓ Result/error tracking
✓ Executor lifecycle
✓ Metrics/logging
```

---

# 25. Interview Scenarios ⭐⭐⭐⭐⭐

### Q1. What is `CountDownLatch`?

> A synchronization utility that allows one or more threads to wait until a fixed count reaches zero.

### Q2. Is `CountDownLatch` reusable?

> No. It is one-shot. Once the count reaches zero, it remains zero.

### Q3. What happens when `countDown()` is called at zero?

> Nothing. The count does not go below zero.

### Q4. Can multiple threads call `await()`?

> Yes. All waiting threads are released when the count reaches zero.

### Q5. Can multiple threads call `countDown()`?

> Yes. The latch is designed for concurrent signaling.

### Q6. What if one worker never calls `countDown()`?

> A waiting thread can remain blocked indefinitely unless it uses timed `await()` or another cancellation/failure mechanism.

### Q7. `CountDownLatch` vs `join()`?

> `join()` waits for a specific thread to terminate; a latch coordinates a logical number of completion signals and works naturally with executor tasks.

### Q8. `CountDownLatch` vs `CyclicBarrier`?

> A latch is one-shot and commonly represents completion/signaling; a barrier is reusable and coordinates participating threads at a synchronization point.

### Q9. Why put `countDown()` in `finally`?

> To ensure the completion signal is sent when the task exits normally or because of an exception.

### Q10. Does `CountDownLatch` tell whether tasks succeeded?

> No. It only tracks the count of signals. Success/failure must be tracked separately.

### Q11. Is `await()` interruptible?

> Yes. It throws `InterruptedException` when the waiting thread is interrupted.

### Q12. How would you use a latch during service startup?

> Start independent initialization tasks, count down when each finishes, then wait with a timeout before accepting traffic.

---

# 26. Quick Revision

```text
CountDownLatch(N)
       ↓
     count=N
       ↓
countDown()
       ↓
count--
       ↓
count == 0
       ↓
await() returns
```

### Key APIs

```java
new CountDownLatch(N)
latch.await()
latch.await(timeout, unit)
latch.countDown()
latch.getCount()
```

### Key facts

```text
✓ Thread-safe
✓ One-shot
✓ Count cannot be reset
✓ Multiple waiters allowed
✓ Multiple countdown callers allowed
✓ await() is interruptible
✓ Timed await supported
```

---

# 🎯 Interview Questions

1. What is `CountDownLatch`?
2. How does `await()` work?
3. How does `countDown()` work?
4. Why is it called a latch?
5. Can it be reset?
6. What happens after count reaches zero?
7. Can multiple threads wait on the same latch?
8. Can multiple threads call `countDown()`?
9. What happens if `countDown()` is called too many times?
10. Why use `finally` around `countDown()`?
11. What if a task fails before `countDown()`?
12. `CountDownLatch` vs `join()`?
13. `CountDownLatch` vs `CyclicBarrier`?
14. `CountDownLatch` vs `Semaphore`?
15. `CountDownLatch` vs `Future`?
16. What is the start-gate pattern?
17. How do you use a latch for startup coordination?
18. Why use timed `await()`?
19. Is `await()` interruptible?
20. How would you coordinate parallel API calls?
21. Does zero count mean success?
22. Explain `CountDownLatch` in 2 minutes.

---

# 🏆 2-Minute Interview Answer

> **"`CountDownLatch` is a synchronization utility from `java.util.concurrent` used when one or more threads need to wait for a fixed number of completion signals. We create it with a count, workers call `countDown()` when their work is complete, and the coordinator calls `await()` to wait until the count reaches zero. It is one-shot and cannot be reset. A common production pattern is to put `countDown()` in a `finally` block and use timed `await()` so the coordinator does not wait forever. The latch only represents completion signals; it does not tell us whether the operations succeeded, so result and error handling must be separate. Compared with `CyclicBarrier`, a latch is one-shot and is commonly used for completion coordination, while a barrier is reusable for repeated synchronization phases."**

---

# 💻 Practice Checklist

- [ ] Create a basic `CountDownLatch`.
- [ ] Use `await()`.
- [ ] Use timed `await()`.
- [ ] Call `countDown()` from multiple workers.
- [ ] Use `getCount()` for observation.
- [ ] Coordinate executor tasks.
- [ ] Put `countDown()` inside `finally`.
- [ ] Simulate a missing `countDown()`.
- [ ] Implement a start-gate pattern.
- [ ] Track success/failure separately.
- [ ] Handle interruption correctly.
- [ ] Implement startup initialization coordination.
- [ ] Simulate parallel API completion.
- [ ] Compare latch vs `join()`.
- [ ] Compare latch vs `CyclicBarrier`.
- [ ] Explain the topic in under 2 minutes.

---

## Navigation

[← 8.10 — Custom `ThreadFactory`](../10-Custom-ThreadFactory/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.12 — `CyclicBarrier`**