# 7.2 — Thread Creation using `Thread` Class

## 🎯 Objective

Understand how to create and start a Java thread using the `Thread` class, what `start()` actually does, why `run()` should not be called directly when creating a new thread, and the interview-level limitations of extending `Thread`.

---

## 1. Basic Concept

Java provides `java.lang.Thread` to represent a thread of execution.

The simplest approach is:

```java
class Worker extends Thread {
    @Override
    public void run() {
        System.out.println("Worker thread is running");
    }
}

public class Main {
    public static void main(String[] args) {
        Worker worker = new Worker();
        worker.start();
    }
}
```

Flow:

```text
Create Thread object
        ↓
     start()
        ↓
JVM schedules thread
        ↓
     run()
        ↓
Thread executes task
```

---

## 2. Why Override `run()`?

`run()` contains the work that the thread should execute.

```java
class Worker extends Thread {

    @Override
    public void run() {
        System.out.println("Task executed by: "
                + Thread.currentThread().getName());
    }
}
```

The JVM invokes the thread's `run()` method when the thread is started through `start()`.

> **Important:** Calling `run()` yourself is just a normal method invocation. It does not start a new thread.

---

## 3. `start()` vs `run()` ⭐⭐⭐⭐⭐

### Correct

```java
Thread t = new Worker();
t.start();
```

Conceptually:

```text
main thread
    │
    └── start()
          │
          └── new thread execution
                    │
                    └── run()
```

### Incorrect for starting a new thread

```java
Thread t = new Worker();
t.run();
```

This executes `run()` on the **current calling thread**.

```text
main thread
    │
    └── run()
          │
          └── task executes on main thread
```

### Interview answer

> `start()` asks the JVM to start a new thread of execution, which eventually executes `run()`. Calling `run()` directly does not create a new thread; it behaves like an ordinary method call on the current thread.

---

## 4. Thread Name

Every thread has a name.

```java
class Worker extends Thread {

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }
}

public class Main {
    public static void main(String[] args) {
        Worker worker = new Worker();
        worker.setName("payment-worker");
        worker.start();
    }
}
```

Useful APIs:

```java
thread.getName();
thread.setName("worker-1");
Thread.currentThread();
Thread.currentThread().getName();
```

Naming threads is useful for debugging, logging and production observability.

---

## 5. Thread ID

A thread also has an ID:

```java
long id = Thread.currentThread().getId();
```

The ID is useful for diagnostics but should not be treated as a business identifier.

---

## 6. Complete Example

```java
class Worker extends Thread {

    @Override
    public void run() {
        for (int i = 1; i <= 5; i++) {
            System.out.println(
                    Thread.currentThread().getName()
                            + " -> " + i
            );
        }
    }
}

public class Main {

    public static void main(String[] args) {
        Worker worker = new Worker();
        worker.setName("worker-1");

        worker.start();

        System.out.println(
                "Main thread: "
                        + Thread.currentThread().getName()
        );
    }
}
```

### Important

The output order between `main` and `worker-1` is not guaranteed because scheduling is controlled by the JVM/OS and runtime environment.

---

## 7. Creating Multiple Threads

```java
class Worker extends Thread {

    private final String taskName;

    Worker(String taskName) {
        this.taskName = taskName;
    }

    @Override
    public void run() {
        System.out.println(
                taskName + " executed by "
                        + Thread.currentThread().getName()
        );
    }
}

public class Main {

    public static void main(String[] args) {
        Thread t1 = new Worker("Task-A");
        Thread t2 = new Worker("Task-B");

        t1.start();
        t2.start();
    }
}
```

Possible execution:

```text
Task-A → worker-1
Task-B → worker-2
```

But execution order is not guaranteed.

---

## 8. Thread Lifecycle Connection

After constructing a `Thread` object:

```java
Thread t = new Worker();
```

it has not started executing.

After:

```java
t.start();
```

it becomes eligible for execution according to Java's thread-state model.

A thread can subsequently terminate when `run()` completes.

High-level flow:

```text
NEW
 ↓
start()
 ↓
RUNNABLE
 ↓
execution / scheduling
 ↓
TERMINATED
```

> Java's `Thread.State` enum does **not** define separate `RUNNING` and `READY` states. Both are represented by `RUNNABLE` from the Java API perspective.

---

## 9. Can `start()` Be Called Twice? ⭐⭐⭐⭐⭐

No.

```java
Thread t = new Worker();

t.start();
t.start(); // IllegalThreadStateException
```

A `Thread` instance cannot be restarted after it has already been started.

If you need another execution, create another `Thread` instance.

```java
new Worker().start();
new Worker().start();
```

### Interview answer

> A Java `Thread` object represents one execution lifecycle. Once started, it cannot be started again. Calling `start()` a second time results in `IllegalThreadStateException`.

---

## 10. What Happens if `run()` Is Called Twice?

Unlike `start()`, `run()` is just a method.

```java
Worker worker = new Worker();

worker.run();
worker.run();
```

Both calls execute synchronously on the calling thread.

```text
main
 ├── run()
 └── run()
```

No new worker thread is created.

---

## 11. Why Extending `Thread` Is Often Not Preferred

Although extending `Thread` is easy to understand, it couples the task to the thread object.

```java
class Worker extends Thread {
    @Override
    public void run() {
        // task
    }
}
```

Java supports single class inheritance.

If your class already needs to extend another class:

```java
class PaymentService extends SomeBaseService {
    // cannot also extend Thread
}
```

you cannot extend `Thread` as well.

This is one reason `Runnable` is an important alternative and is covered in **7.3**.

---

## 12. Thread vs Task ⭐⭐⭐⭐⭐

A useful design distinction:

```text
Task
 ↓
What should be executed?

Thread
 ↓
Who executes the task?
```

Extending `Thread` combines these concepts:

```text
Thread + Task
```

Using `Runnable` separates them:

```text
Runnable = Task
Thread   = Execution mechanism
```

This separation becomes even more important with executors and thread pools.

---

## 13. Constructor-Based Thread Creation

`Thread` also provides constructors that accept a name.

```java
class Worker extends Thread {

    Worker(String name) {
        super(name);
    }

    @Override
    public void run() {
        System.out.println(getName());
    }
}
```

Usage:

```java
Thread worker = new Worker("order-worker");
worker.start();
```

---

## 14. Thread Priority — Important Caveat

Java exposes:

```java
thread.setPriority(Thread.MAX_PRIORITY);
```

with values from:

```text
Thread.MIN_PRIORITY
Thread.NORM_PRIORITY
Thread.MAX_PRIORITY
```

However, **priority is not a correctness mechanism** and should not be used to guarantee execution order.

Scheduling behavior depends on the JVM and operating system.

---

## 15. `Thread.currentThread()`

Inside `run()`:

```java
System.out.println(Thread.currentThread().getName());
```

This returns the thread that is currently executing the code.

Example:

```java
class Worker extends Thread {

    @Override
    public void run() {
        Thread current = Thread.currentThread();
        System.out.println(current.getName());
    }
}
```

This is especially useful for logging and debugging concurrent applications.

---

## 16. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — Calling `run()` instead of `start()`

```java
worker.run();
```

❌ Does not create a new thread.

### Mistake 2 — Starting the same object twice

```java
worker.start();
worker.start();
```

❌ Throws `IllegalThreadStateException`.

### Mistake 3 — Assuming execution order

```java
t1.start();
t2.start();
```

❌ Does not guarantee `t1` finishes before `t2`.

### Mistake 4 — Using priority as synchronization

```java
t1.setPriority(...);
```

❌ Priority does not provide a correctness guarantee.

### Mistake 5 — Assuming `start()` immediately means CPU execution

`start()` makes the thread eligible to run; scheduling and actual execution timing are not controlled by the Java application in a deterministic way.

---

## 17. Interview Questions

### Q1. How do you create a thread by extending `Thread`?

Create a subclass, override `run()`, instantiate it and call `start()`.

### Q2. What is the difference between `start()` and `run()`?

`start()` initiates a new thread of execution; `run()` is the task method and a direct call executes synchronously on the caller thread.

### Q3. Can you call `start()` twice?

No. It throws `IllegalThreadStateException`.

### Q4. Can you call `run()` twice?

Yes. It is a normal method call, so it can be invoked multiple times, synchronously on the calling thread.

### Q5. Why is `Runnable` often preferred over extending `Thread`?

It separates task from execution mechanism and avoids consuming the class's single inheritance slot.

### Q6. Is thread execution order guaranteed?

No. Calling `start()` in a particular order does not guarantee completion or execution order.

### Q7. Does `start()` directly call `run()` on the main thread?

No. It starts the thread lifecycle and the new thread eventually executes `run()`.

### Q8. What exception occurs when `start()` is called twice?

`IllegalThreadStateException`.

### Q9. Does thread priority guarantee scheduling?

No. It is only a scheduling-related hint and is platform/runtime dependent.

### Q10. What is the difference between a task and a thread?

A task describes work to execute; a thread is an execution mechanism. `Runnable` helps separate those concerns.

---

## 18. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"We can create a Java thread by extending `Thread` and overriding its `run()` method. We then create an instance and call `start()`. The important distinction is that `start()` initiates a new thread of execution, while directly calling `run()` is just a normal method call on the current thread. A Thread instance cannot be started twice; a second `start()` throws `IllegalThreadStateException`. Extending Thread is simple for learning, but in production design we usually separate the task from the execution mechanism using `Runnable` or higher-level executor APIs. Thread names are useful for debugging, and thread priority should never be used to guarantee correctness or execution order." 

---

## 19. Quick Revision

```text
Extend Thread
      ↓
Override run()
      ↓
Create object
      ↓
Call start()
      ↓
New thread becomes eligible to execute
      ↓
run() executes on that thread
```

### Remember

```text
start() → starts thread lifecycle
run()   → task method
```

```text
start() twice → IllegalThreadStateException
run() twice   → normal method calls
```

```text
Thread = execution mechanism
Runnable = task abstraction
```

---

## 20. Completion Checklist

- [x] `Thread` class introduction
- [x] Extending `Thread`
- [x] Overriding `run()`
- [x] `start()` behavior
- [x] `start()` vs `run()`
- [x] Thread naming
- [x] Multiple threads
- [x] Thread lifecycle connection
- [x] Starting a thread twice
- [x] Calling `run()` multiple times
- [x] Thread vs task distinction
- [x] Thread priority caveat
- [x] `currentThread()`
- [x] Common mistakes
- [x] Interview questions
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.1 — Process vs Thread](../01-Process-vs-Thread/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.3 — Thread Creation using `Runnable`**