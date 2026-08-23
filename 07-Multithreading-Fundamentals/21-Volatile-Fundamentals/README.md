# 7.21 — `volatile` Fundamentals

## 🎯 Objective

Understand Java's `volatile` keyword, why it exists, what guarantees it provides, what it does **not** provide, and when it should or should not be used in concurrent programs.

> **Interview rule:** `volatile` is primarily about **visibility and ordering guarantees**. It does **not** provide general mutual exclusion or make compound operations such as `count++` atomic.

---

# 1. What is `volatile`? ⭐⭐⭐⭐⭐

`volatile` is a Java keyword used for a field that may be accessed by multiple threads and requires the Java Memory Model's volatile visibility and ordering guarantees.

```java
private volatile boolean running = true;
```

A volatile write and a subsequent volatile read of the same variable participate in a happens-before relationship.

```text
Thread A
volatile write
      ↓
   happens-before
      ↓
Thread B
volatile read
```

---

# 2. Why Do We Need `volatile`? ⭐⭐⭐⭐⭐

Consider a shared flag:

```java
private boolean running = true;
```

One thread executes:

```java
running = false;
```

Another thread executes:

```java
while (running) {
    // work
}
```

Without an appropriate synchronization mechanism, the program does not have the required cross-thread visibility guarantee for this communication.

Declaring the flag volatile makes the intended visibility/order semantics explicit:

```java
private volatile boolean running = true;
```

---

# 3. The Three Key Properties ⭐⭐⭐⭐⭐

For interview purposes, remember:

```text
volatile
   ↓
Visibility
   +
Ordering guarantees
   +
No general mutual exclusion
   +
No general compound-operation atomicity
```

### Easy memory trick

> **volatile = "see the latest value under JMM rules" + ordering semantics, not "make everything thread-safe".**

---

# 4. Basic Syntax

```java
private volatile boolean ready;
```

It can be used with fields of suitable Java types:

```java
volatile boolean flag;
volatile int count;
volatile long value;
volatile Object reference;
```

The important point is the memory semantics, not the particular field type.

---

# 5. Practice Code — Visibility Flag ⭐⭐⭐⭐⭐

```java
public class VolatileFlagDemo {

    private volatile boolean running = true;

    public void work() {
        while (running) {
            // Simulate work
        }

        System.out.println("Worker stopped");
    }

    public void stop() {
        running = false;
    }

    public static void main(String[] args)
            throws InterruptedException {

        VolatileFlagDemo demo = new VolatileFlagDemo();

        Thread worker = new Thread(demo::work);
        worker.start();

        Thread.sleep(100);
        demo.stop();

        worker.join();
    }
}
```

### What happens?

```text
Main thread
    |
    | running = false
    v
volatile write
    |
    | happens-before
    v
Worker volatile read
    |
    v
while loop terminates
```

This is a classic valid use case for `volatile`.

---

# 6. Visibility Guarantee ⭐⭐⭐⭐⭐

A write to a volatile variable happens-before every subsequent read of that same variable.

Example:

```java
private volatile boolean ready;
```

Writer:

```java
ready = true;
```

Reader:

```java
if (ready) {
    // sees the volatile state according to JMM guarantees
}
```

This is much stronger than simply saying "volatile reads from RAM". The correct explanation is based on the **Java Memory Model**.

---

# 7. Practice Code — Publication with `volatile` ⭐⭐⭐⭐⭐

```java
public class VolatilePublicationDemo {

    private int data;
    private volatile boolean ready;

    public void writer() {
        data = 42;
        ready = true;
    }

    public void reader() {
        if (ready) {
            System.out.println("Data = " + data);
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        VolatilePublicationDemo demo =
                new VolatilePublicationDemo();

        Thread writer = new Thread(demo::writer);
        Thread reader = new Thread(demo::reader);

        writer.start();
        writer.join();

        reader.start();
        reader.join();
    }
}
```

The volatile write to `ready` provides the relevant publication relationship for actions preceding that write.

---

# 8. `volatile` and Happens-Before ⭐⭐⭐⭐⭐

The key rule is:

> **A write to a volatile variable happens-before every subsequent read of that same variable.**

Example:

```text
Thread A
--------
data = 42
ready = true  ← volatile write
                 |
                 | happens-before
                 v
Thread B
--------
read ready    ← volatile read
read data
```

This is why the volatile flag publication pattern works.

---

# 9. `volatile` Does NOT Mean Atomicity ⭐⭐⭐⭐⭐

Classic trap:

```java
private volatile int count;

count++;
```

❌ `count++` is still not atomic.

Conceptually:

```text
read count
   ↓
add 1
   ↓
write count
```

Multiple threads can interleave these steps.

---

# 10. Practice Code — `volatile` Counter Trap ⭐⭐⭐⭐⭐

```java
public class VolatileCounterDemo {

    private volatile int count;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args)
            throws InterruptedException {

        VolatileCounterDemo demo =
                new VolatileCounterDemo();

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

        System.out.println("Expected = 200000");
        System.out.println("Actual   = " + demo.getCount());
    }
}
```

The final value can be lower than the expected value because visibility does not turn the read-modify-write sequence into one atomic operation.

---

# 11. Correct Counter Alternatives ⭐⭐⭐⭐⭐

### Option 1 — `AtomicInteger`

```java
private final AtomicInteger count = new AtomicInteger();

count.incrementAndGet();
```

### Option 2 — `synchronized`

```java
public synchronized void increment() {
    count++;
}
```

Choose based on the complete concurrency requirement rather than automatically replacing every `volatile` with a lock.

---

# 12. `volatile` vs `synchronized` ⭐⭐⭐⭐⭐

| Feature | `volatile` | `synchronized` |
|---|---|---|
| Visibility | ✅ | ✅ |
| Ordering / JMM guarantees | ✅ | ✅ |
| Mutual exclusion | ❌ | ✅ |
| General compound-operation atomicity | ❌ | ✅ for protected critical section |
| Lock acquisition | ❌ | ✅ |
| Good for simple state flag | ✅ | Can be, but may be unnecessary |
| Good for multiple related invariants | Usually ❌ | ✅ |

### Golden Rule

```text
Simple shared state + visibility/order requirement
→ consider volatile

Multiple-step invariant / critical section
→ consider synchronized or Lock

Atomic single-variable update
→ consider Atomic* classes
```

---

# 13. `volatile` vs `AtomicInteger` ⭐⭐⭐⭐⭐

### `volatile`

```java
volatile int count;
```

Provides volatile memory semantics for accesses to `count`, but:

```java
count++;
```

is not atomic.

### `AtomicInteger`

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```

Provides atomic operations such as increment, CAS and update operations, together with the memory semantics specified by the API.

---

# 14. `volatile` and Object References ⭐⭐⭐⭐

A volatile reference:

```java
private volatile Config config;
```

means reads/writes of the **reference** have volatile semantics.

It does **not** automatically make the mutable object referenced by that field thread-safe.

Example:

```java
config.setTimeout(5000);
```

Whether this mutation is thread-safe depends on `Config`'s own design.

### Important interview point

> **Volatile reference ≠ immutable object ≠ thread-safe object.**

---

# 15. Practice Code — Volatile Reference

```java
public class VolatileReferenceDemo {

    private volatile Config config;

    public void update() {
        config = new Config(5000);
    }

    public Config getConfig() {
        return config;
    }

    static class Config {
        private final int timeout;

        Config(int timeout) {
            this.timeout = timeout;
        }

        public int getTimeout() {
            return timeout;
        }
    }
}
```

Because `Config` is immutable here, replacing the reference is a simple publication pattern.

---

# 16. `volatile` and `final` ⭐⭐⭐⭐

Do not confuse:

```java
final
```

with:

```java
volatile
```

### `final`

Primarily expresses that a variable/reference cannot be reassigned after initialization.

### `volatile`

Provides volatile memory semantics for reads/writes of a field.

Example:

```java
private final int id;
private volatile boolean active;
```

They solve different problems.

---

# 17. `volatile` Does Not Provide Mutual Exclusion ⭐⭐⭐⭐⭐

Suppose:

```java
volatile boolean available = true;
```

Two threads can both observe:

```text
available == true
```

and then both attempt to perform an action.

If the operation is:

```text
check state
   ↓
modify state
```

then a single volatile variable may not be enough.

Use a suitable atomic operation, lock, or higher-level concurrency abstraction.

---

# 18. Practice Code — Check-Then-Act Problem

```java
public class VolatileCheckThenActDemo {

    private volatile boolean available = true;

    public void useResource() {
        if (available) {
            // Another thread may also pass this check.
            available = false;
            System.out.println(
                    Thread.currentThread().getName()
                            + " acquired resource");
        }
    }
}
```

`volatile` makes the flag visible, but it does not make the **check + update** operation one indivisible action.

Possible solutions include:

```java
AtomicBoolean.compareAndSet(...)
```

or synchronization/locking.

---

# 19. Correct Check-Then-Act with `AtomicBoolean` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicBoolean;

public class AtomicBooleanDemo {

    private final AtomicBoolean available =
            new AtomicBoolean(true);

    public boolean acquire() {
        return available.compareAndSet(true, false);
    }
}
```

Now the state transition:

```text
true → false
```

is attempted atomically through CAS.

---

# 20. When Should You Use `volatile`? ⭐⭐⭐⭐⭐

Good examples:

### 1. Shutdown flag

```java
private volatile boolean shutdown;
```

### 2. Configuration/state publication

```java
private volatile Config config;
```

especially when replacing an immutable object/reference.

### 3. Simple state indicators

```java
private volatile boolean initialized;
```

when the communication protocol requires the volatile guarantees.

### 4. One-writer/simple-publication designs

When the state transition is simple and no compound invariant needs to be protected.

---

# 21. When Should You NOT Use `volatile` Alone? ⭐⭐⭐⭐⭐

Do not rely on `volatile` alone when you need:

- Mutual exclusion
- Atomic read-modify-write operations
- Multiple fields updated as one invariant
- Check-then-act atomicity
- Compound business transactions
- Complex state transitions

Examples:

```java
count++;
```

```java
if (balance >= amount) {
    balance -= amount;
}
```

```java
if (!initialized) {
    initialize();
    initialized = true;
}
```

These may require atomic operations or synchronization depending on the design.

---

# 22. `volatile` and Instruction Reordering ⭐⭐⭐⭐⭐

A common oversimplification is:

> "volatile prevents all compiler/CPU reordering."

A better explanation is:

> **Volatile accesses have specific ordering semantics under the Java Memory Model. They establish happens-before relationships between a volatile write and subsequent reads of the same variable and constrain reorderings in ways required by those semantics.**

Do not explain volatile purely as a CPU cache trick.

---

# 23. Safe Publication with `volatile` ⭐⭐⭐⭐⭐

A classic pattern:

```java
public class Service {
    // immutable or safely constructed object
}

public class Holder {
    private volatile Service service;

    public void publish() {
        service = new Service();
    }

    public Service get() {
        return service;
    }
}
```

The volatile reference provides the required visibility/order semantics for publication of the reference and preceding construction actions under the JMM.

The object's own thread safety still depends on how the object is constructed and used.

---

# 24. Practice Code — Safe Publication

```java
public class SafePublicationDemo {

    private volatile Settings settings;

    public void publish() {
        settings = new Settings("production", 8080);
    }

    public Settings getSettings() {
        return settings;
    }

    static final class Settings {
        private final String environment;
        private final int port;

        Settings(String environment, int port) {
            this.environment = environment;
            this.port = port;
        }

        public String getEnvironment() {
            return environment;
        }

        public int getPort() {
            return port;
        }
    }
}
```

Immutable objects are particularly useful for safe publication because readers do not need to coordinate mutable state after publication.

---

# 25. `volatile` and Multiple Writers ⭐⭐⭐⭐

`volatile` does not mean "only one thread can write."

Multiple threads can write a volatile variable:

```java
volatile int state;
```

The reads/writes receive volatile semantics, but a sequence such as:

```text
read state
calculate new state
write state
```

is still a compound operation if the calculation depends on the previous value.

For such cases, consider atomic classes or synchronization.

---

# 26. Practice Code — Multiple Writers

```java
public class VolatileMultipleWriterDemo {

    private volatile int state;

    public void update(int value) {
        state = value;
    }

    public int getState() {
        return state;
    }
}
```

Individual volatile accesses have volatile semantics, but application-level correctness may require a stronger protocol when multiple writers coordinate through the same state.

---

# 27. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1 — `volatile` makes `count++` thread-safe

❌ False.

### Trap 2 — `volatile` provides locking

❌ False.

### Trap 3 — `volatile` is only for performance

❌ False. It defines important JMM semantics.

### Trap 4 — `volatile` makes referenced objects immutable

❌ False.

### Trap 5 — `sleep()` can replace volatile

❌ False.

### Trap 6 — volatile means direct RAM access

❌ Too simplistic and misleading.

### Trap 7 — volatile solves every visibility problem

❌ Only when the program's communication protocol is correctly designed around the volatile semantics.

---

# 28. Practice Question — Identify the Bug

```java
class Counter {
    private volatile int count;

    public void increment() {
        count++;
    }
}
```

### Answer

The field has volatile visibility/order semantics, but `count++` is still a compound read-modify-write operation.

### Fix

Use:

```java
AtomicInteger
```

or:

```java
synchronized
```

according to the requirements.

---

# 29. Practice Question — Is This Valid?

```java
class Worker {
    private volatile boolean running = true;

    void stop() {
        running = false;
    }

    void run() {
        while (running) {
            // work
        }
    }
}
```

### Answer

Yes, this is a classic valid use case when the only required shared-state operation is observing the shutdown flag and no additional invariant needs protection.

---

# 30. Practice Question — Is This Enough?

```java
private volatile boolean initialized;

if (!initialized) {
    initialize();
    initialized = true;
}
```

### Answer

Not necessarily. Multiple threads can observe `false` and both enter initialization. This is a check-then-act race.

You may need synchronization, an atomic state transition, or a suitable initialization mechanism.

---

# 31. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is volatile?

A Java keyword that gives a field volatile memory semantics, including specified visibility and ordering guarantees under the JMM.

### Q2. What problem does volatile solve?

It is useful for cross-thread visibility and ordering of a shared field when mutual exclusion and compound-operation atomicity are not required.

### Q3. Does volatile guarantee atomicity?

Not for arbitrary compound operations. For example, `count++` is not made atomic by volatile.

### Q4. Does volatile provide mutual exclusion?

No.

### Q5. What is the happens-before rule for volatile?

A write to a volatile variable happens-before every subsequent read of that same variable.

### Q6. Give a common volatile use case.

A worker shutdown flag such as `volatile boolean running`.

### Q7. Can a volatile reference make an object thread-safe?

No. It gives volatile semantics to the reference access; the referenced object's mutable state still needs its own thread-safety design.

### Q8. Volatile vs AtomicInteger?

Volatile is appropriate for visibility/order of a field; `AtomicInteger` provides atomic operations such as increment and compare-and-set.

### Q9. Volatile vs synchronized?

Volatile does not provide mutual exclusion; synchronized provides mutual exclusion plus memory visibility and ordering guarantees.

### Q10. Can volatile replace `sleep()`?

They solve different problems. Volatile can provide visibility for a shared flag; sleep is a timing/blocking mechanism and is not a general synchronization mechanism.

### Q11. Can multiple threads write a volatile field?

Yes, but volatile alone does not make a multi-step state transition atomic.

### Q12. What is the biggest volatile interview trap?

Saying "volatile makes the variable thread-safe" without explaining that compound operations and object invariants may still require stronger synchronization.

---

# 32. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Volatile is a Java keyword used when a shared field needs the Java Memory Model's volatile visibility and ordering guarantees. A volatile write happens-before a subsequent read of the same variable, so it is commonly used for simple state flags such as a worker shutdown flag or for publishing a new reference. However, volatile does not provide mutual exclusion and does not make compound operations atomic. For example, `volatile int count; count++` is still a read-modify-write operation and can lose updates when multiple threads execute it. In that case I would use `AtomicInteger` or synchronization depending on the requirement. Also, a volatile reference does not make the referenced mutable object thread-safe. So I use volatile when the concurrency requirement is primarily visibility/order of a shared state, and I use atomic classes or locks when I need atomic state transitions or protection of larger critical sections."**

---

# 33. Quick Revision ⭐⭐⭐⭐⭐

```text
volatile
   ↓
JMM visibility + ordering semantics

volatile write
   ↓ happens-before
volatile read of same variable

Good use
→ shutdown flag
→ simple state publication
→ immutable reference replacement

volatile does NOT provide
→ mutual exclusion
→ general compound-operation atomicity
→ automatic object thread safety

volatile int count
count++
❌ NOT atomic

Need atomic update?
→ AtomicInteger / AtomicBoolean / AtomicLong

Need critical section?
→ synchronized / Lock

Need simple visibility flag?
→ volatile
```

### Golden Rule

> **Use `volatile` for shared state visibility/order when you do not need mutual exclusion or compound-operation atomicity.**

---

# 34. Practice Checklist

- [x] `volatile` definition
- [x] Why `volatile` exists
- [x] Visibility guarantee
- [x] Happens-before rule
- [x] Shutdown flag example
- [x] Publication example
- [x] `volatile` + `count++` trap
- [x] `volatile` vs `synchronized`
- [x] `volatile` vs `AtomicInteger`
- [x] Volatile references
- [x] Check-then-act problem
- [x] `AtomicBoolean.compareAndSet()` solution
- [x] Safe publication
- [x] Multiple writers
- [x] Instruction-ordering explanation
- [x] Common interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.20 — Happens-Before Relationship](../20-Happens-Before-Relationship/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.22 — `volatile` vs `synchronized`**