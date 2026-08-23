# 7.30 — Deadlock

## 🎯 Objective

Understand **deadlock** in Java: what it is, how it happens, the four Coffman conditions, classic code examples, detection, prevention, avoidance strategies, and interview scenarios.

> **Golden rule:** Deadlock occurs when threads wait indefinitely for resources/locks held by each other, so none of them can make progress.

---

## 1. What Is Deadlock? ⭐⭐⭐⭐⭐

A **deadlock** is a situation where two or more threads are permanently blocked because each thread is waiting for a resource/lock held by another thread.

Classic situation:

```text
Thread-1 holds Lock-A
Thread-1 waits for Lock-B

Thread-2 holds Lock-B
Thread-2 waits for Lock-A

        ↓
   Circular waiting
        ↓
      DEADLOCK
```

Neither thread can proceed.

---

# 2. Simple Real-World Example

Imagine:

```text
Person A has Pen
Person B has Paper

A needs Paper → waits for B
B needs Pen   → waits for A
```

Both wait forever.

Java locks can create exactly the same situation.

---

# 3. Classic Java Deadlock ⭐⭐⭐⭐⭐

```java
public class DeadlockDemo {

    private static final Object LOCK_A = new Object();
    private static final Object LOCK_B = new Object();

    public static void main(String[] args) {

        Thread thread1 = new Thread(() -> {
            synchronized (LOCK_A) {
                System.out.println("Thread-1 acquired LOCK_A");

                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }

                synchronized (LOCK_B) {
                    System.out.println("Thread-1 acquired LOCK_B");
                }
            }
        }, "Thread-1");

        Thread thread2 = new Thread(() -> {
            synchronized (LOCK_B) {
                System.out.println("Thread-2 acquired LOCK_B");

                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }

                synchronized (LOCK_A) {
                    System.out.println("Thread-2 acquired LOCK_A");
                }
            }
        }, "Thread-2");

        thread1.start();
        thread2.start();
    }
}
```

### What happens?

```text
Thread-1 → LOCK_A → waits for LOCK_B
Thread-2 → LOCK_B → waits for LOCK_A
```

Both remain blocked.

---

# 4. Why Does `sleep()` Appear in the Demo?

The sleep is only used to make the timing easier to reproduce.

It gives both threads an opportunity to acquire their first lock before trying to acquire the second lock.

Important:

> `sleep()` itself does **not** create the deadlock. The circular lock acquisition does.

---

# 5. Four Coffman Conditions ⭐⭐⭐⭐⭐

Deadlock requires all four classic conditions:

### 1. Mutual Exclusion

A resource can be held by only one thread at a time.

### 2. Hold and Wait

A thread holds one resource while waiting for another.

### 3. No Preemption

A resource cannot simply be forcibly taken away from the thread holding it.

### 4. Circular Wait

A circular dependency exists:

```text
T1 waits for T2
T2 waits for T3
T3 waits for T1
```

### Interview memory trick

```text
M → Mutual Exclusion
H → Hold and Wait
N → No Preemption
C → Circular Wait
```

**MHNC = four Coffman conditions.**

---

# 6. Resource Allocation Graph Concept ⭐⭐⭐⭐

A deadlock can be visualized as a cycle in a resource dependency graph.

```text
Thread-1 → Lock-B
   ↑         |
   |         ↓
 Lock-A ← Thread-2
```

More directly:

```text
T1 holds A → waits B
T2 holds B → waits A
```

The cycle represents circular waiting.

---

# 7. Lock Ordering — Most Practical Prevention ⭐⭐⭐⭐⭐

The easiest way to prevent many lock-ordering deadlocks is to always acquire locks in the same global order.

Bad:

```text
Thread-1: A → B
Thread-2: B → A
```

Good:

```text
Thread-1: A → B
Thread-2: A → B
```

No circular ordering is created.

---

# 8. Practice Code — Fixed Lock Ordering ⭐⭐⭐⭐⭐

```java
public class FixedLockOrderDemo {

    private static final Object LOCK_A = new Object();
    private static final Object LOCK_B = new Object();

    static void work() {
        synchronized (LOCK_A) {
            System.out.println(Thread.currentThread().getName()
                    + " acquired A");

            synchronized (LOCK_B) {
                System.out.println(Thread.currentThread().getName()
                        + " acquired B");
            }
        }
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(FixedLockOrderDemo::work, "T1");
        Thread t2 = new Thread(FixedLockOrderDemo::work, "T2");

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

Both threads follow:

```text
A → B
```

so circular wait is avoided.

---

# 9. Deadlock vs Race Condition ⭐⭐⭐⭐⭐

| Deadlock | Race Condition |
|---|---|
| Threads wait indefinitely | Result depends on timing/interleaving |
| No progress | Incorrect/unpredictable result possible |
| Usually caused by lock dependency cycle | Usually caused by unsynchronized shared state |
| Application may appear frozen | Application may continue but produce wrong data |

Example:

```text
Race condition → wrong result
Deadlock       → no progress
```

---

# 10. Deadlock vs Starvation ⭐⭐⭐⭐⭐

### Deadlock

A group of threads wait on each other indefinitely.

### Starvation

A thread keeps getting denied the resources/CPU time it needs because other threads keep getting access.

```text
Deadlock:
T1 ↔ T2
both stuck

Starvation:
T1 needs resource
T2/T3 repeatedly get resource
T1 waits indefinitely
```

---

# 11. Deadlock vs Livelock ⭐⭐⭐⭐⭐

### Deadlock

Threads are blocked and do not progress.

### Livelock

Threads remain active but keep responding to each other without making useful progress.

```text
Deadlock → blocked + no progress
Livelock  → active + no useful progress
```

---

# 12. Nested Locks and Deadlock ⭐⭐⭐⭐⭐

Nested synchronization increases deadlock risk:

```java
synchronized (A) {
    synchronized (B) {
        // work
    }
}
```

If another thread uses the reverse order:

```java
synchronized (B) {
    synchronized (A) {
        // work
    }
}
```

there is a potential cycle.

---

# 13. Practice Code — Nested Lock Problem ⭐⭐⭐⭐⭐

```java
public class NestedLockDeadlock {

    private static final Object A = new Object();
    private static final Object B = new Object();

    public static void main(String[] args) {

        Thread t1 = new Thread(() -> {
            synchronized (A) {
                System.out.println("T1 -> A");

                synchronized (B) {
                    System.out.println("T1 -> B");
                }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (B) {
                System.out.println("T2 -> B");

                synchronized (A) {
                    System.out.println("T2 -> A");
                }
            }
        });

        t1.start();
        t2.start();
    }
}
```

This is structurally vulnerable to deadlock because the lock ordering differs.

---

# 14. Can `wait()` Cause Deadlock? ⭐⭐⭐⭐

`wait()` itself is not a deadlock.

A thread calling `wait()` releases the monitor and waits for notification/interruption/timeout.

However, application-level coordination can still produce indefinite waiting if the required condition/event never occurs.

Important distinction:

```text
wait()       → intentional waiting + releases monitor
Deadlock     → circular resource dependency / no progress
```

---

# 15. `sleep()` and Deadlock ⭐⭐⭐⭐⭐

`sleep()` does **not release an intrinsic monitor**.

Therefore this is dangerous:

```java
synchronized (LOCK_A) {
    Thread.sleep(5000);
    synchronized (LOCK_B) {
        // work
    }
}
```

If another thread holds `LOCK_B` and waits for `LOCK_A`, a deadlock can occur.

But remember:

> `sleep()` is not the root cause. Holding locks while acquiring other locks creates the dependency problem.

---

# 16. Deadlock Prevention Strategies ⭐⭐⭐⭐⭐

### Strategy 1 — Consistent Lock Ordering

Always acquire multiple locks in a predefined order.

### Strategy 2 — Reduce Lock Scope

Hold locks for the shortest practical duration.

### Strategy 3 — Avoid Nested Locks Where Possible

Nested lock acquisition increases dependency complexity.

### Strategy 4 — Use Higher-Level Concurrency Utilities

Prefer abstractions such as:

- `java.util.concurrent` locks/utilities
- concurrent collections
- executors
- higher-level coordination primitives

### Strategy 5 — Timed Lock Acquisition

With `ReentrantLock`, `tryLock(timeout, unit)` can avoid waiting forever.

---

# 17. `ReentrantLock.tryLock()` ⭐⭐⭐⭐⭐

Unlike intrinsic `synchronized`, `ReentrantLock` provides APIs for timed lock acquisition.

```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // critical section
    } finally {
        lock.unlock();
    }
} else {
    // could not acquire lock in time
}
```

This can be useful when the application needs a bounded waiting policy.

---

# 18. Practice Code — Timed Lock Acquisition ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class TryLockDemo {

    private static final ReentrantLock LOCK_A = new ReentrantLock();
    private static final ReentrantLock LOCK_B = new ReentrantLock();

    static void work() throws InterruptedException {
        if (!LOCK_A.tryLock(1, TimeUnit.SECONDS)) {
            System.out.println(Thread.currentThread().getName()
                    + " could not acquire A");
            return;
        }

        try {
            if (!LOCK_B.tryLock(1, TimeUnit.SECONDS)) {
                System.out.println(Thread.currentThread().getName()
                        + " could not acquire B");
                return;
            }

            try {
                System.out.println(Thread.currentThread().getName()
                        + " acquired A and B");
            } finally {
                LOCK_B.unlock();
            }
        } finally {
            LOCK_A.unlock();
        }
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> {
            try {
                work();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "T1");

        Thread t2 = new Thread(() -> {
            try {
                work();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "T2");

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

`tryLock()` does not magically make every concurrency design deadlock-proof, but bounded acquisition can prevent indefinite waiting in appropriate designs.

---

# 19. Deadlock Detection ⭐⭐⭐⭐⭐

When a Java application appears frozen, inspect thread dumps.

Useful tools include:

```text
jstack <pid>
```

or JVM monitoring/profiling tools that expose thread information.

A thread dump can show threads waiting to acquire locks and identify deadlock cycles.

---

# 20. Programmatic Detection with `ThreadMXBean` ⭐⭐⭐⭐⭐

The JVM management API can detect deadlocked threads.

```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadMXBean;

public class DeadlockDetectionDemo {

    public static void main(String[] args) {
        ThreadMXBean bean =
                ManagementFactory.getThreadMXBean();

        long[] ids = bean.findDeadlockedThreads();

        if (ids == null) {
            System.out.println("No deadlock detected");
        } else {
            System.out.println("Deadlock detected");
        }
    }
}
```

`findDeadlockedThreads()` can identify threads involved in monitor or ownable synchronizer deadlocks supported by the JVM management API.

---

# 21. Practice Code — Detect the Classic Deadlock ⭐⭐⭐⭐⭐

```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadMXBean;

public class DetectDeadlockPractice {

    private static final Object A = new Object();
    private static final Object B = new Object();

    public static void main(String[] args) throws Exception {

        Thread t1 = new Thread(() -> {
            synchronized (A) {
                sleep();
                synchronized (B) {
                    System.out.println("T1 acquired both");
                }
            }
        }, "T1");

        Thread t2 = new Thread(() -> {
            synchronized (B) {
                sleep();
                synchronized (A) {
                    System.out.println("T2 acquired both");
                }
            }
        }, "T2");

        t1.start();
        t2.start();

        Thread.sleep(500);

        ThreadMXBean bean =
                ManagementFactory.getThreadMXBean();

        long[] deadlocked = bean.findDeadlockedThreads();

        if (deadlocked != null) {
            System.out.println("Deadlock detected for thread IDs:");
            for (long id : deadlocked) {
                System.out.println(id);
            }
        } else {
            System.out.println("No deadlock detected");
        }
    }

    private static void sleep() {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

---

# 22. Why `synchronized` Can Deadlock

`synchronized` provides mutual exclusion, but it does not automatically impose a global ordering across multiple locks.

Therefore:

```java
synchronized (A) {
    synchronized (B) {
    }
}
```

and:

```java
synchronized (B) {
    synchronized (A) {
    }
}
```

can create a circular wait.

---

# 23. Immutable / Stateless Design Can Reduce Locking ⭐⭐⭐⭐

One way to reduce deadlock opportunities is to reduce shared mutable state.

Prefer, where practical:

```text
Immutable objects
        ↓
Less shared mutation
        ↓
Less synchronization
        ↓
Fewer lock dependencies
        ↓
Lower deadlock risk
```

This does not mean immutability is a universal deadlock solution, but it can simplify concurrent design.

---

# 24. Deadlock Prevention Checklist ⭐⭐⭐⭐⭐

Before acquiring multiple locks, ask:

```text
1. Do I really need both locks?
2. Can I avoid nested locking?
3. Is there a global lock order?
4. Can I reduce lock scope?
5. Can I use a higher-level concurrent abstraction?
6. Can I use timed acquisition where appropriate?
7. Is there a clear ownership rule?
```

---

# 25. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1

> Two threads using `synchronized` automatically cause deadlock.

❌ False.

Deadlock requires a problematic dependency cycle.

### Trap 2

> `sleep()` causes deadlock.

❌ False.

It can make a timing scenario easier to reproduce, but the lock dependency is the actual issue.

### Trap 3

> `wait()` and deadlock are the same.

❌ False.

`wait()` is a coordination mechanism; deadlock is a circular dependency/no-progress condition.

### Trap 4

> More synchronized code means deadlock.

❌ False.

Synchronization can be correct; inconsistent lock ordering is a common cause of deadlock.

### Trap 5

> `interrupt()` can always break a `synchronized` deadlock.

❌ False.

Intrinsic monitor acquisition through `synchronized` is not generally interruptible.

### Trap 6

> `notify()` can fix any deadlock.

❌ False.

Deadlock involving lock acquisition is not solved by arbitrary notification.

### Trap 7

> A thread dump is useless if the application is frozen.

❌ False.

Thread dumps are one of the most useful ways to diagnose blocked/deadlocked threads.

---

# 26. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is deadlock?

A condition where threads wait indefinitely because each is waiting for a resource/lock held by another thread in a dependency cycle.

### Q2. What are the four Coffman conditions?

Mutual exclusion, hold and wait, no preemption, and circular wait.

### Q3. How do you prevent deadlock?

The most practical strategy is consistent global lock ordering. Other strategies include reducing lock scope, avoiding nested locks, and using timed acquisition/higher-level concurrency utilities where appropriate.

### Q4. What is the most common Java deadlock example?

Two threads acquiring two locks in opposite order.

### Q5. How do you detect deadlock?

Thread dumps such as `jstack`, JVM monitoring tools, or programmatically with `ThreadMXBean`.

### Q6. Can `interrupt()` break a `synchronized` lock acquisition?

No. Intrinsic monitor acquisition is not generally interruptible.

### Q7. Does `sleep()` release a lock?

No. If a thread sleeps while holding an intrinsic monitor, it continues to hold that monitor during sleep.

### Q8. Difference between deadlock and starvation?

Deadlock involves a cycle of waiting; starvation means a thread repeatedly fails to get the resources/CPU access it needs.

### Q9. Difference between deadlock and livelock?

Deadlocked threads are blocked; livelocked threads remain active but make no useful progress.

### Q10. Can deadlock happen with one lock?

The classic lock-cycle deadlock requires multiple dependencies, but poor synchronization/coordination can create indefinite waiting scenarios even when only one monitor is involved. In interview discussions, the standard deadlock example uses at least two locks/resources.

---

# 27. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Deadlock is a situation where two or more threads are permanently blocked because each is waiting for a resource held by another thread. The classic Java example is Thread-1 acquiring Lock-A and waiting for Lock-B while Thread-2 acquires Lock-B and waits for Lock-A. Deadlock requires the four Coffman conditions: mutual exclusion, hold and wait, no preemption, and circular wait. The most practical prevention technique is consistent lock ordering, so all threads acquire multiple locks in the same order. We can also reduce lock scope, avoid unnecessary nested locking, and use timed lock acquisition such as `ReentrantLock.tryLock()` where appropriate. For diagnosis, thread dumps and `ThreadMXBean.findDeadlockedThreads()` can help identify the deadlocked threads. A key distinction is that `sleep()` does not release intrinsic locks, while `wait()` does release the waited-on monitor."**

---

# 28. Quick Revision ⭐⭐⭐⭐⭐

```text
DEADLOCK
   ↓
Threads wait forever
   ↓
Circular resource dependency

Classic:
T1: A → waits B
T2: B → waits A

4 Coffman Conditions:
1. Mutual Exclusion
2. Hold and Wait
3. No Preemption
4. Circular Wait

Prevention:
→ Global lock ordering
→ Reduce lock scope
→ Avoid nested locks
→ Timed tryLock()
→ Higher-level concurrency utilities

Detection:
→ Thread dump / jstack
→ ThreadMXBean

Remember:
sleep() → does NOT release lock
wait()  → releases waited-on monitor
interrupt() → does NOT generally break synchronized lock acquisition
```

### One-line memory trick

> **Deadlock = I hold what you need, you hold what I need.**

---

# 29. Practice Checklist

- [x] Deadlock definition
- [x] Classic two-lock example
- [x] Coffman conditions
- [x] Resource dependency cycle
- [x] Lock ordering
- [x] Nested lock problem
- [x] Deadlock prevention
- [x] `ReentrantLock.tryLock()`
- [x] Deadlock detection
- [x] `ThreadMXBean`
- [x] Thread dump concept
- [x] Deadlock vs race condition
- [x] Deadlock vs starvation
- [x] Deadlock vs livelock
- [x] `sleep()` relationship
- [x] `wait()` relationship
- [x] `interrupt()` limitation
- [x] Common traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.29 — `interrupt()`](../29-interrupt/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.31 — Starvation**