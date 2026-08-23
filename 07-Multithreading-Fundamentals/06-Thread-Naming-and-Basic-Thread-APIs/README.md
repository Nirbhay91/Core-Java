# 7.6 — Thread Naming & Basic Thread APIs

## 🎯 Objective

Understand how to name threads, inspect basic thread information, control common thread properties, and use core APIs such as `getName()`, `setName()`, `getId()`, `getPriority()`, `setPriority()`, `isAlive()`, `currentThread()`, and `isDaemon()`.

---

## 1. Why Thread Naming Matters

In real applications, many threads may execute concurrently. Meaningful names make logs, debugging, thread dumps, and production troubleshooting much easier.

Instead of:

```text
Thread-0
Thread-1
Thread-2
```

prefer names such as:

```text
payment-worker-1
order-consumer-1
notification-worker
```

---

## 2. Default Thread Name

If no name is supplied, Java assigns a default name.

```java
Thread t = new Thread(() -> {
    System.out.println("Task");
});

System.out.println(t.getName());
```

Typical output:

```text
Thread-0
```

The exact default numbering depends on the JVM/application context.

---

## 3. Set Thread Name in Constructor ⭐⭐⭐⭐

You can provide a name while creating the thread:

```java
Thread worker = new Thread(
        () -> System.out.println("Processing"),
        "payment-worker"
);
```

Then:

```java
System.out.println(worker.getName());
```

Output:

```text
payment-worker
```

This is usually cleaner than creating the thread first and renaming it later.

---

## 4. `getName()`

Returns the thread's current name.

```java
Thread current = Thread.currentThread();
System.out.println(current.getName());
```

---

## 5. `setName()`

Changes the name of a thread.

```java
Thread worker = new Thread(() -> {
    System.out.println(Thread.currentThread().getName());
});

worker.setName("payment-worker");
worker.start();
```

Output:

```text
payment-worker
```

### Best practice

Use descriptive names that communicate the responsibility of the thread.

---

## 6. `currentThread()` ⭐⭐⭐⭐⭐

```java
Thread.currentThread()
```

returns the `Thread` object representing the thread that is currently executing the code.

Example:

```java
public class CurrentThreadDemo {

    public static void main(String[] args) {
        System.out.println(Thread.currentThread().getName());
    }
}
```

Typical output:

```text
main
```

Inside a worker:

```java
Thread worker = new Thread(() -> {
    Thread current = Thread.currentThread();
    System.out.println(current.getName());
}, "worker");

worker.start();
```

---

## 7. `getId()`

Returns the identifier associated with the thread.

```java
Thread t = new Thread(() -> {});

System.out.println(t.getId());
```

The ID is useful for diagnostics, but application logic should generally prefer meaningful names rather than depending on numeric thread IDs.

> Note: Modern Java versions also expose `threadId()`; `getId()` is retained for compatibility.

---

## 8. `getPriority()` ⭐⭐⭐⭐

Every thread has a priority value.

```java
int priority = thread.getPriority();
```

Java defines:

```java
Thread.MIN_PRIORITY   // 1
Thread.NORM_PRIORITY  // 5
Thread.MAX_PRIORITY   // 10
```

The default priority is normally `NORM_PRIORITY` unless inherited/changed.

---

## 9. `setPriority()`

You can request a different priority:

```java
thread.setPriority(Thread.MAX_PRIORITY);
```

or:

```java
thread.setPriority(8);
```

The value must be within the valid Java priority range.

### Important interview point ⭐⭐⭐⭐⭐

Thread priority is a **scheduling hint**, not a guarantee that a higher-priority thread will always execute first or receive more CPU time in a predictable way.

Do not build correctness logic around thread priority.

---

## 10. `isAlive()` ⭐⭐⭐⭐⭐

Checks whether a thread has been started and has not yet terminated.

```java
Thread worker = new Thread(() -> {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

System.out.println(worker.isAlive()); // false

worker.start();
System.out.println(worker.isAlive()); // normally true here
```

After completion:

```java
System.out.println(worker.isAlive()); // false
```

### Remember

```text
NEW         → isAlive() = false
RUNNING/eligible execution → true
TERMINATED  → false
```

The precise intermediate observation is timing-dependent.

---

## 11. `isDaemon()` and `setDaemon()` ⭐⭐⭐⭐

A daemon thread is a background thread associated with the application.

Check:

```java
thread.isDaemon();
```

Set before starting:

```java
thread.setDaemon(true);
thread.start();
```

### Critical rule

`setDaemon(true)` must be called **before** `start()`.

Calling it after the thread has started results in `IllegalThreadStateException`.

---

## 12. Daemon vs User Thread

A JVM can terminate when no live **non-daemon** threads remain.

This does not mean daemon threads are gracefully completed first. They may be stopped as the JVM shuts down.

Therefore, do not use daemon threads for critical work that must complete, such as mandatory transaction processing or reliable persistence.

---

## 13. `start()`

Starts a new thread of execution.

```java
Thread t = new Thread(() -> {
    System.out.println("Worker");
});

t.start();
```

Important:

```text
start() → new thread execution
run()   → normal method invocation if called directly
```

Calling `start()` more than once on the same `Thread` instance causes `IllegalThreadStateException`.

---

## 14. `run()`

`run()` contains the task code.

```java
Thread t = new Thread(() -> {
    System.out.println("Task");
});
```

Calling:

```java
t.run();
```

does **not** start a new thread. It executes the method in the current thread.

This distinction is covered separately in **7.7 — `start()` vs `run()`**.

---

## 15. `sleep()`

Temporarily pauses the currently executing thread for a specified duration.

```java
Thread.sleep(1000);
```

It puts the current thread into `TIMED_WAITING` for the requested duration, subject to interruption and scheduling.

### Important

`sleep()`:

- is static
- affects the current thread
- does not release intrinsic monitor locks
- can throw `InterruptedException`

Detailed treatment is covered in **7.8 — `sleep()`**.

---

## 16. `join()`

Allows one thread to wait for another thread to finish.

```java
worker.start();
worker.join();
```

The calling thread waits until `worker` terminates, unless a timed join or interruption changes the situation.

Detailed treatment is covered in **7.9 — `join()`**.

---

## 17. `yield()`

```java
Thread.yield();
```

is a scheduler hint that the current thread is willing to yield execution opportunity.

It does not guarantee that another thread will run.

It does not release a monitor lock held by the thread.

The Java thread remains in the `RUNNABLE` state at the Java API level.

Detailed treatment is covered in **7.10 — `yield()`**.

---

## 18. `isInterrupted()` vs `interrupted()`

Two commonly confused APIs:

### `isInterrupted()`

```java
thread.isInterrupted();
```

Checks the target thread's interrupt status without clearing it.

### `Thread.interrupted()`

```java
Thread.interrupted();
```

Checks the **current thread's** interrupt status and clears the status when it returns `true`.

This distinction is important for interview questions and production code.

Detailed interruption is covered later in Chapter 7.

---

## 19. `interrupt()`

```java
thread.interrupt();
```

requests interruption of the target thread.

It does not forcibly kill the thread.

If the target is blocked in interruptible operations such as `sleep()`, `wait()`, or `join()`, it may receive `InterruptedException`.

Otherwise, the interrupt status may be set and the target code should respond cooperatively.

---

## 20. Basic API Summary ⭐⭐⭐⭐⭐

| API | Purpose |
|---|---|
| `getName()` | Read thread name |
| `setName()` | Change thread name |
| `currentThread()` | Get currently executing thread |
| `getId()` | Get thread identifier |
| `threadId()` | Get modern thread identifier |
| `getPriority()` | Read priority |
| `setPriority()` | Set scheduling priority hint |
| `isAlive()` | Check whether thread is alive |
| `isDaemon()` | Check daemon status |
| `setDaemon()` | Mark thread daemon before start |
| `start()` | Start new thread execution |
| `run()` | Task method; direct call does not create a new thread |
| `sleep()` | Timed pause of current thread |
| `join()` | Wait for another thread |
| `yield()` | Scheduler hint |
| `interrupt()` | Request interruption |
| `isInterrupted()` | Check target interrupt status |
| `interrupted()` | Check and clear current thread interrupt status |
| `getState()` | Observe current Java-level thread state |

---

# 21. Practice Code ⭐⭐⭐⭐⭐

Create:

`ThreadBasicApiPractice.java`

```java
public class ThreadBasicApiPractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            Thread current = Thread.currentThread();

            System.out.println("Name       : " + current.getName());
            System.out.println("ID         : " + current.getId());
            System.out.println("Priority   : " + current.getPriority());
            System.out.println("Daemon     : " + current.isDaemon());
            System.out.println("Alive      : " + current.isAlive());
            System.out.println("State      : " + current.getState());

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "payment-worker");

        System.out.println("Before start");
        System.out.println("Name     : " + worker.getName());
        System.out.println("Priority : " + worker.getPriority());
        System.out.println("Daemon   : " + worker.isDaemon());
        System.out.println("Alive    : " + worker.isAlive());
        System.out.println("State    : " + worker.getState());

        worker.setPriority(Thread.NORM_PRIORITY + 1);

        worker.start();

        System.out.println("After start");
        System.out.println("Alive    : " + worker.isAlive());
        System.out.println("State    : " + worker.getState());

        worker.join();

        System.out.println("After completion");
        System.out.println("Alive    : " + worker.isAlive());
        System.out.println("State    : " + worker.getState());
    }
}
```

---

# 22. Practice Code — Naming Threads ⭐⭐⭐⭐⭐

```java
public class ThreadNamingPractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread paymentWorker = new Thread(
                () -> process("payment"),
                "payment-worker"
        );

        Thread notificationWorker = new Thread(
                () -> process("notification"),
                "notification-worker"
        );

        paymentWorker.start();
        notificationWorker.start();

        paymentWorker.join();
        notificationWorker.join();
    }

    private static void process(String type) {
        System.out.println(
                Thread.currentThread().getName()
                        + " processing " + type
        );
    }
}
```

Possible output order can vary:

```text
payment-worker processing payment
notification-worker processing notification
```

or the reverse.

---

# 23. Practice Exercise — Daemon Thread

Create a daemon thread:

```java
Thread background = new Thread(() -> {
    while (true) {
        System.out.println("Background work");

        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        }
    }
}, "background-worker");

background.setDaemon(true);
background.start();
```

Then let `main` finish.

Observe that the JVM does not remain alive solely because of this daemon thread.

---

# 24. Practice Exercise — `isAlive()`

Predict the output before running:

```java
Thread t = new Thread(() -> {
    try {
        Thread.sleep(500);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

System.out.println(t.isAlive());

t.start();
System.out.println(t.isAlive());

t.join();
System.out.println(t.isAlive());
```

Expected conceptual result:

```text
false
true
false
```

The middle observation is timing-dependent for very short tasks, but with the deliberate sleep it should normally be `true`.

---

# 25. Practice Exercise — Priority

Create three threads:

```text
LOW       → MIN_PRIORITY
NORMAL    → NORM_PRIORITY
HIGH      → MAX_PRIORITY
```

Print their priorities and observe execution.

### Question

Does `HIGH` guarantee execution before `LOW`?

**Answer: No.**

Priority is not a correctness or ordering guarantee.

---

# 26. Practice Exercise — Current Thread

Write a program that prints:

```text
Main thread: main
Worker thread: worker-1
```

using only:

```java
Thread.currentThread().getName()
```

This reinforces the distinction between the current executing thread and a `Thread` object reference.

---

# 27. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — Calling `run()` to create a new thread

❌ Wrong.

```java
t.run();
```

is a normal method call.

Use:

```java
t.start();
```

### Mistake 2 — Setting daemon after start

❌ Wrong:

```java
t.start();
t.setDaemon(true);
```

This can throw `IllegalThreadStateException`.

### Mistake 3 — Treating priority as guaranteed scheduling order

❌ Wrong.

Priority is only a scheduling-related hint and behavior is platform/JVM dependent.

### Mistake 4 — Using thread ID as business identity

❌ Avoid.

Use descriptive thread names for application-level diagnostics.

### Mistake 5 — Thinking `sleep()` pauses the entire JVM

❌ No.

Only the current thread sleeps.

### Mistake 6 — Thinking `interrupt()` kills the thread

❌ No.

It requests interruption; application code must cooperate.

---

# 28. Interview Questions

### Q1. How do you name a thread?

```java
new Thread(task, "worker-name");
```

or:

```java
thread.setName("worker-name");
```

### Q2. How do you get the current thread?

```java
Thread.currentThread();
```

### Q3. How do you check whether a thread is alive?

```java
thread.isAlive();
```

### Q4. When can `setDaemon(true)` be called?

Before `start()`.

### Q5. Does a high-priority thread always execute first?

No.

### Q6. Difference between `isInterrupted()` and `Thread.interrupted()`?

`isInterrupted()` checks a thread's interrupt status without clearing it. `Thread.interrupted()` checks the current thread and clears the interrupt status when it is set.

### Q7. Does `run()` create a new thread?

No. Direct invocation runs in the current thread.

### Q8. What happens if `start()` is called twice?

The same `Thread` instance cannot be started twice; a subsequent `start()` results in `IllegalThreadStateException`.

### Q9. Why are thread names useful?

They make logs, debugging, monitoring, and thread dumps much easier to understand.

### Q10. What is the difference between daemon and non-daemon threads?

The JVM can terminate once no live non-daemon threads remain; daemon threads are background threads and should not be relied upon for critical completion work.

---

# 29. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Java provides several basic Thread APIs for controlling and inspecting threads. We can assign meaningful names using the constructor or `setName()`, and retrieve the current executing thread using `Thread.currentThread()`. `isAlive()` tells whether a thread has started and has not terminated. Thread priority ranges from 1 to 10, with 5 as normal priority, but priority is only a scheduling hint and should not be used for correctness. A thread can be marked daemon using `setDaemon(true)`, but this must happen before `start()`. `start()` creates a new thread of execution, whereas directly calling `run()` is just a normal method call. Other common APIs include `sleep()`, `join()`, `yield()`, `interrupt()`, and state/interrupt-status methods. Meaningful thread names are especially useful in production debugging and thread dumps."**

---

# 30. Quick Revision

```text
Naming
  ↓
getName() / setName()

Current thread
  ↓
Thread.currentThread()

Identity
  ↓
getId() / threadId()

Priority
  ↓
getPriority() / setPriority()

Status
  ↓
isAlive() / getState()

Daemon
  ↓
isDaemon() / setDaemon()
  ↓
setDaemon() BEFORE start()

Execution
  ↓
start() → new thread
run()   → normal method call

Coordination
  ↓
sleep() / join() / yield()

Interruption
  ↓
interrupt()
```

### Must Remember ⭐

```text
Thread name → debugging
Priority → hint, NOT guarantee
Daemon → set before start()
start() → new thread
run() → current thread
interrupt() → request, NOT force-kill
```

---

# 31. Completion Checklist

- [x] Thread naming
- [x] `getName()`
- [x] `setName()`
- [x] `currentThread()`
- [x] `getId()` / `threadId()`
- [x] Thread priority
- [x] `getPriority()` / `setPriority()`
- [x] `isAlive()`
- [x] Daemon threads
- [x] `isDaemon()` / `setDaemon()`
- [x] `start()` overview
- [x] `run()` overview
- [x] `sleep()` overview
- [x] `join()` overview
- [x] `yield()` overview
- [x] Interrupt APIs overview
- [x] Practice code
- [x] Practice exercises
- [x] Interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.5 — `Thread.State` and `RUNNABLE`](../05-Thread-State-and-Runnable/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.7 — `start()` vs `run()`**