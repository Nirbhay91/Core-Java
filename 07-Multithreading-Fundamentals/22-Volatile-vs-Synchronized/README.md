# 7.22 — `volatile` vs `synchronized`

## 🎯 Objective

Understand the difference between `volatile` and `synchronized`, what guarantees each provides, when to use them, and the interview traps around **visibility, atomicity, mutual exclusion, ordering, and thread safety**.

> **Interview rule:** `volatile` is primarily for **visibility + ordering semantics** of a shared field. `synchronized` provides **mutual exclusion + visibility + ordering** for a critical section.

---

# 1. Core Difference ⭐⭐⭐⭐⭐

```text
volatile
   ↓
Visibility + ordering semantics
   ↓
No mutual exclusion
No general compound-operation atomicity

synchronized
   ↓
Mutual exclusion
+ visibility
+ ordering
+ atomicity of the protected critical section
```

### One-line interview answer

> **Use `volatile` when threads need to safely observe a shared state change without a critical section; use `synchronized` when multiple operations must execute as one protected critical section.**

---

# 2. What Does `volatile` Provide? ⭐⭐⭐⭐⭐

For a volatile field:

```java
private volatile boolean running;
```

Java provides volatile memory semantics for reads and writes of that field.

A volatile write happens-before a subsequent read of the same volatile variable.

```text
Thread A
volatile write
      ↓
 happens-before
      ↓
Thread B
volatile read
```

This makes `volatile` useful for simple communication flags and publication patterns.

---

# 3. What Does `synchronized` Provide? ⭐⭐⭐⭐⭐

`synchronized` uses a monitor to protect a critical section.

```java
synchronized (lock) {
    // critical section
}
```

It provides:

- Mutual exclusion
- Visibility
- Ordering/memory synchronization
- Atomicity of the sequence protected by the same monitor

The atomicity statement applies to the **protected critical section**, not to arbitrary operations elsewhere in the program.

---

# 4. Quick Comparison ⭐⭐⭐⭐⭐

| Feature | `volatile` | `synchronized` |
|---|---|---|
| Visibility | ✅ | ✅ |
| Ordering / memory semantics | ✅ | ✅ |
| Mutual exclusion | ❌ | ✅ |
| Makes `count++` atomic | ❌ | ✅ if protected by same lock |
| Protects multiple related fields | Usually ❌ | ✅ |
| Uses monitor lock | ❌ | ✅ |
| Simple shutdown flag | ✅ Excellent fit | Possible but often unnecessary |
| Critical section | ❌ | ✅ |
| Check-then-act | ❌ alone | ✅ when properly protected |
| Complex invariants | Usually ❌ | ✅ |

---

# 5. Practice Code — `volatile` Shutdown Flag ⭐⭐⭐⭐⭐

```java
public class VolatileShutdownDemo {

    private volatile boolean running = true;

    public void run() {
        while (running) {
            // Work
        }

        System.out.println("Worker stopped");
    }

    public void stop() {
        running = false;
    }

    public static void main(String[] args)
            throws InterruptedException {

        VolatileShutdownDemo demo =
                new VolatileShutdownDemo();

        Thread worker = new Thread(demo::run);
        worker.start();

        Thread.sleep(100);
        demo.stop();

        worker.join();
    }
}
```

### Why `volatile` is suitable

The requirement is simply:

```text
Thread A → publish stop signal
Thread B → observe stop signal
```

There is no compound critical section to protect.

---

# 6. Practice Code — `synchronized` Counter ⭐⭐⭐⭐⭐

```java
public class SynchronizedCounterDemo {

    private int count;

    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }

    public static void main(String[] args)
            throws InterruptedException {

        SynchronizedCounterDemo demo =
                new SynchronizedCounterDemo();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                demo.increment();
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                demo.increment();
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println("Count = " + demo.getCount());
    }
}
```

Here `count++` executes while holding the object's monitor, so concurrent increments on the same object are serialized.

---

# 7. Why `volatile count++` Is Not Equivalent ⭐⭐⭐⭐⭐

This is the most common interview trap.

```java
private volatile int count;

public void increment() {
    count++;
}
```

`count++` is conceptually:

```text
READ count
   ↓
ADD 1
   ↓
WRITE count
```

Two threads can interleave:

```text
Thread A             Thread B
--------             --------
read 10              read 10
add 1                add 1
write 11             write 11
```

Expected:

```text
12
```

Actual:

```text
11
```

`volatile` does not turn this read-modify-write sequence into one atomic operation.

---

# 8. Practice Code — Compare the Two Counters ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicInteger;

public class VolatileVsSynchronizedCounter {

    private volatile int volatileCount;
    private int synchronizedCount;
    private final AtomicInteger atomicCount = new AtomicInteger();

    public void incrementVolatile() {
        volatileCount++;
    }

    public synchronized void incrementSynchronized() {
        synchronizedCount++;
    }

    public void incrementAtomic() {
        atomicCount.incrementAndGet();
    }

    public int getVolatileCount() {
        return volatileCount;
    }

    public synchronized int getSynchronizedCount() {
        return synchronizedCount;
    }

    public int getAtomicCount() {
        return atomicCount.get();
    }
}
```

Use this class to experiment with three different concurrency mechanisms.

---

# 9. Visibility vs Mutual Exclusion ⭐⭐⭐⭐⭐

### Visibility

Question:

> Can another thread reliably observe the updated state?

Typical tools:

```text
volatile
synchronized
Lock
Atomic classes
higher-level concurrency utilities
```

### Mutual exclusion

Question:

> Can two threads execute this critical section concurrently?

`synchronized` provides mutual exclusion for the same monitor.

`volatile` does not.

---

# 10. Practice Code — Check-Then-Act ⭐⭐⭐⭐⭐

### Unsafe with `volatile`

```java
public class VolatileCheckThenAct {

    private volatile boolean available = true;

    public void acquire() {
        if (available) {
            available = false;
            System.out.println(
                    Thread.currentThread().getName()
                            + " acquired resource");
        }
    }
}
```

Two threads can both pass the check before either update becomes the application's complete state transition.

### Safe with `synchronized`

```java
public class SynchronizedCheckThenAct {

    private boolean available = true;

    public synchronized void acquire() {
        if (available) {
            available = false;
            System.out.println(
                    Thread.currentThread().getName()
                            + " acquired resource");
        }
    }
}
```

The check and update execute as one protected critical section on the same object monitor.

---

# 11. Multiple Fields / Invariants ⭐⭐⭐⭐⭐

Suppose a bank account has:

```java
private int balance;
private int transactionCount;
```

Business rule:

```text
balance and transactionCount must always remain consistent
```

A single volatile field cannot protect the relationship between these two fields.

A synchronized critical section can:

```java
synchronized (this) {
    balance -= amount;
    transactionCount++;
}
```

This protects the invariant as one operation.

---

# 12. Practice Code — Protecting an Invariant ⭐⭐⭐⭐⭐

```java
public class BankAccount {

    private int balance = 10_000;
    private int transactionCount;

    public synchronized boolean withdraw(int amount) {
        if (amount <= 0 || balance < amount) {
            return false;
        }

        balance -= amount;
        transactionCount++;
        return true;
    }

    public synchronized int getBalance() {
        return balance;
    }

    public synchronized int getTransactionCount() {
        return transactionCount;
    }
}
```

The validation, balance update and transaction-count update are protected together.

---

# 13. Lock Scope Matters ⭐⭐⭐⭐⭐

This is correct only if all participating threads use the **same lock**:

```java
synchronized (lock) {
    // protected state
}
```

Another thread synchronizing on a different object does not coordinate with the first thread.

```java
Object lock1 = new Object();
Object lock2 = new Object();
```

These are two different monitors.

### Interview rule

> **Mutual exclusion is only meaningful when competing threads synchronize on the same lock/monitor.**

---

# 14. Practice Code — Same Lock vs Different Lock

```java
public class SameLockDemo {

    private final Object lock = new Object();
    private int value;

    public void write() {
        synchronized (lock) {
            value = 42;
        }
    }

    public void read() {
        synchronized (lock) {
            System.out.println(value);
        }
    }
}
```

The same `lock` object is used for coordination.

---

# 15. Performance: `volatile` vs `synchronized` ⭐⭐⭐⭐

Do not give the simplistic interview answer:

> "volatile is always faster than synchronized."

That is too broad.

Actual performance depends on:

- JVM implementation
- CPU architecture
- contention
- critical-section size
- number of threads
- workload
- optimization/JIT behavior
- alternative concurrency primitives

The correct decision should be based first on **correctness and required semantics**, then performance should be measured.

---

# 16. Does `synchronized` Always Mean Slow? ⭐⭐⭐⭐

No.

Modern JVMs optimize synchronization extensively, and uncontended synchronization can be inexpensive.

Do not avoid `synchronized` simply because of an outdated rule that locks are always slow.

### Better interview answer

> "I choose the synchronization primitive based on the concurrency semantics I need, then benchmark if performance is a concern."

---

# 17. `volatile` Does Not Block ⭐⭐⭐⭐

A volatile read/write does not acquire a monitor lock.

Therefore:

```java
volatile boolean running;
```

can be useful for lightweight state communication.

But if you need:

```text
check
  ↓
modify
  ↓
validate
  ↓
commit
```

as one indivisible operation, volatile alone is insufficient.

---

# 18. `synchronized` Blocks Concurrent Entry ⭐⭐⭐⭐⭐

For the same monitor:

```java
synchronized (lock) {
    criticalSection();
}
```

only one thread at a time can own that monitor and execute the protected section.

Other threads attempting to acquire the same monitor must wait until it becomes available.

---

# 19. `synchronized` Also Provides Visibility ⭐⭐⭐⭐⭐

Suppose Thread A does:

```java
synchronized (lock) {
    value = 42;
}
```

and Thread B subsequently acquires the same monitor:

```java
synchronized (lock) {
    System.out.println(value);
}
```

The monitor synchronization establishes the relevant happens-before relationship:

```text
Thread A
write value
   ↓
unlock
   ↓ happens-before
lock
   ↓
Thread B
read value
```

So `synchronized` is not merely a "one thread at a time" mechanism.

---

# 20. `volatile` and Ordering ⭐⭐⭐⭐⭐

Volatile accesses have specific ordering semantics under the Java Memory Model.

For example:

```java
int data = 42;
ready = true; // volatile
```

A thread that subsequently observes the volatile write through a volatile read can rely on the relevant ordering/publication guarantees.

Do not explain volatile only as:

> "It forces data directly into RAM."

That is an oversimplification.

---

# 21. Practice Code — Publication: `volatile` vs Lock ⭐⭐⭐⭐⭐

### Volatile publication

```java
class VolatilePublisher {

    private int data;
    private volatile boolean ready;

    public void publish() {
        data = 100;
        ready = true;
    }

    public void consume() {
        if (ready) {
            System.out.println(data);
        }
    }
}
```

### Synchronized publication

```java
class SynchronizedPublisher {

    private int data;
    private boolean ready;

    public synchronized void publish() {
        data = 100;
        ready = true;
    }

    public synchronized void consume() {
        if (ready) {
            System.out.println(data);
        }
    }
}
```

Both provide synchronization semantics, but they use different mechanisms and have different structural implications.

---

# 22. When `volatile` Is the Better Fit ⭐⭐⭐⭐⭐

Use/consider `volatile` when:

- Shared state is simple.
- Threads mainly publish/observe a state change.
- No critical section is required.
- No compound read-modify-write operation is required.
- No multi-field invariant must be maintained.

Typical examples:

```java
private volatile boolean shutdown;
private volatile boolean initialized;
private volatile Config currentConfig;
```

provided the surrounding protocol is designed correctly.

---

# 23. When `synchronized` Is the Better Fit ⭐⭐⭐⭐⭐

Use/consider `synchronized` when:

- Multiple operations must be atomic as a unit.
- A critical section exists.
- Multiple fields form one invariant.
- Check-then-act must be protected.
- Shared mutable state needs mutual exclusion.

Examples:

```java
if (balance >= amount) {
    balance -= amount;
}
```

or:

```java
if (!initialized) {
    initialize();
    initialized = true;
}
```

when multiple threads can perform these operations concurrently.

---

# 24. When Neither Alone Is the Best Answer ⭐⭐⭐⭐⭐

Sometimes the right tool is neither a plain volatile field nor a synchronized method.

Examples:

### Atomic single-variable transition

```java
AtomicInteger
AtomicBoolean
AtomicReference
```

### Many concurrent tasks

```java
ExecutorService
```

### Waiting/signaling

```java
CountDownLatch
Semaphore
BlockingQueue
```

### Explicit locking policy

```java
Lock
ReentrantLock
ReadWriteLock
```

Choose the highest-level abstraction that naturally expresses the requirement.

---

# 25. `volatile` vs `synchronized` Decision Tree ⭐⭐⭐⭐⭐

```text
Do multiple threads share state?
          |
         Yes
          |
Do you only need visibility/order of a simple state?
       /       \
     Yes        No
      |          |
   volatile   Do you need atomicity / mutual exclusion?
                 |
                Yes
                 |
        synchronized / Lock / Atomic*
```

Then refine the choice based on the exact operation and contention profile.

---

# 26. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1
**"volatile makes a variable thread-safe."**

❌ Incomplete. It provides volatile memory semantics, not arbitrary thread safety.

### Trap 2
**"volatile makes `count++` atomic."**

❌ False.

### Trap 3
**"synchronized is only for mutual exclusion."**

❌ Incomplete. It also establishes memory synchronization/visibility and ordering guarantees.

### Trap 4
**"volatile is always faster."**

❌ Cannot be claimed universally.

### Trap 5
**"Any synchronized blocks protect each other."**

❌ Only if they synchronize on the same monitor/lock.

### Trap 6
**"synchronized means the entire method is always locked globally."**

❌ Instance synchronized methods lock the instance; static synchronized methods lock the `Class` object.

### Trap 7
**"Use volatile for every shared variable."**

❌ Wrong. Choose the mechanism according to the required concurrency semantics.

---

# 27. Practice Question — Which One?

### Scenario

A worker thread should stop when another thread sets a shutdown flag.

```java
private volatile boolean shutdown;
```

### Answer

`volatile` is a strong candidate because this is simple state publication and no compound critical section is required.

---

# 28. Practice Question — Which One?

### Scenario

Two threads update:

```java
balance
transactionCount
```

and both must remain consistent.

### Answer

Use synchronization/locking or another atomic design that protects the complete invariant. A single volatile field is not enough to coordinate the multi-field transaction.

---

# 29. Practice Question — Which One?

### Scenario

```java
volatile int count;
count++;
```

### Answer

Neither "volatile alone" nor a plain read/write protocol makes the increment atomic. Use `AtomicInteger` or protect the increment with synchronization/locking.

---

# 30. Practice Question — Explain the Difference in One Sentence

> **`volatile` provides visibility and ordering semantics for a shared field; `synchronized` provides mutual exclusion plus visibility and ordering for a protected critical section.**

Memorize this sentence for interviews.

---

# 31. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is the main difference between volatile and synchronized?

`volatile` provides volatile memory semantics for a field; `synchronized` provides monitor-based mutual exclusion plus memory synchronization for a critical section.

### Q2. Does volatile provide mutual exclusion?

No.

### Q3. Does synchronized provide visibility?

Yes. Monitor synchronization establishes the required happens-before relationships.

### Q4. Is `volatile int count; count++` thread-safe?

No. `count++` is a compound read-modify-write operation.

### Q5. When would you choose volatile?

For simple shared-state visibility/order requirements, such as shutdown flags or suitable publication patterns, when no mutual exclusion or compound atomicity is required.

### Q6. When would you choose synchronized?

When multiple operations need to execute atomically, a critical section needs protection, or multiple shared fields form one invariant.

### Q7. Is synchronized always slower than volatile?

No universal claim is correct. Performance depends on workload, contention, JVM and hardware. Correctness comes first; benchmark when necessary.

### Q8. Does volatile prevent all reordering?

Do not describe it that way. Volatile accesses have specific JMM ordering semantics that constrain reorderings as required by the memory model.

### Q9. Can volatile protect a mutable object?

No, not by itself. A volatile reference only gives volatile semantics to accesses to the reference.

### Q10. Can two synchronized blocks run simultaneously?

Yes, if they synchronize on different monitors. Threads contending for the same monitor are mutually exclusive.

### Q11. Is synchronized method lock object-specific?

An instance synchronized method locks the receiver object. A static synchronized method locks the corresponding `Class` object.

### Q12. What if I need atomic update of one variable but don't want a lock?

Consider the appropriate `Atomic*` class and CAS-based operations.

---

# 32. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"The main difference between volatile and synchronized is the level of concurrency guarantee they provide. Volatile gives a field volatile memory semantics, so writes and subsequent reads of the same variable have the required visibility and ordering guarantees under the Java Memory Model. It does not provide mutual exclusion and does not make compound operations like `count++` atomic. Synchronized uses a monitor and provides mutual exclusion for the protected critical section, along with the required memory visibility and ordering guarantees. So I would use volatile for simple shared-state communication such as a shutdown flag, while I would use synchronized when I need to protect a critical section, a check-then-act operation, or an invariant involving multiple fields. If I need an atomic update of one variable, I would also consider an Atomic class such as `AtomicInteger`. I would choose based on correctness first and benchmark if performance is a concern."**

---

# 33. Quick Revision ⭐⭐⭐⭐⭐

```text
                volatile             synchronized
                --------             ------------
Visibility          ✅                     ✅
Ordering            ✅                     ✅
Mutual exclusion    ❌                     ✅
count++ atomic      ❌                     ✅ if protected
Critical section    ❌                     ✅
Multi-field         ❌ alone               ✅
invariant
Monitor lock        ❌                     ✅

Simple flag
→ volatile

Critical section
→ synchronized / Lock

Atomic single value
→ Atomic*
```

### Golden Rule

> **`volatile` = visibility/order of shared state. `synchronized` = mutual exclusion + visibility/order for a protected critical section.**

---

# 34. Practice Checklist

- [x] Core difference
- [x] Volatile visibility
- [x] Volatile happens-before
- [x] Synchronized mutual exclusion
- [x] Synchronized visibility
- [x] `volatile count++` trap
- [x] Check-then-act
- [x] Multi-field invariants
- [x] Same lock requirement
- [x] Performance misconceptions
- [x] Safe publication comparison
- [x] Decision tree
- [x] Atomic classes as alternatives
- [x] Common interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.21 — `volatile` Fundamentals](../21-Volatile-Fundamentals/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.23 — `wait()`**