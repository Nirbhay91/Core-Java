# 7.18 — Thread Safety

## 🎯 Objective

Understand what **thread safety** means, why shared mutable state causes problems, how race conditions break correctness, and the main Java techniques used to make code thread-safe.

---

## 1. What is Thread Safety? ⭐⭐⭐⭐⭐

A piece of code is **thread-safe** when it behaves correctly even when multiple threads execute it concurrently, according to its intended contract.

> **Thread safety is about correctness under concurrent access.**

Thread-safe code must correctly handle shared state when multiple threads can access or modify it at the same time.

---

# 2. Shared Mutable State ⭐⭐⭐⭐⭐

The most common source of thread-safety problems is:

```text
Shared state
    +
Mutable state
    +
Concurrent access
    ↓
Possible race condition
```

Example:

```java
private int count;
```

If several threads execute:

```java
count++;
```

concurrently, the result may be incorrect.

---

# 3. Why `count++` Is Not Thread-Safe ⭐⭐⭐⭐⭐

This:

```java
count++;
```

is conceptually a read-modify-write operation:

```text
1. Read count
2. Add 1
3. Write count
```

Two threads can interleave these operations:

```text
Initial count = 0

T1 reads 0
T2 reads 0
T1 writes 1
T2 writes 1

Expected = 2
Actual   = 1
```

This is a classic lost-update race.

---

# 4. Practice Code — Unsafe Counter ⭐⭐⭐⭐⭐

```java
public class UnsafeCounter {

    private int count;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args)
            throws InterruptedException {

        UnsafeCounter counter = new UnsafeCounter();

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

        System.out.println("Expected: 200000");
        System.out.println("Actual:   " + counter.getCount());
    }
}
```

The actual value can be less than the expected value because the increment operation is not atomic.

---

# 5. Thread-Safe Version Using `synchronized` ⭐⭐⭐⭐⭐

```java
public class SynchronizedCounter {

    private int count;

    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }

    public static void main(String[] args)
            throws InterruptedException {

        SynchronizedCounter counter =
                new SynchronizedCounter();

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

        System.out.println("Expected: 200000");
        System.out.println("Actual:   " + counter.getCount());
    }
}
```

The critical operation is protected by the same monitor.

---

# 6. Thread Safety Is More Than `synchronized` ⭐⭐⭐⭐⭐

`synchronized` is one mechanism for thread safety, but not the definition of thread safety.

Common approaches include:

```text
1. Immutability
2. Synchronization
3. Atomic classes
4. Concurrent collections
5. Thread confinement
6. Safe publication
7. Locks
8. Stateless design
9. Message passing / actor-style ownership
```

The correct choice depends on the shared-state design.

---

# 7. Immutable Objects ⭐⭐⭐⭐⭐

An immutable object cannot change its state after construction.

Typical properties:

- State is initialized during construction.
- No setters that mutate state.
- Mutable internal state is not exposed.
- Fields are appropriately encapsulated.
- The object is safely constructed/publicly exposed.

Because state does not change, concurrent readers do not need synchronization merely to protect mutations that cannot happen.

---

# 8. Practice Code — Immutable Object

```java
public final class User {

    private final String name;
    private final int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }
}
```

There are no methods that mutate `name` or `age` after construction.

---

# 9. Atomic Classes ⭐⭐⭐⭐⭐

For simple atomic state transitions, Java provides classes such as:

```java
AtomicInteger
AtomicLong
AtomicBoolean
```

Example:

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```

This can avoid explicit synchronization for suitable atomic operations.

> **Atomic does not automatically mean every multi-step business operation is thread-safe.**

---

# 10. Practice Code — `AtomicInteger`

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {

    private final AtomicInteger count =
            new AtomicInteger();

    public void increment() {
        count.incrementAndGet();
    }

    public int getCount() {
        return count.get();
    }

    public static void main(String[] args)
            throws InterruptedException {

        AtomicCounter counter = new AtomicCounter();

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

---

# 11. Concurrent Collections ⭐⭐⭐⭐⭐

Standard collections such as `ArrayList` and `HashMap` are not generally safe for arbitrary concurrent mutation.

Java provides concurrency-oriented collections, for example:

```java
ConcurrentHashMap
CopyOnWriteArrayList
BlockingQueue
```

These are designed for particular concurrent access patterns.

---

# 12. Practice Code — `ConcurrentHashMap`

```java
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentMapPractice {

    public static void main(String[] args)
            throws InterruptedException {

        ConcurrentHashMap<String, Integer> map =
                new ConcurrentHashMap<>();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 1_000; i++) {
                map.merge("Java", 1, Integer::sum);
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 1_000; i++) {
                map.merge("Java", 1, Integer::sum);
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(map.get("Java"));
    }
}
```

---

# 13. Thread Confinement ⭐⭐⭐⭐

A simple way to avoid shared-state races is to ensure state belongs to one thread.

```text
Thread A → private state
Thread B → private state
```

If no other thread can access the mutable state, synchronization may not be needed for that state.

Examples include properly used local variables and thread-confined objects.

---

# 14. Stateless Design ⭐⭐⭐⭐

A stateless component does not maintain mutable request-specific shared state.

Example:

```java
public class Calculator {

    public int add(int a, int b) {
        return a + b;
    }
}
```

Each call operates only on local parameters.

Such designs are naturally easier to use concurrently.

---

# 15. Safe Publication ⭐⭐⭐⭐⭐

Thread safety also depends on how an object becomes visible to other threads.

An object may be internally correct but still have unsafe visibility if it is improperly published.

Common mechanisms that establish useful visibility/order guarantees include:

- `final` field initialization semantics for properly constructed objects
- `volatile`
- `synchronized`
- locks
- concurrent collections
- static initialization
- executor/concurrency framework synchronization mechanisms

> **Thread safety includes visibility and ordering, not just mutual exclusion.**

---

# 16. `volatile` and Thread Safety ⭐⭐⭐⭐⭐

`volatile` provides visibility and ordering guarantees for the volatile variable, but it does not make compound operations such as:

```java
count++
```

atomic.

So this is not a correct general-purpose counter:

```java
private volatile int count;

count++;
```

### Interview point

> **`volatile` can make a state change visible; it does not turn arbitrary compound operations into atomic operations.**

---

# 17. Practice Code — `volatile` Flag

```java
public class VolatileFlagPractice {

    private volatile boolean running = true;

    public void stop() {
        running = false;
    }

    public void work() {
        while (running) {
            // perform work
        }

        System.out.println("Worker stopped");
    }

    public static void main(String[] args)
            throws InterruptedException {

        VolatileFlagPractice worker =
                new VolatileFlagPractice();

        Thread thread = new Thread(worker::work);
        thread.start();

        Thread.sleep(100);
        worker.stop();

        thread.join();
    }
}
```

This is an example where visibility of a simple state flag is the important requirement.

---

# 18. Atomicity vs Thread Safety ⭐⭐⭐⭐⭐

Do not use these terms interchangeably.

### Atomicity

An operation appears indivisible to other threads.

### Visibility

A thread can observe another thread's relevant updates according to the Java Memory Model.

### Ordering

The Java Memory Model constrains how operations can be observed and reordered.

### Thread safety

The overall concurrent behavior remains correct according to the object's contract.

```text
Atomicity + Visibility + Ordering
          ↓
Important building blocks
          ↓
Thread-safe design
```

---

# 19. Thread-Safe vs Immutable ⭐⭐⭐⭐⭐

An immutable object is a strong way to achieve safe sharing, but the concepts are not identical.

```text
Immutable
→ state cannot change after construction

Thread-safe
→ concurrent operations remain correct
```

A mutable class can be thread-safe if access is correctly coordinated.

---

# 20. Thread-Safe vs Synchronized ⭐⭐⭐⭐⭐

These are also different concepts.

```text
Thread-safe
→ property of behavior/design

synchronized
→ one mechanism for coordination
```

A class can be thread-safe without using `synchronized`, for example with suitable atomic variables or concurrent collections.

---

# 21. Thread Safety and Compound Actions ⭐⭐⭐⭐⭐

A common mistake is assuming individually thread-safe operations make a multi-step operation thread-safe.

Example:

```java
if (!map.containsKey(key)) {
    map.put(key, value);
}
```

Even if the individual map operations were safe, the **check-then-act** sequence may not be atomic as a whole.

Prefer an atomic compound operation when the collection provides one, such as:

```java
map.putIfAbsent(key, value);
```

or use suitable synchronization/design.

---

# 22. Practice Code — Check-Then-Act Problem

```java
import java.util.concurrent.ConcurrentHashMap;

public class CheckThenActPractice {

    private final ConcurrentHashMap<String, String> map =
            new ConcurrentHashMap<>();

    public void unsafeBusinessOperation(String key) {
        if (!map.containsKey(key)) {
            // Another thread can change the map here.
            map.put(key, "created");
        }
    }

    public void saferOperation(String key) {
        map.putIfAbsent(key, "created");
    }
}
```

The second method expresses the intended atomic map operation directly.

---

# 23. Lock Granularity ⭐⭐⭐⭐

Locking too broadly can reduce concurrency.

```java
public synchronized void entireMethod() {
    // lots of unrelated work
}
```

A smaller critical section may allow more concurrency:

```java
public void method() {
    // non-critical work

    synchronized (lock) {
        // only shared-state update
    }

    // more non-critical work
}
```

But making a critical section smaller is safe only when all required invariants remain protected.

---

# 24. Practice Code — Narrow Critical Section

```java
public class NarrowCriticalSection {

    private final Object lock = new Object();
    private int count;

    public void process() {
        // Non-critical work
        String result = "data".toUpperCase();

        synchronized (lock) {
            count++;
        }

        // More non-critical work
        System.out.println(result);
    }

    public int getCount() {
        synchronized (lock) {
            return count;
        }
    }
}
```

---

# 25. Thread Safety and Invariants ⭐⭐⭐⭐⭐

A thread-safe class must protect its **invariants**, not merely individual fields.

Suppose a bank account requires:

```text
balance >= 0
```

A thread-safe withdrawal operation must protect the complete check-and-update sequence.

```text
check balance
      ↓
update balance
```

Both operations belong to one logical critical section.

---

# 26. Practice Code — Thread-Safe Bank Account

```java
public class ThreadSafeBankAccount {

    private int balance;

    public ThreadSafeBankAccount(int balance) {
        this.balance = balance;
    }

    public synchronized boolean withdraw(int amount) {
        if (amount <= 0 || amount > balance) {
            return false;
        }

        balance -= amount;
        return true;
    }

    public synchronized int getBalance() {
        return balance;
    }
}
```

The check and modification are protected by the same monitor, preserving the account invariant under concurrent withdrawals.

---

# 27. Common Thread-Safety Strategies ⭐⭐⭐⭐⭐

| Strategy | Idea | Typical Use |
|---|---|---|
| Immutability | Never mutate shared state | Value/config objects |
| `synchronized` | Mutual exclusion + visibility | Simple shared mutable state |
| `volatile` | Visibility/order for variable | Flags/state publication |
| Atomic classes | Atomic state operations | Counters, CAS-style state |
| Concurrent collections | Concurrent data structures | Shared maps/queues/lists |
| Locks | Explicit coordination | Complex locking policies |
| Thread confinement | Keep state private to one thread | Per-thread context |
| Stateless design | Avoid shared mutable state | Services/utilities |
| Message passing | Transfer ownership/data | Concurrent pipelines |

---

# 28. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1

> `count++` is one atomic operation.

❌ It is a compound read-modify-write operation.

### Mistake 2

> `volatile` makes a counter thread-safe.

❌ `volatile` does not make `count++` atomic.

### Mistake 3

> Every thread-safe class must use `synchronized`.

❌ No. Atomic classes, locks, concurrent collections, immutability and other designs can provide thread safety.

### Mistake 4

> Concurrent collection means every multi-step business operation is automatically atomic.

❌ The whole sequence may still require an atomic API or additional coordination.

### Mistake 5

> Thread safety means only preventing race conditions.

❌ Visibility, ordering and invariant preservation also matter.

### Mistake 6

> Immutable means synchronization is always required.

❌ Immutable state can generally be shared safely after proper construction/publication.

### Mistake 7

> Making every method synchronized is always the best design.

❌ It can unnecessarily serialize operations and reduce scalability.

---

# 29. Interview Questions

### Q1. What is thread safety?

The property that concurrent execution by multiple threads remains correct according to the intended contract.

### Q2. What causes most thread-safety problems?

Unsafely shared mutable state and incorrect coordination between concurrent operations.

### Q3. Is `count++` thread-safe?

No. It is a compound read-modify-write operation.

### Q4. Does `volatile` make `count++` thread-safe?

No. It provides visibility/order guarantees for the variable but not atomicity of the compound increment.

### Q5. How can you make a counter thread-safe?

For example, use `synchronized` or an appropriate atomic class such as `AtomicInteger`.

### Q6. Is `synchronized` the only way to achieve thread safety?

No.

### Q7. How does immutability help thread safety?

If state cannot change after construction, concurrent readers do not race over mutations to that state.

### Q8. What is thread confinement?

Keeping mutable state accessible to only one thread so concurrent sharing does not occur.

### Q9. Are `HashMap` and `ArrayList` thread-safe for concurrent mutation?

No, they are not designed as general-purpose concurrent collections.

### Q10. Name some concurrent collections.

`ConcurrentHashMap`, `CopyOnWriteArrayList`, and blocking queues.

### Q11. What is a check-then-act race?

A race where a condition is checked and an action is performed separately, allowing another thread to change the state between them.

### Q12. What should a thread-safe class protect?

Its shared mutable state and the invariants that must remain true across related state changes.

### Q13. Can a class be thread-safe without synchronization?

Yes.

### Q14. Why can excessive synchronization hurt performance?

It can create unnecessary contention and reduce concurrency.

### Q15. Is an immutable object automatically thread-safe?

An immutable object is generally safe to share concurrently because its state cannot change after construction, assuming it is properly constructed and does not expose mutable internals.

---

# 30. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Thread safety means that a class or operation behaves correctly when accessed concurrently by multiple threads. The biggest source of problems is shared mutable state. For example, `count++` is a read-modify-write operation, so multiple threads can lose updates. We can achieve thread safety using mechanisms such as `synchronized`, atomic classes, concurrent collections, locks, immutability, thread confinement and stateless design. An important point is that thread safety is broader than mutual exclusion: visibility, ordering and preservation of object invariants also matter. `volatile` provides visibility and ordering guarantees but does not make compound operations such as `count++` atomic. Also, even if individual operations are thread-safe, a multi-step business operation such as check-then-act may still require atomic coordination. The goal is not to synchronize everything, but to correctly protect shared state while maintaining as much concurrency as the design allows."**

---

# 31. Quick Revision ⭐⭐⭐⭐⭐

```text
Thread Safety
     ↓
Correct under concurrent access
     ↓
Protect shared mutable state
     ↓
Preserve invariants
```

### Remember

```text
count++
→ NOT atomic

volatile
→ visibility/order
→ NOT compound-operation atomicity

synchronized
→ mutual exclusion + relevant memory visibility/order

AtomicInteger
→ atomic operations

ConcurrentHashMap
→ concurrent map operations

Immutable
→ no state mutation after construction

Thread confinement
→ don't share mutable state

Stateless
→ avoid shared mutable state
```

### Golden Rule

> **The safest concurrent design is often the one that minimizes shared mutable state.**

---

# 32. Practice Checklist

- [x] Thread safety definition
- [x] Shared mutable state
- [x] Race condition
- [x] `count++` problem
- [x] `synchronized` solution
- [x] Immutability
- [x] Atomic classes
- [x] Concurrent collections
- [x] Thread confinement
- [x] Stateless design
- [x] Safe publication basics
- [x] `volatile` limitations
- [x] Atomicity vs visibility vs ordering
- [x] Check-then-act
- [x] Lock granularity
- [x] Object invariants
- [x] Practice programs
- [x] Common mistakes
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.17 — Reentrancy of `synchronized`](../17-Reentrancy-of-Synchronized/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.19 — Atomicity vs Visibility vs Ordering**