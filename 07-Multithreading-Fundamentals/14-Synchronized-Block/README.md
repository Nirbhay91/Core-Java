# 7.14 — `synchronized` Block

## 🎯 Objective

Understand how a `synchronized` block works, why it is preferred when only part of a method is a critical section, how lock objects work, and the common interview traps around `this`, shared locks, static state, and synchronization scope.

---

## 1. What is a `synchronized` Block? ⭐⭐⭐⭐⭐

A `synchronized` block protects a specific region of code using an object's intrinsic monitor.

```java
synchronized (lock) {
    // critical section
}
```

The thread must acquire the monitor associated with `lock` before entering the block and releases it when leaving the block.

### Interview definition

> A synchronized block is a Java construct that provides mutual exclusion for a specific region of code by acquiring the intrinsic monitor of a chosen lock object.

---

# 2. Basic Structure

```java
synchronized (LOCK) {
    counter++;
}
```

Conceptually:

```text
Thread
  ↓
Acquire LOCK monitor
  ↓
Execute protected code
  ↓
Exit block
  ↓
Release LOCK monitor
```

If another thread tries to enter a synchronized block using the **same lock object**, it must wait to acquire that monitor.

---

# 3. Why Use a Synchronized Block? ⭐⭐⭐⭐⭐

Suppose only one line needs synchronization:

```java
public void process() {
    doExpensiveWork();

    synchronized (LOCK) {
        updateSharedState();
    }

    sendNotification();
}
```

Using a synchronized block can keep the critical section small instead of locking the entire method.

### Benefits

- Smaller critical section
- Less lock contention
- Better concurrency
- Ability to choose the lock object
- Protect only the state that requires coordination

---

# 4. `synchronized` Method vs Block ⭐⭐⭐⭐⭐

### Synchronized method

```java
public synchronized void increment() {
    counter++;
}
```

For an instance method, this is conceptually equivalent to:

```java
public void increment() {
    synchronized (this) {
        counter++;
    }
}
```

### Synchronized block

```java
public void increment() {
    doSomething();

    synchronized (LOCK) {
        counter++;
    }

    doSomethingElse();
}
```

### Key difference

```text
synchronized method → entire method uses its intrinsic monitor
synchronized block  → selected code uses chosen monitor
```

---

# 5. Practice Code — Basic Synchronized Block ⭐⭐⭐⭐⭐

```java
public class BasicSynchronizedBlockPractice {

    private int counter;
    private final Object LOCK = new Object();

    public void increment() {
        synchronized (LOCK) {
            counter++;
        }
    }

    public int getCounter() {
        synchronized (LOCK) {
            return counter;
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        BasicSynchronizedBlockPractice object =
                new BasicSynchronizedBlockPractice();

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

        System.out.println("Expected: 200000");
        System.out.println("Actual:   " + object.getCounter());
    }
}
```

Expected result:

```text
Expected: 200000
Actual:   200000
```

---

# 6. Why Use a Dedicated Lock Object? ⭐⭐⭐⭐⭐

A common pattern is:

```java
private final Object LOCK = new Object();
```

Then:

```java
synchronized (LOCK) {
    // protected state
}
```

This makes the synchronization policy explicit and avoids exposing the object's monitor through `this`.

### Important

The lock object should generally be:

- private
- final
- shared by all relevant accesses

Example:

```java
private final Object lock = new Object();
```

---

# 7. Why `final` for the Lock? ⭐⭐⭐⭐

Prefer:

```java
private final Object lock = new Object();
```

instead of allowing the lock reference to be reassigned.

If the lock reference changes while different threads use different objects as the lock, the synchronization protocol can break.

### Memory trick

> **One shared state → one consistent synchronization protocol.**

---

# 8. Synchronizing on `this` ⭐⭐⭐⭐⭐

You can write:

```java
synchronized (this) {
    counter++;
}
```

This uses the current object's intrinsic monitor.

It is conceptually equivalent to an instance synchronized method for the protected region.

### Example

```java
public void increment() {
    synchronized (this) {
        counter++;
    }
}
```

### Caution

Because `this` can be visible to external code, synchronizing on `this` can create unintended lock coupling if outside code also synchronizes on the same object.

A private lock is often a cleaner design when you control the class.

---

# 9. Practice Code — `this` Lock

```java
public class ThisLockPractice {

    private int counter;

    public void increment() {
        synchronized (this) {
            counter++;
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        ThisLockPractice object = new ThisLockPractice();

        Thread first = new Thread(() -> {
            for (int i = 0; i < 50_000; i++) {
                object.increment();
            }
        });

        Thread second = new Thread(() -> {
            for (int i = 0; i < 50_000; i++) {
                object.increment();
            }
        });

        first.start();
        second.start();

        first.join();
        second.join();

        System.out.println("Counter = " + object.counter);
    }
}
```

---

# 10. Same Lock = Mutual Exclusion ⭐⭐⭐⭐⭐

Consider:

```java
private final Object LOCK = new Object();

void methodA() {
    synchronized (LOCK) {
        // A
    }
}

void methodB() {
    synchronized (LOCK) {
        // B
    }
}
```

For the same object instance, `methodA()` and `methodB()` cannot execute their protected regions simultaneously because both use the same monitor.

---

# 11. Different Locks = No Coordination ⭐⭐⭐⭐⭐

Consider:

```java
private final Object LOCK_A = new Object();
private final Object LOCK_B = new Object();
```

Then:

```java
synchronized (LOCK_A) {
    // A
}
```

and:

```java
synchronized (LOCK_B) {
    // B
}
```

use different monitors.

Therefore they do not mutually exclude each other merely because both blocks use `synchronized`.

### Interview trap

> `synchronized` does not mean globally locked. The **lock object matters**.

---

# 12. Practice Code — Different Locks

```java
public class DifferentLocksPractice {

    private final Object LOCK_A = new Object();
    private final Object LOCK_B = new Object();

    public void methodA() {
        synchronized (LOCK_A) {
            System.out.println("A entered");
            sleep();
            System.out.println("A leaving");
        }
    }

    public void methodB() {
        synchronized (LOCK_B) {
            System.out.println("B entered");
            sleep();
            System.out.println("B leaving");
        }
    }

    private void sleep() {
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        DifferentLocksPractice object =
                new DifferentLocksPractice();

        Thread first = new Thread(object::methodA);
        Thread second = new Thread(object::methodB);

        first.start();
        second.start();

        first.join();
        second.join();
    }
}
```

The two blocks can execute concurrently because they use different monitors.

---

# 13. Narrow Critical Section ⭐⭐⭐⭐⭐

Prefer:

```java
public void process() {
    String data = calculateData();

    synchronized (LOCK) {
        cache.put(data, value);
    }

    sendResponse(data);
}
```

over unnecessarily doing:

```java
public void process() {
    synchronized (LOCK) {
        String data = calculateData();
        cache.put(data, value);
        sendResponse(data);
    }
}
```

**provided** the moved operations do not need the lock for correctness.

### Important

Smaller is not automatically better. The critical section must include the **complete logical operation required by the invariant**.

---

# 14. Practice Code — Narrow vs Wide Block

```java
public class NarrowCriticalSectionPractice {

    private final Object LOCK = new Object();
    private int count;

    public void process() {
        String data = expensiveCalculation();

        synchronized (LOCK) {
            count++;
        }

        System.out.println(data);
    }

    private String expensiveCalculation() {
        return "calculated";
    }

    public static void main(String[] args) {
        new NarrowCriticalSectionPractice().process();
    }
}
```

The expensive calculation does not unnecessarily hold the lock.

---

# 15. Check-Then-Act Must Be One Critical Section ⭐⭐⭐⭐⭐

Incorrect:

```java
if (inventory > 0) {
    synchronized (LOCK) {
        inventory--;
    }
}
```

The check occurs outside the protected region.

Better:

```java
synchronized (LOCK) {
    if (inventory > 0) {
        inventory--;
    }
}
```

The check and update form one logical operation.

---

# 16. Practice Code — Inventory Reservation ⭐⭐⭐⭐⭐

```java
public class InventoryBlockPractice {

    private int inventory = 1;
    private final Object LOCK = new Object();

    public boolean reserve() {
        synchronized (LOCK) {
            if (inventory > 0) {
                inventory--;
                return true;
            }
            return false;
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        InventoryBlockPractice inventory =
                new InventoryBlockPractice();

        Runnable task = () -> {
            boolean success = inventory.reserve();
            System.out.println(
                    Thread.currentThread().getName()
                            + " -> " + success);
        };

        Thread first = new Thread(task, "Customer-1");
        Thread second = new Thread(task, "Customer-2");

        first.start();
        second.start();

        first.join();
        second.join();
    }
}
```

With one item, the check and decrement are coordinated as one critical operation.

---

# 17. Synchronizing Static State with a Block ⭐⭐⭐⭐⭐

For shared static state, a common pattern is:

```java
synchronized (MyClass.class) {
    staticCounter++;
}
```

This uses the class object's monitor.

Equivalent conceptually to a static synchronized method for that class monitor:

```java
public static synchronized void increment() {
    staticCounter++;
}
```

---

# 18. Practice Code — Static Lock

```java
public class StaticLockPractice {

    private static int counter;

    public static void increment() {
        synchronized (StaticLockPractice.class) {
            counter++;
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        Thread first = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                increment();
            }
        });

        Thread second = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                increment();
            }
        });

        first.start();
        second.start();

        first.join();
        second.join();

        System.out.println(counter);
    }
}
```

---

# 19. Synchronized Block and Memory Visibility ⭐⭐⭐⭐⭐

Synchronization is not only about preventing simultaneous execution.

For the same monitor, releasing the monitor happens-before a subsequent acquisition of that monitor.

Therefore, synchronized blocks can be used to establish the necessary visibility and ordering between threads.

Example:

```java
synchronized (LOCK) {
    sharedValue = 100;
}
```

followed by another thread acquiring the same `LOCK`:

```java
synchronized (LOCK) {
    System.out.println(sharedValue);
}
```

The synchronization establishes the relevant happens-before relationship.

---

# 20. `sleep()` Inside a Synchronized Block ⭐⭐⭐⭐⭐

```java
synchronized (LOCK) {
    Thread.sleep(1000);
}
```

`sleep()` does **not** release `LOCK`'s monitor.

So another thread trying:

```java
synchronized (LOCK) {
    // ...
}
```

may remain blocked until the first thread leaves the synchronized block.

### Remember

```text
sleep() → keeps monitor
wait()  → releases monitor when waiting on that monitor
```

---

# 21. Practice Code — Sleep Holds Lock

```java
public class SleepInBlockPractice {

    private final Object LOCK = new Object();

    public void first() {
        synchronized (LOCK) {
            System.out.println("T1 entered");

            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            System.out.println("T1 leaving");
        }
    }

    public void second() {
        synchronized (LOCK) {
            System.out.println("T2 entered");
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        SleepInBlockPractice object =
                new SleepInBlockPractice();

        Thread first = new Thread(object::first, "T1");
        Thread second = new Thread(object::second, "T2");

        first.start();
        Thread.sleep(100);
        second.start();

        first.join();
        second.join();
    }
}
```

---

# 22. Lock Object Must Be Shared ⭐⭐⭐⭐⭐

Wrong:

```java
public void increment() {
    synchronized (new Object()) {
        counter++;
    }
}
```

Each invocation creates a different lock object, so threads do not coordinate through one common monitor.

Correct:

```java
private final Object LOCK = new Object();

public void increment() {
    synchronized (LOCK) {
        counter++;
    }
}
```

### Golden rule

> Threads that need mutual exclusion must synchronize on the same monitor.

---

# 23. Practice Code — Wrong Lock Example

```java
public class WrongLockPractice {

    private int counter;

    public void increment() {
        synchronized (new Object()) {
            counter++;
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        WrongLockPractice object = new WrongLockPractice();

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

        System.out.println("Actual: " + object.counter);
    }
}
```

The `new Object()` lock is different for each call, so it does not protect the shared counter across calls.

---

# 24. Locking on Public Mutable Objects ⚠️

Avoid using publicly accessible mutable objects as private synchronization locks when you control the design.

For example:

```java
synchronized (somePublicObject) {
    // ...
}
```

External code may also synchronize on that same object, creating unexpected lock coupling or contention.

Prefer:

```java
private final Object LOCK = new Object();
```

when a private intrinsic lock is appropriate.

---

# 25. Synchronized Block and Deadlock ⭐⭐⭐⭐

Synchronized blocks can participate in deadlocks when multiple locks are acquired in inconsistent order.

Example pattern:

```text
Thread A:
LOCK_A → LOCK_B

Thread B:
LOCK_B → LOCK_A
```

Possible result:

```text
A holds A and waits for B
B holds B and waits for A
```

The detailed deadlock topic is covered later in Chapter 7.

### Prevention idea

Use a consistent lock ordering when multiple locks must be acquired.

---

# 26. Synchronized Block vs `Lock` API ⭐⭐⭐⭐

### Intrinsic monitor

```java
synchronized (LOCK) {
    // ...
}
```

### Explicit lock

```java
lock.lock();
try {
    // ...
} finally {
    lock.unlock();
}
```

`Lock` provides additional capabilities such as:

- `tryLock()`
- interruptible lock acquisition
- optional fairness configuration with implementations such as `ReentrantLock`

Use the mechanism appropriate to the requirement.

---

# 27. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — Every synchronized block uses one global lock

❌ False. The expression inside the parentheses determines the monitor.

### Mistake 2 — `synchronized` on different objects coordinates threads

❌ False.

### Mistake 3 — Creating `new Object()` each time is a valid shared lock

❌ False for coordinating across calls.

### Mistake 4 — `sleep()` releases the lock

❌ False.

### Mistake 5 — Smaller critical section is always correct

❌ Not if it splits a required logical operation.

### Mistake 6 — Synchronizing a read automatically makes the entire class thread-safe

❌ Thread safety depends on all relevant shared-state access and invariants.

### Mistake 7 — `this` is always the best lock

❌ Not necessarily. A private lock can avoid external lock coupling.

### Mistake 8 — `volatile` can replace every synchronized block

❌ False. Visibility is not the same as mutual exclusion or atomicity of compound operations.

---

# 28. Interview Questions

### Q1. What is a synchronized block?

A specific code region protected by an intrinsic monitor associated with a chosen lock object.

### Q2. Why use a synchronized block instead of a synchronized method?

To protect only the required critical section and/or choose a specific lock object.

### Q3. What does `synchronized (this)` lock?

The current object's intrinsic monitor.

### Q4. What happens if two blocks use different lock objects?

They do not mutually exclude each other merely because both are synchronized.

### Q5. Why should a lock object usually be `private final`?

To keep the synchronization policy encapsulated and ensure the lock reference remains stable.

### Q6. Why is `synchronized (new Object())` usually wrong for shared state?

Each invocation creates a new monitor, so different threads do not coordinate on a common lock.

### Q7. Does a synchronized block provide memory visibility?

Yes, through monitor release/acquisition and the corresponding happens-before relationship when the same monitor is used.

### Q8. Does `sleep()` release the monitor?

No.

### Q9. Can synchronized blocks cause deadlock?

Yes, if multiple locks are acquired in an unsafe ordering.

### Q10. What is the main advantage of a synchronized block?

Fine-grained control over synchronization scope and lock choice.

### Q11. Can static state be protected with a synchronized block?

Yes, for example using `synchronized (MyClass.class)` or another appropriate shared lock.

### Q12. Does synchronization guarantee fairness?

No. Intrinsic monitors do not provide a general fairness guarantee.

---

# 29. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"A synchronized block protects a specific critical section using the intrinsic monitor of a chosen lock object. The main advantage over a synchronized method is finer control: I can protect only the code that accesses shared mutable state and can choose a dedicated private lock. For example, `synchronized (lock) { counter++; }` allows only one thread at a time to execute that protected region for the same lock object. The important point is that `synchronized` is not a global lock; two blocks using different lock objects do not mutually exclude each other. I generally prefer a `private final Object lock` when I need a private monitor, rather than creating a new object for every call or exposing `this` unnecessarily. Synchronized blocks also provide the relevant memory-visibility and happens-before guarantees for the same monitor. We should keep the critical section appropriately small to reduce contention, but never split a multi-step invariant such as check-then-act. Finally, multiple synchronized blocks can contribute to deadlock if multiple locks are acquired in inconsistent order."**

---

# 30. Quick Revision

```text
synchronized (LOCK) {
    critical section
}
        ↓
acquire LOCK monitor
        ↓
execute
        ↓
release LOCK monitor
```

### Golden Rules ⭐⭐⭐⭐⭐

```text
Same lock       → mutual exclusion
Different locks → no automatic coordination
private final lock → preferred dedicated monitor
new Object() per call → NOT a shared lock
this             → current object's monitor
Class.class      → class object's monitor
sleep()          → does NOT release monitor
synchronized    → mutual exclusion + visibility/happens-before
```

### Memory Trick

> **Block = choose the lock + choose the scope.**

---

# 31. Completion Checklist

- [x] Synchronized block definition
- [x] Basic syntax
- [x] Why use synchronized block
- [x] Method vs block
- [x] `this` lock
- [x] Dedicated private lock
- [x] `final` lock reference
- [x] Same lock vs different locks
- [x] Narrow critical section
- [x] Check-then-act
- [x] Static lock
- [x] Memory visibility / happens-before
- [x] `sleep()` behavior
- [x] Wrong lock example
- [x] Deadlock basics
- [x] Lock API comparison
- [x] Practice code
- [x] Interview questions
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.13 — `synchronized` Method](../13-Synchronized-Method/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.15 — Intrinsic Monitor / Object Lock**