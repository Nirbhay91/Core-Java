# 7.24 — `notify()`

## 🎯 Objective

Understand Java's `Object.notify()` mechanism, monitor ownership, how it interacts with `wait()`, what a notification actually does, why it does **not** immediately transfer the lock, and how to use it correctly in condition-based thread coordination.

> **Interview rule:** `notify()` wakes **one** thread waiting on this object's monitor, if any. The notifying thread must own that monitor, and the notified thread does not immediately acquire the lock; it must compete to reacquire the monitor after the notifier releases it.

---

# 1. What is `notify()`? ⭐⭐⭐⭐⭐

`notify()` is a method of `java.lang.Object` used for thread coordination.

```java
lock.notify();
```

It wakes one thread that is waiting on the same object's monitor.

If multiple threads are waiting, `notify()` selects one waiting thread according to JVM scheduling/selection rules; the API does not provide an application-level guarantee about which waiting thread is selected.

---

# 2. Why Does `notify()` Exist? ⭐⭐⭐⭐⭐

`notify()` is used when one thread changes a shared condition and wants to make a waiting thread eligible to continue.

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
notify()

Thread A
   ↓
becomes eligible
   ↓
reacquires monitor
   ↓
checks condition again
```

The notification itself does not make the condition true. The shared-state change is what makes the condition potentially true.

---

# 3. `notify()` Belongs to `Object` ⭐⭐⭐⭐⭐

`notify()` is declared by `Object` because Java's monitor-based coordination is associated with objects.

Example:

```java
Object lock = new Object();
```

Waiting:

```java
synchronized (lock) {
    lock.wait();
}
```

Notification:

```java
synchronized (lock) {
    lock.notify();
}
```

Both operations must use the same monitor for the coordination protocol.

---

# 4. Monitor Ownership Requirement ⭐⭐⭐⭐⭐

The current thread must own the object's monitor before calling `notify()`.

### Correct

```java
synchronized (lock) {
    lock.notify();
}
```

### Incorrect

```java
lock.notify();
```

The incorrect version throws:

```text
IllegalMonitorStateException
```

### Interview sentence

> **`notify()` must be called while the current thread owns the monitor of the object on which `notify()` is invoked.**

---

# 5. Practice Code — Basic `notify()` ⭐⭐⭐⭐⭐

```java
public class BasicNotifyDemo {

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
            System.out.println("Notifier calls notify()");
            lock.notify();
            System.out.println("Notifier still owns the lock");
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        BasicNotifyDemo demo = new BasicNotifyDemo();

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

# 6. What Does `notify()` Actually Do? ⭐⭐⭐⭐⭐

Suppose Thread A is waiting:

```java
synchronized (lock) {
    lock.wait();
}
```

Then Thread B executes:

```java
synchronized (lock) {
    lock.notify();
}
```

Conceptually:

```text
Thread A
  ↓
WAITING

Thread B
  ↓
owns lock
  ↓
notify()
  ↓
Thread A becomes eligible to compete
  ↓
Thread B still owns lock
  ↓
Thread B exits synchronized block
  ↓
Thread A can reacquire lock
  ↓
Thread A continues
```

### Critical point

> `notify()` does **not** directly hand the monitor to the waiting thread.

---

# 7. `notify()` Does Not Release the Lock ⭐⭐⭐⭐⭐

Consider:

```java
synchronized (lock) {
    lock.notify();
    System.out.println("Still inside synchronized block");
}
```

The notifying thread still owns the monitor after `notify()`.

The lock is released only when execution leaves the synchronized region, subject to normal monitor semantics.

This is one of the most common interview traps.

---

# 8. Practice Code — Notification Does Not Transfer the Lock ⭐⭐⭐⭐⭐

```java
public class NotifyLockDemo {

    private final Object lock = new Object();

    public void waiter() throws InterruptedException {
        synchronized (lock) {
            System.out.println("Waiter: waiting");
            lock.wait();
            System.out.println("Waiter: reacquired lock");
        }
    }

    public void notifier() {
        synchronized (lock) {
            System.out.println("Notifier: acquired lock");
            lock.notify();
            System.out.println("Notifier: still inside synchronized block");
        }

        System.out.println("Notifier: released lock");
    }
}
```

Depending on scheduling, output order after the notification may vary, but the notifier cannot be forced to release its monitor merely by calling `notify()`.

---

# 9. `notify()` Wakes One Waiting Thread ⭐⭐⭐⭐⭐

If three threads are waiting:

```text
Thread A → WAITING
Thread B → WAITING
Thread C → WAITING
```

and another thread executes:

```java
lock.notify();
```

one waiting thread becomes eligible.

Conceptually:

```text
notify()
   ↓
one waiter selected
   ↓
that waiter becomes eligible
   ↓
must reacquire monitor
```

The other waiting threads remain waiting unless another event wakes them.

---

# 10. `notify()` Does Not Guarantee Which Thread Wakes ⭐⭐⭐⭐⭐

Suppose:

```text
A → waiting
B → waiting
C → waiting
```

After:

```java
lock.notify();
```

Java does not give your application a portable guarantee such as:

```text
A will always wake first
```

Do not build business logic around a particular waiter-selection order.

---

# 11. Practice Code — Multiple Waiters ⭐⭐⭐⭐⭐

```java
public class MultipleWaitersNotifyDemo {

    private final Object lock = new Object();

    public void waitForSignal(String name)
            throws InterruptedException {

        synchronized (lock) {
            System.out.println(name + " waiting");
            lock.wait();
            System.out.println(name + " resumed");
        }
    }

    public void signalOne() {
        synchronized (lock) {
            System.out.println("notify one waiter");
            lock.notify();
        }
    }
}
```

If several threads are waiting, call `signalOne()` repeatedly when the application condition requires multiple waiters to proceed.

---

# 12. `notify()` vs `notifyAll()` ⭐⭐⭐⭐⭐

| Feature | `notify()` | `notifyAll()` |
|---|---|---|
| Wakes | One waiting thread | All waiting threads become eligible |
| Monitor required | ✅ | ✅ |
| Releases monitor immediately | ❌ | ❌ |
| Waiting threads reacquire monitor | ✅ | ✅ |
| Selection of one waiter | Yes | Not applicable; all are eligible |
| Typical use | When one waiter is sufficient and protocol is carefully designed | Safer when multiple waiters may have different conditions |

### Golden interview line

> **`notify()` wakes one waiter; `notifyAll()` wakes all waiters, but neither one immediately transfers the monitor.**

---

# 13. Why `notifyAll()` Is Often Safer ⭐⭐⭐⭐⭐

Consider multiple condition types waiting on the same monitor:

```text
Queue not empty
Queue not full
Shutdown requested
```

A single `notify()` may wake a thread whose condition is still false.

That thread checks:

```java
while (!condition) {
    lock.wait();
}
```

and goes back to waiting.

No other eligible waiter may be selected, potentially causing progress problems depending on the protocol.

`notifyAll()` makes all waiting threads eligible to re-check their conditions.

---

# 14. Condition Loop With `notify()` ⭐⭐⭐⭐⭐

Correct pattern:

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }

    // use shared state
}
```

Notifier:

```java
synchronized (lock) {
    condition = true;
    lock.notify();
}
```

The condition change must be protected consistently by the same monitor.

---

# 15. Practice Code — Condition + `notify()` ⭐⭐⭐⭐⭐

```java
public class ConditionNotifyDemo {

    private final Object lock = new Object();
    private boolean ready;

    public void awaitReady() throws InterruptedException {
        synchronized (lock) {
            while (!ready) {
                lock.wait();
            }

            System.out.println("Worker sees ready = true");
        }
    }

    public void makeReady() {
        synchronized (lock) {
            ready = true;
            lock.notify();
        }
    }
}
```

### Why the order matters

```java
ready = true;
lock.notify();
```

The shared condition is changed before the notification, while holding the same monitor.

---

# 16. `notify()` Does Not Mean "Condition Is True" ⭐⭐⭐⭐⭐

Incorrect mental model:

```text
notify() → condition becomes true
```

Correct mental model:

```text
Thread changes shared state
       ↓
condition may now be true
       ↓
notify()
       ↓
waiting thread gets an opportunity to re-check
```

The waiting thread must still evaluate the actual condition.

---

# 17. `notify()` and Happens-Before ⭐⭐⭐⭐⭐

When a thread modifies shared state while holding a monitor and another thread later acquires the same monitor, Java's monitor synchronization provides the relevant happens-before relationship.

Example:

```java
synchronized (lock) {
    ready = true;
    lock.notify();
}
```

The notified thread later reacquires `lock` before continuing through the synchronized region.

The correctness comes from the synchronization/monitor protocol, not from `notify()` acting as an independent memory-visibility primitive.

---

# 18. Practice Code — Shared State + Notification ⭐⭐⭐⭐⭐

```java
public class NotifyVisibilityDemo {

    private final Object lock = new Object();
    private int value;

    public void reader() throws InterruptedException {
        synchronized (lock) {
            while (value == 0) {
                lock.wait();
            }

            System.out.println("Value = " + value);
        }
    }

    public void writer() {
        synchronized (lock) {
            value = 42;
            lock.notify();
        }
    }
}
```

Both the condition check and state update participate in the same monitor protocol.

---

# 19. `notify()` With No Waiting Threads ⭐⭐⭐⭐

If no thread is currently waiting on the object's monitor, calling:

```java
lock.notify();
```

does not queue a future notification for a thread that might wait later.

There is no notification "stored" for future waiters.

### Interview point

> **`notify()` is not a message queue. If no thread is waiting at that moment, the notification has no future waiter to wake.**

This is why the shared condition/state is more important than the notification itself.

---

# 20. Practice Code — Notification Is Not a Stored Signal ⭐⭐⭐⭐⭐

```java
public class NotifyNoWaiterDemo {

    private final Object lock = new Object();

    public void notifyWithoutWaiter() {
        synchronized (lock) {
            lock.notify();
            System.out.println("No waiting thread may exist here");
        }
    }

    public void waitLater() throws InterruptedException {
        synchronized (lock) {
            lock.wait();
        }
    }
}
```

If the application needs a persistent signal, model that signal explicitly as shared state, for example a boolean, counter, queue, or a higher-level concurrency primitive.

---

# 21. `notify()` Requires the Same Monitor ⭐⭐⭐⭐⭐

This is wrong:

```java
Object lockA = new Object();
Object lockB = new Object();

synchronized (lockA) {
    lockB.notify();
}
```

The current thread owns `lockA`, not `lockB`.

Calling `lockB.notify()` throws `IllegalMonitorStateException`.

Correct:

```java
synchronized (lockB) {
    lockB.notify();
}
```

---

# 22. Practice Code — Same Monitor Requirement ⭐⭐⭐⭐⭐

```java
public class SameMonitorNotifyDemo {

    private final Object lock = new Object();

    public void signal() {
        synchronized (lock) {
            lock.notify();
        }
    }
}
```

The waiter must also wait on this exact coordination object:

```java
synchronized (lock) {
    lock.wait();
}
```

---

# 23. `notify()` and `wait()` Must Use the Same Coordination Object ⭐⭐⭐⭐⭐

Correct:

```java
synchronized (lock) {
    lock.wait();
}

synchronized (lock) {
    lock.notify();
}
```

Incorrect concept:

```java
synchronized (lockA) {
    lockB.notify();
}
```

The monitor object is part of the protocol.

A useful mental model is:

```text
lock object
   ↓
condition + wait set + monitor ownership
```

---

# 24. `notify()` Does Not Wake a Thread Sleeping With `sleep()` ⭐⭐⭐⭐⭐

This does not work as a wake-up mechanism for `sleep()`:

```java
Thread.sleep(5000);
```

and:

```java
lock.notify();
```

`notify()` is for threads waiting through `Object.wait()` on that object's monitor.

To interrupt a sleeping thread, use the thread interruption mechanism instead.

---

# 25. `notify()` vs `interrupt()` ⭐⭐⭐⭐⭐

| Aspect | `notify()` | `interrupt()` |
|---|---|---|
| Target | Waiter(s) on an object's monitor | Specific thread |
| Purpose | Monitor-based coordination | Request interruption |
| Requires monitor ownership | ✅ | ❌ |
| Works with `wait()` | ✅ | ✅ |
| Works with `sleep()` | ❌ | ✅ |
| Throws `InterruptedException` itself | ❌ | Causes interruptible waits to react |

Do not use `notify()` when the requirement is to cancel/interruption-signal a specific thread.

---

# 26. Producer-Consumer With `notify()` ⭐⭐⭐⭐⭐

Classic pattern:

```java
synchronized (queue) {
    while (queue.isEmpty()) {
        queue.wait();
    }

    int value = queue.remove();
    queue.notify();
}
```

Producer:

```java
synchronized (queue) {
    queue.add(value);
    queue.notify();
}
```

This can work when the coordination protocol guarantees that one notification is sufficient.

For multiple condition types or more complex designs, `notifyAll()` or higher-level concurrency utilities may be preferable.

---

# 27. Practice Code — Producer/Consumer With `notify()` ⭐⭐⭐⭐⭐

```java
import java.util.LinkedList;
import java.util.Queue;

public class ProducerConsumerNotifyDemo {

    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 2;

    public void produce(int value) throws InterruptedException {
        synchronized (queue) {
            while (queue.size() == capacity) {
                queue.wait();
            }

            queue.add(value);
            queue.notify();
        }
    }

    public int consume() throws InterruptedException {
        synchronized (queue) {
            while (queue.isEmpty()) {
                queue.wait();
            }

            int value = queue.remove();
            queue.notify();
            return value;
        }
    }
}
```

### Interview caveat

This demonstrates the mechanism, but real producer-consumer code should generally prefer `BlockingQueue` rather than manually implementing the coordination protocol.

---

# 28. Why `notify()` Can Be Dangerous With Multiple Conditions ⭐⭐⭐⭐⭐

Suppose one monitor protects:

```text
Condition A: queue not empty
Condition B: queue not full
```

A single `notify()` can select a thread waiting for the wrong condition.

That thread may execute:

```java
while (!itsCondition) {
    lock.wait();
}
```

and return to waiting.

Depending on the protocol, another thread that could make progress may remain asleep.

This is one reason `notifyAll()` is often safer when multiple kinds of waiters share one monitor.

---

# 29. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1
Calling `notify()` outside synchronized context.

❌ `IllegalMonitorStateException`

### Mistake 2
Thinking `notify()` releases the monitor.

❌ It does not.

### Mistake 3
Thinking `notify()` immediately runs the waiter.

❌ The waiter must reacquire the monitor first.

### Mistake 4
Assuming the first waiting thread will always be selected.

❌ No application-level ordering guarantee.

### Mistake 5
Using `notify()` instead of a condition variable/state model.

❌ Notification is not stored as a future signal.

### Mistake 6
Using `if` instead of `while` around `wait()`.

❌ Condition must be rechecked.

### Mistake 7
Using different monitor objects for waiting and notification.

❌ They must participate in the same monitor protocol.

### Mistake 8
Using `notify()` to wake a thread sleeping with `Thread.sleep()`.

❌ Different mechanisms.

### Mistake 9
Assuming `notify()` guarantees visibility by itself.

❌ Correct visibility comes from the synchronization/monitor protocol.

---

# 30. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is `notify()`?

`Object.notify()` wakes one thread waiting on the object's monitor, if one exists.

### Q2. Where is `notify()` defined?

In `java.lang.Object`.

### Q3. Can we call `notify()` without synchronized?

No. The current thread must own the object's monitor or `IllegalMonitorStateException` is thrown.

### Q4. Does `notify()` release the lock?

No. The notifying thread continues to own the monitor until it leaves the synchronized region.

### Q5. Does `notify()` immediately transfer the lock?

No. The notified thread must reacquire the monitor before continuing.

### Q6. How many threads does `notify()` wake?

At most one waiting thread.

### Q7. Which waiting thread does `notify()` wake?

The API does not provide an application-level guarantee about which waiter is selected.

### Q8. What if no thread is waiting?

The notification does not become a stored future signal.

### Q9. `notify()` vs `notifyAll()`?

`notify()` makes one waiter eligible; `notifyAll()` makes all waiters eligible. Both require monitor ownership and neither immediately releases/transfers the lock.

### Q10. Why is `notifyAll()` often safer?

When multiple condition types share a monitor, waking all waiters allows each to re-check its own condition instead of relying on a single selected waiter.

### Q11. Can `notify()` wake a thread in `sleep()`?

No. `notify()` works with `Object.wait()` and the object's monitor.

### Q12. Is `notify()` a memory visibility primitive by itself?

No. Correct visibility in this pattern comes from monitor synchronization and reacquisition of the same monitor.

---

# 31. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"`notify()` is an Object method used for monitor-based thread coordination. A thread must own the object's monitor before calling it, so it is normally called inside a synchronized block or method. `notify()` makes one thread waiting on that object's monitor eligible to continue. It does not release the lock and it does not immediately transfer the lock to the waiting thread. The notifier continues holding the monitor until it exits the synchronized region, after which the awakened thread must reacquire the monitor before it can continue. If multiple threads are waiting, Java does not give the application a guarantee about which one `notify()` selects. The waiting side should always check its condition in a `while` loop. When multiple types of conditions share the same monitor, `notifyAll()` is often safer because all waiters get an opportunity to re-check their conditions. In production code, higher-level concurrency utilities are generally preferred when they express the coordination requirement directly."**

---

# 32. Quick Revision ⭐⭐⭐⭐⭐

```text
notify()
   ↓
Object method
   ↓
Current thread must own monitor
   ↓
Selects one waiting thread
   ↓
Waiting thread becomes eligible
   ↓
Notifier STILL owns monitor
   ↓
Notifier exits synchronized region
   ↓
Waiter competes to reacquire monitor
   ↓
Waiter continues
```

### Golden Rules

```text
notify() ≠ unlock()
notify() ≠ immediate context switch
notify() ≠ stored message
notify() ≠ condition becomes true

notify()
→ one waiter eligible

notifyAll()
→ all waiters eligible

wait()
→ releases waited-on monitor
```

---

# 33. Practice Checklist

- [x] `notify()` definition
- [x] Why `notify()` belongs to `Object`
- [x] Monitor ownership requirement
- [x] `IllegalMonitorStateException`
- [x] One waiter behavior
- [x] No waiter behavior
- [x] Monitor is not released by `notify()`
- [x] Reacquisition by awakened thread
- [x] `notify()` vs `notifyAll()`
- [x] Condition + `while` pattern
- [x] Shared-state visibility through monitor synchronization
- [x] Same coordination object
- [x] `notify()` vs `sleep()` / `interrupt()`
- [x] Producer-consumer example
- [x] Multiple-condition caveat
- [x] Common mistakes
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.23 — `wait()`](../23-wait/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.25 — `notifyAll()`**