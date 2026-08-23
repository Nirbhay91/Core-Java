# 8.21 — Atomic Variables & CAS

> **Goal:** Understand lock-free atomic operations in Java, the `java.util.concurrent.atomic` package, CAS (Compare-And-Set), and when atomic variables are preferable to `synchronized` for simple shared-state updates.

---

## 1. What Are Atomic Variables? ⭐⭐⭐⭐⭐

Atomic variables provide **thread-safe operations on a single variable without requiring an explicit lock around each operation**.

Common classes:

```text
AtomicInteger
AtomicLong
AtomicBoolean
AtomicReference<T>
AtomicIntegerArray
AtomicLongArray
AtomicReferenceArray<T>
```

Package:

```java
java.util.concurrent.atomic
```

Example:

```java
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();
```

### Memory Trick

```text
Normal int
   ↓
read → modify → write
   ↓
can race

AtomicInteger
   ↓
atomic operation
   ↓
thread-safe update
```

---

# 2. Why Atomic Variables? ⭐⭐⭐⭐⭐

Consider:

```java
count++;
```

This is not one indivisible operation. Conceptually:

```text
read count
   ↓
add 1
   ↓
write count
```

Two threads can interleave these operations and lose an update.

With:

```java
atomicCount.incrementAndGet();
```

the increment is performed atomically.

---

# 3. Atomicity of `count++` ⭐⭐⭐⭐⭐

Wrong for concurrent increments:

```java
private int count;

public void increment() {
    count++;
}
```

Even if `count` is `volatile`, this is still not an atomic increment.

```text
Thread A       Thread B
   ↓              ↓
 read 10       read 10
   ↓              ↓
 add 1          add 1
   ↓              ↓
 write 11      write 11

Expected = 12
Actual   = 11
```

Correct simple counter:

```java
private final AtomicInteger count = new AtomicInteger();

public void increment() {
    count.incrementAndGet();
}
```

---

# 4. CAS — Compare-And-Set ⭐⭐⭐⭐⭐

CAS stands for:

> **Compare And Set** / **Compare And Swap**

The basic idea:

```text
Expected value = E
New value      = N

If current == E
    current = N
else
    do nothing / report failure
```

Conceptually:

```java
if (currentValue == expectedValue) {
    currentValue = newValue;
    return true;
}
return false;
```

The important part is that the comparison and update happen as **one atomic operation**.

---

# 5. CAS Example ⭐⭐⭐⭐⭐

```java
AtomicInteger value = new AtomicInteger(10);

boolean updated = value.compareAndSet(10, 20);

System.out.println(updated); // true
System.out.println(value.get()); // 20
```

If the expected value is wrong:

```java
AtomicInteger value = new AtomicInteger(10);

boolean updated = value.compareAndSet(5, 20);

System.out.println(updated); // false
System.out.println(value.get()); // 10
```

---

# 6. CAS Loop Pattern ⭐⭐⭐⭐⭐

Many atomic read-modify-write operations can be understood using a retry loop:

```java
while (true) {
    int current = value.get();
    int next = current + 1;

    if (value.compareAndSet(current, next)) {
        break;
    }
}
```

If another thread changes the value before the CAS succeeds:

```text
read
 ↓
CAS fails
 ↓
read latest value
 ↓
calculate again
 ↓
CAS
```

This is a fundamental lock-free pattern.

---

# 7. `AtomicInteger` Fundamentals ⭐⭐⭐⭐⭐

```java
AtomicInteger value = new AtomicInteger(10);
```

Useful methods:

```java
get()
set()
lazySet()
getAndSet()
compareAndSet()
weakCompareAndSet()
incrementAndGet()
getAndIncrement()
decrementAndGet()
getAndDecrement()
addAndGet()
getAndAdd()
updateAndGet()
getAndUpdate()
accumulateAndGet()
getAndAccumulate()
```

---

# 8. `getAndIncrement()` vs `incrementAndGet()` ⭐⭐⭐⭐⭐

### `getAndIncrement()`

Returns the **old** value, then increments.

```java
AtomicInteger value = new AtomicInteger(10);

int result = value.getAndIncrement();

System.out.println(result);      // 10
System.out.println(value.get()); // 11
```

### `incrementAndGet()`

Increments first, then returns the **new** value.

```java
AtomicInteger value = new AtomicInteger(10);

int result = value.incrementAndGet();

System.out.println(result);      // 11
System.out.println(value.get()); // 11
```

### Memory Trick

```text
getAndIncrement     → get OLD, then increment
incrementAndGet     → increment, then get NEW
```

---

# 9. `addAndGet()` vs `getAndAdd()`

```java
AtomicInteger value = new AtomicInteger(10);

int a = value.addAndGet(5);
// a = 15

value.set(10);

int b = value.getAndAdd(5);
// b = 10
// value = 15
```

Same atomic update idea; return-value timing differs.

---

# 10. Practice — Atomic Counter ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {

    private final AtomicInteger counter = new AtomicInteger();

    public void increment() {
        counter.incrementAndGet();
    }

    public int get() {
        return counter.get();
    }

    public static void main(String[] args) throws InterruptedException {
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

        System.out.println(counter.get()); // 200000
    }
}
```

---

# 11. Practice — Compare Normal Counter vs Atomic Counter ⭐⭐⭐⭐⭐

Unsafe:

```java
class UnsafeCounter {
    private int count;

    void increment() {
        count++;
    }

    int get() {
        return count;
    }
}
```

Safe simple counter:

```java
class SafeCounter {
    private final AtomicInteger count = new AtomicInteger();

    void increment() {
        count.incrementAndGet();
    }

    int get() {
        return count.get();
    }
}
```

Use this as an interview demonstration of **atomicity**, not as a claim that every shared-state problem can be solved with an atomic variable.

---

# 12. `AtomicBoolean` ⭐⭐⭐⭐⭐

Useful for a simple thread-safe boolean state.

```java
AtomicBoolean started = new AtomicBoolean(false);
```

Example:

```java
if (started.compareAndSet(false, true)) {
    System.out.println("This thread started the task");
}
```

This is useful when exactly one thread should successfully transition a state from `false` to `true`.

---

# 13. Practice — One-Time Initialization ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicBoolean;

public class OneTimeInitialization {

    private final AtomicBoolean initialized = new AtomicBoolean(false);

    public void initialize() {
        if (initialized.compareAndSet(false, true)) {
            System.out.println(Thread.currentThread().getName()
                    + " performed initialization");
        } else {
            System.out.println(Thread.currentThread().getName()
                    + " found it already initialized");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        OneTimeInitialization service = new OneTimeInitialization();

        Thread t1 = new Thread(service::initialize, "Thread-1");
        Thread t2 = new Thread(service::initialize, "Thread-2");

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

Only one thread can successfully perform the `false → true` CAS transition.

---

# 14. `AtomicLong` ⭐⭐⭐⭐

Use `AtomicLong` for atomic operations on a `long` value.

```java
AtomicLong sequence = new AtomicLong(1000);

long id = sequence.incrementAndGet();
```

Common scenario:

```text
Request 1 → ID 1001
Request 2 → ID 1002
Request 3 → ID 1003
```

It can be useful for simple in-memory sequence generation where the requirements fit atomic-counter semantics.

---

# 15. `AtomicReference<T>` ⭐⭐⭐⭐⭐

Atomicity is not limited to primitive values.

```java
AtomicReference<String> state =
        new AtomicReference<>("NEW");
```

CAS:

```java
boolean changed = state.compareAndSet("NEW", "PROCESSING");
```

This atomically changes the reference if it still points to the expected value.

---

# 16. Practice — Atomic State Transition ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicReference;

public class AtomicStateTransition {

    private final AtomicReference<String> state =
            new AtomicReference<>("NEW");

    public boolean start() {
        return state.compareAndSet("NEW", "PROCESSING");
    }

    public String getState() {
        return state.get();
    }

    public static void main(String[] args) {
        AtomicStateTransition service = new AtomicStateTransition();

        System.out.println(service.start());
        System.out.println(service.getState());

        System.out.println(service.start());
    }
}
```

Expected concept:

```text
NEW → PROCESSING  : true
PROCESSING → ???  : false
```

---

# 17. Atomic Reference Does Not Make Object Internals Automatically Thread-Safe ⚠️⭐⭐⭐⭐⭐

Consider:

```java
AtomicReference<Account> account;
```

The reference update can be atomic, but this does **not** automatically make every mutable field inside `Account` thread-safe.

```text
AtomicReference
      ↓
atomic reference replacement

Account object internals
      ↓
may still have races
```

This distinction is important in interviews.

---

# 18. `updateAndGet()` ⭐⭐⭐⭐⭐

```java
AtomicInteger value = new AtomicInteger(10);

int result = value.updateAndGet(x -> x * 2);

System.out.println(result); // 20
```

The function may be evaluated more than once under contention, so the update function should be **side-effect free**.

Good:

```java
value.updateAndGet(x -> x + 1);
```

Avoid relying on external side effects inside the function.

---

# 19. Practice — Atomic Update ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicUpdateExample {

    public static void main(String[] args) {
        AtomicInteger value = new AtomicInteger(5);

        int result = value.updateAndGet(x -> x * 10);

        System.out.println("Result: " + result);
        System.out.println("Value: " + value.get());
    }
}
```

---

# 20. `accumulateAndGet()` ⭐⭐⭐⭐

```java
AtomicInteger value = new AtomicInteger(10);

int result = value.accumulateAndGet(5, Integer::max);
```

The accumulator function combines the current value and supplied value atomically.

Similarly:

```java
getAndAccumulate(...)
```

returns the previous value.

---

# 21. Weak CAS ⭐⭐⭐

Some atomic APIs provide weak CAS operations.

Conceptually:

```text
compareAndSet
→ strong CAS semantics

weakCompareAndSet
→ may fail spuriously
```

A weak CAS can be useful inside retry loops where failure is already handled by retry logic.

Do not confuse a **spurious failure** with a value mismatch.

---

# 22. CAS Is Lock-Free, Not Wait-Free ⭐⭐⭐⭐⭐

This distinction is an important interview point.

### Lock-free

The system as a whole makes progress even if some individual operation retries.

### Wait-free

Every operation completes in a bounded number of steps.

Atomic CAS-based algorithms are commonly associated with lock-free progress, but an individual thread may repeatedly lose CAS races and retry.

Therefore:

```text
Lock-free ≠ wait-free
```

---

# 23. CAS Contention ⭐⭐⭐⭐

If many threads repeatedly update the same atomic variable:

```text
Thread A ─┐
Thread B ─┼──→ same AtomicInteger
Thread C ─┤
Thread D ─┘
```

many CAS attempts can fail.

Possible consequences:

- retries
- CPU consumption
- contention
- reduced scalability

For very high-contention counters, classes such as `LongAdder` may be more appropriate; that is covered in the next topic.

---

# 24. ABA Problem ⭐⭐⭐⭐⭐

CAS can have an important conceptual issue called the **ABA problem**.

Example:

```text
Initial: A

Thread 1 reads A

Thread 2 changes:
A → B
B → A

Thread 1 performs CAS(A → C)
```

Thread 1 sees `A` again and may not know that the value changed in between.

This matters in algorithms where **identity/history** of the state change matters, not just the current value.

---

# 25. `AtomicStampedReference` ⭐⭐⭐⭐⭐

One way to address ABA is to associate a stamp/version with the reference.

```java
AtomicStampedReference<String> ref =
        new AtomicStampedReference<>("A", 1);
```

Conceptually:

```text
A + version 1
   ↓
B + version 2
   ↓
A + version 3
```

The value may be `A` again, but the stamp reveals that it changed.

---

# 26. Practice — ABA Detection ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class AtomicStampedReferenceExample {

    public static void main(String[] args) {

        AtomicStampedReference<String> ref =
                new AtomicStampedReference<>("A", 1);

        int[] stampHolder = new int[1];
        String value = ref.get(stampHolder);

        int oldStamp = stampHolder[0];

        boolean updated = ref.compareAndSet(
                value,
                "B",
                oldStamp,
                oldStamp + 1
        );

        System.out.println("Updated: " + updated);
        System.out.println("Value: " + ref.getReference());
        System.out.println("Stamp: " + ref.getStamp());
    }
}
```

---

# 27. Atomic Arrays ⭐⭐⭐⭐

Java provides:

```java
AtomicIntegerArray
AtomicLongArray
AtomicReferenceArray<T>
```

Example:

```java
AtomicIntegerArray array =
        new AtomicIntegerArray(5);

array.incrementAndGet(2);
```

The element-level operations are atomic.

---

# 28. Practice — AtomicIntegerArray ⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicIntegerArray;

public class AtomicArrayExample {

    public static void main(String[] args) {
        AtomicIntegerArray array = new AtomicIntegerArray(3);

        array.incrementAndGet(0);
        array.addAndGet(1, 10);
        array.compareAndSet(2, 0, 50);

        for (int i = 0; i < array.length(); i++) {
            System.out.println(array.get(i));
        }
    }
}
```

---

# 29. Atomic Variables vs `synchronized` ⭐⭐⭐⭐⭐

| Atomic Variables | `synchronized` |
|---|---|
| Good for simple atomic state operations | Good for compound critical sections |
| Usually uses CAS-based techniques internally | Uses monitor-based synchronization |
| No explicit lock API | Implicit monitor lock |
| Excellent for counters/state transitions | Better for multi-variable invariants |
| Can reduce blocking for suitable operations | Threads may block waiting for monitor |
| Not a universal replacement for locks | Handles arbitrary critical sections |

### Key Rule

> Use an atomic variable when the shared operation can be expressed cleanly as an atomic variable-level operation. Use locking when correctness depends on a larger multi-step invariant or multiple pieces of shared state.

---

# 30. Atomic Variables vs `volatile` ⭐⭐⭐⭐⭐

`volatile` gives visibility and ordering guarantees, but does not make compound operations such as:

```java
count++;
```

atomic.

Atomic variables provide atomic read-modify-write operations such as:

```java
count.incrementAndGet();
```

### Memory Trick

```text
volatile
→ visibility + ordering guarantees
→ NOT general compound-operation atomicity

AtomicInteger
→ atomic operations
→ volatile-like visibility semantics for its accesses
```

---

# 31. Atomic Variables vs `AtomicInteger` Locking Mental Model ⭐⭐⭐⭐⭐

Do not think:

```text
AtomicInteger = magically locks everything
```

Think:

```text
AtomicInteger
    ↓
atomic operations on ONE atomic variable
```

If you need:

```text
balance + transactionStatus + auditFlag
```

to change consistently as one invariant, a lock or another higher-level design may be required.

---

# 32. Practice — Why Atomic Alone May Be Insufficient ⚠️⭐⭐⭐⭐⭐

Suppose:

```java
class Account {
    int balance;
    boolean active;
}
```

If a business operation requires:

```text
balance changes
AND
active changes
```

as one indivisible transaction, two independent atomic variables do not automatically create a single atomic multi-variable operation.

You need a design that protects the invariant.

---

# 33. CAS vs `synchronized` ⭐⭐⭐⭐⭐

### CAS

```text
read
 ↓
calculate
 ↓
compare-and-set
 ↓
retry if failed
```

### `synchronized`

```text
acquire monitor
 ↓
execute critical section
 ↓
release monitor
```

CAS can avoid blocking for suitable algorithms, but high contention can still cause repeated retries.

---

# 34. Common Mistake — Thinking `AtomicInteger` Solves Everything ⚠️

Wrong:

> "If I make every field atomic, the entire object becomes thread-safe."

Not necessarily.

Thread safety depends on the **invariants and operations across the object**, not merely on the individual fields.

---

# 35. Common Mistake — Using `get()` + `set()` for Compound Logic ⚠️⭐⭐⭐⭐⭐

This can still race:

```java
int current = counter.get();
counter.set(current + 1);
```

Two threads can both read the same value.

Prefer:

```java
counter.incrementAndGet();
```

or an appropriate CAS/update method.

---

# 36. Common Mistake — Side Effects in `updateAndGet()` ⚠️

Avoid:

```java
counter.updateAndGet(x -> {
    sendMessage();
    return x + 1;
});
```

The function may be retried under contention.

A side effect could therefore happen more than once.

Prefer a pure update function and perform external side effects separately when appropriate.

---

# 37. Practice — CAS Retry Loop ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicInteger;

public class CasRetryExample {

    private final AtomicInteger value = new AtomicInteger();

    public void incrementUsingCAS() {
        while (true) {
            int current = value.get();
            int next = current + 1;

            if (value.compareAndSet(current, next)) {
                return;
            }
        }
    }

    public int get() {
        return value.get();
    }

    public static void main(String[] args) throws InterruptedException {
        CasRetryExample counter = new CasRetryExample();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                counter.incrementUsingCAS();
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                counter.incrementUsingCAS();
            }
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(counter.get());
    }
}
```

This is useful for understanding what an atomic increment can look like conceptually.

---

# 38. Practice — Atomic Ticket Generator ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicTicketGenerator {

    private final AtomicInteger nextTicket = new AtomicInteger(1);

    public int nextTicket() {
        return nextTicket.getAndIncrement();
    }

    public static void main(String[] args) throws InterruptedException {
        AtomicTicketGenerator generator = new AtomicTicketGenerator();

        Runnable task = () -> {
            for (int i = 0; i < 3; i++) {
                System.out.println(
                        Thread.currentThread().getName()
                                + " -> Ticket "
                                + generator.nextTicket());
            }
        };

        Thread t1 = new Thread(task, "User-1");
        Thread t2 = new Thread(task, "User-2");
        Thread t3 = new Thread(task, "User-3");

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();
    }
}
```

---

# 39. Practice — AtomicBoolean Shutdown Flag ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicBoolean;

public class AtomicShutdownFlag {

    private final AtomicBoolean running = new AtomicBoolean(true);

    public void stop() {
        running.set(false);
    }

    public void run() {
        while (running.get()) {
            // do work
        }

        System.out.println("Worker stopped");
    }
}
```

For modern executor-based applications, prefer proper task cancellation/shutdown mechanisms where appropriate; this example is specifically for understanding atomic state.

---

# 40. Atomic Snapshot / Immutable State ⭐⭐⭐⭐⭐

A useful pattern is to store an immutable object behind an `AtomicReference`.

```java
record Config(String host, int port) {}
```

Then:

```java
AtomicReference<Config> config =
        new AtomicReference<>(new Config("localhost", 8080));
```

A new immutable snapshot can be published atomically:

```java
config.set(new Config("server", 9090));
```

Readers can obtain a consistent reference to one immutable snapshot.

---

# 41. Practice — Atomic Configuration Update ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicReference;

public class AtomicConfigExample {

    record Config(String host, int port) {}

    public static void main(String[] args) {
        AtomicReference<Config> config =
                new AtomicReference<>(new Config("localhost", 8080));

        System.out.println(config.get());

        config.updateAndGet(oldConfig ->
                new Config(oldConfig.host(), oldConfig.port() + 1));

        System.out.println(config.get());
    }
}
```

---

# 42. Memory Visibility and Atomic Variables ⭐⭐⭐⭐⭐

Atomic classes provide memory semantics appropriate for their atomic operations.

You can safely use:

```java
atomicValue.set(...);
atomicValue.get();
```

for cross-thread state communication, subject to the algorithm's correctness.

But atomicity of one variable does not automatically make a larger algorithm thread-safe.

---

# 43. Linearizable Atomic Operations ⭐⭐⭐⭐⭐

Many operations on atomic classes have **linearizable** semantics.

For example:

```java
counter.incrementAndGet();
```

can be reasoned about as if the operation takes effect at one instant between its invocation and completion.

This makes atomic classes useful building blocks for concurrent algorithms.

---

# 44. When Should You Use Atomic Variables? ⭐⭐⭐⭐⭐

Good candidates:

- counters
- sequence numbers
- simple flags
- one-time state transitions
- atomic state machines
- atomic references to immutable snapshots
- simple statistics/state updates

Less suitable when:

- multiple variables must change together
- a complex invariant spans multiple operations
- a large critical section is required
- the algorithm needs blocking coordination

---

# 45. Production Scenario — Request Counter ⭐⭐⭐⭐⭐

```java
private final AtomicLong requestCount = new AtomicLong();

public void recordRequest() {
    requestCount.incrementAndGet();
}
```

Useful for simple in-memory counters.

For high-contention counters, `LongAdder` may provide better scalability in many workloads; it is covered in **8.22**.

---

# 46. Production Scenario — State Transition ⭐⭐⭐⭐⭐

```java
private final AtomicReference<State> state =
        new AtomicReference<>(State.NEW);
```

Then:

```java
state.compareAndSet(State.NEW, State.PROCESSING);
```

This can prevent two threads from both successfully performing the same state transition.

---

# 47. Interview Comparison ⭐⭐⭐⭐⭐

| Concept | Atomic Variable | `volatile` | `synchronized` |
|---|---|---|---|
| Visibility | ✅ | ✅ | ✅ |
| Atomic simple operations | ✅ | ❌ generally | ✅ |
| Compound critical section | ❌ by itself | ❌ | ✅ |
| Multiple-variable invariant | ❌ by itself | ❌ | ✅ |
| Explicit blocking | Usually no | No | Yes, monitor acquisition may block |
| Typical mechanism | CAS / atomic primitives | JVM memory semantics | Monitor synchronization |

---

# 48. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is CAS?

Compare-And-Set atomically updates a value only if it still equals the expected value.

### Q2. Why is `count++` not thread-safe?

Because it is a read-modify-write sequence rather than one atomic operation.

### Q3. Is `volatile int count` enough for `count++`?

No. `volatile` provides visibility/order guarantees but does not make the increment compound operation atomic.

### Q4. Is `AtomicInteger` a replacement for `synchronized`?

No. It is excellent for suitable single-variable atomic operations, but it does not replace locks for arbitrary critical sections or multi-variable invariants.

### Q5. What happens when CAS fails?

The update is not applied. A CAS-based algorithm may retry using the latest value.

### Q6. What is the ABA problem?

A value can change A→B→A, causing a CAS that only checks the current value to miss the intermediate change.

### Q7. How can ABA be addressed?

A version/stamp can be associated with the value, for example using `AtomicStampedReference`.

### Q8. Is lock-free the same as wait-free?

No. Lock-free means system-wide progress; a particular thread may still retry indefinitely in theory. Wait-free provides a bounded completion guarantee.

### Q9. When would you use `AtomicReference`?

For atomic replacement or CAS-based transitions of object references, especially when the referenced state is immutable or treated as an immutable snapshot.

### Q10. Why should `updateAndGet()` functions avoid side effects?

Because the function can be evaluated more than once during contention/retries.

---

# 49. 2-Minute Interview Answer 🏆

> **"Java's atomic variables are provided mainly through `java.util.concurrent.atomic`. Classes such as `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, and `AtomicReference` provide thread-safe atomic operations on individual variables. They commonly use CAS, or Compare-And-Set, where an update succeeds only if the current value still equals the expected value. For example, `AtomicInteger.incrementAndGet()` is atomic, whereas `count++` is a read-modify-write operation and is not thread-safe. Atomic variables are useful for counters, flags, sequence generators and simple state transitions. They are not a replacement for `synchronized` or locks when multiple variables must change together or a larger critical section must be protected. An important CAS issue is the ABA problem, which can be addressed with versioning such as `AtomicStampedReference`. CAS-based algorithms can be lock-free but that does not mean they are wait-free."**

---

# 50. Quick Revision ⭐⭐⭐⭐⭐

```text
Atomic Variables
       ↓
java.util.concurrent.atomic
       ↓
┌────────┬──────────┬──────────────┐
↓        ↓          ↓              ↓
Integer  Long     Boolean      Reference
       ↓
      CAS
       ↓
Compare expected
       ↓
If equal → update
If not   → fail/retry
```

### Golden Rules

```text
count++                  → NOT atomic
volatile count++         → NOT atomic
AtomicInteger.increment  → atomic
compareAndSet            → conditional atomic update
CAS failure              → retry may be required
Atomic ≠ thread-safe object automatically
Lock-free ≠ wait-free
ABA → use version/stamp when required
```

---

# 51. 💻 Practice Checklist

- [ ] Create `AtomicInteger`
- [ ] Use `get()` / `set()`
- [ ] Use `incrementAndGet()`
- [ ] Use `getAndIncrement()`
- [ ] Use `addAndGet()` / `getAndAdd()`
- [ ] Use `compareAndSet()`
- [ ] Write a CAS retry loop
- [ ] Use `updateAndGet()`
- [ ] Use `accumulateAndGet()`
- [ ] Create `AtomicLong`
- [ ] Create `AtomicBoolean`
- [ ] Use `AtomicReference`
- [ ] Build an atomic state transition
- [ ] Build an atomic ticket generator
- [ ] Use `AtomicIntegerArray`
- [ ] Understand ABA
- [ ] Practice `AtomicStampedReference`
- [ ] Compare atomic vs `volatile`
- [ ] Compare atomic vs `synchronized`
- [ ] Understand lock-free vs wait-free
- [ ] Identify multi-variable invariant problems
- [ ] Explain CAS in 2 minutes

---

## Navigation

[← 8.20 — `Condition`](../20-Condition/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.22 — `LongAdder` / `LongAccumulator`**