# 7.7 — `start()` vs `run()`

## 🎯 Objective

Understand the most common Java multithreading interview question:

```java
thread.start();
```

vs

```java
thread.run();
```

The key difference is simple:

> **`start()` creates a new thread of execution; directly calling `run()` is just a normal method call in the current thread.**

---

# 1. `start()`

When you call:

```java
thread.start();
```

Java starts a new thread of execution and that new thread eventually invokes the thread's `run()` method.

Conceptually:

```text
main thread
    |
    | start()
    ↓
new worker thread
    |
    ↓
run()
```

Example:

```java
public class StartDemo {

    public static void main(String[] args) {

        Thread worker = new Thread(() -> {
            System.out.println("Running in: "
                    + Thread.currentThread().getName());
        }, "worker");

        worker.start();

        System.out.println("Running in: "
                + Thread.currentThread().getName());
    }
}
```

Possible output:

```text
Running in: main
Running in: worker
```

or the reverse order.

The exact order is not guaranteed because the threads execute concurrently.

---

# 2. `run()`

`run()` is the method containing the task's code.

If you directly call:

```java
thread.run();
```

you are simply invoking a normal instance method.

No new thread is created.

Example:

```java
public class RunDemo {

    public static void main(String[] args) {

        Thread worker = new Thread(() -> {
            System.out.println("Running in: "
                    + Thread.currentThread().getName());
        }, "worker");

        worker.run();

        System.out.println("Running in: "
                + Thread.currentThread().getName());
    }
}
```

Output:

```text
Running in: main
Running in: main
```

The worker thread was **not started**.

---

# 3. Internal Concept ⭐⭐⭐⭐⭐

The important mental model is:

### With `start()`

```text
main
  |
  | start()
  ↓
JVM creates/schedules new execution thread
  |
  ↓
new thread executes run()
```

### With direct `run()`

```text
main
  |
  | run()
  ↓
run() executes normally in main
```

Therefore:

```text
start() → new thread → run()
run()   → current thread → run()
```

---

# 4. Why Does `start()` Eventually Invoke `run()`?

A `Thread` object represents a thread of execution and contains the task represented by `run()`.

Calling `start()` tells the JVM to arrange for the thread to begin execution.

The new thread then executes the `run()` method.

You should think of:

```java
thread.start();
```

as the **thread creation/start mechanism**, while:

```java
thread.run();
```

is simply the **task method invocation**.

---

# 5. Most Important Difference ⭐⭐⭐⭐⭐

| `start()` | `run()` |
|---|---|
| Starts a new thread of execution | Does not start a new thread |
| JVM schedules the new thread | Normal method invocation |
| `run()` executes in the new thread | `run()` executes in current thread |
| Enables concurrent execution | No concurrency by itself |
| Can only start a `Thread` instance once | Can be invoked like a normal method |
| Calling `start()` twice throws `IllegalThreadStateException` | Repeated direct calls are ordinary method calls |

---

# 6. Thread Name Proof ⭐⭐⭐⭐⭐

The easiest way to prove the difference is with:

```java
Thread.currentThread().getName()
```

### `start()`

```java
Thread worker = new Thread(() -> {
    System.out.println(Thread.currentThread().getName());
}, "worker");

worker.start();
```

The task prints:

```text
worker
```

### `run()`

```java
Thread worker = new Thread(() -> {
    System.out.println(Thread.currentThread().getName());
}, "worker");

worker.run();
```

The task prints:

```text
main
```

This is one of the best interview demonstrations.

---

# 7. `start()` Does Not Guarantee Immediate Execution ⭐⭐⭐⭐⭐

Consider:

```java
Thread worker = new Thread(() -> {
    System.out.println("Worker");
});

worker.start();
System.out.println("Main");
```

Possible outputs:

```text
Worker
Main
```

or:

```text
Main
Worker
```

Calling `start()` makes the new thread eligible for scheduling; it does not guarantee which thread prints first.

---

# 8. `run()` Does Guarantee Current-Thread Execution

If the main thread directly calls:

```java
worker.run();
```

then `run()` executes as part of the current call stack.

So:

```text
main
 ↓
worker.run()
 ↓
run() body
 ↓
return to main
```

There is no separate worker thread involved.

---

# 9. Calling `start()` Twice ⭐⭐⭐⭐⭐

This is invalid:

```java
Thread worker = new Thread(() -> {
    System.out.println("Task");
});

worker.start();
worker.start();
```

The second `start()` throws:

```text
java.lang.IllegalThreadStateException
```

### Why?

A `Thread` instance represents one thread execution lifecycle. Once that thread has been started, the same `Thread` object cannot be started again.

---

# 10. Can `run()` Be Called Multiple Times?

Yes.

```java
Thread worker = new Thread(() -> {
    System.out.println("Task");
});

worker.run();
worker.run();
worker.run();
```

These are simply three ordinary method calls in the current thread.

If `main` calls them, all three execute in `main`.

```text
main → run()
main → run()
main → run()
```

No new thread is created.

---

# 11. Does Direct `run()` Change Thread State?

This is a useful interview trap.

```java
Thread worker = new Thread(() -> {
    System.out.println(Thread.currentThread().getName());
}, "worker");

System.out.println(worker.getState());
worker.run();
System.out.println(worker.getState());
```

The `Thread` object's lifecycle is not started merely because its `run()` method was directly invoked.

The task executes in the caller thread.

Conceptually:

```text
Thread object → NEW
       |
       | direct run()
       ↓
run() executes in caller
       |
Thread object itself was never started
```

---

# 12. `start()` and `run()` with `Runnable`

The same principle applies when using `Runnable`.

```java
Runnable task = () -> {
    System.out.println(Thread.currentThread().getName());
};

Thread worker = new Thread(task, "worker");
worker.start();
```

The new thread executes the `Runnable` task.

If you instead do:

```java
worker.run();
```

the current thread executes the task.

---

# 13. Why `Runnable` Is Still Important

`Runnable` represents the **task**.

`Thread` represents the **thread of execution**.

```text
Runnable
   ↓
task / work

Thread
   ↓
execution mechanism
```

Example:

```java
Runnable task = () -> {
    System.out.println("Processing order");
};

Thread worker = new Thread(task, "order-worker");
worker.start();
```

This separation is useful because the task and execution mechanism are different concepts.

---

# 14. Common Interview Question

### Q: What happens when `start()` is called?

A strong answer:

> Calling `start()` causes the JVM to start the thread's execution. The new thread is scheduled for execution and eventually invokes `run()`. It does not guarantee immediate execution or ordering relative to the calling thread.

---

# 15. Common Interview Question

### Q: What happens when `run()` is called directly?

Answer:

> Directly calling `run()` does not create or start a new thread. It is a normal method call, so the `run()` body executes in the thread that called it.

---

# 16. Practice Code ⭐⭐⭐⭐⭐

Create:

`StartVsRunPractice.java`

```java
public class StartVsRunPractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            System.out.println(
                    "Task executed by: "
                            + Thread.currentThread().getName()
            );
        }, "worker-thread");

        System.out.println("Main thread: "
                + Thread.currentThread().getName());

        System.out.println("Before start: "
                + worker.getState());

        worker.start();

        worker.join();

        System.out.println("After start/join: "
                + worker.getState());

        System.out.println("--------------------");

        Thread directRun = new Thread(() -> {
            System.out.println(
                    "Task executed by: "
                            + Thread.currentThread().getName()
            );
        }, "direct-run-thread");

        System.out.println("Before direct run: "
                + directRun.getState());

        directRun.run();

        System.out.println("After direct run: "
                + directRun.getState());
    }
}
```

### Expected conceptual output

```text
Main thread: main
Before start: NEW
Task executed by: worker-thread
After start/join: TERMINATED
--------------------
Before direct run: NEW
Task executed by: main
After direct run: NEW
```

The important observation is:

```text
start() → worker-thread
run()   → main
```

---

# 17. Practice Code — Calling `run()` Multiple Times

```java
public class RunMultipleTimesPractice {

    public static void main(String[] args) {

        Thread worker = new Thread(() -> {
            System.out.println(
                    "Running in: "
                            + Thread.currentThread().getName()
            );
        }, "worker");

        worker.run();
        worker.run();
        worker.run();
    }
}
```

Expected:

```text
Running in: main
Running in: main
Running in: main
```

### Why?

Because every `run()` call is just a normal method invocation.

---

# 18. Practice Code — Calling `start()` Twice

```java
public class StartTwicePractice {

    public static void main(String[] args) {

        Thread worker = new Thread(() -> {
            System.out.println("Worker running");
        });

        worker.start();

        // Invalid: same Thread object cannot be started again
        worker.start();
    }
}
```

Expected exception:

```text
java.lang.IllegalThreadStateException
```

---

# 19. Practice Exercise — Predict the Output ⭐⭐⭐⭐⭐

```java
public class PredictOutput {

    public static void main(String[] args) {

        Thread t = new Thread(() -> {
            System.out.println("A: "
                    + Thread.currentThread().getName());
        }, "worker");

        t.run();

        System.out.println("B: "
                + Thread.currentThread().getName());
    }
}
```

### Question

Will `A` print `worker`?

**Answer: No.**

It prints:

```text
A: main
B: main
```

because `run()` was called directly.

---

# 20. Practice Exercise — Predict the Ordering

```java
Thread t = new Thread(() -> {
    System.out.println("Worker");
});

t.start();
System.out.println("Main");
```

Possible:

```text
Worker
Main
```

or:

```text
Main
Worker
```

### Question

Is either ordering guaranteed?

**Answer: No.**

---

# 21. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1

```java
t.run();
```

means a new thread starts.

❌ False.

### Mistake 2

```java
t.start();
```

means `run()` executes immediately before the next statement.

❌ False.

### Mistake 3

Calling `start()` twice restarts the thread.

❌ False.

### Mistake 4

Calling `run()` twice creates two worker threads.

❌ False.

### Mistake 5

`run()` and `start()` are interchangeable.

❌ False.

---

# 22. Interview Traps ⭐⭐⭐⭐⭐

### Trap 1 — `run()` creates a thread

❌ No.

It is an ordinary method call when invoked directly.

### Trap 2 — `start()` calls `run()` on the main thread

❌ No.

The task's `run()` method is executed by the newly started thread.

### Trap 3 — `start()` guarantees output order

❌ No.

Scheduling is nondeterministic.

### Trap 4 — Same Thread object can be started again

❌ No.

Second `start()` throws `IllegalThreadStateException`.

### Trap 5 — `run()` changes the Thread object to RUNNABLE

❌ Direct invocation does not start that `Thread` object.

---

# 23. Comparison Table ⭐⭐⭐⭐⭐

| Point | `start()` | Direct `run()` |
|---|---|---|
| New thread? | ✅ Yes | ❌ No |
| Executes task? | ✅ Yes | ✅ Yes |
| Execution thread | New thread | Caller thread |
| Concurrent execution? | ✅ Possible | ❌ No |
| Can invoke repeatedly? | ❌ No on same Thread instance | ✅ Yes |
| Second call | `IllegalThreadStateException` | Normal method call |
| Scheduler involved in starting execution? | ✅ Yes | ❌ No new thread scheduling |
| Typical use | Multithreading | Rarely direct for Thread task execution |

---

# 24. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"The main difference is that `start()` actually starts a new thread of execution, while directly calling `run()` is just a normal method call. When we call `start()`, the JVM makes the new thread eligible for scheduling and that thread eventually executes the `run()` method. Because two threads can execute concurrently, the order of output is not guaranteed. If we call `run()` directly, the method executes in the caller thread, so no new thread is created. A `Thread` instance can only be started once; calling `start()` a second time throws `IllegalThreadStateException`. In short, `start()` gives us a new execution thread, whereas `run()` by itself only executes the task code in the current thread."**

---

# 25. Quick Revision

```text
start()
  ↓
New thread
  ↓
Scheduler
  ↓
run()
```

versus:

```text
run()
  ↓
Normal method call
  ↓
Current thread
```

### One-Line Memory Trick

> **`start()` starts a thread; `run()` runs code.**

### Golden Rule ⭐⭐⭐⭐⭐

```text
start() → NEW THREAD
run()   → CURRENT THREAD
```

---

# 26. Completion Checklist

- [x] `start()` explained
- [x] `run()` explained
- [x] New thread vs current thread
- [x] Internal execution model
- [x] Scheduling behavior
- [x] Thread-name proof
- [x] `start()` twice
- [x] `run()` multiple times
- [x] `Runnable` relationship
- [x] Practice code
- [x] Output prediction exercises
- [x] Interview traps
- [x] Comparison table
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.6 — Thread Naming & Basic Thread APIs](../06-Thread-Naming-and-Basic-Thread-APIs/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.8 — `sleep()`**