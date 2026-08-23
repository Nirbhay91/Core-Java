# 7.17 — Reentrancy of `synchronized`

## 🎯 Objective

Understand why Java's intrinsic monitors are **reentrant**, how the same thread can acquire the same monitor multiple times, how reentrant acquisition works with synchronized methods and blocks, and why a thread does not deadlock itself when synchronized methods call other synchronized methods on the same object.

---

## 1. What is Reentrancy? ⭐⭐⭐⭐⭐

A lock is **reentrant** when a thread that already owns the lock can acquire the same lock again without blocking itself.

Java's intrinsic monitors used by `synchronized` are reentrant.

```java
public synchronized void methodA() {
    methodB();
}

public synchronized void methodB() {
    // same object's monitor
}
```

If the same thread enters `methodA()` and then calls `methodB()`, it can acquire the same object's monitor again.

### Interview definition

> **Reentrancy means the thread that already owns an intrinsic monitor can enter another synchronized region guarded by the same monitor without waiting for itself.**

---

# 2. Why Is Reentrancy Needed? ⭐⭐⭐⭐⭐

Without reentrancy, synchronized methods could easily deadlock when one synchronized method calls another synchronized method on the same object.

```text
Thread T1
   ↓
methodA() acquires monitor
   ↓
methodA() calls methodB()
   ↓
methodB() needs same monitor
   ↓
Reentrant monitor allows T1 to continue
```

The thread does not wait for itself.

---

# 3. Basic Reentrant Example ⭐⭐⭐⭐⭐

```java
public class ReentrantBasic {

    public synchronized void methodA() {
        System.out.println("Inside methodA");
        methodB();
    }

    public synchronized void methodB() {
        System.out.println("Inside methodB");
    }

    public static void main(String[] args) {
        ReentrantBasic object = new ReentrantBasic();
        object.methodA();
    }
}
```

### Flow

```text
T1 acquires object monitor
        ↓
methodA()
        ↓
methodB()
        ↓
T1 re-acquires same monitor
        ↓
methodB() completes
        ↓
methodA() completes
        ↓
monitor fully released
```

---

# 4. Reentrancy Is About the Same Thread ⭐⭐⭐⭐⭐

The important condition is:

> The **same thread** must already own the monitor.

Reentrancy does not mean multiple threads can simultaneously enter the same synchronized region.

```text
T1 owns monitor
T2 requests monitor
    ↓
T2 waits
```

But:

```text
T1 owns monitor
T1 requests same monitor again
    ↓
T1 allowed to continue
```

---

# 5. Practice Code — Same Thread Re-enters

```java
public class SameThreadReentrancy {

    public synchronized void first() {
        System.out.println("first(): " +
                Thread.currentThread().getName());
        second();
    }

    public synchronized void second() {
        System.out.println("second(): " +
                Thread.currentThread().getName());
        third();
    }

    public synchronized void third() {
        System.out.println("third(): " +
                Thread.currentThread().getName());
    }

    public static void main(String[] args) {
        SameThreadReentrancy object =
                new SameThreadReentrancy();

        object.first();
    }
}
```

The same thread enters the same object's monitor three times through nested synchronized methods.

---

# 6. Conceptual Reentrancy Count ⭐⭐⭐⭐⭐

For learning purposes, think of the monitor as tracking ownership and reentrant acquisition.

```text
T1 acquires monitor → acquisition level 1
T1 acquires again   → acquisition level 2
T1 acquires again   → acquisition level 3

T1 exits inner sync → level 2
T1 exits next sync  → level 1
T1 exits outer sync → level 0 → monitor available
```

The exact JVM implementation details should not be confused with a Java-level API counter; the important semantic rule is that the owning thread may re-enter and must exit the corresponding synchronized regions before the monitor becomes available to another thread.

---

# 7. Reentrancy with `synchronized` Blocks ⭐⭐⭐⭐⭐

Reentrancy is not limited to synchronized methods.

```java
private final Object lock = new Object();

public void outer() {
    synchronized (lock) {
        inner();
    }
}

public void inner() {
    synchronized (lock) {
        // same monitor
    }
}
```

The same thread can enter the nested block because it already owns `lock`'s monitor.

---

# 8. Practice Code — Nested Synchronized Blocks

```java
public class NestedSynchronizedBlocks {

    private final Object lock = new Object();

    public void outer() {
        synchronized (lock) {
            System.out.println("Outer acquired lock");
            inner();
            System.out.println("Outer releasing lock");
        }
    }

    private void inner() {
        synchronized (lock) {
            System.out.println("Inner acquired same lock");
        }
    }

    public static void main(String[] args) {
        new NestedSynchronizedBlocks().outer();
    }
}
```

---

# 9. Reentrant Method Calls Through `this` ⭐⭐⭐⭐⭐

An unqualified call such as:

```java
methodB();
```

from an instance method is effectively a call on the current object.

Therefore:

```java
public synchronized void methodA() {
    methodB();
}
```

where `methodB()` is also synchronized uses the same object's monitor.

This is one of the most common interview examples of reentrancy.

---

# 10. Practice Code — Synchronized Method Chain

```java
public class SynchronizedMethodChain {

    public synchronized void levelOne() {
        System.out.println("Level 1");
        levelTwo();
    }

    public synchronized void levelTwo() {
        System.out.println("Level 2");
        levelThree();
    }

    public synchronized void levelThree() {
        System.out.println("Level 3");
    }

    public static void main(String[] args) {
        new SynchronizedMethodChain().levelOne();
    }
}
```

### Key point

All three synchronized methods use the same `this` monitor.

---

# 11. Reentrancy vs Mutual Exclusion ⭐⭐⭐⭐⭐

These concepts are not opposites.

### Mutual exclusion

Different threads cannot simultaneously own the same monitor.

```text
T1 → owns monitor
T2 → waits
```

### Reentrancy

The owner thread can acquire that same monitor again.

```text
T1 → owns monitor
T1 → acquires same monitor again → allowed
```

### Together

> **One monitor owner at a time, but that owner may re-enter the monitor.**

---

# 12. Practice Code — Two Threads and Reentrancy

```java
public class ReentrancyWithTwoThreads {

    public synchronized void outer() {
        System.out.println("Outer: " +
                Thread.currentThread().getName());
        inner();
    }

    private synchronized void inner() {
        System.out.println("Inner: " +
                Thread.currentThread().getName());

        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        ReentrancyWithTwoThreads object =
                new ReentrancyWithTwoThreads();

        Thread t1 = new Thread(object::outer, "T1");
        Thread t2 = new Thread(object::outer, "T2");

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

T1 can re-enter the monitor in `inner()`, while T2 still has to wait for the monitor to become available.

---

# 13. Reentrancy with Instance Lock ⭐⭐⭐⭐⭐

For:

```java
public synchronized void work() {
}
```

the monitor is `this`.

If the same thread calls another synchronized instance method on the same object, the monitor is re-entered.

```text
object monitor
      ↑
      │
T1 → workA()
      │
      └──→ workB()
             │
             └──→ same monitor
```

---

# 14. Reentrancy with Explicit Object Lock ⭐⭐⭐⭐⭐

```java
private final Object lock = new Object();

void first() {
    synchronized (lock) {
        second();
    }
}

void second() {
    synchronized (lock) {
        // re-entry
    }
}
```

The monitor is reentrant because the same thread owns `lock` in both regions.

---

# 15. Reentrancy with Class Lock ⭐⭐⭐⭐⭐

Class monitors are intrinsic monitors too.

Therefore class-level synchronization is also reentrant.

```java
public static synchronized void first() {
    second();
}

public static synchronized void second() {
}
```

Both methods use the class monitor.

The same thread can enter `second()` while already holding that monitor through `first()`.

---

# 16. Practice Code — Static Reentrancy

```java
public class StaticReentrancy {

    public static synchronized void first() {
        System.out.println("Static first");
        second();
    }

    public static synchronized void second() {
        System.out.println("Static second");
    }

    public static void main(String[] args) {
        first();
    }
}
```

The same thread re-enters the class monitor.

---

# 17. Reentrancy Does NOT Mean Recursive Calls Are Safe ⭐⭐⭐⭐

Reentrancy allows a monitor to be acquired again, but it does not make recursion automatically correct.

This can still overflow the stack:

```java
public synchronized void recursive() {
    recursive();
}
```

The monitor is reentrant, so synchronization does not stop the recursive call. Eventually the call stack can overflow.

### Important

> Reentrancy solves lock self-deadlock; it does not solve logical recursion problems.

---

# 18. Practice Code — Reentrant but Recursive Failure

```java
public class ReentrantRecursionWarning {

    public synchronized void recursive() {
        recursive();
    }

    public static void main(String[] args) {
        new ReentrantRecursionWarning().recursive();
    }
}
```

This demonstrates that reentrancy does not prevent `StackOverflowError` caused by unbounded recursion.

---

# 19. Reentrancy and Exceptions ⭐⭐⭐⭐

If an exception causes the synchronized region to exit, Java's synchronization mechanism releases the monitor as part of leaving the synchronized method/block.

```java
public synchronized void work() {
    throw new RuntimeException("failure");
}
```

The monitor is not permanently held because of the exception.

This is another reason synchronized blocks/methods are safer than manually managed locking when simple mutual exclusion is sufficient.

---

# 20. Reentrancy vs `Lock` ⭐⭐⭐⭐

`ReentrantLock` is explicitly designed to provide reentrant locking.

```java
ReentrantLock lock = new ReentrantLock();
```

A thread that owns it can lock it again.

Intrinsic monitors are also reentrant:

```java
synchronized (lockObject) {
}
```

### Difference

```text
synchronized
→ intrinsic reentrant monitor

ReentrantLock
→ explicit reentrant lock API
```

---

# 21. Important Difference: Monitor Identity Matters ⭐⭐⭐⭐⭐

Reentrancy only applies when the same thread tries to acquire the **same monitor**.

These are different monitors:

```java
Object lock1 = new Object();
Object lock2 = new Object();
```

Even if the same thread owns `lock1`, entering:

```java
synchronized (lock2) {
}
```

is not re-entering `lock1`; it is acquiring another monitor.

---

# 22. Practice Code — Two Different Monitors

```java
public class TwoMonitorPractice {

    private final Object lock1 = new Object();
    private final Object lock2 = new Object();

    public void work() {
        synchronized (lock1) {
            System.out.println("lock1 acquired");

            synchronized (lock2) {
                System.out.println("lock2 acquired");
            }
        }
    }

    public static void main(String[] args) {
        new TwoMonitorPractice().work();
    }
}
```

The same thread acquires two different monitors. That is not reentrancy of one monitor; it is nested acquisition of two monitors.

---

# 23. Reentrancy and Deadlock ⭐⭐⭐⭐⭐

Reentrancy prevents this particular self-deadlock:

```text
T1 owns Lock A
T1 requests Lock A again
```

Because A is reentrant, T1 is allowed to continue.

But reentrancy does **not** prevent classic multi-thread deadlock:

```text
T1 owns A → waits for B
T2 owns B → waits for A
```

So:

```text
reentrant lock
        ≠
no deadlocks
```

---

# 24. Practice Code — Reentrancy Does Not Prevent Deadlock

```java
public class ReentrancyDoesNotPreventDeadlock {

    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void taskOne() {
        synchronized (lockA) {
            synchronized (lockB) {
                System.out.println("Task one");
            }
        }
    }

    public void taskTwo() {
        synchronized (lockB) {
            synchronized (lockA) {
                System.out.println("Task two");
            }
        }
    }
}
```

If two threads execute these methods concurrently, they can create a circular wait. Reentrancy does not eliminate that risk.

---

# 25. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1

> Reentrancy means multiple threads can enter the synchronized block.

❌ Wrong. Only the owning thread can re-enter its monitor.

### Mistake 2

> `synchronized` is not reentrant.

❌ Wrong. Intrinsic monitors are reentrant.

### Mistake 3

> Reentrancy means deadlocks are impossible.

❌ Wrong. Multiple-monitor deadlocks are still possible.

### Mistake 4

> Any nested synchronized block is reentrant.

❌ Not necessarily. Reentrancy specifically means re-acquiring the same monitor by its owner.

### Mistake 5

> Two different lock objects are the same monitor.

❌ Wrong. Monitor identity follows object identity.

### Mistake 6

> Reentrancy makes recursive code safe.

❌ Wrong. Recursive code can still cause `StackOverflowError`.

### Mistake 7

> A second thread can benefit from the first thread's reentrant ownership.

❌ No. Reentrancy is a property of the current owning thread's ability to reacquire the same monitor.

---

# 26. Interview Questions

### Q1. What is reentrancy in Java?

It means a thread that already owns a monitor can acquire that same monitor again without blocking itself.

### Q2. Is `synchronized` reentrant?

Yes. Java intrinsic monitors are reentrant.

### Q3. Why is reentrancy useful?

It allows synchronized methods/blocks to call other synchronized methods/blocks protected by the same monitor without self-deadlocking.

### Q4. Does reentrancy allow two threads to enter simultaneously?

No.

### Q5. What happens when a synchronized method calls another synchronized method on the same object?

The same thread re-enters the object's monitor.

### Q6. Is a nested synchronized block always reentrant?

Only if the nested region uses the same monitor already owned by the thread.

### Q7. Are static synchronized methods reentrant?

Yes. They use the class monitor, which is also reentrant.

### Q8. Does reentrancy prevent deadlock?

No. It prevents self-deadlock from reacquiring the same monitor, but deadlocks involving multiple monitors can still occur.

### Q9. Does reentrancy make recursion safe?

No. Recursive calls can still exhaust the stack.

### Q10. What is the difference between reentrancy and mutual exclusion?

Mutual exclusion limits ownership of a monitor to one thread at a time; reentrancy allows that owning thread to acquire the same monitor again.

### Q11. Is `ReentrantLock` also reentrant?

Yes.

### Q12. When is synchronization reentrant?

When the same thread that currently owns the monitor attempts to acquire that same monitor again.

---

# 27. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Java's intrinsic monitors are reentrant. Reentrancy means that if a thread already owns an object's monitor, it can acquire that same monitor again without blocking itself. This is important because a synchronized method can call another synchronized method on the same object. The outer method acquires the `this` monitor, and the inner method can re-enter that same monitor. The same concept applies to synchronized blocks and class-level synchronization. Reentrancy does not mean multiple threads can enter simultaneously; other threads still wait for the monitor. It also does not eliminate all deadlocks because deadlocks involving multiple different monitors are still possible. So the key distinction is: mutual exclusion controls different threads, while reentrancy allows the current owner to reacquire the same monitor."**

---

# 28. Quick Revision ⭐⭐⭐⭐⭐

```text
Thread T1
   ↓
acquires Monitor A
   ↓
T1 acquires Monitor A again
   ↓
ALLOWED → reentrancy
```

```text
Thread T1
   ↓
owns Monitor A

Thread T2
   ↓
requests Monitor A

T2 → waits
```

### Golden Rules

```text
synchronized → reentrant
same thread + same monitor → re-entry allowed
other thread + same monitor → waits
same monitor → required for reentrancy
multiple monitors → deadlock still possible
recursion → StackOverflowError still possible
```

### Memory Trick

> **"Lock owner can enter the same lock again; other threads cannot."**

---

# 29. Practice Checklist

- [x] Reentrancy definition
- [x] Why synchronized is reentrant
- [x] Same thread vs different thread
- [x] Synchronized method reentrancy
- [x] Synchronized block reentrancy
- [x] Instance monitor reentrancy
- [x] Class monitor reentrancy
- [x] Reentrancy vs mutual exclusion
- [x] Reentrancy vs recursion
- [x] Reentrancy vs deadlock
- [x] Multiple monitor scenario
- [x] `ReentrantLock` comparison
- [x] Practice code
- [x] Common mistakes
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.16 — Class Lock vs Object Lock](../16-Class-Lock-vs-Object-Lock/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.18 — Thread Safety**