# 7.26 — Monitor Ownership with `wait/notify`

## 🎯 Objective

Understand the most important rule behind `wait()`, `notify()`, and `notifyAll()`:

> **A thread must own the monitor of the same object before it can call `wait()`, `notify()`, or `notifyAll()` on that object.**

This topic connects intrinsic monitors, `synchronized`, `wait/notify`, and `IllegalMonitorStateException`.

---

## 1. What Is Monitor Ownership? ⭐⭐⭐⭐⭐

Every Java object can have an **intrinsic monitor** associated with it.

When a thread enters:

```java
synchronized (lock) {
    // critical section
}
```

the thread acquires/owns the monitor associated with `lock` for the duration of that synchronized execution.

Conceptually:

```text
Object: lock
       ↓
Intrinsic Monitor
       ↓
Current owner → Thread-A
```

Only the monitor owner can perform monitor operations such as `wait()`, `notify()`, and `notifyAll()` on that object.

---

## 2. The Core Rule ⭐⭐⭐⭐⭐

For:

```java
lock.wait();
lock.notify();
lock.notifyAll();
```

the current thread must own `lock`'s monitor.

Usually this means:

```java
synchronized (lock) {
    lock.wait();
}
```

and:

```java
synchronized (lock) {
    lock.notifyAll();
}
```

### Interview sentence

> **`wait()`, `notify()`, and `notifyAll()` are monitor operations, so the calling thread must own the monitor of the object on which the method is invoked.**

---

## 3. Why Are These Methods Defined in `Object`? ⭐⭐⭐⭐

`wait()`, `notify()`, and `notifyAll()` are methods of `java.lang.Object`, not `Thread`.

Reason:

```text
Object
  ↓
Intrinsic Monitor
  ↓
wait set + ownership
```

The coordination is associated with an object's monitor.

That is why this is valid:

```java
Object lock = new Object();
lock.wait();
```

provided the caller owns `lock`'s monitor.

---

## 4. Correct `wait()` Usage ⭐⭐⭐⭐⭐

```java
Object lock = new Object();

synchronized (lock) {
    lock.wait();
}
```

What happens?

```text
Thread owns lock
      ↓
wait() called
      ↓
Thread enters WAITING
      ↓
Thread releases lock's monitor
      ↓
Another thread can acquire lock
```

Important:

> `wait()` releases the monitor **after the thread enters the waiting protocol**; the waiting thread does not continue holding that monitor while waiting.

---

## 5. Correct `notify()` Usage ⭐⭐⭐⭐⭐

```java
Object lock = new Object();

synchronized (lock) {
    lock.notify();
}
```

The notifier owns the monitor while calling `notify()`.

`notify()` makes one waiting thread eligible to compete for the monitor.

It does **not** immediately transfer the monitor to that thread.

---

## 6. Correct `notifyAll()` Usage ⭐⭐⭐⭐⭐

```java
Object lock = new Object();

synchronized (lock) {
    lock.notifyAll();
}
```

All threads waiting on that object's monitor become eligible to compete for the monitor after it becomes available.

The notifier still owns the monitor until it exits the synchronized region.

---

# 7. Practice Code — Correct Monitor Ownership ⭐⭐⭐⭐⭐

```java
public class MonitorOwnershipDemo {

    private final Object lock = new Object();
    private boolean ready;

    public void waitForReady() throws InterruptedException {
        synchronized (lock) {
            while (!ready) {
                System.out.println(Thread.currentThread().getName()
                        + " waiting");
                lock.wait();
            }

            System.out.println(Thread.currentThread().getName()
                    + " continuing");
        }
    }

    public void makeReady() {
        synchronized (lock) {
            ready = true;
            lock.notifyAll();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        MonitorOwnershipDemo demo = new MonitorOwnershipDemo();

        Thread worker = new Thread(() -> {
            try {
                demo.waitForReady();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Worker");

        worker.start();

        Thread.sleep(200);
        demo.makeReady();

        worker.join();
    }
}
```

### Flow

```text
Worker
  ↓
synchronized(lock)
  ↓
owns monitor
  ↓
wait()
  ↓
WAITING + releases monitor

Main
  ↓
synchronized(lock)
  ↓
owns monitor
  ↓
ready = true
  ↓
notifyAll()
  ↓
exits synchronized block

Worker
  ↓
reacquires monitor
  ↓
checks while condition
  ↓
continues
```

---

# 8. What Happens Without Monitor Ownership? ⭐⭐⭐⭐⭐

This is incorrect:

```java
Object lock = new Object();
lock.wait();
```

The caller does not own `lock`'s monitor.

Result:

```text
IllegalMonitorStateException
```

The same applies to:

```java
lock.notify();
```

and:

```java
lock.notifyAll();
```

when called without owning the monitor.

---

## 9. Practice Code — `wait()` Without Ownership ⭐⭐⭐⭐⭐

```java
public class InvalidWaitDemo {

    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();

        // WRONG
        lock.wait();
    }
}
```

Expected result:

```text
java.lang.IllegalMonitorStateException
```

Correct version:

```java
synchronized (lock) {
    lock.wait();
}
```

---

# 10. Practice Code — `notify()` Without Ownership ⭐⭐⭐⭐⭐

```java
public class InvalidNotifyDemo {

    public static void main(String[] args) {
        Object lock = new Object();

        // WRONG
        lock.notify();
    }
}
```

Expected result:

```text
java.lang.IllegalMonitorStateException
```

Correct:

```java
synchronized (lock) {
    lock.notify();
}
```

---

# 11. Practice Code — `notifyAll()` Without Ownership ⭐⭐⭐⭐⭐

```java
public class InvalidNotifyAllDemo {

    public static void main(String[] args) {
        Object lock = new Object();

        // WRONG
        lock.notifyAll();
    }
}
```

Expected result:

```text
java.lang.IllegalMonitorStateException
```

Correct:

```java
synchronized (lock) {
    lock.notifyAll();
}
```

---

# 12. Same Object Is Important ⭐⭐⭐⭐⭐

Consider:

```java
Object lock1 = new Object();
Object lock2 = new Object();
```

This is a different monitor:

```java
synchronized (lock1) {
    lock2.notifyAll(); // WRONG
}
```

The thread owns `lock1`'s monitor, not `lock2`'s monitor.

Result:

```text
IllegalMonitorStateException
```

Correct:

```java
synchronized (lock2) {
    lock2.notifyAll();
}
```

### Golden rule

```text
Object whose method is called
        ==
Object whose monitor you own
```

---

# 13. Practice Code — Two Different Locks ⭐⭐⭐⭐⭐

```java
public class TwoLocksDemo {

    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void invalidNotification() {
        synchronized (lockA) {
            // lockA monitor is owned
            // lockB monitor is NOT owned
            lockB.notifyAll(); // IllegalMonitorStateException
        }
    }

    public void validNotification() {
        synchronized (lockB) {
            lockB.notifyAll();
        }
    }
}
```

This is one of the most common interview traps.

---

# 14. Monitor Ownership vs Thread Ownership ⭐⭐⭐⭐⭐

A thread can own multiple monitors at the same time if it enters nested synchronized blocks.

Example:

```java
synchronized (lockA) {
    synchronized (lockB) {
        // current thread owns both monitors
    }
}
```

Conceptually:

```text
Thread-A
   │
   ├── owns lockA monitor
   │
   └── owns lockB monitor
```

When it exits the inner block, it releases `lockB`.

When it exits the outer block, it releases `lockA`.

---

# 15. Practice Code — Nested Monitor Ownership ⭐⭐⭐⭐⭐

```java
public class NestedMonitorDemo {

    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void execute() {
        synchronized (lockA) {
            System.out.println("Owned lockA");

            synchronized (lockB) {
                System.out.println("Owned lockA and lockB");
                lockB.notifyAll();
            }

            System.out.println("Released lockB, still own lockA");
        }

        System.out.println("Released lockA");
    }
}
```

---

# 16. Reentrancy and Monitor Ownership ⭐⭐⭐⭐⭐

Java intrinsic monitors are **reentrant**.

If a thread already owns a monitor, it can enter another synchronized section guarded by the same monitor.

```java
synchronized (lock) {
    method();
}

void method() {
    synchronized (lock) {
        // Same thread can reacquire the same monitor
    }
}
```

The same thread can call `wait()` only while it owns the monitor.

After `wait()` returns, the thread must reacquire that monitor before continuing beyond the `wait()` call.

---

# 17. `wait()` Releases Which Lock? ⭐⭐⭐⭐⭐

This is a major interview question.

```java
synchronized (lockA) {
    synchronized (lockB) {
        lockB.wait();
    }
}
```

When the thread waits on `lockB`:

```text
lockB monitor → released
lockA monitor → still held
```

The thread is waiting for `lockB`, but it does not automatically release unrelated monitors it owns.

This can contribute to deadlock if monitor ownership is designed poorly.

---

# 18. Practice Code — `wait()` Releases Only the Waited Monitor ⭐⭐⭐⭐⭐

```java
public class WaitReleaseDemo {

    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void waitOnB() throws InterruptedException {
        synchronized (lockA) {
            System.out.println("Acquired lockA");

            synchronized (lockB) {
                System.out.println("Acquired lockB");
                lockB.wait();
                System.out.println("Returned from wait()");
            }
        }
    }
}
```

While waiting on `lockB`:

```text
lockB → released
lockA → still owned
```

This is an important distinction between `wait()` and simply blocking for a monitor.

---

# 19. `wait()` vs Entering `synchronized` ⭐⭐⭐⭐⭐

### `synchronized`

If a thread cannot acquire the monitor:

```java
synchronized (lock) {
}
```

it waits to acquire the monitor but does not execute `wait()` and does not voluntarily release some other monitor it may hold.

### `wait()`

If the thread owns the monitor and calls:

```java
lock.wait();
```

it enters `WAITING` and releases that object's monitor.

---

# 20. Monitor Ownership and `notifyAll()` ⭐⭐⭐⭐⭐

Suppose:

```text
lock wait set:
  Thread-A
  Thread-B
  Thread-C
```

Thread-D executes:

```java
synchronized (lock) {
    lock.notifyAll();
}
```

Then:

```text
Thread-D owns monitor
        ↓
notifyAll()
        ↓
A/B/C become eligible
        ↓
D still owns monitor
        ↓
D exits synchronized block
        ↓
A/B/C compete for monitor
```

Only after acquiring the monitor can an awakened waiter proceed beyond its `wait()` call.

---

# 21. Practice Code — Notification Does Not Transfer Ownership ⭐⭐⭐⭐⭐

```java
public class NotificationOwnershipDemo {

    private final Object lock = new Object();

    public void waitForSignal() throws InterruptedException {
        synchronized (lock) {
            System.out.println("Worker waiting");
            lock.wait();
            System.out.println("Worker reacquired monitor");
        }
    }

    public void signal() {
        synchronized (lock) {
            System.out.println("Notifier owns monitor");
            lock.notifyAll();
            System.out.println("Notifier still owns monitor after notifyAll()");
        }

        System.out.println("Notifier left synchronized block");
    }
}
```

The print order demonstrates the key concept: notification does not immediately transfer ownership.

---

# 22. Condition Predicate + Monitor Ownership ⭐⭐⭐⭐⭐

A robust monitor protocol generally looks like:

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }

    // use protected state
}
```

Notifier:

```java
synchronized (lock) {
    changeState();
    lock.notifyAll();
}
```

Both the condition check/state update and the coordination use the same monitor.

---

# 23. Practice Code — Complete Condition Protocol ⭐⭐⭐⭐⭐

```java
public class CompleteMonitorProtocolDemo {

    private final Object lock = new Object();
    private boolean available;

    public void consume() throws InterruptedException {
        synchronized (lock) {
            while (!available) {
                lock.wait();
            }

            available = false;
            System.out.println("Consumed resource");
        }
    }

    public void produce() {
        synchronized (lock) {
            available = true;
            System.out.println("Produced resource");
            lock.notifyAll();
        }
    }
}
```

The same monitor protects:

- the condition (`available`)
- the state change
- the wait operation
- the notification

---

# 24. Why `if` Is Not Enough ⭐⭐⭐⭐⭐

Incorrect:

```java
synchronized (lock) {
    if (!available) {
        lock.wait();
    }

    consume();
}
```

Correct:

```java
synchronized (lock) {
    while (!available) {
        lock.wait();
    }

    consume();
}
```

The thread must validate the condition after reacquiring the monitor.

---

# 25. Monitor Ownership Does Not Mean Application-Level Ownership ⭐⭐⭐⭐

Monitor ownership is a JVM synchronization concept.

It means:

```text
Current thread owns the intrinsic monitor of object X
```

It does **not** automatically mean:

```text
Current thread owns the business resource
```

Keep these concepts separate in design discussions.

---

# 26. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1

> `wait()` can be called anywhere because it is an Object method.

❌ False. The caller must own the object's monitor.

### Trap 2

> `notify()` releases the lock.

❌ False.

### Trap 3

> `notifyAll()` immediately runs every waiter.

❌ False.

### Trap 4

> Owning `lockA` allows notifying `lockB`.

❌ False.

### Trap 5

> `wait()` releases every lock held by the thread.

❌ False. It releases the monitor of the object on which `wait()` was called; unrelated monitors remain held.

### Trap 6

> A previous `notifyAll()` is stored for future waiters.

❌ False.

### Trap 7

> `wait()` should normally be inside `while`.

✅ Correct.

---

# 27. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is monitor ownership?

It means a thread currently owns the intrinsic monitor associated with a particular object.

### Q2. Why does `wait()` require monitor ownership?

Because `wait()` operates on that object's monitor and wait set, and Java requires the caller to own that monitor.

### Q3. What exception occurs when ownership is missing?

`IllegalMonitorStateException`.

### Q4. Does `wait()` release the monitor?

Yes, it releases the monitor of the object on which `wait()` was invoked while the thread waits.

### Q5. Does `wait()` release all monitors owned by the thread?

No.

### Q6. Does `notify()` release the monitor?

No.

### Q7. Does `notifyAll()` release the monitor?

No.

### Q8. What must match in a wait/notify protocol?

The object used for `wait()` and notification must be the same coordination object whose monitor is owned.

### Q9. Can a thread own multiple monitors?

Yes, through nested synchronized regions.

### Q10. Why is `while` preferred over `if` around `wait()`?

Because the condition must be re-checked after the thread reacquires the monitor.

### Q11. What happens to unrelated locks when `wait()` is called?

They remain held by the thread.

### Q12. What is the relationship between `synchronized` and monitor ownership?

Entering a synchronized block/method acquires the relevant intrinsic monitor; leaving it releases that monitor.

---

# 28. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Monitor ownership means that a thread currently owns the intrinsic monitor of a Java object. `wait()`, `notify()`, and `notifyAll()` are monitor operations, so the calling thread must own the monitor of the exact object on which the method is invoked. Otherwise Java throws `IllegalMonitorStateException`. A typical wait protocol is `synchronized(lock)`, followed by a `while` condition check and `lock.wait()`. `wait()` releases the monitor of `lock` while the thread waits and reacquires it before returning from `wait()`. `notify()` and `notifyAll()` make waiting threads eligible, but they do not release or immediately transfer the monitor. Also, `wait()` releases only the monitor on which it is called, not unrelated monitors the thread may hold. The same monitor should protect the condition, state changes, waiting, and notification."**

---

# 29. Quick Revision ⭐⭐⭐⭐⭐

```text
synchronized(lock)
       ↓
thread owns lock monitor
       ↓
wait()/notify()/notifyAll() allowed
       ↓
wait()
       → releases lock monitor
       → WAITING
       → later reacquires lock

notify()
       → one waiter eligible
       → notifier keeps monitor

notifyAll()
       → all waiters eligible
       → notifier keeps monitor
```

### Golden Rules

```text
Own monitor → then wait/notify/notifyAll
Wrong monitor → IllegalMonitorStateException
wait() → releases waited-on monitor
wait() → does NOT release unrelated monitors
notify() → does NOT release monitor
notifyAll() → does NOT release monitor
Use while(condition) around wait()
Use the SAME coordination object
```

---

# 30. Practice Checklist

- [x] Intrinsic monitor
- [x] Monitor ownership
- [x] `synchronized` and monitor acquisition
- [x] `wait()` ownership requirement
- [x] `notify()` ownership requirement
- [x] `notifyAll()` ownership requirement
- [x] `IllegalMonitorStateException`
- [x] Same-object monitor rule
- [x] `wait()` releases waited-on monitor
- [x] Unrelated monitors remain held
- [x] Nested monitor ownership
- [x] Reentrancy
- [x] Condition + `while`
- [x] Notification vs monitor ownership
- [x] Common interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.25 — `notifyAll()`](../25-notifyAll/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.27 — `wait()` vs `sleep()`**