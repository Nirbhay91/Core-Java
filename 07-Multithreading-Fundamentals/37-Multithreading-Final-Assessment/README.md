# 7.37 — Multithreading Final Assessment

> **Chapter 7 Final Assessment — Multithreading Fundamentals**
>
> This assessment is designed for **5-year Java interview level**. Try the questions first without opening the answers.

---

## 🎯 Assessment Rules

- **Total:** 40 questions
- **Conceptual:** 20
- **Scenario-based:** 10
- **Coding / debugging:** 10
- **Target:** 80%+
- **Interview target:** Explain the important answers in under 2 minutes.

---

# Part A — Core Concepts

### Q1. What is the difference between a process and a thread?

**Expected answer:** A process has its own execution environment/resources, while threads are execution units within a process and share process resources.

### Q2. `start()` vs `run()`?

**Answer:** `start()` initiates a new thread of execution; direct `run()` invocation is just a normal method call on the current thread.

### Q3. List Java thread states.

**Answer:** `NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, `TERMINATED`.

### Q4. Does Java have separate `RUNNING` and `READY` states?

**Answer:** No. Java's `Thread.State` exposes both from the Java API perspective as `RUNNABLE`.

### Q5. Does `sleep()` release a monitor lock?

**Answer:** No.

### Q6. Does `wait()` release the monitor?

**Answer:** Yes. The thread releases the monitor it is waiting on and later reacquires it before `wait()` returns.

### Q7. What does `join()` do?

**Answer:** It makes the calling thread wait until the target thread terminates.

### Q8. What does `yield()` guarantee?

**Answer:** Nothing. It is only a scheduler hint and does not guarantee another thread will run.

### Q9. What is a race condition?

**Answer:** Program correctness depends on the timing/interleaving of concurrent operations on shared state.

### Q10. What is a critical section?

**Answer:** A section of code that accesses shared state and must be protected to preserve the required invariant.

---

# Part B — Synchronization & Locks

### Q11. What lock does an instance `synchronized` method use?

**Answer:** The monitor of the current object (`this`).

### Q12. What lock does a static `synchronized` method use?

**Answer:** The monitor of the corresponding `Class` object.

### Q13. Is Java `synchronized` reentrant?

**Answer:** Yes. A thread that already owns a monitor can acquire the same monitor again.

### Q14. Object lock vs class lock?

**Answer:** They are different monitors. Object lock protects synchronization associated with one instance; class lock protects synchronization associated with the class object.

### Q15. Why prefer a small synchronized block sometimes?

**Answer:** It can reduce the protected region and lock contention while keeping only the required critical section synchronized.

### Q16. Can `synchronized` solve every concurrency problem?

**Answer:** No. It provides mutual exclusion and monitor-based memory visibility/ordering guarantees, but design issues such as deadlock, starvation, poor lock granularity, and incorrect coordination can still exist.

---

# Part C — Java Memory Model

### Q17. Explain atomicity, visibility and ordering.

**Answer:**

```text
Atomicity  → operation is indivisible
Visibility → writes become observable under a defined memory-model guarantee
Ordering   → actions have a defined happens-before relationship
```

### Q18. Is `count++` atomic?

**Answer:** No. It is a read-modify-write operation.

### Q19. Does `volatile` make `count++` atomic?

**Answer:** No. `volatile` provides visibility and relevant ordering guarantees, not compound-operation atomicity.

### Q20. Give examples of happens-before relationships.

**Answer:**

```text
unlock → subsequent lock on same monitor
volatile write → subsequent read of same volatile variable
start() → actions in started thread
thread actions → successful join() return
```

---

# Part D — `wait()` / `notify()` / Interruption

### Q21. Why must `wait()` normally be called inside a `while` loop?

**Answer:** Because after waking, the condition must be re-checked before proceeding; a wake-up does not itself prove the condition is true.

### Q22. What happens if `wait()` is called without owning the monitor?

**Answer:** `IllegalMonitorStateException` is thrown.

### Q23. What is the difference between `notify()` and `notifyAll()`?

**Answer:** `notify()` wakes one waiting thread; `notifyAll()` wakes all waiting threads. Awakened threads still need to reacquire the monitor.

### Q24. Does `interrupt()` forcibly terminate a thread?

**Answer:** No. It is a cooperative interruption request.

### Q25. What should you usually do after catching `InterruptedException` if the current method cannot fully handle cancellation?

**Answer:** Restore the interrupt status:

```java
Thread.currentThread().interrupt();
```

---

# Part E — Concurrency Problems

### Q26. What is deadlock?

**Answer:** Threads become unable to progress because each waits for a resource/lock held by another, forming a cycle.

### Q27. Name the four Coffman conditions.

```text
1. Mutual exclusion
2. Hold and wait
3. No preemption
4. Circular wait
```

### Q28. Starvation vs deadlock?

**Answer:** In starvation, a thread repeatedly fails to get enough opportunity/resources to progress. In deadlock, a cycle of dependencies prevents progress.

### Q29. What is livelock?

**Answer:** Threads remain active and keep reacting to each other but make no useful progress.

### Q30. How can deadlock be prevented?

**Answer:** Consistent lock ordering, avoiding unnecessary nested locks, reducing lock scope, and using timed/non-blocking acquisition where appropriate.

---

# Part F — Scenario-Based Questions

### Q31. A shared counter gives different results every run. What do you check first?

**Answer:** Identify shared mutable state and the read-modify-write operation. Then protect the invariant using `synchronized`, an atomic class, or another suitable concurrency mechanism.

### Q32. A worker thread loops on a boolean flag but does not stop reliably. What may be wrong?

**Answer:** The flag may not have an appropriate visibility guarantee. A suitable `volatile` flag or another happens-before mechanism may be required.

### Q33. A thread sleeps while holding a lock and other threads stop progressing. Why?

**Answer:** `sleep()` does not release the monitor, so other threads requiring that lock remain blocked.

### Q34. Two synchronized methods appear to run simultaneously. Is that always a bug?

**Answer:** Not necessarily. They may be synchronized on different objects/monitors, or one may be instance synchronized and the other static synchronized.

### Q35. Producer-consumer code uses manually managed `wait()`/`notify()`. What higher-level API could simplify it?

**Answer:** `BlockingQueue` is usually a strong choice for a standard producer-consumer design.

### Q36. `ConcurrentHashMap` code uses `containsKey()` followed by `put()`. What is the issue?

**Answer:** The check and update are separate operations and can race. Prefer an atomic method such as `putIfAbsent()` when appropriate.

### Q37. A thread pool task stores request data in `ThreadLocal` and later another request sees it. What is the likely issue?

**Answer:** The worker thread is reused. The `ThreadLocal` value was not cleaned up. Use `remove()` in a `finally` block where appropriate.

### Q38. A system has two locks and sometimes freezes. What should you inspect?

**Answer:** Lock acquisition order and thread dump evidence. Look for a circular wait such as `A → B` in one thread and `B → A` in another.

### Q39. A developer changed a field to `volatile`, but a multi-field transfer is still inconsistent. Why?

**Answer:** `volatile` on individual fields does not make a multi-step business invariant atomic. The complete invariant may need synchronization, locking, immutable state replacement, or another atomic design.

### Q40. How would you choose between `volatile`, `AtomicInteger`, `synchronized`, and `BlockingQueue`?

**Answer:**

```text
Simple visible state flag        → volatile (when sufficient)
Atomic single-variable update   → AtomicInteger / AtomicLong etc.
Compound critical-section rule  → synchronized / Lock
Producer-consumer coordination  → BlockingQueue
```

Always choose based on the required correctness guarantee, not just performance assumptions.

---

# Part G — Coding Assessment

## Coding 1 — Thread-Safe Counter ⭐⭐⭐⭐⭐

### Problem
Two threads increment the same counter 10,000 times. The final result must reliably be `20,000`.

### Solution

```java
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadSafeCounter {
    public static void main(String[] args) throws InterruptedException {
        AtomicInteger counter = new AtomicInteger(0);

        Runnable task = () -> {
            for (int i = 0; i < 10_000; i++) {
                counter.incrementAndGet();
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

**Expected:** `20000`

---

## Coding 2 — Graceful Shutdown ⭐⭐⭐⭐⭐

```java
public class GracefulShutdown {
    private static volatile boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            try {
                while (running) {
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        worker.start();
        Thread.sleep(500);

        running = false;
        worker.interrupt();
        worker.join();

        System.out.println("Stopped safely");
    }
}
```

**Concepts tested:** `volatile`, interruption, `join()`, cooperative cancellation.

---

## Coding 3 — Producer Consumer ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ProducerConsumerAssessment {
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(3);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    queue.put(i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    System.out.println("Consumed: " + queue.take());
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();

        producer.join();
        consumer.join();
    }
}
```

**Concepts tested:** blocking coordination, interruption, producer-consumer pattern.

---

# Part H — Debug This Code

## Debug 1 — Wrong Thread Creation

```java
Thread t = new Thread(() ->
        System.out.println(Thread.currentThread().getName()));

t.run();
```

### Question
Why is this not concurrent execution?

### Answer
`run()` is called directly. Use `t.start()` to request execution on a new thread.

---

## Debug 2 — Volatile Counter

```java
private volatile int count;

void increment() {
    count++;
}
```

### Question
Is this thread-safe?

### Answer
No. `volatile` does not make `count++` atomic.

---

## Debug 3 — Wait Without Monitor

```java
Object lock = new Object();
lock.wait();
```

### Question
What is wrong?

### Answer
The current thread does not own `lock`'s monitor. It should be inside:

```java
synchronized (lock) {
    lock.wait();
}
```

---

## Debug 4 — Deadlock

```java
// Thread 1
synchronized (A) {
    synchronized (B) {
        // work
    }
}

// Thread 2
synchronized (B) {
    synchronized (A) {
        // work
    }
}
```

### Answer
Potential circular wait. Establish one global lock order, for example `A → B`.

---

# Part I — Scoring Rubric

| Score | Level | Meaning |
|---:|---|---|
| 36–40 | 🟢 Excellent | Interview-ready for fundamentals |
| 32–35 | 🟢 Strong | Minor revision needed |
| 28–31 | 🟡 Good | Revise weak areas |
| 24–27 | 🟠 Needs Revision | Revisit concepts + code |
| <24 | 🔴 Not Ready | Repeat Chapter 7 revision |

### Coding requirement

Even with a high theoretical score, you should be able to write the three coding exercises without copying.

---

# 🎯 Final Interview Checklist

Before marking Chapter 7 complete, you should be able to explain without notes:

- [ ] Process vs Thread
- [ ] `Thread` vs `Runnable`
- [ ] `start()` vs `run()`
- [ ] All Java thread states
- [ ] Why `RUNNABLE` includes execution-ready/running states from Java API perspective
- [ ] `sleep()` vs `wait()`
- [ ] `join()`
- [ ] `yield()` limitations
- [ ] Race condition
- [ ] Critical section
- [ ] Object/class locks
- [ ] Reentrancy
- [ ] Thread safety
- [ ] Atomicity / visibility / ordering
- [ ] Happens-before
- [ ] `volatile`
- [ ] `wait()` / `notify()` / `notifyAll()`
- [ ] Interruption and graceful shutdown
- [ ] Deadlock / starvation / livelock
- [ ] Producer-consumer
- [ ] Atomic classes
- [ ] Concurrent collections
- [ ] `ThreadLocal` cleanup
- [ ] Correct synchronization strategy selection

---

# 🏆 2-Minute Final Interview Answer

> **"For Java multithreading, I start by identifying shared mutable state and the invariant that must remain correct. Then I determine whether I need visibility, atomicity, ordering, mutual exclusion, or coordination. `volatile` is appropriate for certain simple shared-state visibility cases, atomic classes for suitable single-variable atomic updates, and `synchronized` or `Lock` when a compound invariant must be protected. For producer-consumer I generally prefer `BlockingQueue` instead of implementing the protocol manually. I use `interrupt()` for cooperative cancellation and `join()` when a thread must wait for another to finish. I also look for race conditions, unsafe publication, deadlocks caused by inconsistent lock ordering, starvation, and livelock. The important part is choosing the simplest mechanism that provides the exact correctness guarantees required by the design."**

---

# ✅ Chapter 7 Completion Criteria

Chapter 7 can be considered complete when:

1. All 7.1–7.36 topics are understood.
2. This final assessment is attempted without notes.
3. Score is **80% or higher**.
4. The three coding exercises can be written independently.
5. Common concurrency bugs can be explained and debugged.
6. You can give the 2-minute interview summary confidently.

---

## Navigation

[← 7.36 — Multithreading Quick Revision](../36-Multithreading-Quick-Revision/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Chapter 7 — Multithreading Fundamentals → 🎯 Final Assessment**