# 7.12 — Critical Section

## 🎯 Objective

Understand what a **critical section** is, why it needs protection in multithreaded programs, how to identify it, and how `synchronized`, locks and atomic operations can protect shared state.

---

## 1. What is a Critical Section? ⭐⭐⭐⭐⭐

A **critical section** is a section of code that accesses shared state or a shared resource and therefore must be executed with the required concurrency control to preserve correctness.

Example:

```java
counter++;
```

If multiple threads execute this against the same shared counter, the update is a critical operation that needs appropriate protection.

### Interview definition

> A critical section is the part of a program where shared mutable state or a shared resource is accessed or modified and which must be appropriately coordinated so concurrent execution does not violate correctness.

---

## 2. Simple Mental Model ⭐⭐⭐⭐⭐

```text
Multiple Threads
       ↓
Shared Resource
       ↓
Critical Section
       ↓
Concurrency Control
       ↓
Correct Result
```

Think:

> **Critical section = sensitive shared-resource code.**

---

## 3. Example of a Critical Section

```java
private int balance;

void withdraw(int amount) {
    if (balance >= amount) {
        balance -= amount;
    }
}
```

The logical operation:

```text
check balance
     ↓
subtract amount
```

is a critical section if multiple threads can modify the same account concurrently.

The check and update may need to be protected **as one logical operation**.

---

## 4. Why Do We Need a Critical Section? ⭐⭐⭐⭐⭐

Without protection, threads can interleave operations in an unsafe way.

Example:

```text
Initial balance = 1000

Thread A checks → 1000 >= 800 ✓
Thread B checks → 1000 >= 800 ✓
Thread A withdraws 800
Thread B withdraws 800
```

If both operations are allowed without proper coordination, the account invariant can be violated.

The exact manifestation depends on the implementation, but the fundamental problem is an unsafe concurrent check-and-update.

---

## 5. Critical Section vs Entire Method ⭐⭐⭐⭐⭐

A critical section is **not automatically the entire method**.

Example:

```java
void process() {
    doExpensiveCalculation();

    synchronized (lock) {
        updateSharedState();
    }

    logResult();
}
```

Only the code that actually requires protection may need to be inside the critical section.

### Important

Do not synchronize large amounts of unrelated work without a reason.

---

# 6. Practice Code — Unprotected Critical Section ⭐⭐⭐⭐⭐

```java
public class CriticalSectionUnsafePractice {

    private static int counter = 0;

    public static void main(String[] args)
            throws InterruptedException {

        Thread first = new Thread(() -> increment());
        Thread second = new Thread(() -> increment());

        first.start();
        second.start();

        first.join();
        second.join();

        System.out.println("Expected: 200000");
        System.out.println("Actual:   " + counter);
    }

    private static void increment() {
        for (int i = 0; i < 100_000; i++) {
            counter++;
        }
    }
}
```

Here:

```java
counter++;
```

is the sensitive operation accessing shared mutable state.

---

# 7. Protect the Critical Section with `synchronized` ⭐⭐⭐⭐⭐

```java
public class CriticalSectionSynchronizedPractice {

    private static int counter = 0;
    private static final Object LOCK = new Object();

    public static void main(String[] args)
            throws InterruptedException {

        Thread first = new Thread(() -> increment());
        Thread second = new Thread(() -> increment());

        first.start();
        second.start();

        first.join();
        second.join();

        System.out.println("Expected: 200000");
        System.out.println("Actual:   " + counter);
    }

    private static void increment() {
        for (int i = 0; i < 100_000; i++) {
            synchronized (LOCK) {
                counter++;
            }
        }
    }
}
```

Only one thread at a time can execute the protected update while holding the same monitor.

---

# 8. Critical Section and Mutual Exclusion ⭐⭐⭐⭐⭐

When a critical section is protected by an intrinsic monitor:

```java
synchronized (LOCK) {
    counter++;
}
```

multiple threads cannot simultaneously execute that synchronized block using the same `LOCK` monitor.

Conceptually:

```text
Thread A → acquire LOCK → critical section → release LOCK
Thread B → waits for LOCK → critical section → release LOCK
```

This is **mutual exclusion**.

---

## 9. Mutual Exclusion Does Not Mean Every Code Line Is Protected

Only code protected by the same synchronization mechanism participates in that mutual exclusion.

Example:

```java
synchronized (LOCK) {
    counter++;
}
```

Another thread doing:

```java
counter++;
```

outside that protection is not automatically synchronized with the first operation.

### Golden rule

> Threads must use a compatible/common synchronization protocol to coordinate access to the same shared state.

---

# 10. Practice Code — Narrow vs Wide Critical Section ⭐⭐⭐⭐⭐

### Wide critical section

```java
synchronized (LOCK) {
    doExpensiveCalculation();
    counter++;
    saveResult();
}
```

### Narrow critical section

```java
doExpensiveCalculation();

synchronized (LOCK) {
    counter++;
}

saveResult();
```

The second design can allow more concurrency if the surrounding operations do not require the lock.

### Important

Narrowing a critical section is a performance/design optimization only when it remains correct. Do not move operations outside the protected region if they are part of the shared-state invariant.

---

# 11. Critical Section and `synchronized` Method ⭐⭐⭐⭐⭐

```java
public synchronized void increment() {
    counter++;
}
```

For an instance method, the intrinsic lock is associated with the current object (`this`).

For a static synchronized method:

```java
public static synchronized void increment() {
    counter++;
}
```

the lock is associated with the class object.

Conceptually:

```text
instance synchronized → this
static synchronized   → Class object
```

---

# 12. Practice Code — Synchronized Method

```java
public class SynchronizedMethodPractice {

    private int counter;

    public synchronized void increment() {
        counter++;
    }

    public int getCounter() {
        return counter;
    }

    public static void main(String[] args)
            throws InterruptedException {

        SynchronizedMethodPractice object =
                new SynchronizedMethodPractice();

        Thread first = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                object.increment();
            }
        });

        Thread second = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                object.increment();
            }
        });

        first.start();
        second.start();

        first.join();
        second.join();

        System.out.println(object.getCounter());
    }
}
```

Expected:

```text
200000
```

---

# 13. Critical Section and `AtomicInteger` ⭐⭐⭐⭐⭐

For a simple counter, an atomic variable can avoid an explicit synchronized block:

```java
import java.util.concurrent.atomic.AtomicInteger;

AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();
```

Here the atomic increment operation provides the required atomicity for that counter update.

### Important

Do not conclude that `AtomicInteger` automatically solves every critical-section problem. If several operations together form one invariant, you may need a larger synchronization/locking strategy.

---

# 14. Practice Code — Atomic Counter

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCriticalSectionPractice {

    private static final AtomicInteger counter =
            new AtomicInteger();

    public static void main(String[] args)
            throws InterruptedException {

        Thread first = new Thread(() -> increment());
        Thread second = new Thread(() -> increment());

        first.start();
        second.start();

        first.join();
        second.join();

        System.out.println("Expected: 200000");
        System.out.println("Actual:   " + counter.get());
    }

    private static void increment() {
        for (int i = 0; i < 100_000; i++) {
            counter.incrementAndGet();
        }
    }
}
```

---

# 15. Critical Section and `Lock` ⭐⭐⭐⭐

A `Lock` can also protect a critical section.

```java
lock.lock();
try {
    counter++;
} finally {
    lock.unlock();
}
```

The `finally` block is important so that the lock is released even if an exception occurs.

Example:

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class LockCriticalSectionPractice {

    private static int counter;
    private static final Lock LOCK = new ReentrantLock();

    private static void increment() {
        for (int i = 0; i < 100_000; i++) {
            LOCK.lock();
            try {
                counter++;
            } finally {
                LOCK.unlock();
            }
        }
    }
}
```

---

# 16. Critical Section and Shared Resource

Critical sections are not limited to integer counters.

Examples:

```text
Bank account balance
Inventory quantity
Seat reservation
Shared queue
Order state
Cache mutation
File/resource access
Shared object state
```

The key question is:

> **Can two threads access this state concurrently in a way that can violate correctness?**

If yes, identify the critical section and choose appropriate coordination.

---

# 17. Check-Then-Act Critical Section ⭐⭐⭐⭐⭐

Consider:

```java
if (inventory > 0) {
    inventory--;
}
```

The critical section is not merely:

```java
inventory--;
```

The complete logical operation is:

```text
check inventory > 0
        ↓
      update
```

Both steps may need to be protected together.

### Correct protection

```java
synchronized (LOCK) {
    if (inventory > 0) {
        inventory--;
    }
}
```

---

# 18. Practice Code — Inventory Reservation ⭐⭐⭐⭐⭐

```java
public class InventoryPractice {

    private static int inventory = 1;
    private static final Object LOCK = new Object();

    public static void main(String[] args)
            throws InterruptedException {

        Thread first = new Thread(() -> reserve("Customer-1"));
        Thread second = new Thread(() -> reserve("Customer-2"));

        first.start();
        second.start();

        first.join();
        second.join();

        System.out.println("Remaining inventory: " + inventory);
    }

    private static void reserve(String customer) {
        synchronized (LOCK) {
            if (inventory > 0) {
                inventory--;
                System.out.println(customer + " reserved item");
            } else {
                System.out.println(customer + " failed");
            }
        }
    }
}
```

Only one reservation can pass the check and update at a time.

---

# 19. Critical Section vs Race Condition ⭐⭐⭐⭐⭐

These are not the same thing.

### Critical section

A **region of code/resource access that requires concurrency protection**.

### Race condition

A **concurrency bug/problem caused when correctness depends on unsafe timing/interleaving**.

Think:

```text
Critical section → area that needs protection
Race condition   → problem caused by unsafe concurrent access
```

---

# 20. Critical Section vs Thread Safety

```text
Critical section → specific sensitive code/resource access
Thread safety     → overall property that concurrent use remains correct
```

A thread-safe class may internally contain multiple critical sections protected by different mechanisms.

---

# 21. Does Every Shared Read Need a Critical Section? ⭐⭐⭐⭐

Not necessarily.

The answer depends on:

- whether the data can change
- whether visibility is guaranteed
- whether the read participates in a larger invariant
- the chosen synchronization protocol
- whether the object is immutable
- whether an atomic/concurrent data structure is being used

A read of immutable state is very different from reading a mutable value concurrently with writes.

Do not use a blanket rule such as:

> "Every shared variable must always be synchronized."

Analyze the memory-model and correctness requirements.

---

# 22. Critical Section and `volatile` ⭐⭐⭐⭐⭐

`volatile` can provide visibility/order guarantees for a volatile variable, but it does not create mutual exclusion for a compound critical section.

Example:

```java
volatile int counter;

counter++;
```

`counter++` is still a compound read-modify-write operation and is not made atomic by `volatile`.

For a simple counter, consider:

```java
AtomicInteger
```

For a multi-step invariant, use an appropriate locking/synchronization strategy.

---

# 23. Practice Exercise — Identify the Critical Section

```java
void transfer(Account from, Account to, int amount) {
    validate(amount);
    from.debit(amount);
    to.credit(amount);
    logTransfer();
}
```

### Question

Which part may belong to the critical section?

### Answer

It depends on the account/transaction design, but the shared-state updates that must preserve the transfer invariant need to be coordinated as one logical operation.

The exact lock strategy is a design decision and may involve ordering locks, a transaction, or another concurrency mechanism.

---

# 24. Practice Exercise — Why Not Lock Everything?

### Question

Why not simply synchronize the entire application?

### Answer

Because excessive locking can:

- reduce concurrency
- increase contention
- reduce throughput
- increase latency
- make deadlocks easier to introduce if multiple locks are involved
- make the design harder to reason about

The goal is **correct synchronization with appropriate scope**, not maximum synchronization.

---

# 25. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — Critical section means whole method

❌ Not necessarily.

Only the shared-resource-sensitive portion may require protection.

### Mistake 2 — `counter++` is automatically safe

❌ False.

It is a compound operation.

### Mistake 3 — `volatile` creates mutual exclusion

❌ False.

### Mistake 4 — `yield()` protects a critical section

❌ False.

### Mistake 5 — `sleep()` protects a critical section

❌ False.

### Mistake 6 — Locking one code path protects all accesses

❌ False.

All relevant accesses need to follow a compatible synchronization protocol.

### Mistake 7 — Bigger critical section is always safer

❌ Not necessarily.

It may be correct but unnecessarily reduce concurrency.

### Mistake 8 — Smaller critical section is always better

❌ Not if moving code out breaks the invariant.

Correctness comes first.

---

# 26. Interview Questions

### Q1. What is a critical section?

A code region that accesses/modifies shared state or a shared resource and requires appropriate coordination to preserve correctness under concurrent execution.

### Q2. How do you protect a critical section in Java?

Depending on the use case, `synchronized`, `Lock`, atomic classes, concurrent data structures, immutability, confinement, transactions, or higher-level concurrency utilities.

### Q3. Is a critical section always an entire method?

No. It can be a small block inside a method.

### Q4. What is mutual exclusion?

A property where only one thread at a time can execute a protected critical section for the same synchronization resource.

### Q5. Does `volatile` provide mutual exclusion?

No.

### Q6. Does `yield()` protect a critical section?

No.

### Q7. Does `sleep()` protect a critical section?

No.

### Q8. What is the difference between critical section and race condition?

A critical section is the sensitive code/resource access that needs coordination; a race condition is the correctness problem that can occur when concurrent access is not safely coordinated.

### Q9. Why keep critical sections small?

To reduce lock contention and improve concurrency while preserving correctness.

### Q10. What if two different locks protect the same shared variable?

That does not necessarily provide mutual exclusion between those accesses. Threads need a compatible synchronization protocol.

### Q11. Can `AtomicInteger` replace every critical section?

No. It is useful for specific atomic operations, but multi-step invariants may require locking or another higher-level mechanism.

### Q12. What is a logical critical section?

A set of operations that must be treated as one atomic unit to preserve a business/data invariant, even if it contains multiple individual statements.

---

# 27. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"A critical section is the part of a program where shared mutable state or a shared resource is accessed or modified and therefore requires appropriate concurrency control. For example, `counter++` is a critical operation when multiple threads update the same counter. The important point is that the critical section is not necessarily the whole method; we should protect the smallest region that is required for correctness. In Java, we can use `synchronized`, `Lock`, atomic classes, concurrent data structures, immutability or higher-level concurrency utilities depending on the problem. Synchronization provides mutual exclusion for the protected region and also important memory-visibility guarantees. We should avoid unnecessarily large critical sections because they increase contention, but we must not make them too small if that breaks a multi-step invariant such as check-then-act. In short, identify the shared resource, identify the complete logical operation that must be protected, and choose the appropriate concurrency mechanism."**

---

# 28. Quick Revision

```text
Critical Section
      ↓
Shared mutable state/resource
      ↓
Sensitive concurrent operation
      ↓
Needs appropriate coordination
      ↓
Correctness + safe concurrency
```

### Golden Rules ⭐⭐⭐⭐⭐

```text
Critical section ≠ necessarily whole method
Critical section → protect shared-resource-sensitive code
synchronized → mutual exclusion + memory visibility
volatile → visibility/order, NOT mutual exclusion
AtomicInteger → useful for atomic counter operations
yield() → NOT synchronization
sleep() → NOT synchronization
join() → NOT a critical-section protection mechanism
```

### Memory Trick

> **Find the shared state → find the complete logical operation → protect that operation.**

---

# 29. Completion Checklist

- [x] Critical section definition
- [x] Shared mutable state
- [x] Mutual exclusion
- [x] Critical section vs whole method
- [x] `synchronized` block
- [x] synchronized method
- [x] `AtomicInteger`
- [x] `Lock` / `ReentrantLock`
- [x] Check-then-act
- [x] Inventory practice
- [x] Narrow vs wide critical section
- [x] `volatile` limitation
- [x] Race condition comparison
- [x] Thread-safety comparison
- [x] Practice code
- [x] Interview questions
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.11 — Race Condition](../11-Race-Condition/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.13 — `synchronized` Method**