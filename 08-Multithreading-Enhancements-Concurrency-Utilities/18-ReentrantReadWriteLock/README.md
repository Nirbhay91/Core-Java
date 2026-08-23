# 8.18 — `ReentrantReadWriteLock`

> **Goal:** Understand how `ReentrantReadWriteLock` allows multiple concurrent readers while ensuring writers have exclusive access.

---

## 1. Why `ReentrantReadWriteLock`? ⭐⭐⭐⭐⭐

A normal `ReentrantLock` uses one exclusive lock:

```text
Reader 1 → waits
Reader 2 → waits
Writer  → waits
```

`ReentrantReadWriteLock` separates access into:

```text
Read Lock  → shared
Write Lock → exclusive
```

Therefore:

```text
Reader 1 ─┐
Reader 2 ─┼─ can read concurrently
Reader 3 ─┘

Writer → requires exclusive access
```

This is useful when **reads are frequent and writes are relatively rare**.

---

# 2. Basic Structure

```java
ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

Lock readLock = lock.readLock();
Lock writeLock = lock.writeLock();
```

Usage:

```java
readLock.lock();
try {
    // read shared state
} finally {
    readLock.unlock();
}
```

```java
writeLock.lock();
try {
    // modify shared state
} finally {
    writeLock.unlock();
}
```

---

# 3. Read Lock vs Write Lock ⭐⭐⭐⭐⭐

| Feature | Read Lock | Write Lock |
|---|---|---|
| Multiple readers | Yes | — |
| Multiple writers | No | No |
| Reader + reader | Allowed | — |
| Reader + writer | Not simultaneously | Not simultaneously |
| Writer + writer | — | Not simultaneously |
| Purpose | Shared read access | Exclusive modification |
| Reentrant | Yes | Yes |

### Golden Rule

> **Many readers can coexist, but a writer needs exclusive access.**

---

# 4. Basic Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class BasicReadWriteLockExample {

    private static final ReentrantReadWriteLock lock =
            new ReentrantReadWriteLock();

    private static int value = 0;

    public static void main(String[] args) throws InterruptedException {

        Thread reader1 = new Thread(() -> readValue(), "Reader-1");
        Thread reader2 = new Thread(() -> readValue(), "Reader-2");
        Thread writer = new Thread(() -> writeValue(100), "Writer");

        reader1.start();
        reader2.start();
        writer.start();

        reader1.join();
        reader2.join();
        writer.join();
    }

    private static void readValue() {
        lock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName()
                    + " reading: " + value);
        } finally {
            lock.readLock().unlock();
        }
    }

    private static void writeValue(int newValue) {
        lock.writeLock().lock();
        try {
            value = newValue;
            System.out.println(Thread.currentThread().getName()
                    + " wrote: " + value);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

---

# 5. Multiple Readers Can Run Concurrently ⭐⭐⭐⭐⭐

```java
private static void readValue() {
    lock.readLock().lock();
    try {
        System.out.println(Thread.currentThread().getName()
                + " started reading");
        Thread.sleep(1000);
        System.out.println(Thread.currentThread().getName()
                + " finished reading");
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        lock.readLock().unlock();
    }
}
```

With several readers, they can hold the read lock together when no writer owns or is otherwise blocking the lock according to its scheduling policy.

---

# 6. Writer Is Exclusive ⭐⭐⭐⭐⭐

```java
private static void update() {
    lock.writeLock().lock();
    try {
        System.out.println("Writing...");
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        lock.writeLock().unlock();
    }
}
```

While the write lock is held:

```text
New readers → blocked
New writers → blocked
```

The exact ordering/fairness depends on the lock's fairness configuration.

---

# 7. Read-Heavy Cache ⭐⭐⭐⭐⭐

A common use case is an in-memory cache:

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteCache {

    private final Map<String, String> cache = new HashMap<>();
    private final ReentrantReadWriteLock lock =
            new ReentrantReadWriteLock();

    public String get(String key) {
        lock.readLock().lock();
        try {
            return cache.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }

    public void put(String key, String value) {
        lock.writeLock().lock();
        try {
            cache.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

### Why?

Many threads can perform `get()` concurrently, while `put()` gets exclusive access to the map.

---

# 8. Practice — Read-Heavy Cache ⭐⭐⭐⭐⭐

```java
public class CacheDemo {

    public static void main(String[] args) throws InterruptedException {

        ReadWriteCache cache = new ReadWriteCache();
        cache.put("Java", "Concurrency");

        Thread r1 = new Thread(() ->
                System.out.println(cache.get("Java")), "Reader-1");

        Thread r2 = new Thread(() ->
                System.out.println(cache.get("Java")), "Reader-2");

        Thread writer = new Thread(() ->
                cache.put("Java", "Multithreading"), "Writer");

        r1.start();
        r2.start();
        writer.start();

        r1.join();
        r2.join();
        writer.join();
    }
}
```

---

# 9. `ReentrantReadWriteLock` Is Reentrant ⭐⭐⭐⭐⭐

A thread holding the write lock can acquire the write lock again.

```java
writeLock.lock();
try {
    writeLock.lock();
    try {
        System.out.println("Reentered write lock");
    } finally {
        writeLock.unlock();
    }
} finally {
    writeLock.unlock();
}
```

The lock must be unlocked the same number of times it was acquired.

---

# 10. Read Lock Reentrancy

A thread can acquire the read lock multiple times:

```java
readLock.lock();
try {
    readLock.lock();
    try {
        System.out.println("Read lock reentered");
    } finally {
        readLock.unlock();
    }
} finally {
    readLock.unlock();
}
```

### Rule

```text
acquire twice → unlock twice
```

---

# 11. Write Lock Can Acquire Read Lock ⭐⭐⭐⭐⭐

A thread holding the write lock can acquire the read lock.

```java
writeLock.lock();
try {
    readLock.lock();
    try {
        // allowed for the owning writer
    } finally {
        readLock.unlock();
    }
} finally {
    writeLock.unlock();
}
```

This is an important distinction from attempting to upgrade a read lock to a write lock.

---

# 12. Read → Write Lock Upgrade ⚠️⭐⭐⭐⭐⭐

Do **not** assume that a thread holding a read lock can safely acquire the write lock.

Potential pattern:

```java
readLock.lock();
try {
    writeLock.lock(); // dangerous design
} finally {
    // ...
}
```

### Why?

If multiple readers simultaneously try to upgrade:

```text
Reader A → holds read lock → waits for write lock
Reader B → holds read lock → waits for write lock

Write lock cannot be granted until readers leave.
```

This can deadlock.

### Better Pattern

Release the read lock before acquiring the write lock, then re-check the condition:

```java
readLock.lock();
try {
    if (alreadyUpdated()) {
        return;
    }
} finally {
    readLock.unlock();
}

writeLock.lock();
try {
    if (!alreadyUpdated()) {
        update();
    }
} finally {
    writeLock.unlock();
}
```

---

# 13. Safe Cache Initialization Pattern ⭐⭐⭐⭐⭐

```java
public String getOrLoad(String key) {

    readLock.lock();
    try {
        String value = cache.get(key);
        if (value != null) {
            return value;
        }
    } finally {
        readLock.unlock();
    }

    writeLock.lock();
    try {
        String value = cache.get(key);

        if (value == null) {
            value = loadFromDatabase(key);
            cache.put(key, value);
        }

        return value;
    } finally {
        writeLock.unlock();
    }
}
```

### Key Point ⭐

After switching from read lock to write lock, **re-check the condition** because another thread may have changed the state while the lock was released.

---

# 14. Fair vs Non-Fair `ReentrantReadWriteLock` ⭐⭐⭐⭐

Default:

```java
new ReentrantReadWriteLock();
```

uses non-fair mode.

Fair mode:

```java
new ReentrantReadWriteLock(true);
```

Fairness affects how waiting threads are considered for access. It can reduce starvation in some workloads but may reduce throughput.

### Interview Answer

> Fair mode provides a stronger ordering policy for waiting threads, while non-fair mode generally favors throughput.

---

# 15. `getReadLockCount()` ⭐⭐⭐⭐

```java
ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

System.out.println(lock.getReadLockCount());
```

It reports the number of current read-lock holders.

This is useful for diagnostics and learning, not as a substitute for synchronization.

---

# 16. `isWriteLocked()` ⭐⭐⭐⭐

```java
System.out.println(lock.isWriteLocked());
```

Returns whether the write lock is currently held.

---

# 17. `isWriteLockedByCurrentThread()` ⭐⭐⭐⭐

```java
if (lock.isWriteLockedByCurrentThread()) {
    System.out.println("Current thread owns write lock");
}
```

Useful for diagnostics and lock-state checks.

---

# 18. `getReadHoldCount()` ⭐⭐⭐⭐

```java
System.out.println(lock.getReadHoldCount());
```

Reports the number of read holds by the **current thread**.

This differs from `getReadLockCount()`, which represents the total current read-lock holders.

---

# 19. Practice — Lock Diagnostics ⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class LockDiagnostics {

    public static void main(String[] args) {

        ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

        lock.readLock().lock();
        try {
            System.out.println("Total readers: "
                    + lock.getReadLockCount());
            System.out.println("Current thread read holds: "
                    + lock.getReadHoldCount());
            System.out.println("Write locked: "
                    + lock.isWriteLocked());
        } finally {
            lock.readLock().unlock();
        }
    }
}
```

---

# 20. `readLock()` vs `writeLock()`

```java
Lock readLock = lock.readLock();
Lock writeLock = lock.writeLock();
```

They are views over the same underlying `ReentrantReadWriteLock`.

Do not treat them as two unrelated locks.

---

# 21. `tryLock()` With Read/Write Locks ⭐⭐⭐⭐⭐

Both lock views support non-blocking acquisition.

```java
if (lock.readLock().tryLock()) {
    try {
        // read
    } finally {
        lock.readLock().unlock();
    }
}
```

Timed version:

```java
if (lock.writeLock().tryLock(1, TimeUnit.SECONDS)) {
    try {
        // write
    } finally {
        lock.writeLock().unlock();
    }
}
```

The same successful-acquisition rule applies:

```text
acquired → unlock
not acquired → don't unlock
```

---

# 22. Practice — Timed Write Lock ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class TimedWriteLockExample {

    private final ReentrantReadWriteLock lock =
            new ReentrantReadWriteLock();

    public boolean update() {
        try {
            if (!lock.writeLock().tryLock(500, TimeUnit.MILLISECONDS)) {
                return false;
            }

            try {
                System.out.println("Updating shared state...");
                return true;
            } finally {
                lock.writeLock().unlock();
            }

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
}
```

---

# 23. `ReentrantReadWriteLock` vs `ReentrantLock` ⭐⭐⭐⭐⭐

| Feature | `ReentrantLock` | `ReentrantReadWriteLock` |
|---|---|---|
| Lock modes | One | Read + Write |
| Concurrent readers | No | Yes |
| Exclusive writer | Yes | Yes |
| Complexity | Lower | Higher |
| Best for | General mutual exclusion | Read-heavy shared state |
| Upgrade concerns | No read/write distinction | Read→write upgrade needs care |

### Decision Rule

```text
Mostly reads + occasional writes
        ↓
ReentrantReadWriteLock may help

Reads and writes both frequent
        ↓
Benchmark before choosing
```

---

# 24. `synchronized` vs `ReentrantReadWriteLock` ⭐⭐⭐⭐⭐

`synchronized` provides simple mutual exclusion.

`ReentrantReadWriteLock` provides separate shared/exclusive access modes and additional lock APIs such as timed acquisition and lock-state inspection.

Use the more complex lock only when its concurrency model actually benefits the workload.

---

# 25. When NOT to Use It ⚠️

Avoid automatically using `ReentrantReadWriteLock` when:

- Writes are very frequent.
- Critical sections are tiny and contention is low.
- The data structure already provides appropriate thread safety.
- The added complexity does not improve throughput.
- You have not measured the workload.

### Interview Point ⭐

> A read-write lock is not automatically faster than a normal lock. Its advantage depends on workload characteristics.

---

# 26. Practice — Reader/Writer Simulation ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReaderWriterSimulation {

    private final ReentrantReadWriteLock lock =
            new ReentrantReadWriteLock();

    public void read() {
        lock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName()
                    + " reading");
            sleep(500);
        } finally {
            lock.readLock().unlock();
        }
    }

    public void write() {
        lock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName()
                    + " writing");
            sleep(500);
        } finally {
            lock.writeLock().unlock();
        }
    }

    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) throws InterruptedException {

        ReaderWriterSimulation service = new ReaderWriterSimulation();

        Thread r1 = new Thread(service::read, "Reader-1");
        Thread r2 = new Thread(service::read, "Reader-2");
        Thread r3 = new Thread(service::read, "Reader-3");
        Thread writer = new Thread(service::write, "Writer");

        r1.start();
        r2.start();
        r3.start();
        writer.start();

        r1.join();
        r2.join();
        r3.join();
        writer.join();
    }
}
```

---

# 27. Lock Downgrading ⭐⭐⭐⭐⭐

A write lock can be downgraded to a read lock by acquiring the read lock before releasing the write lock:

```java
writeLock.lock();
try {
    update();

    readLock.lock();
    try {
        readUpdatedValue();
    } finally {
        // release read after write is released below
    }
} finally {
    writeLock.unlock();
    readLock.unlock();
}
```

A clearer pattern:

```java
writeLock.lock();
try {
    update();
    readLock.lock();
} finally {
    writeLock.unlock();
}

try {
    readUpdatedValue();
} finally {
    readLock.unlock();
}
```

### Important

The read lock is acquired while the write lock is still held, so another writer cannot modify the state between the update and the read phase.

---

# 28. Practice — Lock Downgrade ⭐⭐⭐⭐⭐

```java
public void updateAndRead() {

    writeLock.lock();
    try {
        update();
        readLock.lock();
    } finally {
        writeLock.unlock();
    }

    try {
        readUpdatedState();
    } finally {
        readLock.unlock();
    }
}
```

This pattern is called **lock downgrading**.

---

# 29. Lock Upgrade vs Downgrade ⭐⭐⭐⭐⭐

```text
Read → Write
   ⚠️ dangerous / can deadlock

Write → Read
   ✅ supported as downgrading
```

### Interview Memory Trick

> **Downgrade is possible; upgrade is not a safe general pattern.**

---

# 30. Exception Safety ⭐⭐⭐⭐⭐

Always use `finally`:

```java
readLock.lock();
try {
    read();
} finally {
    readLock.unlock();
}
```

and:

```java
writeLock.lock();
try {
    write();
} finally {
    writeLock.unlock();
}
```

Never depend on the normal completion of the critical section to release the lock.

---

# 31. Common Mistakes ⚠️

### Mistake 1 — Forgetting unlock

```java
writeLock.lock();
write();
// missing unlock
```

### Mistake 2 — Unlocking the wrong lock

```java
readLock.lock();
try {
    read();
} finally {
    writeLock.unlock(); // wrong
}
```

### Mistake 3 — Unsafe read→write upgrade

```java
readLock.lock();
writeLock.lock();
```

### Mistake 4 — Assuming read lock always improves performance

Performance depends on workload and contention.

### Mistake 5 — Performing long I/O under the lock

Long critical sections increase contention.

---

# 32. Production Scenario — Configuration Service ⭐⭐⭐⭐⭐

```java
public class ConfigurationService {

    private final Map<String, String> config = new HashMap<>();
    private final ReentrantReadWriteLock lock =
            new ReentrantReadWriteLock();

    public String get(String key) {
        lock.readLock().lock();
        try {
            return config.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }

    public void reload(Map<String, String> newConfig) {
        lock.writeLock().lock();
        try {
            config.clear();
            config.putAll(newConfig);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

### Why this design?

```text
Normal operation → many reads
Configuration reload → occasional exclusive write
```

This is a classic read-heavy workload.

---

# 33. Production Scenario — In-Memory Metadata ⭐⭐⭐⭐

Examples:

- Application configuration
- Read-heavy reference data
- Local metadata cache
- Routing tables
- Feature configuration
- Read-mostly service state

Always validate with measurements before adopting it.

---

# 34. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is `ReentrantReadWriteLock`?

> It is a lock implementation that provides a shared read lock and an exclusive write lock, allowing concurrent readers while preventing readers and writers from accessing the protected state concurrently in conflicting modes.

### Q2. Why use it?

> It can improve concurrency for read-heavy workloads because multiple readers can proceed together.

### Q3. Can multiple threads hold the read lock?

> Yes, multiple readers can hold the read lock concurrently when no writer has exclusive access.

### Q4. Can multiple threads hold the write lock?

> No. The write lock is exclusive, although the owning thread can reenter it.

### Q5. Is it reentrant?

> Yes. Both read and write locking support reentrancy, subject to the lock's documented rules.

### Q6. Can a reader upgrade to writer safely?

> No, not as a general pattern. Attempting to acquire the write lock while holding a read lock can deadlock, especially when multiple readers attempt it.

### Q7. Can a writer acquire the read lock?

> Yes. This supports lock downgrading patterns when the read lock is acquired before releasing the write lock.

### Q8. What is lock downgrading?

> Acquiring the read lock while holding the write lock and then releasing the write lock, allowing the thread to continue under shared read protection.

### Q9. What is the default fairness mode?

> Non-fair mode.

### Q10. How do you create a fair lock?

```java
new ReentrantReadWriteLock(true);
```

### Q11. Is a read-write lock always faster?

> No. It helps primarily when reads dominate and contention makes concurrent reads valuable. Extra lock bookkeeping can hurt some workloads.

### Q12. What is the difference between `getReadLockCount()` and `getReadHoldCount()`?

> `getReadLockCount()` reports the total number of current read-lock holders, while `getReadHoldCount()` reports the current thread's read-lock hold count.

### Q13. Can read and write locks be treated as independent locks?

> No. They are two access modes of the same `ReentrantReadWriteLock` and coordinate with each other.

### Q14. Does fairness guarantee strict FIFO execution?

> No. Fairness provides a stronger ordering policy for queued threads but should not be interpreted as a universal strict FIFO execution guarantee.

---

# 35. 2-Minute Interview Answer 🏆

> **"`ReentrantReadWriteLock` is useful when shared state is read frequently but written less frequently. It provides two locks: a shared read lock and an exclusive write lock. Multiple threads can hold the read lock concurrently, but a writer requires exclusive access, so readers and writers cannot proceed in conflicting modes at the same time. It is reentrant, supports timed acquisition through its lock views, and can be configured as fair or non-fair. A key interview point is that read-to-write lock upgrading is dangerous because multiple readers can block each other while waiting for the write lock. Write-to-read downgrading is supported by acquiring the read lock before releasing the write lock. I would use it only when the workload is genuinely read-heavy and measurements show that concurrent reads provide a benefit."**

---

# 36. Quick Revision ⭐⭐⭐⭐⭐

```text
ReentrantReadWriteLock
        ↓
 ┌──────┴──────┐
 ↓             ↓
Read          Write
 ↓             ↓
Shared       Exclusive
 ↓             ↓
Many readers  One writer
```

### Golden Rules

```text
Many readers → ✅
Reader + writer → ❌
Many writers → ❌
Write → Read downgrade → ✅
Read → Write upgrade → ⚠️ dangerous
```

### Main APIs

```java
lock.readLock()
lock.writeLock()
lock.getReadLockCount()
lock.getReadHoldCount()
lock.isWriteLocked()
lock.isWriteLockedByCurrentThread()
```

---

# 💻 Practice Checklist

- [ ] Create `ReentrantReadWriteLock`
- [ ] Acquire read lock
- [ ] Acquire write lock
- [ ] Run multiple readers
- [ ] Demonstrate writer exclusivity
- [ ] Build read-heavy cache
- [ ] Understand reentrancy
- [ ] Understand read→write upgrade danger
- [ ] Implement write→read downgrade
- [ ] Use fair mode
- [ ] Use `tryLock()` on read/write locks
- [ ] Use timed write acquisition
- [ ] Inspect lock state
- [ ] Build configuration service
- [ ] Compare with `ReentrantLock`
- [ ] Compare with `synchronized`
- [ ] Identify when not to use it
- [ ] Answer interview traps
- [ ] Explain in 2 minutes

---

## Navigation

[← 8.17 — `tryLock()` and Timed Locking](../17-tryLock-and-Timed-Locking/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.19 — `StampedLock`**