# 8.19 — `StampedLock`

> **Goal:** Understand `StampedLock`, its read/write modes, optimistic reading, validation, conversion APIs, and the situations where it is useful compared with `ReentrantReadWriteLock`.

---

## 1. What is `StampedLock`? ⭐⭐⭐⭐⭐

`StampedLock` is a lock designed for read/write scenarios where **optimistic reads** can improve concurrency.

Instead of returning a boolean or using a reentrant ownership model, lock operations return a **stamp** (`long`).

```java
StampedLock lock = new StampedLock();
```

Main modes:

```text
Read Lock          → shared, pessimistic read
Write Lock         → exclusive write
Optimistic Read    → non-blocking read attempt
```

### Memory Trick

```text
StampedLock
    ↓
  stamp
    ↓
read / write / optimistic read
```

---

# 2. Why Optimistic Reading? ⭐⭐⭐⭐⭐

With a normal read lock:

```text
Reader 1 ─┐
Reader 2 ─┼─ acquire read lock
Reader 3 ─┘
```

Readers still participate in lock coordination.

With optimistic reading:

```text
Reader
  ↓
read without acquiring a traditional read lock
  ↓
validate stamp
  ↓
valid? ── yes → use result
        └─ no  → retry with read lock
```

This can be useful when writes are relatively infrequent and the read operation is short.

---

# 3. Basic API Overview ⭐⭐⭐⭐⭐

```java
long stamp = lock.writeLock();
```

```java
long stamp = lock.readLock();
```

```java
long stamp = lock.tryOptimisticRead();
```

Validation:

```java
boolean valid = lock.validate(stamp);
```

Unlocking:

```java
lock.unlockWrite(stamp);
lock.unlockRead(stamp);
```

### Important

The stamp returned by a lock acquisition must be used with the corresponding unlock operation.

---

# 4. Basic Write Lock ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.StampedLock;

public class BasicStampedLockExample {

    private final StampedLock lock = new StampedLock();
    private int value;

    public void write(int newValue) {
        long stamp = lock.writeLock();
        try {
            value = newValue;
            System.out.println("Value updated: " + value);
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

### Rule

```text
writeLock() → unlockWrite(stamp)
```

---

# 5. Basic Read Lock ⭐⭐⭐⭐⭐

```java
public int read() {
    long stamp = lock.readLock();
    try {
        return value;
    } finally {
        lock.unlockRead(stamp);
    }
}
```

### Rule

```text
readLock() → unlockRead(stamp)
```

---

# 6. Optimistic Read ⭐⭐⭐⭐⭐

The basic pattern is:

```java
long stamp = lock.tryOptimisticRead();

int currentValue = value;

if (!lock.validate(stamp)) {
    stamp = lock.readLock();
    try {
        currentValue = value;
    } finally {
        lock.unlockRead(stamp);
    }
}

return currentValue;
```

### Key Idea

The optimistic read does **not** acquire the read lock.

It reads the state and then checks whether a conflicting write occurred.

---

# 7. Practice — Optimistic Read ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.StampedLock;

public class OptimisticReadExample {

    private final StampedLock lock = new StampedLock();
    private int x = 10;
    private int y = 20;

    public int distance() {

        long stamp = lock.tryOptimisticRead();

        int localX = x;
        int localY = y;

        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                localX = x;
                localY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }

        return localX + localY;
    }

    public void update(int x, int y) {
        long stamp = lock.writeLock();
        try {
            this.x = x;
            this.y = y;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

---

# 8. Why `validate()` Matters ⭐⭐⭐⭐⭐

This is the most important optimistic-read rule.

```java
long stamp = lock.tryOptimisticRead();

int value = sharedValue;

if (lock.validate(stamp)) {
    return value;
}
```

If validation fails, the values read optimistically may have been observed while a write was occurring.

Therefore, retry under a real read lock when the application requires a consistent protected snapshot.

### Golden Rule

> **Optimistic read is only a speculation until `validate(stamp)` succeeds.**

---

# 9. Practice — Writer Interference ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.StampedLock;

public class OptimisticReadWithWriter {

    private final StampedLock lock = new StampedLock();
    private int value = 100;

    public int readValue() {

        long stamp = lock.tryOptimisticRead();
        int localValue = value;

        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                localValue = value;
            } finally {
                lock.unlockRead(stamp);
            }
        }

        return localValue;
    }

    public void update() {
        long stamp = lock.writeLock();
        try {
            value++;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

---

# 10. Optimistic Read Is Not a Read Lock ⚠️⭐⭐⭐⭐⭐

Do not write:

```java
long stamp = lock.tryOptimisticRead();

// assume this behaves exactly like readLock()
```

It does not.

An optimistic read:

- Does not block a writer from starting.
- Does not provide the same mutual exclusion as a read lock.
- Requires validation.
- May need a fallback to a real read lock.

---

# 11. Read Lock vs Optimistic Read ⭐⭐⭐⭐⭐

| Feature | `readLock()` | `tryOptimisticRead()` |
|---|---|---|
| Traditional shared lock | Yes | No |
| Blocks writer | Yes, while held | No |
| Returns stamp | Yes | Yes |
| Needs validation | No | Yes |
| Can fail immediately | No | Stamp may become invalid |
| Best for | Consistent protected read | Short read-mostly operations |

---

# 12. Write Lock Is Exclusive ⭐⭐⭐⭐⭐

```java
long stamp = lock.writeLock();
try {
    value++;
} finally {
    lock.unlockWrite(stamp);
}
```

While the write lock is held:

```text
Readers → blocked
Writers → blocked
Optimistic readers → may read but validation can fail
```

The optimistic reader is not protected from the writer; validation is what detects the conflict.

---

# 13. `StampedLock` Is Not Reentrant ⚠️⭐⭐⭐⭐⭐

This is one of the biggest differences from `ReentrantLock` and `ReentrantReadWriteLock`.

Do not assume:

```java
writeLock();
writeLock();
```

is safe reentrant acquisition by the same thread.

`StampedLock` does **not** support reentrant locking.

### Interview Memory Trick

```text
ReentrantLock           → Reentrant ✅
ReentrantReadWriteLock  → Reentrant ✅
StampedLock             → Reentrant ❌
```

---

# 14. No Ownership-Based Unlocking

A stamp identifies the lock acquisition mode/state.

Correct:

```java
long stamp = lock.writeLock();
try {
    update();
} finally {
    lock.unlockWrite(stamp);
}
```

The stamp must be retained until the corresponding unlock/conversion operation is complete.

---

# 15. Lock Conversion ⭐⭐⭐⭐⭐

`StampedLock` provides conversion APIs such as:

```java
tryConvertToWriteLock(stamp)
tryConvertToReadLock(stamp)
tryConvertToOptimisticRead(stamp)
```

These can attempt to change the current lock mode without always releasing and reacquiring from scratch.

---

# 16. Read → Write Conversion ⭐⭐⭐⭐⭐

Example pattern:

```java
long stamp = lock.readLock();

try {
    if (needsUpdate()) {
        long writeStamp = lock.tryConvertToWriteLock(stamp);

        if (writeStamp != 0L) {
            stamp = writeStamp;
            update();
        } else {
            lock.unlockRead(stamp);
            stamp = lock.writeLock();
            update();
        }
    }
} finally {
    if (StampedLock.isWriteLockStamp(stamp)) {
        lock.unlockWrite(stamp);
    } else {
        lock.unlockRead(stamp);
    }
}
```

### Important

A conversion attempt can fail and return `0`.

If it fails, release the current mode and acquire the desired mode explicitly when appropriate.

---

# 17. Practice — Conditional Update ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.StampedLock;

public class ConditionalUpdate {

    private final StampedLock lock = new StampedLock();
    private int value;

    public void incrementIfZero() {

        long stamp = lock.readLock();

        try {
            if (value != 0) {
                return;
            }

            long writeStamp = lock.tryConvertToWriteLock(stamp);

            if (writeStamp != 0L) {
                stamp = writeStamp;
                value++;
                return;
            }

        } finally {
            if (StampedLock.isWriteLockStamp(stamp)) {
                lock.unlockWrite(stamp);
            } else {
                lock.unlockRead(stamp);
            }
        }

        // If conversion failed, a production implementation should
        // release the read lock and retry with a write lock as required.
    }
}
```

### Design Point

Conversion is an optimization/coordination mechanism, not a guarantee that the desired mode will always be acquired.

---

# 18. Write → Read Conversion ⭐⭐⭐⭐

A write lock can be converted to a read lock:

```java
long stamp = lock.writeLock();

try {
    update();

    long readStamp = lock.tryConvertToReadLock(stamp);

    if (readStamp != 0L) {
        stamp = readStamp;
        readUpdatedState();
    }

} finally {
    // unlock according to the current stamp mode
}
```

This can support a write-to-read downgrade style without first exposing the state to another writer.

---

# 19. Write → Optimistic Read ⭐⭐⭐⭐

After completing a write, a lock can be converted to optimistic read mode:

```java
long stamp = lock.writeLock();

try {
    update();

    long optimisticStamp = lock.tryConvertToOptimisticRead(stamp);

    if (optimisticStamp != 0L) {
        stamp = optimisticStamp;
    }

} finally {
    if (StampedLock.isWriteLockStamp(stamp)) {
        lock.unlockWrite(stamp);
    }
}
```

An optimistic read stamp does not need `unlockRead()` because no read lock is actually held.

---

# 20. `tryOptimisticRead()` Returns a Stamp ⭐⭐⭐⭐⭐

```java
long stamp = lock.tryOptimisticRead();
```

The stamp is a token used by:

```java
lock.validate(stamp)
```

It is **not** a traditional ownership token meaning that the current thread owns a read lock.

---

# 21. Practice — Point Object ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.StampedLock;

public class Point {

    private double x;
    private double y;

    private final StampedLock lock = new StampedLock();

    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    public double distanceFromOrigin() {

        long stamp = lock.tryOptimisticRead();

        double currentX = x;
        double currentY = y;

        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }

        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

This is the classic shape of a read-mostly `StampedLock` example.

---

# 22. Why Read Both Values Before Validation? ⭐⭐⭐⭐⭐

For related state:

```java
int localX = x;
int localY = y;

if (!lock.validate(stamp)) {
    // retry under read lock
}
```

The optimistic snapshot is accepted only if validation succeeds.

If a write occurred, retrying under a read lock ensures the protected read is performed under the normal read-lock protocol.

---

# 23. Practice — Multiple Readers

```java
import java.util.concurrent.locks.StampedLock;

public class MultiReaderStampedLock {

    private final StampedLock lock = new StampedLock();
    private int value = 100;

    public int read() {
        long stamp = lock.readLock();
        try {
            return value;
        } finally {
            lock.unlockRead(stamp);
        }
    }

    public void write(int newValue) {
        long stamp = lock.writeLock();
        try {
            value = newValue;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

Unlike optimistic reading, this uses a genuine shared read lock.

---

# 24. Timed Lock Acquisition ⭐⭐⭐⭐⭐

`StampedLock` also supports timed attempts.

```java
long stamp = lock.tryReadLock(1, TimeUnit.SECONDS);
```

and:

```java
long stamp = lock.tryWriteLock(1, TimeUnit.SECONDS);
```

If acquisition fails:

```java
stamp == 0L
```

Timed methods can throw `InterruptedException`.

---

# 25. Practice — Timed Write Lock ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.StampedLock;

public class TimedStampedWrite {

    private final StampedLock lock = new StampedLock();

    public boolean update() {
        long stamp = 0L;

        try {
            stamp = lock.tryWriteLock(500, TimeUnit.MILLISECONDS);

            if (stamp == 0L) {
                System.out.println("Could not acquire write lock");
                return false;
            }

            System.out.println("Write lock acquired");
            return true;

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;

        } finally {
            if (stamp != 0L) {
                lock.unlockWrite(stamp);
            }
        }
    }
}
```

---

# 26. `tryWriteLock()` vs `writeLock()`

| API | Behavior |
|---|---|
| `writeLock()` | Waits until write lock is acquired |
| `tryWriteLock()` | Immediate attempt; returns `0` on failure |
| `tryWriteLock(timeout, unit)` | Waits up to timeout |

Similarly:

```text
readLock()
tryReadLock()
tryReadLock(timeout, unit)
```

---

# 27. Important: No `Condition` Support ⚠️

Unlike `ReentrantLock`, `StampedLock` does not provide `newCondition()`.

So this is not available:

```java
lock.newCondition(); // no such API
```

If your design depends heavily on `Condition` objects, `ReentrantLock` may be a better fit.

---

# 28. Important: No Reentrant Semantics ⚠️

Do not use `StampedLock` as a drop-in replacement for:

```java
ReentrantLock
```

The programming model is different.

The caller must carefully manage stamps and lock modes.

---

# 29. `StampedLock` vs `ReentrantReadWriteLock` ⭐⭐⭐⭐⭐

| Feature | `ReentrantReadWriteLock` | `StampedLock` |
|---|---|---|
| Read lock | Yes | Yes |
| Write lock | Yes | Yes |
| Optimistic read | No | Yes |
| Reentrant | Yes | No |
| Stamp/token based | No | Yes |
| Lock conversion | Limited | Yes |
| `Condition` support | Yes via write lock | No |
| Complexity | Lower | Higher |
| Best use | General read/write locking | Read-heavy optimistic workloads |

### Interview Rule

> Use `StampedLock` because you need its optimistic-read/conversion model, not simply because it is newer or more advanced.

---

# 30. `StampedLock` vs `ReentrantLock` ⭐⭐⭐⭐⭐

```text
ReentrantLock
→ general exclusive locking
→ reentrant
→ Condition support

StampedLock
→ read/write + optimistic read
→ stamp-based
→ not reentrant
→ no Condition
```

---

# 31. When Should You Use `StampedLock`? ⭐⭐⭐⭐⭐

Good candidates:

- Read-heavy shared state
- Short read operations
- Writes are relatively infrequent
- Optimistic reads can avoid unnecessary blocking
- You can tolerate retrying reads when validation fails
- Lock conversion is useful

Examples:

```text
Geometry / coordinates
Read-mostly caches
Reference state
Configuration snapshots
In-memory indexes
```

---

# 32. When Should You NOT Use It? ⚠️

Avoid it when:

- You need reentrant locking.
- You need `Condition` objects.
- The workload is mostly writes.
- The critical sections are complicated.
- The team cannot safely manage stamp lifetimes.
- Simpler synchronization is sufficient.

### Senior Interview Point

> More concurrency primitives do not automatically mean better performance. Correctness and measured workload characteristics come first.

---

# 33. Common Mistake — Forgetting Validation ⚠️

Wrong:

```java
long stamp = lock.tryOptimisticRead();
int value = sharedValue;
return value;
```

Correct:

```java
long stamp = lock.tryOptimisticRead();
int value = sharedValue;

if (!lock.validate(stamp)) {
    stamp = lock.readLock();
    try {
        value = sharedValue;
    } finally {
        lock.unlockRead(stamp);
    }
}

return value;
```

---

# 34. Common Mistake — Unlocking Optimistic Read ⚠️

Wrong:

```java
long stamp = lock.tryOptimisticRead();

try {
    read();
} finally {
    lock.unlockRead(stamp); // wrong
}
```

An optimistic read does not hold a read lock.

Use:

```java
lock.validate(stamp);
```

instead.

---

# 35. Common Mistake — Assuming Stamp `0` Is Success ⚠️

For `tryReadLock()` / `tryWriteLock()`:

```java
long stamp = lock.tryWriteLock();
```

The result:

```text
stamp == 0L → acquisition failed
stamp != 0L → acquisition succeeded
```

Always check the stamp before unlocking.

---

# 36. Common Mistake — Unlocking With Wrong Mode ⚠️

If a write lock was acquired:

```java
long stamp = lock.writeLock();
```

use:

```java
lock.unlockWrite(stamp);
```

If a read lock was acquired:

```java
long stamp = lock.readLock();
```

use:

```java
lock.unlockRead(stamp);
```

---

# 37. Production Scenario — Read-Mostly Coordinates ⭐⭐⭐⭐⭐

```java
public class Coordinates {

    private double x;
    private double y;

    private final StampedLock lock = new StampedLock();

    public double getDistance() {

        long stamp = lock.tryOptimisticRead();
        double localX = x;
        double localY = y;

        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                localX = x;
                localY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }

        return Math.sqrt(localX * localX + localY * localY);
    }

    public void move(double dx, double dy) {
        long stamp = lock.writeLock();
        try {
            x += dx;
            y += dy;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

---

# 38. Practice — Threaded Read/Write Demo ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.StampedLock;

public class StampedLockThreadDemo {

    private final StampedLock lock = new StampedLock();
    private int value = 0;

    public int optimisticRead() {
        long stamp = lock.tryOptimisticRead();
        int localValue = value;

        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                localValue = value;
            } finally {
                lock.unlockRead(stamp);
            }
        }

        return localValue;
    }

    public void increment() {
        long stamp = lock.writeLock();
        try {
            value++;
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    public static void main(String[] args) throws InterruptedException {

        StampedLockThreadDemo demo = new StampedLockThreadDemo();

        Thread writer = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                demo.increment();
            }
        }, "Writer");

        Thread reader1 = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Reader-1: "
                        + demo.optimisticRead());
            }
        }, "Reader-1");

        Thread reader2 = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Reader-2: "
                        + demo.optimisticRead());
            }
        }, "Reader-2");

        writer.start();
        reader1.start();
        reader2.start();

        writer.join();
        reader1.join();
        reader2.join();
    }
}
```

---

# 39. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is `StampedLock`?

> A Java lock designed for read/write coordination that additionally supports optimistic reads using stamps.

### Q2. What are its three modes?

> Write lock, read lock, and optimistic read.

### Q3. What is optimistic reading?

> It reads shared state without acquiring a traditional read lock and then validates the stamp. If validation fails, the read should be retried using a proper read lock when consistency requires it.

### Q4. Why call `validate()`?

> To check whether a conflicting write occurred after the optimistic read stamp was obtained.

### Q5. Is `StampedLock` reentrant?

> No. It does not provide reentrant lock semantics.

### Q6. What does `tryOptimisticRead()` return?

> A stamp representing the optimistic read attempt.

### Q7. How do you unlock a write lock?

```java
lock.unlockWrite(stamp);
```

### Q8. How do you unlock a read lock?

```java
lock.unlockRead(stamp);
```

### Q9. Do you unlock an optimistic read?

> No. An optimistic read does not acquire a read lock. You validate the stamp instead.

### Q10. What does a zero stamp mean for `tryReadLock()` / `tryWriteLock()`?

> It means the lock was not acquired.

### Q11. Can `StampedLock` convert between modes?

> Yes. It provides conversion methods such as `tryConvertToWriteLock`, `tryConvertToReadLock`, and `tryConvertToOptimisticRead`.

### Q12. Does conversion always succeed?

> No. A conversion can return `0`, so the caller must handle failure.

### Q13. `StampedLock` vs `ReentrantReadWriteLock`?

> `StampedLock` adds optimistic reads and stamp-based conversion but is not reentrant and does not support `Condition`. `ReentrantReadWriteLock` is reentrant and provides a simpler read/write locking model.

### Q14. When is `StampedLock` useful?

> In read-heavy workloads where short optimistic reads can frequently succeed without requiring a normal read lock.

### Q15. Does `StampedLock` guarantee better performance?

> No. Its benefit is workload-dependent and should be verified with measurement.

### Q16. What is the biggest trap with optimistic reads?

> Treating an optimistic read as if it were a real read lock and forgetting to validate the stamp.

---

# 40. 2-Minute Interview Answer 🏆

> **"`StampedLock` is a Java concurrency utility designed for read/write scenarios where optimistic reads can improve concurrency. It provides three modes: an exclusive write lock, a shared read lock, and an optimistic read. With an optimistic read, I call `tryOptimisticRead()`, copy the required state, and then call `validate(stamp)`. If validation fails, I retry under the normal read lock when I need a consistent protected read. Unlike `ReentrantLock` and `ReentrantReadWriteLock`, `StampedLock` is not reentrant and it does not provide `Condition` support. It also uses stamps for unlocking and supports lock conversion methods. I would use it for read-heavy workloads with short reads and relatively infrequent writes, but I would not use it just because it is more advanced; the workload and measured performance should justify the additional complexity."**

---

# 41. Quick Revision ⭐⭐⭐⭐⭐

```text
StampedLock
     ↓
 ┌───┼────────────┐
 ↓   ↓            ↓
Read Write  Optimistic Read
 ↓    ↓           ↓
shared exclusive  no real lock
                 ↓
              validate()
```

### Golden Pattern

```java
long stamp = lock.tryOptimisticRead();

int value = sharedValue;

if (!lock.validate(stamp)) {
    stamp = lock.readLock();
    try {
        value = sharedValue;
    } finally {
        lock.unlockRead(stamp);
    }
}
```

### Remember

```text
Optimistic read → validate
Read lock       → unlockRead(stamp)
Write lock      → unlockWrite(stamp)
Try lock fail   → stamp == 0
StampedLock     → NOT reentrant
```

---

# 42. 💻 Practice Checklist

- [ ] Create `StampedLock`
- [ ] Acquire write lock
- [ ] Acquire read lock
- [ ] Understand optimistic read
- [ ] Use `tryOptimisticRead()`
- [ ] Use `validate()`
- [ ] Implement fallback to read lock
- [ ] Understand writer interference
- [ ] Use `tryReadLock()`
- [ ] Use `tryWriteLock()`
- [ ] Use timed lock acquisition
- [ ] Understand stamps
- [ ] Unlock with correct stamp/mode
- [ ] Understand read→write conversion
- [ ] Understand write→read conversion
- [ ] Understand write→optimistic conversion
- [ ] Remember non-reentrancy
- [ ] Compare with `ReentrantReadWriteLock`
- [ ] Know lack of `Condition` support
- [ ] Identify suitable workloads
- [ ] Identify when not to use it
- [ ] Answer interview traps
- [ ] Explain in 2 minutes

---

## Navigation

[← 8.18 — `ReentrantReadWriteLock`](../18-ReentrantReadWriteLock/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.20 — `Condition`**