# 7.19 — Atomicity vs Visibility vs Ordering

## 🎯 Objective

Understand the three core Java Memory Model concepts that frequently appear in multithreading interviews:

- **Atomicity** — whether an operation happens indivisibly.
- **Visibility** — whether one thread can observe another thread's update.
- **Ordering** — what execution orderings are guaranteed to be observed between threads.

> **Thread safety is not just about locking. You must understand atomicity, visibility and ordering together.**

---

# 1. The Big Picture ⭐⭐⭐⭐⭐

```text
                 Concurrent Program
                         |
          +--------------+--------------+
          |              |              |
      Atomicity       Visibility      Ordering
          |              |              |
    indivisible       updates         happens-before /
    operation         observed        allowed ordering
```

A program can have one property without automatically having all three.

---

# 2. Atomicity ⭐⭐⭐⭐⭐

An operation is **atomic** when it appears to happen as one indivisible action from the perspective relevant to concurrent execution.

Example:

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```

The increment operation provided by `AtomicInteger` is atomic.

But:

```java
count++;
```

is not an atomic compound operation.

Conceptually:

```text
read count
   ↓
add 1
   ↓
write count
```

Another thread can interleave between these steps.

---

# 3. Practice Code — Lost Update ⭐⭐⭐⭐⭐

```java
public class AtomicityProblem {

    private int count;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args)
            throws InterruptedException {

        AtomicityProblem counter = new AtomicityProblem();

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

        System.out.println("Expected = 200000");
        System.out.println("Actual   = " + counter.getCount());
    }
}
```

The final value may be lower than expected because increments can be lost.

---

# 4. How `synchronized` Fixes Atomicity ⭐⭐⭐⭐⭐

```java
public synchronized void increment() {
    count++;
}
```

The synchronized method ensures that only one thread at a time executes the protected critical section for the same monitor.

For this example, the read-modify-write operation is protected as one logical critical section.

---

# 5. Atomic Classes ⭐⭐⭐⭐⭐

For suitable single-variable state transitions, use atomic classes:

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

### Important

Atomic classes do not automatically make arbitrary multi-object or multi-step business operations atomic.

---

# 6. Practice Code — `AtomicInteger`

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicIntegerPractice {

    private final AtomicInteger count = new AtomicInteger();

    public void increment() {
        count.incrementAndGet();
    }

    public int getCount() {
        return count.get();
    }

    public static void main(String[] args)
            throws InterruptedException {

        AtomicIntegerPractice counter =
                new AtomicIntegerPractice();

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

# 7. Visibility ⭐⭐⭐⭐⭐

**Visibility** means that when one thread updates a variable, another thread can observe the update according to the Java Memory Model's guarantees.

Without appropriate synchronization or another visibility mechanism, a thread may continue observing an older value.

Example:

```java
private boolean running = true;
```

One thread changes:

```java
running = false;
```

Another thread repeatedly checks:

```java
while (running) {
    // work
}
```

Without an appropriate visibility guarantee, this is not a correct general-purpose communication mechanism.

---

# 8. Practice Code — Visibility with `volatile` ⭐⭐⭐⭐⭐

```java
public class VisibilityPractice {

    private volatile boolean running = true;

    public void work() {
        while (running) {
            // Simulate work
        }

        System.out.println("Worker observed stop signal");
    }

    public void stop() {
        running = false;
    }

    public static void main(String[] args)
            throws InterruptedException {

        VisibilityPractice worker =
                new VisibilityPractice();

        Thread workerThread = new Thread(worker::work);
        workerThread.start();

        Thread.sleep(100);
        worker.stop();

        workerThread.join();
    }
}
```

`volatile` is appropriate here because the requirement is visibility of a simple state flag, not an atomic compound counter operation.

---

# 9. `volatile` Does NOT Fix `count++` ⭐⭐⭐⭐⭐

This is a classic interview trap:

```java
private volatile int count;

count++;
```

❌ Still not an atomic increment.

Why?

```text
volatile read
    ↓
add 1
    ↓
volatile write
```

Another thread can interleave between the read and write.

So:

```text
volatile → visibility/order
volatile ≠ general compound-operation atomicity
```

---

# 10. Practice Code — Volatile Counter Trap

```java
public class VolatileCounterTrap {

    private volatile int count;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}
```

This variable is visible across threads, but `count++` is still a read-modify-write sequence.

For a concurrent counter, consider:

```java
AtomicInteger
```

or suitable synchronization.

---

# 11. Ordering ⭐⭐⭐⭐⭐

**Ordering** concerns which relationships between operations are guaranteed to be observed by other threads.

The Java Memory Model does not simply promise that every source-code statement is globally observed in source order by every thread.

Instead, it defines formal ordering guarantees such as **happens-before**.

---

# 12. Why Ordering Matters

Consider:

```java
int data = 0;
boolean ready = false;
```

Thread 1:

```java
data = 42;
ready = true;
```

Thread 2:

```java
if (ready) {
    System.out.println(data);
}
```

The intended meaning is:

```text
data = 42
      ↓
ready = true
      ↓
Thread 2 sees ready
      ↓
Thread 2 sees data = 42
```

To make such cross-thread reasoning correct, the program needs an appropriate happens-before relationship.

---

# 13. Practice Code — Ordering and `volatile` ⭐⭐⭐⭐⭐

```java
public class OrderingPractice {

    private int data;
    private volatile boolean ready;

    public void writer() {
        data = 42;
        ready = true;
    }

    public void reader() {
        if (ready) {
            System.out.println(data);
        }
    }
}
```

The volatile write to `ready` and a subsequent read of `ready` establish the relevant happens-before relationship for actions before the write becoming visible through that synchronization mechanism.

Therefore, when the reader observes `ready == true`, the preceding write to `data` is ordered before the reader's subsequent actions under the Java Memory Model.

---

# 14. Happens-Before ⭐⭐⭐⭐⭐

The **happens-before** relation is one of the most important concepts for understanding visibility and ordering.

A happens-before relationship means that the Java Memory Model guarantees the effects of one action are ordered before another action in the required way.

Common examples include:

```text
Program order
     ↓
Lock unlock → subsequent lock on same monitor
     ↓
volatile write → subsequent read of same volatile variable
     ↓
Thread.start() → actions in started thread
     ↓
Actions in a thread → successful Thread.join() return
```

The complete rules are defined by the Java Memory Model.

---

# 15. `start()` and Ordering ⭐⭐⭐⭐⭐

Example:

```java
int value = 42;

Thread t = new Thread(() -> {
    System.out.println(value);
});

t.start();
```

The actions performed before calling `start()` happen-before actions in the started thread.

This gives a formal visibility/order guarantee for this publication pattern.

---

# 16. Practice Code — `start()` Happens-Before

```java
public class StartHappensBeforePractice {

    private int value;

    public static void main(String[] args)
            throws InterruptedException {

        StartHappensBeforePractice example =
                new StartHappensBeforePractice();

        example.value = 42;

        Thread thread = new Thread(() ->
                System.out.println("Value = " + example.value));

        thread.start();
        thread.join();
    }
}
```

The write to `value` before `start()` is ordered before the actions in the started thread.

---

# 17. `join()` and Ordering ⭐⭐⭐⭐⭐

If thread A successfully returns from:

```java
threadB.join();
```

then actions performed by thread B happen-before the successful return from `join()` in thread A.

Example:

```text
Thread B
  ↓
write result
  ↓
B terminates
  ↓
A returns from B.join()
  ↓
A reads result
```

---

# 18. Practice Code — `join()` Happens-Before

```java
public class JoinHappensBeforePractice {

    public static void main(String[] args)
            throws InterruptedException {

        final int[] result = new int[1];

        Thread worker = new Thread(() -> {
            result[0] = 100;
        });

        worker.start();
        worker.join();

        System.out.println("Result = " + result[0]);
    }
}
```

The successful return from `join()` provides the relevant happens-before relationship for the worker's actions.

---

# 19. Locking and Ordering ⭐⭐⭐⭐⭐

For the same monitor:

```java
synchronized (lock) {
    // critical section
}
```

An unlock on that monitor happens-before a subsequent lock on the same monitor.

This is one reason synchronization provides more than simple mutual exclusion: it also participates in the Java Memory Model's visibility and ordering guarantees.

---

# 20. Practice Code — Lock Visibility

```java
public class LockVisibilityPractice {

    private final Object lock = new Object();
    private int value;

    public void writer() {
        synchronized (lock) {
            value = 42;
        }
    }

    public void reader() {
        synchronized (lock) {
            System.out.println(value);
        }
    }
}
```

The same monitor coordinates access and establishes the relevant memory-ordering relationship between unlock and a subsequent lock.

---

# 21. Atomicity vs Visibility vs Ordering ⭐⭐⭐⭐⭐

| Concept | Main Question | Example Mechanism |
|---|---|---|
| **Atomicity** | Can the operation be observed as an indivisible action? | `AtomicInteger`, `synchronized` |
| **Visibility** | Can another thread observe the update? | `volatile`, locks |
| **Ordering** | What execution/observation order is guaranteed? | happens-before, locks, `volatile` |

### Easy memory trick

```text
Atomicity  → ONE operation
Visibility → SEE the update
Ordering   → ORDER between actions
```

---

# 22. One Mechanism Can Provide Multiple Properties ⭐⭐⭐⭐⭐

### `synchronized`

Can provide:

```text
Mutual exclusion
+ visibility
+ ordering
```

### `volatile`

Provides relevant:

```text
visibility
+ ordering guarantees
```

But not general compound-operation atomicity.

### `AtomicInteger`

Provides atomic operations and the associated memory semantics of its operations.

Do not reduce each mechanism to a single keyword like:

```text
volatile = visibility only
synchronized = locking only
```

That oversimplifies the Java Memory Model.

---

# 23. Atomicity Does Not Automatically Mean Full Thread Safety ⭐⭐⭐⭐⭐

Consider:

```java
AtomicInteger balance = new AtomicInteger(100);
```

Each individual atomic operation can be safe, but a business operation such as:

```text
check balance
   ↓
if sufficient
   ↓
deduct amount
```

may require an atomic compound design.

Use an operation such as `compareAndSet`, a suitable atomic update function, or explicit locking depending on the invariant.

---

# 24. Practice Code — CAS

```java
import java.util.concurrent.atomic.AtomicInteger;

public class CompareAndSetPractice {

    private final AtomicInteger balance =
            new AtomicInteger(100);

    public boolean withdraw(int amount) {
        while (true) {
            int current = balance.get();

            if (current < amount) {
                return false;
            }

            int updated = current - amount;

            if (balance.compareAndSet(current, updated)) {
                return true;
            }
        }
    }

    public int getBalance() {
        return balance.get();
    }
}
```

`compareAndSet` attempts to update only if the value is still the expected value. If another thread changed it first, the loop retries with the new value.

---

# 25. Common Interview Trap — `volatile` Flag vs Counter ⭐⭐⭐⭐⭐

### Good use

```java
private volatile boolean shutdown;
```

When threads need visibility of a simple state flag.

### Bad assumption

```java
private volatile int count;
count++;
```

`volatile` does not make the complete increment atomic.

### Better

```java
AtomicInteger count = new AtomicInteger();
```

or use synchronization where appropriate.

---

# 26. Common Mistake — "CPU Executes One Line at a Time"

Do not reason about concurrency only from source-code line order.

The Java Memory Model defines what cross-thread observations are guaranteed.

Correct reasoning uses:

```text
program order
happens-before
synchronization
volatile semantics
atomic operations
```

rather than assuming a single global execution order.

---

# 27. Common Mistake — "Visibility Means Immediate Physical Memory Flush"

Avoid simplistic explanations such as:

> "volatile immediately writes directly to RAM."

A better interview explanation is:

> `volatile` provides Java Memory Model visibility and ordering guarantees for accesses to that variable; it should be understood through the JMM rather than as a simplistic hardware-cache flush rule.

---

# 28. Common Mistake — "Atomic Means Lock-Free Everything"

Atomic classes often use hardware-supported atomic operations and compare-and-set techniques, but do not assume every atomic operation is universally lock-free at every implementation level or that a business transaction becomes atomic merely because it uses an atomic variable.

Focus on the API's documented semantics and the complete operation being designed.

---

# 29. Practical Comparison ⭐⭐⭐⭐⭐

| Scenario | Suitable Concept / Tool |
|---|---|
| Stop worker thread with a flag | `volatile` |
| Increment shared counter | `AtomicInteger` / `synchronized` |
| Protect multiple related fields | `synchronized` / `Lock` |
| Share immutable configuration | Immutability + safe construction/publication |
| Concurrent map updates | `ConcurrentHashMap` |
| Publish data before `start()` | `Thread.start()` happens-before |
| Wait for completed thread result | `Thread.join()` |
| Check-and-update atomic state | CAS / atomic update / lock |

---

# 30. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is atomicity?

Atomicity means an operation is performed as an indivisible action from the relevant concurrent-observation perspective.

### Q2. Is `i++` atomic in Java?

No. It is a compound read-modify-write operation.

### Q3. What is visibility?

Visibility concerns whether updates made by one thread can be observed by another thread under the Java Memory Model's guarantees.

### Q4. How does `volatile` help?

It provides specified visibility and ordering guarantees for accesses to the volatile variable.

### Q5. Does `volatile` make `i++` atomic?

No.

### Q6. What is ordering?

Ordering concerns the relationships between actions that the Java Memory Model guarantees, especially through happens-before rules.

### Q7. What is happens-before?

It is a formal JMM ordering relationship that provides guarantees about the ordering and visibility of effects between actions.

### Q8. Does `synchronized` provide only mutual exclusion?

No. It also provides relevant visibility and ordering guarantees through the monitor's synchronization semantics.

### Q9. What happens-before relationship does `start()` provide?

Actions before `Thread.start()` happen-before actions in the started thread.

### Q10. What happens-before relationship does `join()` provide?

Actions performed by the joined thread happen-before a successful return from `join()`.

### Q11. Is atomicity enough for thread safety?

No. Visibility, ordering and preservation of higher-level invariants can also matter.

### Q12. Give a simple example of visibility.

A worker thread observing a `volatile` shutdown flag written by another thread.

### Q13. Give a simple example of ordering.

A writer publishes data and then a synchronization action; a reader observes the synchronization action and is thereby given the required ordering/visibility relationship for the preceding write.

### Q14. What is the easiest way to remember the three concepts?

```text
Atomicity  → indivisible
Visibility → observable
Ordering   → happens-before relationship
```

---

# 31. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Atomicity, visibility and ordering are three important Java Memory Model concepts. Atomicity means an operation is performed as an indivisible action; for example, `count++` is not atomic because it is a read-modify-write sequence, while an appropriate `AtomicInteger` operation can be atomic. Visibility means that an update made by one thread becomes observable by another thread according to the JMM guarantees. `volatile` is commonly used when we need visibility of a simple state such as a shutdown flag, but it does not make compound operations atomic. Ordering is about the relationships between actions that the JMM guarantees, especially through happens-before. For example, actions before `Thread.start()` happen-before actions in the started thread, and actions in a thread happen-before a successful return from `join()`. `synchronized` provides mutual exclusion along with relevant visibility and ordering guarantees. So, when designing concurrent code, I don't ask only whether there is a lock; I ask whether the operation is atomic, whether updates are visible, and what ordering or happens-before relationship makes the design correct."**

---

# 32. Quick Revision ⭐⭐⭐⭐⭐

```text
Atomicity
→ indivisible operation
→ count++ is NOT atomic
→ AtomicInteger can provide atomic operations

Visibility
→ one thread observes another thread's update
→ volatile / synchronization can provide guarantees

Ordering
→ relationship between actions
→ happens-before is the key JMM concept

synchronized
→ mutual exclusion
→ visibility
→ ordering

volatile
→ visibility + ordering guarantees
→ NOT general compound-operation atomicity

start()
→ previous actions happen-before started thread

join()
→ joined thread actions happen-before successful join return
```

### Golden Rule

> **Atomicity answers "indivisible?", visibility answers "can I observe it?", and ordering answers "what relationship between actions is guaranteed?"**

---

# 33. Practice Checklist

- [x] Atomicity definition
- [x] `count++` and lost updates
- [x] `synchronized` and atomic critical sections
- [x] `AtomicInteger`
- [x] Visibility definition
- [x] `volatile` visibility example
- [x] `volatile` counter trap
- [x] Ordering definition
- [x] Happens-before
- [x] `start()` ordering
- [x] `join()` ordering
- [x] Lock ordering
- [x] CAS practice
- [x] Atomicity vs visibility vs ordering table
- [x] Common interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.18 — Thread Safety](../18-Thread-Safety/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.20 — Happens-Before Relationship**