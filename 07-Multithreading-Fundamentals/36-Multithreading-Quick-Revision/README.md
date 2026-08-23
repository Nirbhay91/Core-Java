# 7.36 — Multithreading Quick Revision

> **Purpose:** Fast interview revision of Chapter 7 — Multithreading Fundamentals.
>
> Use this page for a **10–20 minute revision before an interview**. Each section contains the key idea, common trap, and a small practice snippet.

---

## 1. Process vs Thread

| Process | Thread |
|---|---|
| Independent execution environment | Execution unit inside a process |
| Own process resources/address space | Shares process memory/resources |
| Heavier | Lighter |
| Communication is comparatively expensive | Shared-memory communication is easier but creates synchronization concerns |

**Interview line:** A thread is a lightweight unit of execution within a process.

---

## 2. Creating a Thread

### `Thread` class

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running");
    }
}

public class ThreadCreationRevision {
    public static void main(String[] args) {
        new MyThread().start();
    }
}
```

### `Runnable`

```java
public class RunnableRevision {
    public static void main(String[] args) {
        Runnable task = () -> System.out.println("Running");
        new Thread(task).start();
    }
}
```

**Preferred basic design:** Separate the task (`Runnable`) from the thread that executes it.

---

## 3. `start()` vs `run()` ⭐⭐⭐⭐⭐

```java
Thread t = new Thread(() ->
        System.out.println(Thread.currentThread().getName()));

t.start(); // asks JVM to start a new thread
// t.run(); // ordinary method invocation; does not create a new thread
```

**Trap:** Calling `run()` directly does not start concurrent execution.

---

## 4. Thread States ⭐⭐⭐⭐⭐

Java exposes:

```text
NEW
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
TERMINATED
```

### Important
Java's `Thread.State` does **not** have separate `RUNNING` and `READY` states. Both are represented by `RUNNABLE` from the Java API perspective.

---

## 5. `sleep()` ⭐⭐⭐⭐⭐

```java
Thread.sleep(1000);
```

- Pauses the current thread for the requested duration.
- Does **not** release a monitor lock held by the thread.
- Throws `InterruptedException`.

```java
synchronized (lock) {
    Thread.sleep(1000); // lock remains held
}
```

---

## 6. `join()` ⭐⭐⭐⭐⭐

```java
Thread worker = new Thread(() -> doWork());
worker.start();
worker.join();
System.out.println("Worker finished");
```

`join()` makes the calling thread wait for another thread to terminate.

**Happens-before:** Actions in a thread happen-before another thread successfully returns from `join()` on that thread.

---

## 7. `yield()` ⭐⭐⭐⭐

```java
Thread.yield();
```

- Scheduler hint only.
- No guarantee another thread will run.
- Does not release a held monitor lock.
- Do not use it for correctness.

---

## 8. Race Condition ⭐⭐⭐⭐⭐

### Unsafe

```java
class Counter {
    private int count;

    void increment() {
        count++;
    }
}
```

`count++` is a read-modify-write sequence, not one atomic operation.

### Safe with synchronization

```java
class SafeCounter {
    private int count;

    synchronized void increment() {
        count++;
    }

    synchronized int get() {
        return count;
    }
}
```

---

## 9. Critical Section

A **critical section** is code that accesses shared state and must be protected so concurrent execution cannot violate the required invariant.

```java
synchronized (lock) {
    // critical section
}
```

**Rule:** Keep critical sections as small as practical, but never split an operation in a way that breaks the business invariant.

---

## 10. `synchronized` Method vs Block ⭐⭐⭐⭐⭐

### Method

```java
public synchronized void update() {
    // whole instance method is protected
}
```

### Block

```java
public void update() {
    // non-critical work
    synchronized (lock) {
        // only this part is protected
    }
}
```

A block gives finer control over the protected region and lock object.

---

## 11. Object Lock vs Class Lock ⭐⭐⭐⭐⭐

```java
class LockDemo {
    public synchronized void instanceMethod() {
        // this object's monitor
    }

    public static synchronized void staticMethod() {
        // LockDemo.class monitor
    }
}
```

### Quick memory trick

```text
instance synchronized → this
static synchronized   → Class object
```

They are different monitors.

---

## 12. Reentrancy ⭐⭐⭐⭐

```java
class ReentrantDemo {
    synchronized void outer() {
        inner();
    }

    synchronized void inner() {
        System.out.println("Same thread reacquires monitor");
    }
}
```

Java intrinsic monitors are **reentrant**: a thread that owns a monitor can acquire it again.

---

## 13. Thread Safety

A component is thread-safe when it maintains its correctness guarantees under the intended concurrent access.

Ask three questions:

```text
1. What state is shared?
2. What invariant must remain true?
3. Which mechanism protects that invariant?
```

---

## 14. Atomicity vs Visibility vs Ordering ⭐⭐⭐⭐⭐

| Concept | Meaning |
|---|---|
| Atomicity | Operation appears indivisible |
| Visibility | One thread can observe another thread's writes as guaranteed by the memory model |
| Ordering | Required ordering of actions is preserved by happens-before guarantees |

**Important:** Solving visibility does not automatically solve atomicity.

---

## 15. `volatile` ⭐⭐⭐⭐⭐

### Good use

```java
private volatile boolean running = true;
```

Useful for a simple shared state where visibility and the required ordering guarantee are enough.

### Trap

```java
private volatile int count;
count++; // NOT atomic
```

`volatile` does not turn compound read-modify-write operations into atomic operations.

---

## 16. `volatile` vs `synchronized`

| `volatile` | `synchronized` |
|---|---|
| Visibility + ordering guarantees for the variable | Mutual exclusion + visibility/ordering through monitor synchronization |
| No mutual exclusion | Mutual exclusion |
| Does not make `count++` atomic | Can protect compound operations |
| Good for simple state flags when appropriate | Good for critical sections/invariants |

---

## 17. Happens-Before ⭐⭐⭐⭐⭐

Happens-before is a Java Memory Model ordering relationship that gives visibility and ordering guarantees.

Important examples:

```text
unlock(monitor) → later lock(same monitor)
volatile write → later volatile read of same variable
Thread.start() → actions in started thread
thread actions → successful join() return
```

**Interview line:** Happens-before is about guaranteed visibility and ordering, not simply about wall-clock time.

---

## 18. `wait()` ⭐⭐⭐⭐⭐

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
}
```

Key points:

- Must wait on the monitor associated with the object.
- Calling thread must own that monitor.
- Releases that monitor while waiting.
- Reacquires the monitor before `wait()` returns.
- Always re-check the condition, normally using `while`.

---

## 19. `notify()` vs `notifyAll()` ⭐⭐⭐⭐⭐

```text
notify()    → wakes one waiting thread
notifyAll() → wakes all waiting threads
```

Neither operation immediately hands the lock to the awakened thread. The awakened thread must reacquire the monitor before proceeding.

**Common safe pattern:**

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
    // use condition
}
```

---

## 20. `wait()` vs `sleep()` ⭐⭐⭐⭐⭐

| `wait()` | `sleep()` |
|---|---|
| `Object` method | `Thread` method |
| Used for coordination | Used for timed pause |
| Releases the monitor it waits on | Does not release held monitors |
| Must own the relevant monitor | No monitor ownership requirement just to sleep |
| Usually paired with notification/condition checking | Usually paired with timing or delay logic |

---

## 21. Thread Interruption ⭐⭐⭐⭐⭐

```java
thread.interrupt();
```

Interruption is a **cooperative cancellation/request mechanism**. It does not forcibly kill a thread.

### Best practice

```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

Preserve the interrupt status when the current layer cannot fully handle the cancellation.

---

## 22. Deadlock ⭐⭐⭐⭐⭐

Four classic Coffman conditions:

```text
1. Mutual exclusion
2. Hold and wait
3. No preemption
4. Circular wait
```

### Typical pattern

```text
T1 holds A → waits for B
T2 holds B → waits for A
```

### Prevention

- Consistent lock ordering
- Smaller critical sections
- Avoid unnecessary nested locks
- Timed lock acquisition where appropriate

---

## 23. Starvation vs Livelock vs Deadlock ⭐⭐⭐⭐⭐

| Problem | Quick definition |
|---|---|
| Deadlock | Threads are stuck waiting in a circular dependency |
| Starvation | A thread repeatedly fails to get enough opportunity/resources to progress |
| Livelock | Threads remain active and react to each other but make no useful progress |

**Memory trick:**

```text
Deadlock  = blocked
Starvation = denied
Livelock  = busy but useless
```

---

## 24. Thread Communication Patterns

### Low-level

```text
wait / notify / notifyAll
```

### Higher-level

```text
BlockingQueue
CountDownLatch
Semaphore
ExecutorService
CompletableFuture
```

Use the highest-level abstraction that clearly fits the requirement.

---

## 25. Producer-Consumer ⭐⭐⭐⭐⭐

### Preferred basic solution

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ProducerConsumerRevision {
    public static void main(String[] args) throws Exception {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(2);

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
                    System.out.println(queue.take());
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

---

## 26. Atomic Classes ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicRevision {
    private final AtomicInteger counter = new AtomicInteger();

    void increment() {
        counter.incrementAndGet();
    }
}
```

Use atomic classes when the operation can be expressed as atomic updates to individual variables.

**Trap:** An atomic field does not automatically make a multi-field business transaction atomic.

---

## 27. `ConcurrentHashMap` ⭐⭐⭐⭐⭐

### Avoid

```java
if (!map.containsKey(key)) {
    map.put(key, value);
}
```

The pair is not one atomic operation.

### Prefer

```java
map.putIfAbsent(key, value);
```

Use the concurrent collection's atomic compound methods where they match the requirement.

---

## 28. `ThreadLocal` ⭐⭐⭐⭐

```java
private static final ThreadLocal<String> USER = new ThreadLocal<>();

try {
    USER.set("request-123");
    // work
} finally {
    USER.remove();
}
```

**Important with thread pools:** Always consider cleanup because the worker thread can be reused for another task.

---

## 29. Common Thread-Safety Strategies

```text
1. Immutability
2. Thread confinement
3. Synchronization
4. Atomic variables
5. Concurrent collections
6. Safe publication
7. Minimize shared mutable state
8. Reduce critical-section scope carefully
```

**Best strategy:** Avoid sharing mutable state when practical.

---

# 30. Practice: Thread-Safe Counter ⭐⭐⭐⭐⭐

Try this without looking at the answer first.

### Task
Two threads must increment a counter 10,000 times each. Make the result reliably `20,000`.

### Solution

```java
import java.util.concurrent.atomic.AtomicInteger;

public class CounterPractice {
    public static void main(String[] args) throws Exception {
        AtomicInteger counter = new AtomicInteger();

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

Expected result:

```text
20000
```

---

# 31. Practice: Graceful Worker Shutdown ⭐⭐⭐⭐⭐

```java
public class ShutdownPractice {
    private static volatile boolean running = true;

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            try {
                while (running) {
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            System.out.println("Worker stopped");
        });

        worker.start();
        Thread.sleep(500);

        running = false;
        worker.interrupt();
        worker.join();
    }
}
```

This combines visibility, interruption, and `join()`.

---

# 32. Practice: Deadlock Detection Thinking

Given:

```java
synchronized (A) {
    synchronized (B) {
        // work
    }
}
```

and another thread:

```java
synchronized (B) {
    synchronized (A) {
        // work
    }
}
```

### Answer
Potential deadlock due to inconsistent lock ordering.

### Fix
Both paths should acquire:

```text
A → B
```

rather than one path using `A → B` and the other `B → A`.

---

# 33. Interview One-Liners ⭐⭐⭐⭐⭐

| Question | Answer |
|---|---|
| Does `run()` create a thread? | No, direct invocation is a normal method call. |
| Does `sleep()` release a lock? | No. |
| Does `wait()` release its monitor? | Yes, while waiting. |
| Does `yield()` guarantee another thread runs? | No. |
| Is `count++` atomic? | No. |
| Does `volatile` make `count++` atomic? | No. |
| Is `synchronized` reentrant? | Yes. |
| What does instance synchronized use? | The instance monitor. |
| What does static synchronized use? | The `Class` object's monitor. |
| Does `interrupt()` kill a thread? | No. |
| Why use `while` with `wait()`? | Re-check the condition after waking. |
| Best basic producer-consumer abstraction? | Usually `BlockingQueue`. |
| Can `ConcurrentHashMap` solve every compound operation? | No. |
| What is deadlock? | Circular waiting/blocking among threads/resources. |
| What is starvation? | A thread is repeatedly denied progress. |
| What is livelock? | Threads are active but make no useful progress. |
| Does Java have RUNNING and READY states separately? | No; both are represented by `RUNNABLE`. |

---

# 34. 30-Second Mental Model

```text
THREAD
  ↓
Shared State?
  ↓
Race / Invariant?
  ↓
What guarantee is required?
  ├── Visibility → volatile / happens-before mechanism
  ├── Single atomic update → Atomic* / suitable concurrent primitive
  ├── Compound invariant → synchronized / Lock
  └── Coordination → BlockingQueue / latch / condition / higher-level API
  ↓
Check interruption + shutdown
  ↓
Check deadlock / starvation / livelock
```

---

# 35. Final Quick Revision Checklist

- [x] Process vs Thread
- [x] `Thread` vs `Runnable`
- [x] `start()` vs `run()`
- [x] Thread states
- [x] `sleep()`
- [x] `join()`
- [x] `yield()`
- [x] Race condition
- [x] Critical section
- [x] `synchronized` method/block
- [x] Object lock / class lock
- [x] Reentrancy
- [x] Thread safety
- [x] Atomicity / visibility / ordering
- [x] Happens-before
- [x] `volatile`
- [x] `wait()` / `notify()` / `notifyAll()`
- [x] `wait()` vs `sleep()`
- [x] Interruption
- [x] Deadlock
- [x] Starvation
- [x] Livelock
- [x] Thread communication
- [x] Atomic classes
- [x] `ConcurrentHashMap`
- [x] `ThreadLocal`
- [x] Common thread-safety strategies
- [x] Interview one-liners
- [x] Practice code

---

# 🎯 2-Minute Interview Summary

> **"In Java multithreading, I first identify shared mutable state and the invariant that must remain correct. Then I determine whether the problem needs visibility, atomicity, ordering, mutual exclusion, or thread coordination. I use `volatile` for appropriate visibility/state-flag scenarios, atomic classes for suitable single-variable atomic updates, and `synchronized` or `Lock` when multiple operations must be protected as one critical section. For producer-consumer I prefer `BlockingQueue` when applicable. I use `interrupt()` for cooperative cancellation and `join()` when one thread must wait for another to finish. I also check for deadlock, starvation, livelock, race conditions, unsafe publication, and lock-ordering problems. The key is choosing the smallest mechanism that provides the required correctness guarantee."**

---

## Navigation

[← 7.35 — Multithreading Interview Scenarios](../35-Multithreading-Interview-Scenarios/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.37 — Multithreading Final Assessment**