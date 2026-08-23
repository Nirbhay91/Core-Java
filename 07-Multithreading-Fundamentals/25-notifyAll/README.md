# 7.25 — `notifyAll()`

## 🎯 Objective

Understand `Object.notifyAll()`, how it differs from `notify()`, monitor ownership, waiting threads, condition re-checking, and why `notifyAll()` is often safer when multiple condition types share the same monitor.

> **Interview rule:** `notifyAll()` makes **all threads waiting on this object's monitor eligible to continue**, but it does **not** release or immediately transfer the monitor. Each awakened thread must reacquire the monitor and re-check its condition.

---

## 1. What is `notifyAll()`? ⭐⭐⭐⭐⭐

`notifyAll()` is a method of `java.lang.Object` used for monitor-based thread coordination.

```java
lock.notifyAll();
```

It wakes all threads currently waiting on that object's monitor by moving them out of the object's wait set so they can compete to reacquire the monitor.

It does **not** mean all threads run simultaneously.

---

## 2. Why Does `notifyAll()` Exist?

Consider multiple waiting threads:

```text
Thread A → WAITING
Thread B → WAITING
Thread C → WAITING
```

Calling:

```java
lock.notifyAll();
```

makes all of them eligible to compete for the monitor after the notifying thread releases it.

Conceptually:

```text
notifyAll()
     ↓
all waiters become eligible
     ↓
notifier still owns monitor
     ↓
notifier exits synchronized region
     ↓
waiters compete for monitor
     ↓
one acquires it
     ↓
checks condition
     ↓
if false → waits again
```

---

## 3. `notifyAll()` Belongs to `Object` ⭐⭐⭐⭐⭐

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
    lock.notifyAll();
}
```

`wait()`, `notify()`, and `notifyAll()` are all methods of `Object` because they operate with an object's intrinsic monitor.

---

## 4. Monitor Ownership Requirement ⭐⭐⭐⭐⭐

The current thread must own the monitor before calling `notifyAll()`.

### Correct

```java
synchronized (lock) {
    lock.notifyAll();
}
```

### Incorrect

```java
lock.notifyAll();
```

The incorrect version throws:

```text
IllegalMonitorStateException
```

### Interview sentence

> **`notifyAll()` must be called while the current thread owns the monitor of the object on which it is invoked.**

---

## 5. Practice Code — Basic `notifyAll()` ⭐⭐⭐⭐⭐

```java
public class BasicNotifyAllDemo {

    private final Object lock = new Object();

    public void waitForSignal(String name) throws InterruptedException {
        synchronized (lock) {
            System.out.println(name + " waiting");
            lock.wait();
            System.out.println(name + " resumed");
        }
    }

    public void signalAll() {
        synchronized (lock) {
            System.out.println("Notifier calls notifyAll()");
            lock.notifyAll();
            System.out.println("Notifier still owns the lock");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        BasicNotifyAllDemo demo = new BasicNotifyAllDemo();

        Thread t1 = new Thread(() -> {
            try {
                demo.waitForSignal("Worker-1");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread t2 = new Thread(() -> {
            try {
                demo.waitForSignal("Worker-2");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread t3 = new Thread(() -> {
            try {
                demo.waitForSignal("Worker-3");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        t1.start();
        t2.start();
        t3.start();

        Thread.sleep(200);
        demo.signalAll();

        t1.join();
        t2.join();
        t3.join();
    }
}
```

All waiting workers become eligible, but they acquire the monitor one at a time.

---

## 6. `notifyAll()` Does NOT Release the Lock ⭐⭐⭐⭐⭐

```java
synchronized (lock) {
    lock.notifyAll();
    System.out.println("Notifier still owns the monitor");
}
```

Calling `notifyAll()` does not release the monitor.

The notifier continues executing until it leaves the synchronized region.

### Golden rule

```text
notifyAll() ≠ unlock()
```

---

## 7. Does `notifyAll()` Run All Threads Simultaneously?

**No.**

If five threads are waiting, `notifyAll()` makes all five eligible, but only one can own a given intrinsic monitor at a time.

```text
5 WAITING threads
       ↓
   notifyAll()
       ↓
5 eligible threads
       ↓
compete for monitor
       ↓
one gets lock
       ↓
checks condition
       ↓
next eligible thread gets opportunity
```

---

## 8. `while` Is Mandatory for Condition Waiting ⭐⭐⭐⭐⭐

Correct:

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }

    // condition is true
}
```

Why?

1. `notifyAll()` wakes all waiters.
2. They compete for the monitor.
3. Another thread may change the shared state before a particular thread gets the lock.
4. The condition may already be false again.
5. Therefore the condition must be checked again.

---

## 9. Practice Code — Condition + `notifyAll()` ⭐⭐⭐⭐⭐

```java
public class ConditionNotifyAllDemo {

    private final Object lock = new Object();
    private boolean ready;

    public void awaitReady(String name) throws InterruptedException {
        synchronized (lock) {
            while (!ready) {
                System.out.println(name + " waiting");
                lock.wait();
            }

            System.out.println(name + " sees ready = true");
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

Here, every waiter gets an opportunity to observe the updated condition.

---

## 10. Why `notifyAll()` Is Often Safer Than `notify()` ⭐⭐⭐⭐⭐

Suppose the same monitor protects multiple conditions:

```text
Condition A → queue is not empty
Condition B → queue is not full
```

With `notify()`, one arbitrary waiter may be selected. That thread may discover its condition is still false and return to waiting.

With `notifyAll()`:

```text
all waiters become eligible
        ↓
each checks its own condition
        ↓
threads whose condition is false wait again
        ↓
threads whose condition is true proceed
```

This avoids depending on a particular waiter-selection choice for progress.

---

## 11. `notify()` vs `notifyAll()` ⭐⭐⭐⭐⭐

| Feature | `notify()` | `notifyAll()` |
|---|---|---|
| Waiters made eligible | One | All |
| Monitor required | ✅ | ✅ |
| Immediately releases monitor | ❌ | ❌ |
| Immediate lock transfer | ❌ | ❌ |
| Which waiter selected | One, without application-level ordering guarantee | All become eligible |
| Typical risk | Can select a waiter whose condition is still false | More wake-ups / contention |
| Useful when | Exactly one waiter can safely proceed and protocol is designed for it | Multiple conditions or waiter types share a monitor |

### Interview answer

> `notify()` wakes one waiter; `notifyAll()` wakes all waiters. Neither releases the monitor immediately.

---

## 12. Practice Code — Multiple Waiters ⭐⭐⭐⭐⭐

```java
public class MultipleWaitersDemo {

    private final Object lock = new Object();

    public void waitTask(String name) throws InterruptedException {
        synchronized (lock) {
            System.out.println(name + " -> WAITING");
            lock.wait();
            System.out.println(name + " -> resumed");
        }
    }

    public void wakeEveryone() {
        synchronized (lock) {
            System.out.println("Calling notifyAll()");
            lock.notifyAll();
        }
    }
}
```

---

## 13. `notifyAll()` and the Wait Set ⭐⭐⭐⭐⭐

Each object's intrinsic monitor has an associated wait set for threads that call `wait()` on that object.

```text
Object lock
   │
   ├── Monitor owner
   └── Wait set
         ├── Thread A
         ├── Thread B
         └── Thread C
```

`notifyAll()` makes all threads in that wait set eligible to compete for the monitor.

It does not mean all threads immediately leave the JVM `WAITING` state and execute concurrently.

---

## 14. `notifyAll()` With No Waiting Threads ⭐⭐⭐⭐

If no thread is waiting on the monitor:

```java
synchronized (lock) {
    lock.notifyAll();
}
```

there are no waiters to make eligible.

The notification is not stored as a future signal.

### Important

If a thread later calls `wait()`, it does not automatically continue because `notifyAll()` happened earlier.

Model persistent conditions using shared state instead.

---

## 15. Practice Code — No Stored Notification ⭐⭐⭐⭐⭐

```java
public class NotifyAllNoWaiterDemo {

    private final Object lock = new Object();

    public void signalNow() {
        synchronized (lock) {
            lock.notifyAll();
            System.out.println("No notification is stored for future waiters");
        }
    }

    public void waitLater() throws InterruptedException {
        synchronized (lock) {
            lock.wait();
        }
    }
}
```

For a persistent signal, use state such as:

```java
boolean ready;
int count;
Queue<T> queue;
```

and protect/check it with the appropriate synchronization protocol.

---

## 16. `notifyAll()` and Happens-Before ⭐⭐⭐⭐⭐

The notification itself is not a standalone memory-visibility primitive.

Correct pattern:

```java
synchronized (lock) {
    ready = true;
    lock.notifyAll();
}
```

A waiting thread later reacquires the same monitor before continuing through its synchronized region.

The monitor synchronization establishes the relevant memory-visibility relationship.

---

## 17. Practice Code — Shared State + `notifyAll()` ⭐⭐⭐⭐⭐

```java
public class NotifyAllVisibilityDemo {

    private final Object lock = new Object();
    private int value;

    public void reader(String name) throws InterruptedException {
        synchronized (lock) {
            while (value == 0) {
                lock.wait();
            }

            System.out.println(name + " reads " + value);
        }
    }

    public void writer() {
        synchronized (lock) {
            value = 42;
            lock.notifyAll();
        }
    }
}
```

---

## 18. Important Ordering: Change State Then Notify ⭐⭐⭐⭐⭐

Preferred pattern:

```java
synchronized (lock) {
    condition = true;
    lock.notifyAll();
}
```

The notification tells waiters to reconsider the condition; the condition itself is represented by shared state.

Think:

```text
State change
    ↓
notifyAll()
    ↓
waiters re-check state
```

---

## 19. `notifyAll()` Does Not Guarantee Fairness

`notifyAll()` does not promise:

- FIFO execution
- Fair scheduling
- Equal CPU time
- A specific execution order

The awakened threads still compete for the monitor and are scheduled by the JVM/OS according to implementation behavior and scheduling policies.

Do not build correctness around an assumed order.

---

## 20. Practice Code — Do Not Assume Ordering ⭐⭐⭐⭐

```java
public class NotifyAllOrderingDemo {

    private final Object lock = new Object();

    public void await(String name) throws InterruptedException {
        synchronized (lock) {
            System.out.println(name + " waiting");
            lock.wait();
            System.out.println(name + " acquired monitor after wake-up");
        }
    }

    public void wakeAll() {
        synchronized (lock) {
            lock.notifyAll();
        }
    }
}
```

The order of resumed workers should not be treated as a contract.

---

## 21. Producer-Consumer With `notifyAll()` ⭐⭐⭐⭐⭐

For educational monitor-based producer-consumer code:

```java
synchronized (queue) {
    while (queue.isEmpty()) {
        queue.wait();
    }

    int value = queue.remove();
    queue.notifyAll();
}
```

Producer:

```java
synchronized (queue) {
    queue.add(value);
    queue.notifyAll();
}
```

`notifyAll()` is useful when both producers and consumers can be waiting on the same monitor because every waiter gets an opportunity to re-check its condition.

For production applications, prefer `BlockingQueue` when it fits the requirement.

---

## 22. Practice Code — Producer/Consumer ⭐⭐⭐⭐⭐

```java
import java.util.LinkedList;
import java.util.Queue;

public class ProducerConsumerNotifyAllDemo {

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

### Why `while`?

After being awakened, a thread must reacquire the monitor and verify that its condition is still satisfied.

---

## 23. `notifyAll()` vs `sleep()` ⭐⭐⭐⭐⭐

`notifyAll()` only coordinates threads waiting through `Object.wait()` on the corresponding monitor.

It does not wake a thread sleeping through:

```java
Thread.sleep(5000);
```

Use interruption for an interruptible sleeping thread:

```java
thread.interrupt();
```

---

## 24. `notifyAll()` vs `interrupt()` ⭐⭐⭐⭐⭐

| Aspect | `notifyAll()` | `interrupt()` |
|---|---|---|
| Target | All waiters on one object's monitor | A specific thread |
| Purpose | Condition-based coordination | Interruption/cancellation request |
| Monitor ownership | Required | Not required for caller |
| Works with `wait()` | Yes | Yes, by interrupting the wait |
| Works with `sleep()` | No | Yes |
| Main idea | Re-check shared condition | Signal interruption |

---

## 25. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — Calling outside synchronized

```java
lock.notifyAll();
```

❌ Can throw `IllegalMonitorStateException`.

### Mistake 2 — Thinking all threads run immediately

❌ They become eligible and compete for the monitor.

### Mistake 3 — Thinking `notifyAll()` releases the lock

❌ The notifier keeps the monitor until it exits the synchronized region.

### Mistake 4 — Using `if` instead of `while`

❌ Always re-check the condition after waiting.

### Mistake 5 — Assuming fairness/order

❌ No application-level ordering guarantee.

### Mistake 6 — Treating notification as stored state

❌ A future waiter does not consume an earlier notification.

### Mistake 7 — Using different monitors

❌ The wait/notification protocol must use the same coordination object.

### Mistake 8 — Using `notifyAll()` as a cancellation mechanism

❌ Use interruption or an explicit cancellation/state protocol instead.

---

# 26. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is `notifyAll()`?

`Object.notifyAll()` makes all threads waiting on the object's monitor eligible to continue.

### Q2. Does `notifyAll()` release the monitor?

No.

### Q3. Do all awakened threads execute simultaneously?

No. They must compete to reacquire the monitor.

### Q4. Why must `notifyAll()` be called inside synchronized?

Because the current thread must own the monitor of the object whose wait set it is modifying.

### Q5. What happens if it is called without monitor ownership?

`IllegalMonitorStateException`.

### Q6. Why use `while` around `wait()`?

Because a thread must re-check the actual condition after waking; another thread may have changed the state before it reacquires the monitor, and spurious wakeups are also permitted.

### Q7. `notify()` vs `notifyAll()`?

`notify()` makes one waiter eligible; `notifyAll()` makes all waiters eligible.

### Q8. Why is `notifyAll()` often safer?

When multiple condition types share a monitor, all waiters can re-check their own conditions instead of correctness depending on which single waiter was selected.

### Q9. Does `notifyAll()` guarantee execution order?

No.

### Q10. Does `notifyAll()` wake sleeping threads?

No. It coordinates `Object.wait()`, not `Thread.sleep()`.

### Q11. Is `notifyAll()` a memory visibility primitive by itself?

No. Correct visibility comes from the monitor synchronization and reacquisition protocol.

### Q12. Is `notifyAll()` always the best choice?

No. It can cause unnecessary wake-ups and contention. Higher-level concurrency utilities or more precise coordination mechanisms are often preferable in production code.

---

# 27. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"`notifyAll()` is an `Object` method used for monitor-based thread coordination. When called by a thread that owns the object's monitor, it makes all threads waiting on that object's monitor eligible to continue. It does not release the monitor and it does not mean all threads execute simultaneously. The notifying thread keeps the monitor until it exits the synchronized region, and the awakened threads then compete to reacquire it one by one. Each waiter should check its condition inside a `while` loop because another thread may change the shared state before it gets the monitor, and spurious wakeups are possible. `notifyAll()` is often safer than `notify()` when multiple kinds of conditions or waiters share one monitor, because every waiter gets an opportunity to re-check its condition. However, it can cause extra wake-ups and contention, so higher-level concurrency utilities are often preferable in production code."**

---

# 28. Quick Revision ⭐⭐⭐⭐⭐

```text
notifyAll()
      ↓
Object method
      ↓
Current thread must own monitor
      ↓
All waiters become eligible
      ↓
Notifier STILL owns monitor
      ↓
Notifier exits synchronized region
      ↓
Waiters compete for monitor
      ↓
One acquires monitor at a time
      ↓
Each checks condition
      ↓
false → wait again
true  → continue
```

### Golden Rules

```text
notify()      → one waiter eligible
notifyAll()   → all waiters eligible
wait()        → releases waited-on monitor
notifyAll()   → does NOT release monitor
notifyAll()   → does NOT guarantee order
wait()        → always use while(condition-not-met)
```

---

# 29. Practice Checklist

- [x] `notifyAll()` definition
- [x] Why it belongs to `Object`
- [x] Monitor ownership
- [x] `IllegalMonitorStateException`
- [x] All waiter behavior
- [x] Monitor is not released by `notifyAll()`
- [x] Reacquisition behavior
- [x] `notify()` vs `notifyAll()`
- [x] Condition + `while` pattern
- [x] Multiple-condition scenario
- [x] Wait set concept
- [x] No stored notification
- [x] Happens-before / visibility through synchronization
- [x] Producer-consumer practice
- [x] Ordering/fairness caveat
- [x] `notifyAll()` vs `sleep()` / `interrupt()`
- [x] Common interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.24 — `notify()`](../24-notify/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.26 — Monitor Ownership with `wait/notify`**