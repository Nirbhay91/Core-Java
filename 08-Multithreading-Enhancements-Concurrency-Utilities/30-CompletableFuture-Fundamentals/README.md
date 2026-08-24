# 8.30 — `CompletableFuture` Fundamentals

> **Goal:** Understand `CompletableFuture` as Java's composable asynchronous programming model: how to create stages, run work asynchronously, transform results, combine stages, wait for completion, and handle failures.

---

## 1. Big Picture ⭐⭐⭐⭐⭐

`CompletableFuture<T>` represents a computation that will eventually complete with either a result or an exception.

```text
start async computation
        ↓
   CompletableFuture<T>
        ↓
  not completed yet
        ↓
   ┌────┴────┐
 success   failure
   ↓          ↓
 result    exception
```

### Interview one-liner

> `CompletableFuture` is a `Future` implementation and `CompletionStage` that lets us model asynchronous computations and compose dependent or independent stages without manually coordinating threads.

---

# 2. Why `CompletableFuture`? ⭐⭐⭐⭐⭐

Traditional `Future` often leads to:

```java
Future<String> future = executor.submit(task);
String result = future.get();
```

`get()` can block the current thread.

With `CompletableFuture`, we can express:

```java
CompletableFuture
      ↓
 async task
      ↓
 transform
      ↓
 combine
      ↓
 handle error
      ↓
 final result
```

This gives us a declarative pipeline for asynchronous work.

---

# 3. Core Interfaces ⭐⭐⭐⭐⭐

`CompletableFuture` implements both:

```java
Future<T>
CompletionStage<T>
```

Mental model:

```text
Future
→ represents eventual result

CompletionStage
→ represents a stage in an async pipeline

CompletableFuture
→ Future + CompletionStage + explicit completion APIs
```

---

# 4. Creating an Already-Completed Future ⭐⭐⭐⭐⭐

```java
CompletableFuture<String> future =
        CompletableFuture.completedFuture("Java");

System.out.println(future.join());
```

This is useful when an API expects a `CompletableFuture` but the result is already available.

---

# 5. Practice — `completedFuture()`

```java
import java.util.concurrent.CompletableFuture;

public class CompletedFutureDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.completedFuture("Hello Java");

        System.out.println("Done = " + future.isDone());
        System.out.println("Result = " + future.join());
    }
}
```

---

# 6. `supplyAsync()` ⭐⭐⭐⭐⭐

Use `supplyAsync()` when the asynchronous task produces a value.

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "Java");
```

Mental model:

```text
supplyAsync
    ↓
Supplier<T>
    ↓
CompletableFuture<T>
```

---

# 7. Practice — First Async Task ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class SupplyAsyncDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    return "Hello from async task";
                });

        System.out.println(future.join());
    }
}
```

`join()` waits if necessary and returns the result.

---

# 8. `runAsync()` vs `supplyAsync()` ⭐⭐⭐⭐⭐

### `runAsync()`

No result:

```java
CompletableFuture<Void> future =
        CompletableFuture.runAsync(() -> {
            System.out.println("Running...");
        });
```

### `supplyAsync()`

Produces a result:

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "Java");
```

### Easy memory trick

```text
runAsync     → run something → no value
supplyAsync  → supply value → returns T
```

---

# 9. Practice — `runAsync()` vs `supplyAsync()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class RunVsSupplyDemo {

    public static void main(String[] args) {
        CompletableFuture<Void> run =
                CompletableFuture.runAsync(() ->
                        System.out.println("Side effect"));

        CompletableFuture<Integer> supply =
                CompletableFuture.supplyAsync(() -> 10 + 20);

        run.join();
        System.out.println("Result = " + supply.join());
    }
}
```

---

# 10. Default Executor ⭐⭐⭐⭐⭐

When using:

```java
CompletableFuture.supplyAsync(task)
```

without an explicit executor, the async stage uses the default asynchronous execution facility, commonly the `ForkJoinPool.commonPool()` in the standard JDK implementation.

Do not hard-code the assumption that every async continuation uses the common pool; synchronous and asynchronous continuation methods have different execution semantics.

---

# 11. Custom Executor ⭐⭐⭐⭐⭐

For application-specific workloads, pass an executor explicitly:

```java
ExecutorService executor =
        Executors.newFixedThreadPool(4);

CompletableFuture<String> future =
        CompletableFuture.supplyAsync(
                () -> "Result",
                executor);
```

This gives control over thread-pool sizing and workload isolation.

---

# 12. Practice — Custom Executor ⭐⭐⭐⭐⭐

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
                        System.out.println(
                                "Running on " +
                                Thread.currentThread().getName());
                        return "Data loaded";
                    }, executor);

            System.out.println(future.join());
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 13. `isDone()` ⭐⭐⭐⭐

Check whether a future has completed:

```java
future.isDone();
```

It returns `true` after normal, exceptional, or cancelled completion.

---

# 14. `isCompletedExceptionally()` ⭐⭐⭐⭐

```java
future.isCompletedExceptionally();
```

Use it to check whether completion happened exceptionally, including cancellation.

---

# 15. `join()` ⭐⭐⭐⭐⭐

```java
String result = future.join();
```

It waits for completion if necessary.

If the computation fails, `join()` throws an unchecked `CompletionException` whose cause represents the underlying failure.

### Interview point

```text
get()
→ checked InterruptedException / ExecutionException

join()
→ unchecked CompletionException
```

---

# 16. `get()` vs `join()` ⭐⭐⭐⭐⭐

| | `get()` | `join()` |
|---|---|---|
| Waits | Yes | Yes |
| Checked exceptions | Yes | No |
| Failure wrapper | `ExecutionException` | `CompletionException` |
| Common in CF pipelines | Less convenient | Very common |

Neither method is inherently asynchronous; calling either can block the calling thread until completion.

---

# 17. Practice — `get()` vs `join()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

public class GetVsJoinDemo {

    public static void main(String[] args)
            throws ExecutionException, InterruptedException {

        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> "Java");

        System.out.println("get = " + future.get());
        System.out.println("join = " + future.join());
    }
}
```

---

# 18. `thenApply()` Preview ⭐⭐⭐⭐⭐

`thenApply()` transforms the result of one stage into another result.

```java
CompletableFuture<String> result =
        CompletableFuture
                .supplyAsync(() -> "java")
                .thenApply(String::toUpperCase);
```

Pipeline:

```text
"java"
  ↓
thenApply
  ↓
"JAVA"
```

Detailed composition methods are covered in **8.31**.

---

# 19. Practice — Simple Pipeline ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class BasicPipelineDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture
                        .supplyAsync(() -> "java")
                        .thenApply(String::toUpperCase)
                        .thenApply(value -> value + " CONCURRENCY");

        System.out.println(future.join());
    }
}
```

---

# 20. Completion Stages ⭐⭐⭐⭐⭐

A `CompletableFuture` pipeline can contain multiple stages:

```text
Stage 1
  ↓
Stage 2
  ↓
Stage 3
  ↓
Final result
```

Each stage can be represented by another `CompletionStage` / `CompletableFuture`.

---

# 21. Explicit Completion ⭐⭐⭐⭐⭐

A `CompletableFuture` can be manually completed:

```java
CompletableFuture<String> future =
        new CompletableFuture<>();

future.complete("Manual result");
```

This is useful when adapting callback-style or external asynchronous APIs.

---

# 22. Practice — Manual Completion ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ManualCompletionDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                new CompletableFuture<>();

        System.out.println("Done before = " + future.isDone());

        future.complete("Completed manually");

        System.out.println("Done after = " + future.isDone());
        System.out.println("Result = " + future.join());
    }
}
```

---

# 23. `complete()` Is First-Completion-Wins ⭐⭐⭐⭐⭐

Consider:

```java
future.complete("A");
future.complete("B");
```

Only the first successful completion wins.

```text
complete("A") → accepted
complete("B") → false
result → A
```

---

# 24. Practice — First Completion Wins ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class FirstCompletionWinsDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                new CompletableFuture<>();

        boolean first = future.complete("A");
        boolean second = future.complete("B");

        System.out.println("First accepted = " + first);
        System.out.println("Second accepted = " + second);
        System.out.println("Result = " + future.join());
    }
}
```

---

# 25. Exceptional Completion ⭐⭐⭐⭐⭐

A future can complete exceptionally:

```java
future.completeExceptionally(
        new RuntimeException("Something failed"));
```

Then:

```java
future.join();
```

fails with `CompletionException` wrapping the cause.

---

# 26. Practice — Exceptional Completion ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ExceptionalCompletionDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                new CompletableFuture<>();

        future.completeExceptionally(
                new IllegalStateException("Database unavailable"));

        try {
            future.join();
        } catch (Exception e) {
            System.out.println("Type = " +
                    e.getClass().getSimpleName());
            System.out.println("Cause = " +
                    e.getCause().getClass().getSimpleName());
        }
    }
}
```

---

# 27. Cancellation ⭐⭐⭐⭐

`CompletableFuture` supports cancellation through `Future`:

```java
future.cancel(true);
```

Cancellation completes the future exceptionally with a `CancellationException`.

Important nuance: cancellation of a `CompletableFuture` does not provide an automatic guarantee that an independently running underlying computation is interrupted or stopped.

---

# 28. Practice — Cancellation ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class CancellationDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                new CompletableFuture<>();

        boolean cancelled = future.cancel(true);

        System.out.println("Cancelled = " + cancelled);
        System.out.println("Done = " + future.isDone());
        System.out.println("Cancelled state = " + future.isCancelled());
    }
}
```

---

# 29. `completeOnTimeout()` and `orTimeout()` Preview ⭐⭐⭐⭐

Modern Java provides timeout helpers.

### `orTimeout()`

If the future does not complete in time, it completes exceptionally with a timeout exception.

```java
future.orTimeout(2, TimeUnit.SECONDS);
```

### `completeOnTimeout()`

If it does not complete in time, it completes normally with a fallback value.

```java
future.completeOnTimeout("fallback", 2, TimeUnit.SECONDS);
```

These are useful production features and will be revisited with asynchronous composition.

---

# 30. Practice — Timeout APIs ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class TimeoutDemo {

    public static void main(String[] args) {
        CompletableFuture<String> fallback =
                new CompletableFuture<>();

        fallback.completeOnTimeout(
                "Fallback response",
                100,
                TimeUnit.MILLISECONDS);

        System.out.println(fallback.join());

        CompletableFuture<String> timeout =
                new CompletableFuture<>();

        timeout.orTimeout(100, TimeUnit.MILLISECONDS);

        try {
            timeout.join();
        } catch (Exception e) {
            System.out.println("Timed out: " +
                    e.getClass().getSimpleName());
        }
    }
}
```

---

# 31. Async Does NOT Mean Parallel ⭐⭐⭐⭐⭐

This is a common interview trap.

```java
CompletableFuture.supplyAsync(task);
```

means the task is submitted for asynchronous execution.

It does not automatically mean:

```text
all tasks execute simultaneously
```

Actual concurrency depends on the executor and available threads.

---

# 32. Practice — Observe Thread Names ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class AsyncThreadDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    System.out.println("Task thread = " +
                            Thread.currentThread().getName());
                    return "Done";
                });

        System.out.println("Main thread = " +
                Thread.currentThread().getName());

        System.out.println(future.join());
    }
}
```

Run it multiple times and observe the thread names.

---

# 33. Async Pipeline vs Blocking Style ⭐⭐⭐⭐⭐

Blocking style:

```java
String user = userFuture.get();
String orders = orderFuture.get();
```

Composition style:

```java
userFuture
    .thenCompose(user -> loadOrders(user));
```

The second style lets the computation be represented as a continuation pipeline instead of manually blocking for each intermediate result.

---

# 34. Important: `CompletableFuture` Does Not Create Infinite Threads ⭐⭐⭐⭐⭐

This:

```java
CompletableFuture.supplyAsync(task);
```

does not create one new OS thread per future.

The task is submitted to an executor.

```text
Many CompletableFutures
        ↓
      executor
        ↓
 limited worker threads
```

This is why executor configuration matters.

---

# 35. Practice — Many Futures, Limited Pool ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class LimitedPoolDemo {

    public static void main(String[] args) {
        ExecutorService executor =
                Executors.newFixedThreadPool(2);

        try {
            CompletableFuture<?>[] futures = new CompletableFuture[5];

            for (int i = 0; i < futures.length; i++) {
                int taskId = i + 1;

                futures[i] = CompletableFuture.runAsync(() -> {
                    System.out.println(
                            "Task " + taskId + " -> " +
                            Thread.currentThread().getName());
                }, executor);
            }

            CompletableFuture.allOf(futures).join();
        } finally {
            executor.shutdown();
        }
    }
}
```

The five tasks share a pool containing only two worker threads.

---

# 36. `CompletableFuture` and Thread Safety ⭐⭐⭐⭐

A `CompletableFuture` can safely be completed by concurrent actors; only one completion wins.

But the objects captured by your lambdas are not automatically thread-safe.

For example:

```java
int[] counter = {0};
```

and multiple asynchronous tasks modifying it can still create a race condition.

### Rule

> Thread-safe future coordination does not make arbitrary shared mutable state thread-safe.

---

# 37. Practice — Shared State Trap ⭐⭐⭐⭐⭐

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CompletableFuture;

public class SharedStateTrapDemo {

    public static void main(String[] args) {
        List<Integer> values = new ArrayList<>();

        CompletableFuture<Void> first =
                CompletableFuture.runAsync(() -> values.add(1));

        CompletableFuture<Void> second =
                CompletableFuture.runAsync(() -> values.add(2));

        CompletableFuture.allOf(first, second).join();

        System.out.println(values);
    }
}
```

Do not use this pattern when correctness depends on thread-safe mutation of the list. Prefer immutable results, independent data, or a suitable concurrent collection/synchronization strategy.

---

# 38. Exception from Async Supplier ⭐⭐⭐⭐⭐

If the supplier throws:

```java
CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("Failure");
});
```

that future completes exceptionally.

A downstream stage can observe and recover from it using exception-handling methods, covered in detail later.

---

# 39. Practice — Async Failure ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class AsyncFailureDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new IllegalArgumentException("Invalid input");
                });

        try {
            future.join();
        } catch (Exception e) {
            System.out.println("Failed: " + e.getCause());
        }
    }
}
```

---

# 40. `CompletableFuture` vs `Future` ⭐⭐⭐⭐⭐

| Capability | `Future` | `CompletableFuture` |
|---|---|---|
| Represents async result | ✅ | ✅ |
| `get()` | ✅ | ✅ |
| Explicit completion | ❌ | ✅ |
| Composition | Limited | ✅ |
| Transform result | Limited | ✅ |
| Combine stages | Limited | ✅ |
| Exception pipeline | Limited | ✅ |
| Completion callbacks | ❌ | ✅ |
| Implements `CompletionStage` | ❌ | ✅ |

### Interview answer

> `Future` is primarily a handle for retrieving or cancelling an asynchronous result, while `CompletableFuture` additionally provides a rich completion-stage API for composing, transforming, combining, and handling asynchronous computations.

---

# 41. Common Mistake — Calling `join()` Too Early ⭐⭐⭐⭐⭐

This:

```java
String a = futureA.join();
String b = futureB.join();
```

can be valid, but if placed too early inside an async workflow it can reintroduce blocking.

Prefer composing stages when the architecture benefits from non-blocking orchestration.

---

# 42. Practice — Avoid Premature Blocking ⭐⭐⭐⭐⭐

Less composable:

```java
CompletableFuture<String> a =
        CompletableFuture.supplyAsync(() -> "A");

CompletableFuture<String> b =
        CompletableFuture.supplyAsync(() -> "B");

String result = a.join() + b.join();
System.out.println(result);
```

The caller waits explicitly for both results.

Later topics will show composition using `thenCombine()` and `allOf()`.

---

# 43. Production Scenario ⭐⭐⭐⭐⭐

### Requirement

An API needs:

```text
Load customer
      ↓
Load customer preferences
      ↓
Build response
```

A `CompletableFuture` pipeline can represent the workflow:

```text
loadCustomerAsync()
        ↓
transform/combine
        ↓
loadPreferencesAsync()
        ↓
buildResponse()
```

The key benefit is not "threads disappear"; it is that asynchronous dependencies become explicit and composable.

---

# 44. Production Scenario — Independent Calls ⭐⭐⭐⭐⭐

Suppose an API needs:

```text
Customer service ─┐
                  ├──→ response
Order service ────┘
```

The calls may be started independently and combined later.

This can reduce end-to-end latency compared with strictly sequential calls, assuming the downstream services and executor have capacity for the concurrency.

---

# 45. Interview Scenario — Which Executor? 🏆

### Question

> Should every `CompletableFuture` use the common pool?

### Strong answer

> No. The common pool is convenient for general asynchronous tasks, but production applications may need explicit executors for workload isolation, predictable concurrency, CPU versus I/O separation, custom thread naming, or operational control. For blocking I/O, I would carefully consider the executor strategy rather than blindly consuming common-pool workers.

---

# 46. Interview Scenario — Is `CompletableFuture` Non-Blocking? 🏆

### Strong answer

> The API supports non-blocking composition, but `CompletableFuture` itself does not guarantee that every operation is non-blocking. Methods such as `get()` and `join()` can block the caller. Also, the work executed inside a stage may itself block. So I would distinguish between asynchronous execution and non-blocking application design.

---

# 47. Interview Scenario — Does `supplyAsync()` Guarantee a New Thread? 🏆

### Strong answer

> No. `supplyAsync()` submits the supplier to an executor. It does not guarantee creation of a new thread. With the overload that does not accept an executor, the default asynchronous execution facility is used. With the executor overload, the supplied executor determines where the task is executed.

---

# 48. Interview Scenario — Cancellation 🏆

### Question

> If I call `future.cancel(true)`, will the running task definitely stop?

### Strong answer

> No. Cancellation completes the `CompletableFuture` as cancelled, but it does not guarantee that the underlying independently executing computation is interrupted or stops. Cancellation and interruption of the actual work are separate concerns that depend on how the work is executed and responds to interruption.

---

# 49. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1
**`CompletableFuture` means a new thread is always created.**

❌ False.

### Trap 2
**`join()` is non-blocking.**

❌ False. It can wait.

### Trap 3
**`runAsync()` returns a result.**

❌ It returns `CompletableFuture<Void>`.

### Trap 4
**`supplyAsync()` is only for CPU tasks.**

❌ The API does not enforce this; executor/workload design matters.

### Trap 5
**Every continuation runs on the same thread.**

❌ Execution depends on whether the method is synchronous or async and on completion timing/executor selection.

### Trap 6
**A thread-safe future makes captured mutable objects thread-safe.**

❌ False.

### Trap 7
**`cancel(true)` guarantees the underlying task stops.**

❌ False.

### Trap 8
**`get()` and `join()` make the pipeline asynchronous.**

❌ They are result retrieval methods and may block.

---

# 50. 2-Minute Interview Answer 🏆

> **"`CompletableFuture` is Java's implementation of `Future` and `CompletionStage` for composing asynchronous computations. We can create asynchronous tasks using `runAsync()` when there is no result and `supplyAsync()` when a result is produced. A future can be transformed and chained using completion-stage methods, combined with other futures, and completed normally or exceptionally. `join()` and `get()` can wait for the result, so using them too early can reintroduce blocking. By default, async stages use the default asynchronous execution facility, but production applications can supply a custom executor for workload isolation and operational control. `CompletableFuture` does not make arbitrary shared state thread-safe, and cancellation does not guarantee that underlying work is interrupted. Its main value is composable asynchronous workflows rather than simply creating threads."**

---

# 51. Quick Revision ⭐⭐⭐⭐⭐

```text
CompletableFuture<T>
        ↓
Future + CompletionStage
        ↓
runAsync() → no result
supplyAsync() → result
        ↓
thenApply() → transform
thenCompose() → dependent async stage
thenCombine() → combine independent results
        ↓
exception handling
        ↓
join()/get() → retrieve result / may block
        ↓
complete() → manual completion
completeExceptionally() → exceptional completion
cancel() → cancellation state
        ↓
custom Executor → control execution
```

### Golden Rule 🧠

> **`CompletableFuture` is about composing asynchronous results, not about creating threads.**

---

# 52. 💻 Practice Checklist

- [ ] Create `completedFuture()`
- [ ] Practice `runAsync()`
- [ ] Practice `supplyAsync()`
- [ ] Compare `runAsync()` vs `supplyAsync()`
- [ ] Observe default executor thread names
- [ ] Use a custom `ExecutorService`
- [ ] Practice `isDone()`
- [ ] Practice `join()`
- [ ] Compare `get()` vs `join()`
- [ ] Practice `thenApply()` basics
- [ ] Manually `complete()` a future
- [ ] Understand first-completion-wins
- [ ] Complete exceptionally
- [ ] Practice cancellation
- [ ] Practice `orTimeout()` / `completeOnTimeout()`
- [ ] Understand async vs parallel
- [ ] Understand executor sharing
- [ ] Understand shared-state thread-safety risks
- [ ] Practice async failure
- [ ] Compare `Future` vs `CompletableFuture`
- [ ] Explain why premature `join()` can block
- [ ] Answer production executor scenarios
- [ ] Give the 2-minute interview answer

---

## Navigation

[← 8.29 — `ConcurrentLinkedQueue`](../29-ConcurrentLinkedQueue/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.31 — `thenApply` / `thenCompose` / `thenCombine`**