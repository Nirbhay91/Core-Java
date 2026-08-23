# 7.27 — `wait()` vs `sleep()`

## 🎯 Objective

Understand the difference between `Object.wait()` and `Thread.sleep()`—especially **monitor ownership, lock release, wake-up mechanism, interruption, and common interview traps**.

> **Golden rule:** `wait()` is a monitor/condition-coordination mechanism; `sleep()` is a time-based pause. The biggest difference is that `wait()` releases the monitor of the object being waited on, while `sleep()` does **not** release monitors held by the sleeping thread.

---

## 1. Quick Comparison ⭐⭐⭐⭐⭐

| Feature | `wait()` | `sleep()` |
|---|---|---|
| Declared in | `Object` | `Thread` |
| Main purpose | Thread coordination | Pause execution for a duration |
| Requires monitor ownership | Yes | No |
| Releases waited-on monitor | Yes | No |
| Must be inside `synchronized` on same object | Yes | No |
| How it resumes | Notification, interruption, timeout, or spurious wakeup | Timeout or interruption |
| Can be indefinite | Yes, plain `wait()` | No; duration is specified |
| Throws `InterruptedException` | Yes | Yes |
| Condition-based | Yes | No |
| Typical use | Producer-consumer / condition waiting | Delay / pacing / retry delay |

---

# 2. `wait()` Fundamentals ⭐⭐⭐⭐⭐

`wait()` is an `Object` method.

```java
synchronized (lock) {
    lock.wait();
}
```

The calling thread must own `lock`'s monitor.

When it waits:

```text
Thread owns lock
      ↓
wait()
      ↓
WAITING
      ↓
releases lock monitor
      ↓
another thread can acquire lock
```

It later needs to reacquire the same monitor before returning from `wait()`.

---

# 3. `sleep()` Fundamentals ⭐⭐⭐⭐⭐

`sleep()` is a static method of `Thread`.

```java
Thread.sleep(1000);
```

It pauses the currently executing thread for approximately the requested duration, subject to scheduling and timing limitations.

Important:

> `sleep()` does **not** release monitors held by the thread.

---

# 4. Most Important Difference — Lock Release ⭐⭐⭐⭐⭐

Consider:

```java
synchronized (lock) {
    lock.wait();
}
```

`wait()` releases `lock` while waiting.

But:

```java
synchronized (lock) {
    Thread.sleep(5000);
}
```

`sleep()` does **not** release `lock` during those five seconds.

### Interview line

> **`wait()` releases the monitor of the object on which it is called; `sleep()` does not release any monitor.**

---

# 5. Practice Code — `wait()` Releases the Lock ⭐⭐⭐⭐⭐

```java
public class WaitReleasesLockDemo {

    private final Object lock = new Object();

    public void waitInsideLock() throws InterruptedException {
        synchronized (lock) {
            System.out.println(Thread.currentThread().getName()
                    + " acquired lock");

            lock.wait();

            System.out.println(Thread.currentThread().getName()
                    + " reacquired lock after wait");
        }
    }

    public void useLock() {
        synchronized (lock) {
            System.out.println(Thread.currentThread().getName()
                    + " acquired lock while another thread was waiting");
        }
    }
}
```

A different thread can acquire `lock` while the first thread is waiting.

---

# 6. Practice Code — `sleep()` Does NOT Release the Lock ⭐⭐⭐⭐⭐

```java
public class SleepDoesNotReleaseLockDemo {

    private final Object lock = new Object();

    public void sleepInsideLock() throws InterruptedException {
        synchronized (lock) {
            System.out.println(Thread.currentThread().getName()
                    + " acquired lock");

            Thread.sleep(3000);

            System.out.println(Thread.currentThread().getName()
                    + " woke up and still owns lock");
        }
    }

    public void useLock() {
        synchronized (lock) {
            System.out.println(Thread.currentThread().getName()
                    + " acquired lock");
        }
    }
}
```

A second thread attempting the same monitor must wait for the sleeping thread to leave the synchronized region.

---

# 7. Side-by-Side Timeline ⭐⭐⭐⭐⭐

### `wait()`

```text
Thread-A
   ↓
acquire lock
   ↓
wait()
   ↓
release lock
   ↓
WAITING
   ↓
notify/timeout/interrupt
   ↓
reacquire lock
   ↓
continue
```

### `sleep()`

```text
Thread-A
   ↓
acquire lock
   ↓
sleep()
   ↓
TIMED_WAITING
   ↓
STILL HOLDS LOCK
   ↓
timeout/interrupt
   ↓
continue
   ↓
release lock later
```

---

# 8. `wait()` Needs the Correct Monitor ⭐⭐⭐⭐⭐

This is correct:

```java
synchronized (lock) {
    lock.wait();
}
```

This is wrong:

```java
lock.wait();
```

without owning `lock`'s monitor.

Result:

```text
IllegalMonitorStateException
```

---

# 9. `sleep()` Does Not Need a Monitor ⭐⭐⭐⭐⭐

This is valid:

```java
Thread.sleep(1000);
```

No `synchronized` block is required.

```java
public void pause() throws InterruptedException {
    Thread.sleep(1000);
}
```

---

# 10. Practice Code — Invalid `wait()` vs Valid `sleep()` ⭐⭐⭐⭐⭐

```java
public class WaitSleepMonitorDemo {

    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();

        // Correct for sleep: no monitor required
        Thread.sleep(100);

        // Wrong for wait: current thread does not own lock's monitor
        // lock.wait();

        synchronized (lock) {
            lock.wait(100);
        }
    }
}
```

The timed `wait(100)` is used here only to make the example self-contained; normal condition waiting should usually use a condition + `while` loop.

---

# 11. How Does `wait()` Resume? ⭐⭐⭐⭐⭐

A thread in `wait()` can stop waiting because of several events:

1. Another thread calls `notify()` on the same object.
2. Another thread calls `notifyAll()` on the same object.
3. A timed wait reaches its timeout.
4. The waiting thread is interrupted.
5. A spurious wakeup occurs.

Therefore:

```java
while (!condition) {
    lock.wait();
}
```

is the standard pattern.

---

# 12. How Does `sleep()` Resume? ⭐⭐⭐⭐⭐

A sleeping thread normally resumes after its requested sleep duration has elapsed, subject to scheduling.

It can also be interrupted:

```java
thread.interrupt();
```

which causes `sleep()` to throw `InterruptedException`.

There is no `notify()` / `notifyAll()` mechanism for `sleep()`.

---

# 13. Practice Code — `wait()` + `notifyAll()` ⭐⭐⭐⭐⭐

```java
public class WaitNotifyDemo {

    private final Object lock = new Object();
    private boolean ready;

    public void awaitReady() throws InterruptedException {
        synchronized (lock) {
            while (!ready) {
                System.out.println("Waiting for ready state...");
                lock.wait();
            }

            System.out.println("Ready state observed");
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

This is condition-based coordination.

---

# 14. Practice Code — `sleep()` for Delay ⭐⭐⭐⭐

```java
public class SleepDelayDemo {

    public static void main(String[] args) throws InterruptedException {
        for (int i = 1; i <= 3; i++) {
            System.out.println("Attempt " + i);
            Thread.sleep(1000);
        }

        System.out.println("Done");
    }
}
```

This is a delay/pacing use case, not condition synchronization.

---

# 15. `wait()` vs `sleep()` — Thread State ⭐⭐⭐⭐⭐

For a normal call:

```java
lock.wait();
```

the thread enters `WAITING`.

For:

```java
Thread.sleep(1000);
```

the thread enters `TIMED_WAITING`.

For timed wait:

```java
lock.wait(1000);
```

the thread enters `TIMED_WAITING`.

### Important

Java's `Thread.State` does not have a separate portable `RUNNING` state; running and ready-to-run threads are represented by `RUNNABLE`.

---

# 16. Practice Code — Observe Thread States ⭐⭐⭐⭐⭐

```java
public class WaitSleepStateDemo {

    public static void main(String[] args) throws Exception {
        Object lock = new Object();

        Thread waiter = new Thread(() -> {
            synchronized (lock) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "Waiter");

        Thread sleeper = new Thread(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Sleeper");

        waiter.start();
        sleeper.start();

        Thread.sleep(100);

        System.out.println("Waiter  = " + waiter.getState());
        System.out.println("Sleeper = " + sleeper.getState());

        synchronized (lock) {
            lock.notifyAll();
        }

        waiter.join();
        sleeper.join();
    }
}
```

Typical observation:

```text
Waiter  = WAITING
Sleeper = TIMED_WAITING
```

Exact timing can affect observations, so don't treat a tiny sleep in a demo as a synchronization guarantee.

---

# 17. `wait()` Releases Only the Waited-On Monitor ⭐⭐⭐⭐⭐

Consider:

```java
synchronized (lockA) {
    synchronized (lockB) {
        lockB.wait();
    }
}
```

While waiting on `lockB`:

```text
lockB → released
lockA → still held
```

This is a major interview point.

`sleep()` is different:

```java
synchronized (lockA) {
    Thread.sleep(5000);
}
```

During sleep:

```text
lockA → still held
```

---

# 18. Practice Code — Two Locks ⭐⭐⭐⭐⭐

```java
public class TwoLockWaitSleepDemo {

    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void waitOnB() throws InterruptedException {
        synchronized (lockA) {
            synchronized (lockB) {
                lockB.wait();
            }
        }
    }

    public void sleepWithA() throws InterruptedException {
        synchronized (lockA) {
            Thread.sleep(2000);
        }
    }
}
```

### Remember

```text
wait on B → releases B, keeps A
sleep     → releases neither
```

---

# 19. Interruption ⭐⭐⭐⭐⭐

Both methods are interruptible:

```java
try {
    Thread.sleep(5000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

and:

```java
try {
    synchronized (lock) {
        lock.wait();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

A good practice is to restore the interrupt status when the current layer cannot fully handle the interruption:

```java
Thread.currentThread().interrupt();
```

---

# 20. Practice Code — Interrupt Both ⭐⭐⭐⭐⭐

```java
public class InterruptWaitSleepDemo {

    public static void main(String[] args) throws Exception {
        Object lock = new Object();

        Thread waiter = new Thread(() -> {
            try {
                synchronized (lock) {
                    lock.wait();
                }
            } catch (InterruptedException e) {
                System.out.println("Waiter interrupted");
                Thread.currentThread().interrupt();
            }
        });

        Thread sleeper = new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                System.out.println("Sleeper interrupted");
                Thread.currentThread().interrupt();
            }
        });

        waiter.start();
        sleeper.start();

        Thread.sleep(200);

        waiter.interrupt();
        sleeper.interrupt();

        waiter.join();
        sleeper.join();
    }
}
```

---

# 21. Why `sleep()` Inside `synchronized` Can Be Dangerous ⭐⭐⭐⭐

Example:

```java
synchronized (lock) {
    Thread.sleep(5000);
    updateState();
}
```

The thread keeps the monitor during the sleep.

Other threads that need the same monitor can remain blocked unnecessarily.

Better design:

```java
Thread.sleep(5000);

synchronized (lock) {
    updateState();
}
```

when the delay does not need to happen while holding the lock.

> Do not blindly move code outside synchronization; preserve the required correctness of the shared-state protocol.

---

# 22. `wait()` Is Not a General Delay Mechanism ⭐⭐⭐⭐⭐

Avoid using:

```java
synchronized (lock) {
    lock.wait(5000);
}
```

just because you need a five-second delay.

If you simply need a delay, use:

```java
Thread.sleep(5000);
```

Use `wait()` when the thread is waiting for a **condition/state change** coordinated through a monitor.

---

# 23. Practice Code — Correct Choice ⭐⭐⭐⭐⭐

### Delay

```java
Thread.sleep(1000);
```

### Condition waiting

```java
synchronized (lock) {
    while (!ready) {
        lock.wait();
    }
}
```

### Notification

```java
synchronized (lock) {
    ready = true;
    lock.notifyAll();
}
```

This separates:

```text
Delay              → sleep()
Condition waiting  → wait()
Signal condition   → notify()/notifyAll()
```

---

# 24. `wait()` vs `sleep()` — Interview Table ⭐⭐⭐⭐⭐

| Question | `wait()` | `sleep()` |
|---|---|---|
| Class | `Object` | `Thread` |
| Static? | No | Yes |
| Monitor ownership required? | Yes | No |
| Releases monitor? | Yes, the waited-on monitor | No |
| Used for communication? | Yes | No |
| Uses notification? | Yes | No |
| Can be interrupted? | Yes | Yes |
| Plain call state | `WAITING` | `TIMED_WAITING` |
| Timeout form | `wait(timeout)` | `sleep(timeout)` |
| Condition re-check needed? | Yes | Not as a monitor-condition protocol |
| Typical use | Coordination | Delay/pacing |

---

# 25. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1

> `sleep()` releases the lock.

❌ False.

### Trap 2

> `wait()` is a method of `Thread`.

❌ False. It is inherited from `Object`.

### Trap 3

> `sleep()` requires `synchronized`.

❌ False.

### Trap 4

> `wait()` can be called without owning the monitor.

❌ False. It causes `IllegalMonitorStateException`.

### Trap 5

> `notify()` wakes a sleeping thread.

❌ False. It coordinates `wait()`.

### Trap 6

> `wait()` is just another way to sleep.

❌ False. `wait()` participates in monitor-based condition coordination and releases the waited-on monitor.

### Trap 7

> `sleep()` always pauses for exactly the requested duration.

❌ The thread cannot resume before the sleep duration has elapsed, but actual execution after that depends on scheduling and system timing.

### Trap 8

> `wait()` releases all locks held by the thread.

❌ False. It releases the monitor of the object on which it waits, not unrelated monitors.

---

# 26. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is the biggest difference between `wait()` and `sleep()`?

`wait()` releases the monitor of the object it waits on; `sleep()` does not release monitors.

### Q2. Where is `wait()` defined?

`java.lang.Object`.

### Q3. Where is `sleep()` defined?

`java.lang.Thread`.

### Q4. Does `wait()` require synchronization?

The thread must own the monitor of the object on which `wait()` is invoked, normally achieved with `synchronized`.

### Q5. Does `sleep()` require synchronization?

No.

### Q6. Can `sleep()` release a lock?

No.

### Q7. Can `wait()` be interrupted?

Yes. It throws `InterruptedException`.

### Q8. Can `sleep()` be interrupted?

Yes. It throws `InterruptedException`.

### Q9. What state does `wait()` normally produce?

`WAITING`; timed wait produces `TIMED_WAITING`.

### Q10. What state does `sleep()` produce?

`TIMED_WAITING`.

### Q11. Why use `wait()` instead of `sleep()` for producer-consumer?

Because the consumer needs to release the monitor while waiting for the condition and be coordinated when the shared state changes.

### Q12. Why is `sleep()` inside synchronized code potentially problematic?

It keeps the monitor while sleeping and can unnecessarily block other threads that need the same monitor.

### Q13. Does `wait()` guarantee that the thread resumes immediately after `notify()`?

No. The notifier still owns the monitor, and the awakened thread must reacquire it before continuing.

### Q14. Does `sleep()` have a `notify()` mechanism?

No.

---

# 27. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"`wait()` and `sleep()` are fundamentally different. `wait()` is an `Object` method used for monitor-based thread coordination, while `sleep()` is a static `Thread` method used to pause execution for a duration. To call `wait()`, the current thread must own the object's monitor, normally through `synchronized`. When `wait()` starts waiting, it releases that object's monitor and later reacquires it before returning. `sleep()` does not require monitor ownership and does not release any monitor held by the sleeping thread. `wait()` can resume due to notification, timeout, interruption, or a spurious wakeup, so condition waits should use a `while` loop. `sleep()` normally resumes after its timeout, subject to scheduling, or can be interrupted. In short: `wait()` is for condition-based coordination; `sleep()` is for time-based delay."**

---

# 28. Quick Revision ⭐⭐⭐⭐⭐

```text
wait()
  → Object method
  → monitor ownership required
  → releases waited-on monitor
  → WAITING / TIMED_WAITING
  → condition coordination
  → notify / notifyAll / timeout / interrupt / spurious wakeup
  → reacquires monitor before returning

sleep()
  → Thread method
  → monitor ownership NOT required
  → releases NO monitors
  → TIMED_WAITING
  → time-based delay
  → timeout / interrupt
```

### One-line memory trick

> **WAIT = Give up the monitor and wait for a condition. SLEEP = Keep your monitors and pause for time.**

---

# 29. Practice Checklist

- [x] `wait()` definition
- [x] `sleep()` definition
- [x] `Object` vs `Thread`
- [x] Monitor ownership
- [x] Lock release difference
- [x] `wait()` wake-up mechanisms
- [x] `sleep()` wake-up mechanism
- [x] Thread states
- [x] `wait()` + `notifyAll()` practice
- [x] `sleep()` delay practice
- [x] Interrupt behavior
- [x] Multiple-monitor behavior
- [x] `sleep()` inside synchronized
- [x] Condition + `while`
- [x] Common interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.26 — Monitor Ownership with `wait/notify`](../26-Monitor-Ownership-with-wait-notify/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.28 — Thread Interruption**