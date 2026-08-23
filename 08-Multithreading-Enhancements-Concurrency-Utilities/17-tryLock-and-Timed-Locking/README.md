# 8.17 — `tryLock()` and Timed Locking

> **Goal:** Learn how to acquire a `ReentrantLock` without waiting forever, including immediate `tryLock()`, timed acquisition, interruption, fallback design, and deadlock-avoidance patterns.

---

## 1. Why `tryLock()`? ⭐⭐⭐⭐⭐

`lock.lock()` waits until the lock becomes available.

```java
lock.lock();
```

`tryLock()` attempts to acquire the lock immediately:

```java
if (lock.tryLock()) {
    try {
        // critical section
    } finally {
        lock.unlock();
    }
} else {
    // lock unavailable
}
```

### Memory Trick

```text
lock()
→ Wait until acquired

tryLock()
→ Try now → true / false

tryLock(timeout)
→ Wait up to a limit → true / false
```

---

# 2. Basic `tryLock()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class BasicTryLockExample {

    private static final ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) {

        if (lock.tryLock()) {
            try {
                System.out.println("Lock acquired");
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println("Lock unavailable");
        }
    }
}
```

### Important

`tryLock()` does **not** mean "try for a little while". The no-argument version returns immediately if the lock cannot be acquired.

---

# 3. `lock()` vs `tryLock()` ⭐⭐⭐⭐⭐

| Feature | `lock()` | `tryLock()` |
|---|---|---|
| Waits for lock | Yes | No, immediate form |
| Return value | None | `boolean` |
| Fallback possible | Not directly | Yes |
| Useful for bounded waiting | No | Use timed overload |
| Typical use | Mandatory critical section | Optional/fallback work |

---

# 4. Practice — Busy Resource ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class BusyResourceExample {

    private static final ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException {

        Thread owner = new Thread(() -> {
            lock.lock();

            try {
                System.out.println("Owner working...");
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                lock.unlock();
            }
        });

        Thread worker = new Thread(() -> {
            if (lock.tryLock()) {
                try {
                    System.out.println("Worker acquired lock");
                } finally {
                    lock.unlock();
                }
            } else {
                System.out.println("Worker: resource busy, using fallback");
            }
        });

        owner.start();
        Thread.sleep(100);
        worker.start();

        owner.join();
        worker.join();
    }
}
```

---

# 5. Timed `tryLock()` ⭐⭐⭐⭐⭐

Use the overloaded version when the thread should wait for a bounded period.

```java
import java.util.concurrent.TimeUnit;

if (lock.tryLock(2, TimeUnit.SECONDS)) {
    try {
        // critical section
    } finally {
        lock.unlock();
    }
} else {
    // timeout
}
```

The method may throw `InterruptedException`, so interruption must be handled.

---

# 6. What Does Timed Locking Mean?

```text
Thread attempts lock
        ↓
Lock available?
   ↙          ↘
 Yes           No
  ↓             ↓
Acquire     Wait up to timeout
                ↓
        ┌───────┴───────┐
        ↓               ↓
    Acquired         Timeout
        ↓               ↓
     true             false
```

### Key Point

The timeout is a maximum waiting bound for acquisition. It is not a guarantee that the thread will execute immediately after acquiring the lock.

---

# 7. Practice — Timed Lock ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class TimedLockExample {

    private static final ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException {

        Thread owner = new Thread(() -> {
            lock.lock();

            try {
                System.out.println("Owner acquired lock");
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                lock.unlock();
            }
        });

        Thread waiter = new Thread(() -> {
            try {
                boolean acquired = lock.tryLock(1, TimeUnit.SECONDS);

                if (acquired) {
                    try {
                        System.out.println("Waiter acquired lock");
                    } finally {
                        lock.unlock();
                    }
                } else {
                    System.out.println("Waiter timed out");
                }

            } catch (InterruptedException e) {
                System.out.println("Waiter interrupted");
                Thread.currentThread().interrupt();
            }
        });

        owner.start();
        Thread.sleep(100);
        waiter.start();

        owner.join();
        waiter.join();
    }
}
```

---

# 8. Timeout vs Immediate Attempt ⭐⭐⭐⭐⭐

### Immediate

```java
lock.tryLock();
```

Means:

> Try now. Do not wait for the lock.

### Timed

```java
lock.tryLock(500, TimeUnit.MILLISECONDS);
```

Means:

> Wait for acquisition for up to the specified duration, unless interrupted.

---

# 9. `tryLock()` With `finally` ⭐⭐⭐⭐⭐

Correct pattern:

```java
if (lock.tryLock()) {
    try {
        updateSharedState();
    } finally {
        lock.unlock();
    }
}
```

### Critical Rule

Only unlock when **this thread successfully acquired the lock**.

Wrong:

```java
if (lock.tryLock()) {
    // work
}

lock.unlock(); // dangerous
```

---

# 10. Common Mistake — Unlocking After Failed `tryLock()` ⚠️

Wrong:

```java
if (!lock.tryLock()) {
    lock.unlock();
}
```

The current thread did not acquire the lock, so it must not release it.

Correct:

```java
if (lock.tryLock()) {
    try {
        // work
    } finally {
        lock.unlock();
    }
}
```

---

# 11. `tryLock()` and Fallback Logic ⭐⭐⭐⭐⭐

A useful pattern is:

```java
if (lock.tryLock()) {
    try {
        return primaryPath();
    } finally {
        lock.unlock();
    }
}

return fallbackPath();
```

### Real-world examples

- Try cache update; otherwise queue the request.
- Try optional reporting; otherwise skip it.
- Try a secondary resource; otherwise use another resource.
- Try a short critical section; otherwise return a temporary response.

---

# 12. Practice — Cache With Fallback ⭐⭐⭐⭐⭐

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReentrantLock;

public class CacheWithFallback {

    private final Map<String, String> cache = new HashMap<>();
    private final ReentrantLock lock = new ReentrantLock();

    public String get(String key) {

        if (lock.tryLock()) {
            try {
                return cache.get(key);
            } finally {
                lock.unlock();
            }
        }

        return "CACHE_BUSY";
    }
}
```

### Design Question

Returning a fallback is only correct if the business requirement allows it. `tryLock()` should not be used just to avoid waiting without defining what happens when acquisition fails.

---

# 13. Timed Locking for Service Protection ⭐⭐⭐⭐⭐

```java
public String process() {

    try {
        if (!lock.tryLock(200, TimeUnit.MILLISECONDS)) {
            return "TEMPORARILY_BUSY";
        }

        try {
            return performWork();
        } finally {
            lock.unlock();
        }

    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return "REQUEST_CANCELLED";
    }
}
```

### Interview Point

Bounded waiting can prevent a request thread from remaining blocked indefinitely, but the timeout/fallback policy must match the application requirements.

---

# 14. `tryLock()` for Deadlock Avoidance ⭐⭐⭐⭐⭐

Suppose two threads need two locks.

Potential deadlock:

```text
Thread A: Lock A → waits for B
Thread B: Lock B → waits for A
```

A timed `tryLock()` can provide an escape path:

```java
if (first.tryLock()) {
    try {
        if (second.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
                // both locks acquired
            } finally {
                second.unlock();
            }
        } else {
            // could not acquire second lock
        }
    } finally {
        first.unlock();
    }
}
```

### Important

`tryLock()` does not magically eliminate deadlocks. The program must release acquired locks and retry/back off or use another recovery strategy.

---

# 15. Practice — Two-Lock Transfer ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class TwoLockTransfer {

    private final ReentrantLock lockA = new ReentrantLock();
    private final ReentrantLock lockB = new ReentrantLock();

    public boolean transfer() throws InterruptedException {

        if (!lockA.tryLock(100, TimeUnit.MILLISECONDS)) {
            return false;
        }

        try {
            if (!lockB.tryLock(100, TimeUnit.MILLISECONDS)) {
                return false;
            }

            try {
                System.out.println("Both locks acquired");
                return true;
            } finally {
                lockB.unlock();
            }

        } finally {
            lockA.unlock();
        }
    }
}
```

### Improvement

In production, if acquisition fails, consider retry with backoff, a consistent global lock order, or another concurrency design rather than blindly retrying in a tight loop.

---

# 16. Timed Locking + Interruption ⭐⭐⭐⭐⭐

Timed acquisition can throw `InterruptedException`.

```java
try {
    if (lock.tryLock(1, TimeUnit.SECONDS)) {
        try {
            // work
        } finally {
            lock.unlock();
        }
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Why restore the flag?

Calling code may need to know that interruption occurred. Restoring the interrupted status preserves that information after catching the exception.

---

# 17. `tryLock()` and Reentrancy ⭐⭐⭐⭐⭐

If the current thread already owns the lock, `tryLock()` can succeed again because the lock is reentrant.

```java
lock.lock();

try {
    boolean acquiredAgain = lock.tryLock();
    System.out.println(acquiredAgain); // true

    try {
        System.out.println(lock.getHoldCount()); // 2
    } finally {
        lock.unlock();
    }

} finally {
    lock.unlock();
}
```

### Key Point

The immediate `tryLock()` does not mean "only acquire if nobody currently holds the lock". The owning thread can reacquire its own `ReentrantLock`.

---

# 18. Fair Lock + `tryLock()` ⭐⭐⭐⭐

A subtle interview point:

```java
ReentrantLock fairLock = new ReentrantLock(true);
```

The untimed `tryLock()` can acquire the lock immediately if it is available, even when other threads are queued. Therefore, do not interpret fairness as a strict FIFO guarantee for every acquisition API.

For designs where fairness policy matters, understand the exact acquisition method being used.

---

# 19. Practice — Immediate vs Timed Acquisition ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class CompareLockAcquisition {

    private static final ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) throws Exception {

        lock.lock();

        try {
            System.out.println("Lock held by main thread");

            System.out.println("Immediate tryLock: "
                    + lock.tryLock());

            // Reentrant: main thread can acquire again.
            System.out.println("Timed tryLock: "
                    + lock.tryLock(100, TimeUnit.MILLISECONDS));

            lock.unlock();
            lock.unlock();

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

Expected concept:

```text
Immediate tryLock → true
Timed tryLock     → true
```

because the same thread already owns the reentrant lock.

---

# 20. `tryLock()` vs `lockInterruptibly()` ⭐⭐⭐⭐⭐

| Feature | `tryLock()` | `tryLock(timeout)` | `lockInterruptibly()` |
|---|---|---|---|
| Immediate | Yes | No | No |
| Bounded wait | No | Yes | No |
| Can throw `InterruptedException` | No | Yes | Yes |
| Returns boolean | Yes | Yes | No |
| Main use | Optional acquisition | Bounded acquisition | Cancellable waiting |

---

# 21. Avoid Tight Retry Loops ⚠️

Bad:

```java
while (!lock.tryLock()) {
    // spin continuously
}
```

This can consume CPU and create contention.

If retrying is required, consider a bounded retry strategy with backoff, or redesign the synchronization boundary.

Example:

```java
while (!lock.tryLock()) {
    Thread.sleep(10);
}
```

This is only a teaching example; production retry policies should be designed according to workload and latency requirements.

---

# 22. Practice — Retry With Backoff ⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class RetryWithBackoff {

    private final ReentrantLock lock = new ReentrantLock();

    public void process() throws InterruptedException {

        int attempts = 0;

        while (attempts < 3) {

            if (lock.tryLock()) {
                try {
                    System.out.println("Work completed");
                    return;
                } finally {
                    lock.unlock();
                }
            }

            attempts++;
            Thread.sleep(50L * attempts);
        }

        System.out.println("Could not acquire lock after retries");
    }
}
```

---

# 23. Critical Section Should Stay Small ⭐⭐⭐⭐⭐

Timed locking does not make a large critical section efficient.

Avoid:

```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        callRemoteAPI();
        readFile();
        performDatabaseCall();
        updateSharedState();
    } finally {
        lock.unlock();
    }
}
```

Prefer to perform non-shared work outside the lock when correctness allows:

```java
Response response = callRemoteAPI();

lock.lock();
try {
    updateSharedState(response);
} finally {
    lock.unlock();
}
```

The exact boundary depends on what must be protected atomically.

---

# 24. Timed Lock Does Not Cancel the Work ⭐⭐⭐⭐⭐

If acquisition times out:

```java
if (!lock.tryLock(200, TimeUnit.MILLISECONDS)) {
    return;
}
```

only the **lock acquisition attempt** has ended.

It does not cancel work already running elsewhere, and it does not interrupt the current thread's arbitrary operations.

---

# 25. Timed Lock Does Not Guarantee Success ⭐⭐⭐⭐⭐

```java
boolean acquired = lock.tryLock(1, TimeUnit.SECONDS);
```

Possible results:

```text
true  → lock acquired by current thread
false → timeout expired before acquisition
```

The program must correctly handle both paths.

---

# 26. Production Scenario — Inventory Update ⭐⭐⭐⭐⭐

```java
public boolean reserveStock() {

    try {
        if (!lock.tryLock(250, TimeUnit.MILLISECONDS)) {
            return false;
        }

        try {
            if (stock <= 0) {
                return false;
            }

            stock--;
            return true;

        } finally {
            lock.unlock();
        }

    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return false;
    }
}
```

### Interview Discussion

The protected invariant is:

```text
stock cannot become negative
```

The check and decrement must occur inside the same critical section.

---

# 27. Production Scenario — Request Throttling

Suppose a shared resource should not be occupied indefinitely.

```java
if (!lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    throw new IllegalStateException("Resource is busy");
}

try {
    processRequest();
} finally {
    lock.unlock();
}
```

This can provide bounded waiting, but application-level timeout handling should be designed separately from the lock itself.

---

# 28. Common Interview Trap ⭐⭐⭐⭐⭐

### Question

Does `tryLock(5, TimeUnit.SECONDS)` guarantee the thread waits exactly five seconds?

### Answer

No.

It waits **up to** the specified time while attempting acquisition. It may acquire earlier, time out, or be interrupted.

---

# 29. Common Interview Trap ⭐⭐⭐⭐⭐

### Question

If `tryLock()` returns `false`, should we call `unlock()`?

### Answer

No.

The current thread did not acquire the lock and therefore must not unlock it.

---

# 30. Common Interview Trap ⭐⭐⭐⭐⭐

### Question

Can `tryLock()` return `true` when the lock is already held?

### Answer

Yes, if the **current thread** already owns a `ReentrantLock`, because the lock is reentrant.

It will not return `true` merely because another thread owns the lock.

---

# 31. Common Interview Trap ⭐⭐⭐⭐⭐

### Question

Does timed locking prevent deadlocks automatically?

### Answer

No.

It can help detect failed acquisition and provide a recovery path, but the application still needs correct lock ordering, rollback/retry logic, or another deadlock-avoidance strategy.

---

# 32. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is `tryLock()`?

> It attempts to acquire a `ReentrantLock` without waiting in the no-argument form and returns a boolean indicating success.

### Q2. What does timed `tryLock()` do?

> It waits for acquisition for up to a specified duration and returns `true` if acquired, otherwise `false`; it can throw `InterruptedException` while waiting.

### Q3. Why use `tryLock()`?

> To avoid indefinite blocking and enable fallback, timeout, retry, or alternate-resource strategies.

### Q4. What happens if `tryLock()` fails?

> The current thread does not own the lock and should not call `unlock()`.

### Q5. Can the owner call `tryLock()` again?

> Yes. `ReentrantLock` is reentrant, so the owning thread can acquire it again and increase its hold count.

### Q6. Does `tryLock()` prevent deadlock?

> Not automatically. It can support deadlock-avoidance strategies when failed acquisition causes the thread to release acquired locks and retry or back off.

### Q7. What happens if a timed `tryLock()` thread is interrupted?

> It throws `InterruptedException`. A common response is to restore the interrupt status with `Thread.currentThread().interrupt()` after handling it.

### Q8. `tryLock()` vs `lockInterruptibly()`?

> `tryLock()` provides immediate or bounded acquisition and a boolean result; `lockInterruptibly()` waits until acquisition but allows the waiting operation to be interrupted.

### Q9. Does a timeout cancel another thread's work?

> No. It only bounds the current thread's lock-acquisition wait.

### Q10. Why should the critical section be small?

> To reduce contention, improve throughput, reduce latency, and lower the chance of lock-related bottlenecks.

### Q11. Is `tryLock()` FIFO?

> The untimed form can acquire an available lock immediately and should not be treated as a strict FIFO mechanism, even if the lock was constructed as fair.

### Q12. What should happen after timeout?

> It depends on the business requirement: fallback, retry with backoff, queue the operation, return a temporary error, or use another resource.

---

# 33. 2-Minute Interview Answer 🏆

> **"`tryLock()` is an acquisition method provided by `ReentrantLock` that lets a thread attempt to acquire a lock without waiting indefinitely. The no-argument form returns immediately with `true` or `false`, while the timed overload can wait up to a specified duration. This is useful when we need bounded waiting, fallback behavior, or a strategy for avoiding indefinite lock contention. If acquisition succeeds, I always release the lock in a `finally` block. If `tryLock()` returns false, I must not call `unlock()` because the current thread does not own the lock. Timed acquisition can throw `InterruptedException`, which I normally handle by restoring the interrupt status. `tryLock()` can support deadlock-avoidance strategies, but it does not automatically prevent deadlocks. I would use it when the application has a meaningful policy for what to do if the lock cannot be acquired within the required time."**

---

# 34. Quick Revision ⭐⭐⭐⭐⭐

```text
tryLock()
    ↓
Immediate attempt
    ↓
true / false
```

```text
tryLock(timeout)
    ↓
Bounded waiting
    ↓
true / false
    ↓
InterruptedException possible
```

### Golden Pattern

```java
if (lock.tryLock()) {
    try {
        // critical section
    } finally {
        lock.unlock();
    }
}
```

### Timed Pattern

```java
try {
    if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
        try {
            // critical section
        } finally {
            lock.unlock();
        }
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Remember

```text
false → DON'T unlock
true  → unlock in finally
```

---

# 💻 Practice Checklist

- [ ] Basic `tryLock()`
- [ ] `lock()` vs `tryLock()`
- [ ] Timed `tryLock()`
- [ ] Handle `InterruptedException`
- [ ] Correct `finally` unlock pattern
- [ ] Failed acquisition must not unlock
- [ ] Fallback strategy
- [ ] Timed service protection
- [ ] Two-lock acquisition
- [ ] Deadlock-avoidance strategy
- [ ] Retry with backoff
- [ ] Reentrant `tryLock()` behavior
- [ ] Fair lock nuance
- [ ] Keep critical sections small
- [ ] Understand timeout semantics
- [ ] Inventory/reservation example
- [ ] Answer interview traps
- [ ] Explain topic in 2 minutes

---

## Navigation

[← 8.16 — `ReentrantLock`](../16-ReentrantLock/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.18 — `ReentrantReadWriteLock`**