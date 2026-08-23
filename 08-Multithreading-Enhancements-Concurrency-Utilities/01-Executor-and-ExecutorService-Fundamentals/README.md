# 8.1 — `Executor` and `ExecutorService` Fundamentals

> **Goal:** Understand why Java provides executor-based task execution and how `Executor` / `ExecutorService` separate **task submission** from **thread management**.

---

## 1. Why Executors?

Low-level thread creation is simple:

```java
Thread t = new Thread(task);
t.start();
```

But production applications may execute thousands of tasks. Creating one new thread per task can lead to:

- Too many threads
- Higher memory consumption
- More context switching
- Difficult lifecycle management
- Difficult graceful shutdown
- Poor control over concurrency

The Executor framework lets us submit tasks without manually managing every worker thread.

```text
Application
    ↓
  Task
    ↓
 Executor / ExecutorService
    ↓
 Worker Threads
```

---

# 2. `Executor` Interface

`Executor` is the simplest abstraction for executing a `Runnable` task.

```java
public interface Executor {
    void execute(Runnable command);
}
```

The caller says:

> "Execute this task."

The executor decides **how and when** it is executed.

It could run the task:

- On a new thread
- On an existing worker thread
- On the caller thread
- Through a thread pool

---

# 3. Basic `Executor` Practice

```java
import java.util.concurrent.Executor;

public class ExecutorExample {

    public static void main(String[] args) {

        Executor executor = command -> {
            Thread thread = new Thread(command);
            thread.start();
        };

        executor.execute(() -> {
            System.out.println("Task executed by: "
                    + Thread.currentThread().getName());
        });
    }
}
```

### Key Point

`Executor` does **not** provide task result retrieval or lifecycle management by itself.

---

# 4. `ExecutorService`

`ExecutorService` extends `Executor` and provides a richer task-execution and lifecycle API.

Important capabilities:

```text
ExecutorService
   ├── execute()
   ├── submit()
   ├── shutdown()
   ├── shutdownNow()
   ├── isShutdown()
   ├── isTerminated()
   ├── awaitTermination()
   └── invokeAll() / invokeAny()
```

It can manage a pool of worker threads and coordinate shutdown.

---

# 5. Basic `ExecutorService` Practice ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorServiceExample {

    public static void main(String[] args) {

        ExecutorService executorService =
                Executors.newFixedThreadPool(2);

        executorService.execute(() ->
                System.out.println("Task 1: "
                        + Thread.currentThread().getName()));

        executorService.execute(() ->
                System.out.println("Task 2: "
                        + Thread.currentThread().getName()));

        executorService.shutdown();
    }
}
```

### What happens?

```text
newFixedThreadPool(2)
        ↓
Creates/manages up to 2 worker threads as needed
        ↓
Task 1 ──┐
         ├── Worker threads
Task 2 ──┘
        ↓
shutdown()
        ↓
No new tasks accepted
        ↓
Previously submitted tasks can finish
```

---

# 6. `Executor` vs `ExecutorService`

| Feature | `Executor` | `ExecutorService` |
|---|---|---|
| Execute `Runnable` | ✅ | ✅ |
| `execute()` | ✅ | ✅ |
| Submit tasks/results | ❌ | ✅ |
| `Future` support | ❌ | ✅ |
| Shutdown API | ❌ | ✅ |
| Await termination | ❌ | ✅ |
| Bulk operations | ❌ | ✅ |
| Lifecycle management | ❌ | ✅ |

### Interview Answer

> **`Executor` is the basic abstraction for task execution, while `ExecutorService` extends it with task submission, `Future` support, bulk execution and executor lifecycle management such as shutdown.**

---

# 7. `execute()` Practice

`execute()` accepts a `Runnable` and does not return a result.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecuteExample {

    public static void main(String[] args) {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        executor.execute(() ->
                System.out.println("Running task"));

        executor.shutdown();
    }
}
```

---

# 8. `submit()` Preview

`submit()` returns a `Future` and can be used with `Runnable` or `Callable`.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class SubmitPreview {

    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        Future<?> future = executor.submit(() ->
                System.out.println("Task submitted"));

        future.get();

        executor.shutdown();
    }
}
```

> Detailed `execute()` vs `submit()` is covered in **8.3**.

---

# 9. Graceful Shutdown Practice ⭐⭐⭐⭐⭐

Always consider the lifecycle of an `ExecutorService`.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class GracefulExecutorShutdown {

    public static void main(String[] args) throws InterruptedException {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        executor.submit(() -> {
            try {
                Thread.sleep(500);
                System.out.println("Task completed");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        executor.shutdown();

        if (!executor.awaitTermination(2, TimeUnit.SECONDS)) {
            executor.shutdownNow();
        }
    }
}
```

### Important

`shutdown()` means:

> Stop accepting new tasks, while previously submitted tasks may continue.

It does **not** mean "immediately kill every worker thread".

---

# 10. Production-Style Pattern

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

try {
    executor.submit(task);
} finally {
    executor.shutdown();
}
```

For applications with a long-lived executor, ownership and shutdown should be designed explicitly rather than creating and destroying pools repeatedly.

---

# 11. Practice — Fixed Pool Observation

Run this code and observe that multiple tasks are handled by a limited number of worker threads.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class FixedPoolPractice {

    public static void main(String[] args) {

        ExecutorService executor = Executors.newFixedThreadPool(3);

        for (int i = 1; i <= 8; i++) {
            int taskId = i;

            executor.execute(() -> {
                System.out.println("Task " + taskId
                        + " -> "
                        + Thread.currentThread().getName());
            });
        }

        executor.shutdown();
    }
}
```

### Observe

There are **8 tasks**, but the executor is configured with a pool size of **3**.

The number of concurrently active worker threads is therefore bounded by the executor's configured pool behavior rather than creating one application thread for every task.

---

# 12. Wrong Approach ❌

```java
for (int i = 0; i < 10_000; i++) {
    new Thread(task).start();
}
```

Why can this be problematic?

- Potentially huge thread count
- More memory usage
- More scheduling/context-switching overhead
- No central lifecycle management
- Harder resource control

A bounded executor can provide controlled concurrency.

---

# 13. Interview Scenario

### Question

> You receive 10,000 independent tasks from incoming requests. Would you create 10,000 threads?

### Strong Answer

> **No. I would normally use an appropriately sized executor or application-managed task executor. The executor separates task submission from worker-thread management, allows bounded concurrency, provides lifecycle control, and makes shutdown and monitoring easier. The pool size and queueing strategy should be selected based on whether the workload is CPU-bound, I/O-bound, latency-sensitive, and on the application's resource limits.**

---

# 14. Common Mistakes

### ❌ Mistake 1 — Forgetting shutdown

```java
ExecutorService executor = Executors.newFixedThreadPool(2);
executor.submit(task);
```

For a short-lived owner, failing to shut down the executor can keep application resources alive.

### ❌ Mistake 2 — Assuming `shutdown()` interrupts every task

It does not immediately stop already submitted tasks.

### ❌ Mistake 3 — Creating an executor for every request

```java
void handleRequest() {
    ExecutorService executor = Executors.newFixedThreadPool(10);
    // ...
}
```

This can create excessive pools and resource pressure. Pool ownership should normally be managed at an appropriate application/service scope.

### ❌ Mistake 4 — Treating executor size as universally optimal

There is no single pool size that is correct for every workload.

---

# 15. Quick Revision

```text
Executor
   ↓
Basic task execution abstraction
   ↓
execute(Runnable)

ExecutorService
   ↓
Executor + lifecycle + task submission/results + bulk operations
   ↓
execute()
submit()
shutdown()
shutdownNow()
Future
invokeAll()
invokeAny()
```

### Remember

> **Executor = execute tasks**  
> **ExecutorService = execute + submit + manage lifecycle**

---

# 🎯 Interview Questions

1. Why was the Executor framework introduced?
2. What is the difference between `Executor` and `ExecutorService`?
3. What does `execute()` do?
4. Why does `ExecutorService` extend `Executor`?
5. What is a thread pool?
6. Why should applications avoid uncontrolled thread creation?
7. What does `shutdown()` do?
8. Does `shutdown()` wait for tasks to finish?
9. When would you use `shutdownNow()`?
10. Why is executor ownership/lifecycle important?
11. What is the difference between `execute()` and `submit()`?
12. Why might a bounded executor be preferable in production?

---

# 🏆 2-Minute Interview Answer

> **"Java's Executor framework separates task submission from thread management. `Executor` is the basic abstraction with `execute(Runnable)`. `ExecutorService` extends it and adds task submission through `submit`, `Future` support, bulk operations, and lifecycle management such as `shutdown` and `awaitTermination`. Instead of creating a new thread for every task, an executor can reuse worker threads and control concurrency through a pool. In production I also consider executor ownership, queueing, rejection behavior, pool sizing, graceful shutdown and the workload characteristics before selecting an implementation."**

---

# 💻 Practice Checklist

- [ ] Run `ExecutorExample`
- [ ] Run `ExecutorServiceExample`
- [ ] Compare `execute()` and `submit()`
- [ ] Observe fixed pool reuse
- [ ] Practice graceful shutdown
- [ ] Create a 10-task fixed pool
- [ ] Explain why uncontrolled thread creation is risky
- [ ] Explain `Executor` vs `ExecutorService` in under 2 minutes

---

## Navigation

[← Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.2 — Thread Pool Fundamentals**