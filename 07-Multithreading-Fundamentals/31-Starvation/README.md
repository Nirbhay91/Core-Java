# 7.31 — Starvation

## 🎯 Objective

Understand **thread starvation** in Java: what it is, why it happens, how unfair scheduling/lock contention can cause it, how it differs from deadlock and livelock, and how to reduce its risk.

> **Golden rule:** Starvation occurs when a thread is repeatedly denied the CPU, a lock, or another required resource and therefore cannot make sufficient progress.

---

## 1. What Is Starvation? ⭐⭐⭐⭐⭐

**Starvation** means a thread is ready or waiting to do useful work but repeatedly fails to get the resource or execution opportunity it needs.

Example:

```text
T1 needs LOCK
T2 repeatedly gets LOCK
T3 repeatedly gets LOCK
T1 keeps waiting
        ↓
   STARVATION
```

Unlike deadlock, the entire system does not necessarily stop. Other threads may continue making progress while one thread is continually denied access.

---

# 2. Real-World Analogy

Imagine three people waiting for one service counter:

```text
T1 → waiting
T2 → repeatedly served
T3 → repeatedly served
```

If T1 keeps getting skipped, T1 is starved even though the counter is active.

---

# 3. Starvation with `synchronized` ⭐⭐⭐⭐⭐

A synchronized monitor does not provide a general fairness guarantee about which waiting thread acquires the monitor next.

This means that under contention, an application should not assume strict FIFO lock acquisition merely because it uses `synchronized`.

```java
synchronized (LOCK) {
    // critical section
}
```

The JVM/thread scheduler decides which eligible thread gets an opportunity, subject to the platform and synchronization semantics.

---

# 4. Practice Code — Lock Contention ⭐⭐⭐⭐⭐

The following program demonstrates a **starvation-prone design**, not a guaranteed starvation outcome. Scheduling is nondeterministic.

```java
public class StarvationDemo {

    private static final Object LOCK = new Object();

    public static void main(String[] args) {

        Thread fastWorker = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                synchronized (LOCK) {
                    System.out.println("Fast worker acquired lock");
                }
            }
        }, "Fast-Worker");

        Thread waitingWorker = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                synchronized (LOCK) {
                    System.out.println("Waiting worker acquired lock");
                }
            }
        }, "Waiting-Worker");

        fastWorker.start();
        waitingWorker.start();
    }
}
```

### Important

Do **not** say that this code guarantees starvation.

Correct interview wording:

> "This creates heavy lock contention and demonstrates a starvation-prone situation. Actual starvation depends on scheduling and timing."

---

# 5. Common Causes of Starvation ⭐⭐⭐⭐⭐

### 1. Unfair Lock Acquisition

A thread may repeatedly lose the race to acquire a lock.

### 2. Long Critical Sections

One thread holds a lock for too long.

### 3. High-Priority Threads

Excessive priority differences can contribute to a lower-priority thread receiving fewer execution opportunities on some platforms/schedulers.

### 4. Thread Pools with Poor Task Design

A worker can remain unable to execute because available workers are occupied by long-running tasks.

### 5. Resource Contention

A shared resource may be continuously consumed by other threads.

### 6. Unfair Scheduling / Synchronization Policy

The implementation does not necessarily guarantee fair access.

---

# 6. Starvation vs Deadlock ⭐⭐⭐⭐⭐

| Starvation | Deadlock |
|---|---|
| One or more threads may be repeatedly denied a resource | Threads are stuck in a dependency cycle |
| Other threads may continue | Deadlocked threads make no progress |
| No circular wait is required | Circular wait is a classic cause |
| Usually unfairness/resource competition | Usually lock/resource dependency |
| System may continue operating | Part of the system can become permanently blocked |

### Memory trick

```text
Starvation → "I am continuously skipped."
Deadlock   → "We are waiting for each other."
```

---

# 7. Starvation vs Livelock ⭐⭐⭐⭐⭐

### Starvation

A thread cannot obtain sufficient access to the resource/execution opportunity it needs.

### Livelock

Threads remain active and repeatedly react to one another, but useful progress is not made.

```text
Starvation → denied progress
Livelock   → active but ineffective
```

---

# 8. Starvation vs Race Condition ⭐⭐⭐⭐

### Race Condition

The result depends on the timing/interleaving of concurrent operations on shared state.

### Starvation

A thread repeatedly fails to get sufficient access to a required resource or execution opportunity.

They are different problems, although a poorly designed concurrent system can exhibit multiple problems at once.

---

# 9. Lock Fairness ⭐⭐⭐⭐⭐

`ReentrantLock` can optionally provide a fairness policy.

```java
ReentrantLock fairLock = new ReentrantLock(true);
```

The `true` argument requests a **fair** lock policy.

A fair lock generally favors the longest-waiting eligible thread rather than allowing the same style of barging that an unfair policy may permit.

### Important

Fairness can reduce starvation risk, but it does not mean:

```text
100% guaranteed no starvation
```

Nor does it mean:

```text
best performance
```

Fairness can introduce additional scheduling/queueing overhead and may reduce throughput in some workloads.

---

# 10. Practice Code — Fair vs Unfair `ReentrantLock` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class FairLockDemo {

    private static final ReentrantLock LOCK =
            new ReentrantLock(true);

    static void work() {
        LOCK.lock();
        try {
            System.out.println(Thread.currentThread().getName()
                    + " acquired lock");
        } finally {
            LOCK.unlock();
        }
    }

    public static void main(String[] args) throws Exception {

        Thread t1 = new Thread(FairLockDemo::work, "T1");
        Thread t2 = new Thread(FairLockDemo::work, "T2");
        Thread t3 = new Thread(FairLockDemo::work, "T3");

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();
    }
}
```

Compare:

```java
new ReentrantLock(false); // unfair/default policy
new ReentrantLock(true);  // fair policy requested
```

---

# 11. Why Long Critical Sections Increase Starvation Risk ⭐⭐⭐⭐⭐

Bad design:

```java
synchronized (LOCK) {
    performVeryLongOperation();
    callRemoteService();
    processLargeData();
}
```

The lock is held while unrelated/slow work executes.

Better:

```java
Data data;

synchronized (LOCK) {
    data = readSharedData();
}

processOutsideLock(data);
```

The exact design depends on consistency requirements, but the principle is:

> Keep the critical section as small as correctness allows.

---

# 12. Practice Code — Reduce Critical Section ⭐⭐⭐⭐⭐

```java
public class ShortCriticalSectionDemo {

    private static final Object LOCK = new Object();
    private static int counter;

    static void increment() {
        synchronized (LOCK) {
            counter++;
        }

        // Expensive work should not unnecessarily hold LOCK.
        performExpensiveWork();
    }

    static void performExpensiveWork() {
        // Simulate non-critical work.
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(ShortCriticalSectionDemo::increment);
        Thread t2 = new Thread(ShortCriticalSectionDemo::increment);

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(counter);
    }
}
```

---

# 13. Thread Priority and Starvation ⭐⭐⭐⭐⭐

Java provides thread priorities:

```java
Thread.MIN_PRIORITY
Thread.NORM_PRIORITY
Thread.MAX_PRIORITY
```

However, do not design correctness around thread priority.

Thread priority is a **scheduler hint**, and actual scheduling behavior is platform/JVM dependent.

### Interview answer

> "Thread priority can influence scheduling, but it is not a reliable synchronization or fairness mechanism and should not be used to guarantee that one thread will run before another."

---

# 14. Practice Code — Priority Is Not a Guarantee ⭐⭐⭐⭐

```java
public class ThreadPriorityDemo {

    public static void main(String[] args) throws Exception {

        Thread low = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Low priority");
            }
        });

        Thread high = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("High priority");
            }
        });

        low.setPriority(Thread.MIN_PRIORITY);
        high.setPriority(Thread.MAX_PRIORITY);

        low.start();
        high.start();

        low.join();
        high.join();
    }
}
```

The output order should **not** be treated as a portable guarantee.

---

# 15. Thread Pool Starvation ⭐⭐⭐⭐⭐

Starvation can also occur at the executor/task level.

For example, if all worker threads are occupied by tasks that wait for work that requires the same executor, the required work may never get a chance to execute.

Conceptually:

```text
Fixed pool size = 2

Worker-1 → waits for nested task
Worker-2 → waits for nested task

Nested tasks → need a free worker

No free worker
      ↓
Task starvation / thread-pool starvation
```

This is often called **thread-pool starvation deadlock** when the design can produce a deadlock-like permanent wait.

---

# 16. Practice Code — Executor Starvation Pattern ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class ExecutorStarvationDemo {

    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        Future<?> task1 = executor.submit(() -> {
            try {
                Future<Integer> nested = executor.submit(() -> 100);
                System.out.println("Task 1 result = " + nested.get());
            } catch (Exception e) {
                System.out.println("Task 1 could not complete: " + e);
            }
        });

        Future<?> task2 = executor.submit(() -> {
            try {
                Future<Integer> nested = executor.submit(() -> 200);
                System.out.println("Task 2 result = " + nested.get());
            } catch (Exception e) {
                System.out.println("Task 2 could not complete: " + e);
            }
        });

        task1.get();
        task2.get();

        executor.shutdown();
    }
}
```

### Why is this dangerous?

Both workers can become blocked on `get()` while the nested tasks remain queued because no worker is available to execute them.

Avoid this design by separating blocking and dependent tasks across appropriate executors or, better, by composing asynchronous work without blocking worker threads unnecessarily.

---

# 17. How to Reduce Starvation ⭐⭐⭐⭐⭐

### 1. Use Fair Locks Where Appropriate

```java
new ReentrantLock(true);
```

### 2. Keep Critical Sections Short

Do not hold locks during unnecessary I/O or expensive work.

### 3. Avoid Unnecessary Lock Contention

Use more granular state management where appropriate.

### 4. Avoid Priority-Based Correctness

Do not rely on thread priority for business correctness.

### 5. Design Executor Tasks Carefully

Avoid blocking worker threads waiting for tasks submitted to the same saturated executor.

### 6. Use Appropriate Concurrency Utilities

Choose abstractions that match the coordination problem instead of building complex lock chains.

### 7. Measure and Diagnose

Use thread dumps, profiling, metrics and lock contention information in production troubleshooting.

---

# 18. Is Fairness Always Better? ⭐⭐⭐⭐

No.

Fairness can reduce the chance of one thread being repeatedly bypassed, but it can also reduce throughput in some workloads.

```text
Unfair lock
→ potentially better throughput
→ possible unfair access

Fair lock
→ more predictable waiting order
→ potentially more overhead
```

Choose based on application requirements rather than assuming fair is always faster or always necessary.

---

# 19. Diagnosing Starvation in Production ⭐⭐⭐⭐⭐

Look for:

```text
1. A thread waiting for a long time
2. Heavy lock contention
3. Very long critical sections
4. Repeated acquisition by other threads
5. Executor queues growing
6. Saturated worker pools
7. Priority/scheduling anomalies
```

Useful diagnostics include thread dumps and JVM monitoring/profiling tools.

For lock-based problems, inspect:

```text
WAITING / BLOCKED threads
        ↓
Which lock/resource?
        ↓
Who owns/acquires it?
        ↓
How long is it held?
        ↓
Is access unfair or overly contended?
```

---

# 20. Starvation Does Not Mean Every Thread Is Blocked ⭐⭐⭐⭐⭐

This is an important interview distinction.

```text
Deadlock:
Some threads → waiting for each other

Starvation:
One/some threads → repeatedly denied
Other threads → may continue normally
```

Therefore an application can remain responsive while one operation/thread suffers starvation.

---

# 21. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1

> Starvation and deadlock are the same.

❌ False.

Deadlock involves circular waiting; starvation is repeated denial of resources/execution opportunities.

### Trap 2

> `synchronized` guarantees fair lock acquisition.

❌ False.

Do not assume FIFO/fairness from intrinsic synchronization.

### Trap 3

> `ReentrantLock(true)` guarantees that starvation can never happen.

❌ False.

It requests fair acquisition behavior, but overall application progress can still be blocked by poor design or other resource constraints.

### Trap 4

> Higher thread priority guarantees execution first.

❌ False.

Priority is not a portable ordering guarantee.

### Trap 5

> Starvation means the whole application is frozen.

❌ False.

Other threads may continue progressing.

### Trap 6

> Shorter critical sections automatically make every program thread-safe.

❌ False.

They reduce contention, but correctness still depends on proper synchronization/visibility/atomicity design.

---

# 22. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is starvation?

Starvation occurs when a thread repeatedly fails to obtain the resource, lock, CPU opportunity, or execution capacity it needs to make progress.

### Q2. Difference between starvation and deadlock?

Deadlock is a circular dependency where affected threads wait for one another. Starvation is repeated unfair denial of a resource or execution opportunity, while other work may continue.

### Q3. Can `synchronized` cause starvation?

It can contribute to starvation-prone situations under heavy contention because intrinsic monitors do not provide a general fairness guarantee.

### Q4. How can `ReentrantLock` help?

`ReentrantLock` supports an optional fairness policy via `new ReentrantLock(true)`, which can provide more predictable lock acquisition behavior.

### Q5. Does a fair lock guarantee better performance?

No. Fairness can add overhead and sometimes reduce throughput.

### Q6. Can thread priority cause starvation?

Priority differences can contribute to scheduling imbalance on some platforms, but priority is not a reliable mechanism for correctness or guaranteed scheduling order.

### Q7. How can you reduce starvation?

Use appropriate fairness, keep critical sections short, reduce contention, avoid unnecessary nested locking, design executors carefully, and avoid relying on thread priority.

### Q8. What is thread-pool starvation?

It occurs when all executor workers are occupied, often waiting for work that itself requires a worker from the same saturated executor.

### Q9. Is starvation always deterministic?

No. Scheduling and timing are nondeterministic, so a starvation-prone design does not necessarily reproduce starvation on every run.

### Q10. How do you diagnose starvation?

Use thread dumps, profiling, lock contention metrics, executor queue metrics, and inspect long-held locks or saturated worker pools.

---

# 23. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Starvation is a concurrency problem where a thread repeatedly fails to get the resource, lock, CPU opportunity, or execution capacity it needs, while other threads continue making progress. It differs from deadlock because deadlock usually involves a circular dependency where threads wait for each other, whereas starvation is primarily about repeated denial or unfair resource access. In Java, intrinsic `synchronized` monitors do not provide a general fairness guarantee, so heavy contention can create starvation-prone situations. When fairness is important, `ReentrantLock` can be created with a fair policy using `new ReentrantLock(true)`. We should also keep critical sections short, avoid unnecessary lock contention, not rely on thread priority for correctness, and design executor tasks so worker threads do not block waiting for tasks queued behind them in the same saturated pool. In production, thread dumps, lock contention metrics and executor metrics help diagnose starvation."**

---

# 24. Quick Revision ⭐⭐⭐⭐⭐

```text
STARVATION
   ↓
Thread repeatedly denied progress
   ↓
Other threads may continue

Common causes:
→ Lock contention
→ Unfair access
→ Long critical sections
→ Priority/scheduling imbalance
→ Saturated executor

Prevention:
→ Fair lock where appropriate
→ Short critical sections
→ Reduce contention
→ Avoid priority-based correctness
→ Design executor dependencies carefully

Fair lock:
new ReentrantLock(true)

Remember:
Starvation → "I keep getting skipped."
Deadlock   → "We keep waiting for each other."
Livelock   → "We keep reacting but don't progress."
```

---

# 25. Practice Checklist

- [x] Starvation definition
- [x] Real-world analogy
- [x] Lock contention
- [x] `synchronized` fairness limitation
- [x] Common causes
- [x] Starvation vs deadlock
- [x] Starvation vs livelock
- [x] Starvation vs race condition
- [x] Fair `ReentrantLock`
- [x] Critical-section design
- [x] Thread priority caveats
- [x] Thread-pool starvation
- [x] Prevention strategies
- [x] Production diagnosis
- [x] Common interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.30 — Deadlock](../30-Deadlock/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.32 — Livelock**