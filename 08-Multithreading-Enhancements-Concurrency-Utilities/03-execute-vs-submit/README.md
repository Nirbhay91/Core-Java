# 8.3 — `execute()` vs `submit()`

> **Goal:** Understand the practical and interview-level differences between `Executor.execute()` and `ExecutorService.submit()`.

---

## 1. Core Difference

```text
execute()
   ↓
Runnable
   ↓
No return value

submit()
   ↓
Runnable / Callable
   ↓
Future
   ↓
Result / completion / exception handling
```

### One-line interview answer

> **`execute()` is fire-and-forget task execution, while `submit()` returns a `Future` and supports `Runnable` as well as `Callable`, allowing task completion, result retrieval and exception inspection.**

---

# 2. `execute()`

`execute()` comes from `Executor`:

```java
void execute(Runnable command);
```

It accepts only a `Runnable` and returns nothing.

## Practice Code

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecuteExample {
    public static void main(String[] args) {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        executor.execute(() -> {
            System.out.println("Task running on: "
                    + Thread.currentThread().getName());
        });

        executor.shutdown();
    }
}
```

---

# 3. `submit()` with `Runnable`

`submit(Runnable)` returns a `Future<?>`.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class SubmitRunnableExample {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        Future<?> future = executor.submit(() -> {
            System.out.println("Runnable task running");
        });

        future.get();
        System.out.println("Task completed");

        executor.shutdown();
    }
}
```

### Important

A `Runnable` itself does not return a value, so `Future.get()` for a successfully completed `submit(Runnable)` normally returns `null`.

---

# 4. `submit()` with `Callable` ⭐⭐⭐⭐⭐

`Callable<V>` can return a value.

```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class SubmitCallableExample {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        Callable<Integer> task = () -> {
            return 10 + 20;
        };

        Future<Integer> future = executor.submit(task);

        Integer result = future.get();

        System.out.println("Result = " + result);

        executor.shutdown();
    }
}
```

Output:

```text
Result = 30
```

---

# 5. `execute()` vs `submit()`

| Feature | `execute()` | `submit()` |
|---|---|---|
| Defined by | `Executor` | `ExecutorService` |
| Accepts `Runnable` | ✅ | ✅ |
| Accepts `Callable` | ❌ | ✅ |
| Returns `Future` | ❌ | ✅ |
| Returns task result | ❌ | ✅ via `Future` |
| Can observe completion | Not directly | ✅ via `Future` |
| Exception can be observed through returned `Future` | ❌ | ✅ |
| Fire-and-forget style | ✅ | Possible, but returns a `Future` |
| Cancellation via returned `Future` | ❌ | ✅ |

---

# 6. Exception Handling — Important Interview Point ⭐⭐⭐⭐⭐

This is one of the most commonly asked differences.

## `execute()`

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecuteExceptionExample {
    public static void main(String[] args) {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        executor.execute(() -> {
            throw new RuntimeException("Failure in execute");
        });

        executor.shutdown();
    }
}
```

An exception escaping a task submitted with `execute()` is not captured in a returned `Future`; with a typical `ThreadPoolExecutor`, it can reach the worker thread's uncaught-exception handling and may be reported by the thread's uncaught exception mechanism.

## `submit()`

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class SubmitExceptionExample {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        Future<?> future = executor.submit(() -> {
            throw new RuntimeException("Failure in submit");
        });

        try {
            future.get();
        } catch (Exception e) {
            System.out.println("Task failed: "
                    + e.getCause());
        } finally {
            executor.shutdown();
        }
    }
}
```

The exception is captured as part of the task's `Future` outcome and becomes observable through `Future.get()` (wrapped in `ExecutionException`).

---

# 7. Why `ExecutionException`?

If the task throws an exception:

```java
future.get();
```

may throw:

```text
ExecutionException
       ↓
getCause()
       ↓
Original task exception
```

Practice:

```java
try {
    future.get();
} catch (java.util.concurrent.ExecutionException e) {
    Throwable cause = e.getCause();
    System.out.println("Original exception: " + cause);
}
```

---

# 8. `Future` Gives More Control

A returned `Future` can be used to:

```java
future.get();
future.get(2, java.util.concurrent.TimeUnit.SECONDS);
future.isDone();
future.isCancelled();
future.cancel(true);
```

Practice:

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

public class FutureControlExample {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        Future<String> future = executor.submit(() -> {
            Thread.sleep(1000);
            return "Completed";
        });

        System.out.println("Done? " + future.isDone());

        String result = future.get(2, TimeUnit.SECONDS);
        System.out.println(result);

        System.out.println("Done? " + future.isDone());

        executor.shutdown();
    }
}
```

---

# 9. Cancellation Practice

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class SubmitCancellationExample {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        Future<?> future = executor.submit(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    System.out.println("Working...");
                    Thread.sleep(200);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                System.out.println("Task interrupted");
            }
        });

        Thread.sleep(600);
        future.cancel(true);

        executor.shutdown();
    }
}
```

### Important

`cancel(true)` requests interruption of the executing thread; interruption is cooperative. It does not guarantee that arbitrary task code stops immediately.

---

# 10. `execute()` — When to Use?

Use `execute()` when:

- You only need to start a `Runnable` task.
- You do not need a task result.
- You do not need a returned handle for cancellation or completion.
- You intentionally want simple fire-and-forget execution.

Example:

```java
executor.execute(() -> auditLogger.log("Order received"));
```

The exact production design should still consider whether losing direct task-result/error handling is acceptable.

---

# 11. `submit()` — When to Use?

Use `submit()` when you need:

- A result
- Completion tracking
- Cancellation
- Timeout-based waiting
- `Callable`
- Explicit inspection of task failure through `Future`

Example:

```java
Future<PaymentStatus> future =
        executor.submit(() -> paymentService.checkStatus());
```

---

# 12. Practice — Compare Both ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class ExecuteVsSubmitPractice {
    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        executor.execute(() ->
                System.out.println("execute(): no Future"));

        Future<Integer> future = executor.submit(() -> {
            System.out.println("submit(): Future returned");
            return 100;
        });

        System.out.println("submit result = " + future.get());

        executor.shutdown();
    }
}
```

---

# 13. Common Mistake ❌

### Calling `submit()` and ignoring the `Future`

```java
executor.submit(task);
```

This is valid, but if you do not need a result, cancellation handle, or completion/failure observation, `execute()` may communicate the intent more clearly.

However, do not treat this as an absolute rule. Frameworks and application designs may intentionally use `submit()` for uniform task handling.

---

# 14. Common Interview Trap ❌

### "`submit()` catches exceptions and `execute()` doesn't."

Too simplistic.

Correct explanation:

> `submit()` captures task failure in the returned `Future`, so the caller can observe it through `get()`. With `execute()`, there is no returned `Future`; an exception escaping the task can be handled through the worker thread's uncaught-exception mechanism depending on executor/thread configuration.

---

# 15. Interview Scenarios

### Scenario 1

> I only need to trigger a background `Runnable` and don't need a result. Which one?

**Answer:** Usually `execute()` if no `Future` handle is needed.

### Scenario 2

> I need a calculated value from a background task.

**Answer:** `submit(Callable<T>)` and obtain the result through `Future`.

### Scenario 3

> I need to cancel a task later.

**Answer:** Use `submit()` and retain its `Future`.

### Scenario 4

> I need to detect whether a task failed.

**Answer:** With `submit()`, inspect the `Future` using `get()` and handle `ExecutionException`.

---

# 16. Quick Revision

```text
execute()
   ↓
Runnable only
   ↓
void
   ↓
No Future
   ↓
Simple task execution

submit()
   ↓
Runnable / Callable
   ↓
Future
   ↓
Result / completion / cancellation
   ↓
Task failure observable via Future.get()
```

### Memory Trick

> **execute = Execute**  
> **submit = Submit + Future**

---

# 🎯 Interview Questions

1. What is the difference between `execute()` and `submit()`?
2. Which interface defines `execute()`?
3. Can `execute()` accept `Callable`?
4. Can `submit()` accept `Callable`?
5. What does `submit(Runnable)` return?
6. What does `Future.get()` do?
7. How are exceptions observed with `submit()`?
8. What is `ExecutionException`?
9. Can a `Future` cancel a task?
10. Does `cancel(true)` guarantee immediate task termination?
11. When would you prefer `execute()`?
12. When would you prefer `submit()`?
13. What happens to an exception escaping an `execute()` task?
14. Why might ignoring a returned `Future` be a code smell in some designs?
15. Explain `execute()` vs `submit()` in 30 seconds.

---

# 🏆 2-Minute Interview Answer

> **"`execute()` is defined by the `Executor` interface and accepts a `Runnable` without returning a result or `Future`. `submit()` is provided by `ExecutorService` and accepts both `Runnable` and `Callable`, returning a `Future`. The Future lets us wait for completion, retrieve a result, cancel the task and observe task failure through `get()`. With `submit()`, task exceptions are captured in the Future and `get()` reports them through `ExecutionException`. With `execute()`, there is no Future, so an exception escaping the task can reach the worker thread's uncaught-exception handling. I choose based on whether I need result, completion, cancellation or explicit failure observation."**

---

# 💻 Practice Checklist

- [ ] Run `execute()` example.
- [ ] Run `submit(Runnable)` example.
- [ ] Run `submit(Callable)` example.
- [ ] Compare returned values.
- [ ] Practice `Future.get()`.
- [ ] Practice timeout with `get(timeout, unit)`.
- [ ] Practice `Future.cancel(true)`.
- [ ] Compare exception behavior.
- [ ] Explain `ExecutionException`.
- [ ] Explain `execute()` vs `submit()` in under 2 minutes.

---

## Navigation

[← 8.2 — Thread Pool Fundamentals](../02-Thread-Pool-Fundamentals/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.4 — `shutdown()` vs `shutdownNow()`**