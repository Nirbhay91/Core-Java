# 7.11 — Race Condition

## 🎯 Objective

Understand why multiple threads accessing shared mutable state can produce incorrect or unpredictable results, how a **race condition** occurs, and how to demonstrate and fix it.

---

## 1. What is a Race Condition? ⭐⭐⭐⭐⭐

A **race condition** occurs when the correctness of a program depends on the timing or interleaving of multiple threads accessing shared state.

Typical situation:

```text
Thread A ──┐
           ├──→ shared mutable state
Thread B ──┘
```

If the required operation is not properly coordinated, the final result can depend on which thread executes certain steps first.

### Interview definition

> A race condition occurs when multiple threads concurrently access shared state and at least one thread modifies it, while the program's correctness depends on the timing/interleaving of those accesses.

---

## 2. Simple Example ⭐⭐⭐⭐⭐

Suppose:

```java
counter++;
```

looks like one operation, but conceptually it involves:

```text
1. Read counter
2. Add 1
3. Write counter
```

If two threads execute it concurrently:

```text
Initial counter = 0

Thread A: read 0
Thread B: read 0
Thread A: write 1
Thread B: write 1

Expected = 2
Actual   = 1
```

One increment is lost.

This is a classic **lost update** caused by a race condition.

---

## 3. Race Condition Requires Shared State? ⭐⭐⭐⭐⭐

The classic data race/race-condition scenario involves shared mutable state.

For example:

```java
class Counter {
    int count;
}
```

If multiple threads modify the same `Counter` instance:

```text
Thread A ──→ counter.count
Thread B ──→ counter.count
```

there is a concurrency problem if access is not properly coordinated.

If each thread has its own independent variable, there is no shared-state race on that variable.

---

## 4. Why `counter++` Is Not Thread-Safe ⭐⭐⭐⭐⭐

```java
counter++;
```

is a read-modify-write operation.

Conceptually:

```java
int temp = counter;
temp = temp + 1;
counter = temp;
```

Multiple threads can interleave these steps.

Therefore, `counter++` is not automatically atomic merely because it appears as one Java statement.

---

## 5. Practice Code — Reproduce a Race Condition ⭐⭐⭐⭐⭐

Create:

`RaceConditionPractice.java`

```java
public class RaceConditionPractice {

    private static int counter = 0;

    public static void main(String[] args)
            throws InterruptedException {

        Thread first = new Thread(() -> increment(), "worker-1");
        Thread second = new Thread(() -> increment(), "worker-2");

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

### Expected

```text
Expected: 200000
Actual:   often less than 200000
```

The exact result is timing-dependent. A particular run may even appear correct; that does **not** make the code thread-safe.

---

## 6. Why the Bug May Not Appear Every Time ⭐⭐⭐⭐⭐

Concurrency bugs are often nondeterministic.

The scheduler may produce different interleavings on different runs.

```text
Run 1 → 198731
Run 2 → 200000
Run 3 → 199412
Run 4 → 197856
```

The exact values are only examples.

### Important interview point

> A race condition does not have to fail on every execution.

It is enough that an unsafe interleaving is possible and the program's correctness depends on timing.

---

## 7. The Critical Interleaving ⭐⭐⭐⭐⭐

Assume:

```text
counter = 10
```

Two threads execute:

```text
Thread A                  Thread B
--------                  --------
read counter = 10
                          read counter = 10
calculate 11
                          calculate 11
write 11
                          write 11
```

Final value:

```text
11
```

Expected:

```text
12
```

This is a **lost update**.

---

## 8. Race Condition vs Data Race

These terms are related but not always identical.

### Data race

A data race generally refers to conflicting unsynchronized accesses to the same memory location, where at least one access is a write, under the language's memory model.

### Race condition

A race condition is broader: program correctness depends on the relative timing/order of concurrent operations.

### Interview-safe distinction

> A data race is a specific memory-access problem; a race condition is a broader correctness problem caused by timing/interleaving.

Not every race condition must be described as a data race.

---

## 9. Race Condition vs Thread Safety ⭐⭐⭐⭐⭐

| Race Condition | Thread Safety |
|---|---|
| A concurrency bug/problem | A property/design goal |
| Usually caused by unsafe concurrent access | Means behavior remains correct under supported concurrent use |
| Depends on timing/interleaving | Achieved through appropriate design/synchronization |
| Can cause lost updates/inconsistent state | Prevents or safely handles such problems |

### Memory trick

```text
Race condition → problem
Thread safety   → desired property
```

---

## 10. Fix Using `synchronized` ⭐⭐⭐⭐⭐

```java
public class SynchronizedCounterPractice {

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

    private static synchronized void increment() {
        for (int i = 0; i < 100_000; i++) {
            counter++;
        }
    }
}
```

The `synchronized` method ensures that only one thread at a time executes that critical section for the class monitor.

This is one simple way to protect the shared update.

---

## 11. Better Granularity: Synchronize the Critical Update ⭐⭐⭐⭐⭐

Instead of synchronizing a larger operation unnecessarily, identify the actual critical section.

```java
private static void increment() {
    for (int i = 0; i < 100_000; i++) {
        synchronized (RaceConditionPractice.class) {
            counter++;
        }
    }
}
```

This protects each increment, although it introduces synchronization overhead for every iteration.

For production code, choose synchronization scope based on correctness and performance requirements rather than blindly synchronizing everything.

---

## 12. Fix Using `AtomicInteger` ⭐⭐⭐⭐⭐

For a simple atomic counter:

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounterPractice {

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

`AtomicInteger` provides atomic operations suitable for this simple counter use case.

---

## 13. Race Condition Is Not Only About Counters ⭐⭐⭐⭐⭐

The problem can occur with any shared mutable state.

Examples:

- Bank balance
- Inventory quantity
- Seat availability
- Shared collection
- Order status
- Account state
- Cache updates
- Shared configuration/state

Example:

```text
Thread A → checks seat available
Thread B → checks seat available
Thread A → books seat
Thread B → books same seat
```

If the check-and-update operation is not properly coordinated, both threads may believe they successfully reserved the same resource.

---

## 14. Check-Then-Act Race ⭐⭐⭐⭐⭐

A common race pattern is:

```java
if (balance >= amount) {
    balance -= amount;
}
```

The problem is that the operation consists of multiple steps:

```text
CHECK
  ↓
ACT
```

Two threads can both pass the check before either performs the update.

Therefore the required operation may need to be made atomic as a whole.

---

## 15. Practice Code — Check-Then-Act

```java
public class CheckThenActPractice {

    private static int seats = 1;

    public static void main(String[] args)
            throws InterruptedException {

        Thread user1 = new Thread(() -> book("User-1"));
        Thread user2 = new Thread(() -> book("User-2"));

        user1.start();
        user2.start();

        user1.join();
        user2.join();

        System.out.println("Remaining seats: " + seats);
    }

    private static void book(String user) {
        if (seats > 0) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }

            seats--;
            System.out.println(user + " booked a seat");
        }
    }
}
```

This demonstrates why separating the check from the update can be unsafe under concurrency.

---

## 16. Fix Check-Then-Act

```java
private static synchronized void book(String user) {
    if (seats > 0) {
        seats--;
        System.out.println(user + " booked a seat");
    }
}
```

Now the check and update happen while holding the same monitor.

The operation becomes protected as one critical section.

---

## 17. Race Condition and Atomicity ⭐⭐⭐⭐⭐

A key question is:

> Is the whole logical operation atomic?

For example:

```java
counter++;
```

is not atomic as a read-modify-write operation.

But an operation such as:

```java
AtomicInteger.incrementAndGet();
```

provides an atomic increment operation.

### Important

Atomicity alone is not always enough for a larger multi-step invariant. You must choose a synchronization mechanism that protects the complete logical operation.

---

## 18. Race Condition and `volatile` ⭐⭐⭐⭐⭐

A common interview mistake is:

> "Make the counter `volatile` and the race condition is solved."

❌ Not generally true.

```java
volatile int counter;
```

`volatile` provides visibility and ordering guarantees according to the Java Memory Model, but it does not turn compound operations such as:

```java
counter++;
```

into atomic operations.

So:

```text
volatile → visibility
synchronized / atomic operation → appropriate atomicity
```

The exact design depends on the problem.

---

## 19. Race Condition and Immutability

Immutable objects reduce concurrency problems because their state cannot be changed after construction.

For example:

```java
String value = "Java";
```

A shared immutable object can generally be safely observed by multiple threads without coordinating mutations to its internal state.

However, immutable references do not automatically make every surrounding operation thread-safe.

---

## 20. Race Condition and Local Variables ⭐⭐⭐⭐

Local variables belong to a thread's execution context.

Example:

```java
private static void work() {
    int local = 0;
    local++;
}
```

If each thread executes its own invocation, each has its own `local` variable.

The same shared instance field is different:

```java
private int count;
```

Multiple threads using the same object can concurrently access the same field.

---

## 21. How to Detect Race Conditions

Useful approaches include:

1. Stress testing
2. Running concurrent tests repeatedly
3. Increasing thread count
4. Repeating operations many times
5. Code review for shared mutable state
6. Concurrency testing tools/profilers
7. Looking for non-atomic check-then-act logic
8. Reviewing synchronization and memory-visibility assumptions

### Important

A test passing once does not prove the absence of a race condition.

---

# 22. Practice Code — Stress Test ⭐⭐⭐⭐⭐

```java
public class RaceStressPractice {

    private static int counter;

    public static void main(String[] args)
            throws InterruptedException {

        int threadCount = 8;
        int incrementsPerThread = 100_000;

        Thread[] threads = new Thread[threadCount];

        for (int i = 0; i < threadCount; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0;
                     j < incrementsPerThread;
                     j++) {
                    counter++;
                }
            });

            threads[i].start();
        }

        for (Thread thread : threads) {
            thread.join();
        }

        long expected =
                (long) threadCount * incrementsPerThread;

        System.out.println("Expected: " + expected);
        System.out.println("Actual:   " + counter);
    }
}
```

This is useful for demonstrating that increasing concurrency can make a timing-sensitive bug easier to observe, but it still does not guarantee a failure on every run.

---

# 23. Race Condition vs Deadlock ⭐⭐⭐⭐⭐

These are different problems.

### Race condition

```text
Wrong result because of unsafe timing/interleaving
```

### Deadlock

```text
Threads wait forever for resources held by each other
```

Example conceptual difference:

```text
Race:
A and B modify shared state incorrectly

Deadlock:
A waits for B's lock
B waits for A's lock
```

---

# 24. Race Condition vs Starvation

### Race condition

Correctness depends on unsafe concurrent interleaving.

### Starvation

A thread repeatedly fails to obtain the CPU/resource it needs because other threads keep getting access.

Again, these are different concurrency problems.

---

# 25. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — `counter++` is atomic

❌ False.

It is a compound read-modify-write operation.

### Mistake 2 — `volatile` solves every race

❌ False.

`volatile` does not make `counter++` atomic.

### Mistake 3 — Race condition always reproduces

❌ False.

It can be timing-dependent and intermittent.

### Mistake 4 — `synchronized` only provides mutual exclusion

Oversimplified.

Synchronization also establishes important memory-visibility/happens-before guarantees.

### Mistake 5 — `yield()` fixes a race

❌ False.

`yield()` is only a scheduler hint.

### Mistake 6 — `sleep()` fixes a race

❌ False.

Changing timing may hide or expose a bug but does not make shared state safe.

### Mistake 7 — `join()` fixes the shared update

❌ False.

`join()` can ensure the caller waits for completion, but it does not make concurrent modifications to shared state atomic.

---

# 26. Interview Questions

### Q1. What is a race condition?

A race condition occurs when multiple threads access shared state and program correctness depends on their timing/interleaving.

### Q2. Give a simple example.

Two threads execute `counter++` on the same shared counter, causing lost updates.

### Q3. Why is `counter++` unsafe?

Because it is a compound read-modify-write operation and can be interleaved between threads.

### Q4. How can you fix a race condition?

Depending on the problem, use synchronization, locks, atomic variables, immutable design, confinement, or higher-level concurrency utilities.

### Q5. Does `volatile` solve `counter++`?

No. It helps with visibility but does not make the increment atomic.

### Q6. Does `yield()` solve race conditions?

No. It is only a scheduler hint.

### Q7. Does `sleep()` solve race conditions?

No. It only changes timing.

### Q8. Does `join()` make shared data thread-safe?

No. It only provides waiting for target-thread termination.

### Q9. What is a lost update?

When concurrent read-modify-write operations overwrite each other's results, causing some updates to disappear.

### Q10. What is check-then-act?

A pattern where a thread checks a condition and later acts on it; without atomic coordination, another thread can change the state between those steps.

### Q11. Race condition vs data race?

A data race is a specific unsynchronized conflicting-memory-access situation; race condition is the broader correctness problem caused by timing/interleaving.

### Q12. Race condition vs deadlock?

A race condition can produce incorrect results due to unsafe interleaving; deadlock causes threads to wait indefinitely for each other/resources.

---

# 27. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"A race condition occurs when multiple threads concurrently access shared mutable state and the correctness of the result depends on their timing or interleaving. A classic example is `counter++`. Although it looks like one statement, it is conceptually a read, modify and write operation. Two threads can read the same old value and then overwrite each other's updates, producing a lost update. The problem is nondeterministic, so the program may appear correct in some runs and fail in others. We can prevent the race depending on the use case by protecting the complete critical section with `synchronized` or locks, using atomic classes such as `AtomicInteger` for suitable atomic operations, or designing the state to avoid shared mutable access. `volatile` alone does not make compound operations atomic, and `sleep()`, `yield()` or `join()` should not be treated as race-condition fixes. In short, race condition is a correctness problem caused by unsafe concurrent interleaving of shared state."**

---

# 28. Quick Revision

```text
Multiple Threads
       ↓
Shared Mutable State
       ↓
Concurrent Access
       ↓
Unsafe Interleaving
       ↓
Race Condition
       ↓
Wrong / Inconsistent Result
```

### Golden Rules ⭐⭐⭐⭐⭐

```text
counter++ ≠ atomic
volatile ≠ atomic compound operation
sleep() ≠ synchronization
yield() ≠ synchronization
join() ≠ synchronization
```

### Fix options

```text
synchronized
Lock
AtomicInteger / AtomicLong
Immutable design
Thread confinement
Concurrent collections
Higher-level concurrency utilities
```

### Memory Trick

> **Race condition = shared mutable state + unsafe concurrency + timing-dependent correctness.**

---

# 29. Completion Checklist

- [x] Race condition definition
- [x] Shared mutable state
- [x] `counter++` read-modify-write
- [x] Lost update
- [x] Nondeterministic behavior
- [x] Check-then-act race
- [x] Race condition vs data race
- [x] Race condition vs thread safety
- [x] `synchronized` fix
- [x] `AtomicInteger` fix
- [x] `volatile` limitation
- [x] Immutability/confinement discussion
- [x] Stress-test practice
- [x] Multiple-thread practice
- [x] Interview traps
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.10 — `yield()`](../10-yield/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.12 — Critical Section**