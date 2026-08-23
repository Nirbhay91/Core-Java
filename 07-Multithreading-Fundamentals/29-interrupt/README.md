# 7.29 — `interrupt()`

## 🎯 Objective

Understand the `Thread.interrupt()` API deeply: what it does, what it does **not** do, how it behaves for running and blocked threads, interrupt status, `InterruptedException`, and how to use it for graceful cancellation.

> **Golden rule:** `interrupt()` is a **cooperative interruption request**, not a forceful thread-kill operation.

---

## 1. What Is `interrupt()`? ⭐⭐⭐⭐⭐

`interrupt()` is an instance method of `Thread` used by one thread to request interruption of another thread.

```java
worker.interrupt();
```

Conceptually:

```text
Caller Thread
     |
     | worker.interrupt()
     ↓
Target Thread
     |
     ↓
interrupt request / status
     |
     ↓
Target code decides how to respond
```

The target thread is **not forcibly terminated**.

---

# 2. Method Signature ⭐⭐⭐⭐⭐

```java
public void interrupt()
```

Example:

```java
Thread worker = new Thread(() -> {
    // work
});

worker.start();
worker.interrupt();
```

`interrupt()` itself does not throw `InterruptedException`.

The target thread may encounter `InterruptedException` if it is blocked in an interruptible operation.

---

# 3. What `interrupt()` Does ⭐⭐⭐⭐⭐

For a thread that is not currently blocked in an interruptible operation, calling:

```java
worker.interrupt();
```

sets the target thread's interrupt status.

The target can inspect it:

```java
worker.isInterrupted();
```

or, from inside the target thread:

```java
Thread.currentThread().isInterrupted();
```

---

# 4. What `interrupt()` Does NOT Do ⭐⭐⭐⭐⭐

Calling:

```java
worker.interrupt();
```

does **not** automatically:

- kill the thread
- stop the thread immediately
- release intrinsic locks
- release arbitrary resources
- roll back application state
- guarantee that the target exits

### Interview line

> **`interrupt()` sends a cancellation/interruption request; the target thread must cooperate.**

---

# 5. Basic Practice Code ⭐⭐⭐⭐⭐

```java
public class InterruptBasicDemo {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                // Simulate work
            }

            System.out.println("Worker received interrupt request");
        }, "Worker");

        worker.start();

        Thread.sleep(200);
        worker.interrupt();

        worker.join();
        System.out.println("Main finished");
    }
}
```

The worker checks its own interrupt status and exits voluntarily.

---

# 6. `interrupt()` vs `isInterrupted()` ⭐⭐⭐⭐⭐

```java
worker.interrupt();
```

requests interruption.

```java
worker.isInterrupted();
```

checks the target's interrupt status without clearing it.

Example:

```java
worker.interrupt();

System.out.println(worker.isInterrupted()); // true
System.out.println(worker.isInterrupted()); // true
```

---

# 7. `interrupt()` vs `Thread.interrupted()` ⭐⭐⭐⭐⭐

```java
worker.interrupt();
```

acts on the target thread.

```java
Thread.interrupted();
```

checks the **current thread** and clears its interrupt status.

Memory trick:

```text
interrupt()        → request another/current target thread
isInterrupted()    → check target, don't clear
interrupted()      → check current thread + clear
```

---

# 8. Practice Code — Status Behavior ⭐⭐⭐⭐⭐

```java
public class InterruptStatusPractice {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            System.out.println("Initial: "
                    + Thread.currentThread().isInterrupted());

            while (!Thread.currentThread().isInterrupted()) {
                // work
            }

            System.out.println("After interrupt: "
                    + Thread.currentThread().isInterrupted());
        });

        worker.start();

        Thread.sleep(200);
        worker.interrupt();
        worker.join();
    }
}
```

Typical output:

```text
Initial: false
After interrupt: true
```

---

# 9. Interrupting `sleep()` ⭐⭐⭐⭐⭐

If the target thread is sleeping:

```java
Thread.sleep(10_000);
```

and another thread calls:

```java
worker.interrupt();
```

`sleep()` responds by throwing:

```java
InterruptedException
```

Example:

```java
try {
    Thread.sleep(10_000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

---

# 10. Practice Code — Interrupt Sleeping Worker ⭐⭐⭐⭐⭐

```java
public class InterruptSleepPractice {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            try {
                System.out.println("Worker sleeping...");
                Thread.sleep(10_000);
                System.out.println("Sleep completed");
            } catch (InterruptedException e) {
                System.out.println("Sleep interrupted");
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

---

# 11. Interrupting `wait()` ⭐⭐⭐⭐⭐

A thread blocked in:

```java
synchronized (lock) {
    lock.wait();
}
```

can be interrupted.

```java
worker.interrupt();
```

causes the waiting operation to terminate with `InterruptedException`.

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
public class InterruptWaitPractice {

    public static void main(String[] args) throws Exception {
        Object lock = new Object();

        Thread worker = new Thread(() -> {
            try {
                synchronized (lock) {
                    System.out.println("Worker waiting...");
                    lock.wait();
                }
            } catch (InterruptedException e) {
                System.out.println("Worker interrupted");
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

No `notify()` is needed because interruption itself ends the waiting operation.

---

# 13. Interrupting `join()` ⭐⭐⭐⭐⭐

Suppose Thread-A calls:

```java
worker.join();
```

Thread-A is waiting for `worker` to finish.

If another thread interrupts Thread-A:

```java
threadA.interrupt();
```

then Thread-A's `join()` throws `InterruptedException`.

Important:

> Interrupting the thread waiting in `join()` does **not** automatically interrupt the thread being joined.

---

# 14. Practice Code — Interrupt `join()` Waiter ⭐⭐⭐⭐⭐

```java
public class InterruptJoinPractice {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Worker");

        Thread waiter = new Thread(() -> {
            try {
                System.out.println("Waiter calling join...");
                worker.join();
                System.out.println("Join completed");
            } catch (InterruptedException e) {
                System.out.println("Waiter interrupted");
                Thread.currentThread().interrupt();
            }
        }, "Waiter");

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

# 15. Running Thread Can Observe Interrupt ⭐⭐⭐⭐⭐

Interruption is not limited to blocking methods.

A normal running thread can check:

```java
while (!Thread.currentThread().isInterrupted()) {
    doWork();
}
```

This is the basis of cooperative cancellation.

---

# 16. Practice Code — CPU Work + Interrupt ⭐⭐⭐⭐⭐

```java
public class InterruptCpuWorkPractice {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            long count = 0;

            while (!Thread.currentThread().isInterrupted()) {
                count++;
            }

            System.out.println("Stopped. Count = " + count);
        });

        worker.start();

        Thread.sleep(500);
        worker.interrupt();

        worker.join();
    }
}
```

The interrupt request becomes meaningful because the worker explicitly checks the status.

---

# 17. Interrupted Status Is Separate from `Thread.State` ⭐⭐⭐⭐⭐

There is no:

```text
INTERRUPTED
```

value in `Thread.State`.

A thread can be:

```text
RUNNABLE + interrupt status = true
```

The interrupt status and thread state are separate concepts.

---

# 18. `InterruptedException` and the Interrupt Flag ⭐⭐⭐⭐⭐

For interruptible blocking methods such as:

```java
sleep()
wait()
join()
```

an interruption causes `InterruptedException` and the interrupt status is cleared as part of that response.

Therefore:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

is commonly used to restore the signal when the current method is not going to fully handle cancellation.

---

# 19. Practice Code — Restore the Interrupt Status ⭐⭐⭐⭐⭐

```java
public class RestoreInterruptPractice {

    static void doWork() {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            // Blocking method responded to interruption.
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            doWork();

            System.out.println("Status after restoration = "
                    + Thread.currentThread().isInterrupted());
        });

        worker.start();
        Thread.sleep(300);
        worker.interrupt();
        worker.join();
    }
}
```

Expected status after restoration:

```text
true
```

---

# 20. `interrupt()` Does Not Release Intrinsic Locks ⭐⭐⭐⭐⭐

Consider:

```java
synchronized (lock) {
    while (!Thread.currentThread().isInterrupted()) {
        // work
    }
}
```

Calling:

```java
worker.interrupt();
```

does not itself release `lock`.

The worker must respond to interruption and eventually leave the synchronized block.

### Important comparison

```text
wait()      → releases waited-on monitor while waiting
interrupt() → does NOT release monitor by itself
```

---

# 21. Practice Code — Interrupt Does Not Unlock ⭐⭐⭐⭐⭐

```java
public class InterruptLockPractice {

    private static final Object LOCK = new Object();

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            synchronized (LOCK) {
                while (!Thread.currentThread().isInterrupted()) {
                    // Hold lock while doing demonstration work.
                }

                System.out.println("Worker detected interrupt");
            }

            System.out.println("Worker released lock by leaving block");
        });

        worker.start();

        Thread.sleep(300);
        worker.interrupt();

        worker.join();
    }
}
```

The lock is released because the thread exits the synchronized block—not because `interrupt()` released it.

---

# 22. `interrupt()` Is Not `stop()` ⭐⭐⭐⭐⭐

Do not confuse:

```java
worker.interrupt();
```

with forceful thread termination.

`Thread.stop()` is deprecated and unsafe because asynchronous termination can leave shared state inconsistent.

Preferred model:

```text
interrupt()
    ↓
request cancellation
    ↓
target observes request
    ↓
cleanup
    ↓
exit gracefully
```

---

# 23. Common Pattern — Cancellation Flag ⭐⭐⭐⭐⭐

```java
while (!Thread.currentThread().isInterrupted()) {
    processNextItem();
}
```

If `processNextItem()` blocks interruptibly:

```java
try {
    while (!Thread.currentThread().isInterrupted()) {
        processNextItem();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

This combines flag-based and exception-based interruption handling.

---

# 24. Practice Code — Graceful Worker ⭐⭐⭐⭐⭐

```java
public class GracefulWorkerPractice {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    System.out.println("Processing...");
                    Thread.sleep(400);
                }
            } catch (InterruptedException e) {
                System.out.println("Cancellation requested");
                Thread.currentThread().interrupt();
            } finally {
                System.out.println("Cleanup completed");
            }
        });

        worker.start();

        Thread.sleep(1500);
        worker.interrupt();

        worker.join();
    }
}
```

---

# 25. `interrupt()` After Thread Completion ⭐⭐⭐⭐

If a thread has already terminated, interrupting it does not restart it.

```java
Thread worker = new Thread(() -> {
    System.out.println("Done");
});

worker.start();
worker.join();

worker.interrupt();
```

The thread remains terminated.

---

# 26. Interrupt Before `start()` — Interview Trap ⭐⭐⭐⭐⭐

You can set the interrupt status before a thread starts:

```java
Thread worker = new Thread(() -> {
    System.out.println(Thread.currentThread().isInterrupted());
});

worker.interrupt();
worker.start();
```

The new thread can observe the status as part of its execution.

Do not confuse this with forcibly cancelling thread startup—the thread still starts when `start()` is called.

---

# 27. Practice Code — Interrupt Before Start ⭐⭐⭐⭐

```java
public class InterruptBeforeStartPractice {

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            System.out.println("Interrupted = "
                    + Thread.currentThread().isInterrupted());
        });

        worker.interrupt();
        worker.start();
        worker.join();
    }
}
```

This demonstrates that interrupt status is independent of the thread's execution work itself.

---

# 28. `interrupt()` and `Thread.interrupted()` Trap ⭐⭐⭐⭐⭐

Suppose the current thread's status is set:

```java
Thread.currentThread().interrupt();
```

Then:

```java
Thread.interrupted();
```

returns `true` and clears the status.

Calling it again:

```java
Thread.interrupted();
```

returns `false` unless another interruption occurred.

### Memory

```text
isInterrupted()       → peek
Thread.interrupted()  → consume/reset
```

---

# 29. Best Practices ⭐⭐⭐⭐⭐

1. Treat interruption as a cancellation/cooperative-control signal.
2. Do not assume `interrupt()` kills the thread.
3. Check interruption in long-running loops.
4. Handle `InterruptedException` deliberately.
5. Restore interrupt status when propagating cancellation responsibility upward.
6. Do not swallow interruption silently.
7. Do not use interruption as a substitute for proper locking.
8. Do not assume interruption releases intrinsic locks.
9. Keep cleanup in `finally` where appropriate.
10. Prefer structured cancellation mechanisms such as `ExecutorService`/`Future` where applicable.

---

# 30. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1

> `interrupt()` terminates a thread.

❌ False.

### Trap 2

> `interrupt()` releases `synchronized` locks.

❌ False.

### Trap 3

> `interrupt()` always throws `InterruptedException`.

❌ False. The target thread's interruptible blocking operation may throw it.

### Trap 4

> `isInterrupted()` clears the flag.

❌ False.

### Trap 5

> `Thread.interrupted()` checks another thread's flag.

❌ False. It checks the current thread.

### Trap 6

> `notify()` is required when interrupting `wait()`.

❌ False.

### Trap 7

> An interrupted thread must have a special `INTERRUPTED` state.

❌ False. No such `Thread.State` exists.

### Trap 8

> Interrupting the thread being joined interrupts the joining thread.

❌ The thread that calls `join()` is the one that is waiting and can be interrupted.

---

# 31. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What does `Thread.interrupt()` do?

It requests interruption of the target thread, generally by setting its interrupt status or causing an interruptible blocking operation to throw `InterruptedException`.

### Q2. Does `interrupt()` kill the thread?

No. It is cooperative.

### Q3. What happens if the thread is sleeping?

`sleep()` throws `InterruptedException`.

### Q4. What happens if the thread is in `wait()`?

`wait()` throws `InterruptedException`.

### Q5. What happens if the thread is in `join()`?

`join()` throws `InterruptedException` in the thread that is waiting in `join()`.

### Q6. Does `interrupt()` release a lock?

No.

### Q7. Why call `Thread.currentThread().interrupt()` in a catch block?

To restore the interrupt signal when the current layer cannot fully handle the interruption.

### Q8. What is the difference between `interrupt()` and `isInterrupted()`?

`interrupt()` requests interruption; `isInterrupted()` checks the status without clearing it.

### Q9. What is the difference between `isInterrupted()` and `Thread.interrupted()`?

The former checks without clearing; the latter checks the current thread and clears the status.

### Q10. Can a thread ignore an interrupt?

Yes, if its code does not respond to the status and it is not in an interruptible blocking operation.

### Q11. Does interruption automatically clean up resources?

No. Application code must perform appropriate cleanup.

### Q12. Is interruption the same as cancellation?

Interruption is a mechanism commonly used to implement cooperative cancellation, but cancellation policy is an application-level concept.

---

# 32. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"`Thread.interrupt()` is Java's cooperative interruption mechanism. When I call `worker.interrupt()`, I am requesting the worker to stop or change its current operation; I am not forcibly killing it. If the worker is running normally, it can observe the interrupt status with `isInterrupted()`. If it is blocked in an interruptible operation such as `sleep()`, `wait()`, or `join()`, that operation can throw `InterruptedException`. After catching the exception, if my layer cannot fully handle cancellation, I usually restore the interrupt status with `Thread.currentThread().interrupt()`. `interrupt()` also does not release intrinsic locks. So the correct mental model is: interrupt is a cooperative request, the target decides how to respond, and cleanup must be handled by the application."**

---

# 33. Quick Revision ⭐⭐⭐⭐⭐

```text
interrupt()
   ↓
cooperative request
   ↓
NOT kill

Running target
   → interrupt status set
   → target checks isInterrupted()

sleep()/wait()/join()
   → interruption causes InterruptedException
   → blocking operation ends
   → restore flag if appropriate

isInterrupted()
   → check target
   → don't clear

Thread.interrupted()
   → current thread
   → check + clear

interrupt()
   → does NOT release intrinsic locks
```

### One-line memory trick

> **`interrupt()` = "Please stop/change what you're doing" — not "Die immediately."**

---

# 34. Practice Checklist

- [x] `interrupt()` definition
- [x] Interrupt status
- [x] Cooperative interruption
- [x] Running-thread interruption
- [x] Interrupting `sleep()`
- [x] Interrupting `wait()`
- [x] Interrupting `join()`
- [x] `InterruptedException`
- [x] Restoring interrupt status
- [x] Interrupt does not release intrinsic locks
- [x] `isInterrupted()` vs `Thread.interrupted()`
- [x] Interrupt before start
- [x] Interrupt after termination
- [x] Graceful cancellation
- [x] Best practices
- [x] Common traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.28 — Thread Interruption](../28-Thread-Interruption/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.30 — Deadlock**