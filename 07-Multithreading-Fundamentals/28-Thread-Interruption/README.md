# 7.28 — Thread Interruption

## 🎯 Objective

Understand Java thread interruption: what `interrupt()` actually does, how interrupted status works, how blocking methods react, and how to write interruption-friendly code.

> **Golden rule:** `interrupt()` does not forcibly kill a thread. It requests interruption by setting the thread's interrupt status; blocking interruptible methods may react by throwing `InterruptedException`.

---

## 1. What Is Thread Interruption? ⭐⭐⭐⭐⭐

Thread interruption is Java's cooperative mechanism for requesting a thread to stop what it is doing or change its current operation.

```java
thread.interrupt();
```

It does **not** mean:

```text
"Kill this thread immediately"
```

Instead:

```text
Thread A
   |
   | interrupt()
   ↓
Thread B's interrupt status is set
   |
   ↓
Thread B can observe/respond
```

The target thread decides how to respond.

---

# 2. What Does `interrupt()` Actually Do? ⭐⭐⭐⭐⭐

For a thread that is not currently blocked in an interruptible operation, calling:

```java
thread.interrupt();
```

sets its interrupt status.

A running thread can observe it with:

```java
Thread.currentThread().isInterrupted();
```

Important distinction:

```java
isInterrupted()
```

checks the status without clearing it.

```java
Thread.interrupted()
```

checks the **current thread's** status and clears it.

---

# 3. `interrupt()` Does Not Forcefully Stop a Thread ⭐⭐⭐⭐⭐

This is one of the most important interview points.

```java
thread.interrupt();
```

does not guarantee:

- immediate termination
- stack unwinding
- resource cleanup
- lock release
- JVM-level thread destruction

It is a **cooperative cancellation signal**.

### Interview line

> **`interrupt()` requests interruption; it does not forcibly terminate the target thread.**

---

# 4. Basic Practice Code ⭐⭐⭐⭐⭐

```java
public class BasicInterruptDemo {

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            System.out.println("Worker started");

            while (!Thread.currentThread().isInterrupted()) {
                // Simulate useful work
            }

            System.out.println("Worker detected interruption");
        });

        worker.start();

        Thread.sleep(100);
        worker.interrupt();

        worker.join();
        System.out.println("Main finished");
    }
}
```

Here the worker cooperatively checks its interrupt status and exits.

---

# 5. `isInterrupted()` ⭐⭐⭐⭐⭐

```java
thread.isInterrupted();
```

checks the specified thread's interrupt status.

It **does not clear** the status.

Example:

```java
Thread worker = Thread.currentThread();

worker.interrupt();

System.out.println(worker.isInterrupted()); // true
System.out.println(worker.isInterrupted()); // true
```

The status remains set.

---

# 6. `Thread.interrupted()` ⭐⭐⭐⭐⭐

`Thread.interrupted()` is static and checks the **current thread**.

It also clears the interrupt status.

```java
Thread.currentThread().interrupt();

System.out.println(Thread.interrupted()); // true
System.out.println(Thread.interrupted()); // false
```

Think:

```text
isInterrupted()   → check only
interrupted()     → check + clear
```

---

# 7. Practice Code — Check vs Clear ⭐⭐⭐⭐⭐

```java
public class InterruptStatusDemo {

    public static void main(String[] args) {
        Thread current = Thread.currentThread();

        current.interrupt();

        System.out.println("isInterrupted #1 = "
                + current.isInterrupted());

        System.out.println("isInterrupted #2 = "
                + current.isInterrupted());

        System.out.println("interrupted #1 = "
                + Thread.interrupted());

        System.out.println("interrupted #2 = "
                + Thread.interrupted());
    }
}
```

Expected:

```text
isInterrupted #1 = true
isInterrupted #2 = true
interrupted #1 = true
interrupted #2 = false
```

---

# 8. `sleep()` and Interruption ⭐⭐⭐⭐⭐

`sleep()` is interruptible.

```java
try {
    Thread.sleep(10_000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

If another thread interrupts the sleeping thread:

```java
worker.interrupt();
```

`sleep()` throws:

```text
InterruptedException
```

The important point is that the exception is the blocking operation's response to interruption.

---

# 9. Practice Code — Interrupt Sleeping Thread ⭐⭐⭐⭐⭐

```java
public class InterruptSleepDemo {

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            try {
                System.out.println("Worker going to sleep...");
                Thread.sleep(10_000);
                System.out.println("Worker completed sleep");
            } catch (InterruptedException e) {
                System.out.println("Worker interrupted during sleep");
                Thread.currentThread().interrupt();
            }
        });

        worker.start();

        Thread.sleep(500);
        worker.interrupt();

        worker.join();
    }
}
```

The worker does not have to wait ten seconds once interruption causes `sleep()` to throw.

---

# 10. What Happens to Interrupt Status When `InterruptedException` Is Thrown? ⭐⭐⭐⭐⭐

For interruptible blocking methods such as `sleep()`, `wait()`, and `join()`, interruption causes `InterruptedException` and the interrupt status is cleared as part of that response.

Therefore this is often appropriate when the current method cannot handle the interruption itself:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

This restores the status so higher-level code can observe the cancellation request.

---

# 11. `wait()` and Interruption ⭐⭐⭐⭐⭐

A thread waiting through:

```java
synchronized (lock) {
    lock.wait();
}
```

can be interrupted.

The call throws:

```java
InterruptedException
```

and the waiting operation ends.

Example:

```java
try {
    synchronized (lock) {
        lock.wait();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

---

# 12. Practice Code — Interrupt `wait()` ⭐⭐⭐⭐⭐

```java
public class InterruptWaitDemo {

    public static void main(String[] args) throws Exception {
        Object lock = new Object();

        Thread worker = new Thread(() -> {
            try {
                synchronized (lock) {
                    System.out.println("Worker waiting...");
                    lock.wait();
                    System.out.println("Worker resumed normally");
                }
            } catch (InterruptedException e) {
                System.out.println("Worker interrupted while waiting");
                Thread.currentThread().interrupt();
            }
        });

        worker.start();

        Thread.sleep(500);
        worker.interrupt();

        worker.join();
    }
}
```

No `notify()` is needed because interruption itself causes the waiting operation to terminate with `InterruptedException`.

---

# 13. `join()` and Interruption ⭐⭐⭐⭐⭐

A thread waiting in:

```java
worker.join();
```

can itself be interrupted.

Example:

```java
try {
    worker.join();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

Important:

> The interruption applies to the thread that is **waiting in `join()`**, not automatically to the worker being joined.

---

# 14. Practice Code — Interrupt a Thread Waiting in `join()` ⭐⭐⭐⭐⭐

```java
public class InterruptJoinDemo {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread waiter = new Thread(() -> {
            try {
                System.out.println("Waiter calling join...");
                worker.join();
                System.out.println("Waiter finished join");
            } catch (InterruptedException e) {
                System.out.println("Waiter interrupted during join");
                Thread.currentThread().interrupt();
            }
        });

        worker.start();
        waiter.start();

        Thread.sleep(500);
        waiter.interrupt();

        waiter.join();
        worker.interrupt();
        worker.join();
    }
}
```

---

# 15. Interrupting a Thread That Ignores the Flag ⭐⭐⭐⭐⭐

A thread can technically ignore interruption if its code never checks the status and is not blocked in an interruptible operation.

```java
public class IgnoreInterruptDemo {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            long end = System.currentTimeMillis() + 2000;

            while (System.currentTimeMillis() < end) {
                // Does not check interruption
            }

            System.out.println("Worker completed");
        });

        worker.start();
        Thread.sleep(100);
        worker.interrupt();

        worker.join();
    }
}
```

The interrupt request does not magically stop the loop.

---

# 16. Cooperative Cancellation ⭐⭐⭐⭐⭐

A good cancellable worker looks like:

```java
while (!Thread.currentThread().isInterrupted()) {
    doWork();
}
```

Or:

```java
while (!Thread.currentThread().isInterrupted()) {
    try {
        doBlockingWork();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        break;
    }
}
```

The exact handling depends on the application's cancellation policy.

---

# 17. Practice Code — Graceful Cancellation ⭐⭐⭐⭐⭐

```java
public class GracefulCancellationDemo {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    System.out.println("Processing...");
                    Thread.sleep(500);
                }
            } catch (InterruptedException e) {
                System.out.println("Cancellation requested");
                Thread.currentThread().interrupt();
            }

            System.out.println("Worker exiting gracefully");
        });

        worker.start();

        Thread.sleep(1500);
        worker.interrupt();

        worker.join();
    }
}
```

---

# 18. Why Restore the Interrupt Status? ⭐⭐⭐⭐⭐

Consider:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

Why?

Because the interruptible method has consumed/cleared the interrupt status when throwing `InterruptedException`.

Restoring it communicates:

```text
"This thread was asked to stop/interrupted."
```

to higher-level code.

### Interview line

> **If I cannot fully handle an `InterruptedException`, I usually restore the interrupt status with `Thread.currentThread().interrupt()` and let higher-level code decide what to do.**

---

# 19. Bad Practice — Swallowing Interruption ⭐⭐⭐⭐⭐

Avoid:

```java
catch (InterruptedException e) {
    // ignore
}
```

This can hide cancellation requests.

Better:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

Or handle cancellation explicitly according to the application's policy.

---

# 20. Interrupt Does Not Release Locks ⭐⭐⭐⭐⭐

`interrupt()` does not generally release intrinsic monitors.

Example:

```java
synchronized (lock) {
    while (true) {
        // interrupt does not automatically release lock
    }
}
```

Calling:

```java
thread.interrupt();
```

does not force the thread out of the synchronized block.

This is different from `wait()`, which releases the waited-on monitor as part of waiting.

---

# 21. Interrupt vs `stop()` ⭐⭐⭐⭐⭐

Do not confuse cooperative interruption with forceful thread termination APIs.

```java
thread.interrupt();
```

is the standard cooperative cancellation signal.

`Thread.stop()` is deprecated and unsafe because asynchronous termination can leave shared state inconsistent.

### Interview line

> **Use cooperative interruption rather than unsafe forceful thread termination.**

---

# 22. Interrupt Status and `RUNNABLE` ⭐⭐⭐⭐

An interrupted thread does not automatically enter a special Java `INTERRUPTED` state.

Java `Thread.State` has no `INTERRUPTED` state.

A thread can be:

```text
RUNNABLE + interrupt status = true
```

The interrupt status is separate from `Thread.State`.

---

# 23. Practice Code — Interrupt Flag Without Blocking ⭐⭐⭐⭐⭐

```java
public class InterruptFlagDemo {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                // Busy work for demonstration only
            }

            System.out.println("Interrupt detected");
        });

        worker.start();

        Thread.sleep(200);
        worker.interrupt();

        worker.join();
    }
}
```

Here the worker notices the flag explicitly because no interruptible blocking call is involved.

---

# 24. Important APIs ⭐⭐⭐⭐⭐

| API | Meaning |
|---|---|
| `thread.interrupt()` | Request interruption of another thread |
| `thread.isInterrupted()` | Check target thread's flag without clearing |
| `Thread.interrupted()` | Check current thread's flag and clear it |
| `sleep()` | Interruptible timed blocking |
| `wait()` | Interruptible monitor waiting |
| `join()` | Interruptible waiting for another thread |

---

# 25. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1

> `interrupt()` kills the thread.

❌ False. It is a cooperative interruption request.

### Trap 2

> `interrupt()` releases the thread's lock.

❌ False.

### Trap 3

> `isInterrupted()` clears the flag.

❌ False.

### Trap 4

> `Thread.interrupted()` checks any arbitrary thread.

❌ False. It checks the current thread.

### Trap 5

> A thread must be sleeping for `interrupt()` to work.

❌ False. A thread can observe its interrupt status while running.

### Trap 6

> `InterruptedException` means the thread was killed.

❌ False. It means an interruptible blocking operation responded to an interruption request.

### Trap 7

> `notify()` is required to stop a waiting thread after interruption.

❌ False. Interruption itself can cause `wait()` to throw `InterruptedException`.

### Trap 8

> Catching `InterruptedException` and doing nothing is always safe.

❌ False. It can swallow an important cancellation signal.

---

# 26. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is thread interruption?

A cooperative mechanism for requesting a thread to stop or change its current operation.

### Q2. Does `interrupt()` stop a thread?

No. It sets the interrupt status or causes an interruptible blocking operation to throw `InterruptedException`.

### Q3. What is the difference between `isInterrupted()` and `Thread.interrupted()`?

`isInterrupted()` checks without clearing. `Thread.interrupted()` checks the current thread and clears the status.

### Q4. What happens when a sleeping thread is interrupted?

`sleep()` throws `InterruptedException`.

### Q5. What happens when a waiting thread is interrupted?

`wait()` throws `InterruptedException`.

### Q6. What happens to interrupt status when `InterruptedException` is thrown?

For the standard interruptible blocking methods, the status is cleared as part of throwing the exception.

### Q7. Why restore interrupt status?

To preserve the cancellation signal for higher-level code when the current method cannot fully handle it.

### Q8. Does interrupt release a synchronized lock?

No.

### Q9. Is there an `INTERRUPTED` `Thread.State`?

No. Interrupt status is separate from Java's `Thread.State` enum.

### Q10. Can a running thread ignore an interrupt?

Yes, if it does not check/respond to the interrupt and is not blocked in an interruptible operation.

---

# 27. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Thread interruption in Java is a cooperative mechanism for cancellation or changing a thread's current operation. Calling `interrupt()` does not forcibly stop the thread; it requests interruption. If the target thread is running normally, its interrupt status is set and the code can check it using `isInterrupted()`. If it is blocked in an interruptible operation such as `sleep()`, `wait()`, or `join()`, that operation throws `InterruptedException`. `isInterrupted()` checks a thread's status without clearing it, while `Thread.interrupted()` checks the current thread and clears the status. When handling `InterruptedException`, if the current layer cannot fully handle cancellation, it is usually good practice to restore the status using `Thread.currentThread().interrupt()`. Interrupting a thread does not automatically release intrinsic locks. So interruption is a cooperative cancellation signal, not a kill mechanism."**

---

# 28. Quick Revision ⭐⭐⭐⭐⭐

```text
interrupt()
   ↓
cooperative cancellation request
   ↓
NOT forceful termination

Running thread
   → interrupt flag becomes true
   → code can check isInterrupted()

sleep()/wait()/join()
   → interruption causes InterruptedException
   → blocking operation ends
   → restore flag if appropriate

isInterrupted()
   → check, don't clear

Thread.interrupted()
   → current thread
   → check + clear

interrupt()
   → does NOT release intrinsic locks
```

### One-line memory trick

> **Interrupt = request, not kill.**

---

# 29. Practice Checklist

- [x] Meaning of interruption
- [x] `interrupt()` behavior
- [x] Cooperative cancellation
- [x] `isInterrupted()`
- [x] `Thread.interrupted()`
- [x] Interrupting `sleep()`
- [x] Interrupting `wait()`
- [x] Interrupting `join()`
- [x] `InterruptedException`
- [x] Restoring interrupt status
- [x] Interrupt does not release intrinsic locks
- [x] Interrupt status vs `Thread.State`
- [x] Graceful cancellation
- [x] Common traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.27 — `wait()` vs `sleep()`](../27-wait-vs-sleep/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.29 — `interrupt()`**