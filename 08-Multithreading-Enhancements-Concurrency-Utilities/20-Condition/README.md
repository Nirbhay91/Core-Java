# 8.20 — `Condition`

> **Goal:** Understand how `Condition` provides explicit thread communication with `ReentrantLock`, and how `await()`, `signal()`, and `signalAll()` work internally with lock ownership.

---

## 1. What is `Condition`? ⭐⭐⭐⭐⭐

`Condition` is a Java concurrency interface from `java.util.concurrent.locks` used for **thread communication and coordination** with an associated `Lock`, typically a `ReentrantLock`.

It provides methods similar in purpose to `wait()`, `notify()`, and `notifyAll()`, but with an explicit lock and potentially **multiple independent waiting conditions**.

```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
```

### Memory Trick

```text
synchronized + wait/notify
        ↓
ReentrantLock + Condition
        ↓
await / signal / signalAll
```

---

# 2. Why Do We Need `Condition`? ⭐⭐⭐⭐⭐

With intrinsic monitors:

```java
synchronized (lock) {
    lock.wait();
    lock.notify();
}
```

A monitor has one implicit wait set.

With `ReentrantLock`, you can create multiple conditions:

```java
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();
```

This is especially useful for producer-consumer designs.

```text
             ReentrantLock
                  │
          ┌───────┴────────┐
          ↓                ↓
      notEmpty           notFull
          ↓                ↓
       consumers        producers
```

---

# 3. Creating a `Condition` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
```

The condition is associated with that particular lock.

---

# 4. `await()` ⭐⭐⭐⭐⭐

Basic pattern:

```java
lock.lock();
try {
    while (!conditionIsTrue()) {
        condition.await();
    }

    // continue work
} finally {
    lock.unlock();
}
```

### What `await()` Does

Conceptually:

```text
Thread owns lock
      ↓
await()
      ↓
releases lock
      ↓
thread waits
      ↓
signal()
      ↓
thread becomes eligible to reacquire lock
      ↓
reacquires lock
      ↓
await() returns
```

The thread does **not** continue immediately after being signalled; it must reacquire the associated lock first.

---

# 5. `signal()` ⭐⭐⭐⭐⭐

```java
lock.lock();
try {
    condition.signal();
} finally {
    lock.unlock();
}
```

`signal()` wakes one waiting thread on that specific condition, if one exists.

Important:

> `signal()` does not hand the lock directly to the waiting thread.

The signalling thread still owns the lock until it releases it.

---

# 6. `signalAll()` ⭐⭐⭐⭐⭐

```java
lock.lock();
try {
    condition.signalAll();
} finally {
    lock.unlock();
}
```

`signalAll()` signals all threads currently waiting on that condition.

They will compete to reacquire the associated lock.

---

# 7. The Golden Pattern ⭐⭐⭐⭐⭐

Always protect the condition with the associated lock:

```java
lock.lock();
try {
    while (!conditionIsTrue()) {
        condition.await();
    }

    // perform work
} finally {
    lock.unlock();
}
```

For signalling:

```java
lock.lock();
try {
    changeState();
    condition.signal();
} finally {
    lock.unlock();
}
```

---

# 8. Why `while`, Not `if`? ⭐⭐⭐⭐⭐

Wrong:

```java
if (!available) {
    condition.await();
}
useResource();
```

Correct:

```java
while (!available) {
    condition.await();
}
useResource();
```

Why?

- A thread can wake and find that the condition is no longer true.
- Multiple waiting threads may compete for the resource.
- The state may change before the awakened thread reacquires the lock.
- `await()` can return because of interruption or timeout depending on the overload used.

### Interview Rule

> **Wait for a condition in a loop; never assume that being awakened means the condition is true.**

---

# 9. Practice — Basic `Condition` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class BasicConditionExample {

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();

    private boolean ready = false;

    public void awaitReady() throws InterruptedException {
        lock.lock();
        try {
            while (!ready) {
                System.out.println(Thread.currentThread().getName()
                        + " waiting...");
                condition.await();
            }

            System.out.println(Thread.currentThread().getName()
                    + " continuing...");
        } finally {
            lock.unlock();
        }
    }

    public void makeReady() {
        lock.lock();
        try {
            ready = true;
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
```

---

# 10. Practice — Complete Producer/Consumer ⭐⭐⭐⭐⭐

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ConditionProducerConsumer {

    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 3;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public void put(int value) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();
            }

            queue.add(value);
            System.out.println("Produced: " + value);

            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public int take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }

            int value = queue.remove();
            System.out.println("Consumed: " + value);

            notFull.signal();
            return value;
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {

        ConditionProducerConsumer buffer = new ConditionProducerConsumer();

        Thread producer = new Thread(() -> {
            for (int i = 1; i <= 10; i++) {
                try {
                    buffer.put(i);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        }, "Producer");

        Thread consumer = new Thread(() -> {
            for (int i = 1; i <= 10; i++) {
                try {
                    buffer.take();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        }, "Consumer");

        producer.start();
        consumer.start();

        producer.join();
        consumer.join();
    }
}
```

### Why two conditions?

```text
notFull
→ producers wait when buffer is full

notEmpty
→ consumers wait when buffer is empty
```

This is more precise than having every thread wait on the same condition.

---

# 11. Multiple Conditions ⭐⭐⭐⭐⭐

A single lock can have multiple conditions:

```java
ReentrantLock lock = new ReentrantLock();

Condition conditionA = lock.newCondition();
Condition conditionB = lock.newCondition();
```

Example:

```text
             One Lock
                │
       ┌────────┴────────┐
       ↓                 ↓
    notEmpty           notFull
       ↓                 ↓
   Consumers          Producers
```

This is one of the major advantages over a single intrinsic monitor wait set.

---

# 12. `Condition` vs `Object.wait()` ⭐⭐⭐⭐⭐

| `Object.wait()` | `Condition.await()` |
|---|---|
| Used with intrinsic monitor | Used with associated `Lock` |
| `synchronized` required | `lock.lock()` required |
| One wait set per object monitor | Multiple conditions can be created from one lock |
| `notify()` | `signal()` |
| `notifyAll()` | `signalAll()` |
| Releases monitor while waiting | Releases associated lock while waiting |
| Reacquires monitor before returning | Reacquires associated lock before returning |

---

# 13. `Condition` vs `wait/notify` ⭐⭐⭐⭐⭐

Conceptual mapping:

```text
Object.wait()      ↔ Condition.await()
Object.notify()    ↔ Condition.signal()
Object.notifyAll() ↔ Condition.signalAll()
```

But they are **not interchangeable APIs**.

`Condition` belongs to the explicit-lock framework.

---

# 14. `await()` Releases the Lock ⭐⭐⭐⭐⭐

Suppose:

```java
lock.lock();
try {
    condition.await();
} finally {
    lock.unlock();
}
```

When `await()` starts waiting:

```text
Thread A owns lock
       ↓
A calls await()
       ↓
A releases lock
       ↓
Thread B can acquire lock
```

Without releasing the lock, another thread could never acquire the lock to change the condition and signal the waiter.

---

# 15. `await()` Reacquires Before Returning ⭐⭐⭐⭐⭐

Important sequence:

```text
await()
  ↓
release lock
  ↓
wait
  ↓
signal
  ↓
become eligible
  ↓
reacquire lock
  ↓
await() returns
```

Therefore, after `await()` returns, the current thread again holds the associated lock.

---

# 16. `signal()` Does Not Release the Lock ⚠️⭐⭐⭐⭐⭐

Wrong mental model:

```text
signal()
  ↓
waiting thread immediately runs
```

Correct:

```text
signaller owns lock
       ↓
signal()
       ↓
waiter becomes eligible
       ↓
signaller eventually unlocks
       ↓
waiter competes to reacquire lock
       ↓
waiter continues from await()
```

---

# 17. Practice — `signal()` vs `signalAll()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class SignalVsSignalAll {

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();
    private boolean ready;

    public void waitForReady() throws InterruptedException {
        lock.lock();
        try {
            while (!ready) {
                System.out.println(Thread.currentThread().getName()
                        + " waiting");
                condition.await();
            }
            System.out.println(Thread.currentThread().getName()
                    + " resumed");
        } finally {
            lock.unlock();
        }
    }

    public void signalOne() {
        lock.lock();
        try {
            ready = true;
            condition.signal();
        } finally {
            lock.unlock();
        }
    }

    public void signalEveryone() {
        lock.lock();
        try {
            ready = true;
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
```

---

# 18. `signal()` or `signalAll()`? ⭐⭐⭐⭐⭐

Use `signal()` when:

- One waiter is enough.
- The signalled thread can make useful progress.
- You understand the condition/state relationship.

Use `signalAll()` when:

- Multiple waiters may now proceed.
- You cannot safely identify one suitable waiter.
- The state change can satisfy multiple waiting threads.

### Caution

`signalAll()` can cause many threads to wake and then compete for the same lock, so it may create a **thundering-herd style effect**.

---

# 19. `awaitUninterruptibly()` ⭐⭐⭐⭐

`Condition` provides:

```java
condition.awaitUninterruptibly();
```

It waits without responding to interruption in the same way as interruptible `await()`.

However, interruption status handling still matters. Do not use this method casually when interruption is part of your application's cancellation/shutdown protocol.

---

# 20. Timed `await()` ⭐⭐⭐⭐⭐

`Condition` supports timed waiting:

```java
condition.await(1, TimeUnit.SECONDS);
```

It returns a boolean indicating whether the wait timed out.

Another option:

```java
condition.awaitNanos(nanos);
```

There are also deadline-style APIs such as:

```java
condition.awaitUntil(deadline);
```

---

# 21. Practice — Timed Condition Wait ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class TimedConditionWait {

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();

    public void waitForSignal() {
        lock.lock();
        try {
            boolean signalled = condition.await(2, TimeUnit.SECONDS);

            if (signalled) {
                System.out.println("Condition signalled");
            } else {
                System.out.println("Timed out");
            }

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            lock.unlock();
        }
    }
}
```

---

# 22. `awaitNanos()` ⭐⭐⭐⭐

```java
long remaining = condition.awaitNanos(timeoutNanos);
```

It can be useful when repeatedly waiting while tracking the remaining timeout.

Example pattern:

```java
long remaining = timeoutNanos;

while (!ready && remaining > 0) {
    remaining = condition.awaitNanos(remaining);
}
```

The condition must still be checked in a loop.

---

# 23. `awaitUntil()` ⭐⭐⭐

```java
Date deadline = ...;
boolean signalled = condition.awaitUntil(deadline);
```

It waits until the specified deadline or until signalled/interrupted.

---

# 24. Interrupt Handling ⭐⭐⭐⭐⭐

`await()` is interruptible.

```java
try {
    condition.await();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Best Practice

If your method cannot meaningfully propagate `InterruptedException`, restore the interruption status:

```java
Thread.currentThread().interrupt();
```

Do not silently swallow interruption.

---

# 25. Common Mistake — Calling `await()` Without Lock ⚠️

Wrong:

```java
condition.await();
```

without holding the associated lock.

This results in:

```text
IllegalMonitorStateException
```

Correct:

```java
lock.lock();
try {
    condition.await();
} finally {
    lock.unlock();
}
```

---

# 26. Common Mistake — Calling `signal()` Without Lock ⚠️

Wrong:

```java
condition.signal();
```

without holding the associated lock.

Correct:

```java
lock.lock();
try {
    condition.signal();
} finally {
    lock.unlock();
}
```

---

# 27. Common Mistake — Using `if` Instead of `while` ⚠️

Wrong:

```java
if (queue.isEmpty()) {
    notEmpty.await();
}
return queue.remove();
```

Correct:

```java
while (queue.isEmpty()) {
    notEmpty.await();
}
return queue.remove();
```

---

# 28. Common Mistake — Signalling the Wrong Condition ⚠️

Suppose:

```java
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();
```

After adding an item:

```java
notEmpty.signal();
```

After removing an item:

```java
notFull.signal();
```

Signalling the wrong condition can leave the required threads sleeping.

---

# 29. Common Mistake — Unlocking Before State Change ⚠️

Prefer:

```java
lock.lock();
try {
    ready = true;
    condition.signalAll();
} finally {
    lock.unlock();
}
```

The state change and signalling should be coordinated under the same lock protecting the condition predicate.

---

# 30. Practice — Bounded Buffer ⭐⭐⭐⭐⭐

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedBuffer<T> {

    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();

    public BoundedBuffer(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be > 0");
        }
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();
            }

            queue.add(item);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }

            T item = queue.remove();
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

# 31. `Condition` With `ReentrantLock` ⭐⭐⭐⭐⭐

The relationship is:

```text
ReentrantLock
     │
     ├── Condition A
     ├── Condition B
     └── Condition C
```

Every condition has its own waiting set.

This allows more targeted signalling than a single intrinsic monitor wait set.

---

# 32. Fair `ReentrantLock` + `Condition`

```java
ReentrantLock lock = new ReentrantLock(true);
Condition condition = lock.newCondition();
```

The fairness setting belongs to the lock, not to the `Condition` itself.

Fairness does not mean that every application-level scheduling requirement is automatically guaranteed.

---

# 33. Condition Is Not a Lock ⭐⭐⭐⭐⭐

Important distinction:

```text
ReentrantLock → controls mutual exclusion
Condition     → controls waiting/signalling around a state predicate
```

You do not use a `Condition` by itself to protect shared state.

---

# 34. Condition Is Not a Queue ⭐⭐⭐

A `Condition` does not store your application data.

For producer-consumer:

```text
Queue       → stores items
Lock        → protects queue state
Conditions  → coordinate producers/consumers
```

---

# 35. Practice — State Machine Coordination ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class StateMachine {

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition running = lock.newCondition();

    private boolean isRunning;

    public void waitUntilRunning() throws InterruptedException {
        lock.lock();
        try {
            while (!isRunning) {
                running.await();
            }
            System.out.println("System is running");
        } finally {
            lock.unlock();
        }
    }

    public void startSystem() {
        lock.lock();
        try {
            isRunning = true;
            running.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
```

---

# 36. `signal()` and Multiple Waiters ⭐⭐⭐⭐⭐

If five threads are waiting:

```java
condition.signal();
```

signals one waiter.

```java
condition.signalAll();
```

signals all five waiters.

But after `signalAll()`:

```text
5 threads become eligible
       ↓
all compete for lock
       ↓
one acquires first
       ↓
checks predicate
       ↓
others acquire later
       ↓
re-check predicate
```

That is another reason the predicate must be inside `while`.

---

# 37. Why Multiple Conditions Improve Producer/Consumer ⭐⭐⭐⭐⭐

With one condition:

```text
Producer + Consumer
       ↓
   same wait set
```

A signal can wake a thread that cannot make progress.

With two conditions:

```text
notFull  → producers
notEmpty → consumers
```

The signal is more targeted.

This can improve efficiency and clarity.

---

# 38. `Condition` vs `Semaphore`

They solve different problems.

| `Condition` | `Semaphore` |
|---|---|
| Wait for a state predicate | Control permits |
| Usually associated with a lock | Independent synchronizer |
| `await()` / `signal()` | `acquire()` / `release()` |
| Good for explicit state coordination | Good for resource/concurrency limits |

---

# 39. `Condition` vs `CountDownLatch`

| `Condition` | `CountDownLatch` |
|---|---|
| Reusable coordination mechanism | One-shot countdown |
| State predicate controlled by application | Counter reaches zero |
| `await()` / `signal()` | `await()` / `countDown()` |
| Can have multiple conditions per lock | Fixed count |

---

# 40. Production Scenario — Order Processing ⭐⭐⭐⭐⭐

Imagine an order processor:

```text
ORDER_CREATED
      ↓
processing thread waits
      ↓
PAYMENT_COMPLETED
      ↓
signal processing condition
      ↓
order processing continues
```

The state predicate might be:

```java
while (!paymentCompleted) {
    condition.await();
}
```

When payment completes:

```java
paymentCompleted = true;
condition.signalAll();
```

The actual production design may prefer higher-level concurrency abstractions depending on architecture, but `Condition` is useful for understanding and implementing explicit state coordination.

---

# 41. Production Scenario — Connection Pool ⭐⭐⭐⭐⭐

A bounded connection pool can use:

```text
Condition: connectionAvailable
```

When no connection exists:

```java
while (availableConnections.isEmpty()) {
    connectionAvailable.await();
}
```

When a connection is returned:

```java
availableConnections.add(connection);
connectionAvailable.signal();
```

In real applications, existing concurrency utilities may be preferable, but the pattern demonstrates the role of a condition predicate.

---

# 42. Interview Traps ⭐⭐⭐⭐⭐

### Trap 1
**Can `await()` be called without holding the lock?**

No. The associated lock must be held.

### Trap 2
**Does `signal()` release the lock?**

No.

### Trap 3
**Does `signal()` immediately run the waiter?**

No. The waiter must reacquire the lock first.

### Trap 4
**Why use `while` instead of `if`?**

Because waking up does not guarantee that the predicate is true when the thread gets the lock again.

### Trap 5
**Can one `ReentrantLock` have multiple conditions?**

Yes.

### Trap 6
**Does `Condition` replace the lock?**

No. The lock protects state; the condition coordinates waiting/signalling around that state.

### Trap 7
**What happens if the wrong condition is signalled?**

The intended waiting threads may remain blocked because each condition has its own wait set.

---

# 43. `Condition` vs `wait/notify` Interview Table ⭐⭐⭐⭐⭐

| Concept | Monitor | Explicit Lock |
|---|---|---|
| Lock | `synchronized` | `ReentrantLock` |
| Wait | `wait()` | `await()` |
| Wake one | `notify()` | `signal()` |
| Wake all | `notifyAll()` | `signalAll()` |
| Wait sets | One per monitor | Multiple per lock |
| Timed waiting | Yes | Yes, with multiple overloads |
| Interruptible wait | Yes | Yes |

---

# 44. 2-Minute Interview Answer 🏆

> **"`Condition` is part of Java's explicit locking API and is used with a `Lock`, commonly `ReentrantLock`, for thread coordination. It is conceptually similar to `wait`, `notify`, and `notifyAll`, but it gives us explicit conditions and allows multiple wait sets for a single lock. A thread calls `await()` while holding the associated lock; `await()` atomically releases that lock and waits, then reacquires the lock before returning. A signalling thread calls `signal()` or `signalAll()` while holding the same lock. `signal()` wakes one waiter and `signalAll()` wakes all waiters, but signalling does not release the lock. The condition predicate should always be checked in a `while` loop. A common example is a bounded producer-consumer buffer using separate `notFull` and `notEmpty` conditions. I would use `Condition` when I need precise, explicit thread coordination with a lock and multiple waiting conditions."**

---

# 45. Quick Revision ⭐⭐⭐⭐⭐

```text
ReentrantLock
      ↓
  newCondition()
      ↓
   Condition
      ↓
 ┌────┼─────────┐
 ↓    ↓         ↓
await signal signalAll
 ↓
releases lock
 ↓
wait
 ↓
signal
 ↓
reacquire lock
 ↓
continue
```

### Golden Rules

```text
Condition belongs to a Lock
await() → releases lock while waiting
await() → reacquires lock before returning
signal() → one waiter
signalAll() → all waiters
signal() does NOT unlock
Use while, not if
One Lock → multiple Conditions possible
```

---

# 46. 💻 Practice Checklist

- [ ] Create `ReentrantLock`
- [ ] Create a `Condition`
- [ ] Use `await()`
- [ ] Use `signal()`
- [ ] Use `signalAll()`
- [ ] Understand lock release during `await()`
- [ ] Understand lock reacquisition after `await()`
- [ ] Use `while` predicate checks
- [ ] Handle `InterruptedException`
- [ ] Use timed `await()`
- [ ] Understand `awaitNanos()`
- [ ] Understand `awaitUntil()`
- [ ] Use multiple conditions with one lock
- [ ] Build producer-consumer
- [ ] Build bounded buffer
- [ ] Compare `Condition` with `wait/notify`
- [ ] Compare `Condition` with `Semaphore`
- [ ] Compare `Condition` with `CountDownLatch`
- [ ] Identify wrong-condition bugs
- [ ] Explain `signal()` lock ownership
- [ ] Answer interview traps
- [ ] Explain in 2 minutes

---

## Navigation

[← 8.19 — `StampedLock`](../19-StampedLock/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.21 — Atomic Variables & CAS**