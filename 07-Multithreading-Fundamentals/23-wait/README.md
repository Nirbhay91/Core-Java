# 7.23 — `wait()`

## 🎯 Objective

Understand Java's `Object.wait()` mechanism, monitor ownership, why `wait()` must be called while holding the same object's monitor, what happens to the lock, how interruption works, and how `wait()` differs from `sleep()`.

> **Interview rule:** `wait()` is an inter-thread coordination mechanism. It must be called while owning the object's monitor, and when it waits, the thread **releases that object's monitor** and later competes to reacquire it before continuing.

---

# 1. What is `wait()`? ⭐⭐⭐⭐⭐

`wait()` is a method of `java.lang.Object` used for thread coordination.

```java
object.wait();
```

It causes the current thread to wait until it is notified, interrupted, or otherwise awakened according to the Java specification.

Unlike `sleep()`, `wait()` is tied to an object's monitor and synchronization protocol.

---

# 2. Why Does `wait()` Exist? ⭐⭐⭐⭐⭐

`wait()` is useful when one thread cannot continue until another thread changes some shared condition.

Typical pattern:

```text
Thread A
   ↓
condition is false
   ↓
wait()
   ↓
releases monitor

Thread B
   ↓
changes condition
   ↓
notify()/notifyAll()

Thread A
   ↓
wakes
   ↓
reacquires monitor
   ↓
checks condition again
```

The important idea is **condition waiting**, not simply delaying execution.

---

# 3. `wait()` Belongs to `Object` ⭐⭐⭐⭐⭐

A common interview question:

> Why is `wait()` defined in `Object` instead of `Thread`?

Because the coordination mechanism is associated with an object's **monitor**.

Every Java object can participate in monitor-based synchronization.

```java
Object lock = new Object();
```

The same object can be used for:

```java
synchronized (lock) {
    lock.wait();
}
```

and:

```java
synchronized (lock) {
    lock.notify();
}
```

---

# 4. Basic Syntax ⭐⭐⭐⭐⭐

```java
synchronized (lock) {
    lock.wait();
}
```

The thread must own `lock`'s monitor before calling `wait()`.

Calling it outside the synchronized region causes:

```text
IllegalMonitorStateException
```

---

# 5. Practice Code — Basic `wait()` ⭐⭐⭐⭐⭐

```java
public class BasicWaitDemo {

    private final Object lock = new Object();

    public void waitingTask() throws InterruptedException {
        synchronized (lock) {
            System.out.println("Worker waiting...");
            lock.wait();
            System.out.println("Worker resumed");
        }
    }

    public void notifyTask() {
        synchronized (lock) {
            System.out.println("Notifier notifying...");
            lock.notify();
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        BasicWaitDemo demo = new BasicWaitDemo();

        Thread worker = new Thread(() -> {
            try {
                demo.waitingTask();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        worker.start();
        Thread.sleep(100);
        demo.notifyTask();

        worker.join();
    }
}
```

---

# 6. What Happens During `wait()`? ⭐⭐⭐⭐⭐

Suppose:

```java
synchronized (lock) {
    lock.wait();
}
```

Conceptually:

```text
1. Thread owns lock's monitor
2. Thread calls lock.wait()
3. Thread enters waiting state
4. Thread releases lock's monitor
5. Another thread can acquire the monitor
6. Another thread may call notify()/notifyAll()
7. Waiting thread becomes eligible to continue
8. It must reacquire the same monitor
9. Only after reacquiring can it continue
```

### Critical point

> `wait()` **releases the monitor** it is waiting on.

---

# 7. `wait()` Releases the Lock ⭐⭐⭐⭐⭐

Example:

```java
synchronized (lock) {
    lock.wait();
}
```

After entering `wait()`, the thread does **not** keep holding `lock`'s monitor while waiting.

This allows another thread to acquire the same lock and change the condition.

This is one of the most important differences from ordinary blocking inside a synchronized section.

---

# 8. `wait()` vs Holding a Monitor ⭐⭐⭐⭐⭐

```text
Before wait()
Thread A owns lock
       ↓
wait()
       ↓
Thread A releases lock
       ↓
Thread B can acquire lock
       ↓
Thread B changes condition
       ↓
Thread B notifyAll()
       ↓
Thread A becomes eligible
       ↓
Thread A reacquires lock
       ↓
Thread A continues
```

The notification does **not** transfer the lock directly to the waiting thread.

---

# 9. Practice Code — Observe Lock Release ⭐⭐⭐⭐⭐

```java
public class WaitReleasesLockDemo {

    private final Object lock = new Object();

    public void waiter() throws InterruptedException {
        synchronized (lock) {
            System.out.println("Waiter acquired lock");
            lock.wait();
            System.out.println("Waiter reacquired lock");
        }
    }

    public void anotherThread() {
        synchronized (lock) {
            System.out.println("Other thread acquired lock");
        }
    }

    public void signal() {
        synchronized (lock) {
            lock.notifyAll();
        }
    }
}
```

Run this with multiple threads and observe that another thread can enter the synchronized region while the waiter is waiting.

---

# 10. `wait()` Requires Monitor Ownership ⭐⭐⭐⭐⭐

### Wrong

```java
Object lock = new Object();
lock.wait();
```

This results in:

```text
IllegalMonitorStateException
```

### Correct

```java
synchronized (lock) {
    lock.wait();
}
```

### Interview sentence

> **A thread must own the object's monitor before invoking that object's `wait()` method.**

---

# 11. Practice Code — `IllegalMonitorStateException` ⭐⭐⭐⭐⭐

```java
public class IllegalWaitDemo {

    public static void main(String[] args)
            throws InterruptedException {

        Object lock = new Object();

        // Wrong:
        // lock.wait();

        // Correct:
        synchronized (lock) {
            // lock.wait();
        }
    }
}
```

The commented incorrect call demonstrates the rule without intentionally terminating the program.

---

# 12. `wait()` Is Usually Used with a Condition ⭐⭐⭐⭐⭐

The correct conceptual pattern is:

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }

    // condition is true
}
```

The `while` loop is essential.

Do not write condition waiting as a one-time `if` check in robust code.

---

# 13. Why `while`, Not `if`? ⭐⭐⭐⭐⭐

A waiting thread must re-check the condition after it wakes because:

- A wake-up does not itself prove the condition is true.
- Another thread may have consumed/changed the condition first.
- The specification permits spurious wakeups.
- Multiple waiting threads may compete for the condition.

Therefore:

```java
while (!condition) {
    lock.wait();
}
```

is the standard pattern.

---

# 14. Practice Code — Correct Condition Waiting ⭐⭐⭐⭐⭐

```java
public class ConditionWaitDemo {

    private final Object lock = new Object();
    private boolean ready;

    public void awaitReady() throws InterruptedException {
        synchronized (lock) {
            while (!ready) {
                lock.wait();
            }

            System.out.println("Ready = " + ready);
        }
    }

    public void makeReady() {
        synchronized (lock) {
            ready = true;
            lock.notifyAll();
        }
    }
}
```

The state change and notification occur while holding the same monitor used for the condition protocol.

---

# 15. Spurious Wakeup ⭐⭐⭐⭐⭐

A thread waiting with:

```java
lock.wait();
```

may return without the application condition becoming true.

Therefore:

```java
while (!condition) {
    lock.wait();
}
```

is required.

### Interview answer

> **`wait()` should always be used in a condition-checking loop because wake-up does not guarantee that the condition the thread needs is true.**

---

# 16. `wait()` and `notify()` ⭐⭐⭐⭐⭐

`wait()` and `notify()` are designed to work together through the same monitor.

```java
synchronized (lock) {
    lock.wait();
}
```

and:

```java
synchronized (lock) {
    lock.notify();
}
```

The notifier must own the same monitor to invoke `notify()`.

---

# 17. `notify()` Does Not Immediately Give the Lock ⭐⭐⭐⭐⭐

Suppose Thread B executes:

```java
synchronized (lock) {
    lock.notify();
}
```

Thread A may become eligible to continue, but Thread B still owns the monitor until it exits the synchronized region.

So:

```text
notify()
   ↓
waiting thread becomes eligible
   ↓
not immediate lock transfer
   ↓
notifier exits synchronized block
   ↓
waiting thread competes to reacquire lock
```

---

# 18. Timed `wait()` ⭐⭐⭐⭐⭐

`Object` provides:

```java
wait(long timeout)
```

and:

```java
wait(long timeout, int nanos)
```

Example:

```java
synchronized (lock) {
    lock.wait(1000);
}
```

The thread waits for the specified timeout or until another valid wake-up event occurs.

Timed waiting is still subject to the condition-loop rule.

---

# 19. Practice Code — Timed `wait()`

```java
public class TimedWaitDemo {

    private final Object lock = new Object();

    public void waitForEvent() throws InterruptedException {
        synchronized (lock) {
            long deadline =
                    System.nanoTime()
                            + 1_000_000_000L;

            long remaining;

            while ((remaining = deadline - System.nanoTime()) > 0) {
                lock.wait(
                        remaining / 1_000_000,
                        (int) (remaining % 1_000_000)
                );
            }

            System.out.println("Timed wait completed");
        }
    }
}
```

For real applications, prefer higher-level concurrency utilities when they express the coordination requirement more naturally.

---

# 20. `wait()` and `InterruptedException` ⭐⭐⭐⭐⭐

`wait()` declares:

```java
throws InterruptedException
```

A waiting thread can be interrupted.

Typical handling:

```java
try {
    lock.wait();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

Restoring the interrupt status is often appropriate when the current method cannot meaningfully handle the interruption itself.

---

# 21. Practice Code — Interruption During `wait()` ⭐⭐⭐⭐⭐

```java
public class WaitInterruptDemo {

    private final Object lock = new Object();

    public void waitForSignal() {
        try {
            synchronized (lock) {
                lock.wait();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("Wait interrupted");
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        WaitInterruptDemo demo =
                new WaitInterruptDemo();

        Thread worker = new Thread(demo::waitForSignal);
        worker.start();

        Thread.sleep(100);
        worker.interrupt();
        worker.join();
    }
}
```

---

# 22. `wait()` vs `sleep()` ⭐⭐⭐⭐⭐

| Feature | `wait()` | `sleep()` |
|---|---|---|
| Declared in | `Object` | `Thread` |
| Purpose | Thread coordination | Timing/delay |
| Must own monitor? | ✅ Yes | ❌ No |
| Releases current monitor? | ✅ Yes, the waited-on monitor | ❌ No |
| InterruptedException | ✅ | ✅ |
| Wakes by notification | ✅ | ❌ |
| Timeout | Optional | Required |
| Condition-based protocol | ✅ | ❌ |

### Golden interview line

> **`wait()` is for coordination and releases the object's monitor while waiting; `sleep()` is for timing and does not release monitors held by the sleeping thread.**

---

# 23. Practice Code — `wait()` vs `sleep()`

```java
public class WaitVsSleepDemo {

    private final Object lock = new Object();

    public void waitExample() throws InterruptedException {
        synchronized (lock) {
            lock.wait();
        }
    }

    public void sleepExample() throws InterruptedException {
        synchronized (lock) {
            Thread.sleep(1000);
            // The lock is still held during sleep.
        }
    }
}
```

This is one of the most frequently asked Java multithreading interview comparisons.

---

# 24. `wait()` Is Not a General Delay Mechanism ⭐⭐⭐⭐

Do not use:

```java
lock.wait(1000);
```

just because you want a one-second delay.

If the requirement is simply to pause the current thread:

```java
Thread.sleep(1000);
```

is the appropriate API.

`wait()` belongs to a monitor/condition protocol.

---

# 25. Which Lock Does `wait()` Release? ⭐⭐⭐⭐⭐

If you have:

```java
synchronized (lockA) {
    synchronized (lockB) {
        lockB.wait();
    }
}
```

the call to:

```java
lockB.wait();
```

releases `lockB`'s monitor.

It does **not** automatically release every other monitor the thread currently owns.

So the outer `lockA` monitor remains held while the thread is waiting on `lockB`.

### Important interview trap

> `wait()` releases the monitor associated with the object on which `wait()` was invoked, not all monitors owned by the thread.

---

# 26. Practice Code — Only the Waited-On Monitor Is Released

```java
public class MultipleMonitorWaitDemo {

    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void example() throws InterruptedException {
        synchronized (lockA) {
            synchronized (lockB) {
                lockB.wait();
            }
        }
    }
}
```

During `lockB.wait()`, `lockB` is released. `lockA` remains owned by the waiting thread.

This pattern should generally be avoided unless there is a strong reason because holding another monitor while waiting can cause contention or deadlock risks.

---

# 27. `wait()` Is Different from `LockSupport.park()` ⭐⭐⭐⭐

For modern concurrency code, Java also provides APIs such as:

```java
LockSupport.park();
```

`park()` is not the same mechanism as `Object.wait()`.

For this chapter, remember:

```text
Object.wait()
→ object monitor / condition protocol

LockSupport.park()
→ permit-based thread parking mechanism
```

Higher-level concurrency utilities often avoid direct use of low-level `wait/notify`.

---

# 28. Producer-Consumer Foundation ⭐⭐⭐⭐⭐

A classic use case is producer-consumer coordination.

Conceptually:

```text
Producer
   ↓
adds item
   ↓
notifyAll()

Consumer
   ↓
if queue empty
   ↓
wait()
   ↓
re-check queue
   ↓
consume item
```

Modern Java code should often prefer `BlockingQueue`, but understanding `wait()` is essential for interviews and for understanding older monitor-based designs.

---

# 29. Practice Code — Simple Producer/Consumer ⭐⭐⭐⭐⭐

```java
import java.util.LinkedList;
import java.util.Queue;

public class ProducerConsumerWaitDemo {

    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 2;

    public void produce(int value) throws InterruptedException {
        synchronized (queue) {
            while (queue.size() == capacity) {
                queue.wait();
            }

            queue.add(value);
            queue.notifyAll();
        }
    }

    public int consume() throws InterruptedException {
        synchronized (queue) {
            while (queue.isEmpty()) {
                queue.wait();
            }

            int value = queue.remove();
            queue.notifyAll();
            return value;
        }
    }
}
```

### Key rules shown

- Same monitor is used.
- Condition is checked in a `while` loop.
- `wait()` releases the queue monitor.
- State changes happen while holding the monitor.
- `notifyAll()` wakes waiting participants.

---

# 30. Prefer `BlockingQueue` in Production Code ⭐⭐⭐⭐⭐

The producer-consumer problem can usually be expressed more safely with:

```java
BlockingQueue<Integer> queue =
        new ArrayBlockingQueue<>(2);
```

Producer:

```java
queue.put(value);
```

Consumer:

```java
int value = queue.take();
```

### Interview point

> Understand `wait/notify` deeply, but prefer higher-level concurrency utilities such as `BlockingQueue` when they directly express the requirement.

---

# 31. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1
Calling `wait()` without synchronization.

❌ `IllegalMonitorStateException`

### Mistake 2
Using `if` instead of `while` for condition waiting.

❌ Can continue with a false condition.

### Mistake 3
Thinking `notify()` immediately transfers the lock.

❌ It does not.

### Mistake 4
Thinking `wait()` releases every lock held by the thread.

❌ It releases the monitor associated with the `wait()` object.

### Mistake 5
Using `wait()` as a replacement for `sleep()`.

❌ Different purposes.

### Mistake 6
Ignoring interruption.

❌ `wait()` is interruptible.

### Mistake 7
Using different lock objects for the condition and notification.

❌ The coordination protocol must use the same monitor.

### Mistake 8
Assuming notification means condition is true.

❌ Always re-check the condition.

---

# 32. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is `wait()`?

`Object.wait()` causes the current thread to wait for a notification, interruption, timeout, or other specified wake-up condition while participating in an object's monitor protocol.

### Q2. Why is `wait()` in Object?

Because it is associated with an object's monitor and condition coordination.

### Q3. Can we call `wait()` without synchronized?

No. The current thread must own the object's monitor; otherwise `IllegalMonitorStateException` is thrown.

### Q4. Does `wait()` release the lock?

It releases the monitor of the object on which `wait()` was invoked while the thread waits.

### Q5. Does `wait()` release all locks held by the thread?

No.

### Q6. Why use `while` around `wait()`?

Because a wake-up does not guarantee the desired condition is true, and spurious wakeups are permitted.

### Q7. Does `notify()` release the lock?

No. The notifier retains the monitor until it exits the synchronized region.

### Q8. `wait()` vs `sleep()`?

`wait()` is monitor-based coordination and releases the waited-on monitor; `sleep()` is a timing mechanism and does not release monitors.

### Q9. Can `wait()` be interrupted?

Yes. It throws `InterruptedException`.

### Q10. What happens after `notify()`?

A waiting thread becomes eligible to continue, but it must reacquire the monitor before it can proceed.

### Q11. What is a spurious wakeup?

A waiting thread can return from `wait()` without the application-level condition becoming true, so the condition must be checked in a loop.

### Q12. What is a modern alternative to manual wait/notify for producer-consumer?

`BlockingQueue` is a common higher-level alternative.

---

# 33. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"`wait()` is an Object method used for monitor-based thread coordination. A thread must own the object's monitor before calling `wait()`, so it is normally called inside a synchronized block or method. When the thread calls `wait()`, it releases that object's monitor and enters a waiting state. Another thread can acquire the same monitor, change the shared condition and call `notify()` or `notifyAll()`. The waiting thread does not immediately receive the lock; after being awakened, it must reacquire the monitor before continuing. We normally use `wait()` inside a `while` loop because a wake-up does not guarantee that the required condition is true and spurious wakeups are permitted. Unlike `sleep()`, `wait()` is a coordination mechanism and releases the waited-on monitor. In production code, higher-level utilities such as `BlockingQueue` are often preferable for producer-consumer coordination."**

---

# 34. Quick Revision ⭐⭐⭐⭐⭐

```text
wait()
  ↓
Object method
  ↓
Must own object's monitor
  ↓
Releases that monitor
  ↓
Waits
  ↓
notify / notifyAll / interrupt / timeout
  ↓
Becomes eligible
  ↓
Reacquires monitor
  ↓
Continues

Always prefer:
while (!condition) {
    lock.wait();
}

wait() ≠ sleep()

wait()
→ coordination
→ releases waited-on monitor

sleep()
→ delay
→ does NOT release monitors
```

### Golden Rule

> **`wait()` means: "I cannot continue until the condition changes, so I will release this object's monitor and wait; after waking, I will reacquire the monitor and re-check the condition."**

---

# 35. Practice Checklist

- [x] `wait()` definition
- [x] Why `wait()` belongs to `Object`
- [x] Monitor ownership requirement
- [x] `IllegalMonitorStateException`
- [x] Monitor release behavior
- [x] Reacquisition after wake-up
- [x] Condition waiting
- [x] `while` vs `if`
- [x] Spurious wakeup
- [x] `notify()` interaction
- [x] Timed `wait()`
- [x] Interrupt handling
- [x] `wait()` vs `sleep()`
- [x] Multiple monitors
- [x] Producer-consumer
- [x] `BlockingQueue` alternative
- [x] Common mistakes
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.22 — `volatile` vs `synchronized`](../22-Volatile-vs-Synchronized/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.24 — `notify()`**