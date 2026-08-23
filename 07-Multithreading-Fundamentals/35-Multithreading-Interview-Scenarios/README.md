# 7.35 — Multithreading Interview Scenarios

## 🎯 Objective

Apply Chapter 7 concepts to realistic interview problems. The focus is not memorizing APIs; it is identifying the concurrency problem, choosing the right guarantee, explaining the Java Memory Model, and writing correct code.

---

# 1. Scenario: Two Threads Increment a Counter

### Problem
Two threads each increment a shared counter 10,000 times. The expected answer is 20,000, but the result may be smaller.

### Why?
`count++` is a read-modify-write operation and is not atomic.

### Practice Code — Unsafe

```java
public class UnsafeCounter {
    private int count;

    void increment() {
        count++;
    }

    int get() {
        return count;
    }

    public static void main(String[] args) throws Exception {
        UnsafeCounter counter = new UnsafeCounter();

        Runnable task = () -> {
            for (int i = 0; i < 10_000; i++) {
                counter.increment();
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

### Fix
Use `synchronized`, `AtomicInteger`, or another strategy appropriate to the required invariant.

---

# 2. Scenario: Counter with `AtomicInteger` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounterScenario {
    private final AtomicInteger count = new AtomicInteger();

    void increment() {
        count.incrementAndGet();
    }

    int get() {
        return count.get();
    }

    public static void main(String[] args) throws Exception {
        AtomicCounterScenario counter = new AtomicCounterScenario();

        Runnable task = () -> {
            for (int i = 0; i < 10_000; i++) {
                counter.increment();
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(counter.get()); // 20000
    }
}
```

### Interview Point
`AtomicInteger` is suitable for this single-variable atomic update. It does not automatically make arbitrary multi-step business logic thread-safe.

---

# 3. Scenario: `volatile` Stop Flag ⭐⭐⭐⭐⭐

### Problem
A worker thread loops until another thread asks it to stop.

### Practice Code

```java
public class StopFlagScenario {
    private static volatile boolean running = true;

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            while (running) {
                // simulate work
            }
            System.out.println("Worker stopped");
        });

        worker.start();
        Thread.sleep(500);

        running = false;
        worker.join();
    }
}
```

### Interview Answer
`volatile` is appropriate because the requirement is visibility of the latest state. It does not make compound operations atomic.

---

# 4. Scenario: `volatile` Counter Trap ⭐⭐⭐⭐⭐

This is still unsafe:

```java
private volatile int count;

void increment() {
    count++;
}
```

Why?

```text
read → add 1 → write
```

The entire sequence is not atomic.

### Correct alternatives

```java
AtomicInteger
```

or:

```java
synchronized
```

or an appropriate `Lock`.

---

# 5. Scenario: Thread-Safe Bank Account ⭐⭐⭐⭐⭐

### Requirement
Multiple threads can deposit and withdraw, but the balance must never become negative.

### Practice Code

```java
public class BankAccountScenario {
    private int balance;

    public BankAccountScenario(int initialBalance) {
        this.balance = initialBalance;
    }

    public synchronized void deposit(int amount) {
        if (amount > 0) {
            balance += amount;
        }
    }

    public synchronized boolean withdraw(int amount) {
        if (amount <= 0 || balance < amount) {
            return false;
        }

        balance -= amount;
        return true;
    }

    public synchronized int getBalance() {
        return balance;
    }
}
```

### Why synchronize the whole withdrawal?
The check and update form one invariant. Protecting only the write would still allow a race between the check and the update.

---

# 6. Scenario: Two Locks → Deadlock ⭐⭐⭐⭐⭐

### Practice Code

```java
public class DeadlockScenario {
    private final Object lockA = new Object();
    private final Object lockB = new Object();

    void task1() {
        synchronized (lockA) {
            System.out.println("T1 acquired A");

            synchronized (lockB) {
                System.out.println("T1 acquired B");
            }
        }
    }

    void task2() {
        synchronized (lockB) {
            System.out.println("T2 acquired B");

            synchronized (lockA) {
                System.out.println("T2 acquired A");
            }
        }
    }
}
```

### Why can it deadlock?

```text
T1 → holds A → waits for B
T2 → holds B → waits for A
```

### Prevention
Use a consistent global lock ordering:

```text
Always acquire A before B
```

Other approaches include reducing lock scope and using timed `tryLock()` where appropriate.

---

# 7. Scenario: `tryLock()` to Avoid Indefinite Waiting ⭐⭐⭐⭐

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class TryLockScenario {
    private final ReentrantLock lock = new ReentrantLock();

    void work() throws InterruptedException {
        if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
            try {
                System.out.println("Lock acquired");
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println("Could not acquire lock");
        }
    }
}
```

`tryLock()` can help design a bounded response instead of waiting indefinitely, but it does not magically eliminate all deadlocks if lock acquisition is designed poorly.

---

# 8. Scenario: `wait()` / `notifyAll()` Producer-Consumer ⭐⭐⭐⭐⭐

### Practice Code

```java
import java.util.LinkedList;
import java.util.Queue;

public class WaitNotifyScenario {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 3;

    public synchronized void put(int value) throws InterruptedException {
        while (queue.size() == capacity) {
            wait();
        }

        queue.offer(value);
        notifyAll();
    }

    public synchronized int take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();
        }

        int value = queue.poll();
        notifyAll();
        return value;
    }
}
```

### Why `while`, not `if`?
After waking, the condition must be checked again because another thread may have consumed or changed the state before this thread reacquires the monitor.

### Modern alternative
For production producer-consumer code, prefer `BlockingQueue` when it fits the requirement.

---

# 9. Scenario: Producer-Consumer with `BlockingQueue` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class BlockingQueueScenario {
    public static void main(String[] args) throws Exception {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(3);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    queue.put(i);
                    System.out.println("Produced " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    int value = queue.take();
                    System.out.println("Consumed " + value);
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

# 10. Scenario: `sleep()` vs `wait()` ⭐⭐⭐⭐⭐

### Interview Question
Does `sleep()` release a monitor lock?

**No.**

Does `wait()` release the monitor on which it waits?

**Yes.**

### Practice Code

```java
public class SleepVsWaitScenario {
    private final Object lock = new Object();

    void sleepExample() throws InterruptedException {
        synchronized (lock) {
            Thread.sleep(1000); // lock is still held
        }
    }

    void waitExample() throws InterruptedException {
        synchronized (lock) {
            lock.wait(1000); // monitor is released while waiting
        }
    }
}
```

---

# 11. Scenario: `join()` for Ordered Startup/Completion ⭐⭐⭐⭐⭐

### Requirement
Thread B must start its dependent work only after Thread A completes.

### Practice Code

```java
public class JoinScenario {
    public static void main(String[] args) throws Exception {
        Thread taskA = new Thread(() ->
                System.out.println("Task A completed"));

        Thread taskB = new Thread(() ->
                System.out.println("Task B completed"));

        taskA.start();
        taskA.join();
        taskB.start();
        taskB.join();
    }
}
```

`join()` is about waiting for thread termination; it is not a general synchronization replacement.

---

# 12. Scenario: `interrupt()` and Graceful Shutdown ⭐⭐⭐⭐⭐

### Practice Code

```java
public class InterruptScenario {
    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    Thread.sleep(1000);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                System.out.println("Shutdown requested");
            }
        });

        worker.start();
        Thread.sleep(500);
        worker.interrupt();
        worker.join();
    }
}
```

### Interview Point
Interruption is a cooperative cancellation signal. It does not forcibly kill a thread.

When catching `InterruptedException`, preserve the interruption status when the current layer cannot fully handle the cancellation.

---

# 13. Scenario: `yield()` Misconception ⭐⭐⭐⭐

### Question
Does `Thread.yield()` guarantee that another thread will run?

**No.** It is only a scheduler hint.

### Important Java detail
`Thread.State` does not have separate `RUNNING` and `READY` states. Both are represented by `RUNNABLE` from the Java API perspective.

### Practice Code

```java
public class YieldScenario {
    public static void main(String[] args) {
        Runnable task = () -> {
            for (int i = 0; i < 5; i++) {
                System.out.println(Thread.currentThread().getName()
                        + " -> " + i);
                Thread.yield();
            }
        };

        new Thread(task, "T1").start();
        new Thread(task, "T2").start();
    }
}
```

Never use `yield()` as a correctness mechanism.

---

# 14. Scenario: `notify()` vs `notifyAll()` ⭐⭐⭐⭐⭐

### Question
Which should be preferred when multiple threads may be waiting for different conditions?

Usually `notifyAll()` is safer when the awakened thread must re-check a condition and there may be multiple waiting roles.

### Key rule

```text
notify()    → wakes one waiting thread
notifyAll() → wakes all waiting threads
```

The awakened thread still has to reacquire the monitor before continuing.

---

# 15. Scenario: Class Lock vs Object Lock ⭐⭐⭐⭐

```java
public class LockScenario {

    public synchronized void instanceMethod() {
        // object lock: this
    }

    public static synchronized void staticMethod() {
        // class lock: LockScenario.class
    }
}
```

### Interview Question
Can an instance synchronized method and a static synchronized method block each other?

Not merely because both are `synchronized`: they use different monitors (`this` versus the `Class` object).

---

# 16. Scenario: Reentrancy ⭐⭐⭐⭐

```java
public class ReentrancyScenario {

    public synchronized void outer() {
        inner();
    }

    public synchronized void inner() {
        System.out.println("Re-entered synchronized method");
    }
}
```

The same thread can reacquire a monitor it already owns. Java intrinsic monitors are reentrant.

---

# 17. Scenario: Concurrent Map Check-Then-Act ⭐⭐⭐⭐⭐

### Unsafe logic

```java
if (!map.containsKey(key)) {
    map.put(key, value);
}
```

Even with `ConcurrentHashMap`, the combination is not one atomic operation.

### Better

```java
map.putIfAbsent(key, value);
```

### Practice Code

```java
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentMapScenario {
    public static void main(String[] args) throws Exception {
        ConcurrentHashMap<String, Integer> map =
                new ConcurrentHashMap<>();

        Runnable task = () -> {
            map.putIfAbsent("java", 1);
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(map);
    }
}
```

---

# 18. Scenario: `ThreadLocal` with Thread Pools ⭐⭐⭐⭐⭐

### Problem
A worker thread from a pool may execute multiple unrelated tasks. A `ThreadLocal` value can remain attached to that worker if it is not removed.

### Practice Code

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadLocalPoolScenario {
    private static final ThreadLocal<String> USER = new ThreadLocal<>();

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(1);

        executor.submit(() -> {
            try {
                USER.set("request-A");
                System.out.println(USER.get());
            } finally {
                USER.remove();
            }
        }).get();

        executor.submit(() ->
                System.out.println("Next task sees = " + USER.get())).get();

        executor.shutdown();
    }
}
```

Expected second task value:

```text
Next task sees = null
```

---

# 19. Scenario: Happens-Before Through `join()` ⭐⭐⭐⭐⭐

### Practice Code

```java
public class JoinVisibilityScenario {
    private static int result;

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> result = 42);

        worker.start();
        worker.join();

        System.out.println(result); // 42
    }
}
```

The successful return from `join()` establishes the relevant happens-before relationship from actions in the terminated thread to actions after the `join()` returns.

---

# 20. Scenario: Safe Publication with `volatile` Reference ⭐⭐⭐⭐

```java
public class PublicationScenario {
    private static volatile Config config;

    static class Config {
        final int port;

        Config(int port) {
            this.port = port;
        }
    }

    static void publish() {
        config = new Config(8080);
    }

    static void read() {
        Config local = config;
        if (local != null) {
            System.out.println(local.port);
        }
    }
}
```

The volatile reference provides a visibility/ordering mechanism for publication of the constructed object's state, assuming correct construction and no unsafe escape during construction.

---

# 21. Scenario: Starvation ⭐⭐⭐⭐

### Definition
A thread remains unable to make progress because other threads repeatedly obtain the resources or scheduling opportunities it needs.

### Interview Distinction

```text
Deadlock   → threads wait forever on each other
Starvation → a thread keeps getting denied progress
Livelock   → threads keep reacting but make no useful progress
```

### Prevention ideas

- fair locking where appropriate
- avoid long critical sections
- avoid unnecessary lock contention
- design bounded resource ownership

---

# 22. Scenario: Livelock ⭐⭐⭐⭐

Two threads repeatedly detect each other's activity and keep changing their behavior, but neither completes useful work.

### Interview example

```text
Thread A: sees B → backs off
Thread B: sees A → backs off
Thread A: retries → sees B
Thread B: retries → sees A
...
```

Unlike deadlock, threads may remain active rather than blocked.

---

# 23. Scenario: Race Condition vs Data Race ⭐⭐⭐⭐⭐

### Race condition
Program correctness depends on timing/interleaving.

### Data race
Two conflicting accesses to the same memory location occur concurrently, at least one is a write, without an appropriate happens-before ordering.

They are related but not identical concepts.

A program can have a logical race even when individual operations are protected in ways that do not preserve the required higher-level invariant.

---

# 24. Scenario: Atomicity vs Visibility vs Ordering ⭐⭐⭐⭐⭐

| Requirement | Typical Tool / Mechanism |
|---|---|
| Visibility | `volatile`, lock/unlock, thread start/join and other happens-before edges |
| Atomic single-variable update | Atomic classes / CAS |
| Compound invariant | `synchronized` / `Lock` |
| Ordering | Java Memory Model happens-before guarantees |
| Concurrent collection access | `ConcurrentHashMap`, queues, etc. |

### Interview Rule
Never choose a concurrency primitive only because it sounds faster. First identify the required correctness guarantee.

---

# 25. Scenario: Multiple Conditions on One Lock ⭐⭐⭐⭐

If multiple independent waiting conditions exist, `ReentrantLock` with separate `Condition` objects can be clearer than one intrinsic wait set.

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ConditionScenario {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();
}
```

This allows signaling a relevant condition rather than waking every waiter on the same intrinsic monitor.

---

# 26. Scenario: Which Primitive Should I Choose? ⭐⭐⭐⭐⭐

### Simple stop flag

```text
volatile boolean
```

### Single numeric counter

```text
AtomicInteger / LongAdder depending on workload
```

### Multiple fields must change together

```text
synchronized / Lock
```

### Producer-consumer

```text
BlockingQueue
```

### Shared concurrent map

```text
ConcurrentHashMap
```

### Per-thread context

```text
ThreadLocal, with cleanup in pooled threads
```

### Read-heavy listener/configuration list

```text
CopyOnWriteArrayList
```

### Need lock timeout / interruptible acquisition / conditions

```text
ReentrantLock
```

---

# 27. Rapid-Fire Interview Questions ⭐⭐⭐⭐⭐

### Q1. Does `sleep()` release a lock?
No.

### Q2. Does `wait()` release the monitor it waits on?
Yes, until it reacquires it before returning.

### Q3. Does `yield()` guarantee context switching?
No.

### Q4. Does `volatile` make `count++` atomic?
No.

### Q5. Is `synchronized` reentrant?
Yes.

### Q6. What lock does a synchronized instance method use?
The instance's monitor, normally `this`.

### Q7. What lock does a static synchronized method use?
The monitor of the `Class` object.

### Q8. Why use `while` around `wait()`?
To re-check the condition after waking and before proceeding.

### Q9. What is a good modern producer-consumer primitive?
Usually `BlockingQueue`, when the requirement fits it.

### Q10. Does `interrupt()` kill a thread?
No. It requests interruption/cooperative cancellation.

### Q11. How can deadlock be prevented?
Consistent lock ordering, smaller critical sections, avoiding unnecessary nested locks, and suitable timed acquisition strategies.

### Q12. Is `ConcurrentHashMap` enough for every compound operation?
No. Use its atomic compound APIs or external coordination when the business invariant spans multiple operations.

### Q13. What is safe publication?
Publishing a reference through a mechanism that gives the receiving thread the required visibility and initialization guarantees.

### Q14. What is the difference between starvation and deadlock?
Deadlock is circular waiting; starvation is indefinite denial of progress without necessarily having circular wait.

### Q15. What is the difference between livelock and deadlock?
In livelock threads remain active but fail to make useful progress; in deadlock threads are blocked waiting for each other/resources.

---

# 28. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"For a multithreading problem, I first identify whether the requirement is visibility, atomicity, ordering, mutual exclusion, or coordination. For a simple stop flag I can use `volatile`; for a single atomic counter I may use `AtomicInteger`; for a compound invariant I use `synchronized` or `Lock` to protect the complete critical section. For producer-consumer I prefer `BlockingQueue` when appropriate. I use `ConcurrentHashMap` for concurrent map access and its atomic methods for compound map operations. For cancellation I use `interrupt()` cooperatively, and for thread completion I can use `join()`. I also check for deadlock, starvation, livelock, race conditions and unsafe publication. Finally, I explain why the chosen primitive provides the exact correctness guarantee required rather than simply saying it is thread-safe."**

---

# 29. Interview Problem-Solving Framework ⭐⭐⭐⭐⭐

When an interviewer gives a concurrency scenario, answer in this order:

```text
1. Identify shared state
        ↓
2. Identify the race / invariant
        ↓
3. Identify required guarantee
   visibility / atomicity / ordering / mutual exclusion / coordination
        ↓
4. Select primitive
   volatile / atomic / synchronized / Lock / concurrent collection / queue
        ↓
5. Explain why
        ↓
6. Consider interruption / shutdown
        ↓
7. Check deadlock / starvation / livelock
        ↓
8. Discuss performance trade-off
```

This framework is more valuable in interviews than memorizing isolated definitions.

---

# 30. Final Practice Checklist

- [x] Race condition counter
- [x] Atomic counter
- [x] `volatile` stop flag
- [x] `volatile` counter trap
- [x] Thread-safe bank account
- [x] Deadlock
- [x] `tryLock()`
- [x] `wait()` / `notifyAll()`
- [x] `BlockingQueue`
- [x] `sleep()` vs `wait()`
- [x] `join()`
- [x] `interrupt()`
- [x] `yield()`
- [x] `notify()` vs `notifyAll()`
- [x] Class lock vs object lock
- [x] Reentrancy
- [x] ConcurrentHashMap compound operations
- [x] ThreadLocal + thread pools
- [x] Happens-before with `join()`
- [x] Safe publication
- [x] Starvation
- [x] Livelock
- [x] Race condition vs data race
- [x] Atomicity / visibility / ordering
- [x] Conditions
- [x] Primitive selection
- [x] Rapid-fire interview Q&A

---

## Navigation

[← 7.34 — Common Thread-Safety Strategies](../34-Common-Thread-Safety-Strategies/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.36 — Multithreading Quick Revision**