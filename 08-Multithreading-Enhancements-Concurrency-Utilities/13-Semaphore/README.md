# 8.13 — `Semaphore`

> **Goal:** Understand how `Semaphore` controls concurrent access to a limited resource using permits.

---

## 1. What is `Semaphore`? ⭐⭐⭐⭐⭐

`Semaphore` is a synchronization utility from `java.util.concurrent` that maintains a set of **permits**.

A thread must acquire a permit before entering a protected section and should release it when finished.

```text
             Semaphore(3)
             3 permits
          /      |      \
       T1       T2       T3
       ↓        ↓        ↓
    resource resource resource

T4 → acquire() → waits

When T1 releases:
T4 → gets permit → continues
```

### Memory Trick

> **Semaphore = control how many threads can access something at the same time.**

---

# 2. Why `Semaphore`? ⭐⭐⭐⭐⭐

Use it when a resource has a **limited capacity**.

Examples:

- Database connection pool
- Limited external API concurrency
- File/resource access
- Worker slots
- Parking spaces
- Connection throttling
- Rate/concurrency limiting inside a JVM

---

# 3. Basic Example ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Semaphore;

public class BasicSemaphoreExample {

    public static void main(String[] args) {

        Semaphore semaphore = new Semaphore(2);

        for (int i = 1; i <= 5; i++) {
            int id = i;

            new Thread(() -> {
                try {
                    semaphore.acquire();

                    System.out.println("Thread " + id + " acquired permit");
                    Thread.sleep(2000);

                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    semaphore.release();
                    System.out.println("Thread " + id + " released permit");
                }
            }).start();
        }
    }
}
```

At most **2 threads** can hold permits simultaneously.

---

# 4. Constructor ⭐⭐⭐⭐

```java
Semaphore semaphore = new Semaphore(3);
```

This creates a semaphore with 3 permits.

There is also a fairness option:

```java
Semaphore semaphore = new Semaphore(3, true);
```

`true` requests **fair ordering** for threads acquiring permits.

### Important

Fairness does not mean absolute real-time ordering guarantees. It affects the semaphore's acquisition policy.

---

# 5. `acquire()` ⭐⭐⭐⭐⭐

```java
semaphore.acquire();
```

If a permit is available, the thread acquires it.

If no permit is available, the thread waits.

`acquire()` is interruptible:

```java
try {
    semaphore.acquire();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

---

# 6. `release()` ⭐⭐⭐⭐⭐

Return a permit:

```java
semaphore.release();
```

The most important pattern is:

```java
semaphore.acquire();
try {
    // protected resource
} finally {
    semaphore.release();
}
```

### Why `finally`?

If the protected operation throws an exception and the permit is not released, future threads can remain blocked unnecessarily.

---

# 7. `tryAcquire()` ⭐⭐⭐⭐⭐

Non-blocking attempt:

```java
if (semaphore.tryAcquire()) {
    try {
        // use resource
    } finally {
        semaphore.release();
    }
} else {
    System.out.println("No permit available");
}
```

The thread does not wait indefinitely for a permit.

---

# 8. Timed `tryAcquire()` ⭐⭐⭐⭐⭐

```java
if (semaphore.tryAcquire(2, TimeUnit.SECONDS)) {
    try {
        // use resource
    } finally {
        semaphore.release();
    }
} else {
    System.out.println("Could not acquire permit in time");
}
```

This is useful when waiting forever is undesirable.

---

# 9. `acquire(int permits)` ⭐⭐⭐⭐

A thread can acquire multiple permits:

```java
semaphore.acquire(2);
```

It must then release the corresponding number:

```java
semaphore.release(2);
```

### Rule

> Keep acquisition and release counts balanced unless an intentional design explicitly transfers permits.

---

# 10. `release(int permits)` ⭐⭐⭐⭐

```java
semaphore.release(3);
```

This releases three permits.

Unlike some lock APIs, `Semaphore` does not automatically track that the releasing thread was the thread that acquired the permit.

This is an important conceptual difference.

---

# 11. Semaphore Does Not Have Ownership ⭐⭐⭐⭐⭐

A semaphore permit is not tied to a specific thread.

For example:

```java
Semaphore semaphore = new Semaphore(1);

Thread A → acquire()
Thread B → release()
```

This is allowed by `Semaphore`.

### Compare with `Lock`

A `ReentrantLock` has ownership semantics: the thread that locks it is expected to unlock it.

### Interview Point ⭐

> `Semaphore` controls permits, not thread ownership.

---

# 12. Counting Semaphore vs Binary Semaphore ⭐⭐⭐⭐⭐

### Counting semaphore

```java
new Semaphore(5);
```

Allows up to 5 permits concurrently.

### Binary semaphore

```java
new Semaphore(1);
```

Has only one permit.

It can be used to limit access to one concurrent participant, but it is not identical to an ownership-based lock.

---

# 13. Fair vs Non-Fair Semaphore ⭐⭐⭐⭐

### Non-fair

```java
new Semaphore(3);
```

Default behavior.

### Fair

```java
new Semaphore(3, true);
```

Attempts to honor FIFO-style ordering among waiting threads.

### Trade-off

Fairness can reduce throughput in some workloads because stronger ordering constraints can add coordination overhead.

---

# 14. Practice — Limited Resource Pool ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class ResourcePoolExample {

    private static final Semaphore RESOURCE_LIMIT = new Semaphore(3);

    public static void main(String[] args) {

        ExecutorService executor = Executors.newFixedThreadPool(6);

        for (int i = 1; i <= 10; i++) {
            int requestId = i;

            executor.submit(() -> {
                try {
                    RESOURCE_LIMIT.acquire();

                    System.out.println(
                            "Request " + requestId + " using resource");

                    Thread.sleep(1000);

                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    RESOURCE_LIMIT.release();
                    System.out.println(
                            "Request " + requestId + " released resource");
                }
            });
        }

        executor.shutdown();
    }
}
```

### Mental Model

```text
10 requests
     ↓
Semaphore(3)
     ↓
max 3 resource users
     ↓
permit released
     ↓
next waiting request
```

---

# 15. Practice — Non-Blocking Admission ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class TryAcquireExample {

    public static void main(String[] args) {

        Semaphore semaphore = new Semaphore(1);

        Thread first = new Thread(() -> {
            try {
                semaphore.acquire();
                System.out.println("First thread acquired permit");
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                semaphore.release();
            }
        });

        Thread second = new Thread(() -> {
            if (semaphore.tryAcquire()) {
                try {
                    System.out.println("Second thread acquired permit");
                } finally {
                    semaphore.release();
                }
            } else {
                System.out.println("Second thread rejected immediately");
            }
        });

        first.start();

        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        second.start();
    }
}
```

---

# 16. `availablePermits()` ⭐⭐⭐⭐

Check the number of currently available permits:

```java
int permits = semaphore.availablePermits();
```

Example:

```java
Semaphore semaphore = new Semaphore(5);
System.out.println(semaphore.availablePermits()); // 5
```

### Important

This is a snapshot. Another thread can acquire/release permits immediately after the value is read.

Do not use it as a correctness check like:

```java
if (semaphore.availablePermits() > 0) {
    // unsafe assumption
}
```

Prefer `tryAcquire()` when the actual operation is acquisition.

---

# 17. `drainPermits()` ⭐⭐⭐

Acquire all currently available permits at once:

```java
int drained = semaphore.drainPermits();
```

Example:

```java
Semaphore semaphore = new Semaphore(5);

int drained = semaphore.drainPermits();

System.out.println(drained); // 5
System.out.println(semaphore.availablePermits()); // 0
```

This is an advanced API and should be used only when the design explicitly requires draining the available capacity.

---

# 18. `reducePermits()` ⭐⭐⭐

`reducePermits(int reduction)` reduces the number of available permits without waiting.

It is a protected method, so it is generally used through a subclass rather than directly from application code.

```java
class AdjustableSemaphore extends Semaphore {

    AdjustableSemaphore(int permits) {
        super(permits);
    }

    void reduce(int reduction) {
        reducePermits(reduction);
    }
}
```

### Interview Point

> `reducePermits()` is protected and reduces available permits; it does not wait for existing holders to release permits.

---

# 19. `hasQueuedThreads()` and `getQueueLength()` ⭐⭐⭐

You can inspect waiting threads:

```java
semaphore.hasQueuedThreads();
semaphore.getQueueLength();
```

Example:

```java
System.out.println(semaphore.hasQueuedThreads());
System.out.println(semaphore.getQueueLength());
```

These values are observational snapshots and should not be treated as exact synchronization guarantees.

---

# 20. Practice — Database Connection Limit ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class DatabaseConnectionLimitExample {

    private static final Semaphore CONNECTIONS = new Semaphore(3, true);

    public static void main(String[] args) {

        ExecutorService executor = Executors.newFixedThreadPool(8);

        for (int i = 1; i <= 12; i++) {
            int requestId = i;

            executor.execute(() -> executeQuery(requestId));
        }

        executor.shutdown();
    }

    private static void executeQuery(int requestId) {
        boolean acquired = false;

        try {
            CONNECTIONS.acquire();
            acquired = true;

            System.out.println(
                    "Request " + requestId + " acquired DB connection");

            Thread.sleep(800);

            System.out.println(
                    "Request " + requestId + " completed query");

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (acquired) {
                CONNECTIONS.release();
            }
        }
    }
}
```

### Why `acquired`?

If `acquire()` itself is interrupted before obtaining a permit, blindly calling `release()` would create an extra permit.

This is an important production-level detail.

---

# 21. Critical Pitfall — Releasing Without Acquiring ⚠️ ⭐⭐⭐⭐⭐

This is dangerous:

```java
try {
    semaphore.acquire();
} finally {
    semaphore.release();
}
```

It can be correct only if the permit was definitely acquired before entering the `finally` logic.

For interruptible acquisition, a safer pattern is:

```java
boolean acquired = false;

try {
    semaphore.acquire();
    acquired = true;

    // protected work
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} finally {
    if (acquired) {
        semaphore.release();
    }
}
```

### Why?

If `acquire()` throws `InterruptedException`, no permit was acquired by that call. Releasing anyway can increase the semaphore's permit count beyond the intended capacity.

---

# 22. Critical Pitfall — Forgetting `release()` ⚠️

```java
semaphore.acquire();

// work

// release forgotten
```

Eventually all permits can be consumed permanently from the application's perspective.

Result:

```text
T1 → acquire
T2 → acquire
T3 → acquire

No permits left

T4 → waits forever
T5 → waits forever
...
```

Use `finally` whenever ownership of a permit is established.

---

# 23. Critical Pitfall — Permit Leaks in Exception Paths ⚠️

Bad:

```java
semaphore.acquire();
performOperation();
semaphore.release();
```

If `performOperation()` throws:

```text
acquire()
   ↓
exception
   ↓
release() skipped ❌
```

Correct:

```java
semaphore.acquire();
try {
    performOperation();
} finally {
    semaphore.release();
}
```

---

# 24. Semaphore vs `synchronized` ⭐⭐⭐⭐⭐

### `synchronized`

Provides mutual exclusion around a monitor:

```java
synchronized (lock) {
    // one thread at a time
}
```

### `Semaphore(1)`

Provides one permit:

```java
semaphore.acquire();
try {
    // one permit holder
} finally {
    semaphore.release();
}
```

### Key Difference

`Semaphore` is **permit-based** and does not have thread ownership semantics.

`synchronized` is based on **monitor ownership** by the executing thread.

---

# 25. Semaphore vs `ReentrantLock` ⭐⭐⭐⭐⭐

| Feature | `Semaphore` | `ReentrantLock` |
|---|---|---|
| Main concept | Permits | Lock ownership |
| Multiple concurrent holders | Yes | No |
| Permit count | N | One lock at a time |
| Ownership | No | Yes |
| `tryAcquire()` | Yes | `tryLock()` |
| Timed acquisition | Yes | Yes |
| Typical use | Capacity limiting | Mutual exclusion |

### Memory Trick

> **Lock = one owner. Semaphore = N permits.**

---

# 26. Semaphore vs `CountDownLatch` ⭐⭐⭐⭐⭐

| Feature | `Semaphore` | `CountDownLatch` |
|---|---|---|
| Main purpose | Limit concurrent access | Wait for completion/signals |
| Reusable | Yes | No |
| Controls permits | Yes | No |
| Threads acquire/release | Yes | `countDown()` decreases count |
| Typical use | Resource capacity | Startup/task completion coordination |

### Memory Trick

```text
Semaphore
→ "Only N can enter."

CountDownLatch
→ "Wait until count reaches zero."
```

---

# 27. Semaphore vs `CyclicBarrier` ⭐⭐⭐⭐⭐

| Feature | `Semaphore` | `CyclicBarrier` |
|---|---|---|
| Main idea | Limit concurrent access | Synchronize parties |
| Resource capacity | Excellent fit | Not its purpose |
| Phase synchronization | Not its purpose | Excellent fit |
| Reusable | Yes | Yes |
| Main API | `acquire/release` | `await` |

### Memory Trick

> **Semaphore = capacity. Barrier = coordination.**

---

# 28. Practice — API Concurrency Limiter ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class ApiConcurrencyLimiter {

    private final Semaphore permits = new Semaphore(4);

    public String callExternalApi(int requestId) throws InterruptedException {

        if (!permits.tryAcquire(2, TimeUnit.SECONDS)) {
            return "Request " + requestId + " rejected: capacity busy";
        }

        try {
            System.out.println("Calling external API: " + requestId);
            Thread.sleep(500);
            return "Success: " + requestId;
        } finally {
            permits.release();
        }
    }

    public static void main(String[] args) {

        ApiConcurrencyLimiter limiter = new ApiConcurrencyLimiter();

        ExecutorService executor = Executors.newFixedThreadPool(10);

        for (int i = 1; i <= 20; i++) {
            int id = i;

            executor.submit(() -> {
                try {
                    System.out.println(limiter.callExternalApi(id));
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        executor.shutdown();
    }
}
```

> This limits **concurrent executions**, not total requests per second. For rate limiting, a different algorithm/tool may be more appropriate.

---

# 29. Important: Semaphore Is Not a Rate Limiter ⭐⭐⭐⭐⭐

A semaphore limits the number of operations happening **concurrently**.

It does not inherently limit how many operations happen per second.

```text
Semaphore
→ concurrency limit
→ "How many are running now?"

Rate limiter
→ throughput/time limit
→ "How many can start per second?"
```

This distinction is commonly asked in interviews.

---

# 30. Practice — Parking Lot Scenario 🚗 ⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class ParkingLotExample {

    public static void main(String[] args) {

        Semaphore parkingSlots = new Semaphore(3);

        ExecutorService executor = Executors.newFixedThreadPool(6);

        for (int i = 1; i <= 8; i++) {
            int car = i;

            executor.submit(() -> {
                boolean parked = false;

                try {
                    parked = parkingSlots.tryAcquire(1, TimeUnit.SECONDS);

                    if (!parked) {
                        System.out.println("Car " + car + " could not park");
                        return;
                    }

                    System.out.println("Car " + car + " parked");
                    Thread.sleep(1500);

                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    if (parked) {
                        parkingSlots.release();
                        System.out.println("Car " + car + " left");
                    }
                }
            });
        }

        executor.shutdown();
    }
}
```

---

# 31. Semaphore with Multiple Permits ⭐⭐⭐⭐

```java
Semaphore semaphore = new Semaphore(5);

semaphore.acquire(2);
try {
    // operation requiring two capacity units
} finally {
    semaphore.release(2);
}
```

Think of each permit as one unit of capacity.

---

# 32. Permit Count Can Increase ⭐⭐⭐⭐

`release()` does not require a previous successful `acquire()` by the same thread.

Therefore this is legal:

```java
Semaphore semaphore = new Semaphore(0);
semaphore.release();
```

Now one permit is available.

### Interview Insight ⭐

> A semaphore's permit count is a synchronization resource, not a strict ownership record.

---

# 33. Practice — Producer Adds Capacity ⭐⭐⭐⭐

```java
import java.util.concurrent.Semaphore;

public class SemaphoreCapacityExample {

    public static void main(String[] args) {

        Semaphore semaphore = new Semaphore(0);

        Thread consumer = new Thread(() -> {
            try {
                System.out.println("Consumer waiting for permit");
                semaphore.acquire();
                System.out.println("Consumer received permit");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread producer = new Thread(() -> {
            System.out.println("Producer creates capacity");
            semaphore.release();
        });

        consumer.start();
        producer.start();
    }
}
```

---

# 34. Common Mistakes ❌

### Mistake 1 — Forgetting `release()`

Causes permit leaks.

### Mistake 2 — Releasing without acquiring

Can increase capacity unexpectedly.

### Mistake 3 — Assuming ownership

A semaphore does not enforce the acquiring thread as the releasing thread.

### Mistake 4 — Using `availablePermits()` for correctness

It is only a snapshot.

### Mistake 5 — Confusing concurrency limiting with rate limiting

Semaphore limits concurrent operations, not requests per second.

### Mistake 6 — Ignoring interruption

`acquire()` is interruptible.

### Mistake 7 — Using a semaphore when phase synchronization is needed

Use `CyclicBarrier` for repeated phase synchronization.

### Mistake 8 — Overusing fairness

Fairness may have throughput implications; enable it when ordering/fairness matters.

---

# 35. Interview Scenarios ⭐⭐⭐⭐⭐

### Q1. What is `Semaphore`?

> A concurrency utility that controls access to a resource using a fixed number of permits.

### Q2. What does `acquire()` do?

> Obtains a permit or waits until one becomes available; the interruptible form can throw `InterruptedException`.

### Q3. What does `release()` do?

> Returns/adds a permit to the semaphore.

### Q4. Is a semaphore reusable?

> Yes.

### Q5. What is `Semaphore(1)`?

> A semaphore with one permit, often used to allow only one concurrent permit holder.

### Q6. Is `Semaphore(1)` the same as `synchronized`?

> No. A semaphore is permit-based and does not have monitor ownership semantics.

### Q7. Does a semaphore have ownership?

> No. A thread other than the acquiring thread can release a permit.

### Q8. What is a fair semaphore?

> A semaphore configured with fairness, such as `new Semaphore(3, true)`, which uses a fairer acquisition policy for waiting threads.

### Q9. `acquire()` vs `tryAcquire()`?

> `acquire()` waits for a permit; `tryAcquire()` can return immediately or wait only for a specified timeout.

### Q10. Why use `finally` with `release()`?

> To prevent permit leaks when protected work throws an exception.

### Q11. What is `availablePermits()`?

> A snapshot of permits currently available.

### Q12. Is `availablePermits()` safe for deciding whether to enter?

> Not as a correctness mechanism because the value can change immediately. Use `tryAcquire()`.

### Q13. What is `drainPermits()`?

> It removes and returns all currently available permits.

### Q14. Semaphore vs `CountDownLatch`?

> Semaphore controls concurrent capacity and is reusable; latch is a one-shot countdown/signal mechanism.

### Q15. Semaphore vs `CyclicBarrier`?

> Semaphore limits concurrent access; barrier synchronizes a group at a common point.

### Q16. Semaphore vs `ReentrantLock`?

> Semaphore uses permits and supports multiple concurrent holders; `ReentrantLock` represents one ownership-based lock.

### Q17. Is a semaphore a rate limiter?

> No. It limits concurrency, not operations per time unit.

### Q18. What happens if all permits are consumed?

> A blocking `acquire()` waits until a permit becomes available or the wait is interrupted.

### Q19. Can permits be acquired/released in batches?

> Yes, using `acquire(int)` and `release(int)`.

### Q20. Explain a real-world use case.

> For example, if only 10 database connections can safely be used concurrently, a `Semaphore(10)` can limit concurrent database work.

---

# 36. Quick Revision

```text
Semaphore(N)
      ↓
N permits
      ↓
acquire()
      ↓
permit available?
   ↙       ↘
 yes       no
  ↓          ↓
enter      wait
  ↓
work
  ↓
release()
  ↓
next waiter may proceed
```

### Key APIs

```java
new Semaphore(permits)
new Semaphore(permits, true)
acquire()
acquire(int)
tryAcquire()
tryAcquire(timeout, unit)
release()
release(int)
availablePermits()
drainPermits()
hasQueuedThreads()
getQueueLength()
isFair()
```

### Key facts ⭐

```text
✓ Permit-based
✓ Reusable
✓ Controls concurrency
✓ Can have N concurrent holders
✓ Semaphore(1) gives one permit
✓ No thread ownership
✓ Supports fairness
✓ Supports timed acquisition
✓ acquire() is interruptible
✓ release() adds a permit
```

---

# 🏆 2-Minute Interview Answer

> **"`Semaphore` is a Java concurrency utility used to control access to a limited resource using permits. For example, `new Semaphore(5)` allows up to five permit holders at the same time. A thread calls `acquire()` before entering the protected section and `release()` when it is done, normally in a `finally` block to prevent permit leaks. It also supports `tryAcquire()` and timed acquisition when we don't want to wait indefinitely. Unlike `ReentrantLock` or `synchronized`, a semaphore is permit-based and does not enforce thread ownership, so another thread can release a permit. A semaphore is reusable and is commonly used for connection pools, resource capacity limits and concurrency throttling. It is important to distinguish concurrency limiting from rate limiting: a semaphore controls how many operations are active simultaneously, not how many operations can start per second."**

---

# 💻 Practice Checklist

- [ ] Create a basic `Semaphore`.
- [ ] Practice `acquire()`.
- [ ] Practice `release()`.
- [ ] Use `tryAcquire()`.
- [ ] Use timed `tryAcquire()`.
- [ ] Acquire/release multiple permits.
- [ ] Practice fair vs non-fair semaphore.
- [ ] Check `availablePermits()`.
- [ ] Practice `drainPermits()`.
- [ ] Understand `release()` without ownership.
- [ ] Build a limited resource pool.
- [ ] Build a database connection limit example.
- [ ] Build an API concurrency limiter.
- [ ] Build a parking-slot example.
- [ ] Demonstrate permit leak.
- [ ] Handle interruption correctly.
- [ ] Compare with `synchronized`.
- [ ] Compare with `ReentrantLock`.
- [ ] Compare with `CountDownLatch`.
- [ ] Compare with `CyclicBarrier`.
- [ ] Explain concurrency vs rate limiting.
- [ ] Explain the topic in under 2 minutes.

---

## Navigation

[← 8.12 — `CyclicBarrier`](../12-CyclicBarrier/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.14 — `Exchanger`**