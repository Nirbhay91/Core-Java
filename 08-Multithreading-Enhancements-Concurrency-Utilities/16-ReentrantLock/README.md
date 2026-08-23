# 8.16 — `ReentrantLock`

> **Goal:** Understand `ReentrantLock` as an explicit locking mechanism that provides more control than Java's intrinsic `synchronized` monitor.

---

## 1. What is `ReentrantLock`? ⭐⭐⭐⭐⭐

`ReentrantLock` is a class from `java.util.concurrent.locks` that provides explicit mutual exclusion.

```java
import java.util.concurrent.locks.ReentrantLock;

ReentrantLock lock = new ReentrantLock();
```

A thread acquires the lock with:

```java
lock.lock();
```

and releases it with:

```java
lock.unlock();
```

### Memory Trick

> **`synchronized` = implicit lock management**  
> **`ReentrantLock` = explicit lock management + extra control**

---

# 2. Why `ReentrantLock`? ⭐⭐⭐⭐⭐

Use it when you need features that are not directly available with a basic `synchronized` block/method, such as:

- `tryLock()`
- timed lock acquisition
- interruptible lock acquisition
- multiple `Condition` objects
- explicit lock/unlock control
- optional fairness policy
- lock state inspection APIs

---

# 3. Basic Example ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class BasicReentrantLockExample {

    private static final ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) {

        Thread t1 = new Thread(() -> increment(), "Thread-1");
        Thread t2 = new Thread(() -> increment(), "Thread-2");

        t1.start();
        t2.start();
    }

    private static void increment() {
        lock.lock();

        try {
            System.out.println(Thread.currentThread().getName()
                    + " acquired lock");
        } finally {
            lock.unlock();
        }
    }
}
```

### Golden Rule ⭐⭐⭐⭐⭐

Always prefer:

```java
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```

Never rely on normal control flow to reach `unlock()`.

---

# 4. Why `finally` is Mandatory ⭐⭐⭐⭐⭐

Wrong:

```java
lock.lock();
doSomething();
lock.unlock();
```

If `doSomething()` throws an exception, `unlock()` may never execute.

Correct:

```java
lock.lock();
try {
    doSomething();
} finally {
    lock.unlock();
}
```

### Interview Point

> Failure to unlock a `ReentrantLock` can leave other threads blocked indefinitely.

---

# 5. Reentrancy ⭐⭐⭐⭐⭐

The same thread can acquire the same `ReentrantLock` multiple times without deadlocking itself.

```java
import java.util.concurrent.locks.ReentrantLock;

public class ReentrancyExample {

    private final ReentrantLock lock = new ReentrantLock();

    public void methodA() {
        lock.lock();

        try {
            System.out.println("methodA acquired lock");
            methodB();
        } finally {
            lock.unlock();
        }
    }

    private void methodB() {
        lock.lock();

        try {
            System.out.println("methodB acquired lock again");
        } finally {
            lock.unlock();
        }
    }
}
```

### What happens?

```text
Thread acquires lock → hold count = 1
        ↓
methodB acquires same lock → hold count = 2
        ↓
methodB unlocks → hold count = 1
        ↓
methodA unlocks → hold count = 0
```

The lock is actually released when the owning thread has balanced all acquisitions with unlocks.

---

# 6. `getHoldCount()` ⭐⭐⭐⭐

Returns the number of holds on the lock by the current thread.

```java
lock.lock();

try {
    System.out.println(lock.getHoldCount());
} finally {
    lock.unlock();
}
```

Output:

```text
1
```

After a nested acquisition:

```text
2
```

### Important

This is mainly useful for diagnostics/debugging, not normal business logic.

---

# 7. `isHeldByCurrentThread()` ⭐⭐⭐⭐

Checks whether the current thread owns the lock.

```java
if (lock.isHeldByCurrentThread()) {
    System.out.println("Current thread owns the lock");
}
```

Again, this is primarily a diagnostic/advanced-use API.

---

# 8. `tryLock()` ⭐⭐⭐⭐⭐

Attempts to acquire the lock without waiting indefinitely.

```java
if (lock.tryLock()) {
    try {
        // critical section
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("Could not acquire lock");
}
```

### Difference

```text
lock.lock()
→ wait until acquired

lock.tryLock()
→ attempt immediately; return true/false
```

The next topic, **8.17**, will cover `tryLock()` and timed locking in depth.

---

# 9. Timed `tryLock()` ⭐⭐⭐⭐⭐

You can wait for a limited amount of time.

```java
import java.util.concurrent.TimeUnit;

if (lock.tryLock(2, TimeUnit.SECONDS)) {
    try {
        // critical section
    } finally {
        lock.unlock();
    }
}
```

This can be useful when indefinite waiting is undesirable.

---

# 10. `lockInterruptibly()` ⭐⭐⭐⭐⭐

Allows a waiting thread to respond to interruption while trying to acquire the lock.

```java
try {
    lock.lockInterruptibly();

    try {
        // critical section
    } finally {
        lock.unlock();
    }

} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Difference

```text
lock()
→ waiting for acquisition is not interruptible through InterruptedException

lockInterruptibly()
→ waiting can be interrupted
```

---

# 11. Fair vs Non-Fair Lock ⭐⭐⭐⭐⭐

By default:

```java
ReentrantLock lock = new ReentrantLock();
```

creates a non-fair lock.

A fair lock can be requested with:

```java
ReentrantLock lock = new ReentrantLock(true);
```

### Concept

```text
Non-fair
→ a thread may acquire the lock even if another thread has been waiting longer

Fair
→ attempts to honor waiting order more closely
```

### Important

Fairness generally has a throughput cost. It does **not** mean that every scheduling outcome is strictly FIFO.

---

# 12. Practice — Fair vs Non-Fair ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class FairLockExample {

    private static final ReentrantLock lock = new ReentrantLock(true);

    public static void main(String[] args) {

        for (int i = 1; i <= 5; i++) {
            int id = i;

            new Thread(() -> {
                lock.lock();

                try {
                    System.out.println("Worker " + id
                            + " acquired lock");
                } finally {
                    lock.unlock();
                }
            }).start();
        }
    }
}
```

Do not treat the printed order as a guaranteed demonstration of strict FIFO scheduling.

---

# 13. `ReentrantLock` vs `synchronized` ⭐⭐⭐⭐⭐

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Mutual exclusion | Yes | Yes |
| Reentrant | Yes | Yes |
| Explicit lock/unlock | No | Yes |
| `tryLock()` | No | Yes |
| Timed acquisition | No direct equivalent | Yes |
| Interruptible acquisition | No direct equivalent | Yes |
| Multiple `Condition`s | No | Yes |
| Fairness option | No | Yes |
| Automatic unlock | Yes | No |
| Simpler syntax | Yes | No |

### Interview Answer

> Use `synchronized` when simple mutual exclusion is enough. Use `ReentrantLock` when you need advanced acquisition, fairness, interruption, timeout, or condition-based coordination.

---

# 14. Practice — Thread-Safe Counter ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class LockCounter {

    private int count;
    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {
        lock.lock();

        try {
            count++;
        } finally {
            lock.unlock();
        }
    }

    public int getCount() {
        lock.lock();

        try {
            return count;
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {

        LockCounter counter = new LockCounter();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                counter.increment();
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                counter.increment();
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(counter.getCount());
    }
}
```

Expected:

```text
200000
```

---

# 15. Practice — Bank Account ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class BankAccount {

    private double balance;
    private final ReentrantLock lock = new ReentrantLock();

    public BankAccount(double balance) {
        this.balance = balance;
    }

    public boolean withdraw(double amount) {
        lock.lock();

        try {
            if (amount > balance) {
                return false;
            }

            balance -= amount;
            return true;

        } finally {
            lock.unlock();
        }
    }

    public void deposit(double amount) {
        lock.lock();

        try {
            balance += amount;
        } finally {
            lock.unlock();
        }
    }

    public double getBalance() {
        lock.lock();

        try {
            return balance;
        } finally {
            lock.unlock();
        }
    }
}
```

### Interview Point

The lock protects the invariant:

```text
balance must be checked and modified as one atomic critical section
```

---

# 16. Practice — Nested Reentrant Calls ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class NestedLockExample {

    private final ReentrantLock lock = new ReentrantLock();

    void outer() {
        lock.lock();

        try {
            System.out.println("outer hold count: " + lock.getHoldCount());
            inner();
        } finally {
            lock.unlock();
        }
    }

    void inner() {
        lock.lock();

        try {
            System.out.println("inner hold count: " + lock.getHoldCount());
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        new NestedLockExample().outer();
    }
}
```

Expected concept:

```text
outer hold count: 1
inner hold count: 2
```

---

# 17. Practice — Avoid Waiting Forever ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class TryLockExample {

    private static final ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException {

        Thread owner = new Thread(() -> {
            lock.lock();

            try {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            } finally {
                lock.unlock();
            }
        });

        Thread contender = new Thread(() -> {
            if (lock.tryLock()) {
                try {
                    System.out.println("Contender acquired lock");
                } finally {
                    lock.unlock();
                }
            } else {
                System.out.println("Contender could not acquire lock");
            }
        });

        owner.start();
        Thread.sleep(100);
        contender.start();

        owner.join();
        contender.join();
    }
}
```

---

# 18. Practice — Interruptible Lock Acquisition ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class InterruptibleLockExample {

    private static final ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) throws Exception {

        Thread owner = new Thread(() -> {
            lock.lock();

            try {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            } finally {
                lock.unlock();
            }
        });

        Thread waiter = new Thread(() -> {
            try {
                lock.lockInterruptibly();

                try {
                    System.out.println("Waiter acquired lock");
                } finally {
                    lock.unlock();
                }

            } catch (InterruptedException e) {
                System.out.println("Waiter was interrupted while waiting");
                Thread.currentThread().interrupt();
            }
        });

        owner.start();
        Thread.sleep(100);
        waiter.start();

        Thread.sleep(500);
        waiter.interrupt();

        owner.join();
        waiter.join();
    }
}
```

---

# 19. Lock Ownership ⭐⭐⭐⭐⭐

A `ReentrantLock` has an owning thread when locked.

Useful diagnostic APIs include:

```java
lock.isLocked();
lock.isHeldByCurrentThread();
lock.getHoldCount();
```

### Important

`isLocked()` only tells you whether some thread currently holds the lock. It does not tell you that the current thread owns it.

For ownership of the current thread:

```java
lock.isHeldByCurrentThread();
```

---

# 20. `getQueueLength()` ⭐⭐⭐⭐

Provides an estimate of the number of threads waiting to acquire the lock.

```java
int waiting = lock.getQueueLength();
```

### Important

This is an **estimate**, not a synchronization mechanism or exact snapshot for business decisions.

---

# 21. `hasQueuedThreads()` ⭐⭐⭐⭐

Checks whether there may be threads waiting to acquire the lock.

```java
if (lock.hasQueuedThreads()) {
    System.out.println("Threads are waiting");
}
```

Again, treat this as diagnostic information rather than a correctness mechanism.

---

# 22. Locking a Shared Resource ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class SharedResource {

    private final ReentrantLock lock = new ReentrantLock();

    public void access() {
        lock.lock();

        try {
            System.out.println(Thread.currentThread().getName()
                    + " accessing resource");
        } finally {
            lock.unlock();
        }
    }
}
```

The important design rule is:

> The same lock must guard every access that participates in the protected invariant.

---

# 23. One Lock Per Object ⭐⭐⭐⭐⭐

```java
class Account {

    private final ReentrantLock lock = new ReentrantLock();
    private int balance;
}
```

Different `Account` instances have different locks.

```text
Account A → Lock A
Account B → Lock B
```

Therefore, operations on A do not unnecessarily block operations on B.

This can improve concurrency compared with one global lock.

---

# 24. Avoid Locking on `this` When API Control Matters ⭐⭐⭐⭐

With `synchronized`:

```java
synchronized (this) {
    // critical section
}
```

Other code that can obtain the same object reference can potentially synchronize on it too.

With an explicit private lock:

```java
private final ReentrantLock lock = new ReentrantLock();
```

you control the synchronization mechanism more directly.

### Design Point

A private lock object can reduce accidental coupling between your class's synchronization and external code.

---

# 25. Common Mistake — Unlocking Without Ownership ⚠️

Wrong:

```java
if (lock.isLocked()) {
    lock.unlock();
}
```

This is dangerous because `isLocked()` does not mean **this thread owns the lock**.

Correct design:

```java
lock.lock();
try {
    // work
} finally {
    lock.unlock();
}
```

Or, for diagnostics:

```java
if (lock.isHeldByCurrentThread()) {
    lock.unlock();
}
```

But the preferred pattern is still structured acquisition/release.

---

# 26. Common Mistake — Unlocking in the Wrong Method ⚠️

Because the lock is reentrant, nested methods may acquire it.

Every acquisition must be paired with an unlock by the owning thread.

```text
lock()        → hold = 1
lock()        → hold = 2
unlock()      → hold = 1
unlock()      → hold = 0
```

Missing one unlock can leave the lock held indefinitely.

---

# 27. Common Mistake — Holding a Lock During Slow I/O ⚠️

Avoid unnecessarily doing this:

```java
lock.lock();
try {
    callRemoteService();
    readLargeFile();
    updateDatabase();
} finally {
    lock.unlock();
}
```

If only a small section requires synchronization, keep the critical section narrow.

```java
callRemoteService();

lock.lock();
try {
    updateSharedState();
} finally {
    lock.unlock();
}
```

The correct boundary depends on the invariant being protected.

---

# 28. Deadlock Risk ⭐⭐⭐⭐⭐

`ReentrantLock` does not automatically prevent deadlocks.

Example:

```text
Thread A:
Lock A → waiting for Lock B

Thread B:
Lock B → waiting for Lock A
```

### Prevention

Use a consistent lock acquisition order:

```text
Always acquire Lock A before Lock B
```

Or use timed `tryLock()` where appropriate and design recovery logic.

---

# 29. `ReentrantLock` Does Not Mean Fair Lock ⭐⭐⭐⭐⭐

The word **reentrant** means:

> The owning thread can acquire the same lock again.

It does **not** mean:

> Every waiting thread gets the lock in FIFO order.

Fairness is a separate configuration:

```java
new ReentrantLock(true);
```

---

# 30. `ReentrantLock` and Memory Visibility ⭐⭐⭐⭐⭐

A successful unlock followed by a subsequent successful lock by another thread provides the synchronization relationship needed for visibility of writes protected by that lock.

For interview purposes:

```text
Thread A
lock → write shared state → unlock
                         ↓
Thread B
lock → can observe synchronized state
```

But both threads must consistently use the same lock for the protected state.

---

# 31. Production Pattern — Cache Update ⭐⭐⭐⭐⭐

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReentrantLock;

public class SafeCache {

    private final Map<String, String> cache = new HashMap<>();
    private final ReentrantLock lock = new ReentrantLock();

    public void put(String key, String value) {
        lock.lock();

        try {
            cache.put(key, value);
        } finally {
            lock.unlock();
        }
    }

    public String get(String key) {
        lock.lock();

        try {
            return cache.get(key);
        } finally {
            lock.unlock();
        }
    }
}
```

### Interview Point

`HashMap` itself is not made thread-safe merely because one method uses a lock. All relevant concurrent access must follow the same locking protocol.

---

# 32. Practice — Two Resources With Consistent Lock Ordering ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class TransferService {

    private final ReentrantLock firstLock = new ReentrantLock();
    private final ReentrantLock secondLock = new ReentrantLock();

    public void transfer() {

        firstLock.lock();

        try {
            secondLock.lock();

            try {
                System.out.println("Transfer executed");
            } finally {
                secondLock.unlock();
            }

        } finally {
            firstLock.unlock();
        }
    }
}
```

### Key idea

All callers must follow the same order:

```text
firstLock → secondLock
```

This avoids the classic circular lock-order deadlock pattern.

---

# 33. `isLocked()` vs `isHeldByCurrentThread()` ⭐⭐⭐⭐⭐

| API | Meaning |
|---|---|
| `isLocked()` | Is the lock currently held by any thread? |
| `isHeldByCurrentThread()` | Does the current thread own the lock? |
| `getHoldCount()` | How many times has current thread acquired it? |
| `getQueueLength()` | Estimated number of waiting threads |

---

# 34. Practice — Lock Diagnostics ⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class LockDiagnosticsExample {

    public static void main(String[] args) {

        ReentrantLock lock = new ReentrantLock();

        lock.lock();

        try {
            System.out.println("isLocked = " + lock.isLocked());
            System.out.println("isHeldByCurrentThread = "
                    + lock.isHeldByCurrentThread());
            System.out.println("holdCount = " + lock.getHoldCount());
            System.out.println("queueLength = " + lock.getQueueLength());
        } finally {
            lock.unlock();
        }
    }
}
```

---

# 35. When Should You Prefer `synchronized`? ⭐⭐⭐⭐⭐

Prefer `synchronized` when:

- Mutual exclusion is all you need.
- The locking scope is straightforward.
- You want automatic release when leaving the block/method.
- You do not need timed/interruptible acquisition or conditions.
- Simplicity is more valuable than explicit lock control.

### Example

```java
public synchronized void increment() {
    count++;
}
```

Do not use `ReentrantLock` merely because it looks more advanced.

---

# 36. When Should You Prefer `ReentrantLock`? ⭐⭐⭐⭐⭐

Prefer it when requirements include:

```text
tryLock()
Timed acquisition
Interruptible acquisition
Fairness configuration
Multiple Condition objects
Explicit lock lifecycle
Lock diagnostics
```

### Interview Memory

> **Need simple locking → `synchronized`.**  
> **Need advanced lock control → `ReentrantLock`.**

---

# 37. Important Relationship With `Condition` ⭐⭐⭐⭐⭐

A `ReentrantLock` can create one or more condition variables:

```java
Condition condition = lock.newCondition();
```

This enables patterns such as:

```text
Lock
 ├── Condition: notEmpty
 └── Condition: notFull
```

This is one major advantage over intrinsic monitor methods when designing more complex producer-consumer coordination.

`Condition` will be covered separately in **8.20**.

---

# 38. Interview Scenario — Why Not Just `synchronized`? ⭐⭐⭐⭐⭐

### Question

You need to attempt a lock for 500 milliseconds and if it cannot be acquired, return a fallback response. What would you use?

### Answer

```java
if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
    try {
        return doWork();
    } finally {
        lock.unlock();
    }
}

return fallback();
```

This is a good case for `ReentrantLock` because `synchronized` does not provide an equivalent timed lock-acquisition API.

---

# 39. Interview Scenario — Cancel a Waiting Thread ⭐⭐⭐⭐⭐

### Requirement

A thread waiting for a lock must be cancellable.

Use:

```java
lock.lockInterruptibly();
```

Then handle:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Key Point

Do not swallow interruption silently.

---

# 40. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is `ReentrantLock`?

> It is an explicit mutual-exclusion lock implementation from `java.util.concurrent.locks` that supports reentrancy and advanced lock-acquisition features.

### Q2. Why is it called reentrant?

> Because the thread that owns the lock can acquire it again without blocking itself.

### Q3. What happens if the same thread locks twice?

> The hold count increases. The thread must unlock the same number of times before the lock is fully released.

### Q4. Why use `finally`?

> To guarantee that the lock is released even if the protected code throws an exception.

### Q5. `lock()` vs `tryLock()`?

> `lock()` waits until the lock is acquired. `tryLock()` attempts acquisition and returns immediately with success/failure; the timed overload can wait for a bounded duration.

### Q6. What is `lockInterruptibly()`?

> It allows a thread waiting to acquire the lock to respond to interruption by throwing `InterruptedException`.

### Q7. Is `ReentrantLock` fair by default?

> No. The default constructor creates a non-fair lock. A fair lock can be requested with `new ReentrantLock(true)`.

### Q8. Does fair locking guarantee strict FIFO execution?

> No. It provides a fairness policy that generally favors the longest-waiting eligible thread, but it is not a general guarantee of strict execution order.

### Q9. `ReentrantLock` vs `synchronized`?

> Both provide mutual exclusion and reentrancy. `ReentrantLock` provides explicit control such as timed/interruptible acquisition, `tryLock()`, fairness and multiple conditions, while `synchronized` is simpler and automatically releases the monitor when the block exits.

### Q10. Can `ReentrantLock` cause deadlock?

> Yes. Explicit locks can deadlock if multiple threads acquire multiple locks in inconsistent order.

### Q11. What is `getHoldCount()`?

> It returns the number of holds on the lock by the current thread.

### Q12. What is `isHeldByCurrentThread()`?

> It checks whether the current thread currently owns the lock.

### Q13. What is `getQueueLength()`?

> It provides an estimate of the number of threads waiting to acquire the lock.

### Q14. Does `ReentrantLock` automatically make a collection thread-safe?

> No. The lock only provides coordination when all relevant accesses follow the same locking protocol.

### Q15. When should you avoid `ReentrantLock`?

> When simple mutual exclusion is enough and the extra control is unnecessary; `synchronized` is often clearer.

---

# 41. Quick Revision ⭐⭐⭐⭐⭐

```text
ReentrantLock
     ↓
Explicit lock
     ↓
lock()
     ↓
try/finally
     ↓
unlock()
```

### Advanced APIs

```text
tryLock()
tryLock(timeout)
lockInterruptibly()
new ReentrantLock(true)
newCondition()
```

### Reentrancy

```text
lock()
  ↓
hold = 1
  ↓
lock()
  ↓
hold = 2
  ↓
unlock()
  ↓
hold = 1
  ↓
unlock()
  ↓
hold = 0
```

### Core Interview Comparison

```text
synchronized
→ simple
→ automatic unlock
→ monitor-based

ReentrantLock
→ explicit
→ tryLock
→ timed acquisition
→ interruptible acquisition
→ fairness option
→ multiple Conditions
```

---

# 🏆 2-Minute Interview Answer

> **"`ReentrantLock` is an explicit mutual-exclusion lock from `java.util.concurrent.locks`. It is called reentrant because the thread that already owns the lock can acquire it again, with a hold count tracking nested acquisitions. Unlike `synchronized`, it gives us additional control through `tryLock()`, timed acquisition, `lockInterruptibly()`, configurable fairness and `Condition` objects. The standard pattern is to call `lock()` and put the critical section inside a `try` block, with `unlock()` in `finally` so the lock is released even when an exception occurs. `ReentrantLock` does not automatically prevent deadlocks, so multiple locks should be acquired in a consistent order. I would use `synchronized` when simple mutual exclusion is sufficient and `ReentrantLock` when advanced lock-control features are actually required."**

---

# 💻 Practice Checklist

- [ ] Create a `ReentrantLock`.
- [ ] Use `lock()` / `unlock()`.
- [ ] Always release in `finally`.
- [ ] Practice reentrancy.
- [ ] Observe `getHoldCount()`.
- [ ] Use `isHeldByCurrentThread()`.
- [ ] Practice `tryLock()`.
- [ ] Practice timed `tryLock()`.
- [ ] Practice `lockInterruptibly()`.
- [ ] Compare fair and non-fair locks.
- [ ] Build a thread-safe counter.
- [ ] Build a thread-safe bank account.
- [ ] Protect a shared `HashMap` with one lock.
- [ ] Practice consistent lock ordering.
- [ ] Create a deadlock scenario and identify it.
- [ ] Practice lock diagnostics.
- [ ] Understand `Condition` as the next step.
- [ ] Compare `ReentrantLock` with `synchronized`.
- [ ] Explain reentrancy in under 2 minutes.

---

## Navigation

[← 8.15 — `Phaser`](../15-Phaser/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.17 — `tryLock()` and Timed Locking**