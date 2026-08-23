# 7.34 — Common Thread-Safety Strategies

## 🎯 Objective

Understand the practical strategies used to make shared mutable state safe when multiple threads access it concurrently.

> **Core idea:** Thread safety means the program maintains its correctness when multiple threads execute concurrently, not simply that it uses `synchronized`.

---

# 1. What Is Thread Safety? ⭐⭐⭐⭐⭐

A component is thread-safe when it behaves correctly under concurrent access according to its intended contract.

Example of unsafe shared state:

```java
count++;
```

This is a read-modify-write operation and is not automatically atomic.

Possible interleaving:

```text
Thread A: read 10
Thread B: read 10
Thread A: write 11
Thread B: write 11

Expected: 12
Actual:   11
```

Thread-safety strategies aim to prevent incorrect outcomes like this.

---

# 2. Main Thread-Safety Strategies ⭐⭐⭐⭐⭐

Common approaches:

1. Immutability
2. Thread confinement
3. Synchronization / intrinsic locks
4. Explicit locks
5. Atomic classes
6. Volatile variables for visibility/state flags
7. Concurrent collections
8. Safe publication
9. Immutable snapshots / copy-on-write
10. Minimize shared mutable state

No single strategy is best for every problem.

---

# 3. Strategy 1 — Immutability ⭐⭐⭐⭐⭐

An immutable object cannot change its observable state after construction.

Example:

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

If the object's state never changes, multiple threads can safely share the same instance without synchronizing mutations.

### Benefits

- Simple reasoning
- Naturally safe to share
- No locking for state changes
- Easier testing

### Requirements

Typical immutable-class design includes:

- `final` class or controlled inheritance
- `private final` fields
- no setters
- defensive copies for mutable fields
- complete initialization in the constructor

---

# 4. Practice Code — Immutable Object ⭐⭐⭐⭐⭐

```java
public final class ImmutableAccount {

    private final String accountNumber;
    private final int balance;

    public ImmutableAccount(String accountNumber, int balance) {
        this.accountNumber = accountNumber;
        this.balance = balance;
    }

    public String getAccountNumber() {
        return accountNumber;
    }

    public int getBalance() {
        return balance;
    }

    public ImmutableAccount deposit(int amount) {
        return new ImmutableAccount(
                accountNumber,
                balance + amount
        );
    }
}
```

Instead of modifying the existing object, `deposit()` returns a new object.

---

# 5. Strategy 2 — Thread Confinement ⭐⭐⭐⭐⭐

Keep mutable state owned by a single thread so other threads cannot concurrently mutate it.

Conceptually:

```text
Thread A → owns state A
Thread B → owns state B

No shared mutation → less synchronization
```

Examples include:

- local variables
- thread-local state
- single-threaded event loops

---

# 6. Practice Code — Thread Confinement with Local State ⭐⭐⭐⭐

```java
public class ThreadConfinementDemo {

    static void calculate() {
        int counter = 0;

        for (int i = 0; i < 1000; i++) {
            counter++;
        }

        System.out.println(Thread.currentThread().getName()
                + " = " + counter);
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(ThreadConfinementDemo::calculate, "T1");
        Thread t2 = new Thread(ThreadConfinementDemo::calculate, "T2");

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

Each invocation owns its own local variable.

---

# 7. Strategy 3 — `ThreadLocal` ⭐⭐⭐⭐⭐

`ThreadLocal` provides each thread with its own independent value.

```java
ThreadLocal<Integer> local = ThreadLocal.withInitial(() -> 0);
```

Each thread sees its own value:

```text
Thread A → local = 10
Thread B → local = 20
```

No synchronization is needed merely to protect those separate values from each other.

### Important

`ThreadLocal` is not a replacement for synchronization when the underlying data is actually shared.

---

# 8. Practice Code — `ThreadLocal` ⭐⭐⭐⭐⭐

```java
public class ThreadLocalDemo {

    private static final ThreadLocal<Integer> COUNTER =
            ThreadLocal.withInitial(() -> 0);

    static void work() {
        COUNTER.set(COUNTER.get() + 1);
        COUNTER.set(COUNTER.get() + 1);

        System.out.println(Thread.currentThread().getName()
                + " = " + COUNTER.get());

        COUNTER.remove();
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(ThreadLocalDemo::work, "T1");
        Thread t2 = new Thread(ThreadLocalDemo::work, "T2");

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

### Production note

When using thread pools, call `remove()` when appropriate so request/task-specific values do not accidentally remain associated with a reused worker thread.

---

# 9. Strategy 4 — `synchronized` ⭐⭐⭐⭐⭐

Use intrinsic locking when a critical section protects shared mutable state.

```java
synchronized (lock) {
    count++;
}
```

Benefits:

- mutual exclusion
- visibility guarantees associated with monitor synchronization
- reentrancy
- simple syntax

### Important

Do not synchronize unnecessarily large sections because it can reduce concurrency.

---

# 10. Practice Code — Synchronized Counter ⭐⭐⭐⭐⭐

```java
public class SynchronizedCounter {

    private int count;

    public synchronized void increment() {
        count++;
    }

    public synchronized int get() {
        return count;
    }

    public static void main(String[] args) throws Exception {
        SynchronizedCounter counter = new SynchronizedCounter();

        Runnable task = () -> {
            for (int i = 0; i < 10_000; i++) {
                counter.increment();
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(counter.get());
    }
}
```

Expected output:

```text
20000
```

---

# 11. Strategy 5 — Explicit Locks ⭐⭐⭐⭐⭐

`ReentrantLock` provides explicit lock management and additional capabilities such as:

- `tryLock()`
- timed lock acquisition
- interruptible lock acquisition
- multiple `Condition` objects

Example:

```java
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```

### Golden rule

Always release the lock in `finally`.

---

# 12. Practice Code — `ReentrantLock` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockCounter {

    private final ReentrantLock lock = new ReentrantLock();
    private int count;

    void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }

    int get() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws Exception {
        ReentrantLockCounter counter = new ReentrantLockCounter();

        Runnable task = () -> {
            for (int i = 0; i < 10_000; i++) {
                counter.increment();
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(counter.get());
    }
}
```

---

# 13. Strategy 6 — Atomic Classes ⭐⭐⭐⭐⭐

Atomic classes provide atomic operations without manually writing a synchronized block around each operation.

Common classes:

- `AtomicInteger`
- `AtomicLong`
- `AtomicBoolean`
- `AtomicReference`
- `LongAdder`
- `LongAccumulator`

Example:

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```

---

# 14. Practice Code — `AtomicInteger` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicIntegerDemo {

    private static final AtomicInteger COUNT =
            new AtomicInteger();

    static void increment() {
        for (int i = 0; i < 10_000; i++) {
            COUNT.incrementAndGet();
        }
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(AtomicIntegerDemo::increment);
        Thread t2 = new Thread(AtomicIntegerDemo::increment);

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(COUNT.get());
    }
}
```

Expected:

```text
20000
```

---

# 15. Atomic Does Not Mean Every Compound Operation Is Safe ⭐⭐⭐⭐⭐

This is safe:

```java
count.incrementAndGet();
```

But this sequence may still be logically unsafe:

```java
if (count.get() < 10) {
    count.incrementAndGet();
}
```

Another thread can change `count` between the check and increment.

If the entire check-and-update must be one atomic decision, use an appropriate atomic compound operation such as CAS or protect the complete invariant with synchronization/locking.

---

# 16. Practice Code — CAS with `compareAndSet()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicInteger;

public class CompareAndSetDemo {

    private static final AtomicInteger COUNT =
            new AtomicInteger(0);

    static void incrementIfBelowTen() {
        while (true) {
            int current = COUNT.get();

            if (current >= 10) {
                return;
            }

            if (COUNT.compareAndSet(current, current + 1)) {
                return;
            }
        }
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 20; i++) {
                incrementIfBelowTen();
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 20; i++) {
                incrementIfBelowTen();
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(COUNT.get());
    }
}
```

This demonstrates a compare-and-set loop: the update succeeds only if the value is still the value that was observed.

---

# 17. Strategy 7 — `volatile` for Visibility ⭐⭐⭐⭐⭐

Use `volatile` when threads need visibility of a variable and the operation does not require a multi-step invariant.

Classic example:

```java
private volatile boolean running = true;
```

One thread can update the flag and another thread can observe the updated value.

### But:

```java
volatile int count;
count++;
```

is **not atomic**.

`volatile` provides visibility and ordering guarantees for accesses to that variable; it does not make arbitrary compound operations atomic.

---

# 18. Practice Code — `volatile` Stop Flag ⭐⭐⭐⭐⭐

```java
public class VolatileStopFlag {

    private static volatile boolean running = true;

    public static void main(String[] args) throws Exception {

        Thread worker = new Thread(() -> {
            while (running) {
                // do work
            }

            System.out.println("Worker stopped");
        });

        worker.start();

        Thread.sleep(500);
        running = false;

        worker.join();
    }
}
```

This is a good use of `volatile`: a simple shared state flag.

---

# 19. Strategy 8 — Concurrent Collections ⭐⭐⭐⭐⭐

Use collections designed for concurrent access instead of manually synchronizing ordinary collections in every operation.

Important examples:

- `ConcurrentHashMap`
- `CopyOnWriteArrayList`
- `ConcurrentLinkedQueue`
- `BlockingQueue`
- `ConcurrentLinkedDeque`

---

# 20. Practice Code — `ConcurrentHashMap` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentHashMapDemo {

    public static void main(String[] args) throws Exception {

        ConcurrentHashMap<String, Integer> counts =
                new ConcurrentHashMap<>();

        Runnable task = () -> {
            for (int i = 0; i < 1_000; i++) {
                counts.merge("java", 1, Integer::sum);
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(counts.get("java"));
    }
}
```

`merge()` expresses the update as one map-level operation rather than a separate unsafe `get()` followed by `put()`.

---

# 21. `CopyOnWriteArrayList` ⭐⭐⭐⭐

Useful when:

```text
Reads >>> Writes
```

Every structural write creates a new underlying array snapshot.

Good examples:

- listener lists
- configuration snapshots
- mostly-read collections

Bad choice for:

- frequent writes
- very large collections with many mutations

---

# 22. Practice Code — `CopyOnWriteArrayList` ⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class CopyOnWriteDemo {

    public static void main(String[] args) throws Exception {

        CopyOnWriteArrayList<String> listeners =
                new CopyOnWriteArrayList<>();

        listeners.add("Listener-A");
        listeners.add("Listener-B");

        Thread reader = new Thread(() -> {
            for (String listener : listeners) {
                System.out.println("Calling " + listener);
            }
        });

        Thread writer = new Thread(() ->
                listeners.add("Listener-C"));

        reader.start();
        writer.start();

        reader.join();
        writer.join();

        System.out.println(listeners);
    }
}
```

---

# 23. Strategy 9 — Safe Publication ⭐⭐⭐⭐⭐

Even a correctly designed object needs to be safely published when one thread makes it available to another.

Common safe-publication mechanisms include:

- initializing through a static initializer
- storing the reference in a `volatile` field
- publishing through a lock
- publishing through thread-safe/concurrent collections
- publishing through properly synchronized handoff mechanisms
- final-field guarantees when the object is constructed correctly and `this` does not escape during construction

### Dangerous pattern

```java
class Service {
    static Service instance;

    Service() {
        instance = this;
    }
}
```

Publishing `this` during construction can expose a partially constructed object.

---

# 24. Practice Code — Safe Publication with `volatile` ⭐⭐⭐⭐

```java
public class SafePublicationDemo {

    private static volatile Config config;

    static class Config {
        final String name;

        Config(String name) {
            this.name = name;
        }
    }

    static void publish() {
        config = new Config("Production");
    }

    static void read() {
        Config local = config;

        if (local != null) {
            System.out.println(local.name);
        }
    }

    public static void main(String[] args) throws Exception {
        Thread writer = new Thread(SafePublicationDemo::publish);
        Thread reader = new Thread(SafePublicationDemo::read);

        writer.start();
        writer.join();

        reader.start();
        reader.join();
    }
}
```

---

# 25. Strategy 10 — Minimize Shared Mutable State ⭐⭐⭐⭐⭐

The simplest synchronization problem is the state you never share.

Prefer:

```text
immutable data
     ↓
local variables
     ↓
message passing / queues
     ↓
small synchronized state
```

instead of:

```text
many threads
     ↓
large shared mutable object
     ↓
many locks
     ↓
complex bugs
```

---

# 26. Strategy Comparison ⭐⭐⭐⭐⭐

| Strategy | Main Benefit | Main Limitation |
|---|---|---|
| Immutability | Simple sharing | State cannot be mutated in place |
| Thread confinement | Eliminates sharing | State cannot be freely shared |
| `ThreadLocal` | Per-thread state | Can retain stale data in pools |
| `synchronized` | Simple mutual exclusion | Contention / blocking |
| `ReentrantLock` | Flexible locking | Must manage unlock correctly |
| Atomic classes | Efficient atomic operations | Not enough for arbitrary multi-variable invariants |
| `volatile` | Visibility / ordering | Does not provide general atomicity |
| Concurrent collections | Built-in concurrent behavior | Semantics differ from ordinary collections |
| Copy-on-write | Excellent read concurrency | Expensive writes |
| Safe publication | Correct visibility of constructed state | Requires disciplined publication |

---

# 27. Choosing the Right Strategy ⭐⭐⭐⭐⭐

### Case 1 — Configuration object never changes

Use **immutability**.

### Case 2 — Per-request state inside a worker thread

Consider **thread confinement / `ThreadLocal`**, with careful cleanup for thread pools.

### Case 3 — Simple shared counter

Use **`AtomicInteger`** or synchronization depending on the required semantics.

### Case 4 — Multiple fields must change together

Use **`synchronized` / `Lock`** to protect the invariant.

### Case 5 — Producer/consumer

Use **`BlockingQueue`**.

### Case 6 — Shared concurrent map

Use **`ConcurrentHashMap`**.

### Case 7 — Simple stop flag

Use **`volatile`**.

### Case 8 — Many reads, very few writes

Consider **`CopyOnWriteArrayList`**.

---

# 28. Thread Safety vs Atomicity ⭐⭐⭐⭐⭐

These terms are related but not identical.

### Atomicity

An operation appears indivisible.

```java
atomicInteger.incrementAndGet();
```

### Visibility

A thread can observe another thread's update according to the Java Memory Model.

### Thread safety

The complete component maintains its correctness under concurrent use.

A class can use atomic variables and still be logically thread-unsafe if multiple operations must satisfy a shared invariant.

---

# 29. Example — Thread-Safe Class with an Invariant ⭐⭐⭐⭐⭐

Suppose:

```text
balance must never become negative
```

This is not necessarily enough:

```java
if (balance >= amount) {
    balance -= amount;
}
```

Two threads can both pass the check.

Protect the entire check-and-update operation:

```java
public synchronized boolean withdraw(int amount) {
    if (balance < amount) {
        return false;
    }

    balance -= amount;
    return true;
}
```

The invariant is protected as one critical operation.

---

# 30. Practice Code — Thread-Safe Bank Account ⭐⭐⭐⭐⭐

```java
public class ThreadSafeBankAccount {

    private int balance;

    public ThreadSafeBankAccount(int initialBalance) {
        this.balance = initialBalance;
    }

    public synchronized boolean withdraw(int amount) {
        if (amount <= 0 || balance < amount) {
            return false;
        }

        balance -= amount;
        return true;
    }

    public synchronized void deposit(int amount) {
        if (amount <= 0) {
            return;
        }

        balance += amount;
    }

    public synchronized int getBalance() {
        return balance;
    }

    public static void main(String[] args) throws Exception {
        ThreadSafeBankAccount account =
                new ThreadSafeBankAccount(1000);

        Runnable withdrawTask = () -> {
            for (int i = 0; i < 10; i++) {
                account.withdraw(10);
            }
        };

        Thread t1 = new Thread(withdrawTask);
        Thread t2 = new Thread(withdrawTask);

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println("Balance = " + account.getBalance());
    }
}
```

---

# 31. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — Thinking `volatile` makes `count++` safe

It does not.

### Mistake 2 — Synchronizing only the write

If the check and update form one invariant, protect the whole operation.

### Mistake 3 — Using `AtomicInteger` for a multi-variable invariant

One atomic variable does not automatically make several related variables consistent.

### Mistake 4 — Using `ConcurrentHashMap` but performing unsafe multi-step logic

For example:

```java
if (!map.containsKey(key)) {
    map.put(key, value);
}
```

Use an atomic map operation such as `putIfAbsent()` when that matches the requirement.

### Mistake 5 — Forgetting `finally` with explicit locks

```java
lock.lock();
try {
    // work
} finally {
    lock.unlock();
}
```

### Mistake 6 — Sharing mutable objects unnecessarily

Prefer immutable data or confinement where practical.

### Mistake 7 — Forgetting `ThreadLocal.remove()` in thread pools

Task-specific state can leak between logically unrelated tasks executed by the same worker thread.

---

# 32. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What are common ways to make a class thread-safe?

Use immutability, synchronization, explicit locks, atomic variables, thread confinement, concurrent collections, safe publication, and minimizing shared mutable state.

### Q2. Is `volatile` enough for `count++`?

No. `count++` is a read-modify-write operation and is not atomic merely because `count` is volatile.

### Q3. When would you use `AtomicInteger` instead of `synchronized`?

For suitable single-variable atomic state transitions where lock-based coordination is unnecessary. If a larger invariant spans multiple variables or operations, locking may be clearer and more appropriate.

### Q4. Is `ConcurrentHashMap` completely lock-free?

No. It is designed for high concurrency using sophisticated synchronization and non-blocking techniques internally; do not equate it with being entirely lock-free.

### Q5. Why is immutability thread-safe?

Because state cannot be changed after construction, so concurrent readers do not race over mutations.

### Q6. What is thread confinement?

Restricting mutable state to one thread so concurrent threads cannot mutate that state simultaneously.

### Q7. What is safe publication?

Ensuring that a reference and the object's properly initialized state become visible to another thread through a mechanism that provides the required Java Memory Model guarantees.

### Q8. When is `CopyOnWriteArrayList` useful?

When reads greatly outnumber writes and snapshot-style iteration is useful.

### Q9. Can a thread-safe collection make application logic thread-safe?

Not necessarily. Compound business operations can still have race conditions even when individual collection operations are thread-safe.

### Q10. What is the simplest way to improve thread safety?

Reduce shared mutable state. Prefer immutable objects, confinement, and message passing where practical.

---

# 33. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Common thread-safety strategies include immutability, thread confinement, synchronization, explicit locks, atomic classes, volatile state, concurrent collections and safe publication. My first preference is usually to reduce shared mutable state—for example, use immutable objects or confine state to a thread. For a simple atomic counter, `AtomicInteger` can be appropriate. For a compound invariant involving multiple operations or fields, I would use `synchronized` or a lock to protect the entire critical section. `volatile` is useful for visibility, such as a stop flag, but it does not make compound operations like `count++` atomic. For collections, I prefer purpose-built classes such as `ConcurrentHashMap`, `BlockingQueue`, or `CopyOnWriteArrayList` depending on the access pattern. Finally, I make sure shared objects are safely published so other threads see correctly initialized state."**

---

# 34. Quick Revision ⭐⭐⭐⭐⭐

```text
THREAD SAFETY
      ↓
Reduce shared mutable state
      ↓
Prefer immutable objects
      ↓
Confine state where possible
      ↓
If shared:
      ↓
Need visibility? → volatile
Need single atomic update? → Atomic classes
Need compound invariant? → synchronized / Lock
Need concurrent collection? → ConcurrentHashMap / BlockingQueue / etc.
Need read-heavy snapshots? → CopyOnWrite
      ↓
Always safely publish shared objects
```

### Memory Trick

```text
Immutable → Don't change
Confinement → Don't share
Volatile → See changes
Atomic → One atomic state transition
Lock → Protect a critical section
Concurrent Collection → Shared collection safely
Safe Publication → Publish correctly
```

---

# 35. Practice Checklist

- [x] Thread-safety definition
- [x] Immutability
- [x] Thread confinement
- [x] `ThreadLocal`
- [x] `synchronized`
- [x] `ReentrantLock`
- [x] Atomic classes
- [x] CAS / `compareAndSet()`
- [x] `volatile`
- [x] Concurrent collections
- [x] `CopyOnWriteArrayList`
- [x] Safe publication
- [x] Minimizing shared mutable state
- [x] Atomicity vs thread safety
- [x] Invariant protection
- [x] Bank account example
- [x] Common mistakes
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.33 — Thread Communication Patterns](../33-Thread-Communication-Patterns/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.35 — Multithreading Interview Scenarios**