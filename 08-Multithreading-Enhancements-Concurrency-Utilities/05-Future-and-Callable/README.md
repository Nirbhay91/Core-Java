# 8.5 — `Future` and `Callable`

> **Goal:** Understand how `Callable` produces results from asynchronous tasks and how `Future` represents the pending/completed result.

---

## 1. Core Idea ⭐⭐⭐⭐⭐

```text
Callable<T>
   ↓
submit()
   ↓
Future<T>
   ↓
get()
   ↓
Result
```

### Interview definition

> **`Callable<V>` is a task that can return a result and throw a checked exception. `Future<V>` is a handle representing the result of an asynchronous computation and provides operations such as retrieving the result, checking completion and cancelling the task.**

---

# 2. `Runnable` vs `Callable`

| Feature | `Runnable` | `Callable<V>` |
|---|---|---|
| Method | `run()` | `call()` |
| Return value | No | Yes, `V` |
| Checked exception | Cannot declare checked exception | Can throw checked exception |
| Typical submission | `execute()` / `submit()` | `submit()` |
| Result handle | `Future<?>` when submitted | `Future<V>` |

---

# 3. Basic `Callable` Example ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Callable;

public class CallableExample {
    public static void main(String[] args) throws Exception {

        Callable<Integer> task = () -> {
            return 10 + 20;
        };

        Integer result = task.call();

        System.out.println("Result = " + result);
    }
}
```

> Calling `call()` directly is synchronous. The asynchronous behavior comes when the `Callable` is submitted to an executor.

---

# 4. `Callable` with `ExecutorService` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class CallableExecutorExample {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        Callable<String> task = () -> {
            Thread.sleep(500);
            return "Task completed";
        };

        Future<String> future = executor.submit(task);

        System.out.println("Waiting for result...");

        String result = future.get();

        System.out.println(result);

        executor.shutdown();
    }
}
```

---

# 5. What is `Future`?

`Future<V>` is a handle for an asynchronous computation.

It allows you to:

```java
future.get();
future.get(2, TimeUnit.SECONDS);
future.isDone();
future.isCancelled();
future.cancel(true);
```

Think:

```text
submit task
    ↓
Future returned immediately
    ↓
Task runs in worker thread
    ↓
Future eventually has outcome
    ↓
get() retrieves result
```

---

# 6. `Future.get()` ⭐⭐⭐⭐⭐

```java
Future<Integer> future = executor.submit(() -> 100);

Integer result = future.get();
```

`get()` waits if the computation has not completed yet.

### Important

`get()` can block the calling thread.

This is a major limitation of the traditional `Future` model and one reason modern code may use `CompletableFuture` for composition.

---

# 7. `get()` with Timeout

```java
import java.util.concurrent.TimeUnit;

Integer result = future.get(2, TimeUnit.SECONDS);
```

Possible outcomes include:

- Result available → returned
- Timeout expires → `TimeoutException`
- Task fails → `ExecutionException`
- Calling thread is interrupted → `InterruptedException`

Practice:

```java
import java.util.concurrent.*;

public class FutureTimeoutExample {
    public static void main(String[] args) {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        Future<String> future = executor.submit(() -> {
            Thread.sleep(3000);
            return "Completed";
        });

        try {
            String result = future.get(1, TimeUnit.SECONDS);
            System.out.println(result);
        } catch (TimeoutException e) {
            System.out.println("Task took too long");
            future.cancel(true);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } catch (ExecutionException e) {
            System.out.println("Task failed: " + e.getCause());
        } finally {
            executor.shutdownNow();
        }
    }
}
```

---

# 8. `isDone()`

```java
if (future.isDone()) {
    System.out.println("Task completed");
}
```

`isDone()` becomes `true` when the computation has completed, failed, or been cancelled.

It does **not** mean that the task completed successfully.

For successful completion, you need to retrieve/inspect the result appropriately.

---

# 9. `isCancelled()`

```java
if (future.isCancelled()) {
    System.out.println("Task was cancelled");
}
```

It returns `true` when cancellation succeeded for that Future.

---

# 10. `cancel()` ⭐⭐⭐⭐⭐

```java
boolean cancelled = future.cancel(true);
```

Two common forms:

```java
future.cancel(false);
future.cancel(true);
```

### `cancel(false)`

Does not interrupt a task that has already started.

### `cancel(true)`

Attempts to interrupt the thread executing the task if it has started.

> Cancellation is cooperative. It does not guarantee that arbitrary code stops immediately.

---

# 11. Cancellation Practice

```java
import java.util.concurrent.*;

public class FutureCancellationExample {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        Future<String> future = executor.submit(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    System.out.println("Working...");
                    Thread.sleep(200);
                }
                return "Finished";
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return "Interrupted";
            }
        });

        Thread.sleep(700);

        System.out.println("Cancelled = " + future.cancel(true));
        System.out.println("isCancelled = " + future.isCancelled());

        executor.shutdown();
    }
}
```

---

# 12. Callable Can Throw Checked Exceptions ⭐⭐⭐⭐⭐

```java
Callable<String> task = () -> {
    throw new java.io.IOException("File unavailable");
};
```

A `Callable` can declare/throw checked exceptions.

When submitted to an executor, the failure becomes observable through the returned `Future`.

```java
try {
    future.get();
} catch (ExecutionException e) {
    Throwable original = e.getCause();
    System.out.println(original);
}
```

---

# 13. `ExecutionException`

If the task fails:

```text
Callable
   ↓
throws exception
   ↓
Future stores failure
   ↓
future.get()
   ↓
ExecutionException
   ↓
getCause()
   ↓
Original exception
```

Practice:

```java
import java.util.concurrent.*;

public class CallableExceptionExample {
    public static void main(String[] args) {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        Future<Integer> future = executor.submit(() -> {
            throw new IllegalStateException("Database unavailable");
        });

        try {
            future.get();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } catch (ExecutionException e) {
            System.out.println("Cause = " + e.getCause());
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 14. Multiple `Callable` Tasks ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;
import java.util.*;

public class MultipleCallableExample {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newFixedThreadPool(3);

        List<Callable<Integer>> tasks = List.of(
                () -> 10,
                () -> 20,
                () -> 30
        );

        List<Future<Integer>> futures = executor.invokeAll(tasks);

        for (Future<Integer> future : futures) {
            System.out.println(future.get());
        }

        executor.shutdown();
    }
}
```

`invokeAll()` waits for all supplied tasks to complete before returning its list of Futures.

---

# 15. `invokeAny()` Preview

`ExecutorService` also provides:

```java
String result = executor.invokeAny(tasks);
```

It returns a successfully completed result from one of the supplied tasks and attempts to cancel unfinished tasks.

Detailed usage can be practiced separately when working with executor bulk operations.

---

# 16. `Future` Limitation ⭐⭐⭐⭐⭐

Traditional `Future` is useful but has limitations.

Example:

```java
Future<Integer> future = executor.submit(task);

Integer result = future.get();
```

If you need to combine several asynchronous operations:

```text
Future A ──┐
           ├── combine ──→ Future C
Future B ──┘
```

`Future` does not provide a rich fluent composition API.

This is one reason `CompletableFuture` is important in modern Java concurrency.

---

# 17. Practice — Non-Blocking Submission

```java
import java.util.concurrent.*;

public class FutureSubmissionExample {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        Future<String> future = executor.submit(() -> {
            Thread.sleep(1000);
            return "Background result";
        });

        System.out.println("Main thread continues immediately");

        while (!future.isDone()) {
            System.out.println("Doing other work...");
            Thread.sleep(200);
        }

        System.out.println("Result = " + future.get());

        executor.shutdown();
    }
}
```

The submission itself does not block waiting for task completion. `get()` may block if the result is not ready.

---

# 18. `FutureTask` — Advanced Preview

`FutureTask<V>` implements both `Runnable` and `Future<V>`.

```java
import java.util.concurrent.*;

public class FutureTaskExample {
    public static void main(String[] args) throws Exception {

        FutureTask<Integer> task =
                new FutureTask<>(() -> 50 + 50);

        Thread thread = new Thread(task);
        thread.start();

        System.out.println("Result = " + task.get());
    }
}
```

This is useful to know for interviews, although `ExecutorService.submit()` is generally the more common application-level approach.

---

# 19. `Future` State Model

Conceptually:

```text
             submit()
                ↓
            [Pending]
             /     \
            /       \
        execute    cancel
          ↓           ↓
     [Completed]  [Cancelled]
        /    \
       /      \
   success   failure
      ↓          ↓
   result    ExecutionException
```

`isDone()` is true for completed, failed or cancelled computation.

---

# 20. Real-World Example ⭐⭐⭐⭐⭐

Imagine an order service needs customer information from another service.

```java
Future<Customer> future = executor.submit(
        () -> customerClient.getCustomer(customerId)
);
```

The application can:

```text
submit request
     ↓
Future<Customer>
     ↓
continue other work
     ↓
get result with timeout
     ↓
handle success / failure / cancellation
```

Production code should also consider executor ownership, timeout policy, retries, downstream capacity, cancellation semantics and observability.

---

# 21. Common Mistakes ❌

### Mistake 1 — Calling `get()` immediately when asynchronous work is not needed

```java
Future<String> future = executor.submit(task);
String result = future.get();
```

This may remove much of the benefit of asynchronous execution because the caller waits immediately.

### Mistake 2 — Ignoring timeout requirements

Unbounded `get()` can block indefinitely.

### Mistake 3 — Treating `isDone()` as success

A failed task can also make `isDone()` true.

### Mistake 4 — Assuming `cancel(true)` forcibly kills a task

It requests interruption; task cooperation matters.

### Mistake 5 — Swallowing `InterruptedException`

Restore interrupt status when appropriate:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

---

# 22. Interview Scenarios

### Scenario 1

> I need a return value from an asynchronous task.

**Answer:** `Callable<T>` + `submit()` → `Future<T>`.

### Scenario 2

> I need to wait only 2 seconds for a result.

**Answer:** `future.get(2, TimeUnit.SECONDS)`.

### Scenario 3

> The task failed. How do I find the original exception?

**Answer:** Catch `ExecutionException` and call `getCause()`.

### Scenario 4

> I want to cancel an executing task.

**Answer:** `future.cancel(true)` requests interruption; task must cooperate.

### Scenario 5

> Why use `CompletableFuture` instead of `Future`?

**Answer:** For richer asynchronous composition, chaining, combining, explicit completion and non-blocking-style pipelines.

---

# 23. Quick Revision

```text
Callable<T>
   ↓
returns T
   ↓
can throw checked exception

submit(callable)
   ↓
Future<T>
   ↓
get()
   ↓
result / exception

Future
   ├── get()
   ├── get(timeout)
   ├── isDone()
   ├── isCancelled()
   └── cancel()
```

### Memory Trick

> **Callable = task that Calls and returns a value**  
> **Future = handle for the future result**

---

# 🎯 Interview Questions

1. What is `Callable`?
2. `Runnable` vs `Callable`?
3. What does `Callable.call()` return?
4. Can `Callable` throw checked exceptions?
5. What is `Future`?
6. What does `Future.get()` do?
7. Why can `get()` be dangerous in latency-sensitive code?
8. `get()` vs `get(timeout, unit)`?
9. What is `ExecutionException`?
10. How do you retrieve the original task exception?
11. Difference between `isDone()` and `isCancelled()`?
12. What does `cancel(true)` mean?
13. Does cancellation guarantee immediate termination?
14. What does `invokeAll()` do?
15. What does `invokeAny()` do?
16. What are the limitations of `Future`?
17. Why is `CompletableFuture` useful?
18. What is `FutureTask`?
19. How would you implement a timeout around a remote call?
20. Explain `Callable` + `Future` in 2 minutes.

---

# 🏆 2-Minute Interview Answer

> **"Callable is similar to Runnable but it can return a value and throw checked exceptions. When I submit a Callable to ExecutorService, I get a Future representing that asynchronous computation. Future lets me retrieve the result using get(), wait with a timeout, check whether the computation is done, and cancel it. If the Callable fails, get() throws ExecutionException and the original exception is available through getCause(). One important point is that get() can block, so for more complex asynchronous composition I would prefer CompletableFuture, which provides chaining and combination APIs."**

---

# 💻 Practice Checklist

- [ ] Create a basic `Callable`.
- [ ] Submit `Callable` to `ExecutorService`.
- [ ] Retrieve result with `Future.get()`.
- [ ] Practice `get(timeout, unit)`.
- [ ] Handle `TimeoutException`.
- [ ] Handle `ExecutionException`.
- [ ] Retrieve the original exception using `getCause()`.
- [ ] Practice `isDone()`.
- [ ] Practice `isCancelled()`.
- [ ] Practice `cancel(true)`.
- [ ] Practice multiple Callables with `invokeAll()`.
- [ ] Understand `invokeAny()`.
- [ ] Explain Future limitations.
- [ ] Explain `Callable` + `Future` in under 2 minutes.

---

## Navigation

[← 8.4 — `shutdown()` vs `shutdownNow()`](../04-shutdown-vs-shutdownNow/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.6 — `ScheduledExecutorService`**